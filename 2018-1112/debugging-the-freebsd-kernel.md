# 调试 FreeBSD 内核

**作者：Mark Johnston**

写这篇文章时，期待已久的 FreeBSD 12.0 版本即将发布。当然，FreeBSD 开发者花数月时间精心打磨和测试 12.0 的内核，我们信心十足，认为所有 bug 都已修复。

对于典型的 FreeBSD 用户来说，内核调试这个主题只是出于好奇才会关注的话题，对吧？至少应该如此；FreeBSD 的稳定性是其主要卖点之一，为其声誉增色不少。操作系统内核的稳定性尤其重要，因为内核中的故障通常会让整个系统崩溃。然而事实上，无论 12.0 之前修复了多少 bug，残酷的现实是，一些 bug 仍然潜伏着，稍后会冒出来让用户吃苦头，通常引发令人紧张的内核 panic。

即使你不是 FreeBSD 开发者，熟悉一些 FreeBSD 内核调试也很有用。BSD 许可证宽松，许多公司得以在 FreeBSD 源码树上构建产品，常常直接扩展内核。这些扩展当然会包含 bug，或暴露 FreeBSD 自身潜在的 bug。此外，在这样的代码库上工作的开发者通常无法直接与 FreeBSD 社区合作，因此需要在向上游报告问题之前先自行做一些调试。另一种非常常见的情况是，开发者无法直接访问出现问题的系统，因此无法进行实时调试。此时，调试所需的数据须由管理员或用户收集。知道收集什么数据、如何收集，可以大大加快调试过程。

最后，熟悉我们的调试工具是成为熟练内核开发者的关键；作为新手，你不可避免会犯错，有些微不足道，有些则不然，而调试器常常能比盯着自己的代码更快揭示问题。

内核 bug 当然可能有多种不同的原因，但解决它们的高层方法总是相同的：

1. 找到 bug 的根本原因，或尽可能接近。
2. 如果既可能又必要，找到绕过 bug 的方法。
3. 修复 bug。

如果 bug 影响生产工作负载，步骤 2 可能比步骤 3 重要得多，而步骤 3 可能无法实现；硬件故障和固件 bug 是看似内核 bug 的常见来源，通常无法直接修复。然而，步骤 1 通常包括内核调试，是步骤 2 和 3 的先决条件。调试操作系统内核看似令人生畏，确实会带来一些独特的挑战，但方法论上与调试任何用户空间应用程序相同。

按大多数标准，FreeBSD 内核是庞大的代码库，除了琐碎的 bug 外，无法依靠代码检查和 debug-by-printf 走得很远。测试新内核必须重启，这也限制了这类原始调试技术。FreeBSD 提供各种设施用于事后调试，并帮助在没有已知原因的情况下为 bug 找到可复现的触发器，此时可调用调试工具解决问题。本文旨在突出其中一些设施，并提供一些使用技巧。

## 触发内核 Bug

意外的内核 panic 总令人不快。几乎同样糟糕的是，拼命尝试复现别人的内核 panic 却失败了。这通常发生在 bug 依赖时序时，如竞态条件或释放后使用（use-after-free）。这些情况没有银弹，但 FreeBSD 内核提供了若干设施，可能派上用场。

### INVARIANTS 内核

INVARIANTS 是在 FreeBSD 内核开发版本中配置的内核选项——如果你运行的不是 FreeBSD release 版本，很可能你正在使用配置了 INVARIANTS 的 GENERIC。这种情况下，恭喜你！你的内核启用了数千个完整性检查，不断测试程序员编写代码时的假设。这些检查带来运行时开销，使 INVARIANTS 不适合许多生产工作负载，但只要可行就应该启用。如果你向邮件列表报告 FreeBSD 内核 panic，且尚未使用 INVARIANTS 内核，很可能对方会要求你用 INVARIANTS 内核测试。其他内核选项，如 DIAGNOSTIC 和 DEBUG_VFS_LOCKS，启用进一步更严格的完整性检查，可以捕获许多 bug。

### 内存 Trash

