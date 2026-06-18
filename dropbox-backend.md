# Dropbox 后端代码分析

本文档深入分析 rclone Dropbox 后端的核心架构，重点讲解**远端路径映射**、**根命名空间**、**共享文件/文件夹模式**、**根路径文件回退处理**、**分页列表**和**重试边界**如何进入 rclone 的**统一传输流程**。

---

## 一、整体架构概览

Dropbox 后端位于 `backend/dropbox/` 目录，核心文件包括：

| 文件 | 职责 |
|------|------|
| `dropbox.go` | 主后端实现，包含 `Fs` 和 `Object` 结构 |
| `batcher.go` | 批量上传提交实现 |
| `dbhash/dbhash.go` | Dropbox 内容哈希算法 |

### 核心接口实现

Dropbox 后端实现了 rclone 的标准文件系统接口（定义于 `fs/types.go`）：

```go
var (
    _ fs.Fs           = (*Fs)(nil)
    _ fs.Copier       = (*Fs)(nil)    // 服务端拷贝
    _ fs.Mover        = (*Fs)(nil)    // 服务端移动
    _ fs.ListPer      = (*Fs)(nil)    // 分页列表
    _ fs.Abouter      = (*Fs)(nil)    // 配额查询
    _ fs.Object       = (*Object)(nil)
)
```

### Fs 结构核心字段

```go
type Fs struct {
    name           string         // remote 名称
    root           string         // 逻辑根路径
    slashRoot      string         // "/" + root
    slashRootSlash string         // "/" + root + "/"
    opt            Options        // 配置选项
    srv            files.Client   // Dropbox Files API 客户端
    sharing        sharing.Client // Dropbox Sharing API 客户端
    users          users.Client   // Dropbox Users API 客户端
    pacer          *fs.Pacer      // 统一速率/重试控制器
    ns             string         // 根命名空间 ID
    batcher        *batcher.Batcher[...] // 批量上传
}

type Options struct {
    ChunkSize        fs.SizeSuffix
    SharedFiles      bool                 // 共享文件模式
    SharedFolders    bool                 // 共享文件夹模式
    RootNsid         string               // 根命名空间覆盖
    Enc              encoder.MultiEncoder // 路径编码器
    // ...
}
```

---

## 二、NewFs 初始化分支全景

`NewFs` 函数是后端初始化的唯一入口，包含四个主要决策分支：

```
NewFs(ctx, name, root, m)
    │
    ├─ 分支 1: SharedFiles == true
    │    └─ 共享文件模式，功能受限
    │
    ├─ 分支 2: SharedFolders == true
    │    └─ 共享文件夹模式 → 尝试挂载 → 成功则降级为普通模式
    │
    ├─ 分支 3: 根命名空间设置
    │    ├─ RootNsid != "" → 直接使用
    │    └─ root 以 "/" 开头 → 查询用户账户获取
    │
    └─ 分支 4: 根路径指向文件检测
         └─ root 是文件 → 回退到父目录 + 返回 ErrorIsFile
```

各分支的执行顺序至关重要：SharedFiles → SharedFolders → RootNsid → 文件检测。

---

## 三、实际根命名空间（Root Namespace）

### 3.1 背景：Dropbox 命名空间概念

Dropbox API 使用**命名空间（Namespace）**隔离不同的数据域：
- **个人用户命名空间**：用户自己的文件空间
- **团队命名空间**：Dropbox Business 团队的共享空间
- **共享文件夹命名空间**：每个共享文件夹独立的命名空间

路径解析需要指定在哪个命名空间下进行。

### 3.2 命名空间的两种设置方式

#### 方式一：显式配置 RootNsid

```go
if f.opt.RootNsid != "" {
    f.ns = f.opt.RootNsid
    fs.Debugf(f, "Overriding root namespace to %q", f.ns)
}
```

用户通过 `--dropbox-root-namespace` 参数直接指定命名空间 ID。

#### 方式二：自动检测（root 以 "/" 开头）

```go
else if strings.HasPrefix(root, "/") {
    var acc *users.FullAccount
    err = f.pacer.Call(func() (bool, error) {
        acc, err = f.users.GetCurrentAccount()
        return shouldRetry(ctx, err)
    })
    switch x := acc.RootInfo.(type) {
    case *common.TeamRootInfo:      // 团队账户
        f.ns = x.RootNamespaceId
    case *common.UserRootInfo:      // 个人用户
        f.ns = x.RootNamespaceId
    }
}
```

**触发条件**：当传入的 root 路径以 "/" 开头时，表示需要使用**实际根**（绝对路径），而非用户默认的 home 命名空间。

### 3.3 命名空间的注入机制

命名空间通过 `headerGenerator` 注入到**每一个 API 请求**：

```go
func (f *Fs) headerGenerator(hostType string, namespace string, route string) map[string]string {
    if f.ns == "" {
        return map[string]string{}
    }
    return map[string]string{
        "Dropbox-API-Path-Root": `{".tag": "namespace_id", "namespace_id": "` + f.ns + `"}`,
    }
}
```

