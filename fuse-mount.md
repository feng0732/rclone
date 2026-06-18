# Rclone FUSE 挂载集成分析

## 一、架构概览

Rclone 的 FUSE 挂载采用分层架构设计，各层职责清晰，协作有序：

```
用户空间进程
    │
    ▼
┌─────────────────────────────────────────────────┐
│              命令行入口 (cmd/mount)            │
│  mount / mount2 / cmount 三种实现               │
├─────────────────────────────────────────────────┤
│              mountlib 通用挂载库                │
│  命令注册、参数解析、生命周期管理                │
├─────────────────────────────────────────────────┤
│              VFS 虚拟文件系统层                 │
│  目录缓存、文件句柄管理、后端抽象                │
├─────────────────────────────────────────────────┤
│              FUSE 库 (bazil/go-fuse)            │
│  内核 FUSE 协议用户态绑定                       │
└─────────────────────────────────────────────────┘
    │
    ▼
 内核 FUSE 模块
    │
    ▼
 用户态文件系统操作
```

### 核心模块与文件

| 模块 | 主要文件 | 职责 |
|------|---------|------|
| mountlib | [mountlib/mount.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go) | 通用挂载逻辑、命令注册、生命周期管理 |
| mount (bazil) | [mount/mount.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/mount.go) | 基于 bazil.org/fuse 的挂载实现 |
| mount2 (go-fuse) | [mount2/mount.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount2/mount.go) | 基于 hanwen/go-fuse/v2 的挂载实现 |
| VFS 核心 | [vfs/vfs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/vfs.go) | 虚拟文件系统核心 |
| VFS 文件 | [vfs/file.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/file.go) | VFS 文件节点实现 |
| VFS 目录 | [vfs/dir.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/dir.go) | VFS 目录节点实现 |
| VFS 读句柄 | [vfs/read.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/read.go) | 文件读取句柄实现 |
| atexit | [atexit/atexit.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/lib/atexit/atexit.go) | 退出清理钩子 |

---

## 二、挂载启动流程

### 2.1 命令注册与入口

挂载命令通过 `mountlib.NewMountCommand` 统一注册，三种实现（mount/mount2/cmount）共享相同的命令框架：

```go
// [mount/mount.go:17-20](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/mount.go#L17-L20)
func init() {
    mountlib.NewMountCommand("mount", false, mount)
    mountlib.AddRc("mount", mount)
}
```

