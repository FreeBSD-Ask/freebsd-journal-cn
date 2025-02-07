# WPT/CFT：Lumina 桌面征集开发人员

- 原文链接：[WPT/CFT：Lumina Desktop Calls for Developers](https://freebsdfoundation.org/wp-content/uploads/2022/04/wipcft.pdf)
- 作者：TOM JONES & JT PENNINGTON

在自由桌面（Free Desktop）领域，桌面环境（Desktop Environment）选择丰富多样。从仅包含窗口管理器和一些装饰性工具的极简环境，到 Mate、KDE 和 Gnome 这类可以媲美 macOS 或 Windows 的完整桌面系统，应有尽有。

然而，在这个领域里，专门面向 BSD、且采用自由许可证的项目却相对稀缺。Lumina 桌面环境正是填补这一空白的项目。Lumina 最初由 Ken Moore 创建，旨在为 TrueOS 提供一个 BSD 许可证的桌面环境。它可以在任何操作系统上运行，但其核心目标是 FreeBSD 平台。

正因如此，Lumina 能够支持 FreeBSD 的特性，并与 FreeBSD 的桌面接口良好集成。

目前，Lumina 的开发由 JT Pennington 领导。你可能听说过他——他是知名播客 BSDNow 的幕后推手（作为该节目的主持人，我当然要夸赞一下我们自己）。JT 早年曾参与 PC-BSD 和 TrueOS 项目，并在此过程中结识了 Ken Moore。随着 Ken 的工作日益繁忙，他无法投入足够的时间来维护 Lumina，JT 便接手了该项目。

JT 近日公开表示 Lumina 需要更多开发者的帮助，以推动项目继续向前发展。他设想了一些重要的改进计划，但由于时间有限，难以独立完成。

## 开放项目请求

在 Lumina 官网的博客[https://lumina-desktop.org/post/2022-02-08/] 中，JT 发布了三个领域的帮助请求：

- 将构建系统从 Qmake 转移到 Cmake
- 重写 Lumina 文件管理器
- Lumina 2.0 窗口管理器

### 将构建系统从 Qmake 转移到 Cmake

Qmake 是 QT 的构建系统，但它开始成为移植到 Linux 发行版的障碍（在 FreeBSD 中可能也是一个麻烦）。在这方面的帮助将增加 Lumina 的用户和测试基础，更多的测试基础意味着更好的桌面环境。

### 重写 Lumina 文件管理器

Lumina 文件管理器的独特之处在于它能够快速与操作系统集成。它提供了特定于 ZFS 的功能，能够以简单方便的视图访问快照，这是其他文件管理器所不提供的功能。

Lumina-FM 准备重写，是时候从目前已有的优秀功能中学习，并增加更多定制化功能。JT 的博客文章详细列出了一些他对更好的文件管理器的想法，包括显示文件层级的磁盘使用情况细分，以及更好的缩略图缓存，能够理解网络驱动器。

### Lumina 2.0 窗口管理器

Lumina 窗口管理器的 2.0 版本不需要引入宏大的新功能，而是需要更新和增强，以便它能够替代目前的 Fluxbox 窗口管理器，而 Fluxbox 是 Lumina 的基础。

首先，Lumina 窗口管理器需要与 Fluxbox 达到功能对等，然后接下来的步骤是添加一些显然缺失的功能，比如从任何角落调整窗口大小的能力，以及现代功能，如屏幕角落对齐。

像 Wayland 兼容性这样的高级功能可以稍后再加，但首先 Lumina 项目需要一个可用的窗口管理器。

## 如何贡献

Lumina 使用 QT 作为其窗口工具包的基础。要能够帮助，你需要了解（或愿意学习）C++ 和 QT 接口。对于文件管理器和构建系统的改进，还需要了解或愿意学习有关 zfs 和 cmake 的许多复杂细节。

Lumina 及其网站都在 GitHub 上 [github.com/lumina-desktop/lumina]，该项目欢迎在 GitHub 上以拉取请求的形式贡献错误修复、新功能，或者通过报告问题和提出功能请求来进行贡献。

Lumina 是 BSD 生态系统中的一个重要部分，如果你有时间以任何形式进行贡献，Lumina 项目将很高兴收到你的消息。

---

**TOM JONES** 希望基于 FreeBSD 的项目能获得应有的关注。他住在苏格兰东北部，并提供 FreeBSD 咨询服务。
