# Rclone 命令执行链路分析

## 一、整体架构概览

Rclone 使用 `cobra` 作为命令行框架，采用 **插件式注册 + 集中分发** 的架构模式。整个链路分为三个核心阶段：

1. **命令注册阶段** - 程序启动初始化阶段（main 执行前）利用 Go 包初始化机制完成命令树和标志的注册
2. **参数解析阶段** - 运行期由 cobra 解析命令行参数和环境变量
3. **运行分发阶段** - 根据解析结果分发到具体命令的执行函数

```
【程序启动初始化 - main() 执行前】
├─ 依赖包级变量初始化 (flags 包: All=nil)
├─ 依赖包 init() 执行 (flags 包: 创建14个标志分组)
├─ cmd 包级变量初始化
│  ├─ help.go: Root 等帮助类 Command 实例创建
│  └─ cmd.go: cpuProfile/statsInterval 等全局标志注册
├─ 子命令包级变量初始化 (commandDefinition 实例创建)
├─ 子命令包 init() 执行 (cmd.Root.AddCommand + 命令级标志注册)
└─ backend 包 init() 执行 (后端注册到 fs.Registry)

【运行期 - main() 执行后】
        ↓
启动入口 (rclone.go:main)
        ↓
cmd.Main() [cmd/cmd.go]
        ↓
setupRootCommand() [cmd/help.go] → 添加全局标志/帮助子命令
        ↓
AddBackendFlags() [cmd/cmd.go] → 注册所有后端专属标志
        ↓
Root.Execute() [cobra 框架入口]
        ↓
cobra.OnInitialize → initConfig() [cmd/cmd.go] → 配置初始化
        ↓
子命令 Run 函数 (如 cmd/copy/copy.go)
        ↓
cmd.Run() 包装执行 [cmd/cmd.go] → 重试/统计/清理/退出码
        ↓
实际业务逻辑 (如 sync.CopyDir / operations.CopyFile)
```

---

## 二、命令注册机制（初始化阶段详细时序）

### 2.1 Go 包初始化规则

Go 程序在执行 `main()` 函数前，会严格按照以下规则完成所有导入包的初始化：

1. **依赖优先**：先初始化所有被导入的依赖包（递归）
2. **包内顺序**：每个包内部
   - 第一步：按源文件编译顺序，依次初始化**包级变量**（var 声明带赋值表达式的立即求值执行）
   - 第二步：按源文件编译顺序，依次执行所有 `init()` 函数

### 2.2 入口与依赖链

**rclone.go** 是程序启动的唯一入口，通过空白导入建立依赖链：

```go
package main

import (
    _ "github.com/rclone/rclone/backend/all"  // 导入所有后端
    "github.com/rclone/rclone/cmd"            // 导入 cmd 核心包
    _ "github.com/rclone/rclone/cmd/all"      // 导入所有命令包 ← 关键
    _ "github.com/rclone/rclone/lib/plugin"
)

func main() {
    cmd.Main()
}
```

### 2.3 初始化阶段时序（按实际执行顺序）

#### 步骤 1：`fs/config/flags` 包初始化（cmd 包的依赖）

由于 cmd 包 import 了 `fs/config/flags`，该包先完成初始化：

**包级变量初始化**（fs/config/flags/flags.go:108）：
```go
var All *Groups  // 初始化为 nil
```

**init() 函数执行**（fs/config/flags/flags.go:111-127）：
```go
func init() {
    All = NewGroups()                          // 创建分组容器
    All.NewGroup("Copy", "...")                // 14 个标志分组
    All.NewGroup("Sync", "...")
    All.NewGroup("Important", "...")
    // ... 共 14 组
}
```
> **关键点**：标志分组的创建是在 `init()` 函数中，而非包级变量初始化。

#### 步骤 2：`cmd` 包级变量初始化

在所有依赖包（包括 flags 包）初始化完成后，开始 cmd 包自身的包级变量初始化。