Dropbox SDK 在每次发起 HTTP 请求前调用此函数获取额外请求头。

**注意**：`headerGenerator` 在 SDK 客户端创建时就已绑定：
```go
cfg := dropbox.Config{
    Client:          oAuthClient,
    HeaderGenerator: f.headerGenerator,  // 在此绑定
}
f.srv = files.New(cfg)
```

这意味着即使 `f.ns` 后续被修改（如 SharedFolders 挂载后），所有新建请求也会自动使用最新的命名空间。

### 3.4 命名空间接入传输链路

```
所有 API 调用 (ListFolder / GetMetadata / Download / ...)
    ↓
Dropbox SDK 内部发送 HTTP 请求
    ↓
SDK 调用 headerGenerator(hostType, namespace, route)
    ↓
如果 f.ns != ""，注入 Dropbox-API-Path-Root 请求头
    ↓
请求到达 Dropbox 服务端，在指定命名空间下解析路径
```

命名空间机制是**透明**的——它不影响 pacer、重试等统一传输链路的逻辑，仅作为 HTTP 头在请求发出时注入。

---

## 四、共享文件模式（SharedFiles Mode）

### 4.1 模式说明

当 `--dropbox-shared-files` 启用时，后端工作在**共享文件模式**。此模式下 rclone 仅操作通过共享链接单独分享给用户的文件，而非用户命名空间内的常规文件。

### 4.2 NewFs 中的初始化分支

```go
if f.opt.SharedFiles {
    f.setRoot(root)
    if f.root == "" {
        return f, nil  // 空 root，直接返回
    }
    // 尝试以共享文件方式查找 root
    _, err := f.findSharedFile(ctx, f.root)
    f.root = ""  // 无论是否找到，root 都重置为空
    if err == nil {
        return f, fs.ErrorIsFile  // 找到 → 告诉上层 root 是文件
    }
    return f, nil  // 没找到 → 当作空目录
}
```

**关键点**：
- `setRoot(root)` 先正常设置路径
- `findSharedFile` 遍历所有共享文件查找匹配项
- **不管查找结果如何，最终 `f.root` 被重置为空字符串**
- 这是因为共享文件没有真正的"目录"概念，所有文件都列在虚拟根下

### 4.3 共享文件模式下的功能限制

此模式功能极度受限，以下操作统一返回 `errNotSupportedInSharedMode`：

| 操作 | 位置 |
|------|------|
| `Put` / `PutStream` | 上传文件 |
| `Mkdir` | 创建目录 |
| `Rmdir` / `Purge` | 删除目录 |
| `Update` | 更新文件 |
| `Remove` | 删除文件 |
| `Hash` | 计算哈希 |

错误定义：
```go
var errNotSupportedInSharedMode = fserrors.NoRetryError(
    errors.New("not supported in shared files mode"),
)
```

### 4.4 列表链路接入

`ListP` 中的分支：

```go
func (f *Fs) ListP(ctx context.Context, dir string, callback fs.ListRCallback) error {
    list := list.NewHelper(callback)
    if f.opt.SharedFiles {
        err := f.listReceivedFiles(ctx, list.Add)
        if err != nil {
            return err
        }
        return list.Flush()
    }
    // ... 正常模式代码
}
```

**接入点**：
- 不调用标准的 `ListFolder` API
- 改为调用 `sharing.ListReceivedFiles`（Sharing API）
- 通过 `list.Add` → `list.Flush` 仍接入统一列表缓冲流程
- 通过 `f.pacer.Call` 仍接入统一传输/重试链路

`listReceivedFiles` 实现：

```go
func (f *Fs) listReceivedFiles(ctx context.Context, callback func(fs.DirEntry) error) error {
    for {
        if !started {
            arg := sharing.ListFilesArg{Limit: 100}
            err := f.pacer.Call(func() (bool, error) {
                res, err = f.sharing.ListReceivedFiles(&arg)
                return shouldRetry(ctx, err)
            })
            started = true
        } else {
            arg := sharing.ListFilesContinueArg{Cursor: res.Cursor}
            err := f.pacer.Call(func() (bool, error) {
                res, err = f.sharing.ListReceivedFilesContinue(&arg)
                return shouldRetry(ctx, err)
            })
        }
        for _, entry := range res.Entries {
            o := &Object{
                fs:      f,
                url:     entry.PreviewUrl,  // 保存共享链接 URL
                remote:  entry.Name,
                modTime: *entry.TimeInvited,
            }
            err = callback(o)
        }
        if res.Cursor == "" { break }
    }
}
```

**与正常模式的差异**：
- API 从 `files.ListFolder` 变为 `sharing.ListReceivedFiles`
- Object 不保存 `id` / `bytes` / `hash`，仅保存 `url` + `remote` + `modTime`
- `bytes` 字段为 0（零值），`hash` 为空

