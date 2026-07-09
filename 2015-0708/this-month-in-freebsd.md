# FreeBSD 本月动态

- 原文：[This Month in FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-in-the-enterprise/)
- 作者：**Dru Lavigne**

7 月 24–26 日，我参加了在德国埃森 Linuxhotel 举办的 FreeBSD 黑客马拉松。这次活动是一次实验，将一群 FreeBSD 提交者聚集在一起，在轻松环境中周末编码。之后，我采访了 LARS ENGELS（lme@），黑客马拉松的组织者之一，了解他对这次活动的看法。

**问：** 请介绍一下你自己。你是如何开始接触 FreeBSD 的？你在 FreeBSD 项目中参与哪些工作？

**答：** 折腾过几个 Linux 发行版后，我发现了 FreeBSD 4.7 或 4.8。借助出色的 FreeBSD Handbook 和德国 BSD 邮件列表，我安装了 FreeBSD 并学到了很多。不久后，我把同一块硬盘上的 Windows 安装搞坏了，又懒得修，所以就一直用 FreeBSD，它满足了我桌面使用的大部分需求。从那以后，我大部分时间在笔记本、台式机和服务器上使用它。当我在 Windows 上使用的一些程序（主要是 Java 程序）不属于 FreeBSD Ports 树时，我开始深入 Ports 系统，创建了我的第一批 Ports。空闲时间多了，我开始寻找未维护但有新上游版本的 Ports，更新它们并接手维护。2007 年，Martin Wilke（miwi@）问我是否愿意成为 Ports 提交者，我愉快地接受了。

此外，我还是 FreeBSD 论坛管理团队的成员，做了一些琐碎修复，将 Handbook 的一章翻译成德语，你经常可以在 FOSDEM、FrOSCon 等活动的 FreeBSD 展位找到我。

**问：** DevSummit 在 BSD 大会上已经举办好几年了。是什么促使你组织一个更小型的区域性活动？你的目标是什么？你认为达成了这些目标吗？

**答：** 我知道 Linuxhotel 以非常合理的价格承办社区活动，认为它是举办 DevSummit 的绝佳场所。当我和 Benedict Reuschling（bcr@）谈起这件事时，他主动提出帮我组织活动，并建议举办黑客马拉松而非正式的 DevSummit。我们都很喜欢这样的活动理念：专注于代码或文档工作，而非演示和演讲。虽然后者很重要，但 BSD 大会期间已经有几个传统 DevSummit，而 FreeBSD 黑客马拉松却很罕见。

我们没有特定主题或共同目标。参与者为周末设定自己的目标，据我所知，大多数目标都达成了。

**问：** 对于有兴趣在自己所在地区组织黑客马拉松的读者，需要哪些步骤？需要多少时间？

**答：** 实际上工作量没我想象的那么多。有了这个想法后，我在 FreeBSD wiki 上找到了一份有用的 DevSummit 组织指南。其中开篇列出的“黄金法则”非常正确：找到合适的场地，在活动前大约六个月开始组织，在邀请和提醒方面要持之以恒。向 FreeBSD 开发者邮件列表发送邮件、联系本地 BSD 和 Linux 用户组是寻找感兴趣的人的好方法。务必发送提醒，让犹豫不决的人还有机会考虑是否参加。

确定参加人数后，就可以在餐厅组织食物和桌位。还要考虑晚上的社交活动。

**问：** 参加黑客马拉松的有哪类人？活动期间日程如何安排？

**答：** 我们既有本地人，也有来自美国、加拿大、瑞典、比利时和德国其他地区的与会者。所有与会者都是现任或前任 FreeBSD 提交者。他们的工作领域涵盖 FreeBSD 的所有部分：文档、src 和 Ports，还有一位 OpenBSM 项目的开发者。

第一天，人们在下午晚些时候到深夜之间陆续到达，我们在公园碰面，享受烧烤，并处理一些文档、Ports 和 src 的 bug。

周六狂风暴雨，所以我们整天待在酒店里。Sean Chittenden 做了一场关于 DTrace 和 Brendan Gregg 火焰图的有趣介绍，他在工作中用它来发现公司软件中的瓶颈。Kristof Provost 讲解了他如何在 TP-Link 路由器上刷入 FreeBSD、如何配置相当复杂的无线设置。酒店的无线网络正是由同样的 TP-Link 路由器提供的，但我们没有尝试用 FreeBSD 替换 Linux。后来，我们去了附近的一家餐厅，吃到了美味的炸肉排。当天余下的时间我们在酒店的壁炉间编码。

周日温暖晴朗，我们整天在公园里，处理各种事情，午餐又来了一次烧烤。傍晚时分，每个人都离开了，并收到一个装有糖果的“FreeBSD Hackathon 2015”包作为礼物。

整个周末我们向文档、src 和 Ports 仓库提交了 50 次，向 OpenBSM GitHub 仓库提交了 4 次，还将 BSMtrace 仓库从 Perforce 迁移到 GitHub。

**问：** 请介绍一下 Linuxhotel 和你是如何知道这个场地的。

**答：** Linuxhotel 是一栋带周边公园的别墅，用作开源操作系统的培训设施，包括 Linux、FreeBSD、OpenBSD、Tomcat 和 MySQL 等软件，还有 Perl、C 和 Java 等语言课程。几年前，我兄弟在那里教授 Apache 课程，这让我知道了它。后来，我问 Linuxhotel 的老板是否愿意让我开设 FreeBSD 入门课程，他喜欢这个想法。所以现在该课程每半年开设一次。除此之外，Benedict 和我正计划开发一门 OpenZFS 课程。

**问：** 你认为明年还会组织这样的活动吗？经历了整个过程，有什么你会做得不同？

**答：** 今年的黑客马拉松是一次非常好的活动，希望与会者玩得开心并希望明年再来。这是我们第一次组织这样的活动，所以并非一切都如我们希望的那样顺利。但我们在黑客马拉松后进行的调查显示，人们大多对活动感到满意。

下次我们应该尝试吸引更多非开发者，也许还有其他 BSD 的人。我们还可以鼓励人们就自己正在做的事情做简短的演讲，然后一起协作。另一个想法是设一个“每日主题”或围绕单一主题运行整个活动。•

**Dru Lavigne** 是 FreeBSD 基金会理事、BSD 认证组主席。

以下是本文提到的资源链接：

- <https://wiki.freebsd.org/201507DevSummit>
- <http://www.linuxhotel.de/>
- <https://fosdem.org/2016/>
- <http://www.froscon.de>
- <https://wiki.freebsd.org/DevSummitHowTo>
- <http://www.trustedbsd.org/openbsm.html>
- <http://www.brendangregg.com/flamegraphs.html>
- <http://www.trustedbsd.org/bsmtrace.html>
- <http://www.linuxhotel.de/kurs/freebsd/>
