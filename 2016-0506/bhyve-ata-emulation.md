# bhyve ATA/ATAPI 仿真

- 原文：[bhyve ATA/ATAPI Emulation](https://freebsdfoundation.org/wp-content/uploads/2016/06/bhyve-ATA-Emulation.pdf)
- 作者：**Teaca Ionut-Alexandru**、**Mihai Carabas**、**Peter Grehan**

bhyve 的 ATA/ATAPI 仿真是更宏大项目的一部分，旨在确保 FreeBSD Hypervisor（bhyve）与旧版本 FreeBSD 客户机的向后兼容性。目前 bhyve hypervisor 为驱动器和 atapi 设备仿真 AHCI 标准。为了支持 FreeBSD 4 等没有 ahci 驱动的客户机，需要仿真 ATA/ATAPI 控制器。

## 引言

本文介绍通过主机 PCI 适配器连接到 PCI 总线的通用 ATA 驱动控制器仿真。ATA 控制器和主机适配器都是我们从头设计和开发的实现的一部分。使用这个仿真器，我们仿真一个 ATA 磁盘控制器，供 bhyve 虚拟机中的客户机 ATA 驱动使用。

目前 bhyve 支持 FreeBSD 8.0 发布以来的任何 i386/amd64 版本。标准 AHCI 模式自 FreeBSD 8.0 起即开箱即用。本文的范围是提供客户机操作系统与 FreeBSD 4/5 等旧版本的兼容性方案。因此，需要在 bhyve hypervisor 中仿真 ATA 主机适配器标准，以支持仅有 ATA 控制器驱动的客户机操作系统。

我们先介绍 ATA 仿真相关的已有工作，以及从头编写此驱动的动机。接着概述 bhyve hypervisor 中的设备仿真。大多数设备在用户空间的 **usr.sbin/bhyve** 中仿真。其他设备，如可编程中断控制器（PIC）和定时器，在内核的 **vmm/io** 中实现。bhyve 中仿真的设备分两类：ISA 和 PCI 设备。ISA 设备包括 UART 控制器和 RTC（实时时钟）控制器。PCI 设备的一部分由 virtio 类构成，包含 block、net 和 rng（来自 **/dev/random** 的随机熵）子类。AHCI 控制器也是 bhyve 中仿真的 PCI 设备。我们的 ATA 主机适配器实现将作为 PCI 设备通过向 PCI 控制器注册来仿真。它同样在 bhyve hypervisor 中仿真。

我们介绍 bhyve 的一些通用信息，以及与 ATA 仿真相关的 PATA、SATA、AHCI、PCI 等技术标准。

ATA（AT Attachment）定义了存储设备内部连接到主机系统的物理、电气、传输和命令协议 [4]。可以是并行 ATA（PATA）或串行 ATA（SATA）接口标准。并行 ATA（PATA）是传统的 AT Attachment，是连接硬盘、软盘驱动器和光盘驱动器等存储设备的接口标准 [7]。串行 ATA 取代了旧的 AT Attachment 标准，相比旧接口有诸多优势：线缆尺寸和成本更小（7 根导线而非 40 或 80 根）、原生热插拔、通过更高信号速率实现更快的数据传输，以及通过可选的 I/O 队列协议实现更高效的传输 [8]。

高级主机控制器接口（AHCI）是串行 ATA 磁盘驱动器控制器的主机适配器。此规范定义了高级主机控制器接口的功能行为和软件接口，它是一种硬件设备，用于软件与串行 ATA 设备之间的接口通信。AHCI 是 PCI 类设备，在系统主机内存和串行 ATA 设备之间传输数据 [1]。

并行 ATA 协议也有主机适配器控制器接口。ATA/ATAPI 主机适配器标准规定了使用直接内存访问协议的主机系统与存储设备之间的 AT Attachment 接口。AT Attachment 接口可用于任何具有 PCI 总线且存储设备连接到处理器的主机系统 [2]。

## bhyve Hypervisor 结构概览

bhyve 是 BSD hypervisor 的缩写，是被引入 FreeBSD 操作系统的 hypervisor/虚拟机管理器。它类似于 Linux KVM，运行在宿主操作系统上，依赖 Intel VTx、扩展页表和 VirtIO 网络及存储驱动等现代 CPU 特性。

bhyve 由 vmm.ko 可加载内核模块、libvmmapi 库以及 bhyve、bhyveload 和 bhyvectl 工具组成。使用这些工具前，必须先加载 vmm.ko 模块。bhyveload 工具帮助从磁盘镜像加载 FreeBSD 内核。例如，`/usr/sbin/bhyveload -m 256 -d ./vm0.img vm0`。它会显示 FreeBSD 加载器界面，你应该能看到设备 **/dev/vmm/vm0**。

bhyve 工具用于启动具有 2 个 vCPU、与前面相同的 256M 内存和 tap0 网络接口的虚拟机。

```sh
/usr/sbin/bhyve -c 2 -m 256 -A -H -P -s 0:0,hostbridge -s 1:0,virtio-net,tap0 -s 2:0,ahci-hd,./vm0.img -s 31,lpc -l com1,stdio vm0
```

虚拟机关闭后，可用以下命令回收资源：

```sh
/usr/sbin/bhyvectl --destroy --vm=vm0
```

bhyve 进程启动时在 init_pci 函数中初始化 PCI 控制器。此函数开始总线枚举以查找所有 PCI 设备。它遍历每个总线、每个插槽、每个功能，查找是否存在 PCI 设备。例如，如果 bhyve 程序的输入参数为 `3:0,ahci-hd,./diskdev`（即 bnum=0、snum=3、fnum=0），PCI 控制器将此组合关联到自定义设备，并将其映射到 pci_devemu 结构，该结构包含指向初始化其 PCI 设备回调的指针，本例中是 `pci_ahci_hd_init` 函数。调用此函数时，通过向 pci_devinst 结构提供标识信息（厂商和设备信息，以及类、子类和 progif 信息）来初始化 ahci 设备。

PCI 仿真器将（总线、插槽、功能）组合映射到 pci_devinst 结构，这样 pci_devinst 结构中的标识信息就属于设备控制器。此外，`pci_ahci_hd_init` 回调分配 BAR 寄存器，baridx = 5，type = PCIBAR MEM32。客户机操作系统中的 ahci 驱动通过向 PCI 控制器写入配置命令，对基地址寄存器编程，告知 ahci 设备其地址映射。

## 相关工作

bhyve hypervisor 中没有其他仿真 ATA 主机适配器标准的相关实现。GXemul 框架中实现了 ATA 控制器仿真器，该框架支持全系统计算机架构仿真。然而，由于使用不同的应用程序接口与系统其余部分通信，且使用 C++ 语言编写，几乎不可能将其移植到 bhyve 源代码树。但我们可以使用此软件更好地理解其中实现的 ATA 协议，如 PIO、DMA，或有关 ATA 命令的其他信息。

bhyve 中实现了高级主机控制器接口标准的仿真。为了理解 AHC 标准仿真中使用的机制，需要对此实现进行文档说明。

为捕获发送到 AHCI 控制器的主机命令，bhyve 中的 pci_ahci 模块注册了两个处理程序，`pci_ahci_write` 和 `pci_ahci_read`，每当主机软件试图通过 BAR 寄存器寻址 ahci 设备时就会调用它们。基本上，这些回调从 baridx = 5 寄存器计算的地址加上偏移量读/写值。这些回调的实现取决于接口控制器（ahci/ata/atapi）。它们通过访问 prdt 中的 iovec 数组并写入 diskdev 文件描述符来仿真。命令完成时，会调用 ioctl 系统调用仿真中断。I/O 请求通过这些回调创建。blockif_req 请求由 blockif 线程在 `ata_ioreq_cb` 回调中处理。`ata_ioreq_cb` 例程通过对 vmm 的 ioctl 系统调用完成。

对 I/O 请求的操作有两个例程，`blockif_dequeue` 和 `blockif_enqueue`。首先有一个 I/O 请求元素数组和空闲/使用队列，组合在 blockif_ctxt 结构中。`blockif_dequeue` 例程从 blockif 线程上下文调用，从使用队列尾部取出第一个元素，并更新空闲/使用队列。`blockif_enqueue` 获取一个空闲元素，用 blockif_req 请求填充它，并相应更新空闲/使用队列。

DMA 读/写请求由 `ahci_handle_dma` 实现处理，它基于物理区域描述符表（prdt）构建 iovec。这些 iovec 请求在 `blockif_proc` 中处理，后者使用 `pwritev` 系统调用将 iovec 数组写入 diskdev 文件描述符。DMA 命令以触发中断结束。

## 架构

本节聚焦实现基础的最重要设计概念。首先解释主/次和主/从关系，接着介绍仿真器接口涉及的寄存器描述，最后介绍驱动与仿真器通信中使用的基于中断的机制，这也是 ATA 仿真器采用事件驱动架构的原因。

### 主驱动器与从驱动器

在介绍 ATA 控制器架构之前，先看看主/次和主/从关系，以及 ATA 仿真器的配置方式。

大多数主板有两个 IDE 接口（主和次），也称为通道。主和次 IDE 通道只能连接 ATA，这意味着 IDE 只支持 ATA/ATAPI 驱动器。每个接口可支持两个设备，总共最多四个驱动器。两个驱动器需要自行决定如何共享同一 ATA 通道。为此，每个通道上的一个驱动器被指定为“主”，另一个驱动器（如果存在）被指定为“从”。因此我们有以下可能：

- 主通道主驱动器
- 主通道从驱动器
- 次通道主驱动器
- 次通道从驱动器

每个驱动器可以是 ATA 或 ATAPI。

我们的 PCI ATA 适配器仿真 FreeBSD 代码库中 **sys/dev/ata/ata-pci.c** 下的 ATA 旧版驱动。对 ATA 旧版驱动调查后，我们发现它只支持一个通道，即只有两个驱动器。更多细节请参阅 **sys/dev/ata/ata-pci.c** 驱动设备中的 `int_ata_legacy(device_t dev)` 函数。

考虑到这些，我们选择使用以下参数配置 ATA 仿真器：

```sh
-s N:M,ata-hd,X,./DISK_MASTER,./DISK_SLAVE
```

或

```sh
-s N:M,ata-hd,X,./DISK_MASTER
```

其中 N:M 是 PCI 插槽信息，X 为 0 或 1（主或次通道），后跟磁盘名称，第一个是主驱动器，第二个是从驱动器。也可以只有一个磁盘表示主驱动器。无论哪种情况，仿真器实现都支持两个通道，并为两个通道分别维护不同的数据结构，即使客户机旧版驱动只使用一个（主或次）。

### PCI 和 I/O 寄存器描述

为仿真 ATA 控制器磁盘，应实现两组寄存器，因为控制器通过 PCI 总线仿真，客户机驱动发出的命令通过此 PCI 接口提供。适配器寄存器集合表示 PCI 空间配置，客户机驱动在枚举阶段后读取以检测控制器配置详情。这些寄存器的配置详见本文的实现部分。

ATA 控制器还包含一组寄存器（ATA 寄存器）。基本上，这组寄存器由客户机驱动配置以与驱动控制器通信。仿真器的实现应提供此寄存器接口，以及维护控制器状态和仿真驱动命令所需的数据结构。ATA 控制器寄存器分为两类。第一类是 ATA 总线主控寄存器。总线主控 ATA 功能使用 16 字节的 I/O 空间。所有总线主控 ATA I/O 空间寄存器可作为字节、字或双字访问。这些寄存器的基地址是 PCI BAR 4 [3]。第二类是 ATA 通道寄存器。

命令块寄存器用于向设备发送命令或从设备发布状态。这些寄存器包括 LBA High、LBA Mid、Device、Sector Count、Command、Status、Features、Error 和 Data 寄存器。控制块寄存器用于设备控制和发布备用状态。这些寄存器包括 Device Control 和 Alternate Status 寄存器 [5]。主通道的这些寄存器由 BAR0 和 BAR1 寄存器寻址，次通道使用 BAR2 和 BAR3。

### 中断

ATA 仿真器的设计基于事件驱动架构，事件代表仿真器寄存器上的写操作，即客户机驱动命令。这些命令被解释和处理后，仿真器状态相应更新。命令完全仿真后，通过触发中断通知客户机驱动。仿真器适配器支持符合主和次通道地址的两个通道，每个通道有独立的 IRQ。并行 ATA 控制器使用 14 和 15 号 IRQ。仿真器在初始化阶段保留这些 IRQ 编号，并在每个命令完成时触发 IRQ。

### 块设备仿真

为更好地管理读写调用等磁盘操作，ATA 仿真器使用 bhyve 中块设备仿真的实现。每个磁盘驱动器关联一个 blockif_ctxt 结构实例。基本上，ATA 仿真器最多支持两个磁盘驱动器——主驱动器和从驱动器。因此，每个驱动器分配一个 blockif_ctxt 结构。除了选择此 API 的设计原因外，块模型还有额外的线程，用于在其自身上下文中执行读写请求。读/写操作的公共 API 通过向块设备队列提交块请求来工作，这些请求在块上下文中拉取并执行。我们希望延迟 I/O 请求在块上下文中的执行，因为虚拟机循环——CPU 指令运行的地方——是单线程的。如果 I/O 操作在虚拟机循环上下文中执行……

> 注：原文 PDF 在此处之后仍有内容。文章继续介绍 ATA 仿真器的实现细节。完整文章请参阅原版 PDF：<https://freebsdfoundation.org/wp-content/uploads/2016/06/bhyve-ATA-Emulation.pdf>
