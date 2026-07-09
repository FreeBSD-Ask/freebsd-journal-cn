# 在 FreeBSD 上组织你的虚拟网络

作者：Michael Gmelin

现代 FreeBSD 提供一系列虚拟化选项，从传统的 jail 环境（与宿主操作系统共享网络栈），到 vnet jail（每个 jail 拥有自己的网络栈），再到运行各自内核/操作系统的 bhyve 虚拟机。根据不同需求，配置虚拟网络有不同的方式。jail 和 VM 管理工具通过抽象（至少部分）底层复杂性，可以简化这一过程。

---

## 文档约定

本文基于 FreeBSD 12.1-RELEASE，撰写时为 FreeBSD 最新发行版。假设运行 jail 和 bhyve VM 的宿主机使用 ZFS。

除特别说明外，示例中使用的软件包来自 FreeBSD 2019Q4 季度分支（`https://svnweb.freebsd.org/ports/branches/2019Q4/`）。使用季度软件包仓库是全新安装的 FreeBSD 系统的默认配置。

为简化说明，所有示例仅涉及 IPv4。

本文介绍各种 jail 和 VM 管理工具，尽管所有展示的内容都可以直接修改 **/etc/jail.conf** 等系统配置文件手动配置，无需安装任何软件包。

代码和终端交互/输出格式如下。如果你想复制本文中的代码，可以访问 `https://blog.grem.de/ayvn`。

为清晰起见，示例中有时（并非总是）会展示命令输出。

手册页引用以斜体显示，括号中标注手册章节；如 security(7) 指安全手册页，可输入 `man 7 security` 或 `man security` 显示（因无同名手册页，可省略章节号）。

FreeBSD 软件包引用以斜体显示，使用 Port 的 origin，即 `<分类>/<名称>`，如 `security/sudo`，可作为二进制包安装（`pkg install sudo`），也可从 Ports 安装（`cd /usr/ports/security/sudo && make install clean`）。

## 许可证

本作品采用知识共享署名 4.0 国际许可协议（CC BY 4.0）（`https://creativecommons.org/licenses/by/4.0/`）。

## 普通 Jail

普通（即非 VNET）jail 与所运行的 jailhost 共享网络栈。因此网络配置和防火墙工作在 jailhost 上完成，不在 jail 内部。这是创建 jail 的传统方式，尽管有局限，但在容器化软件、文件系统和服务方面仍然非常有用。例如 `ports-mgmt/poudriere`（FreeBSD 的批量包构建器和 Port 测试工具）大量使用普通 jail。

### 使用继承 IP 配置的普通 Jail

最基础的选项是运行一个从 jailhost 继承网络配置（所有接口/IP 地址）的 jail。如果 jail 主要用作容器以保持 jailhost（运行 jail 的宿主机）”干净”无依赖，这是常见做法。这样宿主机只需最少数量的软件包（基础如 `security/sudo`、`shells/bash`、`sysutils/tmux`），而应用 jail 可单独快照、备份、管理/迁移。

使用 `sysutils/iocage` 配置、运行和销毁一个简单 jail 的示例：

> **注意**：这会继承所有接口并从 jailhost 复制 resolver 配置。

```sh
root@jailhost:~ # pkg install py36-iocage
...
root@jailhost:~ # iocage activate zroot
ZFS pool 'zroot' successfully activated.
root@jailhost:~ # iocage create -r 12.1-RELEASE -n simplecage ip4=inherit
...
simplecage successfully created!
root@jailhost:~ # iocage console -f simplecage
...
root@simplecage:~ # fetch -q -o - http://canhazip.com
<your public facing IP shown here>
root@simplecage:~ # logout
root@jailhost:~ # iocage destroy -f simplecage
...
Stopping simplecage
Destroying simplecage
root@jailhost:~ #
```

使用 `sysutils/pot` 做同样示例，它是一个替代的 jail 管理器，旨在提供类容器功能和 `sysutils/nomad` 集成：

> **注意**：`sysutils/pot` 是一个演进中的项目，所以你可能想通过修改 **/etc/pkg/FreeBSD.conf** 从 FreeBSD 的 latest 分支获取版本，或从 Ports 安装（**/usr/ports/sysutils/pot**）。如果网络接口名不是 `em0`，请确保在 **/usr/local/etc/pot/pot.cfg** 中设置 `POT_EXTIF`。

```sh
root@jailhost:~ # pkg install pot
...
root@jailhost:~ # pot init
...
root@jailhost:~ # pot create-base -r 12.1
...
root@jailhost:~ # pot create -p simplepot -b 12.1
...
root@jailhost:~ # pot run simplepot
root@simplepot:~ # fetch -q -o - http://canhazip.com
<your public facing IP shown here>
root@simplepot:~ # exit
root@jailhost:~ # pot destroy -Fp simplepot
===>
Destroying pot simplepot
root@jailhost:~ #
```

### 使用专用 IP 地址的普通 Jail

如果想要更多隔离，可以分配专用静态 IP 地址。这能防止 jail 监听 jailhost 使用的端口，例如允许 `sshd(8)` 直接监听标准端口（22）连接到 jail。

下例中，jailhost 使用 **192.168.0.2** 作为主 IP 地址，**192.168.0.1** 作为默认网关，**192.168.0.3** 是新增的、由 jail 使用的附加 IP 地址。目标是在局域网内创建一个使用专用静态 IP 地址的 jail，并在其内运行 `sshd(8)`。

这假设 jailhost 的安全 shell 守护进程已配置为只监听相关 IP 地址，方法是在 **/etc/ssh/sshd_config** 中设置 `ListenAddress` 为 **192.168.0.2**，并通过 `service sshd reload` 重新加载。

使用 `sysutils/pot` 的静态局域网 IP jail 示例：

```sh
root@jailhost:~ # pot create -p aliaspot -b 12.1 -N alias -i 192.168.0.3
===>
Creating a new pot
===>
pot name : aliaspot
===>
type : multi
===>
base : 12.1
===>
pot_base :
===>
level : 1
===>
network-type: alias
===>
ip : 192.168.0.3
===>
dns : inherit
root@jailhost:~ # pot run aliaspot
```

你可以在供应过程的不同阶段运行 `ifconfig(8)`，了解 IP 别名（**192.168.0.33/32**）何时被加入/移除到接口。根据用例，将别名永久添加到 jailhost 的 **/etc/rc.conf** 也可能有意义。

> **注意**：在这种静态单 IP 配置下，jail 的唯一 IP 地址（神奇地）也用作 localhost，这有时会令人困惑，也意味着通常监听 localhost 因此外部无法访问的服务（如上例中的 `sendmail_submit`）突然暴露在外。所以在这种配置下，对 jailhost 上的服务做正确的防火墙（遵循默认阻止所有流量的最佳实践）很重要。可以为 jail 添加独立的回环地址（如 **127.0.0.2/8**），但考虑到额外引入的复杂性，这种做法只在少数用例中值得。

```sh
...
root@aliaspot:~ # service sshd enable
sshd enabled in /etc/rc.conf
root@aliaspot:~ # service sshd start
...
root@aliaspot:~ # exit
root@jailhost:~ # sockstat -4lj aliaspot
LOCAL ADDRESS
FOREIGN ADDRESS
root
sshd
7324
tcp4
192.168.0.3:22
*:*
root
sendmail
7183
tcp4
192.168.0.3:25
*:*
root
syslogd
7102
udp4
192.168.0.3:514
*:*
root@jailhost:~ # pot destroy -Fp aliaspot
===>
Destroying pot aliaspot
root@jailhost:~ #
```

### 使用 VLAN IP 地址的普通 Jail

前一配置的变体是配置 jail 监听专用（私有）网络上的 IP 地址，方法是使用专用接口，或在现有物理接口上添加 VLAN 接口。这提供更好的网络分段，并允许部署中央过滤、出站网络地址转换（NAT）和入站重定向（DNAT）。

下例中，在 jailhost 上（接口 `em0`）配置了一个 VLAN 接口（VLAN 标签 101，**10.1.1.1/24**），并使用 `sysutils/iocage` 在配置好的 jail 中使用：

```sh
root@jailhost:~ # sysrc ifconfig_em0_101="10.1.1.1/24"
ifconfig_em0_101:
-> 10.1.1.1/24
root@jailhost:~ # ifconfig em0.101 create
root@jailhost:~ # iocage create \
-r 12.1-RELEASE -n vlancage ip4_addr="em0.101|10.1.1.2"
...
vlancage successfully created!
root@jailhost:~ # iocage console -f vlancage
...
root@vlancage:~ # ifconfig -g vlan
em0.101
root@vlancage:~ # ifconfig em0.101
em0.101: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 \
mtu 1500
options=3<RXCSUM,TXCSUM>
ether f4:4d:30:aa:bb:cc
inet 10.1.1.2 netmask 0xffffffff broadcast 10.1.1.2
groups: vlan
vlan: 101 vlanpcp: 0 parent interface: em0
media: Ethernet autoselect (100baseTX <full-duplex>)
status: active
root@vlancage:~ # logout
root@jailhost:~ # iocage destroy -f vlancage
...
Stopping vlancage
Destroying vlancage
root@jailhost:~ #
```

