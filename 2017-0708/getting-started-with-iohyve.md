# iohyve 入门

作者：Trent Thompson

如果你曾在系统管理的波涛中航行，大概接触过虚拟化——它能让你轻松不少。不必再架子上摆满计算机硬件，虚拟化能将一台计算机变成多台，节省宝贵的空间和计算周期。虚拟化既能帮上小忙，比如在工作站上安装 VirtualBox 或 VMWare 来试用最新最热门的 Linux 发行版；也能撑起大生意——科技巨头亚马逊用 Xen 虚拟化驱动其 AWS 云产品，2016 年带来超过 120 亿美元营收。当然，本文的目的不是让你成为亿万富翁，而是让你的生活轻松一点。我们将借助 FreeBSD 内置的两项关键技术来实现：ZFS 与 bhyve。

## iohyve 的由来

ZFS 是文件系统兼逻辑卷管理器，早在 2008 年的 FreeBSD 7 中就已进入 FreeBSD，当时还被视为实验性。它源自 Sun Microsystems（现 Oracle）在 2005 年随 OpenSolaris 操作系统发布，此后被移植到许多其他操作系统。ZFS 的关键特性包括自动数据完整性、软件 RAID（称为 RAID-Z）、可充当裸盘的逻辑卷（Zvol）以及快照（还能回滚到之前的快照）。我们稍后再说这些特性为何对 iohyve 重要。

当然，要把一台计算机变成多台，还需要某种虚拟化技术。事实上，FreeBSD 自大约 FreeBSD 4 起就有了 Jail（比 ZFS 支持更早）。FreeBSD Jail 提供轻量级虚拟化 FreeBSD 用户空间的手段——你只能虚拟化 FreeBSD，因为所有 Jail 共享同一个 FreeBSD 内核。它是把一台 FreeBSD 主机变成多台 FreeBSD“虚拟机”的好办法，每台都有自己的 IP 地址，并彼此隔离（按设计和默认配置，一个 Jail 无法访问另一个 Jail 中的数据）。Solaris 有类似技术叫 Solaris Zones，至今仍被 SmartOS（基于 OpenSolaris）等虚拟化方案使用。Linux 也有自己的容器，由 LXC 甚至 Docker 提供。这些方案大多有一个问题：无法虚拟化其他操作系统。比如，你无法在 Jail 中运行 Linux，也无法在 Docker 容器中运行 FreeBSD。这正是硬件辅助虚拟化派上用场的地方。借助 Intel CPU 上的 VT-x 或 AMD CPU 上的 AMD-V 等 CPU 技术，一种称为 hypervisor 的特殊软件套件允许一个操作系统虚拟化另一个。注意我用的不是“模拟”这个词。模拟器（如 QEMU）只模拟处理器；而 hypervisor 把虚拟机直接接到 CPU 本身，因此相比模拟，性能提升巨大。我提到过几个方案，包括 Xen。Xen 是强大的 hypervisor，在 FreeBSD 上也可用。它自 2003 年起步，发展相当可观。Linux 发行版内置了 KVM（Kernel-based Virtual Machine）。Linux KVM 自 2007 年起就有，SmartOS 甚至用它来虚拟化 Zones 派不上用场的其他操作系统。iohyve 使用的 hypervisor 是 bhyve，自 FreeBSD 10（2014 年）起内置于 FreeBSD。你会注意到，相比前面提到的其他 hypervisor，它相当年轻。bhyve 是更轻量级的 hypervisor，其内核模块和用户空间工具占用不到 500 KB。bhyve 也被移植到其他操作系统，包括 Mac OS X，在那里叫 xhyve，Docker 用它来为 Linux 容器干重活。bhyve 虽轻量，但功能强大、特性丰富。

