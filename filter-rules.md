# Rclone 过滤规则系统深度解析

本文档沿着代码路径，从规则解析、匹配算法、文件遍历三个维度，逐层拆解过滤规则如何影响同步决策。

---

## 一、核心文件概览

过滤系统分布在以下关键文件中：

| 模块 | 核心文件 | 职责 |
|------|----------|------|
| 规则引擎 | [fs/filter/filter.go](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go) | Filter 结构体、Include 总入口 |
| 规则数据结构 | [fs/filter/rules.go](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/rules.go) | rule/rules 结构、parseRules |
| Glob 转译 | [fs/filter/glob.go](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/glob.go) | rsync 风格 glob → 正则表达式 |
| 命令行 Flags | [fs/filter/filterflags/filterflags.go](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filterflags/filterflags.go) | 注册所有过滤相关 flag |
| 目录列表过滤 | [fs/list/list.go](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/list/list.go) | DirSorted / filterDir 过滤入口 |
| 递归遍历 | [fs/walk/walk.go](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go) | Walk/ListR 遍历策略选择 |
| 同步匹配 | [fs/march/march.go](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/march/march.go) | 源/目的端双向对齐遍历 |
| 同步操作 | [fs/sync/sync.go](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/sync/sync.go) | run() 组装 march 并执行同步决策 |

---

## 二、规则解析：从命令行字符串 → 内部数据结构

### 2.1 顶层入口：NewFilter 构造流程

一切始于 [NewFilter](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L188-L261)。它把 `Options` 结构体（用户通过 flags 传入的配置）转换为 `Filter` 对象。

关键步骤：

```
NewFilter(opt)
    │
    ├─ 1. 解析时间窗口（MinAge/MaxAge → ModTimeTo/ModTimeFrom）
    │
    ├─ 2. 解析哈希分片（HashFilter → hashFilterK/hashFilterN）
    │
    ├─ 3. parseRules(RulesOpt) ──┐
    │                             │ 构建 fileRules / dirRules
    │                             ▼
    │                        调用 f.Add(Include, glob)
    │
    ├─ 4. parseRules(MetaRules) ───── 构建 metaRules
    │
    ├─ 5. 处理 --files-from / --files-from-raw
    │      遍历文件，每行 → f.AddFile()
    │         └─ 填充 f.files (FilesMap)
    │         └─ 推导父目录填充 f.dirs
    │
    └─ 6. 若启用 --dump filters，打印最终规则
```

### 2.2 规则标志的三大类

[Options](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L142-L155) 中包含三个维度的过滤配置：

- **文件/目录规则**（嵌入 `RulesOpt`）：`--filter`、`--include`、`--exclude` 等
- **元数据规则**（`MetaRules RulesOpt`）：`--metadata-include` 等
- **物理属性**：`MinAge/MaxAge`、`MinSize/MaxSize`、`ExcludeFile`、`HashFilter`、`FilesFrom`

### 2.3 parseRules：顺序决定默认行为

