# 用 DTrace 探索网络活动

- 原文：[Exploring Network Activity with DTrace](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking/exploring-network-activity-with-dtrace/)
- 作者：**Mark Johnston**

FreeBSD 提供了大量工具和技巧，用于回答关于网络栈活动的问题。`systat(1)` 和 `netstat(1)` 这类工具能给出若干起点：例如它们可以按接口或按协议展示包速率和位速率。更高级的工具会利用伯克利包过滤器（Berkeley Packet Filter，BPF）来追踪进出系统的单个数据包；Ports 中的 `sysutils/iftop` 就用这种技术按四元组展示位速率。老牌的 `tcpdump(1)` 也通过 BPF 接口实时捕获并记录数据包，从而支持基于事后分析的提问。

FreeBSD 10 起，内核新增了一组 DTrace 探测器，让用户能深入洞察网络栈的内部工作。具体而言，用户现在可以基于 FreeBSD 内核 IP、TCP 和 UDP 层中的数据包发送和接收事件编写脚本，并实时窥探指定连接的内部 TCP 状态。这对任何程序员或系统管理员的工具箱都是一项有力的补充，因为它提供框架来回答关于 FreeBSD IP 栈行为的任意问题；DTrace 不受既有工具输出的局限，使得网络提供商可以编写自己的工具来探索网络活动，无论是为了监控性能指标、定位问题源头，还是单纯为了更深入地了解网络协议。

本文将概述每个新探测器，讲解其用法并给出示例。本文假定读者对 DTrace 有基本了解并熟悉其使用，但本文中的所有示例要么是 `dtrace(1)` 命令——可直接在 shell 中运行——要么是可执行脚本。这些示例均在 FreeBSD 10 上开发并测试，鼓励有可用测试系统的读者动手运行，以体会 DTrace 的能力。脚本可从 <http://people.freebsd.org/~markj/dtrace/network-providers/examples/> 下载。

需要提醒的是，这些探测器的 FreeBSD 实现相对较新，因此在你尝试时自然可能遇到 bug 或难以解释的行为。DTrace 保证脚本不会让系统崩溃或破坏其状态，所以在 FreeBSD 上运行这些示例或任何 DTrace 脚本都没有危险。但是，如果你在使用这些新探测器或 DTrace 时遇到问题，请发送邮件至 <freebsd-dtrace@FreeBSD.org> 邮件列表报告。不针对 FreeBSD 的 DTrace 相关问题和讨论请发送至 <dtracediscuss@lists.dtrace.org> 邮件列表；DTrace 的多位原始及现任开发者都订阅了该列表，并乐于回应该列表上的帖子。

## FreeBSD 上的 DTrace

熟悉 FreeBSD DTrace 实现的用户可能知道 `fbt` provider，它用于追踪 FreeBSD 内核中发生的函数调用。该 provider 对熟悉 FreeBSD 内核内部的用户而言极为便利，但有几个缺点：

- 它的使用要求对内核代码有相当程度的了解，这对希望编写自己脚本的大多数用户来说是一道不低的门槛。
- `fbt` 探测器按定义与内核代码紧密耦合；如果脚本所基于的代码发生改动，脚本可能无法运行或行为异常。因此为某个 FreeBSD 版本编写的脚本未必能在另一版本上工作，几乎可以肯定无法在其他操作系统上工作。
- 单个 `fbt` 探测器往往不能与逻辑系统事件很好对应。假设你想编写 DTrace 脚本，打印 FreeBSD 交给网卡驱动的每个 IP 数据包的目的地址。事实证明这是一项令人望而生畏的任务：它至少需要在 IPv4 和 IPv6 代码的不同部分埋点四个不同函数，每个函数的调用参数都不同。

新的网络探测器让用户能够编写脚本，并使用不受上述问题困扰的接口来追踪网络相关事件。它们提供了稳定而简洁的窗口来窥探内核活动：要追踪 IP 数据包的目的地址，只需运行：

