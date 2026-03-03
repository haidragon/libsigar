# FreeBSD 系统信息采集实现文档 (SIGAR)

## 1. 概述 (Overview)

本文档详细描述了 **SIGAR (System Information Gatherer And Reporter)** 库在 **FreeBSD** 操作系统上的底层实现机制。FreeBSD 以其先进的 UFS/ZFS 文件系统、强大的网络栈和高效的 `kqueue` 机制闻名。其 SIGAR 实现深度依赖于 **`sysctl`** 接口获取核心指标，利用 **`libkvm`** 处理交换空间，并通过 **`libgeom`** (或直接的 `ioctl`) 获取磁盘几何与统计信息。

与 Linux 的 `/proc` 文本解析不同，FreeBSD 的实现完全基于**二进制结构体**的直接读取，具有极高的性能和类型安全性。

### 1.1 核心设计目标

- **Sysctl 为核心**：利用 FreeBSD 强大的 MIB (Management Information Base) 树获取 CPU、内存、进程和网络统计。
- **Libgeom 集成**：使用 `libgeom` 库抽象底层磁盘设备，统一处理 UFS、ZFS 及 RAID (gmirror, graid) 的统计信息。
- **Jail 感知**：正确运行在 FreeBSD Jails (容器) 中，仅报告 Jail 可见的资源视图。
- **ZFS 原生支持**：针对 FreeBSD 广泛使用的 ZFS 文件系统，优化内存和磁盘统计逻辑。

---

## 2. 核心依赖与接口 (Core Dependencies)

代码主要依赖以下四类 FreeBSD 特有接口：

### 2.1 Sysctl 接口 (核心数据源)

位于 `<sys/sysctl.h>`。这是 FreeBSD 暴露内核状态的首选方式。
| MIB 路径 (`ctlname`) | 用途 | 对应 SIGAR 功能 |
| :--- | :--- | :--- |
| `CTL_HW`, `HW_MODEL` / `HW_MACHINE` | 硬件型号与架构 | 系统信息 |
| `CTL_HW`, `HW_NCPU` | CPU 核心数量 | CPU 拓扑 |
| `CTL_VM`, `VM_LOADAVG` | 负载平均值 (loadavg) | 系统负载 |
| `CTL_VM`, `VM_TOTAL` | 内存页统计 (活跃/非活跃/空闲) | 内存统计 |
| `CTL_KERN`, `KERN_PROC` | 进程信息 (`kinfo_proc`) | 进程列表、详情、参数 |
| `CTL_KERN`, `KERN_CPTIME` | 全局 CPU 时间片 | CPU 总使用率 |
| `CTL_KERN`, `KERN_CP_TIME` | 每核 CPU 时间片 (注意复数 CP) | CPU 核心列表 |
| `CTL_NET`, `PF_INET`, `INET` | TCP/UDP 统计与连接表 (PCBList) | 网络连接列表 |
| `CTL_NET`, `PF_ROUTE` | 路由表信息 | 路由与 ARP |

### 2.2 KVM (Kernel Virtual Memory) 库

位于 `<kvm.h>` 和 `libkvm`。

- **主要用途**: 在 FreeBSD 上，`sysctl` 不提供详细的 Swap 设备列表（如多个 swap 文件的分别使用情况）。因此，SIGAR 必须使用 `kvm_getswapinfo()` 来遍历所有 swap 分区。
- **权限**: 通常只需要读权限，但在某些严格配置下可能需要 `kmem` 组权限。

### 2.3 Libgeom 与 Disk IOCTLs

位于 `<geom/geom.h>` (`libgeom`) 和 `<sys/disk.h>`。

- **`libgeom`**: FreeBSD 的 GEOM 框架是存储栈的核心。使用 `geom_gettree()` 可以遍历所有 Provider (磁盘分区)，获取名称、大小和统计信息。这比直接解析 `/dev` 更可靠，能自动处理软 RAID 和加密卷。
- **`DIOCGSTATS`**: 如果不用 `libgeom`，可直接对 `/dev/ada0` 等设备文件进行 `ioctl` 获取 `struct devstat`。

### 2.4 Network IOCTLs & Sockets

位于 `<net/if.h>`, `<sys/sockio.h>`。

