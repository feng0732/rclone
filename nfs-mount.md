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

## 2. VFS 访问边界的三类划分（核准版）

Handler 结构体同时持有 `*vfs.VFS` 和 `*FS`（billy 适配器）两个引用，但 VFS 的访问在代码中实际上分布在三个**严格不同的层级和语义**中：

| 类别 | 定义 | 判定依据 | 代表 |
|------|------|---------|------|
| **类别一：NFS 层直接调用 VFS** | 调用点**不在** `billy.Filesystem` 接口方法内部，直接由 NFS 服务层（Handler / Server / nfsmount）触发 | 代码在 `Handler.Mount`、`Handler.FSStat`、`nfsmount` 中，不进入 FS 结构体的 billy 方法 | Mount 路径验证、FSSTAT 空间统计、VFS 生命周期关闭 |
| **类别二：billy 适配层内部特殊调用** | 调用点在 billy 接口方法实现内部，但**不是简单的同名方法一一映射**，在适配层内部做了组合、循环、间接反查等特殊处理 | 虽然被 FS.XXX() 包裹，但实际调用的 VFS 方法不是直接对应（如 MkdirAll 用循环而非 `vfs.MkdirAll`；Chmod 先 Open 再 Chmod；setSys 从 Node 反查 VFS） | MkdirAll 递归建目录、Chmod/Chown 先 Open 再操作、setSys 中 node.VFS()/node.Inode() |
| **类别三：仅读取配置** | 只读取 `vfs.Opt.*` 字段或 `vfs.Fs()`（后端 fs 对象），不触发任何文件系统 I/O 操作 | 访问路径为 `vfs.Opt.XXX` 或 `vfs.Fs()`，没有方法调用语义（`Stat`/`Read`/`Mkdir` 等） | CacheMode 判断、UID/GID 注入、构造缓存目录名、Root() 返回后端路径 |

**另：billy 常规一一映射**（非特殊、非本类重点，但作为参照列）：
FS.ReadDir → vfs.ReadDir, FS.Create → vfs.Create, FS.Open → vfs.Open, FS.OpenFile → vfs.OpenFile, FS.Stat → vfs.Stat, FS.Rename → vfs.Rename, FS.Remove → vfs.Remove, FS.Lstat → vfs.Stat, FS.Symlink → vfs.Symlink, FS.Readlink → vfs.Readlink, FS.Chtimes → vfs.Chtimes。这些属于最简单的"billy 方法名 → VFS 同名方法"直接对应。

---

## 3. 类别一：NFS 层直接调用 VFS（4处）

**分类判定标准：调用函数不在 `FS` 结构体（billy 适配层）的任何方法内部。**

这些访问由 go-nfs 通过 `nfs.Handler` 接口直接回调到 Handler 层，Handler 层无法也不应该通过 billy 接口完成使命。

| # | 位置 | 代码 | 所在方法 / 层 | 分类理由 |
|---|------|------|-------------|---------|
| I-1 | `cmd/serve/nfs/handler.go#L72` | `h.vfs.Stat(cleaned)` | `Handler.Mount`（nfs.Handler 接口） | Mount 的职责是**决定**返回哪个 billy FS（rootFS 还是 subFS）。在决策之前 billy FS 身份尚未确定，无法通过 billy 接口验证路径。这是**架构层面的必然穿越**。 |
| I-2 | `cmd/serve/nfs/handler.go#L95` | `h.vfs.Statfs()` | `Handler.FSStat`（nfs.Handler 可选接口） | `billy.Filesystem` 接口**根本没有**空间统计方法，但 NFSv3 的 `NFSPROC3_FSSTAT` 过程必须响应 total/free/available。**billy 接口设计的缺口**。 |
| I-3 | `cmd/nfsmount/nfsmount.go#L101` | `VFS.Shutdown()` | `unmount` 闭包（nfsmount 层） | VFS 生命周期管理。在 unmount 时连同 NFS Server 一起关闭，不属于文件系统操作。 |
| I-4 | `cmd/nfsmount/nfsmount.go#L114` | `VFS.Shutdown()` | `OnUnmountFunc` 回调（nfsmount 层） | 同上，当 NFS 客户端主动 umount 时，触发回调关闭 VFS。 |

### I-1 调用链（Mount 路径验证）

