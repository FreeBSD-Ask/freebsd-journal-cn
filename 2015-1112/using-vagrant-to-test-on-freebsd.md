# 在 FreeBSD 上使用 Vagrant 进行测试

- 原文：[Using Vagrant to Test on FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/migrating-jail-management-from-warden-to-iocage/)
- 作者：**Brad Davis**

随着持续集成在全球开发领域日益普及，让相关团队能够快速拉起一个操作系统（OS）容器、针对该容器进行测试、然后迅速回收这些资源，变得愈发必要。

这种做法已普遍流行的一种常见场景，是为 SaltStack、Puppet 或 Ansible 等众多配置引擎（CFE）工具编写 recipe。这让现代运维开发（DevOps）团队能够快速评估对配置管理器的改动：拉起一台他们需要测试的目标机器版本，运行 recipe，必要时进行调试，直到其正常工作。测试成功后，测试机即可销毁。

## 简化构建

对测试环境快速访问的需求，催生了若干旨在让敏捷开发者能按需快速创建、测试并回收测试环境的工具与工具集。Vagrant 让任何人都能轻松下载一个 OS 镜像，并迅速在虚拟机（VM）中拉起用于测试。

正如 Vagrant 官网（<http://www.vagrantup.com/about.html>）所述：

> Vagrant 是一款用于构建完整开发环境的工具。凭借易用的工作流和对自动化的专注，Vagrant 缩短了开发与环境的搭建时间，提升了开发与生产环境的一致性，并让“在我机器上能跑”这类借口成为历史。

Vagrant 由 Mitchell Hashimoto 于 2010 年 1 月发起。在将近三年的时间里，Vagrant 一直是 Mitchell 的副业项目，他在全职工作之余的闲暇时间投入其中。这段时间里，Vagrant 逐步赢得了从个人到大型公司整个开发团队的信任与使用。

2012 年 11 月，Mitchell 创立了 HashiCorp，以全职支持 Vagrant 的开发。HashiCorp 构建商业附加组件，并为 Vagrant 提供专业支持和培训。

Vagrant 始终是一款采用宽松许可的开源项目，并将一直如此。Vagrant 的每一次发布都是数百位贡献者共同参与开源项目的成果。

Vagrant 可用于 Windows、Mac OS X、CentOS、Debian 系统，可从 <https://www.vagrantup.com/downloads.html> 获取。常见支持的虚拟化平台包括 VMware 和 Oracle VirtualBox。

使用 Vagrant 这类工具能让此流程极为高效，并可借助 Packer 等工具轻松配置成可重复的流程。

## Packer

如前所述，Vagrant 在测试各类配置变更时非常有用。许多在不同企业和开发团队工作的同事每天都在使用它。我最初接触 Vagrant 是通过 Packer。Packer 是 HashiCorp 的另一个项目，旨在通过虚拟机中的虚拟键盘执行命令来编排安装程序。Packer 支持 Amazon Web Services（AWS）、Digital Ocean、VMware、QEMU、Oracle VirtualBox 等众多虚拟化环境中的虚拟机。

HashiCorp 在其网站（<http://www.packer.io>）上对 Packer 的定义是：

> Packer 是一款从单一源配置为多个平台创建机器镜像和容器镜像的工具。

简而言之，Packer 提供了一个进入虚拟机的接口，命令可以像操作员坐在键盘前一样输入。因此，Packer 脚本主要由键盘命令和等待语句构成。Packer 的输出通过模拟的 VGA 控制台产生，因此脚本无法判断某条命令是否已完成。

这一自动化层带来了一些困难，因为 Packer 要求操作员理解所输入命令的时序。如果一条命令在下一命令执行前未能完成，可能无法正确缓冲，导致脚本彻底失败，或之后的每条命令各自失败。因此建议把等待时间留得充裕些。

操作员还必须根据虚拟机所建硬件类型理解系统时序。是用 SSD？机械盘？5400 还是 7200 转 SATA？10K 还是 15K 的 SAS/SCSI？系统的整体健康状况和负载如何？这些因素都会影响所运行命令的时序。

幸运的是，借助 Packer 和 Vagrant，设置和测试这些配置非常轻松，可按需调整。最终，一份 Packer 脚本可以调优，用于为多个平台构建 VM 镜像。

## Packer 与 Vagrant

借助 Packer，我开发出了一套 recipe，能够创建虚拟机、安装 FreeBSD，再将其打包供 Vagrant 使用。该脚本随后可以修改和调优，以支持传统裸金属硬件以及基于固态硬盘的系统。由于 Packer 让构建虚拟机变得如此轻松高效，做实验其实并不痛苦。

## 发布 Vagrant 镜像

在完成 Packer 相关工作后，我们发现 FreeBSD 项目已经支持构建各种类型的虚拟机，包括 Amazon EC2、Google Cloud Compute、VMware 和 Oracle VirtualBox。这意味着从正常发布流程创建 FreeBSD Vagrant 镜像应当相对简单。在 FreeBSD 发布工程师 Glen Barber 的帮助下，几周内就构建出了镜像，可随 FreeBSD 10.2-RELEASE 使用。

## Vagrant 入门

默认情况下，Vagrant 有所谓的“base box”。base box 是预先配置好与 Vagrant 协同工作、并预装了一些工具的 OS 镜像。base box 至少应具备在虚拟化环境下良好运行所需的工具。有些 base box 被预配置为完整的开发栈环境，已预装数据库、Web 服务器等。

下面演示在 Apple 电脑上设置 Vagrant 的步骤。

环境参数——物理系统：

- 运行 OS X 的 MacBook Pro
- Vagrant base box：FreeBSD base box

