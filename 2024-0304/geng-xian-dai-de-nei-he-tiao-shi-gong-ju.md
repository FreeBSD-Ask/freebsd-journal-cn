# 更现代的内核调试工具

- 原文链接：[More Modern Kernel Debugging Tools](https://freebsdfoundation.org/our-work/journal/browser-based-edition/development-workflow-and-ci/more-modern-kernel-debugging-tools/)
- 作者：Tom Jones

恐慌（Panic）是一个奇妙的词！

它精要概述了一个极其复杂的情绪事件。我们可以说“士兵们慌乱了”，于是便能想象一场战斗的局势。我们可以用它来表达小小的疏忽带来的分量，也可以解释当我们踏上飞机却怀疑自己是否关掉了烤箱时的大脑想法。

肯定是关了吧。

“恐慌（Panic）”也许是我最喜欢的书名词汇之一：《Panic! Unix 系统崩溃转储分析》（*Panic! Unix System Crash Dump Analysis*）——哇！我一定得有一本。所有使用此类标题的作者肯定写了一本好书，即使它只是一本陈旧的 Sun OS/Solaris 技术手册。

优秀的书名比技术论文更具有时效性，而最先过时的内容往往是现有系统的厂商文档。到了 2024 年，我发现非常难找到在操作系统调试方法方面的最新资料。看看当下的出版作品，你会以为我们在 2004 年就已经完善了操作系统。我从未使用过 SunOS 和 Solaris，但《Panic!》给了我一直想要的崩溃分析入门。

我承认，我一直不理解为什么人们如此渴望核心转储——我会想，核心转储能比堆栈跟踪给我们更多什么呢？但从《Panic!》中，我迈出了崩溃分析的第一步，体验了在 FreeBSD 上使用顶尖内核调试工具的旅程。让我来展示我学到的东西。别担心，不需要等着湿透的柠檬纸巾了。（**译者注：此处指不必无聊地等待和无意义的步骤，即本文切中主旨，简单高效**）

## 获取内核转储

我得坦白，明知你不会尝试进行内核转储分析，除非确有必要。当你遇到一台卡住的机器时，这种情况下的生活就是打印调试的世界。

获取能用的核心转储并不难——你需要把系统设置成接收崩溃转储（参见 dumpon(8)），然后进入调试器。通常，系统会主动帮助你进入调试器，让系统陷入 Panic。FreeBSD 提供了一项功能，即使在没有出现问题时也可以让内核陷入 Panic。在测试系统上将 sysctl `debug.kdb.panic` 设置 1，即可进入调试提示符：

```sh
# sysctl debug.kdb.panic=1
```

如果你在桌面和云中的重要工作机上运行了这个命令，可能会遇到麻烦（如果是桌面，这篇文章可能已经消失了）。我建议在虚拟机和至少不会引发大麻烦的设备上学习内核调试。

此外，FreeBSD 虚拟机镜像默认配置为在启动时运行 savecore，且保存崩溃转储文件。

在设置 `debug.kdb.panic` 后，你将进入 ddb(4) 提示符。ddb 是一款功能齐全的实时系统调试器——它是款很好的分析工具，但今天我们用不到它。

在 ddb 中，可以使用 `dump` 命令导出运行中的内核。

```sh
ddb> dump
Dumping 925 out of 16047 MB:..2%..11%..21%..32%..42%..51%..61%..71%..82%..92%
```

Panic 状态下系统无法使用，因此需要重启以继续操作。

```sh
ddb> reboot
```

当虚拟机重启时，`savecore` 会显示一条信息，提示已经提取并保存了核心文件。

核心文件将被放置在目录 `/var/crash` 下，还有其他一些文件。

```sh
$ ls /var/crash
bounds core.txt.0 info.0 info.last minfree
vmcore.0 vmcore.last
```

本次测试的核心文件是 `vmcore.0`，并附有相应的 `info.0` 和 `core.txt.0` 文件。info 文件是主机和转储的摘要，`core.txt` 是转储文件的摘要、消息缓冲区中的未读部分以及可能存在的恐慌字符串和堆栈跟踪。

```sh
Dump header from device: /dev/nvd0p3
Architecture: amd64
Architecture Version: 2
Dump Length: 970199040
Blocksize: 512
Compression: none
Dumptime: 2023-05-17 14:07:58 +0100
Hostname: displacementactivity
Magic: FreeBSD Kernel Dump
Version String: FreeBSD 14.0-CURRENT #2 main-n261806-d3a49f62a284: Mon Mar 27 16:15:25 UTC 2023
tj@displacementactivity:/usr/obj/usr/src/amd64.amd64/sys/GENERIC
Panic String: Duplicate free of 0xfffff80339ef3000 from zone 0xfffffe001ec2ea00(malloc-2048) slab 0xfffff80325789168(0)
Dump Parity: 3958266970
Bounds: 0
Dump Status: good
```

bounds 文件告诉转储程序下一个核心转储将被命名为 `vmcore.1` 当前机器上的 bounds 是：

```sh
# cat /var/crash/bounds
1
```

最后，`vmcore.last` 是指向最近的核心转储文件的链接，以防你经历了有趣的一周，忘记了最新的崩溃记录。

### 符号

分析核心转储的第二个必要内容是内核符号。版本发布的内核符号可以通过包 `kernel-dbg` 获取，安装到目录 `/usr/lib/debug/`，或者可以从你的内核构建目录中提取出来。

### 使用 gdb 查看核心转储（入门）

首先，我们用 `kgdb` 简单快速地查看核心转储，作为后续使用 lldb 进行崩溃转储调试的参考位置。

```sh
$ kgdb kernel.debug vmcore.0

Unread portion of the kernel message buffer:
panic: Assertion !tcp_in_hpts(tp) failed at /usr/src/sys/netinet/tcp_subr.c:2432
cpuid = 2
time = 1706644478
KDB: stack backtrace:
db_trace_self_wrapper() at db_trace_self_wrapper+0x2b/frame 0xfffffe0047d2f480
vpanic() at vpanic+0x132/frame 0xfffffe0047d2f5b0
panic() at panic+0x43/frame 0xfffffe0047d2f610
tcp_discardcb() at tcp_discardcb+0x25b/frame 0xfffffe0047d2f660
tcp_usr_detach() at tcp_usr_detach+0x51/frame 0xfffffe0047d2f680
sorele_locked() at sorele_locked+0xf7/frame 0xfffffe0047d2f6b0
tcp_close() at tcp_close+0x155/frame 0xfffffe0047d2f6e0
rack_check_data_after_close() at rack_check_data_after_close+0x8a/frame 0xfffffe0047d2f720
rack_do_fin_wait_1() at rack_do_fin_wait_1+0x141/frame 0xfffffe0047d2f7a0
rack_do_segment_nounlock() at rack_do_segment_nounlock+0x243b/frame 0xfffffe0047d2f9a0
rack_do_segment() at rack_do_segment+0xda/frame 0xfffffe0047d2fa00
tcp_input_with_port() at tcp_input_with_port+0x1157/frame 0xfffffe0047d2fb50
tcp_input() at tcp_input+0xb/frame 0xfffffe0047d2fb60
ip_input() at ip_input+0x2ab/frame 0xfffffe0047d2fbc0
netisr_dispatch_src() at netisr_dispatch_src+0xad/frame 0xfffffe0047d2fc20
ether_demux() at ether_demux+0x17a/frame 0xfffffe0047d2fc50
ether_nh_input() at ether_nh_input+0x39f/frame 0xfffffe0047d2fca0
netisr_dispatch_src() at netisr_dispatch_src+0xad/frame 0xfffffe0047d2fd00
ether_input() at ether_input+0xd9/frame 0xfffffe0047d2fd60
vtnet_rxq_eof() at vtnet_rxq_eof+0x73e/frame 0xfffffe0047d2fe20
vtnet_rx_vq_process() at vtnet_rx_vq_process+0x9c/frame 0xfffffe0047d2fe60
ithread_loop() at ithread_loop+0x266/frame 0xfffffe0047d2fef0
fork_exit() at fork_exit+0x82/frame 0xfffffe0047d2ff30
fork_trampoline() at fork_trampoline+0xe/frame 0xfffffe0047d2ff30
--- trap 0, rip = 0, rsp = 0, rbp = 0 ---
KDB: enter: panic

Reading symbols from /boot/kernel/zfs.ko...
Reading symbols from /usr/lib/debug//boot/kernel/zfs.ko.debug...
Reading symbols from /boot/kernel/tcp_rack.ko...
Reading symbols from /usr/lib/debug//boot/kernel/tcp_rack.ko.debug...
Reading symbols from /boot/kernel/tcphpts.ko...
Reading symbols from /usr/lib/debug//boot/kernel/tcphpts.ko.debug...
__curthread () at /usr/src/sys/amd64/include/pcpu_aux.h:57
57 __asm(“movq %%gs:%P1,%0” : “=r” (td) : “n” (offsetof(struct pcpu,
(kgdb)
```

`kgdb` 启动时会显示许可证（已省略），然后打印内核信息缓冲区的最后部分，这是由 `kgdb` 提供的一个很有用的功能。消息缓冲区的最后部分向我们提供了 Panic 信息、一些其他信息以及堆栈跟踪。

使用 `kgdb` 的 `bt`（backtrace）命令，我们可以获取堆栈跟踪，并通过 `frames` 命令在堆栈中移动，查看 Panic 发生时的具体情况。

```sh
(kgdb) bt
...
#10 0xffffffff80b51233 in vpanic (fmt=0xffffffff811f87ca “Assertion %s failed at %s:%d”, ap=ap@entry=0xfffffe0047d2f5f0) at /usr/src/sys/kern/kern_shutdown.c:953
#11 0xffffffff80b51013 in panic (fmt=0xffffffff81980420 <cnputs_mtx> “\371\023\025\201\377\377\377\377”) at /usr/src/sys/kern/kern_shutdown.c:889
#12 0xffffffff80d5483b in tcp_discardcb (tp=tp@entry=0xfffff80008584a80) at /usr/src/sys/netinet/tcp_subr.c:2432
#13 0xffffffff80d60f71 in tcp_usr_detach (so=0xfffff800100b6b40) at /usr/src/sys/netinet/tcp_usrreq.c:215
#14 0xffffffff80c01357 in sofree (so=0xfffff800100b6b40) at /usr/src/sys/kern/uipc_socket.c:1209
#15 sorele_locked (so=so@entry=0xfffff800100b6b40) at /usr/src/sys/kern/uipc_socket.c:1236
#16 0xffffffff80d545b5 in tcp_close (tp=<optimized out>) at /usr/src/sys/netinet/tcp_subr.c:2539
#17 0xffffffff82e37e0a in tcp_tv_to_usectick (sv=0xfffffe0047d2f698) at /usr/src/sys/netinet/tcp_hpts.h:177
#18 tcp_get_usecs (tv=0xfffffe0047d2f698) at /usr/src/sys/netinet/tcp_hpts.h:232
...
(kgdb) frame 12
#12 0xffffffff80d5483b in tcp_discardcb (tp=tp@entry=0xfffff80008584a80) at /usr/src/sys/netinet/tcp_subr.c:2432
warning: Source file is more recent than executable.
2432
(kgdb) list
2427 #endif
2428
2429 CC_ALGO(tp) = NULL;
2430 if ((m = STAILQ_FIRST(&tp->t_inqueue)) != NULL) {
2431 struct mbuf *prev;
2432
2433 STAILQ_INIT(&tp->t_inqueue);
2434 STAILQ_FOREACH_FROM_SAFE(m, &tp->t_inqueue, m_stailqpkt, prev)
2435 m_freem(m);
2436 }
```

在这里，我列出了导致 Panic 的回溯路径，识别了大约在 frame #11 处调用的 `panic`，并要求 `kgdb` 跳转到 frame **#12**（导致 Panic 的具体代码），然后列出该处的代码。进一步调查可以帮助我们找出在此崩溃转储中导致 Panic 的原因。

这是内核调试的基本步骤，查看当时的情况并分析崩溃转储，以找出变量所持有的值。为了在内核上下文中有用，lldb 也需要能够完成这些任务。

### lldb

在过去的十年里，FreeBSD 一直在向更加自由许可的 llvm/clang 工具链迈进。尽管一度缺失调试支持，但在 2024 年，FreeBSD 内核的调试功能终于在 lldb 实现。

lldb 可以将 FreeBSD 内核转储导入为核心文件，并在堆栈帧之间移动。

lldb 是由苹果开发的。我还记得他们将默认调试器从 gdb 切换为 lldb 时，我感到非常不适应。过去从简陋的 GNU 文档中学到的所有调试命令都不见了，取而代之的是一套全新的命令。

实际上，lldb 并非专为命令行界面设计，而是为了通过 API 被软件驱动。这导致了它的许多命令显得冗长。幸运的是，lldb 的命令行现在对 gdb 风格的命令兼容得更多了，许多命令界面更为一致。基本命令如打印操作现已兼容，但其他选项仍然有差异，有些更好，有些则差很多。

### 使用 lldb 探索

无需特殊配置 lldb 即可分析内核转储。加载崩溃转储的方式与 kgdb 类似，只是参数位置略有不同：

```sh
$ lldb --core <核心文件> path/to/kernel/symbols
```

在我们的示例中，命令如下：

```sh

$ lldb --core ../gdb/coredump/vmcore.0 ../gdb/coredump/kernel-debug/kernel.debug
(lldb) target create “../gdb/coredump/kernel-debug/kernel.debug” --core “../gdb/coredump/vmcore.0”
Core file '/home/tj/code/scripts/gdb/coredump/vmcore.0' (x86_64) was loaded.
(lldb)
```

这比 kgdb 的启动过程更简洁，但缺少了一些崩溃转储的关键信息。究竟是什么导致了此次转储呢？

`kgdb` 并没有进行什么神奇的操作（否则它可能会有个“修复”（fix）命令来搭配“断点”（break）命令）。它所做的只是查找转储中的已知符号，并在启动时将其打印出来。

我们可以自己做这些操作。

首先，内核中的恐慌信息存储在字符串 `panicstr` 中，并由 `vpanic` 设置（位于 `kern/kern_shutdown.c`）。我们可以通过 lldb 从转储中轻松提取这一信息：

```sh
(lldb) p panicstr
(const char *) 0xffffffff819c1a00 “Assertion !tcp_in_hpts(tp) failed at /usr/src/sys/netinet/tcp_subr.c:2432”
```

这可能足够让人开始进行调试。我也喜欢通过 lldb 的 `bt` 命令获取堆栈跟踪：

```sh
(lldb) bt
* thread #1, name = '(pid 1025) tcplog_dumper'
* frame #0: 0xffffffff80b83d2a kernel.debug`sched_switch(td=0xfffff800174be740, flags=259) at sched_ule.c:2297:26
frame #1: 0xffffffff80b5e9e3 kernel.debug`mi_switch(flags=259) at kern_synch.c:546:2
frame #2: 0xffffffff80bb0dc4 kernel.debug`sleepq_switch(wchan=0xffffffff817e1448, pri=0) at subr_sleepqueue.c:607:2
frame #3: 0xffffffff80bb11a6 kernel.debug`sleepq_catch_signals(wchan=0xffffffff817e1448, pri=0) at subr_sleepqueue.c:523:3
frame #4: 0xffffffff80bb0ef9 kernel.debug`sleepq_wait_sig(wchan=<unavailable>, pri=<unavailable>) at subr_sleepqueue.c:670:11
frame #5: 0xffffffff80b5df3c kernel.debug`_sleep(ident=0xffffffff817e1448, lock=0xffffffff817e1428, priority=256, wmesg=”tcplogdev”, sbt=0, pr=0, flags=256) at kern_synch.c:219:10
frame #6: 0xffffffff8091190e kernel.debug`tcp_log_dev_read(dev=<unavailable>, uio=0xfffffe0079b4ada0, flags=0) at tcp_log_dev.c:303:9
frame #7: 0xffffffff809d99ce kernel.debug`devfs_read_f(fp=0xfffff80012857870, uio=0xfffffe0079b4ada0, cred=<unavailable>, flags=0, td=0xfffff800174be740) at devfs_vnops.c:1413:10
frame #8: 0xffffffff80bc9bc6 kernel.debug`dofileread [inlined] fo_read(fp=0xfffff80012857870, uio=0xfffffe0079b4ada0, active_cred=<unavailable>, flags=<unavailable>, td=0xfffff800174be740) at file.h:340:10
frame #9: 0xffffffff80bc9bb4 kernel.debug`dofileread(td=0xfffff800174be740, fd=3, fp=0xfffff80012857870, auio=0xfffffe0079b4ada0, offset=-1, flags=0) at sys_generic.c:365:15
frame #10: 0xffffffff80bc9712 kernel.debug`sys_read [inlined] kern_readv(td=0xfffff800174be740, fd=3, auio=0xfffffe0079b4ada0) at sys_generic.c:286:10
frame #11: 0xffffffff80bc96dc kernel.debug`sys_read(td=0xfffff800174be740, uap=<unavailable>) at sys_generic.c:202:10
frame #12: 0xffffffff810556a3 kernel.debug`amd64_syscall [inlined] syscallenter(td=0xfffff800174be740) at subr_syscall.c:186:11
frame #13: 0xffffffff81055581 kernel.debug`amd64_syscall(td=0xfffff800174be740, traced=0) at trap.c:1192:2
frame #14: 0xffffffff8102781b kernel.debug`fast_syscall_common at exception.S:578
```

我们还可以选择一个感兴趣的 frame 来进一步查看：

```sh
(lldb) frame select 12
frame #12: 0xffffffff810556a3 kernel.debug`amd64_syscall [inlined] syscallenter(td=0xfffff800174be740) at subr_syscall.c:186:11
183 if (!sy_thr_static)
184 syscall_thread_exit(td, se);
185 } else {
-> 186 error = (se->sy_call)(td, sa->args);
187 /* Save the latest error return value. */
188 if (__predict_false((td->td_pflags & TDP_NERRNO) != 0))
189 td->td_pflags &= ~TDP_NERRNO;
```

### 获取内核缓冲区

打印信息并在堆栈中移动是内核崩溃转储调试所需的大部分操作。gdb 的启动消息很不错，会展示内核消息缓冲区的最后部分，类似于从本地控制台输出的内容。

lldb 目前还没有类似的启动命令。不过，通过查看“Panic!”信息，我们可以找到提取该信息的方法。“Panic!”使用了一个名为 `msgbuf` 的宏，通过 `struct msgbuf` 打印内核消息缓冲区。

通过查阅 FreeBSD 的源代码，我们可以找到类似的代码来完成此操作：

```sh
(lldb) p *msgbufp
(msgbuf) {
msg_ptr = 0xfffff8001ffe8000 “---<<BOOT>>---\nCopyright (c) 1992-2023 The FreeBSD Project.\nCopyright (c) 1979, 1980, 1983, 1986, 1988, 1989, 1991, 1992, 1993, 1994\n\tThe Regents of the University of California. All rights reserved.\nFreeBSD is a registered trademark of The FreeBSD Foundation.\nFreeBSD 15.0-CURRENT #0 main-272a40604: Wed Nov 29 13:42:38 UTC 2023\n tj@vpp:/usr/obj/usr/src/amd64.amd64/sys/GENERIC amd64\nFreeBSD clang version 16.0.6 (https://github.com/llvm/llvm-project.git llvmorg-16.0.6-0-g7cbf1a259152)\nWARNING: WITNESS option enabled, expect reduced performance.\nVT: init without driver.\nCPU: 12th Gen Intel(R) Core(TM) i7-1260P (2500.00-MHz K8-class CPU)\n Origin=\”GenuineIntel\” Id=0x906a3 Family=0x6 Model=0x9a Stepping=3\n Features=0x9f83fbff<FPU,VME,DE,PSE,TSC,MSR,PAE,MCE,CX8,APIC,SEP,MTRR,PGE,MCA,CMOV,PAT,PSE36,MMX,FXSR,SSE,SSE2,SS,HTT,PBE>\n Features2=0xfeda7a17<SSE3,PCLMULQDQ,DTES64,DS_CPL,SSSE3,SDBG,FMA,CX16,xTPR,PCID,SSE4.1,SSE4.2,MOVBE,POPCNT,AESNI,XSAVE,OSXSAVE,AVX,F16C,RDRAND,HV>\n AMD Features=0x2c100800<SYSCALL,”...
msg_magic = 405602
msg_size = 98232
msg_wseq = 16777
msg_rseq = 15001
msg_cksum = 1421737
msg_seqmod = 1571712
msg_lastpri = -1
msg_flags = 0
msg_lock = {
lock_object = {
lo_name = 0xffffffff81230bcc “msgbuf”
lo_flags = 196608
lo_data = 0
lo_witness = NULL
}
mtx_lock = 0
}
}
```

在内核中有一个全局可见的结构体 `msgbuf`，用于实现内核的消息缓冲区。lldb 可以显示缓冲区的起始部分。字段 `msg_wseq` 和 `msg_rseq` 告诉我们写入和读取的位置。

读取消息缓冲区中未读部分非常简单：

```sh
(lldb) p msgbufp->msg_ptr+msgbufp->msg_rseq
(char *) 0xfffff8001ffeba99 “panic: Assertion !tcp_in_hpts(tp) failed at /usr/src/sys/netinet/tcp_subr.c:2432\ncpuid = 2\ntime = 1706644478\nKDB: stack backtrace:\ndb_trace_self_wrapper() at db_trace_self_wrapper+0x2b/frame 0xfffffe0047d2f480\nvpanic() at vpanic+0x132/frame 0xfffffe0047d2f5b0\npanic() at panic+0x43/frame 0xfffffe0047d2f610\ntcp_discardcb() at tcp_discardcb+0x25b/frame 0xfffffe0047d2f660\ntcp_usr_detach() at tcp_usr_detach+0x51/frame 0xfffffe0047d2f680\nsorele_locked() at sorele_locked+0xf7/frame 0xfffffe0047d2f6b0\ntcp_close() at tcp_close+0x155/frame 0xfffffe0047d2f6e0\nrack_check_data_after_close() at rack_check_data_after_close+0x8a/frame 0xfffffe0047d2f720\nrack_do_fin_wait_1() at rack_do_fin_wait_1+0x141/frame 0xfffffe0047d2f7a0\nrack_do_segment_nounlock() at rack_do_segment_nounlock+0x243b/frame 0xfffffe0047d2f9a0\nrack_do_segment() at rack_do_segment+0xda/frame 0xfffffe0047d2fa00\ntcp_input_with_port() at tcp_input_with_port+0x1157/frame 0xfffffe0047d2fb50\ntcp_input() at tcp_input+0xb/frame 0xfffffe0047d2fb60\nip_input() at ip_in”...
```

输出的格式不太友好，控制字符直接打印出来了，不过我们仍然能读取内核消息缓冲区的内容。由于输出被截断，无法获取完整的回溯信息。让我们试试其他命令：

```sh
(lldb) x/b msgbufp->msg_ptr+msgbufp->msg_rseq
0xfffff8001ffeba99: “panic: Assertion !tcp_in_hpts(tp) failed at /usr/src/sys/netinet/tcp_subr.c:2432\ncpuid = 2\ntime = 1706644478\nKDB: stack backtrace:\ndb_trace_self_wrapper() at db_trace_self_wrapper+0x2b/frame 0xfffffe0047d2f480\nvpanic() at vpanic+0x132/frame 0xfffffe0047d2f5b0\npanic() at panic+0x43/frame 0xfffffe0047d2f610\ntcp_discardcb() at tcp_discardcb+0x25b/frame 0xfffffe0047d2f660\ntcp_usr_detach() at tcp_usr_detach+0x51/frame 0xfffffe0047d2f680\nsorele_locked() at sorele_locked+0xf7/frame 0xfffffe0047d2f6b0\ntcp_close() at tcp_close+0x155/frame 0xfffffe0047d2f6e0\nrack_check_data_after_close() at rack_check_data_after_close+0x8a/frame 0xfffffe0047d2f720\nrack_do_fin_wait_1() at rack_do_fin_wait_1+0x141/frame 0xfffffe0047d2f7a0\nrack_do_segment_nounlock() at rack_do_segment_nounlock+0x243b/frame 0xfffffe0047d2f9a0\nrack_do_segment() at rack_do_segment+0xda/frame 0xfffffe0047d2fa00\ntcp_input_with_port() at tcp_input_with_port+0x1157/frame 0xfffffe0047d2fb50\ntcp_input() at tcp_input+0xb/frame 0xfffffe0047d2fb60\nip_input() at ip_i”
warning: unable to find a NULL terminated string at 0xfffff8001ffeba99. Consider increasing the maximum read length.
(lldb) x/2048b msgbufp->msg_ptr+msgbufp->msg_rseq
error: Normally, ‘memory read’ will not read over 1024 bytes of data.
error: Please use --force to override this restriction just once.
error: or set target.max-memory-read-size if you will often need a larger limit.
```

我们遇到了打印限制。尽管进行了尝试，还是无法让 lldb 读取更多内容。是时候使用更高级的工具了。

## 一些来自 Lua 的帮助

lldb 还提供了一个脚本接口进行控制，这也是为什么许多命令的输入非常冗长。目前，lldb 支持 C++ 和 Python 脚本，且已试验性地支持 Lua。FreeBSD 的基本系统中自带 Lua，而 2024 年的 FreeBSD 版本的 lldb 默认支持 Lua。

可以通过以下方式简单试验：

```lua
(lldb) script
>>> print(“hello esteemed FreeBSD Journal readers!”)
hello esteemed FreeBSD Journal readers!
>>> quit
```

提示符 `>>>` 表明我们已进入 Lua 解释器。

在“Panic!”一书中，我们了解到 SunOS/Solaris 调试器 adb 有一个方便易懂的宏，用于查找和打印消息缓冲区：

```sh
msgbuf/”magic”16t”size”16t”bufx”16t”bufr”n4X
+,(*msgbuf+0t8)-*(msgbuf+0t12)))&80000000$<msgbuf.wrap
.+*(msgbuf+0t12),(*(msgbuf+0t8)-*(msfbuf+0t12))/c
```

使用 Lua 实现类似的机制应该没问题，因为这已经提供了一个示例。

lldb 的 Lua 接口是通过 swig 绑定生成的。swig 是一种使用 C++ 格式来描述库间接口的工具。Python 和 Lua 绑定是用相同的方式生成的。如果对 API 及如何使用有疑问，可通过查看 lldb 项目提供的 Python API 文档找到答案。虽说这种方法不太方便，但还是能用。

很快我就厌倦了在解释器中运行命令，考虑到命令的长度，有时操作起来很麻烦。lldb 可以在解释器运行后从文件中加载 Lua 脚本。使用一个全新的会话：

```lua
$ lldb --core coredump/vmcore.1 coredump/kernel-debug/kernel.debug
(lldb) target create “coredump/kernel-debug/kernel.debug” --core “coredump/vmcore.1”
Core file ‘/home/tj/code/scripts/gdb/coredump/vmcore.1’ (x86_64) was loaded.
(lldb) script
>>> print(“hello”)
hello
>>> quit
(lldb) command script import ./hello.lua
hello from the script hello.lua
```

假设文件 `hello.lua` 内容为：

```lua
print(“hello from the script hello.lua”)
```

lldb 的 Lua 环境提供了一个变量 `lldb`，包含了访问目标、调试器、帧、进程和线程的成员。这些对象映射到 Python API 中描述的对象。

我并不是很喜欢 lldb 的 API，编写起来比较繁琐，如果选择的函数和变量布局有问题，理解起来也比较困难。

不过积累了一些经验后，就能更好地理解它的需求。

让我通过一个示例来说明如何使用 lldb 的 Lua 绑定从崩溃转储中打印消息缓冲区。

通过 lldb 的 Lua 变量，我们能访问转储镜像中的文件。当我开始分析核心转储时，一个主要的困难是理解如何在内存中定位信息。可以从各种内核全局变量入手，大多数子系统都有可以用于构建的内容。

正如我们之前所看到的，`msgbufp` 是内核消息缓冲区的全局实例。在 lldb Lua 中可以通过以下方式访问它：

```lua
msgbuf = lldb.target:FindFirstGlobalVariable(“msgbufp”)
```

这会返回一个 SBValue 实例，代表核心转储中内存中的这个结构体实例。我们可以使用 `GetChildMemberWithName` 方法和成员名（例如 `msg_rseq`）来访问结构体的子成员。

对象 `lldb.process` 允许我们从内核转储中读取内存。正确组合引用、地址和值来执行所需操作有时可能需要一些调整。

使用这些方法，我们可以确定消息缓冲区的起始位置，从核心转储中读取它，并使用 Lua 打印出来。我将所有内容放入了一个名为 `msgbuf.lua` 的脚本中：

```lua
msgbuf = lldb.target:FindFirstGlobalVariable(“msgbufp”)

