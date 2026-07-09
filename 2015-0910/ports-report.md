# Ports 报告

- 原文：[Ports Report](https://freebsdfoundation.org/our-work/journal/browser-based-edition/cloudabi/)
- 作者：**Frederic Culot**

与往年一样，夏季（7–8 月）Ports 领域相对平静。尽管按合入树中的 commit 数量计算活跃度下降了近 25%，但仍取得重大改进。

## 新任 Ports Committer 与代为保管

很高兴欢迎两位新/回归的 Ports committer。

- 首先是 Jason Unovitch 获得 Ports commit 权限，由 feld@、delphij@ 与 pgollucci@ 担任导师。Jason（现用名 junovitch@）极为活跃，已提交超过一百次 commit，现已脱离导师指导。感谢你的辛勤付出，Jason！
- 在一段时间的沉寂后回归的是 Babak Farrokhi（farrokhi@）。为协助其回归，philip@、bapt@ 与 mat@ 将负责指导。

部分 commit 权限应 committer 本人要求代为保管（xmj@、stefan@ 与 brix@）。

## 重要 Ports 更新

antoine@ 与 mat@ 进行了大量 exp-run（共 15 次），以验证主要 port 更新的安全性，包括：

- Gnome 升级至 3.16
- gettext 升级至 0.19.5.1
- doxygen 升级至 1.8.10
- libreoffice 升级至 5.0.1
- ghc Haskell 编译器升级至 7.10.2

一如既往，升级前请仔细阅读 **/usr/ports/UPDATING** 文件，因为可能需要手动操作步骤。

## 数据统计

我们提供几项数据，以便大家了解志愿者在暑假期间完成的出色工作量。7 月与 8 月期间，Ports 树共合入 4,212 次 commit，关闭 1,009 份 PR，portmgr@ 收到 1,112 封邮件（尚未计算垃圾邮件……）。而所有这些仅由平均 122 名活跃的 Ports 开发者处理！

数字无疑令人印象深刻！但请注意，Ports 树由志愿者构建与维护。因此，如果你使用 Ports 或软件包，请考虑加入并伸出援手！如果你已是 porter，请考虑分享你的知识，指导对 Ports 树内部更生疏的新人。若你已在做这些事，万分感谢！

## Ports 树 21 周年

8 月 21 日，我们的 Ports 树满 21 周岁！去年为庆祝 Ports 树 20 周年制作了一段视频（<http://youtu.be/LiFq5D-zmBs>），错过的不妨一看。

**作者简介**

Frederic Culot 在完成物理学计算机建模博士学位后，已在 IT 行业工作 10 年。业余时间他研习商业与管理，并刚取得 MBA 学位。Frederic 于 2010 年以 Ports committer 身份加入 FreeBSD，此后累计提交约 2,000 次 commit，指导过六位新 committer，现承担 portmgr-secretary 的职责。