示例 1：

```sh
# dtrace -n 'ip:::send {printf("%s", args[2]->ip_daddr);}'
```

用 `fbt` provider 实现完全相同的功能至少需要写五十行难以理解的 D 代码——这是作者在勉强尝试这道练习后给出的估算。相比之下，这条示例相当直观透明，除了有些神秘的 `args[2]`。用通俗的话说，它的意思是“每次我们发送 IP 数据包，就打印其目的地址”。在我的笔记本上运行数秒，同时 ping 一个互联网地址，得到如下输出：

```sh
# dtrace -n 'ip:::send {printf("%s", args[2]->ip_daddr);}'
dtrace: description 'ip:::send ' matched 1 probe
CPU ID FUNCTION:NAME
0 36564 :send 8.8.178.110
0 36564 :send 8.8.178.110
0 36564 :send 8.8.178.110
0 36564 :send 8.8.178.110
```

这是 DTrace 的默认输出格式。我们可以在 `dtrace(1)` 参数中加入 `-x quiet`，并自行打印换行符（`\n`），以获得更多控制：

```sh
# dtrace -x quiet -n 'ip:::send {printf("%s\n", args[2]->ip_daddr);}'
8.8.178.110
8.8.178.110
8.8.178.110
8.8.178.110
```

在 D 脚本中，可通过在文件开头加入下面这行达到相同效果：

```sh
#pragma D option quiet
```

## 新的网络探测器

本文讨论的网络探测器起源于 Solaris，在 illumos 和 OS X 中也存在。它们属于新的 `ip`、`tcp` 和 `udp` DTrace provider；下面的示例 2 命令将在你的系统上列出这些探测器：

示例 2：

```sh
# dtrace -l -P ip -P tcp -P udp
ID PROVIDER MODULE FUNCTION NAME
36492 ip kernel receive
36493 ip kernel send
36494 tcp kernel accept-established
36495 tcp kernel accept-refused
36496 tcp kernel connect-established
36497 tcp kernel connect-refused
36498 tcp kernel connect-request
36499 tcp kernel receive
36500 tcp kernel send
36501 tcp kernel state-change
36502 udp kernel receive
36503 udp kernel send
```

可以看出，FreeBSD 现在为 IP、TCP 和 UDP 的数据包发送和接收事件都提供了探测器。IP 发送和接收探测器（即 `ip:::send` 和 `ip:::receive`）在 FreeBSD 发送或接收 IPv4 或 IPv6 数据包时触发。类似地，TCP 和 UDP 的发送和接收探测器在内核发送或接收 TCP 或 UDP 数据包时触发。可以看到，`tcp` provider 还包含对应 TCP 协议事件的额外探测器；目前我们先关注发送和接收探测器，并构建几个示例。

每个网络探测器都接受若干参数，这些参数共同描述了触发该探测器的数据包。参数本身各自是相关信息的集合。例如，`ip:::send` 的第三个参数（即前面示例中的 `args[2]`）包含 IPv4 与 IPv6 共有的高层 IP 字段：IP 版本（`ip_ver`）、负载长度（`ip_plength`）以及源地址和目的地址（`ip_saddr` 和 `ip_daddr`）。第四个参数包含描述用于传输该数据包的网络接口的信息，第五和第六个参数分别给出该数据包的详细 IPv4 和 IPv6 字段。也就是说，如果该数据包使用 IPv4，`args[4]` 会暴露其 IPv4 头部字段；而 IPv6 数据包的字段则通过 `args[5]` 暴露。`ip:::send` 探测器参数的完整列表与描述见 [1]，此处不再赘述；`tcp` 和 `udp` provider 的对应页面分别在 [2] 和 [3]。利用发送和接收探测器，我们可以为每个数据包实时打印输出。在负载较重的系统上，这显然不切实际；此时用户会希望使用 DTrace 的数据聚合功能，或加入谓词以有选择地追踪数据包。

