# 在 FreeBSD 上使用 GDB 调试

调试器用于检查系统中进程的状态。对于运行中的进程，调试器可以查看变量的值，也可以控制进程的执行。对于因 bug 而异常终止的进程，调试器可以解析内核生成的进程核心转储，查看崩溃时进程的状态。

大多数调试器都支持一组基础功能，比如查看全局和局部变量的值、生成栈回溯、通过断点中断进程执行，以及通过单步执行控制进程或线程的执行。本文重点关注现代版本的 GNU 调试器（gdb）在 FreeBSD 上支持的其他一些功能。其中部分功能仅 gdb 的最新版本（撰写本文时为 8.3）支持，另一些功能在旧版本中也可用。

要开始使用，先安装一个较新版本的 gdb。最简单的办法是运行 `pkg install gdb` 安装预编译包。也可以通过 devel/gdb port（<https://www.freshports.org/devel/gdb>）从源码构建 gdb。

## info proc 命令

`info proc` 命令可以查看进程在内存和线程之外的状态。默认情况下，`info proc` 提供基本信息，比如进程 ID 和命令行。不过，通过子命令可以获取更多信息，包括通过 `info proc files` 查看打开的文件描述符列表，以及通过 `info proc mappings` 查看活动内存映射列表。在前两个示例中，命令 `wc /usr/src/bin/ls/ls.c` 在调试器下执行。执行在 `cnt` 函数内暂停，目标文件已打开。第一个示例展示基本命令提供的信息。第二个示例展示打开的文件列表，其中包含 `ls.c` 文件描述符的偏移量，表明 `wc` 进程已读取了多少文件内容。

虽然 `info proc` 提供的所有信息也可以通过其他工具获得，比如 ps(1)（<https://www.freebsd.org/cgi/man.cgi?query=ps(1)>）和 procstat(1)（<https://www.freebsd.org/cgi/man.cgi?query=procstat(1)>），但 `info proc` 命令允许用户在调试器内访问这些信息，无需另开窗口。这些命令也可以用于在非 FreeBSD 操作系统上运行的交叉调试器查看核心转储的场景。

关于 `info proc` 命令及其子命令的更多详细信息，可以查阅《Debugging with GDB 手册》（<https://sourceware.org/gdb/current/onlinedocs/gdb/>）中的 Process Information 章节（<https://sourceware.org/gdb/current/onlinedocs/gdb/Process-Information.html>）。

示例 1：info proc

```sh
(gdb) info proc
process 85146
cmdline = '/usr/bin/wc /usr/src/bin/ls/ls.c'
cwd = '/usr/home/john'
exe = '/usr/bin/wc'
```

示例 2：info proc files

```sh
(gdb) info proc files
process 85153
Open files:
Type    Offset  Flags       Name
file    -       r---------  /usr/bin/wc
ctty    chr     -           rw-------
                            /dev/pts/20
cwd     dir     -           r---------
                            /usr/home/john
root    dir     -           r---------
                            /
        chr     0xac82      rw-------
                            /dev/pts/20
        chr     0xac82      rw-------
                            /dev/pts/20
        chr     0xac82      rw-------
                            /dev/pts/20
        file    0x63dd      r---------
                            /usr/src/bin/ls/ls.c
```

## 拦截系统调用

GDB 支持一类特殊的断点，称为捕获点（catchpoint）。捕获点允许用户在执行期间发生某些类型的事件时暂停执行。GDB 支持的捕获点类型之一是系统调用捕获点。系统调用捕获点在进入和退出系统调用时暂停执行。

系统调用捕获点通过 `catch syscall` 命令创建。如果不指定参数，执行会在所有系统调用的进入和退出时暂停。可以通过向命令传入系统调用列表作为参数，定义更具体的捕获点。系统调用可以按名称或编号指定。例如，`catch syscall write` 会设置一个捕获点，在进入和退出 write(2)（<https://www.freebsd.org/cgi/man.cgi?query=write(2)>）系统调用时暂停执行。

系统调用捕获点创建后，可以用其他断点命令管理。`info breakpoints` 命令会列出捕获点及其他断点。捕获点通过 `delete` 命令删除。

