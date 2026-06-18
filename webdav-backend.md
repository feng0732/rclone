# WebDAV Backend 代码分析

本文档围绕 rclone 的 WebDAV backend 实现，深入分析三大核心机制：**远端能力探测**、**属性读取**、**上传覆盖语义**。

---

## 一、代码文件总览

| 文件 | 职责 |
|------|------|
| [webdav.go](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go) | 主文件，Fs/Object 结构定义，核心方法（列表、上传、属性、copy/move 等） |
| [api/types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/api/types.go) | API 类型定义：Prop、Response、Multistatus、Error、Time、Quota |
| [chunking.go](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/chunking.go) | Nextcloud 分块上传（chunked upload）逻辑 |
| [tus.go](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/tus.go) | ownCloud Infinite Scale TUS 协议上传入口 |
| [tus-uploader.go](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/tus-uploader.go) | TUS 上传器实现 |
| [tus-upload.go](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/tus-upload.go) | TUS Upload 结构定义 |
| [tus-errors.go](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/tus-errors.go) | TUS 错误定义 |

---

## 二、远端能力探测（Capability Detection）

WebDAV 协议本身缺乏统一的能力自描述机制（RFC 4918 未定义能力发现协议），rclone 采用的是 **vendor（厂商）静态映射 + 配置参数** 的能力探测模式。

### 2.1 核心数据结构：Fs 的能力标志位

在 [Fs](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L209-L233) 结构体中定义了大量能力布尔字段：

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

在 [setQuirks()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L631-L725) 函数中，根据 `vendor` 参数一次性设置所有能力标志位：

```go
func (f *Fs) setQuirks(ctx context.Context, vendor string) error {
    switch vendor {
    case "fastmail":
        f.canStream = true
        f.precision = time.Second
        f.useOCMtime = true
        f.hasMESHA1 = true
    case "owncloud":
        f.canStream = true
        f.precision = time.Second
        f.useOCMtime = true
        f.propsetMtime = true
        f.hasOCMD5 = true
        f.hasOCSHA1 = true
    case "infinitescale":
        f.precision = time.Second
        f.useOCMtime = true
        f.propsetMtime = true
        f.hasOCMD5 = false
        f.hasOCSHA1 = true
        f.canChunk = false
        f.canTus = true              // 启用 TUS 协议
        f.opt.ChunkSize = 10 * fs.Mebi
    case "nextcloud":
        f.precision = time.Second
        f.useOCMtime = true
        f.propsetMtime = true
        f.hasOCSHA1 = true
        f.canChunk = true            // 启用分块上传
        // 解析并设置 chunksUploadURL ...
    case "sharepoint":
        f.srv.RemoveHeader("Authorization")
        f.retryWithZeroDepth = true
        // ... Cookie 认证逻辑
    case "sharepoint-ntlm":
        f.retryWithZeroDepth = true
        f.checkBeforePurge = true
    case "rclone":
        f.canStream = true
        f.precision = time.Second
        f.useOCMtime = true
    case "other":
        f.useStandardProps = true
    }

    // 若不支持流式上传，则从 features 中移除 PutStream
    if !f.canStream {
        f.features.PutStream = nil
    }
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
NewFs() [webdav.go#L440]
  └─► f.setQuirks(ctx, opt.Vendor) [webdav.go#L533]
        └─► 根据 vendor 设置各能力标志位
        └─► 对于 nextcloud，调用 f.getChunksUploadURL() [chunking.go#L62]
        └─► 对于 sharepoint，通过 odrvcookie 获取认证 cookies
        └─► 移除不支持的 features（如 PutStream）
```

### 2.5 动态运行时探测点

除了静态 vendor 映射外，以下情况会触发**运行时能力检查**：

