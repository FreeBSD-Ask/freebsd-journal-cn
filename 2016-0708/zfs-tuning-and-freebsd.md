# ZFS 调优与 FreeBSD

- 原文：[ZFS Tuning and FreeBSD](https://freebsdfoundation.org/wp-content/uploads/2016/08/ZFS-Tuning-and-FreeBSD.pdf)
- 作者：**Michael W. Lucas**

系统管理员学习 ZFS 时，通常会对 ZFS 的空间使用感到困惑。把存储池、dataset、快照和克隆组合在一起，让 ZFS 的空间利用率变得非常复杂。当你开始调整 dataset 的 `recordsize` 属性时，空间使用会直接进入迷离境界。

`recordsize` 属性决定 ZFS 文件系统 dataset 中逻辑块的最大尺寸。默认 `recordsize` 是 128 KB，在 4 KB 扇区的磁盘上为 32 个扇区，在 512 字节扇区的磁盘上为 256 个扇区。2015 年引入 `large_blocks` 特性标志后，最大记录尺寸增至 1 MB。许多数据库引擎偏好更小的块，如 4 KB 或 8 KB。在专用于此类文件的 dataset 上调整 `recordsize` 是有意义的。即使你不改 `recordsize`，ZFS 也会按需自动调整记录尺寸。写一个 16 KB 的文件应该只占 16 KB 空间（加上元数据和冗余空间），不会浪费整个 128 KB 的记录。

这在数据库场景中最为关键。

## 数据库与 ZFS

ZFS 的许多特性对数据库非常有利。每位 DBA 都想要快速、简单且高效的复制、快照、克隆、可调缓存和池化存储。ZFS 虽然是作为通用文件系统设计的，但你可以调优它，让数据库飞起来。

数据库通常包含不止一种类型的文件，由于每种文件都有不同的特征和使用模式，各自需要不同的调优。我们会重点讨论 MySQL 和 PostgreSQL，但这些原则适用于任何数据库软件。

对数据库来说最重要的调优是 dataset 块尺寸——通过 `recordsize` 属性控制。任何可能被覆写的文件，其 ZFS `recordsize` 都需要与应用程序使用的块尺寸匹配。

调整块尺寸还能避免写放大。写放大指修改少量数据需要写入大量数据的情况。假设你要修改一个 128 KB 块中间的 8 KB。ZFS 必须读取这 128 KB，修改其中的 8 KB，计算新的校验和，并写入新的 128 KB 块。ZFS 是写时复制文件系统，最终会写一个全新的 128 KB 块，只为修改那 8 KB。你不会想这样。把这种情况乘以数据库的写入次数。写放大会重创性能。

虽然这种优化对许多人来说并非必需，但对于高性能系统可能不可或缺。它还会影响 SSD 和其他基于闪存、寿命期内写入量有限的存储的寿命。当然，不同的数据库引擎不会让这件事变得简单，每个都有不同的需求。日志、二进制复制日志、错误和查询日志以及其他杂项文件也需要不同的调优。

在创建小 `recordsize` 的 dataset 之前，务必理解 VDEV 类型与空间利用率之间的相互作用。某些情况下，512 字节小扇区的磁盘可以提供更好的存储效率。完全有可能为数据库单独建一个池会更合适，主池用于其他文件。

对高性能系统，使用镜像而非任何类型的 RAID-Z。是的，从弹性角度看你可能想要 RAID-Z。艰难的取舍正是系统管理的乐趣所在！

## 所有数据库

在数据库上启用 lz4 压缩出人意料地能降低延迟。压缩数据从物理介质上读取更快，因为读取量更少，从而缩短传输时间。凭借 lz4 的提前中止特性，最坏情况只比不压缩慢几毫秒，但收益通常相当可观。这就是 ZFS 自身所有元数据和 L2ARC 都使用 lz4 压缩的原因。

当压缩 ARC 特性进入 OpenZFS 后，在 dataset 上启用压缩还能让更多数据保留在 ARC——最快的 ZFS 缓存中。在 Delphix 进行的一项生产案例研究中，一台配有 768 GB RAM 的数据库服务器，从占用超过 90% 内存来缓存数据库，变为仅用 446 GB 缓存 1.2 TB 压缩数据。压缩内存缓存带来了显著的性能提升。由于该机器无法再加内存，压缩是唯一的提升途径。

ZFS 元数据也会影响数据库。当数据库快速变化时，为每次变更写入两到三份元数据副本，会占用后端存储大量可用 IOPS。通常，相比默认的 128 KB 记录尺寸，元数据量相对较小。但数据库更适合小记录尺寸。保留三份元数据副本引发的磁盘活动，可能等于甚至超过向池中写入实际数据的活动。

较新版本的 OpenZFS 还包含 `redundant_metadata` 属性，默认值为 `all`，这是以往 ZFS 版本的原始行为。但该属性也可设为 `most`，使 ZFS 减少其保留的某些类型元数据的副本数。

根据你的需求和工作负载，让数据库引擎管理缓存可能更好。ZFS 默认将数据库的大部分或全部数据缓存在 ARC 中，而数据库引擎也保留自己的缓存，造成浪费的双重缓存。把 `primarycache` 属性设为 `metadata` 而非默认的 `all`，告知 ZFS 不要在 ARC 中缓存实际数据。`secondarycache` 属性类似地控制 L2ARC。

根据访问模式和数据库引擎，ZFS 可能已经更高效。使用 zfs-tools 包中的 `zfsmon` 等工具监控 ARC 缓存命中率，并与数据库内部缓存进行比较。

压缩 ARC 特性可用后，考虑减小数据库内部缓存，改由 ZFS 处理缓存或许是明智之举。ARC 在相同 RAM 中能容纳的数据可能远多于你的数据库。

## MySQL——InnoDB/XtraDB

InnoDB 在 MySQL 5.5 中成为默认存储引擎，特征与之前使用的 MyISAM 引擎有显著差异。Percona 的 XtraDB（也被 MariaDB 使用）与 InnoDB 类似。InnoDB 和 XtraDB 都使用 16 KB 块尺寸，因此包含实际数据文件的 ZFS dataset 应将 `recordsize` 属性设为匹配值。我们还推荐使用 MySQL 的 `innodb_one_file_per_table` 设置，把每个表的 InnoDB 数据放在单独文件中，而非全部归入单一 `ibdata` 文件。这让快照更有用，并允许更有选择性地恢复或回滚。

把不同类型的文件放在不同 dataset 上。数据文件需要 16 KB 块尺寸、lz4 压缩和精简元数据。仅缓存元数据可能带来性能提升，但也会禁用预取。多做实验，看看你的环境如何表现。

```sh
# zfs create -o recordsize=16k -o compress=lz4 -o redundant_metadata=most -o primarycache=metadata mypool/var/db/mysql
```

MySQL 的主日志用 gzip 压缩效果最好，且不需要在内存中缓存。

```sh
# zfs create -o compress=gzip1 -o primarycache=none mysql/var/log/mysql
```

复制日志用 lz4 压缩效果最好。

```sh
# zfs create -o compress=lz4 mypool/var/log/mysql/replication
```

通过 **/usr/local/etc/my.cnf** 中的以下设置，告知 MySQL 使用这些 dataset。

```ini
data_path=/var/db/mysql
log_path=/var/log/mysql
binlog_path=/var/log/mysql/replication
```

现在可以初始化数据库并开始加载数据。

## MySQL——MyISAM

许多 MySQL 应用仍使用较老的 MyISAM 存储引擎，要么因其简单，要么只是尚未转换为 InnoDB。

MyISAM 使用 8 KB 块尺寸。dataset 记录尺寸应设为匹配。dataset 布局其余部分与 InnoDB 相同。

## PostgreSQL

如果调优得当，ZFS 可以支撑非常大且快的 PostgreSQL 系统。在创建所需 dataset 之前，不要初始化数据库。

PostgreSQL 默认对所有内容使用 8 KB 存储块。如果改了 PostgreSQL 的块尺寸，必须相应改变 dataset 尺寸。

默认 FreeBSD 安装中，PostgreSQL 位于 **/usr/local/pgsql/data**。对于大型部署，你可能为此数据单独准备一个池。这里我使用名为 `pgsql` 的池。

```sh
# zfs set mountpoint=/usr/local/pgsql pgsql
# zfs create pgsql/data
```

现在遇到了鸡生蛋蛋生鸡的问题。PostgreSQL 的数据库初始化例程期望自己创建目录树，但我们希望特定子目录有各自的 dataset。最简单的办法是先让 PostgreSQL 完成初始化，再创建 dataset 并移动文件。

```sh
# /usr/local/etc/rc.d/postgresql oneinitdb
```

初始化例程创建数据库、视图、模式、配置文件以及高端数据库的所有其他组件。现在可以为特殊部分创建 dataset 了。

PostgreSQL 把数据库存在 **/usr/local/pgsql/data/base**。预写日志（WAL）位于 **/usr/local/pgsql/data/pg_xlog**。把这两者都移开。

```sh
# cd /usr/local/pgsql/data
# mv base base-old
# mv pg_xlog pg_xlog-old
```

两者都使用 8 KB 块尺寸，你希望分别对它们做快照，因此为每个创建一个 dataset。与 MySQL 一样，告知 ARC 仅缓存元数据。还通过 `logbias` 属性告知这些 dataset 在吞吐量与延迟之间偏向吞吐量。

```sh
# zfs create -o recordsize=8k -o redundant_metadata=most -o primarycache=metadata logbias=throughput pgsql/data/pg_xlog
# zfs create -o recordsize=8k -o redundant_metadata=most -o primarycache=metadata logbias=throughput pgsql/data/base
```

把原始目录的内容复制到新 dataset 中。

```sh
# cp -Rp base-old/* base
# cp -Rp pg_xlog-old/* pg_xlog
```

现在可以启动 PostgreSQL 了。

> ZFS 设计为良好的通用文件系统。如果你的 ZFS 系统作为典型办公室的文件服务器，确实不必为文件尺寸调优。但如果你知道自己会有什么尺寸的文件，可以通过调整来提升性能。你必须自行测试，决定哪种设置适合你的工作负载。

## 按文件尺寸调优

### 小文件

在没有 SLOG 的系统中高速创建大量小文件时，ZFS 会花费大量时间等待文件和元数据刷入稳定存储。

如果你愿意承担过去五秒（若 **vfs.zfs.txg.timeout** 更高则更多）内新建文件丢失的风险，把 `sync` 属性设为 `disabled` 会告知 ZFS 把所有写入视为异步。即使应用程序请求在文件安全之前不要被告知写入完成，ZFS 也会立即返回，并在下一个定期 txg 中写入文件。

高速 SLOG 让你既能同步又能快速地存储那些小文件。

### 大文件

ZFS 近期通过 `large_block` 特性加入了对大于 128 KB 块的支持。如果你要存储许多大文件，确实应考虑这一点。默认最大块尺寸为 1 MB。

理论上可以使用大于 1 MB 的块尺寸。但很少有系统对此进行过广泛测试，且与内核内存分配子系统的交互在长期使用下也未经检验。你可以尝试极大的记录尺寸，但出问题时记得提交 bug 报告。`sysctl` **vfs.zfs.max_recordsize** 控制最大块尺寸。

一旦激活 `large_blocks`（或任何其他特性），该池就不能再被不支持该特性的主机使用。要停用该特性，需销毁任何 `recordsize` 曾设为大于 128 KB 的 dataset。

存储系统在延迟与吞吐量之间艰难平衡。ZFS 使用 `logbias` 属性决定倾向哪一方。ZFS 默认使用 `latency` 作为 `logbias`，让数据快速同步到磁盘，使数据库和其他应用能继续工作。处理大文件时，把 `logbias` 属性改为 `throughput` 可能获得更好性能。你必须自行测试，决定哪种设置适合你的工作负载。

MICHAEL W. LUCAS 是《Absolute FreeBSD》《Absolute OpenBSD》《DNSSEC Mastery》等书的作者。他与妻子和一群大鼠住在密歇根州底特律。访问他的网站：<https://www.michaelwlucas.com>。
