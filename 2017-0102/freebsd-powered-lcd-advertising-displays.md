# 由 FreeBSD 驱动的 LCD 广告显示屏

作者：李潇（Li, Xiao）

## 由 FreeBSD 驱动的 LCD 广告显示屏

2011 年，在上海至徐州的高速铁路沿线 11 个候车室中，安装了 101 台独立式 LCD 广告显示屏。2013 年，在沈阳至大连的高速铁路沿线 4 个候车室中，安装了 7 座 LCD 广告显示塔。这些设备中，定制的 FreeBSD 镜像与 Xorg 运行在 Intel CPU 和闪存之上。FreeBSD 稳健的底子保证了广告图片和视频的稳定播放，以及塔中的电动机在广告程序控制下正确运转。

> 上图：独立式 LCD 显示屏。仅下层 LCD 由运行 FreeBSD 的主机驱动，正在播放由 Mega-info Media 定制的广告图片。

> 哈尔滨──沈阳──北京──大连──上海

> 中国大陆京沪线和哈大线的地理位置。

自 2000 年起，LCD 和 LED 等电子显示屏在全球范围内被广泛用于广告。在这些广告设备内部，为嵌入式系统编译的开源软件支持音视频播放、SD 卡或 U 盘访问、联网和远程控制。

自 2000 年起，中国建成了世界上最长的高速铁路线 [1]。为开发这些高铁线路沿线车站候车室的经济价值，安装了 LCD 和 LED 显示屏以播放商业广告图片和视频。2011 至 2013 年间，我作为某些此类地点的广告设备供应商，采用 FreeBSD 和 Xorg 作为产品的操作系统。FreeBSD 的稳定性加上我的软件产品出乎意料地极少出现软件故障，令许多客户惊讶。

## 产品与安装

2011 年和 2013 年，我分别获得了两份设备供应合同，分别对应京沪高铁的一段和哈大高铁的一段。上面的地图显示了中国大陆这两条线路的地理位置 [17]。

### 京沪高速铁路线

> G = 装有我产品的火车站　H = 未装我产品的火车站

在上海至徐州段（约 626 公里 [6]）的京沪高铁 [2-3]（约 1318 公里 [6]）沿线，客户订购了第 10 页插图所示的独立式 LCD 广告显示屏 [9]。它们被安装在图 1 所列各火车站的候车室中。运行 FreeBSD 的硬件配置见表 1 [7-8]。2011 年时，以太网和音频控制器已被 FreeBSD 的 `re(4)` 与 `snd_hda(4)` 驱动完整支持。但 GPU 对 FreeBSD 而言尚属新事物，需要一个较新的 Xorg 驱动 `xf86-video-intel29`，它并不属于 Xorg 依赖的主线，位于 FreeBSD Ports 树中。当时 Xorg 已在 GNU/Linux 社区引入了内核模式设置（KMS）机制，但 FreeBSD 还未跟上。幸运的是，`x11-drivers/xf86-video-intel29` 在 Xorg 7.5.1 和板载 GPU 的用户态下工作良好。显示模式可以平滑设置为 1920×1080p@60Hz，完全符合显示器最佳性能。Xorg 的 X-Video 扩展在该驱动下也工作良好，避免播放视频时 CPU 过载。

**表 1. 京沪高铁线产品的硬件配置**

| 组件 | 型号 |
| --- | --- |
| 主板 | Intel Desktop Board D410PT 或 D425KT |
| CPU | Intel Atom D410 或 D425 |
| GPU | Intel GMA 3150 |
| 芯片组 | Intel NM10 |
| 以太网 | Realtek 8103EL 或 8105E |
| 音频 | Realtek ALC662 |
| 内存 | 1GB 或 2GB |
| 存储 | 8GB U 盘 |

> 图 1. 京沪高速铁路线主线。自 2016 年起，我的一部分产品已从南京南站等一些车站撤除。

### 哈大高速铁路线的沈阳至大连段

> G = 装有我产品的火车站　H = 未装我产品的火车站

