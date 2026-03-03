# libsigar AIX 平台源码分析

## 概述

libsigar 在 AIX (IBM Advanced Interactive Executive) 平台的实现主要通过 `libperfstat` 库和 `/dev/kmem` 设备获取系统信息。AIX 是 IBM 的 UNIX 变体，具有独特的系统架构和 API 设计。

## 核心架构

### sigar_t 结构体 (sigar_os.h:49-67)

```c
struct sigar_t {
    SIGAR_T_BASE;
    int kmem;                              // /dev/kmem 文件描述符
    long koffsets[KOFFSET_MAX];            // 内核符号偏移量
    proc_fd_func_t getprocfd;             // 获取进程文件描述符函数
    int pagesize;                          // 页大小位移值
    swaps_t swaps;                         // 交换空间信息
    time_t last_getprocs;                  // 最后获取进程时间
    sigar_pid_t last_pid;                  // 最后处理的 PID
    struct procsinfo64 *pinfo;             // 进程信息缓冲区
    struct cpuinfo *cpuinfo;               // CPU 信息缓冲区
    int cpuinfo_size;                      // CPU 信息大小
    int cpu_mhz;                           // CPU 频率 (MHz)
    char model[128];                        // CPU 型号
    int aix_version;                       // AIX 版本号
    int thrusage;                          // 线程资源使用标志
    sigar_cache_t *diskmap;                // 磁盘映射缓存
};
```

### swaps_t 结构体 (sigar_os.h:41-45)

```c
typedef struct {
    time_t mtime;    // /etc/swapspaces 修改时间
    int num;         // 交换设备数量
    char **devs;     // 交换设备路径数组
} swaps_t;
```

### 内核符号偏移量 (sigar_os.h:28-39)

```c
enum {
    KOFFSET_LOADAVG,   // 负载平均值
    KOFFSET_VAR,       // 内核变量
    KOFFSET_SYSINFO,   // 系统信息
    KOFFSET_IFNET,     // 网络接口
    KOFFSET_VMINFO,    // 虚拟内存信息
    KOFFSET_CPUINFO,   // CPU 信息
    KOFFSET_TCB,       // TCP 控制块
    KOFFSET_ARPTABSIZE,// ARP 表大小
    KOFFSET_ARPTABP,   // ARP 表指针
    KOFFSET_MAX
};
```

## 核心技术

### 1. libperfstat 库

libperfstat 是 AIX 提供的性能统计库，提供了访问各种系统性能数据的 API。

#### 主要 perfstat API

```c
// 内存统计
perfstat_memory_total_t memory;
perfstat_memory_total(NULL, &memory, sizeof(memory), 1);

// CPU 统计
perfstat_cpu_total_t cpu_total;
perfstat_cpu_total(NULL, &cpu_total, sizeof(cpu_total), 1);

// 交换空间统计
perfstat_pagingspace_t pspace;
perfstat_id_t id;
id.name[0] = '\0';
perfstat_pagingspace(&id, &pspace, sizeof(pspace), 1);
```

### 2. /dev/kmem 访问

当 libperfstat 不可用时，通过读取 `/dev/kmem` 设备直接访问内核内存。

#### 内核符号查找 (aix_sigar.c:101-130)

```c
static int get_koffsets(sigar_t *sigar)
{
    struct nlist klist[] = {
        {"avenrun", 0, 0, 0, 0, 0},       // 负载平均值
        {"v", 0, 0, 0, 0, 0},            // 内核变量
        {"sysinfo", 0, 0, 0, 0, 0},       // 系统信息
        {"ifnet", 0, 0, 0, 0, 0},        // 网络接口
        {"vmminfo", 0, 0, 0, 0, 0},       // 虚拟内存信息
        {"cpuinfo", 0, 0, 0, 0, 0},      // CPU 信息
        {"tcb", 0, 0, 0, 0, 0},          // TCP 控制块
        {"arptabsize", 0, 0, 0, 0, 0},   // ARP 表大小
        {"arptabp", 0, 0, 0, 0, 0},      // ARP 表指针
        {NULL, 0, 0, 0, 0, 0}
    };

    if (knlist(klist, sizeof(klist)/sizeof(klist[0]),
               sizeof(klist[0])) != 0) {
        return errno;
    }

    for (i=0; i<KOFFSET_MAX; i++) {
        sigar->koffsets[i] = klist[i].n_value;
    }

    return SIGAR_OK;
}
```

