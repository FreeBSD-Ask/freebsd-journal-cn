# 书评：《DNSSEC Mastery》

- 原文：[Book Review](https://freebsdfoundation.org/our-work/journal/browser-based-edition/zfs-best-practices/)
- 作者：**Joseph Kong**

如果你已经了解 DNS，又想学习如何为它做安全加固，那《DNSSEC Mastery》正合你意。这是一本清晰精炼的指南，手把手讲解，并配以大量示例。

《DNSSEC Mastery》

Michael W. Lucas

2013 Tilted Windmill Press • ISBN 978-1484924471 • 130 页

我得先承认，自己大概不是这本书的目标读者，因为我从未、也希望永远不必用域名系统安全扩展（Domain Name System Security Extensions，DNSSEC）来加固域名系统（Domain Name System，DNS）。不过作为一名安全从业者，我读 Michael W. Lucas 的《DNSSEC Mastery》是为了对 DNSSEC 有所了解。

这本书切中要害，不浪费你的时间。如果你已经了解 DNS，又想学习如何为它做安全加固，那《DNSSEC Mastery》正合你意。这是一本清晰精炼的指南，手把手讲解，并配以大量示例。此外，全书仅一百多页，可以一口气读完（我之所以知道，是因为我自己就这么干过）。

Lucas 用自己的域名详细讲解了 DNSSEC 的工作原理、搭建方法、出问题（问题难免会出现）时的调试手段、维护方式。书中甚至有一章介绍如何把 DNSSEC 用作经过加密验证的分发机制，例如可以无需联系证书颁发机构（CA）就验证 SSL 证书。

如果非要对这本书挑点毛病（而且是个小毛病），那就是这个话题有些枯燥。这并不是 Lucas 的错。他在全书中穿插幽默笔调，很好地抓住了读者的注意力。例如第 27 页他写道：

> 如今再举着火把和干草叉去抄家伙未免不合时宜，但如果你真有这个念头，那些老旧的域名注册商倒是很值得一闹。

第 44 页又写道：

> 我在密钥名里把主机名缩写了，因为我还没自虐到那个地步。

书中随处可见这样的幽默片段，对此我十分欣赏（不过也有人不喜欢，这点也说得通）。

总之，如果你打算部署 DNSSEC，就读读这本书。它会替你省下时间，免去不少头疼。

---

**Joseph Kong** 是自学成才的计算机爱好者，涉猎漏洞利用开发、逆向代码工程、rootkit 开发和系统编程（FreeBSD、Linux 和 Windows）。他是广受好评的《Designing BSD Rootkits》和《FreeBSD 设备驱动程序开发》的作者。Joseph Kong 的更多信息请访问 <http://www.thestackframe.org>。
