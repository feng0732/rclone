# Rclone 配置文件加密与敏感字段保护分析

## 概述

Rclone 的配置安全体系分为三个层次：
1. **Obscure 字段级混淆** — 对单个密码字段进行可逆加密（AES-CTR），防止"肩窥"
2. **配置文件级加密** — 对整个配置文件使用 NaCl secretbox（XSalsa20-Poly1305）进行对称加密
3. **敏感字段标记** — 通过 `IsPassword` 和 `Sensitive` 元数据标记控制显示/脱敏输出

---

## 一、Obscure 处理机制

### 1.1 核心实现

位置：[fs/config/obscure/obscure.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/obscure/obscure.go)

#### 加密算法

采用 **AES-256-CTR** 模式，使用硬编码的 32 字节密钥：

```go
// [obscure.go:19-24](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/obscure/obscure.go#L19-L24)
var cryptKey = []byte{
    0x9c, 0x93, 0x5b, 0x48, 0x73, 0x0a, 0x55, 0x4d,
    0x6b, 0xfd, 0x7c, 0x63, 0xc8, 0x86, 0xa9, 0x2b,
    0xd3, 0x90, 0x19, 0x8e, 0xb8, 0x12, 0x8a, 0xfb,
    0xf4, 0xde, 0x16, 0x2b, 0x8b, 0x95, 0xf6, 0x38,
}
```

#### 加密流程 [Obscure()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/obscure/obscure.go#L50-L64)

```
明文 → 生成随机 16 字节 IV → AES-CTR 加密 → [IV | 密文] → Base64(RawURLEncoding)
```

关键特征：
- IV 为每个加密独立生成的 16 字节随机值（`crypto/rand`）
- IV 拼接在密文前面，共 `16 + len(明文)` 字节
- 输出使用 `base64.RawURLEncoding`（无填充、URL 安全字符）
- 最大长度限制：`MaxInt32 - BlockSize`

#### 解密流程 [Reveal()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/obscure/obscure.go#L76-L90)

```
Base64 解码 → 拆分为 [前16字节=IV | 剩余=密文] → AES-CTR XOR → 明文
```

由于 CTR 模式是流密码，加密和解密是**同一操作**（XORKeyStream）。

#### 错误检测

解密时会检测：
- Base64 解码失败 → "base64 decode failed when revealing password - is it obscured?"
- 长度不足 16 字节 → "input too short when revealing password - is it obscured?"

### 1.2 安全边界

| 特性 | 说明 |
|------|------|
| 算法强度 | AES-256-CTR 本身是安全的 |
| 密钥管理 | **密钥硬编码在源码中**，无安全性可言 |
| 完整性保护 | **无 MAC/AEAD**，无法检测密文被篡改 |
| 设计目的 | 仅防止"肩窥"（someone seeing a password by accident） |
| 可逆性 | 完全可逆，任何拿到源码的人都能解密 |

