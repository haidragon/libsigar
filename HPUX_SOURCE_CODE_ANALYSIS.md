# HPUX 分支源码分析

## 概述

HPUX (HP-UX) 是惠普公司开发的专有 UNIX 操作系统,运行在 HP PA-RISC 和 Itanium 架构的服务器上。libsigar 在 HPUX 平台的实现主要通过 `pstat` 系列系统调用获取系统信息,同时结合 MIB (Management Information Base) 接口获取网络统计数据。

## 核心架构

### 1. sigar_t 结构体扩展

```c
struct sigar_t {
    SIGAR_T_BASE;              // 基础结构
    struct pst_static pstatic; // 静态系统信息 (不随系统运行变化)
    time_t last_getprocs;      // 最后一次获取进程信息的时间
    sigar_pid_t last_pid;      // 最后查询的 PID
    struct pst_status *pinfo;  // 进程状态信息缓存
    int mib;                   // MIB 文件描述符
};
```

### 2. 核心数据结构

#### pst_static - 静态系统信息
在系统初始化时一次性获取,包含:
- `physical_memory`: 物理内存页数
- `page_size`: 页面大小
- `boot_time`: 系统启动时间

#### pst_dynamic - 动态系统信息
- `psd_free`: 可用内存页数
- `psd_proc_cnt`: CPU 数量
- `psd_cpu_time[]`: CPU 时间统计数组
- `psd_avg_1_min/5_min/15_min`: 负载平均值

#### pst_status - 进程状态信息
- `pst_pid`: 进程 ID
- `pst_ppid`: 父进程 ID
- `pst_uid/pst_euid`: 用户 ID
- `pst_gid/pst_egid`: 组 ID
- `pst_utime/pst_stime`: CPU 使用时间
- `pst_start`: 启动时间
- `pst_nlwps`: 线程数
- `pst_stat`: 进程状态

## 核心技术

### 1. pstat 系统调用系列

HPUX 提供了一系列 `pstat` 函数用于获取系统统计信息:

| 函数 | 用途 | 数据结构 |
|------|------|----------|
| `pstat_getstatic` | 静态系统信息 | `pst_static` |
| `pstat_getdynamic` | 动态系统信息 | `pst_dynamic` |
| `pstat_getprocessor` | CPU 信息 | `pst_processor` |
| `pstat_getproc` | 进程信息 | `pst_status` |
| `pstat_getvminfo` | 虚拟内存信息 | `pst_vminfo` |
| `pstat_getswap` | 交换空间信息 | `pst_swapinfo` |
| `pstat_getfile2` | 打开的文件描述符 | `pst_fileinfo2` |
| `pstat_getpathname` | 文件路径 | `char[]` |
| `pstat_getlv` | 逻辑卷信息 | `pst_lvinfo` |

### 2. MIB (Management Information Base) 接口

用于获取网络统计信息,通过 `/dev/ip` 设备访问:

```c
int mib = open_mib("/dev/ip", O_RDONLY, 0, 0);

// 获取网络接口数量
int count;
struct nmparms parms;
parms.objid = ID_ifNumber;
parms.buffer = &count;
parms.len = &len;
get_mib_info(mib, &parms);

// 获取路由表
mib_ipRouteEnt *routes;
parms.objid = ID_ipRouteTable;
parms.buffer = routes;
parms.len = &len;
```

### 3. 架构兼容性处理

#### PA-RISC vs Itanium

```c
#ifdef _PSTAT64
typedef int64_t pstat_int_t;  // 64 位 Itanium
#else
typedef int32_t pstat_int_t;  // 32 位 PA-RISC
#endif

#ifdef __ia64__
    _lwp_info(&info);  // LWP 信息 (仅 PA-RISC)
#else
    return SIGAR_ENOTIMPL;  // Itanium 不支持 LWP
#endif
```

#### CPU 时间映射

HPUX 将 CPU 时间分为多个类别:
```c
cpu->user  = cpu_time[CP_USER];
cpu->sys   = cpu_time[CP_SYS] + cpu_time[CP_SSYS];
cpu->nice  = cpu_time[CP_NICE];
cpu->idle  = cpu_time[CP_IDLE];
cpu->wait  = cpu_time[CP_SWAIT] + cpu_time[CP_BLOCK];
cpu->irq   = cpu_time[CP_INTR];
```

