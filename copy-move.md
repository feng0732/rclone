# rclone Copy 和 Move 操作执行顺序分析

## 核心代码文件

| 操作 | 文件 | 关键函数 |
|------|------|----------|
| Copy (单文件) | [fs/operations/copy.go](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go) | `Copy()`, `copy.copy()`, `copy.serverSideCopy()`, `copy.manualCopy()` |
| Copy (目录) | [fs/sync/sync.go](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/sync/sync.go) | `CopyDir()`, `runSyncCopyMove()`, `pairCopyOrMove()` |
| Move (单文件) | [fs/operations/operations.go](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/operations.go) | `Move()`, `MoveTransfer()`, `move()` |
| Move (目录) | [fs/sync/sync.go](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/sync/sync.go) | `MoveDir()`, `moveDir()`, `runSyncCopyMove()` |

---

## Copy 操作执行顺序

### 1. 入口调用链

```
cmd/copy/copy.go
  └─ sync.CopyDir() / operations.CopyFile()
      └─ runSyncCopyMove(deleteMode=Off, DoMove=false)
          └─ pairCopyOrMove()
              └─ operations.Copy()
```

### 2. 单文件 Copy 详细流程 ([operations.Copy](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L390-L425))

#### 阶段一：初始化

1. 创建传输统计（accounting transfer）
2. 确定公共 hash 类型（`CommonHash`）
3. **检查 partial 模式** ([checkPartial](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L91-L116)):
   - 若启用 `--inplace`、目标不支持 Move、不支持 PartialUploads、或文件是 `.rclonelink` → **inplace 模式**（直接写入最终文件名）
   - 否则 → **partial 模式**（写入带 `.partial` 后缀的临时文件，完成后重命名）

#### 阶段二：复制主循环 ([copy.copy](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L307-L383))

循环 `LowLevelRetries` 次：

##### 2.1 服务端复制优先 ([serverSideCopy](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L139-L172))

**触发条件：**
- 目标后端实现了 `Features.Copy` 接口
- 源和目标**同配置**（`SameConfig`），或**同远程类型且支持跨配置服务端复制**

**执行：**
- 调用 `dstFeatures.Copy(ctx, src, remoteForCopy)`
- 成功 → 跳过手动复制，进入验证阶段
- 返回 `fs.ErrorCantCopy` → **降级到手动复制**
- 其他错误 → 根据是否可重试决定是否重试

##### 2.2 手动复制（降级方案）([manualCopy](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L244-L283))

1. 注册 atexit 钩子，程序异常退出时清理 partial 文件
2. **多线程复制判定**：
   - 若 `doMultiThreadCopy()` 为 true → [multiThreadCopy](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L175-L183)
3. 否则：
   - 打开源对象读取流
   - 若源大小未知（-1）→ [rcat](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L188-L210)（流式上传）
   - 否则 → [updateOrPut](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L213-L241)（若目标存在则 Update，否则 Put）

##### 2.3 重试逻辑

- `fserrors.IsRetryError` / `fserrors.ShouldRetry` / `pacer.IsRetryAfter` → 重试
- 重试前重置 accounting 统计 (`c.tr.Reset`)
- 超过最大重试次数 → 进入失败回退

#### 阶段三：失败回退

| 失败场景 | 回退行为 | 代码位置 |
|----------|----------|----------|
| 复制主循环错误（非 inplace） | 删除 partial 文件 | [copy.copy:348-349](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L345-L352) |
| 复制过程中程序异常退出（非 inplace） | atexit 钩子删除 partial 文件 | [copy.manualCopy:247-250](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L244-L251) |
| 校验（size/hash）失败 | 删除已复制的目标文件 | [copy.copy:355-360](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L354-L361) |
| partial → 最终文件名 Move 失败 | 删除 partial 文件 | [copy.copy:364-370](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L364-L374) |

#### 阶段四：partial 文件转正

若使用 partial 模式且 `remoteForCopy != remote`：
1. 调用 `dstFeatures.Move(ctx, newDst, c.remote)` 将 partial 文件重命名为最终文件名
2. Move 失败 → 删除 partial 文件并返回错误
3. Move 成功 → 完成

---

## Move 操作执行顺序

### 1. 入口调用链

