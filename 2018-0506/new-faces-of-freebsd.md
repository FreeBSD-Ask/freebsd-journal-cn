# FreeBSD 新面孔

作者：Dru Lavigne

此专栏旨在聚焦最近获得 commit bit 的贡献者，并向 FreeBSD 社区介绍他们。

本期聚焦：Jason Bacon（11 月获得 ports bit）、Koichiro Iwao（3 月获得 ports bit）、Sean Fagan（4 月获得 src bit）、Vincenzo Maffione（3 月获得 src bit）、Fernando Apesteguía（3 月获得 ports commit）、Tom Jones（4 月获得 src bit）和 Eric Turgeon（3 月获得 ports bit）。

## 介绍一下你自己、背景和兴趣

**Jason**：我从高中开始接触计算机，当时上了我们学校有史以来开设的第一门计算机课。我立刻被计算机编程带来的刺激挑战和即时反馈所吸引。上大学时，在最后一刻我选择了计算机科学而非化学作为本科专业。之后某种程度上顺其自然地进入了硕士项目。教了几年计算机科学后，我攻读生物学博士项目，但因家庭责任不得不退学。我内心仍是生物学家，但我的计算机科学背景很有用，我作为开发者和系统管理者找到了有回报的工作。我现在在 UW Milwaukee 支持研究计算和高性能计算，与许多领域的研究人员合作，主要是工程和基因组学。

我是坚定的操作系统和语言不可知论者，总是反对计算中的宗教式狂热。自我设限的理想主义伤害从终端用户到系统管理员的每个人。我总是开放地看待替代方案，选择最高效的工具。有时是 BSD，有时是 Linux，有时是 Windows。

我也从小参与体育，包括高中 4 年的网球和越野跑，以及大学网球队。大学以来，我转向中国武术、自行车、XC 滑雪和海上皮划艇。（我住在密尔沃基南部，离密歇根湖畔公园绿地几条街。）

哲学上，最接近我的标签是佛教/道教。我信奉完全的开放、自我消除，任何愤怒的暗示都是指向个人成长空间的路标。

**Koichiro**：我出生在日本西南的九州岛，现在仍在九州。我是 xrdp 的上游开发者，xrdp 通过 Microsoft Remote Desktop Protocol 为类 UNIX 操作系统提供到远程客户端的图形登录。我在一个 xrdp 项目上工作，改善 FreeBSD 兼容性并处理非 ASCII 字符如 CJK。我也对桌面虚拟化感兴趣。你应该已经知道 GNU screen 和 tmux 有多有用。桌面虚拟化将提供 GUI 版的 GNU screen。你可以从任何地方通过网络恢复现有的远程会话。

**Sean**：从决定用 C 写一个 PDP-11 模拟器而不是使用过载的 RSTS/E 系统来做大学汇编语言课程作业时起，我就对类 UNIX 系统感兴趣。我从大学到 The Santa Cruz Operation 工作，大部分时间在 XENIX 上移植 Microsoft 的 C Compiler。我玩了一些内核方面的东西（添加简单形式的 ACL，只是为了看看能不能做到），当 SCO 推出基于 SysVr3.2 的系统时（为了 POSIX 合规性支持作业控制），我开始让它工作（这也涉及从 BSD 移植 tcsh 到 SysVr3.2 并让 emacs 在其上工作）。我把补丁发回给开源代码，也开始为当时新的 386BSD 写一些简单的底层 libc 例程。这导致我为 Cygnus 工作，意味着更直接地参与开源（虽然过了一段时间才有这个名字）。

**Vincenzo**：我是软件工程师，住在意大利比萨。目前是比萨大学最后一年博士生。我的研究兴趣包括快速用户态网络、网络虚拟化和内核网络驱动。但最重要的是我热爱编程和操作系统。业余时间我也喜欢弹钢琴、骑车和做饭。

**Fernando**：我叫 Fernando Apesteguía。我是住在马德里（西班牙）的软件开发者。获得计算机科学硕士学位后，我搬到荷兰为 ESA 的 YES2 项目工作。那非常酷，我们甚至获得了太空中部署的最长人造结构的吉尼斯世界纪录。在那里我为运行 QNX 的机载计算机编写软件。之后，我回到西班牙加入 Open Sistemas。这里，我们用开源软件开发解决方案。我通常在 Linux 中用 Qt 和 DBus 进行 IPC 做 C/C++ 编程。

在个人方面，我和女朋友住在一起，她苦于我缺乏空闲时间。我尽量多旅行和练习武术。我有空手道三段黑带，练习了 25 年，我还有其他学科如 SAMBO 和自卫的黑带。

