# 会议报告

作者：Roller Angel

## BSDCan 2017

Dan Langille 在 BSDCan 第一天的开场致辞中提到，对许多人来说，这个会议就像家一样。能感到被欢迎、能成为社区的一部分，这种感觉真好。

Devsummit 第一天之前，我从科罗拉多乘飞机前往加拿大渥太华。放下行李后，我直接去 The Royal Oak 酒馆参加 Goat BoF。已经有不少人边吃边喝边聊。The Royal Oak 离我住的地方只有几条街，每天步行去会场、酒馆和杂货店都很方便。坐在我周围的人也都在用 BSD，乐于全程谈论 BSD，这让我从一开始就有宾至如归的感觉——大家都很友善，对我做的事也很感兴趣。说实话，我没想过自己能这么融入。我曾以为这些人难以接近，毕竟有些名声在外、技术上又那么厉害。一开始，大家似乎只在各自的小圈子里活动。但面孔逐渐熟悉后，我发现这些圈子并不总是固定的一群人，他们也只是就特定话题交流的”普通人”。我发现参与这些对话很容易，大家也很乐意听取反馈。我主动请缨，凡是能帮上忙的地方都搭把手，最后加入了视频团队，负责录制演讲的音频和幻灯片。

## Devsummit 第一天

Devsummit 第一天上午，我早早到达领取胸牌并找位置坐下。Gordon Tetlow 接待了我，帮我找到胸牌。我认出了 Dru Lavigne，便走过去自我介绍。Dru 又把我介绍给 Warren，他送了我一张 FreeBSD 狗粮贴纸——奖励我把 FreeBSD 装在笔记本电脑上”吃自己的狗粮”。我又向 Allan Jude、Benedict Reuschling、Sean Webb 和 Kirk McKusick 作了自我介绍。不久，Gordon 简单介绍了一下当天议程。Benno Rice 主持了一场关于新行为准则的讨论，提到欢迎大家加入一个专门协助处理行为准则相关问题的兴趣小组。Allan 提出一个想法：为正式讨论项目重大变更建立一个流程，即 FCP（FreeBSD Community Proposal，FreeBSD 社区提案）。这类似于 Python 的 PEP（Python Enhancement Proposal）。

会上还讨论了如何对待非代码贡献以及如何在项目内认可这些贡献者。有人建议设立一种 FreeBSD 成员身份，授予那些非代码贡献对项目有价值的人。这件事很多人都在想，认为应尽快落地。FreeBSD Wiki 上的 junior jobs 也被提及，是了解如何参与 FreeBSD 贡献的好去处。休息时，Benedict 跟我聊了聊作为新人加入社区的感受，欢迎我参与讨论、分享想法。

休息后，英特尔代表介绍了他们的 Quick Assist Driver 技术，以及他们在 FreeBSD 上的实现。午餐时，我遇到 Pierre Pronchery，最后跟他一起去了黑客休息室。Pierre 是一名安全顾问，愿意带我练手提升安全技能。在黑客休息室里，我和 Pierre 一起准备以系统管理员身份加入 EdgeBSD 项目。我们开始讨论用 Flask 重建 edgebsd.org 网站——我过去一年在学 Python Web 开发，觉得用真实项目练手是不错的体验。Pierre 帮我创建了 PGP 和 SSH 密钥，这样就能安全地认证到我们要用的虚拟机。接着 Pierre 给我演示了一个叫 `talk` 的老程序——它让你和登录到同一台机器的其他用户通过彼此的终端交流。`talk` 真是太酷了！我打算参考 FreeBSD 期刊近期的一篇文章，让 Salt 配合这个基于 Flask 的新网站基础设施工作。离开黑客休息室前，我和 Michael Dexter 聊了几句，他告诉我文档团队非常需要帮忙，并且有充足的指导帮助我入门。我告诉 Michael 我打算去文档休息室，把编辑文档的工具上手一遍，以便帮忙。

## Devsummit 第二天

Devsummit 第二天，Ed Maste 介绍了即将发布的 FreeBSD 12 中”我们已有、需要和想要”的内容（关于何时停止支持 SPARC 基础设施的讨论颇有意思）。一次发布背后涉及大量规划。我们走到渥太华大学的一处楼梯口，拍了数张合影。随后是为新人举办的导师和定向交流会，让新人互相认识、找到志同道合的伙伴，便于交流协作。

参加完新人交流会后，我帮忙把会议登记材料装进 FreeBSD 手提袋。这些手提袋随后从大学运到 Red Lion 酒馆，我还在那里做了录制设备的培训。培训前，我和 Peter Hansteen 坐在一起，聊起用 JavaScript 写演讲幻灯片的方法——这种思路是合理的，尽管也有些争议。培训结束后，我去了文档休息室，开始配置计算机以便编辑文档。

## BSDCan 第一天

