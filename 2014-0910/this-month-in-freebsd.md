# FreeBSD 本月动态

- 原文标题：This Month in FreeBSD
- 作者：**Dru Lavigne**

FreeBSD 项目每年会发布若干期状态报告，概述近期的工作进展。由于这些状态更新由实际参与工作的开发者撰写，状态报告能让人深入了解正在开发的新功能、工作推进情况以及尚待完成的事项。状态报告还反映了项目的发展方向，体现了项目内工作的数量、质量和多样性。报告由开发者、应用移植者、文档作者与译者，以及基础设施、核心、安全和发布工程团队成员撰写。

第一份状态报告于 2001 年 6 月发布，其中这样描述其目的：

> FreeBSD 开发模式的优势之一，是注重集中化的设计与实现——操作系统维护在中央仓库中，并在中央维护的邮件列表上讨论。这使系统各组件作者之间能高度协调，政策也能贯穿整个系统执行，涵盖从架构到风格等各方面。然而，随着 FreeBSD 开发者社区壮大，邮件列表流量和源码树修改频率都在增长，即便是再专注的开发者也难以跟上源码树中所有工作的进展。
>
> FreeBSD 月度开发状态报告尝试解决这一问题，为开发者提供一种途径，让更广泛的社区了解他们在 FreeBSD 上的持续工作，无论是在中央源码仓库内还是仓库外。这是第一期，因此是一次实验。对于每个项目和子项目，都包含一段摘要，说明自上次摘要以来的进展（本次仅说明近期进展，因为此前没有摘要）。

本月，我们回顾第一份状态报告发布时的工作，以及 10 年前、5 年前和 1 年前最后一个季度的工作。

## 2001 年 6 月

第一份状态报告包含 23 条条目。当时正在开发、至今仍被 FreeBSD 用户使用的一些重要功能包括：

- FreeBSD 二进制更新的安全分发机制
- Java 移植与授权的二进制版本
- TrustedBSD 工作，如 ACL、审计和 MAC

## 2004 年第四季度

到 2004 年，状态报告还纳入了 FreeBSD Ports 的状态信息以及发布工程团队的更新。2004 年第四季度：

- Ports 包含超过 12,000 个 port
- FreeBSD 5.3 终于在 2004 年 11 月发布

该报告包含 44 条条目。一些亮点包括：

- 从 OpenBSD 移植 CARP
- Freesbie 和 Frenzy live CD 项目的更新
- 发布 portsnap 和 freebsd-update 工具，提供安全更新
- 硬件说明文档的自动化生成
- netperf 工作，提升 FreeBSD 网络栈性能
- SMPng 工作
- TCP 清理与优化
- 将 OpenBSD PF 防火墙移植到基本系统

## 2009 年第四季度

该报告包含 38 条条目。此时：

- Ports 包含超过 21,000 个 port
- FreeBSD 8.0 于 2009 年 11 月 26 日发布

当时正在进行的一些项目包括：

- 3G USB 支持
- clang 工作
- HAST 项目，通过 TCP/IP 网络同步复制任意 GEOM provider
- SUJ（日志化软更新）
- webcamd，支持数百种不同的 USB 摄像头设备以及 Linux 模拟器中的 V4L（Video for Linux）支持
- 无线 mesh 网络（802.11s）
- 新的基于 CAM 的 ATA 实现
- ZFS 和 UFS 中的原生 NFSv4 ACL 支持
- 扁平设备树（FDT）支持

## 2013 年第四季度

该报告包含 37 条条目。2013 年最后一个季度：

- Ports 包含约 24,500 个 port
- FreeBSD 发布工程团队正在完成 10.0-RELEASE 周期

当时正在进行的一些项目包括：

- 在 FreeBSD 集群中测试 Jenkins 和 bhyve，提供持续集成和构建测试
- 原生 iSCSI 栈
- UEFI 启动
- 新的自动挂载器（autofs）
- newcons（vt）替换虚拟终端模拟器（syscons）
- 让 FreeBSD 成为 OpenStack 完全支持的宿主机，使用 OpenContrail 虚拟化网络
- 向 Cubieboard、Freescale i.MX6 处理器、Freescale Vybrid VF6xx 以及更新的 ARM 主板移植
- Capsicum 和 Casper
- 用于集中式 panic 报告的 panicmail
- 测试 Kyua 测试套件和 LLDB 调试器

项目的下一份状态报告将于 2014 年 10 月发布。所有项目状态报告可从 <https://www.freebsd.org/news/status/status.html> 获取。
