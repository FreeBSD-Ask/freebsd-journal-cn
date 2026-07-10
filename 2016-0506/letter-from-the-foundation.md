# 基金会来信：FreeBSD 为最新 64 位架构做好了更充分的准备

- 原文：[Foundation Letter: FreeBSD is Even Better ARMed for the Latest 64-bit Architecture](https://freebsdfoundation.org/wp-content/uploads/2016/06/Foundation-Letter.pdf)
- 作者：**George Neville-Neil**

致 FreeBSD 期刊编辑委员会

本期有两篇文章介绍 ARMv8 架构的最新进展，ARM 公司正借此进军 64 位服务器计算领域。FreeBSD 是 ARMv8 架构的早期采用者，FreeBSD 基金会、ARM 公司和 Cavium 都投入了资金和工程资源，确保 FreeBSD 在这一新架构上得到良好支持。Andrew Wafaa 描述了让 FreeBSD 在 ARMv8 上成为 Tier 1 架构所付出的努力。Tier 1 架构是指能够自托管并拥有 FreeBSD Ports 中完整的第三方软件包的架构。达到 Tier 1 是任何运行 FreeBSD 的架构的重要里程碑，标志着长期的郑重承诺。

Zbigniew Bodek 和 Wojciech Macek 讨论了 FreeBSD 在 ARMv8 架构的特定实现——Cavium ThunderX 平台上的运行情况。ARM 自身并不制造芯片，而是授权设计供他人生产芯片，并允许在芯片中增减特性。这种专业化程度正是 ARM 在移动和嵌入式平台广受欢迎的原因。硬件厂商只需在基本核心之外选取构建平台所需的部分。要在真实硬件而非处理器模拟器上启动 FreeBSD，必须有硬件可用，Cavium 在此过程中较早推出其平台，成为 FreeBSD 开发者首选的目标硬件。Zbigniew Bodek 和 Wojciech Macek 并非 Cavium 员工，而是与 Cavium 合作在 ThunderX 平台上启动 FreeBSD。

第三篇文章介绍 FreeBSD 原生虚拟化方案 bhyve 的一项重要新特性。Teaca Ionut-Alexandru、Mihai Carabas 和 Peter Grehan 描述了他们为 bhyve 提供 ATA 仿真层的工作。ATA 和 ATAPI 仿真是支持在现代 FreeBSD 宿主上运行旧版 FreeBSD 客户机所必需的。许多人在旧版本上运行 FreeBSD 应用长达数年，为这些遗留应用提供虚拟平台，对许多公司而言是重要的过渡路径。

Steven Kreuzer 在他的 svn 更新专栏中讨论了因 ARM 相关工作而进入 FreeBSD 代码树的所有变更。他接手了这个专栏，显然已驾轻就熟。

Dru Lavigne 采访了 Benedict Reuschling。Benedict 多年来以多种方式为 FreeBSD 做出贡献，如今与 Dru 一同担任 FreeBSD 基金会董事会成员。Benedict 在德国达姆施塔特从事教学工作，热衷于推动 FreeBSD 走进更多课堂。

最后，无论是邮件还是当面交流，我们都很乐意听到读者的反馈。如果在接下来的会议上遇到编辑委员会的成员——2016 年还有数场会议将举办——请务必告诉我们你对期刊的看法，以及它如何更好地服务 FreeBSD 和更广泛的计算社区。

此致

**George Neville-Neil**

FreeBSD 期刊编辑委员会
