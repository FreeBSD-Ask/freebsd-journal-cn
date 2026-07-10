# SVN 动态

- 原文：[SVN Update](https://freebsdfoundation.org/wp-content/uploads/2014/03/svn-update.pdf)
- 作者：**Glen Barber**

又到了一年中的这个时候——FreeBSD 发布工程团队已启动 9.3-RELEASE 的发布周期。本期 SVN 动态涵盖了 9.3-RELEASE 中可期待的内容和 FreeBSD 源码树中其他活跃分支的亮点。

## bhyve 的 ZFS 引导支持

head@r262331（<http://svnweb.freebsd.org/base?view=revision&revision=r262331>）

`bhyve.8` hypervisor 现已支持从 ZFS 数据集引导虚拟机，从而支持纯“root on ZFS”的虚拟机。在此变更之前，只能从 UFS 文件系统引导。

## urndis(4) 支持

stable/9@r262362（<http://svnweb.freebsd.org/base?view=revision&revision=r262362>）

`urndis.4` 驱动已合并到 stable/9。该驱动最初在 head/ 中提供（修订 r261541、r261543 和 r261544），通过 Remote NDIS 提供以太网接入，使手机和平板等移动设备能够通过 USB tethering 提供网络接入。`urndis.4` 驱动自修订 r262363 起在 stable/10 中也可用。`urndis.4` 驱动应支持任何 USB RNDIS 提供方，例如 Android 设备上的那些。`urndis.4` 驱动从 OpenBSD 移植而来。

## ext4 文件系统支持

stable/9@r262564（<http://svnweb.freebsd.org/base?view=revision&revision=r262564>）

`ext2fs.5` 代码已更新，启用了 ext4 文件系统的只读模式支持。挂载只读 ext4 文件系统的支持在 head/ 中通过修订 r262346 加入。只读 ext4 文件系统支持自修订 r262563 起在 stable/10 中也可用。

## BIND 发布更新

stable/9@r262706（<http://svnweb.freebsd.org/base?view=revision&revision=r262706>）

stable/9 基本系统中的 BIND DNS 服务器已更新至 9.9.5 版本。此次软件更新的发行说明可在此查阅：<https://lists.isc.org/pipermail/bind-announce/2014-January/000896.html>。

## pkg(8) 引导签名验证

stable/9@r263038（<http://svnweb.freebsd.org/base?view=revision&revision=r263038>）

最初在 10.0-RELEASE 中提供的 `pkg.8` 引导签名验证代码已合并到 stable/9。有了这一变更，当管理员发出 `pkg bootstrap` 命令时，下载的 `pkg.8` 二进制发行版将对照基本系统中直接包含的可用密钥验证，这为那些不希望构建 `ports-mgmt/pkg` port 来安装 `pkg.8` 软件包管理工具的管理员提供了巨大的安全益处。

## Radeon KMS 驱动支持

stable/9@r263170（<http://svnweb.freebsd.org/base?view=revision&revision=r263170>）

Radeon KMS 驱动已从 head/ 合并。它支持内核模式设置（Kernel Mode Setting，KMS），类似于较新主板上 Intel KMS 驱动所提供的功能。该驱动利用较新 Xorg 驱动（如 xf86-video-ati）中的特性。

## VT 驱动合并

stable/9@r263817（<http://svnweb.freebsd.org/base?view=revision&revision=r263817>）

新的系统控制台驱动 VT（又称“NewCons”）已从 head/ 合并到 stable/9。VT 驱动作为 `sc.4` 控制台驱动的替代品，提供了一系列增强功能，从 UTF-8 字体支持，到让运行 X11 并启用内核模式设置（简称“KMS”）的用户能够从 X11 切换回控制台。VT 驱动自修订 r262861 起在 stable/10 中也可用。

VT 驱动由 Aleksandr Rybalko 在 FreeBSD 基金会赞助下开发。

## FreeBSD/arm 发布版支持

stable/10@r264106（<http://svnweb.freebsd.org/base?view=revision&revision=r264106>）

将 FreeBSD/arm 镜像构建纳入发布流程的初始支持已合并到 stable/10。这一变更使 FreeBSD 发布工程团队能够发布可通过 `dd.1` 工具写入 SD 卡的 ARM 镜像。目前支持 BeagleBone Black、树莓派、PandaBoard、WandBoard Quad 和 ZedBoard。

FreeBSD/arm 镜像构建支持在 head/ 中通过修订 r262810 加入，并使用由 Tim Kientzle 编写的 Crochet 作为后端构建系统。head/ 和 stable/10 分支每周发布镜像，可在 FreeBSD FTP 镜像获取：

<ftp://ftp.FreeBSD.org/pub/FreeBSD/snapshots/ISO-IMAGES/arm/armv6/>

此项工作由 Glen Barber 在 FreeBSD 基金会赞助下完成。

## 压缩的发布 ISO 发行

stable/9@r264246（<http://svnweb.freebsd.org/base?view=revision&revision=r264246>）

作为发布构建流程的一部分，使用 `xz.1` 压缩的安装镜像现可提供，从而缩短具备 `xz.1` 解压工具的用户的整体 ISO 下载时间。就当前而言，未压缩镜像仍会保留可用，因此没有 `xz.1` 解压工具的用户不必担心。

压缩镜像支持最初在 head/ 中通过修订 r264027 加入，并合并到 stable/10（修订 r264245）。

---

**Glen Barber** 作为爱好者，自 2007 年左右深度参与 FreeBSD 项目。此后他参与过各种职能，最近的职务使他得以专注于项目中的系统管理和发布工程。Glen 居住在美国宾夕法尼亚州。
