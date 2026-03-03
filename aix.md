# AIX 系统信息采集实现文档 (SIGAR)

## 1. 概述 (Overview)

本文档详细描述了 **SIGAR (System Information Gatherer And Reporter)** 库在 **IBM AIX** 操作系统上的底层实现机制。AIX 作为 IBM 基于 System V UNIX 的商业 Unix 系统，拥有独特的内核架构、对象数据库 (ODM) 和性能监控接口。其 SIGAR 实现主要依赖于 **`libperfstat`** 性能库、**`libodm`** 对象数据库管理器、**`getkerninfo`** 系统调用以及标准的 **`ioctl`** 接口。

与 Linux 的 `/proc` 或 BSD 的 `sysctl` 不同，AIX 倾向于提供专门的 C 库 (`libperfstat`) 来返回结构化的性能数据，这使得数据采集非常高效且类型安全。

### 1.1 核心设计目标

- **Libperfstat 优先**：充分利用 AIX 专用的 `libperfstat` 库获取 CPU、内存、磁盘、网络和文件系统统计，这是 AIX 上最标准、最高效的性能采集方式。
- **ODM 集成**：使用对象数据库管理器 (ODM) 查询硬件配置、设备属性和逻辑卷信息，弥补纯性能统计在配置信息上的不足。
- **LPAR 感知**：准确识别逻辑分区 (LPAR) 环境，区分物理资源与分配给当前分区的虚拟资源，支持动态逻辑分区 (DLPAR)。
- **JFS2 支持**：针对 AIX 默认的 JFS2 文件系统进行优化，正确处理其特有的统计信息。

---

## 2. 核心依赖与接口 (Core Dependencies)

代码主要依赖以下四类 AIX 特有接口：

### 2.1 Libperfstat (性能统计库)

位于 `<libperfstat.h>` 和 `libperfstat.a`。这是 AIX 性能监控的核心。
| 函数名 | 用途 | 对应 SIGAR 功能 |
| :--- | :--- | :--- |
| `perfstat_cpu_total` | 获取全局 CPU 统计 | CPU 总使用率、负载 |
| `perfstat_cpu` | 获取每个逻辑 CPU 的统计 | CPU 核心列表 |
| `perfstat_memory_total` | 获取内存总体统计 | 内存总量、空闲、分页 |
| `perfstat_partition_total` | 获取 LPAR 分区统计 | 虚拟化环境下的资源视图 |
| `perfstat_disk_total` / `perfstat_disk` | 获取磁盘 I/O 统计 | 磁盘读写速率、操作次数 |
| `perfstat_netinterface_total` / `..._netinterface` | 获取网络接口统计 | 网卡流量、错误包 |
| `perfstat_fs_total` / `..._filesystem` | 获取文件系统统计 | 挂载点使用量、I/O |

### 2.2 ODM (Object Database Manager)

位于 `<odm.h>` 和 `libodm.a`。AIX 使用 ODM 存储系统和设备配置信息。
| 函数/概念 | 用途 | 对应 SIGAR 功能 |
| :--- | :--- | :--- |
| `odm_open`, `odm_get_first`, `odm_get_next` | 遍历 ODM 类 (如 `PdAt`, `CuDv`) | 获取硬件型号、序列号、设备属性 |
| `odm_close` | 关闭 ODM 句柄 | 清理资源 |
| **关键类**: `PdAt` (属性), `CuDv` (定制设备) | 存储设备详细信息 | 磁盘型号、网卡 MAC 地址、CPU 类型 |

### 2.3 Getkerninfo 系统调用

位于 `<sys/getkerninfo.h>`。类似于 BSD 的 `sysctl`，用于获取内核进程和表信息。

- **`KINFO_PROC`**: 获取进程列表和详细信息 (`struct kinfo_proc`)。
- **`KINFO_SWAP`**: 获取交换空间信息 (作为 `perfstat` 的备选)。
- **`KINFO_ROUTE`**: 获取路由表信息。

### 2.4 Standard IOCTLs & System Calls

- **`ioctl`**: 用于网络接口配置 (`SIOCGIFCONF`, `SIOCGIFADDR` 等，定义在 `<sys/sockio.h>`).
- **`mntctl`**: AIX 特有的文件系统控制调用，用于获取挂载信息 (比读取 `/etc/mnt` 更可靠)。
- **`statfs` / `statvfs`**: 获取文件系统使用量。

