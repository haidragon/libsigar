# Windows 系统信息采集实现文档 (SIGAR)

## 1. 概述 (Overview)

本文档详细描述了 **SIGAR (System Information Gatherer And Reporter)** 库在 **Microsoft Windows** 操作系统上的底层实现机制。与 HP-UX 版本依赖专有的 `pstat` 和 MIB 接口不同，Windows 版本主要依托于 **Windows API**、**Performance Data Helper (PDH)** 库以及 **WMI (Windows Management Instrumentation)** 的底层接口，来实现对 CPU、内存、进程、网络、文件系统等核心监控指标的高效采集。

### 1.1 核心设计目标

- **原生集成**：深度利用 Windows NT 内核 API (`NtQuerySystemInformation`) 和 PDH 接口，确保数据源的原生性和准确性。
- **高性能**：避免频繁启动 `wmic` 等外部进程，直接通过 DLL 调用获取内核数据。
- **兼容性**：支持从 Windows 2000/XP 到 Windows 10/11 及 Windows Server 全系列，兼容 x86, x64 架构。
- **Unicode 支持**：全面处理 Windows 的宽字符 (UTF-16) 字符串，确保进程名、路径等信息的正确显示。

---

## 2. 核心依赖与接口 (Core Dependencies)

代码主要依赖以下三类 Windows 特有接口：

### 2.1 Performance Data Helper (PDH) API (主要性能数据源)

位于 `<pdh.h>` 和 `pdh.lib`。这是 Windows 推荐的性能计数器访问接口，比直接注册表查询更高效、更稳定。
| 函数名 | 用途 | 对应 SIGAR 功能 |
| :--- | :--- | :--- |
| `PdhOpenQuery` | 创建性能查询句柄 | 初始化性能采集会话 |
| `PdhAddCounter` | 添加具体的性能计数器路径 (如 `\Processor(_Total)\% Processor Time`) | 绑定 CPU、内存、磁盘、网络指标 |
| `PdhCollectQueryData` | 收集当前样本数据 | 刷新所有绑定的计数器值 |
| `PdhGetFormattedCounterValue` | 获取格式化后的数值 (Double, Large Int) | 读取具体的 CPU 使用率、字节数等 |
| `PdhCloseQuery` | 关闭查询句柄 | 清理资源 |

**常用计数器路径示例**:

- CPU: `\Processor(_Total)\% Processor Time`
- 内存: `\Memory\Available Bytes`, `\Memory\Committed Bytes`
- 磁盘: `\LogicalDisk(C:)\Disk Read Bytes/sec`
- 网络: `\Network Interface(*)\Bytes Total/sec`

### 2.2 Windows Native API & Kernel32 (系统与进程信息)

位于 `<windows.h>`, `<winternl.h>`。用于获取 PDH 无法提供的底层结构和列表。

- **`NtQuerySystemInformation`** (ntdll.dll):
  - `SystemProcessInformation`: 获取完整的进程列表（PID, PPID, 线程数，句柄数，IO 计数）。
  - `SystemPerformanceInformation`: 获取全局性能快照。
  - `SystemTimeOfDayInformation`: 获取系统启动时间。
- **`CreateToolhelp32Snapshot` / `Process32First` / `Process32Next`**:
  - 用于遍历进程列表，获取进程名、父 PID 等传统信息（作为 `NtQuery` 的备选或补充）。
- **`GetProcessTimes`**, **`GetProcessMemoryInfo`** (PSAPI):
  - 获取特定进程的用户态/内核态时间、工作集大小 (Working Set)、私有字节等。
- **`GetTickCount64`**: 获取系统运行毫秒数。

### 2.3 Winsock & IP Helper API (网络信息)

位于 `<winsock2.h>`, `<iphlpapi.h>`。

- **`GetExtendedTcpTable` / `GetExtendedUdpTable`**:
  - 获取 TCP/UDP 连接表，包含本地/远程 IP、端口、状态、 owning PID。
- **`GetAdaptersAddresses`** (替代旧的 `GetAdaptersInfo`):
  - 获取网卡详细信息（名称、MAC 地址、IPv4/IPv6 地址、DNS、DHCP 状态）。
- **`GetIfEntry2`**: 获取网卡的流量统计（字节、包、错误、丢弃）。

### 2.4 文件系统 API

- **`GetLogicalDriveStrings`**, **`GetDriveType`**: 枚举磁盘驱动器。
- **`GetDiskFreeSpaceEx`**: 获取磁盘总容量、可用空间、用户可用空间。
- **`FindFirstVolume` / `GetVolumePathNamesForVolumeName`**: 处理挂载点和卷标。

---

## 3. 功能模块详解 (Module Details)