#### 内核内存读取 (aix_sigar.c:132-147)

```c
static int kread(sigar_t *sigar, void *data, int size, long offset)
{
    if (sigar->kmem < 0) {
        return SIGAR_EPERM_KMEM;
    }

    if (lseek(sigar->kmem, offset, SEEK_SET) != offset) {
        return errno;
    }

    if (read(sigar->kmem, data, size) != size) {
        return errno;
    }

    return SIGAR_OK;
}
```

### 3. 初始化流程 (aix_sigar.c:164-211)

```c
int sigar_os_open(sigar_t **sigar)
{
    int kmem = -1;
    struct utsname name;

    // 1. 打开 /dev/kmem
    kmem = open("/dev/kmem", O_RDONLY);

    // 2. 分配 sigar 结构
    *sigar = malloc(sizeof(**sigar));

    (*sigar)->kmem = kmem;
    (*sigar)->pagesize = 0;
    (*sigar)->ticks = sysconf(_SC_CLK_TCK);
    (*sigar)->boot_time = 0;

    // 3. 计算页大小位移值
    i = getpagesize();
    while ((i >>= 1) > 0) {
        (*sigar)->pagesize++;
    }

    // 4. 尝试获取内核符号偏移量
    if (kmem > 0) {
        if ((status = get_koffsets(*sigar)) != SIGAR_OK) {
            // libperfstat only mode (aix 6)
            close((*sigar)->kmem);
            (*sigar)->kmem = -1;
        }
    }

    // 5. 获取 AIX 版本
    uname(&name);
    (*sigar)->aix_version = atoi(name.version);

    // 6. 设置线程资源使用模式
    (*sigar)->thrusage = PTHRDSINFO_RUSAGE_STOP;

    return SIGAR_OK;
}
```

## 核心功能模块

### 1. 内存监控

#### 内存信息获取 (aix_sigar.c:257-279)

```c
int sigar_mem_get(sigar_t *sigar, sigar_mem_t *mem)
{
    perfstat_memory_total_t minfo;
    sigar_uint64_t kern;

    if (sigar_perfstat_memory(&minfo) == 1) {
        mem->total = PAGESHIFT(minfo.real_total);     // 物理内存总页数
        mem->free  = PAGESHIFT(minfo.real_free);      // 空闲页数
        kern = PAGESHIFT(minfo.numperm);              // 文件缓存页数
    }

    mem->used = mem->total - mem->free;
    mem->actual_used = mem->used - kern;             // 实际使用 (不含缓存)
    mem->actual_free = mem->free + kern;             // 实际可用 (含缓存)

    sigar_mem_calc_ram(sigar, mem);

    return SIGAR_OK;
}
```

#### 交换空间信息 (aix_sigar.c:430-476)

```c
int sigar_swap_get(sigar_t *sigar, sigar_swap_t *swap)
{
    perfstat_pagingspace_t ps;
    perfstat_id_t id;

    id.name[0] = '\0';

    SIGAR_ZERO(swap);

    do {
        // 遍历所有交换空间
        if (perfstat_pagingspace(&id, &ps, sizeof(ps), 1) != 1) {
            continue;
        }

        if (ps.active != 1) {
            continue;  // 跳过非活动的交换空间
        }

        // 转换 MB 为字节
        swap->total += SWAP_MB_TO_BYTES(ps.mb_size);
        swap->used  += SWAP_MB_TO_BYTES(ps.mb_used);
    } while (id.name[0] != '\0');

    swap->free = swap->total - swap->used;

    // 获取分页统计
    if (sigar_perfstat_memory(&minfo) == 1) {
        swap->page_in = minfo.pgins;
        swap->page_out = minfo.pgouts;
    }

    return SIGAR_OK;
}
```

#### 交换空间配置解析 (aix_sigar.c:304-365)

