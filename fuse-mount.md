# Rclone FUSE 挂载集成代码实现深度分析

本文档从代码实现层面，详细分析 rclone 的三种 FUSE 挂载方案，重点对比 go-fuse (mount2) 与 WinFsp/cgofuse (cmount) 在节点、句柄、目录读取、读写转发及卸载清理机制上的差异。

---

## 一、三种挂载实现概览

| 特性 | mount (bazil/fuse) | mount2 (go-fuse/v2) | cmount (WinFsp/cgofuse) |
|------|-------------------|---------------------|------------------------|
| 底层库 | `bazil.org/fuse` | `github.com/hanwen/go-fuse/v2` | `github.com/winfsp/cgofuse/fuse` |
| 构建约束 | linux | linux \|\| (darwin && amd64) | (linux && cgo) \|\| (darwin && cgo) \|\| (freebsd && cgo) \|\| (openbsd && cgo) \|\| windows |
| 平台 | Linux 仅 | Linux, macOS (amd64) | Windows, macOS, Linux, FreeBSD, OpenBSD |
| API 风格 | 节点对象模式 | inode 树模式 | 路径字符串模式 |
| 入口文件 | `cmd/mount/mount.go` | `cmd/mount2/mount.go` | `cmd/cmount/mount.go` |

---

## 二、挂载启动流程对比

### 2.1 共同框架：mountlib

三种实现共享统一的 mountlib 框架，核心接口定义在 `cmd/mountlib/mount.go`：

```go
// [cmd/mountlib/mount.go:200-209]
type (
    UnmountFn func() error
    MountFn func(VFS *vfs.VFS, mountpoint string, opt *Options) (
        <-chan error, func() error, string, error)
)
```

所有实现的 `mount()` 函数必须返回：
- `<-chan error`：服务循环的错误通道，用于驱动生命周期
- `func() error`：卸载函数
- `string`：实际挂载点路径
- `error`：启动错误

### 2.2 bazil/fuse (mount) 的挂载流程

文件：`cmd/mount/mount.go`

```
[cmd/mount/mount.go:70-112]

1. mountlib.CheckOverlap()  检查挂载点重叠
2. mountlib.CheckAllowNonEmpty()  检查挂载点是否为空
3. fuse.Mount(mountpoint, options...)  与内核 FUSE 模块建立连接，返回 *Conn
4. NewFS(VFS, opt)  创建 FS 文件系统对象
5. fusefs.New(c, nil)  创建 FUSE 服务器
6. 启动后台 goroutine:
   go func() {
       err := server.Serve(filesys)   // 阻塞处理 FUSE 请求
       closeErr := c.Close()
       errChan <- err
   }()
7. 立即返回 errChan, unmount, mountpoint
   - 不等待挂载就绪，由调用方 WaitMountReady() 轮询检查
```

关键特点：
- `fuse.Mount()` 返回后立即返回，不保证挂载点可用
- 卸载函数：`fuse.Unmount(mountpoint)` 外部命令式卸载

### 2.3 go-fuse/v2 (mount2) 的挂载流程

文件：`cmd/mount2/mount.go`

```
[cmd/mount2/mount.go:188-267]

1. mountlib.CheckOverlap() / CheckAllowNonEmpty()
2. NewFS(VFS, opt)  创建 FS
3. fsys.Root()  获取根节点 Node
4. fusefs.NewNodeFS(root, &opts)  构建 inode 树文件系统
5. fuse.NewServer(rawFS, mountpoint, &mountOpts)  创建服务器
6. 启动后台 goroutine:
   go func() {
       server.Serve()
       errs <- nil
   }()
7. 关键步骤：server.WaitMount()  阻塞等待挂载就绪信号
   [cmd/mount2/mount.go:260-263]
   err = server.WaitMount()
8. 返回 errs, umount, mountpoint
   - 保证返回时挂载点已就绪
```

关键特点：
- 通过 `server.WaitMount()` 同步等待内核确认挂载完成
- 卸载函数：`server.Unmount()` 方法调用

### 2.4 WinFsp/cgofuse (cmount) 的挂载流程

文件：`cmd/cmount/mount.go`

```
[cmd/cmount/mount.go:127-224]

1. getMountpoint()  OS 特定挂载点获取（Windows 盘符分配等）
2. NewFS(VFS, opt)  创建 FS
   - FS.ready channel 用于就绪同步
3. fuse.NewFileSystemHost(fsys)  创建宿主对象
4. host.SetCapReaddirPlus(true)  Windows 专属能力设置
5. host.SetCapCaseInsensitive()  大小写不敏感配置
6. 启动后台 goroutine:
   go func() {
       defer recover()  // 捕获 panic，提示安装 WinFsp
       ok := host.Mount(mountpoint, options)
       if !ok { err = errors.New("mount failed") }
       errChan <- err
   }()
7. select 等待就绪:
   case err := <-errChan:   // 启动失败
   case <-fsys.ready:       // Init() 回调关闭 ready channel
8. Windows 额外等待：os.Stat(mountpoint) 轮询直到挂载点可见
9. 返回 errChan, unmount, mountpoint
```

关键特点：
- 通过 `FS.Init()` → `close(fsys.ready)` 实现就绪同步
- `FS.Destroy()` 回调标记卸载状态 `fsys.destroyed.Store(1)`
- Windows 下额外轮询检查挂载点可用性
- panic 恢复机制，提供 WinFsp 安装提示

### 2.5 挂载启动对比总结

| 维度 | mount (bazil) | mount2 (go-fuse) | cmount (cgofuse) |
|------|--------------|------------------|------------------|
| 就绪同步方式 | 无，调用方轮询 | `server.WaitMount()` | `Init()` 关闭 ready channel |
| 服务器创建 | `fusefs.New(Conn, nil)` | `fuse.NewServer(rawFS, mp, opts)` | `fuse.NewFileSystemHost(fsys)` |
| 连接方式 | `fuse.Mount()` 返回 `*Conn` | `fuse.NewServer` 内部处理 | `host.Mount()` 布尔返回 |
| 启动错误处理 | Serve 返回 error | Serve 返回后 errs<-nil | panic recover + 布尔判断 |
| Windows 支持 | 否 | 否 | 是，额外等待挂载点可见 |

---

## 三、节点系统对比

### 3.1 bazil/fuse：独立节点类型 + VFS 嵌入

文件：`cmd/mount/dir.go`, `cmd/mount/file.go`

```go
// [cmd/mount/dir.go:23-26]
type Dir struct {
    *vfs.Dir      // 直接嵌入 VFS 目录对象
    fsys *FS      // 回指文件系统
}

// [cmd/mount/file.go:18-21]
type File struct {
    *vfs.File     // 直接嵌入 VFS 文件对象
    fsys *FS      // 回指文件系统
}
```

