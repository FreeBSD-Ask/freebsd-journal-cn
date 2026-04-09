# TCP/IP 探险记：静态 Pacing

- 原文：[Static Pacing in FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-3/adventures-in-tcp-ip-static-pacing/)
- 作者：Randall Stewart、Michael Tüxen

之前的文章介绍了 FreeBSD 通过其高精度定时系统（High Precision Timing System，HPTS）支持的 pacing 机制，以及 RACK 协议栈中一种名为动态吞吐率 pacing（Dynamic Goodput Pacing，DGP）的软件 pacing 方法，该方法能根据当前网络状况动态调整到最佳速率。RACK 协议栈还提供另一种 pacing 方法，本文将介绍尚未描述的静态 pacing。

静态 pacing 结构简单，是最早添加到 RACK 协议栈中的 pacing 方法之一，主要用于测试和改进 pacing 功能（如 HPTS）。尽管最初是作为测试方法设计的，但在某些情况下，应用程序也可以使用静态 pacing。使用静态 pacing 时，pacing 速率不会由 RACK 协议栈计算，而是需要应用程序通过套接字选项（socket options）提供。因此，应用程序代码必须支持静态 pacing。

## TCP 分段发送

有三种事件会触发 TCP 分段的发送：

1. 应用程序通过 **send()** 系统调用向 TCP 协议栈提供用户数据。
2. TCP 定时器（例如重传定时器或延迟确认定时器）到期。
3. 收到一个 TCP 分段。

在这些事件中可发送的分段数量主要由两种机制控制：

1. 流量控制：保护较慢的接收方不被发送方淹没。接收方通过在发送给发送方的 TCP 分段的 **SEG.WND** 字段中通告发送方允许发送的字节数来实现流控。
2. 拥塞控制：保护较慢的网络不被发送方淹没。发送方通过计算允许发送的字节数，也就是拥塞窗口（**CWND**），来实现拥塞控制。

发送方只会发送流量控制和拥塞控制允许的字节。

拥塞控制有多种算法。FreeBSD 长期以来默认的拥塞控制算法是 New Reno。New Reno 包含两个阶段：

1. 慢启动：这是初始阶段和基于定时器重传后的阶段。在此阶段，**CWND** 指数增长。
2. 拥塞避免：这是 TCP 连接大部分时间运行的阶段。在此阶段，**CWND** 线性增长。

TCP 连接开始时，**CWND** 被设置为初始拥塞窗口，其大小由 **sysctl** 变量 **net.inet.tcp.initcwnd\_segments** 控制，默认值为 10 个 TCP 分段。

TCP 端点重传用户数据的方式有两种：一种是重传定时器到期时触发；另一种是检测到 TCP 分段丢失时进入恢复状态进行重传。

以上描述表明，在流量控制和拥塞控制允许的情况下，TCP 端点可能会发送一连串突发的 TCP 分段。

下面的 **packetdrill** 脚本演示了 FreeBSD 在 TCP 连接建立后，应用程序立刻提供 10 个 TCP 分段数据时的行为。**packetdrill** 是一种用于测试 TCP 协议栈的工具，其脚本包含应用程序的系统调用以及 TCP 协议栈发送和接收的 TCP 分段。脚本中的每行以秒为单位的时间戳开始。如果时间信息以 **+** 开头，则表示相对于前一事件的时间。例如，**+0.100** 表示事件发生在前一事件之后 100 毫秒。

