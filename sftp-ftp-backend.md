# SFTP 与 FTP 后端 — 错误适配与目录对象边界梳理

本文档对照 `backend/sftp/sftp.go`、`backend/sftp/ssh.go` 与 `backend/ftp/ftp.go`，逐行核对**对象查找**、**目录判断**、**标准错误转换**、**连接归还时的错误分类**四个核心边界。

---

## 一、对象查找 (Object Lookup)

### 1.1 调用链对比

**SFTP 调用链**：

```
NewObject(ctx, remote)                              // sftp.go 第 1342 行
  └─ o.stat(ctx)                                    // sftp.go 第 2210 行
       └─ f.stat(ctx, o.remote)                     // sftp.go 第 2195 行
            └─ c.sftpClient.Stat(absPath)           // 单次 LSTAT/STAT
```

**FTP 调用链**：

```
NewObject(ctx, remote)                              // ftp.go 第 846 行
  └─ f.findItem(ctx, path.Join(f.root, remote))    // ftp.go 第 788 行
       ├─ [MLST 可用] c.GetEntry(encodedPath)       // 单次 MLST
       └─ [MLST 不可用] c.List(dir) + 遍历匹配      // 列父目录后扫描
```

**关键差异**：SFTP 始终用单次 `Stat` 完成查找；FTP 有两条路径——支持 MLST 时用 `GetEntry`，否则退化为列父目录后逐项匹配。

### 1.2 "未找到"的返回方式

| 后端 | 中间层返回 | NewObject 最终返回 |
|------|-----------|-------------------|
| SFTP | `f.stat` 返回包装了 `os.ErrNotExist` 的 error | `o.stat` 检测 `os.IsNotExist(err)` 转换为 `fs.ErrorObjectNotFound` |
| FTP | `findItem` 返回 `(nil, nil)` — entry 为 nil、error 也为 nil | `NewObject` 判断 `entry == nil` → 返回 `fs.ErrorObjectNotFound` |

FTP 的 `findItem` 采用 `(nil, nil)` 表示"未找到"是一个重要设计选择——调用方无法区分"路径不存在"和"路径指向目录"（见下节）。

### 1.3 路径指向目录时的错误

这是两个后端最显著的语义差异：

**SFTP** — Object.stat (sftp.go 第 2210-2223 行)：

```go
// sftp.go 第 2218-2219 行
if info.IsDir() {
    return fs.ErrorIsDir       // ← 目录明确返回 ErrorIsDir
}
```

**FTP** — NewObject (ftp.go 第 846-866 行)：

```go
// ftp.go 第 852-865 行
if entry != nil && entry.Type != ftp.EntryTypeFolder {
    // ...创建 Object
    return o, nil
}
return nil, fs.ErrorObjectNotFound   // ← 目录也返回 ErrorObjectNotFound
```

| 场景 | SFTP NewObject 返回 | FTP NewObject 返回 |
|------|--------------------|--------------------|
| 路径不存在 | `fs.ErrorObjectNotFound` | `fs.ErrorObjectNotFound` |
| 路径是目录 | `fs.ErrorIsDir` | `fs.ErrorObjectNotFound` |
| 路径是文件 | `(*Object, nil)` | `(*Object, nil)` |

**影响**：上层调用者通过 `NewObject` 无法区分 FTP 上"路径不存在"和"路径是目录"两种情况，而 SFTP 可以。

### 1.4 FTP 对象查找遇到 501 状态码的完整返回路径

FTP 的 501 状态码（`StatusBadArguments`）表示服务器不支持 MLST 命令。`findItem` 中有专门的处理逻辑：

```go
// ftp.go findItem 第 805-820 行
if c.IsTimePreciseInList() {
    entry, err := c.GetEntry(...)
    f.putFtpConnection(&c, err)
    if err != nil {
        err = translateErrorFile(err)
        if err == fs.ErrorObjectNotFound {
            return nil, nil              // ← 路径 1：550/450 → (nil, nil)
        }
        if errX := textprotoError(err); errX != nil {
            switch errX.Code {
            case ftp.StatusBadArguments:  // 501
                err = nil                 // ← 路径 2：501 错误被吞没，置为 nil
            }
        }
        return nil, err                   // ← 返回 (nil, nil)
    }
    // entry != nil → 正常返回 entry
    return entry, nil
}
```

