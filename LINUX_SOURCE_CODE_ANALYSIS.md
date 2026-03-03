# libsigar Linux 平台源码分析

## 概述

libsigar 在 Linux 平台的实现主要通过 `/proc` 文件系统和 `/sys` 文件系统获取系统信息。与 Windows 和 macOS 不同，Linux 提供了统一的 procfs 接口，使得系统信息的获取相对简单和统一。

## 核心架构

### sigar_t 结构体 (sigar_os.h:59-70)

```c
struct sigar_t {
    SIGAR_T_BASE;
    int pagesize;              // 页面大小的位移值
    int ram;                  // 物理内存大小 (缓存)
    int proc_signal_offset;    // 进程信号字段偏移量
    linux_proc_stat_t last_proc_stat; // 进程统计信息缓存
    int lcpu;                 // 每个物理 CPU 的逻辑 CPU 数
    linux_iostat_e iostat;    // 磁盘 I/O 统计类型
    char *proc_net;           // 自定义 /proc/net 路径
    int has_nptl;             // 是否支持 NPTL (Native POSIX Thread Library)
};
```

### linux_proc_stat_t 结构体 (sigar_os.h:33-50)

```c
typedef struct {
    sigar_pid_t pid;          // 进程 ID
    time_t mtime;             // 最后修改时间 (缓存过期检查)
    sigar_uint64_t vsize;     // 虚拟内存大小
    sigar_uint64_t rss;       // 常驻集大小
    sigar_uint64_t minor_faults; // 次要缺页错误
    sigar_uint64_t major_faults; // 主要缺页错误
    sigar_uint64_t ppid;      // 父进程 ID
    int tty;                  // 终端编号
    int priority;             // 优先级
    int nice;                 // nice 值
    sigar_uint64_t start_time; // 启动时间
    sigar_uint64_t utime;     // 用户 CPU 时间
    sigar_uint64_t stime;     // 系统 CPU 时间
    char name[SIGAR_PROC_NAME_LEN]; // 进程名称
    char state;               // 进程状态
    int processor;            // 最后运行的处理器
} linux_proc_stat_t;
```

## 核心技术

### 1. /proc 文件系统

libsigar Linux 实现的核心是 `/proc` 文件系统，这是一个虚拟文件系统，提供内核和进程信息。

#### 关键路径 (linux_sigar.c:54-68)

```c
static char *gPROC_FS_ROOT;      // "/proc"
static char *gPROCP_FS_ROOT;     // "/proc/"
static char *gPROC_MEMINFO;      // "/proc/meminfo"
static char *gPROC_VMSTAT;       // "/proc/vmstat"
static char *gPROC_MTRR;         // "/proc/mtrr"
static char *gPROC_STAT;         // "/proc/stat"
static char *gPROC_UPTIME;       // "/proc/uptime"
static char *gPROC_LOADAVG;      // "/proc/loadavg"
static char *gPROC_DISKSTATS;    // "/proc/diskstats"
static char *gPROC_PARTITIONS;   // "/proc/partitions"
static char *gSYS_BLOCK;         // "/sys/block"
static char *gMOUNTED;           // "/etc/mtab"
```

### 2. 初始化流程 (linux_sigar.c:176-236)

```c
int sigar_os_open(sigar_t **sigar)
{
    // 1. 设置 proc 文件路径
    set_proc_locations();

    // 2. 分配 sigar 结构
    *sigar = malloc(sizeof(**sigar));

    // 3. 计算页大小位移值
    i = getpagesize();
    while ((i >>= 1) > 0) {
        (*sigar)->pagesize++;
    }

    // 4. 获取系统启动时间
    status = sigar_boot_time_get(*sigar);

    // 5. 获取系统时钟频率
    (*sigar)->ticks = sysconf(_SC_CLK_TCK);

    // 6. 检测可用的磁盘 I/O 统计方法
    if (stat(gPROC_DISKSTATS, &sb) == 0) {
        (*sigar)->iostat = IOSTAT_DISKSTATS;    // 2.6 内核
    }
    else if (stat(gSYS_BLOCK, &sb) == 0) {
        (*sigar)->iostat = IOSTAT_SYS;          // 2.6 内核 /sys
    }
    else if (stat(gPROC_PARTITIONS, &sb) == 0) {
        (*sigar)->iostat = IOSTAT_PARTITIONS;   // 2.4 内核
    }
    else {
        (*sigar)->iostat = IOSTAT_NONE;
    }

    // 7. 检测 NPTL 支持 (2.6+ 内核)
    uname(&name);
    kernel_rev = atoi(&name.release[2]);
    if (kernel_rev >= 6) {
        has_nptl = 1;
    }
    else {
        has_nptl = getenv("SIGAR_HAS_NPTL") ? 1 : 0;
    }

    return SIGAR_OK;
}
```

## 核心功能模块

### 1. 内存监控

#### 内存信息获取 (linux_sigar.c:346-376)

```c
int sigar_mem_get(sigar_t *sigar, sigar_mem_t *mem)
{
    // 读取 /proc/meminfo
    // MemTotal:   总物理内存
    // MemFree:    空闲内存
    // Buffers:    缓冲区
    // Cached:     页面缓存

    mem->total  = sigar_meminfo(buffer, MEMINFO_PARAM("MemTotal"));
    mem->free   = sigar_meminfo(buffer, MEMINFO_PARAM("MemFree"));
    mem->used   = mem->total - mem->free;

    buffers = sigar_meminfo(buffer, MEMINFO_PARAM("Buffers"));
    cached  = sigar_meminfo(buffer, MEMINFO_PARAM("Cached"));

    kern = buffers + cached;
    mem->actual_free = mem->free + kern;   // 实际可用内存
    mem->actual_used = mem->used - kern;   // 实际已用内存

    // 尝试从 MTRR 获取物理内存大小
    get_ram(sigar, mem);

    return SIGAR_OK;
}
```

#### MTRR 内存检测 (linux_sigar.c:258-318)

```c
static int get_ram(sigar_t *sigar, sigar_mem_t *mem)
{
    // 读取 /proc/mtrr (Memory Type Range Registers)
    // write-back 寄存器总和等于总物理内存

    if (!(fp = fopen(gPROC_MTRR, "r"))) {
        return errno;
    }

    while ((ptr = fgets(buffer, sizeof(buffer), fp))) {
        if (strstr(ptr, "size=") && strstr(ptr, "write-back")) {
            ptr += 5;  // 跳过 "size="
            total += atoi(ptr);
        }
    }

    if ((total - sys_total) > 256) {
        // MTRR 寄存器总和偏差太大，忽略
        total = 0;
    }

    mem->ram = sigar->ram = total;
    return SIGAR_OK;
}
```

#### 交换空间信息 (linux_sigar.c:378-424)