### 4. 进程信息缓存

通过 60 秒缓存减少 `pstat_getproc` 调用:

```c
static int sigar_pstat_getproc(sigar_t *sigar, sigar_pid_t pid)
{
    time_t timenow = time(NULL);

    if (sigar->last_pid == pid) {
        if ((timenow - sigar->last_getprocs) < SIGAR_LAST_PROC_EXPIRE) {
            return SIGAR_OK;  // 使用缓存
        }
    }

    sigar->last_pid = pid;
    sigar->last_getprocs = timenow;

    return pstat_getproc(sigar->pinfo, sizeof(*sigar->pinfo), 0, pid);
}
```

## 核心功能模块

### 1. 内存监控

```c
int sigar_mem_get(sigar_t *sigar, sigar_mem_t *mem)
{
    struct pst_dynamic stats;
    struct pst_vminfo vminfo;

    // 物理内存总量
    mem->total = sigar->pstatic.physical_memory * pagesize;

    // 动态内存统计
    pstat_getdynamic(&stats, sizeof(stats), 1, 0);
    mem->free = stats.psd_free * pagesize;
    mem->used = mem->total - mem->free;

    // 内核动态内存
    pstat_getvminfo(&vminfo, sizeof(vminfo), 1, 0);
    kern = vminfo.psv_kern_dynmem * pagesize;
    mem->actual_free = mem->free + kern;
    mem->actual_used = mem->used - kern;
}
```

### 2. 交换空间监控

```c
int sigar_swap_get(sigar_t *sigar, sigar_swap_t *swap)
{
    struct pst_swapinfo swapinfo;
    int i = 0;

    // 遍历所有交换设备
    while (pstat_getswap(&swapinfo, sizeof(swapinfo), 1, i++) > 0) {
        // 转换为字节 (nfpgs 是 512 字节块)
        swapinfo.pss_nfpgs *= 4;

        // 累计交换空间
        swap->total += swapinfo.pss_nblksenabled;
        swap->free  += swapinfo.pss_nfpgs;
    }

    // 页换入/换出统计
    pstat_getvminfo(&vminfo, sizeof(vminfo), 1, 0);
    swap->page_in = vminfo.psv_spgin;
    swap->page_out = vminfo.psv_spgout;
}
```

### 3. CPU 监控

#### 总体 CPU 统计

```c
int sigar_cpu_get(sigar_t *sigar, sigar_cpu_t *cpu)
{
    struct pst_dynamic stats;

    pstat_getdynamic(&stats, sizeof(stats), 1, 0);
    sigar->ncpu = stats.psd_proc_cnt;

    get_cpu_metrics(sigar, cpu, stats.psd_cpu_time);
}
```

#### 每个 CPU 统计

```c
int sigar_cpu_list_get(sigar_t *sigar, sigar_cpu_list_t *cpulist)
{
    for (i = 0; i < sigar->ncpu; i++) {
        struct pst_processor proc;

        if (pstat_getprocessor(&proc, sizeof(proc), 1, i) < 0) {
            continue;
        }

        get_cpu_metrics(sigar, cpu, proc.psp_cpu_time);
    }
}
```

#### CPU 信息

```c
int sigar_cpu_info_list_get(sigar_t *sigar, sigar_cpu_info_list_t *cpu_infos)
{
    struct pst_processor proc;

    // CPU 频率计算 (HPUX 11.31+)
    #ifdef PSP_MAX_CACHE_LEVELS
        info->mhz = proc.psp_cpu_frequency / 1000000;
    #else
        info->mhz = ticks * proc.psp_iticksperclktick / 1000000;
    #endif

    // 架构标识
    #ifdef __ia64__
        SIGAR_SSTRCPY(info->vendor, "Intel");
        SIGAR_SSTRCPY(info->model, "Itanium");
    #else
        SIGAR_SSTRCPY(info->vendor, "HP");
        SIGAR_SSTRCPY(info->model, "PA RISC");
    #endif
}
```

### 4. 进程监控

#### 进程列表

