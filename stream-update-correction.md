# PutStream 流式更新：勘误与重新核准

## 一、之前的两处错误

| # | 错误描述 | 之前写的 | 实际代码 |
|---|----------|----------|----------|
| 1 | 旧对象定位 | "旧对象的 meta 已加载" | NewObject 返回的对象 meta **已经完整加载**，但 PutStream 对旧对象的**元数据文件没有做任何复用或原地 Update**——它完全忽略了旧对象的元数据引用 |
| 2 | 元数据写入方式 | "元数据被原地覆盖"、"putMeta 用 Fs.Put 有原地覆盖语义" | `f.Fs.Put` 传入 `putMetadata` 后实际执行的是**创建新 .json 文件**（底层 Put 语义是"若存在则覆盖同名对象"，而非"调用旧对象 Update"）。关键区别：PutStream 没有像 `Object.Update` 那样注入 `updateMeta` 闭包来复用旧元数据对象的引用 |

---

## 二、旧对象定位：NewObject 完整加载 + 仅用于清理

[PutStream L740-L744](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L740-L744)：

```go
oldObj, err := f.NewObject(ctx, src.Remote())
if err != nil && err != fs.ErrorObjectNotFound {
    return nil, err
}
found := err == nil
```

[NewObject()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L495-L515) 的执行流程：

1. 用底层 `Fs.NewObject` 定位 `.json` 元数据文件 → 得到 `mo`
2. 用 `readMetadata(ctx, mo)` 读取并解码 JSON → 得到 `meta`（**完整加载**）
3. 从 `meta` 取出原始大小，用 `makeDataName` 构造数据文件名
4. 用底层 `Fs.NewObject` 定位数据文件 → 得到 `o`
5. 用 `f.newObject(o, mo, meta)` 构造返回，其中 `meta` 字段**不为 nil**

所以 `oldObj` 是一个**完整加载了元数据的 Object**——它同时持有：
- `oldObj.Object` — 旧数据文件的底层引用
- `oldObj.mo` — 旧元数据文件的底层引用
- `oldObj.meta` — 旧元数据结构体（Mode/Size/MD5/块索引 全部已填充）

**但 PutStream 对这个旧对象唯一做的事就是**：

```go
if found && (oldObj.(*Object).meta.Mode != Uncompressed || compressible) {
    err = oldObj.(*Object).Object.Remove(ctx)  // ★ 只删除旧数据文件
}
```

它**只读取了 `meta.Mode` 来判断旧对象是否压缩**，然后**只删除了旧数据文件**。旧元数据文件 `.json` 的清理则完全交给了底层 `Fs.Put` 的覆盖语义——新 `.json` 上传后覆盖同名旧 `.json`。

### 与 Object.Update 的关键差异

[Object.Update L1120-L1128](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1120-L1128)：

```go
err = o.loadMetadataIfNotLoaded(ctx)  // 加载旧元数据

updateMeta := func(ctx context.Context, in io.Reader, src fs.ObjectInfo, options ...fs.OpenOption) (fs.Object, error) {
    return o.mo, o.mo.Update(ctx, in, src, options...)  // ★ 复用旧元数据对象引用，原地 Update
}
```

Update 路径：
- 旧元数据文件 `o.mo` 被**复用**，通过闭包注入 `putMeta`
- `o.mo.Update()` 对**同一个物理文件**执行原地内容替换
- 不创建新文件，保留服务端版本历史

PutStream 路径：
- 旧元数据文件 `oldObj.mo` **没有被复用**
- `putMeta` 传入的是 `f.Fs.Put`（而非 `updateMeta` 闭包）
- 底层 `Fs.Put` 对 `.json` 执行的是**同名创建/覆盖**，等价于"删旧建新"

---

## 三、元数据写入方式：Fs.Put 而非 updateMeta 闭包

[PutStream L750](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L750)：

```go
newObj, err := f.putWithCustomFunctions(ctx, in, src, options,
    f.Fs.Features().PutStream,  // putData: 流式上传数据
    f.Fs.Put,                   // putMeta: ★ 用 Fs.Put 而非 updateMeta 闭包
    compressible, mimeType)
```

追踪 `f.Fs.Put` 在 `putMetadata` 中的实际行为 [putMetadata L658-L680](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L658-L680)：

```go
func (f *Fs) putMetadata(ctx context.Context, meta *ObjectMetadata, src fs.ObjectInfo, options []fs.OpenOption, put putFn) (mo fs.Object, err error) {
    data, err := json.Marshal(meta)
    metaReader := bytes.NewReader(data)

    // ★ 这里 put = f.Fs.Put，不是 o.mo.Update
    mo, err = put(ctx, metaReader,
        f.wrapInfo(src, makeMetadataName(src.Remote()), int64(len(data))),
        options...)
    ...
}
```

