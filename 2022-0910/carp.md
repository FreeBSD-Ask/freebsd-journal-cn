# CARP 简介

- 原文地址：[Introduction to CARP](https://freebsdfoundation.org/wp-content/uploads/2022/11/zaborski_CARP.pdf)
- 作者：**MARIUSZ ZABORSKI**

在大型网络中，高可用性问题可能会变得充满挑战与复杂性，因此我们应当寻求那些简单、易于维护和理解的解决方案。毫无疑问，CARP 协议正是其中之一。CARP 是 Common Address Redundancy Protocol，共用地址冗余协议的缩写，其基本功能是让多台主机共享一组 IP 地址。CARP 协议并非新鲜事物——它最初作为 CISCO 协议 VRRP 的替代方案，于 2003 年出现在 OpenBSD。由于专利问题（超出本文讨论范围），CARP 被用来取代 VRRP 协议。在 OpenBSD 推出 CARP 后，它随后被集成到了 FreeBSD 和 NetBSD 中。最终，ucarP 被引入——这是一种 CARP 协议的用户态实现，为内核实现提供了另一种选择，并使其在 Linux 上也得以使用。

## CARP 背景

CARP 可构建一个冗余组——一组共享 IP 地址的主机。然而，从物理上讲，只有一个网络接口拥有这些 IP 地址（该接口称为活跃主机）。当活跃主机消失（例如被关闭或网络出现问题）时，冗余组中的其他主机会检测到这一情况，并选举出新的活跃主机。该情况如图 1 所示。

在这个冗余组中，有两台机器——蓝色和绿色，但只有蓝色的是活跃节点，因此网络中的其他机器能够顺利地选择连接目标。

**图 1. 活跃与被动的 CARP 节点**

![](https://github.com/user-attachments/assets/90eacfa5-5025-46b1-b584-f0b94af3c996)

在 CARP 中，活跃节点会广播其状态，如图 2 所示。由于蓝色主机是活跃节点，因此它正在广播 CARP 数据包，而绿色和/或橙色节点则处于待命状态，不发送任何数据包。CARP 数据包体积很小，仅包含一些基本信息，例如：

- **vhid（虚拟服务器 ID）**：用于标识冗余组；冗余组中的所有机器必须共享相同的 vhid  
- 关于 CARP 版本以及 CARP 数据包类型的信息

CARP 中的所有数据包均经过加密签名，这意味着冗余组中的每个节点都必须共享同一密钥。CARP 永远不会以明文形式将密码发送到网络上。非常重要的一点是，冗余组中的每台机器都必须配置有完全相同的一组 IP 地址。不过，这些 IP 地址并不会在网络上传输——它们仅用于计算加密签名。在图 2 中所示的情形下，只要蓝色服务器使用指定的 vhid 正确地广播经过加密签名的数据包，其余节点便不执行任何操作，只是处于监听状态。

**图 2. CARP 数据包广播**

![](https://github.com/user-attachments/assets/31f6e10e-b01f-4efe-b2d6-152e714538c2)

当某个节点一段时间内未能接收到 CARP 数据包时，另一节点便会介入并成为活跃节点。图 3 展示了这一情况。由于某种原因，蓝色节点停止了数据包广播，绿色节点发现这一情况后便开始广播 CARP 数据包。当蓝色节点恢复时，它会注意到绿色节点已成为活跃节点，从而保持被动状态。

**图 3. 新的活跃节点**

![](https://github.com/user-attachments/assets/bf2da29e-2f53-434d-88c8-199c26f9c952)


以上所有示例均展示了冗余组中只有一个 IP 地址的情况。但实际上，冗余组可以包含多个 IP 地址，而且主机也可以属于多个冗余组——这可以通过不同的 vhid 实现。得益于这一特性，我们还可以在网络中的服务之间实现某种程度的负载均衡。例如，绿色节点可以在提供 Web 服务器服务的冗余组中担任活跃节点，而蓝色节点则在提供时间服务的冗余组中担任活跃节点。如果其中某一节点消失，另一节点则会在相应冗余组中成为活跃节点。

## CARP 与脑裂

在某种情况下，两个节点同时发现某个节点消失，便可能同时尝试成为活跃节点。这种情况被称为脑裂，即出现多个活跃节点的情况。当节点之间的链路断开，彼此无法接收对方的数据包，经过一段时间后问题恢复时也可能出现这种情况，如图 4 所示。

CARP 同样能解决这一问题。当两个主机均处于活跃状态时，它们都在广播 CARP 数据包。广播数据包频率更高、在更短时间内发送更多数据包的节点会被优先选为新的主控节点。这一机制在 CARP 中通过优先级来控制，优先级越低表示数据包发送频率越高。当另一节点发现对方的数据包发送频率比自己高时，就会自动切换回被动模式。在两个节点以相同优先级发送数据包的情况下，将随机选择其中一节点作为活跃节点。

**图 4. 脑裂情况**

![](https://github.com/user-attachments/assets/dfccd1fc-fa41-4af0-a593-e80295450110)

## FreeBSD 内核模块 CARP 配置

CARP 模块包含在 FreeBSD 的默认安装中。从 FreeBSD 10.0 开始，CARP 不再是伪接口，而是直接在网络接口上进行配置。列表 1 展示了 CARP 的基本配置。首先，我们需要加载 FreeBSD 的 CARP 模块，这可以通过 **kldload(8)** 命令完成。然后，使用 **ifconfig(8)** 命令，定义 CARP 应在哪个接口上工作（在我们的例子中是 **em0**）。接下来，我们定义冗余组的 ID（将 **vhid** 设为 1）。另一个重要的配置项是用于计算校验和的密码短语；这个密码短语必须在冗余组内的所有主机之间共享。在命令中，我们还定义了优先级（或称为广告间隔）。这由两个参数控制：  

- **advbase**（广告基准），以秒为单位指定；  
- **advskew**（广告偏移——此参数在列表中未显示），以 1/256 秒为单位测量。  

需要提醒的是——较低的优先级意味着主机广告发送得更频繁，也就表示它是优选节点。最后，我们定义了浮动地址。在同一列表中，有两次 **ifconfig(8)** 的运行；与 CARP 无关的部分已被省略。在第一次运行中，我们可以看到冗余组处于 **BACKUP** 状态，这意味着该接口处于待命模式，正监听 CARP 数据包。由于网络中没有 CARP 数据包，它便切换为 **MASTER**（活跃）状态，并开始广播其状态。在图 5 中，我们可以看到捕获到的 CARP 数据包，其使用第二个静态 IP 地址向多播 IP 地址发送 CARP 数据包。因此，除了共享的 IP 地址外，还必须配置一个额外的 IP 地址。

**清单 1. FreeBSD 中 CARP 的配置**

```sh
# kldload carp
# ifconfig em0 vhid 1 pass randompass advbase 1 alias 192.168.1.50/32
# ifconfig
em0:
    inet 192.168.1.50 netmask 0xffffffff broadcast 192.168.1.50 vhid 1
    carp: BACKUP vhid 1 advbase 1 advskew 0
# ifconfig
em0:
    inet 192.168.1.50 netmask 0xffffffff broadcast 192.168.1.50 vhid 1
    carp: MASTER vhid 1 advbase 1 advskew 0
    status: active
```


**图 5. 使用 Wireshark 捕获的 CARP 流量**

![](https://github.com/user-attachments/assets/808433d6-02fe-4f66-a2cd-80557fcb97d9)

或许有用的一点是，FreeBSD 的 devd(8) 守护进程允许在状态发生变化时运行额外的脚本。列表 2 展示了来自 FreeBSD 手册页的一个此类配置示例。当冗余组状态改变时，将执行 `/root/carpcontrol.sh` 脚本，第一个参数为 `vhid@inet`，第二个参数为当前组状态。

列表 2. 针对 CARP 的 devd(8) 配置

```sh
notify 0 {
       match "system" "CARP";
       match "subsystem" "[0-9]+@[0-9a-z.]+";
       match "type" "(MASTER|BACKUP)";
       action "/root/carpcontrol.sh $subsystem $type";
};
```

## ucarp

另外，一个非常有前景的项目是 **ucarp**，即 CARP 协议的用户态实现。它减少了内核空间中的代码量。而内核空间的实现可能略有不同，在那种情况下，代码库是为多个平台共享的。然而，该项目似乎已经被放弃——GitHub 上的项目已关闭，且 ucarp 域名已过期。不过，你仍然可以在不同操作系统上找到 ucarp 的分发版本，因此如果你在寻找跨平台的实现，我们仍然推荐你关注该项目。它的配置选项与内核实现非常相似。  

列表 1 展示了如何在 FreeBSD 系统上安装 ucarp，紧接着的命令显示了其基本用法。此时，大部分选项都不言自明。下面我们详细介绍一下 **upscript** 和 **downscript** 选项。由于 ucarp 被设计为一个多平台工具，它本身并不知道如何将 IP 地址添加到网络接口上——这部分工作由管理员负责。用户必须自行定义脚本，以便将 IP 地址添加到正确的接口上。

**清单 3. ucarp 的基本用法**

```sh
# pkg install carp
# ucarp --interface=eth0 --srcip=192.168.1.157 --vhid=1 --pass=randompass
--addr=192.168.1.50 --upscript=up.sh --downscript=down.sh
```

另一个关于 ucarp 的小问题是：我们只能为该协议定义一个单独的浮动 IP 地址。虽然我们可以在 upscript 和 downscript 中添加许多 IP 地址，但只有一个（来自参数 addr）会被加入到加密校验中。这意味着如果我们希望混合使用内核和用户态的 CARP 实现，则在单个冗余组中使用多个浮动地址将无法正常工作，因为校验和不匹配。



## 总结

CARP 是一款简单但功能强大的工具，能够为我们的网络提供高可用性。主要有两种 CARP 实现：一种是在内核空间中实现的（每个 BSD 操作系统均自带此实现），另一种是跨平台的用户态实现 ucarp（亦支持 Linux）。遗憾的是，用户态实现似乎已经被放弃。不过，如果你正在寻找一个简单易用的解决方案来提供浮动地址，那么你仍然可以考虑使用它。



## 参考文献

- **CARP on Wikipedia**  
  [https://en.wikipedia.org/wiki/Common_Address_Redundancy_Protocol](https://en.wikipedia.org/wiki/Common_Address_Redundancy_Protocol)
  
- **UCARP GitHub Project**  
  [https://github.com/jedisct1/UCarp](https://github.com/jedisct1/UCarp)
  
- **CARP in FreeBSD Handbook**  
  [https://docs.freebsd.org/en/books/handbook/advanced-networking/#carp](https://docs.freebsd.org/en/books/handbook/advanced-networking/#carp)
  
- **CARP FreeBSD man page**  
  [https://www.freebsd.org/cgi/man.cgi?query=carp&sektion=4](https://www.freebsd.org/cgi/man.cgi?query=carp&sektion=4)


## 致谢

本文中的图片资源来自 [flaticon.com](https://www.flaticon.com)。

---

**MARIUSZ ZABORSKI** 目前在 4Prime 担任安全专家。他自 2015 年起就自豪地拥有 FreeBSD 提交权限。他的主要兴趣领域是操作系统安全和底层编程。Mariusz 曾在 Fudo Security 工作，领导团队开发 IT 基础设施中最先进的 PAM 解决方案。2018 年，他组织了波兰 BSD 用户组。在业余时间，Mariusz 喜欢在 [https://oshogbo.vexillium.org](https://oshogbo.vexillium.org) 上撰写博客。