在沈阳至大连段（约 380 公里 [6]）的哈大高铁（约 918 公里 [6]）沿线，客户订购了左侧照片所示的 LCD 广告显示塔 [16]。它们被安装在图 2 所列各火车站的候车室中 [4-5]。运行 FreeBSD 的硬件配置见表 2 [10-11]。2013 年时，Atheros GbE 以太网控制器尚未被 FreeBSD 支持。但我没有足够时间自己 hack 驱动，只得在主板上额外安装了一块带 RTL8139 的 PCI 以太网卡。在此案例中，每台主机都设计为通过 VGA 端口驱动 5 台 LCD 显示屏。除板载 GPU 外，我还需在每块主板上加装两块带 VGA 和 DVI-I 端口的 PCI-E 显卡。当时 NVIDIA 的 GPU 在 NVIDIA 闭源驱动下工作良好 [12]，但 Intel GPU 只能用 `xf86-video-vesa` 驱动。再次幸运的是，由于应用了 Xorg 的 Xinerama 扩展将 5 台 LCD 显示屏合并为单个虚拟显示屏，XVideo 扩展并非必需（实际上 Xinerama 会禁用大部分硬件渲染），`xf86-video-vesa` 对本案例已经足够。

**表 2. 哈大高铁线产品的硬件配置**

| 组件 | 型号 |
| --- | --- |
| 主板 | GIGABYTE GA-B75-D3V |
| CPU | Intel Core i5（LGA1155） |
| GPU | Intel HD Graphics 2500（板载）+ NVIDIA GeForce 210（PCI-E，两块） |
| 芯片组 | Intel B75 |
| 以太网 | Atheros GbE（板载，未使用）+ Realtek RTL8139（PCI） |
| 音频 | Realtek ALC887 |
| 内存 | 2GB |
| 存储 | Intel SATA SSD 120GB |

> 图 2. 哈大高速铁路线的沈阳至大连段。

> LCD 广告显示塔，正在播放新华社定制的广告内容。

## 技术要点

定制的 FreeBSD 镜像连同 Xorg 和我的软件被写入 U 盘或 SATA SSD。

### 定制的 FreeBSD 文件集

为简化工作，每块 U 盘或 SATA SSD 上都创建了 MBR 分区表。每块盘上仅创建一个分区，并在该分区中创建 BSD 标签 "a"。切片 **/dev/da0s1a**（U 盘）或 **/dev/ada0s1a**（SATA SSD）被格式化为带软更新和日志的 FreeBSD FFS 2。实际使用中，对盘的写入操作并不频繁，每天不超过约 100 MB。但异常断电每天可能发生 10 次。因此，稳健的文件系统能有效减少启动失败。得益于带软更新和日志的 FreeBSD FFS 2，我的产品启动失败最近只由硬件损坏引起。切片 **/dev/da0s1a** 或 **/dev/ada0s1a** 用作 FreeBSD 的根文件系统。其层次结构见表 3。自从我发现 Martin Matuška 制作的 "mfsBSD" [13] 之后，我查阅了他的工作以作参考。

**表 3. U 盘或 SATA SSD 中文件系统的概览布局**

