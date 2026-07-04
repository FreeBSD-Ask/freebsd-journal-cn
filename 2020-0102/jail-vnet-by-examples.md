# Jail vnet 实例教程

为理解虚拟网络特性（vnet），切勿与 VirtIO Ethernet 驱动 vtnet(4) 混淆，我们先引用 vnet(9) 手册页的一段：

## DESCRIPTION

```sh
vnet 是虚拟化网络栈的技术名称。
(...)。
每个（虚拟）网络栈都附加到一个 prison，其中 vnet0
是基础系统不受限制的默认网络栈。
```

作为相关的 prison 特性，我们查看 jail(8) 手册页中关于 vnet 的部分：

```sh
vnet    创建带有自身虚拟网络栈的 jail，拥有自身的
        网络接口、地址、路由表等。内核必须
        编译了 VIMAGE 选项才能使用此特性。可选值为 "inherit"
        使用系统网络栈（可能带有受限制的 IP 地址），
        或 "new" 创建新的网络栈。
```

简而言之，这一特性让每个 Jail 拥有自己的路由表、ARP 与 NDP 缓存以及接口。

## 词汇表

- Host：承载你 Jail 的系统

## 示例

这些示例使用基于 host `/` 的"空" Jail，仅聚焦 vnet 特性。它们都处于 "persist" 模式（因为没有进程运行）。

关于操作系统要求：

- 使用的 shell 是 `/bin/sh`。
- FreeBSD 12.1 起步（可以是 12-STABLE 或更佳的 -head）

作者：OLIVIER COCHARD-LABBÉ

## 无用的隔离 vnet Jail

这个无用的示例展示如何创建一个隔离的 vnet Jail。

命令行参数详情：

- `-c`：创建新 Jail
- name：Jail 名称，避免后续使用其 jail ID（JID）
- host.hostname：本示例中仅用于让 `jls` 命令输出美观
- persist：因 Jail 无进程运行，故强制其处于运行状态
- vnet：启用虚拟网络栈

```sh
# jail -c name=useless host.hostname=jvnet persist vnet
# jls
```

   JID  IP Address      Hostname                      Path
     1                  jvnet                         /

下面是该新 Jail 默认分配的网络接口及其路由表内容。

```sh
# jexec useless ifconfig
```

lo0: flags=8008<LOOPBACK,MULTICAST> metric 0 mtu 16384
        options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
        groups: lo
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>

```sh
# jexec useless netstat -rn
```

Routing tables

存在一个未配置的 loopback（禁用且未分配 IP 地址）和空路由表。我们来修复它。

```sh
# jexec useless service netif restart
```

Stopping Network: lo0.
lo0: flags=8008<LOOPBACK,MULTICAST> metric 0 mtu 16384
        options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
        groups: lo
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
Starting Network: lo0.
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384
        options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
        inet6 ::1 prefixlen 128
        inet6 fe80::1%lo0 prefixlen 64 scopeid 0x1
        inet 127.0.0.1 netmask 0xff000000
        groups: lo
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>

```sh
# jexec useless netstat -rn
```

Routing tables
Internet:
Destination        Gateway            Flags     Netif Expire
127.0.0.1          link#1             UH          lo0
Internet6:
Destination                     Gateway                       Flags     Netif Expire
::1                             link#1                        UH          lo0
fe80::%lo0/64                   link#1                        U           lo0
fe80::1%lo0                     link#1                        UHS         lo0

好多了！但我们只有 loopback 接口在运行。下一步是创建一个虚拟 Ethernet tap 接口并分配给 Jail。`ifconfig(8)` 手册页摘录：

```sh
vnet jail
    将接口移动到 jail(8)，按 name 或 JID 指定。如果
    jail 拥有虚拟网络栈，该接口将从当前环境消失，
    对 jail 可见。
```

```sh
# TAP=$(ifconfig tap create)
# ifconfig $TAP
```