下面的脚本在每个 TCP 数据包进出系统时打印其基本信息。注意，被转发的 TCP 数据包不会触发 `tcp` 探测器，因为 TCP 代码并不会处理它们。

示例 3 用到三个探测器。`dtrace:::BEGIN` 探测器用于为 `tcp` 探测器动作的输出打印列头。如列头所示，`tcp` 探测器为每个 TCP 数据包打印与之关联的四元组，以及 TCP 负载大小和 TCP 标志。一些样本输出展示了经过 NAT 的 SSH、HTTP 和 IMAPS 流量：

示例 3：

```sh
#!/usr/sbin/dtrace -s
#pragma D option quiet
#pragma D option switchrate=10Hz
dtrace:::BEGIN
{
printf(" %30s %-6s %30s %-6s %-6s %s\n\n", "SADDR", "SPORT",
"DADDR", "DPORT", "BYTES", "FLAGS");
}
tcp:::receive,
tcp:::send
{
printf(" %30s %-6u %30s %-6u %-6u (%s%s%s%s%s%s\b)\n",
args[2]->ip_saddr, args[4]->tcp_sport,
args[2]->ip_daddr, args[4]->tcp_dport,
args[2]->ip_plength - args[4]->tcp_offset,
(args[4]->tcp_flags & TH_FIN) ? "FIN|" : "",
(args[4]->tcp_flags & TH_SYN) ? "SYN|" : "",
(args[4]->tcp_flags & TH_RST) ? "RST|" : "",
(args[4]->tcp_flags & TH_PUSH) ? "PSH|" : "",
(args[4]->tcp_flags & TH_ACK) ? "ACK|" : "",
(args[4]->tcp_flags & TH_URG) ? "URG|" : "");
}
```

```sh
SADDR                           SPORT DADDR                           DPORT BYTES FLAGS
fe80:3::fa1a:67ff:fe03:f659     22    fe80:3::250:b6ff:fe0e:a825      42705 36    (PSH|ACK)
fe80:3::fa1a:67ff:fe03:f659     22    fe80:3::250:b6ff:fe0e:a825      42705 628   (PSH|ACK)
fe80:3::250:b6ff:fe0e:a825      42705 fe80:3::fa1a:67ff:fe03:f659     22    0     (ACK)
fe80:3::fa1a:67ff:fe03:f659     22    fe80:3::250:b6ff:fe0e:a825      42705 100   (PSH|ACK)
fe80:3::250:b6ff:fe0e:a825      42705 fe80:3::fa1a:67ff:fe03:f659     22    0     (ACK)
192.168.0.27                    41116 173.194.76.108                  993   37    (PSH|ACK)
173.194.76.108                  993   192.168.0.27                    41116 20    (ACK)
173.194.76.108                  993   192.168.0.27                    41116 63    (PSH|ACK)
192.168.0.27                    41116 173.194.76.108                  993   0     (ACK)
173.252.102.241                 443   192.168.0.27                    50220 429   (PSH|ACK)
192.168.0.27                    50220 173.252.102.241                 443   911   (PSH|ACK)
173.252.102.241                 443   192.168.0.27                    50220 20    (ACK)
192.168.0.27                    16039 31.13.69.160                    443   37    (PSH|ACK)
31.13.69.160                    443   192.168.0.27                    16039 20    (ACK)
```

`tcp` 探测器的动作是一次 `printf()` 调用，用到了 `tcp:::send` 和 `tcp:::receive` 的第三个和第五个参数；这些参数包含相应数据包的 IP 和 TCP 头部，便于取出与该数据包关联的四元组。TCP 负载大小的计算略麻烦：IP 头部中含 IP 负载大小，TCP 头部中含从 TCP 头部起点到 TCP 负载的偏移量，二者之差即为 TCP 负载大小。注意，若出站接口启用了 TSO，DTrace 报告的出站段的 TCP 负载大小可能大于该连接的 MSS。最后，`args[4]->tcp_flags` 含该段的 TCP 标志，DTrace TCP 库（在 FreeBSD 上位于 **/usr/lib/dtrace/tcp.d**）为每个 TCP 标志提供了符号名。

