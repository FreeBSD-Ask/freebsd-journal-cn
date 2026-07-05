# FreeBSD 与处理器趋势

作者：Matt Macy

在 Computex 2018 上，Intel 展示了一台 28 核原型系统。几个月内，AMD 推出了世界上最并行的桌面处理器 ThreadRipper 2，具有 32 个核心（64 个硬件线程）。AMD 的 EPYC2 已经在实验室中，据传是 64 核（128 个硬件线程），为商品服务器双路系统带来 256 个硬件线程。历史上，FreeBSD 一直存在于硬件商品曲线的“拐点”处。为了保持在服务器领域的相关性，FreeBSD 需要跟上最新的处理器发展。

## 处理器演进

随着核心数量增加，设计变得越来越复杂。AMD 的 Shanghai 和 Intel 的 Nehalem 使用广播总线来处理缓存一致性。Intel 的 Haswell 后来将其改为芯片上的多个环。而在 Skylake 上，Intel 转向了网状结构（见右图）。AMD 采取以良率为中心的扩展方式，在其产品线中重用相同的设计。每颗芯片由两个核心复合体（CCX）组成，一个 EPYC 封装由四颗芯片组成（通常被称为“chiplets”）（见右下）。

## 定义可扩展性

可扩展性可以从多个维度定义 [Culler, 1999]：

- 问题约束强扩展（Problem-Constrained strong scaling）——用户希望使用更大的机器更快地解决相同的问题。随着可用于完成任务的处理器数量增加，完成任务的时间减少程度：

```sh
                                     Time(1 processor)
SpeedupPC(n processors) = ---------------------------------
                                     Time(n processors)
```

- 时间约束弱扩展（Time-Constrained weak scaling）——执行给定工作负载的时间保持不变；用户希望解决尽可能大的问题。它是随着处理器数量增加完成的工作量增加的程度：

```sh
                                     Work(n processors)
SpeedupTC(n processors) = ---------------------------------
                                     Work(1 processor)
```

- 内存约束（Memory-Constrained）——用户希望解决能放入内存的最大问题。

```sh
                         Work(p processors)     Time(1 processor)          Increase in Work
SpeedupMC(p processors) = ------------------- x ---------------- = ----------------------------
                          Time(p processors)     Time(1 processor)      Increase in Execution Time
```

在本文中，可扩展性指的是时间约束可扩展性（“弱扩展”），将以基准测试期间执行的聚合操作数来表征。性能瓶颈是特定于应用程序和工作负载的。因此，从这些可扩展性测量中推断实际应用程序性能是有问题的。尽管如此，操作系统对任何给定工作负载的影响可以表示为每次系统调用的平均时间和调度决策影响的组合。系统调用开销可以通过简单的微基准测试捕获。调度决策更难测量，但可以通过测量具有不同调度器限制（即限制调度器可用的 CPU 集合）的工作负载，或通过比较单路结果与双路/多路结果来在一定程度上测量。

读者需要理解，微基准测试的目的不是测量工作负载本身。它们是观察单个操作系统服务扩展性以隔离测量可扩展性的手段。这些测量仅在真实世界工作负载使用所测单个服务的程度上，才能预测其性能。

## 什么使扩展困难——序列化和调度

如果 n 个线程试图执行一个操作，序列化开销可以大致定义为每线程吞吐量从 1 下降到 1/n 的程度，因为每个线程都在等待获取同一个锁。调度开销更难定义。在理想世界中，给定线程只会在一个核心上运行；与之通信的任何其他线程都会在同一个“核心复合体”上——共享 L3 缓存，这样 IPI（处理器间中断——允许 CPU 中断其他 CPU 的设施）和缓存一致性流量就不必穿越互连，任何未命中都可以在不访问内存的情况下重新填充。遗憾的是，在实践中，这在一般情况下是不可能的。CPU 通常被超额订阅，调度器无法推断不同进程中线程之间的关系。一个 CPU 离另一个 CPU 越远，网络和同步 IPC 等延迟敏感操作的性能就越差。两个 CPU 之间的距离越大，线程迁移时重新填充缓存的成本就越高。

扩展挑战源于三个因素：

- 内存延迟
- 一致性流量的限制
- 共享的全局唯一资源

在很大程度上，扩展解决方案都是以下内容的组合：