示例 3 拦截 ls(1)（<https://www.freebsd.org/cgi/man.cgi?query=ls(1)>）进程的 write(2)（<https://www.freebsd.org/cgi/man.cgi?query=write(2)>）系统调用。

示例 3：捕获系统调用

```sh
% gdb -q --args /bin/ls -l /bin/sh
Reading symbols from /bin/ls...
Reading symbols from /usr/lib/debug//bin/ls.debug...
(gdb) catch syscall write
Catchpoint 1 (syscall 'write' [4])
(gdb) info breakpoints
Num     Type        Disp Enb Address    What
1       catchpoint  keep y             syscall "write"
(gdb) run
Starting program: /bin/ls -l /bin/sh
Catchpoint 1 (call to syscall write), _write () at _write.S:3
3       PSEUDO(write)
(gdb) c
Continuing.
-r-xr-xr-x  1 root  wheel  168880 Nov 17 17:38 /bin/sh
Catchpoint 1 (returned from syscall write), _write () at _write.S:3
3       PSEUDO(write)
(gdb) c
Continuing.
[Inferior 1 (process 24875) exited normally]
```

对于 FreeBSD，GDB 能识别兼容性系统调用。按名称捕获某个为旧版本提供兼容性系统调用的系统调用时，会捕获该系统调用的所有版本。例如，由于 `struct stat` 的变化，几个系统调用在 FreeBSD 12 中迁移到了新的编号。原有系统调用继续使用旧的 `struct stat` 布局，但被重命名，加上了 `freebsd11_` 前缀。

按名称捕获这类系统调用时，GDB 会同时捕获两个版本，因为应用程序可能使用任一版本。示例 4 中，捕获 fstat(2)（<https://www.freebsd.org/cgi/man.cgi?query=fstat(2)>）系统调用会为两个版本都注册捕获点。

示例 4：捕获 fstat(2)

```sh
(gdb) catch syscall fstat
Catchpoint 1 (syscalls 'freebsd11_fstat' [189] 'fstat' [551])
(gdb) info breakpoints
Num     Type        Disp Enb Address    What
1       catchpoint  keep y             syscalls "freebsd11_fstat, fstat"
```

## 调试 fork

许多程序通过 fork(2)（<https://www.freebsd.org/cgi/man.cgi?query=fork(2)>）和 vfork(2)（<https://www.freebsd.org/cgi/man.cgi?query=vfork(2)>）系统调用创建新进程。GDB 提供了多种工具来处理 fork 产生的子进程。下面的示例将使用清单 1 中的测试程序演示这些功能。

清单 1：forktest.c

```c
#include <sys/types.h>
#include <sys/wait.h>
#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int
main(void)
{
    pid_t pid, wpid;
    pid = fork();
    if (pid == -1)
        err(1, "fork");
    if (pid == 0) {
        printf("I'm in the child\n");
        exit(1);
    }
    printf("I'm in the parent\n");
    wpid = waitpid(pid, NULL, 0);
    if (wpid < 0)
        err(1, "waitpid");
    return (0);
}
```

### Fork 跟随模式

被调试进程 fork 时，GDB 必须选择继续调试哪个进程：原来的（父）进程，还是新的（子）进程。默认情况下，GDB 跟随父进程，让子进程在 fork 后自由运行，如示例 5 所示。注意，子进程会向控制台输出并退出，即便父进程在调试器中暂停。

示例 5：跟随父进程

```sh
(gdb) start
Temporary breakpoint 1 at 0x201354: file forktest.c, line 13.
Starting program: /usr/home/john/work/johnsvn/test/forktest/forktest
Temporary breakpoint 1, main () at forktest.c:13
13      pid = fork();
(gdb) n
[Detaching after fork from child process 25302]
I'm in the child
14      if (pid == -1)
(gdb) p pid
$1 = 25302
(gdb) n
15      printf("I'm in the parent\n");
(gdb) c
Continuing.
I'm in the parent
[Inferior 1 (process 25297) exited normally]
```

GDB 通过 `follow-fork-mode` 设置决定 fork 后跟随哪个进程。要改为跟随子进程而非父进程，使用“child”设置。要恢复默认行为，使用“parent”设置。该设置通过 `set follow-fork-mode` 命令更改。`show follow-fork-mode` 命令显示当前设置。示例 6 再次执行测试程序，但改为跟随子进程。