```c
int sigar_swap_get(sigar_t *sigar, sigar_swap_t *swap)
{
    // 从 /proc/meminfo 读取交换空间
    swap->total  = sigar_meminfo(buffer, MEMINFO_PARAM("SwapTotal"));
    swap->free   = sigar_meminfo(buffer, MEMINFO_PARAM("SwapFree"));
    swap->used   = swap->total - swap->free;

    // 从 /proc/vmstat (2.6+) 读取分页统计
    // pswpin:  换入页数
    // pswpout: 换出页数

    // 或从 /proc/stat (2.2, 2.4) 读取
    // swap: 换入换出统计

    return SIGAR_OK;
}
```

### 2. CPU 监控

#### CPU 统计 (linux_sigar.c:426-462)

```c
static void get_cpu_metrics(sigar_t *sigar, sigar_cpu_t *cpu, char *line)
{
    // /proc/stat 格式:
    // cpu  user nice system idle iowait irq softirq steal

    cpu->user += SIGAR_TICK2MSEC(sigar_strtoull(ptr));
    cpu->nice += SIGAR_TICK2MSEC(sigar_strtoull(ptr));
    cpu->sys  += SIGAR_TICK2MSEC(sigar_strtoull(ptr));
    cpu->idle += SIGAR_TICK2MSEC(sigar_strtoull(ptr));

    // 2.6+ 内核
    if (*ptr == ' ') {
        cpu->wait += SIGAR_TICK2MSEC(sigar_strtoull(ptr));      // iowait
        cpu->irq += SIGAR_TICK2MSEC(sigar_strtoull(ptr));       // irq
        cpu->soft_irq += SIGAR_TICK2MSEC(sigar_strtoull(ptr));  // softirq
    }

    // 2.6.11+ 内核
    if (*ptr == ' ') {
        cpu->stolen += SIGAR_TICK2MSEC(sigar_strtoull(ptr));    // steal (虚拟化)
    }

    cpu->total = user + nice + sys + idle + wait + irq + soft_irq + stolen;
}
```

#### CPU 列表 (linux_sigar.c:464-517)

```c
int sigar_cpu_list_get(sigar_t *sigar, sigar_cpu_list_t *cpulist)
{
    // 读取 /proc/stat 中的 cpu0, cpu1, ...
    // 支持超线程核心合并

    if (core_rollup && (i % sigar->lcpu)) {
        // 合并逻辑处理器的时间
        cpu = &cpulist->data[cpulist->number-1];
    }
    else {
        // 创建新的 CPU 条目
        SIGAR_CPU_LIST_GROW(cpulist);
        cpu = &cpulist->data[cpulist->number++];
    }

    get_cpu_metrics(sigar, cpu, ptr);
}
```

#### CPU 信息 (linux_sigar.c:1784-1822)

```c
int sigar_cpu_info_list_get(sigar_t *sigar, sigar_cpu_info_list_t *cpu_infos)
{
    // 读取 /proc/cpuinfo
    // processor : 0
    // vendor_id : GenuineIntel
    // model name : Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz
    // cpu MHz : 2400.000
    // cache size : 4096 KB

    // 从 /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq 获取最大频率
    // 从 /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_min_freq 获取最小频率

    return SIGAR_OK;
}
```

### 3. 进程监控

#### 进程列表 (linux_sigar.c:666-707)

```c
int sigar_os_proc_list_get(sigar_t *sigar, sigar_proc_list_t *proclist)
{
    DIR *dirp = opendir(gPROCP_FS_ROOT);  // 打开 /proc

    while ((ent = readdir(dirp)) != NULL) {
        if (!sigar_isdigit(*ent->d_name)) {
            continue;  // 跳过非数字目录
        }

        // 检查是否为线程 (非 NPTL 模式)
        if (threadbadhack &&
            proc_isthread(sigar, ent->d_name, strlen(ent->d_name)))
        {
            continue;  // 跳过线程
        }

        proclist->data[proclist->number++] =
            strtoul(ent->d_name, NULL, 10);
    }

    return SIGAR_OK;
}
```

#### 线程检测 (linux_sigar.c:600-664)

```c
static SIGAR_INLINE int proc_isthread(sigar_t *sigar, char *pidstr, int len)
{
    // 检查 /proc/<pid>/stat 的 exit_signal 字段
    // '17' == SIGCHLD == 真正的进程
    // '33' 或 '0' == 线程

    // /proc/self/stat 字段:
    // 1-37: 标准字段
    // 38: exit_signal

    // 从后向前解析 exit_signal
    while (offset-- > 0) {
        while ((n > 0) && isdigit(buffer[n--])) ;   // 跳过字段
        while ((n > 0) && !isdigit(buffer[n--])) ; // 跳过空格
    }

    ptr = &buffer[n];
    if ((*ptr++ == '1') && (*ptr++ == '7') && (*ptr++ == ' ')) {
        return 0;  // 真正的进程
    }

    return 1;  // 线程
}
```

#### 进程状态读取 (linux_sigar.c:709-808)

```c
static int proc_stat_read(sigar_t *sigar, sigar_pid_t pid)
{
    // 短期缓存: 60 秒内重复读取同一进程使用缓存
    if (pstat->pid == pid) {
        if ((timenow - pstat->mtime) < SIGAR_LAST_PROC_EXPIRE) {
            return SIGAR_OK;
        }
    }

    // 读取 /proc/<pid>/stat
    // 格式 (部分字段):
    // 1  pid           %d
    // 2  comm          %s (进程名，可能被截断)
    // 3  state         %c
    // 4  ppid          %d
    // 14 utime         %lu (用户时间，时钟周期)
    // 15 stime         %lu (系统时间，时钟周期)
    // 22 starttime     %lu (启动后的时钟周期数)
    // 23 vsize         %lu (虚拟内存大小)
    // 24 rss           %lu (常驻集大小，页数)

    // 解析进程名 (处理括号内的可能包含空格的进程名)
    if (!(ptr = strchr(ptr, '('))) {
        return EINVAL;
    }
    if (!(tmp = strrchr(++ptr, ')'))) {
        return EINVAL;
    }
    len = tmp-ptr;
    memcpy(pstat->name, ptr, len);
    pstat->name[len] = '\0';

    // 跳过其他字段...
    // 解析关键信息

    pstat->utime = SIGAR_TICK2MSEC(sigar_strtoull(ptr));  // 用户时间 (毫秒)
    pstat->stime = SIGAR_TICK2MSEC(sigar_strtoull(ptr));  // 系统时间 (毫秒)

    pstat->start_time  = sigar_strtoul(ptr);  // 时钟周期数
    pstat->start_time /= sigar->ticks;        // 转换为秒
    pstat->start_time += sigar->boot_time;     // 加上启动时间
    pstat->start_time *= 1000;                 // 转换为毫秒

    pstat->vsize = sigar_strtoull(ptr);        // 虚拟内存 (字节)
    pstat->rss   = pageshift(sigar_strtoull(ptr));  // 常驻集 (字节)

    return SIGAR_OK;
}
```

