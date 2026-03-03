# Solaris 系统信息采集实现文档 (SIGAR)

## 1. 概述 (Overview)

本文档详细描述了 **SIGAR (System Information Gatherer And Reporter)** 库在 **Oracle Solaris** (及 Sun Solaris) 操作系统上的底层实现机制。Solaris 版本的设计哲学与 Linux 和 Windows 截然不同，它深度依赖于 **`/proc` 文件系统** (procfs) 进行进程和系统状态查询，并利用 **Kstat (Kernel Statistics)** 接口获取高性能的设备驱动统计信息（CPU、网络、磁盘）。此外，对于网络路由和接口配置，它广泛使用 **`ioctl`** 系统调用。

### 1.1 核心设计目标

- **Procfs 优先**：利用 Solaris 强大的 `/proc` 文件系统，以文件 I/O 的方式直接读取内核数据结构，避免复杂的系统调用封装。
- **Kstat 高效采集**：使用 `libkstat` 获取高精度的 CPU、网络和磁盘 I/O 计数器，这是 Solaris 性能监控的标准方式。
- **区域 (Zone) 感知**：能够正确识别并运行在 Solaris Zones (容器) 中，仅报告当前区域可见的资源。
- **多线程安全**：适配 Solaris 的 LWP (Light Weight Process) 模型，准确统计线程级资源。

---

## 2. 核心依赖与接口 (Core Dependencies)

代码主要依赖以下三类 Solaris 特有接口：

### 2.1 `/proc` 文件系统 (进程与核心状态)

位于 `<proc.h>`, `<sys/proc.h>`, `<sys/psw.h>`。Solaris 的 `/proc` 不仅仅是文本信息，更是二进制结构体的映射。
| 文件/路径 | 用途 | 对应 SIGAR 功能 |
| :--- | :--- | :--- |
| `/proc` (目录) | 包含以 PID 命名的子目录 | 进程列表遍历 |
| `/proc/<pid>/psinfo` | `psinfo_t` 结构体 | 进程基本信息 (PID, PPID, 状态, 命令行, 参数) |
| `/proc/<pid>/status` | `prstatus_t` 结构体 | 进程运行时状态 (CPU 时间, 信号, 寄存器) |
| `/proc/<pid>/usage` | `prusage_t` 结构体 | 详细的资源使用 (字符 I/O, 系统/用户 CPU, 上下文切换) |
| `/proc/<pid>/map` | 内存映射列表 | 进程虚拟内存布局 (Size, Resident, Shared) |
| `/proc/<pid>/cred` | 凭证信息 | 进程 UID, GID, EUID, EGID |
| `/proc/<pid>/fd` | 文件描述符目录 | 统计打开的文件句柄数量 |
| `/proc/system_info` | 系统全局信息 | 启动时间, 主机名, 架构 |

### 2.2 Kstat (Kernel Statistics) API (性能计数器)

位于 `<kstat.h>` 和 `libkstat` 库。这是 Solaris 内核暴露统计数据的标准接口。
| 函数名 | 用途 | 对应 SIGAR 功能 |
| :--- | :--- | :--- |
| `kstat_open` | 打开 kstat 控制句柄 | 初始化性能采集会话 |
| `kstat_lookup` | 查找特定的 kstat 条目 (按模块、实例、名称) | 定位特定 CPU、网卡或磁盘的统计块 |
| `kstat_read` | 读取最新的统计数据到缓冲区 | 刷新 CPU 时间、网络字节计数、磁盘 IO |
| `kstat_close` | 关闭句柄 | 清理资源 |
| **关键数据类型** | `kstat_named_t` (键值对), `kstat_io_t` (IO 统计) | 解析具体数值 |

### 2.3 Socket IOCTLs (网络配置)

位于 `<sys/sockio.h>`, `<net/if.h>`。

- **`SIOCGLIFCONF`**: 获取逻辑接口列表 (支持 IPv6 和多实例)。
- **`SIOCGLIFADDR`**, **`SIOCGLIFNETMASK`**: 获取 IP 地址和掩码。
- **`SIOCGLIFFLAGS`**: 获取接口状态 (UP, RUNNING, LOOPBACK)。
- **`SIOCGLIFHWADDR`**: 获取 MAC 地址。
- **`route` 套接字**: 读取路由表 (`RTM_GET`)。

---

## 3. 功能模块详解 (Module Details)

