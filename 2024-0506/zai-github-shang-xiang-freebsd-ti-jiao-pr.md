# 在 GitHub 上向 FreeBSD 提交 PR

- 作者：**WARNER LOSH**
- 原文链接：<https://freebsdfoundation.org/our-work/journal/browser-based-edition/configuration-management-2/submitting-github-pull-requests-to-freebsd/>


为了让人们更易贡献，FreeBSD 项目在最近支持了 GitHub 上的拉取请求（pull request，PR）。我们发现使用缺陷跟踪器 Bugzilla 接受补丁会造成许多有用的贡献被搁置直至过期。因此我们建议贡献者首选 GitHub PR 来进行修改，仅把 Bug 放在 Bugzilla。虽然 Phabricator 对开发人员很有用，但我们也发现，在 Phabricator 极易失去外部贡献者的贡献。除非你和直接让你用 Phabricator 的 FreeBSD 开发人员一起工作，否则还是建议用 GitHub。GitHub PR 更易跟踪和处理，并且大部分开源社区都对此更为熟悉。我们希望有更快的决策，更少的提交被放弃，能为所有人提供更佳的体验。

由于 FreeBSD 的志愿者时间有限，本项目制定了标准、规范和政策，以合理规划志愿者的时间。你需要了解这些内容才能提交好的 PR。我们有一些自动化程序来帮助提交者修复常见错误，从而使志愿者可以审查几乎就绪的提交。请理解我们仅接受最有用的贡献，而某些贡献我们无法接受。

接下来，我将介绍：如何将你的提交切换为 Git 分支，如何完善它们以符合 FreeBSD 项目的标准和规范，如何从你的分支创建 PR，以及对审查流程的预估。然后，我将为志愿者说明如何评估 PR，并提供完善 PR 的技巧。

本文关注于基本系统的提交，不涉及文档和 Ports。这些团队仍在修订有关这些存储库的细节。

## FreeBSD 项目标准

