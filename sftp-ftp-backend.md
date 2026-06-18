# SFTP 与 FTP 后端共同边界梳理

本文档对照 [sftp.go](file:///d:/fz/0601-2/solo-dogfeeding/code/46-rclone/backend/sftp/sftp.go) 与 [ftp.go](file:///d:/fz/0601-2/solo-dogfeeding/code/46-rclone/backend/ftp/ftp.go)，从**连接建立**、**目录遍历**、**错误适配**三个维度梳理二者的共同边界与设计模式。

---

## 一、连接建立 (Connection Establishment)

### 1.1 整体架构对比

| 维度 | SFTP 后端 | FTP 后端 |
|------|-----------|----------|
| 底层库 | `github.com/pkg/sftp` + `golang.org/x/crypto/ssh` | `github.com/jlaffaye/ftp` |
| 连接结构体 | `conn { sshClient, sftpClient }` | 直接使用 `*ftp.ServerConn` |
| 新建连接函数 | `sftpConnection(ctx)` 第 719 行 | `ftpConnection(ctx)` 第 446 行 |
| 获取连接函数 | `getSftpConnection(ctx)` 第 798 行 | `getFtpConnection(ctx)` 第 562 行 |
| 归还连接函数 | `putSftpConnection(pc, err)` 第 837 行 | `putFtpConnection(pc, err)` 第 589 行 |
| 排空连接池 | `drainPool(ctx)` 第 879 行 | `drainPool(ctx)` 第 621 行 |

### 1.2 共同边界：连接池模式

两个后端均采用**连接池 + 互斥锁**的经典模式，结构高度一致：

```
poolMu (sync.Mutex)
    ↓
pool ([]*conn / []*ftp.ServerConn)
    ↓
drain (*time.Timer)  — 空闲超时后自动排空
```

**共同行为**：
- `getXxxConnection`：先从池中取，无可用连接则新建
- `putXxxConnection`：归还到池尾，若有错误则先检查连接存活
- `drainPool`：关闭所有池中连接，清空池
- `IdleTimeout`：空闲超时后自动排空连接池（通过 `time.AfterFunc`）

### 1.3 共同边界：并发控制

两个后端均使用 `pacer.TokenDispenser` 控制并发连接数：

| 配置项 | SFTP | FTP |
|--------|------|-----|
| 选项名 | `connections` (第 388 行) | `concurrency` (第 88 行) |
| 默认值 | 0 (无限制) | 0 (无限制) |
| 取令牌 | `f.tokens.Get()` | `f.tokens.Get()` |
| 还令牌 | `f.tokens.Put()` | `f.tokens.Put()` |

### 1.4 共同边界：重试机制 (Pacer)

| 项目 | SFTP | FTP |
|------|------|-----|
| Pacer 位置 | `f.pacer` (第 628 行) | `f.pacer` (第 298 行) |
| minSleep | 100ms (第 44 行) | 10ms (第 41 行) |
| maxSleep | 2s (第 45 行) | 2s (第 42 行) |
| decayConstant | 2 | 2 |
| 创建方式 | `fs.NewPacer(ctx, pacer.NewDefault(...))` | 相同 |

**差异**：SFTP 的 minSleep 为 100ms，FTP 为 10ms，反映了两种协议的预期响应延迟不同。

### 1.5 共同边界：连接存活检查

归还连接时，若存在错误，均会执行**存活探测**以决定是否丢弃连接：

| 后端 | 探测方法 | 判断逻辑 |
|------|----------|----------|
| SFTP | `c.sftpClient.Getwd()` | 先判断是否为"常规错误"(`StatusError`/`PathError`/`os.ErrNotExist`)，非常规错误才探测 |
| FTP | `c.NoOp()` | 若存在 `textproto.Error` 则探测 |

**设计意图一致**：协议层面的业务错误（如文件不存在）不应导致连接被丢弃；只有连接层面的错误才需要丢弃重连。

### 1.6 共同边界：代理支持

两个后端均支持 SOCKS5 代理和 HTTP CONNECT 代理：

| 配置项 | SFTP (第 511-536 行) | FTP (第 190-214 行) |
|--------|-----------------------|----------------------|
| SOCKS 代理 | `socks_proxy` | `socks_proxy` |
| HTTP 代理 | `http_proxy` | `http_proxy` |
| 实现方式 | `proxy.SOCKS5Dial` / `proxy.HTTPConnectDial` | 相同 |
| 基础 Dialer | `fshttp.NewDialer(ctx)` | 相同 |

### 1.7 共同边界：初始化验证

两个后端在 `NewFs` 中都会**建立一条初始连接**并放入池中，用于尽早暴露配置错误：

- SFTP: `NewFsWithConnection` 第 1217 行 `f.getSftpConnection(ctx)`
- FTP: `NewFs` 第 707 行 `f.getFtpConnection(ctx)`

---

## 二、目录遍历 (Directory Traversal)

### 2.1 核心接口对比

| 接口 | SFTP 实现 (行号) | FTP 实现 (行号) | 共同行为 |
|------|------------------|-----------------|----------|
| `List(ctx, dir)` | 第 1387 行 | 第 889 行 | 列出目录条目，区分文件与目录 |
| `NewObject(ctx, remote)` | 第 1342 行 | 第 846 行 | 获取单个文件对象 |
| `dirExists(ctx, dir)` | 第 1356 行 | 第 869 行 | 判断目录是否存在 |
| `Mkdir(ctx, dir)` | 第 1504 行 | 第 1077 行 | 创建目录（含父目录） |
| `Rmdir(ctx, dir)` | 第 1519 行 | 第 1086 行 | 删除空目录 |

### 2.2 共同边界：路径编码 (Encoder)

两个后端均使用 `encoder.MultiEncoder` 处理文件名编码转换：

| 阶段 | SFTP | FTP |
|------|------|-----|
| 发出请求前 | `f.opt.Enc.FromStandardPath(remote)` 第 2095 行 | `f.dirFromStandardPath(dir)` 第 779 行 |
| 收到响应后 | `f.opt.Enc.ToStandardName(info.Name())` 第 1406 行 | `f.entryToStandard(entry)` 第 769 行 |

**共同模式**：标准路径 (rclone 内部使用) ⇄ 编码路径 (远程协议使用)

### 2.3 共同边界：条目分类

遍历结果均分为 **目录** (`fs.Dir`) 和 **文件** (`Object`) 两类，通过 `fs.DirEntries` 返回：

- SFTP (第 1423-1433 行): `info.IsDir()` 判断，`fs.NewDir` 或 `&Object{}`
- FTP (第 941-961 行): `object.Type == ftp.EntryTypeFolder` 判断，跳过 `.` 和 `..`

### 2.4 共同边界：根路径为文件的处理

当 `root` 参数指向一个文件时，两个后端都将 `f.root` 调整为其父目录，并返回 `fs.ErrorIsFile`：

- SFTP: 第 1286-1311 行
- FTP: 第 718-736 行

### 2.5 差异点：空目录判断

| 后端 | 策略 | 原因 |
|------|------|------|
| SFTP | 直接通过 `ReadDir` 返回空切片即可判断 | SFTP 协议对不存在的目录会返回错误 |
| FTP | 空列表时需额外调用 `dirExists` 验证 (第 928-936 行) | FTP 协议对不存在目录可能返回空列表和成功码 |

### 2.6 差异点：符号链接处理

| 后端 | 处理方式 |
|------|----------|
| SFTP | 非普通文件且非目录时，重新 `stat` 解析目标 (第 1409-1422 行)，支持 `skip_links` 选项跳过 |
| FTP | 依赖 FTP 库对 `EntryTypeLink` 的处理，通过 `entry.Target` 记录目标 (第 775 行) |

---

## 三、错误适配 (Error Adaptation)

### 3.1 错误转换总览

两个后端都将**协议特定错误**转换为 **rclone 标准错误**，形成统一的错误抽象层。

| rclone 标准错误 | SFTP 触发条件 | FTP 触发条件 |
|-----------------|---------------|--------------|
| `fs.ErrorObjectNotFound` | `os.IsNotExist(err)` (Object.stat 第 2213 行) | `StatusFileUnavailable`(550) / `StatusFileActionIgnored`(450) (translateErrorFile 第 747 行) |
| `fs.ErrorDirNotFound` | `errors.Is(err, os.ErrNotExist)` (List 第 1400 行) | 同上 (translateErrorDir 第 758 行) |
| `fs.ErrorIsFile` | `info.IsDir() == false` (NewObject 第 2218 行) | `entry.Type != EntryTypeFolder` (NewObject 第 852 行) |
| `fs.ErrorIsDir` | 目录 Stat 返回 `info.IsDir() == true` (NewObject 第 2218 行) | 间接通过 `findItem` 返回 `nil` + `ErrorObjectNotFound` |

### 3.2 共同边界：错误分层

两个后端都采用三层错误模型：

```
┌─────────────────────────────┐
│   rclone 标准错误 (fs.*)     │  ← 对外暴露
├─────────────────────────────┤
│   后端适配层 (转换函数)      │  ← 适配逻辑
├─────────────────────────────┤
│   协议库原始错误             │  ← 底层实现
└─────────────────────────────┘
```

- SFTP 适配层：散落在各方法中（如 `Object.stat` 的 `os.IsNotExist → fs.ErrorObjectNotFound`）
- FTP 适配层：集中在 `translateErrorFile()`、`translateErrorDir()` 等函数中

### 3.3 共同边界：连接归还时的错误分类

两个后端在 `putXxxConnection` 中都区分**业务错误**与**连接错误**：

**SFTP (第 848-868 行)** 的"常规错误"判定：
```go
var statusErr *sftp.StatusError
var pathErr *os.PathError
switch {
case errors.Is(err, os.ErrNotExist):
    isRegularError = true
case errors.As(err, &statusErr):
    isRegularError = true
case errors.As(err, &pathErr):
    isRegularError = true
}
```

**FTP (第 601-610 行)** 的"协议错误"判定：
```go
if tpErr := textprotoError(err); tpErr != nil {
    nopErr := c.NoOp()
    // ...
}
```

**共同逻辑**：只有非业务类错误才触发存活探测，业务错误不影响连接复用。

### 3.4 共同边界：重试判定

| 后端 | 重试入口 | 重试判断 |
|------|----------|----------|
| SFTP | `f.pacer.Call(...)` 第 818 行 | 依赖 `pacer` 默认策略 + `fserrors.ShouldRetry` |
| FTP | `shouldRetry(ctx, err)` 第 400 行 | `isRetriableFtpError` (421/426 状态码) + `fserrors.ShouldRetry` |

**共同依赖**：都依赖 `fserrors.ShouldRetry()` (第 404 行 `fserrors/error.go`) 处理通用网络错误（如连接断开、超时等）。

### 3.5 共同边界：Mkdir 幂等性处理

创建目录时，"目录已存在"均被视为成功：

| 后端 | 处理方式 |
|------|----------|
| SFTP | `os.IsExist(err)` → 调试日志 + 返回 nil (第 1494 行) |
| FTP | 状态码 250/550/521 → 置为 nil (第 1056-1064 行) |

### 3.6 差异点：错误处理集中度

| 后端 | 风格 | 特点 |
|------|------|------|
| SFTP | 分散式 | 错误转换散落在各操作函数中，没有统一的转换函数 |
| FTP | 集中式 | 有 `translateErrorFile`、`translateErrorDir`、`textprotoError` 等专门函数 |

### 3.7 FTP 特有：超时保护

FTP 后端的 `List` 和 `Close` 操作有独立的超时保护（goroutine + timer）：

- `List` 超时: `f.ci.TimeoutOrInfinite()` (第 912 行)
- `Close` 超时: `f.opt.CloseTimeout` (第 1279 行)

这是因为 FTP 采用数据连接/控制连接分离的模式，某些操作可能长时间阻塞。

---

## 四、总结：共同边界矩阵

| 模块 | 共同设计模式 | 关键差异点 |
|------|-------------|-----------|
| **连接池** | pool + poolMu + drain + IdleTimeout | SFTP 封装 conn 结构体，FTP 直接用 ServerConn |
| **并发控制** | TokenDispenser + connections/concurrency 选项 | 选项命名不同，机制相同 |
| **重试机制** | fs.Pacer + pacer.NewDefault + fserrors.ShouldRetry | minSleep 不同 (100ms vs 10ms) |
| **存活检查** | 归还时遇错探测，常规错误不复用检查 | 探测方法不同 (Getwd vs NoOp)，判断条件不同 |
| **代理支持** | SOCKS5 + HTTP CONNECT + fshttp dialer | 完全一致 |
| **路径编码** | encoder.MultiEncoder 双向转换 | 完全一致 |
| **目录列表** | 区分文件/目录 + DirEntries 返回 | 空目录判断策略不同 |
| **错误转换** | 协议错误 → rclone 标准错误 | SFTP 分散式，FTP 集中式 |
| **Mkdir 幂等** | 目录已存在视为成功 | 判断条件不同 (os.IsExist vs 状态码) |
| **初始化验证** | NewFs 时建立初始连接验证 | 完全一致 |

---

## 五、可抽象的公共组件

基于上述共同边界，理论上可抽取以下公共组件供两类后端复用：

1. **连接池泛型**：`ConnectionPool[T]` 封装 pool/poolMu/drain/tokens 逻辑
2. **错误适配器基类**：提供 `isRegularError`、`shouldRetry` 等模板方法
3. **路径编码工具**：标准路径与编码路径双向转换的辅助函数
4. **目录遍历模板**：`List` 方法的骨架实现 (编码 → 调用 → 解码 → 分类)

目前两个后端各自独立实现了这些逻辑，存在一定的代码重复。
