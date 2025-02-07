# Pinebook Pro 上的 FreeBSD

- 原文链接：[FreeBSD on the Pinebook Pro](https://freebsdfoundation.org/wp-content/uploads/2022/04/FreeBSD-on-the-Pinebook-Pro.pdf)
- 作者：**JESPER SCHMITZ MOURIDSEN**

Pinebook Pro 是一款基于 Rockchip rk3399 的 ARM64 笔记本电脑。由于全球元件短缺（据 Pine64 称），目前它尚未开售。不过，你可能已经有了一台，却错过了在其上运行 FreeBSD 的机会。本文将介绍如何让 FreeBSD 在 Pinebook Pro 上作为实用的桌面系统运行。如果你没有 Pinebook Pro，我提供的测试镜像和构建步骤同样适用于 RockPRO64 开发板。（唯一例外的是 U-Boot，对于 RockPRO64 请使用 FreeBSD 自带的版本。）

正如我所说，它基于 rk3399，但它并不是一个装在 RockPRO64 外壳内的设备——它有自己专用的主板。因此，官方的 RockPRO64 构建方案并不适用于此。现在有一些正在审核的补丁能使 Pinebook Pro 成为一个实用的桌面系统，其中最引人注目的是 Emmanuel Vadot 的 drm 子树以及 Ruslan Bukin 早前关于 panfrost 的相关工作。此外，Alexander Tymoshenko 编写了实现声音功能的补丁。由 SleepWalker 在 forums.freebsd.org 上发起的 Pinebook Pro 讨论帖中，许多网友都为此贡献了大量工作。如果你想深入了解 FreeBSD 在 Pinebook Pro 上运行的开发历程，值得一读。

## 支持的硬件

- **图形堆栈**：通过 Ruslan Bukin 的 panfrost 驱动工作以及 Emmanuel Vadot 在 [drm 子树](https://github.com/evadot/drm-subtree/drm-subtree)中的相关工作，实现了硬件加速图形支持。  
- **声音**：由 Alexander Tymoshenko 实现了录音和播放功能。  
- **eMMC 和 SD 卡**：均得到全面支持。  
- **SPI 闪存**：已能被检测到，但可能仍受到 bug [244146](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=244146) 的影响。  
- **PCI 桥**：支持搭配 SSD 使用，尽管在我的测试中，ufs 显示出一些异常表现。  
- **CPU**：所有 CPU 均受支持，但 FreeBSD 无法充分利用 big.LITTLE 架构，即较快的核心必须跟随较慢核心的频率。  
- **USB 2.0 端口**。  
- **触摸板和键盘**：触摸板目前仅能作为简单鼠标使用。  
- **网络摄像头**：通过 webcamd 驱动可正常工作。

## 不支持的硬件

- **WiFi 和蓝牙**，以及通过 USB-C 的 DP（DisplayPort）功能。  
- **Type-C 接口**：如果通过 gpioctl 启用，Type-C 才能以 Type-C  模式工作。但如果你不完全确定设备树中正确的引脚设置及其配置方法，请不要尝试此操作。

## 可以运行哪些软件？

我已经测试了 sway 和 hikari / wayland 以及 X11 与几个桌面环境。在此过程中，我发现 openbox 与图形堆栈存在问题：它在窗口边框内并不能正确地渲染任何内容。幸运的是，xfwm4 没有这个问题。因此，LXQt 必须将默认的 openbox 替换为 xfwm4。LibreOffice 运行良好。Electron 相关的软件在 FreeBSD 上仅支持 amd64，所以你可能会错过一些基于 Electron 的应用，例如 vscode-oss。Sway 必须回退到 14.1 版本，否则会出现 “Cannot use DRM dumb buffers with non-primary DRM FD.” 错误。我怀疑这是因为系统中存在两个显卡条目，即 /dev/dri/card0 和 /dev/dri/card1，而只有 card1 拥有渲染设备，不过我并没有进一步深入研究。为此，我选择将 sway 和 wlroots 回退到在 Pinebook Pro 上最后一个已知可用的版本。

在 arm64 平台上，Firefox 以及其他一些应用程序经常崩溃，除非通过 `proccontrol -m aslr -s disable firefox` 禁用 ASLR（地址空间布局随机化）后再启动。另外，为了在 Firefox 中测试 webgl，你可能需要在 `about:config` 中将 `webgl.force-enabled` 设置为 true。由于 webcamd 运行良好并支持内置摄像头，我在 Nextcloud Talk 中使用 Firefox 测试了一个 WebRTC 通话，效果也很不错。通话中的屏幕共享也能正常工作，不过一次只能共享一个窗口。Vlc 对 `screen:///` 捕获的支持也不理想，所以全屏捕获目前存在一些问题。我还在 YouTube 上观看了全屏视频，未发现任何卡顿。需要注意的是，由于这项工作基于 14-CURRENT，默认的软件包仓库并非每季度构建一次——因此部分软件包可能偶尔构建失败，从而缺失。



## 引导过程

没有支持视频的 U-Boot 版本（即启动时屏幕上什么也不显示的版本）太旧，无法满足需求。此外，我在测试镜像中跳过了面板驱动，所以面板必须由 U-Boot 启动。背光功能显然也依赖于较新的 U-Boot。我使用过最成功的版本是 2021.7，该版本在本文撰写时正好存在于 FreeBSD ports 中。

NetBSD 注意到面板在软重启时会出现故障。他们提供了一个补丁，我强烈推荐使用，因为面板在温重启时似乎会以一种极其不正确——而且可能对面板非常不利——的状态启动。这个 NetBSD 补丁已被适配到 FreeBSD ports 树中，并包含在测试镜像里。

如果你使用 FreeBSD U-Boot 从 eMMC 启动，需要知道它默认不会从 SD 卡启动。在这种情况下，要从 SD 卡启动，你应当按任意键停止自动启动，然后输入 `run bootcmd_mmc1`。请注意，U-Boot 下的键盘支持不太完善，它可以输入，但最好慢慢敲击。你也可以完全禁用 eMMC，只从 SD 卡直接测试测试镜像。这样可以避免 U-Boot 版本问题，但使用笔记本内部 eMMC（较为脆弱）的断电开关会稍显不便。打开笔记本很容易——不过我认为那个开关的位置设置得很糟糕。因此，如果你使用的是原版 Manjaro 和 Debian 安装，可能需要以同时更新 U-Boot 的方式升级，或者将 FreeBSD 安装到 eMMC 上，亦或按上述方式使用其断电开关。

最后，切记，用于引导的 U-Boot 应当是打过补丁的版本。

## 快速开始

稍后我会介绍如何从源码安装，但你可以直接从 [GitHub](https://github.com/jsm222/drm-subtree/releases) 获取测试镜像，以及后面提到的修改后软件包。  

如果你已经禁用了 eMMC，或者已为 Pinebook Pro 安装了补丁版 U-Boot（你可以在[这里](https://github.com/jsm222/u-boot-pinebookpro/releases/tag/0.1)找到），那么你就可以开始了。请注意，RELEASE(7) 默认创建了两个不安全的用户：`root`（密码：`root`）和 `freebsd`（密码：`freebsd`）。SSH 也是默认启用的。这样的默认设置适用于 ARM 开发板，但对于笔记本来说并不太合适。此外，别忘了将自己的用户添加到 `video` 组，否则图形堆栈将无法访问图形设备节点。  



## 联网

网络连接当然是必需的，但目前内置 Wi-Fi 仍然不受支持。我选择了一个基于 `rtwn_usb` 芯片组的 Wi-Fi 网卡。你也可以在 M.2 插槽中安装一张 Wi-Fi 网卡（如果你有 PCI 桥扩展）。不过，我自己并没有尝试后者。此外，你的手机也可以通过 USB 共享网络，让 Pinebook Pro 联网。  

对于 Wi-Fi 网卡，你需要知道如何通过命令行连接。我建议你直接编辑 `/etc/rc.conf` 进行配置，具体请参考 FreeBSD [手册](https://docs.freebsd.org/en/books/handbook/advanced-networking/#network-wireless)。如果使用手机共享网络，则只需运行 `dhclient ue0` 即可。  



## 如何在 amd64 上交叉编译 FreeBSD 源码

测试镜像基于 14-CURRENT，提交哈希为 `c9e023541aef`。要构建它，你可以使用我的 `PINEBOOKPRO.conf` 和 `release.sh`，它们可以在 [people.freebsd.org/~jsm/pbp](https://people.freebsd.org/~jsm/pbp) 找到。

```sh
./release.sh -c arm64/PINEBOOKPRO.conf
```

但编译需要一段时间。（在一台 10 代 10 核 Intel CPU 上大约需要 1.5 到 2 小时。）  

在撰写本文时，Panfrost 仍处于开发阶段，因此我修改了 Ruslan Bukin 提交的最新 PR（针对 `drm-subtree`），以避免使用连续内存。因为在相对高负载（比如在 Firefox 中运行 [webglsamples.org](https://webglsamples.org) 的水族馆示例）时，Pinebook Pro 似乎很难保持足够的连续空闲内存。我将 Panfrost 构建为模块，因为如果将其编译为设备，系统在启动时会崩溃。要构建该模块，你可以 `chroot` 到 `release.sh` 的 `scratchdir` 目录，并在 `/usr/src` 目录下设置以下环境变量进行编译：

```sh
setenv TARGET arm64
setenv WORKSPACE /usr
setenv MAKEOBJDIRPREFIX $WORKSPACE/obj/
setenv ROOTFS $WORKSPACE/rootfs
setenv SRC /usr/src
setenv MAKESYSPATH $SRC/share/mk
```

使用：

```sh
make buildenv TARGET_ARCH=aarch64 BUILDENV_SHELL=/bin/sh
```

我编写了一个 `Makefile` 并修复了一些编译时错误，主要是添加函数原型和修正 `printf` 格式字符串。在 `buildenv` 中，将目录切换到 `/usr/src/sys/dev/drm/panfrost`，然后执行以下命令：

```sh
make
make DESTDIR=$ROOTFS install
```

然后，你就可以通过修改

```sh
-   if (1 == 1)
+   if (1 == 0)
           panfrost_alloc_pages_iommu(bo);
    else
           panfrost_alloc_pages_contig(bo);
```

在 `panfrost_gem.c` 中进行了修改。它没有 `sysctl` 控制选项，而是直接修改了代码。详细信息请参考 `drm-subtree` 讨论，pull [#13](https://github.com/evadot/drm-subtree/pull/13)。  

## Port 和修改后的软件包  

如前所述，我将 `sway`、`wlroots` 和 `hikari` 还原到了较早的版本。同时，我对 `libdrm` 进行了修改，以便通过此[补丁](https://gist.github.com/jsm222/7df208cc5a72918a70cfbef8ee15b51b)检测 `panfrost`。`Hikari` 还需要一个小补丁来修复其参数解析[问题](https://hub.darcs.net/raichoo/hikari/issue/20)。 此外，`mesa-dri` 和 `mesa-libs` 也经过修改，以便编译 `panfrost` 驱动程序，并启用 `gles1` 和 `gles2`，即按照 Ruslan Bukin 在其文章中描述的方式进行编译。  

这些软件包均已预编译，可直接下载。同时，如果你更倾向于从源码构建，我也提供了 `ports` 树的补丁。你可以利用 `pkg lock` 功能，防止 `pkg` 命令重新安装这些修改后的软件包。只需运行 `pkg lock`。在我的系统上，我锁定了以下软件包：

```sh
hikari-2.3.2
libdrm-2.4.109,1
mesa-dri-21.3.6
mesa-libs-21.3.6
sway-1.6.1_2
wlroots-0.14.1_2
```

## 给 RockPRO64 用户的注意事项  

请不要忘记在测试镜像上重新安装来自 `ports` 的 RockPRO64 `u-boot`，另外请注意，音频尚未打入补丁。  

---

**JESPER SCHMITZ MOURIDSEN** 是一名自学成才的系统管理员和开发者，目前担任系统管理员，主要从事 OpenStack 相关工作。他是 FreeBSD `ports` 的提交者，主要关注 `LXQt`，并且是 `rtsx(4)` 的合著者。工作之余，他喜欢骑自行车，是一名骑行爱好者。
