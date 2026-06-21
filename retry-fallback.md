# Rclone 重试与回退策略深度分析

## 一、整体架构：多层级重试嵌套

Rclone 的重试机制采用**洋葱式多层嵌套**设计，从最外层命令执行到最内层单次 API 调用，共分为 **6 个层级**。每一层都有独立的重试计数、退避算法和错误判定逻辑，层层包裹，层层保护。

```
┌───────────────────────────────────────────────────────────────────┐
│ L1: cmd.Run()        命令级重试 (--retries, 默认 3 次)              │
│   └─ 处理: FatalError / NoRetryError / RetryAfter / 全局错误统计    │
│   └─ 退避: --retries-interval 固定间隔                               │
│     ┌───────────────────────────────────────────────────────────┐ │
│     │ L2: sync.processError()  同步级错误分类                     │ │
│     │   └─ 分类: fatalErr > err > noRetryErr 优先级              │ │
│     │     ┌───────────────────────────────────────────────────┐ │ │
│     │     │ L3: operations.Retry() / copy()  操作级重试        │ │ │
│     │     │   └─ LowLevelRetries 控制(默认 10 次)              │ │ │
│     │     │   └─ 判定: IsRetryError / ShouldRetry / RetryAfter │ │ │
│     │     │     ┌───────────────────────────────────────────┐ │ │ │
│     │     │     │ L4: ReOpen.Read()  流读取级重试            │ │ │ │
│     │     │     │   └─ 断点续读: RangeOption 偏移续传        │ │ │ │
│     │     │     │     ┌───────────────────────────────────┐ │ │ │ │
│     │     │     │     │ L5: Pacer.Call()  API 级重试      │ │ │ │ │
│     │     │     │     │   └─ 速率控制 + 低级别重试        │ │ │ │ │
│     │     │     │     │   └─ 退避: Calculator 动态计算    │ │ │ │ │
│     │     │     │     │     ┌───────────────────────────┐ │ │ │ │ │
│     │     │     │     │     │ L6: rest.Client.Call()    │ │ │ │ │ │
│     │     │     │     │     │   └─ 单次 HTTP 请求       │ │ │ │ │ │
│     │     │     │     │     └───────────────────────────┘ │ │ │ │ │
│     │     │     │     └───────────────────────────────────┘ │ │ │ │
│     │     │     └───────────────────────────────────────────┘ │ │ │
│     │     └───────────────────────────────────────────────────┘ │ │
│     └───────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

---

## 二、错误分类体系：`fs/fserrors/error.go`

错误分类是整个重试机制的**基石**。Rclone 通过接口组合模式，将错误标记为不同的「语义类型」，每一层根据这些语义决定是否重试。

### 2.1 核心错误接口定义

| 接口 | 含义 | 重试行为 | 代码位置 |
|------|------|----------|---------|
| `Retrier` (`.Retry() bool`) | 标记为「需要重试」 | 触发高层级重试 | `fs/fserrors/error.go` L26-L29 |
| `Fataler` (`.Fatal() bool`) | 标记为「致命错误」 | **立即终止**所有层级重试 | `fs/fserrors/error.go` L96-L99 |
| `NoRetrier` (`.NoRetry() bool`) | 标记为「不重试」 | **禁止**命令级 (L1) 重试 | `fs/fserrors/error.go` L149-L152 |
| `NoLowLevelRetrier` (`.NoLowLevelRetry() bool`) | 标记为「不做低级重试」 | **禁止** L3~L5 的低级别重试 | `fs/fserrors/error.go` L196-L199 |
| `RetryAfter` (`.RetryAfter() time.Time`) | 携带「服务器建议延迟」 | 按指定时间 sleep 后重试 | `fs/fserrors/error.go` L245-L248 |

### 2.2 包装器模式：Decorator 风格的错误标记

Rclone 不修改原始错误，而是使用**包装器（Wrapper）结构体**叠加语义标记，通过 `errors.Unwrap()` 保留原始错误链：

```go
// 将任意错误标记为「需要重试」
func RetryError(err error) error {
    return wrappedRetryError{err}   // 包装后 .Retry() = true
}

// 将任意错误标记为「致命」
func FatalError(err error) error {
    return wrappedFatalError{err}   // 包装后 .Fatal() = true
}