遇到 501 后完整的调用链：

```
GetEntry 返回 501 StatusBadArguments
  → translateErrorFile 不匹配 550/450，err 保持不变
  → textprotoError 提取到 Code=501
  → err 被置为 nil
  → findItem 返回 (nil, nil)
  → NewObject 收到 (nil, nil)，判断 entry == nil
  → NewObject 返回 fs.ErrorObjectNotFound
```

501 的触发场景是：**服务器声称支持 MLST（通过 `IsTimePreciseInList`），但实际返回不支持**。当前代码会吞掉这个错误，让外层把它当作“未找到”。

需要注意的是，`IsTimePreciseInList() == true` 的分支在处理完 501 后直接 `return`，不会继续执行后面的 `List` 遍历分支。因此 501 的实际效果是 `findItem` 返回 `(nil, nil)`，`NewObject` 最终返回 `fs.ErrorObjectNotFound`。

---

## 二、目录判断 (Directory Existence)

### 2.1 dirExists 实现对比

**SFTP** — dirExists (sftp.go 第 1356-1376 行)：

```go
// sftp.go 第 1356-1376 行
info, err := c.sftpClient.Stat(dir)   // 直接 Stat
if err != nil {
    if os.IsNotExist(err) {
        return false, nil              // 不存在 → (false, nil)
    }
    return false, fmt.Errorf("dirExists stat failed: %w", err)
}
if !info.IsDir() {
    return false, fs.ErrorIsFile       // ← 是文件 → 返回 ErrorIsFile
}
return true, nil                       // 是目录 → (true, nil)
```

**FTP** — dirExists (ftp.go 第 869-878 行)：

```go
// ftp.go 第 869-878 行
entry, err := f.findItem(ctx, path.Join(f.root, remote))
if err != nil {
    return false, fmt.Errorf("dirExists: %w", err)
}
if entry != nil && entry.Type == ftp.EntryTypeFolder {
    return true, nil                   // 是目录 → (true, nil)
}
return false, nil                      // ← 不存在或是文件 → 统一 (false, nil)
```

### 2.2 返回值矩阵

| 场景 | SFTP dirExists | FTP dirExists |
|------|---------------|---------------|
| 目录存在 | `(true, nil)` | `(true, nil)` |
| 路径不存在 | `(false, nil)` | `(false, nil)` |
| 路径是文件 | `(false, fs.ErrorIsFile)` | `(false, nil)` |
| 连接/协议错误 | `(false, wrapped error)` | `(false, wrapped error)` |

**差异**：SFTP 在路径是文件时返回 `fs.ErrorIsFile`，调用方可以区分"不存在"和"是文件"。FTP 在两种情况下统一返回 `(false, nil)`，丢失了区分信息。

---

## 三、标准错误转换 (Standard Error Translation)

### 3.1 转换函数对比

**SFTP — 无集中式转换函数**，转换散落在各方法中：

| 方法 | 原始错误 | 转换逻辑 | 目标错误 |
|------|---------|---------|---------|
| Object.stat 第 2213 行 | `os.IsNotExist(err)` | `if` 判断 | `fs.ErrorObjectNotFound` |
| Object.stat 第 2218 行 | `info.IsDir()` | `if` 判断 | `fs.ErrorIsDir` |
| List 第 1400 行 | `errors.Is(err, os.ErrNotExist)` | `if` 判断 | `fs.ErrorDirNotFound` |
| dirExists 第 1367 行 | `os.IsNotExist(err)` | `if` 判断 | `(false, nil)` |
| dirExists 第 1372 行 | `!info.IsDir()` | `if` 判断 | `fs.ErrorIsFile` |
| mkdir 第 1494 行 | `os.IsExist(err)` | `if` 判断 | `nil` (幂等) |

**FTP — 集中式转换函数**：

translateErrorFile (ftp.go 第 747-755 行)：

```go
func translateErrorFile(err error) error {
    if errX := textprotoError(err); errX != nil {
        switch errX.Code {
        case ftp.StatusFileUnavailable, ftp.StatusFileActionIgnored:  // 550, 450
            err = fs.ErrorObjectNotFound
        }
    }
    return err   // 不匹配时原样返回
}
```

