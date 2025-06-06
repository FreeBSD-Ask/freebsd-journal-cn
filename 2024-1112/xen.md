# Xen 与 FreeBSD

- 原文地址：[Xen and FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/virtualization-2/xen-and-freebsd/)
- 作者：**Roger Pau Monné**

Xen 虚拟机监视器（hypervisor）最初始于 1990 年代末剑桥大学计算机实验室，项目名称为 Xenoservers。当时，Xenoservers 的目标是提供“一种新的分布式计算范式，称为‘全球公共计算’，能让任何用户在任何地方运行任何代码。这种平台对计算资源进行定价，并最终按资源消耗向用户收费。”

使用虚拟机监视器可以安全地在多个操作系统之间共享物理机器的硬件资源。虚拟机监视器是管理所有操作系统（通常称为客户机或虚拟机）的软件，并提供它们之间的分离和隔离。Xen 于 2003 年首次作为开源虚拟机监视器在 GPLv2 许可下发布，其设计不依赖操作系统，这使得添加 Xen 支持到新操作系统中变得非常容易。自发布以来，Xen 已获得个人开发者和企业贡献者的大力支持。

## 架构

虚拟机监视器可以分为两类：

- 第 1 类：直接运行在裸金属上，直接控制硬件。
- 第 2 类：作为操作系统的一部分的虚拟机监视器。

常见的第 1 类虚拟机监视器有 VMware ESX/ESXi 和 Microsoft Hyper-V，而 VMware Workstation 和 VirtualBox 则是第 2 类虚拟机监视器的典型代表。Xen 是一种第 1 类的虚拟机监视器，但它的设计与微内核有许多相似之处。Xen 本身仅控制 CPU、原生和 I/O APIC、MMU、IOMMU 和一个定时器。其他部分由控制域（Dom0）处理，Dom0 是一个由虚拟机监视器提供特权的专用虚拟机。这样，Dom0 就能管理系统中的所有硬件，以及运行在虚拟机监视器上的所有其他虚拟机。还需要注意的是，Xen 几乎没有硬件驱动程序，这样可以避免与操作系统中已有的驱动程序发生代码重复。

### 架构

