# rclone dedupe 去重功能实现流程详解

## 概述

rclone 的 `dedupe` 命令用于查找并处理重复文件，主要针对 Google Drive、Mega 等允许同名文件存在的云存储后端。该功能支持两种去重模式：按名称（by name）和按哈希（by hash）。

核心代码位于：
- cmd/dedupe/dedupe.go — CLI 命令入口，负责参数解析和标志注册
- fs/operations/dedupe.go — 核心业务逻辑实现

---

## 一、命令入口与参数传递流程

### 1.1 包级变量与标志注册

命令入口的初始化部分位于 cmd/dedupe/dedupe.go#L15-L25：

```go
var (
    dedupeMode = operations.DeduplicateInteractive  // 默认模式：交互式
    byHash     = false                              // 默认按名称去重
)

func init() {
    cmd.Root.AddCommand(commandDefinition)
    cmdFlag := commandDefinition.Flags()
    // 注册 --dedupe-mode 标志，值通过 Set() 方法写入 dedupeMode
    flags.FVarP(cmdFlag, &dedupeMode, "dedupe-mode", "",
        "Dedupe mode interactive|skip|first|newest|oldest|largest|smallest|rename", "")
    // 注册 --by-hash 布尔标志，写入 byHash 变量
    flags.BoolVarP(cmdFlag, &byHash, "by-hash", "", false,
        "Find identical hashes rather than names", "")
}
```

### 1.2 Cobra Run 处理函数：从命令行到核心 Deduplicate

实际执行逻辑在 cmd/dedupe/dedupe.go#L151-L167，完整的参数传递链如下：

```
命令行: rclone dedupe [mode] remote:path [--by-hash] [--dedupe-mode xxx]
    │
    ├─ 1. CheckArgs(1, 2): 校验参数个数为 1~2 个
    │
    ├─ 2. 解析位置参数 mode（如果 len(args)>1）:
    │     dedupeMode.Set(args[0])
    │       → 根据字符串设置 DeduplicateMode（不区分大小写）
    │       → 未知模式返回错误并 Fatal 退出
    │     args = args[1:]   // 剩余的参数是远程路径
    │
    ├─ 3. fdst = cmd.NewFsSrc(args)  // 构造目标文件系统
    │
    ├─ 4. 后端能力检查 (仅按名称模式):
    │     if !byHash && !fdst.Features().DuplicateFiles {
    │         Log 警告: "Can't have duplicate names here. Perhaps --by-hash ?"
    │         // 不终止执行，仅提示
    │     }
    │
    └─ 5. cmd.Run() 中调用核心函数:
          operations.Deduplicate(ctx, fdst, dedupeMode, byHash)
          // 三个关键参数全部传入: 文件系统、去重模式、按哈希开关
```

### 1.3 模式字符串解析：DeduplicateMode.Set

fs/operations/dedupe.go#L222-L246 实现了模式从字符串到枚举的转换，不区分大小写：

```go
case "interactive" → DeduplicateInteractive   默认值
case "skip"        → DeduplicateSkip
case "first"       → DeduplicateFirst
case "newest"      → DeduplicateNewest
case "oldest"      → DeduplicateOldest
case "rename"      → DeduplicateRename
case "largest"     → DeduplicateLargest
case "smallest"    → DeduplicateSmallest
case "list"        → DeduplicateList
default           → 返回错误 "unknown mode for dedupe"
```

---

## 二、整体执行流程（Deduplicate 函数内部）

主入口函数见 fs/operations/dedupe.go#L402-L506。

