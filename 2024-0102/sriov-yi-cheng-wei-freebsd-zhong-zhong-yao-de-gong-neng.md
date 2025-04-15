# SR-IOV 已成为 FreeBSD 的重要功能

- 原文链接：[SR-IOV is a First Class FreeBSD Feature](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/sr-iov-is-a-first-class-freebsd-feature/)
- 作者：Mark McBride

### 如何在 FreeBSD 中使用支持 SR-IOV 的设备设置硬件驱动虚拟化

我最喜欢的硬件功能之一是被称为[单根输入/输出虚拟化（SR-IOV）](https://en.wikipedia.org/wiki/Single-root_input/output_virtualization)的技术。它让单一物理设备在操作系统中看起来如同多个类似的设备。FreeBSD 在实现 SR-IOV 功能方面的做法，是我[更倾向于在服务器上使用 FreeBSD 的原因之一](https://markmcb.com/freebsd/vs_linux/)。

## SR-IOV 网络概述

虚拟化是当你的网络设备需求超过服务器上物理网络端口数量时的理想解决方案。虽然有很多软件方式能实现这一点，但基于硬件的替代方案是 SR-IOV，它能让单个物理 PCIe 设备向操作系统呈现为多个设备。

使用 SR-IOV 有几个优势。同其他虚拟化方式相比，它提供了最佳的性能。如果你极为注重安全性，SR-IOV 更好地隔离了内存和它创建的虚拟化 PCI 设备。它还带来了非常整洁的配置，因为一切都作为 PCI 设备存在，也就是说，无需虚拟桥接、交换机等。

要使用 SR-IOV 网络，你需要支持 SR-IOV 的网络适配器和支持 SR-IOV 的主板。多年来，我使用了几块支持 SR-IOV 的网卡，例如 [Intel i350-T4V2 Ethernet Adapter](https://ark.intel.com/content/www/us/en/ark/products/84805/intel-ethernet-server-adapter-i350-t4v2.html)、[Mellanox ConnectX-4 Lx](https://www.nvidia.com/en-us/networking/ethernet/connectx-4-lx/) 和 [Chelsio T520-SO-CR Fiber Network Adapter](https://www.chelsio.com/nic/unified-wire-adapters/t520-so-cr/)。在本文中，我将使用 [Intel X710-DA2 Fiber Network Adapter](https://ark.intel.com/content/www/us/en/ark/products/83964/intel-ethernet-converged-network-adapter-x710da2.html) ([产品简介](https://www.intel.com/content/dam/www/public/us/en/documents/product-briefs/ethernet-x710-brief.pdf))，它被安装在 [FreeBSD 14.0-RELEASE 服务器](https://www.freebsd.org/releases/14.0R/announce/) 上。这是个不错的选择，因为它不需要特别的固件配置，并且 FreeBSD 内核默认内置了驱动支持。而且，它消耗的功率比许多其他方案要少，最多仅为 3.7 w。

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig1.jpg)

X710-DA2 拥有两个物理的 SFP+ 光纤端口。在 SR-IOV 术语中，这些端口对应于物理功能（PF）。如未启用 SR-IOV，这些 PF 就像任何网络适配器卡上的端口一样工作，将在 FreeBSD 中显示为两个网络接口。如启用 SR-IOV，每个 PF 都能够创建、配置和管理多个虚拟功能（VF）。每个 VF 都会在操作系统中显示为一个 PCIe 设备。

具体来说，对于 X710-DA2，它的 2 个 PF 最多可以为虚拟化 128 个 VF。从 FreeBSD 的角度来看，就好像你有一张带有 128 个端口的网卡。然后就可以把这些 VF 分配给 jail 和虚拟机，用于隔离的网络连接。

## 在 FreeBSD 中使用 SR-IOV

我们已经简要介绍了 SR-IOV 的概念性工作原理，但我发现通过实例更容易理解。让我们一步步走过，如何在 FreeBSD 中从头开始设置 SR-IOV。为此，我们将重点关注：

- [硬件安装](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/sr-iov-is-a-first-class-freebsd-feature/#hardware_installation)
- [硬件配置](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/sr-iov-is-a-first-class-freebsd-feature/#hardware_configuration)
- [FreeBSD 中的 SR-IOV 配置](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/sr-iov-is-a-first-class-freebsd-feature/#freebsd_configuration)
- [在 Jail 中使用 SR-IOV 网络 VF](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/sr-iov-is-a-first-class-freebsd-feature/#vf_jail)
- [在 Bhyve 虚拟机中使用 SR-IOV 网络 VF](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/sr-iov-is-a-first-class-freebsd-feature/#vf_bhyve)

## 硬件安装

支持 SR-IOV 的 X710-DA2 安装非常简单，但有一个主要的考虑因素。并非所有的 PCIe 插槽都是一样的。我强烈建议你在开始之前看看主板手册。在这个例子中，我将使用 [Supermicro X12STH-F 主板](https://www.supermicro.com/en/products/motherboard/x12sth-f)。其 [手册](https://www.supermicro.com/manuals/motherboard/X12/MNL-2367.pdf) 提供了两张非常有用的图表：

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig2.jpg)

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig3.jpg)

在第一张图中，我们看到 PCIe 插槽从左到右编号为 4、5 和 6。如果你仔细观察，会看到插槽 4 有前缀“PCH”，而 5 和 6 则有前缀“CPU”。第二张图则更详细地显示了这些插槽的连接方式。插槽 5 和 6 直连到 LGA1200 插座上的 CPU，而插槽 4 连接到[平台控制器集线器](https://en.wikipedia.org/wiki/Platform_Controller_Hub)。根据你设备中的具体组件，这可能会决定哪些插槽能够使 SR-IOV 按预期工作。直到后续配置 FreeBSD 时，你才会知道哪个插槽适合，一般来说，尤其是对于较旧的主板，CPU 插槽是个可靠的选择。如果后续步骤中发现 SR-IOV 无法正常工作，可以尝试换成 PCIe 插槽。主板文档有时并不详尽，所以试验和错误有时是最快速的方式，能帮助你找出哪个插槽能正常工作。

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig5.jpg)

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig6.jpg)

## 硬件配置

在未启用 SR-IOV 时，X710-DA2 会表现成一张不支持 SR-IOV 的网卡。启用 SR-IOV 很简单，但也容易被忘记，所以一定不要跳过这一重要步骤。

具体操作会根据主板的不同而有所变化，但大多数主板都有个 PCIe 配置参数的界面。找到该界面，启用 SR-IOV。与此同时，最好检查是否启用了你可能与 SR-IOV 一道使用的其他设置，例如 CPU 虚拟化。

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig7.jpg)

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/mcbride_fig8.jpg)

现在，我们可以启动 FreeBSD，并查看 [dmesg(8)](https://man.freebsd.org/dmesg)。以下是我系统中 `dmesg` 的一段输出。

```sh
ixl0: <Intel(R) Ethernet Controller X710 for 10GbE SFP+ - 2.3.3-k> mem 0x6000800000-0x6000ffffff,0x6001808000-0x600180ffff irq 16 at device 0.0 on pci1
ixl0: fw 9.120.73026 api 1.15 nvm 9.20 etid 8000d87f oem 1.269.0
ixl0: PF-ID[0]: VFs 64, MSI-X 129, VF MSI-X 5, QPs 768, I2C
ixl0: Using 1024 TX descriptors and 1024 RX descriptors
ixl0: Using 4 RX queues 4 TX queues
ixl0: Using MSI-X interrupts with 5 vectors
ixl0: Ethernet address: 3c:fd:fe:9c:9e:30
ixl0: Allocating 4 queues for PF LAN VSI; 4 queues active
ixl0: PCI Express Bus: Speed 2.5GT/s Width x8
ixl0: SR-IOV ready ixl0: netmap queues/slots: TX 4/1024, RX 4/1024
```

在第三行，我们看到了一些 SR-IOV 的信息。“PF-ID[0]”与 ixl0 相关，并且这个 PF 能支持 64 个 VF。而在第十行，我们可以看到明确确认：这个 PCIe 设备已是“SR-IOV 就绪”（SR-IOV ready）。之所以名称是“ixl”，是因为这张网卡使用了 [ixl(4)](https://man.freebsd.org/cgi/man.cgi?query=ixl) Intel Ethernet 700 系列驱动。

除了检查硬件状态外，无需其他配置。某些网卡（比如前面提到的 Mellanox）需要你配置网卡的固件，而其他网卡（比如前面提到的 Chelsio）则需要在 `/boot/loader.conf` 中进行驱动配置。但 X710-DA2 并不需要这些配置，尽管你可能需要检查并更新卡的固件版本（如有必要）。

至此，我们可以从硬件设置转到 FreeBSD 配置的部分。

## 在 FreeBSD 中配置 SR-IOV

### 使用 PF（物理功能）

SR-IOV 的一个优点是，无论是否用 PF 创建 VF，你仍可以将 PF 用作网络接口。我在我的 `/etc/rc.conf` 中添加了以下内容，并为 PF 分配了一个 IP 地址，用于主机的连接：

```sh
ifconfig_ixl0=”inet 10.0.1.201 netmask 255.255.255.0” defaultrouter=”10.0.1.1”
```

现在，当我启动系统时，我可以预期 ixl0 设备会有一个 IP 地址，我可以用它来连接到系统——无论 SR-IOV 启用与否。

### 指示 PF 创建 VF

在 FreeBSD 中，是通过 [iovctl(8)](https://man.freebsd.org/cgi/man.cgi?query=iovctl) 实现 PF 和 VF 的管理的，`iovctl` 是操作系统的基础工具之一。要创建 VF，我们需要在 `/etc/iov/` 目录下创建一个文件，指定我们需要的配置。我们将采用一个简单的策略：创建一个 VF 分配给 jail，另一个 VF 分配给 bhyve 虚拟机。可以参考手册页 [iovctl.conf(5)](https://man.freebsd.org/iovctl.conf) 了解最重要的参数。

```INI
OPTIONS
    以下参数为所有 PF 驱动程序所接受：
    device (string)
    该参数指定 PF 设备的名称。此参数是必需的。
    num_vfs (uint16_t)
    该参数指定要创建的 VF 子设备的数量。此参数不能为空。该参数的最大值由设备决定。
```

我喜欢将 `num_vfs` 设置为实际需要的数量。我们本可以将其设置为最大值，但我发现这样会使查看 `ifconfig` 等命令的输出变得更加困难。

另外，由于不同的网卡有不同的驱动程序，每款驱动程序都有一些可以根据硬件能力设置的参数。手册页面 [ixl(4)](https://man.freebsd.org/ixl) 列出了多个可选参数。

```sh
IOVCTL OPTIONS
    驱动程序支持使用 iovctl(8) 创建 VF 时的其他可选参数：

    mac-addr (unicast-mac)
    设置 VF 将使用的以太网 MAC 地址。若未指定，则 VF 将使用随机生成的 MAC 地址。
```

或者，你也可以使用命令 `iovctl`，快速查看 PF 及其 VF 支持的参数，以及它们的默认值。

```sh
(host) $ sudo iovctl -S -d ixl0
以下配置参数可以在 PF 上进行配置：
    num_vfs : uint16_t (必需)
    device : string (必需)

以下配置参数可以在 VF 上进行配置：
    passthrough : bool (默认 = false)
    mac-addr : unicast-mac (可选)
    mac-anti-spoof : bool (默认 = true)
    allow-set-mac : bool (默认 = false)
    allow-promisc : bool (默认 = false)
    num-queues : uint16_t (默认 = 4)
```

我们将使用参数 `mac-addr` 为每个 VF 设置特定的 MAC 地址。在此例中，把 MAC 地址设置为随意生成的，但我将演示如何在配置文件中设置 PF 参数、默认的 VF 参数以及特定于单个 VF 的参数。

```ini
PF {
       device : “ixl0”
       num_vfs : 2
}

DEFAULT {
       allow-set-mac : true;
}

VF-0 {
       mac-addr : “aa:88:44:00:02:00”;
}

VF-1 {
       mac-addr : “aa:88:44:00:02:01”;
}
```

这将用 ixl0 创建两个 VF。在默认情况下，每个 VF 都可以设置自己的 MAC 地址。每个 VF 会被分配一个初始的 MAC 地址（该地址可以通过之前的默认设置来覆盖）。

在使配置生效之前，让我们先查看当前的环境。我们会找到两个 ixl PCI 设备和两个 ixl 网络接口。

```sh
(host) $ ifconfig -l
ixl0 ixl1 lo0

(host) $ pciconf -lv | grep -e ixl -e iavf -A4
ixl0@pci0:1:0:0:        class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086
device=0x1572 subvendor=0x8086 subdevice=0x0007
    vendor     = 'Intel Corporation'
    device     = 'Ethernet Controller X710 for 10GbE SFP+'
    class      = network
    subclass   = ethernet
ixl1@pci0:1:0:1:        class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086
device=0x1572 subvendor=0x8086 subdevice=0x0000
    vendor     = 'Intel Corporation
    device     = 'Ethernet Controller X710 for 10GbE SFP+'
    class      = network
    subclass   = ethernet
```

要使 `/etc/iov/ixl0.conf` 配置文件生效，我们使用 [iovctl(8)](https://man.freebsd.org/cgi/man.cgi?query=iovctl)。

```sh
(host) $ sudo iovctl -C -f /etc/iov/ixl0.conf
```

若你修改了配置文件，记得先删除再重新创建 VF。

```sh
(host) $ sudo iovctl -D -f /etc/iov/ixl0.conf
(host) $ sudo iovctl -C -f /etc/iov/ixl0.conf
```

要检查是否成功创建了 VF，我们可以再次运行之前的命令 `ifconfig` 和 `pciconf`。

```sh
(host) $ ifconfig -l
ixl0 ixl1 lo0 iavf0 iavf1

(host) $ pciconf -lv | grep -e ixl -e iavf -A4
ixl0@pci0:1:0:0:        class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x1572 subvendor=0x8086 subdevice=0x0007
    vendor     = 'Intel Corporation'
    device     = 'Ethernet Controller X710 for 10GbE SFP+'
    class      = network
    subclass   = ethernet
ixl1@pci0:1:0:1:        class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x1572 subvendor=0x8086 subdevice=0x0000
    vendor     = 'Intel Corporation'
    device     = 'Ethernet Controller X710 for 10GbE SFP+'
    class      = network
    subclass   = ethernet
--
iavf0@pci0:1:0:16:        class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x154c subvendor=0x8086 subdevice=0x0000
    vendor     = 'Intel Corporation'
    device     = 'Ethernet Virtual Function 700 Series'
    class      = network
    subclass   = ethernet
iavf1@pci0:1:0:17:        class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x154c subvendor=0x8086 subdevice=0x0000
    vendor     = 'Intel Corporation'
    device     = 'Ethernet Virtual Function 700 Series'
    class      = network
    subclass   = ethernet
```

瞧！我们已经创建了崭新的 VF 设备。在 `pciconf` 的输出中，我们仍然可以看到原来的 `ixl` 设备，但现在有了两个 [iavf](https://man.freebsd.org/iavf) 设备。手册页面 `iavf(4)` 告诉我们，这些是 Intel Adaptive Virtual Functions 驱动程序驱动的。

除了看到新的 PCI 设备外，`ifconfig` 也确认它们已经被识别为网络接口。对于大多数网络设备的常见功能，你可能区分不开 PF 和 VF。想要了解更详细的区别，可以查看驱动文档及使用 `pciconf` 的 `-c` 功能参数，例如 `pciconf -lc iavf`。

为了确保在重启后配置能够保持有效，修改 `/etc/rc.conf` 文件：

```sh
# 配置 SR-IOV
iovctl_files=”/etc/iov/ixl0.conf”
```

现在我们有了两个准备好的 VF，可以投入使用了！

## 在 Jail 中使用 SR-IOV 网络 VF

本节假设你对 FreeBSD Jail 有基本的了解。因此，从头开始设置 Jail 的流程不在本文范围内。有关如何设置 Jail 的更多信息，请参阅 FreeBSD 手册中的章节 [Jail 与容器](https://docs.freebsd.org/en/books/handbook/jails/) 。

我不使用什么 Jail 管理软件，而是靠基本操作系统自带的工具。如果你使用过像 [Bastille](https://bastillebsd.org/) 这样的管理工具，配置文件的位置和方式可能会有所不同，但原理是一样的。在此例中，我们使用一个名为“desk”的 Jail。

```sh
exec.start += “/bin/sh /etc/rc”;
exec.stop = “/bin/sh /etc/rc.shutdown”;
exec.clean;
mount.devfs;

desk {
        host.hostname = “desk”;
        path = “/mnt/apps/jails/desk”;
        vnet;
        vnet.interface = “iavf0”;
        devfs_ruleset=”5”;
        allow.raw_sockets;
}
```

就这样！这个 Jail 现在能通过 [vnet(9)](https://man.freebsd.org/vnet) 访问自己专用的 VF 网络设备。我将修改这个 Jail 的 `/etc/rc.conf` 文件，启用网络配置：

```sh
ifconfig_iavf0=”inet 10.0.1.231 netmask 255.255.255.0”
defaultrouter=”10.0.1.1”
```

现在，让我们启动 Jail 并检查其是否正常工作。

```sh
(host) $ sudo service jail start desk
Starting jails: desk.

(host) $ sudo jexec desk ifconfig iavf0
iavf0: flags=1008843 metric 0 mtu 1500
        options=4e507bb TSO6,LRO,VLAN_HWFILTER,VLAN_HWTSO,RXCSUM_IPV6,TXCSUM_IPV6,HWSTATS,MEXTPG>
        ether aa:88:44:00:02:00
10.0.1.231 netmask 0xffffff00 broadcast 10.0.1.255
        media: Ethernet autoselect (10Gbase-SR )
        status: active
        nd6 options=29

(host) $ sudo jexec desk ping 9.9.9.9
PING 9.9.9.9 (9.9.9.9): 56 data bytes
64 bytes from 9.9.9.9: icmp_seq=0 ttl=58 time=19.375 ms
64 bytes from 9.9.9.9: icmp_seq=1 ttl=58 time=19.809 ms
64 bytes from 9.9.9.9: icmp_seq=2 ttl=58 time=19.963 ms
```

正如预期的那样，我们在 Jail 中看到了网络接口 `iavf0`，并且它似乎正常工作。但是物理机操作系统中的设备呢？它还在吗？让我们看一下。

```sh
(host) $ ifconfig -l
ixl0 ixl1 lo0 iavf1
```

## 在 Bhyve 虚拟机中使用 SR-IOV 网络 VF

通过虚拟机 [bhyve(8)](https://man.freebsd.org/bhyve)，你也能实现差不多的效果，虽然方法稍有不同。对于 Jail，我们可以在运行时分配和释放 VF。而在 bhyve 中，这必须在启动时完成分配和释放 VF，并且需要调整 SR-IOV 配置。首先，在做任何更改之前，让我们再看一下 `pciconf`。

```sh
(host) $ pciconf -l | grep iavf
iavf0@pci0:1:0:16:      class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x154c
subvendor=0x8086 subdevice=0x0000
iavf1@pci0:1:0:17:      class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x154c
subvendor=0x8086 subdevice=0x0000
```

看看未使用的 VF，`iavf1`。第一列可以理解为：“有一个使用 iavf 驱动的 PCI0 设备，ID 为 1，PCI 选择符为总线 1，插槽 0，功能 17”。虽然现在你还不需要它们，但这三个数字最终会告诉 bhyve 我们需要使用哪个设备。在此之前，我们需要确保在启动时加载 [vmm(4)](https://man.freebsd.org/vmm) 以启用 bhyve，再调整我们的第二个 VF 以便将其传给 bhyve。

```sh
## 启动虚拟机监控程序（bhyve 的内核部分）
vmm_load="YES"

# 另一种传递 VF 或任何 PCI 设备的方法是
# 在 /boot/loader.conf 中指定设备。我在此列出供参考。
# 我们将使用 iovctl 配置，因为它将所有内容集中在一个地方。
# pptdevs="1/0/17"
```

要将 VF 保留为 bhyve 的 PCI 直通设备，我们使用 `iovctl` 的 `passthrough` 参数。

```sh
    passthrough (boolean)
        该参数控制是否将 VF 保留为 bhyve(8) 虚拟机的 PCI 直通设备。如果设置为 true，VF 将被保留为 PCI 直通设备，并且无法从物理机操作系统访问。此参数的默认值为 false。
```

```ini
PF {
        device : “ixl0”
        num_vfs : 2
}

DEFAULT {
        allow-set-mac : true;
}

VF-0 {
        mac-addr : “aa:88:44:00:02:00”;
}

VF-1 {
        mac-addr : “aa:88:44:00:02:01”;
        passthrough : true;
}
```

当我们下次启动系统时，会发现 `iavf1` 不见了，因为 `iavf` 驱动程序不会被分配给我们的第二个 VF。相反，它会被标记为“ppt”（PCI 直通），并且只有 bhyve 才能使用它。

在做了这些调整后，重新启动系统。

你会立刻发现，`dmesg` 输出有了变化。这次没有提到 `iavf1`。记得我们在 `pciconf` 中看到的选择符 `1:0:17` 吗？在这里我们以稍微不同的格式看到了它。

```sh
ppt0 at device 0.17 on pci1
```

`pciconf` 确认该设备已被保留用于直通。

```sh
(host) $ pciconf -l | grep iavf
iavf0@pci0:1:0:16:      class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x154c subvendor=0x8086 subdevice=0x0000

(host) $ pciconf -l | grep ppt
ppt0@pci0:1:0:17:      class=0x020000 rev=0x01 hdr=0x00 vendor=0x8086 device=0x154c subvendor=0x8086 subdevice=0x0000
```

接下来，所有操作都在 bhyve 中完成。本文假定你知道如何启动运行 bhyve 虚拟机。我使用工具 [vm-bhyve](https://man.freebsd.org/cgi/man.cgi?query=vm) 来方便地管理虚拟机（但如果你不使用 vm-bhyve，请参考本节末尾的原始 bhyve 参数）。我将把直通的 VF 添加到名为 `debian-test` 的 Debian 虚拟机中。我们只需要在配置中定义要直通的设备，并移除与虚拟网络相关的任何配置行。

```sh
loader="grub"
cpu=1
memory=4G
disk0_type="virtio-blk"
disk0_name="disk0.img"
uuid="b997a425-80d3-11ee-a522-00074336bc80"

# 为网络直通 VF
passthru0="1/0/17"

# 不需要网络配置行，因为有了 VF
# network0_type="virtio-net"
# network0_switch="public"
# network0_mac="58:9c:fc:0c:fd:b7"
```

现在我们只需启动我们的 bhyve 虚拟机。

```sh
(host) $ sudo vm start debian-test
Starting debian-test
  * found guest in /mnt/apps/bhyve/debian-test
  * booting...

(host) $ sudo vm console debian-test
Connected

debian-test login: root
Password:
Linux debian-test 6.1.0-16-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.67-1 (2023-12-12) x86_64

root@debian-test:~# lspci | grep -i intel
00:05.0 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series
(rev 01)

root@debian-test:~# ip addr
2: enp0s5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether aa:88:44:00:02:01 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.99/24 brd 10.0.1.255 scope global dynamic enp0s5
       valid_lft 7186sec preferred_lft 7186sec
    inet6 fdd5:c1fa:4193:245:a888:44ff:fe00:201/64 scope global dynamic mngtmpaddr
       valid_lft 1795sec preferred_lft 1795sec
    inet6 fe80::a888:44ff:fe00:201/64 scope link
       valid_lft forever preferred_lft forever

root@debian-test:~# ping 9.9.9.9
PING 9.9.9.9 (9.9.9.9) 56(84) bytes of data.
64 bytes from 9.9.9.9: icmp_seq=1 ttl=58 time=20.6 ms
64 bytes from 9.9.9.9: icmp_seq=2 ttl=58 time=19.8 ms
```

**成功！** 现在，我们在 bhyve 虚拟机中为网络配置了一个 SR-IOV VF 设备。如果你是极简主义者，不想使用 `vm-bhyve`，可通过 `vm` 命令查看 `vm-bhyve.log` 文件，其中会列出传递给 `grub-bhyve` 和 `bhyve` 的参数，用来启动虚拟机。

```sh
create file /mnt/apps/bhyve/debian-test/device.map
      -> (hd0) /mnt/apps/bhyve/debian-test/disk0.img
grub-bhyve -c /dev/nmdm-debian-test.1A -S \
      -m /mnt/apps/bhyve/debian-test/device.map \
      -M 4G -r hd0,1 debian-test
bhyve -c 1 -m 4G -AHP
      -U b997a425-80d3-11ee-a522-00074336bc80 -u -S \
      -s 0,hostbridge -s 31,lpc \
      -s 4:0,virtio-blk,/mnt/apps/bhyve/debian-test/disk0.img \
      -s 5:0,passthru,1/0/17
```

### bhyve PCI 直通是在开发中的功能

虽然在 Jail 中使用 VF 配合 vnet 非常稳定，但在 14.0-RELEASE 版本的 bhyve PCI 直通功能仍在开发中。仅使用 bhyve 配合直通功能表现良好。然而，我发现如果同时在使用 VF 和 Jail 时，某些硬件组合和设备数量可能会导致意外的行为。随着每次版本发布，都会有改进。如果你遇到极端情况，请务必[提交 bug](https://bugs.freebsd.org/)。

### FreeBSD SR-IOV 总结

要在 FreeBSD 中使用启用 SR-IOV 的虚拟 PCIe 设备，我们需要：

- 安装一张支持 SR-IOV 的网络卡到支持 SR-IOV 的主板上
- 确保主板的 SR-IOV 功能已启用
- 创建 `/etc/iov/ixl0.conf` 并指定我们想要的 VF 个数
- 在 `/etc/rc.conf` 中引用 `/etc/iov/ixl0.conf` 以便在重启时保留配置

就这么简单！

为了演示它的工作原理，我们使用 vnet 将一个 VF 分配给了一个 Jail。我们还在启动时预先为 bhyve 虚拟机分配了另一个 VF。在这两种情况下，我们只需要在各自的 Jail/虚拟机配置文件中添加几行配置。

接下来的部分将对比 FreeBSD 和 Linux 中 SR-IOV 的使用方式，让你了解两者的差异。

## 在 Linux 中使用 SR-IOV

SR-IOV 在 Linux 中工作得非常好。配置完成后，你可能发现不了 FreeBSD 和 Linux 之间明显的差异。然而，配置过程可能需要一些时间。

最大的区别在于，Linux 中没有像 FreeBSD 的 `iovctl` 那样的标准工具来配置 SR-IOV。实现一个工作配置有多种方式，但这些方法不太明晰。我将重点介绍如何使用 `udev` 配置 Mellanox 卡的 PF 和 VF。

`udev` 是一款功能强大的工具，能干很多活。它能在启动时启用 SR-IOV 设备。这个工具本身非常出色，但困难在于如何为它提供正确的数据。获取所需的属性可能需要在网上搜索一番，但只要你找到了这些属性，编写 `udev` 规则就非常简单。

```sh
# 不要探测将用于虚拟机的 VF
KERNEL==”0000:05:00.0”, SUBSYSTEM==”pci”, ATTRS{vendor}==”0x15b3”, ATTRS{device}==”0x1015”,
ATTR{sriov_drivers_autoprobe}=”0”, ATTR{sriov_numvfs}=”4”

# 探测将用于 LXD 的 VF
KERNEL==”0000:05:00.1”, SUBSYSTEM==”pci”, ATTRS{vendor}==”0x15b3”, ATTRS{device}==”0x1015”,
ATTR{sriov_drivers_autoprobe}=”1”, ATTR{sriov_numvfs}=”16”
```

这段规则的意思是：“匹配 PCI 设备 `0000:05:00.0`，其供应商 ID 为 `0x15b3`，设备 ID 为 `0x1015`，并且对于这个设备不要自动分配驱动程序，并创建 4 个 VF”（即为直通保留）。第二条规则类似，但针对不同的 PF，它会分配驱动程序并创建 16 个 VF（即为容器分配做好准备）。

根据所使用的卡和具体的 Linux 发行版，可能并非所有的属性都适用。例如，如果你使用的是 Fedora，你可能需要添加 `ENV{NM_UNMANAGED}="1"`，以避免 NetworkManager 在启动时接管 VF。

与 `pciconf` 类似，`lspci` 能帮助我们获取匹配规则所需的大部分信息，如 PCI 地址、供应商和设备 ID。在这个系统中，我们看到的是 Mellanox ConnectX-4 Lx 卡。

```sh
lspci -nn | grep ConnectX
05:00.0 Ethernet controller [0200]: Mellanox Technologies MT27710 Family [ConnectX-4 Lx] [15b3:1015]
05:00.1 Ethernet controller [0200]: Mellanox Technologies MT27710 Family [ConnectX-4 Lx] [15b3:1015]
```

可以在目录 `/sys/bus/pci/devices/0000:05:00.*/` 下查看通过 `udev` 设置的属性，此外还有很多其他属性。列出该目录的内容是查找需要传给 `udev` 的信息的好方法。

```sh
(linux) $ ls -AC /sys/bus/pci/devices/0000:05:00.0/
aer_dev_correctable       device            irq               net           resource0                subsystem
aer_dev_fatal             dma_mask_bits     link              numa_node     resource0_wc             subsystem_device
aer_dev_nonfatal          driver            local_cpulist     pools         revision                 subsystem_vendor
ari_enabled               driver_override   local_cpus        power         rom                      uevent
broken_parity_status      enable            max_link_speed    power_state   sriov_drivers_autoprobe  vendor
class                     firmware_node     max_link_width    ptp           sriov_numvfs             virtfn0
config                    hwmon             mlx5_core.eth.0   remove        sriov_offset             virtfn1
consistent_dma_mask_bits  infiniband        mlx5_core.rdma.0  rescan        sriov_stride             virtfn2
current_link_speed        infiniband_verbs  modalias         reset         sriov_totalvfs           virtfn3
current_link_width        iommu             msi_bus           reset_method  sriov_vf_device          vpd
d3cold_allowed            iommu_group       msi_irqs          resource      sriov_vf_total_msix
```

在这个列出的目录中，我们看到 `sriov_drivers_autoprobe` 和 `sriov_numvfs`，这是我们在启动时需要设置的属性。其他属性的作用是什么？你可能需要通过搜索引擎来获取答案。

通过 `udev`，我们已经完成了两大步骤中的第一步。它有效地“解放”了硬件的 SR-IOV 能力。接下来，我们需要为网络使用配置 SR-IOV，这是第二步。根据我们使用的网络管理方式，这个过程有极大的差异。例如，如果你使用的是 `systemd-networkd`，可以像这样进行配置：

```sh
#/etc/systemd/network/21-wired-sriov-p1.network
[Match]
Name=enp5s0f1np1

[SR-IOV]
VirtualFunction=0
Trust=true

[SR-IOV]
VirtualFunction=1
Trust=true
```

幸运的是，对于 `systemd-networkd`，文档并不难找，你能找到大部分需要的信息。在完成这些配置后，我们重启服务，VF 就可以使用了。

但并非所有文档都这么简洁，除了网络软件本身，像 AppArmor 和 SELinux 等安全防护工具可能会在运行时对 SR-IOV 产生阻碍，这些阻碍是“按预期”运行的，但会让系统表现得像是出现了故障。

以我最近在 Fedora 39 上运行 LXD 容器为例，我发现需要在 `udev` 中设置 `ENV{NM_UNMANAGED}="1"`，这样可以让 LXD 管理我的 VF。一切运行正常——直到我重新启动容器。突然间，LXD 开始报错没有 VF。

原来，尽管 `udev` 规则在启动时阻止了 NetworkManager 管理 VF，但当容器重启时，NetworkManager 仍会接管它们。我发现 VF 设备的名称在容器重启后发生了变化。例如，原本是 `enp5s0f0np0`，在容器重启后变成了类似 `physZqHm0g` 这样的名字。

最终，我找到了一种方法来阻止 NetworkManager 执行这个操作。以下是阻止 LXD 和 NetworkManager 争抢 VF 的关键配置文件，供参考：

```sh
[keyfile]
unmanaged-devices=interface-name:enp5s0f1*,interface-name:phys*
```

这只是一个例子。以为一切都配置好，结果几天后才发现系统出现问题的情况并不罕见。一般来说，所有的烦恼都有一个根本原因：Linux 生态系统中并没有一个现成/正在兴起的标准配置 SR-IOV 的方式。虽然设置过程不够直观，但只要你克服了这些难题，Linux 中的 SR-IOV 网络配置就能正常工作。

## 结论

SR-IOV 是 FreeBSD 中的一等公民。本文中提到的所有内容都可以通过操作系统提供的手册页找到。你可以通过简单的 [apropos(1)](https://man.freebsd.org/apropos) 搜索来开始。

```sh
(host) $ apropos “SR-IOV”
iovctl(8) - PCI SR-IOV configuration utility
```

`iovctl` 手册会帮你入门，驱动程序的手册页会为你提供硬件的详细信息。当事情变得显而易见并且易于查找时，系统管理就不再是负担。

Linux 发行版同样可以完成这项工作，但在 SR-IOV 的一致性和系统内文档方面存在不足。虽然我在很多方面需要 Linux，但我确实很欣赏 FreeBSD 配置的有组织性。它让我能够轻松地回到一年未曾触碰的系统，并快速通晓我所做的改动。相比之下，我更倾向于这种方式，而不是详细记录并依赖于那些不太明确的 URL 或论坛评论。

正如所有事情一样，做出明智的选择，选择最符合自己需求的方式。

---

**Mark McBride** 在美国华盛顿州西雅图从事 CAR-T 细胞疗法工作，专注于在个性化医疗的新领域中整合供应链、制造和患者运营解决方案。在闲暇时间，他喜欢过度工程化自己的车库实验室，为西雅图本地的运动队加油。他以 @markmcb 的身份活跃在 Libera IRC 服务器的 #freebsd 频道中，还可通过个人网站 [markmcb.com](https://www.markmcb.com/) 与他联系。
