# AArch64：成为 FreeBSD 新的一级架构

- 原文：[AArch64: Bringing a New FreeBSD Architecture to Tier-1](https://freebsdfoundation.org/wp-content/uploads/2023/06/maste.pdf)
- 作者：ED MASTE
- 译者：ykla【】为译者注

FreeBSD 起源于 386BSD，而且在最初仅支持一种 CPU 架构，即英特尔 80386。对第二种架构 DEC Alpha【是由迪吉多公司开发的 64 位 RISC 指令集架构微处理器】的支持在 FreeBSD 3.2 中被加入的，接着是对 64 位 x86（amd64）的支持。支持等级的概念尚未完全确定，但 amd64 在 2003 年被提升为一级支持状态。64 位的 ARM 架构 AArch64（也称为 arm64）在 2021 年获得了一级支持的地位。我们将探讨这意味着什么，以及我们是如何实现这一步的。

在 FreeBSD 中引入新的一级支持架构是份具有挑战性的任务，它需要大量的努力，以确保该架构得到充分支持、稳定、高性能，并且与现有的 FreeBSD 生态系统兼容。

**FreeBSD 基金会看到了 64 位 ARM 架构的潜力，并了解到其他开发者对在该平台上进行 FreeBSD 移植感兴趣。**

## 一级架构状态

在 FreeBSD 项目中，一级架构（Tier-1）状态是指完全支持的架构，二级架构是开发或小众架构，三级架构是实验性架构。在 FreeBSD 项目网站 <https://docs.freebsd.org/en/articles/committers-guide/#archs> 上有关支持等级的文件记录了这三个等级。

一级架构状态主要涉及 FreeBSD 项目对该架构的保证，包括生成 RELEASE 版本的软件包、提供预构建软件包、由安全团队提供支持，并在更新中保持向后兼容性目标。

一级架构还意味着该平台正在积极维护，定期进行测试，并及时修复错误和提供安全更新。预期 一级架构平台完全融入 FreeBSD 构建系统，以使工具链的所有组件都能正常工作，开发人员可以轻松地在该平台上构建、安装和维护操作系统。

一级架构状态还涉及些隐含特征，例如硬件可用性。FreeBSD 并未明确要求 一级架构平台必须广泛可用或受欢迎，但在实际应用中，一级架构状态要求多样化的硬件平台存在，并且以合理的成本可用。这是因为 FreeBSD 依赖于社区支持和厂商贡献来维护和改进对不同硬件平台的支持，并构建和测试第三方软件以适应该架构。

一级架构平台还应该是自举的，也就是说，在该平台上可以构建新版本的内核、C 运行时、用户空间工具和其他基本系统的 FreeBSD。

## 平台的起源

和其他几个平台一样，FreeBSD/arm64 起源于一位积极开发者的兴趣。Andrew Turner 是位长期从事 FreeBSD/arm 开发的开发者，在 Arm 宣布 AArch64 架构后不久就开始研究。FreeBSD 基金会看到了 64 位 Arm 的潜力，并了解到其他实体对在该平台上进行 FreeBSD 移植感兴趣。基金会组建了一个项目，协调并赞助了 Andrew Turner 和工程公司 Semihalf 的工作，同时得到了 Arm 和 CPU 供应商 Cavium 的支持。

在 FreeBSD 中最早与 arm64 相关的提交是为 kernel-toolchain 构建目标添加构建基础设施。顾名思义，该目标用于构建工具链（编译器、链接器等），然后用于编译、链接和转换内核。在当时，FreeBSD 的基本系统中包含了 Clang，所以编译器支持相对简单。然而，当时 FreeBSD 仍包含旧版本的 GNU ld 链接器，不支持 AArch64。因此，早期的构建支持依赖于安装了 aarch64-binutils port 或软件包，并自动使用提供的链接器。**第一个针对 arm64 的内核修改是：**

```
commit 412042e2aeb666395b3996808aff3a8e2273438f
Author: Andrew Turner <andrew@FreeBSD.org>
Date: Mon Mar 23 11:54:56 2015 +0000
Add the start of the arm64 machine headers. This is the subset
needed to start getting userland libraries building.
Reviewed by: imp
Sponsored by: The FreeBSD Foundation
```

经过几年的开发，FreeBSD 拥有了一个基本但功能齐全的自举的 FreeBSD/arm64 移植版本，提供了一些 port 和软件包。虽然仍需要大量的开发工作、调试、性能优化、文档编写和其他工作，但 FreeBSD 已经在将另一种架构添加到支持列表的道路上。FreeBSD 11.0 成为第一个包含 arm64 支持和可安装软件的版本，作为 二级架构平台。

## 工具链

**对 arm64/AArch64 支持的开始：**

```
commit 8daa81674ed800f568b87f5e4b8881d028c92aea
Author: Andrew Turner <andrew@FreeBSD.org>
Date: Thu Mar 19 13:53:47 2015 +0000
Start to import support for the AArch64 architecture from ARM. This
change only adds support for kernel-toolchain, however it is expected
further changes to add kernel and userland support will be committed
as they are reviewed.
As our copy of binutils is too old the devel/aarch64-binutils port needs
to be installed to pull in a linker.
To build either TARGET needs to be set to arm64, or TARGET_ARCH
set to aarch64. The latter is set so uname -p will return aarch64 as
existing third party software expects this.
Differential Revision: https://reviews.freebsd.org/D2005
Relnotes: Yes
Sponsored by: The FreeBSD Foundation
```

一级架构平台的首要要求之一是具备完全支持且融合的工具链。在构建 FreeBSD 时，Clang 是主要的编译器，并且来自几家大型公司的 AArch64 开发工作一直在进行，因此整个平台的编译器支持一直非常良好。

其他工具链组件，如链接器、调试器和各种二进制实用工具，需要更多的工作。在最初的 FreeBSD/arm64 移植工作接近完成时，FreeBSD 仍然使用 GNU binutils 链接器（"BFD 链接器"），由于许可证问题，已经有一段时间没有更新过链接器的版本。因此，基本系统中包含的链接器不支持 AArch64，最初的支持依赖于安装 port/软件包 binutils。我们尽可能地为最终用户提供了便利，但这并不符合 一级架构架构的要求。

幸运的是，LLVM 社区的 LLD 链接器也在迅速取得进展。LLD 提供了更短的链接时间，支持不同于 BFD 链接器的优化，并拥有更广泛的架构支持。到了 2016 年底，我们在 FreeBSD/arm64 上成功切换到 LLD 作为系统链接器。事实上，这是 FreeBSD 上首个采用 LLD 链接器的架构。

FreeBSD 使用 ELF Tool Chain 项目来提供一些杂项二进制实用工具，如 strings 和 strip。这些工具有一些与机器相关的功能（例如重定位类型列表），但添加 arm64 支持的工作相对较轻松。

最后一个需要大量开发工作的工具链组件是 LLDB，即 LLVM 家族的调试器。幸运的是，LLDB 的开发工作也在为其他操作系统提供支持，因此只需要逐步增加对 FreeBSD 的支持。我们成功地将这些工具链工作合并到 FreeBSD 11 的稳定分支中，而 FreeBSD 11.1 成为了第一个避免使用变通方法并包含功能性链接器的版本。

## Ports 和软件包

FreeBSD 提供了 30,000 多款第三方软件包在其 ports 中，并且其中许多软件包具有与架构相关的特性。机器相关的基础设施（例如，对于给定 port 在给定架构上进行编译的控制）是 ports 的基本部分。FreeBSD/arm64 作为 二级架构架构可用，FreeBSD 社区成员进行了实验并发现了无法构建的 port。这些问题要么被修复，要么在适当情况下从 aarch64 架构上排除。

将 FreeBSD/arm64 带入 一级架构阶段的目标带来了 ports 的一些额外要求。ports 没有官方的层次结构或 port 的分级分类，但有一些关键的 port。这些关键 port 提供工具链组件或其他依赖项，这些依赖项对于构建整个 ports 中的大型 port 是必需的。我们必须确保这些关键 port 在 arm64 上工作正常，并满足 一级架构的要求。确保这些软件包在 FreeBSD/arm64 上可用且可以持续构建是必要的。我们还需要及时为 一级架构架构构建软件包，这需要具备能力的服务器硬件。FreeBSD 基金会从 Ampere Computing 购买了服务器，并且该项目还收到了 Ampere 捐赠的额外服务器。这些硬件使得 arm64 软件包可以按照 x86 架构的每周频率进行构建。

## FreeBSD 团队的支持

**将新的架构提升到 一级架构状态需要得到 FreeBSD 项目内的几个团队的支持和同意。**

将新的架构带入 一级架构状态需要得到 FreeBSD 项目内几个团队的支持和认可。这包括如上所述的 ports 管理和软件包管理团队，以及安全团队、发布工程团队和核心团队。

发布工程团队负责构建和测试发布制品，包括 ISO 和 USB 存储设备镜像，以及云计算目标。这些制品可以从其他架构进行交叉构建，因此发布工程团队并不绝对需要 arm64 构建主机，但是需要测试和 QA 硬件。

成为 一级架构架构需要安全团队提供安全问题和勘误的源代码更新，以及通过 freebsd-update 提供二进制更新。

最后，核心团队的支持是必要的，以便与其他团队、社区协调，并正式宣布该平台为官方的一级架构。

## 硬件生态系统

硬件生态系统是成为 一级架构平台的一个隐含要求，正如前面提到的。硬件在许多不同的价格/性能组合下需要提供支持：

- 高端服务器用于构建软件包，
- 中档的服务器级硬件用于开发者工作站、远程访问进行移植和测试等，
- 低端嵌入式平台用于广泛的测试和开发者使用，
- 云资源用于开发、测试和生产的不同层次。

AArch64 最初在可用硬件方面存在一些明显的缺口，特别是与中端（和中等价格）的开发者和移植工作相关的平台。在 2010 年代末期，可选项非常有限。SoftIron OverDrive 1000 是一款价格合理、性能良好的系统，采用 AMD A1100 处理器，并且采用便捷的开发者形式因子。不幸的是，A1100 和 OverDrive 1000 在推出后不久即停产。

随着硬件可用性的不断提高，平台选择也在增加，例如树莓派 4 和 Pine A64-LTS 等低端平台，苹果设备和微软的 AArch64 开发者平台等中端平台，以及基于 Ampere Altra 的高端服务器系统。主要云服务提供商也提供 AArch64 虚拟机，其中使用 Ampere 平台或定制的 CPU 设计（如 AWS 的 Graviton）。

将 FreeBSD/arm64 带入 一级架构状态需要大量的时间和资源投入。64 位 Arm 生态系统在服务器市场上占据了重要的份额，并且没有放缓的迹象。FreeBSD 将通过这个 一级架构平台从服务器市场中获益。

![image](https://github.com/FreeBSD-Ask/freebsd-journal-cn/assets/10327999/6dc86389-911d-4fa3-9982-8451be051c82)

AWS Graviton

![image](https://github.com/FreeBSD-Ask/freebsd-journal-cn/assets/10327999/79e6732f-9b84-4d9c-adb5-c2b782506da2)

Ampere “Mount Jade”

![image](https://github.com/FreeBSD-Ask/freebsd-journal-cn/assets/10327999/4721725a-d91b-479b-b3d8-30cffcb9a44a)

树莓派 4

![image](https://github.com/FreeBSD-Ask/freebsd-journal-cn/assets/10327999/57263b2a-c50f-4583-8186-ec07baedd486)

微软 Arm 开发套件

![image](https://github.com/FreeBSD-Ask/freebsd-journal-cn/assets/10327999/23ac923c-ed09-45d1-85b8-1d9f28d1b5fd)

Pine A64 LTS

---

**ED MASTE** 是 FreeBSD 基金会的高级技术总监，负责管理基金会的技术路线图、开发团队和赞助项目。他还是当期选举产生的 FreeBSD 核心团队成员。除了 FreeBSD，他还为许多其他开源项目做出了贡献，包括 LLVM、ELF Tool Chain、QEMU 和 Open vSwitch。他与妻子 Anna 和孩子们一起生活在加拿大的基奇纳 - 滑铁卢地区。
