# Netflix Open Connect

- 原文链接：[Netflix Open Connect](https://freebsdfoundation.org/wp-content/uploads/2021/03/Netflix-Open.pdf)
- 作者：**GREG WALLACE**

## 概述

Netflix（纳斯达克股票代码：NFLX）是全球领先的流媒体娱乐服务，拥有超过 1.83 亿付费会员，覆盖 190 多个国家，会员可以观看各种类型和语言的电视系列、纪录片和电影。会员可以随时随地在一切连接互联网的屏幕上观看自己想看的内容，播放、暂停并继续观看，所有这些都没有广告或任何承诺。`www.netflix.com`

Open Connect 是负责将 Netflix 电视节目和电影全球传送到会员的全球网络。此类网络通常被称为内容分发网络（CDN），因为其工作是通过将人们观看的内容靠近他们观看的地方，来高效地传递基于互联网的内容（通过 HTTP/HTTPS）。Open Connect 设备运行的是经过轻微定制的 FreeBSD 版本。<https://openconnect.netflix.com/ Open-Connect-Overview.pdf>

Netflix 拥有几位 FreeBSD 提交者，Open Connect 团队的其他成员也为上游贡献了代码。

## Open Connect 达到超过 100 Tb/s 的峰值流量

我们中那些记得互联网泡沫和电信繁荣的人，可能会记得 1999 年 Quest Communications 的标志性广告——一个疲惫的旅客在偏远地区的酒店办理入住。前台工作人员承诺提供平凡的早餐，但娱乐？

他们有的是娱乐。“每部电影，任何语言，任何时间，白天和晚上。”

客人惊讶地大声问：“这怎么可能？”怎么可能！(继续阅读)。二十年后的今天，酒店电视可能是最后一个提供“每部电影”的设备。技术似乎并不缺乏讽刺意味。

在讨论流媒体娱乐和使其成为可能的技术的最新趋势时，Netflix 绝对是绕不开的话题。截至 2019 年 4 月，Netflix 美国目录中包含了 47,000 部电视节目和 4,000 部电影。Netflix 报告称，全球的 Open Connect 网络在峰值时推送超过 100 Tb/s 的流量。根据 Sandvine 的数据，这代表了 2019 年全球总互联网流量的约 15%。

## Open Connect：一个网络与一个计划

Netflix 于 2011 年启动了 Open Connect 项目，以应对 Netflix 流媒体服务规模的不断扩大。该项目的主要动机有两个：

1. 随着 Netflix 成为消费者互联网服务提供商（ISP）网络流量的一个重要部分，与这些 ISP 直接合作变得至关重要。

2. 为 Netflix 创建一个定制的内容分发解决方案，使他们的工程师能够设计一个主动的、定向缓存解决方案，这比标准的需求驱动型 CDN 更加高效。定向缓存架构通过几个数量级减少了对上游网络带宽的总体需求。

**Netflix 播放过程**

![](https://github.com/user-attachments/assets/239c4cef-9779-440c-8b9d-a9fcf1099d5d)

### 网络

大多数 CDN（内容分发网络）是按需驱动的。这意味着网络缓存的内容及其位置是根据某个地区的请求而定的。对于那些无法预测用户需求的通用 CDN，这种方式效果很好。

由于 Netflix 控制最终用户的应用程序，并且拥有详细的观看趋势数据，它们能够通过转向定向 CDN 获得显著的效率提升。在 Netflix 的定向 CDN 模式中，它们的 Open Connect Appliances（OCA）群体，在观看量非常低的 Fill 窗口期间，每天接收目录更新。

### 计划

Netflix 有一个开放对等互联政策，这意味着它们会与任何同意计划条款的 ISP（互联网服务提供商）进行对等互联。开放对等互联通过本地化流量来改善互联网用户体验。它还具有降低传输成本的优势，对 Netflix、ISP 以及整个互联网都有益。

除了在 Netflix 数据中心和互联网交换点（IXP）安装 OCAs，Netflix 还免费提供 OCA 给符合条件的 ISP，让其直接在 ISP 的网络中安装。这进一步提高了本地化程度，并减少了上游流量¹。有趣的是，这些 OCA 虽然由 Netflix 拥有，但由 ISP 使用，这引发了一些许可方面的考量，最初促使 Open Connect 工程师选择 FreeBSD，因为其宽松的许可证²。

1 详情请见 <https://openconnect.netflix.com/Open-Connect-Overview.pdf>。  
2 <https://www.nginx.com/blog/why-netflix-chose-nginx-as-the-heart-of-its-cdn/>  

## Open Connect Appliances

Open Connect CDN 的主力是 Open Connect Appliances，简称 OCA。这些设备有三种主要配置，运行的是轻度定制的 FreeBSD head（开发）分支。这样的一个庞大且关键的网络使用快速变化的开发分支，乍一看似乎有些冒险。在 2019 年的 FOSDEM 大会上，Netflix 工程经理 Jonathan Looney 解释了使用 FreeBSD head 分支的理由。

首先，Jonathan 和他的团队认为 FreeBSD 的代码通常非常稳定且高质量。其次，他们更倾向于迅速找到并修复那些相对较少且大多数影响较小的 bug。否则，Jonathan 解释说，等待长期（稳定）分支的开发团队可能会陷入他所称之为恶性循环的境地——即合并不频繁，冲突/回归问题多，最终导致功能开发进度变慢。

跟踪 head 分支帮助 Netflix 更快地添加新功能。他们还发现，跟踪 head 分支使得与开发社区中的其他人合作变得更加容易。

![](https://github.com/user-attachments/assets/38ee677e-6c0d-43ef-99e0-a3fb7b86b41d)

- 40Gb/s OCA 存储设备，具有 248TB 存储（2RU 机架形式）  
  - FreeBSD  
  - NGINX  
  - BIRD 互联网路由守护进程

## 吞吐量效率

这些 OCA 有多高效？通过使用 FreeBSD 和商用零件，Netflix 在 1RU 机架中实现了 90 Gb/s 的吞吐量，支持 TLS 加密连接，CPU 使用率约为 55%，配备 Intel 6122 CPU、96GB RAM 和 16TB NVMe 闪存。

因为他们的目标是尽可能多地将代码提交给上游，所以所有 FreeBSD 用户都能受益于帮助 Netflix 实现这种性能的许多增强功能。其中一些贡献包括 NUMA 增强、异步 sendfile、内核 TLS、Pbuf 分配增强、“未映射”mbufs、I/O 调度、TCP 算法和 TCP 日志基础设施。

为了以具有成本效益的方式实现这种性能，Netflix 工程师意识到，他们需要尽可能减少内核与用户空间之间的上下文切换。异步 sendfile 就是帮助实现这一目标的关键技术之一。

新的 sendfile(2) 系统调用实现是对旧版本的替代，它加速了 TCP 数据传输，因为它避免了将文件数据复制到缓冲区再发送。新的 sendfile 通过支持异步 I/O 进一步加速并简化了大数据传输。

新的 sendfile 是 NGINX 和 Netflix 之间开发合作的成果，并在 2016 年 Netflix 服务扩展到近 200 个国家时发布。

![](https://github.com/user-attachments/assets/8a09e035-af1d-42cd-9492-45a74206ca5e)

**Async 服务器**

## 提高效率和隐私保护 — 内核 TLS

为了保护最终用户的隐私，Netflix 在 2016 年加入了传输层安全性（TLS）。Jan Ozer 在他的《Streaming Media》文章中很好地总结了这一举措：

> Netflix 长期以来部署了数字版权管理（DRM）来防止盗版，并通过 HTTPS 保护客户数据在帐户登录和管理过程中的安全。然而，实际的视频数据传输并没有受到保护，因此服务器和客户端之间通信中的任何信息都可能被黑客、网络管理员或 ISP 访问。这些信息可以用来确定观众正在观看哪些内容，甚至其他细节。

添加 TLS 加密效率地要求对 OCA 软件堆栈进行额外的性能增强。这是因为现有的 TLS 技术依赖于 Web 服务器 —— Netflix 的流媒体标准总监 Mark Watson 在 2014 年报告称，这种方法会降低“30-53%”的容量。

解决方案是内核端 TLS，简称 kTLS，它将 TLS 与新的 sendfile 模型结合。这个混合 TLS 方案（由 John Baldwin 在 vBSDCon 2019 会议上描述）将会话管理保留在应用程序空间，并将大规模加密插入到内核中的 sendfile 数据管道。TLS 会话协商和密钥交换消息从 Nginx 传递到 TLS 库，会话状态存储在库的应用程序空间中。一旦 TLS 会话建立，并与客户端交换生成适当的密钥，这些密钥将与客户端的通信套接字关联，并共享到内核中。

![](https://github.com/user-attachments/assets/401261fc-9804-4b88-9388-c837f20574c1)

**传统 TLS Web 服务**

![](https://github.com/user-attachments/assets/46a6b639-454c-4eb4-970f-f89c76de942f)

**内核内 TLS Web 服务**

在他们的 2019 年 EuroBSD 演讲中，Drew Gallatin 和 Slava Shwartsman 展示了 kTLS 如何将带宽性能提升 50 Gb/s，同时减少 CPU 占用率。TLS 性能提升的下一个前沿是所谓的 NIC TLS，其中加密操作由硬件完成。如下面的图表所示，这有望显著减少 CPU 利用率。

![](https://github.com/user-attachments/assets/df46414e-8b5c-46bd-98a5-aa6829c3d180)

内核 TLS 性能 90Gb/s，68% CPU（软件），35% CPU（T6 NIC kTLS）  

原始（~2016）Netflix 100G NVME 闪存设备

**Netflix 使用 TLS 视频服务**

## 达到 200 Gb/s 的 NUMA 优化

随着成员对更多节目和更高分辨率的需求不断增长，Netflix 一直在寻找提高 OCA 吞吐量的方法。随着高核心数系统的发展，团队自 2014 年以来一直在开发和测试非一致性内存架构（NUMA）支持，现在已经开始取得成效。在典型系统中，只有一个 CPU、磁盘和内存，而 NUMA 系统则可以有更多的 CPU、内存和磁盘。与 sendfile 和 TLS 类似，NUMA 也可能会带来吞吐量瓶颈，Netflix 工程师一直在努力将其降到最低。

NUMA 使得 CPU 访问本地资源（例如内存）变得更加便宜，而访问连接到其他节点的资源则变得更加昂贵。因此，内存和 I/O 本地化会影响性能。为了充分利用 NUMA 提供的更高计算密度，Netflix 必须找到一种方法，将尽可能多的磁盘到 CPU 到网络的流量保持在本地节点，并最小化会影响性能的 NUMA 总线传输。为此，Netflix 进行了多项增强，目前这些增强正在逐步合并到上游，包括：

* 为通过 sendfile(9) 发送的文件分配 NUMA 本地内存
* 为内核 TLS 加密缓冲区分配 NUMA 本地内存
* 将连接引导到绑定到本地域的 TCP Pacers 和 kTLS 工作线程
* 通过对监听套接字 `SO_REUSEPORT_LB` 的修改，将传入连接引导到绑定到本地域的 Nginx 工作线程

在测试中，这些增强将 Xeon 性能从 105Gb/s 提高到 191Gb/s，同时将 NUMA 布局利用率从 40% 降低到 13%。对于 AMD EPYC，性能从 68Gb/s 增加到 194Gb/s。

![](https://github.com/user-attachments/assets/22d2396b-2521-4095-9f54-c2b53c3e14ae)

**四节点配置在 AMD EPYC 上很常见**

## FreeBSD 为 NETFLIX 提供三种效率：吞吐量、开发和运营

对于常见问题“为什么选择 FreeBSD？”Jonathan 说他们最初是因为许可证选择 FreeBSD，但最终留了下来是因为效率——Netflix 衡量效率有三种方式：

1. 吞吐量或性能效率，如前一部分所述
2. 开发效率
3. 运营效率

从开发角度来看，与 FreeBSD 社区合作的便利性帮助 Netflix 将其改进向社区上游提交，以便社区进行持续维护。他们还喜欢与其他在相同领域工作的社区成员合作。与这些社区成员共享代码可以改善各方正在开发的代码。

最后，庞大的 OCA 集群需要复杂的监控和操作工具。一些他们所需的工具已经存在，其余的他们自己编写。对于后者，Jonathan 发现 FreeBSD 能很好地提供所需的挂钩，如果没有，团队也能够自己实现。

## Open Connect 智囊团的下一步

除了 NUMA 和持续探索 NIC TLS，团队还在努力将一些改进提交到 kTLS 和 UFS 改进中。

最后，Open Connect 的巨大规模加上团队对效率的关注以及他们对开源的承诺，意味着每个有类似使用案例的 FreeBSD 用户都能获得相同的性能收益。开启 kTLS 并利用异步 Sendfile 的能力，使得任何通过 HTTPS 提供静态内容的用户都能延长硬件寿命、减少密度并更高效地提供卓越的用户体验。

---

**GREG WALLACE** 是一位自由职业的技术营销人员，自 2005 年以来一直与开源软件和社区合作。除了目前在 FreeBSD 基金会的工作外，Greg 还涉及 Kubernetes、安全性、DevOps 和路由等领域。此前，他曾领导 Node.js、ODPi 和 Hyperledger 的营销工作。