[parseRules](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/rules.go#L190-L254) 是规则解析的核心，其处理顺序至关重要：

```go
func parseRules(opt *RulesOpt, add addFn, clear clearFn) error {
    addImplicitExclude := false   // 只要出现 --include，末尾要补排除所有
    foundExcludeRule := false

    // 阶段 1：先加所有 include 规则
    for _, rule := range opt.IncludeRule { add(true, rule); addImplicitExclude = true }
    for _, rule := range opt.IncludeFrom { ... }

    // 阶段 2：再加所有 exclude 规则
    for _, rule := range opt.ExcludeRule { add(false, rule); foundExcludeRule = true }
    for _, rule := range opt.ExcludeFrom { ... }

    // 阶段 3：加 --filter 规则（带 + / - / ! 前缀，按序处理）
    for _, rule := range opt.FilterRule { addRule(rule, add, clear) }
    for _, rule := range opt.FilterFrom { ... }

    // 阶段 4：若存在 include → 末尾补一条 "排除一切"
    // 关键：只要用了 --include，默认排除所有未明确 include 的
    if addImplicitExclude {
        add(false, "/**")   // 等价于 - /**
    }
}
```

> ⚠️ **关键设计**：只要使用了 `--include` 系列，系统会在规则列表**末尾**自动追加一条 `- /**`（排除所有路径）。这是理解 include 与 exclude 语义不对称的根源。

如果同时使用 `--include` 和 `--exclude`，系统会输出警告：

> "Using --filter is recommended instead of both --include and --exclude
as the order they are parsed in is indeterminate"

### 2.4 Add 单条规则：glob 如何分流到 fileRules + dirRules

[Filter.Add](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L314-L345) 是单条 glob 的分发器：

```
Add(Include, glob):
    │
    ├─ 规则以 / 结尾 → isDirRule=true
    │     └─ 若为排除：glob += "**"      （排除 "dir/" 等效于 "dir/**"）
    │
    ├─ glob 含 **    → isDirRule=isFileRule=true（同时影响文件和目录）
    │
    ├─ GlobPathToRegexp(glob) → 编译为 *regexp.Regexp
    │
    ├─ 若是文件规则（isFileRule）：
    │     ├─ fileRules.add(Include, re)
    │     └─ 若 Include=true 或 glob=="*"：
    │           └─ addDirGlobs(Include, glob)  ── 推导目录规则
    │
    └─ 若是目录规则（isDirRule）：
          └─ dirRules.add(Include, re)
```

**目录规则推导函数 [globToDirGlobs](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/glob.go#L218-L257)：
- 目的：从文件 glob "a/b/c/*.go" → 推导出可能需要扫描的目录
  - "a/b/c/"、"a/b/"、"a/"、"/**"
- 这样遍历到 "a/b/x/" 这种不含目标文件的目录可以提前跳过

### 2.5 Glob → 正则：globToRegexp 转译规则

[globToRegexp](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/glob.go#L41-L206) 将 rsync 风格 glob 编译为 Go regexp：

| Glob 语法 | 转译结果（pathMode=true） | 说明 |
|-----------|---------------------------|------|
| 以 `/` 开头 | `^` + ... | 锚定根目录 |
| 不以 `/` 开头 | `(^|/)` + ... | 可在任意层级匹配 |
| `*` | `[^/]*` | 单层通配，不跨 `/` |
| `**` | `.*` | 跨目录通配 |
| `?` | `[^/]` | 单字符 |
| `{a,b}` | `(a\|b)` | 选择组 |
| `{{regexp}}` | 原生 regexp | 内嵌正则 |
| 其余正则元字符 `.+()|^$` | `\` 转义 | 保持字面量 |

**锚定：结尾补 `$`**

示例：
- `*.log` → `(^|/)[^/]*\.log$`
- `/src/**/*.go` → `^src/.*[^/]*\.go$`

### 2.6 rules 内部存储

[rules](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/rules.go#L45-L48) 是规则列表容器：

```go
type rules struct {
    rules    []rule              // 顺序敏感的规则数组
    existing map[string]struct{}   // 去重用
}

type rule struct {
    Include bool               // true=include, false=exclude
    Regexp  *regexp.Regexp
}
```

通过 `rules.add()` 追加，按出现顺序存储。**顺序即优先级**——第一个命中的规则决定结果。

---

## 三、匹配逻辑：如何判断文件/目录是否被过滤

### 3.1 总入口：Filter.Include（七层决策链

[Filter.Include](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L504-L549) 是对象级别的综合判断，检查顺序严格如下（早返回 false）：

```
Include(remote, size, modTime, metadata) → bool
    │
    ├─ 1. --files-from 优先（f.files != nil）
    │     └─ remote ∈ f.files ? true : false
    │
    ├─ 2. ModTime 下限：modTime < ModTimeFrom → false（太新）
    ├─ 3. ModTime 上限：modTime > ModTimeTo   → false（太老）
    ├─ 4. Size 下限：size < MinSize → false
    ├─ 5. Size 上限：size > MaxSize → false
    ├─ 6. 元数据规则（若存在）
    │     └─ metaRules.includeMany(["k=v", ...)
    │
    └─ 7. 路径匹配：IncludeRemote(remote)
          ├─ 哈希分片（hashFilterN != 0）
          │   └─ md5(lowercase(NFC(remote))) % N == K
          │
          └─ fileRules.include(remote)
```

### 3.2 rules.include：首个命中原则

[rules.include](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/rules.go#L93-L100) 的实现极其简洁但语义核心：

```go
func (rs *rules) include(remote string) bool {
    for _, rule := range rs.rules {
        if rule.Match(remote) {
            return rule.Include  // 第一条匹配立即返回
        }
    }
    return true  // 无任何规则命中 → 默认**通过**
}
```

> 💡 **关键默认行为：**
> - 没有任何规则匹配时返回 `true`（通过）。
> - 但只要用了 `--include`，parseRules() 会在末尾补 `- /**`，所以实际上"未 include 的都会被那条排除。

### 3.3 IncludeRemote：路径匹配 + 哈希分片

[IncludeRemote](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L418-L440) 只关心路径：

```go
func (f *Filter) IncludeRemote(remote string) bool {
    if f.files != nil { _, include := f.files[remote]; return include }
    if f.hashFilterN != 0 {
        normalized := norm.NFC.String(remote)
        normalized = strings.ToLower(normalized)
        hash := binary.LittleEndian.Uint64(md5.Sum([]byte(normalized))
        if hash % f.hashFilterN != f.hashFilterK { return false }
    }
    return f.fileRules.include(remote)
}
```

哈希分片 `--hash-filter k/n`（或 `@/n` 随机）：对路径做 MD5 取模，只保留落在第 k 份的文件，用于并行分片同步。

### 3.4 IncludeDirectory：目录过滤决策

[IncludeDirectory](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L461-L482) 返回闭包函数，供遍历算法调用：

```
IncludeDirectory(ctx, fs)(remote) → (bool, error):
    │
    ├─ 1. exclude_if_present 检查：目录内是否存在标记文件
    │     └─ DirContainsExcludeFile → true 则排除
    │
    ├─ 2. --files-from 模式：remote ∈ f.dirs
    │
    └─ 3. 普通模式：remote 尾部补 "/" 后匹配 dirRules
          └─ dirRules.include(remote + "/")
```

### 3.5 元数据匹配：includeMany

[includeMany](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/rules.go#L108-L115) 是"任一命中即生效"：

```go
func (rs *rules) includeMany(remotes []string) bool {
    for _, rule := range rs.rules {
        if slices.ContainsFunc(remotes, rule.Match) {
            return rule.Include
        }
    }
    return true
}
```

元数据被展开为 `["key1=val1", "key2=val2", ...]`，只要**任何一条 k=v 字符串匹配规则就返回。

---

## 四、文件遍历：过滤规则如何嵌入同步流程

### 4.1 同步入口：sync.Sync → runSyncCopyMove → s.run() → march.Run

```
sync.Sync(ctx, fdst, fsrc)
    │
    ▼
syncCopyMove.run()                          [sync.go L936]
    │
    ├─ startCheckers/Transfers/Deleters...
    │
    └─ march.March{
    │    Ctx: inCtx,
    │    Fdst/Fsrc: 源目的,
    │    Callback: s（syncCopyMove 本身实现 Marcher）,
    │    DstIncludeAll: s.fi.Opt.DeleteExcluded,  // --delete-excluded 时为 true
    │    ...
    │  }.Run(ctx)
    │
    ▼
march.Run()
    │
    ├─ init():
    │   ├─ srcListDir = makeListDir(Fsrc, SrcIncludeAll=false)  ← 源端应用过滤
    │   └─ dstListDir = makeListDir(Fdst, DstIncludeAll)       ← 目的端：
    │             DstIncludeAll = DeleteExcluded ? true : false
    │
    └─ processJob():
        ├─ srcListDir(job.srcRemote, ...)  ← 列出源端条目（已过滤）
        ├─ dstListDir(job.dstRemote, ...)  ← 列出目的端条目
        │
        └─ matchListings(): 按名称对齐
             ├─ SrcOnly(src)  → 进入 Checker → 需要传输
             ├─ DstOnly(dst)   → Deleter 队列 → 删除（视 DeleteMode）
             └─ Match(dst,src) → 检查是否需要更新
```

> ⚠️ **DstIncludeAll 的设计意图**：当 `--delete-excluded=true` 时，目的端**不应用过滤，完整列出。这样 march 才能看到"被源端排除的文件"在目的端存在，并把它们视为 DstOnly 从而删除。这是理解 delete-excluded 的实现关键。

### 4.2 march.makeListDir：两条遍历路径

[makeListDir](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/march/march.go#L123-L171) 根据配置选择遍历策略：

```
makeListDir(f, includeAll, keyFn):
    │
    ├─ 若 !UseListR 且 !(--no-traverse + files-from)：
    │     普通逐目录：
    │     list.DirSortedFn(...)
    │        └─ 每个目录实时过滤（filterDir）
    │
    └─ 否则（--fast-list 或 files-from + no-traverse）：
          walk.NewDirTree() 一次性构建完整目录树
             └─ 然后按需取出各目录
```

关键点：在调用 list / walk 时，设置 ctx 上设置 `SetUseFilter` 传递 filter-aware 后端（如 Google Drive）可以在服务端直接过滤。

### 4.3 walk.Walk：三大遍历策略

[Walk](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go#L65-L77) 是所有列表遍历的总入口：

```
Walk(ctx, f, path, includeAll, maxLevel, fn):
    │
    ├─ 路径 1：NoTraverse+FilesFrom
    │   walkR(..., fi.MakeListR(...))
    │   直接从 files-from 列表生成对象
    │
    ├─ 路径 2：UseListR + f.Features().ListR != nil（--fast-list）
    │   walkListR → walkR → walkRDirTree
    │      用 ListR 一次性递归列出，然后 walkRDirTree 组装
    │
    └─ 路径 3：逐目录列出（默认）
        walkListDirSorted → walk(..., list.DirSorted)
        每个子目录独立 List()
```

### 4.4 list.DirSorted：逐目录过滤

[DirSorted](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/list/list.go#L24-L40)：

```
DirSorted(ctx, f, includeAll, dir) → DirEntries:
    │
    ├─ f.List(ctx, dir)        ← 后端列出原始条目
    │
    ├─ fi.ListContainsExcludeFile(entries)  ← 根目录 exclude_if_present
    │
    └─ filterAndSortDir(ctx, entries, includeAll, dir,
                        fi.IncludeObject, fi.IncludeDirectory)
          │
          ├─ Object：!includeAll → IncludeObject(ctx, x)
          │         └─ 调 Filter.Include()（7 层检查链）
          │
          └─ Directory：!includeAll → IncludeDirectory(remote)
                    └─ 调 dirRules.include()
```

[filterDir](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/list/list.go#L99-L156) 中的过滤发生在"内存中"就地过滤 entries[:0]"，原地修改 slice，不分配新内存。

### 4.5 walk.listR：ListR 模式的过滤

当后端支持递归列出（ListR）时，[listR](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go#L288-L343) 在回调中逐条过滤：

```go
doListR(ctx, path, func(entries fs.DirEntries) error {
    // ...
    if !includeAll {
        filteredEntries := entries[:0]
        for _, entry := range entries {
            switch x := entry.(type) {
            case fs.Object:
                include = fi.IncludeObject(ctx, x)
            case fs.Directory:
                include, err = includeDirectory(x.Remote())
            }
            if include { filteredEntries = append(filteredEntries, entry) }
        }
        entries = filteredEntries
    }
    return fn(entries)
})
```

### 4.6 --files-from 模式：绕过所有规则

`--files-from` / `--files-from-raw` 具有最高优先级，[NewFilter](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L227-L253) 会强制要求此时其他规则都不激活：

```go
for _, rule := range f.Opt.FilesFrom {
    if !inActive { return error("... overrides all other filters ...") }
    f.initAddFile()
    forEachLine(rule, false, f.AddFile)
}
```

此时 `f.files` 与 `f.dirs` 被填充，后续所有 Include 检查的第一步就命中 files map，直接返回。

---

## 五、同步决策：过滤如何影响 SrcOnly / DstOnly / Match

### 5.1 march.matchListings：三路分叉点

[matchListings](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/march/march.go#L292-L374) 将源/目的两侧按排序后的条目逐个比较，根据过滤已经在 listDir 阶段应用完毕。

过滤的影响体现在：

| 场景 | 源端过滤结果 | 目的端过滤结果 | match 结果 | 同步动作 |
|------|-------------|---------------|-----------|---------|
| 文件被源端排除 | ❌ 不在 srcList | ✅ 在 dstList（DstIncludeAll=false） | 不出现 | 保留（不删） |
| 文件被源端排除+--delete-excluded | ❌ 不在 srcList | ✅ 在 dstList（DstIncludeAll=true） | DstOnly | **删除** |
| 文件被目的端排除（例如目的侧 include 限制） | ✅ 在 srcList | ❌ 不在 dstList | SrcOnly | 复制 |
| 文件被两端都排除 | ❌ | ❌ | 都不出现 | 忽略 |
| 文件被源通过 | ✅ | ✅ | Match | 按需更新 |

### 5.2 目录级剪枝：ErrorSkipDir

[walk.walkRDirTree](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go#L459-L548) 在构建 DirTree 时，`IncludeDirectory` 返回 false 的目录根本不会进入。

这就是 `addDirGlobs() 推导出的 dirRules 的价值：

- `--include "/a/b/*.go"` 推导出目录 `"/a/b/"`, `"/a/"`, `"/"`
- 遍历到 `/c/` 时，dirRules 命中 exclude → 整个目录不再递归

### 5.3 UsesDirectoryFilters：决定遍历方式选择

[UsesDirectoryFilters](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L662-L672) 判断 `(dirRules 存在且不是简单的 include /** 时返回 true。

用于 [ListR](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go#L149-L164) 的决策：

```go
if fi.UsesDirectoryFilters() {
    // 不能用后端的 ListR，需要回退到 walk（逐目录），因为 ListR 无法对后端对目录做细粒度剪枝
    return listRwalk(...)
}
```

原因：ListR 递归列出所有，然后在回调里再过滤就太迟了——已经列出全部条目。对于有目录剪枝只能逐目录，先过滤后再决定是否进入子目录。

---

## 六、全局数据流全景图

```
命令行 flags
    │
    ▼
filterflags.AddFlags ──► Options 结构体
    │
    ▼
filter.NewFilter(opt)
    │
    ├─ parseRules(RulesOpt)
    │     ├─ Include* → f.Add(true, glob)
    │     ├─ Exclude* → f.Add(false, glob)
    │     ├─ Filter*  → addRule("+ / - / !")
    │     └─ [include → add(false, "/**")  ← 隐式排除
    │
    └── fileRules = rules{[]rule{Include, *Regexp}}
    └── dirRules  = rules{...}
    │
    ▼
march.makeListDir()
    │
    ├── list.DirSorted / walk.NewDirTree
    │         │
    │         ▼
    │     filterDir()
    │         │
    │         ├── Object: fi.IncludeObject
    │         │     ├─ files-from?
    │         │     ├─ ModTime?
    │         │     ├─ Size?
    │         │     ├─ Metadata?
    │         │     ├─ HashFilter?
    │         │     └─ fileRules.include(path)
    │         │           ▲
    │         │           （首条命中规则
    │         │           决定结果
    │         │
    │         └── Directory: fi.IncludeDirectory
    │                   ├─ exclude_if_present?
    │                   ├─ files-from (dirs)?
    │                   └── dirRules.include(path+"/")
    │
    ▼
march.matchListings()
    │
    ├─ SrcOnly → copy/move
    ├─ DstOnly → delete（if deleteMode）
    └─ Match   → check → update or skip
```

---

## 七、容易踩坑与常见误区

1. **`--include` 和 `--exclude` 顺序的顺序是陷阱**

   `parseRules 先 include 后 exclude，再在末尾补排除所有。结果是 exclude 写在 include 之前的 exclude 会被末尾的排除所有覆盖。正确方式：用 `--filter` + `- 前缀写在末尾。

2. **"目录以 `/` 结尾的 glob**

   `Add()` 将 `"dir/"` → 自动变成 `dir/**`，会同时加一条 exclude，都加一条 exclude `dir/` 目录本身。

3. **DstIncludeAll=DeleteExcluded 的意义**

   只看源端不理解"不应用规则，只有这样才能在目的端看到"源端排除的文件"才能被视为 DstOnly 删除。

4. **没有规则命中默认通过**

   rules.include() 无匹配返回 true。如果不用 include 系列，什么都不禁用——默认行为就不排除任何东西。

5. **files-from 与其他规则互斥**

   只要其他规则激活（非激活，files-from 报错。这是强制的。

6. **addDirGlobs 对 exclude 不推导目录**

   Add 中 `Include=false（排除规则）时不会调用 addDirGlobs。所以排除 `--exclude /secret/**` 不会推导出目录剪枝——只能在文件级别过滤。这解释了"为什么 exclude 比 exclude 比 glob 更高效。
