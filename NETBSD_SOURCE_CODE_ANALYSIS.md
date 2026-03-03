# libsigar NetBSD 平台源码分析

## 概述

libsigar 在 NetBSD 平台的实现主要通过 `sysctl` 系统调用和 `kvm` 库获取系统信息。NetBSD 是一个专注于设计简洁性和可移植性的开源 UNIX 操作系统。

## 核心架构

### sigar_t 结构体

NetBSD 与 FreeBSD 共享相同的头文件 `sigar_os.h` (sigar_os.h:54-75)。

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
    KOFFSET_TCPSTAT,    // TCP 统计
    KOFFSET_TCBTABLE,   // TCP 控制块表
    KOFFSET_MAX
};
```

### 进程信息类型 (sigar_os.h:49-52)

```c
#if defined(__OpenBSD__) || defined(__NetBSD__)
typedef struct kinfo_proc2 bsd_pinfo_t;
#else
typedef struct kinfo_proc bsd_pinfo_t;
#endif
```

NetBSD 使用 `kinfo_proc2` 结构，这是 NetBSD 特有的扩展进程信息结构。

## 核心技术

### 1. sysctl 系统调用

NetBSD 的 `sysctl` 系统调用类似于 FreeBSD，提供了统一的内核信息访问接口。

#### 基本用法

```c
int mib[2];
size_t len;

mib[0] = CTL_HW;
mib[1] = HW_PAGESIZE;
len = sizeof(pagesize);
sysctl(mib, 2, &pagesize, &len, NULL, 0);
```

### 2. kvm 库

NetBSD 的 `kvm` 库提供了访问内核内存和数据的能力。

#### 初始化

```c
(*sigar)->kmem = kvm_open(NULL, NULL, NULL, O_RDONLY, NULL);
```

#### 读取内核数据

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

### 3. 初始化流程

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
    (*sigar)->pagesize = getpagesize();
    (*sigar)->ticks = sysconf(_SC_CLK_TCK);

    return SIGAR_OK;
}
```

## 核心功能模块

### 1. 内存监控

NetBSD 的内存监控与 FreeBSD 类似，使用 `sysctl` 获取物理内存和虚拟内存统计。

#### 内存信息获取

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

    mem->actual_free = mem->free + kern;
    mem->actual_used = mem->used - kern;

    sigar_mem_calc_ram(sigar, mem);

    return SIGAR_OK;
}
```

### 2. CPU 监控

NetBSD 的 CPU 统计使用 `sysctl` 或 `kvm` 读取内核的 `cp_time` 数组。

```c
int sigar_cpu_get(sigar_t *sigar, sigar_cpu_t *cpu)
{
    int status;
    cp_time_t cp_time[CPUSTATES];
    size_t size = sizeof(cp_time);

    // 优先使用 sysctl
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
    cpu->sys = SIGAR_TICK2MSEC(cp_time[CP_SYS]);
    cpu->idle = SIGAR_TICK2MSEC(cp_time[CP_IDLE]);
    cpu->total = cpu->user + cpu->nice + cpu->sys + cpu->idle;

    return SIGAR_OK;
}
```

### 3. 进程监控

NetBSD 使用 `kvm_getproc2` 获取进程列表，这是 NetBSD 特有的扩展函数。

```c
int sigar_os_proc_list_get(sigar_t *sigar, sigar_proc_list_t *proclist)
{
    int count, i;
    struct kinfo_proc2 *pinfo;

    // 获取所有进程 (NetBSD 特定接口)
    pinfo = kvm_getproc2(sigar->kmem, KERN_PROC_ALL, 0,
                         sizeof(struct kinfo_proc2), &count);

    for (i=0; i<count; i++) {
        proclist->data[proclist->number++] = pinfo[i].p_pid;
    }

    return SIGAR_OK;
}
```

### 4. 网络监控

NetBSD 的网络监控通过 `sysctl` 和内核结构实现。

#### 网络接口统计

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

## 性能优化

### 1. sysctl 优先使用

优先使用 `sysctl` 获取信息，只有在必要时才使用 `kvm`。

### 2. 进程信息批量获取

使用 `kvm_getproc2` 一次性获取所有进程信息。

## 特殊技术

### 1. kinfo_proc2 结构

NetBSD 使用扩展的进程信息结构 `kinfo_proc2`，包含更多字段。

```c
struct kinfo_proc2 {
    struct proc kp_proc;
    struct eproc eproc;
    // 更多 NetBSD 特定字段
};
```

### 2. 进程状态映射

NetBSD 进程状态到通用状态的映射。

```c
#define PIDL_IDLE      'I'
#define PIDL_RUN       'R'
#define PIDL_SLEEP     'S'
#define PIDL_STOP      'T'
#define PIDL_ZOMBIE    'Z'
#define PIDL_DEAD      'X'
```

## 技术亮点

### 1. 设计简洁性

NetBSD 以设计简洁著称，代码结构清晰易懂。

### 2. 可移植性

NetBSD 的代码具有良好的可移植性，易于在其他平台上运行。

### 3. 扩展性

使用 `kinfo_proc2` 等扩展结构提供更多功能。

## 限制和挑战

### 1. kmem 权限

访问 `/dev/kmem` 需要 root 权限。

### 2. /proc 可选

NetBSD 的 `/proc` 文件系统是可选的。

### 3. 文档不足

NetBSD 的某些 API 文档较少。

## 总结

libsigar NetBSD 实现充分利用了 NetBSD 的 `sysctl`、`kvm` 库和扩展的进程信息结构，提供了全面的系统监控功能。其设计特点包括：

1. **简洁设计**: 代码结构清晰，易于理解和维护
2. **扩展结构**: 使用 kinfo_proc2 等扩展结构提供更多信息
3. **双模式**: 同时支持 sysctl 和 kvm 两种方式
4. **可移植性**: 良好的可移植性设计

这种实现方式展示了 NetBSD 系统架构的特点，强调了简洁性和可移植性。
