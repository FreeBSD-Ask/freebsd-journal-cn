# FreeBSD 本月动态

作者：Dru Lavigne

FreeBSD 今年迎来 21 周年——首个 RELEASE 公告于 1993 年 11 月 1 日发布。FreeBSD 1.0-RELEASE 的公告可在 <https://www.freebsd.org/releases/1.0/announce.html> 查阅。1.0 版本的发布工程师 Jordan Hubbard 在 21 周年当天的 MeetBSD 大会上做了题为《FreeBSD：接下来的 10 年》的演讲，大会在圣何塞举行。演讲录像可访问 <https://archive.org/download/bsdtalk247/bsdtalk247.ogg>。

今年 11 月也有发布公告：FreeBSD 10.1 于 11 月 14 日发布。本月的栏目将介绍 10.1-RELEASE 的部分特性，按兴趣领域分类。

## 桌面用户

期待已久的新控制台驱动 **vt(4)** 已经加入。该驱动支持 Unicode UTF-8 文本，包括双宽字符和大字号字体映射，并对亚洲字符集提供支持。与 KMS 集成意味着 `Ctrl Alt Fx` 在虚拟终端之间切换的功能现在又能用了。

FreeBSD/amd64 架构加入了 UEFI 引导的初步支持，包括串口控制台和 null 控制台支持。包含此支持的 UEFI 镜像可从 FreeBSD 网站下载。

系统的自动挂载设施由 **autofs(5)** 替换。新的自动挂载器是从头实现的，解决了旧 automount 系统的局限。它通过基于 autofs 文件系统实现的恰当内核支持，提供了多数其他 Unix 系统都具备的功能。新自动挂载器与轻量级目录访问协议（LDAP）服务集成，并已在若干企业和大学环境中经过测试，可支持数千条映射项。

新增了 UDP Lite 支持。这种无连接协议允许将可能损坏的数据包送达应用程序，而不是自动丢弃。这样可以把数据完整性的判断交给应用程序或编解码器处理。它专为多媒体协议设计，例如流式视频——在这种情况下，收到负载损坏的数据包胜过完全收不到数据包。

## 虚拟化用户

CAM target 层 **ctl(4)** 新增了对若干 VAAI（vStorage APIs for Array Integration）原语的支持。VAAI 是一套 API 框架，能将某些存储任务（如精简配置）从虚拟化硬件卸载到存储阵列。以下新增内容大幅改善了对 VMware VAAI 加速、Microsoft ODX（Offloaded Data Transfer）加速和 Windows 2012 集群的支持：

- unmap：告知 ZFS 删除文件释放的空间应当归还。没有 unmap，ZFS 无法感知通过 VMware 或 Hyper-V 等虚拟化技术释放的空间。
- atomic test and set：允许虚拟机仅锁定其正在使用的部分，而非锁定整个 LUN，后者会阻止其他主机同时访问同一个 LUN。
- write same：以厚配置分配虚拟机时，必要的零写入在本地完成，而非通过网络，虚拟机创建因此快得多。
- xcopy：类似 Microsoft ODX，复制在本地完成而非通过网络。

BSD Hypervisor **bhyve(8)** 是 Type-2 hypervisor，支持多种客户机，包括 FreeBSD、OpenBSD、NetBSD 和若干 Linux 发行版。本次发布对 bhyve 做了若干改进：

- 支持从 ZFS 文件系统引导。
- 通过 **acpi(4)** S5 状态实现软关机。
- 虚拟化的 XSAVE 支持，允许客户机使用 XSAVE 相关特性，例如 AVX（Advanced Vector Extensions）。

**bhyve(8)** 新增若干选项：`-U` 用于指定客户机的 UUID，`-e` 设置 **loader(8)** 环境变量，`-C` 指定客户机控制台设备，`-H` 把宿主路径传给 **bhyveload(8)**。

FreeBSD 的 VirtIO 实现 **virtio(4)** 提供了 BSD 许可的从头实现版本，实现了为 Linux KVM（Kernel-based Virtual Machine）开发的半虚拟化接口。本次发布对 VirtIO 做了若干改进：

- API 从 32 位扩展到 64 位。
- **virtio_blk(4)** 和 **virtio_scsi(4)** 驱动已更新支持未映射 I/O。未映射 I/O 大幅降低延迟，并在多处理器系统上提升 I/O 可扩展性和性能。
- 新增 **virtio_random(4)** 驱动，允许 FreeBSD 虚拟机从 hypervisor 收集熵。

## ARM 用户

对 ARM 架构的支持持续改进。10.1 中新增以下支持：

- CHROMEBOOK（Samsung Exynos 5250）
- COLIBRI（Freescale Vybrid）
- COSMIC（Freescale Vybrid）
- IMX53-QSB（Freescale i.MX53）
- QUARTZ（Freescale Vybrid）
- RADXA（Rockchip rk30xx）
- WANDBOARD（Freescale i.MX6）

为支持 TI 平台（如 BEAGLEBONE 和 PANDABOARD），新增以下驱动：

- PRUSS（Programmable Realtime Unit Subsystem）
- MBOX（Mailbox hardware）
- SDHCI（用于 MMC/SD 存储的更快新驱动）
- PPS（GPIO/定时器引脚上的 Pulse Per Second 输入）
- PWM（Pulse Width Modulation 输出）
- ADC（Analog to Digital Converter）

后续 FreeBSD 版本中还会有更多改进，以及对新兴 ARMv8 架构的支持。例如，FreeBSD 基金会 与 Cavium Inc. 最近宣布合作，为 ARMv8 架构创建 FreeBSD Tier 1 认可以及 Cavium ThunderX 处理器家族的优化实现。新闻稿见 <http://www.prnewswire.com/news-releases/cavium-to-sponsor-freebsd-armv8-based-implementation-277724361.html>。

## ZFS 用户

新增了 bookmarks 特性标志。书签标记创建快照时的时间点，可作为 `zfs send` 命令的增量来源。

`vfs.zfs.min_auto_ashift` sysctl 可用于在 Advanced Format 驱动器上创建新的顶层虚拟设备时设置最小 ashift 值。

`libzfs` 线程池 API 已从 OpenSolaris 导入并适配到 FreeBSD。这允许并行扫描磁盘，可缩短某些工作负载下的池导入时间。

**restore(8)** 工具已更新，防止把 UFS 文件系统转储还原到 ZFS 文件系统时出现断言失败。

默认 ARC 哈希表大小已增大，并新增了 **loader(8)** 可调参数 `vfs.zfs.arc_average_blocksize`。

FreeBSD 安装器 **bsdinstall(8)** 已更新，在安装到 ZFS 文件系统时可选 GELI 加密或镜像的交换设备。此外，父 ZFS 数据集现在默认启用 LZ4 压缩。

## 更多信息

我们只介绍了 FreeBSD 10.1-RELEASE 中部分新特性。更多信息请参阅发布公告和发行说明：

- <https://www.freebsd.org/releases/10.1R/announce.html>
- <https://www.freebsd.org/releases/10.1R/relnotes.html>

---

**Dru Lavigne** 是 FreeBSD 基金会 的董事，BSD Certification Group 主席。