**help.go 文件**中的包级变量（按声明顺序）：
```go
// cmd/help.go:26-40
var Root = &cobra.Command{                    // ← 创建根命令实例
    Use:   "rclone",
    Short: "Show help for rclone commands...",
    // ...
}

var GeneratingDocs = false

var helpCommand = &cobra.Command{Use: "help", ...}  // help 子命令
var helpFlags = &cobra.Command{Use: "flags", ...}   // flags 子命令
var helpBackends = &cobra.Command{Use: "backends", ...}
var helpBackend = &cobra.Command{Use: "backend", ...}

// cmd/help.go:58-62
var (
    filterFlagsGroup     string
    filterFlagsRe        *regexp.Regexp
    filterFlagsNamesOnly bool
)
```

**cmd.go 文件**中的包级变量（按声明顺序）：
```go
// cmd/cmd.go:45-55
var (
    // ↓↓↓ 注意：这些赋值表达式会立即执行 ↓↓↓
    cpuProfile    = flags.StringP("cpuprofile", "", "", "...", "Debugging")
    memProfile    = flags.StringP("memprofile", "", "", "...", "Debugging")
    statsInterval = flags.DurationP("stats", "", time.Minute*1, "...", "Logging")
    version       bool

    errorCommandNotFound    = errors.New("command not found")
    errorNotEnoughArguments = errors.New("not enough arguments")
    errorTooManyArguments   = errors.New("too many arguments")
)

// cmd/cmd.go:519
var backendFlags map[string]struct{}  // 初始化为 nil
```

> **关键点**：`cpuProfile = flags.StringP(...)` 的赋值右侧是函数调用，**在包级变量初始化阶段立即执行**。`flags.StringP()` 内部会调用 `pflag.StringP()` 注册标志到 `pflag.CommandLine`，再调用 `installFlag()` 将标志加入 `flags.All` 的分组中。由于 flags 包已先初始化完成，此时 `flags.All` 及所有分组均已就绪。

#### 步骤 3：子命令包级变量初始化

以 `cmd/copy` 包为例（被 `cmd/all/all.go` 导入）：

```go
// cmd/copy/copy.go:16-20
var (
    createEmptySrcDirs = false
    loggerOpt          = operations.LoggerOpt{}
    loggerFlagsOpt     = operationsflags.AddLoggerFlagsOptions{}
)

// cmd/copy/copy.go:30
var commandDefinition = &cobra.Command{   // ← 创建命令实例
    Use:   "copy source:path dest:path",
    Short: `Copy files from source to dest...`,
    // ...
    Run: func(command *cobra.Command, args []string) { ... },
}
```

#### 步骤 4：子命令包 init() 执行

继续以 `cmd/copy` 包为例：

```go
// cmd/copy/copy.go:22-28
func init() {
    cmd.Root.AddCommand(commandDefinition)  // ← 注册到根命令
    cmdFlags := commandDefinition.Flags()
    // 注册命令级标志
    flags.BoolVarP(cmdFlags, &createEmptySrcDirs,
        "create-empty-src-dirs", "", createEmptySrcDirs,
        "Create empty source dirs on destination after copy", "")
    operationsflags.AddLoggerFlags(cmdFlags, &loggerOpt, &loggerFlagsOpt)
    loggerOpt.LoggerFn = operations.NewDefaultLoggerFn(&loggerOpt)
}
```

> **关键点**：`cmd.Root.AddCommand()` 在 `init()` 中调用，而非包级变量初始化。这样保证 `Root` 变量已经存在（cmd 包级变量先初始化），同时 commandDefinition 实例也已就绪。

#### 步骤 5：后端包 init() 执行

同理，各 backend 包在 `init()` 中将自身注册到 `fs.Registry`，供后续 `AddBackendFlags()` 使用。

### 2.4 注册流程时序汇总表

