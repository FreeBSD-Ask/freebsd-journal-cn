# SVN 动态

- 原文：[svn update](https://freebsdfoundation.org/wp-content/uploads/2016/06/svn-update.pdf)
- 作者：Steven Kreuzer

过去几年，我们看到对 ARM 这一系列精简指令集计算（RISC）处理器的兴趣与日俱增。其爆炸式流行部分归功于 BeagleBone 和 树莓派 等廉价单板计算机，它们为人们提供了简单而强大的实验平台。在生产环境中，IT 专业人员也把目光投向 ARM，以扩展基础设施，同时尝试降低功耗和散热成本。

我们看到 ARM 驱动的设备从手机、平板到高性能计算环境无所不包，随着物联网让沙发能发推文、冰箱在 Facebook 上加你的搅拌机为好友，我预计其版图还会继续扩张。

虽然 ARM 仍非 Tier 1 平台，但每天我们都看到大量工作投入 ARM64 支持的改进，参与者既有业余开发者，也有意在以 FreeBSD 为核心构建 ARM 设备的商业公司。除此之外，提交了全新的 IO 调度器，还有几项变更提升了 ZFS 性能，并让你能更好地控制用户或进程对存储池的访问强度。如果对 Haswell 的支持还不够，最近又引入了 GPU 支持，使 FreeBSD 能在多种笔记本配置上提供改进的图形和更长的电池续航。

## 全新 CAM I/O 调度器

<https://svnweb.freebsd.org/changeset/base/298002>

这是一个全新的调度器，优先读取而非写入，并能对 SSD 的写入吞吐量进行节流。这个新调度器还为支持 NCQ TRIM 的 SSD 带来了相应支持。除了更好地满足读密集型工作负载的需求，这个新调度器还保留了比默认调度器多得多的统计信息，便于了解系统何时饱和、需要减负。目前它不是默认调度器，可能仍有些不完善之处，但在 Netflix 使用超过一年。

## ZFS 的 I/O 限制

<https://svnweb.freebsd.org/changeset/base/297633>

我们的新资源——`readbps`、`readiops`、`writebps` 和 `writeiops`——已加入 `rctl`，现在允许你为进程、用户、登录类别或 Jail 设置磁盘 I/O 限制。这对任何需要与 IOP 密集型应用或用户共享机器的系统管理员来说，都是受欢迎的新增功能。

## 改进的 ZFS 间接块推测性预取

<https://svnweb.freebsd.org/changeset/base/297832>

宽 ZFS 池上许多操作的可扩展性，受限于必须先预取间接块的要求。最近添加的异步间接块读取部分缓解了问题，但未完全解决。此变更扩展现有预取器功能，使其明确处理间接块。在此变更之前，预取器最多提前发出 8MB 数据的读取。此变更后，它还会提前发出最多 64MB 数据的间接块读取，因此到真正需要读取这些数据时，可以立即完成。

## ARM64 copyinout 改进

<https://svnweb.freebsd.org/changeset/base/297209>

复制对齐缓冲区时使用更宽的加载/存储指令，为 FreeBSD/arm64 带来了巨大的性能提升。用 `dd` 从 **/dev/zero** 复制 1G 到 **/dev/null** 的简单测试显示，速度从 410MB/s 提升到 3.6GB/s。

## 在 gptboot 和 gptzfsboot 中实现 GELI

<https://svnweb.freebsd.org/changeset/base/296963>

你可能注意到最近启动加载器变胖了一点。最近新增了从 GELI 加密的 UFS 和 ZFS 分区启动的支持。其实现方式的更多细节，可在 Allan Jude 于 AsiaBSDCon 2016 上发表的论文中找到。

## 启动时 DTrace 支持

<https://svnweb.freebsd.org/changeset/base/297773>

现在可在 i386 和 amd64 上，在启动早期、`dtrace(1)` 可调用之前启用 DTrace 探针。所需的启用通过 `dtrace -A` 创建，它会写入一个 **/boot/dtrace.dof** 文件，并使用 `nextboot(8)` 确保在随后的启动中加载 DTrace 内核模块，并由 `loader(8)` 加载描述此启用的 DOF 文件。追踪输出随后可通过 `dtrace -a` 获取。

## ARM64 上用户空间应用的追踪支持

<https://svnweb.freebsd.org/changeset/base/297611>

允许捕获函数调用栈的 `dtrace_getupcstack` 已移植到 ARM64。这使你能使用 DTrace 探测用户空间应用。

## 更新的 i915 GPU 驱动

<https://svnweb.freebsd.org/changeset/base/296548>

最近 FreeBSD 图形团队发起了一项大规模行动，将 i915 GPU 驱动更新到与 Linux 3.8.13 一致。此次更新最激动人心的特性是为 FreeBSD 带来了 Haswell GPU 的支持。

## ARM64 内核地址空间增至 512GB

<https://svnweb.freebsd.org/changeset/base/297914>

此变更同时增加了 DMAP 区域，后者也增至 2TiB。

## ARM64 上的 4 级页表支持

<https://svnweb.freebsd.org/changeset/base/297446>

用户空间地址空间增至 256TiB。

## 为 linux 和 linux64 模块添加 kern.features 标志

<https://svnweb.freebsd.org/changeset/base/297597>

为帮助第三方应用判断宿主是否支持 linux 仿真，为 linux 和 linux64 模块引入了新的 `kern.features` 标志。如果 `kern.features.linux` 等于 1，则支持 32 位 Linux 二进制文件；如果 `kern.features.linux64` 设为 1，则支持 64 位二进制文件。

## head/contrib 更新

FreeBSD 基础用户空间由不少工具组成，其中一部分在项目之外开发。过去几个月，我们看到了一些帮助创建良好用户体验的第三方软件更新。

- bmake 升级到 20160307 版本 — <https://svnweb.freebsd.org/changeset/base/297357>
- byacc 升级到 20160324 版本 — <https://svnweb.freebsd.org/changeset/base/297276>
- libxo 升级到 0.6.1 版本 — <https://svnweb.freebsd.org/changeset/base/298083>

---

**Steven Kreuzer** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车有兴趣。他与妻子、女儿和狗住在纽约皇后区。
