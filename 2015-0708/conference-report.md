# 会议报告

- 原文：[Conference Report](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-in-the-enterprise/)
- 作者：**SHONALI BALAKRISHNA**

BSDCan 2015（<www.BSDCan.org/2015/>）

BSDCan 大会开幕演讲中的一张幻灯片写道：“一个房间里聚集了这么多聪明人。这就是我参加 BSD 大会的原因。”我会心一笑，开始沉浸于我第一次 BSD 大会的兴奋之中。在听完九场演讲并与上述那些聪明人多次交流之后，我彻底被说服了！BSD 大会不仅把太多太多非常聪明的人聚到了一个房间里，他们还是最和善的一群人。你常常听到 FreeBSD 社区所做的出色工作，但无论怎么说，FreeBSD 社区令人难以置信，它所带来的启发再多赞誉也不为过。

我在研究生院令人筋疲力尽的期末考试周之后的第一天抵达预定的演讲日程。我无疑是睡眠不足的，但大会的兴奋和活力让我完全清醒。我发现自己参加的九场演讲每一场都很有趣。在全体大会上，听 THE Stephen Bourne 讲述 <https://www.youtube.com/watch?v=2kEJoWfobpA> Bourne shell 的诞生过程、他的诸多洞见和内幕故事，真是令人难以置信。我喜欢他讲述如何说服 Dennis Ritchie 将 `void` 类型引入 C 语言那一段，也很欣赏他对 shell 脚本调试的坦率。

我尤其喜欢以下演讲：Limelight Networks 的 FreeBSD 运维——内容分发网络（CDN）引起我的兴趣，今年在研究生院我有一个相关的研究课题，从运维角度、从他们使用 FreeBSD 的方式来看，非常有趣。他们部署了自己的骨干网，并在骨干网上使用 FreeBSD。对于一个拥有许多数据中心的大型 CDN 来说，在边缘大规模部署 FreeBSD 是一个重要因素。这样的运维工作负载要求软件和配置变更的流畅性——而 Limelight Networks 在设计和运维上参与的人数相比通常模式要少得多。使用 FreeBSD 能同时满足这两点，因为你可以被拉入源代码树，并参与系统的运维。在部署 FreeBSD 时，他们的策略包括向上游贡献一切、使用 Ports、组建 src 团队。演讲讨论了 Limelight 使用的工具——用 Zabbix 监控以确保 API 驱动的配置管理，在测试、开发和 QA 环境中监控以形成高效的反馈回路，用 OpenTSDB 存储时间序列指标，用 SaltStack 以声明式风格进行配置管理（系统根据策略通过编排总线采取行动）、用 Vagrant 将 FreeBSD 推送到边缘服务器。

Ed Schouten 的 CloudABI 是一种 Unix 应用二进制接口，提供基于能力的（capability-based）安全。它扩展了 Capsicum 的概念，通过仅约 60 个系统调用提供更紧凑的表示。这可以应用于云计算环境，替代使用完整的系统虚拟化或虚拟化命名空间。CloudABI 能以远为简单的方式使用，性能更佳且无需复杂配置，同时通过控制对套接字、文件和其他资源的访问来提供安全。它提供了一种全新的创建环境的方式，基于能力的安全在进程级别提供隔离和资源访问控制，而非通过完全隔离环境来提供所需的安全级别。

Kirk McKusick 的《An Introduction to the Implementation of ZFS》是一场有趣的演讲，原因不止于这是 Kirk McKusick 本人在讲 ZFS。演讲中间还响起了火警，紧接着消防车到达！我喜欢这场演讲，McKusick 博士清晰地讲解了 ZFS 块结构和检查点机制，让理解文件系统和快照块的释放这类相当复杂的概念变得容易。Dan Langille 主持的闭幕环节非常有趣，BSD 社区的精彩再次大放异彩。我参加了在附近的 Lowertown Brewery 举办的闭幕派对，令人愉快，也是拓展人脉的好机会。

很高兴终于见到了 Gavin Atkinson，他是去年夏天我们在 FreeBSD 的所有 GSOC 学生的管理员。我还与 Diane Bruce、Ed Schouten、Deb Goodkin、Justin Gibbs 和 David Chisnall 等人交谈，每一次对话都令人耳目一新，从不同角度呈现 FreeBSD 所做的工作、FreeBSD 中的具体项目、通用技术、代码库、如何为 FreeBSD 做贡献、历史、音乐等无数主题。

至于我未来想为 FreeBSD 做的工作，除了收尾 BSNMP IPv6 项目的测试之外，我还了解了其他项目，如进一步支持 IPv6（用户空间清理、统一 ping 和 ping6、统一 traceroute 和 traceroute6）、改进蓝牙模块、802.11 改进、空间通信协议。

渥太华是一座值得造访的不可思议的城市——历史悠久、文化丰富——我沉浸在它提供的博物馆、艺术馆和美丽的徒步小径之中。尽管我到达时精神疲惫，但回家时满怀兴奋和启发，对未来的 FreeBSD 贡献有许多计划，也非常感激 FreeBSD 基金会为我提供这次参加第一次（希望是更多次中的第一次）BSDCan 大会的机会。

**SHONALI BALAKRISHNA** 是 2014 年 FreeBSD 项目的 谷歌 Summer of Code 学生，为 BSNMP 项目做出贡献，致力于为 BSNMP 添加 IPv6 支持。她目前是加州大学欧文分校网络系统专业的硕士研究生。Shonali 拥有电信工程学士学位，本科研究和论文带来两篇研究论文发表。她目前的研究兴趣位于分布式系统和机器学习的交叉点，她也希望继续为 FreeBSD 贡献代码。
