# rclone VFS 抽象层代码理解

本文档对照源码讲解 rclone VFS 抽象层的三大核心机制：**打开文件**、**目录缓存**、**写回策略**，并说明三者如何协同支撑 FUSE 挂载访问。所有引用均使用仓库相对路径，形如 `vfs/file.go#L834-L929`，可在 IDE 中点击跳转。

---

## 一、整体架构

VFS 层位于挂载层（FUSE）与后端存储（`fs.Fs`）之间，是实现「云存储当本地磁盘」的关键抽象层。

### 1.1 调用链

```
cmd/mount (FUSE)
    │  cmd/mount/fs.go#L42-L49  FS.Root() → vfs.Root()
    ▼
vfs.VFS  (vfs/vfs.go#L178-L191)
    │  vfs/vfs.go#L562-L588  VFS.OpenFile()
    ▼
vfs.File.Open()  (vfs/file.go#L834-L929)  句柄选择
    ├──► ReadFileHandle   (vfs/read.go)            只读，分块流式
    ├──► WriteFileHandle  (vfs/write.go)           只写，io.Pipe 直传
    └──► RWFileHandle     (vfs/read_write.go)      读写，落本地缓存
                │
                └──► vfscache.Item        (vfs/vfscache/item.go)    本地磁盘缓存
                        └──► vfscache.writeback  (vfs/vfscache/writeback/writeback.go)  异步写回队列
    │
    ├──► vfs.Dir  (vfs/dir.go)            目录缓存 + 虚拟条目
    │
    └──► fs.Fs    后端对象存储接口
```

### 1.2 核心接口

`Node` 是文件/目录的统一抽象，定义在 `vfs/vfs.go#L58-L72`：

```go
type Node interface {
    os.FileInfo
    IsFile() bool
    Inode() uint64
    Open(flags int) (Handle, error)   // 打开文件/目录的统一入口
    SetModTime(modTime time.Time) error
    Sync() error
    Remove() error
    Path() string
    // ...
}
```

`Handle` 接口在 `vfs/vfs.go#L128-L137` 定义，聚合了 `*os.File` 的全部方法并增加 `Flush()`/`Release()` 等 FUSE 必需回调。`baseHandle`（`vfs/vfs.go#L139-L163`）为所有未实现方法返回 `ENOSYS`，使具体句柄只需实现自己支持的部分。

### 1.3 VFS 结构体

`VFS` 顶层结构体见 `vfs/vfs.go#L178-L191`，关键字段：

| 字段 | 含义 | 来源 |
| --- | --- | --- |
| `f fs.Fs` | 后端文件系统 | `vfs/vfs.go#L179` |
| `root *Dir` | 根目录节点 | `vfs/vfs.go#L181` |
| `Opt vfscommon.Options` | VFS 配置 | `vfs/vfs.go#L182` |
| `cache *vfscache.Cache` | 磁盘缓存（`CacheMode>Off` 时启用） | `vfs/vfs.go#L183` |
| `pollChan chan time.Duration` | 变更通知通道 | `vfs/vfs.go#L189` |
| `inUse atomic.Int32` | 打开计数，用于复用/回收 | `vfs/vfs.go#L190` |

`New()`（`vfs/vfs.go#L205-L284`）创建根目录、按 `CacheMode` 调用 `SetCacheMode()`（`vfs/vfs.go#L362-L378`）初始化磁盘缓存，并在后端支持 `ChangeNotify` 时注册回调（`vfs/vfs.go#L249-L255`）。

---

## 二、打开文件机制

打开文件是 VFS 的核心调度逻辑：根据 `flags` 推断读/写意图，再按 `CacheMode` 选择三种句柄之一。整个调度集中在 `File.Open()`。

### 2.1 入口：VFS.OpenFile()

`VFS.OpenFile()` 位于 `vfs/vfs.go#L562-L588`，负责路径解析与创建：

```go
func (vfs *VFS) OpenFile(name string, flags int, perm os.FileMode) (fd Handle, err error) {
    // O_RDONLY + O_TRUNC 未定义行为，rclone 返回 EINVAL
    if flags&accessModeMask == os.O_RDONLY && flags&os.O_TRUNC != 0 {
        return nil, EINVAL
    }
    node, err := vfs.Stat(name)            // vfs/vfs.go#L483-L508 逐级 Stat
    if err != nil {
        if err != ENOENT || flags&os.O_CREATE == 0 {
            return nil, err
        }
        // 不存在但带 O_CREATE → 先在父目录创建 File 节点
        dir, leaf, err := vfs.StatParent(name)
        node, err = dir.Create(leaf, flags) // vfs/dir.go#L1035-L1063
    }
    return node.Open(flags)                // 进入 File.Open()
}
```

`VFS.Open()`（`vfs/vfs.go#L593-L595`）与 `VFS.Create()`（`vfs/vfs.go#L601-L603`）都是它的薄包装。

### 2.2 句柄选择：File.Open()

完整调度逻辑在 `vfs/file.go#L834-L929`。

**阶段 1：解析 flags，确定读/写意图**（`vfs/file.go#L859-L889`）

```go
switch {
case rdwrMode == os.O_RDONLY:
    read = true
case rdwrMode == os.O_WRONLY:
    write = true
case rdwrMode == os.O_RDWR:
    read = true
    write = true
}
// O_APPEND 强制 read=true：追加写入前需要定位到文件末尾，需 RW 句柄
if flags&os.O_APPEND != 0 { read = true; f.appendMode = true }
// O_TRUNC / O_CREATE 强制 write=true
if flags&os.O_TRUNC != 0  { write = true }
if flags&os.O_CREATE != 0 { write = true }
```

