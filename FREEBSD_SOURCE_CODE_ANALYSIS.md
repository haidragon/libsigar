# libsigar FreeBSD 平台源码分析

## 概述

libsigar 在 FreeBSD 平台的实现主要通过 `sysctl` 系统调用、`kvm` 库和 `/proc` 文件系统获取系统信息。FreeBSD 是一个完整的、高性能的 UNIX 操作系统，提供了丰富的系统监控接口。

## 核心架构

### sigar_t 结构体 (sigar_os.h:54-75)

```c
struct sigar_t {
    SIGAR_T_BASE;
    int pagesize;                    // 页面大小
    time_t last_getprocs;            // 最后获取进程时间
    sigar_pid_t last_pid;            // 最后处理的 PID
    bsd_pinfo_t *pinfo;              // 进程信息缓冲区
    int lcpu;                        // 每个物理 CPU 的逻辑 CPU 数
    size_t argmax;                   // 参数最大长度
    kvm_t *kmem;                     // kvm 句柄
    unsigned long koffsets[KOFFSET_MAX]; // 内核符号偏移量
    int proc_mounted;                // /proc 是否挂载
};
```

### 内核符号偏移量 (sigar_os.h:38-46)

```c
enum {
    KOFFSET_CPUINFO,    // CPU 时间统计
    KOFFSET_VMMETER,    // 虚拟内存统计
    KOFFSET_MAX
};
```

### FreeBSD 版本宏 (freebsd_sigar.c:76-82)

```c
#ifdef __FreeBSD__
#  if (__FreeBSD_version >= 500013)
#    define SIGAR_FREEBSD5
#  else
#    define SIGAR_FREEBSD4
#  endif
#endif
```

## 核心技术

### 1. sysctl 系统调用

`sysctl` 是 FreeBSD 获取内核信息的主要接口。

#### 基本用法

```c
int mib[2];
size_t len;

mib[0] = CTL_HW;
mib[1] = HW_PAGESIZE;
len = sizeof(pagesize);
sysctl(mib, 2, &pagesize, &len, NULL, 0);
```

#### sysctlbyname 接口

```c
size_t size = sizeof(value);
sysctlbyname("vm.stats.sys.v_swtch", &value, &size, NULL, 0);
```

### 2. kvm 库

`kvm` (Kernel VM) 库提供访问内核内存和数据的接口。

#### 初始化 (freebsd_sigar.c:196)

```c
(*sigar)->kmem = kvm_open(NULL, NULL, NULL, O_RDONLY, NULL);
```

#### 读取内核数据 (freebsd_sigar.c:154-165)

```c
static int kread(sigar_t *sigar, void *data, int size, long offset)
{
    if (!sigar->kmem) {
        return SIGAR_EPERM_KMEM;
    }

    if (kvm_read(sigar->kmem, offset, data, size) != size) {
        return errno;
    }

    return SIGAR_OK;
}
```

#### 内核符号查找 (freebsd_sigar.c:132-152)

```c
static int get_koffsets(sigar_t *sigar)
{
    struct nlist klist[] = {
        { "_cp_time" },   // CPU 时间统计
        { "_cnt" },       // 虚拟内存计数
        { NULL }
    };

    if (!sigar->kmem) {
        return SIGAR_EPERM_KMEM;
    }

    kvm_nlist(sigar->kmem, klist);

    for (i=0; i<KOFFSET_MAX; i++) {
        sigar->koffsets[i] = klist[i].n_value;
    }

    return SIGAR_OK;
}
```

### 3. /proc 文件系统

FreeBSD 的 `/proc` 文件系统提供进程信息。

#### 检查挂载状态 (freebsd_sigar.c:197-202)

```c
struct stat sb;
if (stat("/proc/curproc", &sb) < 0) {
    (*sigar)->proc_mounted = 0;
}
else {
    (*sigar)->proc_mounted = 1;
}
```

### 4. 初始化流程 (freebsd_sigar.c:172-218)

