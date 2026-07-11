# 书评：*FreeBSD Mastery: Advanced ZFS*

- 原文：[Book Review](https://freebsdfoundation.org/wp-content/uploads/2016/10/Book-Review.pdf)
- 作者：**Joseph Kong**

*FreeBSD Mastery: Advanced ZFS*

- 作者：Allan Jude 和 Michael W. Lucas
- 出版社：Tilted Windmill Press
- ISBN-10：0692688684
- ISBN-13：978-0692688687
- 页数：242

Allan Jude 和 Michael W. Lucas 合著的 *FreeBSD Mastery: Advanced ZFS* 清晰简洁地讲解了管理 Z 文件系统（ZFS）中较为复杂和深奥的部分。本书直击主题，不浪费你的时间。作者假设你已经熟悉 ZFS 存储池、数据集、快照、克隆等概念，而你翻开本书正是为了高级内容。

## 章节摘要

第 1 章“启动环境”介绍使用 ZFS 创建 Solaris 风格的启动环境，即可启动的操作系统内核和用户空间备份，让你轻松回退变更。

第 2 章“委派与 Jail”介绍 ZFS 的委派系统，允许你指定用户或组在每个数据集上可以执行的命令。还讨论了该委派系统如何与 FreeBSD Jail 协同工作。

第 3 章“共享数据集”讨论 ZFS 的网络文件共享特性，带你了解 FreeBSD 的 iSCSI 和 NFS 实现，并讨论它们与 ZFS 的关系。

第 4 章“复制”介绍如何在别处创建文件系统的精确副本——例如存储池中的另一个数据集、系统上的第二个存储池、外部驱动器、远程系统、磁带，或仅一个文件。

第 5 章“ZFS 卷”概述 ZFS 卷（zvol）的基础知识，然后深入探讨一些常见陷阱。

第 6 章“高级硬件”涵盖不同类型的硬件、它们与 ZFS 的交互，包括 SCSI 机箱、主机总线适配器（HBA）、SAS 多路径、固态硬盘（SSD）和非易失性内存 Express（NVMe）。

第 7 章“缓存”介绍 ZFS 的缓存机制，包括对 ZFS 自适应替换缓存（ARC）的深入讨论，这部分占了本章大部分篇幅。

第 8 章“性能”详细介绍如何针对特定环境提升 ZFS 性能。本章内容相当深入，也是我最喜欢的一章。它讨论了如何评估 ZFS 性能、ZFS 的预取系统、事务组（txg）调优、I/O 调度、I/O 队列、写入节流等。

第 9 章“调优”某种程度上是第 8 章的延续，介绍如何针对环境调整 ZFS。我特别喜欢关于 ZFS 与 MySQL 和 PostgreSQL 交互的讨论。

第 10 章“ZFS 杂谈”收录了若干简短的 ZFS 使用指南，包括将镜像存储池拆分为多个相同的存储池、删除相邻的多个快照、恢复被销毁的存储池、使用 ZFS 调试器（zdb）、详细检查数据集等。

## 评价

与 Lucas 的其他作品一样，本书中幽默随处可见。例如第 195 页有四个脚注，记录了两位作者之间的来回争论。这些小插曲有助于打破密集内容的沉闷；不过有些读者可能不喜欢。

总而言之，*FreeBSD Mastery: Advanced ZFS* 对合适的读者而言是一本出色的书。如果你已经掌握了 ZFS 的基础知识并想进一步学习，这本书就是为你准备的。

---

**JOSEPH KONG** 是自学成才的计算机爱好者，涉猎漏洞利用开发、逆向代码工程、rootkit 开发和系统编程（FreeBSD、Linux 和 Windows）。他是广受好评的《Designing BSD Rootkits》和《FreeBSD 设备驱动程序开发》的作者。目前是 EMC Isilon 部门的高级软件工程师。更多 Joseph Kong 的信息，请访问 <www.thestackframe.org> 或在 Twitter 上关注 @JosephJKong。
