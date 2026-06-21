# Rclone 重试与回退策略深度分析

## 一、整体架构：多层级重试嵌套

Rclone 的重试机制采用**洋葱式多层嵌套**设计，从最外层命令执行到最内层单次 API 调用，共分为 **6 个层级**。每一层都有独立的重试计数、退避算法和错误判定逻辑，层层包裹，层层保护。

```
┌───────────────────────────────────────────────────────────────────┐
│ L1: cmd.Run()        命令级重试 (--retries, 默认 3 次, 仅当 cmd.Run 第一参数为 true 时生效) │
│   └─ 处理: FatalError / NoRetryError / RetryAfter / 全局错误统计    │
│   └─ 退避: --retries-interval 固定间隔                               │
│   └─ 注意: lsd/ls等列表命令传 false → 本层完全禁用                  │
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

`cmd.Run` 函数签名：
```go
func Run(Retry bool, showStats bool, cmd *cobra.Command, f func() error)
```

**第一个参数 `Retry` 是总开关**。调用方传入 `false` 时，整个命令级重试完全禁用（无论 `ci.Retries` 设为多少）。

```
命令执行入口（cmd.Run 被调用）
    │
    ├─ 进入 for try := 1; try <= ci.Retries; try++ 循环
    │
    ├─ 执行 f() → 实际业务逻辑
    │
    ├─ ↓ 关键判断 ↓
    │   if !Retry || !accounting.GlobalStats().Errored() {
    │       break  // Retry=false 时直接退出循环
    │   }
    │
    ├─ ↓ 以下逻辑仅在 Retry=true 且有错误时才会执行 ↓
    │
    │   ├─ HadFatalError() → true → 打印"Fatal error received" → 停止重试
    │   │
    │   ├─ HadRetryError() → false → 打印"Can't retry any of the errors" → 停止重试
    │   │   (所有错误都是 NoRetryError)
    │   │
    │   ├─ RetryAfter() 非零 → time.Sleep() 直到服务器指定时间
    │   │
    │   ├─ RetriesInterval > 0 → 固定间隔 sleep
    │   │
    │   └─ try < ci.Retries → ResetErrors() + 重新执行整个命令
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

### 8.5 完整调用链：从 `rclone lsd s3:` 入口到 `listBuckets`

以用户执行 `rclone lsd s3:` 为例，完整追踪从命令入口到 S3 桶列出的每一层调用，以及哪些重试层真正生效。

#### 8.5.1 命令入口：`cmd.Run` 的第一个参数决定一切

在 `cmd/lsd/lsd.go` L58-L68：

```go
Run: func(command *cobra.Command, args []string) {
    ci := fs.GetConfig(context.Background())
    cmd.CheckArgs(1, 1, command, args)
    if recurse {
        ci.MaxDepth = 0
    }
    fsrc := cmd.NewFsSrc(args)  // 此处会初始化 S3 Fs（含 pacer.SetRetries(2)）
    // 关键！第一个参数是 false → 不启用命令级重试
    cmd.Run(false, false, command, func() error {
        return operations.ListDir(context.Background(), fsrc, os.Stdout)
    })
},
```

`cmd.Run` 函数签名（`cmd/cmd.go` L240）：
```go
func Run(Retry bool, showStats bool, cmd *cobra.Command, f func() error)
```

**第一个参数 `Retry=false` 是核心**。在 `cmd.Run` 内部（`cmd/cmd.go` L261）：
```go
for try := 1; try <= ci.Retries; try++ {
    cmdErr = f()
    // ...
    if !Retry || !accounting.GlobalStats().Errored() {  // ← Retry=false，这里直接 true
        if try > 1 {
            fs.Errorf(nil, "Attempt %d/%d succeeded", try, ci.Retries)
        }
        break  // ← 第一次执行完就 break，不会进入重试循环
    }
    // ... 后面的重试逻辑永远不会执行
}
```

> **结论**：L1 命令级重试**完全禁用**。无论 `ci.Retries` 设为多少（默认 3），循环只会执行 1 次。