```
NFS Client 发送 MOUNTPROC3_MNT(Dirpath = "/photos/2024")
    │
    ▼
go-nfs 协议层 (MOUNT 服务端)
    │
    ▼
Handler.Mount()                          ← nfs.Handler 接口回调
    │
    ├─ path.Clean("/" + req.Dirpath)     ← 规范化，消除 ".."
    │
    ├─ h.vfs.Stat(cleaned)                ← ⚡ I-1 类别一：NFS层直接调VFS
    │   └─ 不存在 → MountStatusErrNoEnt
    │   └─ 非目录 → MountStatusErrNotDir
    │
    └─ cleaned == "/" ?
        ├─ 是 → 返回 h.billyFS (rootFS)
        └─ 否 → 返回 h.billyFS.subFS(cleaned)
```

### I-2 调用链（FSSTAT）

```
NFS Client 发送 NFSPROC3_FSSTAT
    │
    ▼
go-nfs 协议层
    │
    ▼
Handler.FSStat(ctx, billyFS, &s)         ← nfs.Handler 可选接口回调
    │
    ├─ 注意：参数 billyFS 被**完全忽略**
    │
    └─ total, _, free := h.vfs.Statfs()   ← ⚡ I-2 类别一：NFS层直接调VFS
        ├─ s.TotalSize = uint64(total)
        ├─ s.FreeSize = uint64(free)
        └─ s.AvailableSize = uint64(free)
```

---

## 4. 类别二：billy 适配层内部特殊调用（6处）

**分类判定标准：调用点在 `FS` 结构体的 billy 接口方法内部，但不是 "FS.XXX → f.vfs.XXX" 的简单同名映射。**

常规 billy 适配是 `FS.Stat(filename)` → `f.vfs.Stat(filename)`，方法名、参数签名几乎一一对应。而"特殊调用"在适配层内部做了额外的组合逻辑。

| # | 位置 | 代码 | 所属 billy 方法 | 特殊之处 | 直接的 vfs 对应方法为什么不用 |
|---|------|------|----------------|---------|---------------------------|
| II-1 | `cmd/serve/nfs/filesystem.go#L148` | `f.vfs.Stat(current)` | `FS.MkdirAll` | 循环逐级检查每个路径组件是否存在，而不是调用对应方法 | 存在 `vfs.MkdirAll(name, perm)`，但代码注释声称其"doesn't honor the permissions"（不遵循权限）；详见本节附录的代码事实核查 |
| II-2 | `cmd/serve/nfs/filesystem.go#L150` | `f.vfs.Mkdir(current, perm)` | `FS.MkdirAll` | 循环逐级创建，每次传递相同 perm | 同上 |
| II-3 | `cmd/serve/nfs/filesystem.go#L189` | `f.vfs.Open(name)` → `file.Chmod()` | `FS.Chmod` | 先 Open 拿到 vfs.Handle，再通过 Handle 调用 Chmod | VFS 顶层没有 `vfs.Chmod(path, mode)` 方法；只有 Handle 级别的 `file.Chmod()` |
| II-4 | `cmd/serve/nfs/filesystem.go#L216` | `f.vfs.Open(name)` → `file.Chown()` | `FS.Chown` | 先 Open 拿到 vfs.Handle，再通过 Handle 调用 Chown | VFS 顶层没有 `vfs.Chown(path, uid, gid)` 方法；只有 Handle 级别的 `file.Chown()` |
| II-5 | `cmd/serve/nfs/filesystem.go#L34` | `node.VFS()` | `setSys`（被 Stat/Lstat/ReadDir 间接调用） | 通过类型断言 `fi.(vfs.Node)` 从 FileInfo 反查 VFS 引用 | billy 接口没有办法把 VFS 引用作为参数传入；只能利用 vfs.Node 同时实现了 os.FileInfo 的特性做反查 |
| II-6 | `cmd/serve/nfs/filesystem.go#L40` | `node.Inode()` | `setSys` | 从 Node 获取 inode 号作为 NFS fileid | fileid 是 NFS 属性的必填字段，Linux 客户端依赖它判断文件同一性；billy 接口没有单独暴露 inode |

### II-1 / II-2 详细分析：MkdirAll 递归建目录的特殊处理（含代码事实核查）

**代码片段**（`cmd/serve/nfs/filesystem.go#L139-L157`）：

```go
// MkdirAll creates a directory and all the ones above it
// it does not redirect to VFS.MkDirAll because that one doesn't
// honor the permissions
func (f *FS) MkdirAll(filename string, perm os.FileMode) (err error) {
    filename = f.fullPath(filename)                   // ① 统一先 fullPath 重写
    parts := strings.Split(filename, "/")
    for i := range parts {
        current := strings.Join(parts[:i+1], "/")
        _, err := f.vfs.Stat(current)                 // ② ⚡ II-1: 逐级检查
        if err == vfs.ENOENT {
            err = f.vfs.Mkdir(current, perm)          // ③ ⚡ II-2: 逐级创建
            ...
        }
    }
    return nil
}
```

