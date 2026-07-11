# SVN 动态

- 原文：[svn update](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking/svn-update/)
- 作者：**Glen Barber**

又到了一年中的这个时候——FreeBSD 发布工程团队已开始 9.3-RELEASE 的发布周期。本期 svn 更新涵盖 9.3-RELEASE 中可期的内容，并介绍 FreeBSD 源代码树中其他活跃分支的亮点。

## BIND 发布版更新：stable/9@r262706

<http://svnweb.freebsd.org/base?view=revision&revision=r262706>

stable/9 基本系统中的 BIND DNS 服务器更新至 9.9.5 版本。该软件更新的发布说明见：<https://lists.isc.org/pipermail/bind-announce/2014-January/000896.html>。

## Bhyve 的 ZFS 引导支持：head@r262331

<http://svnweb.freebsd.org/base?view=revision&revision=r262331>

**bhyve(8)** hypervisor 现在支持从 ZFS 数据集引导虚拟机，从而支持纯粹的“root on ZFS”虚拟机。此前只能从 UFS 文件系统引导。

## urndis(4) 支持：stable/9@r262362

<http://svnweb.freebsd.org/base?view=revision&revision=r262362>

**urndis(4)** 驱动已加入 stable/9。该驱动最初在 head/ 中通过修订 r261541、r261543 和 r261544 提供，提供基于远程 NDIS 的以太网访问，使手机和平板等移动设备能通过 USB tethering 提供网络访问。**urndis(4)** 驱动在 stable/10 中自修订 r262363 起可用。该驱动应支持任何 USB RNDIS 提供者，例如 Android 设备上的那些。**urndis(4)** 驱动从 OpenBSD 移植而来。

## ext4 文件系统支持：stable/9@r262564

<http://svnweb.freebsd.org/base?view=revision&revision=r262564>

**ext2fs(5)** 代码已更新，在 stable/9 中启用只读模式下的 ext4 文件系统支持。head/ 中以修订 r262346 加入了对挂载只读 ext4 文件系统的支持。只读 ext4 文件系统支持在 stable/10 中自修订 r262563 起可用。

## pkg(8) 引导签名校验：stable/9@r263038

<http://svnweb.freebsd.org/base?view=revision&revision=r263038>

最初在 10.0-RELEASE 中可用的 **pkg(8)** 引导签名校验代码已合并至 stable/9。有了这一变更，当管理员发出 `pkg bootstrap` 命令时，下载的 **pkg(8)** 二进制发行包会直接用基本系统中包含的密钥校验，为不愿构建 **ports-mgmt/pkg** Port 来安装 **pkg(8)** 软件包管理工具的管理员提供了巨大的安全收益。

## Radeon KMS 驱动支持：stable/9@r263170

<http://svnweb.freebsd.org/base?view=revision&revision=r263170>

Radeon KMS 驱动从 head/ 合并而来。它支持内核模式设置（KMS），与较新主板上 Intel KMS 驱动类似。该驱动利用较新 Xorg 驱动（如 `xf86-video-ati`）提供的特性。

## VT 驱动合并：stable/9@r263817

<http://svnweb.freebsd.org/base?view=revision&revision=r263817>

新的系统控制台驱动 VT（又称“NewCons”）从 head/ 合并至 stable/9。VT 驱动作为 **sc(4)** 控制台驱动的替代品，提供从 UTF-8 字体支持到让运行 X11 并启用内核模式设置（“KMS”）的用户能从 X11 切回控制台等大量改进。VT 驱动在 stable/10 中自修订 r262861 起可用。VT 驱动由 Aleksandr Rybalko 在 FreeBSD 基金会赞助下开发。

## FreeBSD/arm 发行版支持：stable/10@r264106

<http://svnweb.freebsd.org/base?view=revision&revision=r264106>

将 FreeBSD/arm 镜像作为发布流程一部分进行构建的初步支持已合并至 stable/10。该变更将允许 FreeBSD 发布工程团队发布可用 **dd(1)** 写入 SD 卡的 ARM 镜像。目前已支持 BeagleBone Black、树莓派、PandaBoard、WandBoard Quad 和 ZedBoard。

FreeBSD/arm 镜像构建支持在 head/ 中以修订 r262810 加入，并使用 Tim Kientzle 编写的 Crochet 作为后端构建系统。head/ 和 stable/10 分支的每周镜像发布在 FreeBSD FTP 镜像：<ftp://ftp.FreeBSD.org/pub/FreeBSD/snapshots/ISO-IMAGES/arm/armv6/>。该工作由 Glen Barber 在 FreeBSD 基金会赞助下完成。

## 压缩的发行版 ISO：stable/9@r264246

<http://svnweb.freebsd.org/base?view=revision&revision=r264246>

作为发布构建流程的一部分，用 **xz(1)** 压缩的安装镜像现在可用，对那些拥有 **xz(1)** 解压工具的人而言可缩短整体 ISO 下载时间。在近期内，未压缩镜像仍将提供，因此没有 **xz(1)** 解压工具的人无需担心。压缩镜像支持最初在 head/ 中以修订 r264027 加入，并以修订 r264245 合并至 stable/10。

作为爱好者，Glen Barber 自 2007 年前后深度参与 FreeBSD 项目。此后他参与了多项工作，最近的岗位让他能专注于系统管理和发布工程。Glen 居住在美国宾夕法尼亚州。
