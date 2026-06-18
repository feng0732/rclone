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

#### 2.2.4 NoTraverse 模式：不遍历目标定位对象

`--no-traverse` 标志下，march 不列出目标目录树，而是对每个源条目通过 `NewObject()` 按需定位目标对象。此模式在源文件较少、目标文件很多时性能更优。

**核心机制**（fs/march/march.go 第 419-491 行）：

```
1. 读取源列表到 originalSrcChan
2. 构建双通道流水线：
   ├── matchTasks (chan matchTask)   → workers 并行处理
   └── dstMatches (chan <-chan DirEntry) → 保证顺序输出
3. 每个 worker 处理 matchTask：
   ├── 目录条目：直接返回 nil（NewObject 不能匹配目录，没有"目录匹配"概念）
   ├── 文件条目：NewObject(m.Ctx, path.Join(job.dstRemote, leaf))
      ├── 成功 → 返回目标对象
      └── 失败 → 返回 nil（表示目标不存在）
4. dstMatches 按输入顺序读回结果，发送到 dstChan
5. srcChan 重新发送原始源条目，与 dstChan 一一对应
```

**并发控制**：使用 `newObjectSem` 信号量（`ci.Checkers` 个槽位）限制同时调用 `NewObject` 的数量，避免对目标后端造成压力。

**顺序保证**：即使 worker 乱序完成，`dstMatches` 通道队列也确保结果按原始源顺序发送到 `dstChan`。

**目录不会匹配为目标目录**：
- src 是 Directory 时，worker 直接 `t.dstMatch <- nil`（第 454-457 行）
- matchListings 中 `dst == nil` 走 `case dst == nil: srcOnly(src)`（第 368-370 行）
- 回调 SrcOnly(Directory) → checkMarch 返回 recurse=true
- march 为其创建新 listDirJob：`srcRemote=src.Remote(), dstRemote=src.Remote(), noDst=true`
- **关键点**：NoTraverse 分支（第 421 行）条件是 `m.NoTraverse && !m.NoCheckDest`，不依赖 `job.noDst`，因此子目录 job 仍然走 NewObject 流程处理其下的文件
- 子目录下的文件仍通过 NewObject 按需定位目标对象，目录本身永远不会被"匹配"

**局限性**：
- 目录条目永远走 SrcOnly(Directory) → recurse，没有"目录 match"的路径
- 无法检测目标侧多余文件（dstChan 永远不会有源中没有的条目）

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

### 2.4 目标目录不存在时的递归报告

当目标目录不存在时（`dstListErr == fs.ErrorDirNotFound`），march 框架有特殊处理，触发整个源侧的递归报告。

**错误处理分支**（fs/march/march.go 第 543-545 行）：

```go
if dstListErr == fs.ErrorDirNotFound {
    // Copy the stuff anyway
} else if dstListErr != nil {
    // 其他错误：记录错误并返回
}
```

`ErrorDirNotFound` 被静默忽略，不报错返回，允许流程继续。

**递归报告链路**：

```
processJob 处理某目录（顶层，noDst=false，未开启 NoTraverse）
│
├── dstListDir(job.dstRemote) → fs.ErrorDirNotFound
│
├── dstChan 被关闭（dstListDir 返回后 wg goroutine close(dstChan)）
│
└── matchListings() 比较
    │
    ├── dstHasMore 始终为 false（dstChan 已关闭）
    │
    ├── 每个 src 条目都走 srcOnly()  →  Callback.SrcOnly(src)
    │   │
    │   ├── src 是 Object  →  report('+', MissingOnDst), differences++, dstFilesMissing++
    │   │
    │   └── src 是 Directory  →  checkMarch.SrcOnly 返回 recurse=true
    │       │
    │       └── march 生成新 listDirJob
    │           ├── srcRemote = src.Remote()
    │           ├── dstRemote = src.Remote()
    │           ├── srcDepth = job.srcDepth - 1
    │           └── noDst = true  (跳过目标列表)
    │
    └── 新 job 入队继续处理
        │
        └── processJob 处理子目录（noDst=true）
            │
            ├── 第 407 行条件 !m.NoTraverse && !job.noDst → false（noDst=true）
            ├── 不调用 dstListDir，startedDst 保持 false
            ├── 第 492 行 if !startedDst { close(dstChan) }
            │
            └── dstChan 直接关闭，无任何 dst 条目
                └── 所有 src 条目走 srcOnly()
                    ├── Object → report('+')
                    └── Directory → recurse=true → 生成 noDst=true 的孙目录 job
                        └── 重复此流程（不再调用 dstListDir，不再产生 ErrorDirNotFound）
```