### 3.1 初始化与清理 (Initialization)

- **`sigar_os_open`**:
  - 调用 `kstat_open()` 初始化 Kstat 句柄，用于后续所有性能数据读取。
  - 扫描 `/proc` 目录以验证访问权限。
  - 获取系统页面大小 (`sysconf(_SC_PAGESIZE)`) 用于内存计算。
- **`sigar_os_close`**:
  - 调用 `kstat_close()` 释放 Kstat 资源。
  - 关闭所有打开的 `/proc` 文件描述符。

### 3.2 内存统计 (Memory Statistics)

- **实现函数**: `sigar_mem_get`
- **逻辑**:
  1.  **Kstat 来源**: 查找 kstat 模块 `unix`，名称 `system_pages`。
      - 读取 `pagestotal` (总页数), `pagesfree` (空闲页数), `pageslocked` (锁定页)。
      - 计算公式: `Total = pagestotal * pagesize`, `Free = pagesfree * pagesize`.
  2.  **Swap 统计**: 查找 kstat 模块 `swap`，名称 `swapinfo` (或通过 `pstat_getswap` 的 Solaris 等价逻辑，通常直接读 `/proc` 或调用 `swapctl`)。
      - SIGAR 通常解析 `kstat` 中的 `avail` 和 `total` 字段。
  3.  **ZFS ARC (可选)**: 如果系统使用 ZFS，可能会额外读取 `zfs` 模块的 `arcstats` 来区分 "实际可用内存" (因为 ZFS ARC 会占用大量空闲内存但可随时释放)。

### 3.3 CPU 统计 (CPU Statistics)

- **实现函数**: `sigar_cpu_get`, `sigar_cpu_list_get`
- **逻辑**:
  1.  **Kstat 查询**:
      - 遍历 CPU 实例 (0 到 N-1)。
      - 查找模块 `cpu_info` (获取频率、型号) 和 `sys` (获取时间统计)。
      - 更常见的是查找模块 `unix` 或直接通过 `kstat_lookup(cpustat, "cpu", instance, "sys")`。
  2.  **时间字段映射**:
      - Solaris `kstat` 提供 `cpu_ticks[CPU_STATE_USER]`, `CPU_STATE_KERNEL`, `CPU_STATE_IDLE`, `CPU_STATE_WAIT` (IO Wait)。
      - 直接累加这些 tick 值。
  3.  **单位转换**: Solaris 的 tick 频率由 `sysconf(_SC_CLK_TCK)` 决定 (通常 100Hz)。
      - `Milliseconds = ticks * 1000 / clk_tick`.
  4.  **负载平均**: 读取 `/proc/loadavg` 或使用 `getloadavg()`。

### 3.4 进程管理 (Process Management)

#### 3.4.1 进程列表 (`sigar_os_proc_list_get`)

- **目录遍历**: 打开 `/proc` 目录 (`opendir("/proc")`)。
- **过滤**: 读取每个条目 (`readdir`)，检查是否为数字 (PID)。
- **优势**: 这种方法在 Solaris 上非常高效，且天然支持 Zone 隔离 (进程只能看到本 Zone 的 `/proc` 条目)。

#### 3.4.2 进程详情

- **打开文件**: 构造路径 `/proc/<pid>/psinfo` 和 `/proc/<pid>/status`，使用 `open()` 打开。
- **读取结构体**: 直接使用 `read()` 将数据读入 `psinfo_t` 和 `prstatus_t` 结构体。无需解析文本，效率极高。
- **内存 (`sigar_proc_mem_get`)**:
  - 解析 `prmap_t` (通过 `/proc/<pid>/map`)：遍历所有内存段，累加 `pr_map_size` (Size) 和 `pr_rss` (Resident)。
  - 或者直接从 `prusage_t` (如果版本支持) 读取 `pr_vsize` 和 `pr_rssize`。
- **时间 (`sigar_proc_time_get`)**:
  - 从 `prstatus_t` 读取 `pr_utime` (User) 和 `pr_stime` (System)，类型为 `timestruc_t` (sec, nsec)。
  - 直接转换为毫秒，无需 tick 转换，精度更高。
- **状态 (`sigar_proc_state_get`)**:
  - 读取 `prstatus_t.pr_sname` (单字符状态):
    - 'S': Sleep
    - 'R': Run
    - 'Z': Zombie
    - 'T': Stop
    - 'I': Idle
