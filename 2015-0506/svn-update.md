# SVN 动态

- 原文：[SVN Update](https://freebsdfoundation.org/our-work/journal/browser-based-edition/measure-twice-code-once/)
- 作者：**Glen Barber**

本期 SVN 动态介绍若干虚拟化更新、`xz`(1) 压缩工具的一次令人瞩目的更新。

## stable/10@r280632（<https://svnweb.freebsd.org/changeset/base/r280632>）

FreeBSD 历来提供出色的 ABI/API 向后兼容性，使得为较旧 FreeBSD 版本编译的二进制能在较新的 FreeBSD 发行版上运行。例如，只要内核编译时包含了 `COMPAT_FREEBSD4`（以提供 4.x 内核兼容层，包含在内核配置中），且安装了 `misc/compat4x` port（在需要时提供较旧的共享库），就可以在 FreeBSD 11-CURRENT 上运行为 FreeBSD 4.4 编译的二进制。

这也意味着可以在 11-CURRENT 机器上运行 FreeBSD 4.x Jail。但有一个问题：尽管 Jail 用户态会识别为 4.x，Jail 环境仍会反映 11-CURRENT。

最近，`jail`(8) 已更新，允许为 Jail 环境调整 `kern.osreldate` `sysctl`(8)，这样任何需要知道内核版本的程序都会看到 4.x 内核。

用更近的 FreeBSD 版本为例，如果你有一台运行 11-CURRENT 的机器，上面有两个 Jail，一个用于 9.3-RELEASE，一个用于 10.1-RELEASE，那么两个 Jail 的 `kern.osreldate` 都会反映 11000XX（目前 XX 是 73），你可以分别为每个 Jail 设置内核版本，使其与该发行版的 `__FreeBSD_version` 值（来自 **/usr/src/sys/sys/param.h**）匹配。

## head@r281316（<https://svnweb.freebsd.org/changeset/base/r281316>）

`xz`(1) 是一款通用压缩工具，类似 `gzip`(1) 与 `bzip2`(1)，已更新至上游版本 5.2.1。5.2.x 版本提供的一项令人瞩目的特性是多线程支持，压缩文件时可使用系统上所有可用的 CPU 核。

对于近期的 FreeBSD 发行版，发布工程师提供了压缩与未压缩两种格式的安装镜像。自加入多线程支持以来，一组快照构建从开始到完成（覆盖该分支支持的所有架构）的时间显著减少。（事实证明，大部分时间花在压缩生成的镜像上，而不是实际构建 FreeBSD 源代码或制作安装介质。）此前，一整套 FreeBSD 11-CURRENT 快照需要 12 到 16 小时，现在大约只需 7 小时，相当惊人。

刚加入多线程支持时，我做了一些基准测试，看看传给 `xz`(1) 的不同线程标志会如何影响生成的镜像大小。`-T` 标志接受一个数值参数，即要使用的核数，特殊参数 `0`（零）默认使用系统上所有可用核。

FreeBSD 基金会最近为支持发布工程团队而购置的发布构建机器是 48 核、128GB 内存。我进行的测试分别使用 `-T 1`（单线程）、`-T 10`（10 线程）、`-T 20`（20 线程）与 `-T 0`（本例中为 48 线程）。我还用 `-e` `xz`(1) 标志做了额外测试，该标志在某些情况下能以更长的压缩时间为代价获得更高的压缩率。

用于此测试的 disc1.iso 镜像为 613353472 字节，略低于 585 MB。单线程测试在 4.5 分钟内生成了 336.15 MB 的镜像。10 线程测试在 32.55 秒内生成了 337.47 MB 的镜像。令我惊讶的是，20 线程测试也生成了 337.47 MB 的镜像，耗时 23.71 秒。48 线程测试（使用 `-T 0`）也产生了同样的 337.47 MB 镜像，耗时 17.37 秒。这的确相当惊人。使用 `-T 0` 与 `-e` 的测试在 20.34 秒内生成了 337.39 MB 的镜像，多花 3 秒并未带来太大收益。

为完整起见，上方附上了 `time`(1) 输出与目录列表。

对于大文件压缩，这次 `xz`(1) 更新的确意义重大。

```sh
# time xz-T 1-k disc1.iso ; mv disc1.iso.xz disc1.iso.1.xz
266.534u 1.149s 4:27.68 99.9% 81+193k 4+43029io 0pf+0w

# time xz-T 10-k disc1.iso; mv disc1.iso.xz disc1.iso.10.xz
256.588u 2.850s 0:32.55 797.0% 81+192k 5+43198io 0pf+0w

# time xz-T 20-k disc1.iso; mv disc1.iso.xz disc1.iso.20.xz
303.180u 5.393s 0:23.71 1301.4% 81+192k 5+43198io 0pf+0w

# time xz-T 0-k disc1.iso; mv disc1.iso.xz disc1.iso.48.xz
285.227u 9.958s 0:17.37 1699.3% 81+192k 4+43198io 0pf+0w

# time xz-T 0-e-k disc1.iso ; mv disc1.iso.xz disc1.iso.48e.xz
345.370u 8.275s 0:20.34 1738.6% 81+192k 4+43187io 0pf+0w

# ls-al disc1.iso*
-rw-r-r-1 root wheel 613353472 Feb 20 19:50 disc1.iso
-rw-r-r-1 root wheel 352488020 Feb 20 19:50 disc1.iso.1.xz
-rw-r-r-1 root wheel 353870676 Feb 20 19:50 disc1.iso.10.xz
-rw-r-r-1 root wheel 353870676 Feb 20 19:50 disc1.iso.20.xz
-rw-r-r-1 root wheel 353870676 Feb 20 19:50 disc1.iso.48.xz
-rw-r-r-1 root wheel 353786156 Feb 20 19:50 disc1.iso.48e.xz
```

## head@r280259（<https://svnweb.freebsd.org/changeset/base/r280259>）

FreeBSD 基金会与 ARM、Cavium、Semihalf sp.j. 合作，资助 FreeBSD 开发者 Andrew Turner 将 FreeBSD 移植到 64 位 ARM 平台（即 aarch64）。

FreeBSD/aarch64 的初始支持在 r280259 中加入 11-CURRENT，此后持续开发。该项目的目标是将 FreeBSD/aarch64 提升至 Tier-1 支持状态，包括发行安装介质与第三方 package。

制作 FreeBSD/aarch64 发行介质的初始支持在 r281802 中加入 11-CURRENT，该工作由 FreeBSD 基金会资助。目前，FreeBSD FTP 镜像上提供了用于 Qemu 模拟器的虚拟机镜像、内存棒安装镜像。

请注意，启动虚拟机镜像需要一个 Qemu EFI loader 文件。更多细节（以及 EFI loader 文件的校验和），请参见 snapshots 公告邮件列表归档，其中包含了一个如何启动镜像的示例（<https://lists.freebsd.org/pipermail/freebsd-snapshots/2015-May/000147.html>）。

感谢各位读者支持 FreeBSD 社区、FreeBSD 期刊，当然还有 FreeBSD 基金会。

别忘了，FreeBSD-CURRENT 与 FreeBSD-STABLE 分支的开发 ISO 与预装虚拟机镜像（VHD、VMDK、QCOW2 与 RAW 格式）可在 FTP 镜像上找到，每周构建：<ftp://ftp.freebsd.org/pub/FreeBSD/snapshots/>。

一如既往，开发快照不用于生产环境；但我们鼓励定期测试，这样我们才能让下一个 FreeBSD 发行版如你所期望的那般出色。

作为爱好者，Glen Barber 约在 2007 年起深度参与 FreeBSD 项目。此后他参与了多种职能，最近的岗位让他能专注于项目中的系统管理与发布工程。Glen 住在美国宾夕法尼亚州。
