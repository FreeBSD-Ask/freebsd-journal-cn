# USE 方法

- 原文：[The USE Method](https://freebsdfoundation.org/our-work/journal/browser-based-edition/virtualization/the-use-method/)
- 作者：**Brendan Gregg**

性能调查中最难的部分可能是知道从哪里入手：先运行哪些分析工具、读哪些指标、和如何解读输出。FreeBSD 上的选择众多，标准工具包括 **top(1)**、**vmstat(8)**、**iostat(8)**、**netstat(1)**，更高级的选项如 **pmcstat(8)** 和 DTrace。这些工具共同提供数百个指标，并可定制以提供数千个。然而，我们大多数人并非全职性能工程师，可能只有时间检查熟悉的工具和指标，从而忽视许多潜在的问题区域。

利用率、饱和度和错误（USE）方法解决了这一问题，面向那些——尤其是系统管理员——偶尔进行性能分析的人。这是在调查早期快速检查系统性能、识别常见瓶颈和错误的流程。USE 方法不从工具及其统计信息入手，而是从我们希望得到回答的问题开始。这样，我们就不会因为对工具不熟悉或工具本身缺失而忽视问题。

## 它是什么？

USE 方法可用一句话概括：**对每个资源，检查利用率、饱和度和错误**。资源是指数据路径上的任何硬件，包括 CPU、内存、存储设备、网络接口、控制器和总线。软件资源（如互斥锁、线程池和资源控制限额）也可研究。利用率通常指资源忙于服务的工作时间比例，存储设备除外——其利用率也可以表示已用容量。饱和度是资源过载工作的程度，例如积压队列的长度。

通常还需要考察单个资源的统计信息。将 32 个 CPU 的 CPU 利用率平均可能会掩盖以 100% 运行的单个 CPU，磁盘利用率同理。识别这些是 USE 方法的目标，因为它们可能成为系统性瓶颈。

我最近在一次性能评估中使用了 USE 方法，我们的系统接受了基准测试，并与一个运行 Linux 的类似系统比较。潜在客户对 ZFS 相比之下的表现不满意，尽管我们的系统由 SSD 支撑。我之前研究过 ZFS 和 SSD 的病理，知道这可能需要数天才能理清，需要研究 ZFS 内部和 SSD 固件的复杂行为。我没有朝这些方向深入，而是在潜在客户的基准测试运行时，先用 USE 方法快速检查系统健康状况。这立即发现 CPU 利用率达 100%，而磁盘（和 ZFS）完全空闲。这让评估得以重置，将责任归到应有的地方（不是 ZFS），节省了我和他们的时间。

应用 USE 方法需要开发一份资源和 USE 统计信息组合的清单，以及你当前环境中用于观察它们的工具。这里包含一些示例，在 FreeBSD 10 alpha 2 上开发，涵盖硬件资源和部分软件资源。其中包括一些新的 DTrace 单行命令用于显示各种指标。你可以将这些复制到自己的公司文档（wiki）中，并用你环境中使用的其他工具增强。这些清单包含目标、工具和待研究指标的简要摘要。某些情况下，指标可直接从命令行工具读取；许多其他指标需要数学计算、推断或大量挖掘。随着工具的开发使 USE 方法指标更易获取，未来这有望变得更简单。

## 物理资源

| 组件 | 类型 | 指标 |
| :--- | :--- | :--- |
| CPU | 利用率 | 系统级：`vmstat 1`，“us” + “sy”；按 CPU：`vmstat -P`；按进程：`top`，“WCPU” |
| CPU | 饱和度 | 系统级：`uptime`，“load averages” > CPU 数；`vmstat 1`，“procs:r” > CPU 数；按 CPU：DTrace 采样 CPU 运行队列长度 [1]；按进程：DTrace 调度器事件 [2] |
| CPU | 错误 | `dmesg`；**/var/log/messages**；`pmcstat` 检查 PMC 和任何支持的错误计数器（如热降频） |
| 内存容量 | 利用率 | 系统级：`vmstat 1`，“fre”为主存空闲；`top`，“Mem:”；按进程：`top -o res`，“RES”为主存驻留大小，“SIZE”为虚拟内存大小；`ps -auxw` 也可；按内核进程：`top -S`，“WCPU” |
| 内存容量 | 饱和度 | 系统级：`vmstat 1`，“sr”为扫描率，“w”为交换线程（之前已饱和，现在可能不再）；`swapinfo`，“Capacity”同样为交换/分页证据；按进程：DTrace [3] |
| 内存容量 | 错误 | 物理：`dmesg`？；**/var/log/messages**？；虚拟：DTrace 失败的 `malloc()` |
| 网络接口 | 利用率 | 系统级：`netstat -i 1`，假设一个非常忙的接口并使用输入/输出“bytes” / 已知最大值（注意：包括 localhost 流量）；按接口：`netstat -I <接口> 1`，输入/输出“bytes” / 已知最大值 |
| 网络接口 | 饱和度 | 系统级：`netstat -s`，查找饱和度相关指标，例如 `netstat -s \| egrep 'retrans\|drop\|out-of-order\|memory problems\|overflow'`；按接口：DTrace |
| 网络接口 | 错误 | 系统级：`netstat -s \| egrep 'bad\|checksum'`，查找各种指标；按接口：`netstat -i`，“Ierrs”、“Oerrs”（如晚冲突）、“Colls” [5] |
| 存储设备 I/O | 利用率 | 系统级：`iostat -xz 1`，“%b”；按进程：DTrace io provider，例如 iosnoop 或 iotop（DTT，需移植） |
| 存储设备 I/O | 饱和度 | 系统级：`iostat -xz 1`，“qlen”；DTrace 查找队列时长或长度 [4] |
| 存储设备 I/O | 错误 | DTrace `io:::done` 探测点当 `/args[0]->b_error != 0/` |
| 存储容量 | 利用率 | 文件系统：`df -h`，“Capacity”；交换：`swapinfo`，“Capacity”；`pstat -T`，也显示交换空间 |
| 存储容量 | 饱和度 | 不确定此项是否有意义——一旦满了，ENOSPC |
| 存储容量 | 错误 | DTrace；**/var/log/messages** 文件系统满消息 |
| 存储控制器 | 利用率 | `iostat -xz 1`，将同一控制器上各设备的 IOPS 和吞吐量指标求和，与已知限制比较 [5] |
| 存储控制器 | 饱和度 | 检查利用率并使用 DTrace 查找内核排队 |
| 存储控制器 | 错误 | DTrace 驱动程序 |
| 网络控制器 | 利用率 | 系统级：`netstat -i 1`，假设一个繁忙控制器并检查输入/输出“bytes” / 已知最大值（注意：包括 localhost 流量） |
| 网络控制器 | 饱和度 | 见网络接口饱和度 |
| 网络控制器 | 错误 | 见网络接口错误 |
| CPU 互联 | 利用率 | `pmcstat`（PMC）检查 CPU 互联端口，吞吐量 / 最大值 |
| CPU 互联 | 饱和度 | `pmcstat` 和相关 PMC 检查 CPU 互联停顿周期 |
| CPU 互联 | 错误 | `pmcstat` 和相关 PMC 检查任何可用项 |
| 内存互联 | 利用率 | `pmcstat` 和相关 PMC 检查内存总线吞吐量 / 最大值，或测量 CPI 并推断 |
| 内存互联 | 饱和度 | `pmcstat` 和相关 PMC 检查内存停顿周期 |
| 内存互联 | 错误 | `pmcstat` 和相关 PMC 检查任何可用项 |
| I/O 互联 | 利用率 | `pmcstat` 和相关 PMC 检查吞吐量 / 最大值（如可用）；通过已知的 `iostat`/`netstat`/... 推断 |
| I/O 互联 | 饱和度 | `pmcstat` 和相关 PMC 检查 I/O 总线停顿周期 |
| I/O 互联 | 错误 | `pmcstat` 和相关 PMC 检查任何可用项 |

[1] 例如，使用每 CPU 运行队列长度作为饱和度指标：`dtrace -n 'profile-99 { @[cpu] = lquantize(` tdq_cpu [cpu].tdq_load, 0, 128, 1); } tick-1s { printa(@); trunc(@); }'`，其中 > 1 为饱和。如果使用较旧的 BSD 调度器，使用` profile runq_length [] `。还有` sched:::load-change` 和其他 sched 探测点。

