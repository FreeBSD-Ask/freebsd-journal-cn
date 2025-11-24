# FreeBSD 中的 RACK 栈和替代 TCP 栈

- 原文链接：[RACK and Alternate TCP Stacks for FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/rack-and-alternate-tcp-stacks-for-freebsd/)
- 作者：Randall Stewart、Michael TÜxen

在 2017 年，FreeBSD 对 TCP 栈进行了修改，实现了多 TCP 栈共存。以这种方式，现有的 TCP 栈可保持不变，从而能在有限的函数调用数量下进行创新。某些功能仍然是所有 TCP 栈共享的：例如，SYN 缓存的实现，包括 SYN Cookie 的处理；以及处理传入 TCP 段的初步步骤，比如校验和验证和根据端口号与 IP 地址查找 TCP 端点。在任何时刻，单个 TCP 连接只由单个 TCP 栈处理，但在 TCP 连接的生命周期内，此 TCP 栈是可更换的。

这就是 TCP RACK 栈的起源，它从调用函数 `tcp_do_segment()` 及许多其他模块化子函数开始，完全重写了原始的 TCP 栈。最初目标是支持一种名为“Recent Acknowledgement”（RACK）的丢包检测方法。RACK 最初出现在一份互联网草案中，并在 2021 年成为了 RFC 8985。这也是该 TCP 栈名称——RACK 的由来。然而，TCP RACK 栈的功能已经远超对 RFC 8985 的支持。重写的那部分，以完全不同的方式来处理 SACK（selective acknowledgement）信息。在 TCP RACK 栈中，维护了所有发送的用户数据的完整映射，这使得用户数据的重传处理得到了改进，并且实现了 RFC 8985 中说明的 RACK 丢包检测。许多额外的特性也是从这次重写中衍生出来的，本文将详细介绍这些特性。

## 如何使用 TCP RACK 栈

在 FreeBSD CURRENT 和 FreeBSD 14.0，均可使用 TCP RACK 栈。如何启用取决于使用的 FreeBSD 版本。

对于 FreeBSD 14.0，需要在内核配置文件中添加如下两行：

```sh
option TCPHPTS
makeoptions WITH_EXTRA_TCP_STACKS=1
```

然后重新编译内核。首行将 TCP 高精度定时器系统（HPTS）编译进内核。第二行会生成一个 TCP RACK 栈的可加载内核模块（`tcp_rack.ko`）。要使用 TCP RACK 栈，必须加载此内核模块。可以在 `/boot/loader.conf` 文件中添加如下行，使其在每次重启时自动加载：

```sh
tcp_rack_load=”YES”
```

在 FreeBSD CURRENT 中，TCP RACK 和 HPTS 均默认作为内核模块构建。由于 `tcphpts.ko` 是 `tcp_rack.ko` 的依赖项，因此仅需加载后者即可。要在每次重启时加载 TCP RACK 栈，需在 `/boot/loader.conf` 文件中添加如下两行：

```sh
tcphpts_load=”YES”
tcp_rack_load=”YES”
```

在 FreeBSD CURRENT 中，也可以将如下两行添加到内核配置文件中，将 TCP RACK 栈静态编译进内核：

```sh
option TCPHPTS
option TCP_RACK
```

然后重新编译内核。

值得注意的是，现在在所有 64 位平台上的 FreeBSD 14.0 及更高版本和 FreeBSD CURRENT 上，都默认构建了 TCP BBLog（参数 `TCP_BLACKBOX`），因为它是 TCP 传输开发人员用来对各种 TCP 栈进行测试和调试的标准方式。

上述内容概述了如何在 FreeBSD 系统上启用 TCP RACK 栈。可以在 shell 中运行以下命令，来查看所有可用的 TCP 栈列表：

```sh
sysctl net.inet.tcp.functions_available
```

在即将发布的版本——FreeBSD 14.1 及更高版本——TCP RACK 栈的使用方法与上述 FreeBSD CURRENT 一致。

实际使用 TCP RACK 栈的方式有很多种，有些方式需要修改应用程序的源代码，而有些方式只需要进行配置更改。