几年前，我在自己的系统管理工作中需要虚拟化一些 Linux 服务器。尝试多种方案后，我选定了 FreeBSD，主要看中 ZFS 支持。当时我用 Oracle VirtualBox 配合 phpVirtualBox 作为 GUI，远程管理虚拟机。我把 VirtualBox 配置成把所有虚拟机存放在单个 ZFS dataset 上。这样做还行，可以给 dataset 拍快照做备份，但无法把虚拟机分别存放在独立的 ZFS dataset 上——对我来说后者更合理，这样可以单独为每台虚拟机拍快照，而不是给所有虚拟机一起拍。我曾短暂尝试修改 VirtualBox，让它为每台虚拟机创建独立的 dataset，但工作量超出我的预期。差不多同一时间，我在玩 FreeBSD Jail，甚至用 Jail 把 phpVirtualBox 与 VirtualBox 主机隔离开——万一 PHP Web 服务器被攻破，攻击者也被困在 Jail 里，到不了 VirtualBox 主机本身。起初我用 ezjailadmin 实现这一点，但很快换成了 iocage——一个以类似我想对 VirtualBox 做的方式利用 ZFS 的 Jail 管理器（每个 Jail 都在自己的 ZFS dataset 上）。如前所述，Jail 很棒，但只能“虚拟化”FreeBSD，不能虚拟化 Linux 等其他操作系统。不过我非常喜欢 iocage 的设计理念：每个 Jail 不仅有自己的 dataset，还把每个 Jail 的属性存储在该 dataset 的 ZFS 用户属性中，从根本上消除了配置文件或对数据库依赖的需要。iocage 项目还吸引我的一点是使用 shell 脚本——代码一眼就能看懂，无需编译。不过 iocage 项目后来放弃了这两点，主要因为此后它成长太多——他们改用 UCL 配置以加快速度（ZFS 用户属性查询在繁忙系统上可能很慢），又改用 Python 来摆脱编写庞大 shell 脚本的头疼。

起初我的想法是给 iocage 加上 bhyve 支持，但很快发现这无异于把方钉硬塞进圆孔，而圆孔本身还在不断变化（iocage 增长太快，难以跟上变化）。终于有一天下午，我拼凑出一个基础脚本，模仿 iocage 的简洁命令行界面，把 bhyve 与 ZFS 粘合在一起。最初它只是一个脚本，功能非常有限，比如只支持 FreeBSD 虚拟机，且只支持存为文件的单个虚拟磁盘。我把脚本作为文本文件传到 GitHub 的 Gist 仓库，给少数人看了看。他们都希望功能更多，于是我最终创建了正式的 GitHub 仓库——至今仍在。这让我能把一个临时拼凑的脚本变成一个完整项目：有可作手册的 wiki，有问题跟踪器帮助报 bug，还允许任何人通过 pull request 贡献代码。自创建以来，iohyve 不断成长，特性与 bug 修复接连加入。iohyve 历史上一个标志性时刻是有人提议用 ZFS ZVOL 更好地利用 ZFS。ZVOL 可以像普通 dataset 一样创建和拍快照，但对操作系统而言基本就是磁盘设备。这与许多 Linux KVM 和 Xen 方案在 LVM 分区上存储虚拟机的方式很像，能减少一些性能开销。后来，借助 grub2-bhyve 支持（顾名思义，相当于 bhyve 的 GRUB 引导加载器），iohyve 也支持运行其他 BSD（如 NetBSD 和 OpenBSD）以及各种 Linux 发行版。这是因为 bhyve 没有内置 BIOS 来加载操作系统的重要部分。曾有 `bhyveload` 用于加载基于 FreeBSD 的操作系统，以及 `grub-bhyve` 帮助启动可用 GRUB 引导的操作系统。后来 bhyve 支持 UEFI 启动，意味着不再需要 `bhyveload` 或 `grub-bhyve`——UEFI 固件可用来加载操作系统，就像如今许多现代计算机在“裸机”上加载操作系统的方式。借助这种 UEFI 固件，你能运行许多不同类型的操作系统，包括现代版的 Microsoft Windows 操作系统家族。bhyve 近期新增的另一项功能是 UEFI-GOP 提供的图形控制台。最初 bhyve 只支持模拟串行控制台，意味着虚拟机不支持带鼠标和键盘的 GUI（如 Oracle VirtualBox 的远程显示功能）。自 FreeBSD 11 起，bhyve 内置了这种 UEFI-GOP 支持，作为基础 VNC 服务器。自 BSDCan 2016 起，iohyve 就能利用这个 VNC 服务器来帮助安装带有图形安装程序的操作系统，如 CentOS 和 Windows。虽然这是个很酷的功能，但我们不会过多讨论 UEFI 启动的 bhyve 虚拟机。

