# Object 更新路径深度分析

## 一、总体概览：为什么对象更新如此复杂？

Compress 层中，一个逻辑文件对应两个物理文件（数据+元数据），且**压缩数据文件的文件名中编码了原始文件的大小**。当文件内容变更导致大小变化时，数据文件名必然改变——这使得"原地更新"无法像普通文件系统那样直接覆盖，而需要先上传新文件、再删除旧文件、最后修正引用。

更新路径分为三种入口：
- `Fs.Put()` — 已知大小的常规上传（最常用路径）
- `Fs.PutStream()` — 未知大小的流式上传
- `Fs.Copy()/Move()` — 服务端复制/移动时的目标覆盖

三者共享相同的底层机制，但各自处理逻辑有显著差异。

---

## 二、Fs.Put() 入口：新建 vs 更新的分叉

[Fs.Put()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L719-L736) 是大多数写入操作的统一入口：

```go
func (f *Fs) Put(ctx context.Context, in io.Reader, src fs.ObjectInfo, options ...fs.OpenOption) (fs.Object, error) {
    // 1. 先尝试定位已有对象
    o, err := f.NewObject(ctx, src.Remote())
    
    if err == fs.ErrorObjectNotFound {
        // 分支 A：对象不存在 → 走新建流程
        in, compressible, mimeType, err := checkCompressAndType(in, f.mode, f.modeHandler)
        return f.putWithCustomFunctions(ctx, in, src, options, f.Fs.Put, f.Fs.Put, compressible, mimeType)
    }
    if err != nil {
        return nil, err
    }
    // 分支 B：对象已存在 → 委托给 Object.Update() 处理
    return o, o.Update(ctx, in, src, options...)
}
```

**设计意图**：注释明确说明优先选择 Update 而非"先删后建"，因为删除会**破坏服务端版本历史**（如 S3 的版本控制功能）。Update 能尽量保留已有对象的版本链。

---

## 三、Object.Update()：核心更新逻辑详解

[Object.Update()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1120-L1166) 是整个更新机制的核心，内部根据**旧压缩模式**和**新内容是否可压缩**分为两条截然不同的路径。

### 3.1 前置准备：元数据加载与闭包注入

```go
func (o *Object) Update(ctx context.Context, in io.Reader, src fs.ObjectInfo, options ...fs.OpenOption) (err error) {
    // 步骤 1：加载已有元数据（获取旧的 Mode、块索引等）
    err = o.loadMetadataIfNotLoaded(ctx)

    // 步骤 2：定义元数据更新闭包
    // ★ 关键：无论走哪条分支，元数据文件始终是原地 Update（不会改名）
    updateMeta := func(ctx context.Context, in io.Reader, src fs.ObjectInfo, options ...fs.OpenOption) (fs.Object, error) {
        return o.mo, o.mo.Update(ctx, in, src, options...)
    }

    // 步骤 3：检测新内容是否可压缩
    in, compressible, mimeType, err := checkCompressAndType(in, o.meta.Mode, o.f.modeHandler)
```

**关于元数据文件**：`xxx.json` 这个元数据文件名**不随大小变化**（因为逻辑名 `xxx` 没变），所以元数据总是可以原地 `Update`，无需删除重建。这是整个更新机制的稳定锚点。

### 3.2 分支判定条件

```go
    if o.meta.Mode != Uncompressed || compressible {
        // 路径 A（下文 3.3 详解）：
        // 旧对象是压缩的 - 或 - 新内容可压缩
        // → 使用 Put 上传新数据文件（可能产生新文件名）
    } else {
        // 路径 B（下文 3.4 详解）：
        // 旧对象未压缩 && 新内容也不可压缩
        // → 可原地 Update 数据文件
    }
```

判定矩阵：

| 旧模式 \ 新内容 | 可压缩 | 不可压缩 |
|-----------------|--------|----------|
| Gzip/Zstd | 路径 A（Put 新文件）| 路径 A（Put 新文件）|
| Uncompressed | 路径 A（Put 新文件）| **路径 B（原地 Update）** |

### 3.3 路径 A："先建后删"式更新（压缩场景）

