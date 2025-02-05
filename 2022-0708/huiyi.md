# FreeBSD 开发者峰会报告

- 原文链接：[FreeBSD Developer Summit Report](https://freebsdfoundation.org/wp-content/uploads/2022/08/developer_summit_report.pdf)
- 作者：**ANNE DICKISON**

每年夏天，FreeBSD 开发者社区的成员及其嘉宾都会聚集在 FreeBSD 开发者峰会上。规划委员会从 2022 年初开始筹备，怀着能将大家 —— 现场见面 —— 连接在一起的美好愿望。然而，不幸的是，疫情改变了计划，我们很快意识到又将举行一次虚拟活动。委员会成员共同努力，招募讲者和工作小组，并寻找新的方式让虚拟活动显得更加个性化。首先，增加了一个扩展的走廊环节。通过使用 SpatialChat 虚拟会议服务，参与者可以在虚拟房间里四处游走，与周围的人交谈。我们还将休息时间延长至 30 分钟，以便参与者之间有更多的交流时间。开发者峰会由 FreeBSD 基金会赞助，于 6 月 16 日和 17 日举行。活动全程录制并通过 YouTube 直播。你可以在[这里](https://youtube.com/playlist?list=PLugwS7L7NMXwVfBq5eDRky450jp7LTRJj)找到完整的内容和单个讲座。幻灯片可以在 [Wiki](https://wiki.freebsd.org/DevSummit/202206) 上找到。

第一天以我们规划委员会的领导和长期主持人 John Baldwin 的欢迎致辞拉开帷幕。欢迎之后，FreeBSD 基金会的 Deb Goodkin、Ed Maste 和 Joseph Migrone 更新了基金会的最新动态，介绍了技术路线图的进展，并分享了接下来一年大家可以期待的事项。

在第一次休息之后，走廊环节还包括了一段 Rick Roll 视频，接着 Brooks Davis 更新了令人兴奋的 CHERI/Morello 项目。他的讲座讨论了 CHERI 是什么，ARM 的 Morello CPU 是什么，以及这些项目对 FreeBSD 的影响。

经过 SpatialChat 平台上的另一个热烈休息之后，Ed Maste 和 Warner Losh 主持了一场关于 FreeBSD Pre-commit CI 的圆桌讨论。该环节的目标是解释为什么 Pre-commit CI 对 FreeBSD 很重要，当前的实施情况，以及鼓励其他开发者加入正在增强 Pre-commit CI 过程的团队。

接下来是 30 分钟的休息，Pre-commit CI 话题继续在走廊环节中展开。随后，Mark Johnston、Mariusz Zaborski 和 Ed Maste 主持了一个关于 FreeBSD 安全性的面板讨论。在会议期间，他们讨论了 Sec Team 以及随着时间推移发生的一些变化，未来可能会有哪些变化。他们还回顾了问题是如何被发现和处理的，包括对 FreeBSD 添加的漏洞缓解措施。最后，他们讨论了一些提升安全性的积极措施。

为了确保日程按时进行，接着进行了 20 分钟的休息。然后我们进入了当天的最后一个环节。由于开发者和供应商峰会已经转为在线形式，委员会通常将第一天的最后一个环节保留为关于 FreeBSD 历史的特别炉边谈话。以往的峰会包括了 Kirk McKusick 和 Warner Losh 的演讲，可以在项目的 YouTube 频道上找到。今年，Jordan Hubbard 加入了我们，讲述了 FreeBSD 的早期历史。这个环节更多的是个问答环节，而不是正式的演讲，可能是我最喜欢的部分。Jordan 分享了很多精彩的故事，我也很羡慕他能如此详细地回忆过去。如果你对旧时的 FreeBSD 感兴趣，我强烈推荐观看 Jordan 的演讲。

第二天的峰会在我们的无畏领导者 John Baldwin 的再次欢迎中开始，然后直奔可能是峰会中最忙碌却最富有成效的部分——有、需求和计划环节，也就是 14.0 规划。为了尽可能多地覆盖内容，规划环节分为两部分，中间有一个休息。讨论的更新 HackMD 页面可以在这里找到。

经过一段急需的休息后，Warner 回来主持了一个关于他们使用的 QEMU BSD 用户模拟器的环节，讨论了它如何用于构建包以及它的上游进展。Warner 最后呼吁大家帮助审查补丁、重构、提交系统调用并修复错误。查看他的演讲，了解如何提供帮助。

当天的最后一个休息后，进行了一个关于 Linux 专业学院（LPI）BSD 认证课程的环节。LPI 的 Fabian Thorns 讲解了 BSD 认证考试，以及 LPI 的使命和新的会员计划。他还呼吁大家帮助创建与 BSD 认证考试相关的培训材料。一定要查看 Fabian 的完整演讲，了解你如何能从 LPI 中受益或提供帮助。

峰会最后由 John Baldwin 做了闭幕演讲，讨论了即将到来的 FreeBSD 活动，并提醒大家填写峰会调查。作为一个参与组织过不少会议和活动的人，我可以告诉你，活动后的调查对规划委员会极为重要。它帮助我们了解哪些方面做得好，哪些不行，也帮助我们决定未来的方向。委员会已经开始着手规划 2020 年秋季的 FreeBSD 供应商峰会。有关更多信息应很快发布。希望我们最终能亲自见面。

感谢每一位参加 2022 年 6 月 FreeBSD 开发者峰会的人。我们期待着在今年晚些时候以某种方式见到你们。

---

**ANNE DICKISON** 于 2015 年加入基金会，拥有 20 余年的技术营销和传播经验。特别是她在 USENIX 协会担任市场总监和联合执行董事期间，帮助她深刻理解了宣传自由和开源技术重要性的承诺。
