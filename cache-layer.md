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

## 四、Chunk 数据缓存的状态流转（三级缓存）

### 4.1 三级缓存架构

Chunk 数据缓存与元数据缓存是**两套独立体系**，有自己的状态流转：

```
            读请求
              │
              ▼
    ┌──────────────────┐
    │  Level 1: Memory  │  RAM，go-cache，短期
    │  (storage_memory) │
    └────────┬─────────┘
             │ miss
             ▼
    ┌──────────────────┐
    │ Level 2: Persistent│ 磁盘文件，持久化
    │  (storage_persistent)│
    └────────┬─────────┘
             │ miss
             ▼
    ┌──────────────────┐
    │ Level 3: Remote  │  真实 remote，HTTP Range
    │  (worker 下载)   │
    └──────────────────┘
             │ 下载后双写
             ▼
        写入 Level 1 + Level 2
```

### 4.2 Chunk 的状态与变迁

| 状态 | 说明 | 触发 |
|------|------|------|
| **不存在** | 任一级都没有该 chunk | 文件刚变更 / 被清理 / 首次访问 |
| **仅 Level 2 有** | 只在磁盘文件中 | 之前已下载过，持久化保存 |
| **两级都有** | RAM + 磁盘都有 | 最近刚访问过，刚从 Level 2 升级到 Level 1 |
| **worker 下载中** | worker 正在从 remote 拉取 | 预加载队列提交，正在处理 |

### 4.3 Chunk 状态变迁的关键代码

#### 不存在 → 两级都有：worker 下载

