# Rclone 过滤规则补正：源端/目的端过滤差异与目录剪枝机制

本文档是 `filter-rules.md` 的补充与纠正，聚焦两个最容易误解的主题：
1. **源端 vs 目的端在同步时的过滤策略差异**（特别是 `--delete-excluded` 的真实工作原理）
2. **目录过滤如何决定遍历策略选择（ListR vs Walk）以及剪枝实现**

---

## 一、核心纠正：March 结构体的 SrcIncludeAll/DstIncludeAll 实际值

### 1.1 March 构建处的代码证据

在 [sync.go L954-L964](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/sync/sync.go#L954-L964) 中构造 March：

```go
m := &march.March{
    Ctx:                    s.inCtx,
    Fdst:                   s.fdst,
    Fsrc:                   s.fsrc,
    Dir:                    s.dir,
    NoTraverse:             s.noTraverse,
    Callback:               s,
    DstIncludeAll:          s.fi.Opt.DeleteExcluded,   // 只设置了这一个
    NoCheckDest:            s.noCheckDest,
    NoUnicodeNormalization: s.noUnicodeNormalization,
    // 注意：SrcIncludeAll 字段未显式赋值！
}
```

对比 [March 结构体定义](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/march/march.go#L32-L49)：

```go
type March struct {
    // ...
    SrcIncludeAll          bool  // 注释写着 "don't include all files in the src" — 实际语义相反！
    DstIncludeAll          bool  // 注释写着 "don't include all files in the destination" — 实际语义相反！
    // ...
}
```

> ⚠️ **字段注释与实际语义完全相反**：
> - `includeAll = true` → "包含所有文件" = **不应用过滤**
> - `includeAll = false` → 只包含通过过滤规则的 = **应用过滤**

### 1.2 同步（sync/copy/move）的真实过滤矩阵

基于默认值（SrcIncludeAll 未赋值 = false，Go 零值），得到：

| 场景 | SrcIncludeAll | DstIncludeAll | 源端行为 | 目的端行为 |
|------|:------------:|:------------:|----------|-----------|
| 默认（无 delete-excluded） | **false** | **false** | 应用过滤 | **应用过滤** |
| `--delete-excluded` | **false** | **true** | 应用过滤 | **不应用过滤** |

这是最关键的纠正：**默认情况下源端和目的端都会应用过滤**！之前版本中对目的端行为的描述不准确。

### 1.3 其他调用者的比较

| 调用位置 | SrcIncludeAll | DstIncludeAll | 说明 |
|---------|:------------:|:------------:|------|
| [sync.go](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/sync/sync.go#L954-L964) | 零值 false | `= DeleteExcluded` | 同步/复制/移动 |
| [check.go L224-L232](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/operations/check.go#L224-L232) | 零值 false | 零值 false | check 操作两端都过滤 |
| [bisync/march.go L33-L43](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/cmd/bisync/march.go#L33-L43) | 零值 false | 显式 false | bisync 两端都过滤 |

**没有任何调用者将 SrcIncludeAll 设为 true。** 源端永远应用过滤，目的端只有 `--delete-excluded` 时才不过滤。

### 1.4 includeAll 参数在过滤函数中的使用

在 [makeListDir](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/march/march.go#L123-L171) 中将 includeAll 传递给两种路径：

```go
// 路径 A：逐目录模式
return list.DirSortedFn(dirCtx, f, includeAll, dir, callback, keyFn)
                                 ▲

// 路径 B：DirTree 一次性构建模式
dirs, dirsErr = walk.NewDirTree(dirCtx, f, m.Dir, includeAll, ci.MaxDepth)
                                                ▲
```

最终在 [filterDir](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/list/list.go#L99-L156) 中生效：

```go
func filterDir(ctx context.Context, entries fs.DirEntries, includeAll bool, ...) {
    for _, entry := range entries {
        switch x := entry.(type) {
        case fs.Object:
            if !includeAll && !IncludeObject(ctx, x) {   // includeAll=true 直接跳过过滤
                ok = false
            }
        case fs.Directory:
            if !includeAll {
                include, _ := IncludeDirectory(x.Remote())
                if !include { ok = false }
            }
        }
    }
}
```

---

## 二、同步决策矩阵：纠正后的过滤影响

### 2.1 默认模式（无 --delete-excluded）

两端都应用过滤。过滤规则是**全局上下文共享**的（`filter.GetConfig(ctx)` 从 context 读取同一个 Filter 对象）：

假设规则：`--include "*.jpg"`（会自动追加末尾的排除所有）

| 文件 | 源端列表（SrcIncludeAll=false） | 目的端列表（DstIncludeAll=false） | match 结果 | 同步动作 |
|------|:---------------------------:|:---------------------------:|-----------|---------|
| `a.jpg` | ✅ 存在 | ✅ 存在 | Match | 按需更新（若大小/时间不同） |
| `b.png` | ❌ 不存在（被排除） | ❌ 不存在（被排除） | — | **完全消失，不被处理** |
| 仅源端存在的 `c.jpg` | ✅ 存在 | ❌ 不存在 | SrcOnly | **复制到目的** |
| 仅目的端存在的 `d.jpg` | ❌ 不存在 | ✅ 存在 | DstOnly | **删除（如果 deleteMode 开启）** |
| 仅目的端存在的 `e.png` | ❌ 不存在 | ❌ 不存在 | — | **保留，不被删除！** |

**关键点：默认模式下，目的端被排除的文件不会出现在 dstList 中，因此不会被当作 DstOnly 而删除。这就是"被排除文件默认保留在目的端"的原理。**

### 2.2 --delete-excluded 模式

目的端 DstIncludeAll=true（完整列出）：

假设规则：`--include "*.jpg" --delete-excluded`

| 文件 | 源端列表 | 目的端列表 | match 结果 | 同步动作 |
|------|:------:|:------:|-----------|---------|
| `a.jpg` | ✅ | ✅ | Match | 按需更新 |
| `b.png` | ❌ | ✅ | **DstOnly** | **被删除！这就是 --delete-excluded 的效果** |
| 仅目的端存在的 `e.png` | ❌ | ✅ | **DstOnly** | **被删除** |
| 仅目的端存在的 `d.jpg` | ❌ | ✅ | DstOnly | 删除（符合正常 sync 语义） |

### 2.3 DstOnly 回调的删除流程

[DstOnly](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/sync/sync.go#L1042-L1091) 决定删除时机：

```go
func (s *syncCopyMove) DstOnly(dst fs.DirEntry) (recurse bool) {
    switch x := dst.(type) {
    case fs.Object:
        switch s.deleteMode {
        case fs.DeleteModeAfter:
            s.dstFiles[x.Remote()] = x           // 先存到 map，run 末尾调用 deleteFiles(false)
        case fs.DeleteModeDuring, DeleteModeOnly:
            s.deleteFilesCh <- x                 // 立即送入删除通道
        }
    case fs.Directory:
        if s.fdst.Features().CanHaveEmptyDirectories {
            s.dstEmptyDirs[dst.Remote()] = dst   // 登记，末尾再删空目录
        }
        return true  // 目录继续递归！要把里面的 DstOnly 文件也枚举出来
    }
}
```

> 💡 目录返回 `recurse=true` 的意义：即使某个目录整体是 DstOnly（源端根本没有这个目录），也要递归进入，找到其中的所有文件再逐个删除。这和 ErrorSkipDir 的剪枝语义正好相反。

---

## 三、目录过滤对遍历策略选择的影响

### 3.1 ListR 入口的策略判断

[walk.ListR](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go#L149-L164) 是全局递归列出的入口：

```go
func ListR(ctx context.Context, f fs.Fs, path string, includeAll bool, maxLevel int, listType ListType, fn fs.ListRCallback) error {
    fi := filter.GetConfig(ctx)
    doListR := f.Features().ListR

    // 满足任意一条 → 放弃后端 ListR，回退到逐目录 Walk
    if doListR == nil ||                    // 后端不支持
        fi.HaveFilesFrom() ||               // 使用了 --files-from
        maxLevel >= 0 ||                    // 有限深度
        len(fi.Opt.ExcludeFile) > 0 ||      // 使用了 --exclude-if-present
        fi.UsesDirectoryFilters() {         // ⭐ 使用了目录过滤器
        return listRwalk(ctx, f, path, includeAll, maxLevel, listType, fn)
    }
    // 否则直接走后端原生 ListR
    ctx = filter.SetUseFilter(ctx, f.Features().FilterAware && !includeAll)
    return listR(ctx, f, path, includeAll, listType, fn, doListR, ...)
}
```

### 3.2 UsesDirectoryFilters 的精确判断

[UsesDirectoryFilters](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L662-L672)：

```go
func (f *Filter) UsesDirectoryFilters() bool {
    if len(f.dirRules.rules) == 0 {
        return false
    }
    rule := f.dirRules.rules[0]
    re := rule.Regexp.String()
    if rule.Include && re == "^.*$" {   // 只有第一条规则是"包含一切"？
        return false
    }
    return true
}
```

解读：
- 没有 dirRules → 不需要目录级过滤，可用 ListR
- 有 dirRules，但第一条规则就是 `+ ^.*$` → 意味着 dirRules 只是形式上存在（如末尾追加的隐式排除 `/**` 在 dirRules 里的效果），其实没有目录剪枝需求
- 其他情况：有真正的目录剪枝意图 → 必须用 Walk 逐目录

**设计动机：后端 ListR 一次性返回所有条目，无法做到"遍历到某个目录时决定是否进入"的剪枝，只能拿回全部数据再在本地过滤，浪费带宽和 API 调用。**

### 3.3 Walk 逐目录模式下的目录剪枝实现

#### 3.3.1 filterDir 中剔除不合格目录

在 [list.DirSorted → filterAndSortDir → filterDir](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/list/list.go#L99-L156) 中：

```go
for _, entry := range entries {
    switch x := entry.(type) {
    case fs.Directory:
        if !includeAll {
            include, err := IncludeDirectory(x.Remote())
            if !include {
                ok = false           // 从当前目录 entries 中移除
                fs.Debugf(x, "Excluded")
            }
        }
    }
}
```

#### 3.3.2 Walk 调度器只从过滤后的 entries 生成递归 job

在 [walk 函数](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go#L366-L457) 中：

```go
entries, err := listDir(ctx, f, includeAll, job.remote)   // 返回的 entries 已过滤
var jobs []listJob
if err == nil && job.depth != 0 {
    entries.ForDir(func(dir fs.Directory) {
        // ⭐ 只对 entries 中仍然存在的 Directory 生成下一层 job
        jobs = append(jobs, listJob{
            remote: dir.Remote(),
            depth:  job.depth - 1,
        })
    })
}
err = fn(job.remote, entries, err)   // 调用用户回调（如 march 的 DirSortedFn 回调）
```

**剪枝在两个层面起作用：**
1. **生成 jobs 时**：被 IncludeDirectory 判 false 的目录不会出现在 entries.ForDir 中 → 没有下一层 job → 不递归进入子目录
2. **用户回调层（ErrorSkipDir）**：即使目录通过了 filter，回调函数（如 march 的某个业务逻辑）也可以返回 ErrorSkipDir 阻止递归

### 3.4 WalkR（DirTree 一次性构建）模式的剪枝实现

当匹配条件选择了 walk.NewDirTree 时（如 march.makeListDir 中路径 B），实际走 [walkRDirTree](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go#L459-L548)：

```go
func walkRDirTree(ctx context.Context, f fs.Fs, startPath string, includeAll bool, maxLevel int, listR fs.ListRFn) (dirtree.DirTree, error) {
    includeDirectory := fi.IncludeDirectory(ctx, f)
    listR(ctx, startPath, func(entries fs.DirEntries) error {
        for _, entry := range entries {
            switch x := entry.(type) {
            case fs.Directory:
                inc, _ := includeDirectory(x.Remote())
                if includeAll || inc {
                    dirs.AddDir(x)        // 合格目录加入 DirTree
                }
                // 不合格的目录不会加入 DirTree → 后续遍历不到
            case fs.Object:
                if includeAll || fi.IncludeObject(ctx, x) {
                    dirs.Add(x)
                }
            }
        }
        return nil
    })
    // ... 然后遍历 DirTree 时还可以用 ErrorSkipDir 进一步剪枝
}
```

然后在 [walkR](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go#L596-L628) 遍历 DirTree 时，还有一层：

```go
skipping := false
skipPrefix := ""
for _, dirPath := range dirs.Dirs() {
    if skipping {
        if strings.HasPrefix(dirPath, skipPrefix) { continue }  // ⭐ 整棵子树跳过
        skipping = false
    }
    err = fn(dirPath, entries, nil)
    if err == ErrorSkipDir {
        skipping = true
        skipPrefix = dirPath
        if skipPrefix != "" { skipPrefix += "/" }
    }
}
```

**WalkR 模式的两层剪枝：**
1. **构建 DirTree 时**：不合格目录直接不加入 → 节省内存 + 后续不遍历
2. **回调返回 ErrorSkipDir 时**：记录 skipPrefix → 跳过所有以该前缀开头的子目录

### 3.5 两种模式的剪枝能力对比

| 能力 | Walk（逐目录 list.DirSorted） | WalkR（DirTree 一次性构建） |
|------|:--------------------------:|:-------------------------:|
| 避免进入被排除目录的后端 List | ✅ 完全做到（没生成 job） | ❌ 后端 ListR 已返回全部，只在内存中过滤 |
| 节省后端 API 调用/带宽 | ✅ 最佳 | ❌ 取决于后端 |
| 目录剪枝生效时机 | 调度前（listDir 前） | 构建 DirTree 时 + 遍历回调时 |
| ErrorSkipDir 语义 | 不生成下一层 job | 记录 skipPrefix，跳过前缀 |
| 触发条件（策略选择） | UsesDirectoryFilters=true 或 无 ListR 支持 | UsesDirectoryFilters=false 且 后端支持 ListR |

---

## 四、全局信息流：从 flag 到删除决策的完整链条

```
命令行：rclone sync --include "*.jpg" --delete-excluded src: dst:
    │
    ▼
filterflags.AddFlags 注册 → Options 结构体
    ├─ IncludeRule = ["*.jpg"]
    ├─ DeleteExcluded = true
    │
    ▼
filter.NewFilter(opt)
    ├─ parseRules:
    │   ├─ Add(true, "*.jpg")     → fileRules: [+ *.jpg]
    │   │                        → dirRules:  [+ *, + /, + **]  （由 addDirGlobs 推导）
    │   └─ addImplicitExclude=true → Add(false, "/**")
    │   └─ fileRules 最终：[+ *.jpg, - /**]
    │   └─ dirRules 最终： [+ *, + /, + **, - /**]
    │
    ▼
syncCopyMove.run()
    │
    ▼
march.March{
    SrcIncludeAll: false (零值),
    DstIncludeAll: DeleteExcluded = true   ← ⭐
}.Run(ctx)
    │
    ├─ srcListDir: makeListDir(Fsrc, false, ...)   → 应用过滤
    │     └─ list.DirSortedFn 或 walk.NewDirTree(includeAll=false)
    │         └─ filterDir: IncludeObject/IncludeDirectory
    │             结果：只含 *.jpg 文件和必要的父目录
    │
    ├─ dstListDir: makeListDir(Fdst, true, ...)    → 不应用过滤！
    │     └─ list.DirSortedFn 或 walk.NewDirTree(includeAll=true)
    │         └─ filterDir: includeAll=true → 全部保留
    │             结果：目的端完整目录树
    │
    ▼
matchListings(srcChan, dstChan):
    ├─ *.jpg 在两侧都有 → Match → 检查是否更新
    ├─ *.png 只在 dstList 中有 → DstOnly → 送入 deleteFilesCh
    └─ 其他 dst 独有 jpg → DstOnly → 送入 deleteFilesCh
    │
    ▼
deleters goroutine → 真正执行删除
```

---

## 五、澄清后的常见问答

**Q1：不用 `--delete-excluded` 时，目的端被排除的文件会怎样？**
**A：保留不动。** 因为两端都会应用过滤，目的端的排除文件不在 dstList 里，march 根本看不到它们。

**Q2：`--include "*.jpg"` 规则下，为什么会生成目录规则 `+ *, + /, + **`？**
A：这是 `addDirGlobs("*.jpg")` 推导的结果，确保遍历器会进入 `a/`, `a/b/` 等各级目录去寻找 .jpg 文件。如果没有这些隐含的目录规则，Walk 模式下可能因为根目录被末尾的 `- /**` 规则排除而什么都看不到。

**Q3：UsesDirectoryFilters 检查为什么只看第一条规则是否为 `+ ^.*$`？**
A：这是一个启发式判断：如果 dirRules 有内容，但开头就是"包含一切"，说明目录规则只是被隐式逻辑填充的，用户没有真正的目录剪枝意图。这在用户只用 `--exclude` 排除文件而未排除目录时常见，这种情况走 ListR 没问题。

**Q4：ErrorSkipDir 和目录规则剪枝有什么区别？**
A：
- **目录规则剪枝**：在 listDir 返回后，调度下一层 jobs 之前生效。属于"被动式"：过滤掉的目录连回调都不会被调用。
- **ErrorSkipDir**：回调函数主动返回的信号。Walk 模式下是"处理了这个目录但不进入其子目录"；WalkR 模式下是"记录 skipPrefix 用前缀匹配跳过"。bisync、dedupe 等复杂场景用它做业务层决策。

**Q5：filter-aware 后端（如 Drive）的服务端过滤如何结合？**
A：在调用 List/ListR 前设置 ctx 上的 `SetUseFilter(ctx, FilterAware && !includeAll)`，后端在自己的 List 实现中可以通过 `GetUseFilter(ctx)` 读取到这个布尔值，如果为 true 就尝试在服务端 API 层面预先过滤（如 Drive 的 q 参数），避免回传不必要的数据。服务端过滤做不到 100% 精确时，本地 filterDir 还会再做最终把关。
