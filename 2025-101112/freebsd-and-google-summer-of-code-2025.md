# FreeBSD 与 2025 谷歌编程之夏

- 原文：[FreeBSD and Google Summer of Code 2025](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/freebsd-and-google-summer-of-code-2025/)
- 作者：Joe Mingrone


2025 谷歌编程之夏，Google Summer of Code (GSoC) 的圆满完成标志着 FreeBSD 第 21 年参与该项目。今年有三大亮点。首先，我们收到了 64 份申请，是去年的两倍多，约 2023 年的四倍。人工智能工具可能推动了申请数量的激增，虽然产生了一些低质量提交，但大部分申请整体质量仍然很高。其次，我们注意到来自南亚的兴趣显著增加。鉴于 FreeBSD 社区调查显示 85% 的受访者来自欧洲或北美，来自亚洲及其他地区的关注尤为可喜。第三，被接受项目的数量和质量令人鼓舞。在 185 个参与组织中，共有 1200 个编程之夏项目，而 FreeBSD 的 12 个被接受项目几乎是每个组织平均值的两倍，而且与去年不同，今年 12 个项目均顺利完成。

在讨论各个项目之前，先回顾一下我们参与谷歌编程之夏的原因。该项目需要大量投入，从组织申请流程、定义项目创意，到指导贡献者。投入这些时间和资源是否值得——这些本可直接用于开发——短期技术角度来看尚可争议；虽然部分项目带来了可提交的代码，但很多并未如此。然而，从长期来看，答案更明确。谷歌编程之夏在吸引和培养新贡献者方面发挥着重要作用。自 2022 年以来，已有五名 FreeBSD 新提交者通过谷歌编程之夏加入，而一名 2017 年参与者在后来成为了第 12 届核心团队成员。

## 2025 谷歌编程之夏项目

### 改进 Sockstat UI

对于那些不熟悉 sockstat(1) 的人来说，它是用于列出打开的互联网或 Unix 域套接字的命令。Damin Rido 的项目目标是通过支持动态列宽和集成 libxo 提供结构化输出，从而增强命令输出的灵活性。以下三条 Damin 的 pull request 均已合并到 src 中：