#### 进程内存 (linux_sigar.c:810-833)

```c
int sigar_proc_mem_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_mem_t *procmem)
{
    // 从缓存的 stat 读取缺页错误
    procmem->minor_faults = pstat->minor_faults;
    procmem->major_faults = pstat->major_faults;
    procmem->page_faults = procmem->minor_faults + procmem->major_faults;

    // 读取 /proc/<pid>/statm
    // 格式: size resident shared text lib data dt
    // size:     总虚拟内存大小 (页)
    // resident: 常驻集大小 (页)
    // shared:   共享内存页数

    status = SIGAR_PROC_FILE2STR(buffer, pid, "/statm");

    procmem->size     = pageshift(sigar_strtoull(ptr));
    procmem->resident = pageshift(sigar_strtoull(ptr));
    procmem->share    = pageshift(sigar_strtoull(ptr));

    return SIGAR_OK;
}
```

#### 进程磁盘 I/O (linux_sigar.c:845-861)

```c
int sigar_proc_cumulative_disk_io_get(sigar_t *sigar, sigar_pid_t pid,
                                       sigar_proc_cumulative_disk_io_t *proc_cumulative_disk_io)
{
    // 读取 /proc/<pid>/io (2.6.20+)
    // read_bytes:  读字节数
    // write_bytes: 写字节数
    // cancel_write_bytes: 取消写回的字节数

    status = SIGAR_PROC_FILE2STR(buffer, pid, "/io");

    proc_cumulative_disk_io->bytes_read = get_named_proc_token(buffer, "\nread_bytes");
    proc_cumulative_disk_io->bytes_written = get_named_proc_token(buffer, "\nwrite_bytes");
    proc_cumulative_disk_io->bytes_total =
        proc_cumulative_disk_io->bytes_read + proc_cumulative_disk_io->bytes_written;

    return SIGAR_OK;
}
```

#### 进程凭据 (linux_sigar.c:865-900)

```c
int sigar_proc_cred_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_cred_t *proccred)
{
    // 读取 /proc/<pid>/status
    // Uid: real euid saved setuid fsuid
    // Gid: real egid saved setgid fsgid

    status = SIGAR_PROC_FILE2STR(buffer, pid, PROC_PSTATUS);

    if ((ptr = strstr(buffer, "\nUid:"))) {
        ptr = sigar_skip_token(ptr);
        proccred->uid  = sigar_strtoul(ptr);  // 真实 UID
        proccred->euid = sigar_strtoull(ptr);  // 有效 UID
    }

    if ((ptr = strstr(ptr, "\nGid:"))) {
        ptr = sigar_skip_token(ptr);
        proccred->gid  = sigar_strtoul(ptr);  // 真实 GID
        proccred->egid = sigar_strtoull(ptr);  // 有效 GID
    }

    return SIGAR_OK;
}
```

#### 进程状态 (linux_sigar.c:943-997)

```c
int sigar_proc_state_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_state_t *procstate)
{
    // 从 stat 读取基本状态
    procstate->state = pstat->state;         // R|S|D|Z|T
    procstate->ppid = pstat->ppid;
    procstate->tty = pstat->tty;
    procstate->priority = pstat->priority;
    procstate->nice = pstat->nice;
    procstate->processor = pstat->processor;

    // 进程名处理: 如果被截断为 15 字符，尝试从命令行获取完整名称
    if (strlen(pstat->name) == 15) {
        if (sigar_procfs_args_get(sigar, pid, &procargs) == SIGAR_OK &&
            procargs.number >= 1) {
            procname = procargs.data[0];
        }
    }

    // 从 status 读取线程数
    ptr = strstr(buffer, "\nThreads:");
    if (ptr) {
        ptr = sigar_skip_token(ptr);
        procstate->threads = sigar_strtoul(ptr);
    }

    // 获取打开的文件描述符数量
    sigar_proc_fd_t proc_fd_count;
    if (sigar_proc_fd_get(sigar, pid, &proc_fd_count) == SIGAR_OK) {
        procstate->open_files = proc_fd_count.total;
    }

    return SIGAR_OK;
}
```

#### 进程环境变量 (linux_sigar.c:1010-1067)

```c
int sigar_proc_env_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_env_t *procenv)
{
    // 读取 /proc/<pid>/environ
    // 格式: key1=value1\0key2=value2\0...\0

    if ((fd = open(name, O_RDONLY)) < 0) {
        if (errno == ENOENT) {
            return ESRCH;  // 进程不存在
        }
        return errno;
    }

    len = read(fd, buffer, sizeof(buffer));
    close(fd);

    buffer[len] = '\0';
    ptr = buffer;

    end = buffer + len;
    while (ptr < end) {
        char *val = strchr(ptr, '=');

        if (val == NULL) {
            break;
        }

        klen = val - ptr;
        SIGAR_SSTRCPY(key, ptr);
        key[klen] = '\0';
        ++val;

        vlen = strlen(val);
        status = procenv->env_getter(procenv->data, key, klen, val, vlen);

        if (status != SIGAR_OK) {
            break;  // 回调要求停止
        }

        ptr += (klen + 1 + vlen + 1);
    }

    return SIGAR_OK;
}
```

#### 进程可执行文件信息 (linux_sigar.c:1078-1107)

```c
int sigar_proc_exe_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_exe_t *procexe)
{
    // /proc/<pid>/cwd -> 工作目录 (符号链接)
    // /proc/<pid>/exe -> 可执行文件 (符号链接)
    // /proc/<pid>/root -> 根目录 (符号链接)

    (void)SIGAR_PROC_FILENAME(name, pid, "/cwd");
    if ((len = readlink(name, procexe->cwd, sizeof(procexe->cwd)-1)) >= 0) {
        procexe->cwd[len] = '\0';
    }

    (void)SIGAR_PROC_FILENAME(name, pid, "/exe");
    if ((len = readlink(name, procexe->name, sizeof(procexe->name)-1)) >= 0) {
        procexe->name[len] = '\0';
    }

    (void)SIGAR_PROC_FILENAME(name, pid, "/root");
    if ((len = readlink(name, procexe->root, sizeof(procexe->root)-1)) >= 0) {
        procexe->root[len] = '\0';
    }

    // 通过 ELF 文件猜测架构
    procexe->arch = sigar_elf_file_guess_arch(sigar, procexe->name);

    return SIGAR_OK;
}
```

#### 进程模块 (linux_sigar.c:1109-1151)