#### 附录：MkdirAll 代码事实核查

以下是对注释和代码的层层核对，纠正了之前版本文档中的事实错误。

| 核查项 | VFS 代码事实 | 结论 |
|--------|-------------|------|
| **VFS.MkdirAll 是否接受 perm 参数？** | `vfs/vfs.go#L790`：<br>`func (vfs *VFS) MkdirAll(name string, perm os.FileMode) error` | ✅ **接受 perm**。之前文档中"不接受 perm 参数"的说法**错误**。 |
| **VFS.Mkdir 是否接受 perm 参数？** | `vfs/vfs.go#L757`：<br>`func (vfs *VFS) Mkdir(name string, perm os.FileMode) error` | ✅ **接受 perm**。 |
| **perm 参数实际在哪里被丢弃？** | `vfs/vfs.go#L747-L753` 中 `vfs.mkdir(name, perm)` 内部调用 `dir.Mkdir(leaf)`；<br>`vfs/dir.go#L1066` 中 `Dir.Mkdir(name)` **签名没有 perm**，<br>最终调用 `d.f.Mkdir(ctx, path)`（fs.Fs 后端接口也无 perm） | ❌ perm 在 `Dir.Mkdir` 处被**静默丢弃**。注释 "doesn't honor the permissions" 在这个意义上**属实**——VFS 接口有 perm 参数但**实际不生效**。 |
| **NFS 手动循环能让 perm 生效吗？** | NFS 循环调用的是 `f.vfs.Mkdir(current, perm)`，<br>最终同样走到 `Dir.Mkdir(leaf)`，同样丢弃 perm | ❌ **同样无法让 perm 生效**。注释给出的理由在逻辑上是**自相矛盾**的——两种方式 perm 都会被丢弃。 |
| **两种实现的路径处理有差异吗？** | NFS 版：`strings.Split → strings.Join`，**不做** `strings.Trim`，<br>对 `"/a/b/c"` 生成 `""`、`"/a"`、`"/a/b"`、`"/a/b/c"`<br><br>VFS 版：`mkdirAll` 第一句 `strings.Trim(name, "/")`，<br>递归时 `path.Split(parent)`，只处理无前后斜杠的路径 | ⚠️ 两者对路径规范化的处理有**微妙差异**。VFS.Stat 入口也会做 `Trim("/")`（`vfs/vfs.go#L484`），<br>所以行为在功能上等价，但代码路径不同。 |

#### 真实结论（基于代码）

- 注释 "doesn't honor the permissions" 描述的客观现象正确（perm 不生效），但**不能作为手动循环的正当理由**——循环方式同样无法让 perm 生效。
- 手动循环的**真正原因无法从代码中直接确认**。可能原因：① 历史遗留（早期 VFS.MkdirAll 没有 perm，后来 VFS 接口扩展了但 NFS 适配层没跟进，注释没改）；② 为了与 billy 层的路径语义完全一致（不依赖 VFS 内部的 Trim 行为）。
- 但无论原因是什么，**调用位置和层归属是明确的**：`f.vfs.Stat` 和 `f.vfs.Mkdir` 在 `FS.MkdirAll` 方法体内，执行了 billy 层标准的 `fullPath()` 路径重写，调用者是 billy 接口方法的实现——因此分类为**类别二：billy 适配层内部特殊调用**（非常规同名映射）。

#### 为什么属于"billy适配层内部特殊调用"而不是"NFS层直接调用"

- 调用点**在** `FS.MkdirAll` 方法体内，它本身就是 `billy.Filesystem` 接口的实现。
- 执行了 `fullPath()` 重写（subFS 时会拼接 root 前缀），这是 billy 适配层的标准职责。
- 只是因为"实际行为上未调用对应的 `vfs.MkdirAll`"（而用低级 Stat+Mkdir 组合），才被标记为"特殊"而非"常规 R 类"。
- 相比之下，类别一（I-1, I-2）根本不进入 billy 适配层。

### II-3 / II-4 详细分析：Chmod/Chown 先 Open 再操作

**代码片段**（`cmd/serve/nfs/filesystem.go#L185-L226`）：

```go
func (f *FS) Chmod(name string, mode os.FileMode) error {
    name = f.fullPath(name)
    file, err := f.vfs.Open(name)            // ⚡ II-3: 先 Open 拿 Handle
    defer file.Close()
    err = file.Chmod(mode)                   // 再调用 Handle.Chmod
    if err == vfs.ENOSYS { err = nil }       // 后端不支持则静默
    return err
}
```