translateErrorDir (ftp.go 第 758-766 行)：

```go
func translateErrorDir(err error) error {
    if errX := textprotoError(err); errX != nil {
        switch errX.Code {
        case ftp.StatusFileUnavailable, ftp.StatusFileActionIgnored:  // 550, 450
            err = fs.ErrorDirNotFound
        }
    }
    return err
}
```

### 3.2 相同 FTP 状态码，不同 rclone 错误

FTP 的 `translateErrorFile` 和 `translateErrorDir` 对**同一组状态码** (550, 450) 映射到**不同的 rclone 标准错误**：

| FTP 状态码 | 含义 | translateErrorFile | translateErrorDir |
|-----------|------|--------------------|-------------------|
| 550 | File Unavailable | `fs.ErrorObjectNotFound` | `fs.ErrorDirNotFound` |
| 450 | File Action Ignored | `fs.ErrorObjectNotFound` | `fs.ErrorDirNotFound` |

这是因为 FTP 协议本身不区分"文件不存在"和"目录不存在"——都用 550/450 表示。rclone 通过**调用上下文**（文件操作 vs 目录操作）来选择映射目标。

SFTP 不需要这种区分——`pkg/sftp` 库将协议错误统一转换为 Go 标准错误 (`os.ErrNotExist`)，由 `os.IsNotExist` / `errors.Is(err, os.ErrNotExist)` 判断。

### 3.3 完整的错误转换映射表

| rclone 标准错误 | SFTP 触发条件 (代码位置) | FTP 触发条件 (代码位置) |
|-----------------|------------------------|------------------------|
| `fs.ErrorObjectNotFound` | `os.IsNotExist(err)` [Object.stat 第 2213 行] | `translateErrorFile`: 550/450 [第 750 行] |
| `fs.ErrorDirNotFound` | `errors.Is(err, os.ErrNotExist)` [List 第 1400 行] | `translateErrorDir`: 550/450 [第 761 行] |
| `fs.ErrorIsDir` | `info.IsDir()` [Object.stat 第 2218 行] | **不产生** — NewObject 对目录返回 `ErrorObjectNotFound` |
| `fs.ErrorIsFile` | `!info.IsDir()` [dirExists 第 1372 行]；mkdir 会包装该错误 | mkdir 通过 `getInfo` 判断路径是文件后直接返回 `fs.ErrorIsFile` |
| `fs.ErrorDirectoryNotEmpty` | `len(entries) != 0` [Rmdir 第 1527 行] | **不产生** — 直接调用 RemoveDir，依赖服务端返回错误 |

### 3.4 目录创建碰到文件路径时的标准错误

**SFTP** — mkdir (sftp.go 第 1469-1501 行)：

```go
// sftp.go 第 1475-1478 行
ok, err := f.dirExists(ctx, dirPath)
if err != nil {
    return fmt.Errorf("mkdir dirExists failed: %w", err)
}
```

SFTP 的 `dirExists` 在路径是文件时返回 `(false, fs.ErrorIsFile)`。这个 `err` 被 `fmt.Errorf("mkdir dirExists failed: %w", err)` 包装。

`errors.Is(err, fs.ErrorIsFile)` 仍然成立，但错误消息变成了 `"mkdir dirExists failed: ..."`。

**FTP** — mkdir (ftp.go 第 1031-1067 行)：

```go
// ftp.go 第 1036-1041 行
fi, err := f.getInfo(ctx, abspath)
if err == nil {
    if fi.IsDir {
        return nil
    }
    return fs.ErrorIsFile          // ← 直接返回裸标准错误
}
```

FTP 不通过 `dirExists`，而是通过 `getInfo` 独立判断，发现是文件时直接返回裸的 `fs.ErrorIsFile`，不包装。

| 维度 | SFTP mkdir 碰到文件 | FTP mkdir 碰到文件 |
|------|--------------------|--------------------|
| 返回值 | `fmt.Errorf("mkdir dirExists failed: %w", fs.ErrorIsFile)` | `fs.ErrorIsFile` |
| `errors.Is(err, fs.ErrorIsFile)` | ✅ 成立 | ✅ 成立 |
| 错误消息 | 包含包装前缀 `"mkdir dirExists failed: "` | 无包装前缀 |

