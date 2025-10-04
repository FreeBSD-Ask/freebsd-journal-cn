# FreeBSD WiFi 开发第一部分——尝试 WiFi

- 原文：[FreeBSD WiFi Development Part 1 – Experimenting with WiFi](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-3/freebsd-wifi-development/)
- 作者：Tom Jones

我在过去六个月中一直在 FreeBSD 上从事 WiFi 相关工作，该项目由 FreeBSD 基金会资助。这个项目的主要成果是将 OpenBSD 的 `iwx` 驱动移植到 FreeBSD，用于支持 Intel 802.11ac/ax 网卡。通过这个项目，我接触到了 WiFi 协议栈的大部分内容，也让我深刻认识到，我们需要更多的人参与 WiFi 的开发，同时也需要更简单的途径让新人能够开始进行开发。

WiFi 出现已经有 25 年了，已经成为我们生活的核心部分，以至于人们进入一个新地方的第一个问题往往是“这有 WiFi 吗？”

FreeBSD 缺乏 WiFi 驱动。在过去五年中，引入了带有 WiFi 支持的 linuxkpi 层，到 2025 年，这些驱动已经开始支持数百兆比特的 IEEE 802.11ac 速度。市面上的 WiFi 7 设备已经能达到 2Gbit 的速度。

FreeBSD 处于落后状态，这是无可争辩的事实。我们需要更多的人投入到 WiFi 相关的开发中。如果你对操作系统开发感兴趣，我想没有比参与改进 FreeBSD WiFi 协议栈更好的选择了。虽然这不会是最容易的事情，但正因为落后，我们有大量难度不一的开放任务，迫切需要更多贡献者来追赶。

本系列文章将讲述在 FreeBSD 上开发 WiFi 的路径。关于驱动开发的底层细节以及 LLVM 编译优化等内容，已经有很多优秀的资料。本篇作为三篇文章中的第一篇，将解释相关术语，并演示如何在 FreeBSD 上配置 WLAN 接口以进行测试。这个最小化配置足以开始发现问题。后续文章将讨论 WiFi 驱动的工作原理，以及它们如何与 FreeBSD 中更大的 `net80211` 协议栈交互。

## 术语

网络通信就是信息的有效传输和准确接收。我在互联网标准的开发和撰写方面工作了十年，但即使是通信领域的专家，我们仍然没能完全讲清楚通信的全貌。从 IETF 背景来看，WiFi 是那个让电子跳舞的怪异 IEEE 的东西。

要实现良好的通信，我们需要对术语达成共识，这让我想起多年前在奥斯陆大学（**译者注：这是挪威最大及最古老的大学**）食堂与一位德国同事的对话。

**我：** “我很确定它是 why-phi，就是 wireless fidelity（无线保真），就像 high fidelity（高保真）那样。”

**他：** “不，是 wee-fee（**译者注：发音相近**），像 hee-fee。”

在这些文章中我会用到很多缩写和术语，这是避免不了的。遇到不懂的，搜索互联网是你的好帮手。我刚提到的两个缩写 IETF（互联网工程任务组）和 IEEE（电气电子工程师学会）中，本文重点会围绕 IEEE。遗憾的是，在文档和代码中，FreeBSD WiFi 基础设施源代码里对 IEEE、net80211 和 IEEE80211 的称呼反复出现多种变体。

WiFi 联盟（负责认证的组织）在标准（来自 IEEE）和品牌名之间造成了一团乱。使用标准名称更准确，比如 `IEEE80211n`（或者简称 `11n`、`n`）比用 WiFi 4 更清晰。无论用哪个术语，都能帮助你搞清楚事物的名称、产品功能，或者别人问你的内容。FreeBSD 往往会倾向于使用标准名称，甚至是文档修订版本（如果幸运的话），而非营销名称。

理解 WiFi 的概念就像记名字一样困难。我建议你阅读一些资料，Matthew Ghast 的第一本关于 WiFi 的书 *802.11 Wireless Networks*（《802.11 无线网络权威指南》ISBN: 9787564103163）是非常好的入门读物，涵盖了所有主要概念，适合初学者。不用担心书的年代（2002），基础依旧，只是数字更大，调制方式更复杂罢了。