### 3.1 初始化与清理 (Initialization)

- **`sigar_os_open`**:
  - 初始化 PDH 查询句柄 (`PdhOpenQuery`)。
  - 预加载常用的性能计数器路径 (`PdhAddCounter`)，如 CPU 总使用率、可用内存等。
  - 加载 `ntdll.dll` 并获取 `NtQuerySystemInformation` 函数指针（因为该函数未公开在头文件中）。
  - 初始化 Unicode 转换环境。
- **`sigar_os_close`**:
  - 关闭所有 PDH 查询句柄 (`PdhCloseQuery`)。
  - 释放动态加载的 DLL 资源。
  - 清理缓存的字符串和列表。

### 3.2 内存统计 (Memory Statistics)

- **实现函数**: `sigar_mem_get`
- **逻辑**:
  1.  **全局内存状态**: 调用 `GlobalMemoryStatusEx` 获取 `MEMORYSTATUSEX` 结构体。
      - `dwTotalPhys`: 物理内存总量。
      - `dwAvailPhys`: 物理内存空闲量。
      - `dwTotalPageFile`: 提交限制 (Commit Charge Limit)。
      - `dwAvailPageFile`: 可用提交量。
  2.  **详细页面统计**: 通过 PDH 计数器 `\Memory\Pages/sec`, `\Memory\Page Faults/sec` 获取分页活动速率。
  3.  **计算**:
      - `Used = Total - Free`.
      - `Actual Used/Free`: 考虑系统缓存 (System Cache) 的调整值。

### 3.3 CPU 统计 (CPU Statistics)

- **实现函数**: `sigar_cpu_get`, `sigar_cpu_list_get`
- **逻辑**:
  1.  **总 CPU**:
      - 方法 A (推荐): 读取 PDH 计数器 `\Processor(_Total)\% Processor Time`。需采集两次样本计算差值以获得瞬时速率，或直接读取累积时间计算。
      - 方法 B (底层): 调用 `NtQuerySystemInformation(SystemProcessorPerformanceInformation)` 获取每个处理器的 `KERNEL_TIME`, `USER_TIME`, `IDLE_TIME`。
  2.  **单核 CPU**:
      - 遍历 PDH 计数器 `\Processor(0)\% Processor Time`, `\Processor(1)...`。
      - 或者遍历 `SystemProcessorPerformanceInformation` 数组。
  3.  **时间转换**: Windows 时间单位为 **100 纳秒 (100ns)**。
      - 转换公式: `Milliseconds = Ticks / 10000`.
  4.  **分类**: 将 `Kernel Time` 拆分为 System 和 Idle (如果可能)，通常 `Kernel - Idle = System`。

### 3.4 进程管理 (Process Management)

#### 3.4.1 进程列表 (`sigar_os_proc_list_get`)

- **首选方案**: `NtQuerySystemInformation(SystemProcessInformation)`。
  - 返回一个巨大的缓冲区，包含链表结构的 `SYSTEM_PROCESS_INFORMATION`。
  - 遍历链表提取 `UniqueProcessId` (PID)。
  - **优势**: 一次性获取所有进程基本信息，速度极快，无竞态条件。
- **备选方案**: `CreateToolhelp32Snapshot` + `Process32First/Next`。
  - 用于旧版 Windows 或作为 fallback。

#### 3.4.2 进程详情

- **打开进程**: 使用 `OpenProcess(PROCESS_QUERY_INFORMATION | PROCESS_VM_READ, ...)`。
  - _注意_: 对于系统进程 (如 System Idle Process)，可能无法打开，需特殊处理。
- **内存 (`sigar_proc_mem_get`)**:
  - 调用 `GetProcessMemoryInfo` (PSAPI) 获取 `WORKING_SET_SIZE` (Resident), `PRIVATE_USAGE` (Size)。
  - 从 `SYSTEM_PROCESS_INFORMATION` (如果之前缓存了) 获取 Page Fault 计数。
- **时间 (`sigar_proc_time_get`)**:
  - 调用 `GetProcessTimes` 获取 `CreationTime`, `ExitTime`, `KernelTime`, `UserTime`。
  - 转换为毫秒。
- **状态 (`sigar_proc_state_get`)**:
  - Windows 没有直接的 "State" 字段。通常通过检查线程状态推断，或简单标记为 "Running" / "Sleeping"。
  - 如果进程已退出但未被回收，标记为 'Z' (Zombie，但在 Windows 中很少见，通常是句柄未关)。
  - 主要通过 `GetExitCodeProcess` 判断是否还在运行。
- **命令行 (`sigar_os_proc_args_get`)**:
  - **高权限**: 使用 `NtQueryInformationProcess(..., ProcessCommandLine, ...)` 直接从内核读取 `UNICODE_STRING`。
  - **低权限**: 尝试读取远程进程内存 (复杂且不稳定)，或返回空。
