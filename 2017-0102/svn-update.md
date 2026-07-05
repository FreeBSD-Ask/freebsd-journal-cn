# SVN 动态

作者：Steven Kreuzer

对 FreeBSD 项目而言，过去这一年令人难以置信。src 树有超过 10,000 次提交，Ports 树已接近 27,000 个软件包，23 位新开发者获颁提交权限，FreeBSD 10.3 与 11 也相继发布。出于种种原因，我有一种奇怪的感觉：当我们回顾 2016 年时，“有趣”一词会被频繁使用，而 FreeBSD 项目也未能例外。虽然 12 月 base 的活动略有减少，但契合主题，我确实看到几条我会形容为”有趣”的提交。

## clang、llvm、lldb、compiler-rt 与 libc++ 升级至 3.9.0，lld 已导入

（<https://svnweb.freebsd.org/changeset/base/309124>）

此提交最激动人心的部分是它把 lld 引入了 base 系统，这对 LLVM 项目而言是重要里程碑。这是首个能链接真实世界大型用户态程序的版本，包括 LLVM/Clang/LLD 自身。事实上，它已能用于产出 FreeBSD 分发的大部分用户态程序。

## 支持 Ingenic XBurst JZ4780 与 X1000 系统级芯片

（<https://svnweb.freebsd.org/changeset/base/308857>）

Ingenic Semiconductor 设计基于 MIPS 指令集的 CPU 微架构。此次提交为 Imgtec CI20 和 Ingenic CANNA 单板机引入了支持。

## 给若干简单 stdio 程序添加 Capsicum 支持

（<https://svnweb.freebsd.org/changeset/base/308432>）

我们看到越来越多应用开始利用 Capsicum——一个轻量级 OS 能力与沙箱框架。近期一次提交为 echo、sleep、basename、dc、dirname、fold、getopt、locate、logname、printenv 和 yes 添加了 Capsicum 支持，因为这些用户态工具只与 stdio 交互。

## 新增 WITH_LLD_AS_LD 构建开关

（<https://svnweb.freebsd.org/changeset/base/309142>）

若设置，它会把 LLD 安装为 **/usr/bin/ld**。LLD（截至 3.9 版本）尚不能链接 world 与 kernel，但能自举并链接许多体量可观的应用。无论该开关如何设置，GNU ld 仍用于构建 world 与 kernel。它在 arm64 上默认启用，在其他所有 CPU 架构上默认关闭。

## pw 操作性能改进（编辑 **/etc/group** 或 **/etc/passwd**）

（<https://svnweb.freebsd.org/changeset/base/308806>）

r285050 修复了 `pw` 在断电时可能导致 **/etc/passwd** 或 **/etc/group** 损坏的 bug。然而，修复方式是用 `O_SYNC` 打开这些文件，这非常慢，尤其在 ZFS 上。本次提交用位置得当的 `fsync()` 替换了 `O_SYNC`，速度大幅提升。使用 ZFS tmpdir 时，运行 `pw` 的 kyua 测试时间从 245 秒下降到 35 秒。

## NFS 使用缓冲 pager

（<https://svnweb.freebsd.org/changeset/base/308980>）

该 pager 因其构造方式，为 page-in 实现了聚簇。具体而言，buildworld 负载显示 READ RPC 从 39k 降至 24k。未观察到实际或 CPU 时间变化。

## Concurrency Kit 加入 base 系统

（<https://svnweb.freebsd.org/changeset/base/309266>）

CK 是一个采用 BSD 许可的工具包，提供并发原语、安全内存回收机制和非阻塞数据结构，用于研究、设计和实现高性能并发系统。

## contrib/ 中的其他更新

- Subversion 更新至 1.9.5 版本（<https://svnweb.freebsd.org/changeset/base/309356>）
- ntp 升级至 4.2.8p9 版本（<https://svnweb.freebsd.org/changeset/base/308957>）
- ACPICA 升级至 20161117（<https://svnweb.freebsd.org/changeset/base/308953>）
- am-utils 更新至 6.2 版本（<https://svnweb.freebsd.org/changeset/base/308493>）
- jemalloc 更新至 4.3.1 版本（<https://svnweb.freebsd.org/changeset/base/308473>）
- file 更新至 5.29 版本（<https://svnweb.freebsd.org/changeset/base/308420>）
- tzdata 更新至 2016i 版本（<https://svnweb.freebsd.org/changeset/base/308270>）
- libarchive 更新至 3.2.2（<https://svnweb.freebsd.org/changeset/base/307861>）

---

## 欢迎投稿

如有文章创意，请联系 Jim Maurer（<jmaurer@freebsdjournal.com>）。

---

**Steven Kreuzer** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车情有独钟。他与妻子、女儿和一只狗住在纽约皇后区。
