# 本地文件系统后端代码梳理

本文档梳理 rclone 本地文件系统后端（local backend）的核心实现，重点关注路径归一化、权限处理和跨平台差异的实现边界。

## 目录

1. [整体架构](#整体架构)
2. [路径归一化](#路径归一化)
3. [权限处理](#权限处理)
4. [跨平台差异实现边界](#跨平台差异实现边界)
5. [关键数据结构](#关键数据结构)

---

## 整体架构

本地文件系统后端位于 [backend/local/](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/) 目录，核心文件包括：

| 文件 | 职责 |
|------|------|
| [local.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go) | 主实现，Fs/Object/Directory 结构及核心方法 |
| [metadata.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/metadata.go) | 元数据读写的通用逻辑 |
| [xattr.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/xattr.go) | 扩展属性（xattr）读写 |
| [symlink.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/symlink.go) | 符号链接处理 |
| [lchmod.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/lchmod.go) | 符号链接权限修改 |

辅助库：

- [lib/file/](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/lib/file/) - 文件操作工具（UNC路径、预分配、稀疏文件等）
- [lib/encoder/](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/lib/encoder/) - 文件名编码转换
- [fs/fspath/](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/fs/fspath/) - 路径解析工具

---

## 路径归一化

### 1. 路径模型：双重路径表示

本地后端维护两套路径体系：

- **remote 路径**：rclone 内部统一表示，使用 `/` 分隔，经过 encoder 编码为标准形式
- **local 路径**：操作系统原生路径，使用本地分隔符，符合 OS 命名规范

转换关系：

```
remote (标准编码, /分隔)
    ↓ encoder.FromStandardPath + filepath.FromSlash
local (OS原生编码, 原生分隔符)
```

### 2. 根路径归一化

核心函数：[cleanRootPath](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go#L1695-L1738)

处理流程：

```
输入 root 字符串
    ↓
Windows: 提取卷名（vol）
    ↓
非 Standard 编码: 逐段编码（保留 . 和 .. 不编码）
    ↓
Windows: 恢复卷名
    ↓
filepath.Abs: 转换为绝对路径
    ↓
!noUNC: file.UNCPath 转换为 UNC 长路径（仅Windows有效）
    ↓
返回归一化后的 root
```

关键设计点：

- **`.` 和 `..` 保护**：路径段为 `.` 或 `..` 时不进行 encoder 编码，避免语义改变
- **卷名分离**：Windows 上先分离卷名再处理路径主体，最后恢复，防止 UNC 前缀干扰
- **UNC 长路径**：Windows 上转换为 `\\?\` 前缀格式以突破 260 字符路径限制

### 3. UNC 路径转换

实现：[lib/file/unc_windows.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/lib/file/unc_windows.go)

转换规则：

| 输入 | 输出 |
|------|------|
| `C:\path\to\file` | `\\?\C:\path\to\file` |
| `\\server\share\path` | `\\?\UNC\server\share\path` |
| `\\?\C:\already\unc` | 原样返回 |
| 非 Windows 平台 | 原样返回 |

非 Windows 平台：[unc.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/lib/file/unc.go) 中 `UNCPath` 为恒等函数。

### 4. remote ↔ local 路径转换

**remote → local**：[localPath](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go#L770-L772)

```go
func (f *Fs) localPath(name string) string {
    return filepath.Join(f.root, filepath.FromSlash(f.opt.Enc.FromStandardPath(name)))
}
```

**local → remote**：[cleanRemote](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go#L753-L768)

```
输入: dir (父目录remote), filename (本地文件名)
    ↓
UTFNorm=true: Unicode NFC 规范化
    ↓
path.Join 拼接 + encoder.ToStandardName 编码
    ↓
无效UTF-8: 记录警告（去重）
    ↓
返回 remote 路径
```

### 5. 文件名编码（Encoder）

实现：[lib/encoder/encoder.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/lib/encoder/encoder.go)

本地后端默认使用 `encoder.OS`，针对不同 OS 启用不同编码规则：

- **Windows**：编码 `:?"*<>|` 等非法字符及前后导空格/点
- **macOS**：编码特殊字符以兼容 HFS+ 的 NFD 规范化
- **其他**：按需编码

编码策略：将受限字符映射到 Unicode 全角（FULLWIDTH）变体，用 `‛`（SINGLE HIGH-REVERSED-9 QUOTATION MARK）作为转义符。

### 6. 路径解析（fspath）

实现：[fs/fspath/path.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/fs/fspath/path.go)

`Parse` 函数判断路径是本地路径还是远程路径：

- 不含 `:` → 本地路径
- 含 `:` 但为盘符（如 `C:`）→ 本地路径
- 含 `:` 且为远程名 → 远程路径

所有路径最终统一转换为 `/` 分隔。

---

## 权限处理

### 1. 文件创建权限

本地后端创建文件/目录时使用的权限位：

| 操作 | 权限 | 位置 |
|------|------|------|
| 创建普通文件 | 0666 | [local.go#L1466](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go#L1466) |
| 创建目录 | 0777 | [local.go#L793](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go#L793) |

> 上述权限会被操作系统的 umask 修正，实际权限 = mode & ~umask。

### 2. 权限修改（Chmod）

#### 普通文件权限修改

使用标准 `os.Chmod`，所有平台均支持。

#### 符号链接权限修改（lChmod）

核心函数：`lChmod` - 修改符号链接本身的权限而非目标。

| 平台 | 支持情况 | 实现文件 |
|------|----------|----------|
| Linux | ❌ 不支持 | （同 lchmod.go） |
| Windows | ❌ 不支持 | [lchmod.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/lchmod.go) |
| macOS/BSD 等 Unix | ✅ 支持 | [lchmod_unix.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/lchmod_unix.go) |
| plan9/js | ❌ 不支持 | （同 lchmod.go） |

**Linux 不支持原因**：Linux 的 `fchmodat` 系统调用不接受 `AT_SYMLINK_NOFOLLOW` 标志，会返回 `ENOTSUP`。

实现方式（Unix）：使用 `unix.Fchmodat` + `unix.AT_SYMLINK_NOFOLLOW` 标志。

### 3. 所有权处理（Chown）

实现位置：[metadata.go#L118-L137](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/metadata.go#L118-L137)

| 平台 | 支持情况 | 说明 |
|------|----------|------|
| Windows | ❌ 忽略 | 仅输出 debug 日志 |
| plan9 | ❌ 忽略 | 仅输出 debug 日志 |
| Unix 系列 | ✅ 支持 | `os.Chown` / `os.Lchown` |

符号链接使用 `os.Lchown`，普通文件使用 `os.Chown`。

### 4. 权限错误处理

#### 目录列表权限

[List 方法](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go#L622-L751) 中：

- 目录打开权限错误（Permission denied）：
  - 记录错误日志
  - 上报为 NoRetryError（失败同步但不重试）
  - 返回 nil error（继续执行但标记失败）

#### 单独文件 stat 失败

- 非 Windows 平台使用 `Readdirnames` 逐个 `Lstat`
- 单个文件 stat 失败不终止整个目录遍历
- 被过滤规则排除的文件不报告错误

### 5. Windows 特殊权限处理

#### 隐藏文件更新

[Update 方法](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go#L1466-L1479)：

- 以 `O_CREATE|O_TRUNC` 打开失败且为 Permission denied 时
- 尝试以 `O_WRONLY|O_TRUNC`（不带 CREATE）重新打开
- 解决 Windows 上更新隐藏/系统文件的权限问题

#### 目录删除权限

[Rmdir 方法](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go#L870-L874)：

- Windows 上删除目录遇 `ErrPermission` 时
- 先 `Chmod` 为 `0o600` 再尝试删除
- 对应 Go issue #26295 的 workaround

---

## 跨平台差异实现边界

### 1. 文件系统特性

#### 大小写敏感性

[caseInsensitive](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go#L523-L537)

默认判定：

| 平台 | 默认大小写敏感 |
|------|---------------|
| Windows | ❌ 不敏感 |
| macOS | ❌ 不敏感 |
| 其他 | ✅ 敏感 |

可通过 `--local-case-sensitive` / `--local-case-insensitive` 强制覆盖。

> 注意：此判定不完全准确（如 macOS 可格式化为大小写敏感的 APFS），但作为默认值是合理的。

#### 单文件系统边界（one_file_system）

[readDevice](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/read_device_unix.go)

- 仅 Unix 系列支持读取设备号（`stat.Dev`）
- 启用 `--one-file-system` 时，跨设备的目录被跳过
- Windows/plan9/js 等平台 `readDevice` 始终返回 `devUnset`

### 2. 符号链接差异

| 特性 | Unix | Windows |
|------|------|---------|
| 符号链接类型 | `os.ModeSymlink` | `os.ModeSymlink \| os.ModeIrregular` |
| Junction Points | N/A | 视为符号链接处理 |
| 循环检测 | `ELOOP` 错误 | 无专门检测 |
| lchmod | 部分支持 | 不支持 |
| lchown | 支持 | 不支持 |

Windows 特殊处理：

```go
symlinkFlag := os.ModeSymlink
if runtime.GOOS == "windows" {
    symlinkFlag |= os.ModeIrregular
}
```

参见：[local.go#L699-L702](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go#L699-L702)

循环符号链接检测（Unix  only）：[symlink.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/symlink.go) - 通过 `syscall.ELOOP` 判断。

### 3. 时间类型支持

[time_type 选项](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go#L295-L317)

| 时间类型 | Windows | macOS | Linux | BSD | plan9/js |
|---------|---------|-------|-------|-----|----------|
| mtime | ✅ | ✅ | ✅ | ✅ | ✅ |
| atime | ✅ | ✅ | ✅ | ✅ | ❌ |
| btime (创建/出生) | ✅ (CreationTime) | ✅ | ✅ (statx, 4.11+) | ✅ | ❌ |
| ctime (状态变更) | ❌ | ✅ | ✅ | ✅ | ❌ |

**注意**：`btime` 在 Linux 上需要内核 4.11+ 支持的 `statx()` 系统调用，旧内核回退到 `fstatat()` 且不返回 btime。

实现文件：
- Windows: [metadata_windows.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/metadata_windows.go)
- Linux: [metadata_linux.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/metadata_linux.go)
- Unix (通用): [metadata_unix.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/metadata_unix.go)
- BSD: [metadata_bsd.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/metadata_bsd.go)

### 4. 元数据（Metadata）

#### 系统元数据

| 字段 | Windows | Linux | macOS | BSD |
|------|---------|-------|-------|-----|
| mode | ✅ (简化) | ✅ | ✅ | ✅ |
| uid | ❌ | ✅ | ✅ | ✅ |
| gid | ❌ | ✅ | ✅ | ✅ |
| rdev | ❌ | ✅ | ✅ | ✅ |
| atime | ✅ | ✅ | ✅ | ✅ |
| mtime | ✅ | ✅ | ✅ | ✅ |
| btime | ✅ | ✅ (statx) | ✅ | ✅ |

#### 扩展属性（xattr）

实现：[xattr.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/xattr.go)

| 平台 | 支持用户 xattr | 说明 |
|------|---------------|------|
| Linux | ✅ | ext4/xfs/btrfs 等 |
| macOS | ✅ | HFS+/APFS |
| FreeBSD | ✅ | |
| NetBSD | ✅ | |
| Solaris | ✅ | |
| Windows | ❌ | pkg/xattr#47 未解决 |
| OpenBSD | ❌ | 编译排除 |
| Plan9 | ❌ | 编译排除 |

xattr 前缀：`user.`（Unix 惯例）

**动态降级**：运行时遇 `ENOTSUP`/`ENOATTR`/`EINVAL` 错误时，自动标记 xattr 不支持并静默降级。

### 5. 文件删除行为

- **Unix**：直接 `os.Remove`，一次尝试
- **Windows**：[remove_windows.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/remove_windows.go)
  - 遇 `ERROR_SHARING_VIOLATION` 时指数退避重试
  - 最多重试 10 次
  - 初始等待 1ms，每次翻倍

### 6. 文件打开行为

实现：[lib/file/file_windows.go](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/lib/file/file_windows.go)

**Windows 特殊点**：
- 共享模式：`FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE`
- 允许打开的文件被重命名或删除（Unix 默认行为，Windows 需显式设置）
- 使用 `FILE_FLAG_BACKUP_SEMANTICS` 以支持目录句柄操作

**Unix**：直接使用 `os.OpenFile`，行为与标准库一致。

### 7. 文件系统高级特性

| 特性 | Windows | Linux | macOS | 说明 |
|------|---------|-------|-------|------|
| 预分配 (Preallocate) | ✅ | ✅ | ✅ | 防止磁盘碎片 |
| 稀疏文件 (Sparse) | ✅ | - | - | 多线程下载优化 |
| Reflink 克隆 | ❌ | ❌ | ✅ (APFS) | 服务器端拷贝加速 |
| 磁盘空间查询 | ✅ | ✅ | ✅ | about 命令 |

### 8. 目录读取策略

[List 方法](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/backend/local/local.go#L649-L690)

**Windows/Plan9**：使用 `Readdir()` 批量读取 FileInfo
- 优点：性能好，一次系统调用获得全部信息
- 缺点：单个条目错误可能影响整个目录读取

**其他 OS**：使用 `Readdirnames()` 读取名称，再逐个 `Lstat`
- 优点：单个文件错误不阻断整个目录列表
- 缺点：系统调用次数多，性能略低

### 9. 保留名称检查

[IsReserved](file:///d:/fz/0601-2/solo-dogfeeding/code/48-rclone/lib/file/file_windows.go#L72-L101)（仅 Windows）

检查以下非法命名：

- 末尾空格或句点
- DOS 设备名：`CON`, `PRN`, `AUX`, `NUL`, `COM1-9`, `LPT1-9`

非 Windows 平台 `IsReserved` 始终返回 nil。

---

## 关键数据结构

### Fs 结构

```go
type Fs struct {
    name        string              // 远程名称
    root        string              // 根目录 (OS 路径)
    opt         Options             // 配置选项
    features    *fs.Features        // 功能特性
    dev         uint64              // 根节点设备号
    lstat       func(string) (os.FileInfo, error)  // os.Lstat 或 os.Stat
    // ...
}
```

### Options 配置项

| 选项 | 作用 | 平台相关 |
|------|------|----------|
| NoUNC | 禁用 UNC 长路径转换 | Windows only |
| FollowSymlinks | 跟随符号链接 | 全平台 |
| TranslateSymlinks | 转换符号链接为 .rclonelink 文件 | 全平台 |
| OneFileSystem | 不跨文件系统 | Unix only |
| CaseSensitive/CaseInsensitive | 强制大小写敏感性 | 全平台（默认值与 OS 相关） |
| NoPreAllocate | 禁用预分配 | 全平台 |
| NoSparse | 禁用稀疏文件 | Windows |
| NoSetModTime | 禁用修改时间设置 | 全平台 |
| UTFNorm | Unicode NFC 规范化 | 全平台（主要 macOS） |
| TimeType | 返回的时间类型 | 支持度因 OS 而异 |
| Enc | 文件名编码器 | 全平台（默认值因 OS 而异） |

### Object 结构

```go
type Object struct {
    fs             *Fs
    remote         string   // remote 路径 (编码后)
    path           string   // 本地路径 (OS 路径)
    size           int64    // 文件大小
    mode           os.FileMode
    modTime        time.Time
    hashes         map[hash.Type]string
    translatedLink bool     // 是否为转换后的符号链接
}
```

---

## 总结

本地文件系统后端的跨平台策略遵循以下设计原则：

1. **分层抽象**：核心逻辑在 `local.go`，平台差异通过 build tag 分离到 `_windows.go`/`_unix.go`/`_other.go` 等文件
2. **优雅降级**：高级特性（xattr、btime、reflink 等）不可用时静默降级，不报错
3. **路径双轨制**：remote 路径（标准化）与 local 路径（OS 原生）分离，通过 encoder 转换
4. **Windows 特殊照顾**：UNC 路径、共享模式、保留名称、隐藏文件、删除重试等
5. **动态检测**：部分特性（如 Linux statx）运行时探测，不可用则回退
