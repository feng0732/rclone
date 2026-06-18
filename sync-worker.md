# rclone Sync 同步流程深度追踪

## 总览

rclone 的 `sync` 命令将源端（fsrc）同步到目的端（fdst），使目的端与源端完全一致。核心实现在 [sync.go](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go) 中，由 `syncCopyMove` 结构体驱动，三个核心机制——**差异计算**、**传输队列**、**删除策略**——协同推进整个同步流程。

入口调用链：

```
cmd/sync/sync.go  →  sync.Sync()  →  runSyncCopyMove()  →  syncCopyMove.run()
```

---

## 一、差异计算：March + Callback 三路分发

### 1.1 March 引擎——目录并行遍历与匹配

[March](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/march/march.go#L30-L49) 是 rclone 的目录并行遍历引擎。它同时遍历 fsrc 和 fdst 的目录树，对每个目录中的条目执行排序后双指针匹配：

```
srcChan ──┐
           ├── matchListings() ──→ SrcOnly / DstOnly / Match
dstChan ──┘
```

关键实现在 [matchListings](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/march/march.go#L292-L374)：

- 从 `srcChan` 和 `dstChan` 分别读取条目
- 对名字做 Unicode NFC 归一化和大小写归一化（如果目的端大小写不敏感）
- 双指针比较排序后的条目名，产生三种结果：
  - **srcName < dstName** → `srcOnly(src)`：条目仅存在于源端
  - **srcName > dstName** → `dstOnly(dst)`：条目仅存在于目的端
  - **srcName == dstName** → `match(dst, src)`：两端都有同名条目
- 检测重复条目并跳过

March 的 `Run` 方法（[march.go#L184-L272](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/march/march.go#L184-L272)）用 `checkers` 数量的 goroutine 并行处理目录遍历作业（`listDirJob`），以工作窃取模式递归调度子目录。

### 1.2 Callback 接口——三路分发驱动同步

`syncCopyMove` 实现了 [Marcher](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/march/march.go#L52-L59) 接口的三个方法，March 的匹配结果直接路由到不同的同步动作。

**三路分发的真实数据流向（对照代码）：

| 分发结果 | 流入位置 | 数据去向 |
|---------|---------|-----------|
| **SrcOnly（文件） | [sync.go#L1245-L1268](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1245-L1268) | trackRenames=true → `trackRenamesCh<br/>

**SrcOnly（文件） | [sync.go#L1253-L1267](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1253-L1267) | trackRenames=false → CompareOrCopyDest → `toBeUploaded` |
| **DstOnly（文件） | [sync.go#L1059-L1089](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1059-L1089) | DeleteModeAfter → `dstFiles` map<br/>DeleteModeDuring → `deleteFilesCh` |
| **Match（文件vs文件） | [sync.go#L1287-L1306](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1287-L1306) | `toBeChecked` |

#### SrcOnly（[sync.go#L1236-L1282](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1236-L1282)

源端独有的条目：

```go
if s.trackRenames {
    // 保存到 trackRenamesCh，后续再处理
    select {
    case <-s.ctx.Done():
        return
    case s.trackRenamesCh <- x:
    }
} else {
    // 直接走 CompareOrCopyDest，检查是否能从 compare-dest/copy-dest 免除传输
    NoNeedTransfer, err := operations.CompareOrCopyDest(s.ctx, s.fdst, nil, x, s.compareCopyDest, s.backupDir)
    if !NoNeedTransfer {
        // 直接放入 toBeUploaded，不经过 toBeChecked！
        ok := s.toBeUploaded.Put(s.inCtx, fs.ObjectPair{Src: x, Dst: nil})
    }
}
```

**关键发现**：SrcOnly 当 trackRenames=false 时，**直接进入 `toBeUploaded`，**不经过 `toBeChecked`。

#### DstOnly（[sync.go#L1041-L1091](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1041-L1091)

目的端独有的条目：

- **文件**：根据 `deleteMode` 决定如何处理：
  - `DeleteModeAfter`：将对象加入 `dstFiles` map，留待同步结束后批量删除
  - `DeleteModeDuring`/`DeleteModeOnly`：直接送入 `deleteFilesCh`，由后台 deleter goroutine 即时删除
- **目录**：记录到 `dstEmptyDirs`，返回 `recurse=true` 继续递归（以便遍历子内容以供删除）

#### Match（[sync.go#L1285-L1343](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1285-L1343)

两端都存在的条目：

- **文件 vs 文件**：构造 `ObjectPair{Src, Dst}` 放入 `toBeChecked` 管道（[sync.go#L1296](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1296)），交给 checker goroutine 判定是否需要传输
- **文件 vs 目录**或**目录 vs 文件**：报错，无法覆盖
- **目录 vs 目录**：复制目录元数据/ModTime，返回 `recurse=true` 递归子目录

### 1.3 NeedTransfer——文件级差异判定

[NeedTransfer](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/operations/operations.go#L1733-L1801) 由 checker goroutine 调用，判断源文件是否需要传输到目的端。判定优先级：

1. **dst == nil** → 需要传输（目的端不存在）
2. **--ignore-existing** → 跳过
3. **--ignore-times** → 强制传输
4. **--update-older** → 目的端更新则跳过；源端更新时用 `equal()` 判定
5. **常规路径** → 调用 `Equal`

`equal()` 的判定逻辑（[operations.go#L246-L359](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/operations/operations.go#L246-L359)：

```
size不同 → 不相等
--size-only → 仅比较大小
--checksum → 比较哈希
mtime在modifyWindow内 → 相等
mtime不同 → 比较哈希:
  哈希相同 → 更新dst的mtime(如果支持)，视为相等
  哈希不可用 → 视为不相等(除非--refresh-times)
```

---

## 二、传输队列：并行管道架构（纠正：不是依次串联！

### 2.1 pipe——带优先级排序的无界管道

[pipe](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/pipe.go#L22-L31) 是同步引擎的核心数据结构，提供带背压的无界缓冲通道：

- **内部存储**：`queue []fs.ObjectPair` 切片 + 信号通道 `c chan struct{}`
- **排序支持**：使用 `deheap`（双端堆）实现 `--order-by` 排序，支持 `name`/`size`/`modtime` 三种排序维度，以及 `mixed` 分数模式（一部分取最小端、一部分取最大端）
- **背压控制**：`Put` 时向 `c` 发送信号，`Get` 时从 `c` 接收；`c` 的容量由 `--max-backlog` 控制
- **统计回调**：每次 Put/Get 更新队列项数和总大小，实时反映到 `stats`

### 2.2 真实管道架构（对照代码验证）

三个 pipe 的关系是**并行汇聚**到 `toBeUploaded`，**不是依次串联**！

```
                          ┌──────────────┐
                          │ pairChecker │   toBeChecked 的唯一消费者
                          └──────────────┘
                                 │
                                 ▼
toBeChecked ────────────────────────────┐
  (检查管道)                     │
  唯一入口: Match 回调            │
  [sync.go#L1296]                ▼
                              toBeUploaded ──────→ pairCopyOrMove
                                 ▲                    (传输管道)
                                 │                    消费者: transfer goroutines
                                 │
                                 │
                          ┌──────────────┐
                          │ pairRenamer │   toBeRenamed 的消费者
                          └──────────────┘
                                 ▲
                                 │
toBeRenamed ─────────────────────┘
  (重命名管道)
  唯一入口: run() 中 track-renames 路径
  [sync.go#L973]
  (March 结束后批量填充)
```

**关键代码证据**（grep 结果）：

| Pipe | Put 位置 | 说明 |
|------|---------|------|
| `toBeChecked.Put` | [sync.go#L1296](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1296) | 仅 Match 回调，唯一入口 |
| `toBeUploaded.Put` | [sync.go#L1263](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1263) | SrcOnly (trackRenames=false) |
| `toBeUploaded.Put` | [sync.go#L438](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L438), [L444](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L444), [L462](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L462) | pairChecker 输出 |
| `toBeUploaded.Put` | [sync.go#L491](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L491) | pairRenamer 输出（失败时） |
| `toBeRenamed.Put` | [sync.go#L973](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L973) | 仅 run() 中 track-renames 路径 |

### 2.3 Goroutine 启停时序

在 [run()](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L936-L1039) 中的启动顺序：

```go
s.startCheckers()     // 启动 checker goroutines（消费 toBeChecked，输出到 toBeUploaded）
s.startRenamers()     // 启动 renamer goroutines（仅 trackRenames，消费 toBeRenamed，失败时输出到 toBeUploaded）
if !s.checkFirst {
    s.startTransfers() // 启动 transfer goroutines（消费 toBeUploaded）
}
s.startDeleters()     // 启动 deleter goroutines（DeleteModeDuring）
s.startTrackRenames() // 启动 trackRenames 收集器（消费 trackRenamesCh）
m.Run(s.ctx)          // March 遍历开始，向管道填充数据
```

停止顺序（[sync.go#L967-L988](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L967-L988)）：

```go
s.stopTrackRenames()  // 关闭 trackRenamesCh，等待收集完成
if s.trackRenames {
    s.makeRenameMap()
    // 将 renameCheck 中的源文件批量送入 toBeRenamed
}
s.stopCheckers()      // 关闭 toBeChecked，等待 checker 完成
if s.checkFirst {
    s.startTransfers() // --check-first 模式下此时才启动传输
}
s.stopRenamers()      // 关闭 toBeRenamed，等待 renamer 完成
s.stopTransfers()    // 关闭 toBeUploaded，等待 transfer 完成
s.stopDeleters()    // 关闭 deleteFilesCh，等待 deleter 完成
```

**关键时序保证**：

1. `toBeChecked` → checker → `toBeUploaded` 是**直接串联**，checker 关闭后 toBeChecked 不会再有数据
2. `toBeRenamed` → renamer → `toBeUploaded` 是**独立并行路径**，renamer 关闭后 toBeRenamed 不会再有数据
3. `toBeUploaded` 是最终汇聚点，最后关闭

### 2.4 pairChecker 内部逻辑

[pairChecker](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L371-L476) 是检查管道的消费者：

```
从 toBeChecked 取出 {Src, Dst}
  │
  ├─ NeedTransfer() == false → 不需传输
  │    └─ DoMove? → 删除源端文件（或通过 toBeUploaded 有序删除）
  │
  └─ NeedTransfer() == true → 需要传输
       ├─ CompareOrCopyDest() → 可能免除传输
       ├─ --fix-case → 尝试重命名
       ├─ --immutable → 报错
       ├─ backupDir != nil && Dst != nil → 先备份再传输
       └─ 放入 toBeUploaded（[sync.go#L438/L444/L462](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L438)
```

### 2.5 pairRenamer 内部逻辑

[pairRenamer](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L480-L497) 是重命名管道的消费者：

```go
func (s *syncCopyMove) pairRenamer(in *pipe, out *pipe, ...) {
    for {
        pair, ok := in.GetMax(s.inCtx, fraction)
        src := pair.Src
        if !s.tryRename(src) {
            // 重命名失败，送入 toBeUploaded 正常传输
            ok = out.Put(s.inCtx, pair)
        }
        // 重命名成功，不进入 toBeUploaded
    }
}
```

---

## 三、track-renames：启用、降级、删除策略边界

### 3.1 启用条件与降级逻辑

在 [newSyncCopyMove](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L240-L270) 中，`trackRenames` 有严格的启用和降级检查：

```go
if s.trackRenames {
    // 降级条件 1：目的端不支持服务端 Move/Copy
    if !operations.CanServerSideMove(fdst) {
        fs.Errorf(fdst, "Ignoring --track-renames as the destination does not support server-side move or copy")
        s.trackRenames = false
    }
    // 降级条件 2：hash 策略且无共同 hash
    if s.trackRenamesStrategy.hash() && s.commonHash == hash.None {
        fs.Errorf(fdst, "Ignoring --track-renames as the source and destination do not have a common hash")
        s.trackRenames = false
    }
    // 降级条件 3：modtime 策略且不支持 modtime
    if s.trackRenamesStrategy.modTime() && s.modifyWindow == fs.ModTimeNotSupported {
        fs.Errorf(fdst, "Ignoring --track-renames as either the source or destination do not support modtime")
        s.trackRenames = false
    }
    // 降级条件 4：DeleteModeOff（copy/move 模式）
    if s.deleteMode == fs.DeleteModeOff {
        fs.Errorf(fdst, "Ignoring --track-renames as it doesn't work with copy or move, only sync")
        s.trackRenames = false
    }
}
if s.trackRenames {
    // 强制约束 1：必须使用 DeleteModeAfter
    if s.deleteMode != fs.DeleteModeOff {
        s.deleteMode = fs.DeleteModeAfter
    }
    // 强制约束 2：禁用 noTraverse
    if s.noTraverse {
        fs.Errorf(nil, "Ignoring --no-traverse with --track-renames")
        s.noTraverse = false
    }
}
```

### 3.2 与删除模式的交互边界

在 [runSyncCopyMove](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1358-L1373) 中，`DeleteModeBefore` 与 `track-renames 的不兼容检测：

```go
if deleteMode == fs.DeleteModeBefore && ci.TrackRenames {
    return nil, errors.New("can't use --track-renames with --delete-before")
}
if deleteMode == fs.DeleteModeBefore {
    // 第一遍：DeleteModeOnly，仅删除
    do, _ := newSyncCopyMove(ctx, fdst, fsrc, fs.DeleteModeOnly, ...)
    do.run()
    // 第二遍：DeleteModeOff，仅复制
    deleteMode = fs.DeleteModeOff
}
```

**边界总结**：

| 删除模式 | 与 track-renames 兼容性 | 原因 |
|----------|------------------------|------|
| `DeleteModeBefore` | ❌ 不兼容 | 两遍扫描第一遍是 DeleteModeOnly，会删除所有待重命名的目的端文件 |
| `DeleteModeDuring` | ❌ 不兼容（被强制降级为 After） | 边传边删可能删除待重命名文件 |
| `DeleteModeAfter` | ✅ 兼容（强制启用） | 传输完成后才删除，确保重命名检测完整 |
| `DeleteModeOff` | ❌ 不兼容（copy/move 模式） | 不删除目的端，重命名无意义 |

### 3.3 track-renames 完整数据流

```
SrcOnly (trackRenames=true)
    │
    ▼
trackRenamesCh ──→ startTrackRenames 收集到 renameCheck 切片
    │                      ([sync.go#L586-L590](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L586-L590)
    │
    ▼
March.Run() 结束
    │
    ▼
stopTrackRenames() 关闭 trackRenamesCh
    │
    ▼
makeRenameMap() 为 dstFiles 中 size 匹配的对象计算 hash
    │                      ([sync.go#L855-L892](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L855-L892)
    │
    ▼
遍历 renameCheck，逐个送入 toBeRenamed
    │                      ([sync.go#L970-L977](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L970-L977)
    │
    ▼
pairRenamer 消费 toBeRenamed
    │
    ├─ tryRename() 成功 → 服务端 Move 重命名
    │                        从 dstFiles 中移除，不进入 toBeUploaded
    │
    └─ tryRename() 失败 → 送入 toBeUploaded 正常传输
                              ([sync.go#L491](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L491)
```

### 3.4 tryRename 内部逻辑

[tryRename](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L896-L927)：

```go
func (s *syncCopyMove) tryRename(src fs.Object) bool {
    // 1. 计算 renameID：size + [hash] + [modtime] + [leaf]
    id := s.renameID(src, s.trackRenamesStrategy, s.modifyWindow)
    if id == "" {
        return false
    }
    // 2. 从 renameMap 中查找匹配
    s.renameMapMu.Lock()
    candidates := s.renameMap[id]
    if len(candidates) == 0 {
        s.renameMapMu.Unlock()
        return false
    }
    dst := candidates[0]
    s.renameMap[id] = candidates[1:]
    s.renameMapMu.Unlock()
    // 3. 如果有 modtime 策略，检查 modtime 是否在 modifyWindow 内
    if s.trackRenamesStrategy.modTime() {
        if !fs.ModTimeEqual(src.ModTime(s.ctx), dst.ModTime(s.ctx), s.modifyWindow) {
            return false
        }
    }
    // 4. 服务端 Move 重命名
    _, err := operations.Move(s.ctx, s.fdst, nil, src.Remote(), dst)
    if err != nil {
        return false
    }
    // 5. 从 dstFiles 中移除，避免被删除
    delete(s.dstFiles, dst.Remote())
    return true
}
```

### 3.5 renameID 构造

[renameID](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L775-L804) 根据 `--track-renames-strategy` 构造唯一标识：

```
size,hash                // hash 策略
size,modtime          // modtime 策略（modtime 在 popRenameMap 中检查
size,leaf              // leaf 策略
size,hash,leaf         // hash+leaf 组合策略
```

---

## 四、删除策略：五种模式 + 边界守卫

### 4.1 DeleteMode 定义

[deletemode.go](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/deletemode.go#L4-L14) 定义了五种删除模式：

| 模式 | 值 | 含义 |
|------|---|------|
| `DeleteModeOff` | 0 | 不删除（copy 模式） |
| `DeleteModeBefore` | 1 | 传输前先删除（两遍扫描） |
| `DeleteModeDuring` | 2 | 传输时边传边删 |
| `DeleteModeAfter` | 3 | 传输完成后删除（默认） |
| `DeleteModeOnly` | 4 | 仅删除，不传输（DeleteModeBefore 的第一遍） |

### 4.2 DeleteModeBefore——两遍扫描策略

[runSyncCopyMove](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1352-L1379) 中处理 `DeleteModeBefore`：

```go
if deleteMode == fs.DeleteModeBefore {
    // 第一遍：DeleteModeOnly，仅删除目的端多余文件
    do, _ := newSyncCopyMove(ctx, fdst, fsrc, fs.DeleteModeOnly, ...)
    do.run()
    // 第二遍：DeleteModeOff，仅复制
    deleteMode = fs.DeleteModeOff
}
do, _ := newSyncCopyMove(ctx, fdst, fsrc, deleteMode, ...)
return do.run()
```

两遍扫描的优点：在复制之前清理空间，避免目的端空间不足；缺点是需要两遍完整的目录遍历。

### 4.3 DeleteModeDuring——边传边删

`DstOnly` 回调中对文件直接发送到 `deleteFilesCh`（[sync.go#L1067-L1073](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1067-L1073)：

```go
case fs.DeleteModeDuring, fs.DeleteModeOnly:
    select {
    case <-s.ctx.Done():
        return
    case s.deleteFilesCh <- x:
    }
```

后台 [deleter goroutine](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L603-L611) 消费 `deleteFilesCh`，调用 `DeleteFilesWithBackupDir` 执行删除。

### 4.4 DeleteModeAfter——默认策略

`DstOnly` 回调中将对象存入 `dstFiles` map（[sync.go#L1062-L1066](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1062-L1066)），在 `run()` 的末尾阶段批量调用 [deleteFiles](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L627-L666)：

```go
if s.deleteMode == fs.DeleteModeAfter {
    s.processError(s.deleteFiles(false))
}
```

`deleteFiles` 遍历 `dstFiles` map，将对象送入 `toDelete` 通道，由 `ci.Checkers` 个 goroutine 并发执行 `DeleteFileWithBackupDir`。

### 4.5 空目录清理

在文件删除之后，[deleteEmptyDirectories](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L670-L710) 按路径长度从长到短（最深目录优先）删除空目录：

```go
sort.Sort(entries)        // 按路径排序
for i := len(entries)-1; i >= 0; i-- {
    err := operations.TryRmdir(ctx, f, dir.Remote())
}
```

### 4.6 backup-dir 保护

如果设置了 `--backup-dir`，所有删除操作不会真正删除，而是移动到备份目录。这由 [DeleteFileWithBackupDir](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/operations/operations.go#L550-L578) 实现：

```go
if backupDir != nil {
    err = MoveBackupDir(ctx, backupDir, dst)
} else {
    err = dst.Remove(ctx)
}
```

### 4.7 错误安全守卫

删除操作受错误状态保护（[sync.go#L628-L641](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L628-L641)）：

```go
if accounting.Stats(s.ctx).Errored() && !s.ci.IgnoreErrors {
    fs.Errorf(s.fdst, "%v", fs.ErrorNotDeleting)
    return fs.ErrorNotDeleting
}
```

如果同步过程中有任何错误且未设置 `--ignore-errors`，则**放弃所有删除操作**，防止在不确定状态下丢失数据。

---

## 五、三者协同：完整数据流图（对照代码验证）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          syncCopyMove.run()                        │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    March.Run() — 目录遍历                         │  │
│  │                                                                │  │
│  │   fsrc 目录 ──┐                                           │  │
│  │                   ├── matchListings() ──→ 三路分发             │  │
│  │   fdst 目录 ──┘                                           │  │
│  │                                                                │  │
│  │   SrcOnly (文件)                                            │  │
│  │     │                                                          │  │
│  │     ├─ trackRenames=true  ──→ trackRenamesCh ──→ renameCheck │  │
│  │     │                          (收集，不立即处理)                 │  │
│  │     │                                                          │  │
│  │     └─ trackRenames=false ──→ CompareOrCopyDest ──→ toBeUploaded │  │
│  │                                                             │  │
│  │   DstOnly (文件)                                            │  │
│  │     │                                                          │  │
│  │     ├─ DeleteModeAfter  ──→ dstFiles map (留待后删)         │  │
│  │     ├─ DeleteModeDuring ──→ deleteFilesCh → deleter goroutine │  │
│  │     └─ DeleteModeOnly   ──→ deleteFilesCh → deleter goroutine │  │
│  │                                                                │  │
│  │   Match (文件vs文件) ──→ toBeChecked ──→ pairChecker ──→ toBeUploaded │  │
│  │                                                                │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────── 差异计算 & 传输队列 ────────────┐                │
│  │                                                     │                │
│  │  toBeChecked ──→ pairChecker ──┐            │                │
│  │                    │                  │            │                │
│  │                    ├─ NeedTransfer() 判定            │                │
│  │                    ├─ CompareOrCopyDest() 免除       │            │                │
│  │                    └─ backupDir 备份后传输 ────┘            │                │
│  │                                                     │                │
│  │  [trackRenames 路径 — 与上面并行！]                   │                │
│  │  trackRenamesCh ──→ renameCheck ──(March结束)──→ makeRenameMap │
│  │                                                     │                │
│  │  makeRenameMap: 为 dstFiles 中与 src size 匹配的对象计算 hash │  │
│  │    构建 renameMap: renameID → []dstObject            │                │
│  │                                                     │                │
│  │  renameCheck ──→ toBeRenamed ──→ pairRenamer ──┐            │
│  │                                                     │            │
│  │                                                     ├─ tryRename 成功 → 服务端重命名，从 dstFiles 移除 │  │
│  │                                                     └─ tryRename 失败 → toBeUploaded │  │
│  │                                                     │                │
│  │  toBeUploaded ──→ pairCopyOrMove ──→ Copy/Move │                │
│  │    (汇聚点：SrcOnly + pairChecker + pairRenamer)  │                │
│  │                                                     │                │
│  │  --order-by 排序: name/size/modtime + mixed 分数    │                │
│  │  --check-first: 先完成所有检查再启动传输             │                │
│  └─────────────────────────────────────────────────────┘                │
│                                                                     │
│  ┌──────────── 删除策略 ────────────┐                │
│  │                                                     │                │
│  │  DeleteModeBefore: 两遍扫描（先删后传）              │                │
│  │    第一遍 DeleteModeOnly → 仅删除                  │                │
│  │    第二遍 DeleteModeOff → 仅复制                    │                │
│  │    ❌ 不兼容 track-renames                          │                │
│  │                                                     │                │
│  │  DeleteModeDuring: deleteFilesCh → deleter goroutine│                │
│  │    边传边删，节省时间但有风险                        │                │
│  │                                                     │                │
│  │  DeleteModeAfter: dstFiles map → deleteFiles() │                │
│  │    ✅ 兼容 track-renames（强制启用）                  │                │
│  │    最安全，传输完成后才删除                          │                │
│  │                                                     │                │
│  │  删除后 → deleteEmptyDirectories() (最深目录优先)    │                │
│  │  错误守卫 → 有错误时不执行删除                       │                │
│  │  backup-dir → 删除变更为移动到备份目录               │                │
│  └─────────────────────────────────────────────────────┘                │
│                                                                     │
│  最终阶段:                                                          │
│    1. 设置目录 ModTime (setDirModTimeAfter)                         │
│    2. 清理空目录 (dstEmptyDirs)                                   │
│    3. 清理源端空目录 (DoMove + deleteEmptySrcDirs)                 │
│    4. 检查 max-duration 超时                                        │
│    5. track-renames 强制 DeleteModeAfter（确保重命名检测完整）          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 六、关键设计洞察

### 6.1 并行汇聚架构，不是线性流水线

**关键纠正**：之前的 "三级管道依次串联" 是错误的。真实架构是**两路并行汇聚到 `toBeUploaded`：

- **路径 1**：Match → toBeChecked → pairChecker → toBeUploaded
- **路径 2**：SrcOnly (trackRenames=false) → toBeUploaded
- **路径 3**：SrcOnly (trackRenames=true) → trackRenamesCh → toBeRenamed → pairRenamer → (失败时) toBeUploaded

这种架构的优势：
- SrcOnly 不需要经过检查队列，减少不必要的 hash 计算
- track-renames 是可选的独立路径，不影响正常传输路径
- toBeUploaded 作为统一的传输调度入口，统一应用 --order-by 排序

### 6.2 track-renames 的两阶段设计

track-renames 采用"先收集、后处理的两阶段设计：

1. **收集阶段**（March 遍历期间，SrcOnly 文件存入 trackRenamesCh，仅收集不处理
2. **处理阶段**（March 结束后），构建 renameMap，逐个尝试重命名

这样设计的原因：
- 重命名检测需要完整的 dstFiles map，必须等遍历完成
- 避免在遍历期间进行耗时的 hash 计算影响遍历性能
- 强制 DeleteModeAfter 确保 dstFiles 中的对象不会被提前删除

### 6.3 删除策略的安全优先设计

- **DeleteModeAfter（默认）**：最安全，确保传输完成后才删除，任何传输错误都会阻止删除
- **DeleteModeBefore**：最激进，先删后传可能因空间不足导致失败
- **DeleteModeDuring**：折中方案，删除和传输并行，节省时间但有风险
- 错误守卫机制（`Errored() && !IgnoreErrors`）在所有模式下都保护删除操作
- track-renames 强制 DeleteModeAfter 进一步增强安全性

### 6.4 pipe 的 mixed 分数调度

`--order-by size,mixed,75` 这类配置让 75% 的 worker 从堆顶取最小文件、25% 从堆底取最大文件，实现了大小文件混合调度，避免小文件被大文件阻塞。

### 6.5 双重上下文控制

`syncCopyMove` 使用两个 context：
- `s.ctx`：主上下文，硬超时或致命错误时取消
- `s.inCtx`：输入上下文，优雅超时时仅停止 March 遍历和管道填充，允许已入队的传输完成
