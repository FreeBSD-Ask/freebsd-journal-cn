# 在 GitHub 上向 FreeBSD 提交 PR

作者：WARNER LOSH

FreeBSD 项目最近开始支持 GitHub 拉取请求（PR）以让人们更易贡献。我们发现使用缺陷跟踪器 Bugzilla 接受补丁会导致许多有用的贡献被忽视并过期，因此贡献者应该更倾向于使用 GitHub PR 进行修改，只把 Bug 放在 Bugzilla。虽然 Phabricator 对开发人员很有效，但我们也发现在那里极易丢失外部贡献者的提交。除非您与直接告诉您使用 Phabricator 的 FreeBSD 开发人员一起工作，否则还是建议用 GitHub。GitHub PR 更易跟踪和处理，并且对大部分开源社区都更为熟悉。我们希望更快的决策，更少的提交被放弃，并为所有人提供更好的体验。

由于 FreeBSD 的志愿者时间有限，本项目制定了标准、规范和政策，以有效利用他们的时间。您需要了解这些内容才能提交好的 PR。我们有一些自动化程序来帮助提交者修复常见错误，从而使志愿者可以审查几乎就绪的提交。请理解我们只能接受最有用的贡献，有些贡献是无法被接受的。

接下来，我将介绍如何将您的提交切换为 Git 分支，如何完善它们以符合 FreeBSD 项目的标准和规范，如何从您的分支创建 PR，以及如何预期审查过程。然后，我将为志愿者介绍如何评估 PR，并提供完善 PR 的技巧。

本文关注于基本系统的提交，不涉及文档和 ports。这些团队仍在修订有关这些存储库的详细内容。

## 项目标准

FreeBSD 项目对系统的各个方面都有详细的标准。这些标准在 FreeBSD 开发者手册和 FreeBSD 提交者指南中进行了说明。代码标准在 FreeBSD 手册页中有记述。根据惯例，手册页被分为多个部分。出于历史原因都所有风格手册页在第 9 部分。对手册页的引用通常呈现为页面名称，后跟其部分编号在括号中，例如 style(9) 或 cat(1)。这些文档可以在任何 FreeBSD 系统上使用 man 命令获取，也可在线浏览。

FreeBSD 项目致力于创建文档齐全的集成系统，涵盖了控制机器的内核以及常见 Unix 工具的用户空间实现。提交应写得清晰，并包含相关评论。当行为发生变化时，应更新相关手册页。例如，当您向命令添加了参数时，也应将其添加到手册页中。当库中添加新功能时，应为这些功能添加新的 man 页面。最后，FreeBSD 项目认为源代码控制系统中的元数据是系统的一部分，因此提交信息也应符合项目的标准。

FreeBSD 项目对 C 和 C++ 代码的规范在 style(9) 中说明。这种风格通常被称为“内核规范形式”，并采用了 Kernighan 和 Ritchie 的《C 程序设计语言》中使用的风格。这是研究 unix 使用的标准，后来在伯克利的 CSRG 中继续使用，并产生了 BSD 发行版。FreeBSD 项目在这些实践的基础上进行了现代化。这种风格是提交代码的首选风格，并说明了系统中大多数代码使用的风格。更改这些代码的贡献应遵循这种风格，但某些文件有自己独特的风格。Lua 和 Makefiles 也有自己的标准，分别能在 style.lua(9)和 style.Makefile(9)找到。

