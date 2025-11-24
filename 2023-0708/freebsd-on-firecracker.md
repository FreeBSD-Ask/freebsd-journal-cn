# 在 Firecracker 上的 FreeBSD

- 原文链接：<https://freebsdfoundation.org/wp-content/uploads/2023/08/percival_firecracker.pdf>
- 作者：COLIN PERCIVAL
- 译者：ykla & ChatGPT

许多出色的开源软件源于“解决问题”的动机。Firecracker 也是如此：在 2014 年，亚马逊推出了 AWS Lambda 作为“无服务器”计算平台：用户只需提供一个函数，比如十行 Python 代码，Lambda 就会提供在 HTTP 请求到达并调用函数以处理请求并生成响应之间所需的全部基础架构。

为了高效且安全地提供此服务，亚马逊需要能够以最小开销来启动虚拟机。因此，Firecracker 诞生了：这是一个与 Linux KVM 配合工作的虚拟机监视器，用于创建和管理具有最小开销的“microVMs”（微型虚拟机）。

## 为什么在 Firecracker 上使用 FreeBSD？

在 2022 年 6 月，我开始着手将 FreeBSD 移植到 Firecracker 上来运行。我的兴趣受到了几个方面的因素驱动。首先，我一直在加速 FreeBSD 启动过程方面进行了大量工作，我想知道在最精简的虚拟化管理程序下可以达到何种极限。其次，将 FreeBSD 移植到新平台上总是有助于发现 FreeBSD 本身以及这些平台上的错误。第三，目前 AWS Lambda 仅支持 Linux；我一直渴望让 FreeBSD 在 AWS 中更加具有可用性（尽管在 Lambda 使用与否不在我控制之内，但支持 Firecracker 将是一个必要的前提）。然而，最大的原因只是因为它存在。Firecracker 是一个有趣的平台，我想看看自己能否使其运行。

## 启动 FreeBSD 内核

虽然 Firecracker 是针对 Lambda 的需求设计的，即启动 Linux 内核，但自 2020 年起，已有补丁添加了对 PVH 引导模式的支持，即除了“linuxboot”外。FreeBSD 在 Xen 下支持 PVH 引导，因此我决定看看是否能够适用。

在这里，我遇到了第一个问题：Firecracker 可以将 FreeBSD 内核加载到内存中，但找不到开始运行内核的地址（“内核入口点”）。根据 PVH 引导协议，此值在 ELF（可执行和链接格式）文件中的一个特殊元数据中指定。事实证明，有两种类型的 ELF 注释：PT_NOTEs 和 SHT_NOTEs，而 FreeBSD 没有提供 Firecracker 正在寻找的注释类型。我对 FreeBSD 内核链接器脚本所做的微小更改修复了这个问题，现在 Firecracker 能够开始运行 FreeBSD 内核了。

这个过程持续了大约 1 微秒。

## 早期调试

FreeBSD 具有出色的调试功能，但如果内核在调试器初始化或串行控制台设置之前崩溃，你就无法获得太多帮助。在这种情况下，Firecracker 进程退出，告诉我 FreeBSD 系统发生了三重错误，但这就是我所知道的一切。

然而，事实证明，有鉴于一些创意，这已经足够让我开始。如果 FreeBSD 内核执行达到 hlt 指令，Firecracker 进程将继续运行，但使用主机 CPU 时间的 0%（因为它正在虚拟化已停止的 CPU）。因此，我可以通过插入 hlt 指令来区分“FreeBSD 在此点之前崩溃”和“FreeBSD 在此点之后崩溃”——如果 Firecracker 退出，我就能知道它在达到该指令之前崩溃了。因此开始了一个被我称之为“内核二分法”的过程——与其对提交列表进行二分查找以找到引入错误的提交（如 git bisect 中那样），我会通过内核启动代码进行二分搜索，以找到导致 FreeBSD 崩溃的代码行。

## Xen 超级调用

在这个过程中我发现的第一件事是 Xen 超级调用。PVH 引导模式起源于 Xen/PVH 引导模式，而 FreeBSD 的 PVH 入口点实际上是专门用于在 Xen 下引导的入口点——代码对此进行了相当合理的假设，即它确实在 Xen 内部运行，因此可以进行 Xen 超级调用。当然，提供 Firecracker 所使用的内核功能的 KVM（不是 Xen）并不提供这些超级调用；尝试使用其中任何一个都会导致虚拟机崩溃。作为一个原始的解决方法，我仅注释掉了所有 Xen 超级调用；稍后，我添加了代码，通过检查 CPUID 中的 Xen 签名，在进行调用之前检查 Xen 超级调用，例如将调试输出写入 Xen 调试控制台。