- 每 CPU 资源
- 放松约束
- 区分存在性保证和互斥

内存延迟和一致性流量的边界是过去十年计算机硬件演进的基础。曾经只在高端系统中看到的设计产物，现在即使在 AMD ThreadRipper 等消费级 CPU 中也是重要考量。共享内存编程模型正成为一个越来越有漏洞的抽象。处理器中的缓存一致性逻辑提供了程序员都已习惯的单写多读（SWMR）保证 [Sorin, 2011]。然而，在极限情况下，观察到的性能由分布式内存的实际实现定义，所有更新都通过消息传递执行 [Hackenberg, 2009], [Molka, 2015]。如今，消息延迟和带宽是观察性能的主导因素。

受硬件线程数量增加影响的实现问题：

- 锁定粒度
- 使用锁提供存在性保证
- 使用原子引用提供存在性保证
- L3 缓存或 NUMA 域之间缓存局部性差

### 锁定粒度

锁定粒度指单个锁保护的操作数量。Linux 中的“Big Kernel Lock”或“BKL”，或 FreeBSD 中的“Giant”最初将整个内核包含在单个锁中。这演变为单个子系统的锁，然后是单个数据结构的锁，最后是数据结构中的字段。即使有了细粒度锁定，也会出现只能一次访问一个的广泛引用的全局资源（内存、路由表条目等）的情况。一般来说，FreeBSD 中的锁定粒度已经相对细粒度。尽管如此，在 FreeBSD 11 和 FreeBSD 12 之间，有许多通过增加锁定粒度、转向每 CPU 资源或减少全局更新发生频率来减少锁争用的例子。

### 用于存在性保证的锁定

可以使用锁来保证系统全局或进程全局结构中的条目在使用时没有被释放。FreeBSD 11.x 与 FreeBSD 12.x 中的一个例子是如何为每协议哈希表中的连接状态提供存在性保证。FreeBSD 11.x 通过要求所有表读取者获取每表读写锁的共享（用于读取）获取，向线程保证表中找到的任何连接都是有效的。这允许多个同时读取者，同时阻止任何表更新。虽然概念上简单，但这代价高昂，而且保证比所需的更强。FreeBSD 12.x 弱化了保证，提供在查找期间找到的任何连接尚未被释放。查找受 epoch 保护，更新用互斥锁序列化。连接状态查找仍然返回锁定的连接以保证查找后的存在性。然而，一旦获取了锁，查找现在会检查连接是否设置了 INP_FREED 标志。如果设置了该标志，表示连接待释放。在这种情况下，我们释放锁并返回 NULL，就好像没有找到连接一样。此更改给读取者增加了一些额外的复杂性，但作为交换，我们不再需要 `rwlock` 的全局原子操作 [附录 A]，更新可以与查找并行进行（查找不再阻塞更新，反之亦然）。此更改在加载的多路服务器上将查找中花费的时间减少了 10–20 倍。

### 用于存在性保证的原子引用计数

原子地更新对象的引用计数器比使用锁序列化更新性能更好。更新可以完全并行进行，同时进行所有权更改。每个持有指向对象的指针的新线程或对象都会递增引用。当引用从对象移除或线程的引用超出范围时，引用递减。当计数变为零时，被引用对象被释放。尽管如此，随着一致性流量成本的上升，它不能很好地扩展。对于被许多线程频繁引用的对象，在 L2 和 L3 缓存之间使缓存行失效和迁移的一致性流量很快成为瓶颈。这里有两个独立的问题需要解决：

- 这里是否需要引用计数？
- 能否做些什么使引用计数更便宜？

也许令人惊讶的是，对于栈局部引用，引用计数实际上并不是必需的。SMR“安全内存回收”技术，如基于 Epoch 的回收 [Fraser, 2004]、Hazard Pointers [Michael, 2004; Hart, 2006]、RCU [McKenney, 2011]、可扩展并行段 [Wang, 2016] 等，可以让我们在不进行任何共享内存修改的情况下提供存在性保证。而且引用计数在许多情况下可以变得更便宜。

最近在 UDP 中的工作扩展了与网络栈 epoch 结构绑定的对象范围。epoch 结构现在也用于保证接口地址的存在性。这意味着对它们的栈局部引用不再需要更新对象的引用计数。