FreeBSD 内核有一个 malloc(9) 实现，与常规 C 应用程序中使用的非常相似。此外，名为 UMA（“Universal Memory Allocator”）的内存子系统为高性能代码路径提供 slab 分配 API。当内核配置了 INVARIANTS 时，使用这些 API 之一释放的内存会被”trashed”，即其内容被特定模式覆盖。每次重用 trashed 内存时，其内容都会与该模式比对验证。如果验证失败，说明内存在释放后又被写入，内核 panic。

这是一种非常标准的技术，大多数功能齐全的内存分配器都实现了此技术。在 FreeBSD 上，trash 模式是 `0xdeadc0de`，因此虽然分配器本身会捕获 write-after-free，任何读取 trashed 内存位置（即 read-after-free）的代码如果将值当作指针，也很可能引发崩溃。调试或分诊内核 panic 时，留意 `0xdeadc0de` —— 它通常意味着 use-after-free。

内存 trash 让 use-after-free 所做的事清晰可见，使其更容易被发现。看下面的内存转储：

```sh
0xc00000037d3c0ba0:   0                        fffffe00017f2530
0xc00000037d3c0bd0:   fffff80002268e80         ffffffff80b35c58
0xc00000037d3c0be0:   deadc0dedeadc0de         deadc0dedeadc0de
0xc00000037d3c0bf0:   deadc0dedeadc0de         deadc0dedeadc0de
0xc00000037d3c0c00:   0                        deadc0dedeadc0de
0xc00000037d3c0c10:   deadc0dedeadc0de         deadc0dedeadc0de
0xc00000037d3c0c20:   deadc0dedeadc0de         deadc0dedeadc0de
0xc00000037d3c0c30:   deadc0dedeadc0de         deadc0dedeadc0de
```

这里我们可以看到 write-after-free 在结构中特定偏移处写入了 8 字节的零；这有助于识别出问题的代码。

use-after-free 是一种特别恶劣的 bug，因为 bug 的原因和结果可能相距甚远。内存 trash 能识别 use-after-free 是否发生，但无法当场捕获出问题的代码。这就是 MemGuard 派上用场的地方。

### MemGuard

MemGuard 是内存子系统特性，主动检测特定内存类型的 use-after-free。内存类型由 malloc(9) 的 type 参数或 UMA zone 定义。例如，这行代码：

```c
kq = malloc(sizeof *kq, M_KQUEUE, M_WAITOK | M_ZERO);
```

引用 M_KQUEUE，定义为：

```c
MALLOC_DEFINE(M_KQUEUE, "kqueue", "memory for kqueue system");
```

因此 M_KQUEUE malloc(9) 类型命名为“kqueue”。每个 malloc(9) 类型的统计信息可以用 `vmstat -m` 列出，UMA zone 的类似统计信息可以用 `vmstat -z` 列出。

MemGuard 默认不包含在 FreeBSD 内核中，必须通过将 DEBUG_MEMGUARD 选项添加到内核配置中编译进来。然后，要为特定内存类型配置 MemGuard，将 `vm.memguard.desc` 设置为 malloc(9) 类型或 UMA zone 的名称，方法是在 **/boot/loader.conf** 中添加条目或设置 sysctl。配置 MemGuard 后，它会挂钩该内存类型的所有分配和释放。每次分配都填充到完整页面，当一块内存被释放时，整个已分配内存范围会被取消映射。这意味着在内存释放后访问它，很可能导致页面错误，从而引发内核崩溃，在发生时捕获 use-after-free。

MemGuard 设施相当强大，但代价也不小。由于每次内存分配必须填充到完整页面，如果内存类型用于小型分配，将浪费大量内存。例如，在 mbuf zone 上启用 MemGuard 会导致每个 mbuf 浪费 4096 - 256 = 3840 字节。MemGuard 分配和释放钩子也会严重影响分配性能：取消映射内核内存是一项昂贵的操作，必须在系统所有 CPU 之间串行化。因此，在拥有许多 CPU 和高分配率的系统上，MemGuard 很容易成为主要瓶颈。如果 use-after-free 依赖时序（通常如此），MemGuard 的开销可能使 bug 不再出现。尽管如此，它在实践中相当有效。

### Fail Points