| 术语                     | 解释                                                                                            |
| ---------------------- | --------------------------------------------------------------------------------------------- |
| WiFi                   | IEEE 制定的一系列标准和 WiFi 联盟的市场品牌名称的总称。通俗来说，就是“你笔记本用来上网的那个东西”，这个定义对我们来说已经足够。如果有人过于纠正你的术语，那他并不是你的朋友。 |
| IEEE80211              | IEEE 802.11 是定义 WiFi 的标准家族。                                                                   |
| net80211               | 也称为协议栈，指 FreeBSD 中实现 IEEE80211 状态机的代码。                                                        |
| Band（频段）               | 客户端或接入点可能使用的频率范围。（例如常见的 2.4GHz 频段，2462 MHz 频率所在的频段）                                           |
| Channel（信道）            | 一个射频频率及其参数，有时与“频段”可互换使用。                                                                      |
| Station（站点）            | 你的设备、网络中的其他客户端（也是一种工作模式）。                                                                     |
| Access Point（接入点）      | 你连接的网络设备（也是一种工作模式）。                                                                           |
| Monitor（监视模式）          | 一种使网络适配器捕获该频段上所有数据包的工作模式。                                                                     |
| Network Adapter（网络适配器） | 含有所有无线电设备的设备，使你能够使用 WiFi（也称为网卡——虽然 USB 网卡不是卡，可能是网络接口，但最好避免混淆硬件和软件模型）。                         |
| Driver（驱动）             | FreeBSD 中直接或通过固件与网卡通信的代码。                                                                     |
| Firmware（固件）           | 运行在网卡上的代码，帮你完成部分工作。一般只在需要由操作系统或驱动加载时才会特别提及。                                                   |
| MAC（Media Access Control Address，媒体访问控制地址）                | 802.11 协议中负责共享媒介（介质访问控制）的部分。                                                                  |
| Full MAC（全 MAC）        | 用于驱动描述——全 MAC 设备实现了 MAC 层，net80211 做的许多工作可以被跳过。                                               |
| HT                     | 高吞吐量（High Throughput，也称为 802.11n）。                                                            |
| VHT                    | 非常高吞吐量（Very High Throughput，也称为 802.11ac）（**译者注：WIFI5**）。                                                    |
| EHT                    | 极高吞吐量（Extremely High Throughput，也称为 802.11ax）（**译者注：WIFI6**）。                                                |

本文还涉及一些你需要熟悉的核心概念。我认为对大多数人来说，这就是他们日常网络访问的工作方式，但值得注意的是，一些技术对我们中的某些人（包括我）来说是从小就接触的，或者是我们这代人才被引入的（刚才让不少人感觉自己老了），这些技术其实比部分读者的年龄还要长十年。

大多数家庭中最常见的 WiFi 网络是一种接入点（AP），它作为站点（或客户端）通往更大互联网的网关。在许多部署场景中，很可能就是你的家中，接入点通常是一个路由器，负责将客户端的流量转发到互联网（可能通过调制解调器）、分配地址以及执行其他大量网络任务。

要加入网络，站点需要经历一系列与接入点通信的状态。简而言之，它会：

- 扫描（scan）
- 探测（probe）
- 接收信标（receive beacons）
- 认证（authenticate）
- 关联（associate）
- 协商加密密钥（negotiate encryption keys）
- 获取 IP 地址（acquire an IP address）

除了最后一步外，其余步骤都是 WiFi 特有的，从某种角度看，这些步骤相当于你找到了网线、接口，并把电脑插入路由器。最后一步发生在 IP 层，使用的工具和有线网络一样。

所有这些阶段和状态都由 `net80211` 协议栈和设备驱动处理。具体谁来做什么取决于硬件支持，有些功能由网络适配器上的固件完成。当硬件无法完成时，`net80211` 协议栈可以自行实现大部分功能，但有些依赖无线电硬件的功能无法用软件模拟。

在 FreeBSD 上进行 WiFi 开发时，我们需要了解硬件和 `net80211` 层分别承担的任务。上面是简短的总结，实际上还有许多其他状态和认证模式。`IEEE80211` 标准已接近 30 年历史，拥有丰富的发展历程。

## 在 FreeBSD 上试验 WiFi

在开始阅读代码和做改动之前，我们先讨论一下如何在 FreeBSD 上管理 WLAN 适配器。

`net80211` 协议栈在网络适配器之上提供了一个抽象层，给我们虚拟接口。在代码中，这些接口被称为 VAP（虚拟访问点），具体功能可参考 `ieee80211_vap` 的手册页。

这意味着我们必须先从物理设备创建一个 WLAN 接口，才能使用。

