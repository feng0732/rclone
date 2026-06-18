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

密码获取优先级：
1. 已设置的 `configKey`（内存缓存）
2. `--password-command` 命令执行结果
3. `RCLONE_CONFIG_PASS` 环境变量
4. `_RCLONE_CONFIG_KEY_FILE` 临时文件（守护进程模式）
5. 交互提示（若 `AskPassword=true`）

解密失败会循环提示重新输入密码（除非非交互模式）。

#### 守护进程密钥传递

当 `PassConfigKeyForDaemonization=true` 时：
1. 计算出的 `configKey` 用 `obscure.Obscure()` 再次混淆
2. 写入临时文件，路径存入环境变量 `_RCLONE_CONFIG_KEY_FILE`
3. 子进程读取该文件后立即删除

参见：[SetConfigPassword()](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/crypt.go#L272-L316)

### 2.4 文件写入原子性

位置：[configfile.go:101-206](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/configfile/configfile.go#L101-L206)

保存策略：
1. 在同目录创建临时文件写入新配置
2. 将原文件重命名为 `.old` 备份
3. 将临时文件重命名为最终路径
4. 删除备份（成功后）
5. 文件权限默认 `0600`，保留原有权限

---

## 三、敏感字段保护的实现边界

### 3.1 字段标记元数据

位置：[fs/registry.go:224-241](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/registry.go#L224-L241)

```go
type Option struct {
    Name       string
    IsPassword bool    // 标记为密码，将自动 obscure 存储
    Sensitive  bool    // 标记为敏感，redacted 输出时脱敏为 XXX
    // ... 其他字段
}
```

### 3.2 两种标记的区别

| 特性 | IsPassword | Sensitive |
|------|------------|-----------|
| **存储时自动混淆** | ✅ 是 — 自动调用 `obscure.Obscure()` | ❌ 否 — 明文存储 |
| **交互输入方式** | ✅ 是 — 使用 `ChoosePassword()` 特殊流程 | ❌ 否 — 普通文本输入 |
| **命令行帮助** | ✅ 是 — 追加 `(obscured)` 提示 | ❌ 否 |
| **显示脱敏（普通）** | ✅ 显示 `*** ENCRYPTED ***` | ❌ 显示原值 |
| **显示脱敏（redacted）** | ✅ 显示 `XXX` | ✅ 显示 `XXX` |

### 3.3 IsPassword 的自动 Obscure 流程

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
            // Reveal 失败说明是明文，需要 obscure
            // 或强制 obscure 模式
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

### 3.4 密码输入流程

位置：[fs/config/ui.go:234-275](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/ui.go#L234-L275)

`ChoosePassword()` 提供三种方式：
- **y** — 手动输入密码（两次确认）
- **g** — 生成随机密码（64~1024 位强度）
- **n** — 保留现有值或留空

选择后**立即调用 `obscure.MustObscure()` 加密**后返回。

### 3.5 显示脱敏

位置：[fs/config/ui.go:338-367](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/ui.go#L338-L367)

`printRemoteOptions()` 中的处理：

```go
// 普通显示模式
if isPassword && value != "" {
    fmt.Printf("%s%s%s*** ENCRYPTED ***\n", prefix, key, sep)
}

// Redacted 显示模式
if redacted && (isSensitive || isPassword) && value != "" {
    fmt.Printf("%s%s%sXXX\n", prefix, key, sep)
}
```

- `rclone config show` → 普通模式，密码显示 `*** ENCRYPTED ***`，其他敏感字段显示原值
- `rclone config redacted` → Redacted 模式，`IsPassword` 和 `Sensitive` 字段均显示 `XXX`

### 3.6 命令行标志中的处理

位置：[fs/config/flags/flags.go:357-397](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/flags/flags.go#L357-L397)

当 `opt.IsPassword` 为 true 时，命令行帮助文本自动追加 `(obscured)` 后缀。

### 3.7 环境变量覆盖

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

## 四、三层安全防护总结

### 防护层次图

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: 字段级显示脱敏                                      │
│  ┌─────────────┐    ┌─────────────┐                          │
│  │ IsPassword  │    │  Sensitive  │                          │
│  │ (*** ENCRYPTED)│  │  (XXX)      │  <- 仅影响 UI 输出       │
│  └─────────────┘    └─────────────┘                          │
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
└─────────────────────────────────────────────────────────────┘
```

### 各层的边界与局限

| 层次 | 保护对象 | 攻击者模型 | 安全性 |
|------|---------|-----------|--------|
| 文件级加密 | 整个配置文件 | 获取配置文件但不知道密码 | **强** — 需要破解密码 |
| Obscure 混淆 | 单个密码字段 | 能看到配置文件内容但无 rclone 源码 | **弱** — 密钥在源码中公开 |
| 显示脱敏 | 日志/终端输出 | 肩窥或查看日志 | **弱** — 仅 UI 层面不显示真实值 |

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
| 标志处理 | [flags.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/config/flags/flags.go) |
| ConfigMap 优先级 | [configmap.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/configmap.go) |
| 后端配置状态机 | [backend_config.go](file:///d:/fz/0601-2/solo-dogfeeding/code/60-rclone/fs/backend_config.go) |
