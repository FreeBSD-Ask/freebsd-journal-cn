# 告别 printf：用 DTrace 深入内核

- 原文：[No More Printfs: Digging into the Kernel with DTrace](https://freebsdfoundation.org/our-work/journal/browser-based-edition/video-drivers/)
- 作者：**Mark Johnston**

DTrace 是一款通用性能分析与追踪工具，最初由 Sun Microsystems 为 Solaris 开发，后来移植到许多其他操作系统。FreeBSD 最近几个主要版本都包含了 DTrace 实现，从 FreeBSD 9.2 和 10.0 起内核已默认在 GENERIC 中启用 DTrace 支持。

DTrace 是窥探 FreeBSD 内核内部运作的利器；它的动态特性让开发者和系统管理员无需像其他许多内核调试工具那样耗时重新编译内核并重启，就能回答各种问题。因此，DTrace 应当是每位内核开发者工具箱中的常备工具。但和任何工具一样，DTrace 也有自身的局限和问题；了解这些局限并掌握规避方法很重要。

本文将通过若干精选的 DTrace 技巧，展示如何回答关于内核行为的问题。DTrace 常被提及的一项能力，是实时探测任意内核函数的进入与返回，并打印其参数和返回值。这本身就很有用，但追踪或隔离内核中的复杂行为时往往不够：有些函数（如 `tcp_do_segment()`）极长，函数级追踪作用有限。你可能还希望在探测点之间保存状态，比如追踪内存分配时。我们将通过例子逐步展开更高级的 DTrace 技巧来回答问题、排查故障。此外，本文还会指出一些常见错误与陷阱。

我们先为用过 DTrace 的读者做个快速回顾，想要全面入门的读者强烈建议阅读 Brendan Gregg 和 Jim Mauro 的《The DTrace Book》[1]。本文余下部分假设读者对 DTrace 和 FreeBSD 内核内部有基本了解。

## 极简概览

DTrace 本身由若干组件构成：

- `dtrace(1)` 是 DTrace 的主命令行接口，用于执行 DTrace 脚本、向内核查询 DTrace 探测点信息，以及完成若干构建期功能。FreeBSD 还包含若干更专用的基于 DTrace 的工具，包括 `lockstat(1)` 和 `dtruss(1)`。
- D 编程语言用于编写 DTrace 程序，由 libdtrace 编译为字节码后交给内核执行。
- DTrace 探测点是 DTrace 的插桩点。D 脚本可启用内核或用户态程序中的一个或多个探测点。被启用的探测点触发时，脚本决定是否执行动作以及执行什么动作。在 FreeBSD 系统上运行 `dtrace -l` 可查看所有可用探测点。DTrace provider 是相关探测点的集合，例如 FBT（function boundary tracing）provider 在每个内核函数开头定义一个探测点，名为 `fbt::<函数名>:entry`；IP provider 定义了一小组对应 FreeBSD IPv4/IPv6 协议栈事件的探测点。

D 程序由一组“探测点-谓词-动作”元组构成——每个元组指定一个或多个探测点、当某个探测点触发时要执行的动作，以及一个可选的谓词，该谓词在探测点触发时动态决定是否执行动作。语法上，每个元组类似 AWK 程序：

```d
probe
/predicate/
{
    actions
}
```

例如，下面这个 D 脚本会打印 re1 接口上收到的每个 IPv6 数据包的源地址：

```d
ip:::receive
/args[2]->ip_ver == 6 && args[3]->if_name == "re1"/
{
    printf("%s", args[2]->ip_saddr);
}
```

这个例子中，`ip:::receive` 探测点在内核收到 IP 数据包时触发，源地址存于 `args[2]->ip_saddr` 并由 `printf()` 打印，仅当数据包的 IP 版本为 6 且接收接口名为 re1 时才执行 `printf()`。把脚本复制到文件（通常以 `.d` 为扩展名）后执行 `dtrace -s script.d` 即可运行。

`ip:::receive` 探测点是静态定义的探测点——探测点嵌在内核中的一处或多处，禁用时几乎无开销。DTrace 首次加载时还会动态创建一批探测点——这些是动态探测点。最主要的一类是 FBT provider，它为内核中的每个 C 函数创建进入和返回探测点。这些探测点能让你精细观察内核行为；但随着内核内部细节在不同版本间变化，为某个内核编写的 D 脚本可能无法在另一个内核上运行。静态定义的探测点更稳定，因为它们对应的是高层逻辑事件，而非绑定于实现细节。

## 追查内存泄漏

动态探测点尽管有短板，在缩小问题来源范围时往往很有用。调查阶段写的 D 脚本通常是一次性工具，只为隔离某种具体行为，所以探测点稳定性不是问题。`dtrace(1)` 的输出也可高度定制，把 DTrace 与其他程序组合使用往往很方便。对 D 脚本的原始输出做后处理是常见做法，比如用于生成火焰图[2]。

定位内存泄漏的原因可能是个艰苦的过程。内核里尤其如此——既缺乏用户态程序那样成熟的调试工具，`malloc()` 接口又有大量各异的调用者。FreeBSD 的 DTrace 实现提供 dtmalloc provider，能帮助快速定位内存泄漏来源。介绍它之前，我们先回顾一下 FreeBSD 内核提供的 `malloc()` 接口：

```c
void *malloc(unsigned long size, struct malloc_type *type, int flags);
void free(void *addr, struct malloc_type *type);
```

注意内核接口包含一个 malloc type 参数，用于区分不同子系统并按子系统跟踪分配统计。运行 `vmstat -m` 可查看各种 malloc type 及其使用情况。

dtmalloc DTrace provider 为内存分配事件定义探测点，并为每种 malloc type 定义对应探测点。运行 `dtrace -l -P dtmalloc` 可查看这些探测点。下面这条单行命令能快速列出最活跃的子系统：

```sh
# dtrace -n 'dtmalloc:::malloc {@[probefunc] = count()}'
```

该程序在按 Ctrl-C 结束时，会按分配次数从多到少打印活跃的 malloc type 名：

```sh
...
jnewblk                                732
newblk                                 732
temp                                   918
jsegdep                               1381
rpc                                   3358
iov                                   3443
NFS_fh                                5817
```

值得注意的是 temp type——它用于不绑定任何特定内核子系统的短命分配。把这条单行命令扩展为按秒记录并展示最活跃的 malloc type 也很有用：

```d
dtmalloc:::malloc
{
    @types[probefunc] = count();
}

tick-1s
{
    printa(@types);
    trunc(@types);
}
```

在较新版本的 FreeBSD 上，可以用 DTrace 新的 aggpack 选项查看各子系统的分配请求大小分布：

```sh
# dtrace -n 'dtmalloc:::malloc {@[probefunc] = quantize(args[3]);}' -x aggpack
```

如果某子系统泄漏了 `malloc()` 分配的内存，可以借助 dtmalloc 和 fbt provider 定位导致泄漏的代码路径。考虑这样一个简单场景：运行一个使用 `pmc(3)` 的测试后，观察到 `hwpmc(4)` 在卸载时泄漏内存（通过控制台消息）。

```sh
Warning: memory type pmc leaked memory on destroy (8 allocations, 1024 bytes leaked).
```

修复的第一步是搞清楚内存何时、如何分配。`hwpmc(4)` 代码里大约有 45 处 `malloc(9)` 调用——单靠代码审查很难定位，尤其是不熟悉 `hwpmc(4)` 代码的人。这正是用 D 的 `stack()` 函数识别泄漏代码路径的好机会：

```d
#pragma D option quiet

dtmalloc::$1:malloc
{
    self->trace = 1;
}

fbt::malloc:return, fbt::contigmalloc:return
/self->trace == 1/
{
    printf("alloc 0x%p", args[1]);
    stack();
    self->trace = 0;
}

fbt::free:entry, fbt::contigfree:entry
{
    self->addr = (uintptr_t)args[0];
}

dtmalloc::$1:free
/self->addr != 0/
{
    printf("free 0x%p\n", self->addr);
    self->addr = 0;
}

fbt::free:return, fbt::contigfree:return
/self->addr != 0/
{
    self->addr = 0;
}
```

以 `pmc` 为参数运行时，该脚本会为每次 pmc 内存分配打印栈回溯以及所分配内存的地址；当 pmc 内存被释放时，打印其地址。脚本利用线程局部变量（以 `self->` 为前缀）在连续探测点之间保存状态。`dtmalloc::pmc:malloc` 探测点触发时，设置线程局部变量，让 DTrace 在 `malloc(9)` 返回时打印栈。同样，释放内存时记录被释放地址，以便确认类型正确后记录。`free(9)` 返回时把 `self->addr` 重置为 0 很重要，这保证 DTrace 释放与该变量关联的动态存储。

注意该脚本没有插桩 `realloc(9)`：`realloc(9)` 底层是通过 `malloc(9)` 分配新块、把原块内容复制到新块、再释放原块来实现的。因此如果 `realloc(9)` 分配的块泄漏了，我们会在最后一次 `realloc(9)` 调用时打印栈回溯。

运行脚本产生的输出可以用一段简短的 Perl 脚本做后处理：

```perl
my %stacks;
while (<>) {
    if (/^alloc (0x[a-f0-9]+)$/) {
        while (<>) {
            last if /^[af]/;
            stacks{$1} .= $_;
        }
    }
    if (/^free (0x[a-f0-9]+)$/) {
        delete $stacks{$1};
    }
}

foreach my $key (keys %stacks) {
    print $stacks{$key} . "\n";
}
```

把这段脚本作用在 D 脚本输出上会得到：

```sh
hwpmc.ko`pmc_syscall_handler+0x1fb6
kernel`amd64_syscall+0x25a
kernel`0xffffffff809342cb

hwpmc.ko`pmc_syscall_handler+0x1fb6
kernel`amd64_syscall+0x25a
kernel`0xffffffff809342cb
```

因此我们看到的内存泄漏必然源自 `pmc_syscall_handler+0x1fb6`；用 `kgdb(1)` 很容易找到对应代码：

```sh
# kgdb
...
(kgdb) list *pmc_syscall_handler+0x1fb6
```

结果位于 PMC_OP_PMCALLOCATE 处理函数中，该函数负责分配 PMC。既然定位了泄漏内存的来源，找到内存泄漏的根因就大有希望了。

继续之前，值得再细看上面的 D 脚本。另一种做法是在 D 脚本里用按分配地址索引的关联数组跟踪内存（去）分配。这种方式也有用，但有几点不足：

- DTrace 为全局变量预留的内存是固定的。如果被追踪的子系统有大量长期分配，DTrace 可能存不下。
- DTrace 没有把栈回溯存入变量的简便方法；下面这种写法是不允许的：

```d
fbt::malloc:return, fbt::contigmalloc:return
/self->trace == 1/
{
    stacks[args[1]] = stack();
    self->trace = 0;
}
```

不过可以把 `stack()` 函数的输出作为聚合键。

- 后处理更灵活。如果被追踪的子系统总有若干未释放的分配，就要小心区分它们和泄漏的内存。给后处理方案加时间戳很直接；而在关联数组里存额外数据会增加开销——既增加探测点动作的执行时间，也增加存储所需的内存。

## 建立关联

用 DTrace 构建监控工具时，一个常见难点是所需数据可能散落在多个探测点。考虑按进程监控 UDP 流量的任务。自然的起点是用 `udp:::send` 和 `udp:::receive` 探测点统计字节数或包数。但 FreeBSD 上这些探测点的参数无法识别负责流量的进程。`curthread` 变量在这里也帮不上忙——数据包从套接字缓冲区到网络接口之间一般由 `netisr(4)` 线程处理，这些是专门负责网络数据包协议处理的专用中断优先级线程：

```sh
# dtrace -n 'udp:::receive {printf("%s", curthread->td_name);}'
dtrace: description 'udp:::receive ' matched 1 probe
CPU     ID                    FUNCTION:NAME
1  37374                         :receive swi1: netisr 0
6  37374                         :receive swi1: netisr 0
5  37374                         :receive swi1: netisr 0
5  37374                         :receive swi1: netisr 0
```

到这里，你可能想放弃了。从“udp”provider 探测点的上下文确实无法直接找到关联的进程。但我们可以利用一个事实：`udp:::send` 和 `udp:::receive` 第二个参数的 `cs_cid` 字段是指向该数据包关联的 `struct inpcb`（互联网协议控制块）的指针。该结构保存 TCP 和 UDP 套接字的连接状态，其中特别包含指向套接字的指针。借助 FBT 探测点，我们可以在进程创建 UDP 套接字时执行动作，用关联数组把 PCB 映射到进程的 PID。然后在 UDP 探测点中用 PCB 地址查找 PID：

```d
fbt::udp_attach:entry
{
    self->so = args[0];
}

/*
 * 等到 udp_attach() 成功返回后再写入映射——出错时什么也不做。
 */
fbt::udp_attach:return
/args[1] == 0 && self->so != NULL/
{
    procs[(uintptr_t)self->so->so_pcb] = curproc->pr_pid;
    self->so = 0;
}

/*
 * 套接字不再存在时，务必释放数组条目。
 */
fbt::in_pcbdetach:entry
{
    procs[(uintptr_t)args[0]] = 0;
}
```

然后在 UDP 探测点中，用 PCB 地址在 procs 数组中查找：

```d
udp:::send, udp:::receive
/procs[args[1]->cs_cid] != 0/
{
    /* 用 PID 做点什么。 */
}
```

当然，如果套接字在脚本运行前已经创建，该脚本无法记录其 PID。除了在 `udp_attach()` 和 `udp_detach()` 中更新 procs 数组，还可以在进程对 UDP 套接字执行 I/O 时顺手更新映射：

```d
fbt::sosend_dgram:entry, fbt::soreceive:entry
/args[0]->so_proto->pr_protocol == IPPROTO_UDP/
{
    procs[(uintptr_t)args[0]->so_pcb] = curproc->pr_pid;
}
```

这样我们假设的监控工具既能处理长连接，也能轻松处理简短的 DNS 查询。

这种通过构建查找表来收集信息的通用技巧相当强大，不过需要你对内核及各数据结构之间的关系有一定了解。另一个应用是把文件路径映射到 vnode。vnode 是 FreeBSD 内核中文件在内存中的表示，用于在文件被访问时缓存各种信息。但用于查找文件的文件系统路径存储在另一个缓存——name cache 中。路径并不与 vnode 存在一起，所以给定一个 vnode，在 D 脚本中没有直接的方法找到关联路径。但我们可以挂钩 name cache 查找函数，以 vnode 地址为键构建映射表：

```d
fbt::vn_fullpath1:entry
{
    self->vn = args[1];
    self->buf = args[4];
}

fbt::vn_fullpath1:return
/args[1] == 0 && self->buf/
{
    paths[self->vn] = stringof(*self->buf);
    self->vn = 0;
    self->buf = 0;
}

vfs::lookup:hit
/paths[args[0]] != "" && paths[args[2]] == ""/
{
    paths[args[2]] = strjoin(paths[args[0]], strjoin("/",
        stringof(args[1])));
}

fbt::cache_purge:entry
/paths[args[0]] != ""/
{
    paths[args[0]] = 0;
}
```

## 查看进程参数向量

execsnoop 是 Brendan Gregg 编写的经典 DTrace 脚本，可在 DTrace toolkit[3] 中找到。它通过在 `execve(2)` 系统调用返回时追踪 `curpsinfo->pr_psargs` 变量（定义在 **/usr/lib/dtrace/psinfo.d**），让用户实时观察进程执行；简化版如下：

```d
#pragma D option quiet

syscall::execve:return
{
    printf("%s\n", curpsinfo->pr_psargs);
}
```

在 FreeBSD 系统上运行 execsnoop（或本例）可能产生一些看起来吓人的警告：

```sh
...
cc --version
sh -c echo 3.4.1 | awk -F. '{print $1 * 10000 + $2 * 100 + $3;}'
awk -F. {print $1 * 10000 + $2 * 100 + $3;}
dtrace: error on enabled probe ID 1
(ID 39505: syscall:freebsd:execve:return):
invalid address (0x4) in action #1 at DIF offset 136
```

这其实是 FreeBSD 内核某些行为的副作用：内核会在与进程关联的 `struct proc` 中缓存进程参数，但仅当它们能放入 `kern.ps_arg_cache_limit` sysctl 定义的限额时才缓存。该 sysctl 默认值是 256，对很多程序来说不够。一种办法是按需调大 sysctl 值，但这并不理想——依赖自定义系统配置的脚本难以复用，修改这一配置还可能带来难以预料的后果。

但有一个时刻内核中能拿到完整的参数向量——进入 `kern_execve()` 函数时。因此另一种做法是，在 `execve(2)` 首次进入内核时复制参数，待 DTrace 确认 `execve(2)` 系统调用成功后再打印。这可以借助 DTrace 推测（speculation）完成：

```d
#pragma D option nspec=32
#pragma D option quiet
#pragma D option strsize=4096

syscall::execve:entry
{
    self->argv = speculation();
}

fbt::kern_execve:entry
/self->argv/
{
    speculate(self->argv);
    printf("%s\n", memstr(args[1]->begin_argv, ' ',
        args[1]->begin_envv - args[1]->begin_argv));
}

syscall::execve:return
/args[1] == 0 && self->argv/
{
    commit(self->argv);
    self->argv = 0;
}

syscall::execve:return
/args[1] != 0 && self->argv/
{
    discard(self->argv);
    self->argv = 0;
}
```

这段脚本用到了 DTrace 几个不那么常见的特性。首先是 `self->argv` 变量与多个 D 函数配合使用，这是 DTrace 推测追踪特性的应用——它让我们能在尚不确定是否需要某份数据时先把它捕获下来。这里我们用 `kern_execve()` 的参数提取程序参数，但不能在 `fbt::kern_execve:entry` 探测点直接打印，因为系统调用可能失败。解决办法是先推测性地追踪参数；等确认系统调用成功后再打印，否则就丢弃这份字符串。

推测涉及四个 D 函数。首先用 `speculation()` 分配一个推测缓冲区，把句柄存入 `self->argv` 线程局部变量，作为参数传给其他推测函数。`speculate()` 为该探测点余下部分启用推测追踪——数据采集动作的输出暂存到推测缓冲区以备后用。最后，`commit()` 和 `discard()` 分别把推测数据存入 DTrace 的输出缓冲区或丢弃。DTrace 只能分配固定数量的推测缓冲区；可用 `nspec` 选项（如上例所示）调整可用缓冲区数量。DTrace 推测追踪特性的完整文档见[4]。

本例另一个不常见之处是 `memstr()` D 函数。撰写本文时该函数为 FreeBSD 独有，专门为处理 FreeBSD 内核中参数字符串的内存布局而添加。比如字符串 `wc -w article.txt` 会被存储为：

```sh
w   c   \   -   w   \   a   r   t   i   c   l   e   .   t   x   t   \
```

其中 `\` 表示空字节。和 C 中一样，D 字符串以空字符结尾，所以对上述字符串用 `printf()` 只会返回 `wc`。由于 D 无法遍历字符串的每个组成部分，所以添加了 `memstr()` 函数，把第一个参数中的所有空字节替换为第二个参数，从而转换为 D 字符串。第三个参数表示第一个参数的长度。

## 结语

希望上述例子能稍稍证明 DTrace 的适应能力。和任何自省工具一样，DTrace 也有能力边界；在函数调用级追踪内核执行和子程序参数确实是强大的特性，但单凭它窥探运行中的内核、回答其行为问题往往不够。此外，在任意内核上下文中追踪这一能力对探测点动作施加了许多限制。另一方面，DTrace 提供了许多特性来解决或至少绕过这些限制，FreeBSD 和 illumos 对 DTrace 的持续开发也在不断扩大它的用途。

---

**Mark Johnston** 是居住在西雅图的软件工程师。他来自多伦多，2013 年在滑铁卢大学取得数学学位，2010 年起成为 FreeBSD 用户。获得提交权限后，他的主要精力放在改进 FreeBSD 的 DTrace 实现上。可通过邮箱 <markj@FreeBSD.org> 联系他。

## 参考文献

[1] DTrace: Dynamic Tracing in Oracle Solaris, Mac OS X, and FreeBSD by Brendan Gregg and Jim Mauro, Prentice Hall 2011.

[2] CPU Flame Graphs, <http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html>

[3] DTrace Toolkit, <http://www.brendangregg.com/dtracetoolkit.html>

[4] Speculative Tracing, <https://wikis.oracle.com/display/DTrace/Speculative+Tracing>
