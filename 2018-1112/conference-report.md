# 会议报告

**作者：Roller Angel**

## EuroBSDcon 2018——罗马尼亚布加勒斯特

## 会议前

九月，我来到布加勒斯特参加 EuroBSDcon 2018，这是我第一次在地球另一端参加 BSD 会议！我的提案——围绕 BSD 技术的教授、学习与信心培养——被接受后，我立刻开始为这个看似一生一次的机会做安排，但如今我明白，这是一扇打开的门。

## 周三

我于罗马尼亚时间周三早晨抵达，此时仍是科罗拉多时间周二深夜。讽刺的是，演讲者下榻的酒店名字非常贴切——Hotel Yesterday（昨日酒店）！由于时间还早，我在酒店准备房间办理入住期间，在布加勒斯特街头漫步观光。这座城市历史悠久，人口约 210 万；纽约市人口 860 万。但人口密度大致相同。

## 周四

我从酒店步行前往会议地点 Politehnica University of Bucharest（布加勒斯特理工大学），这是罗马尼亚最大的技术大学，1864 年作为桥梁、道路、矿山与建筑学校创立，1920 年更名。一位名叫 Pedro 的与会者是我遇到的第一人。我们同时办理注册，然后坐在一起。我们立刻开始讨论各自的 FreeBSD 笔记本，分享各种技巧。例如，Pedro 演示了如何轻松地通过 USB 将 Android 的网络共享到 FreeBSD：启用 USB 网络共享，插上电缆，运行 `dhclient` 后跟设备名（`ue0` 或类似设备），等待几秒钟，会议 WiFi 的互联网就通过 Android 的 USB 共享连接到 FreeBSD。Pedro 在笔记本上运行 FreeBSD-CURRENT，我在我的笔记本上运行 FreeBSD-RELEASE。Benedict Reuschling 加入后，我们一起走向他的 Ansible 教程。

Reuschling 的教程充满了实用示例，在演示如何以实用方式使用系统各部分的同时，深入讲解各种组件。我很高兴能学到这么多管理服务器的技术。

接下来，我参加了 Niclas Zeising 主讲的 Poudriere 教程，他精彩地介绍了 Poudriere，演示了编译 Ports 时可能用到的一些功能。他还带领参与者逐步创建 port。教程参与者受邀参加 Devsummit 晚宴，于是我决定参加，结果与 Deb Goodkin 和 Benedict Reuschling 同桌。晚宴后不久，Mahdi Mokhi 加入我们的桌子，我们讨论了我用 FreeBSD 和 Python 所做的工作。

## 周五

Pedro 又出现了，这次我们讨论了为我的 BSD.pw 域名搭建邮件。他推荐 caesonia 邮件服务器，使用 OpenBSD，由 vedetta 防火墙的同一批人开发。然后是周五的全天教程，由出色的 Peter Hessler 讲授 BGP。这是多年反馈与迭代的成果，因此我体验到了教程的高级版本，包含带两个并排终端的 Web 界面，全部就绪。Peter 不是让每位教程参与者安装两台配置为运行 BGP 的 OpenBSD 虚拟机，而是在他的笔记本上为每位学生托管两台 OpenBSD 虚拟机，笔记本连接到 EuroBSDcon WiFi，他在那里设置 URL **bad.network**。每位参与者拿到一张纸条，上面有访问特定一对机器所需的 URL、用户名和密码，只需使用 Web 浏览器即可。这种方式效果极好，整个房间的人都能快速启动运行。教程涵盖网络、子网和静态路由信息，这些是理解 BGP 及其在互联网上重要作用的先决条件。BGP 全称是 Border Gateway Protocol（边界网关协议），设计目的是允许 AS 路由器——即自治系统——共享路由表。它基本上允许互联网上的计算机相互找到对方以建立连接。我愉快地学习了 BGP，与其他参与者共享路由表，并了解了过滤对端通告路由的重要性。Hessler 让参与者共享路由，然后演示了不使用适当过滤时会发生什么。对端可以通告每一个地址——是的，每一个地址——让你的机器处理陷入疯狂。大家都感到愉快，度过了美好时光。我向任何想学习 BGP 的人强烈推荐这个教程。

全天教程结束后，我前往 Devsummit 晚宴。我坐在 Marshall Kirk McKusick 旁边，整晚都在和他交谈。事实上，他的丈夫在晚上结束时告诉我，我垄断了 Kirk，因此我要负责确保他安全回到酒店。哎呀！我想我活该。我没办法！坐在这位革命性的 FreeBSD committer 旁边，是我提出许多问题的机会。那真是一次难忘的经历。

## 周六

早晨公告由 Michai Stanek 主持，然后他介绍了来自 Politehnica University of Bucharest 的同事 Costin Raiciu。Costin 给出了关于“Lightweight Virtualization”（轻量级虚拟化）的早晨主题演讲。他展示了关于该主题的一些有趣研究，以及在机器上运行非常小的 Python 环境的示例。

