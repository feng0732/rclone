# S3 客户端后端代码分析

## 一、整体架构

rclone 的 S3 客户端后端位于 `backend/s3/` 目录，基于 **AWS SDK v2 for Go**（`github.com/aws/aws-sdk-go-v2/service/s3`）构建，向上实现 rclone 的 `fs.Fs` 统一后端接口，使 rclone 能够读写任何 S3 兼容存储服务。

### 架构分层

```
┌──────────────────────────────────────────────────────────────┐
│                  rclone 上层逻辑 (sync / copy / ls 等)       │
└─────────────────────────────┬────────────────────────────────┘
                              │ fs.Fs / fs.Object 统一接口
┌─────────────────────────────▼────────────────────────────────┐
│                 S3 客户端后端 (backend/s3/)                  │
│                                                               │
│  Fs (实现 fs.Fs)                                              │
│    ├─ List / ListP / ListR    → 对象列表                      │
│    ├─ NewObject               → 获取对象引用                  │
│    ├─ Put                     → 上传入口                      │
│    ├─ Mkdir / Rmdir           → 目录 / 桶操作                 │
│    └─ Copy / Purge 等         → 扩展功能                      │
│                                                               │
│  Object (实现 fs.Object)                                      │
│    ├─ Open                    → 下载 (GetObject)              │
│    ├─ Update                  → 上传 (PutObject / Multipart)  │
│    ├─ Remove                  → 删除 (DeleteObject)           │
│    └─ SetModTime / Hash 等    → 元数据操作                    │
│                                                               │
│  Pacer: 令牌桶限流 + 重试                                     │
│  Provider / Quirks: 多厂商适配                                │
└─────────────────────────────┬────────────────────────────────┘
                              │ AWS SDK v2 S3 Client API
┌─────────────────────────────▼────────────────────────────────┐
│                 AWS SDK v2 (service/s3)                      │
│                                                               │
│  中间件栈 (Middleware Stack):                                 │
│    Initialize → Serialize → Build →  Signing → Retry → Send  │
│                                                               │
│  Signing 阶段签名器:                                          │
│    - 默认: aws/signer/v4 (Signature v4)                       │
│    - 可替换: HTTPSignerV4 接口 (v2 / IBM IAM / 自定义)       │
│                                                               │
│  Credentials Provider:                                       │
│    - StaticCredentials                                        │
│    - EnvCredentials                                           │
│    - AssumeRole (STS)                                         │
│    - Anonymous                                                │
└─────────────────────────────┬────────────────────────────────┘
                              │ HTTP / HTTPS
┌─────────────────────────────▼────────────────────────────────┐
│              S3 兼容存储 (AWS S3 / MinIO / Ceph / ...)       │
└──────────────────────────────────────────────────────────────┘
```

### Provider 适配机制

S3 后端通过 **provider.yaml** 文件定义数十种 S3 兼容提供商的特性与 quirk，在 [providers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/providers.go) 中加载：

- **`Provider` 结构体**：定义每个提供商的 region、endpoint、ACL、storage class、SSE 等配置选项
- **`Quirks` 结构体**：定义行为差异（列表版本、路径风格、URL 编码、ETag 是否为 MD5、是否支持 multipart 等）
- 所有 provider 配置通过 `embed.FS` 内嵌在 `provider/*.yaml` 文件中，通过 `loadProvider(name)` 按名查找

---

## 二、认证配置与签名注入

### 2.1 认证配置项

认证相关配置定义在 [s3.go](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L89-L200) 的 Options 结构体中，主要包括：

| 配置项 | 说明 |
|--------|------|
| `provider` | S3 提供商类型（AWS / MinIO / Ceph / IBMCOS 等） |
| `env_auth` | 是否从环境 / EC2 元数据 / 配置文件加载凭证 |
| `access_key_id` | AWS Access Key ID |
| `secret_access_key` | AWS Secret Access Key |
| `session_token` | 会话令牌（临时凭证） |
| `region` | 区域 |
| `endpoint` | 自定义 S3 API 端点 |
| `role_arn` | 需扮演的 IAM Role ARN（STS AssumeRole） |
| `role_session_name` | Role 会话名称 |
| `role_external_id` | Role External ID |
| `profile` | AWS 配置文件 profile 名 |
| `v2_auth` | 是否使用 S3 v2 签名（兼容老 S3） |
| `ibm_api_key` + `ibm_service_instance_id` | IBM COS IAM 认证 |

### 2.2 客户端创建与签名注入总览

核心入口函数为 `s3Connection`，定义在 [s3.go:1475-1629](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L1475-L1629)：

```go
func s3Connection(ctx context.Context, opt *Options, client *http.Client) (
    s3Client *s3.Client, provider *Provider, err error)
```

整个过程可分为 **凭证准备 → 签名器注入 → SDK 客户端创建** 三个阶段：

