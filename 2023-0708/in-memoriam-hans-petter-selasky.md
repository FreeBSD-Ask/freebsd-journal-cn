# 悼念：Hans Petter Selasky

- 原文链接：<https://freebsdfoundation.org/wp-content/uploads/2023/08/gallatin_memoriam.pdf>
- 作者：DREW GALLATIN
- 译者：ykla

悼念：Hans Petter William Sirevåg Selasky

FreeBSD 社区近期因一位最多产的贡献者的不幸离世而深感悲痛。我们得知，Hans Petter Selasky 于 2023 年 6 月 23 日在挪威利勒桑德的一场交通事故中去世，享年 41 岁。Hans 极其聪明、为人善良，为 FreeBSD 贡献了许多宝贵成果。他的父亲 Gordon 先于他去世，身后留下母亲 Inger Elisabeth、兄弟 Mark 和 Leif Conrad，还有侄女和侄子 Petra、David 和 Signe。

Hans 大约在 25 年前开始为 FreeBSD 贡献代码，最初是修复 FreeBSD 的 ISDN 支持。他担任 FreeBSD 提交者近 15 年，最为人所知的是重写和维护 USB 协议栈。Hans 编写了 webcamd 包，该包支持在 FreeBSD 用户空间运行 Linux 摄像头驱动程序，使我们这些在桌面上使用 FreeBSD 的人能够参与现代视频会议。最近，他在 Mellanox（现为 NVIDIA）工作，负责在 FreeBSD 上支持其 ConnectX 系列高速网卡。Hans 的工作包括对内核 TLS 框架的重大贡献，以及对 `mce(4)` 驱动程序中网卡 kTLS 发送和接收卸载的支持，还有对 Linux 设备驱动兼容层的许多改进。

照片由 Ollivier Robert 提供。

我第一次见到 Hans 是在 2015 年，当时他正在为 Mellanox 网卡开发 `mce(4)` 驱动程序。我们合作使 `mce(4)` 驱动程序成为 FreeBSD 中性能最高的网卡驱动程序之一。正是在这段时间里，我见识到他的才华。他经常有些听起来"疯狂"的想法，实则精妙。有一例：他提出使用网卡提供的 RSS 流标识符对传入的 TCP 数据包进行排序，以便将来自同一 TCP 连接的所有数据包连续交给 LRO。这个想法我最初认为不切实际而未采纳，但对于 Netflix 实现从单台服务器提供 100Gb/s 视频流量的性能目标至关重要，并且持续为 Netflix 节省大量 CPU 资源。

Hans 善良、好客。我第一次参加 EuroBSDCon 是在 2019 年挪威利勒哈默尔，Hans 坚持要尽地主之谊。Hans 和他的父亲从家乡格里姆斯塔开车穿越挪威到利勒哈默尔参加 EuroBSDCon，带我参观了奥林匹克跳台滑雪场和镇上的其他几个景点。然后他带我去吃晚餐，回到他和父亲租的房子里，畅谈一晚。

在 FreeBSD 之外，Hans 的爱好有音乐和数学。他在教会中很活跃，并为音响团队服务。他充满爱心、尽职尽责，疼爱他的侄女和侄子。他热爱动物，尤其是他的猫 Pumba。

即使你自己不使用 FreeBSD，Hans 的工作也很可能影响着你的日常生活。例如，如果你使用 Playstation，你很可能在使用 Hans 编写的 USB 协议栈。如果你观看 Netflix，你正在看的节目很可能是通过运行 Hans 编写的 `mce(4)` 驱动程序的 ConnectX 网卡传递给你的。

Hans，如果你正在阅读这篇文章，请知道我们会思念你。

作者：Drew Gallatin
