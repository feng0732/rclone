# rclone mount2 NFS 实现：请求映射、句柄管理与 VFS 访问边界

## 1. 整体架构概览

rclone 的 NFS 服务实现分为三个清晰的层次，自上而下为：

```
NFS Client (操作系统内核)
      │
      ▼
┌─────────────────────────────────────────────┐
│  Layer 1: NFS 请求映射 (go-nfs + Handler)    │  cmd/serve/nfs/
│  - NFSv3 协议 RPC 解码                        │
│  - MOUNT / NFS 过程分发                        │
│  - 文件句柄 ↔ 路径映射                         │
└──────────────┬──────────────────────────────┘
               │ billy.Filesystem 接口
               ▼
┌─────────────────────────────────────────────┐
│  Layer 2: billy 适配层 (FS)                  │  cmd/serve/nfs/filesystem.go
│  - billy.Filesystem → vfs.VFS 调用转译       │
│  - 路径重写 (subFS root)                      │
│  - setSys() 填充 uid/gid/inode              │
└──────────────┬──────────────────────────────┘
               │ vfs.VFS 方法调用
               ▼
┌─────────────────────────────────────────────┐
│  Layer 3: VFS 层                             │  vfs/
│  - 虚拟文件系统核心                            │
│  - 读写缓存、目录树、Handle 生命周期            │
│  - 最终调用 fs.Fs 后端                        │
└─────────────────────────────────────────────┘
```

核心源码文件：

| 文件 | 职责 |
|------|------|
| [server.go](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/server.go) | NFS Server 生命周期：创建、监听、Serve |
| [handler.go](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/handler.go) | `nfs.Handler` 实现：Mount、ToHandle、FromHandle、HandleLimit |
| [filesystem.go](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/filesystem.go) | `billy.Filesystem` 适配器：将 billy 调用转译为 vfs 调用 |
| [cache.go](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/cache.go) | 句柄缓存：memory / disk / symlink 三种策略 |
| [symlink_cache_linux.go](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/symlink_cache_linux.go) | Linux 专用 symlink 缓存实现 |
| [nfs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/nfs.go) | CLI 入口与选项定义 |
| [nfsmount.go](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/nfsmount/nfsmount.go) | `rclone nfsmount` 命令：自动启动 NFS 服务器 + 本地 mount |

---

## 2. Layer 1：NFS 请求映射

### 2.1 协议栈

