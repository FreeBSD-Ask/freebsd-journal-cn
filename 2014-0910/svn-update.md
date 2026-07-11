# SVN 动态

- 原文标题：svn update
- 作者：**Glen Barber**

又到了一年中的这个时候，FreeBSD 10.1-RELEASE 周期已经开始。本专栏介绍 FreeBSD 10.1 中预计首发的一些激动人心的新功能。

## bhyve 的改进

**bhyve(8)** 是原生于 FreeBSD 运行的虚拟机管理程序。**bhyve(8)** 最初出现在 FreeBSD 10.0 中，此后又加入了许多改进和新功能。

—stable/10@r267399—

FreeBSD 10.1 中的 **bhyve(8)** 将支持 32 位 FreeBSD 客户机虚拟机，而 FreeBSD 10.0 中的版本仅支持 64 位客户机。

—stable/10@r268932—

**bhyve(8)** 现在支持启动使用完整 ZFS 文件系统的 FreeBSD 虚拟机。

—stable/10@r261090—

新增对 ACPI S5 状态的支持，允许客户机使用“软关机”ACPI 功能。

## 扩展的 ARM 平台支持

FreeBSD 10.1 在 FreeBSD 10.1 引入的 ARM 平台支持基础上，新增了对若干系统、内核更新和用户态更新。

除以下引用的提交外，还新增了对以下主板的支持：

- COLIBRI
- IMX53-QSB
- RADXA
- CHROMEBOOK
- COSMIC
- QUARTZ
- WANDBOARD

此外，还添加了对称多处理（SMP）支持，所有支持 SMP 的主板的内核配置文件现在默认启用 SMP。

—stable/10@r266000—

现在可以通过设置 u-boot 环境变量 `loaderdev` 来指定启动设备，例如 `loaderdev=/dev/mmcsd0p1`。如果 u-boot 环境中未指定启动设备，将回退到设备探测机制来确定启动设备。

## UEFI 支持

从 FreeBSD-Current 合并了对 FreeBSD/amd64 加载器的更改，提供对 UEFI 启动的支持。FreeBSD 10.1-RELEASE 将同时支持 UEFI 和 BIOS 启动机制，10.1 发布时将分别提供独立的 ISO 和内存盘镜像。未来版本计划将 UEFI 和 BIOS 启动支持合并到单个镜像中。

UEFI 支持开发的大部分工作由 FreeBSD 基金会赞助。

## 新的自动挂载设施——**autofs(5)**

新的文件系统自动挂载设施 **autofs(5)** 已从 FreeBSD-Current 合并。新的自动挂载器类似于其他类 UNIX 操作系统（如 Apple OS X 和 Solaris）中的实现，使用标准化的、兼容 Sun 的配置文件。

**autofs(5)** 实现的开发由 FreeBSD 基金会赞助。

## 新的系统控制台驱动——**vt(4)**

**vt(4)** 驱动最初在 9.3-RELEASE 中登场，现在默认可用，无需重新编译内核。**vt(4)** 驱动是与 KMS（内核模式设置）显卡集成的系统控制台驱动。

在 **vt(4)** 驱动出现之前，Xorg 环境启动后无法切回系统虚拟终端。本次更改新增了一个 **loader(8)** 可调参数 `kern.vty`，将其设置为 `vt` 时激活 **vt(4)** 虚拟终端驱动。

自 FreeBSD 9.3-RELEASE 以来，**vt(4)** 驱动经历了大量更新和增强，并将对新系统控制台驱动的支持添加到了多种非 x86 架构，如 powerpc64。

**vt(4)** 的大部分开发工作以及自 FreeBSD 9.3-RELEASE 以来的性能改进和更新，由 FreeBSD 基金会赞助。
