# rclone NFS 实现代码梳理：请求映射、句柄管理与 VFS 访问边界

> 代码引用格式：`相对路径#L<起始行>-L<结束行>`，如 `cmd/serve/nfs/handler.go#L66-L83`

---

## 1. 整体架构与两种使用模式

rclone 的 NFS 功能有两种使用模式，共享同一套核心代码：

```
模式 A: rclone serve nfs remote: --addr 0.0.0.0:20869
         → 启动 NFS 服务器，客户端手动 mount

模式 B: rclone nfsmount remote: /mnt/remote
         → 自动启动 NFS 服务器 + 执行本地 mount 命令（一键挂载）
```

两者关系：

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

`cmd/nfsmount/nfsmount.go#L40-L118` 中 `mount()` 函数的关键步骤：

1. `cmd/nfsmount/nfsmount.go#L41`：`nfs.NewServer(context.Background(), VFS, &nfs.Opt)` — 创建 NFS 服务器
2. `cmd/nfsmount/nfsmount.go#L46-L48`：后台启动 `s.Serve()`
3. `cmd/nfsmount/nfsmount.go#L51`：`net.SplitHostPort(s.Addr().String())` — 获取随机端口
4. `cmd/nfsmount/nfsmount.go#L77`：`exec.Command(mount ...)` — 执行操作系统 mount 命令
5. `cmd/nfsmount/nfsmount.go#L111-L115`：`nfs.OnUnmountFunc` — 检测 `mount.Umnt` RPC 后触发 VFS.Shutdown()

核心源码文件：

| 文件 | 职责 |
|------|------|
| `cmd/serve/nfs/server.go` | NFS Server 生命周期：创建、监听、Serve |
| `cmd/serve/nfs/handler.go` | `nfs.Handler` 实现：Mount、ToHandle、FromHandle、HandleLimit |
| `cmd/serve/nfs/filesystem.go` | `billy.Filesystem` 适配器：将 billy 调用转译为 vfs 调用 |
| `cmd/serve/nfs/cache.go` | 句柄缓存：memory / disk / symlink 三种策略 + pathRewriter |
| `cmd/serve/nfs/symlink_cache_linux.go` | Linux 专用 symlink 缓存实现 |
| `cmd/serve/nfs/nfs.go` | CLI 入口与选项定义 |
| `cmd/nfsmount/nfsmount.go` | `rclone nfsmount` 命令：自动启动 NFS 服务器 + 本地 mount |

---

## 2. VFS 访问的三类边界：彻底厘清

Handler（NFS 层）持有 `*vfs.VFS` 和 `*FS`（billy 适配器）两个引用。所有对 VFS 的访问可以严格分为 **三大类**：

| 类别 | 定义 | 数量 | 典型场景 |
|------|------|------|---------|
| **A 类：绕过 billy 直接访问** | NFS 层/Handler 层/Server 层直接调用 `vfs.XXX()` 方法，不经过 billy.Filesystem 接口 | 7 处 | Mount 路径验证、FSSTAT 空间统计、MkdirAll 权限透传 |
| **B 类：经 billy 适配后访问** | go-nfs 调用 `billy.Filesystem.XXX()`，FS 适配层做 `fullPath()` 重写后调用 `f.vfs.XXX()`，并在需要时调用 `setSys()` 属性注入 | 18 处 | 所有文件系统操作：Stat/ReadDir/Open/Create/Remove 等 |
| **C 类：仅读取配置** | 读取 `vfs.Opt.*` 或 `vfs.Fs()` 获取配置元数据，不发起任何文件系统操作 | 8 处 | 判断 CacheMode、构造缓存目录名、读取 UID/GID、填充属性 |

---

## 3. A 类：绕过 billy 直接访问 VFS（7处）

这类访问发生在 billy 适配层之外，由 NFS 层（Handler / Server / FS.MkdirAll 内部）直接调用 VFS 方法。