### 3.5 FTP findItem 的特殊错误吞没

findItem (ftp.go 第 788-842 行) 的 GetEntry 路径中有多处错误吞没：

```go
// ftp.go 第 809-811 行
err = translateErrorFile(err)
if err == fs.ErrorObjectNotFound {
    return nil, nil    // ← ErrorObjectNotFound 被吞没，变为 (nil, nil)
}
```

以及：

```go
// ftp.go 第 813-818 行
if errX := textprotoError(err); errX != nil {
    switch errX.Code {
    case ftp.StatusBadArguments:
        err = nil      // ← 501 Bad Arguments 也被吞没
    }
}
```

**原因**：某些 FTP 服务器对 MLST 命令返回 501，表示不支持该功能。`findItem` 将此视为"未找到"而非错误；由于当前 MLST 分支处理完后直接 return，本次调用不会再走 List 遍历分支。

SFTP 没有类似的多路径回退机制，也不存在错误吞没的情况。

---

## 四、连接归还时的错误分类 (Connection Return Error Classification)

### 4.1 核心发现：逻辑反转

两个后端在 `putXxxConnection` 中的错误分类逻辑是**反转的**。

**SFTP** — putSftpConnection (sftp.go 第 837-876 行)：

```go
// sftp.go 第 846-868 行
if err != nil {
    isRegularError := false
    var statusErr *sftp.StatusError
    var pathErr *os.PathError
    switch {
    case errors.Is(err, os.ErrNotExist):  isRegularError = true
    case errors.As(err, &statusErr):      isRegularError = true
    case errors.As(err, &pathErr):        isRegularError = true
    }
    if !isRegularError {                  // ← 非常规错误才探测
        _, nopErr := c.sftpClient.Getwd()
        if nopErr != nil {
            _ = c.close()
            return
        }
    }
}
// 常规错误 / 无错误 / 探测通过 → 归还池中
```

**SFTP 策略**：协议层业务错误（StatusError / PathError / ErrNotExist）是"常规"的，**跳过**存活探测；其他错误（如网络断开）才探测。

**FTP** — putFtpConnection (ftp.go 第 589-618 行)：

```go
// ftp.go 第 601-610 行
if err != nil {
    if tpErr := textprotoError(err); tpErr != nil {  // ← 是 textproto 错误才探测
        nopErr := c.NoOp()
        if nopErr != nil {
            _ = c.Quit()
            return
        }
    }
}
// 非 textproto 错误 / 无错误 / 探测通过 → 归还池中
```

**FTP 策略**：`textproto.Error`（即 FTP 协议响应错误）才触发探测；非协议错误（如网络错误）**直接归还池中**。

### 4.2 逻辑对比

| 条件 | SFTP 行为 | FTP 行为 |
|------|----------|---------|
| 协议层业务错误 (如文件不存在) | **跳过**探测，直接归还 | **触发**探测，验证连接 |
| 非协议错误 (如网络断开) | **触发**探测 | **跳过**探测，直接归还 |
| 无错误 | 直接归还 | 直接归还 |

两个后端对"何时触发存活探测"的判断逻辑完全相反。SFTP 的逻辑更直觉——协议层错误说明连接正常（服务端正常响应了），不需要检查；网络层错误才需要验证连接。FTP 的逻辑则相反——收到协议响应时反而去验证连接，而网络错误时却直接归还。

> 注：FTP 代码注释写的是 "If not a regular FTP error code then check the connection"，但实际代码条件为 `tpErr != nil`（即**是** textproto 错误时检查），注释与代码矛盾。若注释是正确意图，则代码中的条件应为 `tpErr == nil`。

### 4.3 SFTP 独有：CanReuse 检查

SFTP 在归还连接前还有一个 FTP 没有的检查：

```go
// sftp.go 第 842-844 行
if !c.sshClient.CanReuse() {
    return    // 外部 SSH 进程不可复用，直接丢弃
}
```

这用于外部 SSH 二进制模式（`--sftp-ssh`），该模式下每次操作启动新的 SSH 进程，不可复用。FTP 不支持外部进程模式，因此无此检查。

### 4.4 SFTP 独有：从池中取连接时的存活检查

SFTP 的 `getSftpConnection` 在从池中取连接时会检查连接是否已关闭：