| 步骤 | 时机 | 包/文件 | 操作类型 | 具体动作 |
|------|------|---------|----------|----------|
| 1 | main 前 | fs/config/flags/flags.go | 包级变量 | `var All *Groups` = nil |
| 2 | main 前 | fs/config/flags/flags.go | init() 函数 | 创建 14 个标志分组 |
| 3 | main 前 | cmd/help.go | 包级变量 | 创建 `Root`/`helpCommand` 等 Command 实例 |
| 4 | main 前 | cmd/cmd.go | 包级变量 | 调用 `flags.StringP()`/`DurationP()` 注册全局标志（cpuprofile/stats 等）到 pflag + 分组 |
| 5 | main 前 | cmd/copy/copy.go 等 | 包级变量 | 创建子命令的 `commandDefinition` 实例及变量 |
| 6 | main 前 | cmd/copy/copy.go 等 | init() 函数 | `cmd.Root.AddCommand()` 注册子命令 + 注册命令级标志 |
| 7 | main 前 | backend/*/ 包 | init() 函数 | 注册后端到 `fs.Registry` |
| 8 | 运行期 | cmd/cmd.go → help.go | main 中调用 | `setupRootCommand()` 追加全局标志配置、help 子命令关联 |
| 9 | 运行期 | cmd/cmd.go | main 中调用 | `AddBackendFlags()` 遍历 `fs.Registry` 注册所有后端标志 |

---

## 三、参数解析机制

### 3.1 标志（Flag）系统分层

| 层级 | 职责 | 注册时机 | 注册位置 |
|------|------|----------|----------|
| 全局标志 | 所有命令共享（--verbose、--transfers 等） | 包级变量初始化 / setupRootCommand | cmd/cmd.go 包级变量 + configflags.AddFlags() |
| 后端标志 | 各存储后端专属（--s3-region 等） | main 运行期 | cmd/cmd.go: AddBackendFlags() |
| 命令级标志 | 单个命令专属（--create-empty-src-dirs 等） | 子命令包 init() | 各 cmd/*/init() 中 |

### 3.2 全局标志注册（两阶段）

**第一阶段 - 包级变量初始化**（cmd/cmd.go:45-55）：
```go
cpuProfile    = flags.StringP("cpuprofile", "", "", "Write cpu profile...", "Debugging")
memProfile    = flags.StringP("memprofile", "", "", "Write memory profile...", "Debugging")
statsInterval = flags.DurationP("stats", "", time.Minute*1, "Interval between...", "Logging")
```
> 仅注册 3 个最核心的全局标志（Debugging/Logging 组）

**第二阶段 - main 运行期 setupRootCommand()**（cmd/help.go:133-143）：
```go
func setupRootCommand(rootCmd *cobra.Command) {
    ci := fs.GetConfig(context.Background())
    configflags.AddFlags(ci, pflag.CommandLine)  // 配置类（--config/--cache-dir 等）
    filterflags.AddFlags(pflag.CommandLine)      // 过滤类（--include/--exclude 等）
    rcflags.AddFlags(pflag.CommandLine)          // RC API 类
    logflags.AddFlags(pflag.CommandLine)         // 日志类补充

    Root.Run = runRoot
    Root.Flags().BoolVarP(&version, "version", "V", false, "Print the version number")
    // ... 帮助子命令挂载、模板注册等
}
```

### 3.3 后端标志注册

**cmd/cmd.go:522-533** 的 `AddBackendFlags()` 在 main 运行期遍历所有后端：

```go
func AddBackendFlags() {
    backendFlags = map[string]struct{}{}  // ← 之前包级变量初始化为 nil，这里实际赋值
    for _, fsInfo := range fs.Registry {
        flags.AddFlagsFromOptions(pflag.CommandLine, fsInfo.Prefix, fsInfo.Options)
        for i := range fsInfo.Options {
            opt := &fsInfo.Options[i]
            name := opt.FlagName(fsInfo.Prefix)
            backendFlags[name] = struct{}{}  // 记录用于帮助文档分类
        }
    }
}
```

### 3.4 增强型 Flag 包装

**fs/config/flags/flags.go** 对 `pflag` 进行了增强，核心是 `installFlag()` 函数：

