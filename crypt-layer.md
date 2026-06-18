# Crypt 加密层代码实现分析

## 一、整体架构

rclone 的 crypt 后端是一个**包装层（Wrapper）**，它不直接存储数据，而是将加密/解密操作透明地叠加在另一个底层 remote 之上。

```
用户操作 → crypt.Fs → 文件名加密/解密 → 底层 fs.Fs → 实际存储
         ↓
      内容加密/解密
         ↓
      数据流包装
```

核心文件（相对路径，基于项目根目录）：
- `backend/crypt/cipher.go` — 密码学核心实现（密钥派生、文件名加密、内容加密流）
- `backend/crypt/crypt.go` — FS 和 Object 的包装层实现
- `backend/crypt/pkcs7/pkcs7.go` — PKCS#7 填充

---

## 二、密钥派生（Key Derivation）

### 2.1 Cipher 结构体

`Cipher`（`backend/crypt/cipher.go`）是加密核心，包含三类密钥：

```go
type Cipher struct {
    dataKey   [32]byte       // 内容加密密钥（NACL secretbox 使用）
    nameKey   [32]byte       // 文件名加密密钥（AES 使用）
    nameTweak [16]byte       // 文件名加密的调整量（EME 模式使用）
    block     gocipher.Block // AES cipher 实例
    mode      NameEncryptionMode
    // ...
}
```

### 2.2 scrypt 密钥派生

`Key()` 方法（`backend/crypt/cipher.go`）使用 **scrypt** 算法从用户密码派生密钥：

```go
func (c *Cipher) Key(password, salt string) (err error) {
    keySize := len(c.dataKey) + len(c.nameKey) + len(c.nameTweak) // 32+32+16 = 80 字节
    // scrypt 参数: N=16384, r=8, p=1
    key, err := scrypt.Key([]byte(password), saltBytes, 16384, 8, 1, keySize)

    copy(c.dataKey[:], key[:32])          // 前 32 字节 → 内容加密密钥
    copy(c.nameKey[:], key[32:64])        // 中间 32 字节 → 文件名加密密钥
    copy(c.nameTweak[:], key[64:80])      // 后 16 字节 → EME tweak

    c.block, err = aes.NewCipher(c.nameKey[:])
    return err
}
```

---

## 三、文件名加密机制

### 3.1 三种加密模式

`NameEncryptionMode`（`backend/crypt/cipher.go`）定义了三种模式：

| 模式 | 说明 |
|------|------|
| `NameEncryptionOff` | 不加密，仅添加后缀（默认 `.bin`，可配置） |
| `NameEncryptionStandard` | 标准加密，使用 EME-AES + PKCS#7 |
| `NameEncryptionObfuscated` | 简单混淆，字符旋转 |

### 3.2 四个公开方法 — 模式分流与跳过规则

文件名加密的对外入口是四个公开方法（均在 `backend/crypt/cipher.go`）。每个方法首先做**模式分流**和**目录名跳过判断**，再决定是否进入实际加密/解密路径。

#### `EncryptFileName(in)` — 加密文件路径

```go
func (c *Cipher) EncryptFileName(in string) string {
    if c.mode == NameEncryptionOff {
        return in + c.encryptedSuffix // off 模式：原文件名 + 后缀（如 .bin）
    }
    return c.encryptFileName(in) // standard/obfuscate 模式：逐段处理
}
```

| 模式 | 行为 | 后缀 |
|------|------|------|
| `Off` | **不加密**，直接在末尾追加 `encryptedSuffix` | 追加 `.bin`（或自定义后缀） |
| `Standard` | 逐段 EME-AES 加密 | 不追加后缀 |
| `Obfuscated` | 逐段字符旋转混淆 | 不追加后缀 |

#### `DecryptFileName(in)` — 解密文件路径

```go
func (c *Cipher) DecryptFileName(in string) (string, error) {
    if c.mode == NameEncryptionOff {
        // off 模式：剥掉后缀，并校验剩余是否合法
        remainingLength := len(in) - len(c.encryptedSuffix)
        if remainingLength == 0 || !strings.HasSuffix(in, c.encryptedSuffix) {
            return "", ErrorNotAnEncryptedFile
        }
        decrypted := in[:remainingLength]
        // 如果去掉后缀后整个名字看起来像个版本号（如 v1.2.3），则拒绝
        if version.Match(decrypted) {
            _, unversioned := version.Remove(decrypted)
            if unversioned == "" {
                return "", ErrorNotAnEncryptedFile
            }
        }
        return decrypted, nil
    }
    return c.decryptFileName(in) // standard/obfuscate 模式：逐段解密
}
```

| 模式 | 行为 | 后缀处理 |
|------|------|---------|
| `Off` | **剥掉** `encryptedSuffix`，返回剩余部分 | 校验后缀必须存在，否则报 `ErrorNotAnEncryptedFile` |
| `Standard` | 逐段 EME-AES 解密 | 不涉及后缀 |
| `Obfuscated` | 逐段字符旋转反混淆 | 不涉及后缀 |

