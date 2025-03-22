# TCP/IP 历险记：TCP BBLog


- 原文链接：[Adventures in TCP/IP: TCP Black Box Logging](https://freebsdfoundation.org/adventures-in-tcp-ip-tcp-black-box-logging/)
- 作者：RANDALL STEWART、MICHAEL TÜXEN

## FreeBSD 中 TCP 日志记录的演变

4.2 BSD 发布于 1983 年，内置了 BSD 中第一款 TCP 实现。4.2 BSD 还添加了调试 TCP 实现的功能。内核部分由内核参数 `TCP_DEBUG`（默认禁用）控制，提供了一个包含 `TCP_NDEBUG`（默认 100）个元素的全局环形缓冲区，并在发送或接收 TCP 段、TCP 计时器到期或处理与 TCP 协议相关的用户请求时，将条目添加到环形缓冲区。仅当启用 `SOL_SOCKET` 级别套接字参数 `SO_DEBUG` 时，才会为套接字添加这些事件。4.2 BSD 还提供了命令行工具 `trpt`（transliterate protocol trace），可以从活动系统和核心文件中读取环形缓冲区并将其打印出来。它不仅打印发送和接收的 TCP 段的 TCP 头部，还在发送或接收 TCP 段、TCP 计时器到期和处理 TCP 协议相关的用户请求时打印 TCP 端点的最重要参数。值得注意的是，在系统崩溃的情况下，环形缓冲区的内容可能提供了充足的信息来分析系统进入错误状态的原因。然而，由于此功能不再符合当前 TCP 的使用需求，它在 FreeBSD 14 中被移除。在早期版本的 FreeBSD 中，构建非默认配置的内核是必需的。

2010 年，内核模块 `siftr`（TCP 统计信息收集模块）被添加到了 FreeBSD 中。无需对 FreeBSD 内核进行更改，只需加载该模块即可使用它。仅通过 `sysctl` 变量控制 `siftr`。启用时（由 `sysctl` 变量 `net.inet.siftr.enabled` 控制），`siftr` 将输出写入一个文件，文件路径由 `sysctl` 变量 `net.inet.siftr.logfile` 控制（默认 `/var/log/siftr.log`）。除第一个和最后一个条目外，所有条目对应于发送和接收的 TCP 段，并提供方向、IP 地址、TCP 端口号和内部 TCP 状态的信息。由于设想它与类似 `tcpdump` 的数据包捕获工具配合使用，因此不会存储有关 TCP 段的额外信息（例如 TCP 头部）。每个 TCP 连接的每 n 个 TCP 段将分别记录在发送和接收方向中，n 由 `sysctl` 变量 `net.inet.siftr.ppl` 控制。可以通过 `sysctl` 变量 `net.inet.siftr.port_filter` 的 TCP 端口过滤器来关注特定的 TCP 连接。所有信息均以 ASCII 格式存储，因此无需额外的用户态工具即可访问这些信息。在默认配置中，仅支持 TCP/IPv4。添加对 TCP/IPv6 的支持需要重新编译内核模块 `siftr`。

2015 年，内核中添加了由内核参数 `TCP_PCAP`（默认禁用）控制的功能。若在非默认内核上启用，每个 TCP 端点将包含两个环形缓冲区：一个用于发送的 TCP 段，一个用于接收的 TCP 段。需注意的是，不会存储任何附加信息，甚至不会记录发送和接收 TCP 段的时间。每个环形缓冲区中 TCP 段的最大数量由 `IPPROTO_TCP` 级别的套接字选项 `TCP_PCAP_OUT` 和 `TCP_PCAP_IN` 控制。默认值由 `sysctl` 变量 `net.inet.tcp.tcp_pcap_packets` 控制。由于没有用户态工具来提取环形缓冲区的内容，此功能的使用仅限于分析核心文件。还需注意，TCP 负载也会被记录，因此共享包含此类信息的核心文件可能存在隐私问题。此功能计划在即将发布的 FreeBSD 15 中移除。

最新的 TCP 日志记录功能，即 TCP BBLog（TCP black box logging，黑匣子日志记录）添加于 2018 年。最初它被称为 TCP BBR（black box recorder，黑匣子记录器），但为避免与 TCP 拥塞控制（BBR，bottleneck bandwidth and round trip propagation time，即瓶颈带宽和往返传播时间）混淆，现在称为 BBLog。在所有 64 位平台的 FreeBSD 生产版本中均已启用 BBLog。BBLog 结合了 `TCP_DEBUG` 和 `TCP_PCAP` 的优点却没有二者的缺点。因此，它旨在取代二者。BBLog 可以通过 `sysctl` 接口和套接字 API 控制，具体见本文后续部分。

## BBLog 简介

BBLog 由内核参数 `TCP_BLACKBOX`（在所有 64 位平台上默认启用）控制，源代码在 `sys/netinet/tcp_log_buf.c`，对应的头文件是 `sys/netinet/tcp_log_buf.h`。在启用了 BBLog 的内核上，有一个设备（`/dev/tcp_log`），用于向用户态工具提供 BBLog 信息，每个 TCP 端点包含一个 BBLog 事件列表。

每个事件包含一组标准的 TCP 状态信息以及（可选的）事件特定数据块。这些事件收集到设定的限制，当达到限制时，这些事件可以发送到 `/dev/tcp_log`，若该设备被打开，则将信息转发给读取进程进行记录。请注意，如果没有进程打开该设备，则数据会被丢弃。

可以通过 FreeBSD ports 中的 `tcplog_dumper` 从 `/dev/tcp_log` 读取信息，具体操作如本文所述。

所有 FreeBSD TCP 栈都最少支持以下事件类型：

- `TCP_LOG_IN`——在收到 TCP 段时生成。
- `TCP_LOG_OUT`——在发送 TCP 段时生成。
- `TCP_RTO`——在计时器到期时生成。
- `TCP_LOG_PRU`——当 PRU 事件调用到栈时生成。

TCP RACK 和 BBR 栈生成许多其他日志；`netinet/tcp_log_buf.h` 中目前定义了 72 种事件类型。这些日志记录了多种条件，且 TCP BBR 和 RACK 栈在调试时还支持详细模式。这些详细选项可通过特定栈的 `sysctl` 变量 `net.inet.tcp.rack.misc.verbose` 和 `net.inet.tcp.bbr.bb_verbose` 进行设置。

每个 TCP 端点可以处于以下 BBLog 状态之一：

- `TCP_LOG_STATE_OFF (0)`——BBLog 已禁用。
- `TCP_LOG_STATE_TAIL (1)`——仅记录连接上的最后一些事件。每个连接分配有限数量（默认 5000 个）日志条目。当到达最后一个条目时，将重新使用第一个条目并覆盖其内容。
- `TCP_LOG_STATE_HEAD (2)`——仅记录连接上处理的最初一些事件，直到达到上限。
- `TCP_LOG_STATE_HEAD_AUTO (3)`——记录连接上处理的最初事件，当达到上限时，将数据输出到日志转储系统以供收集。
- `TCP_LOG_STATE_CONTINUAL (4)`——记录所有事件，当达到最大收集事件数时，将数据发送到日志转储系统并开始分配新事件。
- `TCP_LOG_STATE_TAIL_AUTO (5)`——记录连接尾部的所有事件，当达到上限时，将数据发送到日志转储系统。

注意，对于一般调试，通常使用 BBLog 状态 `TCP_LOG_STATE_CONTINUAL`。但在某些特定情况下（如调试系统崩溃），优先使用 BBLog 状态 `TCP_LOG_STATE_TAIL`，以便在崩溃转储中记录最后的 BBLog 事件。

BBLog 状态可以在 TCP 连接建立时设置，也可以通过套接字 API 设置。此外，还可以在 TCP 连接满足特定条件时设置状态。这称为跟踪点，指定给特定的 TCP 栈并通过编号标识。一个跟踪点的示例是，当 TCP 栈调用 IP 输出例程时遇到 `ENOBUF` 错误。

每个事件的内容包括三部分：

1. BBLog 头，包含 TCP 连接的 IP 地址和 TCP 端口号、事件时间、标识符、原因和标签。
2. 一组 TCP 连接的强制性状态变量，包括 TCP 连接状态和各种序列号变量。
3. 一组可选数据，如发送和接收缓冲区占用情况、TCP 头信息以及其他事件特定信息。

注意，BBLog 事件中不包含 TCP 负载信息，但每个 BBLog 事件中均包含 IP 地址和 TCP 端口号信息。

## 配置 BBLog

配置 BBLog 主要有两种方式：通用配置通过 `sysctl` 接口进行，针对特定 TCP 连接的配置通过套接字 API 进行。

### 通过 `sysctl` 接口的通用配置

以下是与 BBLog 相关的 `sysctl` 变量列表，均在 `net.inet.tcp.bb` 下：

```sh
[rrs]$ sysctl net.inet.tcp.bb
net.inet.tcp.bb.pcb_ids_tot: 0
net.inet.tcp.bb.pcb_ids_cur: 0
net.inet.tcp.bb.log_auto_all: 1
net.inet.tcp.bb.log_auto_mode: 4
net.inet.tcp.bb.log_auto_ratio: 1
net.inet.tcp.bb.disable_all: 0
net.inet.tcp.bb.log_version: 9
net.inet.tcp.bb.log_id_tcpcb_entries: 0
net.inet.tcp.bb.log_id_tcpcb_limit: 0
net.inet.tcp.bb.log_id_entries: 0
net.inet.tcp.bb.log_id_limit: 0
net.inet.tcp.bb.log_global_entries: 5016
net.inet.tcp.bb.log_global_limit: 5000000
net.inet.tcp.bb.log_session_limit: 5000
net.inet.tcp.bb.log_verbose: 0
net.inet.tcp.bb.tp.count: 0
net.inet.tcp.bb.tp.bbmode: 4
net.inet.tcp.bb.tp.number: 0
```

通过 `sysctl` 接口，可以为 TCP 连接启用 BBLog。关键的 `sysctl` 变量是 `net.inet.tcp.bb.log_auto_all`、`net.inet.tcp.bb.log_auto_mode` 和 `net.inet.tcp.bb.log_auto_ratio`。

第一个需要考虑的 `sysctl` 变量是 `net.inet.tcp.bb.log_auto_all`。如果将此变量设置为 1，则所有连接将纳入 BBLog ratio 范围内。如果设置为 0，则只有具有 `TCP_LOGID`（见下文）的连接才会应用 BBLog ratio。在大多数情况下，当使用 `sysctl` 方法启用 BBLog 时，应用程序可能没有设置 `TCP_LOGID`，因此将 `net.inet.tcp.bb.log_auto_all` 设置为 1 可确保所有连接都将被纳入考虑范围。

接下来要设置的 `sysctl` 变量是 `net.inet.tcp.bb.log_auto_ratio`。此值确定 n 分之一（n 由 `net.inet.tcp.bb.log_auto_ratio` 设置的值决定）的连接将启用 BBLog。例如，如果将 `net.inet.tcp.bb.log_auto_ratio` 设置为 100，则每 100 个连接中就会有 1 个启用 BBLog。如果需要为每个连接启用 BBLog，应将 `net.inet.tcp.bb.log_auto_ratio` 设置为 1。

最后需要考虑的 `sysctl` 变量是 `net.inet.tcp.bb.log_auto_mode`。此值是 BBLog 状态的数值常量。对于 TCP 开发，默认值可以设置为 4，即 `TCP_LOG_STATE_CONTINUAL`，用于记录所有连接生成的每个事件，以便进行调试。

`sysctl` 变量中的其他一些项目也可能有用。例如，`net.inet.tcp.bb.log_session_limit` 控制一个连接可以收集的 BBLog 事件数量上限，达到此上限后需要对数据进行处理（例如，发送到收集系统或覆盖事件）。`net.inet.tcp.bb.log_global_limit` 则在全局范围内限制系统允许分配的 BBLog 事件总数。

最后三个 `sysctl` 变量与跟踪点相关。`net.inet.tcp.bb.tp.bbmode` 指定了触发跟踪点时要使用的 BBLog 状态。`net.inet.tcp.bb.tp.count` 限制了允许触发指定跟踪点的连接数量。例如，如果设置为 4，则允许 4 个连接触发跟踪点，之后不再触发该特定点（用于限制生成的 BBLog 事件数量）。`net.inet.tcp.bb.tp.number` 指定要启用的跟踪点编号。


### 通过套接字 API 的 TCP 连接特定配置

以下是用于控制单个连接上的 BBLog 的 `IPPROTO_TCP` 级别套接字选项：

- `TCP_LOG`——此选项设置连接上的 BBLog 状态。所有对此套接字选项的使用都会覆盖之前的设置。
- `TCP_LOGID`——此选项传递一个字符串，用于命名由 `tcplog_dumper` 生成的文件。该字符串作为与连接关联的“ID”。注意，多个连接可以使用相同的“ID”字符串，这是可能的，因为 `tcplog_dumper` 在生成的文件名中也包含 IP 地址和端口信息。
- `TCP_LOGBUF`——此套接字选项可用于从当前连接的日志缓冲区读取数据。通常不使用该选项，而是通过 `/dev/tcp_log` 使用通用工具（如 `tcplog_dumper`）读取和存储 BBLog。不过，它提供了一种可选方案，让用户进程能够收集多个日志。
- `TCP_LOGDUMP`——此套接字选项指示 BBLog 系统将连接队列中的任何记录转储到 `/dev/tcp_log`。如果未提供转储原因或 ID，则日志文件中的“reason”字段将使用当前日志类型的系统默认值。
- `TCP_LOGDUMPID`——此套接字选项类似于 `TCP_LOGDUMP`，让 BBLog 系统把所有记录转储到 `/dev/tcp_log`，但额外指定了用户给定的“原因”，该原因会包含在 BBLog 的“原因”字段中。
- `TCP_LOG_TAG`——此选项将一个额外的“标签”以字符串形式与该连接的所有 BBLog 记录关联。

例如，如果可以访问使用 TCP 连接的程序的源代码，则可以使用套接字选项 `TCP_LOG` 将连接的 BBLog 状态设置为 `TCP_LOG_STATE_CONTINUAL`：

```c
#include <netinet/tcp_log_buf.h>

int err;
int log_state = TCP_LOG_STATE_CONTINUAL;
err = setsockopt(sd, IPPROTO_TCP, TCP_LOG, &log_state, sizeof(int));
```

此代码也可用于前面提到的所有 BBLog 状态。

若无法访问源代码，可以使用 root 权限进行设置。、

```c
tcpsso -i id TCP_LOG 4
```

其中 `id` 是 `inp_gencnt`，可以通过运行 `sockstat -iPtcp` 确定，而 `4` 是 `TCP_LOG_STATE_CONTINUAL` 的数值。

## 生成 BBLog 文件

在为特定的 TCP 连接启用 BBLog 之前，首先需要确保 BBLog 的收集正在进行。FreeBSD 提供了一个名为 `tcplog_dumper` 的工具来完成此任务，可以在 ports 中找到该工具（路径为 `net/tcplog_dumper`）。可以以 root 权限运行以下命令来安装：

```sh
pkg install tcplog_dumper
```

添加：

```sh
tcplog_dumper_enable=”YES”
```

将该行添加到文件 `/etc/rc.conf` 后，在下次重启时会自动启动守护进程。也可以通过以 root 权限手动运行以下命令来启动守护进程：

```sh
tcplog_dumper -d
```

默认情况下，`tcplog_dumper` 会将 BBLog 收集到目录 `/var/log/tcplog_dumps` 中。它还支持其他几个选项，包括：

- `-J`——此选项会使 `tcplog_dumper` 输出到 `xz` 压缩的文件。
- `-D directory path`——将收集的文件存储在指定的目录路径中，而非默认目录。此路径也可以通过 `rc.conf` 变量 `tcplog_dumper_basedir` 控制。

`tcplog_dumper` 会输出文件格式 pcapng（pcap next generation）。pcapng 可存储元信息以及数据包信息。对于事件 `TCP_LOG_IN` 和 `TCP_LOG_OUT`，`tcplog_dumper` 从事件中生成一个 IP 头（因此，除了源和目标 IP 地址外，IP 头中的其他字段可能与实际传输的不同），使用事件中的 TCP 头（即与网络上传输的段相同），并添加一个具有正确长度的虚拟负载。对于每个 TCP 连接，`tcplog_dumper` 会创建一系列文件，每个文件大约包含 5000 个 BBLog 事件，按序号命名为 .0、.1、.2 等。以下是单个 TCP 连接的一系列 7 个文件的示例：


```sh
[rrs]$ ls /var/log/tcplog_dumps/
UNKNOWN_18262_10.1.1.1_9999.0.pcapng UNKNOWN_18262_10.1.1.1_9999.4.pcapng
UNKNOWN_18262_10.1.1.1_9999.1.pcapng UNKNOWN_18262_10.1.1.1_9999.5.pcapng
UNKNOWN_18262_10.1.1.1_9999.2.pcapng UNKNOWN_18262_10.1.1.1_9999.6.pcapng
UNKNOWN_18262_10.1.1.1_9999.3.pcapng records
```

因此，`TCP_LOGID` 并未在该连接上设置，其中一个 TCP 端口为 18262，另一个 TCP 端口为 9999，远程 IPv4 地址为 10.1.1.1。

目前正在开发从核心转储中生成 BBLog 文件的功能。该功能将使用调试器提取信息并将其提供给 `tcplog_dumper`，以实际写入 BBLog 文件。

## 读取 BBLog 文件

有两个可以轻松读取 BBLog 文件的工具：`read_bbrlog` 和 `wireshark`，两者都可以通过 ports 或软件包获得。

### read\_bbrlog

`read_bbrlog` 是一款小型程序，能读取一系列 BBLog 文件，并以文本形式显示每条日志记录。它需要指定 BBLog 文件的前缀作为输入源，并找到与该 TCP 连接关联的所有文件，将每个事件以文本形式输出到 `stdout`。需要注意的是，也可以选择将输出重定向到文件（强烈建议这样做，因为会显示大量数据）。以下是运行 `read_bbrlog` 的示例：

```sh
[rrs]$ read_bbrlog -i UNKNOWN_18262_10.1.1.1_9999 -o
my_output_file.txt -e Files:7 Processed 30964 records Saw
30964 records from stackid:3 total_missed:0 dups:0
```

在这种情况下，使用了三个参数：`-i input`，其中输入参数是基本连接 ID，即 `ls` 命令显示的文本，去除后缀 `.X.pcapng`；`-o outfile` 用于将输出重定向到输出文件 `my_output_file.txt`；最后是参数 `-e`，通常用于输出“扩展”信息，显示更详细的内容。

以下是文件 `my_output_file.txt` 中的一小段数据，展示了数据的格式。请注意，由于行长度较长，部分数据显示已被截断：

```ini
106565924 0 rack [50] PKT_OUT    Sent(0) 763046978:5  (PUS|ACK fas:0 bas:1) bw:208.00 bps(26)
                       avail:5 cw:14480 scw:14480 rw:65535 flt:0 (spo:64 ip:0)
106565979 0 rack [55] TCP_HYSTART  -- New round begins round:1 ends:763046983 cwnd:14480 106565982 0 rack [3] BBR_PACING_CALC Old rack burst mitigation len:5 slot:0 trperms:369 106565985 0 rack [3] TIMERSTAR type:TLP(timer:4) srtt:39001 (rttvar:17063 * 4) rttmin:30000 106565986 0 rack [1] USERSEND   avail:5 pending:5 snd_una:763046978 snd_max:763046983 out:5 106565986 0 rack [0] TCP_LOG_PRU pru_method:SEND (9) err:0
106607480 0 rack [2] IN         Ack:Normal 5 (PUS|ACK) off:32 out:5 lenin:5 avail:5 cw:14480
                                 rw:4000512 una:763046978 ack:763046983
```

这表明，在时间标记 106565924 时发送了一个 5 字节的包，序列号为 763046978。当时的拥塞窗口大小为 14480 字节，flight size（flt，在途字节数）为 0。未启用 Pacing 控制。大约 41 毫秒后（41,494，即 106604046-106607480），收到了对这些字节的确认。

## Wireshark

也可以用 `wireshark` 和 `tshark`  来显示 BBLog 文件。它们只能操作单个文件，而不像 `read_bbrlog` 那样操作文件系列。目前，不会显示事件特定的详细信息。对于事件 `TCP_LOG_IN` 和 `TCP_LOG_OUT`，BBLog 信息会显示在帧信息中。对于所有其他事件，BBLog 信息会直接显示。

---

**RANDALL STEWART**（[rrs@freebsd.org](mailto:rrs@freebsd.org)）是一位操作系统开发人员，已有 40 余年的经验，自 2006 年以来一直是 FreeBSD 开发者。他专注于传输协议，包括 TCP 和 SCTP，但也曾涉猎操作系统的其他领域。目前，他是一名独立顾问。

**MICHAEL TÜXEN**（[tuexen@freebsd.org](mailto:tuexen@freebsd.org)）是明斯特应用技术大学的教授，同时为 Netflix 做兼职承包商，自 2009 年以来一直是 FreeBSD 源代码提交者。他的研究重点是传输协议，如 SCTP 和 TCP，它们在 IETF 的标准化及其在 FreeBSD 中的实现。

