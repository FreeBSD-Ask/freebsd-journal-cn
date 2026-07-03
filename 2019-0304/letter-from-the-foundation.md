# 基金会书信

FreeBSD Journal（ISBN: 978-0-615-88479-0）每年出版 6 期（1/2 月、3/4 月、5/6 月、7/8 月、9/10 月、11/12 月）。

由 FreeBSD Foundation 出版

地址：2222 14th Street, Boulder, CO 80302

电话：720/207-5142 • 传真：720/222-2350

邮箱：info@freebsdfoundation.org

版权所有 © 2019 FreeBSD Foundation。保留所有权利。未经出版商书面许可，不得全部或部分复制本杂志。

来自董事会

出版人 • Walter Andrzejewski walter@freebsdjournal.com

主编 • James Maurer jmaurer@freebsdjournal.com

文字编辑 • Annaliese Jakimides

美术指导 • Dianne M. Kischitz dianne@freebsdjournal.com

广告销售 • Walter Andrzejewski walter@freebsdjournal.com

致电 888/290-9469

P.O. BOX 408, BELFAST, MAINE 04915

## 编辑委员会

- John Baldwin • FreeBSD 开发者、FreeBSD 核心团队成员、FreeBSD Journal 编辑委员会联合主席。
- Brooks Davis • SRI International 高级计算机科学家、剑桥大学访问工业研究员、FreeBSD 核心团队成员。
- Bryan Drewery • EMC Isilon 高级软件工程师、FreeBSD Portmgr 团队成员、FreeBSD 提交者。
- Justin Gibbs • FreeBSD 基金会创始人、FreeBSD 基金会董事会董事、Facebook 软件工程师。
- Daichi Goto • BSD Consulting Inc.（东京）董事。
- Joseph Kong • Dell EMC 高级软件工程师、《FreeBSD Device Drivers》作者。
- Steven Kreuzer • FreeBSD Ports 团队成员。
- Dru Lavigne • iXsystems 存储工程总监、《BSD Hacks》和《The Best of FreeBSD Basics》作者。
- Michael W Lucas • 《Absolute FreeBSD》作者。
- Ed Maste • FreeBSD 基金会项目开发总监。
- Kirk McKusick • FreeBSD 基金会董事会司库、《The Design and Implementation》系列书籍主要作者。
- George V. Neville-Neil • FreeBSD 基金会董事会主席、《The Design and Implementation of the FreeBSD Operating System》合著者。
- Philip Paeps • FreeBSD 基金会董事会秘书、FreeBSD 提交者、独立顾问。
- Hiroki Sato • FreeBSD 基金会董事会董事、Asia BSDCon 主席、FreeBSD 核心团队成员、东京工业大学助理教授。
- Benedict Reuschling • FreeBSD 基金会董事会副主席、FreeBSD 文档提交者、FreeBSD 核心团队成员。
- Robert N. M. Watson • FreeBSD 基金会董事会董事、TrustedBSD 项目创始人、剑桥大学高级讲师。

北半球大部分地区春意盎然，而在春天，年轻开发者的心思也会转向调试和测试。如果你对此存疑，那么本期刊登的三篇精彩文章足以让你改观。

作为佐证，我们奉上：John Baldwin 的《Debugging with GDB》、Kristof Provost 的《The Automated Testing Framework》以及 Benedict Reuschling 的《Diagnosing Excess LDAP Connections Using DTrace》。虽然许多开发者仍在用 `printf()` 调试，但你不必继续这样过活——John Baldwin 带我们领略了 GNU 调试器 gdb 最新版本（8.3）的功能，并展示了它如何为我们节省数小时追踪 bug 的挫败时间。

防止代码出 bug 的最佳手段是好的测试，这正是 Kristof Provost 在文章中向我们阐述的内容。三篇中的第三篇是 Benedict Reuschling 的实践经历文章，讲述在运行中的系统上用 DTrace 调试问题。

本期春季刊还带来了全新的专栏阵容。Dru Lavigne 介绍了新 Ports 提交者 Kai Knoblich；Michael W Lucas 回答了一些读者来信；Steven Kreuzer 在 svn Update 中向我们介绍源代码树中的最新动态。FreeBSD 基金会执行董事 Deb Goodkin 带来了 FOSDEM 报告——这是 2019 年首场大型开源会议，吸引了来自欧洲和世界各地逾 5000 名开源项目参与者，在比利时度过一个周末的会议、演讲和聚会。说到会议，如果你有兴趣参加 FreeBSD 相关活动，可以在由 Dru Lavigne 和 Anne Dickison 整理的活动日历中找到清单。

我们期待在 5 月中旬于加拿大安大略省渥太华举办的 BSDCan 上见到尽可能多的读者。大多数编辑委员会成员都会到场，如果你在那里看到我们，请停下来打个招呼，告诉我们 FreeBSD Journal 做得如何。

GEORGE NEVILLE-NEIL