- **可执行文件路径 (`sigar_proc_exe_get`)**:
  - 调用 `GetModuleFileNameEx` (PSAPI) 或 `QueryFullProcessImageName` (Vista+)。
  - 处理 Unicode 转 UTF-8/ASCII。

### 3.5 网络统计 (Network Statistics)

- **实现函数**: `sigar_net_interface_stat_get`, `sigar_net_connection_walk`
- **接口流量**:
  - 调用 `GetAdaptersAddresses` 获取适配器列表和名称。
  - 调用 `GetIfEntry2` (传入 LUID) 获取精确的 `InOctets`, `OutOctets`, `Discards`, `Errors`。
- **连接列表**:
  - **TCP**: 调用 `GetExtendedTcpTable` (设置 `TCP_TABLE_OWNER_PID_ALL`)。
    - 返回包含 PID 的 `MIB_TCPROW_OWNER_PID` 数组。
    - 映射状态: `MIB_TCP_STATE_ESTAB` -> `ESTABLISHED`, `LISTEN` -> `LISTEN` 等。
  - **UDP**: 调用 `GetExtendedUdpTable` (设置 `UDP_TABLE_OWNER_PID`)。
    - UDP 无状态，仅区分监听 (LocalAddr != 0) 或非监听。
- **路由与 ARP**:
  - 路由: `GetIpForwardTable2` (IPv4/IPv6)。
  - ARP: `GetIpNetTable2`。

### 3.6 文件系统与磁盘 I/O (File System & Disk I/O)

- **文件系统列表 (`sigar_file_system_list_get`)**:
  - 调用 `GetLogicalDriveStrings` 获取盘符列表 (C:\, D:\...)。
  - 调用 `GetDriveType` 区分 `DRIVE_FIXED`, `DRIVE_CDROM`, `DRIVE_REMOTE`。
  - 调用 `GetVolumeInformation` 获取文件系统类型 (NTFS, FAT32)。
- **磁盘使用量 (`sigar_file_system_usage_get`)**:
  - 调用 `GetDiskFreeSpaceEx` 获取 `TotalBytes`, `FreeBytes`, `TotalNumberOfBytes`。
- **磁盘 I/O (`sigar_disk_usage_get`)**:
  - **PDH 方式**: 读取计数器 `\PhysicalDisk(*)\Disk Read Bytes/sec`, `\LogicalDisk(*)\Disk Write Bytes/sec`。
  - **DeviceIoControl 方式**: 打开 `\\.\PhysicalDrive0`，发送 `IOCTL_DISK_PERFORMANCE` 控制码获取底层统计 (需要管理员权限)。
  - SIGAR 通常优先使用 PDH，因为它自动处理多分区和逻辑卷映射。

---

## 4. 关键数据结构与类型转换

### 4.1 时间单位转换

Windows 内核时间戳是 **FILETIME** 结构 (100 纳秒间隔，自 1601 年起)。

```c
// 转换为毫秒
#define FILETIME_TO_MSEC(ft) \
    ((sigar_uint64_t)(((ft).dwHighDateTime << 8) | (ft).dwLowDateTime) / 10000)

// 计算相对时间 (如进程运行时间)
start_time = FILETIME_TO_MSEC(creation_time) - FILETIME_TO_MSEC(system_boot_time);
```

### 4.2 字符编码转换 (Unicode <-> ASCII/UTF-8)

Windows API 广泛使用 `WCHAR` (UTF-16)。SIGAR 内部通常使用 `char` (UTF-8 或 Local ANSI)。

- **工具函数**: `WideCharToMultiByte` 和 `MultiByteToWideChar`。
- **逻辑**:
  1.  调用 Windows API 获取 `WCHAR*` 字符串。
  2.  计算所需缓冲区大小。
  3.  分配内存并转换。
  4.  存入 SIGAR 结构体。

### 4.3 进程状态映射

Windows 没有直接的进程状态枚举，通常根据上下文推断：
| 推断逻辑 | SIGAR State |
| :--- | :--- |
| 进程存在且未退出 | 'R' (Running) 或 'S' (Sleeping) |
| 线程主要在 Wait 状态 | 'S' |
| `GetExitCodeProcess` != STILL*ACTIVE | 'Z' (或视为已死) |
| *注\_: Windows 进程模型中，用户态看到的通常是 "Running" 或 "Stopped" (调试时)。

### 4.4 TCP 状态映射

