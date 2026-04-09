# 面向 Linux 和 Windows 用户的 bhyve

- 原文链接：[bhyve for the Linux and Windows Users](https://freebsdfoundation.org/our-work/journal/browser-based-edition/virtualization-2/bhyve-for-the-linux-and-windows-users/)
- 作者：**Jason Tubnor**

FreeBSD bhyve 虚拟化程序在 2011 年 5 月由 Neel Natu 和 Peter Grehan 向全世界宣布，随后由 NetApp 赠送给 FreeBSD。这为 FreeBSD 提供了与 Linux KVM 虚拟化程序竞争的机会。然而，bhvy 还有其他好处，它不仅小巧且稳健，性能优异，并且高度依赖 CPU 指令集，而不是通过解释执行。

最初的实现仅适用于 FreeBSD 客户机，直到过了一段时间，我们才看到 bhyve 能够运行其他操作系统。

首先是 Linux，然后有了一种方法来重新打包 Windows 8 或 Windows Server 2012 以使其安装。这对于普通用户来说管理起来过于复杂，直到 bhyve 引入 UEFI 启动特性，事情才真正起飞。

UEFI 启动是 bhyve 所等待的杀手级特性。它使得可以在 FreeBSD bhyve 上安装并运行各种操作系统。当 FreeBSD 11 发布时，我们终于拥有了与其他操作系统相媲美的虚拟化组件。

虽然 UEFI 启动是 bhyve 的杀手级特性，但 bhyve 的杀手级应用是 Windows Server 2016。这是一个转折点，企业能够在没有修改的情况下运行 bhyve 和 Windows，获得一个可靠的企业级虚拟化程序，以稳定的方式运行商业工作负载。

突然间，企业能够广泛部署设备，使用一个具有 2-Clause BSD 许可证的解决方案，并能够通过硬件或软件调整虚拟化程序，以解决其问题。

然而，问题仍然存在，因为 Windows 需要安装多个驱动程序，无论是在镜像中还是在安装后，以避免性能问题。在 2018 年 7 月，通过实现 PCI-NVMe 存储仿真，这个问题得到了部分解决，最终让 bhyve 在一般工作负载的存储性能上超越了 KVM。

如今，Windows 在 bhyve 上运行仍然至少需要 RedHat 提供的 VirtIO-net 驱动程序，以确保网络传输可靠且超过 1Gb/s。RedHat 提供的 Windows MSI 包中还有其他驱动程序，建议在生产环境实现前加载这些驱动程序。对于 Linux，大多数发行版都包括所有相关驱动程序，包括本文中使用的 AlmaLinux。对于 Linux 安装，推荐使用 NVMe 仿真存储作为后端，然而，如果你计划将 Linux 工作负载在 bhyve 和 KVM 之间迁移，建议将客户机设置为使用 VirtIO-blk 存储。

以下配置适用于典型的类型 2 虚拟化程序配置中的标准 FreeBSD 工作站，或用于仅托管客户机工作负载并与客户机相关联的存储和网络的专用 FreeBSD 服务器，适用于类型 1 虚拟化程序。

## 准备工作

通常，过去十年的所有现代处理器都适合用于 bhyve 虚拟化。

确保你的硬件配置启用了虚拟化技术，并且启用了 VT-d 支持。PCI 直通的使用超出了本文的范围，但建议启用 VT-d，以便在需要时可以使用。在你的计算机 BIOS/固件中配置好之后，你可以通过查看 CPU 的 Features2 中是否包含 POPCNT 来检查 FreeBSD 是否能够识别到。

```sh
# dmesg | grep Features2
  Features2=0x7ffafbff<SSE3,PCLMULQDQ,DTES64,MON,DS_CPL,VMX,SMX,EST,TM2,SSSE3,SDBG,FMA,
  CX16,xTPR,PDCM,PCID,SSE4.1,SSE4.2,x2APIC,MOVBE,POPCNT,TSCDLT,AESNI,XSAVE,OSXSAVE,AVX,
  F16C,RDRAND>
```

现在我们已经确认 CPU 准备就绪，我们需要安装一些软件包，以便轻松创建和管理客户操作系统：

```sh
pkg install openntpd vm-bhyve bhyve-firmware
```

简而言之，OpenNTPD 是来自 OpenBSD 项目的一个简单时间守护进程。它可以防止主机时间偏移。当虚拟化程序由于客户工作负载而面临极大压力时，这可能导致系统时间迅速失去同步。OpenNTPD 通过确保你的上游时间源通过 HTTPS 协议报告正确的时间，来保持时间的准确性。bhyve-firmware 是个元包，用于从软件包中加载 bhyve 支持的最新 EDK2 固件。最后，vm-bhyve 是用于 bhyve 的管理系统，使用 shell 编写，避免了复杂的依赖关系。

使用 vm-bhyve 为机器引导并做好准备非常简单，但需要注意一些 ZFS 选项，以确保客户在底层存储上能够保持良好的性能，特别是对于一般工作负载：

```sh
# zfs create -o mountpoint=/vm -o recordsize=64k zroot/vm
# cat <<EOF >> /etc/rc.conf
vm_enable=”YES”
vm_dir=”zfs:zroot/vm”
vm_list=””
vm_delay=”30”
EOF
# vm init
```

在我们深入讨论之前，我们应该先下载稍后将使用的 ISO 文件，以便 vm-bhyve 安装程序可以使用它们。要将 ISO 下载到 vm-bhyve ISO 存储中，可以使用 `vm iso` 命令：

```sh
# vm iso https://files.bsd.engineer/Windows11-bhyve.iso
```

(sha256 – 46c6e0128d1123d5c682dfc698670e43081f6b48fcb230681512edda216d3325)

```sh
# vm iso https://repo.almalinux.org/almalinux/9.5/isos/x86_64/AlmaLinux-9.5-x86_64-dvd.iso
```

(sha256 – 3947accd140a2a1833b1ef2c811f8c0d48cd27624cad343992f86cfabd2474c9)

这些文件将被下载到目录 `/vm/.iso` 中。注意：AlmaLinux ISO 文件直接从项目官网下载，校验和可以上游验证。而 Windows11-bhyve ISO 文件是从 Microsoft 下载的，并经过修改，确保它能够安装在 Microsoft 认为是“未支持”硬件的设备上，并已提供用于协助本文。因此，该 ISO 文件仅应在实验室环境中使用。它已移除 CPU 和 TPM 要求，并且不再需要创建 Microsoft 帐户。

## 网络配置

默认情况下，vm-bhyve 使用桥接将系统的物理接口与分配给每个虚拟机的 tap 接口连接。在将物理接口添加到桥接时，某些功能，如 TCP 分段卸载（TSO）和大接收卸载（LRO），不会被禁用，但需要禁用这些功能，以确保虚拟机的网络功能正常工作。如果主机有 em(4) 接口，可以通过以下命令禁用它：

```
# ifconfig em0 -tso -lro
```

为了避免每次重启后都需要禁用这些功能，可以将其添加到系统的 `/etc/rc.conf` 文件中：

```
# ifconfig_em0_ipv6="inet6 2403:5812:73e6:3::9:0 prefixlen 64 -tso -lro"
```

根据使用的网卡不同，以上步骤可能不是每种情况都需要，但如果你遇到虚拟机网络性能问题，这就是可能的原因。

要配置虚拟交换机（桥接），可以使用 `switch` vm 子命令：

```sh
# vm switch create public
# vm switch add public em0
```

这将创建一个名为 `public` 的虚拟交换机，并将 em0 物理接口添加到该虚拟交换机中。

## 模板

模板用于帮助设置虚拟机配置，确保虚拟机使用正确的虚拟硬件和其他必要的设置。使用 root 用户，添加以下内容到模板仓库：

```sh
# cat <<EOF > /vm/.templates/linux-uefi.conf
loader=”uefi”
graphics=”yes”
cpu=2
memory=1G
disk0_type=”virtio-blk”
disk0_name=”disk0.img”
disk0_dev=”file”
graphics_listen=”[::]”
graphics_res=”1024x768”
xhci_mouse=”yes”
utctime=”yes”
virt_random=”yes”
EOF

# cat <<EOF > /vm/.templates/windows-uefi.conf
loader=”uefi”
graphics=”yes”
cpu=2
memory=4G
disk0_type=”nvme”
disk0_name=”disk0.img”
disk0_dev=”file”
graphics_listen=”[::]”
graphics_res=”1024x768”
xhci_mouse=”yes”
utctime=”no”
virt_random=”yes”
EOF
```

## 创建虚拟机

在准备好存储、网络、安装程序和模板后，现在可以创建虚拟机了。

```sh
# vm create -t windows-uefi -s 100G windows-guest
# vm add -d network -s public windows-guest

# vm create -t linux-uefi -s 100G linux-guest
# vm add -d network -s public linux-guest
```

上述操作使用相应的模板创建了 Windows 和 Linux 虚拟机，并为每个虚拟机分配了 100GB 的存储（使用文件存储）。同时，为每个虚拟机添加了一个连接到 `public` vSwitch 的网络接口。

Windows 虚拟机在安装过程中需要稍作调整，以便在安装 VirtIO 驱动程序之前能够进行网络访问。编辑虚拟机配置，将网络接口从 virtio-net 更改为 e1000：

```sh
# vm configure windows-guest
```

将 `network0_type="e1000"` 修改为 `e1000`。

待安装了 RedHat VirtIO 驱动程序，可以恢复为 `virtio-net`。

## 安装和使用虚拟机

要安装每个虚拟机：

```sh
# vm install windows-guest Windows11-bhyve.iso
# vm install linux-guest AlmaLinux-9.5-x86_64-dvd.iso
```

待安装启动，bhyve 将进入等待状态。它将在与 VNC 控制台端口建立连接之前不会开始 ISO 启动过程。要确定哪个虚拟机控制台正在相应的 VNC 端口上运行，可以使用子命令 `list` ：

```sh
# vm list
NAME DATASTORE LOADER CPU MEMORY VNC AUTO STATE
linux-guest default uefi 2 1G [::]:5901 No Locked (host)
windows-guest default uefi 2 8G [::]:5900 No Locked (host)
```

使用像 TigerVNC 和 TightVNC 这样的 VNC 查看器，连接每个虚拟机的 IPv6 地址和端口以开始安装：

```sh
[2001:db8:1::a]:5900 # if the host is remote
[::1]:5900 # if it is on your local machine using localhost
```

安装完 Windows 客户端后，可以加载 VirtIO 驱动程序。可以在以下位置找到驱动程序：

[https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.266-1/virtio-win-gt-x64.msi](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.266-1/virtio-win-gt-x64.msi)

(sha256 – 37b9ee622cff30ad6e58dea42fd15b3acfc05fbb4158cf58d3c792b98dd61272)

安装完 Windows 后，使用 Edge 浏览器访问上述链接下载并安装这些驱动程序。安装完成后，关闭主机，将虚拟机配置文件中的网络接口切换回 virtio-net，系统即可正常启动。

现在，我们已经安装了可用的客户端，但需要对其进行控制，以便根据需要启动和停止。以下命令将对你的客户端执行基本操作，如启动、停止或立即关闭：

```sh
# vm start linux-guest
# vm stop windows-guest
# vm poweroff windows-guest
```

`stop` 和 `poweroff` 之间的区别在于，`stop` 向客户端发出 ACPI 关机请求，而 `poweroff` 立即终止 bhyve 进程，并不会干净地关闭客户端。

## 总结

本文简要介绍了如何控制 bhyve 并使用 FreeBSD 包管理库中直接提供的工具安装常见操作系统。vm-bhyve 能做的远不止本文所述的内容，详细信息可以参考 vm(8) 手册页。

---

Jason Tubnor 拥有超过 28 年的 IT 行业经验，涉及多个领域，目前是 Latrobe Community Health Service（澳大利亚维多利亚州）的 ICT 高级安全负责人。自 1990 年代中期接触 Linux 和开源软件，并于 2000 年开始使用 OpenBSD，Jason 利用这些工具在不同行业的组织中解决了各种问题。Jason 还是 BSDNow 播客的共同主持人。