```c
int sigar_proc_modules_get(sigar_t *sigar, sigar_pid_t pid, sigar_proc_modules_t *procmods)
{
    // 读取 /proc/<pid>/maps
    // 格式:
    // address           perms offset  dev   inode   pathname
    // 7f8c5c000000-7f8c5c010000 rw-p 00000000 00:00 0
    // 7f8c5c010000-7f8c5c020000 r-xp 00000000 08:01 123456 /lib/libc.so.6

    while ((ptr = fgets(buffer, sizeof(buffer), fp))) {
        // 跳过前 4 个字段 (address, perms, offset, dev)
        ptr = sigar_skip_multiple_token(ptr, 4);
        inode = sigar_strtoul(ptr);

        // inode 为 0 表示匿名映射，跳过
        if ((inode == 0) || (inode == last_inode)) {
            last_inode = 0;
            continue;
        }

        last_inode = inode;
        ptr = strrchr(ptr, ' ') + 1;  // 路径名在最后
        len = strlen(ptr);
        ptr[len-1] = '\0';  // 去掉换行符

        status = procmods->module_getter(procmods->data, ptr, len-1);

        if (status != SIGAR_OK) {
            break;
        }
    }

    return SIGAR_OK;
}
```

### 4. 网络监控

#### 网络接口统计 (linux_sigar.c:1946-2016)

```c
int sigar_net_interface_stat_get(sigar_t *sigar, const char *name,
                                 sigar_net_interface_stat_t *ifstat)
{
    // 读取 /proc/net/dev
    // 格式:
    // Inter-|   Receive                                                |  Transmit
    //  face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
    //   eth0: 123456   1000    0    0    0     0          0         654321   2000    1    0    0      0       0          0

    while (fgets(buffer, sizeof(buffer), fp)) {
        dev = buffer;
        while (isspace(*dev)) dev++;
        if (!(ptr = strchr(dev, ':'))) continue;
        *ptr++ = 0;

        if (!strEQ(dev, name)) continue;

        // 接收统计
        ifstat->rx_bytes    = sigar_strtoull(ptr);
        ifstat->rx_packets  = sigar_strtoull(ptr);
        ifstat->rx_errors   = sigar_strtoull(ptr);
        ifstat->rx_dropped  = sigar_strtoull(ptr);
        ifstat->rx_overruns = sigar_strtoull(ptr);
        ifstat->rx_frame    = sigar_strtoull(ptr);

        ptr = sigar_skip_multiple_token(ptr, 2);  // 跳过 compressed, multicast

        // 发送统计
        ifstat->tx_bytes      = sigar_strtoull(ptr);
        ifstat->tx_packets    = sigar_strtoull(ptr);
        ifstat->tx_errors     = sigar_strtoull(ptr);
        ifstat->tx_dropped    = sigar_strtoull(ptr);
        ifstat->tx_overruns   = sigar_strtoull(ptr);
        ifstat->tx_collisions = sigar_strtoull(ptr);
        ifstat->tx_carrier    = sigar_strtoull(ptr);

        // 从 /sys/class/net/<name>/speed 获取网卡速度
        ifstat->speed = nic_speed_get(name);

        break;
    }

    return found ? SIGAR_OK : ENXIO;
}
```

#### 网卡速度 (linux_sigar.c:1915-1944)

```c
static SIGAR_INLINE int nic_speed_get(const char *name)
{
    // 读取 /sys/class/net/<name>/speed
    // 值为 Mbps

    char buffer[BUFSIZ];
    const char *fname = nic_speed_filename(buffer, BUFSIZ, name);

    if (fname) {
        FILE *fp = fopen(fname, "r");
        if (fp) {
            int speed;
            int n = fscanf(fp, "%d", &speed);
            fclose(fp);
            return n == 1 ? speed : SIGAR_FIELD_NOTIMPL;
        }
    }

    return SIGAR_FIELD_NOTIMPL;
}
```

#### 网络连接 (linux_sigar.c:2099-2221)

```c
static int proc_net_read(sigar_net_connection_walker_t *walker,
                         const char *fname, int type)
{
    // 读取 /proc/net/tcp, /proc/net/tcp6, /proc/net/udp, /proc/net/udp6, /proc/net/raw
    // 格式:
    //   sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode
    //    0: 0100007F:0035 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 12345 1 0000000000000000

    // local_address 和 rem_address 为十六进制
    // IPv4: 0100007F = 127.0.0.1, 0035 = 53 (端口)
    // IPv6: 32 字符十六进制

    while ((ptr = fgets(buffer, sizeof(buffer), fp))) {
        // 跳过前导空格
        SKIP_WHILE(ptr, ' ');

        // 跳过 "%d: "
        SKIP_PAST(ptr, ' ');

        // 解析本地地址
        laddr = ptr;
        while (*ptr && (*ptr != ':')) {
            laddr_len++;
            ptr++;
        }
        SKIP_WHILE(ptr, ':');
        conn.local_port = (strtoul(ptr, &ptr, 16) & 0xffff);

        // 解析远程地址
        raddr = ptr;
        while (*ptr && (*ptr != ':')) {
            raddr_len++;
            ptr++;
        }
        SKIP_WHILE(ptr, ':');
        conn.remote_port = (strtoul(ptr, &ptr, 16) & 0xffff);

        // 转换地址
        convert_hex_address(&conn.local_address, laddr, laddr_len);
        convert_hex_address(&conn.remote_address, raddr, raddr_len);

        // 解析状态 (2 位十六进制)
        conn.state = hex2int(ptr, 2);
        ptr += 2;

        // 解析队列
        conn.send_queue = hex2int(ptr, HEX_ENT_LEN);
        ptr += HEX_ENT_LEN+1;
        conn.receive_queue = hex2int(ptr, HEX_ENT_LEN);

        // 解析 UID 和 inode
        SKIP_PAST(ptr, ' ');  // tr:tm->whem
        SKIP_PAST(ptr, ' ');  // retrnsmt
        conn.uid = sigar_strtoul(ptr);
        SKIP_WHILE(ptr, ' ');
        SKIP_PAST(ptr, ' ');  // timeout
        conn.inode = sigar_strtoull(ptr);

        more = walker->add_connection(walker, &conn);
        if (more != SIGAR_OK) {
            break;
        }
    }

    return SIGAR_OK;
}
```

#### 十六进制地址转换 (linux_sigar.c:2018-2035)

```c
static SIGAR_INLINE void convert_hex_address(sigar_net_address_t *address,
                                             char *ptr, int len)
{
    if (len > HEX_ENT_LEN) {  // 8 字符以上为 IPv6
        int i;
        for (i=0; i<=3; i++, ptr+=HEX_ENT_LEN) {
            address->addr.in6[i] = hex2int(ptr, HEX_ENT_LEN);
        }
        address->family = SIGAR_AF_INET6;
    }
    else {  // IPv4
        address->addr.in = (len == HEX_ENT_LEN) ? hex2int(ptr, HEX_ENT_LEN) : 0;
        address->family = SIGAR_AF_INET;
    }
}
```