### 4.5 NewObject 链路接入

```go
func (f *Fs) NewObject(ctx context.Context, remote string) (fs.Object, error) {
    if f.opt.SharedFiles {
        return f.findSharedFile(ctx, remote)
    }
    return f.newObjectWithInfo(ctx, remote, nil)
}
```

`findSharedFile` 通过遍历 `listReceivedFiles` 线性查找：

```go
func (f *Fs) findSharedFile(ctx context.Context, name string) (*Object, error) {
    errFoundFile := errors.New("found file")
    err = f.listReceivedFiles(ctx, func(entry fs.DirEntry) error {
        if entry.(*Object).remote == name {
            o = entry.(*Object)
            return errFoundFile
        }
        return nil
    })
    if errors.Is(err, errFoundFile) {
        return o, nil
    }
    return nil, fs.ErrorObjectNotFound
}
```

**注意**：每次 `NewObject` 都会触发一次完整的共享文件列表遍历，复杂度 O(n)。

### 4.6 文件读取（Open）链路接入

`Object.Open` 中的分支：

```go
func (o *Object) Open(ctx context.Context, options ...fs.OpenOption) (io.ReadCloser, error) {
    if o.fs.opt.SharedFiles {
        if len(options) != 0 {
            return nil, errors.New("OpenOptions not supported for shared files")
        }
        arg := sharing.GetSharedLinkMetadataArg{
            Url: o.url,  // 使用 listReceivedFiles 保存的共享 URL
        }
        err = o.fs.pacer.Call(func() (bool, error) {
            _, in, err = o.fs.sharing.GetSharedLinkFile(&arg)
            return shouldRetry(ctx, err)
        })
        return in, err
    }
    // ... 正常模式：调用 files.Download
}
```

**接入要点**：
- 不使用 Object ID，使用共享链接 URL（`o.url`）
- API 从 `files.Download` 变为 `sharing.GetSharedLinkFile`
- 不支持 Range 请求（`OpenOptions` 被拒绝）
- 通过 `pacer.Call` + `shouldRetry` 仍接入统一传输链路

### 4.7 共享文件模式接入全景

```
SharedFiles == true
    │
    ├─ List / ListP
    │    └─ listReceivedFiles (sharing.ListReceivedFiles + pacer.Call)
    │         └─ list.NewHelper → 统一列表缓冲
    │
    ├─ NewObject
    │    └─ findSharedFile → 遍历 listReceivedFiles
    │
    ├─ Object.Open
    │    └─ sharing.GetSharedLinkFile(o.url) + pacer.Call
    │         └─ 统一重试/速率控制
    │
    ├─ Put / Update / Mkdir / Rmdir / Remove / Hash
    │    └─ 直接返回 errNotSupportedInSharedMode (NoRetryError)
    │
    └─ 命名空间 / headerGenerator
         └─ 不影响，仍正常工作
```

---

## 五、共享文件夹模式（SharedFolders Mode）

### 5.1 模式说明

当 `--dropbox-shared-folders` 启用时，后端工作在**共享文件夹模式**。该模式允许用户浏览和挂载被分享的文件夹。

### 5.2 NewFs 中的初始化分支

```go
if f.opt.SharedFolders {
    f.setRoot(root)
    if f.root == "" {
        return f, nil  // 空 root → 列出所有共享文件夹
    }

    // root 非空，解析出共享文件夹名
    dir := path.Dir(f.root)
    if dir == "." {
        dir = f.root
    }

    // 遍历所有共享文件夹，按名称查找 ID
    id, err := f.findSharedFolder(ctx, dir)
    if err != nil {
        return nil, err
    }

    // 挂载共享文件夹到用户的根命名空间
    err = f.mountSharedFolder(ctx, id)
    if err != nil {
        switch e := err.(type) {
        case sharing.MountFolderAPIError:
            // 已经挂载过不算错误
            if e.EndpointError != nil && 
               e.EndpointError.Tag == sharing.MountFolderErrorAlreadyMounted {
                // 忽略
            } else {
                return nil, err
            }
        default:
            return nil, err
        }
    }

    // 关键：挂载成功后关闭共享文件夹模式，走正常流程
    f.opt.SharedFolders = false
}
```

**执行流程**：
1. 空 root → 直接返回，后续列表时列出所有共享文件夹
2. 非空 root → 按名称查找共享文件夹 ID → 挂载 → `SharedFolders = false`
3. 挂载后，共享文件夹变成用户命名空间下的普通文件夹，后续所有操作走正常路径

### 5.3 共享文件夹列表链路

当 root 为空且 `SharedFolders == true` 时，`ListP` 的分支：

```go
func (f *Fs) ListP(ctx context.Context, dir string, callback fs.ListRCallback) error {
    list := list.NewHelper(callback)
    // ... SharedFiles 分支 ...
    if f.opt.SharedFolders {
        err := f.listSharedFolders(ctx, list.Add)
        if err != nil {
            return err
        }
        return list.Flush()
    }
    // ... 正常模式代码
}
```