| # | 位置 | 代码 | 所属方法 | 绕过原因 |
|---|------|------|---------|---------|
| A1 | `cmd/serve/nfs/handler.go#L72` | `h.vfs.Stat(cleaned)` | `Handler.Mount` | Mount 的职责是决定返回哪个 billy FS。在决定之前 billy FS 尚未确定，无法通过 billy 接口验证路径。**架构上的必然**。 |
| A2 | `cmd/serve/nfs/handler.go#L95` | `h.vfs.Statfs()` | `Handler.FSStat` | `billy.Filesystem` 接口没有空间统计方法，但 NFS FSSTAT 过程必须返回 total/free/available。**billy 接口的局限**。 |
| A3 | `cmd/serve/nfs/filesystem.go#L148` | `f.vfs.Stat(current)` | `FS.MkdirAll` 内部 | VFS 的 `MkdirAll` 不接受权限参数（perm），而 billy 的 `MkdirAll` 带 perm。必须手动逐级 `Stat + Mkdir` 来保证权限被正确传递。**VFS 接口不完整的妥协**。 |
| A4 | `cmd/serve/nfs/filesystem.go#L150` | `f.vfs.Mkdir(current, perm)` | `FS.MkdirAll` 内部 | 同上。 |
| A5 | `cmd/serve/nfs/server.go#L28` | `vfs.Opt.CacheMode` 读取 | `NewServer` | 创建服务器前判断是否需要警告 NFS 只读。虽然在 `server.go` 中读取，但 VFS 仍未通过 billy。**启动时的前置检查**。 |
| A6 | `cmd/nfsmount/nfsmount.go#L101` | `VFS.Shutdown()` | `unmount` 闭包 | VFS 生命周期管理，unmount 时关闭 VFS。不属于文件系统操作。 |
| A7 | `cmd/nfsmount/nfsmount.go#L114` | `VFS.Shutdown()` | `OnUnmountFunc` | 同上，外部卸载事件触发。 |

**A 类调用的完整链路（以 A1 Mount 为例）：**

```
NFS Client 发送 MOUNT 请求
    │
    ▼
go-nfs 协议层 (MOUNTPROC3_MNT)
    │
    ▼
Handler.Mount()                       ← nfs.Handler 接口
    │
    ├─ path.Clean("/" + req.Dirpath)
    │
    ├─ h.vfs.Stat(cleaned)             ← ⚡ A1 直接访问 VFS，不经过 billy
    │   └─ 验证路径存在且是目录
    │
    └─ 返回 h.billyFS / h.billyFS.subFS()
```

**A 类调用的完整链路（以 A2 FSStat 为例）：**

```
NFS Client 发送 FSSTAT 请求
    │
    ▼
go-nfs 协议层 (NFSPROC3_FSSTAT)
    │
    ▼
Handler.FSStat()                      ← nfs.Handler 可选接口
    │
    └─ h.vfs.Statfs()                  ← ⚡ A2 直接访问 VFS，billy 接口无此方法
        └─ 返回 total/used/free
```

---

## 4. B 类：经 billy 适配后访问 VFS（18处）

这是 NFS 文件系统操作的主干路径。所有调用通过 `cmd/serve/nfs/filesystem.go` 中 `FS` 结构体（实现 `billy.Filesystem` 接口）进入。

**统一的处理模式：**

```
go-nfs 调用 billy.Filesystem.XXX()
    │
    ▼
FS.XXX(name, ...)                    ← billy 接口实现
    │
    ├─ fullPath(name)                 ← [如果 subFS] 追加 root 前缀
    │   └─ path.Join(f.root, name)
    │
    ├─ f.vfs.XXX(fullName, ...)       ← ⚡ B 类：通过 billy 适配后调用 VFS
    │
    ├─ setSys(returnedFileInfo)       ← [如果返回 FileInfo] 注入 uid/gid/inode
    │
    └─ 返回给 go-nfs
```

### 4.1 B 类详细分类表