```sh
–ip_version=ipv4

0.000 `kldload -n tcp_rack`
+0.000 `kldload -n cc_newreno`
+0.000 `sysctl kern.timecounter.alloweddeviation=0`

+0.000 socket(…, SOCK_STREAM, IPPROTO_TCP) = 3
+0.000 setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
+0.000 setsockopt(3, IPPROTO_TCP, TCP_FUNCTION_BLK, {function_set_name=”rack”,
                                                     pcbcnt=0}, 36) = 0
+0.000 setsockopt(3, IPPROTO_TCP, TCP_CONGESTION, “newreno”, 8) = 0
+0.000 bind(3, …, …) = 0
+0.000 listen(3, 1) = 0
+0.000 < S      0:0(0)                  win 65535 <mss 1460,sackOK,eol,eol>
+0.000 > S.     0:0(0)        ack     1 win 65535 <mss 1460,sackOK,eol,eol>
+0.050 <  .     1:1(0)        ack     1 win 65535
+0.000 accept(3, …, …) = 4
+0.000 close(3) = 0
+0.100 send(4, …, 14600, 0) = 14600
+0.000 >  .     1:1461(1460)  ack     1 win 65535
+0.000 >  .  1461:2921(1460)  ack     1 win 65535
+0.000 >  .  2921:4381(1460)  ack     1 win 65535
+0.000 >  .  4381:5841(1460)  ack     1 win 65535
+0.000 >  .  5841:7301(1460)  ack     1 win 65535
+0.000 >  .  7301:8761(1460)  ack     1 win 65535
+0.000 >  .  8761:10221(1460) ack     1 win 65535
+0.000 >  . 10221:11681(1460) ack     1 win 65535
+0.000 >  . 11681:13141(1460) ack     1 win 65535
+0.000 > P. 13141:14601(1460) ack     1 win 65535
+0.050 <  .     1:1(0)        ack  2921 win 65535
+0.000 <  .     1:1(0)        ack  5841 win 65535
+0.000 <  .     1:1(0)        ack  8761 win 65535
+0.000 <  .     1:1(0)        ack 11681 win 65535
+0.000 <  .     1:1(0)        ack 14601 win 65535
+0.000 close(4) = 0
+0.000 > F. 14601:14601(0)    ack     1 win 65535
+0.050 < F.     1:1(0)        ack 14602 win 65535
+0.000 >  . 14602:14602(0)    ack     2
```

假设往返时间（RTT）为 50 毫秒。脚本显示，调用 **send()** 会触发一次同时发送 10 个 TCP 分段的突发。

## 静态 Pacing

静态 pacing 是一种缓解流量突发性的方法。它用一系列较小的突发代替一次大的突发。较小突发的大小称为 pacing 突发大小（pacing burst size）。这些较小突发之间的时间间隔由 pacing 速率决定。

例如，发送速率为 12,000,000 比特/秒，相当于 1,500,000 字节/秒。若数据包大小为 1500 字节，则意味着每毫秒发送一个包，或者每两毫秒发送两个包，依此类推。

通过指定 pacing 速率和 pacing 突发大小，可以计算出突发之间的时间间隔，从而使发送速率达到设定的 pacing 速率。

在 RACK 协议栈中，静态 pacing 需要应用程序分别为慢启动（slow start）、拥塞避免（congestion avoidance）和恢复（recovery）状态提供单独的 pacing 速率，以及 pacing 突发大小。包大小计算时会考虑 IP 头部大小、TCP 头部大小和 TCP 载荷大小，但不考虑链路层头部和尾部大小。

静态 pacing 通过应用程序源码中使用的 **IPPROTO\_TCP** 级别套接字选项进行控制。这些套接字选项中，有三个用于控制拥塞控制算法不同状态下的 pacing 速率，有一个用于设置 pacing 突发大小，还有一个用于启用或禁用静态 pacing。

以下表格列出了这些套接字选项。

| 套接字选项名称                        | 值类型           | 说明                                                            |
| ------------------------------ | ------------- | ------------------------------------------------------------- |
| **TCP\_RACK\_PACE\_RATE\_CA**  | **uint64\_t** | 设置拥塞控制算法处于拥塞避免（congestion avoidance）阶段时的静态 pacing 速率，单位为字节每秒。 |
| **TCP\_RACK\_PACE\_RATE\_SS**  | **uint64\_t** | 设置拥塞控制算法处于慢启动（slow start）阶段时的静态 pacing 速率，单位为字节每秒。            |
| **TCP\_RACK\_PACE\_RATE\_REC** | **uint64\_t** | 设置拥塞控制算法进入恢复（recovery）阶段时的静态 pacing 速率，单位为字节每秒，用于恢复丢失的数据包。    |
| **TCP\_RACK\_PACE\_ALWAYS**    | **int**       | 启用或禁用 pacing 功能的选项。                                           |
| **TCP\_RACK\_PACE\_MAX\_SEG**  | **int**       | 设置 pacing 突发大小（pacing burst size）。                            |

