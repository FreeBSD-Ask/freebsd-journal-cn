# TCP/IP 历险记：FreeBSD TCP 协议栈中的 Pacing

- 原文地址：[Adventures in TCP/IP: Pacing in the FreeBSD TCP Stack](https://freebsdfoundation.org/our-work/journal/browser-based-edition/kernel-development/adventures-in-tcp-ip-pacing-in-the-freebsd-tcp-stack/)
- 作者：Randall Stewart、Michael Tüxen

TCP 的发送和接收行为已经经历了 40 多年发展。在这期间，许多进展帮助 TCP 能够以非常高的速度传输可靠的数据流。然而，一些增强功能（无论是在协议栈中还是在网络中）也带来了负面影响。最初，TCP 协议栈会响应接收到的 TCP 段（ACK 应答）、上层提供的新数据或定时器到期时发送一个 TCP 段。TCP 发送方还实现了拥塞控制，以防止过快地向网络发送数据。这些特性结合在一起，可能会使得 TCP 端点大多数时间都受到“ACK 时钟”的影响，如下图所示。

![image](https://freebsdfoundation.org/wp-content/uploads/2024/11/stewart_chart1.png)

**图 1：ACK 时钟的示例**

为简化说明，假设数据包已编号并 ACK，并且接收方每 ACK 一个数据包后，跳过一个包（以减少开销）。图中底部的 ACK“1”ACK 了之前发送的 0 和 1 号包（图中未显示），移除了两个未确认的包，并允许发送两个新的包，即包 8 和 9。此时，由于拥塞控制，发送方被阻塞，等待 ACK“3”的到来，这会 ACK 包 2 和包 3，并再次发送下两个包。这种 ACK 时钟现象在早期互联网的许多数据流中普遍存在，今天在某些情况下仍能看到。ACK 时钟形成了一种自然的数据传输 Pacing，允许数据包通过瓶颈发送，而通常在发送方的下一个数据包到达瓶颈时，前一个数据包已经被传输了。

然而，随着时间的推移，网络和 TCP 的优化已改变了这种行为。一个例子可以在有线网络中看到。在这种网络中，下行带宽很大，但上行带宽较小（通常只有下行带宽的一部分）。

由于回传路径带宽的稀缺，有线调制解调器通常会仅保持最后一个已发送的 ACK（假设它看到的 ACK 是按顺序的），直到决定发送。因此，在上述示例中，有线调制解调器可能只会发送 ACK-5，而不是允许发送 ACK-1、ACK-3 和 ACK-5。这将导致发送方发送一个较大的数据包突发，而不是将三次两个数据包的突发分开。

另一个修改行为的例子可以在其他时隙技术中看到（这些技术会等到自己的时间片到达时才发送数据包，然后一次性发送所有排队的包），在这些技术中，ACK 会被排队，然后一次性以三个 ACK 包的突发形式发送。这种类型的技术与 TCP-LRO（在上期中提到）相互作用，从而可能会将 ACK 合并成一个单一的 ACK（如果使用的是旧的方式），或者在发送函数调用之前，将所有 ACK 排队一起处理。在任何情况下，又会发送一个大数据包突发，而不是一系列的小两包突发，这些小包之间有小的时间间隔（大致接近瓶颈带宽加上一些传播延迟）。

我们的示例显示了六个数据包，但实际上在一次`tcp_output()`调用中，可能会有数十个数据包突发。这对 CPU 优化有好处，但可能会导致网络中的数据包丢失，因为路由器的缓冲区有限，大量突发数据包更容易导致尾丢失。这种丢包会减少拥塞窗口，从而影响整体性能。

除了上述两个例子外，还有其他原因可能导致 TCP 变得突发（其中一些在 TCP 中始终存在），例如应用程序受限的时段。这种情况是指发送应用程序由于某种原因暂停，延迟几毫秒才发送数据。在此期间，ACK 会到达，但没有数据需要发送。在网络中的所有数据尚未传输完之前（这可能由于空闲而导致拥塞控制减少），应用程序会发送另一个较大的数据块。此时，拥塞窗口是打开的，因此可以生成较大的发送突发。

另一个突发性源可能来自对等 TCP 实现，它们可能决定每 ACK 第八个或第十六个包，而不是遵循 TCP 标准每 ACK 一个包。这种较大范围的 ACK 会再次导致对应的突发。TCP Pacing 控制（在下一节中描述）是一种改进 TCP 段发送的方式，用以平滑这些突发。

## TCP Pacing

以下示例说明了 TCP Pacing 控制。假设使用 TCP 连接以 12 兆比特每秒的速度向对等方传输数据，包括 IP 和 TCP 头。假设最大 IP 数据包大小为 1500 字节（即 12000 位），这将导致每秒发送 1000 个 TCP 段。在使用 Pacing 控制时，将每毫秒发送一个 TCP 段。可以使用一个每毫秒触发的定时器来实现这一点。以下图形展示了这种发送行为。

![](https://freebsdfoundation.org/wp-content/uploads/2024/11/stewart_chart2.png)

**图 2：以 12Mbps 速率传输的 TCP 连接 Pacing 控制**

以这种方式进行 Pacing 控制是可行的，但由于需要较高的 CPU 成本，这并不理想。因此，为了提高效率，TCP 在进行 Pacing 控制时，通常会发送小的突发数据包，并且这些数据包之间会有一定的时间间隔。突发包的大小通常与协议栈希望的 Pacing 速度相关。如果以较高的速率进行 Pacing 控制，则需要较大的突发数据包；如果以较低的速率进行 Pacing 控制，则使用较小的突发数据包。

在设计 TCP 协议栈的 Pacing 控制方法时，可以采取多种方法。常见的做法是让 TCP 协议栈在较低层设置一个速率，然后将大批量数据的发送交给该层处理。较低层再通过合适的定时器来多路复用来自不同 TCP 连接的数据包，从而实现数据的间隔发送。然而，这不是 FreeBSD 协议栈采用的方法，特别是因为这种方法限制了 TCP 协议栈对发送过程的控制。如果连接需要发送重传数据包，重传数据包将排在所有待发送的队列包之后。

在 FreeBSD 中，采用了不同的方法，让 TCP 协议栈控制发送，并创建了一个专门的定时系统，当 Pacing 间隔结束时，该系统会调用 TCP 协议栈发送数据。这种方法将发送控制完全交给 TCP 协议栈，但可能会带来一些性能问题，需要进行相应的补偿。下一节将介绍为 FreeBSD 创建的这一新子系统。

## 高精度定时系统

### 概念概述

高精度定时系统（HPTS）是一个可加载的内核模块，提供了一个简单的接口，供任何希望使用它的 TCP 协议栈调用。基本上，TCP 协议栈会调用两个主要函数来使用 HPTS 提供的服务：

- `tcp_hpts_insert()`：将 TCP 连接插入到 HPTS 中，在指定的时间间隔内调用`tcp_output()`，或者在某些情况下，调用 TCP 的入站数据包处理函数`tfb_do_queued_segments()`。
- `tcp_hpts_remove()`：要求 HPTS 从 HPTS 中移除一个连接。通常在连接关闭或不再需要发送数据时使用。

HPTS 中还提供了一些其他辅助功能，帮助进行定时和其他管理任务，但上述两个函数是 TCP 协议栈用于实现 Pacing 控制的基本构建块。

### 详细信息

在内部，每个 CPU 都有一个 HPTS 轮盘，这是一个连接列表的数组，表示在不同时间点需要服务的连接。轮盘中的每个槽位代表 10 微秒。当一个 TCP 连接被插入时，它会被分配一个从现在开始的槽位数量（即 10 微秒的时间间隔），直到调用`tcp_output()`函数。这个轮盘由系统定时器（即 FreeBSD 的调用系统）和一个软定时器（如[1]中提议的）共同管理。基本上，每次系统调用返回时，在返回用户空间之前，HPTS 可能会被调用，以查看是否需要服务某个 HPTS 轮盘。

HPTS 系统还会自动调优 FreeBSD 的系统定时器，首先设置最小值（默认为 250 微秒）和最大值，如果 HPTS 轮盘中有更多连接且调用频繁，则 FreeBSD 系统超时期间的小量处理将增加系统定时器的长度。如果连接数低于某个阈值，则仅使用基于系统定时器的方法。这有助于避免通过在轮盘上保持连接时间过长而导致连接饿死。HPTS 尝试提供定时器最小精度（即 250 微秒），但这并不保证。

使用 HPTS 进行 Pacing 控制的 TCP 协议栈有一些特定的责任，以便与 HPTS 协作，实现所需的 Pacing 速率，包括：

- 待启动了 Pacing 定时器，协议栈必须禁止任何发送或其他对`tcp_output()`的调用，直到 Pacing 定时器到期。协议栈可以查看`t_flags2`字段中的`TF2_HPTS_CALLS`标志。当 HPTS 调用`tcp_output()`函数时，该标志将被设置，协议栈应在其`tcp_output()`函数内进行清除。
- 在 Pacing 定时器到期后，HPTS 的调用中，协议栈需要验证它的空闲时间。HPTS 可能会比预期稍晚调用协议栈，甚至可能提前调用协议栈（尽管这种情况非常罕见）。协议栈需要将晚到或早到的时间纳入下一个 Pacing 超时计算中，之后再发送数据。
- 如果协议栈决定使用 FreeBSD 定时器系统，它还必须阻止定时器调用发送数据。RACK 和 BBR 协议栈不会使用 FreeBSD 定时器系统进行超时，而是直接使用 HPTS。
- 如果协议栈从 LRO 队列中排队数据包，则 HPTS 可能会调用输入函数，而不是`tcp_output()`。如果发生这种情况，则不会再调用`tcp_output()`，因为假设协议栈会在需要时调用其输出函数。

HPTS 还提供了一些工具来协助 TCP 协议栈，包括：

- `tcp_in_hpts()`：告诉协议栈是否已在 HPTS 系统中。
- `tcp_set_hpts()`：设置连接将使用的 CPU，这是一个可选调用，如果协议栈未调用此函数，HPTS 会为该连接执行此操作。
- `tcp_tv_to_hptstick()`：将`struct timeval`转换为 HPTS 槽位数。
- `tcp_tv_to_usectick()`：将`struct timeval`转换为 32 位无符号整数。
- `tcp_tv_to_lusectick()`：将`struct timeval`转换为 64 位无符号整数。
- `tcp_tv_to_msectick()`：将`struct timeval`转换为 32 位无符号毫秒定时器。
- `get_hpts_min_sleep_time()`：返回 HPTS 强制执行的最小睡眠时间。
- `tcp_gethptstick()`：可选地填充一个`struct timeval`并返回当前的单体时间，作为 32 位无符号整数。
- `tcp_get_u64_usecs()`：可选地填充一个`struct timeval`并返回当前的单体时间，作为 64 位无符号整数。

### `sysctl`变量

HPTS 系统可以通过`sysctl`变量进行配置，以改变其性能特征。默认情况下，这些值设置为一组“合理”的值，但根据应用的不同，可能需要进行更改。值可以在`net.inet.tcp.hpts`系统控制节点下进行设置。

以下是可用的调优参数：

| 名称                | 默认值 | 描述                                                                                                                                                                                                                                                                                                                                                                                |
| ------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| no_wake_over_thresh | 1      | 当将连接插入 HPTS 时，如果此布尔值为真且连接数大于`cnt_thresh`，则不允许调度 HPTS 运行。如果值为 0（假），则在将连接插入 HPTS 时，可能会导致 HPTS 系统运行连接，即调用`tcp_output()`以调度的连接。                                                                                                                                                                                  |
| less_sleep          | 1000   | 当 HPTS 完成运行时，它会知道它运行了多少个槽。如果运行的槽数超过此值，则需要减少动态定时器。                                                                                                                                                                                                                                                                                        |
| more_sleep          | 100    | 当 HPTS 完成运行时，如果运行的槽数少于此值，则动态睡眠时间将增加。                                                                                                                                                                                                                                                                                                                  |
| min_sleep           | 250    | 这是 HPTS 定时器可以降低到的绝对最小值。减小此值会导致 HPTS 运行更多，使用更多的 CPU。增大它会导致 HPTS 运行更少，使用更少的 CPU，但会负面影响精度。                                                                                                                                                                                                                                |
| max_sleep           | 51200  | 这是定时器可以达到的最大睡眠值（以 HPTS 槽位为单位）。通常仅在没有连接被服务时使用，即 HPTS 每 51200 x 10 微秒（大约半秒）唤醒一次。                                                                                                                                                                                                                                                |
| loop_max            | 10     | 此值表示 HPTS 在尝试为所有需要服务的连接提供服务时将循环的次数。当 HPTS 启动时，它会将需要服务的连接列表集合在一起，然后开始对每个连接调用`tcp_output()`。如果花费的时间太长，可能还需要服务更多的连接，因此它将再次循环以提供服务。此值表示 HPTS 在被强制休眠之前最多可以执行多少次此循环。注意，从函数调用返回时不会导致任何循环发生；只有 FreeBSD 定时器调用会受到此参数的影响。 |
| dyn_maxsleep        | 5000   | 当调整调用时间并看到更多睡眠需求时，动态定时器可以提高的最大值。                                                                                                                                                                                                                                                                                                                    |
| dyn_minsleep        | 250    | 当调整调用时间并看到较少睡眠时，动态定时器可以降低的最小值。                                                                                                                                                                                                                                                                                                                        |
| cnt_thresh          | 100    | 这是轮盘上需要的连接数，用于开始更多依赖系统调用返回的情况。超过此阈值时，系统调用返回和超时都会导致 HPTS 运行；低于此阈值时，我们更依赖调用系统来运行 HPTS。                                                                                                                                                                                                                       |

## 在 RACK 堆栈中的流量控制优化

使用 HPTS 系统进行流量控制时，与运行在 TCP 堆栈之下的流量控制系统相比，可能会有一些性能损失。这是因为每次调用`tcp_output()`时，会做出很多关于发送内容的决策。这些决策通常会引用多个缓存行，并涉及大量代码。例如，默认的 TCP 堆栈在`tcp_output()`路径中有超过 1500 行代码，其中不包括任何关于流量控制或突发流量缓解的代码。对于没有流量控制的默认堆栈，通过这么多行代码和大量缓存未命中的影响，仍然能够通过一次发送多个数据段来进行补偿。然而，当实现一个低层流量控制系统时，它可以通过追踪接下来需要发送的数据以及发送的数量，轻松地优化它必须发送的各种数据包的发送过程。这使得低层流量控制系统减少了许多缓存未命中。

为了在像 HPTS 这样的高级系统中获得类似的性能，TCP 堆栈必须找到优化发送路径（包括传输和重传）的方式。堆栈可以通过创建“快速路径”发送通道来实现。RACK 堆栈已经实现了这些快速路径，以使流量控制的成本降低。BBR 堆栈当前也进行流量控制，符合已实现的 BBRv1 规范，但它尚未实现以下所述的快速路径。

### 快速路径传输

当 RACK 首次进行流量控制时，发送调用会通过它的`tcp_output()`路径，并计算出可以发送的字节数。然后将其降低到符合已建立的流量微突发大小，但在这一过程中，会设置一个“快速发送块”，记录剩余要发送的字节数以及这些数据在套接字发送缓冲区中的位置。同时还设置一个标志，以便 RACK 在下次知道快速路径处于活动状态。请注意，如果发生超时，快速路径标志会被清除，以便做出正确的决策，决定哪个重传需要发送。

在 RACK 的`tcp_output()`例程入口处，在验证可以发送数据后，会检查快速路径标志。如果标志被设置，则会使用之前保存的信息发送新数据，而不进行通常的输出路径检查。这大大降低了流量控制的成本，因为大部分代码和缓存未命中都从这个快速输出路径中消除了。

### 快速路径重传

RACK 中的重传也有一个快速路径。这得益于 RACK 的发送映射（sendmap），它跟踪所有已发送的数据。当某个数据块需要重传时，发送映射条目会精确告诉快速路径需要发送的数据的位置和大小。这跳过了典型的套接字缓冲区查找和其他开销，即使在发送重传时也能提供一定的效率。

## 结论

HPTS 为 TCP 堆栈提供了一种新颖的服务，使其能够实现流量控制。为了实现与竞争设计方法相当的效率，TCP 堆栈和 HPTS 需要合作，减少开销并实现高效的包突发发送。本文讨论了流量控制的必要性以及 FreeBSD 中为此提供的基础设施。未来的文章将探讨 TCP 堆栈进行流量控制时的另一个关键问题，即选择合适的流量控制速率。

## 参考文献

1. Mohit Aron, Peter Druschel: *Soft Timers: Efficient Microsecond Software Timer Support for Network Processing*. In: ACM Transactions on Computer Systems, Vol. 18, No. 3, August 2000, pp 197-228. [https://dl.acm.org/doi/pdf/10.1145/319344.319167](https://dl.acm.org/doi/pdf/10.1145/319344.319167).

---

**RANDALL STEWART** ([rrs@freebsd.org](mailto:rrs@freebsd.org)) 已从事操作系统开发四十余年年，自 2006 年以来一直是 FreeBSD 的开发者。他专注于传输协议，包括 TCP 和 SCTP，但也曾涉及操作系统的其他领域。目前他是独立顾问。

**MICHAEL TÜXEN** ([tuexen@freebsd.org](mailto:tuexen@freebsd.org)) 是明斯特应用科技大学的教授，Netflix 的兼职承包商，自 2009 年以来是 FreeBSD 源代码的提交者。他专注于传输协议，如 SCTP 和 TCP，及其在 IETF 的标准化以及在 FreeBSD 中的实现。