rclone 使用第三方库 [willscott/go-nfs](https://github.com/willscott/go-nfs) 处理 NFSv3 协议。该库负责：

1. **MOUNT 协议** — 客户端通过 `MOUNTPROC3_MNT` 请求获取初始文件句柄
2. **NFS 协议** — 客户端使用文件句柄发起 `NFSPROC3_GETATTR`、`NFSPROC3_READ`、`NFSPROC3_READDIR` 等操作
3. **RPC 消息编解码** — XDR 编解码、TCP 分帧

`go-nfs` 将所有协议细节封装后，通过 `nfs.Handler` 接口向 rclone 回调：

```go
// handler.go — Handler 实现的 nfs.Handler 接口方法
type Handler struct {
    vfs     *vfs.VFS
    opt     Options
    billyFS *FS
    Cache
}
```

### 2.2 MOUNT 请求处理

[handler.go#L66-L83](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/handler.go#L66-L83)

```go
func (h *Handler) Mount(ctx context.Context, conn net.Conn, req nfs.MountRequest) (
    status nfs.MountStatus, hndl billy.Filesystem, auths []nfs.AuthFlavor)
```

流程：

1. 客户端发送 `MountRequest`，其中 `Dirpath` 指定要挂载的路径（如 `/photos/2024`）
2. `path.Clean` 规范化路径，消除 `..` 等穿越片段
3. 若路径为 `/`（VFS 根），直接返回 `h.billyFS`
4. 否则通过 `h.vfs.Stat(cleaned)` 验证路径存在且为目录
5. 返回 `h.billyFS.subFS(cleaned)` — 一个以子路径为根的 billy 文件系统视图

**关键点**：NFS 客户端可通过在 mount 路径中指定子目录来挂载 VFS 的子树，无需额外配置。

### 2.3 NFS 过程 → billy 调用映射

`go-nfs` 内部将 NFS 过程分发到 `billy.Filesystem` 的对应方法：

| NFS 过程 | billy 方法 | 说明 |
|----------|-----------|------|
| GETATTR | `Stat()` | 获取文件属性 |
| LOOKUP | `Stat()` + handle 映射 | 目录项查找 |
| READ | `Open()` → `Read()` | 读取文件数据 |
| WRITE | `OpenFile()` → `Write()` | 写入文件数据 |
| CREATE | `Create()` | 创建新文件 |
| MKDIR | `MkdirAll()` | 创建目录 |
| REMOVE | `Remove()` | 删除文件 |
| RMDIR | `Remove()` | 删除目录 |
| RENAME | `Rename()` | 重命名 |
| READDIR / READDIRPLUS | `ReadDir()` | 读取目录内容 |
| READLINK | `Readlink()` | 读取符号链接 |
| SYMLINK | `Symlink()` | 创建符号链接 |
| SETATTR | `Chmod()` / `Chtimes()` / `Truncate()` | 修改文件属性 |
| FSSTAT | `FSStat()` | 文件系统统计信息 |

### 2.4 句柄在请求中的流转

```
NFS Client                   go-nfs                    Handler/Cache            billy FS / VFS
   │                           │                           │                         │
   │── MOUNT(path) ──────────►│                           │                         │
   │                          │── Mount() ──────────────►│                         │
   │                          │◄── billyFS ──────────────│                         │
   │                          │── ToHandle(billyFS,[]) ─►│                         │
   │                          │◄── fh (root handle) ─────│                         │
   │◄── fh ──────────────────│                           │                         │
   │                           │                           │                         │
   │── LOOKUP(fh, name) ────►│                           │                         │
   │                          │── FromHandle(fh) ───────►│                         │
   │                          │◄── (billyFS, path) ──────│                         │
   │                          │── billyFS.Stat(path) ─────────────────────────────►│
   │                          │── ToHandle(billyFS,path)►│                         │
   │                          │◄── newFh ────────────────│                         │
   │◄── newFh + attrs ───────│                           │                         │
   │                           │                           │                         │
   │── READ(fh, offset) ────►│                           │                         │
   │                          │── FromHandle(fh) ───────►│                         │
   │                          │◄── (billyFS, path) ──────│                         │
   │                          │── billyFS.Open(path) ─────────────────────────────►│
   │                          │── file.Read() ─────────────────────────────────────►│
   │◄── data ────────────────│                           │                         │
```

---

## 3. Layer 2：句柄管理（Handle Cache）

### 3.1 核心接口

[cache.go#L44-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/cache.go#L44-L58)

```go
type Cache interface {
    ToHandle(f billy.Filesystem, path []string) []byte
    FromHandle(fh []byte) (billy.Filesystem, []string, error)
    InvalidateHandle(fs billy.Filesystem, handle []byte) error
    HandleLimit() int
}
```

NFS 是有状态协议（客户端持有 opaque file handle），但 rclone 的 VFS 本身是路径驱动的。因此需要 **Cache** 将 `(billyFS, splitPath)` 二元组映射为不透明的 `[]byte` 句柄，并在后续请求中反查。

### 3.2 三种缓存策略

#### 策略一：memory（默认）

[cache.go#L66](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/cache.go#L66)

```go
inner = nfshelper.NewCachingHandler(h, h.opt.HandleLimit)
```

- 使用 `go-nfs/helpers` 提供的 `CachingHandler`
- 在内存中维护 `map[handleID] → (billyFS, splitPath)` 映射
- 句柄为自增 ID
- 服务器重启后所有句柄失效，客户端收到 `NFS3ERR_STALE`
- `HandleLimit` 控制最大缓存数（默认 1,000,000）

#### 策略二：disk

[cache.go#L67-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/cache.go#L67-L68)

```go
inner, err = newDiskHandler(h)
```

- 以 `MD5(fullPath)` 作为文件名，在磁盘上存储 `fullPath → 句柄` 的映射
- 磁盘文件内容为完整路径字符串
- 句柄为 MD5 哈希值（16 字节）
- 服务器重启后句柄仍然有效（只要缓存文件存在）
- 缓存目录：`--cache-dir/serve-nfs-handle-cache-disk/<remote>/` 或 `--nfs-cache-dir`

#### 策略三：symlink（Linux 专用）

[symlink_cache_linux.go](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/symlink_cache_linux.go)

- 在磁盘缓存基础上，使用 **符号链接** 替代普通文件存储路径映射
  - 缓存文件本身是符号链接，指向目标路径字符串
- 句柄使用 `name_to_handle_at()` 获取底层文件系统的句柄
- 通过 `open_by_handle_at()` 反查，无需目录遍历，性能最优
- 需要 root 权限或 `CAP_DAC_READ_SEARCH`
- 句柄格式：`[4字节长度前缀][原始句柄字节]` + 可选 `metadataSuffix`

### 3.3 pathRewriter：子路径挂载的句柄统一

[cache.go#L85-L135](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/cache.go#L85-L135)

```go
type pathRewriter struct {
    inner  Cache
    rootFS *FS
}
```

`pathRewriter` 是所有缓存策略的外层包装，确保：

- 无论客户端通过哪个子路径 mount 进入，同一文件始终获得相同的句柄
- `ToHandle` 时将 `(subFS, relativePath)` 重写为 `(rootFS, absolutePath)`
- `FromHandle` 返回的总是 `(rootFS, absolutePath)`，go-nfs 使用 rootFS 进行后续操作

这保证了 NFS 语义中"子路径挂载等价于 `cd` 进子目录"。

### 3.4 句柄失效

[handler.go#L122-L125](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/handler.go#L122-L125)

```go
func (h *Handler) InvalidateHandle(f billy.Filesystem, b []byte) error
```

在 rename 和 delete 操作后，`go-nfs` 调用 `InvalidateHandle` 使旧句柄失效。对于 disk/symlink 缓存，这会删除对应的磁盘缓存文件；对于 memory 缓存，从内存映射中移除。

### 3.5 元数据文件句柄

当启用了 `--vfs-metadata-extension` 时，元数据文件共享原始文件的句柄，后缀追加 `0x00 0x00 0x00 0x01`（4 字节，大端序 1）。这样 NFS 客户端可以直接从父文件句柄派生出元数据文件句柄。

---

## 4. Layer 3：VFS 访问边界（billy → VFS 适配层）

### 4.1 FS 结构体

[filesystem.go#L46-L48](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/filesystem.go#L46-L48)

```go
type FS struct {
    vfs  *vfs.VFS
    root string // 绝对路径；空表示 VFS 根
}
```

`FS` 实现了 `billy.Filesystem` 和 `billy.Change` 接口，是 NFS 层和 VFS 层之间的 **唯一桥梁**。

### 4.2 路径重写

[filesystem.go#L53-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/filesystem.go#L53-L58)

所有 billy 方法在调用 VFS 之前，都通过 `fullPath()` 将相对路径转换为 VFS 绝对路径：

```go
func (f *FS) fullPath(name string) string {
    if f.root == "" {
        return name
    }
    return path.Join(f.root, name)
}
```

这确保了子路径挂载时，billy 层看到的相对路径能正确映射到 VFS 中的绝对路径。

### 4.3 方法映射表

| billy.Filesystem 方法 | VFS 方法 | 路径转换 | 备注 |
|----------------------|----------|---------|------|
| `ReadDir(p)` | `vfs.ReadDir(fullP)` | ✅ | 返回的每个 FileInfo 调用 `setSys()` |
| `Create(name)` | `vfs.Create(fullName)` | ✅ | |
| `Open(name)` | `vfs.Open(fullName)` | ✅ | |
| `OpenFile(name, flag, perm)` | `vfs.OpenFile(fullName, flag, perm)` | ✅ | |
| `Stat(name)` | `vfs.Stat(fullName)` | ✅ | + `setSys()` |
| `Lstat(name)` | `vfs.Stat(fullName)` | ✅ | + `setSys()`；不跟踪符号链接 |
| `Rename(old, new)` | `vfs.Rename(fullOld, fullNew)` | ✅✅ | 两个路径都转换 |
| `Remove(name)` | `vfs.Remove(fullName)` | ✅ | |
| `MkdirAll(name, perm)` | 手动逐级 `vfs.Stat` + `vfs.Mkdir` | ✅ | VFS.MkDirAll 不接受 perm |
| `Symlink(target, link)` | `vfs.Symlink(target, fullLink)` | 仅 link | target 不转换（符号链接目标可能是相对路径） |
| `Readlink(link)` | `vfs.Readlink(fullLink)` | ✅ | |
| `Chmod(name, mode)` | `vfs.Open` → `file.Chmod` | ✅ | 若返回 ENOSYS 则静默忽略 |
| `Chown(name, uid, gid)` | `vfs.Open` → `file.Chown` | ✅ | |
| `Lchown(name, uid, gid)` | → `Chown` | ✅ | |
| `Chtimes(name, atime, mtime)` | `vfs.Chtimes(fullName, ...)` | ✅ | |
| `TempFile(dir, prefix)` | 返回 `os.ErrInvalid` | — | 不支持 |
| `Chroot(path)` | 返回 `os.ErrInvalid` | — | 不支持 |

### 4.4 setSys：跨层属性传递

[filesystem.go#L28-L43](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/filesystem.go#L28-L43)

```go
func setSys(fi os.FileInfo) {
    node, ok := fi.(vfs.Node)
    vfs := node.VFS()
    stat := file.FileInfo{
        Nlink:  1,
        UID:    vfs.Opt.UID,
        GID:    vfs.Opt.GID,
        Fileid: node.Inode(),
    }
    node.SetSys(&stat)
}
```

billy 接口不暴露 uid/gid/inode，但 `go-nfs` 会查询 `os.FileInfo.Sys()` 获取 `syscall.Stat_t` 或 `file.FileInfo`。`setSys()` 在每次返回 `FileInfo` 给 billy 层之前，将 VFS 配置的 uid/gid/inode 注入到节点中。

### 4.5 Capabilities 与缓存模式

[filesystem.go#L251-L258](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/filesystem.go#L251-L258)

```go
func (f *FS) Capabilities() billy.Capability {
    if f.vfs.Opt.CacheMode == vfscommon.CacheModeOff {
        return billy.ReadCapability | billy.SeekCapability
    }
    return billy.WriteCapability | billy.ReadCapability |
        billy.ReadAndWriteCapability | billy.SeekCapability | billy.TruncateCapability
}
```

当 VFS 缓存关闭时，billy 层只暴露只读能力，NFS 客户端无法写入。这与 [server.go#L28-L30](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/server.go#L28-L30) 的警告一致。

---

## 5. 三层边界总结

```
┌────────────────────────────────────────────────────────────────────┐
│ NFS 请求映射层                                                      │
│                                                                    │
│ 职责：协议解码、RPC 分发、句柄生命周期                                 │
│ 边界：对 VFS 一无所知，只操作 billy.Filesystem + []byte 句柄          │
│ 关键类型：Handler, Cache, pathRewriter                              │
│ 关键接口：nfs.Handler (Mount/ToHandle/FromHandle/InvalidateHandle) │
│                                                                    │
│                    ↓ billy.Filesystem 接口                         │
├────────────────────────────────────────────────────────────────────┤
│ billy 适配层                                                        │
│                                                                    │
│ 职责：路径重写、属性注入、接口适配                                     │
│ 边界：不管理句柄、不管理缓存、不直接处理 NFS 协议                       │
│ 关键类型：FS (billy.Filesystem + billy.Change)                     │
│ 关键方法：fullPath(), setSys(), subFS()                            │
│                                                                    │
│                    ↓ vfs.VFS 方法调用                               │
├────────────────────────────────────────────────────────────────────┤
│ VFS 层                                                             │
│                                                                    │
│ 职责：虚拟文件系统核心、读写缓存、目录树管理、Handle 生命周期           │
│ 边界：不了解 NFS 协议、不了解 billy、不了解句柄缓存                    │
│ 关键类型：vfs.VFS, vfs.Node, vfs.Dir, vfs.File, vfs.Handle        │
│                                                                    │
│                    ↓ fs.Fs 后端调用                                 │
└────────────────────────────────────────────────────────────────────┘
```

### 边界规则

| 维度 | NFS 层知道 | NFS 层不知道 |
|------|-----------|-------------|
| 路径 | 通过 `[]string` 分段路径 | VFS 内部的目录树结构 |
| 句柄 | 句柄的字节表示与缓存策略 | VFS Handle（文件打开句柄） |
| 文件内容 | 通过 billy 的 `Open/Read/Write` | VFS 的 `ReadAt/WriteAt` 直接调用 |
| 属性 | uid/gid/inode 通过 `setSys()` 注入 | VFS Node 的内部属性管理 |
| 子路径 | `Mount()` 返回 subFS | VFS 不感知子路径挂载 |
| 缓存 | 句柄缓存（memory/disk/symlink） | VFS 数据缓存（vfs-cache-mode） |
| 失效 | `InvalidateHandle` 删除句柄 | VFS 内部的缓存一致性 |

### 句柄双重含义澄清

- **NFS 文件句柄**（`[]byte`）：NFS 协议层面的 opaque 标识，由 `Cache` 管理，映射到 `(billyFS, splitPath)`
- **VFS Handle**（`vfs.Handle`）：VFS 层面的文件打开句柄，由 `vfs.Node.Open()` 返回，管理读写偏移和缓冲

两者生命周期独立：NFS 句柄在 Cache 中存在，直到被回收或失效；VFS Handle 在文件关闭后释放。NFS 层每次请求通过 `FromHandle` 获取路径后，经 billy 适配层调用 VFS，VFS 可能返回新的 Handle 或复用已有的。

---

## 6. mount2 (FUSE) 与 NFS 的对比

| 维度 | mount2 (FUSE) | serve nfs |
|------|--------------|-----------|
| 协议 | FUSE (内核 → 用户态) | NFSv3 (TCP) |
| 依赖库 | hanwen/go-fuse/v2 | willscott/go-nfs |
| 适配接口 | fusefs.InodeEmbedder | billy.Filesystem |
| 文件节点 | `Node` (嵌入 `fusefs.Inode`) | `[]byte` 句柄 |
| 文件句柄 | `FileHandle` (包装 `vfs.Handle`) | 无独立句柄对象，每次通过路径 open |
| 路径查找 | `Node.Lookup` → `vfs.Dir.Stat` | `FromHandle` → `billyFS.Stat` |
| 目录读取 | `dirStream` + `vfs.Handle.Readdir` | `billyFS.ReadDir` |
| 属性填充 | `setAttr` / `setAttrOut` | `setSys` + `file.FileInfo` |
| 子路径 | 不支持 | `Mount()` 支持 |
| 平台 | Linux / macOS (amd64) | 所有 Unix |