内核和用户空间中的许多 bug 都是错误处理不当或不充分的结果。这些 bug 长期潜伏，直到相应错误发生，因此通常执行的”happy path”不受影响。当错误确实发生时，原因可能是硬件或固件故障，或其他看似不太可能发生的事件。假设你需要调试错误处理问题，好不容易发现自己发现了 bug 并想验证这个想法。该从何入手？如何编写回归测试，确保修复在可预见的未来依然有效？

FreeBSD 的 fail point 子系统（文档见 fail(9)）在此可派上用场。Fail points 是一种机制，允许程序员在内核代码的预定义位置注入错误并修改行为。Fail points 默认没有任何效果，但可以使用 sysctl(8) 激活：每个 fail point 由唯一的 sysctl 控制，通常在 `debug.fail_point` 下。Fail points 可用于修改栈变量（通常注入错误）、改变控制流（通过 goto 或从函数提前返回）或睡眠、暂停（用于触发竞态条件）。

fail point 接口由一组宏组成，其中大多数是 KFAIL_POINT_CODE 的便捷包装器。例如，我们可以用 fail point 模拟 malloc(9) 的随机失败。在 malloc() 末尾添加以下 fail point 即可：

```c
void *
malloc(size_t size, struct malloc_type *mtp, int flags)
{
  ...
  if ((flags & M_WAITOK) == 0)
    /* M_WAITOK 分配不允许失败。 */
    KFAIL_POINT_CODE(DEBUG_FP, malloc, {
      free(va, mtp);
      va = 0;
    });
  return ((void *)va);
}
```

同一个 fail point 可以更简洁地定义：

```c
void *
malloc(size_t size, struct malloc_type *mtp, int flags)
{
    ...
    KFAIL_POINT_CODE_COND(DEBUG_FP, malloc, (flags & M_WAITOK) == 0, 0, {
      free(va, mtp);
      va = 0;
    });
    return ((void *)va);
}
```

无论哪种方式，在模拟分配失败之前，我们必须自行释放新分配的内存：否则该内存将永久泄漏。对于临时调试，这或许可以接受，但一般来说，必须确保 fail point 不引入新 bug。

定义了这个 fail point 后，我们得到一个名为 `debug.fail_point.malloc` 的 sysctl。Fail points 通过将相应的 sysctl 设置为描述当线程执行 fail point 时采取何种操作的字符串来激活。默认操作是”off”，意味着 fail point 没有效果。“return”操作会触发 fail point 的代码执行；手册页中定义了其他操作。如果所有 malloc(M_NOWAIT) 分配都失败，内核将无法长时间运行，因此 fail points 可以配置为以指定概率执行。设置

```sh
# sysctl debug.fail_point.malloc='1%*return'
```

将导致 fail point 的代码大约每 100 次 malloc(M_NOWAIT) 调用执行一次。

Fail points 在模拟磁盘错误时特别有用；这类错误想必相当罕见，但文件系统等必须稳健地处理它们。集成得当的话，fail points 可以成为此类代码回归测试方案的重要组成部分。例如，FreeBSD 的 RAID1 实现 gmirror 包含用于模拟底层磁盘错误的 fail points。随着时间推移，gmirror 错误处理中的许多 bug 已经修复，fail points 现已用于若干自动化的 gmirror 测试。

### 内核栈追踪

偶尔内核 bug 表现为线程在内核中陷入无限循环。系统不会崩溃，但细心的观察者可能会在 top(1) 中发现某任务在不明原因地消耗 100% CPU。如果系统不用于生产，可以进入内核调试器来询问相关任务，但这并不总能做到。

FreeBSD 可以用 procstat(1) 工具的 `-k` 选项在不暂停系统的情况下打印内核栈。`procstat -kk <pid>` 打印内核栈，`procstat -kka` 打印系统中每个线程的内核栈。虽然这不能直接帮助找到 bug 的触发器，但它为”卡住”无响应的任务，以及似乎无限循环的任务提供了入手点。当 procstat(1) 要捕获某个线程的栈，而该线程正在 CPU 上运行时，procstat(1) 会在该 CPU 上引发中断，从中断处理程序中捕获线程的栈帧。虽然这可能不足以当场分析出 bug 的根本原因，但它通常是有用的线索，而且它可以在不关闭整个系统的情况下使用，使其成为开始调查的便捷起点。

## 内核转储和 kgdb

