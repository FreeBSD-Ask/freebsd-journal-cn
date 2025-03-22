# 通往 FreeBSD 桌面的直线路径

- 原文链接：[A Straight Path to the FreeBSD Desktop](https://freebsdfoundation.org/wp-content/uploads/2021/08/A-Straight-Path-to-the-FreeBSD-Desktop.pdf)
- 作者：**VERMADEN**

所以，你想试试 FreeBSD 桌面？  

嗯，也许你应该再考虑一下，因为即使是每天编写和修改 FreeBSD 的人——FreeBSD 开发者——多年来也更倾向于使用 Macbook，而不是尝试运行 FreeBSD 桌面。如果连他们都是这样，那你为什么还要尝试呢？  

在经历了从 4.4BSD 和 386BSD 过渡到 FreeBSD 的多年演变后，它依然保持着 UNIX 的本质，并且遵循 UNIX 的方式做事。简单的事情依然简单，而复杂的事情尽可能地保持简单。它仍然基于 X11，并使用传统的纯文本文件进行配置。有些人实际上更喜欢这种方式——尤其是在 Linux 世界引入 systemd(1) 之后。但 FreeBSD 更加简单和统一，并且拥有 Linux 世界中找不到的多个子系统或特性。例如，ZFS 启动环境 (Boot Environments) 或 GEOM 存储框架。此外，还有 Jail、基本系统 (Base System) 概念、直接集成在内核中的优秀音频子系统，以及更多特性。  

在本文中，我将尝试为你理清通往 FreeBSD 桌面的道路。  

## 硬件  

![](https://github.com/user-attachments/assets/bbe6d8e2-844f-4b1b-9215-726255727c9c)

硬件是最重要的部分。真的。我已经不止一次犯过这个错误。我买了 FreeBSD 不支持的硬件。我当时非常想要一款“最强”的 Intel X3000 显卡，它搭载在 Intel G965 主板上。我还想把它与当时非常强大的 Intel Core 2 Quad Q6600 处理器搭配使用。不幸的是，FreeBSD 并不支持那款显卡。它支持所有其他 Intel 显卡，如 GMA 950 和 GMA 3000，但唯独不支持 X3000。这个决定让我不得不进入 Ubuntu Linux 的世界大约一年，而你可能也能想象，那并不是一段愉快的时光。  

关于 FreeBSD 硬件的第一条经验法则是：**不要选择最新的硬件**。为了稳妥起见，尽量选择已经上市至少两年的硬件。这样，你就能最大程度地减少遇到不受支持设备的风险。此外，你还应该查阅 FreeBSD 最新版本的 **硬件支持说明 (Hardware Notes)**。在撰写本文时，最新版本包括 12.2-RELEASE 和 13.0-RELEASE。以下是相关链接：  

- [FreeBSD 13.0-RELEASE 硬件支持](https://www.freebsd.org/releases/13.0R/hardware/)  
- [FreeBSD 12.2-RELEASE 硬件支持](https://www.freebsd.org/releases/12.2R/hardware/)  

请记住，这些硬件支持列表也并不完美。例如，它们列出了许多受支持的 AMD 处理器，但却没有提到 AMD EPYC 服务器 CPU 和 AMD Ryzen 桌面/移动 CPU，而 FreeBSD 在这些平台上运行得非常流畅。

### 笔记本电脑  

许多人更喜欢“移动计算”而不是传统的 PC。总体而言，IBM 和联想 ThinkPad 笔记本电脑对 FreeBSD 的支持相当不错。我个人偏爱经典的 7 排 IBM 键盘，因此我使用的是 2011 年（是的，一台十年前的电脑）推出的 ThinkPad W520，在这台设备上，一切都运行得非常顺畅。其他配备该经典键盘的机型包括 X220、T420、T420s 和 T520。  

![](https://github.com/user-attachments/assets/ad1f2407-72f9-42b9-84d3-59bbf59cb08f)



不过，你不必回溯那么久的时间。例如，ThinkPad X1 Carbon 第 5 代和第 6 代运行 FreeBSD 非常稳定。ThinkPad X460、X470、T460 和 T470 也表现良好。实际上，有一个完整的笔记本测试列表，其中包含针对 FreeBSD 的有用说明，此外，还有一个 BSD 硬件数据库。在购买设备之前，我建议你查看这两个资源：  

- [FreeBSD 笔记本电脑支持列表](https://wiki.freebsd.org/Laptops)  
- [BSD 硬件数据库](http://bsd-hardware.info/)  

### WiFi  

这是 FreeBSD 领域最需要改进的部分之一，不过在许多方面，它的进展已经开始了。这个话题在最近的 FreeBSD 开发者峰会上被讨论过，FreeBSD 基金会也已经意识到了这个问题，并启动了一项计划来解决它。  

目前，最稳妥的选择是 802.11n 芯片——大多数都能正常工作，但有些只能在 802.11g 模式（速度较慢）下运行。例如，Realtek RTL8188CUS 就是一种“临时解决方案”，当你发现笔记本的内置 WiFi 芯片不受 FreeBSD 支持时，可以考虑使用它。这款 USB 802.11n WiFi 适配器体积非常小，从 USB-A 端口伸出仅几毫米。缺点是 FreeBSD 仅支持它的 802.11g 模式，但至少它能正常工作。  

### 显卡

![](https://github.com/user-attachments/assets/057aa3f5-7840-4d74-b38b-007393408aa8)


在 FreeBSD 下运行良好的显卡大致可以分为三类：  

1. **受开源驱动支持的 Intel 和 AMD 显卡**  
2. **由 Nvidia 官方闭源驱动良好支持的 Nvidia 显卡**  
3. **太旧和太新的显卡，可能根本不受支持**  

我个人使用低功耗的 Intel 集成显卡，它们的表现对我来说足够好。如果你需要更强的性能，那么可以考虑 AMD 和 Nvidia 的产品。我更倾向于 AMD，因为它们的驱动和硬件设计是开源的。不过，Nvidia 的二进制闭源驱动在很多情况下也能很好地运行。

### 蓝牙  

就个人而言，我认为蓝牙更像是手机的功能，而非笔记本电脑的功能，但蓝牙在所有笔记本电脑中都有存在，甚至许多 SBC（单板计算机）如树莓派也配备了蓝牙。FreeBSD 上的蓝牙应该可以正常使用鼠标和键盘，但在一些笔记本上，它也可能会“破坏”你的睡眠/恢复周期。  

### BIOS 设置  

有时你需要更改一些 BIOS 设置，才能让笔记本电脑正常睡眠/恢复。需要禁用的选项有时包括：  

- 蓝牙  
- 受信平台模块 (TPM)  

## X11  

![](https://github.com/user-attachments/assets/c4d4a13a-ef00-47ca-9612-7cd17627801e)


无论你选择 FreeBSD 12.2 还是 13.0——两者在桌面上的表现都不错。对于 Intel 和 AMD 显卡，你分别需要使用 `graphics/drm-fbsd12.0-kmod` 或 `graphics/drmfbsd13-kmod`。对于 Nvidia 显卡，则需要使用相应版本的 `x11/nvidia-driver`，因为有多个版本，所以请查看 Nvidia 发布说明，了解哪个版本最适合你的 GPU。至于 X11 本身，你可以使用 `x11/xorg` 或 `x11/xorg-minimal`。如果你希望安装最少的包，建议使用后者。不过，从某种意义上说，这其实没有意义，因为当你开始安装 GTK/QT 桌面软件时，你最终会安装大约 1,000 个包，且占用大约 10 GB 的空间。别忘了，FreeBSD 在 X11 方面有大量的文档——在手册和 FAQ 中：

- [FreeBSD X11 手册](https://freebsd.org/handbook/x11)  
- [FreeBSD X11 FAQ](https://freebsd.org/faq/#X11)  

## 图形环境  

![](https://github.com/user-attachments/assets/b39b34df-1ae7-474c-82d8-2b4b93ba5760)


图形环境是由你来决定的，有很多选择。有人偏好简约的 Openbox 和 FVWM 栈式窗口管理器。其他人喜欢 i3 或 Awesome 等平铺式窗口管理器。一些人选择轻量级环境，如 MATE 或 XFCE。还有一些人觉得完整的桌面环境，如 GNOME 或 KDE 最为舒适。如果你坚持使用更简约的窗口管理器，你将需要自己创建环境，包括状态栏和信息栏。决定是否使用通知守护进程，是否使用系统托盘和剪贴板管理器——而在桌面环境中，这些决策已经为你做出了。你无需重新造轮子 😀  

## 现成的解决方案  

你可以安装 FreeBSD 并按照自己的方式设置桌面，但如果你只想体验使用 FreeBSD 桌面的感觉，可以尝试一些现成的、专为桌面设计的 FreeBSD 解决方案。我将简要介绍其中的几种，更多信息可以参考 FreeBSD 基金会页面：[FreeBSD 桌面发行版指南](https://freebsdfoundation.org/freebsd-project/resources/guide-to-freebsd-desktop-distributions/)

### GhostBSD  

它可能是其中一个较老且更加成熟的解决方案。它使用 MATE 作为图形桌面，并且采用 OpenRC 初始化系统，而不是默认的 FreeBSD rc(8) 系统。这可能对一些已经熟悉 OpenRC 系统的 Linux 用户有吸引力。它们还有一个 XFCE 变体，如果你觉得它更合适，可以尝试。详情请访问 [https://www.ghostbsd.org/](https://www.ghostbsd.org/)。  

### NomadBSD  

这个系统专注于 Openbox，并进行了少量附加，适合非常轻量级的桌面环境。它还使用了 Tint2 和 Plank，看起来与 MacOS 布局相似。它提供了几个有趣的 DSB 工具，用于自动挂载或音量控制。你可以访问他们的页面 [https://nomadbsd.org/](https://nomadbsd.org/)。  

### helloSystem  

helloSystem 仍处于非常早期的开发阶段，但它具有一些 Linux 上甚至没有的独特功能，例如全局菜单搜索/过滤系统。它的图形部分主要是用 QT 编写的。详情请查看 [https://hellosystem.github.io/docs/](https://hellosystem.github.io/docs/)。  

## 结束语  

我选择了“创建自己的桌面”路径，并通过 Openbox 作为窗口管理器构建了我的图形环境——这在我的 FreeBSD 桌面系列中分为 26 部分进行描述——[https://vermaden.wordpress.com/freebsd-desktop/](https://vermaden.wordpress.com/freebsd-desktop/)。你还应该查看 FreeBSD 基金会对 X11 设置的总结——[https://freebsdfoundation.org/freebsd-project/resources/installing-a-desktop-environment-on-freebsd/](https://freebsdfoundation.org/freebsd-project/resources/installing-a-desktop-environment-on-freebsd/)。你也可以使用 desktop-installer 来为你完成很多工作——[https://www.grayhatfreelancing.com/freebsd-desktop-workstation-quickbuild/](https://www.grayhatfreelancing.com/freebsd-desktop-workstation-quickbuild/)。  

---

VERMADEN 是另一位 `${RANDOM}` 系统管理员，分享他在 IT 行业的工作经验。
