# rclone Cache 缓存层深度分析

## 一、整体架构与三层缓存

| 层级 | 代码位置 | 职责 |
|------|---------|------|
| L1 通用 KV | [lib/cache/cache.go](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/lib/cache/cache.go) | 基于 TTL 的通用 KV，支持 Pin/Unpin 引用计数，供上层复用 |
| L2 Fs 实例缓存 | [fs/cache/cache.go](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/fs/cache/cache.go) | 缓存 `fs.Fs` 后端实例，避免重复初始化，内置 L1，通过 `cache.Get()` 获取已存在的 Fs |
| L3 Backend Cache | [backend/cache/](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/) | **本文核心**：独立的 fs.Fs 装饰器后端，包装任意 remote，提供元数据缓存 + 数据块(chunk)缓存 + 写缓冲 |

本文聚焦 **L3 Backend Cache**（即 `backend/cache` 包），重点厘清 **通知失效、普通元数据命中、remote 透传**三者之间的判断边界。

---

## 二、核心数据结构

### 2.1 Fs：缓存后端主体（装饰器）

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
    tempFs    fs.Fs                 // 临时写目录（可选，对应 tmp_upload_path）
    rateLimiter   *rate.Limiter     // RPS 限速
    plexConnector *plexConnector    // Plex 集成
    backgroundRunner *backgroundWriter // 后台上传协程
    cleanupChan chan bool
    parentsForgetFn []func(string, fs.EntryType) // ChangeNotify 上游订阅者（通常是 VFS）
    notifiedRemotes  map[string]bool // 通知失效标记表
    notifiedMu       sync.Mutex
}
```

**设计要点**：`Fs` 嵌入 `fs.Fs`（底层 remote），对外呈现标准 `fs.Fs` 接口，内部拦截所有操作：**先查本地缓存 → 判断是否命中 → miss 时透传到底层**。

### 2.2 Object：文件元数据缓存

[object.go#L24-L40](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L24-L40)

```go
type Object struct {
    fs.Object                       // 嵌入底层 fs.Object（可能为 nil，需 refreshFromSource 填充）
    ParentFs      fs.Fs             // 实际所在 FS（tempFs 或 remote）
    CacheFs       *Fs               // 所属缓存 FS
    Name          string
    Dir           string
    CacheModTime  int64
    CacheSize     int64
    CacheStorable bool
    CacheType     string            // "Object" | "TempObject"
    CacheTs       time.Time         // 元数据缓存时间戳，TTL 判定核心
    CacheHashes   map[hash.Type]string // 哈希懒缓存
    refreshMutex  sync.Mutex        // 防并发重复 refresh
}
```

### 2.3 Directory：目录元数据缓存

[directory.go#L14-L26](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/directory.go#L14-L26)

```go
type Directory struct {
    Directory fs.Directory          // 嵌入底层 fs.Directory（可能为 nil）
    CacheFs      *Fs
    Name         string
    Dir          string
    CacheModTime int64
    CacheSize    int64
    CacheItems   int64
    CacheType    string              // "Directory"
    CacheTs      *time.Time          // 目录缓存时间戳（指针，可能为 nil）
}
```

### 2.4 Handle：文件读句柄 + 预加载工作池

[handle.go#L43-L60](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L43-L60)

```go
type Handle struct {
    ctx          context.Context
    cachedObject *Object
    cfs          *Fs
    memory       *Memory             // RAM 中转缓存（go-cache）
    preloadQueue chan int64          // 待预加载的 chunk offset 队列
    preloadOffset  int64             // 上次触发预加载的起始 offset
    offset         int64             // 当前读位置
    seenOffsets    map[int64]bool    // 已提交给 worker 的 chunk offset
    mu             sync.Mutex
    workersWg      sync.WaitGroup
    workers        int               // 当前 worker 数
    maxWorkerID    int
    UseMemory      bool
}
```

### 2.5 存储层双实现

| 实现 | 文件 | 介质 | 用途 | 失效策略 |
|------|------|------|------|---------|
| `Memory` | [storage_memory.go](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_memory.go) | RAM (go-cache) | 流式读取短期中转，读完即丢 | CleanChunksByNeed：seek 后删除 offset 之前的 chunk |
| `Persistent` | [storage_persistent.go](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go) | Bolt DB + 本地文件系统 | 元数据持久化 + chunk 持久化 | CleanChunksBySize：按时间戳从最旧开始删，至总大小 ≤ ChunkTotalSize |

---

## 三、通知失效机制（边界最容易混淆的部分）

### 3.1 通知失效的两条路径

通知失效存在 **两条独立路径**，作用于不同层面：

#### 路径 A：CacheTs 回拨（持久化，影响下一次 TTL 判定）

`ExpireObject` / `ExpireDir` 将 `CacheTs` 回拨 `InfoAge`，使得下一次 TTL 判断 `now > CacheTs + InfoAge` 恒成立：

[storage_persistent.go#L342-L372](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L342-L372)

```go
func (b *Persistent) ExpireDir(cd *Directory) error {
    // 将 CacheTs 回拨 InfoAge，相当于立即使 TTL 判定过期
    t := time.Now().Add(-cd.CacheFs.opt.InfoAge)
    cd.CacheTs = &t
    // 同时向上递归失效所有祖先目录
    // ...
}
```

**作用范围**：写回 Bolt DB，持久化存在，影响所有后续通过 `NewObject` / `List` 从 DB 读出的判断。

#### 路径 B：notifiedRemotes 标记（内存态，一次性消费）

`receiveChangeNotify` 将被通知的路径写入内存 map，`isNotifiedRemote` 读取并 **消费（删除）** 该标记：

[cache.go#L856-L860](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L856-L860)

```go
f.notifiedMu.Lock()
defer f.notifiedMu.Unlock()
f.notifiedRemotes[forgetPath] = true
f.notifiedRemotes[cd.Remote()] = true
```

[cache.go#L1872-L1883](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1872-L1883)

```go
func (f *Fs) isNotifiedRemote(remote string) bool {
    f.notifiedMu.Lock()
    defer f.notifiedMu.Unlock()

    n, ok := f.notifiedRemotes[remote]
    if !ok || !n {
        return false
    }
    delete(f.notifiedRemotes, remote) // 消费后删除，一次性
    return n
}
```

**作用范围**：仅内存态，一次性消费，**只在 `Object.refresh()` 中被检查**（见第 5 节），不影响 `NewObject` / `List` 的 DB 级 TTL 判断。

### 3.2 receiveChangeNotify 完整流程

[cache.go#L819-L860](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L819-L860)

```
底层 remote 触发 ChangeNotify
        │
        ▼