### 前置条件

第一步，如果尚未安装 Oracle VirtualBox（<https://www.virtualbox.org/wiki/Downloads>）或 VMWare（<https://www.vmware.com/products/desktop-virtualization.html>），请先在你的机器上安装。

然后从 <http://www.vagrantup.com/downloads.html> 下载 Vagrant。

将两个软件包都安装到你的系统上。

### 拉起虚拟机

打开终端，创建一个目录用于存放配置和 base box。配置存放在名为 `Vagrantfile` 的文件中。

本例中创建一个名为 testing 的目录，并安装 FreeBSD 10.2-RELEASE base box。

```sh
brad@penelope:~> mkdir testing
brad@penelope:~> cd testing
brad@penelope:~> vagrant init FreeBSD/FreeBSD-10.2-RELEASE
brad@penelope:~> vagrant up
```

init 阶段会从 FreeBSD 发布工程师发布的官方镜像下载 base box。下载并设置完毕后，下一步会克隆该 VM 并启动它。这一步很重要，因为它能让快速测试在克隆中进行，随后可轻松回滚到干净状态并重新启动。

VM 启动后，登录到虚拟机很简单：

```sh
brad@penelope:~> vagrant ssh
```

默认情况下，所有 Vagrant base box 都有一个名为“vagrant”的用户，并预装了 ssh 密钥。通常在一个系统上存在已知用户和公开可用的 ssh 密钥是重大的安全漏洞。Vagrant 试图通过把 VM 配置为 NAT 模式、将 VM 隐藏在你本机 IP 地址之后来回应这一顾虑。这能阻止本机以外的任何人直接访问该虚拟机。虽然还可以采取其他安全措施更好地保护本地系统，但这已超出本文范围，此处不做讨论。

关于 Vagrant 系统中 root 用户的另一重要提示：通常会包含 sudo 软件包并配置成允许“vagrant”用户做任何它需要做的事。但如果出于某种原因需要 root 密码，默认的 root 密码是“vagrant”。最新信息请始终参考 Vagrant 文档站点：<https://docs.vagrantup.com/v2/boxes/base.html>。

好了，该装些工具开始了。首先安装 NGINX 和 VIM 软件包（以下命令需以 root 身份执行）：

```sh
# pkg install -y nginx vim
```

安装完成后，验证 NGINX 能按预期启动并做一些测试：

```sh
# service onestart nginx
```

找到机器的 IP 地址，并确认 Nginx 正常工作：

```sh
vagrant$ ifconfig
em0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
options=9b<RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,VLAN_HWCSUM>
ether 00:0c:29:32:33:06
inet 172.16.245.130 netmask 0xffffff00 broadcast 172.16.245.255
nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>
media: Ethernet autoselect (1000baseT <full-duplex>)
status: active
```

如上所示，这台 Vagrant 机器的 IP 当前是 **172.16.245.130**。打开浏览器访问 <http://172.16.245.130>，正常运行的 NGINX 实例应返回默认页面。恭喜！NGINX 工作正常。

### Vagrant 的威力

现在来做点危险的事，借以展示 Vagrant 的威力。

举例来说，假设 FreeBSD 内核“神秘地”消失了（需以 root 身份执行）：

```sh
# rm -fr /boot/kernel/kernel
```

VM 此刻仍运行正常，并且只要内核还驻留在内存中就很可能继续如此。可一旦重启会发生什么？简而言之，它再也回不来了：

```sh
# reboot
```

留出充足时间让 VM 重启，再尝试重新连接：

```sh
brad@penelope:~> vagrant ssh
```

它最终会超时。现在该怎么办？

当然，修复 VM 有多种方式；如果这是生产系统，我们可能会采取其中任何一种。但这是测试环境，Vagrant 的威力就在于能快速回滚到干净状态：

```sh
brad@penelope:~> vagrant destroy
brad@penelope:~> vagrant up
brad@penelope:~> vagrant ssh
```

现在我们又回来了，可以继续。

其他一些有用的命令包括关闭 Vagrant 实例：

```sh
brad@penelope:~> vagrant halt
```

当然内置帮助也很有用：

```sh
brad@penelope:~> vagrant help
```

用完某个 box 后，可使用 `vagrant box` 子命令与之交互。例如列出可用的 box：

```sh
brad@penelope:~> vagrant box list
```

销毁不再需要的 box：

```sh
brad@penelope:~> vagrant box destroy FreeBSD/FreeBSD-10.2-RELEASE
```

## 总结

本文介绍了 Vagrant，以及如何用它拉起虚拟机进行测试。希望这能为你简化工作流、让软件或基础设施的测试与开发更轻松提供一些思路。更多信息请访问 Vagrant 官网：<http://www.vagrantup.com>。HashiCorp 维护着一个名为 Atlas 的 Vagrant box 仓库，用于测试和运行各种操作系统与配置。官方 FreeBSD Vagrant box 列表可在 Atlas 的 FreeBSD 区块查看：<https://atlas.hashicorp.com/freebsd/>。

---

BRAD DAVIS 曾担任系统架构师、开发者和顾问。作为 FreeBSD committer 已逾 10 年，参与项目的许多不同领域。他从文档 committer 起步，并以此知识协助集群管理和 Postmaster 团队。从这些项目退休后，他涉猎过 pkg 和 poudriere 项目。目前 Brad 从事文档、Ports 以及 RaspBSD 项目的工作，该项目将为 FreeBSD 在 BeagleBone Black 和树莓派等 ARM 系统上提供额外工具。难得有空闲时，Brad 喜欢滑雪和骑摩托车。