```
Deduplicate(ctx, f, mode, byHash)
    │
    ├─ 1. 哈希类型选择
    │     ht := f.Hashes().GetOne()
    │     if byHash 且 ht == None { 返回错误 "%v has no hashes" }
    │
    ├─ 2. (仅按名称模式) 查找并合并重复目录
    │     ├─ dedupeFindDuplicateDirs() → duplicateDirs 列表
    │     └─ mode==list → 仅打印；否则 dedupeMergeDuplicateDirs()
    │
    ├─ 3. 遍历所有文件，按名称/哈希分组
    │     walk.ListR() → map[string][]fs.Object{分组键: [对象...]}
    │     ├─ 按哈希模式: 分组键 = o.Hash(ctx, ht)
    │     │     哈希失败 → remote="" → 跳过该对象（不计入任何分组）
    │     └─ 按名称模式: 分组键 = o.Remote()
    │
    └─ 4. 处理每组重复文件（len(objs) > 1 的分组）
          │
          ├─ 4.1 (仅按名称模式) 自动删除哈希相同的副本
          │     ├─ dedupeDeleteIdentical(ctx, ht, remote, objs)
          │     │     ├─ 先过滤掉 ID 重复出现的对象（防止 API 误删）
          │     │     ├─ 按 size/hash 分组 → 每组仅保留第一个
          │     │     └─ 剩余对象返回 remainingObjs
          │     └─ 若 len(remainingObjs) <= 1 → 日志 "All duplicates removed"，跳过后续
          │
          └─ 4.2 根据 mode 执行最终保留策略（switch 无 byHash 守卫，所有模式均会执行）
                interactive / skip / first / newest
                oldest / rename / largest / smallest / list
```

---

## 三、哈希相关完整路径

### 3.1 哈希类型选择：GetOne() 的实现

代码位于 fs/operations/dedupe.go#L404-L413：

```go
func Deduplicate(ctx context.Context, f fs.Fs, mode DeduplicateMode, byHash bool) error {
    ci := fs.GetConfig(ctx)
    // ① 选择哈希类型
    ht := f.Hashes().GetOne()      // 从后端支持的哈希集合中挑一个
    what := "names"
    if byHash {
        // ② 按哈希模式下，无可用哈希 → 直接报错返回
        if ht == hash.None {
            return fmt.Errorf("%v has no hashes", f)
        }
        what = ht.String() + " hashes"
    }
    fs.Infof(f, "Looking for duplicate %s using %v mode.", what, mode)
    // ...
}
```

`GetOne()` 的实现位于 fs/hash/hash.go#L346-L360，它从 `hash.Set`（位集合）中取最低位对应的哈希类型：

```go
func (h Set) GetOne() Type {
    v := int(h)
    i := uint(0)
    for v != 0 {
        if v&1 != 0 {
            return Type(1 << i)   // 返回第一个非零位对应的类型
        }
        i++
        v >>= 1
    }
    return None  // 集合为空，返回 None
}
```

哈希类型的注册顺序（决定 GetOne 的优先级）见 fs/hash/hash.go#L122-L132：
MD5 → SHA1 → Whirlpool → CRC32 → SHA256 → SHA512 → BLAKE3 → XXH3 → XXH128

因此如果后端同时支持 MD5 和 SHA256，`GetOne()` 会优先返回 MD5。

### 3.2 无哈希报错路径

**报错场景**：当 `byHash=true`（即使用 `--by-hash` 标志）但后端不支持任何哈希类型时：

```
触发条件: byHash=true && f.Hashes().GetOne() == hash.None
   ↓
返回 error: fmt.Errorf("%v has no hashes", f)
   ↓
由 cmd.Run() 捕获，输出错误并退出程序
```

**注意**：按名称模式（`byHash=false`）下，`ht == None` 是允许的。此时 `dedupeDeleteIdentical` 中的哈希判断会走到 `ID == ""` 分支，所有对象被加入 `remainingObjs`，等同于跳过了"自动删除相同内容副本"的步骤。

### 3.3 哈希失败跳过对象的路径

在文件分组阶段（按哈希模式），单个对象哈希计算失败会导致该对象被排除，不计入任何分组。

代码位于 fs/operations/dedupe.go#L443-L456：