相比 OpenBSD 中简单的接口管理命令（如 `ifconfig iwx0 up`），这种方式看起来有些笨重，但它能在单个适配器上实现虚拟功能。如果硬件支持，你甚至可以同时作为接入点和站点，或者同时作为站点和监控模式。

通常，这种管理工作由 `rc.conf` 中的配置自动完成，安装程序会添加类似如下行：

```ini
wlans_iwlwifi0="wlan0"
ifconfig_wlan0="WPA SYNCDHCP"
```

第一行让 `rc` 系统从 `iwlwifi0` 适配器创建一个名为 wlan0 的接口，第二行是类似有线接口的常见 `ifconfig` 配置。

内核开发时控制设备创建很有帮助，接下来我们先看看如何手动创建接口。

## 手动创建接口

可以通过读取 sysctl `net.wlan.devices` 来列出使用 `net80211` 注册的 WLAN 设备：

```sh
$ sysctl net.wlan.devices
net.wlan.devices: iwx0 rtwn0
```

在我的笔记本上，你能看到它有一块基于 `iwx` 的网卡（PCIe 上的 `iwx0`）和一块 `rtwn` 网卡（USB 上的 `rtwn0`）。

我们可以使用 ifconfig 命令从这些设备创建接口，如下所示：

```
# ifconfig wlan create wlandev iwx0
wlan0
# ifconfig wlan create wlandev rtwn0 wlanmode ap
wlan1
# ifconfig wlan create wlandev rtwn0 wlanmode monitor
wlan2
```

第一个例子中，我们在没有额外参数的情况下创建了 `wlan0`，默认工作模式是站点（station）。FreeBSD 中的 WLAN 设备在创建时只能工作于一种模式，具体支持哪些模式取决于硬件和驱动。

驱动支持的模式会在对应的手册页中列出。对比 `iwm`、`iwlwifi` 和 `iwx`（它们支持部分相同硬件），你会发现目前 `iwm` 和 `iwlwifi` 只能工作在站点模式，而 `iwx` 支持站点和监控模式。`iwx` 支持的 Intel 硬件能在 2.4GHz 频段有限度地作为主机接入点（host AP），但这部分支持尚未实现。

`rtwn` 驱动支持站点、adhoc、主机接入点（host AP）和监控模式。`net80211` 支持的完整模式列表可在 `ifconfig(8)` 手册页中查阅：

```
wlanmode mode
        指定此克隆设备的操作模式。mode 可为 sta、ahdemo（或 adhoc-demo）、ibss（或 adhoc）、AP（或 hostap）、wds、tdma、mesh 和 monitor。克隆接口的操作模式不能更改。tdma 模式实际上是具有特殊属性的 adhoc-demo 接口。
```

配置完成后，`ifconfig` 会显示一个接口，很多信息还未填充：

```sh
$ ifconfig wlan0
wlan0: flags=8802<BROADCAST,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=0
        ether e4:5e:37:af:13:5b
        groups: wlan
        ssid "" channel 1 (2412 MHz 11b)
        regdomain FCC country US authmode OPEN privacy OFF txpower 30
        bmiss 10 scanvalid 60 bgscan bgscanintvl 300 bgscanidle 250
        roam:rssi 7 roam:rate 1 wme bintval 0
        parent interface: iwx0
        media: IEEE 802.11 Wireless Ethernet autoselect (autoselect)
        status: no carrier
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
```

相比有线设备，`ifconfig` 输出中多了很多参数，我们来看几个关键的：

- `ssid "" channel 1 (2412 MHz 11b)`
  作为站点，我们有一个指定的 SSID（这里为空），当前处于信道 1，频率和模式也显示出来了。

- `regdomain FCC country US authmode OPEN privacy OFF txpower 30`
  监管域和区域代码默认设置为 FCC 和美国，启动接口后会更新以匹配本地监管域和区域。

- `bmiss 10 scanvalid 60 bgscan bgscanintvl 300 bgscanidle 250`
  驱动相关参数，控制扫描行为、网络切换和多媒体扩展（用于服务质量，不是播放 MP3 那个）。

- `parent interface: iwx0`
  父接口，方便管理多个 WiFi 接口。

- `media: IEEE 802.11 Wireless Ethernet autoselect (autoselect)`
  当前媒体模式，通常为自动选择，但调试或优化时可以强制设置。

如果看 `rtwn` 接口上创建的 VAP，虽然相似但略有不同，原因是它们的工作模式不同。