BSDCan 第一天，我早早到达，与录制团队协调。我们选定了各自负责录制的房间，并通过 WhatsApp 建立群聊用于协调。会议开始后，我们在欢迎会上听取了 Dan Langille 的致辞。基调演讲由加拿大互联网与电子商务法研究主席 Michael Geist 教授带来。他指出，法律界需要倾听技术社区的声音，并重点介绍了法律与技术的十大议题。其中之一是接入问题——所有人都应能享有体面的网速，通过让小运营商使用大网络的漫游服务来促进竞争。他总结道，法律界亟需想法、专业知识和更多发声的人。Michael 回答了观众的一系列问题后，我们分散到不同房间开始听演讲。我录制了 Michael Tuexen 关于 packetdrill 的演讲，非常有趣，这也是 FreeBSD 开发者峰会专题中唯一一场。接着，我参加了 Emmanuel Vadot 关于 FreeBSD on ARM 的演讲。紧接着是 poudriere image BoF（在同一房间），讨论主题与上一场衔接得很好——已经在谈嵌入式设备，自然过渡到设备镜像的创建。我问到当前对为 MIPS 架构主板构建镜像的支持情况，得知 poudriere 暂不支持，相关功能在 FreeBSD WiFi build 项目下推进。之后我留在同一房间听了 Brian Kidney 关于 DTrace 在 FreeBSD 上现状的演讲。这场演讲很出色，关于 DTrace 现状的信息量很大，观众参与度也很高，提问踊跃。休息片刻后，我去了文档休息室继续磨炼技能。Tim Moore 在场，帮我配置好文本编辑器，让文档编辑更顺手。我很快就学会了如何编辑和提交文档补丁。Dru 就坐在我旁边，随时回答我在流程中遇到的问题。从文档休息室出来，我又去了黑客休息室。那里已经有好几桌人在聊各种项目、推进各自的工作。还有一大群热热闹闹的人协作开发新的 libtrue 库。我稍微应酬了几句，然后打开笔记本，用刚学到的新技能继续做文档工作。晚上能去黑客休息室，真是让人愉快。

## BSDCan 第二天

会议第二天，我早早就来帮视频团队从笔记本电脑中提取前一天的录像，并把每间房的笔记本重新架好，准备最后一天的录制。我录的第一场是 Ken Moore 关于 SysAdm 的演讲，他分享了项目的进展，并讨论了各项功能及实现方式。接着录制了 Pierre Pronchery 关于加固 Pkgsrc 的演讲。我原以为 pkgsrc 加固会很麻烦，没想到有多种方式可以做到，而且它几乎能跑在任何 BSD 上。我打算根据这场演讲的提示，未来深入研究 pkgsrc。

上午议程结束后，我匆匆吃了午餐，赶去 BSD 用户组 BoF。我们讨论了过去的 BUG（BSD 用户组）如何成功、现在的 BUG 如何运作，以及为新人起步组建 BUG 的建议。我分享了自己在科罗拉多 BSD 用户组（CoBUG）和 Front Range BSD 用户组（FRBUG）的经历。我还就”我想组建一个 Python BSD 用户组——面向在 BSD 上使用 Python 的人”的想法征求了意见。大家鼓励我放手去做，看看是否有人感兴趣，等我有空时大概会试试。BoF 之后，视频团队的人手够用，于是我决定去听微软 Kylie Liang 关于在云中用 FreeBSD 构建高性能网站的演讲。Microsoft Azure 的现场演示占了相当篇幅。看到微软的人用 Windows 部署 FreeBSD 感觉挺新奇，但效果很好，用起来似乎也挺简单。下一场我听了 Stephen Herwig 的演讲，他是马里兰大学系统与网络实验室的博士生。他为 NetBSD 开发了一个新的安全模型，类似 OpenBSD 的 pledge 和 FreeBSD 的 capsicum，命名为 secmodel_sandbox。这是一场相当技术性的演讲，但我颇感兴趣。观众提出几个尖锐问题，Stephen 的回答细致而周全。

当天我参加的最后一场是 Rodney Grimes 的演讲，谈在企业环境中更好利用 FreeBSD 的挑战与机遇。构建一个开源操作系统工作量巨大，FreeBSD 项目的志愿者与厂商之间的关系至关重要。这场演讲涵盖许多过往经验，并提出如何利用当前趋势的建议。演讲结束后是闭幕会。会上宣布了一个令人兴奋的消息：将在台湾举办一场新会议，网址 <http://bsdtw.org>。还有一场出人意料的有趣慈善拍卖。所有物品拍卖完毕后，大家前往 The Red Lion 参加闭幕社交活动。我和 Henning Brauer 拍了合影，他穿着”I love FreeBSD”T 恤，为捐出 10 美元慈善款。我还整理了一份会议期间与我交谈过的人名单——不计闭幕社交活动中的交谈，因为太多了，没顾上记。名单上有 30 多人，包括 Moore 三兄弟（Moore 王朝）、Allan Jude、Michael W. Lucas、Benedict Reuschling、Kirk McKusick、Reyk Floeter、Pierre Pronchery、Sean Webb、Peter Hessler、Michael Dexter、Aaron Poffenberger、Dan Langille、Gordon Tetlow 等。我想对 FreeBSD 基金会批准我的旅行资助表示由衷感谢。能与社区见面并参与讨论，是一次很棒的体验。我很感激能参加自己的第一次 BSDCan。在文档休息室去过几次后，我逐渐熟悉了编辑文档所需的工具。会议结束时，我已经向 FreeBSD Bugzilla 提交了两个文档补丁，另有几个还在进行中。会议前，我以为会花很多时间摆弄 Onion Omega 和 Edge Router Lite 项目，但实际上总有更有趣的事情可做或可参与。那些项目回家也能做。我和 FreeBSD 社区相处得很愉快，期待通过编辑文档、配合 Bugzilla 等方式继续与他们合作。

ROLLER ANGEL 是科罗拉多州博尔德 The GLOBE Program 的技术支持人员和助理系统管理员。他喜欢长板冲浪、在计算机上学习新东西，以及与家人和四只小狗共度时光。
