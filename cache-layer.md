# rclone Cache 缓存层分析

## 一、缓存架构总览

rclone 中存在 **三层缓存体系**，自底向上逐层封装：

| 层级 | 代码位置 | 职责 |
|------|---------|------|
| L1 通用缓存 | [lib/cache/cache.go](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/lib/cache/cache.go) | 基于 TTL 的通用 KV 缓存，带 Pin/Unpin 引用计数 |
| L2 Fs 实例缓存 | [fs/cache/cache.go](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/fs/cache/cache.go) | 缓存 `fs.Fs` 后端实例，避免重复初始化，内置 L1 |
| L3 Backend Cache | [backend/cache/](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/) | **本文重点**：独立缓存后端，包装任意 remote，提供元数据缓存 + 数据块(chunk)缓存 + 写缓冲 |

本文聚焦 **L3 Backend Cache**，即 `backend/cache` 包。

---

## 二、核心数据结构

### 2.1 Fs：缓存后端主体

[cache.go#L319-L340](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L319-L340)

```go
type Fs struct {
    fs.Fs                           // 嵌入底层 remote，实现透传
    wrapper   fs.Fs                 // 上层包装者（如 crypt）
    name      string
    root      string
    opt       Options               // 配置参数
    features  *fs.Features
    cache     *Persistent           // 持久化存储（Bolt DB + 文件系统）
    tempFs    fs.Fs                 // 临时写目录（可选）
    rateLimiter   *rate.Limiter     // RPS 限速
    plexConnector *plexConnector    // Plex 集成
    backgroundRunner *backgroundWriter // 后台上传协程
    cleanupChan chan bool
    parentsForgetFn []func(string, fs.EntryType) // ChangeNotify 订阅者
}
```

关键设计：`Fs` 嵌入 `fs.Fs`（底层 remote），对外部表现为标准 `fs.Fs` 接口，内部拦截所有操作先查缓存，miss 时透传到底层。

### 2.2 Object：文件元数据缓存

[object.go#L24-L40](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L24-L40)

```go
type Object struct {
    fs.Object                       // 嵌入底层 fs.Object
    ParentFs      fs.Fs             // 实际所在 FS（可能是 tempFs 或 remote）
    CacheFs       *Fs               // 所属缓存 FS
    Name          string
    Dir           string
    CacheModTime  int64
    CacheSize     int64
    CacheStorable bool
    CacheType     string            // "Object" | "TempObject"
    CacheTs       time.Time         // 缓存时间戳，用于 TTL 判定
    CacheHashes   map[hash.Type]string // 哈希值懒缓存
}
```

### 2.3 Directory：目录元数据缓存

[directory.go#L14-L26](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/directory.go#L14-L26)

```go
type Directory struct {
    Directory fs.Directory
    CacheFs      *Fs
    Name         string
    Dir          string
    CacheModTime int64
    CacheSize    int64
    CacheItems   int64
    CacheType    string              // "Directory"
    CacheTs      *time.Time          // 缓存时间戳
}
```

### 2.4 Handle：文件句柄 + 预加载工作池

[handle.go#L43-L60](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L43-L60)

```go
type Handle struct {
    ctx          context.Context
    cachedObject *Object
    cfs          *Fs
    memory       *Memory             // 内存中转缓存
    preloadQueue chan int64          // 待预加载的 chunk offset 队列
    offset       int64               // 当前读偏移
    seenOffsets  map[int64]bool      // 已提交的 chunk
    workers      int                 // 工作协程数
    UseMemory    bool
}
```

### 2.5 存储层双实现

| 实现 | 文件 | 存储介质 | 用途 |
|------|------|---------|------|
| `Memory` | [storage_memory.go](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_memory.go) | RAM (go-cache) | 流式读取时的短期中转，读完即丢 |
| `Persistent` | [storage_persistent.go](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go) | Bolt DB + 本地文件系统 | 元数据持久化 + chunk 持久化 |

---

## 三、元数据缓存机制

### 3.1 存储结构（Bolt DB）

[storage_persistent.go#L143-L168](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L143-L168)

Bolt DB 中按路径分层建立 Bucket，镜像远端目录树：

```
RootBucket/
├── . (目录元数据 JSON)
├── subdir1/
│   ├── . (subdir1 元数据)
│   ├── fileA (Object JSON)
│   └── fileB (Object JSON)
└── fileRoot (Object JSON)
```

- 目录：用 **子 Bucket** 表示，Bucket 内的 `"."` key 存 `Directory` JSON
- 文件：用 **KV** 表示，value 是 `Object` JSON

此外还有 3 个顶级辅助 Bucket：
- `RootTsBucket`：文件时间戳追踪
- `DataTsBucket`：chunk 时间戳 + 大小追踪，用于 LRU 清理
- `tempBucket`：待后台上传的队列

### 3.2 缓存命中判定

所有元数据缓存都遵循统一的 TTL 策略，核心是 `CacheTs + InfoAge` 判定：

[object.go#L158-L166](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L158-L166)

```go
func (o *Object) refresh(ctx context.Context) error {
    isNotified := o.CacheFs.isNotifiedRemote(o.Remote())
    isExpired := time.Now().After(o.CacheTs.Add(time.Duration(o.CacheFs.opt.InfoAge)))
    if !isExpired && !isNotified {
        return nil // 命中，无需刷新
    }
    return o.refreshFromSource(ctx, true)
}
```

两个条件任一满足即触发失效：
1. **TTL 过期**：`now - CacheTs > InfoAge`（默认 6 小时）
2. **收到通知**：该路径在 `notifiedRemotes` 中被标记（来自 ChangeNotify 或主动失效）

### 3.3 典型元数据操作流程

#### NewObject（查单个文件）
[cache.go#L932-L973](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L932-L973)

```
1. cache.GetObject(co) → 从 Bolt DB 反序列化 Object
2. 检查 CacheTs + InfoAge
   ├─ 命中且未过期 → 直接返回
   └─ 未命中/已过期 → 继续
3. 如启用 tempFs → 先查 tempFs.NewObject
4. 未找到 → f.Fs.NewObject (透传到底层 remote)
5. 将返回结果 ObjectFromOriginal().persist() → 写回 Bolt DB
6. 返回缓存对象
```

#### List（列目录）
[cache.go#L976-L1089](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L976-L1089)

```
1. cache.GetDirEntries(cd) → 读整个目录 Bucket
2. 检查目录 CacheTs + InfoAge
   ├─ 命中且非空 → 直接返回 entries
   └─ 未命中/过期/空 → 继续
3. 如启用 tempFs → 读 pending upload 队列，合并本地临时文件
4. f.Fs.List(ctx, dir) → 透传到底层 remote 列目录
5. 对比缓存与源数据：
   - 源中已不存在的缓存条目 → RemoveObject/RemoveDir
   - 源中的新条目 → ObjectFromOriginal().persist() / AddBatchDir
6. 更新当前目录的 CacheTs = now，写回 DB
```

---

## 四、失效触发机制

缓存失效分为 **显式失效**、**TTL 被动失效**、**ChangeNotify 通知失效** 三类。

### 4.1 写操作触发的显式失效

所有修改目录内容的操作都会 **失效父目录缓存**（将父目录 CacheTs 回拨 `InfoAge`，使 TTL 判定立即过期）：

| 操作 | 失效内容 | 代码位置 |
|------|---------|---------|
| `Put` | 文件 chunks + 父目录 | [cache.go#L1488-L1503](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1488-L1503) |
| `Update` | 文件 chunks | [object.go#L272-L282](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L272-L282) |
| `Remove` | 文件 + chunks + 父目录 | [object.go#L307-L312](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L307-L312) |
| `Mkdir` | 父目录 | [cache.go#L1169-L1175](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1169-L1175) |
| `Rmdir` | 目录 + 全部内容 + 父目录 | [cache.go#L1231-L1245](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1231-L1245) |
| `Move` | 旧文件 + 旧父目录 + 新父目录 | [cache.go#L1677-L1705](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1677-L1705) |
| `DirMove` | 源目录 + 源/目的父目录 | [cache.go#L1340-L1368](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1340-L1368) |
| `Copy` | 源/目的父目录 | [cache.go#L1585-L1606](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1585-L1606) |

**ExpireDir 实现原理**：
[storage_persistent.go#L342-L372](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L342-L372)

```go
func (b *Persistent) ExpireDir(cd *Directory) error {
    t := time.Now().Add(-cd.CacheFs.opt.InfoAge) // 回拨 InfoAge，立即使 TTL 判定过期
    cd.CacheTs = &t
    // 同时向上递归失效所有祖先目录
    // ...
}
```

### 4.2 ChangeNotify：底层 remote 主动通知

[cache.go#L819-L860](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L819-L860)

如果底层 remote 支持 `ChangeNotify`（如 OneDrive、Google Drive 等），cache backend 会订阅：

```go
// NewFs 中注册
if doChangeNotify := wrappedFs.Features().ChangeNotify; doChangeNotify != nil {
    doChangeNotify(ctx, f.receiveChangeNotify, pollInterval)
}
```

收到通知后的处理：
1. 如通知的是文件 → `ExpireObject(file, true)`（含 chunks）+ 失效父目录
2. 如通知的是目录 → `ExpireDir(dir)`
3. 将路径加入 `notifiedRemotes`，后续对象访问时 `isNotifiedRemote()` 立即触发刷新
4. 向上游（如 VFS 层）转发通知 `notifyChangeUpstream()`

### 4.3 手动/RC 命令失效

- **RC `cache/expire`**：[cache.go#L638-L681](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L638-L681) 支持按路径失效，可选 `withData=true` 同时删除 chunk
- **`DirCacheFlush()`**：收到 SIGHUP 信号时清空根目录缓存 [cache.go#L472-L475](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L472-L475)
- **启动时 `db_purge=true`**：删除整个 Bolt DB 和 chunk 目录

### 4.4 TTL 被动失效 + Chunk 空间清理

Chunk 清理是定时触发的：
[cache.go#L505-L517](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L505-L517)

```go
go func() {
    for {
        time.Sleep(time.Duration(f.opt.ChunkCleanInterval)) // 默认 1 分钟
        f.CleanUpCache(false)
    }
}()
```

`CleanChunksBySize` 实现：
[storage_persistent.go#L544-L607](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L544-L607)

按 `DataTsBucket` 的时间戳顺序（FIFO/LRU 混合）从最旧的 chunk 开始删除，直到总大小 ≤ `ChunkTotalSize`。

---

## 五、底层 Remote 透传关系

Cache Backend 是典型的 **装饰器模式**，对外呈现标准 `fs.Fs`，内部按策略决定缓存或透传。

### 5.1 读操作透传链

```
用户调用 Object.Open()
    │
    ▼
Object.refresh() ── TTL/Notified 过期? ── 是 ──► refreshFromSource()
    │                                              │
    │ 未过期                                       ▼
    │                                    CacheFs.Fs.NewObject() ──► 底层 remote
    │                                              │
    ▼                                              ▼
NewObjectHandle(创建读句柄)                  更新 CacheTs + persist()
    │
    ▼
Handle.Read(p)
    │
    ▼
Handle.getChunk(offset)
    ├─ 对齐到 ChunkSize 边界
    ├─ queueOffset() → 向 preloadQueue 提交 N 个连续 chunk
    │
    ├─ Memory.GetChunk() ─────── 命中? ──► 返回
    │    (RAM 中转)
    │
    └─ Persistent.GetChunk() ─── 命中? ──► 返回
         (磁盘文件)
              │
              │ 未命中
              ▼
         worker 池异步下载
              │
              ▼
         Object.Object.Open(RangeOption) ──► 底层 remote HTTP Range 请求
              │
              ▼
         下载后同时写入 Memory + Persistent
```

**Worker 预加载**：
[handle.go#L386-L491](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L386-L491)

每个 Handle 启动 N 个 worker（默认 4，Plex 播放时动态扩缩），从 `preloadQueue` 取 offset 下载 chunk，先检查 Memory→Persistent 中是否已有，没有才发起真实远程请求。

### 5.2 写操作透传链

写入有三种模式，由配置决定：

#### 模式 A：直接透传（默认）
`writes=false` 且 `tmp_upload_path=""`

```go
obj, err = f.Fs.Put(ctx, in, src, options...) // 直接传给底层 remote
// 成功后删除旧缓存 + persist 新 Object
```

#### 模式 B：写入同时缓存
`writes=true`

[cache.go#L1375-L1436](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1375-L1436)

使用 `io.TeeReader` 将上传流一分为二：
- 一路走底层 remote 上传
- 一路走本地按 chunk 切割写入 Persistent 缓存

#### 模式 C：本地暂存 + 后台上传
`tmp_upload_path` 已配置

[cache.go#L1446-L1463](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1446-L1463)

```
Put() → tempFs.Put()（本地磁盘）
     → cache.addPendingUpload()（加入 tempBucket 队列）
     → 立即返回成功

后台 backgroundWriter.run():
    → 轮询 getPendingUpload()
    → 超过 TempWaitTime（默认 15s）后
    → operations.MoveFile(tempFs → remote)
    → 上传成功后 removePendingUpload() + ExpireDir(父目录)
```

### 5.3 元数据操作透传

所有元数据操作遵循统一模式：

```
1. 先透传到底层 remote 执行真实操作
2. 操作成功后：
   a. 新增/修改 → AddObject/AddDir 写缓存
   b. 删除 → RemoveObject/RemoveDir 删缓存
   c. 目录变更 → ExpireDir 失效父目录 + 所有祖先
   d. 转发 ChangeNotify 给上游订阅者（VFS 等）
```

### 5.4 Fs 接口透传矩阵

| Fs 方法 | 缓存策略 | 透传方式 |
|---------|---------|---------|
| `NewObject` | 先查缓存，TTL 过期后 refresh | `f.Fs.NewObject` |
| `List` | 先查缓存，TTL 过期后重列 | `f.Fs.List` |
| `ListR` | 如底层支持则透传并沿途缓存 | `f.Fs.Features().ListR` |
| `Mkdir` | 透传成功后加缓存 + 失效父目录 | `f.Fs.Mkdir` |
| `Rmdir` | 透传成功后删缓存 + 失效父目录 | `f.Fs.Rmdir` |
| `Put/PutStream/PutUnchecked` | 三种模式（直接/缓存写/暂存） | `f.Fs.Put` 等 |
| `Copy` | 先 refresh 源对象，再透传 | `f.Fs.Features().Copy` |
| `Move` | 先 refresh 源对象，再透传 | `f.Fs.Features().Move` |
| `DirMove` | 透传后清 src 缓存 + 失效双方父目录 | `f.Fs.Features().DirMove` |
| `About/UserInfo/...` | 纯透传，不缓存 | 直接 `f.Fs.Features().Xxx` |
| `Hashes` | 纯透传 | `f.Fs.Hashes()` |
| `CleanUp` | 先清本地 chunk，再透传 | `f.Fs.Features().CleanUp` |

---

## 六、关键协作关系图

```
                    ┌──────────────────────────────┐
                    │        VFS / 调用方           │
                    └──────────────┬───────────────┘
                                   │ ChangeNotify 订阅
                                   ▼
                    ┌──────────────────────────────┐
                    │      backend/cache/Fs         │
                    │  (装饰器: 拦截所有 fs.Fs 操作) │
                    └──────┬───────────┬───────────┘
                           │           │
              TTL 判定/读写 │           │ 未命中时透传
                           ▼           ▼
            ┌──────────────────┐   ┌──────────────────┐
            │  Persistent(DB)  │   │  底层 Remote Fs   │
            │  ├─ Bolt DB 元数据│   │  (任意 backend)   │
            │  └─ FS Chunk 文件│   └──────────────────┘
            └────────┬─────────┘
                     │
           流式读加速 │
                     ▼
            ┌──────────────────┐
            │  Memory (RAM)    │
            │  go-cache 短期块  │
            └──────────────────┘

   失效来源:
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │ 本地写操作   │  │Remote Notify│  │ TTL/空间清理 │
   └─────────────┘  └─────────────┘  └─────────────┘
```

---

## 七、总结

1. **元数据缓存**：Bolt DB 按路径分 Bucket 存储 JSON 序列化的 Object/Directory，用 `CacheTs + InfoAge` 做 TTL，`Object.refresh()` / `List()` 作为统一入口做命中判定。

2. **失效触发**：
   - **显式失效**：所有写操作成功后，`Remove*` 删条目或 `Expire*` 回拨 CacheTs，同时递归失效父目录链
   - **通知失效**：订阅底层 remote 的 ChangeNotify，标记 `notifiedRemotes` 触发下一次访问时 refresh
   - **被动失效**：TTL 超时 + 定时 chunk 大小清理（LRU/FIFO 混合）

3. **底层 Remote 透传**：
   - 读：cache miss → worker 池通过 `Object.Object.Open(RangeOption)` 发起 Range 请求下载 chunk，双写 Memory+Persistent
   - 写：三种策略（直传/Tee 并缓存/本地暂存后台上传），操作成功后更新本地缓存
   - 元数据：先透传真实操作，再更新/失效缓存，最后向上游转发 ChangeNotify