FreeBSD 项目对 FreeBSD 系统的各方面均有详细的标准。这些标准在 [FreeBSD 开发者手册](https://docs.freebsd.org/en/books/developers-handbook/)和 [FreeBSD 提交者指南](https://docs.freebsd.org/en/articles/committers-guide/)中有所说明。代码规范在 FreeBSD 手册页中有所描述。根据惯例，手册页被分为多个部分。出于历史原因，所有风格手册页都在第 9 部分。对手册页的引用通常呈现为页面名称，后跟其部分编号在括号中，例如 style(9)、cat(1)。这些文档可以在所有的 FreeBSD 系统上使用 man 命令获取，亦可在线浏览。

FreeBSD 项目致力于创建文档齐全的集成系统，涉及控制机器的内核以及常见 Unix 工具于用户空间之实现。提交应写得清晰，且包含相关评论（comment）。当行为发生变化，应同步更新相关手册页。例如，当你向命令添加了参数时，也应同时将其添加到手册页上。当库中添加新功能时，应同事把这些功能添加新的 man 页。最后，FreeBSD 项目认为源代码控制系统中的元数据也是系统的一部分，因此提交信息也应符合 FreeBSD 项目的标准。

FreeBSD 项目对 C 和 C++ 的代码规范在 style(9) 中有所说明。这种风格通常被称为“内核规范形式（Kernel Normal Form，KNF）”，即采用了 Kernighan & Ritchie 的《C 程序设计语言》中使用的风格。这是研究 unix（research unix）使用的标准，后来在伯克利的 CSRG（Computer Systems Research Group，计算机系统研究小组）中沿用，进而催生了 BSD 发行版。FreeBSD 项目在这些实践的基础上进行了现代化。这是提交代码的首选风格，且 FreeBSD 系统中大多数代码使用的风格亦如此。有关这些代码的变更贡献应遵循此风格，但某些文件有自己独特的风格。Lua 和 Makefile 也有各自的标准：能在 style.lua(9)、style.Makefile(9) 中找到。

提交信息（Commit messages）采用正使用着 git 的开源社区的通行形式。第一行应概述整个提交，字数在 50 个字符以内。其余部分的提交信息应陈述提交内容及原因。如果改动是显而易见的，最好只解释原因。每行字数不超过 72 个字符。应使用一般现在时，采用祈使语气书写。结尾部分包含一系列 Git 称为“trailers（预告片）”的行，项目用这些行来跟踪有关提交的附加数据，例如提交的来源、与 bug 相关的详细信息等。有关提交日志消息的详细说明，请参阅提交者指南的“[提交日志消息](https://docs.freebsd.org/en/articles/committers-guide/#commit-log-message)”部分。


## 无法接受的提交

在实验性地使用 GitHub 接受提交几年以后，FreeBSD 项目不得不引入一些限制，对那些尚未获得 FreeBSD 项目存储库写入权限的人员，须确保他们使用 GitHub 进行的修改控制在合理范围内。这些限制确保了验证和应用修改的志愿者能够最有效地分配他们的时间。因此，FreeBSD 项目无法接受如下内容：

* 在 GitHub 上无法审核过大的变更
* 注释中的拼写错误
* 运行静态分析程序时发现的更改（除非这些更改包含了针对静态分析程序发现的错误的新测试用例）。对于与我们的测试线束交互不畅的系统部分“显然正确”的修复，可以根据具体情况作出例外处理。
* 理论性的变化，但没有具体错误或者可描述的行为缺陷。
* 没有配备前后对比度量以显示改进的性能优化。微小优化通常不值得，因为编译器和 CPU 发展技术通常会使它们在几年内（甚至更快）变得过时。
* 有争议的变化。这些需要在 <freebsd-arch@freebsd.org> 或最适合的邮件列表上先行讨论。GitHub 并不适合讨论此类问题。

PR 应该以某种用户可见的方式上改进项目。

## 评估标准

* 该变更是否正在被 FreeBSD 项目接受（accept）？
* 变更的范围及规模是否合适？
* 提交数量是否合理（比如少于 20 个）？
* 每个提交的大小是否适合审查（比如少于 100 行）？
* C 和 C++ 代码是否符合 style(9) 风格（或当前文件的风格）
* 对 lua 的更改是否符合 style.lua(9) 规范
* 对 Makefile 的更改是否符合 style.Makefile(9) 规范
* 更改 man 页面是否同时改了 `mdoc -Tlint` 和 `igor` ？
* 有争议的更改是否在相关邮件列表中讨论过？
* `make tinderbox` 是否成功运行？
* 是否修复了特定且明确的问题，或者添加了特定的功能？
* 提交信息是否良好？

同时要避免以下问题：

* 是否引入了新的测试回归？
* 是否引入了行为回归？
* 是否引入了性能回归？

## 流程概述

从最上层来看，提交到 FreeBSD 的过程简单明了，尽管深入到细节，可能会使这种简洁性变得不是那么明显。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/basicflowchart.png)

1. FreeBSD 开发者直接向 FreeBSD 存储库推送提交，该存储库托管在 <FreeBSD.org> 集群。
2. 每 10 分钟，FreeBSD 源代码库会被镜像到 GitHub 存储库——freebsd-src。
3. 想创建 PR 的用户会在他们的 freebsd-src 存储库的复刻上创建一个分支。
4. 通过用户分支上的更改来创建 FreeBSD PR。
5. FreeBSD 开发人员审核 PR，提供反馈，并可能要求用户进行修改。
6. FreeBSD 开发人员将更改推送到 FreeBSD 源代码库。

## 为接受 RP 做准备

如果你还没有 GitHub 账户，请创建之。此[链接](https://github.com/join)将指导你完成创建新 GitHub 账户的过程。由于许多人已因其他原因而拥有 GitHub 账户，我们将跳过具体的详情讨论。

下一步是将 FreeBSD 的存储库 fork（复刻）到你的账户中。使用 GitHub 的网页是创建分支，进行解释的最简方法——因为此操作你仅需执行一次。对分支的更改不会干涉 FreeBSD 存储库。用户可以通过单击“Fork”按钮（如图 1 所示）来复刻存储库。你需要点击突出显示的菜单项“Create a new fork（创建新复刻）”。将打开类似于图 2 的界面。在这里，点击绿色的按钮“Create Fork（创建复刻）”。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/Figure1_witharrows.jpg)

图 1：单击“fork”旁边的向下箭头后，你将看到一个弹出式窗口，显示创建复刻的对话框。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/Figure2.png)

创建复刻图 2，第 2 部分。

在你点击“Create Fork（创建复刻）”后，GUI 将重定向到新创建的存储库。你可以将克隆版本库所需的 URL 复制到常规位置，如图 3 所示。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/Figure3.png)

复制网址来克隆（我以前用旧存储库名称创建的复刻）图 3。

这些是你要在 GitHub 网页上执行的步骤。其他命令将在 FreeBSD 主机上的终端中完成。为简单起见，屏幕截图已更改为命令或命令及其生成的输出。

使用以下命令克隆你新创建的存储库。

```
% git clone
Cloning into 'freebsd-src'...
remote: Enumerating objects: 3287614, done.
remote: Counting objects: 100% (993/993), done.
remote: Compressing objects: 100% (585/585), done.
remote: Total 3287614 (delta 412), reused 815 (delta 397), pack-reused 3286621
Receiving objects: 100% (3287614/3287614), 2.44 GiB | 22.06 MiB/s, done.
Resolving deltas: 100% (2414925/2414925), done.
Updating files: 100% (100972/100972), done.
% cd freebsd-src
```

请注意，你应该把上述命令中的“user”改成你的 GitHub 用户名。“-o github”则会将此远程命名为“github”，在下面的示例中将使用它。（**译者注：这里有些对不上，大概是上面的命令写错了，正确的命令可能是 `git clone -o github https://github.com/user/freebsd-src`**）