`sysctl` 变量 `net.inet.tcp.functions_default` 用于指定新创建的 TCP 端点（通过系统调用 `socket(2)` 创建）默认使用的 TCP 栈。执行以下命令：

```sh
sysctl net.inet.tcp.functions_default=rack
```

会把默认栈设置为 TCP RACK 栈。在 `/etc/sysctl.conf` 文件中添加以下行，在重启后，TCP RACK 栈会成为默认的 TCP 栈：

```sh
net.inet.tcp.functions_default=rack
```

当通过 listener 创建 TCP 端点时，TCP 栈要么继承自 listener，要么基于默认的 TCP 栈，这取决于 `net.inet.tcp.functions_inherit_listen_socket_stack` 的值是非零还是 `0`。该变量的默认值为 `1`。

也可以使用命令行工具 `tcpsso(8)` 来改变单个 TCP 连接的 TCP 栈，如该工具的手册页所述。

如果能修改源代码，则可以使用名为 `TCP_FUNCTION_BLK` 的 `IPPROTO_TCP` 级别的套接字参数，将用于该套接字的 TCP 栈切换到 TCP RACK 栈。选项值的类型为 `struct tcp_function_set`。例如，可用如下代码执行此操作：

```c
struct tcp_function_set tfs;

strncpy(tfs.function_set_name, “rack”, TCP_FUNCTION_NAME_LEN_MAX);
tfs.pcbcnt = 0;
setsockopt(fd, IPPROTO_TCP, TCP_FUNCTION_BLK, &tfs, sizeof(tfs));
```

使用 TCP RACK 栈可启用许多当下默认 TCP 栈不支持的功能。许多这些功能可以通过 `IPPROTO_TCP` 级别的套接字参数和 `sysctl` 变量 `net.inet.tcp.rack` 进行控制。

## TCP RACK 栈的功能

以下部分介绍了 TCP RACK 栈提供的最重要的功能。

### RACK/TLP

TCP RACK 栈中的集成了两项功能：Recent Acknowledgement（RACK）和 Tail Loss Probe（TLP）。RACK 改变了数据包丢失的检测方式及重传的触发方式。FreeBSD 基础栈中实现的丢包检测，参照 RFC 5681，要求三次重复的 ACK 或带 SACK 的 ACK 到达，才会触发 TCP 栈发送重传。在某些情况下，例如发送的包数少于四个时，这会导致 TCP 栈仅在重传超时（RTO）发生后才发送重传。RACK 则改变了这一行为，当 SACK 到达时，如果距离丢失的数据包已经有足够的时间，会立刻重传。若时间不够（时间通常比当前的 RTT 稍长），则启动一个小的 RACK 定时器，在定时器到期后进行重传。这样可以修复很多原本需要通过重传超时来强制发送数据的情况。最后一种情况由 TLP 解决。当 TCP RACK 栈发送数据时，它会启动 TLP 定时器，而非重传定时器。如果 TLP 定时器到期，TCP RACK 栈会发送一个新的数据段或者最后一个发送的段。这个 TLP 发送的数据段的期望效果是：发送方会收到一个 ACK，表示所有数据已经接收（这是上次 ACK 丢失的情况）；或者 TLP 会触发一个 SACK，这样就可以启动正常的快速恢复机制，而不会触发重传超时，从而避免拥塞窗口降至 1 个 MSS（最大报文段大小）。

启用 TCP RACK 栈后，用户能自动获得 RACK 和 TLP 的好处。无需上层进行任何套接字参数和配置。

### 比例速率降低（Proportional Rate Reduction，PRR）

比例速率降低（Proportional Rate Reduction，PRR）是 TCP RACK 栈的另一款自动集成功能，见 RFC 6937，并且目前正在由 IETF 进行更新。PRR 改进了快速恢复（fast recovery）期间数据发送的方式。使用 RFC 5681 中指定的 TCP 拥塞控制时，进入快速恢复阶段时，拥塞窗口会减半。这会导致在快速恢复期间发送新数据时发生停顿（stall）。基本上，发送方必须等待半数未 ACK 的数据被 ACK 后，才能开始发送新数据（以及重传数据）。这会导致发送方和接收方之间的数据流“停顿（stall）”。PRR 的设计目的是解决这个问题，在快速恢复期间，大约每收到一个 ACK，就可以发送一个新的数据段。这样可以避免数据停顿（stall），使数据持续流动，从而保持 RTT 和其他传输指标的活跃和更新。

