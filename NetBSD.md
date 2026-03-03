# NetBSD 系统信息采集实现文档 (SIGAR)

## 1. 概述 (Overview)

本文档详细描述了 **SIGAR (System Information Gatherer And Reporter)** 库在 **NetBSD** 操作系统上的底层实现机制。NetBSD 以其代码整洁、架构清晰和高度可移植性著称。其 SIGAR 实现主要依赖于 **`sysctl`** 接口获取系统和进程信息，利用 **`kvm` (Kernel Virtual Memory)** 库访问深层内核数据结构（如交换空间、部分网络统计），并通过 **`ioctl`** 处理网络接口配置。

与 Linux 的 `/proc` (文本为主) 和 Solaris 的 `/proc` (二进制映射) 不同，NetBSD 倾向于使用统一的 `sysctl` MIB (Management Information Base) 树来暴露内核状态，这使得接口更加规范和类型安全。

### 1.1 核心设计目标

- **Sysctl 优先**：充分利用 `sysctl(3)` 接口获取 CPU、内存、进程列表和内核参数，这是 NetBSD 最推荐的监控方式。
- **KVM 辅助**：对于 `sysctl` 未完全覆盖的领域（如详细的 Swap 设备列表、某些历史兼容的网络统计），使用 `libkvm` 读取内核内存映像。
- **架构无关性**：代码需严格遵循 NetBSD 的多架构支持特性（x86, ARM, MIPS, SPARC, RISC-V 等），避免硬编码结构体大小。
- **权限最小化**：尽可能使用非特权接口，仅在必要时（如读取其他用户进程详情）要求 root 权限。

---

## 2. 核心依赖与接口 (Core Dependencies)

代码主要依赖以下三类 NetBSD 特有接口：

### 2.1 Sysctl 接口 (主要数据源)

位于 `<sys/sysctl.h>`。通过 MIB 名称数组访问内核数据。
| MIB 路径 (`ctlname`) | 用途 | 对应 SIGAR 功能 |
| :--- | :--- | :--- |
| `CTL_HW`, `HW_MODEL` / `HW_MACHINE` | 获取硬件型号、架构 | 系统信息 (System Info) |
| `CTL_HW`, `HW_NCPU` | 获取 CPU 核心数 | CPU 拓扑 |
| `CTL_VM`, `VM_LOADAVG` | 获取系统负载平均值 | 系统负载 |
| `CTL_VM`, `VM_TOTAL` | 获取内存总体统计 (活跃/非活跃页) | 内存统计 |
| `CTL_KERN`, `KERN_PROC` | 遍历或查询进程信息 (`kinfo_proc`) | 进程列表、进程详情 |
| `CTL_KERN`, `KERN_CPTIME` | 获取全局 CPU 时间片 (User, Nice, Sys, Idle) | CPU 使用率 |
| `CTL_KERN`, `KERN_CPTIME2` | 获取每个 CPU 核心的时间片 | CPU 核心列表 |
| `CTL_VM`, `UVMEXP` | 获取 UVM (Unix Virtual Memory) 详细统计 | 页面错误、换页活动 |

### 2.2 KVM (Kernel Virtual Memory) 库

位于 `<kvm.h>` 和 `libkvm`。用于访问内核虚拟内存，通常在 `sysctl` 信息不足时使用。
| 函数名 | 用途 | 对应 SIGAR 功能 |
| :--- | :--- | :--- |
| `kvm_openfiles` | 打开内核内存映像 (通常读 `/dev/kmem` 或当前运行内核) | 初始化 KVM 会话 |
| `kvm_getswapinfo` | 获取交换分区详细信息 | Swap 使用情况 (多设备累加) |
| `kvm_close` | 关闭会话 | 清理资源 |
| **注意**: 现代 NetBSD 鼓励尽量用 `sysctl` 替代 `kvm`，因为 `kvm` 需要 root 权限且可能不稳定。SIGAR 通常仅在 `sysctl` 无法提供 Swap 详情时使用它。 |

### 2.3 Network IOCTLs & Route Sockets

位于 `<net/if.h>`, `<sys/sockio.h>`, `<net/route.h>`。