```go
    origName := o.Remote()
    if o.meta.Mode != Uncompressed || compressible {
        // 用 Put 上传全新的数据文件
        newObject, err = o.f.putWithCustomFunctions(
            ctx, in,
            o.f.wrapInfo(src, origName, src.Size()), // ★ 注意这里包装了 ObjectInfo
            options,
            o.f.Fs.Put,       // putData: 用底层 Fs.Put（会创建新物理文件）
            updateMeta,       // putMeta: 原地更新元数据
            compressible, mimeType,
        )
        if err != nil {
            return err
        }
        // ★ 关键清理逻辑：如果新旧数据文件名不同，删除旧数据文件
        if newObject.Object.Remote() != o.Object.Remote() {
            if removeErr := o.Object.Remove(ctx); removeErr != nil {
                return removeErr
            }
        }
    }
```

**为什么用 Put 而不用 Update？**
- Put 会创建一个全新的物理对象，名称由 `makeDataName(origName, src.Size(), mode)` 决定
- 新文件大小与旧文件几乎必然不同 → 文件名中的 base64(size) 编码部分不同 → 文件名不同
- 如果对旧文件（不同文件名）调用 Update，底层会认为是"对不存在的对象更新"而出错

**文件名不一致的判定**：
- `newObject.Object.Remote()` 是刚上传的新数据文件名（含新大小编码）
- `o.Object.Remote()` 是旧数据文件名（含旧大小编码）
- 只要两者不同，就删除旧数据文件（元数据文件不动，因为已经被 updateMeta 原地覆盖了）

**极端情况——文件名碰巧相同**（大小没变）：
- 跳过删除，新 Put 的对象覆盖了同名旧对象（底层 Put 的语义通常就是覆盖）
- 这符合"保留版本历史"的设计意图

### 3.4 路径 B：原地更新（仅双端未压缩场景）

```go
    } else {
        // 定义数据文件原地更新的闭包
        update := func(ctx context.Context, in io.Reader, src fs.ObjectInfo, options ...fs.OpenOption) (fs.Object, error) {
            return o.Object, o.Object.Update(ctx, in, src, options...)
        }
        // putData 注入 update 闭包，putMeta 注入 updateMeta 闭包
        newObject, err = o.f.putWithCustomFunctions(ctx, in, src, options, update, updateMeta, compressible, mimeType)
```

**为什么只有双端未压缩才能原地更新？**
- 未压缩文件命名：`xxx.bin`，文件名**不含大小编码**
- 无论新文件大小怎么变，数据文件名始终是同一个：`xxx.bin`
- 因此底层 `o.Object.Update()` 可以正确定位并覆盖
- 同时保留底层存储的版本历史

### 3.5 收尾：原子替换 Object 的内部引用

```go
    // 无论哪条分支，最后都用 newObject 的字段覆盖当前对象
    o.Object = newObject.Object   // 指向新数据文件
    o.meta   = newObject.meta     // 指向新元数据结构体
    o.size   = newObject.size     // 更新缓存大小
    return nil
}
```

**注意**：此处是**指针字段的就地修改**，外部调用者持有的还是同一个 `*Object` 指针，但其内部引用已全部指向新内容。这实现了"透明更新"——调用者无需关心对象底层是否换了物理文件。

---

## 四、putWithCustomFunctions 的注入机制

[putWithCustomFunctions()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L686-L712) 是新建与更新的共享编排器。它通过函数参数注入实现"同一个编排逻辑适配不同写入方式"：

```go
func (f *Fs) putWithCustomFunctions(
    ctx context.Context, in io.Reader, src fs.ObjectInfo, options []fs.OpenOption,
    putData  putFn,   // ← 数据写入策略（Put / Update / PutStream）
    putMeta  putFn,   // ← 元数据写入策略（Put / Update）
    compressible bool, mimeType string,
) (*Object, error) {
    // 1. 先写数据文件
    if compressible {
        dataObject, meta, err = f.putCompress(ctx, in, src, options, mimeType)
        // putCompress 内部会根据 src.Size() 和 mode 生成正确的文件名
    } else {
        dataObject, meta, err = f.putUncompress(ctx, in, src, putData, options, mimeType)
        // putUncompress 调用传入的 putData 闭包（而非固定用 Put）
    }

    // 2. 再写元数据文件
    mo, err := f.putMetadata(ctx, meta, src, options, putMeta)
    // putMetadata 调用传入的 putMeta 闭包（而非固定用 Put）

    // 3. 失败回滚：元数据上传失败则删除已上传的数据文件
    if err != nil {
        removeError := dataObject.Remove(ctx)
        ...
    }
    return f.newObject(dataObject, mo, meta), nil
}
```