- **`SIOCGIFCONF`**: 获取接口列表。
- **`SIOCGIFADDR`**, **`SIOCGIFBRDADDR`**, **`SIOCGIFNETMASK`**: 获取 IP 配置。
- **`SIOCGIFDATA`** (或 `ifm_data` in `ifm_req`): 获取流量统计。FreeBSD 的 `if_data` 结构包含标准的 byte/packet/error 计数。
- **`PF_ROUTE` Socket**: 监听或查询路由套接字以获取路由表和 ARP 缓存。

---

## 3. 功能模块详解 (Module Details)

### 3.1 初始化与清理 (Initialization)

- **`sigar_os_open`**:
  - 初始化 KVM: `kvm_openfiles(NULL, "/dev/null", NULL, O_RDONLY, NULL)`。注意第二个参数通常为 `/dev/null`，因为我们需要的是 swap 信息而不是符号表解析。
  - 初始化 `libgeom`: 调用 `geom_gettree()` 构建设备树缓存 (可选，也可按需查询)。
- **`sigar_os_close`**:
  - `kvm_close()`.
  - `geom_deletetree()`.

### 3.2 内存统计 (Memory Statistics)

- **实现函数**: `sigar_mem_get`
- **逻辑**:
  1.  **UVM 统计**: 调用 `sysctlbyname("vm.total", ...)` 获取 `struct vmtotal`。
      - `t_rm`: 常驻内存页。
      - `t_vm`: 虚拟内存页。
      - `t_free`: 空闲页。
  2.  **详细页统计**: 调用 `sysctlbyname("vm.uvmexp", ...)` 获取 `struct uvmexp`。
      - 区分 `active`, `inactive`, `wire` (锁定页), `free`。
      - **计算公式**:
        - `Total = uvmexp.npages * pagesize`
        - `Free = uvmexp.free * pagesize`
        - `Used = Total - Free`
        - `Actual Free = Free + Inactive` (FreeBSD 的 inactive 页可快速重用)。
  3.  **Swap**:
      - 调用 `kvm_getswapinfo(kd, &swinfo, count, 0)`。
      - 遍历返回的 `kvm_swap` 数组，累加 `ksw_total` 和 `ksw_used` (单位通常是 KB 或 block，需根据头文件确认并转换为字节)。

### 3.3 CPU 统计 (CPU Statistics)

- **实现函数**: `sigar_cpu_get`, `sigar_cpu_list_get`
- **逻辑**:
  1.  **全局 CPU**: `sysctlbyname("kern.cptime", ...)` 获取 `long cp_time[CPUSTATES]`。
  2.  **多核 CPU**: `sysctlbyname("kern.cp_times", ...)` (注意复数 's')。
      - 返回一个长数组，每 `CPUSTATES` 个元素代表一个核心。
      - 索引计算: `core[i].user = array[i * CPUSTATES + CP_USER]`.
  3.  **状态分类**: FreeBSD 的 `CPUSTATES` 通常包括:
      - `CP_USER`, `CP_NICE`, `CP_SYS`, `CP_INTR` (中断), `CP_IDLE`。
      - 较新版本可能有 `CP_SOFTINTR`。
  4.  **频率**: `sysctlbyname("dev.cpu.0.freq", ...)` (如果 `cpufreq` 驱动加载)。

### 3.4 进程管理 (Process Management)

#### 3.4.1 进程列表 (`sigar_os_proc_list_get`)

- **Sysctl 方式**:
  - 调用 `sysctl(CTL_KERN, KERN_PROC, KERN_PROC_ALL, 0, ...)`。
  - 获取 `struct kinfo_proc` 数组。
  - **优势**: 原子快照，包含所有基本信息，无需遍历目录。

#### 3.4.2 进程详情

- **数据来源**: 直接从 `kinfo_proc` 数组中匹配 PID。
- **内存 (`sigar_proc_mem_get`)**:
  - `ki_rssize`: RSS (页数) -> 乘以 `pagesize`。
  - `ki_size`: 虚拟内存总大小 (bytes)。
  - `ki_tsize`, `ki_dsize`, `ki_ssize`: 各段大小。
- **时间 (`sigar_proc_time_get`)**:
  - `ki_utime`, `ki_stime`: 类型为 `struct timeval` (秒 + 微秒)。
  - 直接转换: `msec = tv_sec * 1000 + tv_usec / 1000`。精度高于 tick 计算。