提交信息采用使用 git 的开源社区所通行的形式。第一行应对整个提交进行概述，字数在 50 个字符以内。提交信息的其余部分应陈述提交的内容和原因。如果改动是显而易见的，最好只解释原因。行数若为 72 个字符或更少。应使用现在时，用命令的语气。最后有一系列 Git 称为 "预告片 "的行，项目会用它们来追踪提交的其他数据：提交来自哪里、在哪里可以找到关于错误的详细信息等。提交者指南](https://docs.freebsd.org/en/articles/committers-guide/#commit-log-message) 中的 "提交日志消息 "部分涵盖了所有细节。


## 无法接受的提交

在实验性地使用 GitHub 接受提交几年以后，FreeBSD 项目不得不设定一些限制，以确保未获得项目存储库写入权限的人员对使用 GitHub 提交的修改限制在合理范围内。这些限制确保了验证和应用修改的志愿者能够最有效地利用他们的时间。因此，FreeBSD 项目无法接受以下内容：

* 在 GitHub 上无法审核过大的变更
* 注释中的拼写错误
* 通过对源代码上运行的静态分析器发现的提交（除非它们包含了对静态分析器发现的错误的新测试用例）。针对与我们的测试工具箱不良互动的系统部分中的“显然正确”修复，可根据具体情况进行例外处理。
* 理论性的变化，但没有具体的错误及可表达的行为缺陷。
* 没有配备前后对比度量以显示改进的性能优化。微小优化通常不值得，因为编译器和 CPU 发展技术通常会使它们在几年内变得过时（甚至更快）。
* 有争议的变化。这些需要首先在 freebsd-arch@freebsd.org 或最适合的邮件列表上进行社交化。GitHub 不是一个讨论这类问题的好论坛。

PR 应该以某种用户可见的方式上改进项目。

## 评估标准

* 这个变更是项目正在接受的吗？
* 变更的范围和规模是否合适？
* 提交数量是否合理（比如少于 20 个）？
* 每个提交的大小是否适合审查（比如少于 100 行）？
* C 和 C++ 代码是否符合 style(9) 风格（或文件当前的风格）
* 对 lua 的更改是否符合 style.lua(9) 规范
* 对 Makefile 的更改是否符合 style.Makefile(9) 规范
* 更改 man 页面是否同时改了 mdoc -Tlint 和 igor ？
* 有争议的更改是否在适当的邮件列表中讨论过？
* make tinderbox 是否成功运行？
* 是否修复了特定且明确的问题，或者添加了特定的功能？
* 提交信息是否良好？

同时要避免以下问题：

* 引入了新的测试回归吗？
* 引入了行为回归吗？
* 引入了性能回归吗？

## 流程概述

从高层次来看，提交到 FreeBSD 是一个简单明了的过程，尽管深入细节可能会使这种简单性变得不那么明显。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/basicflowchart.png)

1. FreeBSD 开发者直接向 FreeBSD 存储库推送提交，该存储库托管在 FreeBSD.org 集群中。
2. 每 10 分钟，FreeBSD 源代码库会被镜像到 GitHub 存储库 freebsd-src。
3. 想要创建 PR 的用户会在他们的 freebsd-src 存储库的分支上创建一个分支。
4. 用户分支上的更改会用来创建 FreeBSD PR。
5. FreeBSD 开发人员审核 PR，提供反馈，并可能要求用户进行修改。
6. FreeBSD 开发人员将更改推送到 FreeBSD 源代码库。

## 为接受 RP 做准备

如果您还没有 GitHub 帐户，您需要创建之。此链接将指导您完成创建新 GitHub 帐户的过程。由于许多人已经因其他原因拥有 GitHub 帐户，我们将跳过对详情的深入讨论。

下一步是将 FreeBSD 的存储库 fork 到您的帐户中。使用 GitHub 网页是创建分支并解释的最简单方法，因为您只需执行一次此操作。 对分支的更改不会影响 FreeBSD 的存储库。用户可以通过单击“Fork”按钮（如图 1 所示）来分叉存储库。 您将要点击突出显示的“创建新分支”菜单项。 这将打开类似于图 2 的屏幕。从这里，点击绿色的“创建分支”按钮。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/Figure1_witharrows.jpg)

图 1：单击“fork”旁边的向下箭头后，您将看到一个弹出式窗口，显示创建分支对话框。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/Figure2.png)

创建分支图 2，第 2 部分。

您点击“创建分支”后，GUI 将会重定向到新创建的存储库。您可以在通常的位置复制您需要克隆存储库的网址，如图 3 所示。

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/Figure3.png)

复制网址来克隆（我以前用旧存储库名称创建的分支）图 3。

这些是您要在 GitHub 网页上执行的步骤。其他命令将在运行 FreeBSD 的主机上的终端中完成。为简单起见，屏幕截图已更改为命令或命令及其生成的输出。

使用以下命令克隆您新创建的存储库。

`% git cloneCloning into 'freebsd-src'...remote: Enumerating objects: 3287614, done.remote: Counting objects: 100% (993/993), done.remote: Compressing objects: 100% (585/585), done.remote: Total 3287614 (delta 412), reused 815 (delta 397), pack-reused 3286621Receiving objects: 100% (3287614/3287614), 2.44 GiB | 22.06 MiB/s, done.Resolving deltas: 100% (2414925/2414925), done.Updating files: 100% (100972/100972), done.% cd freebsd-src`

请注意，您应该将上述命令中的“user”更改为您的 GitHub 用户名。“-o github”将会将此远程命名为“github”，在下面的示例中将使用它。

PR 工作流通常需要一个分支。我们假设您已经按照类似以下命令操作，尽管有许多使用预先存在的分支的方法超出了本文的范围。

`% git checkout -b journal-demo% # make changes, test them etc% git commit`