```go
var remote string
var err error
if byHash {
    remote, err = o.Hash(ctx, ht)   // 计算哈希
    if err != nil {
        fs.Errorf(o, "Failed to hash: %v", err)   // 打印错误日志
        remote = ""                  // 标记为空字符串
    }
} else {
    remote = o.Remote()
}
// 关键判断：仅当 remote 非空时才加入分组
// 哈希失败时 remote == ""，不会进入 map → 该对象被完全跳过
if remote != "" {
    files[remote] = append(files[remote], o)
}
```

**完整的跳过路径**：
```
o.Hash(ctx, ht) 返回 err != nil
    ↓
remote = ""  (同时打印 Error 级别日志)
    ↓
remote != "" 判断为 false
    ↓
不写入 files map
    ↓
后续 for range files 循环中完全看不到该对象
    ↓
该对象不会被删除、改名，也不会被列为重复项
```

### 3.4 dedupeDeleteIdentical 中的哈希路径

在按名称模式下，`dedupeDeleteIdentical` 内部还有一层哈希处理，位于 fs/operations/dedupe.go#L104-L135：

```go
dupesByID := make(map[string][]fs.Object, len(objs))
for _, o := range objs {
    ID := ""
    if ci.SizeOnly && o.Size() >= 0 {
        // --size-only 标志：按大小作为分组 ID
        ID = fmt.Sprintf("size %d", o.Size())
    } else if ht != hash.None {
        // 正常路径：计算哈希
        hashValue, err := o.Hash(ctx, ht)
        if err == nil && hashValue != "" {
            ID = fmt.Sprintf("%v %s", ht, hashValue)
        }
        // 哈希 err != nil 或 hashValue == "" → ID 保持为 ""
    }
    if ID == "" {
        // 哈希失败 / 无哈希 / SizeOnly 条件不满足
        // → 不参与自动删除，直接加入剩余列表
        remainingObjs = append(remainingObjs, o)
    } else {
        dupesByID[ID] = append(dupesByID[ID], o)
    }
}
```

此处的跳过路径：
| 条件 | ID | 行为 |
|------|----|------|
| `--size-only` 且 `Size()>=0` | `"size <n>"` | 放入 dupesByID，参与删除 |
| 有哈希且计算成功无空值 | `"<type> <hash>"` | 放入 dupesByID，参与删除 |
| 哈希返回 error | `""` | 直接加入 remainingObjs，**跳过** |
| 哈希返回空字符串 | `""` | 直接加入 remainingObjs，**跳过** |
| 无可用哈希 (ht==None) | `""` | 全部跳过，函数等同于空操作 |

---

## 四、重复对象的识别

### 4.1 两种识别维度

由核心函数参数 `byHash bool` 控制（源自命令行 `--by-hash` 标志）：

| 模式 | 分组键 | 适用场景 |
|------|--------|----------|
| 按名称 (byHash=false) | `o.Remote()` 即文件远程路径 | Google Drive 等同名后端 |
| 按哈希 (byHash=true) | `o.Hash(ctx, ht)` 即文件内容哈希 | 任意支持哈希的后端 |

分组逻辑位于 fs/operations/dedupe.go#L437-L462。遍历结束后只处理 `len(objs) > 1` 的分组。

### 4.2 重复目录识别（仅按名称模式）

在处理文件之前，先处理同名目录。关键函数 dedupeFindDuplicateDirs（fs/operations/dedupe.go#L281-L344）：

1. 使用 `walk.ListR` 递归遍历所有目录条目
2. 每个目录条目尝试获取：
   - `ParentID()`：通过 `fs.ParentIDer` 接口获取父目录 ID（如后端支持），否则回退到父路径字符串
   - `ID()`：通过 `fs.IDer` 接口获取自身 ID，否则回退到路径字符串
3. 以路径 `remote` 为 key 聚合目录数组，`len(dirs[remote]) > 1` 即判定为重复目录
4. 按路径字典序排序（`sort.Strings`），确保父目录先于子目录被处理

相关接口定义见 fs/types.go#L167-L176：