#### 路由表 (linux_sigar.c:1855-1913)

```c
int sigar_net_route_list_get(sigar_t *sigar, sigar_net_route_list_t *routelist)
{
    // 读取 /proc/net/route
    // 格式:
    // Iface    Destination     Gateway         Flags    RefCnt Use Metric Mask            MTU Window IRTT
    // eth0     00000000        0101110A        0003     0      0   0      00000000        1500 0      0

    while (fgets(buffer, sizeof(buffer), fp)) {
        num = sscanf(buffer, ROUTE_FMT,
                     route->ifname, net_addr, gate_addr,
                     &flags, &route->refcnt, &route->use,
                     &route->metric, mask_addr,
                     &route->mtu, &route->window, &route->irtt);

        if ((num < 10) || !(flags & RTF_UP)) {  // 必须是活动路由
            --routelist->number;
            continue;
        }

        route->flags = flags;

        // 转换十六进制地址
        sigar_net_address_set(route->destination, hex2int(net_addr, HEX_ENT_LEN));
        sigar_net_address_set(route->gateway, hex2int(gate_addr, HEX_ENT_LEN));
        sigar_net_address_set(route->mask, hex2int(mask_addr, HEX_ENT_LEN));
    }

    return SIGAR_OK;
}
```

#### IPv6 配置 (linux_sigar.c:2349-2389)

```c
int sigar_net_interface_ipv6_config_get(sigar_t *sigar, const char *name,
                                        sigar_net_interface_config_t *ifconfig)
{
    // 读取 /proc/net/if_inet6
    // 格式:
    // 00000000000000000000000000000001 02 40 00 00   eth0
    // addr                    idx prefix scope flags ifname

    while (fscanf(fp, "%32s %02x %02x %02x %02x %8s\n",
                  addr, &idx, &prefix, &scope, &flags, ifname) != EOF)
    {
        if (strEQ(name, ifname)) {
            status = SIGAR_OK;
            break;
        }
    }

    if (status == SIGAR_OK) {
        // 转换 32 字符十六进制地址为 16 字节
        unsigned char *addr6 = (unsigned char *)&(ifconfig->address6.addr.in6);
        char *ptr = addr;
        for (i=0; i<16; i++, ptr+=2) {
            addr6[i] = (unsigned char)hex2int(ptr, 2);
        }
        ifconfig->prefix6_length = prefix;
        ifconfig->scope6 = scope;
    }

    return status;
}
```

#### ARP 表 (linux_sigar.c:2659-2742)

```c
int sigar_arp_list_get(sigar_t *sigar, sigar_arp_list_t *arplist)
{
    // 读取 /proc/net/arp
    // 格式:
    // IP address       HW type     Flags       HW address            Mask     Device
    // 192.168.1.1     0x1         0x2         00:11:22:33:44:55   *        eth0

    while (fgets(buffer, sizeof(buffer), fp))) {
        num = sscanf(buffer, "%128s 0x%x 0x%x %128s %128s %16s",
                     net_addr, &type, &flags,
                     hwaddr, mask_addr, arp->ifname);

        if (num < 6) {
            --arplist->number;
            continue;
        }

        arp->flags = flags;

        // 解析 IP 地址 (支持 IPv4 和 IPv6)
        status = inet_pton(AF_INET, net_addr, &arp->address.addr);
        if (status > 0) {
            arp->address.family = SIGAR_AF_INET;
        }
        else if ((status = inet_pton(AF_INET6, net_addr, &arp->address.addr)) > 0) {
            arp->address.family = SIGAR_AF_INET6;
        }

        // 解析 MAC 地址
        num = sscanf(hwaddr, "%02hhx:%02hhx:%02hhx:%02hhx:%02hhx:%02hhx",
                     &arp->hwaddr.addr.mac[0], &arp->hwaddr.addr.mac[1],
                     &arp->hwaddr.addr.mac[2], &arp->hwaddr.addr.mac[3],
                     &arp->hwaddr.addr.mac[4], &arp->hwaddr.addr.mac[5]);
        if (num < 6) {
            --arplist->number;
            continue;
        }
        arp->hwaddr.family = SIGAR_AF_LINK;

        // 硬件类型
        SIGAR_SSTRCPY(arp->type, get_hw_type(type));
    }

    return SIGAR_OK;
}
```

#### TCP 统计 (linux_sigar.c:2420-2462)

```c
int sigar_tcp_get(sigar_t *sigar, sigar_tcp_t *tcp)
{
    // 读取 /proc/net/snmp
    // 格式:
    // Tcp: RtoAlgorithm RtoMin RtoMax MaxConn ActiveOpens PassiveOpens AttemptFails EstabResets CurrEstab InSegs OutSegs RetransSegs InErrs OutRsts
    // Tcp: 1 200 120000 -1 123 456 7 8 9 101112 131415 161718 19 202122

    while (fgets(buffer, sizeof(buffer), fp)) {
        if (strnEQ(buffer, SNMP_TCP_PREFIX, sizeof(SNMP_TCP_PREFIX)-1)) {
            if (fgets(buffer, sizeof(buffer), fp)) {
                status = SIGAR_OK;
                break;
            }
        }
    }

    if (status == SIGAR_OK) {
        ptr = sigar_skip_multiple_token(ptr, 5);  // 跳过前 5 个字段
        tcp->active_opens = sigar_strtoull(ptr);
        tcp->passive_opens = sigar_strtoull(ptr);
        tcp->attempt_fails = sigar_strtoull(ptr);
        tcp->estab_resets = sigar_strtoull(ptr);
        tcp->curr_estab = sigar_strtoull(ptr);
        tcp->in_segs = sigar_strtoull(ptr);
        tcp->out_segs = sigar_strtoull(ptr);
        tcp->retrans_segs = sigar_strtoull(ptr);
        tcp->in_errs = sigar_strtoull(ptr);
        tcp->out_rsts = sigar_strtoull(ptr);
    }

    return status;
}
```

#### 端口到进程映射 (linux_sigar.c:2744-2852)

```c
int sigar_proc_port_get(sigar_t *sigar, int protocol,
                        unsigned long port, sigar_pid_t *pid)
{
    // 1. 获取连接信息 (包含 inode)
    status = sigar_net_connection_get(sigar, &netconn, port,
                                      SIGAR_NETCONN_SERVER|protocol);

    if (status != SIGAR_OK) {
        return status;
    }

    // 2. 遍历 /proc/<pid>/fd，查找匹配的 inode
    if (!(dirp = opendir(gPROCP_FS_ROOT))) {
        return errno;
    }

    while ((ent = readdir(dirp)) != NULL) {
        if (!sigar_isdigit(*ent->d_name)) continue;

        // 构造路径: /proc/<pid>/fd
        memcpy(&fd_name[0], fd_name, len);
        memcpy(&fd_name[len], "/fd", 3);
        fd_name[len+=3] = '\0';

        if (!(fd_dirp = opendir(fd_name))) continue;

        while ((fd_ent = readdir(fd_dirp)) != NULL) {
            if (!sigar_isdigit(*fd_ent->d_name)) continue;

            // 构造路径: /proc/<pid>/fd/<fd>
            // 检查 inode 是否匹配
            if (stat(fd_ent_name, &sb) < 0) continue;

            if (sb.st_ino == netconn.inode) {
                closedir(fd_dirp);
                closedir(dirp);
                *pid = strtoul(ent->d_name, NULL, 10);
                return SIGAR_OK;
            }
        }
        closedir(fd_dirp);
    }

    closedir(dirp);
    return SIGAR_OK;
}
```