示例 4 演示了简单用法，使用线性分布打印每个接口的 IP 负载大小直方图。该脚本运行期间不向终端打印任何内容，而是持续采集数据，并在退出时打印摘要——退出可以由用户按 Ctrl-C 或脚本调用内置的 `exit()` 函数触发。本例中脚本会一直运行，直到用户结束它：

示例 4：

```sh
#!/usr/sbin/dtrace -s
#pragma D option quiet
#pragma D option switchrate=10Hz
ip:::send,
ip:::receive
{
@num[args[3]->if_name] = lquantize(args[2]->ip_plength, 0, 1500, 100);
}
```

在只有单一活动接口（本例中为 `wlan0`）的系统上，样本输出如下。多接口系统会为每个接口各打印一份直方图。

```sh
wlan0
value————————- Distribution ————————- count
< 0 0
0 |@@@@@@@@@@@@@@@@@@@@@@@ 479
100 |@@@ 61
200 |@ 16
300 |@ 24
400 |@ 27
500 |@ 14
600 | 5
700 | 10
800 | 5
900 | 9
1000 | 5
1100 | 5
1200 |@ 11
1300 |@@@@@@@ 152
1400 | 0
```

再看一个例子，我们用示例 5 中的 `tcp:::state-change` 探测器展示 TCP 连接持续时长的分布。该探测器在状态切换发生时给出切换前后的状态，因此我们可以通过记录时间戳来度量时长：

- 当连接从 CLOSED 转为 SYN-SENT 时；或
- 当连接从 LISTEN 转为 SYN-RECEIVED 时。

随后我们认为：当连接进入 FIN-WAIT2、CLOSING 或 LAST-ACK 状态时，该连接便结束；此事件发生后，我们记录两个事件之间的时间差。为存储初始连接时间戳，我们使用以 `args[1]->cs_cid` 为索引的数组，`cs_cid` 是不透明的整数，唯一标识一条连接。也就是说，我们可以假设多个同时存在的连接不会共享同一连接 ID。完整脚本如下：

示例 5：

```sh
#!/usr/sbin/dtrace -s
#pragma D option quiet
#pragma D option switchrate=10Hz
tcp:::state-change
/(args[5]->tcps_state == TCPS_CLOSED && args[3]->tcps_state == TCPS_SYN_SENT) ||
(args[5]->tcps_state == TCPS_LISTEN && args[3]->tcps_state == TCPS_SYN_RECEIVED)/
{
dur[args[1]->cs_cid] = timestamp;
}
tcp:::state-change
/(args[3]->tcps_state == TCPS_CLOSING ||
args[3]->tcps_state == TCPS_FIN_WAIT_2 ||
args[3]->tcps_state == TCPS_LAST_ACK) &&
dur[args[1]->cs_cid] != 0/
{
@["Connection duration (ms)"] = quantize((timestamp - dur[args[1]->cs_cid]) / 1000000);
dur[args[1]->cs_cid] = 0;
}
```

在 `tcp:::state-change` 探测器中，`args[5]->tcps_state` 给出切换前的状态，`args[3]->tcps_state` 给出切换后的状态。注意第二个探测器通过检查 `dur[args[1]->cs_cid]` 是否非零来确认时间戳是否存在：这能确保我们不会为脚本启动时已经存在的连接记录数据。

到目前为止的示例或许已让你相信，网络 provider 是构建网络工具的良好基础——无论这些工具是用于调查某个特定问题，还是用于追踪现有监控工具难以获取的数据。不过，前面看到的大多数示例原则上都可以用基于 BPF 拦截和检查数据包的自定义程序重新实现。但 TCP 探测器让我们能窥见相应 TCP 连接的内部状态，这是 BPF 无法做到的。这让我们可以，例如，在连接发生时监控其 TCP 状态转换：