| # | 位置 | billy 方法 | 最终 VFS 方法 | fullPath 重写 | setSys | 备注 |
|---|------|-----------|-------------|---------------|--------|------|
| B1 | `filesystem.go#L70` | `ReadDir(p)` | `vfs.ReadDir(fullP)` | ✅ | ✅ 循环注入 | 列目录 |
| B2 | `filesystem.go#L84` | `Create(filename)` | `vfs.Create(fullName)` | ✅ | ❌ | 创建文件，返回 billy.File |
| B3 | `filesystem.go#L91` | `Open(filename)` | `vfs.Open(fullName)` | ✅ | ❌ | 打开文件，返回 billy.File |
| B4 | `filesystem.go#L98` | `OpenFile(filename,flag,perm)` | `vfs.OpenFile(fullName,...)` | ✅ | ❌ | 以 flag 打开文件 |
| B5 | `filesystem.go#L105` | `Stat(filename)` | `vfs.Stat(fullName)` | ✅ | ✅ | 获取属性 |
| B6 | `filesystem.go#L118` | `Rename(old, new)` | `vfs.Rename(fullOld, fullNew)` | ✅✅ 两个路径 | ❌ | 重命名 |
| B7 | `filesystem.go#L125` | `Remove(filename)` | `vfs.Remove(fullName)` | ✅ | ❌ | 删除文件 |
| B8 | `filesystem.go#L163` | `Lstat(filename)` | `vfs.Stat(fullName)` | ✅ | ✅ | 不跟踪符号链接 |
| B9 | `filesystem.go#L175` | `Symlink(target, link)` | `vfs.Symlink(target, fullLink)` | ✅ 仅 link | ❌ | target 不转换（可能是相对路径） |
| B10 | `filesystem.go#L182` | `Readlink(link)` | `vfs.Readlink(fullLink)` | ✅ | ❌ | 读符号链接 |
| B11 | `filesystem.go#L189` | `Chmod(name, mode)` | `vfs.Open(fullName)` → `file.Chmod()` | ✅ | ❌ | 先 Open 再 Chmod；ENOSYS 静默忽略 |
| B12 | `filesystem.go#L216` | `Chown(name, uid, gid)` | `vfs.Open(fullName)` → `file.Chown()` | ✅ | ❌ | 先 Open 再 Chown |
| B13 | `filesystem.go#L232` | `Chtimes(name, atime, mtime)` | `vfs.Chtimes(fullName, ...)` | ✅ | ❌ | 修改时间戳 |
| B14 | `filesystem.go#L148` | `MkdirAll(name, perm)` | 循环调用 `vfs.Stat(current)` | ✅ | ❌ | **注意：MkdirAll 内部实际是 A 类语义** — 绕过了 `vfs.MkdirAll`（不接受 perm） |
| B15 | `filesystem.go#L150` | `MkdirAll(name, perm)` | 循环调用 `vfs.Mkdir(current, perm)` | ✅ | ❌ | 同上 |
| B16 | `filesystem.go#L34` | `setSys(fi)` 内部 | `node.VFS()` — 获取 VFS 引用 | N/A | ✅ 本方法就是注入 | 读取 UID/GID 配置（见 C 类） |
| B17 | `filesystem.go#L40` | `setSys(fi)` 内部 | `node.Inode()` — 获取 inode 号 | N/A | ✅ 本方法就是注入 | 注入 fileid（Linux 客户端必填） |
| B18 | `filesystem.go#L245,L247` | `Root()` | `f.vfs.Fs().Root()` | N/A | ❌ | 返回后端根路径，bililly 接口要求 |

### 4.2 B 类典型调用链（以 GETATTR/LOOKUP 为例）

