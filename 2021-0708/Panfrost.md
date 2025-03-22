# Panfrost 驱动程序

- 原文链接：[The Panfrost Driver](https://freebsdfoundation.org/wp-content/uploads/2021/08/The-Panfrost-Driver.pdf)
- 作者：**RUSLAN BUKIN**

成熟的操作系统和面向 arm64 平台的新图形驱动程序使我们能够在高端服务器/桌面系统、研究平台、嵌入式环境，甚至是当前的智能手机/个人设备上运行 FreeBSD。

每个技术巨头都在考虑将其硬件迁移——或者已经将其硬件迁移——到 arm64 架构。谷歌刚刚宣布了为其新 Pixel 6 设备设计的 Tensor 芯片；亚马逊构建了 AWS Graviton 处理器；Facebook 正在测试大规模的 arm64 服务器部署，并使用 arm64 SoC 为 Oculus AR/VR 眼镜提供动力。苹果的 M1 SoC 首次亮相于 MacBook 中。许多 Android 设备和谷歌 Chromebook 现在都基于 arm64。

同时，FreeBSD 的 arm64 平台已晋升为一级平台，并且目前已有一些 arm64 服务器解决方案可供不同厂商选择。FreeBSD 社区中的一些人已经在使用 arm64 机器作为 FreeBSD 桌面。

目前最大的 ARM 市场仍然是智能手机（ARM 的市场份额接近 100%）和个人设备。这些设备基于多种 ARM 架构的 SoC，并包括各种外设和网络连接选项（如 5G 等）。但是，SoC 所基于的移动 GPU 选择有限：Vivante、Broadcom VideoCore、Nvidia Tegra、Qualcomm Adreno、Imagination PowerVR、ARM Mali，以及苹果 M1 SoC 中包含的图形。

FreeBSD 不支持这些 GPU，但其中一个 GPU 的相关工作正在进行中，所以，让我们谈谈 ARM Mali GPU。

## Mali GPU

Mali GPU 最初由 Falanx Microsystems A/S（挪威公司）于 2005 年推出，该公司是挪威科技大学研究项目的衍生公司，随后于 2006 年被 ARM 收购。自 2007 年以来，ARM 已经开发了多个版本/微架构的 Mali：Utgard、Midgard、Bifrost 和非常新的 Valhall。Utgard GPU（Mali 4xx）是 armv7 时代的 GPU，但它仍然可以在更新的低端 arm64 系统中找到。ARM 声称它是全球出货量最大的移动 GPU。Midgard（Mali Txxx）和 Bifrost（Mali Gxx）是更新的 GPU，通常包含在基于 arm64 的智能手机或平板电脑中。

Mali GPU 有多个线程用于不同的操作。例如，Midgard GPU 使用复杂的 tiling 架构。它有三个线程负责顶点着色器；一个 tiling 负责将三角形排序到切片器中并传递给片段着色器；片段着色器则处理传入 tiling 上的最终光栅化（即写入帧缓冲）。

Mali GPU 支持 Vulcan 1.2、OpenGL ES 3.2、OpenCL 1.2（Midgard 架构）、OpenCL 2.0（Bifrost 架构）以及 OpenCL 2.1（Valhall 架构）。Mali GPU 还支持 SPIR-V。

## 软件

ARM 为这些 GPU 编写了自家的专有软件驱动程序。这些是针对 Linux 的二进制文件（binary blobs）。对于 FreeBSD，二进制文件不可用，且在 Linux 上使用它们将使你受限于特定版本的 Linux 发行版/内核和显示子系统。这限制了你根据自己的需求使用机器的能力，并破坏了自由开源软件的理念。它还无法调试代码，并带来了安全隐患。它不允许内核开发者在他们的桌面机器上使用最新的开发内核。但是，好消息是，目前已经有两个开源的逆向工程（RE）驱动程序可用于 Mali GPU：Lima 和 Panfrost。

## GPU 开源驱动

图形堆栈由多个组件组成，主要包括用户空间图形驱动程序和其内核部分，但也包括直接渲染模块（DRM）、内核模式设置（KMS）和 libdrm。

## 用户空间部分（Mesa 库）

图形驱动程序运行在用户空间，并进行图形 API（如 OpenGL、OpenCL 等）与每个 GPU 指令集架构（ISA）中的本地指令序列（操作码）之间的转换。

驱动程序组成了必须在 GPU 上执行的作业链。一个作业是一系列指令——一组抽象的内存缓冲区，包括临时缓冲区和主数据。一个作业还会有一组依赖项（称为“围栏”），这些依赖项必须在作业可以放到 GPU 线程上执行之前满足。例如，一个作业可能依赖于另一个作业的完成。在图形工作负载期间，每秒会执行几到几百个作业。作业由图形驱动程序组成，而图形驱动程序是 Mesa Gallium 库的一部分。

### Lima 驱动

![](https://github.com/user-attachments/assets/6d214664-717b-4e1f-9d4a-a5ea2c90651d)

你可能听说过 Lima——一款开源的、逆向工程的 Mali 驱动。它是为 Utgard 微架构设计的，支持 OpenGL ES 2.0（嵌入式）以及 OpenGL 2.1（桌面）。由于 Utgard 硬件的限制，它无法支持更新的 API，如 OpenGL ES 3.2 或 OpenCL/Vulkan。最初由 Luc Verhaegen 在 2012 年初开发，这项工作由 Codethink 赞助。该项目在 2013 年被放弃，并停滞了一段时间，直到 2017 年 6 月，由 AMD 的软件工程师 Qiang Yi 重新启动。

Qiang Yi 开始将驱动程序集成到 Mesa 库中，直到 2019 年 3 月，驱动程序才完全合并。Lima 驱动程序支持各种 armv7 板卡和 Chromebook。尽管 Lima 可能可以在 FreeBSD 上顺利构建，但我们尚未为其编写内核部分。

### Panfrost 驱动

一个新的项目支持 Midgard 和 Bifrost——这两种较新的 GPU 架构已被大量 arm64 系统采用。该项目通过追踪 ARM 的专有用户空间驱动程序和开源内核驱动程序而建立。

最初的努力始于 Lima 驱动，旨在支持 Midgard，项目名为 Tamil，但源代码从未发布。随后，由多伦多大学的数学学生 Alyssa Rosenzweig 创建了一个新的项目，代号为 Chai。Alyssa 的目标是为 Midgard GPU（也称为 Mali Txxx 系列）编写一个自由软件驱动。

Alyssa 完成了大部分 Midgard 的逆向工程和驱动开发。一个单独的项目，BiOpenly，开始了对 Bifrost 的逆向工程工作，之后，Chai 项目与 BiOpenly 合并，形成了 Panfrost 项目。随后，Alyssa 加入了 Collabora Ltd 继续在 Panfrost 上的工作。

与 Lima 不同，Panfrost 是与 ARM 合作完成的。最初（几乎完全）是由 Panfrost 社区和 Collabora Ltd 自筹资金完成的，但后来，ARM Ltd 加入了该项目，以优化解决方案。ARM 提供了文档，并帮助将 Mali 内核驱动程序推送到 Linux 上游。

### 在 FreeBSD 上的构建说明

以下是如何在 FreeBSD 上构建 Panfrost 用户空间驱动程序的示例：

```sh
git clone https://gitlab.freedesktop.org/mesa/mesa
mkdir build && cd build
meson .. . -Degl=enabled -Dgles1=enabled -Dgles2=enabled -Ddri-drivers= -Dvulkan-drivers=
-Dgallium-drivers=panfrost,kmsro
ninja
```

## Panfrost 内核驱动

我开始开发 Panfrost 内核驱动，目标是在 FreeBSD arm64 平台上为移动 GPU 提供首次支持。GPU 是一个内存映射设备，就像许多其他的 arm64 SoC 外设一样。图形堆栈的内核驱动程序提供了一个接口（“胶水”）来连接操作系统和实际的 GPU 硬件。它是图形堆栈中唯一直接访问 GPU 内存映射寄存器的部分。它负责 GPU 电源管理、硬件初始化、GEM 对象管理、GPU 线程管理、MMU 配置、GPU 中断和 MMU 页面错误处理。GPU 内核驱动程序接受来自用户空间的作业，分配缓冲区，设置 MMU 映射，将作业提交到 GPU 执行，并控制 GPU 的整体操作。

由于 Panfrost 驱动程序支持各种 GPU 型号和版本，它还负责为特定模型的硬件问题设置缺陷和解决方法。

除了与 DRM 接口并处理其请求外，GPU 内核驱动还为其用户空间部分提供 ioctl 接口。ioctl 层主要用于提交作业到 GPU 进行处理，管理缓冲区对象（BO），以及管理 MMU 映射。

### IOCTL 调用

内核驱动中实现了一组 ioctl 调用：

1. `panfrost_ioctl_submit` – 用于将作业提交给内核驱动。
2. `panfrost_ioctl_wait_bo` – 等待缓冲区对象（BO）上的最后一个 `panfrost_ioctl_submit` 完成。当多个进程渲染到一个 BO 并希望等待所有渲染完成时，这个调用是必须的。
3. `panfrost_ioctl_create_bo` – 创建 Panfrost BO。
4. `panfrost_ioctl_mmap_bo` – 用于将 Panfrost BO 映射到用户空间。这不会执行 mmap 操作，而是返回一个偏移量，在 DRM 设备节点上使用该偏移量执行 mmap。
5. `panfrost_ioctl_get_param` – 允许用户空间驱动读取硬件信息，如 GPU 型号、版本、支持的特性等。
6. `panfrost_ioctl_get_bo_offset` – 返回 GPU 地址空间中 BO 的偏移量。
7. `panfrost_ioctl_madvise` – 在缓冲区内容（后备页面）可以暂时释放回操作系统以节省内存时，向内核提供建议。

### MMU

Mali GPU 具有一个内部 MMU，因此 GPU 操作的所有缓冲区都是虚拟寻址的。MMU 页面表的格式大多数与 arm64 CPU MMU 兼容。然而，在这项工作中，发现了一些差异：

- PTE 条目的页面类型位与 CPU MMU 不同。
- 访问标志位是反向的。
- 一些权限位不一致。
- 控制寄存器和 TLB 维护模式有很大不同。

在将作业提交给 GPU 之前，我们必须设置 MMU 映射，工作完成后，我们必须拆除它们。为 FreeBSD 开发的 Mali GPU MMU 支持已经被开发、审查，并已提交到主干树中。它是 arm64 IOMMU 框架的一部分（见 sys/arm64/iommu 目录）。最初，与 ARM 系统 MMU (SMMU) 类似，GPU MMU 管理例程被引入到 arm64 的 CPU 页面管理（pmap）代码中。后来，人们开始抱怨将 SMMU、GPU 和 CPU MMU 保持在同一 pmap 中会显著减慢开发进程，因此我们决定将 SMMU 和 GPU 例程拆分到一个专门用于 IOMMU 的新 pmap（iommu_pmap.c）中。

## KMS

Mali GPU 仅处理渲染。它处理内存中的数据，并将其写回内存。它不包含显示控制器，无法将任何数据输出到显示器。通常，arm64 平台的图形支持由多个硬件组件组成：

- 处理硬件窗口（平面）的显示处理器。至少应注册两个平面——主窗口平面和光标平面。当我们在屏幕上移动光标时，设备会将它们混合（在硬件中将一个图层放置在另一个图层之上），并将最终数据提供给显示接口控制器。
- 显示接口控制器，在物理层上将数据输出到显示器。通常，HDMI、DisplayPort、LVDS、MIPI DSI 等接口在此设备中实现。
- 外设设备，如 I2C/SPI。通常需要一个 i2c 控制器来读取有关连接的 HDMI 显示器的信息（其分辨率、功能等）或配置 MIPI 显示器。
- GPU 本身。

FreeBSD 应支持这些设备的驱动程序，以及给定平台的时钟框架，因为必须配置一组时钟域以将数据传输到显示器。

KMS（内核模式设置）是 DRM 的一部分。它允许用户在注册多个硬件平面以及提供多个分辨率/连接选项时，配置输出。

## DRM

直接渲染管理器（Direct Rendering Manager，DRM）是 Linux 和 FreeBSD 中用于管理图形处理单元（GPU）的框架。它旨在支持复杂的 GPU，提供对硬件的统一访问。它还负责内存管理、DMA 和中断处理。

DRM 暴露了一个 API（一组 ioctls），允许用户空间程序向 GPU 发送命令/数据并配置内核模式设置（KMS）。libdrm 提供了一组包装器，简化了用户应用程序的 API 调用。

我们目前支持两个 DRM/KMS 项目

### 1. drm-kmod 和 LinuxKPI

DRM 于 2001 年以“drm-kmod”端口的形式出现在 FreeBSD 中，以支持当时的 3D 显卡（也称为 drm1）。从 2010 年到 2012 年，Konstantin Belousov 开发了对新 Intel HD Graphics GPU 的支持，引入了 FreeBSD 基础中的新 DRM 支持（drm2）。

一个新项目 drm-next 于 2017 年左右启动，旨在支持未修改的 DRM。随着上游频繁更新，支持 drm2（一个移植版本的图形驱动程序）变得越来越困难，因为每次驱动程序更新时都需要通过驱动程序并将 Linux-KPI 转换为 FreeBSD-KPI。

DRM 是在 Linux 内核环境中开发的，因此它使用 Linux 内核 API。为了在 FreeBSD 中“按原样”使用 DRM 代码，我们必须将其调用转换为本地 FreeBSD 原语/API。在这个新的“drm-next”项目中，通过使用 LinuxKPI 框架来完成。LinuxKPI 最初是由 Mellanox Inc. 和其他公司为存储设备和网络适配器驱动程序开发的，但现在也需要用于从 Linux 导入图形驱动程序。由于许可问题，它们不能成为 FreeBSD 内核的一部分（只能作为模块）。

LKPI 使得将 Linux DRM 驱动程序带到 FreeBSD 变得更容易，修改最小，并能够保持与 Linux 上游的同步。2018 年晚些时候，drm1 和 drm2 被从内核移到端口树，作为 drm-legacy-kmod 端口；而 drm-next 端口则采用了原始名称“drm-kmod”。

该 DRM/KMS 驱动程序不仅在 x86 平台上支持大多数现代 Intel HD GPU，还在 powerpc 和 arm64 平台上提供支持，目标是基于 PCI-e 的 AMD/Radeon 显卡。

我已经在 ARM Neoverse N1（arm64）服务器上使用了几个月，配备了一块 Radeon PCIe 显卡，它运行得非常顺利。

安装驱动程序非常简单：

```sh
cd /usr/ports/graphics/drm-kmod
make install
```

### 2. drm-subtree & DRMKPI

一个新的项目“drm-subtree”由 Emmanuel Vadot 和 Michal Meloun 启动，旨在支持基于 FDT 的 arm 和 arm64 系统上的 DRM，目标是将 DRM 恢复到内核中。

在基于 FDT 的平台（如 arm 和 arm64）上，DRM 的情况稍有不同：我们希望将 DRM 保持在内核中，因为它更适合嵌入式环境，并且我们不想从 Linux 导入 arm 图形驱动程序。一个原因是，FDT 基于资源（gpio/pinmux/clock/regulator）的框架在 Linux 和 FreeBSD 之间不同，这可能使得 LinuxKPI 抽象变得非常复杂。此外，驱动程序需要与我们定期从 Linux 导入的 DTS 文件同步。

该项目包括一个新的 DRMKPI 框架，它是 LinuxKPI 的简化版——但仅用于 DRM。在 DRMKPI 中，大多数不需要的存储/网络代码（如 netdevice/sysfs 代码）已被移除，消除了 Linux 中 task_struct 的大部分字段，目标是完全消除 task_struct 本身。

在 https://github.com/evadot/drm-subtree 上，您可以找到一组可工作的 DRM/KMS 驱动程序和适用于 Allwinner、Nvidia Tegra 和 Rockchip SoC 的说明，这些驱动程序允许在 FreeBSD 下启用显示功能。

## Panfrost 的目标平台

FreeBSD Panfrost 内核驱动程序是为 Morello 开发的——这是 ARM 与剑桥计算机实验室的合作项目，旨在构建第一个启用 CHERI 的 ASIC。该项目由 UKRI 和 ARM Ltd 支持。

与此板一起提供的 CHERI CPU 基于 ARM Neoverse N1 SoC——ARM 的首个服务器级架构，Cortex 系列的继任者。尽管 Morello 开发板的 SoC 将包括 Bifrost GPU，但 N1 开发平台不包括 GPU，这意味着我们不能用它进行 GPU 驱动程序开发。

Morello 开发板目前正在进入制造的最后阶段，预计今年晚些时候会发布。在此板尚未发布时，我们选择了 Rockchip RK3399 作为 Panfrost 内核驱动程序开发的目标平台。RK3399 是一个成熟且得到良好支持的 FreeBSD 平台，市场上有许多低成本的单板计算机。它包含一个 Mali Midgard GPU、支持全高清和 4K 的 HDMI 控制器，甚至还有一个 PCI-e 控制器。它是一个有趣的平台，因为它可以完全使用自由软件运行，包括 GPU 驱动程序。

![用于 Panfrost 内核驱动开发的 rk3399 开发板](https://github.com/user-attachments/assets/2e841d85-7466-48ad-b056-d89a4dc22201)

## 结果

该项目于 2020 年 11 月开始，大约花了 6 个月的时间才使 Mali GPU 正常工作。大部分时间用于编写 Rockchip DRM 驱动程序（视频引擎、显示控制器）并修复外围设备驱动程序中的错误。Panfrost 驱动程序最难的部分是弄清楚 MMU 单元的行为、其页面表格式以及缓存管理操作。我还需要为 drm-subtree 添加缺失的功能（以导入 DRM 调度器及其依赖项）。感谢 bsdmips 和 panfrost IRC 频道上的人们的帮助和精神支持！

快速测试表明，桌面图形环境的性能良好，响应快速。RK3399 有两个视频显示处理器：Big 和 Little。Big 支持 4K 分辨率，Little 则支持 Full-HD。对 Little 的支持已经添加。Wayland 和 Xorg 都能正常运行，只有一些小问题，比如内存泄漏（目前正在进行修复）。

我能够运行 2D 加速桌面、浏览器以及像 glxgears(1) 和 geartrain(1) 这样的 3D Mesa 演示：轻松达到 HDMI 显示器的最大刷新率（60 FPS）。

我期待将我的主桌面切换到 Morello Board，并每天使用我的 Panfrost 驱动程序。FreeBSD 上的 Panfrost 驱动程序已在 GitHub 上发布以供审查：<https://github.com/evadot/drm-subtree>。

---

**RUSLAN BUKIN** 是剑桥大学计算机实验室的高级研究软件工程师。他的主要研究领域是机器相关软件。他领导了多个项目，包括将 FreeBSD 移植到 RISC-V 架构、为 arm64 添加 IOMMU 支持以及为 amd64 提供 Intel SGX 支持。除了 FreeBSD，他还开发了用于研究项目的 MDEPX 嵌入式内核。在空闲时间，他喜欢在英国骑自行车。
