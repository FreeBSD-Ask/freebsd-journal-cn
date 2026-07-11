# Ports 报告

- 原文：[Ports Report](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking/ports-report/)
- 作者：**Thomas Abthorpe**

Ports 报告是对 Ports 基础设施近期活动的汇总，涵盖 Ports 领域的新闻、技巧与窍门。

## 新增 Ports Committers

只要你专注贡献 Ports 树，其他提交者就会注意到你，进而提议授予你 commit 权限，这样他们就不再需要替你做工作 [sic]。我们最近加入 Ports Committers 行列的是 Jost Meixner。Jost 兴趣广泛，包括维护 FreeBSD 的 Linux Ports。

## portmgr-lurker 项目的新成员

随着 portmgr-lurker 项目的持续成功，我们启动了第二批 lurker 的招募，让他们学习、观察并为 Ports 管理团队贡献。3 月，Alexy (danfe@) Dokuchaev 和 Frederic (culot@) Culot 开始履职。Frederic 还身兼二职，作为 portmgr-secretary 的影子并协助该职位的工作。

## Ports 树的第二条分支

第一条分支——2014Q1——是试验性的，你可能尚未耳闻。2014 年 1 月发布了首个季度分支，旨在提供一个稳定且高质量的 Ports 树。这些稳定分支每三个月从主 Ports 树截取一次快照，目前支持期为三个月，期间会接收安全修复以及构建与运行时修复。软件包会定期（每周）在该分支上构建，并照常通过 `pkg.FreeBSD.org` 发布（路径为 **/quarterly**，而非通常的 **/latest**）。4 月 1 日（并非玩笑），2014Q2 分支创建完成，首批基于该分支的构建随后不久便开始。

## 为改进 Ports 树出一份力

前文提到的 **ports-mgmt/tinderbox** 和 **ports-mgmt/poudriere** 构建系统还有配套工具 **ports-mgmt/porttools**。借助这些工具，你可以创建新 Port，通过 **send-pr(1)** 为更新提交一份 PR，甚至发出 `port test` 命令把它当作简易构建系统使用。更多内容可阅读 <http://www.freebsd.org/doc/en/books/porters-handbook/testing-porttools.html>。

安装该工具包时，你还会得到名叫 **portlint(1)** 的优秀 porter 工具。正如 **lint(1)** 帮你去除 C 程序中的“绒毛”，**portlint(1)** 用启发式方法帮你发现错误的空白、错位的指令，以及其他大量改进 Port 的提示与建议。例如运行：

```sh
portlint -C /usr/ports/devel/fakeport # 这只是一个示例
```

可能得到类似下面的输出：

```sh
WARN: Makefile: [14]: possible direct use of command "env" found. use ${SETENV} instead.
WARN: Makefile: only one MASTER_SITE configured. Consider adding additional mirrors.
WARN: Makefile: "RUN_DEPENDS" has to appear earlier.
WARN: Consider to set DEVELOPER=yes in /etc/make.conf
0 fatal errors and 5 warnings found.
```

我采纳建议，把 `DEVELOPER=yes` 加入 **/etc/make.conf**。这本身对 `portlint` 无益，但会在构建 Port 时给出面向开发者的详尽输出，颇为有用。其余警告则需要你这个 porter 仔细审视，看是否需要进一步行动。请记住：我们驾驭 `portlint`，而非 `portlint` 驾驭我们。它的警告仅供参考。

- <http://fb.me/portmgr> —— 点赞
- <http://twitter.com/freebsd_portmgr> —— 关注
- <http://blogs.freebsdish.org/portmgr/> —— 我们的博客
- <https://plus.google.com/u/0/communities/108335846196454338383> —— 在 G+ +1 我们

## 给准 Port 开发者的提示

使用（半）自动化构建系统测试你的 Port。在上一期中，我们推荐订阅 <http://redports.org> 来测试 Port。该服务对所有人开放、为公共利益服务。有些人拥有必要的硬件资源来构建自己的构建系统。如果你正是其中一员，那么 Ports 树里有些工具可以一试。最初的构建系统是老牌的 Tinderbox，在树中以 **ports-mgmt/tinderbox** 出现，还有其前沿版本 **ports-mgmt/tinderbox-devel**。可阅读其网站 <http://tinderbox.marcuscom.com/> 了解更多。近年来出现了名为 Poudriere 的较新构建系统，译自法语大致就是“tinderbox”。可在 Ports 树中找到 **ports-mgmt/poudriere**，以及其前沿版本 **ports-mgmt/poudriere-devel**。更多信息见 <http://fossil.etoilebsd.net/poudriere>。`poudriere` 构建系统现已成为 Ports 管理团队执行 `-exp` 测试运行和软件包构建的基础。

选择适合你的构建系统，利用它测试 Port 构建，验证其能干净地安装和卸载，甚至搭建你自己的私有软件包系统。

Ports 树是 FreeBSD 的增值软件集合。它是全球贡献者共同协作的成果，每个人都做了一点点让它变得更好的事情。向所有伸出援手的人致以诚挚的谢意。