然而，有一个 Xen 超级调用提供了必要的功能：检索物理内存映射。（当然，在超级调用器内部，“物理”内存实际上只是虚拟物理内存。它一直这样下去。）在这里，我们受到的启发是，Xen/PVH 事后被宣布为 PVH 引导模式的版本 0：从版本 1 开始，通过 PVH start_info 页传递了指向内存映射的指针（当虚拟 CPU 开始执行时，会提供该指针的寄存器）。我必须编写代码，以利用 PVH 版本 1 的内存映射，而不是依赖于 Xen 超级调用来获得相同的信息，但这并不难。

另一个相关问题涉及 Xen 和 Firecracker 如何在内存中排列结构：在内存中，Xen 首先加载内核，然后将 start_info 页放在末尾，而 Firecracker 则将 start_info 页放在固定的低地址，然后在加载内核之后。这本来没问题，但 FreeBSD 的 PVH 代码——考虑到 Xen——假定 start_info 页之后的内存会被用作临时空间。在 Firecracker 下，这很快就意味着覆盖了初始内核堆栈——这并不是一个理想的结果！通过对 FreeBSD 的 PVH 代码进行更改，将临时空间分配给由超级调用器初始化的所有内存区域，就解决了这个问题。

## ACPI——或者没有

在 x86 平台上，FreeBSD 通常使用 ACPI 来了解（并在某些情况下控制）其运行的硬件。FreeBSD 除了通过 ACPI 发现我们可能通常视为“设备”的东西，如磁盘、网络适配器等，还了解一些基本的东西——如 CPU 和中断控制器。

Firecracker 有意地极简化，不去实现 ACPI，当 FreeBSD 无法确定有多少个 CPU 或者如何找到它们的中断控制器时，会感到不安。幸运的是，FreeBSD 支持历史悠久的 Intel 多处理器规范，通过“MPTable”结构提供了这些关键信息；尽管它不是 GENERIC 内核配置的一部分，但在 Firecracker 中运行时，我们会使用一个经过精简的内核配置，因此很容易添加 device mptable 以利用 Firecracker 提供的信息。

然而……这并没有奏效。FreeBSD 仍然无法找到所需的信息！事实证明，Linux 在查找和解析 MPTable 结构方面存在错误——而 Firecracker 设计用于引导 Linux，在提供 MPTable 的方式上得到了 Linux 支持，但在实际上并不符合标准。FreeBSD 的实现独立编写以遵循标准，既未能找到（位置不正确的）MPTable，也未能解析（无效的）MPTable。

因此，FreeBSD 现在有了一个新的内核选参数：如果你需要与 Linux 的 MPTable 处理具有错误兼容性，可以在内核配置中添加选项 `MPTABLE_LINUX_BUG_COMPAT`。有了这个参数，FreeBSD 成功地在 Firecracker 中进一步引导。

## 串行控制台

Firecracker 提供的一些模拟设备之一（与 Virtio 块和网络设备等虚拟化设备相对）是串口。实际上，在常见的配置中，当你启动 Firecracker 时，Firecracker 进程的标准输入和输出会成为虚拟机的串口输入和输出，使其看起来像是虚拟机操作系统只是在你的 Shell 内部运行的另一个进程（从某种意义上讲，确实如此）。至少，这是它应该工作的方式。

在将 FreeBSD 引导到 Firecracker 内的过程中，我能够启动一个已将根磁盘编译到内核映像中的 FreeBSD 内核——虚拟磁盘驱动程序尚未工作——并读取了内核的所有控制台输出。然而，在所有内核控制台输出之后，FreeBSD 进入了引导过程的用户区域，我得到了 16 个字符的控制台输出，然后就停止了。

有趣的是，我在 10 多年前就见过这个相同的问题，当时是我首次在 EC2 实例上使 FreeBSD 工作。QEMU 中的一个错误导致串口在传输 FIFO 清空时不发送中断；FreeBSD 向串口写入 16 字节，然后不再写入，因为它正在等待从未到达的中断。现代的 EC2 实例运行在亚马逊的“Nitro”平台上，但在早期，它们使用了 Xen，并且使用了 QEMU 的代码来模拟设备。不知何故，在 QEMU 中修复了这个错误十年后，完全相同的错误出现在了 Firecracker 中；但幸运的是，我（当时）加入 FreeBSD 内核的解决方法——`hw.broken_txfifo="1"`——仍然可用，添加了这个加载器可调整值后（由于 Firecracker 直接加载内核，而不经过引导加载器，这意味着将该值编译为内核的环境变量）修复了控制台输出。

