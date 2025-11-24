# Netgraph 大众教程

将 FreeBSD 强大的网络框架带给所有运行 jail 和虚拟机操作系统的用户

- 原文：[Netgraph for the Rest of Us](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-3/netgraph-for-the-rest-of-us/)
- 作者：Daniel J. Bell

在几年前，我写过一篇文章，讨论如何在数据中心中利用 FreeBSD 作为一种高性能、低成本的云计算替代方案，依靠的是 FreeBSD 基本系统中的工具。随着我的基础设施不断扩展，我开始探索提升性能和管理效率的方法，而构建理想的网络栈成为我的主要目标之一。这促使我尝试使用 Netgraph，而我对它的喜爱程度甚至让我基于 rc 服务机制创建了一个简化其基础配置的工具 —— **ngbuddy(8)**。

Netgraph 是款网络工具包，在 1999 年被引入了 FreeBSD，它采用“节点与连接”的结构，可用于构建复杂而强大的配置，适用于各类特定的网络工作流。你可以将它想象成一个模块化的搭建系统：节点（即网络构件）通过钩子（即连接节点或其他网络对象的链路）连接在一起，从而构建出多层交换机、流量整形器或自定义虚拟网络。最常见的用途，是将虚拟机和 jail guest 连接到宿主机的物理或虚拟网络上，但 Netgraph 也支持以复杂而有趣的方式组合这些组件。举个例子：我在 BSDCan 遇到的一位 FreeBSD 专家，就利用巧妙的 Netgraph 配置，在多个物理链路上“镜像”流量，就像 RAID 那样冗余地传输，从而极大提升了关键 VoIP 业务的稳定性与弹性。

我将在本文中介绍 Netgraph 的核心概念，探讨像 **ngbuddy** 和 **jng** 这样的简化配置工具，演示其底层机制的手动配置过程，并提供进一步探索的资源。无论你是刚接触 FreeBSD 网络的新手，还是想深入研究的资深管理员，都能从中获得切实可用的方法，尝试解锁这一强大的网络框架。

## 为什么 Netgraph 很棒（但也有局限）

在 FreeBSD 中，为虚拟机（VM）和 jail 设置网络的传统方式通常依赖于接口 `if_bridge(4)`、`tap(4)` 和 `epair(4)`。这种做法已经非常成熟，而且还在持续优化：例如在 2020 年，`if_bridge` 获得了一次巨大的性能提升。但不到一年后，FreeBSD 13 为 bhyve 引入了对 Netgraph 的支持，从而为用户提供了一项非常有吸引力的替代方案。

使用 Netgraph 有充分的理由。老实说，如果 `if_bridge` 始终可以跑得更快，我可能不会考虑它。但我在几个不同年代的服务器上，针对 FreeBSD 14.3 jail 和 bhyve VM 之间的网络通信做了一些简单的 `iperf3` 性能测试，对比了 `if_bridge/tap/epair` 与纯 Netgraph 通信方式。尽管测试不够科学严谨，Netgraph 无论在哪种硬件上都 consistently 表现更快。

