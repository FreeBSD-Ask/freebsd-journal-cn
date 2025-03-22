# 内核开发技巧

- 原文链接：[Kernel Development Recipes](https://freebsdfoundation.org/wp-content/uploads/2021/11/Kernel_Development_Recipes.pdf)
- 作者：**MARK JOHNSTON**

我经常看到的一个问题是类似于“如何开始操作系统内核开发？”这样的问题。这个问题很难普遍回答，但有一个更简单的方面是，“如何为自己设置一个（高效的）构建和测试内核更改的环境？”换句话说，虽然提交第一个内核补丁是一个重要的里程碑，但一个频繁的贡献者可能会处理多个不同的补丁，测试补丁（可能是从其他开发者那里获得的），通过二分查找找出回归的原因，或者调试内核崩溃。拥有一个能够最小化等待时间，避免不断调整配置选项或 shell 脚本的工作流程非常重要。

FreeBSD 是个庞大且历史悠久的项目（比一些开发者还要老），它的目标是支持各种硬件平台（从掌中宝般的小型系统到拥有 TB 级内存的大型多核服务器）。因此，它没有一种适用于所有任务的通用方法。然而，快速的编辑 - 编译 - 测试循环的好处是普遍适用的；本文旨在介绍一些技巧，可以减少许多内核开发任务的摩擦。

以下步骤假设你使用的是 FreeBSD 主机和兼容 POSIX 的 shell（例如 /bin/sh）。虽然当前项目没有提供许多用于构建、启动和测试内核更改的现成脚本，但这里的一些建议可以融入到你的开发环境中，或者集成到由多个开发者共享的 CI 系统中。

## Git 工作树

在开始任何 FreeBSD 的工作之前，你需要一个 `src` 仓库的副本：

```sh
$ git clone https://git.freebsd.org/src.git freebsd
```

在处理多个任务时，拥有多个 FreeBSD 源代码副本将会非常有用。如果只有一个副本，当当前分支的构建正在运行时，如果你想切换到另一个分支进行其他工作的操作就会变得很尴尬，或者切换分支会更新源代码文件的时间戳，从而不必要地减慢未来的增量构建速度。

当然，保持多个仓库副本是可行的，但更好的解决方案是使用 git 工作树，它允许你从单一的克隆中检出多个分支。这样不仅减少了磁盘空间的使用，还确保了你的所有工作都包含在仓库的一个副本中。例如，创建工作树用于频繁访问的稳定和发布分支是非常有用的。

```sh
$ cd freebsd
$ git worktree add dev/stable/13 origin/stable/13
$ git worktree add dev/releng/13.0 origin/releng/13.0
$ git worktree add dev/stable/12 origin/stable/12
$ git worktree add dev/releng/12.2 origin/releng/12.2
...
```

请注意，工作树可以位于克隆之外，例如：

```sh
$ git worktree add ../freebsd-stable/13 origin/stable/13
```

在处理一个较大的项目时，为该分支保留一个专门的工作树是非常有用的。

## 快速构建和启动自定义内核

假设你写了一个小的、简单的内核补丁，并希望在提交审查之前进行一些基本的测试。构建 FreeBSD 内核的标准命令是众所周知的：

```sh
$ cd freebsd
$ make buildkernel
```

这将执行一次干净的、单线程的内核构建。通常，更希望利用尽可能多的 CPU，因此可以考虑将 `-j $(sysctl -n hw.ncpu)` 添加到 `make(1)` 的参数中：这将并行运行与系统中的 CPU 数量相等的构建任务。待从特定源路径构建了内核，通常不需要每次测试更改时都重建每个单独的源文件——增量重建就足够了。要请求增量构建，可以将 `-DKERNFAST` 标志添加到 `make(1)` 命令中：

```sh
$ make -j $(sysctl -n hw.ncpu) -DKERNFAST buildkernel
```

默认情况下，`buildkernel` 会重建内核以及所有随之而来的内核模块，对于运行 GENERIC 内核的 amd64 系统来说，可能会重建多达 822 个模块。许多这些模块还会链接到内核中（取决于正在使用的内核配置文件），因此，`buildkernel` 可能会花费大量时间重建那些永远不会被加载的模块。可以使用 `MODULES_OVERRIDE` 变量来覆盖这一行为；通过 `MODULES_OVERRIDE` 进行构建时，将只构建内核和由该变量值指定的模块，而不是构建所有模块。例如，要仅与模块 `tmpfs.ko` 和 `nullfs.ko` 一起构建内核，请运行以下命令：

```sh
$ make -j $(sysctl -n hw.ncpu) -DKERNFAST \
 MODULES_OVERRIDE="tmpfs nullfs" buildkernel
```

如果你知道你的测试只需要少数几个模块，那么通过设置 `MODULES_OVERRIDE` 可能会加快构建时间，因为这可以大大减少在构建过程中执行的文件系统访问次数。例如，如果源代码树位于慢速磁盘上，或者远程访问并通过 NFS 访问，那么使用选项 `MODULES_OVERRIDE` 可能节省大量时间。

现在，测试内核已经准备好，可以进行测试了。测试更改的过程很大程度上依赖于更改的性质；例如，开发 WiFi 驱动程序的开发人员与开发在新平台上启动 FreeBSD 或内核内存分配器的开发人员将有不同的测试工作流。然而，出于许多目的，简单的 bhyve 虚拟机（VM）就足够了。

FreeBSD 项目提供了 VM 镜像，这些镜像对于测试内核更改非常方便。虽然当然可以通过本地构建 VM 镜像，例如使用 release(7) 脚本，但预构建的镜像非常方便：

```sh
$ IMAGE="FreeBSD-14.0-CURRENT-amd64.raw.xz"
$ URL="https://ftp.freebsd.org/pub/FreeBSD/snapshots/VM-IMAGES/14.0-CURRENT/amd64/Latest"
$ fetch -o "/tmp/${IMAGE}" "${URL}/${IMAGE}"
$ unxz "/tmp/${IMAGE}"
$ sudo pkg install -y bhyve-firmware
$ sudo sh /usr/share/examples/bhyve/vmrun.sh -E -d \
 "/tmp/${IMAGE%.xz}" myvm
```

这将使用 bhyve 启动快照镜像，虚拟机名称为“myvm”。然而，虚拟机将运行与快照一起提供的内核。有几种方法可以更新镜像的内核。例如，可以将源代码树复制或挂载到已启动的虚拟机中，并在虚拟机内构建一个新的内核。然而，这样会很慢，除非虚拟机分配大量的 CPU 和 RAM，并且会面临导出源代码树的后勤问题。另一种方法是将磁盘镜像挂载到主机上，直接安装一个新内核，但这需要一些同步，以确保虚拟机和主机不会同时挂载镜像。

另一种方法是提供一个第二个磁盘，只包含自定义内核和模块，并将虚拟机配置为从该磁盘启动。这样的磁盘镜像可以快速创建，并且不需要主机上的特殊权限。首先，可以使用以下命令来创建这样的镜像：

```sh
$ cd /usr/src
$ make buildkernel
$ make installkernel -DNO_ROOT DESTDIR=/tmp/kernel
$ cd /tmp/kernel
$ makefs -B little -S 512 -Z -o version=2 /tmp/kernfs METALOG
$ rm -f /tmp/kernfs.raw
$ mkimg -s gpt -f raw -S 512 -p freebsd-ufs/kern:=/tmp/kernfs \
 -o /tmp/kernfs.raw
```

这些 make 命令会构建并安装一个内核到 `/tmp/kernel`，而无需 root 权限。这也会在 `/tmp/kernel/METALOG` 创建一个 mtree(8) 清单，供 makefs(8) 用于构建一个小型文件系统。最后，mkimg(1) 会向文件系统添加一个 GPT，使其可供 FreeBSD 引导加载程序访问。

现在，我们可以使用额外的磁盘再次启动虚拟机：

```sh
$ sudo sh /usr/share/examples/bhyve/vmrun.sh -E -d /tmp/kernfs.raw \
 -d "${IMAGE%.xz}" myvm
```

这仍然会从原始镜像启动内核。然而，可以通过在 `/boot/loader.conf` 中添加以下行，配置引导加载程序从 `kernfs.raw` 加载内核：

```sh
kernel="disk0p1:/boot/kernel"
```

这里的“disk0p1”指的是 disk0 的第一个分区，disk0 是在 vmrun.sh 命令行参数中列出的第一个磁盘，在这个例子中是 `/tmp/kernfs.raw`。进行此更改并重启后，虚拟机应该会启动到自定义内核。

默认情况下，自定义内核的文件系统不会自动挂载，这意味着像 kldload(8) 这样的工具将无法自动加载与自定义内核对应的内核模块。为了解决这个问题，可以将以下行添加到相应的系统配置文件中：

```sh
/boot/loader.conf:
# 确保 nullfs 可用。
nullfs_load="YES"
/etc/sysctl.conf:
# 修正内核链接器使用的路径。
kern.bootfile=/boot/kernel/kernel
kern.module_path=/boot/kernel
/etc/fstab:
# 将自定义内核文件系统挂载到 /boot/kernel。
/dev/gpt/kern /mnt ufs ro 0 0
/mnt/boot/kernel /boot/kernel nullfs ro 0 0
```

最后，可以考虑将 `autoboot_delay=1` 添加到 `/boot/loader.conf` 中：这将引导延迟从十秒减少到一秒，当频繁重启时，这会带来相当大的帮助。

虽然设置过程有点复杂，但只需要执行一次，现在我们就有了一个快速启动新构建的内核的方法！在虚拟机运行时可以进行内核构建，重启虚拟机会导致加载并运行最新的内核构建。在创建更复杂的测试环境时，可能希望本地构建基础虚拟机镜像，在这种情况下，上述配置更新可以实现自动化。

## 调试自定义内核

使用 bhyve(8) 的一个好处是可以向主机提供 GDB 协议存根。QEMU 也有类似的功能。使用这个功能，主机系统可以在客户机内核上运行调试器。由于我们的自定义内核是在主机上构建的，因此此功能非常容易使用。`vmrun.sh` 目前不支持启用 GDB 存根，但可以通过原始的 bhyve(8) 调用来启用它：

```sh
# bhyve -c 1 -m 512M -H -A -P -G :1234 \
 -s 0:0,hostbridge \
 -s 1:0,lpc \
 -s 2:0,virtio-blk,kernfs.raw \
 -s 3:0,virtio-blk,FreeBSD-14.0-CUREENT-amd64.raw \
 -l com1,stdio \
 -l bootrom,/usr/local/share/uefi-firmware/BHYVE_UEFI.fd \
 myvm
```

在这里，`-G :1234` 参数指示 bhyve(8) 在端口 1234 上监听来自调试器的连接。启动虚拟机时，bhyve(8) 可以选择暂停，等待连接后再启动内核；这对于调试内核启动过程中的早期问题非常有用。要启用此功能，可以指定 `-G w:1234`。

当虚拟机正在运行（或等待连接）时，可以使用 `kgdb` 程序（与 gdb 软件包一起安装）通过端口 1234 连接到客户机：

```c
$ kgdb -q /tmp/kernel/usr/lib/debug/boot/kernel/kernel.debug
Reading symbols from /tmp/kernel/usr/lib/debug/boot/kernel/kernel.debug...
(kgdb) target remote localhost:1234
Remote debugging using localhost:1234
cpu_idle_acpi (sbt=432162053) at
/usr/home/markj/src/freebsd/sys/x86/x86/cpu_machdep.c:551
551 atomic_store_int(state, STATE_RUNNING);
(kgdb) set solib-search-path /tmp/kernel/usr/lib/debug/boot/kernel
Reading symbols from
/tmp/kernel/usr/lib/debug/boot/kernel/nullfs.ko.debug...
(kgdb) bt
#0 cpu_idle_acpi (sbt=432162053) at
/usr/home/markj/src/freebsd/sys/x86/x86/cpu_machdep.c:551
#1 0xffffffff81096ccf in cpu_idle (busy=0) at
/usr/home/markj/src/freebsd/sys/x86/x86/cpu_machdep.c:668
#2 0xffffffff80c62b41 in sched_idletd (dummy=<optimized out>,
dummy@entry=0x0 <nullfs_init>)
 at /usr/home/markj/src/freebsd/sys/kern/sched_ule.c:2952
#3 0xffffffff80be838a in fork_exit (callout=0xffffffff80c62660
<sched_idletd>, arg=0x0 <nullfs_init>,
 frame=0xfffffe0001141f40) at
/usr/home/markj/src/freebsd/sys/kern/kern_fork.c:1088
#4 <signal handler called>
(kgdb)
```

此时，虚拟机已暂停，等待调试器的命令。可以使用 `continue` 命令恢复虚拟机，按下调试器窗口中的 ctrl-C 会再次暂停虚拟机。这一功能对于调试内核死锁和启动时问题非常有用。也可以在客户机内核崩溃后附加调试器，从而无需配置和恢复内核转储就能轻松检查线程和局部变量。

## 测试自定义内核

现在，我们已经拥有了一个相当快速的循环，用于增量测试 FreeBSD 内核的更改，并且拥有了一些调试工具来解决问题。此时，某些手动测试可能表明补丁是正确的并且准备好进行审查。不过，为了增加对补丁的信心，运行一些自动化测试可能会有帮助；FreeBSD 内核是一个庞大的单体代码库，总是有潜在的未预见回归问题。一个好的起点是 FreeBSD 回归测试套件。

如果你按照上述步骤设置了内核开发虚拟机，那么测试套件已经安装好了；使用以下命令运行它：

```sh
# cd /usr/tests
# kyua -v test_suites.FreeBSD.allow_sysctl_side_effects=1 test
```

`allow_sysctl_side_effects` 标志启用了一些依赖于能够修改全局 sysctl 值的测试，在专用虚拟机中这是完全可以接受的。如果某些测试依赖于第三方端口（如 Python），则它们会被跳过。运行完测试后，可以使用以下命令查看结果总结（包括跳过的测试）：

```sh
# kyua report
```

可以设置 bhyve 虚拟机，在启动时自动运行测试套件。一种简单的方法是添加一个脚本 `/etc/rc.local`，该脚本运行测试套件，将结果打印到控制台，并关闭虚拟机。可以使用一个单独的磁盘来存储 `kyua report` 的输出，这样结果就可以轻松地在主机上恢复。

回归测试套件涵盖了大量的 FreeBSD 特性，但设计上能够快速完成。因此，尽管它可以通过相对较少的工作帮助找到错误，但可能需要进行更为密集的压力测试。FreeBSD 提供了几种进一步测试内核更改的方法。

## stress2

stress2 是一个由 Peter Holm 维护的大型压力测试套件。它包含数百个针对内核核心的文件系统和内存管理子系统的回归测试。stress2 套件包含在 FreeBSD 源代码树中，位于 `tools/test/stress2`，但并不包含在安装中。要运行这些测试，假设测试系统中有源代码树，可以运行：

```sh
# cp -R ${SRCDIR}/tools/test/stress2 /tmp/stress2
# pw user add stress2
# cd /tmp/stress2
# echo stress2 | make test
```

stress2 套件至少需要几 GB 的 RAM 和一个大磁盘。完成这些测试可能需要几天时间，但它是测试内核系统性更改的绝佳方式。各个单独的测试位于 `misc` 子目录下，可以直接运行。

## syzkaller

最后，syzkaller 已成为一种有效的工具，用于测试从系统调用接口可达的内核部分。作为一个模糊测试工具，它对于证明更改的正确性并不特别有用，但它非常擅长触发不常执行的错误路径，因此可以帮助验证错误处理代码，这些代码可能很难触发。它还有效地引发竞争条件。

syzkaller 的详细概述出现在之前的 [FreeBSD Journal 文章](https://freebsdfoundation.org/wp-content/uploads/2021/01/Kernel-Fuzzing.pdf)中。设置一个 syzkaller 实例是一个相对复杂的任务。可以参考 [syzkaller 仓库](https://github.com/google/syzkaller/tree/master/docs/freebsd#readme) 中的文档，了解如何设置一个 FreeBSD 主机来运行 syzkaller（它通过 QEMU 或 bhyve 虚拟机执行模糊测试）。

一种自动化许多设置步骤的替代方法是使用 Bastille 模板。[Bastille](https://bastillebsd.org/) 是一个在 FreeBSD 上部署和管理 jail 系统的工具；Bastille [模板](https://github.com/markjdb/bastille-syzkaller) 允许在运行中的 jail 中运行代码并修改配置。运行 syzkaller 的 Bastille 模板已经可用。要使用它，首先安装 Bastille 并基于 FreeBSD 13.0 创建一个精简的、基于 VNET 的 jail：

```sh
# pkg install bastille
# bastille bootstrap 13.0-RELEASE
# bastille bootstrap https://github.com/markjdb/bastille-syzkaller
# bastille create -V syzkaller 13.0-RELEASE 0.0.0.0 epair0b
```

这假设 epair0b 是 Bastille 的“上行”接口；它可以与主机接口桥接，以提供完整的网络访问。

然后，按照 bastille-syzkaller 模板 README 中记录的设置步骤，创建一个 ZFS 数据集供 syzkaller 使用，并在 syzkaller jail 中启用所需的功能。最后，可以应用该模板并启动 syzkaller，如下所示：

```sh
# bastille template syzkaller markjdb/bastille-syzkaller \
 --arg FREEBSD_HOST_SRC_PATH=${SRCDIR}
# bastille service syzkaller syz-manager onestart
```

通常，\${SRCDIR} 将是一个 git worktree，引用要构建和测试的分支；该 worktree 被通过 nullfs 挂载到 jail 中。syzkaller 测试的内核可以通过模板在 jail 根用户的主目录中安装的 build.sh 脚本重新构建：

```sh
# bastille console syzkaller
# service syz-manager onestop
# sh /root/build.sh
# service syz-manager onestart
```

此时，syzkaller 启动一个监听 8080 端口的 web 服务器，显示崩溃报告和模糊测试过程的进度。

---

自 2010 年以来 **MARK JOHNSTON** 一直是 FreeBSD 用户，并从 2013 年开始成为提交者。目前他在 FreeBSD 基金会工作，致力于提高内核的稳定性和性能，并提供代码审查。