## 如何使用 iohyve

iohyve 无聊的起源故事就讲到这里，下面来实际学习如何使用 iohyve！本教程中使用 FreeBSD 11 Release。虽然 bhyve 最热的新功能（及功能分支）大多在 current 分支上完成，我选择 release 版，纯粹是为了用 `freebsd-update` 和 `pkg` 轻松更新。你的选择可能不同。iohyve 的重要组件 ZFS 可以在安装过程中安装和配置。即便你可能希望把 iohyve 虚拟机或“guest”放在单独的 zpool 上，我仍选择把 FreeBSD 装在 ZFS 上。如果你有运行虚拟机的余力，就有运行 ZFS 的余力（条款和条件可能适用）。我经常把一台机器上所有的磁盘（哪怕只有一块）打包成一个 zpool，把 FreeBSD 装上去（通常叫 zroot）。安装完成后，你会得到一个机会：在重启进入全新 FreeBSD 安装前先进入 shell 做些调整。我借此机会安装一些依赖和工具，让新系统开箱即用。你的“技术栈”可能不同，但这是我开始时觉得有用的一个组合。这条 `pkg` 命令如下：

```sh
pkg install sudo nano tmux git htop bhyve-firmware grub2-bhyve
```

`sudo` 的用法相当直白：bhyve 需要 root（暂时如此），而我不信任用户。与其给他们 root 账号访问权限，不如通过 `sudo` 委派。我知道你们中有些人对我用 nano 投来异样眼光，但自从我开始编辑配置文件，就一直用 nano。当然，base 里的 `edit` 比 `vi` 还简单，但 nano 的操作界面对我来说已是肌肉记忆，无需思考。下一个是 `tmux`，终端多路复用器。它好用之处很多，第一是：即便你的 SSH 会话中断，会话仍能保持打开。由于 bhyve 默认不使用图形控制台，所有通信都通过 null modem（类似串行连接）进行。`tmux` 让你可以为每个 iohyve guest 的控制台开一个新窗口或窗格。我稍后会讲如何用它来帮助监控 iohyve 主机。我喜欢用 `git`，因为即便你可以通过 Ports 或 `pkg` 安装 iohyve，仍能确保从 iohyve GitHub 主分支获得最新最棒的 bug 修复。我还安装了 `htop`，说实话就是为了它那漂亮的 CPU 图表输出，方便一眼监控资源。`bhyve-firmware` 软件包安装 bhyve UEFI 固件，本文不会深入，但备着总没错。iohyve 与 UEFI 的更多信息见 man 手册页。最后是 `grub2-bhyve`，它让我们能把其他 BSD 或 Linux 发行版作为 iohyve guest 运行。一切安装完毕后，我把默认用户加入 sudoers 文件，重启进入新系统。此后我用 SSH 连接机器。理论上，这些操作也都能在控制台上完成。

登录后第一件事，就是开一个新的 `tmux` 会话，简单地：

```sh
tmux
```

我得提醒你，如果你以前没用过 `tmux`，它非常棒，有点像 GNU screen。接下来，我用以下命令从 GitHub 安装新版 iohyve：

```sh
git clone https://github.com/pr1ntf/iohyve.git
cd iohyve/
sudo make install clean
cd ~
```