然后我发现控制台输入也是无效的：FreeBSD 对我在控制台中键入的任何内容都没有响应。事实上，在我跟踪 Firecracker 进程时，我发现 Firecracker 甚至没有从控制台读取——因为 Firecracker 认为模拟串口上的接收 FIFO 已满。结果证明这是 Firecracker 的另一个错误：在初始化串口时，FreeBSD 用垃圾填充接收 FIFO 以测量其大小，然后通过写入 FIFO 控制寄存器来刷新 FIFO。Firecracker 没有实现 FIFO 控制寄存器，因此 FIFO 保持满状态，并且合理地不尝试读取任何更多的字符放入其中。在这里，我向 FreeBSD 添加了另一个解决方法：如果在我们尝试通过 FIFO 控制寄存器刷新 FIFO 后，LSR_RXRDY 仍然被断言（也就是说，如果 FIFO 没有按要求清空），那么我们现在会继续逐个读取和丢弃字符，直到 FIFO 清空。有了这个解决方法，Firecracker 现在可以认识到 FreeBSD 已准备好从串口读取更多输入，我有了一个可工作的双向串行控制台。

## Virtio 设备

虽然没有磁盘或网络的系统对某些用途可能是有用的，但在我们能够在 FreeBSD 中做很多事情之前，我们需要这些设备。Firecracker 支持 Virtio 块和网络设备，并将它们以 mmio（内存映射 I/O）设备的形式暴露给虚拟机。使这些在 FreeBSD 中工作的第一步：在 Firecracker 内核配置中添加 `device virtio_mmio`。

接下来，我们需要告诉 FreeBSD 如何找到虚拟化的设备。FreeBSD 希望通过 FDT（扁平化设备树）发现 mmio 设备，这是一种在嵌入式系统上常用的机制；但是，Firecracker 通过内核命令行传递设备参数，比如 `virtio_mmio.device=4K@0x1001e000:5`，来传递设备参数。在 FreeBSD 中使这些工作的第二步：编写用于解析这些指令并创建 virtio_mmio 设备节点的代码。（我们创建了设备节点后，FreeBSD 的常规设备探测过程就会启动，内核将自动确定 Virtio 设备的类型并连接适当的驱动程序。）

然而，如果我们有多个设备，例如，一个磁盘设备和一个网络设备——则会出现另一个问题：Firecracker 以 Linux 所期望的方式传递指令，即作为内核命令行上的一系列键值对，而 FreeBSD 将内核命令行解析为环境变量……这意味着如果在命令行上传递了两个 `virtio_mmio.device=` 指令，只会保留一个。为了解决这个问题，我重新编写了早期的内核环境解析代码，通过附加带编号的后缀来处理重复变量：我们会得到一个设备的 `virtio_mmio.device=`，而第二个设备则为 `virtio_mmio.device_1=`。

有了这个，我终于让 FreeBSD 能够引导并发现所有设备了，但磁盘设备还出现了另一个问题：如果我没有完整地关闭虚拟机，在下一次引导时，系统会在文件系统上运行 fsck，并且内核会发生崩溃。事实证明，fsck 是 FreeBSD 中极少数会导致非页面对齐磁盘 I/O 的操作之一，而 FreeBSD 的 Virtio 块驱动在尝试将非对齐的 I/O 传递给 Firecracker 时会导致内核崩溃。

当 I/O 跨越页面边界时——这将发生在大小为页面的未对齐 I/O 上，它们没有对齐到页面边界时——物理 I/O 段通常不会连续；大多数设备可以处理指定要访问的一系列内存段的 I/O 请求。然而，极简化的 Firecracker 不这样做：它只接受单个数据缓冲区——这意味着跨越页面边界的缓冲区无法像其他 Virtio 实现一样简单地被拆分为多个片段。幸运的是，FreeBSD 有专门用于处理设备类似情况的系统：busdma。

这可能是使 FreeBSD 在 Firecracker 中运行的最困难的部分，但经过多次尝试，我认为我终于做对了：我修改了 FreeBSD 的 Virtio 块驱动以使用 busdma，现在非对齐的请求会“弹回”（即通过临时缓冲区进行复制），以符合 Firecracker Virtio 实现的限制。