tap0: flags=8802<BROADCAST,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=80000<LINKSTATE>
        ether 00:bd:70:98:00:00
        groups: tap
        media: Ethernet autoselect
        status: no carrier
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>

```sh
# ifconfig $TAP vnet useless
# ifconfig $TAP
```

ifconfig: interface tap0 does not exist

发生了什么？我们启用接口并分配给 Jail 之后，它就消失了！这是预期行为，因为这个接口不再属于 host 网络栈。你可以在 Jail 上检查其状态，甚至为其分配 IP 地址：

```sh
# jexec useless ifconfig $TAP
```

tap0: flags=8802<BROADCAST,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=80000<LINKSTATE>
        ether 00:bd:70:98:00:00
        groups: tap
        media: Ethernet autoselect
        status: no carrier
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>

```sh
# jexec useless ifconfig $TAP inet 192.0.2.1/24 up
# jexec useless ifconfig $TAP inet
```

tap0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=80000<LINKSTATE>
        inet 192.0.2.1 netmask 0xffffff00 broadcast 192.0.2.255

```sh
# jexec useless ping -c 2 192.0.2.1
```

PING 192.0.2.1 (192.0.2.1): 56 data bytes
64 bytes from 192.0.2.1: icmp_seq=0 ttl=64 time=0.248 ms
64 bytes from 192.0.2.1: icmp_seq=1 ttl=64 time=0.525 ms
--- 192.0.2.1 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.248/0.387/0.525/0.139 ms

```sh
# jexec useless arp -na
```

? (192.0.2.1) at 00:bd:70:98:00:00 on tap0 permanent [ethernet]

```sh
# arp -na | grep 192.0.2.1
```

## 验证网络栈隔离

Jail 可以 ping 自己的接口，自己的 ARP 缓存填充了相应条目，且这些都与 host 网络栈隔离。

路由表同理：

```sh
# jexec useless route add -net 198.51.100.0/24 192.0.2.2
```

add net 198.51.100.0: gateway 192.0.2.2

```sh
# jexec useless netstat -4rn
```

Routing tables
Internet:
Destination        Gateway            Flags     Netif Expire
127.0.0.1          link#1             UH          lo0
192.0.2.0/24       link#2             U          tap0
192.0.2.1          link#2             UHS         lo0
198.51.100.0/24    192.0.2.2          UGS        tap0

```sh
# netstat -4rn | grep 198.51.100.0
```

## 清理现有 Jail

继续下一个示例前，我们清理现有 Jail 并销毁 tap 接口。需要使用 `-R`（大写）选项来移除不带配置文件创建的 Jail。使用 `-r`（小写）选项时，vnet 接口不会自动归还给 host。

```sh
# jail -R useless
# ifconfig $TAP destroy
```

## 与 host 相连的 vnet Jail

本示例展示如何在 Jail 与 host 本身之间通信。主要问题是：任何放入 vnet 的接口都会从 host 网络栈消失。

因此，我们可能设想这样的设置：

1. 创建 bridge 并为其分配 IP 地址
2. 创建 tap 接口并加入 bridge
3. 将 tap 接口分配给 jvnet Jail

但这行不通。TAP 接口会从 host（=从 host 消失）移入 Jail 的 vnet，因此也会从 bridge 消失！

为解决此问题，epair(4) 接口（Ethernet pair）应运而生。这种特殊网络接口表现为两个接口（epairXa 与 epairXb），它们的行为如同交叉相连的两个 Ethernet 接口。将每一侧分配到不同 vnet 后，二者仍能相互交换帧。

host
jail1
epair0a
epair0b

首先创建一对新的 epair。

```sh
# ifconfig epair create
```

epair0a

```sh
# ifconfig -g epair
```

epair0b
epair0a

host 显示其两个新接口：epair0a 与 epair0b。

创建名为 "jvnet" 的新 Jail，并将接口 epair0b 分配给它。

```sh
# jail -c name=jvnet host.hostname=jvnet persist vnet vnet.interface=epair0b
```