// 将任意错误标记为「不要重试（命令级）」
func NoRetryError(err error) error {
    return wrappedNoRetryError{err} // 包装后 .NoRetry() = true
}
```

> **设计亮点**：此模式完美支持多重标记——一个错误可以同时被 `RetryError()` 和 `CountableError()` 包装，通过 `liberrors.Walk()` 深度遍历错误链逐一检查接口。

### 2.3 智能判定函数：`ShouldRetry()`

这是重试机制中最「聪明」的函数，位于 `fs/fserrors/error.go` L404-L434。它通过**四步递进式判定**识别「可重试的网络瞬态错误」：

```
ShouldRetry(err) 判定流程：
  │
  ├─ ① 先检查 NoLowLevelRetry 标记 → 如果有，直接返回 false（禁止重试）
  │
  ├─ ② Cause() 深度遍历错误链：
  │     ├─ 检查 Timeout() 接口 → net.Error 超时 → 返回 true
  │     └─ 检查 Temporary() 接口 → 临时错误 → 返回 true
  │
  ├─ ③ 白名单错误类型匹配：
  │     ├─ io.EOF
  │     └─ io.ErrUnexpectedEOF
  │
  └─ ④ 错误字符串关键字匹配（最丑但最有效）：
        ├─ "use of closed network connection"
        ├─ "transport connection broken"
        ├─ "server closed idle connection"
        ├─ "bad record MAC" (TLS 层瞬态错误)
        └─ "tls: use of closed connection"
```

> **代码洞察**：第 ④ 步字符串匹配是「丑陋但必要」的工程妥协——Go 标准库中许多网络错误未导出类型，也未实现 `Timeout()`/`Temporary()` 接口，只能靠字符串匹配。

### 2.4 错误优先级汇总

在 `fs/sync/sync.go` 的 `currentError()` 方法（`fs/sync/sync.go` L356-L366）中，明确了错误优先级：

```
FatalError  >  普通错误(err)  >  NoRetryError
   ↓              ↓                ↓
 立刻终止      可触发重试       仅命令执行完后不重试
```

---

## 三、退避算法：`lib/pacer/pacers.go`

退避算法决定「**失败后等多久再试**」。Rclone 使用 **策略模式（Strategy Pattern）**，通过 `Calculator` 接口支持多种退避算法。

### 3.1 Calculator 接口与 State 状态

核心接口定义在 `lib/pacer/pacer.go` L22-L27：

```go
type State struct {
    SleepTime          time.Duration // 当前 sleep 时间（输入+输出）
    ConsecutiveRetries int           // 连续重试次数
    LastError          error         // 上一次错误（用于提取 RetryAfter）
}

type Calculator interface {
    Calculate(state State) time.Duration  // 输入当前状态，返回下次 sleep 时间
}
```

### 3.2 Default：截断指数攻击-衰减模型

这是默认算法，位于 `lib/pacer/pacers.go` L30-L102。

**核心参数**：
| 参数 | 默认值 | 含义 |
|------|--------|------|
| `minSleep` | 10ms | 最小睡眠时间 |
| `maxSleep` | 2s | 最大睡眠时间（截断上限） |
| `attackConstant` | 1 | 攻击常数（失败时增长速率） |
| `decayConstant` | 2 | 衰减常数（成功时恢复速率） |

**Calculate() 逻辑三分支**：

```go
func (c *Default) Calculate(state State) time.Duration {
    // 分支1: 如果错误携带 pacer.RetryAfter 指示 → 直接用服务器指定的时间
    if t, ok := IsRetryAfter(state.LastError); ok {
        return max(t, c.minSleep)
    }

    // 分支2: 连续重试中（Attack 攻击阶段）
    //   公式: newSleep = oldSleep * 2^attack / (2^attack - 1)
    //   attack=1 时: newSleep = oldSleep * 2 / 1 = 翻倍
    if state.ConsecutiveRetries > 0 {
        sleepTime = (state.SleepTime << c.attackConstant) / ((1 << c.attackConstant) - 1)
        return min(sleepTime, c.maxSleep)  // 截断在 maxSleep
    }

    // 分支3: 成功后（Decay 衰减阶段）
    //   公式: newSleep = oldSleep - oldSleep / 2^decay
    //   decay=2 时: newSleep = oldSleep * 3/4 = 减少 25%
    sleepTime = max((state.SleepTime<<c.decayConstant - state.SleepTime) >> c.decayConstant, c.minSleep)
    return sleepTime
}
```

**时间演化示例**（attack=1, decay=2, min=10ms, max=2s）：

```
请求序列: 成功×3 → 失败 → 失败 → 失败 → 成功 → 成功 → 成功
Sleep:    10ms → 10ms → 10ms → 20ms → 40ms → 80ms → 60ms → 45ms → 34ms → 10ms
          └── 稳定 ──┘  └──── 指数攻击(×2) ────┘  └── 线性衰减(-25%) ──┘
