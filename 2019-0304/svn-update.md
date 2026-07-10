# SVN 动态

作者：Steven Kreuzer

社区里围绕地址空间布局随机化（ASLR）已经争论多年，如今它终于合并进 FreeBSD 13。取决于你问的是谁，这种最先进的缓解技术要么能让你的机器比裹上水泥还安全，要么就是一种早已过时、只带来虚假安全感的技术。至于我对此话题的看法——和谈论政治一样，我不喜欢在餐桌上讨论安全，但我可以提一句：我最近在 **/etc/make.conf** 中加入了 `WITH_PIE=yes`。

## 实现地址空间布局随机化（ASLR）

<https://svnweb.freebsd.org/changeset/base/343964>

这一改动可以为所有非固定映射启用随机化。也就是说，映射的基址会以有保证的熵（位数）来选取。如果映射要求按超级页对齐，随机化会保留超级页属性。

虽然随着漏洞利用作者找到简单的 ASLR 绕过技术，ASLR 的价值在逐渐下降，但它至少在理论上消除了对某些漏洞的简单利用。这一实现规模相对较小，并在正确的架构层级完成。此外，在关闭（目前是默认）的情况下，预计不会在现有用例中引入回归，也不会带来显著的维护负担。

随机化按尽力而为原则进行——也就是说，如果碎片化阻碍了熵注入，分配器会回退到首次适配策略。实现一个失败保证请求熵量就导致映射请求失败的强模式并不难，但我不认为那具有可用性。

## 添加 WITH_PIE 旋钮以构建位置无关可执行文件

<https://svnweb.freebsd.org/changeset/base/344179>

将二进制文件构建为 PIE，可在启用 ASLR 时让可执行文件本身（而不仅仅是其共享库）加载到随机地址。这一改动后，PIE 对象使用 `.pieo` 扩展名，INTERNALLIB 库使用 `libXXX_pie.a`。

某些 kerberos5 工具、Clang 和 Subversion 禁用了 `MK_PIE`，因为它们在 Makefile 中显式引用了 `.a` 库。这些问题可以日后逐个处理。`rtld-elf` 也禁用了 `MK_PIE`，因为它已经通过定制的 Makefile 规则是位置无关的。目前只有动态链接的二进制文件会构建为 PIE。

## 迈向多核 bhyve AMD 支持

<https://svnweb.freebsd.org/changeset/base/343075>

vmm 的 CPUID 仿真此前向客户机呈现 Intel 拓扑信息，却禁用了 AMD 拓扑信息，某些情况下还会传入垃圾数据。CPUID 叶 0x8000_001 [de] 被透传给客户机，但客户机 CPU 会在主机线程之间迁移，因此呈现的信息不一致。这可以通过 `cpucontrol -i 0xfoo /dev/cpuctl0` 轻易观察到。

通过启用 AMD 拓扑特性标志，并至少呈现 FreeBSD 自身用于在较新 AMD64 硬件（Family 15h+）上探测拓扑的 CPUID 字段，略微改善了这一情况。较老的硬件大概不那么重要。我无法实证确认这已足够，但应该不会引起回归。

## 在 AMD 主机上屏蔽 Spectre 特性位

<https://svnweb.freebsd.org/changeset/base/343166>

为与 Intel 主机保持一致——后者已经屏蔽了指示 `SPEC_CTRL` MSR 存在的 CPUID 特性位——在 AMD 上做同样的处理。

未来我们或许希望为客户机提供更好的支持方案，但目前先限制错误指示我们尚不支持的 MSR 所造成的损害。

最终，我们可能希望为管理员提供通用的 CPUID 覆盖系统，或为带故障转移的异构环境提供最低支持特性集。那是一个比这个 bug 修复大得多的工作。

## 添加 AMD Spectre/Meltdown CPUID 信息的定义

<https://svnweb.freebsd.org/changeset/base/343120>

无功能性变化，只是在 CPU 识别时打印已识别的位。这些位记录在 111006-B“Indirect Branch Control Extension”[1] 和 124441“Speculative Store Bypass Disable”[2] 中。

值得注意的缺失项（留作未来工作）：

- 与 `hw.spec_store_bypass_disable` 和 `hw_ssb_active` 标志的集成，目前这些是 Intel 专用的
- 与 `hw_ibrs_active` 全局标志的集成，目前这些是 Intel 专用的
- 在 `hw_ssb_recalculate()` 中集成 SSB_NO
- Bhyve 集成（PR 235010）

## 为 RISC-V 实现每 CPU pmap 激活跟踪

<https://svnweb.freebsd.org/changeset/base/344108>

这通过确保只中断正在使用给定 pmap 的 CPU，减少了 TLB 失效的开销。跟踪在 `pmap_activate()` 中进行，该函数在上下文切换时被调用：从 `cpu_throw()` 调用（如果线程正在退出或 AP 正在启动），或对于常规上下文切换从 `cpu_switch()` 调用。目前，`pmap_sync_icache()` 仍然必须中断所有 CPU。

## 为 RISC-V 实现透明的 2MB 超级页提升

<https://svnweb.freebsd.org/changeset/base/344106>

这些改动很大程度上参考了 amd64。arm64 对超级页创建有更严格的要求，以避免 TLB 冲突中止的可能性，而这些要求不适用于 RISC-V——和 amd64 一样，RISC-V 允许同时缓存给定页面的 4KB 和 2MB 转换。RISC-V 的 PTE 格式仅包含两个软件位，而这两个位已被占用，因此我们没有 amd64 的 `PG_PROMOTED` 等价物。相应地，`pmap_remove_l2()` 总是失效整个 2MB 地址范围。

`pmap_ts_referenced()` 经过修改以清除 `PTE_A`，因为我们现在同时支持硬件和软件管理的引用位和脏位。同时修复了 `pmap_fault_fixup()`，使其不在内核映射上设置 `PTE_A` 或 `PTE_D`。

## 在 i386 上提供 do_cpuid() 和 cpuid_count() 的用户态版本

<https://svnweb.freebsd.org/changeset/base/344118>

生成 PIC 代码时，一些较老的编译器无法处理会破坏 `%ebx` 的内联汇编（因为 `%ebx` 用作 GOT 偏移寄存器）。用户态版本通过在执行 CPUID 指令前将 `%ebx` 保存到栈上来避免破坏它。

---

**STEVEN KREUZER** 是一名 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车感兴趣。他与妻子、女儿和狗居住在纽约皇后区。
