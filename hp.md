# HP-UX 系统信息采集实现文档 (SIGAR)

## 1. 概述 (Overview)

本文档详细描述了 **SIGAR (System Information Gatherer And Reporter)** 库在 **HP-UX** 操作系统上的底层实现机制。该模块 (`sigar_proc_hpux.c`) 负责屏蔽操作系统差异，为上层应用提供统一的 API 以获取 CPU、内存、进程、网络、文件系统等核心监控指标。

### 1.1 核心设计目标

- **高性能**：优先使用内核专用的 `pstat` 系列系统调用，避免低效的文件系统遍历。
- **准确性**：直接读取内核数据结构，确保统计数据的实时性和精确度。
- **兼容性**：适配 HP-UX 11.00 至 11.31+ 多个版本，支持 PA-RISC 和 Itanium (IA64) 架构。
- **标准化**：将 HP-UX 特有的数据结构转换为 SIGAR 通用的标准格式。

---

## 2. 核心依赖与接口 (Core Dependencies)

代码主要依赖以下三类系统接口来获取信息：

### 2.1 HP-UX 专用 `pstat` 接口 (主要数据源)

位于 `<sys/pstat.h>`，是 HP-UX 提供的最高效的系统状态查询接口。
| 函数名 | 用途 | 对应 SIGAR 功能 |
| :--- | :--- | :--- |
| `pstat_getstatic` | 获取静态系统信息 (内存总量, 页大小, 启动时间) | 初始化, 内存总量, 运行时间 |
| `pstat_getdynamic` | 获取动态系统信息 (CPU 时间, 负载, 进程数) | CPU 使用率, 系统负载 |
| `pstat_getproc` / `pstat_getprocs` | 获取单个或多个进程的状态详情 | 进程列表, 进程内存, 进程状态 |
| `pstat_getprocessor` | 获取单个 CPU 核心的详细信息 | CPU 核心列表 |
| `pstat_getswap` | 获取交换分区使用情况 | Swap 使用量 |
| `pstat_getvminfo` | 获取虚拟内存统计 (页换入/换出) | Swap 页面活动 |
| `pstat_getlv` | 获取逻辑卷 (Logical Volume) I/O 统计 | 磁盘读写速率 |
| `pstat_getcommandline` | 获取进程完整命令行 | 进程参数 |
| `pstat_getpathname` | 通过文件 ID 获取文件路径 | 进程执行路径, cwd, root |

### 2.2 SNMP MIB 接口 (网络数据源)

位于 `<sys/mib.h>`，通过 `/dev/ip` 设备访问内核网络栈的 SNMP 管理信息库。

- **关键函数**: `open_mib`, `get_mib_info`, `close_mib`。
- **用途**: 获取网卡流量统计、TCP/UDP 连接表、路由表、ARP 表、TCP 全局统计。
- **机制**: 构造 `struct nmparms`，指定 OID (Object ID)，通过 `ioctl` 获取数据。

### 2.3 标准 POSIX/Unix 接口 (辅助数据源)

- **`sysconf(_SC_CLK_TCK`)**: 获取 CPU 时钟频率 (Ticks per Second)。
- **`getmntent` / `/etc/mnttab`**: 读取文件系统挂载表。
- **`stat` / `fstat`**: 获取文件属性及设备号 (用于关联磁盘设备)。
- **`socket` / `ioctl`**: 获取 IPv6 地址等特定网络配置。

---

## 3. 功能模块详解 (Module Details)

### 3.1 初始化与清理 (Initialization)

- **`sigar_os_open`**:
  - 分配 `sigar_t` 结构体。
  - 调用 `pstat_getstatic` 初始化静态信息 (`pstatic`)，包括物理内存大小、页大小、系统启动时间。
  - 获取系统时钟频率 (`_SC_CLK_TCK`) 用于后续时间转换。
- **`sigar_os_close`**:
  - 释放进程信息缓存 (`pinfo`)。
  - 关闭 MIB 设备句柄 (`close_mib`)。
  - 释放主结构体内存。

### 3.2 内存与交换空间 (Memory & Swap)

- **实现函数**: `sigar_mem_get`, `sigar_swap_get`
- **逻辑**:
  1.  **总内存**: `pstatic.physical_memory * page_size`。
  2.  **空闲内存**: 从 `pstat_getdynamic` 的 `psd_free` 字段获取页数，转换为字节。
  3.  **实际可用内存**: 空闲内存 + 内核动态内存 (`pstat_getvminfo.psv_kern_dynmem`)。HP-UX 内核会占用一部分内存用于动态结构，这部分在压力下可回收，因此算作“实际可用”。
  4.  **Swap**: 循环调用 `pstat_getswap` 累加所有交换设备的总量和空闲量。页面换入/换出速率来自 `pstat_getvminfo`。

### 3.3 CPU 统计 (CPU Statistics)