同理，Chown 也是 `f.vfs.Open(name)` → `file.Chown(uid, gid)`。

这里的"特殊"体现在：VFS 的接口设计中，Chmod/Chown 是**Handle 级**操作（不是 VFS 级），而 billy 将它们暴露为**路径级**方法。适配层必须多做一步 Open→Chmod→Close 的流程转换。

### II-5 / II-6 详细分析：setSys 的反查模式

**代码片段**（`cmd/serve/nfs/filesystem.go#L28-L43`）：

```go
func setSys(fi os.FileInfo) {
    node, ok := fi.(vfs.Node)              // 类型断言：FileInfo 其实就是 vfs.Node
    vfs := node.VFS()                      // ⚡ II-5: 从 Node 反查 VFS
    stat := file.FileInfo{
        Nlink:  1,
        UID:    vfs.Opt.UID,               // ← 属于 III-5：只读取配置
        GID:    vfs.Opt.GID,               // ← 属于 III-6：只读取配置
        Fileid: node.Inode(),              // ⚡ II-6: 从 Node 获取 inode
    }
    node.SetSys(&stat)                     // 注入到 Node.Sys()
}
```

- II-5 `node.VFS()`：**VFS 内部的反向关联**。billy 调用链只传 `os.FileInfo`，不传 `*vfs.VFS` 参数。由于 `vfs.Node` 恰好持有对 VFS 的反向引用，适配层才能从 Node 反查到 VFS 配置。这是架构层的"特殊"模式。
- II-6 `node.Inode()`：inode 属于**文件自身的属性数据**（不是配置），必须从具体 Node 获取，且 Linux NFS 客户端强依赖此值。

---

## 5. 类别三：仅读取配置（8处）

**分类判定标准：访问的是 `vfs.Opt.*` 字段或 `vfs.Fs()` 对象，没有任何文件系统 I/O 方法调用。**

这些访问虽然"触碰了 VFS 对象"，但本质是读取元数据配置。在概念上，它们也可以被理解为"NFS 层从 VFS 读取配置"，而非"调用 VFS 文件系统操作"。

| # | 位置 | 代码 | 所属方法 | 用途 |
|---|------|------|---------|------|
| III-1 | `cmd/serve/nfs/server.go#L28` | `vfs.Opt.CacheMode` | `NewServer` | Server 创建时判断是否 `CacheModeOff`，若是发出"NFS 只读"警告 |
| III-2 | `cmd/serve/nfs/filesystem.go#L253` | `f.vfs.Opt.CacheMode` | `FS.Capabilities` | 决定 billy 暴露的能力：只读 vs 可读写 |
| III-3 | `cmd/serve/nfs/cache.go#L156` | `h.vfs.Fs()` → `fs.ConfigString()` | `newDiskHandler` | 构造磁盘缓存目录名：用 remote 的配置字符串做哈希名，保证不同 remote 有独立缓存 |
| III-4 | `cmd/serve/nfs/cache.go#L173` | `h.vfs.Opt.MetadataExtension` | `newDiskHandler` | 读取元数据扩展名，用于 metadataSuffix 的合成/剥离句柄逻辑 |
| III-5 | `cmd/serve/nfs/filesystem.go#L38` | `vfs.Opt.UID` | `setSys`（通过 II-5 的 `node.VFS()`） | 注入 UID 到 FileInfo.Sys()，NFS 属性响应的 fattr3.uid |
| III-6 | `cmd/serve/nfs/filesystem.go#L39` | `vfs.Opt.GID` | `setSys`（通过 II-5 的 `node.VFS()`） | 注入 GID 到 FileInfo.Sys()，NFS 属性响应的 fattr3.gid |
| III-7 | `cmd/serve/nfs/filesystem.go#L245` | `f.vfs.Fs().Root()` | `FS.Root()`（rootFS 分支） | 返回后端 fs 的 root 路径，满足 billy `Root()` 接口要求 |
| III-8 | `cmd/serve/nfs/filesystem.go#L247` | `f.vfs.Fs().Root()` | `FS.Root()`（subFS 分支） | subFS 时拼接 `f.root` 前缀后返回 |

**交叉引用**：III-5 和 III-6 在调用链上依赖 II-5（`node.VFS()`）先拿到 VFS 引用——分类的不同点在于：II-5 是"从 Node 反查 VFS"的**机制动作**；III-5/III-6 是在拿到 VFS 引用后读取其**配置字段**。两者在同一函数内，但语义不同。

---

## 6. 对照：billy 适配层常规调用（非特殊）

作为对比，下面这些调用是 billy → VFS 的**同名直接映射**，模式统一，不需要列为"特殊调用"：

