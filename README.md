# 源码分析[mettle](https://github.com/rapid7/mettle)后门工具学习 所使用的依赖库

# 各种视频学习资料
<img width="1967" height="2339" alt="securitytech cc" src="https://github.com/user-attachments/assets/5705b8f7-e762-4cbb-a6fa-b07efa176fc8" />

![gzh](https://github.com/user-attachments/assets/769e49dd-3031-47e3-9105-b9acf1349325)

# libsigar
libsigar-1.6.7  源码分析

# libsigar 全平台源码分析文档

## 项目概述

libsigar (System Information Gatherer And Reporter) 是一个跨平台的系统监控库,由 Hyperic 公司开发,目前由 VMware 维护。该库提供了统一的 API 来获取各种操作系统的系统信息,包括 CPU、内存、进程、文件系统、网络等统计数据。

**版本**: 1.6.7  
**许可证**: Apache License 2.0  
**版权**: 2004-2010 Hyperic, Inc. / SpringSource, Inc. / VMware, Inc.

---

## 文档导航

### 全平台总结

### [ALL_PLATFORMS_SUMMARY.md](./ALL_PLATFORMS_SUMMARY.md)
**全平台源码分析总结**

这是 libsigar 所有平台的综合总结文档，提供了全局视角的概览。

**内容包含：**
- 支持的 9 个平台总览对比
- 核心架构和设计模式
- 各功能模块的跨平台对比
- 技术特点和性能优化策略
- 代码规模和文档统计
- 使用建议和局限性

**适用场景：**
- 快速了解 libsigar 的整体架构
- 跨平台功能对比
- 平台选型参考

---

## 平台特定文档

### 1. Linux

### [LINUX_SOURCE_CODE_ANALYSIS.md](./LINUX_SOURCE_CODE_ANALYSIS.md)
**Linux 平台源码分析** (1633 行)

**核心技术：**
- /proc 文件系统统一接口
- /sys 文件系统设备级统计
- 动态内核版本检测
- MTRR 内存检测

**主要内容：**
- 内存监控（物理内存、交换空间、缓存、缓冲区）
- CPU 监控（单核/多核、超线程合并、频率信息）
- 进程监控（列表、内存、I/O、凭据、状态、环境、模块）
- 网络监控（接口、连接、路由、ARP、TCP、NFS、IPv6）
- 文件系统监控（列表、使用情况、类型）
- 磁盘 I/O 监控（多源回退、派生指标）

**支持内核版本：** Linux 2.2 - 2.6.20+

---

### 2. Darwin (macOS)

### [DARWIN_SOURCE_CODE_ANALYSIS.md](./DARWIN_SOURCE_CODE_ANALYSIS.md)
**Darwin (macOS) 平台源码分析** (1364 行)

**核心技术：**
- sysctl 系统调用
- libproc 进程管理库
- IOKit 硬件访问框架
- CFRunLoop 事件处理

**主要内容：**
- 内存监控（包括系统缓存）
- CPU 监控（host_processor_info）
- 进程监控（完整进程信息）
- 网络监控（sysctl 网络统计）
- 文件系统监控（getfsstat）

**技术亮点：** Apple Silicon ARM 支持、高级进程管理接口、硬件信息访问、事件驱动架构

**支持版本：** macOS 10.x+

---

### 3. AIX

### [AIX_SOURCE_CODE_ANALYSIS.md](./AIX_SOURCE_CODE_ANALYSIS.md)
**AIX 平台源码分析** (586 行)

**核心技术：**
- libperfstat 性能统计库
- /dev/kmem 内核内存访问
- swapqry 未公开函数
- pthread_getrusage_np 线程资源

**主要内容：**
- 内存监控（动态内存统计）
- 交换空间监控
- CPU 监控（perfstat_cpu）
- 进程监控（procsinfo64 批量获取）
- 网络监控（perfstat_net）

**技术亮点：** 双模式支持（perfstat + kmem）、批量进程获取、POWER 架构优化

**支持版本：** AIX 5L+

---

### 4. FreeBSD

### [FREEBSD_SOURCE_CODE_ANALYSIS.md](./FREEBSD_SOURCE_CODE_ANALYSIS.md)
**FreeBSD 平台源码分析** (711 行)

**核心技术：**
- sysctl 系统调用
- kvm 库内核访问
- /proc 文件系统
- FreeBSD 4/5 版本兼容

**主要内容：**
- 内存监控（sysctl）
- 交换空间监控
- CPU 监控（sysctl）
- 进程监控（kinfo_proc）
- 网络监控（sysctl）
- 文件系统监控

**技术亮点：** 统一 sysctl 接口、版本兼容性处理、Jails 容器支持、网络统计完整

**支持版本：** FreeBSD 4.x - 11.x+

---

### 5. NetBSD

### [NETBSD_SOURCE_CODE_ANALYSIS.md](./NETBSD_SOURCE_CODE_ANALYSIS.md)
**NetBSD 平台源码分析** (382 行)

**核心技术：**
- sysctl 系统调用
- kvm 库内核访问
- kinfo_proc2 扩展结构

**主要内容：**
- 内存监控（sysctl）
- CPU 监控（sysctl）
- 进程监控（kinfo_proc2）
- 网络监控

**技术亮点：** 设计简洁性、高可移植性、扩展进程信息结构、进程状态映射

**支持版本：** NetBSD 3.x+

---

### 6. OpenBSD

### [OPENBSD_SOURCE_CODE_ANALYSIS.md](./OPENBSD_SOURCE_CODE_ANALYSIS.md)
**OpenBSD 平台源码分析** (504 行)

**核心技术：**
- sysctl 系统调用
- kvm 库内核访问
- /proc 可选支持
- 内核符号解析（kvm_nlist）

**主要内容：**
- 内存监控（sysctl）
- 交换空间监控
- CPU 监控（sysctl）
- 进程监控（kinfo_proc）

**技术亮点：** 安全导向设计、/proc 可选支持、严格权限控制、与 FreeBSD 共享代码

**支持版本：** OpenBSD 4.x+

---

### 7. Solaris

### [SOLARIS_SOURCE_CODE_ANALYSIS.md](./SOLARIS_SOURCE_CODE_ANALYSIS.md)
**Solaris 平台源码分析** (757 行)

**核心技术：**
- kstat 库统一接口
- MIB2 Stream 网络统计
- libproc 库动态加载
- Zone 虚拟化支持
- ZFS ARC 缓存统计

**主要内容：**
- 内存监控（含 Zone）
- 交换空间监控
- CPU 监控（kstat）
- 进程监控（libproc）
- 网络监控（MIB2）
- 文件系统监控

**技术亮点：** kstat 统一接口、Zone 虚拟化支持、ZFS 集成统计、SmartOS 兼容、动态库加载

**支持版本：** Solaris 8 - 10、illumos / OpenIndiana

---

### 8. HPUX

### [HPUX_SOURCE_CODE_ANALYSIS.md](./HPUX_SOURCE_CODE_ANALYSIS.md)
**HPUX 平台源码分析** (651 行)

**核心技术：**
- pstat 系列系统调用
- MIB (Management Information Base) 网络统计
- PA-RISC 和 Itanium 双架构支持
- 逻辑卷磁盘 I/O 统计

**主要内容：**
- 内存监控（pstat）
- 交换空间监控（pstat_getswap）
- CPU 监控（单核/多核）
- 进程监控（pstat_getproc）
- 线程 CPU（PA-RISC）
- 文件系统监控
- 网络监控（MIB）

**技术亮点：** pstat 统一接口、架构自适应（PA-RISC / Itanium）、双模式网络统计、批量进程获取

**支持版本：** HPUX 11.00 - 11.31

---

### 9. Windows

### [WINDOWS_SOURCE_CODE_ANALYSIS.md](./WINDOWS_SOURCE_CODE_ANALYSIS.md)
**Windows 平台源码分析** (1052 行)

**核心技术：**
- Win32 API
- 性能计数器（Performance Counters）
- PDH (Performance Data Helper)
- WMI (Windows Management Instrumentation)

**主要内容：**
- 内存监控（GlobalMemoryStatusEx）
- CPU 监控（GetSystemTimes）
- 进程监控（CreateToolhelp32Snapshot, PSAPI）
- 服务监控
- 网络监控（GetIfEntry, GetTcpTable）
- 文件系统监控

**技术亮点：** 完整的 Win32 API 覆盖、性能计数器系统、服务管理、WMI 补充接口

**支持版本：** Windows 2000+、32 位 / 64 位

---

## 其他相关文档

- [aix.md](./aix.md) - AIX 平台安装和使用说明
- [freebsd.md](./freebsd.md) - FreeBSD 平台安装和使用说明
- [NetBSD.md](./NetBSD.md) - NetBSD 平台安装和使用说明
- [Solaris.md](./Solaris.md) - Solaris 平台安装和使用说明
- [windows.md](./windows.md) - Windows 平台安装和使用说明

---

## 支持的平台总览

| # | 平台 | 主要接口 | 内核访问 | 网络接口 | 架构支持 |
|---|------|----------|----------|----------|----------|
| 1 | **Linux** | /proc 文件系统 | 无需 | /proc/net | x86, ARM, RISC-V |
| 2 | **Darwin** (macOS) | sysctl + libproc | 无需 | sysctl | x86, ARM (Apple Silicon) |
| 3 | **AIX** | libperfstat | /dev/kmem | MIB | POWER |
| 4 | **FreeBSD** | sysctl + kvm | /dev/kmem | sysctl | x86, ARM |
| 5 | **NetBSD** | sysctl + kvm | /dev/kmem | sysctl | x86, ARM, m68k |
| 6 | **OpenBSD** | sysctl + kvm | /dev/kmem | sysctl | x86, ARM, m68k |
| 7 | **Solaris** | kstat + libproc | kstat | MIB2 Stream | SPARC, x86 |
| 8 | **HPUX** | pstat | 无需 | MIB (/dev/ip) | PA-RISC, Itanium |
| 9 | **Windows** | Win32 API | 无需 | Win32 API | x86, x64, ARM |

---

## 文档使用指南

### 按需求选择文档

**快速概览**
- 👉 先阅读 [ALL_PLATFORMS_SUMMARY.md](./ALL_PLATFORMS_SUMMARY.md)

**学习特定平台**
- Linux: [LINUX_SOURCE_CODE_ANALYSIS.md](./LINUX_SOURCE_CODE_ANALYSIS.md)
- macOS: [DARWIN_SOURCE_CODE_ANALYSIS.md](./DARWIN_SOURCE_CODE_ANALYSIS.md)
- 其他平台: 选择对应的平台文档

**跨平台开发**
- 参考 [ALL_PLATFORMS_SUMMARY.md](./ALL_PLATFORMS_SUMMARY.md) 中的对比表
- 阅读各平台文档了解差异

**性能优化**
- 关注各文档中的"技术亮点"和"性能优化"章节

### 文档结构说明

每个平台文档通常包含：
1. **概述** - 平台特点和设计理念
2. **核心架构** - 数据结构和扩展字段
3. **核心技术** - 主要接口和实现方式
4. **核心功能模块** - 各功能模块的实现细节
5. **技术亮点** - 独特的设计和优化
6. **未实现功能** - 平台限制

### 代码引用格式

文档中使用以下格式引用代码：

```
文件路径:行号
```

例如：`linux_sigar.c:1234:src/os/linux`

### 技术术语说明

- **pstat**: HPUX 的性能统计系统调用系列
- **kstat**: Solaris 的内核统计接口
- **sysctl**: BSD 系统的系统控制接口
- **kvm**: BSD 系统的内核虚拟内存访问库
- **libperfstat**: AIX 的性能统计库
- **MIB**: Management Information Base（管理信息库）
- **procfs**: 进程文件系统（Linux 的 /proc）

---

## 核心功能对比

### 内存监控

| 平台 | 主要接口 | 支持指标 |
|------|----------|----------|
| Linux | /proc/meminfo | total, free, used, actual_free, actual_used, cached, buffers |
| Darwin | sysctl, host_statistics | total, free, used, actual_free, actual_used, pageins, pageouts |
| AIX | perfstat_memory_total | total, free, used, actual_free, actual_used |
| FreeBSD | sysctl | total, free, used, actual_free, actual_used |
| NetBSD | sysctl | total, free, used, actual_free, actual_used |
| OpenBSD | sysctl | total, free, used, actual_free, actual_used |
| Solaris | kstat, swapctl | total, free, used, actual_free, actual_used |
| HPUX | pstat_getdynamic | total, free, used, actual_free, actual_used |
| Windows | GlobalMemoryStatusEx | total, free, used, actual_free, actual_used |

### CPU 监控

| 平台 | 单核统计 | 多核统计 | CPU 信息 |
|--------|----------|----------|----------|
| Linux | ✅ /proc/stat | ✅ | ✅ |
| Darwin | ✅ host_processor_info | ✅ | ✅ |
| AIX | ✅ perfstat_cpu_total | ✅ | ✅ |
| FreeBSD | ✅ sysctl | ⚠️ 部分支持 | ✅ |
| NetBSD | ✅ sysctl | ⚠️ 部分支持 | ✅ |
| OpenBSD | ✅ sysctl | ⚠️ 部分支持 | ✅ |
| Solaris | ✅ kstat | ✅ | ✅ |
| HPUX | ✅ pstat_getdynamic | ✅ | ✅ |
| Windows | ✅ GetSystemTimes | ✅ | ✅ |

### 进程监控

| 平台 | 进程列表 | 进程内存 | 进程状态 | 进程时间 | 进程参数 | 进程环境 | 进程文件描述符 |
|--------|----------|----------|----------|----------|----------|----------|----------------|
| Linux | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Darwin | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| AIX | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| FreeBSD | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ⚠️ 部分支持 |
| NetBSD | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| OpenBSD | ✅ | ✅ | ✅ | ✅ | ⚠️ 部分支持 | ❌ | ❌ |
| Solaris | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| HPUX | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Windows | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |

---

## 技术亮点总结

### Linux
- **/proc 文件系统**统一接口
- **多源回退机制**保证兼容性
- **内核版本自适应**
- **高效的缓存机制**

### Darwin
- **libproc** 高级进程管理
- **IOKit** 硬件访问
- **CFRunLoop** 事件处理
- **Apple Silicon** ARM 支持

### AIX
- **libperfstat** 统一接口
- **批量进程获取**
- **性能计数器集成**
- **POWER 架构优化**

### FreeBSD
- **sysctl + kvm** 双接口
- **版本兼容性处理**
- **Jails 容器支持**
- **网络统计完整**

### NetBSD
- **设计简洁性**
- **kinfo_proc2** 扩展结构
- **可移植性优先**
- **进程状态映射**

### OpenBSD
- **安全导向设计**
- **/proc 可选支持**
- **严格权限控制**
- **代码正确性**

### Solaris
- **kstat** 统一接口
- **Zones** 虚拟化支持
- **ZFS** 集成统计
- **MIB2 Stream** 网络接口

### HPUX
- **pstat** 系列接口
- **PA-RISC / Itanium** 双架构
- **MIB** 网络统计
- **逻辑卷集成**

### Windows
- **Win32 API** 完整覆盖
- **性能计数器** 系统
- **WMI** 补充接口
- **服务管理**

---

## 核心价值

libsigar 是一个设计精良的跨平台系统监控库,通过统一的接口封装了 9 个主要操作系统的系统信息获取功能。

1. **统一接口**: 屏蔽平台差异
2. **高性能**: 多种优化策略
3. **可扩展**: 易于添加新平台
4. **稳定性**: 经过大量生产环境验证

### 适用场景
- 系统监控工具
- 性能分析工具
- 资源管理系统
- 运维自动化平台
- 云计算监控

### 学习价值
通过研究 libsigar 的源码,可以深入了解:
- 各操作系统的系统编程接口
- 跨平台开发的设计模式
- 性能优化的最佳实践
- 系统监控的底层原理

---

## 贡献指南

如果您发现文档中有错误或需要补充的内容，欢迎：
1. 提交 Issue 报告问题
2. 提交 Pull Request 改进文档
3. 分享您的使用经验

---

## 许可证

本文档遵循 libsigar 的 Apache License 2.0 许可证。

---

## 更新日志

| 日期 | 版本 | 更新内容 |
|------|------|----------|
| 2025-03-03 | 1.0.0 | 初始版本，完成 9 个平台的源码分析文档 |

---

**最后更新**: 2025-03-03  
**文档版本**: 1.0.0
