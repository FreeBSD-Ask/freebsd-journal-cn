# FreeBSD 与 RTEMS：实时操作系统中的 UNIX

- 原文：[FreeBSD and RTEMS, UNIX in a Real-Time Operating System](https://freebsdfoundation.org/wp-content/uploads/2016/08/FreeBSD-and-RTEMS-Unix-in-a-Real-Time-Operating-System.pdf)
- 作者：**Chris Johns**、**Joel Sherrill**、**Ben Gras**、**Sebastian Huber**、**Gedare Bloom**

RTEMS 开发者有在 RTEMS 源码库中使用 FreeBSD 代码的悠久历史，成果斐然。RTEMS 并非唯一这么做的项目，RTEMS 在使用 FreeBSD 源码时遇到的挑战和问题，也是其他项目同样会遇到的。RTEMS 是一种 POSIX 实时操作系统（RTOS），与所有开源项目一样资源有限。所有可用的精力都必须聚焦在操作系统的实时部分。RTEMS 尽可能采用质量上乘、许可证合适的源码——结合来自 Newlib、RTEMS 和 FreeBSD TCP/IP 协议栈的代码——提供令人惊喜的、相当稳健的 POSIX 子集。下文回顾历史、总结过去 20 年取得的成果，并说明如何在最小化维护负担的前提下解决集成问题。

## RTEMS

RTEMS 全称是 Real-Time Executive for Multiprocessor Systems（多处理器系统实时执行体），自 1991 年起成为开源软件。RTEMS 在过去 25 年中用于一些重要的应用，项目社区健康活跃。它由位于阿拉巴马州亨茨维尔的 On-Line Applications Research（OAR）开发，合同方为美国陆军。起因是项目经理看着导弹飞向试验场后爆炸——每枚导弹里都捆绑着一份操作系统许可证。RTEMS 诞生之初是 20 世纪 80 年代末到 90 年代初典型的传统嵌入式 RTOS 内核。用户 API 基于 VMEBus Industry Trade Association（VITA）Real-Time Executive Interface Definition（RTEID）2.1 标准。内核的 C 源代码交叉编译为库，应用的 object 文件与 RTEMS 库链接，生成单一独立的静态可执行文件。嵌入通常涉及用某种外部编程工具把二进制映像烧录进 EPROM。应用与内核共享同一地址空间，没有内存管理或虚拟内存。RTEMS 应用静态链接到操作系统，在同一单一地址空间中运行。单地址空间环境就像一个没有保护的单一进程，虚拟地址到物理地址空间是 1:1 映射。RTEMS 的早期版本没有 C 库或文件系统，没有网络功能，设备驱动和硬件支持往往是针对特定硬件定制的，被视为应用的一部分。RTEMS 支持多处理器有 20 年。多处理器支持基于在互连总线架构和消息传递上运行、各自独立地址空间的多个处理器。应用可以访问和管理分布在多个节点上的资源。RTEMS 一路演进成长，如今相当复杂精细。但嵌入式实时系统的某些基本特性至今保留且仍然有效。

如今 RTEMS 支持对称多处理（SMP）。在某些方面，在操作系统里实现 SMP 是较容易的部分，因为设计确定性的实时 SMP 应用更具挑战。操作系统中的问题只需解决一次，但每个应用都要面对竞态条件 bug、资源争用、有效利用多核同时还要保证可预测、正确且安全的行为等类似挑战。

## 实时

如今，实时软件和实时操作系统与以往一样重要。面向它们的应用开发依然活跃，支撑这些应用的操作系统开发也同样如此。越来越小的设备中算力不断增强，把 FreeBSD 这样强大的服务器级操作系统带到了小型嵌入式设备上，这些设备的性能足以满足一系列实时应用。然而，这些系统仍然不是完全确定性的，完整的可调度性分析仍然困难。RTEMS 的性能可以是确定性的，并提供了一整套丰富的调度算法，满足实时应用的需求。RTEMS 通过为多核处理器加入 SMP 来扩展对高性能实时计算的支持。SMP 硬件上的实时软件是一项艰巨而复杂的挑战。为协助应用架构师，RTEMS 提供了一组强大的配置接口。多个核心可以分组并赋予特定的调度算法，线程可以分配到核心亲和集合，从而与特定的调度算法关联。把实时设计的组件分区，让架构师能管理可调度性分析这项艰巨任务。RTEMS 允许把中断分配到特定核心，让用户在负载下管理延迟。

## 内核架构与 API

RTEMS 内核源码位于 `git://git.rtems.org/rtems.git` 仓库。它有四大组件：

1. SuperCore
2. 应用编程接口（API）
3. 服务（Services）
4. 板级支持包（BSP）

### SuperCore

SuperCore 是通过各类应用编程接口（API）向用户导出的功能超集。SuperCore 包含 RTEMS 中所有重要的实时算法，包括调度、加锁、同步、时钟、中断和 SMP 支持，还包含用于上下文切换和每种支持架构的低级中断管理的架构与 CPU 支持。各类用户接口创建和管理的资源能共存，因为一切都映射到 SuperCore 资源。这意味着在一个用户接口中创建的线程可以阻塞另一个用户接口创建的互斥锁。这是一项强大的特性，软件组件可以由不同的用户接口实现，并组合进同一个可执行文件。这种支持延伸到调度和时间域，意味着应用的运行时画像与编写它的 API 无关。

### 应用编程接口

RTEMS 目前支持三大应用编程接口（API），分别是：

1. RTEMS Classic API
2. POSIX
3. High Performance API

### RTEMS Classic API

Classic API 是 RTEMS 最初的 API，基于 VITA RTEID 2.1 标准。它是一种经典的实时操作系统编程接口，提供：

1. 任务（Tasks）
2. 信号量（Semaphores）
3. 消息队列（Message Queues）
4. 事件（Events）
5. 屏障（Barriers）
6. 中断（Interrupts）
7. 时间（Time）
8. 定时器（Timers）
9. 速率单调周期（Rate Monotonic Periods）
10. 固定分配内存池（Fixed Allocation Memory Pools）
11. 可变分配内存池（Variable Allocation Memory Pools）

RTEMS Classic API 开销低，常用于资源受限、需要小体积的目标。

### POSIX

RTEMS 的 POSIX 支持按实现位置分为三部分，可由以下实现：

1. C 库，
2. RTEMS，或
3. TCP/IP 协议栈。

RTEMS 使用与 Cygwin 相同的 Newlib C 库，提供核心 C 库和大部分与线程无关的 POSIX 能力。这为包括数学、stdio、字符串乃至宽字符支持在内的核心 POSIX 和 C 库服务提供了稳健的实现。Newlib 的大部分源码起源于 BSD 操作系统，但已移植到许多目标架构。重要的是，RTEMS 依赖由 C 和 POSIX 标准定义的 Newlib 头文件。历史上，Newlib 没有提供完整的 C 和 POSIX 头文件集合，只提供它实现了某些方法的那些头文件。不过，RTEMS 开发者最近一直在推动扩充这套头文件集合，并确保它们与 FreeBSD 内核和用户空间源码兼容。RTEMS 提供所有并发与同步能力的实现，包括线程、互斥锁、条件变量、信号量和消息队列，还实现了 `open(2)`、`read(2)` 等系统调用。此外，RTEMS 还提供 termios、时钟和定时器等 POSIX 能力。RTEMS 依赖 FreeBSD TCP/IP 协议栈提供 POSIX 标准要求的网络 API。仅支持 IPV4 的旧协议栈不能提供 POSIX 要求的全部能力，而 LibBSD 项目提供的新 TCP/IP 协议栈值得称道，提供了完整支持。作为单进程、多线程的操作系统，RTEMS 对应 POSIX 的 PSE 51 和 52 档案。这些档案定义了单进程、多线程 POSIX 实现所提供的服务，PSE 52 包含文件系统支持，而 PSE 51 不包含。RTEMS 应用可以选择禁用所有文件系统支持，因此 RTEMS 可以由用户配置为对应其中任一档案。FreeBSD 是多进程、多用户的 POSIX 实现，对应 POSIX 档案 PSE 54。Open Group 的未来机载能力环境（FACE™）联盟定义了四个新的 POSIX 档案，以满足航空电子软件社区的需求。这些档案反映了已通过航空电子、医疗设备等行业认证的现有实时操作系统和应用，把 POSIX 标准中约 1,300 个方法缩减到符合现有航空电子应用的认证和应用要求：

- Security Profile 较小，仅要求 163 个方法。面向多线程、单进程应用，如信息网关设备。
- Safety Base Profile 较大，要求 246 个方法。面向多线程、单进程应用。
- Safety Extended Profile 更大，要求 335 个方法。面向多线程、多进程应用。
- General Purpose Profile 最大，要求 812 个方法。面向可能没有任何认证要求或要求较宽松的多线程、多进程应用。

作为长期支持标准的单进程、多线程操作系统，RTEMS 自然对应 Safety Base Profile。初次评估时，RTEMS 只比该档案少了不到 10 个方法。目前正努力集成 RTEMS 与 ARINC 653 Deos RTOS，并使 Safety Base Profile 达到 FACE 合规。有趣的是，对照单进程 FACE General Purpose 档案评估时，RTEMS 表现惊人，支持约 90% 的要求方法。对 `fork(2)/exec(3)` 和进程组等所需能力的正确支持超出了 RTEMS 的目标档案。不过，这些缺失的方法中，大多数是不需要多进程的方法。Newlib 不支持 `fenv.h` 或 long double complex 数学，这占据了 RTEMS 本可支持但缺失方法的大部分。

### High Performance API

紧耦合的 High Performance API 是新引入的，不视为应用使用的通用 API。它用于可以用速度换取兼容性的场合。FreeBSD 移植就是一个例子，C11/C++11 线程和 OpenMP 的后端也是。C11/C++11 线程目前在 RTEMS 中具有最低的空间和时间开销。

### 服务

服务提供帮助开发者构建有用应用的功能。支持范围从实现特定协议（如 SNMP）、其他语言（如 Lua 或 Python），到 NTP 等重要服务。

### 板级支持包

板级支持包（BSP）实现是在特定硬件上落地 RTEMS 所需的代码和支持。RTEMS 在 17 种架构上包含 170 多个 BSP。BSP 负责从引导加载程序进入、设置内存、缓存、控制台，并包含用于系统 tick 的定时器驱动。如果处理器有多个核心，BSP 还管理额外核心的启动。大多数 FreeBSD 用户永远不必深究自己硬件的设备驱动细节，但 RTEMS 用户常常使用定制硬件，因此需要为自己的板卡开发驱动。

## FreeBSD 在 RTEMS 中

FreeBSD 是 RTEMS 及其历史的重要组成部分。RTEMS 中的 BSD 代码最早可追溯到加入 Newlib C 库的代码和文件。此后 RTEMS 在多个领域直接引入 FreeBSD 源码。除 C 库外，RTEMS 的 shell 命令（如 `rm`、`cp` 乃至 `dd`）也是这套代码日益增长的消费者之一。

### 网络协议栈

20 世纪 90 年代末，在萨斯卡通的加拿大光源工作的 Eric Norum 开始为 RTEMS 寻找网络协议栈。Eric 是 EPICS 项目成员，该项目需要网络协议栈。他最初的工作包括移植名为 KA9Q 的协议栈，但无论从性能还是许可证角度看都不太成功。在 RTEMS 的那段历史中，应用总是与操作系统静态链接，嵌入式实时应用出厂时与硬件设备集成，通常无意为应用或任何支撑软件重新分发源代码。这些部署特征让 RTEMS 项目始终谨慎评估直接纳入或作为第三方附件提供的软件的许可证。对于 KA9Q，除性能问题外，其协议栈许可证究竟是什么始终不明确，导致 KA9Q 工作被放弃。Eric Norum 开始考虑移植 Linux 网络协议栈，但因许可证原因未能持续，于是有人建议他看看 FreeBSD 协议栈。Eric 用六个月时间把 FreeBSD 网络协议栈移植到 RTEMS。该协议栈至今仍在 RTEMS 中，基本未变，仅在必要时做了特定 bug 修复。该网络协议栈非常成功，让 RTEMS 拥有标准 FreeBSD 安装所具备的全部网络功能。FreeBSD 的协议栈功能完备，移植到 RTOS 后，用户可以创建具有系统级稳健性的应用。RTEMS 中几乎没有无法实现的系统配置——从单一扁平网络端点，到路由、DHCP、无线电 VoIP，再到 SDH 管理通道上的 IP over GRE 隧道。FreeBSD 网络代码的这次最初移植——如今称为传统移植——采用三任务模型。一个网络任务或线程在网络协议栈内运行，让协议栈处理 ICMP 数据包和 TCP 重传等事务。每个接口有一个接收任务和一个发送任务。接收任务从 MAC 接收数据并交付协议栈，发送任务清空输出队列，把 mbuf 发给 MAC。所有网络任务优先级相同，运行时需持有单一网络信号量。这种设计意味着网络并发有限，多接口吞吐量受限，因为信号量把所有处理串行化。不过，该实现在可重入线程系统中安全可靠，实践中工作良好。

### 维护

这次移植完成后的若干年里，开始暴露一些问题。第一个也是最明显的问题是驱动可用性。定制驱动支持要求为 RTEMS 编写定制驱动。代码可以从 FreeBSD 借鉴，但驱动往往需要像新写一样编写和测试。随着时间推移和 FreeBSD 演进，让 RTEMS 用户感到沮丧和困惑的是：FreeBSD 协议栈的移植却不支持较新 FreeBSD 版本的驱动。当初把代码移植到 RTEMS 时，它被复制到简化的目录结构中。FreeBSD 的目录树又大又宽，似乎没有必要再创建一个宽泛的

> **注**：英文原版 PDF 文本提取在此处截断，后续内容请参阅原 PDF：<https://freebsdfoundation.org/wp-content/uploads/2016/08/FreeBSD-and-RTEMS-Unix-in-a-Real-Time-Operating-System.pdf>。
