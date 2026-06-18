# PutStream 元数据复用与版本历史：事实与推断边界

## 一、已确认的硬事实（代码层面可 100% 确认）

以下全部来自 compress 包自身代码可直接验证，无任何假设。

### 事实 1：`f.Fs` 是嵌入的**底层真实 Fs，非压缩 Fs

[Fs 结构 L177-L178](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L177-L178)：

```go
type Fs struct {
    fs.Fs   // ← 嵌入字段，即 wrappedFs（NewFs 中被赋值
    ...
}
```

[NewFs 中 L237-L244](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L237-L244)：

```go
f := &Fs{
    Fs:          wrappedFs,   // ← wrappedFs 是底层真实文件系统
    ...
}
```

因此 `f.Fs.Put` = 指的是**底层真实 Fs 的 Put 方法**（如 S3、本地文件系统等），**不是 compress.Fs 的 Put。

### 事实 2：PutStream 传给 putMeta 实参是 `f.Fs.Put`

[PutStream L750](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L750)：

```go
newObj, err := f.putWithCustomFunctions(
    ctx, in, src, options,
    f.Fs.Features().PutStream,
    f.Fs.Put,          // ← 硬编码：元数据写入使用底层 Fs 的 Put
    compressible, mimeType,
)
```

### 事实 3：`putMetadata` 中直接调用传入的 `put` 参数（即 `f.Fs.Put），无闭包包装

[putMetadata L659-L679](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L659-L679)：

```go
func (f *Fs) putMetadata(ctx context.Context, meta *ObjectMetadata, src fs.ObjectInfo, options []fs.OpenOption, put putFn) (mo fs.Object, err error) {
    data, _ := json.Marshal(meta)
    metaReader := bytes.NewReader(data)

    // 直接调用传入的 put，没有任何闭包或对旧对象的引用
    mo, err = put(
        ctx,
        metaReader,
        f.wrapInfo(src, makeMetadataName(src.Remote()), int64(len(data))),
        options...,
    )
    ...
    return mo, nil
}
```

`put 就是 `f.Fs.Put`（底层 Fs.Put），传入的 ObjectInfo 中 Remote 是 `makeMetadataName(src.Remote())` = `{remote}.json`。

### 事实 4：PutStream 代码中**没有构造任何 updateMeta 闭包

全文搜索 PutStream 函数体，不存在类似以下形态的代码：

```
不存在的语句：
- 不存在 `func(ctx, _ := func(...) { ... o.mo.Update ... }
- 不存在对 oldObj.mo 的任何引用（除了 NewObject 返回后，仅在 L757 读取了 oldObj.meta.Mode 以外
```

PutStream 中对 oldObj 的唯一使用是：
- L740 `NewObject 获取 oldObj
- L744 `found := err == nil`
- L757 `oldObj.(*Object).meta.Mode
- L758 `oldObj.(*Object).Object.Remove(ctx)

`oldObj.mo` 从未被读取、赋值或调用任何方法。

### 事实 5：Object.Update 中**显式构造了 updateMeta 闭包并传入

[Object.Update L1125-L1128](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1125-L1128)：

```go
updateMeta := func(ctx context.Context, in io.Reader, src fs.ObjectInfo, options ...fs.OpenOption) (fs.Object, error) {
    return o.mo, o.mo.Update(ctx, in, src, options...)
}
```

然后 [L1140](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1140) 和 [L1156](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1156) 将 `updateMeta` 作为 putMeta 参数传入。

### 事实 6：两种路径写入元数据的 putMeta 实参对照表（硬事实）

| 路径 | putMeta 实参 |
|------|------------|
| Put（对象不存在时的新建 | `f.Fs.Put`（底层 Fs.Put |
| Object.Update（路径 A/B | `updateMeta` 闭包 → `o.mo.Update` |
| PutStream | `f.Fs.Put`（底层 Fs.Put |

### 事实 7：putFn 的签名

```go
type putFn func(ctx context.Context, in io.Reader, src fs.ObjectInfo, options ...fs.OpenOption) (fs.Object, error)
```

底层 Fs.Put 与 o.mo.Update 的签名都匹配 putFn。

---

## 二、合理推断（代码未明示、但基于上下文无法从 compress 包内部不能完全确认

以下结论**超出 compress 包代码边界，依赖底层 Fs 的实现，不同后端（S3 vs 本地文件系统）行为可能不同。

### 推断 1：底层 `Fs.Put` 对同名文件的具体语义

**compress 包无法确认的是：底层 `f.Fs.Put = 底层真实文件系统的 Put。

**合理推断**：大多数 rclone 后端的通用约定（fs 接口契约中，Put 的语义是「上传指定路径的文件，如果已存在则覆盖。但"覆盖"的具体实现方式是以下哪种——是先删旧再上传，还是对旧对象调用 Update，compress 包代码无法确认。

例如：
- S3 后端：PutObject 是幂等覆盖，可能产生新版本（版本 ID 变化但属于 S3 的版本历史
- 本地文件系统：os.Create 截断重写，不保留历史
- 某些后端：内部可能先 NewObject → Update

### 推断 2：`o.mo.Update 与 底层 Fs.Put 在"是否保留版本历史差异

