# IPFW 概述

- 原文：[IPFW: An Overview](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking/ipfw-an-overview/)
- 作者：**Allan Jude**

纵观其历史，BSD 操作系统家族一直以出色的防火墙著称。IPFW 不如 PF 包过滤器那样广受关注，但功能完备且优势众多。IPFW 最早于 1994 年随 FreeBSD 2.0 引入，`dummynet(4)` 功能则于 2.2.8（1998 年）加入。

IPFW 当前的形态是一次彻底重写后的版本，称为 IPFW2，于 2002 年夏完成并引入。IPFW 速度极快，SMP 可扩展性也极为出色。IPFW 是"首次匹配"型防火墙，意思是每个数据包都按编号规则列表进行比对，一旦某条规则匹配，搜索即告结束。这使得管理员可以按特定顺序编写规则以获得最高速度，并避免把某些数据包与更复杂的规则进行比对。带宽和质量可通过管道和队列定义，由规则强制执行。IPFW 还具有内核内 NAT 实现，对既有的用户态 natd 形成补充；完整支持 VIMAGE/VNET，从而在每个 VNET Jail 中创建独立的防火墙实例；支持多规则集、动态规则，并与操作系统紧密集成，提供包括按生成数据包的用户或 Jail 匹配规则等功能。

本文介绍启用和配置 IPFW 的基础知识，然后讨论若干高级话题，包括规则编号建议、模拟真实网络环境、流量优先级与整形，以及使用内核内 NAT 实现——包括与 Jail 配合配置端口转发。文章最后概述 IPFW 的一些其他特性。

## 加载防火墙

虽然 IPFW 可以编入自定义内核，但通常通过加载其内核可装载模块来启用。由于默认策略是拒绝所有流量，因此应当创建自定义规则集，或加载示例规则集。内置的 **/etc/rc.firewall** 脚本包含若干基本防火墙规则集的逻辑。可用的模板有：`open`、`closed`、`simple`、`client` 和 `workstation`。

要启用 IPFW 但不阻断任何流量，可在 **/etc/rc.conf** 中加入：

```sh
firewall_enable="YES"
firewall_type="OPEN"
```

要创建简单的有状态防火墙：

```sh
firewall_enable="YES"
firewall_type="CLIENT"
firewall_client_net="192.168.0.0/24" # 使用你内部网络的 IP
firewall_client_net_ipv6="" # 如有内部 IPv6 网络请在此指定
```

另一种方式是将自定义规则集文件的完整路径作为 `firewall_type` 指定，IPFW 会读取该文件并将每一行解释为 `ipfw` 命令的参数。IPFW 还支持对指定文件使用预处理器（如 m4），通过 `-p` 标志指定。这使管理员可以创建单一模板，经预处理后生成适用于不同主机的防火墙规则。

规则就位后，`service ipfw start` 会启动防火墙并应用规则。如果通过远程连接操作，记得用 `nohup` 包装该命令，否则在防火墙模块加载、允许连接的规则添加之前，连接可能被关闭。

