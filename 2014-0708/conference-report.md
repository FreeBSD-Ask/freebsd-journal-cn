# 会议报告

- 原文：[Conference Report](https://freebsdfoundation.org/our-work/journal/browser-based-edition/virtualization/conference-report/)
- 作者：**Michael Dexter**

BSDCan 2014 与往年一样令人难忘，但今年的活动有一个主题比其他任何东西都更为突出：**协调**（COORDINATION）。在我参与社区的十几年里，从未见过各 BSD 项目之间有过如此活跃的对话。从互相赞赏到建设性的批评，来自各项目的开发者在会场和 BSDCan 极为宝贵的走廊交流中彼此深入互动。

## bhyve 与跨项目协作

以我十分关心的项目说起：Peter Grehan 在 FreeBSD DevSummit 上宣布 bhyve 虚拟机监控器即将支持 NetBSD，从而完善对 OpenBSD、NetBSD 和 Linux 虚拟机的支持。我想不出有比这更好的方式让开发者亲眼看到每个操作系统的运作方式并相互校验代码。感谢 Peter、Neel Natu、John Baldwin 和所有帮助 bhyve 成为 FreeBSD 中如此实用功能的人。

延续协调精神，微软 Hyper-V 团队的 Abhishek Gupta 也来到现场，与开发者探讨如何确保 FreeBSD 成为 Hyper-V 的一流客户操作系统。听上去微软为 FreeBSD 投入的开发者数量甚至比 Intel 还要多！bhyve 和 Hyper-V 共同构成令人瞩目的原生操作系统虚拟机监控器，而且请放心，bhyve 中的 Windows 虚拟机支持正在积极开发中。

## OpenZFS 与文档冲刺

OpenZFS 项目的 Matt Ahrens 做了他一年一度的更新，介绍即将进入 FreeBSD 的新 ZFS 特性，以保持 FreeBSD 作为一流 ZFS 平台的地位。其中 ZFS“bookmarks”（书签）特性将允许 ZFS 复制不再依赖快照作为历史单元。OpenZFS 项目从后 Sun Microsystems 时代的混乱过渡到稳健的、操作系统无关的贡献，速度之快令人瞩目。我们都应感谢刚刚获得 FreeBSD 提交权限的 Matt，感谢他在 BSDCan 和 AsiaBSDCon 等活动中的积极参与。

FreeBSD 文档冲刺（Doc Sprint）有两大亮点：一是 mandoc 项目的 Ingo Schwarze 参与，他将文档校对工具提交到了 OpenBSD Ports；二是 Allan Jude 凭借文档工作正式进入 FreeBSD 项目并获得提交权限。Allan 和 Kris Moore 通过 BSDNow 播客提升了 FreeBSD 和其他 BSD 项目的知名度，并展示了社区参与和代码提交可以多么无缝衔接。

## 安全与嵌入式主题

虽然我们当中许多人已经因为整天的讨论和深夜编码而疲惫不堪，但会议正会终于开始。更多优秀的人加入其中，交流与编码也持续不断。安全是关键话题，FreeBSD 地址空间布局随机化（ASLR）、Capsicum 和 LibreSSL 的演讲都是必看的活动。每场演讲都得到了来自不同 BSD 项目开发者的深度交叉点评，几乎带着对整个互联网社区的责任感，毕竟 BSD 在互联网发展中扮演着关键角色。

嵌入式主题涵盖了 ARM、MIPS64 和 NAND 闪存存储的演讲，鉴于计算性质的变化也非常及时。Warner Losh 详细讲解了 NAND 闪存存储的工作原理和各种闪存技术所能提供的不同可靠性等级。该主题甚至延伸到 Sean Bruno 主持的午餐时间 MIPS 路由器黑客 BOF。能在真正实惠的硬件上跑真正的 Unix，真是太好了。

## 长期支持与认证

DevSummit 的其他亮点包括澄清 FreeBSD 的“长期支持”政策，令人欣慰的是项目实际上或多或少在遵循提议的 5 年政策。对这类政策的正式确认对从供应商到终端用户的每个人都是宝贵的营销工具。还有人提出将 FreeBSD 基础系统拆分为包，以便模块化更新和部署。处理得当的话，这对嵌入式 FreeBSD 工作将大有裨益。

闭幕拍卖一如既往有趣，云层在周日散开，让不少与会者得以在回家前逛逛渥太华和议会大厦。一些勇敢的系统管理员选择了首届 BSD 认证组 BSD Professional 考试，我听到的反馈非常积极。BSD Professional 是实操考试，旨在补充 BSDCG 提供多年的 BSD Associate 考试。这是令人兴奋的进展，也证明了 BSD 社区的持续增长。

---

**Michael Dexter** 自 1991 年起使用 BSD Unix 系统，并参与了 jail、sysjail、mult、Xen 和最近的 bhyve 虚拟化项目。他是独立的 BSD 作者和支持服务提供者。
