# libsigar Solaris 平台源码分析

## 概述

libsigar 在 Solaris 平台的实现主要通过 `kstat` 库、`MIB2` 流接口和 `libproc` 库获取系统信息。Solaris 是 Sun Microsystems 开发的 UNIX 操作系统，以其强大的性能和可靠性著称。

## 核心架构

### sigar_t 结构体 (sigar_os.h:131-185)

```c
struct sigar_t {
    SIGAR_T_BASE;

    int solaris_version;              // Solaris 版本号
    int use_ucb_ps;                   // 是否使用 UCB ps

    int joyent;                       // Joyent SmartOS 标志
    zoneid_t zoneid;                 // Zone ID
    char *zonenm;                     // Zone 名称
    char *zonenm_short;               // Zone 短名称
    uint64_t cpu_prev_time, cpu_total;

    kstat_ctl_t *kc;                  // kstat 控制句柄

    // kstat 查找结果
    struct {
        kstat_t **cpu;                // CPU kstat 数组
        kstat_t **cpu_info;           // CPU 信息 kstat 数组
        processorid_t *cpuid;         // CPU ID 数组
        unsigned int lcpu;            // CPU 数组大小
        kstat_t *system;              // 系统 kstat
        kstat_t *syspages;            // 系统页面 kstat
        kstat_t *mempages;            // 内存页面 kstat
    } ks;

    // kstat 偏移量
    struct {
        int system[KSTAT_SYSTEM_MAX];
        int mempages[KSTAT_MEMPAGES_MAX];
        int syspages[KSTAT_SYSPAGES_MAX];
    } koffsets;

    int pagesize;

    time_t last_getprocs;
    sigar_pid_t last_pid;
    psinfo_t *pinfo;                  // 进程信息缓冲区
    sigar_cpu_list_t cpulist;

    // libproc.so 接口
    void *plib;
    proc_grab_func_t pgrab;
    proc_free_func_t pfree;
    proc_create_agent_func_t pcreate_agent;
    proc_destroy_agent_func_t pdestroy_agent;
    proc_objname_func_t pobjname;
    proc_dirname_func_t pdirname;
    proc_exename_func_t pexename;
    proc_fstat64_func_t pfstat64;
    proc_getsockopt_func_t pgetsockopt;
    proc_getsockname_func_t pgetsockname;

    sigar_cache_t *pargs;

    solaris_mib2_t mib2;              // MIB2 数据
};
```

### kstat 偏移量枚举

```c
// 系统统计
typedef enum {
    KSTAT_SYSTEM_BOOT_TIME,
    KSTAT_SYSTEM_LOADAVG_1,
    KSTAT_SYSTEM_LOADAVG_2,
    KSTAT_SYSTEM_LOADAVG_3,
    KSTAT_SYSTEM_MAX
} kstat_system_off_e;

// 内存页面统计
typedef enum {
    KSTAT_MEMPAGES_ANON,
    KSTAT_MEMPAGES_EXEC,
    KSTAT_MEMPAGES_VNODE,
    KSTAT_MEMPAGES_MAX
} kstat_mempages_off_e;

// 系统页面统计
typedef enum {
    KSTAT_SYSPAGES_FREE,
    KSTAT_SYSPAGES_MAX
} kstat_syspages_off_e;
```

### kstat 访问宏 (sigar_os.h:193-222)

```c
#define kSTAT_exists(v, type) \
    (sigar->koffsets.type[v] != -2)

#define kSTAT_ptr(v, type) \
    ((kstat_named_t *)ksp->ks_data + sigar->koffsets.type[v])

#define kSTAT_uint(v, type) \
    (kSTAT_exists(v, type) ? kSTAT_ptr(v, type)->value.KSTAT_UINT : 0)

#define kSYSTEM(v) kSTAT_ui32(v, system)
#define kMEMPAGES(v) kSTAT_uint(v, mempages)
#define kSYSPAGES(v) kSTAT_uint(v, syspages)
```