```
NFS Client: NFSPROC3_GETATTR(fh)
    │
    ▼
go-nfs: FromHandle(fh) → (billyFS, splitPath)
    │
    ▼
go-nfs: billyFS.Stat(path.Join(splitPath...))
    │
    ▼
FS.Stat(filename)
    ├─ fullPath(filename)         ← subFS.root != "" 时追加前缀
    │   └─ fullName = path.Join(f.root, filename)
    │
    ├─ f.vfs.Stat(fullName)       ← ⚡ B5 通过 billy 适配访问 VFS
    │   └─ 返回 vfs.Node（实现 os.FileInfo）
    │
    └─ setSys(fi)                 ← 注入属性
        ├─ node.VFS().Opt.UID     ← C6（见下节）
        ├─ node.VFS().Opt.GID     ← C7（见下节）
        ├─ node.Inode()           ← B17
        └─ node.SetSys(&stat)
    │
    ▼
go-nfs: fi.Sys() → *file.FileInfo → 提取 uid/gid/fileid → NFS fattr3
    │
    ▼
NFS Client: GETATTR 响应
```

---

## 5. C 类：仅读取 VFS 配置（8处）

这类访问只读取 `vfs.Opt.*` 字段或 `vfs.Fs()`（后端 fs 对象），不触发任何文件系统 I/O 操作。虽然"触碰了 VFS 对象"，但本质是读取配置元数据。

| # | 位置 | 代码 | 所属方法 | 用途 |
|---|------|------|---------|------|
| C1 | `cmd/serve/nfs/server.go#L28` | `vfs.Opt.CacheMode` | `NewServer` | 判断是否 `CacheModeOff`，若是则发出 NFS 只读警告 |
| C2 | `cmd/serve/nfs/filesystem.go#L253` | `f.vfs.Opt.CacheMode` | `FS.Capabilities` | 决定 billy 暴露的能力：只读 vs 可读写 |
| C3 | `cmd/serve/nfs/cache.go#L156` | `h.vfs.Fs()` → `fs.ConfigString()` | `newDiskHandler` | 构造磁盘缓存目录名（使用 remote 配置字符串） |
| C4 | `cmd/serve/nfs/cache.go#L173` | `h.vfs.Opt.MetadataExtension` | `newDiskHandler` | 读取元数据扩展名，用于 metadataSuffix 处理 |
| C5 | `cmd/serve/nfs/filesystem.go#L245` | `f.vfs.Fs().Root()` | `FS.Root` | 返回后端 fs 的 root 作为 billy Root 路径 |
| C6 | `cmd/serve/nfs/filesystem.go#L38` | `vfs.Opt.UID` | `setSys` | 注入 UID 到 FileInfo.Sys() |
| C7 | `cmd/serve/nfs/filesystem.go#L39` | `vfs.Opt.GID` | `setSys` | 注入 GID 到 FileInfo.Sys() |
| C8 | `cmd/serve/nfs/filesystem.go#L247` | `f.vfs.Fs().Root()` | `FS.Root` | subFS 时拼接 root 前缀后的后端路径 |

---

## 6. 三类访问的可视化边界图