> **特殊校验**：`Off` 模式下若剥掉后缀后整串看起来像版本字符串（如 `1.2.3`），返回错误。防止把版本号误判为文件名。

#### `EncryptDirName(in)` — 加密目录路径

```go
func (c *Cipher) EncryptDirName(in string) string {
    if c.mode == NameEncryptionOff || !c.dirNameEncrypt {
        return in // 两种情况都跳过：off 模式 或 关闭目录名加密
    }
    return c.encryptFileName(in)
}
```

| 条件 | 行为 | 后缀 |
|------|------|------|
| `mode == Off` | **不加密**，返回原目录名 | **不**追加后缀（与文件名不同！） |
| `dirNameEncrypt == false` | **不加密**，返回原目录名 | 不追加后缀 |
| 其他（standard/obfuscate + dirNameEncrypt=true） | 逐段加密 | 不追加后缀 |

> **注意不对称**：`EncryptDirName` 在 `Off` 模式下**不追加** `.bin` 后缀，但 `EncryptFileName` 在 `Off` 模式下**会追加** `.bin` 后缀。这是区分目录与文件的关键。

#### `DecryptDirName(in)` — 解密目录路径

```go
func (c *Cipher) DecryptDirName(in string) (string, error) {
    if c.mode == NameEncryptionOff || !c.dirNameEncrypt {
        return in, nil // off 或关闭目录名加密：原样返回
    }
    return c.decryptFileName(in)
}
```

| 条件 | 行为 |
|------|------|
| `mode == Off` 或 `dirNameEncrypt == false` | **原样返回**，不做任何处理 |
| 其他 | 逐段解密 |

### 3.3 内部逐段处理 — `encryptFileName` / `decryptFileName`

公开方法分流后，standard 和 obfuscate 模式都进入内部逐段处理（`backend/crypt/cipher.go`）。

#### 加密：`encryptFileName()`

```go
func (c *Cipher) encryptFileName(in string) string {
    segments := strings.Split(in, "/") // 按 "/" 切分路径
    for i := range segments {
        // 跳过规则：若关闭目录名加密 且 当前段不是最后一段（即非文件名段）
        if !c.dirNameEncrypt && i != (len(segments)-1) {
            continue // 目录段保持原样
        }

        // 版本处理：只在最后一段（文件名）做版本剥离
        hasVersion := false
        var t time.Time
        if i == (len(segments)-1) && version.Match(segments[i]) {
            t, s := version.Remove(segments[i])
            if s != segments[i] {
                segments[i] = s
                hasVersion = true
            }
        }

        // 模式分流（standard vs obfuscate）
        if c.mode == NameEncryptionStandard {
            segments[i] = c.encryptSegment(segments[i])
        } else {
            segments[i] = c.obfuscateSegment(segments[i])
        }

        // 版本回填：加密/混淆后再加回版本字符串
        if hasVersion {
            segments[i] = version.Add(segments[i], t)
        }
    }
    return strings.Join(segments, "/")
}
```

#### 解密：`decryptFileName()`

```go
func (c *Cipher) decryptFileName(in string) (string, error) {
    segments := strings.Split(in, "/")
    for i := range segments {
        // 跳过规则：同加密方向
        if !c.dirNameEncrypt && i != (len(segments)-1) {
            continue
        }

        // 版本处理：同加密方向，只在最后段剥离
        hasVersion := false
        var t time.Time
        if i == (len(segments)-1) && version.Match(segments[i]) {
            t, s := version.Remove(segments[i])
            if s != segments[i] {
                segments[i] = s
                hasVersion = true
            }
        }

        // 模式分流
        var err error
        if c.mode == NameEncryptionStandard {
            segments[i], err = c.decryptSegment(segments[i])
        } else {
            segments[i], err = c.deobfuscateSegment(segments[i])
        }
        if err != nil {
            return "", err
        }

        // 版本回填
        if hasVersion {
            segments[i] = version.Add(segments[i], t)
        }
    }
    return strings.Join(segments, "/"), nil
}
```

### 3.4 分段跳过规则的真值表

`dirNameEncrypt` 开关 + 当前段是否为最后一段，共同决定该段是否被加密：

| `dirNameEncrypt` | 当前段 | 是否加密/解密 | 说明 |
|:---:|:---:|:---:|------|
| `true` | 任意段 | ✅ 加密 | 目录名也加密 |
| `false` | 最后一段（文件名） | ✅ 加密 | 仅文件名加密 |
| `false` | 中间段（目录名） | ❌ 跳过 | 目录名保持原样 |

> **关键**：`dirNameEncrypt` 只在 standard/obfuscate 模式下有效；off 模式下 `EncryptDirName` 直接返回原样。

### 3.5 后缀规则