[2] 对于此指标，我们可以使用 TDS_RUNQ 中花费的时间作为按线程的饱和度（延迟）指标。这是一个（不稳定的）基于 fbt 的单行命令：`dtrace -n 'fbt::tdq_runq_add:entry { ts[arg1] = timestamp; } fbt::choosethread:return /ts[arg1]/ { @[stringof(args[1]->td_name), "runq (ns)"] = quantize(timestamp - ts[arg1]); ts[arg1] = 0; }'`。如果改用 DTrace sched 探测点会更稳定。最好在 `struct rusage` 或 `rusage_ext` 中有高分辨率的线程状态时间，例如按 `td_state` 累计时间等，这会让读取该指标更容易且开销更低。

[3] 例如，对于交换：`dtrace -n 'fbt::cpu_thread_swapin:entry, fbt::cpu_thread_swapout:entry { @[probefunc, stringof(args[0]->td_name)] = count(); }'`（注意：我会追踪 `vm_thread_swapin()` 和 `vm_thread_swapout()`，但它们的探测点不存在）。在加入 vminfo provider 之前，追踪分页更难；你可以尝试从 `swap_pager_putpages()` 和 `swap_pager_getpages()` 追踪，但我没找到一种简单的方法回溯到 thread struct；另一种方法可能通过 `vm_fault_hold()`。祝你好运。参见线程状态 [2]，这可能让此问题简单很多。

