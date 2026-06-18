# rclone Sync 同步流程深度追踪

## 总览

rclone 的 `sync` 命令将源端（fsrc）的内容同步到目的端（fdst），使目的端与源端完全一致。核心实现在 [sync.go](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go) 中，由 `syncCopyMove` 结构体驱动，三个核心机制——**差异计算**、**传输队列**、**删除策略**——协同推进整个同步流程。

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

`syncCopyMove` 实现了 [Marcher](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/march/march.go#L52-L59) 接口的三个方法，March 的匹配结果直接路由到不同的同步动作：

#### SrcOnly（[sync.go#L1236-L1282](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1236-L1282)）

源端独有的条目：

- **文件**：若启用 `trackRenames`，送入 `trackRenamesCh` 等待后续重命名检查；否则直接执行 `CompareOrCopyDest` 检查后放入 `toBeUploaded` 管道
- **目录**：标记为非空、复制目录元数据/ModTime、返回 `recurse=true` 继续递归

#### DstOnly（[sync.go#L1041-L1091](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1041-L1091)）

目的端独有的条目：

- **文件**：根据 `deleteMode` 决定如何处理：
  - `DeleteModeAfter`：将对象加入 `dstFiles` map，留待同步结束后批量删除
  - `DeleteModeDuring`/`DeleteModeOnly`：直接送入 `deleteFilesCh`，由后台 deleter goroutine 即时删除
- **目录**：记录到 `dstEmptyDirs`，返回 `recurse=true` 继续递归（以便遍历子内容以供删除）

#### Match（[sync.go#L1285-L1343](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1285-L1343)）

两端都存在的条目：

- **文件 vs 文件**：构造 `ObjectPair{Src, Dst}` 放入 `toBeChecked` 管道，交给 checker goroutine 判定是否需要传输
- **文件 vs 目录**或**目录 vs 文件**：报错，无法覆盖
- **目录 vs 目录**：复制目录元数据/ModTime，返回 `recurse=true` 递归子目录

### 1.3 NeedTransfer——文件级差异判定

[NeedTransfer](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/operations/operations.go#L1733-L1801) 由 checker goroutine 调用，判断源文件是否需要传输到目的端。判定优先级：

1. **dst == nil** → 需要传输（目的端不存在）
2. **--ignore-existing** → 跳过
3. **--ignore-times** → 强制传输
4. **--update-older** → 目的端更新则跳过；源端更新时用 `equal()` 判定
5. **常规路径** → 调用 [Equal](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/operations/operations.go#L142-L144)

`equal()` 的判定逻辑（[operations.go#L246-L359](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/operations/operations.go#L246-L359)）：

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

## 二、传输队列：三级管道并行流水线

### 2.1 pipe——带优先级排序的无界管道

[pipe](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/pipe.go#L22-L31) 是同步引擎的核心数据结构，提供带背压的无界缓冲通道：

- **内部存储**：`queue []fs.ObjectPair` 切片 + 信号通道 `c chan struct{}`
- **排序支持**：使用 `deheap`（双端堆）实现 `--order-by` 排序，支持 `name`/`size`/`modtime` 三种排序维度，以及 `mixed` 分数模式（一部分取最小端、一部分取最大端）
- **背压控制**：`Put` 时向 `c` 发送信号，`Get` 时从 `c` 接收；`c` 的容量由 `--max-backlog` 控制
- **统计回调**：每次 Put/Get 更新队列项数和总大小，实时反映到 `stats`

### 2.2 三级管道架构

`syncCopyMove` 使用三个 pipe 构成流水线：

```
                    pairChecker              pairRenamer            pairCopyOrMove
                   ┌──────────┐            ┌──────────┐          ┌──────────────┐
toBeChecked ──────→│checker   │──────────→ │renamer   │─────────→│transfer      │
  (check pipe)     │goroutine │            │goroutine │          │goroutine     │
                   └──────────┘            └──────────┘          └──────────────┘
                        │                       ↑                       ↑
                   toBeUploaded            toBeRenamed             toBeUploaded
                  (transfer pipe)        (rename pipe)           (transfer pipe)
```

1. **toBeChecked**（检查管道）：存放 `Match` 产生的 `{Src, Dst}` 对，由 `ci.Checkers` 个 checker goroutine 消费
2. **toBeRenamed**（重命名管道）：仅 `--track-renames` 时启用，存放待尝试重命名的源文件
3. **toBeUploaded**（传输管道）：存放需要传输的文件对，由 `ci.Transfers` 个 transfer goroutine 消费

### 2.3 Goroutine 启停时序

在 [run()](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L936-L1039) 中的启动顺序：

```go
s.startCheckers()     // 启动 checker goroutines
s.startRenamers()     // 启动 renamer goroutines（仅 trackRenames）
if !s.checkFirst {
    s.startTransfers() // 启动 transfer goroutines
}
s.startDeleters()     // 启动 deleter goroutines（DeleteModeDuring）
s.startTrackRenames() // 启动 trackRenames 收集器
m.Run(s.ctx)          // March 遍历开始，向管道填充数据
```

停止顺序：

```go
s.stopTrackRenames()  // 停止收集，触发 makeRenameMap + 填充 toBeRenamed
s.stopCheckers()      // 关闭 toBeChecked，等待 checker 完成
s.stopRenamers()      // 关闭 toBeRenamed，等待 renamer 完成
s.stopTransfers()     // 关闭 toBeUploaded，等待 transfer 完成
s.stopDeleters()      // 关闭 deleteFilesCh，等待 deleter 完成
```

关键时序保证：**checker 先于 transfer 停止**，确保所有检查完成后再停止传输。`--check-first` 模式更是让所有检查完成后才启动传输。

### 2.4 pairChecker 内部逻辑

[pairChecker](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L371-L476) 是检查管道的消费者：

```
从 toBeChecked 取出 {Src, Dst}
  │
  ├─ NeedTransfer() == false → 不需传输
  │    └─ DoMove? → 删除源端文件
  │
  └─ NeedTransfer() == true → 需要传输
       ├─ CompareOrCopyDest() → 可能免除传输
       ├─ --fix-case → 尝试重命名
       ├─ --immutable → 报错
       ├─ backupDir != nil && Dst != nil → 先备份再传输
       └─ 放入 toBeUploaded
```

---

## 三、删除策略：三种时机 + backup-dir 保护

### 3.1 DeleteMode 定义

[deletemode.go](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/deletemode.go#L4-L14) 定义了五种删除模式：

| 模式 | 值 | 含义 |
|------|---|------|
| `DeleteModeOff` | 0 | 不删除（copy 模式） |
| `DeleteModeBefore` | 1 | 传输前先删除（两遍扫描） |
| `DeleteModeDuring` | 2 | 传输时边传边删 |
| `DeleteModeAfter` | 3 | 传输完成后删除（默认） |
| `DeleteModeOnly` | 4 | 仅删除，不传输（DeleteModeBefore 的第一遍） |

### 3.2 DeleteModeBefore——两遍扫描策略

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

### 3.3 DeleteModeDuring——边传边删

`DstOnly` 回调中对文件直接发送到 `deleteFilesCh`（[sync.go#L1067-L1073](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1067-L1073)）：

```go
case fs.DeleteModeDuring, fs.DeleteModeOnly:
    select {
    case <-s.ctx.Done():
        return
    case s.deleteFilesCh <- x:
    }
```

后台 [deleter goroutine](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L603-L611) 消费 `deleteFilesCh`，调用 `DeleteFilesWithBackupDir` 执行删除。

### 3.4 DeleteModeAfter——默认策略

`DstOnly` 回调中将对象存入 `dstFiles` map（[sync.go#L1062-L1066](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L1062-L1066)），在 `run()` 的末尾阶段批量调用 [deleteFiles](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L627-L666)：

```go
if s.deleteMode == fs.DeleteModeAfter {
    s.processError(s.deleteFiles(false))
}
```

`deleteFiles` 遍历 `dstFiles` map，将对象送入 `toDelete` 通道，由 `ci.Checkers` 个 goroutine 并发执行 `DeleteFileWithBackupDir`。

### 3.5 空目录清理

在文件删除之后，[deleteEmptyDirectories](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L670-L710) 按路径长度从长到短（最深目录优先）删除空目录：

```go
sort.Sort(entries)        // 按路径排序
for i := len(entries)-1; i >= 0; i-- {
    err := operations.TryRmdir(ctx, f, dir.Remote())
}
```

### 3.6 backup-dir 保护

如果设置了 `--backup-dir`，所有删除操作不会真正删除，而是移动到备份目录。这由 [DeleteFileWithBackupDir](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/operations/operations.go#L550-L578) 实现：

```go
if backupDir != nil {
    err = MoveBackupDir(ctx, backupDir, dst)
} else {
    err = dst.Remove(ctx)
}
```

### 3.7 错误安全守卫

删除操作受错误状态保护（[sync.go#L628-L641](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L628-L641)）：

```go
if accounting.Stats(s.ctx).Errored() && !s.ci.IgnoreErrors {
    fs.Errorf(s.fdst, "%v", fs.ErrorNotDeleting)
    return fs.ErrorNotDeleting
}
```

如果同步过程中有任何错误且未设置 `--ignore-errors`，则**放弃所有删除操作**，防止在不确定状态下丢失数据。

---

## 四、三者协同：完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          syncCopyMove.run()                            │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    March.Run() — 目录遍历                         │  │
│  │                                                                    │  │
│  │   fsrc 目录 ──┐                                                   │  │
│  │               ├── matchListings() ──→ 三路分发                     │  │
│  │   fdst 目录 ──┘                                                   │  │
│  │                                                                    │  │
│  │   SrcOnly ──────────────────────────────────────┐                 │  │
│  │     文件 → toBeUploaded (或 trackRenamesCh)      │                │  │
│  │     目录 → 递归                                   │                │  │
│  │                                                    │                │  │
│  │   DstOnly ──────────────────────────────────────┐ │                │  │
│  │     文件 → dstFiles map / deleteFilesCh  ◄──────┤│ 删除策略        │  │
│  │     目录 → dstEmptyDirs + 递归           ◄──────┘│                │  │
│  │                                                    │                │  │
│  │   Match ────────────────────────────────────────┐ │                │  │
│  │     文件 vs 文件 → toBeChecked  ◄───────────────┘ │                │  │
│  │     目录 vs 目录 → copyDirMetadata + 递归          │                │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌───────────────────── 差异计算 ─────────────────────┐                │
│  │                                                     │                │
│  │  toBeChecked ──→ pairChecker ──→ toBeUploaded      │                │
│  │                    │                                │                │
│  │                    ├─ NeedTransfer() 判定            │                │
│  │                    ├─ CompareOrCopyDest() 免除       │                │
│  │                    └─ backupDir 备份后传输           │                │
│  │                                                     │                │
│  │  [trackRenames]                                    │                │
│  │  trackRenamesCh ──→ makeRenameMap ──→ toBeRenamed  │                │
│  │                                          │          │                │
│  │                       pairRenamer ───────┘          │                │
│  │                          │ 成功 → 服务端重命名      │                │
│  │                          │ 失败 → toBeUploaded      │                │
│  └─────────────────────────────────────────────────────┘                │
│                                                                         │
│  ┌───────────────────── 传输队列 ─────────────────────┐                │
│  │                                                     │                │
│  │  toBeUploaded ──→ pairCopyOrMove ──→ Copy/Move     │                │
│  │     (pipe)       (ci.Transfers 个 goroutine)        │                │
│  │                                                     │                │
│  │  --order-by 排序: name/size/modtime + mixed 分数    │                │
│  │  --check-first: 先完成所有检查再启动传输             │                │
│  └─────────────────────────────────────────────────────┘                │
│                                                                         │
│  ┌───────────────────── 删除策略 ─────────────────────┐                │
│  │                                                     │                │
│  │  DeleteModeBefore: 两遍扫描（先删后传）              │                │
│  │  DeleteModeDuring: deleteFilesCh → deleter goroutine│                │
│  │  DeleteModeAfter:  dstFiles map → deleteFiles()     │                │
│  │                                                     │                │
│  │  删除后 → deleteEmptyDirectories() (最深目录优先)    │                │
│  │  错误守卫 → 有错误时不执行删除                       │                │
│  │  backup-dir → 删除变更为移动到备份目录               │                │
│  └─────────────────────────────────────────────────────┘                │
│                                                                         │
│  trackRenames 强制 DeleteModeAfter（确保重命名检测完整）                  │
│                                                                         │
│  最终阶段:                                                              │
│    1. 设置目录 ModTime (setDirModTimeAfter)                             │
│    2. 清理空目录 (dstEmptyDirs)                                         │
│    3. 清理源端空目录 (DoMove + deleteEmptySrcDirs → srcMoveEmptyDirs)   │
│    4. 检查 max-duration 超时                                            │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 五、trackRenames——差异计算的增强

当启用 `--track-renames` 时，差异计算增加了一个重命名检测层：

1. `SrcOnly` 中的文件不直接送入传输队列，而是先放入 `trackRenamesCh`
2. March 遍历结束后，[makeRenameMap](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L855-L892) 为 `dstFiles` 中与源端文件大小匹配的对象计算哈希，构建 `renameMap`
3. [pairRenamer](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L480-L497) 对每个待检查文件调用 [tryRename](file:///d:/fz/0601-2/solo-dogfeeding/code/55-rclone/fs/sync/sync.go#L896-L927)：
   - 计算源文件的 `renameID`（size + hash + 可选的 leaf name）
   - 从 `renameMap` 中查找匹配的目的端对象
   - 匹配成功 → 服务端 Move 重命名，从 `dstFiles` 中移除
   - 匹配失败 → 送入 `toBeUploaded` 正常传输

`trackRenames` 强制将删除模式切换为 `DeleteModeAfter`，确保重命名检测能基于完整的 `dstFiles` map 运行。

---

## 六、关键设计洞察

### 6.1 流水线并行度

整个同步流程是一个**多阶段流水线**：March 产生数据 → Checker 消费并转发 → Transfer 消费。三个阶段通过 pipe 连接，各自有独立的并发度（`--checkers`、`--transfers`），实现了 CPU 密集（哈希计算）和 I/O 密集（网络传输）的并行化。

### 6.2 删除策略的安全优先

- **DeleteModeAfter（默认）**：最安全，确保传输完成后才删除，任何传输错误都会阻止删除
- **DeleteModeBefore**：最激进，先删后传可能因空间不足导致失败
- **DeleteModeDuring**：折中方案，删除和传输并行，节省时间但有风险
- 错误守卫机制（`Errored() && !IgnoreErrors`）在所有模式下都保护删除操作

### 6.3 pipe 的 mixed 分数调度

`--order-by size,mixed,75` 这类配置让 75% 的 worker 从堆顶取最小文件、25% 从堆底取最大文件，实现了大小文件混合调度，避免小文件被大文件阻塞。

### 6.4 双重上下文控制

`syncCopyMove` 使用两个 context：
- `s.ctx`：主上下文，硬超时或致命错误时取消
- `s.inCtx`：输入上下文，优雅超时时仅停止 March 遍历和管道填充，允许已入队的传输完成