后缀（`encryptedSuffix`）由 `setEncryptedSuffix()`（`backend/crypt/cipher.go`）设置，初始默认值 `".bin"`（`newCipher` 中赋值）：

| 配置值 | 结果 `encryptedSuffix` | 说明 |
|--------|----------------------|------|
| `".bin"`（默认） | `".bin"` | 标准后缀 |
| `".txt"` 等自定义 | `".txt"` | 自定义后缀 |
| 不以 `.` 开头（如 `txt`） | `.txt`（自动补点） | 触发 `ErrorSuffixMissingDot` 日志后补齐 |
| `"none"` | `""` | 空后缀（path 长度敏感时可用） |

后缀**仅在 `Off` 模式下**通过 `EncryptFileName` 追加；目录名即使在 `Off` 模式下也不追加。standard/obfuscate 模式下根本不使用后缀（加密段已经不可读）。

### 3.6 版本字符串处理

`version` 包（`lib/version`）支持类似 `file-2024-01-01.txt` 的时间戳版本后缀。处理规则：

1. **只在最后一段（文件名）处理**，目录段不处理版本。
2. 加密前：用 `version.Match()` 检测，`version.Remove()` 剥离版本字符串得到基础名和时间戳。
3. 加密/混淆**基础名**。
4. 加密后：用 `version.Add()` 把版本字符串重新附加到加密后的结果上。

这样版本信息**保持明文可读**，但基础文件名仍被加密。解密方向对称执行。

### 3.7 标准加密段 — `encryptSegment` / `decryptSegment`

`encryptSegment()`（`backend/crypt/cipher.go`）：

```go
func (c *Cipher) encryptSegment(plaintext string) string {
    paddedPlaintext := pkcs7.Pad(nameCipherBlockSize, []byte(plaintext)) // PKCS#7 填充到 16 字节倍数
    ciphertext := eme.Transform(c.block, c.nameTweak[:], paddedPlaintext, eme.DirectionEncrypt) // EME-AES 加密
    return c.fileNameEnc.EncodeToString(ciphertext) // Base32/Base64/Base32768 编码
}
```

`decryptSegment()`（`backend/crypt/cipher.go`）：

```go
func (c *Cipher) decryptSegment(ciphertext string) (string, error) {
    rawCiphertext, err := c.fileNameEnc.DecodeString(ciphertext) // 解码
    // ... 长度校验（必须是 16 的倍数、≤2048 字节）...
    paddedPlaintext := eme.Transform(c.block, c.nameTweak[:], rawCiphertext, eme.DirectionDecrypt) // EME-AES 解密
    plaintext, err := pkcs7.Unpad(nameCipherBlockSize, paddedPlaintext) // 去填充
    return string(plaintext), err
}
```

**EME (ECB-Mix-ECB)** 是一种宽块加密模式：
- **确定性加密**：相同明文 → 相同密文（保证文件名一致性，便于去重和定位）
- 相同前缀的明文不会产生相同前缀的密文（避免泄漏目录结构）
- 基于 AES，使用 `nameTweak` 作为调整量

### 3.8 文件名编码方式 — `fileNameEncoding`（`backend/crypt/cipher.go`）

| 编码方式 | 适用场景 |
|---------|---------|
| `base32` | 大小写不敏感的 remote（默认） |
| `base64` | 大小写敏感的 remote，文件名更短 |
| `base32768` | 按 Unicode 字符计数长度的 remote（如 OneDrive、Dropbox） |

### 3.9 混淆模式段 — `obfuscateSegment` / `deobfuscateSegment`

`obfuscateSegment()`（`backend/crypt/cipher.go`）使用简单的字符旋转：

1. 计算文件名所有字符的 Unicode 码点之和，模 256 得到旋转基数
2. 加上 `nameKey` 的字节值得到实际旋转量
3. 对不同类型的字符（数字、字母、Latin-1、Unicode）应用不同的旋转规则
4. 格式：`旋转量.混淆后的文本`（前缀的数字便于反混淆时还原旋转量）
5. 非 UTF-8 字符串直接加 `!.` 前缀，不旋转

这是一种弱加密，仅用于防止文件名被直接识别，**不能提供真正的机密性**。

---

## 四、内容加密机制

### 4.1 文件格式与常量

加密后的文件结构：

```
+-----------------+-----------------+-------------------+-------------------+
|   Magic (8B)    |   Nonce (24B)   |   Block 1         |   Block 2 ...     |
|  "RCLONE\x00\x00"  |   初始随机数    | secretbox 加密块   |                   |
+-----------------+-----------------+-------------------+-------------------+
         ↑                ↑                   ↑
    文件头 32 字节        随机数        每块: 64KB 明文 + 16B 认证标签
```

常量定义（`backend/crypt/cipher.go` 常量区）：