| Windows MIB State          | SIGAR State             |
| :------------------------- | :---------------------- |
| `MIB_TCP_STATE_LISTEN`     | `SIGAR_TCP_LISTEN`      |
| `MIB_TCP_STATE_ESTAB`      | `SIGAR_TCP_ESTABLISHED` |
| `MIB_TCP_STATE_SYN_SENT`   | `SIGAR_TCP_SYN_SENT`    |
| `MIB_TCP_STATE_TIME_WAIT`  | `SIGAR_TCP_TIME_WAIT`   |
| `MIB_TCP_STATE_CLOSE_WAIT` | `SIGAR_TCP_CLOSE_WAIT`  |

---

## 5. 版本兼容性与条件编译

1.  **API 可用性**:
    - **Vista/Server 2008+**: 推荐使用 `QueryFullProcessImageName`, `GetIfEntry2`, `GetDiskFreeSpaceEx`。
    - **XP/Server 2003**: 需回退到 `GetModuleFileNameEx`, `GetIfEntry`, 旧版 IP Helper API。
    - **检测机制**: 使用 `GetProcAddress` 动态加载新 API，如果失败则使用旧 API 或返回 `ENOTIMPL`。
2.  **架构差异**:
    - **x64**: 指针长度为 8 字节，`SYSTEM_PROCESS_INFORMATION` 结构对齐方式不同。代码需使用 `ULONG_PTR` 而非固定 `int` 来处理偏移量。
    - **Wow64**: 32 位程序监控 64 位系统时，部分 API (如 `NtQuerySystemInformation`) 可能需要特殊处理才能获取完整的 64 位进程信息。
3.  **权限模型**:
    - **UAC (User Account Control)**: 在 Vista+ 上，即使管理员也需要提升权限 (Elevated) 才能访问某些进程 (如 `csrss.exe`, `system`) 或性能计数器。代码需处理 `ACCESS_DENIED` 错误。

---

## 6. 性能优化策略

1.  **PDH 句柄复用**:
    - 在 `sigar_t` 结构中持久化 `PDH_HQUERY` 和 `PDH_HCOUNTER` 句柄。
    - 每次采集仅调用 `PdhCollectQueryData` 和 `PdhGetFormattedCounterValue`，避免重复添加计数器的高开销。
2.  **批量进程查询**:
    - 使用 `NtQuerySystemInformation(SystemProcessInformation)` 一次性抓取全量进程信息，然后在内存中解析，避免对每个 PID 单独调用 `OpenProcess` (这非常慢且消耗句柄)。
3.  **缓存策略**:
    - 对不频繁变化的数据 (如磁盘列表、网卡名称、进程路径) 进行短时缓存。
    - 对高频变化数据 (CPU, 内存，网络流量) 实时查询。
4.  **延迟加载**:
    - 不静态链接所有 API，而是运行时动态加载 (`GetProcAddress`)，确保在旧版 Windows 上也能启动（即使部分功能不可用）。

---

## 7. 局限性与注意事项

1.  **权限限制 (Access Denied)**:
    - 普通用户无法获取其他用户的进程详细信息 (内存、命令行、路径)。
    - 部分性能计数器 (如磁盘 I/O 细分) 可能需要管理员权限。
    - **对策**: 捕获 `ERROR_ACCESS_DENIED` 并跳过该进程或返回默认值，不要导致整个程序崩溃。
2.  **Unicode 复杂性**:
    - Windows 路径和进程名可能包含任意 Unicode 字符。转换过程中需注意缓冲区溢出和编码丢失问题。
3.  **容器与虚拟化**:
    - 在 Docker (Windows Containers) 或 Hyper-V 中，某些全局计数器 (如物理内存总量) 可能反映的是宿主机信息而非容器限制，需结合 Job Object API 获取准确限制。
4.  **PDH 计数器本地化**:
    - 早期 Windows 版本的性能计数器名称可能是本地化的 (非英文)。
    - **对策**: 现代 PDH API 通常接受英文路径并自动映射，但在极少数旧系统上可能需要查询注册表 `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Perflib` 进行映射。SIGAR 通常假设英文路径或使用数字 ID。
5.  **句柄泄漏风险**:
    - 遍历进程时必须确保 `CloseHandle` 每一个 `OpenProcess` 返回的句柄，否则会导致系统句柄表耗尽。

---

## 8. 总结

Windows 版本的 SIGAR 实现充分利用了 **PDH** 进行高性能指标采集，结合 **Native API** (`NtQuery...`) 和 **IP Helper** 进行深层系统和网络探测。其核心挑战在于处理 **Unicode 编码**、**UAC 权限模型** 以及 **不同 Windows 版本间的 API 差异**。通过动态加载、降级处理和精细的资源管理，该模块能够在各种 Windows 环境下提供稳定、准确的系统监控数据，是 Windows 平台运维工具和监控代理的理想底层支撑。