### RACK 快速恢复（RACK Rapid Recovery，RRR）

RACK 快速恢复（RACK Rapid Recovery，RRR）是个有趣的功能，最初源于一个 bug。在最初的开发中，TCP RACK 栈在无意中允许了一种情况，即当一个 SACK 到达，声明多个数据段丢失并且 RACK 定时器为所有数据过期时，TCP RACK 栈会发送一个数据段，并启动 RACK 定时器。当 RACK 定时器过期（定时器设置为 RACK 的最小超时时间 1 毫秒）时，TCP RACK 栈会发送另一个丢失的数据段。这个过程会不断重复，直到所有丢失的数据段都被发送。实际上，这种方式忽略了 PRR，在初始恢复阶段，虽然后续的 PRR 数据段会在稍后的时间发送。举例来说，如果 RRR 发送了 3 个段，第一个是重传，后两个是额外的段，那么需要大约 6 个 ACK 到达后，PRR 才会发送出一个新的数据段。

当发现这个 bug 并“修复”后，用户的体验质量（QoE）反而下降了。这是因为这些早期丢失的数据段往往会阻塞后续多个数据段的传送。因此，这个功能被添加为一个可关闭的特性，并且可以通过编程设定恢复时间间隔，在默认情况下，RRR 恢复速率设置为每毫秒一个数据段。这个设置的速率假定最大传输单元（MTU）为 1500 字节时，恢复速率大约为 12 Mbps。

### SACK 攻击检测

保持一个完整的已发送数据映射是 TCP RACK 栈的一个特点，但这也带来了一个潜在问题：在某些情况下，该映射可能会变得非常大。TCP RACK 栈始终尝试将这个映射压缩到尽可能小，同时仍然能够追踪所有未 ACK 数据的状态。然而，这也引入了一种可能性，即恶意对等方可以设计攻击，利用不断将发送映射分割成更小的片段，迫使 TCP RACK 栈消耗大量内存和 CPU 资源，进行无休止的内存搜索。例如，攻击者可能会发送针对每一个字节的 SACK。这种攻击可能会构成严重威胁，并对机器产生不良影响。

TCP RACK 栈包含一个可选的编译功能 `TCP_SAD_DETECTION`（SACK 攻击检测）。可以将以下行添加到内核配置文件中，再重建内核来启用此功能：

```sh
option TCP_SAD_DETECTION
```

若加入该参数，则在默认情况下它是开启的。该功能会监控是否存在恶意对等方，如果检测到攻击，它会禁用对来自该对等方的 SACK 处理。这样会降低该对等方的性能，但不会阻止连接的进展。实际上，这样的连接会表现得好像 SACK 从未启用过一样。尽管丧失了丢包恢复功能，但连接仍然可以继续进行。

### 突发缓解

TCP RACK 栈内置了突发缓解功能，且无需用户干预。为了缓解突发流量，栈会在每次发送机会时仅发送一定大小的数据（即最大突发大小），并启动一个小的定时器（以便稍后发送更多数据），或者依赖返回的确认流来触发更多数据的发送。这有助于缓解可能导致过多丢包的大量突发数据。

### 对 TCP BBLog 的支持

TCP RACK 栈的一个有趣特性是它对 TCP BBLog 的普遍支持，无论是调试，还是一般的统计分析和仪器化。这使得追踪问题和分析连接行为变得更加容易。

### 集成大接收卸载（Large Receive Offload，LRO）以突发缓解

TCP 大接收卸载（Large Receive Offload，LRO）是一项通过将多个接收到的 TCP 段合并为一个段来减少接收方 CPU 资源消耗的功能。这样通常会丧失有关单个接收段的信息，但可以减少需要被 TCP 栈处理的段的数量，从而减少 CPU 资源的消耗。