```

### 3.3 GoogleDrive：带抖动的指数退避 + 令牌桶

专门适配 Google Drive API，符合 Google 官方推荐的退避策略，位于 `lib/pacer/pacers.go` L149-L210。

**双模式设计**：
```
成功时 (ConsecutiveRetries == 0):
    → 使用 rate.Limiter 令牌桶限速
    → minSleep=10ms, burst=100 (高并发突发容忍)

失败时 (ConsecutiveRetries > 0):
    → 退避 = 2^(n-1) 秒 + 随机抖动(0~1秒)
    → n 截断在 5 (最大退避 = 16秒 + 抖动)
    → 连续重试 5 次后不再增加退避
```

> **设计亮点**：加入随机抖动（Jitter）是防止**惊群效应（Thundering Herd）**的经典手段——避免多个客户端在同一时刻同时重试导致服务器二次过载。

### 3.4 S3：零延迟成功 + 失败退避

适配 S3 的高稳定性特性，位于 `lib/pacer/pacers.go` L220-L294。

**与 Default 的关键区别**：
```
Default: 成功后最低也有 minSleep(10ms) 的基础延迟
S3:      成功后允许 sleepTime 衰减到 0（零延迟）→ 快乐路径极速
         仅在失败时才从 minSleep 开始退避上升
```

这一设计假设 S3 稳定性极高，正常情况下不需要限速，只有在出错时才逐步增加延迟。

---

## 四、Pacer 核心机制：`lib/pacer/pacer.go`

Pacer 是「**速率控制 + 低级别重试**」的统一封装，是整个重试体系中最精巧的部分。

### 4.1 双令牌桶结构

```go
type Pacer struct {
    pacer      chan struct{} // 速率控制令牌（容量 1）
    connTokens chan struct{} // 并发连接数令牌（容量 = maxConnections）
    state      State         // 退避状态机
    retries    int           // 最大重试次数（lib/pacer 默认 3，见下文说明）
    calculator Calculator    // 退避算法
}
```

**Pacer 令牌运作原理**（巧妙的"自定时"设计）：

```
初始状态: pacer 通道内已放入 1 个令牌

beginCall():
    取出 pacer 令牌（阻塞直到拿到）
    ↓
    启动 goroutine: sleep(SleepTime) 后把令牌放回去
    ↓
    这就实现了"每次调用间隔 SleepTime"的效果
    ↓
    取出 connTokens（如果限制并发数）
    ↓
    执行实际 API 调用

endCall():
    归还 connTokens
    根据成功/失败调用 calculator.Calculate() 更新 SleepTime