![](https://freebsdfoundation.org/wp-content/uploads/2025/01/xen_driver_domains.png)

在最初设计 Xen 时，x86 架构上没有硬件虚拟化扩展；虚拟化的选项要么是全软件仿真，要么是二进制翻译。这两种选项在性能上都非常昂贵，因此 Xen 采取了不同的方式。Xen 并不打算模拟当前的 x86 接口，而是为虚拟机提供了新的接口。这个新接口的目的是避免虚拟机监视器在处理硬件接口仿真时的开销，而是使用新的接口，这个接口对虚拟机和 Xen 来说都更自然。这个虚拟化方法也被称为 Ring Deprivileging（特权等级下降）。

然而，这要求虚拟机知道它是在 Xen 下运行，并使用一组与原生运行时不同的接口。这一接口集被指定为 ParaVirtualized，因此使用这些接口的虚拟机通常被称为 PV 虚拟机。以下接口在 PV 虚拟机上被 PV 替代：

- 磁盘和网络。
- 中断和定时器。
- 内核入口点。
- 页表。
- 特权指令。

这种方法的主要限制是它需要对虚拟机操作系统的核心部分进行大量修改。目前，仍然支持 x86 Xen PV 的操作系统只有 Linux 和 NetBSD。曾经有个 Windows 的初步移植版可以作为 PV 虚拟机运行，但它从未发布，Solaris 也曾支持 PV。

随着硬件虚拟化扩展的加入，Xen 也获得了支持运行未修改（非 PV）虚拟机的能力。这些虚拟机依赖于硬件虚拟化以及硬件设备的仿真。在 Xen 系统中，这种仿真要么由虚拟机监视器本身完成（对于性能关键的设备），要么通过默认的 QEMU 外部仿真器在用户空间中完成。这种仿真完整 PC 兼容环境的硬件虚拟化虚拟机，在 Xen 术语中被称为 HVM。

因此，我们现在已经讨论了两种非常不同类型的虚拟机，一方面是使用 PV 接口避免仿真的 PV 虚拟机，另一方面是依赖硬件支持和软件仿真以运行未修改虚拟机的 HVM 虚拟机。

HVM 虚拟机使用的仿真 IO 设备，如磁盘或网卡，由于处理数据传输所需的逻辑和仿真旧接口的开销，性能通常不太好。为了避免这一性能损失，Xen HVM 虚拟机还可以选择使用 PV 接口来处理 IO。一些其他 PV 接口也可供 HVM 虚拟机使用（例如一次性 PV 定时器），以减少使用仿真设备时可能产生的开销。

虽然 HVM 允许每个可能的未修改 x86 虚拟机运行，但由于要仿真所有用于 PC 兼容环境的设备，它也具有较大的攻击面。为了减少暴露给虚拟机的接口数量（从而减少攻击面），Xen 创建了一个稍微修改过的 HVM 虚拟机版本，称为 PVH。这是 HVM 的精简版本，其中很多在 HVM 虚拟机中存在的仿真设备不可用。例如，PVH 虚拟机仅提供仿真的原生 APIC 和可能的仿真 IO APIC，但没有仿真的 HPET、PIT 或传统的 PIC（8259）。然而，PVH 模式可能需要修改虚拟机操作系统内核，以便它意识到自己是在 Xen 下运行，并且一些设备不可用。PVH 模式还使用特定的内核入口点，可直接启动到虚拟机内核，而无需依赖仿真固件（SeaBIOS 或 OVMF），因此大大加快了启动过程。然而需要注意的是，当启动速度不重要并且更希望使用便捷性时，OVMF 也可以在 PVH 模式下运行，以便链式加载操作系统特定的启动加载程序。以下是对 x86 上不同虚拟机模式的简要比较表：

|                           | PV             | PVH                    | HVM                    |
| --------------------------- | -------------- | ---------------------- | ---------------------- |
| I/O 设备                    | PV (xenbus)    | PV (xenbus)            | 仿真 + PV              |
| 传统设备                    | 无             | 无                     | 是                     |
| 特权指令                    | PV             | 硬件虚拟化             | 硬件虚拟化             |
| 系统配置                    | PV (xenbus)    | ACPI + PV (xenbus)     | ACPI + PV (xenbus)     |
| 内核入口点                  | PV             | PV + 原生①            | 原生                   |

① 当使用固件启动时，PVH 虚拟机可以重用原生入口点，但这需要在原生入口点中添加逻辑，以检测是否在 PVH 环境中启动。并非所有操作系统都支持此功能。

PVH 方法也被其他虚拟化技术采用，例如 AWS 的 Firecracker。尽管 Firecracker 基于 KVM，但它重用了 Xen 的 PVH 入口点，并通过不暴露（从而不仿真）传统的 x86 设备来实现相同的攻击面减少。

在 ARM 架构方面，Xen 的移植是在 ARM 已经支持硬件虚拟化扩展的情况下开发的，因此与 x86 相比采取了不同的方法。ARM 只有一种虚拟机类型，它相当于 x86 上的 PVH。该方法的重点是尽量不暴露过多的仿真设备，以减少复杂性和攻击面。

预计即将推出的 RISC-V 和 PowerPC 移植版本也将采取相同的策略，只支持一种虚拟机类型，更类似于 x86 上的 HVM 或 PVH。这些平台也具有硬件虚拟化扩展，因此不需要像经典的 PV 支持那样的东西。

## 使用和独特功能

Xen 的首次商业应用严格集中在服务器虚拟化上，主要是 Xen 基础产品的原生使用或通过云服务提供。然而，由于其多功能性，Xen 现在也扩展到客户端和嵌入式领域。Xen 的小巧体积和安全性重点使其适用于广泛的环境。

Xen 在客户端（桌面）应用的一个很好的例子是 QubesOS，这是一款基于 Linux 的操作系统，专注于通过将不同进程隔离到虚拟机中来提高安全性，所有虚拟机都运行在 Xen 虚拟机监视器之上，甚至支持 Windows 应用程序的使用。QubesOS 在一些关键的 Xen 特性上依赖较大：

- 驱动程序域：网络卡和 USB 驱动程序运行在单独的虚拟机中，从而避免了这些设备的安全问题危及整个系统。请参见关于驱动程序域的图示。
- 孤立域：处理每个 HVM 虚拟机仿真的 QEMU 实例不是在 dom0 中运行，而是在一个单独的 PV 或 PVH 域中运行。这种隔离防止了 QEMU 中的安全问题危及整个系统。
- 限制内存共享：通过使用授权共享接口，一个域可以决定哪些内存页面共享给哪些域，从而防止其他域（甚至是半特权域）访问所有虚拟机内存。

与 QubesOS 类似，还有 OpenXT：这是一个基于 Xen 的 Linux 发行版，专注于客户端安全，供政府使用。

### 驱动程序域

![](https://freebsdfoundation.org/wp-content/uploads/2025/01/xen_driver_domains.png)

Xen 在 x86 上的一些独特功能，广泛应用于多种产品：

- 自省：允许外部监视器（通常在用户空间的另一个虚拟机中运行）请求有关虚拟机执行的操作的通知。例如，这种监视可以包括访问某个寄存器、MSR 或更改执行特权级别。这个技术的一个实际应用是 DRAKVUF，一款恶意软件分析工具，它不需要在虚拟机操作系统中安装任何监视器。
- VM-fork：类似于进程分叉，Xen 允许分叉正在运行的虚拟机。这个功能虽然不能创建一个完全功能的分叉，但足以用于内核模糊测试。KF/x 模糊测试项目将内核置于一个特定状态，然后通过创建虚拟机的分叉开始模糊测试。所有分叉从相同的指令开始执行，但输入不同。在非常特定的状态下快速并行分叉虚拟机，对于每分钟迭代次数的提升至关重要。

自 ARM 移植版发布以来，Xen 在嵌入式部署（从工业到汽车领域）上受到了广泛关注。除了小巧体积和安全性重点外，Xen 还有一些关键特性，使其在这些应用中具有吸引力。首先，与第 2 类虚拟机监视器相比，Xen 的代码量相当有限，因此可以设想尝试对其进行安全认证。目前，上游正在努力使 Xen 符合 MISRA C 标准的相关部分，以便进行安全认证。

一些使 Xen 在嵌入式应用中非常有吸引力的独特功能包括：

- 小巧的代码库：使其可以进行审计和安全认证，此外代码库正在适应以符合 MISRA C 标准。
- CPU 池：Xen 可以将 CPU 分区为不同的组，并为每个组分配不同的调度器。然后，可以将虚拟机分配到这些组，从而使一组虚拟机使用实时调度器（如 RTDS 或 ARINC653）运行，而另一组虚拟机则使用通用调度器（如 credit2）运行。请参见 CPU 池示意图。
- CPU pinning：还可以对哪些主机 CPU 调度哪些虚拟机 CPU 进行限制，例如，当运行延迟敏感工作负载时，可以将虚拟机 CPU 独占分配给主机 CPU。
- 确定性中断延迟：Xen 在确保中断延迟保持低且确定性方面投入了大量工作，即使在有噪音邻居引起缓存压力的情况下也能保持这一特性。目前正在审查的补丁系列为 Xen 增加了缓存着色支持。此外，Xen 正在移植到 Arm-v8R MPU（内存保护单元）基础的系统上。这是 Xen 架构的一个重要变化，因为它一直以来都支持基于内存管理单元（MMU）的系统。在 MPU 中，虚拟地址和物理地址之间有平坦映射，因此可以实现实时效果，因为不涉及地址转换。Xen 可以创建有限数量的内存保护区域，以强制执行不同内存范围上的内存类型和访问限制。
- 无 dom0/hyperlaunch：这是一个起源于 ARM 的功能，目前也在为 x86 实现，可在启动时静态创建多个虚拟机。这对于静态分区系统非常有用，其中虚拟机的数量是固定的并且提前已知。在这种设置中，初始（特权）域的存在是可选的，因为某些设置不需要对最初创建的虚拟机进行进一步操作。

### CPU 池

![](https://freebsdfoundation.org/wp-content/uploads/2025/01/cpu_pools.png)

## FreeBSD Xen 支持

与其他操作系统相比，FreeBSD 对 Xen 的支持加入得相对较晚。例如，NetBSD 是首个正式宣布支持 Xen PV 的操作系统，因为 Linux 完整的 PV 支持补丁直到 Linux 3.0（大约 2012 年）才合并。

FreeBSD 对 PV 有一些初步支持，但该移植版本仅支持 32 位且功能不完全。开发停止后，该版本在实现 PVH 支持时被删除。2010 年初，FreeBSD 增加了在作为 HVM 虚拟机运行时的 PV 优化，使 FreeBSD 可以利用 PV 设备进行 I/O，并使用一些额外的 PV 接口来加速，如 PV 定时器。

2014 年初，FreeBSD 获得了作为 PVHv1 虚拟机的支持，随后不久也支持作为 PVHv1 初始域。遗憾的是，PVH（也称为 PVHv1）的第一次实现设计错误，包含了过多与 PV 相关的限制。PVHv1 试图将经典 PV 虚拟机迁移到 Intel VMX 容器中运行。这是相当有限的，因为虚拟机仍然继承了许多来自经典 PV 的限制，且仅限于英特尔硬件。

在发现这些设计限制后，工作开始转向不同实现的 PVH。新方法从一个 HVM 虚拟机开始，并尽可能去除仿真，包括 QEMU 所做的所有仿真。大部分工作实际上是在 FreeBSD 上开发的，因为这是我的主要开发平台，我进行了大量工作，最终实现了后来称为 PVHv2，现在称为纯 PVH。

FreeBSD x86 既可以作为 HVM 虚拟机也可以作为 PVH 虚拟机运行，并支持作为 PVH dom0（初始域）运行。实际上，x86 PVH 支持比 Linux 更早地合并到 FreeBSD 中。然而，在 PVH 模式下运行仍然缺少一些功能，尤其是缺少 PCI 直通支持，然而，这需要对 FreeBSD 和 Xen 进行更改才能实现。目前，Xen 上游正在努力为 PVH dom0 添加 PCI 直通支持，但仍在进行中，完成后还需要修改 FreeBSD 来使该功能可用。

在 ARM 方面，正在进行工作以使 FreeBSD 能够作为 Aarch64 Xen 虚拟机运行。这需要将 FreeBSD 中的 Xen 代码拆分，分离架构特定的部分和通用部分。进一步的工作正在进行，以将 Xen 中断多路复用与 ARM 中的原生中断处理集成。

## Xen 社区的最新进展

除了之前提到的旨在使 x86 上 PV 和 PVH dom0 达到特性一致性的持续努力外，Xen 上游还在进行许多其他工作。自上一个 Xen 发布版本（4.19）以来，PVH dom0 已成为支持的操作模式，尽管由于一些关键功能仍然缺失，存在一些限制。

RISC-V 和 PowerPC 的移植工作正在进展中，希望在几个版本后它们能够达到一个功能完整的状态，届时初始域可以启动，并且可以创建虚拟机。

至少在 x86 上，近年来大量时间花费在缓解硬件安全漏洞方面。自 2018 年初 Meltdown 和 Spectre 攻击爆发以来，硬件漏洞的数量稳步增加。这需要 Xen 方面付出大量的工作和关注。虚拟机管理程序本身需要修复，以避免成为漏洞的目标，同时很可能需要暴露一些新的控制功能，让虚拟机自我保护。为了减轻未来硬件漏洞对 Xen 的影响，我们正在开发一个新功能，称为地址空间隔离（也被称为 Secret Free Xen），该功能旨在删除直接映射以及所有敏感映射，使其不再永久映射到虚拟机管理程序的地址空间。这将使 Xen 不再容易受到猜测执行攻击，从而可移除大量针对虚拟机管理程序入口点的缓解措施，并可能不再需要对未来的猜测性问题应用更多缓解措施。

自 2021 年初以来，所有 Xen 提交都通过 Cirrus CI 测试系统在 FreeBSD 上进行构建测试。这极大地帮助了 Xen 在 FreeBSD 上的构建，因为使用 Clang 和 LLVM 工具链有时会产生一些使用 GNU 工具链时不会显现的问题。我们目前测试 Xen 在所有受支持的 FreeBSD stable 分支上构建，并测试 HEAD 开发分支。Xen 最近停用了其自定义测试系统 osstest，现在完全依赖 Gitlab CI、Cirrus CI 和 Github actions 进行测试。这使得测试基础设施更加开放和易于文档化，也便于新贡献者参与并添加测试。未来该领域的工作应包括在 FreeBSD 上进行运行时测试，即便最初使用 QEMU 而非真实硬件平台。

最近的版本还添加了工具栈支持，用于向 Xen 虚拟机暴露 VirtIO 设备。Linux 和 QEMU 都支持使用 VirtIO 设备，通过授予而非使用虚拟机内存地址作为前端和后端之间内存共享的基础。这一新增功能并不需要修改 VirtIO 协议，而是作为一个新的传输层实现的。还在努力引入一种不基于内存共享的传输层，因为这是某些安全环境的要求。未来，这将让 Xen 在保持安全性和隔离性的同时使用 VirtIO 设备，这种安全性和隔离性是使用原生 Xen PV IO 设备时所保证的。整体目标是能够在 Xen 部署中将 VirtIO 驱动程序作为一等接口进行重用。

安全认证和采用 MISRA C 规则也是过去几个版本的主要任务之一。最后一个 Xen 版本（4.19）已扩展以支持符合 MISRA C 规范的 18 条指令和 182 条规则中的 7 条指令和 113 条规则。采用过程是逐步进行的，每条规则或指令都可以在被采纳之前进行讨论和达成一致。由于 Xen 的代码库最初并未考虑到 MISRA 合规性，某些规则将需要全局或每个实例的局部偏离。此外，作为安全认证计划的一部分，Xen 已开始添加安全要求和使用假设。安全要求提供了软件（Xen）所有预期行为的详细描述，从而能够进行独立测试和验证这些行为。

## Xen 的未来

回顾 x86 PVH 支持首次在 FreeBSD 上添加的时光，这是一段漫长且并不总是顺利的过程。FreeBSD 是早期采用 PVH 的 dom0 模式的操作系统，很多 Xen 开发工作是在使用 FreeBSD PVH dom0 的过程中进行的。值得注意的是，FreeBSD 在近年来已经成为 Xen 的一等公民，现在每次提交到 Xen 仓库的代码都会在 FreeBSD 上进行构建测试。

最近，FreeBSD 移植到作为 Xen Aarch64 虚拟机运行的工作也取得了一些进展，考虑到 ARM 基础平台在服务器、客户端和嵌入式环境中的日益普及，这无疑是一个值得期待的功能。

看到 Xen 被应用于如此多不同的使用场景，且这些场景与其最初专注于服务器端（云）虚拟化的设计目的大相径庭，令人欣慰。我只能希望未来会有更多新的 Xen 部署和使用案例。

## 如何联系

Xen 社区通过 [xen-devel 邮件列表](https://xenproject.org/resources/mailing-lists/) 进行代码审查。对于更为非正式的交流和讨论，我们还设有一些供所有人访问的 [Matrix 房间](https://xenproject.org/resources/matrix/)。对于 FreeBSD/Xen 特定的问题，还可以通过 [freebsd-xen 邮件列表](https://lists.freebsd.org/subscription/freebsd-xen) 进行咨询，当然，你还可以通过 [FreeBSD Bugzilla](https://bugs.freebsd.org/) 报告与 FreeBSD/Xen 相关的任何问题。

---

**Roger Pau Monné** 是 Cloud Software Group 的软件工程师和 FreeBSD 提交者。他在 Xen 社区中的角色包括 x86 维护者、Xen 安全团队成员以及 Xen 提交者。他在 Xen 和 FreeBSD 的 x86 PVH 实现方面做了大量工作，现在他大部分时间都在处理与安全相关的功能或追踪错误。