| 常量 | 值 | 说明 |
|------|-----|------|
| `fileMagic` | `"RCLONE\x00\x00"` | 6 字母 + 2 零字节 = **8 字节** |
| `fileMagicSize` | `len(fileMagic)` = **8** | 魔术字节长度 |
| `fileNonceSize` | **24** | NACL secretbox 要求的 nonce 大小 |
| `fileHeaderSize` | `8 + 24` = **32** | 文件头总大小（magic + nonce） |
| `blockDataSize` | `64 * 1024` = **65536** | 每个块的明文大小（64KB） |
| `blockHeaderSize` | `secretbox.Overhead` = **16** | secretbox 认证标签 overhead |
| `blockSize` | `16 + 65536` = **65552** | 每个块的密文总大小 |

> **注意**：`"RCLONE\x00\x00"` 是 8 字节（`R` `C` `L` `O` `N` `E` + `\x00` `\x00`），不是 7 字节。`fileHeaderSize = 8 + 24 = 32` 字节。

### 4.2 加密流 — encrypter

`encrypter`（`backend/crypt/cipher.go`）实现 `io.Reader` 接口，对输入流进行流式加密：

```go
type encrypter struct {
    in       io.Reader        // 底层输入流（明文）
    c        *Cipher          // 密码器引用
    nonce    nonce            // 当前块的 nonce，每块递增
    buf      *[blockSize]byte // 加密输出缓冲区（65552B，来自 sync.Pool）
    readBuf  *[blockSize]byte // 明文读取缓冲区（65552B，来自 sync.Pool，使用时只截取前 65536B）
    bufIndex int              // 当前读取位置
    bufSize  int              // 缓冲区有效数据大小
    // ...
}
```

> **关键说明**：`buf` 和 `readBuf` **都是 `*[blockSize]byte`（65552 字节）**，两者都由同一个 `Cipher.buffers`（`sync.Pool`）通过 `getBlock()` 分配。只是用途不同，使用切片截取来控制实际读写长度。

#### 缓冲区池 — `Cipher.buffers` / `getBlock()` / `putBlock()`（`backend/crypt/cipher.go`）

```go
// Cipher 初始化时创建 sync.Pool，每个元素是 65552 字节的数组指针
c.buffers.New = func() any {
    return new([blockSize]byte) // blockSize = 65552
}

// 从池里取一个 65552B 块
func (c *Cipher) getBlock() *[blockSize]byte {
    return c.buffers.Get().(*[blockSize]byte)
}

// 把 65552B 块归还池
func (c *Cipher) putBlock(buf *[blockSize]byte) {
    c.buffers.Put(buf)
}
```

#### 加密流程 — `encrypter.Read()`（`backend/crypt/cipher.go`）

```go
func (fh *encrypter) Read(p []byte) (n int, err error) {
    if fh.bufIndex >= fh.bufSize {
        // 1. 从 65552B 的 readBuf 中**只截取前 65536B** 来读明文
        //    （blockDataSize = 64KB，一个块最多存这么多明文）
        readBuf := (*fh.readBuf)[:blockDataSize]
        n, err = readers.ReadFill(fh.in, readBuf)
        if n == 0 {
            return fh.finish(err) // EOF，finish() 会把 buf 和 readBuf 都归还 Pool
        }
        // 2. 使用 secretbox 加密（XSalsa20 + Poly1305 认证加密）
        //    输出到 buf 的全部 65552B 空间（实际写 16+n 字节）
        secretbox.Seal((*fh.buf)[:0], readBuf[:n], fh.nonce.pointer(), &fh.c.dataKey)
        fh.bufIndex = 0
        fh.bufSize = blockHeaderSize + n // 16 + n 字节
        // 3. nonce 递增（每个块用不同 nonce）
        fh.nonce.increment()
    }
    // 4. 从 buf 中拷贝给调用者
    n = copy(p, (*fh.buf)[fh.bufIndex:fh.bufSize])
    fh.bufIndex += n
    return n, nil
}
```

#### `encrypter.finish()` — 缓冲区归还

```go
func (fh *encrypter) finish(err error) (int, error) {
    // ...
    fh.c.putBlock(fh.buf)     // 归还 65552B 输出块
    fh.buf = nil
    fh.c.putBlock(fh.readBuf) // 归还 65552B 输入块
    fh.readBuf = nil
    return 0, err
}
```

**NACL secretbox** 使用：
- 算法：XSalsa20 流加密 + Poly1305 消息认证码
- 提供**认证加密**（authenticated encryption），可检测篡改
- 每个块有独立的 nonce，nonce 从文件头的初始值逐块递增

### 4.3 解密流 — decrypter

`decrypter`（`backend/crypt/cipher.go`）实现 `io.ReadCloser` + `io.Seeker` 接口：