| # | 位置 | billy 方法 → VFS 方法 | fullPath 重写 | setSys 属性注入 |
|---|------|---------------------|--------------|---------------|
| R-1 | `filesystem.go#L70` | `ReadDir(p)` → `vfs.ReadDir(fullP)` | ✅ | ✅ 循环 |
| R-2 | `filesystem.go#L84` | `Create(f)` → `vfs.Create(fullF)` | ✅ | ❌ |
| R-3 | `filesystem.go#L91` | `Open(f)` → `vfs.Open(fullF)` | ✅ | ❌ |
| R-4 | `filesystem.go#L98` | `OpenFile(f,flag,perm)` → `vfs.OpenFile(fullF,...)` | ✅ | ❌ |
| R-5 | `filesystem.go#L105` | `Stat(f)` → `vfs.Stat(fullF)` | ✅ | ✅ |
| R-6 | `filesystem.go#L118` | `Rename(o,n)` → `vfs.Rename(fullO,fullN)` | ✅✅ | ❌ |
| R-7 | `filesystem.go#L125` | `Remove(f)` → `vfs.Remove(fullF)` | ✅ | ❌ |
| R-8 | `filesystem.go#L163` | `Lstat(f)` → `vfs.Stat(fullF)` | ✅ | ✅ |
| R-9 | `filesystem.go#L175` | `Symlink(target, link)` → `vfs.Symlink(target, fullLink)` | ✅ 仅 link | ❌ |
| R-10 | `filesystem.go#L182` | `Readlink(link)` → `vfs.Readlink(fullLink)` | ✅ | ❌ |
| R-11 | `filesystem.go#L232` | `Chtimes(n,atime,mtime)` → `vfs.Chtimes(fullN,...)` | ✅ | ❌ |

这些常规调用 + 类别二的特殊调用（II-1~II-6）共同构成 billy 适配层与 VFS 的全部交互。

---

## 7. 三类访问的可视化边界图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        go-nfs 协议库                                  │
│            NFSv3 RPC 解码/编码 / MOUNT / 过程分发                     │
└────────────┬──────────────────────────────┬───────────────────────────┘
             │                              │
   billy.Filesystem 接口            nfs.Handler 接口
             │                              │
┌────────────▼──────────────────────────────▼───────────────────────────────┐
│                           NFS 层整体（Handler + billy FS + Cache）         │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  nfs.Handler 接口层 (handler.go)                                    │  │
│  │                                                                     │  │
│  │  ⚡ I-1  Mount() ──────── h.vfs.Stat(cleaned)                       │  │
│  │  ⚡ I-2  FSStat() ─────── h.vfs.Statfs()                            │  │
│  │                                                                     │  │
│  │  ToHandle / FromHandle / InvalidateHandle ──► Cache                 │  │
│  │  Change() ──► billy.Change (FS)                                     │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  billy 适配层 (filesystem.go: FS 实现 billy.Filesystem)             │  │
│  │                                                                     │  │
│  │  【常规一一映射 R-1 ~ R-11】ReadDir/Create/Open/Stat/...            │  │
│  │  统一：fullPath() 重写 + f.vfs.同名方法 + 按需 setSys               │  │
│  │                                                                     │  │
│  │  【⭐ 类别二：特殊调用 II-1 ~ II-6】                                 │  │
│  │  II-1,2 MkdirAll: 循环 f.vfs.Stat + f.vfs.Mkdir (注释称 perm 不生效)│  │
│  │  II-3,4 Chmod/Chown: f.vfs.Open → file.Chmod/Chown → Close         │  │
│  │  II-5,6 setSys:   node.VFS() (反查) + node.Inode() (inode)         │  │
│  │                                                                     │  │
│  │  【⚙ 类别三：仅读取配置 III-1~III-8】                                │  │
│  │  III-1,2  vfs.Opt.CacheMode       → 警告 / Capabilities            │  │
│  │  III-3,4  h.vfs.Fs() / Opt         → 缓存目录 / metadata 扩展名    │  │
│  │  III-5,6  vfs.Opt.UID/GID          → setSys 注入属性               │  │
│  │  III-7,8  f.vfs.Fs().Root()        → billy Root() 返回             │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  句柄缓存层 (cache.go) — III-3, III-4 也在这里执行                 │  │
│  │  pathRewriter (外层) + 3种 inner Cache (memory/disk/symlink)       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
│  ⚡ I-3  nfsmount unmount 闭包: VFS.Shutdown()                             │
│  ⚡ I-4  nfsmount OnUnmountFunc: VFS.Shutdown()                            │
└────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                          ┌─────────────────────────┐
                          │        vfs.VFS           │
                          │  不感知 NFS / billy      │
                          └─────────────────────────┘
