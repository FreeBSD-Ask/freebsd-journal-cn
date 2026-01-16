# 会议报告：2025 BSDCan

- 原文：[Conference Report: BSDCan 2025](https://freebsdfoundation.org/our-work/journal/browser-based-edition/embedded-2/conference-report-bsdcan-2025/)
- 作者：Christos Margiolis

<img width="1920" height="577" alt="image" src="https://github.com/user-attachments/assets/5fa71026-df0a-4424-89ae-0fa19cf5cbe8" />

今年，我在 [BSDCan 2025](https://www.bsdcan.org/2025/) 上做了一场题为 [《Vox FreeBSD: How sound(4) works》](https://www.bsdcan.org/2025/timetable/timetable-Vox-FreeBSD-How.html) 的演讲。

我于 6 月 9 日抵达蒙特利尔，比会议早两天，不过当天没做什么，主要是休息。

![](https://freebsdfoundation.org/wp-content/uploads/2025/10/uoview.png)

第二天 6 月 10 日，我乘巴士从蒙特利尔前往渥太华（会议举办地）。一下车就偶遇了 Olivier Certner（olce@），我们一起在渥太华大学附近餐厅吃了午饭——会议和演讲者住宿都在这里——之后办理入住。晚上，我和 Mateusz Piotrowski（0mp@）、Bojan Novković（bnovkov@）、Kyle Evans（kevans@）在 Father & Sons 餐厅喝酒聚餐。在这个常见的聚会点，我们还遇到了会议的其他人。

会议的前两天（6 月 11–12 日）是 [FreeBSD DevSummit](https://wiki.freebsd.org/DevSummit/202506)。

第一天的亮点是核心团队发起的关于 FreeBSD 项目中 **AI 使用** 的公开讨论，休息时间也在继续。核心团队主要关注 AI 生成代码的许可问题。而我坚持认为应该 **坚决反对 AI 的任何使用**，主要基于伦理和质量方面的担忧。许可问题在我看来只是次要的。比如：我们是否要主动参与可能让世界变得更糟的事？如果放宽 AI 政策，会吸引什么样的人加入项目？这会怎样影响项目的长期质量？我们真的 **需要** AI 及其带来的复杂性吗？如果没有许可问题，那用 AI 就没问题吗？还有很多类似的疑问。好在不少人（有些人犹豫着）支持了我的观点。

我还与 Mark Johnston（markj@）、Joseph Mingrone（jrm@）、Bojan 以及 Charlie Li（vishwin@）进行了技术和非技术讨论。意外的是，Charlie 也对 FreeBSD 音频/音乐制作感兴趣，还用它做 DJ 表演。

DevSummit 第一天结束后，我们在大学宿舍里吃了披萨，但我比较早回房间继续准备幻灯片。

第二天 6 月 12 日的日程以 AlphaOmega 的安全审计演讲开始，接着是关于 [FreeBSD 15.0 技术规划](https://hackmd.io/@jhb/ByWrxQmr2) 的讨论，包括 PkgBase。午饭后，Brooks Davis（brooks@）带来了一场极其精彩的演讲——大概是整个 DevSummit 我最喜欢的技术讲座——主题是 [CheriBSD](https://www.cheribsd.org/) 分支的上游合并。他不仅讲了上游过程，还详细介绍了 CHERI、CheriBSD 及能力机制，并展示了其内建的内存安全特性能如何捕捉 FreeBSD 代码中的各种 bug。随后，[FreeBSD 基金会](https://freebsdfoundation.org/) 汇报了近期工作和资金使用情况，包括我参与的 [Laptop Support and Usability Project](https://github.com/FreeBSDFoundation/proj-laptop)。在会议尾声，我和 Alexander Ziaee（ziaee@）花时间调试了他笔记本的声音问题。晚上，我与 Benedict Reuschling（bcr@）等人去了一家不错的海鲜餐厅。

![](https://freebsdfoundation.org/wp-content/uploads/2025/10/street.png)

BSDCan 正式会议于 6 月 13–14 日举行。开幕主题演讲由著名计算机科学家 Margot Seltzer 主讲，题为 [《Hardware Support for Memory-Hungry Applications》](https://www.bsdcan.org/2025/timetable/timetable-Keynote-Hardware-Support.html)。

这两天我听了许多讲座，其中印象深刻的有：

- [ShengYi Hung：《ABI stability in FreeBSD》](https://www.bsdcan.org/2025/timetable/timetable-ABI-stability-in.html)，展示了一个检测 CTF 数据差异的实验性工具，用于发现 ABI 变化。之后我们（包括我、Mark Johnston、John Baldwin、Warner Losh 和讲者本人）展开了关于工具局限性及如何在实际应用中的讨论。
- [Marshall Kirk McKusick：《A History of the BSD Daemon》](https://www.bsdcan.org/2025/timetable/timetable-A-History-of.html)，以及即将出版的《FreeBSD 操作系统设计与实现》（*The Design and Implementation of the FreeBSD Operating System*）第三版更新。我一向喜欢 Kirk 的演讲风格。
- [Bojan Novković：《Hardware-accelerated program tracing on FreeBSD》](https://www.bsdcan.org/2025/timetable/timetable-Hardware-accelerated-program-tracing.html)，介绍了他在 hwt(8) 框架上的最新工作。
- [Zhuo Ying Jiang Li：《Improvements to FreeBSD KASAN》](https://www.bsdcan.org/2025/timetable/timetable-Improvements-to-FreeBSD.html)，讲解了 FreeBSD 内核地址消毒器 (KASAN) 的改进工作，这是她参与 CheriBSD 的一部分。

我没能参加但本想听的讲座有：

- [John Baldwin：《ELF Nightmares: GOTs, PLTs, and Relocations Oh My》](https://www.bsdcan.org/2025/timetable/timetable-ELF-Nightmares-GOTs,.html)
- [Andrew Hewus Fresh：《The state of 3D-printing from OpenBSD》](https://www.bsdcan.org/2025/timetable/timetable-The-state-of.html)
- [Hans-Jörg Höxer：《Confidential Computing with OpenBSD — The Next Step》](https://www.bsdcan.org/2025/timetable/timetable-Confidential-Computing-with.html)
- [Andreas Kirchner, Benedict Reuschling：《Enhancing Unix Education through Chaos Engineering and Gamification using FreeBSD》](https://www.bsdcan.org/2025/timetable/timetable-Enhancing-Unix-Education.html)

我在 6 月 14 日，也就是 BSDCan 的最后一天做了演讲。引发了大量问题和交流，甚至在讲座结束后仍在继续。显然，比我预想的更多人希望能在 FreeBSD 上做音乐和音频制作，或用于大型音频系统，这场演讲似乎启发了他们，这是非常好的事情。

会议闭幕式后，我们去附近的市场广场参加社交活动。我大部分时间和 Mark Johnston、Andreas Kirchner、Mateusz Piotrowski 在一起，进行了很多有趣的对话。

第二天，我返回蒙特利尔，休息了几天才回家。

一如既往，会议是弥补编程孤独性的绝佳机会，让我们能见到每天通过邮件交流的幕后人。除了完成工作和交换技术想法，我更享受那些意外发生的、深入的交流，包括与此前未曾见过的人。

---

**Christos Margiolis** 是来自希腊的独立开发者和 FreeBSD src 提交者。
