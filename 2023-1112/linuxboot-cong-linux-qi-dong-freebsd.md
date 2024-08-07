# LinuxBoot： 从 Linux 启动 FreeBSD

作者 WARNER LOSH 

## 如何到达这里

三个主题导致 LinuxBoot 变得流行：最初的简单性，不受控制的增长，以及对回归更简单时光的渴望。这三个主题都导致了在 x86 和嵌入式系统上出现的复杂引导生态系统。LinuxBoot 试图在一定程度上简化这些生态系统，尽管通常将整个 Linux 内核卷入其中并不会立刻让人联想到简单性。

在 IBM PC 出现之前的日子里，大多数系统要么需要手动输入引导程序，要么自动引导 ROM 足够简单，可以加载引导扇区，然后一起加载系统的其余部分。使用简单的加载程序逐步加载越来越复杂的加载程序的过程称为“引导”系统，来源于古语“拔起自己的靴带”。而随着时间的推移，这一过程被简化为“引导”系统。

1982 年，IBM 发布了 IBM PC，仅在先前系统的基础上略有改进，提供了其 ROM 中的引导代码和其他基本 I/O。它将这些服务称为 BIOS，这个术语是在它之前的 CP/M 系统中使用的。这就是我们得到“BIOS”这个术语以及为什么围绕它的含义存在一些混淆的原因。支持这两种观点的人都确信自己是对的，但很少有人认识到这种模糊背后的历史。因此，我将使用“CSM”或“CSM 引导”来指代这种引导样式。“UEFI 标准”使用术语“CSM”来描述传统的引导，并且是明确的。

随着时间的推移，各种其他功能被添加到 CSM 引导中：用于磁盘的分区表，用于处理器配置的 MP Table，用于电源管理的 APM，用于系统元数据的 SMBIOS，用于 PCI 的运行时服务，用于 PXE 网络引导以及用于统一以前功能的 ACPI。这些服务的接口与 x86 特定机制相关联。系统演变成一个非常复杂的生态系统，充斥着快速解决方案，特殊情况和细微不同的解释。

当英特尔在 1990 年代末设计 IA-64 CPU 架构时，很快发现这种革命性架构无法使用 CSM 生态系统中的绝大部分技术。整个引导生态系统必须被替换。这就是统一可扩展固件接口（UEFI）引导的起源。起初，它只在英特尔 x86（重新品牌为 IA-32）和 IA-64 架构上。在 2000 年代初，UEFI 固件逐渐取代了英特尔 x86 系统上的传统系统，通常能够执行旧的 CSM 引导或新的 UEFI 引导。到 2006 年，英特尔通过其 TianoCore 项目发布了 UEFI 固件创建的 EDK2 开源开发工具包，促使更多的 OEM 厂商采用 UEFI。

也在 1990 年代和 2000 年代期间，另一个嵌入式系统引导生态系统正在演变。最初，嵌入式空间拥有数十种不同的引导加载程序，它们在引导后续阶段时略有不同。Mangus Damm 和 Wolfgang Denk 在 1999 年创建了 Das U-Boot，最初为 PowerPC，后来支持 ARM、MIPS 和其他架构。U-Boot 起初简单灵活，比竞争对手更易于扩展，因为它是开源的。由于其简单性、支持和丰富的功能集，它迅速成为了通用引导加载程序。它为引导设置了标准，并推动了 Linux 中许多功能的发展，包括扁平设备树（FDT）支持。这种引导系统与 CSM 或 UEFI 完全不同。它如此易于使用，以至于它的所有竞争对手最终都黯然失色，至多只是维基百科页面上的注脚。

2011 年，ARM 推出了 aarch64，这是其 ARM 平台的 64 位版本。U-Boot 和 EDK2 竞相主导系统引导过程，而 FDT 和 ACPI 则竞相枚举系统设备。低端嵌入式系统倾向于使用带有 FDT 的 U-Boot，而高端服务器级系统则使用 UEFI 和 ACPI。最终，UEFI 引导开始占据上风，特别是当 U-Boot 开始提供了一个最小的 UEFI 实现，足以通过 UEFI 引导 Linux。ACPI 和 FDT 合并（现在可以使用 FDT 属性指定 ACPI 节点）。在此过程中，EDK2/UEFI 变得越来越复杂，以支持 SecureBoot、iSCSI、更多网卡、RAM 磁盘支持、initramfs 支持以及太多其他功能以至无法一一列举。

