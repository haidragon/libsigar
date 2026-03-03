# OpenBSD 分支源码分析

## 概述

OpenBSD 是一个专注于安全性、代码正确性和可移植性的免费开源 UNIX 操作系统。libsigar 在 OpenBSD 平台的实现与 FreeBSD 有很高的相似性,主要使用 `sysctl` 系统调用和 `kvm` 库来获取系统信息,并提供了 `/proc` 文件系统的可选支持。

## 核心架构

### 1. sigar_t 结构体扩展

```c
struct sigar_t {
    SIGAR_T_BASE;              // 基础结构
    int pagesize;              // 页面大小
    time_t last_getprocs;      // 最后一次获取进程信息的时间
    sigar_pid_t last_pid;      // 最后查询的 PID
    bsd_pinfo_t *pinfo;        // 进程信息缓存 (kinfo_proc)
    int lcpu;                  // 最后访问的 CPU
    size_t argmax;             // 参数最大长度
    kvm_t *kmem;               // kvm 库句柄
    unsigned long koffsets[KOFFSET_MAX];  // 内核符号偏移量
    int proc_mounted;          // /proc 是否挂载
};
```

### 2. 内核符号偏移量枚举

```c
enum {
    KOFFSET_CPUINFO,    // cp_time
    KOFFSET_VMMETER,    // cnt
    KOFFSET_TCPSTAT,    // tcpstat
    KOFFSET_TCBTABLE,   // tcbtable
    KOFFSET_MAX
};
```

这些偏移量通过 `kvm_nlist` 从内核符号表中解析,用于直接读取内核数据。

### 3. 核心数据类型

```c
typedef struct kinfo_proc bsd_pinfo_t;
```

使用标准的 BSD 进程信息结构体。

## 核心技术

### 1. sysctl 系统调用

OpenBSD 主要通过 sysctl 获取系统信息:

```c
int mib[2];
size_t len;

// 获取 CPU 数量
mib[0] = CTL_HW;
mib[1] = HW_NCPU;
sysctl(mib, NMIB(mib), &ncpu, &len, NULL, 0);

// 获取内存大小
mib[1] = HW_PHYSMEM;
sysctl(mib, NMIB(mib), &mem_total, &len, NULL, 0);

// 获取页面大小
mib[1] = HW_PAGESIZE;
sysctl(mib, NMIB(mib), &pagesize, &len, NULL, 0);

// 获取启动时间
mib[0] = CTL_KERN;
mib[1] = KERN_BOOTTIME;
sysctl(mib, NMIB(mib), &boottime, &len, NULL, 0);
```

### 2. kvm 库内核访问

用于访问内核内存和获取进程信息:

```c
kvm_t *kmem = kvm_openfiles(NULL, NULL, NULL, KVM_NO_FILES, NULL);

// 获取所有进程
struct kinfo_proc *proc = kvm_getprocs(kmem, KERN_PROC_ALL, 0,
                                       sizeof(*proc), &num);

// 读取内核数据
kvm_read(kmem, offset, data, size);
```

### 3. 内核符号解析

```c
static int get_koffsets(sigar_t *sigar)
{
    struct nlist klist[] = {
        { "_cp_time" },     // CPU 时间
        { "_cnt" },         // 虚拟内存统计
        { "_tcpstat" },     // TCP 统计
        { "_tcbtable" },    // TCP 控制块表
        { NULL }
    };

    kvm_nlist(sigar->kmem, klist);

    for (i = 0; i < KOFFSET_MAX; i++) {
        sigar->koffsets[i] = klist[i].n_value;
    }
}
```

### 4. 内核数据读取

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

### 5. 进程信息缓存

```c
static int sigar_get_pinfo(sigar_t *sigar, sigar_pid_t pid)
{
    int mib[] = { CTL_KERN, KERN_PROC, KERN_PROC_PID, 0,
                  sizeof(*sigar->pinfo), 1 };
    size_t len = sizeof(*sigar->pinfo);
    time_t timenow = time(NULL);
    mib[3] = pid;

    if (sigar->pinfo == NULL) {
        sigar->pinfo = malloc(len);
    }

    // 60 秒缓存
    if (sigar->last_pid == pid) {
        if ((timenow - sigar->last_getprocs) < SIGAR_LAST_PROC_EXPIRE) {
            return SIGAR_OK;
        }
    }

    sigar->last_pid = pid;
    sigar->last_getprocs = timenow;

    return sysctl(mib, NMIB(mib), sigar->pinfo, &len, NULL, 0);
}
```

### 6. /proc 可选支持

```c
#define PROCFS_STATUS(status) \
    ((((status) != SIGAR_OK) && !sigar->proc_mounted) ? \
     SIGAR_ENOTIMPL : status)

int sigar_os_open(sigar_t **sigar)
{
    struct stat sb;

    if (stat("/proc/curproc", &sb) < 0) {
        (*sigar)->proc_mounted = 0;
    } else {
        (*sigar)->proc_mounted = 1;
    }
}
```

## 核心功能模块

### 1. 内存监控

