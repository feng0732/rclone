# Google Drive 后端代码理解与串联分析

## 一、整体架构概述

Google Drive 后端核心代码位于 `backend/drive/drive.go`，通过 `Fs` 结构体封装了与 Google Drive API 交互的全部能力。三个核心模块——**文件查询**、**共享盘处理**、**变更检测**——通过 **`dirCache`（目录缓存）** 作为中枢纽带串联工作，同时共享底层的 **`list()` 通用查询函数** 与 **`pacer` API 速率调节器**。

### 核心数据结构

**Fs 结构体** ([drive.go#L833-L856](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L833-L856))：
```go
type Fs struct {
    name             string
    root             string
    opt              Options                    // 配置选项，含 TeamDriveID 等
    svc              *drive.Service             // v3 API 客户端
    rootFolderID     string                     // 根目录ID（共享盘时 = TeamDriveID）
    dirCache         *dircache.DirCache         // ★ 目录路径 ↔ ID 双向缓存
    pacer            *fs.Pacer                  // API 调用速率控制
    isTeamDrive      bool                       // ★ 是否为共享盘标志
    dirResourceKeys  *sync.Map                  // 目录ID → ResourceKey 缓存
    permissions      map[string]*drive.Permission
    // ...
}
```

**Options 结构体中的关键字段** ([drive.go#L785-L830](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L785-L830))：
- `TeamDriveID`：共享盘 ID（当非空时 `isTeamDrive = true`）
- `RootFolderID`：根目录 ID（优先级：配置值 > TeamDriveID > "root"）
- `SharedWithMe`、`TrashedOnly`、`StarredOnly`：查询过滤标志
- `ListChunk`：列表分页大小

---

## 二、文件查询（Listing）的完整链路

### 2.1 三层列表 API 设计

rclone 实现了三层列表接口，从简单到高效：

| 层级 | 函数 | 作用 | 代码位置 |
|------|------|------|----------|
| 1 简单 | `List()` | 单级目录列表，委托给 ListP | [drive.go#L1983-L1985](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1983-L1985) |
| 2 分页 | `ListP()` | 单级目录，回调式分页返回 | [drive.go#L2000-L2041](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2000-L2041) |
| 3 递归高效 | `ListR()` | 递归全量列表（fast-list），并发批处理 | [drive.go#L2221-L2332](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2221-L2332) |

### 2.2 通用查询核心 `list()` 函数

所有列表操作最终都汇聚到 **`list()` 函数** ([drive.go#L999-L1179](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L999-L1179))，它是三大模块的共同基石。

#### 关键流程：

```
输入参数:
  dirIDs []string      // 支持多目录ID同时查询（ListR 批处理核心）
  title string         // 精确文件名（为空则列出全部）
  directoriesOnly      // 仅目录
  filesOnly            // 仅文件
  trashedOnly          // 仅回收站
  includeAll           // 跳过 trashed 过滤
  fn listFn            // 每项回调
```

**步骤 1：构建查询条件 (Query Builder)**
- `trashed=false/true` 基础过滤
- **parents 查询**：用 `OR` 拼接多个目录，支持 `'dirID' in parents` 语法
  - 特殊情况：`SharedWithMe=true` 根目录时，使用 `sharedWithMe=true` 替代 parents
  - 特殊情况：`StarredOnly` 根目录时，附加 `starred=true`
- `title` 查询：精确匹配名称，同时对 Google Docs 尝试去掉扩展名匹配
- 类型过滤：`mimeType='folder'` 或 `mimeType!='folder'`
- **过滤器集成**：从 `filter.GetConfig(ctx)` 读取 `ModTimeFrom/To`，附加 `modifiedTime` 时间范围条件

**步骤 2：附加共享盘/资源密钥参数**
```go
list.SupportsAllDrives(true)
list.IncludeItemsFromAllDrives(true)
if f.isTeamDrive && !f.opt.SharedWithMe {
    list.DriveId(f.opt.TeamDriveID)    // ★ 共享盘ID注入
    list.Corpora("drive")               // ★ 查询域限定为共享盘
}
if resourceKeysHeader != "" {
    list.Header().Add("X-Goog-Drive-Resource-Keys", ...)  // 链接共享文件密钥
}
```

**步骤 3：分页遍历 & 快捷方式解析 & 回调**
- 循环调用 `Files.List()`，用 `NextPageToken` 翻页
- 对每项：
  - 名称编码转换 `Enc.ToStandardName()`
  - 若是快捷方式（`shortcutMimeType`）：调用 `resolveShortcut()` 解引用目标，ID 变为复合ID `actualID#shortcutID`
  - 名称大小写二次精确校验（Google 查询 `=` 不区分大小写）
  - 调用 `fn(item)` 回调

### 2.3 ListP：单级目录查询流程

[drive.go#L2000-L2041](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2000-L2041)

```
ListP(dir)
  → dirCache.FindDir(dir)           // 从路径查目录ID（dirCache 首次填充来自 List）
  → list([directoryID], ...)
    → 回调 itemToDirEntry(item)     // ★ 这里把目录ID反向写入 dirCache！
      → 如果是文件夹: dirCache.Put(remote, item.Id)
      → 转成 Object / Directory
  → teamDriveOK 校验（根目录为空时，验证共享盘访问）
```

### 2.4 ListR：递归高效列表（并发批处理）

[drive.go#L2221-L2332](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2221-L2332)

核心创新点在于 **多目录批查询** 与 **Bug 修复自适应机制**：

**并发模型**：
```
启动 Checkers 个 listRRunner goroutine
  ↓
in channel 接收 listREntry{id, path} 任务
  ↓
每个 runner 从 channel 批量拉取 grouping=50 个目录ID
  ↓
调用 list([50个dirIDs], ...) 一次 API 返回50个目录内容 ★ 减少API调用
  ↓
对每个返回项匹配 parents → 拼出完整 remote 路径
  ↓
如果 item 是目录 → 通过 sendJob() 继续加入 in channel 递归
```

**FastListBugFix 自适应机制** ([drive.go#L2153-L2186](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2153-L2186))：
- Google Drive API 存在已知 Bug：当 `(A in parents) OR (B in parents)` 查询可能返回空
- **检测**：一次批查询 >1 个目录且 `foundItems=false` → 触发修复
- **降级**：`grouping` 从 50 降为 1，把这批目录重新入队逐个查询
- **恢复**：若导致降级的所有目录最终都被证实是空目录，`grouping` 恢复为 50

### 2.5 单个文件查询

**NewObject / getRemoteInfoWithExport** ([drive.go#L4165-L4201](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L4165-L4201))：
```
getRemoteInfoWithExport(remote)
  → dirCache.FindPath(remote)       // 拆成 父目录路径 + leaf 文件名
  → list([父目录ID], leaf, ...)     // 精确文件名查询
  → 对 Google Docs 额外匹配 exportName（带扩展名的名称）
```

**FindLeaf**（dirCache 用来查找子目录ID）([drive.go#L1730-L1751](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1730-L1751))：
```
FindLeaf(pathID, leaf)
  → list([pathID], leaf, directoriesOnly=true)
  → 匹配 name 或 exportName
```

---

## 三、共享盘（Shared Drive / Team Drive）处理逻辑

### 3.1 共享盘初始化与判定

**初始化阶段 NewFs** ([drive.go#L1426-L1507](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1426-L1507))：

```
f.isTeamDrive = (opt.TeamDriveID != "")
                ↓
确定 rootFolderID 优先级:
  1. opt.RootFolderID (配置项)
  2. opt.TeamDriveID  ← ★ 共享盘时直接用其作为根目录ID
  3. 调用 getRootID() 获取 "root" 对应的实际ID
```

### 3.2 共享盘列表 - listTeamDrives()

[drive.go#L3471-L3491](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3471-L3491)

配置向导中调用，获取用户所有共享盘供选择：
```go
listTeamDrives:
  f.svc.Drives.List().PageSize(100)   // 使用 Drives.List 而非 Files.List
  → 分页收集所有 *drive.Drive
  → 返回给 ConfigChoose 选择
```

### 3.3 共享盘参数注入（贯穿所有 API 调用）

共享盘的特殊性在于**每次 API 调用必须附加三个关键参数**，代码中以一致模式出现：

| 参数 | 值 | 作用 |
|------|----|------|
| `SupportsAllDrives(true)` | 固定 true | 告知 API 本次调用支持共享盘语法（2020 年后 Drive API 要求） |
| `IncludeItemsFromAllDrives(true)` | 固定 true | 返回结果中包含共享盘项目 |
| `DriveId(TeamDriveID)` | 共享盘ID | 限定查询范围在指定共享盘内 |
| `Corpora("drive")` | "drive" | 查询语料库域，共享盘用 "drive"，个人盘默认 "user" |

**注入位置分布**：

1. **文件查询 list()** ([drive.go#L1101-L1106](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1101-L1106))：
   ```go
   list.SupportsAllDrives(true)
   list.IncludeItemsFromAllDrives(true)
   if f.isTeamDrive && !f.opt.SharedWithMe {
       list.DriveId(f.opt.TeamDriveID)
       list.Corpora("drive")
   }
   ```

2. **文件获取 getFile()** ([drive.go#L974-L983](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L974-L983))：
   ```go
   f.svc.Files.Get(ID).SupportsAllDrives(true)
   ```

3. **变更检测 changeNotifyStartPageToken()** ([drive.go#L3222-L3236](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3222-L3236))：
   ```go
   changes := f.svc.Changes.GetStartPageToken().SupportsAllDrives(true)
   if f.isTeamDrive {
       changes.DriveId(f.opt.TeamDriveID)
   }
   ```

4. **变更检测 changeNotifyRunner()** ([drive.go#L3249-L3253](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3249-L3253))：
   同样需要 `SupportsAllDrives / IncludeItemsFromAllDrives / DriveId`

5. **所有写操作**（Create/Update/Delete 等）：统一附加 `SupportsAllDrives(true)`

### 3.4 共享盘可达性校验 teamDriveOK()

[drive.go#L2961-L2975](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2961-L2975)

当 ListP / ListR 在共享盘根目录返回 0 条目时，不会直接返回空列表（可能是权限问题），而是调用：
```go
f.svc.Drives.Get(TeamDriveID).Fields("name,id,capabilities,createdTime,restrictions")
```
验证共享盘是否存在、当前账号是否有访问权限。失败则返回错误，避免静默失败。

### 3.5 About 接口的共享盘差异

[drive.go#L2977-L3007](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2977-L3007)
- **个人盘**：调用 `About.Get().Fields("storageQuota")` 获取存储配额
- **共享盘**：Drive API 不提供共享盘独立配额接口，直接返回空 `Usage{}`

---

## 四、变更检测（ChangeNotify）机制

### 4.1 顶层入口 ChangeNotify()

[drive.go#L3178-L3220](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3178-L3220)

用于 VFS 挂载时监听远端变更，使本地挂载点能感知其他客户端对 Drive 的修改。

```
ChangeNotify(ctx, notifyFunc, pollIntervalChan)
  ↓ 启动独立 goroutine
  ↓
  ① changeNotifyStartPageToken()  获取起始游标
  ↓
  ② 监听 pollIntervalChan 动态调整轮询间隔
     → 收到 0 间隔：停止 ticker
     → 收到非 0：创建/重置 Ticker
  ↓
  ③ Ticker 触发 → changeNotifyRunner() 拉取一批变更
     → 成功则 startPageToken = newStartPageToken
     → 失败仅打日志，不退出（下次 ticker 重试）
```

### 4.2 获取起始游标 changeNotifyStartPageToken()

[drive.go#L3222-L3236](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3222-L3236)

```go
f.svc.Changes.GetStartPageToken()
    .SupportsAllDrives(true)
    .DriveId(f.opt.TeamDriveID)       // ★ 共享盘：必须指定 DriveId
    .Do()
→ 返回 startPageToken（字符串游标，类似 "12345"）
```

这一步本质是**告诉 Google API：从"现在"这个时间点开始追踪变更**。

### 4.3 核心轮询 changeNotifyRunner()

[drive.go#L3238-L3323](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3238-L3323)

**调用参数构造**：
```go
f.svc.Changes.List(pageToken)
  .Fields("nextPageToken,newStartPageToken,
           changes(fileId,file(name,parents,mimeType))")  // ★ 只取路径计算所需字段
  .PageSize(ListChunk)
  .SupportsAllDrives(true)
  .IncludeItemsFromAllDrives(true)
  .DriveId(TeamDriveID)          // 共享盘
  .Spaces("appDataFolder")       // 若使用 appDataFolder
  .RestrictToMyDrive(!SharedWithMe)
```

**变更处理逻辑（核心串联 dirCache）**：

对每个 `change` 条目，**从两个方向计算受影响的路径**：

```
变更条目: { fileId, file{name, parents, mimeType} }
          ↓
          ├─ 方向 1：旧路径（通过 dirCache 反查）
          │   dirCache.GetInv(change.FileId)
          │     → 返回该 fileId 之前缓存的路径（如 "docs/report.docx"）
          │     → 根据 mimeType 判断是 EntryObject 还是 EntryDirectory
          │     → 加入 pathsToClear（旧位置需要通知失效）
          ↓
          └─ 方向 2：新路径（通过 parents 正向拼接）
              if file != nil（非删除）:
                for parent in file.Parents:
                  dirCache.GetInv(parent)  // 查父目录路径
                    → 拼接 path.Join(parentPath, fileName)
                    → 加入 pathsToClear（新位置需要通知）
              else（文件被删除）:
                仅依赖方向1的旧路径
          ↓
          对 pathsToClear 去重后逐一调用 notifyFunc(path, entryType)
```

**分页终止条件**：
- `NewStartPageToken != ""`：所有变更处理完毕，返回新游标给下次使用
- `NextPageToken != ""`：本批还有下一页，继续用新 pageToken 循环
- 两者都空：异常情况，直接返回

---

## 五、三大模块串联全景图

### 5.1 数据流向图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Google Drive API                             │
└─────────────────────────────────────────────────────────────────────┘
         ↑                    ↑                     ↑
         │ SupportsAllDrives  │ SupportsAllDrives   │ SupportsAllDrives
         │ DriveId/Corpora    │ DriveId             │ DriveId（共享盘参数）
         │                    │                     │
┌────────┴─────────┐  ┌──────┴──────────┐  ┌───────┴────────────────┐
│  list() 通用查询 │  │ listTeamDrives()│  │ Changes API 变更检测   │
│  ★ 文件查询核心  │  │  配置时调用一次 │  │  changeNotifyRunner() │
└────────┬─────────┘  └────────┬─────────┘  └───────┬────────────────┘
         │                     │                     │
         │ 写入                │ 配置初始化         │ 读 & 失效
         ▼                     ▼                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     dirCache (中枢神经)                              │
│  ┌───────────────────────┐     ┌─────────────────────────────┐      │
│  │ 路径 → ID (正向映射)   │     │ ID → 路径 (反向映射 GetInv) │      │
│  │ FindDir()/FindPath()  │◄────┤ 变更检测通过此定位旧路径     │      │
│  │ 列表操作时填充 Put()  │     │                             │      │
│  └───────────────────────┘     └─────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
         ↑                     ↑                     ↑
         │ 使用                │ 验证                │ 失效旧缓存
         │                     │                     │
┌────────┴─────────┐  ┌──────┴──────────┐  ┌───────┴────────────────┐
│  ListP / ListR   │  │  teamDriveOK()  │  │  notifyFunc(path,type) │
│  单层/递归列表   │  │ 空列表时二次校验│  │ VFS 层触发缓存失效    │
│  itemToDirEntry  │  │                 │  │ 触发重新 List 拉取    │
└──────────────────┘  └─────────────────┘  └────────────────────────┘
```

### 5.2 典型交互场景串联

#### 场景 A：首次挂载共享盘并执行完整列表

```
① 配置阶段（一次性）
   listTeamDrives() → 用户选定 TeamDriveID
        ↓
② NewFs 初始化
   isTeamDrive = true
   rootFolderID = TeamDriveID   ★ 共享盘ID直接作为根目录ID
        ↓
③ 首次 ListP("")
   dirCache.FindDir("") → 返回 rootFolderID
        ↓
   list([rootFolderID])
     → 附加: DriveId + Corpora("drive") + SupportsAllDrives
     → 附加: IncludeItemsFromAllDrives
        ↓
   itemToDirEntry 逐项转换
     → 遇到文件夹: dirCache.Put("docs", folderId)  ★ 填充正向缓存
        ↓
   若返回 0 条目 → teamDriveOK() 二次校验权限
        ↓
④ 调用 ListR("") 递归同步（使用 fast-list）
   listRRunner 批量拉取 50 个目录ID → list([50个IDs])
     → 每次同样附加共享盘参数
     → 若触发 FastListBugFix → grouping 降为 1 逐个重试
```

#### 场景 B：VFS 挂载后台，外部客户端新增文件

```
① 外部创建文件 folder/newfile.txt，Google Drive 内部记录 change 事件
        ↓
② ChangeNotify ticker 触发
   changeNotifyRunner(startPageToken)
     → Changes.List() 同样附加共享盘参数
        ↓
   对该 change 条目:
     1. dirCache.GetInv(fileId) → 空（新文件无旧路径）
     2. file.Parents = [folderId]
        dirCache.GetInv(folderId) → "folder"
        拼接新路径: "folder/newfile.txt"
        ↓
   notifyFunc("folder/newfile.txt", EntryObject)
        ↓
③ VFS 层收到通知，清除该路径的内核缓存
        ↓
④ 用户下次 ls folder → 触发 ListP("folder")
   dirCache.FindDir("folder") → 命中缓存得到 folderId
   list([folderId]) → 拉到 newfile.txt → 展示给用户
```

#### 场景 C：共享盘内移动文件夹

```
① 文件夹 "a/docs" 被移动到 "b/docs"
        ↓
② changeNotifyRunner 处理变更
   dirCache.GetInv(folderId) → "a/docs"（旧路径）
   parents = [newParentId]
   dirCache.GetInv(newParentId) → "b"
   拼接新路径: "b/docs"
        ↓
   notifyFunc("a/docs", EntryDirectory)  // 清除旧位置
   notifyFunc("b/docs", EntryDirectory)  // 标记新位置变更
        ↓
③ 后续所有子文件/目录也会产生 change 事件
   各自通过相同机制清除旧路径、标记新路径
   dirCache 的旧映射（a/docs/xxx → ID）仍存在，
   但 VFS 缓存失效后，下次 List 会重新发现正确位置
   新的 itemToDirEntry 会用 Put 覆盖写入新路径映射
```

---

## 六、关键设计模式总结

### 6.1 统一查询入口

`list()` 函数被设计为**极度通用**的查询引擎，支持：
- 单目录 / 多目录批查询（dirIDs 数组）
- 精确名称 / 全量枚举
- 目录/文件过滤
- 各种查询模式（SharedWithMe、TrashedOnly、StarredOnly）
- 时间过滤（通过 filter 包集成）

这种设计使 ListP、ListR、FindLeaf、NewObject 等所有需要查询的场景都能复用同一份查询构造逻辑，减少了共享盘参数遗漏的风险。

### 6.2 目录缓存作为中心枢纽

`dirCache.DirCache` 的双向映射能力是三大模块协作的关键：

| 操作 | 方向 | 使用者 | 作用 |
|------|------|--------|------|
| `Put(remote, id)` | 路径→ID + ID→路径 | 文件查询的 itemToDirEntry | 列表时填充缓存 |
| `FindDir(remote)` | 路径→ID | ListP/ListR 入口 | 将人类路径转为 API 需要的 ID |
| `GetInv(id)` | ID→路径 ★ 关键 | 变更检测 changeNotifyRunner | 从变更事件中的 ID 反推出路径 |

没有 dirCache，变更检测拿到的 `fileId` 将是无法被 VFS 层理解的无意义字符串。

### 6.3 共享盘参数的一致化模式

共享盘参数被**分散注入到每个独立的 API 调用点**（而非全局拦截器），这种模式：
- **优点**：灵活，每个调用可根据需要微调（如 SharedWithMe 模式下不加 DriveId）
- **缺点**：新增 API 调用时容易忘记附加参数，导致共享盘环境下出现 404 或找不到文件

代码中体现了较强的纪律性：所有对 `svc.Files.*`、`svc.Changes.*`、`svc.Drives.*` 的调用都能看到 `SupportsAllDrives(true)` 的身影。

### 6.4 自适应降级的鲁棒性设计

ListR 的 `FastListBugFix` 机制展示了对第三方 API 缺陷的优雅处理：
1. 不直接依赖 Google 修复 Bug
2. 实时检测异常（多目录查询返回空）
3. 动态降级（grouping 50→1）保证正确性
4. 自动恢复（确认是误报后 grouping 1→50）不损失长期性能

---

## 七、主要代码索引

| 功能模块 | 关键函数 | 代码位置 |
|---------|---------|----------|
| **Fs 初始化** | NewFs / newFs | [drive.go#L1349-L1507](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1349-L1507) |
| **通用查询** | list | [drive.go#L999-L1179](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L999-L1179) |
| **单级列表** | ListP | [drive.go#L2000-L2041](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2000-L2041) |
| **递归列表** | ListR / listRRunner | [drive.go#L2076-L2332](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2076-L2332) |
| **条目转换** | itemToDirEntry / newObjectWithInfo | [drive.go#L1642-L1704](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1642-L1704) + [drive.go#L2422-L2447](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2422-L2447) |
| **单文件查询** | getRemoteInfoWithExport / NewObject | [drive.go#L1708-L1727](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1708-L1727) + [drive.go#L4165-L4201](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L4165-L4201) |
| **共享盘列表** | listTeamDrives | [drive.go#L3471-L3491](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3471-L3491) |
| **共享盘校验** | teamDriveOK | [drive.go#L2961-L2975](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2961-L2975) |
| **变更总入口** | ChangeNotify | [drive.go#L3178-L3220](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3178-L3220) |
| **起始游标** | changeNotifyStartPageToken | [drive.go#L3222-L3236](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3222-L3236) |
| **变更处理** | changeNotifyRunner | [drive.go#L3238-L3323](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3238-L3323) |
| **快捷方式解引用** | resolveShortcut | [drive.go#L2391-L2417](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2391-L2417) |
| **目录叶子查找** | FindLeaf | [drive.go#L1730-L1751](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1730-L1751) |
