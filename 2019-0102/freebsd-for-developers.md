# 面向开发者的 FreeBSD

- 原文链接：[FreeBSD for Developers](https://freebsdfoundation.org/our-work/journal/browser-based-edition/)
- 作者：**MARIUSZ ZABORSKI**

提到 FreeBSD 时，你大概会想到服务器和 NAS。毫无疑问，FreeBSD 是这些用途的优秀操作系统。但坊间也存在一些误解，认为 FreeBSD 对开发者不友好，如果想在 FreeBSD 上或为 FreeBSD 做软件开发，就该去买一台 Mac。事实是，FreeBSD 是一款完整的操作系统，从系统管理员、软件开发者到会计，每个人都能用它完成日常工作。本文从软件开发者的视角审视 FreeBSD。

## 编程语言

对软件开发者来说，最重要的问题是所选语言是否被 FreeBSD 支持。所有主流语言 FreeBSD 都支持，确实没有任何限制。你想用 Node.js、PHP 或 Python 做新的 Web 项目？FreeBSD 全都支持。如果你更关注 Go 或 Rust 这类底层编程，FreeBSD 也有。想要更经典一些？在 Ports 中可以找到新版本的 clang 与 gcc。甚至可以通过 OpenJDK 使用 Java，或通过 Mono 使用 C#。

值得称道的是，在许多情况下，FreeBSD 原生包管理器 **pkg(1)** 可以直接安装所需编译器。对于 PHP、Ruby 或 Python，许多包也由 pkg 提供。若要为 Python 安装 pygame，只需输入：

```sh
# pkg install py{27,36}-game
```

这样你就能用同一套方式管理操作系统中已安装的全部包，简单且整洁。出于某种原因不喜欢这种方式，仍然可以像以往那样使用 pip。WordPress 也是如此，执行下面的命令即可就绪：

```sh
# pkg install wordpress
```

执行以下命令可查找包含语言编译器与解释器的完整软件包列表：

```sh
# pkg search lang/
```

另一个优点是所有包都保持最新。repology 项目（<https://repology.org/repositories/statistics/newest>）对大量包仓库与其它来源进行分析，比较各仓库中的包版本。结论是 FreeBSD 拥有最新程度数一数二的 Ports。例如，其包的更新程度远超 Ubuntu 最新版本。如果你需要保持最新的语言基础设施，FreeBSD 是个好选择。

## 开发者环境

软件开发者关心的另一个问题是自己最爱的 IDE 能否在 FreeBSD 上工作。提到开发者环境，你大概会把 Unix 与 vim 和 emacs 联系起来。这没错，在 FreeBSD 上能用它们，不过你也能找到更现代的 IDE，例如：

- Eclipse
- Sublime Text
- PyCharm（免费版与商业版）
- IntelliJ（旗舰版与社区版）

如果你的 IDE 不直接支持 FreeBSD，可以尝试用 Linuxulator——FreeBSD 的 Linux 兼容层来运行。代码版本控制系统也不必担心。所有流行的开源 VCS 都能找到，例如 git、mercurial 或 svn，还可以使用 perforce 等商业产品。

维护环境、测试软件，没有比 Jail 更好的方式。如果你想为不同客户准备多个版本的应用，或同时运行多个数据库等，Jail 正是所需。还可以把它们与 ZFS 结合，轻松把改动推送至生产环境。如今用 iocage 或 ezjail 管理 Jail 比以往任何时候都更轻松。

有时 Jail 还不够，需要使用不同的操作系统。FreeBSD 也涵盖了这一需求。如果你需要轻量级虚拟化任何现代操作系统，例如 Windows、Linux、OpenBSD、NetBSD 或 FreeBSD，bhyve 是不二之选。

## 收服目录混乱

在 Linux 上把软件安装到哪里：**/usr/bin**、**/bin**、**/usr/local/bin**？或者 bin 只是指向 **/usr/bin** 的符号链接，一切都乱作一团？在 FreeBSD 中，管理类二进制总是落在 sbin，普通二进制落在 bin。目录层级清晰易懂：

- **/bin** 与 **/sbin** 是随操作系统分发的 FreeBSD 专用二进制，是系统最小化工作所必需的；
- **/usr/bin** 与 **/usr/sbin** 是随操作系统分发的 FreeBSD 专用二进制；
- **/usr/local/bin** 与 **/usr/local/sbin** 是所有第三方软件落地之处。

此外，整个文件系统层级结构可以在手册页 **hier(7)** 中查阅。FreeBSD 又一次凭借简洁清晰的方案胜出。

## 深入你的程序

DTrace 是开发者最棒的工具之一，而其他操作系统往往欠缺它。篇幅所限无法尽述 DTrace 的所有功能，但自从开始使用 DTrace，我已不知道此前是怎么熬过来的。DTrace 是动态追踪框架，最初由 Sun Microsystems 开发。用几行 D 脚本，我们就能查看程序的当前栈、分析性能、追踪函数输入，等等，不一而足。

DTrace 的真正威力来自 Brendan Gregg 编写的若干脚本（<https://github.com/brendangregg/FlameGraph>），可以生成火焰图。火焰图是对被追踪软件的可视化，便于识别最频繁的代码路径。可以用来追踪许多对象，从你的软件、FreeBSD 内核，到第三方软件。所需的只是一个带符号信息的二进制文件。使用时不必停止程序，也无需重新编译。我们最近对 PostgreSQL 数据库使用 DTrace，追踪为何插入操作耗时如此之长。借助 DTrace，我们轻松定位到表设置的问题并修复。这是 FreeBSD 相对其他操作系统对软件开发者的一项巨大优势，DTrace 应当常备你的工具箱。

开发底层应用时还有一些 FreeBSD 专属的实用工具：

- `procstat`——功能强大的工具，可获取大量进程信息
- `ktrace`/`kdump`——**strace(1)** 的替代方案，可打印系统调用列表
- `pmccontrol`——控制硬件性能监测计数器
- `pmcstat`——通过硬件性能监测计数器进行性能测量

至于底层调试，默认情况下基本系统中已找不到 gdb，但可以使用来自 LLVM 项目的优秀替代品 lldb，建议一试。

下面是使用 DTrace 一行命令生成 PostgreSQL 火焰图的示例：

```sh
# dtrace -x ustackframes=100 -n 'profile-5000 { ustack(); }' -p PID -o output
```

## 一些值得世界追上的优秀 API

FreeBSD 是许多技术的先驱。它是第一款引入 ZFS 的操作系统——Linux 世界用了多年才追上。FreeBSD 在 docker 出现很久以前就看到了容器的潜力——开发了 chroot 与 Jail。FreeBSD 兼容 POSIX，但还引入或集成了许多流行操作系统中所没有的优秀 API。

所有操作系统都实现了 poll 与 select，它们在可扩展性方面存在问题。我们也见到了 epoll，它非但没解决问题，反而引入了一些新问题。FreeBSD 走了不同路线，引入了 **kqueue(1)**。kqueue 在内核与用户态之间提供高效的输入输出事件管道。尤其在大量文件描述符上轮询事件时，kqueue 效率高得多。

谈到描述符，也应提到进程描述符——给进程添加句柄（描述符）的新方式。许多操作系统使用的进程标识符（PID）并不可靠。在检查进程状态与发送信号之间，操作系统内可能发生许多事情，理论上 PID 可能已被另一个进程占用。进程描述符通过给你一个可靠的进程句柄来解决这些问题。即便进程终止，你仍然能拿到通知它的句柄。

此外还有名为 Capsicum 的 FreeBSD 沙箱技术。Capsicum 是轻量级的操作系统能力与沙箱框架。基于能力的权限控制意味着进程只能执行没有全局影响的行为。进程不能通过绝对路径打开文件，也不能打开网络连接。其思路是在设计应用时考虑进程权限分离，从而保护应用。

要保护应用，还可以考虑使用 CloudABI（<https://github.com/NuxiNL/cloudabi>）。CloudABI 取一个 POSIX 函数，加入基于能力的权限控制，并删除与之不兼容的一切。这迫使软件开发者在应用中使用一组非常具体的函数集合，但提升了应用安全性。

FreeBSD 正在开发许多有趣的技术，未来或许会成为其他操作系统的标准。如果你想与时俱进，应该关注这一操作系统。

## 文档

FreeBSD 以文档优秀著称。软件开发方面也是如此。你多少次搜索过 ASCII 表？FreeBSD 默认就附带它的手册页：**ascii(7)**。架构/语言相关的特定内容同样如此：

- **arch(7)**——架构相关细节，如指针大小、数字、页面等
- **operator(7)**——C 与 C++ 运算符优先级、求值顺序
- **style(9)**——你能找到的最佳 C 风格，FreeBSD 已使用数十年
- **hier(7)**——理解 Unix 文件系统布局

## 天高任鸟飞

FreeBSD 拥有诸多让软件开发者生活更轻松的特性，例如 Jail、bhyve、ZFS 与 DTrace。系统中也能找到非常有用的文档。更不用说大量等待被使用的第三方软件。学习新语言的好方式之一是用它写些游戏——当然，这完全可以在 FreeBSD 上完成！所以别再犹豫，赶紧搭建你的 FreeBSD 开发者工作站！

---

**MARIUSZ ZABORSKI** 是 Fudo Security 的 QA 与开发经理。自 2015 年起自豪地持有 FreeBSD 提交权限。Mariusz 的主要兴趣领域是操作系统安全与底层编程。在 Fudo Security，他领导一支团队，开发用于监控、记录并控制 IT 基础设施流量的最先进解决方案。2018 年，Mariusz 组织了波兰 BSD 用户组。业余时间，他在 <https://oshogbo.vexillium.org> 撰写博客。
