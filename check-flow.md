# rclone check 校验链路分析

## 1. 整体架构概览

rclone 的 check 校验链路由三层组成：

| 层级 | 入口 | 核心逻辑 | 作用 |
|------|------|----------|------|
| 命令层 | cmd/check/check.go | 解析参数，选择校验模式 | CLI 入口 |
| 调度层 | fs/march/march.go | 并行遍历源与目标目录树 | 对齐文件名 |
| 校验层 | fs/operations/check.go | 大小比对 → 哈希/下载比对 → 差异上报 | 判定一致性 |

辅助命令：

- cmd/cryptcheck/cryptcheck.go：加密远端的专用校验
- cmd/checksum/checksum.go：基于 SUM 文件的校验

---

## 2. 源目标比对流程

### 2.1 命令入口与模式选择

在 cmd/check/check.go 第 162-203 行中，`rclone check` 根据参数选择三种校验模式：

```
rclone check source:path dest:path
        ├── --checkfile HASH  →  CheckSum()    (SUM 文件比对)
        ├── --download        →  CheckDownload() (逐字节下载比对)
        └── (默认)            →  Check()        (哈希比对)
```

**关键分支逻辑：**

1. **`--checkfile HASH`**（第 170-188 行）：`source:path` 被解释为 SUM 文本文件路径，调用 `operations.CheckSum()`
2. **`--download`**（第 191-193 行）：调用 `operations.CheckDownload()`，下载双方内容逐字节比对
3. **默认**（第 194-201 行）：先求源与目标支持的哈希交集，调用 `operations.Check()`

### 2.2 March 调度：目录树对齐

fs/march/march.go `March.Run()` 启动并行遍历，核心在 `matchListings()`：

```
srcChan ──┐
          ├── matchListings() ──┬── srcOnly()   → Callback.SrcOnly()
dstChan ──┘                    ├── dstOnly()   → Callback.DstOnly()
                               └── match()     → Callback.Match()
```

#### 2.2.1 排序键构造：类型后缀拆分

`srcOrDstKey()`（第 87-108 行）为每个条目生成排序键，关键机制是**类型后缀**：

```
排序键 = transform(name) + 类型后缀
  Directory → 后缀 "D"   (ASCII 0x44)
  Object    → 后缀 "F"   (ASCII 0x46)
```

由于 'D' < 'F'，同名目录的排序键始终小于同名文件：
- 目录 "foo" → 键 `"fooD"`
- 文件 "foo" → 键 `"fooF"`
- `"fooD" < "fooF"` → 目录排在文件前面

**这意味着文件与目录同名时，两者的排序键不同，永远不会被 `matchListings` 配对为 match。**

名称变换链（第 70-82 行），在追加类型后缀之前施加：

1. 若 `!NoUnicodeNormalization`，施加 NFC 规范化
2. 若目标 Fs 大小写不敏感 或 `IgnoreCaseSync`，施加 `strings.ToLower`
3. 最后追加 "D" 或 "F"

#### 2.2.2 matchListings 匹配逻辑（第 348-371 行）

```go
srcType := fs.DirEntryType(src)   // "object" 或 "directory"
dstType := fs.DirEntryType(dst)   // "object" 或 "directory"

if srcName > dstName || (srcName == dstName && srcType > dstType) {
    dstOnly(dst)       // 目标端条目在源端无匹配
} else if srcName < dstName || (srcName == dstName && srcType < dstType) {
    srcOnly(src)       // 源端条目在目标端无匹配
} else {
    match(dst, src)    // 唯一可达路径：srcName == dstName && srcType == dstType
}
```

由于排序键已包含 D/F 后缀，`srcName == dstName` 意味着两端的基础名和类型后缀都相同，因此 **`match()` 只在同类型条目之间被调用**。

`srcType`/`dstType` 的字符串比较（"directory" < "object"，因 'd' < 'o'）是一个额外的安全兜底，但正常流程中不会触发——因为 D/F 后缀已保证不同类型的排序键不同。

#### 2.2.3 同名文件 vs 目录的实际调度路径

**场景 A：源有目录 "foo"，目标有文件 "foo"**

```
1. srcName="fooD" < dstName="fooF"
   → srcOnly(srcDir)  → Callback.SrcOnly(directory)
   → checkMarch.SrcOnly: directory 分支 → recurse=true（递归进入子目录，不报告差异）

2. 下一轮：src 前进到下一个条目，dst 仍持有文件 "foo"
   → 若 src 无更多条目或下一条目键 > "fooF"
     → dstOnly(dstFile) → Callback.DstOnly(object)
     → checkMarch.DstOnly: object 分支 → differences++, srcFilesMissing++, report('-', MissingOnSrc)
```

