# DTrace：老式跟踪系统的新扩展

- 原文地址：[DTrace: New Additions to an Old Tracing System](https://freebsdfoundation.org/wp-content/uploads/2023/01/stolfa_dtrace.pdf)
- 作者：**DOMAGOJ STOLFA**

## DTrace 简介  

DTrace 是 FreeBSD 内置的软件跟踪框架，允许用户实时检查和修改正在运行的系统。它高度可扩展，最初为 Solaris 设计，但后来被多次移植到其他环境，如 FreeBSD、macOS、Windows 和 Linux。本文将重点介绍在 FreeBSD 上 DTrace 的用法，并提供示例，同时总结 DTrace 在 FreeBSD 领域的最新发展。  

## DTrace 概述

操作系统是极其复杂的软件，包含众多组件。单一的跟踪系统若试图支持几乎整个操作系统的跟踪，可能会因其复杂性而变得难以管理。为了解决这一问题并支持未来的扩展，DTrace 引入了 **provider（提供者）** 的概念。  

在 FreeBSD 中，**provider** 默认作为内核模块（kernel module）运行，并负责实现特定操作系统组件的跟踪功能。它们提供 **DTrace probes（探测点）**，即操作系统代码中的命名位置，可通过 **D** 语言编写的可脚本化例程进行动态跟踪。  

FreeBSD 自带的一些示例 **provider** 包括：  

- **函数边界跟踪提供者（fbt.ko）**：负责对 **内核函数** 的入口和退出点进行跟踪。  
- **性能分析提供者（profile.ko）**：提供基于脚本指定的固定时间间隔触发的探测点。  
- **PID 提供者（fasttrap.ko）**：类似于 fbt，但适用于 **用户进程及其链接的库**。  
- **其他多种 provider**。  

虽然深入了解 DTrace 并非理解本文的必要前提，但希望深入学习 DTrace 的读者可以参考以下资料：  

1. **用户指南**  
2. **DTrace 规范**  
3. **FreeBSD 手册页面**³  
4. **DTrace 白皮书**⁴  
5. **DTrace 书籍**⁵  
6. **FreeBSD Wiki 页面（包括一些 DTrace 单行命令示例⁶）**。  

此外，以往的 FreeBSD Journal 也曾多次发表关于 DTrace 的相关文章⁷⁸⁹。  

## 简单示例

**DTrace 探测点** 由 **provider:module:function:name** **四元组** 指定。其中的每个字段可以使用通配符匹配多个值，或留空表示“匹配所有”。  

下面是一个简单的 DTrace **监视脚本（snooper script）**，用于检测 **用户正在运行的程序**。我们在执行时使用 `-x quiet` 选项，以避免 DTrace 输出额外的信息。

```sh
# dtrace -x quiet -n 'proc:::exec { printf(“user = %u, gid = %u: %s\n”, uid, gid,
stringof(args[0])); }'
user = 1001, gid = 1001: /usr/sbin/service
user = 1001, gid = 1001: /bin/kenv
user = 1001, gid = 1001: /sbin/sysctl
user = 1001, gid = 1001: /sbin/env
user = 1001, gid = 1001: /bin/env
user = 1001, gid = 1001: /usr/sbin/env
user = 1001, gid = 1001: /usr/bin/env
user = 1001, gid = 1001: /etc/rc.d/sendmail
user = 1001, gid = 1001: /bin/kenv
user = 1001, gid = 1001: /sbin/sysctl
user = 1001, gid = 1001: /bin/ls
```

正如我们所看到的，**D 语言** 在语法上与 **C 语言** 非常相似，但也有一些特定的特殊语法形式。与 C 语言不同，D 语言**不支持循环**，因此任何形式的循环都必须**手动展开**。  

在上面的示例中，我们可以通过 **`uid`** 和 **`gid`** 这两个内置变量来访问用户 ID 和组 ID。  

此外，DTrace 还支持以多种方式对跟踪结果进行**聚合**。例如，我们可以统计**每个程序执行的系统调用次数**。

```sh
# dtrace -n 'syscall:::entry { @syscall_agg[execname, pid] = count(); }'
dtrace: description 'syscall:::entry ' matched 1148 probes
 sh                                                    46569                7
 sh                                                    46570                7
 syslogd                                                 703               16
 sshd                                                    848               17
 devd                                                    501               20
 ntpd                                                    771               24
 sh                                                    46565               93
 dtrace                                                46568              138
 ps                                                    46570              254
 sshd                                                  46564            27517
 ls                                                    46569            35755
```

在变量前使用 **`@`** 作为前缀，会将其定义为**聚合变量**。 `@syscall_agg` 以两个键进行索引，不过也可以继续添加更多的键。 `@syscall_agg` 的聚合输出应解读为：

```c
execname                                              pid            count
```

我们的最后一个示例涉及**堆栈追踪**。DTrace 允许用户使用 **`stack()`** 和 **`ustack()`** 例程分别收集**内核**和**用户空间**的堆栈追踪。此外，DTrace 还可以通过**语言特定的堆栈展开器**进行扩展。例如，**`jstack()`** 操作可以为用户提供 Java 程序的可读**回溯信息**。在我们的示例中，我们将重点关注 **`stack()`**：

```c
# dtrace -x quiet -n 'io:::start { @[stack()] = count(); }'
             zfs.ko`zio_vdev_io_start+0x2f5           
             zfs.ko`zio_nowait+0x15f        
             zfs.ko`vdev_mirror_io_start+0xfd
             zfs.ko`zio_vdev_io_start+0x1eb   
             zfs.ko`zio_nowait+0x15f                  
             zfs.ko`arc_read+0x14aa                               
             zfs.ko`dbuf_read+0xc84    
             zfs.ko`dmu_tx_check_ioerr+0x84  
             zfs.ko`dmu_tx_count_write+0x191
             zfs.ko`dmu_tx_hold_write_by_dnode+0x64
             zfs.ko`zfs_write+0x500           
             zfs.ko`zfs_freebsd_write+0x39  
             kernel`VOP_WRITE_APV+0x194    
             kernel`vn_write+0x2ce            
             kernel`vn_io_fault_doio+0x43             
             kernel`vn_io_fault1+0x163            
             kernel`vn_io_fault+0x1cc          
             kernel`dofilewrite+0x81
             kernel`sys_writev+0x6e           
             kernel`amd64_syscall+0x12e        
               1                              
                                              
             zfs.ko`zio_vdev_io_start+0x2f5
             zfs.ko`zio_nowait+0x15f       
             zfs.ko`zil_lwb_write_done+0x360
             zfs.ko`zio_done+0x10d6            
             zfs.ko`zio_execute+0xdf                    
             kernel`taskqueue_run_locked+0xaa
             kernel`taskqueue_thread_loop+0xc2
             kernel`fork_exit+0x80           
             kernel`0xffffffff810a35ae      
               1
```

这个 D 脚本统计了**所有导致块设备 I/O 的内核堆栈追踪**。  

在此脚本中，我们省略了**聚合名称**，因为它只包含一个聚合，并且**键**是 `stack()`——这是一个 DTrace 内置操作，返回一个**程序计数器数组**，在打印结果时会解析为符号。  

此外，DTrace 还可以使用 **profile** 提供者收集 **CPU 上的堆栈追踪**，从而支持**生成火焰图（Flame Graphs）**。  



## 新进展

### dwatch

新工具 **dwatch** 由 **Devin Teske（dteske@freebsd.org）** 开发，并在 **FreeBSD 11.2** 中上游合并。 **dwatch** 使得 DTrace 在常见使用场景下比 `dtrace` 命令行工具更易用。回到我们之前的**进程监视**示例，用户只需运行：

```sh
# dwatch execve
```

即可获得**比之前的简单监视器更丰富的信息**，且输出经过**良好过滤**。

```sh
# dwatch execve
INFO Watching 'syscall:freebsd:execve:entry' ...
2022 Nov 24 18:46:53 1001.1001 sh[46565]: sudo ps auxw
2022 Nov 24 18:46:53 0.0 sudo[46920]: ps auxw
2022 Nov 24 18:46:55 1001.1001 sh[46565]: ls
2022 Nov 24 18:47:01 1001.1001 sh[46565]: ls -lapbtr
2022 Nov 24 18:47:09 1001.1001 sh[46924]: kenv -q rc.debug
2022 Nov 24 18:47:09 1001.1001 sh[46924]: /sbin/sysctl -n -q kern.boottrace.enabled
2022 Nov 24 18:47:09 1001.1001 sh[46565]: env -i -L -/daemon HOME=/ PATH=/sbin:/
bin:/usr/sbin:/usr/bin /etc/rc.d/sendmail onestop
2022 Nov 24 18:47:09 1001.1001 env[46565]: /bin/sh /etc/rc.d/sendmail onestop
2022 Nov 24 18:47:09 1001.1001 sh[46924]: kenv -q rc.debug
2022 Nov 24 18:47:09 1001.1001 sh[46924]: /sbin/sysctl -n -q kern.boottrace.enabled
```

此外，**dwatch** 支持基于 **jail**、用户组、进程等多种条件进行过滤，即便是经验丰富的 **DTrace** 用户也值得学习这款工具。演讲 *All along the dwatch tower* 介绍了 **dwatch** 并详细讲解了其功能。此外，**FreeBSD** 的 **dwatch(1)** 手册页也提供了许多优秀的示例，适合感兴趣的用户尝试。

## CTFv3  

**紧凑 C 类型格式（CTF）** 是一种用于在 **FreeBSD ELF** 二进制文件中编码 **C** 类型信息的格式。它使 **DTrace** 能够了解目标二进制文件（进程或内核）的 **C** 类型布局，以便用户编写的脚本可以引用和探索这些类型。  

过去，由于 **CTFv2** 的实现方式，**DTrace** 在单个二进制文件中最多仅支持 **2^15** 种 **C** 类型。这一限制导致了 **FreeBSD** 中许多与 **DTrace** 相关的错误报告。今年 **3 月**，**Mark Johnston（markj@freebsd.org）** 提交了相关更改，使 **DTrace** 切换为 **CTFv3**，这不仅提高了 **DTrace** 可处理的 **C** 类型数量，同时还扩展了 **CTF** 的其他多个限制。

## kinst —— 用于指令级跟踪的全新 DTrace 提供程序  

在 **2022 年 Google 夏季编程大赛（GSoC）** 中，**Christos Margiolis（christos@freebsd.org）** 在 **Mark Johnston（markj@freebsd.org）** 的指导下成功完成了一项项目，并将 **指令级跟踪（Instruction-level Tracing）** 功能合并到 **FreeBSD**。实现该功能的提供程序被称为 **kinst**。

**kinst** 复用了 **fbt** 机制的部分内容，并扩展了它，使其能够对 **内核函数的任意位置** 进行插桩（Instrumentation），而不仅仅是入口和出口点。对于熟悉 **内核开发** 的读者而言，**kinst** 在分析某些分支的调用栈时所带来的潜力不言而喻。因此，**kinst** 可以帮助更快地发现和修复 **FreeBSD** 中的 **bug** 及 **性能问题**。

以下是一个类似 **C 语言** 伪代码的示例场景：

```c
if (__predict_false(rarely_true)) {
return (slow_operation());
} else {
return (get_from_cache());
}
```

在这个示例中，我们聚焦于 **FreeBSD** 内核中的一个特定函数，其行为类似于此。该函数的简化版如下：

```c
void
_thread_lock(struct thread *td)
{
 ...
 if (__predict_false(LOCKSTAT_PROFILE_ENABLED(spin__acquire)))
 goto slowpath_noirq;
 spinlock_enter();
 ...
 if (__predict_false(m == &blocked_lock))
 goto slowpath_unlocked;
 if (__predict_false(!_mtx_obtain_lock(m, tid)))
 goto slowpath_unlocked;
 ...
 _mtx_release_lock_quick(m);
slowpath_unlocked:
 spinlock_exit();
slowpath_noirq:
 thread_lock_flags_(td, 0, 0, 0);
}
```

可以立即注意到有两个慢路径：`slowpath_unlocked` 和 `slowpath_noirq`。在这两个慢路径中，分别调用了 `spinlock_exit()` 或 `thread_lock_flags_()`，而 `_mtx_release_lock_quick()` 则只是在 amd64 上执行原子比较和交换指令。为了使用 `kinst` 来识别最终进入慢路径的调用堆栈，我们首先需要以某种方式反汇编该函数。一种可能的方法是在 **FreeBSD** 中使用 **kgdb**（通过 `pkg install gdb` 安装）：

```c
# kgdb
(kgdb) disas /r _thread_lock
Dump of assembler code for function _thread_lock:
...
0xffffffff80bc7dcc <+124>: 5d pop %rbp
 0xffffffff80bc7dcd <+125>: e9 4e 72 09 00 jmp 0xffffffff80c5f020
<witness_lock>
 0xffffffff80bc7dd2 <+130>: 48 c7 43 18 00 00 00 00 movq $0x0,0x18(%rbx)
 0xffffffff80bc7dda <+138>: e8 e1 43 4e 00 call 0xffffffff810ac1c0
<spinlock_exit>
 0xffffffff80bc7ddf <+143>: 8b 75 d4 mov -0x2c(%rbp),%esi
...
 0xffffffff80bc7df2 <+162>: 41 5d pop %r13
 0xffffffff80bc7df4 <+164>: 41 5e pop %r14
 0xffffffff80bc7df6 <+166>: 41 5f pop %r15
 0xffffffff80bc7df8 <+168>: 5d pop %rbp
 0xffffffff80bc7df9 <+169>: e9 82 00 00 00 jmp 0xffffffff80bc7e80
<thread_lock_flags_>
```



在这种情况下，我们可以取偏移量 +138 和 +169 处的指令，这些指令分别是对 `spinlock_exit()` 和 `thread_lock_flags_()` 的函数调用。使用这些偏移量，我们现在可以编写我们的 DTrace 脚本：

```c
# dtrace -n 'kinst::_thread_lock:138,kinst::_thread_lock:169 { @[stack(),
probename] = count(); }'
...
 0xcf566bb0
 kernel`ipi_bitmap_handler+0x87
 kernel`0xffffffff810a48b3
 kernel`vm_fault_trap+0x71
 kernel`trap_pfault+0x22d
 kernel`trap+0x48c
 kernel`0xffffffff810a2548
 138 8