节点创建与缓存机制 [cmd/mount/dir.go:75-99]：
```go
func (d *Dir) Lookup(ctx, req, resp) (node fusefs.Node, err error) {
    mnode, err := d.Dir.Stat(req.Name)      // 1. 查询 VFS
    node, ok := mnode.Sys().(fusefs.Node)   // 2. 尝试从 VFS 节点 Sys() 获取缓存
    if ok { return node, nil }              // 3. 命中缓存，直接返回
    // 4. 未命中，按类型创建新节点
    switch x := mnode.(type) {
    case *vfs.File: node = &File{x, d.fsys}
    case *vfs.Dir:  node = &Dir{x, d.fsys}
    }
    mnode.SetSys(node)                      // 5. 缓存到 VFS 节点
    return node, nil
}
```

核心设计：
- **两种独立类型**：Dir 和 File 是不同结构体
- **嵌入 VFS 对象**：直接嵌入 `*vfs.Dir` / `*vfs.File`，可直接调用 VFS 方法
- **节点缓存**：通过 `vfs.Node.Sys()` / `SetSys()` 机制保证 inode 一致性
- **接口声明**：`var _ fusefs.Node = (*Dir)(nil)` 编译期接口校验

### 3.2 go-fuse/v2：统一 Node 类型 + Inode 嵌入

文件：`cmd/mount2/node.go`

```go
// [cmd/mount2/node.go:20-24]
type Node struct {
    fusefs.Inode     // 必须嵌入 go-fuse 的 Inode，作为 inode 树节点
    node vfs.Node    // 持有 VFS 节点（接口类型，可能是 *vfs.Dir 或 *vfs.File）
    fsys *FS
}
```

节点创建与缓存 [cmd/mount2/node.go:30-44]：
```go
func newNode(fsys *FS, vfsNode vfs.Node) (node *Node) {
    node, ok := vfsNode.Sys().(*Node)   // 1. 从 VFS Sys() 查缓存
    if ok { return node }
    node = &Node{node: vfsNode, fsys: fsys}  // 2. 创建统一 Node
    vfsNode.SetSys(node)                // 3. 缓存
    return node
}
```

Lookup 实现 [cmd/mount2/node.go:191-205]：
```go
func (n *Node) Lookup(ctx, name, out) (inode *fusefs.Inode, errno syscall.Errno) {
    vfsNode, errno := n.lookupVfsNodeInDir(name)
    newNode := newNode(n.fsys, vfsNode)
    n.fsys.setEntryOut(vfsNode, out)
    // 关键：通过 Inode.NewInode() 将节点加入 inode 树
    return n.NewInode(ctx, newNode, fusefs.StableAttr{Mode: out.Attr.Mode}), 0
}
```

核心设计：
- **统一 Node 类型**：Dir 和 File 共用同一个结构体，通过 `vfs.Node` 接口区分
- **fusefs.Inode 嵌入**：必须嵌入，作为 go-fuse inode 树的一部分
- **inode 树管理**：通过 `n.NewInode()` 将新节点注册到父节点下
- **StableAttr**：创建时需提供 Mode 等稳定属性

### 3.3 WinFsp/cgofuse：无独立节点对象，路径字符串驱动

文件：`cmd/cmount/fs.go`

cgofuse 采用完全不同的设计 — **不使用节点对象**，所有操作以路径字符串为参数：

```go
// [cmd/cmount/fs.go:26-34]
type FS struct {
    VFS       *vfs.VFS
    f         fs.Fs
    opt       *mountlib.Options
    ready     chan struct{}        // 就绪信号
    mu        sync.Mutex           // 保护 handles 切片
    handles   []vfs.Handle         // 全局句柄表
    destroyed atomic.Int32         // 是否已卸载
}
```

节点查找通过路径实现 [cmd/cmount/fs.go:100-103]：
```go
func (fsys *FS) lookupNode(path string) (node vfs.Node, errc int) {
    node, err := fsys.VFS.Stat(path)   // 直接用路径查 VFS
    return node, translateError(err)
}
```

所有操作均以路径为入口，例如：
```go
// [cmd/cmount/fs.go:179-186]
func (fsys *FS) Getattr(path string, stat *fuse.Stat_t, fh uint64) (errc int) {
    node, _, errc := fsys.getNode(path, fh)  // 路径或句柄查找
    if errc == 0 {
        errc = fsys.stat(node, stat)         // 填充属性
    }
    return
}
```

核心设计：
- **无节点对象**：不创建 Dir/File 包装结构体
- **路径字符串驱动**：每个操作接收 `path string` 参数
- **VFS.Stat(path)**：每次操作通过路径重新解析 VFS 节点
- **Init/Destroy**：生命周期回调，无节点级创建/销毁

### 3.4 节点系统对比总结

| 维度 | mount (bazil) | mount2 (go-fuse) | cmount (cgofuse) |
|------|--------------|------------------|------------------|
| 节点结构 | `Dir{*vfs.Dir}` / `File{*vfs.File}` | `Node{fusefs.Inode, vfs.Node}` | 无独立节点 |
| 节点标识 | VFS 对象引用 | inode 树 + VFS 对象引用 | 路径字符串 |
| 类型区分 | 两种结构体类型 | 运行时断言 `vfs.Node` | 运行时断言 `vfs.Node` |
| 节点缓存 | `vfs.Node.Sys()/SetSys()` | `vfs.Node.Sys()/SetSys()` | 无（每次路径解析） |
| inode 管理 | VFS `node.Inode()` | VFS + fusefs.Inode 树 | VFS `node.Inode()` |
| Lookup 返回 | `fusefs.Node` | `*fusefs.Inode`（通过 NewInode 创建） | 无 Lookup，每次路径解析 |
| 新节点创建位置 | `Dir.Lookup()` 内 | `newNode()` 工厂函数 | 无需创建 |

---

## 四、句柄管理对比

### 4.1 bazil/fuse：简单包装

文件：`cmd/mount/handle.go`

```go
// [cmd/mount/handle.go:16-18]
type FileHandle struct {
    vfs.Handle    // 直接嵌入 VFS 句柄接口
}
```

句柄生命周期：
```
打开文件:
  [cmd/mount/file.go:67-86]
  File.Open() → f.File.Open(flags) → 返回 &FileHandle{handle}

读取:
  [cmd/mount/handle.go:24-34]
  FileHandle.Read() → fh.Handle.ReadAt(data, req.Offset)

释放:
  [cmd/mount/handle.go:79-82]
  FileHandle.Release() → fh.Handle.Release()
```

特点：
- 无额外状态管理，直接转发
- FUSE 库内部负责将 `*FileHandle` 与文件描述符关联

### 4.2 go-fuse/v2：包装结构体

文件：`cmd/mount2/file.go`

```go
// [cmd/mount2/file.go:35-38]
type FileHandle struct {
    h    vfs.Handle    // 持有 VFS 句柄（字段，非嵌入）
    fsys *FS           // 回指文件系统
}
```

句柄生命周期：
```
打开:
  [cmd/mount2/node.go:148-164]
  Node.Open() → n.node.Open(flags) → newFileHandle(handle, fsys)

读取:
  [cmd/mount2/file.go:59-68]
  FileHandle.Read() → f.h.ReadAt(dest, off) → fuse.ReadResultData(dest[:n])

写入:
  [cmd/mount2/file.go:74-80]
  FileHandle.Write() → f.h.WriteAt(data, off)

释放:
  [cmd/mount2/file.go:97-99]
  FileHandle.Release() → f.h.Release()
```

