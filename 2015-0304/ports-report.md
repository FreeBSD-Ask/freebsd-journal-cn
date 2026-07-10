# Ports 报告

- 原文：[Ports Report](https://freebsdfoundation.org/our-work/journal/browser-based-edition/zfs-best-practices/)
- 作者：**Frederic Culot**

新年伊始，PR 方面的活动持续活跃，2 月更是达到高峰，关闭了 600 多份问题报告！感谢所有为此出力的贡献者、开发者。就 Ports 树而言，1 月与 2 月两个月间共有 4000 多次提交。这成绩不算差，但按这个速度，去年超过 37,500 次提交的纪录依然未被打破，因此 2015 年欢迎任何有意参与 Ports 工作的志愿者加入我们的行列！

## 重要的 Ports 更新

为检查重大 Ports 更新是否安全，我们执行了若干次 exp-run（确切地说是 23 次）。其中值得列出的有：

- 默认 python3 版本设为 3.4
- 默认 ruby 版本设为 2.1
- gcc 更新至 4.9
- clang 更新至 3.6.0
- cmake 更新至 3.1.3
- ruby-gems 更新至 2.4.5

一如既往，更新 Ports 前请仔细阅读 **/usr/ports/UPDATING** 文件，因为可能涉及手动步骤。

## 新的 Ports 提交者与保管

1 月与 2 月间有两位新提交者获得 Ports 提交权限：Jan Beich，由 bapt@、flo@ 担任导师；Brad Davis，他此前已拥有 doc 提交权限，由 bdrewery@、swills@、zi@ 担任导师。

过去两个月仅有一位提交者的权限被收回保管（rafan@），但也有令人惋惜的消息：decke@ 决定辞去在 FreeBSD 的职务，转而专注于家庭与职业生活。decke@ 对 FreeBSD 项目贡献卓著，他是 redports.org、QAT 的创建者，也是为 FreeBSD 移植并维护 VirtualBox 的开发者。衷心感谢他的所有辛勤工作。想进一步了解 Bernhard，可阅读他的访谈：<http://blogs.freebsdish.org/portmgr/2013/10/29/getting-to-knowyour-portmgr-bernhardfroehlich/>。

在本期专栏结尾，我们想给出一些统计数据，让大家更直观地了解志愿者完成的大量工作。2015 年前两个月，Ports 树共应用了 4046 次提交，关闭了 1182 个 PR，portmgr@ 共收到 1002 封邮件（未计入垃圾邮件……）。而这一切，是由平均仅 130 名活跃的 Ports 开发者完成的！

**Frederic Culot** 在 IT 行业工作了 10 年。业余时间他学习商业与管理，刚取得 MBA 学位。Frederic 于 2010 年作为 Ports 提交者加入 FreeBSD，此后完成约 2000 次提交，指导了六位新提交者，现担任 portmgr-secretary。

## 活动

欧洲有一场重要活动（FOSDEM），汇集了开源界的各路主要玩家。活动期间，bapt@ 发表了一场精彩演讲，讲述 `pkg`(8) 的历史——`pkg`(8) 已成为在 FreeBSD 上安装第三方软件的主要工具。幻灯片可在 <https://fosdem.org/2015/schedule/event/4yearofpkg/> 在线查阅。
