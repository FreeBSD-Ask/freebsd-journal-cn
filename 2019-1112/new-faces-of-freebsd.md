# FreeBSD 新面孔

作者：DRU LAVIGNE

本专栏旨在聚焦近期获得提交权限的贡献者，并把他们介绍给 FreeBSD 社区。本期聚焦 8 月获得 src 权限的 Santhosh Raju。

---

请介绍一下你自己、背景和兴趣。

- 我是 Santhosh Raju（`fox@FreeBSD.org`），来自印度，目前住在厄瓜多尔，也在那里工作。我很小就迷上计算机，喜欢和它们打交道。2003 年上大学时接触了 Linux。但直到 2009 年才经朋友 Cherry G. Mathew 介绍接触 BSD。总的来说，我喜欢软硬件都玩玩，大多在业余时间作为爱好。时间允许时也爱沉浸在电子游戏世界里。但因为没机器可玩，最近没怎么活跃游戏。我的昵称 fox@）来自我最喜欢的电子游戏里一个有趣角色。

  在计算机世界里，我对编写系统软件、与系统软件打交道有普遍兴趣，虽然至今还没找到与这爱好匹配的工作。和 Cherry 一起，我第一次接触内核内部是在为 NetBSD 编写可移植热插拔 API 时：uvm_hotplug(9)（`https://www.netbsd.org/gallery/presentations/cherry/eurobsdcon2017/uvm_hotplug.pdf`）。我也对形式化验证感兴趣，做过一点 TLA+。

你是怎么知道 FreeBSD 的？FreeBSD 哪些方面让你感兴趣？

- 我经 Philip Paeps 和 Tod McQuillin 介绍认识 FreeBSD，那是 2015 年在 HillHacks（`https://hillhacks.in/`）上，一场喜马拉雅山脚下的技术会议。除了 2009 年试过一次 NetBSD，我没深挖过其他形式的 BSD，虽然听说过 FreeBSD。Philip 鼓励我尝试用 FreeBSD 托管 IRC 客户端，因为我一直在笔记本本地跑客户端，关机就掉线很烦。我正想试试树莓派 2，就想在 RPi2 上跑 FreeBSD/arm 11.0-RELEASE，用 Irssi 搭建 IRC 客户端。这段时间我探索了如何使用 FreeBSD，从源码用 Ports 系统或二进制包管理器安装软件包。从 Linux 背景过来，我以为设置和配置会很麻烦。相反，我发现初始设置和配置更容易、组织得更好。FreeBSD Handbook 中有足够的指南和细节，涵盖从基本安装、初始配置到为第三方软件设置 Ports 的各个步骤。

  这期间我发现一个问题阻碍我使用 FreeBSD：我的 Irssi 设置需要连接 ICB 网络的插件，但 Ports 树里没有。我只好手动从源码编译，但每次升级 Irssi 都要重来。所以，在 Philip 和 FreeBSD Handbook 帮助下，我写了第一个对 FreeBSD Ports 系统的贡献——`irc/irssi-icb`（`https://blog.port0.in/2017/04/my-first-freebsd-port`）包。从那以后，我陆续把一些小软件打包进 FreeBSD，既为学习也为做贡献。一段时间后，我最终在 FreeBSD 上搭起了我用的大部分基础设施。进入 BSD 世界让我接触到的另一件事，是为中小型会议做网络。在基于 FreeBSD 的 UniFi 控制器上玩 DHCP、DNS 和 pf（`https://blog.port0.in/2017/07/networking-like-the-big-boys`）是愉快而有趣的练习。

  FreeBSD 让我觉得有用的一点是，文档充足，既有给新手的（FreeBSD Handbook），也有给想深入内核开发的（man 页）。此外，系统和软件包升级轻而易举；为安全更新构建和部署软件包只需在 Ports 树中升版本号、重新生成校验和、从软件包安装更新（如果修复急需）。我在虚拟机里跑 CURRENT 做开发和包测试，体验很好。即使东西半残——比如通过互联网升级 RPi2（`https://blog.port0.in/2018/04/updating-freebsd-in-rpi2-remotely`），Todd 帮我设置测试 RPi2，我在上面复现问题并修复——或者尝试恢复 FreeBSD 上跑的 UniFi 控制器的正确分区和文件（`https://blog.port0.in/2018/07/a-lesson-in-cloning-disks`），修复都不太痛苦。

你是怎么成为提交者的？

- 第一次贡献 `irc/irssi-icb` 之后，我对维护软件包产生了兴趣，无论是为朋友打包还是我觉得有用的东西。由于在本地 RPi2 或我跑 FreeBSD 的低配虚拟机上测试改动不便，Philip 让我用他的 `poudriere(8)` 实例。这时我才真正开始享受 FreeBSD。在 jail 中构建软件包、为不同架构和不同 FreeBSD 版本从源码构建依赖，用包构建软件非常愉快。这也帮助我更高效地测试自己写的包，并为其他需要修复或升版本的包生成补丁——这一切都能在 poudriere 中轻松测试。Cliqz（`https://www.freshports.org/www/cliqz`）是我帮忙维护的最耗时的 Port 之一。它基于 Firefox，每次更新都有某种有趣问题。但因为基于 Firefox，我通常能跟上 `www/firefox` 包，了解需要做什么才能确保它工作正常。

  在厄瓜多尔开始新工作后，我尝试周末做点软件包维护保持活跃。Philip 建议我加入 EFnet 的 ports 频道，和那里的人交流。虽然我多数时间潜水，偶尔也和人互动，问 Ports 问题。7 月底，新工作四个月后，由于一些非常不幸的情况，我被公司辞退。2019 年 8 月有大量空闲时间，我决定在我用的包里修 bug，也对我维护的包做点 `portlint(1)` / `portfmt(1)`。这期间 Philip 提议给我 Ports 提交权限，并自愿担任我的导师。

加入 FreeBSD Project 以来你的体验如何？对也想成为 FreeBSD 提交者的读者有什么建议？

- 加入 FreeBSD Project 后总结体验还为时过早，但自从开始使用 FreeBSD、与社区互动以来，体验相当积极。大家普遍鼓励、乐于助人，尽量回答我的问题。有时没回复，大概说明大家忙或者我问的太琐碎。多数时候我在 Handbook、man 页、论坛或博客里找到答案。

  我觉得给想成为 FreeBSD 提交者的读者提建议还太嫩，但可以分享一些最终帮我成为提交者的事：

  1. Handbook 是我的参考首选。有时我也漏掉东西，问问题时被指回相关章节。
  2. 损坏东西没关系，不坏就学不到，更重要的是修复。但要在受控环境下损坏，让最少的人不便。
  3. 有疑问时通过可用渠道、邮件列表、IRC 或 Slack 提问。问点傻的比让疑问萦绕然后做更傻的事好。
  4. 承担责任并确保责任被正确履行，是给提交权限时的重要方面。
  5. 一点礼貌能走很远。

---

**DRU LAVIGNE** 是 FreeBSD 文档提交者，《BSD Hacks》和《The Best of FreeBSD Basics》的作者。