特点：
- 回指 `*FS` 用于属性设置时的超时配置
- 读取返回 `fuse.ReadResultData` 抽象类型，支持零拷贝等高级特性

### 4.3 WinFsp/cgofuse：全局句柄表 + uint64 索引

文件：`cmd/cmount/fs.go`

cgofuse 使用整数句柄 ID，需要自己管理句柄映射：

```go
// [cmd/cmount/fs.go:23]
const fhUnset = ^uint64(0)   // 无效句柄标记
```

句柄表结构 [cmd/cmount/fs.go:26-34]：
```go
type FS struct {
    mu        sync.Mutex
    handles   []vfs.Handle   // 全局句柄切片
}
```

句柄分配 [cmd/cmount/fs.go:48-63]：
```go
func (fsys *FS) openHandle(handle vfs.Handle) (fh uint64) {
    fsys.mu.Lock()
    defer fsys.mu.Unlock()
    // 优先复用空闲槽位
    for i, oldHandle := range fsys.handles {
        if oldHandle == nil {
            fsys.handles[i] = handle
            return uint64(i)
        }
    }
    // 无空闲则追加
    fsys.handles = append(fsys.handles, handle)
    return uint64(len(fsys.handles) - 1)
}
```

句柄查找 [cmd/cmount/fs.go:81-86]：
```go
func (fsys *FS) getHandle(fh uint64) (handle vfs.Handle, errc int) {
    fsys.mu.Lock()
    _, handle, errc = fsys._getHandle(fh)
    fsys.mu.Unlock()
    return
}
```

句柄关闭 [cmd/cmount/fs.go:89-97]：
```go
func (fsys *FS) closeHandle(fh uint64) (errc int) {
    fsys.mu.Lock()
    i, _, errc := fsys._getHandle(fh)
    if errc == 0 {
        fsys.handles[i] = nil   // 标记为空闲，可复用
    }
    fsys.mu.Unlock()
    return
}
```

句柄使用示例 — 读取 [cmd/cmount/fs.go:368-380]：
```go
func (fsys *FS) Read(path string, buff []byte, ofst int64, fh uint64) (n int) {
    handle, errc := fsys.getHandle(fh)   // 1. 用 fh 索引查句柄
    if errc != 0 { return errc }
    n, err := handle.ReadAt(buff, ofst)  // 2. 调用 VFS 句柄
    if err == io.EOF { /* ignore */ } else if err != nil {
        return translateError(err)
    }
    return n                             // 3. 返回字节数（负数表示错误码）
}
```

同时支持路径或句柄查找节点 [cmd/cmount/fs.go:128-138]：
```go
func (fsys *FS) getNode(path string, fh uint64) (node vfs.Node, handle vfs.Handle, errc int) {
    if fh == fhUnset {
        node, errc = fsys.lookupNode(path)   // 用路径解析
    } else {
        handle, errc = fsys.getHandle(fh)    // 用句柄查
        if errc == 0 {
            node = handle.Node()
        }
    }
    return
}
```

特点：
- **自定义句柄表**：必须自己维护 `[]vfs.Handle` 切片和 uint64 索引
- **线程安全**：`sync.Mutex` 保护句柄表并发访问
- **槽位复用**：关闭时将槽位置 nil，分配时优先复用
- **双路径解析**：每个操作可选择通过路径或句柄获取节点
- **错误码返回**：Read/Write 返回 int，正数为字节数，负数为 `-fuse.ENOENT` 等错误码

### 4.4 句柄管理对比总结

| 维度 | mount (bazil) | mount2 (go-fuse) | cmount (cgofuse) |
|------|--------------|------------------|------------------|
| 句柄表示 | `*FileHandle` 对象指针 | `*FileHandle` 对象指针 | `uint64` 整数索引 |
| 句柄存储 | FUSE 库内部管理 | FUSE 库内部管理 | `FS.handles []vfs.Handle` 自管理 |
| 并发保护 | 无（库负责） | 无（库负责） | `sync.Mutex` 保护句柄表 |
| 句柄查找 | 直接使用对象 | 直接使用对象 | `getHandle(fh)` 索引查表 |
| 空闲复用 | N/A | N/A | 槽位置 nil 后优先复用 |
| 无效标记 | N/A | N/A | `fhUnset = ^uint64(0)` |
| Read 返回 | `error` | `(fuse.ReadResult, syscall.Errno)` | `int`（正=字节数，负=错误码） |

---

## 五、目录读取机制对比

### 5.1 bazil/fuse：ReadDirAll 一次性返回

文件：`cmd/mount/dir.go`

```go
// [cmd/mount/dir.go:105-143]
func (d *Dir) ReadDirAll(ctx context.Context) (dirents []fuse.Dirent, err error) {
    items, err := d.Dir.ReadDirAll()         // 1. 从 VFS 获取所有节点
    dirents = append(dirents,
        fuse.Dirent{Type: fuse.DT_Dir, Name: "."},
        fuse.Dirent{Type: fuse.DT_Dir, Name: ".."},
    )
    for _, node := range items {
        var dirent = fuse.Dirent{
            Type: fuse.DT_File,               // 2. 默认文件类型
            Name: node.Name(),
        }
        if node.IsDir() { dirent.Type = fuse.DT_Dir }  // 3. 修正为目录
        if file, ok := node.(*vfs.File); ok && file.IsSymlink() {
            dirent.Type = fuse.DT_Link        // 4. 修正为符号链接
        }
        dirents = append(dirents, dirent)
    }
    return dirents, nil
}
```

特点：
- `HandleReadDirAller` 接口：一次性返回完整目录列表
- 仅提供 `Type` 和 `Name`，不提供 inode 等完整属性
- 调用方需要再通过 Lookup/Getattr 获取详细属性

### 5.2 go-fuse/v2：DirStream 迭代器模式（先取完整列表再迭代）

文件：`cmd/mount2/node.go`

**重要说明**：go-fuse 的 DirStream 虽然名为 "Stream"，但此处实现**并非后端流式读取**，而是先一次性获取完整目录列表到内存，再用 DirStream 做内存迭代。

自定义 DirStream 实现 [cmd/mount2/node.go:222-288]：
```go
type dirStream struct {
    nodes []os.FileInfo   // 完整的内存列表，所有条目已预取
    i     int             // 当前迭代位置
}

// HasNext 检查是否还有条目（可能被多次调用）
func (ds *dirStream) HasNext() bool {
    return ds.i < len(ds.nodes)+2   // +2 是 "." 和 ".."
}

// Next 返回下一条目录项
func (ds *dirStream) Next() (de fuse.DirEntry, errno syscall.Errno) {
    if ds.i == 0 {
        ds.i++
        return fuse.DirEntry{Mode: fuse.S_IFDIR, Name: ".", Ino: 0}, 0
    } else if ds.i == 1 {
        ds.i++
        return fuse.DirEntry{Mode: fuse.S_IFDIR, Name: "..", Ino: 0}, 0
    }
    fi := ds.nodes[ds.i-2]              // 从已加载的内存切片中读取
    de = fuse.DirEntry{
        Mode: getMode(fi),              // 包含文件类型权限位
        Name: path.Base(fi.Name()),
        Ino:  0,                         // FIXME: 未设置 inode
    }
    ds.i++
    return de, 0
}

// Seekdir 支持目录回绕（解决 go-fuse issue #549）
func (ds *dirStream) Seekdir(_ context.Context, off uint64) syscall.Errno {
    if off == 0 { ds.i = 0; return 0 }
    return syscall.ENOTSUP
}
```