**Tom**：我是苏格兰东北部阿伯丁大学的研究员，研究互联网传输和标准化。过去几年我一直在 EU NEAT 项目（<https://www.neat-project.org/>）中用现代且适应性强的东西替换 Socket API。有时我需要从计算机后面走出来，把它搬到当地的黑客空间 57North Hacklab。其他时候，如果天气好，我会走得更远，在帐篷里设置计算机。

**Eric**：我是 GhostBSD 的创始人，去年 9 月开始作为 QA 部门的自动化工程师为 iXsystems 工作。我娶了一位名叫 Karine 的好女人，她对我花时间在 GhostBSD 上非常耐心，我有一个去年一月满六岁的儿子 Samuel。像他父亲一样，他喜欢计算机和技术，也是 Minecraft 的狂热粉丝。我目前住在加拿大新不伦瑞克省迪耶普市。我说法语、英语和 Chiac（法语和英语混合）。

我没有完成高中学业，但我是个喜欢技术的好奇的人，对 Unix、BSD 和桌面软件充满热情。我学了 C、shell 脚本和 Python，但 Python 成为我几乎所有不需要像 C 那样扩展的事物的首选语言。我按需学习一切，所以我对 FreeBSD、Python、C 甚至我的新职业的知识随时间增长。

我喜欢和儿子玩，珍惜与妻子共度的时光。我爱读历史、编程书籍和圣经。我喜欢参与 City of Grace 教会活动，和朋友圈一起度过晚会。我喜欢在疯狂的小径上山地骑行，用 PowerBuilder 训练，但最近我在这方面偷懒了。我爱金属音乐，有时间时弹吉他。

## 你最初是怎么了解到 FreeBSD 的，FreeBSD 的什么吸引了你？

**Jason**：在 FreeBSD 存在之前我就参与了 Unix 开发，自 90 年代中期以来不间断地运行 BSD 和 Linux 系统。我喜欢 FreeBSD 在性能和稳定性方面的长期传统，以及对干净、系统化管理和软件部署的专注。FreeBSD ports 系统在规模、能力和灵活性方面鲜有对手。我可以从零开始设置一个 FreeBSD 系统，在不到一小时内服务于大多数目的，而且知道从那天起几乎不会有麻烦。

**Koichiro**：我用 FreeBSD 超过 10 年，记得 FreeBSD 5.4 或 6.0 是我的第一个 FreeBSD。我大学时开始把 FreeBSD 用作网络网关和路由器。`ipfw` 语法简单，易于理解。另外，`dummynet` 是控制流量和模拟真实网络的好工具。总之，网络功能吸引我使用 FreeBSD。

**Sean**：在 Linux 的第一个版本不断因使用 telnet 而崩溃后，我切换到 386BSD 作为个人电脑，这导致了其他 BSD，最终在 FreeBSD 开始几个月后专注于 FreeBSD。当时可分发源代码的 BSD 仍然很新，有很多让我感兴趣的工作。像我一贯的做法，我做了大量库和工具工作，以及一些补充内核代码。在上 McKusick 的 BSD 内部课程时，我决定写一个 procfs 版本，在大量帮助下完成了模板。然后 procfs 工作后，我开始写 truss；这两个仍然是我最大的个人贡献。我还为 Dr Dobb's Journal 写了一些相关文章。

**Vincenzo**：我第一次接触 FreeBSD 是在大学学习 UNIX 系统管理的课程中。然而，我第一次真正与 FreeBSD 源代码互动只是几年前，当时我参加了 谷歌 Summer of Code 项目。在这个项目中，我玩内核比特和虚拟化工具，很开心。我还有机会在一次 FreeBSD 会议上展示我的工作。我对操作系统实现有浓厚兴趣，特别喜欢 FreeBSD，因为其干净的设计和出色的文档。

**Fernando**：我第一次听说 FreeBSD 是在高中。我的一个朋友是早期 Linux 用户，他告诉我 FreeBSD。不久后，我买了一期西班牙版 PC World，其中附带了 FreeBSD 4.2。它在我的硬件上运行得不太好，但我能安装并玩一段时间。之后我升级到 5.0，但仍更常用 Linux。我想大约 7.0 时我成为主要的 FreeBSD 用户。我被其稳定性打动，这种感觉随时间增长。我仍有一台 12 年历史的笔记本，只有 1 GB 内存，运行 FreeBSD 11.0。

**Tom**：我 2008 年 distro hopping 时在 G4 iBook 上运行 PowerPC FreeBSD，但我的冒险真正开始于 2018 年我加入阿伯丁大学研究组时。当时该组正在标准化一种称为 NewCWV（RFC7661）的 TCP 修改。我们有 Linux 的实现，IETF 标准化信条是“rough consensus and running code”，如果有尽可能多平台的实现会很有帮助。有人谈论移植到 BSD 操作系统，我要求参与。移植工作从我的桌面虚拟机开始，但很快我获得了一些二手服务器来构建和运行 FreeBSD。FreeBSD 因其著名的优秀 TCP 协议栈成为移植的首选操作系统。很快我开始怀念 FreeBSD 开发机器上可用的工具，决定必须把 FreeBSD 用作桌面。