#### 8.5.2 完整调用链路追踪（共 11 层函数调用）

```
用户执行: rclone lsd s3:
  │
  ▼
L0  cmd/lsd/lsd.go Run()
  ├─ cmd.CheckArgs(1, 1, command, args)
  ├─ fsrc := cmd.NewFsSrc(args)  // 初始化 S3 Fs: pc.SetRetries(2)
  └─ cmd.Run(false, false, ...) // ← Retry=false，禁用 L1 重试
  │
  ▼
L1  cmd/cmd.go cmd.Run()
  ├─ 由于 Retry=false，for 循环只执行 1 次
  └─ 调用 f() → operations.ListDir()
  │
  ▼
L2  fs/operations/operations.go operations.ListDir()
  └─ walk.ListR(ctx, f, "", false, 1, walk.ListDirs, fn)
     // ↑ ConfigMaxDepth(ctx, false)=1，不递归
  │
  ▼
L3  fs/walk/walk.go walk.ListR()
  ├─ maxLevel=1 >= 0 → 条件成立
  └─ 走 listRwalk 分支（不使用 ListR 优化）
  │
  ▼
L4  fs/walk/walk.go walk.listRwalk()
  └─ Walk(ctx, ...)  // 传入 listType=ListDirs
  │
  ▼
L5  fs/walk/walk.go walk.Walk()
  ├─ maxLevel=1，不满足 (maxLevel<0 || maxLevel>1)
  └─ 走 walkListDirSorted 分支
  │
  ▼
L6  fs/walk/walk.go walk.walkListDirSorted()
  └─ walk(ctx, ..., list.DirSorted)  // 传入 list.DirSorted 回调
  │
  ▼
L7  fs/walk/walk.go walk.walk()  // 内部 goroutine 池
  ├─ 启动 ci.Checkers 个 worker goroutine
  ├─ in <- listJob{remote: "", depth: 0}  // depth=maxLevel-1=0
  └─ 某个 worker 取出 job，调用 listDir → list.DirSorted
  │
  ▼
L8  fs/list/list.go list.DirSorted()
  ├─ entries, err = f.List(ctx, dir)  // 调用 S3.List
  └─ accounting.Stats(ctx).Listed(...) // 更新统计
  │
  ▼
L9  backend/s3/s3.go s3.(*Fs).List()
  └─ list.WithListP(ctx, dir, f)
  │
  ▼
L10 fs/list/helpers.go list.WithListP()
  └─ list.ListP(ctx, dir, callback)  // 调用 S3.ListP
  │
  ▼
L11 backend/s3/s3.go s3.(*Fs).ListP()
  ├─ bucket, directory := f.split(dir)
  │  // dir=""，所以 bucket=""，directory=""
  ├─ bucket == "" → 调用 f.listBuckets(ctx)
  └─ 将 entries 通过 callback 逐层返回
  │
  ▼
L12 backend/s3/s3.go s3.(*Fs).listBuckets()
  └─ f.pacer.Call(...)  // ← 这里才进入唯一的重试层
```

#### 8.5.3 各重试层生效情况总览

在整个调用链中，6 层重试架构中实际生效的只有 1 层：

| 层级 | 模块 | 是否生效 | 原因 |
|------|------|---------|------|
| L1 Cmd | `cmd.Run` | ❌ 无效 | `cmd.Run(false, ...)`，Retry 参数为 false，循环只跑 1 次 |
| L2 sync | `sync.processError` | ❌ 无效 | lsd 不走 sync 命令流程 |
| L3 Operations | `operations.Retry` | ❌ 无效 | `operations.ListDir` 没有用 `operations.Retry` 包装，直接调用 `walk.ListR` |
| L4 ReOpen | `ReOpen.Read` | ❌ 无效 | 列表操作不涉及流式读取，没有 `Open()` 调用 |
| L5 Pacer | `f.pacer.Call` | ✅ 生效 | 在 `listBuckets` 中显式调用，S3 设为 2 次 |

