# 基金会来信

- 原文链接：[Letter from the Foundation](https://freebsdfoundation.org/wp-content/uploads/2022/03/foundation_letter.pdf)
- 作者：Ed Maste、编辑委员会

## 编委会

- **John Baldwin**——FreeBSD 开发者，FreeBSD 期刊编辑委员会主席
- **Tom Jones**——FreeBSD 开发者，阿伯丁大学互联网工程师与研究员
- **Ed Maste**——FreeBSD 基金会技术高级总监，FreeBSD Core Team 成员
- **Benedict Reuschling**——FreeBSD 基金会董事会副主席，FreeBSD 文档提交者
- **Mariusz Zaborski**——FreeBSD 开发者

## 顾问委员会

- **Anne Dickison**——FreeBSD 基金会市场总监
- **Justin Gibbs**——FreeBSD 基金会创始人，FreeBSD 基金会主席，Facebook 软件工程师
- **Daichi Goto**——BSD Consulting Inc.（东京）董事
- **Allan Jude**——Klara Inc. CTO，该公司是全球 FreeBSD 专业服务与支持公司
- **Dru Lavigne**——《BSD Hacks》与《The Best of FreeBSD Basics》作者
- **Michael W Lucas**——《Absolute FreeBSD》、《FreeBSD Mastery》系列、《git commit murder》等 40 余本著作的作者
- **Kirk McKusick**——FreeBSD 基金会董事会财务主管，《The Design and Implementation》系列丛书主要作者
- **George Neville-Neil**——前 FreeBSD 基金会董事会主席，FreeBSD Core Team 成员，《The Design and Implementation of the FreeBSD Operating System》合著者
- **Hiroki Sato**——FreeBSD 基金会董事会董事，AsiaBSDCon 主席，FreeBSD Core Team 成员，东京工业大学助理教授
- **Robert N. M. Watson**——FreeBSD 基金会董事会董事，TrustedBSD 项目创始人，剑桥大学高级讲师

## FreeBSD/arm64 现为一级架构

Arm 的 64 位架构 AArch64 在 FreeBSD 13 中已获得一级架构（Tier 1）地位。沿用 32 位 FreeBSD/arm 的命名，我们使用"arm64"。

一级架构意味着 FreeBSD 发行工程团队除现有的 amd64 和 i386 外，还会为该架构构建并发布正式发行版。安全团队以二进制和源代码更新支持该架构，修复漏洞和勘误。软件包团队提供完整的二进制软件包集合。

Arm 于 2011 年 10 月公开披露了 AArch64 架构细节，Andrew Turner 很快对 FreeBSD 移植产生了兴趣。FreeBSD 基金会于 2014 年开始支持 arm64 移植工作，资金来自 Arm 和 Cavium。与 Andrew 及 Semihalf 合作，Cavium 的 ThunderX 处理器被引入，成为 FreeBSD/arm64 的首个参考平台。基金会在发行工程、工具链支持等方面贡献了力量。

多年来支持持续改进，但晋升一级架构需要多种因素的汇聚。

早期硬件供应有限，尤其是服务器级机器。如今 Ampere Computing 的 eMAG 和 Altra CPU、Arm 的 Neoverse N1 核心、AWS Graviton 实例都得到良好支持。基金会购置了 eMAG 服务器用于构建官方软件包集。Ampere Computing 随后捐赠了更多服务器。这让项目能同时支持多个 FreeBSD src 和 ports 树分支。

工具链曾是早期的限制因素。FreeBSD 仍使用较旧版本的 GNU 链接器，它不支持 arm64，需要尴尬的变通方法来使用树外链接器。在基金会的赞助下，项目在 FreeBSD 13 中迁移到对所有支持的架构使用 LLVM 的 LLD 链接器。

感谢 FreeBSD ports 志愿者们的不懈贡献，我们能迭代解决 arm64 构建失败问题；如今已有超过 30,000 个软件包可用。

2021 年 4 月，我代表 Core Team 宣布 FreeBSD/arm64 将在 FreeBSD 13 中成为一级架构。

希望你喜欢本期关于 arm64 的文章，并尝试使用 FreeBSD/arm64！

**Ed Maste**
FreeBSD 基金会技术高级总监，FreeBSD 期刊编辑委员会