```c
int sigar_os_open(sigar_t **sigar)
{
    int mib[2];
    int ncpu;
    size_t len;
    struct timeval boottime;
    struct stat sb;

    // 1. 获取 CPU 数量
    len = sizeof(ncpu);
    mib[0] = CTL_HW;
    mib[1] = HW_NCPU;
    sysctl(mib, 2, &ncpu, &len, NULL, 0);

    // 2. 获取启动时间
    len = sizeof(boottime);
    mib[0] = CTL_KERN;
    mib[1] = KERN_BOOTTIME;
    sysctl(mib, 2, &boottime, &len, NULL, 0);

    // 3. 打开 kvm
    *sigar = malloc(sizeof(**sigar));
    (*sigar)->kmem = kvm_open(NULL, NULL, NULL, O_RDONLY, NULL);

    // 4. 检查 /proc 是否挂载
    if (stat("/proc/curproc", &sb) < 0) {
        (*sigar)->proc_mounted = 0;
    }
    else {
        (*sigar)->proc_mounted = 1;
    }

    // 5. 获取内核符号偏移量
    get_koffsets(*sigar);

    // 6. 初始化其他字段
    (*sigar)->ncpu = ncpu;
    (*sigar)->lcpu = -1;
    (*sigar)->argmax = 0;
    (*sigar)->boot_time = boottime.tv_sec;

    (*sigar)->pagesize = getpagesize();
    (*sigar)->ticks = 100;
    (*sigar)->last_pid = -1;

    return SIGAR_OK;
}
```

## 核心功能模块

### 1. 内存监控

#### 虚拟内存统计 (freebsd_sigar.c:247-322)

```c
static int sigar_vmstat(sigar_t *sigar, struct vmmeter *vmstat)
{
    int status;
    size_t size = sizeof(unsigned int);

    // 尝试从 kmem 读取
    status = kread(sigar, vmstat, sizeof(*vmstat),
                   sigar->koffsets[KOFFSET_VMMETER]);

    if (status == SIGAR_OK) {
        return SIGAR_OK;
    }

    SIGAR_ZERO(vmstat);

    // 回退到 sysctlbyname
#define GET_VM_STATS(cat, name, used) \
    if (used) sysctlbyname("vm.stats." #cat "." #name, &vmstat->name, &size, NULL, 0)

    // 系统统计
    GET_VM_STATS(sys, v_swtch, 0);      // 上下文切换
    GET_VM_STATS(sys, v_trap, 0);       // 陷阱
    GET_VM_STATS(sys, v_syscall, 0);    // 系统调用
    GET_VM_STATS(sys, v_intr, 0);       // 中断
    GET_VM_STATS(sys, v_soft, 0);       // 软件中断

    // 虚拟内存统计
    GET_VM_STATS(vm, v_vm_faults, 0);
    GET_VM_STATS(vm, v_cow_faults, 0);
    GET_VM_STATS(vm, v_cow_optim, 0);
    GET_VM_STATS(vm, v_zfod, 0);
    GET_VM_STATS(vm, v_ozfod, 0);
    GET_VM_STATS(vm, v_swapin, 1);       // 换入
    GET_VM_STATS(vm, v_swapout, 1);      // 换出
    GET_VM_STATS(vm, v_swappgsin, 0);
    GET_VM_STATS(vm, v_swappgsout, 0);
    GET_VM_STATS(vm, v_vnodein, 1);
    GET_VM_STATS(vm, v_vnodeout, 1);
    GET_VM_STATS(vm, v_vnodepgsin, 0);
    GET_VM_STATS(vm, v_vnodepgsout, 0);
    GET_VM_STATS(vm, v_page_size, 0);
    GET_VM_STATS(vm, v_page_count, 0);
    GET_VM_STATS(vm, v_free_reserved, 0);
    GET_VM_STATS(vm, v_free_target, 0);
    GET_VM_STATS(vm, v_free_min, 0);
    GET_VM_STATS(vm, v_free_count, 1);   // 空闲页数
    GET_VM_STATS(vm, v_wire_count, 0);
    GET_VM_STATS(vm, v_active_count, 0);
    GET_VM_STATS(vm, v_inactive_target, 0);
    GET_VM_STATS(vm, v_inactive_count, 1); // 不活动页数
    GET_VM_STATS(vm, v_cache_count, 1);    // 缓存页数
    GET_VM_STATS(vm, v_forks, 0);
    GET_VM_STATS(vm, v_vforks, 0);
    GET_VM_STATS(vm, v_rforks, 0);
#undef GET_VM_STATS

    return SIGAR_OK;
}
```

#### 内存信息获取 (freebsd_sigar.c:324-364)

