# rclone Cache 状态流转与命中/透传精确分析

## 一、核心概念定义

### 1.1 状态定义

所有缓存条目围绕一个核心变量 **CacheTs（缓存时间戳）** 运作。以 CacheTs 为中心，定义两种基本状态：

| 状态 | 判定公式 | 含义 |
|------|---------|------|
| **新鲜 (Warm/Fresh)** | `now ≤ CacheTs + InfoAge` | 缓存有效，直接返回，不透传 |
| **过期 (Cold/Expired)** | `now > CacheTs + InfoAge` | 缓存失效，必须透传 remote 刷新 |

**关键代码佐证**：三处 TTL 判断全部使用 `time.Now().After(...Add(InfoAge))`，即严格大于才判定为过期。

- 对象：`time.Now().After(co.CacheTs.Add(time.Duration(f.opt.InfoAge)))` [cache.go#L941](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L941)
- 目录：`time.Now().After(cd.CacheTs.Add(time.Duration(f.opt.InfoAge)))` [cache.go#L984](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L984)
- refresh：`time.Now().After(o.CacheTs.Add(time.Duration(o.CacheFs.opt.InfoAge)))` [object.go#L160](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L160)

### 1.2 三条独立的状态触发线

缓存状态变迁由三条独立的线驱动，互不依赖：

| 触发线 | 作用对象 | 操作 | 方向 |
|-------|---------|------|------|
| **时间线** | 所有缓存 | TTL 自然过期 | 新鲜 → 过期 |
| **通知线 A（持久化）** | DB 中存储的条目 | Expire* 回拨 CacheTs | 新鲜 → 过期（立即） |
| **通知线 B（内存标记）** | 已在内存中的 Object 引用 | notifiedRemotes 标记 | 强制刷新一次 |
| **透传线** | 所有缓存 | 从 remote 刷新后重置 CacheTs = now | 过期 → 新鲜 |

---

## 二、对象缓存的完整状态流转（NewObject + refreshFromSource）

### 2.1 状态机总览

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Object 元数据缓存状态机                          │
└──────────────────────────────────────────────────────────────────────┘

                    (A) 时间流逝
   ┌─────────┐  now > CacheTs+InfoAge   ┌─────────┐
   │  新鲜    │ ───────────────────────► │  过期    │
   │  Warm   │ ◄───────────────────────  │  Cold   │
   └────┬────┘   (B) 透传刷新重置CacheTs └────┬────┘
        │                                      │
        │ ▲                                    │ ▲
   (C)  │ │ (E) 通知失效路径A               (D) │ │ (F) 通知失效路径B
        │ │   (ExpireObject回拨CacheTs)        │ │   (notifiedRemotes标记)
        ▼ │                                    ▼ │

   说明：
   (A) 自然过期：时间线驱动，InfoAge 后自动过期
   (B) 刷新重置：NewObject miss 后从 remote 透传，CacheTs = now
   (C) 新鲜态下收到通知路径A：CacheTs 回拨 InfoAge → 立即变过期
   (D) 过期态下收到通知路径A：已经是过期，回拨更过期，无实质变化
   (E) 新鲜态下收到通知路径B：notifiedRemotes 标记 set，但状态仍是新鲜
        （下次 refresh() 时消费标记触发刷新 → 状态仍算新鲜但强制刷新）
   (F) 过期态下收到通知路径B：已经过期，刷新 anyway，标记消费即失
```

### 2.2 状态变迁的代码精确位置

#### (B) 过期 → 新鲜：透传后 CacheTs 重置

**触发点 1：NewObject 从 DB miss 后透传**

[cache.go#L969-L972](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L969-L972)

```go
// 透传成功后
co = ObjectFromOriginal(ctx, f, obj).persist()
```

其中 `ObjectFromOriginal` [object.go#L73-L99](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L73-L99) 中：
```go
co = &Object{
    // ...
    CacheTs: time.Now(),       // ← 构造时设为 now
}
co.updateData(ctx, o)           // ← updateData 中又设了一次 CacheTs = now
```

`updateData` [object.go#L101-L109](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L101-L109)：
```go
func (o *Object) updateData(ctx context.Context, source fs.Object) {
    o.Object = source
    o.CacheModTime = source.ModTime(ctx).UnixNano()
    o.CacheSize = source.Size()
    o.CacheStorable = source.Storable()
    o.CacheTs = time.Now()    // ← 刷新时间戳
    o.CacheHashes = make(map[hash.Type]string)
}
```

最后 `persist()` [object.go#L348-L354](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L348-L354)：
```go
func (o *Object) persist() *Object {
    err := o.CacheFs.cache.AddObject(o)  // ← 写回 Bolt DB
    return o
}
```

**触发点 2：refreshFromSource 强制刷新**

[object.go#L168-L197](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L168-L197)

```go
func (o *Object) refreshFromSource(ctx context.Context, force bool) error {
    liveObject, err = o.CacheFs.Fs.NewObject(ctx, o.Remote()) // ← 透传
    o.updateData(ctx, liveObject) // ← CacheTs = now
    o.persist()                   // ← 写回 DB
    return nil
}
```

**触发点 3：写操作（Put/Update 等）成功后**

[cache.go#L1486-L1491](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1486-L1491)

```go
cachedObj := ObjectFromOriginal(ctx, f, obj)   // ← CacheTs = now
_ = f.cache.RemoveObject(cachedObj.abs())       // ← 删旧条目（含 chunk）
cachedObj.persist()                              // ← 写新条目
```

#### (C)(D) 新鲜/过期 → 过期：通知失效路径 A（ExpireObject）

[storage_persistent.go#L428-L436](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L428-L436)

```go
func (b *Persistent) ExpireObject(co *Object, withData bool) error {
    co.CacheTs = time.Now().Add(time.Duration(-co.CacheFs.opt.InfoAge)) // 回拨
    err := b.AddObject(co)       // ← 写回 DB，持久化生效
    if withData {
        _ = os.RemoveAll(path.Join(b.dataPath, co.abs())) // ← 删磁盘 chunk
    }
    return err
}
```

**数学恒等性**：设通知时刻为 T₀，回拨后 `CacheTs = T₀ - InfoAge`。

TTL 判定：
```
now > CacheTs + InfoAge
    = (T₀ - InfoAge) + InfoAge
    = T₀
```

即：**只要访问发生在 T₀ 之后（now > T₀），恒判定为过期**。临界窗口仅 `now == T₀`（纳秒级）。

#### (E)(F) 通知失效路径 B（notifiedRemotes）

**设置**：[cache.go#L858-L859](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L858-L859)

```go
f.notifiedRemotes[forgetPath] = true
f.notifiedRemotes[cd.Remote()] = true
```

**消费**：[cache.go#L1872-L1883](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1872-L1883)

```go
func (f *Fs) isNotifiedRemote(remote string) bool {
    n, ok := f.notifiedRemotes[remote]
    if !ok || !n { return false }
    delete(f.notifiedRemotes, remote)  // ← 消费即删，一次性
    return n
}
```

**唯一调用点**：`Object.refresh()` 中 [object.go#L159](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L159)

```go
func (o *Object) refresh(ctx context.Context) error {
    isNotified := o.CacheFs.isNotifiedRemote(o.Remote())
    isExpired := time.Now().After(o.CacheTs.Add(time.Duration(o.CacheFs.opt.InfoAge)))
    if !isExpired && !isNotified {
        return nil  // 两个都不满足才跳过刷新
    }
    return o.refreshFromSource(ctx, true) // 任一满足就透传
}
```

**关键**：路径 B **不改变 CacheTs**，它直接触发 refresh 行为。它是对路径 A 的补充——解决内存中已持有的旧对象引用，其 CacheTs 不会因为 DB 更新而自动同步的问题。

### 2.3 为什么通知后首次访问必然透传？

分两种情况讨论：

#### 情况 1：通过 NewObject 新获取对象（从 DB 读）

```
T₀      ChangeNotify 到来
        → ExpireObject → DB 中 CacheTs = T₀ - InfoAge
        → notifiedRemotes[path] = true

T₁ > T₀ 调用 NewObject(path)
        → GetObject(co) 从 DB 读出 co.CacheTs = T₀ - InfoAge
        → TTL 判断：T₁ > (T₀ - InfoAge) + InfoAge = T₀ → true → 过期
        → miss → 透传 f.Fs.NewObject → 拿到新数据
        → ObjectFromOriginal(...).persist() → CacheTs = T₁ → 写 DB
        → 返回对象
```

**结论**：路径 A（CacheTs 回拨）足以保证首次访问透传。`notifiedRemotes` 在 NewObject 路径中**不检查、不影响**。

#### 情况 2：持有旧引用对象，调用属性方法走 refresh()

```
T - 10min  调用方通过 NewObject 拿到 obj，持有引用
           obj.CacheTs = T - 10min（内存中）
           DB 中 CacheTs = T - 10min

T₀ > T     ChangeNotify 到来
           → ExpireObject → DB 中 CacheTs = T₀ - InfoAge（更新了）
           → notifiedRemotes[path] = true（设标记）
           但内存 obj.CacheTs 仍然是 T - 10min（不会自动同步！）

T₁ > T₀   调用方 obj.Size() → refresh()
          → isNotified = isNotifiedRemote(path) → true（消费标记）
          → isExpired = T₁ > (T - 10min) + InfoAge ?
              若 InfoAge = 6h，T₁ - T = 1min → T₁ > T + 5h50m? → false（未过期！）
          → 但 isNotified = true → 走 refreshFromSource → 透传 → CacheTs = T₁
          → 返回新数据
```

**结论**：如果只靠路径 A，内存中的旧引用对象的 TTL 可能尚未过期（因为它的 CacheTs 不是从 DB 重新读的）。路径 B 就是为这种场景兜底的。

### 2.4 刷新后哪些后续访问会重新命中？

**首次透传刷新后**：
- DB 中 CacheTs 被重置为 T_refresh（刷新时刻）
- 内存中对象的 CacheTs（如果是 refreshFromSource 路径）也被重置为 T_refresh

**后续访问的命中条件**：

| 访问方式 | 命中条件 | 说明 |
|---------|---------|------|
| 再 NewObject 一次 | `now ≤ T_refresh + InfoAge` | 从 DB 读新 CacheTs，正常 TTL 判断 |
| 旧引用 .Size()/.ModTime() | `now ≤ T_refresh + InfoAge` 且 notifiedRemotes 无新标记 | refresh() 双条件判断 |
| 旧引用 .Hash() 等其他属性 | 同上 | 共用 refresh() 路径 |
| 再次收到 ChangeNotify | 两条路径任一生效 → 又触发透传 | 循环往复 |

**刷新后的 TTL 窗口**：标准的 InfoAge（默认 6 小时）。期间只要不再收到通知，且 TTL 未过期，所有访问都命中缓存。

---

## 三、目录缓存的完整状态流转（List）

### 3.1 状态机总览

目录缓存比对象多一个**空目录强制确认**的特殊分支：

```
┌───────────────────────────────────────────────────────────────────────┐
│                        Directory 元数据缓存状态机                        │
└───────────────────────────────────────────────────────────────────────┘

              (A) 时间流逝 或 ExpireDir 回拨
   ┌─────────┐  now > CacheTs+InfoAge   ┌─────────┐
   │  新鲜    │ ───────────────────────► │  过期    │
   │  Warm   │ ◄───────────────────────  │  Cold   │
   └────┬────┘   (B) List透传后重置Ts    └─────────┘
        │
        │ (C) entries == 0 且 TTL未过期
        ▼
   ┌──────────────────┐
   │  空目录待确认态    │  → 强制透传（即使TTL未过期）
   │  Empty-Confirm   │  → 透传后如非空则变新鲜
   └──────────────────┘
```

### 3.2 List 命中/透传的完整判断链

[cache.go#L980-L993](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L980-L993)

```go
entries, err = f.cache.GetDirEntries(cd)
if err != nil {
    // 状态1：DB 中不存在 → 直接透传
} else if time.Now().After(cd.CacheTs.Add(time.Duration(f.opt.InfoAge))) {
    // 状态2：TTL 过期 → 透传
} else if len(entries) == 0 {
    // 状态3：空目录 → 强制透传确认（TODO注释：empty dirs from source?）
} else {
    // 状态4：非空且 TTL 未过期 → ✅ 命中，直接返回
    return entries, nil
}
```

**注意**：List 的判断链中 **完全没有** notifiedRemotes 检查。路径 B（内存标记）对 List 无效。

### 3.3 状态变迁的精确代码

#### 过期 → 新鲜：List 透传后重置 CacheTs

[cache.go#L1078-L1086](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1078-L1086)

```go
// cache dir meta
t := time.Now()
cd.CacheTs = &t              // ← 当前目录的 CacheTs 设为 now
err = f.cache.AddDir(cd)     // ← 写回 Bolt DB
```

**⚠️ 特别注意**：List 中当前目录自身的 CacheTs 总是会被更新为 now，但**子目录**的更新是有条件的：

[cache.go#L1060-L1065](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1060-L1065)

```go
case fs.Directory:
    cdd := DirectoryFromOriginal(ctx, f, o)  // ← 构造新 Directory，CacheTs = now
    // check if the dir isn't expired and add it in cache if it isn't
    if cdd2, err := f.cache.GetDir(cdd.abs()); err != nil || time.Now().Before(cdd2.CacheTs.Add(time.Duration(f.opt.InfoAge))) {
        // ↑ 只有当 DB 中不存在 或 DB 中的已过期 时，才写回
        batchDirectories = append(batchDirectories, cdd)
    }
    cachedEntries = append(cachedEntries, cdd)
```

**子目录写回条件**：
- 子目录在 DB 中不存在 → 写入
- 子目录在 DB 中已过期 → 写入（用新的 now 替换旧的）
- 子目录在 DB 中还新鲜 → **不写入**（保持原 CacheTs）

这意味着：**List("父目录") 不会自动刷新其子目录的 CacheTs**（如果子目录缓存本身还新鲜）。子目录要等自己被 List 时才会刷新。

#### 新鲜 → 过期：ExpireDir 递归回拨

[storage_persistent.go#L342-L372](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L342-L372)

```go
func (b *Persistent) ExpireDir(cd *Directory) error {
    t := time.Now().Add(time.Duration(-cd.CacheFs.opt.InfoAge)) // 统一回拨时刻
    cd.CacheTs = &t

    return b.db.Update(func(tx *bolt.Tx) error {
        currentDir := cd.abs()
        for { // 从当前目录向上递归所有祖先
            bucket := b.getBucket(currentDir, false, tx)
            if bucket != nil {
                val := bucket.Get([]byte("."))
                if val != nil {
                    cd2 := &Directory{CacheFs: cd.CacheFs}
                    _ = json.Unmarshal(val, cd2)
                    cd2.CacheTs = &t   // ← 每个祖先都设为同一回拨时刻
                    enc2, _ := json.Marshal(cd2)
                    _ = bucket.Put([]byte("."), enc2)
                }
            }
            if currentDir == "" { break }
            currentDir = cleanPath(path.Dir(currentDir))
        }
        return nil
    })
}
```

**递归失效的语义**：当某个文件变更时，它的父目录、祖父目录……直到根目录，全部标记为过期。这样任何一级目录的 List 都会透传 remote，确保一致性。

### 3.4 通知失效后 List 的状态流转示例

```
假设：InfoAge = 6h

T₀  收到 ChangeNotify("docs/report.pdf", EntryObject)
    → ExpireObject("docs/report.pdf", true)
       对象 CacheTs = T₀ - 6h，删磁盘 chunk
    → ExpireDir("docs")
       "docs".CacheTs = T₀ - 6h（写 DB）
       "".CacheTs = T₀ - 6h（写 DB）
    → notifiedRemotes["docs/report.pdf"] = true
    → notifiedRemotes["docs"] = true（无人消费，Dead Write）

T₀ + 1s  List("docs")
        → GetDirEntries("docs") 读 DB
        → cd.CacheTs = T₀ - 6h
        → TTL：T₀+1s > (T₀ - 6h) + 6h = T₀ → true → 过期
        → miss → 透传 f.Fs.List("docs")
        → sourceEntries 与旧缓存对比，增删合并
        → "docs".CacheTs = T₀+1s → AddDir 写 DB
        → 返回 entries

T₀ + 1s + 1min  再 List("docs")
                 → cd.CacheTs = T₀+1s
                 → TTL：T₀+1s+1m > T₀+1s+6h? → false → 未过期
                 → 非空 → ✅ 命中，不访问 remote
```

**结论**：
- 通知后首次 List：**必然透传**（路径 A 保证，父目录 CacheTs 已回拨）
- 首次 List 之后的窗口：**标准 TTL 窗口（6h）**，期间不再收到通知则全部命中

---

## 四、已打开读取流（Handle）在通知失效后的行为

这是本分析的核心章节：当一个文件已经被 `obj.Open()` 打开并正在读取时，ChangeNotify 到达，后续 Read() 调用如何处理？

### 4.1 Handle 与 Object 的关联结构

#### Handle 字段中的核心引用

[handle.go#L42-L60](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L42-L60)

```go
type Handle struct {
    ctx            context.Context
    cachedObject   *Object        // ← 关键：持有 Object 指针（共享引用！）
    cfs            *Fs
    memory         *Memory        // ← 每个 Handle 独立的 RAM 缓存！
    preloadQueue   chan int64
    preloadOffset  int64
    offset         int64
    seenOffsets    map[int64]bool
    mu             sync.Mutex
    workersWg      sync.WaitGroup
    workers        int
    maxWorkerID    int
    UseMemory      bool           // ← 默认 true（除非 ChunkNoMemory=true）
    // ...
}
```

**关键结构关系**：

| 组件 | 属于谁 | 是否共享 | 说明 |
|------|-------|---------|------|
| `Handle.cachedObject` | Handle 字段 | ✅ 共享 | 与 NewObject 返回的是同一指针 |
| `Handle.memory` | Handle 字段 | ❌ 独立 | `NewObjectHandle` 中 `NewMemory(-1)` 新建，每个 Handle 独有 |
| `Handle.workers` + `preloadQueue` | Handle 字段 | ❌ 独立 | 每个 Handle 有自己的 worker 池 |
| `Fs.cache` (Persistent) | Fs 字段 | ✅ 全局共享 | 所有 Handle 共享同一磁盘缓存 |
| `Fs.notifiedRemotes` | Fs 字段 | ✅ 全局共享 | 全局通知标记表 |

#### Handle 创建时的初始化

[handle.go#L62-L82](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L62-L82)

```go
func NewObjectHandle(ctx context.Context, o *Object, cfs *Fs) *Handle {
    r := &Handle{
        ctx:           ctx,
        cachedObject:  o,                // ← 保存对象指针（共享）
        cfs:           cfs,
        offset:        0,
        preloadOffset: -1,
        UseMemory:     !cfs.opt.ChunkNoMemory,
    }
    r.seenOffsets = make(map[int64]bool)
    r.memory = NewMemory(-1)            // ← 为每个 Handle 创建独立 Memory
    r.preloadQueue = make(chan int64, r.cfs.opt.TotalWorkers*10)
    r.confirmReading = make(chan bool)
    r.startReadWorkers()                // ← 启动本 Handle 的 worker 池
    return r
}
```

### 4.2 Memory 存储（Handle 级私有）的特性

[storage_memory.go](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_memory.go)

```go
func NewMemory(defaultExpiration time.Duration) *Memory {
    mem := &Memory{}
    mem.db = cache.New(defaultExpiration, -1) // ← -1 表示不启动后台清理 goroutine
    return mem
}
```

```go
func (m *Memory) AddChunk(fp string, data []byte, offset int64) error {
    return m.AddChunkAhead(fp, data, offset, time.Second)
}

func (m *Memory) AddChunkAhead(fp string, data []byte, offset int64, t time.Duration) error {
    key := fp + "-" + strconv.FormatInt(offset, 10)
    m.db.Set(key, data, cache.DefaultExpiration) // ← defaultExpiration=-1 → 永不过期
    return nil
}
```

```go
func (m *Memory) CleanChunksByNeed(offset int64) {
    // 遍历所有 chunk，删除 offset 之前的
    for key := range m.db.Items() {
        // 解析 offset
        if keyOffset < offset {
            m.db.Delete(key)
        }
    }
}
```

**Memory 存储的关键特性**：
1. **每个 Handle 独立**：A Handle 的 Memory 与 B Handle 完全隔离
2. **TTL=-1（永不过期）**：`defaultExpiration=-1`，chunk 写入后不会自动过期
3. **唯一清理方式**：`CleanChunksByNeed(offset)`，在 seek 时删除 offset 之前的旧 chunk
4. **不受通知失效直接影响**：`ExpireObject` 只操作 Persistent（磁盘），不碰任何 Handle 的 Memory

### 4.3 Persistent 存储（全局共享）在通知后的变化

通知触发时 [storage_persistent.go#L428-L436](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L428-L436)：

```go
func (b *Persistent) ExpireObject(co *Object, withData bool) error {
    co.CacheTs = time.Now().Add(time.Duration(-co.CacheFs.opt.InfoAge))
    err := b.AddObject(co)
    if withData {
        _ = os.RemoveAll(path.Join(b.dataPath, co.abs()))
        // ↑ 直接删除整个 chunk 目录！
        //   如 dataPath/reports/doc.pdf/ 下的 0, 5242880, 10485760 ... 全部删光
    }
    return err
}
```

**Persistent 的关键特性**：
1. **全局共享**：所有 Handle 共用同一磁盘目录
2. **HasChunk 用 os.Stat 检测**：`os.IsNotExist(err)` 即不存在
3. **GetChunk 用 os.ReadFile**：文件被删后返回 error
4. **DataTsBucket 记录可能过时**：`os.RemoveAll` 删除文件时不同步清理 Bolt DB 中的 DataTsBucket 条目，后台 `CleanChunksBySize` 会清理残留

### 4.4 getChunk 读取路径在通知后的精确行为

[handle.go#L200-L256](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L200-L256)

```go
func (r *Handle) getChunk(chunkStart int64) ([]byte, error) {
    // Step 0: 对齐 chunk 边界 + 触发预加载
    offset := chunkStart % int64(r.cacheFs().opt.ChunkSize)
    chunkStart -= offset
    r.queueOffset(chunkStart)  // ← worker 开始干活

    found := false

    // Step 1: 查 Level 1 - Handle 私有 Memory（独立，不受通知影响）
    if r.UseMemory {
        data, err = r.memory.GetChunk(r.cachedObject, chunkStart)
        if err == nil {
            found = true   // ✅ Memory 命中，直接返回旧数据！
        }
    }

    // Step 2: Memory 未命中 → 查 Level 2 - 全局 Persistent（已被通知删除！）
    if !found {
        for i := range r.cacheFs().opt.ReadRetries * 8 {
            data, err = r.storage().GetChunk(r.cachedObject, chunkStart)
            if err == nil {
                found = true
                break
            }
            fs.Debugf(r, "%v: chunk retry storage: %v", chunkStart, i)
            time.Sleep(time.Millisecond * 500) // ← 每次等 500ms，共 ReadRetries*8 次
        }
    }

    // Step 3: 两级都没找到 → 返回错误
    if err != nil || len(data) == 0 || !found {
        if r.workers == 0 {
            fs.Errorf(r, "out of workers")
            return nil, io.ErrUnexpectedEOF
        }
        return nil, fmt.Errorf("chunk not found %v", chunkStart)
    }

    // Step 4: 对齐偏移裁剪
    if offset > 0 {
        data = data[offset:]
    }
    return data, nil
}
```

**getChunk 在通知失效后的行为矩阵**（假设 ChunkSize=5MB，InfoAge=6h，ChunkNoMemory=false，UseMemory=true）：

| 场景 | Memory (Level 1) | Persistent (Level 2) | getChunk 结果 | 是否访问 remote |
|------|-----------------|---------------------|--------------|----------------|
| **A. chunk 在通知前已读入 Memory，且未被 CleanChunksByNeed 清理** | ✅ 命中 | 已被 `os.RemoveAll` 删除 | ✅ 返回旧 chunk（直接从 Memory） | ❌ 完全不访问 |
| **B. chunk 在通知前已存在 Persistent 上，但 Memory 没有** | ❌ miss | 已被删除 | ❌ 重试 ReadRetries*8 次（每次 500ms）后返回 "chunk not found" | ⚠️ 间接触发 worker 重下 |
| **C. chunk 在通知前已下载完并写了双写（Memory+Persistent）** | ✅ 命中 | 已被删除 | ✅ 返回旧 chunk（从 Memory） | ❌ 不访问 |
| **D. chunk 通知前尚未下载（worker 预加载中或未触发）** | ❌ miss | ❌ miss | ❌ 等待 worker 下载 | ✅ worker 透传下载**新**数据 |
| **E. 读取已超过当前 chunk，seek 触发 CleanChunksByNeed 删除了旧 Memory chunk** | ❌ miss（已被 CleanChunksByNeed 删除） | 已被删除 | ❌ 等待 worker 重新下载 | ✅ worker 透传下载**新**数据 |

### 4.5 Worker 在通知失效后的精确行为

[handle.go#L385-L430](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L385-L430)

```go
func (w *worker) run() {
    defer func() {
        if w.rc != nil { _ = w.rc.Close() }
        w.r.workersWg.Done()
    }()

    for {
        chunkStart, open := <-w.r.preloadQueue
        if chunkStart < 0 || !open { break }

        // ============= Step 1: 再次检查缓存，避免重复下载 =============
        if w.r.UseMemory {
            if w.r.memory.HasChunk(w.r.cachedObject, chunkStart) {
                continue  // Level 1 有，跳过
            }

            // Level 1 没有，试试 Level 2（Persistent）
            data, err = w.r.storage().GetChunk(w.r.cachedObject, chunkStart)
            if err == nil {
                // Level 2 有，升级到 Level 1，不用下载
                err = w.r.memory.AddChunk(w.r.cachedObject.abs(), data, chunkStart)
                continue
            }
            // Level 2 miss → 通知后 Persistent 已被删，必然到这里
        } else if w.r.storage().HasChunk(w.r.cachedObject, chunkStart) {
            continue  // !UseMemory 模式：Persistent 有就跳过
        }

        // ============= Step 2: 三级缓存都 miss，必须透传下载 =============
        chunkEnd := chunkStart + int64(w.r.cacheFs().opt.ChunkSize)
        w.download(chunkStart, chunkEnd, 0)
    }
}
```

#### worker.download：透传 remote 下载 + 失败重试刷新元数据

[handle.go#L432-L478](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L432-L478)

```go
func (w *worker) download(chunkStart, chunkEnd int64, retry int) {
    if retry >= w.r.cacheFs().opt.ReadRetries { return }
    if retry > 0 { time.Sleep(time.Second * time.Duration(retry)) }

    closeOpen := retry > 0
    w.rc, err = w.reader(chunkStart, chunkEnd, closeOpen)
    //   ↑ reader() 内部最终调用：
    //     w.r.cachedObject.Object.Open(w.r.ctx, &fs.RangeOption{Start, End})
    //     ↑↑↑ 这是对底层 remote 的真实透传点

    if err != nil {
        // ====== Open 失败 → 刷新元数据重试 ======
        fs.Errorf(w, "object open failed %v: %v", chunkStart, err)
        err = w.r.cachedObject.refreshFromSource(w.r.ctx, true)
        //   ↑ 关键：直接修改共享的 cachedObject！
        //     o.Object = 新的 remote fs.Object
        //     o.CacheSize, o.CacheModTime, o.CacheTs 全部更新
        w.download(chunkStart, chunkEnd, retry+1)
        return
    }

    data = make([]byte, chunkEnd-chunkStart)
    sourceRead, err = io.ReadFull(w.rc, data)
    if err != nil && err != io.EOF && err != io.ErrUnexpectedEOF {
        // ====== 读取失败 → 刷新元数据重试 ======
        err = w.r.cachedObject.refreshFromSource(w.r.ctx, true)
        w.download(chunkStart, chunkEnd, retry+1)
        return
    }

    data = data[:sourceRead]

    // ====== 成功下载 → 双写 Memory + Persistent ======
    if w.r.UseMemory {
        _ = w.r.memory.AddChunk(w.r.cachedObject.abs(), data, chunkStart)
    }
    _ = w.r.storage().AddChunk(w.r.cachedObject.abs(), data, chunkStart)
    //   ↑ 写回 Persistent，重新建立磁盘 chunk
    //   同时写回 DataTsBucket（时间戳索引）
}
```

#### worker.reader：复用已有 HTTP 连接或重新 Open

[handle.go#L348-L383](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L348-L383)

```go
func (w *worker) reader(offset, end int64, closeOpen bool) (io.ReadCloser, error) {
    if w.rc == nil {
        // 无已有连接 → 新建
        r, err = w.r.cacheFs().openRateLimited(func() (io.ReadCloser, error) {
            return w.r.cachedObject.Object.Open(w.r.ctx, &fs.RangeOption{Start: offset, End: end - 1})
        })
        return r, nil
    }

    if !closeOpen {
        // 尝试复用已有连接
        if do, ok := r.(fs.RangeSeeker); ok {
            _, err = do.RangeSeek(w.r.ctx, offset, io.SeekStart, end-offset)
            return r, err
        } else if do, ok := r.(io.Seeker); ok {
            _, err = do.Seek(offset, io.SeekStart)
            return r, err
        }
    }

    // 不能复用 → 关闭旧连接，开新连接
    _ = w.rc.Close()
    return w.r.cacheFs().openRateLimited(func() (io.ReadCloser, error) {
        r, err = w.r.cachedObject.Object.Open(w.r.ctx, &fs.RangeOption{Start: offset, End: end - 1})
        return r, err
    })
}
```

**Worker 在通知后的关键行为**：

| 情况 | Worker 行为 | 是否重新访问 remote |
|------|------------|-------------------|
| Persistent chunk 已被删除，但 Memory 还有 | `HasChunk(Memory) → true` → continue，不下载 | ❌ 不访问（返回旧数据） |
| Persistent + Memory 都没了（新 chunk 或 seek 过） | 两级都 miss → `download()` → `reader()` → `cachedObject.Object.Open(RangeOption)` | ✅ **是**（获取新版本数据） |
| Open 失败（404/权限过期等） | `refreshFromSource(ctx, true)` → 重新获取 Object → 重试 download | ✅ **是**（且刷新元数据） |
| ReadFull 失败（连接中断） | `refreshFromSource` → 重试 | ✅ **是** |
| 成功下载后 | 双写 Memory + Persistent（重新建立磁盘缓存） | 本次已访问 |

### 4.6 Object.refreshFromSource 对已打开 Handle 的影响（关键！）

Object 是**指针共享**的：

```go
// Handle 持有的是指针
cachedObject *Object

// refreshFromSource 原地修改
func (o *Object) updateData(ctx context.Context, source fs.Object) {
    o.Object = source           // ← 原地替换底层 fs.Object
    o.CacheModTime = ...
    o.CacheSize = source.Size() // ← 原地更新大小
    o.CacheTs = time.Now()
}
```

**对已打开 Handle 的影响矩阵**：

| 影响项 | 是否影响已打开 Handle | 影响方式 |
|-------|---------------------|---------|
| `o.Object` 被替换为新的 remote Object | ✅ 影响 | worker 下次 `reader()` 时用新 Object 调用 Open |
| `o.CacheSize` 变化（文件大小改变） | ✅ 影响 | `queueOffset()` 中 `o >= r.cachedObject.Size()` 的边界判断立即使用新值；`Read()` 中 EOF 判断也立即使用新值 |
| `o.CacheTs` 被重置为 now | ✅ 影响 | 下次属性调用 `refresh()` 时 TTL 判断更新，但 Handle 本身不直接看 CacheTs |
| `Handle.memory` 中的旧 chunk | ❌ **不影响** | 仍在 RAM 中，`getChunk` Level 1 直接返回旧数据 |
| `Handle.workers` 中的活跃 HTTP 连接 | ❌ **不影响** | 已打开的 `w.rc` 连接仍指向旧版本，除非 retry 触发 `closeOpen=true` 关闭重建 |
| `Handle.offset`（当前读位置） | ❌ **不影响** | 读指针保持不变 |

### 4.7 完整时序场景：已打开 Handle 在通知失效后的读取行为

**场景设定**：
- 文件 `docs/big.bin`，大小 50MB，ChunkSize=5MB（共 10 个 chunk）
- InfoAge=6h，ChunkNoMemory=false（UseMemory=true）
- ReadRetries=3（Persistent 重试 24 次，共 12s）
- T₀ 时刻：调用方已 Open 该文件，Handle 正在读取 chunk 2（offset 10MB-15MB）
- T₀ 时刻稍早：worker 已预下载完 chunk 0-5，其中 chunk 0-2 在 Memory 中（最近读过），chunk 3-5 只在 Persistent 中

```
T₀  ChangeNotify("docs/big.bin", EntryObject) 到达
    │
    ├─ receiveChangeNotify
    │   ├─ ExpireObject("docs/big.bin", true)
    │   │   ├─ DB 中 CacheTs = T₀ - 6h（写回）
    │   │   └─ os.RemoveAll(dataPath/docs/big.bin/)
    │   │      → 磁盘上 chunk 3、4、5 等全部被删除！
    │   │      ⚠️ 但 Memory 中的 chunk 0、1、2 完好保留（Handle 私有）
    │   │
    │   ├─ ExpireDir("docs")  → 父目录 CacheTs 回拨
    │   │
    │   └─ notifiedRemotes["docs/big.bin"] = true
    │
    ▼  此时各层状态：
       DB 元数据：过期（CacheTs 已回拨）
       Persistent chunk：全空（被 rm -rf）
       Handle.memory：chunk 0、1、2 存在（旧版本数据）
       Handle.offset：12MB（位于 chunk 2）
       Worker.rc：可能仍有旧 HTTP 连接打开
       notifiedRemotes["docs/big.bin"] = true

T₀ + 1ms  调用方继续 Read()，请求 1MB 数据
    │
    ▼
    Handle.Read(p) → currentOffset=12MB → getChunk(12MB)
    │
    ├─ 对齐到 chunkStart=10MB（chunk 2）
    ├─ queueOffset(10MB) → offset 没变（同 chunk）→ 不触发新预加载
    │
    ├─ Level 1: memory.GetChunk(obj, 10MB) → ✅ 命中（chunk 2 还在 RAM）
    │
    ├─ found=true → 跳过 Level 2
    │
    ├─ offset=12MB-10MB=2MB → 裁掉前 2MB
    │
    └─ 返回 1MB（旧版本 chunk 2 的后半部分）
    │
    ⚠️  返回的是**旧数据**！完全不访问 remote

T₀ + 10ms  调用方继续 Read()，读完 chunk 2 后前进到 chunk 3（15MB）
    │
    ▼
    Handle.Read(p) → currentOffset=15MB → getChunk(15MB)
    │
    ├─ 对齐到 chunkStart=15MB（chunk 3）
    ├─ queueOffset(15MB) → 新位置！
    │   ├─ CleanChunksByNeed(15MB) → 删除 Memory 中 <15MB 的 chunk（chunk 0、1、2 被清）
    │   ├─ seenOffsets[15MB]=true, seenOffsets[20MB]=true...（按 worker 数）
    │   └─ preloadQueue <- 15MB, 20MB, 25MB, 30MB（4 workers）
    │
    ├─ Level 1: memory.GetChunk(obj, 15MB) → ❌ miss（chunk 3 从未进过 Memory）
    │
    ├─ Level 2: storage().GetChunk(obj, 15MB)
    │   ├─ os.ReadFile(...) → ENOENT（文件已被 os.RemoveAll 删除）
    │   ├─ 重试：ReadRetries*8 = 24 次 × 500ms = 12s 等待
    │   │   （期间 worker 正在跑 run()）
    │   │
    │   ▼ Worker 处理 15MB：
    │   ├─ Memory.HasChunk(15MB) → false
    │   ├─ Persistent.GetChunk(15MB) → error（已删）
    │   ├─ 两级都 miss → download(15MB, 20MB, 0)
    │   │   ├─ reader(15MB, 20MB, false)
    │   │   │   ├─ w.rc 已存在（旧连接）
    │   │   │   ├─ 尝试 RangeSeek(15MB)
    │   │   │   │   ├─ 若旧连接仍有效 → seek 成功，继续用旧连接读（返回旧数据！）
    │   │   │   │   └─ 若旧连接失效（远端已失效）→ 关闭，重新 Open
    │   │   │   │       → cachedObject.Object.Open(ctx, Range{15,19.9})
    │   │   │   │         ↑ cachedObject.Object 可能还是旧的！
    │   │   │   │           除非 worker 之前触发过 refreshFromSource
    │   │   │   └─ 返回 rc
    │   │   │
    │   │   ├─ io.ReadFull → 读取 5MB（如果是旧连接读旧数据；新连接读新数据）
    │   │   ├─ 成功 → 双写 Memory + Persistent（建立新版本 chunk 3）
    │   │   └─ 失败（连接断开）→ refreshFromSource → 新 Object.Open → 重试
    │   │
    │   ├─ worker 写完 Persistent.GetChunk(15MB) 终于返回新数据
    │   └─ 重试循环在第 N 次等到了 → found=true
    │
    └─ found=true → 返回 chunk 3 数据
                   （可能是旧连接返回的旧数据，也可能是新连接返回的新数据）

T₀ + 15s  调用方 Read() 请求 chunk 7（35MB），该 chunk 在通知前完全未下载
    │
    ▼
    getChunk(35MB)
    │
    ├─ Level 1: memory → miss
    ├─ Level 2: persistent → miss（已被删）
    ├─ 等待 worker 下载
    │
    ▼ Worker:
    ├─ Memory miss
    ├─ Persistent miss
    ├─ download(35MB, 40MB, 0)
    │   ├─ 若 w.rc 还有效（旧连接）且可 RangeSeek → 用旧连接读旧版本
    │   ├─ 若 w.rc 失效 → reader → cachedObject.Object.Open(Range)
    │   │   ├─ 若 Object 还是旧的，远端文件没变 → 正常读
    │   │   └─ 若 Object 过期（远端文件已改，ETag 变了等）→ Open 失败
    │   │       → refreshFromSource → 新 Object → 新 Open → 读新版本
    │   └─ 读满 → 双写 Memory + Persistent
    │
    └─ 返回新版本 chunk 7（假定连接已失效或文件已改）

T₀ + 5min  调用方 Seek(0) 回到文件开头重新读
    │
    ▼
    Handle.Seek(0, SeekStart) → r.offset = 0
    ├─ queueOffset(0) → 新位置！
    │   ├─ CleanChunksByNeed(0) → 不删（所有 chunk 的 offset ≥ 0）
    │   ├─ preloadQueue <- 0, 5, 10, 15
    │
    ▼ getChunk(0)
    ├─ Memory 中 chunk 0-2 在之前 queueOffset(15MB) 时已被 CleanChunksByNeed 删除
    │   → Memory miss
    ├─ Persistent 中：
    │   chunk 0、1、2 在通知后从未重新下载过 → miss
    │   chunk 3、4、5... 在 T₀+10ms 起已被 worker 重新下载（新版本）→ hit（版本较新）
    │
    └─ 等待 worker 重新下载 chunk 0-2（新版本，因为旧连接可能已失效）
       → 返回新版本数据
```

### 4.8 通知失效后读取流的行为总结

#### 是否重新访问 remote？—— 分三种情况

| 情况 | 是否访问 remote | 数据版本 | 核心机制 |
|------|----------------|---------|---------|
| **Chunk 已在 Handle 私有 Memory 中，未 seek 超过** | ❌ 不访问 | **旧版本** | Memory 每个 Handle 独立，`ExpireObject` 只删 Persistent，不碰 Memory |
| **Chunk 在 Persistent 中但 Memory 没有，且已被通知删除** | ✅ 访问（worker 重下） | **可能旧可能新** | 旧 HTTP 连接若仍有效则继续读旧数据；连接失效则重新 Open 拿新数据 |
| **Chunk 从未下载过（或已被 CleanChunksByNeed 从 Memory 清理）** | ✅ 访问 | **可能旧可能新** | 同上，取决于已有 HTTP 连接是否失效 |

#### 旧数据风险的边界条件

已打开 Handle **可能读到旧版本数据**的条件：
1. `UseMemory=true`（默认）且 chunk 在通知前已读入 Memory，且读取位置未越过该 chunk
2. 或 worker 持有的 HTTP 连接（`w.rc`）在通知后仍然有效，且 RangeSeek 成功复用

旧数据风险**自动消除**的条件：
1. Seek 越过该 chunk → `CleanChunksByNeed` 从 Memory 清理旧 chunk → 下次读取触发 worker 重新下载
2. HTTP 连接失效（超时、远端主动关闭、4xx/5xx）→ worker `refreshFromSource` → 重新 Open 新版本
3. Handle 被 Close → 所有资源释放，下次 Open 走完整 refresh 流程拿到新版本
4. 自然 TTL 过期后重新 Open → refreshFromSource 拿到新版本

#### 为什么不主动关闭已打开 Handle？—— 设计意图

rclone cache backend **故意不主动中断已打开的读取流**。设计意图是：
- 保持读取流的连续性（POSIX 语义：打开文件后即使文件被替换也应继续读旧版本）
- 避免强制关闭导致上层读取错误
- 通过元数据 refresh 让后续的 NewObject/List/refresh 拿到新版本
- chunk 级别的不一致通过 HTTP 连接自然失效和 seek 清理逐步收敛

---

## 五、Chunk 数据缓存的状态流转（三级缓存）

### 5.1 三级缓存架构

Chunk 数据缓存与元数据缓存是**两套独立体系**，有自己的状态流转：

```
            读请求
              │
              ▼
    ┌──────────────────┐
    │  Level 1: Memory  │  RAM，go-cache，短期
    │  (storage_memory) │  每个 Handle 独立，永不过期
    └────────┬─────────┘
             │ miss
             ▼
    ┌──────────────────┐
    │ Level 2: Persistent│ 磁盘文件，持久化，全局共享
    │  (storage_persistent)│
    └────────┬─────────┘
             │ miss
             ▼
    ┌──────────────────┐
    │ Level 3: Remote  │  真实 remote，HTTP Range
    │  (worker 下载)   │  每个 Handle 的 worker 池独立
    └──────────────────┘
             │ 下载后双写
             ▼
        写入 Level 1 + Level 2
```

### 5.2 Chunk 的状态与变迁

| 状态 | 说明 | 触发 |
|------|------|------|
| **不存在** | 任一级都没有该 chunk | 文件刚变更 / 被清理 / 首次访问 |
| **仅 Level 2 有** | 只在磁盘文件中 | 之前已下载过，持久化保存，Memory 已被 Clean |
| **两级都有** | RAM + 磁盘都有 | 最近刚访问过，刚从 Level 2 升级到 Level 1 或刚下载完双写 |
| **worker 下载中** | worker 正在从 remote 拉取 | 预加载队列提交，正在处理 |
| **仅 Level 1 有（已删 Level 2）** | 只在 RAM 中，磁盘被通知失效删除 | ChangeNotify ExpireObject(withData=true) → 此时 getChunk Level 1 仍命中返回旧数据 |

### 5.3 Chunk 状态变迁的关键代码

#### 不存在 → 两级都有：worker 下载

[handle.go#L432-L478](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L432-L478)

```go
func (w *worker) download(chunkStart, chunkEnd int64, retry int) {
    w.rc, err = w.reader(chunkStart, chunkEnd, closeOpen)
    // ↳ 内部最终调用 cachedObject.Object.Open(ctx, &RangeOption{...}) ← 透传 remote
    data = make([]byte, chunkEnd-chunkStart)
    sourceRead, err = io.ReadFull(w.rc, data)

    if w.r.UseMemory {
        _ = w.r.memory.AddChunk(w.r.cachedObject.abs(), data, chunkStart) // Level 1
    }
    _ = w.r.storage().AddChunk(w.r.cachedObject.abs(), data, chunkStart)   // Level 2
    // ↳ AddChunk 会在 DataTsBucket 记录时间戳和大小，用于后续空间清理
}
```

#### Level 2 → Level 1：升级（热数据进 RAM）

[handle.go#L403-L417](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L403-L417)

```go
// worker.run 中
if w.r.UseMemory {
    if w.r.memory.HasChunk(w.r.cachedObject, chunkStart) {
        continue  // Level 1 已有，跳过
    }
    // Level 1 没有，先试试 Level 2 有没有
    data, err = w.r.storage().GetChunk(w.r.cachedObject, chunkStart)
    if err == nil {
        // Level 2 有，升级到 Level 1
        err = w.r.memory.AddChunk(w.r.cachedObject.abs(), data, chunkStart)
        continue  // 不用下载了
    }
}
```

#### Level 1 清理：seek 后清理旧的

[storage_memory.go#L77-L90](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_memory.go#L77-L90)

`CleanChunksByNeed(chunkStart)`：删掉 offset 之前的 chunk。由 `queueOffset` 在位置变化时触发。

#### Level 2 清理：空间不足时 FIFO

[storage_persistent.go#L544-L607](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L544-L607)

`CleanChunksBySize()`：按 DataTsBucket 的时间戳顺序（最早的先删），直到总大小 ≤ ChunkTotalSize。定时 1 分钟触发一次。

### 5.4 ChangeNotify 对 Chunk 的影响

**唯一影响方式**：`ExpireObject(co, true)` 中的 `os.RemoveAll(path.Join(b.dataPath, co.abs()))` 直接删除该文件整个 chunk 目录。

[storage_persistent.go#L428-L435](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L428-L435)

```go
if withData {
    _ = os.RemoveAll(path.Join(b.dataPath, co.abs()))
}
```

**边界**：
- 只删 Level 2（磁盘），不影响 Level 1（RAM，每个 Handle 私有）
- 如果某个 chunk 正在被 Handle 读取且存在于 Memory 中，它不会因通知失效而立即失效
- 但此时该文件的元数据已过期，后续 `refresh()` 会触发元数据刷新；重新 `Open()` 会新建 Handle，旧 Handle 被 Close 后 Memory 也会被 GC

---

## 六、通知失效两条路径的完整协作关系

### 6.1 两条路径的互补性

| 维度 | 路径 A：CacheTs 回拨（持久化） | 路径 B：notifiedRemotes（内存标记） |
|------|------------------------------|-----------------------------------|
| **覆盖范围** | NewObject、List、refresh（所有 TTL 判断）；**Persistent 磁盘 chunk 被删** | 仅 Object.refresh；**不影响 chunk 层级** |
| **作用时机** | 下次从 DB 重读之后 | 下次属性读取 refresh 之前 |
| **对新获取对象** | 完全生效（保证首次透传） | 冗余（路径 A 已保证，路径 B 再消费一次） |
| **对旧引用对象** | 不生效（内存 CacheTs 不会自动同步） | 生效（兜底机制） |
| **对已打开 Handle** | 元数据层面：worker 错误时 refreshFromSource 生效；数据层面：删 Persistent chunk | 无直接影响（Handle 不走 refresh 消费路径） |
| **持续性** | 持久化，直到下次刷新；chunk 删除是不可逆的 | 一次性，消费即没 |
| **重启后** | 仍然有效（DB 里、磁盘已删） | 失效（内存丢失） |

### 6.2 完整协作时序图（典型场景：读取中途收到通知）

```
T - 30s   obj = NewObject("big.bin")  → CacheTs = T-30s
          rc = obj.Open() → 返回 Handle
          → Handle.cachedObject = obj（指针共享）
          → Handle.memory 初始为空
          → workers 启动，preloadQueue 开始推送 chunk 0、1、2...
          → worker.download 双写 Memory + Persistent

T - 20s   rc.Read(buf) → getChunk(0)
          → Memory 命中（worker 刚写完）
          → 返回 chunk 0 数据
          → rc.Read(buf) → getChunk(1)...
          → 陆续读取 chunk 0、1，chunk 2、3 正在下载

T₀        ChangeNotify("big.bin", EntryObject) 到来（远端文件已被覆盖）
          │
          ├─ 路径 A：
          │   ExpireObject("big.bin", true)
          │     ├─ DB CacheTs = T₀ - 6h → 写回
          │     └─ os.RemoveAll(dataPath/big.bin/)
          │         → 磁盘上 chunk 2、3（刚下载完）、chunk 4、5（部分下载）全被删
          │         ⚠️ Handle.memory 中 chunk 0、1、2 仍在 RAM 中，未被触碰
          │   ExpireDir(parent) → 父目录回拨
          │
          └─ 路径 B：
              notifiedRemotes["big.bin"] = true

T₀ + 1ms  rc.Read(buf)  继续读取（当前位置在 chunk 2，offset 10MB-15MB）
          │
          ▼
          getChunk(10MB)
            ├─ queueOffset(10MB) → offset 未变，不触发清理
            ├─ Level 1: memory.GetChunk(10MB) → ✅ 命中（chunk 2 还在 RAM）
            └─ 返回 chunk 2 的旧版本数据
            ⚠️ 读的是旧数据！完全不碰 remote

T₀ + 50ms  rc.Read(buf)  读完 chunk 2，前进到 chunk 3（offset 15MB-20MB）
          │
          ▼
          getChunk(15MB)
            ├─ queueOffset(15MB) → 新位置！
            │   ├─ CleanChunksByNeed(15MB) → 删除 RAM 中 chunk 0、1、2
            │   └─ preloadQueue <- 15、20、25、30（4 workers）
            │
            ├─ Level 1: memory → miss（chunk 3 从未进 Memory）
            ├─ Level 2: persistent.GetChunk(15MB)
            │   → ENOENT（被 os.RemoveAll 删了）
            │   → 开始 24 次重试 × 500ms = 12s 等待
            │
            ▼ Worker 处理 15MB：
            ├─ Memory.HasChunk → false
            ├─ Persistent.GetChunk → ENOENT
            ├─ 两级 miss → download(15, 20, 0)
            │   ├─ reader(15, 20, closeOpen=false)
            │   │   ├─ w.rc 旧连接还在（T-30s 打开的 HTTP Range 连接）
            │   │   ├─ 尝试 RangeSeek(15MB, ...)
            │   │   │   ├─ 如果旧连接仍有效（远端没关）→ seek 成功，读旧版本
            │   │   │   │   → 读到 chunk 3 旧数据
            │   │   │   └─ 如果连接失效（HTTP 416 Range Not Satisfiable 等）
            │   │   │       → _ = w.rc.Close()
            │   │   │       → cachedObject.Object.Open(ctx, Range{15,20})
            │   │   │         ↑ 用旧 Object.Open
            │   │   │         如果远端文件已改 → Open 失败（ETag/Size 不匹配）
            │   │   │           → refreshFromSource(ctx, true)
            │   │   │             → obj.Object = 新 remote Object
            │   │   │             → obj.CacheSize = 新大小
            │   │   │           → 新 Object.Open(ctx, Range{15,20})
            │   │   │           → 读 chunk 3 新版本
            │   │   └─ 返回 rc
            │   │
            │   ├─ io.ReadFull → 读满 5MB
            │   └─ 双写 Memory + Persistent（建立 chunk 3 缓存）
            │
            ├─ Persistent.GetChunk(15MB) 终于在第 N 次重试成功 → found=true
            └─ 返回 chunk 3 数据（可能旧也可能新，取决于连接是否存活）

T₀ + 20s  rc.Read(buf)  chunk 4（20MB）、chunk 5（25MB）...
          → chunk 4、5 在通知时已被 rm，且从未进过 Memory
          → worker 重新下载
          → 若旧 HTTP 连接已因超时/错误关闭，则拿到新版本

T₀ + 5min 调用方 rc.Seek(0) 回到开头
          → queueOffset(0)
          → CleanChunksByNeed(0) 什么都不删（所有 offset ≥ 0）
          → 但 chunk 0、1、2 在 T₀+50ms 那次 queueOffset(15MB) 时已被清
          → getChunk(0) → Memory miss, Persistent miss
          → worker 重新下载 chunk 0
          → 此时旧连接可能已死 → refreshFromSource → Open 新版本
          → 返回 chunk 0 新版本数据

T₀ + 6h  rc.Close()  → Handle.Close
        → close(preloadQueue) → workers 退出 → w.rc 关闭 → workersWg.Wait()
        → Handle.memory 被 GC 回收

后续再 obj.Open() → 走 Object.refresh → notifiedRemotes 已消费但 TTL 过期 → refreshFromSource → 拿到全新 Object + 新 Handle
```

### 6.3 为什么通知后首次访问一定会透传？

| 访问场景 | 保证透传的机制 |
|---------|---------------|
| **NewObject** | 路径 A：DB 中 CacheTs 已回拨，TTL 恒过期 |
| **List** | 路径 A：父目录 CacheTs 已被 ExpireDir 递归回拨，TTL 恒过期 |
| **旧引用 obj.xxx() → refresh** | 路径 B：notifiedRemotes 标记被消费，强制 refreshFromSource（即使 TTL 没过期） |
| **新 NewObject 后立即 .Open()** | 路径 B 冗余触发（路径 A 已确保透传，路径 B 再消费一次标记） |
| **已打开 Handle.Read()** | **不保证！** 见第 4.8 节：Memory 中已有 chunk 直接返回旧数据；chunk 缺失时 worker 重下但旧 HTTP 连接可能仍返回旧数据 |
| **Handle.Read() 触发 worker 下载且 Open 失败** | worker 内部 `refreshFromSource` → 重新获取元数据 + 重新 Open → 拿到新版本 |

**一句话总结**：
- **对象级 API**（NewObject/List）靠路径 A 保证首次透传（回拨持久化 CacheTs）
- **属性级 API**（refresh/Open/ModTime 等）靠路径 B 兜底旧引用（内存标记强制刷新）
- **已打开 Handle 的 chunk 读取**：Memory 中的 chunk 返回旧版本；需要重新下载的 chunk 取决于已有 HTTP 连接是否存活——不保证一定拿到新版本，直到连接失效触发 refreshFromSource

### 6.4 刷新后什么情况下重新命中？

透传刷新完成后，`CacheTs = now`，进入下一轮 TTL 窗口：

| 场景 | 重新命中条件 | 有效窗口 |
|------|-------------|---------|
| 新 NewObject | `now ≤ CacheTs + InfoAge` | InfoAge（默认 6h） |
| List 同目录 | `now ≤ 目录 CacheTs + InfoAge` 且 entries 非空 | InfoAge |
| 旧引用 .Size()/.ModTime() | `now ≤ CacheTs + InfoAge` 且无新通知 | InfoAge |
| Chunk 数据读取（新 Handle） | 首次下载后双写 → 后续相同 chunk 命中 | Memory：直到 CleanChunksByNeed(seek) 清理；Persistent：直到空间不足或下一次通知 |
| Chunk 数据读取（已打开 Handle） | Memory 命中 → 旧数据直到 seek 清理；Persistent 被删后重新下载的新版本 → 正常命中 | 同上 |
| 再次收到 ChangeNotify | 两条路径又触发失效 → 下一轮循环 | - |

**关键边界**：刷新重置只重置被访问的那条路径的 CacheTs。例如 List("docs") 只重置 docs 目录自身的 CacheTs，docs 子目录的 CacheTs 不受影响（除非 DB 中已过期）。

---

## 七、写操作触发的缓存状态变迁

### 7.1 Put 操作的状态流转

[cache.go#L1441-L1505](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1441-L1505)

```
Put 操作（直接写模式）
    │
    ▼
  f.Fs.Put(ctx, in, src, options)  ── 透传 remote
    │
    ▼  成功
    │
    ├─ 1. 删旧缓存：RemoveObject(absPath)
    │     → 删 DB 中 Object 条目
    │     → 删磁盘上该文件的 chunk 目录
    │
    ├─ 2. 写新缓存：ObjectFromOriginal(obj).persist()
    │     → CacheTs = now
    │     → AddObject 写 DB
    │     → 状态：新鲜 ✅
    │
    └─ 3. 失效父目录：ExpireDir(parent)
          → 父目录 CacheTs 回拨 InfoAge
          → 父目录状态：过期 ⏳
          → 下次 List 父目录时透传
```

**文件自身**：Put 成功后立即进入新鲜态（因为刚写入的就是最新数据）
**父目录**：立即进入过期态（下次 List 必须重新列目录以反映新增文件）

### 7.2 Remove 操作的状态流转

[object.go#L300-L320](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L300-L320)

```
o.Remove(ctx)
    │
    ├─ 1. o.refreshFromSource(ctx, false)  ← 确保 o.Object 非空
    │
    ├─ 2. o.Object.Remove(ctx)              ← 透传 remote 真实删除
    │
    ├─ 3. f.cache.RemoveObject(o.abs())     ← 删 DB 条目 + 删磁盘 chunk
    │
    └─ 4. ExpireDir(parent)                 ← 父目录 CacheTs 回拨
```

**文件自身**：从缓存中彻底删除（不是过期，是移除）
**父目录**：进入过期态

---

## 八、总结：状态流转全景矩阵

### 8.1 各操作对缓存状态的影响

| 操作 | 对象缓存状态 | 目录缓存状态（父目录） | Chunk 状态 | 触发路径 |
|------|------------|---------------------|-----------|---------|
| NewObject 命中 | 不变（新鲜） | 不变 | - | 正常 TTL |
| NewObject miss→透传 | 过期 → 新鲜 | 不变 | - | 路径 A/自然过期 |
| List 命中 | - | 不变（新鲜） | - | 正常 TTL |
| List miss→透传 | - | 过期 → 新鲜 | - | 路径 A/自然过期 |
| obj.Size() 等属性 命中 | 不变（新鲜） | - | - | 双条件都不满足 |
| obj.refresh() 透传 | 过期 → 新鲜 | - | - | 路径 A 或 B |
| obj.Open()（新 Handle） | 先 refresh 可能刷新 | - | 新建 Handle，worker 预加载（可能命中旧 Persistent） | 路径 A 或 B + chunk 三级缓存 |
| **Handle.Read()（已打开，通知前 Memory 中有 chunk）** | - | - | **返回旧数据（不访问 remote）** | Memory 私有，不受通知影响 |
| **Handle.Read()（已打开，需要重新下载 chunk）** | - | - | worker 重新下载，可能旧可能新 | HTTP 连接复用/失效决定版本 |
| **Worker.download 出错 → refreshFromSource** | 过期 → 新鲜 | - | 新 chunk 下载（新版本） | worker 内部强制刷新 |
| Put | 无 → 新鲜（新建） | 新鲜 → 过期 | 删旧 chunk | 写操作触发 |
| Remove | 有 → 无（删除） | 新鲜 → 过期 | 全删 | 写操作触发 |
| ChangeNotify（对象） | 新鲜 → 过期（路径 A） + 标记（路径 B） | 父目录新鲜→过期（递归） | 磁盘 chunk 全删；RAM chunk 保留 | 通知触发 |
| ChangeNotify（目录） | 子对象 DB 不变 | 新鲜→过期 + 递归祖先 | - | 通知触发 |

### 8.2 为什么通知后首次访问必然透传？—— 一句话答案（元数据层面）

**对象级查询（NewObject/List）靠路径 A（持久化 CacheTs 回拨）保证 TTL 恒过期**；
**属性级读取（refresh）靠路径 B（notifiedRemotes 内存标记）兜底旧引用对象**；
两条路径互补覆盖所有元数据场景，确保 ChangeNotify 后首次访问无论走哪条入口都必然透传 remote 刷新元数据。

**但 chunk 数据层面例外**：已打开 Handle 的 Memory chunk 返回旧数据；重新下载的 chunk 取决于 HTTP 连接是否存活——不保证一定拿到新版本，直到连接失效触发 worker 内部 refreshFromSource。

### 8.3 刷新后什么时候重新命中？—— 一句话答案

**透传刷新完成后，CacheTs 被重置为 now，进入标准 InfoAge 窗口（默认 6 小时），窗口内无新通知则元数据全部命中**；
**chunk 数据：Memory 命中直到 seek 清理，Persistent 命中直到通知删除或空间淘汰**；
**已打开 Handle：旧 Memory chunk 返回旧数据直到被 CleanChunksByNeed 清理，新下载 chunk 建立新缓存**。

窗口到期（自然 TTL 过期）或再次收到通知（路径 A 或 B）则重新进入过期态，触发下一轮透传。