```c
int sigar_mem_get(sigar_t *sigar, sigar_mem_t *mem)
{
    int mib[2];
    size_t len;
    struct uvmexp vmstat;
    sigar_uint64_t kern = 0;

    // 页面大小
    mib[0] = CTL_HW;
    mib[1] = HW_PAGESIZE;
    sysctl(mib, NMIB(mib), &sigar->pagesize, &len, NULL, 0);

    // 物理内存总量
    mib[1] = HW_PHYSMEM;
    sysctl(mib, NMIB(mib), &mem_total, &len, NULL, 0);
    mem->total = mem_total;

    // 虚拟内存统计
    sigar_vmstat(sigar, &vmstat);

    mem->free = vmstat.free;
    kern = vmstat.inactive;
    kern += vmstat.vnodepages + vmstat.vtextpages;
    kern *= sigar->pagesize;

    mem->used = mem->total - mem->free;
    mem->actual_free = mem->free + kern;
    mem->actual_used = mem->used - kern;

    sigar_mem_calc_ram(sigar, mem);
}
```

### 2. 交换空间监控

```c
int sigar_swap_get(sigar_t *sigar, sigar_swap_t *swap)
{
    struct uvmexp vmstat;

    sigar_vmstat(sigar, &vmstat);

    swap->total = vmstat.swpages * sigar->pagesize;
    swap->used = vmstat.swpginuse * sigar->pagesize;
    swap->free = swap->total - swap->used;
    swap->page_in = vmstat.pageins;
    swap->page_out = vmstat.pdpageouts;
}
```

### 3. CPU 监控

#### 总体 CPU 统计

```c
int sigar_cpu_get(sigar_t *sigar, sigar_cpu_t *cpu)
{
    cp_time_t cp_time[CPUSTATES];
    int mib[] = { CTL_KERN, KERN_CPTIME };
    size_t size = sizeof(cp_time);

    sysctl(mib, NMIB(mib), &cp_time, &size, NULL, 0);

    cpu->user = SIGAR_TICK2MSEC(cp_time[CP_USER]);
    cpu->nice = SIGAR_TICK2MSEC(cp_time[CP_NICE]);
    cpu->sys  = SIGAR_TICK2MSEC(cp_time[CP_SYS]);
    cpu->idle = SIGAR_TICK2MSEC(cp_time[CP_IDLE]);
    cpu->wait = 0;  // N/A
    cpu->irq = SIGAR_TICK2MSEC(cp_time[CP_INTR]);
    cpu->soft_irq = 0;  // N/A
    cpu->stolen = 0;  // N/A
    cpu->total = cpu->user + cpu->nice + cpu->sys + cpu->idle + cpu->irq;
}
```

#### CPU 列表

```c
int sigar_cpu_list_get(sigar_t *sigar, sigar_cpu_list_t *cpulist)
{
    sigar_cpu_list_create(cpulist);

    #ifdef HAVE_KERN_CP_TIMES
        // 支持每个 CPU 的统计
        if (sigar_cp_times_get(sigar, cpulist) == SIGAR_OK) {
            return SIGAR_OK;
        }
    #endif

    // 旧版本: 所有指标在第 1 个 CPU,其余为 0
    sigar_cpu_get(sigar, &cpulist->data[cpulist->number++]);

    for (i = 1; i < sigar->ncpu; i++) {
        SIGAR_CPU_LIST_GROW(cpulist);
        cpu = &cpulist->data[cpulist->number++];
        SIGAR_ZERO(cpu);
    }
}
```

### 4. 进程监控

#### 进程列表

```c
int sigar_os_proc_list_get(sigar_t *sigar, sigar_proc_list_t *proclist)
{
    struct kinfo_proc *proc;
    int num;

    if (!sigar->kmem) {
        return SIGAR_EPERM_KMEM;
    }

    proc = kvm_getprocs(sigar->kmem, KERN_PROC_ALL, 0,
                       sizeof(*proc), &num);

    for (i = 0; i < num; i++) {
        // 跳过系统进程和 PID 0
        if (proc[i].KI_FLAG & P_SYSTEM) {
            continue;
        }
        if (proc[i].KI_PID == 0) {
            continue;
        }

        SIGAR_PROC_LIST_GROW(proclist);
        proclist->data[proclist->number++] = proc[i].KI_PID;
    }
}
```

#### 进程内存

```c
int sigar_proc_mem_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_mem_t *procmem)
{
    bsd_pinfo_t *pinfo = sigar->pinfo;

    sigar_get_pinfo(sigar, pid);

    // 虚拟内存大小 = text + data + stack
    procmem->size =
        (pinfo->p_vm_tsize + pinfo->p_vm_dsize + pinfo->p_vm_ssize) *
        sigar->pagesize;

    procmem->resident = pinfo->p_vm_rssize * sigar->pagesize;
    procmem->share = SIGAR_FIELD_NOTIMPL;

    procmem->minor_faults = pinfo->p_uru_minflt;
    procmem->major_faults = pinfo->p_uru_majflt;
    procmem->page_faults = procmem->minor_faults + procmem->major_faults;
}
```

#### 进程凭据