receiveChangeNotify(forgetPath, entryType)
        │
        ├─ 1. notifyChangeUpstream() → 通知 VFS 等上游订阅者
        │
        ├─ 2. 路径 A（CacheTs 回拨，持久化失效）：
        │     ├─ 若 entryType == EntryObject：
        │     │     ├─ ExpireObject(co, true)  → 回拨 CacheTs + 删除磁盘 chunk
        │     │     └─ 取父目录 cd = dir(forgetPath)
        │     └─ ExpireDir(cd) → 回拨 cd 及所有祖先目录的 CacheTs
        │
        └─ 3. 路径 B（notifiedRemotes 标记，内存一次性失效）：
              notifiedRemotes[forgetPath] = true
              notifiedRemotes[cd.Remote()] = true
```

### 3.3 关键边界：两条路径分别在什么地方被检查

| 失效路径 | 存储位置 | 检查位置 | 触发后果 |
|---------|---------|---------|---------|
| A: CacheTs 回拨 | Bolt DB（持久化） | `NewObject` 第 2 步、`List` 第 2 步、`Object.refresh()` | TTL 判断过期 → 透传 remote 刷新 |
| B: notifiedRemotes | 内存（一次性） | 仅 `Object.refresh()` 中 | 直接触发 `refreshFromSource`，无论 TTL 是否过期 |

**⚠️ 重要边界**：`NewObject` 和 `List` **不检查 notifiedRemotes 标记**，它们只依赖 Bolt DB 中 CacheTs 的 TTL 判断。只有通过已缓存的 Object 调用 `ModTime()/Size()/Storable()/Hash()/Open()` 等方法时，才会走 `Object.refresh()` 从而检查 notifiedRemotes。

---

## 四、单文件查询：NewObject 的完整判断分支

[cache.go#L932-L973](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L932-L973)

```
调用方 → f.NewObject(ctx, remote)
           │
           ▼
     Step 1: NewObject(f, remote)
     构造一个空壳 Object，Dir/Name 填好，Object 字段为 nil
     若启用 tempFs 且该路径在 pending upload 队列中：
         CacheType = "TempObject"，ParentFs = tempFs
     否则：
         CacheType = "Object"，ParentFs = 底层 remote
           │
           ▼
     Step 2: f.cache.GetObject(co)
     从 Bolt DB 按 co.Dir/co.Name 查找并反序列化 JSON 到 co
           │
           ├─ 失败（DB 中不存在 / 父 Bucket 不存在）：
           │     Debugf "find: error" → 进入 Step 4（透传）
           │
           ├─ 成功但 TTL 过期：
           │     time.Now().After(co.CacheTs + InfoAge) → true
           │     Debugf "find: cold object" → 进入 Step 4（透传）
           │     ⚠️ 注意：此处 NOT 检查 notifiedRemotes！
           │
           └─ 成功且 TTL 未过期：
                 Debugf "find: warm object"
                 ──► return co, nil  ← 缓存命中，不访问 remote