所有您进行的提交都必须将您的真实姓名和电子邮件地址作为提交的“作者”。Git 有两个配置字段用于此目的。user.name 包含您的真实姓名。user.email 包含您的电子邮件地址。您可以这样设置它们：

`% git config --global user.name “Pat Bell”% git config –global user.email “pbell@example.com”`

此外，请阅读我们关于提交日志信息的建议，并在创建提交时遵循它。

大多数通过 PR 方式提交的更改都很小，所以我们将继续提交它们。但是，如果您有较大的更改，请在提交之前阅读下面的评估标准，以获得更顺畅的流程。

## 提交您的 PR

下一步是把 journal-demo 分支推送到 GitHub（与上述类似，用您的 GitHub 用户名替换下面的“user”：

`% git push githubEnumerating objects: 24, done.Counting objects: 100% (24/24), done.Delta compression using up to 8 threadsCompressing objects: 100% (16/16), done.Writing objects: 100% (16/16), 5.21 KiB | 1.74 MiB/s, done.Total 16 (delta 13), reused 0 (delta 0), pack-reused 0remote: Resolving deltas: 100% (13/13), completed with 8 local objects.remote:remote: Create a pull request for ‘journal-demo’ on GitHub by visiting:remote: https://github.com/user/freebsd-src/pull/new/journal-demoremote:To github.com:user/freebsd-src.git* [new branch] journal-demo -> journal-demo`

`You’ll notice that GitHub helpfully tells you how to create a pull request. When you visit the above URL, you’re presented with a blank form, as shown in Figure 4.`

![](https://freebsdfoundation.org/wp-content/uploads/2024/07/figure4.png)

图 4: PR 提交表单。

在“添加标题”字段中，添加一个简要描述你的所做的工作，以传达修改的本质。保持在约十几个词以内，以便易于阅读。如果分支只有一个提交，可以使用提交消息的第一行来作为此更改的标题。如果有多个提交，则需要总结它们为一个简短的标题。

在“添加描述”字段中，写下您的更改摘要。如果这是一个只有一个提交的分支，请使用提交消息的正文部分。如果有多个提交，则创建一个简短的摘要，简要描述解决的问题。解释您做出了什么更改以及为什么，如果不是显而易见的话。

Figure 4 中的示例试图解决贝尔实验室和伯克利之间著名的历史争端。这是下面概述的一个存在争议的，提交的好例子。这是一个应该得到社会化的，有争议的，提交的好例子。

## 期待什么

在您提交之后，评估过程就开始了。将运行几个自动检查器。这些检查确保您的提交的格式和样式符合我们的指南。它们确保所提出的更改可以编译。它们将提供反馈，指出您在有人查看之前应该进行的更改。其中一些测试需要时间，因此在提交后几个小时再来检查是个好主意。自动测试标记的项目将是我们的志愿者要求您更正的首要事项，因此积极解决这些问题可以节省每个人的时间。

## 回复反馈

一旦收到反馈，通常需要进行代码更改。请进行建议的更改。通常这意味着您将不得不编辑一些您的部分更改（无论是提交消息还是提交本身）。 GitLab 有一个关于使用 git rebase 机制的好教程。

您进行了更改以后，您将需要将更改推送回您的分支，以便 PR 更新并且重启反馈：

`% git push github --force-with-lease`

## 供应链攻击

最近，恶意行为者攻击了 xz 源代码存储库，插入了某些代码，从而危害了某些 Linux 系统上的 sshd。由于一定的运气和流程，FreeBSD 没有受到这次攻击的影响。我们的流程经过设计，可以通过多重保护层来抵御此类攻击。我们在允许代码进行测试之前会进行代码审查。只有在提交的代码中不存在明显恶意行为时我们才会运行自动化测试。在开源项目必须应对日益恶劣的工作环境的情况下，某些看似不必要的问题往往是由此招致的。

## 结束

无论您是偶尔会对 FreeBSD 进行微调的休闲用户，还是提交变更如此频繁以至于将获得提交权限的开发者，FreeBSD 项目都欢迎您的提交。本文尝试涉足一些基础内容，但更适合偏向休闲用户。在线资源将帮助您处理超出基础情况的情形。

WARNER LOSH 自 FreeBSD 项目成立之前就开始参与开源，甚至早于在“开源”这个术语正式的定义。最近，他深入探索 Unix 的早期历史，揭示其丰富而隐秘的财富。他与妻子和女儿居住在科罗拉多州的一座稻草房屋中，这座房屋由太阳能、小锅炉（偶尔还用古董计算机）来加热。
