# FreeBSD 12.0 工具链更新

**作者：John Baldwin & Ed Maste**

FreeBSD 12.0 延续了近期 FreeBSD 版本的趋势，从过时的 GPLv2 许可证工具链组件向现代工具链过渡。在 12.0 开发周期中，这项工作主要集中于两个领域：尽可能使用更多基于 LLVM 的工具链，以及改善对现代 GNU 工具链的支持。因此，开发者在 12.0 开发周期接近尾声时，开始使用现代工具链独有的特性。

## 扩展 LLVM 工具链使用

FreeBSD 10.0 和 11.0 使用 clang C 和 C++ 编译器作为 x86 和小端 ARM 架构的默认系统编译器。但是，clang 生成的目标文件通过 GNU BFD 链接器链接为二进制文件和可执行文件。对于 x86 和 32 位 ARM，使用基本系统中的旧版 GPLv2 链接器。对于 64 位 ARM，必须从 ports 安装 GPLv3 链接器。

过去几年，LLVM 开发者对 LLD 链接器进行了大幅改进。FreeBSD 开发者通过补丁以及使用 FreeBSD 的基本系统和 ports 作为大型测试基础，发现并修复 bug 和缺失功能，为这项工作做出了贡献。因此，FreeBSD 12.0 现在为 64 位 x86、64 位 ARM 和 ARMv7 架构提供 LLD 作为链接器，取代了 GNU BFD 链接器。32 位 x86 系统也将 LLD 作为编译基本系统和内核的链接器。（对于 32 位 x86，少量 ports 依赖 GNU ld 特有的默认选项或行为，因此它仍安装为 **/usr/bin/ld**。）

除了 LLD 的变化，对其他架构的支持在 LLVM 中也有改进。MIPS 和 PowerPC 的支持在 LLVM 中已成熟。其中一些修复由 FreeBSD 开发者提交，另一些来自 LLVM 社区的其他成员。虽然这些架构尚未准备好在 FreeBSD 12.0 中使用基于 LLVM 的工具链，但正在取得进展。例如，64 位 MIPS 在合并后应该能够使用 LLVM 7.0 中的 clang 和 LLD。

## 外部 GNU 工具链

目前 LLVM 工具链不支持的架构也需要向更现代的工具链过渡。FreeBSD 树中的 GPLv2 工具链不支持 RISC-V 等较新架构。此外，使用现代 GNU 工具链为 LLVM 支持的架构构建基本系统，为用户提供了工具链选择。现代 GNU 工具链作为单独的软件包构建，使用 ports 框架，而非在基本源码树中维护 GPLv3 许可证的工具链。

GNU 工具链软件包有两种形式。第一组软件包将 GCC 和 binutils 作为附加工具链安装到 **/usr/local**，可用于本机构建或交叉构建。第二组软件包构建基本系统编译器，将 GCC 和 binutils 安装到 **/usr** 作为默认工具链。

附加工具链软件包针对每种架构由三个独立软件包组成：arch-binutils、arch-gcc 和 arch-xtoolchain-gcc。最后一个软件包依赖前两个软件包，所有这些软件包都从 devel 类别的 ports 构建。安装外部工具链后，可通过 `CROSS_TOOLCHAIN` make 变量用于构建内核和基本系统。传递给 `CROSS_TOOLCHAIN` 的值是“arch-gcc”。例如，要构建 32 位 MIPS world，可执行以下示例中的步骤。

示例：使用外部 GCC 构建 32 位 MIPS World：

```sh
# pkg install mips-xtoolchain-gcc
# cd /path/to/src
# make buildworld TARGET_ARCH=mips CROSS_TOOLCHAIN=mips-gcc
```

基本系统软件包由两个软件包组成：freebsd-binutils 和 freebsd-gcc。这些软件包从 base/binutils 和 base/gcc ports 构建。与附加工具链软件包不同，这些软件包替换基本系统工具链中的组件，如 **/usr/bin/cc** 和 **/usr/bin/ld**。这些软件包的 ports（连同 pkg(8) 本身）可以从非本机主机交叉构建。这将允许项目即使在项目不提供完整软件包仓库的架构上，也能提供工具链软件包。

