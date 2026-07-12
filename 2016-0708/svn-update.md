# SVN 动态

- 原文：[svn update](https://freebsdfoundation.org/wp-content/uploads/2016/08/svn-update.pdf)
- 作者：**Steven Kreuzer**

撰写本文时，11.0-RELEASE 的代码冻结期已经开始，待你读到此文时，head 将被冻结，为 9 月 2 日的目标发布日期做准备。即便我们快速接近那个日期，仍能看到大量开发，新的激动人心的功能每日都在加入。以下是过去两个月部分改进的小样本，但我鼓励你浏览 `svn log`，查看在高性能存储子系统、低延迟网络、安全性以及对 RISC-V、ARM 和 PowerPC 等异构架构的改进支持方面的持续工作。

## 原生 PCI Express 热插拔支持

自 r299142（<https://svnweb.freebsd.org/base?view=revision&revision=299142>）起，可以把新的 PCI 热插拔适配器插入可用的 PCI 插槽，并让操作系统和应用程序访问该设备，而无需重启机器。PCI Express 热插拔支持通过下游端口 PCI Express capability 的插槽寄存器中的位以及插槽状态寄存器中的位发生变化时触发的中断实现。FreeBSD 中的实现是在附加到表示热插拔插槽下游端口的虚拟 PCI-PCI 桥的 PCI-PCI 桥驱动程序中加入热插拔支持。PCI-PCI 桥驱动程序注册中断处理程序以接收热插拔事件，并使用插槽寄存器确定当前热插拔状态，驱动内部热插拔状态机。为简化实现，当卡从插槽中移除时，PCI-PCI 桥设备会分离并删除子 PCI 设备；当卡插入插槽时，会创建并附加一个 PCI 子设备。PCI Express 热插拔支持取决于选项 `PCI_HP`，该选项在 arm64、x86 和 powerpc 上默认启用。

## bhyve 中的原生图形支持

bhyve hypervisor 在 FreeBSD 10.0 中首次引入时，FreeBSD 社区的兴奋之情可感可知。很长一段时间里，FreeBSD 可用的虚拟化方案都颇有不足，但当一款 BSD 许可的 hypervisor 成为基本系统的一部分后，情况很快改变。最初只支持串行控制台，缺乏图形控制台模拟能力，但在 r300829（<https://svnweb.freebsd.org/base?view=revision&revision=300829>）中加入了原生图形支持。这增加了对原始帧缓冲设备、PS2 键盘/鼠标、XHCI USB 控制器和 USB 平板设备的模拟。还提供了简单的 VNC 服务器用于键盘/鼠标输入和图形输出。借助合适的 UEFI 镜像，可在图形模式下安装并运行 FreeBSD、Windows 和 Linux 客户机，使用 UEFI/GOP 帧缓冲。

## ZFS 故障管理守护进程

很高兴地说，等待终于结束了！自几年前该项目首次宣布以来，凡是熟悉 zpool 命令的人都在等待 r300906（<https://svnweb.freebsd.org/base?view=revision&revision=300906>）；**zfsd(8)** 已导入 head，这是一个处理 ZFS 池中硬盘故障的守护进程。**zfsd(8)** 让 FreeBSD 能够在磁盘事件发生时捕获并处理，并能自动管理热备件和发布物理路径的驱动器槽位中的替换件。管理几 TB 到数 PB 数据的存储管理员现在应该能稍微安心地入睡了。

## bsdinstall/zfsboot 的 GPT+BIOS+GELI 安装现使用 GELIBOOT

FreeBSD 十多年前就引入了使用 GELI 进行全盘加密的支持，但仍需让 loader 和内核保持未加密，以便能加载 GEOM 模块来处理解密。自 r300436（<https://svnweb.freebsd.org/base?view=revision&revision=300436>）起，ZFS 启动环境可与全盘加密结合使用，无需内核和引导加载程序位于启动环境之外。

## Skein 哈希算法

Skein 作为 ZFS 校验和算法的支持在 r289422（<https://svnweb.freebsd.org/base?view=revision&revision=289422>）中引入，但当时未连接，因为 FreeBSD 缺少 Skein 实现。Skein 哈希算法在 r300921（<https://svnweb.freebsd.org/base?view=revision&revision=300921>）中加入，并已连接到用户空间（libmd、libcrypt、**sbin/md5**）和内核（crypto.ko）。未来的提交也会在 ZFS 中启用此哈希功能。

## head/contrib 更新

FreeBSD 基本系统的用户空间由相当多的工具组成，其中一些在项目之外开发。过去几个月，我们看到一些第三方软件的更新，帮助打造出色的用户体验。

- file 升级至 5.26 版。（r298192）
- libucl 升级至 0.8.0 版。（r298166）
- sqlite3 升级至 3.12.1 版。（r298161）
- clang、llvm、lldb 和 compiler-rt 升级至 3.8.0 版。（r296417）
- libc++ 升级至 3.8.0 版。（r300770）
- ACPICA 升级至 20160527 版。（r300879）
- libarchive 升级至 3.2.0 版。（r299529）

---

**STEVEN KREUZER** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车感兴趣。他与妻子、女儿和狗住在纽约皇后区。
