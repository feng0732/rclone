# Filter 规则 vs Include 参数：隐式排除触发条件与目录规则构成

## 一、核心混乱点：隐式排除的触发范围

`parseRules` 中有两个布尔标志控制隐式排除行为，但它们只对**特定的输入参数**起作用：

```go
func parseRules(opt *RulesOpt, add addFn, clear clearFn) (err error) {
    addImplicitExclude := false
    foundExcludeRule := false

    // 阶段 1：IncludeRule 和 IncludeFrom → 设置 addImplicitExclude = true
    for _, rule := range opt.IncludeRule {
        add(true, rule)
        addImplicitExclude = true          // ⭐ 只要用了 --include*
    }
    for _, rule := range opt.IncludeFrom {
        forEachLine(rule, false, func(line string) error {
            return add(true, line)
        })
        addImplicitExclude = true          // ⭐ 或 --include-from*
    }

    // 阶段 2：ExcludeRule 和 ExcludeFrom → 设置 foundExcludeRule = true
    for _, rule := range opt.ExcludeRule {
        add(false, rule)
        foundExcludeRule = true             // ⭐ --exclude*
    }
    for _, rule := range opt.ExcludeFrom {
        forEachLine(rule, false, func(line string) error {
            return add(false, line)
        })
        foundExcludeRule = true             // ⭐ --exclude-from*
    }

    // 阶段 3：FilterRule 和 FilterFrom → 两个标志都不设置！
    for _, rule := range opt.FilterRule {
        addRule(rule, add, clear)            // ⭐ --filter*
    }
    for _, rule := range opt.FilterFrom {
        forEachLine(rule, false, func(rule string) error {
            return addRule(rule, add, clear)
        })                                    // ⭐ --filter-from*
    }

    // 最后：只有 addImplicitExclude 触发隐式排除
    if addImplicitExclude {
        add(false, "/**")                    // ⭐ 仅 include* 系列触发
    }

    return nil
}
```

### 触发条件总结

| 参数系列 | addImplicitExclude | foundExcludeRule | 末尾补隐式排除 `- /**`？ |
|---------|:------------------:|:----------------:|:---------------------:|
| `--include`, `--include-from` | ✅ 设置为 true | — | **补** |
| `--exclude`, `--exclude-from` | — | ✅ 设置为 true | 不补 |
| `--filter`, `--filter-from` | ❌ 不变 | ❌ 不变 | **不补** |

> ⚠️ **关键不对称性**：
> - `--include "*.jpg"` → 末尾自动补 `- /**`
> - `--filter "+ *.jpg"` → **不补**！用户需要自己写 `- *` 或 `- /**`

`foundExcludeRule` 只用于打印警告（当 `addImplicitExclude && foundExcludeRule` 同时为 true 时）：
> "Using --filter is recommended instead of both --include and --exclude
as the order they are parsed in is indeterminate"

它本身不影响规则构成。

---

## 二、代码铁证：TestNewFilterFullExceptFilesFromOpt