```go
type decrypter struct {
    rc           io.ReadCloser     // 底层输入流（密文）
    nonce        nonce             // 当前块的 nonce
    initialNonce nonce             // 初始 nonce（用于 seek）
    c            *Cipher
    buf          *[blockSize]byte  // 解密输出缓冲区（65552B，来自 sync.Pool，存 65536B 明文）
    readBuf      *[blockSize]byte  // 密文读取缓冲区（65552B，来自 sync.Pool，读 65552B 密文块）
    bufIndex     int
    bufSize      int
    limit        int64             // 读取限制（用于 Range 请求）
    open         OpenRangeSeek     // 用于重新打开底层流（seek 时）
}
```

> **关键说明**：`buf` 和 `readBuf` **也都是 `*[blockSize]byte`（65552 字节）**，与 encrypter 共用同一个 `sync.Pool`。不同场景下截取不同长度。

#### 初始化 — `newDecrypter()`（`backend/crypt/cipher.go`）

```go
func (c *Cipher) newDecrypter(rc io.ReadCloser) (*decrypter, error) {
    // 1. 从 65552B 的 readBuf 中截取前 32B 读文件头
    readBuf := (*fh.readBuf)[:fileHeaderSize] // 截取 32 字节
    n, err := readers.ReadFill(fh.rc, readBuf)
    // 2. 校验魔术字节（readBuf 前 8 字节）
    if !bytes.Equal(readBuf[:fileMagicSize], fileMagicBytes) {
        return nil, ErrorEncryptedBadMagic
    }
    // 3. 获取初始 nonce（readBuf 从第 8 字节开始的 24 字节）
    fh.nonce.fromBuf(readBuf[fileMagicSize:]) // [8:32]，共 24 字节
    fh.initialNonce = fh.nonce
    return fh, nil
}
```

> `readBuf` 的复用：32B 文件头读完后，同一块 65552B 内存后续会被当作密文块读取缓冲区再次使用，底层没有重新分配。

#### 解密流程 — `decrypter.Read()`（`backend/crypt/cipher.go`）

```go
func (fh *decrypter) Read(p []byte) (n int, err error) {
    if fh.bufIndex >= fh.bufSize {
        err = fh.fillBuffer() // 读取并解密一个块
        if err != nil {
            return 0, fh.finish(err) // finish() 把 buf 和 readBuf 归还 Pool
        }
    }
    // 从 buf 中拷贝，考虑 limit 限制
    toCopy := fh.bufSize - fh.bufIndex
    if fh.limit >= 0 && fh.limit < int64(toCopy) {
        toCopy = int(fh.limit)
    }
    n = copy(p, (*fh.buf)[fh.bufIndex:fh.bufIndex+toCopy])
    // ...
    return n, nil
}
```

#### 块解密 — `fillBuffer()`（`backend/crypt/cipher.go`）

```go
func (fh *decrypter) fillBuffer() (err error) {
    // 1. 用 readBuf 的**全部 65552B** 读取一个密文块
    n, err := readers.ReadFill(fh.rc, (*readBuf)[:]) // [:65552]
    // 2. secretbox 解密 + 认证：
    //    - 输入：readBuf[:n]（16B 认证头 + 密文）
    //    - 输出：buf 的前 (n - 16)B 明文
    _, ok := secretbox.Open((*fh.buf)[:0], (*readBuf)[:n], fh.nonce.pointer(), &fh.c.dataKey)
    if !ok {
        if !fh.c.passBadBlocks {
            return ErrorEncryptedBadBlock // 认证失败
        }
        // passBadBlocks 模式：用零填充损坏的块
        for i := range (*fh.buf)[:n] { fh.buf[i] = 0 }
    }
    fh.bufSize = n - blockHeaderSize // 明文大小 = 密文大小 - 16
    fh.nonce.increment() // nonce 递增
    return nil
}
```

#### 两结构体的缓冲区对比总结

| 字段 | encrypter 中 | decrypter 中 | 统一底层类型 | 来源 |
|------|-------------|-------------|-------------|------|
| `buf` | 存密文（头32B初始化 + 后续每块16+n B输出） | 存明文（每块解密后最多65536B） | `*[blockSize]byte`（65552B） | `Cipher.buffers` sync.Pool |
| `readBuf` | 存明文，截取 `[:65536]` 读；初始化时不参与 | 先截 `[:32]` 读文件头，再用全 `[:65552]` 读密文块 | `*[blockSize]byte`（65552B） | `Cipher.buffers` sync.Pool |

### 4.4 随机访问（Seek）

`RangeSeek()`（`backend/crypt/cipher.go`）支持随机访问：