msgbuf_start = msgbuf:GetChildMemberWithName(“msg_rseq”):GetValue()
msgbuf_end = msgbuf:GetChildMemberWithName(“msg_wseq”):GetValue()
unread_len = msgbuf_end - msgbuf_start

msgbuf_addr = msgbuf:GetChildMemberWithName(“msg_ptr”)
:Dereference()
:GetLoadAddress() + msgbuf_start
msgbuf_ptr = lldb.process:ReadMemory(msgbuf_addr, unread_len, lldb.SBError())

print(“Unread portion of the kernel message buffer:”)
print(msgbuf_ptr)
```

在 lldb 会话中运行这个脚本，我们得到以下输出：

```lua
(lldb) command script import ./msgbuf.lua
Unread portion of the kernel message buffer:
panic: Assertion !tcp_in_hpts(tp) failed at /usr/src/sys/netinet/tcp_subr.c:2432
cpuid = 2
time = 1706644478
KDB: stack backtrace:
db_trace_self_wrapper() at db_trace_self_wrapper+0x2b/frame 0xfffffe0047d2f480
vpanic() at vpanic+0x132/frame 0xfffffe0047d2f5b0
panic() at panic+0x43/frame 0xfffffe0047d2f610
tcp_discardcb() at tcp_discardcb+0x25b/frame 0xfffffe0047d2f660
tcp_usr_detach() at tcp_usr_detach+0x51/frame 0xfffffe0047d2f680
sorele_locked() at sorele_locked+0xf7/frame 0xfffffe0047d2f6b0
tcp_close() at tcp_close+0x155/frame 0xfffffe0047d2f6e0
rack_check_data_after_close() at rack_check_data_after_close+0x8a/frame 0xfffffe0047d2f720
rack_do_fin_wait_1() at rack_do_fin_wait_1+0x141/frame 0xfffffe0047d2f7a0
rack_do_segment_nounlock() at rack_do_segment_nounlock+0x243b/frame 0xfffffe0047d2f9a0
rack_do_segment() at rack_do_segment+0xda/frame 0xfffffe0047d2fa00
tcp_input_with_port() at tcp_input_with_port+0x1157/frame 0xfffffe0047d2fb50
tcp_input() at tcp_input+0xb/frame 0xfffffe0047d2fb60
ip_input() at ip_input+0x2ab/frame 0xfffffe0047d2fbc0
netisr_dispatch_src() at netisr_dispatch_src+0xad/frame 0xfffffe0047d2fc20
ether_demux() at ether_demux+0x17a/frame 0xfffffe0047d2fc50
ether_nh_input() at ether_nh_input+0x39f/frame 0xfffffe0047d2fca0
netisr_dispatch_src() at netisr_dispatch_src+0xad/frame 0xfffffe0047d2fd00
ether_input() at ether_input+0xd9/frame 0xfffffe0047d2fd60
vtnet_rxq_eof() at vtnet_rxq_eof+0x73e/frame 0xfffffe0047d2fe20
vtnet_rx_vq_process() at vtnet_rx_vq_process+0x9c/frame 0xfffffe0047d2fe60
ithread_loop() at ithread_loop+0x266/frame 0xfffffe0047d2fef0
fork_exit() at fork_exit+0x82/frame 0xfffffe0047d2ff30
fork_trampoline() at fork_trampoline+0xe/frame 0xfffffe0047d2ff30
--- trap 0, rip = 0, rsp = 0, rbp = 0 ---
KDB: enter: panic
```

Lua 已经帮助我们扩展了缓冲区中的控制字符，从而在消息缓冲区中提供了良好格式的输出。

## 更好的调试功能

在内核调试方面，lldb 还是个新手，且在 Lua 环境中仍有许多功能不可用，但现有的功能足以使其成为一个有用的工具。gdb 资深用户可能会对这些示例的价值存疑，毕竟 lldb 的语法复杂许多，可能看起来更像是为了改变而改变。

lldb 及其内置的 Lua 的一个重要价值在于它随 FreeBSD 发行版一同分发。lldb Lua 拥有与 FreeBSD 兼容自由许可协议，从 2024 年初开始，CURRENT 构建版本中默认启用了它。这让内核开发人员和问题排查人员可以使用 lldb Lua 编写脚本，并提供给用户进行分析。

kgdb 很早就支持 gdb 脚本，但这种脚本语言并不友好。而 Lua 虽然有些奇怪，但在许多环境中都很常见，并且是 FreeBSD 启动加载程序的一部分。我编写了个工具，能从崩溃的内核镜像中提取 TCP 日志文件——主要的难题在于如何获取内存。只要获取了数据，将其创建并写入文件则是轻而易举的事。

内核转储包含所有信息，其中可能包含敏感数据。一种合理的脚本语言使开发人员可以提供脚本，从内核镜像中提取进一步的调试信息，无需移动大型核心转储文件，也无需担心让陌生人接触可能的敏感信息。

---

**Tom Jones** 是一位 FreeBSD 提交者，对保持网络栈的高效运行充满兴趣。