kgdb 是许多内核调试会话的主力，尤其是在无法进行实时调试时。顾名思义，kgdb 是 gdb（GNU 调试器）针对 FreeBSD 内核的扩展。它维护活跃，作为 gdb 包的组件提供。与 gdb 类似，它可用于检查运行中的内核或检查内核 panic 的事后输出，通常称为”kernel dump”或”crash dump”。如果遇到内核 panic，开发者可能会要求你提供内核转储进行分析；操作上可能比较复杂，因此值得了解保存内核转储的各种可用选项。

内核转储实际上是 panic 时内核状态的完整副本，通常包含足够信息，可以完整还原崩溃原因。内核 panic 发生时，内核会记录一条消息，如果配置了，会跳转到特殊例程，尝试保存转储，然后自动重启系统。随后的启动过程中，savecore(8) 会检查并恢复之前保存的转储，将其存储在 **/var/crash**。

内核转储有两种类型：minidumps 和 full dumps。Full dumps 基本已被淘汰，用于尚不支持 minidump 的架构。区别在于 full dumps 包含 RAM 的全部内容，而 minidumps 仅包含崩溃时分配给内核的内存：未分配页面、用户内存和缓存文件会被省略。Minidumps 体积更小，但实现和支持更复杂。kgdb 可以处理这两种类型，因此区别通常不重要，甚至难以察觉。需要注意的是，内核转储通常包含敏感用户数据，应谨慎共享。

配置内核转储时的主要考虑因素是内核保存转储的位置。因为实际保存转储的代码在内核 panic 之后运行，它不能依赖内核正常运行，因此必须尽可能简单；我们绝对不希望它尝试写入文件系统！最简单的方法是在系统的 swap 分区上保存转储：swap 的内容在重启后本就无用，覆盖它也无妨。添加下面这行即可配置此行为：

```sh
dumpdev="AUTO"
```

到 **/etc/rc.conf** ——如果在 **/etc/fstab** 中配置了 swap 设备，它将用于内核转储。否则，要将转储保存到本地磁盘，必须指定磁盘设备名而不是“AUTO”。

配置转储设备时，最好触发一次 panic 测试，验证重启后 **/var/crash** 中是否出现 vmcore 文件：

```sh
# sysctl debug.kdb.panic=1
```

本地转储设备的主要困难是存储空间。内核转储大小大致与系统中 RAM 容量成正比。虽然 minidumps 大大减少了对空间的需求，但在拥有 128GB RAM 的系统上仍可能达到数十 GB，大于合理大小的 swap 分区。系统管理员要么浪费磁盘空间，要么可能无法保存内核转储。幸运的是，FreeBSD 12.0 带来若干新功能帮助缓解这个问题。首先，保存内核转储的代码现在可以在写入之前压缩数据，大大减少了空间需求。目前支持 zlib 和新式的 Zstandard 算法。Zstandard 在压缩内核转储方面既快又好，我们经常看到 8:1 或更好的压缩比。两种算法均可通过在 **/etc/rc.d** 中设置 `dumpon_flags` 来配置；详情见 dumpon(8) 手册页。因此 Zstandard 压缩配置起来很轻松：

```sh
dumpdev="AUTO"
dumpon_flags="-Z"
```

压缩暂时缓解了存储需求问题，但根本问题仍然存在。此外，FreeBSD 运行在许多根本就没有合适转储设备的系统上。例如，许多嵌入式设备（如路由器）从只读介质启动，没有任何动态存储。虚拟机通常没有 swap 空间，无盘服务器从 NFS 挂载启动，根本没有本地存储。为解决这些场景，FreeBSD 12.0 将支持通过名为 Netdump 的 IPv4 UDP 协议，经网络接口传输内核转储。

### Netdump

Netdump 让管理员和开发者可以用远程服务器接收来自客户端（panic 内核）的内核转储，提供了灵活性。Netdump 与现有的内核转储设施集成，对前面描述的内核转储特性是透明的；可以像转储到本地磁盘那样，对 Netdump 使用内核转储压缩或加密。从管理员的角度看，Netdump 只需额外配置。