## 显露出的可优化项目

我在让 FreeBSD 在 Firecracker 中运行起来后，很快就明显的看到有一些可改进的空间。我注意到的第一件事是，尽管我正在测试的虚拟机中有 128 MB 的内存，但系统几乎无法使用，进程经常被杀死，因为系统的内存用完了。top(1) 工具显示，几乎一半的系统内存处于“已绑定”状态，这对我来说似乎很奇怪；因此我进一步调查发现 busdma 为反弹页面保留了 32 MB 的内存。显然，这远远超过所需的量——考虑到 Firecracker 的限制以及反弹页面通常不会连续分配，每个磁盘 I/O 最多应使用单个 4 kB 的反弹页面——通过对 busdma 的补丁，我能将这个内存消耗减少到 512 kB，限制其对仅支持少量 I/O 段的设备的反弹页面保留。

系统变得更加稳定之后，我开始关注引导过程。如果你观看系统引导并且消息突然在滚动过程中停顿，那么在那一点可能发生了减缓引导过程的情况。简单地观察引导过程——以及关机过程——揭示了几个改进：

- FreeBSD 的内核随机数生成器通常从硬件设备获取熵，但在虚拟机中这可能不是一个有效的来源。作为熵的备用来源，在 x86 系统上，我们使用 RDRAND 指令从 CPU 获取随机值；但我们每次请求仅获取很少的熵量，每 100 毫秒只请求一次熵。将熵收集系统更改为请求足够的熵以完全填充 Fortuna 随机数生成器，可以将引导时间缩短 2.3 秒。
- 当 FreeBSD 首次引导时，会记录系统的主机 ID。这通常是通过环境变量 `smbios.system.uuid` 从硬件获取的，引导加载程序根据来自 BIOS 或 UEFI 的信息设置它。然而，在 Firecracker 下，没有引导加载程序——因此也没有提供 ID。我们有一个备用系统，可以在没有有效硬件 ID 的系统上在软件中生成一个随机 ID；但我们也打印了一个警告，并等待 2 秒钟以便用户读取。我将此代码更改为在硬件提供无效 ID 时打印警告并等待 2 秒，但如果硬件压根不提供 ID，则静默且快速进行。
- IPv6 要求系统在使用 IPv6 地址之前等待“重复地址检测”。在 `rc.d/netif` 中，在启动接口后，如果我们的任何网络接口启用了 IPv6，我们都会等待 IPv6 DAD。但有一个问题：我们总是在回环接口上启用 IPv6！我将逻辑更改为仅在除回环接口之外的接口上启用了 IPv6 时才等待 DAD，从而加速引导过程 2 秒钟——如果另一个系统尝试在我们的 lo0 上使用与我们相同的 IPv6 地址，那么我们面临的问题比地址冲突要大得多！
- 在重新启动时，FreeBSD 会打印一条消息（“Rebooting...”），然后等待 1 秒钟“等待 printf 完成并读取”。由于人们通常可以看出系统正在重新启动，所以这似乎最小限度地有用——现在有一个 `kern.reboot_wait_time sysctl`，默认为 0。
- 在关闭或重新启动时，FreeBSD 的 BSP（CPU＃0）会等待其他 CPU 发出停止信号...然后再等待额外的 1 秒钟，以确保它们有机会停止。我删除了额外的一秒等待时间。