设计要点：
- `O_APPEND` 升级为读写模式，因为追加前必须知道当前大小（见 `RWFileHandle._writeAt` 的 `vfs/read_write.go#L340-L346`）。
- `O_TRUNC`/`O_CREATE` 强制写权限，确保后续路径能创建/截断文件。

**阶段 2：句柄类型选择**（`vfs/file.go#L891-L922`）

```go
CacheMode := d.vfs.Opt.CacheMode
// 优先级 1：缓存中已有该文件（InUse 或 Exists）→ 强制 RW
if CacheMode >= vfscommon.CacheModeMinimal &&
    (d.vfs.cache.InUse(f.CachePath()) || d.vfs.cache.Exists(f.CachePath())) {
    fd, err = f.openRW(flags)
// 优先级 2：同时读 + 写
} else if read && write {
    if CacheMode >= vfscommon.CacheModeMinimal { fd, err = f.openRW(flags) }
    else                                       { fd, err = f.openWrite(flags) }
// 优先级 3：只写
} else if write {
    if CacheMode >= vfscommon.CacheModeWrites  { fd, err = f.openRW(flags) }
    else                                       { fd, err = f.openWrite(flags) }
// 优先级 4：只读
} else if read {
    if CacheMode >= vfscommon.CacheModeFull     { fd, err = f.openRW(flags) }
    else                                       { fd, err = f.openRead() }
}
```

`CacheMode` 四档定义于 `vfs/vfscommon/cachemode.go#L23-L28`：`Off / Minimal / Writes / Full`。决策矩阵：

| flags 意图 | CacheMode=Off | Minimal | Writes | Full |
| --- | --- | --- | --- | --- |
| 只读 | Read | Read | Read | RW |
| 只写 | Write | Write | RW | RW |
| 读写 | Write* | RW | RW | RW |
| 缓存中已存在 | 按上表 | RW(强制) | RW(强制) | RW(强制) |

\* `CacheMode=Off` 下读写会退化为 `WriteFileHandle`，读操作返回 `EPERM`（`vfs/write.go#L305-L317`）。

若 `flags` 含 `O_CREATE`，最后调用 `d.addObject(f)`（`vfs/file.go#L924-L927`）把新文件作为虚拟条目加入目录缓存（见第三章）。

### 2.3 ReadFileHandle：只读句柄

结构体定义于 `vfs/read.go#L20-L37`，构造函数 `newReadFileHandle()` 在 `vfs/read.go#L47-L69`。

```go
type ReadFileHandle struct {
    baseHandle
    r           *accounting.Account  // 底层 reader，带统计与限速
    offset      int64                // 当前读偏移
    size        int64                // 对象大小（0 表未知长度）
    cond        sync.Cond            // 顺序读等待条件变量
    hash        *hash.MultiHasher    // 哈希校验器
    noSeek      bool                 // 是否禁止 seek
    sizeUnknown bool                 // 源大小未知（流式场景）
}
```

**延迟打开（Lazy Open）**：`openPending()` 在首次 `Read()` 时才建立连接，见 `vfs/read.go#L73-L89`：

```go
func (fh *ReadFileHandle) openPending() (err error) {
    if fh.opened { return nil }
    o := fh.file.getObject()
    opt := &fh.file.VFS().Opt
    // 关键：chunkedreader 分块 + 并发流读取
    r, err := chunkedreader.New(fh.file.ctx, o,
        int64(opt.ChunkSize),       // 单 chunk 大小，默认 128M（vfs/vfscommon/options.go#L80）
        int64(opt.ChunkSizeLimit),  // chunk 上限
        opt.ChunkStreams,           // 并发流数
    ).Open()
    tr := accounting.GlobalStats().NewTransfer(o, nil)
    fh.r = tr.Account(fh.file.ctx, r).WithBuffer()  // 统计 + 预读缓冲
    fh.opened = true
    return nil
}
```

**顺序读优化**：`readAt()`（`vfs/read.go#L257-L346`）在偏移落在「顺序窗口」内时调用 `waitSequential()`（`vfs/read.go#L224-L254`）阻塞等待，避免并发 Range 请求互相抢占；超时（`ReadWait` 默认 20ms，`vfs/vfscommon/options.go#L125`）后再 seek。

**Seek 实现**：`seek()`（`vfs/read.go#L116-L168`）优先尝试缓冲丢弃（`SkipBytes`），失败则停止缓冲并用 `RangeSeek` 重新打开。

**哈希校验**：`checkHash()`（`vfs/read.go#L348-L370`）在读完整个对象后比对本地与远端哈希，不一致则报「corrupted on transfer」。

### 2.4 WriteFileHandle：只写句柄

结构体定义于 `vfs/write.go#L14-L29`，构造函数 `newWriteFileHandle()` 在 `vfs/write.go#L38-L51`。核心设计：**`io.Pipe` 直接流式上传，不落本地磁盘**。

**安全检查**：`safeToTruncate()`（`vfs/write.go#L54-L56`）确保只有 `O_TRUNC` 或新文件才能被覆写，否则 `openPending` 返回 `EPERM`（`vfs/write.go#L61-L87`）：