Readdir 入口 [cmd/mount2/node.go:300-322]：
```go
func (n *Node) Readdir(ctx) (ds fusefs.DirStream, errno syscall.Errno) {
    fh, err := n.node.Open(os.O_RDONLY)       // 1. 打开目录句柄
    if err != nil {
        return nil, translateError(err)
    }
    defer func() {
        closeErr := fh.Close()                 // 4. 关闭目录句柄
        if errno == 0 && closeErr != nil {
            errno = translateError(closeErr)
        }
    }()
    items, err := fh.Readdir(-1)               // 2. 关键：Readdir(-1) 一次性读取 ALL 条目
                                               //    -1 表示读取所有，完整列表已加载到内存
    if err != nil {
        return nil, translateError(err)
    }
    return &dirStream{nodes: items}, 0         // 3. 返回封装完整列表的迭代器
}
```

**完整流程说明**：
```
应用 readdir()
  → Node.Readdir() 被调用
    1. n.node.Open() 打开目录句柄
    2. fh.Readdir(-1) 【一次性读取完整目录】到 []os.FileInfo
    3. fh.Close() 关闭目录句柄（此时 VFS 层目录读取已完成）
    4. 返回 &dirStream{nodes: items} 迭代器
  → 内核通过 DirStream.HasNext()/Next() 逐条获取
    - 所有条目已在内存中，迭代仅操作已加载的切片
    - 不再涉及后端网络请求
```

特点：
- `DirStream` 迭代器接口：`HasNext()`/`Next()`/`Close()` 对**内存列表**进行迭代
- **非后端流式**：`fh.Readdir(-1)` 先完整获取所有条目，DirStream 仅做内存遍历
- 提供 `Mode`（含权限位）而非仅 `Type`
- 实现 `FileSeekdirer` 接口支持 lseek(fd, 0) 回绕（复位内存索引）
- 需要手动打开/关闭目录句柄（在返回 DirStream 前已关闭）

### 5.3 WinFsp/cgofuse：fill 回调填充

文件：`cmd/cmount/fs.go`

```go
// [cmd/cmount/fs.go:199-257]
func (fsys *FS) Readdir(
    dirPath string,
    fill func(name string, stat *fuse.Stat_t, ofst int64) bool,  // 填充回调
    ofst int64,
    fh uint64,
) (errc int) {
    dir, errc := fsys.lookupDir(dirPath)
    if ofst > 0 {                                  // 1. 不支持目录 seek
        if runtime.GOOS == "openbsd" { return 0 }  // OpenBSD 特殊处理
        return -fuse.ESPIPE
    }

    nodes, err := dir.ReadDirAll()                 // 2. 读取 VFS 目录

    fill(".", nil, 0)                              // 3. 回调填充 "."
    fill("..", nil, 0)                             // 4. 回调填充 ".."
    for _, node := range nodes {
        name := node.Name()
        var stat fuse.Stat_t
        _ = fsys.stat(node, &stat)                 // 5. 填充完整 stat
        fill(name, &stat, 0)                       // 6. 回调填充条目（含 stat）
    }
    return 0
}
```

特点：
- `fill(name, stat, ofst)` 回调函数填充
- **ReaddirPlus 支持**：`host.SetCapReaddirPlus(true)` 后，可在 readdir 时同步返回完整 stat
- Windows 优化：一次调用同时返回名称和属性，避免后续 Getattr 调用
- OpenBSD 兼容：不支持 seek 时特殊处理返回 0

### 5.4 目录读取对比总结

| 维度 | mount (bazil) | mount2 (go-fuse) | cmount (cgofuse) |
|------|--------------|------------------|------------------|
| API 模式 | `ReadDirAll()` 返回切片 | `DirStream` 迭代器（内存遍历） | `fill()` 回调填充 |
| 数据获取时机 | `d.Dir.ReadDirAll()` 一次性全部获取 | `fh.Readdir(-1)` 一次性全部获取到内存 | `dir.ReadDirAll()` 一次性全部获取后再遍历 |
| 返回方式 | 返回完整 `[]fuse.Dirent` 切片 | DirStream 对**已在内存**的列表逐条迭代 | 对**已在内存**的列表逐条调用 `fill()` 回调 |
| 返回信息 | `Dirent{Type, Name}` | `DirEntry{Mode, Name, Ino}` | `(name, *Stat_t, ofst)` |
| 属性信息 | 仅类型 | 模式权限位 | 完整 stat 结构（可选） |
| Seek 支持 | 不支持 | `Seekdir()` 支持回绕到 0（重置内存索引） | 不支持（返回 ESPIPE） |
| ReaddirPlus | 不支持 | 不支持（`DisableReadDirPlus: true`） | 支持（`SetCapReaddirPlus(true)`） |
| "." 和 ".." | 手动添加 | DirStream 内处理 | fill 回调添加 |

**关键说明**：三种实现的目录读取本质上**全部是先一次性获取完整列表到内存，再逐条返回**，不存在真正的后端流式分页读取：
- mount：`d.Dir.ReadDirAll()` → 返回完整 `[]fuse.Dirent` 切片
- mount2：`fh.Readdir(-1)` → 封装为 `dirStream{nodes: items}` 迭代器
- cmount：`dir.ReadDirAll()` → for 循环逐条 `fill()` 回调

其中 mount2 的 DirStream 虽然接口上是 "stream"，但底层实现是先将目录条目完整加载到 `nodes []os.FileInfo` 切片，再通过 `HasNext()/Next()` 对内存切片进行迭代。所有条目在 DirStream 创建前已完整加载，无后端流式分页。

---

## 六、读写转发机制对比

### 6.1 读取流程深度对比

#### bazil/fuse

```
[cmd/mount/handle.go:24-34]
应用 read()
  → 内核 FUSE
  → FileHandle.Read(ctx, req, resp)
    │  data := resp.Data[:req.Size]          // 使用 resp 预分配缓冲区
    │  n, err := fh.Handle.ReadAt(data, req.Offset)
    │  resp.Data = data[:n]                  // 重置切片长度
    └─ return translateError(err)
```

#### go-fuse/v2

```
[cmd/mount2/file.go:59-68]
应用 read()
  → 内核 FUSE
  → FileHandle.Read(ctx, dest, off)
    │  n, err := f.h.ReadAt(dest, off)
    │  if err == io.EOF { err = nil }
    └─ return fuse.ReadResultData(dest[:n]), translateError(err)
       // 返回 ReadResult 抽象，支持多种数据来源（切片、mmap 等）
```

