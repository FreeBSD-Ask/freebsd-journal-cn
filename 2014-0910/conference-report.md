# 会议报告

- 原文标题：Conference Report — EuroBSDcon 2014
- 作者：**Daniel Peyrolón**

EuroBSDcon 是欧洲首屈一指的开源 BSD 操作系统会议，吸引了来自欧洲及世界各地的高技能工程专业人员、软件开发者、计算机科学学生、教授和用户。会议的目标是交流 BSD 操作系统知识，促进用户和开发者之间的协调与合作。2014 年 9 月 27 日与 28 日在保加利亚索菲亚举行的主会议之前，先是为期两天的 FreeBSD 开发者峰会和教程。

开发者峰会非常棒！一如既往，开发者从面对面交流和讨论中受益，他们还享受参与 FreeBSD 未来决策的机会。会上还有出色的工具演示。其中令我印象深刻的是 Phabricator，这个工具将为项目的代码审查提供极大帮助。此外还有关于包管理器的讨论，pkg 1.4 将进一步改进——其中一项改进允许在将基本系统作为软件包集合安装时实现完全的粒度控制。

与会者非常关注的是，FreeBSD 将推动嵌入式架构（如 ARM 和 MIPS）的发展，目标是把这两种架构都提升到一级（Tier 1）。上游 ASLR 补丁已从 HardenedBSD 移植，使 ASLR 能在 ARM 架构上工作，并开始进行基准测试。文档团队也在努力找出薄弱环节，以便进一步提升文档的整体质量。

开发者峰会结束后，会议正式开始。首先要说的是，我特别喜欢所有主要 BSD 都参与了本次会议。会议的开场是主题演讲。FreeBSD Ports 的创建者 Jordan Hubbard 发表了一场关于 FreeBSD 未来 20 年的演讲，以及如何使其在市场上更具竞争力。

安全话题在 FreeBSD 和 OpenBSD 阵营都引起了极大关注，演讲涵盖 FreeBSD 的 ASLR 实现、LibreSSL、用 OpenBSD 测试软件正确性、OpenBSD 的 arc4random 实现以及 OpenBSD 的延迟绑定实现。NetBSD 人士也带来了一些相当新颖的项目演讲。有些并不算新颖——比如 JIT 编译的 bpf——但另一些确实很新奇——比如用 Lua 编写 NPF 脚本，或者 Andy Tanembaum 关于 Minix 的演讲，Minix 居然使用了 NetBSD 的用户态。会议上显然还有更多演讲——这清楚表明了 BSD 的强劲势头。

有趣的是，在会议最后一天，George Neville-Neil 和 Sean Bruno 成功在 Linilo（一种廉价的嵌入式 MIPS 系统）上启动了 FreeBSD。会议结束时，Atanas Chobanov 发表了关于匿名文档提交的演讲。随后，FreeBSD 基金会抽奖送出一本《FreeBSD 操作系统设计与实现》（第 2 版），谷歌则抽奖送出一台 Chromebook。

明年 EuroBSDcon 将在瑞典斯德哥尔摩举行。如果今年的会议是个参考指标，那么明年绝对值得参加。现在就开始规划吧！