这假设防火墙和 NAT 在网络中的另一台主机/设备上进行。由于宿主机已配置 IP 地址（**10.1.1.1/24**），将 jail 的 IP 地址作为别名宿主地址（掩码 **/32**）添加正是所需。这也有助于配置，允许在 jail 启动前配置静态路由，并基于接口的网络配置设置防火墙规则。

如果 jail 应使用该接口的主 IP 地址，必须准确按照 VLAN 接口上的配置设置 IP 地址和掩码。上例中即设置 `ip4_addr="em0.101|10.1.1.1/24"`。`iocage(8)` 足够聪明，如果 jail 的 IP 地址在启动 jail 前已配置，则不会从接口移除。

> **提示**：像大多数网络接口一样，可以语义化命名 VLAN 接口；详见 `rc.conf(5)`（`man rc.conf`）。

### 使用回环 IP 地址的普通 Jail

在某些情况下，比如可分配给局域网接口的 IP 地址数量有限，或你只维护单一 jailhost，让 jail 监听本地回环接口可能合理。这种情况通常使用本地防火墙做出站 NAT 和入站流量重定向（如有必要）。

下面 `sysutils/iocage` 示例中，创建了名为 `lo1`、持有 IP 地址 **172.31.255.17/32** 的专用本地接口来服务 jail。本例中 IP 地址由 jailhost 在引导时分配，而不是由 `sysutils/iocage` 启动 jail 时分配：

```sh
root@jailhost:~ # sysrc cloned_interfaces+="lo1"
cloned_interfaces:
-> lo1
root@jailhost:~ # sysrc ifconfig_lo1="172.31.255.17/32"
ifconfig_lo1:
-> 172.31.255.17/32
root@jailhost:~ # ifconfig lo1 create
root@jailhost:~ # ifconfig lo1
lo1: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384
options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
inet 172.31.255.17 netmask 0xffffffff
groups: lo
nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
root@jailhost:~ # iocage create \
-r 12.1-RELEASE -n locage ip4_addr="lo1|172.31.255.17/32"
...
locage successfully created!
root@jailhost:~ # iocage console -f locage
...
root@locage:~ # host freebsd.org
;; connection timed out; no servers could be reached
root@locage:~ # logout
root@jailhost:~ #
```

### 为公共流量添加出站 NAT

此时 jail 存在但无法与外部世界通信。可以通过向防火墙添加出站 NAT 规则解决。

> **警告**：启用/配置防火墙是把自己锁在机器外的好办法。确保你已有方案，万一意外发生能重新获得访问权限。

本例使用 `pf(4)`（packet filter）防火墙。假设尚未配置防火墙。

创建以下 **/etc/pf.conf**：

```ini
ext_if="em0"
www_jail="172.31.255.17"
# 在 jailhost 主 IP 上为 www jail 做出站 NAT
nat on $ext_if from $www_jail -> $ext_if:0
# 将 www 流量重定向到 jail
rdr on $ext_if proto tcp to $ext_if:0 port www -> $www_jail
# 不干预 jailhost 的本地回环
set skip on lo0
# 防止欺骗
antispoof for lo0
antispoof for lo1
antispoof for $ext_if
# 默认阻止全部（最佳实践）
block
# 允许 jail 中不流向 jailhost 的所有流量
# （包括 jail 内部到自身的流量）
pass from $www_jail to !$ext_if:0
# 允许访问 web 服务器
pass proto tcp to $www_jail port www
# 允许管理主机
pass in on $ext_if proto tcp to $ext_if:0 port ssh
# 允许出站流量
pass out on $ext_if
```

接下来启用 `pf(4)` 并验证出站连接按预期工作：

```sh
root@jailhost:~ # service pf enable
pf enabled in /etc/rc.conf
root@jailhost:~ # service pf start
... (会话断开) ...
root@jailhost:~ # iocage console -f locage
root@locage:~ # fetch -q -o - http://canhazip.com
<your public facing IP shown here>
root@locage:~ # logout
root@jailhost:~ #
```

### 运行服务并将流量重定向到它

现在 jail 内有出站连接，让我们安装 `www/nginx` 提供静态内容。所需防火墙规则已就位（检查之前配置的 `rdr` 和 `pass` 规则）：

```sh
root@jailhost:~ # grep -B1 "port www" /etc/pf.conf
# 将 www 流量重定向到 jail
rdr on $ext_if proto tcp to $ext_if:0 port www -> $www_jail
--
# 允许访问 web 服务器
pass proto tcp to $www_jail port www
root@jailhost:~ # iocage console -f locage
root@locage:~ # pkg install nginx
...
root@locage:~ # rm /usr/local/www/nginx
root@locage:~ # mkdir /usr/local/www/nginx
root@locage:~ # echo "Hello Jail" >/usr/local/www/nginx/index.html
root@locage:~ # service nginx enable
nginx enabled in /etc/rc.conf
root@locage:~ # service nginx start
Performing sanity check on nginx configuration:
nginx: the configuration file /usr/local/etc/nginx/nginx.conf syntax is ok
nginx: configuration file /usr/local/etc/nginx/nginx.conf test is\
successful
Starting nginx.
root@locage:~ # logout
root@jailhost:~ # fetch -q -o - http://172.31.255.17
Hello Jail
root@jailhost:~ #
```

此时从外部访问 jail 内托管的 web 服务器应该可用，可通过浏览器指向服务器的主 IP 地址验证。

> **警告**：虽然可以用这种方式创建干净易懂的配置，但这种设置扩展性不佳，维护起来麻烦。而且虽然能在各 jail 之间做一定程度的防火墙，但不太实用，因此 VNET jail 更受推荐。

## VNET Jail 和 bhyve VM

VNET(9)——网络子系统虚拟化基础设施——是一种虚拟化网络栈的技术。

虽然 VNET 最早出现在 FreeBSD 8.0 中，但直到最近都被视为实验特性。随着 FreeBSD 12 的到来，终于人人可用，无需构建自定义内核。

应用到 jail 时，VNET 让每个 jail 拥有自己的网络栈。这解决了之前看到的一些不足，带来了更好的隔离和分割、“正常”的本地回环接口，以及在 jail 内运行独立防火墙的能力。

### 使用 sysutils/pot 的 VNET Jail

本节演示如何使用 `sysutils/pot` 创建多个 VNET jail。在演示之前，先撤销前面 `pf(4)` 的配置（如果你没运行这些示例且 `pf(4)` 未运行，请将 `service pf reload` 替换为 `service pf start`）：

```sh
root@jailhost:~ # rm -f /etc/pf.conf
root@jailhost:~ # pot init
...
Please, check that your PF configuration file
is still valid!
root@jailhost:~ # cat /etc/pf.conf
nat-anchor pot-nat
rdr-anchor "pot-rdr/*"
root@jailhost:~ # service pf enable
root@jailhost:~ # service pf reload
```

当创建网络类型为 `public-bridge` 的 jail 时，pot 自动使用 VNET。基于 **/usr/local/etc/pot/pot.cfg** 中的”内部虚拟网络配置”，它创建桥接接口并分配一个 IP 地址作为 jail 的默认网关。

创建 jail 时，pot 从配置的范围自动分配静态 IP 地址。jail 启动时，会创建一个 `epair(4)` 接口，并将其”a 端”加入桥接：