1. **Nextcloud 分块上传 URL 校验**：[getChunksUploadURL()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/chunking.go#L62-L72) 用正则 `^(.*)/dav/files/([^/]+)` 校验 endpoint 是否为正确格式，否则报错。

2. **流式上传动态判断**：[PutStream()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L984-L987) 只有 `canStream=true` 时才真正可用。

3. **哈希支持动态返回**：[Hashes()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L1301-L1311) 根据 `hasOCMD5` / `hasOCSHA1` / `hasMESHA1` 动态组合支持的哈希集。

---

## 三、属性读取（Attribute Reading）

属性读取通过 WebDAV 的 **PROPFIND** 方法完成。根据不同场景分为**单对象读取**和**目录批量读取**两类。

### 3.1 属性数据模型：api.Prop

在 [Prop](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/api/types.go#L70-L80) 中定义了所有可解析的属性字段，通过 XML 反序列化填充：

```go
type Prop struct {
    Status       []string  `xml:"DAV: status"`          // propstat 的 HTTP 状态
    Name         string    `xml:"DAV: prop>displayname,omitempty"`
    Type         *xml.Name `xml:"DAV: prop>resourcetype>collection,omitempty"`
    IsCollection *string   `xml:"DAV: prop>iscollection,omitempty"` // Microsoft 扩展
    Size         int64     `xml:"DAV: prop>getcontentlength,omitempty"`
    Modified     Time      `xml:"DAV: prop>getlastmodified,omitempty"`
    Checksums    []string  `xml:"prop>checksums>checksum,omitempty"`  // ownCloud: "SHA1:xxx MD5:xxx"
    Permissions  string    `xml:"prop>permissions,omitempty"`        // ownCloud 权限
    MESha1Hex    *string   `xml:"ME: prop>sha1hex,omitempty"`        // Fastmail SHA1
}
```

**状态判定**：
- [StatusOK()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/api/types.go#L104-L116)：正则 `^HTTP/[\d.]+\s+2\d{2}` 判定是否任一 propstat 为 2xx
- [Code()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/api/types.go#L86-L99)：提取首个状态的 HTTP 状态码

### 3.2 三种 PROPFIND 请求体

根据 vendor 能力不同，PROPFIND 的 Body 有三种策略：

#### 策略 A：ownCloud 扩展属性集（hasOCMD5 || hasOCSHA1）

[owncloudProps](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L757-L768)：
```xml
<?xml version="1.0"?>
<d:propfind xmlns:d="DAV:" xmlns:oc="http://owncloud.org/ns" xmlns:nc="http://nextcloud.org/ns">
  <d:prop>
    <d:displayname />
    <d:getlastmodified />
    <d:getcontentlength />
    <d:resourcetype />
    <oc:checksums />
    <oc:permissions />
  </d:prop>
</d:propfind>
```

#### 策略 B：标准属性集（useStandardProps）

[standardProps](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L770-L779)：
```xml
<?xml version="1.0"?>
<d:propfind xmlns:d="DAV:">
  <d:prop>
    <d:displayname/>
    <d:getlastmodified/>
    <d:getcontentlength/>
    <d:resourcetype/>
  </d:prop>
</d:propfind>
```

#### 策略 C：空 Body（allprop）

当既不属于 A 也不属于 B 时，发送**空 Body**。根据 RFC 4918，空 PROPFIND body 默认使用 `allprop` 行为。

### 3.3 单对象属性读取：readMetaDataForPath()

[readMetaDataForPath()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L343-L393) 用于读取单个路径的属性：

```go
func (f *Fs) readMetaDataForPath(ctx context.Context, path string) (info *api.Prop, err error) {
    opts := rest.Opts{
        Method: "PROPFIND",
        Path:   f.filePath(path),
        ExtraHeaders: map[string]string{
            "Depth": "0",         // 关键：Depth=0 只请求自身
        },
        CheckRedirect: rest.PreserveMethodRedirectFn,
    }
    // 根据能力标志选择 Body ...
    var result api.Multistatus
    err = f.pacer.Call(func() (bool, error) {
        resp, err = f.srv.CallXML(ctx, &opts, nil, &result)
        return f.shouldRetry(ctx, resp, err)
    })

    // 错误处理：
    // 404 / 301 / 302 / 303 → 返回 fs.ErrorObjectNotFound
    // 响应数 < 1 → ErrorObjectNotFound
    // status 非 2xx 且非 425(Too Early) → ErrorObjectNotFound
    // 是目录 → fs.ErrorIsDir

    return &item.Props, nil
}
```

### 3.4 对象懒加载机制：readMetaData()

Object 的元数据采用 **懒加载（lazy loading）** 模式：

```go
// readMetaData [webdav.go#L1415-L1424]
func (o *Object) readMetaData(ctx context.Context) (err error) {
    if o.hasMetaData {  // 已加载则直接返回
        return nil
    }
    info, err := o.fs.readMetaDataForPath(ctx, o.remote)
    if err != nil {
        return err
    }
    return o.setMetaData(info)
}

// setMetaData [webdav.go#L1395-L1410]
func (o *Object) setMetaData(info *api.Prop) (err error) {
    o.hasMetaData = true
    o.size = info.Size
    o.modTime = time.Time(info.Modified)
    if o.fs.hasOCMD5 || o.fs.hasOCSHA1 || o.fs.hasMESHA1 {
        hashes := info.Hashes()   // 解析 "SHA1:xxx MD5:xxx" 格式
        o.sha1 = hashes[hash.SHA1]
        o.md5 = hashes[hash.MD5]
    }
    return nil
}
```

**调用链**：在 `Size()`、`ModTime()`、`Hash()` 等方法中都会先调用 `readMetaData()` 确保元数据已加载。

### 3.5 目录列表属性读取：listAll()

[listAll()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L789-L895) 是目录遍历的核心：

```go
func (f *Fs) listAll(ctx context.Context, dir string, directoriesOnly bool,
    filesOnly bool, depth string, fn listAllFn) (found bool, err error) {

    opts := rest.Opts{
        Method: "PROPFIND",
        Path:   f.dirPath(dir),
        ExtraHeaders: map[string]string{
            "Depth": depth,   // 默认 depth="1"（一级子项）
        },
    }
    // 根据能力选择 Body ...

    // 失败重试：对于 sharepoint 等需要 retryWithZeroDepth 的 vendor，
    // 404 后降级到 depth="0" 重试：
    if f.retryWithZeroDepth && depth != "0" {
        return f.listAll(ctx, dir, directoriesOnly, filesOnly, "0", fn)
    }

    // 遍历 Multistatus 响应：
    for i := range result.Responses {
        item := &result.Responses[i]
        isDir := itemIsDir(item)  // 判定是否为目录

        // URL 路径处理：Join → 去前缀 → Decode → 标准化
        remote := path.Join(dir, subPath)

        // 跳过自身（列表中包含请求的目录本身）
        if remote == dir { continue }

        // 过滤：ExcludeShares（含 'S' 权限）、ExcludeMounts（含 'M' 权限）
        if f.opt.ExcludeShares && strings.Contains(item.Props.Permissions, "S") {
            continue
        }
    }
}
```

### 3.6 目录判定逻辑：itemIsDir()

[itemIsDir()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L321-L341) 采用**双重判定**策略：

1. **标准判定**：`resourcetype` 是否为 `DAV:collection`（主判据）
2. **Microsoft 扩展**：若 `iscollection` 属性存在，解析其值为 `0/false` 或 `1/true`
3. **兜底**：都不满足则视为**非目录（文件）**（遵循 WebDAV 规范：未知资源类型默认按非集合处理）

### 3.7 时间解析：自定义 Time 类型

[Time.UnmarshalXML()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/api/types.go#L202-L235) 支持 5 种时间格式（依次尝试）：

| 格式 | 示例 | 说明 |
|------|------|------|
| `time.RFC1123` | `Wed, 27 Sep 2017 14:28:34 GMT` | RFC 标准格式 |
| `time.RFC1123Z` | `Fri, 05 Jan 2018 14:14:38 +0000` | mydrive.ch 使用 |
| `time.UnixDate` | `Wed May 17 15:31:58 UTC 2017` | 某内部服务器 |
| `noZerosRFC1123` | `Fri, 7 Sep 2018 08:49:58 GMT` | #2574 无前导零 |
| `time.RFC3339` | `Wed, 31 Oct 2018 13:57:11 CET` | komfortcloud.de |

解析失败时使用 `Unix(0, 0)`（epoch）兜底，并只输出一次错误日志。

### 3.8 校验和解析：Prop.Hashes()

[Hashes()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/api/types.go#L118-L140) 支持两种格式：

1. **ownCloud 格式**：`<oc:checksum>SHA1:f572d... MD5:b194... ADLER32:084b...</oc:checksum>`，以空格分隔，按前缀匹配 SHA1/MD5
2. **Fastmail 格式**：直接从 `ME:sha1hex` 属性读取

---

## 四、上传覆盖语义（Upload Overwrite Semantics）

WebDAV 的上传/复制/移动涉及多种覆盖场景，rclone 通过不同机制处理。

### 4.1 普通上传覆盖：PUT 方法

#### 4.1.1 核心方法：updateSimple()

[updateSimple()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L1616-L1652) 是最基础的 PUT 上传：

```go
func (o *Object) updateSimple(ctx context.Context, body io.Reader,
    getBody func() (io.ReadCloser, error), filePath string, size int64,
    contentType string, extraHeaders map[string]string, rootURL string,
    options ...fs.OpenOption) (err error) {

    opts := rest.Opts{
        Method:        "PUT",
        Path:          filePath,
        GetBody:       getBody,
        Body:          body,
        ContentLength: &size,
        ContentType:   contentType,
        ExtraHeaders:  extraHeaders,  // 包含 X-OC-Mtime、OC-Checksum
        RootURL:       rootURL,
    }

    // 使用 CallNoRetry，避免 HTTP 层的自动重试
    err = o.fs.pacer.CallNoRetry(func() (bool, error) {
        resp, err = o.fs.srv.Call(ctx, &opts)
        return o.fs.shouldRetry(ctx, resp, err)
    })

    if err != nil {
        time.Sleep(1 * time.Second)  // 等服务端整理内部状态
        _ = o.Remove(ctx)            // 清理可能的部分上传产物
        return err
    }
    return nil
}
```

#### 4.1.2 PUT 的覆盖语义：隐式覆盖

WebDAV PUT 方法**默认会覆盖已存在的目标资源**（RFC 4918 §7.6）。rclone 的实现中**没有设置 `If-None-Match` 或 `Overwrite` 头**，因此：

- **目标不存在** → 创建新文件（201 Created）
- **目标已存在** → **覆盖写入**（200 OK / 204 No Content）

> ⚠️ rclone 不做"仅当不存在时创建"的原子保护，覆盖由上层业务（如 sync 的 check 逻辑）控制。

#### 4.1.3 上传的 ModTime 和 Checksum 传递

通过 [extraHeaders()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L1593-L1614) 设置：

| Header | 条件 | 值 |
|--------|------|----|
| `X-OC-Mtime` | `useOCMtime=true` | `src.ModTime().Unix()`（Unix 时间戳） |
| `OC-Checksum` | `hasOCSHA1=true` 且有 SHA1 | `SHA1:<hex>` |
| `OC-Checksum` | `hasOCMD5=true` 且无 SHA1 但有 MD5 | `MD5:<hex>` |

### 4.2 COPY/MOVE 的覆盖语义：Overwrite 头

#### 4.2.1 copyOrMove() 的处理

在 [copyOrMove()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L1154-L1206) 中：

```go
opts := rest.Opts{
    Method:     method,   // "COPY" 或 "MOVE"
    Path:       srcObj.filePath(),
    ExtraHeaders: map[string]string{
        "Destination": destinationURL.String(),
        "Overwrite":   "T",           // ← 关键：强制允许覆盖
    },
}
if f.useOCMtime {
    opts.ExtraHeaders["X-OC-Mtime"] = fmt.Sprintf("%d", src.ModTime(ctx).Unix())
}
```

**Overwrite 头的规范值**（RFC 4918 §10.6）：
- `"T"` / `"t"` → **允许覆盖**已存在的目标资源
- `"F"` / `"f"` → **不允许覆盖**，若目标存在返回 412 Precondition Failed

rclone 一律使用 `"T"`，即 **COPY/MOVE 始终覆盖目标**。

#### 4.2.2 ModTime 二次确认逻辑

COPY/MOVE 完成后，如果：
1. `useOCMtime=true`（使用了 `X-OC-Mtime` 头）
2. 响应头 `X-OC-Mtime` **不是** `"accepted"`（服务端未确认接受 mtime）
3. `propsetMtime=true`（支持 PROPPATCH 设置 mtime）
4. 实际 ModTime 与预期不相等

则通过 `dstObj.SetModTime()` 发起 **PROPPATCH** 二次修正 mtime。

### 4.3 Nextcloud 分块上传的覆盖

在 [updateChunked()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/chunking.go#L78-L104) 中，分三步：

#### 步骤 1：创建上传目录

[createChunksUploadDirectory()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/chunking.go#L144-L169)：
- 根据目标文件路径做 MD5，生成 `rclone-chunked-upload-<md5>` 目录名
- **先 purge 同名目录**（清理上次未成功的残留）
- 再 `MKCOL` 创建新目录

#### 步骤 2：上传数据块

[uploadChunks()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/chunking.go#L106-L142)：
- 按 `ChunkSize` 分块，文件命名为 `{uploadDir}/{offset:015d}-{endOffset:015d}`
- 每个 chunk 使用 `updateSimple()` 的 PUT 方法，上传到 `chunksUploadURL`（独立 URL，如 `/dav/uploads/<user>/`）
- 使用 `RepeatableLimitReaderBuffer` 支持 HTTP/2 重试（通过 `GetBody`）

#### 步骤 3：MOVE 合并（关键覆盖点）

[mergeChunks()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/chunking.go#L171-L198)：

```go
opts := rest.Opts{
    Method:     "MOVE",
    Path:       path.Join(uploadDir, ".file"),  // 源：虚拟的 .file
    ExtraHeaders: map[string]string{
        "Destination": destinationURL.String(),  // 目标：真实路径
        // X-OC-Mtime 和 OC-Checksum 通过 extraHeaders 注入
    },
    RootURL:    o.fs.chunksUploadURL,
}

// 特殊重试逻辑：423 LOCKED → 退避等待服务端合并
// 423 后收到 404 → 视为合并成功（wasLocked 标记）
```

**关键点**：
- 最终合并使用 MOVE 方法，**隐式覆盖目标**（Nextcloud 的 chunking 协议中 MOVE 始终覆盖）
- 未显式设置 `Overwrite` 头，依赖 Nextcloud 服务端语义
- 423 LOCKED 是合并中的正常状态，按指数退避（5s, 10s, 20s...）重试

### 4.4 TUS 协议的覆盖

ownCloud Infinite Scale 使用 [TUS 协议](https://tus.io/protocols/resumable-upload)：

[updateViaTus()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/tus.go#L20-L43)：
1. **POST 创建上传会话**：`POST /dav/files/<user>/<dir>/`，带 `Upload-Length`、`Upload-Metadata`、`Tus-Resumable`
2. 从响应 `Location` 头获得上传 URL
3. **PATCH 上传数据块**（tus-uploader 实现），支持断点续传
4. 服务端在 PATCH 完成后自动组装为最终文件

**TUS 的覆盖语义**：创建会话时 TUS 协议本身不定义覆盖机制，由服务端实现决定。在 OCIS 中，**重复上传同一路径会被视为新文件覆盖旧文件**。

### 4.5 SetModTime 的独立覆盖

通过 [SetModTime()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L1465-L1518) 可独立修改已有文件的 mtime：

```go
func (o *Object) SetModTime(ctx context.Context, modTime time.Time) error {
    if o.fs.propsetMtime {
        // 构造 PROPPATCH Body：
        // 有校验和 → owncloudPropsetWithChecksum（同时写入 mtime 和 checksums）
        // 无校验和 → owncloudPropset（只写 mtime）

        opts := rest.Opts{
            Method:     "PROPPATCH",
            Path:       o.filePath(),
            NoRedirect: true,
            Body:       strings.NewReader(fmt.Sprintf(...)),
        }

        // 判定成功：
        // 1. 响应 Multistatus 中首个且状态 OK → 直接缓存
        // 2. 否则通过 NewObject() 重读确认，匹配则也算成功
        // 3. 仍不匹配 → 返回 fs.ErrorCantSetModTime
    }
}
```

### 4.6 覆盖语义总览表

| 操作 | HTTP 方法 | 覆盖控制 | 行为 |
|------|-----------|----------|------|
| 普通上传 | PUT | 无（默认覆盖） | 始终覆盖已存在目标 |
| 流式上传 | PUT | 无（默认覆盖） | 始终覆盖 |
| 服务端 COPY | COPY | `Overwrite: T` | 强制覆盖 |
| 服务端 MOVE | MOVE | `Overwrite: T` | 强制覆盖 |
| 目录移动 | MOVE | `Overwrite: T` | 强制覆盖（但先检查目标不存在） |
| Nextcloud 分块合并 | MOVE | 依赖服务端默认 | 隐式覆盖 |
| TUS 上传 | POST + PATCH | 依赖服务端实现 | 隐式覆盖 |
| ModTime 修正 | PROPPATCH | N/A | 只修改属性不覆盖内容 |

### 4.7 失败清理策略

在 `updateSimple()` 中，上传失败后：
1. 等待 1 秒给服务端整理状态
2. 调用 `o.Remove(ctx)` 删除可能的部分上传产物
3. 删除失败被**静默忽略**（`_ = o.Remove(ctx)`）

---

## 五、关键设计模式总结

### 5.1 Vendor 模式的权衡

| 优点 | 缺点 |
|------|------|
| 配置简单，用户只需选 vendor | 无法自动发现能力，依赖用户正确配置 |
| 避免运行时探测的不确定性 | 新 vendor 需要代码改动 |
| 各 vendor 的 quirks 集中管理，可读性好 | 静态映射无法适配同 vendor 不同版本的差异 |

### 5.2 属性读取的弹性设计

- **XML 反序列化的宽松性**：Prop 中所有字段带 `omitempty`，不识别的属性被忽略
- **多时间格式兜底**：5 种格式依次尝试，最后回落到 epoch
- **双路径目录判定**：标准 `resourcetype` + Microsoft `iscollection` 扩展
- **懒加载缓存**：`hasMetaData` 标志避免重复 PROPFIND

### 5.3 上传的多种策略选择

在 [Update()](file:///d:/fz/0601-2/solo-dogfeeding/code/47-rclone/backend/webdav/webdav.go#L1553-L1591) 中按优先级选择：

```
1. canTus → TUS 协议上传（infinitescale）
   ↓
2. shouldUseChunkedUpload(canChunk && ChunkSize>0 && size>ChunkSize)
   → Nextcloud 分块上传
   ↓
3. updateSimple() → 标准 PUT（其余所有情况）
```