- **实现函数**: `sigar_cpu_get`, `sigar_cpu_list_get`, `sigar_cpu_info_list_get`
- **逻辑**:
  1.  **聚合 CPU**: 调用 `pstat_getdynamic` 获取所有 CPU 的时间片总和 (`psd_cpu_time`)。
  2.  **单核 CPU**: 循环调用 `pstat_getprocessor` (索引 0 到 `ncpu-1`) 获取每个核心的 `psp_cpu_time`。
  3.  **时间转换**: 使用宏 `SIGAR_TICK2MSEC(ticks)` 将内核时钟周期转换为毫秒。
  4.  **分类统计**: 将时间片映射为 User, System, Nice, Idle, Wait (IO Wait), IRQ。
      - _注意_: `CP_SSYS` 被合并到 System，`CP_SWAIT` 和 `CP_BLOCK` 被合并到 Wait。

### 3.4 进程管理 (Process Management)

这是最复杂的模块，包含列表获取和详细信息查询。

#### 3.4.1 进程列表 (`sigar_os_proc_list_get`)

- **分页获取**: 定义 `PROC_ELTS` (16) 为批次大小。
- **循环调用**: `pstat_getproc(proctab, ..., idx)`。
- **索引推进**: 利用返回的最后一个进程的 `pst_idx` 作为下一轮的起始索引 (`idx = proctab[num-1].pst_idx + 1`)，直到返回数量为 0。

#### 3.4.2 进程详情缓存策略 (`sigar_pstat_getproc`)

为了避免频繁的系统调用，实现了**短时缓存机制**：

- **缓存键**: `last_pid` (上次查询的 PID)。
- **过期时间**: `SIGAR_LAST_PROC_EXPIRE` (通常为几秒)。
- **逻辑**: 如果请求的 PID 与上次相同，且时间差小于过期阈值，直接复用 `sigar->pinfo` 结构体，跳过 `pstat_getproc` 系统调用。

#### 3.4.3 详细信息提取

- **内存 (`sigar_proc_mem_get`)**: 累加 `pst_vtsize` (文本), `pst_vdsize` (数据), `pst_vssize` (栈) 等虚拟内存区域，乘以页大小。 resident (常驻内存) 来自 `pst_rssize`。
- **状态 (`sigar_proc_state_get`)**: 映射 `pst_stat` 枚举值：
  - `PS_SLEEP` -> 'S'
  - `PS_RUN` -> 'R'
  - `PS_ZOMBIE` -> 'Z'
  - `PS_IDLE` -> 'D' (Uninterruptible Sleep)
- **命令行 (`sigar_os_proc_args_get`)**:
  - HP-UX 11i v2+: 使用 `pstat_getcommandline`。
  - 旧版本: 使用通用 `pstat(PSTAT_GETCOMMANDLINE, ...)`。
  - 解析空格分隔的参数字符串。
- **可执行文件路径 (`sigar_proc_exe_get`)**:
  - 利用 `pstat_getpathname` 和进程结构体中的文件 ID (`pst_fid_text`, `pst_fid_cdir`) 还原绝对路径。

### 3.5 网络统计 (Network Statistics)

全部基于 **SNMP MIB** 接口实现。

- **初始化**: `sigar_get_mib_info` 封装了 `open_mib("/dev/ip", ...)` 和 `get_mib_info` 调用。
- **接口流量 (`sigar_net_interface_stat_get`)**:
  - 查询 OID `ifInOctets`, `ifOutOctets`, `ifInErrors` 等。
  - 处理 HP-UX 特有的 `nmapi_phystat` 结构体。
- **连接列表 (`sigar_net_connection_walk`)**:
  - **TCP**: 查询 `ID_tcpConnTable`。遍历 `mib_tcpConnEnt`，映射 TCP 状态 (如 `TCESTABLISED` -> `ESTABLISHED`)。
  - **UDP**: 查询 `ID_udpLsnTable`。仅支持监听状态 (Server)。
  - **过滤**: 根据 `walker->flags` (SERVER/CLIENT) 过滤结果。
- **路由与 ARP**:
  - 路由: 查询 `ID_ipRouteTable`。
  - ARP: 查询 `ID_ipNetToMediaTable`。
- **TCP 全局统计 (`sigar_tcp_get`)**:
  - 使用查找表 `tcps_lu` 批量查询 `ID_tcpActiveOpens`, `ID_tcpInSegs` 等 OID，填充 `sigar_tcp_t` 结构体。

### 3.6 文件系统与磁盘 I/O (File System & Disk I/O)

- **文件系统列表 (`sigar_file_system_list_get`)**:
  - 读取 `/etc/mnttab` (HP-UX 的挂载表)。
  - 跳过 `swap` 类型。
  - 识别 `hfs` (本地磁盘) 和 `cdfs` (光驱)。
- **磁盘 I/O (`sigar_file_system_usage_get`)**:
  - **设备映射**: 使用 `stat()` 获取挂载点的设备号 (`st_rdev`)。维护一个 `fsdev` 缓存，将设备号映射到设备名称 (如 `/dev/dsk/c0t0d0`)。
  - **获取统计**: 调用 `pstat_getlv` (Logical Volume)，传入设备号，获取 `psl_rxfer` (读次数), `psl_wcount` (写字节数) 等。
  - _注意_: 此方法仅对 LVM 管理的逻辑卷有效。