```sh
root@jailhost:~ # pot vnet-start
pfctl: pf already enabled
root@jailhost:~ # ifconfig bridge0
bridge0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0\
mtu 1500
ether 02:0e:35:b7:6d:00
inet 10.192.0.1 netmask 0xffc00000 broadcast 10.255.255.255
id 00:00:00:00:00:00 priority 32768 hellotime 2 fwddelay 15
maxage 20 holdcnt 6 proto rstp maxaddr 2000 timeout 1200
root id 00:00:00:00:00:00 priority 32768 ifcost 0 port 0
member: epair0a flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
ifmaxaddr 0 port 6 priority 128 path cost 2000
groups: bridge
nd6 options=1<PERFORMNUD>
root@jailhost:~ # pot create -b 12.1 -N public-bridge -p vnetpot1
...
root@jailhost:~ # pot create -b 12.1 -N public-bridge -p vnetpot2
...
root@jailhost:~ # pot start vnetpot1
...
root@jailhost:~ # pot start vnetpot2
...
root@jailhost:~ # pot list
pot name : base-12_1
network : inherit
active : false
pot name : vnetpot1
network : public-bridge
ip : 10.192.0.3
active : true
pot name : vnetpot2
network : public-bridge
ip : 10.192.0.4
active : true
root@jailhost:~ # ifconfig bridge0
bridge0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0\
mtu 1500
ether 02:0e:35:b7:6d:00
inet 10.192.0.1 netmask 0xffc00000 broadcast 10.255.255.255
id 00:00:00:00:00:00 priority 32768 hellotime 2 fwddelay 15
maxage 20 holdcnt 6 proto rstp maxaddr 2000 timeout 1200
root id 00:00:00:00:00:00 priority 32768 ifcost 0 port 0
member: epair1a flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
ifmaxaddr 0 port 7 priority 128 path cost 2000
member: epair0a flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
ifmaxaddr 0 port 6 priority 128 path cost 2000
groups: bridge
nd6 options=1<PERFORMNUD>
root@jailhost:~ # ifconfig -g epair
epair0a
epair1a
root@jailhost:~ # ifconfig epair0a
epair0a: flags=8943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST>\
metric 0 mtu 1500
options=8<VLAN_MTU>
ether 02:f5:dd:bc:cd:0a
groups: epair
media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
status: active
nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
root@jailhost:~ # ifconfig epair1a
epair1a: flags=8943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST>\
metric 0 mtu 1500
options=8<VLAN_MTU>
ether 02:d8:52:a6:86:0a
groups: epair
media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
status: active
nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
```

`epair(4)` 接口的”b 端”由 jail 使用，jail 按其 **/etc/rc.conf** 中的配置设置静态 IP 地址：

```sh
root@jailhost:~ # jexec vnetpot1 grep ifconfig /etc/rc.conf
ifconfig_epair0b="inet 10.192.0.3 netmask 255.192.0.0
root@jailhost:~ # jexec vnetpot1 ifconfig epair0b
epair0b: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0\
mtu 1500
options=8<VLAN_MTU>
ether 02:f5:dd:bc:cd:0b
inet 10.192.0.3 netmask 0xffc00000 broadcast 10.255.255.255
groups: epair
media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
status: active
nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
root@jailhost:~ # jexec vnetpot2 grep ifconfig /etc/rc.conf
ifconfig_epair1b="inet 10.192.0.4 netmask 255.192.0.0"
root@jailhost:~ # jexec vnetpot2 ifconfig epair1b
epair1b: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0\
mtu 1500
options=8<VLAN_MTU>
ether 02:d8:52:a6:86:0b
inet 10.192.0.4 netmask 0xffc00000 broadcast 10.255.255.255
groups: epair
media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
status: active
nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
```

使用 **/etc/pf.conf** 中配置的 anchor，pot 向 `pf(4)` 添加 NAT 规则，可通过 `pfctl(8)` 查看：

```sh
root@jailhost:~ # pfctl -s nat -a pot-nat
nat on em0 inet from 10.192.0.0/10 to any -> (em0) round-robin
```

像前面普通 jail 重定向示例那样，我们希望其中一个 VNET jail 运行 web 服务器，并通过入站重定向（DNAT）对外提供。在第一个 jail 内安装 `www/nginx`，并使用 pot 的”export-port”功能创建入站重定向：

```sh
root@jailhost:~ # pot run vnetpot1
root@vnetpot1:~ # pkg install nginx
...
root@vnetpot1:~ # rm /usr/local/www/nginx
root@vnetpot1:~ # mkdir /usr/local/www/nginx
root@vnetpot1:~ # echo "Hello Jail" >/usr/local/www/nginx/index.html
root@vnetpot1:~ # service nginx enable
nginx enabled in /etc/rc.conf
root@vnetpot1:~ # service nginx start
...
root@vnetpot1:~ # exit
root@jailhost:~ # fetch -q -o - http://10.192.0.3
Hello Jail
root@jailhost:~ # pot export-ports -p vnetpot1 -e 80:80
root@jailhost:~ # pot stop vnetpot1
root@jailhost:~ # pot start vnetpot1
...
root@jailhost:~ # pot show
pot vnetpot1
disk usage
: 55.2M
===>
runtime memory usage require rctl enabled
Network port redirection
192.168.0.2 port 80 -> 10.192.0.3 port 80
pot vnetpot2
disk usage
: 308K
===>
runtime memory usage require rctl enabled
root@jailhost:~ #
```

pot 在 `pf(4)` 中创建包含导出端口重定向 pass 规则的 anchor，同样可通过 `pfctl(8)` 查看：

```sh
root@jailhost:~ # pfctl -a pot-rdr -s Anchors
pot-rdr/vnetpot1
root@jailhost:~ # pfctl -a pot-rdr/vnetpot1 -s nat
rdr pass on em0 inet proto tcp \
from any to 192.168.0.2 port = http -> 10.192.0.3 port 80
```

> **注意**：此时 `pf(4)` 仅用于 NAT 和重定向流量，不阻止任何流量。

### 使用 sysutils/iocage 的 VNET Jail

相比 `sysutils/pot`，`sysutils/iocage` 在创建自定义配置时提供更大灵活性和细粒度控制。

#### 管理桥接

`iocage(8)` 期望用户自行管理 VNET jail 连接的桥接，所以通常通过 `ifconfig(8)` 和修改 **/etc/rc.conf** 手动完成。

另一种管理桥接的方式是使用 `sysutils/vm-bhyve`，它将桥接抽象为”虚拟交换机”。由于 `vm-bhyve(8)` 是管理 bhyve VM 的出色工具，我们利用它管理 `iocage(8)` jail 的桥接，之后将 bhyve VM 连接到它。

> **注意**：`vm-bhyve(8)` 支持不同交换机类型。我们暂时限于默认的”standard”。

运行 `rcorder /usr/local/etc/rc.d/*` 可看到 `vm` 在系统启动时先于 `iocage` 执行。这意味着 jail 连接时桥接已可用。

理清这点后，我们创建第一个交换机：

```sh
root@jailhost:~ # pkg install vm-bhyve
vm enabled in /etc/rc.conf
root@jailhost:~ # service vm enable
vm enabled in /etc/rc.conf
root@jailhost:~ # sysrc vm_dir=zfs:zroot/vms
vm_dir:
-> zfs:zroot/vms
root@jailhost:~ # zfs create zroot/vms
root@jailhost:~ # vm init
root@jailhost:~ # vm help | grep switch
...
root@jailhost:~ # vm switch create -a 10.1.1.1/24 services
root@jailhost:~ # vm switch list
services
standard
vm-services
10.1.1.1/24
no
-
-
-
root@jailhost:~ # ifconfig vm-services
vm-services: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0\
mtu 1500
ether f6:b5:a8:80:78:5e
inet 10.1.1.1 netmask 0xffffff00 broadcast 10.1.1.255
id 00:00:00:00:00:00 priority 32768 hellotime 2 fwddelay 15
maxage 20 holdcnt 6 proto rstp maxaddr 2000 timeout 1200
root id 00:00:00:00:00:00 priority 32768 ifcost 0 port 0
groups: bridge vm-switch viid-10cd3@
nd6 options=1<PERFORMNUD>
```

并将新的 `iocage(8)` jail 连接到它：

```sh
root@jailhost:~ # iocage create -r 12.1-RELEASE -n vnetcage \
interfaces="vnet0:vm-services" ip4_addr="vnet0|10.1.1.2/24" \
defaultrouter="10.1.1.1" vnet_default_interface="vm-services" \
vnet=on
vnetcage successfully created!
root@jailhost:~ # iocage start vnetcage
No default gateway found for ipv6.
* Starting vnetcage
+ Started OK
+ Using devfs_ruleset: 5
+ Configuring VNET OK
+ Using IP options: vnet
+ Starting services OK
+ Executing poststart OK
root@jailhost:~ # ifconfig -g epair
vnet0.29
root@jailhost:~ # ifconfig vnet0.29
vnet0.29: flags=8943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST>\
metric 0 mtu 1500
description: associated with jail: vnetcage as nic: epair0b
options=8<VLAN_MTU>
ether f4:4d:30:13:1e:5c
hwaddr 02:82:25:a0:cf:0a
inet6 fe80::f64d:30ff:fe13:1e5c%vnet0.29 prefixlen 64 scopeid 0x4
groups: epair
media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
status: active
nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
root@jailhost:~ # ifconfig vm-services
vm-services: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0\
mtu 1500
ether 0a:3b:ea:3e:5c:eb
inet 10.1.1.1 netmask 0xffffff00 broadcast 10.1.1.255
id 00:00:00:00:00:00 priority 32768 hellotime 2 fwddelay 15
maxage 20 holdcnt 6 proto rstp maxaddr 2000 timeout 1200
root id 00:00:00:00:00:00 priority 32768 ifcost 0 port 0
member: vnet0.29 flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
ifmaxaddr 0 port 4 priority 128 path cost 2000
groups: bridge vm-switch viid-10cd3@
nd6 options=1<PERFORMNUD>
```