- **状态 (`sigar_proc_state_get`)**:
  - `ki_stat`: 映射 `SRUN`, `SSLEEP`, `SZOMB`, `SSTOP`, `SIDL` 到 SIGAR 字符。
- **命令行与参数**:
  - `ki_comm`: 短命令名。
  - **完整参数**: 调用 `sysctl(CTL_KERN, KERN_PROC, KERN_PROC_ARGS, pid, ...)`。
    - 这是 FreeBSD 获取完整 argv 的标准方法，返回一个以 null 分隔的字符串块。
- **文件描述符**:
  - FreeBSD 的 `kinfo_proc` 包含 `ki_numfds`，直接提供打开的文件描述符数量，无需遍历 `/proc/<pid>/fd` (FreeBSD 没有 procfs 默认挂载或结构不同)。

### 3.5 网络统计 (Network Statistics)

- **接口列表与流量**:
  - `socket(PF_INET, SOCK_DGRAM, 0)` + `ioctl(SIOCGIFCONF)`。
  - `ioctl(SIOCGIFDATA)` 获取 `if_data` 结构。
  - **MAC 地址**: `ioctl(SIOCGIFLLADDR)` (Link Layer Address)。
- **连接列表 (`sigar_net_connection_walk`)**:
  - **FreeBSD 优势**: 提供了强大的 `sysctl` 接口直接获取 PCB (Protocol Control Block) 列表。
  - **TCP**: `sysctl(CTL_NET, PF_INET, IPPROTO_TCP, TCP_STATS, TCB_LIST, ...)` (具体 OID 可能随版本微调，通常是 `net.inet.tcp.pcblist`)。
    - 返回 `xtcpcb` 或类似结构体数组，包含本地/远程 IP、端口、状态、** owning PID** (`xt_socket.so_pcb` -> `inp_socket` -> `so_cred` -> `cr_pid`，或者直接在新版结构中提供)。
    - _注意_: 获取 PID 通常需要 root 权限。
  - **UDP**: 类似，查询 `net.inet.udp.pcblist`。
  - 这种方法比遍历 `/proc` 快得多且更准确。

### 3.6 文件系统与磁盘 I/O

- **文件系统列表**:
  - 调用 `getmntinfo(&mntbuf, MNT_WAIT)`。
  - 遍历 `struct statfs` (FreeBSD 使用 `statfs` 而非 `statvfs`，但字段类似)。
  - 识别类型: `ufs`, `zfs`, `nfs`, `tmpfs`, `cd9660`。
- **磁盘使用量**:
  - 从 `statfs` 的 `f_blocks`, `f_bfree`, `f_bavail`, `f_bsize` 计算。
- **磁盘 I/O**:
  - **Libgeom 方式 (推荐)**:
    - 调用 `geom_gettree()` 获取 XML 树或结构体树。
    - 遍历 `class[name="DISK"]` 下的 `provider`。
    - 读取 `stats` 节点: `bytes_read`, `bytes_written`, `operations_read`, `operations_write`。
    - 优点: 自动处理 `ada`, `da`, `vtbd` 等各种驱动命名，且支持 GEOM 层级的聚合统计。
  - **Devstat 方式**:
    - 使用 `devstat` 库 (`<devstat.h>`) 封装的 `getdevs()`, `computestat()` 等函数。这是 `libgeom` 底层的传统方式，依然有效。

---

## 4. 关键数据结构与类型转换

### 4.1 `kinfo_proc` 结构体

FreeBSD 的 `kinfo_proc` 非常庞大且随版本演变。

```c
struct kinfo_proc {
    // ...
    struct pstats ki_pstats;
    struct rusage ki_rusage;
    // ...
    segsz_t ki_rssize;      // Resident Set Size (pages)
    segsz_t ki_swrss;       // Swap RSS
    // ...
    char ki_stat;           // State
    char ki_tdflags;        // Thread flags
    // ...
    pid_t ki_pid;
    pid_t ki_ppid;
    uid_t ki_uid;
    // ...
};
```

_注意_: 必须使用 `sysctl` 动态获取大小，不要 `sizeof(kinfo_proc)`，因为用户态头文件和内核实际结构可能因版本不同而有填充差异。

### 4.2 时间单位

- **CPU**: Ticks (`kern.cptime`)。转换: `msec = ticks * 1000 / hz`。
- **进程**: `timeval` (微秒精度)。直接转换，无精度损失。

### 4.3 进程状态映射