**compress 包可确认的是：
- `o.mo.Update 走的是底层 fs.Object 已有对象的 Update 方法（由 NewObject 返回的具体对象类型）
- 底层 Fs.Put 走的是底层 fs.Fs 的 Put 方法

**合理推断**：对于支持版本控制的后端（如 S3）：
- `o.mo.Update` 可能内部实现可能走同一个对象 ID 不变，只更新内容，版本链延续
- `f.Fs.Put 可能创建新对象、或覆盖同名对象，但对象 ID 可能变化

但这是推断，compress 包代码里没有明确证据。

### 推断 3：为什么 PutStream 选择 `f.Fs.Put 而非 updateMeta 闭包

代码事实：PutStream 没有构造 updateMeta 闭包。

**合理推断**（作者可能的原因（多个可能，无法确认哪一个：

1. PutStream 作为 Fs 级别的方法，不是 Object 级别的方法，没有天然持有的旧 Object 引用（不像 Object.Update 的接收者 o）。

2. 即使构造了闭包引用 oldObj.mo，如果 oldObj 不存在（found=false）则闭包会 panic，需要额外 nil 处理逻辑

3. PutStream 使用场景是流式上传，大小未知、临时错误文件名等复杂流程，优先保证正确性高于保留版本历史不是优先考虑

这只是推断，代码中没有注释明示作者选择 `f.Fs.Put 的具体动机。

---

## 三、版本历史影响：能确定的边界

### 能确定的（基于 compress 包内部代码：

| 方面 | 能否从 compress 代码可确认的判断边界 |
|------|------------------------------------|
| **元数据文件对象引用复用 | Object.Update 路径**复用了** `o.mo 旧引用；PutStream **未复用**任何旧元数据对象引用 |
| **调用的方法不同 | Object.Update：在旧对象上调用 `o.mo.Update(ctx, ...)`；PutStream：在 Fs 上调用 `f.Fs.Put(ctx, ...)` |
| **方法语义层级不同 | Object.Update：对象级方法；PutStream：文件系统级方法 |
| **PutStream：是否"原地内容替换 | 无法从 compress 包确认 | 取决于底层 Fs.Put 具体实现 |
| **PutStream 是否中断版本链 | 无法从 compress 包确认 | 取决于底层 Fs.Put 具体实现 |

### 版本历史影响：只能推断（不能确认的

关于"版本历史保留与否，compress 包代码没有任何一个后端（S3/本地），后端实现相关，compress 包都能做的只是调用对应的方法（Put/Update，但具体版本链是否保留是后端自己的事。

compress 包作者在 [Put 的作者在 Fs.Put 注释中写了：

```go
// Put in to the remote path with the modTime given of the given size
//
// May create the object even if it returns an error - if so
// will return the object and the error, otherwise will return
// nil and the error
```

注释说了 Put 实现差异，没提版本历史。

---

## 四、当前实现的元数据写入选择：事实总结

### PutStream 的元数据写入路径（事实）

```
PutStream
    │
    └─ putWithCustomFunctions(putMeta = f.Fs.Put)
         │
         └─ putMetadata(meta, src, put=f.Fs.Put)
              │
              └─ f.Fs.Put(ctx, metaReader, ObjectInfo{Remote="xxx.json"}, options)
                   │
                   └─ 底层真实文件系统的 Put
                        (语义：对 "xxx.json"
```

### Object.Update 的元数据写入路径（事实）

```
Object.Update
    │
    ├─ updateMeta 闭包 = func(...) { return o.mo, o.mo.Update(...) }
    │
    └─ putWithCustomFunctions(putMeta = updateMeta 闭包)
         │
         └─ putMetadata(meta, src, put=updateMeta)
              │
              └─ updateMeta(ctx, metaReader, ObjectInfo{Remote="xxx.json"}, options)
                   │
                   └─ o.mo.Update(ctx, metaReader, ...)
                        │
                        └─ 旧元数据对象上的 Update 方法
```

### 两条路径的核心差异（事实）

| 维度 | PutStream | Object.Update |
|------|-----------|----------------|
| 调用入口 | `f.Fs.Put（底层 Fs 级 Put | `o.mo.Update（已有元数据对象级 Update |
| 旧对象引用 | 未引用 oldObj.mo | 持有 o.mo |
| ObjectInfo Remote | 由 wrapInfo 新构造 Remote="xxx.json" | 由 wrapInfo 新构造 Remote="xxx.json" |
| 返回的 Remote 参数 | 两者 ObjectInfo Remote 值相同（都是 `xxx.json |
| 方法签名匹配 putFn 签名 | 都符合 putFn |

### 为什么 PutStream 没有使用 updateMeta 闭包

```

### 五、边界结论

- **事实**：PutStream 中 `oldObj.mo` 从未被读取或调用方法调用，也从未构造 updateMeta 闭包。Object.Update 中构造了 updateMeta 闭包并传入。两者调用的是不同（Fs.Put vs o.mo.Update）。

- **推断**：两者在支持版本控制的后端（如 S3），Object.Update 路径可能比 PutStream 更可能保留版本链，因为对象级 Update 通常是在同一对象上内容替换，而 Fs 级 Put 更可能是新建对象。

- **边界**：上述"版本历史保留与否最终取决于底层 Fs.Put 和 Object.Update 的具体实现，compress 包代码无法给出确定结论。