```

```
           │（Step 2 未命中，进入透传）
           ▼
     Step 3: 确定实际查询目标 FS
           │
           ├─ tmp_upload_path 已配置：
           │     3a: tempFs.NewObject(ctx, remote)
           │     ├─ 成功 → 使用 tempFs 返回的 obj
           │     └─ 失败 → 继续 3b
           │
           └─ 3b: f.Fs.NewObject(ctx, remote)  ← 透传到底层 remote
                 ├─ 成功 → 使用 remote 返回的 obj
                 └─ 失败 → return nil, err
           │
           ▼
     Step 4: 回写缓存
     ObjectFromOriginal(ctx, f, obj).persist()
         ├─ updateData() → 用底层 obj 填充 CacheModTime/CacheSize/CacheStorable
         ├─ CacheTs = now
         └─ cache.AddObject() → JSON 序列化写入 Bolt DB
           │
           ▼
     return co, nil
```

### NewObject 命中/透传边界总结

| 条件 | 结果 | 是否访问 remote |
|------|------|----------------|
| DB 存在 + `now ≤ CacheTs + InfoAge` | ✅ 命中返回 | 否 |
| DB 不存在 | ❌ miss | 是 |
| DB 存在 + `now > CacheTs + InfoAge` | ❌ miss（TTL 过期） | 是 |
| notifiedRemotes[remote] = true | ❌ **不影响** NewObject！仍按 TTL 判断 | 可能否（若 TTL 未过期） |

**关键结论**：`NewObject` 是 **DB 直读 + TTL 判断**，完全不知道 notifiedRemotes 的存在。如果对象刚被 ChangeNotify 标记但 TTL 尚未过期，`NewObject` 仍可能返回陈旧的缓存对象。但该对象后续调用 `ModTime()/Open()` 等方法时，会通过 `refresh()` 中的 notifiedRemotes 检查触发真正刷新（见第 5 节）。

---

## 五、对象读取刷新：Object.refresh / refreshFromSource / Open

### 5.1 Object.refresh：元数据懒刷新

[object.go#L154-L166](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L154-L166)

`refresh()` 被 Object 的几乎所有属性读取方法隐式调用：
- `ModTime(ctx)` → `o.refresh(ctx)`
- `Size()` → `o.refresh(context.TODO())`
- `Storable()` → `o.refresh(context.TODO())`
- `Hash(ctx, ht)` → `o.refresh(ctx)`
- `Open(ctx, ...)` → 有条件调用

```
调用方（如 obj.ModTime()）
           │
           ▼
     o.refresh(ctx)
           │
           ▼
     isNotified = isNotifiedRemote(o.Remote())
     ├─ true  → 消费（删除）notifiedRemotes 中的该标记
     └─ false → 未被通知
           │
           ▼
     isExpired = now > CacheTs + InfoAge
           │
           ▼
     if !isExpired && !isNotified:
         return nil  ← ✅ 两个条件都不满足，不刷新，直接用缓存
           │
           ▼（任一条件满足）
     o.refreshFromSource(ctx, true)