需要注意以下四点：

1. 设置套接字选项（以及任何其他系统调用）可能会失败，应用程序必须检查返回结果。导致上述套接字选项设置失败的一个原因是当前套接字使用的 TCP 协议栈不是 RACK。FreeBSD 默认不支持静态 pacing。此外，如果使用 pacing 的 TCP 连接数达到系统整体限制，启用静态 pacing 也会失败。该限制由 **sysctl** 变量 **net.inet.tcp.pacing\_limit** 控制。

2. 在设置第一个 pacing 速率时，该速率不仅会应用于所指定的模式，还会同时应用于拥塞避免（congestion avoidance）、慢启动（slow start）和恢复（recovery）三种模式。

3. 仅设置 pacing 速率并不能启用静态 pacing，必须显式使用 **TCP\_RACK\_PACE\_ALWAYS** 套接字选项来启用静态 pacing。

4. 如果不设置 pacing 突发大小，则默认使用值 40。

下面的 **packetdrill** 脚本演示了在慢启动阶段，RACK 协议栈以 12,000,000 比特/秒的 pacing 速率和 pacing 突发大小为 1 进行 pacing 的情况。

```sh
–ip_version=ipv4

0.000 `kldload -n tcp_rack`
+0.000 `kldload -n cc_newreno`
+0.000 `sysctl kern.timecounter.alloweddeviation=0`

+0.000 socket(…, SOCK_STREAM, IPPROTO_TCP) = 3
+0.000 setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
+0.000 setsockopt(3, IPPROTO_TCP, TCP_FUNCTION_BLK, {function_set_name=”rack”,
                                                     pcbcnt=0}, 36) = 0
+0.000 setsockopt(3, IPPROTO_TCP, TCP_CONGESTION, “newreno”, 8) = 0
+0.000 bind(3, …, …) = 0
+0.000 listen(3, 1) = 0
+0.000 < S      0:0(0)                  win 65535 <mss 1460,sackOK,eol,eol>
+0.000 > S.     0:0(0)        ack     1 win 65535 <mss 1460,sackOK,eol,eol>
+0.050 <  .     1:1(0)        ack     1 win 65535
+0.000 accept(3, …, …) = 4
+0.000 close(3) = 0

+0.000 setsockopt(4, IPPROTO_TCP, TCP_RACK_PACE_RATE_SS, [1500000], 8) = 0 # 黄色高亮
+0.000 setsockopt(4, IPPROTO_TCP, TCP_RACK_PACE_MAX_SEG, [1], 4) = 0 # 黄色高亮
+0.000 setsockopt(4, IPPROTO_TCP, TCP_RACK_PACE_ALWAYS, [1], 4) = 0 # 黄色高亮

+0.100 send(4, …, 14600, 0) = 14600
+0.000 >  .     1:1461(1460)  ack     1 win 65535

+0.001 >  .  1461:2921(1460)  ack     1 win 65535 # 灰色部分开始
+0.001 >  .  2921:4381(1460)  ack     1 win 65535
+0.001 >  .  4381:5841(1460)  ack     1 win 65535
+0.001 >  .  5841:7301(1460)  ack     1 win 65535
+0.001 >  .  7301:8761(1460)  ack     1 win 65535
+0.001 >  .  8761:10221(1460) ack     1 win 65535
+0.001 >  . 10221:11681(1460) ack     1 win 65535
+0.001 >  . 11681:13141(1460) ack     1 win 65535
+0.001 > P. 13141:14601(1460) ack     1 win 65535
+0.042 <  .     1:1(0)        ack  2921 win 65535
+0.002 <  .     1:1(0)        ack  5841 win 65535
+0.002 <  .     1:1(0)        ack  8761 win 65535
+0.002 <  .     1:1(0)        ack 11681 win 65535
+0.002 <  .     1:1(0)        ack 14601 win 65535 # 灰色部分结束

+0.000 close(4) = 0
+0.000 > F. 14601:14601(0)    ack     1 win 65535
+0.050 < F.     1:1(0)        ack 14602 win 65535
+0.000 >  . 14602:14602(0)    ack     2
```

