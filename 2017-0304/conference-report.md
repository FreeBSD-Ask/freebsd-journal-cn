# 会议报告：FOSDEM 2017

作者：Benedict Reuschling

FOSDEM（Free and Open Source Software Developers' European Meeting，自由与开源软件开发者欧洲会议）是一个非商业性、由志愿者组织的欧洲活动，以自由与开源软件开发为中心。它每年举办一次，通常在 2 月的第一个周末，地点在比利时布鲁塞尔东南部的布鲁塞尔自由大学 Solbosch 校区。

## 开发者峰会

去年在大会第一天举办的 FreeBSD 单日开发者峰会反响很好。今年我们决定额外加一天——安排在 FOSDEM 正式开始前的周五——这样就不会与参会者可能想听（或想讲）的任何演讲冲突，也给大家一个了解最新开发进展的机会。FreeBSD 基金会慷慨赞助了午餐和酒店会议室。周五共有 11 位开发者在那间会议室碰面，简单宣布几件事后，我们直接列出与会者感兴趣的话题。讨论结果如下：

- libifconfig（ifconfig 的库化）
- Wayland（FreeBSD 中现已有一个 port 可用）
- GSoC 参与和议题（头脑风暴）
- FreeBSD 对 Olimex 笔记本的支持，详见此篇博文：<https://olimex.wordpress.com/2017/02/01/teres-i-do-it-yourself-open-source-hardware-andsoftware-hackers-friendly-laptop-is-complete/>
- pkg flavors 与子包（pkg(8) 的进一步改进）
- base 中的 OpenSSL 私有化（ports 将始终使用来自 ports 的 OpenSSL），或以 embedTLS 替换 OpenSSL
- JuniorJobs 与 Ideas 页面（更新与刷新）

以下是峰会期间或随后几天取得的部分成果：

- 移除了翻译项目中包含过时发行版的下载页面。在某些情况下，这导致了荒唐的发行版，例如 FreeBSD 11 的 alpha、ia64 和 pc98 版本。这些已是不再支持的架构，但翻译工作尚未跟上现实，因此我们决定移除这些页面。其中一处，翻译者迅速响应并修复了问题，对应语言的页面也得以重新激活。
- 移除了 <www.freebsd.org> 上的镜像选择下拉菜单。过去这用于选择一个离自己位置更近的镜像以缩短页面加载时间。如今 FreeBSD 服务器基础设施会自动选择最近的服务器，因此无需再手动选择这些镜像（何况它们往往本就过时）。
- 移除 bdes(1)：<https://svnweb.freebsd.org/changeset/base/313329>

对我而言，这次开发者峰会提供了一个与 Sevan Janiyan 面对面接受指导的好机会。我们讨论了如何将手册页从 HEAD 合并到 stable/11，过程中遇到了一些合并未正确完成、产生意外结果的问题。Sevan 次日找到另一位更熟悉 Subversion 内部的提交者继续跟进。看来我们触发了一个罕见 bug，目前已构建测试场景作进一步调查。所幸找到了一个临时的本地变通方案，Sevan 得以继续他的工作。

上下午的茶歇让峰会参会者得以就着咖啡和当地糕点继续讨论。咖啡因储备得到补充后，我们回到各自的工作，或继续打磨 GSoC 的点子。当晚的峰会晚宴让计划次日参加 FOSDEM 的人也加入进来，大家就着食物和比利时啤酒聊起各种 BSD 相关话题。一位特别嘉宾是 Groff——那只 BSD 山羊，陪同它的是 Peter Hessler，他向我们讲述了过去几个月二位的行踪（你可以在 Twitter 上关注：<https://twitter.com/GroffTheBSDGoat>）。

## FOSDEM 现场

周六清晨，Allan Jude、Marie Helene Kvello-Aune 和我带着传单和宣传材料前往布鲁塞尔的 ULB 大学，参加 FOSDEM 的第一天。我们到达时，许多展台已搭好，但展台位置固定，问题不大。每个展台前都贴着一张介绍项目并附有大号二维码的海报，扫码即可了解更多。FreeBSD 基金会寄来了传单、贴纸、笔等小礼品供发放。Kristof Provost 带来了之前被比利时海关扣下的剩余物资——他总算把它救了出来。

