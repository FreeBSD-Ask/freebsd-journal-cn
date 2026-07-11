# SVN 动态

- 原文：[svn Update](https://freebsdfoundation.org/wp-content/uploads/2016/10/svn-Update.pdf)
- 作者：**Steven Kreuzer**

7 月 8 日，svn 的 head 合并到 **stable/11** 分支，次日源代码仓库解冻后，FreeBSD-CURRENT 可以亲切地称作 FreeBSD 12。开发者和终端用户都在忙着测试并协助清除最后时刻的 bug，确保 11-RELEASE 像其他版本一样坚如磐石；与此同时 FreeBSD 10 仍在积极开发中，不少有意思的提交从 head 流入了 **stable/10**，聚焦于改进性能和兼容性。

## MFC r302931

MFH r301732, r302288（<https://svnweb.freebsd.org/base?view=revision&revision=302931>）

EC2 中 `boot_multicons="YES"` 切换为 `boot_multicons="YES"`。这使运行在 Elastic Compute Cloud 中的 FreeBSD 实例能够利用 Amazon 近期推出的 API（<https://aws.amazon.com/blogs/aws/ec2-instance-console-screenshot/>），对模拟的 VGA 设备截图。

## Xen blkfront 驱动

在 EC2 上运行时，Xen blkfront 驱动默认启用 indirect segment I/O。得益于 EC2 的改进，某些 EC2 实例上曾经存在的性能损失不复存在，启用该特性现在能稳定带来约 20% 的吞吐量提升，且延迟不变甚至更低。

## MFC r302901

MFC r297187（<https://svnweb.freebsd.org/base?view=revision&revision=302901>）

Intel PRO/1000 千兆以太网适配器现在支持 TCP/IPv6 和 UDP/IPv6 的校验和卸载，并支持 SCTP/IPv6 的 SCTP 校验和卸载。

## MFC r302800

MFH r297023（<https://svnweb.freebsd.org/base?view=revision&revision=302800>）

**kldstat(8)** 现在能够以合适的格式打印模块特定的信息。除其他用途外，我们能查询动态加载、向量号动态分配的 syscall 编号。

## MFC r302791

MFC r302402（<https://svnweb.freebsd.org/base?view=revision&revision=302791>）

**ahci(4)** 驱动现在能够附加到具备 32 个端口的控制器。由于符号错误，本应是位域的变量扩展导致无限循环。修复此问题后，系统能够正确检测 bhyve 中 AHCI HBA 上配置的 32 个设备。

## MFC r302714

MFC r302265, r302382（<https://svnweb.freebsd.org/base?view=revision&revision=302714>）

ZFS ARC 的最小值和最大值现在可以在运行时调整。在此变更之前，ARC 的最小值和最大值只能通过启动时的可调参数修改。现在可以在运行时通过 sysctl `vfs.zfs.arc_max` 和 `vfs.zfs.arc_min` 修改这些值。

## MFC r302688

MFC r302174（<https://svnweb.freebsd.org/base?view=revision&revision=302688>）

修复了配置超过 2TB 虚拟内存的机器上 `sysctl vm.vmtotal` 的输出。

## MFC r302270

MFC r301545（<https://svnweb.freebsd.org/base?view=revision&revision=302270>）

mlx5en 驱动新增 SR-IOV 客户机支持。这补全了设备设置的缺失部分：在通过 SR-IOV 提供硬件访问的虚拟机内，可以使用 mlx5en 驱动进行设备设置。

---

**STEVEN KREUZER** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车感兴趣。他与妻子、女儿和狗住在纽约皇后区。