```go
func (fh *WriteFileHandle) openPending() (err error) {
    if fh.opened { return nil }
    if !fh.safeToTruncate() {
        return EPERM  // "Can't open for write without O_TRUNC ..."
    }
    var pipeReader *io.PipeReader
    pipeReader, fh.pipeWriter = io.Pipe()
    go func() {
        // operations.Rcat 处理分片、并发、统计等全部上传逻辑
        o, err := operations.Rcat(fh.file.ctx, fh.file.Fs(), fh.remote,
            pipeReader, time.Now(), nil)
        fh.o = o
        fh.result <- err  // 通过 channel 返回上传结果
    }()
    fh.file.setSize(0)
    fh.truncated = true
    fh.opened = true
    return nil
}
```

**WriteAt 约束**（`vfs/write.go#L128-L155`）：只能按 `offset` 递增顺序写；非顺序写先用 `waitSequential()`（`WriteWait` 默认 1000ms，`vfs/vfscommon/options.go#L120`）等待，仍不匹配则返回 `ESPIPE`。

**Close 流程**（`vfs/write.go#L187-L213`）：关闭 `pipeWriter` → 阻塞等待 `result` channel（即上传完成）→ `setObject` 更新目录中的对象引用。若上传失败且对象为 nil，调用 `File.Remove()` 清理虚拟条目。

### 2.5 RWFileHandle：读写句柄

结构体定义于 `vfs/read_write.go#L18-L31`，是唯一支持完整随机读写的句柄。所有读写都落到 `vfscache.Item` 代表的本地磁盘文件。

**构造时与缓存层对接**（`vfs/read_write.go#L43-L77`）：

```go
func newRWFileHandle(d *Dir, f *File, flags int) (fh *RWFileHandle, err error) {
    item := d.vfs.cache.Item(f.CachePath())          // 获取/创建缓存项
    exists := f.exists() || (item.Exists() && !item.WrittenBack())
    if flags&(os.O_CREATE|os.O_EXCL) == os.O_CREATE|os.O_EXCL && exists {
        return nil, EEXIST                            // O_CREATE|O_EXCL 但已存在
    }
    fh = &RWFileHandle{file: f, d: d, flags: flags, item: item}
    // O_TRUNC 或 O_CREATE+不存在 → 立即截断并标记脏（确保空文件也能被上传）
    if !fh.readOnly() && (fh.flags&os.O_TRUNC != 0 || (fh.flags&os.O_CREATE != 0 && !exists)) {
        err = fh.Truncate(0)
        item.Dirty()                                  // vfs/vfscache/item.go#L450-L456
    }
    if !fh.readOnly() {
        fh.file.addWriter(fh)                         // vfs/file.go#L330-L335
    }
    return fh, nil
}
```

**延迟打开**（`vfs/read_write.go#L92-L117`）：`openPending()` 持有 `file.muRW` 后调用 `item.Open(o)`，可能触发从后端下载到本地缓存（见第四章）。

**写入流程**：`_writeAt()`（`vfs/read_write.go#L329-L362`）→ `item.WriteAt()`（`vfs/vfscache/item.go#L1378-L1411`）→ 写本地磁盘 → `item._dirty()` 标记脏。

**关闭流程**：`close()`（`vfs/read_write.go#L156-L180`）→ `item.Close(setObject)`，由 Item 内部决定同步/异步写回（见第四章）。

### 2.6 三种句柄对比

| 特性 | ReadFileHandle | WriteFileHandle | RWFileHandle |
| --- | --- | --- | --- |
| 数据位置 | 内存缓冲 + 后端流 | `io.Pipe` 直传 | 本地磁盘文件 |
| 随机读 | 支持（Range） | 不支持（`EPERM`） | 支持 |
| 随机写 | 不支持 | 顺序写 | 完全支持 |
| Truncate | 不适用 | 仅打开时 | 任意时刻 |
| 打开延迟 | 低 | 低 | 可能高（需下载） |
| 内存占用 | `ChunkSize×流数` | 上传缓冲 | 低（仅 fd） |
| 适用场景 | 大文件顺序读、流媒体 | 一次性写新文件 | 数据库、随机修改 |

---

## 三、目录缓存实现

目录缓存的目标：**减少远程 List API 调用** + **在刷新前列表前保持本地修改可见**。实现集中在 `vfs.Dir`。

### 3.1 数据结构

`Dir` 结构体定义于 `vfs/dir.go#L26-L45`：

```go
type Dir struct {
    vfs          *VFS
    f            fs.Fs
    cleanupTimer *time.Timer   // 缓存过期清理定时器
    mu      sync.RWMutex
    parent  *Dir
    path    string
    read    time.Time         // 上次从后端读取时间
    items   map[string]Node   // ★ 目录条目缓存（核心）
    virtual map[string]vState // ★ 虚拟条目状态表
    _virtuals atomic.Int32    // 本目录+子目录虚拟条目数
}
```

**虚拟条目状态**（`vfs/dir.go#L49-L57`）：

```go
type vState byte
const (
    vOK      vState = iota // 正常条目（来自后端）
    vAddFile               // 本地新增的文件
    vAddDir                // 本地新增的目录
    vDel                   // 本地删除的条目
)
```

设计意图：`virtual` 表解决「本地 `mkdir`/`create`/`delete` 后，下一次 List 结果尚未合并」的一致性问题——本地修改先以虚拟状态记下，等下次 List 合并时再升级或清除。

### 3.2 缓存有效性与过期