接口 epair0b 不再属于 host 系统网络栈，但 epair0a 仍属于 host！我们在 epair0a 上配置 IP 地址。

```sh
# ifconfig -g epair
```

epair0a

```sh
# ifconfig epair0a inet 192.0.2.1/24 up
```

然后在属于 Jail 的 epair0b 上做同样的事，并检查它们的连通性。

```sh
# jexec jvnet ifconfig epair0b inet 192.0.2.2/24 up
# ping -c 2 192.0.2.2
```

PING 192.0.2.2 (192.0.2.2): 56 data bytes
64 bytes from 192.0.2.2: icmp_seq=0 ttl=64 time=0.285 ms
64 bytes from 192.0.2.2: icmp_seq=1 ttl=64 time=0.532 ms
--- 192.0.2.2 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.285/0.408/0.532/0.124 ms

```sh
# arp -na | grep 192.0.2.
```

? (192.0.2.2) at 02:77:29:04:9e:0b on epair0a expires in 1139 seconds [ethernet]
? (192.0.2.1) at 02:77:29:04:9e:0a on epair0a permanent [ethernet]

通过显示它们的 MAC 地址，你会注意到 epair 使用了特定的 MAC 地址。

继续下一个示例前，销毁此 Jail 与 epair 接口对：

```sh
# jail -R jvnet
# ifconfig epair0a destroy
```

基础设置已能工作，下面用多个串行配置且带路由的 Jail 稍微复杂化一些。

## 链式路由 vnet Jail

本示例中我们将创建四个 Jail，将流量从 host 路由到第五个 Jail。

host
jail1
jail2
jail3
jail4
jail5
epair0a
192.0.2.0/31
epair0b
192.0.2.1/31
epair1a
192.0.2.2/31
epair1b
192.0.2.3/31
epair2a
192.0.2.4/31
epair2b
192.0.2.5/31
epair4b
192.0.2.9/31
epair4a
192.0.2.8/31
epair3b
192.0.2.7/31
epair3a
192.0.2.6/31
static route
192.0.2.0/24 → 192.0.2.1
static route
default → 192.0.2.8
static route
default → 192.0.2.3
static route
default → 192.0.2.5
192.0.2.0/31 → 192.0.2.2
static route
default → 192.0.2.4
192.0.2.8/31 → 192.0.2.7
static route
default → 192.0.2.6

用第一个循环生成五个 epair。

```sh
# for i in $(jot 5); do ifconfig epair create; done
```

epair0a
epair1a
epair2a
epair3a
epair4a

然后用循环与手动分配混合生成五个 Jail，并将 epair 分配给它们。

```sh
# for i in $(jot 4); do jail -c name=hop$i host.hostname=hop$i persist vnet \
vnet.interface=epair$((i-1))b vnet.interface=epair${i}a; done
# jail -c name=hop5 host.hostname=hop5 persist vnet vnet.interface=epair4b
# jls
```

   JID  IP Address      Hostname                      Path
     3                  hop1                          /
     4                  hop2                          /
     5                  hop3                          /
     6                  hop4                          /
     7                  hop5                          /

现在配置 IP 地址、在某些 Jail 上启用路由，并设置静态路由。

```sh
# ifconfig epair0a inet 192.0.2.0/31 up
# jexec hop1 ifconfig epair0b inet 192.0.2.1/31 up
# jexec hop1 ifconfig epair1a inet 192.0.2.2/31 up
# jexec hop2 ifconfig epair1b inet 192.0.2.3/31 up
# jexec hop2 ifconfig epair2a inet 192.0.2.4/31 up
# jexec hop3 ifconfig epair2b inet 192.0.2.5/31 up
# jexec hop3 ifconfig epair3a inet 192.0.2.6/31 up
# jexec hop4 ifconfig epair3b inet 192.0.2.7/31 up
# jexec hop4 ifconfig epair4a inet 192.0.2.8/31 up
# jexec hop5 ifconfig epair4b inet 192.0.2.9/31 up
# for i in $(jot 4); do jexec hop$i sysctl net.inet.ip.forwarding=1; done
```

