# SVN 动态

作者：Steven Kreuzer

自本专栏开栏以来，重点一直放在记录近期加入 FreeBSD 的新变化和令人兴奋的功能。本期我决定转换方向，重点介绍一些近期被移除的功能。FreeBSD 项目已经存在了很长时间（超过 20 年），1993 年 11 月发布 FreeBSD 1.0 时，大多数机器运行在 33 MHz，配备 16 MB 内存。从那时起到现在，无数新技术成为主流，又随着被更快、更便宜、更可靠的替代品取代而逐渐淡出舞台。所有技术都有保质期，最终用户基数会归零，或者硬件会越来越难找。虽然 FreeBSD 开发者尽量长期支持这些系统，但到了某一点，继续支持所花的时间和精力就不再划算。今天，我们向那些已经超过自身使用寿命的老技术致以最后的敬意。

## 移除 pc98 支持

（<https://svnweb.freebsd.org/changeset/base/312910>）

NEC PC-9801 最早于 1982 年 10 月发布，是日本版的 IBM 个人电脑。该系统基于 Intel 8086 及兼容处理器，虽然可以运行本地化的 Windows 和 DOS，但其设计专有，与 IBM PC 或兼容机不兼容。PC-98 在 20 世纪 90 年代主导了日本计算机市场，到 1999 年累计销量超过 1800 万台。但当 Windows 95 发布后，IBM 兼容 PC 也能显示日文，这一平台的受欢迎程度开始下滑。

## 移除 System V Release 4 二进制兼容支持

（<https://svnweb.freebsd.org/changeset/base/314373>）

UNIX System V 由 AT&T 开发，于 1983 年首次发布，是最早的商业版 Unix 操作系统之一。1988 年发布的 System V Release 4（又称 SRV4）是商业上最成功的版本。整个 20 世纪 90 年代，市面上有各种针对 x86 PC 平台的商业版 SVR4 Unix，但随着 Linux 和 BSD Unix 越来越流行，桌面 PC 上的商业版 Unix 市场迅速萎缩。到 21 世纪初，SVR4 几乎不复存在。FreeBSD 一直保留 SRV4 二进制兼容支持，但后来发现 socket 层兼容性早已失效，这清楚说明几乎没人再用这个功能了。

## 移除微通道架构支持

（<https://svnweb.freebsd.org/changeset/base/313783>）

微通道架构由 IBM 于 1987 年推出，是一种专有的 16 位或 32 位并行总线，用以取代 ISA 总线。它主要用于 PS/2 计算机，直到 20 世纪 90 年代中期最终被 PCI 取代。在常见的机型中，只有少数 486 机器采用，而这些机器早已没有足够内存运行 FreeBSD。

## 移除附加卡的 EISA 总线支持

（<https://svnweb.freebsd.org/changeset/base/313839>）

扩展工业标准架构是 1988 年公布的 IBM PC 兼容机总线标准，作为 IBM 专有微通道架构的替代方案推出。尽管 EISA 向后兼容 ISA 且非专有总线，但在桌面计算机上一直未真正普及，仅在服务器市场取得一定成功，因为它适合带宽密集型任务。

## 移除 **bdes(1)**

（<https://svnweb.freebsd.org/changeset/base/313329>）

`bdes` 工具实现了 FIPS PUB 81 中描述的全部 DES 操作模式，包括备用密文反馈模式和两种认证模式。DES 用于任何用途都不再推荐，尤其是静态 IV 为 0 的情况下，但如果需要 **bdes(1)**，仍可从 Ports 系统安装。

## 移除 Intel 82586 ISA 以太网适配器的 **ie(4)** 驱动

（<https://svnweb.freebsd.org/changeset/base/304513>）

该驱动仅支持通过外设输入输出（Peripheral Input/Output）实现的 10 Mb 以太网，且没有任何 PC Card 适配器受支持，仅支持 ISA 卡。

---

**STEVEN KREUZER** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车感兴趣。他与妻子、两个女儿和狗住在纽约皇后区。
