# VPP 移植到了 FreeBSD：基础用法

- 原文链接：[Porting VPP to FreeBSD: Basic Usage](https://freebsdfoundation.org/our-work/journal/browser-based-edition/kernel-development/porting-vpp-to-freebsd-basic-usage/)
- 作者：Tom Jones

**Vector Packet Process (VPP)** 是一个高性能的框架，用于在用户空间处理数据包。得益于 FreeBSD 基金会和 RGNets 的项目，我获得资助将 VPP 移植到 FreeBSD，并且非常高兴与《FreeBSD 期刊》的读者分享一些基本的使用方法。

VPP 使得转发和路由应用程序可以在用户空间编写，并提供一个 API 可控的接口。通过 DPDK（在 Linux 上）和 DPDK 及 netmap（在 FreeBSD 上），VPP 实现了高性能的网络处理。这些 API 允许直接的零拷贝数据访问，并可用于创建可以显著超过主机转发性能的转发应用程序。

VPP 是一个完整的网络路由器替代方案，因此需要一些主机配置才能使用。本文展示了如何在 FreeBSD 上使用 VPP 的完整示例，大多数用户应该能够通过自己的虚拟机来跟随这些步骤。VPP 在 FreeBSD 上也可以运行在真实硬件上。

本文将介绍如何在 FreeBSD 上使用 VPP，并给出设置示例，展示如何在 FreeBSD 上完成各种任务。VPP 的资源可能较难找到，项目的文档质量很高，可以访问[https://fd.io](https://fd.io/)获取。

## 构建一个路由器

![](https://freebsdfoundation.org/wp-content/uploads/2024/11/Porting_VPP_to_FreeBSD__Basic_usage.png)

VPP 可以用于许多用途，其中最常见且最容易配置的是作为某种形式的路由器或桥接器。以将 VPP 用作路由器为例，我们需要构建一个包含三个节点的小型网络——一个客户端，一个服务器和一个路由器。

为了展示如何在 FreeBSD 上使用 VPP，我将构建一个最小开销的示例网络。你只需要 VPP 和一个 FreeBSD 系统。我还将安装 iperf3，以便我们能够生成并观察通过我们的路由器传输的一些流量。

在具有最新 Ports 的 FreeBSD 系统中，可以使用 pkg 命令安装我们所需的两个工具，如下所示：


```sh
host # pkg install vpp iperf3
```

为了为我们的网络创建三个节点，我们将利用 FreeBSD 最强大的功能之一——VNET jail。VNET jail 为我们提供了完全隔离的网络堆栈实例，它们的操作方式类似于 Linux 的网络命名空间（Network Namespaces）。为了创建一个 VNET，我们需要在创建 jail 时添加`vnet`选项，并传递它将使用的接口。

最后，我们将使用`epair`接口连接我们的节点。`epair`接口提供了以太网电缆两端的功能——如果你熟悉 Linux 上的`veth`接口，它们提供了类似的功能。

我们可以通过以下 5 个命令构建我们的测试网络：

```sh
host # ifconfig epair create
epair0a
host # ifconfig epair create
epair1a
# jail -c name=router persist vnet vnet.interface=epair0a vnet.interface=epair1a
# jail -c name=client persist vnet vnet.interface=epair0b
# jail -c name=server persist vnet vnet.interface=epair1b
```

在这些 jail 命令中需要注意的标志是`persist`，如果没有这个选项，jail 将会因为没有运行进程而自动删除；`vnet`使这个 jail 成为一个 VNET  jail；以及`vnet.interface=`，它将指定的接口分配给 jail。

当一个接口被移到新的 VNET 时，它的所有配置都会被清除——这是需要注意的，因为如果你配置了一个接口然后将其移到 jail，可能会困惑于为什么一切都不工作。

## 设置对等端

在转到 VPP 之前，我们先设置网络的客户端和服务器端。每一方都需要分配一个 IP 地址，并将接口设置为启用状态。我们还需要为客户端和服务器 jail 配置默认路由。


```sh
host # jexec client
# ifconfig
lo0: flags=8008<LOOPBACK,MULTICAST> metric 0 mtu 16384
options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
        groups: lo
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
epair0b: flags=1008842<BROADCAST,RUNNING,SIMPLEX,MULTICAST,LOWER_UP> metric 0
mtu 1500
        options=8<VLAN_MTU>
        ether 02:90:ed:bd:8b:0b
        groups: epair
        media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
        status: active
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
# ifconfig epair0b inet 10.1.0.2/24 up
# route add default 10.1.0.1
add net default: gateway 10.1.0.1
```

```sh
host # jexec server
# ifconfig epair1b inet 10.2.0.2/24 up
# route add default 10.2.0.1
add net default: gateway 10.2.0.1
```

我们的客户端和服务器 jail 现在已经有了 IP 地址，并且配置了通向 VPP 路由器的路由。

## Netmap 要求

在我们的示例中，我们将使用 VPP 与 Netmap，Netmap 是一个高性能的用户空间网络框架，作为 FreeBSD 的默认组件提供。使用 Netmap 之前，接口需要进行一些配置——接口需要处于启用状态，并且需要配置`promisc`选项。

```sh
host # jexec router
# ifconfig epair0a promisc up
# ifconfig epair1a promisc up
```

现在我们可以开始使用 VPP 了！

## VPP 的基本命令

VPP 非常灵活，提供通过配置文件、命令行接口和具有成熟 Python 绑定的 API 进行配置。VPP 需要一个基础配置文件，告诉它从哪里获取命令，以及它使用的控制文件的名称（如果这些文件不是默认的）。我们可以在命令行中将一个最小配置文件作为 VPP 的参数之一。对于这个示例，我们让 VPP 进入交互模式——提供给我们一个 CLI，并告诉 VPP 只加载我们将使用的插件（netmap），这是一个合理的默认设置。

如果我们不禁用所有插件，我们将需要配置机器使用 DPDK，或者单独禁用该插件。禁用插件的语法与启用 netmap 插件的语法相同。

```sh
host # vpp “unix { interactive} plugins { plugin default { disable } plugin
netmap_plugin.so { enable } plugin ping_plugin.so { enable } }”
      _______     _       _   _____  ___
    __/ __/ _ \  (_)__   | | / / _ \/ _ \
    _/ _// // / / / _ \  | |/ / ___/ ___/
    /_/ /____(_)_/\___/  |___/_/  /_/

vpp# show int
         Name            Idx        State  MTU (L3/IP4/IP6/MPLS)
Counter      Count
local0                    0         down          0/0/0/0
```

如果一切设置正确，你将看到 VPP 的横幅和默认的 CLI 提示符（`vpp#`）。

VPP 命令行界面提供了很多选项，用于创建和管理接口、像桥接一样的组、添加路由以及用于检查 VPP 实例性能的工具。

接口配置命令的语法类似于 Linux 的 iproute2 命令——对于来自 FreeBSD 的用户来说，这些命令可能稍显陌生，但待开始使用，它们还是相当清晰的。

我们的 VPP 服务器还没有配置任何主机接口，`show int`只列出了默认的`local0`接口。

要在 VPP 中使用我们的 netmap 接口，我们需要先创建它们，然后再进行配置。

创建命令允许我们创建新的接口，我们使用`netmap`子命令和主机接口。

```sh
vpp# create netmap name epair0a
netmap_create_if:164: mem 0x882800000
netmap-epair0a
vpp# create netmap name epair1a
netmap-epair1a
```

每个 netmap 接口都以`netmap-`为前缀创建。接口创建完成后，我们可以配置它们以供使用，并开始将 VPP 用作路由器。

```sh
vpp# set int ip addr netmap-epair0a 10.1.0.1/24
vpp# set int ip addr netmap-epair1a 10.2.0.1/24
vpp# show int addr
local0 (dn):
netmap-epair0a (dn):
  L3 10.1.0.1/24
netmap-epair1a (dn):
  L3 10.2.0.1/24
```

命令 `show int addr`（`show interface address` 的简写）确认了我们的 IP 地址分配已经成功。然后，我们可以将接口启用：

```sh
vpp# set int state netmap-epair0a up
vpp# set int state netmap-epair1a up
vpp# show int
          Name              Idx State  MTU (L3/IP4/IP6/MPLS)
Counter   Count
local0                         0  down         0/0/0/0
netmap-epair0a                 1   up          9000/0/0/0
netmap-epair1a                 2   up          9000/0/0/0
```

配置好接口后，我们可以通过使用 ping 命令来测试 VPP 的功能：

```sh
vpp# ping 10.1.0.2
116 bytes from 10.1.0.2: icmp_seq=2 ttl=64 time=7.9886 ms
116 bytes from 10.1.0.2: icmp_seq=3 ttl=64 time=10.9956 ms
116 bytes from 10.1.0.2: icmp_seq=4 ttl=64 time=2.6855 ms
116 bytes from 10.1.0.2: icmp_seq=5 ttl=64 time=7.6332 ms

Statistics: 5 sent, 4 received, 20% packet loss
vpp# ping 10.2.0.2
116 bytes from 10.2.0.2: icmp_seq=2 ttl=64 time=5.3665 ms
116 bytes from 10.2.0.2: icmp_seq=3 ttl=64 time=8.6759 ms
116 bytes from 10.2.0.2: icmp_seq=4 ttl=64 time=11.3806 ms
116 bytes from 10.2.0.2: icmp_seq=5 ttl=64 time=1.5466 ms

Statistics: 5 sent, 4 received, 20% packet loss
```

如果我们跳到客户端 jail，我们可以验证 VPP 正在充当路由器：

```sh
client # ping 10.2.0.2
PING 10.2.0.2 (10.2.0.2): 56 data bytes
64 bytes from 10.2.0.2: icmp_seq=0 ttl=63 time=0.445 ms
64 bytes from 10.2.0.2: icmp_seq=1 ttl=63 time=0.457 ms
64 bytes from 10.2.0.2: icmp_seq=2 ttl=63 time=0.905 ms
^C
--- 10.2.0.2 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.445/0.602/0.905/0.214 ms
```

作为初始设置的最后一步，我们将在服务器 jail 中启动一个 iperf3 服务器，并使用客户端进行 TCP 吞吐量测试。

```sh
server # iperf3 -s

client # iperf3 -c 10.2.0.2
Connecting to host 10.2.0.2, port 5201
[ 5] local 10.1.0.2 port 63847 connected to 10.2.0.2 port 5201
[ ID]   Interval         Transfer    Bitrate         Retr  Cwnd
[  5]   0.00-1.01   sec   341 MBytes  2.84 Gbits/sec  0     1001 KBytes
[  5]   1.01-2.01   sec   488 MBytes  4.07 Gbits/sec  0     1.02 MBytes
[  5]   2.01-3.01   sec   466 MBytes  3.94 Gbits/sec  144   612 KBytes
[  5]   3.01-4.07   sec   475 MBytes  3.76 Gbits/sec  0     829 KBytes
[  5]   4.07-5.06   sec   452 MBytes  3.81 Gbits/sec  0     911 KBytes
[  5]   5.06-6.03   sec   456 MBytes  3.96 Gbits/sec  0     911 KBytes
[  5]   6.03-7.01   sec   415 MBytes  3.54 Gbits/sec  0     911 KBytes
[  5]   7.01-8.07   sec   239 MBytes  1.89 Gbits/sec  201   259 KBytes
[  5]   8.07-9.07   sec   326 MBytes  2.75 Gbits/sec  0     462 KBytes
[  5]   9.07-10.06  sec   417 MBytes  3.51 Gbits/sec  0     667 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID]   Interval         Transfer     Bitrate         Retr
[  5]   0.00-10.06  sec  3.98 GBytes  3.40 Gbits/sec  345          sender
[  5]   0.00-10.06  sec  3.98 GBytes  3.40 Gbits/sec               receiver

iperf Done.
```

## VPP 分析

现在我们已经通过 VPP 发送了一些流量，`show int` 命令的输出包含了更多信息：

```sh
vpp# show int
     Name               Idx   State  MTU (L3/IP4/IP6/MPLS)             Counter           Count
local0                   0    down        0/0/0/0
netmap-epair0a           1    up          9000/0/0/0       rx packets            4006606
                                                           rx bytes           6065742126
                                                           tx packets            2004365
                                                           tx bytes            132304811
                                                           drops                       2
                                                           ip4                   4006605
netmap-epair1a           2    up          9000/0/0/0       rx packets            2004365
                                                           rx bytes            132304811
                                                           tx packets            4006606
                                                           tx bytes           6065742126
                                                           drops                       2
                                                           ip4                   2004364
```


接口命令现在给出了通过 VPP 接口传输的字节和数据包的摘要。这在调试流量如何流动时非常有用，特别是当你的数据包丢失时。

VPP 中的 V 代表向量（vector），这在项目中有两层含义。VPP 旨在使用向量化指令加速数据包处理，同时它还将一组数据包打包成向量，以优化处理。其理论是将一组数据包一起通过处理图，从而减少缓存争用并提供最佳性能。

VPP 提供了许多工具来查询数据包处理过程中的情况。深度调优超出了本文的讨论范围，但一个了解 VPP 内部发生情况的初步工具是运行时命令。

运行时数据会在每个向量通过 VPP 处理图时收集，记录每个节点的传输时间以及处理的向量数量。

要使用运行时工具，最好有一些流量。可以像这样启动一个长时间运行的 iperf3 吞吐量测试：

```sh
client # iperf3 -c 10.2.0.2 -t 1000
```

现在在 VPP jail 中，我们可以清除迄今为止收集到的运行时统计信息，稍等片刻，然后查看我们的运行情况：

```sh
vpp# clear runtime
... wait ~5 seconds ...
vpp# show runtime
Time 5.1, 10 sec internal node vector rate 124.30 loops/sec 108211.07
  vector rates in 4.4385e5, out 4.4385e5, drop 0.0000e0, punt 0.0000e0
          Name                  State         Calls       Vectors     Suspends   Clocks    Vectors/Call
ethernet-input                  active        18478       2265684     0          3.03e1    122.62
fib-walk                        any wait          0       0           3          1.14e4    0.00
ip4-full-reassembly-expire-wal  any wait          0       0           102        7.63e3    0.00
ip4-input                       active        18478       2265684     0          3.07e1    122.62
ip4-lookup                      active        18478       2265 684    0          3.22e1    122.62
ip4-rewrite                     active        18478       2265684     0          3.05e1    122.62
ip6-full-reassembly-expire-wal  any wait          0       0           102        5.79e3    0.00
ip6-mld-process                 any wait          0       0           5          6.12e3    0.00
ip6-ra-process                  any wait          0       0           5          1.18e4    0.00
netmap-epair0a-output           active         8383       755477      0          1.12e1    90.12
netmap-epair0a-tx               active         8383       755477      0          1.17e3    90.12
netmap-epair1a-output           active        12473       1510207     0          1.04e1    121.08
netmap-epair1a-tx               active        12473       1510207     0          2.11e3    121.08
netmap-input                    interrupt wa  16698       2265684     0          4.75e2    135.69
unix-cli-process-0              active            0       0           13         7.34e4    0.00
unix-epoll-input                polling      478752       0           0          2.98e4    0.00
```

`show runtime` 输出中的列为我们提供了关于 VPP 内部运行的极好洞察。它们告诉我们自从运行时计数器被清除以来，哪些节点已经被激活，它们的当前状态，每个节点被调用的次数，所使用的时间，以及每次调用处理的向量数量。默认情况下，VPP 的最大向量大小是 255。

一个最终的调试任务是使用 `show vlib graph` 命令检查整个数据包处理图。此命令显示每个节点及其潜在的父节点和子节点，帮助我们理解数据流向。

## 下一步

VPP 是一款令人印象深刻的软件——待解决了兼容性问题，VPP 的核心部分就相对容易移植。即使仅进行最小的调优，VPP 在 FreeBSD 上与 netmap 配合使用时也能够达到相当令人印象深刻的性能，并且如果配置了 DPDK，它的表现会更好。VPP 的文档正在逐步增加关于在 FreeBSD 上运行的信息，但开发者们确实需要更多关于 FreeBSD 上使用 VPP 的示例案例。

从这个简单网络的例子开始，将其移植到一个拥有更快接口的大型网络上应该是相对直接的。

---

**Tom Jones** 是一名 FreeBSD 提交者，致力于保持网络堆栈的高效运行。