| 路径 | 内容 |
| --- | --- |
| **/boot/** | FreeBSD 内核与驱动模块、`loader(8)` 及配置文件 |
| **/libexec/** | 运行时链接器 ld-elf.so.1 |
| **/bin/**、**/sbin/**、**/usr/bin/**、**/usr/sbin/** | 命令行工具（如 **/bin/sh**、**/bin/rm**、**/usr/bin/killall**、**/sbin/ifconfig** 和 **/sbin/mount**） |
| **/dev/** | 用于 devfs(5) 的空目录 |
| **/var/**、**/tmp/** | 多用途文件（如 **/var/empty**） |
| **/etc/** | 初始化脚本与配置文件 |
| **/lib/**、**/usr/lib/** | 共享对象 |
| **/usr/local/** | Xorg |
| **/root/** | 我自己的软件文件 |

### 定制的 Xorg 文件集

为运行 GUI，必须将 Xorg 的一套文件写入文件系统。层次结构见表 4。我没有把任何典型的 X11 工具包（如 GTK）集成到闪存盘中。

**表 4. **/usr/local/** 下定制 Xorg 的概览文件布局**

| 路径 | 内容 |
| --- | --- |
| **bin/** | 独立可执行文件（如 Xorg、xrandr 和 xkbcomp） |
| **etc/** | xorg.conf 及 font-config 与 pango 的配置文件 |
| **lib/** | 共享对象 |
| **lib/X11/fonts/** | 基础字体 |
| **lib/dri/** | DRI 驱动 |
| **lib/pango/** | Pango 模块 |
| **lib/xorg/modules/** | 驱动 |
| **share/X11/xkb/** | 键盘相关资源文件 |

### 使用 wxWidgets

wxWidgets [14] 已被收录 FreeBSD Ports 树，它使用 GTK 2 编译，但 wxWidgets 也可以直接与 X11 协作。与 X11 配合的 wxWidgets port 称为 wxX11。尽管 wxX11 目前不完整且有 bug，但它能很好支持图片文件处理和基础子窗口管理，这已可满足播放广告内容的需求。对于我的产品，wxWidgets 2.8.x 可按如下方式配置：

```sh
./configure --with-x11 --with-opengl\
 --disable-shared --enable-unicode
```

编译 wxX11 后（需要 GNU make），脚本 `wx-config` 可用于获取 wxX11 头文件和库文件的 C++ 编译与链接参数。命令如下：

```sh
# 编译 C++ 源文件
~/wxWidgets-2.8.12/wx-config --cxxflags
# 链接目标文件
~/wxWidgets-2.8.12/wx-config --libs
~/wxWidgets-2.8.12/wx-config --gl_libs
```

### 联网

系统中启动了配置公钥认证的 SSH 守护进程。SSH 可为各类远程访问提供安全且多用途的通道。通过 `ssh(1)` 执行类似 `rsh(1)` 的远程命令，将网络化简为编写一些小型 **/bin/sh** 脚本，避免设计繁琐且易出错的传输协议。SSH 还可作为 `rsync(1)` 的安全传输机制，用于传输大文件。

### 通过 RS-232-C 端口控制电机

为简化我的程序，使用 `stty(1)` 对 **/dev/ttyu0.init** 设置默认波特率、比特数等选项，再让我的程序打开 **/dev/ttyu0**。板载串口与一块我设计的印刷电路板（PCB）相连，用于设置电机的行为：启动和停止、转动方向以及转速。同时，PCB 上传感器捕获的物理量（如环境温度和角位移）通过串口发送给 FreeBSD。

### 定制 **/etc/rc** 脚本

FreeBSD 内核启动后，通常会执行 **/sbin/init**，它会尝试运行 **/etc/rc** 完成大部分必要的初始化 [15]。我定制的 **/etc/rc** 脚本包含从完整 FreeBSD base 和完整 Xorg 发行版中精简而来的初始化步骤。脚本最后启动我自己的程序。脚本主线如下：

1. 等待 USB 设备在数秒内就绪。
2. 在可能的 **/dev/da0s1a** 与 **/dev/ada0s1a** 集合中查找并检查存储设备，以读写模式挂载。
3. 在可能的 `re(4)`、`rl(4)` 和 `em(4)` 集合中查找并配置网络接口，同时设置网络路由。在 VirtualBox 中调试文件集时使用 `em(4)` 驱动。
4. 通过 `adjkerntz(8)` 调整内核时区设置。
5. 调节音量。
6. 启动 SSH 守护进程。
7. 启动 Xorg 服务器。
8. 启动我自己的程序。

## 建议

