# FreeBSD 上的通用闪存存储（UFS）

- 原文：[Universal Flash Storage on FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/universal-flash-storage-on-freebsd/)
- 作者：Jaeyoon Choi

通用闪存存储（UFS）是一种高性能、低功耗的存储接口，专为移动设备、汽车以及嵌入式环境设计。如今，UFS 已被广泛应用，并在大多数 Android 旗舰智能手机中取代了 eMMC 成为主流存储方案。它还出现在部分平板电脑、笔记本电脑和汽车系统中。Linux 自约 2012 年起支持 UFS，OpenBSD 自 7.3 版本起支持，Windows 自 Windows 10 起支持。

然而，FreeBSD 之前并没有 UFS 驱动。如果将 UFS 在其他生态系统中经过验证的成熟技术引入 FreeBSD 存储架构，FreeBSD 将在许多移动和嵌入式领域成为可行方案。“为什么 FreeBSD 还没有 UFS 驱动？”这个问题成为了我个人的动力，而本文将说明我为解决这一问题所走的路径。

直到去年，我从未使用过 FreeBSD。在大约六个月的时间里，我学习了 FreeBSD、分析了其代码库，并最终实现了 UFS 驱动。我希望这个案例能够表明，哪怕是 FreeBSD 新手，也可以通过系统化的方法开发设备驱动。

本文简要介绍了 UFS，解释了其架构、开发过程和驱动设计，分享了当前状态与后续计划，并最终展示了一个基于 QEMU 的实践环境，方便读者跟随操作。

注：FreeBSD 也有传统的 UFS（Unix 文件系统）。本文中的 `UFS` 指的是通用闪存存储，而非文件系统 UFS。代码库中该驱动名称为 ufshci(4)。

## 通用闪存存储概览

在移动存储中，低延迟和低功耗至关重要，同时与现有系统的兼容性也非常重要。UFS 并非完全发明全新标准，而是通过整合现有标准来实现这些目标。因此，它迅速被市场接受。UFS 保留了其组件标准的低功耗和可靠性优势，但在实现上复杂度有所增加。

在互连层，UFS 使用 MIPI M-PHY（一种可靠的差分高速串行接口）结合 MIPI UniPro（具有强大电源管理功能的链路层）。在此基础上，传输协议层使用 UTP，应用层使用经过验证的 SCSI 命令子集，其可靠性和兼容性已被充分证明。

## UFS 的应用场景

对 UFS 可能感觉陌生，但你很可能已经在使用它。大多数 Android 旗舰智能手机的内部存储采用 UFS。UFS 还被用于部分低功耗平板、超轻便笔记本，以及可靠性要求极高的汽车信息娱乐系统中。

