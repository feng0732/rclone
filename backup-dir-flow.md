# 备份目录（backup-dir）处理流程详解

本文档梳理 rclone 中 `--backup-dir` 和 `--suffix` 选项的处理逻辑，
说明在文件**删除**或**覆盖**前如何跳转到备份位置，以及与后端文件操作层之间的交互。

---

## 一、配置入口

### 1.1 选项定义

相关配置在 [fs/config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/config.go) 中声明：

| 配置项 | 结构体字段 | 行号 | 说明 |
|---|---|---|---|
| `--backup-dir` | `BackupDir string` | [L616](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/config.go#L616-L616) | 备份目录的远程路径 |
| `--suffix` | `Suffix string` | [L617](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/config.go#L617-L617) | 追加到被覆盖/删除文件名的后缀 |
| `--suffix-keep-extension` | `SuffixKeepExtension bool` | [L618](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/config.go#L618-L618) | 加后缀时保留扩展名（如 `a.bak.txt`） |

对应的 flag 注册位于 [L261-L274](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/config.go#L261-L274)，属于 `Sync` 组。

### 1.2 两种模式

备份功能有两种触发方式，可以单独或组合使用：

1. **`--backup-dir <DIR>`**：把被替换/删除的文件**移动**到另一个目录层级（保持相对路径结构）。
2. **`--suffix <SUF>`**：不换目录，只在**原位置**给被替换/删除的文件加后缀重命名。
3. **两者同时使用**：文件被移动到 `--backup-dir` 下，同时文件名还会加上 `--suffix`。

---

## 二、backupDir Fs 的构造与校验

### 2.1 构造时机

在同步流程初始化时，`newSyncCopyMove()` 负责创建 `backupDir`：

位置：[fs/sync/sync.go L271-L278](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/sync/sync.go#L271-L278)

```go
if ci.BackupDir != "" || ci.Suffix != "" {
    var err error
    s.backupDir, err = operations.BackupDir(ctx, fdst, fsrc, "")
    if err != nil {
        return nil, err
    }
}
```

### 2.2 BackupDir() 函数详解

核心函数位于 [fs/operations/operations.go L1916-L1952](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L1916-L1952)。

#### 流程分支：

**分支 A：设置了 `--backup-dir`**

```
ci.BackupDir != ""
    │
    ▼
cache.Get(ctx, ci.BackupDir)   // 通过缓存创建 backupDir 的 Fs 实例
    │
    ▼
SameConfig(fdst, backupDir)?   // 必须与目标同 remote（否则无法 server-side move）
    │ 否 → FatalError: "parameter to --backup-dir has to be on the same remote as destination"
    ▼ 是
OverlappingFilterCheck(backupDir, fdst)?  // 不能与 dst 目录重叠
    │ 是 → FatalError
    ▼ 否
OverlappingFilterCheck(backupDir, fsrc)?  // 不能与 src 目录重叠
    │ 是 → FatalError
    ▼ 否
SameDir(fdst, backupDir) / SameDir(fsrc, backupDir)?  // 不能与 src/dst 完全相同
    │ 是 → FatalError
    ▼ 否
继续最后一步检查
```

**分支 B：只设置了 `--suffix`（无 `--backup-dir`）**

```
ci.BackupDir == "" && ci.Suffix != ""
    │
    ▼
backupDir = fdst   // 直接复用目标 Fs（因为是同目录改名，不是跨目录移动）
```

#### 最后一道通用检查

```
CanServerSideMove(backupDir)?   // 后端必须支持 Move 或 Copy 特性
    │ 否 → FatalError: "can't use --backup-dir on a remote which doesn't support server-side move or copy"
    ▼ 是
return backupDir, nil
```

> **CanServerSideMove** 判断依据（[operations.go L526-L530](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L526-L530)）：
> - `fdst.Features().Move != nil`（原生服务端移动）
> - `fdst.Features().Copy != nil`（通过 copy+delete 模拟移动）

---

## 三、核心跳转函数：MoveBackupDir

位于 [fs/operations/operations.go L1954-L1960](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L1954-L1960)：

```go
func MoveBackupDir(ctx context.Context, backupDir fs.Fs, dst fs.Object) (err error) {
    remoteWithSuffix := SuffixName(ctx, dst.Remote())
    overwritten, _ := backupDir.NewObject(ctx, remoteWithSuffix)
    _, err = Move(ctx, backupDir, overwritten, remoteWithSuffix, dst)
    return err
}
```

### 3.1 三步跳转

| 步骤 | 说明 | 关键调用 |
|---|---|---|
| ① 生成目标名 | 根据 `--suffix` 计算备份文件名 | `SuffixName(ctx, dst.Remote())` |
| ② 定位目标位置 | 在 backupDir Fs 下查找同名对象（可能已存在） | `backupDir.NewObject(ctx, remoteWithSuffix)` |
| ③ 执行移动 | 调用通用 Move 函数将 dst 移动到 backupDir 下 | `Move(ctx, backupDir, overwritten, remoteWithSuffix, dst)` |

### 3.2 SuffixName 的命名逻辑

位于 [operations.go L532-L543](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L532-L543)：

```
Suffix 为空 → 原样返回 remote
Suffix 非空 + SuffixKeepExtension = true → a.txt → a.bak.txt （扩展名前插入）
Suffix 非空 + SuffixKeepExtension = false → a.txt → a.txt.bak （末尾追加）
```

### 3.3 Move() 内部的后端交互

Move 函数位于 [operations.go L433-L519](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L433-L519)，是备份跳转与后端交互的**关键桥梁**。

```
move(ctx, fdst=backupDir, dst=overwritten, remote=remoteWithSuffix, src=原dst文件)
    │
    ▼
是否具备 Server-Side Move 能力？
    │
    ├─ 是（backend.Features.Move 存在 + SameConfig 等条件满足）
    │      │
    │      ▼
    │   dst(overwritten) 存在吗？
    │      │ 是 → 先 DeleteFile(dst) 删掉旧备份（保证新备份能写进去）
    │      ▼ 否
    │   doMove(ctx, src, remote)        // ← 调用后端具体实现的 Move
    │      │
    │      ├─ 成功 → 返回，走 ServerSide 统计
    │      └─ 返回 ErrorCantMove → 降级走 copy 路径
    │
    └─ 否（或 Move 失败降级）
           │
           ▼
        Copy(ctx, backupDir, dst, origRemote, src)   // 先复制到备份位置
           │
           ▼
        DeleteFile(ctx, src)                          // 再删源文件（即原 dst）
```

> 关键：当 backend 不支持原生 Move 时，rclone 会自动退化为 **Copy + Delete** 两阶段，
> 效果等价于 Move，因此只要支持 Copy 也能使用 backup-dir。

---

## 四、场景一：文件覆盖（Overwrite）前的备份跳转

### 4.1 sync/copy 批量操作流程

发生在 `pairChecker`（检查阶段 worker）中，位于 [fs/sync/sync.go L371-L476](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/sync/sync.go#L371-L476)。

```
对每个 src/dst pair（由 march 生成）
    │
    ▼
operations.NeedTransfer(dst, src)?
    │ 否 → 跳过（不需要传输）
    ▼ 是
operations.CompareOrCopyDest(..., backupDir)?
    │ 命中 compare-dest/copy-dest 优化 → 内部也可能触发 MoveBackupDir（详见 4.2）
    ▼ 仍需传输
┌─ pair.Dst != nil 且 s.backupDir != nil ? ─┐
│  │ 是                                      │
│  ▼                                         │
│  operations.MoveBackupDir(s.ctx,           │
│      s.backupDir, pair.Dst)                │
│    │                                       │
│    ├─ 失败 → 记录错误，不继续传输          │
│    └─ 成功 → pair.Dst = nil                │
│           （告诉后续流程：目的地已清空）    │
│  │ 否 / 成功后                              │
│  ▼                                         │
└─ out.Put(pair) → 送入 toBeUploaded 管道 ──┘
                                    │
                                    ▼
                            pairCopyOrMove worker
                                    │
                                    ▼
                    operations.Copy(ctx, fdst, pair.Dst=nil, ...)
                                    │
                                    ▼
                           写新文件到目的地
```

关键代码行（[sync.go L429-L448](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/sync/sync.go#L429-L448)）：

```go
if pair.Dst != nil && s.backupDir != nil {
    err := operations.MoveBackupDir(s.ctx, s.backupDir, pair.Dst)
    if err != nil {
        // 失败：不继续
    } else {
        pair.Dst = nil   // ← 清空，这样 Copy 就以为是"新文件"
        out.Put(..., pair)
    }
}
```

### 4.2 CompareOrCopyDest 中的备份跳转

当启用 `--copy-dest`（服务器端复制优化）时，也可能触发备份，路径在：

[operations.go L1661-L1702](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L1661-L1702) 的 `copyDest()`：

```
copyDest 命中（src 与 copyDest 目录中的文件相等）
    │
    ▼
dst == nil 或 !Equal(src, dst)?    // 目标需要更新
    │
    ▼
dst != nil 且 backupDir != nil?
    │ 是
    ▼
MoveBackupDir(ctx, backupDir, dst)   // 先把旧目标挪走
    │
    ▼
dst = nil
    │
    ▼
Copy(ctx, fdst, dst=nil, remote, CopyDestFile)  // 从 copyDest 服务器端复制
```

### 4.3 单文件操作（moveOrCopyFile）

在 `moveOrCopyFile()`（[operations.go L2068-L2121](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L2068-L2121)）中，
对 copyto/moveto 等单文件命令，流程完全类似：

```go
if dstObj != nil && backupDir != nil {
    err = MoveBackupDir(ctx, backupDir, dstObj)  // 覆盖前先备份
    ...
    dstObj = nil
}
_, err = Op(ctx, fdst, dstObj, dstFileName, srcObj)  // Op = Move/Copy
```

---

## 五、场景二：文件删除（Delete）前的备份跳转

### 5.1 两个删除入口

| 入口 | 触发时机 | 位置 |
|---|---|---|
| `startDeleters()` + `deleteFilesCh` | `--delete-during` / `--delete-only`：边遍历边删 | [sync.go L602-L620](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/sync/sync.go#L602-L620) |
| `deleteFiles()` 函数 | `--delete-after`：全部传输完再删；或 `--delete-before` | [sync.go L627-L666](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/sync/sync.go#L627-L666) |

### 5.2 DeleteFilesWithBackupDir 工作原理

位于 [operations.go L588-L628](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L588-L628)：

```
启动 ci.Checkers 个 goroutine 并发消费 toBeDeleted 管道
    │
    ▼ （每个 worker）
for dst := range toBeDeleted {
    DeleteFileWithBackupDir(ctx, dst, backupDir)
}
```

### 5.3 DeleteFileWithBackupDir 核心逻辑

位于 [operations.go L545-L578](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L545-L578)：

```
DeleteFileWithBackupDir(dst, backupDir)
    │
    ├─ backupDir != nil ──────────┐
    │     "move into backup dir"  │
    │     （日志动作名改为此）     │
    │                             │
    └─ backupDir == nil ──┐       │
          "delete"        │       │
          （日志动作名）  │       │
                          ▼       ▼
                SkipDestructive?（--dry-run 等）
                    │
           ┌────────┴────────┐
           ▼                 ▼
         skip          backupDir != nil?
                       │         │
                  ┌────┘         └────┐
                  ▼                   ▼
         MoveBackupDir()        dst.Remove(ctx)
         （移动到备份目录）       （直接调用后端删除接口）
```

删除路径的关键代码：

```go
if backupDir != nil {
    action, actioned = "move into backup dir", "Moved into backup dir"
}
...
if backupDir != nil {
    err = MoveBackupDir(ctx, backupDir, dst)   // ← 跳转：删→挪
} else {
    err = dst.Remove(ctx)                       // ← 正常后端删除
}
```

这样，对于 sync 命令中"多出的文件要被删除"的情况，在 backup-dir 模式下**实际不会被删除**，
而是被**移动**到备份目录中。这就是 backup-dir 最核心的安全语义。

---

## 六、完整调用链汇总

### 6.1 覆盖场景调用链

```
sync/copy/move 命令
  │
  └─► newSyncCopyMove()                       [fs/sync/sync.go]
        │
        └─► operations.BackupDir()            [fs/operations/operations.go L1916]
              │  校验 + 构造 backupDir Fs
              ▼
  └─► pairChecker goroutine × N
        │
        ├─► NeedTransfer()                    [判断是否需要传]
        ├─► CompareOrCopyDest()               [compare/copy-dest 优化，可嵌套]
        │     └─► copyDest()
        │           └─► MoveBackupDir()       ← （可选跳转点 1）
        │
        └─► pair.Dst 存在 且 backupDir != nil
              └─► MoveBackupDir()             ← （主要跳转点 2）
                    │
                    ├─► SuffixName()          [生成备份文件名]
                    ├─► backupDir.NewObject() [定位备份目标位置]
                    └─► Move()                [调用通用 move]
                          │
                          ├─► backend.Features.Move()   （服务端移动）
                          │     或
                          └─► Copy() + dst.Remove()    （复制+删除 模拟）
                                │
                                └─► backend.Features.Copy() / Object.Update()
                                      等后端文件写操作
```

### 6.2 删除场景调用链

```
sync 命令（启用 --delete-*）
  │
  ├─ (delete-during)  march 时遇到 dst 多出 → 送入 deleteFilesCh
  │        │
  │        └─► startDeleters()
  │              └─► DeleteFilesWithBackupDir()
  │
  └─ (delete-before/after)  deleteFiles()
              └─► DeleteFilesWithBackupDir()
                    │
                    └─► DeleteFileWithBackupDir()  × N（并发）
                          │
                     backupDir != nil?
                          │
                     ┌────┴────┐
                     ▼         ▼
             MoveBackupDir()   dst.Remove()
               （挪到备份目录）  （调用后端 Object.Remove）
```

---

## 七、与后端文件系统的交互边界

整个 backup-dir 功能**不要求后端实现任何"备份专用"接口**，完全复用已有能力：

| 后端需要提供的能力 | 对应的 Fs/Object 接口 | 用途 |
|---|---|---|
| 创建 Fs 实例 | `fs.Fs` 通用 | backupDir 作为独立 Fs 使用，路径不同 |
| 定位对象 | `fs.Fs.NewObject(ctx, remote)` | 在 backupDir 中定位"是否已有同名备份" |
| 服务端移动 | `fs.Features().Move(ctx, src, remote)` | 首选路径（高效，单 API 调用） |
| 服务端复制 | `fs.Features().Copy(ctx, src, remote)` | 当 Move 不可用时降级使用 |
| 删除对象 | `fs.Object.Remove(ctx)` | Copy 后删除旧文件，以及 Remove 真删路径 |
| 上传对象 | `fs.Fs.Put` 等 | 新文件写入目标（**不受 backup-dir 影响**） |

> 核心抽象：backup-dir 功能完全是**操作层（fs/operations）和同步层（fs/sync）的编排逻辑**，
> 不侵入任何具体后端。所有后端只要正确实现了 `Move`/`Copy`/`Remove` 标准接口，
> 就能自动获得 backup-dir 能力，无需额外适配。

---

## 八、关键约束汇总（来自 BackupDir 校验）

1. **同 remote 要求**：`--backup-dir` 必须与 `dst` 使用同一 remote 配置（`SameConfig` 为 true），
   因为 server-side move/copy 无法跨云厂商进行。
2. **不重叠要求**：backup-dir 不能是 dst 或 src 的子目录（反之亦然），避免
   扫描时把已经挪走的旧文件又当作新内容处理。
3. **非同目录要求**：当 `srcFileName == ""`（即批量模式）时，backup-dir 不能和
   src/dst 是同一目录。
4. **Server-side 能力要求**：后端至少支持 `Move` 或 `Copy` 特性之一。
5. **与 --no-check-dest 互斥**：两者不能同时使用，因为不检查目标就无法判断要备份什么。