示例 6：跟随子进程

```sh
(gdb) set follow-fork-mode child
(gdb) start
Temporary breakpoint 1 at 0x201354: file forktest.c, line 13.
Starting program: /usr/home/john/work/johnsvn/test/forktest/forktest
Temporary breakpoint 1, main () at forktest.c:13
13      pid = fork();
(gdb) n
[Attaching after LWP 100857 of process 26342 fork to child LWP 101958 of process 26347]
[New inferior 2 (process 26347)]
[Detaching after fork from parent process 26342]
[Inferior 1 (process 26342) detached]
I'm in the parent
[Switching to LWP 101958 of process 26347]
main () at forktest.c:14
14      if (pid == -1)
(gdb) p pid
$1 = 0
(gdb) n
18      printf("I'm in the child\n");
(gdb) c
Continuing.
I'm in the child
[Inferior 2 (process 26347) exited with code 01]
```

### fork 时分离

除了决定 fork 后跟随哪个进程，GDB 还可以选择如何处理非跟随的进程。默认情况下，GDB 从非跟随进程分离，让它在 fork 后自由运行。可以将 `detach-on-fork` 设置改为“no”来改变这一行为。设置为“no”时，GDB 保持附加在两个进程上，并在 fork 后让两个进程都暂停。

要管理这两个进程，会使用 GDB 的多进程支持（<https://sourceware.org/gdb/current/onlinedocs/gdb/Inferiors-and-Programs.html#Inferiors-and-Programs>）。在 GDB 术语中，每个进程关联一个“inferior”。`info inferiors` 命令用于列出活动的 inferior。`inferior` 命令用于在 inferior 之间切换。来自不同进程的线程也会在 `info threads` 命令中显示。切换到不同 inferior 的线程也是切换 inferior 的一种方式。fork 后，跟随进程的 inferior 被设为当前 inferior。

示例 7 再次给出测试程序的示例。这次禁用 `detach-on-fork`，让两个进程在 fork 后都保持暂停。使用默认的 fork 跟随模式，因此 fork 后 GDB 仍关注父进程。注意，子进程停在 fork 后的第一条指令处。

示例 7：fork 后保持附加

```sh
(gdb) set detach-on-fork off
(gdb) start
Temporary breakpoint 1 at 0x201354: file forktest.c, line 13.
Starting program: /usr/home/john/work/johnsvn/test/forktest/forktest
Temporary breakpoint 1, main () at forktest.c:13
13      pid = fork();
(gdb) n
[New inferior 2 (process 26828)]
14      if (pid == -1)
(gdb) p pid
$1 = 26828
(gdb) info inferiors
Num     Description         Executable
*1      process 26823       /usr/home/john/work/johnsvn/test/forktest/forktest
2       process 26828       /usr/home/john/work/johnsvn/test/forktest/forktest
(gdb) inferior 2
[Switching to inferior 2 [process 26828]
(/usr/home/john/work/johnsvn/test/forktest/forktest)]
[Switching to thread 2.1 (LWP 101425 of process 26828)]
Reading symbols from
/usr/home/john/work/johnsvn/test/forktest/forktest.debug...done.
Reading symbols from /usr/lib/debug/lib/libc.so.7.debug...done.
Reading symbols from /usr/lib/debug/libexec/ld-elf.so.1.debug...done.
#0  _fork () at _fork.S:3
3       PSEUDO(fork)
Warning: the current language does not match this frame.
(gdb) n
main () at forktest.c:14
14      if (pid == -1)
(gdb) p pid
$2 = 0
(gdb) info threads
Id   Target Id                          Frame
1.1  LWP 100970 of process 26823        main () at forktest.c:14
*2.1 LWP 101425 of process 26828        main () at forktest.c:14
(gdb) c
Continuing.
I'm in the child
[Inferior 2 (process 26828) exited with code 01]
(gdb) thread 1.1
[Switching to thread 1.1 (LWP 100970 of process 26823)]
#0 main () at forktest.c:14
14      if (pid == -1)
(gdb) c
Continuing.
I'm in the parent
[Inferior 1 (process 26823) exited normally]
```

### 捕获 fork

