# ZPool 检查点

作者：**Serapheim Dimitropoulos**

今年 3 月（2018 年），Alexander Motin（`mav@freebsd.org`）将 OpenZFS 的 Pool Checkpoint 功能从 Illumos 移植到 FreeBSD。pool 检查点可以看作是“池范围快照”，捕获创建检查点时 pool 的整个状态，允许用户将整个 pool 回滚到该状态或丢弃它。一般用例是管理任务，如操作系统升级，涉及更改或销毁 ZFS 状态和元数据的操作。此类操作的示例包括：启用新的 pool 功能、更改数据集属性、销毁快照和文件系统。在执行此类操作之前，管理员可以创建 pool 的检查点，然后应用更改。如果升级出问题，管理员可以回滚到检查点，就像从未采取过这些操作一样。就像快照可以帮助你将用户数据返回到先前状态一样，检查点可以帮助你将 ZFS pool 返回到先前状态。

已经有两个在线教程演示如何使用此功能，源代码中也有一个块注释，提供了其实现的高级概述（见参考框）。本文介于两者之间，给出每次管理操作期间底层发生情况的高级描述。

## 检查点 Pool

在 ZFS 中，我们使用事务组（Transaction Groups，即 TXG）跟踪随时间的变化。在每个 TXG 期间，ZFS 在内存中累积变化，当满足某些条件时，将这些变化同步到磁盘，然后打开下一个 TXG。每个数据块记录其创建时的 TXG，称为其出生 TXG。出生 TXG 对 pool 检查点功能很重要，因为它们帮助我们区分在检查点之前和之后创建的块。最后，TXG 同步到磁盘的最后一步是将新的 uberblock 写入每个磁盘开头和结尾的称为 ZFS 标签的区域，这与 EFI 标签不同。每个 uberblock 是该 TXG 的整个 pool 状态树的根。Uberblocks 在 pool 导入时用作起始点，以查找所有 pool 数据和元数据的最新版本。

每当管理员检查点 pool（使用 `zpool checkpoint` 命令）时，ZFS 将当前 TXG 的 uberblock 复制到 pool 状态中称为 Meta-Object Set（MOS）的区域内。当此更改同步到磁盘时，后续 TXG 的 uberblocks 通过 MOS 引用“已检查点”的 uberblock。

## 检查点 Pool 中块的生命周期

当存在检查点时，分配新块的过程保持不变，但释放块的过程不同。我们不能释放属于检查点的块，因为我们希望能够回滚到那个时间点。同时，我们也不能完全停止释放块，因为这会填满 pool。因此，每次我们要释放块时，都会查看其出生 TXG 并将其与保存在 MOS 中的“已检查点”uberblock 的 TXG 进行比较。如果块在已检查点 uberblock 的 TXG 处或之前出生，则意味着该块是检查点的一部分（即被已检查点的 uberblock 引用）。这些“已检查点”块实际上从未被释放。相反，我们将它们的范围添加到磁盘上的列表中，称为检查点 spacemaps（每个 vdev 一个），并将它们的段标记为已分配，以便 pool 不会重用它们。出生 TXG 在检查点 TXG 之后的块不是已检查点状态的一部分，可以正常释放。

## 回滚到检查点

如果管理员想回滚到检查点，只需 export 然后使用回滚选项 re-import pool（`zpool import --rewind-to-checkpoint`）。在这种情况下，导入过程照常进行，但有一个额外步骤。ZFS 首先查看当前 uberblock，并从中在 MOS 中找到已检查点的 uberblock。然后使用已检查点的 uberblock 而不是当前 uberblock 进行导入过程，有效地将 pool 回滚到已检查点的状态。导入过程完成后，回滚完成。检查点 spacemaps 不再存在，因为它们是在已检查点 uberblock（现在是当前 uberblock）之后创建的。出于同样原因，MOS 中没有检查点 uberblock。这意味着回滚后没有额外清理，pool 不再有检查点。

也可以以只读方式导入检查点，以访问检查点创建时存在的 pool 状态，而无需实际撤销自检查点创建以来发生的所有更改。这可以让管理员恢复特定数据或已销毁的文件系统，而无需回滚整个 pool。

## 丢弃检查点

如果管理员决定丢弃检查点，他们运行 discard 命令（`zpool checkpoint --discard`）。该命令指示 ZFS 从 MOS 中移除已检查点的 uberblock。此时，pool 被视为不再有检查点，这允许块正常释放，无论其出生 TXG 如何。ZFS 还将释放之前记录在检查点 spacemaps 中的所有范围——这些块是因为被已检查点 uberblock 引用而无法实际释放的块。

这些块的数量可能很大，具体取决于检查点存在的时间和 pool 进行了多少更改。在单个 TXG 中释放它们全部会很昂贵。相反，ZFS 生成一个线程，通过将它们分块预取到内存中并在多个 TXG 中每个 TXG 释放一个块来逐步释放它们。

## 致谢

没有 Delphix 同事，尤其是与我一起启动原型的 Dan Kimmel 和一路指导我的 Matt Ahrens 的帮助，此功能的开发将无法完成。我还要感谢 Alexander Motin 将该功能移植到 FreeBSD，感谢 Marius Zaborski 推广它。

---

**Serapheim Dimitropoulos** 是在 Delphix 从事 ZFS 工作的软件工程师。他对项目的主要贡献是 Log Spacemap 和 Pool Checkpoint 功能。不编程时，Serapheim 把时间花在踢足球、跳萨尔萨舞和弹古典吉他。

## 参考资料

- Serapheim 的 ZPool Checkpoint 教程：<http://sdimitro.github.io/post/zpool-checkpoint/>
- Marius 的 ZPool Checkpoint 教程：<http://oshogbo.vexillium.org/blog/46/>
- 源代码中的实现概述块注释：<https://github.com/freebsd/freebsd/blob/master/sys/cddl/contrib/opensolaris/uts/common/fs/zfs/spa_checkpoint.c>