如果我们能正确处理零检测，观察到的引用计数可以安全地与“真实”引用计数不同。可扩展引用计数的不同方法都依赖于这一洞察。虽然文献中有其他方法 [Ellen, 2007]，但我认为最有趣的是 Linux 的 percpu refcount [Corbet, 2013] 和 Refcache [Clements, 2013]。前者是一个每 CPU 计数器，当初始引用持有者“杀死”percpu refcount 时，它会退化为传统的原子更新引用计数。它的优点是简单，如果对象的生命周期与初始引用持有者的生命周期密切对应，可以非常轻量。如果对象明显比初始所有者存活时间长，它的效果就不好。Refcache 维护一个每 CPU 的引用更新缓存，在冲突时或在一个“epoch”结束时刷新它们。在这种情况下，“epoch”是 10 毫秒。零检测通过在对象的全局引用计数达到零时将其放入每 CPU“审查”列表来完成。当全局引用计数保持为零两个“epoch”时，可以假定它为真实引用计数。Refcache 不依赖于具有密切相关的生命周期的初始引用持有者来避免退化状态。在某些方面，这使得它更具通用性。然而，可能多次通过审查队列会给零检测过程增加大量开销。初始候选释放和最终释放之间的延迟使其不适用于周转率高的对象。例如，10 毫秒的网络 mbuf 或 VM 页结构积压可能产生惩罚性的开销。

## 缓存局部性

为缓存局部性设计的简单示例是将结构连续打包，而不是使用链表，这样预取器可以在一个线程迭代它们时提供下一个元素。在高操作速率下，结构中字段排序的方式可以产生可测量的性能差异。当核心内存分配结构的重组将最常访问字段的缓存行数从三减少到二时，测量到每秒 `brk` 调用增加了 45%。

一旦消除了序列化瓶颈，内核性能就由缓存未命中的频率决定。

最小共享和缓存未命中是序列化和局部性的容易定义的理想。然而，算法上为任意的、前所未见的工作负载定义最优调度是不可实现的理想。即使信息存在，调度器可获得的数据结构知识和共享也会减慢调度决策。当共享 L2 缓存时，两个具有小工作集的通信线程会受益，而两个具有较大工作集的通信线程会受到不利影响。FreeBSD 由于多路插槽上的线程调度而失败的一个特别严重的例子是测量 TCP 到 localhost 连接的吞吐量。在约 3GHz 的单路系统上，FreeBSD 可以达到 50–60Gbps，部分通过将网络处理卸载到 `netisr` 线程。在相同时钟速度的双路系统上，测量的吞吐量下降到 18–32Gbps。在最坏的情况下，两个通信进程在一个插槽上，`netisr` 线程在另一个插槽上。因此，每个数据包的通知都必须穿越插槽之间的互连。至少在双路上，Linux 在执行 TCP 到 localhost 时内联进行网络处理（即没有服务线程）。在单路上，Linux 会比 FreeBSD 达到更低的吞吐量。然而，如果两个进程都调度在同一个插槽上，它能达到一致的 35Gbps。这里有许多问题需要解决：

- 整个系统只有一个 `netisr` 线程
- 发送者、接收者和 `netisr` 线程应该在哪里调度
- 如何向调度器传达这三个不同的线程正在相互通信

尽管如此，这里的关键洞察是，延迟可以决定可用带宽，当我们从单路移动到双路时，糟糕的调度决策可能对性能产生毁灭性的影响。

对于事先了解将要运行什么工作负载的用户，情况是可管理的。`cpuset` 命令允许将处理器集分配给进程，限制调度器可用的选择。

## 测量可扩展性

就本文而言，扩展测量将限于在两个系统上运行 FreeBSD 11、FreeBSD 12 和 Ubuntu 18（Linux 4.15.1）的“Will-it-scale”系统调用微基准测试套件 [Blanchard, 2013]——双路 EPYC 7601（2x32 核 @2.2Ghz）和双路 Intel Xeon 6130（2x16 核 @2.1Ghz）。这两者不能直接比较，因为 EPYC 7601 是顶级处理器，零售价比中端 Xeon 6130 高 130%。尽管如此，更复杂的 EPYC 可能会显示出非常不同的扩展特性和更高的局部性差惩罚。