```go
// sftp.go 第 804-813 行
for len(f.pool) > 0 {
    c = f.pool[0]
    f.pool = f.pool[1:]
    err := c.closed()      // ← 检查 SSH 连接是否已关闭
    if err == nil {
        break              // 可用
    }
    fs.Errorf(f, "Discarding closed SSH connection: %v", err)
    c = nil
}
```

FTP 的 `getFtpConnection` **没有**这个检查——直接取出池中第一个连接使用，依赖后续操作的错误来发现连接已断开。

```go
// ftp.go 第 568-571 行
if len(f.pool) > 0 {
    c = f.pool[0]
    f.pool = f.pool[1:]    // 直接取出，无存活检查
}
```

### 4.5 Token 操作时序差异

| 步骤 | SFTP getSftpConnection | FTP getFtpConnection |
|------|----------------------|---------------------|
| 1 | `accounting.LimitTPS(ctx)` | `f.tokens.Get()` (若 concurrency > 0) |
| 2 | `f.tokens.Get()` (若 connections > 0) | `accounting.LimitTPS(ctx)` |
| 3 | 从池中取/新建连接 | 从池中取/新建连接 |

SFTP 先限流再取令牌，FTP 先取令牌再限流。这意味着在令牌有限时，FTP 会先占用令牌再等待限流，SFTP 则先等限流再占用令牌。

### 4.6 Pacer 使用差异

| 后端 | 新建连接是否经 Pacer | 位置 |
|------|---------------------|------|
| SFTP | 是 | `getSftpConnection` 中 `f.pacer.Call(func(){...})` 第 818 行 |
| FTP | 是（但 Pacer 在 `ftpConnection` 内部） | `ftpConnection` 中 `f.pacer.Call(func(){...})` 第 543 行 |

SFTP 在 `getSftpConnection` 层面使用 Pacer 包装整个 `sftpConnection` 调用。FTP 在 `ftpConnection` 内部使用 Pacer 包装 `Dial + Login`。效果相同，但 Pacer 的作用范围不同——SFTP 对建连+初始化 SFTP 子系统整体限速，FTP 仅对 Dial+Login 限速。

---

## 五、List 中的错误路径

### 5.1 SFTP List 错误路径

List (sftp.go 第 1387-1436 行)：

```
c.sftpClient.ReadDir(sftpDir)
  ├─ errors.Is(err, os.ErrNotExist) → fs.ErrorDirNotFound
  ├─ 其他错误 → fmt.Errorf("error listing %q: %w", dir, err)
  └─ 成功 → 遍历 infos
       ├─ 符号链接 → f.stat(ctx, remote) 再解析
       │    ├─ os.IsNotExist → 忽略，用原始 info
       │    └─ 其他错误 → fs.Errorf 日志，用原始 info
       ├─ info.IsDir() → fs.NewDir
       └─ 否则 → &Object{}
```

SFTP 的 `ReadDir` 对不存在的目录直接返回错误，因此不需要"空列表二次验证"。

### 5.2 FTP List 错误路径

List (ftp.go 第 889-964 行)：

```
c.List(encodedPath)   [goroutine + timeout]
  ├─ 错误 → translateErrorDir → 550/450 变为 fs.ErrorDirNotFound
  ├─ 超时 → errors.New("timeout when waiting for List")
  └─ 成功但空列表 → f.dirExists(ctx, dir) 二次验证
       ├─ 不存在 → fs.ErrorDirNotFound
       └─ 存在 → 返回空 entries (合法空目录)
```

FTP 需要"空列表二次验证"是因为 FTP 协议对不存在的目录可能返回成功 + 空列表。这是 FTP 协议的已知缺陷。

### 5.3 遍历中的条目处理差异

| 行为 | SFTP | FTP |
|------|------|-----|
| 符号链接 | 重新 stat 解析目标，stat 失败时用原始 info 兜底 | 依赖 ftp.Entry 的 Type 字段，不额外解析 |
| `.` / `..` 条目 | sftp 库通常不返回 | 显式 `continue` 跳过 第 943 行 |
| 编码转换 | `f.opt.Enc.ToStandardName(info.Name())` 单独转换 | `f.entryToStandard(entry)` 同时转换 Name 和 Target |