```go
func installFlag(flags *pflag.FlagSet, name string, groupsString string) {
    flag := flags.Lookup(name)

    // 1. 从环境变量读取默认值（如 RCLONE_STATS 对应 --stats）
    envKey := fs.OptionToEnv(name)
    if envValue, envFound := os.LookupEnv(envKey); envFound {
        err := flags.Set(name, envValue)
        flag.DefValue = envValue
    }

    // 2. 将标志添加到分组（用于分类帮助文档）
    if groupsString != "" && flags == pflag.CommandLine {
        for groupName := range strings.SplitSeq(groupsString, ",") {
            group := All.ByName[groupName]
            group.Add(flag)
        }
    }
}
```

**执行时机**：每当调用 `flags.StringP()`、`flags.BoolVarP()` 等包装函数时立即调用。例如：
- `cpuProfile = flags.StringP(...)` → 包级变量初始化阶段调用
- `flags.BoolVarP(cmdFlags, &createEmptySrcDirs, ...)` → 子命令包 init() 阶段调用

### 3.5 参数解析流程

```
命令行输入: rclone copy --verbose --transfers 8 source: dest:
        ↓
Root.Execute() [cobra 框架]
        ↓
├─ cobra 匹配子命令树: "copy"
├─ 解析所有标志:
│  ├─ --verbose → verbose=1
│  └─ --transfers 8 → transfers=8
└─ 触发 cobra.OnInitialize 钩子 → initConfig()

initConfig() [cmd/cmd.go:383-482]:
  ├─ fs.GlobalOptionsInit()    # 从已解析标志值写入全局 ConfigInfo
  ├─ fslog.InitLogging()       # 初始化日志系统
  ├─ configflags.SetFlags()    # 处理 -v/-q/--dump-headers 等特殊标志
  ├─ configfile.Install()      # 加载 rclone.conf 配置文件
  ├─ accounting.Start()        # 启动统计系统
  ├─ rcserver.Start()          # 启动 RC 服务（如配置了 --rc）
  └─ CPU/内存 profiling 初始化
```

---

## 四、运行分发机制

### 4.1 主入口 Main 函数

**cmd/cmd.go**:

```go
func Main() {
    setupRootCommand(Root)    // 阶段二：补充全局标志、挂载帮助子命令
    AddBackendFlags()          // 注册所有后端专属标志
    if err := Root.Execute(); err != nil {  // 进入 cobra 框架
        if strings.HasPrefix(err.Error(), "unknown command") && selfupdateEnabled {
            Root.PrintErrf("You could use '%s selfupdate'...\n", Root.CommandPath())
        }
        fs.Logf(nil, "Fatal error: %v", err)
        os.Exit(exitcode.UsageError)
    }
}
```

### 4.2 命令执行包装器 cmd.Run()

**cmd/cmd.go** 的 `Run()` 是所有业务命令执行的统一包装器，提供以下横切功能：

```go
func Run(Retry bool, showStats bool, cmd *cobra.Command, f func() error) {
    ctx := context.Background()
    ci := fs.GetConfig(ctx)

    // 1. 启动统计/进度显示 goroutine
    stopStats := StartStats()  // 或 startProgress()（--progress 时）

    // 2. 注册 SIGINFO 信号处理（Ctrl+T 打印运行状态）
    SigInfoHandler()

    // 3. 可配置的重试循环（默认最多 3 次）
    for try := 1; try <= ci.Retries; try++ {
        cmdErr = f()  // 执行实际业务函数
        cmdErr = fs.CountError(ctx, cmdErr)

        if !Retry || !accounting.GlobalStats().Errored() {
            break  // 无错误或禁用重试则跳出
        }
        if accounting.GlobalStats().HadFatalError() {
            break  // 致命错误不重试
        }
        // Retry-After 退避等待
        if retryAfter := accounting.GlobalStats().RetryAfter(); !retryAfter.IsZero() {
            time.Sleep(time.Until(retryAfter))
        }
        if ci.RetriesInterval > 0 {
            time.Sleep(time.Duration(ci.RetriesInterval))
        }
        accounting.GlobalStats().ResetErrors()
    }

    stopStats()  // 停止统计输出

    // 4. 清理缓存和后端连接
    cache.Clear()

    // 5. 统一解析退出码
    resolveExitCode(cmdErr)
}
```

