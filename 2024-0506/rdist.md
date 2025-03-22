# rdist

- 作者：**CY SCHUBERT**
- 原文链接：<https://freebsdfoundation.org/our-work/journal/browser-based-edition/configuration-management-2/rdist/>

## 什么是 rdist？

引用手册的话说，“rdist is a program to maintain identical copies of files over multiple hosts（rdist 是一款在多个主机上维护相同文件副本的程序）”rdist 是一款可用于多种用途的通用工具，比如通过网络实现文件的同步——类似于 rsync 和 unison；亦可作为分布式配置管理工具：如 cfengine、ansible、puppet、salt 等其他配置管理工具。

## 为什么要使用 rdist？

要理解为什么要使用 rdist，知道点历史能让我们更好地了解它因何而产生。

Ralph Campbell 在加州大学伯克利分校（1983-1984 年间）编写了 rdist。rdist 首次出现在 4.3BSD（1986 年），是首个解决分布式软件管理问题的软件。从 20 世纪 80 年代末到 90 年代初，它几乎随所有商业 UNIX 一同分发。当时它成为了远程平台管理的标准。

rdist 是同类软件中最早出现的（也早于 rsync）。rsync 是一款备份工具，用于备份和克隆目录，而 rdist 通常用作网络文件的分发工具。它们都针对其各自设计目的进行了改造。

有这么多可选，为什么还要用 rdist 呢？

- rdist 比后续出现的配置分发应用程序（如 ansible 等）较为轻量化。
- rdist 容易集成到 shell 脚本和 Makefile 中。
- rsync 无法像 rdist 一样并行分发到多个主机。rsync 也不能使用配置文件来同步文件。但这些 rdist 都可以做到。rsync 和 rdist 被设计用于不同的目的，rsync 用于备份和文件克隆，而 rdist 更适合用作配置管理工具。

另一方面，为什么会有人想用别的工具呢？与诸如 cfengine 和 ansible 之类的工具相比，rdist 更轻量化——其配置远程节点的能力仅限于分发文件和执行简单的分发后任务。而较为重量级的工具可以用来执行分发前的任务：这个问题通过简单的 shell 脚本和 Makefile 就能解决。以我自己为例，我使用了一款名为 ipfmeta 的配置工具来管理我的 ipfilter 防火墙规则，该工具使用 Makefile 从规则文件和对象文件中生成防火墙配置文件。而 Makefile 使用 rdist 把生成的文件分发到 rdist 的 Distfile 中定义的远程防火墙中。Distfile 与 rdist 的关系有点类似于 Makefile 和 make。与 rsync 不同，rdist 根据其 Distfile 中的具体规则来分发文件。

## rdist 是如何工作的？

如同 make（1）通过解析其 Makefile 来构建程序一样，rdist 解析其 Distfile——来获得要分发的文件和目录以及要在分发后执行哪些任务。最初，rdist 使用不安全的 rcmd（3）接口进行网络通信。rcmd（）会连接到远程 rshd（8）。当连接建立时，它会生成 rdistd（8）远程文件分发服务器，在远程服务器上执行分发功能。这类似于 ssh 的 sftp：ssh 为 sftp 提供了远程功能。

伯克利的“r”命令（如 rsh）是不安全的。如今的 rdist 传输可基于 ssh 实现，而不用 rsh。使用 ssh 可以使用 ssh 密钥和 GSSAPI（kerberos）进行身份验证。与 ansible 不同，在 ansible 中的连接是使用你自己的账户进行的，并通过“become”进行提权；而 rdist 必须登录到目标服务器上的 root。为了便于实现这一点，可将 sshd_config 中的 PermitRootLogin 设置为 prohibit-password，从而强制使用 ssh 密钥和 Kerberos 凭据。

rdist 本身不进行身份验证。它依赖于传输机制进行身份验证。与 ansible 相比，它还依赖于 ssh 传输机制进行身份验证，并依赖于 su(1)、sudo(1)（或 ksu(1)）进行提权。

rdist 可用于管理服务账户中的应用程序文件，如 mysql、oracle 和其他应用程序账户。用所需的账户名替换 root 即可。

rdistd(8) 必须位于目标服务器的用户搜索路径 ($PATH)。

rdist 协商协议版本：systuils/rdist6（软件包、Port）使用 RDIST 第六版本协议，而 sysutils/rdist7（alpha 版本）使用 RDIST 第七版本协议。

## 安装 rdist

安装 rdist，只需，