为让 jail 能与外界通信，我们添加最简配置为其做出站 NAT（假设 `pf(4)` 尚未运行）：

```sh
root@jailhost:~ # echo "set skip on lo0" >/etc/pf.conf
root@jailhost:~ # echo "nat on em0 from 10.1.1/24 -> em0:0" >>/etc/pf.conf
root@jailhost:~ # service pf enable
pf enabled in /etc/rc.conf
root@jailhost:~ # service pf start
Enabling pf.
root@jailhost:~ # iocage console vnetcage
root@vnetcage:~ # fetch -q -o - http://canhazip.com
<your public facing IP shown here>
root@vnetcage:~ # logout
root@jailhost:~ #
```

## 加入 bhyve VM 和 DHCP

上例创建了名为”services”的虚拟交换机，由桥接接口”vm-services”支撑，连接了一个名为”vnetcage”的 VNET jail，它使用 NAT 与外界通信。

下一步将一台运行 FreeBSD 的 bhyve VM 加入同一虚拟交换机，在网络 **10.1.1.0/24** 上使用不同 IP 地址。

网络图：

```sh
.-,(
),-.
.-(
)-.
(
internet
)
'-(
).-'
.------).-'
/
/
.--------/-----------------------------.
|
.--'--.
jailhost
|
|
|
| em0 |
|
|
'---.-'
|
|
\
|
|
.-'-----------------.
|
|
| services (switch) |
|
|
'---.------------.--'
|
|
/
\
|
|
/
\
|
| .-------'-------.
.-------'-------. |
| |
vnetcage
|
|
guest
| |
| | (iocage jail) |
|
(bhyve vm)
| |
| '---------------'
'---------------' |
|
|
'--------------------------------------'
```

为简化供应过程，我们让 bhyve VM 通过 DHCP 获取 IP 地址。使用 `dns/dnsmasq` 实现这一目的：

```sh
root@jailhost:~ # pkg install dnsmasq
...
cat >/usr/local/etc/dnsmasq.conf <<EOF
domain-needed
listen-address=10.1.1.1
interface=vm-services
bind-interfaces
local-service
dhcp-authoritative
dhcp-range=10.1.1.10,10.1.1.20
root@jailhost:~ # service dnsmasq enable
dnsmasq enabled in /etc/rc.conf
root@jailhost:~ # service dnsmasq start
```

我们前面已安装 `sysutils/vm-bhyve`；剩下的工作是下载安装 ISO、创建 VM、连接到交换机并运行安装程序：

```sh
root@jailhost:~ # vm iso https://download.freebsd.org/ftp/releases/\
ISO-IMAGES/12.1/FreeBSD-12.1-RELEASE-amd64-bootonly.iso
...
root@jailhost:~ # vm create guest
root@jailhost:~ # vm add -d network -s services guest
root@jailhost:~ # vm install guest \
FreeBSD-12.1-RELEASE-amd64-bootonly.iso
root@jailhost:~ # vm console guest
...
root@jailhost:~ # vm stop guest
root@jailhost:~ # vm start guest
```

这使用 bootonly ISO（基于网络的安装），应该可行，因为 IP 地址由 `dnsmasq(8)`（同时提供 DNS 服务）分配给客户机，且 NAT 已在宿主防火墙上配置。

> **注意**：这显示两个网络接口。`vtnet1` 是我们设置中要使用的接口。可通过修改 VM 配置文件 **/zroot/vms/guest/guest.conf** 手动移除 `vtnet0`。

如预期，`dnsmasq(8)` 从 DHCP 池给新 VM 分配了一个 IP 地址（本例中为 **10.1.1.11/24**）。在需要稳定 IP 的环境中，基于虚拟网络接口的 MAC 地址在 **/usr/local/etc/dnsmasq.conf** 中分配固定 IP（不属动态池）更有利，例如：

```ini
dhcp-host=11:22:33:44:55:66,192.168.0.60
```

在新 VM 中添加非特权用户并启用 `sshd(8)` 后，我们就能 `ssh(1)` 进去：

```sh
root@jailhost:~ # ssh user@10.1.1.11
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.1.1.11' (ECDSA) to the list of known hosts.
Password for user@guest:
$ fetch -q -o - http://canhazip.com
<your public facing IP shown here>
$ ^DConnection to 10.1.1.11 closed.
root@jailhost:~ #
```

> **注意**：通过将 `/dev/bpf` 暴露给 VNET jail，可以像此处 VM 那样用 DHCP 配置 jail 的 IP 地址。在 `iocage(8)` 中启用 `dhcp` 属性即可轻松实现。

### 阻止 VNET Jail/VM 之间的流量

阻止 jail 和 VM 之间流量的简单方法是在桥接成员上设置 private 标志（`ifconfig <bridgename> private <interfacename>`）。

任何标记为 private 的接口都无法与同样标记为 private 的其他接口通信。

`sysutils/vm-bhyve` 在 VM 启动时，如果交换机已通过运行 `vm switch private <switchname> on` 配置为 private，会自动设置 private 标志。

遗憾的是，这只对 VM 有效，所以如果将 `iocage` jail 连接到交换机，必须在每次 jail 启动时手动设置 private 标志。

当前配置中，“vnetcage”可以 ssh 到”guest”：

```sh
root@jailhost:~ # vm Stop guest
Sending ACPI shutdown to guest
root@jailhost:~ # vm switch private services on
root@jailhost:~ # vm switch list
services
standard
vm-services
10.1.1.1/24
yes
-
-
-
root@jailhost:~ # vm start guest
Starting guest
* found guest in /zroot/vms/guest
* booting...
root@jailhost:~ # ifconfig vm-services
vm-services: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0\
mtu 1500
ether 0a:3b:ea:3e:5c:eb
inet 10.1.1.1 netmask 0xffffff00 broadcast 10.1.1.255
id 00:00:00:00:00:00 priority 32768 hellotime 2 fwddelay 15
maxage 20 holdcnt 6 proto rstp maxaddr 2000 timeout 1200
root id 00:00:00:00:00:00 priority 32768 ifcost 0 port 0
member: tap1 flags=943<LEARNING,DISCOVER,PRIVATE,AUTOEDGE,AUTOPTP>
ifmaxaddr 0 port 6 priority 128 path cost 2000000
member: vnet0.31 flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
ifmaxaddr 0 port 7 priority 128 path cost 2000
groups: bridge vm-switch viid-10cd3@
nd6 options=1<PERFORMNUD>
root@jailhost:~ # iocage console -f vnetcage
root@vnetcage:~ # nc 10.1.1.11 22
SSH-2.0-OpenSSH_7.8 FreeBSD-20180909
^C
root@vnetcage:~ # logout
root@jailhost:~ #
```

将交换机切换为 private 模式后，VM 连接到桥接的 tap 接口被标记为 PRIVATE。由于 jail 的 epair 接口未标记 PRIVATE，流量仍可流通，ssh 仍可用：

让我们手动将 jail 的连接改为”private”，可以看到连接尝试 jail 时超时，而到外界的 NAT 仍然完好，jailhost 仍能通过 `ssh(1)` 连接到 VM：

```sh
root@jailhost:~ # ifconfig vm-services private vnet0.31
root@jailhost:~ # iocage console -f vnetcage
root@vnetcage:~ # nc -vw 10 10.1.1.11 22
nc: connect to 10.1.1.11 port 22 (tcp) failed: Operation timed out
root@vnetcage:~ # fetch -q -o - http://canhazip.com
<your public facing IP shown here>
root@vnetcage:~ # logout
root@jailhost:~ # nc 10.1.1.11 22
SSH-2.0-OpenSSH_7.8 FreeBSD-20180909
^C
root@jailhost:~ #
```

理想情况下，`iocage(8)` 应支持一个配置选项，允许在桥接成员上自动设置 private 标志。在该功能出现之前，下面的脚本可配置为通过 jail 的 `poststart` 钩子执行，达到（几乎）同样的效果。

**/usr/local/sbin/set_ioc_vnet_private.sh**：