**年龄判断**：`_age()`（`vfs/dir.go#L360-L367`）返回距上次读取的时长及是否过期（超过 `DirCacheTime`，默认 5 分钟，`vfs/vfscommon/options.go#L30`）。

**定时清理**：`newDir()`（`vfs/dir.go#L59-L75`）创建 `cleanupTimer`，到期触发 `cacheCleanup()`（`vfs/dir.go#L77-L92`）；若过期则 `ForgetAll()`（`vfs/dir.go#L224-L258`）。

**读取触发**：`_readDir()`（`vfs/dir.go#L532-L587`）在 `_age()` 判定过期时才调用 `list.DirSorted()` 拉取后端列表，否则直接返回缓存。读取成功后重置 `d.read` 与 `cleanupTimer`（`vfs/dir.go#L583-L584`）。

**变更通知**：启动逻辑位于 `vfs/vfs.go#L247-L255`，严格区分后端是否支持 `ChangeNotify` 特性：

- **后端支持 `ChangeNotify`**：创建 `chan time.Duration` 作为 `pollChan`，将回调函数 `vfs.root.changeNotify` 与该 channel 一同传入后端的 `do(ctx, callback, pollChan)`，随后向 channel 写入初始间隔 `time.Duration(vfs.Opt.PollInterval)`（默认 1 分钟）。间隔的实际使用由各后端 `ChangeNotify` 实现自行决定（如监听该 channel 的动态调整）。回调触发时走 `Dir.changeNotify()`（`vfs/dir.go#L290-L299`）→ `invalidateDir()`（`vfs/dir.go#L274-L284`），将目标目录（及目录条目自身）的 `read` 字段置为零时间，强制下次 `_readDir()` 重新从后端拉取。

- **后端不支持 `ChangeNotify`**：若用户仍配置了 `PollInterval > 0`，仅打印一条 info 日志 `poll-interval is not supported by this remote`，**不做任何轮询或缓存失效动作**，目录缓存仍按 `DirCacheTime`（默认 5 分钟）自然过期。

### 3.3 目录读取与合并：_readDirFromEntries()

将后端 `fs.DirEntries` 合并到缓存的核心算法在 `vfs/dir.go#L732-L787`：

```go
func (d *Dir) _readDirFromEntries(entries fs.DirEntries, dirTree dirtree.DirTree, when time.Time) error {
    mv := d._newManageVirtuals()                 // 先清理过期虚拟条目
    for _, entry := range entries {
        name := path.Base(entry.Remote())
        node := d.items[name]
        if mv.add(d, name) { continue }         // vDel 条目从列表中跳过
        switch item := entry.(type) {
        case fs.Object:
            // 尽可能复用已有 *File，保留 inode 不变
            if file, ok := node.(*File); node != nil && ok {
                file.setObjectNoUpdate(item)     // vfs/file.go#L558-L565
            } else {
                node = newFile(d, d.path, item, name)
            }
        case fs.Directory:
            if node == nil || !node.IsDir() {
                node = newDir(d.vfs, d.f, d, item)
            }
            // 递归更新子目录...
        }
        d.items[name] = node
    }
    mv.end(d)                                    // 收尾清理
    return nil
}
```

### 3.4 虚拟条目合并：manageVirtuals

`manageVirtuals` 是 `_readDirFromEntries` 生命周期内的辅助结构，定义于 `vfs/dir.go#L661-L728`。

**`mv.add()` — 逐条处理后端条目**（`vfs/dir.go#L681-L696`）：

```go
func (mv manageVirtuals) add(d *Dir, name string) bool {
    mv[name] = struct{}{}                        // 记录该名出现在后端列表
    switch d.virtual[name] {
    case vAddFile, vAddDir:
        d._deleteVirtual(name)                  // 后端已确认 → 升级为真实条目
    case vDel:
        return true                             // 本地已删除 → 跳过
    }
    return false
}
```

**`mv.end()` — 处理后端未列出的旧条目**（`vfs/dir.go#L702-L728`）：

```go
func (mv manageVirtuals) end(d *Dir) {
    // Part A：清理 d.items 中在后端消失的条目
    for name := range d.items {
        if _, ok := mv[name]; !ok {
            switch d.virtual[name] {
            case vAddFile, vAddDir:
                // 虚拟新增项后端还没看到 → 保留
            default:
                delete(d.items, name)           // 后端删除 → 同步删除
            }
        }
    }
    // Part B：清理已确认的 vDel 虚拟标记
    for name, virtualState := range d.virtual {
        if _, ok := mv[name]; !ok {
            if virtualState == vDel {
                d._deleteVirtual(name)
            }
        }
    }
}
```

**`_purgeVirtual()`**（`vfs/dir.go#L620-L659`）在每次读取前清理可清除的虚拟条目：`vAddDir` 在后端支持空目录时清除；`vAddFile` 在上传完成且未使用时清除；正在写入/缓存中的保留。

### 3.5 本地修改如何写入虚拟表

- **新增文件**：`Dir.addObject()`（`vfs/dir.go#L445-L462`）将节点加入 `items` 并标记 `vAddFile`/`vAddDir`，同时 `addVirtual(1)`（`vfs/dir.go#L210-L215`）累加父链计数。
- **缓存层注入**：`VFS.AddVirtual()`（`vfs/vfs.go#L846-L862`）→ `Dir.AddVirtual()`（`vfs/dir.go#L471-L500`），让正在上传的文件在目录中可见。
- **删除条目**：`Dir.delObject()`（`vfs/dir.go#L506-L518`）标记 `vDel`，直到下次 List 确认远端确无此项。