即使使用 GNU 工具链，许多工具链组件仍由其他来源提供。例如，所有具有现代工具链的 FreeBSD 架构都使用来自 LLVM 的 libc++ 作为 C++ 运行时库。strip(8) 和 objcopy(8) 等实用程序由 ELF Tool Chain 项目提供。

FreeBSD 11 包含对附加工具链软件包和 `CROSS_TOOLCHAIN` 的支持。在 FreeBSD 12 开发周期中，工作集中于进一步细化此支持。例如，通过补丁和对工具链软件包的配置更改，改进了对 `--sysroot` 标志的支持。此外，构建系统更新为对外部工具链更友好，例如尽可能使用编译器驱动程序链接二进制文件，并支持不同的 MIPS ABI（如 N32）。

基本系统工具链软件包在过去两年中也得到积极开发。已添加对 MIPS 和 x86 架构的支持。适用于附加工具链软件包的相同 `--sysroot` 修复也修复了基本系统软件包中的类似问题。虽然在 FreeBSD 12.0 中它们尚未达到可替代任何架构上旧版 GPLv2 工具链的状态，但开发者已能在 32 位 MIPS 上构建并启动自托管 world 和内核。

## 使用现代工具链特性

迁移到现代工具链的好处之一是能够在基本系统中使用新工具链特性。FreeBSD 12 之前的工具链工作主要集中在为 x86 架构带来宽松许可的工具链以及支持 64 位 ARM 等新架构。但 FreeBSD 仍将旧版 GPLv2 工具链视为决定基本系统使用哪些工具链特性的最低公分母。

在 FreeBSD 12 开发周期接近尾声时，这一重点已转移。随着 LLD 成熟，FreeBSD 已实现在 ARM 和 x86 架构上使用宽松许可的工具链目标。因此，开发者已开始专注于使用（在某些情况下要求）仅现代工具链支持的功能。

突出的例子是在 x86 内核中使用间接函数（indirect functions）。间接函数是工具链特性，允许链接器在解析符号时调用函数，以确定该符号应解析为何地址。这通常用于为不同处理器提供优化例程。例如，C 运行时库可能提供使用 AVX 或 SSE 指令的字符串函数版本，并为当前 CPU 选择最优版本。FreeBSD 12.0 的 x86 架构内核使用此特性，为内存复制、TLB 刷新和 FPU 状态管理提供优化例程。FreeBSD amd64 内核还使用间接函数来支持 Supervisor Mode Access Prevention（SMAP）安全特性。展望未来，FreeBSD 13 开发分支已开始在用户空间 C 运行时库中使用间接函数，为内存清除和内存复制提供优化例程。间接函数的使用未来将在用户空间和内核中继续扩展。

## 结论

FreeBSD 12.0 标志着工具链开发的又一个里程碑。ARM 和 x86 架构现在使用现代、宽松许可的编译器和链接器。对外部 GCC 工具链的支持正在成熟。FreeBSD 13 将不再在任何架构的基本系统工具链中使用 GPLv2 组件。因此，FreeBSD 开发者未来将加速采用新工具链特性。这将涵盖从扩展间接函数的使用，到启用链接时优化（LTO）、构建标识符、压缩调试信息等新功能。

---

**JOHN BALDWIN** 是一名系统软件开发者。19 年来，他直接为 FreeBSD 操作系统提交了跨越内核各部分（包括 x86 平台支持、SMP、各种设备驱动程序和虚拟内存子系统）以及用户空间程序的更改。除了编写代码，John 还曾在 FreeBSD 核心团队和发布工程团队任职。他还为 GDB 调试器和 LLVM 做出了贡献。John 与妻子 Kimberly 和三个孩子 Janelle、Evan 和 Bella 居住在加州康科德。

**ED MASTE** 管理 FreeBSD 基金会的项目开发，并与剑桥大学计算机实验室合作，担任工程支持角色。他还是选举产生的 FreeBSD 核心团队成员。除 FreeBSD 和 LLVM 项目外，他还是其他几个开源项目的贡献者。他与妻子 Anna 和儿子 Pieter、Daniel 居住在加拿大基奇纳。
