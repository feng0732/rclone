# OneDrive 后端代码关系：Drive Item 定位、分片上传与目录缓存

## 核心文件

| 文件 | 职责 |
|---|---|
| [onedrive.go](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go) | OneDrive 后端主文件，包含 Fs / Object 结构体、URL 构建函数、上传逻辑 |
| [api/types.go](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/api/types.go) | Microsoft Graph API 的请求/响应数据类型定义 |
| [dircache.go](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/lib/dircache/dircache.go) | 通用目录缓存库，维护 path ↔ directoryID 的双向映射 |

---

## 一、Drive Item 定位

### 1.1 标准化 ID（Normalized ID）

OneDrive API 中每个 drive item 拥有唯一 ID，但该 ID 仅在所属 Drive 内唯一。rclone 采用 **`driveID#itemID`** 格式作为标准化 ID，确保跨 Drive 场景（如"共享给我的"文件夹）下 ID 的全局唯一性。

- [Item.GetID()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/api/types.go#L436-L443)：若 item 含 RemoteItem，则从 `RemoteItem.ParentReference.DriveID + "#" + RemoteItem.ID` 拼接；否则从 `ParentReference.DriveID + "#" + ID` 拼接。
- [Fs.parseNormalizedID()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L2768-L2781)：将 `driveID#itemID` 拆解为 `itemID`、`driveID`、`rootURL`，用于后续 URL 构建。

### 1.2 URL 构建体系（四层）

rclone 通过一组 `newOptsCall*` 辅助函数将 normalizedID 或路径翻译为 Microsoft Graph API 的 REST 请求 URL。这套体系是 drive item 定位的核心，分为四个层次：

```
newOptsCallWithPath (最高层，自动选路)
    ├── newOptsCallWithIDPath (基于 ID + 相对路径)
    └── newOptsCallWithRootPath (基于绝对路径)

newOptsCall (最底层，基于 normalizedID 直接定位)
```

#### 1.2.1 `newOptsCall` — 基于 normalizedID 直接定位

[Fs.newOptsCall()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L2785-L2799)

```
URL 模板: https://{Endpoint}/drives/{driveID}/items/{itemID}/{route}
```

- 解析 normalizedID 得到 itemID、driveID、rootURL
- 若 driveID 非空，拼接完整 URL（含自定义 RootURL）；否则仅用 `/items/{itemID}/{route}`（相对路径，依赖 `srv` 的默认 RootURL）
- **使用场景**：已知 item ID 的操作，如下载（`GET /content`）、删除（`DELETE`）、移动（`PATCH`）、复制（`POST /copy`）

#### 1.2.2 `newOptsCallWithIDPath` — 基于 ID + 子路径定位

[Fs.newOptsCallWithIDPath()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L2811-L2843)

```
URL 模板 (国际版): https://{Endpoint}/drives/{driveID}/items/{parentID}:/{leaf}/{route}
URL 模板 (中国版): https://{Endpoint}/drives/{driveID}/items/{parentID}/children('{leaf}')/{route}
                  或 https://{Endpoint}/drives/{driveID}/items/{parentID}/children('@a1')/{route}?@a1=URLEncode("'{leaf}'")
```

- 以某个已知 ID 为锚点，在其下定位子项 leaf
- 中国区（Vnet Group）的 API 不支持 `:/{leaf}` 语法，改用 `children()` 方式
- `isPath` 参数决定 leaf 是否为多层路径（如 `a/b/c`）
- **使用场景**：`FindLeaf`（查找子目录）、`readMetaDataForPathRelativeToID`（基于父 ID 读取子路径元数据）

#### 1.2.3 `newOptsCallWithRootPath` — 基于绝对路径定位

[Fs.newOptsCallWithRootPath()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L2848-L2858)

```
URL 模板 (国际版): https://{Endpoint}/drives/{driveID}/root:/{path}/{route}
URL 模板 (中国版): https://{Endpoint}/drives/{driveID}/root/children('@a1')/{route}?@a1=URLEncode({path})
```

- 从 Drive 根目录出发，按路径逐级定位
- **使用场景**：当 dircache 中没有对应的 directoryID 时的 fallback

#### 1.2.4 `newOptsCallWithPath` — 智能选路（dircache 驱动）

[Fs.newOptsCallWithPath()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L2863-L2880)

这是最上层的 URL 构建函数，**目录缓存与 item 定位的交汇点**：

1. 如果 `path == ""`，直接返回 `/root{route}`
2. 调用 `f.dirCache.FindPath(ctx, path, false)` 将路径拆分为 `leaf` + `directoryID`
3. 若 dircache 命中（`directoryID` 非空），优先使用 `newOptsCallWithIDPath`（基于 ID + leaf 定位）
4. 若 dircache 未命中，fallback 到 `newOptsCallWithRootPath`（基于绝对路径定位）

**关键意义**：dircache 的命中与否直接决定 API 请求的方式。命中时使用 ID 路径模式，可正确处理"共享给我的"文件夹；未命中时退化到根路径模式，功能受限但可兜底。

### 1.3 readMetaDataForPath — 元数据读取中的定位策略

[Fs.readMetaDataForPath()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L967-L1030)

此方法体现了 OneDrive Personal vs Business 的定位差异：

- **非 Personal 或路径无斜杠**：直接用 `newOptsCallWithPath` 走标准选路
- **Personal 且路径含子目录**：需要处理"共享给我的"文件夹
  1. 先检查 dircache 是否已找到 root
  2. 若 dircache 有 root 的 ID，以 root ID 为锚点计算相对路径，调用 `readMetaDataForPathRelativeToID`
  3. 若 dircache 未找到 root，手动查询第一个目录的元数据获取其 ID，再以该 ID 为锚点

---

## 二、分片上传

### 2.1 上传策略选择

[Object.Update()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L2694-L2728) 根据文件大小选择上传方式：

```
size >= UploadCutoff (默认 10 MiB) 且 size > 0  →  uploadMultipart（分片上传）
0 <= size < UploadCutoff                           →  uploadSinglepart（单次上传）
size < 0                                           →  报错，不支持未知大小
```

相关常量：
- `UploadCutoff`：默认 10 MiB，最大不超过 4 MiB（`maxSinglePartSize`）
- `ChunkSize`：分片大小，默认 10 MiB，必须是 320 KiB 的整数倍

### 2.2 分片上传流程

[Object.uploadMultipart()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L2593-L2644)

```
1. createUploadSession  →  获取 uploadURL
2. 循环 uploadFragment  →  逐片上传
3. 最后一片返回 200/201 →  上传完成，获得完整 Item
4. setMetaData          →  更新 Object 元数据
5. updateMetadata       →  如果有权限元数据，额外更新
```

#### 2.2.1 创建上传会话

[Object.createUploadSession()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L2465-L2483)

- 调用 `o.fs.newOptsCallWithPath(ctx, o.remote, "POST", "/createUploadSession")` 构建请求 URL
- **注意**：这里 `newOptsCallWithPath` 依赖 dircache 查找目标文件所在目录的 ID，然后基于该 ID 构建创建上传会话的 URL
- 请求体可包含 `item` 字段（如 FileSystemInfo），在会话创建时一并设置元数据
- 返回 `CreateUploadResponse`，核心字段是 `UploadURL`（后续分片上传的目标地址）

#### 2.2.2 分片上传

[Object.uploadFragment()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L2517-L2574)

每个分片的上传直接使用 uploadURL（不再经过 dircache 或 URL 构建函数），按以下格式发 PUT 请求：

```
PUT {uploadURL}
Content-Range: bytes {start}-{end}/{totalSize}
Body: chunk data
```

- 使用 `o.fs.unAuth`（无认证客户端）发送，因为 uploadURL 自带临时认证
- **416 错误恢复**：收到 `416 Range Not Satisfiable` 时，调用 `getPosition()` 查询服务端期望的偏移量，计算 skip 值后重试当前分片
- **404 错误恢复**：上传会话可能存在最终一致性延迟，等待 5 秒后重试
- 上传完成时服务端返回 `200` 或 `201`，响应体为完整的 `api.Item`

#### 2.2.3 取消上传会话

[Object.cancelUploadSession()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L2577-L2589)

- 通过 `atexit.OnError` 注册，上传失败时自动 DELETE uploadURL
- 防止孤立的上传会话占用资源

### 2.3 单次上传

[Object.uploadSinglepart()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L2649-L2689)

- 适用于小文件（< UploadCutoff，上限 4 MiB）
- 调用 `o.fs.newOptsCallWithPath(ctx, o.remote, "PUT", "/content")` 构建请求
- 上传完成后需要额外调用 `fetchAndUpdateMetadata` 设置 modTime（因为单次上传不会自动携带修改时间）

---

## 三、目录缓存

### 3.1 数据结构

[DirCache](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/lib/dircache/dircache.go#L20-L32)

```go
type DirCache struct {
    cacheMu     sync.RWMutex
    cache       map[string]string    // path → directoryID
    invCache    map[string]string    // directoryID → path（反向映射）
    mu          sync.Mutex
    fs          DirCacher            // 后端实现的接口
    trueRootID  string               // Drive 绝对根的 ID
    root        string               // rclone 配置的根路径
    rootID      string               // 根路径对应的目录 ID
    rootParentID string              // 根路径的父目录 ID
    foundRoot   bool                 // 是否已找到根
}
```

- **双向映射**：`cache` 和 `invCache` 互为反向索引，既能从路径查 ID，也能从 ID 查路径
- **线程安全**：`cacheMu` 保护 cache/invCache 的读写，`mu` 保护 FindRoot/FindDir 等状态变更操作

### 3.2 DirCacher 接口

[DirCacher](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/lib/dircache/dircache.go#L38-L41)

OneDrive 后端实现了两个方法：

- [Fs.FindLeaf()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L1254-L1274)：在 pathID 对应的目录下查找名为 leaf 的子目录。调用 `readMetaDataForPathRelativeToID` 按 ID + leaf 查询 API。
- [Fs.CreateDir()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L1277-L1297)：在 dirID 下创建名为 leaf 的子目录。调用 `newOptsCall(dirID, "POST", "/children")` 发 POST 请求。

### 3.3 核心操作

| 方法 | 作用 |
|---|---|
| `FindPath(path, create)` | 将路径拆分为 `leaf` + `directoryID`，递归查找/创建父目录 |
| `FindDir(path, create)` | 查找路径对应的目录 ID，未找到时可选创建 |
| `FindRoot(create)` | 定位 rclone 配置的根路径，初始化缓存 |
| `Put(path, id)` | 写入双向映射 |
| `Get(path)` | 正向查找：path → ID |
| `GetInv(id)` | 反向查找：ID → path |
| `Flush()` | 清空所有缓存 |
| `FlushDir(dir)` | 清空指定目录及其子目录的缓存 |

### 3.4 缓存填充来源

目录缓存的填充发生在以下时机：

1. **NewFs 初始化**：[NewFs()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L1074-L1215) 调用 `dirCache.FindRoot()`，递归查找根路径过程中逐步填充中间目录
2. **目录列表**：[Fs.itemToDirEntry()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L1373-L1396) 遍历子项时，遇到文件夹自动调用 `f.dirCache.Put(remote, id)` 写入缓存
3. **创建目录**：`_findDir` 中 `CreateDir` 返回新目录 ID 后自动 `dc.Put(path, pathID)`
4. **查找路径**：`FindLeaf` 成功后自动 `dc.Put(path, pathID)`

---

## 四、三者协作关系

### 4.1 协作总览

```
                    ┌─────────────────────────┐
                    │    目录缓存 (DirCache)    │
                    │  path ↔ directoryID      │
                    └───────┬─────────┬────────┘
                            │         │
              FindPath/FindDir        │ 缓存命中？
                            │         │
                            ▼         ▼
               ┌────────────────────────────────┐
               │   Drive Item 定位 (URL 构建)    │
               │  newOptsCallWithPath (智能选路)  │
               │    ├─ 命中 → ID+Path 模式       │
               │    └─ 未命中 → RootPath 模式    │
               └──────────────┬─────────────────┘
                              │
                    构建 API 请求 URL
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
        读取元数据      创建上传会话       单次上传
   readMetaDataForPath  createUploadSession  uploadSinglepart
              │               │
              │               ▼
              │        获取 uploadURL
              │               │
              │               ▼
              │        ┌──────────────┐
              │        │  分片上传循环  │
              │        │ uploadFragment│
              │        │ (使用 uploadURL│
              │        │  不再依赖缓存) │
              │        └──────────────┘
              ▼
         setMetaData
        (更新 Object)
```

### 4.2 典型协作流程：上传文件

以 `Put → createObject → uploadMultipart` 为例：

```
步骤 1: Fs.Put(ctx, in, src)
    │
    ├─ 步骤 2: Fs.createObject(ctx, remote, modTime, size)
    │       │
    │       └─ dirCache.FindPath(ctx, remote, true)
    │           │
    │           ├─ 查缓存：path → directoryID
    │           ├─ 缓存未命中 → FindDir 递归查找
    │           │       │
    │           │       └─ FindLeaf(parentID, leaf)
    │           │           └─ readMetaDataForPathRelativeToID
    │           │               └─ newOptsCallWithIDPath(parentID, leaf)
    │           │
    │           └─ 返回 leaf + directoryID
    │
    ├─ 步骤 3: Object.Update(ctx, in, src)
    │       │
    │       └─ size >= UploadCutoff → uploadMultipart
    │               │
    │               ├─ createUploadSession
    │               │       └─ newOptsCallWithPath(ctx, o.remote, "POST", "/createUploadSession")
    │               │           │
    │               │           ├─ dirCache.FindPath(ctx, remote, false)  ← 再次依赖缓存
    │               │           ├─ 命中 → newOptsCallWithIDPath(directoryID, leaf)
    │               │           └─ 未命中 → newOptsCallWithRootPath(path)
    │               │
    │               ├─ uploadFragment 循环（直接用 uploadURL，不再查询缓存）
    │               │
    │               └─ setMetaData → 更新 Object 的 id/size/hash/modTime
    │
    └─ 返回上传结果
```

### 4.3 典型协作流程：读取文件元数据

```
Object.readMetaData(ctx)
    │
    └─ Fs.readMetaDataForPath(ctx, o.rootPath())
        │
        ├─ (非 Personal 或路径无斜杠)
        │   └─ newOptsCallWithPath → 依赖 dircache 选路
        │
        └─ (Personal 且路径含子目录)
            ├─ dirCache.RootID(ctx, false)  ← 检查 root 是否已知
            ├─ dirCache.FindDir(ctx, firstDir, false)  ← 查找首个子目录
            └─ readMetaDataForPathRelativeToID(baseID, relPath)
                └─ newOptsCallWithIDPath(baseID, relPath)
```

### 4.4 关键协作点总结

| 协作点 | 缓存的作用 | 定位的影响 | 上传的影响 |
|---|---|---|---|
| `newOptsCallWithPath` | 提供 directoryID，决定使用 ID+Path 还是 RootPath 模式 | 直接决定 API URL 构建方式 | createUploadSession 的 URL 取决于缓存是否命中 |
| `createObject` | FindPath 确保目标目录存在并获取其 ID | 需要目录 ID 才能定位上传目标 | 上传前必须先定位到正确目录 |
| `FindLeaf` | 查找子目录时需要 parentID（来自缓存） | 以 parentID 为锚点构建 URL | 创建上传会话前可能需要递归创建目录 |
| `itemToDirEntry` | 列目录时主动写入缓存 | 为后续操作提供 ID 映射 | 后续上传可命中缓存，减少 API 调用 |
| 分片上传循环 | **不依赖缓存** | 使用 uploadURL 直连 | 分片过程完全独立于缓存 |

### 4.5 缓存失效与一致性

- [Fs.DirCacheFlush()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-rclone/backend/onedrive/onedrive.go#L1988-L1992)：对外暴露的缓存清除接口
- `FlushDir(dir)`：删除/移动目录后清除受影响的缓存区域
- **ChangeNotify**：通过 delta API 轮询变更，检测到变化时通知上层，但不会主动更新缓存；上层收到通知后通常会触发 `DirCacheFlush` 清除过期数据

### 4.6 设计要点

1. **缓存驱动定位**：dircache 是 drive item 定位的"第一选择器"。命中缓存走 ID+Path 模式（支持共享文件夹），未命中则 fallback 到 RootPath 模式
2. **上传会话与缓存解耦**：createUploadSession 依赖缓存定位目标路径，但获取 uploadURL 后的分片上传完全脱离缓存，直接使用 URL 上传
3. **缓存是增量构建的**：从 NewFs 的 FindRoot 开始，通过列表操作逐步填充。首次操作可能触发多次 API 调用来逐级查找目录，后续操作可复用缓存快速定位
4. **OneNote 文件特殊处理**：FindLeaf 遇到 OneNote 文件时报错（因为它看起来像文件夹但不是），createUploadSession 遇到 `nameAlreadyExists` 错误时提示可能是 OneNote 文件
