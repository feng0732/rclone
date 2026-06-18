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

---

## 八、Drive API 调用参数差异全解

rclone 在调用 Google Drive 不同 API 时使用的参数集合各不相同，体现了对各 API 语义的精确适配。

### 8.1 七大类 API 调用参数矩阵

| API 调用 | 典型场景 | `SupportsAllDrives` | `DriveId` | `Corpora` | `IncludeItemsFromAllDrives` | `Spaces` | `RestrictToMyDrive` | 其他关键参数 |
|---------|---------|---------------------|-----------|-----------|----------------------------|----------|--------------------|-------------|
| **Files.List** | 文件列表查询 | ✅ 始终 true | 共享盘时设置 | 共享盘时 `"drive"` | ✅ 始终 true | appDataFolder 时设 | ❌ 无 | `Q`, `PageSize`, `Fields` |
| **Files.Get** | 单文件获取 | ✅ 始终 true | ❌ 不设 | ❌ 不设 | ❌ 无 | ❌ 无 | ❌ 无 | `Fields` |
| **Files.Create** | 创建文件/目录 | ✅ 始终 true | ❌ 不设 | ❌ 不设 | ❌ 无 | ❌ 无 | ❌ 无 | `Fields`, 父目录ID在 body |
| **Files.Update** | 更新文件/移动/恢复 | ✅ 始终 true | ❌ 不设 | ❌ 不设 | ❌ 无 | ❌ 无 | ❌ 无 | `Fields`, body 数据 |
| **Files.Delete** | 永久删除 | ✅ 始终 true | ❌ 不设 | ❌ 不设 | ❌ 无 | ❌ 无 | ❌ 无 | `Fields` 设空 |
| **Files.EmptyTrash** | 清空回收站 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | 无额外参数 |
| **Drives.List** | 列出所有共享盘 | ❌ 隐式支持 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | `PageSize`, `PageToken` |
| **Drives.Get** | 获取单个共享盘详情 | ❌ 隐式支持 | 路径参数 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | `Fields` |
| **About.Get** | 获取存储配额 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | `Fields("storageQuota")` |
| **Changes.GetStartPageToken** | 获取变更起始游标 | ✅ 始终 true | 共享盘时设置 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | 无 |
| **Changes.List** | 拉取变更列表 | ✅ 始终 true | 共享盘时设置 | ❌ 无 | ✅ 始终 true | appDataFolder 时设 | ✅ SharedWithMe 时 false | `PageSize`, `Fields` |

### 8.2 关键参数深度解读

**`SupportsAllDrives(true)`** — 「声明支持」开关
- 自 Google Drive API v3 2020 年升级后，所有需要访问共享盘的调用都必须显式声明
- 代码中在 `Files.*`（除 List 外有时省略，但核心操作都有）、`Changes.*` 上都设置
- 注意：`Drives.*` API 本身就是共享盘专属，不需要这个参数

**`DriveId` + `Corpora("drive")`** — 查询范围限定
- 只在 `Files.List` 和 `Changes.*` 上使用（查询类 API）
- `DriveId` 告诉 API 「只查这个共享盘里的内容」
- `Corpora` 设定查询语料库范围：`"user"`（个人盘，默认）/ `"drive"`（共享盘）/ `"allDrives"`（全部）
- 代码中共享盘模式下两者**成对出现**，确保查询范围一致

**`IncludeItemsFromAllDrives(true)`** — 结果集开关
- 告诉 API「返回结果中可以包含共享盘项目」
- 与 `DriveId` 配合：`DriveId` 限定范围，`IncludeItems` 允许结果中出现共享盘项目
- 个人盘查询共享给我的共享盘内容时也需要设这个

**`RestrictToMyDrive`** — 变更检测独有
- 仅 `Changes.List` 有此参数
- 设为 `true` 时只返回"我的云端硬盘"中的变更（排除共享内容）
- 代码中：`changesCall.RestrictToMyDrive(!f.opt.SharedWithMe)`
  - SharedWithMe 模式 → false → 不限制，包含共享文件的变更
  - 非 SharedWithMe 模式 → true → 只看自己的盘

**`Spaces`** — 存储空间选择
- `"drive"`（默认，普通文件）/ `"appDataFolder"`（应用数据文件夹）/ `"photos"`（谷歌相册）
- 代码中仅在 `rootFolderID == "appDataFolder"` 时设置为 `"appDataFolder"`
- 在 Files.List 和 Changes.List 中都有相应设置