核心接口定义在 [mountlib/mount.go:200-209](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go#L200-L209)：

```go
type (
    // UnmountFn 卸载函数
    UnmountFn func() error
    // MountFn 挂载函数，返回错误通道、卸载函数、实际挂载点、错误
    MountFn func(VFS *vfs.VFS, mountpoint string, opt *Options) (
        <-chan error, func() error, string, error)
)
```

### 2.2 挂载启动完整流程

```
用户执行: rclone mount remote:path /mnt/remote
    │
    ▼
1. 命令行参数解析
   [mountlib/mount.go:281-365](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go#L281-L365)
   NewMountCommand 创建 cobra.Command
   ├─ 检查参数 (remote:path 和 mountpoint)
   ├─ 处理 --daemon 后台运行选项
   └─ 创建 MountPoint 结构体
    │
    ▼
2. 创建 MountPoint 并调用 Mount()
   [mountlib/mount.go:368-397](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go#L368-L397)
   MountPoint.Mount()
   ├─ 设置卷名和设备名
   ├─ 如需 daemon，调用 daemonize.StartDaemon()
   ├─ 创建 VFS 实例: vfs.New(ctx, f, &vfsOpt)
   │  [vfs/vfs.go:205-284](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/vfs.go#L205-L284)
   │  ├─ 检查是否可复用已有 VFS 实例
   │  ├─ 创建根目录节点
   │  ├─ 启动目录变更轮询 (ChangeNotify)
   │  ├─ 启动信号处理器 (SIGHUP 刷新缓存)
   │  └─ 设置缓存模式
   └─ 调用具体实现的 MountFn (如 mount/mount.go 中的 mount)
    │
    ▼
3. 具体 FUSE 后端挂载 (以 bazil/fuse 为例)
   [mount/mount.go:70-112](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/mount.go#L70-L112)
   mount() 函数
   ├─ 检查挂载点重叠 CheckOverlap
   ├─ 检查挂载点是否为空 CheckAllowNonEmpty
   ├─ 配置 FUSE 挂载选项 mountOptions()
   │  [mount/mount.go:23-62](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/mount.go#L23-L62)
   │  ├─ MaxReadahead、Subtype、FSName
   │  ├─ AllowOther、DefaultPermissions
   │  ├─ ReadOnly、WritebackCache
   │  └─ DaemonTimeout (macOS)
   ├─ 调用 fuse.Mount() 与内核建立连接
   ├─ 创建 FS 文件系统实例: NewFS(VFS, opt)
   ├─ 创建 FUSE 服务器: fusefs.New(c, nil)
   ├─ 启动后台 goroutine 运行 Serve()
   │  [mount/mount.go:96-103](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/mount.go#L96-L103)
   │  go func() {
   │      err := filesys.server.Serve(filesys)
   │      closeErr := c.Close()
   │      errChan <- err  // 服务结束时发送错误
   │  }()
   ├─ 返回 errChan, unmount 函数, mountpoint
   └─ unmount 函数封装了 VFS.Shutdown() 和 fuse.Unmount()
    │
    ▼
4. 等待挂载就绪 (daemon 模式)
   [mountlib/mount.go:340-349](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go#L340-L349)
   WaitMountReady() 轮询检查挂载状态
    │
    ▼
5. 进入 Wait() 循环
   [mountlib/mount.go:399-428](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go#L399-L428)
   MountPoint.Wait()
   ├─ 注册 atexit 钩子 finalise()
   │  └─ 确保退出时调用 Unmount()
   ├─ 阻塞等待 errChan 消息
   └─ 收到消息后执行 finalise() 卸载
```

### 2.3 关键数据结构

**MountPoint** [mountlib/mount.go:212-222](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go#L212-L222)：
```go
type MountPoint struct {
    MountPoint string      // 挂载点路径
    MountedOn  time.Time   // 挂载时间
    MountOpt   Options     // 挂载选项
    VFSOpt     vfscommon.Options  // VFS 选项
    Fs         fs.Fs       // 后端文件系统
    VFS        *vfs.VFS    // VFS 实例
    MountFn    MountFn     // 挂载函数
    UnmountFn  UnmountFn   // 卸载函数
    ErrChan    <-chan error // 错误通道
}
```

**VFS** [vfs/vfs.go:178-191](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/vfs.go#L178-L191)：
```go
type VFS struct {
    f           fs.Fs           // 后端文件系统
    ctx         context.Context
    root        *Dir            // 根目录
    Opt         vfscommon.Options
    cache       *vfscache.Cache // 磁盘缓存
    pollChan    chan time.Duration // 变更轮询通道
    inUse       atomic.Int32    // 引用计数
}
```

---

## 三、文件操作转发机制

### 3.1 三层转发架构

文件操作从内核 FUSE 模块到后端存储经过三层转发：

```
内核 FUSE 请求
    │
    ▼ 1. FUSE 回调层 (mount/*.go)
    │  实现 bazil/fuse 接口，进行类型转换
    │  Dir / File / FileHandle
    │
    ▼ 2. VFS 抽象层 (vfs/*.go)
    │  后端无关的文件系统逻辑
    │  目录缓存、权限检查、句柄管理
    │  vfs.Dir / vfs.File / vfs.Handle
    │
    ▼ 3. 后端存储层 (backend/*/*.go)
    │  具体云存储实现
    │  fs.Fs / fs.Object
```

### 3.2 FUSE 回调层实现

**FS 根节点** [mount/fs.go:21-49](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/fs.go#L21-L49)：
```go
type FS struct {
    *vfs.VFS
    f      fs.Fs
    opt    *mountlib.Options
    server *fusefs.Server
}

// Root() 返回根目录节点
func (f *FS) Root() (node fusefs.Node, err error) {
    root, err := f.VFS.Root()
    if err != nil {
        return nil, translateError(err)
    }
    return &Dir{root, f}, nil
}
```

**Dir 目录节点** [mount/dir.go:23-26](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/dir.go#L23-L26)：
```go
type Dir struct {
    *vfs.Dir        // 嵌入 VFS 目录
    fsys *FS        // 回指文件系统
}
```

实现的 FUSE 接口包括：
- `Attr()` - 获取属性 [mount/dir.go:32-45](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/dir.go#L32-L45)
- `Lookup()` - 查找条目 [mount/dir.go:75-99](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/dir.go#L75-L99)
- `ReadDirAll()` - 读取目录 [mount/dir.go:105-143](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/dir.go#L105-L143)
- `Create()` - 创建文件 [mount/dir.go:148-163](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/dir.go#L148-L163)
- `Mkdir()` - 创建目录 [mount/dir.go:168-177](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/dir.go#L168-L177)
- `Remove()` - 删除 [mount/dir.go:184-191](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/dir.go#L184-L191)
- `Rename()` - 重命名 [mount/dir.go:206-225](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/dir.go#L206-L225)

**File 文件节点** [mount/file.go:18-21](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/file.go#L18-L21)：
```go
type File struct {
    *vfs.File       // 嵌入 VFS 文件
    fsys *FS        // 回指文件系统
}
```

实现的 FUSE 接口包括：
- `Attr()` - 获取属性 [mount/file.go:27-42](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/file.go#L27-L42)
- `Setattr()` - 设置属性 [mount/file.go:48-61](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/file.go#L48-L61)
- `Open()` - 打开文件 [mount/file.go:67-86](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/file.go#L67-L86)

**FileHandle 文件句柄** [mount/handle.go:16-18](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/handle.go#L16-L18)：
```go
type FileHandle struct {
    vfs.Handle      // 嵌入 VFS 句柄
}
```

实现的 FUSE 接口包括：
- `Read()` - 读取 [mount/handle.go:24-34](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/handle.go#L24-L34)
- `Write()` - 写入 [mount/handle.go:40-48](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/handle.go#L40-L48)
- `Flush()` - 刷新 [mount/handle.go:68-71](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/handle.go#L68-L71)
- `Release()` - 释放 [mount/handle.go:79-82](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/handle.go#L79-L82)

### 3.3 典型操作转发流程示例

#### 示例 1: 读取文件 (cat /mnt/remote/file.txt)

```
应用程序 read()
    │
    ▼ 内核
    ▼ FUSE 模块
    │
    ▼ 1. FileHandle.Read()
    │  [mount/handle.go:24-34](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/handle.go#L24-L34)
    │  func (fh *FileHandle) Read(ctx, req, resp) {
    │      data := resp.Data[:req.Size]
    │      n, err := fh.Handle.ReadAt(data, req.Offset)
    │      resp.Data = data[:n]
    │      return translateError(err)
    │  }
    │
    ▼ 2. ReadFileHandle.ReadAt()
    │  [vfs/read.go:213-217](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/read.go#L213-L217)
    │  调用内部 readAt()
    │  ├─ openPending() 延迟打开后端连接
    │  │  [vfs/read.go:73-89](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/read.go#L73-L89)
    │  │  └─ chunkedreader.New() 创建分块读取器
    │  ├─ 检查偏移是否需要 seek
    │  ├─ seek() 处理随机访问
    │  │  [vfs/read.go:116-168](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/read.go#L116-L168)
    │  │  ├─ 尝试 RangeSeeker 接口
    │  │  └─ 失败则重新打开连接
    │  ├─ io.ReadFull() 读取数据
    │  ├─ 重试机制 (lowLevelRetries)
    │  └─ 哈希校验
    │
    ▼ 3. chunkedreader 分块读取
    │  预读、缓存、并发流控
    │
    ▼ 4. 后端 fs.Object.Open()
    │  具体云存储 API 调用
    │
    ▼ 原路返回数据
```

#### 示例 2: 目录列表 (ls /mnt/remote/)

```
应用程序 readdir()
    │
    ▼ 内核 FUSE
    │
    ▼ 1. Dir.ReadDirAll()
    │  [mount/dir.go:105-143](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/dir.go#L105-L143)
    │  func (d *Dir) ReadDirAll(ctx) (dirents []fuse.Dirent, err) {
    │      items, err := d.Dir.ReadDirAll()  // 调用 VFS
    │      for _, node := range items {
    │          // 转换为 fuse.Dirent
    │          // 处理文件/目录/符号链接类型
    │      }
    │      return dirents, nil
    │  }
    │
    ▼ 2. vfs.Dir.ReadDirAll()
    │  [vfs/dir.go:1003-1019](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/dir.go#L1003-L1019)
    │  ├─ _readDir() 检查缓存有效性
    │  │  [vfs/dir.go:532-587](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/dir.go#L532-L587)
    │  │  ├─ 检查缓存是否过期 (DirCacheTime)
    │  │  ├─ 过期则调用 list.DirSorted() 从后端读取
    │  │  └─ _readDirFromEntries() 更新缓存
    │  └─ 返回排序后的节点列表
    │
    ▼ 3. 后端 fs.List()
    │  云存储列表 API
```

#### 示例 3: 错误转换

所有 VFS 错误通过 `translateError()` 转换为 FUSE 错误码：
[mount/fs.go:75-108](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/fs.go#L75-L108)

```go
func translateError(err error) error {
    _, uErr := fserrors.Cause(err)
    switch uErr {
    case vfs.ENOENT, fs.ErrorDirNotFound:
        return fuse.Errno(syscall.ENOENT)    // 不存在
    case vfs.EEXIST, fs.ErrorDirExists:
        return fuse.Errno(syscall.EEXIST)    // 已存在
    case vfs.EPERM, fs.ErrorPermissionDenied:
        return fuse.Errno(syscall.EPERM)     // 无权限
    case vfs.EROFS:
        return fuse.Errno(syscall.EROFS)     // 只读
    case vfs.ENOSYS, fs.ErrorNotImplemented:
        return syscall.ENOSYS                // 不支持
    // ... 更多映射
    }
    return err
}
```

### 3.4 节点缓存机制

为了保证 FUSE 节点的一致性（同一 VFS 节点必须返回同一 FUSE 节点），使用 `SetSys()`/`Sys()` 进行缓存：

[mount/dir.go:82-98](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/dir.go#L82-L98)
```go
func (d *Dir) Lookup(ctx, req, resp) (node fusefs.Node, err error) {
    mnode, err := d.Dir.Stat(req.Name)
    // 检查是否有缓存的 FUSE 节点
    node, ok := mnode.Sys().(fusefs.Node)
    if ok {
        return node, nil  // 返回缓存节点
    }
    // 创建新节点
    switch x := mnode.(type) {
    case *vfs.File:
        node = &File{x, d.fsys}
    case *vfs.Dir:
        node = &Dir{x, d.fsys}
    }
    mnode.SetSys(node)  // 缓存节点
    return node, nil
}
```

---

## 四、卸载清理流程

### 4.1 正常卸载流程

```
用户执行: fusermount -u /mnt/remote
或收到 Ctrl+C / SIGTERM
    │
    ▼
1. FUSE 服务终止
   [mount/mount.go:96-103](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/mount.go#L96-L103)
   server.Serve() 从阻塞中返回
   errChan <- err
    │
    ▼
2. MountPoint.Wait() 被唤醒
   [mountlib/mount.go:420-427](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go#L420-L427)
   err := <-m.ErrChan
   finalise()  // 执行清理
    │
    ▼
3. finalise() 卸载函数
   [mountlib/mount.go:403-416](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go#L403-L416)
   ├─ 检查挂载点是否仍由 rclone 挂载
   │  CheckMountReady(m.MountPoint)
   ├─ 调用 m.Unmount()
   │  └─ 调用注册的 UnmountFn
   │     [mount/mount.go:105-109](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/mount.go#L105-L109)
   │     unmount := func() error {
   │         filesys.VFS.Shutdown()  // 关闭 VFS
   │         return fuse.Unmount(mountpoint)  // 卸载 FUSE
   │     }
   └─ sync.Once 确保只执行一次
    │
    ▼
4. VFS.Shutdown() 深层清理
   [vfs/vfs.go:390-417](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/vfs.go#L390-L417)
   ├─ 引用计数检查 inUse.Add(-1)
   │  仍有引用则直接返回
   ├─ 从 active 缓存中移除 VFS 实例
   ├─ shutdownCache() 关闭磁盘缓存
   │  [vfs/vfs.go:381-386](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/vfs.go#L381-L386)
   │  └─ cancelCache() 取消缓存上下文
   ├─ 关闭 pollChan 停止变更轮询
   └─ cancel() 取消 VFS 上下文，终止所有后台 goroutine
```

### 4.2 信号处理与异常退出

#### atexit 机制 [atexit/atexit.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/lib/atexit/atexit.go)

```go
// 注册退出清理函数
fnHandle := atexit.Register(finalise)

// 信号处理 goroutine
// [atexit/atexit.go:41-56](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/lib/atexit/atexit.go#L41-L56)
go func() {
    sig := <-exitChan       // 等待 SIGINT, SIGTERM 等
    signal.Stop(exitChan)
    signalled.Store(1)
    Run()                   // 执行所有注册的清理函数
    os.Exit(exitCode(sig))
}()
```

注册时机在 [mountlib/mount.go:417](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go#L417)：
```go
func (m *MountPoint) Wait() error {
    // ...
    fnHandle := atexit.Register(finalise)
    defer atexit.Unregister(fnHandle)
    // ...
}
```

#### SIGHUP 缓存刷新

[vfs/vfs.go:296-314](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/vfs.go#L296-L314)
```go
func (vfs *VFS) signalHandler(ctx context.Context) {
    sigHup := make(chan os.Signal, 1)
    NotifyOnSigHup(sigHup)
    for {
        select {
        case <-ctx.Done():
            return
        case <-sigHup:
            root, _ := vfs.Root()
            root.ForgetAll()  // 清空所有目录缓存
        }
    }
}
```

### 4.3 Daemon 模式卸载

当使用 `--daemon` 时，父进程退出，子进程继续运行：

[mountlib/mount.go:328-352](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go#L328-L352)
```go
killDaemon := func(reason string) {
    killOnce.Do(func() {
        mountDaemon.Signal(os.Interrupt)  // 发送中断信号
    })
}

// 等待挂载就绪
if err == nil && Opt.DaemonWait > 0 {
    handle := atexit.Register(func() {
        killDaemon("Got interrupt")  // 父进程退出时杀死子进程
    })
    err = WaitMountReady(mnt.MountPoint, Opt.DaemonWait, mountDaemon)
    atexit.Unregister(handle)
}
```

### 4.4 RC API 卸载

通过远程控制 API 卸载：
[mountlib/rc.go:186-202](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/rc.go#L186-L202)
```go
func unMountRc(_ context.Context, in rc.Params) (out rc.Params, err error) {
    mountPoint, _ := in.GetString("mountPoint")
    mountInfo, found := liveMounts[mountPoint]
    if !found {
        return nil, errors.New("mount not found")
    }
    if err = mountInfo.Unmount(); err != nil {
        return nil, err
    }
    delete(liveMounts, mountPoint)
    return nil, nil
}
```

---

## 五、关键协作机制详解

### 5.1 错误通道 (ErrChan) 协作模式

```
挂载 goroutine
    │
    ├─ 创建 errChan := make(chan error, 1)
    ├─ go func() {
    │      err := server.Serve(filesys)  // 阻塞运行
    │      errChan <- err               // 服务终止时发送
    │   }()
    │
    ▼
Wait() 主 goroutine
    │
    ├─ 阻塞等待 err := <-errChan
    │  ├─ 正常卸载: err == nil
    │  ├─ 异常终止: err != nil
    │  └─ 外部卸载: err == nil (fuse.Unmount 触发)
    │
    └─ 执行卸载清理
```

### 5.2 挂载与卸载的状态机

```
┌─────────┐   Mount()     ┌────────────┐   Wait()     ┌────────────┐
│  INIT   │──────────────▶│  MOUNTING  │────────────▶│  MOUNTED   │
└─────────┘               └────────────┘              └──────┬─────┘
                                                             │
                        ┌────────────────────────────────────┘
                        │ 收到错误 / 信号 / 卸载命令
                        ▼
                   ┌────────────┐   finalise()   ┌──────────┐
                   │ UNMOUNTING │──────────────▶│  EXITED  │
                   └────────────┘                └──────────┘
```

### 5.3 三种挂载实现对比

| 特性 | mount (bazil/fuse) | mount2 (go-fuse/v2) | cmount (cgofuse) |
|------|-------------------|---------------------|------------------|
| 库 | bazil.org/fuse | github.com/hanwen/go-fuse/v2 | github.com/winfsp/cgofuse |
| 平台 | Linux | Linux, macOS | Windows, macOS, Linux |
| 节点接口 | fusefs.Node | fusefs.Inode | 各自定义 |
| 服务启动 | fuse.Mount + server.Serve | fuse.NewServer + server.Serve | Fuse.Mount |
| 卸载方式 | fuse.Unmount | server.Unmount | Fuse.Unmount |
| 高级特性 | 较少 | ID映射、缓存控制 | WinFsp 集成 |

### 5.4 目录缓存与一致性

VFS 层实现了智能目录缓存：
- `DirCacheTime` 控制缓存有效期
- `ChangeNotify` 后端主动通知（如支持）
- `PollInterval` 定期轮询刷新
- `ForgetAll()` 手动清空（SIGHUP）
- 虚拟条目（vAdd/vDel）跟踪本地修改

---

## 六、关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 挂载主入口 | [mountlib/mount.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go) | L368-L397 |
| bazil fuse 挂载 | [mount/mount.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/mount.go) | L70-L112 |
| go-fuse 挂载 | [mount2/mount.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount2/mount.go) | L188-L267 |
| VFS 创建 | [vfs/vfs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/vfs.go) | L205-L284 |
| VFS 关闭 | [vfs/vfs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/vfs.go) | L390-L417 |
| 卸载等待 | [mountlib/mount.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/mount.go) | L399-L428 |
| 错误转换 | [mount/fs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/fs.go) | L75-L108 |
| 文件读取 | [mount/handle.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/handle.go) | L24-L34 |
| 目录查找 | [mount/dir.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mount/dir.go) | L75-L99 |
| 信号处理 | [atexit/atexit.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/lib/atexit/atexit.go) | L41-L56 |
| VFS 信号处理 | [vfs/vfs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/vfs/vfs.go) | L296-L314 |
| RC 卸载 API | [mountlib/rc.go](file:///d:/fz/0601-2/solo-dogfeeding/code/53-rclone/cmd/mountlib/rc.go) | L186-L202 |

---

## 七、总结

Rclone 的 FUSE 挂载设计体现了清晰的分层架构和良好的协作模式：

1. **挂载启动**：`MountPoint.Mount()` → VFS 创建 → FUSE 库挂载 → 后台 Serve 循环
2. **操作转发**：FUSE 回调 → 类型包装节点 → VFS 抽象层 → 后端存储 API
3. **卸载清理**：ErrChan 唤醒 → atexit 钩子 → VFS.Shutdown() → FUSE.Unmount()

关键设计要点：
- **错误通道** 作为挂载生命周期的唯一同步点
- **atexit 机制** 确保异常退出时的资源清理
- **VFS 抽象层** 隔离了 FUSE 协议和后端存储差异
- **节点缓存** 保证了 FUSE inode 一致性
- **引用计数** 支持 VFS 实例复用和安全关闭
- **多挂载实现** 通过统一 MountFn 接口支持不同 FUSE 库