```

> **设计亮点**：使用带缓冲的 channel + goroutine sleep 实现速率控制，相比 `time.Ticker` 方案，**不需要后台常驻 goroutine**，空闲时零开销。

### 4.2 重试主循环：`call()` 方法

核心循环位于 `lib/pacer/pacer.go` L220-L235：

```go
func (p *Pacer) call(fn Paced, retries int) (err error) {
    var retry bool
    for i := 1; i <= retries; i++ {
        p.beginCall(limitConnections)    // 取令牌 + 等待速率
        retry, err = p.invoker(i, retries, fn)  // 执行被包装函数
        p.endCall(retry, err, limitConnections) // 归还令牌 + 更新退避状态
        if !retry {
            break  // 不重试则退出循环
        }
    }
    return err
}
```

**重入检测机制**：
```go
limitConnections := false
if p.maxConnections > 0 && !caller.Present("(*Pacer).call") {
    limitConnections = true
}
```

通过 `caller.Present()` 检查调用栈中是否已存在 `Pacer.call`——如果 Pacer 被重入调用（嵌套调用），内层不再限制连接数，否则会导致**自死锁**（外层持有 connTokens，内层等待同一个）。

### 4.3 重试次数的三层配置体系

Pacer 的 `retries` 并不是单一值，而是分三层配置：

| 配置层 | 位置 | 默认值 | 说明 |
|--------|------|--------|------|
| 底层默认 | `lib/pacer/pacer.go` L80-L83 | 3 | `pacer.New()` 的默认 retries=3 |
| fs 层覆盖 | `fs/pacer.go` L23-L33 | **10** | `fs.NewPacer()` 取 `max(ci.LowLevelRetries, 1)`，而 `ci.LowLevelRetries` 默认值为 **10**（`fs/config.go` L175-L178） |
| 后端覆盖 | 各后端 `NewFs()` 中 | 各异 | 例如 S3 显式调用 `pc.SetRetries(2)`（见下文） |

### 4.4 `fs.Pacer`：带日志的装饰器

`fs/pacer.go` 中对底层 `lib/pacer.Pacer` 做了两层装饰：

**① `logCalculator`：退避日志装饰器**

```go
func (d *logCalculator) Calculate(state pacer.State) time.Duration {
    oldSleepTime := state.SleepTime
    newSleepTime := d.Calculator.Calculate(state)
    if state.ConsecutiveRetries > 0 {
        if newSleepTime != oldSleepTime {
            Debugf("pacer", "Rate limited, increasing sleep to %v", newSleepTime)
        }
    } else {
        if newSleepTime != oldSleepTime {
            Debugf("pacer", "Reducing sleep to %v", newSleepTime)
        }
    }
    return newSleepTime
}
```

**② `pacerInvoker`：重试日志 + RetryError 包装**

```go
func pacerInvoker(try, retries int, f pacer.Paced) (retry bool, err error) {
    retry, err = f()
    if retry {
        Debugf("pacer", "low level retry %d/%d (error %v)", try, retries, err)
        err = fserrors.RetryError(err)  // 包装为 RetryError → 上层可感知
    }
    return
}
```

> **关键串联点**：`pacerInvoker` 在每次低级别重试时，将错误包装为 `RetryError`——这样即便 Pacer 内部重试耗尽，**上层 L3/L2 依然能通过 `IsRetryError()` 知道这是个「本应可重试但已耗尽次数」的错误**，从而继续触发更高级别的重试。

---

## 五、L4 流读取重试：`fs/operations/reopen.go`

网络传输中最脆弱的环节是**大文件流式读取**——读到一半连接断开是家常便饭。`ReOpen` 结构体专门解决此问题。

### 5.1 核心思想：断点续读

```go
type ReOpen struct {
    offset      int64           // 当前已读取到的偏移量（相对 start）
    rangeOption fs.RangeOption  // HTTP Range 请求选项
    tries       int             // 已尝试次数
    maxTries    int             // 最大重试次数（= LowLevelRetries，默认 10）
}
```

### 5.2 Read() 中的重试逻辑

`fs/operations/reopen.go` L186-L234：

```go
func (h *ReOpen) Read(p []byte) (n int, err error) {
    startOffset := h.offset
    for n < len(p) && err == nil {
        nn, err = h.rc.Read(p[n:])   // 从底层流读
        n += nn
        h.offset += int64(nn)        // 更新已读位置

        if err != nil && err != io.EOF {
            h.err = err
            // 关键判定: 如果不是 NoLowLevelRetry → 尝试断点续读
            if !fserrors.IsNoLowLevelRetryError(err) {
                Debugf(h.src, "Reopening on read failure after offset %d bytes: retry %d/%d",
                    h.offset, h.tries, h.maxTries)
                if h.reopen() == nil {  // reopen: close + open with Range
                    err = nil            // 成功 → 清除错误，继续循环读
                }
            }
        }
    }
    return n, err
}
```

**reopen() 的魔法**：
```go
func (h *ReOpen) reopen() error {
    if h.opened { h.rc.Close() }
    // 关键: rangeOption.Start 被设为当前 offset
    // 服务器返回从该位置开始的剩余数据
    h.rangeOption.Start = h.start + h.offset
    return h.open()  // 带 Range: bytes=<offset>- 重新 Open
}
```

> **与 Pacer 的分工**：
> - Pacer 重试整个 API 调用（完整请求-响应）
> - ReOpen 重试**读取流中的某个点**，利用 HTTP Range 实现「从断开处继续」
> - 两者互补：Pacer 在 L5 保护 `Object.Open()` 本身，ReOpen 在 L4 保护 Open 后的读取过程

---

## 六、L3 操作级重试：`fs/operations/operations.go`

### 6.1 通用 `Retry()` 函数

`fs/operations/operations.go` L739-L762：

```go
func Retry(ctx context.Context, o any, maxTries int, fn func() error) (err error) {
    for tries := 1; tries <= maxTries; tries++ {
        err = fn()
        if err == nil { break }

        if fserrors.ContextError(ctx, &err) { break }  // ctx 取消/超时 → 退出

        // 判定 1: RetryError 标记 或 ShouldRetry() 智能判定
        if fserrors.IsRetryError(err) || fserrors.ShouldRetry(err) {
            Debugf(o, "Received error: %v - low level retry %d/%d", err, tries, maxTries)
            continue  // 立即重试（无退避！退避留给 Pacer 做）
        }
        // 判定 2: pacer.IsRetryAfter → 按服务器指示 sleep
        else if t, ok := pacer.IsRetryAfter(err); ok {
            Debugf(o, "Sleeping for %v (as indicated by the server)", t)
            time.Sleep(t)
            continue
        }
        break  // 其他错误 → 不重试
    }
    return err
}
```

> **关键洞察**：`operations.Retry()` **本身不做退避**——它依赖底层 Pacer 已经根据退避算法调节了请求间隔。如果此处再叠加退避，会产生「双重退避」导致过度延迟。

### 6.2 Copy 专用重试：`copy.copy()`

`fs/operations/copy.go` L307-L352 是更复杂的版本，核心差异：

```go
func (c *copy) copy(ctx context.Context) (newDst fs.Object, err error) {
    for tries := 0; retry && tries < c.maxTries; tries++ {
        actionTaken, newDst, err = c.serverSideCopy(ctx)
        // 关键: 回退策略！服务端拷贝失败 → 降级为手动拷贝
        if errors.Is(err, fs.ErrorCantCopy) {
            actionTaken, newDst, err = c.manualCopy(ctx)  // ← 回退(Fallback)
        }
        // ... 重试判定逻辑同上 ...
        if retry {
            c.tr.Reset(ctx)  // 重置传输统计（避免重复计数）
            continue
        }
    }
    // 拷贝失败后的清理：删除可能存在的部分文件
    if err != nil {
        c.removeFailedPartialCopy(ctx, c.f, c.remoteForCopy)
    }
}
```

> **回退（Fallback）设计模式**：先尝试更高效的路径（服务端拷贝，零带宽），失败后回退到通用路径（手动流式拷贝）。这是「乐观优化 + 保底兼容」的经典工程实践。

---

## 七、L1 命令级重试：`cmd/cmd.go`

最外层的全局重试入口是 `Run()` 函数，位于 `cmd/cmd.go` L240-L293。

### 7.1 重试决策流程

```
命令执行完毕
    │
    ├─ 无错误 → 成功退出
    │
    ├─ 有错误 → 进入重试判定：
    │     │
    │     ├─ HadFatalError() → true → 打印"Fatal error received" → 停止重试
    │     │
    │     ├─ HadRetryError() → false → 打印"Can't retry any of the errors" → 停止重试
    │     │   (所有错误都是 NoRetryError)
    │     │
    │     ├─ RetryAfter() 非零 → time.Sleep() 直到服务器指定时间
    │     │
    │     ├─ RetriesInterval > 0 → 固定间隔 sleep
    │     │
    │     └─ try < ci.Retries → ResetErrors() + 重新执行整个命令