## 核心技术

### 1. kstat 库

`kstat` 是 Solaris 提供的内核统计数据访问接口。

#### 初始化 (solaris_sigar.c:69-164)

```c
int sigar_os_open(sigar_t **sigar)
{
    kstat_ctl_t *kc;
    kstat_t *ksp;
    struct utsname name;
    char *ptr;
    char zonenm[256];

    // 1. 打开 kstat
    if ((kc = kstat_open()) == NULL) {
        *sig = NULL;
        return errno;
    }

    // 2. 分配并初始化 sigar 结构
    if ((*sig = sigar = calloc(1, sizeof(*sigar))) == NULL) {
        return ENOMEM;
    }

    // 3. 获取 Zone 信息
    sigar->zoneid = getzoneid();
    if (sigar->zoneid < 0) {
        sigar->zoneid = 0;
        sigar->zonenm = strdup("unknown");
        sigar->zonenm_short = strdup("unknown");
    } else {
        getzonenamebyid(sigar->zoneid, zonenm, sizeof(zonenm));
        sigar->zonenm = strdup(zonenm);
        ptr = strchr(zonenm, '-');
        if (ptr) *ptr = 0;
        ptr = strchr(zonenm, ':');
        if (ptr) *ptr = 0;
        sigar->zonenm_short = strdup(zonenm);
    }

    // 4. 获取 Solaris 版本
    uname(&name);
    if ((ptr = strchr(name.release, '.'))) {
        sigar->solaris_version = atoi(ptr + 1);
    }
    else {
        sigar->solaris_version = 6;
    }

    // 5. 检测 Joyent SmartOS
    sigar->joyent = !strncmp(name.version, "joyent", 6);

    // 6. 检测 UCB ps
    if ((ptr = getenv("SIGAR_USE_UCB_PS"))) {
        sigar->use_ucb_ps = strEQ(ptr, "true");
    }

    if (sigar->use_ucb_ps) {
        if (access(SIGAR_USR_UCB_PS, X_OK) == -1) {
            sigar->use_ucb_ps = 0;
        }
        else {
            sigar->use_ucb_ps = 1;
        }
    }

    // 7. 计算页大小位移
    sigar->pagesize = 0;
    i = sysconf(_SC_PAGESIZE);
    while ((i >>= 1) > 0) {
        sigar->pagesize++;
    }

    sigar->ticks = sysconf(_SC_CLK_TCK);
    sigar->kc = kc;

    // 8. 获取 kstat
    if ((status = sigar_get_kstats(sigar)) != SIGAR_OK) {
        fprintf(stderr, "status=%d\n", status);
    }

    // 9. 初始化偏移量并获取启动时间
    if ((ksp = sigar->ks.system) &&
        (kstat_read(kc, ksp, NULL) >= 0))
    {
        sigar_koffsets_init_system(sigar, ksp);
        sigar->boot_time = kSYSTEM(KSTAT_SYSTEM_BOOT_TIME);
    }

    return SIGAR_OK;
}
```

#### kstat 查找 (solaris_sigar.c:51-67)

```c
static kstat_t *
kstat_next(kstat_t *ksp, char *ks_module, int ks_instance, char *ks_name)
{
    if (ksp) {
        ksp = ksp->ks_next;
    }
    for (; ksp; ksp = ksp->ks_next) {
        if ((ks_module == NULL ||
             strcmp(ksp->ks_module, ks_module) == 0) &&
            (ks_instance == -1 || ksp->ks_instance == ks_instance) &&
            (ks_name == NULL || strcmp(ksp->ks_name, ks_name) == 0))
            return ksp;
    }

    errno = ENOENT;
    return NULL;
}
```

### 2. MIB2 流接口

Solaris 使用 MIB2 (Management Information Base 2) 流接口获取网络统计信息。

#### MIB2 访问 (get_mib2.c:106-262)

