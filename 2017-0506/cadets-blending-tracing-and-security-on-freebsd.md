# CADETS：在 FreeBSD 上融合追踪与安全

在 FreeBSD 上融合追踪与安全 CADETS

FreeBSD 操作系统已被用于许多需要强安全性的产品——从存储设备，到网络交换机、路由器以及游戏主机。在 FreeBSD 二十多年的发展中，安全领域已有显著进展。

作者：Jonathan Anderson、George V. Neville-Neil、Arun Thomas 和 Robert N. M. Watson

Jails [1] 系统引入了今天看来可视为轻量级虚拟化的机制。2002 年加入的强制访问控制（MAC）框架，提供了对用户和程序在系统中可执行操作的细粒度、全系统控制 [2]。审计子系统增加了按系统调用逐条追踪系统中所发生操作及其发起者的能力 [3]。近期 Capsicum 为操作系统引入了能力（capability），成为应用沙箱的基础 [4]。作为一个新研究项目的一部分，因果自适应分布式高效追踪系统（Causal Adaptive Distributed, and Efficient Tracing System，CADETS）正在为 FreeBSD 添加新的安全原语，同时利用现有原语，打造一个具有最高透明度的操作系统。抛开安全影响不谈，对系统具备更好的可见性也将帮助我们提升整体性能，并提供新的运行时调试工具。本文将介绍我们在 DTrace 和审计系统方面的工作——它们构成了这项工作的核心组件，并讨论将 DTrace 作为常驻追踪系统用于跟踪安全及其他事件所面临的挑战。

## 从头开始

我们在 CADETS 中利用的 FreeBSD 主要组件之一是 DTrace。DTrace 最初于 21 世纪初为 Sun 的 Solaris 操作系统开发 [5]，旨在解决以下问题：大多数软件系统都有某种日志框架，通常依赖无处不在的 `printf` 函数，将文本格式化后发送到某种控制台。

```c
printf("Hello world!");
```

所有 C 程序员都熟悉上面的代码，每位程序员也熟悉自己所用语言中的等价惯用法，无论是 Python、Rust、Go 还是 PHP。使用 print 语句构建日志系统存在几个问题。第一个问题是 print 语句的性能开销很高。如果你想了解 `printf` 替程序员做了多少工作，可以参考 Brooks Davis 的“关于‘hello, world’你想知道的一切”[6]。对于本应具有低开销的代码而言，用 print 语句实现日志系统并非可行之选，因此大多数日志系统要么在编译时通过 `#ifdef`/`#endif` 语句启用或禁用，要么在运行时将日志调用包裹在 if 语句中。C 语言中这一惯用法如下所示。

```c
#ifdef LOGGING
if (log)
    printf("You have written %d bytes", len);
#endif /* 日志 */
```

基于 print 的日志系统还存在静态和易错的问题。如果程序员在代码构建前没有通过日志系统暴露某条信息，那么在不修改代码的情况下，将无法得知同一函数可能正在处理的任何其他数据。这两个问题叠加，意味着大多数高性能系统（如操作系统）在发布时不启用日志；即便启用日志，可用数据也仅限于原始程序员希望暴露的内容。作为开源开发者，我们习惯了“重新编译代码就行”的想法，但在生产系统中这并不总是可行。设想你把系统卖给了某家大银行。凌晨 3 点，系统发生某种故障并记录了错误。IT 部门有人收到故障通知，他们联系支持，支持联系程序员，程序员随后说：“停掉系统，重新构建代码，开启日志后重新运行。”以这种方式处理错误对客户和开发者来说都极其糟糕。客户会因为要重建并重启系统而恼火，而开发者也不太可能获得有用信息，因为错误早已时过境迁。

动态追踪应运而生。DTrace 被设计为始终可用，既不降低系统性能，也不会冒导致系统崩溃的风险。理解 DTrace 最清晰的方式，是把它看作具备强大日志和统计能力的运行时调试器。DTrace 通过一些技巧来颠覆已编译代码的执行，从而规避了基于 `printf` 的日志系统的开销。完整细节参见 **The Design and Implementation of the FreeBSD Operating System** [7]，简而言之，DTrace 可以覆盖编译进内核或用户态程序的任何函数的入口或出口点。函数被收集到库和程序中的方式是一个定义明确的过程，每个函数都有一组指示函数起点的指令。DTrace 追踪函数时，会用自己的几条指令替换原有指令，当代码执行到该点时，DTrace 的代码会首先被调用，从而收集并处理函数参数。由此带来的实际效果是：DTrace 未被使用时开销为零。