接下来是 Tom Jones 的“Hacking Together a FreeBSD Presentation Streaming Box for as Little as Possible”（用最少成本拼凑 FreeBSD 演示流媒体盒），提供了一些如何录制和流式传输 BSD 会议幻灯片及演讲者声音的有用示例。会议设计旨在短时间内提供大量多样化信息，因此接着我参加了 Kristaps Dz 的“OpenBSD and Diving”。这个演讲不仅幽默，充满精美照片，还充满了关于使用 OpenBSD 管理和编辑媒体的大量事实。所有媒体传输、编辑和颜色校正都在 OpenBSD 上完成。Kristaps 询问是否有人对好的媒体管理应用程序有建议，我推荐了 Plex。我告诉他我只见它在 FreeBSD Ports 中可用，但他向我保证那不是问题，他会看看能否移植，看是否适合自己的工作流。我觉得可以，因为我每天都在用，它在消除媒体管理的大量复杂性的同时，提供了稳定的媒体处理界面。

Ingo Schwarze 在关于使用 Markdown 的一些头痛问题的演讲中，请求帮助解决控制 OpenBSD 网页上 man 页面呈现的 mandoc.css 文件中的 CSS 问题。我熟悉 CSS，主动提出查看该问题，看看能否帮忙。我看到代码的那一刻就识别出了问题，知道修复需要做什么。我只需要找时间坐下来写代码。Ingo 向我保证我可以慢慢来，有空时联系他。这里不仅有吸收信息的好机会，还有参与公共对话的机会。

下一个演讲是“What's in Store for NetBSD 9.0”，Sevan Janiyan 分享了 NetBSD 项目中已达成的里程碑，包括下一版本中将包含的内容。我强烈推荐查看他的幻灯片。所有会议幻灯片都可通过 EuroBSDCon.org 网站下载。第二个主题演讲“Some Computing and Networking Historical Perspectives”由 Ron Broersma 主讲，对在场所有人来说都是真正的享受。Ron 带来了他早期职业生涯中的各种历史物品，包括他在 Arpanet 上的工作。整个礼堂都喜欢这个演讲，你能感受到房间里每分钟兴趣都在增长。Ron 会时不时从讲台下取出一件物品——例如，一张旧启动卡，过去用于当时使用的昂贵大型计算机。你实际上需要使用这张卡来知道在内存中将程序设置在何处才能启动它。你必须手动输入代码，让一切对齐，这样启动才能正常工作！演讲后，Ron 欢迎人们上前查看他带来的物品。

## 周日

会议最后一天，我有点迟到，但匆忙赶到会场，赶上了 Bob Beck 的“Pledge and Unveil in OpenBSD”。Bob 对操作系统开发的内部机制了如指掌，介绍了一些最佳实践，以及在 OpenBSD 上使用 pledge 和 unveil 编写安全程序的各种方法。

Yang Zheng 的“Integrate libFuzzer with the NetBSD Userland”是一场非常高级、技术性很强的演讲，我完全听不懂。也许有一天我能理解这个主题。

Pierre Pronchery 的“DeforaOS, NetBSD, Future Internet”是一场很有思想性的演讲。Pierre 是一位深思熟虑的开发者，挑战极限，对底层技术有出色的理解。我学到了很多东西，了解到事物为何如此，并受到挑战，要让事物变得更好。

Maya Rashish 的“Debugging Lessons Learned as a Newbie Fixing NetBSD”跟随一位新手的视角，了解为 NetBSD 贡献的过程，详细介绍了新加入项目的人应该注意的一些事情。Maya 的演讲结束后，我赶上了 Niclas Zeising 的“FreeBSD Graphics”结尾部分，我询问了编译 drm-stable-kmod 过程中的一个问题。Makefile 中有一个 ignore 标志，如果想在 FreeBSD 11.2-RELEASE（我使用的版本）上编译 port，需要移除该标志。Niclas 说这肯定是疏忽，在 11.2 上应该能正常工作。我告诉他我从 Makefile 中移除了 ignore 标志，并在我的 11.2-RELEASE 笔记本上编译了 drm-stable-kmod，它一直运行良好。

接下来轮到我演讲：“Being a BSD User”（作为 BSD 用户）。开始前几分钟，肾上腺素飙升。尽管我对公开演讲感到紧张，但演讲进行得相当顺利。观众积极参与并提出问题。我分享了我使用 BSD 技术的经验，以及向年轻科学家教授该技术是怎样一种很好的体验。学习用 BSD 能做的所有酷事，以及了解这个令人惊叹的社区，是我乐于分享给他人的体验。我收到了 Allan Jude、Kirk McKusick 等人的很好反馈。虽然我最初有点担心需要填满 45 分钟的时间块，但一旦开始演讲，我的热情接管了一切，毫无问题地填满了时间。共有来自 37 个国家的 181 名参与者。Groff 由 Sevan Janiyan 带到会议，在闭幕会议上交给 Deb Goodkin 照看。

---

**Roller Angel** 是一位热衷于 BSD 的用户，享受 BSD 技术所能完成的所有令人惊叹的事情。他基于 FreeBSD 教授过编程工作坊，正在搭建一个在线培训平台，用于教授 BSD 及相关技术。详见 BSD.pw。
