# Ports 报告

作者：Thomas Abthorpe

欢迎来到全新《FreeBSD 期刊》的首期 Ports Report 专栏。我是 Thomas Abthorpe，FreeBSD Ports 管理团队秘书，即 `portmgr-secretary@`。让我来撰写这个专栏似乎顺理成章，因为我负责团队的大部分通信事务。

那么你可以在 Ports Report 中期待读到什么？它将是 Ports 基础设施近期活动的汇总、Ports 领域的新闻、提示与技巧，以及我能塞进来的各种零星知识。

## 构建和测试你的 Ports 的资源

成为 porter 是为 FreeBSD 项目做贡献的最简单方式之一。维护叶子 Ports 通常需要的资源很少，基本可以在小型家用系统上完成。但如果你维护的 port 依赖 GTK 或 QT 或其他需要先构建大量 Ports 的东西，该怎么办？这时 RedPorts.org 就派上用场了。RedPorts 是由 `portmgr@` 维护、供 porter 社区使用的机器集群。你只需注册账户并熟悉如何签入你的个人 Ports 树做测试。一旦 port 为测试目的提交，它会交给构建器，由构建器组装所有依赖直到你的 port 被构建。流程结束时，服务器上会保留日志，你随后可以审查完整性。不要气馁；有时 port 需要调整几次才能干净编译。

## Ports 树新动态

Ports 树的本质是不断演化、增长、用新软件更新。基础设施中不时引入值得注意的改进和/或改变 port 构建方式的变更。2013 年 7 月，`pkg_install` 在 10-CURRENT 上停止构建，这是为 pkgng 做准备。2013 年 9 月，Stack Protector 支持引入 amd64 和 i386 的 10-CURRENT。

基础设施的最新增项之一是 staging，它允许将 port 构建到分阶段目录中，而非安装到生产环境。除其他功能外，这还允许以非特权账户创建和打包软件包。你可在 <https://lists.freebsd.org/pipermail/freebsd-ports-announce/2013-> 阅读更多内容。

## 来自 PORTMGR@ 的增值服务

有时，对 Ports 树或 src 树的提交看似微不足道，却对用户态有深远影响。需要时，`portmgr@` 可对 Ports 树执行完整运行，查看变更如何影响 port 的构建。有一些高影响变更需要协调——例如更改 Perl、Ruby 或 Python 的默认版本。还有许多其他变更，你随时可以请求 `portmgr@` 协助。

## 为改善 Ports 树尽一份力

FreeBSD Ports 的优势之一在于我们 porters 的多样性——这些热心的志愿者持续测试并更新 Ports，使其保持最新和安全。随着基础设施演化，操作 Makefile 的新方法不断引入。验证 port 的可靠方法是运行 **portlint(1)**。它能发现语法错误、空白问题，以及你需要清理的各种琐碎问题。最近基础设施中加入了一项新功能，可对你的 Makefile 提供额外警告，提示你可以执行的额外变更。设置环境变量 `DEVELOPER=yes`，或将其加入 **/etc/make.conf**。这样你可能会被提示修改 Makefile 以使用最新支持的功能。

- <https://fb.me/portmgr>——给我们点赞
- <https://twitter.com/freebsd_portmgr>——关注我们
- <https://blogs.freebsdish.org/portmgr/>——我们的博客

## 新任 Ports 提交者

业内有句老话：如果你提交了太多 PR、修复了太多 Ports，并在邮件列表中持续热心贡献，就会被“惩罚”——给予 commit 权限。最近几个月，我们“惩罚”了以下诸位：

- **John Marino**（`marino@`），许多 BSD 项目的贡献者，尤为著名的是他在 DragonFlyBSD 中负责 DPorts
- **Rusmir Dusko**（`nemysis@`），同时为 FreeBSD Ports 和 PC-BSD PBI 工作
- **David Chisnall**（`theraven@`），近年担任 src 提交者，凭其丰富经验将在 FreeBSD 10 发布周期中为 Ports 工作提供关键助力
- **Danilo Gondolfo**，长期为 Ports 树作贡献

## 给准 Port 开发者的建议

如果你有幸维护一个几乎不需要任何额外处理就能直接构建的 port，那真是相当幸运。但并非所有 port 都是如此。你常常需要修补一些代码片段，才能在 FreeBSD 上运行。其中最乏味的工作之一，就是维护你 port 下 files 子目录中的补丁列表。与其手动运行 `diff` 生成补丁，不如在你 port 的目录下运行 `make makepatch`，它会替你汇总所有补丁。此外，也请记得将你的补丁分享给该 port 的上游开发者，这样可以保证持续的兼容性与可移植性。

---

**Thomas Abthorpe** 是服务器管理员，在行业内拥有逾 20 年经验。他于 2007 年 8 月获得 Ports commit 权限，2010 年 3 月加入 Ports 管理团队，2012 年 7 月当选 FreeBSD 核心团队成员。在忙完 FreeBSD 事务之余，他在“Bicycles for Humanity”担任见习自行车技师。
