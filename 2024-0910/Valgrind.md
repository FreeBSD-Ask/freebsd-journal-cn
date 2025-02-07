# FreeBSD 上的 Valgrind

- 原文链接：[Valgrind on FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/kernel-development/valgrind-on-freebsd/)
- 作者：Paul Floyd

我第一次使用 Valgrind 是在 2000 年代初期。在此之前，我有一些在 Solaris/SPARC 上使用 Purify（现在是[Unicom PurifyPlus](https://www.unicomsi.com/products/purifyplus/)）的经验。老实说，我对[Valgrind](https://valgrind.org/)并没有特别的印象。虽然它不需要特别的构建过程，但它缺少与调试器交互的能力。

稍微转到 FreeBSD，我第一次安装的是 1995 年底的 2.1 版本。像 Valgrind 一样，最初我对它也没有太大兴趣。至少它是我家用 PC 上的“一个 Unix”，如果我需要的话。我继续在 FreeBSD 上小试牛刀，偶尔安装新版本。我的主要家用系统一直是 OS/2，直到 90 年代末；之后是长期使用的 Solaris，直到 11.4 版本进入了“停滞期”。我在 2007 年还买了一台 MacBook，主要用于“桌面”工作——我并不认为在 macOS 上开发是一种令人愉快的体验。

我一直对质量抱有信念。在大学时，我上过一门关于产品质量的短期课程，这让我坚定了这一信念。后来，我读了一些[W. Edwards Deming](https://en.wikipedia.org/wiki/W._Edwards_Deming)的著作。虽然他的文字有些粗糙，但他的思想非常明确有力。由于我学的是电子学，显而易见，日本公司在 20 世纪后半期通过质量流程获得的好处。我最终在电子仿真领域作为软件开发人员工作。不出所料，我继续使用像 Valgrind 这样的工具。

大约五年前，我决定是时候开始为开源社区做出一些贡献了。既然我已经是 Valgrind 的专家，并且已经稍微接触过它的源代码，这对我来说是一个逻辑性的项目。我犹豫了一下，是在 macOS 和 FreeBSD 的 Valgrind 移植之间做选择。两件事让我对 macOS 产生了抵触——频繁的操作系统和用户空间的重大更新，导致一切都无法兼容，以及从 Apple 内部获得帮助的难度。虽然有 XNU 源代码和一些书籍，但除此之外，你就只能独自摸索了。最后，我选择了 FreeBSD。这也很适合我，因为我正在考虑摆脱 Solaris 的使用。Illumos 和 FreeBSD 之间有很多交叉合作，我认为这会让我更容易过渡。与此同时，macOS 仍然存在于 Valgrind 的官方代码库中，但自 2016 年 10.12 版本以来，它基本上无法使用了。

## Valgrind 的历史

Valgrind 现在已有超过[20 年的历史](https://nnethercote.github.io/2022/07/27/twenty-years-of-valgrind.html)。它最初在 i386 Linux 上启动。随着时间的推移，添加了多个其他 CPU 架构（amd64、MIPS、ARM、PPC 和 s390x），以及其他操作系统（macOS、Solaris，最近是 FreeBSD）。

这些工具在这 20 年里不断发展。从 2002 年初始版本开始，添加的工具包括：

**2002 memcheck  
2002 helgrind  
2002 cachegrind  
2004 massif  
2006 callgrind  
2008 drd  
2009 exp-bbv  
2018 DHAT**

此外，还有一些工具在树外进行维护（或不再维护）。

Valgrind 的开发由少数几个人进行，大约有二十位做出了重要贡献。一些公司也为其提供了帮助。RedHat/IBM 可能是贡献最多的公司。Sun 在 Solaris 积极开发期间也有贡献。Apple 也曾贡献过，直到他们突然变得反感 GPL。

## Valgrind 在 FreeBSD 上的历史

Valgrind 在 FreeBSD 上的历史相当长且充满波折。我不会提及每一个做出贡献的人（而且我甚至不确定是否有完整的名单，因为一些源代码仓库已经无法访问）。Doug Robson 在 2004 年做了大量的初期工作。接下来的接力者是 Stan Sedov，他在 2009 到 2011 年期间维护了这个 Port。在那时，曾有一段时间推动将 FreeBSD 的源代码提交到上游，但最终未能成功。上游的维护者对质量标准要求非常严格，FreeBSD 的 Port 虽然接近合格，但始终未能达到要求。其次，需要有人维护 Port，最好是 Valgrind 团队的成员。我不知道为什么这一点从未发生过。从 2021 年 4 月开始，我开始维护 [FreeBSD Port](https://github.com/paulfloyd/freebsd_valgrind)，并且已经有 4 年多时间拥有 Valgrind 的提交权限。现在，我是 Valgrind 的主要贡献者。

最近的一次重大变更是添加了对 aarch64 的支持。我在 2024 年 4 月为此 CPU 添加了 Port，及时赶上了 Valgrind 3.23 版本的发布。

## Valgrind 工具

在深入 Valgrind 的内部之前，我将简要概述这些工具。

### Memcheck

这是大多数人在提到 Valgrind 时会想到的工具。它是默认工具。Memcheck 的主要功能是验证内存读取是否来自已初始化的内存，并确保读取和写入操作在分配的堆内存块的边界内。缺失的部分是检查栈内存的边界——这需要使用插桩技术。

### DRD 和 Helgrind

这两个工具都是线程安全检测工具。它们将检测不同线程访问内存时没有使用某种锁机制的情况。它们还会警告 pthread 函数使用中的错误。两者的区别在于，Helgrind 会尝试为所有涉及的线程提供错误上下文，而 DRD 仅为一个线程提供详细信息。

### Callgrind 和 Cachegrind

这两个工具用于 CPU 性能分析。Callgrind 用于分析函数调用。Cachegrind 通常用于使用基本缓存和分支预测模型分析 CPU 指令。这些模型从来都不是很准确，现在甚至显得非常不现实。此外，Valgrind 并不执行任何预测执行。基于这些原因，当前版本的 Valgrind 默认不再使用 Cachegrind 进行缓存仿真。一些人喜欢指令计数的精确性，但就我个人而言，我通常更喜欢像 Google 的[perftools](https://github.com/gperftools/gperftools)（[port devel/google-perftools](https://www.freshports.org/devel/google-perftools/)）、Linux 的 perf 和[gprofng](https://www.sourceware.org/binutils/docs/gprofng.html)这样的采样分析工具，尤其是在处理大型问题时（运行时间为小时或天，内存使用量达到数百 GB）。

### Massif 和 DHAT

这两个工具是内存分析工具。Massif 用于对内存使用情况进行时间序列分析。就我个人而言，我觉得它有些过于复杂。实际上，还有其他工具通常可以在没有 Valgrind 开销的情况下，提供同样优秀的分析结果——例如 Google 的 perftools，和[HeapTrack](https://github.com/KDE/heaptrack)（[port devel/heaptrack](https://www.freshports.org/devel/heaptrack/)）。不过，还是有一个例外。如果你的应用程序大量使用基于 mmap 的自定义分配器，或者静态链接了 malloc 库，那么这些替代工具就无法工作。Massif 不需要插桩共享库中的分配函数，它还可以选择在 mmap 级别对内存进行分析。DHAT 是 Valgrind 工具套件中的隐藏宝石。这个工具分析堆内存的访问情况。它提供的信息可以帮助你了解哪些内存被频繁使用，哪些内存长时间保持分配状态，哪些内存从未被使用。对于那些不太大的内存块，它还会为这些块生成访问直方图。从中，你可以看到结构体或类中的空洞或未使用的成员。你还可以推断访问模式，这可能有助于重新排序成员，使它们位于同一缓存行上。

## Valgrind 基础

### 无依赖

为了让 Valgrind 能够执行所有客户端代码（不仅仅是 main()中的代码，而是程序启动时 ELF 文件中的第一条指令），并避免与诸如 stdio 缓冲区等产生冲突，Valgrind 不与 libc 或任何外部库链接。我有时开玩笑说，这不太像 C++，更像 C- -。这意味着 Valgrind 有自己的 libc 子集实现。为了使函数名不冲突，它使用宏作为伪命名空间。Valgrind 版本的*printf*是*VG\_(printf)*（对于代码导航来说非常有趣！）。这也意味着我们不能直接添加第三方库并使用它。这个库需要被移植，以便使用 Valgrind 的 libc 子集。例如，目前有一个 bugzilla 项，计划添加对 zstd 压缩的 DWARF 段的支持。

### 多疑的编程

Valgrind 非常谨慎，广泛使用了在发布版本中启用的断言。这使得它的速度略有降低，但考虑到出错的可能性，最好是诚实地直接崩溃，而不是假装继续运行下去。

Valgrind 具有广泛的详细信息和调试信息。你可以通过重复使用-v 和-d 最多 4 次来提高调试/详细程度。除此之外，还有一些更为针对性的跟踪选项，比如`--trace-syscalls=yes`。在 Valgrind 中进行调试可能相当困难，这些输出在开发新功能时会是很大的帮助。它们也对支持工作非常有用，例如要求用户上传日志到 Valgrind 的 bugzilla。

### 代码复杂性

Valgrind 本身有点复杂。开发 Valgrind 最困难的部分之一就是它涉及的内容非常广泛。它需要虚拟化四种 CPU 架构（Intel/AMD、ARM、MIPS 和 PPC，并包含一些子变种）。每种架构都有数千页的手册。你通常需要了解所有操作码，甚至是它们可能变化的每一位。你需要对 C、C++和 POSIX 有深入了解。你还需要知道哪些操作系统系统调用需要特殊处理。了解 ELF 标准也很重要——因为 lld 和 mold 的实现方式不同，我们曾遇到过相关问题。除了 ELF，还有用于调试信息的 DWARF。到目前为止，我仅覆盖了 Valgrind 的核心部分。

尽管 Valgrind 的复杂性很高，但我认为它并不包含大量的代码。一个干净的 git 克隆，不包括回归测试，大约有 50 万行代码。加上回归测试，总行数大约为 75 万行——虽然回归测试的数量大约有 1000 个，但其中有一些非常庞大的，涵盖了大量的比特模式组合，使用脚本生成所有输入组合进行测试。

这些工具本身只占代码量的不到 10%。主要的代码量来自 CPU 仿真和“核心”部分。核心包括很多内容——libc 替代、系统调用包装器、内存管理、gdb 接口、DWARF 读取器、信号处理、内部数据结构和函数重定向。

开发 Valgrind 时的一个额外难题是，由于它完全静态化，你无法在其上使用 sanitizers。然而，你可以在 Valgrind 内部运行 Valgrind！这需要一个特殊的构建，最终你会得到一个外部 Valgrind 和一个内部 Valgrind，内部 Valgrind 是外部 Valgrind 的“客体”，而内部 Valgrind 的“客体”是一个可执行程序。显然，这样会使得一切变得更慢。我确实使用免费的 Coverity Scan 服务对 FreeBSD 上构建的 Valgrind 进行静态分析。它大多数时候发现的是常见的假阳性，但也找到了几处实际的 bug，包括一些是我自己添加的。我仍然需要做一些工作，为 Valgrind 的内部 libc 替代，特别是分配函数，提供代码模型。

## Valgrind 在运行时

### 客户执行

Valgrind 中的 CPU 仿真被称为 VEX（不要与 Intel 向量扩展混淆）。VEX 的起源不太清楚，可能是“Valgrind Emulation”的缩写。

当 Valgrind 运行时，实际上只有一个进程——宿主进程。不会使用 ptrace（调试器如 lldb 和 gdb 使用的工具）。客户（有时称为客户端）可执行文件在宿主进程中运行，使用动态二进制插桩（DBI）。为了执行插桩，Valgrind 进行动态重编译，使用即时编译（JIT）。这一过程如下：

- 读取一段机器代码。
- 将这些机器码转换为 Valgrind 中间表示（IR）——这与编译器使用的表示方式相同，恰好 Julian Seward 也曾参与过格拉斯哥 Haskell 编译器的工作。
- 根据工具的需要，对 IR 进行插桩。
- 对 IR 进行优化和重写。
- 将 JIT 生成的操作码存储在缓存中并执行它们。

### 内存隔离

Valgrind 有自己的内存管理器。它严格区分宿主进程使用的内存和客户进程使用的内存。许多工具会替换 C 和 C++的分配与释放函数。对于这些工具，所有的内存管理都由 Valgrind 的内存管理器处理。像 cachegrind 和 callgrind 这类工具不会替换内存分配器（因此，它们会在性能分析中包括分配器）。

### Valgrind 启动

Valgrind 首先在其自己的\_start 例程中以汇编语言启动（记住没有 libc），它做的第一件事是为自己创建一个临时堆栈，设置日志记录，并设置堆分配器。我想要强调的一点是，出错的空间非常有限。如果发生了错误，幸运的话，你可能只是看不到文件名和行号。如果不幸运，你只会看到一堆十六进制地址堆栈跟踪。可以想象，如果没有堆栈，程序会很快失败。待 Valgrind 完成了所有内部设置，它就准备好在合成 CPU 上启动客户可执行文件。它为自己创建了另一个堆栈，大小可以配置，然后启动客户可执行文件。从客户可执行文件的角度来看，它就像是在本地运行一样。

### 处理系统调用、线程和信号

Valgrind 拦截所有系统调用。幸运的是，大多数系统调用要么什么也不做，要么只是进行一些检查（比如寄存器中是否包含已初始化的内存），然后将其转发到内核。更复杂的系统调用则会根据某些操作码（例如 umtx_op 和 ioctl）有不同的行为。最后，还有一些系统调用不会转发到内核，需要 Valgrind 自行实现。例如，‘getcontext’系统调用，Valgrind 需要从其合成 CPU 填充上下文，而不是让内核从 Valgrind 宿主的上下文中填充。

有一件棘手的事情是，运行在虚拟 CPU 上的代码必须保持在虚拟 CPU 上。虽然 Valgrind 会在物理 CPU 上本地执行某些客户代码，但通常这种本地执行范围非常有限。如果客户的控制流逃回物理 CPU，事情就会变得非常糟糕。我将举两个例子，说明为了确保 Valgrind 保持控制所需的扭曲。首先是线程创建。当调用‘pthread_create’时，Valgrind 需要确保操作系统不会运行作为第三个参数传递的函数。相反，它需要用一个“run_thread_in_valgrind”函数挂钩第三个参数。类似地，对于信号，Valgrind 需要确保客户信号处理程序在 Valgrind 下运行，然后信号处理程序返回后也继续在 Valgrind 下运行。这些事情需要一些非常 hacky 的代码。Valgrind 还需要大量调整信号掩码。当客户程序运行时，信号会被屏蔽，宿主会轮询并处理信号。当发生系统调用时，信号会被解屏蔽，执行系统调用，然后再次屏蔽信号。没有这个小舞蹈，阻塞的系统调用就无法被中断。

## Valgrind 移植

当我开始查看 Valgrind 的移植时，它的状态很糟糕。如前所述，从 2009 年到 2011 年，曾有一段推动将其移植到上游的努力。从 2011 年到 2018 年，它的维护变得非常有限。

由于在‘stat’系列函数中添加对大文件支持的更改，Valgrind 在 amd64 上的版本已经损坏。一些人找到了修复该问题的补丁。而 i386 版本则有多种问题。没有 FreeBSD 特定的回归测试。Valgrind 包含许多在所有平台上运行的测试，然后是所有操作系统和 CPU 架构的组合（例如，amd64、freebsd 和 amd64-freebsd）。大约有 600 个这样的通用测试。Linux amd64 平台上，在这些通用测试的基础上还增加了大约 200 个测试。我记不清这些通用测试有多少通过，多少失败了，可能通过的也不多于一半。幸运的是，有很多问题是低垂的果实。在解决了 i386 上的一些严重问题后，经过大约六个月的努力，我让大约 90%的回归测试通过。听起来不错，但仍然存在一些严重的限制。完成剩下的 10%几乎就像是“最后的 10%需要 90%的时间”。

## 战斗故事

### 信号导致断言失败

信号。哦，我一开始真是很难理解这一切。当程序本地运行时，信号将执行以下操作：

- 内核合成一个 ucontext 块，其中包含信号发生的地址和堆栈中的调用帧（或备用堆栈），并将调用帧的返回地址设置为‘retpoline’（一个用于从信号处理程序返回的小型汇编函数）
- 内核将运行的 exe 转移到信号处理程序
- 信号处理程序执行相关操作并返回
- retpoline 调用 sigreturn 系统调用
- 内核从内容中获取信号发生前的原始地址，并将执行转移到该地址

在 Linux 上，对于非线程化和线程化应用程序，这一过程适用。对于 FreeBSD，待你链接了 libthr，这一过程就会有所变化。‘thr_sighandler’替代了用户的信号处理程序。它进行一些信号屏蔽等操作，调用用户的信号处理程序并自己调用 sigreturn。

Valgrind 不能让客户代码执行。所以，它处理所有可能的信号。它合成自己的上下文，加入了更多的信息。它用自己的*run_signal_handler_in_valgrind*函数替换了客户的信号处理程序。返回地址设置了自己的 retpoline，它将调用*valgrind_sigreturn*，将客户程序的执行转移回原来的位置。那么可能出了什么问题呢？事实证明，几乎所有的东西都有问题。至少有两件事情在这个流程中被破坏，我都处理过了。

第一个问题是一个非常小的代码更改。当从客户信号处理程序返回时，Valgrind 在 i386 上崩溃了。经过大量调试，我将问题缩小到汇编语言中的 retpoline 函数 VG\_(_x86_freebsd_SUBST_FOR_sigreturn_)。某个时刻，ucontext 结构的大小发生了变化。VG\_(_x86_freebsd_SUBST_FOR_sigreturn_)在错误的偏移位置寻找返回地址——0x14 而不是 0x1c。这意味着虚拟 CPU 在某个无效的地址处恢复执行。砰！很快就触发了断言。

我与信号的第二次大战是间歇性的。如果信号在 Valgrind 执行“普通”客户代码时到达，那是非常好的，因为它知道应该从哪里恢复。但如果信号在系统调用时到达呢？情况就变得复杂了，因为系统调用是 Valgrind 让客户在物理 CPU 上运行的地方之一。Valgrind 不能在其全局锁中执行客户的系统调用。系统调用可能会阻塞，这会导致多线程进程挂起。相反，它释放锁，然后执行系统调用。现在，如果在锁被释放的窗口期间发生了中断，Valgrind 需要尝试弄清楚中断发生的位置，以便决定是否需要重新启动。为此，执行客户系统调用的机器代码函数 ML(_do_syscall_for_client_WRK_)有一个与之相关的地址表，包含了设置、重启、完成、提交和结束的地址。这个方法通常很好用，但偶尔会因断言失败而出错。问题出在系统调用状态的设置上。在 Linux 上，它只保存在 RAX 寄存器中，并从小型汇编函数返回，所以无需做任何特殊处理。而在 FreeBSD（和 Darwin）上，它保存在进位标志中。这需要调用一个函数来在合成 CPU 中设置进位标志。如果信号在 Valgrind 调用*LibVEX_GuestAMD64_put_rflag_c*函数时到达，这个情况就没有处理——导致了断言。遗憾的是，在 C 中没有简单的方法来判断指令指针正在执行哪个函数。你可以轻松地获取函数开始的地址，但函数结束的位置在哪里呢？我曾考虑过使用 Valgrind 的 DWARF 调试信息（它应该始终存在，且 Valgrind 内置了 DWARF 读取代码）。最后，我采取了一个丑陋且不标准的方法。我获取了*LibVEX_GuestAMD64_put_rflag_c*之后的一个虚拟函数的地址。即使没有保证编译器和链接器会按照源文件中的顺序布局函数，这个方法在 i386 和 amd64 上都有效。然而，后来当我处理[aarch64 移植](https://github.com/paulfloyd/freebsdarm64_valgrind)时，这个方法就不行了，因为设置进位标志的函数使用了几个辅助函数，而这些辅助函数并没有按顺序布局。因此，我改为在执行客户系统调用的汇编例程中设置一个全局变量。

### GlusterFS swapcontext 崩溃

另一个战斗故事。这是我在发布重新启动的 FreeBSD Valgrind 后收到的第一个错误报告之一。一个运行 GlusterFS 的用户在使用 Valgrind 时遇到了崩溃。经过一番反复询问日志文件和跟踪信息后，我将问题缩小到 swapcontext 系统调用。原来，切换到的上下文中有两个指向信号掩码的指针，而 Valgrind 只设置了第一个指针。这又是一次调试了几天才只改了一行代码的问题。

### FreeBSD 问题

我在 Valgrind 上做的工作还揭示了 FreeBSD 中的一些 BUG。我在处理 i386 二进制文件在 amd64 上运行时，早期就遇到过其中一个问题。我在 i386 与 i386、amd64 与 amd64 上没有问题，但 i386 在 amd64 上运行时，在客户启动的早期就崩溃了，发生在链接加载器（lib rtld）阶段。最终，我发现这是一个与页面大小检测有关的问题。正常的独立应用程序将这些信息存储在它们的辅助向量（auxv）中，包含 AT_PAGESZ（实际页面大小）和 AT_PAGESIZES（指向可能页面大小表的指针）。Valgrind 为客户合成了 auxv，但当时它忽略了 AT_PAGESIZES。没有问题，rtld 有回退机制，会使用 HW_PAGESIZE 的 sysctl。i386 有两种可能的页面大小，而 amd64 有三种可能的页面大小。不幸的是，发生的情况是，运行在 amd64 上的 rtld 使用了三种页面大小，但 i386 内核组件使用了两种页面大小。结果，sysctl 返回了[ENOMEM](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=246215)。

## 房间里的大象——Sanitizers

既然我们有了 Sanitizers，为什么还要使用 Valgrind？我也反过来问，既然我们有 Valgrind，为什么还要使用 Sanitizers？大致来说，Address Sanitizer 和 Memory Sanitizer 相当于 Memcheck，而 Thread Sanitizer 相当于 DRD 和 Helgrind。UB sanitizer 没有 Valgrind 的对应工具。

有一种情况使用 Valgrind 根本不可行，那就是在使用不受支持的 CPU 架构时。Valgrind 在 FreeBSD 上只支持 amd64、i386 和 aarch64。如果你使用的是其他架构，那么 Valgrind 就无法使用。接下来，Valgrind 滞后于 CPU 的开发。这意味着如果你的应用依赖于使用[AVX512](https://bugs.kde.org/show_bug.cgi?id=383010)，你就无法使用 Valgrind。

如果 Sanitizers 和 Valgrind 都能在你的系统上工作，你该选择哪个呢？一如既往，这取决于情况。

|              | Valgrind                 | Sanitizer                    |
| ------------ | ------------------------ | ---------------------------- |
| 速度         | 非常慢，有时几乎无法使用 | 慢                           |
| 堆栈边界检查 | 否                       | 是                           |
| 是否需要插桩 | 否                       | 是                           |
| 可用性和支持 | amd64、i386、aarch64     | amd64、i386、aarch64、risc-v |

当我说 Valgrind 不需要插桩时，那是个小白谎言。如果你使用自定义分配器，那么你需要为 Valgrind 或 Sanitizers 写一些注解，才能确保它们正常工作。同样，如果你使用了自定义线程锁定例程（如自旋锁），在这两种情况下你也需要进行注解。Thread Sanitizer 的优势在于，它对不依赖于 pthread 的标准库机制（如 std::atomic）具有内置注解。

FreeBSD 很幸运，工具链是基于 LLVM 的。这意味着内存 Sanitizer 很容易获得。而 GCC 没有内存 Sanitizer，这使得在 Linux 上使用起来更加困难。不要低估“需要插桩”这一要求有多大。为了获得最佳结果，这意味着你应该对所有依赖的库进行插桩。如果你是 KDE 应用程序的开发者，那至少需要插桩以下几组库：KDE、Qt、libc++。还有许多其他依赖（如 libfontconfig、libjpeg 等）。正如我们 Valgrind 的开发者常说的：“祝你好运！”如果你在一家大公司工作，有专门的 devops 团队来设置一切，那就不那么糟糕了。我也很感兴趣听听任何有使用 poudriere 构建 Sanitizer 版本经验的人。我还读过一些拥有大规模单元测试套件的人抱怨，在构建带有 Sanitizer 的版本时，构建时间和磁盘空间要求过高，特别是因为你不能做一个“一站式”Sanitizer 构建（address 和 memory Sanitizers 不兼容）。

我的结论是，你应该根据需求选择最适合的工具。

## 未来工作

不幸的是，Valgrind 是一个很容易退化的工具。FreeBSD 的新版本不断发布，带来了新的和变化的系统调用。辅助向量中不断增加新的项。`_umtx_op`命令也在增多。libc++不断找到使用 pthread 的更奇怪方式。编译器以看似不安全的方式优化代码。这意味着 Valgrind 的工作永远不会完成。

### CPU 架构

Valgrind 在 FreeBSD 上支持 amd64、i386 和 aarch64。我不认为自己会加入对 MIPS 或 PPC 的支持。RISC-V 尚未被加入到官方 Valgrind 源代码中——一个[RISC-V 移植](https://github.com/petrpavlu/valgrind-riscv64)正在进行中，但目前因为关于矢量指令实现的讨论而被拖延。

### 错误列表

Valgrind 的[Bugzilla](https://bugs.kde.org/buglist.cgi?bug_status=UNCONFIRMED&bug_status=CONFIRMED&bug_status=ASSIGNED&bug_status=REOPENED&list_id=2388402&product=valgrind&query_format=advanced)中大约有 1000 个未解决的错误。虽然其中许多只影响 Linux/macOS/Solaris，但也有不少影响 FreeBSD。

- Helgrind 在大量线程创建/销毁时会产生虚假警报，特别是线程局部存储（TLS）。这是因为 pthread 栈包括 TLS 的缓存。Valgrind 没有把回收的 TLS 看作是有不同内存地址的。Linux 通过一个 GNU libc 环境变量禁用 pthread 栈缓存来规避这个问题。但我还没有找到使用 FreeBSD libc 做同样事情的方法。
- 当客户程序生成核心转储时，实际上是 Valgrind 生成了核心文件。目前，核心文件的布局几乎与 Linux 的核心转储相同。这意味着 lldb 和 gdb 无法对核心文件做太多处理。我认为这不是一个大问题，因为现在很少有人使用核心文件了。
- 线程调度器。Valgrind 有一个非常基础的线程调度器。线程上下文切换发生在系统调用边界或每 100000 个基本块。默认调度器会释放全局锁，哪个线程获得锁完全取决于运气。如果前一个线程在 CPU 缓存中很热，那么它就有可能获得锁。Linux 有一个基于 futex 的可选公平调度器。虽然这个调度器不能直接移植到 FreeBSD，但用`_umtx_op`来实现它应该不会太难。
- 在 aarch64 上，偶尔会出现与线程局部存储中的访问相关的 DRD 虚假警报。
- 验证 ioctl 的代码非常有限。几乎所有的 ioctl 都只对它们的参数进行基本的大小检查。这个功能需要扩展，理想情况下还需要添加测试用例。

## 结论

在 Valgrind 上工作是一个巨大的挑战。调试可能非常困难——我常常发现自己一边调试客户程序，一边调试 Valgrind 运行客户程序，同时还用 vgdb 调试在 Valgrind 中运行的客户程序。我学到了很多关于 ELF、信号和系统调用的知识，当然，也学到了关于 Valgrind 本身的很多东西。总是有很多需要学习的地方——例如 aarch64 和 amd64 的操作码细节，以及动态重编译中使用的各种技巧。

---

**Paul Floyd** 自 2.1 版本以来间歇性使用 FreeBSD，并从 10.0 版本起开始认真使用。他已经是 Valgrind 开发团队的成员四年，拥有电子学博士学位，现居住在法国阿尔卑斯山边缘的格勒诺布尔，供职于西门子 EDA，开发模拟电子电路仿真工具。