---

## 3. 功能模块详解 (Module Details)

### 3.1 初始化与清理 (Initialization)

- **`sigar_os_open`**:
  - 初始化 ODM: `odm_open()` 常用类 (如 `PdAt`)。
  - 验证 `libperfstat` 版本兼容性。
  - 获取页面大小 (`getpagesize()`) 和时钟频率 (`sysconf(_SC_CLK_TCK)`).
- **`sigar_os_close`**:
  - 关闭所有 ODM 句柄 (`odm_close()`).
  - 释放动态分配的 `perfstat` 结构体数组。

### 3.2 内存统计 (Memory Statistics)

- **实现函数**: `sigar_mem_get`
- **逻辑**:
  1.  **调用**: `perfstat_memory_total(NULL, &meminfo, sizeof(meminfo), 1)`.
  2.  **关键字段** (`perfstat_memory_total_t`):
      - `real_total`: 物理内存总页数。
      - `real_free`: 空闲页数。
      - `real_system`: 内核占用页。
      - `pgsp_total`: 交换空间总页数。
      - `pgsp_free`: 交换空间空闲页数。
      - `pgsin`, `pgsout`: 页面换入/换出次数。
  3.  **计算**:
      - `Total = real_total * pagesize`.
      - `Free = real_free * pagesize`.
      - `Used = Total - Free`.
      - **实际可用**: AIX 内存管理复杂，通常 `Free + (部分 Cache)` 视为可用。
  4.  **LPAR 视角**: 如果在 LPAR 中运行，`real_total` 反映的是分配给该分区的内存，而非物理机总内存。

### 3.3 CPU 统计 (CPU Statistics)

- **实现函数**: `sigar_cpu_get`, `sigar_cpu_list_get`
- **逻辑**:
  1.  **全局 CPU**: `perfstat_cpu_total(NULL, &cputotal, sizeof(cputotal), 1)`.
      - 字段: `user`, `sys`, `idle`, `wait` (IO Wait), `decrementer` (中断), `softirq`。
      - 单位: ticks (需乘以 1000 / clk_tick 转毫秒)。
  2.  **单核 CPU**:
      - 先调用 `perfstat_cpu(NULL, NULL, sizeof(perfstat_cpu_t), 0)` 获取 CPU 数量。
      - 分配数组，再次调用 `perfstat_cpu(...)` 填充数据。
      - 遍历数组提取每个逻辑 CPU 的统计。
  3.  **LPAR 共享模式**:
      - 在共享处理器池 (Shared Processor Pool) 中，`idle` 时间可能包含 "steal time" (被其他分区抢占的时间)。AIX 较新版本的 `perfstat` 提供了 `steal` 字段，需单独统计。
  4.  **频率**: 通过 ODM 查询 `PdAt` 类中 CPU 设备的 `frequency` 属性，或使用 `perfstat_partition_total` 中的 `processing_unit_capacity` 估算。

### 3.4 进程管理 (Process Management)

#### 3.4.1 进程列表 (`sigar_os_proc_list_get`)

- **Getkerninfo 方式**:
  - 调用 `getkerninfo(KINFO_PROC, 0, &size, 0)` 获取所需缓冲区大小。
  - 分配内存，再次调用 `getkerninfo(KINFO_PROC, buffer, &size, 0)`。
  - 返回 `struct kinfo_proc` 数组。
  - 遍历提取 `ki_pid`。
- **优势**: 原子快照，包含所有基本信息。

#### 3.4.2 进程详情

- **数据来源**: 从 `kinfo_proc` 数组中匹配 PID。
- **内存 (`sigar_proc_mem_get`)**:
  - `ki_rssize`: RSS (页数) -> 乘以 `pagesize`。
  - `ki_size`: 虚拟内存大小。
  - AIX 的 `kinfo_proc` 还包含 `ki_vmdata`, `ki_vmstack` 等详细段大小。
- **时间 (`sigar_proc_time_get`)**:
  - `ki_utime`, `ki_stime`: 类型为 `timestruc_t` (秒 + 纳秒) 或 `timeval`。
  - 直接转换为毫秒，精度极高。
