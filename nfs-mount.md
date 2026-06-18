# rclone NFS 实现代码梳理：请求映射、句柄管理与 VFS 访问边界

## 1. 整体架构与两种使用模式

rclone 的 NFS 功能有两种使用模式，共享同一套核心代码：

```
模式 A: rclone serve nfs remote: --addr 0.0.0.0:20869
         → 启动 NFS 服务器，客户端手动 mount

模式 B: rclone nfsmount remote: /mnt/remote
         → 自动启动 NFS 服务器 + 执行本地 mount 命令（一键挂载）
```

两者关系如下：

```
┌──────────────────────────────────────────────────────────────────────┐
│ rclone serve nfs (nfs.go / server.go / handler.go / filesystem.go) │
│   独立运行的 NFS 服务器进程                                          │
│   客户端通过网络连接，手动执行 mount -t nfs ...                       │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│ rclone nfsmount (nfsmount/nfsmount.go)                              │
│   1. 调用 nfs.NewServer() 在进程内启动 NFS 服务器                    │
│   2. 查询服务器随机分配的端口号                                       │
│   3. exec.Command("mount", "-t nfs", "-o port=XX,mountport=XX")    │
│   4. unmount 时调用 s.Shutdown() + VFS.Shutdown()                  │
│   5. 通过 nfs.OnUnmountFunc 检测外部卸载事件                         │
│                                                                      │
│   本质：serve nfs 的"一键包装"，不是独立的挂载机制                     │
│   NFS 服务器运行在同一进程内，共享同一个 VFS 实例                       │
└──────────────────────────────────────────────────────────────────────┘
```