![性能对比图](https://freebsdfoundation.org/wp-content/uploads/2025/07/bell-chart1.png)

当然，在理想条件之外，Netgraph 在高包速场景下的扩展性会受到限制。其瓶颈主要在于 hook 查找机制的锁竞争。目前已经有一个解决方案正在开发中，希望能赶上 FreeBSD 15，但在那之前，对于像 CDN 前端这种需要高吞吐量的应用来说，Netgraph 仍然不是最佳选择。

不过，对于我们正在讨论的虚拟化密度（每台主机运行几十个混合用途的 bhyve VM 或 jails），Netgraph 的表现是稳定且值得信赖的。在我多个生产环境中的类似部署中，Netgraph 多年来一直保持着优异的吞吐量和稳定性。

除了性能之外，Netgraph 还有一些明显优势：

- **配置更清爽：** 没有太多 ifconfig 虚拟接口的杂乱，命令更聚焦，环境更干净
- **内建指标：** 易于访问的流量统计，便于监控与排错
- **极高的灵活性：** 通过 Netgraph 你可以使用其完整的网络模块功能集
- **可视化能力：** 拓扑是图结构，容易理解主机间的网络结构，还有内建工具可以可视化网络图

Netgraph 的主要劣势就是：**上手门槛较高**。它在 FreeBSD 中拥有 40 余篇 man 页面，涵盖各种概念、工具和内核模块，新手一眼望过去可能完全不知道从哪里开始。但实际上，大多数常见的场景只需要掌握其中少数几个要点就能用起来。


## 学习 Netgraph 基础：节点与钩子

首先，让我们进入节点（nodes）与钩子（hooks）的思维模式。

- **节点（Nodes）**：这些是执行特定网络功能的处理模块。
- **钩子（Hooks）**：节点上的连接点，允许数据在节点之间流动。

要在 bhyve 和 jail 中进行基础网络配置，你应该了解以下几种节点类型：

- **ng_bridge**：虚拟交换机，用于将各个节点连接起来以实现通信。
- **ng_ether**：Netgraph 对现有以太网接口的表示，例如将虚拟设备桥接到物理网络时使用。
- **ng_eiface**：由 Netgraph 创建的虚拟以太网接口，适用于 jail 的网络接口。
- **ng_socket**：供用户进程（如 bhyve）与 Netgraph 交互的连接节点。

Netgraph 的强大之处在于这些组件之间的连接方式。例如，你可以创建虚拟交换机（ng_bridge），将物理接口（ng_ether）与虚拟接口（ng_eiface）连接起来，用于 jail 或虚拟机（VM）。你甚至可以多层嵌套桥接，在单个 FreeBSD 系统上优雅地建模复杂的网络拓扑结构。

## 使用 ngbuddy 简化 Netgraph

尽管 Netgraph 功能强大，但创建和管理节点的语法可能有些繁琐。它的精确性要求你明确指定某些事项，而这些在仅使用 ifconfig(8) 时是自动处理的。我编写了 ngbuddy（Netgraph Buddy）工具，用于自动化混合 VM 与 jail 环境下的配置，并确保这些更改在系统启动时持久生效。

在接下来的示例中，我们将从一个全新的 FreeBSD 14 安装开始，创建并使用一个“公共”的以太网连接桥接网络和一个 "私有" 的主机专用桥接网络，这类似于流行的虚拟机管理工具中的默认网络设置。我们还将使用 vm-bhyve（它在 2022 年 7 月添加了对 Netgraph 的支持）来进一步简化虚拟机管理。

## 设置基础虚拟网络

我们将模仿传统的虚拟网络设置方式，用 `if_bridge`、`tap` 和 `epair` 接口来为 jail 或 VM 创建虚拟网络。借助 Netgraph 与 ngbuddy，我们将改为配置 `ng_bridge` 节点，将 `public` 桥接连接到主机上与默认路由关联的接口。

首先，从 Ports 安装 ngbuddy：

```sh
# pkg install sysutils/ngbuddy
```

然后启用服务以创建一个基础配置：

```sh
# service ngbuddy enable
Adding default bridges.
ngbuddy_public_if: -> ix0
ngbuddy_private_if: -> nghost0
```

这将创建两个桥接：

- **public**：连接到物理接口（此示例中为 ix0）
- **private**：连接到一个新的虚拟接口（nghost0），用于主机专用网络（host-only networking）

如果你希望使用不同的桥接名称（我更偏好如 "lanbridge" 这类更具体的名字），可以编辑 `/etc/rc.conf` 中的 ngbuddy 配置行以符合你的偏好。

启动服务以创建 Netgraph 桥接节点：

```sh
# service ngbuddy start
Created 3 links.
```

此时，`public` 交换机已经就绪，应像连接到主机网络的任意设备一样工作。对于 `private` 主机专用网络，你很可能还需要配置其他网络组件，通常包括为 `nghost0` 接口设置 NAT 规则集和 DHCP 服务器。你可以像配置标准接口一样用 `ifconfig` 命令或向 `/etc/rc.conf` 添加对应条目来配置 `nghost0`。

## 配置 vm-bhyve

如果你使用的是 vm-bhyve 1.5.0 及更新版本，可以利用 ngbuddy 来配置虚拟交换机（注意不要与已有的 vm-bhyve 桥接名称冲突）：

```sh
# service ngbuddy vmconf
```

这将向你的 vm-bhyve 配置文件（定义于 `$VM_DIR/.config/system.conf`）添加以下内容：

```sh
switch_list="public private"
type_public="netgraph"
type_private="netgraph"
```

完成此配置后，vm-bhyve 就可以使用了。在配置你的 VM 时，只需在每个 VM 的配置中包含相应的交换机名称，例如：

```sh
network0_switch="public"
```

## 为 Jail 创建接口

在你的 jail 配置中添加或替换以下行，以便在 jail 启动时自动创建 `ng_eiface` 设备：

```ini
my_jail_name {
   if_name = "$name";
   $bridge = "public";
   vnet.interface = "$if_name";
   exec.prestart = "service ngbuddy jail $if_name $bridge";
   exec.prestop = "service ngbuddy unjail $if_name $name";
…
```

这个示例中使用 jail 名称变量 `$name` 作为接口名，以保证名称的唯一性与一致性。你也可以采用任何适合你环境的接口命名方案，只要每个接口的名称都是唯一的即可。

## 监控流量

使用 ngbuddy 可以轻松监控 Netgraph 节点的流量，它提供了一个简化版的 **ngctl … getstats** 命令视图，展示所有已检测到的接口的流量情况：

```sh
# service ngbuddy status
public
  vtnet0 (upper): RX 1.25 MB, TX 4.37 MB
  vtnet0 (lower): RX 4.37 MB, TX 1.25 MB
  jail1: RX 256.32 KB, TX 128.16 KB
private
  nghost0: RX 0B, TX 0B
  jail2: RX 0B, TX 0B
```

这提供了每个接口上流量的快速总览，对于故障排查与性能监控非常有用。你还可以使用 `service ngbuddy status` 或 `service ngbuddy vmname` 命令来识别 vm-bhyve 节点，否则它们在 `ngctl` 输出中将以 "unnamed" 的 `ng_socket` 设备形式出现。

## 可视化你的网络

我们可以使用 `ngctl dot` 命令快速生成主机虚拟网络的图形表示，该命令会输出如下的有向图文本：

```sh
graph netgraph {
    edge [ weight = 1.0 ];
    node [ shape = record, fontsize = 12 ] {
       "45" [ label = "{nghost0:|{eiface|[45]:}}" ];
       "49" [ label = "{private:|{bridge|[49]:}}" ];
       "4e" [ label = "{public:|{bridge|[4e]:}}" ];
       "52" [ label = "{jpub1:|{eiface|[52]:}}" ];
       "12" [ label = "{bge0:|{ether|[12]:}}" ];
       "13" [ label = "{bge0_42:|{ether|[13]:}}" ];
       "7b" [ label = "{vmpriv:|{socket|[7b]:}}" ];
    };
    subgraph cluster_disconnected {
       bgcolor = pink;
       "12";
    };
    node [ shape = octagon, fontsize = 10 ] {
       "45.ether" [ label = "ether" ];
    };
…
```

这段文本可以直接用 `dot(1)` 命令（来自 `graphics/graphviz` Port）转化为 SVG 或 PNG 图像。下面是一个简单 Netgraph 配置的清理后输出示例。

<img width="803" height="414" alt=" XC~VCKFVVK75F_$)8W(4C61 " src=" https://github.com/user-attachments/assets/669531f9-ef7f-4a77-bec6-5f321be7412a " />


虽然这个输出已经很不错，ngbuddy 还包含一个我编写的辅助脚本 `ngbuddy-mmd.awk`，可以将上述输出转换为更美观、更清晰的 mermaid-js 可视化图形。我们公司的文档系统支持 mermaid-js，因此借助一些 API 魔法，我总能在文档数据库中保持所有主机 Netgraph 拓扑图的实时更新。

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/bell-dev1-mmd-crop.png)