如果你仔细找，也可以下载最新的 master ZIP 文件。也有发布版本可用，同样以 Port 和软件包形式提供。现在我们可以安装它（如果用 `sudo`）了。

瞧！Makefile 的魔法把一切都搬到了它该在的位置，包括 man 手册页和 RC 脚本。现在我们需要设置 iohyve。其中一条设置命令只需运行一次；另一条则要在每次启动 iohyve 主机时运行。这可以用 RC 脚本轻松完成，我们稍后介绍。首先，我们在 zpool 上设置 iohyve。例中使用内置的 zroot 池。你可以用任何已配置好的池。

```sh
sudo iohyve setup pool=zroot
```

接下来，我们要为 iohyve 主机设置所需的内核模块和网络。你可以手动完成，但如果你的 iohyve 主机只用来跑 iohyve，我建议用以下方法。注意此方法不适用于无线网络。关于 iohyve 与无线的搭配，外面有一些文档，但本教程中我们以以太网设备为例设置网络。所有 iohyve guest 会接到一个 bridge 上，而该 bridge 接到一个 interface。还有更复杂的配置，但这些功能对 iohyve 来说较新，截至本文撰写时 bug 仍在修复中。更多信息见 man 手册页。下例中我使用 iohyve 主机的 em0 接口。你可以用一条 `ifconfig` 看出哪些接口已正常联网：

```sh
sudo iohyve setup kmod=1 net=em0
```

要让 iohyve 在每次启动时都做这些，可在 **rc.conf** 中加入以下内容：

```ini
iohyve_enable="YES"
iohyve_flags="kmod=1 net=em0"
```

接下来，我们要创建第一个 iohyve guest。先创建 guest，再修改一些属性，以便运行 Ubuntu Linux。然后直接把 Ubuntu ISO 抓到 iohyve 本地 ISO 仓库。任何安装 ISO 都必须在 iohyve 仓库中，且必须由 iohyve 自己添加到仓库（你不能简单地把 ISO 移到某个目录）。首先是创建，这里我把它命名为 ubantuserver，并给它一个 16 GB 的虚拟硬盘：

```sh
sudo iohyve create ubantuserver 16GB
```

接下来需要修改一些属性。我给 guest 加描述、改 RAM 和 CPU 给更多资源（2 GB 内存和两颗虚拟 CPU），并配置它用 `grub-bhyve` 启动 Ubuntu 内核。这些都可以用一条 iohyve 命令完成：

```sh
sudo iohyve set ubantuserver description="Ubuntu 16.04 Server" ram=2048M cpu=2 loader=grub-bhyve os=Ubuntu
```

记得把描述字符串放在两个双引号之间。你不必非得设描述，默认描述是 guest 创建时的时间戳。接下来，我们从 Canonical 抓取 ISO。由于我们不用 GUI 安装，从最快镜像抓服务器版（你的镜像可能不同，更多信息见 Ubuntu 网站）：

```sh
sudo iohyve fetchiso http://mirror.pnl.gov/releases/xenial/ubuntu-16.04.2-server-amd64.iso
```

这会自动抓取 ISO 并放到该放的位置。如果 ISO 已经抓过了，可以运行类似下面的命令：

```sh
sudo iohyve cpiso /full/path/to/iso.iso
```

现在是 `tmux` 真正派上用场的地方。我们按 Ctrl+B 然后“c”创建一个 `tmux` 窗口。Ctrl+B 是默认动作键，类似 screen 中的 Ctrl+A。“c”创建新窗口，按 Ctrl+B 然后“n”切到下一个窗口，Ctrl+B 然后“p”回到上一个窗口。在这个新的 `tmux` 窗口中，我们打开到新 guest 的控制台：

```sh
sudo iohyve console ubantuserver
```