**各层代码验证**：

| 层级 | 验证代码位置 | 结论 |
|------|-------------|------|
| L1 禁用 | `cmd/lsd/lsd.go` L65 | `cmd.Run(false, false, command, ...)` |
| L3 禁用 | `fs/operations/operations.go` L1040-L1047 | `ListDir` 函数体内直接调用 `walk.ListR`，无 `Retry` 包装 |
| L5 生效 | `backend/s3/s3.go` L2629 | `err = f.pacer.Call(func() (bool, error) { ... })` |

### 8.6 失败后实际停留点：只有 L5 Pacer 会重试

当 S3 `ListBuckets` 失败时（例如返回 503 Slow Down），实际重试路径如下：

```
s3.listBuckets()
  │
  └─ f.pacer.Call() [retries=2]
      │
      ├─ 第 1 次调用 (i=1):
      │   ├─ beginCall(): 取令牌（此时 SleepTime=minSleep=10ms）
      │   ├─ f.c.ListBuckets() → 失败，返回 503
      │   ├─ f.shouldRetry(ctx, err) → 503 ∈ [429,500,503] → return (true, err)
      │   ├─ pacerInvoker: Debugf("low level retry 1/2") + err = RetryError(err)
      │   └─ endCall(retry=true):
      │       ├─ ConsecutiveRetries = 1
      │       ├─ S3.Calculator.Calculate():
      │       │   └─ state.ConsecutiveRetries=1 → sleepTime = 10ms × 2 = 20ms
      │       └─ 归还令牌，goroutine sleep(20ms) 后放回 pacer 令牌
      │
      ├─ 第 2 次调用 (i=2):
      │   ├─ beginCall(): 等待 pacer 令牌（阻塞 20ms）
      │   ├─ f.c.ListBuckets() → 失败，返回 503
      │   ├─ f.shouldRetry(ctx, err) → return (true, err)
      │   ├─ pacerInvoker: Debugf("low level retry 2/2") + err = RetryError(err)
      │   └─ endCall(retry=true):
      │       ├─ ConsecutiveRetries = 2
      │       ├─ S3.Calculator.Calculate():
      │       │   └─ sleepTime = 20ms × 2 = 40ms
      │       └─ 归还令牌
      │
      └─ 循环结束（i 已达 retries=2）→ 返回 RetryError 包装后的 err
  │
  ▼ 错误向上冒泡，不再重试
  │
  ├─ s3.ListP() → return err
  ├─ list.WithListP() → return err
  ├─ s3.List() → return err
  ├─ list.DirSorted() → return err
  ├─ walk.walk() 中的 worker 检测到 err → closeQuit()，通过 errs chan 返回 err
  ├─ walk.Walk() → return err
  ├─ walk.listRwalk() → return err
  ├─ walk.ListR() → return err
  ├─ operations.ListDir() → return err
  └─ cmd.Run() → 由于 Retry=false，不会重试 → 直接退出，打印错误
```

**关键事实**：
1. Pacer 内部循环 2 次耗尽后，错误被 `pacerInvoker` 包装为 `RetryError`
2. 上层（L3/L2/L1）虽然检测到 `IsRetryError(err)=true`，但**因为这些层本身没有重试循环**，错误直接向上冒泡
3. 最终 cmd.Run 接收到错误，但由于 `Retry=false`，不会进入重试逻辑
4. 用户看到的就是原始错误信息，不会有 `Attempt 1/3 failed` 之类的重试日志

> **与 `rclone sync` 的对比**：`sync` 命令调用 `cmd.Run(true, true, ...)`，即 `Retry=true`，所以 L1 命令级重试会生效；同时 sync 内部的 `processError` 和 `operations.Copy` 的 `Retry` 包装也会生效。但 `lsd` 是「简单列表」命令，设计者认为不需要多重重试。

### 8.7 各命令的 `cmd.Run` 重试参数对比

不同命令传入 `cmd.Run` 的第一个参数不同，决定了是否启用命令级重试：