```
pkg install rdist6
```

或

```
pkg install rdist7
```

若使用 ports，以 rdist7 为例，

```
cd /usr/ports/sysutils/rdist7
make install clean
```

## 使用 rdist

如上所述，类似于主机 make 使用其配置文件，rdist 使用类似于其配置文件的配置文件。我们必须编写我们自己的 Distfile。

Distfile 有以下三种类型声明。

### Distfile

和 make 一样，rdist 会寻找名为 Distfile（及 distfile）的文件。就像我们可以覆盖 make 所使用的 Makefile 的名称一样，我们同样也可以覆盖 rdist Distfile 的名字。

Distfiles 应包含一系列条目，指定要分发（复制）的文件，要将这些文件复制到哪些节点，以及在分发文件后要执行的后续操作。

### 变量

可以使用以下格式将一个或多个项目分配给变量。

```
<variable name> '=' <name list>
```

比如，

```
HOSTS = ( matisse root@arpa )
```

以上将字符串 matisse 和 root@arpa 定义成变量 HOSTS。

以下示例将三个目录名分配给了变量 FILES。

```
FILES = ( /bin /lib /usr/bin /usr/games )
```

### 将文件分发到其他主机

第二种语句类型告诉 `rdist` 将文件分发到其他主机。其格式为：

```
[ label: ] <source list> '->' <destination list> <command list>
```

`<source list>` 是文件名或变量名。`<destination list>` 是文件将被复制到的主机列表。而 `<command list>` 是要应用于复制操作的 `rdist` 指令列表。

可选的标签用于在命令行引用时识别语句以进行部分更新。

例如，我的防火墙 Distfile：

```
install-ipf: ipf.conf -> ${HOSTS}
install /etc/ipf.conf ;
special “chown root:wheel /etc/ipf.conf; chmod 0400 /etc/ipf,conf” ;
```

这将要求 `rdist` 把 `ipf.conf` 安装到 HOSTS 变量中列出的节点中。命令行 install 告诉 rdist 文件，安装位置是 /etc/ipf.conf。

命令行 special 要求 `rdist` 在复制操作之后运行 `chown` 和 `chmod`。

标签 `install-ipf` 可以在 `rdist` 命令行中被引用，限制此行为仅用于该操作，即 `rdist install-ipf`。

命令列表涉及的关键字有 `install`、`except`、`special` 和 `cmdspecial`。

|关键字 | 说明|
| -------------- | ------------------------------------------ |
| install        | 指定目标文件的安装位置。                      |
| notify         | 列出复制操作完成后要通知的电子邮件地址。       |
| except         | 无需复制的文件，即例外模式。                  |
| except_pat     | 与 `except` 相同，但使用正则表达式模式。        |
| special        | 每个文件复制后要执行的 shell 命令。            |
| cmdspecial     | 所有文件复制后要执行的 shell 命令。            |

下面是个简单的例子。它将我这篇文章的工作副本复制到 FreeBSD 工作目录树中的某个目录。

```
HOSTS = ( localhost )

FILES = ( /t/tmp/rdist.odt )

${FILES} -> ${HOSTS}
install /home/cy/freebsd/rdist/rdist.odt ;
```

这里我们将文件 `/t/tmp/rdist.odt` 复制到我的笔记本电脑上的 `/home/cy/freebsd/rdist/rdist.odt`。当然，一个简单的 `cp(1)` 命令就可以完成，但这个简单的例子让我们初步了解了单个文件的复制。还请注意，目标是同名文件。如目标是个目录，如 `/home/cy/freebsd/rdist`，它将删除目标目录中的所有文件和子目录，并用单个 `rdist.odt` 文件进行替换。指定目标文件或目录时要小心。其类似于：

```
rsync -aHW --delete /t/tmp /home/cy/freebsd/rdist
```

意料之外的结果可能会造成那些不幸的日子。

`rdist(1)` 手册页提供了一个更好的例子：

```
             HOSTS = ( matisse root@arpa)

              FILES = ( /bin /lib /usr/bin /usr/games
                       /usr/lib /usr/man/man? /usr/ucb /usr/local/rdist )

              EXLIB = ( Mail.rc aliases aliases.dir aliases.pag crontab dshrc
                       sendmail.cf sendmail.fc sendmail.hf sendmail.st uucp vfont )

              ${FILES} -> ${HOSTS}
                       install -oremove,chknfs ;
                       except /usr/lib/${EXLIB} ;
                       except /usr/games/lib ;
                       special /usr/lib/sendmail “/usr/lib/sendmail -bz” ;

              srcs:
              /usr/src/bin -> arpa
                       except_pat ( \\.o\$ /SCCS\$ ) ;

              IMAGEN = (ips dviimp catdvi)

              imagen:
              /usr/local/${IMAGEN} -> arpa
                       install /usr/local/lib ;
                       notify ralph ;

              ${FILES} :: stamp.cory
                       notify root@cory ;
```