- **命令行与参数**:
  - `psinfo_t.pr_psargs`: 包含完整的命令行参数字符串 (固定长度，通常 80 或 512 字节)。
  - `psinfo_t.pr_fname`: 可执行文件名。
- **凭证 (`sigar_proc_cred_get`)**:
  - 读取 `/proc/<pid>/cred` 文件，获取 `uid`, `gid`, `euid`, `egid`。

### 3.5 网络统计 (Network Statistics)

- **接口列表与配置**:
  - 创建 `AF_INET` (或 `AF_INET6`) 套接字。
  - 调用 `ioctl(sock, SIOCGLIFCONF, ...)` 获取 `lifconf_t` 结构，其中包含 `lifreq_t` 数组。
  - 遍历数组，对每个接口调用 `SIOCGLIFADDR`, `SIOCGLIFNETMASK`, `SIOCGLIFFLAGS`。
- **流量统计**:
  - **Kstat 方式 (推荐)**: 查找 kstat 模块 `mac`, `eri`, `e1000g`, `bge` 等 (取决于网卡驱动)，实例为接口索引，名称 `obk` 或直接找 `rbytes`/`obytes`。
  - **通用方式**: 查找 kstat 模块 `link`，名称 `<interface_name>`。读取 `ifspeed`, `rbytes`, `obytes`, `ipackets`, `opackets`, `ierrors`, `oerrors`。
- **连接列表 (`sigar_net_connection_walk`)**:
  - Solaris 没有直接的 `GetExtendedTcpTable` 等价物。
  - **方法 A**: 解析 `netstat` 输出 (不推荐，慢且不稳定)。
  - **方法 B (SIGAR 常用)**: 遍历 `/proc` 中的所有进程，打开 `/proc/<pid>/fd`，检查每个文件描述符是否为 socket (`fstat` + `S_ISSOCK`)。如果是，使用 `ioctl(..., SIOCGETSOCKNAME, ...)` 和 `SIOCGETPEERNAME` 获取本地/远程地址。
    - _缺点_: 速度慢，需要遍历所有进程 FD。
  - **方法 C (Kstat/Devinfo)**: 较新的 Solaris 版本可能通过 `kstat` 模块 `tcp` 或 `udp` 暴露连接表，但结构复杂。SIGAR 通常采用方法 B 或调用私有库。

### 3.6 文件系统与磁盘 I/O (File System & Disk I/O)

- **文件系统列表**:
  - 读取 `/etc/mnttab` (Solaris 的挂载表，类似 Linux `/etc/mtab` 但更稳定)。
  - 解析每一行获取 `mnt_special` (设备), `mnt_mountp` (挂载点), `mnt_fstype` (类型: ufs, zfs, nfs)。
- **磁盘使用量**:
  - 调用 `statvfs(mountpoint, &buf)`。
  - 计算: `Total = buf.f_blocks * buf.f_frsize`, `Free = buf.f_bfree * buf.f_frsize`.
- **磁盘 I/O**:
  - **Kstat 方式**: 查找 kstat 模块 `sd` (SCSI Disk), `ssd` (Solid State), 或 `lofi`。
  - 实例对应磁盘编号 (0, 1, ...)。
  - 读取 `kstat_io_t` 结构体:
    - `nread`, `nwritten` (字节数).
    - `reads`, `writes` (操作次数).
    - `wtime` (等待时间).

---

## 4. 关键数据结构与类型转换

### 4.1 时间处理

Solaris 混合使用 `clock_t` (ticks) 和 `timestruc_t` (sec/nsec)。

- **Procfs 时间**: `prstatus_t` 中的时间是 `timestruc_t`，精度高，直接转换：
  ```c
  msec = (ts.tv_sec * 1000) + (ts.tv_nsec / 1000000);
  ```
- **Kstat 时间**: 通常是 `uint64_t` ticks，需除以 `kstat_header->kr_hz`。

### 4.2 进程状态映射

| Solaris `pr_sname` | SIGAR State | 描述                              |
| :----------------- | :---------- | :-------------------------------- |
| 'S'                | 'S'         | Sleep (等待事件)                  |
| 'R'                | 'R'         | Run (运行或就绪)                  |
| 'Z'                | 'Z'         | Zombie (僵尸)                     |
| 'T'                | 'T'         | Stopped (被信号停止)              |
| 'I'                | 'D'         | Idle (深层睡眠，通常用于内核线程) |