这样，挂载侧 `readdir` 看到的永远是「后端列表 ∪ 本地修改 − 本地删除」的合并视图。

---

## 四、写回策略实现

写回策略由 `vfscache` 包实现：脏数据标记 → 延迟写回队列 → 异步上传 → 缓存清理。它支撑了「关闭即返回、后台同步」的体验。

### 4.1 缓存项结构

`Item` 定义于 `vfs/vfscache/item.go#L56-L72`：

```go
type Item struct {
    c               *Cache
    mu              sync.Mutex
    cond            sync.Cond              // 与 cache cleaner 同步
    name            string                 // VFS 中的名字
    opens           int                    // 打开计数
    downloaders     *downloaders.Downloaders
    o               fs.Object             // 远端对象，可能为 nil
    fd              *os.File               // 本地缓存文件句柄
    info            Info                   // 持久化元数据
    writeBackID     writeback.Handle       // 写回队列中的 ID
    pendingAccesses int                    // 正在访问的线程数（保护 reset）
    modified        bool                   // 自上次 Open 以来是否修改
    graceTimer      *time.Timer            // 延迟关闭宽限定时器
}
```

`Info`（`vfs/vfscache/item.go#L75-L82`）持久化到磁盘，含 `ModTime`/`ATime`/`Size`/`Rs`（已缓存区间）/`Fingerprint`/`Dirty`。锁顺序约定见文件头注释（`vfs/vfscache/item.go#L22-L52`）：`Cache.mu` → `Item.mu`，`downloaders.mu`/`writeback.mu` 在 `Item.mu` 之前。

### 4.2 脏数据标记：_dirty()

`_dirty()`（`vfs/vfscache/item.go#L431-L447`）是写回的起点：

```go
func (item *Item) _dirty() {
    item.info.ModTime = time.Now()
    item.info.ATime = item.info.ModTime
    if !item.modified {
        item.modified = true
        item.mu.Unlock()
        item.c.writeback.Remove(item.writeBackID)  // 重新计时：先取消旧写回
        item.mu.Lock()
    }
    if !item.info.Dirty {
        item.info.Dirty = true
        err := item._save()                        // 立即持久化元数据，支持断点续传
        if err != nil { /* ... */ }
    }
}
```

每次 `Item.WriteAt()`（`vfs/vfscache/item.go#L1378-L1411`）写入成功后调用 `_dirty()`。注意 `_dirty()` 会先 `Remove` 旧写回任务再由后续 `Close` 重新入队，实现「每次修改都重置延迟计时」。

### 4.3 打开缓存文件：Item.Open()

`Open()`（`vfs/vfscache/item.go#L498-L514`）带低层重试与 ENOSPC 恢复，核心在 `open()`（`vfs/vfscache/item.go#L518-L602`）：

1. `createItemDir()` 建目录，`_checkObject(o)`（`vfs/vfscache/item.go#L862-L905`）比对远端指纹，远端变化且本地非脏则丢弃缓存。
2. `opens++`，引用计数；首次打开时 `_createFile()`（`vfs/vfscache/item.go#L467-L494`）创建/打开本地文件并 `_save()` 元数据。
3. `c.put()` 把 Item 放回 Cache map（防止被清理误删）。
4. 若 `item.o != nil`，创建 `downloaders` 负责按需下载缺失区间。

宽限机制：`opens==0` 且非脏时，`Close()`（`vfs/vfscache/item.go#L674-L700`）启动 `graceTimer`（`HandleCaching` 默认 5s，`vfs/vfscommon/options.go#L170`），到期由 `closeAfterGrace()`（`vfs/vfscache/item.go#L706-L721`）真正关闭；期间重新打开可直接复用 fd（`vfs/vfscache/item.go#L540-L564`）。

### 4.4 关闭与写回触发：_actualClose()

`_actualClose()`（`vfs/vfscache/item.go#L726-L814`）是写回的核心调度：

```go
func (item *Item) _actualClose(storeFn StoreFn, syncWriteBack bool) (err error) {
    _, _ = item._getSize()
    // 脏文件关闭前补齐未下载区间（确保上传的是完整文件）
    if item.info.Dirty && item.o != nil {
        err = item._ensure(0, item.info.Size)     // vfs/vfscache/item.go#L1207-L1247
    }
    // 关闭 downloaders 与 fd，保存元数据
    // ...
    if item.info.Dirty {
        if syncWriteBack {
            item._store(item.c.ctx, storeFn)      // 同步上传
        } else {
            // 异步：加入写回队列
            item.c.writeback.SetID(&item.writeBackID)
            id := item.writeBackID
            item.c.writeback.Add(id, item.name, item.info.Size, item.modified,
                func(ctx context.Context) error {
                    return item.store(ctx, storeFn)  // vfs/vfscache/item.go#L667-L671
                })
        }
    }
    item.modified = false
    return err
}
```

同步/异步由 `syncWriteBack := item.c.opt.WriteBack <= 0` 决定（`vfs/vfscache/item.go#L678`）。默认 `WriteBack=5s`（`vfs/vfscommon/options.go#L130`）即异步写回。

### 4.5 上传执行：Item._store()

`_store()`（`vfs/vfscache/item.go#L617-L663`）：