```c
int sigar_mem_get(sigar_t *sigar, sigar_mem_t *mem)
{
    sigar_uint64_t kern = 0;
    unsigned long mem_total;
    struct vmmeter vmstat;
    int mib[2];
    size_t len;

    // 获取页面大小
    mib[0] = CTL_HW;
    mib[1] = HW_PAGESIZE;
    len = sizeof(sigar->pagesize);
    sysctl(mib, 2, &sigar->pagesize, &len, NULL, 0);

    // 获取物理内存总量
    mib[1] = HW_PHYSMEM;
    len = sizeof(mem_total);
    sysctl(mib, 2, &mem_total, &len, NULL, 0);

    mem->total = mem_total;

    // 获取虚拟内存统计
    if ((status = sigar_vmstat(sigar, &vmstat)) == SIGAR_OK) {
        kern = vmstat.v_cache_count + vmstat.v_inactive_count;
        kern *= sigar->pagesize;
        mem->free = vmstat.v_free_count;
        mem->free *= sigar->pagesize;
    }

    mem->used = mem->total - mem->free;

    mem->actual_free = mem->free + kern;    // 实际可用 (含缓存)
    mem->actual_used = mem->used - kern;    // 实际使用 (不含缓存)

    sigar_mem_calc_ram(sigar, mem);

    return SIGAR_OK;
}
```

### 2. 交换空间监控

#### 交换空间统计 (freebsd_sigar.c:440-476)

```c
int sigar_swap_get(sigar_t *sigar, sigar_swap_t *swap)
{
    int status;
    struct kvm_swap kswap[1];
    struct vmmeter vmstat;

    // 尝试使用 sysctl (FreeBSD 5+)
    if (getswapinfo_sysctl(kswap, 1) != SIGAR_OK) {
        // 回退到 kvm
        if (!sigar->kmem) {
            return SIGAR_EPERM_KMEM;
        }

        if (kvm_getswapinfo(sigar->kmem, kswap, 1, 0) < 0) {
            return errno;
        }
    }

    if (kswap[0].ksw_total == 0) {
        swap->total = 0;
        swap->used = 0;
        swap->free = 0;
        return SIGAR_OK;
    }

    swap->total = kswap[0].ksw_total * sigar->pagesize;
    swap->used = kswap[0].ksw_used * sigar->pagesize;
    swap->free = swap->total - swap->used;

    // 获取分页统计
    if ((status = sigar_vmstat(sigar, &vmstat)) == SIGAR_OK) {
        swap->page_in = vmstat.v_swapin + vmstat.v_vnodein;
        swap->page_out = vmstat.v_swapout + vmstat.v_vnodeout;
    }
    else {
        swap->page_in = swap->page_out = -1;
    }

    return SIGAR_OK;
}
```

#### sysctl 方法 (freebsd_sigar.c:369-433)

```c
/* FreeBSD 5+ sysctl 方法 */
static int getswapinfo_sysctl(struct kvm_swap *swap_ary, int swap_max)
{
    int ti, ttl;
    size_t mibi, len, size;
    int soid[SWI_MAXMIB];
    struct xswdev xsd;
    struct kvm_swap tot;
    int unswdev, dmmax;

    // 获取 dmmax
    size = sizeof(dmmax);
    if (sysctlbyname("vm.dmmax", &dmmax, &size, NULL, 0) == -1) {
        return errno;
    }

    // 获取 swap_info 的 MIB
    mibi = SWI_MAXMIB - 1;
    if (sysctlnametomib("vm.swap_info", soid, &mibi) == -1) {
        return errno;
    }

    bzero(&tot, sizeof(tot));
    for (unswdev = 0;; unswdev++) {
        soid[mibi] = unswdev;
        len = sizeof(xsd);
        if (sysctl(soid, mibi + 1, &xsd, &len, NULL, 0) == -1) {
            if (errno == ENOENT) {
                break;
            }
            return errno;
        }

        ttl = xsd.xsw_nblks - dmmax;
        if (unswdev < swap_max - 1) {
            bzero(&swap_ary[unswdev], sizeof(swap_ary[unswdev]));
            swap_ary[unswdev].ksw_total = ttl;
            swap_ary[unswdev].ksw_used = xsd.xsw_used;
            swap_ary[unswdev].ksw_flags = xsd.xsw_flags;
        }
        tot.ksw_total += ttl;
        tot.ksw_used += xsd.xsw_used;
    }

    return SIGAR_OK;
}
```

### 3. CPU 监控

#### CPU 统计 (freebsd_sigar.c:484-500+)