```c
int sigar_os_proc_list_get(sigar_t *sigar, sigar_proc_list_t *proclist)
{
    int idx = 0;
    struct pst_status proctab[PROC_ELTS];

    // 批量获取进程 (每次 16 个)
    while ((num = pstat_getproc(proctab, sizeof(proctab[0]),
                                PROC_ELTS, idx)) > 0)
    {
        for (i = 0; i < num; i++) {
            SIGAR_PROC_LIST_GROW(proclist);
            proclist->data[proclist->number++] = proctab[i].pst_pid;
        }

        idx = proctab[num-1].pst_idx + 1;  // 下一次查询的索引
    }
}
```

#### 进程内存

```c
int sigar_proc_mem_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_mem_t *procmem)
{
    struct pst_status *pinfo = sigar->pinfo;

    // 虚拟内存大小 = text + data + stack + shm + mmap + u-area + io
    procmem->size =
        pinfo->pst_vtsize +    // text
        pinfo->pst_vdsize +    // data
        pinfo->pst_vssize +    // stack
        pinfo->pst_vshmsize +  // shared memory
        pinfo->pst_vmmsize +   // mem-mapped files
        pinfo->pst_vusize +    // U-Area & K-Stack
        pinfo->pst_viosize;    // I/O dev mapping

    procmem->resident = pinfo->pst_rssize * pagesize;
    procmem->share = pinfo->pst_vshmsize * pagesize;

    procmem->minor_faults = pinfo->pst_minorfaults;
    procmem->major_faults = pinfo->pst_majorfaults;
}
```

#### 进程状态

```c
int sigar_proc_state_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_state_t *procstate)
{
    struct pst_status *pinfo = sigar->pinfo;

    procstate->name = pinfo->pst_ucomm;
    procstate->ppid = pinfo->pst_ppid;
    procstate->tty = makedev(pinfo->pst_term.psd_major,
                            pinfo->pst_term.psd_minor);
    procstate->priority = pinfo->pst_pri;
    procstate->nice = pinfo->pst_nice;
    procstate->threads = pinfo->pst_nlwps;
    procstate->processor = pinfo->pst_procnum;

    // 状态映射
    switch (pinfo->pst_stat) {
        case PS_SLEEP:  procstate->state = 'S'; break;
        case PS_RUN:    procstate->state = 'R'; break;
        case PS_STOP:   procstate->state = 'T'; break;
        case PS_ZOMBIE: procstate->state = 'Z'; break;
        case PS_IDLE:   procstate->state = 'D'; break;
    }
}
```

#### 进程参数 (命令行)

```c
int sigar_os_proc_args_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_args_t *procargs)
{
    #ifdef PSTAT_GETCOMMANDLINE  // HPUX 11i v2+
        char buf[1024];  // 内核限制

        if (pstat_getcommandline(buf, sizeof(buf), sizeof(buf[0]), pid) == -1) {
            return errno;
        }
        args = buf;
    #else
        struct pst_status status;
        pstat_getproc(&status, sizeof(status), 0, pid);
        args = status.pst_cmd;  // 旧版本方法
    #endif

    // 解析空格分隔的参数
    while (*args && (arg = sigar_getword(&args, ' '))) {
        SIGAR_PROC_ARGS_GROW(procargs);
        procargs->data[procargs->number++] = arg;
    }
}
```

#### 进程文件描述符

```c
int sigar_proc_fd_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_fd_t *procfd)
{
    struct pst_status status;
    int idx = 0, n;
    struct pst_fileinfo2 psf[16];

    pstat_getproc(&status, sizeof(status), 0, pid);

    // HPUX 11.31 移除了已弃用的 pstat_getfile,使用 pstat_getfile2
    while ((n = pstat_getfile2(psf, sizeof(psf[0]),
                               sizeof(psf)/sizeof(psf[0]),
                               idx, pid)) > 0)
    {
        procfd->total += n;
        idx = psf[n-1].psf_fd + 1;
    }
}
```

#### 进程可执行文件路径

