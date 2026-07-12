# 使用 Mininet 快速浏览 SDN

从 20 世纪 90 年代中期开始，我就不间断地使用 FreeBSD 和 Linux。多年来我运行过许多不同的 Linux 发行版，最近专注于 CentOS。我也有大量 Mac OS X 和 NetBSD 的使用经验，并尝试过许多其他 BSD 平台。作为坚信开放标准价值的不可知论者，我喜欢熟悉 POSIX 世界的所有选项，以便随时为工作选择最佳工具。

作者：**Ayaka Koshibe**

从最基本的意义上说，软件定义网络（Software-Defined Networking，SDN）可以视为一种构建网络的方法，使其能够像单一逻辑实体一样被管理。基于 SDN 的网络通常由可编程的白盒交换机和软件交换机构建，并由控制应用程序管理，这些应用程序利用对网络的全局视图来协调交换机协同工作。对于习惯于“经典”网络（按设备逐个配置，其行为由分布式网络协议决定）的人来说，结果可能看起来相当陌生。网络模拟器是更好地理解这些网络行为和组装方式的有用工具。

## Mininet

Mininet 是相当知名的基于 SDN 的网络模拟器，因与 OpenFlow（SDN 黎明时期的网络控制协议）的关联而普及。它最近也被加入到了 Ports 中。以此为契机，这里以 SDN 入门的形式快速介绍 Mininet。

开始之前：Mininet 依赖 VIMAGE 来模拟网络主机，因此希望跟随操作的读者需要一台支持 VIMAGE 的主机。移植版的 Mininet 也不支持原版的全部功能，目前仍在开发中。它在清理时也比较激进，所以最好不要在用于托管其他 jail 或 Open vSwitch 实例的机器上运行。

## 安装

Mininet 可以像任何其他应用程序一样安装：用 **pkg(8)** 安装为 `py27-mininet`，或从 Ports 树安装为 `net/mininet`。

## mn 命令

Mininet 版的“Hello world”是用 `mn` 命令启动的小型网络：

```sh
# mn --controller=ryu
*** Creating network
  ...
*** Starting CLI:
mininet>
```

这会创建一个网络，其中两台主机通过一台配置为学习交换机的交换机连接。它还启动一个用于与网络交互的 CLI。例如，`links` 显示网络中的所有链路：

```sh
mininet> links
h1-eth0<->s1-eth1 (OK OK)
h2-eth0<->s1-eth2 (OK OK)
```

而 `dump` 显示网络中节点的信息：

```sh
mininet> dump
<Host h1: h1-eth0:10.0.0.1 pid=8410>
<Host h2: h2-eth0:10.0.0.2 pid=8414>
<OVSSwitch s1: lo0:127.0.0.1,s1-eth1:None,s1-eth2:None pid=8420>
<Ryu c0: 127.0.0.1:6653 pid=8401>
```

`?` 或 `help` 会显示所有可用命令。

## 检查控制流量

`dump` 的输出显示每个节点都有名称、一个或多个端口，以及代表它的 bash 进程的 PID。它还显示主机——实际上是 vnet jail——在这个网络中位于 **10.0.0.1** 和 **10.0.0.2**，控制器 Ryu 在端口 6653 上监听交换机。Ryu 使用 OpenFlow 编程连接到此端口的交换机。排查 OpenFlow 交换机和控制器问题的典型方法是在此通道上检查控制消息。我们可以通过在另一个终端运行 `tcpdump`（或其他数据包分析器）来尝试：

```sh
# tcpdump -i lo0 port 6653
```

我们应该能看到在 s1 和 c0 之间发送的保活 ECHO_REQUEST 和 ECHO_REPLY 消息。接下来，从一台主机 ping 另一台主机。CLI 将主机名后的任何命令解释为从该主机运行的 bash 命令：

```sh
mininet> h1 ping -c1 h2
```

主机的名称会被 CLI 转换为相应的 IP 地址。

或者，可以用 `pingall` CLI 命令在所有主机对之间 ping：