一个有趣的功能交互是对 LRO 代码的改进，以更好地支持 TCP RACK 栈中的流量控制。当 TCP 连接正在进行突发缓解时，它往往会更频繁地经过发送路径，发送较小的突发数据。为此，对 LRO 代码进行了修改，使得所有数据包到达时的时间信息可以无损地传递到 TCP RACK 栈。在数据包处理过程中，LRO 代码会检查数据包是否与允许直接将包排队到 TCP RACK 栈的连接相关联。如果是，数据包会直接排队到该连接，并且根据连接的状态，连接可能会被唤醒。在 TCP RACK 栈进行突发缓解哥流量控制时，唤醒操作会延迟，直到定时器过期并可以处理传入的确认。这些步骤也绕过了 IP 栈的处理，因此可以进一步削减所需的 CPU 资源。

### 其他备用功能

TCP RACK 栈提供了许多其他功能，这些功能通过各种套接字选项和 `sysctl` 变量可用。目前，TCP RACK 栈支持 58 个套接字功能，启用各种功能，包括流量控制、突发缓解参数以及恢复响应修改。除了套接字选项之外，还有大约 150 个 `sysctl` 变量，可以将某些套接字参数应用于所有连接，或修改各种 TCP RACK 栈的默认配置。这些功能和配置帮助用户根据网络环境和需求，调整 TCP RACK 栈。

## Netflix 如何演进 TCP RACK 栈

目前，Netflix 仅使用 TCP RACK 栈，FreeBSD 默认栈虽存在，但并未启用。Netflix 使用 TCP RACK 栈的方式有些新颖，值得注意。实际上，Netflix 维护了多个版本的 TCP RACK 栈，并按版本号命名。Netflix 始终使用“最新”的 TCP RACK 栈，该版本包含所有由传输团队正在开发的前沿功能。

每当发布新版本时，Netflix 会复制正在开发的最新 TCP RACK 栈，并根据版本号进行支持。然后，Netflix 会根据用户体验质量（QoE）和 CPU 性能对该栈进行评估，并与先前发布的默认 TCP 栈进行比较。当最新版本的 TCP RACK 栈的表现至少与旧版本相当时，默认栈会在下一个版本中切换为新的 TCP RACK 栈。旧的 TCP RACK 栈会在多个版本中继续维护，最后被移除。

TCP RACK 栈的新特性也通过这种方式进行测试，以确定某项功能是否真的带来了好处。减少网络影响，且不降低 Netflix 用户的体验质量是 Netflix 传输团队的主要目标，这样 Netflix 不仅能更好地利用网络资源，同时还能为用户提供良好的体验质量。

## 总结与展望

TCP RACK 栈是 FreeBSD 默认栈的强大替代方案。它增加了更多的功能和参数，给应用开发者提供了更丰富的选择，以便更好地定制 TCP 体验，满足用户需求。

TCP RACK 栈已经在 Netflix 的设置和工作负载中进行了海量测试。但同样重要的是，还需要在其他设置和工作负载下进行测试。因此，如果用户能够在自己的硬件上、使用自己的设置和工作负载测试 TCP RACK 栈，那将极有价值。请将在测试中遇到的所有问题报告给 [net@freebsd.org](mailto:net@freebsd.org) 和本文作者。根据反馈和进一步的测试，TCP RACK 栈可能会成为未来 FreeBSD 的默认栈。

---

**Randall Stewart**（[rrs@freebsd.org](mailto:rrs@freebsd.org)）在操作系统开发领域已有 40 余年的经验，并且是 FreeBSD 的开发者，专注于 TCP 和 SCTP 等传输协议。他目前在 Netflix 的传输团队工作，支持 TCP 栈，同时不断创新，以提高用户的体验质量。

**Michael Tüxen**（[tuexen@freebsd.org](mailto:tuexen@freebsd.org)）是明斯特应用科技大学的教授，同时也是 Netflix 的兼职雇佣开发者，并且自 2009 年起就是 FreeBSD 源代码提交者。他的工作重点是 SCTP 和 TCP 等传输协议，及其在 IETF 中的标准化和在 FreeBSD 中的实现。
