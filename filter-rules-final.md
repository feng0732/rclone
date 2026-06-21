# Rclone 过滤规则系统最终版：目录规则推导与递归列表策略选择

本文档是对 `filter-rules.md` 和 `filter-rules-followup.md` 的最终修正，聚焦两个关键问题的精确代码追踪：

1. **include 规则怎样派生目录匹配** —— 从 glob 到 dirRules 的完整推导过程
2. **哪些目录过滤会影响递归列表选择** —— 从 ListR 到 Walk 的策略选择完整条件链

---

## 一、目录规则推导：从文件 glob 到 dirRules 的精确过程

### 1.1 Add 函数：分发器

[Filter.Add(Include, glob)](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L314-L345) 是单条规则的入口，决定这条 glob 如何分流到 `fileRules` 和 `dirRules`：

```go
func (f *Filter) Add(Include bool, glob string) error {
    isDirRule := strings.HasSuffix(glob, "/")
    isFileRule := !isDirRule

    // 排除 "dir/" → 自动变成排除 "dir/**"
    if isDirRule && !Include {
        glob += "**"
    }

    // glob 含 ** → 同时匹配文件和目录
    if strings.Contains(glob, "**") {
        isDirRule, isFileRule = true, true
    }

    re, _ := GlobPathToRegexp(glob, f.Opt.IgnoreCase)

    if isFileRule {
        f.fileRules.add(Include, re)
        // ⭐ 关键：仅 Include 规则或 glob=="*" 才推导目录规则
        // 排除规则不推导目录剪枝
        if Include || glob == "*" {
            f.addDirGlobs(Include, glob)   // ← 目录规则推导入口
        }
    }
    if isDirRule {
        f.dirRules.add(Include, re)
    }
    return nil
}
```

> ⚠️ **核心不对称性**：
> - **include 规则** → 调用 `addDirGlobs` 推导所需目录，确保能找到这些文件
> - **exclude 规则** → 不推导目录（除非 `glob=="*"`），只能在文件级过滤
> - **含 `**` 的规则** → 同时加入 fileRules 和 dirRules

### 1.2 addDirGlobs：推导并过滤掉根目录

[addDirGlobs](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L298-L311) 很简单，只是遍历 globToDirGlobs 的结果，并跳过 `"/"`（根目录总是包含）：

```go
func (f *Filter) addDirGlobs(Include bool, glob string) error {
    for _, dirGlob := range globToDirGlobs(glob) {
        if dirGlob == "/" {
            continue
        }
        dirRe, _ := GlobPathToRegexp(dirGlob, f.Opt.IgnoreCase)
        f.dirRules.add(Include, dirRe)
    }
    return nil
}
```

### 1.3 globToDirGlobs：算法核心