```
                    ┌──────────────────────────────────────┐
                    │        go-nfs 协议库                  │
                    │  NFSv3 RPC 解码/编码/分发             │
                    └──────┬──────────────┬────────────────┘
                           │              │
              billy.Filesystem 接口   nfs.Handler 接口
                           │              │
┌──────────────────────────▼──────────────▼──────────────────────────────────┐
│                          NFS 层（Handler + billy FS + Cache）               │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  nfs.Handler 接口实现 (handler.go)                                   │   │
│  │                                                                      │   │
│  │  Mount() ────────────────────── h.vfs.Stat(cleaned)    ⚡ A1 绕过    │   │
│  │  FSStat() ───────────────────── h.vfs.Statfs()          ⚡ A2 绕过    │   │
│  │  ToHandle()   → Cache (pathRewriter → memory/disk/symlink)           │   │
│  │  FromHandle() → Cache                                               │   │
│  │  InvalidateHandle() → Cache                                         │   │
│  │  Change() → billy.Change (FS)                                       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  billy 适配层 (filesystem.go: FS 实现 billy.Filesystem)              │   │
│  │                                                                      │   │
│  │  ⚙ C1~C8  仅读取 VFS 配置：                                           │   │
│  │     C1,C2: vfs.Opt.CacheMode     → 警告 / Capabilities               │   │
│  │     C6,C7: vfs.Opt.UID,GID       → setSys 属性注入                    │   │
│  │     C3:    vfs.Fs()              → 缓存目录名                          │   │
│  │     C5,C8: vfs.Fs().Root()       → billy Root                         │   │
│  │                                                                      │   │
│  │  ⚡ B 类 通过 billy 适配（+ fullPath 重写 + setSys）：                  │   │
│  │     B1: ReadDir   → f.vfs.ReadDir       + setSys(循环)               │   │
│  │     B2: Create    → f.vfs.Create                                     │   │
│  │     B3: Open      → f.vfs.Open                                       │   │
│  │     B4: OpenFile  → f.vfs.OpenFile                                   │   │
│  │     B5: Stat      → f.vfs.Stat         + setSys                      │   │
│  │     B6: Rename    → f.vfs.Rename                                     │   │
│  │     B7: Remove    → f.vfs.Remove                                     │   │
│  │     B8: Lstat     → f.vfs.Stat         + setSys                      │   │
│  │     B9: Symlink   → f.vfs.Symlink                                    │   │
│  │     B10: Readlink → f.vfs.Readlink                                   │   │
│  │     B11: Chmod    → f.vfs.Open → Chmod                               │   │
│  │     B12: Chown    → f.vfs.Open → Chown                               │   │
│  │     B13: Chtimes  → f.vfs.Chtimes                                    │   │
│  │     B18: Root     → f.vfs.Fs().Root()                                │   │
│  │                                                                      │   │
│  │  ⚡ A3/A4  MkdirAll 绕过 vfs.MkdirAll（不接受 perm）：                │   │
│  │     循环 B14: vfs.Stat(current)    A3                                │   │
│  │     循环 B15: vfs.Mkdir(current,p)  A4                                │   │
│  │                                                                      │   │
│  │  🔧 属性注入层（setSys）：                                              │   │
│  │     B16: node.VFS()       → 获取 VFS 引用 (用于 C6,C7)                │   │
│  │     B17: node.Inode()     → 填入 fileid (Linux 必填)                  │   │
│  │     C6,C7 注入 UID/GID                                               │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  句柄缓存层 (cache.go)                                                │   │
│  │     C3,C4  读取配置 (仅构造时)                                         │   │
│  │     不直接操作 VFS 对象                                                │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ⚡ A5  NewServer: vfs.Opt.CacheMode 读取（启动前置警告）                     │
│  ⚡ A6,A7  VFS.Shutdown（nfsmount 生命周期管理）                              │
└─────────────────────────────────────────────────────────────────────────────┘
                           │
                           │  ⚡ A/B/C 三类都最终调用 VFS 方法
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

## 7. 各类访问的合理性评估

### A 类（绕过 billy 直接访问 VFS）合理性

| # | 绕过点 | 是否合理 | 详细理由 |
|---|--------|---------|---------|
| A1 | `Handler.Mount` → `h.vfs.Stat` | ✅ 合理 | Mount 的职责就是**决定**返回哪个 billy FS。在决策之前 billy FS 尚未确定，无法通过 billy 接口。 |
| A2 | `Handler.FSStat` → `h.vfs.Statfs` | ✅ 合理 | billy 接口未提供 FSStat 方法，但 NFS 协议必须响应。这是接口设计的缺口。 |
| A3/A4 | `FS.MkdirAll` → 手动 Stat+Mkdir | ⚠️ 妥协 | `vfs.MkdirAll(path, perm)` 不存在，只有 `vfs.MkdirAll(path string)`。但 NFS MKDIR 携带权限信息。理想的修复是扩展 VFS 接口。 |
| A5 | `NewServer` → `vfs.Opt.CacheMode` | ✅ 合理 | 服务器启动前的前置检查，不属于文件系统操作。 |
| A6/A7 | `VFS.Shutdown()` | ✅ 合理 | VFS 生命周期管理，unmount 时关闭。 |

### B 类（经 billy 适配）合理性

B 类全部合理：18 个调用全部在 billy 接口语义的范围内，billy 适配层正确地承担了路径重写和属性注入的职责。

### C 类（仅读取配置）合理性

C 类全部合理：虽然读取的是 VFS 对象的字段，但完全没有文件系统语义，属于配置层。setSys 中的 C6/C7 是 billy 接口不暴露 uid/gid 的必要补偿。

---

## 8. 请求映射的边界

### 8.1 go-nfs 库的角色

go-nfs（第三方库）**拥有**：XDR 编解码、RPC 分帧、MOUNT 协议处理、NFS 过程分发、错误码映射；**不拥有**：文件系统逻辑、句柄存储、属性数据。

它通过 `nfs.Handler` 接口向 rclone 回调：

```go
// go-nfs 定义的 Handler 接口
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

