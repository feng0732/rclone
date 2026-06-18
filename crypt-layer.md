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
| `NameEncryptionOff` | 不加密，仅添加 `.bin` 后缀（可配置） |
| `NameEncryptionStandard` | 标准加密，使用 EME-AES |
| `NameEncryptionObfuscated` | 简单混淆，字符旋转 |

### 3.2 标准加密模式（Standard）

#### 加密流程 — `encryptSegment()`（`backend/crypt/cipher.go`）

```go
func (c *Cipher) encryptSegment(plaintext string) string {
    paddedPlaintext := pkcs7.Pad(nameCipherBlockSize, []byte(plaintext)) // PKCS#7 填充到 16 字节倍数
    ciphertext := eme.Transform(c.block, c.nameTweak[:], paddedPlaintext, eme.DirectionEncrypt) // EME-AES 加密
    return c.fileNameEnc.EncodeToString(ciphertext) // Base32/Base64/Base32768 编码
}
```

**EME (ECB-Mix-ECB)** 是一种宽块加密模式，特点：
- **确定性加密**：相同明文 → 相同密文（保证文件名一致性）
- 相同前缀的明文不会产生相同前缀的密文
- 基于 AES，使用 `nameTweak` 作为调整量

#### 解密流程 — `decryptSegment()`（`backend/crypt/cipher.go`）

```go
func (c *Cipher) decryptSegment(ciphertext string) (string, error) {
    rawCiphertext, err := c.fileNameEnc.DecodeString(ciphertext) // 解码
    // ... 长度校验 ...
    paddedPlaintext := eme.Transform(c.block, c.nameTweak[:], rawCiphertext, eme.DirectionDecrypt) // EME-AES 解密
    plaintext, err := pkcs7.Unpad(nameCipherBlockSize, paddedPlaintext) // 去填充
    return string(plaintext), err
}
```

#### 文件名编码方式 — `fileNameEncoding`（`backend/crypt/cipher.go`）

| 编码方式 | 适用场景 |
|---------|---------|
| `base32` | 大小写不敏感的 remote（默认） |
| `base64` | 大小写敏感的 remote，文件名更短 |
| `base32768` | 按 Unicode 字符计数长度的 remote（如 OneDrive、Dropbox） |

### 3.3 混淆模式（Obfuscated）

`obfuscateSegment()`（`backend/crypt/cipher.go`）使用简单的字符旋转：

1. 计算文件名所有字符的 Unicode 码点之和，模 256 得到旋转基数
2. 加上 `nameKey` 的字节值得到实际旋转量
3. 对不同类型的字符（数字、字母、Latin-1、Unicode）应用不同的旋转规则
4. 格式：`旋转量.混淆后的文本`

这是一种弱加密，仅用于防止文件名被直接识别。

### 3.4 路径分段加密 — `encryptFileName()`（`backend/crypt/cipher.go`）

```go
func (c *Cipher) encryptFileName(in string) string {
    segments := strings.Split(in, "/") // 按路径分隔
    for i := range segments {
        if !c.dirNameEncrypt && i != (len(segments)-1) {
            continue // 可配置是否加密目录名
        }
        // 处理版本后缀（如 file.txt → 加密部分 + 版本后缀）
        if version.Match(segments[i]) { /* strip version */ }

        segments[i] = c.encryptSegment(segments[i]) // 每段单独加密

        // 加回版本后缀
    }
    return strings.Join(segments, "/")
}
```

**关键点**：每个路径段单独加密，保证目录结构仍然是树形的。

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
    in       io.Reader          // 底层输入流（明文）
    c        *Cipher            // 密码器引用
    nonce    nonce              // 当前块的 nonce，每块递增
    buf      *[blockSize]byte   // 加密输出缓冲区
    readBuf  *[blockDataSize]byte // 明文读取缓冲区
    bufIndex int                // 当前读取位置
    bufSize  int                // 缓冲区有效数据大小
    // ...
}
```

#### 加密流程 — `encrypter.Read()`（`backend/crypt/cipher.go`）

```go
func (fh *encrypter) Read(p []byte) (n int, err error) {
    if fh.bufIndex >= fh.bufSize {
        // 1. 读取一块明文数据（最多 64KB）
        n, err = readers.ReadFill(fh.in, readBuf[:blockDataSize])
        if n == 0 {
            return fh.finish(err) // EOF
        }
        // 2. 使用 secretbox 加密（XSalsa20 + Poly1305 认证加密）
        secretbox.Seal((*fh.buf)[:0], readBuf[:n], fh.nonce.pointer(), &fh.c.dataKey)
        fh.bufIndex = 0
        fh.bufSize = blockHeaderSize + n // 16 + n 字节
        // 3. nonce 递增（每个块用不同 nonce）
        fh.nonce.increment()
    }
    // 4. 从缓冲区拷贝给调用者
    n = copy(p, (*fh.buf)[fh.bufIndex:fh.bufSize])
    fh.bufIndex += n
    return n, nil
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
    rc           io.ReadCloser   // 底层输入流（密文）
    nonce        nonce           // 当前块的 nonce
    initialNonce nonce           // 初始 nonce（用于 seek）
    c            *Cipher
    buf          *[blockSize]byte    // 解密输出缓冲区
    readBuf      *[blockSize]byte    // 密文读取缓冲区
    bufIndex     int
    bufSize      int
    limit        int64           // 读取限制（用于 Range 请求）
    open         OpenRangeSeek   // 用于重新打开底层流（seek 时）
}
```

#### 初始化 — `newDecrypter()`（`backend/crypt/cipher.go`）

```go
func (c *Cipher) newDecrypter(rc io.ReadCloser) (*decrypter, error) {
    // 1. 读取文件头（magic + nonce）= 32 字节
    readBuf := (*fh.readBuf)[:fileHeaderSize] // 32 字节
    n, err := readers.ReadFill(fh.rc, readBuf)
    // 2. 校验魔术字节
    if !bytes.Equal(readBuf[:fileMagicSize], fileMagicBytes) { // 前 8 字节
        return nil, ErrorEncryptedBadMagic
    }
    // 3. 获取初始 nonce（从第 8 字节开始的 24 字节）
    fh.nonce.fromBuf(readBuf[fileMagicSize:]) // 偏移 8，读 24 字节
    fh.initialNonce = fh.nonce
    return fh, nil
}
```

#### 解密流程 — `decrypter.Read()`（`backend/crypt/cipher.go`）

```go
func (fh *decrypter) Read(p []byte) (n int, err error) {
    if fh.bufIndex >= fh.bufSize {
        err = fh.fillBuffer() // 读取并解密一个块
        if err != nil {
            return 0, fh.finish(err)
        }
    }
    // 从缓冲区拷贝，考虑 limit 限制
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
    // 1. 读取一个密文块（最多 65552 字节 = 16 + 65536）
    n, err := readers.ReadFill(fh.rc, (*readBuf)[:])
    // 2. secretbox 解密 + 认证
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

6. **缓冲区池**：使用 `sync.Pool` 复用加密/解密缓冲区（`blockSize` = 65552 字节），减少 GC 压力。

7. **随机访问支持**：`decrypter` 实现 `RangeSeeker` 接口，支持 HTTP Range 请求场景。通过 `calculateUnderlying()` 计算底层偏移，重新定位 nonce 即可实现随机读取。