布置展台时，Xen Project 的朋友们跟我们打招呼并提供帮助，其中几位我们前一晚在晚宴上见过。我们的邻桌是 illumos 项目的人，没过多久我们就 OpenZFS、DTrace 等共同感兴趣的话题聊了起来。看到他们的笔记本上跑着 FreeBSD 的引导加载器，令人欣慰——这是两个项目过去合作的成果。整个周末，我们不断向人们讲述两个项目间富有成效的关系，以及我们共享的技术与理念。

各展厅很快挤满了人，转眼间就有访客来问关于 FreeBSD 的各种问题。我得把展台交给 Allan 和 Kristof，因为上午要去监考 BSDA 考试。午饭时分我回来时，FOSDEM 已是热火朝天，开源项目展台区聚集了一大群人。好在我们有足够多的 FreeBSD 人手帮忙！

总体而言，访客对我们的项目表现出兴趣和开放态度。有人告诉我们 FreeBSD 在他们的机器上跑得有多好，几乎不需要维护。我们向他们介绍 FreeBSD 的最新特性，多数人都很感兴趣。有些人从未听过 FreeBSD，但听过 FreeNAS，甚至在用它。ZFS 是个热门话题，我们能向人们展示 FreeBSD 是家用和企业都可用的可行存储方案。许多人想从 Linux 发行版切换过来，并问这有多难。过去硬件驱动支持是个主要问题，但我们向他们保证，项目支持的硬件很多，情况也比几年前好得多。还有人对把 FreeBSD 用作嵌入式平台感兴趣，我们告诉他们 Netflix、iXsystems 等厂商在为产品选择 BSD 许可证后能做到什么。

其他 BSD 活动包括一些精彩演讲，例如 Brooks Davis 讲《关于”Hello, World”你一直想知道的一切》、Ed Schouten 讲 CloudABI、Arun Thomas 讲 RISC-V。第一天下午，FOSDEM 设有一个 BSD devroom。这是一个独立的房间，各项目在此就特定主题做演讲。Rodrigo Osorio 组织了这个 devroom，FOSDEM 提供的摄制团队录制了演讲。无法到场的人可以在 <https://fosdem.org/2017/schedule/track/bsd/> 观看。演讲座无虚席，又提供了一个向人们展示 BSD 系统能做什么、以及去年以来新增特性的好机会。

与此同时，FreeBSD 展台依然访客不断，我们不得不有意扣住几样热门赠品，以免当天结束前就发完。我们想让周日的参会者也有机会拿到。不过传单充足，我们尽可能多地发放。

周六晚，我和一群 FreeBSD 的人去了一家城外专门用啤酒烹调的餐厅，这绝对是一次明年还得再来的美食体验。

周日感觉比周六稍安静，但参会者仍足以让我们忙于发传单、答问题。有时人们认出了我们展台的 Allan Jude，不敢相信这位前 TechSnap 联合主持、BSDNow.tv 主持人居然在欧洲。这成了绝佳的开场白，人们想跟他聊聊，我们就借机向他们展示 FreeBSD。

最后一日晚餐后，Allan、Sevan 和我决定以小组形式做一点 hacking。我们坐在酒店里处理一些 PR 和遗留事项（也可称作临时黑客休息室）。

## 致谢

感谢 FOSDEM 的组织者、工作人员和志愿者，让这一切成为一场盛会。感谢 Rodrigo Osorio 组织并管理 BSD devroom。感谢所有在 FreeBSD 展台帮忙、参加演讲、与我们交流并提出反馈的人。特别感谢 Kristof Provost 组织 FreeBSD 开发者峰会，并感谢 FreeBSD 基金会的赞助。

---

**BENEDICT REUSCHLING** 于 2009 年加入 FreeBSD 项目。2010 年获得完整文档提交权限后，他积极指导他人成为 FreeBSD 提交者。他是 BSD 认证小组的监考人，并于 2015 年加入 FreeBSD 基金会，现任副总裁。Benedict 持有计算机科学硕士学位，在德国达姆施塔特应用技术大学教授面向软件开发者的 Unix 课程。
