# CheriBSD 近十多年的历程

- 原文：[A Dozen Years of CheriBSD](https://freebsdfoundation.org/wp-content/uploads/2023/06/davis_cambridge.pdf)
- 作者：BROOKS DAVIS
- 译者：ykla

自 2010 年末以来，剑桥大学和 SRI International 的 CHERI 研究项目一直致力于开发、展示和推广提供内存安全性和高效分区功能的体系结构扩展。我们的 CHERIBSD 是 FreeBSD 的一个增强版本，是我们工作中最重要的成果之一。将 FreeBSD 适应 CHERI 的支持，不仅为我们的体系结构变更提供了指导，同时也证明了我们的想法在大型现代操作系统的规模上是可行的。

## 对 CHERI 的简要介绍

CHERI 通过引入一种新的硬件类型，即 CHERI 能力（Capability），扩展了现有体系结构（Armv8-A、MIPS64（已停用）、RISC-V 和 x86_64（正在开发中）。在 CHERI 系统中，所有对内存的访问都是通过 CHERI 能力进行的，可以通过新的指令明确地访问，也可以通过用于整数参数指令的默认数据能力（DDC）和程序计数器能力（PCC）隐式地进行访问。能力授予对特定范围的（虚拟的，或者偶尔是物理的）内存的访问权限，通过基址和长度来表示，并且可以通过权限进一步限制访问，这些权限被压缩成 128 位的表示（地址占 64 位，元数据占 64 位）。在内存和寄存器中，能力由标签进行保护，当非能力指令修改能力数据或能力指令增加能力授予的访问权限时，标签将被清除。标签与数据分开存储，不能直接操作。

![68 7L@D UVUS4F}2LFFD0ZF](https://github.com/FreeBSD-Ask/freebsd-journal-cn/assets/10327999/91a51da3-2408-4d15-bd80-bf328bda911a)

我们最初的 CHERI 工作是作为 DARPA CRASH 项目的一部分，扩展了 MIPS64 体系结构。2014 年，我们开始与 Arm 合作，探索将 CHERI 兼容到 Armv8-A 体系结构的可能性。2017 年，我们开始将 CHERI 移植到 RISC-V，这一工作得到了我们在 MIPS 方面的经验和与 Arm 的合作的启示。这一移植工作是作为 DARPA MTO SSITH 项目的一部分进行的。我们与 Arm 的合作在 2019 年公开，宣布了 1.9 亿英镑的“按设计进行数字安全”（Digital Security by Design）计划，该计划产生了 Morello 体系结构原型，这是一个基于 Neoverse N1 核心的 SoC，用于云平台，如亚马逊网络服务（Amazon Web Services）的 Graviton 节点。

我们设计了 CHERI 能力，使其适用于 C 和 C++ 语言的指针，并修改了 Clang 编译器，以支持两种模式。在混合模式下，使用 \_capability 注释的指针是能力（capabilities），而其他指针保持为整数。在纯能力模式下，所有指针都是能力，包括堆栈上的隐式指针，如返回地址。通过与内核支持以及对 C 启动代码、运行时链接器和标准库的适度更改，我们创建了一个内存安全的 C/C++ 运行时环境，称为 CheriABI1。对这个环境的完善是我们在 CheriBSD 上的关键工作，同时还包括创建纯能力内核环境以及对时间内存安全和分区化的探索。

除了内存安全性，CHERI 还实现了细粒度的分区化。由于所有内存访问都是通过能力来进行的，因此给定线程可以访问的地址空间部分由其寄存器集和从寄存器集中可达（transitively）的内存定义。通过适当的机制来在寄存器集之间进行转换，我们可以在不同的分区之间快速切换。不同的 CHERI 实现采用不同的机制；哪种机制最适合商业实现仍是一个活跃的研究课题。

## 什么是 CheriBSD

CheriBSD 是对 FreeBSD 进行了修改以支持 CHERI。但这实际上意味着什么呢？

当内核编译为 CHERI 时，默认的 ABI 是纯能力 ABI（CheriABI），其中包括系统调用参数在内的所有指针都是能力。我们还通过从 freebsd32 32 位兼容性层派生的 freebsd64 ABI 兼容性层，支持混合二进制和标准的 FreeBSD 二进制。同样，我们默认构建库、程序和运行时链接器为 CheriABI，并为混合二进制构建库，这些库被安装在/usr/lib64，类似于 freebsd32 的/usr/lib32。所有这些意味着默认情况下，用户将获得一个内存安全的 Unix 用户空间，同时保留运行未修改的 FreeBSD 二进制文件的能力。

内核可以编译为混合或纯能力程序。这增加了我们需要进行的一些复杂性（对于混合模式，每个指向用户空间的指针都需要一个注释（\_capability）），但在项目初期我们没有强大的 C 编译器支持，因此我们从混合模式开始，并且由于指针大小增加，纯能力内核的固有开销确实略高。所有内部内核开发都考虑到了纯能力支持。这项工作包括确保所有对用户空间的访问都通过一个能力（capability），对虚拟内存系统进行更改以在分配内存时创建能力，以及修改设备驱动程序，包括 DRM GPU 框架，以使用能力。

在过去，CheriBSD 主要是一个从源代码编译的项目。对于 FreeBSD 开发人员来说，这是熟悉的，并且有很多好处；然而，对于只想将自定义代码库移植到 CHERI 的人来说，这是一个很大的障碍。随着 Arm 的 Morello 原型的发布，我们开始发布完整的版本，包括安装程序和软件包。我们使用一个轻度定制的 FreeBSD 安装程序，添加了支持安装基于 KDE 的 GUI 桌面环境，并删除了一些我们认为令人困惑的对话框。GUI 环境由我们的 FreeBSD ports 分支构建的软件包组成。因为并不是所有软件都被移植到 CHERI，我们构建了两套软件包，并构建和安装了两个版本的 pkg 命令，其中 pkg 是一个脚本，将调用者重定向到其他名称。

有一个 CheriABI 软件包集，由 pkg64c 命令管理，并安装在/usr/local 目录下，还有一个混合软件包集，由 pkg64 命令管理，并安装在 `/usr/local64` 目录下。

大部分桌面环境是 CheriABI 二进制文件，唯一的例外是 Web 浏览器（正在进行一个 CheriABI 版本的 Chromium 移植）。在安装后，混合软件包对于安装尚未移植的软件，如 emacs 和 Morello LLVM，也很有用。

![image](https://github.com/FreeBSD-Ask/freebsd-journal-cn/assets/10327999/8e01e811-efed-4ef0-9332-9d15b5dc1d39)

除了内存安全外，CheriBSD 还承载了我们关于软件隔离的大部分研究工作。在 MIPS 时代，我们实现了一个隔离框架（libcheri），并将其应用于整合版的 tcpdump。虽然我们没有将这项工作移植到 RISC-V 和 Morello，但它对我们早期思考提高可用性的隔离化的方式产生了影响。我们最新的发布包含了一个库隔离模型，在这个模型中，动态链接库在其自己的沙盒中运行。目前的实现是实验性的，但在隔离化几乎没有修改的程序方面显示出了相当大的潜力。此外，在开发分支的堆栈中，我们还有一个共处理隔离模型，在这个模型中，多个进程共享相同的虚拟地址空间，依赖 CHERI 提供内存隔离。结合一个可信切换器组件，这使得从一个进程的线程到另一个进程的线程的执行转换变得非常快速。我们预计在 CheriBSD 的未来工作中，将会有很大一部分工作是与隔离化相关的，因为我们在不断完善我们的模型，面对越来越多的隔离化软件。

CheriBSD 既是一个正在积极开发的研究成果，也是为数十个甚至数百个用户提供服务的产品，他们进行自己的研发工作。即使是针对其他领域（嵌入式系统、Linux、Windows 等）的用户，目前 CheriBSD 仍然是测试 CHERI 技术最容易的地方。

## 为什么是 CheriBSD?

**在过去，硬件研究主要集中在裸机基准测试或嵌入式操作系统上。**

在过去，硬件研究主要集中在裸机基准测试或嵌入式操作系统上。这些系统具有较低的内存占用和通常执行较少的指令（对于模拟很重要），同时代码量较少，更容易理解和修改。然而，结果并不总是能够扩展到现实世界的操作系统，而且很容易对动态链接等问题轻描淡写地认为“只是一个小问题，编程很简单。”而适应 FreeBSD 无疑是更多的工作，但这样做使得我们能够以无与伦比的逼真程度评估 CHERI 的性能。我们之所以能够使用真实的多用户操作系统，部分原因在于时间。2010 年，FPGA 足够大，可以以相当高的速度（100MHz）运行支持完整指令集体系结构的简单核心，并且价格合理（5-10k 美元，相较于 100k 美元或更高）。同样，台式计算机足够大且足够快，可以相对容易地支持像 QEMU 这样的完整系统模拟器。

因此，我们能够在现实世界中对 CHERI 进行更全面、更真实的评估，并且逐步证明其可行性和效果。

人们可能会问：“为什么不选择 Linux？”FreeBSD 对于像 CHERI 这样的研究项目提供了许多优势。在技术方面，FreeBSD 的集成构建系统和早期采用的 LLVM 使得使用实验性编译器（C/C++ 编译器研究主要在 LLVM 中进行）构建大量软件相对容易，无论是在基本系统中还是通过 ports 进行。干净的 ABI（应用程序二进制接口）抽象支持 Linux 二进制文件，而 freebsd32 32 位兼容层则大大简化了 ABI 实验。（相比之下，Linux 只支持单一的 32 位替代 ABI，而 Windows 则通过 DLL 在用户空间内进行所有转换。）尽管这不是我们最初的决定的一部分，但后来发现，选择 FreeBSD 而不是 Linux 是因为 Linux 内核中对整数和指针都使用了 long 类型，导致了能力被无效化。尽管 Arm 和其他地方的人们正在努力进行 Linux 移植，但使用 long 类型是一个重要的障碍。

在技术以外的方面，BSD 和 FreeBSD 在成功的研究和过渡到现实世界产品方面拥有悠久的历史。从 4.2BSD 中的快速文件系统（FFS）和 TCP/IP 的套接字 API 到 FreeBSD 中的 Capsicum 和可插拔的 TCP/IP 堆栈，许多日常使用的想法被孵化在 BSD 中，影响着数十亿人。其中一个成功因素是 FreeBSD 的宽松许可证。将我们的工作发布为两条款的 BSD 许可证意味着潜在的采用者可以在拥有专有操作系统和对 GPL 许可的软件有严格控制的公司中轻松评估我们的工作。这使得 Microsoft 安全响应中心在过去对 Windows 安全漏洞进行了非常积极的评估。

最终，CHERI 的成功取决于多个操作系统的采用。如今，CheriBSD 领先于其他操作系统，拥有最新的功能和最活跃的研究。

**CHERI 时间线**

- 2010.10—开始了第一个 CHERI 项目。
- 2012.5—在 CHERI-MIPS CPU 上运行 CheriBSD。
- 2012.11—在 CheriBSD 上演示了隔离的自定义应用程序。
- 2013.10—将开发工作迁移到 git 版本控制系统。
- 2014.1—使用 CHERI LLVM 编译 CheriBSD。
- 2014.11—在 CheriBSD 上隔离了 tcpdump（每个解码器一个隔离环境）。
- 2015.5—CheriBSD 引入了压缩的能力（128 位而不是 256 位）。
- 2015.9—CheriABI 纯能力进程环境开始运行。
- 2016.1—开始将 FreeBSD 上的 RISC-V 支持合并到 CheriBSD 中。
- 2019 年 4 月—CheriABI 论文获得 ASPLOS 2019 最佳论文奖。
- 2019 年 9 月—Morello CPU、SoC 和开发板宣布发布。
- 2020 年 8 月—CheriBSD 移植到 CHERI-RISC-V。
- 2021 年 6 月—纯能力内核（RISC-V）。
- 2022 年 1 月—首批正式的 Morello 开发板发货。CheriBSD 协助进行验证。
- 2022 年 5 月—CheriBSD 22.05 版本发布，面向 Morello 开发板用户。这是一个初步支持版本，重点是安装程序和基本包基础设施。软件包集包括基本工具，包括 Morello LLVM 编译器。
- 2022 年 12 月—CheriBSD 22.12 版本发布，包括基于库的隔离，ZFS 支持，内置 GPU 的 DRM 支持以及基本 GUI 环境，其中除了 Web 浏览器之外的所有程序都是纯能力程序。

## 对 FreeBSD 的好处

**这表明有超过 1800 次提交到 FreeBSD 源代码库中带有 "Sponsored by:" 标识，表明这些提交很可能是通过对 CHERI 的工作进行资助而完成的。**

像 CHERI 这样的研究项目可以为 FreeBSD 带来显著的好处。我们为 FreeBSD 做出了从拼写错误修复到对 RISC-V 架构的 port 贡献。我们还进行了演讲，新增了新的提交者，并向许多组织介绍了 FreeBSD。

自 2011 年 1 月以来，在 FreeBSD 源代码中有超过 1800 次提交附带 "Sponsored by:" 标识，表明它们很可能是由 CHERI 项目的工作资助的。这占到了除了 contrib 和 sys/contrib 以外的提交数量的 1.5% 以上。这些贡献得益于资助了十多个提交者，其中包括两位新提交者。

**一些显著的贡献包括：**

- 对外部工具链的支持：我最初贡献了对外部工具链的支持，后来由 Baptiste Daroussin 进一步增强，添加了现在使用的 CROSS_TOOLCHAIN 变量。这个功能的添加支持了使用 CHERI Clang 编译器以及为其他两个项目开发的自定义编译器：TESLA 和 SOAAP。TESLA 使得构建和动态强制执行时间逻辑断言成为可能，而 SOAAP 允许探索大型应用程序的隔离假设。

- 无特权安装和镜像：我在 2012 年 1 月从 NetBSD 移植了将已安装文件的所有者和权限元数据存储在 METALOG 文件中的功能。这使得 intallworld 命令可以在无特权的情况下运行。结合 makefs 的支持，现在可以构建任意字节序的 UFS 文件系统。后来我还提到没有办法在分区表中嵌入一个文件系统而不挂载它，于是 Marcel Moolenaar 在 2014 年 3 月贡献了 mkimg 命令来完成所需的工具。

- MIPS64 维护：虽然 FreeBSD 有一个 MIPS port（对于我们的用途非常重要），但它没有很多用户，也没有得到太多维护。我们做了很多工作来保持它运行，并改进了我们遇到的问题。它为我们服务得很好，但当我们将最后的工作转移到 RISC-V 上并从主分支中移除 MIPS 时，我们松了一口气。

- RISC-V port：虽然 MIPS 为我们服务得很好，并且我们正试图在我们的基础 BERI MIPS FPGA 实现周围建立一个社区，但研究界显然在转向 RISC-V。因此，我们让 Ruslan Bukin 负责将 FreeBSD 移植到 RISC-V，他在 2016 年 1 月将其提交到代码库。

- Arm N1SDP 平台支持：Morello 平台基于 Arm 的 N1SDP 开发板。Ruslan 与 Andrew Turner 一起在 2020 年支持了附加的外设，包括 PCI 根复杂和 IOMMU。

- 从 macOS 和 Linux 进行交叉构建：在 2020 年 9 月，Alex Richardson 贡献了一个 make 包装器（tools/build/make.py），允许在非 FreeBSD 系统上引导 bmake 和其他构建工具。这使得用户可以在非 FreeBSD 的台式机和笔记本电脑上进行构建，以及在不支持 FreeBSD 的 CI 环境中进行构建。Alex 和 Jessica Clarke 持续维护这项支持。

- 统一的兼容系统调用存根：历史上，系统调用在 sys/kern/syscalls.master 中声明，而兼容版本在 sys/compat/freebsd32/syscalls.master 中声明。开发者往往会忘记同步它们，或者误解是否需要兼容包装。作为为 CheriBSD 添加两个 ABI 的一部分，我扩展了 syscalls.master 文件格式和存根生成代码，使得脚本能够了解所需的 ABI。现在只有一个系统调用列表，freebsd32 有一个 syscalls.conf 指定 ABI 细节。我在 2022 年初把这个工作上游。

- 无特权、跨版本构建：为了支持数百名 Morello 硬件的用户，我们需要开始生产发布版本。我们的大部分 CI 和构建基础设施都在 Linux 主机上进行无特权构建，所以 Jessica 在无特权构建和交叉构建支持方面填补了最后的空白，使我们能够在 2022 年 2 月构建发布镜像。

除了这些改进，我们在过程中还进行了许多更小的改进。有超过 1800 个提交信息，如果列出其中一小部分，我将用尽所有的字数限制。

除了技术贡献，CHERI 项目还为社区做出了贡献。我们新增了两位 committer：Alexander Richardson 和 Jessica Clarke。我们还收到了来自研究生，包括 Alfredo Mazzinghi 和 Dapeng Gao 的贡献。从短期合同到全职雇佣，我们一度支持过许多 committer，包括：Jonathan Anderson、John Baldwin、Ruslan Bukin、David Chisnall、Jessica Clarke、Brooks Davis、Mark Johnston、Ed Maste、Edward Napierala、George Neville-Neil、Philip Paepes、Alexander Richardson、Hans Petter Selasky、Stacey Son、Andrew Turner、Robert Watson、Konrad Witaszczyk 和 Bjoern Zeeb。

此外，我们还让许多人了解了 FreeBSD 作为研究平台的优势。我们参与了三个 DARPA 项目（CRASH 和 MRC 来自 I2O 项目办公室，SSITH 来自 MTO），参与这些项目的人在支持和评估我们的工作时获得了 FreeBSD 的经验。随着英国 Digital Security by Design 计划的启动，许多组织正在使用 CheriBSD 进行由 Digital Catapult 和国防科学技术实验室（DSTL）资助的演示项目。

## 总结

作为一个研究项目，CHERI 取得了巨大的成功，而 FreeBSD 在其中发挥了重要作用。拥有良好集成的基本操作系统和单体构建系统，再加上庞大的 Ports，使我们能够向广大受众展示 CHERI 的潜力，从而导致了从 Arm 的服务器级 Morello 设计到微软的 CHERIoT 微控制器等真实世界的实现。反过来，CheriBSD 的开发也为 FreeBSD 带来了显著的改进，包括 RISC-V port 和构建系统的改进。

脚注

1. https://www.cl.cam.ac.uk/research/security/ctsrd/pdfs/201904-asplos-cheriabi.pdf
2. 一些子系统通过直接映射访问用户空间，并对其进行验证，而不直接使用能力。
3. https://github.com/CTSRD-CHERI/cheribsd-ports
4. https://msrc-blog.microsoft.com/2020/10/14/securityanalysis-of-cheri-isa/
5. 部分 "由赞助提供" 行中的 "由 DARPA 赞助" 是来自于 CAETS 项目，该项目专注于 Dtrace 工作，但绝大部分是与 CHERI 相关的。

---

BROOKS DAVIS 是 SRI International 计算机科学实验室的首席计算机科学家，也是剑桥大学计算机科学与技术系（计算机实验室）的客座研究员。他领导开发 CheriBSD，这是支持 CHERI ISA 扩展的 FreeBSD 分支。他自 1994 年以来一直是 FreeBSD 的用户，自 2001 年以来是 FreeBSD 的贡献者，并且在核心团队担任了 4 个任期。
