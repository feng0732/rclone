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
