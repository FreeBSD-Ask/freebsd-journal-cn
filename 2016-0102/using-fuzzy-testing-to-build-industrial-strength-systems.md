# 用模糊测试打造工业级强度的系统

- 原文：[Using Fuzzy Testing to Build Industrial-Strength Systems](https://freebsdfoundation.org/wp-content/uploads/2016/03/Using-Fuzzy-Testing-to-Build-Industrial-Strength-Systems.pdf)
- 作者：**Peter Holm**

stress2 压力测试套件是提升 FreeBSD 内核质量的工具，它能在你增加或更新内核代码时帮助你暴露设计与实现问题。它于 2009 年首次提交到 svn，但在此之前已经使用了多年。2014 年，借助它生成了 100 多份问题报告。

## 我是如何开始做压力测试的

上世纪 90 年代某一天，我所在公司的支持部门让我去看看客户的计算机，这台机器每次运行某个厂商的数据库应用都会崩溃。令我惊讶的是，这台机器在常用的系统调用里发生了 panic。逐行跟踪代码之后，我发现触发问题的不过是简单的栈破坏缺陷，来自用户态程序。有两点从此深深印在我的脑海里：a）虽然见到客户的高层很有意思，但如果能在办公室就发现这类错误就更妙了；b）普通的用户态缺陷竟然让计算机崩溃了！就在当天，我写了第一个压力测试程序——syscall fuzzer——它让许多人忙活了好几天。

## 测试设计

FreeBSD 项目中广泛使用的测试是 `buildworld`。这是个不错的测试，但连续跑几次之后继续跑也没有什么收益。Stress2 在大多数测试用例上采用不同策略，每次运行的方式都不同——不同的运行时长、不同的线程数、不同的 VM 压力。这种策略的好处是覆盖面更广。缺点是重现错误稍显困难，但相比收益这点麻烦几乎可以忽略不计。

许多场景用多线程运行同一测试，对共享资源的锁做压力测试。有些测试会同步线程，但另一些不会，希望借此获得更广的覆盖。Stress2 的基石是大多数测试都会附加 VM 压力，以触发内核中的等待点。大量测试在不加 VM 压力时检测不到任何问题，但一旦加上 VM 压力，panic 和死锁就纷纷现身。

少数测试依赖模糊测试。**mmap(2)** 系列测试就是一例。

## Stress2 的典型工作日

我收到 FreeBSD 某位提交者发来的补丁，旨在修复某个 panic。这个问题有时可由现有的某些测试场景触发，有时则需要写新测试。一旦能在合理时间内复现问题，就可以测试补丁了。验证修复之后，我通常会运行所有其它场景，以防修复带来副作用——这种情况出奇常见。补丁在提交前经历几次修订并不少见。高产的日子，我能走完三到四个修订。我尽量让 Bug 报告包含足够的信息，以确保补丁快速更新。一份典型的 Bug 报告长这样：<https://people.freebsd.org/~pho/stress/log/kostik833.txt>。

一次完整测试通常需要 24 小时以上，我才会对修复有信心。

在编写新测试场景时，副产品往往是新测试用例，触发所报告问题之外的其它问题。这些测试用例会标记为待提交，即便可能还是 WIP。这里的理念是：能让内核崩溃的测试就是好测试。

从这个例子可以看出，这是团队协作。我和一群极具天赋的人共事，作为团队我们的产出惊人。

## 内存泄漏

内存泄漏会定期出现。我通过观察 `vmstat -m` 和 `-z` 来检查。我在测试期间运行一个脚本自动完成这件事：**stress2/tools/vmstat.sh**

## 测试硬件

我观察到不同类型的硬件会有差别，因为有些问题只能在某种类型上重现。磁盘速度也有影响，CPU 数量也是如此。最近我又体会到了这一点的重要性。我有待评估的补丁，它通过了所有测试。然而其他人发现，在配置 1GB RAM 的 6 核机器上持续执行 `-j7 buildworld` 会触发 panic。许多测试必须配置 swap。没有 swap 时，很难把压力调到合适的程度。也就是说，没有 swap 盘，VM 压力过大会触发 OOM，杀掉进程。我在真实硬件上同时测试 i386 和 amd64，所有测试主机都使用串行控制台，以便记录输出。

## 示例

下面描述几个我觉得特别有意思的测试场景。它们都位于 **stress2/misc** 目录。该目录下所有测试的共同点是：都是完整、自包含的测试场景，以 shell 脚本形式实现。其中大多数是回归测试；也就是说，它们曾触发过 panic 或死锁。这些测试可以单独运行。大多数（但并非全部）测试使用基于内存的文件系统进行测试。主要好处是文件系统处于初始一致状态。

下面展示 **stress2/misc** 中测试的样例：

```sh
#!/bin/sh
# 出现过 "panic: not suspended thread 0xc674c870"。
for i in `find /proc ! -type d`; do
    dd if=/dev/random of=$i > /dev/null 2>&1
    dd if=$i of=/dev/null > /dev/null 2>&1
done
```

**stress2/misc** 中的许多测试都使用了 **stress2/testcases** 中更通用的压力测试程序。

### trim6.sh

为某个目的编写的测试用例常常也能捕获其它问题。这个测试用例最初是为了解决一个问题：在启用了 TRIM 选项的文件系统上删除大文件。但后来也能在文件创建期间触发死锁和 panic。本例中的测试很简单：向一块高速（SSD）磁盘写入非常大的文件。来自 r287361 提交日志：通过快速扩展文件，可能产生过量 D_NEWBLK 类型的依赖（即 D_ALLOCDIRECT 和 D_ALLOCINDIR），从而耗尽 kmem。

### crossmp.sh

并行化的另一高效示例是 Cross Mount Point 测试用例，它贡献了 6 份 Bug 报告。这些测试并行挂载和卸载 15 个不同的文件系统/挂载点。

### callout_reset_on.sh

我花时间最多的场景来自 pr. kern/166340，一份非常详尽的 Bug 报告：FreeBSD 9.0 下的进程会以不可中断睡眠状态挂起，且看起来没有系统调用（wchan 为空）。后来对 callout wheel 的改动会触发 panic：`Bad link elm 0xfffff80012ba8ec8 prev->next != elm`，就出现在这个场景下。

### mmap10.sh

这是一个模糊测试用例。基本思路是向系统调用传入随机值，试图找出代码中的错误。一旦跨过简单的参数校验缺失问题，更有趣的问题就会浮现。这个测试场景通过向 **mlock(2)**、**mprotect(2)** 和 **mlockall(2)** 传入随机值，触发了如下独特问题：

```sh
panic: deadlkres: possible deadlock detected for 0xcb0ea930, blocked for 1801709 ticks
panic: pmap active 0xfffff800a90cfd78
panic: vm_fault_copy_wired: page missing
panic: vm_object_backing_scan: object mismatch
panic: vm_page_dirty: page is invalid!
panic: vmspace_fork: entry 0xfffff80019793d00 eflags 50c
```

### rename3.sh

有些测试场景是其他人编写或提议的。比如 Tor Egge 编写的这个小型 rename 测试场景：“测试当接近根目录的目录被重命名时，对瞬时故障的敏感性”。它触发了多次死锁。

### isofs2.sh

这是最新的测试，非常简单。创建包含 **date(1)** 副本的 isofs 文件系统。挂载该文件系统并运行“date”副本。这会触发“panic: witness_warn”，报告见：<https://people.freebsd.org/~pho/stress/log/isofs2.txt>

### marcus5.sh

这是测试示例，是搜索另一个问题时的副产品。该测试触发了关于 VFS_SYNC() 实现方式的问题：<https://people.freebsd.org/~pho/stress/log/marcus5.txt>

### md8.sh

这是针对 vnode 后端 **md(4)** 卷上未映射未对齐 IO 的回归测试。在缓冲区“data”页对齐的情况下，测试为：

```c
read(fd, data + 512, MAXPHYS)
write(fd, data + 512, MAXPHYS)
```

## 细节

通过如下方式获取 stress2：

```sh
svnlite checkout svn://svn.freebsd.org/base/user/pho/stress2
cd stress2
make
```

这会在子目录 `testcases` 中构建一些基础测试程序。所有新开发都在 `misc` 目录进行，目前那里有约 400 个测试场景。这些测试常称作回归测试。其真正价值在于它们能对内核的不同角落做压力测试。`misc` 目录中的测试可以单独运行，也可以由 `all.sh` 脚本统一控制。例如，运行所有 **tmpfs(5)** 场景一次：

```sh
$ ./all.sh -o tmpfs*
20150907 22:02:12 all: tmpfs2.sh
20150907 22:06:11 all: tmpfs9.sh
:
```

## 结语

Stress2 是一款开发者工具，用于发现 FreeBSD 内核中的问题。它不能替代真实世界的测试，只是一个用于发现部分问题的工具。

本工作由 EMC / Isilon Storage Division 赞助。

---

**Peter Holm**（`<pho@FreeBSD.org>`）自 1999 年起就一直在 FreeBSD 中查找 Bug。Stress2 测试套件持续开发中，会根据 Bug 报告或补丁测试不断添加新测试。