- 添加自动列宽功能并移除 `-w` 选项：[freebsd/freebsd-src#1720](https://github.com/freebsd/freebsd-src/pull/1720)
- 重新引入 `-w` 标志以自动调整列宽：[freebsd/freebsd-src#1746](https://github.com/freebsd/freebsd-src/pull/1746)
- 添加 libxo 支持：[freebsd/freebsd-src#1770](https://github.com/freebsd/freebsd-src/pull/1770)

### vmm(4) 对 QEMU 的加速支持

VMM（Virtual Machine Monitor）模块是 bhyve(4) 虚拟机监控器的内核组件，可通过 vmm(4) 访问。除了 bhyve，另一款在 FreeBSD 上被官方支持的广泛使用的机器模拟器和虚拟化器是 QEMU。然而，在本项目之前，FreeBSD 上的 QEMU 只能使用软件模拟（通过 Tiny Code Generator），因为缺乏硬件加速虚拟化支持。换句话说，QEMU 在 FreeBSD 上无法利用宿主 CPU 的虚拟化扩展直接运行虚拟机代码。这导致相比硬件辅助虚拟化，CPU 开销显著增加，来宾性能下降。

Abhinav Chavali 的 2025 谷歌编程之夏项目主要目标是将 VMM 加速支持集成到 FreeBSD 上的 QEMU。他通过修改 QEMU 的内存管理层，使其与 VMM 内核分配的来宾内存协作，实现了这一目标。同时，他对 VMM 进行了调整，使某些非关键设备（如 HPET 和 RTC）成为可选项，从而允许 QEMU 在用户空间模拟它们。这实现了一种混合中断模型（内核处理虚拟 LAPIC，用户空间模拟 IOAPIC），有潜力在 FreeBSD 下达到与 bhyve 相当的性能水平。

最新进展是，Abhinav 成功在 QEMU 上使用 VMM 加速启动了 FreeBSD 14。相关代码可查看 [这里](https://github.com/freebsd/freebsd-src/compare/main...dumrich:freebsd-src:vmm-qemu-mods-16)。

### Rust FreeBSD 设备驱动的测试与开发

近年来，在 FreeBSD 和 Linux 社区中引入 Rust 的讨论不断增加。相关讨论示例可参考 Linux 内核邮件列表上发布的 [Rust 支持 RFC](https://lkml.org/lkml/2021/4/14/1023) 以及 [FreeBSD hackers 邮件列表的讨论](https://lists.freebsd.org/archives/freebsd-hackers/2024-January/002823.html)。在两者社区中，讨论焦点各有不同：一些反对者担心会“使构建时间翻倍”，而支持者则认为 Rust 可以使某些工具的实现变得更容易甚至成为可能。除了讨论之外，已经取得了实际进展：Johannes Lundberg 在其 [硕士论文项目](https://github.com/johalun/rustkpi) 中创建了 Rust KPI 和网络驱动，而 David Young 则开发了一个简单的 FreeBSD 内核 Rust 模块，并在 [文章中总结了社区采用 Rust 的现状](https://www.nccgroup.com/research-blog/writing-freebsd-kernel-modules-in-rust/)。

本项目“Rust FreeBSD 设备驱动的测试与开发”基于此前在 FreeBSD 中引入 Rust 的努力。其主要目标之一是为 Rust 内核模块创建测试与持续集成框架。Aaron 在此 [视频](https://www.youtube.com/watch?v=y82-t1tDLWg) 中概述了他的 Rust echo 驱动，并在此 [总结文档](https://gist.github.com/Acesp25/8928e35e710fdce1896b5448fc6327df) 中说明了项目内容。相关代码库如下：

- [https://github.com/Acesp25/rustdrv](https://github.com/Acesp25/rustdrv)
- [https://github.com/Acesp25/freebsd-kernel-module-rust](https://github.com/Acesp25/freebsd-kernel-module-rust)
- [https://github.com/Acesp25/RustKLD](https://github.com/Acesp25/RustKLD)

### FreeBSD 磁盘管理工具

在 Braulio Rivas 的 2025 谷歌编程之夏项目之前，FreeBSD 缺少类似 Linux 上 GParted 的用户友好型磁盘管理工具，用于分区、调整大小、移动和管理文件系统。本项目旨在填补这一空白，开发了一个新的分区管理工具 [geomman](https://www.freshports.org/sysutils/geomman)。项目完成后，geomman 支持以下操作：

- 在同一块磁盘或不同磁盘之间复制和粘贴分区
- 扩展 UFS、NTFS、ext2、ext3 和 ext4 文件系统
- 缩小 NTFS、ext2、ext3 和 ext4 文件系统
- 可视化选择空闲空间以放置分区
- 创建 exFAT、NTFS、ext2、ext3 和 ext4 文件系统
- 检查文件系统：fsck_ufs (UFS)、fsck_msdos (FAT)、fsck.exfat (exFAT)、ntfsfix (NTFS) 以及 e2fsck (ext)
- 创建并标记分区
- 创建并加密分区

剩余工作：

- 添加 ZFS 支持
- 解决移动分区时的问题
- 编写测试用例

上游代码库：[https://gitlab.com/brauliorivas/geomman](https://gitlab.com/brauliorivas/geomman)

### 为 mkimg 添加 QCOW2 压缩镜像支持

QCOW2（QEMU Copy-On-Write version 2）是一种广泛使用的虚拟化磁盘镜像格式，支持精简配置（thin provisioning）和内建压缩等功能。FreeBSD 的 mkimg(1) 工具能够创建多种格式的磁盘镜像，包括 QCOW2。但此前，mkimg 对 QCOW2 的支持有限，无法直接创建压缩的 QCOW2 镜像。

在本次 GSoC 项目中，Christos Komis 对 mkimg 进行了改进，实现了以下目标：

- 支持 QCOW2 v2 压缩镜像
- 支持 QCOW2 v3 压缩与非压缩镜像
- 更新用户界面以提供新功能
- 扩展测试套件以验证功能正确性
- 更新 man 页以反映新增功能
- 进行代码重构以提升可读性与可维护性

实现已通过全面测试，并可直接提交至主代码库。用户现在可以直接使用 mkimg 生成压缩 QCOW2 镜像，简化虚拟机镜像生成流程，减少对外部转换工具的依赖。Christos 的代码可在此查看：[https://github.com/ckkomis/freebsd-src/commits/mkimg/qcow2-compression/](https://github.com/ckkomis/freebsd-src/commits/mkimg/qcow2-compression/)

### 在 Loader 中初始化 ACPI 并添加 Lua 绑定

Kayla Powell 的项目将 ACPICA 库的初始化扩展到 FreeBSD 的 loader 中，确保在内核加载前完整的 ACPI 命名空间可用。该工作替换了原本较为零散的 bootloader ACPI 处理方式，通过在 loader 内调用标准 ACPICA 函数（如 AcpiInitializeSubsystem、AcpiLoadTables、AcpiWalkNamespace、AcpiEvaluateObject）实现。这使得在启动早期就能一致地发现和查询 ACPI 对象。为了保持 loader 的轻量化，仅移植了初始化或脚本所必需的 ACPICA 组件，省略了许多非必要功能。

在此基础层之上，项目引入了 Lua 绑定，使脚本在 loader 中可以访问 ACPI 命名空间和对象评估功能。换言之，现在可以在内核加载前编写 Lua loader 脚本来遍历 ACPI 树、检查设备表条目、读取或附加 ACPI 节点数据。项目同时包含了单元测试和回归测试，例如对比 C 与 Lua 的命名空间输出，以及在不同架构下构建 loader。

Kayla 的项目总结可见她的博客：[https://kmpow.com/content/gsoc-writeup](https://kmpow.com/content/gsoc-writeup)
相关 pull requests：[1818](https://github.com/freebsd/freebsd-src/pull/1818)、[1819](https://github.com/freebsd/freebsd-src/pull/1819) 和 [1843](https://github.com/freebsd/freebsd-src/pull/1843)

### 改进 mac_do(4) 与 mdo(1)

Kushagra Srivastava 的项目旨在通过改进内核端 MAC 模块 mac_do(4) 及其配套的用户空间工具 mdo(1)，增强 FreeBSD 的凭证转换基础设施。该项目目标是避免依赖传统的 setuid 可执行文件（存在固有风险），在 FreeBSD 的 MAC 框架下实现可控的、精细化的凭证转换。

在内核方面，mac_do(4) 被扩展以支持针对每个 jail 的授权可执行文件配置，使管理员能够明确指定某个 jail 内哪些二进制文件允许请求凭证转换，而不局限于硬编码路径。此外，它现在会拦截标准的凭证更改系统调用（如 setuid(2)、setgid(2)、setgroups(2)），并将其视为完整的转换请求，受 mac_do(4) 策略模块管理。

在用户空间方面，mdo(1) 工具改进了凭证转换请求的精细控制。它现在支持通过 -g、-G、-s 等标志明确覆盖用户/组 ID 及补充组，并新增了 –print-rule 选项，用于显示匹配请求转换的 mac_do(4) 规则，从而帮助管理员创建规则并进行调试。

这些增强功能使凭证转换更加灵活、安全，并与 FreeBSD 的 jail 和 MAC 框架紧密集成，减少了对高风险 setuid 二进制文件的依赖，同时提升了审计能力和控制力。

Kushagra 的项目详细介绍可参见：[https://thesynthax.hashnode.dev/my-google-summer-of-code-journey-part-3](https://thesynthax.hashnode.dev/my-google-summer-of-code-journey-part-3)

### 缩短 FreeBSD 启动过程

Lahiru Gunathilake 的项目延续了以往提升 FreeBSD 启动速度的工作，通过分析启动序列、识别瓶颈并实施优化来缩短启动时间。利用内置的 TSLOG 跟踪框架，Lahiru 生成了启动路径的火焰图，以了解时间消耗集中在哪些环节，以及可以消除哪些不必要的延迟。

在分析发现的热点环节包括设备附加阶段、大型文件系统子系统（尤其是 ZFS）的初始化，以及 vfs_mountroot（根文件系统挂载）中的 sleep 延时后，优化工作进入实施阶段，包括：

- 将基准测试缓冲区从 16MB 减小到 256KB，使启动时间从 989 ms 缩短到 67 ms
- 缩短键盘和鼠标初始化中的长等待循环，并引入可调参数 `hw.atkbd.short_delay`
- 消除对 USB 设备的不必要等待

整体来看，Lahiru 报告称在内核初始化阶段减少了 8.2 秒，ZFS 和输入设备优化后减少了 3.5 秒，跳过 USB 启动等待后减少了 1.9 秒。

### WiFi 管理界面

Muhammad Saheed 承担了开发 FreeBSD 上 WiFi 网络管理工具的项目，包括 CLI 工具 **wutil** 和 TUI 工具 **wutui**。目标是覆盖“工作站模式操作，例如扫描、连接/断开无线网络”，并将这些功能封装到更清晰、一致的用户界面中。其他完成的工作包括：

- 更新相关 man 手册
- 创建 [wutil 的 Ports](https://www.freshports.org/net/wutil)
- [为 security/wpa_supplicant Port 添加 libwpa_client 构建选项](https://cgit.freebsd.org/ports/commit/?id=edaddcd1a5bb374e58de0d4f99a7cccf6aed09ec)
- 创建 [libifconfig 的 Port](https://www.freshports.org/net/libifconfig)
- 将所需的 ifconfig 辅助功能提取到 libifconfig 中 ([D52130](https://reviews.freebsd.org/D52130), [D52131](https://reviews.freebsd.org/D52131))

更多内容可参考 Muhammad 的 [博客](https://saheed.tech/writings)。

### FreeBSD ExtFS 日志功能

Pau Sum 的项目旨在为 FreeBSD 的 ext2fs 文件系统实现 Linux 兼容的日志功能。FreeBSD 原有的 ext2fs 驱动已能挂载和使用 ext2/3/4 文件系统，但缺乏日志功能，因此非正常关机后需要通过 fsck 进行较长时间的恢复。Pau 的工作引入了磁盘日志感知和事务记录，提高了崩溃恢复能力和文件系统完整性，使 FreeBSD 能够在 ext3/4 卷上挂载和重放日志，并与 Linux 系统更好地互操作。

设计上没有完全复制 Linux 的日志框架，而是实现了传统的元数据日志（metadata-only journal），使用相同的磁盘结构以保持兼容性。新代码定义了关键数据结构：

- **ext2fs_journal**：表示活动日志
- **ext2fs_journal_transaction**：分组原子元数据更新
- **ext2fs_journal_buf**：跟踪每个块的状态

核心文件系统操作（如 `ext2_link`、`ext2_mkdir`、`ext2_write`）扩展了日志钩子，用于开始事务、标记脏数据和结束事务。提交事务时会写入描述符块、元数据块、撤销块和提交块，并进行检查点操作以刷新磁盘更新。恢复分三步进行：验证事务范围、收集撤销块、重放未撤销的元数据。

项目结束时，12 个日志钩子中已有 11 个完成，正在进行截断和基于 extents 的操作开发。计划的扩展包括对 extents 和截断的日志支持、日志完整性校验、更全面的崩溃模拟和文档清理。已通过 [fsx](https://www.freshports.org/devel/fsx/) 和 [dirconc](https://www.netbsd.org/~riastradh/tmp/dirconc.c) 测试该实现。可在 [他 Fork 的 FreeBSD src 仓库](https://github.com/pxsum/freebsd-ext34/tree/extfs-journaling) 获取 Pau 的代码。

### 电源分析工具

Kasyap Jannabhatla 的项目旨在为 FreeBSD 提供细粒度的、基于进程的电源使用分析，以弥补 ACPI 全系统功耗统计的局限。受 [Performance Co-Pilot (PCP)](https://github.com/pxsum/freebsd-ext34/tree/extfs-journaling) 和 RAPL（Running Average Power Limit）启发，该项目实现了 FreeBSD 原生框架，而非移植 Linux 的 PowerTop。解决方案包括：

- **内核组件**：收集与功耗相关的指标
- **用户态守护进程**：提供命令行界面，跟踪每个进程的 CPU 使用和能耗

通过结合 RAPL 读数和由 `kvm_getprocs` 得到的每进程 CPU 利用率，该工具能够估算单个进程和线程的能耗，为精细化电源分析和 FreeBSD 电源管理的后续增强提供基础。

开发过程中，重点包括：

- 构建轻量级的守护进程架构
- 实现用于结构化访问 RAPL 数据的库 **librapl**
- 与进程计数整合，计算每线程的能耗

测试阶段使用了 OpenSSL Speed 等压力基准，并在多核运行环境下使用线程 ID 精确处理能耗计算。项目结束时，框架可可靠报告每进程随时间变化的能耗，所有交付物（守护进程、库和文档）均已完成。

实现代码可在 [GitHub 仓库](https://github.com/cheeseburger9309/freebsd-src/tree/gsoc2025-powerprofilingtool) 获取。

### 将 FreeBSD 移植到 QEMU microvm

[QEMU microvm](https://www.qemu.org/docs/master/system/i386/microvm.html) 是受 [Firecracker](https://firecracker-microvm.github.io/) 启发的极简虚拟机。虽然 FreeBSD 已被移植到 Firecracker，但该平台仅支持 Linux，限制了可移植性。本项目由 Wyatt Geckle 负责，目标是开发 FreeBSD 内核的 QEMU microvm 版本，借鉴 Firecracker 移植经验。为此，Wyatt 复现了之前的移植尝试，并分析了内核初始化问题，尤其是导致长启动时间的定时器配置问题。通过研究 NetBSD 的 microvm 实现和 Intel 文档，项目发现了 FreeBSD 的 timecounter 与 APIC 初始化存在的差异。

当前 FreeBSD microvm 内核可在 QEMU microvm 下启动，但未调用 `tc_init`，限制了可用定时器。Firecracker 移植仍存在 MPTables 问题，需要进一步研究。尽管存在这些限制，该项目拓展了 Wyatt 对 FreeBSD 与 NetBSD 内核内部机制、虚拟化及 microvm 平台的理解，并生成了详细文档，涵盖 FreeBSD 在 microvm 和 Firecracker 中的构建、运行与调试方法。该工作为未来对 FreeBSD、QEMU microvm 和 Firecracker 的支持，以及可复现的 microvm 调试流程奠定了基础。

更多信息可见 [Wyatt 的博客](https://wgeckle80.github.io/blog/categories/port-freebsd-to-qemu-microvm/)，代码在 [Wyatt 的 src 仓库 fork](https://github.com/wgeckle80/freebsd-src/tree/microvm-port)。

### 导师峰会

Robert Clausecker，FreeBSD 谷歌编程之夏联合管理员及导师，代表 FreeBSD 参加了 10 月 23 至 25 日在德国慕尼黑举办的导师峰会。会议讨论了 AI 生成及垃圾申请的问题，目前尚无确定方案。其中一个考虑方向是要求申请者在提交申请前与潜在导师会面，以保证导师、贡献者与项目的良好匹配，这也是 FreeBSD 过去几年已推广的做法。Robert 还与 Linux 基金会代表会面，探讨两家基金会在操作系统开发学生吸引方面的潜在合作。

峰会还涉及开源项目资金问题。Robert 与 RTEMS（一种用于多种设备的实时操作系统）的开发者交流，了解到其系统中集成了 FreeBSD 的网络栈。他还会见了谷歌编程之夏组织者 Stephanie Taylor，分享了谷歌编程之夏对 FreeBSD 的积极影响。当然，他也带回了一些周边，包括一件 T 恤和一对编程之夏纪念袜。

# 最终思考

回顾 2025 谷歌编程之夏项目的成功令人欣慰，但 2026 谷歌编程之夏很快就会到来。和往常一样，要重复今年的成功，最大的挑战在于设计合适的项目、寻找投入的导师，以及将申请者与导师匹配。幸运的是，谷歌编程之夏最近的变化将有助于缓解这些问题。

- **灵活的时间表与项目规模**：项目时间安排更加灵活。贡献者和导师可选择小型（90 小时）、中型（175 小时）或大型（350 小时）项目，总项目时间可延长，超出标准 8 周（小型）或 12 周（中型和大型）的限制。
- **扩展的贡献者池**：申请者群体扩大。参与者不必是大学生，任何开源新手都有资格参与。

但目前，让我们先享受这一集体成功，并肯定今年项目付出的巨大努力。

# 致谢

感谢所有向 [https://wiki.freebsd.org/SummerOfCodeIdeas](https://wiki.freebsd.org/SummerOfCodeIdeas) 提供项目创意的人，尤其是 2025 年的导师们：

- John Baldwin
- Olivier Certner
- Robert Clausecker
- Pedro Giffuni
- Tom Jones
- Warner Losh
- Ed Maste
- Getz Mikalsen
- Joe Mingrone
- Mehdi Mokhtari
- George Neville-Neil
- Colin Percival
- Alfonso Sabato Siciliano
- Alan Somers
- Toomas Soome
- Fedor Uporov
- Aymeric Wibo

---

Joe Mingrone 是 FreeBSD Ports 开发者，为 FreeBSD 基金会工作。他与妻子及两只猫共同居住在加拿大新斯科舍省的达特茅斯。
