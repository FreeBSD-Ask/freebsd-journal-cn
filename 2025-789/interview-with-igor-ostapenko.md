# 开启 FreeBSD 开发之路 —— 专访 Igor Ostapenko

- [Starting FreeBSD Development An Interview with with Igor Ostapenko](https://freebsdfoundation.org/our-work/journal/browser-based-edition/embedded-2/interview-with-igor-ostapenko/)
- 作者：Tom Jones

**TJ:** 通往 FreeBSD 开发之路的有很多因素，有些是通过大学课程，有些是通过工作经验。你是如何了解到这个项目的？最初又是什么吸引你走向操作系统开发的？

**IO:** 在上世纪九十年代，我在上学期间有机会接触到了几种编程语言和技术。比如，我花了很多时间鼓捣 TR-DOS，小心规划行号（这在我开始用 ipfw 时勾起了回忆），为自己设想的另一个游戏写 DATA 和 GOTO，还尝试过在 MS-DOS 下用中断向量开发常驻程序。因为有这些经历，极简的命令行界面对我来说并不陌生。后来我在大学学习计算机科学时初次接触到 FreeBSD，当时唯一的问题就是要买哪些书。

很明显，掌握 FreeBSD 会很有挑战，但也会非常值得，因为我必须一路补充基础知识。为什么是 FreeBSD？高年级同学推荐它，因为我们整个宿舍的网络都是基于 FreeBSD 搭建的。这是在千禧年初，还延续着上一个十年的惯性，当时 FreeBSD 是网络领域的事实标准。近年来，我更多专注在操作系统开发上。凭借广泛的软件开发背景，我积累了一系列想法，思考如何利用操作系统内部机制来支持更高层的解决方案。

**TJ:** 你最初是如何迈出修改 FreeBSD 的第一步的？最初又是怎么决定要做什么工作的？

**IO:** 第一步是准备。我希望能完善自己已有的 FreeBSD 知识，填补空白，形成更系统的认知。一本非常著名的参考书是 *The Design and Implementation of the FreeBSD Operating System*（《FreeBSD 操作系统设计与实现》），作者是 Marshall Kirk McKusick、George V. Neville-Neil 和 Robert N.M. Watson。

FreeBSD 源码对我来说并不完全陌生，多年来我已经对其结构有了整体印象。但我想要专业的指导，避免错过关键概念、风格细节或结构要点。幸运的是，有 McKusick 主讲的 FreeBSD 内核课程，它帮我节省了很多时间，解答了我的问题，并且在最佳的方式下提供了历史背景来回答“为什么”。另外，George Neville-Neil 的《FreeBSD Networking from the Bottom Up》（FreeBSD 网络体系结构自底向上）课程则进一步完善了我在网络栈方面的理解。

我考虑过先做大项目还是小项目，并和 mckusick@ 以及 kib@ 讨论过。Konstantin Belousov 建议我从小任务入手，比如修复 bug，这事实证明是最有效的方法。我最初处理的是一些最新报告的 pf 漏洞，这又衍生出对 jail 子系统的改进、对 Kyua 的 execenv=jail 测试工具的改进，甚至还开发了一个新的模块 dummymbuf 用于特定的网络测试。结果是，我继续和 Kristof Provost、Mark Johnston 以及其他 FreeBSD 开发者一起推动项目改进。

**TJ:** 修 bug 是新手进入一个项目的好办法。你对 2025 年的新 FreeBSD 贡献者有没有推荐的入门方向？

**IO:** 项目官网已经提供了正式的指导和具体方向，例如 [IdeasPage](https://wiki.freebsd.org/IdeasPage)。我更愿意提出一种替代思路：最好的路径往往是和个人兴趣一致的。

比如，如果有人对学习或使用 FreeBSD 的网络工具或内核模块（如 netstat、route、pf、ipfw、netgraph 等）感兴趣，那么阅读相关文档和手册页的同时，可能会发现可以通过补充示例、重写复杂概念或补充缺失部分来改进它们。如果 FreeBSD 里没有相应工具或模块，那么将有用的程序加入 Ports 或保持其更新，也是非常重要的参与方式。这类项目通常既有趣又有教育意义，因为可能需要更深入地理解 FreeBSD 内核接口。

如果目标是深入理解内核代码，也可以采用类似的方法——选择自己使用或计划使用的功能，能获得更多收益。比如防火墙：理解其规则在幕后是如何运作的，可以给高级用户带来优势；或者对路由机制进行研究，以解决特定问题。可能还需要在内核中实现缺失的功能或 RFC。偶尔也会有把其他平台的方案移植到 FreeBSD 的需求，例如正在进行中的 Netlink 实现，或是 VPP 框架的移植。这些都为进一步改进留下了空间。最终，和现有代码的工作总会揭示出优化机会——减少每单位数据传输或处理的资源消耗，这对所有依赖 FreeBSD 的企业都有利。

如果贡献不止一个小补丁，我建议两个起步步骤：做好功课并沟通。与 FreeBSD 开发者联系的方式很多（见 [community 页面](https://www.freebsd.org/community/)），最基本的是邮件列表，例如 [hackers@FreeBSD.org](mailto:hackers@FreeBSD.org)。先讨论潜在项目，可以达成方向上的共识，也可以发现是否有人已经在做。对于开源项目来说，准备工作越充分，沟通效果越好。

**TJ:** 开发过程往往令人望而生畏，还有很多死胡同。你能分享一些捷径，帮助新开发者更轻松地调试和开发吗？

**IO:** 我认为《FreeBSD 期刊》是一项非常棒的专业经验分享渠道。我建议浏览过往期刊的目录，寻找能填补知识空白或提供新视角的文章。比如，Mark Johnston 的《Kernel Development Recipes》（内核开发秘籍）和《DeBUGGING the FreeBSD Kernel》（调试 FreeBSD 内核），Navdeep Parhar 的《FreeBSD Kernel Development Workflow》（FreeBSD 内核开发工作流），以及你写的《More Modern Kernel Debugging Tools》（更现代的内核调试工具）。这些文章可以快速概览构建系统的功能和一些技巧（比如避免耗时的完整重建），以及如何利用虚拟化或第三方软件提升效率。

要养成项目推荐的良好实践，我建议阅读 Ed Maste 的《Writing Good FreeBSD Commit Messages》（撰写优质的 FreeBSD 提交信息）。迟早也该熟悉 John Baldwin 的《FreeBSD Code Review with git-arc》（使用 git-arc 进行 FreeBSD 代码审查），这个工具大大提升了补丁发布、审查、更新和合并的效率。

在内核网络方面，我建议仔细研究 FreeBSD 的 Jail 和 VNET 功能。如果工作不涉及特定硬件支持，那么基于 VNET 的 Jail 可以大大简化开发时测试新网络功能的过程。它们可以当作轻量级虚拟机网络来测试特定数据包路径或网络栈行为。这比其他方式更简单，即使同一个 mbuf 数据包缓冲在场景中扮演了所有角色，因为数据都不会离开主机。同时，这样的实验场景也可以成为开发新自动化测试的良好起点。Kristof Provost 的《The Automated Testing Framework》（自动化测试框架）文章和相关防火墙测试代码，以及他在 YouTube 上的演讲，都能提供灵感。

此外，投入时间来熟悉源码环境也很重要。内核是一种特殊的软件，它支持多种架构、编译器和特殊场景，预处理器的 ifdefs 并不足以解决所有问题，有些条件甚至是在代码之外解决的。另外，源码中包含一些自动生成的代码，比如系统调用或 VFS 操作的模板。因此，单靠 grep 或编辑器默认功能是不够的。我个人的配置是 Neovim + clangd + intercept-build，效果不错，但最好先了解有哪些可选方案。[这篇文档](https://docs.freebsd.org/en/articles/freebsd-src-lsp/) 介绍了如何利用 LSP 来简化代码理解和导航，这对新开发者来说至关重要。

**TJ:** 感谢你接受采访。最后你有没有给新贡献者的建议，或者想补充的内容？

**IO:** 也谢谢你。我的总体建议适用于任何有大量志愿者的国际开源项目：保持开放心态和一定的耐心。打个比方，在社区网络中有许多“主机”（贡献者），并不是随时都能通过任何协议访问。每个“主机”都可能忙于自己的工作，网络接口可能已经过载。因此，对我们的连接超时并不意味着严格的拒绝（RST），更可能是需要更多时间来处理我们的 SYN。像电子邮件这样的沟通方式能提供深度缓冲，通常“先到达的请求”可能最后才被考虑，所以礼貌的重发往往是实用的做法。还需要重新设定对延迟的预期，因为存在时区、优先级（工作优先于志愿贡献）或停机（周末、假期等）的问题。

这个网络已经运行了数十年，一些“主机”从早期就一直参与。他们的积累可以帮助我们应对新挑战，给出方向，或提醒我们避免一些尚未显现的问题。同时，新加入的“主机”也能带来新想法或新视角。每位贡献者都为拼图增添了独特的一块。因此，保持开放心态，努力理解别人信息背后的意图，对新贡献者建立更好的连接至关重要。

虽然不是必须的，但如果你能在“带宽”上做到对等（即也能接受外部连接），这将提升整个网络的容量并促进扩展。换句话说，要准备好在自己这边“打开一些端口”，接纳他人的输入。

---

**Tom Jones** 是一位 FreeBSD 提交者，专注于保持网络栈的高性能。

**Igor Ostapenko** 是 FreeBSD 和 OpenZFS 的贡献者，拥有广泛的软件开发经验，涉及导航设备测试系统、企业流程优化解决方案、逆向工程，以及 B2B/B2C 创业项目等多个领域。