```
阶段 1: 凭证准备 (Credentials Provider)
    │
    ├─► 默认: StaticCredentialsProvider{AccessKeyID, SecretAccessKey, SessionToken}
    │
    ├─► env_auth + key 为空
    │    └─► awsconfig.LoadDefaultConfig()
    │         从环境变量 / 共享配置文件 / EC2 Instance Profile 加载
    │
    ├─► IBMCOS + v2_auth
    │    └─► NoOpCredentialsProvider (mock 凭证，实际用 IAM token)
    │
    ├─► key 都为空（非 env_auth）
    │    └─► aws.AnonymousCredentials (匿名)
    │
    └─► role_arn 非空
         └─► sts.NewFromConfig(baseConfig)
         └─► stscreds.NewAssumeRoleProvider(stsClient, roleARN, options)
         └─► aws.NewCredentialsCache(...) 包装缓存

阶段 2: 签名器注入 (Signer Injection)
    │
    ├─► 默认: 使用 SDK 内置 aws/signer/v4 (Signature v4)
    │
    ├─► v2_auth 或 region == "other-v2-signature"
    │    └─► HTTPSignerV4 = &v2Signer{opt: opt}
    │         (替换为 S3 v2 签名实现)
    │
    └─► IBMCOS + ibm_api_key + ibm_instance_id
         └─► HTTPSignerV4 = &IbmIamSigner{...}
              (替换为 IBM IAM token 认证)

阶段 3: 创建 s3.Client
    └─► s3.NewFromConfig(awsConfig, options...)
         将凭证和签名器装配进 SDK 中间件栈
```

### 2.3 签名器接口：HTTPSignerV4

AWS SDK v2 允许通过 `s3.Options.HTTPSignerV4` 字段替换默认的 v4 签名器。该接口定义（SDK 内部）签名方法签名为：

```go
SignHTTP(ctx context.Context, credentials aws.Credentials, req *http.Request,
    payloadHash string, service string, region string, signingTime time.Time,
    optFns ...func(*v4signer.SignerOptions)) error
```

rclone 自定义了两种签名器实现：

#### v2Signer — S3 v2 签名