```c
int
get_mib2(solaris_mib2_t *mib2,
         struct opthdr **opt,
         char **data,
         int *datalen)
{
    struct strbuf d;
    int err;
    int f;
    int rc;

    // 如果 MIB2 访问未打开，打开它
    if (mib2->sd < 0) {
        if ((err = open_mib2(mib2))) {
            return(err);
        }

        // 设置消息请求和选项
        mib2->req = (struct T_optmgmt_req *)mib2->smb;
        mib2->op = (struct opthdr *)&mib2->smb[sizeof(struct T_optmgmt_req)];
        mib2->req->PRIM_type = T_OPTMGMT_REQ;
        mib2->req->OPT_offset = sizeof(struct T_optmgmt_req);
        mib2->req->OPT_length = sizeof(struct opthdr);
        mib2->req->MGMT_flags = MI_T_CURRENT;
        mib2->op->level = MIB2_IP;
        mib2->op->name = mib2->op->len = 0;

        // 发送消息
        if (putmsg(mib2->sd, &mib2->ctlbuf, (struct strbuf *)NULL, 0) == -1) {
            return(GET_MIB2_ERR_PUTMSG);
        }
    }

    // 获取下一个回复消息
    f = 0;
    if ((rc = getmsg(mib2->sd, &mib2->ctlbuf, NULL, &f)) < 0) {
        return(GET_MIB2_ERR_GETMSGR);
    }

    // 检查数据结束
    if (rc == 0
        && mib2->ctlbuf.len >= sizeof(struct T_optmgmt_ack)
        && mib2->op_ack->PRIM_type == T_OPTMGMT_ACK
        && mib2->op_ack->MGMT_flags == T_SUCCESS
        && mib2->op->len == 0)
    {
        err = close_mib2(mib2);
        if (err) {
            return(err);
        }
        return(GET_MIB2_EOD);
    }

    // 分配数据缓冲区
    if (mib2->op->len >= mib2->db_len) {
        mib2->db_len = mib2->op->len;
        if (mib2->db == NULL) {
            mib2->db = (char *)malloc(mib2->db_len);
        }
        else {
            mib2->db = (char *)realloc(mib2->db, mib2->db_len);
        }
    }

    // 获取数据部分
    d.maxlen = mib2->op->len;
    d.buf = mib2->db;
    d.len = 0;
    f = 0;
    if ((rc = getmsg(mib2->sd, NULL, &d, &f)) < 0) {
        return(GET_MIB2_ERR_GETMSGD);
    }

    *opt = mib2->op;
    *data = mib2->db;
    *datalen = d.len;
    return(GET_MIB2_OK);
}
```

#### MIB2 打开 (get_mib2.c:275-321)

```c
int
open_mib2(solaris_mib2_t *mib2)
{
    if (mib2->sd >= 0) {
        return(GET_MIB2_ERR_OPEN);
    }

    // 打开 ARP 流设备，推送 TCP 和 UDP
    if ((mib2->sd = open(GET_MIB2_ARPDEV, O_RDWR, 0600)) < 0) {
        return(GET_MIB2_ERR_ARPOPEN);
    }

    if (ioctl(mib2->sd, I_PUSH, GET_MIB2_TCPSTREAM) == -1) {
        return(GET_MIB2_ERR_TCPPUSH);
    }

    if (ioctl(mib2->sd, I_PUSH, GET_MIB2_UDPSTREAM) == -1) {
        return(GET_MIB2_ERR_UDPPUSH);
    }

    // 分配流消息缓冲区
    mib2->smb_len = sizeof(struct opthdr) + sizeof(struct T_optmgmt_req);
    if (mib2->smb_len < (sizeof(struct opthdr) + sizeof(struct T_optmgmt_ack))) {
        mib2->smb_len = sizeof(struct opthdr) + sizeof(struct T_optmgmt_ack);
    }
    if (mib2->smb_len < sizeof(struct T_error_ack)) {
        mib2->smb_len = sizeof(struct T_error_ack);
    }
    if ((mib2->smb = (char *)malloc(mib2->smb_len)) == NULL) {
        return(GET_MIB2_ERR_NOSPC);
    }

    return(GET_MIB2_OK);
}
```

