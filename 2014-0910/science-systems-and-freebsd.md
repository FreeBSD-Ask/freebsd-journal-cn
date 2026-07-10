# 科学、系统与 FreeBSD

- 原文标题：Science, Systems, and FreeBSD
- 作者：**George Neville-Neil**

FreeBSD 操作系统源于学术研究，源自加州大学伯克利分校计算机系统研究组（CSRG）在 20 世纪 70 年代中期至 90 年代期间开发的 4.4-Lite 版本的伯克利软件发行版（Berkeley Software Distribution）。CSRG 在操作系统研究领域耕耘二十年 [11] 后，于 1995 年解散。在这二十年的活跃开发期间，CSRG 在虚拟内存 [1]、文件系统 [12] 方面产出了研究成果，开发了 Sockets API 和最受欢迎的 TCP/IP 网络协议栈，以及伯克利数据包过滤器 [10]。

尽管 FreeBSD 如今在工业界广泛使用，它在学术界仍享有良好声誉，被视为开展研究的稳定基础。FreeBSD 之所以成为出色的研究平台，原因众多，包括优秀的工具链、一致的开发与发布流程、友好的开源许可证，以及对测量的坚定承诺。

FreeBSD 团队重视工具链建设，包括 DTrace 这类系统，以及 LLVM 等现代编译器和 LLDB 调试器。FreeBSD 遵循一致的开发与发布流程，让研究者确信：若系统为某个特定的小版本（如 10.1）构建，那么在后续所有更新（10.2、10.3 等）中都能不加修改地正常运行。发布分支内部的这种一致性，减少了研究者在更新实验和系统时需要担心的变量数量。

BSD 许可证允许研究者在自己的工作中使用和改造操作系统的部分内容，并将其整合到研究或工业产品中，而不必担心被迫公开自己的工作成果。该许可证鼓励共享，但并不强制其用户共享。FreeBSD 用于众多商业产品这一事实，为研究者提供了大量真实场景来检验他们的想法——无论是改进 CDN 使用的 TCP 软件，还是开发新型文件系统以存储泽字节量级的数据。

最后，FreeBSD 项目承诺以测量为本，始终用数据支撑关于系统的任何论断。对测量的重视是一种文化价值，由最初的 CSRG 团队传承下来，并延续到系统的最新版本。当前 FreeBSD 上的研究工作继续在多个领域扩展系统，包括文件系统、网络和安全。

## 文件系统

快速文件系统（FFS）是计算机科学中寿命最长的文件系统开发项目之一，始于 20 世纪 80 年代初期并延续至今。FFS 有着被其他存储研究项目超越、随后又反超它们的历史。20 世纪 90 年代初期，关于日志结构文件系统（LFS）的研究——这类文件系统将数据存储在日志中，而非作为磁盘上随机访问的块集合——成为多篇研究论文的主题 [17]。人们对 FFS 与日志结构文件系统进行了性能比较，结果显示日志结构系统相比 FFS 这类传统文件系统具有显著的性能优势。在 LFS 工作之后，研究表明 FFS 处理元数据的方式——即把文件相关信息存储在磁盘上，而非文件中的数据——是两种系统性能差异的主要来源。对软更新的研究，即安全地延迟元数据写入磁盘的能力，提升了 FFS 的整体性能，使其达到或超过了基于日志的文件系统 [6]。

1999 年，FFS 新增了文件系统数据快照功能，可用作快速周期性备份，或让普通用户恢复误删的文件。

随着磁盘容量增大，超过 1 TB 后，系统崩溃后恢复磁盘状态所需的时间从几分钟增长到 1 小时以上，大型存储阵列甚至需要数小时。著名的文件系统检查程序 `fsck` 在检查磁盘状态一致性时，阻塞了系统启动过程。为消除这一系统恢复瓶颈，需要给软更新系统加入小型日志 [13]。任何待处理的更新都会保存在日志中，系统重启时只需读取日志——它很少超过 1 MB——即可恢复一致的磁盘状态。

FFS 的工作延续至今，可变大小磁盘块和增强的文件元数据等改进，将作为计划中的 UFS3 一部分推出。

FreeBSD 从 OpenSolaris 引入了 ZFS，有望为超大数据集开启新的研究领域。虽然尚无关于 FreeBSD 上 ZFS 的研究论文发表，但在 FreeBSD 上存储极其庞大、相互关联的数据集，对数据科学研究者而言至关重要。随着 ZFS 在 FreeBSD 上稳固确立其作为海量数据集——那些跨越数百 TB 及以上的数据集——首选文件系统的地位，必将催生新一轮关于高效数据检索方法的研究。存储大量数据并不难，例如通过记录物理现象的实时测量，但研究重点将转向在这些大型数据集中查找相关数据并加以利用。