```go
type IDer interface {
    ID() string                    // 返回对象/目录的后端内部 ID
}
type ParentIDer interface {
    ParentID() string              // 返回父目录 ID
}
```

---

## 五、保留策略实现

rclone 提供了 9 种去重模式，定义为 fs/operations/dedupe.go#L182-L195 枚举：

```go
const (
    DeduplicateInteractive DeduplicateMode = iota  // 交互式（默认）
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

### 5.1 前置步骤：自动删除内容完全相同的副本

**仅在按名称模式（`!byHash`）且模式不是 `DeduplicateList` 时执行**，入口位于 fs/operations/dedupe.go#L469-L475：

```go
if !byHash && mode != DeduplicateList {
    objs = dedupeDeleteIdentical(ctx, ht, remote, objs)
    if len(objs) <= 1 {
        fs.Logf(remote, "All duplicates removed")
        continue   // 已全部去重，直接进入下一组
    }
}
```

dedupeDeleteIdentical（fs/operations/dedupe.go#L76-L138）逻辑：

1. **ID 重复安全检查**：先统计 `fs.IDer.ID()` 出现次数。若某 ID 在列表中出现 >1 次（可能是后端列表 API 返回重复条目），则这些对象被排除出删除候选，防止误删。
2. **按内容分组**（详见上文 3.4 节）：
   - `--size-only` 标志：以 `size <大小>` 为分组 ID
   - 否则：以 `<哈希类型> <哈希值>` 为分组 ID
   - 无法获取 ID 的对象直接加入 `remainingObjs`（不参与此轮删除）
3. **删除相同内容**：每组 `dupes` 只保留 `dupes[0]`，`dupes[1:]` 调用 `DeleteFile` 删除；删除失败的对象重新 append 回 `remainingObjs`。

### 5.2 排序策略

对于基于时间或大小的策略，先排序再选择保留下标：

- **Oldest/Newest**：使用 sortOldestFirst（fs/operations/dedupe.go#L386-L390），按 `ModTime()` 升序（最旧在前）
  - Oldest → 保留下标 0
  - Newest → 保留下标 `len(objs)-1`

- **Smallest/Largest**：使用 sortSmallestFirst（fs/operations/dedupe.go#L393-L397），按 `Size()` 升序（最小在前）
  - Smallest → 保留下标 0
  - Largest → 保留下标 `len(objs)-1`

### 5.3 各模式最终行为总览

switch 语句（fs/operations/dedupe.go#L476-L503）**没有任何 `byHash` 守卫**，所有模式在按名称和按哈希两种去重方式下都会进入执行分支：

| 模式 | 操作 | 代码位置 |
|------|------|----------|
| `interactive` | 列出后询问用户：跳过(s) / 保留一个(k) / 改名(r,仅按名称) / 退出(q) | dedupeInteractive（fs/operations/dedupe.go#L161-L179） |
| `skip` | 不做任何操作，仅输出日志 | fs/operations/dedupe.go#L497-L498 |
| `first` | `dedupeDeleteAllButOne(ctx, 0, ...)` 删除除第一个以外的所有副本 | fs/operations/dedupe.go#L481-L482 |
| `newest` | 排序后保留最后一个（ModTime 最大） | fs/operations/dedupe.go#L483-L485 |
| `oldest` | 排序后保留第一个（ModTime 最小） | fs/operations/dedupe.go#L486-L488 |
| `rename` | 所有对象重命名，加数字后缀 | fs/operations/dedupe.go#L489-L490 |
| `largest` | 排序后保留最后一个（Size 最大） | fs/operations/dedupe.go#L491-L493 |
| `smallest` | 排序后保留第一个（Size 最小） | fs/operations/dedupe.go#L494-L496 |
| `list` | 仅打印重复项信息（大小/时间/哈希/路径），不修改 | dedupeList（fs/operations/dedupe.go#L141-L158） |

---

## 六、改名操作实现

改名操作由 dedupeRename（fs/operations/dedupe.go#L20-L56）实现。

### 6.1 按哈希模式下 rename 的实际行为

**关键事实**：switch 语句中 `case DeduplicateRename` 分支（fs/operations/dedupe.go#L489-L490）**没有 `byHash` 守卫**，在按哈希去重时同样会执行 `dedupeRename()`。

但这两种模式下 `remote` 参数的含义完全不同：

| 去重方式 | `remote` 值 | 改名结果 |
|---------|------------|---------|
| 按名称 (`byHash=false`) | 文件路径，如 `"photos/one.txt"` | `photos/one-1.txt`, `photos/one-2.txt` — 语义正确 |
| 按哈希 (`byHash=true`) | 哈希字符串，如 `"1eedaa9fe86fd4b8632e2ac549403b36"` | `1eedaa9fe86fd4b8632e2ac549403b36-1` — 无扩展名，文件名即哈希串 |

原因是 `dedupeRename` 使用 `path.Ext(remote)` 分离扩展名，而哈希字符串无扩展名，因此 `ext=""`, `base=哈希串`，新文件名形如 `<哈希>-<序号>`。这在功能上可行但语义上不合理——通常用户不会期望按哈希去重后文件被重命名为哈希串。

**交互模式下的区别**：`dedupeInteractive`（fs/operations/dedupe.go#L161-L179）仅在 `!byHash` 时才在菜单中显示 rename 选项：

```go
commands := []string{"sSkip and do nothing", "kKeep just one (choose which in next step)"}
if !byHash {
    commands = append(commands, "rRename all to be different (by changing file.jpg to file-1.jpg)")
}
commands = append(commands, "qQuit")
```

因此在交互模式下按哈希去重时，用户不会看到 rename 选项。但非交互的 `--dedupe-mode rename` 在按哈希模式下仍会执行。

### 6.2 算法流程

```
对重复组中每个对象 objs[i] (i 从 0 开始):
    suffix = 1
    outer:
    for {
        newName = "<base>-<i+suffix><ext>"
        // 例: i=0, suffix=1 → "file-1.txt"
        //     i=0, suffix=2 → "file-2.txt"

        _, err := f.NewObject(ctx, newName)  // 检查目标是否存在

        if err == fs.ErrorObjectNotFound {
            break  // 目标不存在，可用
        }
        if err != nil {
            // 除"不存在"以外的错误（如网络错误）
            // 记录错误，跳过该对象的改名
            continue outer
        }
        // err == nil → 目标已存在，suffix+1 重试
        suffix++
        if suffix > 100 {
            // 超过 100 次仍找不到可用名称，放弃
            continue outer
        }
    }

    if !SkipDestructive(ctx, o, "rename") {
        newObj, err := doMove(ctx, o, newName)  // 执行改名
    }
