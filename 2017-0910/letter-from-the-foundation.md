# 基金会来信

来自编辑委员会的信

“FreeBSD 不就是个 Linux 发行版吗？“

这是从 Linux 世界来到 FreeBSD 的人常问的问题。本刊读者都知道答案是”不”，但他们也明白这并不是完整的回答。FreeBSD 不仅是另一套代码库，更有着不同的哲学和社区。本期我们将直面这一问题，通过一组文章证明：FreeBSD 不是 Linux 发行版。

第一篇文章由我这位谦卑的主编亲笔撰写，其标题最早出现在 2015 年初。当时我受邀为 Digital Ocean 的一群工程师做一场关于 FreeBSD 的演讲。我们许多从事 FreeBSD 工作的人都回避提及那个开源操作系统，但我觉得是时候正视这一话题了。那次演讲被录制下来，可在 <https://www.youtube.com/watch?v=wwbO4eTieQY> 观看。第一次演讲之后，许多人邀请我去其他场合做这个演讲，并提供更多更新。后来，FreeBSD 基金会董事 Philip Paeps 以他自己的风格，在今年的 Rootconf 上做了这一演讲的更新版本（<https://www.youtube.com/watch?v=ps67ECyh0sM>）。当我们开始筹备这期 FreeBSD vs. Linux 主题时，显然需要把这个演讲变成一篇文章。

虽然两个系统差异的概述很有趣，但我们想更深入探讨 FreeBSD 为何不是 Linux。Allan Jude 撰写的《FreeBSD vs. Linux: ZFS》介绍了唯一开源、得到良好支持、可用于 PB 级存储的文件系统。随着 Linux 上 Btrfs 的衰落，唯一可行的替代方案就是 OpenZFS，而 OpenZFS 在 FreeBSD 上运行得非常完美。Jonathan Anderson 带我们了解 Unix 系统上用于沙箱化的各种技术与方法，并特别介绍 FreeBSD 原生的能力系统 Capsicum。而 Kirk McKusick（曾协助定义项目早期治理结构）和 Benno Rice（现任核心团队成员之一）合作撰写了一篇权威文章，讲述 FreeBSD 项目是如何运作的。

与一群 FreeBSD 开发者坐在会议午餐或晚宴上，提起 Linux 缺少 Control-T（即 SIGINFO）这件事，你会听到一片响亮的合唱：他们都对 Linux 上缺少这一功能感到非常恼火。看似只是一个小差异，但对实际编程的人来说，这真的重要。Benedict Reuschling 撰写了一篇精彩的短文，介绍这一重要功能。最后，Dave Cottlehuber 撰写了一篇关于微服务实践的文章，为本期主题文章收尾。微服务如今风头正劲，而在 FreeBSD 上实现起来也很容易。Dave 告诉我们如何去做。

我们相信，当你读完本期后，如果同事再问”FreeBSD 不就是个 Linux 发行版吗？“，你将能给出明确的回答。

George Neville-Neil，FreeBSD 基金会董事会主席

## 编辑委员会

- John Baldwin —— FreeBSD 核心团队成员、FreeBSD 期刊编辑委员会联合主席
- Brooks Davis —— SRI International 高级软件工程师、剑桥大学访问工业研究员、FreeBSD 核心团队前成员
- Bryan Drewery —— EMC Isilon 高级软件工程师、FreeBSD Portmgr 团队成员、FreeBSD Committer
- Justin Gibbs —— FreeBSD 基金会创始人兼主席、Spectra Logic Corporation 高级软件架构师
- Daichi Goto —— BSD Consulting Inc.（东京）董事
- Joseph Kong —— EMC 高级软件工程师、《FreeBSD Device Drivers》作者
- Steven Kreuzer —— FreeBSD Ports 团队成员
- Dru Lavigne —— FreeBSD 基金会董事、BSD 认证小组主席、《BSD Hacks》作者
- Michael W Lucas —— 《Absolute FreeBSD》作者
- Ed Maste —— FreeBSD 基金会项目开发总监
- Kirk McKusick —— FreeBSD 基金会董事、《The Design and Implementation》系列图书首席作者
- George V. Neville-Neil —— FreeBSD 基金会董事会主席、FreeBSD 期刊编辑委员会主席、《The Design and Implementation of the FreeBSD Operating System》合著者
- Philip Paeps —— FreeBSD 基金会董事、FreeBSD Committer、独立顾问
- Hiroki Sato —— FreeBSD 基金会董事、Asia BSDCon 主席、FreeBSD 核心团队成员、东京工业大学助理教授
- Benedict Reuschling —— FreeBSD 基金会副主席、FreeBSD 文档提交者
- Robert N. M. Watson —— FreeBSD 基金会董事、TrustedBSD 项目创始人、剑桥大学高级讲师

## 工作人员

- 出版人：Walter Andrzejewski（<walter@freebsdjournal.com>）
- 特约编辑：James Maurer（<jmaurer@freebsdjournal.com>）
- 文字编辑：Annaliese Jakimides
- 艺术总监：Dianne M. Kischitz（<dianne@freebsdjournal.com>）
- 办公室管理员：Michael Davis（<davism@freebsdjournal.com>）
- 广告销售：Walter Andrzejewski（<walter@freebsdjournal.com>，电话 888/290-9469）

FreeBSD Journal（ISBN: 978-0-615-88479-0）每年出版 6 期（1/2 月、3/4 月、5/6 月、7/8 月、9/10 月、11/12 月）。由 FreeBSD Foundation 出版，地址：5757 Central Ave., Suite 201, Boulder, CO 80301，电话：720/207-5142，传真：720/222-2350，邮箱：<info@freebsdfoundation.org>。版权所有 © 2017 FreeBSD Foundation。保留所有权利。未经出版商书面许可，本杂志不得全部或部分复制。