首先要注意的是，大多数基准测试的多线程变体比它们的多进程对应物扩展得差得多。共享地址空间、文件描述符数组和进程结构都需要为多线程情况添加锁定。在基准测试扩展时，曲线通常在中间标记处变平。这是因为我们从每个核心一个基准测试线程或进程的机制转向超额订阅核心并使用两个硬件线程（1）。

从 FreeBSD 11 到 FreeBSD 12 有许多显著的改进。`getuid` 基准测试显示系统调用开销减少了 50% 以上。页面错误性能提高了 20–80 倍（2, 3, 4）。

128MB 匿名内存 `mmap`/`munmap` 有了显著改进，目前优于 Linux（5, 6）。

Unix 域套接字性能提高了 19 倍。性能以前在八个硬件线程处变平，现在继续增加到 128 个硬件线程（7）。至少在 Linux 4.15 中，FreeBSD 在 UNIX 套接字上实际上比 Linux 扩展得更好（8）。

单独文件读取仍在 32 个硬件线程处达到峰值，但这是 8 倍的改进（9, 10）。

遗憾的是，有一些领域的投资不足表现得相当明显。虽然通过更改交换预留的处理在 `brk` 基准测试中有所改进（11），但还有几个其他系统范围的序列化点。Linux 在这里几乎线性扩展，在峰值时能够执行 32 倍的 `brk` 操作/秒（12）。

可以说这在实践中不是什么大问题，因为它在真实世界工作负载中相对不突出。作为一类，FreeBSD 和 Linux 之间最令人不安的差异在于文件系统操作。Linux 在许多情况下几乎线性扩展，而 FreeBSD 在四个硬件线程处停止扩展（13, 14, 15, 16）。

如果有更多时间，我们会提供更多真实世界工作负载（如 nginx Web 服务器提供小型静态对象、memcached、PostgreSQL 等）的基准测试。

## 替代方法

我们从这里走向何方？基准测试可以识别系统执行的好坏，但特定于一个工作负载和配置。微基准测试对于识别系统瓶颈很有用。然而，它们不提供系统性保证不存在扩展限制的方法。是否有更一致地识别问题的方法？直到最近，答案都是否定的。然而，Clements 在 2014 年的工作 [Clements, 2014] 建立在不相交访问并行内存系统 [Israeli, 94] 的概念之上，严格识别了软件接口及其实现的可扩展性限制。这项工作提出了可扩展交换律规则，它本质上说，只要接口操作交换，就可以以可扩展的方式实现。其背后的直觉很简单：当操作交换时，它们的结果（返回值和任何副作用）与顺序无关。

他首先观察到许多可扩展性问题不在于实现，而在于软件接口的设计。不允许两个操作交换的接口定义在两次调用之间强制序列化。`open` 系统调用的 POSIX 定义要求它返回最低可用文件描述符。这意味着对不同文件的两个 `open` 调用需要在文件描述符分配上序列化。其他一些具有不必要不可扩展接口的系统调用有：`fork`（当立即后跟 `exec` 时）、`stat`、`sigpending` 和 `munmap`。

这是一个有趣的观察，但这项工作的真正贡献是开发了一个名为 COMMUTER 的工具：

1. 接受接口的符号模型，并计算该接口操作何时交换的精确条件。
2. 使用这些条件生成根据接口模型交换的操作集合的具体测试，因此根据交换律规则应该有无冲突的实现。
3. 检查特定实现对于每个测试用例是否无冲突。

他将此应用于 18 个 POSIX 系统调用，生成了 26,238 个测试用例，并用这些来比较 Linux 与 sv6（他的小组开发的研究操作系统）。他发现在 Linux 3.8 上 17,206 个用例可扩展，而在 sv6 上为 26,115 个。未能扩展的测试用例集合可以作为重新设计子系统的起点，正如 will-it-scale 基准测试使我们能够识别更窄的问题集（见图表左侧）。

将 COMMUTER 移植到 FreeBSD 上工作将是未来工作的有趣方向。•

## 参考文献

[Attiya, 2011] Attiya, H., Guerraoui, R., Hendler, D., Kuznetsov, P., Michael, M. M., Vechev, M. 2011. Laws of order: expensive synchronization in concurrent algorithms cannot be eliminated. In Proceedings of the 38th Annual ACM SIGPLAN-SIGACT Symposium on Principles of Programming Languages: 487-498; <http://doi.acm.org/10.1145/1926385.1926442>

[Blanchard, 2013] Will-It-Scale benchmark suite. <https://github.com/ScaleBSD/will-it-scale>.

