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
| `Retrier` (`.Retry() bool`) | 标记为「需要重试」 | 触发高层级重试 | [error.go:26-29](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/fs/fserrors/error.go#L26-L29) |
| `Fataler` (`.Fatal() bool`) | 标记为「致命错误」 | **立即终止**所有层级重试 | [error.go:96-99](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/fs/fserrors/error.go#L96-L99) |
| `NoRetrier` (`.NoRetry() bool`) | 标记为「不重试」 | **禁止**命令级 (L1) 重试 | [error.go:149-152](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/fs/fserrors/error.go#L149-L152) |
| `NoLowLevelRetrier` (`.NoLowLevelRetry() bool`) | 标记为「不做低级重试」 | **禁止** L3~L5 的低级别重试 | [error.go:196-199](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/fs/fserrors/error.go#L196-L199) |
| `RetryAfter` (`.RetryAfter() time.Time`) | 携带「服务器建议延迟」 | 按指定时间 sleep 后重试 | [error.go:245-248](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/fs/fserrors/error.go#L245-L248) |

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

这是重试机制中最「聪明」的函数，位于 [error.go:404-434](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/fs/fserrors/error.go#L404-L434)。它通过**四步递进式判定**识别「可重试的网络瞬态错误」：

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

在 `fs/sync/sync.go` 的 `currentError()` 方法（[sync.go:356-366](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/fs/sync/sync.go#L356-L366)）中，明确了错误优先级：

```
FatalError  >  普通错误(err)  >  NoRetryError
   ↓              ↓                ↓
 立刻终止      可触发重试       仅命令执行完后不重试
```

---

## 三、退避算法：`lib/pacer/pacers.go`

退避算法决定「**失败后等多久再试**」。Rclone 使用 **策略模式（Strategy Pattern）**，通过 `Calculator` 接口支持多种退避算法。

### 3.1 Calculator 接口与 State 状态

核心接口定义在 [pacer.go:22-27](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/lib/pacer/pacer.go#L22-L27)：

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

这是默认算法，位于 [pacers.go:30-102](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/lib/pacer/pacers.go#L30-L102)。

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
    // 分支1: 如果错误携带 RetryAfter 指示 → 直接用服务器指定的时间
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

专门适配 Google Drive API，符合 Google 官方推荐的退避策略，位于 [pacers.go:149-210](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/lib/pacer/pacers.go#L149-L210)。

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

适配 S3 的高稳定性特性，位于 [pacers.go:220-294](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/lib/pacer/pacers.go#L220-L294)。

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
    retries    int           // 最大重试次数（默认 3）
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

核心循环位于 [pacer.go:220-235](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/lib/pacer/pacer.go#L220-L235)：

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

### 4.3 `fs.Pacer`：带日志的装饰器

[fs/pacer.go](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/fs/pacer.go) 中对底层 `lib/pacer.Pacer` 做了两层装饰：

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

## 五、L4 流读取重试：`operations/reopen.go`

网络传输中最脆弱的环节是**大文件流式读取**——读到一半连接断开是家常便饭。`ReOpen` 结构体专门解决此问题。

### 5.1 核心思想：断点续读

```go
type ReOpen struct {
    offset      int64           // 当前已读取到的偏移量（相对 start）
    rangeOption fs.RangeOption  // HTTP Range 请求选项
    tries       int             // 已尝试次数
    maxTries    int             // 最大重试次数（= LowLevelRetries）
}
```

### 5.2 Read() 中的重试逻辑

[reopen.go:186-234](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/fs/operations/reopen.go#L186-L234)：

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

## 六、L3 操作级重试：`operations/operations.go`

### 6.1 通用 `Retry()` 函数

[operations.go:739-762](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/fs/operations/operations.go#L739-L762)：

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
        // 判定 2: pacer.RetryAfter → 按服务器指示 sleep
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

[copy.go:307-352](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/fs/operations/copy.go#L307-L352) 是更复杂的版本，核心差异：

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

最外层的全局重试入口是 `Run()` 函数，位于 [cmd.go:240-293](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/cmd/cmd.go#L240-L293)。

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

以 S3 后端的 `listBuckets` 为例（[s3.go:2626-2634](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/backend/s3/s3.go#L2626-L2634)），看完整的调用链：

```go
func (f *Fs) listBuckets(ctx context.Context) (entries fs.DirEntries, err error) {
    req := s3.ListBucketsInput{}
    var resp *s3.ListBucketsV2Output

    // 步骤③: 通过 fs.Pacer 发起调用（L5 Pacer 在此生效）
    err = f.pacer.Call(func() (bool, error) {
        resp, err = f.c.ListBuckets(ctx, &req)       // 步骤④: 实际 SDK 调用
        return f.shouldRetry(ctx, err)                // 步骤⑤: 后端自定义判定
    })
    // ...
}
```

`shouldRetry()` 的判定链（[s3.go:1280-1311](file:///d:/fz/0601-2/solo-dogfeeding/code/101-rclone/backend/s3/s3.go#L1280-L1311)）：

```
f.shouldRetry(ctx, err)
  │
  ├─ 是 smithy.APIError (AWS SDK 错误)？
  │   ├─ ① fserrors.ShouldRetry(awsError) → 通用瞬态网络错误？
  │   ├─ ② ErrorCode == "RequestTimeout" → S3 专属超时码？
  │   ├─ ③ ErrorCode == "SlowDown" / "InternalError" → 包装为 RetryAfterError（带退避）
  │   └─ ④ HTTP 状态码在 retryErrorCodes [500, 502, 503] → 重试
  │
  └─ 非 AWS 错误 → fserrors.ShouldRetry(err) → 通用判定
```

**完整执行路径**（一次 ListBuckets 的生命周期）：

```
用户执行: rclone lsd s3:
  ↓
cmd.Run() [L1, ci.Retries=3]
  ↓
sync / list 操作
  ↓
operations.ListFn()
  ↓
s3.listBuckets()
  ↓
fs.Pacer.Call() [L5, retries=2]
  │
  ├─ pacer.beginCall(): 取 pacer + connTokens 令牌
  ├─ pacerInvoker(): 进入
  │   ↓
  │   s3.shouldRetry(): 判定 (返回 retry=true, err)
  │   ↓
  │   pacerInvoker(): Debugf 日志 + 包装为 fserrors.RetryError(err)
  │   ↓
  ├─ pacer.endCall(retry=true):
  │   ├─ ConsecutiveRetries++
  │   ├─ S3.Calculator.Calculate(): 计算新 SleepTime（翻倍增长）
  │   └─ 归还 connTokens
  │
  └─ 循环 2 次后退出 → 返回 RetryError
  ↓
operations.Retry() [L3, maxTries=10]:
  ├─ 检测到 IsRetryError(err)=true → continue
  └─ 重新调用 s3.listBuckets() → 回到 Pacer（此时 Pacer SleepTime 已升高）
  ↓
... (重复直到成功或所有层耗尽)
  ↓
cmd.Run() 最终:
  ├─ 成功 → break
  ├─ FatalError → "Fatal error received - not attempting retries"
  └─ 所有尝试耗尽 → 退出码 RetryError
```

---

## 九、关键设计模式总结

### 9.1 错误语义叠加（Semantic Wrapping）
不修改错误本体，通过 Decorator 包装器叠加 `Retrier`/`Fataler`/`NoRetrier` 等语义标记。好处：原始错误信息永远不丢失，任意层都可以通过 `errors.Unwrap()` 查根因。

### 9.2 分层重试（Layered Retry）
每层职责明确、独立计数：
- L5 Pacer：单 API 调用 × 3 次（带智能退避）
- L4 ReOpen：流读取 × LowLevelRetries 次（断点续传）
- L3 Operations：单操作 × LowLevelRetries 次（兜底）
- L1 Cmd：全命令 × `--retries` 次（面向统计）

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
| `--low-level-retries` | 10 | L3/L4 | 操作级和流读取级重试次数 |
| `--max-connections` | 0 | L5 Pacer | 最大并发连接数（0=不限） |
| `--tpslimit` | 0 | Pacer Config | 每秒事务数上限（0=不限） |
| `--tpslimit-burst` | 1 | Pacer Config | TPS 限制的突发容量 |
| Pacer Calculator | per-backend | L5 Pacer | 退避算法（S3/GoogleDrive/Default） |
| `minSleep` | 10ms | Calculator | 退避下限 |
| `maxSleep` | 2s | Calculator | 退避上限 |
