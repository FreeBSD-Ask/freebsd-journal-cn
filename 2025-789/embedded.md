# CHERIoT

- 原文：[CHERIoT](https://freebsdfoundation.org/our-work/journal/browser-based-edition/embedded-2/27494/)
- 作者：David Chisnall

[CHERI](https://cheri-cpu.org/) 项目与 FreeBSD 一直关系密切。最初的出发点是这样的洞察：基于 Capsicum 的分区隔离对新代码非常有效，但要把它改造进既有库（每个库实例对应一个进程）却很难，原因有二：

首先，库希望共享复杂的数据结构；当把接口改造成通过某种进程间通信（IPC）通道发送消息时，会引入大量序列化开销。在一个普通库中，函数调用只需传递对象的指针即可共享数据结构。某个进行特权分离的库则需要对调用方与被调方之间移动的一切进行授权。库还常常需要长期共享，这又会带来额外的同步开销。

其次，进程是通过内存管理单元（MMU）进行隔离的。MMU 提供了一种虚拟内存抽象，将虚拟地址空间中的地址映射到底层物理内存。现代 MMU 很快，因为它们有转换后备缓冲区（TLB），这是一个快速的地址翻译缓存。TLB 会缓存虚拟地址到物理地址的翻译。如果同一页被 10 个进程共享，就需要占用 10 条 TLB 表项。MMU 在隔离方面很强，但在共享方面很差。

这两个问题引出了一个普遍结论：隔离容易，共享难。

[CHERI](https://cheri-cpu.org/) 是一组体系结构扩展，为从汇编代码往上的一切提供细粒度的内存安全。与 Capsicum 一样，CHERI 是个能力（capability）系统。在能力系统中，每个动作都必须伴随一项能力 —— 一种不可伪造的授权令牌 —— 来对该动作进行授权。

与之相对，许多其他优化机制包含“环境权限”（ambient authority）的概念：你可以仅因“你是你”而执行某个动作。比如，在传统的 UNIX 系统上（没有类似 FreeBSD MAC 框架或 SELinux 的机制），如果某进程以 UID 0 运行，它就能执行大量特权操作，不管它本意是否如此。

以 root 身份运行的进程往往是安全问题的根源，因为它们很容易被诱导去做不该做的事。在能力系统中，特权进程改为持有一组令牌，每个令牌分别授权某个具体动作。进程在每个时刻都需要选择要使用哪个令牌，从而避免无意中使用提升的权限。

MMU 也有类似的环境权限概念。正在运行的程序可以访问一切具有有效映射的内存。如果出现缓冲区溢出，MMU 并不会在意你并非有意访问相邻对象：只要你“有权”访问那段内存，你就能访问。相关的代码片段也可能持有指向另一个对象的指针，因此被授权访问该对象，但它在那个时间点并非有意如此。

Capsicum 将文件描述符扩展成了能力。

在进入能力模式（通过 `cap_enter` 系统调用）之后，带有丰富权限集合的 Capsicum 文件描述符开始发挥作用：进程如果不向系统调用提供具有正确权限的有效文件描述符（能力），就无法访问自身内存之外的任何东西。

例如，在一个普通的 POSIX 进程中，你可以调用 `open` 来访问用户（或某个涉及用户、进程以及其他因素的 MAC 策略）所能访问的任意文件。相比之下，能力模式下的进程必须调用 `openat`，并向其传入一个授权访问特定目录的能力。这样就贯彻了“有意性”的原则，避免了一大类漏洞。

再比如，如果你“本意上”要访问的是临时目录中的某个东西，但“碰巧”被给了一个你不该访问的、更重要目录的路径（例如 `<span> </span>../etc/rc.conf`），那么使用指向临时目录的文件描述符调用 `openat` 会失败（正确地阻止了利用），而 `open` 则会成功。

CHERI 保护内存访问的方式，类似于 Capsicum 保护文件系统访问的方式。传统的指令集架构提供取数、存数和跳转指令（有时如 x86，还与更复杂的操作组合在一起），这些指令都以某个地址作为基址。这种内存模型与传统 UNIX 对文件系统的看法很相似：只要进程被允许在某个位置进行取 / 存操作，任何取 / 存都可能成功。缓冲区溢出或释放后使用（use-after-free）之于内存，就如同路径遍历漏洞之于文件系统。

在 CHERI 系统中，这个“地址”被 CHERI 能力所取代；它是一种不可伪造的值，授权访问地址空间中的一个范围。这些值存放在寄存器和内存中，并由硬件防止篡改。

这如何让编程变得容易？

程序员是否必须把能力当作某种额外的东西来跟踪？

不，完全不用。

事实证明，大多数编程语言早就有一种抽象，用于表示“授权你访问一段内存的令牌”。它们把它称为指针（或在某些情况下称为引用）。作为一个以 CHERI 系统为目标的程序员，你大多数时候并不会想着 CHERI 能力，而只会想着指针。

如果你进行的指针算术把指针带出了对象范围，那么它就不能再用于存取。而且，由于硬件能精确知道内存中哪些东西是指针、哪些不是，所以在对象被释放时可以让指针失效。这意味着，在 CHERI 系统上，你只需把对象的指针作为参数传给库暴露的函数，就能与库共享对象。

最早期的 CHERI 操作系统是一款很小的微内核，但绝大多数工作都是在 FreeBSD 上完成的。CheriABI（面向 FreeBSD 的 CHERI 用户态 ABI）在 FreeBSD 的一个友好分支（[CheriBSD](https://cheribsd.org/)）上展示了一项完全内存安全的用户态与内核，该分支的目标是在 16.x 发布前将这些变更上游合入。CHERI 基础架构目前正作为 RISC-V 的一部分进行标准化，而 FreeBSD 16 预计将对所有即将推出、实现该指令集的应用级 CHERI 内核提供一等公民级支持。

FreeBSD 对 CHERI 项目至关重要。CHERI 是一项长期的软硬件协同设计项目，需要同时修改栈中的硬件与软件部分，以探索各种理念在何处、如何实现最佳。这要求一款便于修改的生产级操作系统。FreeBSD 清晰的结构和定义良好的抽象使这一切变得容易。我们后来在把 Linux 运行在 CHERI 上的工作中看到，相比之下，适配 FreeBSD 的工作量低得多；如果一开始就以 Linux 为目标，项目大概没有足够的软件工程师来完成同样的事情。其宽松许可也便于向其他操作系统的厂商展示各类硬件特性的使用方式。FreeBSD 还是 LLVM 的早期采用者，在易于修改性和许可方面同样具有优势。使用经过修改的 LLVM 编译整个 FreeBSD 基本系统非常容易，从而让测试新的 CPU 特性变得轻而易举。Brooks Davis 在 2023 年 5/6 月的 FreeBSD 期刊上撰写了一篇更长的[文章](https://freebsdfoundation.org/wp-content/uploads/2023/06/davis_cambridge.pdf)，介绍了 FreeBSD 对 CHERI 研究的益处。


## 缩减 CHERI

CheriABI 展示了，你可以在一个内存安全的世界里运行真正的 POSIX 应用，包括像 Chromium 这样的大型程序，以及带有 Wayland 和 3D 驱动的完整 KDE 桌面环境。它可以通过 COMPAT64 层与现有二进制共存，这一层的工作方式与 FreeBSD 上允许 64 位系统运行 32 位程序的 COMPAT32 层类似。从手机到服务器的大多数系统都采用了这一套已被验证有效的抽象。

2019 年，我们在微软的一些人决定尝试，看看能否将同样的抽象缩减到最小的系统中。

我们有三个问题：

- 在 64 位系统上运行良好的 CHERI 机制，能否在 32 位系统上也能运作？
- 如果有了 CHERI，可以舍弃哪些东西？
- 如果从零开始假定存在 CHERI，这个操作系统会是什么样子？

第一个问题并不是显而易见的。CHERI 提供的抽象似乎与地址大小无关，除了一个问题：CHERI 能力的所有元数据必须放进与地址同样多的比特中。这意味着在 32 位系统上，我们只有一半的空间存放元数据。

CHERI 使用一种压缩编码来表示边界，它利用了对象基址、对象顶部和指针指向该对象的地址三者之间的冗余。在较小的系统中，冗余更少，边界可用空间也更小。幸运的是，在嵌入式系统中，总的地址空间往往更小，因此没有必要精确表示非常大的地址区域。一个高端的微控制器通常总 RAM 远低于 4 MiB，所以大多数对象都非常小。

我们在权限上可用的比特位也比 64 位系统少，因此必须压缩权限编码，去掉那些对安全有害、无用或无意义的组合。

## 我如何在开发 CHERIoT 时使用 FreeBSD

当我们构建第一个 [CHERIoT](https://cheriot.org/) 原型时，有三个核心组件：

- 用 BlueSpec SystemVerilog 编写的 CPU 内核；
- 移植版的 LLVM；
- 从零开始的 RTOS。

BlueSpec SystemVerilog 是一种基于 Haskell 的高级硬件描述语言，使快速原型开发变得容易。[BlueSpec 编译器](https://github.com/B-Lang-org/bsc) 在 FreeBSD 上编译只需做少量小改动，我们把这些修改上游合并了，现在 FreeBSD 是一个受支持的平台。它目前还没进入 ports，但这会是一个很好的改进。有了它，我们就能在 FreeBSD 上构建运行 CPU 内核的模拟器。后来我们转向了用 SystemVerilog 实现的生产级代码，并使用（来自软件包的）verilator 在 FreeBSD 上构建模拟器。

在 FreeBSD 上进行 LLVM 开发非常容易。FreeBSD 是 LLVM 的一等公民目标，LLVM 的构建也非常简单。在某种程度上甚至“太容易了”：当我们需要支持一些 Linux 长期支持版本的用户时，发现引导我们使用的 LLVM 版本很困难，因为它依赖的 C++ 版本比他们系统自带的工具链更新。

而在前沿开发上，FreeBSD 更容易：ports 里提供多个版本的 GCC 和 LLVM。这也让我们很容易在其他系统上复现一些构建失败问题 —— 只需安装他们自带的旧版 GCC，然后配置构建使用它即可。一旦这些工作正常，交叉编译和测试 RTOS 就很容易了。

## 从 FreeBSD 汲取的经验教训

与 FreeBSD 一样，CHERIoT RTOS 是一款采用宽松许可、由社区开发的操作系统。

最重要的是，CHERIoT 旨在借鉴 FreeBSD“先设计再实现”的模式。改代码的最佳时机是“还没写出来的时候”。FreeBSD 之所以拥有 Jail、kqueue 和 Capsicum 等特性，正是源于在向用户抛出 API 之前，进行充分思考与反复推敲，而不是寄望它“也许能用”，然后承受后续的种种后果。

我们从 Capsicum 学到：可以把对程序员来说看起来像传统文件描述符或句柄的东西，做成能力（capability）。我们的软件抽象遵循这一模式。

我们也从 `kqueue` 学到：用一种单一、简单、统一的方式轮询任意阻塞事件是可行的。`kqueue` 的设计并不太适合一个实施特权分离的 RTOS，但其核心思想适用。我们的调度器把 futex（即“比较且相等则等待”的简单原子操作）作为唯一的阻塞事件源。中断被映射为 futex，因此线程只需等待一个 futex 就能等待中断。调度器在此之上叠加了一个多等待者 API，允许线程同时等待多个来自硬件或软件的事件。

或许更重要的是，我们从 FreeBSD 得到的启示是：文档为王。再优秀的系统，如果没有人能看懂并用起来，也无济于事。凭借撰写良好的手册页和《Handbook》，FreeBSD 容易上手。为希望学习 CHERIoT RTOS 的开发者撰写一本书是我们的优先事项之一，该书已于今年早些时候出版。除此之外，我们为每个 API 都写了文档注释，现代 IDE（以及使用我们版本 `clangd` 作为语言服务器协议实现的 vim）都能解析这些注释。

## FreeBSD 可以从 CHERIoT 学到的东西

CHERIoT RTOS 目前完全用 C++ 编写。相较于 C，C++ 在系统编程上有许多优势。它便于创建在编译期即可校验的丰富抽象。举例来说，我们有一个消息队列设计：为生产者和消费者指针各使用一个计数器，并需要正确处理这些值发生回绕的情形。

在 C++ 中，我们可以定义执行自增的 `constexpr` 函数，然后编写模板对其行为进行 `static_assert`。每次编译定义这些逻辑的文件时，编译器都会对某些小队列规模下生产者与消费者指针的所有可能取值，穷尽式检查其溢出行为。

使用丰富的类型还能让我们在“构造时”就避免大量错误。比如，我们有一个 `PermissionSet` 类来管理 CHERIoT 能力上的权限集合。它是一个 `constexpr` 集合，允许你按名称构造权限，并生成相关指令所需的位图。

在装载器中，我们用它同时描述我们希望赋予某个能力的权限，以及各个根对象各自具备的权限。如果我们试图派生一个原始权限中不存在的权限，编译就会失败。这比在后续某条指令因为其某个操作数缺少预期权限而失败，要容易调试得多。

我们大量采用这样一种模式：用一个内联模板函数做一些编译期检查（最终会被优化掉），然后调用一个类型擦除的函数。我们的类型安全日志就是这样实现的。我们有一个类似 `printf` 的函数，接受一个用于打印的参数数组及其联合体判别值。模板会基于类型生成这些判别值，因此无论你要记录的是指针、枚举值还是 MAC 地址，都能得到正确输出，而无需在格式字符串里手工匹配类型，同时还能进行编译期检查。

这套机制是可扩展的。比如 MAC 地址并非内建；由网络栈定义其回调以及处理映射的模板特化。

我们目前还没有 Rust 编译器（很快就会有！），但一旦具备，我们预计也会在部分组件中使用 Rust。具备更丰富类型的系统编程语言，既能在编译期避免缺陷，也能让我们写更少的代码。

如果使用 C，在 RTOS 的许多关键部位我们需要写更多源代码，这些代码也更难维护；同时更难找到熟悉该语言的开发者。就我们当下看到的各类系统编程语言的新代码行数而言，排序如下：

1. C++
2. C
3. Rust

如今找 C++ 开发者比另两者更容易，但这忽略了趋势。自 C++11 引入以降，C++ 一直在缓慢增长；同一时期，C 则在更陡峭、稳定地下滑。Rust 过去几年保持稳定增长，并在最近三年加速。我预计，很快找 Rust 开发者会比找 C 开发者更容易；再过大约十年，找 Rust 开发者也许会比找 C++ 更容易。

这并不意外。

开发者往往偏爱能让自己更高效的语言。FreeBSD 正在推进在基本系统中支持 Rust，但要采用现代 C++ 的路径更简单；C++ 能比 C 更容易表达复杂概念，并在编译期提供更多检查。

FreeBSD 还在非绝对性能关键的领域（包括用户态部分、引导加载器中的策略、以及 ZFS channel programs）出色地运用了 Lua。与上述语言相比，Lua 学习起来要简单得多，也更容易让经验尚浅的程序员快速产出。我们在构建系统中使用 Lua（以及为 CHERIoT 书籍排版！），但遗憾的是，Lua 虚拟机占用的内存大致等于一个典型微控制器的全部可用内存，因此我们无法在 RTOS 中使用它。

更丰富的系统编程语言很重要，但 FreeBSD 对 Lua 的运用提醒我们：很多事情——即便在内核里——其实并不一定需要系统编程语言来完成。

## 在 FreeBSD 上开发 CHERIoT

CHERIoT 工具链已经进入 FreeBSD 的 ports，`xmake` 构建工具也同样在其中，因此你可以很简单地通过以下命令来安装：

```sh
# pkg ins cheriot-llvm xmake-io git
```

这样你就具备了构建 CHERIoT 固件所需的前置条件。

```sh
# pkg ins cheriot-llvm xmake-io git
```

这将为你提供构建 CHERIoT 固件所需的前置条件。需要注意的是，在撰写本文时，季度分支中的 CHERIoT LLVM 版本是 18，而最新分支中的版本是 20（并即将更新到 21），因此在开发时最好使用最新分支。

现在，你应该可以尝试克隆 RTOS 并构建一个简单的固件镜像。

首先，克隆 RTOS：

```sh
$ git clone --recurse https://github.com/CHERIoT-Platform/cheriot-rtos
$ cd cheriot-rtos
```

接下来，你需要配置构建：

```sh
$ cd examples/01.hello_world/
$ xmake config --sdk=/usr/local/llvm-cheriot
checking for platform ... cheriot
checking for architecture ... cheriot
Board file saved as build/cheriot/cheriot/release/hello_world.board.json
Remapping priority of thread 1 from 1 to 0
generating /tmp/cheriot-rtos/sdk/firmware.rocode.ldscript.in ... ok
generating /tmp/cheriot-rtos/sdk/firmware.ldscript.in ... ok
generating /tmp/cheriot-rtos/sdk/firmware.rwdata.ldscript.in ... ok
$ xmake
...
[100%]: build ok, spent 13.796s
```

如果你有 lowRISC 的 Sonata 开发板，可以在 `xmake config` 命令行的末尾添加 `--board=sonata`，这样就会生成一个面向该开发板的 ELF 文件。该开发板会作为带有 FAT 文件系统的 USB 大容量存储设备出现，能够加载 UF2 文件。如果你运行 `xmake run`，它会提示你需要从 `pip` 安装用于将 ELF 转换为 UF2 文件的缺失 Python 软件包。安装完成后，如果 `SONATA` 文件系统挂载在常见位置，它要么会告诉你需要复制哪个文件，要么无需进一步交互就会直接完成复制。

如果你没有这块板卡，那么可以在模拟器中运行生成的固件。默认情况下，这些示例将以 Sail 模拟器为目标。这需要 OCaml，并且可以在 FreeBSD 上构建，但需要一些手动步骤。该项目还提供了一个 Linux 开发容器，在 FreeBSD 的 Linux 兼容层中配合 Podman 使用效果很好：

```sh
# pkg ins podman-suite
# podman run --rm -it --os linux -v path/to/cheriot-rtos:/home/cheriot/cheriot-rtos ghcr.io/cheriot-platform/devcontainer:x86_64-latest
```

这会让你进入临时容器，其中包含挂载在 `/home/cheriot/cheriot-rtos` 的 RTOS 源代码克隆。需要注意的是，ports 中当前版本的 Podman 并不喜欢在宿主操作系统与容器 OS 不匹配时使用指向多架构容器的标签。我们提供了 x86-64 和 AArch64 的二进制文件，所以如果你是在 AArch64 上，只需把上面命令中的 `x86_64` 替换为 `aarch64` 即可。

开发容器在 `/cheriot-tools` 中安装了所有工具，因此你可以像之前一样再次尝试构建示例：

```sh
$ cd examples/01.hello_world/
$ xmake f --sdk=/cheriot-tools
...
$ xmake
...
[100%]: build ok, spent 13.524s
$ xmake run
Board file saved as build/cheriot/cheriot/release/hello_world.board.json
Remapping priority of thread 1 from 1 to 0
Running file hello_world.
ELF Entry @ 0x80000000
tohost located at 0x80006448
Hello world compartment: Hello world
SUCCESS
```

恭喜你，你已经在由 ISA 形式化模型构建的模拟器中运行了内存安全的 C++ 代码！

开发容器还包含另外三个模拟器：

- 针对 Ibex 内核的逐周期精确（cycle-accurate）verilator 模拟器，带有最小化的外设集。
- 针对 lowRISC Sonata 开发板的模拟器。
- 来自 Google 的 MPact 模拟器，它提供了一个高性能模拟器，并带有集成调试器。

MPact 模拟器与 Sail（简化）机器兼容。

你可以直接运行它：

```sh
$ /cheriot-tools/bin/mpact_cheriot build/cheriot/cheriot/release/hello_world
Starting simulation
Hello world compartment: Hello world
Simulation halted: exit 0
Simulation done: 106508 instructions in 0.2 sec (0.5 MIPS)
Exporting counters
```

你可以在 FreeBSD 上进行开发，只在容器中运行模拟器；也可以选择在容器中完成所有开发。在其他平台上，大多数开发者会使用带有 dev container 集成功能的编辑器（例如 Visual Studio Code）。在 FreeBSD 上同样也可以这样做，只需配置 Podman 来运行容器的 Linux 版本即可。

另外，你也可以在 FreeBSD 上原生构建所有模拟器。相关的操作步骤过长，无法在本文中全部展开，但你可以参考 RTOS 仓库中的文档说明。它们所需的所有依赖已经在 ports 中，模拟器本身也有望很快进入 ports。

---

**David Chisnall** 的背景涵盖操作系统、编译器、硬件与安全领域。他是 *The Definitive Guide to the Xen Hypervisor* 的作者，自 2008 年起担任 LLVM 提交者，并在 2012 年至 2016 年期间担任 FreeBSD 核心小组成员。他于 2012 年加入 CHERI 项目，负责剑桥大学该研究方向中的编译器与语言部分。2018 年至 2023 年，他继续在微软从事 CHERI 相关工作，包括主导创建 CHERIoT。他现为 SCI Semiconductor 的联合创始人兼系统架构总监，该公司生产 CHERIoT SoC，同时也是 CHERIoT 平台开源项目的共同维护者。