[4] 例如，以 99 Hertz 采样 GEOM 队列长度：`dtrace -n 'profile-99 { @["geom qlen"] = lquantize(` g_bio_run_down.bio_queue_length, 0, 256, 1); }'`

[5] 此方法不同于存储设备（磁盘）利用率。对于控制器，busy 百分比意义不大，因此我们基于吞吐量（字节/秒）和 IOPS 计算利用率。控制器的这些限制通常取决于其总线和处理能力。如不知道限制，可通过实验确定。

### 缩写说明

- **PMC**：性能监控计数器（Performance Monitoring Counters），又称 CPU 性能计数器（CPC）、性能插桩计数器（PICs）等。这些是处理器硬件计数器，通过每个 CPU 上的可编程寄存器读取。这些计数器的可用性取决于处理器类型。参见 **pmc(3)** 和 **pmcstat(8)**。
- **pmcstat(8)**：FreeBSD 上用于插桩 PMC 的工具。使用前可能需要先运行 `kldload hwpmc`。要弄清需使用哪些 PMC 以及如何使用，通常需要花费大量时间（数天）查阅处理器供应商手册；例如《Intel 64 and IA-32 Architectures Software Developer’s Manual, Volume 3B, Appendix A-E: Performance-Monitoring Events》。
- **CPI**：每指令周期数（其他人使用 IPC = 每周期指令数）。
- **DTT**：DTraceToolkit（<http://www.brendangregg.com/dtrace.html#DTraceToolkit>）脚本。这些脚本位于 FreeBSD 源码（<http://svnweb.freebsd.org/base/head/cddl/contrib/dtrace-toolkit/>）的 `cddl/contrib/dtracetoolkit` 下，`dtruss` 位于 **/usr/sbin**。随着 DTrace 加入新特性，参见 freebsd-dtrace 邮件列表（<https://lists.freebsd.org/mailman/listinfo/freebsd-dtrace>），可移植更多脚本。
- **I/O 互联**：包括 CPU 到 I/O 控制器的总线、I/O 控制器以及设备总线（如 PCIe）。

## 软件资源