### 4.3 典型命令执行流程

以 **cmd/copy/copy.go** 的 copy 命令为例：

```go
var commandDefinition = &cobra.Command{
    Use:   "copy source:path dest:path",
    Short: `Copy files from source to dest, skipping identical files.`,
    Long:  `...`,
    Annotations: map[string]string{
        "groups": "Copy,Filter,Listing,Important",  // 帮助文档分组
    },
    Run: func(command *cobra.Command, args []string) {
        // 1. 参数数量校验
        cmd.CheckArgs(2, 2, command, args)

        // 2. 创建源/目标文件系统实例
        fsrc, srcFileName, fdst := cmd.NewFsSrcFileDst(args)

        // 3. 通过 cmd.Run 包装执行实际逻辑
        cmd.Run(true, true, command, func() error {
            ctx := context.Background()

            // 配置日志记录器（--log-file 等）
            close, err := operationsflags.ConfigureLoggers(
                ctx, fdst, command, &loggerOpt, loggerFlagsOpt)
            if err != nil {
                return err
            }
            defer close()

            if loggerFlagsOpt.AnySet() {
                ctx = operations.WithSyncLogger(ctx, loggerOpt)
            }

            // 根据源是目录还是文件分发到不同实现
            if srcFileName == "" {
                return sync.CopyDir(ctx, fdst, fsrc, createEmptySrcDirs)
            }
            return operations.CopyFile(ctx, fdst, fsrc, srcFileName, srcFileName)
        })
    },
}
```

### 4.4 参数校验 CheckArgs

**cmd/cmd.go** 统一校验参数数量，不足或超出均打印用法后退出：

```go
func CheckArgs(MinArgs, MaxArgs int, cmd *cobra.Command, args []string) {
    if len(args) < MinArgs {
        _ = cmd.Usage()
        fmt.Fprintf(os.Stderr, "Command %s needs %d arguments minimum...\n", ...)
        resolveExitCode(errorNotEnoughArguments)
    } else if len(args) > MaxArgs {
        _ = cmd.Usage()
        fmt.Fprintf(os.Stderr, "Command %s needs %d arguments maximum...\n", ...)
        resolveExitCode(errorTooManyArguments)
    }
}
```

### 4.5 退出码解析

**cmd/cmd.go** 的 `resolveExitCode()` 根据错误类型映射到标准退出码：

```go
func resolveExitCode(err error) {
    atexit.Run()  // 执行所有注册的退出回调
    if err == nil {
        if ci.ErrorOnNoTransfer && accounting.GlobalStats().GetTransfers() == 0 {
            os.Exit(exitcode.NoFilesTransferred)  // 9
        }
        os.Exit(exitcode.Success)  // 0
        return
    }
    switch {
    case errors.Is(err, fs.ErrorDirNotFound):        os.Exit(exitcode.DirNotFound)       // 3
    case errors.Is(err, fs.ErrorObjectNotFound):     os.Exit(exitcode.FileNotFound)      // 4
    case errors.Is(err, accounting.ErrorMaxTransferLimitReached): os.Exit(exitcode.TransferExceeded) // 8
    case fssync.ErrorMaxDurationReached:             os.Exit(exitcode.DurationExceeded)  // 10
    case fserrors.ShouldRetry(err):                  os.Exit(exitcode.RetryError)        // 1
    case fserrors.IsFatalError(err):                 os.Exit(exitcode.FatalError)        // 7
    case errors.Is(err, errorNotEnoughArguments):    os.Exit(exitcode.UsageError)        // 2
    default:                                          os.Exit(exitcode.UncategorizedError) // 11
    }
}
```

---

## 五、完整调用链示例（按实际执行顺序）

以 `rclone copy src: dst: --verbose --transfers 16` 为例：