> **提示**：**/usr/share/examples/ipfw/** 中的 `change_rules.sh` 脚本会应用一组新的防火墙规则，然后提示管理员确认是否满意新规则。如果管理员不满意，或被新规则锁在门外，旧规则将在 30 秒后恢复。

## 简单规则

IPFW 规则相当直观，易于编写。下面简要演示 OPEN 防火墙的基本管理命令：

使用 `ipfw list` 查看现有规则：

```sh
# ipfw list
00100 allow ip from any to any via lo0
00200 deny ip from any to 127.0.0.0/8
00300 deny ip from 127.0.0.0/8 to any
65000 allow ip from any to any
65535 deny ip from any to any
```

第一列是规则编号。每条规则被赋予一个编号，决定了数据包被比对的顺序。可以有多条规则共用同一编号，但这样会增加管理难度。下一个字段是匹配该规则的数据包数，其后是这些匹配数据包的总字节数。这些计数器可以用 `ipfw` 的 `zero` 子命令清零。每行余下部分就是规则本身。

要添加一条阻止到 25 端口入站连接的基本规则，使用以下命令：

```sh
# ipfw add 5001 unreach port tcp from any to me dst-port 25
```

这会创建编号为 5001 的规则。规则始终通过关键字 `add` 创建，并立即生效。在这条规则中，匹配数据包将被施加的动作是 `unreach port`，它会生成一条 ICMP 回复，告知远端主机该端口不可访问；与之相对，`deny` 会静默丢弃数据包。规则体 `tcp from any to me dst-port 25` 决定哪些数据包匹配。第一个关键字是协议（`ip`、`icmp`、`tcp`、`udp` 等）。规则体的下一节指明数据包的源地址和目的地址；它允许使用关键字 `any` 和 `me`，后者会匹配分配给本机任何接口的任意地址。最后可指定附加选项，包括源端口和目的端口（`src-port` 和 `dst-port`）、方向（`in` 或 `out`）、接口（`via em0`）、尝试建立新连接（关键字 `setup`，即 SYN 标志置位而 ACK 标志未置位的数据包）、已建立的连接（关键字 `established`，即 ACK 或 RST 标志置位的数据包），以及大多数其他 IP 和 TCP 协议头部字段。

更高级规则的示例：

```sh
# ipfw add 5002 allow log logamount 50 ip from 192.168.0.0/24 to any dst-port 80 setup
```

这创建编号为 5002 的规则（为了一致性以及管理员的理智，示例规则都从 5000 起编号）。该规则会匹配、记录并允许从指定子网到任意主机 80 端口的新建连接尝试。日志上限设为 50 条：虽然额外的数据包仍被允许通过，但只有前 50 个匹配该规则的数据包会被记录，以防日志被允许通过的数据包填满。如有需要，可用 `ipfw resetlog` 命令将日志计数器重置为 0。

## 命名

规则、管道和队列各自有独立的规则编号空间，范围在 1 到 65535 之间。这意味着一条规则和一个管道可以共用同一编号，且彼此无关。无论采用何种编号体系，请记住：如果你在远程操作防火墙（这是常态，且往往在深夜），在编号中保留某种功能性隔离会很有益处。隔离编号系统（5000 段用于规则、30000 段用于管道、40000 段用于队列）并可能按相关数字集分组（5001、30001、40001），有助于清晰并避免混淆。

## 模拟真实网络

在真实世界条件下测试应用往往是必要的——真实世界的网络并非没有延迟和拥塞的安静局域网。IPFW 提供了名为 `dummynet(4)` 的功能，用于模拟公共互联网的真实世界条件。`dummynet(4)` 提供人工限制、排队、延迟或丢弃数据包的能力，以创建所需的模拟网络条件。

要启用 `dummynet(4)` 功能，必须加载内核模块。这可以通过在 **/etc/rc.conf** 中加入以下内容来自动完成：

```sh
dummynet_enable="YES"
```

`dummynet(4)` 最基本的示例是创建带宽受限的管道：

```sh
# ipfw pipe 30003 config bw 5Mbit/s
# ipfw add 5003 pipe 30003 ip from any to 192.168.0.101
```

第一条命令把管道 30003 配置为 5 兆比特每秒的带宽。管道编号与规则编号相互独立（为了一致性以及管理员的理智，示例管道编号在 30000 段，最低几位与对应规则匹配）。第二条命令用 `add` 关键字创建防火墙规则 5003。规则 5003 把所有匹配的流量（即未匹配此前规则的全部流量）导向管道 30003。这会将主机 `192.168.0.101` 通过防火墙的下行带宽有效限制为 5 兆比特每秒。

这对于整形来自特定主机的流量，或模拟较慢链路很有用，但在定性网络多样性方面并不能提供特别真实的模拟。更好的模拟可以这样写：

```sh
# ipfw pipe 30003 config bw 5Mbit/s delay 150 burst 128k plr 0.001 queue 32KBytes noerror
```