[globToDirGlobs](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/glob.go#L218-L257) 把文件 glob 逆推为"可能包含该文件的所有目录"：

```go
func globToDirGlobs(glob string) (out []string) {
    // 含 {**/} 或 {{regexp}} → 无法推导，退化为"所有目录"
    if tooHardRe.MatchString(glob) {
        return []string{"/**"}
    }

    // 从右向左扫描，每次切掉最后一个路径段
    for {
        i := strings.LastIndex(glob, "/")     // 最后一个 / 的位置
        j := strings.LastIndex(glob, "**")   // 最后一个 ** 的位置
        what := ""
        if j > i {    // ** 在 / 右边 → 以 ** 为切点
            i = j
            what = "**"
        }
        if i < 0 {    // 没有 / 也没有 **
            if len(out) == 0 {
                out = append(out, "/**")   // 形如 "*.jpg" 的全局通配 → 需要扫描所有目录
            }
            break
        }
        glob = glob[:i]                       // 切掉切点右边的部分
        newGlob := glob + what + "/"          // 组成目录 glob
        if out 为空 或 最后一个 != newGlob {
            out = append(out, newGlob)
        }
    }
    return out
}
```

### 1.4 实例推导演示

下面通过 4 个典型场景，精确展示推导过程和最终的 dirRules 构成。

---

#### 场景 1：`--include "*.jpg"`

| 步骤 | 过程 | 结果 |
|------|------|------|
| 1. Add(true, "*.jpg") | isFileRule=true, 不含 ** | |
| 2. fileRules.add | + `(^|/)[^/]*\.jpg$` | fileRules: [+ *.jpg] |
| 3. Include=true → addDirGlobs(true, "*.jpg") | 调用 globToDirGlobs("*.jpg") | |
| 4. globToDirGlobs: 无 /, 无 **, len(out)==0 | 返回 `["/**"]` | |
| 5. addDirGlobs 遍历: "/**" 不是 "/" | dirRules.add(true, `^.*$`) | dirRules: [+ ^.*$] |
| 6. parseRules 末尾补隐式排除 | Add(false, "/**") | |
| 7. Add(false, "/**"): 含 ** → isDirRule=true | fileRules.add(false, `^.*$`) | fileRules: [+ *.jpg, - ^.*$] |
| 8. | dirRules.add(false, `^.*$`) | dirRules: [+ ^.*$**,\n- **^.*$] |

**关键观察**：
- `*.jpg` 这种不带路径前缀的 include → 推导 `/**` → 目录规则第一条是 `+ ^.*$`（包含一切）
- 后面追加的隐式排除 `- ^.*$` 在目录规则中排在 **第二位**

---

#### 场景 2：`--include "dir/*.jpg"`

| 步骤 | 过程 | 结果 |
|------|------|------|
| 1. Add(true, "dir/*.jpg") | 调用 globToDirGlobs("dir/*.jpg") | |
| 2. globToDirGlobs 第一次: 最后 / 在 index 3 | glob→"dir", newGlob→"dir/" | out=["dir/"] |
| 3. 第二次: glob="dir", 无 / 无 **, len(out)≠0 | break | 返回 ["dir/"] |
| 4. addDirGlobs: "dir/" 不是 "/" | dirRules.add(true, `(^|/)dir/$`) | dirRules: [+ (^|/)dir/$] |
| 5. 末尾补 Add(false, "/**") | 含 ** → isDirRule=true | |
| 6. | dirRules.add(false, `^.*$`) | dirRules: [+ (^|/)dir/$**, **- ^.*$] |

**关键观察**：
- `dir/*.jpg` 带路径前缀 → 推导 `["dir/"]` → 目录规则第一条是 `+ (^|/)dir/$`
- 这不是 `^.*$` → UsesDirectoryFilters() 返回 **true**

---

#### 场景 3：`--include "a/b/**/c/*.jpg"`

| 步骤 | 过程 | 结果 |
|------|------|------|
| 1. Add(true, "a/b/**/c/*.jpg") | 调用 globToDirGlobs | |
| 2. 第一次: 最后 / 在 index 8（"/c/"） | glob→"a/b/**/c", newGlob→"a/b/**/c/" | out=["a/b/**/c/"] |
| 3. 第二次: 最后 ** 在 index 4（> 最后 / 在 3） | i=4, what="**", glob→"a/b/", newGlob→"a/b/**/" | out=["a/b/**/c/", "a/b/**/"] |
| 4. 第三次: 最后 / 在 index 3 | glob→"a/b", newGlob→"a/b/" | out=[..., "a/b/"] |
| 5. 第四次: 最后 / 在 index 1 | glob→"a", newGlob→"a/" | out=[..., "a/"] |
| 6. 第五次: 无 / 无 **, len(out)≠0 | break | |
| 7. addDirGlobs 跳过 "/" | dirRules 添加 4 条 include | |
| 8. 末尾补 Add(false, "/**") | dirRules 加 - ^.*$ | dirRules: [+ a/b/**/c/, + a/b/**/, + a/b/, + a/, - ^.*$] |

---

#### 场景 4：`--exclude "*.png"`（不带 include）

| 步骤 | 过程 | 结果 |
|------|------|------|
| 1. Add(false, "*.png") | isFileRule=true, Include=false, glob!="*" | |
| 2. fileRules.add(false, ...) | | fileRules: [- *.png] |
| 3. Include=false && glob!="*" | **不调用 addDirGlobs** | |
| 4. 无 include → 末尾不补隐式排除 | | dirRules: **空** |

**关键观察**：纯 exclude 场景下 dirRules 为空 → UsesDirectoryFilters() 返回 **false**

---

### 1.5 dirRules 构成总结表

| 过滤配置 | dirRules 内容 | 第一条规则 | UsesDirectoryFilters |
|---------|--------------|-----------|:-------------------:|
| 无规则 | 空 | — | false |
| `--include "*.jpg"` | `[+ ^.*$, - ^.*$]` | `+ ^.*$` | false |
| `--include "*.jpg" --exclude "*.png"` | `[+ ^.*$, - (^|/)[^/]*\.png$, - ^.*$]` | `+ ^.*$` | false |
| `--include "dir/*.jpg"` | `[+ (^|/)dir/$, - ^.*$]` | `+ (^|/)dir/$` | **true** |
| `--include "/a/**"` | `[+ ^a/.*?/, + ^a/$, - ^.*$]` | `+ ^a/.*?/` | **true** |
| `--exclude "*.png"` | 空 | — | false |
| `--exclude "dir/**"` | `[- (^|/)dir/.*$]` | `- (^|/)dir/.*$` | **true** |
| `--exclude "/dir/**"` | `[- ^dir/.*$]` | `- ^dir/.*$` | **true** |
| `--filter "+ *.jpg" --filter "- *"` | `[+ ^.*$, - ^.*$]` | `+ ^.*$` | false |
| `--filter "- dir/"` | 先 Add(false, "dir/")→glob 变 "dir/**"→含 **→dirRules.add(false, re) | `- (^|/)dir/.*$` | **true** |

---

## 二、UsesDirectoryFilters：精确判定

[UsesDirectoryFilters](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L662-L672) 的实现极其简洁，但判断逻辑极其关键：

```go
func (f *Filter) UsesDirectoryFilters() bool {
    if len(f.dirRules.rules) == 0 {
        return false
    }
    rule := f.dirRules.rules[0]      // ⭐ 只看第一条规则！
    re := rule.Regexp.String()
    if rule.Include && re == "^.*$" {   // 第一条是"包含一切"？
        return false
    }
    return true
}
```

**判定逻辑拆解**：

1. `dirRules` 为空 → false（无目录规则，不需要目录剪枝）
2. 否则，取 **第一条** 规则：
   - 如果第一条是 include 且其正则正好是 `^.*$`（即 `/**` 的编译结果）→ false（没有真正的目录剪枝意图）
   - 其他所有情况 → true

**为什么只看第一条？**
因为 parseRules 的处理顺序是 include → exclude → filter → 隐式排除。只要有 include 且路径前缀非全局通配，它推导出的目录规则就会排在最前面，且不是 `^.*$`。如果第一条已经是 `^.*$`，说明用户的 include 是全局的（如 `*.jpg`），目录过滤不会带来任何剪枝收益——反正所有目录都要进。

**测试用例验证**（来自 [filter_test.go L854-L925](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter_test.go#L854-L925)）：

| 规则 | 期望 | 原因 |
|------|------|------|
| `[]` | false | 空 |
| `["+ *"]` | false | 第一条 + ^.*$ |
| `["+ *.jpg", "- *"]` | false | 第一条 + ^.*$ |
| `["- *.jpg"]` | false | dirRules 空 |
| `["- *.jpg", "+ *"]` | false | 第一条 + ^.*$ |
| `["+ dir/*.jpg", "- *"]` | **true** | 第一条 + (^|/)dir/$ |
| `["+ dir/**"]` | **true** | 第一条不是 ^.*$ |
| `["- dir/**"]` | **true** | 第一条是 exclude |
| `["- /dir/**"]` | **true** | 第一条是 exclude |

---

## 三、递归列表策略选择：两条平行的决策链

rclone 中有 **两条平行的遍历入口**，各自有独立的策略选择逻辑。之前的文档混淆了两者。

### 3.1 决策链 A：walk.ListR（API 层入口）

[walk.ListR](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go#L149-L164) 是供 vfs、size、check 等功能调用的高层 API：

```go
func ListR(ctx context.Context, f fs.Fs, path string, includeAll bool, maxLevel int, listType ListType, fn fs.ListRCallback) error {
    fi := filter.GetConfig(ctx)
    doListR := f.Features().ListR

    // ⭐ 5 个 OR 条件，任一满足 → 回退到逐目录 Walk
    if doListR == nil ||                  // 后端不支持 ListR
        fi.HaveFilesFrom() ||             // 使用了 --files-from
        maxLevel >= 0 ||                  // 有限深度
        len(fi.Opt.ExcludeFile) > 0 ||    // 使用了 --exclude-if-present
        fi.UsesDirectoryFilters() {       // ⭐ 使用了目录过滤器
        return listRwalk(ctx, f, path, includeAll, maxLevel, listType, fn)
    }

    // 否则直接使用后端原生 ListR
    ctx = filter.SetUseFilter(ctx, f.Features().FilterAware && !includeAll)
    return listR(ctx, f, path, includeAll, listType, fn, doListR, ...)
}
```

**UsesDirectoryFilters 在这里是"一票否决"**。只要它返回 true，即使后端支持 ListR 且开了 --fast-list，也强制回退逐目录 Walk，确保目录剪枝生效。

调用者包括：[vfs.go](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/vfs/vfs.go#L676)（计算使用量）、[operations.go](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/operations/operations.go#L764)（ListFn）、各种测试清理函数等。

### 3.2 决策链 B：march.makeListDir（同步专用入口）

[march.makeListDir](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/march/march.go#L123-L171) 是 sync/copy/check/move/bisync 等同步操作的专用入口，**不检查 UsesDirectoryFilters**：

```go
func (m *March) makeListDir(ctx context.Context, f fs.Fs, includeAll bool, keyFn list.KeyFn) listDirFn {
    ci := fs.GetConfig(ctx)
    fi := filter.GetConfig(ctx)

    // ⭐ 仅两个 OR 条件，都不涉及 UsesDirectoryFilters
    if !(ci.UseListR && f.Features().ListR != nil) &&   // !(--fast-list + 支持 ListR)
        !(ci.NoTraverse && fi.HaveFilesFrom()) {         // !(--files-from + --no-traverse)
        // 路径 1：逐目录模式
        return func(dir string, callback fs.ListRCallback) error {
            dirCtx := filter.SetUseFilter(m.Ctx, f.Features().FilterAware && !includeAll)
            return list.DirSortedFn(dirCtx, f, includeAll, dir, callback, keyFn)
        }
    }

    // 路径 2：--fast-list 或 --files-from + --no-traverse → 一次性构建 DirTree
    var dirs dirtree.DirTree
    var started bool
    return func(dir string, callback fs.ListRCallback) error {
        if !started {
            dirCtx := filter.SetUseFilter(m.Ctx, f.Features().FilterAware && !includeAll)
            dirs, dirsErr = walk.NewDirTree(dirCtx, f, m.Dir, includeAll, ci.MaxDepth)
            started = true
        }
        entries, _ := dirs[dir]
        delete(dirs, dir)
        sort.SortStable(entries, keyFn)
        return callback(entries)
    }
}
```

**关键差异**：march.makeListDir 的条件里 **没有** `fi.UsesDirectoryFilters()` 检查。这意味着：即使有真正的目录剪枝需求，只要用户开了 --fast-list，march 仍然会走路径 2（一次性 DirTree）。

### 3.3 路径 2 内部的决策：walk.NewDirTree

当 march.makeListDir 选择路径 2 时，[walk.NewDirTree](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go#L581-L594) 内部还要再做一次决策：

```go
func NewDirTree(ctx context.Context, f fs.Fs, path string, includeAll bool, maxLevel int) (dirtree.DirTree, error) {
    ci := fs.GetConfig(ctx)
    fi := filter.GetConfig(ctx)

    // 条件 1：--no-traverse + --files-from
    if ci.NoTraverse && fi.HaveFilesFrom() {
        return walkRDirTree(ctx, f, path, includeAll, maxLevel, fi.MakeListR(ctx, f.NewObject))
    }
    // 条件 2：有 ListR + 递归深度 >1 + 无 files-from
    // ⭐ 同样不检查 UsesDirectoryFilters！
    if ListR := f.Features().ListR; (maxLevel < 0 || maxLevel > 1) && ListR != nil && !fi.HaveFilesFrom() {
        return walkRDirTree(ctx, f, path, includeAll, maxLevel, ListR)   // ← 原生 ListR
    }
    // 条件 3：其他 → 逐目录
    return walkNDirTree(ctx, f, path, includeAll, maxLevel, list.DirSorted)
}
```

**注意**：NewDirTree 也不检查 UsesDirectoryFilters！如果 --fast-list + 支持 ListR + UsesDirectoryFilters=true → 仍然会走 `walkRDirTree` 用原生 ListR。

### 3.4 两种剪枝的本质区别

| 剪枝方式 | 实现位置 | 生效时机 | 实际效果 |
|---------|---------|---------|---------|
| **真正目录剪枝**（Walk 逐目录模式） | [walk 调度器](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go#L403-L410) | listDir 返回后，生成子目录 job 前 | 被排除的目录不生成 job → **根本不进入** → 节省 API 调用和带宽 |
| **内存剔除**（walkRDirTree 模式） | [walkRDirTree](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/walk/walk.go#L459-L548) 回调中 | 后端 ListR 返回所有条目后 | 本地内存中过滤 → 条目已从后端返回 → **不节省 API/带宽** |

walkRDirTree 的过滤逻辑（L514-L528）：

```go
case fs.Directory:
    inc, _ := includeDirectory(x.Remote())
    if includeAll || inc {
        dirs.AddDir(x)   // 合格目录加入 DirTree
    }
    // 不合格的不加入 → 后续遍历不到
```

虽然最终效果都是"被排除的目录不出现在结果中"，但前者是"调度前就不进入"，后者是"后端返回后才剔除"。

### 3.5 完整决策树

```
用户发起操作（如 rclone sync src: dst:）
    │
    ▼
syncCopyMove.run() → march.March{
    SrcIncludeAll: false,          // 源端永远应用过滤
    DstIncludeAll: DeleteExcluded  // 目的端：--delete-excluded 时不过滤
}.Run(ctx)
    │
    ▼
march.makeListDir() 选择路径：
    │
    ├─ ✗ --fast-list 或 ✗ 支持 ListR？ → 路径 1：list.DirSortedFn（逐目录，真正剪枝）
    │      │
    │      └─ list.DirSorted → filterDir → walk 调度器
    │             └─ 被 IncludeDirectory 判 false 的目录不生成子目录 job
    │
    └─ ✓ --fast-list 且 ✓ 支持 ListR？ → 路径 2：walk.NewDirTree
           │
           ├─ ✗ --no-traverse + --files-from？ → walkRDirTree with MakeListR
           │
           ├─ ✓ 递归深度>1 且 ✓ 无 files-from？ → walkRDirTree with 原生 ListR
           │      │
           │      └─ 后端一次性返回所有条目
           │      └─ 回调中 includeDirectory 内存剔除
           │      └─ 无真正目录剪枝
           │
           └─ 其他 → walkNDirTree with list.DirSorted（逐目录，真正剪枝）


walk.ListR 入口（vfs/size/operations 等）的选择：
    │
    ├─ 5 个 OR 条件（含 UsesDirectoryFilters）任一满足？
    │   └─ ✓ → listRwalk → Walk → 逐目录（真正剪枝）
    │
    └─ 全部不满足？ → listR with 原生 ListR（无剪枝）
```

---

## 四、目录过滤对递归列表选择的影响矩阵

综合以上分析，不同过滤配置在不同入口下的实际行为：

| 过滤配置 | UsesDirectoryFilters | walk.ListR 入口行为 | march.makeListDir 行为（--fast-list 开） | march.makeListDir 行为（--fast-list 关） |
|---------|:-------------------:|-------------------|--------------------------------------|---------------------------------------|
| 无规则 | false | 原生 ListR | walkRDirTree（原生 ListR） | walkNDirTree（逐目录） |
| `--include "*.jpg"` | false | 原生 ListR | walkRDirTree（原生 ListR） | walkNDirTree（逐目录） |
| `--include "dir/*.jpg"` | **true** | listRwalk → 逐目录（✓ 剪枝） | walkRDirTree（原生 ListR，✗ 无剪枝） | walkNDirTree（逐目录，✓ 剪枝） |
| `--exclude "*.png"` | false | 原生 ListR | walkRDirTree（原生 ListR） | walkNDirTree（逐目录） |
| `--exclude "dir/**"` | **true** | listRwalk → 逐目录（✓ 剪枝） | walkRDirTree（原生 ListR，✗ 无剪枝） | walkNDirTree（逐目录，✓ 剪枝） |
| `--files-from list.txt` | false（但 HaveFilesFrom=true） | listRwalk → 逐目录 | walkRDirTree（MakeListR） | walkNDirTree（逐目录） |
| `--exclude-if-present .nobackup` | false（但 ExcludeFile 非空） | listRwalk → 逐目录 | walkNDirTree（逐目录，✓ 剪枝） | walkNDirTree（逐目录，✓ 剪枝） |

> ⚠️ **红框警示**：当 `UsesDirectoryFilters=true` 且使用 `--fast-list` 进行同步操作时，**目录剪枝不会真正生效**！后端 ListR 会返回所有目录，然后在本地内存中过滤。这可能带来不必要的 API 调用和带宽消耗。

---

## 五、修正后的同步决策矩阵

| 场景 | SrcIncludeAll | DstIncludeAll | 源端列表 | 目的端列表 | match 结果 | 同步动作 |
|------|:------------:|:------------:|---------|-----------|-----------|---------|
| **默认**（无 --delete-excluded） | false | false | 应用过滤 | **应用过滤** | | |
| 例：`--include "*.jpg"` | | | 只含 .jpg | 只含 .jpg | `photo.jpg` 在两侧 | Match → 按需更新 |
| （规则 `+ *.jpg`, `- **`） | | | | | `doc.png` 在两侧 | 都消失 → **保留，不处理** |
| | | | | | 仅目的有 `old.png` | 目的端也看不到 → **保留** |
| | | | | | 仅源有 `new.jpg` | SrcOnly → **复制** |
| | | | | | 仅目的有 `old.jpg` | DstOnly → **删除（如果开了 delete）** |
| **--delete-excluded** | false | **true** | 应用过滤 | **不应用过滤** | | |
| 例：`--include "*.jpg" --delete-excluded` | | | 只含 .jpg | 全部列出 | `photo.jpg` 在两侧 | Match → 按需更新 |
| | | | | | `doc.png` 仅目的有 | DstOnly → **被删除**（这就是 --delete-excluded 的效果） |
| | | | | | 仅目的有 `old.jpg` | DstOnly → **被删除**（正常 sync 语义） |

---

## 六、关键澄清总结

### 1. include 规则如何派生目录匹配？
- 仅 `Include=true` 或 `glob=="*"` 时调用 `addDirGlobs`
- `globToDirGlobs` 从右向左扫描，每次切掉最后一个 `/` 或 `**`，构成父目录的 glob
- 不带路径前缀的 include（如 `*.jpg`）→ 推导 `/**` → 目录规则第一条是 `+ ^.*$`
- 带路径前缀的 include（如 `dir/*.jpg`）→ 推导 `"dir/"` → 目录规则第一条不是 `^.*$` → UsesDirectoryFilters=true

### 2. 哪些目录过滤会影响递归列表选择？
- **walk.ListR 入口**：5 个条件任一满足就回退逐目录，包括 `UsesDirectoryFilters()`
- **march.makeListDir（同步操作）**：**不检查** UsesDirectoryFilters。仅看 --fast-list 和 --files-from+--no-traverse
- **walk.NewDirTree 内部**：也不检查 UsesDirectoryFilters
- **最终结果**：同步操作下，即使 UsesDirectoryFilters=true，只要开了 --fast-list，就没有真正的目录剪枝

### 3. 目录剪枝的两层含义
- **真正剪枝**（Walk 模式）：在生成下一层 job 前就排除 → 不进入目录 → 节省 API/带宽
- **内存剔除**（ListR 模式）：后端返回所有数据后在本地过滤 → 不节省 API/带宽