```
cmd/move/move.go
  └─ sync.MoveDir() / operations.MoveFile()
      ├─ [目录级] 尝试服务端 DirMove → 失败降级
      └─ runSyncCopyMove(deleteMode=Off, DoMove=true)
          └─ pairCopyOrMove()
              └─ operations.MoveTransfer()
```

### 2. 目录级 Move 优先 ([sync.MoveDir](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/sync/sync.go#L1403-L1432))

#### 阶段一：尝试服务端目录迁移（DirMove）

**触发条件：**
- 目标后端实现了 `Features.DirMove` 接口
- 源和目标**同配置**（`SameConfig`）
- 无激活的过滤器（`fi.InActive()`）

**执行与降级：**

```
调用 fdst.Features().DirMove(ctx, fsrc, "", "")
  ├─ 成功 → 直接返回，操作完成
  ├─ 返回 fs.ErrorCantDirMove → 降级到逐文件移动
  ├─ 返回 fs.ErrorDirExists → 降级到逐文件移动
  └─ 其他错误 → 直接返回错误（不降级）
```

#### 阶段二：降级到逐文件移动

调用 `moveDir()` → `runSyncCopyMove(DoMove=true)`，通过 march 遍历，每个文件进入 `pairCopyOrMove()` → `MoveTransfer()`。

### 3. 单文件 Move 详细流程 ([move](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/operations.go#L433-L519))

#### 阶段一：尝试服务端 Move

**触发条件：**
- 目标后端实现了 `Features.Move` 接口
- 源和目标**同配置**，或**同远程类型且支持跨配置服务端操作**

**前置处理：**
- 若目标已存在（dst != nil）且与源不是同一对象 → **先删除目标**
- 处理大小写不敏感文件系统的特殊情况（MoveCaseInsensitive）

**执行与降级：**

```
调用 doMove(ctx, src, remote)
  ├─ 成功 → 返回，操作完成
  ├─ 返回 fs.ErrorCantMove → 降级到 Copy + Delete
  └─ 其他错误 → 直接返回错误（不降级，不复制，不删除源）
```

#### 阶段二：降级到 Copy + Delete（对象迁移）

服务端 Move 不可用时：

1. **先 Copy**：调用 `Copy(ctx, fdst, dst, origRemote, src)` 将文件复制到目标
   - Copy 失败 → **不删除源文件**，直接返回错误
   - Copy 成功 → 继续
2. **后 Delete**：调用 `DeleteFile(ctx, src)` 删除源文件

> **关键原则**：只有 Copy 完全成功后才会删除源文件，确保数据不丢失。

### 4. Move 的同步匹配优化