```c
// 解析 /etc/swapspaces 文件
static int swaps_get(swaps_t *swaps)
{
    FILE *fp;
    char buf[512];
    char *ptr;
    struct stat statbuf;

    if (stat(SWAPSPACES, &statbuf) < 0) {
        return errno;
    }

    // 文件未修改则使用缓存
    if (swaps->mtime == statbuf.st_mtime) {
        return 0;
    }

    swaps->mtime = statbuf.st_mtime;
    swaps_free(swaps);

    if (!(fp = fopen(SWAPSPACES, "r"))) {
        return errno;
    }

    while ((ptr = fgets(buf, sizeof(buf), fp))) {
        if (!isalpha(*ptr)) {
            continue;
        }

        // 格式:
        // hd6:
        //    dev = /dev/hd6

        if (strchr(ptr, ':')) {
            int len;

            ptr = fgets(buf, sizeof(buf), fp);

            while (isspace(*ptr)) ++ptr;

            if (strncmp(ptr, "dev", 3)) {
                continue;
            }
            ptr += 3;
            while (isspace(*ptr) || (*ptr == '=')) ++ptr;

            len = strlen(ptr);
            ptr[len-1] = '\0';  // 去掉换行符

            swaps->devs = realloc(swaps->devs,
                                  (swaps->num+1) * sizeof(char *));
            swaps->devs[swaps->num] = malloc(len);
            memcpy(swaps->devs[swaps->num], ptr, len);

            swaps->num++;
        }
    }

    fclose(fp);
    return 0;
}
```

### 2. CPU 监控

#### CPU 统计 (aix_sigar.c:478-497)

```c
int sigar_cpu_get(sigar_t *sigar, sigar_cpu_t *cpu)
{
    perfstat_cpu_total_t cpu_data;

    if (sigar_perfstat_cpu(&cpu_data) == 1) {
        cpu->user  = SIGAR_TICK2MSEC(cpu_data.user);   // 用户时间
        cpu->nice  = SIGAR_FIELD_NOTIMPL;               // N/A
        cpu->sys   = SIGAR_TICK2MSEC(cpu_data.sys);    // 系统时间
        cpu->idle  = SIGAR_TICK2MSEC(cpu_data.idle);   // 空闲时间
        cpu->wait  = SIGAR_TICK2MSEC(cpu_data.wait);   // 等待 I/O
        cpu->irq = 0;                                  // N/A
        cpu->soft_irq = 0;                             // N/A
        cpu->stolen = 0;                               // N/A
        cpu->total = cpu->user + cpu->sys + cpu->idle + cpu->wait;

        return SIGAR_OK;
    }
    else {
        return errno;
    }
}
```

### 3. 进程监控

AIX 进程监控使用 `getprocs`/`getprocs64` 系统调用。

#### 进程列表获取

```c
int sigar_os_proc_list_get(sigar_t *sigar, sigar_proc_list_t *proclist)
{
    int num, i;
    struct procsinfo64 *pinfo;

    // 获取进程列表
    num = getprocs64(pinfo, sizeof(struct procsinfo64),
                     NULL, 0, &pid, 1);

    for (i=0; i<num; i++) {
        // 处理每个进程
        ...
    }

    return SIGAR_OK;
}
```

#### 进程内存

```c
int sigar_proc_mem_get(sigar_t *sigar, sigar_pid_t pid,
                       sigar_proc_mem_t *procmem)
{
    struct procentry64 procentry;

    if (getprocs64(&procentry, sizeof(procentry),
                   NULL, 0, &pid, 1) != 1) {
        return errno;
    }

    procmem->size = PAGESHIFT(procentry.pi_size);
    procmem->resident = PAGESHIFT(procentry.pi_rssize);
    procmem->share = PAGESHIFT(procentry.pi_msize);

    procmem->minor_faults = SIGAR_FIELD_NOTIMPL;
    procmem->major_faults = SIGAR_FIELD_NOTIMPL;
    procmem->page_faults = SIGAR_FIELD_NOTIMPL;

    return SIGAR_OK;
}
```

### 4. 网络监控

#### 网络接口统计

AIX 使用 `getkerninfo` 系统调用获取网络统计信息。

```c
int sigar_net_interface_stat_get(sigar_t *sigar, const char *name,
                                 sigar_net_interface_stat_t *ifstat)
{
    int size;
    struct ifnet ifnet;
    char *data, *ptr;

    // 获取网络接口信息
    size = getkerninfo(KINFO_IFNET, &data, 0, 0);

    if (size <= 0) {
        return errno;
    }

    ptr = data;
    while (ptr < data + size) {
        memcpy(&ifnet, ptr, sizeof(ifnet));

        if (strcmp(ifnet.if_name, name) == 0) {
            ifstat->rx_packets = ifnet.if_ipackets;
            ifstat->tx_packets = ifnet.if_opackets;
            ifstat->rx_bytes = ifnet.if_ibytes;
            ifstat->tx_bytes = ifnet.if_obytes;
            ifstat->rx_errors = ifnet.if_ierrors;
            ifstat->tx_errors = ifnet.if_oerrors;
            break;
        }

        ptr += ifnet.if_size;
    }

    free(data);
    return SIGAR_OK;
}
```

