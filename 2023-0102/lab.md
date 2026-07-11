# 虚拟实验室——BSD 编程研讨会

- 原文：[Virtual Lab – BSD Programming Workshop](https://freebsdfoundation.org/wp-content/uploads/2023/02/angel_virtuallab.pdf)
- 作者：**ROLLER ANGEL**

我们的虚拟实验室将由一台 FreeBSD 主机系统组成，该系统利用 FreeBSD Jail 技术为实验室中我们想要安装的每个系统提供独立的环境，以运行服务并执行其任务。这些任务可以是多种多样的事情，比如提供网页、存储和检索数据库记录、查询和应答 DNS 请求、缓存系统更新文件等。我们的目标是建立一个坚实的基础，以支持虚拟实验室未来的扩展。考虑到 FreeBSD Jail 的本质是为容纳我们的服务而提供轻量级系统，我们可以放心，无论我们发现多少个“兔子洞”，都不会因为达到资源限制而约束我们的创造力或探索。我们可以为每个想法创建一个独立的环境，而不必担心为支持该想法所需服务而部署另一套操作系统所产生的费用。

我尝试过许多不同的操作系统安装托管方法，每次配置新的 Jail 时，那种安心感都让我惊喜不已！它太经济了！我不再受财务担忧的困扰，也不必再考虑这项开支将持续多长时间，它只是我不断扩展的实验室中的又一个环境，本身并不附带最低月租费用。我们不必过于技术化地深入探讨电力、互联网连接、主机硬件等成本问题；是的，我同意这些都有成本，但一旦这些资源到位，添加新主机远不像初始实验室设置那样需要大量考量和费用。

我建议将现有的机器重新利用作为主机，甚至可以是一台笔记本电脑，因为它自带电池备份，在持续断电的情况下能给你时间从容关闭系统。关于需要多个物理网络接口来连接物理以太网交换机、以太网电缆和接入点供物理主机使用的问题，在我们的虚拟实验室中并不重要。这些接口可以通过文本文件中的文字创建，虚拟以太网电缆也可以用来连接我们虚拟网络中的各个组件。

## FreeBSD 主机

这台主机需要几个网络接口。我们希望主机能够有自己的互联网连接方式。这可以是由本地路由器分配的经典 DHCP 地址，或者主机可以充当路由器的角色，通过调制解调器直接连接到外部世界。无论主机如何获得互联网连接，我们都希望有一个额外的网络接口，仅供我们的实验室网络使用。这个接口的名称可能是 `em0`、`igb0` 或类似名称，具体取决于网卡驱动程序和安装的接口数量。即使我们没有多个物理网络接口，也总是可以创建一个单独的网络接口与 `cloned_interfaces` 一起使用。

就我而言，我正在重新利用一些笔记本电脑。一台笔记本电脑有一个可用的无线网卡，用来连接互联网。这是一台来自 2010 年代初期的老款 Asus ROG 笔记本，它也内置了以太网口。我的 X1 Carbon 7 代 ThinkPad 没有内置的以太网口，我只使用 USB 端口和 Android USB 网络共享。然而，最让我兴奋的是 Dell Precision 笔记本电脑。它是一台很棒的实验室主机，配备 Intel Xeon 处理器，能够支持 128GB 内存和多块 NVME 固态硬盘。笔记本背面有一个内置的以太网端口，第二个则是一个 USB-C 转接器，上面有一个 Intel igb0 以太网端口，我用它作为实验室网络的专用接口。最后，旧的塔式机箱也是很好的主机选择，你可以安装一些带有多个以太网端口的出色 PCI 网卡。

关键是找到适合你的方案，安装 FreeBSD，然后开始我们的虚拟实验室。如果你想了解一些安装和配置 FreeBSD 的技巧，请查看 2022 年 7/8 月号《FreeBSD 期刊》中的《Getting Started With FreeBSD Workshop》。

## 虚拟网络设计

就像在物理服务器世界中，我们的路由器/防火墙系统上有多个物理以太网端口一样，我们可以为实验室的路由器/防火墙系统配置多个接口，从现在开始我将其称为网关。这些接口将通过桥接相互连接。可以将桥接视为物理网络中的以太网交换机，桥接提供类似的功能。对于我们的以太网电缆，将使用 **epair(4)** 接口。它们由两端组成，A 端和 B 端。我们将 A 端连接到桥接，B 端连接到我们的 Jail。这样，所有的 Jail 就可以通过虚拟桥接相互通信，就像物理主机通过以太网电缆连接到交换机后能够相互通信一样。

网关 Jail 将有一根额外的虚拟电缆，用于连接到一个单独的桥接，该桥接允许从实验室内部连接到外部互联网。所有的 Jail 都会将默认路由指向这个网关。我们通过将一个物理接口分配为桥接的成员，使该独立桥接能够连接到外部世界。同样地，这类似于将以太网电缆插入交换机，但在这种情况下，以太网电缆插入到 FreeBSD 主机系统中，然后虚拟地插入到桥接中。通过这种方式，属于同一桥接的其他虚拟以太网电缆将能够与其通信，并使数据包在实验室环境内外流动。

分解来看：每个 Jail 都使用一种叫做 vnet 的技术获得自己的网络。我们将 FreeBSD 主机上的物理接口连接到一个虚拟桥接上，并为实验室网络创建第二个虚拟桥接，只有实验室 Jail 的虚拟接口连接到该桥接。然后，我们使用路由和防火墙规则按需转发数据包。

## 防火墙

我们将使用 PF 作为防火墙。FreeBSD Jail 的工作方式是每个 Jail 都共享主机内核，因此如果你希望在 Jail 内访问某个内核模块，只需允许它，然后在 `jail.conf` 设置中配置该 Jail 的访问权限。请查看文件 **/etc/defaults/devfs.rules**。为了让网关使用 PF 防火墙，我们需要设置 Jail 的配置，使其使用此文件中列出的 pf 规则集。稍后我们会设置自定义的 devfs 规则，并包含 `devfsrules_jail_vnet` 规则中的配置。

## 配置

现在我们了解了要做什么，接下来开始实际操作。我们从运行 FreeBSD 13.1-RELEASE 的主机开始。正如上面所描述的，这台主机应该有一个可用的互联网连接。我们将利用该连接下载一些文件，用于创建我们的 Jail。主机上运行着 NTPD，因此能够获取准确的时间。使用 `sockstat -46` 检查主机上是否有任何服务在监听，如果有不需要的服务，请将其关闭。记住，主机应保持精简——我们将在实验室的 Jail 内进行很多有趣的工作，因此尽量限制主机上的服务。我计划通过键盘和显示器直接登录主机进行管理，因此没有启用主机的 SSH。

现在，我们准备启用 Jail。只需运行 `sysrc jail_enable=YES` 即可。无需安装任何软件包，Jail 管理功能内置于 FreeBSD 中。可以查看 **/usr/share/examples/jails** 中的 `README` 文件，了解如何配置你的 Jail。正如你所看到的，有很多方法可以进行 Jail 配置。我做了研究，并选择了本文使用的配置方法。你可以尝试其他方法，看看哪个最适合你。如果你采取了不同的方式，建议写一些内容分享你的经验，让其他人也能看到你所发现的有用之处并亲自尝试。

此时，我们确认主机准备好托管 Jail，并启用了 Jail 服务，因此可以快速重启，再次确认主机上只有最少的服务在监听，然后开始为所有 Jail 创建基本配置。编辑配置文件时，我们将使用 vim，对于基本的编辑任务，你只需要了解一些基本操作，花几分钟时间通过 `vimtutor` 命令的互动练习就能掌握它，你很快就能成为 vim 新手。

注意：我们将以 root 用户身份运行以下所有命令。输入 `sudo -i` 切换到 root 用户。

编辑 `jail.conf`：

```sh
vim /etc/jail.conf
```

将以下行放入 **/etc/jail.conf**：

```sh
$labdir="/lab";
$domain="lab.bsd.pw";
path="$labdir/$name";
host.hostname="$name.$domain";
exec.clean;
exec.start="sh /etc/rc";
exec.stop="sh /etc/rc.shutdown";
exec.timeout=90;
stop.timeout=30;
mount.devfs;
exec.consolelog="/var/tmp/${host.hostname}";
```

`base.txz`

```sh
mkdir -p /lab/media/13.1-RELEASE
cd /lab/media/13.1-RELEASE
fetch http://ftp.freebsd.org/pub/FreeBSD/releases/amd64/13.1-RELEASE/base.txz
```

网关 Jail

```sh
mkdir /lab/gateway
tar -xpf /lab/media/13.1-RELEASE/base.txz -C /lab/gateway
```

编辑 `jail.conf`：

```sh
vim /etc/jail.conf
```

将以下行放入 **/etc/jail.conf**：

```sh
gateway {
 ip4=inherit;
}
```

可以随意使用以下可选命令为 Jail 添加一个用户账户，但在本文中我们将只使用 root 用户。

```sh
chroot /lab/gateway adduser
```

为 Jail 设置 root 密码：

```sh
chroot /lab/gateway passwd root
```

使用 OpenDNS 服务器设置 DNS 解析：

```sh
vim /lab/gateway/etc/resolv.conf
```

把以下行放入 `resolv.conf`：

```sh
nameserver 208.67.222.222
nameserver 208.67.220.220
```

复制主机时区设置：

```sh
cp /etc/localtime /lab/gateway/etc/
```

创建一个空的文件系统表：

```sh
touch /lab/gateway/etc/fstab
```

启动 Jail：

```sh
jail -vc gateway
```

登录到 Jail：

```sh
jexec -l gateway login -f root
```

退出 Jail：

```sh
logout
```

列出所有 Jail：

```sh
jls
```

停止 Jail：

```sh
jail -vr gateway
```

创建 `devfs.rules`：

```sh
vim /etc/devfs.rules
```

在 `devfs.rules` 中添加以下行：

```sh
[devfsrules_jail_gateway=666]
add include $devfsrules_jail_vnet
add path bpf* unhide
```

重新启动 devfs 服务：

```sh
service devfs restart
```

验证 devfs 规则：

```sh
devfs rule showsets
```

将新的规则集分配给网关 Jail：

```sh
vim /etc/jail.conf
```

在 `gateway` 配置块中添加以下行：

```sh
devfs_ruleset=666;
```

重新启动网关 Jail：

```sh
service jail restart gateway
```

验证规则集是否已应用到网关 Jail：

```sh
jls -j gateway devfs_ruleset
```

我们期望看到上述命令输出 `666`。

在主机上加载 PF 内核模块：

```sh
sysrc -f /boot/loader.conf pf_load=YES
kldload pf
```

在网关 Jail 上启用 PF：

```sh
sysrc -j gateway pf_enable=YES
```

编辑网关 Jail 上的 `pf.conf`：

```sh
vim /lab/gateway/etc/pf.conf
```

写入以下配置：

```sh
ext_if = "e0b_gateway"
int_if = "e1b_gateway"
table <rfc1918> const { 192.168.0.0/16, 172.16.0.0/12, 10.0.0.0/8 }

# 允许回环接口上的所有流量
set skip on lo0

# 规范化所有传入流量
scrub in

no nat on $ext_if from $int_if:network to <rfc1918>

# 对外出流量进行 NAT
nat on $ext_if inet from $int_if:network to any -> ($ext_if:0)

# 拒绝带有伪造地址的流量
antispoof quick for { $int_if, lo0 } inet

# 默认阻止所有传入流量，但允许所有传出流量
block all
pass out all

# 允许 LAN 访问外部网络
pass in on $int_if from any to any
block in on $int_if from any to self

# 允许 LAN ping 网关
pass in on $int_if inet proto icmp to self icmp-type echoreq
```

## 配置虚拟网络

设置一个接口。在这里，我们使用名为 `alc0` 的专用网卡，并将其命名为 `lab0`。所有后续的配置都将使用接口名称 `lab0`，实际分配给它的物理设备可以通过编辑一行配置来更改。通过这种方式，如果我们将实验室迁移到另一台主机或将来添加一个新的网络接口（例如 `igb0`、`re0` 或 `em0`），可以轻松切换接口。

```sh
sysrc ifconfig_alc0_name=lab0
sysrc ifconfig_lab0=up
service netif restart
```

将 Jail 接口桥接自动化脚本复制到我们的实验室脚本目录中，并使其可执行：

```sh
mkdir /lab/scripts
cp /usr/share/examples/jails/jib /lab/scripts/
chmod +x /lab/scripts/jib
```

编辑 `jail.conf`：

```sh
vim /etc/jail.conf
```

## 网关 jail.conf

此时，我们准备好从主机继承 ip4 网络并改用 vnet，移除 **/etc/jail.conf** 中的配置块 `gateway {}`，并将其替换为以下内容：

```sh
gateway {
 vnet;
 vnet.interface=e0b_$name, e1b_$name;
 exec.prestart+="/lab/scripts/jib addm $name lab0 labnet";
 exec.poststop+="/lab/scripts/jib destroy $name";
 devfs_ruleset=666;
}
```

为实验室中的 Jail 创建内部 LAN 网络：

```sh
sysrc cloned_interfaces=vlan2
sysrc ifconfig_vlan2_name=labnet
sysrc ifconfig_labnet=up
service netif restart
```

销毁并重新创建网关：

```sh
jail -vr gateway
jail -vc gateway
```

配置网关 Jail 的网络设置：

```sh
sysrc -j gateway gateway_enable=YES
sysrc -j gateway ifconfig_e0b_gateway=SYNCDHCP
sysrc -j gateway ifconfig_e1b_gateway="inet 10.66.6.1/24"
service jail restart gateway
jexec -l gateway login -f root
```

测试连通性：

```sh
host bsd.pw
ping -c 3 bsd.pw
```

退出 Jail：

```sh
logout
```

创建另一个只有一个接口并连接到 labnet LAN 网络的 Jail：

编辑 **/etc/jail.conf** 文件，在文件末尾添加以下内容：

```sh
client1 {
 vnet;
 vnet.interface="e0b_$name";
 exec.prestart+="/lab/scripts/jib addm $name labnet";
 exec.poststop+="/lab/scripts/jib destroy $name";
 devfs_ruleset=4;
 depend="gateway";
}
```

为新 Jail 创建目录结构：

```sh
mkdir /lab/client1
tar -xpf /lab/media/13.1-RELEASE/base.txz -C /lab/client1
```

设置 root 密码：

```sh
chroot /lab/client1 passwd root
```

使用 OpenDNS 服务器设置 DNS 解析：

编辑 **/lab/client1/etc/resolv.conf** 文件，添加以下行：

```sh
nameserver 208.67.222.222
nameserver 208.67.220.220
```

复制主机的时区设置：

```sh
cp /etc/localtime /lab/client1/etc/
```

创建一个空的文件系统表：

```sh
touch /lab/client1/etc/fstab
```

启动 Jail：

```sh
jail -vc client1
```

配置 `client1` Jail 的网络：

```sh
sysrc -j client1 ifconfig_e0b_client1="inet 10.66.6.2/24"
sysrc -j client1 defaultrouter="10.66.6.1"
service jail restart client1
```

登录到 Jail：

```sh
jexec -l client1 login -f root
```

测试连通性：

```sh
host bsd.pw
ping -c 3 10.66.6.1
ping -c 3 bsd.pw
```

获取示例的 tcsh 配置文件：

```sh
fetch -o .tcshrc http://bsd.pw/config/tcshrc
chsh -s tcsh
```

退出 Jail：

```sh
logout
```

下次登录时，由于 tcshrc 配置的设置，你会看到绿色的提示符，尽情享受吧！

现在，你有了一个虚拟实验室，拥有自己的虚拟网络，并为外部互联网连接保留了一个物理接口。由于我们将接口命名为 `lab0`，所以可以很容易地更新它。试试看吧！例如，你可以插入一部安卓手机，手机有互联网连接，无论是 WiFi 还是移动数据都可以。将手机连接到主机后，进入手机的网络设置并启用 USB 网络共享。一个名为 `ue0` 的接口将可供使用。接着，更新 **/etc/rc.conf** 中如下行：

```sh
ifconfig_alc0_name="lab0"
```

改为：

```sh
ifconfig_ue0_name="lab0"
```

重启系统。

登录到任意一个 Jail，测试连接。你的网络切换完成。现在你的实验室可以移动了！

我希望你在跟随这篇文章的过程中玩得开心。我非常热衷于 FreeBSD，与他人分享所学给我带来快乐。我希望你在实验室中用 FreeBSD 做出精彩的事情，也期待在精彩的 BSD 大会上与你们中的许多人交流。

---

**ROLLER ANGEL** 大部分时间都在帮助人们学习如何使用技术实现他们的目标。他是狂热的 FreeBSD 系统管理员和 Python 爱好者，喜欢学习用开源技术——特别是 FreeBSD 和 Python——所能完成的奇妙之事来解决各种问题。他坚信只要人们下定决心，就能学会任何想学的东西。Roller 总是在寻找创造性的解决方案来解决问题，享受挑战。他有着强烈的动力，喜欢学习、探索新想法并保持技能熟练。他喜欢参与研究社区并分享他的想法。