```sh
#!/bin/sh
set -e
if [ "$#" -ne 1 -a "$#" -ne 2 ]; then
echo "Usage: $0 jailname [vnetprefix]" >&2
exit 1
fi
JAILNAME=$1
VNETPREFIX=${2:-vnet0}
JAILINFO=$(jls -j ioc-$JAILNAME)
JID=$(echo "$JAILINFO" | grep $JAILNAME | awk '{ print $1 }')
for BRIDGE in $(ifconfig -g bridge); do
CONFIG=$(ifconfig $BRIDGE)
set +e
echo "$CONFIG" | grep "member: ${VNETPREFIX}\."$JID >/dev/null
NOTFOUND=$?
set -e
if [ $NOTFOUND -eq 0 ]; then
ifconfig $BRIDGE private $VNETPREFIX.$JID
exit 0
fi
done
echo "Couldn't find interface $VNETPREFIX.$JID on any bridges" >&2
exit 1
```

脚本以 jail 名称为必需参数，可选 vnet 接口（iocage 内部编号）。后者默认为”vnet0”。

现在重启”vnetcage”jail，检查桥接成员配置，然后修改其配置以使用新脚本，再次重启并比较结果桥接配置：

```sh
root@jailhost:~ # iocage restart vnetcage
...
root@jailhost:~ # ifconfig vm-services | grep vnet
member: vnet0.15 flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
root@jailhost:~ # ifconfig vm-services | grep vnet
root@jailhost:~ # iocage set \
exec_poststart="/usr/local/sbin/set_ioc_vnet_private.sh vnetcage" \
vnetcage
exec_poststart: /usr/bin/true -> /usr/local/sbin/set_ioc_vnet_private.sh \
vnetcage
root@jailhost:~ # iocage restart vnetcage
...
root@jailhost:~ # ifconfig vm-services | grep vnet
member: vnet0.16 flags=943<LEARNING,DISCOVER,PRIVATE,AUTOEDGE,\
AUTOPTP>
root@jailhost:~ #
```

如你所见，桥接成员现在在 jail 启动时正确配置为 private。

> **注意**：这个方案不完美，因为在 jail 启动到运行 `exec_poststart` 命令之间的短时间内流量是可能的。

### VNET Jail/VM 内的防火墙

bhyve VM 内的防火墙很直接——只需运行 VM 内操作系统自带的宿主防火墙。

jail 内运行防火墙稍复杂。`ipfw(8)` 是推荐选项，但 `pf(4)` 此时也应可用。为不在本文引入更多语法，我们这里介绍后者，尽管 `iocage` 文档另有推荐。

要在 VNET jail 内运行 `pf(4)`，必须加载 `pf(4)` 内核模块，并暴露多个设备。方法是在 **/etc/devfs.rules** 中添加新规则集：

```ini
[vnet_jail_pf=501]
add include $devfsrules_hide_all
add include $devfsrules_unhide_basic
add include $devfsrules_unhide_login
add include $devfsrules_jail
add path pf unhide
add path pflog unhide
```

并应用到 jail：

```sh
root@jailhost:~ # service devfs restart
root@jailhost:~ # iocage stop vnetcage
...
root@jailhost:~ # iocage set devfs_ruleset=501 vnetcage
devfs_ruleset: 4 -> 501
root@jailhost:~ # iocage console -f vnetcage
...
root@vnetcage:~ # ls /dev/pf
/dev/pf
root@vnetcage:~ # logout
root@jailhost:~ #
```

> **注意**：由于 `iocage(8)` 的一个 bug，每次 jail 停止时配置的 `devfs(8)` 规则会从 `devfs(8)` 中移除。未来版本可能会修复；目前有一个补丁可用（`https://github.com/iocage/iocage/pull/1106`）可解决该问题。

使用最简防火墙配置测试——检查阻止特定 IP 地址是否有效，以及状态是否如预期创建：

```sh
root@jailhost:~ # iocage console -f vnetcage
root@vnetcage:~ # echo "set skip on lo0" >/etc/pf.conf
root@vnetcage:~ # echo "block quick to 1.1.1.1" >>/etc/pf.conf
root@vnetcage:~ # echo "pass" >>/etc/pf.conf
root@vnetcage:~ # service pf enable
pf enabled in /etc/rc.conf
root@vnetcage:~ # service pf start
Enabling pf.
root@vnetcage:~ # service pf start
root@vnetcage:~ # ping -c1 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=56 time=16.475 ms
--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 16.475/16.475/16.475/0.000 ms
root@vnetcage:~ # pfctl -s state
all icmp 10.1.1.2:39231 -> 8.8.8.8:39231
0:0
root@vnetcage:~ # ping -c1 1.1.1.1
PING 1.1.1.1 (1.1.1.1): 56 data bytes
ping: sendto: Permission denied
--- 1.1.1.1 ping statistics ---
1 packets transmitted, 0 packets received, 100.0% packet loss
root@vnetcage:~ # fetch -q -o - http://canhazip.com
93.104.68.83
root@vnetcage:~ # pfctl -s state
all icmp 10.1.1.2:39231 -> 8.8.8.8:39231
0:0
all udp 10.1.1.2:32682 -> 8.8.8.8:53
MULTIPLE:SINGLE
all tcp 10.1.1.2:58192 -> 104.16.223.38:80
FIN_WAIT_2:FIN_WAIT_2
root@vnetcage:~ # logout
root@jailhost:~ #
```

> **注意**：同一桥接上的 jail/VM 可设置/盗用 IP、更改 MAC 地址，所以这层保护只足以防止不必要/意外的流量发生。如需更严格的安全，可在 jailhost 上启用 `ipfw(8)` 的二层过滤，并在宿主桥接和 jail 的 epair 接口之间插入额外的桥接。这种配置的细节超出本文范围。

Jail 和 VM 是虚拟化技术，为资源供应增加了大量灵活性和敏捷性。

## VXLAN

VXLAN（虚拟可扩展局域网接口）是一种隧道协议，旨在将虚拟网络与底层物理网络解耦，从而简化自动化和编排。它通过将二层以太网帧封装到三层 IP/UDP 包中实现。可以把 VXLAN 看作多租户数据中心版的 VLAN。就像 VLAN 使用唯一 VLAN 标签，VXLAN 使用 VXLAN 网络标识符（VNI）——VXLAN 头部中的 24 位值——区分网段。

### VXLAN 示例概览

示例设置包含同一局域网上的三台主机：一台网关主机（局域网 IP **192.168.0.1**），以及两台 jailhost（“jailhost-a”和”jailhost-b”），局域网 IP 地址分别为 **192.168.0.10** 和 **192.168.0.20**。jailhost 托管 VNET jail 和 VM；“jailhost-b”还托管一个普通 jail。

三台主机通过两个 VXLAN（VXLAN id 111 和 222）连接；网关主机通过专用上行链路使用 NAT 提供互联网访问。

Jail 和 VM 在两个 VXLAN 上获取 IP 地址（网络 **10.0.111.0/24** 和 **10.0.222.0/24**）。

本例中，许多配置通过重启相关机器来应用。虽然这主要是为简洁起见，但作为副作用也重启测试了设置——本就需要做这件事。所有描述的操作也可不重启完成。

网络图：

```sh
.-,(
),-.
.-(
)-.
(
internet
)
'-(
).-'
'-.( ).-'
.---
/
.--------/-------------------.
|
.---'--.
Gateway
|
|
| em0
|
|
|
'------'
|
|
|
|
.---------.----------.
|
.............| vxlan111| vxlan222 |..............
.
|
'---------'----------'
|
.
.
|
|
em1
|
|
.
.
|
'----------.---------'
|
.
.
'--------------|-------------'
.
.
|
.
'................>.-----'-----.<................'
............................>| IP Switch |<............................
.
.----^-^---.'
.
.
/
. .
\
.
.-.--------------------------/----. . ..----\---------------------------.-.
| .
jailhost-a
/
| . .|
\
jailhost-b
. |
| .
/
| . .|
\
. |
| .
.-------------------'.
| . .|
.'-------------------.
. |
| .
|
igb0
|
| . .|
|
em0
|
. |
| .
.---------.----------.
| . .|
.---------.----------.
. |
| '...| vxlan111| vxlan222 |........' '.......| vxlan111| vxlan222 |....' |
|
'----.----'-----.----'
|
|
'-----.---'------.---'
|
|
/
\
|
|
/
\
|
|
/
\
|
|
'
\
|
| .-----'-------. .------'------. |
|
.------'------. .------'------.|
| |
switch222
| |
switch111
| |
|
|
switch111
| |
switch222
||
| '------.------' '------.------' |
|
'-------.-----' '------.------'|
|
|
|
|
|
||
|
|
|
|
|
|
|
.-------'|
|
|
| .------'-----.
.------'-----.
|
|
| .------'-----. .------'-----. |
| | vnetjail-a |
guest-a
|
|
|
| | vnetjail-b | |
guest-b
| |
| '------------'
'------------'
|
|
| '------------' '------------' |
|
|
|
| .-------------.
|
'---------------------------------'
|
'>| plainjail-b |
|
|
'-------------'
|
|
|
'----------------------------------'
```