**Eric**：我那段旅程始于厌倦 Windows XP 上的病毒，想成为黑客。我切换到 PC Linux OS，搬到 Mandriva，然后到 Ubuntu，感觉有点像在家。在我成为臭名昭著的黑客的征途上，大约在同一时间我发现了 Eric Steven Raymond 的“How To Become A Hacker”，这是一个改变人生的事件，让我停止了成为黑客的旅程。“How To Become A Hacker”简要谈到 BSD Unix，这引起了我的注意。我开始 谷歌 BSD，这开始了我通向 FreeBSD 的道路。我安装了 FreeBSD 7.0，比 XP 更容易安装，但 shell 是我的克星。所以，我重装了 Ubuntu，做了更多研究，找到并尝试了 PC-BSD 1.4。我不喜欢 KDE，所以我在磁盘的一半重装了 Ubuntu，打印了整个 FreeBSD 手册和文档来安装和设置 Gnome，在另一半重装了 FreeBSD。这就是我对 FreeBSD 所有的爱开始的地方，我开始 GhostBSD 以拥有 PC-BSD 的 Gnome 等价物。

今天我对 FreeBSD 的主要兴趣是 GhostBSD 以及我所有只运行 FreeBSD 的服务器和 VPS。我对 FreeBSD 桌面端比作为服务器的 FreeBSD 更感兴趣，这就是 GhostBSD 开始的原因。

FreeBSD 和 Eric Steven Raymond 的文章开始了我成为程序员的道路，从我在 IRC 上建立的所有联系中，我在 iXsystems 找到了一份很棒的职业。

## 你是怎么成为 committer 的？

**Jason**：我从 2004 年左右开始维护 ports，当时我发现 FreeBSD 为我们在威斯康星医学院的 fMRI 研究解决了很多稳定性问题。我目前维护几十个活跃的 ports，工作进度集合中还有两百多个。

我被邀请成为 committer 两次。第一次是几年前我生活非常忙碌的时候，我勉强不得不拒绝。第二次大约一年前，我仍然很忙，但事情朝着正确的方向发展，所以我咬紧牙关接受了。

大约同一时间我也成为 pkgsrc 开发者，比较两个项目的优缺点很有趣。我希望在未来几年帮助弥合两者，使双方受益。我们广泛使用 pkgsrc 在 CentOS 系统上安装最新的开源软件包。

**Koichiro**：我多年来一直在为 FreeBSD 项目做贡献，特别是作为 xrdp 的 port 维护者。有一天，我自愿成为 ports committer。Hiroki Sato（一位日本 FreeBSD 核心开发者）提名我为 ports committer，最终我成为了 committer。

**Sean**：我在早期就曾是 committer。然而，2001 年我开始在 Apple 工作，可以说，他们不赞成我继续那样做。所以，有十年期间我几乎没做任何公开可见的事。离开 Apple 后，我去了 iXsystems，在那里我第一次全职与 FreeBSD 工作。然而，前几年我做的大部分涉及非操作系统代码（主要是安装程序和更新程序）。偶尔，我会遇到一些我想做的事情需要操作系统更改，其中大部分我通过邮件发回给最后修改相关文件的人。有些补丁目前仍在 iX，需要回去。

此外，我开始将一些 ZFSOnLinux 的更改移植回 FreeBSD，这些更改太大，无法通过邮件作为简单补丁发送，所以在 Alexander Motin 和 Kris Moore 的帮助下，我拿回了我的 commit bit（仍在试用期）。但我非常期待尽快整理好 ZFS 扫描代码并完成原生 ZFS 加密代码的移植。

**Vincenzo**：我是开源 Netmap 项目的维护者，该项目为应用提供从用户态执行快速网络 I/O 的 API。Netmap 在 FreeBSD 和 Linux 上运行，其源代码目前托管在 GitHub 上。虽然 Netmap 已经包含在 FreeBSD 树中，但其中的代码并没有真正维护，且相对于上游版本不断过时。因此，在上次 AsiaBSDCon 上我被邀请成为 committer，以保持 FreeBSD Netmap 代码的良好状态并与上游对齐。我非常高兴地接受了。

**Fernando**：成为全职 FreeBSD 用户后，我开始贡献一些 port PR。2011 年，我发了几个 PR，移植了 wiki 的 WantedPorts 页面中列出的一些 ports。这是一件有趣的事，让我学到了很多，得益于所有提交我补丁的人的提示。我甚至在母校（Universidad de Valladolid）做了几次关于 FreeBSD 和如何创建 ports 的演讲。一段时间后，我觉得应该更深入地参与项目，所以我开始用“嘿，如果有人能指导我完成导师过程，我愿意提升”之类的话结束所有 PR。一段时间后，tz@ 回应了我的请求。tz@ 和 tcberner@ 是我耐心的导师。我仍在愉快地学习如何成为一名好的 ports committer。