示例 6

```sh
#!/usr/sbin/dtrace -s
#pragma D option quiet
#pragma D option switchrate=10Hz
dtrace:::BEGIN
{
printf(" %30s %-6s %30s %-6s %-9s\n", "LADDR", "LPORT",
"RADDR", "RPORT", "DELTA(us)");
}
int last[uint64_t];
tcp:::state-change
{
this->delta = last[args[1]->cs_cid] != 0 ?
(timestamp - last[args[1]->cs_cid]) / 1000 : 0;
last[args[1]->cs_cid] = timestamp;
printf(" %30s %-6u %30s %-6u %-9u %s -> %s\n",
args[3]->tcps_laddr, args[3]->tcps_lport,
args[3]->tcps_raddr, args[3]->tcps_rport,
this->delta,
tcp_state_string[args[5]->tcps_state],
tcp_state_string[args[3]->tcps_state]);
}
tcp:::state-change
/args[3]->tcps_state == TCPS_CLOSING ||
args[3]->tcps_state == TCPS_FIN_WAIT_2 ||
args[3]->tcps_state == TCPS_LAST_ACK/
{
last[args[1]->cs_cid] = 0;
}
```

示例 6 的脚本在以连接 ID（`args[1]->cs_cid`）为索引的 `last` 数组中记录每条连接上一次状态切换的时间戳。每次发生状态转换时，打印与该连接关联的四元组、自上次状态转换以来经过的时间，以及转换本身（例如“state-established -> state-close-wait”）。

`tcp_state_string` 数组定义在 **/usr/lib/dtrace/tcp.d** 中，为每个 TCP 状态提供字符串表示。状态的符号名也可用，例如 `TCPS_TIME_WAIT`、`TCPS_LAST_ACK`。

注意，当 TCP 连接结束时——即进入 FIN-WAIT-2、CLOSING 或 LAST-ACK 状态——我们会把 `last` 数组中的对应条目置零。此时内核中的连接状态即将被拆除，因此若该 CID 被未来某条连接复用，该数组条目将不再有效。

## 高级技巧

前面提到过，可以按进程追踪网络活动。未来有望通过探测器参数直接取得关联进程的 PID，但目前由于相关数据在内核中的组织方式尚无法做到。不过，借助 `fbt` provider，仍可以把 PID 或进程名与 TCP 和 UDP 探测器中通过 `args[1]->cs_cid` 取得的连接 ID 关联起来。完成此事的 D 代码有些晦涩，但已包含在下面的示例中，可在其他脚本中复用。

下面的示例 7 脚本每两秒打印一次按进程汇总的 TCP 活动，报告 TCP 上发送和接收的总字节数，并按进程和四元组细分：

示例 7：

```sh
#!/usr/sbin/dtrace -s
#pragma D option quiet
#pragma D option switchrate=10hz
dtrace:::BEGIN
{
in = 0;
out = 0;
}
fbt::tcp_usr_attach:entry
{
self->so = args[0];
}
fbt::tcp_usr_attach:return
/args[1] == 0 && self->so != NULL/
{
procs[(uintptr_t)self->so->so_pcb] = execname;
self->so = NULL;
}
fbt::sosend_generic:entry, fbt::soreceive:entry
/args[0]->so_proto->pr_protocol == IPPROTO_TCP/
{
procs[(uintptr_t)args[0]->so_pcb] = execname;
}
fbt::in_pcbdetach:entry
{
procs[(uintptr_t)args[0]] = 0;
}
tcp:::send
/procs[args[1]->cs_cid] != ""/
{
this->bytes = args[2]->ip_plength - args[4]->tcp_offset;
out += this->bytes;
@bytes[procs[args[1]->cs_cid], args[2]->ip_saddr, args[4]->tcp_sport,
args[2]->ip_daddr, args[4]->tcp_dport] = sum(this->bytes);
}
tcp:::receive
/procs[args[1]->cs_cid] != ""/
{
this->bytes = args[2]->ip_plength - args[4]->tcp_offset;
in += this->bytes;
@bytes[procs[args[1]->cs_cid], args[2]->ip_daddr, args[4]->tcp_dport,
args[2]->ip_saddr, args[4]->tcp_sport] = sum(this->bytes);
}
profile:::tick-2sec
{
out /= 1024;
in /= 1024;
printf("%Y, TCP in: %6dKB, TCP out: %6dKB, TCP total: %6dKB\n", walltimestamp,
in, out, in + out);
printf("%-12s %-15s %5s %-15s %5s %9s\n", "PROC", "LADDR", "LPORT",
"RADDR", "RPORT", "SIZE");
printa("%-12s %-15s %5d %-15s %5d %@9d\n", @bytes);
printf("\n");
trunc(@bytes);
in = 0;
out = 0;
}
```

