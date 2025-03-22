# 基本系统中的 mfsBSD

- 作者：SOOBIN RHO
- 原文链接 <https://freebsdfoundation.org/our-work/journal/browser-based-edition/configuration-management-2/mfsbsd-in-base/>


mfsBSD 是一款基于 FreeBSD 的内存操作系统。

mfsBSD 的不同之处在于，它是完全运行于内存的 FreeBSD 实例——因此叫做 mfs（memory file system，内存文件系统）。所以，这意味着当我们在使用 mfsBSD 时，不会对现有的磁盘设备造成任何干扰。例如，我们可以将其用于本地服务器和云服务器的故障排除①。在邮件列表中搜索 mfsBSD，看看人们是如何解决各种问题的，实乃一大趣事。我个人最喜欢的应用场景是在仅有单个磁盘的设备中安装 FreeBSD，如果因某种原因无法使用 FreeBSD 安装介质，我就会：先制作 mfsBSD 镜像、将这个镜像安装到磁盘设备上、启动 mfsBSD、最后运行 bsdinstall。示例如下：

 首先，构建 mfsBSD：

```
# 现在正在对将 mfsBSD 集成到基本系统中的补丁集进行审查，可在网址查看：
# https://reviews.freebsd.org/D41705
cd /usr/src/releasemake mfsbsd-se.img
# 此处 se 是指 mfsBSD 特别版本（special edition），它包含了
# dist 文件——即 base.txz 和 kernel.txz：bsdinstall 需要它们
cd /usr/obj/usr/src/${ARCH}/release/ls -lh
```

然后，将 mfsBSD 写入磁盘设备：

```
# 把 mfsBSD 安装到你的目标磁盘设备上
# 把 ada0 换成你自己的目标磁盘设备
dd if=./mfsbsd-se.img of=/dev/ada0 bs=1Mreboot
```

启动到 mfsBSD，然后执行 bsdinstall：

```
# 复制特别版本（special edition）的 dist 文件，以便 bsdinstall 用其进行安装。
mkdir /mnt/distmount /dev/ada0p3 /mnt/dist
mkdir /usr/freebsd-distcp /mnt/dist/<version>/*.txz /usr/freebsd-dist/bsdinstall
```

mfsBSD 能够把整个 FreeBSD 系统都加载到内存。加载完成后，就可以对原始磁盘进行修改任意了，因为现有的 mfsBSD 文件都运行在内存中。正如 Matuška 在其白皮书（2009 年）中所述，“mfsBSD 是个工具集，能创建基于 mfsroot 的短小精悍版 FreeBSD，它将所有文件都放在内存。”②

## mfsBSD 简史

mfsBSD 的作者是 Martin Matuška（`mm@FreeBSD.org`）。回顾 mfsBSD 存储库的提交日志，他首次提交于 2007 年 11 月 11 日，大约是在 FreeBSD 7.0 BETA 的发布同期。“这个项目[mfsBSD]基于 Depenguinator 项目的想法”，Depenguinator 是 Colin Percival 在 2003 年创建的项目，旨在为仅提供 Linux 发行版的专用服务器远程安装 FreeBSD③。Matuška 想开发 FreeBSD 6.x Depenguinator 的功能实现，这就是 mfsBSD 的起源。

自此以降，Matuška 维护着 <https://mfsbsd.vx.sk/> ，用于分发 mfsBSD 的镜像。他已经在 GitHub④ 上维护了十七年的 mfsBSD，一路修复了无数 bug，并添加了对 zfsinstall、`/usr` tar 压缩包的支持等。在此期间，mfsBSD 知名度不断增长，

2023 年 5 月（即此篇文章写作的前一年），这个谷歌编程之夏（Google Summer of Code，GSoC）项目开始了把 mfsBSD 集成到基本系统中的尝试。

## 谷歌编程之夏

这一切是如何开始的呢？当时，我正在阅读黑客新闻（可能我正在拖延大学作业）。这是我第一次知道谷歌编程之夏（GSoC）。那里的某条热门评论说 FreeBSD 项目也参与了。有趣的是，评论中只字未提其他事情，好像其他事情都是不言自明的。

