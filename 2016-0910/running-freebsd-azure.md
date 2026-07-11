# 在 Azure 上运行 FreeBSD

- 原文：[Running FreeBSD Azure](https://freebsdfoundation.org/wp-content/uploads/2016/10/Running-FreeBSD-Azure.pdf)
- 作者：**Sepherosa Ziehau**

2016 年 6 月 8 日，一个标准的 FreeBSD 10.3 镜像发布到 Azure Marketplace。Microsoft 作为 FreeBSD 社区的一部分，与 FreeBSD 基金会协作发布了该镜像。这是一个里程碑，标志着 Microsoft 与 FreeBSD 社区多年合作的结晶。FreeBSD 被用作 Azure 中运行的若干虚拟设备的基础操作系统，因此 Microsoft 自然有兴趣确保它运行良好。

## 缘起

Microsoft 对 FreeBSD 的兴趣源自客户和合作伙伴的反馈。客户希望运行种类繁多的操作系统，包括 Windows、Linux 和 FreeBSD，无论是在本地环境还是在 Azure 公有云中。合作伙伴有相当多基于 FreeBSD 的虚拟设备，因此 Microsoft 自然希望合作伙伴的产品能在 Hyper-V 和 Azure 上运行良好，从而为 Microsoft 的客户提供完整的选择范围。为提供这种操作系统多样性，Microsoft 向相关开源社区贡献力量。具体到 FreeBSD，Microsoft 安排开发者参与代码工作，确保 FreeBSD 在 Hyper-V 和 Azure 上运行出色。

## 起点

FreeBSD 在 Hyper-V/Azure 上的初期工作，是 Microsoft、NetApp 和 Citrix Systems 协作的成果。2011 年底，Microsoft 与 NetApp 和 Citrix Systems 携手，将“enlightened”的 Hyper-V 设备驱动引入 FreeBSD。虽然 Microsoft 编码风格与 FreeBSD 的 **style(9)** 之间存在差异，将这些设备驱动接入 FreeBSD 的 **device(9)** 框架时也面临挑战，但这次协作相当成功：所有主要的“enlightened”Hyper-V 设备驱动，即总线驱动、存储驱动和网络驱动，都开始在 Hyper-V/Azure 上的 FreeBSD 中工作。这也促成了 BSDCan 2012 上关于 FreeBSD on Hyper-V/Azure 的联合演讲：<http://www.bsdcan.org/2012/schedule/events/287.en.html> 。在这些“enlightened”设备驱动的初期工作之后，Microsoft 开发者取得了进一步进展，2013 年这些驱动导入到 FreeBSD 主源代码树中。

## 滚动向前

Microsoft 看到 FreeBSD 在 Hyper-V/Azure 上的市场需求不断增长，以及让 FreeBSD 在 Hyper-V/Azure 上的性能和功能丰富程度达到与 Windows 和 Linux 同等水平的要求。于是 2014 年中，Microsoft 在上海组建了一个团队，专注于 FreeBSD on Hyper-V/Azure。由于大多数开发者来自 Linux 背景，他们花了一些时间熟悉 FreeBSD on Hyper-V/Azure 的代码，然后才开始进一步开发。在 FreeBSD 的 Xin Li（delphij@）帮助下，各种 Hyper-V/Azure 实用工具的驱动——例如关机和 KVP（key-value-pair）——导入到 FreeBSD 源代码树中。

## FreeBSD on Hyper-V/Azure 的第一座里程碑

在 Microsoft TechEd 2014 大会（现更名为 Microsoft Ignite）上，Microsoft 首次在《Virtualizing Linux and FreeBSD Workloads on the Next Release of Windows Server Hyper-V》演讲中正式提及 FreeBSD on Hyper-V。当时，FreeBSD 的 Glen Barber（gjb@）已经将若干 FreeBSD 镜像导入到 Azure 的 VM Depot 中。

## 飞转的车轮

2014 年底，Microsoft 再次与 NetApp 携手，改进 FreeBSD 上“enlightened”存储设备的性能。这次协作的成果相当令人振奋，存储设备性能获得了显著提升。这段代码于 2015 年初毫无保留地贡献给了 FreeBSD 源代码树。

## 陡峭的学习曲线，我们来了，我们看了，我们征服了

在结束前一个阶段的存储设备改进后，Microsoft 决定改进“enlightened”网络设备驱动的性能。校验和卸载与 TCP 分段卸载的开发进展顺利。然而如前所述，团队中大多数开发者来自 Linux 背景，他们在理解 FreeBSD 网络栈与“enlightened”网络设备驱动之间交互时面临陡峭的学习曲线。那场艰苦的战斗持续了半年多。越来越多的开发者加入战局。在“enlightened”网络设备驱动经历了一系列严肃的手术之后，FreeBSD 虚拟机终于能在 10Gbps 物理网络上跑满线速，并在 40Gbps 物理网络上达到相当不错的性能。开发团队在 BSDCan 2016 的演讲中记录了这段“走出兔子洞”的旅程：<http://www.bsdcan.org/2016/schedule/events/681.en.html> 。

## 我们仍在前行

FreeBSD on Hyper-V/Azure 的更多开发正在进行中，包括但不限于进一步改进“enlightened”存储设备驱动 SR-IOV 的性能以及许多其他方面。Microsoft 也持续帮助基于 FreeBSD 的 Azure 虚拟设备供应商和 Hyper-V 用户，使其产品运行得既高效又稳定。

## FreeBSD on Hyper-V/Azure 的第二座里程碑

在 BSDCan 2016 上，Microsoft 宣布 FreeBSD 10.3 在全球 Azure Marketplace 中可用，同时还宣布了若干基于 FreeBSD 的 Azure 虚拟设备，例如 Citrix Systems 的 Netscaler 和 Netgate 的 pfsense。

---

**SEPHEROSA ZIEHAU** 自 2007 年起就是 FreeBSD src 提交者。今年早些时候他帮助 Microsoft 进行 Hyper-V FreeBSD 开发，随后加入 Microsoft，持续推动 Hyper-V FreeBSD 开发。