```sh
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2
h2 -> h1
*** Results: 0% dropped (2/2 received)
```

在任一情况下，我们都应该看到交换机（localhost.<高端口>）向控制器（localhost.6653）发送 PACKET_IN 消息，还有控制器作为响应发送回的 PACKET_OUT 和 FLOW_MOD 消息。交换机使用 PACKET_IN 将其不知道如何处理的数据包发送给控制器，在这种情况下是 ARP 和 ICMP 消息。控制器使用 PACKET_OUT 指示交换机输出特定数据包（即在 PACKET_IN 中发送的那个，这样它就不会“丢失”），使用 FLOW_MOD 修改交换机处理不同类型流量的方式。在修改因不使用而过期之前，s1 不应因另一次 ping 而生成新的 PACKET_IN。

`Ctrl-D` 或 `exit` 命令将退出 CLI 并拆除网络。

## 试验控制器

控制器在网络中的作用可以通过运行一个没有控制器的网络来直接演示。可以通过向 `mn` 的选项 `--controller` 传递 `none` 而不是 `ryu` 来实现。这个“无头”网络上的主机应该无法相互 ping。

另一个有用的选项是 `remote`，它允许网络使用在 Mininet 控制之外运行的控制器。开发者可能会通过此选项将 Mininet 网络指向他们编写的控制器来测试。假设控制器运行在 **192.168.0.100** 并监听端口 6633，以下命令将启动一个网络并将交换机连接到它：

```sh
# mn --controller=remote,ip=192.168.0.100,port=6633
```

## 创建各种拓扑

选项 `--topo` 用于通过 `mn` 创建各种拓扑。linear 和 tree 拓扑对于创建更大的无环网络很有用，而 torus 拓扑对于测试控制器的环路处理能力很有用。拓扑是参数化的，以便可以指定其大小。例如，创建一个三层高、扇出为二的树：

```sh
# mn --controller=ryu, topo=tree,3,2
```

torus 也接受两个值，linear 接受一个。

## 用 Mininet 编写脚本

Mininet 也可作为 Python 库集合，用于编写实验脚本。需要注意的是，这些示例脚本保持原始形式（很可能无法在 FreeBSD 上运行），包中包含几个示例脚本，演示如何创建自定义拓扑、网络组件和实验。与其他附带示例的应用程序一样，它们应该位于 **/usr/local/share/examples/mininet/** 下。但作为小示例，以下脚本定义了类似于 `mn` 默认拓扑的自定义拓扑，使用主机 h1 ping h2 的地址，然后退出：

```python
from mininet.topo import Topo
from mininet.net import Mininet

class MinimalTopo(Topo):
    def build(self):
        h1 = self.addHost('h1', ip='192.168.0.1')
        h2 = self.addHost('h2', ip='192.168.0.2')
        s1 = self.addSwitch('s1')

        self.addLink(h1, s1)
        self.addLink(s1, h2)

net = Mininet(topo=MinimalTopo())
net.start()
h1 = net.getNodeByName('h1')
print(h1.cmd('ping -c1 192.168.0.2'))
net.stop()
```

保存后，可以像任何 Python 脚本一样运行：

```sh
# python example.py
```

## 了解更多

虽然此 Port 只支持上游功能的一个子集，但主项目维护的资源应该能更好地了解如何使用 Mininet。这些资源可在以下地址找到：

<https://github.com/mininet/mininet/wiki/Documentation>

而 Port 本身维护在：

<https://github.com/akoshibe/mininet>

我们的 Mininet 旋风之旅到此结束。希望它能为那些有兴趣探索 SDN 领域的人提供一个不错的起点。

---

**AYAKA KOSHIBE** 在大学时期协助部署 GENI OpenFlow 校园试验的基础设施而涉足 SDN 领域。她目前在 Big Switch Networks 工作，是 SDN 控制器平台团队的成员，同时也是 FreeBSD 和 OpenBSD 上 Port Mininet 的维护者和上游。