```

### 6.3 关键细节

- **命名规则**：`file.txt` → `file-1.txt`, `file-2.txt`, ...
  - 使用 `path.Ext()` 分离扩展名（保留后缀 `.txt`）
  - base = 去掉扩展名的部分
  - 数字起始值为 `i + suffix`，i 是对象在组中的下标，避免与已存在的同名加后缀文件冲突
- **冲突检测**：通过 `f.NewObject(ctx, newName)` 探测目标路径
  - 返回 `fs.ErrorObjectNotFound` → 路径可用
  - 返回 `nil`（对象存在）→ suffix++ 重试
  - 返回其他错误 → 放弃该对象改名
- **上限保护**：suffix 超过 100 则放弃该对象并输出 Error 日志 `Could not find an available new name`
- **安全检查**：改名前调用 `SkipDestructive()`，尊重 `--dry-run` 和 `--interactive/-i` 标志
- **后端要求**：`f.Features().Move` 必须非 nil，否则直接 `fs.Fatalf` 退出整个程序

---

## 七、删除操作实现

### 7.1 dedupeDeleteAllButOne

dedupeDeleteAllButOne（fs/operations/dedupe.go#L59-L73）是通用的"保留一个，删除其余"函数：

```go
func dedupeDeleteAllButOne(ctx context.Context, keep int, remote string, objs []fs.Object) {
    count := 0
    for i, o := range objs {
        if i == keep {
            continue              // 跳过要保留的对象
        }
        err := DeleteFile(ctx, o)
        if err == nil {
            count++
        }
        // 删除失败不做特殊处理（错误已在 DeleteFile 内部统计）
    }
    if count > 0 {
        fs.Logf(remote, "Deleted %d extra copies", count)
    }
}
```

参数 `keep` 指定要保留对象的数组下标。

### 7.2 DeleteFile 完整调用链

实际删除操作位于 fs/operations/operations.go#L550-L586：

```
DeleteFile(ctx, obj)
  └─ DeleteFileWithBackupDir(ctx, obj, nil)
        │
        ├─ 1. accounting.Stats.NewCheckingTransfer() 开始统计
        ├─ 2. accounting.Stats.DeleteFile() 记录删除大小
        │
        ├─ 3. SkipDestructive() 检查是否应跳过
        │     ├─ --dry-run → skip=true
        │     ├─ --interactive → 询问用户
        │     └─ 默认 → skip=false
        │
        ├─ 4. skip=true → 直接返回（仅打印 Skipped 日志）
        │
        ├─ 5. backupDir != nil → MoveBackupDir() 移动到备份目录
        │
        └─ 6. 否则 → obj.Remove(ctx) 调用后端接口实际删除
              成功 → 打印 "Deleted" Info 日志
              失败 → 打印 Error 日志并 fs.CountError()