Netdump 客户端使用 dumpon(8) 配置客户端参数。不指定块设备，而是指定要使用的网络接口名。这最好是低流量管理接口，而不是系统的数据端口：如果 Netdump 使用的接口在 panic 时繁忙，Netdump 的可靠性会下降。Netdump 不会自动支持所有网络适配器，但确实支持许多常见适配器，支持的驱动程序列表见 netdump(4) 手册页。当然，网络接口名不够：Netdump 客户端代码需要知道如何联系服务器！必须指定三个网络地址参数：

- 要使用的客户端 IP 地址（`-c`）。此参数是必需的。
- 服务器的主机名或 IP 地址（`-s`）。此参数是必需的。
- 客户端和服务器之间第一跳路由器的 IP 地址（`-g`）。即，将数据包从客户端转发到服务器的路由器地址。通常就是 `netstat -rn` 输出中列出的系统默认路由器。此参数是可选的：如果未指定，dumpon(8) 会使用系统已配置的默认路由（如果有），否则，假定客户端和服务器在同一链路上。

这些参数都可以在 **/etc/rc.conf** 中指定：

```sh
dumpdev="em0"
dumpon_flags="-s 192.168.1.34 -c 192.168.2.154 -g 192.168.2.1"
```

撰写本文时，仅支持 IPv4 参数。我们希望在未来的 FreeBSD 版本中支持 IPv6。

Netdump 客户端代码仅在系统 panic 后运行，它不能使用任何通常构建 IPv4 数据包并将其发送到网络的内核代码。尤其是，它不能使用内核的 ARP 缓存，因此第一步必须解析下一跳的以太网地址。事实上，Netdump 客户端总是先尝试直接解析服务器，然后再尝试解析网关。这样，如果客户端和服务器碰巧在同一链路上，它们可以直接通信。否则，客户端依赖网关将所有 Netdump 数据包转发到服务器。

Netdump 服务器程序名为 netdumpd，作为 FreeBSD 11 和 12 的软件包提供。可通过 **/etc/rc.conf** 配置；其主要参数指定根目录，所有来自客户端的内核转储都保存在此目录下。netdumpd 监听 UDP 端口 20023 以接收来自客户端的 HERALD 消息（如下所述）。使用临时端口，它用 ACK 消息回复所有客户端消息。当客户端收到其初始 HERALD 的 ACK 时，它将所有后续消息发送到 ACK 来源的 UDP 端口。当 Netdump 会话成功完成时，服务器在其根目录下保存两个文件：一个 vmcore 文件，与 savecore(8) 创建的相同；以及一个文本文件（info），包含描述内核转储的元数据。还可以选择配置在会话完成时（无论成功与否）执行用户定义的脚本。例如，这可用于在系统 panic 并传输内核转储时向系统所有者发送电子邮件。

Netdump 协议设计上极其简单，与 TFTP 协议非常相似。所有消息由固定大小的头部组成。客户端消息以：

```c
struct netdump_msg_hdr {
        uint32_t  mh_type;
        uint32_t  mh_seqno;
        uint64_t  mh_offset;
        uint32_t  mh_len;
        uint32_t  mh__pad;
};
```

而服务器消息长度始终为 4 字节：

```c
struct netdump_ack {
        uint32_t  na_seqno;
};
```

有五种 Netdump 客户端消息类型：

- 客户端发送 HERALD 消息表示内核转储开始。它包含可选有效负载：服务器根目录下的相对路径，用于保存转储。该路径必须在服务器上存在。
- 客户端完成转储时发送 FINISHED 消息。收到此消息后，服务器刷新其输出文件并调用用户指定的完成钩子。
- VMCORE 消息包含内核转储数据。`mh_offset` 字段指定数据在输出文件中的相对偏移量。
- KDH（“kernel dump header”）消息包含内核转储元数据，如客户端版本和 panic 消息。
- EKCD_KEY 包含加密的内核转储密钥（如果在客户端配置了内核转储加密）。否则不使用此消息。

服务器消息非常简单：通过包含相应的序列号来确认收到客户端消息。

## kgdb

经过一番触发 panic 并获取内核转储的工作后，是时候做一些实际调试了。使用 kgdb，打开内核转储就像使用 gdb 打开用户空间核心转储一样：你只需将 kgdb 指向内核二进制文件和 vmcore 文件：