DTrace 的这一特性使其得以默认随 Solaris、FreeBSD 和 Mac OS 一起发布。只有找到一种不会对运行中系统产生性能影响的追踪系统部署方式后，这样的系统才能投入实用。那么 DTrace 开始追踪时会发生什么？取决于追踪的对象和收集的数据量，系统的整体性能会受到或大或小的影响。如果追踪引入的开销过高，内核将作为一种自我保护机制终止追踪。DTrace 系统的第二条原则——追踪系统不得过度拖累系统——正是将 DTrace 用作安全技术的关键挑战之一。知晓可能存在追踪的攻击者，会先制造大量无关负载，迫使追踪系统退出，然后才对系统发起攻击。CADETS 项目的另一个组成部分研究追踪的来源，以应对针对追踪系统本身的此类攻击。DTrace 当前的大多数用例都将其用作运行时调试器——当有人怀疑系统存在问题时才开启追踪，而非让追踪始终运行。少数用户围绕 DTrace 构建了复杂的遥测系统，包括 Fishworks [8]，但在这些早期的常驻追踪尝试中，被追踪的事件只是所有可能追踪点的很小一部分。要构建一个具备完全可见性的系统，我们不仅需要常驻追踪，还需要提升追踪系统的性能，使得收集数据的开销不会压垮系统执行有效工作的能力。面向安全的追踪系统还有一个要求：追踪记录不能因高负载而被丢弃或丢失。DTrace 的设计允许在高负载时丢弃追踪记录，远在内核因系统无响应而终止 dtrace 收集进程之前。但一个旨在为后续取证分析收集追踪数据的系统，恰恰要将这一观念颠倒过来。任何追踪始终运行的系统都会产生大量数据，这些数据需要被分析以追踪攻击者以及他们攻破系统的方式。已有若干系统可用于接收各种工具的任意文本输出并尝试理解其含义，包括 Splunk [9]。CADETS 项目的目标就是为这类工具生产数据。如果数据采用机器可读格式，消费方将轻松许多。

## 面向安全的追踪

我们如何将 DTrace 应用于安全问题？安全的一个方面是找出恶意行为者潜伏在系统的何处。审视系统时需要问的一个关键问题是“谁在何时对谁做了什么？”我们称之为“问题 1”。建立操作树以便追溯其根源，是取证分析的一部分，能帮助我们找到恶意行为者，并揭示需要修改什么来防止未来发生安全违规。设想，无论性能代价如何，我们对计算机系统上执行的每个操作都拥有完全的透明度。只要有足够的时间、分析和工具，我们就能从追踪系统产生的输出中找到问题 1 的答案。DTrace 为我们构建这样的系统奠定了基础，但仍有若干挑战。上一节提到，DTrace 通过巧妙的技巧在运行时替换某些指令，在不使用时开销为零。正是这一特性让 DTrace 得以默认随 Solaris、FreeBSD 和 Mac OS 发布。DTrace 开始追踪后会发生什么？取决于追踪的内容和数据量，系统整体性能会受到不同程度的影响。如果追踪引入的开销过高，内核会终止追踪以自我保护。DTrace 系统的第二条原则——追踪系统不得过度拖累系统——是将 DTrace 用作安全技术的关键挑战之一。

## 面向软件工具的追踪记录

在将 DTrace 用作常驻追踪系统的过程中，我们很快意识到，解析系统产生的大量输出不仅对人类分析师构成挑战，对任何消费这些数据的工具也是如此。我们为 CADETS 给 DTrace 添加的首批功能之一就是机器可读输出。借助 libxo 库，我们将 DTrace 的输出从纯文本扩展为同时支持 XML、JSON 和 HTML——libxo 支持的三种输出格式。我们添加机器可读输出时，Illumos 版本的 DTrace 虽然已有输出 JSON 的方式，但那是通过类似 print 的操作实现的，而非全局性改动。在 CADETS 版本的 DTrace 中，单个命令行选项即可将用户从 DTrace 看到的全部输出从纯文本转换为机器可读格式。上面的代码示例展示了纯文本输出与机器可读输出之间的差异。我们的示例让 DTrace 追踪所有对 `write` 系统调用的调用。示例 1 展示了 `dtrace` 命令默认的文本输出，按列排列。要编写解析该输出的工具，需要对输出有相当深入的了解，因为列名只在输出开头标注一次。示例 2 展示了相同的追踪点，但增加了 `-O json` 命令行参数，指示 `dtrace` 以 JSON 格式输出所有结果。