```

**refresh 的 AND 语义**：必须同时满足「TTL 未过期」**且**「未被通知」，才跳过刷新。任一条件成立即刷新。

### 5.2 Object.refreshFromSource：真实透传 remote

[object.go#L168-L197](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L168-L197)

```
o.refreshFromSource(ctx, force)
     │
     ├─ refreshMutex 加锁（防并发重复刷新）
     │
     ├─ 短路判断：if o.Object != nil && !force → return nil
     │   （已有底层对象且不强制刷新，直接返回）
     │
     ├─ 根据 CacheType 选择目标 FS：
     │   ├─ isTempFile() == true → ParentFs.NewObject(ctx, o.Remote())  ← tempFs
     │   └─ 否则 → CacheFs.Fs.NewObject(ctx, o.Remote())               ← 底层 remote
     │
     ├─ 失败 → return err
     │
     └─ 成功：
          o.updateData(ctx, liveObject)
              → o.Object = liveObject
              → CacheModTime/CacheSize/CacheStorable 从 liveObject 同步
              → CacheTs = now
              → CacheHashes 清空
          o.persist() → 写回 Bolt DB
          return nil
```

### 5.3 Object.Open：文件数据读取的完整链路

[object.go#L217-L246](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L217-L246)

```
o.Open(ctx, options...)
     │
     ▼
     Step A: 确保元数据有效
     │
     ├─ o.Object == nil（空壳对象，还从未填充底层对象）：
     │     → o.refreshFromSource(ctx, true)  ← 强制透传
     │
     └─ o.Object != nil：
           → o.refresh(ctx)  ← 走 TTL + notifiedRemotes 双条件判断
     │
     ▼（Step A 失败直接 return error）
     │
     Step B: 构造 Handle + 处理 Range/Seek
     cacheReader := NewObjectHandle(ctx, o, o.CacheFs)
         ├─ workers = TotalWorkers（默认 4；Plex 连接时先降为 1）
         ├─ memory = NewMemory(-1)   ← 无限期 TTL，由 CleanChunksByNeed 控制
         ├─ preloadQueue = make(chan, TotalWorkers*10)
         └─ startReadWorkers() → 启动 N 个 worker goroutine
     │
     ▼
     Step C: 解析 OpenOption，定位初始 offset
     for option in options:
         SeekOption → offset
         RangeOption → offset, limit
         cacheReader.Seek(offset, io.SeekStart)
     │
     ▼
     return readers.NewLimitedReadCloser(cacheReader, limit)
            （后续调用方 Read() 时才真正触发 chunk 获取）
```

### 5.4 Handle.Read → getChunk：chunk 三级缓存

[handle.go#L262-L288](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L262-L288)

```
调用方 → handle.Read(p)
           │
           ├─ offset >= cachedObject.Size() → return 0, io.EOF
           │
           ▼
     handle.getChunk(currentOffset)
           │
           ▼
     Step 1: 对齐到 chunk 边界
     offsetInChunk = currentOffset % ChunkSize
     chunkStart = currentOffset - offsetInChunk
           │
           ▼
     Step 2: queueOffset(chunkStart) → 触发预加载
     │
     ├─ chunkStart == preloadOffset → 相同位置，跳过
     │
     └─ 新位置：
          ├─ UseMemory → CleanChunksByNeed(chunkStart) 删除之前的 RAM chunk
          ├─ preloadOffset = chunkStart
          ├─ 重置 seenOffsets 中 < chunkStart 的项
          └─ 为每个 worker 提交一个连续 chunk：
              for i in range(workers):
                  o = chunkStart + ChunkSize * i
                  if o >= Size → skip
                  if seenOffsets[o] == true → skip
                  seenOffsets[o] = true
                  preloadQueue <- o
           │
           ▼
     Step 3: 三级缓存按优先级查找
     │
     ├─ Level 1: UseMemory == true → memory.GetChunk(obj, chunkStart)
     │     命中 → goto Step 4
     │
     ├─ Level 2: storage().GetChunk(obj, chunkStart)
     │     （重试 ReadRetries*8 次，每次间隔 500ms，等待 worker 写完）
     │     命中 → goto Step 4
     │
     └─ Level 3: 全部 miss → return "chunk not found" error
                  （理论上不应发生，因为 Step 2 已提交 worker 下载，
                   除非 worker 全部退出或网络持续故障）
           │
           ▼
     Step 4: 处理非对齐偏移
     if offsetInChunk > 0:
         data = data[offsetInChunk:]
           │
           ▼
     copy(p, data)
     handle.offset += readSize
     return readSize