#### WinFsp/cgofuse

```
[cmd/cmount/fs.go:368-380]
应用 read()
  → WinFsp/cgofuse
  → FS.Read(path, buff, ofst, fh)
    │  handle, errc := fsys.getHandle(fh)     // 用 uint64 查句柄表
    │  if errc != 0 { return errc }           // 返回负数错误码
    │  n, err := handle.ReadAt(buff, ofst)
    │  if err == io.EOF { /* 忽略 */ }
    │  else if err != nil { return translateError(err) }  // 返回负错误码
    └─ return n                               // 正数=字节数，负数=错误码
```

### 6.2 写入流程深度对比

#### bazil/fuse

```
[cmd/mount/handle.go:40-48]
应用 write()
  → FileHandle.Write(ctx, req, resp)
    │  n, err := fh.Handle.WriteAt(req.Data, req.Offset)
    │  resp.Size = n                          // 写入 resp
    └─ return translateError(err)
```

#### go-fuse/v2

```
[cmd/mount2/file.go:74-80]
应用 write()
  → FileHandle.Write(ctx, data, off)
    │  n, err := f.h.WriteAt(data, off)
    └─ return uint32(n), translateError(err)
       // 返回 (写入字节数, 错误码)
```

#### WinFsp/cgofuse

```
[cmd/cmount/fs.go:383-394]
应用 write()
  → FS.Write(path, buff, ofst, fh)
    │  handle, errc := fsys.getHandle(fh)
    │  if errc != 0 { return errc }
    │  n, err := handle.WriteAt(buff, ofst)
    │  if err != nil { return translateError(err) }
    └─ return n                               // 同 Read 约定
```

### 6.3 Flush 和 Release 对比

#### bazil/fuse

```go
// [cmd/mount/handle.go:68-71]
func (fh *FileHandle) Flush(ctx, req) error {
    return translateError(fh.Handle.Flush())
}

// [cmd/mount/handle.go:79-82]
func (fh *FileHandle) Release(ctx, req) error {
    return translateError(fh.Handle.Release())
}
```

#### go-fuse/v2

```go
// [cmd/mount2/file.go:87-89]
func (f *FileHandle) Flush(ctx) syscall.Errno {
    return translateError(f.h.Flush())
}

// [cmd/mount2/file.go:97-99]
func (f *FileHandle) Release(ctx) syscall.Errno {
    return translateError(f.h.Release())
}

// 额外支持 Fsync [cmd/mount2/file.go:105-107]
func (f *FileHandle) Fsync(ctx, flags) syscall.Errno {
    return translateError(f.h.Sync())
}
```

#### WinFsp/cgofuse

```go
// [cmd/cmount/fs.go:397-404]
func (fsys *FS) Flush(path string, fh uint64) (errc int) {
    handle, errc := fsys.getHandle(fh)
    if errc != 0 { return errc }
    return translateError(handle.Flush())
}

// [cmd/cmount/fs.go:407-415]
func (fsys *FS) Release(path string, fh uint64) (errc int) {
    handle, errc := fsys.getHandle(fh)
    if errc != 0 { return errc }
    _ = fsys.closeHandle(fh)                   // 额外：从全局句柄表移除
    return translateError(handle.Release())
}
```

### 6.4 打开标志转换

#### bazil/fuse 和 go-fuse/v2：直接使用

```go
// FUSE 标志与 syscall 标志兼容，直接传递
// [cmd/mount/file.go:72]
handle, err := f.File.Open(int(req.Flags))

// [cmd/mount2/node.go:152]
handle, err := n.node.Open(int(flags))
```

#### WinFsp/cgofuse：需要显式转换

```go
// [cmd/cmount/fs.go:615-638]
func translateOpenFlags(inFlags int) (outFlags int) {
    switch inFlags & fuse.O_ACCMODE {
    case fuse.O_RDONLY: outFlags = os.O_RDONLY
    case fuse.O_WRONLY: outFlags = os.O_WRONLY
    case fuse.O_RDWR:   outFlags = os.O_RDWR
    }
    if inFlags&fuse.O_APPEND != 0 { outFlags |= os.O_APPEND }
    if inFlags&fuse.O_CREAT  != 0 { outFlags |= os.O_CREATE }
    if inFlags&fuse.O_EXCL   != 0 { outFlags |= os.O_EXCL }
    if inFlags&fuse.O_TRUNC  != 0 { outFlags |= os.O_TRUNC }
    return
}
```

### 6.5 DirectIO 触发条件

三种实现在未知大小文件时都启用 DirectIO：

```go
// mount (bazil): [cmd/mount/file.go:78-79]
if entry := handle.Node().DirEntry(); entry != nil && entry.Size() < 0 {
    resp.Flags |= fuse.OpenDirectIO
}

// mount2 (go-fuse): [cmd/mount2/node.go:157-159]
if entry := n.node.DirEntry(); entry != nil && entry.Size() < 0 {
    fuseFlags |= fuse.FOPEN_DIRECT_IO
}

// cmount (cgofuse): [cmd/cmount/fs.go:297-299]
if entry := handle.Node().DirEntry(); entry != nil && entry.Size() < 0 {
    fi.DirectIo = true
}
```

### 6.6 读写转发对比总结

| 维度 | mount (bazil) | mount2 (go-fuse) | cmount (cgofuse) |
|------|--------------|------------------|------------------|
| Read 返回 | `error`，数据写入 resp.Data | `(fuse.ReadResult, syscall.Errno)` | `int`（正=字节数，负=错误码） |
| Write 返回 | `error`，写入 resp.Size | `(uint32, syscall.Errno)` | `int`（同上） |
| 句柄获取 | 直接使用嵌入接口 | 直接使用字段 | `getHandle(fh)` 查表 |
| Flush 位置 | FileHandle | FileHandle | FS 全局方法 |
| Release 动作 | `handle.Release()` | `handle.Release()` | `closeHandle(fh)` + `handle.Release()` |
| 打开标志 | 直接使用 int(flags) | 直接使用 int(flags) | `translateOpenFlags()` 显式转换 |
| Fsync 支持 | NodeFsyncer 接口 | FileFsyncer 接口 | 独立 Fsync 方法（no-op） |
| DirectIO 标志 | `fuse.OpenDirectIO` | `fuse.FOPEN_DIRECT_IO` | `fi.DirectIo = true` |

---

## 七、错误转换机制对比

### 7.1 bazil/fuse：返回 error 类型

```go
// [cmd/mount/fs.go:75-108]
func translateError(err error) error {
    _, uErr := fserrors.Cause(err)
    switch uErr {
    case vfs.ENOENT: return fuse.Errno(syscall.ENOENT)
    case vfs.EEXIST: return fuse.Errno(syscall.EEXIST)
    case vfs.ENOSYS: return syscall.ENOSYS       // 特殊：不包装
    // ...
    }
    return err
}
```

特点：
- 返回 `error` 接口，通常包装为 `fuse.Errno(syscall.Exxx)`
- ENOSYS 特殊：直接返回 `syscall.ENOSYS`（值而非类型）