### 8.2 NFS 过程到方法的完整映射

| NFSv3 过程 | 入口接口 | 类别 | billy 方法 | VFS 最终方法 |
|-----------|---------|------|-----------|-------------|
| GETATTR | FromHandle → billy | B | `Stat()` | `vfs.Stat()` + setSys |
| LOOKUP | FromHandle → billy | B | `Stat()` | `vfs.Stat()` + setSys |
| READDIR | FromHandle → billy | B | `ReadDir()` | `vfs.ReadDir()` + setSys |
| READDIRPLUS | FromHandle → billy | B | `ReadDir()` + `Stat()` | `vfs.ReadDir()` + `vfs.Stat()` |
| READ | FromHandle → billy | B | `Open()` → file.Read() | `vfs.Open()` → `Handle.ReadAt()` |
| WRITE | FromHandle → billy | B | `OpenFile()` → file.Write() | `vfs.OpenFile()` → `Handle.WriteAt()` |
| CREATE | FromHandle → billy | B | `Create()` | `vfs.Create()` |
| MKDIR | FromHandle → billy | B (A) | `MkdirAll()` | ⚠️ 手动 `vfs.Stat()` + `vfs.Mkdir()`（A 类内部） |
| REMOVE | FromHandle → billy | B | `Remove()` | `vfs.Remove()` |
| RMDIR | FromHandle → billy | B | `Remove()` | `vfs.Remove()` |
| RENAME | FromHandle → billy | B | `Rename()` | `vfs.Rename()` |
| READLINK | FromHandle → billy | B | `Readlink()` | `vfs.Readlink()` |
| SYMLINK | FromHandle → billy | B | `Symlink()` | `vfs.Symlink()` |
| SETATTR | FromHandle → billy | B | `Chmod/Chtimes/Truncate` | `vfs.Open()`→Chmod / `vfs.Chtimes()` |
| FSSTAT | Handler.FSStat | **A** | — | ⚡ 直接 `h.vfs.Statfs()` |
| MOUNT | Handler.Mount | **A** | — | ⚡ 直接 `h.vfs.Stat()` + `subFS()` |

---

## 9. 句柄管理的边界

### 9.1 两种"句柄"的严格区分

| | NFS 文件句柄 | VFS Handle |
|--|-------------|------------|
| **类型** | `[]byte` | `vfs.Handle`（接口） |
| **产生** | `Cache.ToHandle(billyFS, splitPath)` | `vfs.Open()` / `vfs.Create()` |
| **含义** | 标识一个路径在某个 billy FS 中的位置 | 标识一个已打开文件的读写状态 |
| **生命周期** | Cache 管理，直到被回收或 InvalidateHandle | 文件 Close() 后释放 |
| **作用域** | 跨请求持久（客户端持有） | 单次 Open-Close 间 |
| **所在层** | Handler / Cache | VFS |

### 9.2 句柄缓存的分层结构

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
    │
    ├── diskHandler (disk)        ← cache.go 实现
    │   磁盘文件 MD5(path) → path字符串
    │
    └── diskHandler (symlink)     ← symlink_cache_linux.go 实现
        磁盘符号链接 MD5(path) → path
        句柄 = name_to_handle_at() 内核文件句柄