```sh
# dtrace -n 'syscall::write:entry'
dtrace: description 'syscall::write:entry' matched 2 probes
CPU ID FUNCTION:NAME
0  59780 write:entry
0  59780 write:entry
```

示例 1

```sh
# dtrace -O json -n 'syscall::write:entry'
dtrace: description 'syscall::write:entry' matched 2 probes
CPU ID FUNCTION:NAME
{ "probe": {
"timestamp": 3594774042481656,
"cpu": 1,
"id": 59780,
"func": "write",
"name": "entry"
}
```

示例 2

机器可读输出为每个元素打上标签，从探针（probe）开始。探针是一个包含若干元素的对象，包括 `cpu`、`id`、`func` 和 `name`，这些在示例 1 的机器不可读输出中都未标注。机器可读输出新增的一个元素是 `timestamp`，以纳秒为单位报告探针触发的时间，自 UNIX 纪元起算。虽然 DTrace 脚本可以使用 `timestamp` 或 `walltimestamp` 变量输出时间，但我们认为在每个机器可读记录中都包含时间戳将简化安全取证工具的构建。虽然探针可能在不同核心上并行触发（由 `cpu` 变量指示），但现代基于 Intel 的系统拥有同步的时间戳，DTrace 使用的正是它，这意味着这些时间戳能很好地指示单个系统上操作的先后顺序。

## DTrace 与审计

审计子系统自 2004 年起成为 FreeBSD 的一部分，是一个可选的内核组件，连同用户态守护进程 `auditd` 一起，实现了“对安全相关事件的细粒度、可配置日志记录”[10]。它旨在满足美国政府的“通用准则（CC）通用访问保护 Profile（CAPP）评估”这一安全标准 [11]。审计子系统通过 C 宏在内核中访问数据和资源的位置添加了一组手工编写的追踪点。例如，在启用审计的系统中，每次访问文件描述符时都会在审计记录中记上一笔。审计记录会定期刷新到永久存储。我们近期在 DTrace 和安全方面的工作促使我们希望在审计系统与 DTrace 之间搭建桥梁。Robert Watson 在 FreeBSD 的 DTrace 系统中添加了审计提供者（audit provider）。

DTrace 提供者（provider）使得 DTrace 可以访问一组追踪点。一些知名的提供者示例包括处理函数边界追踪点（fbt）、系统调用（syscall）和网络协议（tcp、udp、ip）的提供者。审计提供者让 DTrace 能够记录系统上发生的审计事件信息，同时应用 DTrace 的功能，如通过谓词过滤事件以及通过聚合收集统计信息。在没有 DTrace 的情况下使用审计系统时，我们无法方便地编写运行时分析脚本来更精确地定位想要调查的进程。审计系统会针对特定进程，但我们希望能够仅在该进程执行特定操作时才收集数据。设想这样一个场景：我们想查看谁在与 Web 服务器通信；我们可能只想了解来自一组特定 Internet 地址的连接，也许因为我们知道这些地址已被标记为僵尸网络的一部分。借助审计提供者，我们可以编写一个简单的 D 脚本，只请求与 `connect(2)` 相关的审计事件，然后根据事件中包含的 IP 地址进行过滤。尽可能靠近信息源头进行数据缩减，不仅减轻了系统的整体负载，还减少了人类分析师或软件工具在后续分析阶段需要审视的数据量。

## OpenDTrace：DTrace 的未来

