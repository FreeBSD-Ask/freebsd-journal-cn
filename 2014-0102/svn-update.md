# SVN 动态

作者：Glen Barber

FreeBSD 10-RELEASE 发布周期正在高速推进，9.2-RELEASE 正式发布后，10 成为发布工程团队的主要焦点。

除 bug 修复和稳定性增强外，FreeBSD 10-RELEASE 还包含一系列令人兴奋的新特性。

## 虚拟机的 VirtIO 支持

在虚拟机中，hypervisor 传统上会模拟真实的物理设备提供给虚拟机。模拟原始设备往往效率低下，常导致虚拟化环境中的输入/输出性能损失。

VirtIO 是针对虚拟机中半虚拟化输入/输出设备的规范。VirtIO 模块在虚拟机和 hypervisor 之间提供共享内存传输。此共享内存传输称为“virtqueue”。

VirtIO PCI 驱动创建模拟的 PCI 设备，然后提供给虚拟机。模拟的 PCI 设备使用 virtqueue 直接访问分配给设备的内存，从而提升虚拟化环境中的性能。

VirtIO 最初为 Linux KVM 开发，但后来适配到其他虚拟机 hypervisor，如 BHyVe、VirtualBox 和 Qemu。

VirtIO 支持在 r227652 中加入。

## BHyVe

BHyVe 是 BSD Hypervisor，由 Peter Grehan 和 Neel Natu 开发。BHyVe 的设计目标是在 FreeBSD 上提供轻量级半虚拟化环境。

BHyVe 需要支持 VT-x 和扩展页表（EPT）的 Intel CPU。这些特性在所有 Nehalem 及更新 CPU 上都有，但 Atom CPU 不支持。

BHyVe 在 r245652 中进入 FreeBSD 10-CURRENT。

## RDRAND

RDRAND 是 Intel CPU 指令集，用于访问硬件随机数生成器。

软件随机数生成器使用从各种来源获取的种子熵。例如，以太网接口和软件中断处理程序可作为随机数生成的熵源。

硬件随机数生成器通过物理方式收集熵，例如设备内的热“噪声”。通过使用此类不可预测的物理熵源，硬件随机数生成器可收集更高级别的随机性，进而意味着更大可能性的真正随机数。将设备直接置于 CPU 上减少了向系统添加额外硬件设备的需求。

Intel 随机数生成器在“Bull Mountain”CPU 上可用，存在于 Ivy Bridge 及更新型号上。

`rdrand` 驱动在 r240135 中加入 FreeBSD 10-CURRENT。

## PF 中的多处理器支持

自最初从 OpenBSD 导入以来，PF（Packet Filter）的性能限制之一是它只能绑定到单个 CPU。这意味着在多处理器系统上，PF 无法利用额外的 CPU，也就是说 PF 在 2 核或 24 核机器上不一定能表现出任何性能提升。

FreeBSD 10-CURRENT 中引入的工作为 PF 增加多处理器支持，引入了细粒度锁。这让 PF 能利用系统上的多个 CPU，显著提升性能。

PF 的多处理器支持在 r240233 中引入。

源自 OpenBSD 的 pf 防火墙获得升级，支持细粒度锁和更好的多 CPU 机器利用率，性能显著提升。

## 磁盘驱动中的非映射 I/O

FreeBSD 内核在内核页表中映射 I/O 缓冲区。在多核系统上，由于此全局映射，必须在所有 TLB（转译后备缓冲器）上刷新映射。当系统上的核心数量增加时，会出现性能瓶颈，因为在缓冲区创建和销毁期间，发起线程必须等待系统上的所有其他核心执行。

FreeBSD 10 引入非映射 I/O 缓冲区，消除了在缓冲区创建和销毁时执行转译后备缓冲器 shootdown 的需求，可在 I/O 密集型工作负载上消除高达 30% 的系统时间。

非映射 I/O 支持最初在 r248508 中为 **ahci(4)** 和 **sdhci(4)** 驱动引入。后续修订为其他驱动提供了支持。

## 树莓派和 BeagleBone 支持

FreeBSD 10 可在树莓派、BeagleBone 和若干其他嵌入式平台上运行。虽然 FreeBSD 项目尚未为这些平台提供官方镜像，但已有若干工具集可创建可写入 compact flash 卡的镜像。

这些工具之一是“Crochet”，可用于为树莓派、BeagleBone 和若干其他平台构建镜像。Crochet 可在 <https://github.com/kientzle/crochet-freebsd> 找到。

树莓派支持在 r239922 中引入。

## Clang 作为默认编译器

GCC 在大多数架构上不再是默认基本系统的一部分。FreeBSD 项目已从 GCC 切换到 Clang 作为默认编译器。这为 FreeBSD 提供了更现代、活跃开发的默认编译器。

虽然 GCC 默认不再构建，但在 FreeBSD 10 基本系统中仍可用。默认禁用 GCC 的变更在 r255348 中完成。

---

**Glen Barber** 作为业余爱好者，约在 2007 年开始深度参与 FreeBSD 项目。此后他参与了多种职能，最近的职位让他能够专注于系统管理和项目中的发布工程。Glen 居住在美国宾夕法尼亚州。

## BSDCan 2014

第 11 届年度 BSDCan！

技术 BSD 大会。高价值、低成本，人人都能找到适合自己的内容！BSDCan 是在加拿大渥太华举办的大会，已迅速确立自己作为面向 4.4BSD 操作系统及相关项目工作人员和用户的技术大会地位。组织者找到了一套吸引从初学者到高级开发者广泛人群的绝佳方案。

- **地点**：加拿大渥太华，渥太华大学
- **时间**：5 月 14–15 日（周三、周四，教程）；5 月 16–17 日（周五、周六，大会）

详情请访问：<https://www.bsdcan.org/>
