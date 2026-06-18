# Compress 压缩层代码分析

## 一、总体架构

Compress 是 rclone 的一个后端包装层（wrapper backend），它在底层真实文件系统之上实现透明压缩。核心设计采用策略模式，通过 `compressionModeHandler` 接口抽象不同压缩算法（gzip/zstd），每个逻辑文件在物理上对应 **两个对象**：一个数据文件和一个元数据文件。

关键文件：
- [compress.go](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go) — 主逻辑、Fs/Object 包装、核心函数
- [gzip_handler.go](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/gzip_handler.go) — gzip 压缩策略实现
- [zstd_handler.go](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/zstd_handler.go) — zstd 压缩策略实现
- [szstd_helper.go](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/szstd_helper.go) — zstd 可寻址格式支持（seekable zstd）
- [uncompressed_handler.go](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/uncompressed_handler.go) — 未压缩模式处理
- [unknown_handler.go](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/unknown_handler.go) — 未知模式兜底

---

## 二、压缩对象映射

### 2.1 逻辑对象 → 物理对象的映射关系

一个逻辑文件（用户视角）对应底层的 **两个物理文件**：

| 类型 | 命名规则 | 说明 |
|------|----------|------|
| 数据文件 | `{remote}.{base64(size)}.{gz\|zst\|bin}` | 存储实际内容（压缩或未压缩） |
| 元数据文件 | `{remote}.json` | JSON 格式，存储原始大小、MD5、MIME、压缩元数据 |

命名规则核心实现在：

