# 基金会致信

来自编辑委员会的信

欢迎来到 FreeBSD 期刊的夏末刊——至少从我所在的英国剑桥来看是夏末。下周就是 BSDCam，这是每年 8 月学生离校、教授们撒欢时举办的 FreeBSD 开发者峰会——撒欢的程度仅限教授们能撒到的范围。Summit 结束后我们会有更多内容分享，但现在先来看看本期为你准备了什么——你在海滩或任何阅读期刊的地方放松时可以读一读。

过去一两年，读到关于虚拟化和操作系统的内容，几乎不可能不遇到 Docker。任何新技术都会引来下一个理所当然的问题：“FreeBSD 呢？“Kurt Lidl 用一篇关于 Docker 与 FreeBSD 的概述来回答这个问题，不仅涵盖现状，也展望了 FreeBSD 与 Docker 的未来。

接下来，FreeBSD 期刊与 FreeBSD 基金会董事会成员 Benedict Reuschling 采访了 TrueOS 项目的 Joe Maloney，谈谈 TrueOS 最新发行版中部署的新 OpenRC 系统。

Pawel Jakub Dawidek 与 Milosz Kaniewski 讲述他们如何用 FreeBSD 进行网络流量分析，深入 TLS 加密流。许多人对 bhyve——FreeBSD 原生 hypervisor——和 ZFS——这一泽字节文件系统——并不陌生。Trent Thompson 撰文介绍 iohyve，它把这两项技术结合在一起，让你的系统部署工作更轻松。

本期还以三个专栏收尾：New Faces——Dru Lavigne 与两位新 Ports 提交者对谈；svn Update——Steven Kreuzer 带我们看看 FreeBSD 中所有崭新且闪亮的内容；以及 Roller Angel 撰写的 BSDCan 会议报告。想知道接下来 FreeBSD 还有哪些活动？Dru Lavigne 的活动日历会告诉你 FreeBSD 人在全球何处相聚。

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