- **状态 (`sigar_proc_state_get`)**:
  - `ki_stat`: 映射 `SRUN`, `SSLEEP`, `SZOMB`, `SSTOP`, `SIDL`。
  - AIX 特有状态: `SWAIT` (等待事件), `SLOCK` (锁定)。
- **命令行与参数**:
  - `ki_comm`: 短命令名。
  - **完整参数**: AIX 的 `getkerninfo` 不直接返回完整 argv 数组。
    - **替代方案**: 需要读取 `/proc/<pid>/cmdline` (如果 procfs 挂载) 或使用 `readlink` 相关技巧。如果 procfs 未挂载，可能只能获取到 `ki_comm`。
    - _注_: 现代 AIX 通常默认挂载 `procfs` 在 `/proc`。可以直接 `open("/proc/<pid>/cmdline")` 读取 null 分隔的参数字符串。
- **凭证**:
  - `ki_uid`, `ki_gid`, `ki_euid`, `ki_egid` 直接在 `kinfo_proc` 中。

### 3.5 网络统计 (Network Statistics)

- **接口列表与流量**:
  - **Perfstat 方式**: `perfstat_netinterface(NULL, &netif, sizeof(netif), count)`.
    - 直接获取 `if_name`, `ibytes`, `obytes`, `ipackets`, `opackets`, `ierrors`, `oerrors`。
    - 这是最简单高效的方法。
  - **IOCTL 方式**: 作为备选，使用 `socket` + `SIOCGIFCONF` + `SIOCGIFDATA`。
- **连接列表 (`sigar_net_connection_walk`)**:
  - AIX 没有直接的 `sysctl` PCB 列表。
  - **方法 A**: 解析 `netstat -An` 输出 (慢，不推荐)。
  - **方法 B**: 遍历 `/proc/<pid>/fd` (如果 procfs 挂载)，检查 socket。
  - **方法 C**: 使用 `getkerninfo(KINFO_SOCKET, ...)` (如果支持) 获取 socket 表，然后关联到进程。这通常比较复杂且依赖具体 AIX 版本。
  - _现状_: 在 AIX 上获取完整的带 PID 的连接列表比较困难，SIGAR 可能仅能提供部分信息或需要 root 权限进行深入扫描。

### 3.6 文件系统与磁盘 I/O

- **文件系统列表**:
  - 调用 `mntctl(MCTL_QUERY, size, buffer)` 获取 `struct vmount` 数组。
  - 解析 `vmount` 结构获取设备名、挂载点、类型 (jfs, jfs2, nfs, cdrfs)。
  - 或者读取 `/etc/mnt` 文件 (格式固定)。
- **磁盘使用量**:
  - 调用 `statfs(mountpoint, &buf)` 或 `statvfs`。
  - 计算: `Total = buf.f_blocks * buf.f_bsize`, `Free = buf.f_bfree * buf.f_bsize`.
- **磁盘 I/O**:
  - **Perfstat 方式**: `perfstat_disk(NULL, &disk, sizeof(disk), count)`.
    - 获取 `disk_name` (如 `hdisk0`), `rbytes`, `wbytes`, `reads`, `writes`, `time`。
    - 这是获取 AIX 磁盘统计的标准方法，自动处理逻辑卷映射。
  - **逻辑卷关联**: `perfstat` 返回的是物理磁盘 (`hdisk`) 统计。如果需要逻辑卷 (`lv`) 统计，可能需要查询 ODM (`LV` 类) 进行映射，或者 `perfstat_filesystem` 提供了基于文件系统的 I/O 统计。

---

## 4. 关键数据结构与类型转换

### 4.1 `perfstat` 结构体

AIX 的 `perfstat` 结构体版本控制严格。

```c
typedef struct {
    // ...
    uint64_t user;
    uint64_t sys;
    uint64_t idle;
    uint64_t wait;
    // ...
} perfstat_cpu_total_t;
```

_注意_: 调用 `perfstat_*` 函数时，必须传入结构体大小 (`sizeof(struct)`)，以便库函数进行版本检查。

### 4.2 时间单位

- **CPU Ticks**: `perfstat` 返回的是 ticks。
  - 转换: `msec = ticks * 1000 / sysconf(_SC_CLK_TCK)`.
