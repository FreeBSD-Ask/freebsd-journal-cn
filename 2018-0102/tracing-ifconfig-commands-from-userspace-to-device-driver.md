# 从用户空间到设备驱动追踪 ifconfig 命令

作者：FARHAN KHAN

我目前正在扩展 FreeBSD 的 `rtwn`(4) 无线设备驱动。基础功能掌握，比如初始化、开关机、加载固件等，现在正在补充具体的 `ifconfig`(8) 方法。这要求深入理解 `ifconfig`(8) 命令最终如何传递到驱动。我找不到简明扼要的文档描述各阶段过程，于是自己写了一篇！

本文针对 FreeBSD 12.0-CURRENT，但应适用于任何未来版本以及使用 net80211(4) 的其他操作系统，如 OpenBSD、NetBSD、DragonFlyBSD 和 illumos。希望它能帮助 FreeBSD 社区继续开发 WiFi 和其他设备驱动。这不是一份详尽的指南，但应能提供基本的操作顺序。本例中，我将讲解如何更改 WiFi 网卡的信道并将其置为监控模式，如下：

```sh
# ifconfig wlan0 channel 6 wlanmode monitor
```

## 高层概述

FreeBSD 的 `ifconfig`(8) 使用 `lib80211`(3) 用户空间库，该库作为 API 填充内核数据结构并发起 `ioctl`(2) 系统调用。内核在新线程中接收 `ioctl`(2) 系统调用，解释结构，将命令路由到相应的栈。我们的例子中是 net80211(4)。内核随后创建一个排队任务并终止线程。之后，另一个内核线程接收排队任务并运行关联的 net80211(4) 处理函数，该函数立即将执行传递给设备驱动。

再次总结：

- 用户空间：`ifconfig`(8) 二进制
- 用户空间：`lib80211`(3) 库
- 内核：net80211(4) 代码
- 内核：通过 taskqueue(9) 到驱动
- 内核：设备驱动

开始吧！

## 用户空间：ifconfig(8) + lib80211(3) 库

在 `ifconfig`(8) 早期，它在 **/usr/src/sbin/ifconfig/ifconfig.c** 中打开一个 SOCK_DGRAM 套接字，如下：

```sh
s = socket(AF_LOCAL, SOCK_DGRAM, 0)
```

此套接字作为用户空间到内核通信的接口。与其从 `main()` 的 if-else 迷宫中追踪，我搜索了字符串“channel”，在 **/usr/src/sbin/ifconfig/ifieee80211.c** 末尾定义的 `ieee80211_cmd[]` 中找到了它。

该表枚举了所有 ieee80211 `ifconfig`(8) 命令。“channel”命令定义如下：

```sh
DEF_CMD_ARG ("channel" set80211channel)
```

注意第二个参数。我查找 DEF_CMD_ARG，发现它是一个预处理器宏，定义了用户向 `ifconfig`(8) 发送命令时运行的函数。快速搜索显示 `set80211channel` 定义在 **/usr/src/sbin/ifconfig/ifieee80211.c** 中。参数容易识别：`val` 是新的信道号（1 到 14），`s` 是我们之前打开的套接字。这执行 `ifconfig`(8) 的 `set80211` 函数，其唯一目的是干净地将执行转入 `lib80211`(3) 库。

`lib80211`(3) 是一个 802.11 无线网络管理库，用于与内核正式通信。值得注意的是，OpenBSD 和 NetBSD 都没有这个库，而是选择直接与内核通信。如前所述，`ifconfig`(8) 的 `set80211` 函数调用位于 **/usr/src/lib/lib80211/lib80211_ioctl.c** 的 `lib80211_set80211`。`lib80211_set80211` 函数填充 `ieee80211reqdata` 结构，用于用户到内核的 ieee80211 通信。下例中是 `ireq` 变量，包含 WiFi 接口名和目标信道。

该库随后调用 `ioctl`(2)，如下：

```sh
ioctl(s, SIOCS80211, &ireq)
```

这运行系统调用正式进入内核态执行。本质上，`ifconfig`(8) 不过是个花哨的 `ioctl`(2) 控制器。你可以编写自己的接口配置工具直接调用 `ioctl`(2) 系统调用并得到相同结果。现在进入内核！

## 内核命令路由到 net80211(4)

继续之前有两点简要说明：首先，在高层面上，BSD 内核的运作方式类似 IP 路由器，它在内核中路由执行，沿途填充相关数据值，直到执行到达目标处理函数。以下说明将展示内核如何识别系统调用类型、判断它针对接口卡、判断接口卡类型，最后为将来执行排队一个任务。