### 5. 文件系统监控

#### 文件系统列表

AIX 使用 `mntctl` 系统调用获取挂载的文件系统。

```c
int sigar_file_system_list_get(sigar_t *sigar,
                               sigar_file_system_list_t *fslist)
{
    int size, count, i;
    char *data;
    struct vmount *vmount;

    // 获取需要的缓冲区大小
    size = mntctl(MCTL_QUERY, sizeof(size), &size);

    data = malloc(size);

    // 获取挂载点信息
    count = mntctl(MCTL_QUERY, size, data);

    vmount = (struct vmount *)data;

    for (i=0; i<count; i++) {
        fsp = &fslist->data[fslist->number++];

        fsp->dir_name = vmt2str(vmount, VMT_STUB);
        fsp->dev_name = vmt2str(vmount, VMT_OBJECT);
        fsp->type_name = vmt2str(vmount, VMT_FSTYPE);

        vmount = (struct vmount *)((char *)vmount + vmount->vmt_length);
    }

    free(data);
    return SIGAR_OK;
}
```

## 性能优化

### 1. 交换空间缓存 (aix_sigar.c:315-318)

```c
// 文件未修改则使用缓存
if (swaps->mtime == statbuf.st_mtime) {
    return 0;
}
```

### 2. perfstat 优先使用

优先使用 `libperfstat` API，只有在不可用时才使用 `/dev/kmem`。

### 3. 批量处理

使用 `getprocs64` 批量获取进程信息，减少系统调用次数。

## 特殊技术

### 1. 页大小位移 (aix_sigar.c:254-255)

```c
#define PAGESHIFT(v) \
    ((v) << sigar->pagesize)
```

将页数转换为字节数，使用位移代替乘法。

### 2. 固定点数转换 (aix_sigar.c:92-97)

```c
/*
 * from libperfstat.h:
 * "To calculate the load average, divide the numbers by (1<<SBITS).
 *  SBITS is defined in <sys/proc.h>."
 */
#define FIXED_TO_DOUBLE(x) (((double)x) / (1<<SBITS))
```

AIX 使用固定位点数表示负载平均值。

### 3. swapqry 函数 (aix_sigar.c:373)

```c
// 文档中提到但未在头文件中定义的函数
int swapqry(char *path, struct pginfo *info);
```

直接调用未公开的内核函数获取交换空间信息。

### 4. pthread_getrusage_np (aix_sigar.c:149-152)

```c
static int sigar_thread_rusage(struct rusage *usage, int mode)
{
    return pthread_getrusage_np(pthread_self(), usage, mode);
}
```

AIX 特有的线程资源使用查询函数。

## 技术亮点

### 1. libperfstat 统一接口

`libperfstat` 提供了统一的性能数据访问接口，涵盖了 CPU、内存、磁盘、网络等各个方面。

### 2. 双模式支持

同时支持 `libperfstat` 和 `/dev/kmem` 两种模式，确保在各种 AIX 版本上都能工作。

### 3. 内核符号查询

使用 `knlist` 查询内核符号地址，直接访问内核数据结构。

### 4. 进程信息批量获取

使用 `getprocs64` 一次性获取多个进程的信息，提高效率。

## 限制和挑战

### 1. 权限要求

访问 `/dev/kmem` 需要 root 权限或特定权限。

### 2. 未公开 API

某些功能需要调用未公开的 API (如 `swapqry`)，可能在不同版本中失效。

### 3. 文档不足

AIX 的某些 API 文档不完整，需要通过逆向工程或经验学习。

### 4. 版本差异

不同 AIX 版本之间存在 API 差异，需要兼容性处理。

## 总结

libsigar AIX 实现充分利用了 AIX 的 `libperfstat` 库和内核访问机制，提供了全面的系统监控功能。其设计特点包括：

1. **统一接口**: libperfstat 提供了统一的性能数据访问接口
2. **双模式**: 同时支持 libperfstat 和 /dev/kmem 两种模式
3. **高效批量**: 使用批量 API 减少系统调用次数
4. **版本兼容**: 支持多种 AIX 版本
5. **内存优化**: 使用缓存减少重复读取

这种实现方式展示了 AIX 系统架构的特点，为其他 UNIX 变体的系统监控提供了参考。