本例使用多播模式。VXLAN 也可配置为单播模式（详见 `vxlan(4)`）。

### 网关配置

网关主机使用简单的 `ipfw(8)`/`natd(8)` 配置和两个 `vxlan(4)` 接口。假设上行链路已配置好。

防火墙、NAT 和 IP 转发：

```sh
root@gateway:~ # service ipfw enable
ipfw enabled in /etc/rc.conf
root@gateway:~ # sysrc firewall_type=open
firewall_type: UNKNOWN -> open
root@gateway:~ # service natd enable
natd enabled in /etc/rc.conf
root@gateway:~ # sysrc natd_interface=em0
natd_interface:
-> em0
root@gateway:~ #
sysrc gateway_enable=YES
gateway_enable: NO -> YES
```

VXLAN 接口：

```sh
root@gateway:~ # sysrc cloned_interfaces+="vxlan111 vxlan222"
cloned_interfaces:
-> vxlan111 vxlan222
root@gateway:~ # sysrc ifconfig_vxlan111="inet 10.0.111.1/24 mtu 1450"
ifconfig_vxlan111:
-> inet 10.0.111.1/24 mtu 1450
root@gateway:~ # sysrc create_args_vxlan111="vxlanid 111 vxlanlocal\
192.168.0.1 vxlandev em1 vxlangroup 239.0.0.111"
create_args_vxlan111:
-> vxlanid 111 vxlanlocal 192.168.0.1
vxlandev em1 vxlangroup 239.0.0.111
root@gateway:~ # sysrc ifconfig_vxlan222="inet 10.0.222.1/24 mtu 1450"
ifconfig_vxlan222:
-> inet 10.0.222.1/24 mtu 1450
root@gateway:~ # sysrc create_args_vxlan222="vxlanid 222 vxlanlocal\
192.168.0.1 vxlandev em1 vxlangroup 239.0.0.222"
create_args_vxlan222:
-> vxlanid 222 vxlanlocal 192.168.0.1
vxlandev em1 vxlangroup 239.0.0.222
```

到 VXLAN 多播地址的静态路由：

```sh
root@gateway:~ # sysrc static_routes+="vxlan111 vxlan222"
static_routes:
-> vxlan111 vxlan222
root@gateway:~ # sysrc route_vxlan111="239.0.0.111/32 -interface em1"
route_vxlan111:
-> 239.0.0.111/32 -interface em1
root@gateway:~ # sysrc route_vxlan222="239.0.0.222/32 -interface em1"
route_vxlan222:
-> 239.0.0.222/32 -interface em1
root@gateway:~ # reboot
```

### Jailhost-a

“jailhost-a”托管一台 VM 和一个 VNET jail，通过交换机（桥接接口）连接到 VXLAN 接口，VXLAN 接口再使用物理接口 `igb0` 在局域网传输封装流量。

#### 网络配置（jailhost-a）

创建 VXLAN 接口：

```sh
root@jailhost-a:~ # sysrc cloned_interfaces+="vxlan111 vxlan222"
cloned_interfaces:
-> vxlan111 vxlan222
root@jailhost-a:~ # sysrc ifconfig_vxlan111="inet 10.0.111.10/24 mtu 1450"
ifconfig_vxlan111:
-> inet 10.0.111.10/24 mtu 1450
root@jailhost-a:~ # sysrc create_args_vxlan111="vxlanid 111 vxlanlocal\
192.168.0.10 vxlandev igb0 vxlangroup 239.0.0.111"
create_args_vxlan111:
-> vxlanid 111 vxlanlocal 192.168.0.10
vxlandev igb0 vxlangroup 239.0.0.111
root@jailhost-a:~ # sysrc ifconfig_vxlan222="inet 10.0.222.10/24 mtu 1450"
ifconfig_vxlan222:
-> inet 10.0.222.10/24 mtu 1450
root@jailhost-a:~ # sysrc create_args_vxlan222="vxlanid 222 vxlanlocal\
192.168.0.10 vxlandev igb0 vxlangroup 239.0.0.222"
create_args_vxlan222:
-> vxlanid 222 vxlanlocal 192.168.0.10
vxlandev igb0 vxlangroup 239.0.0.222
```

为多播流量设置静态路由（如果默认路由经过同一接口，技术上不需要）：

```sh
root@jailhost-a:~ # sysrc static_routes+="vxlan111 vxlan222"
static_routes:
-> vxlan111 vxlan222
root@jailhost-a:~ # sysrc route_vxlan111="239.0.0.111/32 -interface igb0"
route_vxlan111:
-> 239.0.0.111/32 -interface igb0
root@jailhost-a:~ # sysrc route_vxlan222="239.0.0.222/32 -interface igb0"
route_vxlan222:
-> 239.0.0.222/32 -interface igb0
root@jailhost-a:~ # reboot
```

创建交换机（桥接接口）连接 jail/VM，并将相应 VXLAN 接口加入它们：

```sh
root@jailhost-a:~ # sysrc cloned_interfaces+="bridge0 bridge1"
cloned_interfaces: vxlan111 vxlan222 -> vxlan111 vxlan222 bridge0 bridge1
root@jailhost-a:~ # sysrc ifconfig_bridge0_name="switch111"
ifconfig_bridge0_name:
-> switch111
root@jailhost-a:~ # sysrc \
ifconfig_switch111="inet 10.0.111.12/32 addm vxlan111"
ifconfig_switch111:
-> inet 10.0.111.12/32 addm vxlan111
root@jailhost-a:~ # sysrc ifconfig_bridge1_name="switch222"
ifconfig_bridge1_name:
-> switch222
root@jailhost-a:~ # sysrc \
ifconfig_switch222="inet 10.0.222.12/32 addm vxlan222"
ifconfig_switch222:
-> inet 10.0.222.12/32 addm vxlan222
root@jailhost-a:~ # reboot
```

#### VM 配置（jailhost-a）

安装 `sysutils/vm-bhyve` 并创建一个”manual”类型（不由 vm-bhyve 管理）的交换机，引用前面创建的桥接接口（switch111）：

```sh
root@jailhost-a:~ # pkg install vm-bhyve
...
root@jailhost-a:~ # service vm enable
vm enabled in /etc/rc.conf
root@jailhost-a:~ # sysrc vm_dir=zfs:zroot/vms
vm_dir:
-> zfs:zroot/vms
root@jailhost-a:~ # zfs create zroot/vms
root@jailhost-a:~ # vm init
root@jailhost-a:~ # vm switch create -t manual -b switch111 switch111
root@jailhost-a:~ # vm switch list
switch111
manual
switch111
n/a
no
n/a
n/a
n/a
```

下载 FreeBSD ISO 并安装名为”guest-a”的 VM；让新 VM 在引导时启动：

```sh
root@jailhost-a:~ # vm iso https://download.freebsd.org/ftp/releases/\
ISO-IMAGES/12.1/FreeBSD-12.1-RELEASE-amd64-bootonly.iso
root@jailhost-a:~ # vm create guest-a
root@jailhost-a:~ # vm add -d network -s switch111 guest-a
root@jailhost-a:~ # vm install \
guest-a FreeBSD-12.1-RELEASE-amd64-bootonly.iso
root@jailhost-a:~ # vm console guest-a
... （将 vtnet1 的 IP 设为 10.0.111.13/24，默认网关设为 10.0.111.1）
root@jailhost-a:~ # sysrc vm_list+="guest-a"
root@jailhost-a:~ # vm Stop guest-a
```

启动 VM 并测试连接（注意公共流量如何通过 VXLAN 111 发送并在网关主机 NAT，因为网关主机配置为默认网关）：

```sh
root@jailhost-a:~ # vm start guest-a
root@jailhost-a:~ # vm console guest-a
root@guest-a:~ # ping -c 3 10.0.111.1
PING 10.0.111.1 (10.0.111.1): 56 data bytes
64 bytes from 10.0.111.1: icmp_seq=0 ttl=64 time=1.545 ms
64 bytes from 10.0.111.1: icmp_seq=1 ttl=64 time=0.793 ms
64 bytes from 10.0.111.1: icmp_seq=2 ttl=64 time=0.790 ms
--- 10.0.111.1 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.790/1.043/1.545/0.355 ms
root@guest-a:~ # traceroute www.freebsd.org
10.0.111.1 (10.0.111.1)
1.251 ms
1.421 ms
1.070 ms
...
root@guest-a:~ # logout
$ ^D
```