`listSharedFolders` 实现：

```go
func (f *Fs) listSharedFolders(ctx context.Context, callback func(fs.DirEntry) error) error {
    for {
        if !started {
            arg := sharing.ListFoldersArgs{Limit: 100}
            err := f.pacer.Call(func() (bool, error) {
                res, err = f.sharing.ListFolders(&arg)
                return shouldRetry(ctx, err)
            })
            started = true
        } else {
            arg := sharing.ListFoldersContinueArg{Cursor: res.Cursor}
            err := f.pacer.Call(func() (bool, error) {
                res, err = f.sharing.ListFoldersContinue(&arg)
                return shouldRetry(ctx, err)
            })
        }
        for _, entry := range res.Entries {
            leaf := f.opt.Enc.ToStandardName(entry.Name)
            d := fs.NewDir(leaf, time.Time{}).SetID(entry.SharedFolderId)
            err = callback(d)
        }
        if res.Cursor == "" { break }
    }
}
```

**返回类型**：返回 `*fs.Dir` 目录条目（而非 Object），因为浏览的是文件夹列表。

### 5.4 文件夹查找与挂载

`findSharedFolder` 遍历共享文件夹列表按名称匹配：

```go
func (f *Fs) findSharedFolder(ctx context.Context, name string) (string, error) {
    errFoundFile := errors.New("found file")
    err = f.listSharedFolders(ctx, func(entry fs.DirEntry) error {
        if entry.(*fs.Dir).Remote() == name {
            id = entry.(*fs.Dir).ID()  // 取得 SharedFolderId
            return errFoundFile
        }
        return nil
    })
    if errors.Is(err, errFoundFile) {
        return id, nil
    }
    return "", fs.ErrorDirNotFound
}
```

`mountSharedFolder` 调用 Sharing API 挂载：

```go
func (f *Fs) mountSharedFolder(ctx context.Context, id string) error {
    arg := sharing.MountFolderArg{
        SharedFolderId: id,
    }
    err := f.pacer.Call(func() (bool, error) {
        _, err := f.sharing.MountFolder(&arg)
        return shouldRetry(ctx, err)
    })
    return err
}
```

### 5.5 模式降级：从共享文件夹到普通模式

**关键设计**：当 `NewFs` 中指定了一个具体的共享文件夹路径并成功挂载后，执行：

```go
f.opt.SharedFolders = false
```

这意味着后续的 `List` / `NewObject` / `Put` / `Open` 等所有操作**不再走共享文件夹分支**，而是直接使用正常的 Files API。

```
挂载前（SharedFolders=true）        挂载后（SharedFolders=false）
─────────────────────────         ─────────────────────────
ListP → listSharedFolders          ListP → files.ListFolder
NewObject → 不适用（返回 Dir）      NewObject → files.GetMetadata
Put → errNotSupportedInSharedMode  Put → files.Upload / UploadSession
...                                 ...
```

这也是为什么配置说明中提到："首次使用某个共享文件夹后，`--dropbox-shared-folders` 参数可以省略"——因为文件夹一旦被挂载，就永久出现在用户的正常命名空间中了。

### 5.6 共享文件夹模式接入全景

```
SharedFolders == true
    │
    ├─ NewFs
    │    ├─ root == ""
    │    │    └─ 直接返回，不做命名空间处理
    │    └─ root != ""
    │         ├─ findSharedFolder → listSharedFolders 遍历查找
    │         ├─ mountSharedFolder → sharing.MountFolder + pacer.Call
    │         └─ f.opt.SharedFolders = false → 降级为普通模式
    │
    ├─ List / ListP（仅 root 为空时走此分支）
    │    └─ listSharedFolders (sharing.ListFolders + pacer.Call)
    │         └─ list.NewHelper → 统一列表缓冲
    │
    ├─ NewObject（仅 root 为空时，实际上返回 Dir 而非 Object）
    │    └─ 普通模式分支（因为 SharedFolders 已被设为 false 或 root 为空）
    │
    ├─ Put / Update / Mkdir / 等写入操作
    │    └─ root 为空时返回 errNotSupportedInSharedMode
    │    └─ root 非空时走普通模式（SharedFolders=false）
    │
    └─ 命名空间
         └─ root 非空挂载后，正常使用 f.ns（如果有设置）
```

---

## 六、根路径指向文件时的回退处理

### 6.1 场景说明

用户可能会执行类似命令：
```
rclone ls dropbox:path/to/file.txt
```

这里 `root` 是 `"path/to/file.txt"`，指向一个文件而不是目录。后端需要正确处理这种情况。

### 6.2 检测与回退逻辑

`NewFs` 末尾的代码（注意：此逻辑在 SharedFiles/SharedFolders 分支之后执行）：