### 3. libproc 库

Solaris 的 `libproc` 库提供了进程查询和控制功能。

#### 动态加载 libproc

```c
sigar->plib = dlopen("libproc.so", RTLD_LAZY);

// 加载函数指针
sigar->pgrab = dlsym(sigar->plib, "Pgrab");
sigar->pfree = dlsym(sigar->plib, "Pfree");
sigar->pcreate_agent = dlsym(sigar->plib, "Pcreate_agent");
sigar->pdestroy_agent = dlsym(sigar->plib, "Pdestroy_agent");
// ... 其他函数
```

### 4. Zone 支持

Solaris Zones 是操作系统级虚拟化技术。

```c
// 获取 Zone ID
sigar->zoneid = getzoneid();

// 获取 Zone 名称
getzonenamebyid(sigar->zoneid, zonenm, sizeof(zonenm));
```

## 核心功能模块

### 1. 内存监控

#### 内存信息获取 (solaris_sigar.c:272-332)

```c
int sigar_mem_get(sigar_t *sigar, sigar_mem_t *mem)
{
    kstat_ctl_t *kc = sigar->kc;
    kstat_t *ksp;
    sigar_uint64_t kern = 0;

    SIGAR_ZERO(mem);

    // 获取物理内存总量
    mem->total = sysconf(_SC_PHYS_PAGES);
    mem->total <<= sigar->pagesize;

    if (sigar_kstat_update(sigar) == -1) {
        return errno;
    }

    // Zone 感知处理
    if (getzoneid() != GLOBAL_ZONEID) {
        return zone_mem_get(sigar, mem);
    }

    // 获取空闲内存
    if ((ksp = sigar->ks.syspages) && kstat_read(kc, ksp, NULL) >= 0) {
        sigar_koffsets_init_syspages(sigar, ksp);
        mem->free = kSYSPAGES(KSTAT_SYSPAGES_FREE);
        mem->free <<= sigar->pagesize;
        mem->used = mem->total - mem->free;
    }

    // 获取内存页面统计
    if ((ksp = sigar->ks.mempages) && kstat_read(kc, ksp, NULL) >= 0) {
        sigar_koffsets_init_mempages(sigar, ksp);
    }

    // ZFS ARC 缓存
    if ((ksp = kstat_lookup(sigar->kc, "zfs", 0, "arcstats")) &&
        (kstat_read(sigar->kc, ksp, NULL) != -1))
    {
        kstat_named_t *kn;

        if ((kn = (kstat_named_t *)kstat_data_lookup(ksp, "size"))) {
            kern = kn->value.i64;
        }
        if ((kn = (kstat_named_t *)kstat_data_lookup(ksp, "c_min"))) {
            // c_min 不能回收
            if (kern > kn->value.i64) {
                kern -= kn->value.i64;
            }
        }
    }

    mem->actual_free = mem->free + kern;
    mem->actual_used = mem->used - kern;

    sigar_mem_calc_ram(sigar, mem);

    return SIGAR_OK;
}
```

#### Zone 内存 (solaris_sigar.c:223-270)