```

---

## 8. 各类访问的合理性评估

### 类别一（NFS 层直接调用 VFS）

| # | 合理性 | 说明 |
|---|--------|------|
| I-1 `Handler.Mount` → `h.vfs.Stat` | ✅ 必然 | Mount 的职责就是决策 billy FS，决策前 billy FS 不存在。架构上无法通过 billy 接口。 |
| I-2 `Handler.FSStat` → `h.vfs.Statfs` | ✅ 合理 | billy.Filesystem 接口根本没有 FSStat 方法。 |
| I-3/I-4 `VFS.Shutdown()` | ✅ 合理 | VFS 生命周期管理，不属于文件系统操作。 |

### 类别二（billy 适配层内部特殊调用）

| # | 合理性 | 说明 |
|---|--------|------|
| II-1/II-2 MkdirAll 循环 Stat+Mkdir | ⚠️ 注释与代码不一致 | `vfs.MkdirAll(name, perm)` **存在且接受 perm**（`vfs/vfs.go#L790`），但 perm 实际在 `Dir.Mkdir`（`vfs/dir.go#L1066`）处被静默丢弃。注释 "doesn't honor the permissions" 客观现象正确，但循环方式同样无法让 perm 生效。真正的循环原因无法从代码直接确认（可能是历史遗留）。层归属明确：在 billy 方法内，经 fullPath 重写 → 类别二。 |
| II-3/II-4 Chmod/Chown 先 Open 再操作 | ✅ 设计匹配 | VFS 选择了"Handle 级 Chmod/Chown"而非"路径级"，是后端兼容性考量（有些后端只能对打开的文件改权限）。 |
| II-5 `node.VFS()` 反查 VFS | ✅ 巧妙依赖 | billy 接口参数链限制导致只能利用 Node 反向引用。虽然间接，但依赖的是 VFS 内部稳定的关联。 |
| II-6 `node.Inode()` | ✅ 必要 | Linux NFS 客户端必填。属于文件自身属性，不是配置。 |

### 类别三（仅读取配置）

| # | 合理性 | 说明 |
|---|--------|------|
| III-1 ~ III-8 | ✅ 全部合理 | 纯配置读取，没有文件系统语义。 |

---

## 9. 请求映射与子路径挂载边界

### 9.1 NFS 过程 → 调用类别完整映射表

| NFSv3 过程 | 入口接口 | VFS 访问类别 | billy 方法 | VFS 最终方法 |
|-----------|---------|-------------|-----------|-------------|
| GETATTR | FromHandle → billy | **R**（常规） | `Stat()` | `vfs.Stat()` + setSys |
| LOOKUP | FromHandle → billy | **R**（常规） | `Stat()` | `vfs.Stat()` + setSys |
| READDIR | FromHandle → billy | **R**（常规） | `ReadDir()` | `vfs.ReadDir()` + setSys |
| READDIRPLUS | FromHandle → billy | **R**（常规） | `ReadDir()` + `Stat()` | `vfs.ReadDir()` + `vfs.Stat()` |
| READ | FromHandle → billy | **R**（常规） | `Open()` → file.Read() | `vfs.Open()` → `Handle.ReadAt()` |
| WRITE | FromHandle → billy | **R**（常规） | `OpenFile()` → file.Write() | `vfs.OpenFile()` → `Handle.WriteAt()` |
| CREATE | FromHandle → billy | **R**（常规） | `Create()` | `vfs.Create()` |
| MKDIR | FromHandle → billy | **⭐ 类别二** | `MkdirAll()` | ⚠️ 循环 `vfs.Stat()` + `vfs.Mkdir()` (II-1,II-2)；注意：注释声称 `vfs.MkdirAll` 不 honor perm，但实际签名接受 perm，只是两种方式 perm 都在 Dir.Mkdir 处被丢弃 |
| REMOVE | FromHandle → billy | **R**（常规） | `Remove()` | `vfs.Remove()` |
| RMDIR | FromHandle → billy | **R**（常规） | `Remove()` | `vfs.Remove()` |
| RENAME | FromHandle → billy | **R**（常规） | `Rename()` | `vfs.Rename()` |
| READLINK | FromHandle → billy | **R**（常规） | `Readlink()` | `vfs.Readlink()` |
| SYMLINK | FromHandle → billy | **R**（常规） | `Symlink()` | `vfs.Symlink()` |
| SETATTR (chmod) | FromHandle → billy | **⭐ 类别二** | `Chmod()` | ⚠️ `vfs.Open()` → `Chmod()` (II-3) |
| SETATTR (chown) | FromHandle → billy | **⭐ 类别二** | `Chown()` | ⚠️ `vfs.Open()` → `Chown()` (II-4) |
| SETATTR (truncate) | FromHandle → billy | **R**（常规） | OpenFile→Truncate | `vfs.OpenFile()` → Handle.Truncate |
| FSSTAT | Handler.FSStat | **⚡ 类别一** | — | ⚡ 直接 `h.vfs.Statfs()` (I-2) |
| MOUNT | Handler.Mount | **⚡ 类别一** | — | ⚡ 直接 `h.vfs.Stat()` + `subFS()` (I-1) |