```go
f.setRoot(root)

// See if the root is actually an object
if f.root != "" {
    _, err = f.getFileMetadata(ctx, f.slashRoot)
    if err == nil {
        // 成功获取文件元数据 → root 确实是文件
        newRoot := path.Dir(f.root)
        if newRoot == "." {
            newRoot = ""
        }
        f.setRoot(newRoot)  // 回退到父目录
        // 返回 fs.ErrorIsFile 告诉上层：root 指向的是文件
        return f, fs.ErrorIsFile
    }
}
return f, nil
```

**执行步骤**：
1. 先按正常方式设置 root（`f.setRoot(root)`）
2. 调用 `getFileMetadata` 检查 `slashRoot` 是否为文件
3. 如果是文件（`err == nil`）：
   - 用 `path.Dir` 计算父目录路径
   - 特殊处理：`path.Dir("file.txt") == "."` → 转换为 `""`（根目录）
   - 重新设置 root 为父目录
   - 返回 `(f, fs.ErrorIsFile)` 对

### 6.3 getFileMetadata 的实现

```go
func (f *Fs) getFileMetadata(ctx context.Context, filePath string) (*files.FileMetadata, error) {
    var res getMetadataResult

    // 尝试所有可能的路径组合（包括导出文件映射）
    possibleMetadatas := f.possibleMetadatas(ctx, filePath)
    for _, ch := range possibleMetadatas {
        res = <-ch
        if res.err != nil {
            return nil, res.err
        }
        if !res.notFound {
            break
        }
    }

    if res.notFound {
        return nil, fs.ErrorObjectNotFound
    }

    fileInfo, ok := res.entry.(*files.FileMetadata)
    if !ok {
        if _, ok = res.entry.(*files.FolderMetadata); ok {
            return nil, fs.ErrorIsDir
        }
        return nil, fs.ErrorNotAFile
    }
    return fileInfo, nil
}
```

注意 `getFileMetadata` 内部也走 `pacer.Call` + `shouldRetry`，接入了统一传输链路。

### 6.4 ErrorIsFile 的语义

`fs.ErrorIsFile` 是一个约定错误，告诉调用方：

> "你传入的路径指向一个文件而不是目录。我已经将 Fs 的 root 调整为该文件所在的父目录，你可以在此 Fs 上调用 `NewObject("filename")` 来获得这个文件对象。"

上层（如 `cmd/copy`、`fs/walk`、`fs/sync` 等）根据此错误做相应处理，例如：
- `rclone ls dropbox:path/to/file.txt` → 列出该文件所在目录并过滤显示该文件
- `rclone copy dropbox:path/to/file.txt /tmp/` → 直接复制这一个文件

### 6.5 回退处理接入传输链路

```
NewFs(root = "path/to/file.txt")
    │
    ├─ f.setRoot("path/to/file.txt")
    │    └─ f.slashRoot = "/path/to/file.txt"
    │
    ├─ f.getFileMetadata(ctx, f.slashRoot)
    │    └─ f.getMetadata(...)
    │         └─ f.pacer.Call(func() {
    │              f.srv.GetMetadata(...)
    │              return shouldRetry(ctx, err)
    │            })
    │              ↓ 接入统一传输链路（重试 + 速率控制）
    │
    ├─ 确认是文件 → newRoot = path.Dir("path/to/file.txt") = "path/to"
    │
    ├─ f.setRoot("path/to")
    │    └─ f.slashRoot = "/path/to"
    │       f.slashRootSlash = "/path/to/"
    │
    └─ return (f, fs.ErrorIsFile)
```

**关键**：回退检测本身也通过 `pacer.Call` 执行，享受统一的重试和速率控制。如果 `GetMetadata` API 调用失败（如网络错误），错误会按正常重试逻辑处理，而不会误判为"不是文件"。

---

## 七、远端路径映射机制

### 7.1 路径层级结构

Dropbox 后端维护三层路径表示：

```
rclone 逻辑路径 → 规范化路径 → Dropbox API 路径
     (remote)    (slashRoot)    (编码后路径)
```

**核心字段**：

```go
type Fs struct {
    root           string   // 逻辑根路径，如 "folder/subfolder"
    slashRoot      string   // 带 "/" 前缀的根路径，如 "/folder/subfolder"
    slashRootSlash string   // 带前后 "/" 的根路径，如 "/folder/subfolder/"
    opt            Options  // 包含编码器 Enc
}
```

### 7.2 路径初始化：setRoot

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

### 7.3 对象路径计算：remotePath

```go
func (o *Object) remotePath() string {
    return o.fs.slashRootSlash + o.remote
}
```

**示例**：
- `f.root = "work/docs"` → `f.slashRootSlash = "/work/docs/"`
- `o.remote = "report.pdf"` → `remotePath() = "/work/docs/report.pdf"`

### 7.4 路径编解码

Dropbox 使用 `encoder.MultiEncoder` 处理特殊字符：