```

### 9.3 pathRewriter 与子路径挂载的交互

`cmd/serve/nfs/cache.go#L85-L135` 中 `pathRewriter` 保证：

- 客户端 A 通过 `mount :/sub` 访问 `hello.txt` → `ToHandle(subFS, ["hello.txt"])` → translate 为 `(rootFS, ["sub","hello.txt"])`
- 客户端 B 通过 `mount :/` 访问 `sub/hello.txt` → `ToHandle(rootFS, ["sub","hello.txt"])` → translate 不变
- 两者得到**完全相同的句柄字节**（测试见 `cmd/serve/nfs/handler_test.go#L135-L143`、`cmd/serve/nfs/cache_test.go#L176-L220`）

**关键：** `FromHandle` 总是返回 rootFS，因为 ToHandle 时已归一化。后续所有 billy 调用都通过 rootFS 进行——rootFS 的 `fullPath()` 因 `root == ""` 而不做任何前缀拼接。

---

## 10. 子路径挂载的边界

### 10.1 完整流程

```
1. NFS Client: MOUNTPROC3_MNT(Dirpath = "/photos/2024")
                    │
2. Handler.Mount()
   ├─ path.Clean("/photos/2024") = "/photos/2024"
   │
   ├─ h.vfs.Stat("/photos/2024")    ← ⚡ A1 绕过 billy
   │   ├─ 不存在 → MountStatusErrNoEnt
   │   └─ 非目录 → MountStatusErrNotDir
   │
   └─ 返回 subFS = h.billyFS.subFS("/photos/2024")
                    │
3. go-nfs: ToHandle(subFS, [])
   └─ pathRewriter.translate(subFS, [])
       └─ (rootFS, ["photos", "2024"])
   → 根句柄返回给客户端
                    │
4. 后续操作: FromHandle(fh)
   → 总是返回 (rootFS, absoluteSplitPath)
   → rootFS.fullPath(name) 因 root=="" 直接返回 name
   → billy 方法调用 VFS
```

### 10.2 安全保证

`cmd/serve/nfs/handler.go#L68`：`cleaned := path.Clean("/" + string(req.Dirpath))`

- 前置 `/` 保证绝对路径
- `path.Clean` 消除 `..`：`"/../../etc"` → `"/etc"`（无法穿越 VFS 根）
- 测试：`cmd/serve/nfs/handler_test.go#L84-L108` 的 traversal 用例

### 10.3 fullPath 路径重写

`cmd/serve/nfs/filesystem.go#L53-L58`：

```go
func (f *FS) fullPath(name string) string {
    if f.root == "" { return name }                   // rootFS：直通
    return path.Join(f.root, name)                    // subFS：前缀拼接
}
```

但注意：子路径挂载**不提供安全隔离**，只是便捷功能。文档明确说明等价于 mount `/` 然后 cd 进子目录，句柄全局共享。

---

## 11. 属性注入的边界

### 11.1 问题

billy 接口的 `os.FileInfo` 不暴露 uid/gid/inode，但 NFS GETATTR/LOOKUP 响应必须包含这些字段。

### 11.2 setSys 机制

`cmd/serve/nfs/filesystem.go#L28-L43`：

```go
func setSys(fi os.FileInfo) {
    node, ok := fi.(vfs.Node)    // FileInfo 实质就是 vfs.Node
    vfs := node.VFS()
    stat := file.FileInfo{
        Nlink:  1,
        UID:    vfs.Opt.UID,     // ← C6 读取配置
        GID:    vfs.Opt.GID,     // ← C7 读取配置
        Fileid: node.Inode(),    // ← B17 读取 inode (Linux 必填)
    }
    node.SetSys(&stat)           // 注入到 vfs.Node.Sys()
}
```

### 11.3 setSys 调用时机矩阵

