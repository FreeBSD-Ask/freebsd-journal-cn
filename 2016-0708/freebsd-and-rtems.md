# FreeBSD 与 RTEMS：实时操作系统中的 UNIX

- 原文：[FreeBSD and RTEMS, UNIX in a Real-Time Operating System](https://freebsdfoundation.org/wp-content/uploads/2016/08/FreeBSD-and-RTEMS-Unix-in-a-Real-Time-Operating-System.pdf)
- 作者：**Chris Johns**、**Joel Sherrill**、**Ben Gras**、**Sebastian Huber**、**Gedare Bloom**

RTEMS 开发者有在 RTEMS 源码库中使用 FreeBSD 代码的悠久历史，成果斐然。RTEMS 并非唯一这么做的项目，RTEMS 在使用 FreeBSD 源码时遇到的挑战和问题，也是其他项目同样会遇到的。RTEMS 是一种 POSIX 实时操作系统（RTOS），与所有开源项目一样资源有限。所有可用的精力都必须聚焦在操作系统的实时部分。RTEMS 尽可能采用质量上乘、许可证合适的源码——结合来自 Newlib、RTEMS 和 FreeBSD TCP/IP 协议栈的代码——提供令人惊喜的、相当稳健的 POSIX 子集。下文回顾历史、总结过去 20 年取得的成果，并说明如何在最小化维护负担的前提下解决集成问题。

## RTEMS

RTEMS 全称是 Real-Time Executive for Multiprocessor Systems（多处理器系统实时执行体），自 1991 年起成为开源软件。RTEMS 在过去 25 年中用于一些重要的应用，项目社区健康活跃。它由位于阿拉巴马州亨茨维尔的 On-Line Applications Research（OAR）开发，合同方为美国陆军。起因是项目经理看着导弹飞向试验场后爆炸——每枚导弹里都捆绑着一份操作系统许可证。RTEMS 诞生之初是 20 世纪 80 年代末到 90 年代初典型的传统嵌入式 RTOS 内核。用户 API 基于 VMEBus Industry Trade Association（VITA）Real-Time Executive Interface Definition（RTEID）2.1 标准。内核的 C 源代码交叉编译为库，应用的目标文件与 RTEMS 库链接，生成单一独立的静态可执行文件。嵌入通常涉及用某种外部编程工具把二进制映像烧录进 EPROM。应用与内核共享同一地址空间，没有内存管理或虚拟内存。RTEMS 应用静态链接到操作系统，在同一单一地址空间中运行。单地址空间环境就像一个没有保护的单一进程，虚拟地址到物理地址空间是 1:1 映射。RTEMS 的早期版本没有 C 库或文件系统，没有网络功能，设备驱动和硬件支持往往是针对特定硬件定制的，被视为应用的一部分。RTEMS 已支持多处理器 20 年。多处理器支持基于在互连总线架构和消息传递上运行、各自独立地址空间的多个处理器。应用可以访问和管理分布在多个节点上的资源。RTEMS 一路演进成长，如今相当复杂精细。但嵌入式实时系统的某些基本特性至今保留且仍然有效。

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

SuperCore 是通过各类应用编程接口（API）向用户导出的功能超集。SuperCore 包含 RTEMS 中所有重要的实时算法，包括调度、加锁、同步、时钟、中断和 SMP 支持，还包含用于上下文切换和每种支持架构的低级中断管理的架构与 CPU 支持。各类用户接口创建和管理的资源能共存，因为一切都映射到 SuperCore 资源。这意味着在一个用户接口中创建的线程可以阻塞在另一个用户接口创建的互斥锁上。这是一项强大的特性，软件组件可以由不同的用户接口实现，并组合进同一个可执行文件。这种支持延伸到调度和时间域，意味着应用的运行时画像与编写它的 API 无关。

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

RTEMS 使用与 Cygwin 相同的 Newlib C 库，提供核心 C 库和大部分与线程无关的 POSIX 能力。这为包括数学、stdio、字符串乃至宽字符支持在内的核心 POSIX 和 C 库服务提供了稳健的实现。Newlib 的大部分源码起源于 BSD 操作系统，但已移植到许多目标架构。重要的是，RTEMS 依赖由 C 和 POSIX 标准定义的 Newlib 头文件。历史上，Newlib 没有提供完整的 C 和 POSIX 头文件集合，只提供它实现了某些方法的那些头文件。不过，RTEMS 开发者最近一直在推动扩充这套头文件集合，并确保它们与 FreeBSD 内核和用户空间源码兼容。RTEMS 提供所有并发与同步能力的实现，包括线程、互斥锁、条件变量、信号量和消息队列，还实现了 **open(2)**、**read(2)** 等系统调用。此外，RTEMS 还提供 termios、时钟和定时器等 POSIX 能力。RTEMS 依赖 FreeBSD TCP/IP 协议栈提供 POSIX 标准要求的网络 API。仅支持 IPV4 的旧协议栈不能提供 POSIX 要求的全部能力，而 LibBSD 项目提供的新 TCP/IP 协议栈值得称道，提供了完整支持。作为单进程、多线程的操作系统，RTEMS 对应 POSIX 的 PSE 51 和 52 档案。这些档案定义了单进程、多线程 POSIX 实现所提供的服务，PSE 52 包含文件系统支持，而 PSE 51 不包含。RTEMS 应用可以选择禁用所有文件系统支持，因此 RTEMS 可以由用户配置为对应其中任一档案。FreeBSD 是多进程、多用户的 POSIX 实现，对应 POSIX 档案 PSE 54。Open Group 的未来机载能力环境（FACE™）联盟定义了四个新的 POSIX 档案，以满足航空电子软件社区的需求。这些档案反映了已通过航空电子、医疗设备等行业认证的现有实时操作系统和应用，把 POSIX 标准中约 1,300 个方法缩减到符合现有航空电子应用的认证和应用要求：

- Security Profile 较小，仅要求 163 个方法。面向多线程、单进程应用，如信息网关设备。
- Safety Base Profile 较大，要求 246 个方法。面向多线程、单进程应用。
- Safety Extended Profile 更大，要求 335 个方法。面向多线程、多进程应用。
- General Purpose Profile 最大，要求 812 个方法。面向可能没有任何认证要求或要求较宽松的多线程、多进程应用。

作为长期支持标准的单进程、多线程操作系统，RTEMS 自然对应 Safety Base Profile。初次评估时，RTEMS 只比该档案少了不到 10 个方法。目前正努力集成 RTEMS 与 ARINC 653 Deos RTOS，并使 Safety Base Profile 达到 FACE 合规。有趣的是，对照单进程 FACE General Purpose 档案评估时，RTEMS 表现惊人，支持约 90% 的要求方法。对 **fork(2)/exec(3)** 和进程组等所需能力的正确支持超出了 RTEMS 的目标档案。不过，这些缺失的方法中，大多数是不需要多进程的方法。Newlib 不支持 `fenv.h` 或 long double complex 数学，这占据了 RTEMS 本可支持但缺失方法的大部分。

### High Performance API

紧耦合的 High Performance API 是新引入的，不视为应用使用的通用 API。它用于可以用速度换取兼容性的场合。FreeBSD 移植就是一个例子，C11/C++11 线程和 OpenMP 的后端也是。C11/C++11 线程目前在 RTEMS 中具有最低的空间和时间开销。

### 服务

服务提供帮助开发者构建有用应用的功能。支持范围从实现特定协议（如 SNMP）、其他语言（如 Lua 或 Python），到 NTP 等重要服务。

### 板级支持包

板级支持包（BSP）实现是在特定硬件上落地 RTEMS 所需的代码和支持。RTEMS 在 17 种架构上包含 170 多个 BSP。BSP 负责从引导加载程序进入、设置内存、缓存、控制台，并包含用于系统 tick 的定时器驱动。如果处理器有多个核心，BSP 还管理额外核心的启动。大多数 FreeBSD 用户永远不必深究自己硬件的设备驱动细节，但 RTEMS 用户常常使用定制硬件，因此需要为自己的板卡开发驱动。

## FreeBSD 在 RTEMS 中

FreeBSD 是 RTEMS 及其历史的重要组成部分。RTEMS 中的 BSD 代码最早可追溯到加入 Newlib C 库的代码和文件。此后 RTEMS 在多个领域直接引入 FreeBSD 源码。除 C 库外，RTEMS 的 shell 命令（如 rm、cp 乃至 dd）也是这套代码日益增长的消费者之一。

### 网络协议栈

20 世纪 90 年代末，在萨斯卡通的加拿大光源工作的 Eric Norum 开始为 RTEMS 寻找网络协议栈。Eric 是 EPICS 项目成员，该项目需要网络协议栈。他最初的工作包括移植名为 KA9Q 的协议栈，但无论从性能还是许可证角度看都不太成功。在 RTEMS 的那段历史中，应用总是与操作系统静态链接，嵌入式实时应用出厂时与硬件设备集成，通常无意为应用或任何支撑软件重新分发源代码。这些部署特征让 RTEMS 项目始终谨慎评估直接纳入或作为第三方附件提供的软件的许可证。对于 KA9Q，除性能问题外，其协议栈许可证究竟是什么始终不明确，导致 KA9Q 工作被放弃。Eric Norum 开始考虑移植 Linux 网络协议栈，但因许可证原因未能持续，于是有人建议他看看 FreeBSD 协议栈。Eric 用六个月时间把 FreeBSD 网络协议栈移植到 RTEMS。该协议栈至今仍在 RTEMS 中，基本未变，仅在必要时做了特定 bug 修复。该网络协议栈非常成功，让 RTEMS 拥有标准 FreeBSD 安装所具备的全部网络功能。FreeBSD 的协议栈功能完备，移植到 RTOS 后，用户可以创建具有系统级稳健性的应用。RTEMS 中几乎没有无法实现的系统配置——从单一扁平网络端点，到路由、DHCP、无线电 VoIP，再到 SDH 管理通道上的 IP over GRE 隧道。FreeBSD 网络代码的这次最初移植——如今称为传统移植——采用三任务模型。一个网络任务或线程在网络协议栈内运行，让协议栈处理 ICMP 数据包和 TCP 重传等事务。每个接口有一个接收任务和一个发送任务。接收任务从 MAC 接收数据并交付协议栈，发送任务清空输出队列，把 mbuf 发给 MAC。所有网络任务优先级相同，运行时需持有单一网络信号量。这种设计意味着网络并发有限，多接口吞吐量受限，因为信号量把所有处理串行化。不过，该实现在可重入线程系统中安全可靠，实践中工作良好。

### 维护

这次移植完成后的若干年里，开始暴露一些问题。第一个也是最明显的问题是驱动可用性。定制驱动支持要求为 RTEMS 编写定制驱动。代码可以从 FreeBSD 借鉴，但驱动往往需要当作全新代码来编写和测试。随着时间推移和 FreeBSD 演进，让 RTEMS 用户感到沮丧和困惑的是：FreeBSD 协议栈的移植却不支持较新 FreeBSD 版本的驱动。当初把代码移植到 RTEMS 时，它被复制到简化的目录结构中。FreeBSD 的目录树又大又宽，似乎没有必要再创建一个宽泛的稀疏树来存放少量文件集合。这使得将文件与 FreeBSD 原始文件进行比较变得困难，因为永远不清楚 RTEMS 源码中的哪些文件应该与 FreeBSD 源码中的哪些文件对比。为了让代码能构建，做了修改，但没有任何明确的标记说明哪些是原始代码、哪些被改过。随着时间推移和 FreeBSD 改进协议栈（例如加入 IPv6），从 RTEMS 源码树中的快照升级变得几乎不可能。针对局部修复，会手工引入特定的 bug 修复，但这既耗时又范围狭窄。这导致维护问题随代码老化日益严重。

在斯坦福 SLAC 国家加速器实验室工作的 Till Straumann 构建了一个名为 libbsdports 的附加库，允许 FreeBSD 的驱动以最小改动使用。这非常成功，引发了能否以最小改动使用 FreeBSD 代码的想法。他的工作仅限于网络驱动和特定架构范围，但这是首个公开证明此举可行的例子。

这段时间还有其他 FreeBSD 相关工作。Chris Johns 拿来 USB 协议栈，在 NIOS-II 上的 RTEMS 上运行。然而，这些工作是孤立的、特定的，对 RTEMS 项目没有长期价值。这也凸显了将 FreeBSD 内核的特定部分分别移植的问题：如何把这些分离的部分整合到一个静态可执行文件中？RTEMS 需要一个统一、完整的 FreeBSD 内核代码移植，覆盖所有感兴趣的部分。因此，制定了一个计划来解决 RTEMS 中 FreeBSD 源码的使用问题。第一个决定是退一步，不分散精力。RTEMS 要使用的 FreeBSD 代码库的各个部分需要放在一起，作为单一实体维护。该项目命名为 RTEMS LibBSD。

## RTEMS LibBSD

RTEMS 中最近的 FreeBSD 移植项目名为 RTEMS LibBSD 或简称 LibBSD。该项目托管在 RTEMS 项目 Git 服务器上的独立 Git 仓库中，仓库地址为 <https://git.rtems.org/rtems-libbsd.git>。这是由 Joel Sherrill 和 Sebastian Huber 牵头的联合努力。

该项目为 RTEMS 创建了单一的 FreeBSD 移植，提供 FreeBSD 中对 RTEMS 有用的诸多特性，如网络、USB、SATA 和 MMC 设备。

随着工作推进，制定了一套广泛的规则来指导 RTEMS LibBSD 的开发者。这些规则随团队发现什么可行、什么不可行而逐渐成形。规则可概括为：

1. RTEMS LibBSD 代码的目录结构必须与 FreeBSD 源码树一致。
2. RTEMS 版本代码中的所有修改都必须限定在标准条件定义语法内。这样可以用 Python 脚本移除 RTEMS 修改，将源码与原始 FreeBSD 代码对比。
3. 不要编辑 FreeBSD 代码，包括任何空白字符修改。所有编辑都放在预处理条件中。

能够透明地使用原始 FreeBSD 源码是 LibBSD 工作的核心，“源码透明性”一词被用来描述这一方法。任何将 FreeBSD 代码与自己的系统和机器头文件嵌入的人，都希望能取一部分 FreeBSD 源文件直接构建而无需修改。目前这还不可能。从 RTEMS 项目的视角看，任何透明源码都没有修改，随着修改增多，原始 FreeBSD 代码变得越来越不清晰。

嵌入 FreeBSD 内核代码时需要解决的主要问题有：

1. 头文件和所需声明。RTEMS 目标的系统和机器头文件与 FreeBSD 内核代码使用的头文件不匹配。头文件标准化的改进对双方都有帮助，但仍有一系列内核类型和定义需要添加。
2. 在内核中使用基于标准的用户空间函数名但签名不同，例如 `malloc`。RTEMS 是单地址空间、静态链接的可执行文件，这些名称冲突需要管理，通常是——用糟糕的 hack 管理。
3. 支持 FreeBSD 使用的 SYSINIT 初始化。这需要链接器支持以实现正确的段管理。FreeBSD 的实现方式非常好，RTEMS 内核也采用了类似机制，效果显著。RTEMS 采用了 FreeBSD 通过 SYSINIT 使用的链接器初始化机制。此前，RTEMS 要求用户在构建系统中管理需要链接的 RTEMS 部分，对于不需要的部分，需要链接虚版本。有了链接器集合初始化，这一切都自动化了，包括初始化发生的顺序，使 RTEMS 更易使用。尽管一直关注可执行文件的小体积，但得益于改为 SYSINIT 风格初始化，RTEMS 最小参考应用的体积也有所减小。
4. SMP 支持要求 FreeBSD 移植的某些部分在 RTEMS SMP 支持的上下文中管理。
5. 在单一、静态链接的可执行文件中运行 FreeBSD 用户空间代码，需要一些有趣的 hack 来避免全局符号冲突并让已初始化变量正常工作。将 shell 命令移植到单地址空间时尤为明显。每个命令都有自己的 `main()`，调用 `exit()` 不会退出命令调用——而是退出整个 RTEMS 应用。

## 源码管理

源码树由四个主要目录组成：

1. freebsd——RTEMS 的 FreeBSD 源码
2. rtemsbsd——RTEMS 的 FreeBSD 支持源码
3. testsuite——RTEMS LibBSD 的测试
4. rtems_waf——RTEMS 针对 RTEMS BSP 构建的 waf 支持

使用 LibBSD 时，必须初始化 `rtems_waf` 的 Git 子模块，将其支持文件带入克隆的仓库。该模块帮助为 BSP 配置和构建 LibBSD。还需要已构建并安装 RTEMS 工具链和合适的板级支持包（BSP）。

LibBSD 的开发者还需要从快照点克隆 FreeBSD 源码树。这会创建一个名为 `freebsd-org` 的额外目录。

RTEMS LibBSD 项目有约 850 个构建目标。为管理如此多的文件以及复杂的编译时定义、架构特定文件和头文件，定义被隔离到独立于构建系统的单一文件中。初始开发生成了 makefile，最近改为生成 waf 脚本（<https://www.waf.io>）。waf 构建脚本与 `rtems_waf` 支持集成，更易提取和使用 BSP 的各种机器特定编译标志。由于 RTEMS 是单地址空间，操作系统、用户代码和 LibBSD 等库必须使用相同 ABI 构建，这一点至关重要。

LibBSD 的构建定义是名为 `libbsp.py` 的单一 Python 文件。该文件包含若干模块定义，这些模块被传递给源码管理工具，管理在原始 FreeBSD 源码树和 RTEMS FreeBSD 源码树之间移动源码。模块数据也传递给构建脚本生成器以创建 waf 构建脚本。

支持多种不同类型的模块。模块可以是 RTEMS 头文件和源码、FreeBSD 内核空间头文件和源码，或 FreeBSD 用户空间头文件和源码。模块定义可以指定特定的编译器标志，也可以针对特定架构特化。大量源码对所有架构通用；但某些文件特定于某些架构，例如架构特定的 IP 校验和例程。

要向 LibBSD 添加新源文件，首先初始化 FreeBSD Git 子模块，用 LibBSD 当前基于的特定版本填充 `freebsd-org` 源码树。反向运行 `freebsd-to-rtems.py` 脚本，将 RTEMS FreeBSD 中已修改的源码移回原始 FreeBSD 源码树。编辑源码定义添加新文件，正向运行同一脚本。源码将被复制到 LibBSD 树，构建脚本也会更新。然后即可构建 LibBSD 并编辑源码使其运行。

用户空间符号冲突通过维护一个头文件来管理，该头文件将符号重定义到不同的命名空间。这并不理想，但可以工作。然而，冲突的符号引发了一个常见问题：需要在任何 FreeBSD 代码出现之前包含某些头文件。必须修改标准代码来包含头文件，这占据了对原始源码修改的相当比例。在正常的 FreeBSD 内核树中添加一个空头文件会有助于此类移植工作，因为它可以包含一个 RTEMS 特定版本，其中包含符号重定义。这是内核中一个简单的修改，但对 RTEMS 项目影响很大。

## FreeBSD 核心 API 与 RTEMS 映射

FreeBSD 内核支持基于 RTEMS 服务实现。以下描述这些映射：

1. 共享/独占锁 **SX(9)** 映射为二进制信号量。这忽略了允许共享访问的能力。
2. 互斥 **MUTEX(9)** 映射为二进制信号量。非递归互斥锁不受支持。
3. 读写锁 **RWLOCK(9)** 映射为二进制信号量。这忽略了允许多个读取者访问的能力。
4. 睡眠队列 **SLEEPQUEUE(9)** 使用 FreeBSD 实现，针对 RTEMS 做了调整。
5. 条件变量 **CONDVAR(9)** 使用 FreeBSD 实现（不支持信号）。
6. 定时器函数 **CALLOUT(9)** 主要使用 FreeBSD 实现。轮转驱动是 RTEMS 的定时器服务器例程。
7. 任务 **KTHREAD(9)**、**KPROC(9)** 映射为 RTEMS 任务。
8. 设备管理 **DEVCLASS(9)**、**DEVICE(9)**、**DRIVER(9)**、**MAKE_DEV(9)** 使用 FreeBSD 实现。
9. 总线和 DMA 访问 **BUS_SPACE(9)**、**BUS_DMA(9)** 映射为板级支持包实现。提供了内存映射线性访问的默认实现，当前 RTEMS 堆实现支持 `bus_dma` 要求的所有属性。
10. 通用内存分配器 **UMA(9)** 使用简单的页面分配器作为后端分配器来支持。

从维护角度看，RTEMS 开发者希望这些 FreeBSD 内核 API 保持稳定。这降低了 FreeBSD 更新需要对 RTEMS 的核心内核 API 实现进行大量工作的风险。

## 用户空间

使用 FreeBSD 的重要部分是用户空间支持。`sysctl`、`ifconfig`、`netstat` 等命令提供了重要的用户界面。支持这些类型的命令不仅使 FreeBSD 软件可用，还让 RTEMS 能利用大量现有的 FreeBSD 文档。这是一个重要但常被忽视的复用来源。

将独立的用户空间程序移植到单一可执行文件是一项复杂而精细的操作。由于 RTEMS 只有单一地址空间，**fork(2)** 和 **exec(3)** 都不可用于设置新的进程上下文。C 代码的许多常见假设不能依赖。RTEMS 必须模拟 **fork(2)** 和 **exec(3)** 语义的子集。这通过构建时、链接时和运行时技术的组合实现。LibBSD 以包装函数的形式提供了一些抽象来辅助。

将这类代码引入 RTEMS 时，关键是没有全局变量。可以使用一系列预处理技巧，但没有一个是优雅的。任何全局数据都需要在命令每次运行时初始化。`getopts` 的使用需要替换为 Newlib 提供的非标准可重入版本。最后，`main()` 被替换为诸如 `main_dd()` 的函数名。`exit()`、`perror()` 等少数重要函数也被替换为对 LibBSD 命令支持代码的调用。

LibBSD 中对命令的通用支持只允许一次运行一个命令。虽然这不够理想，但 RTEMS 的多用户访问通常不是问题。通用支持将特定 `main()` 的调用包装在 `setjmp()` 调用中。`exit()` 调用由 `longjmp()` 调用实现。由于 RTEMS 是单进程、多线程环境，在应用中执行原生 **exit(3)** 会关闭整个应用。

生成的代码在某些地方不太美观；但结果相当可观。有一个工作的网络协议栈及相关的 FreeBSD 命令，包括 `ifconfig`、`netstat`、`ping` 和 `tcpdump`——在实时操作系统上运行这个命令挺有趣。在 ARM、PowerPC 和 x86 系统上，收发两个方向都实现了完整的千兆以太网性能。

## 初始化

随着 LibBSD 中的代码更趋稳定并支持更多 BSP，用户开始迁移到 LibBSD，RTEMS 开发者正在处理引导时初始化。为解决这个问题，决定采用 FreeBSD 使用的标准方法。

最近 Chris Johns 向源码树添加了支持 **rc.conf(5)** 的代码。这是 C 实现，因为 RTEMS 没有 POSIX sh。支持 **rc.conf(5)** 来初始化代码的好处是互联网上有大量文档和解决方案可用。这对 RTEMS 项目在文档方面是巨大的精力节省。

Chris Johns 还移植了 `sysctl(8)` 命令，正在添加对读取 **sysctl.conf(5)** 的支持。随着这项支持成熟，用于管理各种内存大小配置选项的 RTEMS 特定代码将被移除。这将既改善可移植性，又缩小使用和配置 LibBSD 与 FreeBSD 的差异。

## 与 FreeBSD 的未来

LibBSD 目前有多好？这个问题的答案取决于视角。任何需要使用 RTOS 的 FreeBSD 用户都会对 LibBSD 的工作感到满意。他们拥有功能完备、高性能、稳定的网络协议栈，支持 IPv6、包过滤、虚拟 LAN 等，性能达到世界级。用户可以访问广泛的驱动，设备驱动通常能快速移植，改动很少。

RTEMS 开发者则挑剔得多。理想状态是能不加修改地使用 FreeBSD 代码。虽然这可能永远不会实现，但确保团队朝此努力至关重要，因为这能降低维护负担。

LibBSD 目前停留在 FreeBSD 9.x 版本，因为升级需要的精力超过现有资源。这使 LibBSD 落后两个主要 FreeBSD 发行版（即 FreeBSD 10 和 11）。这是由多种问题共同造成的，包括 FreeBSD 的一些变更和一些未明确标记的代码。

LibBSD 目前有 1,295 个源文件，其中 797 个（即 61%）没有修改。在有修改的 498 个文件中，平均不透明度为 1.6%。不透明度是一个简化计算，将插入和删除的数量作为总代码行数（含插入和删除）的百分比。不透明度超过 10% 的文件不到 50 个。可以说这是一个相当成功的结果。

还有哪些工作要做？

- 升级当前代码以更好地跟踪 FreeBSD。这方面始终需要投入精力。但 RTEMS 开发者希望，通过向 FreeBSD 内核维护者说明一些简单而有帮助的修改，LibBSD 的维护负担将可控。
- LibBSD 支持的架构数量需要增加。LibBSD 目前已知可在三种架构上工作，而 RTEMS 本身支持 18 种架构。RTEMS 是在不常可访问的架构上进行构建的良好基础参考。
- 虽然从功能角度看并不关键，但 LibBSD 中的编译时警告目前被忽略。RTEMS 构建环境与正常 FreeBSD 内核环境差异显著。但希望看到这些警告得到处理。一如既往，许多警告无害，但有些可能预示真实 bug。

FreeBSD 代码库很重要，有许多用户以原作者从未想到的方式复用源码。RTEMS 项目公开且尽一切机会承认从 FreeBSD 获取的代码。这是一项不可思议的资源，RTEMS 项目感谢它能被使用。

## 参考文献

[1] VMEBus Industry Trade Association（VITA），Real-Time Executive Interface Definition（RTEID）2.1 标准。RTEID 2.1 归档于 <ftp://ftp.rtems.org/pub/rtems/people/joel/RTEID-ORKID/RTEID-2.1/>。

[2] Newlib C 库。<https://sourceware.org/newlib>。

[3] Cygwin。<https://www.cygwin.com>。

[4] G. Gilliand、J. Sherrill。《A Unique Approach to FACE conformance》。美国陆军航空 FACE 技术交流会。<http://face.intrepidinc.com/wp-content/uploads/2016/01/DDC-I-OAR-A-Unique-Approach-to-FACE-Conformance.pdf>。（2016 年 2 月）

[5]《Proceedings of Technology Showcase Held in Huntsville, Alabama on 7–9 August 1990》，包括位于阿拉巴马州红石兵工厂的陆军导弹研究开发与工程中心代表关于 RTEMS 的演讲。<http://www.dtic.mil/dtic/tr/fulltext/u2/a247043.pdf>。（1990 年 8 月）

[6] KA9Q。<http://www.ka9q.net/code/ka9qnos/>。

[7] A. Subbarao。《POSIX—25 Years of Open Standard APIs》。<http://www.rtcmagazine.com/articles/view/103514>。

[8] FACE 联盟产品，包括技术标准、一致性测试套件及其他支持产物。<https://www.opengroup.org/face>。

---

**Chris Johns**

Chris Johns 是 RTEMS 开发者和实时软件工程师，对底层工具和调试器感兴趣。他与家人和两只狗居住在澳大利亚悉尼。chrisj@rtems.org

**Joel Sherrill**

Joel Sherrill 是阿拉巴马州亨茨维尔 OAR 公司的研发总监，也是 RTEMS 最初的团队成员之一。joel@rtems.org

**Sebastian Huber**

Sebastian Huber 是 RTEMS 开发者，目前专注于 SMP 支持。他居住在德国慕尼黑。sebh@rtems.org

**Ben Gras**

Ben Gras 是荷兰阿姆斯特丹 VU 大学的系统安全研究员，也是 RTEMS 贡献者。他居住在阿姆斯特丹。beng@rtems.org

**Gedare Bloom**

Gedare Bloom 是华盛顿特区霍华德大学计算机科学助理教授，也是 RTEMS 的维护者。gedare@rtems.org