GDB 还提供了一组用于调试 fork 进程的工具：一组与 fork 相关事件的捕获点。`catch fork` 命令为非 vfork(2)（<https://www.freebsd.org/cgi/man.cgi?query=vfork(2)>）的 fork 调用安装捕获点。捕获点在跟随进程从 fork 返回时触发。`catch vfork` 命令为 vfork(2) 的 fork 调用安装捕获点。最后，`catch exec` 命令为 exec 系列系统调用的返回安装捕获点。示例 8 跟随一个 shell 进程，该进程 fork 出一个子进程来执行命令。

示例 8：捕获 fork 和 exec

```sh
% gdb -q /bin/sh
(gdb) catch fork
Catchpoint 1 (fork)
(gdb) catch exec
Catchpoint 2 (exec)
(gdb) set follow-fork-mode child
(gdb) run
Starting program: /bin/sh
$ ls -l /dev/null; exit
Catchpoint 1 (forked process 27644), _fork () at _fork.S:3
3       PSEUDO(fork)
(gdb) c
Continuing.
[Attaching after LWP 100469 of process 27639 fork to child LWP 101734 of
process 27644]
[New inferior 2 (process 27644)]
[Detaching after fork from parent process 27639]
[Inferior 1 (process 27639) detached]
process 27644 is executing new program: /bin/ls
Thread 2.1 hit Catchpoint 2 (exec'd /bin/ls), .rtld_start ()
at /usr/src/libexec/rtld-elf/amd64/rtld_start.S:33
33      xorq %rbp,%rbp     # 出于良好习惯，清空帧指针
(gdb) c
Continuing.
crw-rw-rw- 1 root wheel 0xf Feb 2 18:00 /dev/null
[Inferior 2 (process 27644) exited normally]
```

关于使用 GDB 调试 fork 的更多信息，请参阅 GDB 手册的 Debugging Forks（<https://sourceware.org/gdb/current/onlinedocs/gdb/Forks.html>）章节。

## 调试 C++ STL 类

对于某些数据类型，数据结构的原始布局可能与该结构在源代码中的使用和表示不一致。C++ 标准模板库（STL）类尤其如此。为了便于检查这些结构，GDB 允许 Python 脚本提供两类辅助类：pretty printer（<https://sourceware.org/gdb/current/onlinedocs/gdb/Pretty-Printing-API.html#Pretty-Printing-API>）和 xmethod（<https://sourceware.org/gdb/current/onlinedocs/gdb/Xmethods-In-Python.html#Xmethods-In-Python>）。

pretty printer 会覆盖 `print` 命令对对象的默认显示。每个 pretty printer 关联一个或多个 C++ 类，也可以关联模板类。例如，`std::vector` 的 pretty printer 可以将 vector 内容显示为数组。

xmethod 允许 Python 脚本模拟内联 C++ 类方法的效果。在求值表达式时，GDB 会在需要时调用被调试程序中定义的函数来求值，包括调用 C++ 运算符重载函数。然而，如果方法被内联（模板类中很常见），则没有供 GDB 调用的独立函数符号。结果是，在表达式中尝试使用这些函数或运算符会失败。xmethod 可以弥合这一差距。例如，xmethod 可以为 `std::vector` 对象提供 `operator[]`，让用户能以原始 C++ 源代码中相同的语法直接索引 vector。

FreeBSD 使用的 LLVM C++ 库并未提供一组为常用 C++ STL 类提供 pretty printer 和 xmethod 的 Python 脚本。不过，在 <https://github.com/bsdjhb/libcxx-gdbpy> 可以找到一组初始脚本。撰写本文时，这些脚本对 `std::string`、`std::unique_ptr` 和 `std::vector` 提供了有限的支持。示例 9 和示例 10 对比了未安装和安装这些脚本时检查 `std::vector<int>` 的效果。这些 Python 脚本在 devel/gdb port 的 8.2.1_1 及更新版本中默认包含。

示例 9：未使用 Python 脚本的 std::vector