```c
int sigar_proc_exe_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_exe_t *procexe)
{
    #ifdef __pst_fid  // HPUX 11.11+
        struct pst_status status;

        pstat_getproc(&status, sizeof(status), 0, pid);

        // 工作目录
        pstat_getpathname(procexe->cwd, sizeof(procexe->cwd),
                          &status.pst_fid_cdir);

        // 可执行文件
        pstat_getpathname(procexe->name, sizeof(procexe->name),
                          &status.pst_fid_text);

        // 根目录
        pstat_getpathname(procexe->root, sizeof(procexe->root),
                          &status.pst_fid_rdir);

        procexe->arch = sigar_elf_file_guess_arch(sigar, procexe->name);
    #else
        return SIGAR_ENOTIMPL;  // HPUX 11.00 不支持
    #endif
}
```

#### 线程 CPU 使用 (仅 PA-RISC)

```c
int sigar_thread_cpu_get(sigar_t *sigar, sigar_uint64_t id, sigar_thread_cpu_t *cpu)
{
    #ifdef __ia64__
        return SIGAR_ENOTIMPL;  // Itanium 不支持
    #else
        struct lwpinfo info;

        if (id != 0) {
            return SIGAR_ENOTIMPL;  // 仅支持主线程
        }

        _lwp_info(&info);

        cpu->user  = TIME_NSEC(info.lwp_utime);
        cpu->sys   = TIME_NSEC(info.lwp_stime);
        cpu->total = cpu->user + cpu->sys;
    #endif
}
```

### 5. 文件系统监控

#### 文件系统列表

```c
int sigar_file_system_list_get(sigar_t *sigar, sigar_file_system_list_t *fslist)
{
    FILE *fp = setmntent(MNT_MNTTAB, "r");

    while ((ent = getmntent(fp))) {
        // 跳过 swap (devname == "...")
        if (strEQ(ent->mnt_type, "swap")) {
            continue;
        }

        fsp->dir_name = ent->mnt_dir;
        fsp->dev_name = ent->mnt_fsname;
        fsp->sys_type_name = ent->mnt_type;
        fsp->options = ent->mnt_opts;
    }

    endmntent(fp);
}
```

#### 文件系统使用情况

```c
int sigar_file_system_usage_get(sigar_t *sigar, const char *dirname,
                                sigar_file_system_usage_t *fsusage)
{
    sigar_statvfs(sigar, dirname, fsusage);

    // 从逻辑卷获取磁盘 I/O 统计
    if (stat(dirname, &sb) == 0) {
        struct pst_lvinfo lv;
        struct stat devsb;
        char *devname;

        ent = sigar_cache_get(sigar->fsdev, SIGAR_FSDEV_ID(sb));

        if (stat(ent->value, &devsb) == 0) {
            pstat_getlv(&lv, sizeof(lv), 0, (int)devsb.st_rdev);

            fsusage->disk.reads  = lv.psl_rxfer;
            fsusage->disk.writes = lv.psl_wxfer;
            fsusage->disk.read_bytes  = lv.psl_rcount;
            fsusage->disk.write_bytes = lv.psl_wcount;
        }
    }
}
```

#### 文件系统类型

```c
int sigar_os_fs_type_get(sigar_file_system_t *fsp)
{
    switch (*fsp->sys_type_name) {
        case 'h':
            if (strEQ(fsp->sys_type_name, "hfs")) {
                fsp->type = SIGAR_FSTYPE_LOCAL_DISK;
            }
            break;
        case 'c':
            if (strEQ(fsp->sys_type_name, "cdfs")) {
                fsp->type = SIGAR_FSTYPE_CDROM;
            }
            break;
    }
}
```

### 6. 网络监控

#### 网络接口统计

```c
int sigar_net_interface_stat_get(sigar_t *sigar, const char *name,
                                 sigar_net_interface_stat_t *ifstat)
{
    mib_ifEntry mib;

    get_mib_ifstat(sigar, name, &mib);

    ifstat->rx_bytes    = mib.ifInOctets;
    ifstat->rx_packets  = mib.ifInUcastPkts + mib.ifInNUcastPkts;
    ifstat->rx_errors   = mib.ifInErrors;
    ifstat->rx_dropped  = mib.ifInDiscards;
    ifstat->tx_bytes    = mib.ifOutOctets;
    ifstat->tx_packets  = mib.ifOutUcastPkts + mib.ifOutNUcastPkts;
    ifstat->tx_errors   = mib.ifOutErrors;
    ifstat->tx_dropped  = mib.ifOutDiscards;
    ifstat->speed       = mib.ifSpeed;
}
```