net.inet.ip.forwarding: 0 -> 1
net.inet.ip.forwarding: 0 -> 1
net.inet.ip.forwarding: 0 -> 1
net.inet.ip.forwarding: 0 -> 1

```sh
# route add 192.0.2.0/24 192.0.2.1
```

add net 192.0.2.0: gateway 192.0.2.1

```sh
# jexec hop1 route add default 192.0.2.3
```

add net default: gateway 192.0.2.3

```sh
# jexec hop2 route add default 192.0.2.5
```

add net default: gateway 192.0.2.5

```sh
# jexec hop2 route add 192.0.2.0/31 192.0.2.2
```

add net 192.0.2.0: gateway 192.0.2.2

```sh
# jexec hop3 route add default 192.0.2.4
```

add net default: gateway 192.0.2.4

```sh
# jexec hop3 route add 192.0.2.8/31 192.0.2.7
```

add net 192.0.2.8: gateway 192.0.2.7

```sh
# jexec hop4 route add default 192.0.2.6
```

add net default: gateway 192.0.2.6

```sh
# jexec hop5 route add default 192.0.2.8
```

add net default: gateway 192.0.2.8

从 host 网络栈 ping 第五个 Jail 测试设置。

```sh
# ping -c 2 192.0.2.9
```

PING 192.0.2.9 (192.0.2.9): 56 data bytes
64 bytes from 192.0.2.9: icmp_seq=0 ttl=60 time=0.265 ms
64 bytes from 192.0.2.9: icmp_seq=1 ttl=60 time=0.482 ms
--- 192.0.2.9 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.265/0.373/0.482/0.108 ms

```sh
# traceroute -n 192.0.2.9
```

traceroute to 192.0.2.9 (192.0.2.9), 64 hops max, 40 byte packets
  1  192.0.2.1  0.060 ms  0.243 ms  0.244 ms
  2  192.0.2.3  0.180 ms  0.202 ms  0.263 ms
  3  192.0.2.5  0.050 ms  0.159 ms  0.205 ms
  4  192.0.2.7  0.194 ms  0.197 ms  0.191 ms
  5  192.0.2.9  0.261 ms  0.201 ms  0.188 ms

可选地，给此设置增添一点趣味（在拥有较大 IPv6 范围时，应能轻松编写一个填充 DNS 配置文件的 text-to-traceroute 脚本）。

```sh
# cat >> /etc/hosts <<EOF
```

? 192.0.2.1 once.upon.a.time.in.the.middle
? 192.0.2.3 of.winter.when.the.flakes.of
? 192.0.2.5 snow.were.failling.like
? 192.0.2.7 feathers.from.the.sky
? 192.0.2.9 a.queen.sat.at.a.window.sewing
? EOF

```sh
# traceroute 192.0.2.9
```

traceroute to 192.0.2.9 (192.0.2.9), 64 hops max, 40 byte packets
  1  once.upon.a.time.in.the.middle (192.0.2.1) 0.237 ms  0.508 ms  0.540 ms
  2  of.winter.when.the.flakes.of (192.0.2.3) 0.489 ms  0.361 ms  0.373 ms
  3  snow.were.failling.like (192.0.2.5) 0.343 ms  0.337 ms  0.285 ms
  4  feathers.from.the.sky (192.0.2.7) 0.255 ms  0.296 ms  0.271 ms
  5  a.queen.sat.at.a.window.sewing (192.0.2.9) 0.328 ms  0.271 ms  0.242 ms

## 进一步探索

## 将 Jail 与外部世界相连

此处有多个选项：