## 网络

FreeBSD 的网络子系统一直为研究提供平台。最初的 Sockets API 和标准的 TCP/IP 协议栈都是原始 BSD 系统的组成部分。最早的 TCP 实验平台之一是 Dummynet，由比萨大学的 Luigi Rizzo [15] 于 1997 年引入 FreeBSD 2.2.8。Dummynet 的目的是让研究新型 TCP 拥塞控制算法的研究者，能够在实验室环境中引入网络延迟，从而在早期测试时无需将代码实际部署到开放互联网上。Dummynet 通过内部排队数据包，并仅在预设时间释放它们来引入网络延迟。

TCP 研究的主要中心之一位于澳大利亚墨尔本的斯威本大学。斯威本的研究者研究各种 TCP 算法的效率，开发了一套系统，让他们能在运行时精细观察 TCP 状态机及其调优变量的内部工作。由 Lawrence Stewart 和 James Healy 开发的 SIFTR 系统，生成 TCP 状态机变化的运行时日志，帮助评估新的 TCP 算法 [18]。

随着网卡速度提升，先后突破 10 Gbps 并达到 40 Gbps，操作系统内核本身被视为网络应用的瓶颈。netmap 系统提供了一种安全方式，可绕过内核网络协议栈，让应用直接访问底层网络硬件。直接访问底层硬件后，应用能够在商用硬件上以线速收发网络数据 [16]。

对网络数据包访问的改进催生了后续研究，重写了知名的 Web 服务器和域名（DNS）服务器等应用。剑桥大学博士生 Ilias Marinos 开发了 Sandstorm 和 Namestorm 系统，借助 netmap 研究成果，在通用硬件上以 10 Gbps 速率提供静态 Web 内容和 DNS 查询 [8]。通过使用聚焦狭窄、专门构建的软件，以性能取代通用性，Sandstorm 和 Namestorm 服务器能够充分发挥现代 10 Gbps 和 40 Gbps 硬件的能力，同时占用的 CPU 资源比通过 sockets 接口工作的内核 TCP/IP 协议栈更少。

## 安全

Unix 的历史横跨大型分时系统时代。虽然计算世界已从多人共享单台计算机，转变为一个人拥有多台互不共享的计算机，但在保护用户互不干扰方面获得的经验教训具有奠基意义，使安全始终位居操作系统必备特性清单之首。FreeBSD 上的系统安全研究仍在继续，许多前沿思想和首批实现都出现在 FreeBSD 版本中。

最初随 FreeBSD 5.0 发布的强制访问控制（MAC）框架，可在操作系统内核中实现细粒度安全策略。

操作系统内核的传统安全模型通常包含两个安全级别：内核态和其他一切。一旦用户或其代码进入内核，就能访问系统中的任何内容。这种粗粒度的安全模型虽然简单易实现，但限制了可构建的安全系统类型。MAC 框架将更细粒度的访问控制模型引入内核，使得构建具有更窄、明确定义安全策略的系统成为可能。MAC 框架从 FreeBSD 的实验特性，发展到集成进 Mac OS、iOS 和 JunOS（Juniper 路由器的操作系统），至今仍是安全研究的基础 [20, 19, 9]。

建立在 MAC 框架之上的研究系统之一，是哈佛大学开发的 Shill 安全脚本语言 [14]。哈佛团队意识到，计算机安全最大的问题之一是搞清楚系统部署后如何维护和管理。系统管理软件通常以“root”身份运行，即拥有允许用户对系统进行任何操作而不受约束的全能特权。多数系统管理代码用 shell 脚本语言或 Perl 编写。他们开发的 Shill 编程语言借助 MAC 框架，让类似 shell 的脚本能够像在内核中那样表达安全策略。使用 Shill，脚本程序员可以允许访问某些信息，而不必冒出错程序意外覆盖或破坏数据的风险。

早期对计算机系统安全的研究引发了对“能力”（capabilities）的研究——这是一种不可伪造的信息片段，应用可用它证明用户或用户程序所言属实 [24]。在最初的研究时期，能力被认为是过于昂贵的系统安全机制。2009 年，剑桥大学的 Robert Watson 重新审视这一想法，与 FreeBSD 一同开发了 Capsicum 系统。Capsicum 是 FreeBSD 内核中的能力实现，相比以往方法引入的开销极小。运行在支持 Capsicum 的系统上的程序可被沙箱化，与其他应用隔离；开发者也可使用 Capsicum API 在程序内部创建沙箱，保护特别敏感的数据免遭篡改 [21]。Capsicum 已集成进 FreeBSD 10.0。

