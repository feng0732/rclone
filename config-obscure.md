# Rclone 配置文件加密边界分析

## 概述

Rclone 配置安全体系分为三层，各层边界清晰但保护强度差异显著：

1. **文件级加密**（Layer 1）— NaCl Secretbox 对整个 `rclone.conf` 加密，保护强度最高
2. **字段级混淆**（Layer 2）— AES-256-CTR 对 `IsPassword` 字段混淆，密钥硬编码，仅防肩窥
3. **输出级脱敏**（Layer 3）— 不同输出通道对敏感字段的脱敏策略差异极大

---

## 一、Obscure 字段级混淆机制

### 1.1 核心实现

**文件**：[fs/config/obscure/obscure.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/obscure/obscure.go)

**加密算法**：AES-256-CTR，密钥硬编码在源码中：

```go
// [obscure.go:19-24](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/obscure/obscure.go#L19-L24)
var cryptKey = []byte{
    0x9c, 0x93, 0x5b, 0x48, 0x73, 0x0a, 0x55, 0x4d,
    0x6b, 0xfd, 0x7c, 0x63, 0xc8, 0x86, 0xa9, 0x2b,
    0xd3, 0x90, 0x19, 0x8e, 0xb8, 0x12, 0x8a, 0xfb,
    0xf4, 0xde, 0x16, 0x2b, 0x8b, 0x95, 0xf6, 0x38,
}
```

