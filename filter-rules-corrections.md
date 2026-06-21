# 过滤规则勘误：文件级排除规则是否派生目录匹配

## 一、核心错误纠正

`filter-rules-final.md` 中 dirRules 构成总结表的这一行有误：

| （错误）过滤配置 | dirRules 内容 |
|---------|--------------|
| `--include "*.jpg" --exclude "*.png"` | `[+ ^.*$, - (^|/)[^/]*\.png$, - ^.*$]` ❌ |

**正确的 dirRules 内容应该是：**

| （正确）过滤配置 | dirRules 内容 |
|---------|--------------|
| `--include "*.jpg" --exclude "*.png"` | `[+ ^.*$, - ^.*$]` ✅ |

文件级排除规则 `*.png` **不会派生任何目录匹配，也不会出现在 dirRules 中。**

---

## 二、代码铁证：TestFilterAddDirRuleOrFileRule

[TestFilterAddDirRuleOrFileRule](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter_test.go#L661-L738) 这个测试用例直接验证了 `Add(Include, glob)` 的行为，是判断目录规则派生的铁证。

测试断言表（直接来自代码）：

| Include | glob | File filter rules | Directory filter rules |
|:-------:|------|-------------------|------------------------|
| false | `"potato"` | `- (^|/)potato$` | **空** ✅ |
| true | `"potato"` | `+ (^|/)potato$` | `+ ^.*$` |
| false | `"potato/"` | `- (^|/)potato/.*$` | `- (^|/)potato/.*$` |
| true | `"potato/"` | **空** | `+ (^|/)potato/$` |
| false | `"*"` | `- (^|/)[^/]*$` | `- ^.*$` |
| true | `"*"` | `+ (^|/)[^/]*$` | `+ ^.*$` |
| false | `".*{,/**}"` | `- (^|/)\.[^/]*(|/.*)$` | `- (^|/)\.[^/]*(|/.*)$` |
| true | `"a/b/c/d"` | `+ (^|/)a/b/c/d$` | `+ (^|/)a/b/c/$`<br>`+ (^|/)a/b/$`<br>`+ (^|/)a/$` |

从中可以读出三条派生规则。

---

## 三、Add 函数的三条目录规则派生路径

重新梳理 [Filter.Add(Include, glob)](file:///d:/fz/0601-2/solo-dogfeeding/code/103-rclone/fs/filter/filter.go#L314-L345) 中目录规则的派生逻辑：

```go
func (f *Filter) Add(Include bool, glob string) error {
    isDirRule := strings.HasSuffix(glob, "/")
    isFileRule := !isDirRule

    // 路径 A：排除 "dir/" → glob 变 "dir/**"
    if isDirRule && !Include {
        glob += "**"
    }

    // 路径 B：含 ** → 同时是文件规则和目录规则
    if strings.Contains(glob, "**") {
        isDirRule, isFileRule = true, true
    }

    re, _ := GlobPathToRegexp(glob, f.Opt.IgnoreCase)

    if isFileRule {
        f.fileRules.add(Include, re)
        // 路径 C：Include=true 或 glob=="*" → 派生目录规则
        if Include || glob == "*" {
            f.addDirGlobs(Include, glob)   // ← 派生目录匹配的唯一入口
        }
    }
    if isDirRule {
        f.dirRules.add(Include, re)       // ← 直接加入目录规则
    }
    return nil
}
```

**三条路径分析：**

### 路径 A：`dir/` 形式的排除规则 → 自动扩展为 `dir/**`

```go
if isDirRule && !Include {
    glob += "**"
}
```
- 仅作用于排除规则，且 glob 以 `/` 结尾
- 然后因为 glob 含 `**`，触发路径 B
- 例：`- potato/` → `- potato/**` → isDirRule=true → 直接加入 dirRules

### 路径 B：含 `**` → 直接加入 dirRules

```go
if strings.Contains(glob, "**") {
    isDirRule, isFileRule = true, true
}
// ...
if isDirRule {
    f.dirRules.add(Include, re)
}
```
- 只要 glob 含有 `**`（或被路径 A 扩展后含有），就同时是文件规则和目录规则
- 直接把编译后的正则加入 dirRules
- 例：`- dir/**` → isDirRule=true → dirRules 中出现 `- (^|/)dir/.*$`

### 路径 C：`Include=true` 或 `glob=="*"` → 调用 addDirGlobs 派生

```go
if Include || glob == "*" {
    f.addDirGlobs(Include, glob)
}
```

这是**唯一**的目录规则派生（从文件 glob 推导目录 glob）路径，有两个互不排斥的触发条件：

| 条件 | 说明 | 例子 |
|------|------|------|
| `Include == true` | Include 规则需要确保能扫描到包含目标文件的目录 | `+ *.jpg` → 派生 `/**` → `+ ^.*$` |
| `glob == "*"` | 精确等于单个星号（即使是 exclude） | `- *` → 派生 `/**` → `- ^.*$` |

> ⚠️ **注意 `glob == "*"` 是精确比较**：`"*"` 匹配，`"*.jpg"`、`"*.png"`、`"**"` 都不匹配。

---

## 四、文件级排除规则的完整分类

基于三条路径，文件级排除规则（isFileRule=true 的 exclude）是否影响 dirRules：

| 排除 glob | 路径 A? | 路径 B? | 路径 C? | dirRules 是否变化 |
|-----------|:-------:|:-------:|:-------:|:----------------:|
| `*.png` | ❌（不以 `/` 结尾） | ❌（不含 `**`） | ❌（Include=false，且 `glob!="*"`） | **不变** ✅ |
| `*` | ❌ | ❌ | ✅（glob=="*"） | 加入 `- ^.*$` |
| `dir/` | ✅（以 `/` 结尾，exclude）→ 变 `dir/**` | ✅（含 `**`） | ❌ | 加入 `- (^|/)dir/.*$` |
| `dir/**` | ❌ | ✅（含 `**`） | ❌ | 加入 `- (^|/)dir/.*$` |
| `**/secret/**` | ❌ | ✅（含 `**`） | ❌ | 加入 `- .*secret/.*$` |

---

## 五、混合场景 step-by-step 验证

### 场景：`--include "*.jpg" --exclude "*.png"`

完整执行流程：

**步骤 1：parseRules 先处理 include**
```
Add(true, "*.jpg")
  │
  ├─ isDirRule=false（不以 / 结尾），isFileRule=true
  ├─ 不含 ** → 不变
  ├─ fileRules.add(true, (^|/)[^/]*\.jpg$)
  │     → fileRules: [+ *.jpg]
  │
  ├─ Include=true → 触发路径 C，调用 addDirGlobs(true, "*.jpg")
  │     │
  │     └─ globToDirGlobs("*.jpg")
  │           ├─ 无 / 无 **, len(out)==0
  │           └─ 返回 ["/**"]
  │     └─ addDirGlobs 跳过 "/"（"/**" 不是 "/"）
  │     └─ dirRules.add(true, ^.*$)
  │           → dirRules: [+ ^.*$]
  │
  └─ isDirRule=false → 不直接加入 dirRules
```

**步骤 2：parseRules 再处理 exclude**
```
Add(false, "*.png")
  │
  ├─ isDirRule=false（不以 / 结尾），isFileRule=true
  ├─ 不含 ** → 不变
  ├─ fileRules.add(false, (^|/)[^/]*\.png$)
  │     → fileRules: [+ *.jpg, - *.png]
  │
  ├─ Include=false 且 glob="*.png"!="*" → ❌ 不触发路径 C
  │     → addDirGlobs 不被调用
  │
  └─ isDirRule=false → ❌ 不直接加入 dirRules
        → dirRules 仍然是: [+ ^.*$]  （不变！）
```

**步骤 3：parseRules 末尾补隐式排除（因为用了 include）**
```
Add(false, "/**")
  │
  ├─ isDirRule=false（不以 / 结尾），isFileRule=true
  │   （注意："/**" 末尾不是 "/"，所以 isDirRule 初始为 false）
  │
  ├─ 含 ** → isDirRule=true, isFileRule=true  （路径 B 触发）
  │
  ├─ fileRules.add(false, ^.*$)
  │     → fileRules: [+ *.jpg, - *.png, - ^.*$]
  │
  ├─ Include=false 且 glob="/**"!="*" → ❌ 不触发路径 C
  │
  └─ isDirRule=true → 直接加入 dirRules
        → dirRules: [+ ^.*$, - ^.*$]
```

**最终结果：**

| 规则集合 | 内容 |
|---------|------|
| fileRules | `[+ *.jpg, - *.png, - ^.*$]` |
| dirRules | `[+ ^.*$, - ^.*$]` ✅（**不含 `- *.png`**） |
| UsesDirectoryFilters() | 第一条规则是 `+ ^.*$`（include + 正则正好是 `^.*$`）→ **false** |

---

## 六、修正后的 dirRules 构成总结表

| 过滤配置 | dirRules 内容 | 第一条规则 | UsesDirectoryFilters |
|---------|--------------|-----------|:-------------------:|
| 无规则 | 空 | — | false |
| `--include "*.jpg"` | `[+ ^.*$, - ^.*$]` | `+ ^.*$` | false |
| `--include "*.jpg" --exclude "*.png"` | `[+ ^.*$, - ^.*$]` ✅ | `+ ^.*$` | false |
| `--include "*.jpg" --exclude "*"` | `[+ ^.*$, - ^.*$, - ^.*$]` | `+ ^.*$` | false |
| `--include "dir/*.jpg"` | `[+ (^|/)dir/$, - ^.*$]` | `+ (^|/)dir/$` | **true** |
| `--include "/a/**"` | `[+ ^a/.*?/, + ^a/$, - ^.*$]` | `+ ^a/.*?/` | **true** |
| `--exclude "*.png"` | 空 | — | false |
| `--exclude "*"` | `[- ^.*$]` | `- ^.*$` | **true** |
| `--exclude "dir/**"` | `[- (^|/)dir/.*$]` | `- (^|/)dir/.*$` | **true** |
| `--exclude "/dir/**"` | `[- ^dir/.*$]` | `- ^dir/.*$` | **true** |
| `--exclude "dir/"` | 先变 `dir/**` → 含 ** | `- (^|/)dir/.*$` | **true** |
| `--filter "+ *.jpg" --filter "- *"` | `[+ ^.*$, - ^.*$, - ^.*$]` | `+ ^.*$` | false |
| `--filter "- dir/"` | `[- (^|/)dir/.*$]` | `- (^|/)dir/.*$` | **true** |

**注意两行变化：**
- `--include "*.jpg" --exclude "*.png"`：dirRules 只有 `[+ ^.*$, - ^.*$]`，排除的 `*.png` 不影响 dirRules
- `--exclude "*"`：因为 `glob=="*"` 触发路径 C，dirRules 有 `[- ^.*$]` → UsesDirectoryFilters=true

---

## 七、对递归列表策略选择的影响

由于 `--include "*.jpg" --exclude "*.png"` 场景下 UsesDirectoryFilters=false，两条遍历入口的行为：

| 入口 | 行为 |
|-----|------|
| walk.ListR | 走原生 ListR（因为 UsesDirectoryFilters=false） |
| march.makeListDir --fast-list 开 | walkRDirTree 用原生 ListR |
| march.makeListDir --fast-list 关 | walkNDirTree 逐目录 |

如果换成 `--include "*.jpg" --exclude "*"`（glob=="*" 触发路径 C）：
- dirRules = `[+ ^.*$, - ^.*$, - ^.*$]`
- 第一条仍然是 `+ ^.*$` → UsesDirectoryFilters=false
- 行为不变

但如果是纯排除规则 `--exclude "*"`：
- dirRules = `[- ^.*$]`
- 第一条是 `- ^.*$`（不是 include）→ UsesDirectoryFilters=**true**
- walk.ListR 入口会回退逐目录，确保真正剪枝

---

## 八、关键结论

1. **文件级排除规则不派生目录匹配，也不直接加入 dirRules**，除非：
   - glob 以 `/` 结尾（路径 A 自动扩展为 `**` 后触发路径 B）
   - glob 含 `**`（路径 B 直接加入）
   - glob 精确等于 `"*"`（路径 C 派生）

2. `glob == "*"` 是唯一一种"文件级排除规则也派生目录匹配"的特例。`*.jpg`、`*.png`、`a/*` 都不等于 `"*"`，所以不会派生。

3. 混合使用 include 和文件级 exclude（非 `*`）时，exclude 规则只影响 fileRules，不影响 dirRules 的构成，因此也不影响 UsesDirectoryFilters() 的返回值。