那甚至没有算上所有使用剥离版本的 Linux 内核的引导加载程序，比如 coreboot、slimboot、LinuxBIOS 等等，其中一些我将在下面描述。商业 BIOS 之间的差异也没有开始描述。到了 2017 年，谷歌决定采取行动，开始了 NERF 项目以简化这一混乱并加强安全性。该名称代表 Non-Extensible Reduced Firmware，与 UEFI 的 Unified Extensible Firmware Interface 相对。这也是来自电脑游戏术语，用于指改变游戏元素的权力或影响以获得更好的平衡或提高游戏的乐趣。这些努力后来演变为 LinuxBoot。

## Linux 引导

使用 Linux 引导 Linux 有着悠久的历史，但空间只允许简要总结。在上世纪 90 年代中期，当 kexec(2)系统调用族被添加到 Linux 中时，它被用来提高服务器和嵌入式系统的正常运行时间和/或可靠性。Ron Minnich 和 Eric Biederman 于上世纪 90 年代在洛斯阿拉莫斯创立了 LinuxBIOS，将 Linux 内核用于固件来引导系统。这演变成了 coreboot，被 Chromebook 和几台开放平台笔记本电脑使用。在这个过程中，coreboot 变得模块化，以允许二进制块并存于开源组件旁边，因为 CPU 制造商一直抵制开放早期处理器初始化代码，仅为开源和闭源固件创作者提供二进制块。EDK2、U-Boot 和闭源固件也已经开发出了一个模块化系统，以允许这些二进制块与其他组件共存。

## 项目 NERF 变成了 LinuxBoot

Google 的 NERF 项目，由 Ron Minnich 领导，演变成了 LinuxBoot 项目：一系列脚本，帮助创建引导最终操作系统带有 Linux 内核的固件镜像。然而，该项目背后有几个更大的目标。Google 想要创建一个开源固件，其中每个组件都是免费提供的。他们想要简化 UEFI 引导环境，他们觉得已经变得过于复杂，并存在太多潜在的安全漏洞，它们用强化的 Linux 内核来替换它。他们想要创建一个使用广泛部署和审查的代码的通用框架；最小化引导加载程序中不可避免的仅二进制、非源代码部分；尽可能统一 ARM 和其他嵌入式系统的引导；消除冗余代码，加快引导速度；并创建比传统固件甚至 EDK2 提供的更模块化和可定制化的引导体验。他们想要一个可复制的构建，确保任何人无论是下载还是自行构建都能运行完全相同的二进制文件。

结果是一个支持许多引导加载程序的模块化系统。CPU 初始化的最早阶段由特定于 CPU 和引导加载程序的代码处理。LinuxBoot 定义了系统哪些部分在那里初始化，哪些部分推迟到 Linux 内核。这个设置使 CPU 供应商可以继续提供初始化低级时钟、存储器控制器、辅助核心等现代 CPU 需要变得可用的仅二进制 blob。EDK2、coreboot、U-Boot 和 slim boot 都支持这些协议，所以 Linux 内核与所有这些引导加载程序一起启动，而不需要任何特殊代码。LinuxBoot 还提供了 u-root，这是一个用 Go 编写的 ramfs 构建工具，用于查找和加载最终固件及其他几个工具来操作固件镜像。我将在本文的第二部分中讨论这些工具以及如何使用它们。

