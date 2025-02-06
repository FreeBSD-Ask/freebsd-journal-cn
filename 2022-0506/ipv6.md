# 实用 IPv6（第一部分）

- 原文链接：[Pragmatic IPv6 (Part 1)](https://freebsdfoundation.org/wp-content/uploads/2022/06/hiroki_IPv6.pdf)
- 作者：**佐藤広生**

你是否曾在你的 FreeBSD 系统上尝试使用过 IPv6？虽然 IPv4 仍然是当今互联网世界的黄金标准，但 IPv6 被视为一种有前景的协议，预计最终将取代 IPv4。IPv6 的第一版 RFC 于 1995 年发布，FreeBSD 默认支持 IPv6 已 20 余年了。其实现已经成熟，不再是实验性的。

为什么 IPv6 不那么流行？一些人认为 IPv4 已经足够满足他们的需求，不想进行更改。这是个合理的理由。然而，我认为阻止 IPv6 普及的原因之一是系统管理员缺乏经验。与 IPv4 的历史类似，IPv6 的实现和最佳实践也随着时间的推移发生了变化。这使得找到包含 IPv6 实用信息的书籍、文章/文献变得困难。只要使用 IPv6 的利弊尚不明确，没有经验的系统管理员是不会愿意使用这一新技术的。

这系列文章将通过展示你可以在 FreeBSD 系统上尝试的各种例子（不仅仅是理论方面），向你介绍（或重新介绍）IPv6。IPv6 并不是对 IPv4 的完全替代，它与现有的 IPv4 网络兼容良好。即使你已经在使用 IPv6，你也应该能从中学到一些新的或有价值的东西。

## 简介

在深入细节之前，我想从一些你必须理解的基本概念开始。如果你已经在使用 IPv6，可能会觉得这些内容有些枯燥，但本文假设读者熟悉 IPv4 网络管理，但还不熟悉 IPv6。以下技术术语的定义将作为第一部分进行解释：

- IPv6 地址格式和文本表示
- 子网前缀和接口标识符
- 地址范围和区域

### IPv6 地址

这可能是与 IPv4 最显著的不同之处。IPv6 的地址长度为 128 位，而 IPv4 的地址长度为 32 位。图 1 中的第（1）行显示了一个示例。这段 IPv6 地址以冒号分隔的 8 个字段表示。每个字段中有 4 位十六进制数字，因为它的值是 16 位。这是对一个 128 位值的完整表示。然而，有一种推荐的 IPv6 地址文本表示方式。规则如下：

- 连续的“0”或前导“0”必须省略，
- 最长的连续“0”段（超过 16 位）必须用“::”替代，
- 尽可能缩短表示方式。

```sh
(1) 2001:0db8:0000:0000:0001:0000:0000:4444
(2) 2001:0db8:0000:0000:0001:0000:0000:4444
(3) 2001:db8:0:0:1:0:0:4444
(4) 2001:db8::1:0:0:4444
```

**图 1：以十六进制格式表示的 IPv6 地址（16 位字段 x 8）**

图 1 中的 (2) 是去掉连续零后的形式，(3) 是去掉第一个最长连续零后的形式。 (4) 是最终形式，您将在支持 IPv6 的软件输出中看到它，包括 FreeBSD。

这种地址文本表示方式的约定旨在保持一致性和简洁性。您可以使用从 (1) 到 (4) 的所有表示方式作为输入。最好将您的软件设计为在输出中使用 (4) 的形式。例如，日志文件中的地址就应该使用这种格式。这样就能在使用“grep”时便于查找 (4) 格式的地址。

### 子网前缀和接口标识符 (IID)

下一个关键词是“子网前缀”和“接口标识符 (IID)”。在此之前，让我们回顾一下 IPv4 地址。图 2 展示了 IPv6 和 IPv4 地址的结构。在 IPv4 中，地址以“点分八位组”格式表示。一个地址由主机地址和网络地址组成。主机地址标识“节点”或终端主机，而网络地址标识一个网络段，即一个节点可以在没有路由器的情况下相互通信的领域。网络地址是通过使用主机地址和子网掩码计算出来的，如图 2 所示。如果主机地址是 192.168.2.1，子网掩码是 255.255.255.0，那么网络地址就是 192.168.2.0。拥有相同网络地址的机器属于同一个网络段，并且可以直接通信。

![](https://github.com/user-attachments/assets/a9ec0565-32e7-44ff-94d3-203c0ea962e3)

**图 2：子网前缀和接口标识符**

IPv4 子网掩码设计为 32 位值，并且掩码中的“1”值不必是连续的。`255.255.255.0` 在二进制中表示为（`1111 1111 1111 1111 1111 1111 0000 0000`）。因此，前导“1”是连续的。今天你看到的大多数子网掩码应该是连续的，因为这是互联网路由领域工程的标准方式。在早期的阶段，使用了“网络类”，并且通过子网掩码定义了三种 A、B 和 C 类网络——分别为“255.0.0.0”、“255.255.0.0”和“255.255.255.0”。主机地址会自动决定属于哪一类，所以你不需要子网掩码。后来，这种做法发生了变化，引入了 CIDR（无类域间路由）。在 CIDR 策略中，子网掩码与主机地址独立，并且掩码中的前导“1”通常是连续的。因此，子网掩码可以通过前导“1”的长度来表示。这个长度称为“子网掩码长度”，当你想同时显示网络地址信息时，它通常在 IPv4 地址的末尾用斜杠表示（参见图 2 的最后一行）。

一些现代系统不再支持非 CIDR 子网掩码，但 FreeBSD 仍然支持它。你可以尝试以下操作，看看会发生什么。你有没有见过像这样的 `netstat(1)` 输出？

```sh
# ifconfig bge0 inet 192.168.2.1 netmask 255.128.255.0
# ifconfig bge0 | grep netmask
    inet 192.168.2.1 netmask 0xff80ff00 broadcast 192.255.0.255
# netstat -nrf inet | grep bge0
    192.128.0.0&0xff80ff00 link#1 U bge0
```

让我们回到 IPv6。IPv6 与“主机地址”和“网络地址”的概念几乎相同。128 位地址被分为两部分；一部分是“子网前缀”，另一部分是“IID（接口标识符）”。子网前缀对应于 IPv4 中的“网络地址”。它是由子网前缀和低位的“0”组成的 128 位地址。你可以考虑一个 128 位的子网掩码，但它始终通过前导“1”的长度来表示，类似于 IPv4 CIDR 子网掩码长度。在 IPv6 中，这称为“前缀长度”。

你可以将 IPv6 地址分配给你的设备上的一个接口。你需要一个子网前缀、一个 IID 和前缀长度来进行配置。对于 IPv6 节点（例如你的计算机），如果没有特定的原因，前缀长度通常是 64。在 IPv6 的核心规范中，前缀长度是可变的。然而，大多数与 IPv6 相关的协议假定 IID 是 64 位或更长。如果你没有指定前缀长度，默认会使用 64。分配给你 FreeBSD 设备的 IPv6 地址通常被分为 64 位的子网前缀和 64 位的 IID。接口标识符标识同一网络段中的节点。

当你从 ISP 获得互联网访问时，通常会收到一个 IPv4 地址。如果你的 ISP 支持 IPv6，它会提供一个 IPv6 “前缀”。它们通常会为你提供 `/48`、`/56`、`/60` 或 `/64` 的前缀。这意味着，如果是 `/48` 前缀，你可以为你的局域网使用 16 位，因为你设备的地址前缀长度始终是 `/64`。

## 在 FreeBSD 上尝试 IPv6

你是否已经厌倦了抽象的内容？让我们在你的 FreeBSD 设备上进入 IPv6 的世界。如前所述，FreeBSD 默认长期支持 IPv6。GENERIC 内核没有运行时开关来启用或禁用 IPv6。你所需要做的就是将 IPv6 地址添加到其中一个接口。

### 使用 ifconfig(8) 手动配置

为了逐步理解 IPv6，让我们手动配置它。以下示例假设 bge0 是你设备上的主要网卡（NIC）。你可以在你的工作 NIC 和 IPv4 网络上尝试它们，这不会破坏它们。

尝试执行图 3 中显示的命令行。第一个 `ifconfig bge0` 显示 bge0 的当前状态。它应该显示带有 IFDISABLED 选项的 nd6 选项行。这个选项意味着“此接口上的 IPv6 被禁用”。

第二个命令 `ifconfig bge0 inet6 -ifdisabled` 将移除此选项。你可以通过再次执行命令 `ifconfig bge0` 来再次检查状态。

通过第三个 `ifconfig` 命令，你应该能看到 `inet6` 这行。这是配置到 bge0 的 IPv6 地址。`fe80::d63d:7eff:fe78:fc64%bge0` 部分是地址，`prefixlen 64` 部分显示了前缀长度。`%bge0` 部分看起来有些奇怪，但它是地址的一部分，稍后会解释。

你可以对这个地址执行“ping”操作。IPv6 的 ping 命令是 `ping6(8)`。注意，在 FreeBSD 13.x 及以后的版本中，`ping(8)` 和 `ping6(8)` 已经合并为一个命令，但在较新的版本中 `ping6(8)` 仍然可用。

如果你的机器上运行着 `sshd(8)` 守护进程，你甚至可以使用 IPv6 地址进行连接。尝试图 3 中的最后一条命令。注意，如果你的 `sshd_config(8)` 配置文件中将 AddressFamily 限制为 `inet`，则连接会失败。可以通过执行 `sockstat -6 | grep sshd` 查看 `sshd(8)` 是否监听 IPv6 套接字。

从用户的角度来看，支持 IPv6 的 TCP/IP 应用程序（如 `ssh(1)`）的工作方式与 IPv4 完全相同，唯一不同的是地址格式。

```sh
% ifconfig bge0
bge0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
options=c019b <RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,VLAN_HWCSUM,TSO4,
VLAN_HWTSO,LINKSTATE>
ether d4:3d:7e:78:fc:64
inet 192.168.100.104 netmask 0xffffff00 broadcast 192.168.100.255
media: Ethernet autoselect (1000baseT <full-duplex>)
status: active
nd6 options=29<PERFORMNUD,IFDISABLED,AUTO LINKLOCAL>
# ifconfig bge0 inet6 -ifdisabled
% ifconfig bge0
bge0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
options=c019b <RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,VLAN_HWCSUM,TSO4,
VLAN_HWTSO,LINKSTATE>
ether d4:3d:7e:78:fc:64
inet 192.168.100.104 netmask 0xffffff00 broadcast 192.168.100.255
inet6 fe80::d63d:7eff:fe78:fc64%bge0 prefixlen 64 scopeid 0x1
media: Ethernet autoselect (1000baseT <full-duplex>)
status: active
nd6 options=21<PERFORMNUD,AUTO LINKLOCAL>
% ping fe80::d63d:7eff:fe78:fc64%bge0
PING6(56=40+8+8 bytes) fe80::d63d:7eff:fe78:fc64%bge0 --> fe80::d63d:7eff:fe78:fc64%bge0
16 bytes from fe80::d63d:7eff:fe78:fc64%bge0, icmp_seq=0 hlim=64 time=0.181 ms
16 bytes from fe80::d63d:7eff:fe78:fc64%bge0, icmp_seq=1 hlim=64 time=0.107 ms
16 bytes from fe80::d63d:7eff:fe78:fc64%bge0, icmp_seq=2 hlim=64 time=0.094 ms
ˆC
--- fe80::d63d:7eff:fe78:fc64%bge0 ping6 statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/std-dev = 0.094/0.127/0.181/0.038 ms
% ssh -v fe80::d63d:7eff:fe78:fc64%bge0
OpenSSH_7.9p1, OpenSSL 1.1.1k-freebsd 24 Aug 2021
debug1: Reading configuration data /home/hrs/.ssh/config
debug1: Reading configuration data /etc/ssh/ssh_config
debug1: Connecting to fe80::d63d:7eff:fe78:fc64%bge0 [fe80::d63d:7eff:fe78:
fc64%bge0] port 22.
debug1: Connection established.
....
```

**图 3：手动向 bge0 添加 IPv6 地址的命令行示例**

## IPv6 地址自动配置

已经配置了一个 IPv6 地址，并且它是可用的。但是，这个地址是从哪里来的呢？在 IPv4 中，你应该通过在 `ifconfig(8)` 命令行中指定地址来配置地址。我们所做的只是 `ifconfig bge0 inet6 -ifdisabled`。发生了什么？

FreeBSD 支持 IPv6 SLAAC（无状态地址自动配置）。这是 IPv6 中很棒的的一个功能，可以自动配置 IID。记住，IPv6 地址有前缀和 IID。前缀由网络运营商（你或其他人）提供，但 IID 是你必须选择的，它必须在同一网络上唯一。

SLAAC 通过使用 MAC 地址来填充 IID 部分。它并不完全与 MAC 地址相同，但基于 MAC 地址生成一个唯一的 IID。你可以通过检查 `ifconfig(8)` 输出中的 "nd6 options" 行来查看 SLAAC 是否已启用。如果它包含 "AUTO_LINKLOCAL"，则会自动生成并配置 IPv6 地址。

前缀部分呢？如果你没有配置任何 IPv6 网络，或者你的 ISP 没有提供 IPv6 前缀，那么你的 LAN 就没有 IPv6 前缀。然而，bge0 接口有一个地址 `fe80::d63d:7eff:fe78:fc64%bge0`。图 4 (1) 是 `ifconfig(8)` 输出的结果。它意味着在完全的 128 位格式下，地址是 (2)。在简短表示中，前缀是 `fe80::`。

`fe80::` 子网前缀是为“链路本地”通信保留的前缀之一。你可以在任何网络接口上自由使用这个前缀。这个前缀中的地址仅限于直接连接的机器之间在链路上的通信。在 IPv4 中，`169.254.0.0/16` 被保留用于相同的目的，但它是可选的，并且没有广泛用于实际通信。

在 IPv6 中，链路本地子网前缀在核心协议中是必不可少的。这就是为什么只要在接口上启用 IPv6，地址就会自动配置。

因此，你可以认为所有支持 IPv6 的接口都会至少有一个带有链路本地子网前缀的地址。你可以使用这个地址进行标准的 TCP/IP 应用程序通信，同时它也用于各种重要的通信。

![](https://github.com/user-attachments/assets/b6b3bfa2-cda4-4fa4-b278-0195a6056195)

**图 4：自动配置的 IPv6 地址**

## 地址类型、作用域和区域  

自动配置地址的最后一个神秘部分是 `%bge0`。如果你没有这个部分，ping6(8) 命令会失败。这是什么？  

IPv6 地址有它的“作用域”。作用域是一个拓扑范围，在这个范围内，地址可以作为唯一标识符使用。虽然过去的 IPv6 RFC 提出了多个作用域，但如今只有两个作用域是实际可用的，分别是“全局”作用域和“链路本地”作用域。全局作用域意味着地址在互联网中可路由；链路本地作用域仅在单个特定链路上有效。  

“区域”这个关键词与“作用域”相关。图 5 显示了一个包含两个或多个网卡的网络示例。区域是一个作用域地址必须唯一的范围。如果你有两个网卡，并且它们连接到同一网络，那么每个接口上都会自动配置一个 fe80:: 前缀地址。在这种情况下，这两个地址必须是唯一的，这个范围称为“链路本地区域”。每个链路本地区域都有一个区域标识符。在 FreeBSD 上，区域标识符就是网卡的作用域标识符或接口名称。你可以在 ifconfig(8) 的输出中看到关键词“scopeid”，关键词后面的值就是作用域标识符。


![](https://github.com/user-attachments/assets/ba373b77-228f-4431-bdd1-510d378f9fb7)


**图 5：区域与网络段之间的关系**

让我们回到 `%bge0`。这是该地址的区域标识符。链路本地作用域的地址在该区域内是唯一的，但在同一台机器上可能并不唯一。因此，必须指定要使用哪个区域。%bge0 部分是必需的，用于指定区域。bge0 的作用域标识符是 `1`，因此你也可以将其写作 `%1`。没有区域标识符的链路本地地址被视为无效。

IPv6 地址有两种类型：“单播”和“多播”。这与地址的作用域无关。如果是单播，则地址用于常规的 1:1 通信。如果是多播，则通信可以有多个接收者。  
前缀自动确定作用域和类型：  
• 如果前 8 位是 “1111 1111” (`ff00::`)，则是多播前缀；  
• 如果前 10 位是 “1111 1110 10” (`fe80::`)，则是链路本地作用域且是单播前缀（称为 LLA，链路本地地址）；  
• 其他是全局作用域的单播前缀（称为 GUA，全局单播地址）。

### 有用的多播地址  
在标准的 IPv4 部署中，多播地址并不常用。而在 IPv6 中，多播被广泛使用。更多细节将在后续章节中讨论，但我想介绍以下两个地址：  
• `ff02::1`—链路本地作用域，所有节点的多播地址  
• `ff02::2`—链路本地作用域，所有路由器的多播地址  

每个 IPv6 节点都会“加入”所有节点的多播地址。这意味着连接到指定区域的所有机器都会响应该地址的通信。可以尝试图 6 中显示的命令。如果没有其他支持 IPv6 的设备连接到 bge0，你将只看到来自自己机器的响应。如果有其他设备，你将看到多个 ICMPv6 回显应答。如果使用 ff02::2，链路上的路由器将会响应。如果你有 Apple macOS 或 iOS 设备，你也能看到它们，因为这些设备默认启用了 IPv6。

这两个链路本地多播地址对于诊断非常有用。请记住，你可以使用 ff02::1 查找 IPv6 邻居，使用 ff02::2 查找 IPv6 路由器。

```sh
% ping6 ff02::1%bge0
PING6(56=40+8+8 bytes) fe80::d63d:7eff:fe78:fc64%bge0 --> ff02::1%bge0
16 bytes from fe80::d63d:7eff:fe78:fc64%bge0 , icmp_seq=0 hlim=64 time=0.073 ms
16 bytes from fe80::21b:78ff:fe39:84f6%bge0 , icmp_seq=0 hlim=64 time=0.194 ms(DUP!)
16 bytes from fe80::225:90ff:fe13:503a%bge0 , icmp_seq=0 hlim=64 time=0.276 ms(DUP!)
16 bytes from fe80::a6ba:dbff:fee0:b190%bge0 , icmp_seq=0 hlim=64 time=0.351 ms(DUP!)
16 bytes from fe80::202:a5ff:fee9:4104%bge0 , icmp_seq=0 hlim=64 time=0.427 ms(DUP!)
16 bytes from fe80::202:a5ff:fee9:c87d%bge0 , icmp_seq=0 hlim=64 time=0.511 ms(DUP!)
16 bytes from fe80::8c:7aff:fe24:1c64%bge0 , icmp_seq=0 hlim=64 time=0.585 ms(DUP!)
16 bytes from fe80::202:a5ff:fee9:c3c9%bge0 , icmp_seq=0 hlim=64 time=0.658 ms(DUP!)
16 bytes from fe80::202:a5ff:fee9:3952%bge0 , icmp_seq=0 hlim=64 time=0.730 ms(DUP!)
16 bytes from fe80::202:a5ff:fee9:2e5e%bge0 , icmp_seq=0 hlim=64 time=0.802 ms(DUP!)
::
```

IPv6 节点配置  
一个 IPv6 节点必须具有以下配置：  
• 每个 NIC 上至少有一个链路本地地址，  
• 一个回环地址。  

回环地址是 `::1`。在 IPv4 中，使用 `127.0.0.1`，并配置在 `lo0` 接口上。`::1` 也会自动配置在 `lo0` 接口上。  

你可以在单个接口上配置多个 IPv6 地址。在 IPv6 中，多个地址可以用于同一个接口，目的是为不同的功能提供服务。例如，你可以使用 ifconfig(8) 命令添加另一个链路本地地址 `fe80::1/64`：  
```
% ifconfig bge0 inet6 fe80::1/64  
```
与 IPv4 不同，这不会移除或替换已经配置的 IPv6 地址。检查 ifconfig(8) 的输出，并尝试使用 ping6(8) 或 ssh(1) 来使用新地址。如果你想移除特定地址，可以使用 -alias 标志，如下所示：  
```
% ifconfig bge0 inet6 fe80::1/64 -alias  
```
### 能使用什么地址/前缀？  
你可能对可以实际使用的 IPv6 地址感到困惑。在 IPv4 网络中，你可能知道“私有地址空间”始终可以用于本地网络。`10/8`、`172.16/16` 和 `192.168/24` 被广泛使用。那么 IPv6 呢？  

如前文所述，你可以使用 `fe80::/64` 前缀进行链路本地通信。只要没有重复，你可以自由生成这些地址。大多数 TCP/IP 应用程序都可以与这些地址兼容。请注意，如果你使用链路本地单播地址，需要在地址中添加 %zoneid 部分。如果你从 ISP 获得了全球 IPv6 前缀，除了链路本地前缀外，你还可以使用它。如果你配置了 IPv6 的默认路由器，你就可以进行 IPv6 互联网通信。全球前缀并不取代链路本地前缀；链路本地地址对核心 IPv6 协议至关重要。即使你有全球地址，也不要忘记你需要链路本地地址。  

那么 IPv6 中没有类似于私有地址的替代方案吗？是的，也不是。ULA（Unique Local Address，唯一本地地址）是为类似目的保留的地址空间。然而，部署 ULA 需要比 IPv4 更加小心的考虑。  

有关 IPv6 部署场景的更多细节，包括上面问题的澄清，将在下一期中讨论。  

### 在 rc.conf 中的配置  
最后，让我们看看如何在 `/etc/rc.conf` 中配置 IPv6 地址。对于 IPv4，使用 `ifconfig_bge0` 这行来配置 bge0 接口。对于 IPv6，使用 `ifconfig_bge0_ipv6` 来表示 bge0 是支持 IPv6 的。配置的最简单版本如下。在这个配置中，只有一个链路本地地址会自动配置：  
```sh
ifconfig_bge0="inet 192.168.0.10/24"  
ifconfig_bge0_ipv6="inet6 auto_linklocal"
```

如果你想手动添加另一个链路本地地址，可以在 `ifconfig_bge0_ipv6` 中添加 “inet6” 这行。此配置添加了两个链路本地地址，一个是通过 SLAAC 自动配置的，另一个是通过 `ifconfig_bge0_ipv6` 这行手动配置的：  

```sh
ifconfig_bge0="inet 192.168.0.10/24"  
ifconfig_bge0_ipv6="inet6 fe80::1/64"
```

可以通过与 IPv4 相同的方式添加更多地址。以下示例通过使用 `ifconfig_bge0_alias0` 这行添加了一个全球单播地址 `2001:db8::1/64`：  

```sh
ifconfig_bge0="inet 192.168.0.10/24"  
ifconfig_bge0_ipv6="inet6 fe80::1/64"  
ifconfig_bge0_alias0="inet6 2001:db8::1/64"
```

## 总结  
本栏目介绍了 IPv6 的基础知识和在 FreeBSD 上的配置示例。待`你理解了地址结构并通过 ifconfig(8) 配置，便可以使用自动配置的链路本地地址。尽管它们在互联网上不可全球路由，但仍然对本地网络通信或诊断有帮助。  

在下一期中，我们将介绍使用 IPv6 全球前缀的部署细节以及更多关于 FreeBSD 和第三方软件在启用 IPv6 网络上的配置示例。  

## 脚注  
1. RFC4291: “IP Version 6 Addressing Architecture”  
2. RFC 5952: “A Recommendation for IPv6 Address Text Representation”  
3. RFC 1519: “Classless Inter-Domain Routing (CIDR): an Address Assignment and Aggregation Strategy”  
4. RFC 4862: “IPv6 Stateless Address Autoconfiguration”  
5. IID 并不总是通过 MAC 地址生成，但它是最常见的实现之一。RFC 7217 和 RFC 8064 讨论了多个问题及新标准。这个话题将在本栏的后续期刊中介绍。  
6. RFC 3927: “Dynamic Configuration of IPv4 Link-Local Addresses”。微软称之为“自动私有 IP 地址分配（APIPA）”。  
7. RFC 4007: “IPv6 Scoped Address Architecture”  
8. 区域标识符 “bge0” 实际上是接口本地的，而非链路本地的。如果你有 bge0 和 bge1 且它们连接到同一网络段，则 `%bge0` 和 `%bge1` 表示相同的链路本地区域。FreeBSD 上的 IPv6 网络堆栈在内部假设接口本地和链路本地是相同的。  
9. RFC 1918: “Address Allocation for Private Internet”  
10. 不幸的是，少数应用程序与 LLA 不兼容。它们将在后续期刊中介绍。  
11. RFC 4193: “Unique Local IPv6 Unicast Addresses”  

---

**佐藤広生** 是东京工业大学的助理教授。他的研究主题包括晶体管级集成电路设计、模拟信号处理、嵌入式系统、计算机网络和一般软件技术。2006 到 2022 年间，他曾是 FreeBSD 核心团队成员，自 2008 年以来一直是 FreeBSD 基金会董事会成员，并自 2007 年起主持亚洲 BSDCon，亚洲地区的 BSD 派生操作系统国际会议。