我立刻被吸引住了。关于 FreeBSD 最令我惊讶的是：macOS 衍生于 FreeBSD，而且 Netflix 也在其 CDN 中使用了 FreeBSD。谷歌编程之夏的申请流程包括提交项目提案。原则上要求申请者从各组织的项目创意列表中寻找项目主题（除非你有自己的打算）。我看了看列表，对我来说，mfsBSD 项目最有趣，因为其他项目创意似乎比我能接受的内核开发更遥远。

在给我的导师们发了封电子邮件后，我收到了 Joseph Mingrone(`jrm@FreeBSD.org`) 和 Juraj Lutter（`otis@FreeBSD.org`）的回复，并进行了简短的 Zoom 通话。几周后，我收到了谷歌编程之夏的录取通知。然后，我们进入到了所谓的社区粘合期（community bonding period）。在这期间，所有的贡献者和导师们相聚集于虚拟会议，总共开了半个小时，所有人都进行了自我介绍。那天是 2023 年 5 月 12 日（星期五）。

## 基本系统中的 mfsBSD

三个月又二十二天，将 mfsBSD 集成到基本系统中的项目终于完成了，历经许多的反复调试（很多 Bug 是通过谷歌搜索和翻阅所有过去的 GitHub 问题来解决的），测试（使用我的两台笔记本电脑 Thinkpad T440 和 P17 来运行 shell 脚本），并且我向导师提出了许多问题。一整套的三个补丁提交到了 Phabricator⑤。

简单地说，第一个提交“mfsBSD: Vendor import mfsBSD（mfsBSD: 引入 mfsBSD）”将 mfsBSD 集成为 `contrib/mfsbsd`。主要提交“release: Integrate mfsBSD image build targets into the release tool set（发行：将 mfsBSD 镜像构建目标集成到发行工具集）”：在 `release/Makefile` 中添加了目标 `mfsbsd-se.img` 和 `mfsbsd-se.iso`（作为 `release/Makefile.mfsbsd`）。最后一次提交“release(7): Add entries for the new mfsBSD build targets（release(7): 为新的 mfsBSD 构建目标添加条目）”在 `share/man/man7/release.7` 上添加了相应的条目。这意味着，现在我们可以在构建所有 FreeBSD 安装介质（如 cdrom、dvdrom、memstick 和 mini-memstick）的同时，在相同的发行版 Makefile 中使用 `make release WITH_MFSBSD=1` 构建 mfsBSD。

现在，正在对补丁集进行审查。mfsBSD 之前位于 FreeBSD 发布工具链以外，仅生成 release 版本。我的设想是将这个补丁集作为基本系统的一部分，来提供 mfsBSD 镜像，并通过调用 `cd /usr/src/release && make release WITH_MFSBSD=1` 来构建定制 mfsBSD 镜像，然后在 `/usr/obj/usr/src/${ARCH}/release/` 创建 `mfsbsd-se.img` 和 `mfsbsd-se.iso`。

### 参考文献

- ①. Matuška, Martin. (2022). FreeBSD 在 Hetzner 专用服务器上——VX Weblog。[在线] 参考：<https://blog.vx.sk/archives/353>
- ②. Matuška, Martin（2009）。mfsBSD 工具集，用于创建基于内存文件系统的 FreeBSD 发行版。[在线] 可在以下网址找到：<https://people.freebsd.org/~mm/mfsbsd/mfsbsd.pdf>
- ③. Colin Percival（2003）。Depenguinator — FreeBSD 远程安装。[在线] 可在以下网址找到：<http://www.daemonology.net/depenguinator/>
- ④. Matuška, Martin（2024）。mmatuska/mfsbsd。[在线] GitHub。可在以下网址找到：<https://github.com/mmatuska/mfsbsd>
- ⑤. [https://reviews.freebsd.org/D41705](https://reviews.freebsd.org/D41705)

---

**SOOBIN RHO** 是南达科他州奥古斯塔纳大学的大四学生。他出生于韩国，但在迪拜长大，最终选择在美国上大学。他现在是美国合众银行网络安全部门的兼职员工。在毕业后，他将成为一名信息安全分析师。自 2023 年谷歌编程之夏以降，他一直是 FreeBSD 的贡献者。