---

## 六、Mkdir 与 Rmdir 的错误适配

### 6.1 Mkdir 幂等性

**SFTP** — mkdir (sftp.go 第 1469-1501 行)：

```
dirExists(dirPath)
  ├─ (true, nil) → return nil           // 已存在
  ├─ (false, fs.ErrorIsFile) → return wrapped error  // 是文件
  └─ (false, nil) → 递归创建父目录后 Mkdir
       └─ os.IsExist(err) → return nil  // 竞态：别人先建了，视为成功
```

**FTP** — mkdir (ftp.go 第 1031-1067 行)：

```
getInfo(abspath)
  ├─ err == nil && fi.IsDir → return nil     // 已存在
  ├─ err == nil && !fi.IsDir → return fs.ErrorIsFile  // 是文件
  └─ err == ErrorObjectNotFound → 递归创建父目录后 MakeDir
       └─ textproto 状态码 250/550/521 → return nil   // 幂等处理
```

| 幂等场景 | SFTP 判定方式 | FTP 判定方式 |
|---------|-------------|-------------|
| 目录已存在(预检) | `dirExists` 返回 `(true, nil)` | `getInfo` 返回 `fi.IsDir == true` |
| 目录已存在(竞态) | `os.IsExist(err)` | FTP 状态码 550/521 |
| 服务器返回 250 | 不适用（SFTP 无此场景） | `StatusRequestedFileActionOK` → nil |

### 6.2 Rmdir 差异

**SFTP** — Rmdir (sftp.go 第 1519-1538 行)：

```go
// sftp.go 第 1522-1527 行
entries, err := f.List(ctx, dir)     // 先 List 检查目录是否为空
if len(entries) != 0 {
    return fs.ErrorDirectoryNotEmpty  // ← 主动检查
}
err = c.sftpClient.RemoveDirectory(root)
```

**FTP** — Rmdir (ftp.go 第 1086-1094 行)：

```go
// ftp.go 第 1091-1093 行
err = c.RemoveDir(encodedPath)
f.putFtpConnection(&c, err)
return translateErrorDir(err)        // ← 依赖服务端返回错误
```

SFTP 主动检查目录是否为空后再删除（因为某些 SFTP 服务器会递归删除），FTP 依赖服务端返回非空错误。

---

## 七、总结：精确差异矩阵

| 边界 | 共同模式 | SFTP 特有 | FTP 特有 |
|------|---------|----------|---------|
| **对象查找** | 不存在→ErrorObjectNotFound | 目录→`fs.ErrorIsDir` | 目录→`fs.ErrorObjectNotFound` |
| **目录判断** | 存在→(true,nil)；不存在→(false,nil) | 文件→`(false, fs.ErrorIsFile)` | 文件→`(false, nil)` |
| **错误转换** | 协议错误→rclone标准错误 | 散落各方法，用 `os.IsNotExist` | 集中 `translateErrorFile/Dir`，用状态码 550/450 |
| **相同码不同映射** | 不适用 | 不适用 | 550/450 根据上下文映射 ErrorObjectNotFound 或 ErrorDirNotFound |
| **连接归还错误分类** | 有错误时选择性探测 | 协议错误跳过探测，网络错误触发探测 | **反转**：协议错误触发探测，网络错误跳过探测 |
| **取连接存活检查** | 从池中取连接 | `c.closed()` 检查 | 无检查 |
| **List 空目录** | 空列表→空entries | ReadDir 直接区分不存在 | 需 dirExists 二次验证 |
| **Mkdir 幂等** | 目录已存在视为成功 | `os.IsExist` | 状态码 250/550/521 |
| **Mkdir 碰文件** | 最终都能匹配 fs.ErrorIsFile | 包装：`fmt.Errorf("mkdir dirExists failed: %w", err)` | 裸返回：`fs.ErrorIsFile` |
| **Rmdir 空检查** | 依赖空目录约束 | 主动 List 检查 | 依赖服务端错误 |
| **findItem 吞没** | 不适用 | 不适用 | ErrorObjectNotFound→(nil,nil)；501→nil |
| **501 状态码处理** | 不适用 | 不适用 | 吞没为 (nil,nil)，NewObject 最终返回 ErrorObjectNotFound |