- **`SIOCGIFCONF`**: 获取网络接口列表。
- **`SIOCGIFADDR`**, **`SIOCGIFNETMASK`**, **`SIOCGIFFLAGS`**: 获取 IP、掩码、状态。
- **`SIOCGIFDATA`**: 获取接口流量统计 (字节、包、错误)。
- **Route Socket**: 创建 `PF_ROUTE` 套接字，发送 `RTM_GET` 消息获取路由表和 ARP 表。

---

## 3. 功能模块详解 (Module Details)

### 3.1 初始化与清理 (Initialization)

- **`sigar_os_open`**:
  - 初始化 KVM 句柄 (`kvm_openfiles(NULL, NULL, NULL, O_RDONLY, NULL)`)。如果失败，记录警告，后续 Swap 查询可能不可用。
  - 测试 `sysctl` 关键节点的可访问性。
- **`sigar_os_close`**:
  - 调用 `kvm_close()` 释放资源。
  - 释放动态分配的 `sysctl` 缓冲区。

### 3.2 内存统计 (Memory Statistics)

- **实现函数**: `sigar_mem_get`
- **逻辑**:
  1.  **UVM 统计**: 调用 `sysctlbyname("vm.uvmexp", ...)` 获取 `struct uvmexp`。
      - `uvmexp.npages`: 总页数。
      - `uvmexp.free`: 空闲页数。
      - `uvmexp.active`: 活跃页数。
      - `uvmexp.inactive`: 非活跃页数。
  2.  **计算**:
      - `Total = npages * pagesize`.
      - `Free = free * pagesize`.
      - `Used = Total - Free`.
      - `Actual Free`: 通常定义为 `free + inactive` (因为非活跃页可快速回收)。
  3.  **Swap**:
      - 首选: 尝试 `sysctlbyname("vm.swapinfo", ...)` (如果内核支持)。
      - 备选: 使用 `kvm_getswapinfo(kd, ...)` 遍历所有 swap 设备，累加 `ksw_total` 和 `ksw_used`。

### 3.3 CPU 统计 (CPU Statistics)

- **实现函数**: `sigar_cpu_get`, `sigar_cpu_list_get`
- **逻辑**:
  1.  **全局 CPU**:
      - 调用 `sysctlbyname("kern.cptime", ...)` 获取 `long cp_time[CPUSTATES]`。
      - 索引: `CP_USER`, `CP_NICE`, `CP_SYS`, `CP_IDLE`, `CP_INTR` (NetBSD 特有中断时间), `CP_SOFTINTR`。
  2.  **单核 CPU**:
      - 调用 `sysctlbyname("kern.cp_time2", ...)` 获取数组，每个元素包含一个核心的统计。
      - 或者遍历 `CTL_KERN`, `KERN_CPTIME2`, `<core_id>`。
  3.  **单位转换**: NetBSD 的 CPU 时间单位是 ticks (频率由 `sysconf(_SC_CLK_TCK)` 决定，通常 100Hz)。
      - `Milliseconds = ticks * 1000 / clk_tick`.
  4.  **频率**: 通过 `sysctlbyname("machdep.cpu_frequency", ...)` (如果架构支持，如 x86) 获取当前频率。

### 3.4 进程管理 (Process Management)

#### 3.4.1 进程列表 (`sigar_os_proc_list_get`)

- **Sysctl 方式**:
  - 调用 `sysctl(CTL_KERN, KERN_PROC, KERN_PROC_ALL, ...)`。
  - 传入 `NULL` 缓冲区先获取所需大小，然后分配内存再次调用。
  - 返回一个 `kinfo_proc` 结构体数组。
  - 遍历数组提取 `ki_pid`。
- **优势**: 原子操作，一次性获取所有进程快照，避免遍历 `/proc` 目录时进程状态变化的问题。

#### 3.4.2 进程详情

- **数据来源**: 直接从上一步获取的 `kinfo_proc` 数组中查找对应 PID 的结构体，无需再次打开文件。
- **内存 (`sigar_proc_mem_get`)**:
  - `ki_vsize`: 虚拟内存大小。
  - `ki_rssize`: 常驻内存集大小 (RSS)，单位通常是页，需乘以 `pagesize`。
  - `ki_tsize`, `ki_dsize`, `ki_ssize`: 文本、数据、栈段大小。
