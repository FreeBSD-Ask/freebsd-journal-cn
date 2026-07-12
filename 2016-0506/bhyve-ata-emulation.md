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

为更好地管理读写调用等磁盘操作，ATA 仿真器使用 bhyve 中块设备仿真的实现。每个磁盘驱动器关联一个 blockif_ctxt 结构实例。基本上，ATA 仿真器最多支持两个磁盘驱动器——主驱动器和从驱动器。因此，每个驱动器分配一个 blockif_ctxt 结构。除了选择此 API 的设计原因外，块模型还有额外的线程，用于在其自身上下文中执行读写请求。读/写操作的公共 API 通过向块设备队列提交块请求来工作，这些请求在块上下文中拉取并执行。我们希望延迟 I/O 请求在块上下文中的执行，因为虚拟机循环——CPU 指令运行的地方——是单线程的。如果 I/O 操作在虚拟机循环的同一上下文中执行，整个系统会卡住。

### LBA 28 位寻址

逻辑块寻址通过对扇区进行线性映射来定义磁盘设备上数据的寻址。这意味着块通过整数索引定位，第一个块是 LBA 0，第二个是 LBA 1，以此类推。读/写命令无论使用 PIO 还是 DMA 数据传输协议，都使用 28 位来寻址一个扇区。因此，28 位寻址支持的最大尺寸是 2^28 × 512 字节 = 128 GB，因为一个逻辑扇区的尺寸是 512 字节。为支持 LBA 28 位寻址，ATA 标准使用 LBA 组寄存器和 Device 寄存器中的前 4 位，映射如下：LBA Low=LBA(7:0)、LBA Mid=LBA(15:8)、LBA High=LBA(23:16)、Device(3:0)=LBA(27:24)。

## 实现

本节从 ATA 仿真器的初始化阶段开始，接着介绍一些软件协议（如复位、PIO 输入/输出、DMA），以及实现中如何处理主/从设备寻址。最后介绍目前已实现的 ATA 命令。

### 初始化

ATA 适配器的初始化在 `pci_ata_init` 函数中开始，打开后备文件作为磁盘存储。仿真器将处理的 PIO 和 DMA 命令会导致对后备映像文件描述符的读写调用。每个 IDE 控制器作为 PCI 总线上的一个设备出现。如果类代码是 0x01（大容量存储控制器）且子类代码是 0x1（IDE），则此设备是 IDE 设备。PCI 适配器实现了 PCI 标准类型配置头寄存器集的一个子集。首先，`PCIR_DEVICE` 和 `PCIR_VENDOR` 寄存器分别设置为 0x8211 和 0x1283——一个 Waldo ATA 控制器。PCI 类代码必须通过设置 Base-class、Sub-class Codes 为以下值来配置：01h – Mass Storage 和 01h – IDE。编程接口代码应正确处理，设置 `PCIP_STORAGE_IDE_MASTERDEV` 位以指示适配器可进行总线主控操作，并设置 `PCIP_STORAGE_IDE_MODEPRIM` 或 `PCIP_STORAGE_IDE_MODESEC` 位以选择 ATA 通道。在 ATA 主机仿真器的初始化函数中，必须分配 BAR 寄存器。IDE 设备只使用六个 BAR 中的五个。PCI BAR0 和 BAR2 表示主和次通道的基地址。PCI BAR1 和 BAR3 表示主和次通道控制寄存器的基地址。PCI 基地址寄存器 BAR4 是 ATA 总线主控 I/O 寄存器的基地址。初始化通过为每个 ATA 通道保留适当的 IRQ 编号完成。

### 设备寻址注意事项

当向两个设备的寄存器写入值时，主机通过 Device 寄存器中的 DEV 位区分两者。数据在先前从主机传输过来的命令的指引下，并行地传输到或自主存至设备缓冲区。设备执行所有必要的操作，将数据正确写入或读出介质。当两个设备连接在同一条线上时，命令并行写入两个设备，除 EXECUTE DEVICE DIAGNOSTIC 命令外，只有被选中的设备执行命令。写入设备控制寄存器时，无论选中哪个设备，两个设备都会响应。当 DEV 位清零时，选中设备 0。当 DEV 位置一时，选中设备 1。当两个设备连接到线缆时，一个设为设备 0，另一个设为设备 1 [6]。