- **进程时间**: `kinfo_proc` 中通常是 `timestruc_t` (sec, nsec)。
  - 转换: `msec = sec * 1000 + nsec / 1000000`.

### 4.3 进程状态映射

| AIX `ki_stat` | 常量 | SIGAR State |
| :------------ | :--- | :---------- |
| `SRUN`        | 'R'  | 'R'         |
| `SSLEEP`      | 'S'  | 'S'         |
| `SZOMB`       | 'Z'  | 'Z'         |
| `SSTOP`       | 'T'  | 'T'         |
| `SIDL`        | 'I'  | 'D'         |
| `SWAIT`       | 'W'  | 'S'         |

---

## 5. 版本兼容性与条件编译

1.  **AIX 版本差异**:
    - **Libperfstat**: 在 AIX 5.1+ 引入并不断完善。AIX 4.x 及更早版本需回退到 `getkerninfo` 或直接读取 `/dev/kmem` (极不推荐)。
    - **Procfs**: AIX 5.2+ 默认挂载 `/proc`。旧版本可能需要手动挂载 (`mount -v proc /proc`) 才能获取命令行和 FD 信息。
    - **JFS2**: AIX 5.1+ 引入。`mntctl` 能正确识别 `jfs2` 类型。
2.  **LPAR 与 WPAR**:
    - **LPAR (Logical Partition)**: `perfstat` 自动返回分区视角的数据。
    - **WPAR (Workload Partition)**: 在 WPAR 中，某些全局统计 (如物理磁盘 IO) 可能不可见或受限，需使用 `perfstat_wpar` (如果可用) 或接受限制。
3.  **64 位支持**:
    - AIX 很早就全面支持 64 位。确保代码编译为 64 位 (`-q64` 或 `-maix64`) 以访问大内存和完整结构体。

---

## 6. 性能优化策略

1.  **Libperfstat 批量读取**:
    - `perfstat_cpu`, `perfstat_disk` 等函数支持一次性读取所有实例到数组中。避免循环调用单次查询。
2.  **ODM 缓存**:
    - ODM 查询相对较慢。启动时缓存设备属性 (如磁盘型号、网卡 MAC)，运行时仅刷新 `perfstat` 计数器。
3.  **Procfs 直接读取**:
    - 对于命令行和 FD，直接 `open/read` `/proc/<pid>/...` 比 `getkerninfo` 更灵活且开销可控。
4.  **避免 Netstat 解析**:
    - 坚决避免调用 `popen("netstat ...")`，坚持使用 `perfstat` 或 `getkerninfo` API。

---

## 7. 局限性与注意事项

1.  **网络连接枚举困难**:
    - 这是 AIX 的主要短板。缺乏类似 FreeBSD `sysctl net.inet.tcp.pcblist` 的简单接口。获取带 PID 的连接表通常需要 root 权限并遍历 `/proc` 或使用复杂的 `kdb`/`kvm` 技术，SIGAR 在此功能上可能受限。
2.  **ODM 权限**:
    - 读取某些 ODM 类可能需要 root 或特定组权限。
3.  **Procfs 依赖**:
    - 如果 `/proc` 未挂载，进程命令行、FD 计数、部分网络功能将不可用。代码需检测 `/proc` 存在性并提供降级方案。
4.  **动态重配置 (DLPAR)**:
    - 在 DLPAR 操作中 (动态增加/减少 CPU/内存)，`perfstat` 会立即反映新配置，但某些缓存的逻辑可能需要失效重置。
5.  **字符编码**:
    - AIX 默认可能是 ISO-8859-1 或其他本地编码，而非 UTF-8。处理进程名和路径时需注意 locale 设置。

---

## 8. 总结

AIX 版本的 SIGAR 实现深度集成了 **`libperfstat`** 和 **ODM**，充分利用了 AIX 企业级操作系统的结构化数据接口。其在 **CPU (特别是 LPAR 环境)**、**内存** 和 **磁盘 I/O** 统计方面表现卓越，数据精度高且开销低。主要的挑战在于 **网络连接列表的获取** 以及对 **`/proc` 文件系统** 的依赖。通过合理使用 `perfstat` 系列 API 和 `getkerninfo`，该实现在 AIX 平台上提供了稳定、专业的系统监控能力，完全适配 Power 架构和 AIX 独特的虚拟化环境。