**Tom**：去年在布拉格的 IETF 上，约我吃午餐的人放了我鸽子，反而邀请我加入一位同事和 Netflix 团队。吃鲁本三明治时，我和 Jonathan Looney（我的导师）聊起我迄今为止在 FreeBSD 上做的工作。几周后在剑桥的 BSDCam，Jonathan 问我是否有兴趣成为 committer。

实际上，直到 3 月伦敦 IETF 会议我才和 Jonathan 说话并询问启动流程需要哪些步骤。两周后我收到了 Core 的出色邮件。

**Eric**：我有时间时尝试帮助 Koop 处理 MATE 和 Gnome。当我在 iXsystems 开始工作时，我更活跃一些，试图在 ports 中推出 MATE 1.20，但 Koop 没空，所以 Baptiste 提供了 commit bit，我说好。由于 Baptiste 和我在不同时区，Baptiste 让我在同一时区有了第二位导师。我在工作聊天中提到这一点，William Grzybowski 主动提出成为我的共同导师。我被投票成为 ports committer，其余就是待书写的历史了。

## 加入 FreeBSD 项目后体验如何？对想成为 FreeBSD committer 的读者有什么建议？

**Jason**：一直很难找到时间最终掌握我的 porting 技能，但在我导师（@jrm）的帮助和 @mat 等人的重要反馈下，我逐渐掌握了，感觉现在很接近了。导师计划和完善发展的工具（poudriere、phabricator、portlint、stage-qa 等）确实帮助优化了学习过程。

不要犹豫加入团队。你投入向其他开发者学习的时间是一项长期回报丰厚的投资。你将从许多前辈的丰富经验中学习如何更高效地解决问题，并产生不太可能崩溃、不会在未来给你制造更多工作的解决方案。这是在世界级优秀开发者团队中磨练技能的绝佳机会。

**Koichiro**：到目前为止没什么变化，除了我可以自己提交更改。我继续像以前一样贡献给 ports 树。如果你想成为 committer，继续贡献并建立良好的记录。当你自愿成为 FreeBSD 开发者时，这会证明你有足够的经验和知识。

**Sean**：读提交日志，读邮件列表。提问。提交代码——无论是补丁还是应该加入系统的新代码。找导师。学 subversion。

**Vincenzo**：我成为 committer 不到两个月，所以我经验不足，无法给出有意义的建议。然而，我觉得 FreeBSD 是一个非常友好的社区，对新想法和新人开放。这是我非常欣赏的，因为这在开源社区中并不常见。我认为为开源项目做贡献最令人兴奋的事是看到自己的工作成为被成千上万用户使用的东西的一部分。这绝对吸引了我，我认为对于那些考虑成为 FreeBSD committer 的人来说可能很有趣。

**Fernando**：到目前为止，很棒！人们非常欢迎和友好。通过 Phabricator 的同行审查非常好，总是值得使用。所有开发者对工作的细节关注让我印象深刻。仅仅能用还不够，应该以最好的方式工作。关于建议：订阅你感兴趣的邮件列表（如 freebsd-ports@）。阅读你能读到的所有消息，你会学到很多。之后，你可以发一些补丁。对于 Ports Collection，你可以收养一个孤儿 port、提交新 port 或只是发送对现有 ports 的更新。对于其他子系统如 base，查看 bugzilla 的数据库，找到你能帮忙的事情总是好的。另外，不要害怕提问！

**Tom**：社区非常欢迎。对我来说，加入项目是一段漫长的旅程。写代码只是第一部分，让代码被上游接受需要同样甚至更多的努力。最大的进步来自与人面对面见面。去年我在 FOSDEM 遇到 Sevan Janiyan（sevan@），他邀请我参加 BSDCam。你的第一次贡献应该小；邮件列表上没有潜伏的怪物等着喊倒你的 diff。所以，修复一些东西、一个命令、man 页面或一些文档，参与进来！

**Eric**：到目前为止还不错。我对最近学到的一切感到满意，在 Baptist、William 和 Koop 的帮助下，我在创建和维护 ports 方面越来越好。我在 ports 中发布了 MATE 1.20。我还有更多要学，由于我参与了很多项目，一切进展缓慢。对于任何有兴趣参与 FreeBSD 的人，参与你感兴趣的项目的任何部分都很简单。加入论坛、FreeBSD IRC 频道和与你兴趣相关的邮件列表。开始帮助和工作，把你的工作发送给相关的人，迟早你会得到 commit bit。

---

**DRU LAVIGNE** 是 FreeBSD 项目的 doc committer，BSD Certification Group 主席。
