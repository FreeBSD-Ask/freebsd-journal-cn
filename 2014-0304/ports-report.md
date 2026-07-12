# Ports 报告

- 原文：[Ports Report](https://freebsdfoundation.org/wp-content/uploads/2014/03/ports-report.pdf)
- 作者：**Thomas Abthorpe**

Ports 报告是对 Ports 基础设施近期活动、Ports 领域新闻、技巧与窍门的汇总。

## 新的 Ports 提交者

作为 Ports 树的专注贡献者，其他提交者会注意到你，并提议为你争取提交权限，这样他们就不必再替你做工作了。我们最近加入 Ports 提交者行列的是 Jost Meixner。Jost 的众多兴趣之一是维护 FreeBSD 的 Linux Ports。

## portmgr-lurkers 的新成员

随着 portmgr-lurker 项目的持续成功，我们启动了第二批 lurker（旁听者）的招募，让他们学习、观察并为 Ports 管理团队贡献。3 月，Alexy (danfe@) Dokuchaev 和 Frederic (culot@) Culot 开始履职。Frederic 还兼任 portmgr-secretary 的影子角色，协助处理该职位的职责。

## Ports 树的第二个分支

由于第一个分支——2014Q1——是实验性的，你可能还没有听说过它。2014 年 1 月发布了第一个季度分支，旨在提供一个稳定且高质量的 Ports 树。这些稳定分支是每三个月对 head Ports 树做一次快照，目前支持期为三个月，在此期间会接收安全修复与构建和运行时修复。软件包会定期在该分支上（每周）构建并通过 pkg.FreeBSD.org（/quarterly，而非通常的 /latest）发布。4 月 1 日（不开玩笑），2014Q2 分支创建，首批构建随后不久开始。

## 为改善 Ports 树尽一份力

前文提到的 **ports-mgmt/tinderbox** 和 **ports-mgmt/poudriere** 构建系统的配套工具是 **ports-mgmt/porttools**。借助这些工具，你可以创建新的 Port、通过 `send-pr.1` 为更新提交 PR，甚至通过执行 `port test` 命令将其当作简易构建系统使用。更多信息请阅读 <http://www.freebsd.org/doc/en/books/porters-handbook/testing-porttools.html>。

安装这些工具时，你还会获得另一个出色的 porter 工具 `portlint.1`。正如 `lint.1` 帮你剔除 C 程序中的瑕疵，portlint 运用启发式规则帮你发现错误的空白、位置不当的指令，并提供大量其他提示与建议来改进你的 Port。例如，执行：

```sh
portlint -C /usr/ports/devel/fakeport # 这只是示例
```

可能得到类似这样的输出：

```sh
WARN: Makefile: [14]: possible direct use of command "env" found. use ${SETENV} instead.
WARN: Makefile: only one MASTER_SITE configured. Consider adding additional mirrors.
WARN: Makefile: "RUN_DEPENDS" has to appear earlier.
WARN: Consider to set DEVELOPER=yes in /etc/make.conf
0 fatal errors and 5 warnings found.
```

于是我采纳建议，在 **/etc/make.conf** 中加入 `DEVELOPER=yes`。这本身对 portlint 无益，但会在构建 Port 时给出面向开发者的详细输出，颇为有用。其他警告则需要你这位 porter 仔细审视，判断是否需要采取额外行动。请记住：我们驾驭 portlint，而不是 portlint 驾驭我们。它的警告仅供指导参考。

Ports 树是 FreeBSD 的增值软件集合。它是全球贡献者协作的成果，每个人都为让它变得更好一点点而尽了一份力。向所有伸出援手的人致以诚挚的“谢谢”。

- <http://fb.me/portmgr> — 为我们点赞
- <http://twitter.com/freebsd_portmgr> — 关注我们
- <http://blogs.freebsdish.org/portmgr/> — 我们的博客
- <https://plus.google.com/u/0/communities/108335846196454338383> — 在谷歌+ 上圈我们

## 给准 Port 开发者的建议

使用（半）自动化的构建系统来测试你的 Port。在上一期中，我们推荐订阅 <http://redports.org> 来测试你的 Port。它对所有人开放，旨在服务公共利益。有些人可能拥有必要的硬件资源来搭建自己的构建系统。如果你是其中之一，那么 Ports 树中有一些工具可以尝试。最初的构建系统是老牌的 Tinderbox，在 Ports 树中以 **ports-mgmt/tinderbox** 找到，或其前沿版本 **ports-mgmt/tinderbox-devel**。可在其网站 <http://tinderbox.marcuscom.com/> 阅读更多信息。近年来出现了一套名为 Poudriere 的更新构建系统（这个名字大致译自法语，意为火药桶）。它可在 Ports 树中找到，名称为 **ports-mgmt/poudriere**，连同其前沿版本 **ports-mgmt/poudriere-devel**。可在其网站 <http://fossil.etoilebsd.net/poudriere> 阅读更多。Poudriere 构建系统现已成为 Ports 管理团队执行 -exp 测试和软件包构建的基础。

选择适合你的构建系统，善用它来测试构建 Port，验证它们能否干净地安装和卸载，甚至搭建你自己的私有软件包系统。