[nfsmount.go#L40-L118](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/nfsmount/nfsmount.go#L40-L118) 中 `mount()` 函数的关键步骤：

1. [第41行](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/nfsmount/nfsmount.go#L41)：`nfs.NewServer(context.Background(), VFS, &nfs.Opt)` — 创建 NFS 服务器
2. [第46-48行](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/nfsmount/nfsmount.go#L46-L48)：`go func() { errChan <- s.Serve() }()` — 后台启动服务
3. [第51行](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/nfsmount/nfsmount.go#L51)：`net.SplitHostPort(s.Addr().String())` — 获取随机端口
4. [第77行](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/nfsmount/nfsmount.go#L77)：`exec.Command(cmd[0], cmd[1:]...).CombinedOutput()` — 执行操作系统 mount 命令
5. [第111-115行](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/nfsmount/nfsmount.go#L111-L115)：`nfs.OnUnmountFunc` — 检测到 `mount.Umnt` RPC 后触发 VFS.Shutdown()

---

## 2. 直接访问 VFS vs 通过 billy 适配访问

这是理解边界的关键。Handler 持有 `*vfs.VFS` 和 `*FS`（billy 适配器）两个引用，部分逻辑**绕过 billy 直接访问 VFS**。

### 2.1 直接访问 VFS 的代码路径（5处）

| 位置 | 代码 | 原因 |
|------|------|------|
| [handler.go#L72](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/handler.go#L72) | `h.vfs.Stat(cleaned)` | Mount() 中验证子路径是否存在且为目录，此时 billy FS 尚未确定 |
| [handler.go#L95](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/handler.go#L95) | `h.vfs.Statfs()` | FSStat() 需要全局统计信息，不属于任何 billy FS 实例 |
| [cache.go#L156](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/cache.go#L156) | `h.vfs.Fs()` | 获取远程配置字符串用于构造缓存目录名 |
| [cache.go#L173](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/cache.go#L173) | `h.vfs.Opt.MetadataExtension` | 读取元数据扩展名配置 |
| [filesystem.go#L148-L150](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/filesystem.go#L148-L150) | `f.vfs.Stat` + `f.vfs.Mkdir` 循环 | MkdirAll 不使用 `vfs.MkdirAll`，因为后者不接受 perm 参数 |

**分析**：

- **handler.go:72 的 `h.vfs.Stat(cleaned)`** 是最重要的"边界穿越"。在 `Mount()` 被调用时，rclone 还没有确定返回哪个 billy FS（根 FS 还是 subFS），因此无法通过 billy 接口验证路径。这是架构上的必然——Mount 过程的职责就是**决定**返回哪个 billy FS，在此之前无法使用 billy 接口。

- **handler.go:95 的 `h.vfs.Statfs()`** 反映的是 billy 接口的局限：`billy.Filesystem` 没有提供文件系统级别的空间统计方法，而 NFS 的 `FSSTAT` 过程需要返回 total/free/available 空间信息，只能直接查询 VFS。

- **cache.go:156,173** 是构造时读取配置，不涉及文件操作，合理地绕过 billy。

- **filesystem.go:148-150 的 MkdirAll** 是对 VFS 接口不完整的妥协：`vfs.MkdirAll` 不接受权限参数，而 NFS MKDIR 过程携带权限，因此 billy 适配层手动逐级调用 `vfs.Stat` + `vfs.Mkdir`。

### 2.2 通过 billy 适配访问 VFS 的代码路径（15个方法）

所有文件系统操作都通过 [filesystem.go](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/filesystem.go) 中的 `FS` 结构体进行：

```
go-nfs 调用 billy.Filesystem 方法
    → FS.fullPath() 路径重写
    → f.vfs.Xxx(fullPath) 调用 VFS
    → setSys() 属性注入（仅返回 FileInfo 的方法）
    → 返回给 go-nfs
```

详细映射见第5节。

---

## 3. 请求映射的边界

### 3.1 go-nfs 库的角色

`go-nfs`（[willscott/go-nfs](https://github.com/willscott/go-nfs)）是一个 NFSv3 协议实现，它：

- **拥有**：XDR 编解码、RPC 分帧、MOUNT 协议处理、NFS 过程分发、错误码映射
- **不拥有**：文件系统逻辑、句柄存储、属性数据

它通过 `nfs.Handler` 接口向 rclone 回调：

```go
// go-nfs 定义的 Handler 接口（rclone 全部实现）
type Handler interface {
    Mount(ctx, conn, req) (MountStatus, billy.Filesystem, []AuthFlavor)
    ToHandle(billy.Filesystem, []string) []byte
    FromHandle([]byte) (billy.Filesystem, []string, error)
    HandleLimit() int
    InvalidateHandle(billy.Filesystem, []byte) error  // 可选
    Change(billy.Filesystem) billy.Change             // 可选
    FSStat(ctx, billy.Filesystem, *FSStat) error      // 可选
}
```

### 3.2 请求映射流程

每个 NFS 请求的完整处理路径：

```
NFS Client
    │
    │ TCP 连接
    ▼
go-nfs (协议层)
    │ XDR 解码 → 识别 NFS 过程类型
    │
    ├─ MOUNT 过程:
    │   └─ 调用 Handler.Mount()
    │       ├─ h.vfs.Stat(cleaned)     ← 直接访问 VFS（边界穿越）
    │       └─ 返回 billyFS 或 billyFS.subFS()
    │
    ├─ 涉及文件句柄的过程 (GETATTR/LOOKUP/READ/WRITE/...):
    │   ├─ 调用 Handler.FromHandle(fh)    ← 句柄 → (billyFS, splitPath)
    │   ├─ 调用 billyFS.Xxx(splitPath)    ← 通过 billy 适配层访问 VFS
    │   └─ 调用 Handler.ToHandle()         ← 路径 → 句柄（对新节点）
    │
    └─ FSSTAT 过程:
        └─ 调用 Handler.FSStat()
            └─ h.vfs.Statfs()              ← 直接访问 VFS（边界穿越）
```

### 3.3 NFS 过程到 billy 方法的完整映射

| NFSv3 过程 | go-nfs 内部行为 | billy.Filesystem 方法 | VFS 最终方法 |
|-----------|----------------|---------------------|-------------|
| GETATTR | `FromHandle` → 查属性 | `Stat()` | `vfs.Stat()` |
| LOOKUP | `FromHandle` → 查子项 | `Stat()` | `vfs.Stat()` |
| READDIR | `FromHandle` → 列目录 | `ReadDir()` | `vfs.ReadDir()` |
| READDIRPLUS | 同 READDIR | `ReadDir()` + `Stat()` | `vfs.ReadDir()` + `vfs.Stat()` |
| READ | `FromHandle` → 打开读 | `Open()` → file.Read() | `vfs.Open()` → `vfs.Handle.ReadAt()` |
| WRITE | `FromHandle` → 打开写 | `OpenFile()` → file.Write() | `vfs.OpenFile()` → `vfs.Handle.WriteAt()` |
| CREATE | `FromHandle` → 创建 | `Create()` | `vfs.Create()` |
| MKDIR | `FromHandle` → 建目录 | `MkdirAll()` | 逐级 `vfs.Stat()` + `vfs.Mkdir()` |
| REMOVE | `FromHandle` → 删除文件 | `Remove()` | `vfs.Remove()` |
| RMDIR | `FromHandle` → 删除目录 | `Remove()` | `vfs.Remove()` |
| RENAME | `FromHandle` → 重命名 | `Rename()` | `vfs.Rename()` |
| READLINK | `FromHandle` → 读链接 | `Readlink()` | `vfs.Readlink()` |
| SYMLINK | `FromHandle` → 建链接 | `Symlink()` | `vfs.Symlink()` |
| SETATTR | `FromHandle` → 改属性 | `Chmod()` / `Chtimes()` / Truncate | `vfs.Open()` → `file.Chmod()` / `vfs.Chtimes()` / `vfs.Handle.Truncate()` |
| FSSTAT | 调用 Handler.FSStat | — | `vfs.Statfs()` (直接) |

---

## 4. 句柄管理的边界

### 4.1 两种"句柄"的严格区分

| | NFS 文件句柄 | VFS Handle |
|--|-------------|------------|
| **类型** | `[]byte` | `vfs.Handle`（接口） |
| **产生** | `Cache.ToHandle(billyFS, splitPath)` | `vfs.Open()` / `vfs.Create()` |
| **含义** | 标识一个路径在某个 billy FS 中的位置 | 标识一个已打开文件的读写状态 |
| **生命周期** | Cache 管理，直到被回收或 InvalidateHandle | 文件 Close() 后释放 |
| **作用域** | 跨请求持久（客户端持有） | 单次 Open-Close 间 |
| **所在层** | Handler / Cache | VFS |

### 4.2 句柄缓存的分层结构

```
Handler.ToHandle(f, path) / Handler.FromHandle(fh)
    │
    ▼
pathRewriter  ← 外层：统一子路径到根路径
    │  translate(f, splitPath) → (rootFS, absoluteSplitPath)
    │
    ▼
inner Cache  ← 内层：实际存储策略（三选一）
    │
    ├── CachingHandler (memory)   ← go-nfs/helpers 内置
    │   map[autoIncrementID] → (billyFS, splitPath)
    │   句柄 = 自增数字
    │
    ├── diskHandler (disk)        ← cache.go 实现
    │   磁盘文件 MD5(path) → path字符串
    │   句柄 = MD5哈希 (16字节)
    │
    └── diskHandler (symlink)     ← symlink_cache_linux.go 实现
        磁盘符号链接 MD5(path) → path
        句柄 = name_to_handle_at() 返回的内核文件句柄
```

### 4.3 pathRewriter 详解

[cache.go#L85-L135](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/cache.go#L85-L135)

`pathRewriter` 是句柄缓存的最外层包装，解决一个核心问题：**同一文件通过不同子路径挂载必须获得相同的 NFS 句柄**。

```
客户端 A: mount localhost:/sub        → 获得 subFS(root="/sub")
客户端 B: mount localhost:/            → 获得 rootFS(root="")

两者访问 /sub/hello.txt:
  客户端 A: ToHandle(subFS, ["hello.txt"])
    → pathRewriter.translate(subFS, ["hello.txt"])
    → (rootFS, ["sub", "hello.txt"])
    → inner.ToHandle(rootFS, ["sub", "hello.txt"])
    → 句柄 H1

  客户端 B: ToHandle(rootFS, ["sub", "hello.txt"])
    → pathRewriter.translate(rootFS, ["sub", "hello.txt"])
    → (rootFS, ["sub", "hello.txt"])   ← 未改变，rootFS.root==""
    → inner.ToHandle(rootFS, ["sub", "hello.txt"])
    → 句柄 H2

  H1 == H2 ✓  同一文件得到相同句柄
```

这在 [handler_test.go#L135-L143](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/handler_test.go#L135-L143) 和 [cache_test.go#L176-L220](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/cache_test.go#L176-L220) 中有明确的测试覆盖。

### 4.4 句柄失效时机

[handler.go#L122-L125](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/handler.go#L122-L125)

`go-nfs` 在以下 NFS 操作完成后自动调用 `InvalidateHandle`：

- **RENAME** — 旧路径的句柄必须失效
- **REMOVE / RMDIR** — 被删除文件的句柄必须失效

对于元数据文件句柄（带有 `metadataSuffix`），`InvalidateHandle` 是空操作——元数据句柄是合成的，不需要失效。

### 4.5 FromHandle 返回后的 billy FS 身份

`FromHandle` 总是返回 **rootFS**（非 subFS），因为 `pathRewriter.translate` 在 `ToHandle` 时已将所有路径归一化到 rootFS 坐标系。这意味着 go-nfs 后续的 billy 调用总是通过 rootFS 进行，rootFS 的 `fullPath()` 在 `root == ""` 时直接返回路径本身，无需额外重写。

---

## 5. 子路径挂载的边界

### 5.1 子路径挂载的完整流程

```
1. NFS Client 发送 MOUNT 请求: Dirpath = "/photos/2024"
                    │
2. Handler.Mount() 被调用
   │
   ├─ path.Clean("/" + "/photos/2024") = "/photos/2024"
   │
   ├─ h.vfs.Stat("/photos/2024")  ← 直接访问 VFS，验证路径
   │   ├─ 不存在 → return MountStatusErrNoEnt
   │   └─ 非目录 → return MountStatusErrNotDir
   │
   ├─ 验证通过
   │
   └─ return MountStatusOk, h.billyFS.subFS("/photos/2024"), auths
                    │
3. go-nfs 获得子路径 billyFS 后:
   │
   ├─ 调用 ToHandle(subFS, []) 获取根句柄
   │   → pathRewriter 归一化: (subFS, []) → (rootFS, ["photos", "2024"])
   │   → 返回根句柄给客户端
   │
   └─ 后续所有操作:
       FromHandle(fh) → (rootFS, absoluteSplitPath)
       rootFS.Stat(absoluteSplitPath) → 通过 billy 访问 VFS
```

### 5.2 子路径的安全保证

[handler.go#L68](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/handler.go#L68)

```go
cleaned := path.Clean("/" + string(req.Dirpath))
```

`path.Clean` 前置 `/` 确保结果始终是绝对路径，且 `..` 不会穿越到 VFS 根之外。例如 `"/../../etc"` 被 Clean 为 `"/etc"`，而 VFS 中不存在 `/etc`，Stat 返回错误。

测试覆盖见 [handler_test.go#L84-L108](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/handler_test.go#L84-L108)，其中 `"traversal"` 用例验证 `"/../../etc"` 无法穿越。

### 5.3 子路径 FS 的路径隔离机制

[filesystem.go#L53-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/filesystem.go#L53-L58)

```go
func (f *FS) fullPath(name string) string {
    if f.root == "" {
        return name          // rootFS: 路径不重写
    }
    return path.Join(f.root, name)  // subFS: 追加 root 前缀
}
```

当 subFS 的 `root = "/photos/2024"` 时：
- `subFS.Stat("file.txt")` → `vfs.Stat("/photos/2024/file.txt")`
- `subFS.ReadDir("")` → `vfs.ReadDir("/photos/2024")`

但注意：**子路径挂载不提供隔离**。如 [nfs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/nfs.go) 文档所述，子路径挂载等价于"mount `/` 然后 cd 进子目录"，共享同一个 VFS 实例和文件句柄。客户端如果知道其他路径的句柄，仍然可以访问。

---

## 6. 属性注入的边界

### 6.1 问题根源

billy 接口的 `os.FileInfo` 只暴露标准字段（Name, Size, Mode, ModTime），不暴露 uid/gid/inode。但 NFS 协议的 GETATTR/LOOKUP 响应必须包含这些字段。

### 6.2 setSys 注入机制

[filesystem.go#L28-L43](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/filesystem.go#L28-L43)

```go
func setSys(fi os.FileInfo) {
    node, ok := fi.(vfs.Node)    // 类型断言：FileInfo 实际就是 vfs.Node
    vfs := node.VFS()
    stat := file.FileInfo{
        Nlink:  1,
        UID:    vfs.Opt.UID,       // 从 VFS 配置读取
        GID:    vfs.Opt.GID,       // 从 VFS 配置读取
        Fileid: node.Inode(),      // VFS 分配的 inode 编号
    }
    node.SetSys(&stat)             // 注入到 vfs.Node.Sys()
}
```

`go-nfs` 在构造 NFS 属性响应时，会调用 `os.FileInfo.Sys()`，如果能获取到 `file.FileInfo` 或 `syscall.Stat_t`，就从中提取 uid/gid/fileid。

### 6.3 setSys 调用时机

只有返回 `os.FileInfo` 给 go-nfs 的方法才需要调用 `setSys`：

| billy 方法 | 是否调用 setSys | 原因 |
|-----------|---------------|------|
| `Stat()` | ✅ | 返回单个 FileInfo |
| `Lstat()` | ✅ | 返回单个 FileInfo |
| `ReadDir()` | ✅ (循环) | 返回 FileInfo 列表，每个都需要注入 |
| `Create()` | ❌ | 返回 billy.File，不涉及 FileInfo |
| `Open()` | ❌ | 返回 billy.File |
| `OpenFile()` | ❌ | 返回 billy.File |
| `Rename()` | ❌ | 无返回值 |
| `Remove()` | ❌ | 无返回值 |
| `MkdirAll()` | ❌ | 无 FileInfo 返回 |
| `Symlink()` | ❌ | 无返回值 |
| `Readlink()` | ❌ | 返回字符串 |
| `Chmod()` | ❌ | 无返回值 |
| `Chown()` | ❌ | 无返回值 |
| `Chtimes()` | ❌ | 无返回值 |

### 6.4 Inode 的重要性

[filesystem.go#L40](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/serve/nfs/filesystem.go#L40)

```go
Fileid: node.Inode(), // without this mounting doesn't work on Linux
```

Linux NFS 客户端使用 fileid 作为 inode 号来标识文件的唯一性。如果 fileid 为 0 或不可靠，Linux 客户端会出现 `ESTALE` 错误。这个值来自 VFS 层的 `vfs.Node.Inode()` 方法。

### 6.5 属性注入的层间协议

```
VFS 层                    billy 适配层               go-nfs 层
  │                           │                         │
  │ vfs.Stat() 返回           │                         │
  │ vfs.Node (也是 FileInfo)  │                         │
  │◄─────────────────────────│                         │
  │                           │ setSys(fi)              │
  │ node.SetSys(&file.FileInfo{UID,GID,Fileid})        │
  │◄─────────────────────────│                         │
  │                           │ 返回 fi 给 go-nfs       │
  │                           │────────────────────────►│
  │                           │                         │ fi.Sys()
  │                           │                         │──► *file.FileInfo
  │                           │                         │ 提取 UID, GID, Fileid
  │                           │                         │ 填入 NFS fattr3
```

**边界规则**：VFS 层不感知 NFS 属性需求；billy 适配层负责在 VFS 返回值和 NFS 期望值之间桥接。这种设计使得 VFS 可以同时服务于 FUSE 和 NFS 两种前端——FUSE 使用 `setAttr`/`setAttrOut` 填充属性（见 [fs.go#L69-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/54-rclone/cmd/mount2/fs.go#L69-L105)），NFS 使用 `setSys` 填充属性，两者互不干扰。

---

## 7. VFS 访问的完整边界图

```
                    ┌─────────────────────────────────┐
                    │        go-nfs 协议库              │
                    │  NFSv3 RPC 解码/编码/分发         │
                    └────────┬────────────┬───────────┘
                             │            │
              billy.Filesystem 接口      nfs.Handler 接口
                             │            │
                    ┌────────▼────────────▼───────────┐
                    │          Handler                 │
                    │                                  │
                    │  ┌── nfs.Handler 接口实现 ──┐    │
                    │  │                          │    │
                    │  │ Mount() ─────────────────┤────┤── h.vfs.Stat()      ← 直接访问 VFS ①
                    │  │ FSStat() ────────────────┤────┤── h.vfs.Statfs()    ← 直接访问 VFS ②
                    │  │ ToHandle() → Cache       │    │
                    │  │ FromHandle() → Cache     │    │
                    │  │ InvalidateHandle()→Cache │    │
                    │  │ Change() → billy.Change  │    │
                    │  └──────────────────────────┘    │
                    │                                  │
                    │  ┌── Cache 层 ──────────────┐    │
                    │  │ pathRewriter (外层)        │    │
                    │  │  └── inner Cache (内层)   │    │   h.vfs.Fs()        ← 直接访问 VFS ③
                    │  │      ├── memory            │    │   h.vfs.Opt.*       ← 直接访问 VFS ④
                    │  │      ├── disk              │    │
                    │  │      └── symlink           │    │
                    │  └───────────────────────────┘    │
                    │                                  │
                    │  ┌── billy 适配层 ────────────┐   │
                    │  │ FS (billy.Filesystem)       │   │
                    │  │                              │   │
                    │  │  ReadDir  → f.vfs.ReadDir   │   │ ← 全部通过 billy 适配
                    │  │  Create   → f.vfs.Create    │   │
                    │  │  Open     → f.vfs.Open      │   │
                    │  │  OpenFile → f.vfs.OpenFile  │   │
                    │  │  Stat     → f.vfs.Stat      │   │  + setSys() 属性注入
                    │  │  Lstat    → f.vfs.Stat      │   │  + setSys() 属性注入
                    │  │  Rename   → f.vfs.Rename    │   │
                    │  │  Remove   → f.vfs.Remove    │   │
                    │  │  MkdirAll → 手动 Stat+Mkdir │   │ ← 绕过 vfs.MkdirAll ⑤
                    │  │  Symlink  → f.vfs.Symlink   │   │
                    │  │  Readlink → f.vfs.Readlink  │   │
                    │  │  Chmod    → f.vfs.Open+Chmod│   │
                    │  │  Chown    → f.vfs.Open+Chown│   │
                    │  │  Chtimes  → f.vfs.Chtimes   │   │
                    │  │  Capabilities → f.vfs.Opt   │   │ ← 读取配置，非文件操作
                    │  └──────────────────────────────┘   │
                    └──────────────────────────────────────┘
                                       │
                                       ▼
                    ┌──────────────────────────────────────┐
                    │            vfs.VFS                    │
                    │  虚拟文件系统核心                       │
                    │  读写缓存 / 目录树 / Handle 生命周期    │
                    │  不感知 NFS / billy / 句柄缓存          │
                    └──────────────────────────────────────┘
                                       │
                                       ▼
                    ┌──────────────────────────────────────┐
                    │            fs.Fs 后端                  │
                    │  (S3 / Google Drive / local / ...)    │
                    └──────────────────────────────────────┘
```

---

## 8. 边界穿越的合理性评估

| # | 穿越点 | 访问方式 | 是否合理 | 理由 |
|---|--------|---------|---------|------|
| ① | `Handler.Mount` | `h.vfs.Stat(cleaned)` | ✅ 合理 | Mount 的职责是决定返回哪个 billy FS，在决定之前无法使用 billy 接口 |
| ② | `Handler.FSStat` | `h.vfs.Statfs()` | ✅ 合理 | billy 接口没有空间统计方法，NFS FSSTAT 过程必须有 |
| ③ | `newDiskHandler` | `h.vfs.Fs()` | ✅ 合理 | 仅用于构造缓存目录名，非文件操作 |
| ④ | `newDiskHandler` | `h.vfs.Opt.MetadataExtension` | ✅ 合理 | 仅读取配置，非文件操作 |
| ⑤ | `FS.MkdirAll` | 手动 `vfs.Stat` + `vfs.Mkdir` | ⚠️ 妥协 | VFS.MkdirAll 不接受 perm 参数；理想情况下应扩展 VFS 接口 |

---

## 9. mount2 (FUSE) 与 NFS 的架构对比

| 维度 | mount2 (FUSE) | serve nfs |
|------|--------------|-----------|
| 协议 | FUSE (/dev/fuse) | NFSv3 (TCP) |
| 依赖库 | hanwen/go-fuse/v2 | willscott/go-nfs |
| 适配接口 | `fusefs.InodeEmbedder` | `billy.Filesystem` |
| 文件标识 | `Node` (嵌入 `fusefs.Inode`，有内核 inode) | `[]byte` 句柄 (Cache 管理) |
| 文件句柄 | `FileHandle` (显式包装 `vfs.Handle`) | 无独立句柄对象，每次 `FromHandle` → `billy.Open` |
| 路径查找 | `Node.Lookup` → `vfs.Dir.Stat` | `FromHandle` → `billyFS.Stat` |
| 目录读取 | `dirStream` + `vfs.Handle.Readdir` | `billyFS.ReadDir` |
| 属性填充 | `setAttr` / `setAttrOut` (直接写 fuse.Attr) | `setSys` (注入到 vfs.Node.Sys()) |
| 子路径 | 不支持 | `Mount()` 支持 subFS |
| VFS 直接访问 | `FS.Root()` → `vfs.Root()` | `Mount()` → `vfs.Stat()`, `FSStat()` → `vfs.Statfs()` |
| 错误映射 | `translateError()` (vfs 错误 → syscall.Errno) | billy/go-nfs 内部处理 |
| 平台 | Linux / macOS (amd64) | 所有 Unix |
