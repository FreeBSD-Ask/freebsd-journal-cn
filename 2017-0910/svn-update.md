# SVN 动态

作者：STEVEN KREUZER

本期主题是 FreeBSD vs. Linux，因此我想回顾一下过去几个月 LinuxKPI 子系统的若干改动，应该会比较有趣。该项目最初于 2016 年 5 月宣布，旨在不必再把 Linux KMS 和 DRM 驱动的改动逐个移植到 FreeBSD。思路是引入一组垫片（shim）作为兼容层，让这些驱动只需极少改动即可运行，从而更易于跟上游开发，并大幅缩小 FreeBSD 代码与 Linux 原始代码之间的差异。此外，它还加快了 FreeBSD 集成新改动的速度——对于在尖端硬件上运行 FreeBSD 的人来说，这无疑是个好消息。

- 在 LinuxKPI 中正确实现 `poll_wait()`。此举防止在不安全的上下文中直接使用 `linux_poll_wakeup()` 函数，从而避免可能导致 use-after-free 的问题。<https://svnweb.freebsd.org/changeset/base/323349>

- 修复 LinuxKPI 中使用 `ip6_find_dev()` 时的 IPv6 scope ID 问题。<https://svnweb.freebsd.org/changeset/base/323351>

- 在 LinuxKPI 的 `linux_fget()` 中增加更多健全性检查。此举防止返回指向非 LinuxKPI 创建的文件描述符的指针。<https://svnweb.freebsd.org/changeset/base/323347>

- 移除 ibcore 中对 LinuxKPI 文件结构的不安全访问。`selwakeup()` 现在由 `wake_up()` 系列函数完成。<https://svnweb.freebsd.org/changeset/base/323350>

- 增加一些杂项定义以支持 DRM 驱动。<https://svnweb.freebsd.org/changeset/base/322795>

- 修复 LinuxKPI 的 RCU synchronize API 中的死锁问题。<https://svnweb.freebsd.org/changeset/base/322746>

- 在 LinuxKPI 中改用整数类型传递 jiffies 和/或 ticks 值，因为 FreeBSD 中 ticks 是 32 位的。<https://svnweb.freebsd.org/changeset/base/322357>

- 在 LinuxKPI 中实现 hrtimer API 的部分功能。<https://svnweb.freebsd.org/changeset/base/320364>

- 在 LinuxKPI 中新增 `u64_to_user_ptr()`。<https://svnweb.freebsd.org/changeset/base/320337>

- 在 LinuxKPI 中新增 `ns_to_ktime()`。<https://svnweb.freebsd.org/changeset/base/320336>

- 在 LinuxKPI 中新增 `noop_lseek()`。<https://svnweb.freebsd.org/changeset/base/320333>

- 在处理内存映射请求时，允许 LinuxKPI 中的 VM fault handler 为 NULL。当 VM fault handler 为 NULL 时，字符设备 pager 的 populate handler 将返回 `VM_PAGER_BAD`。<https://svnweb.freebsd.org/changeset/base/320189>

- 在 LinuxKPI 中新增 kthread parking 支持。<https://svnweb.freebsd.org/changeset/base/320078>

- 为 LinuxKPI 字符设备新增通用的 `kqueue()` 和 `kevent()` 支持。该实现允许创建 read 和 write 过滤器，并搭便车于 `poll()` 文件操作来决定过滤器何时触发。这种搭便车机制只需检查 `read()`、`write()` 或 `ioctl()` 系统调用是否返回 `EWOULDBLOCK` 或 `EAGAIN`，然后更新 `kqueue()` 轮询状态位。<https://svnweb.freebsd.org/changeset/base/319409>

- 改进 LinuxKPI 中的 `kqueue()` 支持。一些使用 `kqueue()` 的应用程序在事件驱动的文件描述符读取中未设置非阻塞 I/O 模式。这导致 LinuxKPI 内部的 kqueue 读写事件标志必须在下一次 read 和/或 write 系统调用之前更新，否则该系统调用可能阻塞。<https://svnweb.freebsd.org/changeset/base/319501>

- 为 LinuxKPI 实现 64 位原子操作。<https://svnweb.freebsd.org/changeset/base/294521>

STEVEN KREUZER 是一名 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车有兴趣。他与妻子、女儿和狗居住在纽约皇后区。