```

### 7.3 破坏性操作安全检查

SkipDestructive（fs/operations/operations.go#L2600-L2631）是所有改名/删除/合并操作的统一安全闸：

| 配置标志 | 行为 |
|---------|------|
| `--dry-run` | 始终返回 `skip=true`，不执行实际操作，打印 Skipped 日志 |
| `--interactive` / `-i` | 逐个询问用户是否执行，并将选择缓存避免重复询问 |
| 均未设置 | 返回 `skip=false`，直接执行 |

### 7.4 目录合并

对于重复目录，调用后端特性 `Features().MergeDirs` 进行合并，实现位于 dedupeMergeDuplicateDirs（fs/operations/dedupe.go#L347-L383）：

1. 选择条目数最多（`d.count` 最大）的目录置于数组首位，最小化后续文件移动次数
2. `SkipDestructive` 检查是否跳过
3. 调用后端 `mergeDirs(ctx, fsDirs)` 将其余目录内容合并到第一个目录中
4. 所有重复目录处理完后，调用 `DirCacheFlush()` 刷新目录缓存

后端必须同时实现 `MergeDirs` 和 `DirCacheFlush` 两个 Feature，否则函数直接返回错误。

---

## 八、数据安全保障机制

dedupe 功能实现了多层防护避免数据丢失：

1. **重复 ID 过滤**：`dedupeDeleteIdentical` 中若后端返回的对象 ID 重复出现（可能是列表 API Bug），不删除这些对象，防止误删同一文件的多条记录。

2. **Dry-Run 模式**：所有破坏性操作均经过 `SkipDestructive` 检查，`--dry-run` 下绝对不执行修改。

3. **按名称模式自动去重相同内容**：先基于哈希删除内容相同的副本（数据一致，删除安全），剩余才需要策略决策，减少用户判断量。

4. **改名冲突检测**：改名前先通过 `NewObject` 确认目标路径不存在，避免覆盖已有文件。

5. **删除失败保留对象**：`dedupeDeleteIdentical` 中 `DeleteFile` 失败的对象会被重新 append 回 `remainingObjs`，不会从列表中丢失。

6. **哈希失败跳过而非报错**：单个对象哈希计算失败仅跳过该对象，不中断整个去重流程。

7. **按哈希模式下的后端兼容检查**：入口处若后端无哈希能力直接报错返回，不执行后续逻辑。
