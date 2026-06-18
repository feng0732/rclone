# Dropbox 后端代码分析

本文档深入分析 rclone Dropbox 后端的核心架构，重点讲解**远端路径映射**、**分页列表**和**重试边界**如何进入 rclone 的**统一传输流程**。

---

## 一、整体架构概览

Dropbox 后端位于 [`backend/dropbox/`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox) 目录，核心文件包括：

| 文件 | 职责 |
|------|------|
| [`dropbox.go`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go) | 主后端实现，包含 `Fs` 和 `Object` 结构 |
| [`batcher.go`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/batcher.go) | 批量上传提交实现 |
| [`dbhash/dbhash.go`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dbhash/dbhash.go) | Dropbox 内容哈希算法 |

### 核心接口实现

Dropbox 后端实现了 rclone 的标准文件系统接口，位于 [`fs/types.go`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/fs/types.go#L16-L59) 定义的 `fs.Fs` 接口：

```go
// 接口检查 - dropbox.go:2169-2182
var (
    _ fs.Fs           = (*Fs)(nil)
    _ fs.Copier       = (*Fs)(nil)    // 服务端拷贝
    _ fs.Mover        = (*Fs)(nil)    // 服务端移动
    _ fs.ListPer      = (*Fs)(nil)    // 分页列表
    _ fs.Abouter      = (*Fs)(nil)    // 配额查询
    _ fs.Object       = (*Object)(nil)
)
```

---

## 二、远端路径映射机制

### 2.1 路径层级结构

Dropbox 后端维护三层路径表示：

```
rclone 逻辑路径 → 规范化路径 → Dropbox API 路径
     (remote)    (slashRoot)    (编码后路径)
```

**核心字段** ([`dropbox.go:351-369`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L351-L369))：

```go
type Fs struct {
    root           string   // 逻辑根路径，如 "folder/subfolder"
    slashRoot      string   // 带 "/" 前缀的根路径，如 "/folder/subfolder"
    slashRootSlash string   // 带前后 "/" 的根路径，如 "/folder/subfolder/"
    opt            Options  // 包含编码器 Enc
    // ...
}
```

### 2.2 路径初始化：`setRoot`

[`setRoot`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L689-L696) 负责路径规范化：

```go
func (f *Fs) setRoot(root string) {
    f.root = strings.Trim(root, "/")              // 去除首尾斜杠
    f.slashRoot = "/" + f.root                    // 添加前缀斜杠
    f.slashRootSlash = f.slashRoot
    if f.root != "" {
        f.slashRootSlash += "/"                   // 非空根添加后缀斜杠
    }
}
```

### 2.3 对象路径计算：`remotePath`

[`Object.remotePath()`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L1856-L1858) 计算对象的完整远端路径：

```go
func (o *Object) remotePath() string {
    return o.fs.slashRootSlash + o.remote
}
```

**示例**：
- `f.root = "work/docs"` → `f.slashRootSlash = "/work/docs/"`
- `o.remote = "report.pdf"` → `remotePath() = "/work/docs/report.pdf"`

### 2.4 路径编解码

Dropbox 使用 [`encoder.MultiEncoder`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/lib/encoder/encoder.go) 处理特殊字符：

**默认编码规则** ([`dropbox.go:283-287`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L283-L287))：
```go
Default: encoder.Base |
    encoder.EncodeBackSlash |    // 编码反斜杠 "\"
    encoder.EncodeDel |          // 编码 DEL 字符 (0x7F)
    encoder.EncodeRightSpace |   // 编码尾部空格
    encoder.EncodeInvalidUtf8,   // 编码无效 UTF-8
```

**编解码调用点**：
- **发送到 Dropbox API**：`f.opt.Enc.FromStandardPath(path)`
  - 示例：[`getMetadata`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L708) 中的 `Path: f.opt.Enc.FromStandardPath(objPath)`
  
- **从 Dropbox API 接收**：`f.opt.Enc.ToStandardPath(path)`
  - 示例：[`changeNotifyRunner`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L1687) 中的 `notifyFunc(f.opt.Enc.ToStandardPath(entryPath), entryType)`

### 2.5 导出文件路径映射（Export Path Munging）

Dropbox Paper 等文件需要导出才能访问，涉及特殊的路径映射逻辑：

[`possibleMetadatas`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L760-L792) 尝试多种路径组合：

```go
func (f *Fs) possibleMetadatas(ctx context.Context, filePath string) (ret []<-chan getMetadataResult) {
    // 1. 优先精确匹配（常规文件）
    ret = append(ret, f.getMetadataForExt(ctx, filePath, ""))
    
    // 2. 检查是否为导出路径（如 .md, .html）
    ext := exportExtension(dotted[1:])
    
    // 3. 尝试 foo.md → foo 或 foo.md → foo.paper
    base := strings.TrimSuffix(filePath, dotted)
    ret = append(ret, f.getMetadataForExt(ctx, base, ext))
    ret = append(ret, f.getMetadataForExt(ctx, base+paperExtension, ext))
    return
}
```

**映射规则**：
- `document.md` ← 实际文件 `document` 或 `document.paper`（导出为 markdown）
- `document.html` ← 实际文件 `document` 或 `document.paper`（导出为 html）

[`setMetadataForExport`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L1796-L1820) 设置导出元数据时修改 remote 路径：

```go
func (o *Object) setMetadataForExport(info *files.FileMetadata) {
    // 移除 .paper 扩展名
    o.remote = strings.TrimSuffix(o.remote, paperExtension)
    // 添加导出格式扩展名
    o.remote += "." + string(exportExt)
}
```

### 2.6 路径长度校验

[`checkPathLength`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L2091-L2104) 确保路径各部分不超过 Dropbox 限制（255 字符）：

```go
func checkPathLength(name string) error {
    for next := ""; len(name) > 0; name = next {
        // 按 "/" 分割检查每一部分
        length := utf8.RuneCountInString(name)
        if length > maxFileNameLength {
            return fserrors.NoRetryError(fs.ErrorFileNameTooLong)
        }
    }
    return nil
}
```

---

## 三、分页列表流程

### 3.1 列表接口层次

Dropbox 实现了两级列表接口：

| 接口 | 位置 | 用途 |
|------|------|------|
| `List` | [`dropbox.go:1028-1030`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L1028-L1030) | 传统列表，返回全量结果 |
| `ListP` | [`dropbox.go:1045-1148`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L1045-L1148) | 流式分页列表，支持回调 |

### 3.2 统一列表入口：`list.WithListP`

[`List`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L1028-L1030) 方法直接委托给 `list.WithListP`，这是**进入统一传输流程的第一个入口**：

```go
func (f *Fs) List(ctx context.Context, dir string) (entries fs.DirEntries, err error) {
    return list.WithListP(ctx, dir, f)
}
```

[`list.WithListP`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/fs/list/list.go) 是 rclone 的统一列表封装，它会调用后端的 `ListP` 实现。

### 3.3 `ListP` 核心实现

[`ListP`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L1045-L1148) 方法实现了完整的分页逻辑：

```go
func (f *Fs) ListP(ctx context.Context, dir string, callback fs.ListRCallback) error {
    list := list.NewHelper(callback)
    
    // 特殊模式处理：共享文件/文件夹
    if f.opt.SharedFiles { return f.listReceivedFiles(ctx, list.Add) }
    if f.opt.SharedFolders { return f.listSharedFolders(ctx, list.Add) }
    
    // 构建 Dropbox API 路径
    root := f.slashRoot
    if dir != "" {
        root += "/" + dir
    }
    
    started := false
    var res *files.ListFolderResult
    
    for {
        if !started {
            // 首次调用：ListFolder
            arg := files.NewListFolderArg(f.opt.Enc.FromStandardPath(root))
            arg.Recursive = false
            arg.Limit = 1000
            if root == "/" { arg.Path = "" }  // 根目录特殊处理
            
            err = f.pacer.Call(func() (bool, error) {
                res, err = f.srv.ListFolder(arg)
                return shouldRetry(ctx, err)
            })
            started = true
        } else {
            // 后续调用：ListFolderContinue（使用 cursor）
            arg := files.ListFolderContinueArg{Cursor: res.Cursor}
            err = f.pacer.Call(func() (bool, error) {
                res, err = f.srv.ListFolderContinue(&arg)
                return shouldRetry(ctx, err)
            })
        }
        
        // 处理返回条目
        for _, entry := range res.Entries {
            // 转换为 fs.DirEntry
            entryPath := metadata.PathDisplay
            leaf := f.opt.Enc.ToStandardName(path.Base(entryPath))
            remote := path.Join(dir, leaf)
            
            if folderInfo != nil {
                d := fs.NewDir(remote, time.Time{}).SetID(folderInfo.Id)
                err = list.Add(d)
            } else if fileInfo != nil {
                o, err := f.newObjectWithInfo(ctx, remote, fileInfo)
                if o.(*Object).exportType.listable() {
                    err = list.Add(o)
                }
            }
        }
        
        // 检查是否还有更多
        if !res.HasMore {
            break
        }
    }
    return list.Flush()
}
```

### 3.4 分页机制详解

**Dropbox API 分页流程**：

```
1. 调用 ListFolder(path, limit=1000)
   ↓ 返回 { entries[], cursor, has_more }
2. 如果 has_more == true
   ↓ 调用 ListFolderContinue(cursor)
   ↓ 返回 { entries[], cursor, has_more }
3. 重复步骤 2 直到 has_more == false
```

**关键特性**：
- **Cursor 机制**：Dropbox 使用 cursor 标记分页位置，保证一致性
- **Limit 控制**：每次最多返回 1000 条（API 限制）
- **流式处理**：通过 `list.Add()` 分批回调，避免全量内存占用

### 3.5 `list.NewHelper` 统一缓冲

[`list.NewHelper`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/fs/list/helpers.go) 提供统一的条目缓冲和刷新机制：

```go
// list.NewHelper 创建一个辅助对象，内部维护条目缓冲区
// list.Add(entry) 添加条目到缓冲区，满了自动调用 callback
// list.Flush() 强制刷新剩余条目
```

这确保了所有后端的列表行为一致，无论后端是否原生支持分页。

---

## 四、重试边界控制

### 4.1 重试架构层次

```
┌─────────────────────────────────────────────────┐
│              应用层（sync/operations）          │
│  处理 RetryError / FatalError / NoRetryError    │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│              fs.Pacer (统一传输层)              │
│  low-level retry (ci.LowLevelRetries 次)        │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│         lib/pacer.Pacer (核心重试引擎)          │
│  速率控制 + 并发控制 + 重试循环                 │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│       Dropbox.shouldRetry (边界判定)            │
│  区分可重试错误 / 不可重试错误 / 致命错误       │
└─────────────────────────────────────────────────┘
```

### 4.2 重试边界判定：`shouldRetry`

[`shouldRetry`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L441-L460) 是 Dropbox 重试机制的核心边界：

```go
func shouldRetry(ctx context.Context, err error) (bool, error) {
    // 1. 先检查排除列表（绝对不可重试）
    if retry, err := shouldRetryExclude(ctx, err); !retry {
        return retry, err
    }
    
    // 2. 处理 Dropbox 官方速率限制（带 Retry-After）
    switch e := err.(type) {
    case auth.RateLimitAPIError:
        if e.RateLimitError.RetryAfter > 0 {
            err = pacer.RetryAfterError(err, time.Duration(e.RateLimitError.RetryAfter)*time.Second)
        }
        return true, err
    }
    
    // 3. 向后兼容的错误字符串匹配
    errString := err.Error()
    if strings.Contains(errString, "too_many_write_operations") || 
       strings.Contains(errString, "too_many_requests") {
        return true, err
    }
    
    // 4. 通用网络错误重试（fserrors.ShouldRetry）
    return fserrors.ShouldRetry(err), err
}
```

### 4.3 不可重试错误：`shouldRetryExclude`

[`shouldRetryExclude`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L417-L437) 定义**绝对不可重试**的错误边界：

```go
func shouldRetryExclude(ctx context.Context, err error) (bool, error) {
    // 上下文取消/超时不可重试
    if fserrors.ContextError(ctx, &err) {
        return false, err
    }
    
    errString := err.Error()
    
    // 空间不足 → 致命错误，终止整个操作
    if strings.Contains(errString, "insufficient_space") {
        return false, fserrors.FatalError(err)
    }
    
    // 路径格式错误 → 不可重试，重试也没用
    if strings.Contains(errString, "malformed_path") {
        return false, fserrors.NoRetryError(err)
    }
    
    return true, err
}
```

### 4.4 特殊场景的重试边界

#### 4.4.1 下载时的版权错误

[`Open`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L1952-L1958) 方法中处理版权受限内容：

```go
switch e := err.(type) {
case files.DownloadAPIError:
    // 版权违规错误，不可重试
    if e.EndpointError != nil && e.EndpointError.Path != nil && 
       e.EndpointError.Path.Tag == files.LookupErrorRestrictedContent {
        return nil, fserrors.NoRetryError(err)
    }
}
```

#### 4.4.2 移动操作的最终一致性重试

[`Move`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L1367-L1378) 处理 Dropbox API 的最终一致性问题：

```go
err = f.pacer.Call(func() (bool, error) {
    result, err = f.srv.MoveV2(&arg)
    switch e := err.(type) {
    case files.MoveV2APIError:
        // 刚创建的对象可能由于最终一致性暂时找不到，需要重试
        if e.EndpointError != nil && e.EndpointError.FromLookup != nil && 
           e.EndpointError.FromLookup.Tag == files.LookupErrorNotFound {
            fs.Debugf(srcObj, "Retrying move on %v error", err)
            return true, err  // 强制重试
        }
    }
    return shouldRetry(ctx, err)
})
```

#### 4.4.3 分块上传的偏移量错误处理

[`uploadChunked`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L2013-L2031) 处理上传偏移量不一致：

```go
if uErr, ok := err.(files.UploadSessionAppendV2APIError); ok {
    if uErr.EndpointError != nil && uErr.EndpointError.IncorrectOffset != nil {
        correctOffset := uErr.EndpointError.IncorrectOffset.CorrectOffset
        delta := int64(correctOffset) - int64(cursor.Offset)
        
        if skip == chunkSize {
            // 块已成功接收，继续下一块
            return false, nil
        } else if skip > 0 {
            // 调整偏移量后重试
            cursor.Offset = uint64(int64(cursor.Offset) + delta)
            return true, err
        }
    }
}
```

#### 4.4.4 分块上传后的重试边界

一旦第一个块上传成功，[`uploadChunked`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L2004-L2034) 和 [`finishBatch`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/batcher.go#L21-L28) 放宽重试策略：

```go
// uploadChunked 中的块上传
err = o.fs.pacer.Call(func() (bool, error) {
    // ...
    // 会话启动后，除了排除的错误外全部重试
    return err != nil, err
})

// finishBatch 中的批提交
err = f.pacer.Call(func() (bool, error) {
    complete, err = f.srv.UploadSessionFinishBatchV2(arg)
    if retry, err := shouldRetryExclude(ctx, err); !retry {
        return retry, err
    }
    // 第一个块上传后，除排除错误外全部重试
    return err != nil, err
})
```

---

## 五、进入统一传输流程的入口

### 5.1 统一传输流程架构

rclone 的**统一传输流程**由以下核心组件构成：

| 组件 | 位置 | 职责 |
|------|------|------|
| `fs.Pacer` | [`fs/pacer.go`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/fs/pacer.go) | 带日志的 pacer 包装 |
| `lib/pacer.Pacer` | [`lib/pacer/pacer.go`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/lib/pacer/pacer.go) | 核心速率控制和重试引擎 |
| `fserrors` | [`fs/fserrors/error.go`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/fs/fserrors/error.go) | 错误类型系统 |
| `list` 包 | [`fs/list/list.go`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/fs/list/list.go) | 统一列表处理 |

### 5.2 所有 API 调用的统一入口：`pacer.Call`

Dropbox 后端的**每一个 API 调用**都通过 `f.pacer.Call()` 进入统一传输流程。这是**关键的统一接入点**。

#### 5.2.1 Pacer 初始化

[`NewFs`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L514-L519) 中创建 Pacer：

```go
f := &Fs{
    // ...
    pacer: fs.NewPacer(ctx, pacer.NewDefault(
        pacer.MinSleep(opt.PacerMinSleep),    // 默认 10ms
        pacer.MaxSleep(maxSleep),              // 最大 2s
        pacer.DecayConstant(decayConstant),    // 衰减系数 2
    )),
}
```

[`fs.NewPacer`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/fs/pacer.go#L23-L37) 配置全局重试次数：

```go
func NewPacer(ctx context.Context, c pacer.Calculator) *Pacer {
    ci := GetConfig(ctx)
    retries := max(ci.LowLevelRetries, 1)          // 低级别重试次数
    maxConnections := max(ci.MaxConnections, 0)    // 最大并发连接
    p := &Pacer{
        Pacer: pacer.New(
            pacer.InvokerOption(pacerInvoker),     // 重试调用包装
            pacer.MaxConnectionsOption(maxConnections),
            pacer.RetriesOption(retries),
            pacer.CalculatorOption(c),
        ),
    }
    return p
}
```

#### 5.2.2 `pacer.Call` 执行流程

[`lib/pacer.Pacer.Call`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/lib/pacer/pacer.go#L243-L248) 是核心重试循环：

```go
func (p *Pacer) Call(fn Paced) error {
    p.mu.Lock()
    retries := p.retries  // 从配置读取，默认 10 次
    p.mu.Unlock()
    return p.call(fn, retries)
}

func (p *Pacer) call(fn Paced, retries int) error {
    for i := 1; i <= retries; i++ {
        p.beginCall(limitConnections)        // 获取 pacing token 和连接 token
        retry, err = p.invoker(i, retries, fn)  // 调用实际函数 + Dropbox.shouldRetry
        p.endCall(retry, err, limitConnections) // 计算下一次 sleep 时间
        if !retry {
            break
        }
    }
    return err
}
```

#### 5.2.3 调用包装器：`pacerInvoker`

[`fs/pacer.go:85-91`](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/fs/pacer.go#L85-L91) 在重试时记录日志并包装错误：

```go
func pacerInvoker(try, retries int, f pacer.Paced) (retry bool, err error) {
    retry, err = f()  // f() 内部调用 Dropbox.shouldRetry
    if retry {
        Debugf("pacer", "low level retry %d/%d (error %v)", try, retries, err)
        err = fserrors.RetryError(err)  // 标记为 RetryError
    }
    return
}
```

### 5.3 典型调用链路示例

#### 5.3.1 列表操作调用链

```
operations.ListDir
    ↓
fs.List (统一接口)
    ↓
dropbox.Fs.List
    ↓
list.WithListP (统一列表入口)
    ↓
dropbox.Fs.ListP
    ↓
f.pacer.Call(func() {
    f.srv.ListFolder(arg)        // Dropbox SDK 调用
    return shouldRetry(ctx, err) // 边界判定
})
    ↓
lib/pacer.Pacer.call (重试循环)
    ↓
pacerInvoker (日志 + RetryError 包装)
```

#### 5.3.2 上传操作调用链

```
operations.Copy
    ↓
fs.Put (统一接口)
    ↓
dropbox.Fs.Put
    ↓
dropbox.Object.Update
    ↓
dropbox.Object.uploadChunked (分块上传)
    ↓ [每个块]
f.pacer.Call(func() {
    f.srv.UploadSessionAppendV2(...)
    // 偏移量错误特殊处理
    return err != nil, err      // 会话启动后放宽重试
})
```

#### 5.3.3 元数据操作调用链

```
operations.Stat
    ↓
fs.NewObject (统一接口)
    ↓
dropbox.Fs.NewObject
    ↓
dropbox.Object.readEntryAndSetMetadata
    ↓
dropbox.Fs.getFileMetadata
    ↓
dropbox.Fs.getMetadata
    ↓
f.pacer.Call(func() {
    f.srv.GetMetadata(...)
    return shouldRetry(ctx, err) // 边界判定
})
```

### 5.4 错误类型流向

```
Dropbox SDK 返回原始错误
    ↓
dropbox.shouldRetry(err) → (retry bool, wrappedErr error)
    ├─ 不可重试 → fserrors.FatalError / NoRetryError
    ├─ 速率限制 → pacer.RetryAfterError (带 Retry-After)
    └─ 可重试 → 原始错误或 fserrors.ShouldRetry 判定
    ↓
pacerInvoker 包装 → fserrors.RetryError (如果 retry==true)
    ↓
lib/pacer 决定是否继续循环
    ↓
上层应用（sync/operations）处理最终错误
    ├─ RetryError → 高层重试
    ├─ FatalError → 立即终止
    └─ NoRetryError → 记录错误但不重试
```

---

## 六、关键设计模式总结

### 6.1 路径映射三要素

1. **层级表示**：`root` / `slashRoot` / `slashRootSlash` 三级缓存
2. **编解码分离**：`FromStandardPath` / `ToStandardPath` 双向转换
3. **特殊映射**：导出文件路径 munging 处理 Paper 等特殊文件

### 6.2 分页列表两阶段

1. **统一入口**：`List` → `list.WithListP` → `ListP` 标准化调用
2. **流式处理**：`list.NewHelper` 缓冲 + 回调，避免全量加载

### 6.3 重试边界三层防护

1. **排除层**：`shouldRetryExclude` 过滤绝对不可重试错误
2. **适配层**：`shouldRetry` 处理 Dropbox 特定错误（速率限制、最终一致性）
3. **通用层**：`fserrors.ShouldRetry` 处理网络等通用错误

### 6.4 统一传输接入点

**所有 Dropbox API 调用必经之路**：
```go
err = f.pacer.Call(func() (bool, error) {
    result, err = f.srv.SomeAPICall(args)
    return shouldRetry(ctx, err)  // ← 边界判定接入点
})
```

这种设计确保：
- ✅ 所有调用自动获得速率控制
- ✅ 所有调用自动获得重试机制
- ✅ 错误分类统一处理
- ✅ 并发连接数全局控制
- ✅ 重试日志统一记录

---

## 七、代码参考速查

| 功能 | 文件位置 |
|------|----------|
| 路径规范化 `setRoot` | [dropbox.go:689-696](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L689-L696) |
| 对象路径 `remotePath` | [dropbox.go:1856-1858](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L1856-L1858) |
| 重试排除 `shouldRetryExclude` | [dropbox.go:417-437](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L417-L437) |
| 重试判定 `shouldRetry` | [dropbox.go:441-460](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L441-L460) |
| 分页列表 `ListP` | [dropbox.go:1045-1148](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L1045-L1148) |
| 分块上传 `uploadChunked` | [dropbox.go:1968-2079](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/backend/dropbox/dropbox.go#L1968-L2079) |
| Pacer 创建 `NewPacer` | [fs/pacer.go:23-37](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/fs/pacer.go#L23-L37) |
| 核心重试 `pacer.call` | [lib/pacer/pacer.go:220-235](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/lib/pacer/pacer.go#L220-L235) |
| 错误类型系统 | [fs/fserrors/error.go](file:///d:/fz/0601-2/solo-dogfeeding/code/43-rclone/fs/fserrors/error.go) |