---

## 4. 关键数据结构与类型转换

### 4.1 时间转换

HP-UX 内核时间单位为 "Ticks" (时钟周期)。

```c
#define SIGAR_TICK2MSEC(t) ((t) * 1000 / sigar->ticks)
```

其中 `sigar->ticks` 由 `sysconf(_SC_CLK_TCK)` 获取 (通常为 100 或 1024)。

### 4.2 进程状态映射

| HP-UX (`pst_stat`) | SIGAR (`state`) | 描述                                     |
| :----------------- | :-------------- | :--------------------------------------- |
| `PS_SLEEP`         | 'S'             | 休眠 (Interruptible Sleep)               |
| `PS_RUN`           | 'R'             | 运行或可运行 (Runnable)                  |
| `PS_STOP`          | 'T'             | 停止 (Stopped)                           |
| `PS_ZOMBIE`        | 'Z'             | 僵尸进程 (Zombie)                        |
| `PS_IDLE`          | 'D'             | 休眠 (Uninterruptible Sleep, usually IO) |

### 4.3 TCP 状态映射

| HP-UX MIB State | SIGAR State             |
| :-------------- | :---------------------- |
| `TCLISTEN`      | `SIGAR_TCP_LISTEN`      |
| `TCESTABLISED`  | `SIGAR_TCP_ESTABLISHED` |
| `TCFINWAIT1`    | `SIGAR_TCP_FIN_WAIT1`   |
| `TCTIMEWAIT`    | `SIGAR_TCP_TIME_WAIT`   |
| `TCCLOSED`      | `SIGAR_TCP_CLOSE`       |
| ...             | ...                     |

---

## 5. 版本兼容性与条件编译

代码通过大量的 `#ifdef` 处理不同 HP-UX 版本的差异：

1.  **命令行获取**:
    - **11i v2+**: 直接使用 `pstat_getcommandline()`。
    - **旧版**: 使用 `pstat(PSTAT_GETCOMMANDLINE, ...)` 联合体方式。
2.  **文件信息**:
    - **11.31+**: 移除了 `pstat_getfile`，必须使用 `pstat_getfile2`。
    - **11.11+**: 支持 `__pst_fid`，可获取详细路径；旧版返回 `ENOTIMPL`。
3.  **CPU 频率**:
    - **11.31+**: `pst_processor` 结构体包含 `psp_cpu_frequency` (Hz)。
    - **旧版**: 通过 `psp_iticksperclktick * ticks` 估算。
4.  **架构差异**:
    - **IA64 (Itanium)**: 禁用 `_lwp_info` (Light Weight Process)，因为 IA64 线程模型不同。
    - **PA-RISC**: 使用特定的 CPU 型号标识。

---

## 6. 性能优化策略

1.  **批量系统调用**:
    - 获取进程列表时，一次 `pstat_getprocs` 获取 16 个进程，减少内核态/用户态切换次数。
2.  **进程信息缓存**:
    - 针对同一 PID 的短时间重复查询，直接返回缓存数据，避免昂贵的 `pstat_getproc` 调用。
3.  **MIB 句柄复用**:
    - `sigar->mib` 文件描述符在 `sigar_os_open` 时打开，在所有网络查询中复用，直到 `sigar_os_close` 才关闭。
4.  **设备缓存**:
    - `fsdev` 缓存避免了在每次查询磁盘 I/O 时重新遍历文件系统列表和调用 `stat`。

---

## 7. 局限性与注意事项

1.  **权限要求**:
    - 部分 `pstat` 调用和 MIB 查询可能需要 **root** 权限才能获取其他用户的进程详情或完整的网络统计。非 root 用户可能收到 `EPERM` 错误。
2.  **磁盘 I/O 限制**:
    - `pstat_getlv` 仅适用于 **LVM (Logical Volume Manager)** 管理的设备。对于裸设备或非 LVM 文件系统，磁盘 I/O 统计可能不可用 (`SIGAR_ENOTIMPL` 或 0)。
3.  **进程参数长度**:
    - `pstat_getcommandline` 有内核限制 (通常 1024 字节)，超长命令行可能会被截断。
4.  **IPv6 支持**:
    - 代码中包含 IPv6 获取逻辑，但依赖 `SIOCGLIFADDR` ioctl，需确保内核和网络驱动支持。
5.  **线程 CPU**:
    - `sigar_thread_cpu_get` 在 IA64 架构上返回 `ENOTIMPL`，因为 HP-UX 在 Itanium 上的线程实现与 PA-RISC 不同。

---

## 8. 总结

该实现充分利用了 HP-UX 操作系统的特性，通过 `pstat` 和 MIB 接口构建了高效、稳定的系统监控底层。其核心优势在于**直接内核访问**带来的低开销和高精度，以及完善的**版本兼容处理**。对于需要在 HP-UX 平台上进行性能监控、自动化运维或资源管理的應用，此模块提供了坚实的数据基础。