### 软件复位协议

在某些时刻，客户机驱动会对 ATA 控制器进行软件复位。一种情况是客户机驱动在通道中寻找 ATA/ATAPI 存在的迹象。为复位控制器，驱动在 `ATA_CONTROL` 寄存器中写入 `ATA_A_RESET` 位。为仿真此操作，ATA 仿真器在 `ATA_ERROR` 寄存器中设置 `ATA_E_ILI` 位，在 `ATA_STATUS` 寄存器中设置 `ATA_S_READY` 位。更多细节和更好理解，请查看 **sys/dev/ata/ata-lowlevel.c** FreeBSD 驱动中的 `ata_generic_reset` 函数。

### PIO 数据输入命令协议

PIO 数据输入协议用于将一个或多个数据块从磁盘设备传输到主存。它与使用此协议的命令密切相关，因为这些命令负责准备 PIO 数据缓冲区。此协议的使用对 ATA 驱动是透明的，因为它只需为特定 PIO 命令指定参数。因此有一个包含使用此协议的命令的 PIO 数据输入类。此类命令包括：IDENTIFY DEVICE、READ MULTIPLE、READ SECTOR(S)。当 ATA 客户机驱动发出其中一个 PIO 命令后，仿真器从块磁盘读取数据到 PIO 缓冲区，并向主机发送中断，表示数据已准备好传输。此时，主机通过轮询 DATA 寄存器开始传输数据，每次读取获得 4 字节。当缓冲区为空时，PIO 命令完成。主机为每个传输的块获得中断，默认为 128 个扇区。如果总扇区数不能被块计数整除，仿真器在最后一个不完整块传输完成后中断。

### PIO 数据输出命令协议

PIO 数据输出协议用于将一个或多个数据块从主存传输到磁盘设备。与 PIO 数据输入协议（数据缓冲区由仿真器在传输开始前准备）不同，主机将数据写入缓冲区。它与使用此协议的命令密切相关，因为这些命令负责传输参数（如扇区数和块磁盘中的偏移量）。有一个包含 WRITE MULTIPLE、WRITE SECTOR(S) 等使用此协议的命令的 PIO 数据输出类。发出其中一个命令后，仿真器保存传输参数（扇区数和偏移量），并等待主机传输数据而不发送中断。ATA 驱动通过轮询 DATA 寄存器开始传输数据到 PIO 缓冲区，每次写入添加 4 字节。当总扇区数写入块磁盘后，PIO 命令完成。对于每个传输的块，仿真器将数据写入块磁盘并向主机发送中断。如果总扇区数不能被块计数整除，仿真器在最后一个不完整块传输完成后中断。

### DMA 命令协议

DMA 协议用于将一个或多个数据块从主存传输到磁盘设备，或从磁盘设备传输到主存。它与使用此协议的命令密切相关，因为这些命令负责传输参数（如扇区数和块磁盘中的偏移量）。有一个包含 READ DMA、WRITE DMA 等使用此协议的命令的 DMA 类。在进一步说明之前，我们先介绍 DMA 事务的概念。DMA 事务基本上包含三部分：READ/WRITE DMA 命令、启动，以及最终的 DMA 过程停止。发出其中一个命令后，主机驱动通过向仿真器提供物理区域描述符表（PRDT）的地址并设置 ATA 总线主控命令寄存器中的 Start/Stop Bus Master 位来启动 DMA 事务。

仿真器获取客户机内存空间中的物理地址，需要将其转换到 bhyve 进程内存空间。为此，仿真器使用 `paddr_guest2host` 函数，该函数以客户机地址为参数，返回指向主机内存空间的指针。PRD 表包含若干物理区域描述符（PRD），描述参与数据传输的客户机内存区域。每个 PRD 条目有 8 字节，指定数据将传输到/来的位置。前 4 字节表示物理客户机内存区域的地址；接下来 2 字节指定要传输到/从该区域的字节数。条目的物理地址使用 `paddr_guest2host` 以与 PRDT 地址相同的方式转换到 bhyve 地址空间。如果最后一个字节的第 7 位置位，则表示表结束。此时，仿真器遍历条目，准备将由块实例执行的块请求。块请求基本上包含一个 iovec 结构数组，每个 iovec 结构指示 PRD 表中的一个条目。块请求准备好后，在块实例的上下文中执行，即对与块关联的文件描述符进行写或读操作。客户机驱动通过触发中断收到传输完成通知，因为它必须通过清除 ATA 总线主控命令寄存器中的 Start/Stop Bus Master 位来发起事务的最后阶段。此时，仿真器验证事务状态，设置状态寄存器以指示传输是否成功完成，并最终将事务标记为已完成。