```sh
$ ifconfig wlan1
wlan1: flags=8802<BROADCAST,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=0
        ether 74:da:38:33:c0:62
        groups: wlan
        ssid “” channel 1 (2412 MHz 11b)
        regdomain FCC country US authmode OPEN privacy OFF txpower 30
        scanvalid 60 wme dtimperiod 1 -dfs bintval 0
        parent interface: rtwn0
        media: IEEE 802.11 Wireless Ethernet autoselect <hostap> (autoselect <hostap>)
        status: no carrier
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
$ ifconfig wlan2
wlan2: flags=8802<BROADCAST,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=0
        ether 74:da:38:33:c0:62
        groups: wlan
        ssid “” channel 1 (2412 MHz 11b)
        regdomain FCC country US authmode OPEN privacy OFF txpower 30
        scanvalid 60 wme bintval 0
        parent interface: rtwn0
        media: IEEE 802.11 Wireless Ethernet autoselect <monitor> (autoselect <monitor>)
        status: no carrier
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
```

当我们完成操作，或者如果不小心创建了错误模式的设备，可以用 ifconfig 来删除它：

```sh
# ifconfig wlan0 destroy
```

## 使用接口

我之前有点偷懒，FreeBSD 的 WiFi 栈主要由两个部分组成，但很多 WiFi 状态是由两个用户空间程序驱动的，分别是 `wpa_supplicant` 和 `hostapd`。

### 使用 wpa\_supplicant 的站点模式

`wpa_supplicant` 最初是一款管理 WPA（无线保护访问）加密状态的程序，适用于站点模式下的设备。它已发展成为完整的用户空间 WiFi 管理接口。Linux 上无线配置常用 `wpa_supplicant`。

`hostapd` 是同一项目的主机接入点用户空间守护进程，功能与 `wpa_supplicant` 相似，但它处理的是主机任务而非客户端任务。

我们可以用 ifconfig 管理 WLAN 接口，以下命令会启用站点接口并配置它连接名为“Test”的 SSID：

```sh
# ifconfig wlan0 ssid "Test" up
```

这会让 FreeBSD 站点尝试关联到名为“Test”的接入点。FreeBSD 的 `net80211` 协议栈不直接支持 WPA 状态机，因此大多数网络需要配合使用 `wpa_supplicant`。之前提到的 `rc.conf` 第二行配置即用于启动接口上的 `wpa_supplicant` 并启用 DHCP。

手动运行 `wpa_supplicant` 需要配置文件，`wpa_supplicant.conf` 的手册页里有很多示例，当中最简单的配置文件如下：

```ini
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=wheel

network={
        ssid="Open Network"
        key_mgmt=NONE
}
```

然后可以这样启动 `wpa_supplicant`：

```sh
# wpa_supplicant -i wlan0 -c /etc/wpa_supplicant.conf -D bsd
```

在默认情况下，`wpa_supplicant` 回在前台运行，加上 `-B` 参数则会后台运行。可以通过控制接口获取它的日志信息。

`wpa_passphrase` 命令可用来为 `wpa_supplicant` 配置文件生成 WPA 网络配置，例如：

```ini
$ wpa_passphrase "Closed Network" superpassword
network={
        ssid="Closed Network"
        #psk="superpassword"
        psk=852c26a07d84c48e4bfeec71289214a39bcd9d881bc66aedf6a2d11372f59752
}
```

这个命令只生成添加到 `wpa_supplicant.conf` 所需的配置，免去学习复杂语法的麻烦。

FreeBSD 自带 `wpa_cli` 工具，用于与 `wpa_supplicant` 交互。通过 `wpa_cli`，可以列出网络、连接（选择）、重新配置和断开连接。下面是一个连接 WPA 保护网络的示例会话。

```sh
$ wpa_cli
wpa_cli v2.11
Copyright (c) 2004-2024, Jouni Malinen <j@w1.fi> and contributors

This software may be distributed under the terms of the BSD license.
See README for more details.


Selected interface ‘wlan0’

Interactive mode

> list_networks
network id / ssid / bssid / flags
0       Open Network       [DISABLED]
1       Closed Network     [DISABLED]
> select_network 1
OK
<3>CTRL-EVENT-SCAN-RESULTS
<3>WPS-AP-AVAILABLE
<3>Trying to associate with 20:05:b6:fa:13:f1 (SSID=’Closed Network’ freq=5180 MHz)
<3>Associated with 20:05:b6:fa:13:f1
<3>WPA: Key negotiation completed with 20:05:b6:fa:13:f1 [PTK=CCMP GTK=CCMP]
<3>CTRL-EVENT-CONNECTED - Connection to 20:05:b6:fa:13:f1 completed [id=0 id_str=]
disable_network disconnect
> disconnect
OK
<3>CTRL-EVENT-DISCONNECTED bssid=20:05:b6:fa:13:f1 reason=3 locally_generated=1
<3>CTRL-EVENT-DSCP-POLICY clear_all
```