```

### 7.2 与 accounting 统计系统的深度绑定

L1 重试不直接检查错误本身，而是通过 `accounting.GlobalStats()` 全局统计系统做判断：

| 统计方法 | 含义 | 对应错误标记 |
|---------|------|-------------|
| `.Errored()` | 有任何错误 | — |
| `.HadFatalError()` | 出现过致命错误 | `FatalError` |
| `.HadRetryError()` | 出现过可重试错误 | `RetryError` / `ShouldRetry=true` |
| `.RetryAfter()` | 最近一次 RetryAfter 时间 | `RetryAfter` 接口 |

> **设计原理**：命令级重试不关心单个错误——它是**面向统计结果**的全局决策。执行一次 `rclone sync` 可能传输上万文件，其中部分成功部分失败。命令级重试只需知道「整体上值不值得再跑一次」。

---

## 八、后端实战：S3 的完整串联示例

以 S3 后端的 `listBuckets` 为例，对照代码讲清每一个环节。

### 8.1 S3 Pacer 初始化：显式覆盖重试次数

在 `backend/s3/s3.go` L1845-L1850：

```go
ci := fs.GetConfig(ctx)
pc := fs.NewPacer(ctx, pacer.NewS3(pacer.MinSleep(minSleep)))
// Set pacer retries to 2 (1 try and 1 retry) because we are
// relying on SDK retry mechanism, but we allow 2 attempts to
// retry directory listings after XMLSyntaxError
pc.SetRetries(2)
```

**关键点**：
- 虽然 `fs.NewPacer()` 默认从 `ci.LowLevelRetries` 取值（默认 10），但 S3 **显式覆盖为 2**
- 原因：AWS SDK 自身已经内置了重试逻辑，rclone 外层只需额外兜底 1 次重试（应对目录列表 XML 语法错误等 SDK 未覆盖的场景）
- 实际是「1 次初调 + 1 次重试」= 总共 2 次

### 8.2 S3 可重试 HTTP 状态码

在 `backend/s3/s3.go` L1265-L1271：

```go
// retryErrorCodes is a slice of error codes that we will retry
// See: https://docs.aws.amazon.com/AmazonS3/latest/API/ErrorResponses.html
var retryErrorCodes = []int{
    429, // Too Many Requests
    500, // Internal Server Error - "We encountered an internal error. Please try again."
    503, // Service Unavailable/Slow Down - "Reduce your request rate"
}
```

**可重试状态码为 `[429, 500, 503]`**：
- **429 Too Many Requests**：请求频率超限
- **500 Internal Server Error**：服务端内部错误（AWS 官方明确建议重试）
- **503 Service Unavailable**：服务不可用或 Slow Down 降速提示

> 注意：没有 502。S3 的错误处理中不包含 502 Bad Gateway。

### 8.3 S3 `listBuckets` 返回类型与调用链

代码位于 `backend/s3/s3.go` L2626-L2643：

```go
// listBuckets lists the buckets to out
// 返回值: (entries fs.DirEntries, err error)
func (f *Fs) listBuckets(ctx context.Context) (entries fs.DirEntries, err error) {
    req := s3.ListBucketsInput{}
    // ↓ 返回类型是 *s3.ListBucketsOutput（不是 V2 版本）
    var resp *s3.ListBucketsOutput

    err = f.pacer.Call(func() (bool, error) {
        resp, err = f.c.ListBuckets(ctx, &req)       // 实际 AWS SDK 调用
        return f.shouldRetry(ctx, err)                // 后端自定义重试判定
    })
    if err != nil {
        return nil, err
    }
    // 将 SDK 返回的 Buckets 转换为 rclone 的 fs.DirEntries
    for _, bucket := range resp.Buckets {
        bucketName := f.opt.Enc.ToStandardName(deref(bucket.Name))
        f.cache.MarkOK(bucketName)
        d := fs.NewDir(bucketName, deref(bucket.CreationDate))
        entries = append(entries, d)
    }
    return entries, nil
}
```

**返回类型对照**：
| 变量 | 类型 | 说明 |
|------|------|------|
| `req` | `s3.ListBucketsInput` | AWS SDK 请求输入（结构体，不含分页参数） |
| `resp` | `*s3.ListBucketsOutput` | AWS SDK 响应输出（指针，含 `Buckets []types.Bucket`） |
| `entries` | `fs.DirEntries` | rclone 抽象层的目录条目列表（最终返回给上层） |

> 注意：S3 ListBuckets API **没有 V2 版本**，返回类型是 `s3.ListBucketsOutput`。只有对象列表有 V1/V2 之分（`ListObjectsV2Output`）。

### 8.4 S3 `shouldRetry()` 的真实判定路径

代码位于 `backend/s3/s3.go` L1276-L1312：

```go
// 返回值: (shouldRetry bool, err error)
// 注意: S3 后端不会将错误包装为 pacer.RetryAfterError —— 没有服务器退避专用路径
func (f *Fs) shouldRetry(ctx context.Context, err error) (bool, error) {
    if fserrors.ContextError(ctx, &err) {
        return false, err  // context 取消/超时 → 不重试
    }

    var awsError smithy.APIError
    if errors.As(err, &awsError) {
        // ① 通用瞬态网络错误判定（Timeout/Temporary/EOF 等）
        if fserrors.ShouldRetry(awsError) {
            return true, err
        }
        // ② S3 专属错误码: RequestTimeout
        if awsError.ErrorCode() == "RequestTimeout" {
            return true, err
        }
    }

    // ③ 301 MovedPermanently → 跨区域桶 → 自动更新 region 后重试
    if httpStatusCode := getHTTPStatusCode(err); httpStatusCode > 0 {
        if f.rootBucket != "" {
            if httpStatusCode == http.StatusMovedPermanently {
                urfbErr := f.updateRegionForBucket(ctx, f.rootBucket)
                if urfbErr != nil {
                    fs.Errorf(f, "Failed to update region for bucket: %v", urfbErr)
                    return false, err
                }
                return true, err  // region 已更新 → 重试
            }
        }
        // ④ HTTP 状态码白名单: [429, 500, 503]
        if slices.Contains(retryErrorCodes, httpStatusCode) {
            return true, err
        }
    }

    // ⑤ 兜底: 通用网络错误判定（ShouldRetry）
    return fserrors.ShouldRetry(err), err
}
```

**判定流程图解**：

```
f.shouldRetry(ctx, err)  →  返回 (bool, error)
  │
  ├─ context 已取消/超时？ → 是 → 返回 (false, err)
  │
  ├─ 是 smithy.APIError (AWS SDK 错误类型)？
  │   ├─ ① fserrors.ShouldRetry(awsError) = true？ → 返回 (true, err)
  │   └─ ② ErrorCode == "RequestTimeout"？        → 返回 (true, err)
  │
  ├─ 能提取到 HTTP 状态码？
  │   ├─ ③ 状态码 301 + rootBucket 非空？
  │   │   └─ 自动 updateRegionForBucket() 成功 → 返回 (true, err)
  │   └─ ④ 状态码 ∈ [429, 500, 503]？ → 返回 (true, err)
  │
  └─ ⑤ 兜底 fserrors.ShouldRetry(err) → 返回 (判定结果, err)