其次，BSD 内核使用一种常见模式：模板方法调用一系列函数指针。确切的函数指针有条件地填充，使代码保持一致结构而具体实现可能不同。这种方式效果很好，但如果你只是从头阅读代码，追踪执行路径会很困难。遇到困难时，我通常使用 illumos 的 OpenGrok（<http://src.illumos.org/source/>）或 `dtrace`(1)。

让我们简要偏离到 `dtrace`(1)。Solaris 的动态追踪工具导入到 FreeBSD，用于实时监控内核或进程。它有助于理解操作系统的行为，省去了 `printf`(3) 式调试的麻烦。我在编写本指南时使用 `dtrace`(1) 来识别内核在执行什么、函数参数以及任意时刻的栈追踪。例如，如果我想监控 `ifioctl` 函数，可以运行：

```sh
# dtrace -n '
> fbt:kernel:ifioctl:entry {
> self->cmd = args[1];
> stack(10);
> }
> fbt:kernel:ifioctl:return {
> printf("ifioctl(cmd=%x) = %x", self->cmd, arg1);
> exit(0);
> } '
```

这条 `dtrace`(1) 单行命令为 `ifioctl` 的入口和返回探针设置处理函数。入口时，`dtrace`(1) 记录第二个参数 `cmd` 的值，并显示栈的最后 10 个元素。返回时，显示函数参数和返回值。我在研究中反复使用此基本命令模板的变体，尤其是在追踪代码遇到困惑或无法识别函数参数时。

第一个非汇编函数是 amd64 特定的系统调用处理函数 `amd64_syscall`，它接收新的线程结构并识别类型为系统调用。我们的例子中是 `ioctl`(2)，所以 `amd64_syscall` 调用位于 **/usr/src/sys/kern/sys_generic.c** 的 `sys_ioctl`。

在 FreeBSD 上，`sys_ioctl` 执行输入验证并格式化接收的数据，然后调用 `kern_ioctl`，后者判断 `ioctl`(2) 操作的文件描述符类型、套接字的能力，并相应分配函数指针 `fo_ioctl`。（NetBSD 和 OpenBSD 没有 `kern_ioctl`，`sys_ioctl` 直接调用 `fo_ioctl`。）我们的文件描述符对应一个接口，所以 FreeBSD 将 `fo_ioctl` 分配为指向 `ifioctl` 的函数指针，`ifioctl` 处理接口层 `ioctl`(2) 调用。该函数位于 **/usr/src/sys/net/if.c**。

`ifioctl` 负责所有类型的接口：以太网、WiFi、`epair`(4) 等。`ifioctl` 以基于 `cmd` 参数的 switch 条件开始。这检查命令是否可由 net80211(4) 处理而无需跳入驱动，比如创建克隆接口或更新 MTU。快速的 `dtrace`(2) 探针显示 `cmd` 参数为 `SIOCS80211`，不满足任何 switch 条件，因此执行跳到底部。函数继续并调用 `ifp->if_ioctl`，对于 WiFi，这是指向 `ieee80211_ioctl` 的函数指针，位于 **/usr/src/sys/net80211/ieee80211_ioctl.c**。

`ieee80211_ioctl` 包含另一个 switch-case。`cmd` 设为 `SIOCS80211` 时，执行匹配关联的 case 并调用 `ieee80211_ioctl_set80211`，位于 **/usr/src/sys/net80211/ieee80211_ioctl.c**。

`ieee80211_ioctl_set80211` 还有另一个 switch-case，有几十个条件。`ireq->i_type` 由 `lib80211`(3) 设为 `IEEE80211_IOC_CHANNEL`，因此将匹配关联的 case 并执行 `ieee80211_ioctl_setchannel`。该函数的要点是判断输入信道是否有效，或内核是否需要设置其他值。它最终调用 `setcurchan`，后者做两件事：首先判断信道有效性及是否需设置附加值；其次运行 `ieee80211_runtask`，这是线程级的最终调用 `taskqueue_enqueue`。

## 内核：任务执行

`taskqueue_enqueue` 不是 ieee80211(9) 函数，但值得简要回顾。简言之，taskqueue(9) 框架允许你将代码执行推迟到未来。例如，若想延迟 3 秒执行，运行内核版的 `sleep`(3) 会使整个 CPU 核心停顿 3 秒，这不可接受。taskqueue(9) 允许你指定一个函数，由内核在稍后执行。

我们的信道变更示例中，调度的函数是 net80211(4) 的 `update_channel`，位于 **/usr/src/sys/net80211/ieee80211_proto.c**。当 taskqueue(9) 到达我们入队的任务时，它首先启动 `update_channel` 处理函数接收任务，并立即将执行交给 `ic_set_channel` 指向的驱动代码。