```c
int sigar_proc_cred_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_cred_t *proccred)
{
    bsd_pinfo_t *pinfo = sigar->pinfo;

    sigar_get_pinfo(sigar, pid);

    proccred->uid  = pinfo->p_ruid;
    proccred->gid  = pinfo->p_rgid;
    proccred->euid = pinfo->p_uid;
    proccred->egid = pinfo->p_gid;
}
```

#### 进程时间

```c
#define tv2msec(tv) \
    (((sigar_uint64_t)tv.tv_sec * SIGAR_MSEC) + (((sigar_uint64_t)tv.tv_usec) / 1000))

int sigar_proc_time_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_time_t *proctime)
{
    bsd_pinfo_t *pinfo = sigar->pinfo;

    sigar_get_pinfo(sigar, pid);

    proctime->user  = tv2msec(pinfo->p_uutime);
    proctime->sys   = tv2msec(pinfo->p_stime);
    proctime->total = proctime->user + proctime->sys;
    proctime->start_time = tv2msec(pinfo->p_ustart);
}
```

#### 进程状态

```c
int sigar_proc_state_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_state_t *procstate)
{
    bsd_pinfo_t *pinfo = sigar->pinfo;

    sigar_get_pinfo(sigar, pid);

    SIGAR_SSTRCPY(procstate->name, pinfo->p_comm);
    procstate->ppid = pinfo->p_ppid;
    procstate->priority = pinfo->p_priority;
    procstate->nice = pinfo->p_nice;
    procstate->processor = pinfo->p_cpuid;

    // 状态映射
    switch (pinfo->p_stat) {
        case SIDL:    procstate->state = 'D'; break;
        case SRUN:    procstate->state = 'R'; break;
        case SSLEEP:  procstate->state = 'S'; break;
        case SSTOP:   procstate->state = 'T'; break;
        case SZOMB:   procstate->state = 'Z'; break;
        case SDEAD:   procstate->state = 'Z'; break;
    }
}
```

#### 进程参数

OpenBSD 使用自定义的 `sigar_proc_args_get` 实现来获取进程参数。

## 技术亮点

### 1. 统一的 sysctl 接口

- 所有系统信息通过 sysctl 获取
- 结构化的 mib 数组
- 高效的二进制接口

### 2. kvm 库内核访问

- 直接访问内核内存
- 获取完整的进程信息
- 内核符号解析支持

### 3. /proc 可选支持

- 检测 `/proc` 是否挂载
- 在失败时优雅降级
- 适应不同系统配置

### 4. 进程信息缓存

- 60 秒有效期
- 减少 sysctl 调用
- 提高性能

### 5. 版本兼容性

- `KERN_CPTIME` vs `KERN_CP_TIME`
- `HAVE_KERN_CP_TIMES` 检测
- 新旧 API 兼容

### 6. 与 FreeBSD 共享头文件

```c
typedef struct kinfo_proc bsd_pinfo_t;
```

大量复用 FreeBSD 的代码和结构定义。

## 错误处理

### 自定义错误码

```c
#define SIGAR_EPERM_KMEM (SIGAR_OS_START_ERROR+EACCES)
#define SIGAR_EPROC_NOENT (SIGAR_OS_START_ERROR+2)
```

### 错误字符串

```c
char *sigar_os_error_string(sigar_t *sigar, int err)
{
    switch (err) {
        case SIGAR_EPERM_KMEM:
            return "Failed to open /dev/kmem for reading";
        case SIGAR_EPROC_NOENT:
            return "/proc filesystem is not mounted";
        default:
            return NULL;
    }
}
```

## 未实现功能

以下功能在 OpenBSD 上未实现:

- `sigar_sys_info_get_uuid` - 系统 UUID
- `sigar_system_stats_get` - 系统统计
- `sigar_proc_cumulative_disk_io_get` - 进程磁盘 I/O
- `sigar_proc_env_get` - 进程环境变量
- `sigar_proc_fd_get` - 进程文件描述符
- `sigar_proc_exe_get` - 进程可执行文件

## 与 FreeBSD 的区别

| 特性 | OpenBSD | FreeBSD |
|------|---------|---------|
| 进程结构 | kinfo_proc | kinfo_proc |
| 内核访问 | kvm | kvm |
| /proc 支持 | 可选 | 完整支持 |
| sysctl 扩展 | KERN_CPTIME | KERN_CPTIME |
| 进程过滤 | P_SYSTEM 标志 | 无特定过滤 |
| 共享内存 | 自定义实现 | 更完整 |

## 总结

OpenBSD 分支的实现特点:

1. **安全导向** - 严格的访问控制和错误处理
2. **sysctl 为主** - 主要通过 sysctl 获取信息
3. **kvm 辅助** - 使用 kvm 获取进程信息
4. **兼容性强** - 与 FreeBSD 共享大量代码
5. **/proc 可选** - 不依赖 /proc 文件系统
6. **简洁设计** - 代码清晰,易于维护
7. **性能优化** - 进程信息缓存减少系统调用

OpenBSD 的实现体现了该系统对安全性和代码正确性的重视,同时也保持了与其他 BSD 系统的高度兼容性。