### 8.3 参数设置的代码位置对照

| 参数 | Files.List 位置 | Changes.List 位置 | 备注 |
|------|----------------|------------------|------|
| `SupportsAllDrives` | [drive.go#L1101](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1101) | [drive.go#L3249](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3249) | 查询类 API 设置 |
| `IncludeItemsFromAllDrives` | [drive.go#L1102](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1102) | [drive.go#L3250](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3250) | 查询类 API 设置 |
| `DriveId` | [drive.go#L1104](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1104) | [drive.go#L3252](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3252) | 共享盘时设置 |
| `Corpora("drive")` | [drive.go#L1105](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1105) | ❌ 无 | 仅 Files.List 需要 |
| `Spaces("appDataFolder")` | [drive.go#L1109](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1109) | [drive.go#L3256](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3256) | appDataFolder 模式 |
| `RestrictToMyDrive` | ❌ 无 | [drive.go#L3258](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3258) | 仅变更检测 |

---

## 九、缓存未命中场景边界分析

`dirCache` 是文件查询与变更检测的桥梁，但它本质上是一个**惰性填充的缓存**——只有被访问过的路径才会进入缓存。当缓存未命中时，三大模块的行为边界差异显著。

### 9.1 dirCache 的填充时机与生命周期

**填充路径（正向 + 反向同时写入）**：
```
itemToDirEntry()
  → 遇到 folder 类型 → dirCache.Put(remote, item.Id)
  → 同时写入 cache[path]=id 和 invCache[id]=path
```

只有 `ListP`、`ListR` 等列表操作会触发填充。单次文件查询（NewObject）**不会**向 dirCache 写入新目录条目。

**清空路径**：
- `DirCacheFlush()` → `ResetRoot()` → 全部清空，回到 trueRoot 状态
- `FlushDir(dir)` → 清空某目录及其所有子项（用于移动/删除后）
- VFS 挂载时 `ForgetAll()` 也会级联触发

### 9.2 缓存未命中的五种典型场景

#### 场景 1：首次启动，冷启动状态

```
初始状态: dirCache 只有 { "": rootFolderID }
  ↓
用户访问 a/b/c.txt
  ↓
dirCache.FindDir("a/b")  // 未命中
  → _findDir("a/b")
    → 先递归找 "a" → 未命中 → FindLeaf(root, "a") → 调用 list()
    → 找到后 Put("a", aId)
    → 再找 "b" → 未命中 → FindLeaf(aId, "b") → 调用 list()
    → 找到后 Put("a/b", bId)
  ↓
返回 bId，继续找文件 c.txt
```

**代价**：每深一级目录多一次 API 调用。冷启动阶段的列表操作是缓存填充的黄金时期。

#### 场景 2：变更检测遇到新文件（新路径不在缓存）

外部客户端在 `docs/` 下新建了 `report.docx`，而 `docs/` 已经在缓存中：

```
change = { fileId: newId, file: { name: "report.docx", parents: [docsId] } }
  ↓
① 旧路径: dirCache.GetInv(newId) → ❌ 未命中（新文件无旧路径）
  → 不加入 pathsToClear
  ↓
② 新路径: for parent in parents {
     dirCache.GetInv(docsId) → ✅ 命中 → "docs"
     → 拼接: "docs/report.docx" → 加入 pathsToClear
   }
  ↓
notifyFunc("docs/report.docx", EntryObject)
```

**效果**：VFS 层收到通知后会失效对应路径缓存。下次用户 ls docs 时触发 ListP 重新拉取，就能看到新文件。

#### 场景 3：变更检测遇到移动到未缓存目录

文件从 `a/` 移动到 `b/c/`，而 `b/c/` 从未被访问过：

```
change = { fileId: fileId, file: { name: "x.txt", parents: [bcId] } }
  ↓
① 旧路径: dirCache.GetInv(fileId) → ✅ 命中 → "a/x.txt"
  → 加入 pathsToClear
  ↓
② 新路径: for parent in parents {
     dirCache.GetInv(bcId) → ❌ 未命中
     → 跳过，不加入 pathsToClear
   }
  ↓
只通知了 "a/x.txt"（旧位置失效）
新位置 "b/c/x.txt" 完全不知道
```

**后果**：
- 旧位置的 VFS 缓存会失效
- 新位置不会主动通知，用户如果不主动进入 `b/c/` 目录，就看不到文件
- 只有当用户主动 `ls b/c/` 触发 ListP 后，新位置才会被发现

#### 场景 4：文件被移动到更深的未缓存路径树

与场景 3 类似，但影响范围更大——整个子树都在盲区。

```
文件夹 docs/ 被移动到 archive/2023/
  ↓
① dirCache 中所有 docs/ 下的子项 → GetInv 能查到旧路径
  → 所有旧路径都能被通知失效
  ↓
② 新父目录 archive/2023/ → 不在缓存中
  → 新位置完全不可见
  ↓
结果: 用户感知到 "docs/ 消失了"，但不知道它去了哪里
```

#### 场景 5：深层目录的孙子文件变更

```
dirCache 中有: a/ → aId
            a/b/ → 不在缓存（从未 list 过）
  ↓
a/b/c.txt 被外部修改
  ↓
① 旧路径: GetInv(fileId) → ❌ 未命中（从未访问过）
  ↓
② 新路径: parents = [abId] → GetInv(abId) → ❌ 未命中
  ↓
结果: 完全没有通知 ★
用户完全不知道 a/b/c.txt 变了
```

**这是变更通知最大的盲区**：对于从未 list 过的子目录内的变化，变更检测无能为力。这也是 rclone 的 VFS 还需要配合 `PollInterval` 和 `refresh` 机制的原因。

### 9.3 缓存未命中的设计哲学

rclone 的变更通知设计遵循**「尽力而为」**原则：
- 能在缓存中找到路径 → 精确通知
- 找不到 → 静默跳过，不做额外 API 调用
- 不尝试「递归反查父目录路径」（代价太高）

这种设计的权衡：
- ✅ 不增加额外 API 调用，不影响性能
- ✅ 不阻塞变更处理流水线
- ❌ 存在通知盲区（未访问过的目录树）
- ❌ 移动类变更可能只通知一半（旧位置）

---

## 十、删除事件在变更通知中的处理边界

删除事件是变更通知中最容易被忽视、也是行为最特殊的一类。

### 10.1 删除事件的两种形态

Google Drive API 的变更事件中，删除有两种表现形式：

| 删除方式 | change.File | change.Removed | 说明 |
|---------|------------|---------------|------|
| **移入回收站**（Trashed） | 非 nil，`file.trashed=true` | false | 文件还在，标记为已删除 |
| **永久删除**（Permanently Deleted） | nil | true | 文件彻底消失，没有 file 信息 |

> 注意：rclone 当前代码**没有读取 `change.Removed` 字段**，完全依赖 `change.File != nil` 来区分。

### 10.2 代码中的处理逻辑

[drive.go#L3271-L3302](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3271-L3302)

```go
for _, change := range changeList.Changes {
    // ① 找旧路径（只看 fileId，不依赖 change.File）
    if path, ok := f.dirCache.GetInv(change.FileId); ok {
        // 根据 mimeType 判断类型；如果 change.File 为 nil，默认按目录处理
        if change.File != nil && change.File.MimeType != driveFolderType {
            pathsToClear = append(pathsToClear, entryType{path: path, entryType: fs.EntryObject})
        } else {
            pathsToClear = append(pathsToClear, entryType{path: path, entryType: fs.EntryDirectory})
        }
    }

    // ② 找新路径（只有 change.File != nil 时才尝试）
    if change.File != nil {
        // ... 计算新路径 ...
    }
}
```

**关键点解读**：

1. **旧路径查找不依赖 `change.File`**
   - 只需要 `change.FileId`
   - 无论文件是被移到回收站还是永久删除，只要 fileId 在 dirCache 中，就能找到旧路径并发出通知

2. **永久删除时 `change.File == nil`**
   - 新路径计算块完全跳过（`if change.File != nil` 不进入）
   - 只发旧路径通知（失效）
   - 符合预期：删除了就是没了，不需要新路径

3. **类型判定的 fallback**
   - 如果 `change.File == nil`（永久删除），`mimeType` 判断走 else 分支 → **默认按目录处理**
   - 这个细节对 VFS 影响不大，因为无论是 EntryObject 还是 EntryDirectory，最终都是清除缓存

### 10.3 删除事件的级联盲区

**最大的坑：删除目录不会产生子文件的变更事件**

Google Drive API 的行为：删除一个目录时，**只产生一条该目录的变更记录**，目录内的所有子文件、子目录**不会**各自产生变更事件。

```
删除 docs/ 目录（内含 report.docx, data.csv, subdir/ 等）
  ↓
Google Drive 只产生 1 条 change（docs/ 本身）
  ↓
rclone 处理:
  GetInv(docsId) → 命中 → "docs" 通知
  子文件 report.docx → 没有 change → 不会被通知
  子目录 subdir/ → 没有 change → 不会被通知
  ↓
dirCache 中 "docs/report.docx"、"docs/subdir/" 等条目 → 仍留在缓存中 ★
```

**后果**：
- VFS 中 `docs/` 目录本身会被标记为失效
- 但 `docs/report.docx` 等子项的 dirCache 条目**仍然存在**
- VFS 可能在一段时间内还能查到这些子项的元数据（读缓存）
- 直到用户主动再次 list `docs/` → 触发 404 → 才会知道整个目录都没了

### 10.4 回收站 vs 永久删除的行为差异

| 行为 | 移入回收站（trashed=true） | 永久删除（permanent delete） |
|------|--------------------------|----------------------------|
| change.File | ✅ 存在，`trashed=true` | ❌ nil |
| 旧路径通知 | ✅ 有（前提是在缓存） | ✅ 有（前提是在缓存） |
| 新路径通知 | ❌ 无（parents 不变） | ❌ 无（没 file） |
| 子文件变更事件 | ❌ 无（只有目录自身） | ❌ 无（只有目录自身） |
| TrashedOnly 模式下可见 | ✅ 是 | ❌ 否 |
| 可恢复 | ✅ untrash | ❌ 不可恢复 |

### 10.5 与查询模块的联动：TrashedOnly 模式

当 `--drive-trashed-only` 启用时，查询和变更检测的行为都发生变化：

**查询侧（list）**：
- `trashedOnly=true` 传递给 `list()`
- 查询条件从 `trashed=false` 变成 `trashed=true`
- 同时对文件夹特殊处理：`(mimeType='folder' or trashed=true)` 确保目录结构可见

**变更检测侧**：
- 没有特殊处理
- 移入回收站的文件 → 旧路径通知（因为在缓存中）→ VFS 失效
- 用户下次 list → 会在 TrashedOnly 模式下看到这些文件

---

## 十一、根目录对象特殊场景解析

「根目录对象」指的是 `parents` 数组为空的文件或目录——它们直接位于某个存储空间的最顶层。

### 11.1 什么情况下会出现根对象？

**场景 1：Shared With Me 模式的根级别**

[drive.go#L1021-L1030](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1021-L1030)

当 `--drive-shared-with-me` 开启时，"共享给我"的文件和文件夹在 Google Drive 中**没有 `parents` 属性**（它们不在你的目录树里）。

```
SharedWithMe 模式下列表:
  list() 中遇到 dirID == rootFolderID 时
  → 用 "sharedWithMe=true" 替代 "'root' in parents" 查询
  → 返回的文件 parents 数组为空
```

**场景 2：StarredOnly 模式**

类似地，加星标的文件可能来自任何位置，根列表时也用 `starred=true` 查询。

**场景 3：应用数据文件夹（appDataFolder）**

应用数据文件夹是一个特殊的隔离空间，其根就是它自己。

**场景 4：共享盘根目录下的对象（实际上有 parents）**

共享盘的根本身是一个 drive，其下的文件 parents 指向根目录 ID，**通常不为空**。所以严格来说不算是「根对象」。

### 11.2 列表查询中的特殊处理

在 `list()` 函数中，根目录查询有特殊分支：

[drive.go#L1021-L1033](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L1021-L1033)

```go
if (f.opt.SharedWithMe || f.opt.StarredOnly) && dirID == f.rootFolderID {
    if f.opt.SharedWithMe {
        _, _ = parentsQuery.WriteString("sharedWithMe=true")
    }
    if f.opt.StarredOnly {
        if f.opt.SharedWithMe {
            _, _ = parentsQuery.WriteString(" and ")
        }
        _, _ = parentsQuery.WriteString("starred=true")
    }
} else {
    _, _ = fmt.Fprintf(parentsQuery, "'%s' in parents", dirID)
}
```

**核心逻辑**：
- 如果是根目录 + SharedWithMe/StarredOnly → 用特殊标志查询，不用 parents 过滤
- 其他情况 → 标准的 `'dirID' in parents` 查询
- 注意：只有**根目录这一级**有特殊处理。深入共享文件夹内部时，仍然用正常的 parents 查询

### 11.3 变更检测中的根对象处理

[drive.go#L3289-L3301](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L3289-L3301)

```go
if len(change.File.Parents) > 0 {
    for _, parent := range change.File.Parents {
        if parentPath, ok := f.dirCache.GetInv(parent); ok {
            newPath := path.Join(parentPath, change.File.Name)
            pathsToClear = append(pathsToClear, entryType{path: newPath, entryType: changeType})
        }
    }
} else { // a true root object that is changed
    pathsToClear = append(pathsToClear, entryType{path: change.File.Name, entryType: changeType})
}
```

**这段代码的意图**：
- `parents > 0` → 有父目录 → 通过父目录路径 + 文件名 计算完整路径
- `parents == 0` → 根对象 → **直接用文件名作为路径**（相对根目录）

**潜在问题**：
1. **SharedWithMe 根目录的新文件** → 直接用文件名作为路径 → 正确（因为就在根视图下）
2. **但如果文件本来就不在 rclone 的视图范围内呢？** → 会产生一条路径通知，但 VFS 可能根本没有这个路径
3. **多个根对象同名的情况** → 通知可能只对应其中一个，语义模糊

### 11.4 ListR 中的根目录特殊处理

在 `listRRunner` 中，对根目录也有特殊判断：

[drive.go#L2102-L2144](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/backend/drive/drive.go#L2102-L2144)

```go
if f.opt.SharedWithMe && len(item.Parents) == 0 && len(paths) == 1 && paths[0] == "" {
    item.Parents = dirs  // 人为补上 parents
}
// ...
if len(paths) == 1 {
    i = 0                  // 根目录时直接用第一个（也是唯一的）路径
    earlyExit = true       // 只插入一次
}
```

**这段代码解释**：
- SharedWithMe 模式下，根目录返回的文件 `parents` 为空
- 为了后续能正常匹配路径，人为把 `dirs`（也就是 root ID）塞进去
- 根目录只有一个路径，所以不需要二分查找，直接用索引 0
- `earlyExit = true` 表示只插入一次（即使文件有多个 parents）

### 11.5 根目录 ID 别名机制

dirCache 还有一个 `SetRootIDAlias` 机制，用于处理根 ID 的不同表示形式：

[dircache.go#L129-L141](file:///d:/fz/0601-2/solo-dogfeeding/code/45-rclone/lib/dircache/dircache.go#L129-L141)

```go
func (dc *DirCache) SetRootIDAlias(rootID string) {
    dc.rootID = rootID
    dc.Put("", dc.rootID)
}
```

这个函数用于当发现"root"这个ID只是别名、实际有另一个真实ID时，可以更新缓存而不清空。在 Google Drive 中：
- `"root"` 是一个魔法别名，实际对应你的根目录
- `getRootID()` 会把 `"root"` 解析成真实 ID（一串字母数字）
- 但 rclone 初始化时直接用配置值作为 rootFolderID，可能是 `"root"` 也可能是真实 ID

---

## 十二、边界场景串联全景：当共享盘 + 变更通知 + 缓存未命中交织

最后，我们用一个综合场景把所有边界情况串联起来。

### 场景：共享盘下深层目录的外部删除事件

```
初始状态:
  - 配置了共享盘 TeamDriveX
  - 用户 VFS 挂载后访问过 docs/ 目录，但没进入 docs/2024/
  - dirCache 中有: "" → rootId, "docs" → docsId
  - dirCache 中没有: "docs/2024" 及其子项

外部操作:
  某人在 Google Drive 网页端永久删除了 docs/2024/ 整个目录
  （内含 report.docx、data/ 等一堆文件）
```

#### 第 1 步：变更检测收到事件

```
Changes.List 返回:
  change[0]: { fileId: docs2024Id, file: nil, removed: true }  // 只有这一条！
  (子文件和子目录都没有独立 change)
```

#### 第 2 步：changeNotifyRunner 处理

```
对 change (docs2024Id):

  ① 旧路径:
     dirCache.GetInv(docs2024Id) → ❌ 未命中！
     （因为用户从未进入过 docs/2024/，不在缓存里）
     → 不加入 pathsToClear
     → 旧路径通知: 无

  ② 新路径:
     change.File == nil → 跳过
     → 新路径通知: 无

结果: 零通知 ★
用户什么都感知不到
```

#### 第 3 步：用户后续操作

```
用户输入: ls docs/
  → ListP("docs")
  → list([docsId])
  → 返回 2023/ 目录、2025/ 目录（没有 2024/ 了）
  → itemToDirEntry 更新 dirCache
    → "docs/2023" 已存在，刷新
    → "docs/2025" 已存在，刷新
    → "docs/2024" 消失了（不再返回，所以 Put 不会发生）
  → 但 dirCache 中 "docs/2024" 这条记录还在吗？
    → 不在！因为 dirCache 是惰性缓存，列表操作只会 Put 当前存在的项
    → 已消失的项不会自动从 cache 中删除 ★
    → 但如果 VFS 做了 diff，会发现少了 2024/ 并删除对应条目
```

#### 第 4 步：如果用户之前缓存了 docs/2024/report.docx

```
如果用户之前进入过 docs/2024/
→ dirCache 中有 "docs/2024" → docs2024Id
→ 也有 "docs/2024/report.docx" → reportId（不对，dirCache 只存目录）

等等，dirCache 只存目录，不存文件！
  → dirCache.invCache 中只有目录ID → 路径 的映射
  → 文件ID 不在 dirCache 里 ★

回到删除 docs/2024 的 change:
  GetInv(docs2024Id) → 如果命中 → 通知 "docs/2024" 目录失效
  → 子文件呢？不在 dirCache 里，没法通过 GetInv 通知
  → 但 VFS 层收到目录失效通知后，会级联失效该目录下的所有文件缓存
```

### 关键洞察总结

| 边界因素 | 影响程度 | 说明 |
|---------|---------|------|
| **dirCache 只存目录** | ⭐⭐⭐⭐⭐ | 变更通知中，文件级别的 GetInv 永远不会命中（因为文件不在 dirCache 里）。文件只能通过「父目录路径 + 文件名」的方式定位。 |
| **删除目录不产生子变更** | ⭐⭐⭐⭐ | Google API 只发一条目录变更，子项都没有。这是 API 限制，rclone 无法突破。 |
| **缓存未命中 = 通知盲区** | ⭐⭐⭐⭐ | 没 list 过的目录，其下的所有变更都无法被翻译为路径通知。 |
| **共享盘参数需处处一致** | ⭐⭐⭐⭐ | Files.List 和 Changes.List 都需要设置共享盘参数，遗漏任一都会导致数据不一致。 |
| **根对象的 parents 为空** | ⭐⭐⭐ | SharedWithMe/Starred 模式下根级文件 parents 为空，需要特殊处理路径拼接。 |
| **永久删除时 file 为 nil** | ⭐⭐ | 只有 fileId 可用，依赖旧缓存路径。如果不在缓存中就彻底找不到。 |

### 最终串联关系图（边界版）

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Google Drive API (边界行为)                      │
│  - 删除目录只产生1条change                                          │
│  - 永久删除时 change.File == nil                                    │
│  - 共享盘需要 SupportsAllDrives+DriveId                             │
│  - SharedWithMe根级文件parents为空                                  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    list() / Changes.List()                          │
│  ★ 每次调用都必须正确设置共享盘参数                                  │
│  ★ list() 负责填充 dirCache                                         │
│  ★ Changes.List 只返回 fileId + 有限字段                            │
└─────────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────────────┐
│   填充 dirCache         │     │   changeNotifyRunner            │
│   (只有目录，只有访问过的)│     │   - GetInv(fileId) → 旧路径     │
│                         │     │   - GetInv(parent) → 拼新路径   │
│   Put(path, id)         │     │   - 命中 → 通知                 │
│   双向写入 cache+inv    │     │   - 未命中 → 静默跳过 ★         │
└─────────────────────────┘     └─────────────────────────────────┘
              │                               │
              └───────────────┬───────────────┘
                              ▼
                    ┌──────────────────┐
                    │   dirCache (仅目录)│
                    │   路径 ↔ ID 双向  │
                    │   惰性填充        │
                    │   不级联失效      │
                    └──────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │   VFS 层          │
                    │   - 接收 notify   │
                    │   - 失效对应路径  │
                    │   - 下次访问重拉  │
                    └──────────────────┘
```

**一句话总结**：变更通知的可靠性上限 = dirCache 的覆盖范围。dirCache 覆盖到哪里，变更通知就能精确到哪里；没覆盖到的地方，就是盲区。