这会用一些便于重现由网络问题引起的应用行为的选项重新配置管道 30003。`delay` 选项给每个数据包增加 150 毫秒延迟，模拟洛杉矶到伦敦之间的延迟。`burst` 选项允许略高于最大带宽的瞬时使用——前提是管道此前未满；若管道空闲，则前 128KB 数据不经速率限制通过。下一个选项模拟 0.1% 的丢包率（`plr`），引发偶发重传，正如在不甚理想的网络上常见的那样。`queue` 选项设定在拒绝后续数据包之前，可接受的最大额外数据量（以数据包或 KB 为单位）；通过设置低速率上限配大队列，可借此模拟缓冲膨胀。最后一个选项 `noerror` 使防火墙在丢包时不向调用应用返回错误。

通常，若试图发送的数据超出速率上限，防火墙会以与无限制网络上设备队列已满时相同的错误通知调用应用。抑制该错误可模拟更上游路由器处的丢包——应用将无法感知数据包被丢弃，直至连接另一端未收到确认。

## 控制数据流

管道也可以动态分配。例如，如果防火墙后有大量客户端，可以为每个客户端分配 5Mbit/s 的流，方法是创建基于掩码（本例中为 /24）的动态管道：

```sh
# ipfw pipe 30004 config bw 5Mbit/s mask src-ip 0x000000ff
# ipfw add 5004 pipe 30004 ip from any to 192.168.0.0/24
```

也可以基于目的 IP、源端口或目的端口、或协议进行掩码。还有一种选项是对所有位进行掩码（源和目的 IP、源和目的端口以及协议）。这把任一连接（流）限制为 5Mbit/s，但允许每个客户端以该速度建立多个连接：

```sh
# ipfw pipe 30004 config bw 5Mbit/s mask all
```

有时流量整形的目标不是限制任一主机的流量，而是确保一定带宽在一组主机间均分。此时，与其把所有带宽直接通过一个管道推送，不如用防火墙创建若干具有不同优先级的队列，以对不同类型的流量分类。当用掩码创建动态队列时，每个流（在下面的例子中，子网内每个源 IP 地址为一个流）均分一个父管道。创建带宽受限的管道，然后创建一个使用该管道的队列。队列编号与管道编号相互独立（为了一致性以及管理员的理智，示例队列编号从 40000 起编号）。添加一条匹配所需流量的规则。该队列规则会按指定掩码决定的不同流标识，为每个独立流创建一个动态队列。每个流对受限管道享有同等访问权。

```sh
# ipfw pipe 30005 config bw 75Mbit/s
# ipfw queue 40005 pipe 30005 mask src-ip 0x000000ff
# ipfw add 5005 queue 40005 ip from any to 192.168.0.0/24
```

与之对照的是动态管道，其中每个源 IP 地址（流标识）各有独立的速率上限。

有时均分就够了。但“所有主机生而平等，但有些主机比其他主机更平等”。队列可以加权，允许某些流量获得可用管道中更大的份额：

```sh
# ipfw pipe 30006 config bw 75Mbit/s
# ipfw queue 40006 pipe 30006 mask src-ip 0x000000ff weight 5
# ipfw queue 40007 pipe 30006 mask src-ip 0x000000ff weight 25
# ipfw add 5006 queue 40006 ip from any to 192.168.0.0/24
# ipfw add 5007 queue 40007 ip from any to 192.168.1.0/24
```

创建一个分配带宽的管道和两个权重不同的队列，配上相应子网规则，意味着第二个子网中的主机对所分配带宽享有更高优先级。

同样的流量管理风格也可应用于特定应用和服务。建立一个管道，添加两个权重不同的队列，把队列插入特定服务的规则中，并添加一条最终的兜底规则：

```sh
# ipfw pipe 30008 config bw 50Mbit/s queue 20
# ipfw queue 40008 pipe 30008 mask all weight 100
# ipfw queue 40011 pipe 30008 mask all weight 10
# ipfw add 5008 queue 40008 ip from any to any dst-port 5060
# ipfw add 5009 queue 40008 ip from any to any src-port 5060
# ipfw add 5010 queue 40008 ip from any to any iptos lowdelay
# ipfw add 5011 queue 40011 ip from any to any
```

上述规则集创建了一条带宽为 50 兆比特每秒、最大队列为 20 个数据包的管道。随后创建两个队列，第一个的权重是第二个的 10 倍。流量随后被分入这两个队列之一。源或目的端口为 5060（SIP）的数据包，或带有 `IPTOS_LOWDELAY` 标志的数据包进入高优先级队列（40008），其余流量进入低优先级队列（40011）。这应有助于确保 VoIP 通话在网络活动高峰期不受影响。

