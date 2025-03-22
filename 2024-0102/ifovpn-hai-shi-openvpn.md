# if\_ovpn 还是 OpenVPN

- 原文链接：[if_ovpn or OpenVPN](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/if_ovpn-or-openvpn/)
- 作者：Kristof Provost

今天①，你将了解② OpenVPN 的 DCO（数据通道卸载）功能。

OpenVPN 最初由 James Yonan 开发，首次发布于 2001 年 5 月 13 日。它支持许多常见平台（如 FreeBSD、OpenBSD、Dragonfly、AIX 等）以及一些较为罕见的平台（如 macOS、Linux、Windows）（**译者注：原文如此**）。它支持点对点和客户端 - 服务器模型，并可基于预共享密钥、证书和用户名/密码进行认证。

正如你所期望的，所有存在 20 余年的项目都会在各种的使用场景下不断增长许多功能。

## 问题

尽管 OpenVPN 异常出色，但它也存在明显的问题。没有问题，这篇文章就不会那么有趣③。的确，有毛病，那就是 OpenVPN 单线程实现的用户空间进程。

OpenVPN 使用 `if_tun` 向网络栈注入数据包。因此，它的性能跟不上现今的连接速度。同时，这也使得它难以利用现代多核硬件和加密卸载硬件。

![OpenVPN 性能图](https://freebsdfoundation.org/wp-content/uploads/2024/02/provost_fig1.png)

OpenVPN 的主要性能问题在于它的用户空间特性。进入的流量自然会被网卡接收，网卡通常会将数据包通过 DMA 传送到内核内存。然后，这些数据包会继续被网络栈处理，直到网络栈确定数据包属于哪个套接字，并将其传到用户空间。这个套接字可能是 UDP/TCP。

将数据包传递到用户空间涉及到复制数据包，此时用户空间的 OpenVPN 进程会验证、解密数据包，然后通过 `if_tun` 将其重新注入到网络栈中。这意味着需要将明文数据包复制回内核进行进一步处理。

无可避免地，这种上下文切换和数据包来回复制对性能产生了明显的影响。

在当前架构下，想要显著提高性能非常困难。

## DCO 是什么

既然我们已经明确了问题所在，就可以开始想想解决方案了④。

如果把问题锁定在上下文切换到用户空间，那么有一种可行的解决方案就是将工作保持在内核中，这就是 DCO（数据通道卸载）所做的。

![DCO 架构图](https://freebsdfoundation.org/wp-content/uploads/2024/02/provost_fig2.png)

DCO 将数据通道（即加密操作和流量隧道化）移动到内核中。它使用一款新的虚拟设备驱动程序 `if_ovpn` 实现。OpenVPN 用户空间进程仍然负责连接的建立（包括认证和选项协商），并通过一个新的 ioctl 接口与 `if_ovpn` 驱动程序进行协调。

OpenVPN 项目认为，引入 DCO 是一次极好的机会：能删除一些遗留功能，进行一些清理。在这方面，他们采取了 Henry Ford 在加密算法选择上的方法——你可以选择任何加密算法，只要你选择 AES-GCM 和 ChaCha20/Poly1305，且都使用黑色（即未压缩）。此外，DCO 还不支持压缩、第二层流量、非子网拓扑和流量整形⑤。

值得注意的是，DCO 并不会改变 OpenVPN 协议。客户端可以与不支持 DCO 的服务器一起使用，反之亦然。当然，当双方都使用 DCO 时，你能获得最大的好处，但这不是强制要求的。

## 考虑事项

在这一部分，我将讨论这一切有多么困难，以便让你们对我能够成功实现这一功能留下深刻印象。如果我告诉你我正在这么做，这个方式还有效吗？让我们试试看吧！

不管怎样，有几个方面需要特别关注：

## 多路复用

第一个问题是 OpenVPN 使用单一连接来传输隧道数据和控制数据。隧道数据需要由内核处理，而控制数据则由 OpenVPN 用户空间进程处理。

你能看到这个问题。套接字最初是由 OpenVPN 自身完全拥有的。它设置隧道并处理认证。完成后，它会部分地将控制权交给内核端（即 `if_ovpn`）。

这意味着需要通知 `if_ovpn` 文件描述符（内核用来查找内核中的 `struct socket`），以便它可以持有该引用。这样可以确保套接字在内核使用时不会消失，可能因为 OpenVPN 进程被终止，或因为它遇到问题决定捣乱。毕竟它是在用户空间，可能会做一些疯狂的事情。

对于那些想要跟踪内核代码的朋友，你可以关注函数 [`ovpn_new_peer()`](https://cgit.freebsd.org/src/tree/sys/net/if_ovpn.c?id=da69782bf06645f38852a8b23af#n1483)⑥。

通过查找套接字后，我们现在还可以通过 [`udp_set_kernel_tunneling()`](https://cgit.freebsd.org/src/tree/sys/net/if_ovpn.c?id=da69782bf06645f38852a8b23af#n1483) 安装过滤函数。该过滤器 [`ovpn_udp_input()`](https://cgit.freebsd.org/src/tree/sys/net/if_ovpn.c?id=da69782bf06645f38852a8b23af#n1483) 检查所有传入的数据包，决定它是有效负载数据包，应该由内核处理，还是控制数据包，应该由 OpenVPN 用户空间进程处理。

这个隧道功能也是我对其余网络栈做出的唯一更改。需要让它知道某些数据包应该由内核处理，而其他数据包仍然可以传递到用户空间。相关修改已在[这个提交](https://cgit.freebsd.org/src/commit/?id=742e7210d00b359d81b9c778ab520003704e9b6c)中完成。

函数 `ovpn_udp_input()` 是接收路径的主要入口点。网络栈会将到达该套接字的所有 UDP 数据包交给此函数处理。

函数首先检查数据包是否能由内核驱动处理。也就是说，数据包是数据包并且目标是已知的对等体。如果不是这种情况，过滤函数会告诉 UDP 代码按正常流程传递数据包，仿佛没有过滤函数一样。这样，数据包就会到达套接字，并由 OpenVPN 的用户空间进程进行处理。

早期版本的 DCO 驱动有单独的命令 ioctl 来读取和写入控制消息，但 Linux 和 FreeBSD 的驱动都已适配为使用套接字。这简化了控制数据包和新客户端的处理。

另一方面，如果数据包是已知对等体的数据包，它会被解密，验证其签名，然后传递给网络栈进行进一步处理。

对于那些跟踪代码的朋友，相关处理代码在[这里](https://cgit.freebsd.org/src/tree/sys/net/if_ovpn.c?id=da69782bf06645f38852a8b23af#n1483)。

## UDP 

OpenVPN 能通过 UDP 和 TCP 运行。虽然 UDP 是层 3 VPN 协议的优选，但某些用户需要通过 TCP 运行它以便穿越防火墙。

FreeBSD 内核提供了一项方便的 UDP 套接字过滤功能，但 TCP 没有类似的功能，因此 FreeBSD 的 `if_ovpn` 当前仅支持 UDP，不支持 TCP。

Linux 的 DCO 驱动开发者则更加激进，选择实现了对 TCP 的支持。开发者虽然面临重重挑战，但最终还是成功地完成了这一任务，现在他的经验大为增加。

## 硬件加密卸载

`if_ovpn` 依赖于内核中的 OpenCrypto 框架进行加密操作。这意味着它还可以利用系统中存在的任何加密卸载硬件，从而进一步提高性能。

它已经通过英特尔的 QuickAssist 技术（QAT）、SafeXcel EIP-97 加密加速器和 AES-NI 进行过测试。

## 锁设计

看，如果你以为你可以在不讨论锁的情况下阅读内核代码，我真不知道该怎么告诉你。那的确是过于天真的乐观了。

几乎每款现代 CPU 都有多个核心，而能够使用不止一个核心自然是很有意义的。也就是说，我们不能在一个核心执行工作时，锁住其他核心。这不太礼貌，性能表现也不会太好。

幸运的是，这个问题的解决办法相对简单。整个方法基于区分对 `if_ovpn` 内部数据结构的读取和写入操作。也就是说，我们能让多个核心同时查找数据，但每次只有一个核心可以修改数据（并且在修改时不允许有其他核心读取）。这种方法足够有效，因为——大部分时间——我们并不需要修改数据。

常见的情况是，当我们接收或发送数据包时，只需要查找密钥、目标地址、端口和其他相关信息。

仅在我们修改数据时（即在配置更改或重新密钥时），我们才需要获取写锁，并暂停数据通道。这个过程足够短促，我们这些微不足道的人类大脑几乎不会察觉到，这也让大家都感到高兴。

有一个例外是“我们不会修改处理数据”这个规则，那就是数据包计数器。每个数据包都会被计数（甚至两次，一次是数据包计数，另一次是字节计数），这必须并发进行。幸运的是，内核的框架 [`counter(9)`](https://www.freebsd.org/cgi/man.cgi?query=counter&sektion=9) 正好为这种情况而设计。它为每个 CPU 核心保留总计，以确保一个核心不会影响或拖慢其他核心。仅在读取计数器时，内核才会向每个核心请求其总计并将它们加总。

## 控制接口

每款平台的 OpenVPN DCO 都有自己独特的方式来实现用户空间 OpenVPN 和内核模块之间的通信。

在 Linux 上，通过 netlink 完成，但 `if_ovpn` 的工作在 FreeBSD 的 netlink 实现完成之前就已经完成。由于我仍然在为上次的因果关系违规而接受观察期，我决定使用其他方法。

通过现有的 ioctl 接口路径对 `if_ovpn` 驱动进行配置。具体来说，就是调用 [`SIOCSDRVSPEC/SIOCGDRVSPEC`](https://www.freebsd.org/cgi/man.cgi?query=ioctl&sektion=2)。

这些调用将一个 `ifdrv` 结构传递给内核。字段 `ifd_cmd` 用于传递命令，而字段 `ifd_data` 和 `ifd_len` 用于在内核和用户空间之间传递特定于设备的结构。

在某种程度上，`if_ovpn` 偏离了传统的方法，它传输的是序列化的 nvlists，而非结构体。这使得扩展接口更加容易。或者说，它意味着我们可以在不破坏现有用户空间消费者的情况下扩展接口。如果在结构体中添加一个新字段，它的布局会发生变化，这要么意味着现有代码由于大小不匹配⑦拒绝接受它，要么就会感到非常困惑，因为字段的意义不再是原来的那样。

序列化的 nvlists 允许我们添加字段，而不会混淆另一方。任何未知的字段将被忽略。这使得添加新功能变得更加容易。

## 路由查找

你可能认为 `if_ovpn` 不需要担心路由决策。毕竟，内核的网络栈在数据包到达网络驱动程序时已经做出了路由决策。你错了。我本来想取笑你，但我也花了一些时间才弄明白这一点。

问题在于，在一个给定的接口 `if_ovpn` 上，可能存在多个对等体（例如，当它作为服务器并有多个客户端时）。内核已经确定数据包需要发送给其中的某个对等体，但内核假设这些客户端都在同一个广播域中。也就是说，发送到该接口的数据包将对所有客户端可见。然而，在这里并非如此，因此 `if_ovpn` 需要确定数据包应该发送给哪个客户端。

这通过函数 [`ovpn_route_peer()`](https://cgit.freebsd.org/src/tree/sys/net/if_ovpn.c?id=da69782bf06645f38852a8b23af#n1483) 处理。该函数首先查看对等体列表，检查是否有对等体的 VPN 地址与目标地址匹配（通过 [`ovpn_find_peer_by_ip()`](https://cgit.freebsd.org/src/tree/sys/net/if_ovpn.c?id=da69782bf06645f38852a8b23af#n1483) 和 [`ovpn_find_peer_by_ip6()`](https://cgit.freebsd.org/src/tree/sys/net/if_ovpn.c?id=da69782bf06645f38852a8b23af#n1483)，具体取决于地址族）。如果找到匹配的对等体，数据包就会发送给该对等体。如果没有找到，`ovpn_route_peer()` 会执行一次路由查找，并用结果网关地址再次进行对等体查找。

只有当 `if_ovpn` 确定了数据包应该发送给哪个对等体时，数据包才会被加密并传输。

## 密钥轮换

OpenVPN 会定期更换用于加密隧道的密钥。这是一个由 `if_ovpn` 留给用户空间来处理的难题，因此 OpenVPN 和 `if_ovpn` 之间需要进行一些协调。

OpenVPN 会通过 OVPN_NEW_KEY 命令安装新密钥。每个密钥都有一个 ID，且每个数据包中都包含用于加密它的密钥 ID。这意味着，在密钥轮换期间，所有数据包仍然可以解密，因为旧的和新的密钥都会在内核中保持活动状态。

若新密钥安装完成，它可以通过命令 OVPN_SWAP_KEYS 使其生效。也就是说，新的密钥将用于加密即将发送的数据包。

稍后，旧密钥可以通过命令 OVPN_DEL_KEY 删除。

## vnet

是的，我们不得不谈一下 vnet。既然是我写的，避不开这个话题。

我懒得从头解释 vnet，所以我只给你推荐一篇更好的文章，作者是 Olivier Cochard-Labbé：“Jail: vnet by examples”⑧。

你可以把 vnet 看作是将 Jail 转变为拥有自己 IP 栈的虚拟机。

这在 pfSense 的使用场景下不是严格要求的，但它使测试变得更简单得多。这样，我们可以在单台机器上进行测试，而不需要任何外部工具（除了 OpenVPN 本身，这点应该很明显）。

对于有兴趣了解具体操作的人，还有另一篇可能有用的 [FreeBSD Journal](https://freebsdfoundation.org/our-work/journal/) 文章：《The Automated Testing Framework》⑨，哦等等，我好像知道那个人，Kristof Provost。

## 性能

在经历了这一切之后，我敢打赌你在问自己：“这真的有好处吗？”

幸运的是，对我来说：是的，确实有好处。

我在 Netgate 的一位同事花了一些时间用 iperf3 对 Netgate 4100⑩ 设备进行测试，并得到了以下结果：

| 测试类型        | 吞吐量      |
|:-----------------:|:-------------:|
| if_tun          | 207.3 Mbit/s |
| DCO Software    | 213.1 Mbit/s |
| DCO AES-NI      | 751.2 Mbit/s |
| DCO QAT         | 1,064.8 Mbit/s |

“if_tun”是旧的 OpenVPN 方法，无 DCO。值得注意的是，它在用户空间使用了 AES-NI 指令，而 "DCO Software" 设置则没有。尽管有明显的作弊行为，DCO 仍然略快。在公平的条件下（即 DCO 确实使用 AES-NI 指令），差距更是显而易见，DCO 的速度是原来的三倍多。

对于英特尔来说，还有好消息：他们的 QuickAssist 卸载引擎比 AES-NI 更快，使得 OpenVPN 的速度是以前的五倍。

## 后续工作

没有什么是完美的，总有改进的空间，但在某些方面，接下来的这一增强功能正是 DCO 设计成功的结果。OpenVPN 的协议使用了 32 位初始化向量（IV），出于加密原因，我在这里不再解释⑪，重复使用相同密钥的 IV 是不安全的。

这意味着必须在达到这一点之前重新协商密钥。OpenVPN 的默认重新协商间隔是 3600 秒，假设有 30% 的安全余地，这将相当于 2^32 * 0.7 / 3600，即每秒约 835,000 个数据包。这个速度大约是 8 到 9 Gbit/s（假设 1300 字节的数据包）。

对于 DCO 来说，这已经差不多可以被现代硬件所支持了。

虽然这是个好问题，但依然是个问题，因此 OpenVPN 开发人员正在开发一种更新的包格式，使用 64 位 IV。

## 致谢

`if_ovpn` 的工作得到了 Rubicon Communications（以 Netgate 名义运营）的资助，用于 pfSense 产品线。从 22.05 版本的 pfSense Plus 发布⑫以来，它一直在该平台上使用。此工作已上游合并到 FreeBSD，并成为最近的 14.0 版本的一部分。使用该功能需要 OpenVPN 2.6.0 及更高版本。

我还要感谢 OpenVPN 的开发者们，在初始的 FreeBSD 补丁出现时，他们非常热情，还提供了帮助，没有他们的协助，这个项目不会像现在这样顺利进行。

## 脚注

1. 或者是你阅读这篇文章的时候。
2. 好吧，写作。阅读。看，如果你要对这点吹毛求疵，我们会一直讨论下去。
3. 看，如果你对 DCO 不感兴趣，你可以直接去阅读下一篇文章。我敢肯定那篇文章会很不错。
4. 我说“我们”，但尽管我很想为这个解决方案归功于自己，实际上是 OpenVPN 的开发者们设计了 DCO 架构并实现了 Windows 和 Linux 的版本。我做的只是他们做的事情，但为 FreeBSD 做的。
5. 在 OpenVPN 中，DCO 可以与操作系统的流量整形（即 dummynet）结合使用。
6. [查看代码：](https://cgit.freebsd.org/src/tree/sys/net/if_ovpn.c?id=da69782bf06645f38852a8b23af#n490)
7. 你也许可以说因为结构体变得臃肿了。你可能会这么说，但我太客气，不这么说。
8. [Jail: vnet by examples](https://freebsdfoundation.org/wp-content/uploads/2020/03/Jail-vnet-by-Examples.pdf)
9. [The Automated Testing Framework](https://freebsdfoundation.org/wp-content/uploads/2019/05/The-Automated-Testing-Framework.pdf)
10. [Netgate 4100](https://shop.netgate.com/products/4100-base-pfsense)*
11. （主要因为我自己也不懂它们。）
12. [pfSense Plus 软件版本 22.05 已发布](https://www.netgate.com/blog/pfsense-plus-software-version-22.05-now-available)

---

**Kristof Provost** 是一名自由职业的嵌入式软件工程师，专注于网络和视频应用。他是 FreeBSD 的提交者，负责维护 FreeBSD 中的 pf 防火墙。目前，他大部分时间都在为 Netgate 的 pfSense 项目工作。

Kristof 总是不小心碰到 uClibc 的 bug，而且对 FTP 恨之入骨。千万不要和他谈 IPv6 分片的问题。