#### NFS 统计 (linux_sigar.c:2464-2599)

```c
int sigar_nfs_client_v2_get(sigar_t *sigar, sigar_nfs_client_v2_t *nfs)
{
    // 读取 /proc/net/rpc/nfs
    // proc2 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
    // 保留字段 followed by v2 操作计数

    status = sigar_proc_nfs_gets(file, "proc2", buffer, sizeof(buffer));

    ptr = sigar_skip_multiple_token(ptr, 2);
    nfs->null = sigar_strtoull(ptr);
    nfs->getattr = sigar_strtoull(ptr);
    nfs->setattr = sigar_strtoull(ptr);
    // ... 其他操作

    return SIGAR_OK;
}
```

### 5. 文件系统监控

#### 文件系统列表 (linux_sigar.c:1227-1257)

```c
int sigar_file_system_list_get(sigar_t *sigar, sigar_file_system_list_t *fslist)
{
    // 读取 /etc/mtab (或 /proc/mounts)
    // 格式:
    // /dev/sda1 / ext4 rw,relatime,errors=remount-ro 0 0
    // proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0

    if (!(fp = setmntent(gMOUNTED, "r"))) {
        return errno;
    }

    while (getmntent_r(fp, &ent, buf, sizeof(buf))) {
        fsp = &fslist->data[fslist->number++];

        fsp->type = SIGAR_FSTYPE_UNKNOWN;
        SIGAR_SSTRCPY(fsp->dir_name, ent.mnt_dir);      // 挂载点
        SIGAR_SSTRCPY(fsp->dev_name, ent.mnt_fsname);  // 设备名
        SIGAR_SSTRCPY(fsp->sys_type_name, ent.mnt_type); // 文件系统类型
        SIGAR_SSTRCPY(fsp->options, ent.mnt_opts);      // 挂载选项

        sigar_fs_type_get(fsp);  // 根据 sys_type_name 设置 type
    }

    endmntent(fp);
    return SIGAR_OK;
}
```

#### 文件系统类型 (linux_sigar.c:1172-1225)

```c
int sigar_os_fs_type_get(sigar_file_system_t *fsp)
{
    char *type = fsp->sys_type_name;

    switch (*type) {
      case 'e':
        if (strnEQ(type, "ext", 3)) {
            fsp->type = SIGAR_FSTYPE_LOCAL_DISK;
        }
        break;
      case 'x':
        if (strEQ(type, "xfs") || strEQ(type, "xiafs")) {
            fsp->type = SIGAR_FSTYPE_LOCAL_DISK;
        }
        break;
      // ... 其他类型
    }

    return fsp->type;
}
```

#### 文件系统使用情况 (linux_sigar.c:1613-1632)

```c
int sigar_file_system_usage_get(sigar_t *sigar,
                                const char *dirname,
                                sigar_file_system_usage_t *fsusage)
{
    // 使用 statvfs() 获取使用情况
    int status = sigar_statvfs(sigar, dirname, fsusage);

    if (status != SIGAR_OK) {
        return status;
    }

    fsusage->use_percent = sigar_file_system_usage_calc_used(sigar, fsusage);

    // 获取磁盘 I/O 统计
    (void)sigar_disk_usage_get(sigar, dirname, &fsusage->disk);

    return SIGAR_OK;
}
```

### 6. 磁盘 I/O 监控

#### 磁盘统计 (linux_sigar.c:1519-1611)

```c
int sigar_disk_usage_get(sigar_t *sigar, const char *name,
                         sigar_disk_usage_t *disk)
{
    // 根据内核版本选择统计方法
    switch (sigar->iostat) {
      case IOSTAT_SYS:
        // 2.6 内核: /sys/block/<dev>/<dev><partition>/stat
        status = get_iostat_sys(sigar, name, disk, &iodev);
        break;
      case IOSTAT_DISKSTATS:
        // 2.6 内核: /proc/diskstats
        status = get_iostat_proc_dstat(sigar, name, disk, &iodev, &device_usage);
        break;
      case IOSTAT_PARTITIONS:
        // 2.4 内核: /proc/partitions
        status = get_iostat_procp(sigar, name, disk, &iodev);
        break;
      case IOSTAT_NONE:
      default:
        status = ENOENT;
        break;
    }

    // 计算派生指标
    if ((status == SIGAR_OK) && iodev) {
        sigar_uptime_get(sigar, &uptime);

        // 2.6 内核分区没有时间统计，使用整设备统计
        if (iodev->is_partition && (sigar->iostat == IOSTAT_DISKSTATS)) {
            partition_usage = disk;
            disk = &device_usage;
        }

        disk->snaptime = uptime.uptime;

        // 计算服务时间和队列长度
        interval = disk->snaptime - iodev->disk.snaptime;
        ios = (disk->reads - iodev->disk.reads) +
              (disk->writes - iodev->disk.writes);

        if (disk->time != SIGAR_FIELD_NOTIMPL) {
            tput = ((double)ios) * HZ / interval;
            util = ((double)(disk->time - iodev->disk.time)) / interval * HZ;
            disk->service_time = tput ? util / tput : 0.0;
        }

        if (disk->qtime != SIGAR_FIELD_NOTIMPL) {
            util = ((double)(disk->qtime - iodev->disk.qtime)) / interval;
            disk->queue = util / 1000.0;
        }
    }

    return status;
}
```

#### /proc/diskstats 方法 (linux_sigar.c:1322-1439)

