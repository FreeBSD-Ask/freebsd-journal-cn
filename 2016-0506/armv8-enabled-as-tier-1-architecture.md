# ARMv8 成为一级架构：协作开发项目

- 原文：[ARMv8 Enabled as Tier 1 Architecture: A Collaborative Development Project](https://freebsdfoundation.org/wp-content/uploads/2016/06/ARMv8-Enabled-as-Tier-1-Architecture.pdf)
- 作者：Andrew Wafaa

FreeBSD 对 ARM 架构的支持历来由社区自发推动，对大量嵌入式平台而言这种模式运作良好。随着 ARMv8 发布，基于 ARM 的系统开发扩展到企业领域，涵盖服务器基础设施和网络基础设施。为支撑企业市场，我们显然需要更激进的开发模式，以交付最完整、最具竞争力的 FreeBSD。

ARM 内部缺乏 FreeBSD 专长。虽然许多人对 FreeBSD 有兴趣，但没有真正的开发能力。随着 ARM 架构第 8 版（支持 64 位）的问世，我们开始探索在新架构上启用 FreeBSD 的方式。由于不愿独自推进，我们的选择受到限制。

2013 年马耳他 EuroBSDCon 大会上，ARM 做了一场介绍 ARMv8 的演讲。除了评估社区的兴趣和需求，ARM 还在寻找与社区合作的方式，特别是与 FreeBSD 基金会的合作。这次活动建立的联系对 ARM 内部及与合作伙伴的讨论产生了重要影响。

时间推进到 2014 年，ARM 信心充足，决定真正与社区合作，将 ARMv8 作为 Tier 1 架构在 FreeBSD 上提供支持。ARM 认为最佳路径是资助 FreeBSD 基金会，与之合作找到合适的人选，包括 Cavium 公司，共同推进这项工作。这些讨论进行的同时，资深 FreeBSD 开发者和提交者 Andrew Turner 已开始将 64 位 ARM（即 AArch64、ARMv8 或 ARM64）支持引入 FreeBSD。ARM 自 2012 年起就 64 位 ARM 话题与 Semihalf 接触，Cavium 则在 2013 年开始与 Semihalf 洽谈，ARM 和 Cavium 都清楚 Semihalf 是必须参与的合作伙伴。

## 真正的协作

2014 年 10 月 1 日，Cavium 发布公告，描述与 ARM 和 FreeBSD 基金会的合作伙伴关系。ARM 和 Cavium 都向基金会提供资金，基金会再聘请 Andrew Turner 和 Semihalf。ARM 和 Cavium 通过基金会开展工作，希望确保这项工作是开放、协作的。

有些人可能不了解所有参与方，下面简要介绍关键实体：

**ARM Ltd.** 是一家 IP 设计公司，产品线完整，包括 32 位和 64 位 RISC 微处理器、图形处理器、配套软件、单元库、嵌入式存储器、高速连接产品、外设和开发工具。

**Cavium**（纳斯达克：CAVM）是高集成度半导体产品的领先供应商，产品应用于企业、数据中心、云端以及有线和无线服务提供商场景，实现智能处理。Cavium 提供一系列集成且软件兼容的处理器，性能范围从 100 Mbps 到 100 Gbps，为企业、数据中心、宽带/消费者、接入和服务商设备提供安全、智能的功能。2014 年，Cavium 推出面向服务器工作负载和数据中心横向扩展部署的 ThunderX® 工作负载优化 SOC。

**Semihalf** 为操作系统、虚拟化、网络和存储领域的高级解决方案开发软件。他们使软件与底层硬件紧密结合，以实现最高的系统容量。其合作伙伴和客户均为半导体行业的领军企业。Semihalf 团队中负责 ThunderX/ARM64 开发工作的成员包括：Wojciech Macek（项目负责人，FreeBSD 提交者；负责 ARMv8 基础支持、PCI、SMP 和存储领域）、Zbigniew Bodek（FreeBSD 提交者；ARMv8 基础支持、GICv3/ITS、VNIC）、Dominik Ermel（AHCI、测试）、Michał Stanek（PCI、SMP、调试）和 Tomasz Nowicki（KDB、观察点），另外 Michał Mazur 提供了 UEFI 启动和调试方面的协助。

**Andrew Turner** 是自由软件工程师和 FreeBSD 提交者，专注于在基于 ARM 的芯片上运行 FreeBSD。他启动了 ARM64 FreeBSD 移植工作。Andrew 负责整体系统架构以及系统中可供多种 SoC 共用的部分。此前，Andrew 曾在咨询公司和芯片厂商担任嵌入式软件工程师。

## 分而治之

ARMv8 是全新架构，并非旧架构的扩展，因此各方同意将所需工作拆分为三块具体内容：内核、用户空间和平台。ARM 为 Andrew 和 Semihalf 提供支持，自然也为双方配备了所需工具——开发平台、调试工具和文档。作为芯片厂商，Cavium 能够为 Semihalf 提供早期开发板，并将首批开发服务器部署到 Sentex 托管设施，让更多开发者能够访问服务器硬件。

## 消除无序分化

这项工作被视为基础性工程，是 ARM 和 FreeBSD 生态系统构建的基石。因此，ARM 希望尽可能符合标准，避免碎片化（ARM 过去曾被指责存在碎片化问题）。这意味着要确保支持并符合 ARM 服务器基本系统架构（SBSA）、ARM 服务器基础启动要求（SBBR）、电源状态控制接口（PSCI）和虚拟机规范。

SBSA 是面向固件和操作系统开发者的硬件规范。它使应用开发者能够为所有 ARMv8-A 系统编写单一操作系统镜像。操作系统和固件厂商受益于拥有明确定义的目标平台，单一操作系统镜像可在所有符合规范的系统上运行，与厂商或微架构无关。

SBBR 是 SBSA 的配套规范。它定义了符合 SBSA 的系统在操作系统部署、启动、硬件配置和管理以及电源控制方面所需的平台固件抽象。涵盖 UEFI、ACPI 和可信固件等内容。

PSCI 定义了标准的电源管理接口，可供操作系统厂商用于在不同特权级别运行的监管软件。电源管理时，操作系统、hypervisor 和安全固件必须协同工作；这一标准旨在简化不同厂商、不同特权级别的监管软件之间的集成。

虚拟机规范针对虚拟化客户机镜像，确保镜像在不同 hypervisor 之间可移植。

除了符合上述规范，还包括以下内容：

- 基本 CPU 支持
- 初始目标为 ARMv8，包括 Cortex®-A57 和 ThunderX 微架构
- 工具链
- 加载器
- 外设
- 定时器、GIC（全局中断控制器）
- 块存储
- ARM locore
- 汇编宏和定义
- 引导
- 异常处理
- 缓存维护和上下文切换
- 虚拟内存 / pmap
- TLB 维护和页错误处理例程
- 将 pmap 转换为 64 位
- 64 位用户空间胶水代码
- 系统调用、信号处理、库入口点
- SMP
- 带缓存一致性的双插槽

## 启动 FreeBSD ARMv8 移植

工作于 2014 年夏末正式启动，到 2015 年春已能在 AArch64 系统上登录 FreeBSD。支持仍然非常基础，部分组件缺失。关键缺失组件是 DTrace。在基金会建议下，ARM 与剑桥大学合作，实现 DTrace 和硬件性能计数器支持。这项工作极大拓展了调试手段，有助于构建更完整的 FreeBSD 体验。在 BSDCan 2015 上，Semihalf 通过 FreeBSD 在 48 核 Cavium ThunderX 平台上运行的演示，展示了所取得的进展。在 ARM64 移植出现之前，FreeBSD 在 ARM 上依赖 u-boot 作为启动流程的关键环节。虽然 u-boot 在嵌入式领域占据主导地位，但兼容 UEFI 的启动方案在企业/网络领域更为重要。因此，选择 UEFI 兼容作为默认加载器要求。FreeBSD 加载器被移植为可由兼容 UEFI 的启动加载器加载和执行的 EFI 应用。

较具挑战性的领域之一是 CPU 可扩展性。Semihalf 和 Andrew Turner 在支持 Cavium 最低核心数（48 核）的过程中付出了大量心血，尤其是处理内核中的静态假设问题，这些问题逐个得到解决。2015 年 6 月，在加拿大渥太华 BSDCan 大会期间举办的 FreeBSD 开发者峰会上，Semihalf 的 Zbigniew Bodek 在 Cavium 48 核 ThunderX 系统上做了现场演示。

各方共同取得了巨大进展。为持续改进，各方仍在协作巩固和完善早期工作。ARM 和 Cavium 继续与合作伙伴及 FreeBSD 社区合作，并资助 FreeBSD 基金会。

FreeBSD 基金会董事会成员 George Neville-Neil 表示：“将 ARMv8 引入 FreeBSD 是一项重大工程，若没有 ARM 和 Cavium 的支持将无法完成。他们与 FreeBSD 基金会合作，以创纪录的速度让一个全新架构运转起来。ARMv8 现已成为 FreeBSD 生态系统中的重要架构，并与操作系统的其余部分及其工具一同持续发展。”

## 通往 FreeBSD 11 之路

我们目前正准备将 FreeBSD/arm64 移植纳入即将发布的 FreeBSD 11.0，处理 Tier 1 架构所需的剩余事项。包括更新文档、运行 FreeBSD 测试套件并修复失败项、为 FreeBSD 发布工程团队和安全官采购并安装硬件，以及验证安装流程。

FreeBSD 发布工程团队制作 AArch64 快照安装介质和虚拟机镜像。快照在 freebsd-snapshots 邮件列表上公布，可从 **ftp://ftp.freebsd.org/pub/FreeBSD/snapshots/VM-IMAGES/11.0-CURRENT/aarch64/Latest/** 获取。

如果想测试 FreeBSD 在 AArch64 上的运行，有几种方式。可以使用 QEMU 模拟。以下步骤说明具体做法：

```sh
# 安装 QEMU 模拟器 pkg，或从 ports 构建
% pkg install qemu-devel

# 获取快照虚拟机镜像
% fetch ftp://ftp.freebsd.org/pub/FreeBSD/snapshots/VM-IMAGES/11.0-CURRENT/aarch64/Latest/FreeBSD-11.0-CURRENT-arm64-aarch64.qcow2.xz

# 获取特殊版本的 UEFI 启动固件
% fetch http://people.FreeBSD.org/~gjb/QEMU_EFI.fd

# 解压虚拟机镜像
% xz -d FreeBSD-11.0-CURRENT-arm64-aarch64.qcow2.xz

# 启动虚拟机
% qemu-system-aarch64 -m 4096M -cpu cortex-a57 -M virt -bios QEMU_EFI.fd -serial telnet::4444,server -nographic -drive if=none,file=FreeBSD-11.0-CURRENT-arm64-aarch64.qcow2,id=hd0 -device virtio-blk-device,drive=hd0 -device virtio-net-device,netdev=net0 -netdev user,id=net0

# 用 telnet 连接到虚拟机控制台
% telnet localhost 4444
```

模拟对某些用例够用，但没有什么比真实硬件更好。各种硬件平台以不同的价格和性能陆续上市。96Boards 项目提供价格低廉（250 美元以下）的单板系统，包括基于海思的 HiKey 开发板和高通 Dragonboard。Cavium、AMD 和 Applied Micro 则提供更高规格的处理器和系统。

## Cavium ThunderX，主要参考平台

凭借出色的功能和可用性，Cavium 的 ThunderX 平台是 FreeBSD ARMv8 移植的主要硬件参考平台。在 ThunderX 上启动 FreeBSD/arm64 的说明可在 **<https://wiki.freebsd.org/arm64/ThunderX>** 找到。

Cavium 的 ThunderX 平台有两种配置——单插槽 48 核平台和双插槽 96 核变体。

ThunderX 支持丰富的功能和外设，并提供强大的以太网能力，包括 40 Gbps、20/10 Gbps 和 1 Gbps 接口。网络子系统划分为几个核心组件：

- IBGX - 通用以太网接口
- INIC - 网络接口控制器
- ITNS - 流量网络交换

按上述顺序，它们分别实现：MAC 层、网络接口层，以及上述实体与其他片上设备之间的硬件交换。此外，NIC 是支持 SR-IOV（单根 I/O 虚拟化）的设备，可容纳多达 128 个虚拟功能，每个虚拟功能提供完整的网络接口特性。

为支持 ThunderX 网络，FreeBSD 引入了 NIC、VNIC、BGX 等驱动……

> 注：原文 PDF 在此处之后仍有内容。PDF 文本提取在此截断。完整文章请参阅原版 PDF：<https://freebsdfoundation.org/wp-content/uploads/2016/06/ARMv8-Enabled-as-Tier-1-Architecture.pdf>
