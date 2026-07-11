# 从采用到贡献之旅

- 原文：[A Journey from Adoption to Contribution](https://freebsdfoundation.org/our-work/journal/browser-based-edition/mips-and-arm64/)
- 作者：**Rick Miller、Julien Charbon**

与 FreeBSD 项目互动

为产品选择合适的部署平台相当直接，只需根据产品的关键指标评估可用选项。即便部署平台本身也相对简单，唯一复杂之处在于将其与供应和安装框架等系统集成。FreeBSD 满足了我们的要求，因此被采用并部署为 Verisign 运营的域名系统（DNS）基础设施中的多样化平台。但我们并未止步于此。

为鼓励 FreeBSD 的进一步发展，我们成为——并始终是——社区的活跃参与者和协作者，如今我们团队中已有一位工程师成为该项目的提交者（committer）。

在全球范围内采用并部署 FreeBSD 到 Verisign 的域名系统基础设施中，这一决定只是我们参与项目的起点。从一开始我们就知道，主动与 FreeBSD 基金会、项目本身以及支持社区互动协作，既重要又必要。但在与社区接触之前，我们先要理解项目本身、其运作方以及生态系统——各组件之间如何相互关联、如何协同运作。

FreeBSD 网站（<http://freebsd.org>）提供宝贵资源，从哲学、政策、流程和实践的角度描述 FreeBSD 生态系统。FreeBSD 的核心团队、发布团队和安全团队的职能也都有详尽文档，对各自的实践和流程做了细致说明。此外，与项目沟通、提交补丁以及成为提交者的政策也在此处。理解这些职能和流程，能更深入洞察各组件如何互动。

## 电子通信

我们与 FreeBSD 项目的沟通始于各项目邮件列表上的电子邮件。邮件列表存档中包含协作实例，涵盖的技术互动包括自定义和构建发布介质与 Ports/软件包、将 FreeBSD 集成到安装框架、驱动功能、网络性能等等。

在评估硬件厂商与 FreeBSD 兼容性时，我们意外遇到一种沟通模式。虽然只触发过一次，却产生了效果。硬件中的网卡未能按预期工作，我们向厂商反映了问题，厂商随后将我们引荐给一家 FreeBSD 重度用户。该组织有员工是 FreeBSD 提交者，协助我们找到了现有的上游补丁，缓解了功能问题，随后补丁被合并到内部开发分支。虽然此举行之有效，但我们也意识到由于场景罕见，加上我们如今已有内部人才可自行解决此类问题，再次发生的可能性不大。

## 技术人脉网络

电子通信虽然宝贵，但在日益互联的世界中，技术人脉网络仍提供无可估量的价值。人脉建立发生在专门定向的活动中，如本地或区域用户组和会议，例如总部位于华盛顿特区/巴尔的摩地区的区域 BSD 用户组 CapBUG，以及分别位于旧金山湾区和加拿大渥太华的 MeetBSD 和 BSDCan 等会议。参与这些活动的益处众多：从与曾在邮件或邮件列表上交流的提交者进行面对面闲聊，到在会议主办的骇客休息室工作组中坐下来协作完成项目。

在 2012 年的 MeetBSD 上，我们的工程师亲身体验了这些益处，他们有机会与通过邮件列表和电子邮件结识的各方人士面对面交流。iXsystems 与 Verisign 之间的良好关系最终促成 Verisign 在 2013 年举办了首届 vBSDcon——一种主动回馈社区的好方式。

BSDCan 无疑是北美大陆“必去”的 BSD 相关会议。正是在 2013 年，Verisign 派出了首个小型代表团出席。在 MeetBSD 上积累的经验在 BSDCan 上得以延伸，会议包含参加厂商与开发者峰会的邀请，这些峰会先于部分 BSD 相关会议举行。此外，会议让我们的工程师与正在领导 Verisign 特别关注项目（如网络性能）的 FreeBSD 工程师直接接触。

## Verisign 的经验

从开发角度看，将 Verisign 核心 DNS 基础设施软件栈移植到 FreeBSD 上相当顺利。在很大程度上，这得益于许多必备软件默认受支持，以及 FreeBSD 高度的 POSIX 合规性。FreeBSD 作为多样化操作系统通过验证的最后一步是运行稳定性和性能测试。然而，Verisign 的工作负载足够特殊，会在操作系统内引发独特问题，如内核恐慌、驱动崩溃、可扩展性问题等等。

`udp_input()` 中的内核恐慌是最早遇到的有趣问题之一。工程师设计并执行实验，编写代码可靠地重现该问题，以便设计合适的解决方案，随后通过 bug 跟踪系统提交给项目 [1]。在提交 bug 报告之前，我们必须先调试问题，对其影响范围有把握。这加快了与相关提交者的技术讨论，也树立了 FreeBSD 开发者对 Verisign 工程师诊断和解决问题能力的信心。bug 报告和邮件列表沟通中体现出的严谨、耐心和坚持，吸引了在同一领域工作的 FreeBSD 开发者的注意。

后续开发让我们发现更多有趣的边缘案例 [2][3][4]，并促成与 FreeBSD 项目更紧密的协作。BSDCan 2013 之前的开发者峰会是理想场所，让我们能够与相关技术领域的工作者以及更广泛的社区分享和讨论发现。开发者峰会和 BSDCan 上的面对面协作，实现了有针对性的合作，加快了代码和补丁“提交/审阅/接受”生命周期的节奏。

git 这一设计精良的工具，能够协作管理整个 FreeBSD 软件栈，Verisign 利用官方 FreeBSD git 镜像跟踪 FreeBSD 源码的内部变更。这使 Verisign 能够带着补丁构建 FreeBSD，在真实场景下测试——再提交回项目。此外，它有助于保持补丁的时效性，尤其是当审阅和补丁改进的时间线不一致时。我们的相关变更也得到跟踪和发布，供项目纳入其自身的开发分支 [5]。

这些关系持续演进，我们的工作在社区中得到了更广泛的曝光，途径包括 BSDnow 播客、BSDCan 2014 开发者峰会的再次邀请，以及我们发表在 FreeBSD 期刊 2014 年 5/6 月刊上关于 TCP 网络栈性能可扩展性问题及 Verisign 提议的解决方案的文章 [6]。最后同样重要的是，Verisign 的一位工程师 [Julien Charbon] 被接纳为 FreeBSD 提交者社区的一员。

从在 freebsd-questions 上发邮件询问如何自定义安装介质，到提交第一份 bug 报告，再到参加和组织会议，最终获得提交权限，这一历程展示了以有意义方式参与所需的努力和奉献。它需要双方组织的敬业工程师投入大量时间——有时完成的工作与本职并不直接相关。但这是出于对项目的贡献感，该项目旨在为社区提供一个平台，让社区成员实现自己的目标。我们感谢 FreeBSD 项目、社区以及基金会的持续支持，并期待未来的合作，也许就在即将举行的 vBSDcon 2015（具体日期稍后正式公布）。

## 参考资料

[1] Kernel panic in udp_input()
<https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=172963>

[2] Concurrency in ixgbe driving out-of-order packet process and spurious RST
<https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=176446>

[3] TCP stack lock contention with short-lived connections
<https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=183659>

[4] Kernel panic: Sleeping thread owns a non-sleepable lock from netinet/in_multi.c
<https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=185043>

[5] Verisign’s FreeBSD public repository
<https://github.com/verisign/freebsd>

[6] FreeBSD Journal May/June 2014 issue
<https://mydigitalpublication.com/publication/?i=217642>

---

**Julien Charbon** 是 Verisign 公司的软件开发工程师。Julien 参与过公司的大规模网络服务 ATLAS 平台及相关大规模网络服务。Julien 使用 FreeBSD 完成过诸多任务，包括移植软件、开发内核修复和补丁，以及网络栈性能改进。

**Rick Miller** 是 Verisign 公司的 UNIX 系统工程师，负责构建支持全球 DNS 解析平台的基础设施系统。过去五年间，Rick 的重点是整合和部署 FreeBSD 到这些平台。这包括构建开发/运营支持系统，以及管理操作系统源码和镜像构建。