### 9.2 子路径挂载的完整流程

```
1. NFS Client: MOUNTPROC3_MNT(Dirpath = "/photos/2024")
                     │
2. Handler.Mount()  ← 类别一 (I-1)
   ├─ path.Clean("/" + "/photos/2024") = "/photos/2024"
   │
   ├─ h.vfs.Stat("/photos/2024")        ← ⚡ I-1 NFS层直接调VFS
   │   ├─ 不存在 → MountStatusErrNoEnt
   │   └─ 非目录 → MountStatusErrNotDir
   │
   └─ 返回 subFS = h.billyFS.subFS("/photos/2024")
                     │
3. go-nfs: ToHandle(subFS, [])
   └─ pathRewriter.translate(subFS, [])
       └─ (rootFS, ["photos", "2024"])   ← 归一化
   → 根句柄返回给客户端
                     │
4. 后续操作: FromHandle(fh)
   → 总是返回 (rootFS, absoluteSplitPath)
   → rootFS.fullPath(name) 因 root=="" 直接返回 name
   → billy 方法 → 类别二 或 常规 R 调用 VFS
```

安全保证（`cmd/serve/nfs/handler.go#L68`）：`cleaned := path.Clean("/" + string(req.Dirpath))` 前置 `/` 保证绝对路径，`path.Clean` 消除 `..`。测试：`cmd/serve/nfs/handler_test.go#L84-L108` traversal 用例。

---

## 10. 句柄管理的边界

### 10.1 两种"句柄"的严格区分

| | NFS 文件句柄 | VFS Handle |
|--|-------------|------------|
| **类型** | `[]byte` | `vfs.Handle`（接口） |
| **产生** | `Cache.ToHandle(billyFS, splitPath)` | `vfs.Open()` / `vfs.Create()` |
| **含义** | 标识一个路径在某个 billy FS 中的位置 | 标识一个已打开文件的读写状态 |
| **生命周期** | Cache 管理，直到被回收或 InvalidateHandle | 文件 Close() 后释放 |
| **作用域** | 跨请求持久（客户端持有） | 单次 Open-Close 间 |
| **所在层** | Handler / Cache | VFS |

### 10.2 句柄缓存分层结构

```
Handler.ToHandle(f, path) / Handler.FromHandle(fh)
    │
    ▼
pathRewriter  ← 外层：统一子路径到根路径
    │  translate(f, splitPath) → (rootFS, absoluteSplitPath)
    │
    ▼
inner Cache  ← 内层：三种策略（构造时读取 III-3, III-4 配置）
    │
    ├── CachingHandler (memory)   ← map[ID] → (billyFS, splitPath)
    ├── diskHandler (disk)        ← 文件 MD5(path) → path 字符串
    └── diskHandler (symlink)     ← 符号链接 + name_to_handle_at()
```

`cmd/serve/nfs/cache.go#L85-L135` 中 `pathRewriter` 保证同一文件经不同子路径 mount 获得**完全相同的句柄字节**（测试：`cmd/serve/nfs/handler_test.go#L135-L143`、`cmd/serve/nfs/cache_test.go#L176-L220`）。

**关键点**：`FromHandle` 总是返回 rootFS（ToHandle 时已归一化），因此后续所有 billy 调用都通过 rootFS 进行，rootFS 的 `fullPath()` 因 `root == ""` 而不做前缀拼接。

---

## 11. 属性注入的边界（setSys 与 II-5/II-6）

### 11.1 层间协议

```
VFS 层                         billy 适配层                        go-nfs 层
  │                                │                                  │
  │ vfs.Stat() 返回 vfs.Node       │                                  │
  │ (同时实现 os.FileInfo)         │                                  │
  │◄──────────────────────────────│                                  │
  │                                │ setSys(fi)                       │
  │                                │ ├─ node.VFS()         ⚡ II-5    │
  │                                │ ├─ vfs.Opt.UID/GID   ⚙ III-5/6  │
  │ node.SetSys(&file.FileInfo)   │ └─ node.Inode()      ⚡ II-6    │
  │◄──────────────────────────────│                                  │
  │                                │ 返回 fi 给 go-nfs                │
  │                                │─────────────────────────────────►│
  │                                │                          fi.Sys()│
  │                                │                     → *file.FileInfo│
  │                                │                     提取 uid/gid/fileid │
```