### 命令描述

在 FreeBSD 中，ATA 命令在 **sys/dev/ata/ata-lowlevel.c** 驱动中实现。此驱动直接控制我们的 ATA 控制器仿真器。为更好地理解命令结构，我们查看了客户机驱动中与 ATA 命令相关的一些函数：`ata_generic_command`、`ata_tf_write`、`ata_wait`。

ATA 命令首先选择主/从驱动器，等待控制器清除 `ATA_S_BUSY` 寄存器。准备好发出命令时，驱动通过在 `ATA_CONTROL` 寄存器中设置 `ATA_A_4BIT` 位来启用中断，并继续进行一些写寄存器操作，如 `ATA_FEATURE`、`ATA_COUNT`、`ATA_SECTOR`、`ATA_CYL_LSB`、`ATA_CYL_MSB`、`ATA_DRIVE`，以配置命令参数。实际上，ATA 命令通过向 `ATA_COMMAND` 寄存器写入代码来向控制器发出。仿真器将这些参数保存到其内部数据结构中，并用于确定命令的含义。命令完全仿真后，应触发中断通知客户机驱动。

ATA 仿真器中实现的命令满足 ATA 6 标准支持的通用特性集命令。下面描述一些已实现的 ATA 命令。

- **IDENTIFY DEVICE**：此命令使主机能够从设备接收参数信息。设备将 BSY 位置 1，准备向主机传输 256 字的设备识别数据，将 DRQ 和 READY 位置 1，清除 BSY 位为零，并断言 INTRQ。主机然后可以使用 PIO 数据输入协议通过读取数据寄存器传输数据。数据使用 128 次连续的字传输。字内字节顺序在驱动和仿真器内存之间不同。例如，要向主机内存发送 FaK3 MoDeL IDA diSk 字符串，仿真器准备 aF3KM DoLeI ADdSi k 字符串。使用此命令，主机获知磁盘设备的 CHS 参数（如柱面数、磁头数和扇区数）、型号名称、序列号和固件版本，以及支持的能力。ATA 仿真器支持的能力包括：Multi Word DMA2 和 PIO4 数据传输协议（使用 28 位 LBA 寻址）、读写校验支持和缓存刷新。
- **READ MULTIPLE**：此命令由 ATA 驱动发出，使用 PIO 数据输入协议读取数据。它指定要传输的扇区数和使用 LBA 28 位寻址模式的块磁盘偏移量。仿真器从磁盘读取数据，准备 PIO 缓冲区，标记 PIO 读命令进行中，并向主机发送中断，表示数据已准备好。主机收到中断后，开始传输数据。
- **WRITE MULTIPLE**：此命令由 ATA 驱动发出，使用 PIO 数据输出协议写入数据。它指定要传输的扇区数和使用 LBA 28 位寻址模式的块磁盘偏移量。仿真器将命令参数保存到 PIO 设置中，标记 PIO 写命令进行中，并等待主机传输数据。
- **READ AND WRITE DMA**：这些命令由 ATA 驱动发出，使用 DMA 数据协议读/写，它们表示 DMA 协议的第一阶段。协议指定要传输的扇区数和使用 LBA 28 位寻址模式的块磁盘偏移量。仿真器将命令参数保存到 DMA 设置中（包括操作方向——读/写），并标记 DMA 事务为已启动。然后主机准备 PRD 表并激活 DMA 事务。
- **FLUSH CACHE**：此命令是主机用来请求设备刷新写缓存的非数据 ATA 命令。仿真器基本上向块设备创建一个刷新请求，刷新与磁盘驱动器关联的文件描述符。

## 场景与结果