| 命令 | `cmd.Run` 调用 | 是否启用 L1 重试 | 设计考量 |
|------|---------------|-----------------|---------|
| `rclone sync` | `cmd.Run(true, true, ...)` | ✅ 启用 | 数据同步操作，失败重试价值高 |
| `rclone copy` | `cmd.Run(true, true, ...)` | ✅ 启用 | 数据传输操作，失败重试价值高 |
| `rclone move` | `cmd.Run(true, true, ...)` | ✅ 启用 | 数据传输操作 |
| `rclone lsd` | `cmd.Run(false, false, ...)` | ❌ 禁用 | 只读列表操作，依赖 Pacer 层足够 |
| `rclone ls` | `cmd.Run(false, false, ...)` | ❌ 禁用 | 只读列表操作 |
| `rclone lsf` | `cmd.Run(false, false, ...)` | ❌ 禁用 | 只读列表操作 |
| `rclone size` | `cmd.Run(false, false, ...)` | ❌ 禁用 | 只读统计操作 |
| `rclone touch` | `cmd.Run(true, false, ...)` | ✅ 启用 | 有状态修改操作 |

这也解释了为什么 `--retries` 参数对 `rclone lsd` 不起作用——该参数只影响 L1 循环次数，但 `Retry=false` 时循环根本不会跑第二次。

### 8.8 RetryError 标记向上传递后的真实命运

这是最容易被误解的部分：**Pacer 层产生的 RetryError 标记，在 lsd 命令中只是随错误一路返回，不会触发任何上层重试**。让我们沿着代码逐层追踪它的真实去向。

#### 8.8.1 标记起源：`pacerInvoker` 中的包装

RetryError 标记诞生于 `fs/pacer.go` 的 `pacerInvoker` 函数（`fs/pacer.go` L24-L30）：

```go
func pacerInvoker(try, retries int, f pacer.Paced) (retry bool, err error) {
    retry, err = f()
    if retry {
        Debugf("pacer", "low level retry %d/%d (error %v)", try, retries, err)
        err = fserrors.RetryError(err)  // ← 这里产生 RetryError 标记
    }
    return
}
```

当 Pacer 内部重试时（第 1 次失败、第 2 次失败...），每次失败的 err 都会被包装上 `RetryError` 标记。最终 Pacer 耗尽重试次数后，最后一次返回的 err 也是带 `RetryError` 标记的。

#### 8.8.2 逐层向上：每层都只是"原样返回，不做重试决策

从 L12 `listBuckets` 到 L2 `ListDir`，共 11 层函数调用，每一层对错误的处理都**只有一件事：return err**。

让我们按调用栈从下往上看：

| 层级 | 函数 | 对错误的处理 | 代码位置 |
|------|------|-------------|---------|
| L12 | `s3.listBuckets()` | `return nil, err` | `backend/s3/s3.go` L2634 |
| L11 | `s3.ListP()` | `return err`（通过 callback 或直接返回 | `backend/s3/s3.go` L2665 |
| L10 | `list.WithListP()` | `return err` | `fs/list/helpers.go` L20 |
| L9 | `s3.List()` | `return nil, err` | `backend/s3/s3.go` L2657 |
| L8 | `list.DirSorted()` | `return nil, err` | `fs/list/list.go` L29 |
| L7 | `walk.walk()` worker | `err = fs.CountError(ctx, err)` + 塞入 errs channel | `fs/walk/walk.go` L418 |
| L6 | `walk.walkListDirSorted()` | 通过 walk 函数返回 err | `fs/walk/walk.go` L377 |
| L5 | `walk.Walk()` | `return <-errs` | `fs/walk/walk.go` L338 |
| L4 | `walk.listRwalk()` | `return err` | `fs/walk/walk.go` L196 |
| L3 | `walk.ListR()` | `return err` | `fs/walk/walk.go` L127 |
| L2 | `operations.ListDir()` | `return err` | `fs/operations/operations.go` L1047 |