[v2sign.go:44-53](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/v2sign.go#L44-L53)

```go
type v2Signer struct { opt *Options }

func (v2 *v2Signer) SignHTTP(ctx context.Context, credentials aws.Credentials,
    req *http.Request, payloadHash string, service string, region string,
    signingTime time.Time, optFns ...func(*v4signer.SignerOptions)) error {

    date := time.Now().UTC().Format(time.RFC1123)
    req.Header.Set("Date", date)
    // 1. 收集需要签名的头部 (content-md5, content-type, x-amz-*)
    // 2. 收集需要签名的 URL 参数 (acl, uploadId, versionId 等 s3ParamsToSign)
    // 3. 构造签名字符串: "HTTP_METHOD\nmd5\ncontentType\ndate\ncanonicalizedAmzHeaders\ncanonicalizedResource"
    // 4. HMAC-SHA1 签名: base64(hmac(secret, stringToSign))
    // 5. 设置 Authorization 头: "AWS accessKey:signature"
}
```

#### IbmIamSigner — IBM IAM Token 认证

[ibm_signer.go:19-42](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/ibm_signer.go#L19-L42)

```go
type IbmIamSigner struct {
    APIKey      string
    InstanceID  string
    IAMEndpoint string
    Auth        Authenticator
}

func (signer *IbmIamSigner) SignHTTP(...) error {
    authenticator := &core.IamAuthenticator{ApiKey: signer.APIKey, URL: signer.IAMEndpoint}
    token, _ := authenticator.GetToken()       // 调用 IAM 接口获取 token
    req.Header.Set("Authorization", "Bearer "+token)
    req.Header.Set("ibm-service-instance-id", signer.InstanceID)
    return nil
}
```

> 注意：IBM IAM 模式下，凭证 provider 使用 `NoOpCredentialsProvider`（mock 凭证），因为实际认证是通过 IAM token 在 SignHTTP 中完成的。

### 2.4 签名器注入代码

签名器通过 `s3.Options` 的 `HTTPSignerV4` 字段注入，注入位置在 [s3Connection 函数](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L1601-L1612)：

```go
if opt.V2Auth || opt.Region == "other-v2-signature" {
    if opt.Provider == "IBMCOS" && opt.IBMAPIKey != "" && opt.IBMInstanceID != "" {
        // IBM IAM 签名
        options = append(options, func(s3Opt *s3.Options) {
            s3Opt.HTTPSignerV4 = &IbmIamSigner{
                APIKey: opt.IBMAPIKey,
                InstanceID: opt.IBMInstanceID,
                IAMEndpoint: opt.IBMIAMEndpoint,
            }
        })
    } else {
        // S3 v2 签名
        options = append(options, func(s3Opt *s3.Options) {
            s3Opt.HTTPSignerV4 = &v2Signer{opt: opt}
        })
    }
}
```

最终通过 `s3.NewFromConfig(awsConfig, options...)` 将签名器装配进 SDK。

### 2.5 签名中间件在请求中的位置

AWS SDK v2 的每个 API 请求都经过一个中间件栈（Middleware Stack），签名发生在 **Signing 阶段**：

```
请求发起
   │
   ▼
Initialize 中间件   ← 设置默认值、验证参数
   │
   ▼
Serialize 中间件    ← 将输入结构体序列化为 HTTP 请求
   │
   ▼
Build 中间件        ← 构建完整 HTTP 请求 (URL + Headers + Body)
   │
   ▼
【Signing 中间件】  ← ★ 调用 HTTPSignerV4.SignHTTP() 进行签名 ★
   │                  - 从 CredentialsProvider 获取凭证
   │                  - 计算签名 (v4 / v2 / IAM token)
   │                  - 将签名写入 Authorization 头
   │
   ▼
Retry 中间件        ← 失败重试逻辑
   │
   ▼
Send 中间件         ← 通过 http.Client 发送请求
   │
   ▼
响应返回
```

**关键点**：签名发生在 Build 之后、Retry 之前，每次重试都会重新签名（因为时间戳会变化）。

### 2.6 请求修复中间件 (fixupRequest)

除了签名器替换，rclone 还通过 `fixupRequest` 函数在 Signing 阶段前后插入自定义中间件，处理特定厂商的兼容性问题：

[fixupRequest](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L1392-L1458)

```go
func fixupRequest(o *s3.Options, opt *Options) {
    // 在 Signing 之前：删除 Accept-Encoding 头（避免参与签名）
    fixup := FinalizeMiddlewareFunc("FixupRequest",
        func(ctx, in, next) {
            if !opt.SignAcceptEncoding.Value {
                ignored := make(map[string]string)
                for _, h := range []string{"Accept-Encoding"} {
                    ignored[h] = req.Header.Get(h)
                    req.Header.Del(h)    // 签名前移除，不参与签名
                }
                ctx = 保存 ignored 到 ctx
            }
            if !opt.UseXID.Value { /* 删除 x-id URL 参数 */ }
            return next.HandleFinalize(ctx, in)
        })

    // 在 Signing 之后：恢复 Accept-Encoding 头
    restore := FinalizeMiddlewareFunc("FixupRequestRestoreHeaders",
        func(ctx, in, next) {
            if !opt.SignAcceptEncoding.Value {
                从 ctx 取出 ignored, 恢复到 req.Header
            }
            return next.HandleFinalize(ctx, in)
        })

    // 插入到中间件栈
    stack.Finalize.Insert(fixup, "Signing", middleware.Before)
    stack.Finalize.Insert(restore, "Signing", middleware.After)
}
```

**用途**：Google Cloud Storage 等厂商修改 Accept-Encoding 头会破坏 v2 签名，因此在签名前临时移除、签名后恢复。

---

## 三、对象列表 (List) — 签名与实现

### 3.1 接口映射

| `fs.Fs` 接口 | S3 后端实现 | 底层 S3 API | 签名方式 |
|-------------|-------------|------------|---------|
| `List(ctx, dir)` | `Fs.List` → `list.WithListP` → `Fs.ListP` | ListObjectsV2 | Signing 中间件自动签名 |
| `ListP(ctx, dir, callback)` | `Fs.ListP` → `Fs.listDir` → `Fs.list` | ListObjectsV2（带 delimiter） | 同上 |
| `ListR(ctx, dir, callback)` | `Fs.ListR` → `Fs.list(recurse=true)` | ListObjectsV2（无 delimiter） | 同上 |

### 3.2 List 入口

[Fs.List](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L2654-L2656)

```go
func (f *Fs) List(ctx context.Context, dir string) (entries fs.DirEntries, err error) {
    return list.WithListP(ctx, dir, f)
}
```

`list.WithListP` 是通用工具函数，通过调用 `ListP`（分页列表）收集全量结果。

### 3.3 ListP — 非递归列表

[Fs.ListP](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L2671-L2695)

```go
func (f *Fs) ListP(ctx context.Context, dir string, callback fs.ListRCallback) error {
    bucket, directory := f.split(dir)
    if bucket == "" {
        // 根目录：列出所有 bucket
        entries, err := f.listBuckets(ctx)     // ListBuckets API
        for _, entry := range entries { list.Add(entry) }
    } else {
        // 指定 bucket：列出目录
        err := f.listDir(ctx, bucket, directory, f.rootDirectory, f.rootBucket == "", list.Add)
    }
    return list.Flush()
}
```

### 3.4 list — 核心分页列表循环

[Fs.list](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L2387-L2577) 是最核心的列表实现，所有 List 变体最终都调用它。

**签名参与的完整调用链**：

```
上层调用 Fs.list(ctx, listOpt{...}, callback)
   │
   ├─► 1. 构造 ListObjectsV2Input
   │     Bucket / Delimiter / Prefix / MaxKeys (ListChunk)
   │
   ├─► 2. 选择 bucketLister 实现
   │     ├─ 版本化列表 → newVersionsList
   │     ├─ ListVersion=1 → newV1List (ListObjects v1)
   │     └─ 默认 → newV2List (ListObjects v2)
   │
   ├─► 3. 分页循环
   │     for {
   │         ┌──────────────────────────────────────────────┐
   │         │  f.pacer.Call(func() (bool, error) {        │
   │         │                                              │
   │         │    listBucket.URLEncodeListings(...)         │
   │         │                                              │
   │         │    resp, versionIDs, err = listBucket.List(ctx)
   │         │         │                                   │
   │         │         └─► 最终调用 s3.Client.ListObjectsV2 │
   │         │              (进入 SDK 中间件栈)              │
   │         │                                              │
   │         │    // 遇 XML 语法错误自动重试 URL 编码        │
   │         │    if err 是 xml.SyntaxError && !urlEncode { │
   │         │      urlEncodeListings = true                │
   │         │      return true, err   // 触发 pacer 重试  │
   │         │    }                                        │
   │         │                                              │
   │         │    return f.shouldRetry(ctx, err)            │
   │         │  })                                          │
   │         └──────────────────────────────────────────────┘
   │         │
   │         ▼
   │     SDK 内部: ListObjectsV2
   │         │
   │         ├─► Initialize 中间件
   │         ├─► Serialize 中间件
   │         ├─► Build 中间件
   │         ├─► Signing 中间件  ◄─── 【签名发生在这里】
   │         │      └─► 从 CredentialsProvider 取凭证
   │         │      └─► 调用 HTTPSignerV4.SignHTTP()
   │         │      └─► 计算签名，写入 Authorization 头
   │         ├─► Retry 中间件
   │         └─► Send 中间件 → HTTP 请求发出
   │
   ├─► 4. 处理 CommonPrefixes (目录)
   │     URL 解码 → Enc.ToStandardPath → 前缀裁剪 → 回调 fn(..., isDirectory=true)
   │
   ├─► 5. 处理 Contents (文件)
   │     URL 解码 → Enc.ToStandardPath → 识别目录标记 → 回调 fn(..., isDirectory=false)
   │
   └─► 6. if !resp.IsTruncated { break }
```

### 3.5 Pacer 与签名的关系

每个列表请求都包裹在 `f.pacer.Call()` 中：

- **pacer** 负责令牌桶限速（避免触发 S3 限流）和有限次数的重试
- **SDK 内部**也有自己的重试机制（`RetryMaxAttempts`）
- **两层重试的签名**：每次重试（不论是 pacer 层还是 SDK 层）都会重新进入 Signing 中间件，重新计算签名（因为签名中的时间戳会过期）

### 3.6 目录标记识别

S3 没有原生目录概念，rclone 通过 **目录标记**（directory marker）模拟目录：

```go
// 识别逻辑：以 "/" 结尾 且 size == 0
isDirectory := (remote == "" || strings.HasSuffix(remote, "/")) &&
    object.Size != nil && *object.Size == 0
```

识别出的目录标记会被转换为 `fs.Directory` 条目，文件则转换为 `*Object`（实现 `fs.Object`），通过 `itemToDirEntry` 函数完成转换：

[Fs.itemToDirEntry](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L2580-L2594)

```go
func (f *Fs) itemToDirEntry(ctx context.Context, remote string,
    object *types.Object, versionID *string, isDirectory bool) (fs.DirEntry, error) {
    if isDirectory {
        return fs.NewDir(remote, time.Time{}).SetSize(size), nil   // fs.Directory
    }
    return f.newObjectWithInfo(ctx, remote, object, versionID)    // *Object
}
```

### 3.7 列表中的编码处理

- **URL 编码**：部分 S3 兼容存储支持 URL 编码对象键响应，避免特殊字符破坏 XML。遇到 `xml.SyntaxError` 时自动切换到 URL 编码重试。
- **路径编码**：所有对象键经过 `f.opt.Enc.ToStandardPath(remote)` 转换，将 S3 端的转义字符还原为 rclone 内部标准路径表示。

---

## 四、上传 (Put) — 签名与实现

### 4.1 接口映射

| `fs.Fs` / `fs.Object` 接口 | S3 后端实现 | 底层 S3 API | 签名方式 |
|--------------------------|-------------|------------|---------|
| `Fs.Put(ctx, in, src)` | `Fs.Put` → `Object.Update` | PutObject / CreateMultipartUpload + UploadPart | Signing 中间件自动签名 |
| `Object.Update(ctx, in, src)` | 调度单分片/多分片上传 | 同上 | 同上 |
| `Fs.Mkdir(ctx, dir)` | bucket 检测 / 目录标记 | CreateBucket / PutObject (空对象) | 同上 |

### 4.2 Put 入口

[Fs.Put](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L2764-L2771)

```go
func (f *Fs) Put(ctx context.Context, in io.Reader, src fs.ObjectInfo,
    options ...fs.OpenOption) (fs.Object, error) {
    fs := &Object{ fs: f, remote: src.Remote() }
    return fs, fs.Update(ctx, in, src, options...)
}
```

`Put` 创建一个临时 `Object`，然后委托给 `Object.Update` 完成实际上传。

### 4.3 Object.Update — 上传调度

[Object.Update](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L5022-L5105)

```go
func (o *Object) Update(ctx context.Context, in io.Reader,
    src fs.ObjectInfo, options ...fs.OpenOption) error {

    size := src.Size()
    multipart := size < 0 || size >= int64(o.fs.opt.UploadCutoff)

    if multipart {
        // 多分片上传
        wantETag, gotETag, versionID, ui, err = o.uploadMultipart(ctx, src, in, options...)
    } else {
        // 单分片上传
        ui, err = o.prepareUpload(ctx, src, options, false)
        if o.fs.opt.UsePresignedRequest {
            gotETag, lastModified, versionID, err =
                o.uploadSinglepartPresignedRequest(ctx, ui.req, size, in)
        } else {
            gotETag, lastModified, versionID, err =
                o.uploadSinglepartPutObject(ctx, ui.req, size, in)
        }
    }

    // 上传后处理
    if o.fs.opt.NoHead && size >= 0 {
        // 不做 HEAD，根据上传响应构造元数据
    } else {
        head, err = o.headObject(ctx)     // 调用 HeadObject 校验
    }
    o.setMetaData(head)
    return err
}
```

**上传模式决策**：
- **size < 0**（未知大小）→ 多分片上传
- **size >= UploadCutoff**（默认 200MB）→ 多分片上传
- **size < UploadCutoff** → 单分片上传

### 4.4 prepareUpload — 上传准备

[Object.prepareUpload](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L4807-L4936)

构造 `s3.PutObjectInput` 请求对象，设置所有上传参数：

```
1. 确保父目录/bucket 存在 → mkdirParent()
2. 构造 PutObjectInput
   ├─► Bucket, Key
   ├─► ACL (访问控制列表)
   ├─► StorageClass (从 src.GetTier() 获取)
   ├─► 元数据映射
   │    ├─► cache-control → CacheControl
   │    ├─► content-disposition → ContentDisposition
   │    ├─► content-encoding → ContentEncoding
   │    ├─► content-type → ContentType
   │    ├─► x-amz-tagging → Tagging
   │    ├─► mtime → 覆盖源 ModTime
   │    ├─► object-lock-* → ObjectLockMode/RetainUntilDate/LegalHold
   │    └─► 其余 → Metadata (x-amz-meta-*)
   ├─► metaMtime (mtime 元数据，用于 SetModTime)
   ├─► MD5 校验
   │    ├─► ContentMD5 = base64(MD5)
   │    └─► 多分片/SSE 时: Metadata[metaMD5Hash] = base64(MD5)
   ├─► ContentType (自动检测)
   ├─► ContentLength
   ├─► RequestPayer
   └─► SSE (ServerSideEncryption / SSEKMSKeyId / SSECustomer*)
```

> **注意**：`prepareUpload` 阶段只构造请求结构体，**还没有签名也没有发出请求**。

### 4.5 单分片上传 — uploadSinglepartPutObject

[Object.uploadSinglepartPutObject](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L4704-L4735)

```go
func (o *Object) uploadSinglepartPutObject(ctx context.Context,
    req *s3.PutObjectInput, size int64, in io.Reader) (
    etag string, lastModified time.Time, versionID *string, err error) {

    req.Body = io.NopCloser(in)

    // 无符号 payload 模式：替换签名中间件，不计算 payload hash
    if o.fs.opt.UseUnsignedPayload.Value {
        options = append(options, s3.WithAPIOptions(
            v4signer.SwapComputePayloadSHA256ForUnsignedPayloadMiddleware,
        ))
    }

    // 单分片不重试（Reader 只能读一次）
    options = append(options, func(s3opt *s3.Options) {
        s3opt.RetryMaxAttempts = 1
    })

    var resp *s3.PutObjectOutput
    err = o.fs.pacer.CallNoRetry(func() (bool, error) {
        resp, err = o.fs.c.PutObject(ctx, req, options...)
        return o.fs.shouldRetry(ctx, err)
    })

    return *resp.ETag, time.Now(), resp.VersionId, nil
}
```

**签名参与的调用链**（单分片 SDK 方式）：

```
uploadSinglepartPutObject
   │
   ├─► req.Body = io.NopCloser(in)      包装 reader
   │
   ├─► 可选: SwapComputePayloadSHA256ForUnsignedPayloadMiddleware
   │       (替换签名中间件，不计算 payload hash，避免 seek body)
   │
   └─► o.fs.c.PutObject(ctx, req, options...)
         │
         ├─► Initialize 中间件
         ├─► Serialize 中间件
         ├─► Build 中间件
         │      └─► 将 Body 流式化
         ├─► Signing 中间件  ◄─── 【签名发生在这里】
         │      ├─► 计算 payload hash (如果启用)
         │      │    注意：流式 reader 无法 seek，所以
         │      │    - 普通模式：需要先把 body 全读一遍算 hash
         │      │    - UnsignedPayload 模式：跳过 payload hash
         │      └─► 计算 v4/v2/IAM 签名
         │      └─► 写入 Authorization 头
         ├─► Retry 中间件 (被设为 1 次，即不重试)
         └─► Send 中间件 → 发送 HTTP PUT 请求
```

**关键问题**：
- 单分片上传如果使用 `io.Reader`（不可 seek），SDK 默认的 payload hash 计算会要求读取整个 body 两次（一次算 hash，一次发送）。
- **UnsignedPayload 模式**通过 `SwapComputePayloadSHA256ForUnsignedPayloadMiddleware` 替换签名中间件，跳过 payload SHA256 计算，改为使用 `UNSIGNED-PAYLOAD` 标识，避免 seek 问题。

### 4.6 预签名 URL 上传 — uploadSinglepartPresignedRequest

除了 SDK 直接上传，rclone 还支持预签名 URL 上传方式：

[uploadSinglepartPresignedRequest](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L4738-L4796)

```go
func (o *Object) uploadSinglepartPresignedRequest(ctx context.Context,
    req *s3.PutObjectInput, size int64, in io.Reader) (...) {

    // 1. 用 SDK 生成预签名 URL (包含签名)
    putReq, err := s3.NewPresignClient(o.fs.c).PresignPutObject(ctx, req,
        s3.WithPresignExpires(15*time.Minute))
    //    └─► 内部同样走 Signing 中间件，生成预签名 URL
    //         签名参数放在 URL query 中 (X-Amz-Signature 等)

    // 2. 用原生 http.Client 发送 PUT 请求
    httpReq, _ := http.NewRequestWithContext(ctx, "PUT", putReq.URL, in)
    httpReq.Header = putReq.SignedHeader
    httpReq.ContentLength = size

    resp, err := o.fs.srv.Do(httpReq)   // 原生 HTTP 客户端，不走 SDK 中间件
    // ... 解析响应
}
```

**预签名方式的特点**：
- 签名在 `PresignPutObject` 阶段一次性完成（生成带签名的 URL）
- 实际上传使用原生 `http.Client`，**不再经过 SDK 签名中间件**
- 签名有效期 15 分钟

### 4.7 多分片上传 — uploadMultipart

当文件较大（>= UploadCutoff）或大小未知时，使用多分片上传：

1. **CreateMultipartUpload** — 初始化分片上传，获取 UploadID
2. **UploadPart** — 并发上传各个分片（受 `--transfers` 控制）
3. **CompleteMultipartUpload** — 合并所有分片

每个分片的 UploadPart 请求都独立经过 SDK 中间件栈，**每个分片都会单独签名**。

多分片还通过 `OpenChunkWriter` 接口（`fs.ChunkWriter`）支持流式写入。

### 4.8 上传后处理

Update 函数在上传完成后进行：

- **HEAD 校验**（默认）：调用 `headObject` 获取对象元数据，确认上传成功
- **跳过 HEAD**（`--no-head`）：直接根据上传响应构造元数据
- **ETag 校验**（多分片 + `--use-multipart-etag`）：比较期望 ETag 与实际 ETag
- **Object Lock 设置**（如启用）：通过 PutObjectRetention / PutObjectLegalHold 单独 API 设置

---

## 五、下载 (Open) — 签名与实现

### 5.1 接口映射

| `fs.Object` 接口 | S3 后端实现 | 底层 S3 API | 签名方式 |
|-----------------|-------------|------------|---------|
| `Object.Open(ctx, options...)` | `Object.Open` | GetObject | Signing 中间件自动签名 |
| `Object.Hash(ctx, type)` | 从缓存读取 / headObject | HeadObject | 同上 |
| `Fs.NewObject(ctx, remote)` | `headObject` | HeadObject | 同上 |

### 5.2 Object.Open — 下载入口

[Object.Open](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L4301-L4400)

```go
func (o *Object) Open(ctx context.Context, options ...fs.OpenOption) (in io.ReadCloser, err error) {
    bucket, bucketPath := o.split()

    // 自定义下载 URL（如 CloudFront CDN）
    if o.fs.opt.DownloadURL != "" {
        return o.downloadFromURL(ctx, bucketPath, options...)
    }

    // 1. 构造 GetObjectInput
    req := s3.GetObjectInput{
        Bucket:    &bucket,
        Key:       &bucketPath,
        VersionId: o.versionID,
    }
    // SSE-C / RequesterPays 等参数...

    // 2. 处理 OpenOption
    fs.FixRangeOption(options, o.bytes)
    for _, option := range options {
        switch option.(type) {
        case *fs.RangeOption, *fs.SeekOption:
            req.Range = &value      // Range: bytes=start-end
        case *fs.HTTPOption:
            APIOptions = append(APIOptions, smithyhttp.AddHeaderValue(key, value))
        }
    }

    // 3. 调用 GetObject (pacer 限流重试)
    var resp *s3.GetObjectOutput
    err = o.fs.pacer.Call(func() (bool, error) {
        resp, err = o.fs.c.GetObject(ctx, &req, s3.WithAPIOptions(APIOptions...))
        return o.fs.shouldRetry(ctx, err)
    })

    // 4. 解析大小 (ContentLength 或 ContentRange)
    size := resp.ContentLength
    if resp.ContentRange != nil { /* 解析 total size */ }

    // 5. 更新本地元数据缓存
    o.setMetaData(&head)

    // 6. gzip 解压处理
    if content-encoding == "gzip" && o.fs.opt.Decompress {
        return readers.NewGzipReader(resp.Body)
    }

    return resp.Body, nil
}
```

### 5.3 签名参与的完整调用链

```
Object.Open(ctx, options...)
   │
   ├─► 构造 GetObjectInput (Bucket, Key, VersionId, Range, SSE-C, ...)
   ├─► fs.FixRangeOption(options, o.bytes)   规范化范围
   │
   ├─► o.fs.pacer.Call(func() {              限速 + 重试
   │     │
   │     └─► o.fs.c.GetObject(ctx, &req, ...)
   │           │
   │           ├─► Initialize 中间件
   │           ├─► Serialize 中间件
   │           │     └─► 将 Range / SSE-C 等转为 HTTP 头
   │           ├─► Build 中间件
   │           ├─► Signing 中间件  ◄─── 【签名发生在这里】
   │           │      ├─► 从 CredentialsProvider 获取凭证
   │           │      ├─► 对请求头 + 规范 URL 计算签名字符串
   │           │      ├─► 调用 HTTPSignerV4.SignHTTP()
   │           │      └─► 写入 Authorization: AWS4-HMAC-SHA256 ...
   │           ├─► Retry 中间件
   │           └─► Send 中间件 → GET 请求发出
   │
   ├─► 解析响应: ContentLength / ContentRange / ETag / Last-Modified / Metadata
   ├─► o.setMetaData(&head)    更新本地缓存
   └─► 返回 resp.Body (io.ReadCloser)
```

### 5.4 Range 请求支持

S3 后端原生支持字节范围请求：

- **`fs.RangeOption`** / **`fs.SeekOption`** → 转换为 HTTP `Range: bytes=start-end` 头
- `fs.FixRangeOption(options, o.bytes)` — 规范化范围（处理负数偏移、边界检查）
- 响应通过 `Content-Range` 头确定实际返回范围和总大小

签名时，Range 头作为请求头的一部分参与签名计算。

### 5.5 元数据缓存

`Object` 结构体缓存了对象元数据，避免重复调用 HeadObject：

| 缓存字段 | 类型 | 何时填充 |
|---------|------|---------|
| `md5` | `string` | List 响应 / Open 响应 / headObject |
| `bytes` | `int64` | List 响应 / Open 响应 / headObject |
| `lastModified` | `time.Time` | List 响应 / Open 响应 / headObject |
| `meta` | `map[string]string` | Open 响应 / headObject |
| `mimeType` | `string` | Open 响应 / headObject |

- **列表时**：List 返回的 `Contents` 包含 Key、Size、LastModified、ETag → 部分填充
- **Open 时**：GetObject 响应头包含完整元数据 → 全量更新
- **主动获取**：`readMetaData()` → `headObject()` → HeadObject API

### 5.6 自定义下载 URL (download_url)

如果配置了 `--s3-download-url`（如 CloudFront CDN），下载走独立路径：

- 使用 `downloadFromURL` 方法
- 通过原生 HTTP 客户端访问 CDN URL
- **可能绕过 S3 签名**（取决于 CDN 配置）

---

## 六、Pacer 限流与重试

### 6.1 Pacer 作用

每个 `Fs` 实例持有一个 `*fs.Pacer`，基于令牌桶算法对 S3 API 调用进行限速和统一的重试逻辑：

- **初始配置**：`pacer.NewS3(pacer.MinSleep(minSleep))` — S3 专属的 pacer 配置
- **重试次数**：pacer 层重试 2 次（主要用于列表 XML 解析错误回退、特殊错误重试）
- **SDK 层重试**：SDK 自身也有重试机制（由 `RetryMaxAttempts` 控制，默认由 `LowLevelRetries` 配置）

### 6.2 shouldRetry — 重试决策

[Fs.shouldRetry](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L1276-L1343)

该函数判断错误是否需要重试，处理了大量 S3 特定的错误码：

- 5xx 错误 → 重试
- 429 SlowDown / TooManyRequests → 退避重试
- RequestTimeout / OperationAborted / InternalError → 重试
- 特殊的 403（请求过期）→ 重试（时钟漂移）
- 等等

> 每次重试都会重新进入 SDK 中间件栈，签名也会**重新计算**（因为签名中的时间戳会更新）。

---

## 七、关键数据结构

### 7.1 Fs 结构体

[s3.go:1156-1174](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L1156-L1174)

| 字段 | 类型 | 用途 |
|------|------|------|
| `name` | `string` | 远程名称 |
| `root` | `string` | 根路径（bucket + 目录前缀） |
| `opt` | `Options` | 解析后的配置选项 |
| `ci` | `*fs.ConfigInfo` | 全局配置 |
| `c` | `*s3.Client` | AWS SDK S3 客户端（含签名器） |
| `rootBucket` | `string` | root 中的 bucket 部分 |
| `rootDirectory` | `string` | root 中的目录前缀部分 |
| `cache` | `*bucket.Cache` | bucket 存在性缓存 |
| `pacer` | `*fs.Pacer` | API 调用限速与重试 |
| `srv` | `*http.Client` | 原生 HTTP 客户端（预签名上传用） |
| `srvRest` | `*rest.Client` | REST 客户端 |
| `etagIsNotMD5` | `bool` | ETag 是否非 MD5（SSE-KMS / SSE-C / 目录桶） |
| `features` | `*fs.Features` | 可选功能集 |

### 7.2 Object 结构体

[s3.go:1177-1202](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L1177-L1202)

| 字段 | 类型 | 用途 |
|------|------|------|
| `fs` | `*Fs` | 所属 Fs 实例 |
| `remote` | `string` | 远程路径 |
| `md5` | `string` | MD5 哈希（从 ETag 来） |
| `bytes` | `int64` | 对象大小 |
| `lastModified` | `time.Time` | 最后修改时间 |
| `meta` | `map[string]string` | 用户元数据（小写 key） |
| `mimeType` | `string` | MIME 类型 |
| `versionID` | `*string` | 版本 ID（版本化 bucket） |
| `storageClass` | `*string` | 存储层级 |
| `cacheControl` 等 | `*string` | 系统元数据 |
| `objectLock*` | `*string / *time.Time` | Object Lock 相关 |

---

## 八、统一后端接口调用汇总

下表列出 S3 客户端后端对 `fs.Fs` / `fs.Object` 统一接口的实现，以及签名如何参与每个 API 调用。

### 8.1 fs.Fs 接口

| 方法 | 实现位置 | 底层 S3 API | 签名方式 |
|------|---------|------------|---------|
| `Name()` | [Fs.Name](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L1240-L1242) | - | - |
| `Root()` | [Fs.Root](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L1245-L1247) | - | - |
| `String()` | [Fs.String](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L1250-L1257) | - | - |
| `Precision()` | [Fs.Precision](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L1260-L1262) | - | - |
| `Hashes()` | [Fs.Hashes](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L1265-L1273) | - | - |
| `Features()` | 由 `features` 字段提供 | - | - |
| **`List(ctx, dir)`** | [Fs.List](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L2654-L2656) | ListObjectsV2 | SDK Signing 中间件 |
| **`ListP(ctx, dir, cb)`** | [Fs.ListP](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L2671-L2695) | ListObjectsV2 | SDK Signing 中间件 |
| **`ListR(ctx, dir, cb)`** | [Fs.ListR](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L2713-L2761) | ListObjectsV2 | SDK Signing 中间件 |
| **`NewObject(ctx, r)`** | [Fs.NewObject](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L2634-L2651) | HeadObject | SDK Signing 中间件 |
| **`Put(ctx, in, src)`** | [Fs.Put](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L2764-L2771) | PutObject / Multipart | SDK Signing 中间件 |
| `Mkdir(ctx, dir)` | [Fs.Mkdir](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L3234-L3271) | CreateBucket / PutObject | SDK Signing 中间件 |
| `Rmdir(ctx, dir)` | [Fs.Rmdir](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L3288-L3317) | DeleteBucket / DeleteObject | SDK Signing 中间件 |
| `Copy(ctx, dst, src)` | [Fs.Copy](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L3407-L3544) | CopyObject / UploadPartCopy | SDK Signing 中间件 |
| `Purge(ctx, dir)` | - | DeleteObjects | SDK Signing 中间件 |
| `OpenChunkWriter(...)` | [Fs.OpenChunkWriter](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L4427-L4536) | CreateMultipartUpload + UploadPart | SDK Signing 中间件 |

### 8.2 fs.Object 接口

| 方法 | 实现位置 | 底层 S3 API | 签名方式 |
|------|---------|------------|---------|
| `Fs() / Remote() / ModTime() / Size()` | 字段直接返回 | - | - |
| **`Open(ctx, opts)`** | [Object.Open](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L4301-L4400) | GetObject | SDK Signing 中间件 |
| **`Update(ctx, in, src)`** | [Object.Update](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L5022-L5105) | PutObject / Multipart | SDK Signing 中间件 |
| **`Remove(ctx)`** | [Object.Remove](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L5108-L5129) | DeleteObject | SDK Signing 中间件 |
| `SetModTime(ctx, t)` | [Object.SetModTime](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L3918-L3941) | CopyObject (复制自身更新元数据) | SDK Signing 中间件 |
| `Hash(ctx, type)` | [Object.Hash](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L3944-L3990) | 缓存 / HeadObject | SDK Signing 中间件 |
| `Storable()` | 返回 `true` | - | - |
| `MimeType(ctx)` | [Object.MimeType](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L3993-L4013) | 缓存 / HeadObject | SDK Signing 中间件 |
| `Metadata(ctx)` | [Object.Metadata](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go#L4016-L4065) | 缓存 / HeadObject | SDK Signing 中间件 |

---

## 九、关键文件索引

| 文件 | 职责 |
|------|------|
| [backend/s3/s3.go](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/s3.go) | S3 客户端主文件：Fs/Object 定义、NewFs、List、Put、Open、认证连接等 |
| [backend/s3/providers.go](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/providers.go) | Provider 配置加载与 Quirks 定义 |
| [backend/s3/v2sign.go](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/v2sign.go) | S3 v2 签名实现 (HTTPSignerV4 接口) |
| [backend/s3/ibm_signer.go](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/ibm_signer.go) | IBM IAM Token 签名实现 (HTTPSignerV4 接口) |
| [backend/s3/setfrom.go](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/backend/s3/setfrom.go) | 结构体字段拷贝工具（代码生成） |
| [fs/types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/fs/types.go) | 统一后端接口定义 (Fs / Object / Directory) |
| [fs/pacer.go](file:///d:/fz/0601-2/solo-dogfeeding/code/41-rclone/fs/pacer.go) | Pacer 限流与重试抽象 |
