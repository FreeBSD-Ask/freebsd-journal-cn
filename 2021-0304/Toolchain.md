# FreeBSD 13.0 工具链

- 原文链接：[FreeBSD 13.0 Toolchain](https://freebsdfoundation.org/wp-content/uploads/2021/05/FreeBSD-13.0-Tool-Chain.pdf)
- 作者：**ED MASTE**

FreeBSD 13.0 标志着 FreeBSD 工具链演变的一个重要里程碑：完成了长达十年的迁移，采用现代的、许可宽松的编译器、链接器、调试器以及其他各种二进制工具，适用于所有 FreeBSD 支持的架构。

从一开始，FreeBSD 就依赖 GNU 工具链，包括 GCC 编译器、Binutils 链接器和二进制工具，以及 GDB 调试器。这些工具通过一群开发者的定期更新得到了保持，包括志愿者和付费开发者。这一过程持续到 2007 年，当时 GNU 项目将这些工具的许可证更改为 GNU 通用公共许可证第 3 版（GPLv3），之后它们再也没有更新。

几年后，大约二十名开发者在 2010 年 BSDCan 大会上的开发者峰会上会面，规划工具链的未来，目标是迁移到一个现代的、许可宽松的工具链。许可证问题只是其中的一部分理由。新技术、实施新的和定制功能的能力，以及推动开源工具的竞争也成为团队目标的一部分。

与此同时，多个工具链项目在 FreeBSD 项目内部和外部逐渐成形。峰会与会者注意到，LLVM 和 Clang 正在迅速成熟，并且在编译器研究社区中获得越来越多的关注。一款 BSD 许可的 ELF 工具链项目也在进行中，同时还有多个调试器项目。

团队设定了与这些工具链项目合作并迁移到新组件的目标。随着个别组件变得可用，它们被逐步添加到 FreeBSD 中，通常与现有工具并行安装，但默认禁用。在测试和验证后，它们成为默认工具，有时是逐架构启用。最终，新的工具在所有支持的架构中启用，旧的工具则被移除。除了调试器的一个小例外，现在，所有工具链组件和所有支持的架构的迁移已经完成。请继续阅读，了解详细信息。

## 编译器

编译器是工具链中最重要的组件之一，也是我们迁移的首个主要组件。Clang 编译器于 2007 年首次发布，并迅速成熟。Roman Divacky 于 2010 年 6 月将一个版本导入 FreeBSD 的 contrib 软件树，并在 FreeBSD 9.0 中为 x86 和 PowerPC 架构提供了它（作为 /usr/bin/clang）。

在 Clang 开发者的大力努力以及 Roman、Ed Schouten、Dimitry Andric 等人的集成工作下，我们在 2012 年将 Clang 启用为 i386 和 amd64 的默认编译器（`/usr/bin/cc`）。这一版本随 FreeBSD 10.0 发布。

2014 年，Clang 准备好成为小端 Arm 架构的默认编译器。它也在 PowerPC 上提供，但并非默认编译器。这是 FreeBSD 11.0 中的配置。FreeBSD 11.0 还引入了对 64 位 ARM（AArch64）的支持，并且 Clang 是唯一可用的编译器。

在 FreeBSD 12.0 发布之前，编译器更新持续进行，但架构支持保持不变。2019 年和 2020 年，Clang/LLVM 上游的架构改进显著，剩余的架构，包括 RISC-V 和 MIPS，都切换为默认使用 Clang。Sparc64 是唯一仍然依赖过时的内置 GCC 版本的架构。随着 FreeBSD/sparc64 被淘汰，树中不再需要保留 GCC，因此在 2020 年初将其移除。

工具链通常包括汇编器。从 FreeBSD 13.0 开始，我们使用 Clang 的集成汇编器（IAS）来组装 FreeBSD 基本系统的一部分，不再提供独立的 `/usr/bin/as`。

使用 Clang 作为 FreeBSD 的标准编译器，使其成为进行全系统研究的一个引人注目的系统。剑桥大学的时序增强安全逻辑断言（TESLA）和能力硬件增强 RISC 指令（CHERI）研究项目基于 Clang/LLVM，并从提供完整的、Clang 构建的操作系统内核和用户空间中受益。

## 链接器

链接器将目标文件和库合并成可执行文件或共享库。当工具链项目开始时，没有一个具有明确可行路径的替代链接器。MCLinker 项目展示了一些初步的势头，但最终没有支持 FreeBSD 构建所需的许多功能。

到了 2015 年，LLVM 的 lld 取得了良好的进展，我们在 2016 年导入了一个快照。最初，我们将其作为默认构建，但安装时使用非默认文件名。它可以通过编译器参数 `-fuse-ld=lld` 使用。

到了 FreeBSD 12.1，lld 已为除 sparc64 和 riscv64 外的所有架构构建，并成为 32 位和 64 位 ARM 以及 x86 的默认链接器。随着 lld 上游对 riscv64 支持的开发和 sparc64 的淘汰，lld 成为 FreeBSD 13.0 中所有架构的唯一链接器。

## 二进制工具

工具链包括一些小工具，用于检查或处理目标文件、库和可执行文件——在 FreeBSD 中是 ELF 对象。这些工具包括像 `size`、`strings` 和 `readelf` 这样的文件检查工具，`objcopy` 用于转换或处理对象文件，`ar` 用于创建和提取库档案。此类别还包括其他工具用于解析对象的库——例如，`libelf` 和 `libdwarf`。

在较早版本的 FreeBSD 中，这些工具大多来自 GNU binutils，少数例外——例如，档案管理工具是基于 `libarchive` 的定制工具。FreeBSD 从 2006 年开始还实现了 `libelf` 和 `libdwarf` 的版本。

ELF 工具链项目始于 2008 年作为一个独立的努力，由 Joseph Koshy 主导。该项目导入了一些现有的 FreeBSD 工具和库，更多的贡献者也加入了项目。Kai Wang 实现了许多缺失的工具，包括 `elfcopy/objcopy` 和 `readelf`。

到了 2015 年，ELF 工具链版本的 `addr2line`、`elfcopy/objcopy`、`nm`、`size` 和 `strings` 功能足够完善，以至于我们默认切换使用它们，并在 FreeBSD 11.0 中发布。

这一工作没有涉及汇编器或链接器；虽然 ELF 工具链中开始着手这两个工具的工作，但目前它们还不可用。我们通过依赖 Clang 的集成汇编器来解决汇编器问题，并将大多数架构切换到 LLVM 的 lld 作为链接器。Sparc64 是唯一依赖内置 GNU ld 的架构。随着 sparc64 支持的退役，我们能够在 FreeBSD 13.0 发布前将 binutils 从 FreeBSD 树中移除。

从 binutils 的迁移还使得 FreeBSD 在基本系统中移除了 `objdump`。`objdump` 提供的大部分信息（以略微不同的格式）也可以通过 `readelf` 获得，并且 LLVM 提供了一个 `llvm-objdump`，它在很大程度上是兼容的。该工具尚未在基本系统中默认安装，但很可能会在未来的版本中启用。当然，也可以通过安装 binutils 的 Port 或软件包来获取 GNU `objdump`。

## 源代码级调试器

剩下的工具链组件是调试器，LLVM 在这里也提供了前进的路径。LLDB 是 LLVM 的调试器，已在之前的 FreeBSD Journal 文章中详细描述。它基于 LLVM 组件进行反汇编，使用 Clang 作为表达式解析器，并且可以通过 Python 和 Lua 脚本化。

我们在 2013 年将 LLDB 作为实验性功能添加到构建中，在 FreeBSD 10.0 发布之前，并在 2015 年为 amd64 和 arm64 启用默认支持，随 FreeBSD 11.0 发布。在 2017 年，Karnajit Wangkhem 提交了一个补丁，添加了对 i386 JIT 表达式引擎的支持，我们在 FreeBSD 12.0 中将其作为默认功能启用。

LLDB 包括对 32 位 ARM、PowerPC 和 MIPS 的 FreeBSD 目标支持，尽管其测试尚不充分。经过足够的测试和集成后，可能会默认启用这些构建。LLDB 现在是一个功能完整的 FreeBSD 用户空间调试器，但目前缺乏内核调试支持。

在 FreeBSD 基金会的赞助下，Moritz Systems 最近修复了许多未解决的 bug，改善了 arm64 目标支持，添加了初步的跟随 fork 模式，并正在改进用户空间核心文件调试。关于实现实时内核调试和死后内核核心转储支持的项目正在讨论中。我们还需要实现 RISC-V 目标支持。

## CTF 工具

FreeBSD 的 DTrace 支持使用了紧凑 C 类型格式（CTF），该格式提供了最小的调试信息，使 C 语言类型能够在 DTrace 脚本中使用。目前，使用了三种工具，这些工具根据通用开发与分发许可协议（CDDL）发布。它们实现了 CTF 版本 2，自从 2008 年从 OpenSolaris 导入 DTrace 以来几乎没有改变。

也有许多替代的 CTF 工具实现可用。现代的 binutils 包括了 libctf，这是一个解析和编辑 CTF 数据的库。CTF 格式已经扩展到版本 3，并且最近扩展到了版本 4。此外，OpenBSD 的 Martin Pieuchot 实现了一套最小的、具有宽松许可的 CTF 工具。未来的 FreeBSD 工具链工作将需要确定如何使用这些 CTF 工具。

## 诊断工具

Valgrind 是一套用于内存访问、内存泄漏调试以及检查程序行为其他方面的工具。FreeBSD 对 Valgrind 的支持已经在主 Valgrind 树之外维护了近二十年，已修补的 Valgrind 可以从 Ports 和软件包集合中获得。多年来，许多不同的开发者维护并更新了 FreeBSD 的 Valgrind  Port；最近，Paul Floyd 更新了它到最新发布的版本 3.17.0，并正在努力将 FreeBSD 的更改提交到上游。

Clang 包括对各种 sanitizers 的内建支持，这些工具提供调试和诊断信息。AddressSanitizer 检测内存错误，如越界访问或使用后释放，可以通过命令行标志 `-fsanitize=address` 使用（使用基本系统中的 Clang）。一个相关的工具，MemorySanitizer，检测未初始化变量的读取。ThreadSanitizer 检测数据竞争，UndefinedBehaviourSanitizer 捕获运行时的未定义 C 行为（如有符号整数溢出）。其他 sanitizers 也存在，但需要更多的工作才能在 FreeBSD 上启用。

Clang 还包括一款静态分析器，通过 scan-build 前端调用。它可以通过 LLVM 包安装的 Clang 使用，而不是基本系统中的 Clang。静态分析器的作用与编译器警告相似，但层次更高。

对于代码覆盖率分析，LLVM 提供了 `llvm-cov`。它也作为 `gcov` 安装，并在通过别名调用时以 GNU、gcov 兼容模式运行。当程序以 `--coverage` 编译时，它在退出时会生成一个 .gcov 文件，`gcov` 使用该文件显示源代码每行（或基本块）的执行次数。性能分析可以通过定制的 BSD 许可证代码实现：`hwpmc` 内核支持和 `pmcstat` 用户空间工具。

## 后续工作

随着向 Clang/LLVM 的过渡完成，工具链团队正在研究如何在此基础上进行扩展。其中之一是链接时优化（LTO），即在链接阶段执行的模块间优化。实际上，LTO 使用包含 LLVM 中间表示（IR）的目标文件和库，而不是目标二进制 ELF 对象。这允许在链接时将这些文件组合起来，并让 LLVM 优化传递作用于整个二进制文件，而不仅仅是单独的编译单元。LTO 的一个注意事项是，nm 和 ar 工具需要能够解析 LLVM IR 符号表，而基本系统使用的 ELF Tool Chain 版本不具备此功能。我们需要扩展 ELF Tool Chain 中的 nm 和 ar 工具，或切换到 LLVM 版本的这些工具。

Clang 提供的另一个工具链特性是控制流完整性（CFI）。CFI 广泛指的是编译器使用的技术，以避免恶意软件在获得执行权限后，试图破坏程序预定的操作（控制流）。Clang 的 CFI 支持需要 LTO，并包含多个单独的检查，用于检测各种不良类型转换和无效间接调用的情况。

CHERI 项目为 FreeBSD 工具链的未来工作提供了另一个机会。CheriBSD 是 FreeBSD 的衍生版本，实施了 CHERI 内存保护和软件隔离，并在外部代码库中维护。CHERI 还包括一款基于 LLVM 的工具链，支持多个 CHERI 增强的指令集架构（ISA）。

---

**ED MASTE** 负责 FreeBSD 基金会的项目开发。他也是选举产生的 FreeBSD 核心团队成员。除了 FreeBSD，他还为许多其他开源项目做出了贡献，包括 LLVM、ELF Tool Chain、QEMU 和 Open vSwitch。他与妻子 Anna 以及孩子 Pieter 和 Daniel 住在加拿大滑铁卢。

