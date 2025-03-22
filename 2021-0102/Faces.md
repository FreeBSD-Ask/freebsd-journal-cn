# FreeBSD 的新面孔

- 原文链接：[New Faces of FreeBSD](https://freebsdfoundation.org/wp-content/uploads/2021/03/New-Faces.pdf)
- 作者：**DRU LAVIGNE**

在这一期中，焦点放在了 Juraj Lutter 身上，他于 2020 年 12 月获得了 ports bit。

## 请简要介绍一下你自己、你的背景和兴趣爱好

Lutter：我出生在捷克斯洛伐克（现捷克、斯洛伐克）。我的父亲是一个电子爱好者，他购买了我们家的第一台家用计算机（Sinclair ZX81），后来这台电脑被更新的后继机型（ZX Spectrum）所取代。因此，小时候我就接触了 BASIC，后来还学习了 Z80 汇编语言。1989 年我们国家发生的社会变革后，16 位计算机逐渐在国内普及，这些计算机之前是普通人无法获得的。通过这一机遇，我了解了 DOS、PC 硬件、Turbo Pascal、Turbo C、Turbo Assembler 等工具。

自 1990 年代末以来，我一直从事 IT 行业的工作。最初，我作为 PC 维修技术员，然后作为 ISP 的系统管理员，后来成为自由职业者，在系统集成行业工作。除了 FreeBSD（我负责维护各种 ports）外，我还对其他开源技术感兴趣，如 SmartOS 和 illumos，各种基础设施程序（BIND、PowerDNS、Zabbix），数据库（PostgreSQL），计算机网络（交换、路由、防火墙），以及数据存储（无论是单体存储还是 ZFS）。我还拥有 SmartOS 和 NetBSD 的 pkgsrc 的 commit bit。

我收集老式计算机（尤其是 Sinclair 的 8 位计算机），也会把一些时间花在电气工程和电子学上。

我住在我们的首都布拉迪斯拉发（**译者注：斯洛伐克首都**），有两个孩子——一个 8 岁的儿子，正在开始学习 Python；一个 10 岁的女儿，她更喜欢 LUA 语言。

## 你是如何第一次了解到 FreeBSD 的？FreeBSD 的哪些方面吸引了你？

Lutter：我初次接触 UNIX 大约是在 1995 年，具体来说是 SCO Unix 3.2，我的第一个任务之一是配置通过 X.25 线路的 UUCP。随后，我收到了来自 Walnut Creek CD-ROM 的一套 4 张 CD，其中包括 Slackware Linux 的早期版本。因为我已经在接触 UNIX（SCO），所以对 Linux 产生了兴趣，主要是因为它比 SCO UNIX 更容易安装和使用。1996 年秋季，在我上高中的第四年，一位同学向我提到了 FreeBSD，随后不久，我收到了我在 FreeBSD 服务器上的第一个用户账户，主要用于电子邮件。此后，大约在 1999 年，我开始在 Nextra（Telenor Internet）斯洛伐克分公司担任系统管理员，FreeBSD 被部署在数十台 i386 和 Digital Alpha 平台的服务器上，我们还在斯洛伐克建立了一个 FreeBSD 镜像站（包括 www、ftp、CVSup）。

FreeBSD 吸引我的是，与 Linux 相比，它是一个紧凑的系统，内核和用户空间和谐共生，且 Ports 系统使得维护不同服务器的软件包变得非常容易（例如，使用 poudriere）。我至今仍在维护我们本地的 FreeBSD 镜像。

## 你是如何成为一名 提交者的？

Lutter：我使用 FreeBSD 的时间越长，就越多地遇到各个 ports 和基础 OS 中的 bug。随着时间的推移（从 2004 年开始），我开始提交 bug 报告并贡献补丁。直到去年年底，我才与 Sergey A. Osokin 交谈，我提到有时候我提交的 bug 报告长时间未解决，而且我也希望能为开发做出贡献。Sergey 建议我去询问一下。某天，我收到了 René Ladan 的邮件，通知我已经成为 FreeBSD 开发社区的一员，成为了 ports 提交者。我非常高兴和荣幸。我要感谢 Sergey 给我加入这个了不起的社区的机会，还要感谢 Steve Wills 对我那些好奇问题的帮助和解答。

## 自从加入 FreeBSD 项目以来，你的经历如何？你有什么建议给那些有兴趣成为 FreeBSD 提交者的读者？

Lutter：由于我已经为 FreeBSD 贡献了很长时间的补丁，我看到了 提交者和提交过程的运作方式，以及代码评审的工作方式（每个贡献者都应该学习使用 Phabricator）。因此，我的印象一直是积极的。通过建设性的讨论，任何尖锐的问题都可以得到缓解，同时也能从更有经验的 提交者 s 那里学到很多东西，反之亦然。

此外，阅读（并最终记住）《Port 开发者手册》和《提交者指南》中的信息也是一个好主意。例如，我在其中找到许多关于 Subversion 和基于它的流程（如 MFH 等）的信息。

---

**DRU LAVIGNE** 是《BSD Hacks》和《The Best of FreeBSD Basics》的作者。