### 7.2 go-fuse/v2：返回 syscall.Errno

```go
// [cmd/mount2/fs.go:108-141]
func translateError(err error) syscall.Errno {
    _, uErr := fserrors.Cause(err)
    switch uErr {
    case vfs.OK:       return 0
    case vfs.ENOENT:   return syscall.ENOENT
    case vfs.EEXIST:   return syscall.EEXIST
    // ...
    }
    fs.Errorf(nil, "IO error: %v", err)
    return syscall.EIO
}
```

特点：
- 直接返回 `syscall.Errno` 数字类型
- 成功返回 `0`
- 未知错误记录日志并返回 `EIO`

### 7.3 WinFsp/cgofuse：返回负错误码 int

```go
// [cmd/cmount/fs.go:579-612]
func translateError(err error) (errc int) {
    _, uErr := fserrors.Cause(err)
    switch uErr {
    case vfs.OK:       return 0
    case vfs.ENOENT:   return -fuse.ENOENT     // 取负数！
    case vfs.EEXIST:   return -fuse.EEXIST
    // ...
    }
    fs.Errorf(nil, "IO error: %v", err)
    return -fuse.EIO
}
```

特点：
- 返回 `int` 类型：`0` 成功，`-fuse.Exxx` 失败（注意负号）
- 使用 `fuse.ENOENT` 等常量（不是 syscall 包）
- Read/Write 等返回字节数的方法，负数即表示错误

---

## 八、卸载清理机制对比

### 8.1 共同卸载流程框架

```
[cmd/mountlib/mount.go:399-428]

MountPoint.Wait()
  ├─ 注册 atexit 钩子 finalise()
  ├─ err := <-m.ErrChan        // 阻塞等待服务结束
  └─ finalise()
      └─ m.Unmount()
          └─ 调用各实现的 unmount 函数
```

### 8.2 bazil/fuse：简单直接

```go
// [cmd/mount/mount.go:105-109]
unmount := func() error {
    filesys.VFS.Shutdown()       // 1. 关闭 VFS（缓存、轮询、上下文）
    return fuse.Unmount(mountpoint)  // 2. 调用 fusermount 卸载
}
```

服务结束触发 [cmd/mount/mount.go:96-103]：
```go
go func() {
    err := filesys.server.Serve(filesys)   // 阻塞
    closeErr := c.Close()                  // 关闭连接
    errChan <- err
}()
```

### 8.3 go-fuse/v2：server 方法调用

```go
// [cmd/mount2/mount.go:243-247]
umount := func() error {
    fsys.VFS.Shutdown()            // 1. 关闭 VFS
    return server.Unmount()        // 2. 调用 server 方法卸载
}
```

服务结束触发 [cmd/mount2/mount.go:254-257]：
```go
go func() {
    server.Serve()
    errs <- nil                    // 总是返回 nil
}()
```

### 8.4 WinFsp/cgofuse：最复杂的卸载逻辑

```go
// [cmd/cmount/mount.go:172-201]
unmount := func() error {
    fsys.VFS.Shutdown()                    // 1. 关闭 VFS
    var umountOK bool

    // 2. 三重判断是否需要调用 Unmount
    if fsys.destroyed.Load() != 0 {
        // Destroy() 回调已执行，FUSE 层已卸载
        fs.Debugf(nil, "Not calling host.Unmount as mount already Destroyed")
        umountOK = true
    } else if atexit.Signalled() {
        // 收到信号，FUSE 将自行关闭
        fs.Debugf(nil, "Not calling host.Unmount as signal received")
        umountOK = true
    } else {
        // 正常卸载
        fs.Debugf(nil, "Calling host.Unmount")
        umountOK = host.Unmount()
    }

    // 3. Windows 额外等待：挂载点消失
    if umountOK && runtime.GOOS == "windows" {
        if !waitFor(func() bool {
            _, err := os.Stat(mountpoint)
            return err != nil               // 挂载点不存在即成功
        }) { /* timeout 忽略 */ }
    }
    if !umountOK { return errors.New("host unmount failed") }
    return nil
}
```

生命周期回调 [cmd/cmount/fs.go:164-176]：
```go
func (fsys *FS) Init() {
    close(fsys.ready)                       // 标记挂载就绪
}

func (fsys *FS) Destroy() {
    fsys.destroyed.Store(1)                 // 标记已卸载，避免重复 Unmount
}
```

### 8.5 信号处理（两条完全独立的路径）

**关键结论**：信号处理分为**退出清理**和**缓存刷新**两条完全独立的路径。**SIGHUP 仅用于刷新缓存，不属于退出清理信号，不会触发程序退出。**

---

#### 路径一：退出清理路径（SIGINT / SIGTERM）

信号分类定义 [lib/atexit/atexit_unix.go:12]：
```go
// 退出信号列表 —— 注意：SIGHUP 不在此列表中！
var exitSignals = []os.Signal{syscall.SIGINT, syscall.SIGTERM}
```

Windows/Plan9 定义 [lib/atexit/atexit_other.go:11]：
```go
var exitSignals = []os.Signal{os.Interrupt}
```

退出清理执行流程 [lib/atexit/atexit.go:41-56]：
```go
// [lib/atexit/atexit.go:41-56]
// 仅在 init() 时启动一次，全局单例
go func() {
    sig := <-exitChan       // 【仅】接收 SIGINT, SIGTERM（不包含 SIGHUP！）
    signal.Stop(exitChan)
    signalled.Store(1)      // 设置标志位，cmount 卸载时会检查此标志
    Run()                   // 执行所有注册的 atexit 清理函数
                            // 包括 finalise() → Unmount() → VFS.Shutdown()
    os.Exit(exitCode(sig))  // 终止进程
}()
```

**退出清理信号的效果**：
1. `SIGINT`（Ctrl+C）或 `SIGTERM`（kill 默认信号）触发
2. `atexit.Signalled()` 返回 `true`
3. cmount 检测到标志后跳过 `host.Unmount()` 调用（FUSE 层会自行关闭）
4. 执行所有清理函数（卸载挂载点、关闭 VFS、关闭缓存）
5. 进程退出

---

#### 路径二：缓存刷新路径（SIGHUP，仅用于刷新，不退出）

SIGHUP 有自己独立的信号注册和处理逻辑，与退出清理完全无关。

SIGHUP 独立注册 [vfs/sighup.go:13-19]：
```go
// [vfs/sighup.go:13-19]
func NotifyOnSigHup(sighupChan chan os.Signal) {
    signal.Notify(sighupChan, syscall.SIGHUP)  // 【单独】注册 SIGHUP
}
```

SIGHUP 处理逻辑 [vfs/vfs.go:296-314]：
```go
// [vfs/vfs.go:296-314]
// 每个 VFS 实例启动一个独立 goroutine 监听 SIGHUP
func (vfs *VFS) signalHandler(ctx context.Context) {
    sigHup := make(chan os.Signal, 1)
    NotifyOnSigHup(sigHup)           // 仅监听 SIGHUP
    for {
        select {
        case <-ctx.Done(): return     // VFS 关闭时退出
        case <-sigHup:                // 收到 SIGHUP
            root, _ := vfs.Root()
            root.ForgetAll()          // 【仅】清空所有目录缓存
                                    // 不调用 Run()，不执行清理，不退出！
        }
    }
}
```