**场景 B：源有文件 "foo"，目标有目录 "foo"**

```
1. srcName="fooF" > dstName="fooD"
   → dstOnly(dstDir)  → Callback.DstOnly(directory)
   → checkMarch.DstOnly: directory 分支 → recurse=true（递归进入子目录，不报告差异）

2. 下一轮：dst 前进到下一个条目，src 仍持有文件 "foo"
   → 若 dst 无更多条目或下一条目键 > "fooF"
     → srcOnly(srcFile) → Callback.SrcOnly(object)
     → checkMarch.SrcOnly: object 分支 → differences++, dstFilesMissing++, report('+', MissingOnDst)
```

**总结**：同名文件与目录冲突时，目录侧静默递归（不报告该名称本身的差异），文件侧报告为缺失。目录子内容在递归中逐一报告。

### 2.3 checkMarch 回调：三层比对

fs/operations/check.go `checkMarch` 实现 `Marcher` 接口，三个回调分别处理：

#### DstOnly（第 79-101 行）：条目仅存在于目标

```
Object 分支:
  if OneWay → 忽略（不报告）
  else      → "file not in source"，differences++，srcFilesMissing++
              report(MissingOnSrc, '-')

Directory 分支:
  if OneWay → 忽略
  else      → recurse=true（递归进入，不报告差异）
```

#### SrcOnly（第 104-120 行）：条目仅存在于源

```
Object 分支:
  始终报告 → "file not in destination"，differences++，dstFilesMissing++
             report(MissingOnDst, '+')

Directory 分支:
  recurse=true（递归进入，不报告差异）
```

#### Match（第 141-203 行）：双方都有同名且同类型的条目

由于排序键包含类型后缀，**Match 回调只可能收到同类型条目**。代码中虽有类型不匹配的防御分支，但正常流程不可达：

```
src 是 Object, dst 是 Object（唯一可达的文件路径）
  ├── SkipDestructive → 跳过
  └── checkIdentical() → goroutine
      ├── sizeDiffers() == true  → differ=true  → report(Differ, '*')  differences++
      ├── SizeOnly 模式          → differ=false → report(Match, '=')   matches++
      └── opt.Check() 返回 (differ, noHash, err)
          ├── err != nil      → report(Error, '!')   CountError
          ├── differ == true  → report(Differ, '*')  differences++
          └── differ == false
              ├── noHash == true  → matches++ noHashes++  "could not check hash"
              └── noHash == false → matches++  "OK"

src 是 Directory, dst 是 Directory（唯一可达的目录路径）
  └── recurse=true

src 是 Object, dst 非 Object（第 178-184 行，不可达防御分支）
  └── report(MissingOnDst, '+')  differences++  dstFilesMissing++

src 是 Directory, dst 非 Directory（第 186-197 行，不可达防御分支）
  └── report(MissingOnSrc, '-')  differences++  srcFilesMissing++
```

---

## 3. 哈希选择逻辑

### 3.1 默认 Check 模式：交集取一

fs/operations/check.go `Check()` 内部构造 `checkFn`：

```go
same, ht, err := CheckHashes(ctx, src, dst)
if ht == hash.None → noHash=true（无法校验哈希，但不算差异）
if !same           → differ=true（哈希不同，报告差异）
```

### 3.2 CheckHashes：源目标哈希集合求交

fs/operations/operations.go `CheckHashes()` 的核心：

```go
common := src.Fs().Hashes().Overlap(dst.Fs().Hashes())
if common.Count() == 0 → return true, hash.None, nil  // 无公共哈希，视为相等但不精确
equal, ht, _, _, err = checkHashes(ctx, src, dst, common.GetOne())  // 取一种公共哈希
```

**Set.Overlap()**（fs/hash/hash.go）：位与运算 `Set(int(h) & int(t))`，求两种 Set 的交集。

**Set.GetOne()**（fs/hash/hash.go）：返回最低位的哈希类型（即最早注册的那个）。当前注册顺序为：

```
MD5(0) → SHA1(1) → Whirlpool(2) → CRC32(3) → SHA256(4) → SHA512(5) → BLAKE3(6) → XXH3(7) → XXH128(8)
```

因此当源与目标同时支持多种哈希时，优先级为 MD5 > SHA1 > Whirlpool > ... > XXH128。

### 3.3 checkHashes：并行计算双端哈希

fs/operations/operations.go `checkHashes()` 使用 `errgroup` 并行获取两端哈希：