#### 路由表

```c
int sigar_net_route_list_get(sigar_t *sigar, sigar_net_route_list_t *routelist)
{
    mib_ipRouteEnt *routes;

    // 获取路由表数量
    parms.objid = ID_ipRouteNumEnt;
    get_mib_info(sigar, &parms);

    // 获取路由表
    parms.objid = ID_ipRouteTable;
    parms.buffer = routes;
    get_mib_info(sigar, &parms);

    for (i = 0; i < count; i++) {
        sigar_net_address_set(route->destination, routes[i].Dest);
        sigar_net_address_set(route->mask, routes[i].Mask);
        sigar_net_address_set(route->gateway, routes[i].NextHop);
        sigar_if_indextoname(sigar, route->ifname, routes[i].IfIndex);

        route->flags = SIGAR_RTF_UP;
        if (routes[i].Dest == 0 && routes[i].Mask == 0) {
            route->flags |= SIGAR_RTF_GATEWAY;
        }
    }
}
```

#### UDP 监听连接

```c
static int net_conn_get_udp_listen(sigar_net_connection_walker_t *walker)
{
    mib_udpLsnEnt *entries;

    // 获取 UDP 监听连接数量
    parms.objid = ID_udpLsnNumEnt;
    get_mib_info(sigar, &parms);

    // 获取 UDP 监听连接列表
    parms.objid = ID_udpLsnTable;
    parms.buffer = entries;
    get_mib_info(sigar, &parms);
}
```

#### IPv6 接口配置

```c
int sigar_net_interface_ipv6_config_get(sigar_t *sigar, const char *name,
                                        sigar_net_interface_config_t *ifconfig)
{
    int sock = socket(AF_INET6, SOCK_DGRAM, 0);
    struct if_laddrreq iflr;

    SIGAR_SSTRCPY(iflr.iflr_name, name);

    // 获取 IPv6 地址
    ioctl(sock, SIOCGLIFADDR, &iflr);
    struct in6_addr *addr = SIGAR_SIN6_ADDR(&iflr.iflr_addr);
    sigar_net_address6_set(ifconfig->address6, addr);

    // 获取子网掩码
    ioctl(sock, SIOCGLIFNETMASK, &iflr);
    ifconfig->prefix6_length = 10;  // XXX

    close(sock);
}
```

## 技术亮点

### 1. pstat 统一接口

- 单一函数家族提供所有系统信息
- 避免访问 /proc 文件系统
- 批量操作减少系统调用

### 2. 架构自适应

- `_PSTAT64` 宏检测 32/64 位
- IA64 和 PA-RISC 分支处理
- 版本特定的 API 检测

### 3. 双模式网络统计

- MIB 接口提供高效网络统计
- Socket ioctl 提供补充信息
- 避免解析 /proc/net

### 4. 逻辑卷集成

- `pstat_getlv` 获取磁盘 I/O
- 直接与文件系统关联
- 提供读写字节数

### 5. 批量进程获取

- `PROC_ELTS` 控制批量大小
- 减少 `pstat_getproc` 调用
- 使用索引遍历

### 6. 版本兼容性

- HPUX 11.00 基础支持
- HPUX 11.11 文件路径支持
- HPUX 11.31 CPU 频率支持

## 未实现功能

以下功能在 HPUX 上未实现:

- `sigar_system_stats_get` - 系统统计
- `sigar_proc_env_get` - 进程环境变量
- `sigar_proc_modules_get` - 进程模块
- `sigar_disk_usage_get` - 磁盘使用情况
- `sigar_sys_info_get_uuid` - 系统 UUID

## 总结

HPUX 分支的实现特点:

1. **统一的 pstat 接口** - 所有系统统计通过 pstat 系列函数获取
2. **MIB 网络统计** - 使用 MIB 接口获取网络信息
3. **架构兼容** - 支持 PA-RISC 和 Itanium 两种架构
4. **版本适配** - 针对不同 HPUX 版本的 API 差异处理
5. **高效缓存** - 进程信息缓存减少系统调用
6. **逻辑卷集成** - 文件系统和磁盘 I/O 统计关联

HPUX 作为专有 UNIX 系统,其独特的 pstat 接口和 MIB 系统使得实现相对独立且高效。