**默认编码规则**：
```go
Default: encoder.Base |
    encoder.EncodeBackSlash |    // 编码反斜杠 "\"
    encoder.EncodeDel |          // 编码 DEL 字符 (0x7F)
    encoder.EncodeRightSpace |   // 编码尾部空格
    encoder.EncodeInvalidUtf8,   // 编码无效 UTF-8
```

**编解码调用点**：
- **发送到 Dropbox API**：`f.opt.Enc.FromStandardPath(path)`
- **从 Dropbox API 接收**：`f.opt.Enc.ToStandardPath(path)`

### 7.5 导出文件路径映射（Export Path Munging）

Dropbox Paper 等文件需要导出才能访问，涉及特殊的路径映射逻辑：

`possibleMetadatas` 尝试多种路径组合：

```go
func (f *Fs) possibleMetadatas(ctx context.Context, filePath string) []<-chan getMetadataResult {
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

`setMetadataForExport` 设置导出元数据时修改 remote 路径：

```go
func (o *Object) setMetadataForExport(info *files.FileMetadata) {
    // 移除 .paper 扩展名
    o.remote = strings.TrimSuffix(o.remote, paperExtension)
    // 添加导出格式扩展名
    o.remote += "." + string(exportExt)
}
```

### 7.6 路径长度校验

`checkPathLength` 确保路径各部分不超过 Dropbox 限制（255 字符）：

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

## 八、分页列表流程

### 8.1 列表接口层次

Dropbox 实现了两级列表接口：

| 接口 | 用途 |
|------|------|
| `List` | 传统列表，返回全量结果 |
| `ListP` | 流式分页列表，支持回调 |

### 8.2 统一列表入口：list.WithListP

`List` 方法直接委托给 `list.WithListP`，这是**进入统一传输流程的第一个入口**：

```go
func (f *Fs) List(ctx context.Context, dir string) (entries fs.DirEntries, err error) {
    return list.WithListP(ctx, dir, f)
}
```

`list.WithListP` 是 rclone 的统一列表封装，它会根据后端特性选择调用 `ListP` 或回退到 `List`。

### 8.3 ListP 核心实现

```go
func (f *Fs) ListP(ctx context.Context, dir string, callback fs.ListRCallback) error {
    list := list.NewHelper(callback)
    
    // ===== 特殊模式分支（已在第四、五章详述）=====
    if f.opt.SharedFiles { return f.listReceivedFiles(ctx, list.Add) }
    if f.opt.SharedFolders { return f.listSharedFolders(ctx, list.Add) }
    
    // ===== 正常模式 =====
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

### 8.4 分页机制详解

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

### 8.5 list.NewHelper 统一缓冲

`list.NewHelper` 提供统一的条目缓冲和刷新机制：

```
// list.NewHelper 创建一个辅助对象，内部维护条目缓冲区
// list.Add(entry) 添加条目到缓冲区，满了自动调用 callback
// list.Flush() 强制刷新剩余条目
```

这确保了所有后端的列表行为一致，无论后端是否原生支持分页。

---

## 九、重试边界控制

### 9.1 重试架构层次

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

### 9.2 重试边界判定：shouldRetry

`shouldRetry` 是 Dropbox 重试机制的核心边界：

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
            err = pacer.RetryAfterError(err, 
                time.Duration(e.RateLimitError.RetryAfter)*time.Second)
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

### 9.3 不可重试错误：shouldRetryExclude

`shouldRetryExclude` 定义**绝对不可重试**的错误边界：

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

### 9.4 特殊场景的重试边界

#### 9.4.1 下载时的版权错误

`Open` 方法中处理版权受限内容：

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

#### 9.4.2 移动操作的最终一致性重试

`Move` 处理 Dropbox API 的最终一致性问题：

```go
err = f.pacer.Call(func() (bool, error) {
    result, err = f.srv.MoveV2(&arg)
    switch e := err.(type) {
    case files.MoveV2APIError:
        // 刚创建的对象可能由于最终一致性暂时找不到，需要重试
        if e.EndpointError != nil && e.EndpointError.FromLookup != nil && 
           e.EndpointError.FromLookup.Tag == files.LookupErrorNotFound {
            return true, err  // 强制重试
        }
    }
    return shouldRetry(ctx, err)
})
```

#### 9.4.3 分块上传的偏移量错误处理

`uploadChunked` 处理上传偏移量不一致：

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

#### 9.4.4 分块上传后的重试边界

一旦第一个块上传成功，`uploadChunked` 和 `finishBatch` 放宽重试策略：

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

## 十、进入统一传输流程的入口

### 10.1 统一传输流程架构

rclone 的**统一传输流程**由以下核心组件构成：

| 组件 | 位置 | 职责 |
|------|------|------|
| `fs.Pacer` | `fs/pacer.go` | 带日志的 pacer 包装 |
| `lib/pacer.Pacer` | `lib/pacer/pacer.go` | 核心速率控制和重试引擎 |
| `fserrors` | `fs/fserrors/error.go` | 错误类型系统 |
| `list` 包 | `fs/list/list.go` | 统一列表处理 |

### 10.2 所有 API 调用的统一入口：pacer.Call

Dropbox 后端的**每一个 API 调用**都通过 `f.pacer.Call()` 进入统一传输流程。这是**关键的统一接入点**。

#### 10.2.1 Pacer 初始化

`NewFs` 中创建 Pacer：

```go
f := &Fs{
    pacer: fs.NewPacer(ctx, pacer.NewDefault(
        pacer.MinSleep(opt.PacerMinSleep),    // 默认 10ms
        pacer.MaxSleep(maxSleep),              // 最大 2s
        pacer.DecayConstant(decayConstant),    // 衰减系数 2
    )),
}
```

`fs.NewPacer` 配置全局重试次数：

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

#### 10.2.2 pacer.Call 执行流程

`lib/pacer.Pacer.Call` 是核心重试循环：

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

#### 10.2.3 调用包装器：pacerInvoker

`pacerInvoker` 在重试时记录日志并包装错误：

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

### 10.3 典型调用链路示例

#### 10.3.1 列表操作调用链

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
    ├─ [SharedFiles] → listReceivedFiles → sharing.ListReceivedFiles
    ├─ [SharedFolders] → listSharedFolders → sharing.ListFolders
    └─ [正常模式] → files.ListFolder / ListFolderContinue
    ↓
f.pacer.Call(func() {
    f.srv.SomeListAPI(arg)
    return shouldRetry(ctx, err)
})
    ↓
lib/pacer.Pacer.call (重试循环)
    ↓
pacerInvoker (日志 + RetryError 包装)
```

#### 10.3.2 上传操作调用链

```
operations.Copy
    ↓
fs.Put (统一接口)
    ↓
dropbox.Fs.Put
    ├─ [SharedFiles || SharedFolders] → errNotSupportedInSharedMode
    └─ [正常模式]
         ↓
dropbox.Object.Update
    ↓
[size > chunkSize || batching] → uploadChunked (分块上传)
[else] → files.Upload (单块上传)
    ↓ [每个块/请求]
f.pacer.Call(func() {
    f.srv.UploadSessionAppendV2(...)
    // 偏移量错误特殊处理
    return err != nil, err
})
```

#### 10.3.3 元数据操作调用链

```
operations.Stat
    ↓
fs.NewObject (统一接口)
    ↓
dropbox.Fs.NewObject
    ├─ [SharedFiles] → findSharedFile → listReceivedFiles 遍历
    └─ [正常模式]
         ↓
dropbox.Object.readEntryAndSetMetadata
    ↓
dropbox.Fs.getFileMetadata
    ↓
dropbox.Fs.getMetadata
    ↓
f.pacer.Call(func() {
    f.srv.GetMetadata(...)
    return shouldRetry(ctx, err)
})
```

#### 10.3.4 文件下载调用链

```
operations.Copy (下载端)
    ↓
fs.Object.Open (统一接口)
    ↓
dropbox.Object.Open
    ├─ [SharedFiles] → sharing.GetSharedLinkFile (使用 URL)
    ├─ [exportType] → files.Export (导出 Paper)
    └─ [正常模式] → files.Download (使用 ID，支持 Range)
    ↓
f.pacer.Call(func() {
    f.srv.SomeDownloadAPI(...)
    return shouldRetry(ctx, err)
})
    ↓
[版权错误检查] → fserrors.NoRetryError
```

### 10.4 错误类型流向

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

## 十一、关键设计模式总结

### 11.1 路径映射三要素

1. **层级表示**：`root` / `slashRoot` / `slashRootSlash` 三级缓存
2. **编解码分离**：`FromStandardPath` / `ToStandardPath` 双向转换
3. **特殊映射**：导出文件路径 munging 处理 Paper 等特殊文件

### 11.2 命名空间注入模式

1. **延迟绑定**：`headerGenerator` 在请求发送时才读取 `f.ns`，支持运行时修改
2. **透明注入**：命名空间通过 HTTP 头传递，对上层业务逻辑完全透明
3. **双通道**：显式配置（`RootNsid`）与自动检测（root 以 "/" 开头）互补

### 11.3 共享模式分支策略

1. **SharedFiles**：完全独立的 API 路径（Sharing API），仅读不写
2. **SharedFolders**：列表时用 Sharing API，指定路径后挂载并降级为普通模式
3. **统一错误**：`errNotSupportedInSharedMode` 作为所有不支持操作的哨兵错误

### 11.4 文件路径回退模式

1. **检测先行**：`getFileMetadata` 确认 root 是否为文件
2. **路径收敛**：`path.Dir` 回退到父目录，`.` → `""` 特殊处理
3. **协议约定**：通过 `fs.ErrorIsFile` 与上层通信，保持 Fs 接口一致性

### 11.5 分页列表两阶段

1. **统一入口**：`List` → `list.WithListP` → `ListP` 标准化调用
2. **流式处理**：`list.NewHelper` 缓冲 + 回调，避免全量加载

### 11.6 重试边界三层防护

1. **排除层**：`shouldRetryExclude` 过滤绝对不可重试错误
2. **适配层**：`shouldRetry` 处理 Dropbox 特定错误（速率限制、最终一致性）
3. **通用层**：`fserrors.ShouldRetry` 处理网络等通用错误

### 11.7 统一传输接入点

**所有 Dropbox API 调用必经之路**：
```go
err = f.pacer.Call(func() (bool, error) {
    result, err = f.srv.SomeAPICall(args)
    return shouldRetry(ctx, err)  // ← 边界判定接入点
})
```

这种设计确保：
- 所有调用自动获得速率控制
- 所有调用自动获得重试机制
- 错误分类统一处理
- 并发连接数全局控制
- 重试日志统一记录
- 各模式分支（正常/SharedFiles/SharedFolders）共享相同的传输基础设施

---

## 十二、NewFs 完整决策流程图

```
NewFs(ctx, name, root, m)
    │
    ├─ 1. 解析 Options
    ├─ 2. 创建 OAuth 客户端
    ├─ 3. 初始化 Fs + Pacer + Batcher
    ├─ 4. 创建 SDK 客户端 (srv/sharing/users/team)
    │
    ├─ 5. SharedFiles 分支
    │    ├─ setRoot(root)
    │    ├─ root == "" → return (f, nil)
    │    ├─ findSharedFile(root)
    │    ├─ setRoot("")  // 强制重置
    │    ├─ 找到 → return (f, fs.ErrorIsFile)
    │    └─ 没找到 → return (f, nil)
    │
    ├─ 6. SharedFolders 分支
    │    ├─ setRoot(root)
    │    ├─ root == "" → return (f, nil)  // 后续列表共享文件夹
    │    ├─ findSharedFolder(dir)  // 遍历查找 ID
    │    ├─ mountSharedFolder(id)  // 挂载到命名空间
    │    │    └─ 已挂载错误 → 忽略
    │    ├─ f.opt.SharedFolders = false  // 降级为普通模式
    │    └─ ↓ 继续执行后续步骤
    │
    ├─ 7. 命名空间设置
    │    ├─ RootNsid != "" → f.ns = RootNsid
    │    └─ strings.HasPrefix(root, "/")
    │         ├─ users.GetCurrentAccount()
    │         ├─ TeamRootInfo → f.ns = TeamRootNamespaceId
    │         └─ UserRootInfo → f.ns = UserRootNamespaceId
    │
    ├─ 8. setRoot(root)  // 最终设置路径
    │
    ├─ 9. 根路径文件检测
    │    └─ f.root != ""
    │         ├─ getFileMetadata(slashRoot)  // pacer.Call 接入
    │         ├─ 是文件
    │         │    ├─ newRoot = path.Dir(f.root)
    │         │    ├─ newRoot == "." → newRoot = ""
    │         │    ├─ setRoot(newRoot)
    │         │    └─ return (f, fs.ErrorIsFile)
    │         └─ 不是文件 → 继续
    │
    └─ 10. return (f, nil)  // 正常目录 Fs
```

---

## 十三、各模式能力矩阵

| 能力 | 正常模式 | SharedFiles | SharedFolders (列表) | SharedFolders (挂载后) |
|------|----------|-------------|----------------------|-----------------------|
| List 目录 | ✅ Files API | ✅ Sharing API | ✅ Sharing API | ✅ Files API |
| NewObject | ✅ GetMetadata | ✅ 遍历查找 | ❌ (返回 Dir) | ✅ GetMetadata |
| Open 下载 | ✅ Download | ✅ GetSharedLinkFile | ❌ | ✅ Download |
| Put 上传 | ✅ Upload | ❌ NoRetryError | ❌ NoRetryError | ✅ Upload |
| Update 更新 | ✅ UploadSession | ❌ NoRetryError | ❌ NoRetryError | ✅ UploadSession |
| Mkdir | ✅ CreateFolder | ❌ NoRetryError | ❌ NoRetryError | ✅ CreateFolder |
| Remove | ✅ DeleteV2 | ❌ NoRetryError | ❌ NoRetryError | ✅ DeleteV2 |
| Copy 服务端 | ✅ CopyV2 | ❌ | ❌ | ✅ CopyV2 |
| Move 服务端 | ✅ MoveV2 | ❌ | ❌ | ✅ MoveV2 |
| Hash 哈希 | ✅ DropboxHash | ❌ NoRetryError | ❌ | ✅ DropboxHash |
| 命名空间 | ✅ headerGenerator | ✅ (不影响) | ✅ (不影响) | ✅ headerGenerator |
| Pacer 重试 | ✅ | ✅ | ✅ | ✅ |
| list 缓冲 | ✅ list.NewHelper | ✅ list.NewHelper | ✅ list.NewHelper | ✅ list.NewHelper |