```c
static int get_iostat_proc_dstat(sigar_t *sigar,
                                 const char *dirname,
                                 sigar_disk_usage_t *disk,
                                 sigar_iodev_t **iodev,
                                 sigar_disk_usage_t *device_usage)
{
    // 读取 /proc/diskstats
    // 格式:
    //   major minor reads reads_merged reads_sectors reads_ms writes writes_merged writes_sectors writes_ms ios_in_progress ms_weighted_ios
    //    8    0    123  456           7890          1234   567  890            12345        6789   10             12345

    while ((ptr = fgets(buffer, sizeof(buffer), fp))) {
        unsigned long major, minor;

        major = sigar_strtoul(ptr);
        minor = sigar_strtoull(ptr);

        if ((major == ST_MAJOR(sb)) &&
            ((minor == ST_MINOR(sb)) || (minor == 0)))
        {
            ptr = sigar_skip_token(ptr);  // name

            num = sscanf(ptr,
                         "%lu %lu %lu %lu "   // reads, reads_merged, reads_sectors, reads_ms
                         "%lu %lu %lu %lu "   // writes, writes_merged, writes_sectors, writes_ms
                         "%lu %lu %lu",       // ios_in_progress, ms_weighted_ios, ms_weighted_ios(aveq)
                         &rio, &rmerge, &rsect, &ruse,
                         &wio, &wmerge, &wsect, &wuse,
                         &running, &use, &aveq);

            if (num == 11) {  // 完整格式
                disk->rtime = ruse;
                disk->wtime = wuse;
                disk->time = use;
                disk->qtime = aveq;
                disk->ios = running;
            }
            else if (num == 4) {  // 旧格式
                wio = rsect;
                rsect = rmerge;
                wsect = ruse;
                disk->time = disk->qtime = SIGAR_FIELD_NOTIMPL;
            }

            disk->reads = rio;
            disk->writes = wio;
            disk->read_bytes  = rsect;
            disk->write_bytes = wsect;

            // 转换扇区为字节 (512 字节/扇区)
            disk->read_bytes  *= 512;
            disk->write_bytes *= 512;

            break;
        }
    }

    return status;
}
```

### 7. 系统信息

#### 发行版信息 (linux_sigar.c:3017-3079)

```c
static int get_linux_vendor_info(sigar_sys_info_t *info)
{
    // 支持的发行版:
    // - Fedora:      /etc/fedora-release
    // - SuSE:        /etc/SuSE-release
    // - Gentoo:      /etc/gentoo-release
    // - Slackware:   /etc/slackware-version
    // - Mandrake:    /etc/mandrake-release
    // - VMware:      /proc/vmware/version
    // - XenSource:   /etc/xensource-inventory
    // - Oracle:      /etc/oracle-release
    // - Red Hat:     /etc/redhat-release
    // - LSB:         /etc/lsb-release
    // - Debian:      /etc/debian_version

    linux_vendor_info_t linux_vendors[] = {
        { "Fedora",    "/etc/fedora-release", NULL },
        { "SuSE",      "/etc/SuSE-release", NULL },
        { "Gentoo",    "/etc/gentoo-release", NULL },
        { "Red Hat",   "/etc/redhat-release", redhat_vendor_parse },
        { "lsb",       "/etc/lsb-release", lsb_vendor_parse },
        { "Debian",    "/etc/debian_version", NULL },
        { NULL }
    };

    for (i=0; linux_vendors[i].name; i++) {
        vendor = &linux_vendors[i];

        if (stat(vendor->file, &sb) < 0) {
            continue;
        }

        status = sigar_file2str(vendor->file, buffer, sizeof(buffer)-1);
        break;
    }

    SIGAR_SSTRCPY(info->vendor, vendor->name);

    if (vendor->parse) {
        vendor->parse(data, info);
    }
    else {
        generic_vendor_parse(data, info);
    }

    return SIGAR_OK;
}
```

#### 系统启动时间 (linux_sigar.c:146-174)

```c
static int sigar_boot_time_get(sigar_t *sigar)
{
    FILE *fp;
    char buffer[BUFSIZ], *ptr;
    int found = 0;

    if (!(fp = fopen(gPROC_STAT, "r"))) {
        return errno;
    }

    // 读取 /proc/stat 中的 btime 字段
    while ((ptr = fgets(buffer, sizeof(buffer), fp))) {
        if (strnEQ(ptr, "btime", 5)) {
            if ((ptr = sigar_skip_token(ptr))) {
                sigar->boot_time = sigar_strtoul(ptr);
                found = 1;
            }
            break;
        }
    }

    fclose(fp);

    if (!found) {
        sigar->boot_time = time(NULL);  // 回退到当前时间
    }

    return SIGAR_OK;
}
```

#### 系统运行时间 (linux_sigar.c:519-532)

```c
int sigar_uptime_get(sigar_t *sigar, sigar_uptime_t *uptime)
{
    // 读取 /proc/uptime
    // 格式: <uptime_seconds> <idle_seconds>

    status = sigar_file2str(gPROC_UPTIME, buffer, sizeof(buffer));

    uptime->uptime = strtod(buffer, &ptr);

    return SIGAR_OK;
}
```

#### 系统负载 (linux_sigar.c:534-551)

```c
int sigar_loadavg_get(sigar_t *sigar, sigar_loadavg_t *loadavg)
{
    // 读取 /proc/loadavg
    // 格式: <load1> <load5> <load15> <running_tasks> <total_tasks> <last_pid>

    status = sigar_file2str(gPROC_LOADAVG, buffer, sizeof(buffer));

    loadavg->loadavg[0] = strtod(buffer, &ptr);  // 1 分钟平均
    loadavg->loadavg[1] = strtod(ptr, &ptr);     // 5 分钟平均
    loadavg->loadavg[2] = strtod(ptr, &ptr);     // 15 分钟平均
    loadavg->processor_queue = strtod(ptr, &ptr);  // 进程队列长度

    return SIGAR_OK;
}
```

#### 系统 UUID (linux_sigar.c:3081-3118)

```c
int sigar_sys_info_get_uuid(sigar_t *sigar, char uuid[SIGAR_SYS_INFO_LEN])
{
    // 尝试从 /etc/machine-id 读取 (systemd 系统)
    FILE *fp = fopen("/etc/machine-id", "r");
    if (fp) {
        if (fgets(uuid, SIGAR_SYS_INFO_LEN - 1, fp)) {
            found = 1;
        }
        fclose(fp);
    }
    else {
        // 回退到磁盘 UUID
        DIR *ctx = opendir("/dev/disk/by-uuid");
        if (ctx) {
            while ((data = readdir(ctx)) != NULL) {
                const char *fs_uuid = data->d_name;
                if (strlen(data->d_name) == 36 || strlen(data->d_name) == 22) {
                    strncat(uuid, fs_uuid, SIGAR_SYS_INFO_LEN - strlen(fs_uuid) - 1);
                    found = 1;
                    break;
                }
            }
            closedir(ctx);
        }
    }

    return found ? SIGAR_OK : SIGAR_ENOTIMPL;
}
```

#### 容器检测 (linux_sigar.c:3135-3164)