该脚本由若干片段构成。`dtrace:::BEGIN` 探测器用于初始化一对全局变量，分别用于统计当前时间窗口内发送和接收的 TCP 负载字节数；这两个变量由 `profile:::tick-2sec` 探测器每两秒重置一次，该探测器还每两秒向终端打印一次汇总数据。四个 `fbt` 探测器用于填充和清理 `procs` 数组，该数组把连接 ID 映射到进程名（例如 `firefox` 或 `sshd`）。具体而言，`fbt::tcp_usr_attach` 探测器在创建新的 TCP 套接字时触发；`fbt::sosend_generic` 和 `fbt::soreceive` 探测器在进程通过 TCP 发送或接收数据时触发；`fbt::in_pcbdetach` 探测器在 TCP 连接关闭时触发。

脚本余下部分执行实际的按进程统计。每当 TCP 栈发送或接收对应 `procs` 数组中某条目的数据包时，`in` 和 `out` 全局变量便相应递增，`bytes` 数组也相应更新。该数组以进程名和连接四元组为索引，其内容在 `profile:::tick-2sec` 探测器中打印并清空。

这使得我们可以深入到特定进程的用量层面，而这一任务在没有 DTrace 的情况下相当困难。当然，示例 7 的脚本可以改造为执行不同类型的按进程或按用户统计。例如，可以保留运行累计值，而不是每两秒清空统计；当进程退出时（由 `proc:::exit` 探测器指示），可将其 TCP 总用量保存以备后续分析。此外，稍作改动，示例 7 也可改为追踪 UDP 而非 TCP，如示例 8 所示。

示例 8：

```sh
#!/usr/sbin/dtrace -s
#pragma D option quiet
#pragma D option switchrate=10hz
dtrace:::BEGIN
{
in = 0;
out = 0;
}
fbt::udp_attach:entry
{
self->so = args[0];
}
fbt::udp_attach:return
/args[1] == 0 && self->so != NULL/
{
procs[(uintptr_t)self->so->so_pcb] = execname;
self->so = NULL;
}
fbt::sosend_dgram:entry, fbt::soreceive:entry
/args[0]->so_proto->pr_protocol == IPPROTO_UDP/
{
procs[(uintptr_t)args[0]->so_pcb] = execname;
}
fbt::in_pcbdetach:entry
{
procs[(uintptr_t)args[0]] = 0;
}
udp:::send
/procs[args[1]->cs_cid] != ""/
{
/* 减去 UDP 头部长度 */
this->bytes = args[4]->udp_length - 8;
out += this->bytes;
@bytes[procs[args[1]->cs_cid], args[2]->ip_saddr, args[4]->udp_sport,
args[2]->ip_daddr, args[4]->udp_dport] = sum(this->bytes);
}
udp:::receive
/procs[args[1]->cs_cid] != ""/
{
/* 减去 UDP 头部长度 */
this->bytes = args[4]->udp_length - 8;
in += this->bytes;
@bytes[procs[args[1]->cs_cid], args[2]->ip_daddr, args[4]->udp_dport,
args[2]->ip_saddr, args[4]->udp_sport] = sum(this->bytes);
}
profile:::tick-2sec
{
out /= 1024;
in /= 1024;
printf("%Y, UDP in: %6dKB, UDP out: %6dKB, UDP total: %6dKB\n",
walltimestamp, in, out, in + out);
printf("%-12s %-15s %5s %-15s %5s %9s\n",
"PROC", "LADDR", "LPORT", "RADDR", "RPORT", "SIZE");
printa("%-12s %-15s %5d %-15s %5d %@9d\n", @bytes);
printf("\n");
trunc(@bytes);
in = 0;
out = 0;
}
```