1. 使用兼容 SR-IOV 的 NIC，生成多个 Virtual NIC 并分配给 Jail。
2. Virtual-Interface（驱动特定）。
3. 使用 VLAN 并将 VLAN 接口分配给每个 Jail，限制是每个 Ethernet 端口每个 VLAN 最多一个 Jail。
4. 使用 if_bridge 接口（也可与 VLAN 混用）是最简单的设置，但使用 if_bridge 会有一些性能损失。

### SR-IOV

这一最初为虚拟机设计的特性会创建多个 Virtual Function（VF = 我们的场景下即虚拟 NIC）。使用默认的非 passthrough 模式时，它向 host 呈现多个虚拟 NIC，每一个都可附加到 vnet-jail。

下面是一个使用两块 Chelsio 接口（cxl0 与 cxl1）为每块创建 10 个 VF 的示例。

```sh
# sysrc iovctl_files="/etc/iovctl.cxl0.conf /etc/iovctl.cxl1.conf"
# sysrc kld_list+=if_cxgbev
# cat > /etc/iovctl.cxl0.conf <<EOF
```

? PF {
? device : "cxl0";
? num_vfs : 10;
? }
? EOF

```sh
# cat > /etc/iovctl.cxl1.conf <<EOF
```

? PF {
? device : "cxl1";
? num_vfs : 10;
? }
? EOF

```sh
# iovctl -C -f /etc/iovctl.cxl0.conf
# iovctl -C -f /etc/iovctl.cxl1.conf
# kldload if_cxgbev
# tail /var/log/messages
```

(...) kernel: t5vf18: <Chelsio T540-CR VF> at device 0.41 on pci4
(...) kernel: cxlv18: <port 0> on t5vf18
(...) kernel: cxlv18: Ethernet address: 06:44:2e:e5:90:18
(...) kernel: cxlv18: 2 txq, 1 rxq (NIC)
(...) kernel: t5vf18: 1 ports, 2 MSI-X interrupts, 4 eq, 2 iq
(...) kernel: t5vf19: <Chelsio T540-CR VF> at device 0.45 on pci4
(...) kernel: cxlv19: <port 0> on t5vf19
(...) kernel: cxlv19: Ethernet address: 06:44:2e:e5:90:19
(...) kernel: cxlv19: 2 txq, 1 rxq (NIC)
(...) kernel: t5vf19: 1 ports, 2 MSI-X interrupts, 4 eq, 2 iq

```sh
# ifconfig -l
```

cxl0 cxl1 igb0 lo0 cxlv0 cxlv1 cxlv2 cxlv3 cxlv4 cxlv5 cxlv6 cxlv7 cxlv8 cxlv9 cxlv10
cxlv11 cxlv12 cxlv13 cxlv14 cxlv15 cxlv16 cxlv17 cxlv18 cxlv19

现在可以保留 cxl0（=物理接口）给 host，所有 cxlvX 接口可分配给不同的 jvnet-jail。

请注意：

1. 某些 NIC（如 Intel）需要更多参数（`allow-promisc`、`allow-set-mac`、`mac-antispoof`）以允许在 VF 上进行如 CARP 之类的特定用法。
2. 使用 Intel ix(4) 驱动的 FreeBSD 12 无法将驱动附加到这些 VF。

```sh
ixv0: <Intel(R) PRO/10GbE Virtual Function Network Driver> at device 0.128 on pci4
ixv0: ...reset_hw() failure: Reset Failed!
ixv0: IFDI_ATTACH_PRE failed 5
device_attach: ixv0 attach returned 5
```

### Virtual-Interface（驱动特定）

Chelsio 驱动支持另一种称为 Virtual Interface 的模式，下面是请求每端口创建四个 VI 的示例。

```sh
# echo hw.cxgbe.num_vis=\"4\" >> /boot/loader.conf
```

重启后新接口（vcxlX）将可用，并在 dmesg 中显示为：

