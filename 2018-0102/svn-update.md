# SVN 动态

作者：**STEVEN KREUZER**

FreeBSD 相比 Linux 有何优势？这是我常被问到的问题，却难以回答，因为 FreeBSD 有很多方面做得很好。从何说起？你可以滔滔不绝地讲它健壮的网络栈，或 DTrace 和 Capsicum 等前沿技术。然而我认为 FreeBSD 真正出彩的领域是存储。开发者不仅花大量时间确保你写入磁盘的 0 和 1 以正确顺序落盘，还要尽可能快地完成。不仅如此，他们还要确保这些 0 和 1 以正确的顺序、尽可能快地读回。无论你只是在笔记本上归档家庭照片，还是向网络上数千客户端提供多 PB 的 ZFS 卷，FreeBSD 都提供了健壮可靠的平台来满足你的存储需求。

## 提供 sysctl 以强制同步初始化 inode 块

<https://svnweb.freebsd.org/changeset/base/326731>

FS 使用屏障写异步初始化 inode，确保 inode 块在对应的 cylinder group 头更新之前写入。某些 GEOM 似乎不能正确处理 `BIO_ORDERED`，意味着屏障写可能无法按预期工作。该 sysctl 允许以新文件系统上昂贵的文件创建为代价来规避此问题。

## 在安装程序中支持挂载的启动分区

<https://svnweb.freebsd.org/changeset/base/326674>

这允许平台层（例如）指定 EFI 启动分区应挂载到 **/efi**，并用 `newfs msdos` 正常格式化，而非从 **/boot/boot1.efifat** 复制而来。

## zfs_write：修复写入超配额时看似成功的问题

<https://svnweb.freebsd.org/changeset/base/326070>

问题出现在写入的偏移和大小与文件系统的 recordsize（最大块大小）对齐时。此场景下 `dmu_tx_assign()` 会因超配额而失败，但 uio 在我们从 uio 复制数据到借用的 ARC 缓冲区的代码路径中被修改。这造成部分写入的假象，于是 `zfs_write()` 返回成功，uio 与写入单个块一致地被修改。该 bug 会导致数据丢失，因为超配额的写入看似成功，实际数据却被丢弃。此提交通过确保在所有错误检查完成前不修改 uio 来修复该 bug。为此，代码现在使用 `uiocopy()` + `uioskip()`，与原始 illumos 设计一致。由于 `uiocopy()` 在 r326067 中更新为使用 `vn_io_fault_uiomove()`，这成为可能。

## 避免 uread() 和 uwrite() 中持有进程

<https://svnweb.freebsd.org/changeset/base/325887>

通常，上层代码会原子性地验证进程未在退出并持有进程。在一种情况下，我们用 `uwrite()` 将探测到的指令复制到每线程的临时空间块，但 `copyout()` 可用于此目的；此变更实质上回退了 r227291。

## 优化 telldir(3)

<https://svnweb.freebsd.org/changeset/base/326640>

目前每次调用 `telldir()` 都需要一次 malloc，并往链表添加一条目，该链表在后续 `telldir()`、`seekdir()`、`closedir()` 和 `readdir()` 调用中必须遍历。对每个目录条目都调用 `telldir()` 的应用在 `readdir()` 中产生 O(n²) 行为，在 `telldir()` 和 `closedir()` 中产生 O(n) 行为。此优化通过将相关信息打包为单个 long 表示，在大多数情况下消除了 malloc() 和链表。64 位架构上 msdosfs、NFS、tmpfs、UFS 和 ZFS 都可使用打包表示。32 位架构上，msdosfs、NFS 和 UFS 可使用打包表示，但 ZFS 和 tmpfs 仅能对每目录约前 128 个文件使用。每次 **telldir(3)** 调用约节省 50 字节内存。对于每目录一百万文件的 `telldir()` 密集型目录遍历，加速约 20—30 倍。

## 调整 seekdir、telldir 和 readdir，使有删除时对最后保存位置的定位仍可工作

<https://svnweb.freebsd.org/changeset/base/282485>

处理来自 Windows 的删除请求。Samba 需要此功能才能正确工作。这并未完全修复存在删除时的 seekdir 问题，但修复了最严重的情况。真正的解决方案必须涉及对 VFS 和 **getdirentries(2)** API 的某些改动。

## 当没有发出写委托时避免 nfsrv_checkgetattr() 中获取锁的开销

<https://svnweb.freebsd.org/changeset/base/326544>

manu@ 在 freebsd-current@ 邮件列表报告 `nfsrv_checkgetattr()` 中存在显著性能损失，由状态锁的获取/释放引起，即便没有发出写委托时也是如此。此补丁向 NFSv4 服务器添加了未完成写委托的计数。该计数允许 `nfsrv_checkgetattr()` 在计数为 0 时不获取任何锁即返回，避免了无写委托时的性能损失。

## zfsd 应能对消失并返回的 L2ARC 执行 online

<https://svnweb.freebsd.org/changeset/base/325011>

此前此功能不工作，因为 L2ARC 设备的标签不包含池 GUID。修改 zfsd 使其不再要求池 GUID。

## 修复 vfs.aio.enable_unsafe=0 时的 zpool_read_all_labels

<https://svnweb.freebsd.org/changeset/base/324991>

此前 `zpool_read_all_labels` 尝试执行 256KB 读取，超过默认的 `MAXPHYS`，因此必须走慢速、不安全的 AIO 路径。将这些读取缩小到 112KB，使其可使用安全、快速的 AIO 路径。

## 修复在过小设备上创建 zpool 时的错误消息

<https://svnweb.freebsd.org/changeset/base/324940>

按路径打开时不要在 `vdev_geom_attach` 中检查 `SPA_MINDEVSIZE`。它与 `vdev_open` 中的检查重复，且在此处附加失败会打印不正确的错误消息。

## 添加 vfs_zfs.abd_chunk_size 可调参数

<https://svnweb.freebsd.org/changeset/base/323797>

据报道，默认值 4KB 会带来显著的内存使用开销（至少在某些配置上）。使用 1KB 似乎能大幅降低开销。

## 添加 arc 收缩和增长值的 sysctl

<https://svnweb.freebsd.org/changeset/base/323051>

使用数 GiB ARC 时，`arc_no_grow_shift` 的默认值可能并非最优。通过 sysctl 暴露它，用户可轻松调优。同样原因也通过 sysctl 暴露 `arc_grow_retry`。其默认值 60 秒在密集负载下可能过长。

## msdosfs(5)：在文件模式中反映 READONLY 属性

<https://svnweb.freebsd.org/changeset/base/326031>

msdosfs 允许通过清除文件模式的 owner write 位来设置 READONLY。在 `msdosfs_getattr` 中，直观地将该 READONLY 属性反映到用户空间的文件模式中。

## 使用 taskqueue(9) 并发执行对镜像 DS 的写入/提交

<https://svnweb.freebsd.org/changeset/base/324676>

当 NFSv4.1 pNFS 客户端使用指定镜像数据服务器的 Flexible File Layout 时，必须对所有镜像执行写入和提交。此变更并发执行这些写入和提交。修改客户端使用 taskqueue 来执行此操作。taskqueue(9) 的线程数无法更改，因此默认设为 `4 * mp_ncpus`，但可通过设置 sysctl `vfs.nfs.pnfsiothreads` 覆盖。

---

**STEVEN KREUZER** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车有浓厚兴趣。他与妻子、女儿和狗住在纽约皇后区。
