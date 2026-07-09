# 基金会来信

来自编辑委员会的信

“FreeBSD 不就是个 Linux 发行版吗？“

这是从 Linux 世界来到 FreeBSD 的人常问的问题。本刊读者都知道答案是”不”，但他们也明白这并不是完整的回答。FreeBSD 不仅是另一套代码库，更有着不同的哲学和社区。本期我们将直面这一问题，通过一组文章证明：FreeBSD 不是 Linux 发行版。

第一篇文章由我这位谦卑的主编亲笔撰写，其标题最早出现在 2015 年初。当时我受邀为 Digital Ocean 的一群工程师做一场关于 FreeBSD 的演讲。我们许多从事 FreeBSD 工作的人都回避提及那个开源操作系统，但我觉得是时候正视这一话题了。那次演讲被录制下来，可在 <https://www.youtube.com/watch?v=wwbO4eTieQY> 观看。第一次演讲之后，许多人邀请我去其他场合做这个演讲，并提供更多更新。后来，FreeBSD 基金会董事 Philip Paeps 以他自己的风格，在今年的 Rootconf 上做了这一演讲的更新版本（<https://www.youtube.com/watch?v=ps67ECyh0sM>）。当我们开始筹备这期 FreeBSD vs. Linux 主题时，显然需要把这个演讲变成一篇文章。

虽然两个系统差异的概述很有趣，但我们想更深入探讨 FreeBSD 为何不是 Linux。Allan Jude 撰写的《FreeBSD vs. Linux: ZFS》介绍了唯一开源、得到良好支持、可用于 PB 级存储的文件系统。随着 Linux 上 Btrfs 的衰落，唯一可行的替代方案就是 OpenZFS，而 OpenZFS 在 FreeBSD 上运行得非常完美。Jonathan Anderson 带我们了解 Unix 系统上用于沙箱化的各种技术与方法，并特别介绍 FreeBSD 原生的能力系统 Capsicum。而 Kirk McKusick（曾协助定义项目早期治理结构）和 Benno Rice（现任核心团队成员之一）合作撰写了一篇权威文章，讲述 FreeBSD 项目是如何运作的。

与一群 FreeBSD 开发者坐在会议午餐或晚宴上，提起 Linux 缺少 Control-T（即 SIGINFO）这件事，你会听到一片响亮的合唱：他们都对 Linux 上缺少这一功能感到非常恼火。看似只是一个小差异，但对实际编程的人来说，这真的重要。Benedict Reuschling 撰写了一篇精彩的短文，介绍这一重要功能。最后，Dave Cottlehuber 撰写了一篇关于微服务实践的文章，为本期主题文章收尾。微服务如今风头正劲，而在 FreeBSD 上实现起来也很容易。Dave 告诉我们如何去做。

我们相信，当你读完本期后，如果同事再问”FreeBSD 不就是个 Linux 发行版吗？“，你将能给出明确的回答。

George Neville-Neil，FreeBSD 基金会董事会主席