```

### 5.5 Worker.run：后台 chunk 下载

[handle.go#L386-L430](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L386-L430)

```
worker 从 preloadQueue 取 chunkStart
     │
     ▼
     Step 1: 再次检查缓存（避免重复下载）
     │
     ├─ UseMemory:
     │   ├─ memory.HasChunk(obj, chunkStart) → true → continue（跳过）
     │   └─ storage().GetChunk(obj, chunkStart) → 命中
     │         → 写回 memory，continue（只升 Level，不下载）
     │
     └─ !UseMemory:
         storage().HasChunk(obj, chunkStart) → true → continue
     │
     ▼（确认所有层级都没有，才发起下载）
     │
     Step 2: w.download(chunkStart, chunkEnd, retry=0)
     │
     ├─ w.reader(chunkStart, chunkEnd, closeOpen)
     │   ├─ 已有 rc 且支持 RangeSeek → rc.RangeSeek(chunkStart, ...)
     │   ├─ 已有 rc 且支持 io.Seeker → rc.Seek(chunkStart, ...)
     │   └─ 否则 → 关闭旧 rc，重新 open:
     │         cachedObject.Object.Open(ctx, &RangeOption{Start, End})
     │         ↑↑↑ 这是最终对底层 remote 的透传
     │
     ├─ io.ReadFull(rc, data)
     │
     ├─ 失败（非 EOF/UnexpectedEOF）：
     │     cachedObject.refreshFromSource(ctx, true)  ← 刷新元数据
     │     指数退避后 retry+1 重试，最多 ReadRetries 次
     │
     └─ 成功：
          ├─ UseMemory → memory.AddChunk(absPath, data, chunkStart)
          └─ storage().AddChunk(absPath, data, chunkStart)
              → 写磁盘文件 + DataTsBucket 记录时间戳和大小
```

### 对象读取刷新边界总结

| 场景 | 判断条件 | 是否透传 remote |
|------|---------|----------------|
| `refresh()` 判断 | `TTL 未过期 AND notifiedRemotes 无标记` | 否 |
| `refresh()` 判断 | `TTL 过期 OR notifiedRemotes 有标记` | 是（`refreshFromSource`） |
| `Open()` 元数据层 | `o.Object == nil` → 强制 `refreshFromSource` | 是 |
| chunk 读 Level 1 | Memory 命中 | 否 |
| chunk 读 Level 2 | Persistent（磁盘）命中 | 否 |
| chunk 读 Level 3 | 两级都 miss | 是（worker 通过 `Object.Object.Open(RangeOption)` 下载） |

---

## 六、目录列表：List 的完整判断分支

[cache.go#L976-L1089](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L976-L1089)

```
调用方 → f.List(ctx, dir)
           │
           ▼
     Step 1: ShallowDirectory(f, dir)
     构造空壳 Directory，CacheTs = nil
           │
           ▼
     Step 2: f.cache.GetDirEntries(cd)
     从 Bolt DB 读整个目录 Bucket：
       - 读 "." key → JSON 反序列化到 cd（填充 CacheTs 等）
       - 遍历 Bucket：子目录（v==nil）→ 构造 Directory；文件（v!=nil）→ 构造 Object
       - 返回 entries（混合 fs.DirEntries）
           │
           │
           ├─ 失败（Bucket 不存在 / "." key 缺失）：
           │     Debugf "list: error" → 进入 Step 4（透传）
           │
           ├─ 成功但 TTL 过期：
           │     now > cd.CacheTs + InfoAge → true
           │     Debugf "list: cold listing" → 进入 Step 4
           │     ⚠️ 注意：此处 NOT 检查 notifiedRemotes！
           │
           ├─ 成功但 entries 为空（len == 0）：
           │     Debugf "list: empty listing" → 进入 Step 4
           │     （空目录也强制透传确认，TODO 注释质疑此行为）
           │
           └─ 成功且非空且 TTL 未过期：
                 Debugf "list: warm N from cache"
                 ──► return entries, nil  ← ✅ 缓存命中