```c
int sigar_cpu_get(sigar_t *sigar, sigar_cpu_t *cpu)
{
    int status;
    cp_time_t cp_time[CPUSTATES];
    size_t size = sizeof(cp_time);

    // 优先使用 sysctl (不需要 kmem 权限)
    if (sysctlbyname("kern.cp_time", &cp_time, &size, NULL, 0) == -1) {
        // 回退到 kmem
        status = kread(sigar, &cp_time, sizeof(cp_time),
                       sigar->koffsets[KOFFSET_CPUINFO]);
    }
    else {
        status = SIGAR_OK;
    }

    if (status != SIGAR_OK) {
        return status;
    }

    cpu->user = SIGAR_TICK2MSEC(cp_time[CP_USER]);
    cpu->nice = SIGAR_TICK2MSEC(cp_time[CP_NICE]);
    cpu->sys  = SIGAR_TICK2MSEC(cp_time[CP_SYS]);
    cpu->idle = SIGAR_TICK2MSEC(cp_time[CP_IDLE]);
    cpu->wait = 0;  // FreeBSD 没有 iowait
    cpu->irq = SIGAR_TICK2MSEC(cp_time[CP_INTR]);
    cpu->soft_irq = 0;
    cpu->stolen = 0;
    cpu->total = cpu->user + cpu->nice + cpu->sys + cpu->idle + cpu->irq;

    return SIGAR_OK;
}
```

### 4. 进程监控

#### 进程列表获取

FreeBSD 使用 `kvm_getprocs` 获取进程列表。

```c
int sigar_os_proc_list_get(sigar_t *sigar, sigar_proc_list_t *proclist)
{
    int count, i;
    struct kinfo_proc *pinfo;

    // 获取所有进程
    pinfo = kvm_getprocs(sigar->kmem, KERN_PROC_ALL, 0, &count);

    for (i=0; i<count; i++) {
        proclist->data[proclist->number++] = pinfo[i].KI_PID;
    }

    return SIGAR_OK;
}
```

#### 进程兼容性宏 (freebsd_sigar.c:84-126)

```c
#if defined(SIGAR_FREEBSD5)

#define KI_FD   ki_fd
#define KI_PID  ki_pid
#define KI_PPID ki_ppid
#define KI_PRI  ki_pri.pri_user
#define KI_NICE ki_nice
#define KI_COMM ki_comm
#define KI_STAT ki_stat
#define KI_UID  ki_ruid
#define KI_GID  ki_rgid
#define KI_EUID ki_svuid
#define KI_EGID ki_svgid
#define KI_SIZE ki_size
#define KI_RSS  ki_rssize
#define KI_TSZ  ki_tsize
#define KI_DSZ  ki_dsize
#define KI_SSZ  ki_ssize
#define KI_FLAG ki_flag
#define KI_START ki_start

#elif defined(SIGAR_FREEBSD4)

#define KI_FD   kp_proc.p_fd
#define KI_PID  kp_proc.p_pid
#define KI_PPID kp_eproc.e_ppid
#define KI_PRI  kp_proc.p_priority
#define KI_NICE kp_proc.p_nice
#define KI_COMM kp_proc.p_comm
#define KI_STAT kp_proc.p_stat
#define KI_UID  kp_eproc.e_pcred.p_ruid
#define KI_GID  kp_eproc.e_pcred.p_rgid
#define KI_EUID kp_eproc.e_pcred.p_svuid
#define KI_EGID kp_eproc.e_pcred.p_svgid
#define KI_RSS  kp_eproc.e_vm.vm_rssize
#define KI_TSZ  kp_eproc.e_vm.vm_tsize
#define KI_DSZ  kp_eproc.e_vm.vm_dsize
#define KI_SSZ  kp_eproc.e_vm.vm_ssize
#define KI_FLAG kp_eproc.e_flag
#define KI_START kp_proc.p_starttime

#endif
```

这些宏提供了统一的字段访问接口，屏蔽了 FreeBSD 4 和 FreeBSD 5 之间的结构差异。

### 5. 网络监控

#### 网络接口统计

FreeBSD 使用 `sysctl` 和 `ifnet` 结构获取网络接口统计。

```c
int sigar_net_interface_stat_get(sigar_t *sigar, const char *name,
                                 sigar_net_interface_stat_t *ifstat)
{
    int mib[6];
    struct if_msghdr *ifm;
    struct if_data ifdata;
    char *buf, *lim, *next;
    size_t len;

    mib[0] = CTL_NET;
    mib[1] = PF_ROUTE;
    mib[2] = 0;
    mib[3] = AF_INET;
    mib[4] = NET_RT_IFLIST;
    mib[5] = 0;

    // 获取接口列表
    if (sysctl(mib, 6, NULL, &len, NULL, 0) < 0) {
        return errno;
    }

    buf = malloc(len);
    if (sysctl(mib, 6, buf, &len, NULL, 0) < 0) {
        free(buf);
        return errno;
    }

    lim = buf + len;
    for (next = buf; next < lim; next += ifm->ifm_msglen) {
        ifm = (struct if_msghdr *)next;

        if (ifm->ifm_type == RTM_IFINFO) {
            ifdata = ifm->ifm_data;

            if (strcmp(ifm->ifm_name, name) == 0) {
                ifstat->rx_packets = ifdata.ifi_ipackets;
                ifstat->tx_packets = ifdata.ifi_opackets;
                ifstat->rx_bytes = ifdata.ifi_ibytes;
                ifstat->tx_bytes = ifdata.ifi_obytes;
                ifstat->rx_errors = ifdata.ifi_ierrors;
                ifstat->tx_errors = ifdata.ifi_oerrors;
                break;
            }
        }
    }

    free(buf);
    return SIGAR_OK;
}
```