```c
int sigar_os_is_in_container(sigar_t *sigar)
{
    // 检查 /proc/1/cgroup
    // 非容器: 3:cpu:/
    // 容器:    3:cpu:/docker/58db180b111f8cbc4c08cb58c162c740c728c398cc54bec4586e38d058a028d6

    snprintf(buffer, sizeof(buffer), "%s/1/cgroup", PROC_FS_ROOT);

    if (!(fp = fopen(buffer, "r"))) {
        goto out;
    }

    while ((ptr = fgets(buffer, sizeof(buffer), fp))) {
        // OpenSUSE hack: 跳过包含 "=" 的行 (如 1:name=systemd:/system)
        tmp_ptr = strstr(buffer, "=");
        if (tmp_ptr) {
            continue;
        }

        tmp_ptr = strstr(buffer, DOCKER_KEY_STR);  // ":/docker/"
        if (tmp_ptr) {
            in_container = 1;
            break;
        }
    }

    fclose(fp);

out:
    return in_container;
}
```

## 性能优化

### 1. 进程信息缓存 (linux_sigar.c:718-726)

```c
time_t timenow = time(NULL);

if (pstat->pid == pid) {
    if ((timenow - pstat->mtime) < SIGAR_LAST_PROC_EXPIRE) {
        return SIGAR_OK;  // 60 秒内复用缓存
    }
}

pstat->pid = pid;
pstat->mtime = timenow;
```

### 2. 动态缓冲区管理

使用固定大小的栈缓冲区 (`BUFSIZ`) 读取 proc 文件，避免频繁内存分配。

### 3. 字符串解析优化

```c
// 跳过标记的宏
#define SIGAR_SKIP_SPACE(ptr) while (isspace(*ptr)) ++ptr
#define sigar_skip_token(ptr) (SIGAR_SKIP_SPACE(ptr), *ptr++)

// 跳过多个标记
ptr = sigar_skip_multiple_token(ptr, 4);
```

### 4. 延迟初始化

磁盘 I/O 统计方法在初始化时通过 `stat()` 检测，避免运行时重复检查。

### 5. 多级回退机制

- 内存信息: `/proc/meminfo` → `/proc/mtrr`
- 磁盘统计: `/proc/diskstats` → `/sys/block` → `/proc/partitions`
- 系统 UUID: `/etc/machine-id` → `/dev/disk/by-uuid`

## 特殊技术

### 1. 页面位移计算 (linux_sigar.c:36)

```c
#define pageshift(x) ((x) << sigar->pagesize)
```

将页数转换为字节数，避免乘法运算。

### 2. 十六进制转换 (linux_sigar.c:1824-1844)

```c
static SIGAR_INLINE unsigned int hex2int(const char *x, int len)
{
    int i;
    unsigned int j;

    for (i=0, j=0; i<len; i++) {
        register int ch = x[i];
        j <<= 4;
        if (isdigit(ch)) {
            j |= ch - '0';
        }
        else if (isupper(ch)) {
            j |= ch - ('A' - 10);
        }
        else {
            j |= ch - ('a' - 10);
        }
    }

    return j;
}
```

### 3. 进程名解析 (linux_sigar.c:958-971)

```c
// /proc/<pid>/stat 中的进程名可能被截断为 15 字符
if (strlen(pstat->name) == 15) {
    if (sigar_procfs_args_get(sigar, pid, &procargs) == SIGAR_OK &&
        procargs.number >= 1) {
        procname = procargs.data[0];  // 从命令行获取完整名称
    }
}
```

### 4. CPU 核心合并 (linux_sigar.c:492-500)

```c
if (core_rollup && (i % sigar->lcpu)) {
    // 合并超线程逻辑处理器
    cpu = &cpulist->data[cpulist->number-1];
}
else {
    // 创建新的物理核心条目
    SIGAR_CPU_LIST_GROW(cpulist);
    cpu = &cpulist->data[cpulist->number++];
}
```

### 5. 环境变量解析 (linux_sigar.c:1039-1064)

```c
while (ptr < end) {
    char *val = strchr(ptr, '=');

    if (val == NULL) {
        break;
    }

    klen = val - ptr;
    SIGAR_SSTRCPY(key, ptr);
    key[klen] = '\0';
    ++val;

    vlen = strlen(val);
    status = procenv->env_getter(procenv->data, key, klen, val, vlen);

    if (status != SIGAR_OK) {
        break;
    }

    ptr += (klen + 1 + vlen + 1);  // key= + value + \0
}
```

## 内核版本兼容性

### 1. Linux 2.2

- `/proc/stat` 包含磁盘统计
- 简化的 CPU 统计
- 无 `/proc/diskstats`

### 2. Linux 2.4

- `/proc/partitions` 包含磁盘统计
- 增强的 CPU 统计
- 无 `/proc/diskstats`

### 3. Linux 2.6

- `/proc/diskstats` 提供详细磁盘统计
- `/sys/block` 提供设备级统计
- 完整的 CPU 统计 (包括 iowait, irq, softirq, steal)
- `/proc/<pid>/io` 提供进程磁盘 I/O
- NPTL 支持

### 4. Linux 2.6.11+

- 新增 `steal` 字段 (虚拟化 CPU 时间)

### 5. Linux 2.6.20+

- `/proc/<pid>/io` 提供 I/O 统计

### 6. Linux 3.x+

- 改进的调度器统计
- 更多的 proc 文件

## 技术亮点

### 1. 统一的 procfs 接口

Linux 提供了 `/proc` 虚拟文件系统，所有系统信息都通过文件访问获取，无需调用特殊的系统 API。

### 2. 文本格式解析

所有信息都是纯文本格式，易于解析和调试。

### 3. 零拷贝设计

通过直接读取文件内容，避免了额外的系统调用和数据复制。

### 4. 自适应内核版本

根据内核版本和文件可用性自动选择最佳的信息源。

### 5. 高效的缓存机制

进程信息缓存减少了重复的文件读取。

### 6. 便携性

相同代码适用于各种 Linux 发行版和内核版本。

## 限制和挑战

### 1. 文件格式变化

不同内核版本的 proc 文件格式可能变化，需要维护兼容性。

### 2. 性能开销

频繁读取 proc 文件可能有性能开销，特别是在高频率监控时。

### 3. 权限要求

某些信息需要 root 权限才能访问 (如其他用户的进程环境变量)。

### 4. 线程检测

旧内核的线程模型复杂，需要通过 hack 检测线程。

### 5. 容器环境

容器环境中某些信息可能不准确或不可用。

## 总结

libsigar Linux 实现充分利用了 Linux 的 `/proc` 和 `/sys` 文件系统，提供了一个统一、高效、跨发行版的系统信息采集接口。其设计体现了以下特点：

1. **简洁性**: 通过文件系统接口获取信息，代码简洁易懂
2. **高效性**: 使用缓存和优化解析减少开销
3. **兼容性**: 支持从 2.2 到 3.x 的各种内核版本
4. **可扩展性**: 模块化设计，易于添加新功能
5. **健壮性**: 多级回退机制确保在各种环境都能工作

这种实现方式充分展示了 Linux 内核设计的优势，也为其他平台的系统监控提供了参考。