```
=================================================================
【阶段 A：程序启动初始化 - main() 执行前】
=================================================================

  A1. 依赖包初始化
      ├─ fs/config/flags 包
      │   ├─ 包级变量: var All *Groups = nil
      │   └─ init(): All = NewGroups() + 创建 14 个标志分组
      └─ 其他依赖包按深度优先初始化

  A2. cmd 包级变量初始化（依赖包都初始化完成后）
      ├─ cmd/help.go 源文件
      │   ├─ var Root = &cobra.Command{Use:"rclone", ...}
      │   ├─ var GeneratingDocs = false
      │   ├─ var helpCommand = &cobra.Command{Use:"help", ...}
      │   ├─ var helpFlags = &cobra.Command{Use:"flags", ...}
      │   ├─ var helpBackends / helpBackend 实例创建
      │   └─ var filterFlagsGroup/Re/NamesOnly 变量声明
      └─ cmd/cmd.go 源文件
          ├─ cpuProfile = flags.StringP("cpuprofile", ...)
          │   → 立即: pflag 注册 + installFlag() 加入 Debugging 组
          ├─ memProfile = flags.StringP("memprofile", ...)
          │   → 立即: pflag 注册 + installFlag() 加入 Debugging 组
          ├─ statsInterval = flags.DurationP("stats", ...)
          │   → 立即: pflag 注册 + installFlag() 加入 Logging 组
          ├─ version / error* 变量初始化
          └─ backendFlags = nil (map 声明)

  A3. 子命令包初始化（cmd/all 导入触发）
      ├─ cmd/copy 等包级变量初始化:
      │   ├─ createEmptySrcDirs = false
      │   └─ commandDefinition = &cobra.Command{Use:"copy", ...}
      └─ cmd/copy 包 init() 执行:
          ├─ cmd.Root.AddCommand(commandDefinition)  ← 注册到根命令
          └─ flags.BoolVarP(...) 注册 --create-empty-src-dirs

  A4. 后端包初始化（backend/all 导入触发）
      └─ 各后端包 init(): fs.Register(...) 注册到 fs.Registry

=================================================================
【阶段 B：运行期 - main() 开始执行】
=================================================================

  B1. rclone.go: main()
      └─ 调用 cmd.Main()

  B2. cmd/cmd.go: Main()
      ├─ setupRootCommand(Root)
      │   ├─ configflags.AddFlags() → 注册 --config/--verbose/--transfers 等
      │   ├─ filterflags.AddFlags() → 注册 --include/--exclude 等
      │   ├─ rcflags.AddFlags()     → 注册 RC API 相关标志
      │   ├─ logflags.AddFlags()    → 注册日志相关标志
      │   ├─ Root.Run = runRoot     → 设置根命令执行函数
      │   ├─ 注册 help/flags/backends 子命令到 Root
      │   ├─ cobra.OnInitialize(initConfig)  ← 注册解析后钩子
      │   └─ 设置 cobra 使用模板、补全函数等
      ├─ AddBackendFlags()
      │   └─ 遍历 fs.Registry，将所有后端选项注册为 pflag 标志
      └─ Root.Execute()  ← 进入 cobra 框架

  B3. cobra 框架执行 Root.Execute()
      ├─ 解析 os.Args 命令行
      │   ├─ "--verbose" → verbose=1
      │   └─ "--transfers 16" → transfers=16
      ├─ 匹配子命令树: "copy"
      └─ 触发 cobra.OnInitialize → initConfig()

  B4. cmd/cmd.go: initConfig()
      ├─ fs.GlobalOptionsInit() → transfers=16 等写入 ConfigInfo
      ├─ fslog.InitLogging()    → 日志系统初始化
      ├─ configflags.SetFlags()
      │   └─ verbose=1 → ci.LogLevel = LogLevelInfo
      ├─ configfile.Install()   → 读取 rclone.conf
      ├─ accounting.Start()     → 统计 goroutine 启动
      ├─ rcserver.Start()       → 若 --rc 则启动 RC HTTP 服务
      └─ profiling 相关设置

  B5. cmd/copy/copy.go: copy 命令 Run 函数
      ├─ cmd.CheckArgs(2, 2, ...) → 校验 "src:" "dst:" 两个参数
      ├─ cmd.NewFsSrcFileDst(args)
      │   ├─ cache.Get(ctx, "src:") → 创建源 Fs 实例
      │   └─ cache.Get(ctx, "dst:") → 创建目标 Fs 实例
      └─ cmd.Run(true, true, command, func() error { ... })

  B6. cmd/cmd.go: cmd.Run() 包装器
      ├─ startProgress() / StartStats() → 启动进度/统计输出
      ├─ SigInfoHandler() 注册信号处理
      ├─ 重试循环 (try=1)
      │   └─ 调用业务匿名函数
      │       ├─ 配置 Logger (--log-file 等)
      │       └─ sync.CopyDir(ctx, fdst, fsrc, createEmptySrcDirs)
      │           └─ 实际的目录复制逻辑 (fs/sync/sync.go)
      ├─ 停止统计输出
      ├─ cache.Clear() → 清理缓存、关闭后端连接
      └─ resolveExitCode(nil)
          └─ os.Exit(exitcode.Success) → 程序以 0 退出
```

