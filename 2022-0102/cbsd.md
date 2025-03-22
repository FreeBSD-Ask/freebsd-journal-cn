# CBSD：第一部分——生产环境

- 原文链接：[CBSD: Part 1-Production](https://freebsdfoundation.org/wp-content/uploads/2022/03/CBSD-Part-1-Production.pdf)
- 作者：**OLEG GINZBURG**

2012 年，我作为 Nevosoft 的 IT 系统管理员工作，这是一家小型游戏开发公司，所有的服务器基础设施都基于 FreeBSD 操作系统开发。当时，没人听说过 Kubernetes 和 Docker，但得益于 FreeBSD Jail，公司的服务器通过将所有组件分开，每个服务都使用独立的 Jail 容器，从而受益。如今，FreeBSD 拥有十几个（甚至更多）容器编排程序，尽管在 2012 年时，选择并不广泛。那时有 ezjail，但它是为独立服务器提供的解决方案。我们的安装环境有三十到四十台物理服务器，每台服务器上运行着十到二十个容器。除了基本的创建操作外，删除、启动和克隆容器，公司的系统管理员还需要更高级的功能，如远程服务器上的容器管理、容器从一台服务器迁移到另一台服务器的能力，以及将容器保存为可移植镜像的功能。这些结果是通过创建简单的 shell 脚本实现的。然而，2013 年，出于应用专有软件的需要，导致公司开始迁移到 Linux。因为到那时，所有创建的 shell 脚本在功能上已经与 ezjail 平起平坐，某些情况下还增加了独特的功能（例如，最初有 TUI—基于文本的用户界面），于是决定将这些脚本集合合并，并以相同的标题发布到 FreeBSD Ports Tree 中。这标志着 CBSD 项目的开始。

## 与其他管理系统的比较

如今，FreeBSD 至少支持三十种用于管理容器和虚拟机的工具。在 FreeBSD 平台上有 bhyve，从这里开始，列表逐渐变长。2022 年标志着 CBSD 项目的十周年——它持续更新并继续扩展。到目前为止，它是 FreeBSD 平台上最古老的虚拟环境管理系统之一。虚拟环境不仅意味着基于 Jail 的容器化，还支持基于 bhyve、XEN 和 QEMU / NVMM 超级虚拟机监控器的虚拟机。

该项目的开发基于以下概念和哲学：

- 既关注单一模式的安装，也关注由多个主机构成的环境；
- 有机会与其他解决方案集成；
- 对变化保持开放，并从大局出发思考：越大的构想，潜力越大；而较小的想法通常不容易扩展；
- 保持简洁和灵活的使用方式。

> “简单就是终极的复杂。”——列奥纳多·达芬奇

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

这两种对话框的输出是生成文本配置文件，其中包含 jcreate 脚本的参数集合，你可以直接从 TUI 界面或命令行调用该脚本。

选项 3：使用配置文件或命令行参数。

与之前的方法相比，这种方法较为复杂，因为它需要了解可配置参数的名称，但它适合于环境开发的自动化。你可以通过以下方式合成配置文件模板：

```sh
jname="myjail1"
ver=13.0
baserw=0
pkglist="misc/mc shells/bash"
astart=0
ip4_addr="DHCP"
...
```

启动：`cbsd jcreate jconf=<文件路径>`

此外，也可以直接通过命令行参数指定选项，而无需使用配置文件：

```sh
cbsd jcreate jname=myjail1 ver=13.0 baserw=0 pkglist="misc/mc
shells/bash" astart=0 ip4_addr="DHCP"
```

配置文件和命令行参数可以结合使用：

```sh
cbsd jcreate jconf=<文件路径>
```

**选项 4：类似 [Vagrant](https://www.vagrantup.com/) 的风格：CBSDfile**

这是一种描述 CBSD 虚拟环境的标记方式，允许同时描述容器设置并在创建时进行操作自定义（例如，将文件复制到容器文件系统并配置服务）。尽管在一个 CBSD 文件中可以描述无限数量的环境，并通过调用 `cbsd up` 和 `cbsd destroy` 来创建和删除它们，但这种标记方式对于某些目录结构来说更为用户友好，通常每个目录仅描述一个环境，例如：[https://github.com/cbsd/cbsdfile-recipes/tree/master/jail](https://github.com/cbsd/cbsdfile-recipes/tree/master/jail)

此外，通过 CBSD 文件格式创建的环境的独特能力与本地环境以及通过 [CBSD API](https://www.bsdstore.ru/en/cbsd_api_ssi.html) 的环境无关。

## Jail 模板

我们不会详细讨论典型的容器工作操作，因为这些内容已经在 [网站](https://www.bsdstore.ru/en/docs.html) 上描述。我们将介绍一些 CBSD 项目的创新，旨在简化在 jail 中引用，特别是与容器模板的工作。模板是容器描述和配置的方法。对于容器来说，它可以被导出为可移植的镜像。容器可以包含一个标准的运行时环境，但在大多数情况下，它们用于服务/应用程序的隔离和通过镜像进行分发。这种方式有其优缺点。容器方法的一些优点包括服务部署的速度、不会影响基本系统环境，以及能够提交依赖项的版本，使得应用程序完全正常运行。

谈到这种方法的缺点，最主要的问题是安全性，特别是当使用没有自包含构建脚本（模板）的容器或镜像时。这使得对容器进行操作性软件更新变得困难（例如，修复 [0-day](<https://en.wikipedia.org/wiki/Zero-day_(computing)>) 漏洞），并且容易出现后门，这些后门可能是图像收集者故意或无意留下的。考虑到安装某些软件的复杂性，通常会发现容器中配置的服务是手动放置的，或者丢失了组装说明。

另一个缺点是容器服务配置的复杂性。有些镜像可以通过环境变量配置有限数量的参数，尽管这并不总是足够的。一个例子是动态配置，当需要为 WEB 服务器添加和配置多个虚拟主机（vhost）并在数据库管理系统（DBMS）中设置多个数据库时。对于这样的任务，有一个专门的软件部分，称为配置管理。该类别中最知名的产品包括 [Ansible](https://www.ansible.com/)、[Chef](https://www.chef.io/)、[Puppet](https://puppet.com/)、[Rex](https://www.rexify.org/) 和 [SaltStack](https://saltproject.io/)。然而，它们通常是为了适应经典环境而优化的。CBSD 可以结合配置管理器的全部功能和基于容器的方法，从而获得带有管理服务的容器。CBSD 中使用的模板有两种类型：

- 静态（经典）模板，主要仅包含安装软件。例如，CBSD 的一个模板是 sambashare 容器。

类似的模板可以在其他项目中找到：例如，Fockerfile 示例中的 [nginx](https://github.com/sadaszewski/focker/tree/8be1fcaf601cca6afd703f96b1f90946a9831ff3/example/gateway/nginx-http) 模板（[focker](https://github.com/sadaszewski/focker) 项目）：

```sh
base: freebsd-latest
steps:
 - run:
 - ASSUME_ALWAYS_YES=yes IGNORE_OSVERSION=yes pkg install nginx
```

或者，这是 [BastilleBSD](https://bastillebsd.org/) 项目中 [RabbitMQ](https://github.com/BastilleBSD-Templates/rabbitmq/tree/e1a1275090c3d47b38402ab73941afb3c33cf3a6) 模板的示例，其中 CMD 文件的内容是：

```sh
service rabbitmq restart
```

**PKG** 文件内容：

```sh
rabbitmq
python27
```

这些模板的实用性有限，因为它们只是以另一种形式编写了两行 shell 命令：

```sh
pkg install -y rabbitmq python27
service rabbitmq restart
```

这些模板更为复杂，涵盖了服务配置文件的复制，并且在某些情况下，配置过程会伴随着通过复杂的 sed/awk 结构的处理，整个容器配置过程也会包括这些步骤。

作为一般原则：

a) 这是一项不可逆的操作，旨在一次性使用：无法回滚或更改配置，除非重新创建容器；

b) 这种模板对其维护要求很高：即使容器中进行了一次小的服务更新，源配置文件也可能会发生实质性变化；

c) 无法保证服务会正常运行，或者在配置错误的情况下无法回滚到以前的运行时版本；

d) 在 sed/awk 操作和切换到使用配置管理程序之间，难以找到合适的界限。

这是问题的解决方案，也是 CBSD 项目的主要信息：不要重新发明轮子，而是在可能的情况下重复使用他人的工作。许多开发者维护着配置管理模块，因此我们建议为了节省时间，使用他们的工作来委派配置管理，并使用原生的工具来处理配置。为了实现这一目标，CBSD 从 2016 年起引入了 forms 脚本。CBSD 的 [Forms](https://www.bsdstore.ru/en/13.0.x/wf_imghelper_ssi.html) 功能用于获取和保存用户参数，以适合配置模块的 [YAML](https://en.wikipedia.org/wiki/YAML) 格式——一种特殊的中间件。Puppet 被选为配置管理系统，但也可以使用其他任何框架。

## 为使用模板准备 CBSD

为了更好地理解，假设我们创建一个 jail1 容器，并将 [redis](https://redis.io/) 服务模板应用于它。

目前，预计 CBSD 已正确安装和配置：你可以启动一个容器，并且 `cbsd dhcpd` 命令会为客户端分配能够访问互联网的地址（例如通过 NAT）。这是 pkg 操作和包安装在容器中所必需的条件。

1. 必须安装 Git 来从公共仓库 https://github.com/cbsd 安装 CBSD 插件。如果尚未安装，安装 git（约 220 MB）或其轻量版 git-lite（约 40 MB）：

```sh
pkg install -y git-lite
```

2. 从 CBSD 基础分发版中包含的配置文件创建系统容器 cbsdpuppet1。这一步需要在每个新的 CBSD 主机上执行一次：

```sh
cbsd jcreate jname=cbsdpuppet1 jprofile=cbsdpuppet
```

cbsd puppet1 容器不应运行，因为它只执行一个功能——它包含了 CBSD 用于配置容器的 [puppet7](https://www.freshports.org/sysutils/puppet7/) 安装包。

3. 安装 CBSD puppet 模块——这是可选功能，不包含在基本分发版中。该模块包含了由 CBSD 项目检查的 puppet 模块。这一步需要在每个新的 CBSD 主机上执行一次：

```sh
cbsd module mode=install puppet
```

4. 安装 redis 服务配置模块。这一步需要在每个新的 CBSD 主机上执行一次：

```sh
cbsd module mode=install forms-redis
```

5. 创建一个容器，名称任意，在其中我们将获取 redis 服务，例如：jail1：

```sh
cbsd jcreate jname=jail1 runasap=1
```

（注意：参数 `runasap=1` 意味着容器将立即启动）

6. 最后一步：将 redis 服务模板应用到我们的 jail1 容器中：

```sh
cbsd forms module=redis jname=jail1
```

将出现熟悉的 TUI 界面，显示 Redis 模块的参数，你可以根据需要进行配置：

![](https://github.com/user-attachments/assets/6c449efc-a184-40fc-867e-de07e1498e82)

让我们保留默认参数，并通过 [COMMIT] 操作选择它们。脚本运行完成可能需要一段时间（取决于互联网连接速度，因为模块从官方仓库 pkg.FreeBSD.org 安装 redis 服务器），因此你需要再次确认容器中的服务已安装并正在运行：

```sh
~ # cbsd jexec jname=jail1 sockstat -4l
USER COMMAND PID FD PROTO LOCAL ADDRESS FOREIGN ADDRESS
redis redis-serv 8910 7 tcp4 172.16.0.13:6379 *:*
```

通过重新使用 cbsd forms 调用，你可以随时重新配置服务。此外，forms 会保留以前的值，并且在初始化时始终输出当前数据。例如，让我们更改一组参数：

- 将端口设置为 7777；

- 设置连接到 redis 服务器的密码；

- 将 maxmemory 参数设置为 4g；

- 将 maxmemory_policy 参数设置为 noeviction。

应用新参数后，检查状态：

```sh
~ # cbsd jexec jname=jail1 sockstat -4l
USER COMMAND PID FD PROTO LOCAL ADDRESS FOREIGN ADDRESS
redis redis-serv 12587 7 tcp4 172.16.0.13:7777 *:*
~ # cbsd jexec jname=jail1 grep ^maxmemory /usr/local/etc/redis.conf
maxmemory 4g
maxmemory-policy noeviction
```

大多数 Puppet 模块会承诺进行不正确的数据验证、配置文件验证和服务状态检查。因此，由于无效参数输入导致无法服务的服务的可能性，比静态模板要小得多。

现在我们已经了解了 TUI 界面，让我们转向自动化。cbsd forms 允许通过环境变量接受模板的参数，从而省略交互式对话。为了查看某个特定模板接受哪些变量，可以使用 `vars` 参数并指明所需模块：

```sh
~ # cbsd forms module=redis vars
H_BIND H_PORT H_REQUIREPASS H_MAXMEMORY H_MAXMEMORY_POLICY
H_TCP_KEEPALIVE H_LOG_LEVEL H_SYSLOG_ENABLED H_TIMEOUT H_SLAVE_PRIORITY
H_SLAVEOF
```

Forms 会在常规参数的名称前添加 `H_` 前缀，以减少与系统全局变量发生潜在冲突的可能性。让我们第三次重新配置 redis 端口（将其设置为 9999），但不使用交互模式（`inter=0`）：

```sh
env H_PORT=9999 cbsd forms module=redis jname=jail1 inter=0
```

请注意应用模板时输出的值：

![](https://github.com/user-attachments/assets/b7cf7bb8-6ae1-4d78-a816-354ddecc5e24)

这是模板工作结果导出的 CBSD 变量的输出信息。参数化格式可以通过配置文件 `forms_export_vars.conf` 在目录 `~cbsd/etc` 中设置。

查看其默认值：

[https://github.com/cbsd/cbsd/blob/v13.0.18/etc/defaults/forms_export_vars.conf](https://github.com/cbsd/cbsd/blob/v13.0.18/etc/defaults/forms_export_vars.conf)

这些值可以自动导出到文件或各种服务发现服务，如 [Consul](https://www.consul.io/)。在这种情况下，你的集群会自动获取这些值，可以用于构建 SOA（面向服务的架构），但这是另一个话题。

除了 redis，[这里](https://github.com/cbsd) 还有其他服务配置模板：命名为 `modules-forms-XXXX` 的仓库。例如，尝试使用以下模板：

• modules-forms-memcached

• modules-forms-mysql

• modules-forms-grafana

• modules-forms-rabbitmq

• modules-forms-postgresql

• modules-forms-elasticsearch

注意：尽管传统上 1 个服务对应 1 个容器，但对于 CBSD forms，你可以将多个模板应用于同一个环境，从而一次性获得所有服务。

## 它是如何工作的

Puppet 模块（或任何其他类似系统）包含了在 99% 的情况下采用值参数形式的参数，这些参数通常采用 YAML 格式进行参数化。为了方便用户，CBSD 提供了一个 TUI 界面来处理基本表单，在这个界面中，与 HTML 表单类似，以下元素可以存在：

• 单选按钮（布尔类型的值为 true/false、yes/no）；

• 复选框元素；

• 已知特定值的下拉菜单；

• 自定义输入框用于相对用户输入；

为了将参数保持在通用形式中，CBSD forms 使用 [SQLite3](https://www.sqlite.org/index.html) 数据库——一种描述参数输入的表单，存储输入的值。记录（写入）格式是通用的，既适用于 TUI 对话框的自动生成（CBSD forms），也适用于 WEB/HTML 表单（[ClonOS](https://clonos.convectix.com/)），如下例所示。让我们使用 CBSD forms 所需的格式创建一个 SQLite3 数据库：

![](https://github.com/user-attachments/assets/cbb13782-3e4a-49e7-b83a-f761e1a77700)

让我们详细谈谈这些行中的一些内容。对我们来说，最重要的表格列是：

- `param` – 直接的参数名称，表示我们希望获取的值；
- `desc` – 相关的描述（如果适用）；
- `def` – 默认值；
- `new` – 用户输入的新值；
- `type` – 字段类型：inputbox、delimiter、password、group_add、group_del、radio、checkbox。

接下来的两行可以添加两个参数，我们希望获取的值是：FreeBSD 和 DragonFlyBSD，每个都有自己的默认值，并且都使用 inputbox 字段格式，因此用户可以自由地输入任意字符串值。

在这个表单上运行 cbsd forms 脚本：

```sh
cbsd forms formfile=/tmp/myforms.sqlite
```

输出将是一个动态构建的表单，用于处理参数：

![](https://github.com/user-attachments/assets/446765ee-e5bd-41ef-88d0-319feb936317)

redis 服务的表单结构更为复杂，因为它包含布尔类型和下拉菜单类型的字段元素。让我们来看一下表格：

![](https://github.com/user-attachments/assets/2f2b6823-665a-4853-8d8d-61b8a08d794c)

我们以表格 [memory_policy_select](http://clonos.bsdstore.ru:81/phpliteadmin.php?database=%2Fusr%2Fjails%2Fvar%2Fdb%2Fredis.sqlite&table=memory_policy_select&fulltexts=0&numRows=30&action=row_view) 为例，这是 memory_policy 参数的选择类型变体表：

![](https://github.com/user-attachments/assets/670a38c9-b562-4856-9387-b5e7d1931349)

在 WEB 界面中，自动生成的表单（截图来自 ClonOS）如下所示：

![](https://github.com/user-attachments/assets/ed585bb2-caf3-4324-bfb6-906b75830eda)

其中，maxmemory 策略参数位于特定的下拉菜单元素中。

![](https://github.com/user-attachments/assets/26c8f6f0-3743-4ebc-bd4f-c465ddaa6921)

TUI 界面提供了类似的选择：

![](https://github.com/user-attachments/assets/1c718aa8-d542-4031-9aac-652cd4ad54c1)

> **注意**
>
> 所有 CBSD TUI 对话框都可以通过类似的基于 SQL 的表单来描述，这些表单目前仅用于与模板的工作。

在获取并处理参数后，cbsd forms 之后，cbsd puppet 模块介入了操作。接下来发生的事情如下：

- 使用 nullfs 挂载 overlay 文件系统，将 cbsdpuppet1 系统容器中的 puppet7 包文件挂载到目标容器（jail1）的 `/tmp/XXX` 目录；
- 将包含模块参数的 YAML 文件呈现到目标容器（jail1）；
- 获取 puppet apply 指令，应用于可配置容器（jail1），以便从 `/tmp/XXX` 目录应用 puppet 清单。

例如，以下是应用参数时 nullfs 挂载点的样子：

![](https://github.com/user-attachments/assets/276a0979-fc93-4e61-81fb-f2c876349569)

服务完成并重新配置后，临时的 nullfs 文件系统将被卸载，因为它们已经不再需要。因此，如果我们查看上述示例中的 jail1 容器中已安装的软件，将不会看到 puppet7 包或它依赖的文件。

```sh
~ # cbsd jexec jname=jail1 pkg info
pkg-1.17.4 Package manager
redis-6.2.6 Persistent key-value database
```

换句话说，通过使用 cbsdpuppet1 容器，你无需在每个容器中安装 puppet7 和必要的依赖项——配置应用不依赖于有限容器中是否存在任何系统软件。

## 尾声

cbsd forms 是 CBSD 框架在处理基于 jail 的容器时最强大的功能之一，它在 CBSD 和配置管理器之间架起了桥梁。因此，CBSD 项目通过适当的 Puppet 模块为 FreeBSD 提供支持，而不是依赖 sed/awk 脚本支持和静态模板。如果某个模块能够在 FreeBSD 环境中运行，你可以通过 cbsd forms 透明地使用它。这对特定的 CBSD 用户和所有 FreeBSD 上的 Puppet 用户都有好处。如果你使用其他配置系统，可以通过使用 cbsd forms 脚本调用其他系统。

CBSD 项目支持基于现有模板的即用型镜像的振荡和分发，你可以通过 CBSD 仓库使用 cbsd repo 和 cbsd images。请参阅内联文档和示例：

```sh
~ # cbsd repo --help
~ # cbsd images --help
```

在系列文章的下一篇中，我们将探讨 CBSD 的虚拟机管理功能。

---

**OLEG GINZBURG** 生活在俄罗斯，担任 [X5 Retail group](https://x5.ru/en/) 的 DevOps 工程师。他自 ZX Spectrum（**译者注：英国公司 Sinclair Research 于 1982 年发布的个人电脑。**）以来便热爱计算机科学，是 Unix 爱好者和 FreeBSD 爱好者。他是 FreeBSD 的推广者，并参与多个项目：CBSD、[ClonOS](https://clonos.convectix.com/)、[MyB](https://myb.convectix.com/)、[K8Sbhyve](https://k8s-bhyve.convectix.com/)、以及 [AdvanceBSD group](https://www.reddit.com/r/AdvanceBSD/)。
