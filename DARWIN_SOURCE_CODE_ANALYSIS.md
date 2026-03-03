# libsigar Darwin/macOS 分支源码深度解析

## 目录
1. [概述](#概述)
2. [核心架构](#核心架构)
3. [模块组织](#模块组织)
4. [关键技术实现](#关键技术实现)
5. [API调用分析](#api调用分析)
6. [核心功能模块](#核心功能模块)
7. [性能优化机制](#性能优化机制)
8. [特殊技术](#特殊技术)
9. [版本兼容性处理](#版本兼容性处理)

---

## 概述

libsigar 的 Darwin/macOS 分支实现了一套完整的跨平台系统信息收集接口。Darwin 实现主要通过 Mach 内核接口、sysctl 系统调用、I/O Kit 框架以及 libproc 库来获取系统信息。

### 关键文件结构

```
src/os/darwin/
├── darwin_sigar.c       # 主要实现文件 (71.3KB, 1800+行)
├── sigar_os.h          # Darwin特定头文件定义
├── if_types.h           # 网络接口类型定义 (iOS兼容)
├── tcp_fsm.h            # TCP状态机定义 (iOS兼容)
└── Info.plist.in        # Info.plist模板
```

### 平台特性

Darwin 实现支持以下特性：
- **macOS**: 完整功能支持
- **iPhone OS (iOS)**: 通过 `TARGET_OS_IPHONE` 宏提供部分功能支持
- **多版本兼容**: 支持从 Mac OS X 10.3 到最新版本

---

## 核心架构

### 1. sigar_t 结构体 (Darwin 扩展)

```c
struct sigar_t {
    SIGAR_T_BASE;              // 基础字段
    int pagesize;              // 页面大小
    time_t last_getprocs;      // 最后获取进程信息时间
    sigar_pid_t last_pid;      // 最后查询的PID
    bsd_pinfo_t *pinfo;        // 进程信息缓存
    int lcpu;                  // 逻辑CPU数量
    size_t argmax;             // 参数最大长度

    // Mach 端口
    mach_port_t mach_port;     // Mach宿主端口

#ifdef DARWIN_HAS_LIBPROC_H
    // libproc 库函数指针
    void *libproc;             // libproc.dylib 句柄
    proc_pidinfo_func_t proc_pidinfo;      // 获取进程信息
    proc_pidfdinfo_func_t proc_pidfdinfo; // 获取文件描述符信息
#endif
};
```

### 2. BSD 进程信息结构

```c
typedef struct kinfo_proc bsd_pinfo_t;

// 进程信息访问宏
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
#define KI_START kp_proc.p_starttime
```

### 3. Mach 内核接口

Darwin 实现大量使用 Mach 内核接口获取底层系统信息：

| Mach 接口 | 用途 |
|-----------|------|
| `host_statistics()` | 获取主机统计信息 (CPU、内存) |
| `host_processor_info()` | 获取处理器信息 |
| `task_for_pid()` | 获取进程的 Mach 端口 |
| `task_info()` | 获取任务信息 |
| `task_threads()` | 获取线程列表 |
| `thread_info()` | 获取线程信息 |

---

## 模块组织

### 1. 初始化和清理

#### sigar_os_open() - 初始化函数

```c
int sigar_os_open(sigar_t **sigar) {
    int mib[2];
    int ncpu;
    size_t len;
    struct timeval boottime;

    // 获取 CPU 数量
    len = sizeof(ncpu);
    mib[0] = CTL_HW;
    mib[1] = HW_NCPU;
    if (sysctl(mib, NMIB(mib), &ncpu,  &len, NULL, 0) < 0) {
        return errno;
    }

    // 获取系统启动时间
    len = sizeof(boottime);
    mib[0] = CTL_KERN;
    mib[1] = KERN_BOOTTIME;
    if (sysctl(mib, NMIB(mib), &boottime, &len, NULL, 0) < 0) {
        return errno;
    }

    *sigar = malloc(sizeof(**sigar));

    // 创建 Mach 宿主端口
    (*sigar)->mach_port = mach_host_self();

#ifdef DARWIN_HAS_LIBPROC_H
    // 动态加载 libproc.dylib
    if (((*sigar)->libproc = dlopen("/usr/lib/libproc.dylib", 0))) {
        (*sigar)->proc_pidinfo = dlsym((*sigar)->libproc, "proc_pidinfo");
        (*sigar)->proc_pidfdinfo = dlsym((*sigar)->libproc, "proc_pidfdinfo");
    }
#endif

    (*sigar)->ncpu = ncpu;
    (*sigar)->lcpu = -1;
    (*sigar)->argmax = 0;
    (*sigar)->boot_time = boottime.tv_sec;

    (*sigar)->pagesize = getpagesize();
    (*sigar)->ticks = sysconf(_SC_CLK_TCK);
    (*sigar)->last_pid = -1;

    return SIGAR_OK;
}
```

#### sigar_os_close() - 清理函数

```c
int sigar_os_close(sigar_t *sigar) {
    if (sigar->pinfo) {
        free(sigar->pinfo);
    }
    free(sigar);
    return SIGAR_OK;
}
```

---

## 关键技术实现

### 1. sysctl 系统调用

sysctl 是 Darwin 获取系统信息的核心接口。

#### sysctl 使用模式

```c
#define NMIB(mib) (sizeof(mib)/sizeof(mib[0]))

// 获取硬件信息示例
int mib[2];
size_t len;

mib[0] = CTL_HW;
mib[1] = HW_MEMSIZE;
len = sizeof(mem_total);
sysctl(mib, NMIB(mib), &mem_total, &len, NULL, 0);
```

#### 主要 sysctl 键值

| 类别 | 键 | 说明 |
|-----|---|------|
| 硬件 | `HW_NCPU` | CPU 数量 |
| 硬件 | `HW_MEMSIZE` | 物理内存大小 |
| 硬件 | `HW_PAGESIZE` | 页面大小 |
| 硬件 | `HW_CPU_FREQ` | CPU 频率 |
| 硬件 | `HW_MODEL` | CPU 型号 |
| 硬件 | `HW_L2CACHESIZE` | L2 缓存大小 |
| 内核 | `KERN_BOOTTIME` | 系统启动时间 |
| 内核 | `KERN_PROC` | 进程信息 |
| 内核 | `KERN_ARGMAX` | 参数最大长度 |
| 内核 | `KERN_PROCARGS2` | 进程参数和环境 |
| 虚拟内存 | `VM_SWAPUSAGE` | 交换空间使用 (10.4+) |
| 网络 | `CTL_NET` | 网络相关 |

### 2. Mach VM 统计

#### 获取虚拟内存统计

```c
static int sigar_vmstat(sigar_t *sigar, vm_statistics_data_t *vmstat) {
    kern_return_t status;
    mach_msg_type_number_t count = sizeof(*vmstat) / sizeof(integer_t);

    status = host_statistics(sigar->mach_port, HOST_VM_INFO,
                             (host_info_t)vmstat, &count);

    if (status == KERN_SUCCESS) {
        return SIGAR_OK;
    }
    else {
        return errno;
    }
}
```

#### vm_statistics_data_t 结构

```c
typedef struct {
    integer_t free_count;        // 空闲页面数
    integer_t active_count;      // 活跃页面数
    integer_t inactive_count;    // 非活跃页面数
    integer_t wire_count;        // 固定页面数
    integer_t zero_fill_count;   // 填零操作数
    integer_t reactivations;     // 重新激活数
    integer_t pageins;           // 页面换入数
    integer_t pageouts;          // 页面换出数
    integer_t faults;            // 缺页错误数
    integer_t cow_faults;        // 写时复制错误数
    integer_t lookups;           // 页面查找数
    integer_t hits;              // 缓存命中数
} vm_statistics_data_t;
```

### 3. libproc 动态库

libproc 是 macOS 的进程信息库，提供高效的进程查询接口。

#### 动态加载 libproc

```c
#ifdef DARWIN_HAS_LIBPROC_H
    if (((*sigar)->libproc = dlopen("/usr/lib/libproc.dylib", 0))) {
        (*sigar)->proc_pidinfo = dlsym((*sigar)->libproc, "proc_pidinfo");
        (*sigar)->proc_pidfdinfo = dlsym((*sigar)->libproc, "proc_pidfdinfo");
    }
#endif
```

#### proc_pidinfo 使用示例

```c
struct proc_taskinfo pti;
int sz = sigar->proc_pidinfo(pid, PROC_PIDTASKINFO, 0, &pti, sizeof(pti));

if (sz == sizeof(pti)) {
    procmem->size         = pti.pti_virtual_size;
    procmem->resident     = pti.pti_resident_size;
    procmem->page_faults  = pti.pti_faults;
}
```

#### proc_pidinfo 命令

| 命令 | 说明 |
|-----|------|
| `PROC_PIDTASKINFO` | 任务信息（内存、时间） |
| `PROC_PIDREGIONINFO` | 内存区域信息 |
| `PROC_PIDLISTFDS` | 文件描述符列表 |
| `PROC_PIDFDVNODEINFO` | 文件描述符 vnode 信息 |

### 4. I/O Kit 框架

I/O Kit 是 macOS 的设备驱动框架，用于获取磁盘设备信息。

#### 获取磁盘 I/O 统计

```c
#define IoStatGetValue(key, val) \
    if ((number = (CFNumberRef)CFDictionaryGetValue(stats, CFSTR(kIOBlockStorageDriverStatistics##key)))) \
        CFNumberGetValue(number, kCFNumberSInt64Type, &val)

int sigar_disk_usage_get(sigar_t *sigar, const char *name, sigar_disk_usage_t *disk) {
    io_service_t service;
    io_registry_entry_t parent;
    CFDictionaryRef props;
    CFDictionaryRef stats;

    // 映射设备名到 I/O Kit 服务
    /* "/dev/disk0s1" -> "disk0" */
    service = IOServiceGetMatchingService(kIOMasterPortDefault,
                                          IOBSDNameMatching(kIOMasterPortDefault, 0, dname));

    // 获取父级条目（块存储驱动）
    IORegistryEntryGetParentEntry(service, kIOServicePlane, &parent);

    // 获取属性字典
    IORegistryEntryCreateCFProperties(parent, (CFMutableDictionaryRef *)&props,
                                     kCFAllocatorDefault, kNilOptions);

    // 获取统计信息
    stats = (CFDictionaryRef)CFDictionaryGetValue(props,
                                                  CFSTR(kIOBlockStorageDriverStatisticsKey));

    if (stats) {
        IoStatGetValue(ReadsKey, disk->reads);
        IoStatGetValue(BytesReadKey, disk->read_bytes);
        IoStatGetValue(TotalReadTimeKey, disk->rtime);
        IoStatGetValue(WritesKey, disk->writes);
        IoStatGetValue(BytesWrittenKey, disk->write_bytes);
        IoStatGetValue(TotalWriteTimeKey, disk->wtime);
        disk->time = disk->rtime + disk->wtime;
    }

    CFRelease(props);
    IOObjectRelease(service);
    IOObjectRelease(parent);
}
```

### 5. 路由表信息

#### 获取路由表

```c
int sigar_net_route_list_get(sigar_t *sigar, sigar_net_route_list_t *routelist) {
    size_t needed;
    char *buf, *next, *lim;
    struct rt_msghdr *rtm;
    int mib[6] = { CTL_NET, PF_ROUTE, 0, 0, NET_RT_DUMP, 0 };

    // 获取所需缓冲区大小
    if (sysctl(mib, NMIB(mib), NULL, &needed, NULL, 0) < 0) {
        return errno;
    }

    buf = malloc(needed);

    // 获取路由表数据
    if (sysctl(mib, NMIB(mib), buf, &needed, NULL, 0) < 0) {
        free(buf);
        return errno;
    }

    // 解析路由消息
    lim = buf + needed;
    for (next = buf; next < lim; next += rtm->rtm_msglen) {
        struct sockaddr *sa;
        sigar_net_route_t *route;
        rtm = (struct rt_msghdr *)next;

        if (rtm->rtm_type != RTM_GET) {
            continue;
        }

        sa = (struct sockaddr *)(rtm + 1);

        if (sa->sa_family != AF_INET) {
            continue;
        }

        route = &routelist->data[routelist->number++];
        route->flags = rtm->rtm_flags;
        if_indextoname(rtm->rtm_index, route->ifname);

        // 解析路由地址
        for (bit=RTA_DST; bit && ((char *)sa < lim); bit <<= 1) {
            if ((rtm->rtm_addrs & bit) == 0) continue;

            switch (bit) {
                case RTA_DST:
                    sigar_net_address_set(route->destination, rt_s_addr(sa));
                    break;
                case RTA_GATEWAY:
                    sigar_net_address_set(route->gateway, rt_s_addr(sa));
                    break;
                case RTA_NETMASK:
                    sigar_net_address_set(route->mask, rt_s_addr(sa));
                    break;
            }

            sa = (struct sockaddr *)((char *)sa + SA_SIZE(sa));
        }
    }

    free(buf);
}
```

---

## API调用分析

### 1. 内存信息获取

```c
int sigar_mem_get(sigar_t *sigar, sigar_mem_t *mem) {
    sigar_uint64_t kern = 0;
    vm_statistics_data_t vmstat;
    uint64_t mem_total;
    int mib[2];
    size_t len;

    // 获取页面大小
    mib[0] = CTL_HW;
    mib[1] = HW_PAGESIZE;
    sysctl(mib, NMIB(mib), &sigar->pagesize, &len, NULL, 0);

    // 获取总物理内存
    mib[1] = HW_MEMSIZE;
    sysctl(mib, NMIB(mib), &mem_total, &len, NULL, 0);
    mem->total = mem_total;

    // 获取 VM 统计
    sigar_vmstat(sigar, &vmstat);

    mem->free = vmstat.free_count * sigar->pagesize;
    kern = vmstat.inactive_count * sigar->pagesize;

    mem->used = mem->total - mem->free;

    // 实际可用 = 空闲 + 非活跃（缓存）
    mem->actual_free = mem->free + kern;
    mem->actual_used = mem->used - kern;

    sigar_mem_calc_ram(sigar, mem);
    return SIGAR_OK;
}
```

### 2. 交换空间获取

#### 方式一：sysctl (macOS 10.4+)

```c
static int sigar_swap_sysctl_get(sigar_t *sigar, sigar_swap_t *swap) {
#ifdef VM_SWAPUSAGE
    struct xsw_usage sw_usage;
    size_t size = sizeof(sw_usage);
    int mib[] = { CTL_VM, VM_SWAPUSAGE };

    if (sysctl(mib, NMIB(mib), &sw_usage, &size, NULL, 0) != 0) {
        return errno;
    }

    swap->total = sw_usage.xsu_total;
    swap->used = sw_usage.xsu_used;
    swap->free = sw_usage.xsu_avail;

    return SIGAR_OK;
#else
    return SIGAR_ENOTIMPL;
#endif
}
```

#### 方式二：文件系统扫描 (macOS 10.3 及以下)

```c
#define VM_DIR "/private/var/vm"
#define SWAPFILE "swapfile"

static int sigar_swap_fs_get(sigar_t *sigar, sigar_swap_t *swap) {
    DIR *dirp;
    struct dirent *ent;
    char swapfile[256];
    struct stat swapstat;
    struct statfs vmfs;

    swap->used = swap->total = swap->free = 0;

    // 扫描 /private/var/vm 目录
    if (!(dirp = opendir(VM_DIR))) {
        return errno;
    }

    // 查找 swapfile0, swapfile1 等
    while ((ent = readdir(dirp))) {
        if (!strnEQ(ent->d_name, SWAPFILE, SSTRLEN(SWAPFILE))) {
            continue;
        }

        sprintf(swapfile, "%s/%s", VM_DIR, ent->d_name);

        if (stat(swapfile, &swapstat) < 0) {
            continue;
        }

        swap->used += swapstat.st_size;
    }

    closedir(dirp);

    // 获取文件系统信息
    if (statfs(VM_DIR, &vmfs) < 0) {
        return errno;
    }

    swap->total = (vmfs.f_bfree * vmfs.f_bsize / 2) + swap->used;
    swap->free = swap->total - swap->used;

    return SIGAR_OK;
}
```

### 3. CPU 信息获取

```c
int sigar_cpu_get(sigar_t *sigar, sigar_cpu_t *cpu) {
    kern_return_t status;
    mach_msg_type_number_t count = HOST_CPU_LOAD_INFO_COUNT;
    host_cpu_load_info_data_t cpuload;

    status = host_statistics(sigar->mach_port, HOST_CPU_LOAD_INFO,
                             (host_info_t)&cpuload, &count);

    if (status != KERN_SUCCESS) {
        return errno;
    }

    cpu->user = SIGAR_TICK2MSEC(cpuload.cpu_ticks[CPU_STATE_USER]);
    cpu->sys  = SIGAR_TICK2MSEC(cpuload.cpu_ticks[CPU_STATE_SYSTEM]);
    cpu->idle = SIGAR_TICK2MSEC(cpuload.cpu_ticks[CPU_STATE_IDLE]);
    cpu->nice = SIGAR_TICK2MSEC(cpuload.cpu_ticks[CPU_STATE_NICE]);
    cpu->wait = 0;  /* 不适用 */
    cpu->irq = 0;   /* 不适用 */
    cpu->soft_irq = 0; /* 不适用 */
    cpu->stolen = 0;  /* 不适用 */
    cpu->total = cpu->user + cpu->nice + cpu->sys + cpu->idle;

    return SIGAR_OK;
}
```

#### CPU 状态枚举

```c
enum {
    CPU_STATE_USER,    // 用户态
    CPU_STATE_SYSTEM,  // 内核态
    CPU_STATE_IDLE,    // 空闲
    CPU_STATE_NICE,    // Nice 优先级
    CPU_STATE_MAX      // 最大状态数
};
```

### 4. CPU 列表获取

```c
int sigar_cpu_list_get(sigar_t *sigar, sigar_cpu_list_t *cpulist) {
    kern_return_t status;
    mach_msg_type_number_t count;
    processor_cpu_load_info_data_t *cpuload;
    natural_t i, ncpu;

    status = host_processor_info(sigar->mach_port,
                                 PROCESSOR_CPU_LOAD_INFO,
                                 &ncpu,
                                 (processor_info_array_t*)&cpuload,
                                 &count);

    if (status != KERN_SUCCESS) {
        return errno;
    }

    sigar_cpu_list_create(cpulist);

    for (i=0; i<ncpu; i++) {
        sigar_cpu_t *cpu;

        SIGAR_CPU_LIST_GROW(cpulist);
        cpu = &cpulist->data[cpulist->number++];

        cpu->user = SIGAR_TICK2MSEC(cpuload[i].cpu_ticks[CPU_STATE_USER]);
        cpu->sys  = SIGAR_TICK2MSEC(cpuload[i].cpu_ticks[CPU_STATE_SYSTEM]);
        cpu->idle = SIGAR_TICK2MSEC(cpuload[i].cpu_ticks[CPU_STATE_IDLE]);
        cpu->nice = SIGAR_TICK2MSEC(cpuload[i].cpu_ticks[CPU_STATE_NICE]);
        cpu->total = cpu->user + cpu->nice + cpu->sys + cpu->idle;
    }

    // 释放分配的内存
    vm_deallocate(mach_task_self(), (vm_address_t)cpuload, count);

    return SIGAR_OK;
}
```

### 5. 进程列表获取

```c
int sigar_os_proc_list_get(sigar_t *sigar, sigar_proc_list_t *proclist) {
    int mib[4] = { CTL_KERN, KERN_PROC, KERN_PROC_ALL, 0 };
    int i, num;
    size_t len;
    struct kinfo_proc *proc;

    // 获取所需缓冲区大小
    if (sysctl(mib, NMIB(mib), NULL, &len, NULL, 0) < 0) {
        return errno;
    }

    proc = malloc(len);

    // 获取所有进程信息
    if (sysctl(mib, NMIB(mib), proc, &len, NULL, 0) < 0) {
        free(proc);
        return errno;
    }

    num = len / sizeof(*proc);

    // 过滤系统进程
    for (i=0; i<num; i++) {
        if (proc[i].KI_FLAG & P_SYSTEM) {
            continue;
        }
        if (proc[i].KI_PID == 0) {
            continue;
        }
        SIGAR_PROC_LIST_GROW(proclist);
        proclist->data[proclist->number++] = proc[i].KI_PID;
    }

    free(proc);
    return SIGAR_OK;
}
```

### 6. 进程信息缓存机制

```c
static int sigar_get_pinfo(sigar_t *sigar, sigar_pid_t pid) {
    int mib[] = { CTL_KERN, KERN_PROC, KERN_PROC_PID, 0 };
    size_t len = sizeof(*sigar->pinfo);
    time_t timenow = time(NULL);
    mib[3] = pid;

    if (sigar->pinfo == NULL) {
        sigar->pinfo = malloc(len);
    }

    // 缓存检查：如果在缓存过期时间内且PID相同，直接返回
    if (sigar->last_pid == pid) {
        if ((timenow - sigar->last_getprocs) < SIGAR_LAST_PROC_EXPIRE) {
            return SIGAR_OK;
        }
    }

    sigar->last_pid = pid;
    sigar->last_getprocs = timenow;

    if (sysctl(mib, NMIB(mib), sigar->pinfo, &len, NULL, 0) < 0) {
        return errno;
    }

    return SIGAR_OK;
}
```

### 7. 进程内存获取

```c
int sigar_proc_mem_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_mem_t *procmem) {
#ifdef DARWIN_HAS_LIBPROC_H
    struct proc_taskinfo pti;
    struct proc_regioninfo pri;

    if (sigar->libproc) {
        int sz = sigar->proc_pidinfo(pid, PROC_PIDTASKINFO, 0, &pti, sizeof(pti));

        if (sz == sizeof(pti)) {
            procmem->size         = pti.pti_virtual_size;
            procmem->resident     = pti.pti_resident_size;
            procmem->page_faults  = pti.pti_faults;

            // 获取区域信息以计算共享内存
            sz = sigar->proc_pidinfo(pid, PROC_PIDREGIONINFO, 0, &pri, sizeof(pri));
            if (sz == sizeof(pri)) {
                if (pri.pri_share_mode == SM_EMPTY) {
                    mach_vm_size_t shared_size;
                    cpu_type_t cpu_type;

                    if (sigar_proc_cpu_type(sigar, pid, &cpu_type) == SIGAR_OK) {
                        shared_size = sigar_shared_region_size(cpu_type);
                    }

                    // 减去共享区域大小
                    if (procmem->size > shared_size) {
                        procmem->size -= shared_size;
                    }
                }
            }
            return SIGAR_OK;
        }
    }
#endif

    // 回退到 Mach 接口
    mach_port_t task, self = mach_task_self();
    task_basic_info_data_t info;
    task_events_info_data_t events;

    status = task_for_pid(self, pid, &task);

    task_info(task, TASK_BASIC_INFO, (task_info_t)&info, &count);
    task_info(task, TASK_EVENTS_INFO, (task_info_t)&events, &count);

    procmem->size     = info.virtual_size;
    procmem->resident = info.resident_size;
    procmem->page_faults = events.faults;

    mach_port_deallocate(self, task);
}
```

### 8. 进程命令行和环境变量获取

```c
typedef struct {
    char *buf, *ptr, *end;
    int count;
} sigar_kern_proc_args_t;

static int sigar_kern_proc_args_get(sigar_t *sigar, sigar_pid_t pid,
                                    char *exe, sigar_kern_proc_args_t *kargs) {
    int mib[3], len;
    size_t size = sigar_argmax_get(sigar);

    kargs->buf = malloc(size);

    mib[0] = CTL_KERN;
    mib[1] = KERN_PROCARGS2;
    mib[2] = pid;

    if (sysctl(mib, NMIB(mib), kargs->buf, &size, NULL, 0) < 0) {
        return errno;
    }

    kargs->end = &kargs->buf[size];

    // 第一个字段是参数数量
    memcpy(&kargs->count, kargs->buf, sizeof(kargs->count));
    kargs->ptr = kargs->buf + sizeof(kargs->count);

    // 可执行文件路径
    len = strlen(kargs->ptr);
    if (exe) {
        memcpy(exe, kargs->ptr, len+1);
    }
    kargs->ptr += len+1;

    return SIGAR_OK;
}

int sigar_os_proc_args_get(sigar_t *sigar, sigar_pid_t pid,
                           sigar_proc_args_t *procargs) {
    int status, count;
    sigar_kern_proc_args_t kargs;
    char *ptr, *end;

    status = sigar_kern_proc_args_get(sigar, pid, NULL, &kargs);
    if (status != SIGAR_OK) return status;

    count = kargs.count;
    ptr = kargs.ptr;
    end = kargs.end;

    // 解析参数
    while ((ptr < end) && (count-- > 0)) {
        int slen = strlen(ptr);
        char *arg = malloc(slen+1);

        SIGAR_PROC_ARGS_GROW(procargs);
        memcpy(arg, ptr, slen);
        *(arg+slen) = '\0';

        procargs->data[procargs->number++] = arg;
        ptr += slen+1;
    }

    sigar_kern_proc_args_destroy(&kargs);
    return SIGAR_OK;
}
```

### 9. 网络接口列表获取

```c
static int sigar_ifmsg_init(sigar_t *sigar) {
    int mib[] = { CTL_NET, PF_ROUTE, 0, AF_INET, NET_RT_IFLIST, 0 };
    size_t len;

    if (sysctl(mib, NMIB(mib), NULL, &len, NULL, 0) < 0) {
        return errno;
    }

    if (sigar->ifconf_len < len) {
        sigar->ifconf_buf = realloc(sigar->ifconf_buf, len);
        sigar->ifconf_len = len;
    }

    if (sysctl(mib, NMIB(mib), sigar->ifconf_buf, &len, NULL, 0) < 0) {
        return errno;
    }

    return SIGAR_OK;
}

int sigar_net_interface_list_get(sigar_t *sigar, sigar_net_interface_list_t *iflist) {
    int status;
    ifmsg_iter_t iter;

    if ((status = sigar_ifmsg_init(sigar)) != SIGAR_OK) {
        return status;
    }

    iter.type = IFMSG_ITER_LIST;
    iter.data.iflist = iflist;

    return sigar_ifmsg_iter(sigar, &iter);
}
```

---

## 核心功能模块

### 1. 内存监控模块

#### 物理内存
- **API**: `host_statistics(HOST_VM_INFO)`
- **信息**:
  - 总物理内存 (`HW_MEMSIZE`)
  - 空闲页面 (`free_count`)
  - 非活跃页面 (`inactive_count`)
  - 活跃页面 (`active_count`)
  - 固定页面 (`wire_count`)

#### 交换空间
- **macOS 10.4+**: `sysctl(VM_SWAPUSAGE)` - `struct xsw_usage`
- **macOS 10.3 及以下**: 文件系统扫描 `/private/var/vm/swapfile*`
- **信息**:
  - 总交换空间 (`xsu_total`)
  - 已用交换空间 (`xsu_used`)
  - 可用交换空间 (`xsu_avail`)
  - 页面换入/换出 (`pageins`, `pageouts`)

### 2. CPU 监控模块

#### CPU 时间统计
- **User Time**: 用户态时间
- **System Time**: 内核态时间
- **Idle Time**: 空闲时间
- **Nice Time**: Nice 优先级时间

#### CPU 信息
- **API**: `sysctlbyname("hw.cpufrequency")`
- **信息**:
  - CPU 频率 (当前/最大/最小)
  - CPU 型号 (`hw.model`)
  - CPU 厂商 (`machdep.cpu.vendor`)
  - L2 缓存大小 (`HW_L2CACHESIZE`)

#### 多CPU支持
- **API**: `host_processor_info(PROCESSOR_CPU_LOAD_INFO)`
- 支持获取每个逻辑 CPU 的独立统计

### 3. 进程监控模块

#### 进程列表
- **API**: `sysctl(KERN_PROC_ALL)`
- **过滤**: 排除系统进程 (`P_SYSTEM` 标志) 和 PID 0

#### 进程内存
- **优先**: `proc_pidinfo(PROC_PIDTASKINFO)` (libproc)
- **备选**: `task_info(TASK_BASIC_INFO)` (Mach)
- **信息**:
  - 虚拟内存大小 (`pti_virtual_size`)
  - 常驻集大小 (`pti_resident_size`)
  - 页面错误数 (`pti_faults`)

#### 进程时间
- **优先**: `proc_pidinfo(PROC_PIDTASKINFO)`
- **备选**: `task_info(TASK_THREAD_TIMES_INFO)`
- **信息**:
  - 用户态时间 (`pti_total_user`)
  - 内核态时间 (`pti_total_system`)

#### 进程状态
- **线程状态**: 通过 `task_threads()` 和 `thread_info()` 获取
- **状态映射**:
  ```
  TH_STATE_RUNNING     -> 'R' (运行)
  TH_STATE_WAITING     -> 'S' (睡眠) 或 'I' (空闲)
  TH_STATE_STOPPED     -> 'T' (停止)
  TH_STATE_HALTED      -> 'D' (不可中断)
  TH_STATE_UNINTERRUPTIBLE -> 'Z' (僵尸)
  ```

#### 进程环境
- **命令行**: `sysctl(KERN_PROCARGS2)` - 跳过可执行路径后是参数
- **环境变量**: `sysctl(KERN_PROCARGS2)` - 参数结束后是环境变量
- **可执行文件**: `sysctl(KERN_PROCARGS2)` - 第一个字段

### 4. 网络监控模块

#### 网络接口列表
- **API**: `sysctl(NET_RT_IFLIST)`
- **过滤**:
  - 以太网 (`IFT_ETHER`)
  - 回环 (`IFT_LOOP`)
  - 其他有 IP 地址的接口 (`IFT_OTHER`)

#### IP 配置
- **API**: `getifaddrs()`
- **信息**:
  - IPv4 地址
  - IPv6 地址
  - 子网掩码
  - 广播地址

#### 网络统计
- **API**: `sysctl(NET_RT_IFLIST)` 解析 `struct if_msghdr`
- **信息**:
  - 发送/接收字节数
  - 发送/接收包数
  - 发送/接收错误数
  - 丢弃的包数

#### 网络路由
- **API**: `sysctl(NET_RT_DUMP)`
- **信息**:
  - 目标地址
  - 网络掩码
  - 网关
  - 接口索引
  - 跃点数

#### 网络连接
- **API**: `sysctl(NET_TCP_STATS)`
- **信息**:
  - 本地地址和端口
  - 远程地址和端口
  - 连接状态
  - 连接类型 (TCP/UDP)

### 5. 文件系统监控模块

#### 文件系统列表
- **API**: `getfsstat()`
- **过滤**: 跳过自动挂载 (`MNT_AUTOMOUNTED`) 和只读文件系统
- **信息**:
  - 挂载点
  - 设备名称
  - 文件系统类型 (HFS, APFS, 等)
  - 挂载选项

#### 文件系统使用
- **API**: `statvfs()`
- **信息**:
  - 总容量
  - 可用空间
  - 已用空间
  - 使用率

#### 磁盘 I/O 统计
- **API**: I/O Kit 框架
- **信息**:
  - 读取/写入字节数
  - 读取/写入次数
  - 读取/写入时间
  - 队列长度

---

## 性能优化机制

### 1. 进程信息缓存

```c
typedef struct sigar_t {
    time_t last_getprocs;      // 最后获取进程信息时间
    sigar_pid_t last_pid;      // 最后查询的PID
    bsd_pinfo_t *pinfo;        // 进程信息缓存
} sigar_t;
```

缓存有效期检查：`SIGAR_LAST_PROC_EXPIRE`（默认60秒）

### 2. 动态缓冲区管理

```c
// 网络接口信息缓冲区
static int sigar_ifmsg_init(sigar_t *sigar) {
    int mib[] = { CTL_NET, PF_ROUTE, 0, AF_INET, NET_RT_IFLIST, 0 };
    size_t len;

    if (sysctl(mib, NMIB(mib), NULL, &len, NULL, 0) < 0) {
        return errno;
    }

    if (sigar->ifconf_len < len) {
        sigar->ifconf_buf = realloc(sigar->ifconf_buf, len);
        sigar->ifconf_len = len;
    }

    if (sysctl(mib, NMIB(mib), sigar->ifconf_buf, &len, NULL, 0) < 0) {
        return errno;
    }

    return SIGAR_OK;
}
```

### 3. Mach 内存释放

Mach API 返回的缓冲区需要显式释放：

```c
// 释放处理器信息
vm_deallocate(mach_task_self(), (vm_address_t)cpuload, count);

// 释放线程数组
vm_deallocate(self, (vm_address_t)threads, sizeof(thread_t) * count);
```

### 4. libproc 优先使用

libproc 库提供更高效的进程查询：

```c
#ifdef DARWIN_HAS_LIBPROC_H
    if (sigar->libproc) {
        int sz = sigar->proc_pidinfo(pid, PROC_PIDTASKINFO, 0, &pti, sizeof(pti));

        if (sz == sizeof(pti)) {
            // 使用 libproc 的结果
            return SIGAR_OK;
        }
    }
#endif

// 回退到 Mach 接口
```

---

## 特殊技术

### 1. 共享内存区域处理

Darwin 使用共享内存区域来减少内存使用：

```c
static mach_vm_size_t sigar_shared_region_size(cpu_type_t type) {
    switch (type) {
        case CPU_TYPE_ARM:
            return SHARED_REGION_SIZE_ARM;
        case CPU_TYPE_POWERPC:
            return SHARED_REGION_SIZE_PPC;
        case CPU_TYPE_POWERPC64:
            return SHARED_REGION_SIZE_PPC64;
        case CPU_TYPE_I386:
            return SHARED_REGION_SIZE_I386;
        case CPU_TYPE_X86_64:
            return SHARED_REGION_SIZE_X86_64;
        default:
            return SHARED_REGION_SIZE_I386;
    }
}

// 在计算进程虚拟内存时减去共享区域
if (pri.pri_share_mode == SM_EMPTY) {
    if (procmem->size > shared_size) {
        procmem->size -= shared_size;
    }
}
```

### 2. 线程状态映射

Darwin 的线程状态需要映射到 POSIX 状态：

```c
static const char thread_states[] = {
    /*0*/ '-',
    /*1*/ SIGAR_PROC_STATE_RUN,
    /*2*/ SIGAR_PROC_STATE_ZOMBIE,
    /*3*/ SIGAR_PROC_STATE_SLEEP,
    /*4*/ SIGAR_PROC_STATE_IDLE,
    /*5*/ SIGAR_PROC_STATE_STOP,
    /*6*/ SIGAR_PROC_STATE_STOP,
    /*7*/ '?'
};

static int thread_state_get(thread_basic_info_data_t *info) {
    switch (info->run_state) {
        case TH_STATE_RUNNING:
            return 1;
        case TH_STATE_UNINTERRUPTIBLE:
            return 2;
        case TH_STATE_WAITING:
            return (info->sleep_time > 20) ? 4 : 3;
        case TH_STATE_STOPPED:
            return 5;
        case TH_STATE_HALTED:
            return 6;
        default:
            return 7;
    }
}
```

### 3. IPv6 前缀长度计算

```c
static int sigar_in6_prefixlen(struct sockaddr *netmask) {
    struct in6_addr *addr = SIGAR_SIN6_ADDR(netmask);
    u_char *name = (u_char *)addr;
    int size = sizeof(*addr);
    int byte, bit, plen = 0;

    for (byte = 0; byte < size; byte++, plen += 8) {
        if (name[byte] != 0xff) {
            break;
        }
    }

    if (byte == size) {
        return plen;
    }

    for (bit = 7; bit != 0; bit--, plen++) {
        if (!(name[byte] & (1 << bit))) {
            break;
        }
    }

    return plen;
}
```

### 4. 参数最大长度获取

```c
#define SIGAR_ARG_MAX 65536

static size_t sigar_argmax_get(sigar_t *sigar) {
#ifdef KERN_ARGMAX
    int mib[] = { CTL_KERN, KERN_ARGMAX };
    size_t size = sizeof(sigar->argmax);

    if (sigar->argmax != 0) {
        return sigar->argmax;
    }

    if (sysctl(mib, NMIB(mib), &sigar->argmax, &size, NULL, 0) == 0) {
        return sigar->argmax;
    }
#endif
    return SIGAR_ARG_MAX;
}
```

### 5. 文件描述符数量获取

```c
static int proc_fd_get_count(sigar_t *sigar, sigar_pid_t pid, int *num) {
    int pinfo_size;
    struct proc_fdinfo *pbuffer;

    if (!sigar->libproc) {
        return SIGAR_ENOTIMPL;
    }

    // 首先查询所需缓冲区大小
    pinfo_size = sigar->proc_pidinfo(pid, PROC_PIDLISTFDS, 0, NULL, 0);

    if (pinfo_size <= 0) {
        return SIGAR_ENOTIMPL;
    }

    pbuffer = malloc(pinfo_size);
    pinfo_size = sigar->proc_pidinfo(pid, PROC_PIDLISTFDS, 0, pbuffer, pinfo_size);
    free(pbuffer);

    *num = pinfo_size / PROC_PIDLISTFD_SIZE;

    return SIGAR_OK;
}
```

---

## 版本兼容性处理

### 1. iOS 兼容性

通过 `TARGET_OS_IPHONE` 宏控制 iOS 平台的可用功能：

```c
#if TARGET_OS_IPHONE
    // iOS 不支持某些功能
    return SIGAR_ENOTIMPL;
#else
    // macOS 功能
#endif
```

iOS 不支持的功能：
- I/O Kit 磁盘 I/O 统计
- 某些网络路由信息
- NFS 相关头文件

### 2. macOS 版本检测

```c
#if (__MAC_OS_X_VERSION_MAX_ALLOWED >= 120000)
// macOS 12.0+
struct nfsstats {
    uint64_t attrcache_hits;
    uint64_t attrcache_misses;
    // ...
};
#endif
```

### 3. 条件编译

根据不同的 SDK 版本进行条件编译：

```c
#ifdef HAVE_SHARED_REGION_H
#include <mach/shared_region.h>  // 10.5 SDK
#else
#include <mach/shared_memory_server.h>  // 10.4 SDK (deprecated)
#endif
```

### 4. NFS 结构兼容性

macOS 12.0+ 修改了 `nfsstats` 结构，需要手动定义：

```c
#if (__MAC_OS_X_VERSION_MAX_ALLOWED >= 120000)
struct nfsstats {
    uint64_t attrcache_hits;
    uint64_t attrcache_misses;
    uint64_t lookupcache_hits;
    uint64_t lookupcache_misses;
    // ...
};
#endif
```

### 5. Darwin 与其他 BSD 的兼容

```c
#ifdef DARWIN
#include <mach/port.h>
#include <mach/host_info.h>
#else
#include <kvm.h>
#endif

#if defined(__OpenBSD__) || defined(__NetBSD__)
typedef struct kinfo_proc2 bsd_pinfo_t;
#else
typedef struct kinfo_proc bsd_pinfo_t;
#endif
```

---

## 错误处理

### 1. POSIX 错误码

Darwin 使用标准 POSIX 错误码：

```c
#define SIGAR_EPERM_KMEM (SIGAR_OS_START_ERROR+EACCES)
#define SIGAR_EPROC_NOENT (SIGAR_OS_START_ERROR+2)
```

### 2. Mach 错误码

Mach API 返回 `kern_return_t`：

```c
kern_return_t status;

if (status != KERN_SUCCESS) {
    return errno;
}
```

常见的 Mach 错误码：
- `KERN_SUCCESS`: 成功
- `KERN_INVALID_ARGUMENT`: 无效参数
- `KERN_FAILURE`: 操作失败
- `KERN_RESOURCE_SHORTAGE`: 资源不足

### 3. 自定义错误信息

```c
char *sigar_os_error_string(sigar_t *sigar, int err) {
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

---

## 总结

libsigar 的 Darwin/macOS 分支实现展示了以下特点：

1. **多层次API访问**: 从高层 sysctl 到底层 Mach API 的完整覆盖
2. **高效进程查询**: libproc 库提供优化的进程信息获取
3. **动态兼容性**: 通过条件编译支持不同 macOS 版本和 iOS
4. **内存优化**: 缓存机制、动态缓冲区管理、Mach 内存释放
5. **特殊架构支持**: 共享内存区域处理、多架构 (x86/ARM) 支持

### 技术亮点

1. **sysctl 核心接口**: 统一的系统信息查询接口
2. **Mach 内核接口**: 底层系统统计和进程管理
3. **libproc 高效库**: 快速进程信息查询
4. **I/O Kit 框架**: 设备级别的 I/O 统计
5. **路由表解析**: 复杂的路由消息处理

### 与其他平台的差异

| 特性 | Darwin | Linux | Windows |
|-----|--------|-------|---------|
| 主要接口 | sysctl, Mach | /proc, /sys | 性能计数器, API |
| 进程信息 | libproc, Mach | /proc/{pid}/ | PSAPI, WMI |
| 内存信息 | host_statistics | /proc/meminfo | GlobalMemoryStatusEx |
| CPU 信息 | host_processor_info | /proc/stat | 性能计数器, NT API |
| 网络信息 | PF_ROUTE, getifaddrs | /proc/net, netlink | iphlpapi.dll |
| 磁盘信息 | I/O Kit | /proc/diskstats | 性能计数器 |

这种设计使 libsigar 能够在 macOS/iOS 平台上稳定、高效地收集系统信息，同时保持代码的可维护性和跨平台一致性。
