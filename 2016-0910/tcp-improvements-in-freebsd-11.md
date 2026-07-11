# FreeBSD 11 中的 TCP 改进

- 原文：[TCP Improvements in FreeBSD 11](https://freebsdfoundation.org/wp-content/uploads/2016/10/TCP-Improvements-in-FreeBSD-11.pdf)
- 作者：**Hiren Panchasara**

阻碍我们试验 TCP 的主要障碍之一是对未知的恐惧。由于 TCP 本身复杂，且代码过去形式上非模块化，很难隔离一处修复而不影响其余代码。然而引入细微的回归却相对容易。

TCP 备选栈框架试图解决这一问题，将栈模块化，使任何人都能提出新的 TCP 栈，并让两个（或更多）栈同时共存于同一台机器上，服务不同的连接。代码经过模块化，不同的 TCP 操作（如输入或输出处理、段处理、定时器处理和套接字选项解析）细分为不同的函数块，并通过函数指针访问。因此，开发者可以设计采用新方式处理输入或输出的备选栈，同时仍使用默认栈的其余功能。可以通过套接字选项为每个连接选择栈，这使得在试验各种 TCP 特性时能进行真正的 A/B 测试。

## TCP 备选栈框架

以下 sysctl 可用于配置：

- `net.inet.tcp.functions_available` - 列出所有可用的栈
- `net.inet.tcp.functions_default` - 设置/获取默认栈

下面是简短示例，服务端将 `mytcp` 设为监听套接字上使用的栈。它接受的每个新连接都会以 `mytcp` 作为其 TCP 栈。

```c
socklen_t slen;
char *stack_name="mytcp";
struct tcp_function_set fsn;
int error;

memset(&fsn, 0, sizeof(fsn));
strcpy(fsn.function_set_name, stack_name);
slen = sizeof(fsn);
error = setsockopt(s, IPPROTO_TCP, TCP_FUNCTION_BLK, &fsn, slen);
if (error)
    printf("Could not set TCP stack to %s, error:%d\n", func_blk, errno);
```

## TCPPCAP——一项调试特性

有时 TCP 问题以奇怪的方式出现，我们希望访问最后几个数据包或 TCP 状态转换，以便弄清楚出了什么问题。该特性通过在相应的 TCP 控制块中保存最后可配置数量的数据包来满足这一需求。数据包数量可以全局指定（即按系统），也可以通过套接字选项按连接指定。

需要提醒的是：虽然该特性能很好地限制可分配给它的最大内存量，但选择数量时应谨慎，因为保存数据包会消耗内存。

该特性可以通过内核配置中的 TCPPCAP 选项启用。即便如此，该特性默认未激活。激活后，TCP 控制块会包含 `t_inpkts`（保存的输入数据包列表）和 `t_outpkts`（保存的输出数据包列表）。

以下 sysctl 可用于配置：

- `net.inet.tcp.tcp_pcap_packets` - 启用该特性并设置每个连接在每个方向上捕获的数据包数量
- `net.inet.tcp.tcp_pcap_clusters_referenced_max` - 允许引用的最大 mbuf cluster 数量

以下 sysctl 可用于监控该特性的内存使用：

- `net.inet.tcp.tcp_pcap_clusters_referenced_cur` - 引用的 mbuf cluster 总数
- `net.inet.tcp.tcp_pcap_alloc_reuse_ext` - 复用的带外部存储的 mbuf 总数
- `net.inet.tcp.tcp_pcap_alloc_reuse_mbuf` - 复用的带内部存储的 mbuf 总数
- `net.inet.tcp.tcp_pcap_alloc_new_mbuf` - 新分配的 mbuf 总数

## 服务端 TCP Fast Open（TFO）

TCP 在客户端和服务器之间执行三次握手（3-WHS：SYN、SYN-ACK、ACK）后，连接上才能传输实际数据。对于短生命周期的连接，这可能占事务总耗时的很大一部分，即客户端经历的延迟。

TCP Fast Open（TFO）由 RFC 7413 引入，通过允许 SYN 和 SYN-ACK 数据包携带数据来解决这一问题，前提是客户端持有有效的安全 cookie，服务器可据此认证客户端。

FreeBSD 现在支持该特性的服务端。TFO 的主要部分是安全 cookie。客户端首先通过发送带有 FAST OPEN 选项和空 cookie 字段的 SYN 来请求 cookie。服务器生成 cookie，并通过 SYN-ACK 数据包的 FAST OPEN 选项发回。客户端缓存该 cookie，用于与同一服务器的 FAST OPEN 连接。客户端拿到 cookie 后，发送带有数据和缓存 cookie 的 SYN，cookie 放在 FAST OPEN 选项字段中。如果 cookie 有效，服务器同时确认 cookie 和数据，并将数据发送给应用程序。如果 cookie 无效，服务器丢弃数据，只确认 SYN。如果服务器接受 SYN 中提供的数据，可以在 3-WHS 完成前发送响应数据。客户端同时确认 SYN 和数据（如果存在），否则只确认服务器的 SYN。连接的其余部分按常规 TCP 连接进行。

客户端可以发起 TFO 连接，只要 cookie 仍然有效；其有效期可在服务端配置。因此，如果客户端向同一服务器发起大量短生命周期连接，该特性对降低整体延迟大有帮助。

该特性可通过在内核配置中添加 `options TCP_RFC7413` 启用。可在内核配置中通过 `options TCP_RFC7413_MAX_KEYS=<num-keys>` 指定使用多少个并发密钥。目前默认为 2，配置 sysctl 如下：

- `net.inet.tcp.fastopen.enabled` - 启用/禁用该特性
- `net.inet.tcp.fastopen.autokey` - 自动密钥更新超时；默认 120 秒
- `net.inet.tcp.fastopen.acceptany` - 接受客户端提供的任意密钥的 cookie；默认禁用
- `net.inet.tcp.fastopen.keylen` - 密钥长度（字节）
- `net.inet.tcp.fastopen.maxkeys` - 支持的最大密钥数
- `net.inet.tcp.fastopen.numkeys` - 已安装的密钥总数

## 数据中心 TCP

数据中心 TCP（DCTCP）是用于数据中心的拥塞控制机制。它使用显式拥塞通知（ECN）来估计拥塞的程度/规模，而不仅仅检测是否发生拥塞。

传统拥塞控制机制在数据中心内表现不佳，因为流量本质上是高吞吐、低延迟且突发的。由于 ECN 能估计拥塞的严重程度，DCTCP 据此相应地减小拥塞窗口，有助于高效利用可用带宽。

DCTCP 现作为拥塞控制模块纳入，可作为所有连接的默认拥塞控制，也可通过套接字选项按连接设置。当前实现也能以单边方式工作，即只有一端（发送方或接收方）使用 DCTCP。

使用以下 sysctl 配置该特性：

- `net.inet.tcp.cc.dctcp.slowstart` - 决定是否在第一次慢启动后将拥塞窗口减半
- `net.inet.tcp.cc.dctcp.shift_g` - 决定收敛时间和带宽利用率。IETF 草案建议保持为‘4’，我们在实现中遵循此值
- `net.inet.tcp.cc.dctcp.alpha` - 决定在连接初期你希望 DCTCP 多激进，代价是排队延迟。IETF 草案建议用值‘1’不那么激进，但我们决定更激进一些，FreeBSD 中默认为‘0’

## 用于黑洞检测的打包层路径 MTU 发现

最大传输单元（MTU）是一帧中可发送的最大数据量。如果路径上某设备的出口接口 MTU 小于帧大小，且设置了“不分片”（DF）位，帧将被丢弃，并向发送方回送一条 ICMP 消息“需要分片但 DF 已设置”，其中包含出口接口的 MTU 大小。收到此消息后，发送方可调整 MTU 并重发帧。

有时这些 ICMP 消息因防火墙或其他神秘原因被阻挡，数据包被静默丢弃，服务器毫不知情。这称为 PMTU 黑洞。在这种情况下，无需 ICMP 即能检测黑洞非常有用，RFC 4821 引入的打包层路径 MTU 发现（PLPMTUD）黑洞检测正是为此而设。

当发送数据包两次因重传超时（RTO）失败时，它会尝试检测连接路径上是否遭遇黑洞。方法是将最大段大小（MSS）降至预设值。如果连续发送失败，则继续降低 MSS，直到达到配置的最小 MSS 值。如果使用较低的 MSS 能发送数据包，说明路径上确实存在黑洞。但如果即便使用配置的最小 MSS 值仍无法发送数据包，则不是黑洞，而是真正的拥塞导致丢包，此时将 MSS 恢复为原始值。

该特性的另一个重要部分是在成功传输后，以递增的 MSS 值向上探测网络。然而这部分目前在 FreeBSD 中尚未实现，因此该特性默认禁用。

在许多拥塞控制机制中，拥塞窗口的增长取决于 MSS 大小，因此遭遇黑洞的连接可能经历拥塞窗口增长缓慢，从而利用路径容量更慢。

使用以下 sysctl 配置该特性：

- `net.inet.tcp.pmtud_blackhole_detection` - 启用/禁用
- `net.inet.tcp.pmtud_blackhole_mss` - IPv4 降低后的 MSS
- `net.inet.tcp.v6pmtud_blackhole_mss` - IPv6 降低后的 MSS

使用以下 sysctl 监控该特性：

- `net.inet.tcp.pmtud_blackhole_activated` - 检测到黑洞并激活 MSS 下探的次数
- `net.inet.tcp.pmtud_blackhole_activated_min_mss` - 检测到黑洞并在 MSS 下探过程中降至最小 MSS 的次数
- `net.inet.tcp.pmtud_blackhole_failed` - 错误检测到黑洞的次数

## 短生命周期 TCP 连接的可扩展性工作

网络栈的分层设计及其实现意味着复杂的锁和序列化挑战。每层都要与相邻层通信以保持同步，从而接受、服务和拆除连接。甚至在每层内部，视其功能而定，当大量连接共享相同的数据结构和内存资源时，也必须有适当的同步来保持一致性。

struct inpcb 是 IP 层协议控制块结构，捕获 TCP、UDP 等的网络层状态，如主机地址和路由信息。

在此短生命周期连接的改进之前，近 80% 的接收数据包在持有独占写锁的情况下处理。这意味着只有一个 CPU 核心能处理接收数据包，其余核心必须等待锁释放。现在新增了 INP_LIST 锁，保护全局 inpcb 列表的修改。这允许在输入处理等关键路径上使用 INP_INFO_RLOCK（读锁），只在真正需要时偶尔使用 INP_INFO_WLOCK（写锁），即在稳定时遍历全局 inpcb 列表。

每秒 TCP 连接（建立和拆除）的最大数量从 60k 提升到 150k。

## 改进对盲窗口内 SYN/RST 伪造攻击的防护

如果攻击者以某种方式猜出有效的序列号窗口并伪造 SYN 或 RST 数据包，FreeBSD 过去会直接断开连接，而不检查该数据包是否正是当前所期望的。

现在借助 RFC 5961 提出的增强，如果传入的 RST 的序列号正是我们期望的，则断开连接；如果不在精确位置但在窗口内，则发送挑战 ACK。对于传入的 SYN，如果连接处于同步状态，则发送挑战 ACK。

发送挑战 ACK 会使真正的发送方生成与我们期望的精确序列号匹配的 RST。基本上，有了这一机制，攻击者很难伪造能让我们断开连接的有效 SYN/RST。某些中间设备或损坏的 TCP 栈可能仍以 RST 数据包响应挑战 ACK，但该数据包的 SEQ 号不是我们所期望的。在这种情况下，由于该特性，我们可能看到大量虚假的挑战 ACK 和 RST 来回发送。应对挑战 ACK 进行限流以缓解此问题，使我们在每个时间段内只发送有限数量的挑战 ACK。例如，5 秒内只发送 10 个挑战 ACK。允许的时间和 ACK 数量都应是可调参数。这在 FreeBSD 中尚未实现。

该特性在 FreeBSD 上默认启用。可通过将 `net.inet.tcp.insecure_rst` 和 `net.inet.tcp.insecure_syn` 设为‘1’来禁用。

## 借助 DTrace 探针点实现类似 TCPDEBUG 的信息

TCP 有一项调试功能，可以打印连接状态变更的轨迹以及发送和接收数据包等其他有用事件。但要启用该特性，需要用内核配置文件中的 `TCPDEBUG` 选项构建内核，且关联的套接字需要标记 `SO_DEBUG` 选项——这意味着要重新编译应用程序。

现在，通过 DTrace（一种动态跟踪框架）探针点可实现相同功能，无需重新编译内核或用户空间应用程序。**$src/share/dtrace/tcpdebug** 是示例脚本，也是如何使用该特性的好例子。

---

**HIREN PANCHASARA** 在内核代码中摸爬滚打了几年，最近更多专注于网络和 TCP 领域。他在一切可能的设备上运行 FreeBSD，并希望通过指导等方式将越来越多的新手带到 FreeBSD。
