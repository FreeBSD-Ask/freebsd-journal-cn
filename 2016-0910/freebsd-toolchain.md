# FreeBSD 工具链

- 原文：[FreeBSD Toolchain](https://freebsdfoundation.org/wp-content/uploads/2016/10/FreeBSD-Toolchain.pdf)
- 作者：**Ed Maste**

与其他 BSD 发行版一样，FreeBSD 有基本系统的概念，即集成的内核和核心用户空间，由同一团队共同开发、测试和发布。用户空间包括 C 语言运行时、构建工具链、基本系统工具，以及一些第三方库和应用程序。工具链包括编译器和链接器，它们将源代码翻译为可执行对象，以及用于检查或修改这些对象的相关工具。

除基本系统外，还有一个通过 FreeBSD Ports 树和二进制软件包管理的第三方软件生态系统。Ports 树中有超过 26,000 个第三方软件包。

在其历史的大部分时间里，FreeBSD 依赖完整的 GNU 工具链套件，包括 GNU 编译器套件（GCC）编译器和 GNU binutils。它们在 1993 年到 2007 年间为 FreeBSD 项目服务良好，直到 GNU 项目转而以通用公共许可证第 3 版（GPLv3）发布其软件。与许可证第 2 版不同，GPLv3 对软件用户施加了若干新限制，FreeBSD 开发者和用户认为这些限制不可接受。因此，FreeBSD 中的 GNU 工具链未更新到新的上游版本，很快便过时了。

在那次过渡后不久，FreeBSD 项目中的若干开发者开始将 FreeBSD 工具链的组件更新为现代的、无版权限制的替代品。工具链的某些方面与架构相关，一般来说，下面描述的工具适用于 FreeBSD 的一级架构（Tier-1）。

## 编译器

LLVM 项目是一组模块化、可复用的组件集合，用于构建编译器和其他工具链工具。它最初是伊利诺伊大学的一个研究项目，后来发展成为一个生产级工具集，用于专有软件、开源软件和学术环境。Clang 是 LLVM 套件中包含的 C 和 C++ 编译器。

我们从 2009 年开始在 FreeBSD 中试验 Clang，最初以上游项目 Subversion 仓库的一个快照为起点。最初加入时，它作为一个选项提供，以方便测试和开发，但很长一段时间内并未启用为默认编译器。2014 年 1 月发布的 FreeBSD 10 中纳入了 Clang，作为 32 位和 64 位 Intel x86 架构的系统编译器（即安装为 **/usr/bin/cc**）。后来它成为 32 位 FreeBSD ARM 移植版的默认编译器。64 位 ARM 移植版从起步就以 Clang 作为系统编译器。

Clang 对 MIPS 和 PowerPC 架构有一定支持，但尚不能在这些架构上替代系统编译器；这些架构仍继续使用 GCC。

FreeBSD 11 包含 Clang 3.8.0，FreeBSD-CURRENT 中正在更新至 Clang 3.9.0。

## 链接器

链接器以编译器或汇编器生成的各个对象文件为输入，将它们链接为可执行二进制文件或库。迄今为止我们在 FreeBSD 中使用 GNU 链接器，最近一次更新是在 2011 年 2 月，版本为 2.17.50。该链接器缺少链接时优化和对某些调试特性的支持。

基本系统中的 GNU ld 还缺少对 AArch64（arm64）和 RISC-V 的支持，这两种 CPU 架构是最近加入 FreeBSD 的。目前为这些架构构建 FreeBSD 需要从 Ports 树安装链接器或作为软件包安装，此时会自动使用它。

LLVM 项目家族包含一个链接器：LLD。它是一个高性能链接器，支持 ELF、COFF 和 Mach-O。在可能的情况下，它保持与 GNU ld 等现有链接器的命令行和功能兼容性，但当严格兼容性妨碍性能或所需功能时，LLD 的作者不受其约束。

LLD 的 ELF 支持工作始于 2015 年 7 月，进展非常迅速；它目前在 LLVM 支持的若干架构上能自举，并包含构建 FreeBSD 基本系统所需的几乎所有功能。

与 GNU ld 相比，FreeBSD 中的 LLD 将提供 AArch64 及最终的 RISC-V 支持、全程序链接时优化（LTO）、对新的应用二进制接口（ABI）的支持、其他链接器优化、调试特性，以及快得多的链接时间。

如同早期对 Clang 的试验，LLD 将首先作为选项加入 FreeBSD，不会安装为系统链接器（**/usr/bin/ld**）。可通过在编译器命令行参数中添加 `-fuse-ld=lld`（例如通过变量 `CFLAGS`）来使用它。

我们随 Clang 3.9 更新一并纳入了 LLD 3.9，预计它将于 2016 年 10 月或 11 月在 FreeBSD-CURRENT 中可用。预计也会加入 FreeBSD 11.1。

## 二进制工具

工具链包含若干小型二进制工具，用于检查或修改对象文件、二进制文件和库。这些工具报告对象、二进制文件或库的信息，或转换对象的内容或格式。这些工具过去由 GNU binutils 提供。

ELF Tool Chain 项目维护基本编译工具和库的 BSD 许可实现，用于处理 ELF 对象文件和二进制文件以及 DWARF 调试信息。它最初是 FreeBSD 项目的一部分，但后来成为独立项目，以便与更广泛的开源社区协作。

在 FreeBSD 11 中，我们已迁移到 ELF Tool Chain 的 addr2line、c++filt、objcopy、nm、readelf、size、strip 和 strings 实现。libelf 和 libdwarf 库也来自 ELF Tool Chain。

ELF Tool Chain 还包含 FreeBSD 的 ar、brandelf 和 elfdump 工具的版本。如果它们将来获得引人注目的新特性或改进，我们将迁移到这些版本。

## 运行时库

工具链需要若干库来提供运行时支持。compiler-rt 库来自 LLVM 项目，包含若干组件。GCC 或 Clang 代码生成所需的低级目标特定钩子称为“builtins”。这些是执行编译器不会直接输出到对象代码中的操作的例程。

compiler-rt 包含 sanitizer 运行时，为 Clang 提供识别错误的软件行为的运行时支持。AddressSanitizer（asan）检测对堆、栈和全局变量的越界访问、释放后使用及相关错误，以及重复释放和无效释放。可通过在编译器命令行中添加 `-fsanitize=address` 来启用。

UndefinedBehaviourSanitizer（ubsan）捕获未定义行为，即语言未规定行为的操作。示例包括整数运算溢出（如加法和位移）、除以零以及使用未初始化的变量。命令行参数 `-fsanitize=undefined` 启用 ubsan。

Clang 在上游仓库中还包含其他 sanitizer，包括用于检测多线程应用程序数据竞争的 ThreadSanitizer 和用于检测未初始化读取的 MemorySanitizer。正在开展工作，使这些 sanitizer 在后续 FreeBSD 版本中随系统编译器可用。

对于 C++ 运行时支持，我们组合使用 PathScale 的 libcxxrt 来提供低级应用二进制接口（ABI）支持。这包括运行时类型信息（RTTI）、异常处理和线程安全的初始化器。C++ 标准库是 LLVM 的 libc++。

## 调试器

替代调试器同样来自 LLVM 项目：LLDB。与 LLVM 的其余部分一样，LLDB 作为一组可复用组件构建，并基于 LLVM 和 Clang 的其他部分。LLDB 可访问一个完整的 Clang 编译器，并将其用作表达式解析器。这为解释用户表达式提供了高保真度：LLDB 能够解析在被调试的原始源代码中有效的任何表达式。

FreeBSD 11 在 32 位和 64 位 ARM 架构以及 64 位 Intel（amd64）上包含 LLDB。

除基本系统中的 LLDB 外，GNU GDB 调试器可通过从 Ports 树构建或安装软件包来使用。它已更新，包含改进的线程支持和内核调试。通过简单的 `pkg install gdb` 命令即可使用 GDB 7.11.1。

## DTrace

FreeBSD 11 支持在用户空间中使用压缩类型格式（CTF）数据。这意味着当相应的可执行文件或库以 CTF 编译时，用户空间的函数边界跟踪（FBT）探针可以具有类型化参数。FreeBSD 基本系统构建基础设施支持通过 **/etc/src.conf** 文件中的设置以 CTF 编译。

FreeBSD 11 还在用户空间中新增对静态定义跟踪（STD）探针的支持。包含探针定义的 .d 文件可以包含在源文件列表中，构建基础设施会自动生成头文件，其中包含可在源文件中引用的探针宏。

## 未来工作

若干额外的工具链组件正在开发中，未来将在 FreeBSD 中可用。LLVM 提供代码覆盖工具 llvm-cov，它可以在类似于 GNU gcov 的模式下运行，并可与 Clang 基于插桩的性能分析配合使用。LLVM 还提供 OpenMP 运行时（libomp），这是一个用于共享内存多处理的应用编程接口（API）。

以 LLD 作为链接器后，GNU binutils 中仍剩两个工具：汇编器（as）和 objdump。目前没有替代 as 的候选。然而，在一级架构上构建 FreeBSD 基本系统实际上并不需要它：Clang 的集成汇编器用于构建汇编源文件。GNU as 可以直接移除，或者可以从集成汇编器构建与 GNU as 兼容的驱动。objdump 也不是基本系统构建所必需的，可以移除。LLVM 提供了 llvm-objdump，目前功能有限，但适时可能会成为可行的替代品。

FreeBSD 社区和上游 Clang/LLVM 项目正在持续推进，试图将 Clang、LLD 和 LLDB 套件带到仍在使用 GCC 的较低层级 FreeBSD 架构上。

---

**ED MASTE** 为 FreeBSD 基金会管理项目开发，并在剑桥大学计算机实验室担任工程支持职务。他也是选举产生的 FreeBSD Core Team 成员。除 FreeBSD 和 ELF Tool Chain 外，他还是 LLVM、QEMU 和 Open vSwitch 等其他若干开源项目的贡献者。他与妻子 Anna 和儿子 Pieter、Daniel 居住在加拿大基奇纳。