![](https://freebsdfoundation.org/wp-content/uploads/2024/01/losh_graphic.jpg)

源自：https://www.linuxboot.org

虽然未能完全用 Linux 替换整个引导加载程序，LinuxBoot 最小化了保留量。例如，在 UEFI 中，只保留了预-EFI 初始化（PEI）阶段来初始化处理器、缓存和 RAM，以及 UEFI 的运行时服务。

LinuxBoot 消除了所有薄弱测试的 UEFI DEX 驱动程序。Linux 内核接管了已初始化的内存和基础硬件，但没有任何传统固件可能会初始化的其他内容，比如用于 PCI 设备的资源。

除了安全性更好和对固件有更多控制之外，LinuxBoot 使用了在平台上运行 Linux 所需的经过充分测试、经过高度审查的 Linux 驱动程序。有了 LinuxBoot，SOC 供应商和系统集成商可以通过仅为 Linux 编写驱动程序来优化其上市时间。根本不需要创建 UEFI DEX 驱动程序。拥有 Linux 驱动程序技能的程序员比能编写 UEFI DEX 驱动程序的程序员更容易找到。Linux 内核经过了数千名研究人员的审计，而对 EDK2 UEFI 代码库进行研究的研究人员相对较少。然而，这些优势需要其他支持 UEFI 的操作系统进行适应。它们的 UEFI 引导加载程序与剩余的 UEFI 不兼容。这意味着要在这些系统上启动，操作系统必须创建一个新的加载程序来支持 LinuxBoot。

一些较简单的操作系统使用 LinuxBoot 通过 Linux 的 kexec-tools 包提供的基本 ELF 加载进行引导。非常旧版本的 BSD 和 Plan9 就是这样引导的。运行在简化了的时间轴上，具有明确定义的 OpenFirmware 接口的处理器的 FreeBSD/powerpc 也是通过这种方式加载的。然而，Windows 无法通过这种方式引导，LinuxBoot 社区正在研究解决办法。FreeBSD/amd64 和 FreeBSD/aarch64 也无法通过这种方式引导。

FreeBSD 内核适用于 amd64 和 aarch64，但需要引导加载程序提供的元数据。在 amd64 上，引导加载程序在捕获系统内存布局等信息后，将系统设置为长模式，而这些信息只能在进入长模式之前访问。内核依赖于这些数据，没有这些数据无法运行。在 amd64 和 aarch64 上，引导加载程序必须告知内核 UEFI 系统表的地址和其他系统数据。引导加载程序通过设置“可调整项”来调优内核。加载程序预装载动态内核模块，并将初始熵、UUID 等传递给内核。而 kexec-tools 只能加载 ELF 二进制并跳转到其起始地址，不包含这些专门的知识。

## FreeBSD 和 LinuxBoot

FreeBSD 通过 Linux 引导的历史可以追溯到十多年前。2010 年，FreeBSD PS/3 使用 PS/3 的“另一个 OS”选项启动。FreeBSD 开发者 Nathan Whitehorn 在 FreeBSD 引导加载程序中添加了必要的代码，为 FreeBSD 内核设置内存。他创建了一个类似于 Ubuntu PS/3 支持包中 kboot 的小型 Linux 二进制文件。这个 Linux 二进制文件是静态链接的，包含少量系统调用，用于从 PS/3 磁盘读取 FreeBSD 内核。它包含一个类似于 mucl 或 glibc 的小型 libc 以及命令行解析支持。不过，它的源代码结构假定只支持 PowerPC。

在 Netflix 工作时，我开始在 2020 年进行实验，看看用 Linux 启动 FreeBSD 会有多难。Netflix 在世界各地安装了大量服务器。经过多年不断的改进，Netflix 创建了一个非常健壮的系统，可以自动纠正常见问题。即使经过这些改进，启动问题也导致了太多昂贵的 RMAs。使用 UEFI 脚本进行改进启动时间可靠性的实验仅带来了轻微的改善。因为闪存驱动器包含了这些脚本，只有少数微不足道的情况得到改善。闪存驱动器可能发生只读错误，使得 UEFI 固件和脚本混淆，做出了错误的操作。只要这些脚本保留在驱动器上，进展就是不可能的。

LinuxBoot 为 UEFI 提供了一种有吸引力的替代方案，因为它位于主板固件内部，消除了最容易出现故障的组件。Netflix 希望我创建一个容错环境，可以向家中发送有关机器状态信息，使用幸存的 NVMe 驱动器重新配置机器，并提供一个灵活的平台，以启用远程调试、诊断图像等。

我在从 Linux 启动 FreeBSD 时有几个目标：

1. 它必须在 FreeBSD 构建系统内构建。
2. 它必须提供对主机资源的完全访问权限。
3. 它必须能够使用原始内核启动（如果可能的话）。
4. 它必须使用 UEFI 启动接口（不支持 i386 CMS 启动和 arm U-Boot 二进制启动）。
5. 它必须作为 init/PID 1 运行。
6. 当从shell脚本调用时，它还必须能够良好运行，以支持引导不基于 FreeBSD 的不同类型的镜像。

从 Linux 启动 FreeBSD 需要对相对简单的 PS/3 kboot 基础进行多次更改，尤其是在现代架构如 amd64 和 aarch64 上。加载器需要四种类型的更改：按照 MI/MD 模型重构现有的 kboot，扩展对访问主机资源的支持，重构 UEFI 引导代码以供 UEFI loader.efi 和 LinuxBoot loader.kboot 使用，以及清理引导加载程序中的技术债务。

## MI/MD 变化

几个领域需要经典的 MI/MD 分离，其中通用的 MI 代码与每个架构的 MD 代码接口，这些 MD 代码实现了一个通用的 API。Linux 在架构之间的系统调用差异比 FreeBSD 大得多。程序启动需要在架构之间略有不同的汇编粘合。需要不同的链接脚本。加载器元数据虽然大体相似，但在架构上有所不同。最后，从 Linux kexec 重启向量到内核的切换也有所不同。在下面的“重构 UEFI 启动”部分，我将介绍这最后两点。

这些更改的前三项是为了创建 Linux 二进制文件所需的。为了创建静态二进制文件，我编写了 C 运行时支持，提供了 Linux 内核交接和传统主程序之间的接口。我编写了一些针对每个架构的汇编代码，结合了一个调用 main 函数的标准启动例程。为了实现这一点，系统调用的标准 C 接口允许 Linux 的 mini libc 的 MD 部分变得很小。我为系统调用编写了少量针对每个架构的汇编代码。我增加了一个框架来处理 Linux 在每个架构上 ABI 差异的问题，其中最大的差异在于 termios 接口。这反映了 Linux 在二进制兼容性方面复杂的历史。一个针对每个架构的链接脚本生成了一个 Linux ELF 二进制文件。这些元素结合在一起形成了 loader.kboot，Linux ELF 二进制文件。新的 libsa 驱动程序（见下文）与这个 libc 接口。

## 访问主机资源

最初的 loader.kboot 代码访问了一些主机资源，但是它是不完整的。我想要从原始设备引导，或者通过主机系统文件系统中的内核或加载程序引导。引导加载程序一直支持一些不同的指定文件来源的方式，但在重构之前，添加新的方式是困难的。通过对现有代码进行一些更改重构，我添加了访问任何块设备的能力，可以通过其 Linux 名称访问。例如，“/dev/sda4:/boot/loader”这个名称读取了 sda 磁盘上第四个分区内的文件/boot/loader。此外，“lsdev”现在列出所有合格的 Linux 块设备。引导加载程序可以发现 zpools。例如，“zfs:zroot/kboot-example/boot/kernel”指定要引导的内核。最后，让内核和/或加载程序直接放在 Linux initrd 中可能会很方便。引导加载程序本身使用此功能从/sys 和/proc 文件系统获取必要的数据。可以使用“host:‹path-to-file›”访问任何已挂载的文件系统。因此，您可以从“host:/freebsd/boot/kernel”引导，或者使用“more host:/proc/iomem”读取 Linux 内存使用情况。加载程序还支持将“/sys/”或“/proc/”前缀映射到主机的/sys 和/proc 文件系统，而不考虑活动设备。

Loader.kboot 可以替换 Linux initrd 中的/sbin/init。Init 是第一个要运行的程序，必须执行额外的步骤来准备系统。当 loader.kboot 作为 init 运行时，它将在启动之前执行这些额外步骤：挂载所有初始文件系统(/dev、/sys、/proc、/tmp、/var)，创建一些期望的符号链接，以及打开 stdin、stdout 和 stderr。加载程序可以在这个环境中运行，也可以作为从标准 Linux 启动脚本之一启动的进程运行。目前，loader.kboot 无法 fork 和执行 Linux 命令。

## 重构 UEFI 引导

镜像掩盖了 FreeBSD 的通用引导长历史，特别是 amd64 这种情况已经有 30 年之久，而 aarch64 则大约 20 年左右。当然，这些架构并非始终存在，但 amd64 继承了许多 i386 的特殊之处，而 aarch64 的引导则经过了 20 年嵌入式 FreeBSD 系统的演变，因此显得更为清晰。在这种复杂环境下成功引导，loader.kboot 需要重新创建所有这些怪癖。它遵循 UEFI 协议，创建与我们的 UEFI 引导加载程序 loader.efi 相同的元数据结构。

这些工作始于 amd64，因为它更容易进行实验。我选择了 UEFI + ACPI 引导环境来进行模拟。UEFI 是更新、更灵活的接口，似乎没有那么多特例。理论上，FreeBSD 内核可以从 UEFI 或 CSM 引导，而不知道从哪个引导，但实际情况并非如此。内核期望以某种方式获取来自 UEFI 的数据，以稍有不同的方式获取来自 BIOS 的数据。很早就清楚，试图同时支持两者会限制进展，因为通常需要编写和调试两条不同的路径。由于 UEFI 将长期存在（即使只有微小部分通过 LinuxBoot 存活），而 CSM 可能不会，我决定仅支持 amd64 上的 UEFI。

尽管简化了，但进展仍然太慢，因为我不得不通过试错来发现 amd64 内核依赖于引导加载程序的所有怪癖。FreeBSD 开发者 Mark Johnston 建议尝试使 aarch64 正常工作，因为其接口更简单。这证明是正确的。一旦我将 UEFI 数据结构基本转换为 FreeBSD 加载程序的元数据，aarch64 的引导进展了许多。只有一些 bug，稍后会讨论。现在计划使 amd64 正常工作，因为 aarch64 已经可以运行了。

加载程序需要数百行代码来设置来自 UEFI 的元数据。这段代码预期在 UEFI 运行时环境中运行，不足为奇地使用 UEFI API 分配内存，从 UEFI 获取内存信息，并以特定于 UEFI 的方式获取 ACPI 表。我需要重构此代码，以便它可以从 Linux 的 /sys 和 /proc 文件系统中创建正确的元数据结构。此外，Linux 提供 FDT 和 ACPI 数据，并且仅在 ACPI 中提供设备描述。这使 FreeBSD 误认为没有设备存在，因为当两者同时存在时，它倾向于使用 FDT 进行设备枚举。Linux 仅提供执行另一个 kexec 所需的数据，但没有设备数据。

除了常规的 UEFI 数据结构和引导之外，在 kexec 之后，Linux 将硬件保持在与从固件进行的冷启动或热启动略有不同的状态。因此，系统并未完全重置。通常情况下这并不重要——我们可以从该状态引导，并将所有硬件置于正确的状态。但确实存在问题。

UEFI 引导服务是我的第一个问题。当 Linux 退出 UEFI 的“引导服务”时，它会创建内存映射，其中虚拟地址（VA）与物理地址（PA）不匹配。FreeBSD 的 loader.efi 总是创建 1:1 映射，其中 PA 等于 VA（即所谓的 PA = VA）。由于内存映射可能只能设置一次，FreeBSD 的内核必须使用 Linux 创建的映射。内核会因为映射不是 PA = VA 而发生恐慌。幸运的是，恐慌是由于调试 loader.efi 留下的限制性断言导致的。移除这些断言后，暴露出一个 bug，即使用 PA 而不是 VA，但在我修复了那个 bug 之后内核启动了。

我遇到的第二个问题是“gicv3”问题，其中“几乎重置”与设备勘误结合起来就会导致大麻烦。gicv3 中断路由器存在设计缺陷：一旦启动（Linux 在启动时会完全启动它），除非进行完整的系统重置和初始化（kexec 无法做到），否则无法停止它。为了解决这个问题，FreeBSD 内核必须重用这块内存。Linux 通过 UEFI 系统表结构传递 gicv3 状态数据。该表包含一系列保留给 gicv3 使用的物理地址。FreeBSD 解析该表，确保它与 gicv3 正在使用的内容匹配，并将其标记为保留，因此 FreeBSD 的内存分配代码不会分发这些地址。一切在 QEMU 中都正常工作；然而，当我们尝试在 aarch64 机器上运行时，惊讶地发现了这个问题。幸运的是，Linux 社区先前已经发现了这个问题，并提供了一套补丁，我可以借鉴类似的方式来修复 FreeBSD。

## 淘汰技术债务

在这个项目中我发现的一个不太令人惊讶的事情是，引导加载程序中存在大量复制粘贴的代码来实现路径和设备名称解析。这些代码的副本在非明显的方式上有所不同。有时候更改是修复了 bug，但软件考古显示其他副本保留了 bug。其他时候，在副本中引入了新的 bug。可以理解这将是加载程序的命运。在移植到新平台时，很容易只需从工作的加载程序中复制代码，稍微调整一下以适应新环境。很少有人考虑缺乏重构的长期影响。一旦加载程序能够引导内核，为什么还要在加载程序上花更多时间呢？结果表明，这种策略和这些态度是有害的。例如，文件名解析代码在我开始时已经从一个环境复制到另一个环境，所以在我开始时大约有 10 个副本。它们都需要一个通用例程——嗯，除了一个，因为它的设备规范与其他地方使用的“diskXpY：”不同，有一个合理的原因导致它不同。解析中的 bug 让我将所有这些代码重构到一个位置（这样我就可以一次修复 bug）。这使得在访问原始设备时，“/dev/XXXX” 的新用法可以工作作为“设备名称”。它还让“host” 前缀可以在不需要单元编号的情况下工作。加载程序的文件名解析器比我以往所知道的要灵活得多。

## 结论

适应 FreeBSD 引导程序和内核以在这个新环境中引导的过程证明是直接的。 Linux 主机集成的大量小任务，加上 FreeBSD 未记录的引导程序到内核的交接，是本阶段项目面临的最大挑战。意外的硬件缺陷增加了激动情绪，并导致内核需要的最大更改。随着 FreeBSD/aarch64 成功在真实硬件上启动，我们可以继续项目的下一阶段：创建我们自己的固件并发现剩余的 FreeBSD/amd64 缺陷。尽管存在这些限制，我们已经使用 loader.kboot 下载安装程序 RAM 磁盘来配置系统并重新启动结果。它还被整合到去年夏季的 loader 持续集成 GSoC 项目中。

## 接下来

在下一篇文章中，我们将创建一个 LinuxBoot 固件镜像，用于引导 FreeBSD。我将解释如何打包固件镜像，用于创建和操作它们的工具，以及如果您足够勇敢，如何重新刷新您的固件镜像。我将帮助您从 Linux 提供的众多选项中选择适合创建 initrd 的正确工具，并提供一个示例脚本来查找并引导您的 FreeBSD 系统。带着好运，最终您将拥有一个更简单、更快速、更安全的固件。

WARNER LOSH 多年来一直为 FreeBSD 项目做出贡献。他为引导程序贡献了许多功能和修复。他对 Unix 历史的兴趣延伸到引导程序是如何随着 Unix 及其许多衍生版本的演变而发展的。他与爱画猫和狗的妻子林迪以及喜欢演奏各种铜管乐器的女儿一起居住在科罗拉多州。他经常带着腊肠犬散步。

—2023 年 11 月/12 月《FreeBSD Journal》
