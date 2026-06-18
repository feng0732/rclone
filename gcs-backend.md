# GCS 后端代码结构分析

本文档梳理 Google Cloud Storage (GCS) 后端的代码结构，重点说明 bucket 操作、对象元数据和错误转换如何落到 rclone 的通用 fs 抽象上。

## 1. 代码位置与整体结构

核心文件：[backend/googlecloudstorage/googlecloudstorage.go](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/backend/googlecloudstorage/googlecloudstorage.go)

```
backend/googlecloudstorage/
├── googlecloudstorage.go       # 主实现文件（~1530 行）
└── googlecloudstorage_test.go  # 测试文件
```

该后端实现了以下核心类型：

| 类型 | 作用 | 对应通用接口 |
|------|------|-------------|
| `Fs` | 代表一个 GCS 远程根（bucket 或 bucket+path） | `fs.Fs`, `fs.Copier`, `fs.PutStreamer`, `fs.ListRer`, `fs.ListPer` |
| `Object` | 代表 GCS 中的一个对象 | `fs.Object`, `fs.MimeTyper` |
| `Options` | 配置选项结构体 | — |

接口完整性校验在文件末尾：
```go
var (
    _ fs.Fs          = &Fs{}
    _ fs.Copier      = &Fs{}
    _ fs.PutStreamer = &Fs{}
    _ fs.ListRer     = &Fs{}
    _ fs.ListPer     = &Fs{}
    _ fs.Object      = &Object{}
    _ fs.MimeTyper   = &Object{}
)
```

GCS 使用已标记 deprecated 的 `google.golang.org/api/storage/v1` API。

---

## 2. Bucket 操作 → 通用 fs 抽象

GCS 是 bucket-based 后端（`features.BucketBased = true`），其路径模型为 `bucket/path/to/object`。rclone 通过 `lib/bucket` 包提供通用工具，GCS 后端在此基础上将 bucket 操作映射到 `fs.Fs` 的目录语义上。

### 2.1 路径拆分：bucket ↔ directory

路径解析由 `Fs.split()` 和 `Fs.setRoot()` 完成，依赖 [lib/bucket/bucket.go](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/lib/bucket/bucket.go) 中的 `Split` 和 `Join` 函数：

```go
// [googlecloudstorage.go:L504-L515]
func (f *Fs) split(rootRelativePath string) (bucketName, bucketPath string) {
    bucketName, bucketPath = bucket.Split(bucket.Join(f.root, rootRelativePath))
    // ... DirectoryMarkers 特殊处理
    return f.opt.Enc.FromStandardName(bucketName), f.opt.Enc.FromStandardPath(bucketPath)
}

func (f *Fs) setRoot(root string) {
    f.root = parsePath(root)
    f.rootBucket, f.rootDirectory = bucket.Split(f.root)
}
```

| rclone 路径 | `rootBucket` | `rootDirectory` | 含义 |
|------------|-------------|----------------|------|
| `""` | `""` | `""` | GCS 根，可列出所有 buckets |
| `"my-bucket"` | `"my-bucket"` | `""` | 整个 bucket |
| `"my-bucket/sub/dir"` | `"my-bucket"` | `"sub/dir"` | bucket 内的子目录 |

### 2.2 Bucket 列表 → `Fs.ListP` / `Fs.ListR`

当 `bucket == ""`（即处于 GCS 根层级）时，列出 buckets 被映射为"列出根目录下的目录"：

```go
// [googlecloudstorage.go:L859-L883]
func (f *Fs) ListP(ctx context.Context, dir string, callback fs.ListRCallback) error {
    bucket, directory := f.split(dir)
    if bucket == "" {
        if directory != "" {
            return fs.ErrorListBucketRequired
        }
        entries, err := f.listBuckets(ctx)  // 调用 GCS Buckets.List API
        // ... 将每个 bucket 转为 fs.NewDir(...)
    } else {
        err := f.listDir(ctx, bucket, directory, ...)  // 列出 bucket 内的对象
    }
}
```

`listBuckets` 通过 `storage.Buckets.List(projectNumber)` 调用 GCS API，结果通过 `fs.NewDir(bucketName, time.Time{})` 包装为通用 `fs.DirEntry`。

### 2.3 Bucket 创建 → `Fs.Mkdir`

创建目录在 bucket-based 后端的语义是"确保 bucket 存在"：