[TestNewFilterFullExceptFilesFromOpt](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter_test.go#L179-L234) 同时使用了所有三种参数系列：

测试配置：
```go
Opt.IncludeRule = []string{"include1"}
Opt.IncludeFrom =  文件内容 ["include2", "include3"]
Opt.ExcludeRule = []string{"exclude1"}
Opt.ExcludeFrom =  文件内容 ["exclude2", "exclude3"]
Opt.FilterRule =  []string{"- filter1", "- filter1b"}
Opt.FilterFrom =  文件内容 ["+ filter2", "- filter3"]
```

测试期望（DumpFilters 输出）：
```
--- File filter rules ---
+ (^|/)include1$       ← IncludeRule
+ (^|/)include2$       ← IncludeFrom
+ (^|/)include3$       ← IncludeFrom
- (^|/)exclude1$       ← ExcludeRule
- (^|/)exclude2$       ← ExcludeFrom
- (^|/)exclude3$       ← ExcludeFrom
- (^|/)filter1$        ← FilterRule
- (^|/)filter1b$       ← FilterRule
+ (^|/)filter2$        ← FilterFrom
- (^|/)filter3$        ← FilterFrom
- ^.*$                 ← ⭐ 隐式排除（因为用了 Include*）
--- Directory filter rules ---
+ ^.*$                 ← Include* 推导的目录规则
- ^.*$                 ← ⭐ 仅一条隐式排除目录规则
```

**关键观察**：
1. 因为使用了 `Include*` 系列 → `addImplicitExclude=true` → 末尾补 `- /**`（fileRules 的最后一条 `- ^.*$`）
2. `Filter*` 中的 `+ filter2` 不触发隐式排除
3. Directory filter rules 只有两条：`+ ^.*$`（来自 include 推导）和 `- ^.*$`（来自隐式排除的路径 B）
4. **没有**其他中间目录规则，文件级 exclude/filter 规则不派生目录规则

---

## 三、四种典型场景的精确推导

### 场景 1：`--include "*.jpg"`（仅 include 系列）

**parseRules 执行流程：**

| 步骤 | 动作 | fileRules 变化 | dirRules 变化 | addImplicitExclude |
|------|------|---------------|--------------|:------------------:|
| 1 | `IncludeRule=["*.jpg"]` → `add(true, "*.jpg")` | `[+ *.jpg]` | `[+ ^.*$]`（路径 C 派生） | ✅ true |
| 2 | 无 Exclude* | — | — | — |
| 3 | 无 Filter* | — | — | — |
| 4 | `addImplicitExclude=true` → `add(false, "/**")` | 追加 `- ^.*$`（路径 B 含 **） | 追加 `- ^.*$`（路径 B） | — |

**最终规则：**
- fileRules: `[+ *.jpg, - ^.*$]`
- dirRules: `[+ ^.*$, - ^.*$]`
- UsesDirectoryFilters: 第一条 `+ ^.*$` → **false**

---

### 场景 2：`--filter "+ *.jpg" --filter "- *"`（仅 filter 系列）

**parseRules 执行流程：**

| 步骤 | 动作 | fileRules 变化 | dirRules 变化 | addImplicitExclude |
|------|------|---------------|--------------|:------------------:|
| 1 | 无 Include* | — | — | ❌ false（保持） |
| 2 | 无 Exclude* | — | — | — |
| 3 | `FilterRule=["+ *.jpg", "- *"]` | | | |
| 3a | `addRule("+ *.jpg")` → `add(true, "*.jpg")` | `[+ *.jpg]` | `[+ ^.*$]`（路径 C 派生，Include=true） | ❌ false（不变） |
| 3b | `addRule("- *")` → `add(false, "*")` | 追加 `- *` | 追加 `- ^.*$`（路径 C 派生，glob=="*"） | ❌ false（不变） |
| 4 | `addImplicitExclude=false` → **不补** | — | — | — |

**最终规则：**
- fileRules: `[+ *.jpg, - *]`
- dirRules: `[+ ^.*$, - ^.*$]` ✅（**只有两条，没有第三条！**）
- UsesDirectoryFilters: 第一条 `+ ^.*$` → **false**

> 📌 **之前文档的错误**：此场景 dirRules 只有两条，不是三条。因为 `--filter` 系列不触发隐式排除。

---

### 场景 3：`--include "*.jpg" --exclude "*"`（include + exclude 系列）

**parseRules 执行流程：**

| 步骤 | 动作 | fileRules 变化 | dirRules 变化 | addImplicitExclude |
|------|------|---------------|--------------|:------------------:|
| 1 | `IncludeRule=["*.jpg"]` → `add(true, "*.jpg")` | `[+ *.jpg]` | `[+ ^.*$]`（路径 C） | ✅ true |
| 2 | `ExcludeRule=["*"]` → `add(false, "*")` | 追加 `- *` | 追加 `- ^.*$`（路径 C，glob=="*"） | ✅ true（不变） |
| 3 | 无 Filter* | — | — | — |
| 4 | `addImplicitExclude=true` → `add(false, "/**")` | 追加 `- ^.*$` | 追加 `- ^.*$`（路径 B，含 **） | — |

**最终规则：**
- fileRules: `[+ *.jpg, - *, - ^.*$]`
- dirRules: `[+ ^.*$, - ^.*$, - ^.*$]` ✅（三条！）
- UsesDirectoryFilters: 第一条 `+ ^.*$` → **false**

> 📌 **与场景 2 的区别**：多了一条隐式排除的 `- ^.*$`（fileRules 和 dirRules 各多一条）。虽然功能上重复，但确实存在。

---

### 场景 4：`--include "*.jpg" --exclude "*.png"`（include + 文件级 exclude）

**parseRules 执行流程：**

| 步骤 | 动作 | fileRules 变化 | dirRules 变化 | addImplicitExclude |
|------|------|---------------|--------------|:------------------:|
| 1 | `IncludeRule=["*.jpg"]` → `add(true, "*.jpg")` | `[+ *.jpg]` | `[+ ^.*$]`（路径 C） | ✅ true |
| 2 | `ExcludeRule=["*.png"]` → `add(false, "*.png")` | 追加 `- *.png` | **不变**（glob!="*"，不含 **，不以 / 结尾） | ✅ true |
| 3 | 无 Filter* | — | — | — |
| 4 | `addImplicitExclude=true` → `add(false, "/**")` | 追加 `- ^.*$` | 追加 `- ^.*$`（路径 B） | — |

**最终规则：**
- fileRules: `[+ *.jpg, - *.png, - ^.*$]`
- dirRules: `[+ ^.*$, - ^.*$]` ✅（**只有两条，没有中间的 `- *.png`**）
- UsesDirectoryFilters: 第一条 `+ ^.*$` → **false**

> 📌 **之前文档的错误**：此场景 dirRules 只有 `[+ ^.*$, - ^.*$]`，`- *.png` 不派生目录规则，也不直接加入 dirRules。

---

## 四、修正后的 dirRules 构成总结表

| 过滤配置 | fileRules | dirRules | UsesDirectoryFilters |
|---------|-----------|----------|:-------------------:|
| 无规则 | `[]` | `[]` | false |
| `--include "*.jpg"` | `[+ *.jpg, - ^.*$]` | `[+ ^.*$, - ^.*$]` | false |
| `--include "*.jpg" --exclude "*.png"` | `[+ *.jpg, - *.png, - ^.*$]` | `[+ ^.*$, - ^.*$]` ✅ | false |
| `--include "*.jpg" --exclude "*"` | `[+ *.jpg, - *, - ^.*$]` | `[+ ^.*$, - ^.*$, - ^.*$]` | false |
| `--filter "+ *.jpg" --filter "- *"` | `[+ *.jpg, - *]` | `[+ ^.*$, - ^.*$]` ✅ | false |
| `--include "dir/*.jpg"` | `[+ dir/*.jpg, - ^.*$]` | `[+ (^|/)dir/$, - ^.*$]` | **true** |
| `--include "/a/**"` | `[+ ^a/.*$, - ^.*$]` | `[+ ^a/.*?/, + ^a/$, - ^.*$]` | **true** |
| `--exclude "*.png"` | `[- *.png]` | `[]` | false |
| `--exclude "*"` | `[- *]` | `[- ^.*$]` | **true** |
| `--exclude "dir/**"` | `[- (^|/)dir/.*$]` | `[- (^|/)dir/.*$]` | **true** |
| `--exclude "/dir/**"` | `[- ^dir/.*$]` | `[- ^dir/.*$]` | **true** |
| `--exclude "dir/"` | `[- (^|/)dir/.*$]` | `[- (^|/)dir/.*$]` | **true** |
| `--filter "- dir/"` | `[- (^|/)dir/.*$]` | `[- (^|/)dir/.*$]` | **true** |

**主要修正点：**
1. `--include "*.jpg" --exclude "*.png"` → dirRules 不含 `*.png`
2. `--filter "+ *.jpg" --filter "- *"` → dirRules 只有两条，无隐式排除追加
3. `--include "*.jpg" --exclude "*"` → dirRules 有三条（一条来自 `- *` 派生，一条来自隐式排除）

---

## 五、隐式排除对目录规则的影响路径

当 `addImplicitExclude=true` 触发 `add(false, "/**")` 时，这条规则如何进入 dirRules：

```
add(false, "/**")
    │
    ├─ isDirRule=false（"/**" 末尾不是 "/"），isFileRule=true
    │
    ├─ 含 "**" → isDirRule=true, isFileRule=true  （路径 B 触发）
    │
    ├─ fileRules.add(false, ^.*$)
    │
    ├─ Include=false, glob="/**"!="*" → ❌ 不触发路径 C（不派生）
    │
    └─ isDirRule=true → dirRules.add(false, ^.*$)
          → dirRules 追加一条 - ^.*$
```

隐式排除通过**路径 B**（含 `**`）直接加入 dirRules，不经过 addDirGlobs 派生。

---

## 六、对递归列表策略选择的影响矩阵

不同配置下 UsesDirectoryFilters 的返回值及两条遍历入口的行为：

| 过滤配置 | UsesDirectoryFilters | walk.ListR 入口 | march.makeListDir<br>`--fast-list` 开 | march.makeListDir<br>`--fast-list` 关 |
|---------|:-------------------:|----------------|:------------------------------------:|:-----------------------------------:|
| 无规则 | false | 原生 ListR | walkRDirTree（原生 ListR） | walkNDirTree（逐目录） |
| `--include "*.jpg"` | false | 原生 ListR | walkRDirTree（原生 ListR） | walkNDirTree（逐目录） |
| `--include "*.jpg" --exclude "*.png"` | false | 原生 ListR | walkRDirTree（原生 ListR） | walkNDirTree（逐目录） |
| `--filter "+ *.jpg" --filter "- *"` | false | 原生 ListR | walkRDirTree（原生 ListR） | walkNDirTree（逐目录） |
| `--include "dir/*.jpg"` | **true** | listRwalk → 逐目录（✓ 剪枝） | walkRDirTree（原生 ListR，✗ 无剪枝） | walkNDirTree（逐目录，✓ 剪枝） |
| `--exclude "*"` | **true** | listRwalk → 逐目录（✓ 剪枝） | walkRDirTree（原生 ListR，✗ 无剪枝） | walkNDirTree（逐目录，✓ 剪枝） |
| `--exclude "dir/**"` | **true** | listRwalk → 逐目录（✓ 剪枝） | walkRDirTree（原生 ListR，✗ 无剪枝） | walkNDirTree（逐目录，✓ 剪枝） |

> ⚠️ 注意：即使 `UsesDirectoryFilters=true`，同步操作的 march.makeListDir 在 `--fast-list` 开启时仍使用原生 ListR（walkRDirTree），不执行真正的目录剪枝。

---

## 七、关键结论

### 1. 隐式排除只对 include 系列参数生效
- `--include` / `--include-from` → `addImplicitExclude=true` → 末尾补 `- /**`
- `--filter` / `--filter-from` → `addImplicitExclude` 不变 → **不补**
- `--exclude` / `--exclude-from` → 只触发警告标志，不补隐式排除

### 2. 文件级 exclude 规则（非 `*`）不影响 dirRules
`--exclude "*.png"` 这类规则：
- 不以 `/` 结尾 → 不触发路径 A
- 不含 `**` → 不触发路径 B
- `glob!="*"` → 不触发路径 C
- 因此 dirRules **完全不变**

### 3. `--filter "+ *.jpg" --filter "- *"` 与 `--include "*.jpg"` 功能等价但规则构成不同
- 功能上：都只包含 `*.jpg`
- 规则构成上：
  - `--include "*.jpg"` → fileRules 多一条 `- ^.*$`（隐式排除），dirRules 末尾多一条 `- ^.*$`
  - `--filter "+ *.jpg" --filter "- *"` → 没有隐式排除，用户手动写的 `- *` 通过路径 C 派生目录规则

### 4. glob=="*" 是唯一的排除规则派生目录匹配特例
只有 `glob` **精确等于** `"*"` 时，排除规则才会触发路径 C（addDirGlobs）。`*.jpg`、`*.png`、`a/*` 都不等于 `"*"`，不会派生。