`f.Fs.Put` 是底层文件系统的 Put 方法，它接收的 ObjectInfo 的 Remote 为 `makeMetadataName(src.Remote())`，即 `{remote}.json`。

底层 `Fs.Put` 的语义是：
- 若 `{remote}.json` 不存在 → 创建新文件
- 若 `{remote}.json` 已存在 → **覆盖**（底层 Put 内部可能先 NewObject 再 Update，也可能是直接覆盖，取决于底层实现）

**这与 `updateMeta` 闭包（`o.mo.Update`）有本质区别**：

| 方式 | 函数 | 作用对象 | 是否复用旧引用 | 版本历史 |
|------|------|----------|----------------|----------|
| `updateMeta` 闭包 | `o.mo.Update()` | 旧元数据文件对象 | 是，直接在旧对象上 Update | 保留版本链 |
| `f.Fs.Put` | 底层 Fs.Put | 同名新上传 | 否，底层 Put 可能创建新对象 | 可能中断版本链 |

### 为什么 PutStream 不能用 updateMeta 闭包？

根本原因：**PutStream 没有把旧对象"持有"为 Object.Update 那样的自身引用**。

在 `Object.Update` 中：
```go
o.loadMetadataIfNotLoaded(ctx)  // o.mo 被加载
updateMeta := func(...) {
    return o.mo, o.mo.Update(...)  // o 是方法接收者，天然持有旧引用
}
```

而在 `PutStream` 中：
```go
oldObj, err := f.NewObject(ctx, src.Remote())  // oldObj 是局部变量
// ...
newObj, err := f.putWithCustomFunctions(...)    // newObj 是新创建的
// oldObj 和 newObj 之间没有引用关系
```

`putWithCustomFunctions` 内部调用 `putMetadata` 时，需要的 `putMeta` 是一个符合 `putFn` 签名的通用函数。如果要用 `updateMeta`，就需要构造一个闭包引用 `oldObj.mo`，但 `putWithCustomFunctions` 并不知道旧对象的存在——它只接收 `putData`/`putMeta` 两个函数参数。

**理论上可以这样做**：
```go
// 假设的改法（实际代码没有这样做）
updateOldMeta := func(ctx context.Context, in io.Reader, src fs.ObjectInfo, options ...fs.OpenOption) (fs.Object, error) {
    return oldObj.mo, oldObj.mo.Update(ctx, in, src, options...)
}
newObj, err := f.putWithCustomFunctions(ctx, in, src, options,
    f.Fs.Features().PutStream, updateOldMeta, compressible, mimeType)
```

但实际代码选择了 `f.Fs.Put`，因为：
1. 更简单，不需要额外处理 `oldObj` 的类型断言和 nil 检查
2. PutStream 场景下旧对象的版本历史保留不那么重要（流式上传通常不涉及 S3 版本控制等高级功能）
3. `.json` 文件名固定不变（`{remote}.json`），`Fs.Put` 对同名文件的覆盖语义已经足够

---

## 四、PutStream vs Object.Update 完整对比

### 4.1 流程对比表

| 步骤 | Object.Update | Fs.PutStream |
|------|---------------|--------------|
| **旧对象定位** | `Fs.Put` 内部 `NewObject` → 传给 `o.Update`，**o 自身就是旧对象** | 独立 `NewObject` → 局部变量 `oldObj`，**与后续上传流程无关** |
| **旧元数据加载** | `o.loadMetadataIfNotLoaded()` 加载到 `o.meta` | `NewObject` 内部已加载，但在上传流程中**不被使用** |
| **元数据写入** | `updateMeta` 闭包 → `o.mo.Update()` **原地替换内容** | `f.Fs.Put` → 底层 Put **同名覆盖/重建** |
| **数据写入** | 路径A: `f.Fs.Put` 新建；路径B: `o.Object.Update` 原地 | 始终 `f.Fs.Features().PutStream` **流式新建** |
| **旧数据清理** | 路径A: 比较文件名，不同则删 `o.Object`；路径B: 无需清理 | `oldObj.Object.Remove()` 仅删旧数据文件 |
| **旧元数据清理** | 不需要（原地 Update） | 隐式：新 `.json` 覆盖旧 `.json` |
| **文件名修正** | 不需要（数据文件名在 Put 时已正确） | `operations.Move` 重命名（压缩时大小未知，需后修正） |
| **自引用替换** | `o.Object/meta/size = newObject.*` 就地更新 | 直接返回 `newObj`，不替换旧对象引用 |

### 4.2 旧数据清理策略对比

**Object.Update 路径 A** [L1139-L1148](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1139-L1148)：
```go
if o.meta.Mode != Uncompressed || compressible {
    newObject, err = o.f.putWithCustomFunctions(...)
    // ★ 条件删除：只有文件名确实不同时才删
    if newObject.Object.Remote() != o.Object.Remote() {
        if removeErr := o.Object.Remove(ctx); removeErr != nil {
            return removeErr
        }
    }
}
```
- 精确判断：比较新旧 Remote 字符串，同大小（同名）时跳过删除
- 删除失败直接返回错误（中止更新）

