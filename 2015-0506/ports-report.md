# Ports 报告

- 原文：[Ports Report](https://freebsdfoundation.org/our-work/journal/browser-based-edition/measure-twice-code-once/)
- 作者：**Frederic Culot**

3–4 月期间，Ports 方面的活动保持高水平，志愿者带来了重要变更，尤其是在 package 构建基础设施方面。我们也很高兴在此期间迎来了一位新开发者。

## 新的 Ports 提交者与保管

3–4 月期间，仅有一位新提交者获得 ports 提交权限，他就是 Michael Moll，由 swills@ 与 mat@ 担任导师。期间没有提交权限被收回保管。

## 重要的 Ports 更新

我们进行了若干次 exp-run（确切地说是 14 次），以检查重大 ports 更新是否安全。其中可圈可点的有：

- Xfce 更新至 4.12.0
- libtool 更新至 2.4.6

其他一些 exp-run 因报告了失败而阻止了某些更新被应用。boost 库的更新即是一例，它还需要更多工作才能进入 ports 树。

一如既往，更新 ports 前请仔细阅读 **/usr/ports/UPDATING** 文件，因为可能涉及手动步骤。

## 基础设施更新

许多旨在改进 FreeBSD ports 构建基础设施的重要工作在后台进行，因此可能被最终用户忽视。例如，过去两个月添加了新的 package 构建器，以允许每周进行三次 package 构建而非仅一次。当然，这意味着最终用户能获得更频繁的 package 更新。目标是提供每日 package 构建，为此正在优化 poudriere（我们创建和测试 FreeBSD package 的实用工具，见 <https://www.freebsd.org/doc/en/books/handbook/ports-poudriere.html>）。还有一个新工具正在开发中，用于监控最新的 package 构建：结果展示在专用网页上（<https://pkg-status.freebsd.org>）。想参与开发的人可在 GitHub 上找到源代码（<https://github.com/bdrewery/pkg-status.freebsd.org>）。

## 统计

3 月与 4 月两个月间，ports 树共应用了 4602 次提交，较上一周期增长超过 10%。在缺陷报告方面，关闭了 1324 个 PR，相较 1–2 月也有小幅增长。让我们希望这些数字继续增长，未来几个月有更多新开发者加入我们的行列！

**Frederic Culot** 在 IT 行业工作了 10 年。业余时间他学习商业与管理，刚取得 MBA 学位。Frederic 于 2010 年作为 ports 提交者加入 FreeBSD，此后完成了约 2000 次提交，指导了六位新提交者，现担任 portmgr-secretary。
