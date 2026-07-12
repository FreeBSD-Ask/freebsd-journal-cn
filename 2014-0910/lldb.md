# LLDB

- 原文标题：LLDB in FreeBSD — The Search for a New Debugger
- 作者：**Ed Maste**

FreeBSD 区别于其他开源操作系统的一个特征，是“基本系统”（base system）的概念——一个集成核心，作为整体进行开发、维护、测试和发布。基本系统的主要组件包括内核、用户态库、系统二进制文件和开发工具链——而工具链中的一个关键组件就是调试器。

GNU 调试器 GDB 长期以来一直担任基本系统的调试器。1993 年，第一个 FreeBSD 发布版就包含了 GDB 3.5。此后的每个发布版都包含某个版本的 GDB。项目跟踪 GDB 的开发超过十年，许多不同的贡献者将新版本纳入 FreeBSD 源码树。

这项工作产生了一组不断增长的更改，每次新导入的工作量都在增加。项目成员尝试将这些更改合并到上游 GDB 项目，但遭到了项目维护者的阻力。最终，这种不断增长的维护负担压垮了团队，最后一次导入是 2004 年 6 月的 GDB 6.1.1。

## GPLv3

与其他 GNU 项目一样，2007 年 GDB 的许可证更改为 GNU 通用公共许可证第 3 版（GPLv3）。GPLv3 包含一些主要 FreeBSD 贡献者和消费者难以接受的限制，迄今为止，项目一直避免在基本系统中包含任何 GPLv3 许可的代码。

## 寻找新调试器

由于基本系统中的 GDB 版本停滞不前且没有明确的前进方向，FreeBSD 显然需要一个新的调试器。出现了几个开源调试器项目，有的来自 FreeBSD 社区内部，有的来自外部。但没有一个能达到维持开发并产生可用调试器的临界规模。

随后，在 2010 年的全球开发者大会（WWDC）上，Apple 宣布他们有自己的调试器项目 LLDB。该项目于同年 6 月开源，次年成为 Apple IDE Xcode 的默认调试器。LLDB 采用伊利诺伊大学/NCSA 许可证，这是一种宽松的、类似 BSD 的许可证，与 FreeBSD 项目的许可理念完美契合。

此后，LLDB 已超越了 Apple 项目的范畴，来自 Intel 和谷歌等公司内部的开源团队，以及 FreeBSD、Debian 和其他独立开源项目都做出了重要贡献。多人参与了 LLDB 的 FreeBSD 移植，我们的计划是让它成为 FreeBSD 基本系统的标准调试器。

## LLDB 设计

LLDB 构建为 LLVM 编译器基础设施项目和 Clang 前端之上的一组模块化组件。复用 Clang 和 LLVM 组件，使得 LLDB 与其他调试器相比，能用更少的工作量和更小的代码量实现大量功能。例如，LLDB 内置了完整的 Clang 编译器，用于其表达式解析器。如果某个表达式在项目源码中可接受，LLDB 的表达式解析器也能处理，让用户能详细、自信地检查复杂的类和数据类型。LLVM 为 LLDB 提供处理器特定的支持，包括反汇编支持和 CPU 特定功能。模块化设计也为直接支持新处理器、语言和平台奠定了基础。

许多总体目标指导着 LLDB 的设计，其中许多源自 Clang 和 LLVM。为实现高性能和降低内存使用，LLDB 只解析执行操作所需的调试信息。LLVM 提供的线程化高性能类也有助于提升 LLDB 的速度（见下图）。

LLDB 致力于在各处允许自定义，让用户能定制调试体验。变量和值显示、类型格式化器、摘要信息、命令、提示符和别名都可以配置。

与 Clang 和 LLVM 一样，LLDB 在同一个调试器二进制文件中固有地支持多种 CPU 架构和平台。例如，同一个调试器可以用来本地调试 FreeBSD/amd64 应用、检查来自 FreeBSD/MIPS 系统的核心转储文件，以及远程调试在 Linux 上调试服务器下运行的应用。还可以在同一个调试器中同时运行多个调试会话。

作为调试器框架，LLDB 设计为可嵌入或被其他项目使用。IDE 和图形前端可以使用 C++ API 轻松集成并扩展 LLDB 的功能。还提供了完整的脚本 API，目前支持 Python 绑定。Python 可在 LLDB 内部的命令行中使用，用于在断点后控制执行，以及实现新命令。脚本接口也可用于外部使用；Python 脚本可以创建调试器对象，然后用它来检查和控制被调试程序的状态、求值表达式等。

## LLDB 使用

LLDB 的命令解释器采用一致的结构化语法设计。命令通常遵循“名词 动词”的模式——例如 `thread list` 或 `breakpoint set`。命令语法比 GDB 稍显冗长，长期使用 GDB 的用户可能需要一些时间适应。好处是命令集可发现且规则一致；定向自动补全可以为用户提供相关选项。与 GDB 一样，命令可以缩写为最短的唯一前缀。启动调试会话的示例可能如下：

