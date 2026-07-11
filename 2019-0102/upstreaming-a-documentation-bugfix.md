# 向上游提交一处文档 bugfix

- 原文链接：[Upstreaming: a Document Bugfix](https://freebsdfoundation.org/our-work/journal/browser-based-edition/)
- 作者：**BENEDICT REUSCHLING**

FreeBSD 文档集由大量文档、手册页、网站组成。这些都是新人参与社区、为操作系统回馈贡献的好途径。已有现成补丁可供文档提交者审查时，变更通过 Phabricator 提交。其它报告 bug 的方式还有邮件列表（<freebsd-doc@freebsd.org>）和 Bugzilla。

最近有这样一个 bug：<https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=231994>。描述不多，但关键词正中我下怀。主题行写着“sudoers repeated word”，意味着它属于 security/sudo Port 这个广泛使用的软件。重复词是人们匆忙编写文档且不校对（或没有第二双眼睛帮着校对）时常见的错误。我做的第一件事，是确认报告的错误在 sudoers 文件的当前最新版本中确实仍然存在。有时人们报告的 bug 在 Port 的 HEAD 版本中已修复，仅因他们没有更新到最新版本，或者在此期间已有人纠正了错误。本例中，sudoers 手册页里确实存在重复的“and and”。

由于 FreeBSD 本身并不维护 sudo 软件（除了一些本地补丁），正确的处理方式不是仅在我们自己的项目中本地修复，而是向上游报告问题。这样，sudo 项目不仅能知晓该问题，还能着手解决。问题修复并发布更正版本后，其它所有项目（包括 FreeBSD）在安装较新版本时都会收到这些修复。其影响范围远超仅在本地修复问题。

sudo 项目位于 <https://www.sudo.ws/>，同样有一个 bugzilla 实例供人们报告 bug。在那里创建账号很简单，按惯例通过注册确认邮件确认账号后，我就能创建 issue。在创建过程中，我想到重复词很少单独出现，于是又查找了手册页中的其它实例。简要搜索后发现了其它重复。然后我意识到，仅凭人工阅读手册页可能会漏掉某些实例。

几年前，Warren Block 编写了一款基于 perl 的工具 igor——“友好的实验室助手”（<http://www.wonkity.com/~wblock/igor/>），可对手册页与 DocBook XML 页面运行，检查各种问题。其中一项检查就是重复词，幸运的是该工具足够灵活，能在 FreeBSD 之外使用。于是我对 sudoers 手册页运行了 igor，它报告了重复词之外的其它问题。

由于愈发怀疑 sudo 项目其它手册页可能也存在类似问题，我对它们也运行了 igor，发现了若干其它问题。sudo 项目维护多种手册页格式，语法略有不同，因此我需要对它们逐个修改。手册页改动的妙处在于，通过运行 **man(1)** 程序并将改动后的手册页作为参数传入即可立即查看结果。无需冗长编译，能快速得到改动反馈。

修复 igor 报告的所有问题（包括最初引我们注意的那个）后，我用 `diff -ruN` 创建了补丁。补丁附在 sudo bugzilla 的 issue <https://bugzilla.sudo.ws/show_bug.cgi?id=854> 上，并附有描述与 igor 工具链接。

不久，Todd Miller 便接受了补丁并将其集成（<https://www.sudo.ws/repos/sudo/rev/4ddcb625f3b7>）。他对 igor 也产生了兴趣，并在 issue 中写道：“感谢，我已在 doc Makefile 中添加了一个‘igor’目标，会修复它所发现看起来像问题的内容。”这意味着今后每当 sudo 出现文档变更，igor 都会作为手册页流程的一部分运行并可能报告问题。这样，质量问题可在进入 src 仓库之前修复，不会随下次发布扩散出去。

与此同时，sudo 1.8.26 新版本发布，其中包含了文档修复，并甚至在发行说明中提及（<https://www.sudo.ws/stable.html#1.8.26>）。FreeBSD Port 在此不久后也得到更新（<https://svnweb.freebsd.org/ports?view=revision&revision=484929>），这些修复也随之作为 FreeBSD 的一部分可用。就这样，从一个使用 sudo 的下游项目所报告的小错误，演变为针对多个文档问题在上游报告并修复的较大补丁。开源项目中这类事情无时无刻不在发生，不仅限于文档。src 与 Ports 也是如此，这是协作并交换修复、工具与思想使所有人受益的绝佳范例。妙处在于从文档工作入手很容易，尤其是手册页。小修复与大改进同样有价值，正如你所见，可能引发比初看时更大的事。

有意开始文档工作的人不妨一读 FreeBSD Documentation Project primer（<https://www.freebsd.org/doc/en_US.ISO8859-1/books/fdp-primer/>）。快速入门章节解释了在工具与获取源码方面所需的一切。准备就绪后，便可在文档中开始猎虫。使用 FreeBSD 的 pkg 工具安装 textproc/igor Port 非常简单。找到 bug 后，请确认它尚未被报告或修复。如未报告，务必报告给实际维护它的项目与人员。许多项目都有 bug 跟踪系统或可报告问题的邮件列表。请尽量提供详尽信息与清晰描述，并附上你已创建的所有补丁。这样能提高补丁获得审视与处理的机会。对 FreeBSD 文档流程有任何疑问，可在 freebsd-doc 邮件列表提问，或到 Efnet IRC 的 #bsddocs 频道找我们。

---

**BENEDICT REUSCHLING** 于 2009 年加入 FreeBSD 项目。2010 年获得完整的文档提交权限后，他积极开始指导他人成为 FreeBSD 提交者。他是 BSD 认证小组的考官，并于 2015 年加入 FreeBSD 基金会，目前担任副总裁。Benedict 拥有计算机科学硕士学位，在德国达姆施塔特应用技术大学讲授面向软件开发者的 UNIX 课程。
