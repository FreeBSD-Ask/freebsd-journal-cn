# Amazon EC2 云与 FreeBSD

- 原文：[FreeBSD in the Amazon EC2 Cloud](https://freebsdfoundation.org/our-work/journal/browser-based-edition/virtualization/freebsd-in-the-amazon-ec2-cloud/)
- 作者：**Colin Percival**

亚马逊网络服务（AWS）是亚马逊提供的一组计算服务，其共同特征是：通过 HTTP(S) 隧道化的请求配置；通过相同方式或服务特定的标准协议访问；采用自助式、按用量付费的定价模型运营。

凭借亚马逊为其零售业务构建和运营计算基础设施的经验，AWS 是现代云计算行业的最早入局者，并至今仍是规模最大的玩家。亚马逊提供数十种服务，但最常用的可归为四大类。

## AWS 可用性服务

- **身份和访问管理**（Identity and Access Management）——创建具有指定权限的完全特权“root”子账户
- **CloudFormation**——脚本化部署一组 AWS 资源
- **CloudWatch**——采集并监控 AWS 和自定义指标
- **CloudTrail**——AWS API 调用日志

## 核心基础设施服务

- **弹性计算云**（Elastic Compute Cloud，EC2）——虚拟机
- **简单存储服务**（Simple Storage Service，S3）——响应时间约 50 ms 的耐久数据存储
- **Glacier**——响应时间约 4 小时的低成本耐久数据存储
- **DynamoDB**——高性能耐久 NoSQL 数据存储
- **简单队列服务**（Simple Queue Service，SQS）——可靠消息队列

虽然消息队列看似是偏门功能，但对于分布式系统而言，拥有可靠队列来处理“完成一半”的任务至关重要——如果任务未保留在可靠队列中，系统一旦宕机就会丢失任务状态。

## 边缘位置服务

- **Route 53**——权威 DNS
- **CloudFront**——HTTP(S) 内容分发网络

## 托管软件服务

- **ElastiCache**——Memcached 或 Redis
- **关系型数据库服务**（Relational Database Service，RDS）——MySQL、PostgreSQL、Oracle 或 SQL Server
- **Elastic Map Reduce**（EMR）——Hadoop

以上所有服务都通过使用一对 AWS“访问密钥”发起的 API 调用访问。“Access Key ID”（访问密钥 ID）作为请求的一部分发送以标识账户，“Secret Access Key”（秘密访问密钥）则用于创建认证 API 请求的加密签名。亚马逊为多种编程语言提供了库，并附带命令行工具和 Web 界面来管理众多服务。本文仅关注弹性计算云服务，但对其他 AWS 服务感兴趣的读者可参考亚马逊出色的官方文档。

## 开始使用 AWS

要使用 EC2（或其他任何 AWS 服务），你需要先创建 AWS 账户。用浏览器打开 <http://aws.amazon.com/>，点击“Sign Up”按钮。然后你可以用现有的 amazon.com 账户登录或新建一个。

创建或登录到 amazon.com 账户后，系统会要求你提供姓名和联系方式、接受 AWS 客户协议、提供用于计费的信用卡，并完成“身份验证”流程——包括接听电话并输入四位 PIN 码以激活亚马逊网络服务账户。

拥有 AWS 账户后，你需要用于访问账户的密钥。用浏览器打开 <https://console.aws.amazon.com/iam/home?#security_credential>，选择“Access Keys”并点击“Create New Access Key”。记下 Access Key ID 和 Secret Access Key，创建名为 **~/.aws/config** 的文件，包含以下内容（需先创建 **~/.aws/** 目录）：

```ini
[default]
aws_access_key_id = <你的 Access Key ID 写在这里>
aws_secret_access_key = <你的 Secret Access Key 写在这里>
region = us-east-1
```

最后，你需要访问 AWS 服务的软件。本文使用 AWS 命令行界面（<http://aws.amazon.com/cli/>）。要在 FreeBSD 系统上安装它，以 root 身份运行：

```sh
# make -C /usr/ports/devel/awscli install clean
```

或

```sh
# pkg install awscli
```

对于生产用途，你大概会希望使用身份和访问管理服务创建具有受限权限集的“User”，然后为该子账户创建密钥对。为简化起见，本文使用“root”账户密钥。

## Amazon 弹性计算云（EC2）

亚马逊弹性计算云（EC2）核心是按小时计费的虚拟机，运行于 Xen 虚拟机监控器下。

当 EC2 在 2006 年首次推出时，“租用虚拟服务器”的概念早已成熟，许多公司都曾使用 FreeBSD jail 或 Xen 等虚拟化系统提供此类服务，但这类服务通常需要一次租用一个月甚至更长时间，虚拟服务器的作用仅限于作为专用服务器的廉价替代品。通过引入按小时计费的虚拟机并提供快速、自动配置新虚拟机的 API，EC2 带来了虚拟化的全新益处——灵活性。

不再按月租用服务器，EC2 让用户能够根据负载快速扩张和收缩——这正是“弹性计算云”中“弹性”的由来。

启动一台 EC2 实例——其实任何虚拟机都一样——需要指定三个重要参数：在哪里启动虚拟机、启动什么类型的虚拟机、在虚拟机上运行什么。

### AWS 区域

除了一些“支持性”服务（如计费和认证系统）外，亚马逊网络服务（包括 EC2）划分为 8 个区域：北弗吉尼亚、俄勒冈、北加州、爱尔兰、新加坡、东京、悉尼和圣保罗。每个区域独立运作。除了把服务器放在靠近通信对象以减少延迟这一明显好处外，EC2 区域还有利于冗余和合规（例如确保 EU 数据不离开欧盟）。区域名称形如“us-east-1”。

每个 AWS 区域内有两个或更多 EC2 可用区（Availability Zone）。可用区存在的目的是使冗余系统成为可能：亚马逊尽最大努力确保任何故障（电力、网络等）只影响单个可用区。可用区名称形如“us-east-1a”。

### 实例类型

EC2 刚推出时只提供单一类型的虚拟机，但多年来其产品线已扩展到 38 种类型，名称如 m3.medium、c3.xlarge 或 i2.8xlarge（一般形式为 `<family>.<size>`）。这里无法详述全部类型，但有与虚拟化技术相关的重要技术细节——FreeBSD 需要 Xen“HVM”模式的支持，而较老的 EC2 实例类型只在与 Windows 许可证捆绑时才提供该模式。

多数人会希望使用属于“当前一代”21 种实例类型之一的——M3（“通用”）、C3（“高 CPU”）、R3（“高内存”）、I2（“高 I/O”）或 T2（“低成本”）家族——它们提供比旧实例类型更好的性价比。参见下方表格。

| 名称 | CPU 数 | 内存 | 固态硬盘 | 价格（us-east-1 区域） |
| :--- | :----- | :--- | :------- | :--------------------- |
| t2.micro | 10% | 1.0 GB | 无 | $0.013/小时 |
| t2.small | 20% | 2.0 GB | 无 | $0.026/小时 |
| t2.medium | 2 x 20% | 4.0 GB | 无 | $0.052/小时 |
| m3.medium | 1 | 3.75 GB | 4 GB | $0.070/小时 |
| m3.large | 2 | 7.5 GB | 32 GB | $0.140/小时 |
| m3.xlarge | 4 | 15.0 GB | 80 GB | $0.280/小时 |
| m3.2xlarge | 8 | 30.0 GB | 160 GB | $0.560/小时 |
| c3.large | 2 | 3.75 GB | 32 GB | $0.105/小时 |
| c3.xlarge | 4 | 7.5 GB | 80 GB | $0.210/小时 |
| c3.2xlarge | 8 | 15.0 GB | 160 GB | $0.420/小时 |
| c3.4xlarge | 16 | 30.0 GB | 320 GB | $0.840/小时 |
| c3.8xlarge | 32 | 60.0 GB | 640 GB | $1.680/小时 |
| r3.large | 2 | 15.0 GB | 32 GB | $0.175/小时 |
| r3.xlarge | 4 | 30.5 GB | 80 GB | $0.350/小时 |
| r3.2xlarge | 8 | 61.0 GB | 160 GB | $0.700/小时 |
| r3.4xlarge | 16 | 122.0 GB | 320 GB | $1.400/小时 |
| r3.8xlarge | 32 | 244.0 GB | 640 GB | $2.800/小时 |
| i2.xlarge | 4 | 30.5 GB | 800 GB | $0.853/小时 |
| i2.2xlarge | 8 | 61.0 GB | 1600 GB | $1.705/小时 |
| i2.4xlarge | 16 | 122.0 GB | 3200 GB | $3.410/小时 |
| i2.8xlarge | 32 | 244.0 GB | 6400 GB | $6.820/小时 |

注意，除了“medium”和“large”尺寸的 SSD 存储容量之外，每个家族内每升一级，CPU 数、内存、SSD 存储和每小时价格都翻倍。反过来，对比不同家族的 xlarge 尺寸，可以看到“高 CPU”家族与“标准”家族相同，只是内存减半并享受 25% 的价格折扣。“高内存”则反方向移动，内存是“标准”实例的两倍，价格高出 25%；而“高 I/O”实例是“高内存”实例，SSD 存储增加 10 倍，价格再涨 145%。

T2 家族与其他 EC2 实例类型不同，拥有“可突发”CPU 而非专用 CPU：长期来看它们只能使用 CPU 时间的 10% 或 20%，但如果 CPU 利用不足，“余额”会累积，最多可累积一天。这些实例对大部分时间空闲、偶尔需要重建 Ports 或其他类似吃 CPU 任务的系统非常实用。

### Amazon 机器镜像（AMI）

确定启动实例的位置和类型后，你需要告诉 EC2 实例应该运行什么软件。用 EC2 的术语来说，这就是 Amazon 机器镜像（AMI），它包含磁盘镜像（例如 FreeBSD）和一些告诉 EC2 如何启动镜像的元数据。AMI 可以直接从 EC2 的磁盘快照创建，或通过“重新打包”运行中的 EC2 实例创建——但新用户会发现从成千上万个现成的免费 AMI 中选一个最为容易。

由于 EC2 在“云”环境中运行，要安全地连接和使用一台新虚拟机还需考虑两点：防火墙规则和登录凭证。所有 EC2 实例都启动到“安全组”中——也就是说，可以预先创建防火墙规则集并应用到新实例。默认情况下，新的安全组允许所有出站流量，但阻止所有入站流量，对出站连接的响应除外。

最后，EC2 提供指定 SSH 公钥用于登录实例的机制。该密钥通过特殊的管理接口提供（连同其他各种元数据）。在 FreeBSD 中，**sysutils/ec2-scripts** port 中的代码会读取该数据并安排用户登录。

### 启动 FreeBSD EC2 实例

如果你按照“开始使用 AWS”中的说明操作，现在应该已安装 awscli port，并在 **~/.aws/config** 中有配置文件。在启动 EC2 实例之前，我们需要创建用于登录的 SSH 密钥。运行以下命令：

```sh
$ ssh-keygen -f ~/.ssh/ec2login
$ aws ec2 import-key-pair --key-name mykey --public-key-material file://`realpath ~/.ssh/ec2login.pub`
```

这会创建 SSH 密钥（保存在文件 `ec2login` 中），并将公钥部分上传到 EC2，关联到名称“mykey”。（`file://` 这种写法是因为 awscli 代码期望所有输入都是 URI 而非简单的文件路径。我曾建议改变这点，但作者认为使用 URI 提供更好的一致性。）

我们还要创建 EC2 安全组。出于历史原因已存在“默认”组，但在为新目的启动实例时创建新的安全组是好习惯。这样以后你可以在不影响其他实例的情况下调整防火墙规则。运行以下命令创建安全组，并允许你的 IP 地址通过 SSH 连接到其内的实例：

```sh
$ aws ec2 create-security-group --group-name mygroup --description "my security group"
$ aws ec2 authorize-security-group-ingress --group-name mygroup --protocol tcp --port 22 --cidr <你的 IP 地址>/32
```

现在启动实例。通常此时你要查找想启动的 AMI——在 <http://www.daemonology.net/freebsd-on-ec2/> 上有各 AWS 区域不同版本 FreeBSD 的 AMI ID 列表，但本例我们使用 us-east-1 区域（也就是我们创建 **~/.aws/config** 时选择的默认值）中的 FreeBSD 10.0-RELEASE——该 AMI 是 `ami-69dae900`。

运行以下命令，将 FreeBSD 10.0-RELEASE 启动到 m3.medium 虚拟机，使用 mykey SSH 密钥连接，并输出实例 ID：

```sh
$ aws ec2 run-instances --image-id ami-69dae900 --instance-type m3.medium \
  --key-name mykey --security-groups mygroup --output text \
  --query 'Instances[*].InstanceId'
```

记下实例 ID，后续命令都会用到它。如果忘了，可以通过运行 `aws ec2 describe-instances` 列出所有正在运行的实例（并获取它们的大量信息）。

现在去冲一杯咖啡。我们要等大约五分钟，让 FreeBSD 启动，EC2 收集控制台输出。

回来了，咖啡因补充完毕？好，现在可以运行：

```sh
$ aws ec2 get-console-output --output text --instance-id <你的实例 ID 写在这里> | more
```

你会看到 FreeBSD 启动时通常的内核输出。往下滚动，你会看到一些标准 FreeBSD 安装中没有的东西。首先 freebsd-update 运行，下载并安装 FreeBSD 的关键更新——这很重要，因为在“云”中有一台虚拟机时，我们无法遵循通常的安全建议，即先登录并应用安全更新再把新服务器暴露到互联网上！之后你会看到 pkg 引导程序运行，awscli 软件包下载并安装。我们稍后就会看到如何更改首次启动时安装的软件包集合。最后，我们看到 FreeBSD 重启。这样如果 freebsd-update 安装了更新，或安装的软件包添加了 rc.d 脚本，我们就有运行最新代码、所有启用服务都已启动的系统。

查看 FreeBSD 在做什么很有趣，但读控制台输出真正只有一个至关重要的原因：查看 SSH 主机密钥的指纹。这些主机密钥在系统首次启动时生成，而且——再说一遍，因为在云中——我们需要使用这种带外机制来确保我们连接到系统时，SSH 登录的是正确的机器。除了密钥生成时打印指纹外，指纹还会带 `ec2:` 前缀再次打印，以便自动工具容易提取。

说到 SSH，是时候用它了。运行：

```sh
$ aws ec2 describe-instances --output text --query \
  'Reservations[*].Instances[*].PublicIpAddress' --instance-id <你的实例 ID 写在这里>
```

获取实例的公网 IP 地址，然后运行：

```sh
$ ssh -i ~/.ssh/ec2-login ec2-user@<实例 IP 地址写在这里>
```

SSH 会提示你确认 SSH 主机指纹。你可以将其与刚才在控制台输出中看到的值比较。

现在你应该以默认的非特权 `ec2-user` 用户身份登录到 EC2 实例。要切换为 root，运行 `su` 即可（标准镜像中未设置 root 密码）。这台系统现在就像任何其他 FreeBSD 10.0-RELEASE 系统一样运行。看完后退出登录。

除非你想给亚马逊添利润，否则你应该销毁 EC2 实例。运行以下命令：

```sh
$ aws ec2 modify-instance-attribute --block-device-mappings \
  '[ { "DeviceName": "/dev/sda1", "Ebs": { "DeleteOnTermination": true } } ]' \
  --instance-id <你的实例 ID 写在这里>
$ aws ec2 terminate-instances --instance-ids <你的实例 ID 写在这里>
```

第一条命令是必要的，因为默认行为是保留已终止实例的启动盘副本（标准 10 GB 实例启动盘每月花费 $0.50）。如果你想完全清理 AWS 账户，还可以使用 `aws ec2 delete-security-group` 命令删除你创建的安全组，并使用 `aws ec2 delete-key-pair` 命令删除 SSH 密钥，但这些并非必需——亚马逊对安全组或 SSH 密钥不收取任何费用。

## 弹性块存储、弹性 IP 地址、弹性负载均衡器

EC2 作为创建虚拟机的平台已经够用了，但还有几项其他服务为 EC2 增加了重要功能。首先是弹性块存储（EBS），可以创建虚拟磁盘并附加到 EC2 实例；其次是弹性 IP 地址（EIP），可以保留 IP 地址并将其分配给某个实例或在实例间迁移；第三是弹性负载均衡器（ELB），提供在实例池之间负载均衡流量的简便机制。

### 弹性块存储

弹性块存储或许更应该叫 EC2 存储区域网络。通过一次 API 调用，你可以在指定 EC2 可用区内创建 1 GB 到 1 TB 之间的虚拟磁盘，称为 EBS 卷。另一次 API 调用可将该卷附加到某个 EC2 实例。由于这些卷通过亚马逊网络访问，而非物理直连到 EC2 实例所在主机，从一个 EC2 实例卸载卷再附加到另一个实例非常容易。这有助于向新实例迁移数据，或者——FreeBSD 开发者常遇到的情况——如果系统意外变得无法启动，可以临时把启动盘从实例卸下，附加到另一台实例上修复问题，再装回原实例并重启。

EBS 有三种类型：“Magnetic”（旧称“Standard”）、“Provisioned IOPS”和“General Purpose”。Magnetic 卷便宜，但如名字所示，性能水平是典型的“旋转氧化铁”——通常是 128 kB 块上每秒 50 到 100 次 I/O 操作。Provisioned IOPS 卷则允许你指定 1 到 4,000 IOPS 之间的目标 I/O 速率（基于 16 kB 块），但更贵，尤其对大额预留 I/O 速率而言。General Purpose 卷居中：可突发到每秒 3,000 I/O 操作，但性能基线为每 GB 已分配空间每秒 3 次 I/O 操作。重要的是要记住，由于 EBS 通过网络访问，所有类型都比直接附加到 EC2 实例的 SSD 存储慢——在优化性能和优化可用性之间存在不可避免的权衡。

### 弹性 IP 地址

弹性 IP 地址本应命名为“预留 IP 地址”——因为它们正是这样。和 EBS 卷一样，弹性 IP 地址可以分配、附加到 EC2 实例，并在 EC2 实例之间迁移。这对于直接面向公网的服务器是必需的，因为没有这些地址，EC2 实例创建时会随机选取 IP 地址，且新实例无法接管已关闭的旧实例的地址。

虽然弹性 IP 地址使你可以将地址放入 DNS 并知道如果需要可以将其指向新 EC2 实例，但对于拥有大量面向公网服务器的公司来说，这还不够。这就有了弹性负载均衡器：弹性负载均衡器配置有 EC2 实例池——同样通过 API 调用添加或删除 EC2 实例——并将入站连接转发给池中的实例。

EC2 还有许多其他特性：Virtual Private Cloud，让你可以配置包括路由表、IP 子网和 VPN 端点在内的虚拟网络；Elastic Network Interfaces，让你可以为 EC2 实例附加多个地址；CloudWatch，提供对 EC2 实例的监控；Autoscaling，让你可以根据负载自动启动或关闭实例。这里没有足够篇幅详细描述一切，因此只能让读者参考亚马逊出色的网站和文档。

## 通过 configinit 进行实例自动配置

许多人可能满足于启动干净的 FreeBSD 系统并通过 SSH 登录完成必要的系统配置——尤其是对于一次性服务器——但将部分或全部配置过程脚本化并在 EC2 实例首次启动时自动执行，在速度和可重复性方面都有优势。许多 Linux 系统（包括 Ubuntu 和 RedHat）使用 CloudInit 系统，该系统允许在 EC2 实例启动时提供 user-data 文件来指定要执行的一组命令。

CloudInit 并不太适合在 FreeBSD 上使用，原因有二。首先它使用 Python，而 Python 不是 FreeBSD 基本系统的一部分。其次，CloudInit 围绕“通过运行命令配置系统”的概念构建。与 Linux 系统不同，FreeBSD 更“配置文件导向”，因此我决定编写自己的系统，称之为“configinit”。与 CloudInit 不同，configinit 系统采用非常简单的 shell 脚本形式，从 10.0-RELEASE 开始包含在 FreeBSD EC2 镜像中。但与 CloudInit 类似，configinit 在 EC2 实例首次启动时早早运行，下载 EC2 用户数据（通过用于 SSH 公钥的同一管理接口暴露），然后处理该文件。

configinit 能处理四种类型的文件：

- 如果文件以字符 `>/` 开头，则第一行去掉前导 `>` 字符后会被解释为路径，文件其余部分（除第一行外）会写入该位置（替换任何已存在的文件）。
- 如果文件以字符 `>>/` 开头，则第一行去掉前导 `>>` 字符后会被解释为路径，文件其余部分会追加到该位置（如果该位置尚无文件则创建新文件）。
- 如果文件以字符 `#!` 开头，则该文件会被执行（这最有可能用于 shell 脚本）。
- 否则，configinit 会尝试使用 bsdtar（自动检测多种归档和压缩格式）将该文件作为归档解压，然后递归处理其中每个文件，按字典序处理。

此功能最直接的用法是向 FreeBSD 的“主配置文件” **/etc/rc.conf** 添加内容。configinit 代码会指示 rc 系统重新加载该文件，以确保改动的设置会在后续启动过程中反映出来。除了标准 FreeBSD 配置选项（例如 EC2 镜像中 rc.conf 文件里出现的 `sshd_enable="YES"`），EC2 镜像中预装的软件包还有几个有用的选项：

`firstboot_pkgs_list="包列表"` 会让这些软件包在 EC2 实例首次启动时下载并安装，并自动启动相应服务：

- `firstboot_freebsd_enable="NO"` 会禁用启动时的 pkg 系统引导（在实例启动到无法访问 pkg 镜像的网络环境时有用）。
- `firstboot_freebsd_update_enable="NO"` 会禁用启动时下载并安装关键勘误和安全更新。
- `ec2_fetchkey_user="用户名"` 会更改用户名，该用户通过提供给 EC2 的公钥创建并配置，用于 SSH 访问。

例如，提供如下 user-data 文件：

```ini
>>/etc/rc.conf
firstboot_pkgs_list="apache22 python33"
ec2_fetchkey_user="webmaster"
apache22_enable="YES"
```

会预装 Apache 2.2 和 Python 3.3，创建名为“webmaster”的用户用于 SSH 登录，并启动 Apache 2.2。

configinit 的更复杂用法通常需要创建或编辑多个文件。在这种情况下，需要创建归档文件（例如 tar 包），其中包含每次配置变更所需的文件。

### 使用 configinit 启动 Drupal 实例

现在我们用 configinit 启动装好 Drupal 的系统。和之前一样的提醒——EC2 实例要花钱，所以确保探索完后清理。

首先，为我们将要启动的实例创建一对新的 SSH 密钥。虽然我们现在不打算通过 SSH 登录实例，但如果你打算日后使用这台实例，就需要能够 SSH 进去执行升级（FreeBSD 和 Drupal 所需软件包的升级）和备份。运行以下命令：

```sh
$ ssh-keygen -f ~/.ssh/drupal
$ aws ec2 import-key-pair --key-name drupal --public-key-material file://`realpath ~/.ssh/drupal.pub`
```

我们还需要创建新的 EC2 安全组（即防火墙规则集）。本例中我们想对全世界开放 80 端口，而无需 SSH 访问。如果将来需要 SSH 进该实例，可以用 `authorize-security-group-ingress` 添加一条允许 tcp/22 端口入站流量的规则。运行以下命令：

```sh
$ aws ec2 create-security-group --group-name drupal --description "drupal demo security group"
$ aws ec2 authorize-security-group-ingress --group-name drupal --protocol tcp --port 80 --cidr 0.0.0.0/0
```

接下来是 configinit 数据，运行以下命令获取 Drupal 配置文件，无需手动输入：

```sh
$ fetch http://freebsd-ec2-dist.s3.amazonaws.com/drupal-conf.tar
```

如果你查看这个归档，会看到三个名为“rcconf”、“apache”和“drupalinit”的文件。文件名并不重要；当 configinit 解压归档并逐个处理这些文件时，会按字典序处理，但本例中顺序无所谓。文件 `rcconf` 包含以下行：

```ini
>>/etc/rc.conf
ec2_fetchkey_user="drupal-user"
firstboot_pkgs_list="apache22 drupal7 mod_php5 mysql55-server"
apache22_enable="YES"
mysql_enable="YES"
```

第一行意思是“将本文件剩余部分追加到 **/etc/rc.conf**”，其余行是 rc.conf 设置，告诉 EC2 启动脚本为名为“drupal-user”的用户配置 SSH 登录；下载并安装 apache22、drupal7、mod_php5 和 mysql55-server 这些软件包；启动 Apache 2.2 和 MySQL。

文件 `apache` 写入 **/usr/local/etc/apache22/Includes/local.conf**，配置 Apache 从 **/usr/local/www/drupal7**（Drupal port 安装数据的位置）提供文件服务，并启用 PHP（Drupal 使用 PHP）。

文件 `drupalinit` 是 shell 脚本，执行让 Drupal 工作所需的步骤。它将所有者设置为“www”用户，以便 Drupal 代码（通过 Apache 和 mod_php5 运行）能正常运行，并为 Drupal 创建 MySQL 数据库和用户。但有个问题：因为 configinit 在启动过程中早早运行——必然早到能添加指定要安装哪些软件包的 rc.conf 设置——此时还没有 mysql 守护进程运行，Drupal 中需要更改所有权的文件也还不存在。为了规避这一限制，`drupalinit` 脚本创建新 rc.d 脚本，该脚本会在软件包安装完毕且 mysql 运行后执行——脚本在初始化 Drupal 数据后会自我删除，以防下次系统启动时再次运行。

现在我们已了解 Drupal configinit 文件如何工作，来实际运行一下。运行以下命令：

```sh
$ aws ec2 run-instances --image-id ami-69dae900 --instance-type m3.medium \
  --key-name drupal --security-groups drupal --user-data file://drupal-conf.tar \
  --output text --query 'Instances[*].InstanceId'
```

注意，与启动标准 FreeBSD 镜像相比，唯一的差别是新增了 `--user-data file://drupal-conf.tar` 选项。和之前一样，笨拙的 `file://` 语法是 AWS CLI 以 URI 为中心的设计所要求的。

和之前一样，我们需要等 FreeBSD 启动，EC2 收集控制台输出。五分钟后运行：

```sh
$ aws ec2 get-console-output --output text --instance-id <你的实例 ID 写在这里> | more
```

你会看到 FreeBSD 启动、自我更新、下载并安装我们要求的软件包、重启、启动 Apache 和 MySQL 守护进程，最后你会看到一行：

```sh
Drupal password: <12 个字符>
```

这是 MySQL 的“drupal”用户密码（随机生成）。拿到密码后，可以继续配置 Drupal。运行：

```sh
$ aws ec2 describe-instances --output text --query \
  'Reservations[*].Instances[*].PublicIpAddress' --instance-id <你的实例 ID 写在这里>
```

获取实例的公网 IP 地址，然后粘贴到你选择的浏览器中。当提示输入数据库参数时，告诉 Drupal 使用名为“drupal”的数据库、数据库用户名“drupal”和你从控制台输出获取的密码。恭喜，现在 Drupal 已经运行，准备就绪，可以开始向你的新网站添加内容了！

和所有 EC2 实例一样，使用完毕后要清理以避免支付不必要的费用。运行以下命令：

```sh
$ aws ec2 modify-instance-attribute --block-device-mappings \
  '[ { "DeviceName": "/dev/sda1", "Ebs": { "DeleteOnTermination": true } } ]' \
  --instance-id <你的实例 ID 写在这里>
$ aws ec2 terminate-instances --instance-ids <你的实例 ID 写在这里>
```

## FreeBSD/EC2 的未来方向

FreeBSD 自 2010 年 12 月起以某种形式在 EC2 上可用，并自 2011 年中以来稳定到可用于生产环境。此后取得了大量进展，尤其是过去一年。2012 年 10 月，亚马逊宣布 M3 实例家族，随后在 11 月推出 C3 家族，2013 年 12 月推出 I2 家族，2014 年 4 月推出 R3 家族，2014 年 7 月推出 T2 家族——这“当前一代”EC2 实例类型共同覆盖了广泛的硬件配置，让 FreeBSD 无需早期实例类型上运行 FreeBSD 时那些糟糕的 hack 就能运行。

另一方面，从 2014 年 1 月的 FreeBSD 10.0-RELEASE 起，FreeBSD 在 GENERIC 内核配置中加入了 Xen 支持，使官方发行版二进制文件能在 EC2 中运行，并使 FreeBSD Update 系统可用于二进制更新。configinit 的加入让用户能通过 EC2 user-data 配置 FreeBSD 虚拟机，如今已有脚本可以让任何人构建 FreeBSD AMI。

那么接下来呢？这很大程度上取决于 FreeBSD——和 EC2——用户的要求，但一些可能的方向包括：

- 将 EC2 AMI 构建过程整合进 FreeBSD 发布流程，由 FreeBSD 发布工程团队构建 AMI。
- 提供基于 ZFS 或 MFS 根文件系统的 EC2 AMI。
- 自动使用 EC2 临时磁盘为 EBS 卷上的数据提供缓存或镜像，以获得更高的性能和可靠性。
- 使用身份和访问管理“角色”允许 EC2 实例自动创建并向自己添加更多弹性块存储卷，实现真正的“弹性”文件系统。
- 提供更多 configinit 示例，用于开箱即用的服务器配置。

但 FreeBSD/EC2 当前最需要的是更多用户、更多测试和更多反馈。什么能用？什么不能用？你希望加入什么功能？大部分开发工作都完成了。现在是把接力棒交给更广泛的用户社区的时候了——去用吧！

---

**Colin Percival** 自 2004 年起就是 FreeBSD 开发者，2005 年至 2012 年间担任项目的安全官。他从 2006 年起努力把 FreeBSD 引入 EC2 环境，如今在他创立并持续运营的 Tarsnap 在线备份服务中广泛使用 EC2。