```

```
           │（Step 2 未命中，进入透传 + 合并）
           ▼
     Step 3: 合并本地临时文件（仅 tmp_upload_path 已配置）
     │
     searchPendingUploadFromDir(cd.abs()) → 查 Bolt DB tempBucket
     对每个 pending：
         tempFs.NewObject(ctx, cleanRoot(queuedRemote))
         → ObjectFromOriginal().persist() → 写缓存
         → 加入 cachedEntries
           │
           ▼
     Step 4: 透传到底层 remote 列目录
     sourceEntries, err = f.Fs.List(ctx, dir)
           │
           ▼
     Step 5: 清除已失效的缓存条目
     对 Step 2 返回的旧缓存 entries：
         entryRemote 不在 sourceEntries 中（二分查找）：
             ├─ entry 是 Object → cache.RemoveObject(fp)
             └─ entry 是 Directory → cache.RemoveDir(fp)
           │
           ▼
     Step 6: 合并 sourceEntries 到 cachedEntries
     对 sourceEntries 中每个 entry：
         ├─ 是 Object：
         │   ├─ 若与 cachedEntries 中同名（temp 文件）→ 跳过（temp 优先）
         │   └─ 否则 → ObjectFromOriginal().persist() 写缓存，加入结果
         │
         └─ 是 Directory：
             DirectoryFromOriginal() 构造
             ├─ 若 DB 中不存在 或 DB 中的 CacheTs 已过期
             │   → 加入 batchDirectories 待批量写回
             └─ 加入结果
           │
           ▼
     Step 7: 批量写目录缓存 + 更新当前目录 CacheTs
     AddBatchDir(batchDirectories)
     cd.CacheTs = &now
     AddDir(cd) → 将当前目录元数据（含新 CacheTs）写回 Bolt DB
           │
           ▼
     return cachedEntries, nil
```

### List 命中/透传边界总结

| 条件 | 结果 | 是否访问 remote |
|------|------|----------------|
| DB Bucket 存在 + `"."` 元数据存在 + `now ≤ CacheTs + InfoAge` + `len(entries) > 0` | ✅ 命中返回 | 否 |
| DB Bucket 不存在 / `"."` 缺失 | ❌ miss | 是 |
| DB 存在 + `now > CacheTs + InfoAge` | ❌ miss（TTL 过期） | 是 |
| DB 存在 + TTL 未过期 + `len(entries) == 0` | ❌ miss（空目录强制确认） | 是 |
| notifiedRemotes[dir] = true | ❌ **不影响** List！仍按 TTL 判断 | 可能否（若 TTL 未过期且非空） |

**关键结论**：`List` 与 `NewObject` 一样，是 **DB 直读 + TTL 判断**，不检查 notifiedRemotes。目录被 ChangeNotify 标记后，如果 TTL 尚未过期，`List` 仍可能返回陈旧的目录列表。但后续对列表中 Object 的属性读取会通过 `Object.refresh()` 的 notifiedRemotes 检查刷新。

---

## 七、写操作的缓存失效与透传统一模式

所有写操作遵循以下模式：

```
1. 如涉及 tempFs → pause 后台上传
2. 透传到底层 remote 执行真实操作
   f.Fs.Put() / f.Fs.Mkdir() / o.Object.Remove() 等
3. 操作成功后更新本地缓存：
   ├─ 新增/覆盖：AddObject / AddDir 写 Bolt DB，CacheTs = now
   ├─ 删除：RemoveObject / RemoveDir 删 Bolt DB 条目 + 删磁盘 chunk 目录
   └─ 目录变更：ExpireDir(parent) 回拨父目录及所有祖先的 CacheTs
4. 如涉及 tempFs → play 恢复后台上传
5. 如底层 remote 不支持 ChangeNotify，或启用了 tempFs：
   notifyChangeUpstreamIfNeeded() → 手动通知 VFS 等上游