```sh
% lldb
(lldb) target create /bin/ls
Current executable set to '/bin/ls' (x86_64).
(lldb) breakpoint set -name main
Breakpoint 1: where = ls`main + 33 at ls.c:163, address = 0x00000000004023f1
(lldb) process launch
```

LLDB 对命令别名提供强大支持，并内置了许多 GDB 命令的别名。使用 GDB 别名，上述结果可以通过以下方式实现：

```sh
% lldb /bin/ls
Current executable set to '/bin/ls' (x86_64).
(lldb) b main
Breakpoint 1: where = ls`main + 33 at ls.c:163, address = 0x00000000004023f1
(lldb) run
```

不过内置别名是有限的，GDB 命令提供的一些较为冷僻的重载功能无法通过别名使用。断点尤其如此——在 GDB 中，`breakpoint` 命令参数可以是行号、文件名、函数或地址，有时含义重叠或冲突。迁移到 LLDB 的语法并依赖子字符串匹配以使用更简洁的命令，可能是最有效的方式。

部分 GDB 和 LLDB 命令对比：

| 操作 | GDB | LLDB |
| ---- | --- | ---- |
| 无参数启动进程 | `(gdb) run` `(gdb) r` | `(lldb) process launch` `(lldb) run` `(lldb) r` |
| 带参数 args 启动进程 | `(gdb) run args` `(gdb) r args` | `(lldb) process launch -- args` `(lldb) r args` |
| 在新终端窗口中启动进程 | 无 | `(lldb) process launch --tty -- args` |
| 按 pid 附加到进程 | `(gdb) attach pid` | `(lldb) process attach --pid pid` `(lldb) attach -p pid` |
| 源码级单步步入 | `(gdb) step` `(gdb) s` | `(lldb) thread step-in` `(lldb) step` `(lldb) s` |
| 源码级单步步过函数调用 | `(gdb) next` `(gdb) n` | `(lldb) thread step-over` `(lldb) next` `(lldb) n` |
| 源码级单步步出当前函数 | `(gdb) finish` | `(lldb) thread step-out` `(lldb) finish` |
| 指令级单步步入 | `(gdb) stepi` `(gdb) si` | `(lldb) thread step-inst` `(lldb) si` |
| 指令级单步步过函数调用 | `(gdb) nexti` `(gdb) ni` | `(lldb) thread step-inst-over` `(lldb) ni` |
| 立即从当前帧返回 | `(gdb) return return expression` | `(lldb) thread return return expression` |
| 按名称在所有函数设置断点 | `(gdb) break name` `(gdb) b name` | `(lldb) breakpoint set --name name` `(lldb) br s -n name` `(lldb) b name` |
| 按文件和行号设置断点 | `(gdb) break file:line` | `(lldb) breakpoint set --file file --line line` `(lldb) br s -f file -l line` `(lldb) b file:line` |
| 按名称在所有 C++ 方法设置断点 | `(gdb) break name`（假设没有同名的 C 函数） | `(lldb) breakpoint set --method name` `(lldb) br s -M name` |
| 列出断点 | `(gdb) info break` | `(lldb) breakpoint list` `(lldb) br l` |
| 删除断点 | `(gdb) delete 1` | `(lldb) breakpoint delete 1` `(lldb) br del 1` |
| 显示参数和局部变量 | `(gdb) info args` 和 `(gdb) info locals` | `(lldb) frame variable` `(lldb) fr v` |

`target create` 命令可以在调试会话中创建多个目标，用 `target list` 显示活动目标：

```sh
(lldb) target list
Current targets:

target #0: /bin/app1 ( arch=x86-64-unknown-freebsd10.1, platform=host )
target #1: /bin/app2 ( arch=x86-64-unknown-freebsd10.1, platform=host, pid=81166, state=exited )
```

要在它们之间切换，使用 `target select <number>`。

## 断点

断点通过 `breakpoint set` 命令设置。可以在给定地址、文件名和行号、函数或方法名处设置断点，也可以在特定语言的异常上设置。断点还可以限制在指定线程或共享库中。

断点在 LLDB 中作为逻辑断点维护，然后解析为一个或多个位置。逻辑断点和每个解析出的位置都有整数标识符，以点号连接。例如，如果第三个断点匹配两个函数名并解析为两个位置，它们将被称为“3.1.”和“3.2”。断点在整个调试会话中保持活动状态，因此加载一个共享库，其中包含匹配现有断点规范的函数或方法，会向该断点添加新位置。同样，卸载共享库可能会移除断点位置。断点在卸载所有位置后保持设置状态，但处于未解析状态。

每当被调试进程停止时，LLDB 会打印相关信息：停止的线程、进程位置信息（包括地址、文件名和行号）、当前函数及其参数，以及停止原因。报告的停止原因可能是断点、观察点、信号、地址异常，或若干目标或语言特定原因之一。最后，显示当前地址处的源代码的一小部分。

```sh
* thread #1: tid = 100641, 0x00000000004023f1 ls`main(argc=1, argv=0x00007fffffffe760) + 33 at ls.c:163, name = 'ls', stop reason = breakpoint 1.1

frame #0: 0x00000000004023f1 ls`main(argc=1, argv=0x00007fffffffe760) + 33 at ls.c:163

