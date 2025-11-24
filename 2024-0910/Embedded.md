# 嵌入式 FreeBSD：探索 bhyve

- 原文链接：[Embedded FreeBSD: Digression into bhyve](https://freebsdfoundation.org/our-work/journal/browser-based-edition/kernel-development/embedded-freebsd-digression-into-bhyve/)
- 作者：Christopher R. Bowman

在之前的两篇文章中，我们讨论了我一直在实验的 [Digilent Arty Z7-20](https://digilent.com/shop/arty-z7-zynq-7000-soc-development-board/) 开发板。我觉得这是一块很有趣的开发板，因为它不仅可以切换主要引脚与外界进行接口，而且你还可以将自己的电路集成到开发板中，并将其与处理器进行接口。然而，要做到这一点，你需要使用 Xilinx/AMD 的软件来配置和编程芯片。Xilinx 将这个工具套件称为 [Vivado](https://www.xilinx.com/products/design-tools/vivado.html#editions)，你可以从他们的网站上 [免费下载](https://www.xilinx.com/products/design-tools/vivado/vivado-ml.html) 一个适用于 Zynq 芯片的版本。

那么，缺点是什么呢？肯定有什么潜在的问题，对吧？其实有两个问题。首先，Vivado 只有 Windows 和 Linux 版本。其次，下载的文件本身就有 110GB。多年来，我在 Mac 上通过 VMWare 运行 Linux 版本，效果还不错。但最终，我的虚拟机有好几个不同版本，每个版本的软件安装情况都不同。这些虚拟机非常大，早期的版本大约是 30GB，最近的一些安装版本甚至达到了 75GB。假设在几台机器上有几个版本，那么总空间开始变得非常庞大，而且我也很难追踪哪个版本是最新的。

于是，我决定简化并集中管理。我希望将所有下载的原始厂商工具存储在我的网络文件服务器上，这台服务器自然运行 FreeBSD，拥有数 TB 的 ZFS 存储空间。毕竟，我不能依赖于厂商继续提供这些旧版本的工具，尤其是在他们更新到新版本后。这是可以理解的，但作为一个爱好者，我的工作进度并不总是与厂商的进度同步，因此我希望确保能够保留原始工具下载文件。我希望我的主目录以及我的工作文件和项目都存储在我的网络服务器上，这样它们可以从我所有的机器上访问，并像任何其他数据一样进行备份。我希望所有解压并安装好的厂商软件都存储在我的文件服务器上，而不是存放在我的虚拟机里。这样，我可以利用 ZFS 的强大功能对厂商软件安装进行检查点保存。我的虚拟机只是一个轻量级的 Linux 安装，任何我想要实验的 Linux 版本。如果我需要升级或创建新的虚拟机，我知道里面没有任何工作内容，所有东西都在我的文件服务器上，所以我只需要安装一个新的基本系统，其它所有内容仍然保存在文件服务器上。我无需复制或重新安装厂商工具。由于最近我家计算环境的升级包含了一台 16 核 128GB 内存的 FreeBSD 机器，我希望在这台机器上运行这些工具，以应对我的设计规模变大，可能需要数小时来合成的情况。作为额外的好处，我得到了一个设置，其他人如果想要复制我的工作，只需要一台 FreeBSD 机器即可。

目前似乎有两种基本的方法。我可以尝试让安装程序在 FreeBSD 的 Linux 仿真环境下运行，并在我的 FreeBSD 机器上本地运行这些工具，而不需要 Linux 虚拟机。如果能够实现，那将是非常棒的！但我不确定是否能做到，也不清楚可能会遇到哪些问题，或者需要多长时间才能弄明白，而我当时正处于几个项目的中间。这个方法看起来确实是最好的，我很想知道是否有人已经成功实现，或者有谁想尝试，但我选择了一个我认为会涉及较少工作量且更有可能成功的方法。我听说过 bhyve，决定进行调查。如果我能运行一个支持 Vivado 的 Linux 版本的虚拟机，我认为那将是最简单的。只用了一个晚上阅读和另一个晚上实验，我就惊讶地成功运行了。

我从 FreeBSD 手册开始。它真的是一个很棒的资源，感谢所有帮助使其如此出色的人。[第 24.6 章：作为主机使用 FreeBSD 和 bhyve](https://docs.freebsd.org/en/books/handbook/virtualization/#virtualization-host-bhyve) 对如何使用 FreeBSD 作为主机操作系统做了很好的介绍。在大多数情况下，我从那里和互联网上的其他一些地方拼凑出了一个设置。为了设置，我创建了网络的基本主机配置：

```sh
# kldload vmm

# ifconfig tap0 create up
# ifconfig bridge0 create
# ifconfig bridge0 addm igb0 addm tap0
# ifconfig bridge0 up
```

我正在使用 [24.6.5 节：为 bhyve 客户机配置图形 UEFI 帧缓冲](https://docs.freebsd.org/en/books/handbook/virtualization/#virtualization-host-bhyve) 中的说明，因为我不想麻烦地设置 grub。使用 UEFI 帧缓冲还允许我通过 vnc 导出 Linux 显示。如果我使用 FreeBSD 主机，或者从网络上的任何其他机器，都可以连接。不过，也许我应该再考虑一下安全性问题。

我下载了一个 CentOS 的 ISO 版本，是在 RedHat 终止它之前的版本，然后我使用以下虚拟机配置进行安装：

```sh
# bhyve -c 4 -m 32G -w -H \
        -s 0,hostbridge \
        -s 3,ahci-cd,/u1/ISOs/CentOS/CentOS-7-x86_64-DVD-2009.iso \
        -s 4,ahci-hd,/dev/zvol/zroot/vms/centos7 \
        -s 5,virtio-net,tap0 \
        -s 29,fbuf,tcp=0.0.0.0:5900,w=1920,h=1200,wait \
        -s 30,xhci,tablet \
        -s 31,lpc -l com1,stdio \
        -l bootrom,/usr/local/share/uefi-firmware/BHYVE_UEFI.fd \
        vm0
```

通过这种方式，我得到了一个 4 核心、32GB 的虚拟机，ISO 镜像 `/u1/ISOs/CentOS/CentOS-7-x86_64-DVD-2009.iso` 被挂载为客户机中的 CDROM。这样，我可以运行一个标准的图形界面安装 Linux，过程非常直接。我是在裸 zvol 上运行的：`/dev/zvol/zroot/vms/centos7`。这样做是为了我可以使用 ZFS 快照随时对虚拟机进行快照（最初是在干净安装后），以便随时回滚。如果我将虚拟机虚拟磁盘放在 ZFS 文件系统上，快照会应用到该文件系统中的任何内容，而不仅仅是虚拟机虚拟磁盘映像。我还听说使用裸 zvol 可能更快。回想起来，我本可以为每个虚拟机虚拟磁盘映像使用一个单独的数据集，并在该数据集上使用文件作为虚拟磁盘映像，而不是使用 zvol。如果每个虚拟机有一个数据集，快照仍然只会应用于虚拟机。我不确定哪种方法更好，随后的阅读让我对 zvol 与文件支持的速度产生了疑问。我的工具运行时间还不长，因此我还不太关心每一分性能的提升。我还听说，对于支持的客户操作系统，NVMe 设备比 AHCI 硬盘设备更快，但我还没有进行实验。如果这个问题变得严重，创建一个新的虚拟机就很容易了，因为我的数据和安装并不依赖于虚拟机。

安装了 CentOS 后，我配置它通过 NFS 挂载我的 FreeBSD 机器上的主目录，然后我进行了 Xilinx Vivado 工具的安装。这是一个图形界面的安装，过程很简单。我只需要确保安装路径是 NFS 挂载的目录。考虑到文件操作的量，我原本预期这个过程会非常痛苦，但实际上它相当快，尤其是考虑到安装后工具的体积达到了 66GB。要知道，我有一个非常快速的本地网络。我不打算编辑工具的安装，但是我创建了一个快照——这也是为了安心。我不想再重复安装一次。

当我申请 Vivado 许可时，我复制了 Linux 客户机报告的以太网 MAC 地址。这个地址似乎在重启时保持稳定，但我希望能弄清楚如何配置它，以确保我的虚拟机中的 MAC 地址始终与 Vivado 许可中的 MAC 地址匹配。

现在，我有了一个相当通用的 CentOS 虚拟机，安装了 Vivado，可以通过 VNC 从我本地网络上的任何机器访问。到目前为止，我还安装了一个 15 年前的 Linux 版本的《文明：天赋神权》（*Civilization: Call to Power*），以便在我的 FPGA 构建编译时玩。这让我惊讶于它运行得多么流畅（以及我有多么上瘾）。

尽管大部分功能都运行得很好，但与我之前在 Mac 上使用 VMWare 的设置相比，还是有一些不同之处。首先，VMWare 支持将文件系统传递到主机。NFS 挂载基本完成了相同的功能，我没有注意到太多的速度惩罚，但我必须在 Linux 中配置，而不是在 VMWare 的图形界面中配置。这不是一个大问题——我对此非常适应。真正让我希望能有 VMWare 解决方案的地方是复制和粘贴。如果我在 Linux 客户机的图形界面中选择了一些内容，我无法轻松地将其复制并粘贴到运行 VNC 查看器的机器上。这偶尔是个痛点，但由于我在 Linux 中使用的用户文件系统都是从 FreeBSD 挂载的 NFS，我可以像从 FreeBSD（或任何其他挂载它们的机器）一样轻松编辑这些文件。因此，我几乎不在 Linux 下进行编辑或工作。大部分时间，我只是在 Linux 的 VNC 会话窗口中运行 Vivado 编译，其他所有工作都在外面进行。总体来说，这个方法还是相当有效的。

在下期文章中，我们将开始使用 Vivado 构建我们的第一个电路。

如果你对这些内容有反馈、抱怨或批评，我很乐意听到你的声音。你可以通过 [articles@ChrisBowman.com](mailto:articles@ChrisBowman.com) 联系我。

---

**Christopher R. Bowman** 自 1989 年在约翰霍普金斯大学应用物理实验室的 VAX 11/785 上首次使用 BSD 以来，一直在使用 FreeBSD。他在 90 年代中期使用 FreeBSD 设计了自己在马里兰大学的第一个 2 微米 CMOS 芯片。从那时起，他一直是 FreeBSD 的用户，并对硬件设计及其驱动的软件非常感兴趣。他在半导体设计自动化行业工作了 20 年。