```

涉及写操作的完整失效矩阵：

| 操作 | 删条目 | 删 chunks | ExpireDir 父目录 | notifyChangeUpstream |
|------|--------|----------|-----------------|---------------------|
| Put | ✅（旧条目） | ✅ | ✅ | 条件触发 |
| Update | ✅ | ✅ | - | ✅（对象级） |
| Remove | ✅ | ✅ | ✅ | 条件触发 |
| Mkdir | 新增写缓存 | - | ✅ | 条件触发 |
| Rmdir | ✅（整个 Bucket） | ✅ | ✅ | 条件触发 |
| Move | ✅（旧文件） | ✅（旧文件） | ✅（旧+新父目录） | 条件触发 |
| DirMove | ✅（整个源目录） | ✅ | ✅（源+目的父目录） | 条件触发 |
| Copy | - | - | ✅（源+目的父目录） | 条件触发 |

---

## 八、缓存全链路协作图

```
                    ┌──────────────────────────────────┐
                    │         调用方（VFS/CLI 等）        │
                    └──────┬───────────────┬─────────────┘
                           │               │
          NewObject/List   │               │  obj.ModTime/Size/Open/Hash
                           ▼               ▼
                    ┌──────────────────────────────────┐
                    │         backend/cache/Fs           │
                    │  ┌─────────────────────────────┐  │
                    │  │ NewObject: DB + TTL 判断     │  │
                    │  │ List:      DB + TTL + 非空判 │  │
                    │  │ (两者都不看 notifiedRemotes) │  │
                    │  └─────────────────────────────┘  │
                    │  ┌─────────────────────────────┐  │
                    │  │ Object.refresh:              │  │
                    │  │   TTL过期  OR  notified?     │  │
                    │  │   → 任一成立则 refreshFromSrc │  │
                    │  └─────────────────────────────┘  │
                    └──────┬───────────────┬─────────────┘
                           │               │
                   miss/TTL│               │ chunk 三级缓存 miss
                           ▼               ▼
            ┌──────────────────┐   ┌──────────────────────┐
            │  Persistent(DB)  │   │ 底层 Remote Fs        │
            │  Bolt DB 元数据  │   │ (任意 backend)        │
            │  FS Chunk 文件   │   │  Object.Open(Range)   │
            └────────┬─────────┘   └──────────────────────┘
                     │ 流式读加速
                     ▼
            ┌──────────────────┐
            │  Memory (RAM)    │
            │  go-cache 短期块  │
            └──────────────────┘

   失效来源（三条独立路径）：
   ┌──────────────────┐  ┌──────────────────────┐  ┌──────────────────┐
   │ 本地写操作        │  │ ChangeNotify 回调     │  │ TTL超时/空间清理  │
   │ Expire* 回拨Ts   │  │ Expire* + notifiedMap│  │ CleanChunksBySize│
   └──────────────────┘  └──────────────────────┘  └──────────────────┘
```

---

## 九、关键边界结论汇总

### 9.1 notifiedRemotes（通知失效标记）的真实作用域

| API | 是否检查 notifiedRemotes | 说明 |
|-----|--------------------------|------|
| `Fs.NewObject` | ❌ 否 | 仅 DB + TTL 判断 |
| `Fs.List` | ❌ 否 | 仅 DB + TTL + 非空判断 |
| `Fs.ListR` | ❌ 否 | 若底层支持则直接透传并沿途缓存 |
| `Object.ModTime` | ✅ 是 | 通过 `refresh()` 检查 |
| `Object.Size` | ✅ 是 | 通过 `refresh()` 检查 |
| `Object.Storable` | ✅ 是 | 通过 `refresh()` 检查 |
| `Object.Hash` | ✅ 是 | 通过 `refresh()` 检查 |
| `Object.Open` | ✅ 条件性 | `o.Object != nil` 时走 `refresh()`；`o.Object == nil` 时强制透传 |
| `Handle.getChunk` | ❌ 否 | chunk 数据有独立的三级缓存 + worker 机制 |

**语义**：notifiedRemotes 是一种 **「懒标记」**——它不强制立即刷新，而是等调用方下次读取该对象属性时才触发刷新。对象级（NewObject/List）获取不检查，属性级读取才检查。

### 9.2 Expire*（CacheTs 回拨）的真实作用域

| API | 是否受 Expire* 影响 | 说明 |
|-----|---------------------|------|
| `Fs.NewObject` | ✅ 是 | TTL 判断直接基于 DB 中 CacheTs |
| `Fs.List` | ✅ 是 | TTL 判断直接基于 DB 中 CacheTs |
| `Object.refresh` | ✅ 是 | TTL 判断同样基于 CacheTs |
| chunk 数据读取 | ❌ 否（ExpireObject withData=true 会显式删磁盘 chunk） | chunk 另有独立的空间清理机制 |

### 9.3 Remote 透传触发条件

| 层级 | 透传触发条件 | 透传目标 |
|------|-------------|---------|
| 单文件元数据 | DB miss **或** TTL 过期 | `f.Fs.NewObject()` |
| 目录元数据 | DB miss **或** TTL 过期 **或** 缓存为空 | `f.Fs.List()` |
| 对象属性刷新 | TTL 过期 **或** notifiedRemotes 标记 | `o.CacheFs.Fs.NewObject()` |
| Chunk 数据 | RAM miss **且** 磁盘 miss | `Object.Object.Open(RangeOption)` |
| 所有写操作 | 始终透传（先写 remote，后更缓存） | 对应 `f.Fs.Xxx()` / `o.Object.Xxx()` |