在 FreeBSD 源码树中，有一件好东西──`ndis(4)`。它可以在 FreeBSD 内核中运行为 Microsoft Windows 编写的网络设备驱动。但该模块缺乏维护，常常无法与最新的 Windows 设备驱动配合工作，尤其是无线网卡驱动。此外，`ndis(4)` 只能覆盖相当有限的网络设备。实际上，尽管 FreeBSD 拥有非常稳健且设计良好的内核，但它的这些短板却把人们推向了 GNU/Linux。为了 FreeBSD 的未来，我的建议与设备驱动有关：

1. 将 `ndis(4)` 的 Windows 内核兼容层从 FreeBSD 内核移至用户态，这意味着该层与设备驱动将在一个进程中运行。在用户态，所有程序都比在内核中更易 hack 和调试。至少用户态的大部分错误不会导致内核崩溃。
2. 扩展该兼容层的覆盖范围，即覆盖更多网络设备驱动，并尝试覆盖其他类型的设备驱动，如 GPU 驱动。

## 致谢

感谢 Liang, Li（又名 Kylie Liang）对本文的帮助。

---

**Li, Xiao** 是一位生活在中国北京的软硬件工程师，他在北京经营自己的小公司。他是经验丰富的 FreeBSD 开发者，2006 年曾参与 LaTeX-CJK 和 Linux 兼容性的 port 工作。他积极参与中国的 FreeBSD 社区，对印刷电路板设计和跨平台软件设计充满兴趣。

## 参考文献

[1] "High-speed rail in China," Wikipedia, 在线：https://en.wikipedia.org/wiki/High-speed_rail_in_China。

[2] "Beijing–Shanghai High-Speed Railway," Wikipedia, 在线：https://en.wikipedia.org/wiki/Beijing%E2%80%93Shanghai_High-Speed_Railway。

[3]（仅中文）"京沪高速铁路," 百度百科, 在线：http://baike.baidu.com/view/141756.htm。

[4] "Harbin-Dalian High-Speed Railway," Wikipedia, 在线：https://en.wikipedia.org/wiki/Harbin%E2%80%93Dalian_High-Speed_Railway。

[5]（仅中文）"哈大高速铁路," 百度百科, 在线：http://baike.baidu.com/view/1213823.htm。

[6]（仅中文）高铁网, 在线：http://www.gaotie.cn/。

[7] "Technical Product Specification for the Intel® Desktop Board D410PT," Intel, 在线：http://www.intel.com/content/www/us/en/support/boards-and-kits/desktop-boards/000021877.html。

[8] "Technical Product Specification for Intel Desktop Board D425KT/D425KTW," Intel, 在线：http://www.intel.com/content/www/us/en/support/boards-and-kits/desktop-boards/000020992.html。

[9] Mega-info Media Co. Ltd., 官方网站：http://www.megainfomedia.com/。

[10] GA-B75-D3V User's Manual, GIGA-BYTE Technology Co. Ltd., 在线：http://download.gigabyte.cn/FileList/Manual/mb-manual_ga-b75-d3v_v1.2_e.pdf。

[11] Introduction to Intel Core i5-3450, Intel, 在线：http://ark.intel.com/products/65511/Intel-Corei5-3450-Processor-6M-Cache-up-to-3_50-GHz。

[12] GeForce driver for FreeBSD, NVIDIA, 在线：http://www.geforce.com/drivers。

[13] Martin Matuška, "mfsBSD", 在线：http://mfsbsd.vx.sk/。

[14] wxWidgets, a cross-platform C++ GUI library, 在线：http://www.wxwidgets.org/。

[15] "Kernel Initialization," "FreeBSD Architecture Handbook," 在线：https://www.freebsd.org/doc/en_US.ISO8859-1/books/arch-handbook/boot-kernel.html。

[16] Xinhua News Agency, China, 官方网站：http://www.news.cn/english/。

[17] 一幅重新制作的中国大陆轮廓地图，参考自政府数字地图网站"天地图"（Tianditu 的中文含义为"天空-土地地图"或"天地地图"），在线：中文 http://map.tianditu.com/，英文 http://en.tianditu.com/。