| billy 方法 | setSys | fullPath | 类别 | 原因 |
|-----------|--------|----------|------|------|
| `Stat()` | ✅ | ✅ | B5 | 返回单个 FileInfo |
| `Lstat()` | ✅ | ✅ | B8 | 返回单个 FileInfo |
| `ReadDir()` | ✅ 循环 | ✅ | B1 | 返回 FileInfo 列表 |
| `Create()` | ❌ | ✅ | B2 | 返回 billy.File |
| `Open()` | ❌ | ✅ | B3 | 返回 billy.File |
| `OpenFile()` | ❌ | ✅ | B4 | 返回 billy.File |
| `Rename()` | ❌ | ✅✅ | B6 | 无返回值 |
| `Remove()` | ❌ | ✅ | B7 | 无返回值 |
| `MkdirAll()` | ❌ | ✅ | B14/B15 | 无 FileInfo |
| `Symlink()` | ❌ | ✅ (仅 link) | B9 | 无返回值 |
| `Readlink()` | ❌ | ✅ | B10 | 返回字符串 |
| `Chmod()` | ❌ | ✅ | B11 | 无返回值 |
| `Chown()` | ❌ | ✅ | B12 | 无返回值 |
| `Chtimes()` | ❌ | ✅ | B13 | 无返回值 |

### 11.4 层间协议：VFS 同时服务 FUSE 和 NFS

这种设计使得 VFS 层可以同时为两个前端服务，各自有独立的属性填充机制：

- **FUSE 前端**（mount2）：`cmd/mount2/fs.go#L69-L105` 的 `setAttr`/`setAttrOut` 直接写 `fuse.Attr` 结构体
- **NFS 前端**（serve nfs）：`setSys` 将属性注入 `vfs.Node.Sys()`，由 go-nfs 通过 `Sys()` 提取

VFS 层对两者完全无知，是干净的关注点分离。

---

## 12. mount2 (FUSE) 与 NFS 的架构对比

| 维度 | mount2 (FUSE) | serve nfs |
|------|--------------|-----------|
| 协议 | FUSE (/dev/fuse) | NFSv3 (TCP) |
| 依赖库 | hanwen/go-fuse/v2 | willscott/go-nfs |
| 适配接口 | `fusefs.InodeEmbedder` | `billy.Filesystem` |
| 文件标识 | `Node` (嵌入 `fusefs.Inode`) | `[]byte` 句柄 (Cache 管理) |
| 文件句柄 | `FileHandle` (显式包装 `vfs.Handle`) | 无独立句柄对象，每次 `FromHandle`→`billy.Open` |
| 路径查找 | `Node.Lookup` → `vfs.Dir.Stat` | `FromHandle` → `billyFS.Stat` |
| 目录读取 | `dirStream` + `vfs.Handle.Readdir` | `billyFS.ReadDir` |
| 属性填充 | `setAttr/setAttrOut` 直接写 fuse.Attr | `setSys` 注入 vfs.Node.Sys() |
| VFS A 类访问 | `FS.Root()` → `vfs.Root()` | `Mount()`→`vfs.Stat()`, `FSStat()`→`vfs.Statfs()`, `MkdirAll`→手动 |
| VFS B 类访问 | Node/FileHandle 方法全部通过 | billy FS 全部方法 |
| VFS C 类访问 | AttrTimeout、UID、GID | CacheMode、UID/GID、MetadataExtension |
| 子路径 | 不支持 | `Mount()` 支持 subFS |
| 错误映射 | `translateError()` → syscall.Errno | billy/go-nfs 内部处理 |
| 平台 | Linux / macOS (amd64) | 所有 Unix |

---

## 13. 结论：边界规则的一句话总结

> **绝大多数 VFS 调用走 B 类（经 billy 适配）；只有 Mount 路径验证（A1）、FSSTAT 空间统计（A2）、MkdirAll 权限透传（A3/A4）四处需要绕过 billy 直接访问 VFS；其余涉及 VFS 的读取均为 C 类（只读配置），不具文件系统语义。**