[handle.go#L432-L491](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L432-L491)

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

[storage_memory.go#L61-L82](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_memory.go#L61-L82)

`CleanChunksByNeed(chunkStart)`：删掉 offset 之前的 chunk。

#### Level 2 清理：空间不足时 FIFO

[storage_persistent.go#L544-L607](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L544-L607)

`CleanChunksBySize()`：按 DataTsBucket 的时间戳顺序（最早的先删），直到总大小 ≤ ChunkTotalSize。定时 1 分钟触发一次。

### 4.4 ChangeNotify 对 Chunk 的影响

**唯一影响方式**：`ExpireObject(co, true)` 中的 `os.RemoveAll(path.Join(b.dataPath, co.abs()))` 直接删除该文件整个 chunk 目录。

[storage_persistent.go#L428-L435](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L428-L435)

```go
if withData {
    _ = os.RemoveAll(path.Join(b.dataPath, co.abs()))
}
```

**边界**：只删 Level 2（磁盘），不影响 Level 1（RAM）。如果某个 chunk 正在被 Handle 读取且存在于 Memory 中，它不会因通知失效而立即失效。但此时该文件的元数据已过期，后续 `refresh()` 会触发元数据刷新，重新 Open 时会新建 Handle，旧 Handle 被 Close 后 Memory 也会被清理。

---

## 五、通知失效两条路径的完整协作关系

### 5.1 两条路径的互补性

| 维度 | 路径 A：CacheTs 回拨（持久化） | 路径 B：notifiedRemotes（内存标记） |
|------|------------------------------|-----------------------------------|
| **覆盖范围** | NewObject、List、refresh（所有 TTL 判断） | 仅 Object.refresh |
| **作用时机** | 下次从 DB 重读之后 | 下次属性读取 refresh 之前 |
| **对新获取对象** | 完全生效（保证首次透传） | 冗余（路径 A 已保证，路径 B 再消费一次） |
| **对旧引用对象** | 不生效（内存 CacheTs 不会自动同步） | 生效（兜底机制） |
| **持续性** | 持久化，直到下次刷新 | 一次性，消费即没 |
| **重启后** | 仍然有效（在 DB 里） | 失效（内存丢失） |

### 5.2 完整协作时序图（典型场景）

```
T - 10min  ── NewObject("a.txt") → 返回 obj，调用方持有
             obj.CacheTs = T-10min
             DB 中 CacheTs = T-10min
             notifiedRemotes["a.txt"] = 无
             状态：新鲜 ✅

T₀        ── ChangeNotify("a.txt", EntryObject) 到来
             │
             ├─ 路径 A：
             │   ExpireObject("a.txt", true)
             │     DB 中 CacheTs = T₀ - InfoAge
             │     os.RemoveAll → 删磁盘 chunk
             │   ExpireDir(parent)
             │     父目录 CacheTs = T₀ - InfoAge（及祖先）
             │
             └─ 路径 B：
                 notifiedRemotes["a.txt"] = true
                 notifiedRemotes[parent] = true（无人消费）

             DB 状态：已过期 ⏳
             内存 obj 状态：仍为新鲜（CacheTs 还是 T-10min！）⚠️
             notifiedRemotes：true

T₀ + 1s  ── 新调用 NewObject("a.txt")
             → 从 DB 读 CacheTs = T₀ - InfoAge
             → TTL: T₀+1s > T₀ → true → 过期
             → 透传 remote → 拿到新数据
             → persist() → DB CacheTs = T₀+1s
             → 返回新对象 obj2
             → obj2.CacheTs = T₀+1s
             DB 状态：重新新鲜 ✅

T₀ + 1s  ── 旧引用 obj.Size() → refresh()
             → isNotified = isNotifiedRemote("a.txt")
               → true（消费并删除标记）
             → isExpired = T₀+1s > (T-10min)+InfoAge?
               （若 InfoAge=6h，T₁-T=1min → false）
             → isExpired=false AND isNotified=true → 刷新
             → refreshFromSource → 透传 remote
             → updateData → obj.CacheTs = T₀+1s
             → persist() → DB CacheTs = T₀+1s
             DB 状态：新鲜 ✅（已由上一步刷新）
             内存 obj 状态：新鲜 ✅
             notifiedRemotes：已消费，false

T₀ + 2s  ── 旧引用 obj.Hash() → refresh()
             → isNotified = isNotifiedRemote("a.txt") → false（已消费）
             → isExpired = T₀+2s > T₀+1s+6h? → false（远未过期）
             → !isExpired && !isNotified → 跳过
             → 直接返回缓存值
             → ✅ 命中，不透传

T₀ + 6h + 1s  ── obj.ModTime() → refresh()
               → isNotified = false
               → isExpired = T₀+6h+1s > T₀+1s+6h = T₀+6h1s?
                 → 1s > 1s? → After 是严格大于，所以 false？
               → 精确相等时是未过期，但再过 1ns 就过期了
               → 自然 TTL 过期，透传刷新
               → CacheTs 重置为 now
```

### 5.3 为什么通知后首次访问一定会透传？

| 访问场景 | 保证透传的机制 |
|---------|---------------|
| **NewObject** | 路径 A：DB 中 CacheTs 已回拨，TTL 恒过期 |
| **List** | 路径 A：父目录 CacheTs 已被 ExpireDir 递归回拨，TTL 恒过期 |
| **旧引用 obj.xxx() → refresh** | 路径 B：notifiedRemotes 标记被消费，强制 refreshFromSource（即使 TTL 没过期） |
| **新 NewObject 后立即 .Open()** | 路径 B 冗余触发（路径 A 已确保透传，路径 B 再消费一次标记） |
| **Handle.Read → getChunk** | ExpireObject 已删磁盘 chunk；但 RAM 中 chunk 可能短暂残留（通常 Open 会新建 Handle 重新下载） |

**一句话总结**：
- **对象级 API**（NewObject/List）靠路径 A 保证首次透传（回拨持久化 CacheTs）
- **属性级 API**（refresh/Open/ModTime 等）靠路径 B 兜底旧引用（内存标记强制刷新）
- 两者叠加，确保所有场景下通知后首次访问都能透传 remote

### 5.4 刷新后什么情况下重新命中？

透传刷新完成后，`CacheTs = now`，进入下一轮 TTL 窗口：

| 场景 | 重新命中条件 | 有效窗口 |
|------|-------------|---------|
| 新 NewObject | `now ≤ CacheTs + InfoAge` | InfoAge（默认 6h） |
| List 同目录 | `now ≤ 目录 CacheTs + InfoAge` 且 entries 非空 | InfoAge |
| 旧引用 .Size()/.ModTime() | `now ≤ CacheTs + InfoAge` 且无新通知 | InfoAge |
| Chunk 数据读取 | chunk 在 Memory 或 Persistent 中，且未被清理 | Memory：短期（按需清理）；Persistent：直到空间不足被淘汰 |
| 再次收到 ChangeNotify | 两条路径又触发失效 → 下一轮循环 | - |

**关键边界**：刷新重置只重置被访问的那条路径的 CacheTs。例如 List("docs") 只重置 docs 目录自身的 CacheTs，docs 子目录的 CacheTs 不受影响（除非 DB 中已过期）。

---

## 六、写操作触发的缓存状态变迁

### 6.1 Put 操作的状态流转

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

### 6.2 Remove 操作的状态流转

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

## 七、总结：状态流转全景矩阵

### 7.1 各操作对缓存状态的影响

| 操作 | 对象缓存状态 | 目录缓存状态（父目录） | Chunk 状态 | 触发路径 |
|------|------------|---------------------|-----------|---------|
| NewObject 命中 | 不变（新鲜） | 不变 | - | 正常 TTL |
| NewObject miss→透传 | 过期 → 新鲜 | 不变 | - | 路径 A/自然过期 |
| List 命中 | - | 不变（新鲜） | - | 正常 TTL |
| List miss→透传 | - | 过期 → 新鲜 | - | 路径 A/自然过期 |
| obj.Size() 等属性 命中 | 不变（新鲜） | - | - | 双条件都不满足 |
| obj.refresh() 透传 | 过期 → 新鲜 | - | - | 路径 A 或 B |
| obj.Open() | 先 refresh 可能刷新 | - | 新建 Handle，worker 预加载 | 路径 A 或 B + chunk 三级缓存 |
| Put | 无 → 新鲜（新建） | 新鲜 → 过期 | 删旧 chunk | 写操作触发 |
| Remove | 有 → 无（删除） | 新鲜 → 过期 | 全删 | 写操作触发 |
| ChangeNotify（对象） | 新鲜 → 过期（路径 A） + 标记（路径 B） | 父目录新鲜→过期（递归） | 磁盘 chunk 全删 | 通知触发 |
| ChangeNotify（目录） | 子对象 DB 不变 | 新鲜→过期 + 递归祖先 | - | 通知触发 |

### 7.2 为什么通知后首次访问必然透传？—— 一句话答案

**对象级查询（NewObject/List）靠路径 A（持久化 CacheTs 回拨）保证 TTL 恒过期**；
**属性级读取（refresh）靠路径 B（notifiedRemotes 内存标记）兜底旧引用对象**；
两条路径互补覆盖所有场景，确保 ChangeNotify 后首次访问无论走哪条入口都必然透传 remote 刷新。

### 7.3 刷新后什么时候重新命中？—— 一句话答案

**透传刷新完成后，CacheTs 被重置为 now，进入标准 InfoAge 窗口（默认 6 小时），窗口内无新通知则全部命中**；
窗口到期（自然 TTL 过期）或再次收到通知（路径 A 或 B）则重新进入过期态，触发下一轮透传。