```sh
(gdb) p vector
$1 = {<std::__1::__vector_base<int, std::__1::allocator<int> >> =
{<std::__1::__vector_base_common<true>> = {<No data fields>}, __begin_ =
0x800244000,
__end_ = 0x80024400c,
__end_cap_ = {<std::__1::__compressed_pair_elem<int*, 0, false>> = {
__value_ = 0x800244010},
<std::__1::__compressed_pair_elem<std::__1::allocator<int>, 1, true>> =
{std::__1::allocator<int>> = {<No data fields>}, <No data fields>}, <No data fields>}}, <No data fields>}
(gdb) p vector[1]
Could not find operator[].
```

示例 10：使用 Python 脚本的 std::vector

```sh
(gdb) p vector
$1 = std::vector of length 3 = {4, 5, 6}
(gdb) p vector[1]
$2 = 5
```

## 使用系统根目录进行交叉调试

来自 Ports 的 GDB 包默认编译为交叉调试器。这意味着它可以检查其他架构和其他操作系统的二进制文件和核心转储。例如，可以在较快的 x86 主机上用 GDB 进程检查来自嵌入式 FreeBSD ARM 系统的进程核心转储。另一个用例是通过串口连接调试远程机器的内核。

交叉调试自包含二进制文件（如静态二进制文件或单体内核）时，GDB 能从该二进制文件中找到所需的所有信息。但调试依赖其他二进制文件（如共享库或内核模块）的二进制文件时，GDB 需要能找到这些其他二进制文件。通常 GDB 会在运行它的主机上查找这些二进制文件，这在调试本机进程或核心转储时工作良好。但交叉调试时，GDB 需要能访问这些附加二进制文件。这可以通过系统根目录（system root）解决。

系统根目录是系统中共享二进制文件在备用目录中的副本。系统根目录通常可以包含完整的系统安装映像。如果系统根目录包含编译器使用的头文件和库，交叉编译器就能用系统根目录为备用系统编译二进制文件。GCC 和 clang 都用 `--sysroot` 标志指示编译器在系统根目录中查找头文件和库。在 GDB 中，通过将 `sysroot` 变量设为系统根目录的路径来指定。GDB 会在这个系统根目录下而非主机的根文件系统中查找共享库和内核模块。

举个例子，假设在树莓派上运行的进程崩溃生成了核心转储。可以把树莓派上的 SD 卡取出插入 x86 机器。然后挂载 SD 卡，在 x86 机器上检查核心转储。只需告诉 GDB 把 SD 卡的挂载点作为系统根目录即可。示例 11 用此方法在 x86 主机上查看 ARM 核心转储。这里树莓派的 SD 卡挂载在 **/mnt**。

示例 11：在 x86 主机上检查 ARM 核心转储

```sh
> gdb -q sigframe
Reading symbols from sigframe...Reading symbols from
/mnt/home/john/work/johnsvn/test/sigframe/sigframe.debug...done.
done.
(gdb) set sysroot /mnt
(gdb) core-file sigframe.core
[New LWP 100086]
Core was generated by `./sigframe'.
Program terminated with signal SIGABRT, Aborted.
#0 thr_kill () at thr_kill.S:3
3       RSYSCALL(thr_kill)
(gdb) info sharedlibrary
From        To          Syms Read    Shared Object Library
0x20092000  0x201e516c  Yes          /mnt/lib/libc.so.7
0x20016000  0x20030ef4  Yes          /mnt/libexec/ldelf.so.1
```

如果在加载核心文件之前忘记设置 `sysroot` 变量，也可以在核心文件加载后再设置。GDB 会在 `sysroot` 变更后自动在新的系统根目录下查找共享库。

调试远程 FreeBSD 内核时，`sysroot` 变量也有效。GDB 会在系统根目录下查找内核模块及其关联的调试信息。即使目标机器与主机架构相同，但运行的操作系统或操作系统版本不同，这一功能也很有用。•

---

**JOHN BALDWIN** 是一名系统软件开发者。他直接为 FreeBSD 操作系统提交更改已有十九年，涉及内核的多个部分（包括 x86 平台支持、SMP、各种设备驱动和虚拟内存子系统）以及用户态程序。除了写代码，John 还曾任职 FreeBSD 核心团队和发布工程团队。他也为 GDB 调试器和 LLVM 做过贡献。John 与妻子 Kimberly 和三个孩子 Janelle、Evan、Bella 居住在加州康科德。
