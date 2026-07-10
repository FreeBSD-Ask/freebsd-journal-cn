# 欢迎来到 FreeBSD 11

- 原文：[Welcome to FreeBSD 11](https://freebsdfoundation.org/wp-content/uploads/2016/10/Welcome-to-FreeBSD-11.pdf)
- 作者：**John Baldwin**

FreeBSD 系统在不断变化。FreeBSD 11 带来了两年半活跃开发积累的新特性与修复。其中部分改动已合并至近期的 10.x 版本（如 10.2 和 10.3），但大多数都是 11 中全新的内容。

## 桌面与笔记本电脑

FreeBSD 11 为桌面与笔记本电脑用户提供了多项改进。首先，新的系统控制台驱动默认启用。该驱动不再像以往那样以 VGA 和 x86 为中心。控制台不再依赖 BIOS ROM 对 VGA 文本模式的支持，而是在帧缓冲上以软件方式渲染文本。它通过图形模式支持 VGA 适配器，也支持 UEFI 帧缓冲。它还支持在采用 Intel 等现代 GPU 高分辨率图形模式时禁用 VGA 兼容性的图形适配器。软件文本渲染使控制台驱动能够渲染任意字形，从而支持 UTF-8。此外，内核图形驱动原生支持配备第四代 Core（“Haswell”）处理器的系统上的 Intel 图形适配器。

FreeBSD 11 对无线网络的支持更广。新的 `iwm`(4) 驱动通过 802.11a/b/g 支持 Intel 3160、3165、7260、7265 和 8260 芯片组的集成无线适配器。这些适配器用于配备第四代及以后 Intel Core 处理器的大多数笔记本电脑。`iwn`(4) 驱动（用于配备早期 Core 处理器的笔记本电脑）现在支持 5GHz 信道和 802.11n。用于 Atheros 适配器的 `ath`(4) 驱动支持更新的支持 802.11n 的适配器，在 station 和 AP 模式下均提供完整 802.11n 支持。用于 Broadcom BCM43xx 无线适配器的 `bwn`(4) 驱动现在支持配备 N-PHY 的设备（BCM4312 和 BCM4321 芯片组）。这些适配器支持 5GHz 信道上的 802.11n。用于 Realtek USB 适配器的 `rsu`(4) 和 `urtwn`(4) 驱动现在在 2.4GHz 信道上完整支持 802.11n。

11 对 UEFI 的支持也有所改进。FreeBSD amd64（又称 x86_64 或 x64）安装介质现在既可从 UEFI 启动，也可从传统模式启动。UEFI 系统现在可以直接从 ZFS 文件系统启动。其他针对 UEFI 和传统系统的启动改进（包括对 ZFS 启动环境和通过 GELI 实现的整盘加密的支持）详见 Allan Jude 在本期中的文章。

bhyve hypervisor 在 FreeBSD 11 中新增多项特性。客户机现在可以使用 UEFI 固件启动，而不必使用用户空间启动加载器。这使支持 UEFI 的客户机操作系统能够使用原生启动流程。此外，bhyve 在使用 UEFI 时支持图形帧缓冲，并为键盘和鼠标输入添加了额外的设备模拟。这些模拟设备由 VNC 服务器提供后端，使客户机操作系统能够使用图形界面。用户通过 VNC 客户端与这些客户机交互。连同其他修复，这些改动使 Microsoft Windows 可以作为 bhyve 虚拟机的客户机运行。此外，FreeBSD 11 的 bhyve 包含 Intel 82545 网络适配器的设备模拟，使不支持 VirtIO 设备的操作系统也能使用网络。特别是 Windows 可以即装即用，安装时不需要额外的 VirtIO 设备驱动。

最后，FreeBSD 11 支持 PCIexpress 原生 HotPlug。这包括处理笔记本电脑中 ExpressCard 适配器的运行时插拔，以及具备 HotPlug 能力的插槽中 PCI-express 适配器的运行时插拔。

## 企业级

FreeBSD 当然不纯粹是桌面操作系统，11 也为企业级用户带来了多项新特性。除了支持 PCI-express HotPlug 外，11 还支持 PCI Single-Root I/O Virtualization（SR-IOV），包括在受支持的设备驱动上创建 virtual function（VF）。这些 VF 可以传递给在 bhyve hypervisor 下执行的虚拟机，使 I/O 请求能够直接访问硬件。

FreeBSD 10 引入的 iSCSI 栈在 11 中获得了多项改进。FreeBSD 现在支持 iSCSI Extensions for RDMA（iSER），提供更高效的零复制 I/O 访问 SCSI 数据缓冲区。`cxgbei`(4) 驱动支持在具备 TOE 能力的 Chelsio 适配器上对 iSCSI 连接进行硬件加速卸载。最后，reroot 工具提供了一种以 iSCSI 根文件系统启动系统的方法。

本地存储栈在 11 中也有一系列改动。FreeBSD 现在包含 zfsd 守护进程，用于处理热备设备的自动激活以及内核未处理的其他 ZFS 相关事件。`sesutil`(8) 工具支持磁盘机柜管理，`mpsutil`(8) 工具支持 LSI Fusion-MPT 2（`mps`(4)）和 Fusion-MPT 3（`mpr`(4)）SAS/SATA 控制器管理。FreeBSD 11 在 CAM 层支持可插拔的磁盘 I/O 调度。11 中包含一个面向 NVMe 磁盘的 CAM 前端，可以替代 `nvd`(4) 驱动，使 NVMe 磁盘能够使用 CAM 特定的行为。FreeBSD 11 还为各类硬件提供了新驱动。OFED Infiniband 栈更新至 2.1 版本，支持 RoCE。`ixl`(4) 驱动支持 Intel XL710 40Gb 以太网适配器，`mlx5en`(4) 驱动支持 Mellanox ConnectX-4 和 ConnectX-4LX 适配器。

## ARM

FreeBSD 10.1 是首个为受支持的 FreeBSD/arm 系统提供发行镜像的版本。FreeBSD 11 从多个方面扩展了对基于 ARM 系统的支持。

首先，ARM 内核现在包含一个导出给所有用户进程的全局共享页。因此，ARM 系统上的用户进程现在使用不可执行栈。此外，用户进程能够在不产生系统调用开销的情况下获取当前时间。

其次，32 位 ARMv6+ 的默认浮点 ABI 从软浮点改为硬浮点，提升了现代处理器的浮点性能。

第三，FreeBSD 内核的虚拟内存系统在 32 位 ARMv6+ 处理器上支持透明的 1MB 大页。如同 x86 上的大页支持一样，这降低了 TLB 未命中的开销。ARM 系统甚至可以用一个大页映射 C 运行时库的文本段（x86 上的文本段过小，无法使用大页映射）。

第四，FreeBSD 11 支持更多系统。新增对若干 Allwinner SoC 的支持，包括 Banana Pi、Cubieboard 1 和 Cubieboard 2。11 还为 PandaBoard 和树莓派 2 系统提供安装镜像。

最后，FreeBSD 11 通过 FreeBSD/arm64 平台新增对 64 位 ARMv8 处理器的支持。11 在 Cavium ThunderX 系统上可即装即用，更多系统的支持将在后续版本中提供。arm64 平台包含一个全局共享页，使 32 位 ARM 平台上的不可执行栈和快速时间查询同样可用。大页支持已经存在于 HEAD 中，将在 FreeBSD 11.1 中提供。FreeBSD 的软件包系统为 FreeBSD/arm64 提供了超过 2 万个预编译软件包，并定期更新。

## RISC-V

RISC-V 是一种新的开源指令集架构。最初源自计算机架构研究，可免费用于各类用途（包括商用 CPU）。FreeBSD 11 包含一个支持 64 位 RISC-V 架构的 FreeBSD/riscv 平台。RISC-V 的内核模式 ISA 仍在变化中，但 FreeBSD 11 支持草案特权 ISA 规范的 1.9 版本。FreeBSD/riscv 在 Spike 模拟器和 QEMU 模拟器中均可启动至多用户模式。

## 对开发者更友好

FreeBSD 11 比以往对开发者更友好。基本系统中的 llvm 工具链（包括 clang 和 lldb）更新至 3.8.0 版本。lldb 现在作为基本系统的一部分，在 FreeBSD/amd64 和 FreeBSD/arm64 上提供。C++ 运行时库（libc++）也已更新，支持 C++14。

整个基本系统现在使用调试符号构建。这些符号可以在安装时或安装后安装。这些符号使开发者能够检查状态并单步执行基本系统库以及应用程序代码。线程支持也有多项改进。POSIX 线程库现在支持进程共享的原语，如互斥锁和条件变量。线程库中还新增了健壮互斥锁的实现。此外，调试子系统的内部改进使多线程进程的调试更稳健。

最后，DTrace 跟踪工具在 11 中支持更多平台，包括 ARM、MIPS 和 RISC-V。

## 结语

过去两年半里，FreeBSD 社区为 FreeBSD 11 投入了大量精力，成果有目共睹。感谢每一位贡献者——无论是测试快照、报告 bug、提交补丁、维护补丁，还是在社交媒体上与用户互动、组织会议等。希望你喜欢 FreeBSD 11，也期待你为 FreeBSD 12 贡献力量！

---

**JOHN BALDWIN** 于 1999 年作为提交者加入 FreeBSD 项目。他在系统的多个领域工作过，包括 SMP 基础设施、网络栈、虚拟内存和设备驱动支持。John 曾任职于 Core 和发行工程团队，并每年春天组织 FreeBSD 开发者峰会。