- **时间 (`sigar_proc_time_get`)**:
  - `ki_utime`, `ki_stime`: 用户态和内核态时间，类型为 `struct timeval` 或 `uint64_t` (微秒或 tick)，直接转换。
- **状态 (`sigar_proc_state_get`)**:
  - `ki_stat`: 整数枚举。
    - `SSTOP`: 停止
    - `SSLEEP`: 休眠
    - `SRUN`: 运行
    - `SIDL`: 中间状态
    - `SZOMB`: 僵尸
  - 映射到 SIGAR 字符 ('T', 'S', 'R', 'I', 'Z')。
- **命令行与参数**:
  - `ki_comm`: 命令名 (短)。
  - `ki_args`: 指向用户空间参数的指针。由于 `kinfo_proc` 是在内核空间拷贝的，直接解引用 `ki_args` 可能无效 (取决于 NetBSD 版本和 `sysctl` 实现)。
  - **修正**: 现代 NetBSD 的 `KERN_PROC_ARGS` 允许单独查询参数。
    - 调用 `sysctl(CTL_KERN, KERN_PROC_ARGS, pid, KERN_PROC_ARGV, ...)` 获取完整的参数字符串数组。
- **凭证**:
  - `ki_uid`, `ki_gid`, `ki_euid`, `ki_egid` 直接在 `kinfo_proc` 中提供。

### 3.5 网络统计 (Network Statistics)

- **接口列表与流量**:
  - 使用 `socket(PF_INET, SOCK_DGRAM, 0)`。
  - `ioctl(SIOCGIFCONF)` 获取接口列表。
  - `ioctl(SIOCGIFDATA)` 获取 `if_data` 结构，其中包含 `ifi_ipackets`, `ifi_opackets`, `ifi_ibytes`, `ifi_obytes`, `ifi_ierrors` 等。
- **连接列表 (`sigar_net_connection_walk`)**:
  - NetBSD 没有直接的 `/proc/net/tcp`。
  - **方法 A (推荐)**: 使用 `sysctl(CTL_NET, PF_INET, IPPROTO_TCP, TCP_STATS, ...)` 获取全局统计，但这不提供连接列表。
  - **方法 B (Fstat)**: 类似 Solaris，遍历 `/proc` (或使用 `kvm` 解析内核结构) 查找 socket。
  - **方法 C (NetBSD 特有)**: 较新版本的 NetBSD 可能支持通过 `sysctl` 获取 PCB (Protocol Control Block) 列表，但接口不统一。SIGAR 通常回退到解析 `netstat -n` 输出 (作为最后手段) 或使用 `kvm` 遍历内核的 `tcbinfo` 链表 (复杂且易受内核版本影响)。
  - _注_: 许多 Unix 监控工具在 NetBSD 上对连接列表的支持较弱，通常只关注接口流量。

### 3.6 文件系统与磁盘 I/O

- **文件系统列表**:
  - 调用 `getmntinfo(&mntbuf, MNT_WAIT)`。
  - 遍历 `struct statvfs` 数组。
  - 获取 `f_mntfromname` (设备), `f_mntonname` (挂载点), `f_fstypename` (类型: ffs, lfs, nfs, zfs)。
- **磁盘使用量**:
  - 直接从 `statvfs` 的 `f_blocks`, `f_bfree`, `f_bavail`, `f_frsize` 计算。
- **磁盘 I/O**:
  - **Sysctl**: 尝试 `sysctlbyname("hw.diskstats", ...)` (如果内核启用 `diskstat` 驱动)。这会返回所有磁盘的统计数组。
  - **KVM**: 如果没有 `hw.diskstats`，可能需要通过 `kvm` 读取内核的 `dk_ndrive` 和 `dk_wds` 等历史变量 (较老的方法)。
  - **Devprop**: 某些架构可通过设备属性获取。

---

## 4. 关键数据结构与类型转换

### 4.1 `kinfo_proc` 结构体

这是 NetBSD 进程信息的基石。

```c
struct kinfo_proc {
    // ...
    pid_t ki_pid;
    pid_t ki_ppid;
    uid_t ki_uid;
    // ...
    segsz_t ki_rssize;  // Resident Set Size (pages)
    vaddr_t ki_vsize;   // Virtual Size (bytes)
    // ...
    char ki_stat;       // Process state
    char ki_comm[MAXCOMLEN]; // Command name
    // ...
};
```

