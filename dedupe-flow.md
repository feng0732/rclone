# rclone dedupe 去重功能实现流程详解

## 概述

rclone 的 `dedupe` 命令用于查找并处理重复文件，主要针对 Google Drive、Mega 等允许同名文件存在的云存储后端。该功能支持两种去重模式：按名称（by name）和按哈希（by hash）。

核心代码位于：
- [cmd/dedupe/dedupe.go](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/cmd/dedupe/dedupe.go) — CLI 命令入口
- [fs/operations/dedupe.go](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go) — 核心业务逻辑

---

## 一、整体执行流程

```
Deduplicate() 入口
    │
    ├─ 1. 选择哈希类型
    │
    ├─ 2. (按名称模式) 查找并合并重复目录 ──┐
    │     dedupeFindDuplicateDirs()          │
    │     dedupeMergeDuplicateDirs()         │
    │                                         │
    ├─ 3. 遍历所有文件，按名称/哈希分组       │
    │     walk.ListR() → files map           │
    │                                         │
    └─ 4. 处理每组重复文件 ◄──────────────────┘
          │
          ├─ 4.1 (按名称模式) 删除哈希相同的副本
          │     dedupeDeleteIdentical()
          │
          └─ 4.2 根据 DeduplicateMode 执行保留策略
                interactive / skip / first / newest
                oldest / rename / largest / smallest / list
```

主入口函数见 [Deduplicate](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L402-L506)。

---

## 二、重复对象的识别

### 2.1 两种识别维度

由 `byHash` 参数控制（命令行 `--by-hash` 标志）：

| 模式 | 分组键 | 适用场景 |
|------|--------|----------|
| 按名称 (byHash=false) | `o.Remote()` 即文件远程路径 | Google Drive 等同名后端 |
| 按哈希 (byHash=true) | `o.Hash(ctx, ht)` 即文件内容哈希 | 任意支持哈希的后端 |