FUSE 前端（mount2）用 `cmd/mount2/fs.go#L69-L105` 的 `setAttr/setAttrOut` 直接写 `fuse.Attr`；NFS 前端用 `setSys` 注入 vfs.Node.Sys()。**VFS 层对两者完全无知**，是干净的关注点分离。

---

## 12. mount2 (FUSE) 与 NFS 的架构对比

| 维度 | mount2 (FUSE) | serve nfs |
|------|--------------|-----------|
| 协议 | FUSE (/dev/fuse) | NFSv3 (TCP) |
| 依赖库 | hanwen/go-fuse/v2 | willscott/go-nfs |
| 适配接口 | `fusefs.InodeEmbedder` | `billy.Filesystem` |
| 文件标识 | `Node` (嵌入 `fusefs.Inode`) | `[]byte` 句柄 (Cache 管理) |
| 文件句柄 | `FileHandle`（显式包装 `vfs.Handle`） | 每次 `FromHandle` → billy.Open → 拿新 Handle |
| **类别一（层直接调用 VFS）** | `FS.Root()` → `vfs.Root()` | I-1: `Mount()`→`h.vfs.Stat()`, I-2: `FSStat()`→`h.vfs.Statfs()`, I-3/4: Shutdown |
| **类别二（适配层特殊调用）** | 目录流 dirStream、属性合并 AttrOut | II-1/2: `MkdirAll` 循环, II-3/4: Chmod/Chown 先 Open, II-5/6: setSys 反查 |
| **类别三（仅读配置）** | AttrTimeout、UID、GID、MaxReadAhead | III-1~8: CacheMode、UID、GID、MetadataExtension、Fs().Root() |
| 子路径 | 不支持 | `Mount()` 支持 subFS |
| 平台 | Linux / macOS (amd64) | 所有 Unix |

---

## 13. 结论

### 三类访问的精确定位

> **类别一（4处）**：**不经过 billy**，发生在 `nfs.Handler` 接口回调（Mount/FSStat）或 VFS 生命周期管理中，由 NFS 服务层直接调用。
>
> **类别二（6处）**：**在 billy 适配层内部**，但不是简单的同名映射——MkdirAll 循环建目录、Chmod/Chown 先 Open 再操作、setSys 从 Node 反查 VFS/inode。
>
> **类别三（8处）**：**仅读取配置字段**，没有文件系统 I/O，分布在 Server 创建、billy Capabilities、Cache 构造、setSys 属性注入等位置。
>
> **常规 billy 映射（11处）**：ReadDir/Create/Open/Stat/Rename/Remove/Lstat/Symlink/Readlink/Chtimes 对应 VFS 同名方法，模式统一，属于 billy 适配层的正常职责。

### MkdirAll 定位结论（代码事实版）

MkdirAll 中 `f.vfs.Stat` + `f.vfs.Mkdir`（原来文档中的 A3/A4）**不在 NFS 层，而在 billy 适配层内部**（`FS.MkdirAll` 方法体，`cmd/serve/nfs/filesystem.go#L139-L157`）。它执行了 billy 层标准的 `fullPath()` 路径重写，调用者是 billy 接口方法的实现。因此正确分类为**"类别二：billy 适配层内部特殊调用"**，而非"NFS 层绕过 billy 直接访问"。

**代码事实澄清**：
- VFS.MkdirAll 的真实签名是 `func (vfs *VFS) MkdirAll(name string, perm os.FileMode) error`（`vfs/vfs.go#L790`），**接受 perm 参数**。之前版本"不接受 perm"的描述是错误的。
- 注释 "doesn't honor the permissions" 的真实含义是：**虽然 VFS 接口有 perm，但内部 `Dir.Mkdir(leaf)`（`vfs/dir.go#L1066`）签名无 perm，参数在传递过程中被静默丢弃**。
- 然而 NFS 手动循环调用的也是 `f.vfs.Mkdir(current, perm)`，**同样会走到 Dir.Mkdir，同样丢弃 perm**。注释给出的理由与代码行为是**自相矛盾**的。
- 无论循环的真实历史原因是什么，**层归属的判定只看调用位置和代码职责**，不看作者意图——调用点在 billy 方法内，经 fullPath 重写 → 类别二。
