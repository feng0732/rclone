# rclone Cache 与 ChangeNotify 精确关系分析

## 一、ChangeNotify 触发瞬间：两条失效路径的精确行为

### 1.1 receiveChangeNotify 入口代码

[cache.go#L819-L860](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L819-L860)

```go
func (f *Fs) receiveChangeNotify(forgetPath string, entryType fs.EntryType) {
    f.notifyChangeUpstream(forgetPath, entryType) // 先通知 VFS 等上游

    var cd *Directory
    if entryType == fs.EntryObject {
        co := NewObject(f, forgetPath)
        _ = f.cache.GetObject(co)                        // 从 DB 读出当前对象
        _ = f.cache.ExpireObject(co, true)               // 路径 A-1：对象级持久化失效
        cd = NewDirectory(f, cleanPath(path.Dir(co.Remote())))
    } else {
        cd = NewDirectory(f, forgetPath)
    }
    _ = f.cache.ExpireDir(cd)                            // 路径 A-2：目录级持久化失效（递归祖先）

    f.notifiedMu.Lock()
    defer f.notifiedMu.Unlock()
    f.notifiedRemotes[forgetPath] = true                 // 路径 B-1：对象/目录本身标记
    f.notifiedRemotes[cd.Remote()] = true                // 路径 B-2：父目录标记
}
```

**两条路径同时触发，各自独立运作，不互相依赖：**

---

## 二、路径 A：持久化时间戳失效（CacheTs 回拨 + Bolt DB 写回）

### 2.1 ExpireObject：对象级持久化失效

[storage_persistent.go#L428-L436](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L428-L436)

```go
func (b *Persistent) ExpireObject(co *Object, withData bool) error {
    co.CacheTs = time.Now().Add(time.Duration(-co.CacheFs.opt.InfoAge)) // 回拨 InfoAge
    err := b.AddObject(co)           // ← JSON 序列化后写回 Bolt DB（持久化）
    if withData {
        _ = os.RemoveAll(path.Join(b.dataPath, co.abs())) // ← 同步删磁盘上该文件的所有 chunk
    }
    return err
}
```

### 2.2 ExpireDir：目录级持久化失效（向上递归）

[storage_persistent.go#L342-L372](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/storage_persistent.go#L342-L372)

```go
func (b *Persistent) ExpireDir(cd *Directory) error {
    t := time.Now().Add(time.Duration(-cd.CacheFs.opt.InfoAge)) // 统一回拨时刻
    cd.CacheTs = &t

    return b.db.Update(func(tx *bolt.Tx) error {
        currentDir := cd.abs()
        for { // 从当前目录开始，向上遍历所有祖先直到根
            bucket := b.getBucket(currentDir, false, tx)
            if bucket != nil {
                val := bucket.Get([]byte("."))
                if val != nil {
                    cd2 := &Directory{CacheFs: cd.CacheFs}
                    _ = json.Unmarshal(val, cd2)
                    cd2.CacheTs = &t                     // ← 每个祖先都设为同一回拨时刻
                    enc2, _ := json.Marshal(cd2)
                    _ = bucket.Put([]byte("."), enc2)    // ← 逐个写回 Bolt DB
                }
            }
            if currentDir == "" { break }                 // 根目录到达，停止
            currentDir = cleanPath(path.Dir(currentDir))  // 切到父目录
        }
        return nil
    })
}
```

### 2.3 路径 A 的数学恒等性（核心！）

| 操作 | 公式 |
|------|------|
| 失效写入 | `CacheTs = T₀ - InfoAge`（T₀ = 通知到达时刻） |
| TTL 判断 | `now > CacheTs + InfoAge`（3 处全部使用 `time.After` 严格大于） |
| **代入化简** | `now > (T₀ - InfoAge) + InfoAge = T₀` |
| **等价条件** | **`now > T₀`：只要查询发生在通知时刻之后，TTL 恒过期** |

**临界窗口**：仅当查询与通知在**同一纳秒**（`now == T₀`）时，`time.After()` 返回 false，TTL 才不会过期。实际工程中可忽略。

---

## 三、路径 B：内存标记 notifiedRemotes（一次性消费）

### 3.1 写入点（唯一）

[cache.go#L856-L859](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L856-L859)

```go
f.notifiedRemotes[forgetPath] = true    // 被通知的对象或目录本身
f.notifiedRemotes[cd.Remote()] = true   // 其父目录（EntryObject 时为父 dir；EntryDirectory 时为自身）
```

### 3.2 消费点（唯一！整个 cache 包只有 1 处调用）

Grep 结果确证：`isNotifiedRemote` **仅在 `Object.refresh()` 中被调用**。

[cache.go#L1872-L1883](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L1872-L1883)

```go
func (f *Fs) isNotifiedRemote(remote string) bool {
    f.notifiedMu.Lock()
    defer f.notifiedMu.Unlock()
    n, ok := f.notifiedRemotes[remote]
    if !ok || !n { return false }
    delete(f.notifiedRemotes, remote) // ← 消费后立即删除！一次性语义
    return n
}
```

**一次性语义**：读→删→返回。下一次同路径再查就变 false 了。

### 3.3 路径 B 真实作用（路径 A 覆盖不到的死角）

路径 A（CacheTs 回拨）只对 **下一次从 Bolt DB 重新读** 的判断生效。但有一个死角：

> 调用方在通知前已通过 `NewObject` 获取了 Object 实例并持有其引用，内存中该对象的 `CacheTs` 字段仍是**旧值**（DB 已更新，但内存副本不会自动同步）。

完整场景示例（InfoAge = 6h）：

| 时刻 | 事件 | 内存对象 co.CacheTs | DB 中 CacheTs |
|------|------|-------------------|--------------|
| T - 10min | `NewObject("a.txt")` 返回 co，调用方持有引用 | T - 10min | T - 10min |
| T₀ | ChangeNotify 到来，ExpireObject 回拨 DB | T - 10min（不变！） | **T₀ - 6h** |
| T₀ + 1min | 调用方用**旧 co** 执行 `co.Size()` → `refresh()` | T - 10min | T₀ - 6h |

此时 `refresh()` 中 TTL 判断：

```
now = T₀ + 1min
CacheTs (内存) = T - 10min
TTL = (T - 10min) + 6h = T + 5h50m
now > TTL?  →  (T₀ + 1min) > (T + 5h50m)  →  false（TTL 未过期！）
```

**路径 A 在此场景失效**——因为内存对象不会自动从 DB 重新拉取 CacheTs。

路径 B 就是为此而设：`isNotifiedRemote("a.txt")` 返回 true，强制触发 `refreshFromSource`。

### 3.4 路径 B 的边界与限制

| 限制项 | 说明 |
|-------|------|
| 检查位置 | **仅** `Object.refresh()` 中，`NewObject` / `List` **完全不检查** |
| 一次性 | 消费后立即删除，只作用一次。若刷新失败，后续属性读取只能靠 TTL |
| 仅对象级 | 虽然 `notifiedRemotes` 也标记了父目录，但 `refresh()` 是对象级方法，父目录标记实际无消费点（Directory 没有对应的 refresh 方法） |

**父目录标记的命运**：`notifiedRemotes["foo/"] = true` 写入后，**没有任何代码会消费它**。因为 Directory 结构的属性读取不经过 refresh()，且 `List` 不检查 notifiedRemotes。该标记在 `receiveChangeNotify` 中设置但实际是冗余项，会一直存在直到被新通知覆盖或进程退出。

---

## 四、三大核心 API 的判断边界精确定义

### 4.1 NewObject（单文件查询）

[cache.go#L932-L973](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L932-L973)

```
NewObject(ctx, remote)
    │
    ▼
Step 1: 构造空壳 Object（Object=nil）
    │
    ▼
Step 2: GetObject(co)  从 Bolt DB 读 JSON 填充 co
    │
    ├─ 失败（DB 不存在/父 Bucket 不存在）：
    │     → goto Step 4 透传
    │
    ├─ 成功但 TTL 过期：
    │     now > co.CacheTs + InfoAge   ← 严格 After
    │     ⚠️ 此处 NOT 检查 notifiedRemotes
    │     → goto Step 4 透传
    │
    └─ 成功且 TTL 未过期：
          → return co, nil   ← ✅ 命中，不访问 remote
```

```
Step 3（miss 后）: 选择来源 FS
    ├─ tmp_upload_path 已配 → tempFs.NewObject，找不到再 f.Fs.NewObject
    └─ 否则 → f.Fs.NewObject(ctx, remote)  ← 透传 remote
    │
    ▼
Step 4: ObjectFromOriginal(obj).persist()
    CacheTs = now  →  AddObject 写 DB  →  return
```

**NewObject 通知后行为（T > T₀）**：
- 从 DB 读出的 `co.CacheTs = T₀ - InfoAge`
- TTL：`T > (T₀ - InfoAge) + InfoAge = T₀` → **true，恒过期**
- **必然透传 remote**（除纳秒临界窗口）
- notifiedRemotes **不检查、不消费**

---

### 4.2 List（目录列表）

[cache.go#L976-L1089](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/cache.go#L976-L1089)

```
List(ctx, dir)
    │
    ▼
Step 1: ShallowDirectory 构造空壳（CacheTs=nil）
    │
    ▼
Step 2: GetDirEntries(cd)  读整个目录 Bucket
    读 "." key 填充 cd.CacheTs，遍历 entries
    │
    ├─ 失败（Bucket 不存在 / "." key 缺失）：
    │     → goto Step 4 透传
    │
    ├─ 成功但 TTL 过期：
    │     now > cd.CacheTs + InfoAge   ← 严格 After
    │     ⚠️ 此处 NOT 检查 notifiedRemotes
    │     → goto Step 4 透传
    │
    ├─ 成功且 TTL 未过期 但 entries 为空：
    │     （TODO 注释："empty dirs from source?"）
    │     → goto Step 4 透传（强制确认空目录）
    │
    └─ 成功 + TTL 未过期 + len(entries) > 0：
          → return entries, nil   ← ✅ 命中
```

```
Step 3-4（miss 后）:
    3. tmp_upload_path 已配 → 合并 pending upload 队列中 tempFs 文件
    4. f.Fs.List(ctx, dir)  ← 透传 remote 列目录
    5. 旧缓存中存在但 source 没有的条目 → RemoveObject/RemoveDir
    6. sourceEntries 中文件 → ObjectFromOriginal().persist()
       sourceEntries 中目录 → 若 DB 不存在或 DB CacheTs 已过期 → AddBatchDir 批量写回
    7. 当前目录 cd.CacheTs = now → AddDir 写回 DB
    → return cachedEntries
```

**List 通知后行为（T > T₀）**：
- 从 DB 读出的 `cd.CacheTs = T₀ - InfoAge`（ExpireDir 已写回）
- TTL：`T > T₀` → **true，恒过期**
- **必然透传 remote**（除纳秒临界窗口）
- notifiedRemotes **不检查、不消费**
- 空目录判断不影响结论（TTL 已先过期）

**特别注意**：ChangeNotify 通知的是**子对象/子目录变更**，`ExpireDir` 回拨的是**父目录 CacheTs**。所以 List("foo/") 虽然不直接看 notifiedRemotes["foo/"]，但路径 A 已让 foo/ 的 TTL 过期，最终结果一致。

---

### 4.3 Object.refresh / refreshFromSource / Open（对象属性读取与文件打开）

[object.go#L154-L246](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/object.go#L154-L246)

#### refresh() 双条件判断（AND 语义命中，OR 语义刷新）：

```go
func (o *Object) refresh(ctx context.Context) error {
    isNotified := o.CacheFs.isNotifiedRemote(o.Remote()) // 路径 B：消费并删除标记
    isExpired := time.Now().After(o.CacheTs.Add(InfoAge)) // 路径 A：内存 CacheTs 参与 TTL
    if !isExpired && !isNotified {  // ← 两个都不成立才跳过
        return nil
    }
    return o.refreshFromSource(ctx, true)
}
```

**真值表：**

| isExpired (TTL) | isNotified (通知) | 结果 | 说明 |
|-----------------|------------------|------|------|
| false | false | ✅ 不刷新 | 正常缓存命中 |
| false | **true** | ⚡ 刷新 | **路径 B 单独触发**（覆盖路径 A 死角：内存旧 CacheTs） |
| **true** | false | ⚡ 刷新 | 路径 A 单独触发（NewObject 刚刷新/或正常 TTL 过期） |
| **true** | **true** | ⚡ 刷新 | 两条都触发（isNotified 标记被消费，TTL 实际生效） |

#### refreshFromSource：真实透传

```go
func (o *Object) refreshFromSource(ctx context.Context, force bool) error {
    o.refreshMutex.Lock(); defer o.refreshMutex.Unlock()

    if o.Object != nil && !force { return nil } // 已有底层对象且不强制→跳过

    var liveObject fs.Object
    if o.isTempFile() {
        liveObject, err = o.ParentFs.NewObject(ctx, o.Remote())     // tempFs
    } else {
        liveObject, err = o.CacheFs.Fs.NewObject(ctx, o.Remote())   // ← 透传 remote
    }
    o.updateData(ctx, liveObject) // o.Object = liveObject；同步 ModTime/Size/Storable；CacheTs=now；清空 Hash
    o.persist()                   // 写回 Bolt DB
    return nil
}
```

#### Object.Open：打开文件读取数据

```go
func (o *Object) Open(ctx context.Context, options ...fs.OpenOption) (io.ReadCloser, error) {
    var err error
    if o.Object == nil {
        err = o.refreshFromSource(ctx, true)  // 空壳对象：强制透传
    } else {
        err = o.refresh(ctx)                  // 非空壳：走双条件 refresh
    }
    if err != nil { return nil, err }

    cacheReader := NewObjectHandle(ctx, o, o.CacheFs)
    // 解析 SeekOption/RangeOption 定位 offset
    // 启动 N 个 worker 预加载 chunk
    return readers.NewLimitedReadCloser(cacheReader, limit), nil
}
```

**refresh 通知后行为分两种情况：**

| 场景 | TTL 判断 | isNotified | 结果 |
|------|---------|-----------|------|
| 对象是通知后 **新 NewObject 获取**（CacheTs 已在 NewObject 时重置为 T₀+ε） | false | **true（首次调用消费）** → 后续调用 false | 首次仍触发 refresh；之后靠 TTL |
| 对象是通知前 **已持有引用**（CacheTs 仍为旧值） | 取决于旧 CacheTs 是否已过期 | **true（首次消费）** | 一定刷新（至少一个条件成立） |

#### Chunk 数据读取（与元数据通知解耦）

[handle.go#L200-L259](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L200-L259)

Chunk 数据有独立的三级缓存体系，**不直接受 notifiedRemotes 影响**：

```
Handle.Read(p) → getChunk(currentOffset)
    │
    ▼
对齐到 chunk 边界 → queueOffset() 提交 worker 预加载
    │
    ▼ Level 1（RAM）：memory.GetChunk(obj, chunkStart)
    ├─ 命中 → 返回
    └─ 未命中
         │
         ▼ Level 2（磁盘）：storage().GetChunk(obj, chunkStart)
         │  （重试 ReadRetries*8 次，等 worker 落盘）
         ├─ 命中 → 返回
         └─ 未命中 → return "chunk not found" error
```

**ChangeNotify 对 chunk 的影响仅通过 `ExpireObject(..., true)` 中 `os.RemoveAll` 直接删除磁盘上该文件的 chunk 目录实现**，不经过 notifiedRemotes。

Worker 真实下载透传：
[handle.go#L432-L491](file:///d:/fz/0601-2/solo-dogfeeding/code/51-rclone/backend/cache/handle.go#L432-L491)

```go
func (w *worker) download(chunkStart, chunkEnd int64, retry int) {
    w.rc, err = w.reader(chunkStart, chunkEnd, closeOpen)
    //   ↳ 内部最终调用：cachedObject.Object.Open(ctx, &RangeOption{Start, End})
    //     ↑↑↑ 这是 chunk 对底层 remote 的唯一透传点
    data = make([]byte, chunkEnd-chunkStart)
    sourceRead, err = io.ReadFull(w.rc, data)
    // 写 Memory + 写 Persistent（双写）
    if w.r.UseMemory { w.r.memory.AddChunk(...) }
    w.r.storage().AddChunk(...)
}
```

---

## 五、ChangeNotify 后时序场景大全

### 场景假设
- InfoAge = 6h（默认）
- T₀ 时刻收到 ChangeNotify：文件 `docs/report.pdf`（EntryObject）修改
- `ExpireObject(docs/report.pdf, true)` + `ExpireDir(docs)`（含根目录递归）
- `notifiedRemotes["docs/report.pdf"]=true`，`notifiedRemotes["docs"]=true`

### 各时刻调用行为

| 时刻 | 调用 | TTL 计算 | notifiedRemotes | 是否透传 remote | 说明 |
|------|------|---------|----------------|----------------|------|
| T₀ + 1ns | `NewObject("docs/report.pdf")` | `T₀+1ns > (T₀-6h)+6h = T₀` → true | 不检查 | **是** | 正常；仅纳秒级才可能不中 |
| T₀ + 1ns | `List("docs")` | `T₀+1ns > T₀` → true（父 dir CacheTs 已回拨） | 不检查 | **是** | 父目录 CacheTs 由 ExpireDir 回拨 |
| T₀ + 1ns | 旧引用 `obj.Size()` → `refresh()` | 取决于旧 CacheTs（如 T₀-10min：`T₀+1ns > (T₀-10min)+6h` → false） | **消费=true** | **是** | 路径 B 覆盖死角 |
| T₀ + 1ns | 刚 NewObject 后立即 `obj.ModTime()` → `refresh()` | `T₀+1ns > (T₀+1ns)+6h` → false | **消费=true** | **是** | 看似冗余，实则防御：NewObject 到 Open 间可能又有远端变化 |
| T₀ + 1min | `NewObject("docs/report.pdf")` | `T₀+1m > T₀` → true | 不检查 | **是** | 路径 A 生效 |
| T₀ + 1min | 旧引用 `obj.Hash()` → `refresh()` | 取决于旧 CacheTs（过期则 true；未过期则 false） | **已消费=false** | TTL 过期→是；否则否 | 标记已被上一步 Size() 消费！ |
| T₀ + 1min | `List("docs")` | `T₀+1m > T₀` → true | 不检查 | **是** | 路径 A 生效 |
| T₀ + 1min | `NewObject("docs/report.pdf").Open(...)` | Open 内部 refresh：TTL false + **消费=true** | 消费=true | **是（元数据+chunk）** | notifiedRemotes 触发元数据 refresh；chunk 被 ExpireObject 删除，必重新下载 |
| T₀ + 6h + 1s | 旧引用 `obj.Size()` → `refresh()` | `T₀+6h+1s > (T₀-10min)+6h = T₀+5h50m` → true | 已消费=false | **是** | 路径 A（自然 TTL 过期）兜底，无需标记 |

---

## 六、各机制之间的关系总结图

```
底层 remote 触发 ChangeNotify(forgetPath="docs/report.pdf", EntryObject)
        │
        ▼
receiveChangeNotify
        │
        ├──────────────────────────────────────────────────────────┐
        │                                                          │
        ▼ 路径 A（持久化，Bolt DB 写回）                             ▼ 路径 B（内存，一次性标记）
  ExpireObject("docs/report.pdf", true)                   notifiedRemotes["docs/report.pdf"] = true
  ├─ CacheTs = T₀ - 6h  → AddObject 写 DB                notifiedRemotes["docs"] = true
  └─ os.RemoveAll(磁盘 chunk 目录)                          │
        │                                                     │ 唯一消费点：Object.refresh()
        ▼                                                     ▼
  ExpireDir("docs")                               isNotifiedRemote(remote) → 读→删→返回
  ├─ "docs".CacheTs = T₀ - 6h  → 写 DB
  └─ "".CacheTs = T₀ - 6h      → 写 DB                        │
        │                                                     ▼
        ▼                                            Object.refresh() 真值表：
  DB 中所有相关条目 CacheTs 均被回拨                        ┌─ TTL OK    AND NOT notified → ✅ skip
        │                                                 ├─ TTL EXPIRED OR  notified → ⚡ refreshFromSource
        │  影响的判断点：                                       │
        │  ┌─ NewObject: now > CacheTs+6h  ──► 总是 true       ▼
        │  ├─ List:      now > CacheTs+6h  ──► 总是 true    透传 f.Fs.NewObject()
        │  └─ refresh:   now > CacheTs+6h  ──► 视内存 CacheTs 而定
        │
        ▼
  对新获取对象（NewObject → 从 DB 读）：
    now > T₀ 恒成立 → 总是透传
  对已持有旧引用对象（不从 DB 重读）：
    TTL 可能不成立 → 需要路径 B 兜底
```

---

## 七、最终结论矩阵

### 7.1 ChangeNotify 后各 API 是否必然透传 remote

| API | 是否必然透传 | 依赖的失效路径 | 例外/边界 |
|-----|-------------|---------------|----------|
| `NewObject(path)` | **是**（T > T₀） | A（CacheTs 回拨） | 通知后同一纳秒内的极端竞态 |
| `List(dir)` | **是**（T > T₀） | A（父目录 CacheTs 回拨）+ 空目录强制确认 | 同纳秒竞态；若 dir 非通知路径的祖先则不受影响 |
| `obj.ModTime()` / `Size()` / `Storable()` / `Hash()` | **是（至少首次）** | A（新对象 TTL 过期） **或** B（旧引用消费标记） | 若 B 标记已被另一个属性消费，且旧引用 TTL 未过期，第二次属性读取可能不透传（设计边界） |
| `obj.Open()` | **是** | 元数据：A/B 任一；Chunk：ExpireObject 已删磁盘文件 | Chunk 若仅在 RAM 中（未写 Persistent），则不受 ExpireObject 影响（实际场景极罕见：Open 后短时间内收到通知且 RAM 未清理） |
| `Handle.Read()`（chunk） | **Chunk 级独立** | ExpireObject 的 `os.RemoveAll` 删除磁盘 chunk；三级缓存独立判断 | 若 chunk 仅在 RAM 且内存对象未 GC，理论上可能读到旧数据 |

### 7.2 两条失效路径的定位

| 维度 | 路径 A：CacheTs 回拨（持久化） | 路径 B：notifiedRemotes（内存） |
|------|------------------------------|-------------------------------|
| **覆盖的 API** | NewObject、List、Object.refresh（所有涉及 TTL 判断的） | 仅 Object.refresh |
| **作用对象** | 下次从 DB 重新读取的所有对象 | 已在内存中被持有的旧引用对象 |
| **持续性** | 持久化，写到 Bolt DB，进程重启不丢 | 内存态，一次性消费，进程重启丢失 |
| **触发方式** | TTL 判断自动生效（now > T₀ 恒成立） | 下次属性读取时主动消费标记 |
| **冗余度** | 新获取场景下与路径 B 部分冗余 | 旧引用场景下唯一有效的机制 |
| **父目录** | ExpireDir 递归所有祖先，全部回拨 | 标记了父目录但实际无消费代码（Dead Write） |