#### Jail 配置（jailhost-a）

安装 `sysutils/iocage`：

> **注意**：你可能想通过修改 **/etc/fstab** 永久挂载 `fdescfs(5)`。

```sh
root@jailhost-a:~ # pkg install py36-iocage
root@jailhost-a:~ # iocage activate zroot
ZFS pool 'zroot' successfully activated.
root@jailhost-a:~ # mount -t fdescfs null /dev/fd
root@jailhost-a:~ # service iocage enable
iocage enabled in /etc/rc.conf
```

在 VXLAN 222 上创建名为”vnetjail-a”的 jail（配置为引导时启动，因此创建后立即启动）：

```sh
root@jailhost-a:~ # iocage create -n vnetjail-a -r 12.1-RELEASE \
interfaces="vnet0:switch222" ip4_addr="vnet0|10.0.222.13/24" \
boot=1 vnet_default_interface="switch222" defaultrouter="10.0.222.1" \
vnet=on
...
vnetjail-a successfully created!
root@jailhost-a:~ # reboot
```

进入 jail 并测试连接：

```sh
root@jailhost-a:~ # iocage console vnetjail-a
root@vnetjail-a:~ # traceroute -n 141.1.1.1
traceroute to 141.1.1.1 (141.1.1.1), 64 hops max, 40 byte packets
10.0.111.1 (10.0.111.1)
1.353 ms
0.687 ms
1.045 ms
many more interesting hosts...
...
root@vnetjail-a:~ # fetch -q -o - http://canhazip.com
(your public IP here)
root@vnetjail-a:~ # logout
```

最后测试重启后一切是否正常启动。

### Jailhost-b

“jailhost-b”的设置与”jailhost-a”类似，但网络对调（VM 在 VXLAN 222，jail 在 VXLAN 111）。此外，它托管一个普通 jail，使用 VXLAN 接口（vxlan111）上的别名地址直接访问，但不以该网络作为默认网关。在生产环境中，这个 jail 可用于容纳底层 jailhost 提供的支持性服务。

#### 网络配置（jailhost-b）

创建 VXLAN 接口：

```sh
root@jailhost-b:~ # sysrc cloned_interfaces+="vxlan111 vxlan222"
cloned_interfaces:
-> vxlan111 vxlan222
root@jailhost-b:~ # sysrc ifconfig_vxlan111="inet 10.0.111.20/24 mtu 1450"
ifconfig_vxlan111:
-> inet 10.0.111.20/24 mtu 1450
root@jailhost-b:~ # sysrc create_args_vxlan111="vxlanid 111 vxlanlocal\
192.168.0.20 vxlandev em0 vxlangroup 239.0.0.111"
create_args_vxlan111:
-> vxlanid 111 vxlanlocal 192.168.0.20
vxlandev em0 vxlangroup 239.0.0.111
root@jailhost-b:~ # sysrc ifconfig_vxlan222="inet 10.0.222.20/24 mtu 1450"
ifconfig_vxlan222:
-> inet 10.0.222.20/24 mtu 1450
root@jailhost-b:~ # sysrc create_args_vxlan222="vxlanid 222 vxlanlocal\
192.168.0.20 vxlandev em0 vxlangroup 239.0.0.222"
create_args_vxlan222:
-> vxlanid 222 vxlanlocal 192.168.0.20
vxlandev em0 vxlangroup 239.0.0.222
```

为多播流量设置静态路由（如果默认路由经过同一接口，技术上不需要）：

```sh
root@jailhost-b:~ # sysrc static_routes+="vxlan111 vxlan222"
static_routes:
-> vxlan111 vxlan222
root@jailhost-b:~ # sysrc route_vxlan111="239.0.0.111/32 -interface em0"
route_vxlan111:
-> 239.0.0.111/32 -interface em0
root@jailhost-b:~ # sysrc route_vxlan222="239.0.0.222/32 -interface em0"
route_vxlan222:
-> 239.0.0.222/32 -interface em0
root@jailhost-b:~ # reboot
```

#### 普通 Jail 配置（jailhost-b）

安装 `sysutils/iocage`：

> **注意**：你可能想通过修改 **/etc/fstab** 永久挂载 `fdescfs(5)`。

```sh
root@jailhost-b:~ # pkg install py36-iocage
...
root@jailhost-b:~ # iocage activate zroot
ZFS pool 'zroot' successfully activated.
root@jailhost-b:~ # mount -t fdescfs null /dev/fd
root@jailhost-b:~ # service iocage enable
iocage enabled in /etc/rc.conf
```

在别名地址 **10.0.111.21** 上创建一个普通（非 VNET）jail。配置为引导时启动，因此创建后立即启动：

```sh
root@jailhost-b:~ # iocage create -n plainjail-b \
-r 12.1-RELEASE ip4_addr="vxlan111|10.0.111.21/32" \
allow_raw_sockets=1 boot=1
plainjail-b successfully created!
```

运行 jail 并测试连接：

```sh
root@jailhost-b:~ # iocage console -f plainjail-b
...
root@plainjail-b:~ # ping -c 3 10.0.111.1
PING 10.0.111.1 (10.0.111.1): 56 data bytes
64 bytes from 10.0.111.1: icmp_seq=0 ttl=64 time=0.811 ms
64 bytes from 10.0.111.1: icmp_seq=1 ttl=64 time=0.816 ms
64 bytes from 10.0.111.1: icmp_seq=2 ttl=64 time=0.810 ms
--- 10.0.111.1 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.810/0.812/0.816/0.003 ms
root@plainjail-b:~ # ping -c 3 10.0.111.10
PING 10.0.111.10 (10.0.111.10): 56 data bytes
64 bytes from 10.0.111.10: icmp_seq=0 ttl=64 time=0.716 ms
64 bytes from 10.0.111.10: icmp_seq=1 ttl=64 time=0.297 ms
64 bytes from 10.0.111.10: icmp_seq=2 ttl=64 time=0.319 ms
--- 10.0.111.10 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.297/0.444/0.716/0.193 ms
root@plainjail-b:~ # logout
root@jailhost-b:~ #
```

> **注意**：“plainjail-b”这种配置下，其公共流量不会通过 VXLAN 发送并由网关主机 NAT。此设置是有意的；否则会用 VNET jail。

#### 网络交换机设置（jailhost-b）

创建交换机（桥接接口）连接 jail/VM，并将相应 VXLAN 接口加入它们：

```sh
root@jailhost-b:~ # sysrc cloned_interfaces+="bridge0 bridge1"
cloned_interfaces: vxlan111 vxlan222 -> vxlan111 vxlan222 bridge0 bridge1
root@jailhost-b:~ # sysrc ifconfig_bridge0_name="switch111"
ifconfig_bridge0_name:
-> switch111
root@jailhost-b:~ # sysrc \
ifconfig_switch111="inet 10.0.111.22/32 addm vxlan111"
ifconfig_switch111:
-> inet 10.0.111.22/32 addm vxlan111
root@jailhost-b:~ # sysrc ifconfig_bridge1_name="switch222"
ifconfig_bridge1_name:
-> switch222
root@jailhost-b:~ # sysrc \
ifconfig_switch222="inet 10.0.222.22/32 addm vxlan222"
ifconfig_switch222:
-> inet 10.0.222.22/32 addm vxlan222
root@jailhost-b:~ # reboot
```

#### VNET Jail 配置（jailhost-b）

在 VXLAN 111 上创建名为”vnetjail-b”的 jail（配置为引导时启动，因此创建后立即启动）：

```sh
root@jailhost-b:~ # iocage create -n vnetjail-b -r 12.1-RELEASE \
interfaces="vnet0:switch111" ip4_addr="vnet0|10.0.111.23/24" \
boot=1 vnet_default_interface="switch111" defaultrouter="10.0.111.1" \
vnet=on
vnetjail-b successfully created!
```

进入 jail 并测试连接：

```sh
root@jailhost-b:~ # iocage console vnetjail-b
root@vnetjail-b:~ # traceroute -n 141.1.1.1
traceroute to 141.1.1.1 (141.1.1.1), 64 hops max, 40 byte packets
10.0.111.1 (10.0.111.1)
1.353 ms
0.687 ms
1.045 ms
many more interesting hosts...
...
root@vnetjail-b:~ # logout
```

#### VM 配置（jailhost-b）

安装 `sysutils/vm-bhyve` 并创建一个”manual”类型（不由 vm-bhyve 管理）的交换机，引用前面创建的桥接接口（switch222）：

