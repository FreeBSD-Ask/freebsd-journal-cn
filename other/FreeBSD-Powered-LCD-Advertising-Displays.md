# 由 FreeBSD 驱动的中国 LCD 广告显示屏

- 原文：[FreeBSD-powered LCD Advertising Displays](https://freebsdfoundation.org/wp-content/uploads/2017/03/FreeBSD-Powered-LCD-Advertising-Displays.pdf)
- 作者：李潇

2011 年，在上海至徐州的高速铁路沿线 11 个候车室中安装了 101 台独立式 LCD 广告显示屏。2013 年，在沈阳至大连的高速铁路沿线 4 个候车室中安装了 7 座 LCD 广告显示塔。在这些设备中，运行的是定制的 FreeBSD 镜像，配合 Xorg，在 Intel CPU 和闪存上工作。 FreeBSD 强健而稳定的体系结构，确保广告图片和视频能够稳定播放，并且在广告程序的控制下，显示塔中的电机能够正确运行。

自 2000 年以来，LCD、LED 等电子显示设备已在全球范围内被广泛用于广告。在这些广告设备内部，为嵌入式系统编译的开源软件负责支持音频和视频播放、访问 SD 卡或 USB 存储设备，以及网络连接和远程控制。

自 2000 年以来，中国建成了世界上最长的高速铁路线路 [1]。为了开发这些高速铁路沿线车站候车室的经济价值，安装了 LCD 和 LED 显示设备，用于播放商业广告图片和视频。

从 2011 年到 2013 年，我曾作为其中部分场所的广告设备提供商，并在我的产品中采用了 FreeBSD 配合 Xorg 作为操作系统。 FreeBSD 以及我所开发的软件产品的稳定性，使许多客户感到惊讶，因为软件故障极为罕见。

## 产品与安装

我分别在 2011 年和 2013 年获得了两份设备供应合同，分别用于京沪高速铁路的一段线路以及哈大高速铁路的一段线路。上方的地图展示了这两条线路在中国大陆上的地理位置 [17]。

### 京沪高速铁路线路

在京沪高速铁路线路（全长约 1318 公里 [6]）中，上海至徐州这一段（约 626 公里 [6]）[2–3]，客户订购了第 10 页插图照片 [9] 所示的独立式 LCD 广告显示屏。这些设备安装在图 1 所列各铁路车站的候车室中。

用于运行 FreeBSD 的硬件配置如表 1 所示 [7–8]。在 2011 年，以太网和音频控制器已经分别由 FreeBSD 的 re(4) 和 snd_hda(4) 驱动完全驱动。但对 FreeBSD 来说，GPU 仍属较新的硬件，需要更新的 Xorg 驱动 xf86-video-intel29，而该驱动并不属于 FreeBSD Ports 中 Xorg 依赖的主线。

当时，Xorg 已在 GNU/Linux 社区中引入了 kernel mode setting（KMS）机制，但 FreeBSD 尚未跟进。幸运的是，x11-drivers/xf86-video-intel29 能够在 user mode 下与 Xorg 7.5.1 以及板载 GPU 良好配合。显示模式可以平滑设置为 1920×1080p@60Hz，完全符合显示器的最佳性能要求。此外，Xorg 的 XVideo 扩展也能够正常工作。该驱动防止了在播放视频时 CPU 负载过高。


<img width="322" height="199" alt="O3CI5B@ET09_J(L({KJ2F 5" src="https://github.com/user-attachments/assets/f79d72f1-4401-4829-9493-f9b18fc387ac" />

**图 1. 京沪高速铁路主线。自 2016 年起，我的一些产品已从部分铁路车站撤除，例如南京南站。**


| 组件  | 规格                                  |
| --- | ----------------------------------- |
| 主板  | Intel Desktop Board D410PT 或 D425KT |
| CPU | Intel Atom D410 或 D425              |
| GPU | Intel GMA 3150                      |
| 芯片组 | Intel NM10                          |
| 以太网 | Realtek 8103EL 或 8105E              |
| 音频  | Realtek ALC662                      |
| 内存  | 1 GB 或 2 GB                         |
| 存储  | 8 GB，USB 存储设备                         |

**表 1. 用于京沪高速铁路线路的我方产品的硬件配置。**

<img width="303" height="191" alt="OGK4FBU L$XE0O(C 1KV)QY" src="https://github.com/user-attachments/assets/f64faef7-3ae8-4e37-8d93-e923c2ca298d" />

**图 2. 哈大高速铁路线路中的沈阳—大连区段。**


<img width="415" height="304" alt="T31DS{G AY50B B~1WSFM)U" src="https://github.com/user-attachments/assets/828ebd7a-0ac5-40b8-b593-e310d36cc29e" />

LCD 广告显示塔。正在播放由新华社定制的广告内容。


| 组件  | 规格                                                         |
| --- | ---------------------------------------------------------- |
| 主板  | GIGABYTE GA-B75-D3V                                        |
| CPU | Intel Core i5（LGA1155）                                     |
| GPU | Intel HD Graphics 2500（板载）<br>NVIDIA GeForce 210（PCI-E，双卡） |
| 芯片组 | Intel B75                                                  |
| 以太网 | Atheros GbE（板载，未使用）<br>Realtek RTL8139（PCI）                |
| 音频  | Realtek ALC887                                             |
| 内存  | 2 GB                                                       |
| 存储  | SATA SSD 120 GB，Intel                                      |

**表 2. 用于哈大高速铁路线路的我方产品的硬件配置。**

### 哈大高速铁路线路

在哈大高速铁路线路（全长约 918 公里 [6]）中，沈阳至大连这一段（约 380 公里 [6]），客户订购了左侧照片 [16] 所示的 LCD 广告显示塔。这些设备安装在图 2 [4–5] 所列各铁路车站的候车室中。

用于运行 FreeBSD 的硬件配置如表 2 所示 [10–11]。在 2013 年，以太网控制器 Atheros GbE 尚未被 FreeBSD 支持。但我没有足够的时间自行修改驱动，只能在主板上额外安装一块使用 RTL8139 的 PCI 以太网卡。

在该应用场景中，每一台主机被设计为通过 VGA 接口驱动 5 块 LCD 显示屏。除板载 GPU 之外，我还必须在每块主板上安装两块带有 VGA 和 DVI-I 接口的 PCIe 显卡。当时，NVIDIA 的 GPU 能够通过其闭源驱动正常工作 [12]，而 Intel GPU 只能由 xf86-video-vesa 驱动。所幸，由于使用了 Xorg 的 Xinerama 扩展，将 5 块 LCD 显示屏组合为一个虚拟显示器，XVideo 扩展并非必需（实际上，Xinerama 会禁用大多数硬件渲染）；在这种情况下，xf86-video-vesa 已经足够使用。

## 技术要点

定制的 FreeBSD 镜像，连同 Xorg 以及我自己的软件，被写入 USB 存储设备或 SATA SSD 中。

### 定制的 FreeBSD 文件集

为简化工作，在每个 USB 存储设备或 SATA SSD 上都创建了一个 MBR 分区表。每个存储设备只创建一个分区，并在该分区中创建一个 BSD label “a”。切片 `/dev/da0s1a`（用于 USB 存储设备）或 `/dev/ada0s1a`（用于 SATA SSD）被格式化为启用软更新和日志功能的 FreeBSD FFS2 文件系统。

在实际使用中，对存储设备的写入操作并不频繁，每天不超过约 100 MB。但异常断电的情况每天可能发生 10 次。因此，健壮的文件系统能够有效降低启动失败的概率。得益于启用软更新和日志的 FreeBSD FFS 2，近年来我产品的启动失败仅由硬件损坏导致。

切片 `/dev/da0s1a` 或 `/dev/ada0s1a` 被用作 FreeBSD 的根文件系统，其层次结构如表 3 所示。自从我发现 Martin Matuška [13] 制作的 “mfsBSD” 之后，便参考并研究了他的相关工作。

### 定制的 Xorg 文件集

为了运行 GUI，需要将一套 Xorg 文件集写入文件系统，其层次结构如表 4 所示。

我并未将任何典型的 X11 工具包（例如 GTK）集成到闪存中。

### 使用 wxWidgets

wxWidgets [14] 已被收录在 FreeBSD Ports 中，默认是基于 GTK 2 编译的，但 wxWidgets 也可以直接与 X11 配合工作。基于 X11 的 wxWidgets Port 称为 wxX11。尽管目前 wxX11 仍不完整且存在一些缺陷，但它已经能够很好地支持图片文件处理以及基本的子窗口管理，这些功能正是播放广告内容所必需的。

在我的产品中，wxWidgets 2.8.x 可以配置为：

```sh
./configure --with-x11 --with-opengl\
--disable-shared --enable-unicode
```

在编译 wxX11（需要 GNU make）之后，可以使用脚本 wx-config 来获取用于 wxX11 头文件和库文件的 C++ 编译与链接参数。相关命令形式如下：

```sh
# 编译 C++ 源文件
~/wxWidgets-2.8.12/wx-config --cxxflags
# 链接 object 文件
~/wxWidgets-2.8.12/wx-config --libs
~/wxWidgets-2.8.12/wx-config --gl_libs
```

### 通过 RS-232-C Port 进行电机控制

为简化程序，在我的程序打开 `/dev/ttyu0` 之前，先对 `/dev/ttyu0.init` 使用 stty(1) 来设置默认波特率、位数以及其他选项。

板载串行 Port 连接到我自行设计的一块印刷电路板（PCB），用于设定电机的行为：启动与停止、旋转方向以及转速。同时，PCB 上传感器采集到的一些物理量（例如环境温度和角位移）也通过该串行 Port 发送到 FreeBSD。

| 目录                                   | 说明                                                                    |
| ------------------------------------ | --------------------------------------------------------------------- |
| `/boot/`                               | FreeBSD 内核与驱动模块，loader(8) 及其配置文件                                      |
| `/libexec/`                            | 运行时链接编辑器，`ld-elf.so.1`                                                  |
| `/bin/`、`/sbin/`、`/usr/bin/`、`/usr/sbin/` | 命令行工具（例如 `/bin/sh`、`/bin/rm`、`/usr/bin/killall`、`/sbin/ifconfig`、`/sbin/mount`） |
| `/dev/`                                | 用于 devfs(5) 的空目录                                                      |
| `/var/`、`/tmp/`                         | 多用途文件（例如 `/var/empty`）                                                  |
| `/etc/`                                | 初始化脚本和配置文件                                                            |
| `/lib/`, /usr/lib/                     | 共享对象                                                                  |
| `/usr/local/`                          | Xorg                                                                  |
| `/root/`                               | 我自己的软件文件                                                              |

**表 3. USB 存储设备或 SATA SSD 中的文件系统的概览布局。**

### 网络 

系统中启动了配置为使用公钥认证的 SSH 守护进程。 SSH 能够为各种远程访问提供安全且多用途的通道。通过 ssh(1) 实现类似 rsh(1) 的远程命令执行，使得网络交互可以简化为编写一些小型的 `/bin/sh` 脚本，从而避免设计繁琐且容易出错的传输协议。

SSH 还可以作为 rsync(1) 的安全传输机制，用于传输大体量文件。

### 自定义 `/etc/rc` 脚本

在 FreeBSD 内核启动之后，通常会执行 `/sbin/init`，它会尝试运行 `/etc/rc` 来完成大多数必要的初始化工作 [15]。我定制的 `/etc/rc` 脚本包含了从完整的 FreeBSD base 和完整的 Xorg 发行中精简出来的初始化步骤。在脚本末尾，它会启动我自己的程序。该脚本的主流程如下：

1. 等待 USB 设备在数秒内就绪。
2. 在可能的 `/dev/da0s1a` 和 `/dev/ada0s1a` 集合中查找并检查存储设备，并以读写模式挂载。
3. 在可能的 re(4)、rl(4) 和 em(4) 集合中查找并配置网络接口，同时设置网络路由。在 VirtualBox 中调试文件集时使用 em(4) 驱动。
4. 通过 adjkerntz(8) 调整内核的时区设置。
5. 调整音量。
6. 启动 SSH 守护进程。
7. 启动 Xorg 服务器。
8. 启动我自己的程序。

| 目录                | 说明                                     |
| ----------------- | -------------------------------------- |
| `bin/`              | 独立可执行文件（例如 Xorg、xrandr、xkbcomp）        |
| `etc/`              | xorg.conf 以及 font-config 和 pango 的配置文件 |
| `lib/`              | 共享对象                                   |
| `lib/X11/fonts/`    | 基本字体                                   |
| `lib/dri/`          | DRI 驱动                                 |
| `lib/pango/`        | Pango 模块                               |
| `lib/xorg/modules/` | 各类驱动                                   |
| `share/X11/xkb/`    | 与键盘相关的资源文件                             |

**表 4. 位于 `/usr/local/` 下的定制 Xorg 文件布局。我没有将任何典型的 X11 工具包（例如 GTK）集成到闪存盘中。**

## 建议


在 FreeBSD 的源代码树中，有个很棒的东西——ndis(4)。它能在 FreeBSD 内核中运行为 Microsoft Windows 编写的网络设备驱动程序。

但这个模块缺乏维护。它往往无法与最新的 Windows 设备驱动配合工作，尤其是无线以太网设备驱动。

此外，ndis(4) 只能覆盖相当有限的一部分网络设备。事实上，在现实中，尽管 FreeBSD 拥有一个非常健壮且设计良好的内核，但由于这些不足，它在某些方面让人更倾向于 GNU/Linux。

对于 FreeBSD 的未来，我的建议与设备驱动有关：

1. 将 ndis(4) 的 Windows 内核兼容层从 FreeBSD 内核中移至用户态，这意味着该兼容层和设备驱动将作为进程运行。在用户态下，所有程序都比在内核态更容易修改和调试，至少，大多数用户态中的故障不会导致内核崩溃。
2. 扩展该兼容层的覆盖范围，即支持更多的网络设备驱动，并尝试覆盖其他类型的设备驱动，例如 GPU 的驱动。

## 致谢

感谢梁莉（亦称 Kylie Liang）在本文写作过程中给予的帮助。

---

李潇是一名生活在中国的软件与硬件工程师。他在北京经营着自己的小型公司。他是一位经验丰富的 FreeBSD 开发者。2006 年，他参与了 LaTeX-CJK 以及 Linux 兼容层的 ports 工作。他积极参与中国的 FreeBSD 社区。他对印刷电路板设计以及跨平台软件开发感兴趣。

## 参考文献

- [1] “High-speed rail in China,” 维基百科，在线访问：[https://en.wikipedia.org/wiki/High-speed_rail_in_China](https://en.wikipedia.org/wiki/High-speed_rail_in_China)
- [2] “Beijing–Shanghai High-Speed Railway,” 维基百科，在线访问：[https://en.wikipedia.org/wiki/Beijing%E2%80%93Shanghai_High-Speed_Railway](https://en.wikipedia.org/wiki/Beijing%E2%80%93Shanghai_High-Speed_Railway)
- [3] （仅中文）“京沪高速铁路，”百度百科，在线访问：[http://baike.baidu.com/view/141756.htm](http://baike.baidu.com/view/141756.htm)
- [4] “Harbin-Dalian High-Speed Railway,” 维基百科，在线访问：[https://en.wikipedia.org/wiki/Harbin%E2%80%93Dalian_High-Speed_Railway](https://en.wikipedia.org/wiki/Harbin%E2%80%93Dalian_High-Speed_Railway)
- [5] （仅中文）“哈大高速铁路，”百度百科，在线访问：[http://baike.baidu.com/view/1213823.htm](http://baike.baidu.com/view/1213823.htm)
- [6] （仅中文）高铁网，在线访问：[http://www.gaotie.cn/](http://www.gaotie.cn/)
- [7] “Intel® Desktop Board D410PT 技术产品规格，”Intel，在线访问：[http://www.intel.com/content/www/us/en/support/boards-and-kits/desktop-boards/000021877.html](http://www.intel.com/content/www/us/en/support/boards-and-kits/desktop-boards/000021877.html)
- [8] “Intel Desktop Board D425KT/D425KTW 技术产品规格，”Intel，在线访问：[http://www.intel.com/content/www/us/en/support/boards-and-kits/desktop-boards/000020992.html](http://www.intel.com/content/www/us/en/support/boards-and-kits/desktop-boards/000020992.html)
- [9] Mega-info Media 有限公司，官方网站：[http://www.megainfomedia.com/](http://www.megainfomedia.com/)
- [10] GA-B75-D3V 用户手册，技嘉科技有限公司，在线访问：[http://download.gigabyte.cn/FileList/Manual/mb-manual_ga-b75-d3v_v1.2_e.pdf](http://download.gigabyte.cn/FileList/Manual/mb-manual_ga-b75-d3v_v1.2_e.pdf)
- [11] Intel Core i5-3450 介绍，Intel，在线访问：[http://ark.intel.com/products/65511/Intel-Corei5-3450-Processor-6M-Cache-up-to-3_50-GHz](http://ark.intel.com/products/65511/Intel-Corei5-3450-Processor-6M-Cache-up-to-3_50-GHz)
- [12] FreeBSD 版 GeForce 驱动，NVIDIA，在线访问：[http://www.geforce.com/drivers](http://www.geforce.com/drivers)
- [13] Martin Matuška，“mfsBSD”，在线访问：[http://mfsbsd.vx.sk/](http://mfsbsd.vx.sk/)
- [14] wxWidgets，跨平台 C++ GUI 库，在线访问：[http://www.wxwidgets.org/](http://www.wxwidgets.org/)
- [15] “内核初始化，”《FreeBSD 架构手册》，在线访问：[https://www.freebsd.org/doc/en_US.ISO8859-1/books/arch-handbook/boot-kernel.html](https://www.freebsd.org/doc/en_US.ISO8859-1/books/arch-handbook/boot-kernel.html)
- [16] 中国新华社，官方网站：[http://www.news.cn/english/](http://www.news.cn/english/)
- [17] 中国大陆重新制作的地理概览图，参考政府数字地图网站“天地图”，在线访问：中文：[http://map.tianditu.com/](http://map.tianditu.com/) 英文：[http://en.tianditu.com/](http://en.tianditu.com/)