```go
func (fh *decrypter) RangeSeek(ctx context.Context, offset int64, whence int, limit int64) (int64, error) {
    // 1. 计算底层文件的偏移和块号
    underlyingOffset, underlyingLimit, discard, blocks := calculateUnderlying(offset, limit)

    // 2. 重置 nonce 到目标块（从初始 nonce 开始加 blocks 次）
    fh.nonce = fh.initialNonce
    fh.nonce.add(uint64(blocks))

    // 3. 底层流定位（如果支持 RangeSeeker 则直接 seek，否则重新打开）
    if do, ok := fh.rc.(fs.RangeSeeker); ok {
        _, err := do.RangeSeek(ctx, underlyingOffset, 0, underlyingLimit)
    } else {
        _ = fh.rc.Close()
        rc, err := fh.open(ctx, underlyingOffset, underlyingLimit) // 重新打开
        fh.rc = rc
    }

    // 4. 读取并解密第一块，丢弃块内偏移部分
    err := fh.fillBuffer()
    fh.bufIndex = int(discard) // 跳过块内偏移

    fh.limit = limit
    return offset, nil
}
```

#### 偏移量换算 — `calculateUnderlying()`（`backend/crypt/cipher.go`）

```
明文偏移 → 密文偏移换算原理：

明文:   |  块 0 (64KB)  |  块 1 (64KB)  |  ...
密文:   |Hdr| 块 0 (64KB+16B) | 块 1 (64KB+16B) | ...
        ↑
   fileHeaderSize (32B)

blocks = offset / blockDataSize        // 完整块数
discard = offset % blockDataSize       // 块内偏移（需要丢弃的字节数）
underlyingOffset = fileHeaderSize + blocks * (blockHeaderSize + blockDataSize)
                  = 32 + blocks * 65552
```

**数值校验样例**：

| 明文 offset | blocks | discard | 密文 underlyingOffset | 说明 |
|------------:|-------:|--------:|---------------------:|------|
| 0 | 0 | 0 | 32 | 文件开头，跳过 32 字节文件头 |
| 1 | 0 | 1 | 32 | 第 0 块内偏移 1 字节 |
| 65535 | 0 | 65535 | 32 | 第 0 块最后一个字节 |
| 65536 | 1 | 0 | 65584 | 第 1 块起始（32 + 65552） |
| 65537 | 1 | 1 | 65584 | 第 1 块内偏移 1 字节 |
| 131072 | 2 | 0 | 131136 | 第 2 块起始（32 + 2×65552） |
| 70000 | 1 | 4464 | 65584 | 第 1 块内偏移 4464 字节（70000 - 65536） |

> 验证：`65552 = blockHeaderSize + blockDataSize = 16 + 65536` ✓
> 验证：`65584 = 32 + 65552` ✓
> 验证：`131136 = 32 + 2 × 65552 = 32 + 131104` ✓

### 4.5 大小换算

#### `EncryptedSize()` — 明文大小 → 密文大小（`backend/crypt/cipher.go`）

```go
func (c *Cipher) EncryptedSize(size int64) int64 {
    blocks, residue := size/blockDataSize, size%blockDataSize
    encryptedSize := int64(fileHeaderSize) + blocks*(blockHeaderSize+blockDataSize)
    if residue != 0 {
        encryptedSize += blockHeaderSize + residue
    }
    return encryptedSize
}
```

#### `DecryptedSize()` — 密文大小 → 明文大小（`backend/crypt/cipher.go`）

```go
func (c *Cipher) DecryptedSize(size int64) (int64, error) {
    size -= int64(fileHeaderSize) // 先去掉文件头
    if size < 0 {
        return 0, ErrorEncryptedFileTooShort
    }
    blocks, residue := size/blockSize, size%blockSize // blockSize = 65552
    decryptedSize := blocks * blockDataSize           // 完整块的明文
    if residue != 0 {
        residue -= blockHeaderSize                    // 尾块去掉 16 字节认证头
        if residue <= 0 {
            return 0, ErrorEncryptedFileBadHeader
        }
    }
    decryptedSize += residue
    return decryptedSize, nil
}
```

**数值校验样例**：

| 明文大小 | 密文大小（EncryptedSize） | 反向解密（DecryptedSize） | 说明 |
|---------:|-------------------------:|-------------------------:|------|
| 0 | 32 | 0 | 空文件只有文件头 |
| 1 | 49 | 1 | 32 + 16 + 1 = 49 |
| 100 | 148 | 100 | 32 + 16 + 100 = 148 |
| 65536 (64KB) | 65584 | 65536 | 32 + 65552 = 65584（刚好 1 块） |
| 65537 | 65601 | 65537 | 32 + 65552 + 16 + 1 = 65601（1 块 + 1 字节尾块） |
| 131072 (128KB) | 131136 | 131072 | 32 + 2×65552 = 131136（刚好 2 块） |

> 验证：所有样例的 DecryptedSize(EncryptedSize(x)) == x ✓

---

## 五、FS 包装层与底层 Remote 的配合

### 5.1 Fs 结构体

`Fs`（`backend/crypt/crypt.go`）包装底层 `fs.Fs`：

```go
type Fs struct {
    fs.Fs                // 嵌入底层 Fs（匿名嵌入，继承所有方法）
    name     string
    root     string
    opt      Options
    features *fs.Features // 特性掩码
    cipher   *Cipher     // 密码器实例
}
```

### 5.2 初始化流程 — `NewFs()`