网络流量也可以根据防火墙所在机器的特定标准整形。IPFW 可按生成流量的用户、组或 Jail 匹配流量：

```sh
# ipfw pipe 30014 config bw 100Mbit/s
# ipfw pipe 30015 config bw 5Mbit/s
# ipfw pipe 30016 config bw 10Mbit/s
# ipfw add 5012 allow ip from any to any uid root
# ipfw add 5013 allow ip from any to any gid wheel
# ipfw add 5014 pipe 30014 ip from any to any jail 4 in
# ipfw add 5015 pipe 30015 ip from any to any jail 4 out
# ipfw add 5016 pipe 30016 ip from any to any
```

这组规则按生成流量的用户或 Jail 整形。前三条命令配置了具有特定可用带宽的管道。接下来的两条规则允许 root 用户或 wheel 组成员生成的所有流量不受整形地通过。再下面一对规则匹配进出特定 Jail 的流量，创建非对称连接，将入站流量限制为 100mbps，出站流量限制为 5mbps。最后一条规则匹配其余所有流量（其他用户和 Jail），将其限制为 10mbps 总量（不分方向）。

## Jail 的基本 NAT

当你需要快速建立基本 NAT，让一台面向公网的机器上的若干 Jail 都能访问互联网，而每个 Jail 又没有独立 IP 地址时，IPFW 会很有用。本例假设这些 Jail 的内部 IP 地址绑定在 `lo0` 上。

要启用 NAT，在 **/etc/rc.conf** 中加入：

```sh
gateway_enable="YES"
firewall_enable="YES"
firewall_type="OPEN"
firewall_nat_enable="YES"
firewall_nat_interface="em0" # 公共接口
firewall_nat_flags="redirect_port tcp 10.99.0.2:80 80 redirect_port tcp 10.99.0.2:443 443"
```

这会创建开放式防火墙，通过分配给 `em0` 的 IP 地址对出站流量做 NAT。同时配置端口转发，将 80 和 443 端口上的入站流量重定向到该 Jail 的私有 IP。

## 附加功能

IPFW 还有许多其他关键字可用于创建高级规则集。`prob` 关键字作为规则动作的一部分，决定一个数据包匹配该规则的概率。借此，管理员可以构造规则将部分流量以不同方式引导，用于分流测试、负载均衡或模拟故障。数据包也可以被打上数字 ID 标签，供后续规则使用，例如在接口之间建立信任关系。IPFW 还包含转发能力；`fwd` 关键字会在数据包通过防火墙时改变其内部的下一跳字段。它不修改数据包头部，但会改变内核路由该数据包的方式，非常适合用于基于 IP 或端口的负载均衡。IPFW 可用 `tee` 关键字创建软件监控端口，将每个匹配数据包的副本通过 `divert(4)` 套接字发送到用户态。IPFW 还可用于给数据包打上特定 FIB（转发信息库）标记，使匹配的数据包按特定内核路由表路由。

## 结论

本文仅触及 IPFW 能力和特性的皮毛。`ipfw(8)` 手册页为每个特性提供了详尽的文档和丰富示例。FreeBSD 手册中也包含一章对 IPFW 的附加说明和示例。有疑问的用户可发送至 freebsd-questions 邮件列表或在 FreeBSD 论坛发帖。

Allan Jude 是 ScaleEngine Inc. 的运营副总裁，该公司是一家全球性 HTTP 和视频流内容分发网络，他在其中大量使用 FreeBSD 上的 ZFS。他还是 JupiterBroadcasting.com 上视频播客“BSD Now”（与 Kris Moore 主持）和“TechSNAP”的主持人。Allan 目前正在争取 FreeBSD 文档 commit 权限，改进手册并撰写 ZFS 文档。他于 2007 至 2010 年间在加拿大汉密尔顿的 Mohawk College 教授 FreeBSD 和 NetBSD，拥有 12 年 BSD Unix 系统管理员经验。