```go
// [googlecloudstorage.go:L1006-L1013]
func (f *Fs) Mkdir(ctx context.Context, dir string) (err error) {
    bucket, _ := f.split(dir)
    e := f.checkBucket(ctx, bucket)   // 检查/创建 bucket
    if e != nil {
        return e
    }
    return f.createDirectoryMarker(ctx, bucket, dir)  // 可选：创建目录标记文件
}
```

`checkBucket` → `makeBucket` 的实际创建逻辑：

```go
// [googlecloudstorage.go:L1026-L1078]
func (f *Fs) makeBucket(ctx context.Context, bucket string) (err error) {
    return f.cache.Create(bucket, func() error {
        // 1. 先尝试列出 bucket 中 1 个对象来探测是否存在
        //    （这样仅需 Storage Object Admin 角色，无需 Storage Admin）
        err = f.pacer.Call(func() (bool, error) {
            _, err = f.svc.Objects.List(bucket).MaxResults(1).Do()
            return shouldRetry(ctx, err)
        })
        if err == nil {
            return nil  // bucket 已存在
        } else if gErr, ok := err.(*googleapi.Error); ok {
            if gErr.Code != http.StatusNotFound {
                return err  // 非 404 错误直接返回
            }
        } else {
            return err
        }

        // 2. 只有探测到 404，才通过 Buckets.Insert API 创建
        // ⚠️  注意：Buckets.Insert 不是幂等操作
        //    如果第一次 Insert 成功但网络超时没收到响应，
        //    pacer.Call 低层重试时会得到 409 Conflict（bucket 已存在）错误
        bucket := storage.Bucket{...}
        return f.pacer.Call(func() (bool, error) {
            _, err = f.svc.Buckets.Insert(project, &bucket).Do()
            return shouldRetry(ctx, err)
        })
    }, nil)
}
```

#### 幂等性分析

`makeBucket` 的整体流程是"探测-创建"两阶段：
1. **Objects.List 探测**：完全幂等，重复调用无副作用，使用 `pacer.Call` 低层重试安全
2. **Buckets.Insert 创建**：**非幂等**，成功后重试会返回 409 Conflict 错误
   - `shouldRetry` 不会重试 409（只重试 5xx 和限流）
   - 但如果第一次 Insert 成功响应丢失，第二次尝试 Insert 会失败
   - `bucket.Cache` 的存在降低了并发冲突概率，但无法消除网络超时导致的重试冲突

### 2.4 Bucket 删除 → `Fs.Rmdir`

`Rmdir` 仅在删除整个 bucket（`directory == ""`）时调用 GCS 的 Buckets.Delete：

```go
// [googlecloudstorage.go:L1092-L1119]
func (f *Fs) Rmdir(ctx context.Context, dir string) (err error) {
    bucket, directory := f.split(dir)
    // 1. 可选：删除目录标记文件（DirectoryMarkers 模式）
    if bucket == "" || directory != "" {
        return nil  // 非 bucket 根目录，不调用 GCS 删除
    }
    return f.cache.Remove(bucket, func() error {
        return f.pacer.Call(func() (bool, error) {
            deleteBucket := f.svc.Buckets.Delete(bucket).Context(ctx)
            // ...
            err = deleteBucket.Do()
            return shouldRetry(ctx, err)
        })
    })
}
```

### 2.5 Bucket 状态缓存：`bucket.Cache`

所有 bucket 创建/删除操作都通过 `bucket.Cache` 进行去重和状态跟踪：

```go
type Fs struct {
    cache *bucket.Cache  // [googlecloudstorage.go:L422]
}
```