`NewFs()`（`backend/crypt/crypt.go`）初始化步骤：

1. 解析配置，创建 `Cipher` 实例
2. 加密根路径，用加密后的路径创建底层 `Fs`
3. 创建 `Fs` 包装对象
4. 设置 `features`（与底层 Fs 的特性做 AND 掩码）

```go
func NewFs(ctx context.Context, name, rpath string, m configmap.Mapper) (fs.Fs, error) {
    // ... 创建 cipher ...
    remote := opt.Remote

    // 先尝试作为文件路径加密
    remotePath := fspath.JoinRootPath(remote, cipher.EncryptFileName(rpath))
    wrappedFs, err = cache.Get(ctx, remotePath)

    // 如果不是文件，尝试作为目录加密
    if err != fs.ErrorIsFile {
        remotePath = fspath.JoinRootPath(remote, cipher.EncryptDirName(rpath))
        wrappedFs, err = cache.Get(ctx, remotePath)
    }
    // ...
}
```

### 5.3 列表操作 — List / ListP

`ListP()`（`backend/crypt/crypt.go`）是典型的"加密路径调用 → 解密结果返回"模式：

```go
func (f *Fs) ListP(ctx context.Context, dir string, callback fs.ListRCallback) error {
    wrappedCallback := func(entries fs.DirEntries) error {
        // 2. 对返回的条目逐个解密文件名
        entries, err := f.encryptEntries(ctx, entries)
        if err != nil { return err }
        return callback(entries)
    }
    // 1. 用加密后的目录名调用底层 ListP
    encryptedDir := f.cipher.EncryptDirName(dir)
    return listP(ctx, encryptedDir, wrappedCallback)
}
```

条目解密在 `encryptEntries()`（`backend/crypt/crypt.go`）中完成：
- 文件对象 → 调用 `DecryptFileName()` 解密文件名
- 目录对象 → 调用 `DecryptDirName()` 解密目录名
- 解密失败的条目根据 `strict_names` 配置决定是跳过还是报错

### 5.4 上传操作 — Put / PutStream

`put()`（`backend/crypt/crypt.go`）封装了上传流程：

```go
func (f *Fs) put(ctx context.Context, in io.Reader, src fs.ObjectInfo,
    options []fs.OpenOption, put putFn) (fs.Object, error) {

    if f.opt.NoDataEncryption {
        // 不加密数据，直接传递
        return put(ctx, in, f.newObjectInfo(src, nonce{}), options...)
    }

    // 1. 包装输入流为加密流
    wrappedIn, encrypter, err := f.cipher.encryptData(in)

    // 2. （可选）计算加密数据的 hash
    // 用 TeeReader 同时加密和算 hash

    // 3. 调用底层 Put，传入加密后的数据流和 ObjectInfo
    // ObjectInfo 会加密文件名、调整大小
    o, err := put(ctx, wrappedIn, f.newObjectInfo(src, encrypter.nonce), options...)

    // 4. 校验 hash（如果启用）
    // ...

    return f.newObject(o), nil // 包装返回的 Object
}
```

### 5.5 Object 包装

`Object`（`backend/crypt/crypt.go`）包装底层 `fs.Object`，透明地解密文件名和内容：

```go
type Object struct {
    fs.Object  // 底层对象
    f *Fs      // 所属 Fs
}
```

关键方法（均在 `backend/crypt/crypt.go`）：
- `Remote()`：解密文件名后返回
- `Size()`：转换为明文大小
- `Hash()`：返回不支持（因为加密后 hash 无意义）
- `Open()`：返回解密流
- `Update()`：加密上传

### 5.6 ObjectInfo 包装

`ObjectInfo`（`backend/crypt/crypt.go`）用于上传时的源对象信息：

```go
type ObjectInfo struct {
    fs.ObjectInfo
    f     *Fs
    nonce nonce // 加密使用的 nonce（用于 hash 计算）
}
```

关键方法（均在 `backend/crypt/crypt.go`）：
- `Remote()`：加密文件名
- `Size()`：转换为密文大小

### 5.7 打开文件 — `Object.Open()`

`Open()`（`backend/crypt/crypt.go`）是内容解密的入口：

```go
func (o *Object) Open(ctx context.Context, options ...fs.OpenOption) (rc io.ReadCloser, err error) {
    var offset, limit int64 = 0, -1
    for _, option := range options {
        switch x := option.(type) {
        case *fs.SeekOption:
            offset = x.Offset
        case *fs.RangeOption:
            offset, limit = x.Decode(o.Size())
        default:
            openOptions = append(openOptions, option)
        }
    }

    // 使用 DecryptDataSeek，传入一个回调函数用于打开底层流
    rc, err = o.f.cipher.DecryptDataSeek(ctx,
        func(ctx context.Context, underlyingOffset, underlyingLimit int64) (io.ReadCloser, error) {
            // 回调：根据计算出的底层偏移和限制打开文件
            // ... 构造 RangeOption ...
            return o.Object.Open(ctx, newOpenOptions...)
        }, offset, limit)

    return rc, nil
}
```