```sh
# kgdb /boot/kernel/kernel /var/crash/vmcore.0
...
(kgdb) bt
#0  __curthread ()
#1  doadump (textdump=16777216)
#2  0xffffffff804ccdbc in db_fncall_generic (...)
#3  db_fncall (...)
#4  0xffffffff804cc8f9 in db_command (...)
#5  0xffffffff804cc674 in db_command_loop ()
#6  0xffffffff804cf91f in db_trap (...)
#7  0xffffffff8088b553 in kdb_trap (type=9, code=0, tf=<optimized out>)
#8  0xffffffff80badce1 in trap_fatal (frame=0xfffffe00188da6b0, eva=0)
#9  0xffffffff80bad1dd in trap (frame=0xfffffe00188da6b0)
#10 <signal handler called>
#11 0xffffffff808594cf in callout_process (now=1960148907297521)
#12 0xffffffff80c329b8 in handleevents (now=1960148907297521, fake=0)
#13 0xffffffff80c33231 in timercb (et=<optimized out>, arg=<optimized out>)
#14 0xffffffff80bbb289 in hpet_intr_single (arg=0xfffffe0000b590b0)
#15 0xffffffff80bbb32e in hpet_intr (arg=0xfffffe0000b59000)
#16 0xffffffff8080953d in intr_event_handle
    (ie=0xfffff80002169100, frame=0xfffffe00188da9a0)
#17 0xffffffff80c6b748 in intr_execute_handlers
    (isrc=0xfffff80002034e48, frame=0xfffffe00188da9a0)
#18 0xffffffff80c71d54 in lapic_handle_intr (vector=<optimized out>,
    frame=0x6f6bec1fb41fa)
#19 <signal handler called>
#20 acpi_cpu_c1 ()
#21 0xffffffff804f28a7 in acpi_cpu_idle (sbt=<optimized out>)
#22 0xffffffff80c684af in cpu_idle_acpi (sbt=59992010)
#23 0xffffffff80c68567 in cpu_idle (busy=0)
#24 0xffffffff80873c65 in sched_idletd (dummy=<optimized out>)
#25 0xffffffff808069a3 in fork_exit (...)
```

与 gdb 一样，需要调试信息才能让 kgdb 理解内核转储内容。在最近版本的 FreeBSD 中，这些符号外部存储在 **/usr/lib/debug/boot/kernel** 中，假设正在调试的内核位于 **/boot/kernel**。向开发者发送内核转储时，务必包含 **/boot/kernel** 和 **/usr/lib/debug/boot/kernel** 的内容。

kgdb 足够智能，可以展开 trap 帧，如上所示为 `<signal handler called>`。这发生在 CPU 中断运行线程时，将其状态保存在栈上。在示例中，我们有两个这样的帧：首先，定时器中断中断了空闲 CPU，在处理待处理 callout 时，该 CPU 因损坏指针导致保护错误：

```sh
(kgdb) frame 11
#11 0xffffffff808594cf in callout_process (now=1960148907297521)
510                                            LIST_REMOVE(tmp, c_links.le);
(kgdb) p tmp
$1 = (struct callout *)   0x11777be9162acbc1
(kgdb) x tmp
0x11777be9162acbc1:       Cannot access memory at address 0x11777be9162acbc1
```

大多数常用 gdb 命令在 kgdb 中有效。例如，`info threads` 会枚举系统的所有线程，包括仅限内核的线程，可以在它们之间切换。作为扩展，kgdb 可以基于 FreeBSD 线程 ID 识别线程。当数据结构指向特定线程且你想切换到该线程时，这很方便；例如，当当前线程被锁阻塞时，我们可能希望查看持有该锁的线程：

```sh
(kgdb) p kq
$5 = (struct kqueue *) 0xfffff801cb5a4a00
(kgdb) p/x kq->kq_lock
$6 = {
  lock_object = {
    lo_name = 0xffffffff814e8f66,
    lo_flags = 0x1430000,
    lo_data = 0x0,
    lo_witness = 0x0
  },
  mtx_lock = 0xfffff801e3b34000
}
(kgdb) p ((struct thread *)kq->kq_lock.mtx_lock)->td_tid
$7 = 101322
(kgdb) tid 101322 # 切换到 TID 为 101322 的线程。
(kgdb)
```