| FreeBSD `ki_stat` | 常量 | SIGAR State  |
| :---------------- | :--- | :----------- |
| `SRUN`            | 'R'  | 'R'          |
| `SSLEEP`          | 'S'  | 'S'          |
| `SZOMB`           | 'Z'  | 'Z'          |
| `SSTOP`           | 'T'  | 'T'          |
| `SIDL`            | 'I'  | 'D' (或 'I') |
| `SWAIT`           | 'W'  | 'S' (Wait)   |

---

## 5. 版本兼容性与条件编译

1.  **FreeBSD 版本差异**:
    - **KERN_PROC_ARGS**: 在 FreeBSD 8.0+ 引入。旧版本需回退到读取 `ki_comm` 或使用 `kvm` 解析。
    - **Libgeom**: FreeBSD 7.0+ 引入。旧版本需直接使用 `devstat` 库或 `ioctl(DIOCGSTATS)`。
    - **TCP PCB List**: 获取 PID 的功能在不同版本中结构体定义不同。新版 (`freebsd-12+`) 直接在 `xtcpcb` 中包含 PID，旧版可能需要复杂的指针追踪或无法获取 PID。
2.  **ZFS 支持**:
    - FreeBSD 10+ 默认使用 ZFS 作为根文件系统。`statfs` 对 ZFS 的支持非常完善。
    - ARC (Adaptive Replacement Cache) 会占用大量 "Inactive" 内存。SIGAR 应正确识别这部分内存为 "Available"，否则会导致内存使用率虚高。
3.  **网络驱动**:
    - 从 `fxp`, `em` 到 `ixgbe`, `vtnet`，接口名称变化不影响 `SIOCGIFDATA`，因为它是通用的 ioctl。

---

## 6. 性能优化策略

1.  **单次 Sysctl 抓取进程**:
    - 一次性 `KERN_PROC_ALL` 获取所有进程，内存中查找，避免多次系统调用。
2.  **Libgeom 缓存**:
    - 磁盘拓扑结构不会频繁变化，启动时构建一次 geom 树，后续仅更新统计计数器。
3.  **PCB List 直接读取**:
    - 利用 `sysctl` 直接 dump TCP/UDP 表，避免了 Unix 传统上需要遍历所有进程 FD 的低效方法。这是 FreeBSD 实现的一大亮点。
4.  **Timeval 直接转换**:
    - 利用 `kinfo_proc` 中的微秒级时间戳，避免了 tick 转换的舍入误差和频率依赖。

---

## 7. 局限性与注意事项

1.  **Root 权限需求**:
    - 获取其他用户的进程完整命令行 (`KERN_PROC_ARGS`) 和网络连接的 PID (`pcblist`) 通常需要 root 权限。非 root 用户可能看到部分字段为空或受限。
2.  **KVM 依赖**:
    - Swap 统计依赖 `libkvm`。如果 `/dev/mem` 或 `/dev/kmem` 访问受限 (现代安全加固系统常见)，Swap 详细统计可能不可用，只能获取总量 (如果 `sysctl` 提供的话，但 FreeBSD 通常不提供详细 swap sysctl)。
3.  **GEOM 复杂性**:
    - 在复杂的 GEOM 配置 (如多层 mirror+encrypt) 下，统计数据的去重和聚合需要小心处理，避免重复计算底层物理磁盘的 IO。
4.  **Jail 隔离**:
    - 在 Jail 中，`kern.proc.all` 仅返回 Jail 内的进程。`network` 统计也仅限于 Jail 配置的 interface。这是预期行为，但监控工具需知晓当前是否在 Jail 中 (可通过 `prison` 相关 sysctl 检测)。

---

## 8. 总结

FreeBSD 版本的 SIGAR 实现充分利用了 **`sysctl`** 的二进制高效性、**`libkvm`** 的深度访问能力以及 **`libgeom`** 的现代存储抽象。特别是通过 `sysctl` 直接获取 **TCP/UDP 连接列表及其 PID** 的能力，使得 FreeBSD 版本在网络监控方面比许多其他 Unix 变体 (如 Solaris, NetBSD) 更加高效和强大。代码结构清晰，严格遵循 FreeBSD 的编程规范，能够完美适应从嵌入式路由器到大型 ZFS 存储服务器的各种场景，并对 FreeBSD Jails 提供了天然的支持。