```sh
vcxl0: <port 0 vi 1> on cxl0
vcxl0: Ethernet address: 00:07:43:2e:e5:91
vcxl0: netmap queues/slots: TX 2/1023, RX 2/1024
vcxl0: 1 txq, 1 rxq (NIC); 2 txq, 2 rxq (netmap)
vcxl1: <port 0 vi 2> on cxl0
vcxl1: Ethernet address: 00:07:43:2e:e5:92
vcxl1: netmap queues/slots: TX 2/1023, RX 2/1024
vcxl1: 1 txq, 1 rxq (NIC); 2 txq, 2 rxq (netmap)
(...)
```

现在可以将这些 Virtual Interface（vxclX）分配给 vnet-jail。

### VLAN

在没有支持 SR-IOV 或 Virtual-Interface 特性的 NIC 时，另一种可能是创建多个 VLAN 并将 VLAN 接口分配给 vnet-jail。限制是 VLAN ID 在每个接口上唯一，因此如果两个 vnet-jail 需要在同一 VLAN 中分配 vlan 子接口，你需要使用两块物理接口。

```sh
# ifconfig igb0.6 create vlan 6 vlandev igb0 up
# ifconfig igb0.7 create vlan 7 vlandev igb0 up
# ifconfig igb1.6 create vlan 6 vlandev igb1 up
```

此示例中，新接口 igb0.6、igb0.7 与 igb1.6 可用于 vnet-jail。

### Bridge + epair

为消除每物理接口 VLAN ID 唯一的限制，仍有经典方案：使用 bridge 与 epair 设置，但有一些性能影响。

FreeBSD host
igb0
epair1a
epair2a
epair3a
bridge0
Jail1
epair1b
Jail2
epair2b
Jail3
epair3b

此小型示意图由这些命令生成：

```sh
# ifconfig bridge create up
```

bridge0

```sh
# for i in $(jot 3); do ifconfig epair$i create up; done
```

epair1a
epair2a
epair3a

```sh
# ifconfig bridge0 inet 192.0.2.254/24 addm igb1 addm epair1a addm epair2a \
addm epair3a
# for i in $(jot 3); do jail -c name=jail$i host.hostname=jail$i persist vnet \
vnet.interface=epair${i}b; jexec jail$i ifconfig epair${i}b inet \
192.0.2.${i}/24 up; jexec jail$i ifconfig epair${i}b inet; done
```

epair1b: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=8<VLAN_MTU>
        inet 192.0.2.1 netmask 0xffffff00 broadcast 192.0.2.255
epair2b: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=8<VLAN_MTU>
        inet 192.0.2.2 netmask 0xffffff00 broadcast 192.0.2.255
epair3b: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=8<VLAN_MTU>
        inet 192.0.2.3 netmask 0xffffff00 broadcast 192.0.2.255

```sh
# ping -c 2 192.0.2.1
```

PING 192.0.2.1 (192.0.2.1): 56 data bytes
64 bytes from 192.0.2.1: icmp_seq=0 ttl=64 time=0.158 ms
64 bytes from 192.0.2.1: icmp_seq=1 ttl=64 time=0.103 ms
--- 192.0.2.1 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.103/0.130/0.158/0.027 ms

```sh
# ping -c 2 192.0.2.2
```

PING 192.0.2.2 (192.0.2.2): 56 data bytes
64 bytes from 192.0.2.2: icmp_seq=0 ttl=64 time=0.189 ms
64 bytes from 192.0.2.2: icmp_seq=1 ttl=64 time=0.104 ms
--- 192.0.2.2 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.104/0.146/0.189/0.042 ms

```sh
# ping -c 2 192.0.2.3
```

PING 192.0.2.3 (192.0.2.3): 56 data bytes
64 bytes from 192.0.2.3: icmp_seq=0 ttl=64 time=0.201 ms
64 bytes from 192.0.2.3: icmp_seq=1 ttl=64 time=0.091 ms
--- 192.0.2.3 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.091/0.146/0.201/0.055 ms

清理此示例。

```sh
# for i in $(jot 3); do jail -R jail$i; ifconfig epair${i}a destroy; done
# ifconfig bridge0 destroy
```

## 最终练习

