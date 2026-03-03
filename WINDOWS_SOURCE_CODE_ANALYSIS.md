# libsigar Windows 分支源码深度解析

## 目录
1. [概述](#概述)
2. [核心架构](#核心架构)
3. [模块组织](#模块组织)
4. [关键技术实现](#关键技术实现)
5. [API调用分析](#api调用分析)
6. [核心功能模块](#核心功能模块)
7. [性能优化机制](#性能优化机制)
8. [特殊技术](#特殊技术)

---

## 概述

libsigar (System Information Gatherer And Reporter) 的 Windows 分支实现了一套完整的跨平台系统信息收集接口。Windows 实现主要通过 Windows API、性能计数器 (Performance Counters)、WMI (Windows Management Instrumentation) 以及各种系统 DLL 动态加载来获取系统信息。

### 关键文件结构

```
src/os/win32/
├── win32_sigar.c       # 主要实现文件 (113KB, 4300+行)
├── sigar_os.h          # Windows特定头文件定义
├── sigar_pdh.h         # 性能数据助手宏定义
├── peb.c               # 进程环境块(PEB)相关操作
├── wmi.cpp             # WMI接口封装 (C++)
└── sigar.rc.in         # 资源文件模板
```

---

## 核心架构

### 1. sigar_t 结构体 (Windows 扩展)

```c
struct sigar_t {
    SIGAR_T_BASE;              // 基础字段
    char *machine;             // 目标机器名称
    int using_wide;            // 是否使用宽字符
    long pagesize;             // 页面大小

    // 性能计数器相关
    HKEY handle;               // 注册表句柄 (HKEY_PERFORMANCE_DATA)
    char *perfbuf;             // 性能计数器缓冲区
    DWORD perfbuf_size;        // 缓冲区大小

    // DLL 模块句柄集合
    sigar_wmi_handle_t *wmi_handle;    // WMI 句柄
    sigar_wtsapi_t wtsapi;             // 终端服务API
    sigar_iphlpapi_t iphlpapi;         // IP助手API
    sigar_advapi_t advapi;             // 高级API
    sigar_ntdll_t ntdll;               // NT DLL
    sigar_psapi_t psapi;               // 进程状态API
    sigar_winsta_t winsta;             // 工作站API
    sigar_kernel_t kernel;             // 内核API
    sigar_mpr_t mpr;                   // 多提供商路由器

    // 缓存和状态
    sigar_win32_pinfo_t pinfo;         // 进程信息缓存
    sigar_cache_t *netif_adapters;     // 网卡适配器缓存
    sigar_cache_t *netif_mib_rows;     // MIB行缓存
    sigar_cache_t *netif_addr_rows;    // 地址行缓存
    sigar_cache_t *netif_names;        // 网卡名称缓存

    int netif_name_short;              // 网卡名称简化模式
    WORD ws_version;                   // Winsock版本
    int ws_error;                      // Winsock错误
    int ht_enabled;                    // 超线程启用
    int lcpu;                          // 逻辑CPU数量
    int winnt;                         // 是否为Windows NT
};
```

### 2. DLL 动态加载机制

libsigar 使用动态加载来确保向后兼容性，避免旧版本 Windows API 调用失败：

```c
typedef struct {
    const char *name;      // DLL名称
    HINSTANCE handle;      // 加载的句柄
} sigar_dll_handle_t;

typedef struct {
    const char *name;      // 函数名
    FARPROC func;          // 函数指针
} sigar_dll_func_t;

typedef struct {
    sigar_dll_handle_t handle;
    sigar_dll_func_t funcs[12];  // 函数数组
} sigar_dll_module_t;
```

#### 加载的 DLL 列表：

| DLL | 主要用途 | 关键函数 |
|-----|---------|---------|
| **wtsapi32.dll** | 终端服务会话管理 | WTSEnumerateSessions, WTSQuerySessionInformation |
| **iphlpapi.dll** | IP助手功能 | GetAdaptersInfo, GetTcpTable, GetIpAddrTable |
| **advapi32.dll** | 高级安全和服务管理 | ConvertStringSidToSid, QueryServiceStatusEx |
| **ntdll.dll** | NT底层系统信息 | NtQuerySystemInformation, NtQueryInformationProcess |
| **psapi.dll** | 进程状态API | EnumProcesses, EnumProcessModules |
| **winsta.dll** | 工作站信息 | WinStationQueryInformationW |
| **kernel32.dll** | 内核功能 | GlobalMemoryStatusEx |
| **mpr.dll** | 多提供商路由器 | WNetGetConnectionA |

---

## 模块组织

### 1. 初始化和清理

#### sigar_os_open() - 初始化函数

```c
int sigar_os_open(sigar_t **sigar_ptr) {
    sigar_t *sigar = calloc(1, sizeof(*sigar));

    // 连接到性能数据注册表
    RegConnectRegistryA(sigar->machine, HKEY_PERFORMANCE_DATA, &sigar->handle);

    // 获取系统信息
    GetSystemInfo(&sysinfo);
    sigar->ncpu = sysinfo.dwNumberOfProcessors;
    sigar->pagesize = sysinfo.dwPageSize;

    // 初始化所有DLL模块
    DLLMOD_COPY(wtsapi);
    DLLMOD_COPY(iphlpapi);
    // ... 其他DLL

    // 启用调试权限以增加进程可见性
    sigar_enable_privilege(SE_DEBUG_NAME);

    // 初始化WMI连接
    sigar->wmi_handle = (sigar_wmi_handle_t *)wmi_handle_open(&wmi_status);

    // 初始化网络接口缓存
    sigar->netif_mib_rows = NULL;
    sigar->netif_addr_rows = NULL;
    sigar->netif_adapters = NULL;
    sigar->netif_names = NULL;
}
```

#### sigar_os_close() - 清理函数

```c
int sigar_os_close(sigar_t *sigar) {
    // 释放所有DLL
    DLLMOD_FREE(wtsapi);
    DLLMOD_FREE(iphlpapi);
    // ... 其他DLL

    // 释放缓存
    if (sigar->netif_mib_rows) {
        sigar_cache_destroy(sigar->netif_mib_rows);
    }

    // 关闭注册表句柄
    RegCloseKey(sigar->handle);

    // 清理Winsock
    if (sigar->ws_version != 0) {
        WSACleanup();
    }

    free(sigar);
}
```

---

## 关键技术实现

### 1. 性能计数器 (Performance Counters)

libsigar 通过访问 `HKEY_PERFORMANCE_DATA` 注册表键来获取 Windows 性能计数器数据。

#### 性能计数器缓冲区管理

```c
#define PERFBUF_SIZE 8192

static DWORD perfbuf_init(sigar_t *sigar) {
    if (!sigar->perfbuf) {
        sigar->perfbuf = calloc(1, PERFBUF_SIZE);
        sigar->perfbuf_size = PERFBUF_SIZE;
    }
    return sigar->perfbuf_size;
}

static DWORD perfbuf_grow(sigar_t *sigar) {
    sigar->perfbuf_size += PERFBUF_SIZE;
    sigar->perfbuf = realloc(sigar->perfbuf, sigar->perfbuf_size);
    return sigar->perfbuf_size;
}
```

#### 性能计数器访问宏

```c
#define MyRegQueryValue() \
    (USING_WIDE() ? \
        RegQueryValueExW(sigar->handle, \
                         wcounter_key, NULL, &type, \
                         (LPBYTE)sigar->perfbuf, \
                         &bytes) : \
        RegQueryValueExA(sigar->handle, \
                         counter_key, NULL, &type, \
                         (LPVOID)sigar->perfbuf, \
                         &bytes))
```

#### 主要性能计数器键值

| 计数器键 | 标题索引 | 用途 |
|---------|---------|------|
| "2" | PERF_TITLE_SYS_KEY | 系统信息 |
| "4" | PERF_TITLE_MEM_KEY | 内存信息 |
| "230" | PERF_TITLE_PROC_KEY | 进程信息 |
| "238" | PERF_TITLE_CPU_KEY | CPU信息 |
| "236" | PERF_TITLE_DISK_KEY | 磁盘信息 |

#### 性能计数器偏移量定义

```c
// CPU 计数器
#define PERF_TITLE_CPU_USER    142
#define PERF_TITLE_CPU_IDLE    1746
#define PERF_TITLE_CPU_SYS     144
#define PERF_TITLE_CPU_IRQ     698

// 进程计数器
#define PERF_TITLE_CPUTIME    6
#define PERF_TITLE_PAGE_FAULTS 28
#define PERF_TITLE_MEM_VSIZE  174
#define PERF_TITLE_MEM_SIZE   180
#define PERF_TITLE_THREAD_CNT 680
#define PERF_TITLE_HANDLE_CNT 952
#define PERF_TITLE_PID        784
#define PERF_TITLE_PPID       1410
#define PERF_TITLE_PRIORITY   682
#define PERF_TITLE_START_TIME 684
#define PERF_TITLE_IO_READ_BYTES_SEC 1420
#define PERF_TITLE_IO_WRITE_BYTES_SEC 1422

// 磁盘计数器
#define PERF_TITLE_DISK_TIME 200
#define PERF_TITLE_DISK_READ_TIME 202
#define PERF_TITLE_DISK_WRITE_TIME 204
#define PERF_TITLE_DISK_READ  214
#define PERF_TITLE_DISK_WRITE 216
#define PERF_TITLE_DISK_READ_BYTES  220
#define PERF_TITLE_DISK_WRITE_BYTES 222
#define PERF_TITLE_DISK_QUEUE 198
```

### 2. NT系统调用 (NtQuerySystemInformation)

对于某些 Windows NT/2000 系统或需要更底层的系统信息，libsigar 使用未文档化的 NT API：

```c
#define SPPI_MAX 128

typedef struct {
    LARGE_INTEGER IdleTime;
    LARGE_INTEGER KernelTime;
    LARGE_INTEGER UserTime;
    LARGE_INTEGER DpcTime;
    LARGE_INTEGER InterruptTime;
    ULONG InterruptCount;
} SYSTEM_PROCESSOR_PERFORMANCE_INFORMATION;

#define SystemProcessorPerformanceInformation 8

static int sigar_cpu_ntsys_get(sigar_t *sigar, sigar_cpu_t *cpu) {
    SYSTEM_PROCESSOR_PERFORMANCE_INFORMATION info[SPPI_MAX];

    sigar_NtQuerySystemInformation(SystemProcessorPerformanceInformation,
                                   &info, sizeof(info), &retval);

    if (!retval) {
        return GetLastError();
    }

    num = retval/sizeof(info[0]);
    SIGAR_ZERO(cpu);

    for (i=0; i<num; i++) {
        cpu->idle += NS100_2MSEC(info[i].IdleTime.QuadPart);
        cpu->user += NS100_2MSEC(info[i].UserTime.QuadPart);
        cpu->sys  += NS100_2MSEC(info[i].KernelTime.QuadPart -
                                 info[i].IdleTime.QuadPart);
        cpu->irq  += NS100_2MSEC(info[i].InterruptTime.QuadPart);
    }
    cpu->total = cpu->idle + cpu->user + cpu->sys;

    return SIGAR_OK;
}
```

### 3. 进程环境块 (PEB) 操作

PEB (Process Environment Block) 包含进程的运行时信息，包括命令行参数、工作目录等。

#### peb.c 实现

```c
typedef struct _PEB {
    BYTE Reserved1[2];
    BYTE BeingDebugged;
    BYTE Reserved2[1];
    PVOID Reserved3[2];
    PPEB_LDR_DATA Ldr;
    PRTL_USER_PROCESS_PARAMETERS ProcessParameters;
    BYTE Reserved4[104];
    PVOID Reserved5[52];
    BYTE Reserved6[128];
    PVOID Reserved7[1];
    ULONG SessionId;
} PEB, *PPEB;

typedef struct _PROCESS_BASIC_INFORMATION {
    PVOID Reserved1;
    PPEB PebBaseAddress;
    PVOID Reserved2[2];
    UINT_PTR UniqueProcessId;
    PVOID Reserved3;
} PROCESS_BASIC_INFORMATION;

static int sigar_pbi_get(sigar_t *sigar, HANDLE proc, PEB *peb) {
    PROCESS_BASIC_INFORMATION pbi;
    DWORD size = sizeof(pbi);

    sigar_NtQueryInformationProcess(proc,
                                    ProcessBasicInformation,
                                    &pbi, size, NULL);

    if (!pbi.PebBaseAddress) {
        return ERROR_DATATYPE_MISMATCH;  // 可能是32位/64位不匹配
    }

    if (ReadProcessMemory(proc, pbi.PebBaseAddress, peb, size, NULL)) {
        return SIGAR_OK;
    }
    else {
        return GetLastError();
    }
}
```

### 4. WMI 接口 (wmi.cpp)

WMI 提供了更高级的接口来获取系统信息，特别是在某些性能计数器不可用时。

#### WMI 初始化

```cpp
sigar_wmi_handle_t * wmi_handle_open(int *error) {
    sigar_wmi_handle_t *handle = (sigar_wmi_handle_t *)calloc(1, sizeof (*handle));

    // 初始化COM
    HRESULT hres = CoInitializeEx(NULL, COINIT_MULTITHREADED);
    if (FAILED(hres)) goto err;

    // 设置COM安全级别
    hres = CoInitializeSecurity(NULL, -1, NULL, NULL,
                                RPC_C_AUTHN_LEVEL_CONNECT,
                                RPC_C_IMP_LEVEL_IMPERSONATE,
                                NULL, EOAC_NONE, 0);
    if (FAILED(hres) && hres != RPC_E_TOO_LATE) goto err;

    // 创建WMI定位器
    hres = CoCreateInstance(CLSID_WbemLocator, NULL, CLSCTX_ALL,
                           IID_PPV_ARGS(&handle->locator));
    if (FAILED(hres)) goto err;

    // 连接到WMI服务
    wchar_t root[] = L"root\\CIMV2";
    hres = handle->locator->ConnectServer(root, NULL, NULL, NULL,
                                          WBEM_FLAG_CONNECT_USE_MAX_WAIT,
                                          NULL, NULL, &handle->services);
    if (FAILED(hres)) goto err;

    return handle;
}
```

#### WMI 查询

```cpp
HRESULT wmi_get_proc_string_property(sigar_t *sigar, DWORD pid,
                                     TCHAR * name, TCHAR * value, DWORD len) {
    IWbemClassObject *obj;
    VARIANT var;
    wchar_t query[56];
    wsprintf(query, L"Win32_Process.Handle=%d", pid);

    result = sigar->wmi_handle->services->GetObject(query, 0, 0, &obj, 0);
    if (FAILED(result)) return result;

    result = obj->Get(name, 0, &var, 0, 0);
    if (SUCCEEDED(result)) {
        if (var.vt == VT_NULL) {
            result = E_INVALIDARG;
        } else {
            lstrcpyn(value, var.bstrVal, len);
        }
        VariantClear(&var);
    }

    obj->Release();
    return result;
}
```

---

## API调用分析

### 1. 内存信息获取

```c
SIGAR_DECLARE(int) sigar_mem_get(sigar_t *sigar, sigar_mem_t *mem) {
    DLLMOD_INIT(kernel, TRUE);

    if (sigar_GlobalMemoryStatusEx) {
        MEMORYSTATUSEX memstat;
        memstat.dwLength = sizeof(memstat);

        if (!sigar_GlobalMemoryStatusEx(&memstat)) {
            return GetLastError();
        }

        mem->total = memstat.ullTotalPhys;
        mem->free  = memstat.ullAvailPhys;
    }
    else {
        MEMORYSTATUS memstat;
        GlobalMemoryStatus(&memstat);
        mem->total = memstat.dwTotalPhys;
        mem->free  = memstat.dwAvailPhys;
    }

    mem->used = mem->total - mem->free;

    // 从性能计数器获取实际可用内存（考虑文件缓存）
    get_mem_counters(sigar, NULL, mem);

    sigar_mem_calc_ram(sigar, mem);
    return SIGAR_OK;
}
```

### 2. CPU 信息获取

#### 方式一：性能计数器

```c
static int sigar_cpu_perflib_get(sigar_t *sigar, sigar_cpu_t *cpu) {
    PERF_INSTANCE_DEFINITION *inst;
    PERF_COUNTER_BLOCK *counter_block;
    DWORD perf_offsets[PERF_IX_CPU_MAX];

    SIGAR_ZERO(cpu);
    memset(&perf_offsets, 0, sizeof(perf_offsets));

    inst = get_cpu_instance(sigar, (DWORD*)&perf_offsets, 0, &err);

    if (!inst) return err;

    // 第一个实例是总CPU时间
    counter_block = PdhGetCounterBlock(inst);

    cpu->sys  = PERF_VAL_CPU(PERF_IX_CPU_SYS);
    cpu->user = PERF_VAL_CPU(PERF_IX_CPU_USER);
    status = get_idle_cpu(sigar, cpu, -1, counter_block, perf_offsets);
    cpu->irq = PERF_VAL_CPU(PERF_IX_CPU_IRQ);
    cpu->nice = 0;  // Windows没有nice值
    cpu->wait = 0;  // Windows没有wait值
    cpu->total = cpu->sys + cpu->user + cpu->idle + cpu->wait + cpu->irq;

    return SIGAR_OK;
}
```

#### 方式二：NT系统调用

```c
static int sigar_cpu_ntsys_get(sigar_t *sigar, sigar_cpu_t *cpu) {
    SYSTEM_PROCESSOR_PERFORMANCE_INFORMATION info[SPPI_MAX];

    sigar_NtQuerySystemInformation(SystemProcessorPerformanceInformation,
                                   &info, sizeof(info), &retval);

    if (!retval) return GetLastError();

    num = retval/sizeof(info[0]);
    SIGAR_ZERO(cpu);

    for (i=0; i<num; i++) {
        cpu->idle += NS100_2MSEC(info[i].IdleTime.QuadPart);
        cpu->user += NS100_2MSEC(info[i].UserTime.QuadPart);
        cpu->sys  += NS100_2MSEC(info[i].KernelTime.QuadPart -
                                 info[i].IdleTime.QuadPart);
        cpu->irq  += NS100_2MSEC(info[i].InterruptTime.QuadPart);
    }
    cpu->total = cpu->idle + cpu->user + cpu->sys;

    return SIGAR_OK;
}
```

### 3. 进程列表获取

```c
int sigar_os_proc_list_get(sigar_t *sigar, sigar_proc_list_t *proclist) {
    DLLMOD_INIT(psapi, FALSE);

    if (sigar_EnumProcesses) {
        DWORD retval, *pids;
        DWORD size = 0, i;

        // 动态调整缓冲区大小
        do {
            if (size == 0) {
                size = perfbuf_init(sigar);
            }
            else {
                size = perfbuf_grow(sigar);
            }

            if (!sigar_EnumProcesses((DWORD *)sigar->perfbuf,
                                     sigar->perfbuf_size,
                                     &retval))
            {
                return GetLastError();
            }
        } while (retval == sigar->perfbuf_size);

        pids = (DWORD *)sigar->perfbuf;
        size = retval / sizeof(DWORD);

        for (i=0; i<size; i++) {
            DWORD pid = pids[i];
            if (pid == 0) {
                continue;  // 跳过系统空闲进程
            }
            SIGAR_PROC_LIST_GROW(proclist);
            proclist->data[proclist->number++] = pid;
        }

        return SIGAR_OK;
    }
    else {
        // 回退到性能计数器方式
        return sigar_proc_list_get_perf(sigar, proclist);
    }
}
```

### 4. 进程信息缓存机制

```c
static int get_proc_info(sigar_t *sigar, sigar_pid_t pid) {
    sigar_win32_pinfo_t *pinfo = &sigar->pinfo;
    time_t timenow = time(NULL);

    // 缓存检查：如果在缓存过期时间内且PID相同，直接返回缓存数据
    if (pinfo->pid == pid) {
        if ((timenow - pinfo->mtime) < SIGAR_LAST_PROC_EXPIRE) {
            return SIGAR_OK;
        }
    }

    memset(&perf_offsets, 0, sizeof(perf_offsets));

    object = get_process_object(sigar, &err);
    if (!object) return err;

    // ... 获取进程计数器数据并填充 pinfo

    pinfo->pid = pid;
    pinfo->mtime = timenow;

    return SIGAR_OK;
}
```

---

## 核心功能模块

### 1. 内存监控模块

#### 物理内存
- **API**: `GlobalMemoryStatusEx()` 或 `GlobalMemoryStatus()`
- **信息**:
  - 总物理内存 (`ullTotalPhys`)
  - 可用物理内存 (`ullAvailPhys`)
  - 文件缓存 (`System Cache Resident Bytes`)

#### 虚拟内存/交换分区
- **API**: `GlobalMemoryStatusEx()`
- **信息**:
  - 总页文件大小 (`ullTotalPageFile`)
  - 可用页文件大小 (`ullAvailPageFile`)
  - 页面换入/换出速率 (`Pages Input/sec`, `Pages Output/sec`)

### 2. CPU 监控模块

#### CPU 时间统计
- **User Time**: 用户态时间
- **System Time**: 内核态时间（不含空闲）
- **Idle Time**: 空闲时间
- **Interrupt Time**: 中断时间
- **DPC Time**: 延迟过程调用时间

#### 多CPU支持
- 总CPU信息（所有CPU的汇总）
- 单CPU信息（每个逻辑CPU的独立统计）
- 逻辑CPU合并（支持超线程的物理核心合并）

### 3. 进程监控模块

#### 进程列表
- **API**: `EnumProcesses()` (psapi.dll)
- **备选**: 性能计数器 (Process 对象)

#### 进程内存
- **Virtual Bytes**: 虚拟内存大小
- **Working Set**: 工作集（物理内存）
- **Page Faults**: 页面错误数

#### 进程时间
- **User Time**: 用户态时间
- **System Time**: 内核态时间
- **Start Time**: 进程启动时间

#### 进程标识
- **Process ID**: 进程ID
- **Parent Process ID**: 父进程ID
- **Thread Count**: 线程数
- **Handle Count**: 句柄数

#### 进程环境
- **命令行参数**: 通过 PEB 或 WMI
- **可执行文件路径**: 通过 PEB 或 WMI
- **工作目录**: 通过 PEB
- **环境变量**: 通过 PEB

### 4. 网络监控模块

#### 网络接口列表
- **API**: `GetIfTable()` (iphlpapi.dll)
- **信息**:
  - 接口名称和描述
  - MAC 地址
  - MTU
  - 操作状态

#### IP 配置
- **API**: `GetAdaptersInfo()` 或 `GetAdaptersAddresses()`
- **信息**:
  - IPv4 地址
  - IPv6 地址
  - 子网掩码
  - 广播地址

#### 网络统计
- **发送/接收字节数**
- **发送/接收包数**
- **发送/接收错误数**
- **丢弃的包数**

#### 网络路由
- **API**: `GetIpForwardTable()`
- **信息**:
  - 目标地址
  - 网络掩码
  - 网关
  - 接口索引
  - 跃点数

### 5. 文件系统监控模块

#### 磁盘使用
- **API**: `GetDiskFreeSpaceEx()`
- **信息**:
  - 总容量
  - 可用空间
  - 空闲空间
  - 已用空间
  - 使用率

#### 磁盘I/O统计
- **API**: 性能计数器 (LogicalDisk 对象)
- **信息**:
  - 磁盘使用时间
  - 读取时间/写入时间
  - 读取/写入字节数
  - 读取/写入次数
  - 当前队列长度

#### 磁盘列表
- **API**: `GetLogicalDriveStrings()`
- **信息**:
  - 驱动器类型
  - 文件系统类型 (NTFS, FAT32等)
  - 挂载选项

---

## 性能优化机制

### 1. 缓存机制

#### 进程信息缓存
```c
typedef struct {
    sigar_pid_t pid;
    int ppid;
    int priority;
    time_t mtime;        // 最后更新时间
    sigar_uint64_t size;
    sigar_uint64_t resident;
    char name[SIGAR_PROC_NAME_LEN];
    char state;
    sigar_uint64_t handles;
    sigar_uint64_t threads;
    sigar_uint64_t page_faults;
    sigar_uint64_t bytes_read;
    sigar_uint64_t bytes_written;
} sigar_win32_pinfo_t;
```

缓存有效期检查：`SIGAR_LAST_PROC_EXPIRE`（默认60秒）

#### 网络接口缓存
- `netif_adapters`: 网卡适配器信息缓存
- `netif_mib_rows`: MIB行缓存
- `netif_addr_rows`: IP地址行缓存
- `netif_names`: 网卡名称映射缓存

### 2. 动态缓冲区管理

```c
// 缓冲区初始化
static DWORD perfbuf_init(sigar_t *sigar) {
    if (!sigar->perfbuf) {
        sigar->perfbuf = calloc(1, PERFBUF_SIZE);  // 初始8KB
        sigar->perfbuf_size = PERFBUF_SIZE;
    }
    return sigar->perfbuf_size;
}

// 缓冲区增长
static DWORD perfbuf_grow(sigar_t *sigar) {
    sigar->perfbuf_size += PERFBUF_SIZE;  // 每次增加8KB
    sigar->perfbuf = realloc(sigar->perfbuf, sigar->perfbuf_size);
    return sigar->perfbuf_size;
}
```

### 3. 延迟初始化

DLL模块按需初始化：
```c
#define DLLMOD_INIT(name, all) \
    sigar_dllmod_init(sigar, (sigar_dll_module_t *)&(sigar->name), all)
```

参数 `all` 控制是否所有函数都必须成功加载。

### 4. 多级回退机制

对于关键功能，实现多级回退：

1. **CPU信息**:
   - 优先: `NtQuerySystemInformation`
   - 备选: 性能计数器

2. **进程列表**:
   - 优先: `EnumProcesses()` (PSAPI)
   - 备选: 性能计数器

3. **进程命令行**:
   - 优先: PEB (快速)
   - 备选: WMI (慢速但兼容性好)

---

## 特殊技术

### 1. 时间转换

Windows FILETIME 转换为 Unix 时间戳：

```c
#define EPOCH_DELTA INT64_C(11644473600000000)

sigar_uint64_t sigar_FileTimeToTime(FILETIME *ft) {
    sigar_uint64_t time;
    time = ft->dwHighDateTime;
    time = time << 32;
    time |= ft->dwLowDateTime;
    time /= 10;      // 100纳秒转为微秒
    time -= EPOCH_DELTA;  // 减去1601年到1970年的时间差
    return time;
}
```

### 2. 权限提升

为了获取更多进程信息，需要提升调试权限：

```c
static int sigar_enable_privilege(char *name) {
    HANDLE handle;
    TOKEN_PRIVILEGES tok;

    if (!OpenProcessToken(GetCurrentProcess(),
                          TOKEN_ADJUST_PRIVILEGES|TOKEN_QUERY,
                          &handle)) {
        return GetLastError();
    }

    if (LookupPrivilegeValue(NULL, name, &tok.Privileges[0].Luid)) {
        tok.PrivilegeCount = 1;
        tok.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;

        if (AdjustTokenPrivileges(handle, FALSE, &tok, 0, NULL, 0)) {
            status = SIGAR_OK;
        }
    }

    CloseHandle(handle);
    return status;
}

// 在初始化时调用
sigar_enable_privilege(SE_DEBUG_NAME);
```

### 3. 宽字符/ANSI字符转换

支持多语言环境：

```c
#define SIGAR_A2W(lpa, lpw, bytes) \
    (lpw[0] = 0, MultiByteToWideChar(CP_ACP, 0, \
                                     lpa, -1, lpw, (bytes/sizeof(WCHAR))))

#define SIGAR_W2A(lpw, lpa, chars) \
    (lpa[0] = '\0', WideCharToMultiByte(CP_ACP, 0, \
                                        lpw, -1, (LPSTR)lpa, chars, \
                                        NULL, NULL))
```

### 4. 网卡名称处理

针对不同网络接口类型的命名：

```c
if (strEQ((const char *)ifr->bDescr, MS_LOOPBACK_ADAPTER)) {
    // 微软回环适配器特殊处理
    sprintf(name, NETIF_LA "%d", la++);
}
else if (ifr->dwType == MIB_IF_TYPE_LOOPBACK) {
    // 回环接口
    if (!sigar->netif_name_short) {
        status = sigar_net_interface_name_get(sigar, ifr, address_list, name);
    }
    if (status != SIGAR_OK) {
        sprintf(name, "lo%d", lo++);
    }
}
else if ((ifr->dwType == MIB_IF_TYPE_ETHERNET) ||
         (ifr->dwType == IF_TYPE_IEEE80211)) {
    // 以太网或无线网卡
    if (!sigar->netif_name_short &&
        (strstr((const char *)ifr->bDescr, "Scheduler") == NULL) &&
        (strstr((const char *)ifr->bDescr, "Filter") == NULL))
    {
        status = sigar_net_interface_name_get(sigar, ifr, address_list, name);
    }

    if (status != SIGAR_OK) {
        if (sigar->netif_name_short) {
            sprintf(name, "eth%d", eth++);
        }
        else {
            snprintf(name, ifr->dwDescrLen, "%s", ifr->bDescr);
        }
    }
}
```

### 5. CPU 核心合并

支持超线程处理器上的逻辑CPU合并：

```c
int core_rollup = sigar_cpu_core_rollup(sigar);

for (i=0; i<num; i++) {
    sigar_cpu_t *cpu;

    if (core_rollup && (i % sigar->lcpu)) {
        // 合并同一物理核心的逻辑处理器
        cpu = &cpulist->data[cpulist->number-1];
    }
    else {
        SIGAR_CPU_LIST_GROW(cpulist);
        cpu = &cpulist->data[cpulist->number++];
        SIGAR_ZERO(cpu);
    }

    // 累加CPU时间
    cpu->sys  += PERF_VAL_CPU(PERF_IX_CPU_SYS);
    cpu->user += PERF_VAL_CPU(PERF_IX_CPU_USER);
    cpu->irq  += PERF_VAL_CPU(PERF_IX_CPU_IRQ);
    get_idle_cpu(sigar, cpu, i, counter_block, perf_offsets);

    cpu->total = cpu->sys + cpu->user + cpu->idle + cpu->irq;
}
```

---

## 兼容性处理

### 1. 编译器兼容

```c
#if !defined(MSVC) && defined(_MSC_VER)
#define MSVC
#endif

#ifdef MSVC
#define WIN32_LEAN_AND_MEAN
#define snprintf _snprintf
#if _MSC_VER <= 1200
#define SIGAR_USING_MSC6  // Visual Studio 6
#define HAVE_MIB_IPADDRROW_WTYPE 0
#else
#define HAVE_MIB_IPADDRROW_WTYPE 1
#endif
#endif
```

### 2. Windows 版本检测

```c
OSVERSIONINFO version;
version.dwOSVersionInfoSize = sizeof(version);
GetVersionEx(&version);

// dwMajorVersion == 4: Windows NT 4.0
// dwMajorVersion == 5: Windows 2000/XP/2003 Server
// dwMajorVersion == 6: Windows Vista/7/8/10/11
sigar->winnt = (version.dwMajorVersion == 4);
```

### 3. 结构体兼容性

对于旧版本的 Visual Studio，手动定义缺失的结构体：

```c
#ifdef SIGAR_USING_MSC6

// MEMORYSTATUSEX - 在 VC6 中不存在
typedef struct {
    DWORD dwLength;
    DWORD dwMemoryLoad;
    DWORDLONG ullTotalPhys;
    DWORDLONG ullAvailPhys;
    DWORDLONG ullTotalPageFile;
    DWORDLONG ullAvailPageFile;
    DWORDLONG ullTotalVirtual;
    DWORDLONG ullAvailVirtual;
    DWORDLONG ullAvailExtendedVirtual;
} MEMORYSTATUSEX;

// SERVICE_STATUS_PROCESS - 在 VC6 中不存在
typedef struct _SERVICE_STATUS_PROCESS {
    DWORD dwServiceType;
    DWORD dwCurrentState;
    DWORD dwControlsAccepted;
    DWORD dwWin32ExitCode;
    DWORD dwServiceSpecificExitCode;
    DWORD dwCheckPoint;
    DWORD dwWaitHint;
    DWORD dwProcessId;
    DWORD dwServiceFlags;
} SERVICE_STATUS_PROCESS;

#endif /* _MSC_VER */
```

---

## 错误处理

### 1. Windows 错误码转换

```c
#define SIGAR_ENOENT ERROR_FILE_NOT_FOUND
#define SIGAR_EACCES ERROR_ACCESS_DENIED
#define SIGAR_ENXIO  ERROR_BAD_DRIVER_LEVEL

// 自定义错误码
#define SIGAR_NO_SUCH_PROCESS (SIGAR_OS_START_ERROR+1)
```

### 2. WMI 错误映射

```cpp
int wmi_map_sigar_error(HRESULT hres) {
    switch (hres) {
        case S_OK:
            return ERROR_SUCCESS;
        case WBEM_E_NOT_FOUND:
            return ERROR_NOT_FOUND;
        case WBEM_E_ACCESS_DENIED:
            return ERROR_ACCESS_DENIED;
        case WBEM_E_NOT_SUPPORTED:
            return SIGAR_ENOTIMPL;
        default:
            return ERROR_INVALID_FUNCTION;
    }
}
```

---

## 总结

libsigar 的 Windows 分支实现展示了以下特点：

1. **多层次API访问**: 从高层 WMI 到底层 NT API 的完整覆盖
2. **动态兼容性**: 通过运行时检测和动态加载支持不同 Windows 版本
3. **性能优化**: 缓存机制、延迟初始化、缓冲区重用等优化策略
4. **回退机制**: 关键功能都有备选实现方案
5. **跨平台抽象**: 统一的 API 接口，隐藏平台特定细节

这种设计使 libsigar 能够在 Windows 平台上稳定、高效地收集系统信息，同时保持代码的可维护性和扩展性。