```

> **重要澄清**：S3 的 `shouldRetry()` **不会将错误包装为 `pacer.RetryAfterError`**，即没有「服务器建议延迟」的特殊处理路径。S3 的退避完全由 `pacer.NewS3()` 算法根据连续重试次数自动指数增长，不依赖服务器返回的 Retry-After 头。搜索整个 S3 后端代码，不存在任何 `RetryAfterError`、`SlowDown`、`InternalError` 的特殊分支——这与 Google Drive 不同。

### 8.5 完整执行路径（一次 ListBuckets 的生命周期）

```
用户执行: rclone lsd s3:
  ↓
cmd.Run() [L1, ci.Retries=3]
  ↓
list 操作
  ↓
s3.(*Fs).List() → list.WithListP()
  ↓
s3.(*Fs).listBuckets()
  │
  │  返回类型: entries fs.DirEntries, err error
  │
  └─ f.pacer.Call() [L5, retries=2 ← S3 显式设置]
      │
      ├─ pacer.beginCall(): 取 pacer 令牌（connTokens 通常为 nil，S3 不限并发）
      ├─ pacerInvoker() 进入:
      │   ↓
      │   f.c.ListBuckets(ctx, &s3.ListBucketsInput{})
      │     → AWS SDK 内部已含自身重试机制
      │     → 返回 (*s3.ListBucketsOutput, error)
      │   ↓
      │   f.shouldRetry(ctx, err)
      │     → 判定链: Context → smithy.APIError → HTTP 429/500/503 → ShouldRetry
      │     → 返回 (retry=true/false, err)
      │   ↓
      │   若 retry=true:
      │     Debugf("low level retry %d/2 (error %v)")
      │     err = fserrors.RetryError(err)  ← 包装语义标记，供上层判定
      │   ↓
      ├─ pacer.endCall(retry, err):
      │   ├─ retry=true: ConsecutiveRetries++
      │   ├─ retry=false: ConsecutiveRetries = 0
      │   ├─ S3.Calculator.Calculate(state):
      │   │   ├─ retry=true → sleepTime 指数上升（×2，截断在 maxSleep）
      │   │   └─ retry=false → sleepTime 衰减（可降到 0）
      │   └─ 更新 state.SleepTime
      │
      └─ 循环最多 2 次 → 返回最终 err（可能是包装后的 RetryError）
  ↓
