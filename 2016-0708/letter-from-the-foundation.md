# 来自基金会的信

暑期阅读

美国东北部现在是夏天，夏天就要读暑期读物。还有什么比阅读 FreeBSD 相关内容度过这些日子更好的方式呢？本期带来几个有趣的话题，包括两篇关于 ZFS 的文章：一篇是 Michael W. Lucas 撰写的《ZFS 调优》，他是同主题的两本优秀著作《FreeBSD Mastery: ZFS》和《FreeBSD Mastery: Advanced ZFS》的合著者之一。作为笔记本电脑上改用 ZFS、用启动环境降低内核开发风险的人，我可以说，深入了解这一文件系统总能提升我的生产力。Michael 在 ZFS 书籍中的合著者 Allan Jude 撰写了一篇关于 ScaleEngine 使用 FreeBSD 的精彩文章，这是一家自 2008 年起由他运营的内容分发网络。ScaleEngine 使用 FreeBSD，特别是 ZFS，从全球分布的数据中心向世界各地的客户端提供视频和其他内容。

过去几年里，我们许多人看着 Linux 社区因 systemd 之争而分裂。幸灾乐祸固然不错，但 FreeBSD 也需要找到改进自身 init 系统的方法。Mark Heily 撰写了一篇题为《把活干完》的文章，希望能帮助 FreeBSD 以更少的喧嚣与风波渡过这片水域。

FreeBSD Journal 的许多读者熟悉作为整体的 FreeBSD——从内核到用户空间再到工具，但一直以来，包括前 Wind River 的 VxWorks 在内的各种嵌入式操作系统，都出于自身目的采用 FreeBSD 的组件。FreeBSD 中最常被复用的部分是 TCP/IP 协议栈，它是系统中较大且较复杂的部分之一，自 20 世纪 80 年代最初的伯克利软件发行版以来一直在积极开发。RTEMS 的几位开发者讨论了他们在自己的"多处理器系统实时执行体"——一种开源实时操作系统（RTOS）——中使用 FreeBSD 部分组件的情况。

本期主打内容的收尾之作是 Diane Bruce 撰写的《业余无线电与 FreeBSD》。正如 Diane 所指出的，计算能力价格的持续下降，让数字与模拟世界这一迷人的融合，几乎能被每一个对该主题感兴趣的人所触及。

与往常一样，我们的专栏作者让你了解 FreeBSD 各方面的最新动态，包括 Steven Kreuzer 的 `svn update`；Tim Moore 的会议报告；Dru Lavigne 撰写的《FreeBSD 新面孔》专题以及活动日历；最后是 FreeBSD 基金会执行董事 Deb Goodkin 提供的最新动态。希望在你读完本期夏末刊物后，会期待我们在 9/10 月刊上带来的更凉爽气候与精彩文章。

此致，George Neville-Neil

代表 FreeBSD Journal 编辑委员会

## 编辑委员会

- John Baldwin —— FreeBSD 核心团队成员
- Brooks Davis —— SRI International 高级软件工程师、剑桥大学访问工业研究员、FreeBSD 核心团队前成员
- Bryan Drewery —— EMC Isilon 高级软件工程师、FreeBSD Portmgr 团队成员、FreeBSD Committer
- Daichi Goto —— FreeBSD 基金会创始人兼主席、Spectra Logic Corporation 高级软件架构师
- George V. Neville-Neil —— BSD Consulting Inc.（东京）董事
- Joseph Kong —— EMC 高级软件工程师、《FreeBSD Device Drivers》作者
- Steven Kreuzer —— FreeBSD Ports 团队成员
- Kirk McKusick —— FreeBSD 基金会董事、BSD 认证小组主席
- Michael W. Lucas —— 《Absolute FreeBSD》作者
- Ed Maste —— FreeBSD 基金会项目开发总监
- Benedict Reuschling —— FreeBSD 基金会董事、《The Design and Implementation》系列图书首席作者
- Hiroki Sato —— FreeBSD 基金会董事、《The Design and Implementation of the FreeBSD Operating System》合著者
- Robert Watson —— FreeBSD 基金会董事、TrustedBSD 项目创始人、剑桥大学讲师

## 工作人员

- 出版人：Walter Andrzejewski（walter@freebsdjournal.com）
- 特约编辑：James Maurer（jmaurer@freebsdjournal.com）
- 艺术总监/办公室管理员：Dianne M. Kischitz（dianne@freebsdjournal.com）
- 广告销售：Walter Andrzejewski（walter@freebsdjournal.com，电话 888/290-9469）

FreeBSD Journal（ISBN: 978-0-615-88479-0）每年出版 6 期（1/2 月、3/4 月、5/6 月、7/8 月、9/10 月、11/12 月）。

由 FreeBSD 基金会出版，PO Box 20247, Boulder, CO 80308
电话：720/207-5142 • 传真：720/222-2350
邮箱：info@freebsdfoundation.org
版权所有 © 2016 FreeBSD Foundation。保留所有权利。

未经出版商书面许可，本杂志不得全部或部分复制。