[Boyd, 2010] S. Boyd-Wickizer, A. T. Clements, . Mao, A. Pesterev, M. F. Kaashoek, R. Morris, and N. Zeldovich. An analysis of Linux scalability to many cores. In Proceedings of the 9th Symposium on Operating Systems Design and Implementation (OSDI), Vancouver, Canada, Oct. 2010.

[Corbet, 2013] Corbet, J. Per-CPU reference counts, July 2013. <https://lwn.net/Articles/557478/>

[Clements, 2013] Clements, A. T., M. F. Kaashoek, and N. Zeldovich. RadixVM: Scalable address spaces for multithreaded applications. In Proceedings of the ACM EuroSys Conference, Prague, Czech Republic, April 2013.

[Clements, 2014] Clements, A. T. The scalable commutativity rule: Designing scalable software for multicore processors, Ph.D. dissertation, Massachusetts Institute of Technology, Jun. 2014. [Online]. Available: <https://pdos.csail.mit.edu/papers/aclements-phd.pdf>

[Culler, 1999] Culler, D. Singh, J. P., Gupta, A. Parallel Computer Architecture - A Hardware / Software Approach, Morgan Kaufman, 1999

[Ellen, 2007] F. Ellen, Y. Lev, V. Luchango, and M. Moir. SNZI: Scalable nonzero indicators. In Proceedings of the 26th ACM SIGACT-SIGOPS Symposium on Principles of Distributed Computing, Portland, OR, Aug. 2007.

[Fraser, 2004] Fraser, K. Practical lock-freedom, Ph.D. Thesis, University of Cambridge Computer Laboratory, 2004

[Hackenberg, 2009] D. Hackenberg, D. Molka, and W. Nagel. Comparing cache architectures and coherency protocols on x86-64 multicore SMP systems. MICRO 2009, pages 413–422.

[Hart, 2007] Hart, T. E., McKenney, P. E., Demke Brown, A., Walpole, J. 2007. Performance of memory reclamation for lockless synchronization. Journal of Parallel and Distributed Computing 67(12): 1270-1285; <http://dx.doi.org/10.1016/j.jpdc.2007.04.010>

[Herlihy, 2008] Herlihy, M., Shavit, N. 2008. The Art of Multiprocessor Programming. San Francisco: Morgan Kaufmann Publishers Inc.

[Israeli, 1994] Israeli, A., Rappoport, L. Disjoint-access-parallel implementations of strong shared memory primitives. In Proceedings of the 13th ACM SIGACT-SIGOPS Symposium on Principles of Distributed Computing (Los Angeles, CA, August 1994), 151–160.

[McKenney, 2011] McKenney, P. E. 2011. Is parallel programming hard, and, if so, what can you do about it? kernel.org; <https://www.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html>

[Michael, 2004] Michael, M. M. Hazard pointers: safe memory reclamation for lock-free objects, IEEE Trans. Parallel Distrib. Syst. 15 (6) (2004) 491–504.

[Molka, 2015] Daniel Molka, Daniel Hackenberg, Robert Schöne, and Wolfgang E Nagel. 2015. Cache Coherence Protocol and Memory Performance of the Intel Haswell-EP Architecture. In Parallel Processing (ICPP), 2015 44th International Conference on. IEEE, 739–748.

[Sorin, 2011] D. J. Sorin, M. D. Hill, and D. A. Wood. A Primer on Memory Consistency and Cache Coherence. Morgan and Claypool, 2011.

[Wang, 2016] Q. Wang, T. Stamler, and G. Parmer, “Parallel sections: Scaling system-level data-structures,” in Proceedings of the ACM EuroSys Conference, 2016

## 附录 A - FreeBSD 序列化原语

### mutex