#### 网络连接

FreeBSD 从内核读取 TCP/UDP 连接表。

```c
int sigar_net_connection_list_get(sigar_t *sigar,
                                  sigar_net_connection_list_t *connlist,
                                  int flags)
{
    struct tcpcbhead tcbhead;
    struct inpcbhead inpcbhead;
    int i;

    // 读取 TCP 连接表
    if (flags & SIGAR_NETCONN_TCP) {
        kread(sigar, &tcbhead, sizeof(tcbhead),
               sigar->koffsets[KOFFSET_TCBTABLE]);

        LIST_FOREACH(tcpcb, &tcbhead, tcb_list) {
            // 处理每个 TCP 连接
            ...
        }
    }

    // 读取 UDP 连接表
    if (flags & SIGAR_NETCONN_UDP) {
        // 类似处理
        ...
    }

    return SIGAR_OK;
}
```

## 性能优化

### 1. sysctl 优先使用

优先使用 `sysctl` 获取信息，只有在必要时才使用 `kvm`。

### 2. 版本兼容性

通过宏和条件编译支持多个 FreeBSD 版本。

### 3. 批量操作

使用 `kvm_getprocs` 批量获取进程信息。

## 特殊技术

### 1. ARG_MAX 限制 (freebsd_sigar.c:244-245)

```c
/* FreeBSD 6.0 的 ARG_MAX == 262144，会破坏栈 */
#define SIGAR_ARG_MAX 65536
```

限制参数最大长度，避免栈溢出。

### 2. /proc 挂载检查 (freebsd_sigar.c:128-130)

```c
#define PROCFS_STATUS(status) \
    ((((status) != SIGAR_OK) && !sigar->proc_mounted) ? \
     SIGAR_ENOTIMPL : status)
```

如果 `/proc` 未挂载，返回 SIGAR_ENOTIMPL 而非错误。

### 3. 版本检测

```c
#if defined(__FreeBSD__)
#  if (__FreeBSD_version >= 500013)
#    define SIGAR_FREEBSD5
#  else
#    define SIGAR_FREEBSD4
#  endif
#endif
```

### 4. NFS 统计 (freebsd_sigar.c:39-45)

```c
#if defined(__FreeBSD__) && (__FreeBSD_version >= 500013)
#define SIGAR_FREEBSD5_NFSSTAT
#include <nfsclient/nfs.h>
#include <nfsserver/nfs.h>
#else
#include <nfs/nfs.h>
#endif
```

根据版本选择不同的 NFS 头文件。

## 技术亮点

### 1. sysctl 统一接口

`sysctl` 提供了统一的内核信息访问接口，无需特殊权限。

### 2. kvm 库

`kvm` 库提供了内核内存访问能力，支持深度监控。

### 3. 版本兼容性

通过宏和条件编译支持多个 FreeBSD 版本。

### 4. 双模式支持

同时支持 `sysctl` 和 `kvm` 两种方式。

## 限制和挑战

### 1. kmem 权限

访问 `/dev/kmem` 需要 root 权限。

### 2. /proc 可选

FreeBSD 的 `/proc` 文件系统是可选的，某些功能可能不可用。

### 3. 版本差异

不同 FreeBSD 版本之间存在结构差异，需要兼容性处理。

### 4. NFS 变化

NFS 统计接口在不同版本中有变化。

## 总结

libsigar FreeBSD 实现充分利用了 FreeBSD 的 `sysctl`、`kvm` 库和 `/proc` 文件系统，提供了全面的系统监控功能。其设计特点包括：

1. **统一接口**: sysctl 提供了统一的内核信息访问接口
2. **深度访问**: kvm 库提供内核内存访问能力
3. **版本兼容**: 支持多个 FreeBSD 版本
4. **双模式**: 同时支持 sysctl 和 kvm 两种方式
5. **可选功能**: 优雅处理 /proc 未挂载的情况

这种实现方式展示了 FreeBSD 系统架构的特点，为其他 BSD 变体的系统监控提供了参考。