在 `runSyncCopyMove` 的 [pairChecker](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/sync/sync.go#L450-L471) 中：

当 `DoMove=true` 且源和目标文件**不需要传输**（已匹配，内容相同）时：
- 源与目标为同一对象 → 跳过
- 设置了 `--ignore-existing` → 跳过
- 否则 → **直接删除源文件**（无需复制）

---

## 执行顺序总览图

### Copy

```
初始化 (partial 模式判定)
    │
    ▼
┌──────────────────────┐
│  尝试 serverSideCopy │──── ErrorCantCopy ────┐
└──────────────────────┘                         │
    │ 成功                                       ▼
    ▼                               ┌──────────────────────┐
  验证 (size/hash)                  │     manualCopy       │
    │ 失败                          │  (multi-thread /     │
    ▼                               │   rcat / updateOrPut)│
 删除目标                           └──────────────────────┘
    │ 成功                                       │
    ▼                                            ▼
 partial→最终 Move (若非 inplace)         重试 (若可重试)
    │ 失败
    ▼
 删除 partial 文件
```

### Move (目录)

```
尝试 DirMove (服务端目录迁移)
    │
    ├─ 成功 → 完成
    ├─ ErrorCantDirMove / ErrorDirExists → 降级
    └─ 其他错误 → 报错退出
    │
    ▼
逐文件处理 (runSyncCopyMove DoMove=true)
    │
    ▼
每个文件:
  尝试 serverSideMove
      │
      ├─ 成功 → 完成该文件
      ├─ ErrorCantMove → 降级为 Copy + Delete
      │                    ├─ Copy 失败 → 保留源，报错
      │                    └─ Copy 成功 → Delete 源
      └─ 其他错误 → 报错退出 (不降级)
```

### Move (单文件)

```
尝试 serverSideMove
    │
    ├─ 成功 → 完成
    ├─ ErrorCantMove → 降级
    └─ 其他错误 → 报错退出
    │
    ▼
降级: Copy 然后 Delete
    │
    ├─ Copy 失败 → 不删除源，返回错误
    └─ Copy 成功 → DeleteFile(src)
```

---

## 关键设计要点

### 1. 服务端操作优先原则
- **Copy**：优先 `serverSideCopy` → 降级 `manualCopy`
- **Move**：优先 `serverSideMove` → 降级 `Copy + Delete`
- **MoveDir**：优先 `serverSide DirMove` → 降级逐文件 `MoveTransfer`

### 2. 数据安全保障
- Move 降级时 **先 Copy 后 Delete**，Copy 失败绝不删除源
- Copy 使用 **partial 文件**，避免写入中途的损坏文件暴露为最终文件
- 注册 **atexit 钩子**，程序异常退出时清理 partial 文件

### 3. 失败回退策略
| 阶段 | Copy | Move |
|------|------|------|
| 服务端操作返回 CantCopy/CantMove | 降级到手动复制 | 降级到 Copy+Delete |
| 服务端操作返回其他错误 | 可重试则重试，否则删除 partial | 直接返回错误，不降级 |
| 手动复制过程出错 | 删除 partial 文件 | Copy 阶段出错则保留源文件 |
| 校验（size/hash）失败 | 删除已复制目标文件 | — |
| partial→最终 Move 失败 | 删除 partial 文件 | — |

### 4. 重试机制
- 仅 Copy 有内部重试循环（`LowLevelRetries` 次）
- Move 本身无内部重试，依赖外层同步框架的重试
- 可重试错误通过 `fserrors.IsRetryError` / `ShouldRetry` / `Retry-After` 判定

---

## 深度场景分析

### 场景一：复制成功后源对象删除失败

此场景仅发生在 **Move 操作的降级路径**（Copy + Delete）中。

#### 代码路径

[operations.go:512-518](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/operations.go#L512-L518)

```go
newDst, err = Copy(ctx, fdst, dst, origRemote, src)
if err != nil {
    fs.Errorf(src, "Not deleting source as copy failed: %v", err)
    return newDst, err
}
// Delete src if no error on copy
return newDst, DeleteFile(ctx, src)
```

#### 执行顺序分析

```
Copy 成功 (目标端已有完整副本)
    │
    ▼
调用 DeleteFile(ctx, src)
    │
    ├─ 成功 → Move 整体成功，err=nil
    └─ 失败 → Move 整体失败，返回 DeleteFile 的错误
```

#### 后果与数据状态

| 阶段 | 源文件 | 目标文件 | Move 返回值 |
|------|--------|----------|-------------|
| Copy 成功，Delete 成功 | 已删除 | 存在 | nil（成功） |
| Copy 成功，Delete 失败 | **仍然存在** | 存在 | err（失败） |
| Copy 失败 | 仍然存在 | 不存在/partial | err（失败） |

> **关键结果**：当 Copy 成功但 Delete 失败时，会出现**源文件和目标文件同时存在**的状态。此时 Move 返回错误，但数据实际上已经在目标端完整可用。用户需要手动处理源文件残留。

#### DeleteFile 的内部处理

[DeleteFileWithBackupDir](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/operations.go#L550-L577)：
- 支持 `--dry-run` 模式（跳过实际删除）
- 支持 `--backup-dir`（将文件移动到备份目录而非删除）
- 删除失败时通过 `fs.CountError` 累计错误计数
- 删除成功时记录日志 `"Deleted"`

---

### 场景二：目标已存在时的处理

Copy 和 Move 对"目标已存在"有**完全不同的处理逻辑**。

#### Copy 的处理

**入口判断**：[NeedTransfer](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/operations.go#L1733-L1801)

| 条件 | 是否传输 |
|------|----------|
| dst == nil（目标不存在） | 是 |
| `--ignore-existing` 设为 true | 否 |
| `--ignore-times` 设为 true | 是（无条件传输） |
| `--update-older` 且目标比源新 | 否 |
| size + modtime/hash 均匹配 | 否 |
| 上述均不满足 | 是 |

**实际写入时**（目标已存在且需传输）：
- [updateOrPut](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L213-L241) 区分两种模式：
  - `doUpdate && inplace` → 调用 `c.dst.Update(ctx, inAcc, wrappedSrc, ...)`（原地更新）
  - 其他情况 → 调用 `c.f.Put(ctx, inAcc, wrappedSrc, ...)`（覆盖写入）

- partial 模式下（非 inplace）：始终写入 `remoteForCopy`（partial 临时名），不直接覆盖目标。成功后通过 Move 原子重命名。

#### Move 的处理

**服务端 Move 路径**（[move:464-478](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/operations.go#L464-L478)）：

```go
if dst != nil {
    remote = transform.Path(ctx, dst.Remote(), false)
    if !SameObject(src, dst) {
        err = DeleteFile(ctx, dst)   // ← 先删除已存在的目标
        if err != nil {
            return newDst, err      // 删除失败直接返回，不进行 Move
        }
    } else if src.Remote() == remote {
        return newDst, nil          // 同一文件同名，无需操作
    } else if needsMoveCaseInsensitive(...) {
        // 使用 MoveCaseInsensitive 两步走
    }
}
```

**降级 Copy+Delete 路径**：
- 与 Copy 一致，由 Copy 的 `NeedTransfer` + `updateOrPut` 处理目标存在的情况
- Copy 成功后再删源

#### 大小写不敏感文件系统的特殊处理

[needsMoveCaseInsensitive](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/operations.go#L1963-L1970) 触发条件：
- 非 Copy 操作（cp=false）
- 源和目标同后端
- 目标文件系统大小写不敏感
- 源文件名和目标文件名仅大小写不同（如 `File.txt` vs `file.txt`）

此时使用 [MoveCaseInsensitive](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/operations.go#L1978-L2012) 三步走：
1. 生成随机临时名：`dstFileName + "-rclone-move-" + random.String(8)`
2. 第一步 Move：源 → 临时名
3. 第二步 Move：临时名 → 目标名

> **设计原因**：大小写不敏感文件系统中 `File.txt` 和 `file.txt` 被视为同一文件，直接 rename 可能导致文件丢失或被覆盖。

---

### 场景三：服务端操作错误降级

Copy、Move、MoveDir 各有独立的降级触发条件，**并非所有错误都会触发降级**。

#### 降级判定条件总览

| 操作 | 降级触发错误 | 其他错误处理 |
|------|-------------|-------------|
| Copy | `fs.ErrorCantCopy` | 可重试则重试，否则直接返回错误并清理 partial |
| Move | `fs.ErrorCantMove` | 直接返回错误，**不降级，不复制，不删源** |
| MoveDir | `fs.ErrorCantDirMove`, `fs.ErrorDirExists` | 直接返回错误，不降级 |

#### Copy 服务端复制降级细节

[serverSideCopy](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L139-L172) 中：
- 首先判定 `serverSideCopyOK`：
  - `SameConfig(src.Fs(), c.f)` → true
  - 或 `SameRemoteType(src.Fs(), c.f)` 且 `ServerSideAcrossConfigs` 为 true → true
- 条件不满足 → 直接返回 `fs.ErrorCantCopy`（触发降级）
- 调用后端 `doCopy` 后：
  - 返回 `fs.ErrorCantCopy` → 降级到手动复制，同时 `c.tr.Reset(ctx)` 重置 accounting
  - 其他错误 → 进入重试逻辑（不立即降级）

#### Move 服务端移动降级细节

[move](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/operations.go#L487-L506) 中：
```go
newDst, err = doMove(ctx, src, remote)
switch err {
case nil:
    // 成功，返回
case fs.ErrorCantMove:
    fs.Debugf(src, "Can't move, switching to copy")
    // ↓ 继续执行到降级 Copy+Delete
default:
    err = fs.CountError(ctx, err)
    fs.Errorf(src, "Couldn't move: %v", err)
    return newDst, err  // ← 其他错误直接返回，不降级
}
```

#### MoveDir 服务端目录迁移降级细节

[MoveDir](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/sync/sync.go#L1410-L1432) 中：
```go
err := fdstDirMove(ctx, fsrc, "", "")
switch err {
case fs.ErrorCantDirMove, fs.ErrorDirExists:
    fs.Infof(fdst, "Server side directory move failed - fallback to file moves: %v", err)
    // ↓ 降级到逐文件移动
case nil:
    return nil
default:
    err = fs.CountError(ctx, err)
    fs.Errorf(fdst, "Server side directory move failed: %v", err)
    return err  // ← 其他错误直接返回，不降级
}
```

#### SameConfig vs SameRemoteType

- [SameConfig](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/operations.go#L661-L663)：`fdst.Name() == fsrc.Name()` — 同一个配置名（rclone.conf 中同一节）
- [SameRemoteType](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/operations.go#L655-L657)：`fmt.Sprintf("%T", fdst) == fmt.Sprintf("%T", fsrc)` — 同一后端类型（如都是 S3，但可能不同账号/配置）

跨配置服务端操作需要后端显式声明 `Features.ServerSideAcrossConfigs = true` 或用户设置 `--server-side-across-configs`。

---

### 场景四：partial 文件命名和清理（含 suffix 超长处理）

#### 两段式校验：配置加载 vs 每次复制

`--partial-suffix` 的长度限制（>16 字符为非法）存在**两处独立校验**：

| 校验阶段 | 所在函数 | 代码位置 | 行为 |
|----------|----------|----------|------|
| 配置加载时 | `ConfigInfo.Reload()` | [config.go:728-731](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/config.go#L728-L731) | 返回错误，阻止配置生效 |
| 每次执行 Copy 时 | `(c *copy) checkPartial()` | [copy.go:96-97](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L96-L97) | 返回错误 |

配置加载时校验是第一道防线；运行时校验是兜底（防止绕过配置修改的调用方式）。

#### partial-suffix 超长时：错误返回与 inplace 的真实关系

这是最容易产生误解的地方。[checkPartial](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L91-L116) 的完整分支：

```go
func (c *copy) checkPartial(ctx context.Context) (remoteForCopy string, inplace bool, err error) {
    remoteForCopy = c.remote
    // 分支 A: 强制 inplace（正常情况，err=nil）
    if c.ci.Inplace || c.dstFeatures.Move == nil || !c.dstFeatures.PartialUploads || strings.HasSuffix(c.remote, ".rclonelink") {
        return remoteForCopy, true, nil
    }
    // 分支 B: suffix 超长（同时设 inplace=true 且返回 err）
    if len(c.ci.PartialSuffix) > 16 {
        return remoteForCopy, true, fmt.Errorf("expecting length of --partial-suffix to be not greater than %d but got %d", 16, len(c.ci.PartialSuffix))
    }
    // 分支 C: 正常 partial 命名
    // ... 生成 remoteForCopy，返回 inplace=false, err=nil
}
```

**关键事实**：分支 B 返回 `(remote, true, err)` 中 `inplace=true` **实际不会生效**。

查看调用方 [Copy:419-422](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L419-L422)：
```go
c.remoteForCopy, c.inplace, err = c.checkPartial(ctx)
if err != nil {
    return nil, err   // ← 只要 err != nil 就直接返回，c.inplace 被丢弃
}
```

所以后缀超长时的**真实行为**是：
1. `checkPartial` 返回错误
2. `Copy()` 直接 `return nil, err`，复制未开始
3. `c.inplace` 虽被设为 true，但因为函数立即返回，该值从未被使用
4. 没有文件写入、没有 partial 创建、没有清理动作

> **修正之前的理解**：不是"返回错误并强制 inplace"，而是**直接返回错误终止 Copy**。`inplace=true` 是一个死代码路径上的返回值，对执行无任何影响。

#### 正常 partial 模式 vs inplace 模式

以下情况触发 **inplace 模式**（直接写入最终文件名，不使用 partial），来自 [checkPartial:93-94](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L93-L94)：

| 条件 | 原因 |
|------|------|
| `--inplace` 为 true | 用户明确要求 |
| `dstFeatures.Move == nil` | 后端不支持重命名，无法把 partial 改成最终名 |
| `!dstFeatures.PartialUploads` | 后端不支持部分上传可见性控制（上传中途文件可能对用户可见） |
| 文件名后缀为 `.rclonelink` | 符号链接特殊处理 |

其他所有情况 → **partial 模式**。

#### 普通 partial 文件命名规则

命名逻辑在 [checkPartial:102-114](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L102-L114)：

```
完整后缀: "." + CRC32(remote + Fingerprint) + PartialSuffix
          |<-- 8 位十六进制 -->|   |<-- 默认 ".partial" -->|
```

**各部分说明**：

| 组成部分 | 计算方式 |
|----------|----------|
| `crc32_hash` | `CRC32(IEEE, remote + Fingerprint(ctx, src, true))`，输出 8 位十六进制 |
| `Fingerprint` | 基于 modtime、size、id 等生成的对象指纹，见 [fingerprint.go:21](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/fingerprint.go#L21) |
| `PartialSuffix` | 默认 `.partial`，由 `--partial-suffix` 配置 |

**文件名长度保护**：
```go
base := path.Base(remoteForCopy)
if len(base) > 100 {
    // 截断 remote 到 len(remote) - len(suffix)，再拼接 suffix
    remoteForCopy = TruncateString(remoteForCopy, len(remoteForCopy)-len(suffix)) + suffix
} else {
    remoteForCopy += suffix
}
```
- `TruncateString` 是 UTF-8 安全的截断，避免把多字节字符劈成两半
- 仅当文件名（不含目录）**basename > 100 字符**时才触发截断

**示例**：
| 原路径 | basename 长度 | partial 路径 |
|--------|-------------|-------------|
| `docs/report.pdf` | 10 ≤ 100 | `docs/report.pdf.a1b2c3d4.partial` |
| `a/verylongname_120chars....pdf` | 120 > 100 | `a/verylongname_120char....pdf.a1b2c3d4.partial`（前面被截断以容纳后缀） |

> **稳定性设计**：hash 基于 remote + 源文件指纹生成，同一文件的重试会使用**相同的 partial 文件名**，便于跨重试识别残留。

#### 清理函数的错误传播：全部吞掉，只记日志

两个清理函数都**没有返回值**，内部错误只通过 `fs.Infof` 记录：

[removeFailedCopy](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L45-L54)：
```go
func (c *copy) removeFailedCopy(ctx context.Context, o fs.Object) {
    if o == nil {
        return
    }
    fs.Infof(o, "Removing failed copy")
    err := o.Remove(ctx)
    if err != nil {
        fs.Infof(o, "Failed to remove failed copy: %s", err)
        // ← 没有 return err，错误被吞掉
    }
}
```

[removeFailedPartialCopy](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L57-L68)：
```go
func (c *copy) removeFailedPartialCopy(ctx context.Context, f fs.Fs, remote string) {
    o, err := f.NewObject(ctx, remote)
    if errors.Is(err, fs.ErrorObjectNotFound) {
        return   // 对象不存在 → 视为已清理
    }
    if err != nil {
        fs.Infof(remote, "Failed to remove failed partial copy: %s", err)
        return   // ← NewObject 失败只记日志
    }
    c.removeFailedCopy(ctx, o)   // ← 内部同样吞错误
}
```

**统一原则**：清理是 best-effort。清理失败不会影响主错误返回，也不会覆盖原始错误。用户只能通过日志发现 partial 文件残留。

#### 清理触发时机总览

| 触发点 | 条件 | 清理函数 | 代码位置 |
|--------|------|----------|----------|
| 复制主循环最终失败 | `err != nil && !c.inplace` | `removeFailedPartialCopy(ctx, c.f, c.remoteForCopy)` | [copy.copy:345-352](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L345-L352) |
| 程序异常退出（信号） | `!c.inplace`，manualCopy 内注册的 atexit 钩子 | `removeFailedPartialCopy(context.Background(), c.f, c.remoteForCopy)` | [copy.manualCopy:247-250](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L244-L251) |
| 复制成功但 size/hash 校验失败 | `verify()` 返回错误 | `removeFailedCopy(ctx, newDst)` | [copy.copy:354-361](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L354-L361) |
| partial→最终 Move（重命名）失败 | `dstFeatures.Move` 返回错误 | `removeFailedCopy(ctx, newDst)` | [copy.copy:364-374](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L364-L374) |
| 多线程复制 chunk 失败 | 任意 goroutine 返回错误 | `chunkWriter.Abort(ctx)`（通过 `atexit.OnError`） | [multithread.go:165-175](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/multithread.go#L165-L175) |

**注意**：校验失败和 partial→最终 Move 失败时，调用的是 `removeFailedCopy(newDst)` 而不是 `removeFailedPartialCopy()`，因为此时 `newDst` 已经是一个有效的 Object（无论是 partial 名还是最终名），直接对该 Object 调 Remove 即可，无需再按 remote 查找。

#### atexit 钩子的生命周期

[atexit.Register](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/lib/atexit/atexit.go#L32-L59) 的行为：
- 注册一个函数，在收到退出信号（SIGINT/SIGTERM 等）时执行
- `Unregister` 可在正常退出时取消

[copy.manualCopy:246-251](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/copy.go#L244-L251)：
```go
if !c.inplace {
    defer atexit.Unregister(atexit.Register(func() {
        ctx := context.Background()       // ← 注意：用全新 Background ctx，避免原 ctx 已取消
        c.removeFailedPartialCopy(ctx, c.f, c.remoteForCopy)
    }))
}
```

执行路径：
- **正常退出**：`defer Unregister` 在函数返回时执行，钩子被注销，不触发清理
- **信号退出**：atexit 的 signal handler 在 `defer Unregister` 之前抢到执行权，运行清理函数
- 注意使用 `context.Background()` 而不是函数参数中的 ctx，因为信号触发时原 ctx 可能已被取消

#### 多线程复制的 Abort 机制

[multiThreadCopy](file:///d:/fz/0601-2/solo-dogfeeding/code/57-rclone/fs/operations/multithread.go#L162-L175) 使用 `atexit.OnError`：
```go
defer atexit.OnError(&err, func() {
    cancel()
    if info.LeavePartsOnError || uploadedOK {
        return   // ← 配置了留片或已成功，不清理
    }
    abortErr := chunkWriter.Abort(ctx)   // ← 通知后端销毁未完成分片
    ...
})()
```

三种不清理的情况：
1. `info.LeavePartsOnError = true` —— 用户希望保留已上传分片用于断点续传
2. `uploadedOK = true` —— 所有 chunk 写完且 Close 成功
3. `err == nil` —— 函数正常返回，`OnError` 检测到 `*perr == nil` 不执行回调

---

## 综合时序：服务端复制降级 + 目标已存在 + partial 清理

以下是 Copy 操作覆盖最多边界条件的完整时序：

```
operations.Copy(fdst, dst=existing_obj, remote, src)
    │
    ├─ checkPartial()
    │   └─ inplace=false → remoteForCopy = "file.abc123def.partial"
    │
    └─ copy.copy() [主循环，LowLevelRetries 次]
        │
        ├─ serverSideCopy()
        │   ├─ SameRemoteType=true, ServerSideAcrossConfigs=false
        │   │   → serverSideCopyOK=false
        │   │   → 返回 fs.ErrorCantCopy
        │   └─ 触发降级
        │
        ├─ manualCopy()
        │   ├─ 注册 atexit → removeFailedPartialCopy
        │   ├─ dst!=nil 且 inplace=false → 不使用 Update
        │   │   → Put(ctx, inAcc, wrappedSrc) 写入 file.abc123def.partial
        │   │
        │   ├─ [情况A] Put 成功 → 返回复制对象
        │   │   │
        │   │   ├─ verify(size/hash)
        │   │   │   ├─ 成功 → 继续
        │   │   │   └─ 失败 → removeFailedCopy(partial) → 返回错误
        │   │   │
        │   │   └─ dstFeatures.Move(partial, "file.txt")
        │   │       ├─ 成功 → 覆盖原目标文件，完成
        │   │       └─ 失败 → removeFailedCopy(partial) → 返回错误
        │   │
        │   └─ [情况B] Put 失败且不可重试
        │       └─ removeFailedPartialCopy(partial) → 返回错误
        │
        └─ [重试] 若错误可重试，tr.Reset() 后重复上述流程
```