kgdb 提供脚本设施：GDB 脚本（即原生 GDB 命令语言，带有 if 语句和 while 循环等标准编程结构）和 Python API。GDB 脚本语言相当原始且有限，因此我们将专注于更强大的 Python 集成，并提供几个示例。API 文档相当详尽，因此示例仅描绘 API 的部分用途，而不解释其工作原理的每个细节。

内核的 struct thread 包含许多特定于某个子系统的字段；内核开发者通过向这个相当大的结构添加字段，有效地分配线程本地存储。调试时，常见需求是找到当前选中线程的指针。这通常可以临时完成，通过在栈的各个帧中搜索指针的副本，但很繁琐，且无法脚本化。我们可以用 Python 添加一个”便利函数”，其行为类似内核中的魔术 curthread 变量：

```python
import gdb

def _queue_foreach(head, field, headf, nextf):
    elm = head[headf]
    while elm != 0:
        yield elm
        elm = elm[field][nextf]
def list_foreach(head, field):
    return _queue_foreach(head, field, "lh_first", "le_next")
def tailq_foreach(head, field):
    return _queue_foreach(head, field, "tqh_first", "tqe_next")

def tdfind(tid, pid=-1):
    td = tdfind.cached_threads.get(int(tid))
    if td is not None:
         return td

    allproc = gdb.lookup_global_symbol("allproc").value()
    for p in list_foreach(allproc, "p_list"):
        if pid != -1 and pid != p['p_pid']:
             continue
        for td in tailq_foreach(p['p_threads'], "td_plist"):
            ntid = td['td_tid']
            tdfind.cached_threads[int(ntid)] = td
            if ntid == tid:
                return td
tdfind.cached_threads = dict()

class curthread(gdb.Function):
    def __init__(self):
         super(curthread, self).__init__("curthread")
    def invoke(self):
         return tdfind(gdb.selected_thread().ptid[2])
curthread() # 向 kgdb 注册函数。
```

API 提供获取所选线程 ID 的方法，但我们必须自己遍历内核的数据结构以获取相应的 struct thread 指针。这由 tdfind() 例程完成，它模拟同名的内核函数，搜索内核的全局进程列表（allproc），找到具有指定线程 ID 的线程。全局列表可能很大，因此我们缓存搜索过程中遇到的所有线程，加快后续查找。

curthread 类提供了必要的粘合：当调用 `$curthread()` 时，它只是对所选线程的 ID 调用 tdfind() 并返回结果：

```sh
(kgdb) p $curthread()
$1 = (struct thread *) 0xfffff801e3b34000
```

现在我们不必在栈中寻找当前线程指针，查找代码可复用于其他用途。

VIMAGE 加入标准内核配置后，网络中的许多全局数据结构都被虚拟化：对这种全局结构的每次引用实际上都相对于当前 vnet，因此每个引用都有一些隐含上下文。内核使用 vnet.h 中的一组宏来处理此问题，但 kgdb 不知道它们；结果是下面这样行不通：

```sh
502          if (pcbinfo == &V_tcbinfo) {
503                  INP_INFO_RLOCK_ASSERT(pcbinfo);
(kgdb) p V_tcbinfo
No symbol "V_tcbinfo" in current context.
```

不过，我们可以提供一个便利函数，在当前 vnet 上下文中解析符号，或者在碰巧有用时在用户指定的 vnet 中解析符号，以此接近目标：

```python
class vimage(gdb.Function):
    def __init__(self):
        super(vimage, self).__init__("V")

    def invoke(self, sym, vnet=None):
        sym = sym.string()
        if sym.startswith("V_"):
            sym = sym[len("V_"):]
        if gdb.lookup_symbol("sysctl___kern_features_vimage")[0] is None:
            return gdb.lookup_global_symbol(sym).value()

        if vnet is None:
            vnet = tdfind(gdb.selected_thread().ptid[2])['td_vnet']
            if not vnet:
                # 如果 curthread->td_vnet == NULL，vnet0 是当前 vnet。
                vnet = gdb.lookup_global_symbol("vnet0").value()
        base = vnet['vnet_data_base']
        entry = gdb.lookup_global_symbol("vnet_entry_" + sym).value()
        entry_addr = entry.address.cast(gdb.lookup_type("uintptr_t"))
        ptr = gdb.Value(base + entry_addr).cast(entry.type.pointer())
        return ptr.dereference()
vimage() # 向 kgdb 注册函数。
```