文档中明确说明（[cmd/obscure/obscure.go:22-30](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/obscure/obscure.go#L22-L30)）：

> This is **not** a secure way of encrypting these passwords as rclone can decrypt them - it is to prevent "eyedropping"

### 1.3 命令行工具

- **obscure** 命令：[cmd/obscure/obscure.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/obscure/obscure.go) — 支持从参数或 STDIN 读取密码
- **reveal** 命令：[cmd/reveal/reveal.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/reveal/reveal.go) — 隐藏命令（`Hidden: true`），用于调试解密

---

## 二、配置文件存取机制

### 2.1 Storage 接口抽象

位置：[fs/config/config.go:77-109](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go#L77-L109)

```go
type Storage interface {
    GetSectionList() []string
    HasSection(section string) bool
    DeleteSection(section string)
    GetKeyList(section string) []string
    GetValue(section string, key string) (value string, found bool)
    SetValue(section string, key string, value string)
    DeleteKey(section string, key string) bool
    Load() error
    Save() error
    Serialize() (string, error)
}
```

默认实现为基于 INI 文件的 [configfile.Storage](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/configfile/configfile.go)。

### 2.2 配置文件定位

位置：[fs/config/config.go:211-316](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go#L211-L316)

查找优先级：
1. `<rclone_exe_dir>/rclone.conf`
2. Windows: `%APPDATA%/rclone/rclone.conf`
3. `$XDG_CONFIG_HOME/rclone/rclone.conf`
4. `~/.config/rclone/rclone.conf`
5. `~/.rclone.conf`（旧版）

可通过 `--config` 标志或 `RCLONE_CONFIG` 环境变量覆盖。空路径或特殊路径表示**内存配置**。

### 2.3 配置文件级加密

位置：[fs/config/crypt.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go)

#### 加密标识

加密配置文件的第一个非空、非注释行必须为：

```
RCLONE_ENCRYPT_V0:
```

后续是 Base64 编码的加密数据。版本不匹配时报错："unsupported configuration encryption - update rclone for support"。

#### 加密算法

使用 **NaCl secretbox**（XSalsa20 流密码 + Poly1305 MAC），来自 `golang.org/x/crypto/nacl/secretbox`。

- Nonce: 24 字节随机
- 密钥派生: `SHA256("[" + password + "][rclone-config]")` → 32 字节密钥
- 格式: `[24字节nonce][secretbox密文]` → Base64(StdEncoding)

#### 加密流程 [Encrypt()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go#L216-L251)

```
明文 → 生成 24 字节随机 nonce → secretbox.Seal() → [nonce | 密文] → Base64 编码 → 写入文件
```

文件头部写入：
```
# Encrypted rclone configuration File

RCLONE_ENCRYPT_V0:
<base64 data>
```

#### 解密流程 [Decrypt()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go#L50-L179)

密码获取优先级（详见第三章详述）：
1. 已设置的 `configKey`（内存缓存）
2. `_RCLONE_CONFIG_KEY_FILE` 临时文件（守护进程模式）
3. `--password-command` 命令执行结果
4. `RCLONE_CONFIG_PASS` 环境变量
5. 交互提示（若 `AskPassword=true`）

解密失败会循环提示重新输入密码（除非非交互模式）。

#### 守护进程密钥传递

当 `PassConfigKeyForDaemonization=true` 时：
1. 计算出的 `configKey` 用 `obscure.Obscure()` 再次混淆
2. 写入临时文件，路径存入环境变量 `_RCLONE_CONFIG_KEY_FILE`
3. 子进程读取该文件后立即删除

参见：[SetConfigPassword()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go#L272-L316)

### 2.4 文件写入原子性与重试机制

位置：[configfile.go:101-206](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/configfile/configfile.go#L101-L206)

保存策略：
1. 在同目录创建临时文件写入新配置
2. 将原文件重命名为 `.old` 备份
3. 将临时文件重命名为最终路径
4. 删除备份（成功后）
5. 文件权限默认 `0600`，保留原有权限

#### 保存重试机制 [SaveConfig()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go#L383-L397)

```go
for range ci.LowLevelRetries + 1 {
    if err = LoadedData().Save(); err == nil {
        return
    }
    waitingTimeMs := mathrand.Intn(1000)
    time.Sleep(time.Duration(waitingTimeMs) * time.Millisecond)
}
```

- 默认重试 `10 + 1 = 11` 次（`LowLevelRetries` 默认值为 10）
- 每次失败后随机等待 0~1000ms
- 最终失败仅打印错误日志，不返回 error（调用方无法感知失败）

**安全影响**：
- 多次重试增加了部分写入的配置在磁盘上停留的时间窗口
- 随机退避可能被用于防暴力破解的思路不同，此处仅用于并发写入冲突
- `SaveConfig()` 是 fire-and-forget 模式，静默失败可能导致配置变更未持久化

---

## 三、配置密码来源与密钥传递安全边界

### 3.1 配置密码的五种来源与优先级

位置：[fs/config/crypt.go:127-178](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go#L127-L178)

解密时按以下优先级尝试，逐级 fallback：

| 优先级 | 来源 | 触发条件 | 安全级别 |
|--------|------|---------|---------|
| 1 | `configKey` 内存缓存 | 已通过 `SetConfigPassword()` 设置 | 高 — 仅内存 |
| 2 | `_RCLONE_CONFIG_KEY_FILE` 临时文件 | 环境变量存在且文件可读 | 中 — 文件系统可读即泄露 |
| 3 | `--password-command` 命令 | `Command` 非空 | 中 — 命令行参数可能泄露 |
| 4 | `RCLONE_CONFIG_PASS` 环境变量 | 环境变量存在 | 中低 — 环境变量可能被其他进程读取 |
| 5 | 交互提示 | `AskPassword=true` 且上述均失败 | 高 — 仅内存传递 |

#### 优先级代码结构（Decrypt 函数核心循环）：

```
for {
    if 临时文件存在 → 读取并删除临时文件 → 尝试解密
    else if configKey 已缓存 → 尝试解密
    else if passwordCommand → 执行命令获取密码 → 派生密钥 → 尝试解密
    else if RCLONE_CONFIG_PASS → 读取环境变量 → 派生密钥 → 尝试解密
    else if AskPassword → 交互提示输入密码 → 派生密钥 → 尝试解密

    解密成功 → 保存 configKey → 返回
    解密失败 → 若非交互模式 → 返回错误
            → 若交互模式 → 继续循环提示
}
```

### 3.2 临时密钥文件传递机制详解

位置：[fs/config/crypt.go:284-313](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go#L284-L313)

#### 写入流程（父进程）

```go
if PassConfigKeyForDaemonization {
    tempFile, _ := os.CreateTemp("", "rclone")
    tempFile.WriteString(obscure.MustObscure(string(configKey)))
    tempFile.Close()
    os.Setenv("_RCLONE_CONFIG_KEY_FILE", tempFile.Name())
}
```

#### 读取流程（子进程）

```go
if envKeyFile := os.Getenv("_RCLONE_CONFIG_KEY_FILE"); len(envKeyFile) > 0 {
    obscuredKey, _ := os.ReadFile(envKeyFile)
    os.Remove(envKeyFile)  // 读取后立即删除
    configKey = []byte(obscure.MustReveal(string(obscuredKey)))
}
```

#### 安全边界分析

| 风险点 | 说明 | 严重程度 |
|--------|------|----------|
| **临时文件位置 | 使用 `os.CreateTemp("", "rclone")` 在系统临时目录 | 中 |
| **文件权限** | Go 默认权限（通常 0600） | 低 |
| **混淆强度** | 外层再加一层 obscure（密钥硬编码） | 低 — 形同虚设 |
| **TOCTOU 风险 | 文件读取后删除，但中间有时间窗口 | 中 |
| **环境变量泄露** | `_RCLONE_CONFIG_KEY_FILE` 路径可能通过 `/proc/` 环境 | 中 |
| **多子进程共享 | 所有子进程共享同一环境变量 | 中 |
| **读取失败清理** | 读取失败时也会尝试删除文件 | 低 |

**关键结论**：临时密钥文件机制的 obscure 外层加密**不提供真实安全性**，因为 obscure 密钥公开在源码中。它的真实作用是**防止意外肩窥（例如在 `ps` 或 `/proc` 环境变量中直接看到密钥原文）和**避免密钥明文出现在核心转储中。

### 3.3 解密错误重试的安全影响

位置：[fs/config/crypt.go:127-178](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go#L127-L178)

#### 交互模式下的无限重试

```go
for {
    // ... 尝试解密 ...
    if err == nil {
        return out, nil  // 成功
    }
    if !AskPassword {
        return nil, err  // 非交互模式：一次失败即返回
    }
    // 交互模式：打印错误，继续循环提示
    fmt.Fprintf(PasswordPromptOutput, "Error: failed to read configuration: %v\n", err)
}
```

**安全影响**：
- **交互模式**：无限制重试密码 → 无暴力破解防护（本地场景下可暴力破解本就无意义，因为攻击者已能访问配置文件）
- **非交互模式**：一次失败即终止 → 防止脚本环境中不会泄露过多信息
- **password-command 模式**：解密失败立即返回错误 → 防止命令被重复执行 → 避免命令执行侧信道

#### password-command 的特殊处理

```go
if usingPasswordCommand {
    return nil, errors.New("using --password-command derived password, unable to decrypt configuration")
}
```

`--password-command` 解密失败时**不重试也不提示**，直接返回错误。这避免了：
- 重复执行外部命令带来的性能开销
- 命令执行可能产生的侧信道信息泄露
- 恶意命令被多次执行的风险

---

## 四、配置展示与导出的脱敏边界

### 4.1 五种展示/导出方式对比

Rclone 提供了至少 8 种配置输出方式，脱敏策略各异：

| 输出方式 | 实现函数 | IsPassword 字段 | Sensitive 字段 | 其他字段 |
|---------|----------|----------------|---------------|---------|
| `rclone config show | [ShowRemote()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/ui.go#L318-L336) | `*** ENCRYPTED ***` | 原值（obscure后） | 原值 |
| `rclone config redacted | [ShowRedactedRemote()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/ui.go#L380-L405) | `XXX` | `XXX` | 原值 |
| `rclone config dump` | [Dump()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go#L718-L730) | 原值（obscure后） | 原值 | 原值 |
| RC `config/dump` | [DumpRcBlob()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go#L708-L716) | 原值（obscure后） | 原值 | 原值 |
| RC `config/get` | [DumpRcRemote()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go#L699-L706) | 原值（obscure后） | 原值 | 原值 |
| `rclone config string` | [configStringCommand](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/config/config.go#L618-L667) | 原值（obscure后） | 原值 | 原值 |
| RC `options/get` | [rcOptionsGet()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/rc/config.go#L81-L87) | 原值（全局配置） | 原值 | 原值 |
| Storage 接口直接读取 | `GetValue()` | 原值（obscure后） | 原值 | 原值 |

### 4.2 配置展示（config show）

位置：[fs/config/ui.go:338-378](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/ui.go#L338-L378)

`printRemoteOptions(name, prefix, sep, redacted=false)`

```go
if isPassword && value != "" {
    fmt.Printf("%s%s%s*** ENCRYPTED ***\n", prefix, key, sep)
}
```

- **脱敏规则**：
  - IsPassword=true 且非空 → 显示 `*** ENCRYPTED ***`
  - Sensitive=true 但 IsPassword=false → **显示原值**（注意：这是一个边界漏洞吗？不，这是设计——Sensitive 仅在 redacted 模式才生效
  - 其他字段 → 显示原值

### 4.3 Redacted 输出（config redacted）

位置：[fs/config/ui.go:380-405](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/ui.go#L380-L405)

`printRemoteOptions(name, prefix, sep, redacted=true)`

```go
if redacted && (isSensitive || isPassword) && value != "" {
    fmt.Printf("%s%s%sXXX\n", prefix, key, sep)
}
```

- **脱敏规则**：
  - IsPassword=true 或 Sensitive=true 且非空 → 显示 `XXX`
  - 其他字段 → 显示原值
- **设计目的**：生成适合公开求助的配置片段
- **文档提示**：命令末尾打印 "Double check the config for sensitive info before posting publicly"

**注意**：redaction 可能不完美（文档中明确说明 "the redaction may not be perfect"）

### 4.4 JSON 导出（config dump）

位置：[fs/config/config.go:718-730](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go#L718-L730)

```go
func Dump() error {
    out := DumpRcBlob()
    // JSON 编码输出到 stdout
    enc := json.NewEncoder(os.Stdout)
    enc.SetIndent("", "\t")
    return enc.Encode(out)
}
```

- **脱敏规则**：**完全不脱敏**，所有字段明文输出
- IsPassword 字段输出的是 **obscure 后的值（不是明文密码本身）
- Sensitive 字段输出原值

**安全边界**：`config dump` 输出的密码字段是 obscure 混淆后的密文，虽然不是明文。但由于 obscure 密钥公开，拿到 dump 输出等价于明文。

### 4.5 RC API 配置导出

#### config/dump 和 config/get

位置：[fs/config/rc.go:62-89](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/rc.go#L62-L89)

```go
func rcDump(ctx context.Context, in rc.Params) (out rc.Params, err error) {
    return DumpRcBlob(), nil
}

func rcGet(ctx context.Context, in rc.Params) (out rc.Params, err error) {
    name, _ := in.GetString("name")
    return DumpRcRemote(name), nil
}
```

- **完全不脱敏**，与 `config dump` 相同
- 直接从 Storage 读取原始值
- **无权限控制**（RC 接口本身需要外部访问控制）

#### RC API 的安全边界：
- RC 接口的安全性完全依赖于 RC 服务本身的访问控制（`--rc-addr`、`--rc-user`、`--rc-htpasswd`）
- 如果 RC 服务未正确配置认证，任何能访问 RC 端口的人都能获取所有配置（含 obscure 后的密码）
- obscure 后的密码可通过 `rclone reveal` 或源码解密

### 4.6 连接字符串输出

位置：[cmd/config/config.go:618-667](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/config/config.go#L618-L667)

```
:s3,access_key_id=XXX,no_check_bucket,provider=AWS,region=eu-west-2,secret_access_key=YYY:rclone
```

- **完全不脱敏**，所有非默认值明文输出
- 输出的是 obscure 后的值（与配置文件中存储的值相同）
- 设计用于脚本/API 调用方便性，安全性依赖使用者自行保护

---

## 五、敏感字段保护的实现边界

### 5.1 字段标记元数据

位置：[fs/registry.go:224-241](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/registry.go#L224-L241)

```go
type Option struct {
    Name       string
    IsPassword bool    // 标记为密码，将自动 obscure 存储
    Sensitive  bool    // 标记为敏感，redacted 输出时脱敏为 XXX
    // ... 其他字段
}
```

### 5.2 两种标记的区别

| 特性 | IsPassword | Sensitive |
|------|------------|-----------|
| **存储时自动混淆** | ✅ 是 — 自动调用 `obscure.Obscure()` | ❌ 否 — 明文存储 |
| **交互输入方式** | ✅ 是 — 使用 `ChoosePassword()` 特殊流程 | ❌ 否 — 普通文本输入 |
| **命令行帮助** | ✅ 是 — 追加 `(obscured)` 提示 | ❌ 否 |
| **config show** | ✅ 显示 `*** ENCRYPTED ***` | ❌ 显示原值 |
| **config redacted** | ✅ 显示 `XXX` | ✅ 显示 `XXX` |
| **config dump / RC** | ❌ 显示 obscure 后的值 | ❌ 显示原值 |
| **连接字符串** | ❌ 显示 obscure 后的值 | ❌ 显示原值 |

### 5.3 IsPassword 的自动 Obscure 流程

位置：[fs/config/config.go:533-620](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go#L533-L620)

在 `updateRemote()` 中：

```go
// 构建需要 obscure 的字段集合
needsObscure := map[string]struct{}{}
if !opt.NoObscure {
    for _, option := range ri.Options {
        if option.IsPassword {
            needsObscure[option.Name] = struct{}{}
        }
    }
}

// 设置值时自动混淆
for k, v := range keyValues {
    vStr := fmt.Sprint(v)
    if _, ok := needsObscure[k]; ok {
        _, err := obscure.Reveal(vStr)
        if err != nil || opt.Obscure {
            vStr, err = obscure.Obscure(vStr)
        }
    }
    choices.Set(k, vStr)
}
```

智能检测逻辑：
1. 先用 `Reveal()` 尝试解密输入值
2. 若解密成功 → 输入已经是 obscure 后的密文，直接存储
3. 若解密失败 → 输入是明文，调用 `Obscure()` 加密后存储
4. `--obscure` 标志强制加密，`--no-obscure` 标志禁用自动加密

**边界漏洞**：如果密码是 22 字符以上且全为 base64 字符的明文密码可能被误判为已 obscure。文档中明确提示了这一点（[cmd/config/config.go:189-196](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/config/config.go#L189-L196)）。

### 5.4 密码输入流程

位置：[fs/config/ui.go:234-275](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/ui.go#L234-L275)

`ChoosePassword()` 提供三种方式：
- **y** — 手动输入密码（两次确认）
- **g** — 生成随机密码（64~1024 位强度）
- **n** — 保留现有值或留空

选择后**立即调用 `obscure.MustObscure()` 加密**后返回。

### 5.5 命令行标志中的处理

位置：[fs/config/flags/flags.go:357-397](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/flags/flags.go#L357-L397)

当 `opt.IsPassword` 为 true 时，命令行帮助文本自动追加 `(obscured)` 后缀。

### 5.6 环境变量覆盖

位置：[fs/configmap.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/configmap.go)

ConfigMap 的取值优先级（从高到低）：
1. 连接字符串参数
2. 命令行标志值
3. Remote 特定环境变量 `RCLONE_CONFIG_<REMOTE>_<KEY>`
4. Backend 特定环境变量 `RCLONE_<PREFIX>_<KEY>`
5. **配置文件**
6. 默认值

**注意**：通过环境变量传入的 `IsPassword` 字段**不会自动 obscure**，需要用户手动传入 obscure 后的值，或使用 `rclone obscure` 命令预加密。

---

## 六、三层安全防护总结

### 防护层次图

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: 输出脱敏 (UI/Redacted)                          │
│  ┌─────────────┐    ┌─────────────┐                          │
│  │ config show │    │  redacted    │                          │
│  │ *** ENCRYPTED│ │  XXX         │  <- 仅影响显示             │
│  └─────────────┘    └─────────────┘                          │
│  config dump / RC API / 连接字符串 → 全部明文(obscure后)         │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: 字段级混淆 (Obscure)                               │
│  AES-256-CTR + 硬编码密钥                                     │
│  防止肩窥，可逆                                               │
│  作用于 IsPassword=true 的字段存入 rclone.conf 时             │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: 文件级加密 (Config Password)                       │
│  NaCl Secretbox (XSalsa20-Poly1305)                          │
│  SHA256(password + salt) 派生密钥                             │
│  作用于整个 rclone.conf 文件                                  │
│  密码来源: 内存缓存 > 临时文件 > 命令 > 环境变量 > 交互       │
└─────────────────────────────────────────────────────────────┘
```

### 各层的边界与局限

| 层次 | 保护对象 | 攻击者模型 | 安全性 |
|------|---------|-----------|--------|
| 文件级加密 | 整个配置文件 | 获取配置文件但不知道密码 | **强** — 需要破解密码 |
| Obscure 混淆 | 单个密码字段 | 能看到配置文件内容但无 rclone 源码 | **弱** — 密钥在源码中公开 |
| 显示脱敏 (show) | 终端肩窥 | 偷看屏幕/日志 | **弱** — 仅 UI 层面，其他出口仍可见 |
| Redacted 脱敏 | 公开分享 | 公开发布配置求助 | **中** — 双重标记覆盖，但可能遗漏 |
| RC/JSON 导出 | - | **无保护** | **无** — 直接输出 obscure 后的值 |

### 关键实现文件索引

| 功能 | 文件 |
|------|------|
| Obscure 加解密核心 | [obscure.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/obscure/obscure.go) |
| Obscure 单元测试 | [obscure_test.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/obscure/obscure_test.go) |
| 配置文件加解密 | [crypt.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go) |
| 配置存取核心逻辑 | [config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/config.go) |
| INI 文件存储实现 | [configfile.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/configfile/configfile.go) |
| Option 结构定义 | [registry.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/registry.go#L224-L241) |
| UI 交互与显示脱敏 | [ui.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/ui.go) |
| RC 配置接口 | [rc.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/rc.go) |
| RC 全局选项 | [rc/config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/rc/config.go) |
| 标志处理 | [flags.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/flags/flags.go) |
| ConfigMap 优先级 | [configmap.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/configmap.go) |
| 后端配置状态机 | [backend_config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/backend_config.go) |
| config 子命令 | [cmd/config/config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/cmd/config/config.go) |

### 安全边界速查表

| 操作 | 密码明文暴露风险 | 备注 |
|------|--------------|------|
| 配置文件（未加密） | 中 | obscure 混淆，但密钥公开 |
| 配置文件（已加密） | 低 | NaCl secretbox 强加密 |
| `rclone config show | 无 | 显示 `*** ENCRYPTED ***` |
| `rclone config redacted | 无 | 显示 `XXX` |
| `rclone config dump | 中 | obscure 后的值，可逆向解密 |
| RC `config/dump` | 中 | 同 dump，依赖 RC 认证保护 |
| `rclone config string | 中 | 同 dump |
| 环境变量 `RCLONE_CONFIG_PASS` | 中高 | 明文密码，可能被其他进程读取 |
| `--password-command` 输出 | 中 | 命令行可能在进程列表可见 |
| 临时密钥文件 | 中 | obscure 外层，文件系统可读即泄露 |
| 内存中的 `configKey` | 低 | 仅内存，需进程内存读取 |
| `rclone obscure 命令行参数 | 高 | 明文密码出现在命令行参数中 |