**PutStream** [L757-L762](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L757-L762)：
```go
if found && (oldObj.(*Object).meta.Mode != Uncompressed || compressible) {
    err = oldObj.(*Object).Object.Remove(ctx)
}
```
- 粗粒度判断：只要旧压缩或新可压缩就直接删，**不检查新旧文件名是否相同**
- 实际上 PutStream 中新文件名几乎必然不同（src.Size() == -1 时编码出的文件名不同），所以这个简化是合理的
- 但如果新旧恰好大小相同且都用 gzip，理论上可能误删同名文件——不过此时 `operations.Move` 重命名步骤也会覆盖，所以不会丢失数据

### 4.3 返回值语义对比

**Object.Update**：
```go
o.Object = newObject.Object
o.meta = newObject.meta
o.size = newObject.size
return nil
```
- 就地修改调用者持有的 Object，返回 error
- 调用者（Fs.Put）返回 `o, nil`——同一个指针，内部已更新

**PutStream**：
```go
return newObj, nil
```
- 返回全新构造的 Object
- 调用者拿到的是 `putWithCustomFunctions` + 重命名后构造的新对象
- `oldObj` 在函数结束后不再被引用

---

## 五、PutStream 完整执行流程（修正版）

```
PutStream(in, src)       src.Size() == -1 (大小未知)
    │
    ├─ 1. NewObject(src.Remote()) ──► oldObj (完整加载，meta ≠ nil)
    │      │
    │      ├─ 不存在 → found = false
    │      └─ 存在   → found = true, oldObj 有 .Object/.mo/.meta
    │
    ├─ 2. checkCompressAndType() ──► compressible, mimeType
    │
    ├─ 3. putWithCustomFunctions(in, src,
    │        putData = f.Fs.Features().PutStream,   ← 流式上传数据
    │        putMeta = f.Fs.Put,                     ← ★ 用底层 Put 写元数据（非 updateMeta 闭包）
    │        compressible, mimeType)
    │      │
    │      ├─ compressible?
    │      │    ├─ Yes → putCompress()
    │      │    │         ├─ Pipe+goroutine 异步压缩
    │      │    │         └─ rcat(临时文件名, ...) → 上传
    │      │    │              临时文件名 = makeDataName(remote, -1, mode)
    │      │    │              ★ -1 编码后是错误的大小标识
    │      │    │
    │      │    └─ No  → putUncompress(in, putData=PutStream, ...)
    │      │              └─ PutStream(远程.bin) → 上传（.bin 不含大小，文件名正确）
    │      │
    │      └─ putMetadata(meta, src, putMeta=f.Fs.Put)
    │           └─ f.Fs.Put(jsonBytes, {remote}.json)
    │              ★ 底层 Put 对同名 .json 执行覆盖（非原地 Update）
    │
    ├─ 4. 旧数据清理
    │      if found && (oldObj.meta.Mode != Uncompressed || compressible):
    │          oldObj.Object.Remove()   ← 只删旧数据文件
    │          ★ 旧 .json 已被步骤3隐式覆盖，无需额外删除
    │          ★ 未压缩场景下 .bin 同名也会被步骤3覆盖
    │
    ├─ 5. 文件名修正（仅 compressible）
    │      operations.Move(临时名 → makeDataName(remote, newObj.size, true))
    │      ★ newObj.size 来自压缩完成后元数据中的真实原始大小
    │      ★ .bin 文件名不含大小，不需要修正
    │
    └─ 6. return newObj
```

---

## 六、核心结论

1. **旧对象定位**：PutStream 用 `NewObject` 获取完整旧对象，但只读取 `meta.Mode` 做条件判断，只删除旧数据文件。旧对象的元数据引用（`mo`）**没有被复用**，这与 `Object.Update` 中通过 `o.mo.Update()` 原地复用形成鲜明对比。

2. **元数据写入方式**：PutStream 传给 `putWithCustomFunctions` 的 `putMeta` 是 `f.Fs.Put`——底层文件系统的通用 Put 方法。它对 `.json` 文件执行的是**同名覆盖**，而非通过 `o.mo.Update()` 的**原地内容替换**。前者可能中断底层服务端的版本链，后者则保留。

3. **根本原因**：PutStream 的设计是"**全新上传**"模式——数据走流式创建，元行走同名覆盖，旧对象仅做善后清理。而 Object.Update 是"**增量更新**"模式——尽量复用旧引用原地替换内容，只在文件名必须变化时才走"先建后删"。两种模式的设计意图不同：PutStream 追求简单可靠，Update 追求保留版本历史。
