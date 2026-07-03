# 基金会书信

嘘，别告诉任何人。

这是个秘密。我们编排了一整期以"安全"为主题的期刊！好吧，实际上请不要对这一期保密，因为它包含了一系列出色的技术文章，这些技术让 FreeBSD 更加安全，同时也介绍了 FreeBSD 本身如何应用于安全系统的问题。我们以 Michael Lucas 关于 FreeBSD 防火墙的文章开篇。FreeBSD 包含 PF、IPFilter 和 IPFW 三个防火墙，它们表面上实现相同的功能，但存在若干有趣的差异。接下来是关于 CADETS 研究项目的文章。通过 CADETS，我们将追踪——通过 DTrace——与审计系统及 FreeBSD 的其他组件融合，打造一个基于"透明度即谁在何时对谁做了什么"这一理念的平台，使我们能够构建整体上更安全的系统。然后 Jonathan Anderson、Stanley Godfrey 和 Robert N. M. Watson 带领我们走进沙箱的世界，通过他们关于 Capsicum 的文章——这是 FreeBSD 中原生的、基于能力的隔离技术。Capsicum 复兴了 20 世纪 70 年代首次研究的能力系统概念。能力曾被认为在计算上过于昂贵而不适用于实际系统，但 Robert N. M. Watson 在其博士论文中完成的工作表明，可以为类 Unix 系统构建既实用又高效的能力系统。James Dekker、Renato Botelho 和 Luiz Otavio O. Souza 在他们关于这一流行的、基于 FreeBSD 的开源防火墙产品的文章中，带领我们了解 pfSense 的主要安全特性。FreeBSD 可以在许多有趣的环境中找到，包括实现博彩游戏的设备。在欧洲，你会在街角酒吧和其他场所看到这些游戏。Roberto Fernández 试图比常人更多一些偏执，解释在 FreeBSD 上保护一台游戏设备需要做些什么。我们以 Ed Schouten 关于 CloudABI 的更新文章作为本期收尾。CloudABI 是 FreeBSD 的一部分，这个工具帮助创建更易测试和验证的安全软件。Steven Kreuzer 也呼应了安全主题，他评论了 Joseph Kong 的书 Designing BSD Rootkits: An Introduction to Kernel Hacking，并在他的 svn 更新专栏中涵盖了 FreeBSD 源代码树中的所有新内容。通过 Dru Lavigne 的"新面孔"栏目，我们认识了 Eugene Grosbein，一位长期贡献者，于 3 月成为 Ports Committer。也别忘了查看我们应该考虑加入日历的 BSD 相关活动。现在，坐下来，戴上你的锡箔帽，享受阅读这最新一期 FreeBSD Journal 吧。

George Neville-Neil

FreeBSD Journal 主编

FreeBSD Foundation 董事会理事

---

## 编辑委员会

- **John Baldwin** — FreeBSD Core Team 成员，FreeBSD Journal 编辑委员会联席主席
- **Brooks Davis** — SRI International 高级软件工程师，剑桥大学访问学者，前 FreeBSD Core Team 成员
- **Bryan Drewery** — EMC Isilon 高级软件工程师，FreeBSD Portmgr Team 成员，FreeBSD Committer
- **Justin Gibbs** — FreeBSD Foundation 创始人和主席，Spectra Logic Corporation 高级软件架构师
- **Daichi Goto** — BSD Consulting Inc.（东京）董事
- **Joseph Kong** — EMC 高级软件工程师，FreeBSD Device Drivers 作者
- **Steven Kreuzer** — FreeBSD Ports Team 成员
- **Dru Lavigne** — FreeBSD Foundation 董事会理事，BSD Certification Group 主席，BSD Hacks 作者
- **Michael W Lucas** — Absolute FreeBSD 作者
- **Ed Maste** — FreeBSD Foundation 项目开发总监
- **Kirk McKusick** — FreeBSD Foundation 董事会理事，The Design and Implementation 书系主要作者
- **George V. Neville-Neil** — FreeBSD Foundation 董事会理事，FreeBSD Journal 编辑委员会主席，The Design and Implementation of the FreeBSD Operating System 合著者
- **Hiroki Sato** — FreeBSD Foundation 董事会理事，Asia BSDCon 主席，FreeBSD Core Team 成员，东京工业大学助理教授
- **Benedict Reuschling** — FreeBSD Foundation 董事会副主席，FreeBSD Documentation Committer
- **Robert N. M. Watson** — FreeBSD Foundation 董事会理事，TrustedBSD 项目创始人，剑桥大学高级讲师

## 出版信息

FreeBSD Journal（ISBN: 978-0-615-88479-0）每年出版 6 期（1/2 月、3/4 月、5/6 月、7/8 月、9/10 月、11/12 月）。由 FreeBSD Foundation 出版，地址 5757 Central Ave., Suite 201, Boulder, CO 80301。电话：720/207-5142，传真：720/222-2350，邮箱：info@freebsdfoundation.org。

版权所有 © 2017 FreeBSD Foundation。保留所有权利。未经出版商书面许可，不得全部或部分复制本杂志。