（如果 Pacer 仍失败且被包装为 RetryError）
  ↓
L3 或 L2 层检测到 IsRetryError(err)=true → 继续更高级别重试
  ↓
... 重复直到成功或所有层耗尽 ...
  ↓
cmd.Run() 最终判定:
  ├─ 成功 → break
  ├─ HadFatalError → "Fatal error received - not attempting retries"
  ├─ HadRetryError=false → "Can't retry any of the errors - not attempting retries"
  └─ 所有尝试耗尽 → 退出码 RetryError
```

---

## 九、关键设计模式总结

### 9.1 错误语义叠加（Semantic Wrapping）
不修改错误本体，通过 Decorator 包装器叠加 `Retrier`/`Fataler`/`NoRetrier` 等语义标记。好处：原始错误信息永远不丢失，任意层都可以通过 `errors.Unwrap()` 查根因。

### 9.2 分层重试（Layered Retry）
每层职责明确、独立计数：
- L5 Pacer：单 API 调用 × N 次（带智能退避，默认 fs 层为 10 次、S3 特化为 2 次、lib/pacer 底层默认 3 次）
- L4 ReOpen：流读取 × LowLevelRetries 次（断点续传，默认 10 次）
- L3 Operations：单操作 × LowLevelRetries 次（兜底，默认 10 次）
- L1 Cmd：全命令 × `--retries` 次（面向统计，默认 3 次）

避免了「一个超大重试次数循环」的反模式。

### 9.3 策略模式退避（Strategy Pattern）
`Calculator` 接口支持 Default/GoogleDrive/S3/AzureIMDS/ZeroDelay 等多种算法，后端可自由选择最适合云服务商特性的退避策略。

### 9.4 乐观回退（Optimistic Fallback）
`copy.serverSideCopy()` → 失败 → `manualCopy()`：优先尝试高效路径，失败后回退到保底通用路径。

### 9.5 面向统计的全局决策
L1 命令级重试通过 `accounting.GlobalStats()` 做决策，而非检查单个错误——这是大规模批处理系统的典型设计（**结果聚合 → 决策**）。

---

## 十、配置参数速查表

| 参数 | 默认值 | 作用层级 | 说明 |
|------|--------|---------|------|
| `--retries` | 3 | L1 Cmd | 命令级最大重试次数 |
| `--retries-interval` | 0 | L1 Cmd | 每次命令重试前的固定等待时间 |
| `--low-level-retries` | 10 | L3/L4/L5 | 操作级/流读取级/Pacer fs 层默认重试次数 |
| `--max-connections` | 0 | L5 Pacer | 最大并发连接数（0=不限） |
| `--tpslimit` | 0 | Pacer Config | 每秒事务数上限（0=不限） |
| `--tpslimit-burst` | 1 | Pacer Config | TPS 限制的突发容量 |
| Pacer Calculator | per-backend | L5 Pacer | 退避算法（S3/GoogleDrive/Default/AzureIMDS） |
| lib/pacer 默认 retries | 3 | L5 Pacer | `pacer.New()` 内置默认值（通常被 fs.NewPacer 覆盖为 10） |
| S3 专属 retries | 2 | L5 Pacer | S3 后端显式覆盖（1 try + 1 retry，依赖 SDK 自身重试） |
| `minSleep` | 10ms | Calculator | 退避下限 |
| `maxSleep` | 2s | Calculator | 退避上限 |
