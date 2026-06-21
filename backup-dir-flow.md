# 备份目录（backup-dir）处理流程详解

本文档深入梳理 rclone 中 `--backup-dir` / `--suffix` 的完整处理逻辑，
覆盖**校验分支差异**、**批量同步 vs 单文件操作的不同路径**、**后缀组合下的同目录判断**，
以及**服务端移动失败后 Copy+Delete 降级的精确边界条件**，
并阐明其与后端文件操作层之间的交互。

---

## 一、配置入口与三种模式

### 1.1 选项定义

相关配置在 [fs/config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/config.go) 中声明：

| 标志 | 结构体字段 | 位置 | 说明 |
|---|---|---|---|
| `--backup-dir <DIR>` | `BackupDir string` | [config.go L616](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/config.go#L616-L616) | 备份目录远程路径 |
| `--suffix <SUF>` | `Suffix string` | [config.go L617](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/config.go#L617-L617) | 追加到被覆盖/删除文件名的后缀 |
| `--suffix-keep-extension` | `SuffixKeepExtension bool` | [config.go L618](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/config.go#L618-L618) | 加后缀时保留扩展名 |

标志注册位于 [config.go L261-L274](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/config.go#L261-L274)，属于 `Sync` 组。

### 1.2 三种触发模式

备份功能的三种触发方式对校验逻辑有**直接影响**：

| 模式 | `ci.BackupDir` | `ci.Suffix` | backupDir Fs 的来源 | 行为语义 |
|---|---|---|---|---|
| A. 仅 `--backup-dir` | 非空 | 任意 | `cache.Get(ci.BackupDir)` 新 Fs | 跨目录移动，文件结构不变 |
| B. 仅 `--suffix` | 空 | 非空 | **直接复用 `fdst`** | 同目录内重命名（加后缀） |
| C. 两者皆设 | 非空 | 非空 | `cache.Get(ci.BackupDir)` 新 Fs | 跨目录移动 + 文件名加后缀 |

> 关键：模式 B 下 `backupDir == fdst`，所以**同目录判断完全走另一条路**。

---

## 二、backupDir Fs 的构造与三重校验分支

### 2.1 构造入口的两大调用场景

`BackupDir()` 函数有两处调用点，传入的 `srcFileName` 参数完全不同：

| 调用场景 | 调用位置 | 传入的 `srcFileName` | 说明 |
|---|---|---|---|
| **批量同步**（sync/copy/move 命令） | [sync.go L271-L278](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/sync/sync.go#L271-L278) | `""`（空字符串） | `newSyncCopyMove()` 初始化时一次性构造 |
| **单文件操作**（copyto/moveto/transform） | [operations.go L2068-L2075](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L2068-L2075) | 具体源文件名（如 `"docs/a.txt"`） | `moveOrCopyFile()` 中按文件构造 |

对应的调用代码：

```go
// 批量同步场景
// sync.go newSyncCopyMove()
if ci.BackupDir != "" || ci.Suffix != "" {
    s.backupDir, err = operations.BackupDir(ctx, fdst, fsrc, "")  // ← srcFileName = ""
}

// 单文件操作场景
// operations.go moveOrCopyFile()
if ci.BackupDir != "" || ci.Suffix != "" {
    backupDir, err = BackupDir(ctx, fdst, fsrc, srcFileName)      // ← srcFileName = 具体路径
}
```

### 2.2 BackupDir() 校验全景

核心函数：[operations.go L1916-L1952](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L1916-L1952)

完整的分支决策树如下：

```
BackupDir(ctx, fdst, fsrc, srcFileName)
    │
    ▼
┌─ ci.BackupDir != "" ? ─────────────────────────────────────┐
│  │ 是（模式 A 或 C）                                        │
│  ▼                                                          │
│ backupDir = cache.Get(ctx, ci.BackupDir)                    │
│   │                                                        │
│   ▼                                                        │
│ SameConfig(fdst, backupDir)?                               │
│   │ 否 → FatalError: 必须同 remote                          │
│   ▼ 是                                                     │
│ ┌──────────────────────────────────────────────────┐       │
│ │ srcFileName == "" ?                              │       │
│ │ （批量同步）                                      │       │
│ │   │ 是                                           │       │
│ │   ▼                                              │       │
│ │ OverlappingFilterCheck(backupDir, fdst)?         │       │
│ │   │ 是 → FatalError: dst 与 backup-dir 重叠      │       │
│ │   ▼ 否                                           │       │
│ │ OverlappingFilterCheck(backupDir, fsrc)?         │       │
│ │   │ 是 → FatalError: src 与 backup-dir 重叠      │       │
│ │   ▼ 否 → 跳至「最后通用检查」                    │       │
│ │                                                  │       │
│ │ srcFileName != ""（单文件操作） ──────────┐       │       │
│ │   │                                      │       │       │
│ │   ▼                                      │       │       │
│ │ ci.Suffix == "" ?（模式 A，仅 backup-dir）│       │       │
│ │   │ 是                                   │       │       │
│ │   ▼                                      │       │       │
│ │ SameDir(fdst, backupDir)?                │       │       │
│ │   │ 是 → FatalError: 非同目录             │       │       │
│ │   ▼ 否                                   │       │       │
│ │ SameDir(fsrc, backupDir)?                │       │       │
│ │   │ 是 → FatalError: 非同目录             │       │       │
│ │   ▼ 否 → 跳至「最后通用检查」            │       │       │
│ │                                          │       │       │
│ │ ci.Suffix != ""（模式 C，两者皆设）      │       │       │
│ │   │ 是                                   │       │       │
│ │   ▼                                      │       │       │
│ │ ★ 跳过 Overlapping 与 SameDir 所有检查 ★ │       │       │
│ │   （理由：加 suffix 后即使同目录也不会    │       │       │
│ │    与原文件重名，因此不存在冲突）         │       │       │
│ │   │                                      │       │       │
│ │   └──── 跳至「最后通用检查」◄────────────┘       │       │
│ └──────────────────────────────────────────────────┘       │
│                                                            │
│ 否（ci.BackupDir == ""）                                   │
│   │                                                        │
│   ▼ ci.Suffix != "" ?（模式 B：仅 suffix）                 │
│      是 → backupDir = fdst  ★直接复用目标Fs★               │
│          （跳过 Overlapping / SameDir 所有检查，            │
│           因为 backupDir 就是 fdst 本身）                   │
│      否 → FatalError: 内部错误（两者皆空）                  │
│                                                            │
└────────────────────────────┬───────────────────────────────┘
                             ▼
                     ┌───────────────┐
                     │ 最后通用检查  │
                     └───────┬───────┘
                             ▼
              CanServerSideMove(backupDir)?
                  │ 否 → FatalError: 后端必须支持
                  │        server-side move 或 copy
                  ▼ 是
              return backupDir, nil
```

### 2.3 三种校验函数的内部实现

#### 2.3.1 SameConfig — 配置名比较

[operations.go L661-L663](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L661-L663)

```go
func SameConfig(fdst, fsrc fs.Info) bool {
    return fdst.Name() == fsrc.Name()  // 比较 rclone.conf 中的节名（remote 名）
}
```

只检查 remote 名称，不比较根路径。含义：backup-dir 必须和 dst 配在同一个 rclone remote 下，
否则 server-side move/copy 的 API 根本无法跨后端调度。

#### 2.3.2 SameDir — 根目录精确匹配（单文件模式用）

[operations.go L729-L736](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L729-L736)

```go
func SameDir(fdst, fsrc fs.Info) bool {
    if !SameConfig(fdst, fsrc) { return false }
    _, fdstRootFolded := fixRoot(fdst)
    _, fsrcRootFolded := fixRoot(fsrc)
    return fdstRootFolded == fsrcRootFolded  // 严格比较（大小写折叠后）
}
```

被 `fixRoot` 规范化后比较：
1. `filepath.ToSlash` → 统一为 `/` 分隔
2. `strings.Trim(root, "/")` → 去掉两侧斜杠
3. 非空则末尾补 `/`
4. 若 `Features.CaseInsensitive`，整体 `ToLower`

**用途（仅模式 A + 单文件场景）**：禁止 backup-dir 和 src/dst 指向完全相同的目录。
因为单文件操作下，如果 backup-dir 就是 dst 自身（且没加 suffix），移进去等于没动。

#### 2.3.3 OverlappingFilterCheck — 目录重叠+过滤检查（批量场景用）

[operations.go L697-L725](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L697-L725)

批量同步（`srcFileName == ""`）下用更严格的重叠检测：

```
OverlappingFilterCheck(ctx, A, B)
    │
    ▼
SameConfig(A, B)? → 否，直接 return false（不同 remote 不可能重叠）
    │ 是
    ▼
fixRoot(A), fixRoot(B) 获得规范化根目录 + 大小写折叠版本
    │
    ▼
A.root == B.root（折叠后）？ → 是，return true（完全相同）
    │ 否
    ▼
A.root 以 B.root 为前缀？
    │ 是 → 取 A 相对 B 的部分，调用 filterCheck(ctx, B, A_relative)
    │       检查该子目录是否在 B 的 filter 规则下会被包含
    │       包含 → return true（重叠）；不包含 → false（不重叠）
    │ 否
    ▼
B.root 以 A.root 为前缀？
    │ 是 → 同理取 B 相对 A 的部分，走 filterCheck
    │ 否
    ▼
return false（无重叠）
```

> **关键设计**：OverlappingFilterCheck 不仅看目录层级是否有祖先关系，
> 还**结合当前 filter 规则**判断实际上是否会命中被扫描的文件。
> 比如 backup-dir 虽然是 dst 的子目录，但如果 filter 明确排除它，就不算"重叠"。
> filterCheck 出错时会**保守地认为重叠**（返回 true）。

### 2.4 模式 B（仅 --suffix）的特殊语义

当 `ci.BackupDir == "" && ci.Suffix != ""` 时：

```go
backupDir = fdst   // [operations.go L1942-L1944]
```

此时没有任何 SameDir / OverlappingFilterCheck 检查。
因为**同目录重命名（改名不会冲突）**正是这种模式的设计目的。

同时注意：模式 B 下，最终的"移动"操作其实是**同一个 Fs 内部的重命名**——
`Move(ctx, backupDir=fdst, ..., src=原dst文件)`，
如果 remote 支持 Move 特性，就是一次高效的 Rename API 调用。

### 2.5 模式 C（两者皆设）跳过目录检查的原因

当 `srcFileName != ""`（单文件）且同时设置了 `--suffix` 时，
代码有意跳过 SameDir 检查。理由是：**即使 backup-dir 与 dst/src 指向相同目录，
加上 suffix 后文件名必然不同**（如 `a.txt` → `a.txt.bak`），不会产生覆盖冲突。

但注意：这道"免检"**只在单文件路径**上生效，批量路径（`srcFileName == ""`）
仍然强制走 OverlappingFilterCheck，因为批量扫描下目录重叠会带来系统性问题
（旧备份被反复扫描、再次被当作"新文件"处理等）。

### 2.6 最后一关：CanServerSideMove

[operations.go L526-L530](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L526-L530)

```go
func CanServerSideMove(fdst fs.Fs) bool {
    canMove := fdst.Features().Move != nil   // 原生服务端移动
    canCopy := fdst.Features().Copy != nil   // 可退化为 copy+delete
    return canMove || canCopy
}
```

含义：即使后端没有 `Move` 特性，只要能做服务端 `Copy`，通过"先复制后删源"也能等效完成移动。
详见第四章的降级边界分析。

---

## 三、核心跳转函数 MoveBackupDir 详解

[operations.go L1954-L1960](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L1954-L1960)

```go
func MoveBackupDir(ctx context.Context, backupDir fs.Fs, dst fs.Object) (err error) {
    remoteWithSuffix := SuffixName(ctx, dst.Remote())
    overwritten, _ := backupDir.NewObject(ctx, remoteWithSuffix)
    _, err = Move(ctx, backupDir, overwritten, remoteWithSuffix, dst)
    return err
}
```

### 3.1 三步流程（含后端交互点）

| 步骤 | 操作 | 具体后端交互 |
|---|---|---|
| ① 生成备份名 | `SuffixName(ctx, dst.Remote())` | 纯内存计算，无后端调用 |
| ② 定位备份目标 | `backupDir.NewObject(ctx, remoteWithSuffix)` | 调用 `fs.Fs.NewObject` — 在 backupDir Fs 下检查是否已存在同名备份文件。返回的对象存入 `overwritten`，若不存在则为 `nil`（忽略 error，因为 `ErrorObjectNotFound` 属于正常情况） |
| ③ 执行移动 | `Move(ctx, backupDir, overwritten, remoteWithSuffix, dst)` | 进入通用 Move 函数，真正与后端的 Move/Copy/Remove 交互 |

### 3.2 SuffixName 的命名规则

[operations.go L532-L543](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L532-L543)

```
Suffix 为空
  → 原样返回 remote（模式 A 下 backup-dir 重命名不变名）

Suffix 非空 + SuffixKeepExtension = true
  → transform.SuffixKeepExtension(remote, suffix)
    例："report/data.csv" + ".bak" → "report/data.bak.csv"

Suffix 非空 + SuffixKeepExtension = false（默认）
  → remote + suffix
    例："report/data.csv" + ".bak" → "report/data.csv.bak"
```

### 3.3 overwritten 参数的作用

`overwritten = backupDir.NewObject(ctx, remoteWithSuffix)` 找到 backupDir 下可能已存在的旧备份。
它传入 Move 的目的是：**如果该位置已有同名备份，Move 内部会先 `DeleteFile(overwritten)` 腾出位置**
（详见下一章 Move 内部的处理）。保证最新一次备份总是能写入目标路径，旧备份被覆盖替换。

---

## 四、Move() 函数：服务端移动与 Copy+Delete 降级边界

[operations.go L433-L519](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L433-L519)

这是备份跳转真正落地的**核心桥梁**。调用链为：

```
MoveBackupDir → Move(ctx, backupDir, overwritten, remoteWithSuffix, dst)
              → move(ctx, ..., isTransfer=false)
```

### 4.1 完整流程图

```
move(ctx, fdst=backupDir, dst=overwritten, remote=remoteWithSuffix, src=原dst文件, isTransfer=false)
    │
    ▼
[0] 路径规范化 + DryRun 短路
    remote = transform.Path(ctx, remote, false)
    ci.DryRun 且 src==dst 同路径 → 直接返回

    │
    ▼
[1] Server-Side Move 前置条件检查（L463）
    doMove := fdst.Features().Move
    条件 A：doMove != nil（后端声明支持 Move 特性）
       AND
    条件 B1：SameConfig(src.Fs(), fdst)                      （同配置，最常见）
       OR
    条件 B2：SameRemoteType(src.Fs(), fdst)
              AND (fdst.Features().ServerSideAcrossConfigs     （后端主动支持跨配置）
                   OR ci.ServerSideAcrossConfigs)              （用户强制开启跨配置）

    条件不满足 ──────────────────────────────┐
    │                                         │
    ▼ 条件满足                                │
[2] dst(overwritten) 处理（L464-L483）        │
    dst != nil?                               │
      │ 是                                    │
      ├─ !SameObject(src, dst)?               │
      │    │ 是 → DeleteFile(ctx, dst)        │
      │    │       （先删旧的同名备份占位）   │
      │    │       失败 → return err          │
      │    │                                  │
      │    └─ src.Remote() == remote?         │
      │         是 → return（源=目标，无事做）│
      │         否 → 大小写变更？走特殊分支   │
      │                                      │
      否（dst == nil，无同名旧备份）          │
        └─ 大小写变更？走特殊分支             │
    │                                         │
    ▼                                         │
[3] 执行后端 doMove（L484-L506）              │
    in.ServerSideTransferStart()              │
    newDst, err = doMove(ctx, src, remote)    │
       ↓                                      │
       ┌───────────────────────┐              │
       │ err 分类处理           │              │
       ├───────────────────────┤              │
       │ nil → 成功！           │              │
       │   记日志 +              │              │
       │   ServerSideMoveEnd()  │              │
       │   return newDst, nil  │              │
       │                       │              │
       │ fs.ErrorCantMove →    │              │
       │   特殊降级信号！       │              │
       │   Debug 日志，关闭统计 │              │
       │   继续向下 → [4]      │              │
       │                       │              │
       │ 其他任何错误 →        │              │
       │   CountError + Errorf │              │
       │   return newDst, err  │              │
       │   ★ 不降级！直接报错 ★│              │
       └───────────────────────┘              │
                                              │
[4] ──────────── Server-Side Move 不可行，降级为 Copy+Delete ──────────────
    │   （进入条件：L463 条件不满足，或 L498 doMove 返回 ErrorCantMove）
    │
    ▼
[4a] 目标对象修正（L509-L511）
    origRemote != remote? → dst = nil
    （如果 dst 路径已因 L466/L467 被改写，此处清空让 Copy 用原始名）

    │
    ▼
[4b] 执行 Copy（L512）
    newDst, err = Copy(ctx, fdst, dst, origRemote, src)
    │
    ├─ err != nil（复制失败）
    │     Errorf: "Not deleting source as copy failed"
    │     return newDst, err
    │     ★ 不删除源文件！（保留现场避免数据丢失）★
    │
    └─ err == nil（复制成功）
          │
          ▼
[4c] 删除源文件（L518）
    return newDst, DeleteFile(ctx, src)
    ↑ 语义：复制完成后才真正移除原文件
```

### 4.2 降级触发条件的精确边界

#### 4.2.1 什么情况下会触发降级？（进入 Copy+Delete）

| 编号 | 触发条件 | 说明 |
|---|---|---|
| D1 | `fdst.Features().Move == nil` | 后端根本没实现 Move 接口 |
| D2 | Move 存在但 `!SameConfig && !跨配置条件` | 后端实现了 Move，但因跨 remote 配置不可用 |
| D3 | `doMove(ctx, src, remote) == fs.ErrorCantMove` | 后端返回**明确的降级信号**（这是唯一一种错误会触发降级） |

> **fs.ErrorCantMove 是一个特殊哨兵值**：后端在发现该次操作无法通过 server-side move
> 完成（例如跨 bucket、跨 storage class 等限制）时，返回这个特定错误告诉 rclone
> "请走 copy+delete 路线"，而不是当作失败。

#### 4.2.2 什么错误不会降级？（直接失败）

| 条件 | 行为 | 代码位置 |
|---|---|---|
| `doMove` 返回 **非 `ErrorCantMove` 的任何错误**（HTTP 5xx、权限错误、网络超时等） | 立即 return err，不尝试 Copy | [L501-L505](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L501-L505) |
| `DeleteFile(dst)` 删除旧备份失败（L468） | 立即 return err，不尝试 doMove 或 Copy | [L468-L471](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L468-L471) |
| `Copy` 本身失败 | 保留源文件不删，返回 Copy 的错误 | [L513-L515](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L513-L515) |

#### 4.2.3 Copy 成功后 DeleteFile 的边界

只有当 Copy 完全成功（`err == nil`）时，才会走到 L518 的 `DeleteFile(ctx, src)`。
这意味着**降级路径的原子性**：要么文件还在原位（失败），要么已在新位置（成功），
不会出现中间状态（只复制了一半）。

但注意：极端情况下 Copy 成功、Delete 失败时，会出现**源文件和目标文件同时存在**的重复，
这是"至少一次"语义的设计选择——宁可重复也不丢失。

### 4.3 Copy 内部的后端交互

由 [copy.go L390-L425](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/copy.go#L390-L425) 发起：

```
Copy(ctx, f, dst, remote, src)
  → c = &copy{...}
  → c.doUpdate = (dst != nil)   ← 决定是原地覆盖还是新建
  → c.checkPartial()            ← 处理 --partial 等命名
  → c.copy()
       │
       ├─ 优先尝试：dstFeatures.ServerSideAcrossConfigs 等条件下的 Copy 特性
       ├─ 不行则：MultipartUpload / 分块上传
       └─ 最兜底：fs.Put(ctx, in, ...) 流式上传
```

因此，MoveBackupDir → Move → Copy 这条链路中，**Copy 内部也有 server-side 优化**
（如果 backend.Features.Copy 存在则直接走服务端复制，不经过本地带宽），
所以即使降级为 Copy+Delete，对支持 Copy 的后端也是高效的零带宽操作。

---

## 五、场景一：文件覆盖前的备份跳转（三大触发点）

### 5.1 触发点总览

覆盖场景下，`MoveBackupDir` 被调用的位置有 **三处**，分别处理不同优化路径：

| 编号 | 调用位置 | 触发条件 |
|---|---|---|
| O1 | `copyDest()` 内部 | 使用 `--copy-dest` 且从 copy-dest 目录命中服务器端复制源时 |
| O2 | `pairChecker()` 主路径 | 批量同步的常规检查流程（主路径） |
| O3 | `moveOrCopyFile()` 内部 | 单文件 copyto/moveto 命令 |

### 5.2 O1：copyDest 中的备份（服务器端复制优化路径）

[operations.go L1661-L1702](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L1661-L1702)

```
copyDest(ctx, fdst, dst, src, CopyDestFs, backupDir)
    │
    ▼
CopyDestFs.NewObject(ctx, remote)  // 在 --copy-dest 指定的目录里找 src
    │ 找不到 → return false（本优化不适用）
    ▼ 找到
equal(src, CopyDestFile)? → 不相等 → return false
    │ 相等（可以服务器端复制过来）
    ▼
dst == nil || !Equal(src, dst)     // dst 还没有，或内容不同需要覆盖
    │
    ▼
dst != nil && backupDir != nil ?
    │ 是
    ▼
MoveBackupDir(ctx, backupDir, dst)  ← 备份跳转点
    │ 失败 → return error
    ▼ 成功
dst = nil                            ← 清空，避免 Copy 以为是覆盖
    │
    ▼
Copy(ctx, fdst, dst, remote, CopyDestFile)  ← 从 copy-dest 服务器端复制
```

**为什么这里也需要备份？**：因为 `--copy-dest` 的语义是"如果 A 目录里有相同文件，
就从 A 复制到 dst，而不用从 src 重新下载上传"——但不管怎么复制，
dst 里的旧文件要被替换，该备份的还是得备份。

### 5.3 O2：pairChecker 主路径（批量同步核心）

[sync.go L371-L476](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/sync/sync.go#L371-L476)

主处理流程：

```
pairChecker 从 toBeChecked 管道中取出 ObjectPair{Src, Dst}
    │
    ▼
NeedTransfer(dst, src)?
    │ 否（不需要传输）
    │   DoMove? → 删 src（move 场景下），继续下一个 pair
    ▼ 是（需要传输）
    │
    ▼
CompareOrCopyDest(..., backupDir)?   ← 先走 compare-dest/copy-dest 优化（可能命中 O1）
    │ NoNeedTransfer=true → 已完成，跳到下一个 pair
    ▼ 仍需传输
    │
    ▼
Immutable 且 dst 存在 → 报错（不能修改）
    │ 否则
    ▼
┌─ pair.Dst != nil 且 s.backupDir != nil ? ─────────────────┐
│  │ 是（目标有旧文件，且启用了备份）                         │
│  │                                                         │
│  │   err = MoveBackupDir(ctx, s.backupDir, pair.Dst)       │
│  │      │ 失败 → processError + logger，不送入上传队列     │
│  │      ▼ 成功                                             │
│  │   pair.Dst = nil   ← ★ 关键：清空 dst 引用             │
│  │                        告诉后续 pairCopyOrMove：        │
│  │                        目标位置已空，按"新文件"处理    │
│  │                                                         │
│  └─ out.Put(pair) → 送入 toBeUploaded                      │
│                                                             │
└─ 否（目标为空，或未启用备份）                               │
   │                                                          │
   └─ out.Put(pair) → 直接送入 toBeUploaded                  │
                                                              │
                                                              ▼
                                              pairCopyOrMove worker × Transfers
                                                          │
                                                          ▼
                                          src==dst → DeleteFile（DoMove 且 checkFirst 特例）
                                          DoMove=true → MoveTransfer
                                          DoMove=false → Copy
```

`pair.Dst = nil` 的意义：后续的 `Copy(ctx, fdst, pair.Dst=nil, remote, src)`
会按"创建新文件"路径执行（`c.doUpdate=false`，见 copy.go L410），
不会再尝试在 dst 找旧对象做覆盖——因为旧文件已经被移走了。

### 5.4 O3：moveOrCopyFile 单文件路径

[operations.go L2014-L2121](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L2014-L2121)

单文件操作（rclone copyto / moveto / transform）直接在函数内构造 backupDir 并完成跳转：

```
moveOrCopyFile(fdst, fsrc, dstFileName, srcFileName, cp, allowOverlap)
    │
    ▼
srcObj = fsrc.NewObject(srcFileName)
dstObj = ci.NoCheckDest ? nil : fdst.NewObject(dstFileName)
    │
    ▼ （大小写变更等特殊路径先处理）
    ▼
backupDir = BackupDir(ctx, fdst, fsrc, srcFileName)  ← 单文件分支（srcFileName 非空）
    │
    ▼
NeedTransfer(dstObj, srcObj)?
    │ 是 → CompareOrCopyDest（可能触发 O1） → 仍需传输?
    │         │
    │         ▼
    │      dstObj != nil && backupDir != nil ?
    │         │ 是
    │         ▼
    │      MoveBackupDir(ctx, backupDir, dstObj)   ← O3 跳转
    │         │ 失败 → return error
    │         ▼ 成功
    │      dstObj = nil
    │
    ▼
Op(ctx, fdst, dstObj, dstFileName, srcObj)     ← Op = Copy 或 MoveTransfer
```

单文件场景的特点：每次处理一个文件就构造一次 backupDir（有缓存，开销不大），
并且通过传 `srcFileName` 让 `BackupDir()` 走 **SameDir 检查而非 Overlapping 检查**。

---

## 六、场景二：文件删除前的备份跳转

### 6.1 删除三入口

| 入口 | 对应标志 | 调用时机 | 位置 |
|---|---|---|---|
| D1. `deleteFiles()` 函数 | `--delete-before` / `--delete-after`（默认） | 同步前或全部传完后，集中处理 dst 多出的文件 | [sync.go L627-L666](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/sync/sync.go#L627-L666) |
| D2. `startDeleters()` + `deleteFilesCh` | `--delete-during` / `--delete-only` | 边遍历边删，march 过程中发现 dst 多出就送入管道，后台并发删 | [sync.go L602-L620](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/sync/sync.go#L602-L620) |
| D3. `DeleteFile(ctx, src)` 直接调用 | `DoMove=true` 且不需要传输时 | pairChecker 中如果 DoMove 且 src 与 dst 相等，直接删除 src | [sync.go L467](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/sync/sync.go#L467-L470) |

> 注意 D3：它调用的是**无 backupDir 版本**的 `DeleteFile`（`operations.DeleteFile`
> 即 `DeleteFileWithBackupDir(ctx, dst, nil)`），所以 move 命令中清理 src 不会触发备份——
> 毕竟这些文件在 dst 已有副本，不是需要"保护"的内容。

### 6.2 DeleteFilesWithBackupDir 并发模型

[operations.go L588-L628](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L588-L628)

```
DeleteFilesWithBackupDir(ctx, toBeDeleted chan Object, backupDir Fs)
    │
    ├─ 启动 ci.Checkers 个 goroutine（默认并发 = checker 数）
    │
    └─ 每个 worker：
         for dst := range toBeDeleted {
             err := DeleteFileWithBackupDir(ctx, dst, backupDir)
             if err != nil {
                 errorCount++
                 IsFatalError? → fatalErrorCount++; return（worker 退出）
             }
         }
    │
    ▼ wg.Wait()
    errorCount > 0?
      是 → fatalErrorCount > 0 ? FatalError(err) : 普通 err
      否 → nil
```

### 6.3 DeleteFileWithBackupDir 精确决策

[operations.go L545-L578](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L545-L578)

```
DeleteFileWithBackupDir(ctx, dst, backupDir)
    │
    ▼
[1] 统计与限流
    accounting.NewCheckingTransfer(dst, "deleting")
    accounting.DeleteFile(ctx, dst.Size())   // 检查删除限额
       │ err → return（不继续）
       ▼
[2] 按是否启用备份决定动作名
    backupDir != nil?
        是 → action = "move into backup dir"   actioned = "Moved into backup dir"
        否 → action = "delete"                  actioned = "Deleted"
    │
    ▼
[3] --dry-run / --interactive 等保护性检查
    skip = SkipDestructive(ctx, dst, action)
        │ skip=true → 不执行任何操作，直接进入 [5] 日志
        ▼ skip=false
[4] ★ 真·分流点 ★
    ┌─────────┴─────────┐
    ▼                   ▼
backupDir != nil?     否（未设 backup / suffix）
    │ 是                │
    ▼                   ▼
MoveBackupDir()     dst.Remove(ctx)   ← 直接调用后端 Object.Remove
(挪到备份目录)      （真·删除，不可恢复）
    │                   │
    └─────────┬─────────┘
              ▼
[5] 错误处理 + 日志
    err != nil → Errorf + CountError
    err == nil && !skip → Infof(actioned)
    return err
```

### 6.4 删除路径的后端交互

| 路径 | 调用链 | 后端接口 |
|---|---|---|
| 无备份（真删） | `dst.Remove(ctx)` | 直接 `Object.Remove` |
| 有备份（跳转） | `MoveBackupDir → Move → doMove` | `Features.Move`（server-side move） |
| 有备份（跳转降级） | `MoveBackupDir → Move → Copy → DeleteFile` | `Features.Copy` + `Object.Remove`（或 Copy fallback 到 Put） |

---

## 七、完整调用链汇总

### 7.1 覆盖场景（Overwrite）调用链

```
[批量] sync/copy/move 命令                            [单文件] copyto/moveto 命令
    │                                                      │
    ▼                                                      ▼
newSyncCopyMove() [fs/sync/sync.go]                moveOrCopyFile() [operations.go]
    │                                                      │
    └─► BackupDir(..., srcFileName="")                     └─► BackupDir(..., srcFileName="...")
            │  (批量：OverlappingFilterCheck)                      │  (单文件：SameDir 或 跳过)
            ▼                                                      ▼
pairChecker goroutine × N                                    NeedTransfer + CompareOrCopyDest
    │                                                              │
    ├─► NeedTransfer()                                             │
    ├─► CompareOrCopyDest()                                        │
    │     └─► copyDest()                                           │
    │           └─► MoveBackupDir() [O1]                           │
    │                                                              │
    └─► pair.Dst 存在 且 backupDir != nil ◄────────────────────────┘
          └─► MoveBackupDir() [O2 / O3]
                │
                ├─► SuffixName()              [命名]
                ├─► backupDir.NewObject()     [定位 backupDir 下同名旧备份]
                └─► Move()                    [通用 move 入口]
                      │
                      ├─ [条件满足] backend.Features.Move()
                      │     ├─ 先 DeleteFile(overwritten)        [如果旧备份存在先清]
                      │     ├─ doMove(ctx, src, remote)          [调用后端 Move]
                      │     │     ├─ nil → 成功返回
                      │     │     ├─ ErrorCantMove → 降级 → [B]
                      │     │     └─ 其他错误 → 直接报错（不降级）
                      │     │
                      │     └─ [B] 降级：
                      │
                      └─ Copy(ctx, backupDir, dst, origRemote, src)
                           ├─ 错误 → 不删源文件，返回错误
                           └─ 成功 → DeleteFile(src)   [最终删除原位置文件]
```

### 7.2 删除场景（Delete）调用链

```
sync --delete-before/after                    sync --delete-during / --delete-only
       │                                                 │
       ▼                                                 ▼
deleteFiles()                                   march() 匹配中写入 deleteFilesCh
       │  (构造 toDelete 管道)                           │
       └───────────────┬─────────────────────────────────┘
                       ▼
            DeleteFilesWithBackupDir(toBeDeleted, backupDir)
                       │
              启动 ci.Checkers 个 goroutine
                       │
                       ▼ (× 并发)
        DeleteFileWithBackupDir(ctx, dst, backupDir)
                       │
                 ┌─────┴─────┐
                 ▼           ▼
          backupDir?       否 → dst.Remove(ctx)
                 │ 是
                 ▼
            MoveBackupDir()
                 │
                 ▼
            Move() → （同上：Move 或 Copy+Delete）
```

---

## 八、与后端文件系统的交互边界

整个 backup-dir 功能是**纯上层编排**，不要求任何后端实现"备份专用接口"。
所有后端只要正确实现了下列**标准 Fs/Object 接口**即可自动获得备份能力：

| 后端需提供的能力 | 接口签名 | 被谁调用 |
|---|---|---|
| 创建 Fs 实例 | （Fs 构造函数，经 `cache.Get` 调度） | `BackupDir()` L1920 |
| 定位对象 | `Fs.NewObject(ctx, remote) Object` | `MoveBackupDir()` L1957 找旧备份；单文件流程找 dst |
| 服务端移动 | `Features.Move(ctx, src, remote) Object` | `move()` L487 首选路径 |
| 服务端复制 | `Features.Copy(ctx, src, remote) Object` | `move()` 降级时 `Copy()` 内 server-side copy |
| 删除对象 | `Object.Remove(ctx) error` | 删除旧备份（L468）、降级后删源（L518）、无备份时真删（L569） |
| 写入对象 | `Fs.Put` / `MultipartUpload` 等 | **不受 backup-dir 影响**，新文件写入照常进行 |
| 大小写不敏感标记 | `Features.CaseInsensitive` | `fixRoot` / `needsMoveCaseInsensitive` |
| 跨配置复制标记 | `Features.ServerSideAcrossConfigs` | 决定 Move 是否能跨配置生效 |

> **架构洞察**：backup-dir 功能完全落在 `fs/operations`（操作原子）和
> `fs/sync`（流程编排）两层。后端（`backend/*`）完全感知不到它的存在，
> 只需继续做好 Move/Copy/Remove 的正确语义即可——这是典型
> **"高层通过组合低层能力提供高级特性"** 的设计范例。

---

## 九、关键约束与易错点汇总

### 9.1 BackupDir() 校验约束

| 约束项 | 生效模式 | 代码位置 |
|---|---|---|
| backup-dir 必须与 dst 同 remote（`SameConfig`） | 模式 A、C | [L1924-L1926](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L1924-L1926) |
| 批量同步：backup-dir 与 src/dst 不能重叠，考虑 filter 规则 | 模式 A、C + `srcFileName==""` | [L1927-L1933](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L1927-L1933) |
| 单文件 + 仅 backup-dir：backup-dir 与 src/dst 非同目录 | 模式 A + `srcFileName!=""` | [L1934-L1940](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L1934-L1940) |
| 单文件 + 两者皆设：**豁免目录检查** | 模式 C + `srcFileName!=""` | [L1934 条件不满足，跳过整个分支](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L1934-L1941) |
| 仅 suffix：**豁免所有目录检查**，直接复用 fdst | 模式 B | [L1942-L1944](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L1942-L1944) |
| 后端必须支持 Move 或 Copy | 所有模式 | [L1948-L1950](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/operations/operations.go#L1948-L1950) |
| 不能与 `--no-check-dest` 同用 | 批量同步 | [sync.go L236-L238](file:///d:/fz/0601-2/solo-dogfeeding/code/102-rclone/fs/sync/sync.go#L236-L238) |

### 9.2 Move() 降级边界易错点

| 情形 | 结果 | 注意 |
|---|---|---|
| doMove 返回 HTTP 500 / 权限错 / 网络超时 | **直接报错，不降级** | 只有返回 `fs.ErrorCantMove` 才降级 |
| 降级后 Copy 失败 | **保留源文件**，返回错误 | 宁可重复，不丢数据 |
| Copy 成功、DeleteFile 失败 | 源文件和目标文件**同时存在** | at-least-once 语义 |
| `overwritten`（同名旧备份）删除失败 | 整个 Move 失败，**不继续** | 旧备份删不掉就无法写入新位置 |
| `--dry-run` 模式 | Move / Copy / Delete 全被 `SkipDestructive` 短路 | MoveBackupDir 也会被正确跳过 |

### 9.3 批量 vs 单文件的校验差异总结

| 检查项 | 批量同步（srcFileName=""） | 单文件操作（srcFileName!=""）仅 backup-dir | 单文件操作（srcFileName!=""）backup-dir+suffix | 仅 suffix（所有场景） |
|---|---|---|---|---|
| SameConfig | ✓ | ✓ | ✓ | —（fdst 自用） |
| OverlappingFilterCheck(×2) | ✓ | ✗ | ✗ | ✗ |
| SameDir(×2) | ✗ | ✓ | ✗ | ✗ |
| CanServerSideMove | ✓ | ✓ | ✓ | ✓ |

> 从上表可以看出：**模式越"安全"（加 suffix），校验越宽松**——因为 suffix 本身已经提供了文件名层面的隔离保证，目录层面的冲突自然就不存在了。
