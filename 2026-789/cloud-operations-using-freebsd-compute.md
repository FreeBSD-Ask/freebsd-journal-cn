# 使用 FreeBSD 计算资源开展云运维

- 原文：[Cloud Operations Using FreeBSD Compute](https://freebsdfoundation.org/our-work/journal/browser-based-edition/production-deployments/cloud-operations-using-freebsd-compute/)
- 作者：**Jason Tubnor**

## 在云 IaaS 中开展操作系统多样化时，FreeBSD 是出色的选择

现代云已经存在约二十年。亚马逊提出面向零售的计算服务（EC2）这一想法后，AWS 随之成立，出租这些计算资源，并提供对象存储、数据库等其他服务。其他厂商也纷纷加入：微软有 Azure，谷歌有 GCP，甲骨文有 OCI，为客户提供丰富多样的 Tier 1 云基础设施。

起初能用的只有 Linux，主要受限于当时的技术条件：KVM 出现之前的半虚拟化不允许其他操作系统内核运行。

不过，情况开始改变。2008 年，也就是 AWS EC2 公开可用两年之后，FreeBSD 开发者 Colin Percival 开始把 FreeBSD 操作系统移植到 AWS。两年的努力得到回报，2010 年 12 月，Colin 在 AWS EC2 基础设施上成功启动 FreeBSD。又打磨了 5 年，2015 年 4 月，FreeBSD 成为 EC2 上的标准产品。

工作并未止步于此。FreeBSD 继续完善，进入其他 Tier 1 提供商的应用市场，也进入一些 Tier 2 云提供商。遗憾的是，Tier 2 提供商对许多操作系统产品的支持有所收缩，不只是 FreeBSD。不过现在，只要你准备自行提供支持，有些提供商可以让你轻松自带 ISO。这些提供商通常通过 VirtIO 框架提供设备，而 VirtIO 本就内置在 FreeBSD 基本系统中，因此在这些环境里启动原版通用系统很容易。

在云基础设施即服务（IaaS）环境中开展操作系统多样化时，FreeBSD 是出色的选择。与任何技术一样，某些环节应当多样化，万一某个平台出现 bug，服务仍能不中断。FreeBSD 为你提供这份保障：设备驱动原生实现，新近加入的开放容器计划（OCI）让你可以轻松用现有工具管理 FreeBSD IaaS 计算实例上的工作负载。

云上 FreeBSD 镜像还有其他特性：支持 IPv6，有些还支持 ZFS，有些则只支持 UFS。如果你只需要一台基本的 FreeBSD 设备，后者或许就够用；但如果应用与管理需要“Root on ZFS”，也可以选用 ZFS。

FreeBSD 进驻各家云应用市场还有一层好处：使用免费，在其上开发也免费。没有持续的订阅费用，你可以开一个实例，也可以开一千个。除计算资源外，一切成本为零。有些应用市场确实会让你走与订阅制操作系统相同的流程，但点击订阅按钮时不会产生费用。

不过，好处还不止这些。你打算在 ARM64 架构上运行吗？FreeBSD 同样支持。从 2024 年起，AWS 等提供商开始为自家的 Graviton ARM CPU 提供 ARM64，用户可以选择成本更低、功耗更小的平台。越来越多组织转向环境可持续模式，运营的各个环节都需要减少碳足迹，这也是 FreeBSD 和 ARM 计算能帮上忙的地方。

启动 FreeBSD 计算实例，可以通过 Web 控制台，也可以通过 AWS CLI API 工具。

FreeBSD 桌面用户可以轻松获得 CLI 工具，用来管控各家云厂商的租户。只需从 FreeBSD 软件包仓库安装：

```sh
# pkg install py312-awscli
```

安装完成后，用你的 AWS 管理控制台凭据认证：

```sh
$ aws login
```

此时你会在一定时间内保持认证状态，一旦长时间无操作便会自动登出。

这里假定你已经定义好 SSH 公钥、安全组、子网。实例创建并启动后，为把暴露面降到最低，这些都是必需的。完成 EC2 创建还需要其他信息：实例类型、需要多少台，以及你希望它位于哪个区域。

要创建实例，执行：

```sh
$ aws ec2 run-instances --image-id ami-[freebsd AMI ID] --count 1 --instance-type t4g.micro --region ap-southeast-2 --key-name key-[my SSH public key ID] --security-group-ids sg-[security group ID] --subnet-id subnet-[subnet ID]
```

视安全组和子网的设置而定，如果实例设为公开可用，你就可以直接从工作站 SSH 登录：

```sh
$ ssh -I .ssh/id_awsec2 ec2@mynewhost.example.com
```

系统启动时由 cloud-init 完成最基本的主机设置，让它运转起来。获取 root 权限只需执行 su。注意：本文不讨论实例的安全加固。

FreeBSD 计算可以对接云厂商提供的其他 PaaS 服务。需要 ZFS 存储时，可以给计算实例挂载多个块存储卷。用 FreeBSD NFS 客户端可以挂载弹性文件系统，访问按用量计费的文件存储。FUSE 文件系统可以把 S3 对象存储挂载到实例上。或者直接让 FreeBSD 实例上的应用通过网络连接托管数据库。FreeBSD 只提供你创建系统所需的组件，由此得到的系统攻击面小、活动部件少，每台主机的启用成本也随之降低——大型项目需要上千台主机时，这些成本会不断累积。

本文只触及了 FreeBSD 作为计算操作系统在功能与灵活性上的一角，但希望在你需要把“下一个大事件”部署到云上时，能给你一些启发。

---

**Jason Tubnor** 拥有超过 28 年 IT 行业经验，涉猎领域众多，目前是 Latrobe Community Health Service（澳大利亚维多利亚州）的 ICT 高级安全主管。他在 20 世纪 90 年代中期接触 Linux 与开源，2000 年又结识 OpenBSD，此后用这些工具解决了不同行业组织中的各种问题。Jason 还是 [BSDNow Podcast](https://www.bsdnow.tv/) 的联合主持人。