```go
func (item *Item) _store(ctx context.Context, storeFn StoreFn) (err error) {
    cacheObj, err := item.c.fcache.NewObject(ctx, item.name)  // 取本地缓存文件对象
    if cacheObj != nil {
        o, name := item.o, item.name
        unlockMutexForCall(&item.mu, func() {
            o, err = operations.Copy(ctx, item.c.fremote, o, name, cacheObj)  // 上传到远端
        })
        item.o = o
        item._updateFingerprint()                              // 更新指纹
    }
    // 写回 VFS 层（更新目录条目的对象引用），必须在标记 clean 之前
    if storeFn != nil && item.o != nil {
        o := item.o
        item.mu.Unlock()
        storeFn(o)                                              // 即 File.setObject
        item.mu.Lock()
    }
    item.info.Dirty = false                                    // 标记已干净，可被清理
    err = item._save()
    return nil
}
```

### 4.6 写回队列：writeback.WriteBack

`WriteBack` 定义于 `vfs/vfscache/writeback/writeback.go#L29-L41`，用最小堆（`writeBackItems`，`vfs/vfscache/writeback/writeback.go#L81-L115`）按 `expiry` 排序，`Less()` 见 `vfs/vfscache/writeback/writeback.go#L85-L92`。

**入队 / 重置计时**：`Add()`（`vfs/vfscache/writeback/writeback.go#L259-L278`）：

```go
func (wb *WriteBack) Add(id Handle, name string, size int64, modified bool, putFn PutFn) Handle {
    wbItem, ok := wb.lookup[id]
    if !ok {
        wbItem = wb._newItem(id, name, size)        // 新建，expiry = now + WriteBack
    } else {
        if wbItem.uploading && modified {
            wb._cancelUpload(wbItem)                 // 正在上传则取消，稍后重试
        }
        wb.items._update(wbItem, wb._newExpiry())    // ★ 每次修改都重置延迟
    }
    wbItem.putFn = putFn
    wb._resetTimer()                                 // 重排定时器
    return wbItem.id
}
```

**到期处理**：定时器到期触发 `processItems()`（`vfs/vfscache/writeback/writeback.go#L427-L459`），弹出所有已过期项，受 `--transfers` 并发上限约束，逐个起 goroutine 调用 `upload()`。

**上传与重试**：`upload()`（`vfs/vfscache/writeback/writeback.go#L346-L386`）：

```go
func (wb *WriteBack) upload(ctx context.Context, wbItem *writeBackItem) {
    putFn := wbItem.putFn
    wbItem.tries++
    wb.mu.Unlock()
    err := putFn(ctx)                       // 执行 item.store()
    wb.mu.Lock()
    wbItem.uploading = false
    wb.uploads--
    if err != nil {
        wbItem.delay *= 2                   // 指数退避
        if wbItem.delay > maxUploadDelay {  // maxUploadDelay = 5min（writeback.go#L19）
            wbItem.delay = maxUploadDelay
        }
        wb._pushItem(wbItem)                // 重新入队等待重试
        wb.items._update(wbItem, time.Now().Add(wbItem.delay))
    } else {
        wb._delItem(wbItem)                 // 成功 → 移出 lookup
    }
    wb._resetTimer()
    close(wbItem.done)
}
```

**取消与重命名**：`Remove()`/`_remove()`（`vfs/vfscache/writeback/writeback.go#L285-L310`）取消进行中的上传；`Rename()`（`vfs/vfscache/writeback/writeback.go#L315-L341`）更新名字并重排，同时移除同名旧任务。

### 4.7 缓存清理：Cache.cleaner()

后台清理 goroutine 在 `Cache.New()`（`vfs/vfscache/cache.go#L80-L151`）启动，循环体 `cleaner()`（`vfs/vfscache/cache.go#L846-L867`）：按 `CachePollInterval`（默认 60s，`vfs/vfscommon/options.go#L60`）定时，或被 `KickCleaner()`（`vfs/vfscache/cache.go#L566-L590`）在 ENOSPC 时唤醒。

`clean()`（`vfs/vfscache/cache.go#L791-L841`）依次执行：

1. **按年龄清理** `purgeOld()`（`vfs/vfscache/cache.go#L680-L691`）：调用 `Item.RemoveNotInUse()`（`vfs/vfscache/item.go#L973-L1007`），删除 `ATime` 早于 `CacheMaxAge`（默认 1 小时，`vfs/vfscommon/options.go#L65`）且未打开/非脏的项。
2. **按配额清理** `purgeOverQuota()`（`vfs/vfscache/cache.go#L759-L788`）：按 `ATime` 排序，从最旧起删除未使用项直到满足 `CacheMaxSize`/`CacheMinFreeSpace`（`vfs/vfscache/cache.go#L720-L755`）。
3. **紧急清理** `purgeClean()`（`vfs/vfscache/cache.go#L633-L677`）：仍超配额时对非脏项调用 `Item.Reset()`（`vfs/vfscache/item.go#L1011-L1141`）清空内容释放空间，但保留 fd 与元数据以便后续重建。
4. **失败重试** `retryFailedResets()`（`vfs/vfscache/cache.go#L609-L630`）重做之前因 ENOSPC 失败的 reset。

`Reset()` 通过 `preAccess()`/`postAccess()`（`vfs/vfscache/item.go#L1146-L1168`）与 IO 线程互斥：reset 期间 `beingReset=true`，IO 线程在 `cond.Wait()` 等待；reset 时若 `pendingAccesses>0` 则跳过避免死锁（`vfs/vfscache/item.go#L1042-L1044`）。