**job 创建细节**（fs/march/march.go 第 497-506 行）：

```go
srcOnly(src) → recurse=true && srcDepth > 0 →
jobs = append(jobs, listDirJob{
    srcRemote: src.Remote(),
    dstRemote: src.Remote(),
    srcDepth:  job.srcDepth - 1,
    noDst:     true,   // 不列出目标，后续子 job 不再调用 dstListDir
})
```

**关键修正点**：
- 只有顶层 job 会真正调用 `dstListDir` 并得到 `ErrorDirNotFound`
- 子 job 设置 `noDst=true` 后，第 407 行的 `!job.noDst` 条件不成立，**不再调用 dstListDir**
- 子 job 的 `dstChan` 被直接关闭（`startedDst=false → close(dstChan)`），不会再产生 ErrorDirNotFound
- 所有子层级的"目标不存在"是通过 dstChan 为空来驱动的，而非通过 ErrorDirNotFound

**效果**：目标目录不存在时，源目录树被完整遍历，每个文件都报告为目标缺失（'+'），目录逐层递归，不会因顶层目录不存在而中断。

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

### 4.4 CheckSum 模式完整链路

CheckSum 模式不使用 march 框架，而是采用 "清单解析 → 文件系统遍历 → 双边比对 → 未消费清单兜底" 的独立流程。

#### 4.4.1 CheckSum 初始化与参数倒置

`CheckSum()`（fs/operations/check.go 第 409-469 行）入口有特殊的参数约定：

```go
// CheckSum 中 Fsrc 和 Fdst 被重新赋值：
options.Fsrc = nil    // 源 Fs 被置空，表示"源"是 SUM 清单而非文件系统
options.Fdst = fsrc   // 待校验的文件系统作为"目标"
```

这会影响后续报告中"files missing" vs "hashes missing" 的文案（见 reportResults 第 246-248 行）。

#### 4.4.2 清单解析与消费

**步骤 1：解析 SUM 文件**（第 429-436 行）

```
sumObj, err := fsum.NewObject(ctx, sumFile)
hashes, err := ParseSumFile(ctx, sumObj)
  → 正则 `^([^ ]+) [ *](.+)$` 匹配每一行
  → 结果保存为 HashSums map[string]string
  → key = 文件名（经 ApplyTransforms 规范化：NFC + 可选大小写折叠）
  → value = 哈希值（小写）
```

**步骤 2：遍历文件系统**（第 444-447 行）

```go
lastErr := ListFn(ctx, opt.Fdst, func(obj fs.Object) {
    c.checkSum(ctx, obj, download, hashes, hashType)
})
```

对每个文件调用 `checkSum()`（第 472-533 行）进行消费：

```go
normalizedRemote := ApplyTransforms(ctx, obj.Remote())
c.ioMu.Lock()
sumHash, sumFound := hashes[normalizedRemote]
hashes[normalizedRemote] = ""  // 消费标记：设为空字符串
c.ioMu.Unlock()
```

**消费规则**：
- 查找到对应哈希条目 → 标记为消费（置空），进入比对流程
- 未查找到 → 根据 `OneWay` 标志决定：
  - `OneWay=true` → 直接 return（不报告系统中多余的文件）
  - `OneWay=false` → 报告为 src 缺失（'-', MissingOnSrc）

#### 4.4.3 比对分支（download vs 非 download）

