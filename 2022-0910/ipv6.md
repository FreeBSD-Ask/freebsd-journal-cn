# 实用 IPv6（第三部分）

- 原文链接：[Pragmatic IPv6 (Part 3)](https://freebsdfoundation.org/wp-content/uploads/2022/11/sato_IPv6_part3.pdf)
- 作者：**佐藤広生**

上期专栏介绍了 IPv6 的典型部署示例，适用于只有一台上行路由器的小型网络，例如家庭网络。但其中没有涉及某些复杂的配置（比如 DHCPv6/PPPoE），因为在此之前我们需要先掌握一些 IPv6 的技术知识。在深入讨论这些复杂情况之前，让我们先通过配置你的 FreeBSD 设备来进一步了解 IPv6。本专栏将重点讨论以下两个主题：当你的 ISP 无法提供 IPv6 互联网访问时该如何处理，以及如何在 FreeBSD 基础系统中配置必备的 IPv6 实用工具。

### 获得 IPv6 互联网访问的另一种方法

上期专栏中的部署场景假设你的 ISP 提供 IPv6 服务。在这种情况下，你的 ISP 与家庭网络之间的路由器会拥有一个 IPv6 全球单播地址（GUA¹）。你只需配置一个或多个 IPv6 地址，并将 IPv6 默认路由地址指向该路由器即可。

那么，如果你无法获得 IPv6 服务该怎么办？虽然提供 IPv6 服务给终端用户的 ISP 数量在不断增加，但截至 2022 年，大多数 ISP 的 IPv6 服务仍然属于可选项，并非其主要服务之一。获得 IPv6 互联网访问的一种方法是使用隧道连接。图 1 展示了“隧道传输”的工作原理：  
- 网络 A 是一个拥有 IPv6 互联网访问的 IPv6 网络；  
- 网络 B 是一个孤立的 IPv6 网络。  

如果两个网络的终端都拥有 IPv4 地址，它们就可以通过 IPv4 互联网互相连接。隧道传输是一种协议转换技术，它将一个数据包封装在另一种协议的载荷中传输。也就是说，IPv6 数据包可以通过 IPv4 协议（如 IPv4 TCP 或 IPv4 UDP）进行传输。如果你熟悉 VPN（虚拟专用网络）的概念，那么你可以将隧道传输视作 VPN 的技术基础。一旦隧道建立，网络 B 上的主机便可以通过网络 A 访问 IPv6 互联网。

![](https://github.com/user-attachments/assets/455c8a65-f47f-47be-8856-666911bd7868)

**图 1：通过 IPv4 通信连接的 IPv6 网络**

几个隧道服务支持 IPv6-over-IPv4 隧道传输，而且这些服务大多是免费的。其中一个可靠的服务——Hurricane Electric IPv6 Tunnel Broker，将在下面的小节中进行说明。

### Hurricane Electric IPv6 Tunnel Broker

Hurricane Electric 是一家总部位于加利福尼亚的互联网服务提供商，主要提供互联网中转、数据中心托管以及主机服务。他们的网络覆盖规模堪称全球最大。通常，在文章中介绍这种具体的互联网服务是不合适的，因为信息很快就会过时。不过，HE 的 IPv6 隧道传输服务已持续维护了 20 多年，对于那些没有原生 IPv6 访问的人来说，它一直是一个不错的测试环境。因此，作者推荐将其作为实验性获取 IPv6 互联网访问的一种方式。虽然你不能假设其可靠性和性能与 ISP 提供的原生 IPv4 互联网连接完全相同，但 HE 的服务表现良好，至少对个人使用来说足够。而且，另一个优点是 HE 提供一个 /48 的 IPv6 地址前缀。/48 是 ISP 推荐的前缀长度，以便其终端用户可以利用 16 位划分享有多个局域网（LAN）；然而，许多 ISP 仅提供一个或两个 /64 前缀。

接下来，让我们看看如何在你的 FreeBSD 设备上配置隧道传输。

### 服务账户注册

隧道传输使用你网络中的一个端点和 HE 网络中的另一个端点。在这两者之间，将通过 IPv4 互联网建立一个虚拟网络。你这边的端点，即 FreeBSD 设备，将作为 IPv6 路由器。作为前提，你的 FreeBSD 设备必须拥有一个全球可达的 IPv4 地址。该隧道传输使用协议号为 41 的 IP 数据包。这个协议号与 IPv6 相同，因此数据包过滤防火墙不太可能会阻止它们。你必须避免使用 IPv4 私有地址空间（10.0.0.0/8、172.16.0.0/12 或 192.168.0.0/16）进行 IPv4 网络地址转换（NAT）。

访问 [https://www.tunnelbroker.net/](https://www.tunnelbroker.net/) 并创建你的服务账户。之后，在网页界面中选择 “create regular tunnel”（创建常规隧道）。输入你端点的 IPv4 地址，并选择 HE 提供的其中一个端点。你网络中的所有 IPv6 数据包，在到达 IPv6 互联网之前，都会先传送到 HE 的网络，因此你应选择离你最近的端点。例如，作者住在东京，那么选择 HE 的东京端点最佳。点击 “create tunnel” 按钮后，你在 HE 这边的端点就会被配置。你需要点击两次，因为第一次点击时，系统会从 HE 的网络发送 IPv4 ping 数据包以检查该端点是否正常工作。请确保你的防火墙没有阻止传入的 ICMP 回显请求/应答。HE 端点列表中会显示 IPv4 源地址。

### 网络参数

在 “Tunnel Details”（隧道详情）页面上，会显示四个 IP 地址，分别为 “Server IPv4 Address”（服务器 IPv4 地址）、“Server IPv6 Address”（服务器 IPv6 地址）、“Client IPv4 Address”（客户端 IPv4 地址）和 “Client IPv6 Address”（客户端 IPv6 地址）。  
- **Server IPv4 Address** 是 HE 那边端点的地址；  
- **Server IPv6 Address** 则是你网络中应使用的 IPv6 默认路由器地址；  
- **Client IPv6 Address** 是你端点的地址。

请注意，你需要了解两个网络——一个是两个路由器之间的网络（该网络是虚拟的），另一个是你的局域网（LAN）。详见图 2。图中黄色圆圈代表一个接口。这两个网络由你的路由器分隔开，该路由器在本配置中同时作为两个端点，因此你将获得两个 IPv6 前缀。你可以看到服务器和客户端的 IPv6 地址位于同一子网前缀中，即这就是两个端点之间的虚拟网络。你的 IPv6 局域网的前缀显示为 “Routed IPv6 Prefixes”。你可以将其配置到路由器的 LAN 侧。默认情况下，你会获得一个 /64 前缀。点击 “Routed /48 prefix” 按钮后，除了 /64 前缀之外，你还会获得一个 /48 前缀。

![](https://github.com/user-attachments/assets/0e6e9e70-ff1e-4a55-a1aa-83ff6b6e0420)


**图 2：HE 的网络与您的网络之间的隧道传输**  

现在您已经可以开始配置您的 FreeBSD 设备了。

### 在您的 FreeBSD 设备上的配置

在“隧道详情”页面中，有一个“示例配置”标签页，您可以看到一个适用于 FreeBSD 4.4 或更高版本的配置示例，如下所示：

```sh
# ifconfig gif0 create
# ifconfig gif0 tunnel IPV4-CLIENT IPV4-SERVER
# ifconfig gif0 inet6 IPV6-CLIENT IPV6-SERVER prefixlen 128
# route -n add -inet6 default IPV6-SERVER
# ifconfig gif0 up
```

诸如 **IPV4-CLIENT** 这样的关键字表示那四个参数。这个示例并没有错误，但下面这个版本更好。

```sh
# ifconfig gif0 create
# ifconfig gif0 inet tunnel IPV4-CLIENT IPV4-SERVER
# ifconfig gif0 up
# ping6 ff02::1%gif0
(按 Ctrl-C)
# ifconfig gif0 inet6 IPV6-CLIENT IPV6-SERVER prefixlen 128
# route -n add -inet6 default IPV6-SERVER
# ifconfig bge0 inet6 ROUTED-IPV6-PREFIX/64
# sysctl net.inet6.ip6.forwarding=1
```
同样地，一个具有相同传输协议的 IPv6 数据包由三部分组成，不同之处在于第一部分。

如果我们将 IPv6 数据包视为 IPv4 数据包的数据，那么我们就可以通过 IPv4 网络传输 IPv6 数据包。在发送端，一台路由器构建一个包含 IPv6 数据包的 IPv4 数据包；而在接收端，另一台路由器提取出其中的 IPv6 数据包。当使用 gif(4) 接口时，这一过程就发生在 HE 的路由器和你的路由器之间。实际上，将 IPv6 数据包封装到 IPv4 数据包中，是通过在 IPv6 数据包前面加上一个 IPv4 头来完成的，如图 3 所示，因为传输层的头部几乎都是相同的。

要配置此接口，你需要两组地址。首先，必须使用命令 `ifconfig gif0 create` 创建一个新的 gif(4) 接口。接着，可以使用 ifconfig(8) 工具中的 “tunnel” 关键字，并为两个端点配置地址来设置隧道传输。第一个地址是你的，第二个地址是 HE 的。

![](https://github.com/user-attachments/assets/b69df646-736d-4812-b896-d5e23d6ccac6)


**图 3：gif(4) 接口中使用的封装**

之后，一个虚拟网络就建立起来了。请注意，这个虚拟网络没有连接状态，比如“已连接”或“已断开”。数据包仅在两个 IPv4 地址之间按需以“IP 数据报”的形式发送。一个 IPv6 数据包会被封装（如图 3 所示），作为 IPv4 数据包发送到 HE 的端点，并在 HE 的网络中被解封。虽然 IPv4 数据包可能在互联网的某处丢失，但错误恢复是由上层协议完成的，即 IP 数据报内部的 IPv6 数据包。用于隧道传输的地址通常被称为“外层协议地址”，在此案例中，外层协议是 IPv4。

要检查虚拟网络是否正常工作，你可以发送一个 IPv6 ping。在示例配置的第二行之后，输入 `ifconfig gif0 up`，然后使用 gif0 自动配置的链路本地地址（LLA）尝试发送 IPv6 ping。

```sh
# ifconfig gif0 create
# ifconfig gif0 tunnel IPV4-CLIENT IPV4-SERVER
# ifconfig gif0 up
# ping6 ff02::1%gif0
PING6(56=40+8+8 bytes) fe80::80c8:4a75:123a:8a32%gif0 --> ff02::1%gif0
16 bytes from fe80::80c8:4a75:123a:8a32%gif0, icmp_seq=0 hlim=64 time=0.084 ms
16 bytes from fe80::4a52:2e06%gif0, icmp_seq=0 hlim=64 time=4.765 ms(DUP!)
16 bytes from fe80::80c8:4a75:123a:8a32%gif0, icmp_seq=1 hlim=64 time=0.081 ms
16 bytes from fe80::4a52:2e06%gif0, icmp_seq=1 hlim=64 time=2.738 ms(DUP!)
ˆC
--- ff02::1%gif0 ping6 statistics ---
2 packets transmitted, 2 packets received, +2 duplicates, 0.0% packet loss
round-trip min/avg/max/std-dev = 0.081/1.917/4.765/1.970 ms
```

如果你收到 “(DUP!)” 响应，则说明虚拟网络运行正常。这是因为你应该会收到来自 HE 那边路由器的回复以及来自 gif0 接口本身的回复。如果你未收到来自端点另一侧的响应，请仔细检查两个端点之间配置的 IPv4 地址以及数据包过滤规则。

接下来，你需要配置用于通信的实际 IPv6 地址。这可以通过 ifconfig(8) 工具完成，如示例配置的第六行所示。该行配置了一个仅包含两个 IPv6 节点的点对点网络。此时，你可以向 IPV6-SERVER 发送 IPv6 ping。

如果你仅希望从该设备访问 IPv6 互联网，上述配置就足够了。此时，你的 FreeBSD 设备就可以作为一个 IPv6 主机节点工作。如果要使其成为你 /64 和 /48 局域网的路由器，则需要将默认的 IPv6 路由配置为指向 IPV6-SERVER，并启用 IPv6 数据包转发功能。示例中的下一部分就是用来实现这一点的。

```sh
# route -n add -inet6 default IPV6-SERVER
# ifconfig bge0 inet6 ROUTED-IPV6-PREFIX/64
# sysctl net.inet6.ip6.forwarding=1
```

该配置假设面向局域网 (LAN) 的接口为 bge0。**ROUTED-IPV6-PREFIX** 是 HE 服务分配给你的前缀，你应该同时拥有 /64 和 /48。你可以在同一接口上配置两者，也可以分配到不同的接口。

在这里，你需要考虑一个问题：应该使用什么 IID⁵？也就是 LAN 上路由器的地址。

你可能已经注意到，在隧道接口上使用的 IPv6 地址的 IID 分别是 `::1` 和 `::2`，这是一种可行的策略。然而，作者建议为简化起见，使用全零地址。如果你从 HE 获得了 `2001:db8::/48`，可以选择该范围内的一个 /64 网络，例如 `2001:db8:1::/64`，并将其配置为路由器地址。此时，IID 设为全零。

这可能会让你感到不安，因为在 IPv4 中，全零的主机地址具有特殊含义。虽然 RFC 或其他规范未明确定义，但一些传统网络实现将其视为广播地址，而非单播地址。然而，在 FreeBSD 上，全零主机地址不会有任何问题。但如果你是在 20 世纪学习的 TCP/IP 网络，可能已经习惯性地避免使用它。在 IPv6 中，全零 IID 是完全合法的。实际上，使用全零地址作为路由器地址还有其他优点，不过这些将在后续专栏中进一步讨论。

在配置了默认路由器和 LAN 上的路由器地址后，你就可以配置 LAN 上的主机了。你可以参考上一期专栏，了解可以采取的选项。建议在 bge0 上运行 **rtadvd(8)** 以启用自动配置。

### 在 `/etc/rc.conf` 中配置
待确认 IPv6 隧道可以正常工作，你应该将手动输入的配置添加到 `/etc/rc.conf`。

```sh
cloned_interfaces="gif0"
ifconfig_gif0="inet tunnel IPV4-CLIENT IPV4-SERVER"
ifconfig_gif0_ipv6="inet6 IPV6-CLIENT IPV6-SERVER prefixlen 128"
ipv6_defaultrouter="IPV6-SERVER"
ipv6_gateway_enable="YES"
ifconfig_bge0="inet6 ROUTED-IPV6-PREFIX/64"
```

在重启之前，你可以使用 **service(8)** 命令来检查其是否正常工作。例如，执行以下命令：  

这两个 **ping6** 命令用于检查隧道端点的可达性以及默认路由器的配置。

```sh
# service routing start
# service netif restart gif0
# ping6 ff02::1%gif0
# ping6 IPV6-SERVER
```

就是这样！现在，你可以在你的 LAN 内搭建支持 IPv6 的互联网服务器了，因为配置的 IPv6 地址可以从 IPv6 互联网访问。  

关于 **ipv6_defaultrouter**，还有一点需要补充说明。虽然上述示例使用 **IPV6-SERVER**，但它也可以是 HE 端的 LLA（链路本地地址），你可以通过 **ping** 查看，或者将其设置为 `-interface gif0`。  
- **不推荐使用 HE 端的 LLA**，因为当 HE 更换设备时，该地址可能会发生变化。建议仅在手动配置并在自己控制范围内的情况下，才将 LLA 用于静态路由。  
- **推荐的方式是 `-interface gif0`**，这样可以简化配置。由于 **gif0** 被配置为点对点接口，因此可以直接使用本地的接口名称来指定到 HE 端路由器的路由。这样，即使 **IPV6-SERVER** 地址发生变化，你的配置仍然有效。  

### 在基本工具中使用 IPv6  
本专栏的另一个主题是 FreeBSD 基本系统中的软件 IPv6 配置示例。获得 IPv6 互联网访问权限后，我们可以让软件开始使用 IPv6。  

#### 一般规则与注意事项  
从用户的角度来看，IPv6 最显著的区别是地址的表示法。除此之外，TCP 和 UDP 的工作方式与 IPv4 基本相同。因此，你可以先尝试将配置文件中的 IPv4 地址替换为 IPv6 地址。  

第一期专栏以 **OpenSSH** 为示例，让我们看看需要做哪些更改。  

**sshd(8) 守护进程**的配置存储在 **/etc/ssh/sshd_config** 文件中。你可以直接编辑该文件，但使用命令行参数修改配置可以保持默认的配置文件不变。例如，可以将以下行添加到 **/etc/rc.conf** 以部分修改配置：

```sh
sshd_enable="YES"
sshd_flags=" \
 -oPort=22 \
 -oUsePAM=no \
"
```

让我们回到 IPv6 配置的主题。**sshd(8)** 守护进程默认会在 IPv4 和 IPv6 上监听 **tcp/22** 端口。你可以在 **/etc/ssh/sshd_config** 中看到以下内容：  

所有这些行都被注释掉了，但这实际上意味着它们默认是启用的：

```sh
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::
```

“::” 是一个 IPv6 地址，这个全零地址被称为“未指定的 IPv6 地址”（**unspecified IPv6 address**）。需要注意的是，它同时是全零的子网前缀和全零的 IID，而不是前面章节中提到的全零 IID。这个特殊的 IPv6 地址表示“任何 IPv6 地址”，因此 **sshd(8)** 守护进程会监听此设备上配置的所有 IPv6 地址。  

像这样，一些软件在配置文件中直接使用原始的 IPv6 地址。在这种情况下，你可以直接用 IPv6 地址替换 IPv4 地址，无需额外的考虑。**sshd(8)** 足够智能，能够区分所使用的地址族（IPv4 或 IPv6）。如果你想要限制 **sshd(8)** 仅监听特定的 IPv6 地址，可以使用选项 `-oListenAddress` 。

```sh
sshd_enable="YES"
sshd_flags=" \
 -oPort=22 \
 -oUsePAM=no \
 -oListenAddress=2001:db8::1 \
"
```

尽管在配置文件中，IPv6 地址应按照 **RFC 5952**⁶ 推荐的格式书写，但几乎所有软件都接受冗余表示，例如 `2001:0db8:0000:0000:0000:0000:0000:0001`。此外，如果使用的是 LLA（链路本地地址），必须添加 `%zoneid` 部分。  

然而，一些软件不会直接使用原始的 IPv6 地址，因为这样会破坏与 IPv4 的向后兼容性。以 **syslogd(8)** 为例：

```sh
syslogd_enable="YES"
syslogd_flags="-s -cc -b [fe80::f4:a6ff:fe43:50b%epair5b]"
```

以下是作者在其 FreeBSD 设备上使用的配置之一。**syslogd(8)** 守护进程提供 `-b` 选项，用于选择监听的地址和端口号。它会监听指定地址，并接受来自 UDP 数据包的日志信息。在 IPv4 中，地址的格式为 `-b address:service`，其中 `service` 可以是端口号，也可以是 **/etc/services** 文件中列出的服务名称。因此，如果直接使用原始的 IPv6 地址，syslogd(8) 守护进程无法区分哪个冒号用于分隔“地址”和“服务”部分。  

为了解决此问题，**syslogd(8)** 守护进程采用 `[ipv6-address]:service` 这样的格式，即 IPv6 地址必须用方括号括起来。如果地址中包含 `%zoneid`，它也必须放在方括号内。这种格式在使用“地址:端口”表示法的软件中很常见。  

尽管方括号格式适用于支持它的软件，但需要注意的是，方括号在 **/bin/sh** 等 shell 程序中是元字符。实际上，上述 **syslogd(8)** 配置示例可能无法正常工作，因为方括号有时会被解释为元字符。如果字符串匹配某个文件或目录名称，一对方括号可能会被替换，导致配置意外失败。因此，推荐使用另一种可行的写法，如下所示：

```sh
syslogd_enable="YES"
syslogd_flags=" \
 -b 192.168.0.10 \
 -a 192.168.0.0/24 \
 -a [fe80::ffff:2:200%bridge0]:* \
-b [fe80::ffff:1:202%bridge0] \
 -a [fe80::%bridge0]/64 \
"
```

你会看到 `-a` 选项的参数包含 `*`，这表示守护进程接受来自任何 UDP 端口的连接。然而，`*` 可能会匹配一个或多个文件名，并在命令行解析时被替换。你可能会想到使用 `\` 转义 `*`，但这是否有效取决于 shell 变量的解析方式。因此，当软件要求使用方括号格式时，需要特别注意路径名扩展问题。当然，如果该格式出现在配置文件中，则不会有问题。  

另一个需要注意的点是如何使用 `"/64"` 这样的格式来指定前缀长度。在上述示例中，前缀长度位于方括号之外，这是因为前缀长度并不是 IPv6 地址的一部分。原始地址、zone ID 部分以及方括号的位置有时可能会令人困惑。请记住，`[fe80::/64%bridge0]` 和 `[fe80::%bridge0/64]` 这两种写法都是错误的。  

最后一个容易踩坑的地方是使用主机名指定地址的情况。通常，你可以在需要指定地址的地方直接使用主机名。主机名通常由系统可用的名称服务解析，而一个主机名可能会对应多个地址，因为 DNS 支持返回多个 A 记录和 AAAA 记录。在这种情况下，一些软件可能需要明确指定要使用的地址类型和地址族。然而，目前并没有统一的方法来指定主机名的地址族。因此，如果你打算使用主机名，最好确保该主机名唯一，并且只解析到一个地址。  

### 更多实际示例  

**在 /etc/resolv.conf 中配置 DNS 服务器**  

为简单起见，递归 DNS 服务器的配置通常应该由 RA（Router Advertisement，路由通告）消息来处理，上一期的专栏已经介绍过相关内容。然而，如果你的 DNS 服务器位于同一链路上，你可以手动添加一个 LLA，并像下面这样指定它：

```sh
nameserver fe80::ffff:1:35%bge0
```

手动配置 LLA 的一个原因是你可以在不同的网络上使用相同的地址，从而大大简化管理。  

然而，在 `/etc/resolv.conf` 中使用 LLA 并不适用于依赖 LDNS 库但不使用 FreeBSD libc 解析函数的软件。这意味着 `drill(1)` 工具无法在 `/etc/resolv.conf` 中使用 LLA，而使用 IPv6 GUA 则没有问题。此外，`dhclient(8)` 也不支持 LLA。  

作为一种变通方法，你需要在 `nameserver` 行中使用 IPv4 地址或 IPv6 GUA⁷。  

### **NFS 与 /etc/exports**  
`/etc/exports` 文件使用原始 IPv6 地址格式，并支持 `network` 关键字的前缀长度表示法。示例如下：

```sh
/a/ftproot -alldirs -maproot=0:0 -network 2001:db8:1::/64
/a/ftproot -to -alldirs -maproot=nobody:nobody -network fe80::%lagg0/10
```

请注意，NFS 并不完全支持 LLA，因为 RPC 库在处理 IPv6 地址时不会包含 zone ID。尽管第二行可以被指定，并且其语法是正确的，但实际上并不起作用⁸。IPv6 GUA 则可以正常工作。因此，目前应避免在 NFS 中使用 LLA，该问题预计会在 FreeBSD 14 中得到修复。  

### Sendmail  
Sendmail 也使用原始 IPv6 地址格式，并支持为每个地址指定地址族关键字。示例如下：

```sh
sendmail_enable="YES"
sendmail_flags="-L sm-mta -bd -q30m \
 -ODaemonPortOptions=Family=inet,address=127.0.0.1,Name=MTA \
 -ODaemonPortOptions=Family=inet6,address=::1,Name=MTA6,M=O \
 -ODaemonPortOptions=Family=inet,address=0.0.0.0,Port=587,Name=MSA,M=E \
 -ODaemonPortOptions=Family=inet6,address=::,Port=587,Name=MSA6,M=O \
 -ODaemonPortOptions=Family=inet,address=0.0.0.0,Port=465,Name=MSA,M=s \
 -ODaemonPortOptions=Family=inet6,address=::,Port=465,Name=MSA6,M=s \
"
```

你可以安全地同时使用具有单个 IPv4 和单个 IPv6 地址的主机名，因为可以使用 “Family=” 关键字进行指定。  

MSA⁹ 使用的传输配置由 **/etc/mail/FreeBSD.submit.mc** 文件中的以下行处理：

```sh
dnl If you use IPv6 only , change [127.0.0.1] to [IPv6:::1]
FEATURE(‘msp’, ‘[127.0.0.1]’)dnl
```

在这个配置文件中，sendmail 采用了一种修改后的方括号格式，因为 sendmail 需要对 IPv4 也使用方括号格式。如果要使用 IPv6，则必须使用 `[IPv6:raw-ipv6-address]` 这种格式。  

### Syslogd  
上一节已经介绍了命令行选项。在配置文件 **/etc/syslog.conf** 中，可以使用原始 IPv6 地址格式来指定远程主机。以下是一个示例，它用于从运行 ISC BIND 的 jail 环境接收日志：

```sh
+fe80::e:1ff:fec5:e80b%bridge100
!-named
*.notice;authpriv.none;kern.debug;lpr.info;mail.crit;news.err; /var/log/ns/messages
!named
*.* /var/log/ns/named.log
!*
+@
```

### **总结**  

本文介绍了如何使用隧道服务接入 IPv6 互联网，并配置 FreeBSD 基础系统中的关键软件以支持 IPv6。使用 **gif(4)** 接口进行隧道连接并不特定于 **Hurricane Electric IPv6 Tunnel Broker**，它也可以用于构建自定义的虚拟网络。然而，这种方法可能过于原始，不适用于实际用途——它不支持加密或自动配置。  

FreeBSD 提供了一整套丰富的网络实验工具。在用户态程序的 IPv6 配置中，通常只需要在配置文件中将 IPv4 地址替换为 IPv6 地址即可，但本文列出了一些需要注意的陷阱。此外，也有一些软件无法正确处理 IPv6 **LLA**（Link-Local Address）。建议你在自己的设备上尝试 IPv6，如果发现问题，请向作者或 **FreeBSD** 项目反馈。  

在下一期文章中，将会介绍更多软件配置，以及 **NDP（邻居发现协议）**，这是一项 IPv6 核心协议，你应当熟悉它的工作原理。  

---

### 附注 

1. **GUA** 代表 **Global Unicast Address**（全局单播地址），可在 IPv6 互联网中路由。  
2. 记住，IPv6 GUA 由 **前缀（prefix）** 和 **IID（Interface IDentifier，接口标识符）** 组成，IID 通常为 64 位。如果你的前缀是 48 位，你可以使用 16 位的子网编号，在 **LAN** 内创建 **65,536** 个子网；如果前缀是 64 位，你只能创建一个子网。  
3. 该协议定义在 **RFC 4213**《IPv6 主机和路由器的基本过渡机制》中。  
4. **LLA** 代表 **Link-Local Address（链路本地地址）**。在执行 `ifconfig gif0 up` 之后，系统会自动配置一个 **LLA**。详情请参考前一期文章。  
5. **IID** 代表 **Interface Identifier（接口标识符）**，用于 IPv6 地址的低位部分，每个主机的 IID 都是唯一的。长度通常设为 64 位，但理论上任何长度 **≤ 128 −（子网前缀长度）** 都是允许的。  
6. IPv6 地址的文本表示规则已在第一期文章中介绍。详情请参考 **RFC 5952**《IPv6 地址文本表示法的推荐规范》。  
7. 作者正在研究这些问题，并希望在不久的将来修复它们。  
8. 作者同样在研究此问题。  
9. **MSA** 代表 **Mail Submission Agent（邮件提交代理）**，它负责将邮件提交给 **MTA（Mail Transfer Agent，邮件传输代理）**。**sendmail** 既可以作为 **MSA**，也可以作为 **MTA**。  

---


**佐藤広生** 是东京工业大学的助理教授，研究方向包括 **晶体管级集成电路设计、模拟信号处理、嵌入式系统、计算机网络** 以及 **通用软件技术**。他曾是 **FreeBSD Core Team** 核心团队成员（2006-2022），自 2008 年起担任 **FreeBSD 基金会** 董事会成员，并自 2007 年以来一直主持 **AsiaBSDCon**——这一 BSD 操作系统在亚洲的重要国际会议。