**加密流程** [Obscure()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/obscure/obscure.go#L50-L64)：

```
明文 → 随机16字节IV → AES-CTR加密 → [IV|密文] → Base64(RawURLEncoding)
```

**解密流程** [Reveal()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/obscure/obscure.go#L76-L90)：

```
Base64解码 → 拆分[前16字节=IV|剩余=密文] → AES-CTR XOR → 明文
```

### 1.2 安全边界

| 特性 | 状态 |
|------|------|
| 算法强度 | AES-256-CTR 本身安全 |
| 密钥管理 | **硬编码在源码中**，公开可查 |
| 完整性保护 | **无 MAC/AEAD**，密文被篡改无法检测 |
| 设计目标 | 仅防"肩窥"（eyedropping） |
| 可逆性 | 完全可逆，拿到源码即可解密 |

### 1.3 命令行工具

- **obscure 命令**：[cmd/obscure/obscure.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/obscure/obscure.go) — 从参数或 STDIN 读取明文密码，输出混淆值
- **reveal 命令**：[cmd/reveal/reveal.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/reveal/reveal.go) — 隐藏命令（`Hidden: true`），用于调试解密

---

## 二、配置文件级加密与存取机制

### 2.1 Storage 接口抽象

**文件**：[fs/config/config.go:77-109](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go#L77-L109)

```go
type Storage interface {
    GetSectionList() []string
    HasSection(section string) bool
    DeleteSection(section string)
    GetKeyList(section string) []string
    GetValue(section, key string) (value string, found bool)
    SetValue(section, key, value string)
    DeleteKey(section, key string) bool
    Load() error
    Save() error
    Serialize() (string, error)
}
```

默认实现：[configfile.Storage](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/configfile/configfile.go)（基于 INI 文件）

### 2.2 配置文件加密算法

**文件**：[fs/config/crypt.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go)

| 项目 | 说明 |
|------|------|
| 算法 | NaCl Secretbox（XSalsa20 + Poly1305 MAC） |
| Nonce | 24 字节随机数，拼接在密文前 |
| 密钥派生 | `SHA256("[" + password + "][rclone-config]")` → 32字节 |
| 文件标识 | 首行 `RCLONE_ENCRYPT_V0:` |
| 编码格式 | Base64(StdEncoding) |

**加密流程** [Encrypt()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go#L216-L251)：

```
明文 → 24字节随机nonce → secretbox.Seal() → [nonce|密文] → Base64 → 写入文件
```

**解密流程** [Decrypt()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go#L50-L179)：

```
读取首行 → 识别RCLONE_ENCRYPT_V0 → Base64解码 → 拆分[nonce|密文] → secretbox.Open() → 明文
```

### 2.3 密码获取失败的返回与重试机制

**文件**：[fs/config/crypt.go:87-178](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go#L87-L178)

#### 密码来源的执行时机：循环外 vs 循环内

`Decrypt()` 函数将密码来源严格分为两个阶段，这是最关键的实现边界：

```go
// ============================================================
// 阶段一：循环外 — 只执行一次（L87-L118）
// ============================================================
if len(configKey) == 0 {
    // A. --password-command：仅在这里执行一次
    pass, err := GetPasswordCommand(ctx)
    if err != nil {
        return nil, err                          // 命令执行失败：直接返回
    }
    if pass != "" {
        usingPasswordCommand = true               // 打上标记，循环内用此判断
        SetConfigPassword(pass)
        // 命令执行成功 → 进入解密循环
    } else {
        // B. RCLONE_CONFIG_PASS：仅在这里读取一次
        envPassword := os.Getenv("RCLONE_CONFIG_PASS")
        if envPassword != "" {
            usingEnvPassword = true              // 打上标记，循环内用此判断
            SetConfigPassword(envPassword)
        }
    }
}

// ============================================================
// 阶段二：解密循环 — 解密失败进入循环重试（L131-L177）
// ============================================================
var out []byte
for {
    // C. 临时密钥文件：每次循环都重新读取环境变量 + 文件
    if envKeyFile := os.Getenv("_RCLONE_CONFIG_KEY_FILE"); len(envKeyFile) > 0 {
        读取文件 → 删除文件 → 获得 configKey
        // 读取/删除失败：立即返回错误，不重试
    } else if len(configKey) == 0 {
        // D. 通过 usingPasswordCommand 标记判断，不重新执行命令
        if usingPasswordCommand {
            return nil, errors.New("using --password-command derived password, ...")
        }
        // E. 通过 usingEnvPassword 标记判断，不重新读取环境变量
        if usingEnvPassword {
            return nil, errors.New("using RCLONE_CONFIG_PASS env password, ...")
        }
        // F. AskPassword=false：所有来源耗尽，直接返回
        if !ci.AskPassword {
            return nil, errors.New("unable to decrypt configuration and not allowed to ask ...")
        }
        // G. 交互提示：每次循环都重新调用用户输入
        getConfigPassword("Enter configuration password:")
    }

    // 尝试解密
    out, ok = secretbox.Open(nil, box[24:], &nonce, &key)
    if ok {
        break  // 解密成功，退出循环
    }

    // 解密失败：清空 configKey，强制下一轮重新获取
    fs.Errorf(nil, "Couldn't decrypt configuration, most likely wrong password.")
    configKey = nil
}
```

#### 各密码来源的失败返回路径

| 密码来源 | 执行时机 | 失败条件 | 返回行为 | 重试行为 |
|---------|---------|---------|---------|---------|
| **`--password-command`** | **循环外** 仅一次 | 命令执行失败（L88-L91） | 立即返回 `"password command failed: ..."` | **不重试**，根本不进入循环 |
| | | 命令返回空字符串（L209-L211） | 立即返回 `"--password-command returned empty string"` | **不重试**，根本不进入循环 |
| | | 命令执行成功但解密失败（循环内 L149-L151） | 检测 `usingPasswordCommand==true`，直接返回错误 | **不重试**，通过布尔标记短路，不重新执行命令 |
| **`RCLONE_CONFIG_PASS`** | **循环外** 仅一次 | 解密失败（循环内 L152-L154） | 检测 `usingEnvPassword==true`，直接返回错误 | **不重试**，通过布尔标记短路，不重新读取环境变量 |
| **`configKey` 内存缓存** | 循环入口判断 | 解密失败（MAC 验证不通过） | 不返回，清空 `configKey` | **进入下一轮循环**，从下一优先级来源重新获取 |
| **`_RCLONE_CONFIG_KEY_FILE` 临时文件** | **循环内** 每次重读 | 文件不存在或读取失败 | 立即返回错误 | **不重试**，文件 I/O 失败即终止 |
| | | 文件读取成功但删除失败 | 立即返回错误 | **不重试** |
| | | 解密失败（密钥错误） | 不返回，清空 `configKey` | **进入下一轮循环**，从下一优先级重新获取 |
| **交互提示 AskPassword=true** | **循环内** 每次重调 | 密码错误 | 打印错误到 stderr，不返回 | **无限重试**，`getConfigPassword()` 内部也有循环直到输入合法 |
| **AskPassword=false** | 循环内兜底判断 | 以上所有来源均无有效密钥 | 立即返回提示设置环境变量 | **不重试** |

#### 关键安全设计结论

1. **`--password-command` 绝不重复执行** — 通过 `usingPasswordCommand` 布尔标记实现：循环外执行一次，循环内一旦解密失败直接 return，**绝不重新执行外部命令**。这避免了命令重复执行的侧信道泄露，也避免了恶意命令被多次触发。

2. **`RCLONE_CONFIG_PASS` 绝不重新读取** — 通过 `usingEnvPassword` 布尔标记实现：循环外读取一次，循环内一旦解密失败直接 return，**绝不重新读环境变量**。这避免了环境变量在循环中被反复暴露给其他进程的读取窗口。

3. **解密失败后强制清空 `configKey`** — `configKey = nil`，确保下一轮循环 `len(configKey) == 0` 判断成立，不会在循环中重复使用错误密钥。

4. **交互模式无限重试的合理性** — 在本地场景下，攻击者若已能访问加密配置文件，暴力破解防护意义不大；此设计面向用户输错密码的正常场景。

5. **临时密钥文件立即删除** — 无论读取成功或失败，`os.Remove(envKeyFile)` 都会被执行，尽量缩小泄露时间窗口。

### 2.4 临时密钥文件传递的安全风险与边界

**文件**：[fs/config/crypt.go:284-313](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go#L284-L313)

当 `PassConfigKeyForDaemonization = true` 时，`SetConfigPassword()` 会：

```go
// 写入流程（父进程）
tempFile, _ := os.CreateTemp("", "rclone")
tempFile.WriteString(obscure.MustObscure(string(configKey)))
tempFile.Close()
os.Setenv("_RCLONE_CONFIG_KEY_FILE", tempFile.Name())

// 读取流程（子进程 Decrypt 中）
obscuredKey, _ := os.ReadFile(envKeyFile)
os.Remove(envKeyFile)  // 读取后立即删除
configKey = []byte(obscure.MustReveal(string(obscuredKey)))
```

#### 风险分析表

| 风险点 | 说明 | 严重程度 | 建议缓解 |
|--------|------|----------|---------|
| 外层 obscure 形同虚设 | obscure 密钥公开在源码，加解密仅防肩窥 | 低 | 接受，此层设计目标非加密 |
| 临时文件在系统临时目录 | `os.CreateTemp("", "rclone")` 使用 `/tmp` 等公共目录 | 中 | 使用用户私有目录，设置 `os.CreateTemp(os.UserCacheDir(), "rclone")` |
| TOCTOU 时间窗口 | 读文件与删文件之间存在竞态 | 中 | 缩短窗口，考虑使用 `unlink` 后读 / `O_TMPFILE`（Linux） |
| 环境变量路径泄露 | 子进程继承环境变量，`/proc/PID/environ` 可被同用户读取 | 中 | 读取密钥后立即 `os.Unsetenv("_RCLONE_CONFIG_KEY_FILE")` |
| 多子进程共享 | 所有子进程共享同一密钥文件路径 | 中 | 为每个子进程生成独立文件，或使用其他 IPC 机制 |
| 读取失败清理 | 读取失败时也删除文件（正确行为） | 低 | 保持现状 |
| 核心转储暴露 | 密钥明文在内存中，coredump 可能泄露 | 中 | 进程启动时调用 `prctl(PR_SET_DUMPABLE, 0)`（Linux） |
| 命令行可见性 | obscure 后的值非明文，避免 `ps` 直接可见 | - | 已缓解 |

#### 边界结论

临时密钥文件机制的 **obscure 外层加密不提供密码学安全性**，仅提供以下实际价值：
- 防止密钥明文出现在进程列表、环境变量列表中
- 防止意外肩窥（运维人员查看文件时看到明文）
- 减少核心转储中的明文密钥暴露概率

### 2.5 配置保存重试机制的安全风险

**文件**：[fs/config/config.go:383-397](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go#L383-L397)

```go
func SaveConfig() {
    ctx := context.Background()
    ci := fs.GetConfig(ctx)
    var err error
    for range ci.LowLevelRetries + 1 {
        if err = LoadedData().Save(); err == nil {
            return
        }
        waitingTimeMs := mathrand.Intn(1000)
        time.Sleep(time.Duration(waitingTimeMs) * time.Millisecond)
    }
    fs.Errorf(nil, "Failed to save config after %d tries: %v", ci.LowLevelRetries, err)
}
```

#### 风险分析

| 特性 | 值 | 安全影响 |
|------|---|---------|
| 默认重试次数 | `LowLevelRetries=10` → 共 11 次尝试 | 重试窗口内临时文件可能残留 |
| 随机等待 | 0~1000ms 随机退避 | 非安全退避，仅解决并发写入冲突 |
| 失败处理 | 仅打印日志，不返回 error | **静默失败**，调用方无法感知配置未持久化 |
| 文件写入原子性 | 临时文件 → rename 原文件 → rename 新文件 → 删除备份 | 原子性较好，但重试期间多个临时文件可能并存 |

#### 风险缓解建议

| 风险 | 建议 |
|------|------|
| 多次重试产生大量临时文件 | 重试前清理同目录下的旧临时文件（匹配 `rclone*` 模式） |
| 静默失败 | `SaveConfig()` 应返回 error，或提供返回 error 的变体函数 |
| 固定随机种子 | `mathrand` 非加密安全随机，但此场景仅用于退避，可接受 |
| 重试期间明文配置在磁盘 | 若配置文件已加密，临时文件内容也是加密的（由 Encrypt() 决定） |

---

## 三、配置展示与导出的脱敏边界

### 3.1 八种输出方式对比

以下表格对比 Rclone 所有配置输出通道的脱敏行为：

| 输出方式 | 实现函数 | IsPassword 字段 | Sensitive 字段 | 普通字段 | 格式 |
|---------|----------|----------------|---------------|---------|------|
| `rclone config show` | [ShowRemote()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/ui.go#L374-L378) → `printRemoteOptions(redacted=false)` | `*** ENCRYPTED ***` | **原值** | 原值 | INI 文本 |
| `rclone config redacted` | [ShowRedactedRemote()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/ui.go#L380-L384) → `printRemoteOptions(redacted=true)` | `XXX` | `XXX` | 原值 | INI 文本 |
| `rclone config dump` | [Dump()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go#L718-L730) → `DumpRcBlob()` | **obscure 后的值** | **原值** | 原值 | JSON |
| RC `config/dump` | [rcDump()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/rc.go#L62-L65) → `DumpRcBlob()` | **obscure 后的值** | **原值** | 原值 | JSON |
| RC `config/get` | [rcGet()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/rc.go#L82-L89) → `DumpRcRemote()` | **obscure 后的值** | **原值** | 原值 | JSON |
| `rclone config string` | [configStringCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/config/config.go#L618-L667) | **obscure 后的值** | **原值** | 原值 | 连接字符串 |
| RC `options/get` | [rcOptionsGet()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/rc/config.go#L81-L87) | **原值**（全局配置） | **原值** | 原值 | JSON |
| Storage 直接读 `GetValue()` | Storage 接口 | **obscure 后的值** | **原值** | 原值 | 原始字符串 |

### 3.2 脱敏逻辑的核心实现

**文件**：[fs/config/ui.go:338-367](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/ui.go#L338-L367)

```go
func printRemoteOptions(name string, prefix string, sep string, redacted bool) {
    // ... 获取 fsInfo ...
    for _, key := range LoadedData().GetKeyList(name) {
        // 查找该字段的 IsPassword / Sensitive 标记
        isPassword := false
        isSensitive := false
        if fsInfo != nil {
            for _, option := range fsInfo.Options {
                if option.Name == key {
                    if option.IsPassword {
                        isPassword = true
                    } else if option.Sensitive {
                        isSensitive = true
                    }
                }
            }
        }
        value := GetValue(name, key)

        // 脱敏决策 —— 优先级：redacted模式 > isPassword > 原值
        if redacted && (isSensitive || isPassword) && value != "" {
            fmt.Printf("%s%s%sXXX\n", prefix, key, sep)
        } else if isPassword && value != "" {
            fmt.Printf("%s%s%s*** ENCRYPTED ***\n", prefix, key, sep)
        } else {
            fmt.Printf("%s%s%s%s\n", prefix, key, sep, value)
        }
    }
}
```

### 3.3 脱敏边界的关键差异

| 场景 | `IsPassword=true` | `Sensitive=true` 但 `IsPassword=false` |
|------|-------------------|---------------------------------------|
| **存储时** | 自动 obscure 加密 | **明文存储**，不做任何处理 |
| **`config show`** | 显示 `*** ENCRYPTED ***` | **显示原值**（不脱敏！） |
| **`config redacted`** | 显示 `XXX` | 显示 `XXX` |
| **`config dump` / RC** | 输出 obscure 后的值（可逆） | **输出原值** |
| **交互输入** | `ChoosePassword()` 特殊流程 | 普通文本输入 |
| **命令行帮助** | 追加 `(obscured)` 后缀 | 无特殊标记 |

### 3.4 IsPassword 自动混淆的智能检测

**文件**：[fs/config/config.go:533-620](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go#L533-L620)

```go
// 对每个 IsPassword 字段：
// 1. 先用 Reveal() 尝试解密用户输入
// 2. 若解密成功 → 用户输入已经是 obscure 后的密文，直接存储
// 3. 若解密失败 → 用户输入是明文，调用 Obscure() 加密后存储
_, err := obscure.Reveal(vStr)
if err != nil || opt.Obscure {
    vStr, err = obscure.Obscure(vStr)
}
```

**边界漏洞**：若明文密码恰好是 22 字符以上且全为 Base64 URL 安全字符，可能被误判为已 obscure。可通过 `--obscure`（强制加密）或 `--no-obscure`（强制不加密）标志消除歧义。

### 3.5 RC API 的安全边界

**文件**：[fs/config/rc.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/rc.go)

RC 接口（`config/dump`、`config/get`）**完全不做脱敏**，直接从 Storage 读取原始值：

```go
func rcDump(ctx context.Context, in rc.Params) (out rc.Params, err error) {
    return DumpRcBlob(), nil  // 直接返回所有配置，无脱敏
}

func rcGet(ctx context.Context, in rc.Params) (out rc.Params, err error) {
    name, _ := in.GetString("name")
    return DumpRcRemote(name), nil  // 直接返回单个 remote，无脱敏
}
```

**安全边界**：RC API 的安全性完全依赖于 RC 服务自身的访问控制（`--rc-addr`、`--rc-user`、`--rc-htpasswd`）。若 RC 服务未启用认证，任何能访问端口的攻击者都可获取所有配置（含 obscure 后的密码，而 obscure 可逆）。

---

## 四、敏感字段标记体系

### 4.1 Option 结构定义

**文件**：[fs/registry.go:224-241](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/registry.go#L224-L241)

```go
type Option struct {
    Name       string
    Help       string
    Provider   string
    Default    interface{}
    // ...
    IsPassword bool   // 密码字段：自动 obscure + 交互特殊输入 + show脱敏
    Sensitive  bool   // 敏感字段：仅 redacted 输出脱敏
    // ...
}
```

### 4.2 两种标记的完整差异

| 维度 | `IsPassword` | `Sensitive` |
|------|-------------|-------------|
| 存储自动混淆 | ✅ 是 | ❌ 否 |
| 交互特殊输入 | ✅ 是（`ChoosePassword()`） | ❌ 否 |
| CLI 帮助追加 `(obscured)` | ✅ 是 | ❌ 否 |
| `config show` 脱敏 | ✅ `*** ENCRYPTED ***` | ❌ 显示原值 |
| `config redacted` 脱敏 | ✅ `XXX` | ✅ `XXX` |
| `config dump` / RC 脱敏 | ❌ obscure 后的值 | ❌ 原值 |
| 连接字符串脱敏 | ❌ obscure 后的值 | ❌ 原值 |

---

## 五、安全边界速查总结

### 5.1 各出口的密码暴露风险

| 操作 | 密码明文暴露风险 | 说明 |
|------|--------------|------|
| 配置文件（未加密） | **中** | obscure 混淆，但密钥公开在源码 |
| 配置文件（已加密） | **低** | NaCl Secretbox 强加密，需破解密码 |
| `rclone config show` | **无** | 显示 `*** ENCRYPTED ***` |
| `rclone config redacted` | **无** | 显示 `XXX` |
| `rclone config dump` | **中** | 输出 obscure 后的值，可逆向解密 |
| RC `config/dump` | **中** | 同 dump，完全依赖 RC 认证保护 |
| RC `config/get` | **中** | 同 dump |
| `rclone config string` | **中** | 同 dump |
| 环境变量 `RCLONE_CONFIG_PASS` | **中高** | 明文密码，可能被其他进程读取 |
| `--password-command` 输出 | **中** | 命令行可能在进程列表可见 |
| 临时密钥文件 `_RCLONE_CONFIG_KEY_FILE` | **中** | obscure 外层，文件系统可读即泄露 |
| 内存中的 `configKey` | **低** | 仅内存，需进程内存读取权限 |
| `rclone obscure <明文密码>` | **高** | 明文密码出现在命令行参数中 |

### 5.2 三层防护的攻击者模型

| 层次 | 防护对象 | 攻击者模型 | 安全性评估 |
|------|---------|-----------|-----------|
| 文件级加密 | 整个配置文件 | 获取了配置文件但不知道密码 | **强** |
| Obscure 字段混淆 | 单个密码字段 | 能看到配置文件内容但没有 rclone 源码 | **弱**（密钥公开） |
| show 输出脱敏 | 终端肩窥 | 偷看屏幕或日志 | **弱**（仅 UI 层，其他出口仍可见） |
| redacted 输出脱敏 | 公开发布 | 将配置贴到论坛求助 | **中**（双重标记，但可能遗漏） |
| RC / JSON 导出 | — | 任何能调用 API 的人 | **无保护**（完全依赖 RC 自身认证） |

---

## 六、关键代码索引

| 功能 | 文件 |
|------|------|
| Obscure 加解密核心 | [obscure.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/obscure/obscure.go) |
| Obscure 单元测试 | [obscure_test.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/obscure/obscure_test.go) |
| 配置文件加解密 | [crypt.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go) |
| 配置存取核心 | [config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go) |
| INI 文件存储 | [configfile.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/configfile/configfile.go) |
| Option 结构定义 | [registry.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/registry.go#L224-L241) |
| UI 交互与显示脱敏 | [ui.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/ui.go) |
| RC 配置接口 | [rc.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/rc.go) |
| RC 全局选项 | [rc/config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/rc/config.go) |
| 标志处理 | [flags.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/flags/flags.go) |
| ConfigMap 优先级 | [configmap.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/configmap.go) |
| 后端配置状态机 | [backend_config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/backend_config.go) |
| config 子命令 | [cmd/config/config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/config/config.go) |
| obscure 命令 | [cmd/obscure/obscure.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/obscure/obscure.go) |
| reveal 命令 | [cmd/reveal/reveal.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/reveal/reveal.go) |
