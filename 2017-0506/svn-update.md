# SVN 动态

作者：Steven Kreuzer

添加 `ipfw_pmod` 内核模块（<https://svnweb.freebsd.org/changeset/base/316435>）

和生活中的其他事物一样，安全是一种权衡，通常以终端用户失去功能或忍受性能或吞吐量下降为代价。这通常意味着，除非有人或组织决定认真对待安全，否则首先被关闭的就是防火墙或访问控制安全策略。更糟糕的是，有时安全机制会以不明显的方式导致应用程序崩溃，采取“关掉一切直到能运行为止”的大锤方法实在太常见了。在另一个极端，你可能获得虚假的安全感，实际上机器是脆弱的，因为你在语法难以理解的复杂安全子系统中犯了一个微妙的拼写错误。

这个新模块设计用于修改任何协议的数据包，目前仅实现 TCP MSS 修改。它为 `tcp-setmss` 操作添加了外部动作处理程序。带有 `tcp-setmss` 操作的规则会额外检查协议和 TCP 标志。如果存在 SYN 标志，它会解析 TCP 选项并在 MSS 选项的值大于规则中配置的值时修改该选项。

将 `pom` Capsicum 化（<https://svnweb.freebsd.org/changeset/base/317165>）

当我看到这个提交时忍不住笑了，因为我不知道开箱即用的 FreeBSD 居然能报告月相。虽然这看起来是一个攻击面很小的微不足道的应用程序，但它已被转换为在 Capsicum 沙箱中运行，所以你晚上可以安枕无忧。

安全很难，要做得对更难。FreeBSD 项目的一大优势在于系统安全一直是重点。开发者投入大量时间和精力提供安全可靠的系统，使得几乎或完全没有 FreeBSD 使用经验的人也能构建新机器并直接连接到 Internet，且能合理确信不会发生糟糕的事情。过去几年，FreeBSD 一直被用作一些创新概念的试验场，这些计算机安全新概念既轻量又大部分透明，为系统管理员和终端用户都带来了更好的体验。由于本期主题是安全，我想利用这期 svn 更新来强调一些旨在保护你和你的数据安全的近期改进，无论是加密方面的改进、扩展 FreeBSD 以支持正在内嵌于 CPU 本身的硬件安全特性，还是明确界定应用程序能用和不能用 Capsicum 做什么的持续工作。

在更多位置设置 arm64 的 Execute-never 位（<https://svnweb.freebsd.org/changeset/base/316761>）

arm64 上的每个内存区域都可以标记为不包含可执行代码。如果描述符的 Execute-never（XN）位设为 1，任何尝试在该区域执行指令的企图都会导致权限错误。映射设备内存时设置 Execute-never 位，因为硬件可能执行推测性指令预取。在用户态内存上设置特权 Execute-never 位，以在内核被诱骗执行它时阻止内核。

在 ARMv8.1 上启用进程状态位以禁止从内核访问用户态（<https://svnweb.freebsd.org/changeset/base/316756>）

ARMv8.1 引入了新的特权访问禁止（Privileged Access-never，PAN）状态位。此位提供控制，阻止对用户数据的特权访问，除非显式启用，这为可能的软件攻击提供了额外安全防护。

3BSD 许可的 ChaCha20 流密码实现，供即将到来的 `arc4random` 替代方案使用（<https://svnweb.freebsd.org/changeset/base/316982>）

在 dummynet 中使用 `strlcpy` 以安抚静态检查器（<https://svnweb.freebsd.org/changeset/base/316777>）
防止 **restore(8)** 中的堆溢出（<https://svnweb.freebsd.org/changeset/base/316799>）
防止 **mixer(8)** 中可能的 `sscanf()` 溢出（<https://svnweb.freebsd.org/changeset/base/317596>）
修复 **ctm(1)** 中一些微不足道的 argv 缓冲区越界（<https://svnweb.freebsd.org/changeset/base/316795>）
避免 **loader(8)** 中通过环境变量的可能溢出（<https://svnweb.freebsd.org/changeset/base/316771>）
修复传入零长度缓冲区时的越界写入（<https://svnweb.freebsd.org/changeset/base/316768>）

ChaCha20 密码是 Daniel J. Bernstein 于 2008 年发布的高速密码。由于使用对 CPU 友好的加法-旋转-异或（Addition-rotation-XOR）运算，在纯软件实现中它比 AES 快得多。

用 ChaCha20 替换 RC4 算法生成内核安全随机数（<https://svnweb.freebsd.org/changeset/base/317015>）

编写软件很难，但我们很幸运，FreeBSD 的贡献来自一群非常有才华的人。然而，这些开发者是人，人有时会犯错。但我们有一个非常活跃的社区，不断审计代码库以寻找潜在的安全漏洞，无论大小。虽然不像我提到的其他一些改动那样令人兴奋，但我觉得看看几段看似无害但实际上可能被用作入侵你系统途径的代码片段会很有趣。

---

**STEVEN KREUZER** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车感兴趣。他与妻子、两个女儿和一只狗住在纽约皇后区。