四种场景的注入参数对照：

| 场景 | putData | putMeta |
|------|---------|---------|
| **新建对象 (Put)** | `f.Fs.Put` — 创建新数据文件 | `f.Fs.Put` — 创建新元数据 |
| **Update 路径 A** | `f.Fs.Put` — 创建新数据文件（可能新名）| `updateMeta` — 原地更新元数据 |
| **Update 路径 B** | `update` 闭包 — 原地更新数据文件 | `updateMeta` — 原地更新元数据 |
| **PutStream** | `f.Fs.Features().PutStream` — 流式传 | `f.Fs.Put` — 正常上传元数据 |

---

## 五、PutStream 流式更新：临时名 + 重命名

[Fs.PutStream()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L739-L773) 用于上传时大小未知的场景。由于压缩后的大小要到压缩完成才能确定，数据文件一开始只能用**临时错误的文件名**上传，完成后再通过服务端 Move 重命名。

```go
func (f *Fs) PutStream(ctx context.Context, in io.Reader, src fs.ObjectInfo, options ...fs.OpenOption) (fs.Object, error) {
    // 1. 查找旧对象（用于后续清理）
    oldObj, err := f.NewObject(ctx, src.Remote())
    found := err == nil

    // 2. 检测可压缩性，上传新数据 + 元数据
    in, compressible, mimeType, err := checkCompressAndType(in, f.mode, f.modeHandler)
    newObj, err := f.putWithCustomFunctions(ctx, in, src, options,
        f.Fs.Features().PutStream,  // putData: PutStream
        f.Fs.Put,                   // putMeta: 普通 Put
        compressible, mimeType)
```

### 5.1 问题所在：流式上传产生了错误的文件名

压缩模式下，`putCompress` 内部调用 `rcat()` 时传入的目标文件名是：
```go
makeDataName(src.Remote(), src.Size(), f.mode)
```

但 **PutStream 场景中 src.Size() == -1**（未知），编码进文件名的大小就是错误值。上传完成后必须改名。

### 5.2 步骤 3：清理旧数据文件

```go
    // 如果存在旧对象，且（旧是压缩的 或 新是可压缩的），则删除旧数据
    // 注意：只有双端都未压缩时才跳过删除
    if found && (oldObj.(*Object).meta.Mode != Uncompressed || compressible) {
        err = oldObj.(*Object).Object.Remove(ctx)  // 只删数据，不删元数据（元数据已被覆盖）
    }
```

### 5.3 步骤 4：服务端重命名修正文件名

```go
    if compressible {
        // newObj.size 是压缩完成后从元数据中拿到的真实原始大小
        // 用真实大小生成正确的文件名，通过 Move 改名
        wrapObj, err := operations.Move(
            ctx, f.Fs, nil,
            f.dataName(src.Remote(), newObj.size, compressible), // 正确文件名
            newObj.Object,                                       // 上传时的临时文件
        )
        newObj.Object = wrapObj  // 修正引用
    }
    return newObj, nil
}
```

**注意**：PutStream 中 `putMeta` 使用的是 `f.Fs.Put`（原地覆盖语义），而不是 Update 闭包。因为 PutStream 中无法像普通 Put 那样先定位到旧对象再复用其元数据引用。

---

## 六、Copy/Move 中的目标覆盖更新

[Fs.Copy()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L834-L873) 和 [Fs.Move()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L884-L924) 中都涉及"目标文件已存在"的场景：