### 4.3 区域 (Zone) 感知

- SIGAR 在 Solaris 上运行时，`/proc` 和 `kstat` 会自动受到 Zone 的限制。
- **全局区 (Global Zone)**: 可以看到所有进程和设备。
- **非全局区 (Non-Global Zone)**: `/proc` 仅包含本区进程，`kstat` 仅显示分配给该区的虚拟资源。代码无需特殊逻辑，内核已处理隔离。

---

## 5. 版本兼容性与条件编译

1.  **Solaris 9 vs 10/11**:
    - **LIF (Large Interface)**: Solaris 10+ 推荐使用 `SIOCGLIF...` (支持 IPv6 和长名称)，旧版使用 `SIOCGIF...`。代码需检测 `AF_INET6` 支持情况。
    - **Procfs 增强**: Solaris 10 引入了更丰富的 `/proc/<pid>/usage` 和 `/proc/<pid>/cred` 字段。旧版可能需要从 `psinfo` 推断或通过其他方式获取。
2.  **SPARC vs x86/x64**:
    - 结构体对齐可能不同。代码应使用标准类型 (`uint32_t`, `uint64_t`) 并避免硬编码偏移量。
    - Kstat 模块名称可能略有差异 (如网卡驱动名不同)，遍历时需模糊匹配。
3.  **ZFS 支持**:
    - 在 Solaris 10+ 上，ZFS 是根文件系统。`statvfs` 行为与传统 UFS 略有不同 (特别是 `f_files` 在 ZFS 上无意义)，代码需处理 `f_files == 0` 的情况。

---

## 6. 性能优化策略

1.  **直接结构体读取**:
    - 读取 `/proc/<pid>/psinfo` 时，直接 `read()` 到结构体，零解析开销。这是 Solaris 实现比 Linux (需解析文本) 快的主要原因。
2.  **Kstat 句柄缓存**:
    - `kstat_open()` 开销较大，SIGAR 在初始化时打开一次，全程复用。
    - 对于高频调用的计数器 (如 CPU)，缓存 `kstat_t` 指针，避免每次调用都 `kstat_lookup`。
3.  **最小化 FD 打开**:
    - 遍历进程时，仅在需要详细信息时才打开 `/proc/<pid>/...`。列表获取仅依赖 `readdir`。
4.  **批量网络查询**:
    - 使用 `SIOCGLIFCONF` 一次性获取所有接口信息，而不是循环猜测接口名 (如 hme0, hme1...)。

---

## 7. 局限性与注意事项

1.  **权限要求**:
    - 读取 `/proc/<pid>/cred` 或某些进程的 `fd` 可能需要 root 权限或相同 UID。
    - 非全局区用户无法访问全局区的 Kstat 数据。
2.  **网络连接获取困难**:
    - 由于缺乏类似 `/proc/net/tcp` 的简单文本接口，获取全系统 TCP/UDP 连接表在 Solaris 上比较繁琐 (需遍历进程 FD 或使用未公开 ioctl)，性能较差。
3.  **Kstat 命名不一致**:
    - 不同网卡驱动 (e1000g, ixgbe, bnx) 的 kstat 模块名不同。代码需要维护一个驱动名列表或进行通配符查找才能统计所有网卡的流量。
4.  **文件描述符限制**:
    - 遍历所有进程的所有 FD 来获取网络连接时，可能会耗尽文件描述符限制 (`RLIMIT_NOFILE`)。需注意 `ulimit` 设置。
5.  **Zone 资源视图**:
    - 在非全局区，物理内存和 CPU 的统计反映的是“视图”而非物理硬件总量。这对于监控容器资源使用是正确的，但用户需理解其含义。

---

## 8. 总结

Solaris 版本的 SIGAR 实现充分利用了 **`/proc` 文件系统的二进制接口** 和 **Kstat 框架**，实现了极高效率和精度的系统监控。其核心优势在于**结构化数据访问** (无需文本解析) 和**内核统计数据的直接暴露**。尽管在网络连接枚举方面略显复杂，但凭借对 Procfs 和 Kstat 的深度整合，该模块成为了 Solaris 平台上最可靠的系统信息采集方案之一，完美适配 Solaris 的 Zone 虚拟化架构和多线程模型。