现在你应该能独立搭建此类实验环境。

FreeBSD host
igb0
epair1a
epair2a
vlan2
vlan3
epair3a
epair4a
epair51a
epair52a
epair5a
epair53a
epair54a
bridge2
bridge3
bridge5
Jail1
epair1b
epair51b
Jail2
epair2b
epair52b
Jail5
epair5b
Jail3
epair3b
epair53b
Jail4
epair4b
epair54b

## Jail 比 VM 轻多少？

让我们使用 PC Engines APU2 这样的小型设备（4 核 AMD GX-412TC SOC 1Ghz、4GB 内存、16GB 闪存）。在这台设备上，我们能启动多少个运行真实进程（用 bird 通过 OSPF 通告每个 Jail 的 loopback）的 Jail？

jail1
jail2
jail254
epair1b
192.0.2.1/24
epair1a
epair2a
epair2a
bridge0
lo1
198.51.100.1/32
lo1
198.51.100.2/32
lo1
198.51.100.254/32
epair2b
192.0.2.2/24
epair254b
192.0.2.254/24

以下小型 shell 脚本用于启动 480 个 Jail，每个 Jail 运行调整过的 bird OSPF，以便在共享链路上使用大量邻居：

- MTU 增至 9000，以容纳大量邻居（默认 1500 字节 MTU 下最多只能有约 350 个 OSPF 邻居）
- Hello 与 dead 间隔增大，以减少 bridge 接口上的多播风暴

```sh
#!/bin/sh
```

set -eu
dec2dot () {

```sh
    # $1 是十进制数
    # 输出点分十进制（IP 地址格式）
    printf '%d.%d.%d.%d\n' $(printf "%x\n" $1 | sed 's/../0x& /g')
```

}

```sh
# 需要略微调高一些网络值
# 以避免 "No buffer space available" 报错
# mbuf cluster 最大允许数
sysctl kern.ipc.nmbclusters=1000000
sysctl net.inet.raw.maxdgram=16384
sysctl net.inet.raw.recvspace=16384
# 共享 LAN 从 192.0.2.0 起编址（用十进制便于递增）
```

ipepairbase=3221225984

```sh
# loopback 从 198.51.100.0 起编址
```

iplobase=3325256704

```sh
ifconfig bridge create name vnetdemobridge mtu 9000 up
for i in $(jot 480); do
    ifconfig epair$i create mtu 9000 up
    ifconfig vnetdemobridge addm epair${i}a edge epair${i}a
    jail -c name=jail$i host.hostname=jail$i persist \
         vnet vnet.interface=epair${i}b
    ipdot=$( dec2dot $(( iplobase + i)) )
    jexec jail$i ifconfig lo1 create inet ${ipdot}/32 up
    ipdot=$( dec2dot $(( ipepairbase + i)) )
    jexec jail$i ifconfig epair${i}b inet ${ipdot}/20 mtu 9000 up
    cat > /tmp/bird.${i}.conf <<EOF
```

protocol device {}
protocol kernel { ipv4 { export all; }; }
protocol ospf {
  area 0 {
    interface "epair${i}b" {
      hello 60;
      dead 240;
    };
    interface "lo1" {
      stub yes;
    };
  };
}
EOF

```sh
    jexec jail$i bird -c /tmp/bird.$i.conf -P /tmp/bird.$i.pid \
          -s /tmp/bird.$i.ctl -g birdvty
```

done

安装 net/bird2，执行此脚本，OSPF DR/BDR 选举与数据库同步之后，bridge 接口上的网络流量仍应相当高，仅剩 OSPF keep-alive：

```sh
# netstat -ihw 1 -I vnetdemobridge
```

```sh
            input vnetdemobridge           output
   packets  errs idrops      bytes    packets  errs      bytes colls
       490     0     0       826K        29k     0       7.0M     0
       981     0     0       1.7M        65k     0        22M     0
      1.5k     0     0       3.3M        92k     0        25M     0
       596     0     0       337K       102k     0        35M     0
       732     0     0       479K       100k     0        33M     0
```