```c
static int zone_mem_get(sigar_t *sigar, sigar_mem_t *mem)
{
    kstat_t *ksp;
    sigar_vmusage64_t result;
    size_t nres = 1;
    sigar_uint64_t used = 0, cap = 0;
    int ret;

    // 尝试使用 memory_cap kstat (SmartOS 特有)
    if ((ksp = kstat_lookup(sigar->kc, "memory_cap", -1, NULL)) &&
        (kstat_read(sigar->kc, ksp, NULL) != -1))
    {
        kstat_named_t *kn;

        if ((kn = (kstat_named_t *)kstat_data_lookup(ksp, "rss"))) {
            used = kn->value.i64;
        }
        if ((kn = (kstat_named_t *)kstat_data_lookup(ksp, "physcap"))) {
            cap = kn->value.i64;
        }
    }
    else {
        // 回退到 getvmusage()
        ret = sysinfo(SI_ARCHITECTURE_64, path, sizeof(path));
        if (ret < 0) {
            return -1;
        }

        if (getvmusage(VMUSAGE_ZONE, VMUSAGE_INTERVAL, (vmusage_t *)&result,
            &nres) != 0 || nres != 1) {
            return -1;
        }
        used = result.vmu_rss_all;
        cap = mem->total;
    }

    mem->actual_free = mem->free = cap - used;
    mem->actual_used = mem->used = used;

    sigar_mem_calc_ram(sigar, mem);

    return SIGAR_OK;
}
```

### 2. 交换空间监控 (solaris_sigar.c:334-402)

```c
int sigar_swap_get(sigar_t *sigar, sigar_swap_t *swap)
{
    kstat_t *ksp;
    kstat_named_t *kn;
    swaptbl_t *stab;
    int num, i;
    char path[PATH_MAX+1];

    // 获取交换空间数量
    if ((num = swapctl(SC_GETNSWP, NULL)) == -1) {
        return errno;
    }

    stab = malloc(num * sizeof(stab->swt_ent[0]) + sizeof(*stab));

    stab->swt_n = num;
    for (i=0; i<num; i++) {
        stab->swt_ent[i].ste_path = path;
    }

    // 获取交换空间列表
    if ((num = swapctl(SC_LIST, stab)) == -1) {
        free(stab);
        return errno;
    }

    num = num < stab->swt_n ? num : stab->swt_n;
    swap->total = swap->free = 0;

    for (i=0; i<num; i++) {
        if (stab->swt_ent[i].ste_flags & ST_INDEL) {
            continue;  // 交换文件正在删除
        }
        swap->total += stab->swt_ent[i].ste_pages;
        swap->free += stab->swt_ent[i].ste_free;
    }
    free(stab);

    swap->total <<= sigar->pagesize;
    swap->free <<= sigar->pagesize;
    swap->used = swap->total - swap->free;

    // 获取分页统计
    if (sigar_kstat_update(sigar) == -1) {
        return errno;
    }
    if (!(ksp = kstat_lookup(sigar->kc, "cpu", -1, "vm"))) {
        swap->page_in = swap->page_out = SIGAR_FIELD_NOTIMPL;
        return SIGAR_OK;
    }

    swap->page_in = swap->page_out = 0;

    do {
        if (kstat_read(sigar->kc, ksp, NULL) < 0) {
            break;
        }

        if ((kn = (kstat_named_t *)kstat_data_lookup(ksp, "pgin"))) {
            swap->page_in += kn->value.i64;
        }
        if ((kn = (kstat_named_t *)kstat_data_lookup(ksp, "pgout"))) {
            swap->page_out += kn->value.i64;
        }
    } while ((ksp = kstat_next(ksp, "cpu", -1, "vm")));

    return SIGAR_OK;
}
```

### 3. CPU 监控 (solaris_sigar.c:476-500+)

```c
int sigar_cpu_get(sigar_t *sigar, sigar_cpu_t *cpu)
{
    int status, i;

    status = sigar_cpu_list_get(sigar, &sigar->cpulist);

    if (status != SIGAR_OK) {
        return status;
    }

    SIGAR_ZERO(cpu);

    for (i=0; i<sigar->cpulist.number; i++) {
        sigar_cpu_t *xcpu = &sigar->cpulist.data[i];

        cpu->user  += xcpu->user;
        cpu->sys   += xcpu->sys;
        cpu->idle  += xcpu->idle;
        cpu->wait  += xcpu->wait;
        cpu->total += xcpu->total;
    }

    return SIGAR_OK;
}
```

### 4. CPU 信息