| 组件 | 类型 | 指标 |
| :--- | :--- | :--- |
| 内核互斥锁 | 利用率 | `lockstat -H`（持有时间）；DTrace lockstat provider |
| 内核互斥锁 | 饱和度 | `lockstat -C`（争用）；DTrace lockstat provider [6]；自旋显示利用率 |
| 内核互斥锁 | 错误 | `lockstat -E`（错误）；DTrace 和 fbt provider 用于返回探测点和错误 |
| 用户互斥锁 | 利用率 | DTrace pid provider 测量持有时间；例如 `pthread_mutex_*lock()` 返回时间 |
| 用户互斥锁 | 饱和度 | DTrace pid provider 测量争用；例如 `pthread_mutex_*lock()` 入口到 `pthread_mutex_unlock()` 入口 |
| 用户互斥锁 | 错误 | DTrace pid provider 检查 EINVAL、EDEADLK 等；参见 **pthread_mutex_lock(3C)** 等 |
| 进程容量 | 利用率 | 当前/最大：`ps -a \| wc -l` / `sysctl kern.maxproc`；`top`，“Processes:”也可 |
| 进程容量 | 饱和度 | 不确定此项是否有意义 |
| 进程容量 | 错误 | “can’t fork()”消息 |
| 文件描述符 | 利用率 | 系统级：`pstat -T`，“files”；`sysctl kern.openfiles` / `sysctl kern.maxfiles`；按进程：可使用 `fstat -p PID` 和 `ulimit -n` 推断 |
| 文件描述符 | 饱和度 | 我认为此项无意义，因为如果无法分配或扩展数组，它会出错；参见 `fdalloc()` |
| 文件描述符 | 错误 | `truss`、`dtruss` 或自定义 DTrace 查找返回 fd 的系统调用（如 `open()`、`accept()` 等）上的 `errno == EMFILE` |

`lockstat`：你可能需要先运行 `kldload ksyms`，否则 lockstat 才能工作（否则会报“lockstat: can’t load kernel symbols: No such file or directory”）。bustop 是一个潜在的基于 PMC 的工具，可用于显示总线和互联及其利用率、饱和度和错误。

[6] 例如，按调用函数名显示自适应锁阻塞时间总计（纳秒）：`dtrace -n 'lockstat:::adaptive-block { @[caller] = sum(arg1); } END { printa("%40a%@16d ns\n", @); }'`

## 实践

你可能注意到许多指标难以观察，特别是涉及 **pmcstat(8)** 的指标，可能需要数天才能搞清楚。实践中，你可能只有时间执行 USE 方法的子集，研究那些能快速确定的指标，并承认由于时间压力，部分指标仍属未知。

知道自己不知道什么仍然极有价值：可能某天有一个对你的公司至关重要的性能问题需要解决，而你已有一份待办清单，列出可以进一步执行以更彻底检查资源性能的工作。

## 持续分析

USE 方法旨在作为一种早期方法论，用于识别常见瓶颈和错误。然后可使用其他方法论研究它们，如 drill-down 分析和延迟分析。也存在 USE 方法完全无法发现的性能问题，应将其视为工具箱中的一种方法论而已。

## 其他工具

我没有列出 `procstat`、`sockstat`、`gstat` 等工具，因为 USE 方法旨在从问题出发，仅使用能回答问题的工具。这与列出所有可用工具再尝试为它们找到用武之地截然不同。其他工具对其他方法论有用，可在 USE 之后使用。

希望未来开发更多工具使 USE 方法更易用，扩大你有时间定期检查的指标子集。如果你是 FreeBSD 开发者，你可能是第一个开发某种工具让 USE 方法更易用的人。

## 参考资料

- 《Thinking Methodically about Performance》，Brendan Gregg，ACM Queue，vol. 10, no. 12, 2012
- The USE Method: FreeBSD Performance Checklist 摘要，我的博客
- FreeBSD 源码和 man 页
- FreeBSD Wiki PmcTools
- FreeBSD Handbook

---

**Brendan Gregg** 是 Netflix 的高级性能架构师。他是《Systems Performance》（Prentice Hall, 2014）的作者、《DTrace: Dynamic Tracing in Oracle Solaris, Mac OS X, and FreeBSD》（Prentice Hall, 2011）的主要作者，以及 USENIX 2013 LISA 系统管理杰出成就奖得主。此前他曾在 Sun Microsystems 担任性能主管和内核工程师，期间开发了 ZFS L2ARC 和多种 DTrace provider。