```

熟悉 DTrace 的人可能会注意到，这本可以通过使用推测性追踪而不需要使用 kinst 来轻松实现。然而，人们可以很容易地想象出一些场景，其中“慢路径”或其等效物并不是一个简单的函数调用，或者相同的函数调用可能出现在所有的分支中。kinst 对 FreeBSD 上的 DTrace 生态系统也有其他影响。历史上，使用 fbt 对内联函数的内核进行插桩存在一个问题。用于实现 kinst 的机制可以帮助扩展 fbt，以支持可靠地追踪内联函数。

## 正在进行的工作

### DTrace 和 eBPF – 比较

Mateusz Piotrowski (0mp@FreeBSD.org) 一直在研究 FreeBSD 上 DTrace 的性能分析，以及它与 Linux 上 eBPF 的比较。今年，他在 EuroBSDcon 2022 上展示了部分结果。这项工作可能会得出有趣的结果，可以作为进一步优化 DTrace 的基础。这将使在性能关键的系统上启用插桩变得更少干扰。

### HyperTrace

HyperTrace 是建立在 DTrace 之上的一个框架，允许用户使用 D 编程语言应用类似 DTrace 的追踪技术来追踪虚拟机。它源自英国剑桥大学的 CADETS 项目。作为一个简单的例子，我们来看一下最初的 snooper 脚本，并扩展它以使用 HyperTrace：

```c
# dtrace -x quiet -En 'FreeBSD-14*:proc:::exec { printf(“%s: user = %u, gid = %u:
%s\n”, vmname, uid, gid, stringof(args[0])); }'
scylla1-webserver-0: user = 0, gid = 0: /usr/sbin/dtrace
scylla1-webserver-0: user = 0, gid = 0: /sbin/ls
scylla1-webserver-0: user = 0, gid = 0: /bin/ls
scylla1-client-0: user = 0, gid = 0: /usr/sbin/sshd
scylla1-client-0: user = 0, gid = 0: /bin/csh
scylla1-client-0: user = 0, gid = 0: /usr/bin/resizewin
scylla1-client-0: user = 0, gid = 0: /usr/sbin/iperf
scylla1-client-0: user = 0, gid = 0: /usr/bin/iperf
scylla1-client-0: user = 0, gid = 0: /usr/local/bin/iperf
host: user = 0, gid = 0: /bin/sh
host: user = 0, gid = 0: /usr/libexec/atrun
scylla1-client-0: user = 0, gid = 0: /bin/sh
scylla1-client-0: user = 0, gid = 0: /usr/libexec/atrun
scylla1-client-1: user = 0, gid = 0: /bin/sh
scylla1-client-1: user = 0, gid = 0: /usr/libexec/atrun
scylla1-client-2: user = 0, gid = 0: /bin/sh
scylla1-client-2: user = 0, gid = 0: /usr/libexec/atrun
scylla1-client-3: user = 0, gid = 0: /bin/sh
scylla1-client-3: user = 0, gid = 0: /usr/libexec/atrun
scylla1-webserver-0: user = 0, gid = 0: /bin/sh
scylla1-webserver-0: user = 0, gid = 0: /usr/libexec/atrun
```

我们修改了脚本，以实现两个新的功能：使用内建变量 `vmname` 指定每个进程执行的位置前缀，以及在探针规范中添加第五个元组条目：`FreeBSD-14*`。这允许用户指定要插桩的目标机器（虚拟机），并可以通过命令行标志来控制，支持通过操作系统版本或机器的主机名进行名称解析。类似的修改也可以应用到我们的块 I/O 示例中：

```c
# dtrace -x quiet -En 'scylla1-*:io:::start { @[vmname, immstack()] = count(); }'
...
 scylla1-webserver-0
 devstat_start_transacti+0x90
 g_disk_start+0x316
 g_io_request+0x2d7
 g_part_start+0x289