```
g.Go → src.Hash(ctx, ht)
g.Go → dst.Hash(ctx, ht)

任一端哈希为空字符串 → errNoHash → return true, hash.None  (视为相等但无哈希)
任一端返回 error      → return false, ht, err               (报错)
两端哈希非空且不等    → return false, ht, nil                (差异)
两端哈希相等          → return true, ht, nil                  (匹配)
```

### 3.4 命令层的哈希选择

在 cmd/check/check.go 第 194-199 行：

```go
hashType := fsrc.Hashes().Overlap(fdst.Hashes()).GetOne()
if hashType == hash.None → Errorf "No common hash found"
else                     → Infof "Using %v for hash comparisons"
```

**注意**：命令层只做日志提示，不影响 `Check()` 的行为。`Check()` 内部的 `checkFn` 会再次调用 `CheckHashes()` 执行同样的交集逻辑。

### 3.5 其他哈希选择场景

| 命令 | 哈希来源 | 选择方式 |
|------|----------|----------|
| `rclone check` | 源 ∩ 目标 | `Overlap().GetOne()` |
| `rclone check --download` | 不使用哈希 | 逐字节 `CheckEqualReaders` |
| `rclone check --checkfile MD5` | 用户指定 | `hashType.Set(checkFileHashType)` |
| `rclone checksum SHA1 sumfile dst:` | 用户指定 | `hashType.Set(args[0])` |
| `rclone cryptcheck` | 底层远端 | `funderlying.Hashes().GetOne()` |

### 3.6 支持的哈希类型

在 fs/hash/hash.go `init()` 中注册：

| 类型 | 位 | 宽度 | 说明 |
|------|----|------|------|
| MD5 | 1<<0 | 32字符 | 最优先 |
| SHA1 | 1<<1 | 40字符 | |
| Whirlpool | 1<<2 | 128字符 | |
| CRC32 | 1<<3 | 8字符 | |
| SHA256 | 1<<4 | 64字符 | |
| SHA512 | 1<<5 | 128字符 | |
| BLAKE3 | 1<<6 | 64字符 | |
| XXH3 | 1<<7 | 16字符 | 非加密哈希 |
| XXH128 | 1<<8 | 32字符 | 非加密哈希 |

---

## 4. 差异报告的关键分支

### 4.1 报告符号体系

fs/operations/check.go `checkMarch.report()` 定义了统一的符号标记：

| 符号 | 含义 | 输出目标 | 触发条件 |
|------|------|----------|----------|
| `=` | 匹配 | Match | 大小+哈希/内容一致，或无哈希可校但大小相同 |
| `-` | 源缺失 | MissingOnSrc | 目标有文件但源没有（DstOnly Object 分支） |
| `+` | 目标缺失 | MissingOnDst | 源有文件但目标没有（SrcOnly Object 分支） |
| `*` | 内容不同 | Differ | 大小不同或哈希/内容不一致 |
| `!` | 错误 | Error | 哈希计算或读取失败 |

注意：目录条目在 DstOnly/SrcOnly 中递归进入子目录，**不产生差异符号**。只有 Object 条目才触发 `-` 或 `+` 报告。

### 4.2 完整比对分支树

综合 matchListings 调度 + checkMarch 回调，文件 "foo" 的完整判定路径：

```
matchListings 比较 srcKey 与 dstKey
│
├── "fooF" == "fooF"（两端都是同名文件）→ match() → checkMarch.Match
│   ├── sizeDiffers() == true
│   │   └── report(Differ, '*')  differences++
│   ├── SizeOnly 模式
│   │   └── report(Match, '=')   matches++
│   └── opt.Check() → (differ, noHash, err)
│       ├── err != nil      → report(Error, '!')   CountError
│       ├── differ == true  → report(Differ, '*')  differences++
│       └── differ == false
│           ├── noHash == true  → matches++  noHashes++  "could not check hash"
│           └── noHash == false → matches++  "OK"
│
├── "fooD" == "fooD"（两端都是同名目录）→ match() → checkMarch.Match
│   └── recurse=true（递归进入子目录）
│
├── "fooD"(src) < "fooF"(dst)（源是目录，目标是同名文件）→ srcOnly + 后续 dstOnly
│   ├── srcOnly(srcDir) → SrcOnly(Directory) → recurse=true（静默递归，不报告）
│   └── dstOnly(dstFile) → DstOnly(Object)
│       ├── OneWay → 忽略
│       └── 非OneWay → report(MissingOnSrc, '-')  differences++  srcFilesMissing++
│
└── "fooF"(src) > "fooD"(dst)（源是文件，目标是同名目录）→ dstOnly + 后续 srcOnly
    ├── dstOnly(dstDir) → DstOnly(Directory)
    │   ├── OneWay → 忽略
    │   └── 非OneWay → recurse=true（静默递归，不报告）
    └── srcOnly(srcFile) → SrcOnly(Object)
        └── report(MissingOnDst, '+')  differences++  dstFilesMissing++
```

