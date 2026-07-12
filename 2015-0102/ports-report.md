# Ports 报告

- 原文：[Ports Report](https://freebsdfoundation.org/our-work/journal/browser-based-edition/mips-and-arm64/)
- 作者：**Frederic Culot**

11 月和 12 月 Ports 树的活动不算多，主要原因是年底的节庆。不过数字仍然可观：Ports 树上有 3,778 次提交！在 bug 方面，关闭的问题报告比上一周期更多，共修复 1,157 个问题。感谢所有提供反馈、花时间让 FreeBSD Ports 更好的人！

## 重要 Ports 更新

为检查重要 Ports 更新是否安全，我们执行了数次 exp-run（实际上是 22 次）。在这些重要更新中，我们提及以下亮点：

- llvm 和 clang 更新至 3.5.0
- xorg-server 更新至 1.14
- gnome 更新至 3.14.2
- enlightenment 更新至 0.19.2
- pkg 更新至 1.4
- 默认 Perl 版本设为 5.18
- 默认 PostgreSQL 版本设为 9.3

照例，如果更新特定 Ports 需要手动步骤，这些步骤会在 **/usr/ports/UPDATING** 文件中清楚说明。强烈建议在对 Ports 树执行任何更新前先检查此文件！

另外请注意，存在多个版本的 Ports，其默认版本在 **/usr/ports/Mk/bsd.default-versions.mk** 文件中设置。如果出于某些原因某个 Port 的默认版本不符合你的需求，可以在 `make.conf` 中加入 `DEFAULT_VERSIONS` 变量来覆盖，如下所示：

- `DEFAULT_VERSIONS=       perl5=5.16 ruby=1.9`

## 新的 Ports 提交者与代管

过去两个月，几位已拥有 src 提交权限的开发者加入我们的社区，并获得了 Ports 提交权限。他们是：jmg@、jmmv@ 和 truckman@。此外，Muhammad Rahman 获得了 Ports 提交权限，将由 marino@ 和 bapt@ 指导。

过去两个月只有一位提交者的权限被代管（motoyuki@），但我们也收到了来自 miwi@ 的遗憾消息，他决定卸下 FreeBSD 的职责，把更多精力放在家庭和职业生活上。除非你过去十年一直与世隔绝，否则你很可能听说过 Martin（miwi@）Wilke。简而言之，Martin 是向 Ports 树贡献提交数量最多的开发者，自 2006 年以来超过 20,000 次提交！

## 年度数据

2014 年是我们 Ports 树历史上提交数量最多的一年！此前我们从未超过 30,000 次提交，而 2014 年几乎达到 37,500 次 [https://people.freebsd.org/~eadler/datum/ports/commits_by_year.png]。因此我们借此机会感谢所有开发者和贡献者，并希望在 2015 年也能看到这样的奉献！

## 新的 portmgr lurker

每四个月，Ports 管理团队都会迎来一对新的 lurker，即两位 Ports 提交者，他们有机会在更高层面贡献、了解 portmgr@ 的内部运作，并分担工作量。这两位 lurker 加入 portmgr@ 邮件列表，可以接触到机密通信。我们鼓励他们参与所有讨论，并对 portmgr@ 的决策结果发表意见。

新一届任期于 11 月开始，我们的两位 lurker 是 ak@ 和 sunpoet@。按照传统，他们受邀回答一份问卷，以便大家更好地了解他们。以下是 Alex <http://blogs.freebsdish.org/portmgr/2014/11/04/getting-to-know-your-portmgr-lurker-ak/> 和 Po-Chuan <http://blogs.freebsdish.org/portmgr/2014/12/03/getting-to-know-your-portmgr-lurker-sunpoet/> 回答的问卷链接。

---

**Frederic Culot** 在 IT 行业工作了十年。业余时间他学习商业与管理，刚完成 MBA 学业。Frederic 于 2010 年作为 Ports 提交者加入 FreeBSD，至今已有约 2,000 次提交，指导了六位新提交者，现在担任 portmgr-secretary。