为更深入探索能力的潜力，一个更大的研究团队开始扩展现有的硬件指令集架构——基于 MIPS ISA——将能力下沉到接近零开销的层次。由此产生的 CHERI 和 BERI（<http://www.bericpu.org>）开源硬件平台，如今被研究者用于研究通过修改硬件/软件接口能做些什么来增强系统 [5, 22, 23]。BERI 是用 BlueSpec 语言编写的 MIPS 指令集架构开源实现，让研究者能够在硬件/软件接口层面工作，而无需从零开始构建硅片。CHERI 在 BERI 工作的基础上构建，新增对基于硬件的能力系统的支持，通过在基础 MIPS ISA 中加入专用指令来实现。这些实验系统在 FPGA 中实现，便于迭代研究，若投入专用 CPU，将能以与其他原生指令相同的速度提供能力及其安全原语。

FreeBSD 是首个全面采用 LLVM 编译器工具链的开源项目。LLVM 最初在伊利诺伊大学厄巴纳-香槟分校开发，被 Mac OS X 采纳为编译器套件，并成为 FreeBSD 10 的默认编译器套件 [7]。采用 LLVM 的驱动因素有多个，但主要因素在于 LLVM 实际上是一个用于开发编译器的工具包，因此在应用于各种问题时更加灵活。GNU 编译器链设计上不鼓励扩展，而 LLVM 的设计使得针对新指令集或应用进行适配能快速、轻松地完成。这种可重定向到新应用的能力，使 LLVM 被用于多个研究项目。伊利诺伊大学厄巴纳-香槟分校的一组安全研究者修改了 LLVM，研究保护系统免受面向返回的程序攻击。面向返回的编程是一种安全漏洞利用方式，通过栈溢出等攻击手段夺取程序控制权，攻击者的代码在程序执行时操纵其栈。厄巴纳-香槟分校团队扩展了 LLVM，并使用 FreeBSD 证明：即使运行应用的操作系统被攻破，应用仍可受到保护免遭恶意代码侵害 [4]。使用另一套技术但同样借助这些研究工具，同一研究团队证明了他们能阻止面向返回的程序攻击操作系统本身 [3]。

## 培养下一代

研究平台并非静态系统，需要不断成长和变化以保持相关性。为汇聚最聪明的人才与思想，必须用该平台培养研究者，然后让他们根据自身需要和兴趣改造平台。

FreeBSD 适合研究的一个特征，是各种工具为系统提供的可见性，包括用于测量系统性能的硬件性能计数器（`hwpmc`），以及可在运行时动态插桩用户态和内核态代码的 DTrace [2]。

借助这些工具，一组研究者、开发者和教育者正在准备一门操作系统硕士课程，将于 2015 年 1 月起在剑桥大学开设，随后与学术界感兴趣的教育者分享。该课程的最终目标是成为涵盖操作系统设计与实现所有领域的一系列课程的基础。使用追踪和分析工具教授操作系统课程，是对传统教学的激进背离——传统教学侧重讨论理论结果，即使尝试实现，也只是为玩具操作系统添加无关紧要但易于评分的扩展 [25]。这门新课程将以完整的生产版 FreeBSD 作为基础源代码，并使用平台上的可用工具，让学生完整观察操作系统的内部运作。该课程围绕小型、廉价的嵌入式系统构建，普通大学生和计算机系都能轻松负担，每位学生都能在真实硬件平台上操作生产级操作系统。

课程完成后，教学材料将以开源许可证通过 <http://www.teachbsd.org> 网站提供给其他学校和教育者。

## 结论

过去 20 年间，FreeBSD 操作系统一直是工业界和学术界研究的重要贡献者，在将前沿研究部署到商业系统方面有着出色的记录。在文件系统、网络和安全等截然不同的领域都做出了重要的研究贡献。前沿研究继续首先在 FreeBSD 上问世。研究者选择 FreeBSD 的原因与其所从事的工作一样多种多样，但有几个共同主题贯穿其中，包括优秀的工具链、一致的开发方法论、宽松的开源许可证，以及对测量的执着。随着项目持续成长和演进，我们可以预期会看到更多新颖的前沿研究以 FreeBSD 为平台开展并发表。

## 参考文献

[1] Babaoglu, Özalp and Joy, William. “Converting a Swap-based System to Do Paging in an Architecture Lacking Page-referenced Bits”; SOSP ’81: Proceedings of the Eighth ACM Symposium on Operating Systems Principles. (December 1981)

[2] Cantrill, Bryan; Shapiro, Michael; and Leventhal, Adam. “Dynamic Instrumentation of Production Systems”; USENIX Annual Technical Conference. (June 2007)

[3] Criswell, John; Dautenhahn, Nathan; and Adve, Vikram. “KCoFI: Complete Control-Flow Integrity for Commodity Operating System Kernels”; SP ’14: Proceedings of the 2014 IEEE Symposium on Security and Privacy. IEEE Computer Society. (May 2014)