该图示在顶部标注了主机名，中间是 public 与 private 桥接的图形，下方是 jail（椭圆）和 VM（圆柱）。这种可视化在你需要管理带有不同租户网络和不同功能的服务器时尤其有用。

## 管理 MAC 地址冲突

无论我使用哪种虚拟网络技术，MAC 地址冲突总是让我头疼。即使是在只有不超过 5 台 FreeBSD 主机的小型家庭实验室中，也曾有过几个下午苦苦排查，最后才发现原来是某些 jail 争用了同一个 MAC 地址。为缓解此问题，并确保 MAC 地址的确定性生成（这对于虚拟机迁移非常关键），你可以在所有运行 ngbuddy 的主机上执行以下命令：

```sh
sysrc ngbuddy_set_mac=YES
sysrc ngbuddy_set_mac_prefix=02
```

此配置会让 ngbuddy 为它创建的接口分配唯一的 MAC 地址，使用你指定的前缀。前缀 `02` 表示这是一个本地管理地址（locally administered address），这非常适合在你完全控制的虚拟环境中使用。

更多技巧和功能，请参见 ngbuddy 的联机手册（man page）。

## 使用 jng：无需 Ports

`jng` 工具仍是最简单的方式，用于在 jail 中快速启用 Netgraph，并且它是 FreeBSD 系统自带的。该脚本会在需要时智能地创建桥接，非常适合多数 jail 主机的网络配置。首先，将 `jng` 脚本复制到你偏好的目录中：

```sh
install -m 755 /usr/share/examples/jails/jng /usr/local/sbin/jng
```

在 jail 的配置中，可以像使用 ngbuddy 那样调用 `jng`。注意，`jng` 需要你指定想要桥接的物理接口名称。

