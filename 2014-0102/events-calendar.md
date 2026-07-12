# 活动日历

## 新任 Ports 提交者

业内有句老话：如果你提交了太多 PR、修复了太多 Ports，并在邮件列表中持续热心贡献，就会被“惩罚”——给予 commit 权限。最近几个月，我们“惩罚”了以下诸位：John Marino（`marino@`），许多 BSD 项目的贡献者，尤为著名的是他在 DragonFlyBSD 中负责 DPorts；Rusmir Dusko（`nemysis@`），同时为 FreeBSD Ports 和 PC-BSD PBI 工作；David Chisnall（`theraven@`），近年担任 src 提交者，凭其丰富经验将在 FreeBSD 10 发布周期中为 Ports 工作提供关键助力；Danilo Gondolfo，长期为 Ports 树作贡献。

**Thomas Abthorpe** 是服务器管理员，在行业内拥有逾 20 年经验。他于 2007 年 8 月获得 Ports commit 权限，2010 年 3 月加入 Ports 管理团队，2012 年 7 月当选 FreeBSD 核心团队成员。在忙完 FreeBSD 事务之余，他在“Bicycles for Humanity”担任见习自行车技师。

## 给准 Port 开发者的建议

如果你有幸维护一个几乎不需要任何额外处理就能直接构建的 Port，那真是相当幸运。但并非所有 Port 都是如此。你常常需要修补一些代码片段，才能在 FreeBSD 上运行。其中最乏味的工作之一，就是维护你 Port 下 files 子目录中的补丁列表。与其手动运行 `diff` 生成补丁，不如在你 Port 的目录下运行 `make makepatch`，它会替你汇总所有补丁。此外，也请记得将你的补丁分享给该 Port 的上游开发者，这样可以保证持续的兼容性与可移植性。

## 近期 BSD 活动

- 作者：**Dru Lavigne**

2014 年第一季度安排了若干 BSD 相关会议，详见下文。这些活动和本地用户组聚会的更多信息，可访问 <https://bsdevents.org/>。新年快乐！

## FOSDEM

2014 年 2 月 1–2 日

比利时布鲁塞尔

<https://fosdem.org/2014/>

FOSDEM 2014 将在 ULB Campus Solbosch 举行。该活动免费入场，并设有专门的 BSD 开发者会议室，用于 BSD 相关的演讲。活动期间还会举办 BSDA 认证考试。

## NYCBSDCon

2014 年 2 月 8 日

纽约市

<http://www.nycbsdcon.org>

NYC BSD 用户组（NYCBUG）将再次在纽约举办为期一天的会议。地点位于 Suspenders Bar and Restaurant（111 Broadway）。今年会议主题为“生产环境中的 BSD”。与往届 NYCBSDCon 一样，费用将保持亲民，确保活动可及，并充满一系列反映 BSD 生生不息现实的会议与讨论。

## SCALE 12x

2014 年 2 月 21–23 日

加州洛杉矶

<http://www.socallinuxexpo.org/scale12x>

第 12 届南加州 Linux 博览会（SCALE）将在洛杉矶机场希尔顿酒店举行。这是对 BSD 友好的活动，将包括关于 PC-BSD 和 FreeNAS 的演讲，展区内设有 FreeBSD 展位。活动还将在 2 月 23 日举办 BSDA 认证考试。会议注册费为 60 美元。

## AsiaBSDCon

2014 年 3 月 13–16 日

日本东京

<http://2014.asiabsdcon.org/>

AsiaBSDCon 2014 将在东京理科大学举行。会议提供教程与演讲，并发布会议论文集。活动期间还举办 FreeBSD 开发者峰会和厂商峰会。本届活动将首次推出日语版 BSDA 认证考试。
