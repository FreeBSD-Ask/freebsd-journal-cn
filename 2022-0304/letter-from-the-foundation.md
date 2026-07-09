# 基金会来信

- 原文：[Letter from the Foundation](https://freebsdfoundation.org/wp-content/uploads/2022/04/Foundation-Letter.pdf)
- 作者：**Ed Maste**

## FreeBSD/arm64 现为一级架构

Arm 的 64 位架构 AArch64 在 FreeBSD 13 中获得一级架构（Tier 1）地位。沿用 32 位 FreeBSD/arm 的命名，我们使用"arm64"。

一级架构意味着 FreeBSD 发行工程团队除现有的 amd64 和 i386 外，还会为该架构构建并发布正式发行版。安全团队以二进制和源代码更新支持该架构，修复漏洞和勘误。软件包团队提供完整的二进制软件包集合。

Arm 于 2011 年 10 月公开披露了 AArch64 架构细节，Andrew Turner 很快对 FreeBSD 移植产生了兴趣。FreeBSD 基金会于 2014 年开始支持 arm64 移植工作，资金来自 Arm 和 Cavium。与 Andrew 和 Semihalf 合作，Cavium 的 ThunderX 处理器被引入，成为 FreeBSD/arm64 的首个参考平台。基金会在发行工程、工具链支持等方面贡献了力量。

多年来支持持续改进，但晋升一级架构需要多种因素的汇聚。

早期硬件供应有限，尤其是服务器级机器。如今 Ampere Computing 的 eMAG 和 Altra CPU、Arm 的 Neoverse N1 核心、AWS Graviton 实例都得到良好支持。基金会购置了 eMAG 服务器用于构建官方软件包集。Ampere Computing 随后捐赠了更多服务器。这让项目能同时支持多个 FreeBSD src 和 Ports 树分支。

工具链曾是早期的限制因素。FreeBSD 仍使用较旧版本的 GNU 链接器，它不支持 arm64，需要采取变通方法使用树外链接器。在基金会的赞助下，项目在 FreeBSD 13 中对所有支持的架构迁移到使用 LLVM 的 LLD 链接器。

感谢 FreeBSD Ports 志愿者的不懈贡献，我们能迭代解决 arm64 构建失败问题；如今已有超过 30,000 个软件包可用。

2021 年 4 月，我代表 Core Team 宣布 FreeBSD/arm64 将在 FreeBSD 13 中成为一级架构。

希望你喜欢本期关于 arm64 的文章，并尝试使用 FreeBSD/arm64！

**Ed Maste**
FreeBSD 基金会技术高级总监，FreeBSD 期刊编辑委员会