```ini
my_jail_name {
        $if_uplink = "em0";
        $if_name = "ng0_$name";
        vnet.interface = "$if_name";
        exec.prestart += "jng bridge $if_name $if_uplink";
        exec.poststop += "jng shutdown $if_name";
…
```

如果你有兴趣自己编写网络工作流，不妨阅读 `jng` 脚本，它拥有非常优秀的文档注释。

## 以传统方式实现 Netgraph

现在我们来看看为何配置 Netgraph 有时会让人感到烦恼，并探索前文提到的这些工具实际上做了哪些底层操作。FreeBSD 的虚拟网络配置通常通过 `ifconfig` 进行，这些对象可以直接用 `ifconfig` 查看。而 Netgraph 创建的某些对象只能通过 `ngctl` 和其他支持 Netgraph 的工具看到，这一点可能会让人有些困惑。

此外，不同于 ifconfig 操作的网络接口，Netgraph 中的 hook 之间可能存在复杂的连接关系，因此我们必须跟踪并记录每一个 hook。我们将手动创建前面提到的 `private` 与 `public` 的 `ng_bridge` 示例，并添加一个用于主机与虚拟 `private` 桥接通信的 `ng_eiface` 设备，构建一个主机专用网络。在这个例子中，我们将使用 `em0` 作为要桥接的物理接口。

```sh
# 加载 Netgraph 内核模块
kldload ng_ether ng_bridge

# 创建一个 ng_bridge，命名为 "private"
ngctl mkpeer ngeth0: bridge ether link0
ngctl name ngeth0:ether private

# 将 em0 上的流量共享给连接的 Netgraph 节点
ngctl msg em0: setpromisc 1

# 创建一个 ng_bridge 设备，命名为 "public"
ngctl mkpeer em0: bridge lower link1
ngctl name em0:lower public
ngctl connect em0: public: upper link2

# 关闭虚拟交换机无法正常处理的性能特性
ifconfig em0 -lro -tso

# 创建一个虚拟以太网设备 ngeth0（将会出现在 ifconfig 输出中）
ngctl mkpeer eiface ether ether
```

这里的关键在于，我们必须为每个对象创建并分配一个唯一的 `link#`，然后给它命名，才能有希望有效地跟踪管理它。接下来，我们再为一个 jail 创建一个新的 `ng_eiface` 设备。

```sh
# 创建一个新接口，然后打印它的名字
ngctl mkpeer public: eiface link3 ether
ngctl show -n public:link3 | cut -w -f3
```

在这个例子中，我们的新接口名称将是 `ngeth1`。你可以像配置普通接口一样用 `ifconfig` 来配置它。如果这个新接口被用作 VNET jail，它仍然可以通过与桥接的 Netgraph 连接正常通信。

虽然使用 Netgraph 手动配置的命令行操作可能并不比标准的 FreeBSD 虚拟网络多，但由于语法更复杂且需要跟踪 `link` 号，Netgraph 并不太适合手工频繁修改。

## 继续你的 Netgraph 之旅

Netgraph 是 FreeBSD 的一大隐藏宝石——一个强大的网络框架，一旦掌握，就能优雅地解决复杂的网络问题。虽然一开始可能感觉有些难以驾驭，但像 ngbuddy 和 jng 这类工具让它变得对各级管理员都更友好。

无论你是在管理小型家庭实验室还是复杂的生产环境，Netgraph 都能提供灵活强大的功能，帮助你构建精准符合需求的网络拓扑。它的性能优势、更清晰的配置方式以及可视化能力，都使其值得在虚拟化基础设施中考虑使用。我鼓励你深入探索这个强大的子系统，发现它如何简化你的虚拟网络搭建。

## 资源

- [Netgraph 在线手册](https://man.freebsd.org/cgi/man.cgi?query=netgraph&sektion=4)（man 4 netgraph）
- [ngbuddy GitHub 仓库](https://github.com/bellhyve/ngbuddy)（包含完整的 jail.conf 示例）
- [FreeBSD 手册：Jails 与容器](https://docs.freebsd.org/en/books/handbook/jails/)
- [FreeBSD 手册：虚拟化](https://docs.freebsd.org/en/books/handbook/virtualization/)
- [测试征集：Jail 与 bhyve 用户招募](https://callfortesting.org/)

--

**Daniel J. Bell** 是 Bell Tower 的创始人，Bell Tower 是一家位于纽约市的混合云服务提供商，专注于云效率、合规监管以及基于 ZFS 的高级存储。Daniel 积极参与开源项目并在多场会议上发表演讲，包括 2024 年 BSDCan 大会上关于 ZFS 和 Netgraph 的主题演讲。
