# 配置自己的 VPN——基于 FreeBSD、Wireguard、IPv6 和广告拦截


- 原文链接：[Make Your Own VPN —FreeBSD, Wireguard, IPv6 and Ad-blocking Included](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/make-your-own-vpn-freebsd-wireguard-ipv6-and-ad-blocking-included/)
- 作者：Stefano Marinelli


>**注意**
>
>本文操作配置基于 FreeBSD。如果你想要基于 OpenBSD 的版本，请点击[这里](https://it-notes.dragas.net/2023/04/03/make-your-own-vpn-wireguard-ipv6-and-ad-blocking-included/)查看。

VPN 是一种基础工具，用于安全地连接到自己的服务器和设备。许多人出于各种原因使用商业 VPN，从不信任自己的服务提供商（尤其通过公共热点连接时），到希望用不同的 IP 地址（可能是来自别国）来“上网”。在这儿，我想突出一些已被引入基础堆栈的新特性——其中许多是默认开启的，有些可能需要专门打开。每个功能都会介绍一些细节，帮助改善网络体验。

无论出于何种原因，解决方案从未匮乏。我一直在设置管理 VPN，以便服务器/客户端使用安全通道相互通信。最近，我[已在所有设备上启用 IPv6 连接](https://my-notes.dragas.net/posts/2023/the-urgency-of-transitioning-to-ipv6/)（包括桌面/服务器和移动设备），并且我需要快速创建一个节点，将一些网络聚合在一起，并让它们通过 IPv6 连接到外部网络。我使用着、并将要介绍的工具有：

- **VPS** – 在本例中，我使用了基本的 Hetzner Cloud VPS，但所有提供 IPv6 连接的服务商都可以——如果你的确需要 IPv6。
- **[FreeBSD](https://www.freebsd.org/)** – 一款多功能、稳定和安全的操作系统。
- **[WireGuard](https://www.wireguard.com/)** – 轻量级、安全，并且不会占用太多带宽，所以在移动设备上也比较省电。当没有流量时，它完全不会传输/接收任何数据。在所有主要桌面和服务器操作系统以及 Android 和 iOS 设备上支持良好。
- **[Unbound](https://nlnetlabs.nl/projects/unbound/about/)** – 可以直接向根 DNS 服务器发起查询，而非转发器。它还允许插入拦截列表，产生类似 Pi-Hole 的效果（即广告拦截）。
- **[SpamHaus](https://www.spamhaus.org/)** 列表 – 立即阻断与黑名单用户的连接。

### 步骤 1：激活 VPS 并安装 FreeBSD

首先，启用一台 VPS，再安装 FreeBSD。在 Hetzner Cloud 控制台中，可能没有预构建的 FreeBSD 镜像，只能选 Linux 发行版。别担心，随便选择一款 Linux 发行版创建 VPS。创建完成后，FreeBSD ISO 镜像将出现在“ISO 镜像”中。只需插入虚拟光驱，重启 VPS，FreeBSD 安装程序就会出现在控制台中。

我不会详细说明，操作非常简单。唯一需要注意的一点是，在 Hetzner Cloud VPS 中，IPv4 使用“DHCP”进行配置，但暂时别配置 IPv6。IPv6 将在稍后配置。

安装所有的 FreeBSD 更新（使用 `freebsd-update fetch install` 命令）并重启。

### 步骤 2：安装 WireGuard

在 FreeBSD 上，现在 WireGuard 作为内核模块提供，可以用包管理器通过 `pkg install wireguard-tools` 安装用户空间工具。这意味着你可以轻松地将它与系统上的其他软件一起更新。

### 步骤 3：配置 VPS 上的 IPv6

首先配置 VPS 上的 IPv6。对于 Hetzner，遗憾的是，他们只提供了一个 `/64` 地址，因此需要对分配的网络进行细分。在这个示例中，它将被细分为 `/72` 子网——可使用[子网计算器](https://subnettingpractice.com/ipv6-subnet-calculator.html)来找到有效的子类。

在 `/etc/rc.conf` 文件中添加类似以下的条目：

```sh
ifconfig_vtnet0=”DHCP”
ifconfig_vtnet0_ipv6=”inet6 2a01:4f8:cafe:cafe::1 prefixlen 72”
ipv6_defaultrouter=”fe80::1%vtnet0”
```

简而言之，保留 Hetzner 分配的基础地址，但将前缀长度更成 72——这样就可以拥有其他可用网络。现在，需要打开 IPv4 和 IPv6 的转发功能。将以下行添加到 `/etc/sysctl.conf` 文件中：

```sh
net.inet.ip.forwarding=1
net.inet6.ip6.forwarding=1
```

重启后，测试是否配置成功：

```sh
ping6 google.com
```

如果一切配置正确，ping 命令将执行且 google.com 会回复。

### 步骤 4：配置 WireGuard

接下来，需要进行一些步骤来配置 WireGuard。首先，生成私钥：

```sh
wg genkey | tee /dev/stderr | wg pubkey | grep --label PUBLIC -H .
```

这将生成私钥和公钥。记下公钥——它将在配置客户端时用到。

接着创建一个新的配置文件 `/usr/local/etc/wireguard/wg0.conf`：

```sh
[Interface]
Address = 172.14.0.1/24,2a01:4f8:cafe:cafe:100::1/72
ListenPort = 51820
PrivateKey = YUkS6cNTyPbXmtVf/23ppVW3gX2hZIBzlHtXNFRp80w=
```

这将创建一个新的 WireGuard 接口 `wg0`。启动 WireGuard 接口：

```sh
service wireguard enable
sysrc wireguard_interfaces=”wg0”
service wireguard start
```

如果所有信息输入正确，接口应已启动。可以检查其状态：

```sh
wg
```

至于防火墙，FreeBSD 默认未配置 `pf`。在我的设置中，我倾向于阻断不需要的流量，而对可能有用的流量保持宽松。然而，我喜欢把“坏家伙”挡在外面，因此我使用黑名单。`pf` 允许在运行时将元素插入和移出表格，所以防火墙可以根据需要进行配置。

为了下载、应用 Spamhaus 列表，我使用了一个简单但有效的脚本，可以在网上找到它，但原本是为 OpenBSD 设计的。

对于 Spamhaus 列表，继续创建 FreeBSD 脚本。

### 创建脚本 `/usr/local/sbin/spamhaus.sh`

```sh
#!/bin/sh
# 这个脚本通常随 cron 每天运行一次。
#
echo updating Spamhaus DROP lists:
(
  { fetch -o - https://www.spamhaus.org/drop/drop.txt && \
    fetch -o - https://www.spamhaus.org/drop/edrop.txt && \
    fetch -o - https://www.spamhaus.org/drop/dropv6.txt ; \
  } 2>/dev/null | sed “s/;/#/” > /var/db/drop.txt
)
pfctl -t spamhaus -T replace -f /var/db/drop.txt
```

使脚本可执行并运行。由于 `pf` 尚未启用，可能会报错——但是现在创建 `/var/db/drop.txt` 文件：

```sh
chmod a+rx /usr/local/sbin/spamhaus.sh
/usr/local/sbin/spamhaus.sh
```

### 配置 `pf` 防火墙

FreeBSD 上有很多种方式可以配置 `pf`。以下是一个相对简单的示例：

```sh
ext_if="vtnet0"
wg0_if="wg0"
wg0_networks="172.14.0.0/24"

set skip on lo

nat on $ext_if from { $wg0_networks } to any -> ($ext_if)

# Spamhaus 删除列表：
table <spamhaus> persist file "/var/db/drop.txt"

block drop log quick from <spamhaus>

# ipv6 通过 ICMP 
pass quick proto ipv6-icmp
# Block from ipv6 to wg0 network
block in quick on $ext_if inet6 to { 2a01:4f8:cafe:cafe:100::/72 }
# Pass Wireguard traffic - in and out
pass quick on $wg0_if

# 默认拒绝
block in
block out

pass in on $ext_if proto tcp to port ssh
pass in on $ext_if proto udp to port 51820

pass out on $ext_if
```

这是一个非常简单的配置：它阻断了从 Spamhaus 下载的列表中列出的所有流量，允许 WireGuard 网络通过 NAT 访问公共接口，允许 IPv6 的 ICMP 流量（这是网络正常运行所必需的），同时阻断进入 WireGuard IPv6 网络的流量（记住，IP 地址是公开的并且可以直接访问，所以我们默认不想暴露我们的设备）。所有通过 WireGuard 接口的流量都会被允许通过。然后，所有其他流量都将被阻止，并且会指定一些例外规则，比如允许 SSH 和 WireGuard 连接（当然）。还会允许流量从公共网络接口外发。

将此配置保存到 `/etc/pf.conf` 文件中。

启用并启动 `pf`：

```sh
service pf enable
service pf start
```

你可能会被系统踢出去。别担心，只需重新连接即可。`pf` 正在执行其工作。

若一帆风顺，防火墙应该已经加载了新规则。

### 配置 DNS 缓存和广告拦截

现在是配置 `Unbound` 以缓存 DNS 查询并启用广告拦截的时间了。首先，我们需要安装 `Unbound`：

```sh
pkg install unbound
```

不久前，我找到了一个脚本并稍作修改。我记不清楚原始作者是谁了，所以我只在这里贴出来脚本。

### 创建更新 Unbound 广告拦截的脚本 `/usr/local/sbin/unbound-adhosts.sh`

```sh
#!/bin/sh
#
# 使用来自 Pi-Hole 项目的黑名单 https://github.com/pi-hole/ 
# 来启用 Unbound(8) 中的广告拦截
#
PATH=”/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin”

# 可用的黑名单 - 注释掉行以禁用
_disconad=”https://s3.amazonaws.com/lists.disconnect.me/simple_ad.txt”
_discontrack=”https://s3.amazonaws.com/lists.disconnect.me/simple_tracking.txt”
_stevenblack=”https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts”

# 全局变量
_tmpfile=”$(mktemp)” && echo '' > $_tmpfile
_unboundconf=”/usr/local/etc/unbound/unbound-adhosts.conf”

# 从黑名单中移除注释
simpleParse() {
fetch -o - $1 | \
sed -e ‘s/#.*$//’ -e ‘/^[[:space:]]*$/d’ >> $2
}

# 解析 DisconTrack
[ -n “${_discontrack}” ] && simpleParse $_discontrack $_tmpfile

# 解析 DisconAD
[ -n “${_disconad}” ] && simpleParse $_disconad $_tmpfile

# 解析 StevenBlack
[ -n “${_stevenblack}” ] && \
  fetch -o - $_stevenblack | \
  sed -n '/Start/,$p' | \
  sed -e 's/#.*$//' -e '/^[[:space:]]*$/d' | \
  awk '/^0.0.0.0/ { print $2 }' >> $_tmpfile

# 创建 unbound(8) 局域网文件
sort -fu $_tmpfile | grep -v “^[[:space:]]*$” | \
awk '{
  print “local-zone: \”” $1 “\” redirect”
  print “local-data: \”” $1 “ A 0.0.0.0\””
}' > $_unboundconf && rm -f $_tmpfile

service unbound reload 1>/dev/null

exit 0
```

保存脚本后，使其可执行，然后运行：

```sh
chmod a+rx /usr/local/sbin/unbound-adhosts.sh
/usr/local/sbin/unbound-adhosts.sh
```

现在，可以按以下方式修改 `/usr/local/etc/unbound/unbound.conf` 配置文件：

```sh
server:
        verbosity: 1
        log-queries: no
        num-threads: 4
        num-queries-per-thread: 1024
        interface: 127.0.0.1
        interface: 172.14.0.1
        interface: 2a01:4f8:cafe:cafe:100::1
        interface: ::1
        outgoing-range: 64
        chroot: “”

        access-control: 0.0.0.0/0 refuse
        access-control: 127.0.0.0/8 allow
        access-control: ::0/0 refuse
        access-control: ::1 allow
        access-control: 172.14.0.0/24 allow
        access-control: 2a01:4f8:cafe:cafe:100::/72 allow

        hide-identity: yes
        hide-version: yes
        auto-trust-anchor-file: "/usr/local/etc/unbound/root.key"
        val-log-level: 2
        aggressive-nsec: yes
        prefetch: yes
        username: “unbound”
        directory: "/usr/local/etc/unbound"
        logfile: "/var/log/unbound.log"
        use-syslog: no
        pidfile: "/var/run/unbound.pid"
        include: /usr/local/etc/unbound/unbound-adhosts.conf

remote-control:
        control-enable: yes
        control-interface: /var/run/unbound.sock
```

然后启用并启动 `unbound`：

```sh
service unbound enable
service unbound start
```

如果一切设置正确，`unbound` 将能够响应发送到 `172.14.0.1` 和 `2a01:4f8:cafe:cafe:100::1` 的 DNS 请求。接下来，配置 WireGuard 客户端。创建一个新的配置文件，写入 "172.14.0.2/32, 2a01:4f8:cafe:cafe:100::2/128"（这些地址稍后将在服务器的对等配置中使用）。将 DNS 服务器地址设置为 `172.14.0.1` 和其相应的 IPv6 地址（在此示例中为 `2a01:4f8:cafe:cafe:100::1` ——你的地址与此不同）。在对等配置中写入服务器的相关信息，包括其公钥、IP 地址和端口（在此示例中，端口为 51820）以及允许的地址（设置 "`0.0.0.0/0, ::0/0`" 意味着“所有连接将通过 WireGuard”——所有流量将通过 VPN，无论 IPv4 还是 IPv6）。

每种设备的配置程序有所不同（Android、iOS、MikroTik、Linux 等），但基本上只需在服务器和客户端都创建正确的配置即可。

重新打开 WireGuard 配置文件 `/usr/local/etc/wireguard/wg0.conf`，添加：

```sh
[Interface]
Address = 172.14.0.1/24,2a01:4f8:cafe:cafe:100::1/72
ListenPort = 51820
PrivateKey = YUkS6cNTyPbXmtVf/23ppVW3gX2hZIBzlHtXNFRp80w=

[Peer]
PublicKey = *客户端的公钥*
AllowedIPs = 172.14.0.2/32, 2a01:4f8:cafe:cafe:100::2/128
```

客户端的公钥将由客户端显示。

重新加载 WireGuard 配置：

```sh
service wireguard restart
```

如果希望仅使用 VPN 作为广告拦截器，可以仅通过 VPN 路由 DNS 流量。为此，请在客户端配置中仅允许已配置的 Unbound 地址（在此示例中为 `172.14.0.1`、`2a01:4f8:cafe:cafe:100::1`）——DNS 解析将通过 VPN 进行，但浏览将继续通过主要提供商进行。

### 自动更新 Spamhaus 和广告拦截列表

我们将使用 cron 来自动更新列表。首先，创建一个脚本，例如 `/usr/local/sbin/update-blocklists.sh`：

```sh
#!/bin/sh

/usr/local/sbin/unbound-adhosts.sh
/usr/local/sbin/spamhaus.sh
```

使脚本可执行：

```sh
chmod +x /usr/local/sbin/update-blocklists.sh
```

然后，将其添加到 `crontab` 中，以便每天运行：

```sh
echo “@daily /usr/local/sbin/update-blocklists.sh” >> /etc/crontab
```

这种方法从更新管理和安全性的角度都带来了好处。

---

**Stefano Marinelli** 是一位 IT 顾问，拥有二十余年的 IT 咨询、培训、研究和出版经验。他的专业领域涵盖了多种操作系统，尤其专注于 BSD 系统（如 FreeBSD、NetBSD、OpenBSD、DragonFlyBSD）和 Linux 系统。Stefano 还是 BSD Cafe 的咖啡师，这是一家活跃的 BSD 爱好者社区中心。他还领导了博洛尼亚大学的 FreeOsZoo 项目，为虚拟机提供开放源代码操作系统镜像。
