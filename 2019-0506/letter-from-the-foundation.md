# 基金会来信

欢迎阅读本期 FreeBSD Journal。本期我们深入探讨如何在我们最爱的开源操作系统上使用全球最流行的教育计算平台之一——Raspberry Pi。

Tom Jones 以《Introduction to Hardware Hacking on FreeBSD》开篇，随后是 Benedict Reuschling 的《Running Icinga2》，介绍如何使用 FreeBSD 和 RPi 做系统监控。最后，FreeBSD Foundation 的 Roller Angel 暂时放下 RPi，讲解如何使用 FreeBSD 搭建媒体服务器。

FreeBSD 是一个不断演进的开源项目，每天都有变更提交到代码库，正如 Steven Kreuzer 在他告别专栏《svn Update》中所述。借此机会感谢 Steven 多年来撰写了许多这样精彩的专栏。项目在不断变化，也持续欢迎 Doug Moore 和 Sergio Carlavilla 等新开发者加入，两人都亮相于 Dru Lavigne 的《New Faces of FreeBSD》专栏。

FreeBSD Journal 的编辑委员会也在经历一些变化。Brooks Davis 和 Steven Kreuzer 将从编辑委员会轮换离任，感谢他们在过去三年里协助约稿作者和打磨文章。Kristof Provost 加入编辑委员会，已开始协助策划明年的期次。最后，George Neville-Neil 将编辑委员会主席的接力棒交给 John Baldwin。George 自 Journal 创刊起就担任编辑委员会主席。编辑委员会和 Foundation 对此表示感谢，也很高兴他仍将以委员会成员身份参与。

全新组建的 FreeBSD Journal 编辑委员会将继续为我们日益壮大的社区提供关于 FreeBSD 的优质文章和专栏。即便你不是正式的编辑委员会成员，我们也欢迎你就期次主题、文章和作者人选提出建议。

George Neville-Neil

John Baldwin

---

## 编辑委员会

| 姓名 | 职务简介 |
| ---- | -------- |
| John Baldwin | FreeBSD 开发者，FreeBSD Core Team 成员，FreeBSD Journal 编辑委员会主席。 |
| Bryan Drewery | EMC Isilon 高级软件工程师，FreeBSD Portmgr Team 成员，FreeBSD Committer。 |
| Justin Gibbs | FreeBSD Foundation 创始人，FreeBSD Foundation 主席，Facebook 软件工程师。 |
| Daichi Goto | BSD Consulting Inc.（东京）董事。 |
| Joseph Kong | Amazon Web Services 软件开发工程师，FreeBSD Device Drivers 和 Designing BSD Rootkits 作者。 |
| Dru Lavigne | iXsystems 存储工程总监，BSD Hacks 和 The Best of FreeBSD Basics 作者。 |
| Michael W Lucas | Absolute FreeBSD 作者。 |
| Ed Maste | FreeBSD Foundation 项目开发总监。 |
| Kirk McKusick | FreeBSD Foundation 董事会财务主管，The Design and Implementation 书系主要作者。 |
| George V. Neville-Neil | FreeBSD Foundation 董事会理事，The Design and Implementation of the FreeBSD Operating System 合著者。 |
| Philip Paeps | FreeBSD Foundation 董事会秘书，FreeBSD Committer，独立顾问。 |
| Kristof Provost | EuroBSDCon Foundation 财务主管，FreeBSD Committer，独立顾问。 |
| Hiroki Sato | FreeBSD Foundation 董事会理事，Asia BSDCon 主席，FreeBSD Core Team 成员，东京工业大学助理教授。 |
| Benedict Reuschling | FreeBSD Foundation 董事会副主席，FreeBSD Documentation Committer，FreeBSD Core Team 成员。 |
| Robert N. M. Watson | FreeBSD Foundation 董事会理事，TrustedBSD 项目创始人，剑桥大学高级讲师。 |

## 出版信息

FreeBSD Journal（ISBN: 978-0-615-88479-0）每年出版 6 期（1/2 月、3/4 月、5/6 月、7/8 月、9/10 月、11/12 月）。由 FreeBSD Foundation 出版，地址 2222 14th Street, Boulder, CO 80302。电话：720/207-5142，传真：720/222-2350，邮箱：<info@freebsdfoundation.org>。

版权所有 © 2019 FreeBSD Foundation。保留所有权利。未经出版商书面许可，不得全部或部分复制本杂志。