一些显而易见的问题得到解决了以后，我使用 [TSLOG](https://www.daemonology.net/papers/bootprofiling.pdf) 并开始查看引导过程的火焰图。出于两个原因，Firecracker 是进行此操作的绝佳环境：首先，极简的环境消除了很多噪音，使得更容易看到剩下的内容；其次，Firecracker 能够非常快速地启动虚拟机，使得能够快速测试对 FreeBSD 内核的更改——通常在不到 30 秒内构建新内核，启动它，并生成新的火焰图。

通过 TSLOG 的调查揭示了许多可用的优化：

- lapic_init 中有一个迭代 100000 次的循环，用于校准 `lapic_read_icr_lo` 的执行时间；将其缩减到迭代 1000 次可减少 10 毫秒。
- ns8250_drain 在每个字符被读取后调用 DELAY；将此更改为仅在 LSR_RXRDY 可用时调用 DELAY，如果已经有字符可以读取，可减少 27 毫秒。
- FreeBSD 使用一个大多数虚拟化程序用于广播 TSC 和本地 APIC 时钟频率的 CPUID 叶；而与 VMWare、QEMU 和 EC2 不同，Firecracker 没有实现这个。为 Firecracker 添加对此 CPUID 叶的支持可将 FreeBSD 引导时间减少 20 毫秒。
- FreeBSD 设置了 kern.nswbuf（用于为各种临时目的分配缓冲区的数量）为 256，而不管系统的大小；将其更改为 32 \* mp_ncpus 可将小型（1 个 CPU）虚拟机的引导时间减少 5 毫秒。
- FreeBSD 的 mi_startup 函数，用于启动机器无关的系统初始化例程，使用冒泡排序对其调用的函数进行排序；尽管在 90 年代，给定在该时刻需要排序的例程数量很少，这是合理的，但现在有 1000 多个这样的例程，而冒泡排序变得很慢。将其替换为快速排序可节省 2 毫秒。（在杂志出版时尚未提交。）
- FreeBSD 的 vm_mem 初始化例程正在为所有可用的物理内存初始化 vm_page 结构。即使在具有 128MB RAM 的相对较小的 VM 上，这意味着要初始化 32768 个这样的结构，并且需要几毫秒。将此代码更改为在内存分配供使用时“懒惰”地初始化 vm_page 结构可节省 2 毫秒。（在出版时尚未提交。）
- Firecracker 通过匿名 mmap 分配 VM 客户端内存，但 Linux 并未设置整个 VM 客户端地址空间的分页结构。结果是，第一次读取任何页面时，将会发生故障，花费大约 20,000 CPU 周期来解决，同时 Linux 映射了一页内存。在 Firecracker 的 mmap 调用中添加 MAP_POPULATE 标志可节省 2 毫秒。（在杂志出版时尚未提交。）

## 当前状态

FreeBSD 在 Firecracker 下引导——并且非常迅速地完成。包括未提交的补丁（对 FreeBSD 和 Firecracker 都是如此），在具有 1 个 CPU 和 128MB RAM 的虚拟机上，FreeBSD 内核可以在不到 20 毫秒的时间内引导；下面是引导过程的火焰图。

![GLD 8~GA{A 3W SW(H)9$Q](https://github.com/FreeBSD-Ask/freebsd-journal-cn/assets/10327999/e4c4c53e-51a7-4021-8e57-71bb93fc5c73)

仍然有工作要做：除了提交上述提到的补丁，并将 PVH 引导模式支持合并到 Firecracker“main”中，还有大量的“清理”工作要做。由于 PVH 引导模式的历史起源于 Xen，用于 PVH 引导的代码仍与 Xen 支持混合在一起；将它们分开将显著使问题简化。同样，目前无法在没有 PCI 或 ACPI 支持的情况下构建 FreeBSD arm64 内核；查找错误的依赖项并删除它们将允许更小的 FreeBSD/Firecracker 内核（并从引导时间中节省几微秒——我们花费了 25 微秒来检查是否需要为 Intel GPU 保留内存）。

更具雄心的是，如果 Firecracker 能够被移植到运行在 FreeBSD 上，那将是很棒的事情——在某种程度上，虚拟机就是虚拟机，虽然 Firecracker 是为了使用 Linux KVM 而编写的，但没有根本性的理由阻止它使用 FreeBSD 的 bhyve 虚拟化器的内核部分。

任何想要在 Firecracker 中尝试 FreeBSD 的人都可以使用 amd64 FIRECRACKER 内核配置构建 FreeBSD 14.0 内核，并从 Firecracker 项目中 checkout [feature/pvh](https://github.com/firecracker-microvm/firecracker/tree/feature/pvh) 分支；或者如果该分支不存在，这意味着代码已合并到 [Firecracker main 中](https://github.com/firecracker-microvm/firecracker)。

如果你在 Firecracker 上尝试 FreeBSD——尤其是如果你最终在生产环境中使用它。请告诉我！我开始这个项目主要是出于兴趣，但如果它最终变得有用，我会很高兴听到这件事的。

---

COLIN PERCIVAL 自 2004 年以来一直是 FreeBSD 开发者，并且在 2005 年至 2012 年间担任该项目的安全主管。在 2006 年，他创立了 Tarsnap 在线备份服务，并继续运营该服务。在 2019 年，为了表彰他在将 FreeBSD 引入 EC2 中的工作，他被命名为亚马逊网络服务英雄。
