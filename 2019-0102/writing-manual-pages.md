# 编写手册页

- 原文链接：[Writing Manual Pages](https://freebsdfoundation.org/our-work/journal/browser-based-edition/)
- 作者：**AARON ST. JOHN**

手册页（man pages）曾是 Unix 操作系统最早可用的文档形式之一。它们至今仍被用作快速参考，说明系统上已安装程序的使用方式。BSD 操作系统的源代码包含一套无可匹敌的手册页集合，与已安装程序配套。通常，手册页包含 EXAMPLES 章节——一个非常有用的工具——其中给出该命令或函数的常见用法。

创建或更新现有手册页虽简单，却对整个开源社区大有裨益。本文涵盖为大多数 BSD 系统编写新手册页或改进现有手册页的基础知识。

## 手册页分节

BSD 手册页分为多个节。文件的扩展名代表该手册页的节索引。手册按类型分入不同节。九个手册页节列于表 1。通常，某条命令、系统调用或文件只有一两份手册页。

| 节 | 描述 |
| -- | ---- |
| 1 | 用户执行的一般命令 |
| 2 | 系统调用；封装内核操作的函数 |
| 3 | 库函数 |
| 4 | 内核接口 |
| 5 | 文件格式 |
| 6 | 游戏 |
| 7 | 杂项信息 |
| 8 | 系统管理员 |
| 9 | 内核开发者 |

## 编写手册页

### 标记

多年来，手册页使用过若干不同的渲染器，例如 **groff(7)** 与 **mandoc(1)**。FreeBSD 过去曾使用 **roff(7)**、**troff(1)** 与 **man(7)** 标记语言。然而，新手册页和大多数现有手册页都使用 **mdoc(7)**——一种使用宏的语义标记语言。以 `.` 开头的行称为“宏行”。 `.` 之后的两三个字母称作宏名。宏名以大写字母开头，其余字母小写。不以 `.` 开头的行称为“文本行”，提供待打印的自由格式文本。多句书写时的常见做法是每个句子另起一行，以提升读者的清晰度。注释行以 `.\"` 开头。下面是格式正确的宏示例：

```sh
.Sh NAME
.Nm examplecommand
```

### 布局

手册页可以有许多种写法。不过，手册页通常包含特定的章节以保证一致性。首先，手册页必须按顺序包含 `.Dd`、`.Dt`、`.Os` 三个宏作为序言：

```sh
.Dd $Mdocdate$
.Dt PROGNAME section
.Os
```

`.Dd` 是日期宏。修改现有手册页时必须更新日期。日期可以通过在宏后按“月 日，年”格式手动键入来更新。`.Dt` 是文档标题宏。该宏后跟命令或函数名及其节号。最后，`.Os` 宏指定所使用的操作系统。系统可以手动指定。但推荐使用不带任何参数的 `.Os`。常用宏列表见表 2。

| 宏 | 描述 |
| -- | ---- |
| `.Dd` | 文档日期：月、日、年 |
| `.Dt` | 文档标题：TITLE section |
| `.Os` | 操作系统版本：[ system [version] ] |
| `.Sh` | 章节标题（一行） |
| `.Nm` | 命令或函数名 |
| `.Nd` | 命令或函数描述 |
| `.Op` | 可选语法参数 |
| `.Ar` | 命令参数 |
| `.Bl`、`.El` | 列表开始与列表结束 |
| `.It` | 列表项 |
| `.Pp` | 起一个文本段落 |
| `.An` | 作者姓名 |

手册页必须包含的标准章节有：

**NAME**

包含函数或命令名，和一行简明描述，说明它做什么。

**SYNOPSIS**

若是命令，写出该命令可使用的任何选项。若是程序函数，写出函数可使用的参数列表，和包含该定义的头文件。下面是 **iocage(8)** 的 SYNOPSIS 章节正确格式示例：

```sh
.Sh SYNOPSIS
.Nm
.Op Fl -help | Ar SUBCOMMAND Fl -help
.Nm
.Op Fl v | -version
.Pp
.Nm
.Cm activate
.Ar ZPOOL |
```

**DESCRIPTION**

描述中应给出该命令或函数简明而完整的说明。

**EXAMPLES**

描述每个用例作用的用例列表。一个完备的 EXAMPLES 章节至少包含一个平凡的、一个日常的、一个启发性用例。

手册页并不限于这些章节。事实上，大多数手册页还有更多章节。一些常用章节的解释见表 3。

| 常见章节 | 描述 |
| -------- | ---- |
| ENVIRONMENT | 影响操作的环境设置 |
| EXIT STATUS | 退出时返回的错误码 |
| COMPATIBILITY | 与其它实现的兼容性 |
| SEE ALSO | 相关手册页的交叉引用 |
| STANDARDS | 与 POSIX 等标准的兼容性 |
| HISTORY | 实现历史 |
| BUGS | 已知 bug |
| AUTHORS | 创建该命令或编写该手册页的人 |

### 示例

一个平凡的示例可能如下：

```sh
.Dd January 22, 2019
.Dt examplecommand 1
.Os
.Sh NAME
.Nm examplecommand
.Nd This is an example command for an example man page.
.Sh SYNOPSIS
.Nm examplecommand
.Op Fl -help
.Op -l
.Op -o Ar file
.Sh DESCRIPTION
This is formal text describing this command.
This command does this and that.
It can be used for this and that.
.Pp
These options are available:
.Pp
.Bl -tag -width ".Cm activate"
.It Fl -help
List the help screen for the command.
.It Fl l
Does this and that.
.It Fl o Ar file
Opens a file.
.El
.Sh EXAMPLES
Open a file called helloworld.txt.
.Pp
.Dl $ examplecommand -o helloworld.txt
```

输出会生成为：

```sh
examplecommand(1)               FreeBSD General Commands Manual              examplecommand(1)

examplecommand – This is an example command for an example man page.

examplecommand [--help] [-l] [-o file]

DESCRIPTION
This is formal text describing this command. This command does this and that. It can be used for
this and that.

These options are available:
--help
    List the help screen for the command.
-l
    Does this and that.
-o file
    Opens a file.

Open a file called helloworld.txt.
$ examplecommand -o helloworld.txt

FreeBSD 12.0-RELEASE           January 22, 2019           FreeBSD 12.0-RELEASE
(END)
```

下面给出一个更日常的示例，摘自 **ls(1)** 手册页：

```sh
.Dd December 1, 2015
.Dt LS 1
.Sh NAME
.Nm ls
.Nd list directory contents
.Sh SYNOPSIS
.Nm
.Op Fl -libxo
.Op Fl ABCFGHILPRSTUWZabcdfghiklmnopqrstuwxy1,
.Op Fl D Ar format
.Op Ar
.Sh DESCRIPTION
For each operand that names a
.Ar file
of a type other than
directory,
.Nm
displays its name as well as any requested,
associated information.
For each operand that names a
.Ar file
of type directory,
.Nm
displays the names of files contained
within that directory, as well as any requested, associated
Information.
```

使用 `man ls` 渲染时显示为：

```sh
LS(1)               FreeBSD General Commands Manual               LS(1)

ls — list directory contents

ls [--libxo] [-ABCFGHILPRSTUWZabcdfghiklmnopqrstuwxy1,] [-D format]
[file ...]

DESCRIPTION
For each operand that names a file of a type other than directory, ls displays its name as well as
any requested, associated information. For each operand that names a file of type directory, ls
displays the names of files contained within that directory, as well as any requested, associated
information.
```

## 参考文献

- Dzonsons, Kristaps. “man(7).”FreeBSD, 2018 年 4 月 5 日，<www.freebsd.org/cgi/man.cgi?query=man&sektion=7>。
- “Manual Pages.”FreeBSD，<www.freebsd.org/doc/en_US.ISO8859-1/books/fdp-primer/manpages.html>。
- “man page.”Wikipedia，Wikimedia Foundation，2019 年 1 月 5 日，<en.wikipedia.org/wiki/Man_page>。
- Wirzenius, Lars. “Writing Manual Pages.”Writing Manual Pages, 2010 年 11 月 10 日，<liw.fi/manpages/>。
- Dzonsons, Kristaps. “mdoc(7).”FreeBSD，2018 年 7 月 28 日，<www.freebsd.org/cgi/man.cgi?query=mdoc&sektion=7&manpath=freebsd-release-ports>。
- Toth, Peter，与 Brandon Schneider. “iocage(8).”FreeBSD，2017 年 4 月 20 日，<www.freebsd.org/cgi/man.cgi?query=iocage&sektion=8>。
- Dzonsons, Kristaps，与 Ingo Schwarze. “roff(7).”FreeBSD，2018 年 4 月 10 日，<www.freebsd.org/cgi/man.cgi?query=roff&apropos=0&sektion=0>。

## 结论

从零开始编写手册页起初可能让人望而生畏。但稍作研究后，**mdoc(7)** 标记语言便易于使用。手册页是程序不可或缺的组成部分，而 EXAMPLES 章节更是巨大的助力。

---

**AARON ST. JOHN** 就职于 iXsystems。他刚毕业，拥有数学学士学位，对一切技术与电子游戏充满热情。