### 4.8 启动时恢复：reload()

`Cache.reload()`（`vfs/vfscache/cache.go#L543-L563`）扫描磁盘缓存与元数据，对每个 dirty 项调用 `Item.reload()`（`vfs/vfscache/item.go#L822-L851`）：重新打开、触发写回、`AddVirtual` 注入目录树。这保证了进程崩溃重启后未上传的修改不会丢失。

---

## 五、三者如何协同支撑挂载访问

### 5.1 挂载层与 VFS 层的交互

FUSE 层 `cmd/mount/fs.go` 极薄：`FS.Root()`（`cmd/mount/fs.go#L42-L49`）返回 `vfs.Root()`；`Statfs`（`cmd/mount/fs.go#L56-L72`）转发 `vfs.Statfs()`；文件/目录操作全部委托给 VFS 的 `Node`/`Handle`。错误经 `translateError()`（`cmd/mount/fs.go#L75-L107`）映射为 POSIX errno。

因此挂载的「读、写、列目录」语义完全由 VFS 三大机制实现。

### 5.2 典型场景：只读访问（如 `cat`）

1. FUSE `open(O_RDONLY)` → `VFS.OpenFile()`（`vfs/vfs.go#L562-L588`）→ `File.Open()`（`vfs/file.go#L834-L929`）。
2. `CacheMode<Full` 时选 `ReadFileHandle`，`openPending()` 用 `chunkedreader` 按需分块拉取（`vfs/read.go#L73-L89`），不占本地磁盘。
3. `readahead` 由 `accounting` 缓冲提供，`ReadAt` 命中顺序窗口时等待（`vfs/read.go#L269-L271`）。
4. 关闭时 `checkHash()` 校验完整性（`vfs/read.go#L348-L370`）。
5. 目录列表走 `Dir.ReadDirAll()`（`vfs/dir.go#L1003-L1019`），命中缓存则不调远端 List。

### 5.3 典型场景：写入新文件（如 `cp`）

1. `open(O_WRONLY|O_CREATE)` → 若 `CacheMode>=Writes` 选 `RWFileHandle`（`vfs/file.go#L907-L912`）。
2. `newRWFileHandle` 立即 `Truncate(0)`+`Dirty()`（`vfs/read_write.go#L62-L70`），并 `addObject` 加入目录虚拟条目（`vfs/dir.go#L445-L462`）——文件立即可见。
3. 写入走 `item.WriteAt()` → 本地磁盘 → `_dirty()` 标脏并 `_save()` 元数据（`vfs/vfscache/item.go#L1378-L1411`、`L431-L447`）。
4. 关闭 `item.Close()` → `_actualClose()` 因 `WriteBack>0` 走异步：`writeback.Add()` 入队，5s 后到期上传（`vfs/vfscache/item.go#L798-L807`、`vfs/vfscache/writeback/writeback.go#L427-L459`）。
5. 上传成功后目录条目更新按句柄类型分两条路径，最终殊途同归：
   - **RW 句柄（磁盘缓存写回）**：`Item._store()`（`vfs/vfscache/item.go#L1331-L1376`）上传完成后调用传入的 `storeFn(o)`，该函数即 `File.setObject`（`vfs/file.go#L544-L554`）。
   - **Write 句柄（Pipe 直传）**：`WriteFileHandle.close()`（`vfs/write.go#L187-L213`）等待 `<-fh.result` 返回成功后，同样执行 `fh.file.setObject(fh.o)`。
   
   `File.setObject` 的执行链为：更新 `f.o` → 释放 `File.mu` → 调用 `d.addObject(f)`（`vfs/dir.go#L445-L462`）。`addObject` 将节点写入 `d.items[leaf] = f`，并标记 `d.virtual[leaf] = vAddFile`（首次出现时累加父链 `_virtuals` 计数）。此时用户侧 `readdir` 立即可见该文件（因为 `items` 已更新）。
   
   该 `vAddFile` 虚拟标记的最终清理发生在**下一次目录从后端读取时**：
   - 若后端列表已包含同名文件：`_readDirFromEntries()`（`vfs/dir.go#L732-L787`）→ `mv.add()`（`vfs/dir.go#L681-L696`）命中 `case vAddFile`，调用 `_deleteVirtual(name)`（`vfs/dir.go#L596-L607`）清除虚拟标记，条目升级为「真实条目」。
   - 若后端列表尚未出现该文件：由每次 List 前的 `_purgeVirtual()`（`vfs/dir.go#L620-L659`）负责清理——仅当写入已完成（`!f.writingInProgress()`）且未被缓存使用时才删除虚拟标记；否则保留。
6. 挂载层 `flush` 立即返回，用户感知「秒存」；后台 `WaitForWriters()`（`vfs/vfs.go#L434-L464`）在卸载前等待所有写回完成。

### 5.4 典型场景：随机读写（如 SQLite）

1. `open(O_RDWR)` → `CacheMode>=Minimal` 选 `RWFileHandle`（`vfs/file.go#L898-L900`）。
2. 首次读 `item._ensure()`（`vfs/vfscache/item.go#L1207-L1247`）按需下载缺失区间到本地文件，后续读写完全在本地磁盘进行。
3. `Seek`/`WriteAt`/`Truncate` 全部由本地 fd 支持（`vfs/read_write.go#L301-L322`、`L329-L362`、`L401-L411`）。
4. 关闭触发写回；若 `WriteBack<=0` 则同步上传（`vfs/vfscache/item.go#L678`、`L795-L797`），保证一致性敏感场景的数据安全。
5. 缓存清理不会触碰 dirty/in-use 项（`vfs/vfscache/item.go#L980`、`L1027-L1029`），避免误删活跃文件。

