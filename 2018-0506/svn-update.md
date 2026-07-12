# SVN 动态

作者：**Steven Kreuzer**

11.2-RELEASE 代码冻结目前生效，到你阅读本文时，应该已有 `releng/11.2`。预计在 2018 年 6 月 27 日左右看到发布公告，并开始准备升级，因为 FreeBSD 11.1 将于 2018 年 9 月 30 日到达生命周期终点。

## 引入 **dwatch(1)** 作为让 DTrace 更有用的工具

<https://svnweb.freebsd.org/changeset/base/330559>

**dwatch(1)** 是一个让 dtrace 更有用的工具。dwatch 提供了一种有趣且无痛的方式来做各种事情，从实时观看系统进程调度器，到过滤文件系统事件及介于两者之间的一切。

## 添加对 zstd 压缩的用户和内核核心转储的支持

<https://svnweb.freebsd.org/changeset/base/329240>

这与现有的 gzip 压缩支持类似，但 zstd 通常更快且压缩率更好。必须通过在内核配置文件中添加 `ZSTDIO` 来配置对此功能的支持。**dumpon(8)** 的新选项 `-Z` 用于配置内核转储的 zstd 压缩。**savecore(8)** 现在能识别并保存带 `.zst` 扩展名的 zstd 压缩内核转储。

## 为 zfs_getpages() 添加后读/预读支持

<https://svnweb.freebsd.org/changeset/base/329363>

ZFS 将它读取的块缓存在其 ARC 中，因此一般来说，可选页面不如将数据直接读取到目标页面的文件系统有用。但可选页面对于减少页面错误次数和相关的 VM/VFS/ZFS 调用仍然有用。另一个被优化（作为副作用）的情况是从空洞中换入。ZFS DMU 目前不提供方便的 API 来检查空洞。相反，它创建一个临时的零填充块，并允许像访问普通数据块一样访问它。从空洞中逐个获取多个页面会导致临时块（及关联的 ARC 头）的反复创建和销毁。

## 降低 ARC 碎片阈值

<https://svnweb.freebsd.org/changeset/base/315449>

由于 ZFS 可以请求最多 `SPA_MAXBLOCKSIZE` 的内存块（例如在 `zfs recv` 更新期间），我们开始积极回收的阈值使用 `SPA_MAXBLOCKSIZE`（16M）而不是较低的 `zfs_max_recordsize`（默认为 1M）。

## 修复客户端设置 OPEN_SHARE_ACCESS_WANT* 位时 NFSv4.1 的 OpenDowngrade

<https://svnweb.freebsd.org/changeset/base/332790>

NFSv4.1 RFC 规定 `OPEN_SHARE_ACCESS_WANT` 位可以设置在 OpenDowngrade 的参数 `share_access` 中，并且基本被忽略。它还将错误从 `NFSERR_BADXDR` 改为 `NFSERR_INVAL`，因为 NFSv4.1 RFC 规定如果设置了虚假位，应返回此错误。（NFSv4.0 RFC 没有为这种情况指定任何错误，因此 NFSv4.0 也可以更改错误回复。）

## 使 lagg 创建更容错

<https://svnweb.freebsd.org/changeset/base/332645>

当 `SIOCSLAGGPORT` 返回错误时发出警告，而不是退出。当我们在 lagg 创建期间因错误退出时，单个失败的 NIC（不再附加）可以阻止 lagg 创建和其他配置（如添加 IPv4 地址），从而使机器无法访问。为 `SIOCSLAGGPORT` 的退出状态保留非 `EEXISTS` 错误，以防脚本正在查找它。希望如果 ifconfig 的其他部分可以允许“软”失败，这可以扩展。改进警告消息，提及有问题的 lagg 和成员。

## 添加 TCP 高精度定时器系统（tcp_hpts）支持

<https://svnweb.freebsd.org/changeset/base/332770>

这是引入 Rack 和 BBR 的先驱/基础工作，两者都使用 hpts 来控制数据包发送节奏。此功能是可选的，需要先启用选项 `TCPHPTS` 才能激活。使用它的 TCP 模块必须确保基础组件已编译到加载它们的内核中。

## 从 **getlogin(2)** 中移除缓存

<https://svnweb.freebsd.org/changeset/base/332119>

此缓存自 CSRG 导入以来就存在，但没有明显用途。当然，`setlogin()` 很少被调用，但 `getlogin()` 的调用也应该不频繁。所需的失效在 aarch64、arm、mips、amd riscv 上未实现，因此如果在 `setlogin()` 之前调用 `getlogin()`，更新永远不会发生。

## 移除对 Arcnet 协议的支持

<https://svnweb.freebsd.org/changeset/base/332490>

虽然 Arcnet 在工业控制中仍有部署，但市场上缺乏任何 PCI、USB 或 PCIe NIC 的驱动程序，表明此类用户并未运行 FreeBSD。PR 数据库中的证据表明 **cm(4)** 驱动（我们唯一的 Arcnet NIC）在 5.0 中就已损坏，此后一直无法工作。

## 向 syslogd 添加 RFC 5424 syslog 消息输出

<https://svnweb.freebsd.org/changeset/base/332510>

添加 `fprintlog_rfc5424()` 以发出 RFC 5424 格式的日志条目。添加 `-O` 命令行选项以启用 RFC 5424 格式。如果我们支持 `-o rfc5424` 就好了，就像 NetBSD 上那样。不幸的是，标志 `-o` 在 FreeBSD 上已用于不同目的。对于有兴趣使用此功能的人，可以通过在 **/etc/rc.conf** 中添加以下行来启用：

```sh
syslogd_flags="-s -O rfc5424"
```

## 添加 sortbench

<https://svnweb.freebsd.org/changeset/base/332796>

这是一组 `qsort`、`mergesort`、`heapsort` 以及可选的 `wikisort` 的基准测试，以及一个运行它们的脚本。

---

**STEVEN KREUZER** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车感兴趣。他和妻子、女儿和狗住在纽约皇后区。
