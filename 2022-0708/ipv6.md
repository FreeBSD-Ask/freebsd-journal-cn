# 实用 IPv6（第二部分）

- 原文链接：[Pragmatic IPv6 (Part 2)](https://freebsdfoundation.org/wp-content/uploads/2022/08/sato_IPv6.pdf)
- 作者：**佐藤広生**

第一部分解释了 IPv6 协议的基础知识，以及如何在 FreeBSD 设备上开始使用它。阅读完后，你应该能够使用自动配置的链路本地（link-local scope）IPv6 地址。尽管这些地址仅限于你的局域网（LAN），但它们仍然十分强大和有用——如果你只是想与同一网络上的另一台设备通信，那么根本不需要全局 IP 地址。链路本地地址是不可路由的，因此不太可能成为来自互联网恶意用户的攻击面。  

要访问 IPv6 互联网，你至少需要配置一个 IPv6 全局单播（global-scope unicast）地址。本专栏将聚焦于通过路由器部署 IPv6 的场景，以便更深入地理解实际的配置。  

## IPv4 和 IPv6 互联网  

在第一部分尝试了 sshd(8) 示例之后，你可能会想尝试访问 IPv6 互联网。让我们来看看需要做哪些准备，并了解 IPv6 网络设计的基础知识。  

如你所知，互联网是个全球范围的互联网络，由互联网协议（internet protocol，IP）驱动。要访问局域网络之外的资源，你需要一台能够连接到互联网的路由器。路由器掌握“通往外部的路由”，将你的网络中的 IP 数据包转发到其他网络。  

你应该已经拥有一台用于 IPv4 互联网的路由器。它可能是你的 ISP 提供的，或是位于数据中心的一台设备，用于连接你的服务器到互联网。通常，ISP 会提供一个可从互联网访问的上游网络连接端点。  

需要注意的是，尽管 IPv6 是 IPv4 的继任者，但 IPv6 并不兼容 IPv4。这意味着 IPv6 互联网和 IPv4 互联网是完全独立的，你需要一台 IPv6 路由器以及来自 ISP 的 IPv6 端点。  

IPv6 领域常见的误解之一是“IPv6 应该是向下兼容 IPv4 的”。这种误解来源于大多数 IPv6 部署实际上是通过迁移现有的 IPv4 网络实现的。例如，你可以让 FreeBSD 上的公共 IPv4 HTTP 服务器具备“IPv6 支持”，因为 FreeBSD 同时支持 IPv4 和 IPv6。然而，你可能不会选择让它成为“仅支持 IPv6”的服务器，因为没有 IPv6 连接的用户将无法访问你的服务。因此，IPv6 的部署通常是让现有的 IPv4 服务和网络在 IPv4 的基础上具备 IPv6 功能。这种迁移方式称为“双栈（dual-stack）”，也是导致人们误以为 IPv4 和 IPv6 总是可以在同一台机器上使用且彼此相关的原因之一。

## 设计 IPv6 网络  

IPv6 独立于 IPv4，因此你需要单独设计一个 IPv6 网络。幸运的是，在网络元素（如路由器和网络边界）方面，IPv6 与 IPv4 基本相同。让我们先回顾 IPv4 网络的工作方式，然后再了解 IPv6 的具体情况。  

### IPv4 局域网（LAN）配置  

IPv4 使用“网络地址”或“子网”的概念，其表示形式是主机地址和网络掩码（或子网前缀长度）。例如，IPv4 地址 **192.0.2.1/24** 表示你拥有一个网络地址 **192.0.2.0** ，在该网络中，可以使用 **192.0.2.1** 到 **192.0.2.254** 之间的 254 个主机地址。  

具有相同网络地址的节点共享同一个 **L2（数据链路层）** 段。换句话说，同一段内的两个节点可以直接通信，无需路由器。如果一个节点想与 **192.0.2.0** 之外的网络地址通信，则必须将数据包发送到一个能够到达该目标的路由器。因此，要配置一台主机，至少需要以下信息：  

- 一个全局 IPv4 主机地址  
- 一个网络掩码或子网前缀长度，用于计算网络地址  
- 路由器信息，以便与网络外的节点通信  

在 IPv4 端主机上，通常通过指定同一网络上的 **默认路由（default router）** 来配置路由信息。这三个元素也可以通过 **DHCP**（动态主机配置协议）自动配置。虽然 DHCP 是一个可选协议，但它在 IPv4 端主机上非常常见。FreeBSD 提供了 **dhclient(8)** 作为客户端实现。  

在 IPv6 网络中，此三要素在一定程度上可以自动配置。当然，你仍然可以手动配置它们，但不推荐这样做，因为 IPv6 地址空间非常庞大。接下来，我们详细了解 IPv6 网络的工作原理。  

### 单路由器网络，手动配置  

**图 1a** 展示了一个简单的 IPv6 网络，它包含一个路由器和同一段上的两个（及更多）IPv6 主机。要手动配置该网络，可以在其中一台 IPv6 主机上使用以下 **rc.conf** 变量：  

![](https://github.com/user-attachments/assets/665d7a0b-77c1-4425-9e07-dd8fb54ed2d5)

**图 1a：带有路由器的简单 IPv6 网络。**

```sh
ifconfig_bge0="inet 192.168.0.10/24"
ifconfig_bge0_ipv6="inet6 2001:db8:0:1::1/64"
ipv6_defaultrouter="fe80::5a52:8aff:fe10:e323%bge0"
```

这里假设主机的网络接口是 **bge0** ，由 ISP 提供 IPv6 路由，并且你的全局 IPv6 前缀为 **2001:db8:0:1::/64** 。你可以自行选择主机地址，在本例中，选择 **2001:db8:0:1::1/64** 。  

**ifconfig_bge0** 这行用于配置 IPv4 地址。但正如前文所述，IPv6 独立于 IPv4，如果只使用 IPv6，则无需添加 IPv4 地址。对于仅支持 IPv6 的配置，可以将其改写为如下形式：  

```sh
ifconfig_bge0="up"
ifconfig_bge0_ipv6="inet6 2001:db8:0:1::1/64"
ipv6_defaultrouter="fe80::5a52:8aff:fe10:e323%bge0"
```

请注意，你 **不能** 省略 **ifconfig_bge0** 这行，也不能在这一行直接写入 IPv6 配置。这一行的作用是让 **rc.d(8)** 框架知道 **bge0** 接口的存在。如果没有这行，将不会配置 **bge0**。  

虽然这里不需要 IPv4 地址，但该行不能留空。因此， **"up"** 被用作 **ifconfig(8)** 命令的一个无害子命令，**"up"** 仅用于激活接口。这是出于历史原因的要求，后续版本的 FreeBSD 可能会有所更改，但请记住，直到撰写本文时，FreeBSD 13.x 及更老版本都需要这一规则。IPv6 配置应当写在变量 **ifconfig_bge0_ipv6** 中。  

**ifconfig_bge0_ipv6** 这行用于配置 IPv6 地址和相关选项。 **rc.d(8)** 框架会使用这一行来判断接口是否启用了 IPv6。如果省略这一行，该接口上的 IPv6 通信将被阻止。在 **第一部分** ，我们使用了 **ifconfig_bge0_ipv6="inet6 auto_linklocal"** 。即使 IPv6 地址是自动配置的，这行仍然是必需的。  

变量 **ipv6_defaultrouter** 的作用与 **defaultrouter** 在 IPv4 中的作用相同，即指定默认路由器的地址。你需要提供路由器的 IPv6 地址。通常情况下，这一信息不会被明确提供，你可以使用 **ping6(8)** 工具来查找路由器的地址。  

```sh
% ping6 ff02::2%bge0
PING6(56=40+8+8 bytes) fe80::5a9c:fcff:fe10:ffc2%bge0 --> ff02::2%bge0
16 bytes from fe80::5a52:8aff:fe10:e323%bge0, icmp_seq=0 hlim=255 time=0.996 ms
16 bytes from fe80::5a52:8aff:fe10:e323%bge0, icmp_seq=1 hlim=255 time=1.099 ms
ˆC
--- ff02::2%bge0 ping6 statistics ---
2 packets transmitted , 2 packets received , 0.0% packet loss
round -trip min/avg/max/std-dev = 0.996/1.048/1.099/0.052 ms
```

**ff02::2** 是 **所有路由（all-routers）** 的组播地址。使用 **ping6(8)** 工具向该地址发送 **ICMPv6 回显请求（echo request）** 数据包时，网络中的路由器将接收该请求，并返回 **ICMPv6 回显应答（echo reply）** 数据包。你可以通过观察这些应答来获取路由器的地址。例如，返回的地址可能是 **fe80::5a52:8aff:fe10:e323%bge0** 。  

在 **/etc/rc.conf** 中添加上述配置后，你可以使用 **service(8)** 工具来重新配置 **bge0** ：  

```sh
# service netif restart bge0
```

重新配置完成后，你会注意到 **bge0** 拥有两个地址：**2001:db8:0:1::1/64** 和 **fe80::xxx/64** 。后者是第一部分介绍的自动配置的链路本地（link-local）地址。请记住，即使你使用的是 ISP 提供的全局 IPv6 地址，**IPv6 兼容的接口上至少必须配置一个链路本地地址** 。  

其中 **“xxx”** 部分取决于接口的 MAC 地址。系统会根据目标地址，从这两个地址中选择一个作为源地址。  

现在，你可以使用 **ping6(8)** 命令测试 **www.freebsd.org** 连接，如下所示：  

```sh
% ping6 www.freebsd.org
PING6(56=40+8+8 bytes) 2001:db8:0:1::1 --> 2610:1c1:1:606c::50:25
16 bytes from 2610:1c1:1:606c::50:25, icmp_seq=0 hlim=46 time=155.715 ms
16 bytes from 2610:1c1:1:606c::50:25, icmp_seq=1 hlim=46 time=151.051 ms
16 bytes from 2610:1c1:1:606c::50:25, icmp_seq=2 hlim=46 time=152.218 ms
ˆC
--- wfe2.nyi.freebsd.org ping6 statistics ---
3 packets transmitted , 3 packets received, 0.0% packet loss
round -trip min/avg/max/std-dev = 151.051/152.995/155.715/1.982 ms
```

你应该会看到一个全局地址 **2001:db8:0:1::1/64** 。如果目标地址是 **fe80::5a52:8aff:fe10:e323%bge0** ，则会使用链路本地地址进行通信。  

请注意，**DNS 域名解析** 由 **/etc/resolv.conf** 文件中列出的域名服务器（name servers）完成。如果你的 **IPv4 配置已正常工作** ，那么该文件中会列出 IPv4 地址。如果你的 **IPv6 路由器** 充当 **DNS 代理（DNS proxy）** ，你可以在 **/etc/resolv.conf** 中添加 IPv6 地址，例如：  

```sh
nameserver fe80::5a52:8aff:fe10:e323%bge0
```

如果你的 ISP 提供 **DNS 递归解析器（recursive resolver）** ，你也可以类似地添加其 IPv6 地址。  

至此，IPv6 主机的全手动配置已完成。现在，你可以使用支持 IPv6 的常用软件畅享 IPv6 互联网访问了。  

## 单路由器网络，使用 SLAAC 进行配置

**图 1b** 展示了相同的网络结构，但在这种情况下，**IPv6 主机通过路由器发送的消息自动配置** 。  

**NDP（邻居发现协议，Neighbor Discovery Protocol，NDP6）** 是 **IPv6 核心协议** 的一部分，它基于 **ICMPv6** 实现，并定义了 **自动配置能力** 。  

路由器可以通过 **RA（路由通告，Router Advertisement）** 消息 **分发网络参数信息** 。

![](https://github.com/user-attachments/assets/af6ceedb-d9f4-47ac-90c4-42d6b1d9b78a)

**图 1b：通过 SLAAC 进行配置。**

**RA（路由通告）消息** 将发送到地址 **ff02::1**，因此同一网络上的所有 IPv6 节点都将接收到这些消息。**图 1b** 中的 **RS（路由请求）消息** 是 **Router Solicitation** 消息，用于请求 **RA 消息**，并发送到 **ff02::2**，即 **所有路由器（allrouters）组播地址**。当路由器收到来自主机的 **RS 消息** 时，它将发送 **RA 消息**，并且还会定期发送 **非请求性的 RA 消息**，以便让 IPv6 主机了解网络信息的变化。  

**RA 消息** 具有许多选项，这些选项类似于 **IPv4 DHCP** 中的选项。最基本的选项包括 **MTU**、**前缀** 和 **默认路由器地址**，以便 IPv6 主机可以利用这些信息进行自我配置。  

如果你的路由器支持 **RA 消息** 并且你希望依赖它们，以下配置可以在 IPv6 主机上使用：  


```sh
ifconfig_bge0="up"
ifconfig_bge0_ipv6="inet6 accept_rtadv"
```

**inet6 accept_rtadv** 启用 **bge0** 接口接收 **RA 消息** 并进行接口配置。IPv6 路由器通常会提供你网络的前缀信息。当主机接收到前缀信息时，**IID（接口标识符）** 将自动从 **MAC 地址** 生成，进而配置 IPv6 地址。这个机制称为 **SLAAC（无状态自动地址配置）**。你可以在 **ifconfig(8)** 的输出中看到这个自动配置的地址，地址后会显示关键字 **“autoconf”** 。


```sh
% ifconfig bge0
...
inet6 2001:db8:a743:3c00:5a9c:fcff:fe10:ffc2 prefixlen 64 autoconf
...
```

要检查你的 IPv6 路由器是否支持 **RA 消息**，你可以使用 **rtsol(8)** 工具并加上 **-D** 标志。该命令会发送一个 **RS 消息**，并显示来自路由器的 **RA 消息**：

```sh
# rtsol -D bge0
rtsol: link -layer address option has null length on bge0. Treat as not included.
rtsol: checking if bge0 is ready...
...
rtsol: received RA from fe80::5a52:8aff:fe10:e323 on bge0 , state is 2
rtsol: Processing RA
rtsol: ndo = 0x7fffffffe3b0
rtsol: ndo->nd_opt_type = 1
rtsol: ndo->nd_opt_len = 1
rtsol: ndo = 0x7fffffffe3b8
rtsol: ndo->nd_opt_type = 3
rtsol: ndo->nd_opt_len = 4
rtsol: rsid = [bge0:slaac]
rtsol: stop timer for bge0
rtsol: there is no timer
```

当接口变为“up”状态后，**RS 消息** 也会被发送。请注意，处理 **RA 消息** 的是内核，而不是这个工具。因此，如果接口没有配置 **“inet6 accept_rtadv”**，消息会被显示出来，但实际上并没有进行配置。如果你没有看到 **“received RA from...”** 这一行，说明你的 IPv6 路由器没有响应 **RS 消息**。在这种情况下，你无法使用自动配置。在接收到 **RA 消息** 后，**SLAAC 地址** 会自动配置。同时，默认路由器也会通过消息中的地址来配置。因此，只需将 **“inet6 accept_rtadv”** 添加到 **/etc/rc.conf** 中，即可配置全局 IPv6 地址和默认路由器。

这类似于 IPv4 中的 DHCP。然而，**RA/RS 消息** 没有服务器，也没有“状态”。终端主机在接收到这些消息后会自动配置网络参数。虽然这种方法比 **IPv4 DHCP** 更具可扩展性，但你无法控制每个主机上实际配置的地址，因为这些地址是从 MAC 地址生成的。如果需要更精细地控制自动地址配置，你需要采用其他方法，如 DHCPv6。  

也可以通过 **RA 消息** 配置 **DNS 递归解析器**。由于内核无法处理这些信息，**rtsold(8)** 守护进程将负责处理它。你可以通过以下 **rc.conf(5)** 变量来启用 **rtsold(8)** 守护进程：

```sh
ifconfig_bge0="up"
ifconfig_bge0_ipv6="inet6 accept_rtadv"
rtsold_enable="YES"
rtsold_flags="bge0"
```


当接收到 **RA 消息** 时，如果你的路由器提供了相关信息，**/etc/resolv.conf** 将会被更新。

应在配置正确的 IPv6 网络中启用 **RA 消息**，网络中应有一个及多个路由器。如果没有 **RA 消息**，网络配置将变得困难，因为 **RA 消息** 包含了网络参数及其配置方法。

## IPv6 路由器配置

上一部分假设由你的 ISP 提供 IPv6 路由。你也可以自己构建一个 **IPv6 FreeBSD 路由**，就像配置 **IPv4 路由** 一样。让我们来看看构建该路由器需要配置什么内容。

**图 2a** 展示了一个示例，假设你的网络中有一台 **IPv6 路由器**，用于连接两个独立的网络。**LAN 1** 和 **LAN 2** 通过路由器相互连接，另一台路由器提供 **IPv6 互联网可达性**。本节将解释如何配置前者。

![](https://github.com/user-attachments/assets/5381fafe-f9f5-4562-91b0-1615858852a6)

**图 2a：带有两台路由器的 IPv6 网络**

要启用数据包转发，你需要在 **/etc/rc.conf** 中设置 **ipv6_gateway_enable** 变量。这个变量是 **gateway_enable** 的 **IPv6 对应项**，后者用于 **IPv4**。假设 **bge0** 和 **bge1** 是路由器的网络接口，分别连接到 **LAN 1** 和 **LAN 2**，则 **/etc/rc.conf** 配置文件大致如下：

```sh
ipv6_gateway_enable="YES"
ipv6_defaultrouter="fe80::5a52:8aff:fe10:e323%bge0"
ifconfig_bge0="up"
ifconfig_bge0_ipv6="inet6 2001:db8:0:1::1/64"
ifconfig_bge1="up"
ifconfig_bge1_ipv6="inet6 2001:db8:0:2::1/64"
```

请注意，配置此项时，你必须使用比 64 更短的前缀。例如，如果你的 ISP 提供了 **2001:db8::/56** 的网络，你可以通过将 **/56 网络** 分割，使用 **2001:db8:0:1::/64** 和 **2001:db8:0:2::/64**。你可能会想把 **/64 前缀** 切割成更长的前缀。然而，作者不推荐使用比 **64** 更长的前缀来设计你的网络。这个话题将在后续的文章中再次讨论。

ISP 提供的路由器并不知道到 **2001:db8:0:2::1/64** 的路由，你需要通过使用 **bge0** 上的 **链路本地地址** 添加静态路由配置。正如在第一部分中解释的那样，**链路本地地址** 会自动配置。

![](https://github.com/user-attachments/assets/56c01e56-7a8c-4448-b600-a657994b699a)

**图 2b：LAN 1 和 LAN 2 上的 SLAAC 配置**

图 2b 显示了来自这两台路由器的 **RA 消息**。这台 **FreeBSD 路由器** 还没有启用发送 **RA 消息**。要发送 **RA 消息**，你需要使用 **rtadvd(8)** 守护进程。可以通过设置 **rtadvd_enable** 和 **rtadvd_interfaces** 变量来启用它。

```sh
ipv6_gateway_enable="YES"
ipv6_defaultrouter="fe80::5a52:8aff:fe10:e323%bge0"
ifconfig_bge0="up"
ifconfig_bge0_ipv6="inet6 2001:db8:0:1::1/64"
ifconfig_bge1="up"
ifconfig_bge1_ipv6="inet6 2001:db8:0:2::1/64"
rtadvd_enable="YES"
rtadvd_interfaces="bge1"
```

LAN 2 上的 **IPv6 主机** 将接收来自路由器的 **RA 消息** 并执行自动配置。**rtadvd(8)** 守护进程分发默认配置的前缀和链路 MTU 等信息。更多信息（如 DNS 服务器）可以通过创建 **/etc/rtadvd.conf** 来分发。有关详细信息，请参阅 **rtadvd.conf(8)** 手册页面。

需要注意的是，路由器不会接收到 **RA 消息**。在 IPv6 规范中，IPv6 节点分为主机和路由器。**主机** 是网络的叶节点，不转发 IPv6 数据包；而 **路由器** 是一个多宿节点，负责跨网络转发 IPv6 数据包。**RA 消息** 被定义为由路由器发送并由主机接收的消息。

这意味着我们不能通过前面部分中解释的自动配置能力来配置路由器。你必须在路由器上 **不指定** “inet6_accept_rtadv”，并需要像上面示例中那样手动配置网络参数。如果你指定了 **ipv6_gateway_enable="YES"**，即使指定了“inet6_accept_rtadv”，**FreeBSD 内核** 也会忽略 **RA 消息**。

然而，在某些情况下，这种模型过于严格。例如，这种主机与路由器模型不适用于 ISP 提供的 IPv6 路由器。该路由器必须自动配置，但如果它没有接收到 **RA 消息**，就无法配置默认路由。另一方面，如果路由器接收到 **RA 消息** 来配置自己，配置很快会被其他路由器的消息覆盖。另一台路由器会改变该路由器的默认路由。

为了解决这个问题，FreeBSD 采用了以下概念：

- “主机/路由器”是在每个接口上确定的，而不是系统范围的属性。
- 如果接口接受 **RA 消息**，则从其他节点看它是 **“主机”**。

基于此，**accept_rtadv** 标志可以在每个接口的基础上配置。虽然数据包转发能力不能以类似的方式配置，但提供了一个 sysctl 参数 **net.inet6.ip6.rfc6204w3** 。当设置为 **1** 时，内核即使在启用数据包转发的情况下，也会接收 **RA 消息**。虽然这些参数较难理解，但详细信息和具体示例将在后续章节中覆盖。

## 使用 DHCPv6

**DHCP** 也适用于 **IPv6**，被称为 **DHCPv6**。然而，像 IPv4 那样广泛使用的情况不多，因为通过 **RA 消息** 和 **SLAAC** 的自动配置已经足够应付小型网络。图 **3a** 显示了一个 ISP 使用 **DHCPv6** 的示例。虽然你可以使用 **FreeBSD** 来实现一个有 **DHCPv6 服务器和客户端** 的网络，但与配置细节相关的话题将在后续章节中讨论。这里，我们关注的是 **使用 DHCPv6 的网络如何工作**。

![](https://github.com/user-attachments/assets/11365908-8b16-42f2-b548-0cd2ca0d7c4f)

**图 3a：通过 SLAAC 和 DHCPv6 IA-NA 配置**

首先，RA 消息和 DHCPv6 是协同工作的，而不是相互冲突。DHCPv6 是提供 RA 消息中无法提供的信息的一种方式。有些信息只能通过 RA 消息获得，有些则只能通过 DHCPv6 获得。DHCPv6 可以分发 IPv6 地址。DHCPv6 服务器、客户端和分发的地址信息之间的关系称为 IA（身份关联）。IA 是针对地址类型定义的，最著名的有 IA_NA（非临时地址）和 IA_PD（前缀委派）。

IA_NA 类似于 IPv4 DHCP——它是分发给附加到与服务器相同网络的接口的地址。在图 3a 中，主机建立了一个 IA_NA。主机从 DHCPv6 服务器接收 IPv6 地址。同时，主机也会接收到 RA 消息。这意味着，另一种地址通过 SLAAC 被配置。因此，在这种情况下，主机将配置两个地址。

请注意，IA_NA 只分发地址，而不包含前缀和默认路由的信息。主机仍然需要接收 RA 消息以完成网络配置。DHCPv6 并不是 RA 消息的替代品。

![](https://github.com/user-attachments/assets/4b7957f3-bb9e-4906-a994-6bd94e656998)


**图 3b：用于路由器配置的 DHCPv6-PD**

图 3b 显示了一个 IA_PD，这是用于自动配置路由器的一项功能。它是仅在 DHCPv6 中可用的一项新特性。它在另一个网络 LAN 2 上的接口配置一个前缀。当 LAN 1 和 LAN 2 之间的 IPv6 路由器在 LAN 1 上建立 IA_PD 时，获取的前缀信息将用于 LAN 2。LAN 1 上的接口将通过 RA 消息进行配置。这看起来像是一个复杂的行为，但配置 IA_PD 所需的只是指定 IA 所在的接口和用于获取前缀的另一个接口。

在这两种情况下，DHCPv6 的服务发现都是通过 RA 消息进行的。在 IPv4 DHCP 中，客户端通常通过广播 DHCP DISCOVER 消息到附加网络以寻找服务器。而在 IPv6 中，RA 消息则告诉客户端如何配置自己。因此，使用 DHCPv6 的网络配置大致如下：

```sh
ifconfig_bge0="up"
ifconfig_bge0_ipv6="inet6 accept_rtadv"
rtsold_enable="YES"
rtsold_flags="-0 /usr/local/etc/dhcp6c.sh bge0"
```

rtsold(8) 守护进程将处理 RA 消息中的标志，该标志指示是否部署了 DHCPv6 服务器，以及客户端是否应该在网络上使用它。-O 选项接受一个文件名，当启用该标志时，它将作为可执行文件被调用。默认情况下，该选项是禁用的，因为 FreeBSD 的基本系统中没有 DHCPv6 客户端。通常，这是一个 shell 脚本，用于调用 DHCPv6 客户端软件。即使没有 DHCPv6 服务器，这个配置也能正常工作——你不需要特定的配置来直接调用 DHCPv6 客户端软件。

简而言之，DHCPv6 是自动配置的另一种选择。它广泛用于通过 IA_PD 自动配置 IPv6 路由器，这通常称为 DHCPv6-PD。即使部署了 DHCPv6 服务器，RA 消息仍然用于同一网络上的 IPv6 节点。DHCPv6 唯一配置不足的一个重要原因是，DHCPv6 没有配置默认路由器的选项。

## 使用 PPPoE  

一些 ISP 使用 PPPoE 提供客户侧的端点。如前所述，以太网上的网络对 IPv4 和 IPv6 都有效。然而，ISP 无法控制谁连接到他们的网络，因为路由器没有实现身份验证。PPPoE 是克服这一问题的方式之一。

图 4 是一个常见的 PPPoE 配置。这里有两个路由器，PPPoE 路由器和 IPv6 路由器，但在实践中，通常是一个物理设备实现这两种功能。在这种情况下，IPv6 路由器和 ISP 网络通过以太网连接，但网络接口没有 IPv6 地址。路由器和 ISP 侧的端点之间通过以太网连接建立了一个虚拟点对点链路，所有数据包都通过这个链路传输。身份验证可以在链路协商期间进行。


![](https://github.com/user-attachments/assets/7674e4b7-771c-4eef-9de2-6320b887317d)

**图 4：提供 IPv6 可达性的 PPPoE 隧道。**

点对点链路将通过以下方式建立：

- PPPoE 路由器通过向 ISP 网络发送 PADI（PPP 活跃发现启动）来找到 PPPoE 端点。PPP 服务器响应，并在两者之间建立虚拟链路。
- 在链路协商过程中，使用 IPv6CP 协议（PPP 协议的一部分）来获取 IPv6 信息。它为面向 WAN 的虚拟接口提供 IPv6 IID。使用此 IID 和到达接口的 RA 消息，可以生成完整的 IPv6 地址。此时，PPPoE 路由器成为可访问 IPv6 互联网。
- 使用虚拟链路，DHCPv6 IA_PD 与 ISP 端的 DHCPv6 服务器建立。IA_PD 配置面向 LAN 的接口。

在建立 PPPoE 链路并随后建立 IA_PD 后，面向 LAN 的接口可以作为 IPv6 路由器，转发数据包通过 PPPoE 虚拟链路到达 ISP。这是家庭网络中最复杂的配置之一。然而，这只是 RA 消息、PPPoE 和 DHCPv6 的组合。你可以通过使用 FreeBSD 和 net/mpd5 来实现这一点。

## 总结

本篇文章展示了连接到 IPv6 互联网的 IPv6 网络的设计和配置示例。所有这些都可以通过使用 FreeBSD 实现，尽管复杂的配置细节尚未描述。IPv6 网络配置的关键是理解自动配置功能。

如果你的 ISP 提供 IPv6 服务，并且你有全球 IPv6 前缀，请尝试配置你的 FreeBSD 主机。通过 RA 消息和 SLAAC 的自动配置很容易配置，并且 FreeBSD 默认支持它。

在下篇文章中，将涵盖更多 IPv6 部署的细节，包括配置示例和“隧道”上的配置。隧道是一种建立虚拟链路的技术，它可以用于让你的 FreeBSD 主机可达 IPv6 互联网。如果你的 ISP 不提供 IPv6 服务，无法尝试这里的示例，请耐心等待。你将在理解如何配置后使用 IPv6。

此外，IPv6 依赖多个地址，如上文示例所示。这是同 IPv4 的一个显著区别；每个系统管理员都应了解 IPv6 核心协议的详细行为。下篇文章将涵盖哪些 IPv6 地址被配置以及它们如何工作，包括单播和组播地址。

---

脚注：

1. ISP：互联网服务提供商。
2. 192.0.2.0 和 192.0.2.255 是保留地址，通常不能作为主机地址使用。
3. L2 代表“第二层”网络。以太网是 L2 协议的典型例子，IP 协议族在其之上工作。
4. 动态主机配置协议（RFC 2131）。
5. 作者计划提出更改，使得 ifconfig_bge0 可以在 FreeBSD 14.x 版本中接受 IPv6 配置。
6. IPv6 邻居发现协议，RFC 4861。
7. 接口标识符（Interface IDentifier）。IPv6 地址的下部分，用于标识节点。通常为 64 位长。
8. IPv6 无状态地址自动配置，RFC 4862。
9. IPv6 路由器广告 DNS 配置选项，RFC 8106。
10. IPv6 动态主机配置协议（DHCPv6），RFC 8415。
11. 以太网上传输 PPP 的方法（PPPoE），RFC 2516。

---

**佐藤広生** 是东京工业大学的助理教授。他的研究课题包括晶体管级集成电路设计、模拟信号处理、嵌入式系统、计算机网络和软件技术等。他曾是 FreeBSD 核心团队成员（2006-2022），自 2008 年起担任 FreeBSD 基金会董事会成员，并自 2007 年起主持亚洲 BSDCon，这是一项关于 BSD 衍生操作系统的国际会议。