```go
func (f *Fs) Copy(ctx context.Context, src fs.Object, remote string) (fs.Object, error) {
    // 1. 先尝试定位目标文件
    dstFile, err := f.NewObject(ctx, remote)
    if err == nil {
        // 目标文件存在 → 直接全删除（数据+元数据）
        err := dstFile.Remove(ctx)  // Remove 会同时删 .json 和 .gz/.zst/.bin
    }
    
    // 2. 用新大小计算目标文件名，分别 Copy 元数据和数据
    newFilename := makeMetadataName(remote)
    moResult, err := do(ctx, o.mo, newFilename)       // .json
    newFilename = makeDataName(remote, src.Size(), o.meta.Mode)
    oResult, err := do(ctx, o.Object, newFilename)    // .gz/.zst/.bin
}
```

Copy/Move 采用最简单的"先全删再重建"策略。原因：
1. Copy/Move 不涉及重新压缩（已有压缩数据直接复制），所以大小已知且确定
2. 目标文件如果大小不同，数据文件名必然不同，用底层 Copy 到新名再删旧名的方式不如直接 Remove 干净
3. [Object.Remove()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1100-L1111) 会同时删除元数据和数据两个物理文件

---

## 七、更新路径全景流程图

```
用户写入 (Put/PutStream/Update)
    │
    ├─ Fs.Put(src)
    │    │
    │    ├─ NewObject 找到旧对象? ──No──► putWithCustomFunctions(新建)
    │    │                                └─ putData=Fs.Put, putMeta=Fs.Put
    │    │
    │    └─ Yes ──► Object.Update(in, src)
    │                    │
    │                    ├─ 加载旧元数据
    │                    ├─ 闭包 updateMeta = o.mo.Update (★ 元数据始终原地更新)
    │                    ├─ checkCompressAndType 检测新内容
    │                    │
    │                    ├─ 旧压缩 || 新可压缩?
    │                    │    │
    │                    │    ├─ Yes ──► putWithCustomFunctions(路径A)
    │                    │    │           putData=Fs.Put (创建新文件,名含新大小)
    │                    │    │           putMeta=updateMeta
    │                    │    │           │
    │                    │    │           └─ 新文件名 ≠ 旧名? ──Yes──► 删除旧数据文件
    │                    │    │
    │                    │    └─ No (双端未压缩) ──► putWithCustomFunctions(路径B)
    │                    │                            putData=update 闭包 (o.Object.Update 原地)
    │                    │                            putMeta=updateMeta
    │                    │
    │                    └─ o.Object/meta/size = newObject.* (原子替换引用)
    │
    └─ Fs.PutStream(src)
         │
         ├─ 查找旧对象并记录 found
         ├─ checkCompressAndType
         ├─ putWithCustomFunctions (临时错误文件名上传)
         │    putData=Fs.PutStream
         │    putMeta=Fs.Put
         │
         ├─ 旧存在 && (旧压缩 || 新可压缩)? ──Yes──► 删除旧数据文件
         │
         └─ 新可压缩? ──Yes──► operations.Move(临时名 → 正确大小文件名)
              └─ (用 newObj.size 即真实原始大小重命名)
```

---

## 八、关键设计要点总结

| 设计点 | 说明 |
|--------|------|
| **元数据文件是稳定锚点** | `.json` 名不含大小编码，任何更新都可原地 Update，不动引用链 |
| **更新优先于删建** | Put 入口走 Update（而非先 Remove 再 Put），目的是保留底层服务端版本历史 |
| **双端未压缩才能真原地** | 只有 Mode=Uncompressed → 不可压缩时，数据文件名才稳定（`.bin`），才能调用底层 Update |
| **新旧同名判定** | 路径 A 中通过比较 `Remote()` 字符串判断是否需要清理旧文件，避免误删同大小场景 |
| **失败回滚** | `putWithCustomFunctions` 中"元数据失败必删数据"，保证数据与元数据始终成对存在 |
| **流式修正命名** | PutStream 通过"先传临时名 → Move 重命名"解决压缩后大小未知的问题 |
| **Copy/Move 简单粗暴** | 直接 Remove 目标再重建，因为不涉及重新压缩，名大小确定，简单删除更可靠 |