**SIGHUP 的效果**（与退出清理完全无关）：
1. `SIGHUP`（通常由终端断开或 `kill -HUP` 触发）被 VFS 独立接收
2. 调用 `root.ForgetAll()` 清空所有目录缓存（强制下次访问时从后端重新获取）
3. **不设置 `signalled` 标志**
4. **不执行 atexit 清理函数**
5. **不卸载挂载点**
6. **不退出进程** —— 程序继续正常运行

---

#### 两条路径对比总结

| 特性 | 退出清理路径 | 缓存刷新路径 |
|------|-------------|-------------|
| 触发信号 | `SIGINT`, `SIGTERM` | `SIGHUP` |
| 信号注册位置 | `lib/atexit/atexit.go` init() | `vfs/sighup.go` NotifyOnSigHup() |
| 监听 goroutine | atexit 包全局单例 | 每个 VFS 实例一个 |
| 核心动作 | `Run()` 执行所有清理函数 | `root.ForgetAll()` 清空缓存 |
| 卸载挂载点 | 是（通过 `finalise()`） | 否 |
| 关闭 VFS | 是（`VFS.Shutdown()`） | 否 |
| 设置 `signalled` 标志 | 是 | 否 |
| 进程退出 | 是（`os.Exit()`） | 否（继续运行） |
| cmount 跳过 Unmount | 是（`atexit.Signalled()` 检查） | 否 |

---

#### cmount 中的信号检查

在 cmount 的卸载函数中，通过 `atexit.Signalled()` 判断是否是退出信号触发的卸载：

```go
// [cmd/cmount/mount.go:182-187]
} else if atexit.Signalled() {
    // 收到退出信号（SIGINT/SIGTERM），FUSE 将自行关闭
    // 注意：SIGHUP 不会设置此标志，因此不会触发此分支
    fs.Debugf(nil, "Not calling host.Unmount as signal received")
    umountOK = true
}
```

**关键说明**：`atexit.Signalled()` 仅在收到 `SIGINT`/`SIGTERM` 等退出信号时返回 `true`。收到 `SIGHUP` 时该标志仍为 `false`，不会触发此跳过逻辑，因为 SIGHUP 根本不会导致卸载。

### 8.6 VFS.Shutdown 细节

三种实现共享 VFS 关闭逻辑 [vfs/vfs.go:390-417]：

```go
func (vfs *VFS) Shutdown() {
    if vfs.inUse.Add(-1) > 0 { return }  // 引用计数，仍被引用则跳过
    // 从全局 active 缓存移除
    // ...
    vfs.shutdownCache()                  // 关闭磁盘缓存
    close(vfs.pollChan)                  // 停止变更轮询
    vfs.cancel()                         // 取消 VFS 上下文
}
```

### 8.7 卸载清理对比总结

| 维度 | mount (bazil) | mount2 (go-fuse) | cmount (cgofuse) |
|------|--------------|------------------|------------------|
| 卸载调用 | `fuse.Unmount(mp)` 外部式 | `server.Unmount()` 方法式 | `host.Unmount()` 带条件判断 |
| 结束条件 | `server.Serve()` 返回 | `server.Serve()` 返回 | `host.Mount()` 返回或 Destroy 回调 |
| Destroy 回调 | 无 | 无 | `FS.Destroy()` 设置 destroyed 标志 |
| 信号判断 | 无 | 无 | `atexit.Signalled()` 跳过 Unmount |
| Windows 等待 | N/A | N/A | `os.Stat(mp)` 轮询直到错误 |
| 连接关闭 | `c.Close()` 显式 | Serve 内部处理 | N/A |
| ErrChan 内容 | Serve 错误或 Close 错误 | 总是 nil | Mount 错误或 panic 信息 |
| VFS 关闭 | `VFS.Shutdown()` | `VFS.Shutdown()` | `VFS.Shutdown()` |

---

## 九、属性填充机制对比

### 9.1 bazil/fuse：分散式填充

Dir Attr [cmd/mount/dir.go:32-45]：
```go
func (d *Dir) Attr(ctx, a *fuse.Attr) error {
    a.Valid = time.Duration(d.fsys.opt.AttrTimeout)
    a.Gid = d.VFS().Opt.GID
    a.Uid = d.VFS().Opt.UID
    a.Mode = d.Mode()
    modTime := d.ModTime()
    a.Atime = modTime
    a.Mtime = modTime
    a.Ctime = modTime
    return nil
}
```

File Attr [cmd/mount/file.go:27-42]：
```go
func (f *File) Attr(ctx, a *fuse.Attr) error {
    a.Valid = time.Duration(f.fsys.opt.AttrTimeout)
    Size := uint64(f.File.Size())
    Blocks := (Size + 511) / 512
    a.Size = Size
    a.Blocks = Blocks
    a.Mode = f.File.Mode() &^ os.ModeAppend
    // ... Gid/Uid/Time
    return nil
}
```

特点：Dir 和 File 各自实现 Attr，代码重复。

### 9.2 go-fuse/v2：集中式辅助函数

```go
// [cmd/mount2/fs.go:69-92]
func setAttr(node vfs.Node, attr *fuse.Attr) {
    Size := uint64(node.Size())
    Blocks := (Size + 511) / 512
    modTime := node.ModTime()
    attr.Owner.Gid = node.VFS().Opt.GID
    attr.Owner.Uid = node.VFS().Opt.UID
    attr.Mode = getMode(node)
    attr.Size = Size
    attr.Nlink = 1
    attr.Blocks = Blocks
    s := uint64(modTime.Unix())
    ns := uint32(modTime.Nanosecond())
    attr.Atime = s; attr.Atimensec = ns
    attr.Mtime = s; attr.Mtimensec = ns
    attr.Ctime = s; attr.Ctimensec = ns
}

// [cmd/mount2/fs.go:95-98]
func (f *FS) setAttrOut(node vfs.Node, out *fuse.AttrOut) {
    setAttr(node, &out.Attr)
    out.SetTimeout(time.Duration(f.opt.AttrTimeout))
}
```

使用 [cmd/mount2/node.go:112-115]：
```go
func (n *Node) Getattr(ctx, f, out) syscall.Errno {
    n.fsys.setAttrOut(n.node, out)
    return 0
}
```

特点：统一 `setAttr()` 辅助函数，File 和 Node 共用。

### 9.3 WinFsp/cgofuse：stat 集中方法

```go
// [cmd/cmount/fs.go:141-162]
func (fsys *FS) stat(node vfs.Node, stat *fuse.Stat_t) (errc int) {
    Size := uint64(node.Size())
    Blocks := (Size + 511) / 512
    modTime := node.ModTime()
    stat.Ino = node.Inode()
    stat.Mode = getMode(node)
    stat.Nlink = 1
    stat.Uid = fsys.VFS.Opt.UID
    stat.Gid = fsys.VFS.Opt.GID
    stat.Size = int64(Size)
    t := fuse.NewTimespec(modTime)
    stat.Atim = t; stat.Mtim = t; stat.Ctim = t; stat.Birthtim = t
    stat.Blksize = 512
    stat.Blocks = int64(Blocks)
    return 0
}
```