Mutex 是最直接的基元；所有权由线程对它进行比较并交换操作来获取，操作中带有指向获取线程结构的指针。如果锁已经包含线程指针，则它被拥有；如果不包含则空闲。有两类：MTX_DEF 和 MTX_SPIN。MTX_SPIN 互斥锁被认为是“重量级”的，因为它禁用中断。它可以在任何上下文中获取。持有它时（通常也禁用中断），线程只能获取其他 MTX_SPIN 锁，不能分配内存或睡眠。持有 MTX_DEF 锁时，线程可以做任何不会睡眠的事情（获取两种类型的互斥锁、读写锁、不可睡眠的内存分配等）。等待获取 MTX_SPIN 互斥锁时，线程会“自旋”，轮询锁是否被当前持有者释放。等待获取 MTX_DEF 时，线程会“自适应自旋”，如果当前持有者正在运行则轮询释放，如果当前持有者被抢占则排队等待 turnstile。Turnstiles 是优先级传播的设施，允许被阻塞的线程将其调度器优先级“借给”当前锁持有者，作为避免优先级反转的机制。换句话说，如果被阻塞的线程优先级更高，锁持有者的调度器优先级将提升到被阻塞线程的优先级。

### rwlock

`rwlock` 通过支持两种模式——单写多读——扩展了 MTX_DEF 互斥锁的语义。它的实现类似于互斥锁，但在锁字段的低位分配了一些额外状态。在单写模式下，它的行为与互斥锁相同。在读模式下，多个读取者可以获取锁，写者被阻塞直到所有读取者释放锁。在这种模式下，我们无法再有效地跟踪锁持有者的状态，因此无法传播优先级，获取者也无法知道所有持有者是否正在运行。因此，线程只能推测性地自旋。

### sx

`sx` 锁在逻辑上等同于 `rwlock`，关键区别在于锁持有者可以睡眠。被阻塞的读取者和写者在 sleepqueues 上维护，不进行优先级传播。

### rmlock

与 `rwlock` 和 `sx` 类似，`rmlock`（“读多锁”）是读写锁。它的关键区别在于读获取非常快，不涉及任何原子操作。写获取非常昂贵。在当前实现中，它涉及向所有其他 CPU 发送系统范围的 IPI。如果更新足够不频繁，这是一个合理的保证存在的基元。它易于推理，具有与熟悉的 `rwlock` 相同的语义。它支持任意数量的读取者，因此它是开发者在表查找期间保证字段存在性时传统上首选的基元。然而，每次读获取和释放都涉及锁的原子更新。当锁在核心复合体之间共享时（因此更新需要在 LLC 之间进行缓存一致性流量，将前一个持有者的缓存行从修改/独占转换为无效），它的使用很快变得非常昂贵。

### lockmgr

`lockmgr` 有点过时。它有一些 VFS（虚拟文件系统）层所需的独特功能。它在今天的代码中通常不是瓶颈，其特殊性超出了我希望涉及的范围。

### epoch

`epoch` 基元允许内核保证受其保护的结构在线程处于 epoch 段时保持活动。在 epoch 段中执行 `do_stuff()` 看起来像：

```c
epoch_enter(global_epoch);
do_stuff(); ...
epoch_exit(global_epoch);
```

删除在 epoch 段内引用的对象的线程可以通过调用 `epoch_wait(epoch)` 同步等待当前 epoch 加上宽限期内 epoch 段中的所有线程，也可以使用 `epoch_call(epoch, context, callback)` 将对象排队以在稍后时间释放，允许服务线程以比同步操作更低的成本确认宽限期已过。在许多方面，`epoch` 的读侧与 `rmlock` 的读侧具有类似的特征。然而，它不提供互斥保证。对受 `epoch` 保护的数据结构的修改可以与读取者并行进行。修改通常需要相互之间显式序列化。因此，使用互斥锁来保护一个写者免受其他写者的影响。尽管其实现和性能权衡与 Linux 的 RCU 完全不同，但它主要支持相同的编程设计模式。有两种 `epoch` 变体：可抢占和不可抢占。不可抢占的 `epoch` 更轻量，但不允许调用线程获取除 MTX_SPIN 互斥锁之外的任何锁类型。`epoch` 是 FreeBSD 12 中的新功能。它本质上是围绕 ConcurrencyKit 的 `epoch`（基于 Epoch 的回收）API 构建的内核内脚手架。

## 附录 B Will-It-Scale 结果

FreeBSD 11 与 FreeBSD 12 以及 FreeBSD 12 与 Ubuntu 18（Linux 4.15）的 Will-It-Scale 完整结果可在以下地址找到：

<https://github.com/ScaleBSD/scalebsd.github.io/tree/master/media/freebsd_processor_scaling>

MATT MACY 是一名咨询内核工程师和 FreeBSD 提交者。最近他专注于网络性能、内核可扩展性和 ZFS。
