# Ports 报告

- 作者：**Frederic Culot**

1 月至 2 月期间，问题修复方面的活跃度非常高。在此期间，我们也高兴地欢迎 miwi@ 回归 port 管理团队，并迎来 3 位新加入或回归的提交者。此外，若干重要 ports 也得到更新，升级时可能需要谨慎处理，详见下文。

## 新 Ports 提交者与保管

今年前两个月，有 3 个提交位因保管被收回：mmoll、milki 和 brian。另一方面，有 2 个新的提交位被授予。第一个新提交位授予 Olivier Cochard-Labbe，他作为 FreeNAS 和 BSD 路由器项目（BSDRP）的创建者在社区中早已闻名。第二个提交位授予 Christoph Moench-Tegeder，他向 Chromium、Firefox 和 Thunderbird 提交了若干重要补丁，自 2005 年以来累计提交近 200 份问题报告。2 月，我们也很高兴地恢复了 Dima Panov（fluffy@）的提交位——去年 9 月，因 18 个月未活跃该提交位被收回保管。

## 统计

撰写本文时 2 月数据尚未出炉，但根据 1 月数据，数字令人鼓舞：参与 ports 树工作的提交者比上一周期更多（128 人，增幅超过 10%），由此带动提交数增加近 20%。更令人振奋的是 1 月关闭的问题报告数量，增幅超过 30%。这非常鼓舞人心，希望所有开发者能保持这样的节奏！

## 基础设施

bdrewery@ 和 antoine@ 做了大量工作以改进软件包构建与测试基础设施。这包括升级托管这些服务的机器上的系统，和迁移托管 pkgstatus 服务的机器（<https://pkg-status.freebsd.org>）。此外还做了一些提升构建系统性能的工作。

## 管理团队变动

1 月举行了一次投票，决定让 miwi 回归 portmgr 团队。结果全票通过，miwi 成为 portmgr 的第 6 位成员。自 2014 年底 miwi 因家庭和工作事务繁忙而不得不卸任以来，我们一直很想念他。欢迎他归队！如需进一步了解 portmgr 团队和其职责，可从以下链接入手：<https://www.freebsd.org/portmgr/>。

## 重要 Ports 更新

antoine@ 像往常一样进行了大量 exp-run（约 30 次），以检查重要 ports 更新是否安全。这些重要更新中，可以提及以下亮点：

- Ruby 2.3 加入树，默认版本设为 2.2
- ruby-gens 更新至 2.5.1
- clang 更新至 3.8.0
- Qt5 更新至 5.5.1
- setuptools 更新至 20.0
- Gnome 更新至 3.18

像往常一样，更多信息可在 **/usr/ports/UPDATING** 文件中找到，升级重要 ports 之前应仔细阅读该文件。

---

**Frederic Culot** 完成了物理学计算机建模方向的博士学位后，在 IT 行业工作了 10 年。业余时间学习商业与管理，刚刚完成了 MBA。Frederic 于 2010 年以 ports 提交者身份加入 FreeBSD，至今累计约 2000 次提交，指导过 6 位新提交者，现担任 portmgr 秘书。
