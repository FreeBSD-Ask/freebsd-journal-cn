# 如何构建 Port

- 原文：[How to Build a Port](https://freebsdfoundation.org/our-work/journal/browser-based-edition/zfs-best-practices/)
- 作者：**Erwin Lansing**

与 FreeBSD 项目互动

简要介绍 Ports 系统的结构，并带你入门编写简单的 port。

FreeBSD 长期以来以 Ports 树闻名——过去用户必须从编译所有想安装的软件开始，某些情况下这既慢又麻烦。近几年里，大量工作投入到打造一流的 package 系统中，用户现在只需下载并安装预编译好的 package，配套了全新的打包工具和全新的 package 分发基础设施。对大多数用户而言，这已是推荐的软件安装方式；只要使用新的 `pkg.8` 系统（参见《FreeBSD 期刊》2014 年 3/4 月刊），大多数用户不再需要自行编译软件。但这一切之下仍是 FreeBSD Ports 系统，它掌握着如何在 FreeBSD 上构建和安装第三方软件的全部信息。在近 25,000 个 package 之中，仍有可能出现某个 package 尚不存在的情况，于是就需要编写新的 port。同样，要修改现有的 package，也得改动其底层的 port。本文简要介绍 Ports 系统的结构，并带你入门编写简单的 port。

首先你需要取得一份新的 FreeBSD Ports 树。最快的方式是用 `portsnap.8`。这是个两步过程。第一步抓取 portsnap 数据文件，约 70MB，视网络情况可能耗时不少。第二步安装 Ports 树本身，会解压大量小文件，视磁盘速度也可能花费一些时间。

```sh
portsnap fetch
portsnap extract
```

这会在 **/usr/ports** 下放置一份完整的 Ports 树。更新一份先前由 portsnap 安装的 Ports 树是类似的两步流程：

```sh
portsnap fetch
portsnap update
```

查看 **/usr/ports** 目录，你能看到若干文件和大量目录。最先应注意到的是 Makefile 文件。FreeBSD Ports 树用 `make.1` 编写，这是一种在构建和安装软件时广泛使用的语言。当然，单个 200 行的 Makefile 远远不够，Ports 基础设施的大部分代码位于 **/usr/ports/Mk** 目录。其他大部分目录都是分类目录，里面装着实际的 port。

极简的 port 由四个文件组成：Makefile、pkg-descr、pkg-plist 和 distinfo。Makefile 包含该 port 的一些基本信息和如何抓取、构建和安装它的指令。pkg-descr（即 package 描述）包含该 port 是什么、做什么的简短描述，还有该软件包官方项目网站的链接。pkg-plist（即打包清单）是该 port 安装的文件列表；最后，distinfo 包含该 port 需从互联网抓取的任何外部文件的校验和与大小。我下面以 **/usr/ports/shells/sash** 中的 Stand-Alone Shell（SASH）port 为例。在示例 1 中你还能看到名为 files 的目录，里面可以放置对该 port 有用的附加文件。

看示例 2 中 SASH 的 Makefile，第一行是该 port 最初加入 Ports 树时对原作者的署名。接下来是 subversion 标识串，会由 subversion 自动展开，对新 port 而言只应是：

```sh
# $FreeBSD$
```

接下来是一段变量，包含该 port 将要安装的软件的一些关键信息。`PORTNAME` 中的 port 名字在整个 Ports 树中应当唯一，不过也有选项可以为名字加上前缀或后缀。

`PORTVERSION` 是该 port 的版本，通常与原始软件包的版本一致。注意 port 的版本号绝不能回退，因为升级已安装 package 的工具无法处理版本倒退的情况。

`CATEGORIES` 是 port 分类列表，以该 port 所属的主分类开头——这也是它在 Ports 树中所处的目录。port 可以属于多个分类，列在主分类之后，包括那些本身没有对应目录、但仍能帮助用户找到该 port 的虚拟分类。可用分类的完整列表见 Porters Handbook：<https://www.freebsd.org/doc/en_US.ISO8859-1/books/porters-handbook/makefile-categories.html#porting-categories>。

示例 1：

```sh
erwin@panda:/usr/ports/shells/sash % ls -l
total 20
-rw-r--r-- 1 root wheel 349 Apr 11 2014 Makefile
-rw-r--r-- 1 root wheel 123 Apr 11 2014 distinfo
drwxr-xr-x 2 root wheel 512 Jun 24 2014 files
-rw-r--r-- 1 root wheel 458 Jan 22 2014 pkg-descr
-rw-r--r-- 1 root wheel 35 Jun 11 2014 pkg-plist
```

示例 2：

```sh
erwin@panda:/usr/ports/shells/sash % cat Makefile
# Created by: Patrick Gardella patrick@FreeBSD.org
# $FreeBSD: head/shells/sash/Makefile 350947 2014-04-11 13:41:06Z miwi $
PORTNAME= sash
PORTVERSION= 3.8
CATEGORIES=shells
MASTER_SITES= http://members.tip.net.au/~dbell/programs/
MAINTAINER= ports@FreeBSD.org
COMMENT= Stand-Alone Shell combining many common utilities
.include <bsd.port.mk>
```

`MASTER_SITES` 中的 URL 列表供 Ports 系统抓取该软件包的原始源码。建议提供多个，以防某个站点暂时不可达。这里也是某些魔法开始的地方。待抓取文件的实际 URL 由 Ports 系统根据先前指定的若干变量自动构造。最简形式下，默认为：

```sh
${MASTER_SITE}/${PORTNAME}-${PORTVERSION}.tar.gz
```

在 SASH 这个示例 port 中，就是：<http://members.tip.net.au/~dbell/programs/sash-3.8.tar.gz>

许多软件项目使用各自的版本号或命名方案，于是有大量附加变量可影响最终 URL，例如设置 `USE_BZIP2` 会把默认后缀改为 `.tar.bz2`，`USE_XZ` 会改为 `.tar.xz`。对于 SourceForge、CPAN 这类拥有大量镜像的知名下载站点，也有现成的宏可用。另一有趣且广泛使用的站点是 github——它没有“正式发布文件”的概念，于是 port 需要依赖某个给定的 commit 和 tag，以确保下载到一致且经过测试的软件版本。这些都在 Porters Handbook 中有详细描述：<https://www.freebsd.org/doc/en_US.ISO8859-1/books/porters-handbook/makefile-distfiles.html>。

下一段是该 port 的一些基本元数据。FreeBSD Ports 系统有“维护者”（maintainership）的概念：单人或某个邮件列表背后的多人作为该 port 的主要联系人。对该 port 提出修改请求（如升级到新版本）应获得维护者批准，在 FreeBSD bugzilla 中提交 bug 报告时会自动向 `MAINTAINER` 字段所列地址发送邮件。

`COMMENT` 顾名思义，是该 port 一句话的简短描述。更长的描述放在 pkg-descr 文件中。

最后，port 需要引入 Ports 基础设施本身，方式是包含主文件：

```sh
.include <bsd.port.mk>
```

示例 3：

```sh
erwin@panda:/usr/ports/shells/sash % cat pkg-descr SASH (Stand-Alone Shell)

It is a nice combination of bare-bones shell and a dozen or so most useful UNIX commands.

Shell includes: echo pwd cd mkdir mknod rmdir sync rm chmod chown chgrp touch mv ln cp cmp more exit setenv printenv umask kill where

Commands include: dd ed grep gzip ls tar file find mount chattr WWW: http://members.tip.net.au/~dbell/
```

说到 pkg-descr 文件，示例 3 列出了 shells/sash 这个 port 的 pkg-descr 内容，写得相当详尽细致。常规的 pkg-descr 应当只用几段话简明描述该 port，让用户无需阅读文档或访问网站就能知道它做什么。软件项目的官方网站可放在最后一行，以 WWW 开头。

示例 4：

```sh
erwin@panda:/usr/ports/shells/sash % cat pkg-plist
@shell bin/sash
man/man1/sash.1.gz
```

该 port 安装的文件列在 pkg-plist 中。这份清单在 port 生命周期的多个阶段都会用到。生成 package 的工具依据它知道要打包哪些文件；移除已安装 port 或 package 的工具也需要知道该删哪些文件。文件安装到 `${PREFIX}`（默认为 **/usr/local**），在 pkg-plist 中是隐含的，不应写出。如示例 4 所示，我们这个小 port 只安装一个可执行文件及其对应的 man page。Ports 系统的更多威力也在此体现：打包清单中可以使用关键字来处理更复杂的情形——比如仅把某个文件放到某个位置还不够，需要额外照顾，或者需要修改某个附加文件。我们示例中的 `@shell` 关键字不仅把 sash 可执行文件安装到 **/usr/local/bin**，还会把它加到 **/etc/shells** 中，使其成为系统认可的 shell。另一例子是 `@sample` 关键字，对于安装那些不应在日后升级时被默认版本覆盖的配置文件非常有用。使用 `@sample` 关键字时，port 或 package 会把默认版本安装为 `filename.conf.sample`，但在此之前会先检查 `filename.conf` 是否存在：若存在，再检查它是否与上一版的默认文件一致；若不一致或不存在，则同时把默认版本安装为 `filename.conf`。换言之，如果配置文件已存在且被用户修改过，这些修改不会被覆盖。更多关键字见 **/usr/ports/Keywords** 目录。

最后是 distinfo 文件，这也是最容易创建的文件。如果 Makefile 中所有变量都填写正确（尤其是那些影响如何抓取原始源码的变量），只需运行：

```sh
make makesum
```

它会抓取所需文件，生成包含 SHA256 校验和与文件大小的 distinfo 文件。见示例 5。

port 中的 files 目录可存放 port 在构建、安装或打包时需要的任何文件。以 `patch-` 为前缀的文件名会在解压原始源码与编译之间自动应用，对于移植那些原本未考虑 FreeBSD 的软件非常有用。见示例 6。

对于新 port，还需要在所属分类的 Makefile 中添加条目。我们这个例子中，就是 **/usr/ports/shells/Makefile**：

```sh
SUBDIR += sash
```

## 依赖

我们这个示例 port 没有用到的重要且常用的特性，就是依赖。依赖是指当前 port 在其生命周期的某个阶段需要的另一 port，比如运行时需要的某个可执行文件或其他文件，或者构建时需要的某个库。库通过 `LIB_DEPENDS` 变量指定，是 `library:port` 组合的列表。

示例 5：

```sh
erwin@panda:/usr/ports/shells/sash % cat distinfo
SHA256 (sash-3.8.tar.gz) =
13c4f9a911526949096bf543c21a41149e6b037061193b15ba6b707eea7b6579
SIZE (sash-3.8.tar.gz) = 53049
```

示例 6：

```sh
erwin@panda:/usr/ports/shells/sash % ls -l files/
total 20
-rw-r--r-- 1 root wheel 1356 Apr 11 2014 patch-Makefile
-rw-r--r-- 1 root wheel 442 Jan 22 2014 patch-cmd_ls.c
-rw-r--r-- 1 root wheel 2572 Apr 11 2014 patch-cmds.c
-rw-r--r-- 1 root wheel 551 Apr 11 2014 patch-sash.c
-rw-r--r-- 1 root wheel 219 Jan 22 2014 patch-sash.h
```

示例 7：

```sh
erwin@panda:/usr/ports/shells/sash % make describe
sash-3.8|/usr/ports/shells/sash|/usr/local|Stand-Alone shell combining many common utilities|/usr/ports/shells/sash/pkg-descr|ports@FreeBSD.org|shells||||||http://members.tip.net.au/~dbell/
```

```sh
LIB_DEPENDS= libsqlite3.so:${PORTSDIR}/databases/sqlite3
```

运行时或构建时的依赖以类似方式分别在 `RUN_DEPENDS` 和 `BUILD_DEPENDS` 变量中指定，由 `file:port` 组合的列表构成。

```sh
RUN_DEPENDS= ${LOCALBASE}/bin/bash:${PORTSDIR}/shells/bash
```

`${LOCALBASE}` 变量通常与我们前面看到的 `${PREFIX}` 相同，但二者有区别：`LOCALBASE` 表示已安装 port 的位置，`PREFIX` 表示当前 port 将要安装的位置。对于默认路径中的可执行文件，无需写完整路径，所以上面的例子可简写为：

```sh
RUN_DEPENDS= bash:${PORTSDIR}/shells/bash
```

## 测试 Port

强烈建议在测试时开启开发者模式。只需在 **/etc/make.conf** 中设置 `DEVELOPER=yes`：

```sh
# echo DEVELOPER=yes >> /etc/make.conf
```

这会开启额外的质量检查并显示更多警告，比如使用了已弃用的特性。

简单可行的初步测试是运行 `make describe`。它应输出由我们前面谈到的诸多元数据组成的字符串。这些信息供多个工具跟踪依赖关系、搜索 port 名称或描述。见示例 7。

`portlint` 工具位于 ports-mgmt/portlint，或者直接 `pkg install portlint` 即可获得，它评估 port 在语法上是否正确，并给出多年实践沉淀下来的一些最佳实践建议。对新 port 应运行 `portlint -A`；对现有 port 运行 `portlint -C` 即可。见示例 8。

示例 8：

```sh
erwin@panda:/usr/ports/shells/sash % portlint -C
WARN: Makefile: Consider defining LICENSE.
0 fatal errors and 1 warning found.
```

示例 9：

```sh
LICENSE= GPLv2
LICENSE_FILE= ${WRKSRC}/COPYING
```

这是一条有效的警告，因为我们的示例 port 没有设置 `${LICENSE}` 变量。`${LICENSE}` 变量可设为某个已知许可证缩写，列表见 **/usr/ports/Mk/bsd.licenses.db.mk**。如果软件项目带有许可证文件，也应通过 `${LICENSE_FILE}` 变量安装。见示例 9。

## 测试 Port：使用 poudriere

最简单的测试方式当然是把刚出炉的 port 装到本地系统上。我不推荐这么做，因为它很容易破坏现有系统——可能意外覆盖已有文件，或表现出其他意料之外的行为；而已安装的软件也会让检查新 port 到底新增了什么变得更难。比如在已安装了大量软件的系统上，很容易漏掉某个新安装的文件，从而在 pkg-plist 中遗漏。

使用干净安装或 jail 可以规避大多数这类陷阱，但也有工具可以替你完成所有重活。poudriere 系统（<https://fossil.etoilebsd.net/poudriere/doc/trunk/doc/index.wiki>）不仅能用于构建自家的 package 仓库，也是出色的 port 测试框架。它可以测试整个 Ports 树、子集，甚至单个 port 及其依赖。借助 jail 和 ZFS 文件系统，它能保证自身独立、对宿主系统毫无影响。它会按 port 生成非常有用的日志文件，可用于查看 port 是如何构建的、package 如何安装和卸载，对于检查 pkg-plist 是否完整极有帮助。由于重度依赖 ZFS，它可能比较吃硬件，当然编译本身也消耗大量算力。

通过 package `pkg install poudriere` 或 ports-mgmt/poudriere 中的 port 安装 poudriere，然后检查 **/usr/local/etc/poudriere.conf** 中的设置，尤其是涉及 ZFS 文件系统的部分。接下来，用你想测试的 FreeBSD 版本创建 jail：

```sh
# poudriere jail -c -j 93Ramd64 -v 9.3-RELEASE -a amd64
```

如果你打算把 port 提交到 FreeBSD Ports 树，应在所有受支持的主要发行版上测试。Ports 团队支持的范围与 FreeBSD 安全官（<https://www.freebsd.org/security/>）相同，目前包括 FreeBSD 8、9、10 和 HEAD 中的开发版本 11。

当然你也需要 Ports 树，可以这样创建：

```sh
# poudriere ports -c
```

这会创建名为 default 的 Ports 树。现在可以测试你的 port 了：

```sh
# poudriere testport -j 93Ramd64 -p default -o shells/sash
```

日志文件位于 **/poudriere/data/logs/bulk/93Ramd64-default/latest/logs/sash-3.8.log**。把 poudriere 用作测试工具的更详尽入门，可参见 Porters Handbook：<https://www.freebsd.org/doc/en/books/porters-handbook/testing-poudriere.html>。本文撰写时，redports（<https://redports.org>）系统正停机维护。希望该站点能回归，请持续关注。它提供上述全部测试和大量扩展检查，全部通过便捷的 web 界面完成。

## 提交 Port

当你确认 port 按你期望的方式工作后，就可以提交它进入 FreeBSD 官方 Ports 树，这也会让 package 可通过 `pkg` 轻松安装。提交前确保 port 目录干净，没有任何多余文件。用 `make clean` 命令即可轻松删除 work 目录。从 port 目录上一级的分类目录下，用 `shar.1` 工具把文件打包成 shar 归档：

```sh
shar `find sash'> sash.shar
```

得到的 sash.shar 可提交到 FreeBSD bugzilla 数据库。

本文对 Ports 树的介绍仅触及皮毛，请务必查阅 Porters Handbook（<https://www.freebsd.org/doc/en/books/porters-handbook/index.html>），它不断更新，包含 Ports 树全部特性的详尽信息。还有活跃的 porter 社区乐于回答任何问题。你可以在 freebsd-ports 邮件列表（<http://lists.freebsd.org/mailman/listinfo/freebsd-ports>）或 EFnet 上的 IRC 频道 #bsdports 找到他们。祝 porting 愉快！

---

**Erwin Lansing** 与妻子和儿子居住在哥本哈根。他就职于 DK Hostmaster，负责让丹麦互联网的灯一直亮着。他也是 FreeBSD 基金会董事会副主席。