PR 工作流通常需要一个分支。我们假设你已经按照类似以下命令操作，尽管有许多使用预先存在的分支的方法超出了本文的范围。

```
% git checkout -b journal-demo
% # make changes, test them etc
% git commit
```

你进行的所有提交，都必须把你的真实姓名和电子邮件地址作为提交的“作者”。Git 有两个配置字段用于此目的。`user.name` 是你的真实姓名。`user.email` 是你的电子邮件地址。你可以这样设置它们：

```
% git config --global user.name “Pat Bell”
% git config –global user.email “pbell@example.com”
```

此外，请阅读我们关于[提交日志信息](https://docs.freebsd.org/en/articles/committers-guide/#commit-log-message)的建议，并在创建提交时遵循它。

大多数通过 PR 方式提交的更改都很小，所以我们将继续提交它们。但是，如果你有较大的更改，请在提交之前阅读下面的评估标准，以获得更顺畅的流程。

## 提交你的 PR

下一步是把分支 journal-demo 推送到 GitHub（与上述类似，用你的 GitHub 用户名替换下方的“user”：

```
% git push github
Enumerating objects: 24, done.
Counting objects: 100% (24/24), done.
Delta compression using up to 8 threads
Compressing objects: 100% (16/16), done.
Writing objects: 100% (16/16), 5.21 KiB | 1.74 MiB/s, done.
Total 16 (delta 13), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (13/13), completed with 8 local objects.
remote:
remote: Create a pull request for ‘journal-demo’ on GitHub by visiting:
remote: https://github.com/user/freebsd-src/pull/new/journal-demo
remote:
To github.com:user/freebsd-src.git
* [new branch] journal-demo -> journal-demo
```

你会注意到，GitHub 会友好地告诉你如何创建一个 PR。当你访问上述 URL 时，会看到一个空白的表单，如图 4 所示。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/figure4.png)

图 4: PR 提交表单。

在字段“Add a Title（添加标题）”中，添加一个简要描述你的所做的工作，以传达修改的本质。保持在约十几个词以内，以便易于阅读。如果分支仅有单个提交，可以使用提交消息的第一行来作为此更改的标题。如果有多个提交，则需要总结它们为一个简短的标题。

在字段“Add a Description（添加描述）”中，写下你的更改摘要。如果这是一个仅单个提交的分支，请使用提交消息的正文部分。如果有多个提交，则创建一个简短的摘要，简要描述解决的问题。解释你做出了什么更改以及原因，如果不是那么显而易见的话。

图 4 中的示例试图解决贝尔实验室和伯克利之间著名的历史争端。这是下面所概述的一个存在争议的，提交的好例子。这是一个应该得到讨论的，有争议的，提交的好例子。

## 期待什么

在你提交之后，就开始评估过程了。将运行几个自动检查器。这些检查确保你的提交的格式和样式符合我们的指南。它们确保所进行的更改可以编译。它们将提供反馈，指出你在有人查看之前应该进行的更改。其中一些测试需要时间，因此在提交后几个小时再来检查是个好主意。自动测试标记的项目将是我们的志愿者要求你更正的首要事项，因此积极解决这些问题可以节省所有人的时间。

## 回复反馈

若收到反馈，通常需要更改代码。请执行建议的更改。通常这意味着你将不得不编辑你的某些部分更改（无论是提交消息还是提交本身）。GitLab 有一个关于使用 git rebase 机制的好[教程](https://docs.gitlab.com/ee/topics/git/git_rebase.html)。

在你进行了更改以后，你将需要将更改推送回你的分支，以便 PR 更新并重新反馈：

```
% git push github --force-with-lease
```

## 供应链攻击

最近，恶意行为者攻击了 xz 源代码存储库，插入了某些代码，从而影响了某些 Linux 系统上的 sshd。由于一定的运气和流程，FreeBSD 未受本次攻击的影响。我们的流程经过设计，可通过多重保护层抵御此类攻击。我们在允许代码进行测试之前会进行代码审查。仅在所提交的代码中不存在明显的恶意行为时，我们才会运行自动化测试。在今天，开源项目处于日益恶劣的工作环境下，一些看似多余的步骤往往是必要的。

## 结语

无论你是偶尔会对 FreeBSD 进行微调的休闲用户，还是提交变更非常频繁以至于将获得提交权限的开发者，FreeBSD 项目都欢迎你的提交。本文尝试涉足一些基础内容，但更偏向于休闲用户。在线资源将帮助你处理超出基础情况的情形。

---

**WARNER LOSH** 在 FreeBSD 项目成立前就参与了开源——甚至早于“open source，开源”这个术语的正式定义。最近，他深入了探索 Unix 的早期历史，旨在揭示其丰富而隐秘的财富。他与妻子和女儿居住在科罗拉多州的一座稻草屋中，这座房屋由太阳能、小锅炉（偶尔还用古董计算机）供热。