_注意_: 不同 NetBSD 版本中 `kinfo_proc` 的大小可能变化，使用 `sysctl` 时必须先查询大小。

### 4.2 时间单位

- **CPU Ticks**: `kern.cptime` 返回的是 ticks。
- **Process Time**: `kinfo_proc` 中的时间可能是 `struct timeval` (秒 + 微秒) 或 ticks，需仔细检查头文件定义。如果是 `timeval`，转换公式：`msec = tv_sec * 1000 + tv_usec / 1000`。

### 4.3 进程状态映射

| NetBSD `ki_stat` | 常量名   | SIGAR State  |
| :--------------- | :------- | :----------- |
| 'S'              | `SSLEEP` | 'S'          |
| 'R'              | `SRUN`   | 'R'          |
| 'Z'              | `SZOMB`  | 'Z'          |
| 'T'              | `SSTOP`  | 'T'          |
| 'D'              | `SIDL`   | 'D' (或 'S') |

---

## 5. 版本兼容性与条件编译

1.  **NetBSD 版本差异**:
    - **KERN_PROC_ARGS**: 在较旧的 NetBSD 版本中可能不可用或不支持 `KERN_PROC_ARGV`。代码需检测 `sysctl` 返回值，失败则仅返回命令名。
    - **hw.diskstats**: 这是一个较新的功能 (约 NetBSD 6.0+)。旧版本需回退到 `kvm` 或返回 `ENOTIMPL`。
    - **UVM vs VMM**: 早期 NetBSD 使用 `vmmeter`，后来迁移到 `uvm`。`sysctl` 名称已从 `vm.vmmeter` 变为 `vm.uvmexp`。代码应优先尝试 `uvmexp`。
2.  **架构差异**:
    - NetBSD 支持极多架构。避免假设 `int` 是 32 位或指针是 32 位。使用 `size_t`, `pid_t`, `uint64_t` 等标准类型。
    - 某些架构 (如 ARM) 可能没有 `machdep.cpu_frequency`，需处理 `ENOENT` 错误。

---

## 6. 性能优化策略

1.  **单次 Sysctl 抓取**:
    - 获取进程列表时，一次 `sysctl(KERN_PROC_ALL)` 调用获取全量数据，然后在用户空间过滤和解析，避免多次内核往返。
2.  **KVM 缓存**:
    - 如果必须使用 `kvm_getswapinfo`，保持 `kvm_t` 句柄打开状态，不要每次调用都 `kvm_open/close`。
3.  **避免 Fstat 遍历**:
    - 由于 NetBSD 获取连接列表开销大，除非明确请求，否则默认不执行全系统 FD 扫描。

---

## 7. 局限性与注意事项

1.  **连接列表困难**:
    - 这是 NetBSD (以及大多数 BSD 变体) 的弱点。没有简单、高效、无需 root 的 API 来获取所有 TCP/UDP 连接及其 owning PID。SIGAR 在此平台上可能无法提供完整的 `net_connection` 功能，或者需要 root 权限并使用 `kvm`。
2.  **KVM 权限**:
    - 读取 `/dev/kmem` (如果使用 kvm) 通常需要 `kmem` 组权限或 root。如果权限不足，Swap 详细统计或深层网络统计将不可用。
3.  **磁盘统计粒度**:
    - `hw.diskstats` 提供的是物理磁盘统计。对于 RAID 框架 (ccd, raidframe) 或 LVM (lvm)，可能需要额外的逻辑来聚合或映射。
4.  **ZFS 支持**:
    - NetBSD 的 ZFS 端口 (zfs-on-netbsd) 可能在 `statvfs` 或 `sysctl` 中报告的信息不如原生 FFS 完善，需测试验证。

---

## 8. 总结

NetBSD 版本的 SIGAR 实现充分利用了 **`sysctl`** 的规范性和 **`kinfo_proc`** 的高效性，提供了稳定、准确的系统和进程监控能力。其代码风格简洁，符合 NetBSD 的设计哲学。主要的挑战在于**网络连接枚举**的缺失和**磁盘 I/O 统计**在不同版本间的接口差异。通过灵活结合 `sysctl`、`kvm` 和 `ioctl`，该实现在保持高可移植性的同时，为 NetBSD 平台提供了企业级的监控数据支持。