### 使用 hostapd 的 AP 模式

`hostapd` 使用较少，但它能让你启动自己的接入点（AP），并在其中进行完整的内核调试和数据包捕获，这对调试非常有帮助。

下面是一个针对我们示例中“Closed Network”的配置示例：

```ini
hostapd.conf:
```

```sh
ctrl_interface=/var/run/hostapd
ctrl_interface_group=wheel

interface=wlan1

# hw_mode=g
channel=8
ieee80211d=0
ieee80211n=0
wmm_enabled=0

# the name of the AP
ssid=”Closed Network”
# 1=wpa, 2=wep, 3=both
auth_algs=1
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
wpa_passphrase=”superpassword”
```


我们可以这样在前台运行 hostapd：

```sh
# hostapd hostapd.conf
```

运行 hostapd 后，我们就拥有了一个大部分功能齐全的接入点——WiFi 部分很简单！接下来我们还需要给 `wlan1` 分配一个地址，通常还会运行一个进程提供动态地址分配。

我们可以像平常一样给接口分配 IP 地址：

```sh
# ifconfig wlan1 inet 192.168.2.1/24 up
```

动态地址分配需要安装软件包 dhcpd 并创建配置文件。可以在 `dhcpd.conf(5)` 手册页中找到最简配置示例，示例如下：

```ini
/usr/local/etc/dhcpd.conf:

subnet 192.168.2.0 netmask 255.255.255.0 {
    range 192.168.2.100 192.168.2.200
}
```

然后启用并启动 dhcpd 服务：

```sh
# service enable dhcpd
# service start dhcpd
```

### 监控模式（Monitor mode）

我们示例中创建的最后一个 VAP 是监控模式。虽然其他模式下也能抓包，但接口并非混杂模式，只会收到发给该接口地址的数据包。监控模式允许我们接收信道上所有的数据包（具体取决于硬件对不同速率的支持）。

一个简单的验证方法是使用带有 `-y` 选项（设置链路层头部类型）的 `tcpdump`：

```sh
# tcpdump -L -i wlan2
Data link types for wlan0 (use option -y to set):
  EN10MB (Ethernet)
  IEEE802_11_RADIO (802.11 plus radiotap header)
```

我经常记不清这个变量中下划线的顺序，但 `tcpdump` 使用 `-L` 参数加接口名可以显示该接口类型支持的链路层头部类型。

