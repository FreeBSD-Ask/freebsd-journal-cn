# FreeBSD WiFi 开发第三部分：调试

- 原文：[Getting started with WiFi development Part 3: Debugging](https://freebsdfoundation.org/our-work/journal/browser-based-edition/improving-software-quality/getting-started-with-wifi-development-part-3-debugging/)
- 作者：**Tom Jones**

## FreeBSD WiFi 栈经得起追问。

这套 WiFi 开发入门系列里，调试这部分原本完全可以充当第一、第二、第三节，那样我们也许永远都不会去看驱动代码。

WiFi 具备一切让调试变难的特征：它是并发分布式状态机；时间、物理与随机效应会改变它的行为；它揉进了微波射频设计的黑魔法；硬件专有、没有文档，用起来还反直觉。

话虽如此，FreeBSD 的 WiFi 性能很高，过去二十年里许多人为驱动和协议栈支持出过力，大多是利用业余时间。

说到底，人们愿意做 WiFi，是因为一旦有了进展，许多调试环节会让人很有成就感。

本文介绍在 FreeBSD 上做驱动开发时调试 WiFi 的两种途径：被动抓包和系统日志。

抓包让你从外部观察系统。你可以在第三台计算机上抓包，了解你的 AP 与工作站之间发生了什么（或者没发生什么）。它能给出独立于驱动影响的事实依据，说明哪些数据真正穿过了网络。

系统日志让我们看清 FreeBSD 依据收到的包认为正在发生什么。包和事件会在许多地方记入日志，我们由此看出流量如何引起状态机变化，也能看到那些在空口上通过、一进 FreeBSD 机器就消失的流量去了哪里。

### 初步调试

先从针对常见问题的调试说起。

我想多数人调试 WiFi 时，想知道的是自己的工作站为什么连不上网络。多数情况下，连不上网的原因很容易找到，事后往往觉得是个愚蠢的错误。

如果你把 `wpa` 网络的 PSK 填错，wpa_supplicant 会提示问题就在这里：

```sh
<3>Trying to associate with 94:83:c4:58:ed:32 (SSID='o2-enc' freq=2437 MHz)
<3>Associated with 94:83:c4:58:ed:32
<3>CTRL-EVENT-DISCONNECTED bssid=94:83:c4:58:ed:32 reason=0
<3>WPA: 4-Way Handshake failed - pre-shared key may be incorrect
<3>CTRL-EVENT-SSID-TEMP-DISABLED id=8 ssid="o2-enc" auth_failures=1 duration=10 reason=WRONG_KEY
<3>Added BSSID 94:83:c4:58:ed:32 into ignore list, ignoring for 10 seconds
<3>CTRL-EVENT-SCAN-RESULTS
```

这个例子是 wpa_cli 工具的输出，它让你直接对接 wpa_supplicant 的日志等内容。

输错密钥任何工具都救不了，但生成正确的条目可以用 wpa_passphrase 完成。

```sh
$ wpa_passphrase "example network" "excellentpsk"
network={
   ssid="example network"
   #psk="excellentpsk"
   psk=27231323cedae4b8036557f8dcd5f38bfb6e0ac9a5ab894a4ee5104f7998fc93
}
```

尝试加入网络时的其他问题，可能出在距离，或者网卡收不到那些宣告网络存在的 beacon。过去只有一台设备时，这很难查清。但现在我们几乎总还有另一台 WiFi 设备，可以验证网络在这里确实能用，这就说明 FreeBSD 设备上的配置有问题。

第二类想调试的问题是性能。要么网络覆盖不到你想要的位置，要么配置有问题，导致性能比你参照同一网络中其他系统所预期的更低。

这要难查得多，能帮上忙的人也更少。你大概不知道墙里面有什么，互联网上的陌生人也猜不准到底出了什么问题。不过他们可以帮你看症状。

好在拖慢性能的不只是玄学的射频问题，有些配置和排查能帮你提高网速。想弄清究竟能看见哪些网络时，wpa_cli 是很好的起点：

```sh
> scan_results
bssid / frequency / signal level / flags / ssid
c8:e3:06:4d:1d:83       2437    -81     [RSN-SAE-CCMP][MESH]    19c14
c8:e3:06:5d:3d:48       2437    -81     [WPA2-PSK-CCMP][ESS]    Computer Network
18:e8:b9:c2:18:1a       2412    -87     [WPA2-PSK+SAE-CCMP][ESS]        HomeWifi
a8:e3:06:5d:fd:47       2437    -81     [ESS]
20:12:34:c6:18:bd       2412    -87     [ESS]   Guest-Wifi
```

`scan_results` 命令显示 wpa_supplicant 当前已知的网络。它通过 net80211 子系统从绑定的驱动获取这些信息。如果你想连的网络在这里看不到，那可能就是运气不好。可以的话，试着靠近 AP，再慢慢走远，直到信号消失。

如果你在追查性能问题，第一步应该看接收信号强度指示（RSSI）。信号电平太低，就说明设备离接入点不够近。信号电平以 dBm 表示，这个值总是负数。作为相对度量，越接近 1 强度越高。刚好能检测到的极弱信号约为 ~87dBm，强信号可能约为 50dBm。许多协议特性要求信号强度达到某一电平才能使用。

另一种可能限制性能的配置问题非常难调。WiFi 使用的无线电频谱分布在多个频段上。有些地方这些频段完全开放给 WiFi 设备使用，而在世界其他地区，WiFi 可能只是频谱的次级用户，部分频段因此被挡住无法使用。还有些地区必须在 5GHz 频段检测气象雷达。

各个频段与它们的可用情况合称[监管域](https://wiki.freebsd.org/WiFi/RegulatoryDomainSupport)。如果工作站的监管域与接入点不一致，你可能会发现设备只能用较旧的 WiFi 标准。

不一致可能出于许多原因。如果你用的是 ISP 提供的无线路由器，那它*应该*已经针对你所在地正确配置。但你的设备可能来自不清楚它会被卖到哪个监管域的厂商，也可能你随身带着旅行路由器。这种情况下，你应当核实工作站、接入点、邻近接入点的监管域是否一致。没错，别人的接入点确实可能配错，或者配置不同，而你网卡里的固件就据此认定自己身处世界上的另一个地区。

你可以用 ifconfig 查看工作站的监管域：

```sh
$ ifconfig wlan0
wlan0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
             options=0
             ether 74:da:38:33:c0:62
             groups: wlan
             ssid "" channel 10 (2457 MHz 11g)
             regdomain FCC country US authmode WPA1+WPA2/802.11i privacy MIXED
             deftxkey UNDEF txpower 30 bmiss 7 scanvalid 60 protmode CTS wme
             roaming MANUAL bintval 0
             parent interface: rtwn0
             media: IEEE 802.11 Wireless Ethernet autoselect (autoselect)
             status: no carrier
             nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
```

默认情况下，FreeBSD 把我基本没配置过的 `rtwn` 设备归入 `US` 监管域：

```sh
regdomain FCC country US authmode WPA1+WPA2/802.11i privacy MIXED
```

ifconfig 还能用来查看它看得见的接入点的监管域，不过输出相当难读：

```sh
$ ifconfig -v wlan0 scan results
SSID/MESH ID                      BSSID              CHAN RATE    S:N     INT CAPS
HomeWifi                          18:e8:b9:c2:18:1a    1   54M  -87:-95   100 EPS  SSID<HomeWifi> RATES<B2,B4,B11,B22,12,18,24,36> DSPARMS<1> COUNTRY<GB  1-13,20> ERP<0x0> RSN<v1 mc:AES-CCMP uc:AES-CCMP km:8021X-PSK+?> XRATES<48,72,96,108> BSSLOAD<sta count 3, chan load 56, aac 18> RRM_ENCAPS<460573d000000c> HTCAP<cap 0x1ac param 0x1b mcsset[0-15] extcap 0x0 txbf 0x0 antenna 0x0> HTINFO<ctl 1, 8,4,0,0 basicmcs[]> OVERLAP_BSS<4a0e14000a002c01c8-> EXTCAP<7f080500000200000040> WME<qosinfo 0x0 BE[aifsn 3 cwmin 4 cwmax 10 txop 0] BK[aifsn 7 cwmin 4 cwmax 10 txop 0] VO[aifsn 2 cwmin 3 cwmax 4 txop 94] VI[aifsn 2 cwmin 2 cwmax 3 txop 47]> ATH<0x7fff> VEN<dd3900156d00010100010217e5810618e8-> VEN<dd168cfdf0040000490000030209720100-> VEN<dd088cfdf00101020100>
```

扫描结果末尾 caps 字段中的 `COUNTRY` 元素显示接入点认为自己在哪里：

```sh
COUNTRY<GB  1    -13,20>
```

解决办法是用 ifconfig 指定监管域和国家：

```sh
$ ifconfig wlan0 list countries
Country codes:
DEBUG Debug        ZW Zimbabwe        YE Yemen           VN Viet Nam      
VE Venezuela       UZ Uzbekistan      UY Uruguay         US United States  
GB United Kingdom  AE United Arab Emi UA Ukraine         TR Turkey        
TN Tunisia         TT Tobago          TH Thailand        TW Taiwan        
SY Syria           CH Switzerland     SE Sweden          LK Sri Lanka      
ES Spain           ZA South Africa    SI Slovenia        SK Slovak Republic
SG Singapore       SA Saudi Arabia    RU Russia          RO Romania        
QA Quatar          PR Puerto Rico     PT Portugal        PL Poland        
PH Phillipines     PE Peru            PA Panama          PK Pakistan      
OM Oman            NO Norway          NZ New Zealand     NL Netherlands    
NP Nepal           MA Morocco         MC Monaco          MX Mexico        
MT Malta           MY Malaysia        MK Macedonia       MO Macau          
LU Luxemborg       LT Lithuania       LI Liechtenstein   LB Lebanon        
LV Latvia          KW Kuwait          K2 Korea Republic2 KR Korea Republic
KP North Korea     KZ Kazakhstan      JO Jordan          J5 Japan5        
J4 Japan4          J3 Japan3          J2 Japan2          J1 Japan1        
JP Japan           JM Jamaica         IT Italy           IL Israel        
IE Ireland         IR Iran            ID Indonesia       IN India          
IS Iceland         HU Hungary         HK Hong Kong       HN Honduras      
GT Guatemala       GR Greece          DE Germany         GE Georgia        
F2 France2         FR France          FI Finland         EE Estonia        
SV El Salvador     EG Egypt           EC Ecuador         DO Dominican Repub
DK Denmark         CZ Czech Republic  CY Cyprus          HR Croatia        
CR Costa Rica      CO Colombia        CN China           CL Chile          
CA Canada          BG Bulgaria        BN Brunei          BR Brazil        
BO Bolivia         BZ Belize          BE Belgium         BY Belarus        
BD Bangladesh      BH Bahrain         AZ Azerbaijan      AT Austria        
AU Australia       AM Armenia         AR Argentina       DZ Algeria        
AL Albania        
Regulatory domains:
XC900M          GZ901           XR9             SR9            
NONE            ROW             TAIWAN          KOREA          
APAC3           APAC2           APAC            ETSI3          
ETSI2           ETSI            JAPAN           FCC4          
FCC3            FCC             DEBUG          
$ sudo ifconfig wlan0 down
$ sudo ifconfig wlan0 country GB regdomain ETSI
```

### 抓包

想了解空口和 WiFi 接口上的流量时，抓取数据包（即 pcap）是很有用的调试步骤。FreeBSD 中的 wlan 接口能在两个层次生成 pcap：一是作为逻辑以太网设备，类似你用 em 这样的有线接口；二是在链路层，让你访问 80211 包类型。把接口配置成工作站时得到的是第一种，而监视模式的 wlan 接口会给出更多底层链路的信息。

本系列[第一部分](https://freebsdfoundation.org/wp-content/uploads/2025/07/jones_wifi.pdf)里，我们为示例硬件创建了三种不同的 VAP（虚拟接入点）：工作站、host AP 和监视模式。

监视模式 VAP 让底层硬件进入混杂模式，收到的所有包都连着链路头部一起交给 bpf。这样既有更多链路信息，也能拿到某个信道上能听到的全部包，而不只是发往本设备或广播地址的包。

在驱动里，所有包都可以带上接收时的无线电环境信息，这些信息放在名为 radiotap 头部的字段中。

radiotap 头部给出有用的元数据，说明收包时 WiFi 射频的状态，比如接收强度（RSSI）、WiFi 信道和调制方式。这些字段对理解某件事为什么能工作（更多时候是为什么不能工作）至关重要。

监视模式下，我们能拿到射频能收到的所有包。仅此一点就能带来大量线索，说明本地射频环境里正在发生什么。

#### tcpdump

用抓包调试 WiFi 时有两个工具可用，其中 tcpdump 是以最小开销获取信息的关键。tcpdump 是标准抓包工具，随 FreeBSD 基本系统一起发布。

通常 tcpdump 在 IP 层显示包信息，只做少量内省，看看包在做什么。要充分利用 WiFi 接口，我们需要让 tcpdump 包含链路层头部，具体来说是 `IEEE802_11_RADIO`。

```sh
# ifconfig wlan create wlandev rtwn0 wlanmode monitor
```

加上 WiFi 头部以后，我们不再能看到流量内容，但会得到大量描述射频环境的附加字段和 ieee80211 MAC 层的信息。

```sh
$ sudo tcpdump -i wlan0 -y IEEE802_11_RADIO
tcpdump: data link type IEEE802_11_RADIO
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlan0, link-type IEEE802_11_RADIO (802.11 plus radiotap header), snapshot length 262144 bytes
10:27:07.724396 74143272us tsft 1.0 Mb/s 2437 MHz 11g -32dBm signal -95dBm noise Beacon (o2-enc) [1.0* 2.0* 5.5* 11.0* 9.0 18.0 36.0 54.0 Mbit] ESS CH: 6, PRIVACY
10:27:07.938361 74362602us tsft 6.0 Mb/s 2437 MHz 11g -58dBm signal -95dBm noise Clear-To-Send RA:e0:3e:44:08:34:e1 (oui Unknown)
10:27:07.953755 74375625us tsft 6.0 Mb/s 2437 MHz 11g -58dBm signal -95dBm noise Clear-To-Send RA:e0:3e:44:08:34:e1 (oui Unknown)
10:27:07.958151 74382377us tsft 11.0 Mb/s 2437 MHz 11g -68dBm signal -95dBm noise Beacon () [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 6
10:27:07.973090 74394982us tsft 6.0 Mb/s 2437 MHz 11g -58dBm signal -95dBm noise Clear-To-Send RA:e0:3e:44:08:34:e1 (oui Unknown)
10:27:08.003137 74424986us tsft 6.0 Mb/s 2437 MHz 11g -58dBm signal -95dBm noise Clear-To-Send RA:e0:3e:44:08:34:e1 (oui Unknown)
10:27:08.007497 74432483us tsft 6.0 Mb/s 2437 MHz 11g -59dBm signal -95dBm noise Clear-To-Send RA:e0:3e:44:08:34:e1 (oui Unknown)
10:27:08.033133 74454986us tsft 6.0 Mb/s 2437 MHz 11g -61dBm signal -95dBm noise Clear-To-Send RA:e0:3e:44:08:34:e1 (oui Unknown)
10:27:08.037498 74462485us tsft 6.0 Mb/s 2437 MHz 11g -59dBm signal -95dBm noise Clear-To-Send RA:e0:3e:44:08:34:e1 (oui Unknown)
10:27:08.063204 74486734us tsft 11.0 Mb/s 2437 MHz 11g -68dBm signal -95dBm noise Data IV:d43 Pad 20 KeyID 1
10:27:08.063206 74486734us tsft 11.0 Mb/s 2437 MHz 11g -68dBm signal -95dBm noise Data IV:d44 Pad 20 KeyID 1
10:27:08.063211 74486734us tsft 11.0 Mb/s 2437 MHz 11g -68dBm signal -95dBm noise Data IV:d45 Pad 20 KeyID 1
10:27:08.063211 74486734us tsft 11.0 Mb/s 2437 MHz 11g -68dBm signal -95dBm noise Data IV:d46 Pad 20 KeyID 1
```

运行上面的命令时，你很可能会被本地工作站发来的管理（mgmt）流量淹没。要继续下去，得回想一下前几篇文章讨论过的 WiFi 会话建立过程。

在[上一篇文章](https://freebsdfoundation.org/our-work/journal/browser-based-edition/embedded-2/freebsd-wifi-development-part-2-working-on-a-driver/)里，我们谈到工作站如何加入网络，概括起来就是：

- 探测网络（probe request、response）
- 认证（auth request、response）
- 关联（assoc request、response）

每个阶段都是两种不同的 mgmt 帧：请求和响应。不过淹没你的多半根本不是这些帧！

为了加快入网过程，并给客户端一份可加入网络的清单，所有接入点都会不停地 “beacon” 自己的存在。在非常繁忙的环境里，beacon 可能耗掉大量可用空口时间，那份长长的可用 WiFi 网络列表其实不是什么好事。

beacon 帧携带大量描述网络和自身能力的信息，我们可以在 tcpdump 抓包中过滤出只显示 beacon 帧，像这样：

```sh
10:27:07.724396 74143272us tsft 1.0 Mb/s 2437 MHz 11g -32dBm signal -95dBm noise Beacon (o2-enc) [1.0* 2.0* 5.5* 11.0* 9.0 18.0 36.0 54.0 Mbit] ESS CH: 6, PRIVACY
```

beacon 里的信息远多于 tcpdump 单独能显示的内容，要拿到 beacon 承载的更多数据，得改用 wireshark，本例中则是它的命令行界面 tshark。

首先用 tcpdump 导出 pcap：

```sh
$ sudo tcpdump -i wlan0 -y IEEE802_11_RADIO -w testcaputre.pcap -c 1000
```

把抓到的文件交给 tshark，让它显示之前看到的测试网络的第一个 beacon 包：

```sh
$ tshark -V -r testcaputre.pcap -a "packets: 1" 'wlan.ssid == "o2-enc"'
```

对于这个 393 字节的 beacon 帧，它会输出近 900 行包描述，下面是一小段，显示 beacon 帧和部分射频信息。

```sh
802.11 radio information
      PHY type: 802.11b (HR/DSSS) (4)
      Short preamble: False
      Data rate: 11.0 Mb/s
      Channel: 6
      Frequency: 2437MHz
      Signal strength (dBm): -66 dBm
      Noise level (dBm): -95 dBm
      Signal/noise ratio (dB): 29 dB
      TSF timestamp: 1853402918
      [Duration: 461µs]
             [Preamble: 192µs]
             [IFS: 15572µs]
             [Start: 1853402457µs]
             [End: 1853402918µs]
IEEE 802.11 Beacon frame, Flags: ........
      Type/Subtype: Beacon frame (0x0008)
      Frame Control Field: 0x8000
             .... ..00 = Version: 0
             .... 00.. = Type: Management frame (0)
             1000 .... = Subtype: 8
             Flags: 0x00
                    .... ..00 = DS status: Not leaving DS or network is operating in AD-HOC mode (To DS: 0 From DS: 0) (0x0)
                    .... .0.. = More Fragments: This is the last fragment
                    .... 0... = Retry: Frame is not being retransmitted
                    ...0 .... = PWR MGT: STA will stay up
                    ..0. .... = More Data: No data buffered
                    .0.. .... = Protected flag: Data is not protected
                    0... .... = +HTC/Order flag: Not strictly ordered
      .000 0000 0000 0000 = Duration: 0 microseconds
      Receiver address: Broadcast (ff:ff:ff:ff:ff:ff)
             .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
             .... ...1 .... .... .... .... = IG bit: Group address (multicast/broadcast)
      Destination address: Broadcast (ff:ff:ff:ff:ff:ff)
             .... ..1. .... .... .... .... = LG bit: Locally administered address (this is NOT the factory default)
             .... ...1 .... .... .... .... = IG bit: Group address (multicast/broadcast)
      Transmitter address: GLTechnologi_58:ed:32 (94:83:c4:58:ed:32)
             .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
             .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
      Source address: GLTechnologi_58:ed:32 (94:83:c4:58:ed:32)
             .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
             .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
      BSS Id: GLTechnologi_58:ed:32 (94:83:c4:58:ed:32)
             .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
             .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
      .... .... .... 0000 = Fragment number: 0
      1010 0010 1011 .... = Sequence number: 2603
      [WLAN Flags: ........]
```

### 系统日志

调试 WiFi 问题的第三个信息来源是系统工具和日志。多数时候系统刻意保持安静，它打印的消息通常只在出现错误这类异常情况时才会出现。开发者喜欢内置实时调试的选项，我们可以提高调试级别，让内核消息缓冲区装满 WiFi 栈的大量有用信息。

#### wlandebug

```sh
/usr/sbin/wlandebug
```

FreeBSD 自带 **wlandebug(8)** 工具，它提供了更友好的界面，用来配置 net80211 栈输出信息的详细程度。

```sh
# wlandebug -i wlan1 scan+auth+assoc
```

消息直接打印到内核缓冲区，想实时跟踪事件时会觉得很烦。这些新打印的消息可以用 dmesg 显示。

```sh
$ dmesg
wlan0: ieee80211_scanreq: vap 0xfffff80139506000 iv_state 0x1 (SCAN) flags 0x20052 duration 0x7fffffff mindwell 0 maxdwell 0 nssid 1
wlan0: ieee80211_check_scan: active scan, append, nojoin, once
wlan0: ieee80211_swscan_start_scan_locked: active scan, duration 2147483647 mindwell 0 maxdwell 0, desired mode auto, append, nojoin, once
wlan0: scan set
1g, 6g, 11g, 7g, 2g, 3g, 4g, 5g, 8g, 9g, 10g                              
wlan0:  dwell min 20ms max 200ms                                          
wlan0: scan_curchan_task: loop start; scandone=0, scanstop=0, ss_iflags=0x2, ss_next=0, ss_last=11
wlan0: scan_curchan_task: chan   6n ->   1g [active, dwell min 20ms max 200ms]
wlan0: scan_curchan: calling; maxdwell=200                                
wlan0: scan_curchan_task: waiting                
[22:e8:29:c7:48:1a] new probe_resp on chan 1 (bss chan 1) "Guest-Wifi" rssi 15
[22:e8:29:c7:48:1a] caps 0x1421 bintval 100 erp 0x100 country [GB  1-13,20]
wlan0: ieee80211_swscan_add_scan: chan   1g min dwell met (2167162969 > 18446744071581747289)
wlan0: scan_mindwell: called
wlan0: scan_curchan_task: loop start; scandone=0, scanstop=0, ss_iflags=0x21, ss_next=1, ss_last=11
wlan0: scan_curchan_task: chan   1g ->   6g [active, dwell min 20ms max 200ms]
wlan0: scan_curchan: calling; maxdwell=200
wlan0: scan_curchan_task: waiting        
[c8:e3:06:5d:fd:45] new probe_resp on chan 6 (bss chan 6) 0x48c3a47474652c2048c3a47474652c204661687261646b65747465 rssi 29
[c8:e3:06:5d:fd:45] caps 0x1431 bintval 100 erp 0x100 country [GBI 1-13,20]
[c8:e3:06:5d:fd:45] new probe_resp on chan 6 (bss chan 6) 0x48c3a47474652c2048c3a47474652c204661687261646b65747465 rssi 29
[c8:e3:06:5d:fd:45] caps 0x1431 bintval 100 erp 0x100 country [GBI 1-13,20]
[c8:e3:06:5d:fd:45] new probe_resp on chan 6 (bss chan 6) 0x48c3a47474652c2048c3a47474652c204661687261646b65747465 rssi 29
[c8:e3:06:5d:fd:45] caps 0x1431 bintval 100 erp 0x100 country [GBI 1-13,20]
wlan0: ieee80211_swscan_add_scan: chan   6g min dwell met (2167162992 > 18446744071581747312)
wlan0: scan_mindwell: called
[c8:e3:06:5d:fd:45] new probe_resp on chan 6 (bss chan 6) 0x48c3a47474652c2048c3a47474652c204661687261646b65747465
wlan0: scan_curchan_task: loop start; scandone=0, scanstop=0, ss_iflags=0x21, ss_next=2, ss_last=11 rssi 29
wlan0: scan_curchan_task: chan   6g ->  11g [active, dwell min 20ms max 200ms]
[c8:e3:06:5d:fd:45] caps 0x1431 bintval 100 erp 0x100 country [GBI 1-13,20]
wlan0: scan_curchan: calling; maxdwell=200
wlan0: scan_curchan_task: waiting
wlan0: scan_curchan_task: loop start; scandone=0, scanstop=0, ss_iflags=0x20, ss_next=3, ss_last=11
wlan0: scan_curchan_task: chan  11g ->   7g [active, dwell min 20ms max 200ms]
wlan0: scan_curchan: calling; maxdwell=200                                
wlan0: scan_curchan_task: waiting                                        
```

这些消息由每个设备的 sysctl 掩码控制。我们可以手动设置，但 wlandebug 提供了更顺手的界面。配置好之后，系统日志里会出现额外的日志消息，可以用 dmesg 命令取出。

每条打印出来的消息都出自 `IEEE80211_DPRINTF` 内核宏。

示例中最后那些消息来自 `scan_curchan`，在源码里它们长这样：

```sh
IEEE80211_DPRINTF(vap, IEEE80211_MSG_SCAN,
      "%s: calling; maxdwell=%lu\n",
      __func__,
      maxdwell);

IEEE80211_DPRINTF(ss->ss_vap, IEEE80211_MSG_SCAN,
      "%s: loop start; scandone=%d, scanstop=%d, ss_iflags=0x%x, ss_next=%u, ss_last=%u\n",
      __func__,
      scandone,
      scanstop,
      (uint32_t) ss_priv->ss_iflags,
      (uint32_t) ss->ss_next,
      (uint32_t) ss->ss_last);
```

借助 wlandebug，你可以用上内核里内建的调试语句，既不必重新构建内核，也不会一直被输出消息淹没。

#### wlanstat

它通过 `SIOCG80211STATS` ioctl 收集统计信息。在 net80211 内部，这些数据存放在 vap 的 `iv_stats` 成员中，该成员是 `ieee80211_stats` 的实例，这个约 150 个成员的结构体记录 wlan 栈里发生过什么。

好在 wlanstat 打印时很聪明，只显示含有相关信息的统计：

```sh
$ wlanstat                                                  
1821              rx from wrong bssid
2                 rx discard mgt frames
2127              rx beacon frames
17629             rx element unknown
53                rx frame chan mismatch
5                 active scans started
18                ccmp crypto done in s/w
2221              rx management frames
2                 rx action frames
59                A-MSDU frames received
3                 A-MPDU frames discarded for out of range seqno
5034              total data frames received
5016              unicast data frames received
18                multicast data frames received
6044              total data frames transmit
6044              unicast data frames sent
58.5M             current transmit rate
31.5              current rssi
-95               current noise floor (dBm)
-63.5             current signal (dBm)
```

从 wlanstat 我们能得到一些历史信息和当前信息。输出末尾的当前信息显示信号强度、本底噪声和当前传输速率。它们会随你的移动和最近发送流量的多少而变化。

历史信息告诉我们见过哪些类型的帧。调试时一项有用的统计是收到的 `A-MSDU` 帧数量。这个数字偏低，往往是无法达到更高吞吐率的重要线索。

#### ifconfig

ifconfig 能告诉你 wlan 接口配置的大量信息。

```sh
$ ifconfig -v wlan0
wlan0: flags=8802<BROADCAST,SIMPLEX,MULTICAST> metric 0 mtu 1500
      options=0
      ether 74:da:38:33:c0:62
      groups: wlan
      ssid "" channel 1 (2412 MHz 11b) bssid 00:00:00:00:00:00
      regdomain FCC country US anywhere -ecm authmode OPEN -wps -tsn
      privacy OFF deftxkey UNDEF
      powersavemode OFF powersavesleep 100 txpower 30 txpowmax 50.0 -dotd
      rtsthreshold 2346 fragthreshold 2346 bmiss 7
      11b     ucast NONE    mgmt  1 Mb/s mcast  1 Mb/s maxretry 6
      11g     ucast NONE    mgmt  1 Mb/s mcast  1 Mb/s maxretry 6
      11ng    ucast NONE    mgmt  1 Mb/s mcast  1 Mb/s maxretry 6
      scanvalid 60 -bgscan bgscanintvl 300 bgscanidle 250
      roam:11b     rssi    7dBm rate  1 Mb/s
      roam:11g     rssi    7dBm rate  5 Mb/s
      roam:11ng    rssi    7dBm  MCS  1    
      -pureg protmode CTS ht20 htcompat ampdu ampdulimit 64k
      ampdudensity 16 amsdu shortgi htprotmode RTSCTS -puren -smps -rifs
      -stbc -ldpc -uapsd -vht wme -burst -dwds roaming AUTO bintval 0
      AC_BE cwmin  0 cwmax  0 aifs  0 txopLimit   0 -acm ack
              cwmin  0 cwmax  0 aifs  0 txopLimit   0 -acm
      AC_BK cwmin  0 cwmax  0 aifs  0 txopLimit   0 -acm ack
              cwmin  0 cwmax  0 aifs  0 txopLimit   0 -acm
      AC_VI cwmin  0 cwmax  0 aifs  0 txopLimit   0 -acm ack
              cwmin  0 cwmax  0 aifs  0 txopLimit   0 -acm
      AC_VO cwmin  0 cwmax  0 aifs  0 txopLimit   0 -acm ack
              cwmin  0 cwmax  0 aifs  0 txopLimit   0 -acm
      parent interface: rtwn0
      media: IEEE 802.11 Wireless Ethernet autoselect (autoselect)
      status: no carrier
      nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
      drivername: wlan0
```

ifconfig 的详细输出给出大量实用信息，说明 wlan 接口如何配置、连在哪个网络与信道（前提是它连着网络；本例中什么都没连，所以 ssid 是空的）。

ifconfig 输出告诉我们可调的参数，例如发射功率、国家和监管域，也包含从所连网络获取的参数。`AC_BE`、`AC_BK`、`AC_VI`、`AC_VO` 描述网络在管理帧中通告的接入类别参数。

### 运用工具

FreeBSD WiFi 栈能提供大量信息帮你调试问题，但我认为对多数人来说，现实是你很快就会淹没在技术细节里。多数情况下，从较高层的工具入手、再逐层向下，能更快得到有价值的结果。如果连不上网络，运行 wpa_cli 并留意它报告的消息，会让你处于最有利的位置。只要底层设备配置正确、接入点在范围内，你的问题多半能靠 wpa_supplicant 的建议解决。

如果你怀疑问题出在 net80211 层或设备驱动，下一步就该双管齐下：用抓包验证空口上的流量，并核实 net80211 的行为。这需要更多对 WiFi 工作原理的理解，但这里往往能找到问题所在的好线索。

人很容易自己吓自己，把某个字段看成并不存在的错误。从本文和整个系列里，我最想让你记住的一点是：FreeBSD WiFi 栈多么经得起追问。想调试某个问题，联系邮件列表是很好的一步。对方可能会请你提供调试信息，比如本文讨论的某些内容。

这没什么可怕的。启用大多数可用日志机制都很简单，而且只要重启，它们全都会恢复为关闭。

WiFi 能达到的速度和性能令人难以置信，从 1Mbit/s 的链路一路发展到超过千兆，而且多数时候结果相当稳定。我们的 WiFi 栈一直在进步，随着用户和测试增多，它还会更好。

---

**Tom Jones** 是 FreeBSD 提交者，关注如何让网络栈保持高速。
