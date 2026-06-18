# 本地文件系统后端代码梳理

本文档梳理 rclone 本地文件系统后端（local backend）的核心实现，重点关注**路径归一化**、**权限处理**和**跨平台差异**的实现边界。

所有代码引用均使用仓库相对路径。

---

## 目录

1. [整体架构](#整体架构)
2. [路径归一化](#路径归一化)
3. [权限处理](#权限处理)
4. [跨平台差异实现边界](#跨平台差异实现边界)
5. [关键数据结构](#关键数据结构)
6. [设计原则总结](#设计原则总结)

---

## 整体架构

本地文件系统后端位于 `backend/local/` 目录，核心文件及职责：

| 文件 | 职责 |
|------|------|
| `backend/local/local.go` | 主实现，Fs/Object/Directory 结构及核心方法 |
| `backend/local/metadata.go` | 元数据读写的通用逻辑（与平台无关部分） |
| `backend/local/xattr.go` | 扩展属性（xattr）读写（非 OpenBSD/非 plan9） |
| `backend/local/xattr_unsupported.go` | xattr 不支持的平台桩 |
| `backend/local/symlink.go` | 循环符号链接检测（Unix） |
| `backend/local/symlink_other.go` | 符号链接其他平台桩 |
| `backend/local/lchmod.go` | 符号链接权限修改（windows/plan9/js/linux 不支持） |
| `backend/local/lchmod_unix.go` | 符号链接权限修改（Unix 非 Linux） |
| `backend/local/lchtimes.go` | 符号链接时间修改（plan9/js 不支持） |
| `backend/local/lchtimes_unix.go` | 符号链接时间修改（Unix） |
| `backend/local/lchtimes_windows.go` | 符号链接时间修改（Windows） |
| `backend/local/setbtime.go` | btime 设置（非 Windows 不支持） |
| `backend/local/setbtime_windows.go` | btime 设置（Windows） |
| `backend/local/remove_other.go` | 文件删除（非 Windows） |
| `backend/local/remove_windows.go` | 文件删除（Windows，带重试） |
| `backend/local/read_device_unix.go` | 设备号读取（Unix 系列） |
| `backend/local/read_device_other.go` | 设备号读取（其他平台桩） |
| `backend/local/about_windows.go` | 磁盘空间查询（Windows） |
| `backend/local/about_unix.go` | 磁盘空间查询（Unix） |
| `backend/local/clone_darwin.go` | Reflink 克隆（macOS） |

平台特定元数据文件：

| 文件 | 平台 (build tag) | 说明 |
|------|-----------------|------|
| `backend/local/metadata_windows.go` | windows | Windows 平台元数据 |
| `backend/local/metadata_linux.go` | linux | Linux 平台元数据（含 statx/fstatat 双路径） |
| `backend/local/metadata_bsd.go` | darwin \|\| freebsd \|\| netbsd | macOS/FreeBSD/NetBSD 元数据 |
| `backend/local/metadata_unix.go` | openbsd \|\| solaris | OpenBSD/Solaris 元数据 |
| `backend/local/metadata_other.go` | dragonfly \|\| plan9 \|\| js \|\| aix | 其他平台桩 |

辅助库：

- `lib/file/` — 文件操作工具（UNC路径、预分配、稀疏文件等）
- `lib/encoder/` — 文件名编码转换
- `fs/fspath/` — 路径解析工具

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

核心函数：`cleanRootPath`（`backend/local/local.go`）

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

实现：`lib/file/unc_windows.go`（Windows）、`lib/file/unc.go`（非 Windows）

转换规则：

| 输入 | 输出 |
|------|------|
| `C:\path\to\file` | `\\?\C:\path\to\file` |
| `\\server\share\path` | `\\?\UNC\server\share\path` |
| `\\?\C:\already\unc` | 原样返回 |
| 非 Windows 平台 | 原样返回 |

非 Windows 平台：`UNCPath` 为恒等函数。

### 4. remote ↔ local 路径转换

**remote → local**：`localPath` 方法（`backend/local/local.go`）

```go
func (f *Fs) localPath(name string) string {
    return filepath.Join(f.root, filepath.FromSlash(f.opt.Enc.FromStandardPath(name)))
}
```

**local → remote**：`cleanRemote` 方法（`backend/local/local.go`）

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

实现：`lib/encoder/encoder.go`

本地后端默认使用 `encoder.OS`，针对不同 OS 启用不同编码规则：

- **Windows**：编码 `:?"*<>|` 等非法字符及前后导空格/点
- **macOS**：编码特殊字符以兼容 HFS+ 的 NFD 规范化
- **其他**：按需编码

编码策略：将受限字符映射到 Unicode 全角（FULLWIDTH）变体，用 `‛`（SINGLE HIGH-REVERSED-9 QUOTATION MARK）作为转义符。

### 6. 路径解析（fspath）

实现：`fs/fspath/path.go`

`Parse` 函数判断路径是本地路径还是远程路径：

- 不含 `:` → 本地路径
- 含 `:` 但为盘符（如 `C:`）→ 本地路径
- 含 `:` 且为远程名 → 远程路径

所有路径最终统一转换为 `/` 分隔。

---

## 权限处理

权限处理是本地后端跨平台差异最复杂的部分。核心矛盾在于：**Unix 以「权限位」（rwx + SUID/SGID/Sticky）为模型，Windows 以「文件属性」（Hidden/ReadOnly/System 等标志）为模型**，两者并非完全对等。Go 标准库在 Windows 上通过模拟方式提供了「类 Unix」的 `os.FileMode`，但语义被大幅简化。

### 1. Mode 位的数据模型

#### 1.1 存储格式：Unix 风格八进制

元数据中的 `mode` 字段统一使用 Unix 风格的八进制字符串表示，定义在 `backend/local/metadata.go` 的 `systemMetadataInfo`：

```go
"mode": {
    Help:    "File type and mode",
    Type:    "octal, unix style",
    Example: "0100664",
},
```

格式解析：`0100664` 是一个 7 位（及以上）的八进制数：

| 位段（八进制） | 含义 | 示例值 |
|---------------|------|--------|
| 最高位（文件类型） | `001`=FIFO, `002`=字符设备, `004`=目录, `006`=块设备, `010`=普通文件, `012`=符号链接, `014`=Socket | `010` = 普通文件 |
| 次高位（特殊位） | SUID=4, SGID=2, Sticky=1 | `0` = 无 |
| 所有者权限 | r=4, w=2, x=1 | `6` = rw- |
| 组权限 | 同上 | `6` = rw- |
| 其他用户权限 | 同上 | `4` = r-- |

因此 `0100664` = 普通文件 + 无特殊位 + 所有者rw- + 组rw- + 其他r--。

#### 1.2 Mode 读取的跨平台差异

各平台元数据读取时，mode 的来源不同：

| 平台组 | mode 来源 | 实现文件 | 说明 |
|--------|----------|----------|------|
| **真实 Unix**（Linux/macOS/FreeBSD/NetBSD/OpenBSD/Solaris） | `stat.Mode`（`syscall.Stat_t` 原生 mode 字段） | `backend/local/metadata_linux.go` / `metadata_bsd.go` / `metadata_unix.go` | 包含完整的文件类型位 + 12 位权限位（含 SUID/SGID/Sticky） |
| **Windows** | `info.Mode()`（Go 标准库模拟） | `backend/local/metadata_windows.go` | 仅保留文件类型位 + 简化权限（写位 = 非只读），SUID/SGID/Sticky 全部丢失 |
| **其他**（Dragonfly/plan9/js/aix） | `info.Mode()`（Go 标准库模拟） | `backend/local/metadata_other.go` | 同上，简化权限 |

**关键代码对比**：

```go
// Unix: 使用 syscall 原生 stat.Mode（完整信息）
// metadata_linux.go / metadata_bsd.go / metadata_unix.go
m.Set("mode", fmt.Sprintf("%0o", stat.Mode))

// Windows/Other: 使用 Go 模拟的 info.Mode()（信息丢失）
// metadata_windows.go / metadata_other.go
m.Set("mode", fmt.Sprintf("%0o", info.Mode()))
```

#### 1.3 Windows 的 FileAttributes 未被利用

在 `backend/local/metadata_windows.go` 中有一个 FIXME 注释：

```go
// FIXME do something with stat.FileAttributes ?
```

Windows 的 `syscall.Win32FileAttributeData.FileAttributes` 包含大量有价值的信息，但目前**完全未被读取或写入**：

| 属性标志 | 值 | rclone 是否处理 |
|---------|----|----------------|
| `FILE_ATTRIBUTE_READONLY` | 0x00000001 | ✅ 间接通过 `os.Chmod` 处理（见下文） |
| `FILE_ATTRIBUTE_HIDDEN` | 0x00000002 | ❌ 未处理 |
| `FILE_ATTRIBUTE_SYSTEM` | 0x00000004 | ❌ 未处理 |
| `FILE_ATTRIBUTE_DIRECTORY` | 0x00000010 | ✅ 通过 `info.Mode().IsDir()` 间接处理 |
| `FILE_ATTRIBUTE_ARCHIVE` | 0x00000020 | ❌ 未处理 |
| `FILE_ATTRIBUTE_TEMPORARY` | 0x00000100 | ❌ 未处理 |
| `FILE_ATTRIBUTE_SPARSE_FILE` | 0x00000200 | ❌ 未处理（稀疏文件仅在创建时优化） |
| `FILE_ATTRIBUTE_REPARSE_POINT` | 0x00000400 | ✅ 通过符号链接逻辑间接处理 |
| `FILE_ATTRIBUTE_COMPRESSED` | 0x00000800 | ❌ 未处理 |
| `FILE_ATTRIBUTE_OFFLINE` | 0x00001000 | ❌ 未处理 |
| `FILE_ATTRIBUTE_NOT_CONTENT_INDEXED` | 0x00002000 | ❌ 未处理 |
| `FILE_ATTRIBUTE_ENCRYPTED` | 0x00004000 | ❌ 未处理 |

### 2. Mode 写入的跨平台语义差异

#### 2.1 核心流程：writeMetadataToFile

`writeMetadataToFile` 方法（`backend/local/metadata.go`）是权限写入的总入口，处理顺序为：

```
times (atime/mtime/btime) → uid/gid (所有权) → mode (权限)
```

Mode 写入的详细流程：

```go
// 1. 从元数据解析 mode（八进制）
mode, hasMode := o.parseMetadataInt(m, "mode", 8)

// 2. 合法性校验
if hasMode && mode >= 0 && uint(mode) <= math.MaxUint32 {
    // 3. 判断目标是否为符号链接
    if o.translatedLink {
        // 4a. 符号链接：需要 lChmod（平台支持有限）
        if haveLChmod {
            err = lChmod(o.path, os.FileMode(umode))
        } else {
            // 不支持时仅记录 debug，不报错（优雅降级）
            fs.Debugf(o, "Unable to set mode %v on a symlink on this OS", ...)
        }
    } else {
        // 4b. 普通文件/目录：os.Chmod
        err = os.Chmod(o.path, os.FileMode(umode))
    }
}
```

#### 2.2 os.Chmod 在 Windows 上的实际效果

Windows 上调用 `os.Chmod(path, mode)` 时，Go 运行时仅处理**只读位**：

| mode 值 | 对 Windows 文件的影响 |
|---------|---------------------|
| 任意位包含「写权限」被清除（mode & 0222 == 0） | 设置 `FILE_ATTRIBUTE_READONLY` 标志 |
| 任意位包含「写权限」（mode & 0222 != 0） | 清除 `FILE_ATTRIBUTE_READONLY` 标志 |
| 读/执行权限位 | 完全忽略 |
| SUID/SGID/Sticky 位 | 完全忽略 |
| 文件类型位（高 4 位） | 完全忽略 |

换句话说，Windows 上 `os.Chmod` 是一个「**只有写位有效**」的降级实现。

#### 2.3 os.Chmod 在 Unix 上的实际效果

Unix 上 `os.Chmod` 会完整设置权限，但需注意：

- 只有文件所有者或 root 才能修改权限
- SUID/SGID 位在某些文件系统上可能被内核清除
- Sticky 位仅对目录有意义

### 3. 符号链接权限修改（lChmod）

核心函数：`lChmod` — 修改符号链接本身的权限而非其指向的目标。

| 平台 | 支持情况 | 实现文件 | build tag |
|------|----------|----------|-----------|
| Linux | ❌ 不支持 | `backend/local/lchmod.go` | `linux` |
| Windows | ❌ 不支持 | `backend/local/lchmod.go` | `windows` |
| macOS/FreeBSD/NetBSD/OpenBSD/Solaris 等 | ✅ 支持 | `backend/local/lchmod_unix.go` | 非 linux 的 Unix |
| plan9/js | ❌ 不支持 | `backend/local/lchmod.go` | `plan9 \|\| js` |

不支持的平台上 `haveLChmod = false`，调用时仅打 debug 日志，不报错。

**Linux 不支持原因**：Linux 的 `fchmodat` 系统调用不接受 `AT_SYMLINK_NOFOLLOW` 标志，传入时返回 `ENOTSUP`。Linux 内核从设计上就不允许修改符号链接的权限位（符号链接始终为 0777）。

**Unix 实现**（`backend/local/lchmod_unix.go`）：

```go
func syscallMode(i os.FileMode) (o uint32) {
    o |= uint32(i.Perm())         // 0-9 位: rwxrwxrwx
    if i&os.ModeSetuid != 0 { o |= syscall.S_ISUID }  // SUID
    if i&os.ModeSetgid != 0 { o |= syscall.S_ISGID }  // SGID
    if i&os.ModeSticky != 0 { o |= syscall.S_ISVTX }  // Sticky
    return
}

func lChmod(name string, mode os.FileMode) error {
    // NB linux does not support AT_SYMLINK_NOFOLLOW as a parameter to fchmodat
    return unix.Fchmodat(unix.AT_FDCWD, name, syscallMode(mode), unix.AT_SYMLINK_NOFOLLOW)
}
```

**特殊注意**：在 Linux 上通过元数据同步符号链接权限会**静默失败**，仅记录 debug 日志，不会导致同步任务整体失败。

### 4. 文件创建权限

本地后端创建文件/目录时传入的权限位：

| 操作 | 请求权限 | 代码位置 | 说明 |
|------|---------|----------|------|
| 创建普通文件 | `0666` (rw-rw-rw-) | `Update` / `PartialUploads` 方法 (`backend/local/local.go`) | 受 umask 修正 |
| 创建目录 | `0777` (rwxrwxrwx) | `Mkdir` / `MkdirAll` / `serverSideMove` (`backend/local/local.go`) | 受 umask 修正 |

**跨平台差异**：

- **Unix**：内核会执行 `mode & ~umask`，常见 umask 为 022 时文件实际为 0644，目录实际为 0755
- **Windows**：传入的权限位参数基本被忽略，安全描述符由父目录 ACL 继承决定，仅 `FILE_ATTRIBUTE_READONLY` 可能受后续 `os.Chmod` 影响

### 5. 所有权处理（Chown）

实现位置：`writeMetadataToFile` 方法（`backend/local/metadata.go`）

执行顺序：

```go
// 1. 解析 uid/gid
uid, hasUID := o.parseMetadataInt(m, "uid", 10)
gid, hasGID := o.parseMetadataInt(m, "gid", 10)

// 2. 如果只提供了 uid，gid 复用 uid 的值（FIXME 行为）
if hasUID && !hasGID {
    gid = uid
}

// 3. 平台判断
if runtime.GOOS == "windows" || runtime.GOOS == "plan9" {
    fs.Debugf(o, "Ignoring request to set ownership %o.%o on this OS", gid, uid)
} else {
    // 4. 符号链接用 Lchown，普通文件用 Chown
    if o.translatedLink {
        err = os.Lchown(o.path, uid, gid)
    } else {
        err = os.Chown(o.path, uid, gid)
    }
}
```

| 平台 | 支持情况 | 说明 |
|------|----------|------|
| Windows | ❌ 静默忽略 | 仅 debug 日志，Windows 使用 ACL 模型 |
| plan9 | ❌ 静默忽略 | 仅 debug 日志 |
| Unix 系列 | ✅ 支持 | 需要 root 或 CAP_CHOWN 能力才能修改为非当前用户 |

**已知设计缺陷**（代码中标注 FIXME）：

- 未读取当前用户的 uid/gid，即使目标值与当前值相同也会尝试设置，可能产生不必要的 EPERM 错误
- 只设置 uid 时 gid 强制取 uid 的值，可能不符合预期

### 6. 时间相关的权限控制（NoSetModTime）

配置项 `NoSetModTime`（`--local-no-set-modtime`）禁用修改时间设置：

```go
// backend/local/local.go - Precision 方法
if f.opt.NoSetModTime {
    return fs.ModTimeNotSupported
}
```

适用场景：

- 无写权限的只读文件系统
- 无法修改时间的挂载点（如某些 FUSE 文件系统）
- 性能优化场景，跳过 utimensat 系统调用

### 7. 权限错误处理

#### 7.1 目录列表权限

`List` 方法（`backend/local/local.go`）中处理目录打开的权限错误：

```
os.Open 目录失败
    ↓
错误为 fs.ErrPermission (Permission denied)
    ↓
记录 fs.Errorf 错误日志
    ↓
err = fs.NoRetryError(err)  // 标记为不可重试，避免反复尝试
    ↓
返回 (nil, nil)  // 不将错误向上抛出，但错误已被统计
```

这种处理策略确保单个无权限的目录不会导致整个同步任务失败。

#### 7.2 单独文件 stat 失败

**Windows/Plan9** 使用 `Readdir()` 批量读取：
- 单个文件 stat 错误可能被 Go 标准库吸收
- 目录句柄打开失败时整个目录列表失败

**其他 OS** 使用 `Readdirnames()` + 逐个 `Lstat`：
- 单个文件的 stat 失败被 `fs.Errorf` 记录
- 不会终止整个目录遍历，继续处理其他文件
- 被过滤器（如 `--exclude`）排除的文件不报告错误

### 8. Windows 特殊权限 Workaround

#### 8.1 隐藏/系统文件更新

`Update` 方法（`backend/local/local.go`）：

```
尝试以 O_WRONLY|O_CREATE|O_TRUNC 打开文件
    ↓ 失败且错误为 Permission denied
    ↓ （可能是 FILE_ATTRIBUTE_HIDDEN 或 FILE_ATTRIBUTE_SYSTEM）
以 O_WRONLY|O_TRUNC（不带 O_CREATE）重新打开
    ↓ 成功则继续写入
```

背景：Windows 上如果文件带有隐藏或系统属性，带有 `CREATE_ALWAYS` 的打开可能失败。使用 `TRUNCATE_EXISTING`（不带创建标志）可以绕过这个限制。

#### 8.2 只读目录删除

`Rmdir` 方法（`backend/local/local.go`）：

```
尝试 os.Remove 删除目录
    ↓ 失败且错误为 ErrPermission
os.Chmod(path, 0o600)  // 先尝试清除只读属性
    ↓
再次尝试 os.Remove
```

对应 Go issue #26295：即使微软文档声称 `FILE_ATTRIBUTE_READONLY` 对目录无效，但在某些情况下仍会干扰删除操作。此 workaround 先强制设置写权限再删除。

测试用例见 `backend/local/local_internal_windows_test.go` 的 `TestRmdirWindows`。

---

## 跨平台差异实现边界

### 1. 文件系统特性

#### 大小写敏感性

`caseInsensitive` 方法（`backend/local/local.go`）

默认判定：

| 平台 | 默认大小写敏感 |
|------|---------------|
| Windows | ❌ 不敏感 |
| macOS | ❌ 不敏感 |
| 其他 | ✅ 敏感 |

可通过 `--local-case-sensitive` / `--local-case-insensitive` 强制覆盖。

> 注意：此判定不完全准确（如 macOS 可格式化为大小写敏感的 APFS），但作为默认值是合理的。

#### 单文件系统边界（one_file_system）

`readDevice` 函数：

- Unix 系列（darwin/dragonfly/freebsd/linux/netbsd/openbsd/solaris）：从 `syscall.Stat_t.Dev` 读取设备号
- 其他平台：始终返回 `devUnset`

启用 `--one-file-system` 时，跨设备的目录被跳过。

### 2. 时间类型支持

时间类型有两套独立的实现路径：

1. **`time_type` 选项**（用于 `ModTime()` 返回值）：使用 `readTime` 函数，从 `os.FileInfo.Sys()` 读取
2. **Metadata API**（用于 `--metadata` 标志）：使用 `readMetadataFromFile` 函数

#### 2.1 time_type 选项（readTime 函数）

`readTime` 函数各平台实现：

| 时间类型 | Windows | Linux | macOS/FreeBSD/NetBSD | OpenBSD/Solaris | Dragonfly/plan9/js/aix |
|---------|---------|-------|----------------------|-----------------|------------------------|
| mtime | ✅ | ✅ | ✅ | ✅ | ✅ |
| atime | ✅ | ✅ | ✅ | ✅ | ❌ |
| btime (创建/出生) | ✅ (CreationTime) | ❌ | ✅ (Birthtimespec) | ❌ | ❌ |
| ctime (状态变更) | ❌ | ✅ | ✅ | ✅ | ❌ |

实现文件：
- Windows: `backend/local/metadata_windows.go`
- Linux: `backend/local/metadata_linux.go`
- macOS/FreeBSD/NetBSD: `backend/local/metadata_bsd.go`
- OpenBSD/Solaris: `backend/local/metadata_unix.go`
- 其他: `backend/local/metadata_other.go`

**关键修正**：Linux 的 `readTime` 函数**不支持 btime**。因为标准 `syscall.Stat_t` 在 Linux 上没有 birth time 字段，只有通过 `statx()` 系统调用（用于 Metadata API）才能获取 btime。

#### 2.2 Metadata API（readMetadataFromFile）

| 字段 | Windows | Linux (statx) | Linux (fstatat) | macOS/BSD | OpenBSD/Solaris | 其他 |
|------|---------|--------------|-----------------|-----------|-----------------|------|
| mode | ✅ (简化) | ✅ | ✅ | ✅ | ✅ | ✅ |
| uid | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ |
| gid | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ |
| rdev | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ |
| atime | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| mtime | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| btime | ✅ | ✅ (内核 4.11+) | ❌ | ✅ | ❌ | ❌ |
| ctime | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

**说明**：

- **Linux btime**：通过 `statx()` 系统调用获取（内核 4.11+），旧内核回退到 `fstatat()` 则不支持 btime
- **ctime 在 Metadata API 中普遍缺失**：虽然很多平台的 stat 结构有 ctime，但 `readMetadataFromFile` 未将其写入元数据 map
- **Android 特殊处理**：Linux 代码中排除了 Android 平台（`runtime.GOOS != "android"`），Android 始终走 fstatat 路径

#### 2.3 时间设置（写入）

| 操作 | Windows | Unix (非 Windows) |
|------|---------|------------------|
| 设置 mtime/atime | ✅ | ✅ |
| 设置 btime | ✅ | ❌ |
| 符号链接设置 mtime/atime | ✅ | ✅ |
| 符号链接设置 btime | ✅ | ❌ |

实现：
- btime 设置：`backend/local/setbtime_windows.go`（Windows 支持）、`backend/local/setbtime.go`（其他平台空实现）
- 符号链接时间设置：`backend/local/lchtimes_windows.go`、`backend/local/lchtimes_unix.go`

### 3. 符号链接差异

| 特性 | Unix | Windows |
|------|------|---------|
| 符号链接类型标志 | `os.ModeSymlink` | `os.ModeSymlink \| os.ModeIrregular` |
| Junction Points | N/A | 视为符号链接处理 |
| 循环检测 | `ELOOP` 错误 (syscall) | 无专门检测 |
| lchmod | 部分支持 (Linux 除外) | 不支持 |
| lchown | 支持 | 不支持 |
| lchtimes | 支持 (非 plan9/js) | 支持 |

Windows 特殊处理：

```go
symlinkFlag := os.ModeSymlink
if runtime.GOOS == "windows" {
    symlinkFlag |= os.ModeIrregular
}
```

循环符号链接检测（Unix only）：通过 `syscall.ELOOP` 判断（`backend/local/symlink.go`）。

### 4. 元数据（Metadata）

#### 系统元数据

参见上表「Metadata API」部分。

#### 扩展属性（xattr）

实现：`backend/local/xattr.go`

| 平台 | 支持用户 xattr | 说明 |
|------|---------------|------|
| Linux | ✅ | ext4/xfs/btrfs 等 |
| macOS | ✅ | HFS+/APFS |
| FreeBSD | ✅ | |
| NetBSD | ✅ | |
| Solaris | ✅ | |
| Windows | ❌ | pkg/xattr#47 未解决 |
| OpenBSD | ❌ | 编译排除 (build tag: !openbsd && !plan9) |
| Plan9 | ❌ | 编译排除 |

xattr 前缀：`user.`（Unix 惯例）

**动态降级**：运行时遇 `ENOTSUP`/`ENOATTR`/`EINVAL` 错误时，自动标记 xattr 不支持并静默降级。

### 5. 文件删除行为

- **非 Windows**：直接 `os.Remove`，一次尝试（`backend/local/remove_other.go`）
- **Windows**：`backend/local/remove_windows.go`
  - 遇 `ERROR_SHARING_VIOLATION` 时指数退避重试
  - 最多重试 10 次
  - 初始等待 1ms，每次翻倍

### 6. 文件打开行为

实现：`lib/file/file_windows.go`（Windows）、`lib/file/file_other.go`（非 Windows）

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

`List` 方法（`backend/local/local.go`）

**Windows/Plan9**：使用 `Readdir()` 批量读取 FileInfo
- 优点：性能好，一次系统调用获得全部信息
- 缺点：单个条目错误可能影响整个目录读取

**其他 OS**：使用 `Readdirnames()` 读取名称，再逐个 `Lstat`
- 优点：单个文件错误不阻断整个目录列表
- 缺点：系统调用次数多，性能略低

### 9. 保留名称检查

`IsReserved` 函数（`lib/file/file_windows.go`，仅 Windows）

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
| NoClone | 禁用 reflink 克隆 | macOS only (功能仅 macOS 有) |
| FatalIfNoSpace | 磁盘满时返回致命错误 | 全平台 |
| NoCheckUpdated | 不上传时检查文件变化 | 全平台 |

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

## 设计原则总结

本地文件系统后端的跨平台策略遵循以下设计原则：

1. **分层抽象**：核心逻辑在 `local.go`，平台差异通过 build tag 分离到 `_windows.go`/`_unix.go`/`_other.go` 等文件

2. **优雅降级**：高级特性（xattr、btime、reflink 等）不可用时静默降级，不报错

3. **路径双轨制**：remote 路径（标准化）与 local 路径（OS 原生）分离，通过 encoder 转换

4. **Windows 特殊照顾**：UNC 路径、共享模式、保留名称、隐藏文件、删除重试等

5. **动态检测**：部分特性（如 Linux statx）运行时探测，不可用则回退

6. **两套时间 API**：`time_type` 选项和 Metadata API 使用不同的实现路径，支持范围不完全一致