### 4.3 checkFn 内部分支

**默认哈希模式** — fs/operations/check.go `Check()` 构造的 checkFn：

```
CheckHashes(ctx, src, dst) → (same, ht, err)
├── err != nil     → differ=true, noHash=false  (哈希计算出错)
├── ht == None     → differ=false, noHash=true   (无公共哈希)
├── !same          → differ=true, noHash=false   (哈希不同)
└── same           → differ=false, noHash=false   (哈希一致)
```

**下载模式** — fs/operations/check.go `CheckDownload()` 构造的 checkFn：

```
CheckIdenticalDownload(ctx, src, dst) → (same, err)
├── err != nil  → differ=true, noHash=true   (下载失败)
├── !same       → differ=true, noHash=false  (内容不同)
└── same        → differ=false, noHash=false  (内容一致)
```

### 4.4 CheckSum 模式的 matchSum 分支

fs/operations/check.go `matchSum()` 针对基于 SUM 文件的校验：

```
matchSum(sumHash, objHash, ...)
├── err != nil          → report(Error, '!')   "Failed to calculate hash"
├── sumHash == ""       → report(Error, '!')   "duplicate file"
├── objHash == ""       → report(Match, '=')   noHashes++  "could not check hash"
├── objHash == sumHash  → report(Match, '=')   "OK"
└── default             → report(Differ, '*')   differences++  "files differ"
```

### 4.5 最终结果汇总

fs/operations/check.go `reportResults()` 输出总结并决定退出码：

```
reportResults()
├── Log "X files missing"           (dstFilesMissing)
├── Log "X files/hashes missing"    (srcFilesMissing)
├── Log "X differences found"       (differences)
├── Log "X errors while checking"   (如有运行时错误)
├── Log "X hashes could not be checked" (noHashes)
├── Log "X matching files"          (matches)
└── return
    ├── err != nil        → 返回原始错误
    ├── differences > 0   → 返回 FsError("X differences found")
    └── differences == 0  → 返回 nil (成功)
```

---

## 5. cryptcheck 特殊链路

cmd/cryptcheck/cryptcheck.go `cryptCheck()` 专门处理加密远端：

```
1. 验证 fdst 是 crypt 远端
2. 取底层远端 funderlying = fcrypt.UnWrap()
3. 选择哈希：funderlying.Hashes().GetOne()
   └── 若 None → 直接报错退出（cryptcheck 要求底层必须支持哈希）
4. 构造自定义 checkFn：
   ├── 取底层对象哈希 underlyingDst.Hash(ctx, hashType)
   ├── 计算 crypt 哈希 fcrypt.ComputeHash(ctx, cryptDst, src, hashType)
   └── 比较两者
       ├── 任一为空 → noHash=true
       └── 不相等   → differ=true
5. 调用 CheckFn(ctx, opt)
```

与普通 `Check` 的关键区别：
- 哈希只从目标侧的**底层远端**选择（`GetOne()` 而非 `Overlap`）
- 通过 `fcrypt.ComputeHash()` 在源端加密后计算哈希来比对
- 若底层不支持任何哈希则直接失败，不会退化到"无哈希"模式

---

## 6. 关键数据流总结

```
CLI 参数解析 (cmd/check)
    │
    ├── checkfile? → CheckSum()  ── ParseSumFile → ListFn → checkSum → matchSum
    ├── download?  → CheckDownload() ── CheckIdenticalDownload → CheckEqualReaders
    └── default    → Check() ── CheckHashes → checkHashes
    │
    ▼
CheckFn() ── march.Run() ── matchListings（排序键含 D/F 后缀）
    │                           │
    │              ┌────────────┼────────────┐
    │              ▼            ▼            ▼
    │          DstOnly      SrcOnly       Match
    │           │  │          │  │         │  │
    │           │  │          │  │         │  │
    │     Object  Dir    Object  Dir   Object  Dir
    │       │      │       │      │      │       │
    │     '-'    递归    '+'    递归  check   递归
    │    diff++  (静默)  diff++ (静默)  identical
    │
    ▼
reportResults() → 退出码
```

同名文件 vs 目录冲突的专有路径：
```
src=目录"foo" dst=文件"foo"     src=文件"foo" dst=目录"foo"
    │                              │
    ├── srcOnly(dir) → 递归         ├── dstOnly(dir) → 递归
    │   (不报告差异)                │   (不报告差异)
    │                              │
    └── dstOnly(file) → '-'        └── srcOnly(file) → '+'
        srcFilesMissing++              dstFilesMissing++
```