```sh
# sudo tcpdump -i wlan2 -y IEEE802_11_RADIO
tcpdump: data link type IEEE802_11_RADIO
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlan2, link-type IEEE802_11_RADIO (802.11 plus radiotap header), snapshot length 262144 bytes
14:33:48.656430 3757399us tsft 1.0 Mb/s 2412 MHz 11g -76dBm signal -95dBm noise Data IV:c1ed Pad 20 KeyID 1
14:33:50.657087 5759270us tsft 1.0 Mb/s 2412 MHz 11g -72dBm signal -95dBm noise
14:33:50.796280 5895802us tsft 1.0 Mb/s 2412 MHz 11g -76dBm signal -95dBm noise Beacon (HomeWifi) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 2, PRIVACY                            
14:33:53.151514 8251009us tsft 1.0 Mb/s 2412 MHz 11g -74dBm signal -95dBm noise Beacon (HomeWifi) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 2, PRIVACY
14:33:53.970729 9070213us tsft 1.0 Mb/s 2412 MHz 11g -74dBm signal -95dBm noise Beacon (HomeWifi) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 2, PRIVACY
14:34:10.183336 25285260us tsft 6.0 Mb/s 2437 MHz 11g -62dBm signal -95dBm noise Beacon (a2-enc) [6.0* 9.0 12.0* 18.0 24.0* 36.0 48.0 54.0 Mbit] ESS CH: 6, PRIVACY
14:34:10.204045 25306099us tsft 11.0 Mb/s 2437 MHz 11g -68dBm signal -95dBm noise Beacon () [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 6                                            
14:34:10.253438 25356305us tsft 11.0 Mb/s 2437 MHz 11g -58dBm signal -95dBm noise Beacon () [1.0* 2.0* 5.5* 11.0* 6.0* 9.0 12.0* 18.0 Mbit] IBSS CH: 6, PRIVACY
14:34:10.253441 25356305us tsft 11.0 Mb/s 2437 MHz 11g -58dBm signal -95dBm noise Beacon () [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 6
14:34:10.253445 25356305us tsft 11.0 Mb/s 2437 MHz 11g -58dBm signal -95dBm noise Beacon (HM-CM-$tte, HM-CM-$tte, kette) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 6, PRIVACY
14:34:10.285355 25387273us tsft 6.0 Mb/s 2437 MHz 11g -63dBm signal -95dBm noise Beacon (a2-enc) [6.0* 9.0 12.0* 18.0 24.0* 36.0 48.0 54.0 Mbit] ESS CH: 6, PRIVACY
14:34:10.297973 25399988us tsft 11.0 Mb/s 2437 MHz 11g -68dBm signal -95dBm noise Beacon (HM-CM-$tte, HM-CM-$tte, adkette) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 6, PRIVACY
14:34:10.355834 25458704us tsft 11.0 Mb/s 2437 MHz 11g -58dBm signal -95dBm noise Beacon () [1.0* 2.0* 5.5* 11.0* 6.0* 9.0 12.0* 18.0 Mbit] IBSS CH: 6, PRIVACY
14:34:10.355836 25458704us tsft 11.0 Mb/s 2437 MHz 11g -58dBm signal -95dBm noise Beacon () [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 6
```

这个 `tcpdump` 输出聚焦于 IEEE80211 无线电帧。深入分析则需要使用其他工具，比如 Wireshark。

## 测试流量

现在我们已经拥有了构建纯 FreeBSD AP 站点并调试无线流量的所有条件，接下来讨论如何进行测试。

我们可以利用 AP 站点和监控模式来验证和调查关联过程中以及数据发送时的无线包。

测试的第一步是让站点成功连接到网络，能向 AP 发送 `ping`。通过站点上的 `wpa_supplicant`，我们可以选择网络并请求其完成关联。`wpa_supplicant` 会打印关联过程中的消息，记录握手包。

连接成功后，站点需要获取 IP 地址。如果没有自动完成，可以运行：

```sh
# dclient wlan0
```

通常这会让它正常工作。拿到 dhcpd 分配的地址后，下一步测试能否 ping 通 AP：

```sh
# ping 192.168.2.1
```

如果失败，就是排查的开始。很可能是配置问题（示例不一定完美，但经过测试）。调试无法连接时的原因往往不是最愉快的 WiFi 开发体验，但这是我们都遇到过的。

拿到地址后，简单的吞吐量测试通常是第一步的好指标。如果测试的是某个分支上的补丁，只要编译了带补丁的内核，跑网络吞吐量测试就足够了。我喜欢用 `iperf3`，它可以测试到你网络内某主机或互联网的吞吐量。用 `iperf` 在站点和 AP 端互测，可以直观了解该硬件组合可能达到的吞吐率。

如果系统够稳定，可以运行浏览器，我觉得用 <fast.com> 测试也很有参考价值。

这两种测试分别反映了不同的限制：`iperf` 测试给出 WiFi 网络内站点的吞吐能力，<fast.com> 测试显示从你网络到互联网的速率。<fast.com> 的数值可能明显更低。如果快于 `iperf`，那就不太正常。

TCP 和 UDP 测试的吞吐量会有差异。UDP 更能饱和 WiFi 无线电。两者都测试效果好，但若要快速评估，TCP 吞吐量即可。

## 准备开始工作

本文介绍了在 FreeBSD 上开始进行 WiFi 开发所需的背景知识和术语，但还未涉及任何代码。使用这里的示例搭建测试网络是 WiFi 开发的核心部分，如果你迫不及待，不想等待第二部分，我相信只要配备足够硬件进行测试，你就能开始发现问题。

下一篇文章中，我们将探讨 WiFi 驱动的生命周期，它需要具备的核心功能，以及与 net80211 交互的接口，这些接口能够承担驱动的大量工作。

**Tom Jones** 是一位 FreeBSD 提交者，致力于保持网络栈的高速性能。