总结到此：内核将命令路由到网络栈，网络栈路由到 WiFi 专用栈，在那里作为任务排队等待将来执行。当 taskqueue(9) 到达任务时，它立即跳转到驱动专用代码。终于进入驱动了！

## 驱动

从这里开始，代码是驱动专用的，我不会深入实现细节，因为每个设备都有自己独特的信道切换流程。我目前在 `rtwn`(9) 上工作，位于 **/usr/src/sys/dev/rtwn**。NetBSD 和 OpenBSD 分离 USB 和 PCI 驱动，因此同一驱动分别位于 **/usr/src/sys/dev/usb/if_urtwn.c** 和 **/usr/src/sys/dev/pci/if_rtwn.c**。

操作系统需要一种标准方式与设备驱动通信。通常，驱动提供一个包含一系列函数指针的结构，内核将其用作进入驱动代码的入口点。对于 WiFi，此结构是 `ieee80211com`，位于 **/usr/src/sys/net80211/ieee80211_var.h**。按惯例，所有 BSD 衍生系统使用变量名 `ic` 处理 ieee80211(9) 方法。

我们的例子中要切换信道，所以操作系统将调用 `ic->ic_set_channel`，它指向驱动的信道切换函数。对于 `rtwn`(9)，这是 `rtwn_set_channel`，它本身是函数指针，指向 `r92c_set_chan`、`r92e_set_chan` 或 `r12a_set_chan`，取决于你使用的具体设备。

`rtwn`(9) 的细节超出本文范围，但值得讨论驱动如何与硬件通信。softc 结构是维护设备运行时变量、状态和方法实现的 struct。按惯例，每个驱动的 softc 实例称为 `sc`。你可能会想，`ieee80211com` 提供方法函数指针，为何还需要另一个。这是因为 `ieee80211com` 的方法指向命令处理函数，不一定是设备例程。设备驱动可能有自己内部的方法不属于 `ieee80211com`。此外，softc 结构可处理设备版本间的细微差异。`rtwn`(9) 的 softc 结构名为 `rtwn_softc`，位于 **/usr/src/sys/dev/rtwn/if_rtwnvar.h**。

驱动如何向设备发送数据？`rtwn`(9) 使用 `rtwn_write_[1|2|4]` 和 `rtwn_read_[1|2|4]` 方法实际发送或接收字节、字或双字。`rtwn_read_1` 是指向 `sc_read_1` 方法的指针。驱动在初始化时将 `sc_read` 类函数分配给 `rtwn_usb_read_*` 和 `rtwn_usb_write_*` 方法，或 `rtwn_pci_read_*` 和 `rtwn_pci_write_*`。上述函数类是对 PCI 和 USB 总线的抽象。对于 PCI，这些函数调用最终调用 `bus_space_read_*` 和 `bus_space_write_*`，它们属于 PCI 子系统。对于 USB，驱动将调用 `usbd_do_request_flags`，属于 USB 子系统。编写良好的驱动应抽象这些总线特定层，为你提供干净的各种数据大小的读写方法。顺带一提，FreeBSD 早就该有一个 SDIO 栈，这对树莓派、Chromebook 和其他嵌入式设备是重大障碍。不过我跑题了……

举例来说，驱动用以下行启用硬件中断：

```sh
rtwn_write_4(sc, R92C_HIMR, R92C_INT_ENABLE)
```

这会将值 `R92C_INT_ENABLE` 写入 `R92C_HIMR` 设备寄存器。

## 结论

总结这段旅程：`ifconfig`(8) 打开一个套接字并传递给 `lib80211`(3) 库。`lib80211`(3) 通过 `ioctl`(2) 系统调用向内核发送用户到内核的命令结构。系统调用触发内核运行新的内核线程。内核从这里判断 `ioctl`(2) 命令对应网卡，指定类型为 WiFi 卡，再识别确切的命令类型。ieee80211(9) 让 taskqueue 创建新任务来切换 WiFi 信道，然后终止。之后，taskqueue(9) 运行 ieee80211(9) 任务处理函数，将执行传递给驱动。驱动使用 PCI 或 USB 总线与硬件通信来切换 WiFi 信道。

在我看来，FreeBSD 在技术上优于 Linux，但在若干关键领域仍有不足，硬件支持就是其中之一。希望本文能推动 FreeBSD 社区继续产出高质量、更快的设备驱动。

FARHAN KHAN 2012 年毕业于乔治梅森大学，主修应用信息技术安全。他目前担任 McAfee Security 的高级安全工程师。他自视为 IPv6 布道者、热心的程序员，并希望为 FreeBSD 项目做出贡献。
