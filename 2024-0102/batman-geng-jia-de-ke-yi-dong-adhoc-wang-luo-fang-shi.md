# BATMAN：更优的可移动热点网络方式

- 原文链接：[BATMAN: the Better Approach to Mobile Ad-hoc Networks](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/batman-the-better-approach-to-mobile-ad-hoc-networks/)
- 作者：Aymeric Wibo

在广阔的网络协议领域，有一种协议脱颖而出，作为一个多功能且自适应的竞争者：BATMAN，即 Better Approach to Mobile Ad-hoc Networks（随建即连网络优化方案）。在大城市的无线电波中，BATMAN 能让设备在网状网络中无缝通信，却无需让任何一台设备了解整个网络拓扑。

今夏，我参加了谷歌编程之夏 (GSoC)，并将内核模块 `batman-adv`（提供了 Linux 对 BATMAN 协议的支持）移植到了 FreeBSD。谷歌编程之夏是个学生在暑期参与开源项目的计划，谷歌为学生提供津贴，并由导师监督。对我而言，我的导师是那位唯一无二的 Mahdi（或 Mehdi，取决于你问他是星期几）Mokhtari，即 mmokhi@。他是位非常有风度的人，我非常感激在谷歌编程之夏的旅程中有他的指导！

现在，使用 `batman_adv`（FreeBSD 上等同于 `batman-adv`）可以创建、参与网状网络，通过以太网发送/接收数据包。所有这些都在 Linux 兼容层中正常工作。移植的工作主要集中在 LinuxKPI 上（特别是让 `struct net_device` 与 `struct ifnet` 兼容，`mbuf` 与 `sk_buff` 兼容），希望在未来，这将简化将其他与网络相关的驱动程序从 Linux 移植到 FreeBSD 的过程。

## 移植过程什么样？

虽然我远不是内核组件移植方面的专家（这是我首次移植像 BATMAN 这样的大型项目），但下面是我对这个过程的一个高层次概述——希望从我的小冒险中可以获得一些洞见。

第一步——自然而然——是将 Linux 上的 `batman-adv` 代码拉入 `sys/contrib/dev/batman_adv`，再创建一个 Makefile 来构建它，Makefile 中列出了所有源文件，设置了编译参数，例如告诉它包含 LinuxKPI 的内容。正如预期的那样，初次尝试并未成功；`batman-adv` 调用了很多函数，使用了许多仅在 Linux 内核上下文中存在/才有意义的结构体。这正是 LinuxKPI 存在的原因；它提供了兼容层，通过调用 FreeBSD 内核中等效（或者有时并不完全等效）的函数来实现 Linux 内核的子集。

因此，下一步自然就是通过为 LinuxKPI 中所有缺失的函数和结构体编写存根（stub）来使其能够编译，弥补缺失的字段（我们不能直接复制 Linux 中的结构体，因为 Linux 采用 GPL 许可证）。这些存根仅包含调试打印语句，以便我们知道它们何时被调用。

在所有编译通过后，我可以加载内核模块，但这随即引发了内核崩溃。接下来的过程就是逐一检查所有调用的存根，查找并理解它们在 Linux 中的实现，并在 LinuxKPI 中实现一个等效的版本，直到内核不再崩溃为止。这大概占了工作量的 70%。

在每个操作（加载模块、创建接口、发送数据等）都能正常工作且不再崩溃后，就该拉上窗帘，在第二台显示器上打开 Wireshark，锁在我的 kot（= = 比利时宿舍的房间）里一周，确保 1) 所有设备在网状网络中能够互相识别，2) 数据能够从设备 A 通过设备 B 正常传输到设备 C，不会在过程中丢失和被破坏。大部分时间花在了重新审视之前步骤中（有时不充分的）实现上。这可能占了 30% 的工作量，但感觉像是 90%。大部分时间我都在凝视着 Wireshark，Wireshark 也在凝视着我。

最后，BATMAN 在 FreeBSD 上工作了，剩下的就是让用户态工具支持操作 BATMAN 接口，写几行文档，然后拉开窗帘。

## 合并 `batman_adv` 到上游

去年在 EuroBSDCon 上，我与核心团队的一些成员讨论了将 `batman_adv` 合并到上游的可能性，他们表示不大愿意这样做，因为 BATMAN 是 GPL 许可证的。所以它可能会永远作为一款 port 存在，因为 BATMAN 在 FreeBSD 系统中并非必需；如果你需要使用 BATMAN 来做某些事情，你很可能已处于一个可以轻松自行获取、构建 port 的环境中（与网卡驱动程序等情况不同）。

## 还有什么需要做的？

目前 BATMAN 在 FreeBSD 上的最大限制是，它不能加入无线网络。我完全打算在接下来的一年内让无线网络也能够正常工作，因为那才是 BATMAN 的主要使用场景。希望能在 BSDCan 2024 之前完成这项工作 ??

我强烈建议所有符合申请条件的人参加谷歌编程之夏。我从一开始对 FreeBSD 的网络栈和内核了解不多，到现在，在大局上仍然了解不多，但肯定比我刚开始时知道的要多得多。它尤其帮助我更加自如地在源代码树中浏览，调试内核崩溃，即使这些崩溃与网络代码无关。

## 进一步阅读

这是我的谷歌编程之夏项目 Wiki 页面，包含了所有的具体内容、代码和一个小的演示视频：[https://wiki.freebsd.org/SummerOfCode2023Projects/CallingTheBatmanFreeNetworksOnFreeBSD](https://wiki.freebsd.org/SummerOfCode2023Projects/CallingTheBatmanFreeNetworksOnFreeBSD)

此外，你可能会对这个链接感兴趣，它详细介绍了 BATMAN 在 Linux 上的实现（因此也包括在 FreeBSD 上的实现）：[https://www.open-mesh.org/projects/batman-adv/wiki/Wiki](https://www.open-mesh.org/projects/batman-adv/wiki/Wiki)

---

**Aymeric Wibo** 是比利时法语鲁汶大学的计算机科学学生，自高中起就一直使用 FreeBSD，开发基于 FreeBSD 的项目。他的主要兴趣在于图形学和网络。