**关键观察**：
- 所有中间层（L3~L11）**没有任何一层检查 `IsRetryError(err)` 并决定重试**
- 它们要么直接 `return err`，要么通过 channel 传递
- 错误像烫手山芋一样被层层向上扔，直到最外层来决定怎么办

#### 8.8.3 唯一的"副作用"：`fs.CountError` 统计

在 L7 `walk.walk()` 中的这行代码是 RetryError 标记在冒泡过程中**唯一产生实际效果的地方：

```go
// fs/walk/walk.go L418
err = fs.CountError(ctx, err)
```

`fs.CountError` 不是一个普通函数，而是一个**函数指针**，依赖注入**：

| 阶段 | 定义 | 位置 | 行为 |
|------|------|------|------|
| 默认 | `func(ctx, err) error { return err }` | `fs/config.go` L44 | 空实现，直接返回 |
| 运行时 | 被 accounting 包覆盖 | `fs/accounting/accounting.go` L48 | 实际调用 `Stats(ctx).Error(err)` |

> 这是 Go 中罕见的「包级依赖注入」模式——fs 包定义接口（函数指针），accounting 包在 `init()` 中注入实现，避免了 fs 包直接依赖 accounting 包（防止循环依赖）。

#### 8.8.4 `accounting.Stats.Error()` 对 RetryError 的处理

在 `fs/accounting/stats.go` L763-L790：

```go
func (s *StatsInfo) Error(err error) error {
    if err == nil || fserrors.IsCounted(err) {
        return err  // 已统计过的错误不再重复统计
    }
    s.errors++        // 错误计数 +1
    s.lastError = err  // 记录最后一个错误
    err = fserrors.FsError(err)  // 包装为 FsError
    fserrors.Count(err)            // 标记为已统计

    switch {
    case fserrors.IsFatalError(err):
        s.fatalError = true           // 标记：出现过致命错误
    case fserrors.IsRetryAfterError(err):
        s.retryAfter = ...           // 记录服务器建议重试时间
        s.retryError = true           // 标记：出现过可重试错误
    case !fserrors.IsNoRetryError(err):
        s.retryError = true           // 标记：出现过可重试错误（默认都是可重试的，除非显式标记 NoRetry
    }
    return err
}
```

**RetryError 标记在这里的作用：
- `IsNoRetryError(err)` 返回 **false**（因为 RetryError 不是 NoRetryError）
- 所以会进入 `case !fserrors.IsNoRetryError(err):` 分支
- `s.retryError = true` → 全局统计中标记为「可重试错误」

> 但这只是**统计记账**，不是**重试决策**。`retryError` 标记被设置了，但它是否会导致实际重试，取决于最外层 cmd.Run 的 `Retry` 参数。

#### 8.8.5 终点：`cmd.Run` 中 `Retry=false` 时的命运

回到 `cmd/cmd.go` L254-L266：

```go
for try := 1; try <= ci.Retries; try++ {
    cmdErr = f()                              // 执行业务逻辑 → 返回带 RetryError 标记的错误
    cmdErr = fs.CountError(ctx, cmdErr)       // 再统计一次（f() 内部已统计过，IsCounted=true → 跳过）
    lastErr := accounting.GlobalStats().GetLastError()
    // ...
    if !Retry || !accounting.GlobalStats().Errored() {
        // ← Retry=false 时，这里直接为 true！
        break  // 直接退出循环
    }
    // ↓ 以下代码永远不会执行 ↓
    if accounting.GlobalStats().HadFatalError() { ... }
    if !accounting.GlobalStats().HadRetryError() { ... }
    // ...
}
```

**执行顺序图解**：

```
第 1 次循环 (try=1):
  ├─ f() 执行 → 失败，返回 RetryError
  ├─ accounting stats.retryError = true  ← 已被设置了
  ├─ accounting stats.errors = 1
  │
  ├─ if !Retry || !Errored()
  │    = !false || !true
  │    = true || false
  │    = true
  │
  └─ break  ← 直接跳出循环
     │
     └─ 后面的 HadFatalError / HadRetryError 判断
        永远不会被执行到
```

