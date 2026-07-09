# FreeBSD 的新面孔

- 原文链接：[New Faces of FreeBSD](https://freebsdfoundation.org/wp-content/uploads/2021/03/New-Faces.pdf)
- 作者：**DRU LAVIGNE**

本栏目旨在聚焦近期获得 commit bit 的贡献者，向 FreeBSD 社区介绍他们。

这一期的焦点是 Juraj Lutter，他于 2020 年 12 月获得了 Ports bit。

## 请简要介绍一下你自己、你的背景和兴趣爱好

Lutter：我出生在前捷克斯洛伐克。我父亲是电子爱好者，他买回了我们家第一台家用计算机（Sinclair ZX81），不久就被更新的后续机型（ZX Spectrum）取代。因此，小时候我就接触了 BASIC，后来还学习了 Z80 汇编语言。1989 年我们国家发生社会变革后，此前私人无法获得的 16 位计算机在国内逐渐普及。这让我得以了解 DOS、PC 硬件、Turbo Pascal、Turbo C、Turbo Assembler 等工具。

自 1990 年代末以来，我一直从事 IT 行业。最初做 PC 维修技术员，然后做 ISP 的系统管理员，后来作为自由职业者在系统集成行业工作。除了 FreeBSD（我维护着各种 Port）外，我还对其他开源技术感兴趣，如 SmartOS 和 illumos，各种基础设施程序（BIND、PowerDNS、Zabbix），数据库（PostgreSQL），计算机网络（交换、路由、防火墙），以及数据存储（无论是单体存储还是 ZFS）。我还拥有 SmartOS 和 NetBSD 的 pkgsrc commit bit。

我收集老式计算机（尤其是 Sinclair 的 8 位计算机），也会把一些时间花在电气工程和电子学上。

我住在首都布拉迪斯拉发（注：斯洛伐克首都），有两个孩子——8 岁的儿子正开始接触 Python；10 岁的女儿则更喜欢 LUA 语言。

## 你是如何第一次了解到 FreeBSD 的？FreeBSD 的哪些方面吸引了你？

Lutter：我初次接触 UNIX 大约是在 1995 年，具体来说是 SCO Unix 3.2，我的首个任务之一就是配置通过 X.25 线路的 UUCP。不久后，我收到 Walnut Creek CD-ROM 的一套 4 张 CD，除其他内容外还包含 Slackware Linux 的早期版本之一。由于我已经在接触 UNIX（SCO），我对 Linux 产生了兴趣，因为它比 SCO UNIX 更易安装和使用。1996 年秋，在我上高中的第四年，一位同学向我提到了 FreeBSD，不久我就收到 FreeBSD 服务器上的首个用户账户，主要用于收发电子邮件。此后，大约在 1999 年，我开始在 Nextra（Telenor Internet）斯洛伐克分公司担任系统管理员，FreeBSD 部署在数十台 i386 和 Digital Alpha 平台的服务器上，我们还在斯洛伐克建有一个 FreeBSD 镜像站（包括 www、ftp、CVSup）。

FreeBSD 令我着迷的是，与 Linux 相比，它是一个紧凑的系统，内核和用户空间和谐共生，Ports 系统让维护不同服务器的软件包变得非常容易（例如使用 poudriere）。我至今仍在维护本地的 FreeBSD 镜像。

## 你是如何成为一名提交者的？

Lutter：使用 FreeBSD 越久，就越多地遇到各个 Port 和基础 OS 中的 bug。随着时间推移（从 2004 年起），我开始提交 bug 报告并贡献补丁。直到去年年底，我才与 Sergey A. Osokin 谈起此事，提到有时我提交的 bug 报告长时间无人解决，而我也希望参与开发。Sergei 建议由他去打听一下。某天，我收到 René Ladan 的邮件，通知我已作为 Ports 提交者成为 FreeBSD 开发社区的一员。我非常高兴和荣幸。我要感谢 Sergey 让我有机会加入这个了不起的社区，还要感谢 Steve Wills 对我那些好奇问题的帮助和解答。

## 自从加入 FreeBSD 项目以来，你的经历如何？你有什么建议给那些有兴趣成为 FreeBSD 提交者的读者？

Lutter：由于我已经为 FreeBSD 贡献补丁很长时间，所以看到了提交者和提交过程的运作方式，以及代码评审的工作方式（每个贡献者都应该学习使用 Phabricator）。因此，我的印象一直是积极的。通过建设性的讨论，任何尖锐的分歧都可以化解，同时也能从更有经验的提交者那里学到很多，反之亦然。

此外，阅读（并最终记住）《Port 开发者手册》和《提交者指南》中的信息也是个好主意。例如，我在其中找到了许多关于 Subversion 及基于它的流程（如 MFH 等）的信息。

---

**DRU LAVIGNE** 是《BSD Hacks》和《The Best of FreeBSD Basics》的作者。
