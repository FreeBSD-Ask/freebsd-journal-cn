# Ports 报告

- 原文：[Ports Report](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-in-the-enterprise/)
- 作者：**Frederic Culot**

5 月到 6 月期间，Ports 上的活动非常频繁，相比 3 月到 4 月期间，Ports 树的变更数增加了超过 20%！Ports 框架也进行了许多改进，如下文所述。

## 新的 Ports 提交者与权限保管

5 月到 6 月期间，一位新的提交者获得了 Ports 提交权限：Bernard Spil，由 vsevolord@ 和 koobs@ 担任导师。还有一些提交权限被收回保管，有的是应提交者本人要求（如 sahil@ 的情况），有的是因超过 18 个月不活跃（如 clsung@、dhn@、obrien@ 和 tmseck@ 的情况）。

## 重要的 Ports 更新

antoine@ 共执行了 23 次 exp-run，以检查主要 Ports 更新是否安全。在这些重要更新中，以下是一些亮点：

- 默认 perl 版本设为 5.20
- python（2.7 分支）更新至 2.7.10
- cmake 更新至 3.2.3
- bison 更新至 3.0.4
- ffmpeg 更新至 2.7.1

一如既往，更新你的 Ports 之前，请仔细阅读 **/usr/ports/UPDATING** 文件，因为可能涉及手动步骤。

## Ports 框架更新

Ports 树基础设施做了一些最终用户可能不会立即注意到的变更。这包括移除已弃用的功能（例如移除 NEED_ROOT 宏，因为现在所有软件包都可以由普通用户构建）。还涉及改进某些功能，例如检查和安装依赖项的方式。这些是为 Ports 树准备 flavors 和 subpackages 的重要步骤，敬请关注这些重要的新特性，预计很快推出！此外，查看关键基础设施组件的提交日志很有意义，例如 **/usr/ports/Mk** 中的文件，因为这通常能很好地反映接下来的变化。这些提交日志可在线查看，如 bsd.port.mk（<https://svnweb.freebsd.org/ports/head/Mk/bsd.port.mk?view=log>）。

## 统计

一如既往，最后附上一些统计数据：5 月到 6 月的两个月期间，Ports 树共收到 5,595 次提交，相比上一期增长了超过 20%。在 bug 报告方面，关闭了 1,253 个 PR，相比 3 月到 4 月期间略有下降。感谢所有贡献者的辛勤工作，希望未来几个月有更多新开发者加入我们的行列。

**FREDERIC CULOT** 在 IT 行业工作了 10 年。业余时间他学习商业与管理，并刚刚完成 MBA。Frederic 于 2010 年作为 Ports 提交者加入 FreeBSD，至今已做出约 2,000 次提交，指导过 6 位新提交者，现在承担 portmgr 秘书的职责。
