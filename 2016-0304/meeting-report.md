# 会议报道：台北/台湾 PortsCamp

- 原文：[Meeting Report: PortsCamp in Taipei/Taiwan](https://freebsdfoundation.org/wp-content/uploads/2016/04/Meeting-Report.pdf)
- 作者：**Marcelo Araujo**

2016 年 1 月 15 日

这是在台北 HackerSpace 举办的首届 PortsCamp，共有 27 名与会者，其中包括 5 名 FreeBSD 提交者：araujo@、lwhsu@、sunpoet@、kevlo@ 和 ijiliao@。活动有一家赞助商 Gandi，慷慨提供了场地、饮品和披萨。我制作了 FreeBSD 贴纸，与所有人分享。

活动安排了两场演讲：第一场由 Gandi 亚洲子公司总经理 Thomas Kuiper 介绍 Gandi 如何在其云平台中使用 FreeBSD。他还向任何想要在 Gandi 的 IaaS 环境中试用 FreeBSD 镜像的人提供了试用账户（Gandi：http://.www.gandi.net）。

我做了第二场演讲，对 FreeBSD 项目做了概览介绍，并讲解如何贡献代码、成为 FreeBSD 提交者。我的演讲幻灯片位于 <http://www.slideshare.net/araujobsd/portscamp-taiwan>。

我还提供了一台运行 bhyve(8) 的服务器，预装最新的 FreeBSD-HEAD、Ports 树和 poudriere(8)，与会者可以通过 ssh 访问并动手实验 FreeBSD Ports。提供这台服务器是个非常好的主意——环境开箱即用，节省了大量配置时间。这台服务器是 i7 处理器配 16G 内存，托管了 25 个虚拟机，性能相当不错。

我们在 Ports 树中进行了 193 次提交，历史日志标记为：Sponsored by: PortsCamp Taiwan。

与会者把大部分时间花在理解如何使用 poudriere(8) 上。一些人已经在做项目，例如有人测试 glusterfs 补丁，还有一位牙医在做文档中文化工作。所有提交者都很热心，把大部分时间用来回答问题和与大家交流。

我们聚得很顺利，并计划在台湾另一座城市（新竹）举办第二届 PortsCamp。可能会在今年 3 月 AsiaBSDCon 之后紧接着在台湾大学举办。我们可能把 PortsCamp 改为工作组形式，相信这样更具生产力。

我要特别感谢 Mose 在 PortsCamp 前和期间提供的大力支持。

以下是关于 PortsCamp 的其他博客和帖子：

- kevlo：<http://wp.me/p1J5gU-2v>
- miwi：<http://miwi.cc/2016/01/aiwan-portscamp/>

---

**Marcelo Araujo** 是一名 FreeBSD 开发者，主要工作于 Ports，近期也参与内核相关特性。他是生活在台湾的巴西人；在自由软件世界之外，Marcelo 喜欢摩托车、徒步和啤酒。