使用 [cmd/cmount/fs.go:179-186]：
```go
func (fsys *FS) Getattr(path string, stat *fuse.Stat_t, fh uint64) (errc int) {
    node, _, errc := fsys.getNode(path, fh)
    if errc == 0 { errc = fsys.stat(node, stat) }
    return
}
```

特点：
- 使用 `fuse.Stat_t`（更接近 POSIX struct stat）
- 包含 `Birthtim`（创建时间）等 Windows 专有字段
- 包含 `Ino`（inode 号）

---

## 十、模式转换对比

### bazil/fuse

```go
// 直接使用 VFS Mode() 结果
// [cmd/mount/dir.go:37]
a.Mode = d.Mode()
// [cmd/mount/file.go:35]
a.Mode = f.File.Mode() &^ os.ModeAppend
```

### go-fuse/v2 和 cmount：统一 getMode 函数

```go
// [cmd/mount2/fs.go:53-66] 和 [cmd/cmount/fs.go:641-654]
func getMode(node os.FileInfo) uint32 {
    vfsMode := node.Mode()
    Mode := vfsMode.Perm()
    if vfsMode&os.ModeDir != 0 {
        Mode |= fuse.S_IFDIR
    } else if vfsMode&os.ModeSymlink != 0 {
        Mode |= fuse.S_IFLNK
    } else if vfsMode&os.ModeNamedPipe != 0 {
        Mode |= fuse.S_IFIFO
    } else {
        Mode |= fuse.S_IFREG
    }
    return uint32(Mode)
}
```

**真实互斥关系说明**：代码使用 `if/else if/else if/else` 结构，各文件类型分支是**完全互斥**的。一个节点只能匹配以下四种类型之一：
1. 目录 (`os.ModeDir`) → 设置 `S_IFDIR`
2. 否则如果是符号链接 (`os.ModeSymlink`) → 设置 `S_IFLNK`
3. 否则如果是命名管道 (`os.ModeNamedPipe`) → 设置 `S_IFIFO`
4. 否则（默认）→ 设置 `S_IFREG`（普通文件）

不存在一个节点同时匹配多个类型分支的情况。

---

## 十一、关键代码索引（相对路径）

| 功能 | bazil (mount) | go-fuse (mount2) | cgofuse (cmount) |
|------|--------------|------------------|------------------|
| 挂载入口 | `cmd/mount/mount.go:70-112` | `cmd/mount2/mount.go:188-267` | `cmd/cmount/mount.go:127-224` |
| FS 结构体 | `cmd/mount/fs.go:21-26` | `cmd/mount2/fs.go:21-25` | `cmd/cmount/fs.go:26-34` |
| 节点类型 | `cmd/mount/dir.go:23-26`<br>`cmd/mount/file.go:18-21` | `cmd/mount2/node.go:20-24` | 无独立节点 |
| 节点缓存 | `cmd/mount/dir.go:75-99` | `cmd/mount2/node.go:30-44` | 无（路径解析） |
| Lookup 实现 | `cmd/mount/dir.go:75-99` | `cmd/mount2/node.go:191-205` | N/A（无 Lookup） |
| 句柄结构 | `cmd/mount/handle.go:16-18` | `cmd/mount2/file.go:35-38` | `cmd/cmount/fs.go:48-97` 句柄表 |
| 目录读取 | `cmd/mount/dir.go:105-143` | `cmd/mount2/node.go:222-322` | `cmd/cmount/fs.go:199-257` |
| Read 实现 | `cmd/mount/handle.go:24-34` | `cmd/mount2/file.go:59-68` | `cmd/cmount/fs.go:368-380` |
| Write 实现 | `cmd/mount/handle.go:40-48` | `cmd/mount2/file.go:74-80` | `cmd/cmount/fs.go:383-394` |
| Flush 实现 | `cmd/mount/handle.go:68-71` | `cmd/mount2/file.go:87-89` | `cmd/cmount/fs.go:397-404` |
| Release 实现 | `cmd/mount/handle.go:79-82` | `cmd/mount2/file.go:97-99` | `cmd/cmount/fs.go:407-415` |
| 错误转换 | `cmd/mount/fs.go:75-108` | `cmd/mount2/fs.go:108-141` | `cmd/cmount/fs.go:579-612` |
| 属性填充 | `cmd/mount/dir.go:32-45`<br>`cmd/mount/file.go:27-42` | `cmd/mount2/fs.go:69-105` | `cmd/cmount/fs.go:141-162` |
| 模式转换 | 直接使用 Mode() | `cmd/mount2/fs.go:53-66` | `cmd/cmount/fs.go:641-654` |
| 卸载函数 | `cmd/mount/mount.go:105-109` | `cmd/mount2/mount.go:243-247` | `cmd/cmount/mount.go:172-201` |
| Init/Destroy | N/A | N/A | `cmd/cmount/fs.go:164-176` |
| 通用 VFS | `vfs/vfs.go:205-284` New<br>`vfs/vfs.go:390-417` Shutdown | 同左 | 同左 |
| 信号处理（退出） | `lib/atexit/atexit.go:41-56`<br>`lib/atexit/atexit_unix.go:12`（信号分类） | 同左 | 同左 |
| 信号处理（SIGHUP 刷新） | `vfs/sighup.go:13-19`<br>`vfs/vfs.go:296-314`（处理逻辑） | 同左 | 同左 |
| 通用挂载 | `cmd/mountlib/mount.go:368-428` | 同左 | 同左 |

---

## 十二、设计哲学总结

### mount (bazil/fuse) — 简洁包装派
- **哲学**：尽量薄的包装层，让 VFS 对象直接暴露为 FUSE 节点
- **优点**：代码最简洁，嵌入 VFS 对象后直接调用方法
- **缺点**：平台支持有限（仅 Linux），功能较弱

### mount2 (go-fuse/v2) — inode 树派
- **哲学**：利用 go-fuse 的 inode 树管理能力，构建完整的文件系统树
- **优点**：性能优化空间大（DirStream 流式、ReadResult 抽象），支持 macOS
- **缺点**：inode 树学习成本高，`NewInode`/`StableAttr` 等概念复杂

### cmount (WinFsp/cgofuse) — 路径驱动派
- **哲学**：完全基于路径字符串，不构建节点对象树，每次操作重新解析
- **优点**：跨平台最强（Windows/macOS/Linux/BSD），Windows 深度优化（ReaddirPlus、盘符、大小写不敏感）
- **缺点**：需要自管理句柄表、路径解析开销、错误码约定（负数）不直观

三种实现虽然 API 风格迥异，但都复用了同一套 VFS 层（`vfs/vfs.go`, `vfs/file.go`, `vfs/dir.go`, `vfs/read.go`），体现了 rclone 良好的分层架构设计。