### 5.8 目录操作汇总

所有目录操作都遵循**加密路径 → 调用底层 → 返回结果**的模式（均在 `backend/crypt/crypt.go`）：

| 操作 | 加密的内容 | 说明 |
|------|-----------|------|
| `Mkdir` | 目录名 | 创建加密目录 |
| `Rmdir` | 目录名 | 删除加密目录 |
| `Purge` | 目录名 | 清空加密目录 |
| `Copy` | 目标文件名 | 服务端复制（同 remote） |
| `Move` | 目标文件名 | 服务端移动（同 remote） |
| `DirMove` | 源/目标目录名 | 目录级服务端移动 |

---

## 六、完整调用链示例

### 6.1 下载文件流程（带 Range 请求）

```
用户调用: fs.Object.Open(ctx, &fs.RangeOption{Start: offset, End: end})
    ↓
crypt.Object.Open(ctx, options)
    ├─ 解析 SeekOption/RangeOption → offset, limit
    └─ cipher.DecryptDataSeek(ctx, openFn, offset, limit)
        └─ decrypter（实现 ReadSeekCloser 接口）
            ├─ 初始化: newDecrypterSeek()
            │   ├─ 调用 openFn(ctx, 0, 32) 读取文件头（32 字节）
            │   ├─ 校验 magic（前 8 字节）→ 读 nonce（后 24 字节）→ 保存 initialNonce
            │   └─ 如果 offset > 0，调用 RangeSeek()
            │       ├─ calculateUnderlying() 计算 blocks, discard, underlyingOffset
            │       ├─ nonce = initialNonce + blocks
            │       └─ 重新打开底层流（或直接 seek）到底层偏移
            └─ Read() 时:
                ├─ fillBuffer() 读取一个密文块（最多 65552 字节）
                ├─ secretbox.Open() 解密 + 认证（16 字节标签）
                ├─ nonce 递增
                └─ 返回明文字节（最多 65536 字节）
```

### 6.2 上传文件流程

```
用户调用: fs.Put(ctx, in, src)
    ↓
crypt.Fs.put(ctx, in, src, options, f.Fs.Put)
    ├─ cipher.encryptData(in) → encrypter
    │   └─ 内部 nonce 随机生成（24 字节）
    ├─ （可选）TeeReader 计算加密数据的 hash
    ├─ newObjectInfo(src, encrypter.nonce)
    │   ├─ Remote() → 加密文件名
    │   └─ Size() → EncryptedSize(src.Size())
    └─ 底层 Put(ctx, wrappedIn, encryptedObjInfo)
        └─ 上传加密后的数据流（文件头 32B + 逐块加密）
```

### 6.3 列目录流程

```
用户调用: fs.List(ctx, dir)
    ↓
crypt.Fs.ListP(ctx, dir, callback)
    ├─ encryptedDir = cipher.EncryptDirName(dir)
    ├─ 底层 Fs.ListP(ctx, encryptedDir, wrappedCallback)
    └─ wrappedCallback(entries):
        └─ encryptEntries(ctx, entries)
            ├─ 对每个文件: DecryptFileName + newObject
            └─ 对每个目录: DecryptDirName + newDir
```

---

## 七、设计特点总结

1. **透明包装模式**：crypt 完全实现 `fs.Fs` 接口，对上层透明。底层可以是任何其他 remote。

2. **双重加密体系**：
   - 文件名：EME-AES 确定性加密（保证相同文件名映射一致）
   - 文件内容：NACL secretbox 流式认证加密（每个块独立 nonce，支持 tamper detection）

3. **分块加密**：64KB 一块，每块 16 字节认证标签，支持随机访问（seek 到块边界再解密）。

4. **路径分段加密**：每个路径段单独加密，保持目录树结构。可配置是否加密目录名。

5. **可配置性**：文件名加密模式（standard/obfuscate/off）、目录名加密、数据加密、后缀、编码方式（base32/base64/base32768）等均可配置。

6. **统一缓冲区池**：`Cipher.buffers` 使用单一 `sync.Pool` 管理所有块，元素类型统一为 `*[blockSize]byte`（65552 字节）。`encrypter` 和 `decrypter` 的 `buf` / `readBuf` 四个字段**都从这个池分配**，只是用途不同，通过切片截取（`[:blockDataSize]` / `[:fileHeaderSize]` / `[:]`）适配不同场景。流结束时通过 `finish()` 把四个块统一归还，大幅减少高并发场景下的 GC 压力。

7. **随机访问支持**：`decrypter` 实现 `RangeSeeker` 接口，支持 HTTP Range 请求场景。通过 `calculateUnderlying()` 计算底层偏移，从 `initialNonce` 按块数重放 nonce 即可实现随机读取。