使用 `-s 3:0,ata-hd,0,./diskdev_ata,./diskdev_ata_slave` 作为 bhyve 应用程序的输入，客户机 ATA 旧版驱动打印以下输出（见代码清单 1）。

我们观察到 ATA 控制器被 atapci 驱动成功识别，BAR 地址被正确分配（BAR1 = 0x2020-0x2027、BAR2 = 0x2028-0x202b、BAR3 = 0x170-0x177、BAR4 = 0x376、BAR5 = 0x2040-0x204f）。两个 ATA 通道由 ata0 和 ata1 驱动处理。由于输入字符串中指定了两个磁盘驱动器（第一个为主驱动器，第二个为从驱动器），有两个 ada 驱动实例——ada0 和 ada1。

因此，在 FreeBSD 安装程序的分区阶段，有两个磁盘可供安装 FreeBSD 虚拟机，且两者都受支持（见代码清单 2）。ATA 仿真器实现的型号是 ATA-6 设备，支持 PIO 和 WDMA2 数据传输协议。需要强调的是，16.700MB/s 的速度不是客户机操作系统与仿真器之间数据传输的实际速度，而是 WDMA2 标准要求的速度。

```sh
atapci0: <ITE IT8211F UDMA133 controller> port 0x2020-0x2027,0x2028-0x202b,0x170-0x177,0x376,0x2040-0x204f at device 3.0 on pci0
ata0: <ATA channel> at channel 0 on atapci0
ata1: <ATA channel> at channel 1 on atapci0
ada0 at ata0 bus 0 scbus0 target 0 lun 0
ada0: <BHYVE ATA IDE DISK 1.0> ATA-6 device
ada0: Serial Number 123456
ada0: 16.700MB/s transfers (WDMA2, PIO 65536bytes)
ada0: 8192MB
ada0: (16777216 512 byte sectors: 16H 63S/T 16644C)
ada0: Previously was known as ad0
ada1: at ata0 bus 0 scbus0 target 1 lun 0
ada1: <BHYVE ATA IDE DISK 1.0> ATA-6 device
ada1: Serial Number 123456
ada1: 16.700MB/s transfers (WDMA2, PIO 65536bytes)
ada1: 8192MB
ada1: (16777216 512 byte sectors: 16H 63S/T 16644C)
ada1: Previously was known as ad1
```

代码清单 1：ATA 旧版驱动日志

```sh
Select the disk on which to install FreeBSD.
vtbd0 654 MB Disk
ada0 8.0 GB ATA Hard Disk (BHYVE ATA IDE DISK)
ada1 8.0 GB ATA Hard Disk (BHYVE ATA IDE DISK)
```

代码清单 2：FreeBSD 安装程序——分区

目前，客户机 FreeBSD 虚拟机可以安装在 ada0 或 ada1 上，并使用 ATA 仿真器无限制地运行。唯一的限制是磁盘尺寸，由于 LBA 28 位寻址，最大为 128 GB。

尽管虚拟机在 ATA 磁盘上的完整安装证明了 ATA 仿真器的正确性，但仍进行了严格验证的测试。使用与代码清单 3 类似的方法，改变 BLOCK_SIZE 和 MAX_SECTORS 变量，以不同的块尺寸覆盖整个磁盘。所有测试均通过，证明了实现的正确性。

```sh
dd bs=$BLOCK_SIZE count=1 if=/dev/random of=tests/testX
dd bs=$BLOCK_SIZE count=1 if=tests/testX of=/dev/ada1 oseek=$MAX_SECTORS
reboot
dd bs=$BLOCK_SIZE count=1 if=/dev/ada1 of=out/testX iseek=$MAX_SECTORS
diff out/testX tests/testX
```

代码清单 3：ATA 测试

如前所述，ATA-6 标准使用 DMA Multi-word 2 指定的最大传输速率为 16.7MB/s。然而，这个值似乎小于我们仿真器的实际速度，因为它是针对硬件设备的。为获取 ATA 仿真器的传输速率，我们在运行仿真设备的虚拟机内使用 diskinfo 工具。使用 `diskinfo -t /dev/ada1` 命令（其中 **/dev/ada1** 是我们实现仿真的 ATA 磁盘），得到以下结果（见代码清单 4），表明传输速率超过 100MB/s，已经足够。ATA-6 已发展到 Ultra DMA，数据传输速率达 100MB/s，与我们使用 WDMA2 的仿真器相当。因此，我们的实现符合 ATA-6 要求，无需开发 Ultra DMA 特性。