**关键结论**：
1. ✅ `retryError 统计标记**确实被设置为 true**（因为错误不是 NoRetryError）
2. ❌ 但这个标记**永远不会被用于重试决策**（因为 `Retry=false` 直接 break 了）
3. RetryError 标记在 lsd 命令中，只是一个「有状态但不用」——它存在于错误链上、存在于统计数据中，但不会触发任何实际的重试行为

#### 8.8.6 对比：`rclone sync` 中 RetryError 的不同命运

作为对比，在 `rclone sync` 命令中（`cmd.Run(true, true, ...)`），RetryError 标记会经历完全不同的旅程：

```
sync 命令 (Retry=true):
  ├─ 第 1 次执行 → 失败，返回 RetryError
  ├─ stats.retryError = true
  │
  ├─ if !Retry || !Errored()
  │    = !true || !true
  │    = false || false
  │    = false  ← 不 break，继续往下
  │
  ├─ HadFatalError? → false → 继续
  ├─ HadRetryError? → true → 继续（不进入 "Can't retry" 分支）
  │
  ├─ 打印 "Attempt 1/3 failed with ..."
  ├─ ResetErrors() → stats.retryError = false
  │
  └─ 第 2 次循环 → 重新执行整个 sync 流程
     └─ ...
```

> 在 sync 命令中，RetryError 标记是**真正参与重试决策的关键依据**——只有当 `HadRetryError()=true` 时，命令级重试才会继续；如果所有错误都是 NoRetryError，`HadRetryError()=false`，则直接放弃重试。

但在 lsd 命令中，RetryError 标记只是一个**沉默的旁观者**——它被设置了，但从未被问过。

---

## 九、关键设计模式总结

### 9.1 错误语义叠加（Semantic Wrapping）
不修改错误本体，通过 Decorator 包装器叠加 `Retrier`/`Fataler`/`NoRetrier` 等语义标记。好处：原始错误信息永远不丢失，任意层都可以通过 `errors.Unwrap()` 查根因。

### 9.2 分层重试（Layered Retry）
每层职责明确、独立计数：
- L1 Cmd：全命令 × `--retries` 次（**视命令而定**，sync/copy/move 启用，lsd/ls 等只读命令禁用）
- L5 Pacer：单 API 调用 × N 次（带智能退避，默认 fs 层为 10 次、S3 特化为 2 次、lib/pacer 底层默认 3 次）
- L4 ReOpen：流读取 × LowLevelRetries 次（断点续传，默认 10 次）
- L3 Operations：单操作 × LowLevelRetries 次（兜底，默认 10 次，部分操作如 ListDir 不启用）

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
| `--retries` | 3 | L1 Cmd | 命令级最大重试次数（仅当 cmd.Run 第一参数为 true 时生效，lsd/ls 等只读命令传 false 时无效） |
| `--retries-interval` | 0 | L1 Cmd | 每次命令重试前的固定等待时间（仅 L1 启用时生效） |
| `--low-level-retries` | 10 | L3/L4/L5 | 操作级/流读取级/Pacer fs 层默认重试次数 |
| `--max-connections` | 0 | L5 Pacer | 最大并发连接数（0=不限） |
| `--tpslimit` | 0 | Pacer Config | 每秒事务数上限（0=不限） |
| `--tpslimit-burst` | 1 | Pacer Config | TPS 限制的突发容量 |
| Pacer Calculator | per-backend | L5 Pacer | 退避算法（S3/GoogleDrive/Default/AzureIMDS） |
| lib/pacer 默认 retries | 3 | L5 Pacer | `pacer.New()` 内置默认值（通常被 fs.NewPacer 覆盖为 10） |
| S3 专属 retries | 2 | L5 Pacer | S3 后端显式覆盖（1 try + 1 retry，依赖 SDK 自身重试） |
| `minSleep` | 10ms | Calculator | 退避下限 |
| `maxSleep` | 2s | Calculator | 退避上限 |