---

## 六、设计要点总结

### 6.1 插件式注册（基于 Go 包初始化机制）
- 利用 Go 的 **包初始化顺序保证**（依赖先于依赖者、包级变量先于 init()）实现类型安全的自动注册
- 新增命令只需在 `cmd/all/all.go` 添加一行空白导入，无需修改注册中心代码
- 命令与命令之间、命令与框架之间完全解耦

### 6.2 初始化时序精确控制
- **标志分组**（flags 包 init()）先于 **全局标志注册**（cmd 包级变量），保证分组可用
- **Root 实例创建**（cmd/help.go 包级变量）先于 **子命令注册**（子命令包 init()），保证父节点存在
- **子命令实例创建**（子命令包包级变量）先于 **其 init() 注册**，保证引用对象就绪
- **后端注册**（backend 包 init()）先于 `AddBackendFlags()`（main 运行期）

### 6.3 统一横切关注点
`cmd.Run()` 包装器集中处理：
- 重试机制（配置化重试次数、间隔、Retry-After 退避）
- 统计/进度输出（定时 goroutine、--progress 终端渲染）
- 信号处理（SIGINFO/SIGUSR1 打印状态）
- 资源清理（缓存 evict、后端连接关闭）
- 错误统计与标准退出码映射

### 6.4 灵活的参数系统
- 三级参数体系（全局标志/后端标志/命令级标志），分阶段注册
- 环境变量自动映射（`RCLONE_` 前缀 + `OptionToEnv()` 转换）
- 14 个标志分组便于帮助文档分类组织
- 自定义 pflag.Value 类型支持（如 `fs.Duration` 支持 `5m`/`24h` 后缀）

### 6.5 声明式命令定义
每个命令通过 `cobra.Command` 结构体声明式定义：
- `Use` - 使用格式与位置参数描述
- `Short`/`Long` - 帮助文本（支持 Markdown 风格标记）
- `Annotations["groups"]` - 元数据（关联哪些全局标志分组）
- `Run` - 实际执行函数入口

### 6.6 关键文件索引

| 文件路径 | 职责 |
|----------|------|
| rclone.go | 程序入口，通过空白导入触发所有包的初始化链 |
| cmd/cmd.go | 核心命令框架：Main()/Run()/CheckArgs()/initConfig()/resolveExitCode() |
| cmd/help.go | 根命令 Root 定义、setupRootCommand()、帮助子命令及模板 |
| cmd/all/all.go | 所有命令包的批量导入入口（触发子命令 init()） |
| fs/config/flags/flags.go | 增强型 flag 系统：installFlag()、14 个标志分组定义 |
| fs/config/configflags/configflags.go | 全局配置类标志的注册与应用 |
| cmd/copy/copy.go | 典型命令实现示例（展示标准注册与执行模式） |
