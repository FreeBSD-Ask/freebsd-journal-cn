# 书评：*DTrace: Dynamic Tracing in Oracle Solaris, Mac OS X and FreeBSD*

作者：JOSEPH KONG

- **书名：** DTrace: Dynamic Tracing in Oracle Solaris, Mac OS X and FreeBSD
- **作者：** Brendan Gregg、Jim Mauro
- **出版社：** Prentice Hall（2011 年）
- **印刷版定价：** $59.99
- **电子版定价：** $47.99
- **ISBN-10：** 0132091518
- **ISBN-13：** 9780132091510
- **页数：** 1,152

我的同事们钟爱 DTrace，时常称道它定位 bug 根因的能力。于是我买了一本 Brendan Gregg 和 Jim Mauro 合著的 *DTrace: Dynamic Tracing in Oracle Solaris, Mac OS X and FreeBSD*，好让自己跟上进度（好吧，我多少有点嫉妒同事们的 DTrace 功底）。大体上我喜欢这本书，但有几章读起来相当吃力（稍后再谈）。全书分为三部分。第一部分共两章，概述 DTrace。第 1 章《Introduction to DTrace》介绍 DTrace 究竟是什么，以及它如何工作（DTrace 是一个动态追踪框架，让你能观察系统的行为）。第 2 章《D Language》讲解如何编写 DTrace 脚本和一行式命令，并详细说明构成一个 DTrace 脚本的各个组件。读完这一部分并在网上做了若干示例后，我掌握了足够的知识，得以在工作中实际使用 DTrace。

第二部分共八章，每章描述 DTrace 可以观察的一个不同领域，分别是：

- 第 3 章 System View
- 第 4 章 Disk I/O
- 第 5 章 File Systems
- 第 6 章 Network Lower-Level Protocols
- 第 7 章 Application-Level Protocols
- 第 8 章 Languages
- 第 9 章 Applications
- 第 10 章 Databases

这几章塞满了示例和案例研究，我个人很欣赏这种以例带学的方式。它们把 DTrace 的威力展现得淋漓尽致。但是，由于大部分内容与我并无直接关联，加上每章都长得惊人，读起来颇觉乏味（我现在总算明白 No Starch 的编辑为何一再告诫我避免写超长章节了）。在我看来，这几章更适合作为参考资料。

第三部分也是最后一部分共四章，每章描述一个不归属前两部分的主题，分别是：

- 第 11 章 Security，介绍如何将 DTrace 用于实时取证、自定义审计、策略执行和安全调试
- 第 12 章 Kernel，详述如何用 DTrace 洞察操作系统内核；本章与第二部分的部分章节有所重叠
- 第 13 章 Tools，介绍一些基于 DTrace 构建的工具
- 第 14 章 Tips and Tricks，基于作者经验，提供一些高效使用 DTrace 的心得

这几章同样示例丰富，我也依然欣赏这种风格。与第二部分不同的是，这几章篇幅适中，因此内容更易消化。

总体而言，这是一本好书。仅凭第一部分就足以让你上手，第三部分也读来有趣。不过，这可能不是一本你会从头读到尾的书——全书的主体（即第二部分）更适合作为参考资料查阅。此外，《DTrace》还包含七个附录，显然也是作为参考资料设计的。

---

**JOSEPH KONG** 是广受好评的《Designing BSD Rootkits》与《FreeBSD 设备驱动程序开发》的作者，目前任职于 Dell EMC 的 Isilon 部门，担任高级软件工程师。更多关于 Joseph Kong 的信息，请访问 <www.thestackframe.org>，或在 Twitter 上关注 @JosephJKong。