**非 download 模式**（第 498-503 行）：直接调用对象的 `Hash()` 方法获取哈希。

```go
if !download {
    objHash, err = obj.Hash(ctx, hashType)
    c.matchSum(ctx, sumHash, objHash, obj, err, hashType)
    return
}
```

**download 模式**（第 505-532 行）：下载文件内容并流式计算哈希。

```
Open(ctx, obj) → in io.ReadCloser
  → tr.Account(ctx, in).WithBuffer()  // 记账和缓冲
  → hash.StreamTypes(in, hash.NewHashSet(hashType))
    → 流式读取并计算指定哈希
    → objHash = hashVals[hashType]
  → matchSum 报告结果
```

#### 4.4.4 matchSum 结果分支

`matchSum()`（fs/operations/check.go 第 536-566 行）的完整判定：

```
matchSum(sumHash, objHash, obj, err, hashType)
├── err != nil          → report(Error, '!')   "Failed to calculate hash"
├── sumHash == ""       → report(Error, '!')   "duplicate file"（重复消费）
├── objHash == ""       → report(Match, '=')   noHashes++  "could not check hash"
├── objHash == sumHash  → report(Match, '=')   "OK"
└── default             → report(Differ, '*')   differences++  "files differ"
```

**sumHash == ""** 意味着该条目已被另一个同名文件消费过（例如经过规范化后文件名相同的两个文件），此时报告为重复文件错误。

#### 4.4.5 未消费清单兜底（第 449-467 行）

遍历完文件系统后，对 `hashes` map 中仍未消费的条目进行兜底报告：

```go
for filename, hash := range hashes {
    if hash == "" {          // 已消费，跳过
        continue
    }
    if !fi.IncludeRemote(filename) {  // 被过滤器排除，跳过
        continue
    }
    // 清单中有但文件系统中没有，报告为目标缺失
    err := fmt.Errorf("file not in %v", opt.Fdst)
    c.dstFilesMissing.Add(1)
    c.reportFilename(filename, opt.MissingOnDst, '+')
}
```

**注意**：此处使用 `reportFilename` 直接输出文件名（而非 `report` 输出 DirEntry），因为这些文件在文件系统中不存在，没有对应的 DirEntry 对象。

#### 4.4.6 OneWay 单向校验逻辑

`--one-way` 标志对 CheckSum 模式的影响体现在两个位置：

| 位置 | OneWay=false | OneWay=true |
|------|-------------|------------|
| `checkSum()` 第 480-482 行 | 系统中有但清单中没有 → 报告为 src 缺失（'-'） | 系统中有但清单中没有 → 直接 return，不报告 |
| 未消费清单兜底 | 清单中有但系统中没有 → 始终报告为 dst 缺失（'+'） | 清单中有但系统中没有 → 始终报告为 dst 缺失（'+'）|

**OneWay 语义**：只校验"清单中有的文件必须在系统中存在且匹配"，不校验"系统中多余的文件"。

**关键区别于普通 Check 的 OneWay**：
- 普通 Check 的 OneWay 影响 DstOnly（目标有但源没有）→ 不报告
- CheckSum 的 OneWay 影响 checkSum（系统有但清单没有）→ 不报告
- 两者在"源/清单侧有但目标/系统侧没有"的方向上，OneWay 不影响，始终报告

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

### 6.1 普通 Check 模式（遍历目标）

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

### 6.2 NoTraverse 模式（不遍历目标定位对象）

