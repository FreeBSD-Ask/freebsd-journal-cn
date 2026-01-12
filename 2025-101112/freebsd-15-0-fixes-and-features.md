# FreeBSD 15.0：改进与特性

- 原文：[FreeBSD 15.0: Fixes and Features](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/freebsd-15-0-fixes-and-features/)
- 作者：John Baldwin

FreeBSD 社区持续推进 15.0 的发布。相较于 2023 年 11 月发布的 14.0，15.0 版本包含了大量新特性、改进以及 bug 修复。下面列出了一些变更要点，但更多细节可参见[发布说明](https://www.freebsd.org/releases/15.0R/relnotes/)。

## 改进项目结构

在过去两年中，FreeBSD 的开发者虽然合并了大量补丁，同时也重构了项目的若干流程和结构。这些变更旨在优化开发工作流程，合理规划开发者的时间。

在 14.0 发布后不久，Colin Percival 提出了对 FreeBSD 发布计划的若干修改。正如他在今年早些时候的[期刊文章](https://freebsdfoundation.org/our-work/journal/browser-based-edition/downstreams/freebsd-release-engineering-a-new-sheriff-is-in-town/)中详细说明的那样，新计划设定了主版本和次版本发布的固定节奏。15.0 是遵循该新计划的第一个主版本。

在去年的 BSDCan 大会上，FreeBSD 核心团队宣布成立了新的源代码管理团队来管理源代码仓库。将诸如 src commit 权限等任务委派给该团队，使核心团队能够将精力集中于整个项目的战略规划。


## 打包的基本系统

[pkg(8)](https://man.freebsd.org/pkg/8) 工具已被证明是一款成熟的二进制包管理系统。多年来，FreeBSD 一直使用 pkg(8) 来管理从 Ports 构建的第三方包。在过去几年中，一组开发者致力于使用 pkg(8) 为基本系统提供二进制更新。这项工作既包括对 pkg(8) 工具本身的增强，也包括对基本系统构建流程的修改，以便与 pkg(8) 工具集成。15.0 将是首个通过 pkg(8) 支持基本系统二进制更新的主版本。由 freebsd-update(8) 管理的旧式发布集系统在 15.x 中仍将得到支持，使终端用户能够在这些系统之间平滑过渡。FreeBSD 的开发者预计将在下一个主版本中切换安装器，使其使用打包的基本系统。

## 将开发精力集中于未来系统

开发者的时间和精力是稀缺资源。为了提供高质量的系统，FreeBSD 长期以来一直专注于当代且被广泛部署的系统。在最近几个主版本中，FreeBSD 选择逐步弃用对一些在业界使用率下降、且开发者支持有限的旧 CPU 架构的支持。14.0 已弃用若干 32 位架构，这些架构在 15.0 中将不再作为独立架构受到支持，例如 32 位 x86 和 32 位 PowerPC。这两种架构的 64 位版本在 15.0 及以后版本中仍将继续支持运行 32 位二进制程序。然而，这些架构的 32 位内核在 15.0 中已不再受支持，并且 15.0 将不再提供相应的发布产物，例如安装镜像。

## 网络

15.0 包含了对新型网络设备的支持以及对 TCP 的改进。Nvidia 贡献了对内联 IPsec 卸载的支持，使智能 NIC 能够将 IPsec 的加密 / 解密从主机 CPU 卸载到 NIC 上。这类似于内核 TLS 卸载，但针对的是 IPsec。mlx5en(4) 驱动在 ConnectX-6 及之后的适配器上支持 IPsec 卸载。15.0 中对本地（UNIX 域）套接字进行了重构，从而提升了本地流式套接字的吞吐量并降低了延迟。

## 存储

即将发布的版本中包含了多项新的存储特性。三星贡献了一个用于通用闪存存储（Universal Flash Storage, UFS）标准的驱动，这是嵌入式闪存存储中使用的 eMMC 标准的替代方案。该驱动的作者 Jaeyoon Choi 在本期的《Universal Flash Storage on FreeBSD》中对此进行了更为详细的介绍。15.0 还包含了对基于 TCP 传输的 NVMe over Fabrics（NVMe-oF）的支持，此前已有相关文章进行了介绍：[journal article](https://freebsdfoundation.org/our-work/journal/browser-based-edition/storage-and-filesystems/nvme-over-fabrics-in-freebsd-2/) 。自该文章发布以来，对 NVMe-oF 的支持已经合并进了守护进程 [ctld(8)](https://man.freebsd.org/ctld/8)，而 nvmfd 守护进程已被移除。

15.0 还包含了 inotify(2) 系列系统调用的原生实现。该实现与 Linux 中相同系统调用在 API 层面保持兼容，并可用于原生 FreeBSD 二进制程序以及在 [Linux 兼容层](https://man.freebsd.org/linux/4) 下运行的 Linux 二进制程序。对于许多使用场景而言，inotify(2) 相比通过 [kevent(2)](https://man.freebsd.org/kevent/2) 提供的 EVFILT_VNODE 内核事件更加可靠且效率更高。它也是现有桌面软件（如 KDE）中所广泛使用的 API。

## 虚拟化

在 15.0 中，FreeBSD 的二类（Type 2）虚拟机管理程序 bhyve 也进行了多项更新。内核监控器和用户态虚拟机管理程序现在均支持 64 位 ARM 和 RISC-V 架构。一些高级功能，如 PCI 直通（PCI pass-through），尚未支持，但使用现有 bhyve 设备模型（如 VirtIO）的 FreeBSD 和 Linux 客机在这两种新架构上均可完全运行。

除了增加架构支持外，bhyve 现在可以使用 net/libslirp 包为网络设备提供用户态后端。这使得主机无需额外配置网络（如 [tap(4)](https://man.freebsd.org/tap/4) 设备）即可通过网络连接与客机通信。

## 架构相关

15.0 内置了处理器跟踪框架 [hwt(4)](https://man.freebsd.org/hwt/4)，用于收集 CPU 记录的事件流。这些事件包括软件执行的详细信息，如控制流变化、异常和时间信息。该框架支持由 ARM 的 Coresight 和 Statistical Profiling Extension（SPE, 统计分析扩展）以及 Intel 的 Processor Trace（PT）记录的事件。

该版本还支持 AMD 的 IOMMU，这在多核系统上尤其有用。x86 系统上的 IOMMU 提供了多项功能。IOMMU 的主要用途是为设备 DMA 请求提供备用地址空间，这对虚拟化（如 PCI 直通）以及安全性（限制不受信任设备的内存访问）都很有帮助。在 x86 上，IOMMU 还可在中断传递中进行干预，使设备中断可以路由到编号更大的 CPU。FreeBSD 先前版本已支持 Intel 的 IOMMU（DMAR），而 15.0 引入了对 AMD IOMMU 的支持。

## 扩展错误报告

在传统 POSIX 系统中，系统调用在执行过程中通过返回整数错误码报告错误。该错误码保存在特殊全局变量 `errno` 中，可通过 [strerror(3)](https://man.freebsd.org/strerror/3) 等函数转换为字符串。15.0 在内核中引入了新的扩展错误机制，它保存关于错误的额外信息，包括附加的字符串描述和错误在源代码中的位置。字符串描述可在系统调用失败后通过 `uexterr_gettext(3)` 函数获取。系列函数 [err(3)](https://man.freebsd.org/err/3)、[errx(3)](https://man.freebsd.org/errx/3)、[warn(3)](https://man.freebsd.org/warn/3) 和 [warnx(3)](https://man.freebsd.org/warnx/3) 会自动在输出到 stderr 的消息中包含扩展字符串描述。也可通过 [ktrace(1)](https://man.freebsd.org/ktrace/1) 获取扩展错误信息。

## 第三方软件

FreeBSD 基本系统内置了多款由外部维护的组件。与每个版本一样，15.0 通过导入上游软件的新版更新了许多这些组件。更新列表过长，这里仅列出部分重点更新：OpenZFS 已更新至最新版本 2.4.0；OpenSSL 已升级至当前长期支持版本 3.5，确保 stable/15 分支全生命周期的上游支持；MIT Kerberos 已导入基本系统，取代旧版 Heimdal 实现。工具链实用程序，如 [ar(1)](https://man.freebsd.org/ar/1) 和 [size(1)](https://man.freebsd.org/size/1)，现在由 LLVM 提供，而非 ELF 工具链项目，从而使基本系统工具链支持链接时优化（LTO）。

## 结论

FreeBSD 15.0 汇聚了过去两年间广泛社区的修复与新特性。感谢所有通过测试快照、报告错误、提交补丁、在社交媒体上协助用户以及完成无数其他工作的贡献者。希望大家喜欢 FreeBSD 15.0，并请继续关注我们在 FreeBSD 16 开发中的进展！

---

John Baldwin 是系统软件开发人员，二十多年来直接向 FreeBSD 操作系统提交了各类更改，涉及内核的多个部分（包括 x86 平台支持、SMP、各类设备驱动和虚拟内存子系统）以及用户态程序。除了编写代码外，John 还曾在 FreeBSD 核心团队和发布工程团队任职，并对 GDB 调试器作出了贡献。John 与妻子 Kimberly 及三个孩子 Janelle、Evan 和 Bella 一同居住在加利福尼亚州康科德。