`bucket.Cache`（见 [lib/bucket/bucket.go:L62-L196](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/lib/bucket/bucket.go#L62-L196)）提供：
- `Create(bucket, createFn, existsFn)`：带互斥锁的幂等创建
- `Remove(bucket, removeFn)`：带互斥锁的幂等删除
- `MarkOK(bucket)` / `MarkDeleted(bucket)`：标记状态
- `IsDeleted(bucket)`：查询状态

这避免了在同一 bucket 上并发创建/删除时的重复 API 调用。

### 2.6 Bucket 操作映射总览

| 通用 fs 操作 | GCS 后端实现 | 调用的 GCS API |
|-------------|-------------|----------------|
| `List("")` | `listBuckets` | `Buckets.List` |
| `List("bucket")` | `listDir` | `Objects.List` (Delimiter="/") |
| `Mkdir("bucket")` | `makeBucket` | `Objects.List` (探测) → `Buckets.Insert` |
| `Rmdir("bucket")` | `cache.Remove` + `Buckets.Delete` | `Buckets.Delete` |
| `Mkdir("bucket/sub")` | `createDirectoryMarker` | `Objects.Insert` (空对象 trailing `/`) |

---

## 3. 对象元数据 → 通用 fs 抽象

GCS 对象的元数据通过 `Object.setMetaData()` 从 `*storage.Object` 转换，通过 `Object.Update()` 和 `Object.SetModTime()` 写回。

### 3.1 元数据读取：`setMetaData`

```go
// [googlecloudstorage.go:L1229-L1278]
func (o *Object) setMetaData(info *storage.Object) {
    o.url       = info.MediaLink
    o.bytes     = int64(info.Size)
    o.mimeType  = info.ContentType
    o.gzipped   = info.ContentEncoding == "gzip"

    // MD5: GCS 返回 base64 编码 → 转为 hex
    md5sumData, err := base64.StdEncoding.DecodeString(info.Md5Hash)
    if err != nil {
        fs.Logf(o, "Bad MD5 decode: %v", err)
    } else {
        o.md5sum = hex.EncodeToString(md5sumData)
    }

    // mtime 读取优先级：
    // 1. metadata["mtime"] (RFC3339Nano 格式，rclone 自定义)
    // 2. metadata["goog-reserved-file-mtime"] (Unix 秒，GSUtil 兼容)
    // 3. info.Updated (对象最后更新时间)
    //
    // ⚠️ 注意：前两级 mtime 解析成功后会直接 return，跳过后续 gzip 处理
    if mtimeString, ok := info.Metadata[metaMtime]; ok {
        modTime, err := time.Parse(timeFormat, mtimeString)
        if err == nil {
            o.modTime = modTime
            return                                    // ← 早返回：跳过 gzip size/md5 清零
        }
        fs.Debugf(o, "Failed to read mtime from metadata: %s", err)
    }

    if mtimeGsutilString, ok := info.Metadata[metaMtimeGsutil]; ok {
        unixTimeSec, err := strconv.ParseInt(mtimeGsutilString, 10, 64)
        if err == nil {
            o.modTime = time.Unix(unixTimeSec, 0)
            return                                    // ← 早返回：跳过 gzip size/md5 清零
        }
        fs.Debugf(o, "Failed to read GSUtil mtime from metadata: %s", err)
    }

    // 只有前两级都失败时才走到这里
    modTime, err := time.Parse(timeFormat, info.Updated)
    if err != nil {
        fs.Logf(o, "Bad time decode: %v", err)
    } else {
        o.modTime = modTime
    }

    // 若启用 Decompress 且对象是 gzip 压缩的，size 和 md5 未知
    // ⚠️ 这段代码只有在没有自定义 mtime（或自定义 mtime 解析失败）时才会执行
    if o.gzipped && o.fs.opt.Decompress {
        o.bytes = -1
        o.md5sum = ""
    }
}
```

#### mtime 早返回与 gzip 清零的交互影响

rclone 上传的对象**总是**会写入 `metadata["mtime"]` 和 `metadata["goog-reserved-file-mtime"]`（见 `metadataFromModTime`）。因此对于 rclone 自己上传的 gzip 压缩对象，`setMetaData` 在第一级就成功解析并 `return`，**永远不会执行**末尾的 `o.bytes = -1; o.md5sum = ""`。

实际行为：
- **由 rclone 上传的 gzip 对象** + `--gcs-decompress`：`bytes` 和 `md5sum` 保留为压缩后的值（错误的，因为解压后大小和哈希都会变）
- **非 rclone 上传的 gzip 对象**（无自定义 mtime）+ `--gcs-decompress`：`bytes = -1`，`md5sum = ""`（正确的行为）

只有当对象通过第三方工具上传、没有写入自定义 mtime metadata 时，才能正确进入 gzip 清零分支。

### 3.2 元数据写入：`metadataFromModTime` 和 `Update`

上传对象时，modTime 通过 GCS 用户自定义 metadata 持久化：

```go
// [googlecloudstorage.go:L1336-L1341]
func metadataFromModTime(modTime time.Time) map[string]string {
    metadata := make(map[string]string, 1)
    metadata[metaMtime]           = modTime.Format(timeFormat)       // RFC3339Nano
    metadata[metaMtimeGsutil]     = strconv.FormatInt(modTime.Unix(), 10) // Unix 秒
    return metadata
}
```

在 `Object.Update()`（上传）中，除了 modTime，还处理 `fs.OpenOption` 传递的 HTTP Header：

```go
// [googlecloudstorage.go:L1454-L1482]
for _, option := range options {
    key, value := option.Header()
    switch strings.ToLower(key) {
    case "cache-control":
        object.CacheControl = value
    case "content-disposition":
        object.ContentDisposition = value
    case "content-encoding":
        object.ContentEncoding = value
    case "content-language":
        object.ContentLanguage = value
    case "content-type":
        object.ContentType = value
    case "x-goog-storage-class":
        object.StorageClass = value
    default:
        // x-goog-meta-* 前缀 → 用户自定义 metadata
        if strings.HasPrefix(lowerKey, "x-goog-meta-") {
            object.Metadata[metaKey] = value
        }
    }
}
```

### 3.3 ModTime 修改：`SetModTime`

GCS 不支持直接 PATCH 元数据（需要过高权限），因此通过"复制对象到自身"的方式实现：

```go
// [googlecloudstorage.go:L1344-L1377]
func (o *Object) SetModTime(ctx context.Context, modTime time.Time) (err error) {
    object, err := o.readObjectInfo(ctx)  // 读取现有 metadata
    object.Metadata[metaMtime]       = modTime.Format(timeFormat)
    object.Metadata[metaMtimeGsutil] = strconv.FormatInt(modTime.Unix(), 10)

    // 通过 Objects.Copy 将对象复制到自身，触发 metadata 更新
    copyObject := o.fs.svc.Objects.Copy(bucket, bucketPath, bucket, bucketPath, object)
    newObject, err = copyObject.Do()
    o.setMetaData(newObject)
    return nil
}
```

### 3.4 元数据映射总览

| 通用 fs 接口 | GCS 对象字段 | 转换方式 |
|-------------|-------------|---------|
| `Object.Size()` | `storage.Object.Size` | 直接赋值；**gzip+Decompress 时行为取决于是否有自定义 mtime**：<br>• 有自定义 mtime（rclone 上传的对象）→ 保留压缩后大小（Bug）<br>• 无自定义 mtime → 置为 -1（正确表示未知） |
| `Object.Hash(MD5)` | `storage.Object.Md5Hash` | base64 → hex；**gzip+Decompress 时行为取决于是否有自定义 mtime**：<br>• 有自定义 mtime → 保留压缩后 MD5（Bug）<br>• 无自定义 mtime → 置为 `""`（正确表示未知） |
| `Object.ModTime()` | `metadata["mtime"]` / `metadata["goog-reserved-file-mtime"]` / `Updated` | 三级 fallback，前两级解析成功会 `return`，跳过后续 gzip 清零 |
| `Object.MimeType()` | `storage.Object.ContentType` | 直接赋值 |
| `Object.SetModTime()` | 写入 metadata → `Objects.Copy` | 复制自身 |
| `Fs.Put()` / `Object.Update()` | 构建 `storage.Object{Metadata, ContentType, StorageClass, ...}` | 上传时传入 |
| `OpenOption.Header()` | 多种字段 + `x-goog-meta-*` | switch 分发 |

---

## 4. 错误转换机制

错误转换分布在三个层面：**Pacer 调用方式选择**（`Call` vs `CallNoRetry`）、**重试判定**（`shouldRetry`）、**语义错误映射**（将 GCS 特定错误转为 `fs.Error*`）。

### 4.1 Pacer 调用方式：`Call` vs `CallNoRetry`

GCS 后端通过 [fs.NewPacer](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/fs/pacer.go#L23-L37) 创建 Pacer，**重试次数来自全局配置** `ci.LowLevelRetries`，而非 lib/pacer 裸库默认值：

```go
// [fs/pacer.go:L23-L37]
func NewPacer(ctx context.Context, c pacer.Calculator) *Pacer {
    ci := GetConfig(ctx)
    retries := max(ci.LowLevelRetries, 1)     // ← 来自 --low-level-retries 全局 flag
    maxConnections := max(ci.MaxConnections, 0)
    p := &Pacer{
        Pacer: pacer.New(
            pacer.RetriesOption(retries),     // 覆盖裸库默认 retries=3
            pacer.CalculatorOption(c),
            // ...
        ),
    }
    return p
}
```

GCS 在 `NewFs` 中创建 Pacer（见 [googlecloudstorage.go:L587](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/backend/googlecloudstorage/googlecloudstorage.go#L587)）：
```go
f.pacer = fs.NewPacer(ctx, pacer.NewS3(pacer.MinSleep(minSleep)))
```

两种调用模式差异：

| 方法 | 底层行为 | 使用场景 |
|------|---------|---------|
| `pacer.Call(fn)` | `p.call(fn, retries)`，**retries = `--low-level-retries` 配置值**，低层循环重试 + 指数退避 | 读操作（List、Get、Copy）和写操作中幂等的 Delete |
| `pacer.CallNoRetry(fn)` | `p.call(fn, 1)`，**强制只执行 1 次**，忽略 LowLevelRetries 配置；若需重试则把错误包装为 `fserrors.RetryError` 抛给上层 | **上传操作**（Objects.Insert 的 Media 上传） |

`CallNoRetry` 的实现（见 [lib/pacer/pacer.go:L255-L257](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/lib/pacer/pacer.go#L255-L257)）：
```go
func (p *Pacer) CallNoRetry(fn Paced) error {
    return p.call(fn, 1)  // 只尝试 1 次，忽略全局配置的 LowLevelRetries
}
```

当 `fn` 返回 `(true, err)`（表示希望重试）时，`call(..., 1)` 不会再循环，而是将 err 包装为 `fserrors.RetryError`（实现了 `Retrier` 接口）返回，由上层（`fs/operations` 的 sync/copy 逻辑）决定是否高层重试。

**为什么上传用 CallNoRetry：** `Objects.Insert(...).Media(in)` 从 `io.Reader` 流式上传，Reader 消费后不可重放，低层重试会读到空数据或损坏数据，因此必须把重试交给上层（上层会重新打开源文件 Reader）。

### 4.2 重试判定：`shouldRetry`

所有 API 调用都通过 `shouldRetry` 决定是否应重试，它作为 `Paced` 函数的返回值 `(again, err)` 中的 `again`：

```go
// [googlecloudstorage.go:L470-L494]
func shouldRetry(ctx context.Context, err error) (again bool, errOut error) {
    if fserrors.ContextError(ctx, &err) {  // context 取消/超时 → 不重试
        return false, err
    }
    again = false
    if err != nil {
        if fserrors.ShouldRetry(err) {     // 通用网络错误重试（见 fserrors.ShouldRetry）
            again = true
        } else {
            switch gerr := err.(type) {
            case *googleapi.Error:
                if gerr.Code >= 500 && gerr.Code < 600 {
                    again = true            // 所有 5xx 服务器错误
                } else if len(gerr.Errors) > 0 {
                    reason := gerr.Errors[0].Reason
                    if reason == "rateLimitExceeded" || reason == "userRateLimitExceeded" {
                        again = true        // 限流错误
                    }
                }
            }
        }
    }
    return again, err
}
```

通用 `fserrors.ShouldRetry`（见 [fs/fserrors/error.go:L400-L434](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/fs/fserrors/error.go#L400-L434)）检查：
- `NoLowLevelRetrier` 标记 → 不重试
- `Timeout()` / `Temporary()` 接口 → 重试
- `io.EOF` / `io.ErrUnexpectedEOF` → 重试
- 错误字符串包含特定短语（如 "use of closed network connection"）→ 重试

### 4.3 GCS 各操作的 Pacer 调用方式

| 操作 | 代码位置 | 调用方式 | 原因 |
|------|---------|---------|------|
| `Objects.List`（列目录） | `list()` L689 | `pacer.Call` | 幂等读，可安全低层重试 |
| `Objects.Get`（读元数据） | `readObjectInfo()` L1288 | `pacer.Call` | 幂等读 |
| `Buckets.List`（列 bucket） | `listBuckets()` L814 | `pacer.Call` | 幂等读 |
| `Buckets.Insert`（建 bucket） | `makeBucket()` L1065 | `pacer.Call` | **非幂等**；仅当 `Objects.List` 探测返回 404 时才调用 Insert；若 Insert 成功但响应丢失，低层重试会得到 409 Conflict 错误 |
| `Buckets.Delete`（删 bucket） | `Rmdir()` L1110 | `pacer.Call` | 幂等删除 |
| `Objects.Delete`（删对象） | `Remove()` L1507 | `pacer.Call` | 幂等删除 |
| `Objects.Copy`（改 mtime） | `SetModTime()` L1360 | `pacer.Call` | 幂等写 |
| `Objects.Rewrite`（服务端复制） | `Copy()` L1168 | `pacer.Call` | 幂等写 |
| **`Objects.Insert.Media`（上传）** | **`Update()` L1484** | **`pacer.CallNoRetry`** | **Reader 流不可重放，低层重试会读空数据** |
| HTTP GET（初始下载请求） | `Open()` L1409 | `pacer.Call` | **内层 Pacer 请求重试**（`LowLevelRetries` 次），仅对"发请求→收响应"阶段生效；流已建立后的读取失败由外层 ReOpen Range 续读处理 |

### 4.4 HTTP 状态码 → 语义错误

将 GCS 返回的 `*googleapi.Error` 转为 rclone 通用错误常量（定义于 [fs/fs.go:L27-L56](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/fs/fs.go#L27-L56)）：

#### 在 `list()` 中：目录不存在
```go
// [googlecloudstorage.go:L693-L699]
if gErr, ok := err.(*googleapi.Error); ok {
    if gErr.Code == http.StatusNotFound {
        err = fs.ErrorDirNotFound
    }
}
```

#### 在 `readObjectInfo()` 中：对象不存在
```go
// [googlecloudstorage.go:L1296-L1302]
if gErr, ok := err.(*googleapi.Error); ok {
    if gErr.Code == http.StatusNotFound {
        return nil, fs.ErrorObjectNotFound
    }
}
```

#### 在 `NewFs()` 中：根路径指向一个文件
```go
// [googlecloudstorage.go:L612-L632]
// 如果 root 路径对应一个对象（而非目录前缀），则返回 fs.ErrorIsFile
// 并将 Fs root 调整为父目录
return f, fs.ErrorIsFile
```

#### 在 `ListP()` 中：未指定 bucket
```go
// [googlecloudstorage.go:L862-L865]
if bucket == "" {
    if directory != "" {
        return fs.ErrorListBucketRequired
    }
}
```

### 4.5 错误映射总览

| GCS 错误 / 条件 | rclone 通用错误 | 位置 |
|-----------------|----------------|------|
| `googleapi.Error{Code: 404}`（list 目录） | `fs.ErrorDirNotFound` | `list()` L696 |
| `googleapi.Error{Code: 404}`（get 对象） | `fs.ErrorObjectNotFound` | `readObjectInfo()` L1299 |
| `googleapi.Error{Code: 409}`（删非空 bucket） | 透传（文字说明） | `Rmdir` |
| root 路径对应单个对象 | `fs.ErrorIsFile` | `NewFs()` L630 |
| 无 bucket 但有子路径 | `fs.ErrorListBucketRequired` | `ListP()` L864 |
| `googleapi.Error{Code: 5xx}` | 触发 `shouldRetry` → 重试 | `shouldRetry()` L481 |
| `reason: rateLimitExceeded` | 触发 `shouldRetry` → 重试 | `shouldRetry()` L486 |
| context 取消/超时 | 透传，不重试 | `shouldRetry()` L471 |
| 通用网络错误（EOF、连接关闭等） | 触发 `fserrors.ShouldRetry` → 重试 | `shouldRetry()` L476 |

### 4.6 Pacer 调用模式

读/写/删除操作使用低层重试的 `Call` 模式：

```go
err = f.pacer.Call(func() (bool, error) {
    result, err = f.svc.SomeOperation(...).Context(ctx).Do()
    return shouldRetry(ctx, err)  // 返回 (是否重试, 错误)
})
```

上传操作使用仅执行 1 次的 `CallNoRetry` 模式（因为流式 Reader 不可重放）：

```go
err = o.fs.pacer.CallNoRetry(func() (bool, error) {
    insertObject := o.fs.svc.Objects.Insert(bucket, &object).Media(in, ...)
    newObject, err = insertObject.Do()
    return shouldRetry(ctx, err)  // 返回 true 时，pacer 将 err 包装为 RetryError 抛给上层
})
```

Pacer 使用 `pacer.NewS3(pacer.MinSleep(10ms))` 配置，采用指数退避策略控制请求速率。

---

## 5. 关键协作组件

| 组件 | 作用 | 文件 |
|------|------|------|
| `lib/bucket` | bucket 路径拆分/拼接、状态缓存 | [lib/bucket/bucket.go](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/lib/bucket/bucket.go) |
| `fs/fserrors` | 通用错误类型、重试判定 | [fs/fserrors/error.go](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/fs/fserrors/error.go) |
| `fs/hash` | 哈希类型定义（MD5） | [fs/hash/hash.go](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/fs/hash/hash.go) |
| `fs/list` | List/ListP/ListR 辅助 | [fs/list/list.go](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/fs/list/list.go) |
| `lib/pacer` | 请求速率控制、指数退避 | [lib/pacer/pacer.go](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/lib/pacer/pacer.go) |
| `lib/oauthutil` | OAuth 认证流程 | [lib/oauthutil/renew.go](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/lib/oauthutil/renew.go) |
| `lib/encoder` | 文件名编码 | [lib/encoder/encoder.go](file:///d:/fz/0601-2/solo-dogfeeding/code/42-rclone/lib/encoder/encoder.go) |

---

## 6. 数据流示意

### 6.1 上传对象 (`Fs.Put` → `Object.Update`)
```
fs.Fs.Put(in, src)
  → Object.Update(in, src)
    ├─ mkdirParent() → Mkdir() → checkBucket() → makeBucket() [确保 bucket 存在]
    ├─ metadataFromModTime(src.ModTime())                      [构建 mtime metadata]
    ├─ 遍历 options.Header() → 填充 CacheControl/ContentType/StorageClass/x-goog-meta-*
    └─ f.svc.Objects.Insert(bucket, object).Media(in).Do()     [调用 GCS API]
        └─ f.pacer.CallNoRetry(shouldRetry)                     [速率控制 + 仅 1 次低层尝试，失败包装为 RetryError 交上层]
```

### 6.2 下载对象：两层重试机制

下载过程涉及**内外两层**不同层级的重试，分别处理 HTTP 请求建立阶段和数据流读取阶段。命名按调用栈从外到内：

```
operations.Open(src, options)          # 上层统一入口（见 fs/operations/reopen.go:L123）
  │
  ├─ 外层：ReOpen Range 续读包装（maxTries = LowLevelRetries）
  │   ├─ 初始调用 Object.Open() → ↓ 进入内层
  │   └─ 读流失败后 Range 续读：
  │       ReOpen.Read() → io.Copy 读 h.rc.Read() 出错
  │         → h.reopen()
  │             ├─ h.rangeOption.Start = h.start + h.offset     # 续传起点
  │             └─ h.src.Open(ctx, opts...) 重新建立 HTTP 连接
  │
  └─ 内层：Object.Open Pacer 请求重试（仅请求阶段）
      Object.Open(options)
        ├─ fs.FixRangeOption(options, bytes)                     [处理 Range 请求]
        ├─ gzipped && !Decompress → Accept-Encoding: gzip
        └─ f.pacer.Call(func() {                                 # LowLevelRetries 次
               f.client.Do(req)  ← 仅重试"发请求→收响应头"阶段
           })
           返回 http.Response.Body（io.ReadCloser）
```

两层重试的区别：

| 层级 | 重试什么 | 何时生效 | 重试次数 | 实现位置 |
|------|---------|---------|---------|---------|
| **外层：ReOpen Range 续读包装** | Body 读取过程中 `Read()` 返回错误（连接中断等） | Body 已开始读取后 | `LowLevelRetries` | `operations.ReOpen.Read` → `reopen()` |
| **内层：Object.Open Pacer 请求重试** | HTTP 请求建立（`client.Do` 返回错误或 5xx） | Body 未开始读取之前 | `LowLevelRetries` | `Object.Open` 内 `pacer.Call` |

为什么需要两层：
- 内层 Pacer 只能重试"发出请求→收到响应"这一完整 round-trip；一旦 `client.Do` 成功返回 `http.Response.Body`，后续从 Body 流式读取的失败 Pacer 管不到
- 外层 ReOpen 包装层追踪已读取的 `offset`，失败后自动构造 `Range: bytes=<offset>-` 续传请求重新 `Object.Open`

### 6.3 列出目录 (`Fs.ListP`)
```
fs.Fs.ListP(dir, callback)
  ├─ bucket, directory = split(dir)
  ├─ bucket == ""
  │   └─ listBuckets() → Buckets.List → fs.NewDir → callback
  └─ bucket != ""
      └─ listDir() → Objects.List(Prefix=directory, Delimiter="/")
            ├─ Prefixes → fs.NewDir (子目录)
            └─ Items    → newObjectWithInfo() → Object (文件)
```