作为最后一个示例，我们演示如何用 DTrace 深入 TCP 协议的一些较为高级的方面。这里我们主要使用每个 TCP 探测器的第四个参数（`args[3]`）。该参数包含描述连接内部状态的信息，对于研究仅凭单个数据包头部难以观察的现象很有用。

下面我们用 TCP provider 检测乱序段的到达；具体而言，该脚本检查入站的数据包，其序列号与下一个预期序列号（通过 `args[3]->tcps_rnxt` 取得）不符的情况。如果 FreeBSD 看到这样的段，且重组队列有空间，便将其加入队列；否则将其丢弃。乱序段可能是丢包或多路径的结果；一般而言它们会拖累吞吐量，若在一条连接中占比偏高，应当排查。

示例 9 的脚本按远端主机地址统计乱序段，并计算总字节数和包数以便对比：

示例 9：

```sh
#!/usr/sbin/dtrace -s
#pragma D option quiet
#pragma D option switchrate=10Hz
tcp:::receive
/args[3]->tcps_state == TCPS_ESTABLISHED &&
args[2]->ip_plength - args[4]->tcp_offset > 0 &&
args[3]->tcps_rnxt != args[4]->tcp_seq/
{
@invorderb[args[3]->tcps_raddr] = sum(args[2]->ip_plength - args[4]->tcp_offset);
@invorderp[args[3]->tcps_raddr] = count();
}
tcp:::receive
/args[3]->tcps_state == TCPS_ESTABLISHED &&
args[2]->ip_plength - args[4]->tcp_offset > 0/
{
@valorderb[args[3]->tcps_raddr] = sum(args[2]->ip_plength - args[4]->tcp_offset);
@valorderp[args[3]->tcps_raddr] = count();
}
dtrace:::END
{
printf("%-30s %-12s %-12s %-12s %-12s\n", "RADDR", "BYTES",
"OOO BYTES", "PACKETS", "OOO PACKETS");
printa("%-30s %@-12d %@-12d %@-12d %@-12d\n", @valorderb, @invorderb,
@valorderp, @invorderp);
}
```

## 延伸阅读

本文旨在介绍 FreeBSD 10 中可用的网络 provider，并让读者体会这些 provider 适合解决的问题类型。完整探测器集的参考资料见 [1-3]，Brendan Gregg 与 Jim Mauro 合著的 DTrace 一书 [4] 也有介绍。

- [1] <https://wikis.oracle.com/display/DTrace/ip+Provider>
- [2] <https://wikis.oracle.com/display/DTrace/tcp+Provider>
- [3] <https://wikis.oracle.com/display/DTrace/udp+Provider>
- [4] DTrace: Dynamic Tracing in Oracle Solaris, Mac OS X, and FreeBSD.
- [5] <https://people.freebsd.org/~markj/dtrace/network-providers/examples/>

Mark Johnston 是软件工程师，居住在安大略省滑铁卢。他于 2013 年在滑铁卢大学取得数学学士学位，自 2010 年起成为 FreeBSD 用户。他对操作系统开发的各个方面都感兴趣，尤其关注调试与性能分析工具。获得 commit 权限后，他的主要工作重心是改进 FreeBSD 的 DTrace 实现。可通过电子邮件 <markj@FreeBSD.org> 与他联系。
