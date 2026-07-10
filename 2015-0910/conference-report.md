# 会议报告

- 原文：[Conference Report](https://freebsdfoundation.org/our-work/journal/browser-based-edition/cloudabi/)
- 作者：**Dr. Robert N. M. Watson**

<https://wiki.FreeBSD.org/201508DevSummit>

## BSDCam 2015

BSDCam 2015 是一年一度的 FreeBSD 开发者峰会，于 2015 年 8 月 17–19 日在英国剑桥大学计算机实验室举行。BSDCam 由剑桥大学主办与组织，并得到 FreeBSD 基金会 与 Xinuos 的资助。

超过 50 名 FreeBSD 开发者、剑桥当地的博士生与研究人员、使用（或关注）FreeBSD 的科技公司嘉宾出席了此次开发者峰会。

计算机实验室在研究中广泛使用 FreeBSD，尤其在安全、网络、操作系统、程序分析与计算机体系结构研究方面——这些主题贯穿整个峰会。剑桥也是 ARM Ltd. 的所在地，该公司主导 ARM 架构开发；ARM 在 BSDCam 上派员主持了电源管理与嵌入式架构的会议，并在 ABI、工具链与基于处理器的计数器与追踪等议题中作出贡献。其他参会机构包括 BAE Systems、EMC、FreeBSD 基金会、Netflix、Neville-Neil Consulting、Nuxi、ScaleEngine、SRI International、Tarsnap、得克萨斯大学奥斯汀分校、Wheel Systems 与 Xinuos。

计算机实验室非常适合举办开发者峰会，有多个设施齐全的会议室，既能容纳全体大会，也可进行小规模的分组讨论。BSDCam 采用 "非会议" 风格，首先就感兴趣的议题进行头脑风暴，随后形成议程，整个活动期间多数时段有一到两个并行会议。开发者一直聊到深夜，或在酒吧，或于第二晚在 Xinuos 赞助的 Murray Edwards 学院正式会议晚宴上继续交流。

日间会议时长一到两小时，议题包括：

- 存储子系统的未来
- 网络协议栈性能
- 软件包系统
- 工具链/LLVM
- ABI 仿真
- 软件追踪与调试
- 测试与 QA
- 电源管理
- Capsicum 与 CloudABI
- 安全、加密与随机数生成器
- 使用 FreeBSD 教学
- 云中的 FreeBSD
- FreeBSD/RISC-V
- FreeBSD/ARMv8

会议的详细纪要、许多演讲的幻灯片可在 BSDCam 2015 wiki 页面（<https://wiki.FreeBSD.org/201508DevSummit>）查看，以下精选若干会议的摘要。

BSDCam 的举办离不开 Jon Anderson（纽芬兰纪念大学）、Piete Brooks（剑桥大学）、David Chisnall（剑桥大学）、Brooks Davis（SRI International）、George Neville-Neil（Neville-Neil Consulting）、Robert Watson（剑桥大学）与 Bjoern Zeeb（剑桥大学）的贡献。

## 追踪与网络

网络协议栈演进与性能追踪/分析是过去几年在 BSDCam 反复出现的主题，这既源于剑桥当地的研究兴趣，也源于与会者个人与公司的关注。

George Neville-Neil 与 Robert Watson 主持了追踪议题会议，ARM 的 Al Grant 首先介绍了 ARM 的 Streamline 与 Coresight IP 模块。这些工具为基于 ARM 的系统提供硬件辅助追踪与性能可视化，捕获架构与微架构事件、软件生成的事件，并提供便于性能调试的 UI。ARM 与 Intel 一直在尝试统一各自的方法，以提高跨事件源的工具可移植性。随后更广泛的讨论涉及 DTrace 优化、分布式 DTrace、为 DTrace 命令行工具引入机器可读输出（该工具历来以面向用户交互为主）。

显然，DTrace 库与工具亟需大幅重构以支持对象模型输出——但 FreeBSD 中通常用于机器可读输出的 libxo 库可能需要改进，以更好地支持流式数据源与 schema。讨论的另一个话题是如何将 CPU 性能计数器基本集成到 DTrace 框架中——但这受制于 HWPMC（FreeBSD 当前的 PMC 框架）与 DTrace 之间概念不匹配。还有人担心 DTrace 可能引入额外开销，增加使用性能计数器时的探针效应。

George Neville-Neil 主持了网络协议栈会议，涉及广泛议题。重点讨论了继续提升网络协议栈可扩展性，详述了当前已知的多核可扩展性瓶颈，尤其是路由决策与协议栈下层（最好的路由查找是不做查找！）。还大量讨论了 TCP 特性改进、如何在特性增强期间更好地测试 TCP 协议栈以发现回归——过去出现过一些难以在生产环境诊断与调试的微妙问题。剑桥过去在 TCP 状态机建模与基于追踪的测试方面的工作（<http://www.cl.cam.ac.uk/~pes20/Netsem/>），如今借助追踪技术的进步可能获得新应用——但需要更新。讨论的另一个话题是如何将链路层流控事件暴露给 TCP，以更好地处理数据中心网络（这类网络通常无损）。类似地，希望在用户态应用与网络协议栈中更好地支持链路层流控——可能通过 kqueue 改进实现。

## 工具链/LLVM 与 ABI

Ed Maste 主持了工具链与 LLVM 会议，回顾了工具链现状与该领域的持续开发。讨论花了相当时间探讨 LLVM 集成的程度与仍需改进的领域（例如 64 位 MIPS）、LLVM 如今如何促进研究（如 SRI、剑桥、Memorial 与 BAE）——包括通过补丁在 FreeBSD 构建过程中生成 LLVM IR 输出。BSD ELF 工具链的状态也得到讨论——既肯定了其成熟度的巨大提升，也指出了一些持续存在的差距，如对 objcopy 与 objdump 功能的完整支持。LLVM 链接器领域似乎开始向 LLD 收敛，FreeBSD 社区已开始更实质性地贡献。LLDB 移植持续改进，上游 LLDB 进行结构性变更以更好地支持多操作系统。还讨论了是否继续使用 CTF 用于 DTrace，还是尝试直接使用 DWARF——现状似乎会维持，但 BSD 授权的 CTF 生成工具会很有用。Ports 集合中的 LLVM 支持由 arm64 移植驱动，它只使用 LLVM 而非 gcc——但对 Fortran 等编程语言有影响。

Allan Jude 与 Johannes Meixner 主持了 ABI 会议，聚焦 Linux 仿真器的当前工作与其他 ABI 兼容性工作。Linuxulator 开发者继续推进对 Centos 6.7（32 位与 64 位）的支持。对 inotify 缺乏支持似乎仍是 Dropbox 等 Ports 的持续阻碍，其他功能缺失影响了谷歌 Hangouts 等工具，也影响 Xilinx 与 Altera FPGA 工具链。Ed Schouten 介绍了他为 Capsicum 设计的系统调用接口 CloudABI，如今已进入 FreeBSD。Brooks Davis 介绍了他为 CHERI 处理器中用户态胖指针设计的系统调用 ABI——CheriABI。

## Capsicum、CloudABI 与安全

Robert Watson 与 Ed Schouten 主持了 Capsicum 会议，Memorial 的 Jon Anderson 与谷歌的 Ben Laurie 也通过电话会议加入。讨论议题包括谷歌的 Linux Capsicum 项目、Apple 的 launchd 与运行时链接器等概念的趋同、需要为开发者提供更好（更教程式）的文档、Alex Richardson 关于 KDE 支持 Capsicum 的工作、缺乏合适的 RPC 框架。Jon Anderson 介绍了 SOAAP 项目，该项目使用 LLVM 静态分析对应用中的沙箱进行建模，帮助描绘潜在改进并发现 bug——该工作将在 ACM CCS 2015 发表。还讨论了能力“具象化”——从能力派生出文件系统中的持久条目的能力，与 `linkat()` 需要两个权限（from、to）类似 `rename()` 而非如今单一权限的问题，以避免绕过目录能力权限。

由 Robert Watson 主持的安全会议主要聚焦 FreeBSD 新的基于 Fortuna 的随机数生成器的多核可扩展性改进。对熵收集的性能开销一直有担忧，也关注如何避免随机数生成管道中各处的多核通信。Mark Murray 主持讨论了多种模型与限制，最终结论是需要每 socket（甚至每核）一个 Fortuna 实例。这确实需要解决一些难题，如怎样处理不同 socket 或核之间熵源的不对称分布——可能需要再平衡机制。无论如何，目标是继续降低内核随机数生成器的开销，尤其是在熵收集方面。

## ARMv8 与 RISC-V

两场会议讨论了 FreeBSD 正在开发支持的两个新指令集架构（ISA）：ARM 的 64 位 ARMv8 ISA 与加州大学伯克利分校的开源 RISC-V ISA，分别由 Andrew Turner（FreeBSD 基金会——由 ARM 与 Cavium 赞助）与 Ruslan Bukin（剑桥大学）主导。

Andrew 描述了他由 FreeBSD 基金会、ARM 与 Cavium 赞助的 FreeBSD/ARMv8 移植工作，该工作也与 Semihalf 协作进行。FreeBSD 现已在包括 ARM 与 Cavium 评估板在内的多种 64 位 ARM 设备上启动与运行良好。移植工作在稳定性与功能方面持续完善。由 ARM 赞助，Ruslan Bukin 最近基于他早先的 ARMv7 工作完成了 ARMv8 的硬件性能计数器（HWPMC）与 DTrace 初步支持。在 Cavium 对 Semihalf 的支持下，FreeBSD 也可在多核 Cavium Thunder 板上使用。

Ruslan Bukin 与 Ed Maste 主持了 FreeBSD RISC-V 移植的初步会议。RISC-V 是加州大学伯克利分校 RISC-V 团队创建的全新“开源 ISA”；RISC-V 在那里用于多种处理器，也是剑桥正在开发的 LowRISC 开源 SoC 的基础。未来它将用于其他处理器研究与嵌入式系统。Ruslan 现已能让 FreeBSD/RISC-V 在伯克利的 Spike 模拟器中打印启动消息，并开始进行上下文切换与虚拟内存支持的工作。RISC-V 指令集仍在成熟中，他遇到了一些困难——例如 QEMU 实现的是一个较旧（且不兼容）的特权 ISA 版本，工具链也有问题。然而进展迅速。

**作者简介**

Dr. Robert N. M. Watson 是剑桥大学计算机实验室系统、安全与架构方向的大学讲师；FreeBSD 开发者与 core team 成员；FreeBSD 基金会 董事会成员。他领导多个跨越计算机体系结构、编译器、程序分析、程序转换、操作系统、网络与安全的跨层研究项目。近期工作包括 Capsicum 安全模型、用于 Junos 与 Apple iOS 等系统沙箱的 MAC Framework、FreeBSD 网络协议栈的多线程。他是《The Design and Implementation of the FreeBSD Operating Systems》（第二版）的合著者。