```sh
outside: 102400 kbytes in 0.719681 s = 142285 kB/s
middle: 102400 kbytes in 0.796385 s = 128581 kB/s
inside: 102400 kbytes in 0.781721 s = 130993 kB/s
```

代码清单 4：传输速率

尽管 AHCI 和 ATA 磁盘驱动控制器完全不同，且不使用相同的传输数据协议，但测量 AHCI 模块的性能并将其与我们的 ATA 仿真器进行比较仍然很有意义。当我们在 AHCI 驱动控制的分区上使用 diskinfo 工具执行相同过程时，得到以下传输速率（见代码清单 5）。

```sh
outside: 102400 kbytes in 0.326701 s = 313436 kB/s
middle: 102400 kbytes in 0.339824 s = 301332 kB/s
inside: 102400 kbytes in 0.348496 s = 293834 kB/s
```

代码清单 5：AHCI 传输速率

这些结果是预期的，因为 AHCI 标准使用 UDMA6 数据传输协议，对硬件设备要求 300MB/s 的传输速率，远高于 WDMA2 数据协议规定的 16.7MB/s。但 AHCI 仿真具有更好传输速率的主要原因是它使用 LBA 48 位寻址模式，这使得每次 DMA 事务可以传输更多数据扇区（65536），减少了主机驱动与磁盘仿真器之间通信的开销。

## 结论与后续工作

本文介绍了一个在 PCI 附件下仿真 ATA 磁盘驱动控制器的模型。我们将其集成到 bhyve 中，并实现了 ATA-6 标准支持的通用特性集命令。实现经过验证测试，并测量了性能，与 AHCI 仿真进行了比较。还有另一种选择，即 ATA 仿真器可以作为 PCI-ISA 总线的子设备。对于旧版操作系统，ATA 的实用性将大大提高，因为引导加载器能够使用它，因为 I/O 端口和 IRQ 将位于固定位置。后续工作旨在实现作为 PCI-ISA 附件的 ATA 仿真，并将其合并到最终代码库中。

## 参考文献

[1] J. Boyd. Serial ATA Advanced Host Controller Interface, page 9. Intel Corporation, Hillsboro, Oregon. (2008)

[2] T. Goodfellow. ATA/ATAPI Host Adapters Standard, page 15. Pacific Digital Corporation, Irvine, California. (2003)

[3] T. Goodfellow. ATA/ATAPI Host Adapters Standard, page 11. Pacific Digital Corporation, Irvine, California. (2003)

[4] P. T. McLean. Information Technology–AT Attachment with Packet Interface–6, page 16. American National Standards Institute, New York, New York. (2002)

[5] P. T. McLean. Information Technology–AT Attachment with Packet Interface–6, page 63. American National Standards Institute, New York, New York. (2002)

[6] P. T. McLean. Information Technology–AT Attachment with Packet Interface–6, pages 56–57. American National Standards Institute, New York, New York. (2002)

[7] Wikipedia. Parallel ATA–Wikipedia. <http://en.wikipedia.org/wiki/Parallel_ATA>, 2015. (Online; accessed February 14, 2015)

[8] Wikipedia. Serial ATA–Wikipedia. <http://en.wikipedia.org/wiki/Serial_ATA>, 2015. (Online; accessed February 14, 2015)

---

**Teaca Ionut-Alexandru**

Teaca Ionut-Alexandru 毕业于布加勒斯特理工大学，目前是专注于并行与分布式计算机系统的二年级硕士研究生。他的系研究项目是在 bhyve 中与 Mihai Carabas 一起进行 ATA/ATAPI 仿真工作。

**Mihai Carabas**

Mihai Carabas 是布加勒斯特理工大学研究虚拟化的博士生。他为 FreeBSD 项目和 DragonFlyBSD 虚拟化代码做出了贡献。

**Peter Grehan**

Peter Grehan 是 FreeBSD 提交者，从 DEC Ultrix 时代起就以某种形式使用 BSD 衍生操作系统。他与 Neel Natu 共同开发并维护 bhyve hypervisor。