- `makeDataName()` 在 [compress.go#L359-L370](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L359-L370)
- `makeMetadataName()` 在 [compress.go#L339-L341](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L339-L341)

```go
// 压缩 gzip:  file.txt  →  file.txt.AAAAAAAAAAA.gz
// 压缩 zstd:  file.txt  →  file.txt.AAAAAAAAAAA.zst
// 未压缩:     file.txt  →  file.txt.bin
// 元数据:     file.txt  →  file.txt.json
```

其中大小编码采用 **int64 → 8字节小端 → base64 RawURLEncoding**，见 [int64ToBase64()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L298-L302) 和 [base64ToInt64()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L305-L311)。

### 2.2 文件名反向解析（物理 → 逻辑）

[processFileName()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L315-L336) 负责将物理文件名解析回逻辑文件名、扩展名和原始大小：

1. 从最后一个 `.` 分割出扩展名（`.gz` / `.zst` / `.bin`）
2. 若扩展名为 `.bin`，标记为未压缩（`origSize = -2`）
3. 否则用正则 `^(.+?)\.([A-Za-z0-9-_]{11})$` 匹配出 `{原始文件名}.{base64编码大小}`
4. 解码 base64 得到原始 int64 大小

### 2.3 Object 结构映射

[Object](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1074-L1081) 结构体持有三个核心引用：

```go
type Object struct {
    fs.Object          // 嵌入的底层数据对象（o.Object）
    f         *Fs      // 所属压缩 Fs
    mo        fs.Object// 元数据文件对象（延迟加载）
    moName    string   // 元数据文件名
    size      int64    // 缓存的原始大小
    meta      *ObjectMetadata // 元数据结构体（延迟加载）
}
```

对象构造有两种方式：
- [newObject()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1169-L1181) — 完整构造，持有数据对象、元数据对象和元数据结构体
- [newObjectSizeAndNameOnly()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1184-L1196) — 列表时轻量构造，仅缓存大小和元数据文件名，元数据按需延迟加载

### 2.4 目录列表的映射路径

目录列表时通过 `processEntries()` → `addData()` 完成物理→逻辑转换：

1. 底层 `ListP/ListR` 返回原始条目（包含 `.gz`/`.zst`/`.bin`/`.json`）
2. [processEntries()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L405-L420) 过滤掉 `.json` 元数据文件
3. 对每个数据文件调用 `addData()` → `processFileName()` 解析出逻辑名和原始大小
4. 用 `newObjectSizeAndNameOnly()` 构造轻量 Object 返回给上层

---

## 三、大小展示关键路径

Compress 层展示给用户的始终是 **未压缩原始大小**，而非压缩后的物理大小。大小来源有两条路径：

### 路径一：从文件名编码中解析（列表场景，O(1)）

在 `addData()` [compress.go#L381-L392](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L381-L392) 中：

```go
func (f *Fs) addData(entries *fs.DirEntries, o fs.Object) {
    origFileName, _, size, err := processFileName(o.Remote(), f.modeHandler)
    if size == -2 { // 未压缩 .bin 文件
        size = o.Size()  // 直接用物理文件大小
    }
    *entries = append(*entries, f.newObjectSizeAndNameOnly(o, metaName, size))
}
```

此时 Object 的 `size` 字段直接被设置为解析出的原始大小，**无需读取元数据文件**，保证列表操作高效。

### 路径二：从元数据 JSON 中读取（精确场景，按需加载）

当调用 `Object.Size()` [compress.go#L1252-L1257](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1252-L1257) 时：

```go
func (o *Object) Size() int64 {
    if o.meta == nil {
        return o.size       // 列表场景的缓存值
    }
    return o.meta.Size      // 元数据加载后的精确值
}
```

元数据加载触发路径：
- [NewObject()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L495-L515) — 主动调用 `readMetadata()` 读取 `.json` 并解析
- [loadMetadataIfNotLoaded()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1209-L1218) — Open/Hash/MimeType 等操作时按需懒加载

元数据结构 [ObjectMetadata](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1064-L1071)：

```go
type ObjectMetadata struct {
    Mode                    int                 // 压缩模式: 0=未压缩, 2=gzip, 4=zstd
    Size                    int64               // ★ 未压缩原始大小
    MD5                     string              // 原始文件 MD5
    MimeType                string              // 原始 MIME 类型
    CompressionMetadataGzip *sgzip.GzipMetadata // gzip 块索引（用于随机读）
    CompressionMetadataZstd *SzstdMetadata      // zstd 块索引（用于随机读）
}
```

不同压缩模式从元数据中取大小的策略由 handler 实现：
- gzip: [gzipModeHandler.newObjectGetOriginalSize()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/gzip_handler.go#L45-L50) 从 `CompressionMetadataGzip.Size` 取
- zstd: [zstdModeHandler.newObjectGetOriginalSize()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/zstd_handler.go#L45-L50) 从 `CompressionMetadataZstd.Size` 取

---

## 四、读写包装关键路径

### 4.1 写入（Put）流程

写入的核心决策是 **是否压缩**，由启发式检测 `checkCompressAndType()` 完成。

#### 步骤 1：可压缩性检测

[checkCompressAndType()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L519-L534)：

1. 读取前 `heuristicBytes`（1 MiB）数据
2. 用 `mimetype.Detect()` 检测 MIME 类型
3. 调用对应 handler 的 `isCompressible()` 对这 1 MiB 试压缩，计算压缩比
4. 若压缩比 > `minCompressionRatio`（1.1）则判定为可压缩
5. 将已读的 1 MiB 与剩余数据通过 `io.MultiReader` 拼接重新输出

#### 步骤 2：压缩上传（可压缩情况）

以 gzip 为例，[gzipModeHandler.putCompress()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/gzip_handler.go#L93-L183)：

```
原始输入 io.Reader
    │
    ├─► io.TeeReader ──► md5.New()               // 计算原始文件 MD5 存入元数据
    │
    └─► io.Pipe() 管道
          │
          ├─ goroutine: sgzip.NewWriterLevel()    // 在协程中异步压缩
          │     └─► 将压缩后的字节写入 pipeWriter
          │     └─► 压缩完成后通过 channel 返回 gz.MetaData()（含块索引、原始大小）
          │
          └─► pipeReader
                │
                ├─ accounting 包装（进度统计）
                ├─ io.TeeReader ──► hash.MultiHasher  // 可选：计算压缩后数据的 hash 校验
                │
                └─► Fs.rcat() ──► 上传到远端（数据文件名含编码大小）
```

上传完成后：
- 从 channel 接收压缩元数据（块索引用于随机读）
- 构造 `ObjectMetadata` 并序列化为 JSON
- 通过 `putMetadata()` 上传 `.json` 元数据文件

完整编排见 [putWithCustomFunctions()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L686-L712)，它保证数据文件和元数据文件要么都上传成功，要么清理已上传的部分。

#### 步骤 3：未压缩上传（不可压缩情况）

[putUncompress()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L614-L656)：
- 数据直接写入 `{remote}.bin`，文件名不编码大小
- 仍然生成 `.json` 元数据（Mode=Uncompressed，记录 MD5 和 MIME）

#### rcat 的三种上传策略

[rcat()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L563-L606) 根据底层能力和文件大小选择上传方式：
1. **小文件**（< RAMCacheLimit，默认 20 MiB）：读入内存，用普通 `Put` 上传
2. **支持流式**：直接走 `PutStream`
3. **不支持流式**：先写入本地临时文件，再用普通 `Put` 上传

### 4.2 读取（Open）流程

[Object.Open()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L1336-L1359) 是读取入口：

```go
func (o *Object) Open(ctx context.Context, options ...fs.OpenOption) (io.ReadCloser, error) {
    // 1. 确保元数据已加载（包含块索引）
    err = o.loadMetadataIfNotLoaded(ctx)
    
    // 2. 未压缩模式：直接透传到底层 Object.Open
    if o.meta.Mode == Uncompressed {
        return o.Object.Open(ctx, options...)
    }
    
    // 3. 解析 SeekOption / RangeOption 得到 offset 和 limit
    var offset, limit int64 = 0, -1
    
    // 4. 用 ChunkedReader 包装底层对象（支持分块预读、缓存）
    chunkedReader := chunkedreader.New(ctx, o.Object, initialChunkSize, maxChunkSize, chunkStreams)
    
    // 5. 委托给对应 handler 的 openGetReadCloser()
    return o.f.modeHandler.openGetReadCloser(ctx, o, offset, limit, chunkedReader, chunkedReader, options...)
}
```

#### gzip 读取路径

[gzipModeHandler.openGetReadCloser()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/gzip_handler.go#L53-L81)：

- **offset == 0**：使用 `sgzip.NewReader(cr)` 流式顺序解压
- **offset != 0**：使用 `sgzip.NewReaderAt(cr, meta, offset)` 随机读取
  - 依赖 gzip 写入时生成的 `GzipMetadata` 块索引，定位到对应压缩块
  - 只解压需要的块，避免全量解压

最后用 `io.LimitReader` 处理 limit，再用 `ReadCloserWrapper` 包装 Reader + Closer 返回。

#### zstd 读取路径

[zstdModeHandler.openGetReadCloser()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/zstd_handler.go#L53-L81)：

- **offset == 0**：使用标准 `zstd.NewReader(cr)` 流式解压
- **offset != 0**：使用自定义 [NewReaderAtSzstd()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/szstd_helper.go#L134-L162)（基于 seekable zstd 格式）

Szstd 随机读实现（[SzstdReaderAt.ReadAt()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/szstd_helper.go#L188-L318)）：

1. 根据 `SzstdMetadata.BlockData` 块偏移表，计算 offset~endOff 覆盖哪些块
2. 对每个涉及的块，启动 goroutine 并行解码（通过信号量限制为 CPU 核数）
3. 按块索引顺序收集解码结果，拼接出请求的数据范围

### 4.3 Handler 策略接口

[compressionModeHandler](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L126-L164) 是读写压缩逻辑的策略接口：

| 方法 | 职责 |
|------|------|
| `processFileNameGetFileExtension()` | 返回该模式的文件扩展名（`.gz`/`.zst`） |
| `newObjectGetOriginalSize()` | 从元数据中提取原始文件大小 |
| `isCompressible()` | 启发式检测数据是否值得压缩 |
| `putCompress()` | 压缩并上传数据文件，返回对象+元数据 |
| `openGetReadCloser()` | 打开压缩文件并返回解压后的 ReadCloser |
| `putUncompressGetNewMetadata()` | 未压缩上传时构造元数据 |
| `newMetadata()` | 构造 ObjectMetadata（包装算法特定元数据） |

Handler 在 [NewFs()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-rclone/backend/compress/compress.go#L222-L234) 中根据配置模式实例化：

```go
switch compressionMode {
case Gzip:         modeHandler = &gzipModeHandler{}
case Zstd:         modeHandler = &zstdModeHandler{}
case Uncompressed: modeHandler = &uncompressedModeHandler{}
default:           modeHandler = &unknownModeHandler{}
}
```

---

## 五、数据流全景图

```
写入路径:
  用户 Put(src)
      │
      ▼
  checkCompressAndType() ──读1MiB试压缩──► 可压缩?
      │                                    │
      ├──────────── Yes ───────────────────┘
      │               │
      │               ▼
      │         modeHandler.putCompress()
      │               ├─ md5(TeeReader) 计算原始 MD5
      │               ├─ io.Pipe + goroutine 异步压缩
      │               ├─ rcat() 上传数据文件 {name}.{size}.{gz|zst}
      │               └─ 返回压缩元数据（块索引）
      │
      └──────────── No ──► putUncompress()
                              ├─ 直接上传数据文件 {name}.bin
                              └─ hash(TeeReader) 计算原始 MD5
      │
      ▼
  putMetadata() ──► 上传元数据文件 {name}.json（Mode/Size/MD5/MIME/块索引）

读取路径:
  用户 Open(offset, limit)
      │
      ▼
  loadMetadataIfNotLoaded() ──读 .json ──► 得到 Mode/Size/块索引
      │
      ├─ Mode == Uncompressed ──► o.Object.Open() 直接透传
      │
      ├─ Mode == Gzip ──► sgzip.NewReader / NewReaderAt ──► 解压数据流
      │
      └─ Mode == Zstd ──► zstd.NewReader / SzstdReaderAt ──► 解压数据流
                                      │
                                      └─ offset≠0: 按 BlockData 并行解码相关块
```