许多高性能 ARM 应用处理器集成了 UFS 主控器，一些低功耗 x86 平台（如 Intel N100）也支持 UFS 主控器。任天堂最近发布的掌上游戏机 [Nintendo Switch 2](https://en.wikipedia.org/wiki/Nintendo_Switch_2) 内部存储使用的也是 UFS。

随着 UFS 采用率持续增长，在 FreeBSD 中增加原生 UFS 支持可以扩大其在移动和嵌入式系统中的适用性。基于这一需求，我启动了该项目。

## UFS 架构

![](https://freebsdfoundation.org/wp-content/uploads/2026/01/choi_fig2.png)

图 1. UFS 系统模型

典型的 UFS 系统由 UFS 目标设备（通常为 PCB 上的 BGA 封装）和集成在应用处理器 SoC 中的 UFS 主控器组成。两者通过高速串行链路进行通信。

当 I/O 请求到达时，FreeBSD 的 CAM（Common Access Method）子系统会在 CCB（CAM 控制块）中构建 SCSI 命令，并将其传递给驱动程序。驱动程序将 SCSI 命令封装到 UPIU（UFS 协议信息单元）中，并将其加入 UFS 主控器的队列。

主控器将请求入队并触发门控；数据通过 DMA 传输，完成状态通过中断返回。UFS 设备执行 SCSI 子集命令（例如 READ/WRITE），访问 NAND 闪存，并将完成或错误状态返回给主控器。

通过这种方式，UFS 设备驱动控制主控器对 UFS 设备执行读写操作。

![](https://freebsdfoundation.org/wp-content/uploads/2026/01/choi_fig1.png)

图 2. UFS 分层架构

如前所述，UFS 将多个现有标准叠加，并组织为三个层次：互连层、传输层和应用层。各层的作用如下：

- **UIC（UFS 互连层）：** 管理链路。负责通过 M-PHY/UniPro 启动链路，调整档位和通道设置以平衡性能与功耗，支持电源状态（Active、Hibernate），检测错误并执行恢复。UFS 驱动通过寄存器和 UIC 命令控制该层。
- **UTP（UFS 传输协议层）：** 传输管理命令和 SCSI 命令。主控器维护管理队列和 I/O 队列；驱动将请求入队并触发门控。数据通过 DMA 传输，完成状态通过中断返回。
- **UAP（UFS 应用层）：** 处理命令（如 READ/WRITE）及命令队列控制。尽管 UFS 定义了多个命令集，但实际仅使用 SCSI 命令子集。UFS 驱动不会生成 SCSI 命令，而是将 CAM 生成的 SCSI 命令封装到 UPIU 中传递给 UTP，从而允许重用 CAM 的标准路径进行扫描、错误处理和重试。通过该层，CAM 将 UFS 视为标准 SCSI 设备。

## 主要优势（高性能、低功耗）

- **性能：** 相比 eMMC 的半双工并行接口，UFS 的全双工高速串行接口提供更高带宽和更低延迟。提交路径轻量化，基于队列、DMA 和中断构建。因此，即使仅使用单队列，性能也很稳定，多循环队列（MCQ）在多核系统上进一步提高了可扩展性。WriteBooster 利用 NAND 的 SLC 区域进一步提升突发写入性能。
- **功耗效率：** UIC 链路仅在 I/O 活跃时提升档位，空闲时迅速降至低功耗状态。标准定义了电源状态切换，从而在热量和功耗受限的移动设备中延长电池寿命。实际上，有报道显示，使用 UFS 而非 NVMe 的平板电脑可额外延长[约 30-90 分钟的续航](https://psref.lenovo.com/syspool/Sys/PDF/IdeaPad/IdeaPad_Duet_3_11IAN8/IdeaPad_Duet_3_11IAN8_Spec.pdf)。
- **兼容性：** 由于 UFS 使用 SCSI 命令子集，现有 SCSI 基础设施可以复用。在 FreeBSD 中，CAM 处理 SCSI 命令，而 UFS 驱动将 CAM 生成的 SCSI 命令封装到 UPIU 中传递给 UTP，使其与 CAM 的集成十分简便。

## UFS 的历史与后续

UFS 标准以 JESD220（UFS）发布，主控器接口以 JESD223（UFSHCI）发布。UFS/UFSHCI 1.0 于 2011 年发布。2015 年，UFS 2.1 设备首次出现在三星 Galaxy S6 中，标志着广泛商业化的开始。UFS 3.0 提升了链路速度，并引入 WriteBooster 用于突发写入。UFS 4.0 增加了多循环队列（MCQ），改善了多核扩展性。UFS 4.1 是当前版本，UFS 已部署在大多数旗舰智能手机中。

随着智能手机及其他移动设备上的本地 AI 工作负载增长，UFS 正在发展以加快将 LLM 模型传输到 DRAM 的速度。由于本地 LLM 模型需要快速加载，因此需要更高带宽；针对 UFS 5.0 的工作目标是通过提高串行总线时钟速率以提升链路速度，从而提升带宽。

## 驱动概览

我于 2024 年 7 月在 freebsd-hackers 邮件列表上提出了 UFS 设备驱动的构想，并于 2025 年 1 月开始进行分析与设计。幸运的是，Warner Losh（现为我的导师）回复并在 FreeBSD 存储架构方面提供了宝贵指导。FreeBSD 手册和 BSD 大会的演讲也提供了帮助。经过约两个月对 CAM、SCSI 和 NVMe 驱动的分析，我设计了 UFS 驱动。由于 NVMe 与 UFS 结构相似，我复用了许多相同的设计思路。我在 2025 年 5 月 16 日提交了[早期代码评审](https://reviews.freebsd.org/D50370)。经过多轮审查，该工作于 2025 年 6 月 15 日被合并，并内置在 FreeBSD 15.0 中。

开发主要通过 QEMU 的 UFS 仿真进行，随后在实际硬件上验证：包括配备 UFS 2.0/3.1/4.0 设备的 Intel Lakefield 和 Alder Lake 平台。我也曾计划在 ARM SoC 上测试，但合适硬件难以获取。

UFS 驱动与 CAM 子系统紧密集成。在初始化过程中，驱动向 CAM 注册；此后，用于配置和读写的 SCSI 命令由 CAM 传递给驱动。为了保持对 UFS 3.0 及更早版本的向后兼容性，驱动同时支持单门控队列（SDQ）路径和多循环队列（MCQ）路径。

## 设备初始化与注册

初始化遵循 UFS 分层架构，自底向上进行。

- **UFSHCI 寄存器：** 启用主控器，配置所需寄存器，并启用中断。
- **UIC（UFS 互连层）：** 发送 Link Startup 命令以建立主机与设备之间的链路，并验证连通性。
- **UTP（UFS 传输协议层）：** 创建 UTP 命令队列并启用 UTP 中断。发送 NOP UPIU 命令以验证传输路径。
- **配置档位与通道：** 协商档位和通道数量，然后配置链路以实现最大带宽运行。
- **UAP（UFS 应用层）：** 向 CAM 注册以开始基于 SCSI 的初始化；CAM 扫描总线上的目标设备和 LUN，并将 SCSI 命令传递给驱动。

CAM（Common Access Method）是 FreeBSD 的存储子系统，分为三个层次：CAM 外设层、CAM 传输层（XPT）和 CAM SIM 层。初始化完成后，UFS 驱动通过 `cam_sim_alloc()` 创建 SIM 对象，并通过 `xpt_bus_register()` 向 XPT 注册。随后，XPT 扫描总线上的目标设备和 LUN，以发现 SCSI 设备。当找到有效 LUN 时，它调用 `cam_periph_alloc()` 在 CAM 外设层创建并注册一个 Direct Access（da）外设。

在 Direct Access（da）外设注册后，当对 UFS 磁盘发起 I/O 请求时，CAM 外设层会自动构建 SCSI 命令。驱动注册在 SIM 上的 `ufshci_cam_action()` 处理函数接收携带这些命令的 CCB，将其封装到 UPIU 中，入队到 UTP 队列，并在完成后调用 `xpt_done()` 通知 XPT。

由于 CAM 处理标准 SCSI 路径，如扫描、排队、错误处理和重试，因此许多逻辑不需要在 UFS 驱动中实现。驱动的主要工作是通过 UTP 将 SCSI 命令转发到目标设备。

## 队列架构：SDQ 与 MCQ

队列架构是关键设计决策之一。UFS 4.0 引入了多循环队列（MCQ），其概念上类似于 NVMe 的模型。为了向后兼容，仍需支持单门控队列（SDQ），且驱动必须在运行时在 SDQ 与 MCQ 之间选择，因为 UFS 3.1 及更早版本仅支持 SDQ。为此，我定义了一个函数指针操作接口（ufs_qop），用于抽象队列操作，以便在运行时选择具体实现（MCQ 尚未实现，将在近期添加）。

## 当前状态与后续开发计划

UFS 驱动正在积极开发中，目前仅实现了 UFS 4.1 特性的子集。我的目标是实现全部功能特性，随后将完善电源管理和 MCQ。目前支持的平台仅限于基于 PCIe 的 UFS 主控器，我计划增加对 ARM 系统总线平台的支持。同时，我也希望在新 UFS 规范发布后能够及时跟进和采纳。

## 使用 UFS 驱动入门

要测试 UFS 驱动，通常需要配备 UFS 的硬件。幸运的是，QEMU 可以在没有实际硬件的情况下进行开发和测试。本节演示如何在 QEMU 中模拟 UFS 设备并使用驱动（UFS 模拟自 QEMU 8.2 起支持）。

首先准备一个 FreeBSD 快照镜像。

```sh
$ wget https://download.freebsd.org/releases/VM-IMAGES/15.0-RELEASE/amd64/Latest/FreeBSD-15.0-RELEASE-amd64-zfs.qcow2.xz
$ xz -d FreeBSD-15.0-RELEASE-amd64-zfs.qcow2.xz
```

创建一个 1 GiB 文件，用作 UFS 逻辑单元的后端设备：

```sh
$ qemu-img create -f raw blk1g.bin 1G
```

启动 QEMU 并模拟 UFS 设备：

```sh
$ qemu-system-x86_64 -smp 4 -m 4G \
-drive file=FreeBSD-15.0-RELEASE-amd64-zfs.qcow2,format=qcow2 \
-net user,hostfwd=tcp::2222-:22 -net nic -display curses \
-device ufs -drive file=/home/jaeyoon/blk1g.bin,format=raw,if=none,id=luimg \
-device ufs-lu,drive=luimg,lun=0
```

在 amd64 平台上，GENERIC 内核配置已包含 UFS 驱动模块（见 `sys/amd64/conf/GENERIC`）：


```sh
# Universal Flash Storage Host Controller Interface support
device          ufshci                  # UFS host controller
```

如需显式加载该模块，请编辑 `/boot/loader.conf`：

```sh
ufshci_load="YES"
```

重启后，通过 `camcontrol` 验证 UFS 设备是否附加为 `ufshci0/da0`：

```sh
$ camcontrol devlist -v
scbus2 on ufshci0 bus 0:
<QEMU QEMU HARDDISK 2.5+>          at scbus2 target 0 lun 0 (pass2,da0)
<>                                 at scbus2 target -1 lun ffffffff ()
```

使用 `fio` 进行基础性能测试：

```sh
$ fio --name=seq_write  --filename="/dev/da0" --rw=write --bs=128k --iodepth=4 --size=1G --time_based --runtime=60s --direct=1 --ioengine=posixaio --group_reporting
$ fio --name=seq_read  --filename="/dev/da0" --rw=read --bs=128k --iodepth=4 --size=1G --time_based --runtime=60s --direct=1 --ioengine=posixaio --group_reporting
$ fio --name=rand_write  --filename="/dev/da0" --rw=randwrite --bs=4k --iodepth=32 --size=1G --time_based --runtime=60s --direct=1 --ioengine=posixaio --group_reporting
$ fio --name=rand_read  --filename="/dev/da0" --rw=randread --bs=4k --iodepth=32 --size=1G --time_based --runtime=60s --direct=1 --ioengine=posixaio --group_reporting
```


QEMU 是一个模拟器，因此适合检查功能行为。对于性能测试，我使用了 Galaxy Book S。

Galaxy Book S 搭载了 Intel 第十代 i5-L16G7（1.4 GHz，5 核心）和内部 UFS 3.1 设备，我在实验中将其升级到 UFS 4.0（在 3.1 主控器上以 HS-Gear 4 运行）。

| 队列深度 | 顺序读取 (MiB/s) | 顺序写入 (MiB/s) | 随机读取 (kIOPS) | 随机写入 (kIOPS) |
| ---- | ------------ | ------------ | ------------ | ------------ |
| 1    | 709          | 554          | 7.1          | 12.1         |
| 2    | 1,395        | 556          | 14.8         | 29.4         |
| 4    | 1,416        | 559          | 31.6         | 68.2         |
| 8    | 1,417        | 554          | 63.5         | 102.3        |
| 16   | 1,399        | 555          | 103.7        | 105.5        |
| 32   | 1,361        | 556          | 114.2        | 106.6        |

**表 1. FreeBSD UFS 性能**

根据队列深度，顺序写入峰值为 559 MiB/s，顺序读取达到 1,417 MiB/s，对于移动设备而言表现非常出色。

| 队列深度 | 顺序读取 (MiB/s) | 顺序写入 (MiB/s) | 随机读取 (kIOPS) | 随机写入 (kIOPS) |
| ---- | ------------ | ------------ | ------------ | ------------ |
| 1    | 542          | 479          | 6.1          | 11.0         |
| 2    | 1,358        | 548          | 13.0         | 21.0         |
| 4    | 1,351        | 550          | 29.7         | 53.1         |
| 8    | 1,352        | 550          | 61.1         | 84.1         |
| 16   | 1,351        | 552          | 119.0        | 114.0        |
| 32   | 1,355        | 553          | 142.0        | 120.0        |

**表 2. Linux UFS 性能**

在相同条件下，Linux 的性能与 FreeBSD 相当。

## 结论

UFS 是一项快速发展的接口标准。FreeBSD 的 UFS 驱动同样在不断增加新功能并进行优化，使 UFS 能够在各种设备上运行。

希望本文能促进 UFS 在 FreeBSD 上的更广泛使用。对 UFS 驱动的贡献非常欢迎。我感谢所有帮助实现这一工作的评审人员，并将继续为社区做出贡献。

ufshci(4) UFS 设备驱动的开发得到了三星电子的支持。

---

Choi Jaeyoon 是三星电子内存部门的软件工程师，致力于扩展 SSD 和 UFS 的开源生态系统。他自 2024 年开始使用 FreeBSD，并于 2025 年 9 月成为 FreeBSD src 提交者。他曾为 Fuchsia OS 的 F2FS 文件系统做出贡献，目前维护着 Fuchsia OS 的 UFS 驱动，对存储系统的开源工作充满好奇。




