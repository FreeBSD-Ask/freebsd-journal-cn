# CBSD：第一部分——生产环境

- 原文链接：[CBSD: Part 1-Production](https://freebsdfoundation.org/wp-content/uploads/2022/03/CBSD-Part-1-Production.pdf)
- 作者：**OLEG GINZBURG**

2012年，我作为 Nevosoft 的 IT 系统管理员工作，这是一家小型游戏开发公司，所有的服务器基础设施都基于 FreeBSD 操作系统开发。当时，没人听说过 Kubernetes 和 Docker，但得益于 FreeBSD Jail，公司的服务器通过将所有组件分开，每个服务都使用独立的 Jail 容器，从而受益。如今，FreeBSD 拥有十几个（甚至更多）容器编排程序，尽管在 2012 年时，选择并不广泛。那时有 ezjail，但它是为独立服务器提供的解决方案。我们的安装环境有三十到四十台物理服务器，每台服务器上运行着十到二十个容器。除了基本的创建操作外，删除、启动和克隆容器，公司的系统管理员还需要更高级的功能，如远程服务器上的容器管理、容器从一台服务器迁移到另一台服务器的能力，以及将容器保存为可移植镜像的功能。这些结果是通过创建简单的 shell 脚本实现的。然而，2013 年，出于应用专有软件的需要，导致公司开始迁移到 Linux。因为到那时，所有创建的 shell 脚本在功能上已经与 ezjail 平起平坐，某些情况下还增加了独特的功能（例如，最初有 TUI—基于文本的用户界面），于是决定将这些脚本集合合并，并以相同的标题发布到 FreeBSD Ports Tree 中。这标志着 CBSD 项目的开始。

## 与其他管理系统的比较

如今，FreeBSD 至少支持三十种用于管理容器和虚拟机的工具。在 FreeBSD 平台上有 bhyve，从这里开始，列表逐渐变长。2022 年标志着 CBSD 项目的十周年——它持续更新并继续扩展。到目前为止，它是 FreeBSD 平台上最古老的虚拟环境管理系统之一。虚拟环境不仅意味着基于 Jail 的容器化，还支持基于 bhyve、XEN 和 QEMU / NVMM 超级虚拟机监控器的虚拟机。

该项目的开发基于以下概念和哲学：
- 既关注单一模式的安装，也关注由多个主机构成的环境；
- 有机会与其他解决方案集成；
- 对变化保持开放，并从大局出发思考：越大的构想，潜力越大；而较小的想法通常不容易扩展；
- 保持简洁和灵活的使用方式。

>“简单就是终极的复杂。”——列奥纳多·达芬奇

围绕 CBSD 已经出现了广泛的工具包：Web 界面、用于将任务传递给分布式 CBSD 节点的消息代理服务、API 服务以及用于与其交互的瘦客户端。

最后，CBSD 是一个原生的 FreeBSD 产品，而不是一个尝试移植 Linux 解决方案的项目。

了解更多关于项目目标的信息：[https://www.bsdstore.ru/en/cbsd_goals_ssi.html](https://www.bsdstore.ru/en/cbsd_goals_ssi.html)

了解更多关于项目开发的信息：[https://www.bsdstore.ru/en/cbsd_history_ssi.html](https://www.bsdstore.ru/en/cbsd_history_ssi.html)

## 子项目

CBSD 不仅是面向最终用户的产品，也是参与构建复杂解决方案的元素之一，通过委派创建和管理 CBSD 虚拟环境的功能，能够节省大量工时。因此，存在多个独立的项目和发行版，用于展示和获取有关 CBSD 工作重用的见解：

- Reggae（由 Goran Mekic 开发）使用 CBSD 自动化 DevOps 任务；
- 发行包 [https://k8s-bhyve.convectix.com/](https://k8s-bhyve.convectix.com/)（k8s-bhyve）展示了能够快速（从几秒钟到 1 分钟）启动设置在 bhyve 超级虚拟机监控器上的 Kubernetes 集群的能力；
- 发行包 [https://clonos.convectix.com/](https://clonos.convectix.com/) 具有为 CBSD 上的 bhyve 创建容器和虚拟机构建 Web/UI 界面的能力；
- 发行包 [https://myb.convectix.com/](https://myb.convectix.com/)（MyBee）使用户能够通过 CBSD 与云镜像进行交互，而无需使用 UI 和 CLI：可以通过向 CBSD API 发送 HTTP 请求（例如，使用 curl 工具）或使用 nubectl 瘦客户端（[https://github.com/bitcoin-software/nubectl](https://github.com/bitcoin-software/nubectl)）来获取虚拟机。

## CBSD 和 Jail 容器：实际应用

让我们更好地了解可用的 CBSD jail 操作方法。网站上有关于初始 CBSD 安装和自定义过程的描述，假设你已经拥有运行时版本。设置容器的方式有多种：

**选项 1**：命令行对话格式。这种方法不需要学习各种可能的 jail 参数，因为脚本会自动回忆它们：


cbsd jconstruct

![](https://github.com/user-attachments/assets/b9e1bac1-1b99-4dd1-9a99-fbc4761f7a9b)

**选项 2**：通过 TUI 的对话格式。这种方式也适合初学者，因为与第一种选项一样，不需要了解可能的参数和命令：

cbsd jconstruct-tui

![](https://github.com/user-attachments/assets/3add189e-d5e8-4672-b4a9-e14cbece2fcf)

这两种对话框的输出是生成文本配置文件，其中包含 jcreate 脚本的参数集合，您可以直接从 TUI 界面或命令行调用该脚本。

选项 3：使用配置文件或命令行参数。

与之前的方法相比，这种方法较为复杂，因为它需要了解可配置参数的名称，但它适合于环境开发的自动化。您可以通过以下方式合成配置文件模板：

```sh
jname="myjail1"
ver=13.0
baserw=0
pkglist="misc/mc shells/bash"
astart=0
ip4_addr="DHCP"
...
```

启动：` cbsd jcreate jconf=<文件路径>`

此外，也可以直接通过命令行参数指定选项，而无需使用配置文件：

```sh
cbsd jcreate jname=myjail1 ver=13.0 baserw=0 pkglist="misc/mc
shells/bash" astart=0 ip4_addr="DHCP"
```

配置文件和命令行参数可以结合使用：

```sh
cbsd jcreate jconf=<文件路径> ip4_addr="10.0.0.2" runasap=1
```

