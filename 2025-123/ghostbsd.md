# GhostBSD：从易用到挣扎与重生

- 原文链接：[GhostBSD: From Usability to Struggle and Renewal](https://freebsdfoundation.org/our-work/journal/browser-based-edition/downstreams/ghostbsd-from-usability-to-struggle-and-renewal)
- 作者：Eric Turgeon


这篇文章并不是为了呈现技术细节，而是从一个更高层次的角度回顾 GhostBSD 这些年来的发展历程，项目的当下状态，以及我们的后续方向。如你所知，GhostBSD 是一款基于 FreeBSD，用户友好的桌面操作系统。其使命是为那些想要 FreeBSD 强大功能，但不想面对手动设置复杂性的用户，提供简单、稳定和易于使用的桌面体验。在开始这段旅程时，我还是名非技术用户，梦想着有一天会有一款所有人都能使用的 BSD 操作系统。

## GhostBSD 的起源

2007 年，我读了 Eric S. Raymond 的《How To Become A Hacker》（如何成为一名黑客）。书中提到了 BSD Unix 是个让有志贡献者学习和成长的地方。这激发了我对 BSD 的好奇心，促使我开始探索 FreeBSD。那时，我正在使用 Ubuntu，心中产生了一个问题：FreeBSD 能否成为像 Ubuntu 一样适合非技术用户的桌面操作系统？当时，我自己也是非技术用户。我只是喜欢 Ubuntu 的简便性，并且好奇为什么 FreeBSD 不能像 Ubuntu 那样。2008 年，这个问题激发了我创建 GhostBSD 的梦想，我开始学习一切我需要的知识来实现这个项目，使 FreeBSD 对我来说变得像 Ubuntu 一样容易上手。

我花了将近两年的时间才理清一切，尝试使用像 FreeSBIE 这样的工具来制作一款 live CD，并寻找一些代码来构建 GNOME。对初学者来说，FreeSBIE 很难驾驭。作为一位讲法语的加拿大人，《FreeBSD 手册》对我有很大帮助，但我的英语水平有限，我强迫自己去学习它。我翻阅论坛和文档，拼凑 GNOME 的构建，常常是更容易把事情搞乱，而不是让它们正常工作。一个有趣的事实是，“GhostBSD”这个名字来自我的妻子。当时，她还是我的女朋友。她提议的这个名字，我就采纳了。我们在 2009 年 9 月结婚，正好是在项目初具雏形的时候。2009 年 11 月，GhostBSD 1.0 Beta 发布，作为一款运行 GNOME 的 FreeBSD 8.0 live CD。它非常粗糙，充满了问题，但为 GhostBSD 后来的发展奠定了基础。来自 FreeBSD 社区的反馈促进了早期的进展。在这个过程中，有一些人加入进来，帮助我学习源代码控制版本管理和其他技术。像 Ovidiu Angelescu 这样的人，通过他的 shell 脚本技巧让我向 SVN 迈进。我学到了很多东西。现在我们使用 Git，但那时一切都是 SVN。


## 初期阶段

首个版本是 GhostBSD 1.0 Beta，它让用户体验到了 FreeBSD 和 GNOME 的组合。尽管是个不稳定的开始，但它证明了这一概念的可行性。到 2010 年，1.5 版本通过使用来自 PC-BSD 的 pc-sysinstall 工具增加了一款基于文本的安装器，使得设置过程更加简便。那些早期的几年涉及到深入学习 FreeBSD、shell 脚本编程和与像 Ovidiu Angelescu 这样的人合作。我向 Ovidiu 请教了很多 shell 脚本的技巧。2011 年，2.5 版本引入了图形化的 GBI 安装器，基于同样的 pc-sysinstall 后端构建。这个后端至今仍然是核心组件。它的可靠性意味着我不必重新造轮子。这是一个坚持使用有效方法的教训。

大约在 2012 年开始了 NetworkMgr 的基础工作，NetworkMgr 是一款图形化工具，可用来管理以太网和 Wi-Fi 连接。这是朝着提高可用性迈出的第一步，灵感来自 Linux 的 NetworkManager。到 2015 年，我将它从 ghostbsd-build 中提取出来，单独进行优化以作为一款独立工具。2013 年，3.5 版本发生了一个重大转折。GNOME 3 在 FreeBSD 上的发布是笨重且不稳定的，存在延迟和大量资源占用的问题。这种情况与 GhostBSD 的目标相冲突。我们尝试了其他桌面环境，制作了多个带有不同桌面环境（如 LXDE、XFCE 和 Openbox）的 ISO 文件，GhostBSD 的含义——“在 BSD 上运行 GNOME”——面临了身份危机。随着 MATE（GNOME 2 的一个分支）的出现，这次危机得到了挽救。我们转向了 MATE，因为它简单且熟悉：当运行在 FreeBSD 上时，比 GNOME 2 更加轻便。当时，我不确定该如何处理其他桌面环境，因为一些人已经从项目中离开了。我开始放弃所有其他的桌面环境，但 XFCE 仍然作为一种方案保留下来。MATE 成为了旗舰桌面，重新定义了 GhostBSD 对可用性的重视，而非追求炫技。

## 系统基石的转变

我在 2014 年开始着手开发 Update Station，旨在为 GhostBSD 用户带来图形化更新工具。我原本以为这项工作是在之后才开始的，但在回顾 GitHub 后，我发现大约在那个时候，它已开始成型。最初，GhostBSD 依赖于 FreeBSD 的发布源、分发文件和官方包。曾经有一段时间，我们需要开始为像 NetworkMgr 这样的工具提供更新，这促使我们建立了自己的包仓库。我们的自定义包经常与 FreeBSD 的版本升级发生冲突，产生摩擦，要求我们寻求更好的解决方案。此外， 对于 Update Station 的自动化来说，`freebsd-update` 并不容易实现。`freebsd-update` 无法满足我们“图形化优先”目标的需求。于是我们开始关注 TrueOS 的做法。我注意到他们使用了 PkgBase，这引起了我的兴趣。他们由 pkg 驱动的操作系统更新承诺带来了图形化自由。

在 2018 年，GhostBSD 18.10 转向 TrueOS，将其作为基石。TrueOS 提供了 PkgBase，让我们可以使用 pkg 工具更新操作系统，并且我们放弃了 `freebsd-update`。TrueOS 还带来了 OpenRC，这是一款现代的服务管理器，非常适合构建用于管理服务的图形化界面，虽然这个目标最终并未实现。转向 TrueOS 使得能从图形界面使用 `Update Station` 升级软件和操作系统。对我们来说，这是一次革命性的变化：让我们的用户能够通过 `Update Station` 升级操作系统。随后，TrueOS 引入了 OS ports，让我们可以使用 poudriere 从 ports 树构建操作系统包，并为更新提供了更精细的控制。

## 遇到的挑战

TrueOS 于 2020 年关闭了，这给我们带来了压力，需要维系我们所获得的一切。我最初想保留 OpenRC，但维护 ghostbsd-ports 和 ghostbsd-src 上的所有服务变成了一个独立的斗争。那时，我几乎是独自一人维护项目，这耗费了我大量的时间，使我无法专注于改进和管理 GhostBSD。到 2022 年，我放弃了 OpenRC，转而采用 FreeBSD 更简单但可靠的 RC 系统。这样做意味着我可以减少工作量，更加专注于目标。到了 2023 年，维护 OS ports 变得过于繁重。2024 年，我决定将操作系统包的构建从 FreeBSD 维护的 PkgBase 转移到 SRC 中，从而简化了维护负担，并重新聚焦于用户体验。

今年 1 月，我意识到从 STABLE 构建 GhostBSD 对我们的小团队来说太过沉重。在与其他贡献者讨论后，我决定切换回 FreeBSD RELEASE。是的，我们失去了对早期驱动程序的利用，但我们节省了调试 STABLE 变更的时间，获得了更稳定的基础来构建。我并不是说 STABLE 总是出问题，但有时一些变更确实会导致问题。

这些年来，通过做出重大抉择，我尽我所能地管理 GhostBSD，这些选择虽然增加了困难，但也为 GhostBSD 缺失的部分带来了收益。PkgBase 和 OS ports 提供了来自图形界面的操作系统更新，但 OpenRC 增加了额外的工作负担。然而，这也意味着增加了项目的维护负担，超出了项目的处理能力。所有这些最新的变更标志着 GhostBSD 回归其根本，并重新聚焦于可用性，而不是过度复杂化 GhostBSD 的维护。这是一个艰难的教训——保持简单。

## 当前 GhostBSD 的状况

在我写这篇文章时，我们刚刚发布了 GhostBSD 25.01-R14.2p1。这标志着从 FreeBSD STABLE 切换到 FreeBSD RELEASE，使用 14.2-RELEASE-p1 能提供更好的稳定性。新的 GhostBSD 版本号分解如下：25 代表 2025，01 代表 GhostBSD 补丁，R 代表 RELEASE，14.2 代表 FreeBSD 版本，p1 代表 FreeBSD 补丁。这个版本号旨在为用户明确发布内容，不再让用户猜测版本含义。经过这些最近的变动，我感觉我们现在处于一个很好的位置，能够专注于改进我们的工具。

其中一些工具包括：

- **NetworkMgr**：一款用于以太网和 WiFi 的图形化软件，模仿 Linux 的 NetworkManager。通过简单点击替代命令行的混乱。
- **Update Station**：一款图形化软件，用于软件和操作系统升级，升级前会创建 Boot Environment 的备份。安全第一！
- **Software Station**：一款图形化软件，利用 pkg 安装软件。点击、选择、完成。
- **ghostbsd-build**：用于构建 GhostBSD，包括 Joe Maloney 的 ZFS reroot hack，用于在内存中的读写 ZFS 池实时会话。对于演示和安装来说，速度非常快。
- **Backup Station**：由 Mike Jurbala 于 2022 年 9 月引入，它是一款图形化软件，使用 pybectl，这是一款与 bectl 接口的内部 Python 模块，用于管理 Boot Environments。简化系统快照操作。
- **GBI 和 pc-sysinstall**：GhostBSD 的图形安装器最近在界面中去除了 UFS，转而利用 ZFS 的优势。ZFS 的强大优势超越了旧的方式。

## GhostBSD 未来展望

对于 2025 年，我计划记录一些标准操作程序（SOP），以便让更多人能够轻松参与 GhostBSD，希望新的贡献者不需要太多的指导。我将为自己家里的服务器部署一台更快的构建服务器，以便更迅速地构建软件包，当前我正等待 PDU 到货。我还希望完成一款 OEM 友好的安装器，以扩大我们的影响力，重新设计 Update Station 以便在启动时安装更新，并改进 NetworkMgr 与 devd 的集成以提高稳定性。如果可能的话，我还计划为 GBI 添加创建主目录数据集的支持，并且可以选择进行加密。FreeBSD 即将在 2025 或 2026 年提供的 AC 和 AX WiFi 支持将带来更好的连接速度，我对此感到非常兴奋。我们的笔记本电脑非常需要这个。

我不能代表其他贡献者发言，但我们在 GitHub 上有一长串任务。我们确实有一份路线图，可以在 GhostBSD.org 的开发标签下找到它。如果你感兴趣，可以去看看。

从长远来看，我们在期待更多的捐款，以便购买一台 ARM（Ampere）服务器，开始构建 GhostBSD 的 arm64 版本。与此同时，GhostBSD 仍然致力于成为一个完全由图形界面驱动的操作系统，利用 ZFS，这非常适合那些希望享受 FreeBSD 提供的功能但不懂技术的用户。我们正在讨论创建一些桌面组件，逐步替代 MATE，这些组件将更好地与 FreeBSD/GhostBSD 对接，比如一个利用 ZFS 的文件管理器。然而，目前这些讨论还没有形成实质性的成果。


## 结论

GhostBSD 是一段经历，一连串的选择，从 2009 年的 Live CD 到今天的模样。它起步于一名非技术用户的梦想，逐渐发展成了一个社区项目。每一步，像是定制包、TrueOS、ZFS reroot 实时会话和 PkgBase 都解决了一个挑战。过去的经历教会了我坚韧，当前带来了稳定，而未来邀请你们一同塑造。如果你有兴趣参与其中，请访问 GhostBSD.org。你会找到属于你的位置。

---

**ERIC TURGEON** 是 GhostBSD 的创始人和领导者。他还是 FreeBSD ports 的提交者，专注于维护 MATE 和 NetworkMgr 的 ports。Eric 居住在加拿大，自 2000 年代末期以来一直对 BSD 怀有热情。他在 GhostBSD、FreeBSD 的贡献、工作和个人生活之间平衡时间。他的动力来自于让 BSD 更加普及，他欢迎所有人加入 GhostBSD 社区。

