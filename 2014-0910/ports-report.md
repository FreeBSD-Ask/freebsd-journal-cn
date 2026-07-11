# Ports 报告

- 原文标题：Ports Report
- 作者：**Frederic Culot**

我们的月度仪表板显示，由于各位可敬的提交者享受了应得的暑假，夏季活动有所减少。不过，FreeBSD Ports 树上的活动从未停止。以下是过去几个月发生的重要事件。

## 新的 Ports 提交者

很高兴欢迎三位新的人才加入 Ports 提交者行列。

- 首先，Rui Paulo（人称 rpaulo@，因为他此前已拥有 src 提交权限）在为 Ports 树做了大量工作后获得了 Ports 提交权限。看到来自其他领域（无论是 src 还是 doc）的人加入 Ports 提交者行列总是令人高兴，他们常常为我们的流程带来新的视角。

- 接着，Alonso Schaich 加入我们，将由 rakuco@ 和 makc@ 担任导师。

- 最后同样重要的是，Dan Langille（大家可能通过 FreeBSD Diary 或 FreshPorts 认识他，他还是 BSDCan 的组织者之一）也获得了提交权限。

## 重要里程碑

9 月 1 日，我们将与旧的 `pkg_` 工具告别（详见 [http://blogs.freebsdish.org/portmgr/2014/02/03/time-to-bid-farewell-to-the-old-pkg_-tools/]）。经过多年服务，它们将由全新闪亮的 **pkg(8)** 取代。切换后，所有剩余的未暂存（unstaged）Port 将被移除，为更佳的树内 QA 和许多令人兴奋的新功能铺路！准备好迎接出色的新功能吧，比如以非 root 用户身份构建和打包、无需安装即可打包，以及创建子包的能力。Ports 树的未来一片光明！

## 新的软件包仓库

如最近在 freebsd-ports 邮件列表上所宣布的 [https://lists.freebsd.org/pipermail/freebsd-ports/2014-August/094787.html]，创建了一个新的仓库，其中的软件包在构建时启用了栈保护支持（即开启了 `-fstack-protector` 编译标志）。这是在使用 Ports 树中的软件时迈向更好安全性的重要一步，欢迎所有用户在它成为默认仓库之前安装并试用此仓库中的软件包。要用提供更好缓冲区溢出保护的软件包更新你自己的软件包，只需按初始公告 [https://lists.freebsd.org/pipermail/freebsd-security/2014-August/007874.html] 中所述指定新的 SSP 仓库，并按常规方式使用 `pkg upgrade` 更新软件包即可。

## Ports 树 20 周年纪念

如上一期专栏所宣布，Ports 树于 8 月 21 日迎来 20 周年纪念，并为此发布了一段视频 [http://youtu.be/LiFq5D-zmBs]。我们很高兴地看到，这段视频在两天内有约 2,000 人观看！

## Ports 树的值得关注的变更

尽管是暑假，仍有一些值得一提的变更。其中包括：季度分支 2014Q3 已分出，pkg 工具已更新到 1.3.7 版本，还进行了大量清理工作，主要是避免使用基本系统中的 texinfo 和 libreadline，并处理未暂存 Port。