启用静态 pacing 所需的代码更改用黄色高亮显示，而线路上行为的相应变化用灰色标出。需要说明的是，变化仅体现在 TCP 分段的发送时序上，TCP 分段本身未发生改变。现在发送包含用户数据的 TCP 分段之间有了 1 毫秒的延迟。

以下 packetdrill 脚本展示了在慢启动阶段使用相同的 pacing 速率，但 pacing 突发大小为 4 的情况：

```
–ip_version=ipv4

0.000 `kldload -n tcp_rack`
+0.000 `kldload -n cc_newreno`
+0.000 `sysctl kern.timecounter.alloweddeviation=0`

+0.000 socket(…, SOCK_STREAM, IPPROTO_TCP) = 3
+0.000 setsockopt(3, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
+0.000 setsockopt(3, IPPROTO_TCP, TCP_FUNCTION_BLK, {function_set_name=”rack”,
                                                     pcbcnt=0}, 36) = 0
+0.000 setsockopt(3, IPPROTO_TCP, TCP_CONGESTION, “newreno”, 8) = 0
+0.000 bind(3, …, …) = 0
+0.000 listen(3, 1) = 0
+0.000 < S      0:0(0)                  win 65535 <mss 1460,sackOK,eol,eol>
+0.000 > S.     0:0(0)        ack     1 win 65535 <mss 1460,sackOK,eol,eol>
+0.050 <  .     1:1(0)        ack     1 win 65535
+0.000 accept(3, …, …) = 4
+0.000 close(3) = 0
+0.000 setsockopt(4, IPPROTO_TCP, TCP_RACK_PACE_RATE_SS, [1500000], 8) = 0

+0.000 setsockopt(4, IPPROTO_TCP, TCP_RACK_PACE_MAX_SEG, [4], 4) = 0 # 黄色部分

+0.000 setsockopt(4, IPPROTO_TCP, TCP_RACK_PACE_ALWAYS, [1], 4) = 0
+0.100 send(4, …, 14600, 0) = 14600

+0.000 >  .     1:1461(1460)  ack     1 win 65535 # 灰色部分开始
+0.000 >  .  1461:2921(1460)  ack     1 win 65535
+0.000 >  .  2921:4381(1460)  ack     1 win 65535
+0.000 >  .  4381:5841(1460)  ack     1 win 65535
+0.004 >  .  5841:7301(1460)  ack     1 win 65535
+0.000 >  .  7301:8761(1460)  ack     1 win 65535
+0.000 >  .  8761:10221(1460) ack     1 win 65535
+0.000 >  . 10221:11681(1460) ack     1 win 65535
+0.004 >  . 11681:13141(1460) ack     1 win 65535
+0.000 > P. 13141:14601(1460) ack     1 win 65535
+0.042 <  .     1:1(0)        ack  2921 win 65535
+0.000 <  .     1:1(0)        ack  5841 win 65535
+0.004 <  .     1:1(0)        ack  8761 win 65535
+0.000 <  .     1:1(0)        ack 11681 win 65535
+0.004 <  .     1:1(0)        ack 14601 win 65535 # 灰色部分结束

+0.000 close(4) = 0
+0.000 > F. 14601:14601(0)    ack     1 win 65535
+0.050 < F.     1:1(0)        ack 14602 win 65535
+0.000 >  . 14602:14602(0)    ack     2
```