```
CheckFn() ── march.Run(NoTraverse=true)
    │
    ├── processJob
    │   ├── srcListDir  →  originalSrcChan
    │   ├── 不调用 dstListDir
    │   │
    │   ├── matchTasks 队列（ci.Checkers 个 worker 并行）
    │   │   ├── src 是 Directory → dstMatch <- nil（目录永远无法匹配）
    │   │   └── src 是 Object
    │   │       ├── NewObject(path.Join(dstRemote, leaf))
    │   │       ├── 成功 → dstMatch <- dstObject
    │   │       └── 失败 → dstMatch <- nil
    │   │
    │   └── dstMatches 按顺序读回结果 → dstChan
    │
    └── matchListings(srcChan, dstChan)
        ├── src=Object, dst=Object → match → checkIdentical
        ├── src=Object, dst=nil    → srcOnly → report('+', MissingOnDst)
        ├── src=Directory, dst=nil → srcOnly → recurse=true
        │   └── 新 listDirJob{srcRemote=dir, dstRemote=dir, noDst=true}
        │       └── NoTraverse 分支不受 noDst 影响，子目录下文件继续 NewObject
        └── src=nil, dst=* → 不会发生（dstChan 只有 src 中的条目，且目录返回 nil）
```

**关键特征**：目录永远不会通过 NewObject 匹配目标目录，而是走 SrcOnly → recurse → 子 job 继续 NoTraverse 流程。

### 6.3 目标目录不存在时的递归报告

```
processJob 处理顶层目录 X（noDst=false，非 NoTraverse）
│
├── dstListDir(X) → fs.ErrorDirNotFound（仅顶层产生此错误）
├── dstChan 被 dstListDir 所在 goroutine 关闭（dstHasMore=false）
│
└── matchListings()
    │
    ├── dst 始终为 nil
    │
    ├── src=Object → srcOnly → report('+', MissingOnDst)
    └── src=Directory → srcOnly → recurse=true
        └── 新 listDirJob{srcRemote=dir, dstRemote=dir, noDst=true}
            │
            └── processJob 子目录（noDst=true）
                ├── 第 407 行 !m.NoTraverse && !job.noDst → false
                ├── 不调用 dstListDir，startedDst=false
                ├── close(dstChan)（第 492 行）
                │
                └── matchListings()
                    ├── dst=nil → 所有 src 走 srcOnly
                    │   ├── Object → '+'
                    │   └── Directory → recurse → 生成 noDst=true 孙目录 job
                    │       └── 重复此流程（不再调用 dstListDir，不再产生 ErrorDirNotFound）
                    └── 直到 srcDepth 耗尽或所有文件报告完毕
```

**关键特征**：
- 仅顶层 job 产生 `ErrorDirNotFound`，子 job 因 `noDst=true` 根本不调用 `dstListDir`
- 子层级通过 `close(dstChan)` 驱动所有 src 走 `srcOnly`，而非通过 ErrorDirNotFound

### 6.4 CheckSum 模式：清单消费与单向校验

```
CheckSum()
│
├── ParseSumFile(sumFile) → HashSums map (key=文件名, value=哈希)
│
├── ListFn(fdst) → 遍历文件系统每个文件
│   │
│   └── checkSum(obj, hashes, hashType)
│       ├── normalizedRemote = ApplyTransforms(obj.Remote())
│       ├── sumHash, sumFound := hashes[normalizedRemote]
│       ├── hashes[normalizedRemote] = ""  // 消费标记
│       │
│       ├── !sumFound
│       │   ├── OneWay=true → return（不报告）
│       │   └── OneWay=false → report('-', MissingOnSrc)
│       │
│       └── sumFound
│           ├── !download → obj.Hash(ctx, hashType)
│           └── download → Open(obj) → StreamTypes → 计算哈希
│               └── matchSum() → 报告 '=', '*', 或 '!'
│
├── 遍历结束，兜底检查未消费清单
│   │
│   └── for filename, hash := range hashes
│       ├── hash == "" → 已消费，跳过
│       ├── !IncludeRemote → 被过滤，跳过
│       └── 其他 → report('+', MissingOnDst)
│
└── reportResults() → 退出码
```

### 6.5 同名文件 vs 目录冲突的专有路径

```
src=目录"foo" dst=文件"foo"     src=文件"foo" dst=目录"foo"
    │                              │
    ├── srcOnly(dir) → 递归         ├── dstOnly(dir) → 递归
    │   (不报告差异)                │   (不报告差异)
    │                              │
    └── dstOnly(file) → '-'        └── srcOnly(file) → '+'
        srcFilesMissing++              dstFilesMissing++
```