分组逻辑位于 [dedupe.go#L437-L462](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L437-L462)：

```go
files := map[string][]fs.Object{}
err := walk.ListR(ctx, f, "", false, ci.MaxDepth, walk.ListObjects, func(entries fs.DirEntries) error {
    entries.ForObject(func(o fs.Object) {
        var remote string
        if byHash {
            remote, err = o.Hash(ctx, ht)  // 按哈希分组
        } else {
            remote = o.Remote()            // 按路径名称分组
        }
        if remote != "" {
            files[remote] = append(files[remote], o)
        }
    })
    return nil
})
```

之后只处理 `len(objs) > 1` 的分组。

### 2.2 重复目录识别（仅按名称模式）

在处理文件之前，先处理同名目录。关键函数 [dedupeFindDuplicateDirs](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L281-L344)：

1. 递归遍历所有目录条目
2. 每个目录条目尝试获取：
   - `ParentID()`：通过 `fs.ParentIDer` 接口获取父目录 ID（如后端支持），否则回退到父路径
   - `ID()`：通过 `fs.IDer` 接口获取自身 ID，否则回退到路径
3. 以路径为 key 聚合目录，`len(dirs[remote]) > 1` 即判定为重复目录
4. 按路径字典序排序，确保父目录先于子目录处理

相关接口定义见 [fs/types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/types.go#L167-L176)：

```go
type IDer interface {
    ID() string
}
type ParentIDer interface {
    ParentID() string
}
```

---

## 三、保留策略实现

rclone 提供了 9 种去重模式，定义为 [DeduplicateMode](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L182-L195) 枚举：

```go
const (
    DeduplicateInteractive DeduplicateMode = iota  // 交互式
    DeduplicateSkip                                 // 跳过
    DeduplicateFirst                                // 保留第一个
    DeduplicateNewest                               // 保留最新
    DeduplicateOldest                               // 保留最旧
    DeduplicateRename                               // 改名
    DeduplicateLargest                              // 保留最大
    DeduplicateSmallest                             // 保留最小
    DeduplicateList                                 // 仅列出
)
```

### 3.1 前置步骤：自动删除内容完全相同的副本

**仅在按名称模式下执行**，位于 [dedupeDeleteIdentical](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L76-L138)。

该函数逻辑：

1. **安全检查**：先统计对象 ID 出现次数。若某 ID 在列表中出现多次（可能是 API 返回重复），则跳过这些对象以防误删数据。
2. **按内容分组**：
   - 若设置了 `--size-only`，则以 `size <大小>` 为分组 ID
   - 否则以 `<哈希类型> <哈希值>` 为分组 ID
   - 无法获取 ID 的对象直接加入 `remainingObjs`（不参与此轮删除）
3. **删除相同内容**：每组只保留第一个，其余调用 `DeleteFile` 删除；删除失败的对象也保留。

### 3.2 排序策略

对于基于时间或大小的策略，先排序再选择：

- **Oldest/Newest**：使用 [sortOldestFirst](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L386-L390)，按 `ModTime()` 升序排列（最旧在前）
  - Oldest：保留索引 0
  - Newest：保留索引 `len-1`

- **Smallest/Largest**：使用 [sortSmallestFirst](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L393-L397)，按 `Size()` 升序排列（最小在前）
  - Smallest：保留索引 0
  - Largest：保留索引 `len-1`

### 3.3 各模式最终行为

| 模式 | 操作 | 核心代码位置 |
|------|------|-------------|
| `interactive` | 列出后询问用户：跳过/保留一个/改名/退出 | [dedupeInteractive](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L161-L179) |
| `skip` | 什么也不做，仅日志输出 | [dedupe.go#L497-L498](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L497-L498) |
| `first` | 删除除第一个以外的所有副本 | `dedupeDeleteAllButOne(ctx, 0, ...)` |
| `newest` | 按时间升序排序，保留最后一个 | [dedupe.go#L483-L485](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L483-L485) |
| `oldest` | 按时间升序排序，保留第一个 | [dedupe.go#L486-L488](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L486-L488) |
| `rename` | 所有对象重命名，加数字后缀 | `dedupeRename()` |
| `largest` | 按大小升序排序，保留最后一个 | [dedupe.go#L491-L493](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L491-L493) |
| `smallest` | 按大小升序排序，保留第一个 | [dedupe.go#L494-L496](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L494-L496) |
| `list` | 仅打印重复项信息，不做任何修改 | [dedupeList](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L141-L158) |

---

## 四、改名操作实现

改名操作由 [dedupeRename](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L20-L56) 实现，仅在按名称模式下可用。

### 4.1 算法流程

```
对每个待改名对象 objs[i]:
    suffix = 1
    loop:
        newName = "<base>-<i+suffix><ext>"
        检查 newName 是否已存在
        若不存在 → 执行 Move 重命名，break
        若存在 → suffix++，继续尝试（最多100次）
```

### 4.2 关键细节

- **命名规则**：`file.txt` → `file-1.txt`, `file-2.txt`, ... 
  - 使用 `path.Ext()` 分离扩展名
  - 索引从 `i+1` 开始（i 为对象在重复组中的下标），避免与已存在的 `file-1.txt` 冲突
- **冲突检测**：通过 `f.NewObject(ctx, newName)` 检查目标路径，若返回 `fs.ErrorObjectNotFound` 则表示可用
- **上限保护**：suffix 超过 100 则放弃并报错
- **安全检查**：改名前调用 `SkipDestructive()` 尊重 `--dry-run` 和 `--interactive` 标志
- **后端要求**：必须实现 `Features().Move`，否则直接 Fatal 退出

---

## 五、删除操作实现

### 5.1 dedupeDeleteAllButOne

[dedupeDeleteAllButOne](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L59-L73) 是通用的"保留一个，删除其余"函数：

```go
func dedupeDeleteAllButOne(ctx context.Context, keep int, remote string, objs []fs.Object) {
    count := 0
    for i, o := range objs {
        if i == keep {
            continue
        }
        err := DeleteFile(ctx, o)
        if err == nil {
            count++
        }
    }
    if count > 0 {
        fs.Logf(remote, "Deleted %d extra copies", count)
    }
}
```

参数 `keep` 指定保留对象的数组下标。

### 5.2 DeleteFile 调用链

实际删除操作位于 [fs/operations/operations.go](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/operations.go#L550-L586)：

```
DeleteFile(ctx, obj)
  └─ DeleteFileWithBackupDir(ctx, obj, nil)
        │
        ├─ 1. 统计记账 (accounting.Stats)
        ├─ 2. SkipDestructive() 检查 --dry-run / --interactive
        │     └─ 若跳过则直接返回
        ├─ 3. 若设置了 backupDir → MoveBackupDir() 移动到备份目录
        └─ 4. 否则 → obj.Remove(ctx) 调用后端实际删除
```

### 5.3 破坏性操作安全检查

[SkipDestructive](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/operations.go#L2600-L2631) 是所有改名/删除操作的统一安全闸：

| 配置 | 行为 |
|------|------|
| `--dry-run` | 始终 skip，不执行实际操作 |
| `--interactive` / `-i` | 逐个询问用户是否执行，并记住选择 |
| 默认 | 不 skip，直接执行 |

### 5.4 目录合并删除

对于重复目录，调用后端特性 `Features().MergeDirs` 进行合并。在 [dedupeMergeDuplicateDirs](file:///d:/fz/0601-2/solo-dogfeeding/code/104-rclone/fs/operations/dedupe.go#L347-L383) 中：

- 将条目数最多的目录置于队首（以最小化文件移动次数）
- 调用后端 `MergeDirs(ctx, fsDirs)` 将其余目录内容合并到第一个
- 最后调用 `DirCacheFlush()` 刷新目录缓存

---

## 六、数据安全保障机制

dedupe 功能实现了多层防护避免数据丢失：

1. **重复 ID 过滤**：`dedupeDeleteIdentical` 中若后端返回的对象 ID 重复出现（可能是列表 API Bug），不会删除这些对象，防止删错。
2. **Dry-Run 模式**：所有破坏性操作均经过 `SkipDestructive` 检查。
3. **按名称模式自动去重相同内容**：先删除内容相同的副本（安全，因为内容一致），剩余才需要策略决策。
4. **改名冲突检测**：改名前先确认目标路径不存在，避免覆盖已有文件。
5. **删除失败保留**：`dedupeDeleteIdentical` 中删除失败的对象会被重新加入剩余列表，不会丢失。
