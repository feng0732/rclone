# PutStream 元数据复用与版本历史：事实与推断边界

---

## 一、已确认的硬事实（compress 包代码内可直接验证）

以下结论全部来自 compress 包自身代码，不依赖任何外部假设。

### 事实 1：`f.Fs` 是嵌入的底层真实文件系统，不是 compress 自身

参考 [compress.go#L177-L178](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L177-L178) 和 [compress.go#L237-L244](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L237-L244)：

```go
type Fs struct {
    fs.Fs   // 嵌入字段，NewFs 中被赋值为 wrappedFs
    ...
}
```

```go
f := &Fs{
    Fs: wrappedFs,   // wrappedFs 是底层真实文件系统
    ...
}
```

因此，`f.Fs.Put` 指的是底层真实 Fs 的 Put 方法，例如 S3、本地文件系统等后端的 Put，不是 compress.Fs 自己的 Put。

### 事实 2：PutStream 中 putMeta 实参硬编码为 `f.Fs.Put`

参考 [compress.go#L750](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L750)：

```go
newObj, err := f.putWithCustomFunctions(
    ctx, in, src, options,
    f.Fs.Features().PutStream,
    f.Fs.Put,          // 元数据写入使用底层 Fs 的 Put
    compressible, mimeType,
)
```

### 事实 3：`putMetadata` 直接调用传入的 put，不做任何闭包包装

参考 [compress.go#L659-L679](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L659-L679)：

```go
func (f *Fs) putMetadata(
    ctx context.Context, meta *ObjectMetadata, src fs.ObjectInfo,
    options []fs.OpenOption, put putFn,
) (mo fs.Object, err error) {
    data, _ := json.Marshal(meta)
    metaReader := bytes.NewReader(data)

    mo, err = put(
        ctx,
        metaReader,
        f.wrapInfo(src, makeMetadataName(src.Remote()), int64(len(data))),
        options...,
    )
    return mo, nil
}
```

在 PutStream 场景下，此处的 `put` 就是 `f.Fs.Put`。传入的 ObjectInfo 中 Remote 字段为 `makeMetadataName(src.Remote())`，即 `{remote}.json`。

### 事实 4：PutStream 中不存在 updateMeta 闭包，也未引用 `oldObj.mo`

PutStream 函数体内对 oldObj 的全部使用只有四处：

- 第 740 行：`NewObject` 获取 oldObj
- 第 744 行：`found := err == nil`
- 第 757 行：读取 `oldObj.(*Object).meta.Mode`
- 第 758 行：调用 `oldObj.(*Object).Object.Remove(ctx)`

`oldObj.mo` 从未被读取、赋值或调用任何方法。

### 事实 5：Object.Update 中显式构造了 updateMeta 闭包并传入

参考 [compress.go#L1125-L1128](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1125-L1128)：

```go
updateMeta := func(
    ctx context.Context, in io.Reader, src fs.ObjectInfo, options ...fs.OpenOption,
) (fs.Object, error) {
    return o.mo, o.mo.Update(ctx, in, src, options...)
}
```

随后在 [compress.go#L1140](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1140) 和 [compress.go#L1156](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1156) 中，`updateMeta` 作为 putMeta 参数传入 `putWithCustomFunctions`。

### 事实 6：三条路径的 putMeta 实参对照

| 路径 | putMeta 实参 |
|------|-------------|
| Fs.Put（新建对象） | `f.Fs.Put`，底层 Fs 的 Put |
| Object.Update | `updateMeta` 闭包，内部调用 `o.mo.Update` |
| Fs.PutStream | `f.Fs.Put`，底层 Fs 的 Put |

### 事实 7：`f.Fs.Put` 与 `o.mo.Update` 都匹配 `putFn` 签名

```go
type putFn func(
    ctx context.Context, in io.Reader, src fs.ObjectInfo, options ...fs.OpenOption,
) (fs.Object, error)
```

---

## 二、合理推断（超出 compress 包代码边界，无法 100% 确认）

以下结论依赖底层 Fs 的具体实现，S3、本地文件系统等不同后端的行为可能不同。

### 推断 1：底层 `Fs.Put` 对同名文件的具体覆盖方式

从 rclone 后端的通用接口契约来看，Put 的语义通常是「上传指定路径的文件，若已存在则覆盖」。但「覆盖」的内部实现方式 compress 包无法确认，可能是：

- S3 后端：PutObject 幂等覆盖，可能产生新版本
- 本地文件系统：`os.Create` 截断重写，不保留历史
- 某些后端：内部先 `NewObject` 再 `Update`

具体是哪种，compress 包代码无法给出确定答案。

### 推断 2：`o.mo.Update` 与 `f.Fs.Put` 在版本历史保留上的差异

compress 包可确认的只有调用层级不同：
- `o.mo.Update` 走的是对象级 Update 方法，由 NewObject 返回的具体对象类型实现
- `f.Fs.Put` 走的是文件系统级 Put 方法

对支持版本控制的后端，例如 S3：
- 对象级 Update 更可能保持同一个对象 ID，只替换内容，版本链延续
- 文件系统级 Put 更可能创建新对象，对象 ID 可能变化

这只是合理推断，compress 包代码中没有明确证据。

### 推断 3：PutStream 选择 `f.Fs.Put` 而非 updateMeta 闭包的可能原因

代码事实是 PutStream 没有构造 updateMeta 闭包。可能的原因包括：

1. PutStream 是 Fs 级别的方法，不像 Object.Update 那样天然持有旧 Object 的接收者引用。
2. 即使构造闭包引用 `oldObj.mo`，当旧对象不存在时闭包会遇到 nil，需要额外的分支处理。
3. PutStream 场景涉及临时错误文件名、后续重命名等复杂流程，保证正确性优先级高于保留版本历史。

这些是基于代码结构的合理推测，代码中没有注释明示作者选择 `f.Fs.Put` 的具体动机。

---

## 三、版本历史影响：判断边界

### 从 compress 包代码可以确认的内容

| 方面 | compress 包代码可确认的结论 |
|------|---------------------------|
| 元数据文件对象引用复用 | Object.Update 路径复用了 `o.mo` 旧引用；PutStream 路径未复用任何旧元数据对象引用 |
| 调用方法 | Object.Update 在旧对象上调用 `o.mo.Update(ctx, ...)`；PutStream 在 Fs 上调用 `f.Fs.Put(ctx, ...)` |
| 方法语义层级 | Object.Update 是对象级方法；PutStream 是文件系统级方法 |

### 从 compress 包代码无法确认的内容

| 方面 | 无法确认的原因 |
|------|----------------|
| PutStream 是否为原地内容替换 | 取决于底层 `Fs.Put` 的具体实现 |
| PutStream 是否中断版本历史链 | 取决于底层 `Fs.Put` 的具体实现 |
| Object.Update 是否一定保留版本链 | 取决于底层 Object.Update 的具体实现 |

compress 包的职责只是调用对应的方法，版本链是否保留由后端实现决定，compress 包代码无法对此给出确定结论。

---

## 四、当前实现的元数据写入路径

### PutStream 的元数据写入路径

```
PutStream
    │
    └─ putWithCustomFunctions(putMeta = f.Fs.Put)
         │
         └─ putMetadata(meta, src, put = f.Fs.Put)
              │
              └─ f.Fs.Put(ctx, metaReader, ObjectInfo{Remote = "xxx.json"}, options)
                   │
                   └─ 底层真实文件系统的 Put 方法
```

### Object.Update 的元数据写入路径

```
Object.Update
    │
    ├─ updateMeta 闭包 = func(...) { return o.mo, o.mo.Update(...) }
    │
    └─ putWithCustomFunctions(putMeta = updateMeta 闭包)
         │
         └─ putMetadata(meta, src, put = updateMeta)
              │
              └─ updateMeta(ctx, metaReader, ObjectInfo{Remote = "xxx.json"}, options)
                   │
                   └─ o.mo.Update(ctx, metaReader, ...)
                        │
                        └─ 旧元数据对象上的 Update 方法
```

### 两条路径的核心差异

| 维度 | PutStream | Object.Update |
|------|-----------|----------------|
| 调用入口 | `f.Fs.Put`，底层 Fs 级 Put | `o.mo.Update`，已有元数据对象级 Update |
| 旧对象引用 | 未引用 `oldObj.mo` | 持有 `o.mo` |
| ObjectInfo Remote | wrapInfo 新构造，值为 `xxx.json` | wrapInfo 新构造，值为 `xxx.json` |
| putFn 签名匹配 | 匹配 | 匹配 |

---

## 五、总结

- **事实**：PutStream 中 `oldObj.mo` 从未被读取或调用任何方法，也从未构造 updateMeta 闭包。Object.Update 中构造了 updateMeta 闭包并传入。两者调用的是不同层级的方法，分别是 Fs.Put 和 o.mo.Update。

- **推断**：在支持版本控制的后端，Object.Update 路径比 PutStream 更可能保留版本链，因为对象级 Update 通常在同一对象上替换内容，而文件系统级 Put 更可能创建新对象。

- **边界**：版本历史是否保留最终取决于底层 Fs.Put 和 Object.Update 的具体实现，compress 包代码无法给出确定结论。
