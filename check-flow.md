# rclone check 校验链路分析

## 1. 整体架构概览

rclone 的 check 校验链路由三层组成：

| 层级 | 入口 | 核心逻辑 | 作用 |
|------|------|----------|------|
| 命令层 | [check.go](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/cmd/check/check.go) | 解析参数，选择校验模式 | CLI 入口 |
| 调度层 | [march.go](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/march/march.go) | 并行遍历源与目标目录树 | 对齐文件名 |
| 校验层 | [check.go(operations)](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/operations/check.go) | 大小比对 → 哈希/下载比对 → 差异上报 | 判定一致性 |

辅助命令：

- [cryptcheck.go](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/cmd/cryptcheck/cryptcheck.go)：加密远端的专用校验
- [checksum.go](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/cmd/checksum/checksum.go)：基于 SUM 文件的校验

---

## 2. 源目标比对流程

### 2.1 命令入口与模式选择

在 [check.go#L162-L203](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/cmd/check/check.go#L162-L203) 中，`rclone check` 根据参数选择三种校验模式：

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

[March.Run()](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/march/march.go#L184-L272) 启动并行遍历，核心在 [matchListings()](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/march/march.go#L292-L374)：

```
srcChan ──┐
          ├── matchListings() ──┬── srcOnly()   → Callback.SrcOnly()
dstChan ──┘                    ├── dstOnly()   → Callback.DstOnly()
                               └── match()     → Callback.Match()
```

**文件名匹配逻辑（第 348-371 行）：**

- 经 `transforms`（NFC 规范化 + 可选大小写折叠）后，按排序键比较 `srcName` 与 `dstName`
- `srcName > dstName` → 仅目标有，调 `dstOnly`
- `srcName < dstName` → 仅源有，调 `srcOnly`
- `srcName == dstName` 且类型相同 → 双方都有，调 `match`

**名称变换链（第 70-82 行）：**

1. 若 `!NoUnicodeNormalization`，施加 NFC 规范化
2. 若目标 Fs 大小写不敏感 或 `IgnoreCaseSync`，施加 `strings.ToLower`

### 2.3 checkMarch 回调：三层比对

[checkMarch](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/operations/check.go#L50-L62) 实现 `Marcher` 接口，三个回调分别处理：

#### DstOnly（第 79-101 行）：文件仅存在于目标

```
if OneWay → 忽略（不报告）
else → 记录 "file not in source"，differences++，srcFilesMissing++
       报告到 MissingOnSrc（符号 '-'）
```

#### SrcOnly（第 104-120 行）：文件仅存在于源

```
始终报告 → "file not in destination"，differences++，dstFilesMissing++
报告到 MissingOnDst（符号 '+'）
```

#### Match（第 141-203 行）：双方都有同名文件

```
src 是 Object && dst 是 Object
  → 并行 checkIdentical()
      → 1. sizeDiffers() — 大小不同 → differ=true
      → 2. SizeOnly 模式 → 跳过内容比对
      → 3. 调用 opt.Check() — 哈希或下载比对

src 是 Directory && dst 是 Directory
  → 递归进入子目录

src/dst 类型不一致（文件 vs 目录）
  → 报告为缺失，differences++
```

---

## 3. 哈希选择逻辑

### 3.1 默认 Check 模式：交集取一

[Check()](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/operations/check.go#L275-L294) 内部构造 `checkFn`：

```go
same, ht, err := CheckHashes(ctx, src, dst)
if ht == hash.None → noHash=true（无法校验哈希，但不算差异）
if !same           → differ=true（哈希不同，报告差异）
```

### 3.2 CheckHashes：源目标哈希集合求交

[CheckHashes()](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/operations/operations.go#L60-L68) 的核心：

```go
common := src.Fs().Hashes().Overlap(dst.Fs().Hashes())
if common.Count() == 0 → return true, hash.None, nil  // 无公共哈希，视为相等但不精确
equal, ht, _, _, err = checkHashes(ctx, src, dst, common.GetOne())  // 取一种公共哈希
```

**[Set.Overlap()](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/hash/hash.go#L336-L338)**：位与运算 `Set(int(h) & int(t))`，求两种 Set 的交集。

**[Set.GetOne()](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/hash/hash.go#L349-L360)**：返回最低位的哈希类型（即最早注册的那个）。当前注册顺序为：

```
MD5(0) → SHA1(1) → Whirlpool(2) → CRC32(3) → SHA256(4) → SHA512(5) → BLAKE3(6) → XXH3(7) → XXH128(8)
```

因此当源与目标同时支持多种哈希时，优先级为 MD5 > SHA1 > Whirlpool > ... > XXH128。

### 3.3 checkHashes：并行计算双端哈希

[checkHashes()](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/operations/operations.go#L74-L122) 使用 `errgroup` 并行获取两端哈希：

```
g.Go → src.Hash(ctx, ht)
g.Go → dst.Hash(ctx, ht)

任一端哈希为空字符串 → errNoHash → return true, hash.None  (视为相等但无哈希)
任一端返回 error      → return false, ht, err               (报错)
两端哈希非空且不等    → return false, ht, nil                (差异)
两端哈希相等          → return true, ht, nil                  (匹配)
```

### 3.4 命令层的哈希选择

在 [check.go#L194-L199](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/cmd/check/check.go#L194-L199)：

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

在 [hash.go#L122-L132](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/hash/hash.go#L122-L132) 中注册：

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

[checkMarch.report()](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/operations/check.go#L65-L76) 定义了统一的符号标记：

| 符号 | 含义 | 输出目标 | 触发条件 |
|------|------|----------|----------|
| `=` | 匹配 | Match | 哈希/内容一致 |
| `-` | 源缺失 | MissingOnSrc | 目标有但源没有 |
| `+` | 目标缺失 | MissingOnDst | 源有但目标没有 |
| `*` | 内容不同 | Differ | 大小或哈希不一致 |
| `!` | 错误 | Error | 哈希计算或读取失败 |

### 4.2 Match 回调中的分支树

[checkMarch.Match()](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/operations/check.go#L141-L203) 中完整的判断路径：

```
Match(dst, src)
├── src 是 Object, dst 是 Object
│   ├── SkipDestructive → 跳过
│   └── checkIdentical() → goroutine
│       ├── sizeDiffers() == true
│       │   └── differ=true  →  report(Differ, '*')  differences++
│       ├── SizeOnly 模式
│       │   └── differ=false →  report(Match, '=')    matches++
│       └── opt.Check() 返回 (differ, noHash, err)
│           ├── err != nil      →  report(Error, '!')   CountError
│           ├── differ == true  →  report(Differ, '*')  differences++
│           └── differ == false
│               ├── noHash == true  →  matches++ noHashes++  "could not check hash"
│               └── noHash == false →  matches++  "OK"
├── src 是 Object, dst 非 Object (类型不匹配)
│   └── report(MissingOnDst, '+')  differences++
├── src 是 Directory, dst 是 Directory
│   └── recurse=true
└── src 是 Directory, dst 非 Directory (类型不匹配)
    └── report(MissingOnSrc, '-')  differences++
```

### 4.3 checkFn 内部分支

**默认哈希模式** — [Check() 构造的 checkFn](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/operations/check.go#L277-L291)：

```
CheckHashes(ctx, src, dst) → (same, ht, err)
├── err != nil     → differ=true, noHash=false  (哈希计算出错)
├── ht == None     → differ=false, noHash=true   (无公共哈希)
├── !same          → differ=true, noHash=false   (哈希不同)
└── same           → differ=false, noHash=false   (哈希一致)
```

**下载模式** — [CheckDownload() 构造的 checkFn](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/operations/check.go#L369-L384)：

```
CheckIdenticalDownload(ctx, src, dst) → (same, err)
├── err != nil  → differ=true, noHash=true   (下载失败)
├── !same       → differ=true, noHash=false  (内容不同)
└── same        → differ=false, noHash=false  (内容一致)
```

### 4.4 CheckSum 模式的 matchSum 分支

[matchSum()](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/operations/check.go#L536-L566) 针对基于 SUM 文件的校验：

```
matchSum(sumHash, objHash, ...)
├── err != nil          → report(Error, '!')   "Failed to calculate hash"
├── sumHash == ""       → report(Error, '!')   "duplicate file"
├── objHash == ""       → report(Match, '=')   noHashes++  "could not check hash"
├── objHash == sumHash  → report(Match, '=')   "OK"
└── default             → report(Differ, '*')   differences++  "files differ"
```

### 4.5 最终结果汇总

[reportResults()](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/fs/operations/check.go#L240-L272) 输出总结并决定退出码：

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

[cryptCheck()](file:///d:/fz/0601-2/solo-dogfeeding/code/56-rclone/cmd/cryptcheck/cryptcheck.go#L67-L117) 专门处理加密远端：

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
CheckFn() ── march.Run() ── matchListings
    │                           │
    │              ┌────────────┼────────────┐
    │              ▼            ▼            ▼
    │          DstOnly      SrcOnly       Match
    │              │            │            │
    │              ▼            ▼            ▼
    │         report('-')  report('+')  checkIdentical()
    │                                     │
    │                          ┌──────────┼──────────┐
    │                          ▼          ▼          ▼
    │                     sizeDiffers  SizeOnly?  opt.Check()
    │                          │          │          │
    │                     report('*') report('=')  (哈希/下载)
    │
    ▼
reportResults() → 退出码
```
