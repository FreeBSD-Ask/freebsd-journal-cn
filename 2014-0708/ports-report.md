# Ports 报告

- 原文：[Ports Report](https://freebsdfoundation.org/our-work/journal/browser-based-edition/virtualization/ports-report/)
- 作者：**Frederic Culot**

过去两个月 Ports 树上的活动相当多！好消息也不少——新的提交者加入项目，还有令人兴奋的新工具来减轻我们在 ports 上的工作量。

## 新的 Ports 提交者

我们当中一些提交者还记得收到提交权限（commit bit）的那一天是多么特别。我起初感到自己充满力量，因为可以登录到 FreeBSD 官方主机，但很快这种心情转为焦虑，因为常言道“权力越大，责任越大”！现在轮到两位新秀加入——Bartek Rutkowski 和 Stephen Hurd。希望他们和我一样，在 FreeBSD 上的工作中乐在其中！

另一则好消息是 Stefan Esser 在繁忙时期无法参与项目之后，请求恢复他的提交权限。我们都为他的回归感到高兴。可能有人不知道，提交权限在不活跃 18 个月之后会被妥善保管。当不活跃的提交者准备回来时，他的提交权限会被恢复，通常会指派一位导师负责引导，并解释 ports 树上最近应用的变更。

## FreeBSD Ports 管理团队的变更

Ports 管理团队（portmgr）也发生了一些重大变化——有喜讯，也有坏消息。

- 先说喜讯：Steve Wills 被授予 portmgr 团队的正式成员资格。Steve 在 ports 上做了大量工作，尤其是在测试领域。Ports 管理员认为赋予 Steve 完整投票权是合理的回报，吸纳他加入团队也将有助于推进 ports 测试领域的未来改进。
- 现在是坏消息：我们敬爱的加拿大人 Thomas Abthorpe 决定辞去 portmgr 秘书的职务。虽然我怀疑这私下里是因为他储备的加拿大笑话已经枯竭，但官方原因是 Thomas 想把更多精力放在他的私人生活和职业生涯上。毫无疑问，整个 ports 社区都在哀悼。

更糟糕的是，我被忽悠去接替 Thomas，成了新的 portmgr 秘书。Tabthorpe 总说一个加拿大人至少抵得上四个法国人，所以这里我远未达到标准，但我会尽我所能为 ports 社区服务。

## 跟踪 Bug 等的新工具

过去两个月带来了一些令人兴奋的新工具，用于跟踪 Bug 和简化代码评审流程。Bug 跟踪方面，关于脱离 GNATS 的必要性的讨论其实早在 10 年前就开始了！但切换最终才完成，我们现在依赖 Bugzilla（<https://bugs.freebsd.org/bugzilla/>）。这次过渡仍有少数边角情况需要打磨，但开发者对切换到更现代的 Bug 跟踪器总体感到满意。尚未熟悉 FreeBSD 问题报告管理方式的读者，建议从 <http://www.freebsd.org/support/bugreports.html> 入手，该页面提供了有用的链接。

至于代码评审，ports 提交者目前正试用 Phabricator（<https://phabric.freebsd.org/>）来简化代码评审流程。这对 FreeBSD 来说是相当大的变化，因为此前并未使用专门的工具，但请记住目前尚无政策要求评审必须通过 Phabricator 完成。我们仍在试验这一新流程，如果你有兴趣和我们一起尝试，建议从该页面入手：<https://wiki.freebsd.org/CodeReview>。

## 20 周年纪念

为给本专栏收尾，我想提及一件重要事件：FreeBSD Ports 树的 20 周年纪念！这一切始于 1994 年 8 月 21 日，Jordan Hubbard 做了如下提交：

> Commit my new ports make macros. Still not 100% complete yet by any means but fairly usable at this stage.

20 年后，这套宏集合似乎仍然相当好用。从一开始，已有超过 500 名提交者参与该项目，产生了超过 300,000 次提交，让近 25,000 个 ports 保持最新。衷心感谢所有奉献空闲时间和精力促成这一切的人，我热切期待 20 年后我们将走向何方。与此同时，欲了解此纪念日的惊喜，请关注我们的博客（<http://blogs.freebsdish.org/portmgr/>）和社交网络页面：

- <http://fb.me/portmgr>
- <https://twitter.com/freebsd_portmgr>
- <https://plus.google.com/u/0/communities/108335846196454338383>

---

**Frederic Culot** 完成了物理计算机建模方向的博士学位后，过去 10 年在 IT 行业工作。业余时间他学习商业与管理，并刚刚完成 MBA。Frederic 于 2010 年作为 ports 提交者加入 FreeBSD，至今已做出约 2,000 次提交，指导过 6 位新提交者，现在承担 portmgr 秘书的职责。
