# WebDAV Backend 代码分析

本文档围绕 rclone 的 WebDAV backend 实现，深入分析三大核心机制：**远端能力探测**、**属性读取**、**上传覆盖语义**，特别对 TUS 协议的断点续传和同名覆盖做代码级对照分析。

---

## 一、代码文件总览

| 文件（仓库相对路径） | 职责 |
|----------------------|------|
| [backend/webdav/webdav.go](backend/webdav/webdav.go) | 主文件，Fs/Object 结构定义，核心方法（列表、上传、属性、copy/move 等） |
| [backend/webdav/api/types.go](backend/webdav/api/types.go) | API 类型定义：Prop、Response、Multistatus、Error、Time、Quota |
| [backend/webdav/chunking.go](backend/webdav/chunking.go) | Nextcloud 分块上传（chunked upload）逻辑 |
| [backend/webdav/tus.go](backend/webdav/tus.go) | ownCloud Infinite Scale TUS 协议上传入口 |
| [backend/webdav/tus-uploader.go](backend/webdav/tus-uploader.go) | TUS 上传器、单 chunk PATCH、主循环调度 |
| [backend/webdav/tus-upload.go](backend/webdav/tus-upload.go) | TUS Upload 结构、Reader→ReadSeeker 适配、Metadata 编码 |
| [backend/webdav/tus-errors.go](backend/webdav/tus-errors.go) | TUS 错误码定义（ErrOffsetMismatch、ErrVersionMismatch 等） |

---

## 二、远端能力探测（Capability Detection）

WebDAV 协议本身缺乏统一的能力自描述机制（RFC 4918 未定义能力发现协议），rclone 采用的是 **vendor（厂商）静态映射 + 配置参数** 的能力探测模式。

### 2.1 核心数据结构：Fs 的能力标志位