```sh
root@jailhost-b:~ # pkg install vm-bhyve
...
root@jailhost-b:~ # service vm enable
vm enabled in /etc/rc.conf
root@jailhost-b:~ # sysrc vm_dir=zfs:zroot/vms
vm_dir:
-> zfs:zroot/vms
root@jailhost-b:~ # zfs create zroot/vms
root@jailhost-b:~ # vm init
root@jailhost-b:~ # vm switch create -t manual -b switch222 switch222
root@jailhost-b:~ # vm switch list
switch222
manual
switch222
n/a
no
n/a
n/a
n/a
```

下载 FreeBSD ISO 并安装名为”guest-b”的 VM；让新 VM 在引导时启动：

```sh
root@jailhost-b:~ # vm iso https://download.freebsd.org/ftp/releases/\
ISO-IMAGES/12.1/FreeBSD-12.1-RELEASE-amd64-bootonly.iso
root@jailhost-b:~ # vm create guest-b
root@jailhost-b:~ # vm add -d network -s switch222 guest-b
root@jailhost-b:~ # vm install \
guest-b FreeBSD-12.1-RELEASE-amd64-bootonly.iso
root@jailhost-b:~ # vm console guest-b
... （将 vtnet1 的 IP 设为 10.0.222.23/24，默认网关设为 10.0.222.1）
root@jailhost-b:~ # sysrc vm_list+="guest-b"
root@jailhost-b:~ # vm Stop guest-b
```

启动 VM 并测试连接（注意公共流量如何通过 VXLAN 222 发送并在网关主机 NAT，因为网关主机配置为默认网关）：

```sh
root@jailhost-b:~ # vm start guest-b
root@jailhost-b:~ # vm console guest-b
root@guest-b:~ # traceroute www.freebsd.org
10.0.111.1 (10.0.111.1)
1.251 ms
1.421 ms
1.070 ms
...
root@guest-b:~ # logout
$ ^D
```

最后测试重启后一切是否正常启动：

```sh
root@jailhost-b:~ # reboot
```

### VXLAN 多播故障排查

有多种工具可帮助排查 VXLAN 设置。

使用 `ifconfig(8)` 确认接口确实正确配置：

```sh
root@jailhost-b:~ # ifconfig vxlan111
vxlan111: flags=8943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST>
metric 0 mtu 1450
options=80000<LINKSTATE>
ether 58:9c:fc:10:ff:fe
inet 10.0.111.20 netmask 0xffffff00 broadcast 10.0.111.255
inet 10.0.111.21 netmask 0xffffffff broadcast 10.0.111.21
groups: vxlan
vxlan vni 111 local 192.168.0.20:4789 group 239.0.0.111:4789
media: Ethernet autoselect (autoselect <full-duplex>)
status: active
nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
```

> **注意**：MTU 设为 1450（因为 VXLAN 使用 50 字节头部信息）。如果所有组件都支持，建议在网络中配置 jumbo frames。

使用 `sockstat(1)` 确认主机确实监听标准 VXLAN 端口 4789：

```sh
root@jailhost-b:~ # sockstat -l4 | grep 4789
?
?
?
?
udp4
*:4789
*:*
```

使用 `ifmcstat(8)` 验证多播配置合理：

```sh
root@jailhost-b:~ # ifmcstat -i em0
em0:
inet 192.168.0.20
igmpv3 rv 2 qi 125 qri 100 uri 3
group 239.0.0.222 mode exclude
mcast-macaddr 01:00:5e:00:00:de
group 239.0.0.111 mode exclude
mcast-macaddr 01:00:5e:00:00:6f
group 224.0.0.1 mode exclude
mcast-macaddr 01:00:5e:00:00:01
```

使用 `tcpdump(1)` 检查 VXLAN 流量：

```sh
root@jailhost-b:~ # tcpdump -vni em0 -f "udp && port 4789"
tcpdump: listening on em0, link-type EN10MB (Ethernet), capture size\
262144 bytes
14:24:13.288186 IP (tos 0x0, ttl 64, id 9404, offset 0, flags [none],\
proto UDP (17), length 78)
192.168.0.20.17657 > 239.0.0.111.4789: VXLAN, flags [I] (0x08), vni 111
ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 10.0.111.1 tell\
10.0.111.20, length 28
...
```

使用 `arp(8)` 检查 ARP 表（在 VM 内也可）：

```sh
root@jailhost-b:~ # arp -an
? (10.0.222.22) at 02:0e:35:b7:6d:01 on switch222 permanent [bridge]
? (10.0.111.22) at 02:0e:35:b7:6d:00 on switch111 permanent [bridge]
? (10.0.222.20) at 58:9c:fc:10:ff:81 on vxlan222 permanent [ethernet]
? (10.0.111.21) at 58:9c:fc:10:ff:fe on vxlan111 permanent [ethernet]
? (10.0.111.20) at 58:9c:fc:10:ff:fe on vxlan111 permanent [ethernet]
? (10.0.111.12) at 02:f7:16:28:83:00 on vxlan111 expires in 1160 seconds\
[ethernet]
...
```

使用 `sysctl(8)` 检查 VXLAN 转发表：

```sh
root@jailhost-b:~ # sysctl net.link.vxlan.111.ftable.dump
net.link.vxlan.111.ftable.dump:
D 0x01 02:F7:16:28:83:00
192.168.0.10 00002343
D 0x01 58:9C:FC:10:FF:E9
192.168.0.10 00002378
D 0x01 00:BD:0B:07:F7:01
192.168.0.10 00002299
D 0x01 58:9C:FC:03:25:35
192.168.0.10 00002347
...
root@jailhost-b:~ # sysctl net.link.vxlan.222.ftable.dump
...
```

清空 ARP 和 VXLAN 转发表：

```sh
root@jailhost-b:~ # arp -ad
10.0.111.12 (10.0.111.12) deleted
10.0.111.10 (10.0.111.10) deleted
192.168.0.10 (192.168.0.10) deleted
root@jailhost-b:~ # ifconfig vxlan111 vxlanflush
root@jailhost-b:~ # ifconfig vxlan222 vxlanflush
```

如果环境允许，故障排查期间临时禁用主机防火墙，确保它们不干扰流量。

## 结论与延伸阅读

本文意在通过示例展示 FreeBSD 虚拟化特性的各种网络配置方式。有意选择使用 Ports/包中的第三方工具，因为许多用户在实践中会尝试这些工具起步。即使特定工具不会进入生产环境，原型阶段使用它们更易实验、感知已配置的内容，再构建更精简的方案。

所示示例绝非穷举，但应有助于读者入门，找出哪种配置可能满足其具体需求。建议进一步阅读以充分理解可用选项及其影响。 •

---

**Michael Gmelin** 在巴伐利亚乡村长大，90 年代在一家用户友好的 rural userfriendly.org 风格 ISP 开始使用 FreeBSD。本世纪头十年大部分时间在开发软件以自动化 ISP 和网络运营商流程，接近十年末开始为项目贡献。后来投身新兴金融科技行业构建支付系统，最终在 2014 年成为 FreeBSD committer。他用多种语言写过软件，最近喜欢上 Rust。业余时间他喜欢玩经典电子游戏和运行 demoscene 作品。离开键盘时，他享受（制作）音乐、骑行、木工和烹饪。Perl 在他心里永远有一席之地。

## 一些主题信息的优秀来源

- Michael W Lucas —— FreeBSD Mastery: Jails（`https://mwl.io/nonfiction/os#fmjail`）
- John Nielsen —— Using VXLAN to network virtual machines, jails, and other fun things on FreeBSD（`https://www.bsdcan.org/2016/schedule/events/715.en.html`）
  - 幻灯片（`https://www.bsdcan.org/2016/schedule/attachments/341_VXLAN_BSDCan2016.pdf`）
  - 视频（`https://www.youtube.com/watch?v=_1Ne_TgF3MQ`）
- iocage: A FreeBSD Jail Manager（`https://iocage.readthedocs.io`）
- vm-bhyve: Shell based, minimal dependency bhyve manager（`https://github.com/churchers/vm-bhyve`）
- pot: another container framework for FreeBSD, based on jails, ZFS and pf（`https://github.com/pizzamig/pot`）
- FreeBSD as a Host with bhyve（`https://www.freebsd.org/doc/handbook/virtualization-host-bhyve.html`），涵盖直接使用局域地址设置 bhyve。
- FreeBSD ifconfig(8) man page（`https://www.freebsd.org/cgi/man.cgi?query=ifconfig&sektion=8`）
- FreeBSD bridge(4) man page（`https://www.freebsd.org/cgi/man.cgi?query=bridge&sektion=4`）
- FreeBSD vxlan(4) man page（`https://www.freebsd.org/cgi/man.cgi?query=vxlan&sektion=4`）