几分钟后检查检测到的邻居数（应为 479）。DR/BDR 选举应选 jail479 为 BDR、jail480 为 DR，并查看已学到的路由数。

```sh
# birdcl -s /tmp/bird.1.ctl show ospf
```

BIRD 2.0.6 ready.
ospf1:
RFC1583 compatibility: disabled
Stub router: No
RT scheduler tick: 1
Number of areas: 1
Number of LSAs in DB:   481
        Area: 0.0.0.0 (0) [BACKBONE]
                Stub:   No
                NSSA:   No
                Transit:        No
                Number of interfaces:   2
                Number of neighbors:    479
                Number of adjacent neighbors:   2

```sh
# jexec jail1 netstat -4rn | grep UGH1 | wc -l
```

此项测试的当前系统限制源于所有 bird 进程消耗的 4GB 内存。

```sh
last pid: 16459;  load averages: 34.76, 37.35, 28.18
up 0+00:36:15  08:36:46
497 processes: 1 running, 496 sleeping
CPU: 14.4% user,  0.0% nice,  6.3% system, 10.2% interrupt, 69.2% idle
Mem: 1177M Active, 454M Inact, 2816K Laundry, 1891M Wired, 17M Buf, 395M Free
  PID USERNAME    THR PRI NICE   SIZE    RES STATE    C   TIME    WCPU COMMAND
13529 root          1  20    0    32M    20M select   1   0:45   2.04% bird
13553 root          1  20    0    29M    17M select   3   0:41   0.71% bird
16459 root          1  20    0    14M  3568K CPU1     1   0:00   0.62% top
13512 root          1  20    0    20M  7372K select   2   0:03   0.39% bird
 8003 root          1  20    0    20M  7316K select   0   0:03   0.38% bird
 7913 root          1  20    0    20M  7260K select   0   0:03   0.38% bird
13466 root          1  20    0    20M  7172K select   0   0:03   0.34% bird
 7887 root          1  20    0    20M  7288K select   1   0:03   0.33% bird
 7832 root          1  20    0    20M  7260K select   2   0:03   0.33% bird
```

下面是删除/清理所有 Jail 的脚本，但你应该重启系统，因为清理过程中系统可能会 panic：

```sh
#!/bin/sh
```

set -eu

```sh
for i in $(jot 480); do
    echo Deleting jail$i
    jail -R jail$i
    ifconfig epair${i}a destroy
    rm /tmp/bird.$i.*
```

done

```sh
ifconfig vnetdemobridge destroy
```

## 防火墙

pf 与 ipfw 兼容 vnet，可在 HA 场景下构建多租户防火墙，如下所示：

FreeBSD host1
FreeBSD host2
jail1
(master)
epair1a
epair1b
bridge0
igb1.vlan1
igb1.vlan2
pfsync between
each pairs
Customer 1
(root access on jail 11 and 21)
Customer 2
(root access on jail 12 and 22)
igb0
igb1
igb1
igb0
igb1.vlan1
igb1.vlan2
bridge0
epair2a
epair2b
epair1a
epair1b
epair2a
epair2b
jail2
jail1
jail1

此设置更为复杂，因为需要启用特定的内核特性才能作为 Jail 使用。（此类设置的详细示例可在未来文章中介绍。）同时，此设置作为 BSD Router Project 的示例在此说明（使用一些辅助脚本配置 Jail，隐藏了复杂性）：

<https://bsdrp.net/documentation/examples/multi-tenant_ha_pf_firewalls>

OLIVIER COCHARD-LABBÉ 在 2005 年通过定制 m0n0wall 创建 FreeNAS 而接触 FreeBSD。作为网络工程师，他于 2009 年创建 BSD Router Project，此后一直致力于测试 FreeBSD 网络栈。他于 2016 年获得 Ports commit 权限，目前是 Netflix 的测试软件开发者。