在上述示例中，列在 `FILES` 变量中的文件将从本地主机复制到 `HOSTS` 变量中列出的机器上。除了 `EXLIB` 变量中列出的文件、`/usr/games/lib` 和一个模式之外。每个文件复制后，将运行带 `-bz` 参数的 `sendmail`。

`special` 一般用于运行 shell 命令。但在上述例子中，`special` 执行了 `/usr/lib/sendmail`（就如同 shell 一样），将调用的参数传给 `sendmail`。

`/usr/local` 中的三个文件将被复制到目标系统上的 `/usr/local/lib`，复制完成后会发送电子邮件通知 `ralph`。

作业完成时会触发时间戳文件，发送邮件给 `root@cory`。时间戳文件用于避免不必要的复制。例如，如果有列出的文件比时间戳文件更新（而 ansible 使用校验和），才会复制该文件。

## 注意事项

如上所述，一不小心，就会发生意外。如同 `rsync`，`rdist` 也不会验证源文件与目标文件是否为同一类型的对象（文件还是目录）。很容易就会用一个目录替换目标文件或者用一个文件替换目标目录。就像 `rsync` 一样，它可能会破坏系统。请小心，并在沙盒或 jail 中进行测试。

## 总结

`rdist` 是一个很好的工具，当与脚本、makefile、其他工具结合使用时，尤其是在单个工具不能完成全部任务的情况下，可与其他工具联合使用。正如我管理 `ipfilter` 防火墙、`ipfmeta`、make Makefile、`rdist` Distfile 和 git `rdist` 的方式一样，`rdist` 可以很好地被集成以创建一个轻量级的应用程序。在与 ansible 或 cfengine 这类不能与脚本和 makefile 集成的重量级工具集成时，`rdist` 填补了这一独特的空白。`rdist` 恪守了最初的 UNIX 哲学——即单一工具用于单一目的，可以与其他工具集成以创建新工具和应用程序。

## 参考文献

- [FreeBSD 14.0-RELEASE 和 Port 的 44bsd-rdist 手册页](https://man.freebsd.org/cgi/man.cgi?query=44bsd-rdist&sektion=1&apropos=0&manpath=FreeBSD+14.0-RELEASE+and+Ports)
- [Magnicomp 的 rdist 重新设计 PDF](https://www.magnicomp.com/download/rdist/overhaul.pdf)
- [UMB 计算机科学课程的 rdist 项目 PDF](https://www.cs.umb.edu/~ckelly/teaching/common/project/linux/sys_admin/p7_rdist.pdf)

---

**Cy Schubert** 是 FreeBSD src 和 Ports 的提交者。他的职业生涯始于五十多年前，编写和维护着用 Fortran 写就的电气工程应用程序。他的经验包括 IBM MVS（大型机）系统编程，编写 MVS 内核和作业输入子系统 2（JES/2）的扩展。在三十五年前，他的职业生涯转向 UNIX 之路，开始涉足 SunOS、Solaris、Tru64、NCR AT&T、DG/UX、HP-UX、SGI、Linux 和 FreeBSD 系统管理。

Cy 的 FreeBSD 之旅亦始于三十五年前。在尝试了某个 Linux 内核 0.95 的 Linux 发行版后，发现它不支持 UNIX 域套接字（UDS）；他又试了试了某个实验性的 Linux 内核。在那个悲惨的月份，恢复了被实验性内核破坏的 EXTFS 文件系统以后，Cy 在 FreeBSD 和 NetBSD 的 USENET 新闻组上提出了一个问题。唯一回复了 Cy 问题的人是 FreeBSD 项目的 Jordan Hubbard。由于 Jordan 是第一个也是最后一个回复的人，Cy 决定先试试 FreeBSD。自那时起，他就一直在使用着 FreeBSD（从 2.0.5 开始）。他在 2001 年成为了 Port 提交者，于十一年前成为了源代码提交者。目前，他受雇于一家大型托管服务提供商的加拿大子公司。