此时应该什么都看不到，因为我们还没启动 guest。让我们用上面描述的快捷键（Ctrl+B 然后 C）切回前一个 `tmux` 窗口。可以用以下命令查看 iohyve 本地仓库中的 ISO 列表：

```sh
iohyve isolist
```

我们应该能看到刚抓取的 ISO：

ubuntu-16.04.2-server-amd64.iso。

要开始安装，运行以下命令启动新 guest 并附带 Ubuntu ISO：

```sh
sudo iohyve install ubantuserver
```

现在切到打开控制台的 `tmux` 窗口，如果一切顺利，你速度够快的话能看到 GRUB 菜单，或者一片滚动的文字墙。如果你看到的 GRUB 提示符类似“grub>”，那要仔细检查 guest 的 `os` 属性设置。不同发行版有不同特性，iohyve 可以处理这些。安装过程应该和其他 Ubuntu 安装一样，只需确保选择 LVM 安装（这是默认选项）。如果你选择直接装在 ext4 上而不用 LVM，可能要把 `os` 属性设为“debian”，因为非 LVM 的 Ubuntu 安装特性类似。有时安装过程中控制台会出现内核消息，特别是在分区时。这正常，只是使用串行控制台的产物。如果一切顺利，你会被提示重启。点 OK 后回到之前的 `tmux` 窗口。如果 iohyve 没有自动重启 guest，可以先确认它已停止，然后启动 guest 进行第一次完整启动：

```sh
iohyve list
sudo iohyve start ubantuserver
```

在打开控制台的 `tmux` 窗口中，你应该看到新的 Ubuntu guest 启动。然后你就可以用 Ubuntu 做任何想做的事。如果不喜欢 Ubuntu，可以看看 man 手册页，了解 iohyve 能管理哪些操作系统的特性。只要资源够，你可以反复这么操作。为了盯紧资源，我建了一个特殊的 `tmux` 窗口，分四个窗格。如何创建和调整窗格见 `tmux` man 手册页；如果你像我有时一样懒，可以在另一个 `tmux` 窗口里分别跑这四条命令。用窗格只是给它一种花哨的“控制面板”感觉。首先以 root 运行 `htop`，过滤只显示以“bhyve”开头的进程。这让我们能看到各个 bhyve guest 占用多少 CPU 和 RAM（配上那些漂亮的柱状图！）。下一个 `tmux` 窗口或窗格里，我喜欢用 `iohyve info` 看一下已安装 guest 的当前视图、它们的资源以及是否在运行。再下面两条是 `systat` 输出，分别给出磁盘使用和网络使用情况。记得把 `systat` 输出中的“tap”接口与 `iohyve info` 的输出对应起来。

```sh
sudo htop
iohyve info -sv
systat -iostat
systat -ifstat
```

你还可以用 `iohyve info -v` 检查一切是否设置妥当：

```sh
iohyve info -v
```

希望你现在掌握了足够的知识，或许可以把一些散落的 Linux 服务器搬到虚拟主机上，或者只是弄一个不错的沙盒来测试新东西、学习新技能。你可能会发现自己喜欢用别的方式监控资源，或者只需运行 FreeBSD guest，或只运行 Windows guest。希望 iohyve 能帮你顺利航行在系统管理的波涛中。如果你对 iohyve 有问题或疑问，或想申请新功能，甚至想为 iohyve 做贡献，请前往 GitHub 页面（<https://github.com/pr1ntf/iohyve>），总会有人回应。iohyve 项目由志愿者运营，所以别指望即时回复，但我们一般会回复 GitHub issue（<https://github.com/pr1ntf/iohyve/issues>）。

---

**TRENT THOMPSON** 白天是安全工程师，夜晚是 FreeBSD 与虚拟化爱好者，维护并为 iohyve 项目做贡献。不做 BSD 相关活动时，你能在家里发现他在折腾其他技术玩意——比如音乐合成器、模型火箭，或者 20 世纪 80 年代的微型计算机。爱好永远不嫌多。