在 [Fs](backend/webdav/webdav.go#L209-L233) 结构体中定义了大量能力布尔字段：

```go
type Fs struct {
    precision          time.Duration // mod time 精度
    canStream          bool          // 是否支持流式上传（PutStream）
    canTus             bool          // 是否支持 TUS 上传协议
    useOCMtime         bool          // 是否可用 X-OC-Mtime 头设置 mtime
    propsetMtime       bool          // 是否可用 PROPPATCH 设置 mtime
    retryWithZeroDepth bool          // 是否需要在 Depth=1 失败后重试 Depth=0
    checkBeforePurge   bool          // Purge 前是否需要额外检查目录存在性
    hasOCMD5           bool          // 是否支持 ownCloud 风格 MD5 校验和
    hasOCSHA1          bool          // 是否支持 ownCloud 风格 SHA1 校验和
    hasMESHA1          bool          // 是否支持 Fastmail 风格 SHA1 校验和
    useStandardProps   bool          // PROPFIND 是否使用标准属性集（非扩展）
    canChunk           bool          // 是否支持 Nextcloud 分块上传
    chunksUploadURL    string        // Nextcloud 分块上传 URL
    // ...
}
```

### 2.2 能力配置入口：setQuirks()

在 [setQuirks()](backend/webdav/webdav.go#L631-L725) 函数中，根据 `vendor` 参数一次性设置所有能力标志位：

```go
func (f *Fs) setQuirks(ctx context.Context, vendor string) error {
    switch vendor {
    case "fastmail":
        f.canStream = true;  f.precision = time.Second
        f.useOCMtime = true; f.hasMESHA1 = true
    case "owncloud":
        f.canStream = true;  f.precision = time.Second
        f.useOCMtime = true; f.propsetMtime = true
        f.hasOCMD5 = true;   f.hasOCSHA1 = true
    case "infinitescale":
        f.precision = time.Second;    f.useOCMtime = true
        f.propsetMtime = true;        f.hasOCSHA1 = true
        f.canTus = true;              // ← 启用 TUS 协议
        f.opt.ChunkSize = 10 * fs.Mebi
    case "nextcloud":
        f.precision = time.Second;    f.useOCMtime = true
        f.propsetMtime = true;        f.hasOCSHA1 = true
        f.canChunk = true             // ← 启用分块上传
        f.chunksUploadURL = /*从正则解析*/
    case "sharepoint":
        f.retryWithZeroDepth = true   // ← 列表失败后降级重试
        // ... Cookie 认证（odrvcookie）+ 12h 自动续期
    case "sharepoint-ntlm":
        f.retryWithZeroDepth = true;  f.checkBeforePurge = true
    case "rclone":
        f.canStream = true;  f.precision = time.Second;  f.useOCMtime = true
    case "other":
        f.useStandardProps = true
    }
    if !f.canStream { f.features.PutStream = nil }
    return nil
}
```

### 2.3 能力映射表

| Vendor | canStream | precision | useOCMtime | propsetMtime | hasOCMD5 | hasOCSHA1 | hasMESHA1 | canTus | canChunk | retryWithZeroDepth | useStandardProps | checkBeforePurge |
|--------|-----------|-----------|------------|--------------|----------|-----------|-----------|--------|----------|--------------------|------------------|------------------|
| fastmail | ✅ | 1s | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| owncloud | ✅ | 1s | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| infinitescale | ❌ | 1s | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| nextcloud | ❌ | 1s | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| sharepoint | ❌ | - | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| sharepoint-ntlm | ❌ | - | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ |
| rclone | ✅ | 1s | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| other | ❌ | ModTimeNotSupported | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |

### 2.4 能力探测的调用链

能力探测在 **Fs 初始化阶段** 触发，调用链如下：

```
NewFs() [backend/webdav/webdav.go#L440]
  └─► f.setQuirks(ctx, opt.Vendor) [backend/webdav/webdav.go#L533]
        ├─► 根据 vendor 设置各能力标志位
        ├─► nextcloud: f.getChunksUploadURL() [backend/webdav/chunking.go#L62]
        │      └─► 正则 ^(.*)/dav/files/([^/]+) 校验 URL 格式
        ├─► sharepoint: odrvcookie.Cookies() 获取认证 Cookie + 12h 续期
        └─► !canStream → 移除 features.PutStream
```

### 2.5 动态运行时探测点

除了静态 vendor 映射外，以下情况会触发**运行时能力检查**：

1. **Nextcloud 分块上传 URL 校验**：[getChunksUploadURL()](backend/webdav/chunking.go#L62-L72) 用正则校验 endpoint 是否为 `/dav/files/USER` 格式，否则报错提示用户改用正确 URL。
2. **流式上传动态判断**：[PutStream()](backend/webdav/webdav.go#L984-L987) 只有 `canStream=true` 时才真正可用。
3. **哈希支持动态返回**：[Hashes()](backend/webdav/webdav.go#L1301-L1311) 根据 `hasOCMD5` / `hasOCSHA1` / `hasMESHA1` 动态组合支持的哈希集。

---

## 三、属性读取（Attribute Reading）

属性读取通过 WebDAV 的 **PROPFIND** 方法完成。根据不同场景分为**单对象读取**和**目录批量读取**两类。

### 3.1 属性数据模型：api.Prop

在 [Prop](backend/webdav/api/types.go#L70-L80) 中定义了所有可解析的属性字段，通过 XML 反序列化填充：

```go
type Prop struct {
    Status       []string  `xml:"DAV: status"`          // propstat 的 HTTP 状态（多值）
    Name         string    `xml:"DAV: prop>displayname,omitempty"`
    Type         *xml.Name `xml:"DAV: prop>resourcetype>collection,omitempty"`
    IsCollection *string   `xml:"DAV: prop>iscollection,omitempty"` // Microsoft 扩展
    Size         int64     `xml:"DAV: prop>getcontentlength,omitempty"`
    Modified     Time      `xml:"DAV: prop>getlastmodified,omitempty"`
    Checksums    []string  `xml:"prop>checksums>checksum,omitempty"`  // ownCloud
    Permissions  string    `xml:"prop>permissions,omitempty"`        // ownCloud
    MESha1Hex    *string   `xml:"ME: prop>sha1hex,omitempty"`        // Fastmail
}
```

**状态判定**：
- [StatusOK()](backend/webdav/api/types.go#L104-L116)：正则 `^HTTP/[\d.]+\s+2\d{2}` 判定是否**任意一个** propstat 为 2xx
- [Code()](backend/webdav/api/types.go#L86-L99)：提取**首个**状态的 HTTP 状态码

### 3.2 三种 PROPFIND 请求体

根据 vendor 能力不同，PROPFIND 的 Body 有三种策略：

#### 策略 A：ownCloud 扩展属性集（hasOCMD5 || hasOCSHA1）

[owncloudProps](backend/webdav/webdav.go#L757-L768)：
```xml
<?xml version="1.0"?>
<d:propfind xmlns:d="DAV:" xmlns:oc="http://owncloud.org/ns" xmlns:nc="http://nextcloud.org/ns">
  <d:prop>
    <d:displayname />        <d:getlastmodified />
    <d:getcontentlength />   <d:resourcetype />
    <oc:checksums />         <oc:permissions />
  </d:prop>
</d:propfind>
```

#### 策略 B：标准属性集（useStandardProps）

[standardProps](backend/webdav/webdav.go#L770-L779)：
```xml
<?xml version="1.0"?>
<d:propfind xmlns:d="DAV:">
  <d:prop>
    <d:displayname/>         <d:getlastmodified/>
    <d:getcontentlength/>    <d:resourcetype/>
  </d:prop>
</d:propfind>
```

#### 策略 C：空 Body（allprop）

当既不属于 A 也不属于 B 时，发送**空 Body**。根据 RFC 4918，空 PROPFIND body 默认使用 `allprop` 行为。

### 3.3 单对象属性读取：readMetaDataForPath()

[readMetaDataForPath()](backend/webdav/webdav.go#L343-L393) 用于读取单个路径的属性：

```go
func (f *Fs) readMetaDataForPath(ctx context.Context, path string) (info *api.Prop, err error) {
    opts := rest.Opts{
        Method: "PROPFIND",
        Path:   f.filePath(path),
        ExtraHeaders: map[string]string{ "Depth": "0" },
        CheckRedirect: rest.PreserveMethodRedirectFn,
    }
    // 根据 hasOCMD5/hasOCSHA1/useStandardProps 选择 Body ...

    var result api.Multistatus
    err = f.pacer.Call(func() (bool, error) {
        resp, err = f.srv.CallXML(ctx, &opts, nil, &result)
        return f.shouldRetry(ctx, resp, err)
    })

    // 错误映射：
    //   404 / 301 / 302 / 303  → fs.ErrorObjectNotFound
    //   len(Responses) < 1      → ErrorObjectNotFound
    //   !StatusOK() && Code!=425 → ErrorObjectNotFound
    //   itemIsDir(item)==true   → fs.ErrorIsDir

    return &item.Props, nil
}
```

### 3.4 对象懒加载机制：readMetaData()

Object 的元数据采用 **懒加载（lazy loading）** 模式：

```go
// readMetaData [backend/webdav/webdav.go#L1415-L1424]
func (o *Object) readMetaData(ctx context.Context) (err error) {
    if o.hasMetaData { return nil }  // 已加载则直接返回
    info, err := o.fs.readMetaDataForPath(ctx, o.remote)
    if err != nil { return err }
    return o.setMetaData(info)
}

// setMetaData [backend/webdav/webdav.go#L1395-L1410]
func (o *Object) setMetaData(info *api.Prop) (err error) {
    o.hasMetaData = true
    o.size = info.Size
    o.modTime = time.Time(info.Modified)
    if o.fs.hasOCMD5 || o.fs.hasOCSHA1 || o.fs.hasMESHA1 {
        hashes := info.Hashes()  // 解析 "SHA1:xxx MD5:xxx" 格式
        o.sha1 = hashes[hash.SHA1]
        o.md5  = hashes[hash.MD5]
    }
    return nil
}
```

**调用链**：在 `Size()`、`ModTime()`、`Hash()` 等方法中都会先调用 `readMetaData()` 确保元数据已加载。

### 3.5 目录列表属性读取：listAll()

[listAll()](backend/webdav/webdav.go#L789-L895) 是目录遍历的核心：

```go
func (f *Fs) listAll(ctx context.Context, dir string, directoriesOnly bool,
    filesOnly bool, depth string, fn listAllFn) (found bool, err error) {

    opts := rest.Opts{
        Method: "PROPFIND", Path: f.dirPath(dir),
        ExtraHeaders: map[string]string{ "Depth": depth }, // 默认 depth="1"
        AuthRedirect: f.opt.AuthRedirect,
    }
    // Body 选择同上 ...

    var result api.Multistatus
    err = f.pacer.Call(/* CallXML + shouldRetry */)
    if apiErr.StatusCode == 404 && f.retryWithZeroDepth && depth != "0" {
        return f.listAll(ctx, dir, directoriesOnly, filesOnly, "0", fn) // 降级重试
    }

    for i := range result.Responses {
        item := &result.Responses[i]
        isDir := itemIsDir(item)

        // URL 标准化：Join → TrimPrefix → Encode 解码 → 转 remote
        remote := path.Join(dir, subPath)
        if remote == dir { continue } // 跳过自身条目

        // 过滤：ExcludeShares(Permissions 含 'S')、ExcludeMounts(含 'M')
        if f.opt.ExcludeShares && strings.Contains(item.Props.Permissions, "S") { continue }
        if f.opt.ExcludeMounts && strings.Contains(item.Props.Permissions, "M") { continue }

        if fn(remote, isDir, &item.Props) { found = true; break }
    }
}
```

### 3.6 目录判定逻辑：itemIsDir()

[itemIsDir()](backend/webdav/webdav.go#L321-L341) 采用**双重判定**策略：

1. **标准判定**（主判据）：`resourcetype` 元素的 XML name 是否为 `DAV:collection`
2. **Microsoft 扩展判定**：若 `iscollection` 属性存在，解析其值为 `"0"/"false"` → 否，`"1"/"true"` → 是
3. **兜底判定**：都不满足则视为**非目录（文件）**——遵循 WebDAV 规范：未知资源类型默认按非集合处理

### 3.7 时间解析：自定义 Time 类型

[Time.UnmarshalXML()](backend/webdav/api/types.go#L202-L235) 支持 5 种时间格式（依次尝试，命中即停）：

| 格式常量 | 示例 | 来源 |
|----------|------|------|
| `time.RFC1123` | `Wed, 27 Sep 2017 14:28:34 GMT` | RFC 标准（主格式） |
| `time.RFC1123Z` | `Fri, 05 Jan 2018 14:14:38 +0000` | mydrive.ch |
| `time.UnixDate` | `Wed May 17 15:31:58 UTC 2017` | 某内部服务器 |
| `noZerosRFC1123` | `Fri, 7 Sep 2018 08:49:58 GMT` | #2574 无前导零 |
| `time.RFC3339` | `2018-10-31T13:57:11+01:00` | komfortcloud.de |

解析失败时使用 `Unix(0, 0)`（epoch 1970-01-01）兜底，并通过 `sync.Once` 只输出一次错误日志。

### 3.8 校验和解析：Prop.Hashes()

[Hashes()](backend/webdav/api/types.go#L118-L140) 支持两种格式：

1. **ownCloud 格式**：`<oc:checksum>SHA1:f572d... MD5:b194... ADLER32:084b...</oc:checksum>` —— 以空格分隔，按 `sha1:` / `md5:` 前缀匹配
2. **Fastmail 格式**：直接从 `ME:sha1hex` 属性的 `<string>` 指针读取

---

## 四、上传覆盖语义（Upload Overwrite Semantics）

WebDAV 的上传/复制/移动涉及多种覆盖场景，rclone 通过不同机制处理。

### 4.1 Update() 的分发：三种上传路径的选择

在 [Update()](backend/webdav/webdav.go#L1553-L1591) 中按优先级选择上传策略：

```
优先级 1：canTus == true（即 vendor = "infinitescale"）
            └─► updateViaTus()  →  TUS 协议 (POST + PATCH)
            │
优先级 2：shouldUseChunkedUpload() == true
            │   (canChunk && ChunkSize>0 && size>ChunkSize)
            └─► updateChunked() →  Nextcloud 分块（MKCOL目录 + PUT分片 + MOVE合并）
            │
优先级 3：其余所有情况
            └─► updateSimple()  →  标准 PUT（单请求）
```

---

### 4.2 普通上传覆盖：PUT 方法

#### 4.2.1 核心方法：updateSimple()

[updateSimple()](backend/webdav/webdav.go#L1616-L1652) 是最基础的 PUT 上传：

```go
func (o *Object) updateSimple(ctx context.Context, body io.Reader,
    getBody func() (io.ReadCloser, error), filePath string, size int64,
    contentType string, extraHeaders map[string]string, rootURL string,
    options ...fs.OpenOption) (err error) {

    opts := rest.Opts{
        Method: "PUT", Path: filePath,
        GetBody: getBody, Body: body,
        ContentLength: &size, ContentType: contentType,
        ExtraHeaders: extraHeaders,  // X-OC-Mtime、OC-Checksum
        RootURL: rootURL,
    }

    err = o.fs.pacer.CallNoRetry(func() (bool, error) {
        resp, err = o.fs.srv.Call(ctx, &opts)
        return o.fs.shouldRetry(ctx, resp, err)
    })

    if err != nil {
        time.Sleep(1 * time.Second)   // 等服务端整理内部状态
        _ = o.Remove(ctx)             // 清理可能的部分上传（忽略删除错误）
        return err
    }
    return nil
}
```

#### 4.2.2 PUT 的覆盖语义：隐式覆盖

WebDAV PUT 方法**默认会覆盖已存在的目标资源**（RFC 4918 §7.6）。rclone 的实现中**没有设置 `If-None-Match` 或 `Overwrite` 头**，因此：

- **目标不存在** → 创建新文件（201 Created）
- **目标已存在** → **无条件覆盖写入**（200 OK / 204 No Content）

> ⚠️ rclone 不做"仅当不存在时创建"的原子保护，覆盖判定由上层业务（如 `rclone sync` 的 check + 比较逻辑）控制。

#### 4.2.3 上传的 ModTime 和 Checksum 传递

通过 [extraHeaders()](backend/webdav/webdav.go#L1593-L1614) 设置：

| Header | 启用条件 | 值格式 |
|--------|----------|--------|
| `X-OC-Mtime` | `useOCMtime=true` | `src.ModTime().Unix()`（Unix 秒时间戳） |
| `OC-Checksum` | `hasOCSHA1=true` 且能获取 SHA1 | `SHA1:<40位hex>` |
| `OC-Checksum` | `hasOCMD5=true` 且无 SHA1 但能获取 MD5 | `MD5:<32位hex>` |

---

### 4.3 COPY/MOVE 的覆盖语义：Overwrite 头

#### 4.3.1 copyOrMove() 的处理

在 [copyOrMove()](backend/webdav/webdav.go#L1154-L1206) 中：

```go
opts := rest.Opts{
    Method: method,  // "COPY" 或 "MOVE"
    Path:   srcObj.filePath(),
    ExtraHeaders: map[string]string{
        "Destination": destinationURL.String(),
        "Overwrite":   "T",             // ← 强制允许覆盖
    },
}
if f.useOCMtime {
    opts.ExtraHeaders["X-OC-Mtime"] = fmt.Sprintf("%d", src.ModTime(ctx).Unix())
}
```

**Overwrite 头规范值**（RFC 4918 §10.6）：
- `"T"` / `"t"` → **允许覆盖**已存在的目标资源
- `"F"` / `"f"` → **禁止覆盖**，若目标存在则返回 412 Precondition Failed

rclone 一律使用 `"T"`，即 **COPY/MOVE 始终覆盖目标**。

#### 4.3.2 ModTime 二次确认逻辑

COPY/MOVE 完成后，若以下 4 个条件同时满足：
1. `useOCMtime=true`（上传时带了 `X-OC-Mtime` 头）
2. 响应头 `X-OC-Mtime` **不是** `"accepted"`（服务端未显式确认接受 mtime）
3. `propsetMtime=true`（支持 PROPPATCH 设置 mtime）
4. 重读后实际 ModTime 与预期不相等

则通过 `dstObj.SetModTime()` 发起 **PROPPATCH** 二次修正 mtime。

---

### 4.4 Nextcloud 分块上传的覆盖

在 [updateChunked()](backend/webdav/chunking.go#L78-L104) 中分三步：

#### 步骤 1：创建上传目录（清理残留）

[createChunksUploadDirectory()](backend/webdav/chunking.go#L144-L169)：
- 用 `md5(filePath())` 生成 `rclone-chunked-upload-<md5hex>` 目录名
- **先 DELETE purge 同名目录**（清理上次未成功的残留）→ 404 被视为 OK
- 再 `MKCOL` 创建新的临时上传目录

#### 步骤 2：上传数据块

[uploadChunks()](backend/webdav/chunking.go#L106-L142)：
- 按 `ChunkSize`（默认 10 MiB）顺序分块，命名 `{uploadDir}/{offset:015d}-{endOffset:015d}`
- 每个 chunk 用 `updateSimple()` PUT 到独立的 `chunksUploadURL`（如 `/dav/uploads/<user>/`）
- 用 `RepeatableLimitReaderBuffer` + `GetBody()` 支持 HTTP/2 GOAWAY 重试

#### 步骤 3：MOVE 合并（关键覆盖点）

[mergeChunks()](backend/webdav/chunking.go#L171-L198)：

```go
opts := rest.Opts{
    Method: "MOVE",
    Path:   path.Join(uploadDir, ".file"),  // 源：虚拟的 .file
    RootURL: o.fs.chunksUploadURL,
}
opts.ExtraHeaders["Destination"] = destinationURL.String()  // 目标：真实文件路径
// X-OC-Mtime 和 OC-Checksum 通过 extraHeaders() 注入
// ⚠️ 未显式设置 Overwrite 头！
```

**特殊重试**（`shouldRetryChunkMerge()`）：
- 收到 **423 LOCKED**：标记 `wasLocked=true`，指数退避（5s→10s→20s…）等待合并
- 423 后再收到 **404**：视为合并成功（Nextcloud 合并完成后删除虚拟 .file）

**覆盖语义结论—— 分两层看**：

**A. rclone 代码侧事实**：
- MOVE 的 `Destination` 就是最终目标路径
- 代码中**未显式设置 `Overwrite` 头**（对比：[copyOrMove()](backend/webdav/webdav.go#L1154-L1206) 中 COPY/MOVE 显式设了 `Overwrite: T`）
- 未设 `Overwrite` 头时，按 WebDAV 规范（RFC 4918 §9.9）默认 `Overwrite: T`

**B. 服务端行为推断**：
- Nextcloud 的 chunking 协议中，此 MOVE 操作预期会覆盖目标
- 但 Nextcloud 版本差异可能导致行为不同，rclone 代码未做额外保障

---

### 4.5 TUS 协议（ownCloud Infinite Scale）—— 代码级深度分析

#### 4.5.1 TUS 上传入口：updateViaTus()

[updateViaTus()](backend/webdav/tus.go#L20-L43)：

```go
func (o *Object) updateViaTus(ctx context.Context, in io.Reader, contentType string,
    src fs.ObjectInfo, options ...fs.OpenOption) (err error) {

    fn := filepath.Base(src.Remote())
    metadata := map[string]string{
        "filename": fn,
        "mtime":    strconv.FormatInt(src.ModTime(ctx).Unix(), 10),
        "filetype": contentType,
    }

    // ⚠️ Fingerprint 硬编码为空字符串！注释明确说未实现断点续传
    // "Fingerprint is used to identify the upload when resuming. That is not yet implemented"
    fingerprint := ""

    upload := NewUpload(in, src.Size(), metadata, fingerprint)
    uploader, err := o.CreateUploader(ctx, upload, options...)
    if err == nil {
        err = uploader.Upload(ctx, options...)  // 主循环调度
    }
    return err
}
```

#### 4.5.2 步骤 1：创建上传会话 — CreateUploader()

[CreateUploader()](backend/webdav/tus.go#L61-L108)：

```go
func (o *Object) CreateUploader(ctx context.Context, u *Upload, options ...fs.OpenOption) (*Uploader, error) {
    // 从目标文件路径中切出目录部分（TUS 会话创建在目录上）
    p := o.filePath()
    dir, _ := filepath.Split(p)
    if dir == "" { dir = "/" }

    // === 关键检查 ===
    // ❌ 这里没有调用 NewObject() / readMetaDataForPath()
    // ❌ 没有检查目标文件是否已存在
    // ❌ 没有设置 If-None-Match 头
    // ❌ 没有设置 Overwrite 头（TUS 协议也没有）

    l := int64(0)
    opts := rest.Opts{
        Method: "POST", Path: dir, RootURL: o.fs.endpointURL,
        ContentLength: &l, NoResponse: true,
        ExtraHeaders: o.extraHeaders(ctx, o),  // X-OC-Mtime + OC-Checksum（由 Object 复用）
        Options: options,
    }
    opts.ExtraHeaders["Upload-Length"]   = strconv.FormatInt(u.size, 10)
    opts.ExtraHeaders["Upload-Metadata"] = u.EncodedMetadata()  // filename/mtime/filetype base64
    opts.ExtraHeaders["Tus-Resumable"]   = "1.0.0"

    var tusLocation string
    err := o.fs.pacer.CallNoRetry(func() (bool, error) {
        res, err := o.fs.srv.Call(ctx, &opts)
        return o.fs.getTusLocationOrRetry(ctx, res, err)  // 201 → 读 Location 头
    })
    // upload URL 格式：/dav/uploads/<user>/<upload-uuid>

    // ⚠️ offset 硬编码为 0！不从 HEAD 查询已有进度
    uploader := NewUploader(o.fs, tusLocation, u, 0)
    return uploader, nil
}
```

**同名覆盖结论（创建阶段）—— 分两层看**：

**A. rclone 代码侧事实（可从代码直接确认）**：
- **不做存在性预检**：`CreateUploader` 内不调用 `NewObject()` / `readMetaDataForPath()` 检查目标路径是否已有同名文件
- **不设条件头**：POST 请求的 `ExtraHeaders` 中没有 `If-None-Match`、`Overwrite` 或任何防止覆盖的条件头
- **文件名仅通过 Metadata 传递**：`Upload-Metadata: filename <base64(fn)>`，rclone 自身不参与目标路径冲突判定
- **PATCH 阶段同样无冲突控制**：[uploadChunk()](backend/webdav/tus-uploader.go#L57-L105) 只设 `Upload-Offset` / `Tus-Resumable` / `filetype`，无任何条件头

**B. 服务端行为推断（不可从 rclone 代码确认）**：
- TUS 协议规范本身**未定义同名冲突的处理策略**——POST 创建上传会话和 PATCH 传输数据块都与最终文件路径无关
- 同名冲突发生在 OCIS 服务端**将上传会话组装为最终文件**时，具体策略（覆盖 / 拒绝 / 版本化）由 OCIS 实现，rclone 代码无法观测
- 目前社区经验表明 OCIS 默认为覆盖写入，但这是**服务端行为推断**，不是 rclone 代码保证的语义

#### 4.5.3 步骤 2：数据上传主循环 — Uploader.Upload()

[Upload()](backend/webdav/tus-uploader.go#L107-L122) + [UploadChunk()](backend/webdav/tus-uploader.go#L124-L162)：

```go
func (u *Uploader) Upload(ctx context.Context, options ...fs.OpenOption) error {
    cnt := 1
    // ⚠️ 循环条件：从 u.offset 开始，但 CreateUploader 中 offset 固定为 0
    for u.offset < u.upload.size && !u.aborted {
        err := u.UploadChunk(ctx, cnt, options...)
        cnt++
        if err != nil { return err }
    }
    return nil
}

func (u *Uploader) UploadChunk(ctx context.Context, cnt int, options ...fs.OpenOption) error {
    chunkSize := u.fs.opt.ChunkSize  // infinitescale 固定为 10 MiB
    data := make([]byte, chunkSize)

    // ⚠️ 从 u.offset 开始 Seek 读取 —— 如果能从服务端 HEAD 拿到 offset，这里就能断点续传
    _, err := u.upload.stream.Seek(u.offset, 0)
    size, err := u.upload.stream.Read(data)
    body := bytes.NewBuffer(data[:size])

    // PATCH 到临时的 tusLocation
    newOffset, err := u.uploadChunk(ctx, body, int64(size), u.offset, options...)

    u.offset = newOffset  // 从响应的 Upload-Offset 头更新（重要！见下）
    u.upload.updateProgress(u.offset)
    return nil
}
```

#### 4.5.4 步骤 3：单 chunk PATCH —— uploadChunk()

[uploadChunk()](backend/webdav/tus-uploader.go#L57-L105)：

```go
func (u *Uploader) uploadChunk(ctx context.Context, body io.Reader, size int64,
    offset int64, options ...fs.OpenOption) (int64, error) {

    method := "PATCH"
    extraHeaders := map[string]string{}
    extraHeaders["Upload-Offset"] = strconv.FormatInt(offset, 10)  // 客户端声明 offset
    extraHeaders["Tus-Resumable"] = "1.0.0"
    extraHeaders["filetype"] = u.upload.Metadata["filetype"]
    if u.overridePatchMethod {
        method = "POST"
        extraHeaders["X-HTTP-Method-Override"] = "PATCH"  // 兼容不支持 PATCH 的网关
    }

    opts := rest.Opts{
        Method: method, ContentType: "application/offset+octet-stream",
        ContentLength: &size, Body: body, /* ... */
    }

    var newOffset int64
    err = u.fs.pacer.CallNoRetry(func() (bool, error) {
        res, err := u.fs.srv.Call(ctx, &opts)
        // shouldRetryChunk() 关键逻辑：
        //   204 → 从响应头 Upload-Offset 读 newOffset，视为成功
        //   409 → ErrOffsetMismatch（客户端声明的 offset 与服务端不符）
        //   412 → ErrVersionMismatch
        //   413 → ErrLargeUpload
        return u.fs.shouldRetryChunk(ctx, res, err, &newOffset)
    })
    return newOffset, err
}
```

#### 4.5.5 ⚠️ TUS 断点续传：代码级结论 —— **实际上未实现！**

对照 TUS 协议规范（[tus.io/protocols/resumable-upload](https://tus.io/protocols/resumable-upload)），断点续传需要 5 个核心要素。rclone 的实现状态如下：

| TUS 断点续传要素 | 协议要求 | rclone 实现状态 | 代码证据 |
|------------------|----------|-----------------|----------|
| ① Fingerprint 生成 | 为上传创建唯一标识，用于下次恢复时查找 | ❌ **未实现**，硬编码空串 | [backend/webdav/tus.go#L29-L30](backend/webdav/tus.go#L29-L30) 注释：*"That is not yet implemented"*；`fingerprint := ""` |
| ② 持久化 Upload URL | 将 `(fingerprint → tusLocation URL + 当前 offset)` 持久化到本地存储 | ❌ **无持久化代码** | `CreateUploader` 中只把 URL 放入内存变量，返回后随 GC 丢失；[backend/webdav/tus-errors.go](backend/webdav/tus-errors.go) 有 `ErrNilStore` 但从未使用 |
| ③ HEAD 请求查询当前 Offset | 恢复时对旧 tusLocation 发 HEAD，读 `Upload-Offset` 头 | ❌ **无 HEAD 请求实现** | 代码中无任何 HEAD 方法调用；`shouldRetryChunk` 只处理 204/409/412/413 |
| ④ Resume 分支入口 | 检测到本地有未完成的 fingerprint 时走恢复路径 | ❌ **无分支判断** | `CreateUploader` 顶部注释掉的代码 `// if c.Config.Resume ... ErrFingerprintNotSet`（[backend/webdav/tus.go#L67-L69](backend/webdav/tus.go#L67-L69)）显示曾计划但未落地 |
| ⑤ 失败后从 Upload-Offset 重试 | PATCH 失败后下次能从服务端确认的 offset 继续 | ⚠️ **仅有单次 chunk 内的软校验** | `uploadChunk` 成功时从响应头 `Upload-Offset` 更新 `newOffset`（[backend/webdav/tus-uploader.go#L100-L102](backend/webdav/tus-uploader.go#L100-L102)），这是**单 chunk 内的一致性确认**，不是跨进程的断点续传 |

**最终结论**：rclone 的 TUS 实现只完成了**协议的基础上传骨架**（POST 创建 + PATCH 分块 + Upload-Offset 校验），**断点续传功能在代码层面未实现**——`fingerprint` 为空、无持久化、无 HEAD 查询、无 Resume 分支。每次 `Update()` 调用 TUS 都是**从 offset=0 重新创建新会话**。如果上传中途崩溃，之前的数据块在 OCIS 服务端会作为垃圾回收（具体取决于服务端策略），无法恢复。

---

### 4.6 SetModTime 的独立覆盖

通过 [SetModTime()](backend/webdav/webdav.go#L1465-L1518) 可独立修改已有文件的 mtime，不影响文件内容：

```go
func (o *Object) SetModTime(ctx context.Context, modTime time.Time) error {
    if o.fs.propsetMtime {
        checksums := /* SHA1:<hex> 或 MD5:<hex> 或空 */

        bodyTpl := owncloudPropset                // 只写 lastmodified
        if checksums != "" {
            bodyTpl = owncloudPropsetWithChecksum // 同时写 lastmodified + oc:checksums
        }

        opts := rest.Opts{
            Method: "PROPPATCH", Path: o.filePath(), NoRedirect: true,
            Body: strings.NewReader(fmt.Sprintf(bodyTpl, modTime.Unix(), checksums)),
        }

        var result api.Multistatus
        err := o.fs.pacer.Call(/* CallXML + shouldRetry */)

        // 三阶段判定成功：
        // 1. Multistatus 中恰好 1 条响应且 StatusOK() → 直接更新缓存，返回成功
        // 2. 否则 NewObject() 重读真实 mtime，若与预期相等 → 也算成功（服务端格式差异）
        // 3. 仍不匹配 → 返回 fs.ErrorCantSetModTime
    }
    return fs.ErrorCantSetModTime
}
```

---

### 4.7 覆盖语义总览表

| 操作 | HTTP 方法 | 覆盖控制机制 | 行为 |
|------|-----------|--------------|------|
| 普通上传 | PUT | 无控制头，依赖 PUT 默认语义 | **始终覆盖**已存在目标 |
| 流式上传 | PUT | 同上 | **始终覆盖** |
| 服务端 COPY 文件 | COPY | `Overwrite: T` 头 | 强制覆盖 |
| 服务端 MOVE 文件 | MOVE | `Overwrite: T` 头 | 强制覆盖 |
| 服务端 MOVE 目录 | MOVE | `Overwrite: T` + 预检 DirNotEmpty | 强制覆盖（但先检查目标不存在） |
| Nextcloud 分块合并 | MOVE | 未显式设 Overwrite（RFC 4918 默认 `Overwrite: T`）；服务端实际行为依赖 Nextcloud 实现 | **代码侧**：未主动控制覆盖；**服务端推断**：预期覆盖 |
| TUS 上传（infinitescale） | POST + PATCH | 不做预检，无 If-None-Match/Overwrite；TUS 协议未定义同名冲突策略 | **代码侧**：rclone 不参与冲突判定；**服务端推断**：OCIS 组装文件时决定（社区经验为覆盖） |
| SetModTime 修正 | PROPPATCH | N/A | 只修改 DAV 属性，不覆盖文件内容 |

### 4.8 失败清理策略

| 上传方式 | 失败清理行为 | 代码位置 |
|----------|-------------|----------|
| 标准 PUT (`updateSimple`) | Sleep 1s → `Remove()` 尝试删除可能的部分上传 → 忽略删除错误 | [backend/webdav/webdav.go#L1646-L1648](backend/webdav/webdav.go#L1646-L1648) |
| Nextcloud 分块 | 任一步骤失败直接返回；旧目录残留由**下次同路径上传前 purge** 清理（`createChunksUploadDirectory` 先 DELETE） | [backend/webdav/chunking.go#L150-L153](backend/webdav/chunking.go#L150-L153) |
| TUS 上传 | **无失败清理**（已知缺陷） | [backend/webdav/tus-uploader.go#L99-L102](backend/webdav/tus-uploader.go#L99-L102) 有 FIXME 注释提及删除问题但未实现；服务端 TUS 会话由 OCIS 自行 GC |

---

## 五、关键设计模式总结

### 5.1 Vendor 模式的权衡

| 优点 | 缺点 |
|------|------|
| 用户配置简单，只需选择 vendor | **无法自动发现能力**，依赖用户正确配置 |
| 避免运行时 OPTIONS/PROPFIND 探测的不确定性 | 新 vendor 接入需要改代码发版 |
| 各 vendor 的 quirks 集中管理，可维护性好 | 静态映射无法适配同 vendor 不同版本的差异（如 ownCloud 9 vs 10 vs infinitescale） |

### 5.2 属性读取的弹性设计

- **XML 反序列化的宽松性**：所有字段带 `omitempty`，不识别的属性静默忽略
- **多时间格式兜底链**：5 种格式依次尝试，最终回落到 epoch（`sync.Once` 控制只报一次错）
- **双路径目录判定**：标准 `DAV:resourcetype` 为主，Microsoft `iscollection` 扩展为辅
- **懒加载缓存**：`hasMetaData` 布尔 + `readMetaData()` 统一入口，避免 Size/ModTime/Hash 各自发起 PROPFIND
- **Sharepoint 深度降级**：`retryWithZeroDepth` 404 后降级到 `Depth=0` 重试

### 5.3 上传策略的渐进式选择

在 `Update()` 中按 **TUS → 分块 → 标准 PUT** 的优先级选择：

```
条件命中（由高到低）：
  canTus=true ............................................. →  TUS 协议
  └─ vendor=infinitescale

  canChunk && ChunkSize>0 && size>ChunkSize ............... →  Nextcloud 分块
  └─ vendor=nextcloud

  其他所有情况（包括 nextcloud 小文件、所有其他 vendor）..... →  标准 PUT
     └─ fastmail / owncloud / sharepoint / rclone / other
```

### 5.4 TUS 实现的已知局限清单

| 局限 | 影响 |
|------|------|
| **无断点续传** | 大文件上传中断后必须从头开始 |
| **无失败清理** | OCIS 服务端产生孤儿会话（依赖其自身 GC） |
| PATCH 未设置 OC-Checksum | [backend/webdav/tus-uploader.go#L66](backend/webdav/tus-uploader.go#L66) 有 FIXME；OC-Checksum 只在 POST 创建时设置，PATCH 数据块无法被校验 |
| `GetBody` 未实现 | [backend/webdav/tus-uploader.go#L79](backend/webdav/tus-uploader.go#L79) 有 FIXME；HTTP/2 GOAWAY 时无法重试 PATCH |
| offset 读取用 `strconv.ParseInt` 无错误处理 | 204 响应无 `Upload-Offset` 头时 newOffset 保持 0，可能导致下一轮循环写 0 |