这个片段实质上重新实现了 VIMAGE 的真正实现所做的事：从当前线程获取指向当前 vnet 的指针，做一些数学运算以找到该 vnet 的所需结构副本。它还尝试在内核不支持 VIMAGE 时，仅返回全局结构以提供合理行为。此外，如果需要，可以显式指定所需的 vnet。虽然我们不能简单地打印 `V_tcbinfo`，但现在可以接近目标：

```sh
(kgdb) p $V("tcbinfo")
$1 = {
   ipi_lock = {
     lock_object = {
       lo_name = 0xffffffff8125b55e "tcp",
       ...
```

作为最后一个示例，我们将实现来自 DDB 的 acttrace 命令，它打印 panic 时所有在 CPU 上主动执行的线程的回溯。这通常是了解整体情况的重要部分，因为许多 bug 是多个线程之间错误交互的结果。幸运的话，导致 panic 的线程在 panic 时就在 CPU 上。

此示例添加新的顶级命令，模仿 DDB 的接口。它使用每 CPU（pcpu）结构，该结构存储系统中每个 CPU 的执行状态信息。

```python
import math

def cpu_foreach():
    all_cpus = gdb.lookup_global_symbol("all_cpus").value()
    bitsz = gdb.lookup_type("long").sizeof * 8
    maxid = gdb.lookup_global_symbol("mp_maxid").value()

    cpu = 0
    while cpu <= maxid:
        upper = cpu >> int(math.log(bitsz, 2))
        lower = 1 << (cpu & (bitsz - 1))
        if (all_cpus['__bits'][upper] & lower) != 0:
            yield cpu
        cpu = cpu + 1

    class acttrace(gdb.Command):
        def __init__(self):
             super(acttrace, self).__init__("acttrace", gdb.COMMAND_USER)

        def _inferior_thread_by_tid(self, tid):
             threads = gdb.inferiors()[0].threads()
             for td in threads:
                 if td.ptid[2] == tid:
                     return td

        def invoke(self, arg, from_tty):
            # 保存当前选中的线程。
            curthread = gdb.selected_thread()

            # 注意：并非所有平台都有 __pcpu 数组。
            pcpu = gdb.lookup_global_symbol("__pcpu").value()
            for cpu in cpu_foreach():
                td = pcpu[cpu]['pc_curthread']
                p = td['td_proc']
                self._inferior_thread_by_tid(td['td_tid']).switch()

                print("Tracing command {} pid {} tid {} (CPU {})".format(
                  p['p_comm'].string(), p['p_pid'], td['td_tid'], cpu))
                gdb.execute("bt")
                print

            # 切换回起始线程。
            curthread.switch()
    acttrace() # 向 kgdb 注册命令。
```

想法是迭代系统中所有在线 CPU，查找每 CPU 结构并使用它找到 CPU 上的线程。我们使用线程的 ID 查找 kgdb 中该线程的表示，切换到它，并打印回溯。这一切完成后，我们切换回起始线程，避免让用户困惑。

```sh
(kgdb) acttrace
Tracing command idle pid 10 tid 100002 (CPU 0)
#0  cpustop_handler ()
#1  0xffffffff80928074 in ipi_nmi_handler ()
#2  0xffffffff808c0e19 in trap (frame=0xffffffff8104b160 <nmi0_stack+3888>)
#3  <signal handler called>
#4  acpi_cpu_idle_mwait (mwait_hint=0)
...
```

## 结论

希望本文帮助阐明了 FreeBSD 内核开发者典型调试工作流程的几个阶段。当然，经验和直觉可以让你走得很远，但良好的工具在追踪更棘手的 bug 时极有帮助，尤其是当 bug 位于你的专业领域之外时。调试器还能帮助熟悉内核核心功能底层的各种模式和细节，因此随着经验积累，更容易发现”看起来不对劲”的事情：可能引导你找到根本原因的线索。调试愉快！

---

**Mark Johnston** 是自由职业 FreeBSD 开发者，居住在加拿大多伦多。业余时间他喜欢尝试用大提琴发出好听的声音、跑步、旅行，与朋友组队玩躲避球。