除 Illumos 外，还有两个操作系统项目移植并采用了 DTrace。FreeBSD 自 2008 年起拥有 DTrace 移植版本，Apple 的 Mac OS 自 2007 年起也支持 DTrace，并集成到了 Instruments 性能分析工具中。随着 2010 年 Sun Microsystems 的消亡，DTrace 的开发分裂，部分工作在 Oracle 内部进行，但大部分转移到了 Illumos——OpenSolaris 的完全开源延续。FreeBSD 和 Mac OS 持续从 Illumos 引入变更，但这些主要是 bug 修复而非大型新功能。三方都尽力共享代码，但三个内核之间存在显著差异。DTrace 虽然以良好的可移植风格编写，但仍需要许多与操作系统相关的专用钩子，这导致了一定程度的代码分叉。例如，FreeBSD 版本的 DTrace 同时支持 ARMv8 和 ARMv7 处理器，但 Illumos 上尚非如此。作为 CADETS 工作的一部分，我们意识到希望为 DTrace 添加新功能，并进一步将代码与任何特定操作系统解耦，这促使我们创建了 OpenDTrace。OpenDTrace 的目标是提供一个统一、单一的 DTrace 代码上游，使其能被轻松导入其他操作系统，包括 FreeBSD、Mac OS、Illumos 等。这一思路类似于 OpenBSM [12] 和 OpenZFS [13] 的做法。有了统一的代码库，我们就能以远超今天的速度为所有下游 DTrace 消费方添加功能。

## 致谢

作者感谢 Ripduman Sohan 对手稿提出的宝贵意见。•

JONATHAN ANDERSON 是纽芬兰纪念大学电气与计算机工程系的助理教授，研究横跨操作系统、安全和编译器等软件工具。他是 FreeBSD 提交者，并一直在寻找志趣相投的新研究生。

GEORGE V. NEVILLE-NEIL 出于兴趣和生计从事网络与操作系统代码工作，也教授与编程相关的各类课程。他的兴趣领域包括代码探源、操作系统、网络和时间协议。他与 Marshall Kirk McKusick 和 Robert N. M. Watson 合著了 **The Design and Implementation of the FreeBSD Operating System**。十余年来，他以 Kode Vicious 之名撰写专栏。他在马萨诸塞州波士顿的东北大学获得计算机科学学士学位，是 ACM、Usenix 协会和 IEEE 的会员。他热爱骑行和旅行，现居纽约市。

ARUN THOMAS 是 BAE Systems 研发部门的研究员，也是因果自适应分布式高效追踪系统（CADETS）项目的首席研究员。

DR ROBERT N. M. WATSON 是剑桥大学计算机实验室的高级讲师（副教授），领导横跨操作系统、安全和计算机体系结构的研究。他是 FreeBSD 开发者、FreeBSD 基金会 董事会成员，并合著了 **The Design and Implementation of the FreeBSD Operating System**（第二版）。

本研究由美国国防高级研究计划局（DARPA）和美国空军研究实验室（AFRL）资助，合同编号 FA8650-15-C-7558。本文所含的观点、意见和/或发现均为作者本人观点，不应被解读为代表美国国防部或美国政府的官方观点或政策，无论明示或暗示。

[1] “Jails: Confining the omnipotent root.” Poul-Henning Kamp <phk@FreeBSD.org> Robert N. M. Watson <rwatson@FreeBSD.org> (2000)
[2] Watson, R.; Feldman, B.; Migus, A.; and Vance, C. “Design and implementation of the Trusted BSD MAC framework,” Proceedings—DARPA Information Survivability Conference and Exposition, DISCEX 2003, 1(Discex Iii), 38–49. <http://doi.org/10.1109/DISCEX.2003.1194871> (2003)
[3] Watson, R. N. M. and Salamon, W. “The FreeBSD Audit System,” UKUUG LISA Conference, 1–6. (2006)
[4] Watson, R. N. M.; Anderson, J.; Laurie, B.; and Kennaway, K. “Capsicum: practical capabilities for UNIX,” 19th Usenix Security Symposium, (Figure 1), 3. <http://doi.org/10.1145/2093548.2093572> (2010).
[5] Cantril, B.; Shapiro, M. and Leventhal, A. “Dynamic Instrumentation of Production Systems.” (2004)
[6] Davis, B. Everything you ever wanted to know about “hello, world”\* (\*but were afraid to ask), BSDCan. (2016)
[7] McKusick, M.; Neville-Neil, G.; and Watson, R. N. M. The Design and Implementation of the FreeBSD Operating System, Second Edition. Boston, Massachusetts: Pearson Education. (2014)
[8] <http://dtrace.org/blogs/bmc/2008/11/10/fishworks-now-it-can-be-told/>
[9] <https://www.splunk.com>
[10] <https://www.freebsd.org/cgi/man.cgi?query=audit&sektion=4>
[11] <https://www.niap-ccevs.org/pp/pp_os_ca_v1.d.pdf>
[12] <http://www.trustedbsd.org/openbsm.html>
[13] <http://open-zfs.org/wiki/Main_Page>