### 5.5 协同关系总览

```
打开文件（File.Open）        目录缓存（Dir）              写回策略（vfscache）
   │ 选择最优句柄              │ 维持合并视图                 │ 脏标记 + 延迟上传
   │                           │                             │
   ├── Read: 流式不落盘 ───────┤ 读路径命中缓存，少调 List ──┤ 关闭后异步回写
   ├── Write: Pipe 直传 ──────┤ 本地修改即时可见(virtual) ──┤ 上传成功 setObject→addObject→vAddFile
   └── RW: 本地缓存文件 ──────┤ 写时 addObject 注入目录 ────┤ reload 恢复未传数据
                                                              │
                          配额/年龄清理保护本地磁盘不爆 ───────┘
```

三者通过 `CachePath`、虚拟条目、`storeFn` 回调紧密耦合：打开文件决定数据落点，目录缓存保证可见性与一致性，写回策略保证持久性与磁盘可控。写回完成通过 `File.setObject → Dir.addObject` 将条目注入目录并标记 `vAddFile`，待下次后端 List 时由 `mv.add()` 升级为真实条目。

---

## 六、关键配置参数汇总

定义于 `vfs/vfscommon/options.go#L185-L219`，默认值见 `OptionsInfo`（`vfs/vfscommon/options.go#L13-L178`）。

| 参数 | 字段 | 默认值 | 作用 | 关联代码 |
| --- | --- | --- | --- | --- |
| `--vfs-cache-mode` | `CacheMode` | `off` | 句柄选择与是否启用磁盘缓存 | `vfs/file.go#L891-L922` |
| `--dir-cache-time` | `DirCacheTime` | `5m` | 目录列表缓存有效期 | `vfs/dir.go#L360-L367` |
| `--poll-interval` | `PollInterval` | `1m` | 后端支持 ChangeNotify 时通过 channel 传递的轮询间隔；不支持时仅警告不生效 | `vfs/vfs.go#L247-L255` |
| `--vfs-cache-max-age` | `CacheMaxAge` | `1h` | 缓存项最大留存时间 | `vfs/vfscache/cache.go#L803` |
| `--vfs-cache-max-size` | `CacheMaxSize` | `-1`(无限) | 缓存总大小上限 | `vfs/vfscache/cache.go#L738-L743` |
| `--vfs-cache-min-free-space` | `CacheMinFreeSpace` | `-1`(不限制) | 目标最小剩余空间 | `vfs/vfscache/cache.go#L720-L733` |
| `--vfs-cache-poll-interval` | `CachePollInterval` | `60s` | 清理器轮询间隔 | `vfs/vfscache/cache.go#L846-L867` |
| `--vfs-read-chunk-size` | `ChunkSize` | `128M` | 只读分块大小 | `vfs/read.go#L79` |
| `--vfs-read-chunk-streams` | `ChunkStreams` | `0` | 并发读流数 | `vfs/read.go#L79` |
| `--vfs-write-back` | `WriteBack` | `5s` | 关闭后延迟写回时间；`<=0` 同步 | `vfs/vfscache/item.go#L678`、`writeback.go#L128-L135` |
| `--vfs-write-wait` | `WriteWait` | `1000ms` | 顺序写等待窗口 | `vfs/write.go#L135` |
| `--vfs-read-wait` | `ReadWait` | `20ms` | 顺序读等待窗口 | `vfs/read.go#L270` |
| `--vfs-handle-caching` | `HandleCaching` | `5s` | 关闭后 fd 宽限期 | `vfs/vfscache/item.go#L693-L696` |
| `--vfs-read-ahead` | `ReadAhead` | `0` | full 模式额外预读 | `vfs/vfscommon/options.go#L134-L137` |
| `--read-only` | `ReadOnly` | `false` | 只读挂载 | `vfs/file.go#L625`、`vfs/dir.go#L947` |

---

## 七、总结

rclone VFS 通过三层机制协同把「云存储」伪装成「本地磁盘」：

1. **打开文件**（`File.Open`，`vfs/file.go#L834-L929`）以 `flags`+`CacheMode` 决策出 `Read`/`Write`/`RW` 三种句柄，在「零磁盘 IO 流式」「直传上传」「本地缓存随机读写」之间权衡，兼顾性能与功能。
2. **目录缓存**（`Dir`，`vfs/dir.go`）以 `items`+`virtual` 维护「后端列表 ∪ 本地修改 − 本地删除」的合并视图，配合 `DirCacheTime`/`ChangeNotify` 减少远端 List 调用并保证本地修改即时可见。
3. **写回策略**（`vfscache`，`vfs/vfscache/item.go`、`writeback.go`、`cache.go`）以脏标记+延迟队列实现「关闭即返回、后台异步上传」，用元数据持久化支持断点续传，用配额/年龄/紧急三级清理保护本地磁盘。

挂载层（`cmd/mount/fs.go`）仅做 FUSE 适配与错误映射，真正的文件语义全部由 VFS 三大机制承载，从而让任意支持 FUSE 的应用都能像操作本地文件一样操作云存储。