g_io_request+0x2d7
 g_io_request+0x2d7
 ufs_strategy+0x83
 VOP_STRATEGY_APV+0xd2
 bufstrategy+0x3e
 bufwriteL+0x80
 vfs_bio_awrite+0x24f
 flushbufqueues+0x52a
 buf_daemon+0x1f1
 fork_exit+0x80
 fork_trampoline+0xe
 aio_process_rw+0x10c
 aio_daemon+0x285
 fork_exit+0x80
 fork_trampoline+0xe
 fork_trampoline+0xe
 10598
 scylla1-client-0
 g_disk_start+0x316
 g_io_request+0x2d7
 g_part_start+0x289
 g_io_request+0x2d7
 g_io_request+0x2d7
 ufs_strategy+0x83
 VOP_STRATEGY_APV+0x9e
 bufstrategy+0x3e
 bufwriteL+0x3e
 vfs_bio_awrite+0x24f
 flushbufqueues+0x52a
 buf_daemon+0x1f1
 fork_exit+0x80
 fork_trampoline+0xe
 fork_trampoline+0xe
 aio_process_rw+0x10c
 aio_daemon+0x285
 fork_exit+0x80
 fork_trampoline+0xe
 fork_trampoline+0xe
 10605
```

在这里，使用了一个新的 DTrace 动作 `immstack()`，它与 `stack()` 类似，但符号解析发生在内核中，而不是在打印输出时。HyperTrace 的工作原理是旨在在主机内核上执行整个 D 脚本，而不是在客户端内部运行 DTrace，同时每个客户机负责自行插桩，并在客户机执行探针时向主机发出同步的超调用（类似于操作系统中的系统调用）。这种设计使得能够在一个地方保持跨所有客户机和主机的全局状态——提高了在追踪虚拟机时 D 的整体表现力。该工作仍在进行中，可以在 GitHub 上查看。

## 进一步阅读

1. [https://illumos.org/books/dtrace/preface.html#preface](https://illumos.org/books/dtrace/preface.html#preface)  
2. [https://github.com/opendtrace](https://github.com/opendtrace)  
3. [https://docs.freebsd.org/en/books/handbook/dtrace/](https://docs.freebsd.org/en/books/handbook/dtrace/)  
4. [https://www.cs.princeton.edu/courses/archive/fall05/cos518/papers/dtrace.pdf](https://www.cs.princeton.edu/courses/archive/fall05/cos518/papers/dtrace.pdf)  
5. [https://www.brendangregg.com/dtracebook/](https://www.brendangregg.com/dtracebook/)  
6. [https://wiki.freebsd.org/DTrace/One-Liners](https://wiki.freebsd.org/DTrace/One-Liners)  
7. [https://freebsdfoundation.org/wp-content/uploads/2014/05/DTrace.pdf](https://freebsdfoundation.org/wp-content/uploads/2014/05/DTrace.pdf)  
8. [https://issue.freebsdfoundation.org/publication/?m=29305&i=417423&p=14&ver=html5](https://issue.freebsdfoundation.org/publication/?m=29305&i=417423&p=14&ver=html5)  
9. [http://www.onlinedigeditions.com/publication/?m=29305&i=536657&p=4&ver=html5](http://www.onlinedigeditions.com/publication/?m=29305&i=536657&p=4&ver=html5)  
10. [https://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html](https://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html)  
11. [https://papers.freebsd.org/2018/bsdcan/teske-all_along_the_dwatch_tower/](https://papers.freebsd.org/2018/bsdcan/teske-all_along_the_dwatch_tower/)  
12. [https://github.com/freebsd/freebsd-papers/pull/112](https://github.com/freebsd/freebsd-papers/pull/112)  
13. [https://github.com/cadets/freebsd](https://github.com/cadets/freebsd)

---

**DOMAGOJ STOLFA** 是剑桥大学的研究助理，专注于虚拟化系统的动态追踪。他自 2016 年起就一直在 FreeBSD 上与 bhyve 和 DTrace 合作，贡献补丁。Domagoj 还是剑桥大学高级操作系统课程的教学助理，教授使用 FreeBSD、DTrace 和 PMC 的操作系统概念。