正如预期，现在每四毫秒发送四个 TCP 分段。因此，灰色高亮的 TCP 分段的时序受到了影响。

## 其他注意事项

上一个例子中，12Mbps 的 pacing 速率将应用于所有拥塞控制状态，突发大小限制为四个分段，最终启用 pacing。也就是说，RACK 堆栈会发送四个分段，等待四毫秒，然后再发送四个分段，循环往复，直到所有数据包发送完毕。

需要注意的是，RACK 堆栈不会为了等待整整四个分段而阻塞发送机会。如果套接字缓冲区中可用的分段少于四个，堆栈会立即发送现有的分段，并启动一个调整后的 pacing 定时器，使该突发按照请求的速率间隔发送。另一个重要因素是拥塞控制和流控；如果堆栈达到拥塞或流控限制，pacing 速率可能会低于设定速率。因此，RACK 堆栈有时可能只发送一、二或三个 TCP 分段，这并非因为用户数据不足，而是受到拥塞或流控的限制。静态 pacing 可能导致吞吐量降低，因为它无法在竞争流量消失后“补偿”超出设定速率的带宽。

使用静态 pacing 的开发者还需考虑至少四个交互因素。其中之一是与比例速率减少（Proportional Rate Reduction，PRR）的交互。PRR 在 RACK 进入恢复状态时触发，限制发送量大致为每收到一个入站确认就发送一个分段。这意味着数据发送既受 pacing 定时控制，也受对端确认的影响。在多数静态 pacing 应用场景中，这种交互是不可取的。因此，存在一个 **IPPROTO\_TCP** 级别的套接字选项 **TCP\_NO\_PRR** 用于禁用 PRR，该选项的值类型为 **int**。

另一个恢复机制是快速恢复（rapid recovery）；该功能允许 RACK 根据丢包和传输时间更快恢复，而不仅仅依赖三个重复确认。但这可能改变恢复期间的数据发送速率，从而影响预期的 pacing 行为。可以使用 **IPPROTO\_TCP** 级别的套接字选项 **TCP\_RACK\_RR\_CONF** 来调整此行为。该选项允许的整数值为 0、1、2 和 3，默认值为 0，表示 RACK 可完全控制快速恢复。不同值提供调用者对恢复行为的微调。对于静态 pacing，建议将此值设置为 3，以确保仅使用指定速率。

启用 TCP 分段卸载（TSO）时，设置 pacing 突发大小会影响 CPU 负载。使用较小的突发大小会增加 CPU 负载。例如，发送 40 个 TCP 分段时，pacing 突发大小为 40 只需一次 TSO 操作，而设置为 4 则需进行 10 次 TSO 操作。

设置 pacing 突发大小时还需考虑确认延迟。大多数 TCP 堆栈启用此功能，即等待两个（或更多）分段或延迟确认计时器超时（通常 40 至 200 毫秒之间，规范建议 200 毫秒，但许多系统缩短了此时间）后才发送确认。因此，如果将 pacing 突发大小设置为 1 而非 4，可能导致确认延迟计时器与 pacing 计时器交互，从而大幅降低实际 pacing 速率。为避免此类问题，建议 pacing 突发大小不要小于 2。

---

**RANDALL STEWART**（[rrs@freebsd.org](mailto:rrs@freebsd.org)）是一位操作系统开发者，拥有 40 余年经验，自 2006 年起成为 FreeBSD 开发者，专注于 TCP 和 SCTP 等传输协议，也涉猎操作系统其他领域，目前是独立顾问。

**MICHAEL TÜXEN**（[tuexen@freebsd.org](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-3/adventures-in-tcp-ip-static-pacing/)）是明斯特应用科技大学教授，Netflix 兼职承包商，自 2009 年起为 FreeBSD 源代码贡献者，专注于 SCTP 和 TCP 传输协议、IETF 标准化及其在 FreeBSD 中的实现。