160  #ifdef COLORLS
161          char termcapbuf[1024];  /* termcap 定义缓冲区 */
162          char tcapbuf[512];      /* capability 缓冲区 */

-> 163          char *bp = tcapbuf;

164  #endif
165
166          (void)setlocale(LC_ALL, "");

(lldb)
```

## 检查被调试程序状态

遇到断点或因其他原因停止后，LLDB 会选择最相关的线程。这将是遇到断点、收到信号、执行无效内存访问或以其他方式触发停止的线程。

`thread list` 命令列出被调试程序中的所有活动线程，用 `thread select` 选择后续命令使用的线程。

要获取栈回溯，使用 `thread backtrace` 命令，也可使用 `bt` 别名。默认显示当前线程的回溯，但可以提供不同的线程索引作为参数，或使用 `all` 显示每个线程的栈。

检查回溯时，可以使用 `frame select` 命令选择特定帧。`up` 和 `down` 别名提供相对帧选择的简短形式。选定帧后，`frame variable` 命令将显示作用域内的函数参数和局部变量。

## 控制被调试程序

LLDB 将单步进程控制命令归组到顶层 `thread` 命令下。`thread step-in` 单步执行一行源码，进入函数调用。`thread step-over` 也单步执行一行源码，但不会在函数调用内停止。`thread step-out` 继续执行直到程序从当前函数返回。`thread until <line>` 命令继续执行，直到程序到达指定源文件行或从当前函数返回。

单步命令有与 GDB 匹配的别名：`s` 或 `step` 对应 `thread step-in`，`n` 或 `next` 对应 `thread step-over`，`f` 或 `finish` 对应 `thread step-out`。

## 数据格式化器

LLDB 内置支持语言运行时和库使用的多种高级数据结构格式。这些格式化器取变量的内部表示，以方便的用户友好格式显示，如同源码中使用的那样。

对于 FreeBSD 而言，C++ 运行时库格式化器可能是最有价值的。LLDB 包含对 FreeBSD 中两个相关库的支持：libc++ 和 GNU libstdc++。

例如，启用类型格式化器时，`std::string` 默认只显示字符串内容：

```sh
(lldb) expression str
(string) $1 = "This is a string."
```

如有必要，可以禁用格式化器以查看数据结构的完整细节：

```sh
(lldb) type category disable gnu-libstdc++
(lldb) expression str
(string) $2 = {
  _M_dataplus = {_M_p  = "This is a string."}
}
```

## LLDB 脚本

LLDB 的脚本接口可以通过多种方式访问。最基本的是 `script` 命令，它调用嵌入式解释器，可用于通过一组便利变量查询当前程序状态。Python 也可用于实现新的 LLDB 命令。

LLDB 还可以在命中断点后调用脚本。然后脚本可以控制程序状态（例如，继续进程），从而实现非常复杂的断点条件。

最后，LLDB 可以完全从 Python 脚本使用，无需涉及独立的 lldb 二进制文件。脚本可以 `import lldb`，创建调试器实例和目标，设置断点、启动、单步、继续目标，以及检查变量或求值表达式。

## FreeBSD 中的 LLDB 路线图

LLDB FreeBSD 移植的持续开发工作直接在 LLDB 仓库中进行。它目前在 amd64 架构上运行良好，支持基于实时和核心转储文件的用户态调试。MIPS 架构也存在核心转储调试支持。结合谷歌正在进行的 CPU 支持工作和 FreeBSD 社区正在开展的工作，我们预计将支持 x86、ARM、MIPS 和 PowerPC CPU 架构的 64 位和 32 位版本。

2014 年谷歌编程之夏（GSoC）项目交付了 FreeBSD 内核调试支持的概念验证；在将其集成到 LLDB 之前，还需要进一步工作来完善。

LLDB 项目中正在开发的一个关键组件是远程调试桩。它允许在一台计算机上运行的 LLDB 实例访问和控制在另一台计算机上运行的进程。这对于在小型嵌入式设备上调试尤为重要，这些设备可能缺乏运行功能完整的调试器所需的内存或计算能力。调试桩最初为 Linux 实现，但移植到 FreeBSD 相对直接。

LLDB 源码树的快照会偶尔导入 FreeBSD-Current。LLDB 尚未默认构建，但可以通过在运行 `make buildworld` 之前向 **/etc/src.conf** 添加 `WITH_LLDB=yes` 来启用，如 FreeBSD Handbook 中所述。

FreeBSD 基本系统的一个复杂之处在于它不包含 Python，因此 LLDB 快照目前构建时不带脚本支持。它仍然可用，但结果是一些更有趣和高级的功能不可用。我们正在评估不同的方法来解决这个问题，可能将脚本接口迁移到运行时而非编译时选项。这将允许通过简单地安装 Python 软件包或 Port 来启用 LLDB 的所有 Python 功能。

我们预计在不久的将来将 FreeBSD 基本系统中的 Clang 和 LLVM 更新到 3.5 版本。LLDB 将同时更新，然后我们预计将默认启用其构建。
