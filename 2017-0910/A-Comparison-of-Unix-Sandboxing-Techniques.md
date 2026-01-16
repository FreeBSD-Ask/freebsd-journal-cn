# Unix 沙箱技术比较

- 原文：[A Comparison of Unix Sandboxing Techniques](https://freebsdfoundation.org/wp-content/uploads/2017/10/A-Comparison-of-Unix-Sandboxing-Techniques.pdf)
- 作者：JONATHAN ANDERSON

为什么沙箱（sandboxing）不同于历史上的 Unix 安全方法、我们是如何发展至今，以及 Capsicum 与 Linux 的 seccomp(2) 和 OpenBSD 的 pledge(2) 的比较。

如今的用户需要比传统 Unix 系统能提供的更多保护。操作系统的开发者历来非常关注系统级的特权概念（例如，通过模块向内核注入代码的权限），但计算机系统的用户往往需要更细粒度的访问控制模型（例如，共享单个联系人或委托管理单个日历的能力）。当今终端用户设备的操作系统不再只是保护多个用户彼此之间的安全，而是必须保护单个用户免受其应用程序的影响，并保护这些应用程序互相之间的干扰。传统 Unix 衍生系统并没有让这项任务变得容易；在某些情况下，用户所需的保护甚至是不可能实现的。

保护曾是早期通用操作系统及其运行硬件的核心目标 [Lamp69, And72, SS72]。这种早期关注自然地推动了严谨的、通用保护原语的探索和设计，例如能力（capabilities）[DV66] 和虚拟内存 [BCD69, Lamp69]。然而，在从 Multics 向 Unix 主导地位过渡的过程中，这种关注逐渐消失。其结果是，高度可移植的操作系统诞生了，它主导了当代对操作系统的思考，但其安全特性主要围绕单一威胁模型组织：用户攻击其他用户（包括开发中软件的意外破坏）。这种安全模型——自主访问控制（Discretionary Access Control, DAC）——可以通过 Unix 的 owner/group/other 权限或访问控制列表（ACL）实现，但它对应用程序沙箱化的支持不足。

FreeBSD、Linux 和 MacOS 最终引入了强制性安全策略执行框架（如多级安全和完整性执行）[WFMV03, WCM+02, Wat13]，统称为强制访问控制（Mandatory Access Control, MAC）。这些策略体现了系统所有者和管理员的利益，并提供了额外的维度来指定强制执行，但它们更适合用于保护高完整性文件免受低完整性数据影响，而不适合支持非特权应用程序中的沙箱化。

沙箱化的目标是在应用程序接触不受信任的内容时保护用户免受自身应用程序的影响。复杂应用程序经常会接触来自恶意来源的内容，这些内容通常嵌入在难以解析的协议和文件格式中。在互联网环境下尤其如此，即便是最基本的使用场景，也涉及 ASN.1 解析（用于 TLS）、文档解析、网页解析、图像和视频解析，以及脚本解释或 cookie 的编码/解码。即便是简单的 file(1) 命令，也在 2014 年因其解析代码中的漏洞而被修补 [SA14:16]。

若进程被恶意内容入侵，沙箱策略的目标就是将潜在损害限制在一小部分已知输出上。例如，被入侵的文字处理器可能会破坏其输出文件，但不应能够在用户的主目录中搜索私钥或信用卡信息。专门用于沙箱化的特性（或有时称为攻击面缩减特性），如 FreeBSD 的 Capsicum、OpenBSD 的 pledge(2) 以及 Linux 的 seccomp(2)，是在新近问世的；我们将在下文比较它们的有效性。

## DAC/MAC 下的沙箱化

在商用操作系统引入沙箱化特性之前，人们曾竭力利用现有工具对应用程序进行限制或沙箱化。这些努力的成效因安全策略与自主访问控制（DAC）或强制访问控制（MAC）模型的契合程度而异。最成功的沙箱化应用只需对代码进行相对较小的修改即可达到安全目标，这通常符合 DAC 的粗粒度安全模型或 MAC 的系统安全视角。较不成功的尝试往往需要数千行代码，有时还需大量手工编写的汇编代码、系统（超级用户）权限，并且常常无法真正执行预期的安全策略。

早期且相对成功的沙箱化实现是 Provos 等人对 OpenSSH 服务器的特权分离 [PFH03]。该工作利用自主访问控制特性防止被入侵的 SSH 服务器进程行使超级用户权限。SSH 服务器需要超级用户权限才能绑定到 TCP 端口 22，但希望被入侵的 SSH 进程在用户认证之前不能访问系统资源（即服务器应置于认证前沙箱）；同时希望服务器在认证后只能行使授予认证用户的权限（在认证后沙箱内）。

Provos 等人将 SSH 服务器进程拆分为：

1. **受信任的监控进程**：保留超级用户权限。
2. **不受信任的子进程**：使用超级用户权限放弃特权，将其 UID/GID 改为非特权用户。

在认证前沙箱中，SSH 服务器进程可以 nobody 用户身份运行，并通过 chroot(2) 系统调用将根目录更改为一个空目录。在认证后沙箱中，进程的 UID/GID 被更改为认证用户的 UID/GID。

这种沙箱化方法成功的原因有两个：

1. **以用户为导向的策略**：SSH 特权分离的目标是防止被入侵的进程行使超级用户权限。该策略目标与 Unix 的 DAC 模型高度契合：可以完全通过 UID、GID 以及带有 Unix 权限的文件系统目录来表达。该策略并不保护用户在认证后免受应用程序不当行为的影响；它保护的是系统及其他用户。

2. **现有特权**：诸如更改进程 UID 或根目录的操作需要超级用户权限，而正在进行特权分离的 SSH 服务器已经具备这些权限。sshd 进程本身就是以 root 身份运行的安全关键软件：特权分离只是权力的单调下降。然而，对于更一般的沙箱化场景来说，这种方式并不适用：不希望非特权软件为了降低特权而必须以 root 身份运行。

在另一个极端，我们曾比较过几种基于 DAC 和 MAC 的方法，用于在 Chromium 浏览器中对渲染进程进行沙箱化 [WALK10]。研究发现，DAC 和 MAC 机制并不适合应用程序隔离的场景。DAC 的设计目的是保护用户之间的安全，但在 Web 浏览器或其他复杂多进程用户应用程序中，安全目标是限制被恶意内容入侵的进程所能造成的破坏。仅靠 DAC 无法控制对未标记对象的访问，例如 Linux 的 System V 共享内存或 Windows 的 FAT 文件系统。与 OpenSSH 类似，可以使用 chroot(2) 将进程置于有限文件系统访问环境中，但与 OpenSSH 不同的是，Web 浏览器（或办公套件、音乐播放器及其他桌面应用程序）通常并不具备使用 chroot(2) 所需的超级用户权限。因此，为了利用现有的 DAC 保护，应用程序的部分组件必须以 root 所有的二进制文件并设置 setuid 位（如今的 Chrome 在 Linux 上已不再使用基于 DAC 的沙箱，但上述关于 chroot(2) 和特权的讨论仍然适用于新的沙箱模式。

强制访问控制（MAC）也不适合应用程序沙箱化。它要求策略在两处编码：一处在描述应用程序行为的代码中，另一处在描述应用程序允许行为的独立策略中。我们最初对 Capsicum 的比较中使用的 SELinux 策略包含数千行策略，即使是使用领域特定语言编码的现代 AppArmor 配置文件，也可能需要数百行复杂细致的策略，更不包括从系统策略库中包含的策略元素（例如 `#include <abstractions/ubuntu-browsers.d/java>`）。编写和维护复杂的 MAC 策略非常困难，其功能性缺失（例如无法访问 `@{PROC}/[0-9]*/oom_score_adj`）对开发者来说比保护缺失（例如允许访问依赖 PATH 环境变量未被劫持的 /usr/bin/xdg-settings）更为明显。这种复杂性提示我们，其实是在试图把沙箱化这一“方枘”强行套进 MAC 这一“圆凿”。

## 使用系统调用拦截的“沙箱化”

<img width="473" height="341" alt="SD@N49ADX%(UZPK`5G$B%WX" src="https://github.com/user-attachments/assets/3ca60f6e-a5a1-425c-bebc-3f5781216787" />

**图 1 Generic Software Wrapper Toolkit 和 systrace 的系统调用封装器行为策略（分别转载自 [FBF00] 和 [Prov03]）。**

在比较当今的沙箱化方法之前，有必要了解一种在 1990 年代末和 2000 年代初尝试过的过渡方法。这种方法看似简单，但最终未能提供其宣传的安全优势。那些未能从这种现已被否定的方法中吸取教训的人，注定会在其“新”应用程序沙箱化方法中重蹈覆辙。

一种开创性的尝试是 Fraser、Badger 和 Feldman 提出的 **Generic Software Wrappers** [FBF00]，它启发了诸如 Provos 的 systrace [Prov03] 等系统。这些系统调用拦截（system-call interposition）系统通过用户空间的封装器或对操作系统内核系统调用层的浅层修改来拦截系统调用。一旦拦截，就可以检查这些调用的参数，并根据策略决定是否允许该调用。

例如，不使用 chroot(2) 来限制进程访问文件系统，而是可以检查每一次 open(2) 调用，并将文件名参数与进程允许访问的路径白名单进行比对。系统调用策略可以用一种语言描述，虽然像 MAC 一样需要双重编码，但具有简洁性和易理解性的优势，如图所示。

系统调用封装器的双重好处在于：实现相对简单，使用也相对容易。不幸的是，这种简单性导致它未能应对操作系统中并发访问的复杂性，正如 Watson 在 2007 年所证明的 [Wat07]。

由 Unix 系统调用命名的对象在多个层面上都是并发的。在最浅层面上，进程的所有线程都在同一个虚拟地址空间中，因此可以操作相同的数据——这包括作为系统调用参数传递的字符串。当系统调用封装器在用户空间工作时，恶意进程可以提交一个已知在白名单中的路径的系统调用执行，然后在封装器策略检查执行期间，将内存中包含的文件名修改为其他路径。因此，经过策略检查的路径可能与最终访问的路径不同。用 Watson 的术语来说，这就是 **检查时-使用时 (Time-of-check-to-time-of-use, TOCTTOU) 漏洞** [Wat07]。

然而，TOCTTOU 漏洞并不仅存在于这一浅层拦截中。如果只是浅层问题，通过 RPC 进行拦截即可保证安全。系统调用封装器在更深层次、更根本的方式上存在漏洞：即使用于访问操作系统对象（如文件）的名称保持不变，该名称的含义仍可能发生变化。在 Unix 中，路径查找是一个增量操作：查找名为 `/home/jon/foo.txt` 的文件，需要与虚拟文件系统中的至少四个 vnode 交互（根节点、两个目录和文件本身）。虽然每一次单独查找（如检索 `jon` 目录条目）必须妥善处理并发，例如在持锁期间执行，但整个路径查找并不是一个原子操作。

路径是指令列表，而不是简单的名称。当一个进程在遍历目录层级时，另一个进程可能正在修改文件系统：移动文件、移动目录，甚至修改符号链接。即使系统调用拦截通过 RPC 执行且无法进行内存路径替换，也无法保证在策略决策时指定的路径所对应的文件，就是系统调用实际查找的文件。

根本上，系统调用拦截的弱点在于其策略决策（即检查）与这些决策的实际效果不是原子执行的。这不是一个可以通过补丁修复的漏洞，而是该方法的根本性局限；这也是为什么这种方法在当代操作系统中已不再使用（OpenBSD 于去年四月移除了 systrace [Gros16]）。然而，即使系统调用拦截系统已被弃用，其核心概念仍会在现代沙箱化框架中以不同形式出现，带来安全挑战。

<img width="418" height="88" alt="~J9(RKK52)UI6NKSZB4 E4" src="https://github.com/user-attachments/assets/64e697a9-c690-496d-ba9b-a682db7e3e45" />

**图 2 进入 Linux 安全计算模式最初的“纯”版本非常简单。只要进程进入 seccomp 模式，就永远无法退出。**

## 沙箱化框架比较

近来，开源 Unix 派生系统实现了新的框架以辅助应用程序沙箱化。这些框架主要包括 Linux 的 seccomp(2)、OpenBSD 的 pledge(2) 和 FreeBSD 的 Capsicum（capsicum(4)）2（关于苹果 Sandbox 框架及其基于 MAC 的底层实现的讨论，可参考其他文献 [Wat13]）。尽管它们的创建目标都是实现简单的沙箱化，但实际效果各有差异。

### Linux: seccomp(2)

自 2005 年起，Linux 引入了名为“安全计算模式”（secure computing mode，简称 seccomp(2)）的特性 [Cor09]。seccomp(2) 的最初版本提供了一个强大且易于理解的安全策略：处于“安全计算模式”的进程可以使用 read(2) 和 write(2) 系统调用操作它们之前打开（或被委托）的文件，使用 sigreturn(2) 支持信号传递，并通过 exit(2) 系统调用终止进程。进程进入 seccomp 模式非常简单，如图 2（简化版）3 所示，本节示例的完整源代码可在以下网址获取：[https://github.com/trombonehero/sandbox-examples](https://github.com/trombonehero/sandbox-examples)。

该策略的优点在于清晰，并允许进程作为过滤器执行纯计算任务，但很少有应用程序能在如此严格的沙箱中执行有意义的工作。例如，我们之前发现，Chrome 使用“纯” seccomp(2) 模式时，需要超过一千行安全关键的汇编代码，将系统调用从沙箱进程转发到受信任进程，由后者代为执行 [WALK10]。

为了提供更丰富的计算环境，现代 seccomp(2) 能让程序在上述四个系统调用之外指定自己的安全策略。在新版 seccomp(2) 中，进程可以指定一个程序，在执行每个系统调用前检查其有效性。该程序以 BPF 字节码格式编写。BSD 数据包过滤器（BPF）[MV93]，受到 CMU/Stanford 数据包过滤器（CSPF）的启发，而 CSPF 又受到早期 Xerox Alto 工作 [MRA87] 的启发，是一个解释字节码的虚拟机。BPF 最初设计用于高性能网络处理，允许用户空间进程描述一个过滤器，由内核应用于网络数据包，同时保持内核/用户模式隔离的安全性。

<img width="518" height="309" alt="(RRCUWBJ0WU4~LG125319B" src="https://github.com/user-attachments/assets/95e4b835-e14f-46ba-9517-682096df5aef" />

**图 3 一个简单的 seccomp-bpf 过滤器示例，允许 brk(2)、close(2) 和 openat(2) 系统调用继续执行（基于 Bernstein 的示例 [Bern17]）。**

应用于 seccomp(2) 时，BPF 提供了一种语法，用于在 Linux 系统调用处理程序内描述检查系统调用的程序。图 3 展示了一个简单的系统调用白名单过滤器示例，说明了 seccomp-bpf 的极高灵活性和可编程性：几乎任何对系统调用参数的检查都可以用类似汇编的 BPF 语言表达。

然而，这也带来了问题：由于程序员可以检查任何内容，程序员必须检查一切。为了构建有意义的系统调用白名单，不仅需要将系统调用号在更大结构中的偏移暴露给用户模式程序，还必须检查当前架构以正确解释系统调用号（Linux 在不同架构上使用不同的系统调用号）。此外，由于语义完全由程序员决定，很容易构建不一致的系统调用策略：某些操作被拒绝，但等效操作仍可执行。例如，图 3 中的策略不允许无限制的 open(2) 调用，但允许 openat(2)，它可以实现与 open(2) 等效的行为。

seccomp-bpf 过滤器与其过滤的程序行为紧密相关，因此由应用程序作者负责。但构建 seccomp-bpf 系统调用过滤器需要对许多细节高度关注（BPF 操作码的汇编编程、Linux 内核系统调用处理结构的布局与语义），这些完全超出了大多数应用程序作者的经验和知识范畴。

除了简单的系统调用白名单外，seccomp-bpf 既更复杂，也更容易出现问题。可以在系统调用参数（如文件名）上构建 seccomp-bpf 过滤器，但与 GSWTK 和 systrace 一样，在系统调用处理层无法对路径进行有意义的检查。

例如，程序可能被允许访问 `/var/tmp/*`，但如果 `/var/tmp/foo` 是一个符号链接，并且在与 BPF 过滤器竞争的情况下被更新，那么实际上执行了什么策略？[openat 示例](https://github.com/trombonehero/sandbox-examples) 展示了一个受 seccomp-bpf 限制的进程如何突破其预期边界，例如在应用程序预期工作目录之外创建文件。

因此，单靠 seccomp-bpf 并不足以真正沙箱化任意应用程序代码。完整的应用程序沙箱还必须使用 Linux 的 clone(2) 系统调用，将进程隔离到新的命名空间中，包括：

* **IPC 命名空间**：切断对主机全局 System V IPC 命名空间的访问
* **网络命名空间**：隔离网络接口、路由、防火墙、/proc 和 /sys/class/net 等
* **挂载命名空间**：类似于 chroot(2)
* **PID 命名空间**：防止不当使用 kill(2)

创建这些命名空间需要 **CAP_SYS_ADMIN** 权限，在 Linux 上相当于超级用户权限。因此，在 Linux 上创建有效的应用程序沙箱需要以 root 身份运行程序或创建 setuid 二进制文件。

### OpenBSD: pledge(2)

自 2016 年 v5.9 发布以来，OpenBSD 引入了 pledge(2)，一种将进程置于“受限服务操作模式”的机制5。pledge(2) 的手册页并未将其描述为安全机制 [Pled17]，但其开发者的其他交流中明确指出了安全意图 [deRa15]。pledge(2) 的核心是对 seccomp(2) 概念的一种更简单易用的实现。

与需要定义 BPF 程序来过滤系统调用不同，pledge(2) 将系统调用按类别分组，例如 **stdio**（包括 read(2)、write(2)、dup(2) 和 clock_getres(2)）和 **rpath**（允许通过 chdir(2)、openat(2) 等进行只读文件系统操作）。可以使用空字符串进行 pledge，此时除了 `_exit(2)` 外不允许其他系统调用，但这可能导致进程在 `_start()` 中由 C 启动例程触发的 atexit(3) 代码调用 libc 的 mprotect(2) 时中止。

<img width="443" height="85" alt="UHFKHK{Y03U$@L @FAF6K12" src="https://github.com/user-attachments/assets/ee4067b4-8a71-4bcf-83db-72d264bb172a" />

**图 4 使用 pledge(2) 安装系统调用过滤策略比使用 seccomp-bpf 简单得多。**

pledge(2) 的使用比等效的 seccomp-bpf 功能简单得多。图 4 展示了 pledge(2) 的使用示例，为当前进程应用系统调用过滤器，涵盖的系统调用比图 3 中更多。然而，与 seccomp-bpf 类似，这种对系统调用的简单、表面过滤提供的是虚幻的安全保证。提供的系统调用类别可能能有效描述 OpenBSD 基本系统中简单应用的需求，但对于复杂应用，诸如 **wpath** 的类别几乎毫无意义。如果应用需要打开私有文件进行写操作，则必须对 **wpath** 进行 pledge，但 wpath 同时授权打开文件系统上所有具有正确 DAC 模式的文件进行写入。与 seccomp-bpf 不同，pledge(2) 简化了策略构建，但同样默认容易生成不一致或无意义的策略。

pledge(2) 系统调用还接受包含允许路径白名单的 paths 参数，但自 2016 年初以来，该功能在手册页中被标记为“不可用” [Pled17]。即便可用，这种浅层白名单功能也会遭遇与 systrace 和 seccomp-bpf 相同的 TOCTTOU 漏洞。然而，pledge(2) 最大的弱点在于，如果原始 pledge(2) 调用包含 exec 系统调用类别，被入侵进程可以禁用安全机制。尽管有人声称“能力永远无法恢复” [Pled17]，以及“在 OpenBSD 中，一旦缓解机制生效，就无法被禁用” [deRa15]，pledge(2) 并不具备 seccomp(2) 或 capsicum(4) 的单向特性。在这些系统中，进入受限状态的进程及其后续创建的子进程会一直保持该状态，但 OpenBSD 进程的 pledge(2) 受限状态在 exec(2) 时会被清除。

与 seccomp-bpf（以及之前的 GSWTK/systrace）类似，pledge(2) 的系统调用过滤不足以对比读-计算-写过滤器更复杂的应用实施有意义的安全策略。不同之处在于，虽然它是更易用的框架，但 pledge(2) 没有基于 clone(2) 的机制来实现更严格的安全策略。因此，与已被 OpenBSD 停用的 systrace 一样，pledge(2) 应被视为用于调试、防御不熟练攻击者的辅助功能，而非可用来构建严格安全策略的机制。

### FreeBSD: capsicum(4)

Capsicum 隔离框架在两个关键方面不同于 seccomp-bpf 和 pledge(2)：

1. **原则性和一致性的限制模型**：Capsicum 在应用程序隔离时，为进程限制提供了一个系统化、一致的模型。这通过 Capsicum 的 **能力模式（capability mode）** 实现。

2. **细粒度、单调的权限降低**：Capsicum 对通过被削弱的文件描述符（称为能力，capabilities）访问的特定操作系统对象执行细粒度、单向的权限降低。

## 能力模式

与 seccomp-bpf 和 pledge(2) 类似，capsicum(4) 支持将进程置于受限模式，在此模式下系统调用的行为与“普通”进程不同。关键区别在于限制的选择方式。Capsicum 并不只是表面上关注某些特定系统调用——许多系统调用职责重叠，且提供独立方式完成相同目标——而是关注一个基本原则：对全局命名空间的访问。

在 Capsicum 中，**cap_enter(2)** 系统调用使进程进入 **能力模式（capability mode）**，在该模式下，访问操作系统对象（文件、套接字、进程、共享内存等）必须通过能力（capabilities，见下文）进行，而不能使用环境权限（ambient authority）。环境权限描述进程按 Unix DAC 模型代表用户执行操作的常规权限，包括通过 PID 访问其他进程、通过路径或 NFS 文件句柄访问文件、通过协议地址访问套接字、通过 System V IPC 名称或 POSIX 共享内存路径访问共享内存等。

相比之下，进入能力模式的进程不得通过全局命名空间（路径、PID、协议地址等）访问任何新的资源。已打开的文件描述符（或通过 Unix 消息传递传入进程的描述符）表示的资源，通常受以下“能力”中描述的限制约束。新文件描述符也可以从已有描述符派生，例如通过 **accept(2)** 或 **openat(2)** 系统调用，但仅允许使用本地名称。对于 openat(2)，路径查找必须相对于已打开的目录描述符开始，而非 AT_FDCWD，并且路径解析只能向目录内部“向下”遍历，而不能通过 “..” 向上访问。FreeBSD 内核深层的 **namei()** 函数在执行路径查找时原子地强制执行这一限制。

Capsicum 能力模式执行的策略在内部是一致的，因为它基于一个基本原则，而不是浅表的系统调用语法。它可以强制执行与 seccomp-bpf 和 pledge(2) 的有限、内部一致用例相同的限制：如果进程在进入能力模式时仅持有可读/可写文件能力而没有其他资源，则除这些描述符描述的操作外，不可能对系统产生其他副作用。为了支持更复杂的行为，Capsicum 提供能力机制，使资源在一致的安全模型下能够被原则性地共享。

## 能力

计算机科学中“能力（capability）”的历史概念是将对象的标识符与可在其上执行的操作组合在一起。这一定义由 Dennis 和 Van Horn 在 1960 年代后期提出 [DV66]，在今天的 Unix 中仍有其影子。在 Dennis 和 Van Horn 的设想中，能力是一个索引，指向由主管（supervisor）代表进程维护的能力列表。该概念延续到 Multics，并最终演化为我们今天在 Unix 中所知的文件描述符 [RT78]。

类似于能力，文件描述符是指向主管维护的操作系统对象列表的索引；它们还与可以执行的操作相关联，这取决于打开时的标志（如 O_RDONLY）。然而，与能力不同，文件描述符携带了意外的、隐式的权限，这些权限无法单调地降低。例如，应用程序不能对一个以 O_RDWR 标志打开的描述符执行 open(2) 并 dup(2) 生成新描述符，再将其改为只读并与不受信任的工作进程共享。即便文件描述符以只读方式打开，Unix DAC 模型仍可能允许诸如 fchmod(2) 等系统调用意外地修改文件元数据。

Capsicum 对能力的实现提供了对特定对象的细粒度权限（“权力”）的单调降低。它通过将权限（如 CAP_READ、CAP_FSTAT、CAP_MMAP、CAP_FCHMOD 等）附加到描述符上实现这一点。这些行为类别对应于内核对象的方法，并关联到需要它们的一组系统调用。例如，要相对于某个目录使用 openat(2) 打开可读写文件，该目录描述符必须至少启用 CAP_READ、CAP_WRITE 和 CAP_LOOKUP。

在能力模式之外，未沙箱化的进程使用环境权限（ambient authority）通过系统调用（如 open(2)）返回的文件描述符会隐式获得全部权限。这既保持了与传统 Unix 语义的兼容性，又可在能力模式内外统一执行能力权限。

描述符上的权限可以通过 **cap_rights_limit(2)** 进行削减，描述符可以被继承或传递给沙箱化进程，从现有描述符派生的新描述符（如通过 accept(2) 或 openat(2)）将继承其父对象的权限。这就允许安全地进行权限委托。

## 通过能力实现沙箱化

<img width="531" height="316" alt="T6P2QLYN%}NU_@XAFBH6} 0" src="https://github.com/user-attachments/assets/5b0633a3-b7bd-488c-b96e-5474ae44e84f" />

**图 5**

为 bhyve 虚拟机监控器添加 Capsicum 支持只需进行最少的代码更改。
- `caph_cache_catpages()` 预先打开一个目录，
- `caph_limit_std{out,err}()` 限制 stdout 和 stderr 上的权限，
- `cap_enter()` 进入能力模式。


Capsicum 允许应用程序作者以最低限度的工作量对其应用程序实施严格的安全策略。在今天，即便是中等复杂度的应用程序，如虚拟机监控器和 Web 浏览器，也可以通过打开资源（包括承载资源的资源，如目录和服务器套接字）、限制这些资源关联的权限，然后进入能力模式，从而支持丰富的使用场景。以这种方式对 bhyve 虚拟机监控器进行沙箱化所需的工作如图 5 所示。当前仍在努力使 Capsicum 模型适用于更广泛的应用类别，包括需要访问外部资源（如 powerboxes [Yee04]）的应用，即便它们对沙箱化特性毫不知情 [AGW17]。

基于严格的基础，Capsicum 是一个能够支持复杂行为的平台。由于其安全策略既简单又一致，应用程序作者可以在此基础上构建支持服务，而无需掌握内核内部细节，也无需担心构建出不一致的安全策略。因此，我们将 Capsicum 视为一个生成性平台，使应用程序作者能够专注于自身擅长的工作，利用严格的安全工具，而无需极高的安全专业知识。我们希望，通过为作者提供安全软件构建工具，将来的应用程序能够更好地保护用户，不仅防止用户之间的风险，也能防止用户自身应用带来的风险。

## 参考文献

[And72] Anderson, J. P. “Computer Security Technology Planning Study”，ESD-TR-73-51，电子系统部，美国空军，1972，网址：[https://csrc.nist.gov/publications/history/ande72.pdf](https://csrc.nist.gov/publications/history/ande72.pdf)

[AGW17] Anderson, J.; Godfrey, S.; Watson, R. N. M. “Toward Oblivious Sandboxing with Capsicum”，FreeBSD Journal，2017 年 7/8 月，网址：[https://www.freebsdfoundation.org/past-issues/security](https://www.freebsdfoundation.org/past-issues/security)

[Bern17] Bernstein, O. “Denying Syscalls with Seccomp”，Eigenstate，2017 年 8 月检索，网址：[https://eigenstate.org/notes/seccomp.html](https://eigenstate.org/notes/seccomp.html)

[BCD69] Bensoussan, A.; Clingen, C.; Daley, R. “The multics virtual memory”，发表于 SOSP '69: 第二届操作系统原理研讨会论文集，1969，第 30–42 页，DOI: 10.1145/961053.961069

[Cor09] Corbet, J. “Seccomp and sandboxing”，LWN.net，2009，网址：[http://lwn.net/Articles/332974](http://lwn.net/Articles/332974)

[Cor12] Corbet, J. “Yet another new approach to seccomp”，LWN.net，2012，网址：[http://lwn.net/Articles/475043](http://lwn.net/Articles/475043)

[deRa15] de Raadt, T. “pledge(): a new mitigation mechanism”，2015，2017 年 9 月访问，网址：[http://www.openbsd.org/papers/hackfest2015-pledge](http://www.openbsd.org/papers/hackfest2015-pledge)

[DV66] Dennis, J.; Van Horn, E. “Programming semantics for multiprogrammed computations”，Communications of the ACM 9(3)，1966，第 143–155 页，DOI: 10.1145/365230.365252

[FBF00] Fraser, T.; Badger, L.; Feldman, M. “Hardening COTS software with generic software wrappers”，发表于 2000 DARPA 信息生存能力会议论文集 (DISCEX)，2000，DOI: 10.1109/DISCEX.2000.821530

[Gros16] Grosse, J. “systrace(1) is removed for OpenBSD 6.0”，2016，网址：[http://daemonforums.org/showthread.php?t=9795](http://daemonforums.org/showthread.php?t=9795)

[Lamp69] Lampson, B. “Dynamic protection structures”，发表于 AFIPS '69 (Fall)：AFIPS 1969 年秋季联合计算机会议论文集，1969，DOI: 10.1145/1478559.1478563

[MRA87] Mogul, J. C.; Rashid, R. F.; Accetta, M. “The Packet Filter: An Efficient Mechanism for User-level Network Code”，发表于第 11 届操作系统原理研讨会论文集 (SOSP)，1987，第 39–51 页，网址：[https://dl.acm.org/ft_gateway.cfm?id=37505](https://dl.acm.org/ft_gateway.cfm?id=37505)

[MV93] McCanne, S.; Jacobson, V. “The BSD Packet Filter: A New Architecture for User-level Packet Capture”，发表于 USENIX Winter 1993 Conference，1993，网址：[https://www.usenix.org/legacy/publications/library/proceedings/sd93/mccanne.pdf](https://www.usenix.org/legacy/publications/library/proceedings/sd93/mccanne.pdf)


[Pled17] “pledge—restrict system operations”，载于 OpenBSD 系统调用手册，2016–17，2017 年 9 月检索，网址：[https://man.openbsd.org/pledge.2](https://man.openbsd.org/pledge.2)

[Pos1e] IEEE 计算机协会可移植应用标准委员会，“信息技术可移植操作系统接口 (POSIX) 第 1 部分：系统应用程序接口 (API) 草案标准—修订 #: 保护、审计与控制接口 [C 语言]”，IEEE 草案标准（已撤回），1997，网址：[http://wt.tuxomania.net/publications/posix.1e/download.html](http://wt.tuxomania.net/publications/posix.1e/download.html)

[Prov03] Provos, N. “Improving Host Security with System Call Policies”，发表于第 12 届 USENIX 安全研讨会论文集，2003，网址：[https://dl.acm.org/citation.cfm?id=1251371](https://dl.acm.org/citation.cfm?id=1251371)

[PFH03] Provos, N.; Friedl, M.; Honeyman, P. “Preventing Privilege Escalation”，发表于第 12 届 USENIX 安全研讨会论文集，2003，第 231–242 页，网址：[https://www.usenix.org/legacy/events/sec03/tech/provos_et_al.html](https://www.usenix.org/legacy/events/sec03/tech/provos_et_al.html)

[RT78] Ritchie, D.; Thompson, K. “The UNIX time-sharing System”，发表于 Bell System Technical Journal 57(6)，1978，第 1905–1929 页，DOI: 10.1002/j.1538-7305.1978.tb02136.x

[SA14:16] FreeBSD. “Multiple vulnerabilities in file(1) and libmagic(3)”，2016，网址：[https://www.freebsd.org/security/advisories/FreeBSD-SA-14:16.file.asc](https://www.freebsd.org/security/advisories/FreeBSD-SA-14:16.file.asc)

[SS72] Schroeder, M. D.; Saltzer, J. H. “A Hardware Architecture for Implementing Protection Rings”，发表于 Communications of the ACM 15(3)，1972，第 157–170 页，DOI: 10.1145/361268.361275

[Wat07] Watson, R. N. M. “Exploiting concurrency vulnerabilities in system call wrappers”，发表于 2007 年 USENIX Offensive Technologies Workshop (WOOT) 论文集，2007，网址：[http://static.usenix.org/event/woot07/tech/full_papers/watson/watson.pdf](http://static.usenix.org/event/woot07/tech/full_papers/watson/watson.pdf)

[Wat13] Watson, R. N. M. “A Decade of OS Access-Control Extensibility”，发表于 Communications of the ACM 56(2)，2013，第 52–63 页，DOI: 10.1145/2408776.2408792


[WALK10] Watson, R. N. M.; Anderson, J.; Laurie, B.; Kennaway, K. “Capsicum: practical capabilities for UNIX”，发表于第 19 届 USENIX 安全研讨会论文集，2010，网址：[https://www.usenix.org/legacy/events/sec10/tech/full_papers/Watson.pdf](https://www.usenix.org/legacy/events/sec10/tech/full_papers/Watson.pdf)

[WCM+02] Wright, C.; Cowan, C.; Morris, J. 等. “Linux Security Modules: General Security Support for the Linux Kernel”，发表于第 11 届 USENIX 安全研讨会论文集，2002，网址：[https://www.usenix.org/legacy/event/sec02/full_papers/wright/wright.pdf](https://www.usenix.org/legacy/event/sec02/full_papers/wright/wright.pdf)

[WFMV03] Watson, R.; Feldman, B.; Migus, A.; Vance, C. “Design and implementation of the Trusted BSD MAC framework”，发表于 2003 年 DARPA Information Survivability Conference and Exposition (DISCEX '03)，2003，DOI: 10.1109/DISCEX.2003.1194871

[Yee04] Yee, K. “Aligning security and usability”，发表于 IEEE Security and Privacy 2(5)，2004，DOI: 10.1109/MSP.2004.64

---

**Jonathan Anderson** 是纽芬兰纪念大学电气与计算机工程系的助理教授，他的研究涉及操作系统、安全性以及编译器等软件工具的交叉领域。他是 FreeBSD 的提交者，并始终在寻找有相似兴趣的新研究生。