[4] Criswell, John; Dautenhahn, Nathan; and Adve, Vikram. “Virtual Ghost”; ASPLOS 2014. ACM Press. (2014)

[5] Davis, B; Norton, R.; Woodruff, J.; and Watson, R. N. M. “How FreeBSD Boots: A Soft-core MIPS Perspective.” <www.cl.cam.ac.uk>.

[6] Ganger, Gregory R.; McKusick, Marshall Kirk; Soules, Craig A. N.; and Patt, Yale N. “Soft Updates: A Solution to the Metadata Update Problem in File Systems”; Transactions on Computer Systems. (May 2000)

[7] Lattner, Chris and Adve, Vikram. “LLVM: A Compilation Framework for Lifelong Program Analysis and Transformation”; CGO ’04: Proceedings of the International Symposium on Code Generation and Optimization: Feedback-directed and Runtime Optimization. IEEE Computer Society. (March 2004)

[8] Marinos, Ilias; Watson, Robert N. M.; and Handley, Mark. “Network Stack Specialization for Performance”; Proceedings of the 2014 ACM Conference on SIG-COMM. SIGCOMM ’14. (2014)

[9] Mayer, Frank. “Security Enhanced Linux Symposium-SELinux 2007”; Security Enhanced Linux Symposium-SELinux 2007. (March 2007)

[10] McCanne, Steven and Jacobson, Van. “The BSD Packet Filter: A New Architecture for User-level Packet Capture”; USENIX’93: Proceedings of the USENIX Winter 1993 Conference Proceedings. USENIX Association. (January 1993)

[11] McKusick, Marshall Kirk. Open Sources: Voices from the Open Source Revolution. O’Reilly. (1999)

[12] McKusick, M. K.; Joy, W. N.; and Leffler, S. J. “A Fast File System for UNIX”; ACM Transactions on Computing Systems. (1984)

[13] McKusick, M. K. and Roberson, J. “Journaled Soft-Updates”; Proceedings of EuroBSDCon. (2010)

[14] Moore, S. “Shill: A Secure Shell Scripting Language”; 11th USENIX Symposium on Operating Systems Design and Implementation. USENIX Association. (2014)

[15] Rizzo, Luigi. “Dummynet: A Simple Approach to the Evaluation of Network Protocols”; SIGCOMM Computer Communication Review. (January 1997)

[16] Rizzo, Luigi. “Netmap: A Novel Framework for Fast Packet i/o”; 2012 USENIX Annual Technical Conference. USENIX Association. (2012)

[17] Rosenblum, Mendel and Ousterhout, John K. “The Design and Implementation of a Log-structured File System”; ACM Transactions on Computer Systems. (February 1992)

[18] Stewart, L. and Healy, J. “Characterizing the Behavior and Performance of SIFTR v1.1.0.”; <http://caia.swin.edu.au/reports/070824A/CAIA-TR-070824A.pdf> (2007)

[19] Watson, R.; Feldman, B.; Migus, A.; and Vance, C. Design and Implementation of the Trusted BSD MAC framework, Volume 1. IEEE. (2003)

[20] Watson, Robert N. M. “A Decade of OS Access-control Extensibility”; Communications of the ACM. (February 2013)

[21] Watson, Robert N. M.; Anderson, Jonathan; Laurie, Ben; and Kennaway, Kris. “Capsicum: Practical Capabilities for UNIX”; USENIX Security’10: Proceedings of the 19th USENIX Conference on Security. USENIX Association. (August 2010)

[22] Woodruff, J.; Watson, R. N. M.; Chisnall, D.; Moore, S. W.; Anderson, J.; Davis, B.; Laurie, B.; Neumann, P. G.; Norton, R.; and Roe, M. “The CHERI Capability Model: Revisiting RISC in an Age of Risk”; Computer Architecture (ISCA), 2014 ACM/IEEE 41st International Symposium. (2014)

[23] Woodruff, Jonathan; Moore, Simon W.; and Watson, Robert N. M. “Memory Segmentation to Support Secure Applications”; CEUR Workshop: Doctoral Symposium on Engineering Secure Software and Systems (ESSoS). (January 2013)

[24] Dennis, J.B. and VanHorn, E.C. “Programming Semantics for Multi-programmed Computations,” Commun. ACM, Vol. 9, No. 3, (1966). Also <http://www.cs.virginia.edu/~evans/cs551/saltzer/>

[25] <https://www.usenix.org/conference/usenix-winter-1993-conference/presentation/nachos-instructional-operating-system>. A reference to its child is: <http://en.wikipedia.org/wiki/Pintos>