#### CPU 品牌 (solaris_sigar.c:409-453)

```c
static int get_chip_brand(sigar_t *sigar, int processor,
                          sigar_cpu_info_t *info)
{
    kstat_t *ksp = sigar->ks.cpu_info[processor];
    kstat_named_t *brand;

    if (sigar->solaris_version < 10) {
        return 0;  // Solaris 9 不支持
    }

    if (ksp &&
        (kstat_read(sigar->kc, ksp, NULL) != -1) &&
        (brand = (kstat_named_t *)kstat_data_lookup(ksp, "brand")))
    {
        char *name = KSTAT_NAMED_STR_PTR(brand);
        char *vendor = "Sun";
        char *vendors[] = {"Intel", "AMD", NULL};
        int i;

        if (!name) {
            return 0;
        }

        // 检测供应商
        for (i=0; vendors[i]; i++) {
            if (strstr(name, vendors[i])) {
                vendor = vendors[i];
                break;
            }
        }

        SIGAR_SSTRCPY(info->vendor, vendor);
        return 1;
    }
    else {
        return 0;
    }
}
```

## 性能优化

### 1. kstat 缓存

缓存 kstat 查找结果，避免重复查找。

### 2. 批量更新

使用 `sigar_kstat_update` 批量更新 kstat 数据。

### 3. Zone 感知

区分全局 Zone 和非全局 Zone 的处理。

## 特殊技术

### 1. Solaris 11 MIB2 兼容性 (solaris_sigar.c:48)

```c
/*
 * Hack to correctly walk the MIB2 TCP Connection structures
 * for binaries compiled on Sparc 10 running on Sparc 11
 */
#define SOLARIS_11_MIB2_TCP_CONN_SIZE 80
```

### 2. ZFS ARC 统计

```c
if ((ksp = kstat_lookup(sigar->kc, "zfs", 0, "arcstats")) &&
    (kstat_read(sigar->kc, ksp, NULL) != -1))
{
    kstat_named_t *kn;

    if ((kn = (kstat_named_t *)kstat_data_lookup(ksp, "size"))) {
        kern = kn->value.i64;
    }
    if ((kn = (kstat_named_t *)kstat_data_lookup(ksp, "c_min"))) {
        if (kern > kn->value.i64) {
            kern -= kn->value.i64;
        }
    }
}
```

### 3. Joyent SmartOS 支持

```c
sigar->joyent = !strncmp(name.version, "joyent", 6);
```

### 4. libproc 动态加载

```c
sigar->plib = dlopen("libproc.so", RTLD_LAZY);
```

## 技术亮点

### 1. kstat 统一接口

`kstat` 提供了统一的内核统计数据访问接口。

### 2. MIB2 流接口

使用流接口获取网络统计信息。

### 3. Zone 虚拟化

支持 Solaris Zones 的资源监控。

### 4. ZFS 支持

集成 ZFS 文件系统的 ARC 缓存统计。

### 5. SmartOS 兼容性

支持 Joyent SmartOS 平台。

## 限制和挑战

### 1. 版本兼容性

不同 Solaris 版本之间 kstat 结构有差异。

### 2. MIB2 复杂性

MIB2 流接口编程复杂，容易出错。

### 3. libproc 版本

不同 Solaris 版本 libproc 接口有变化。

### 4. Zone 权限

非全局 Zone 某些信息可能不可用。

## 总结

libsigar Solaris 实现充分利用了 Solaris 的 `kstat`、`MIB2` 流接口和 `libproc` 库，提供了全面的系统监控功能。其设计特点包括：

1. **kstat 接口**: 统一的内核统计数据访问
2. **MIB2 流**: 复杂但功能强大的网络统计接口
3. **Zone 支持**: 完整的虚拟化资源监控
4. **ZFS 集成**: 支持现代文件系统统计
5. **动态加载**: libproc 等库的动态加载

这种实现方式展示了 Solaris 系统架构的特点，强调了高性能和可靠性。
