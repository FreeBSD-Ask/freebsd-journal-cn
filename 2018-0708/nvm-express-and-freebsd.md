# NVM Express 与 FreeBSD

作者：**Jim Harris** 与 **Warner Losh**

NVM Express（NVMe）已迅速成为通过 PCI Express 进行高性能非易失性存储器访问的主导标准。FreeBSD 于 2012 年加入对 NVMe 的支持，使 FreeBSD 能够利用可提供超过 500,000 IO/s 的 NVMe 设备 1。Netflix 等公司迅速转向在 FreeBSD 上部署 NVMe 存储，使得 FreeBSD 的 NVMe 子系统现在帮助驱动北美洲互联网流量的很大一部分 2。在本文中，我们描述 NVMe 规范以及 FreeBSD 对该规范的实现，并提供 FreeBSD 用于监控和管理 NVMe 存储的实用工具概览。

让我们首先描述 NVMe 中使用的一些常见术语。NVMe SSD 被称为 NVMe 控制器，大致相当于 SCSI HBA 或 AHCI 控制器。主要差异在于，对于 NVMe，介质位于控制器本身内部——控制器与其介质之间没有单独的协议或线缆。NVMe 控制器内的存储介质被分组为一个或多个 NVMe 命名空间。命名空间可以被认为大致相当于一个 SCSI LUN。这种将 HBA 和介质组合到一个单元中的做法通过简化所需协议并消除抽象层来减少开销。该设计利用 PCIe 的能力，通过支持无锁请求排队来消除驱动瓶颈。

在 FreeBSD 中，NVMe 控制器和命名空间通过 **nvme(4)** 驱动程序枚举和初始化。**nvme(4)** 还负责提供接口，将这些命名空间作为块设备公开，但不直接向 GEOM 或 CAM 注册这些命名空间。**nvd(4)** 将每个命名空间向 GEOM 注册为块设备。自 **nvd(4)** 原始开发以来 NVMe 已成为主流，Netflix 添加了 **nda(4)** 作为 **nvd(4)** 的替代方案。**nvd(4)** 驱动程序是 NVMe 协议之上非常薄的一层，旨在高事务速率下运行。**nda(4)** 驱动程序与 CAM 集成，包括其错误恢复和高级排队。它还通过 I/O 调度器向驱动器提供流量整形，以提升整体性能。

## NVMe 系统集成

从这张理想化的图中可以看到，**nvd(4)** 直接与磁盘层集成。这有利于将命令排队到 NVMe 驱动器，延迟尽可能小。由于 NVMe 驱动器设计为扩展，这相当有效。获取 I/O 到驱动器的最小延迟意味着更低的延迟和更简单的代码路径。相比之下，**nda(4)** 驱动程序通过 CAM 连接。CAM 调度 I/O 到驱动器。这可能涉及重新排序命令。CAM 系统设计为接受高队列
速率，以便当 I/O 可用时将排队的 I/O 发送到驱动器。CAM 还添加了错误处理，
当出现问题时可以比 **nvd(4)**（其错误路径中几乎没有错误恢复）更好地恢复驱动器。**nda(4)** 还可以通过 CAM I/O
调度器在一定程度上对驱动器进行 I/O 流量整形。这可以用于引入一些不公平性以产生更好的结果。例如，视频
流受益于偏向读取操作。许多 NVMe 驱动器在 TRIM 方面存在问题，但很快将能够对发送到驱动器的 TRIM 进行大量节流或整形。**nda(4)** 能够将多个 TRIM
合并到驱动器的一个 TRIM 请求中。
  **nvme(4)** 驱动程序负责在附加驱动程序时枚举 nvme 驱动器及其命名空间。**nvd(4)** 或 **nda(4)** 驱动程序在启动时通过
nvme_register_consumer() 调用注册对这些（和其他）事件的兴趣。因此，当它检测到新命名空间时，会调用新
命名空间回调。对于 **nvd(4)** 驱动程序，会创建并向 GEOM 注册新的磁盘接口。对于
**nda(4)** 驱动程序，回调转到 nvme_sim，后者创建适当的 CAM 设备，导致
**nda(4)** 被创建（CAM 的 SCSI 遗产意味着它通过间接或延迟回调做许多事情）。**nda(4)** 驱动程序然后创建磁盘并向 GEOM 注册。
   无论磁盘接口如何，当请求来自系统上层（具体来说来自
   通过称为 bios 的命令结构的磁盘层）时，nvd 或 nda 将把 BIO_READ、BIO_WRITE
和 BIO_DELETE 命令转换为适当的 nvme 命令，并将这些请求传递给 **nvme(4)**
驱动程序执行。
   通过 **nvd(4)** 还是 **nda(4)** 公开 NVMe 命名空间由 `hw.nvme.use_nvd` 可调
参数控制。它目前默认为 1，意味着命名空间通过 **nvd(4)** 公开。

## NVMe 命令和队列对

READ、WRITE 和 IDENTIFY 等 NVMe 命令由主机通过提交
队列提交给控制器，控制器通过完成
队列通知主机这些命令的完成。规范允许多个提交队列共享一个完成队列，但
FreeBSD 始终将一个提交队列与一个完成队列关联，形成逻辑队列对或
“qpair”。这些 qpair 关联对于在多核系统上解锁 NVMe 并行性至关重要，将
在本文稍后描述。
  提交队列是一个连续的主机内存区域，作为循环缓冲区。每个提交队列
  条目 64 字节。主机通过填写队列中的下一个条目
  然后写入控制器 MMIO 寄存器空间中每个队列的门铃来通知控制器新命令。在 **nvme(4)** 中，这在
  nvme_qpair_submit_tracker() 中完成：

```c
696     /* 将命令从追踪器复制到提交队列。 */
697     memcpy(&qpair->cmd[qpair->sq_tail], &req->cmd, sizeof(req->cmd));
698
699     if (++qpair->sq_tail == qpair->num_entries)
700             qpair->sq_tail = 0;
701
702     wmb();
703     nvme_mmio_write_4(qpair->ctrlr, doorbell[qpair->id].sq_tdbl,
704         qpair->sq_tail);
```

完成队列是一个独立的连续主机内存区域，16 字节条目。控制器通过填写完成队列中的下一个条目来通知主机命令已完成。此条目包含提交队列的 ID 和主机提交命令时使用的每个队列的命令 ID。然后控制器中断主机，驱动程序可以开始处理完成条目。NVMe 在每个完成队列条目中定义一个相位位，使主机能够确定哪些完成条目有效。控制器第一次通过队列时会将此相位位写入 1，然后在随后的队列遍历中在 0 和 1 之间交替。驱动程序根据期望值检查每个完成队列条目的相位位，以确定哪些条目包含新的完成。在 **nvme(4)** 中，这在 nvme_qpair_process_completions() 中完成：

```c
424      while (1) {
425              cpl = &qpair->cpl[qpair->cq_head];

426
427              if (cpl->status.p != qpair->phase)
428                      break;
429
```

这种完成队列处理方法使主机只需读取内存即可确定哪些命令已完成。这确保性能代码路径中没有 MMIO 读取。它还利用 Intel® 的 Data Direct I/O 技术 3 等 CPU 特性，将完成条目直接放入最后一级缓存以优化完成条目处理。NVMe 命令进一步分为两种类型——admin 和 I/O——由两种不同类型的 qpairs——也命名为 admin 和 I/O——处理。前面描述的队列处理同样适用于 admin 和 I/O qpairs，但初始化它们的方法截然不同。了解 NVMe 控制器如何初始化将有助于理解这一点。

## NVMe 控制器初始化

高层而言，控制器初始化分为两个阶段：

- 控制器重置——此阶段由主机使用 NVMe MMIO 寄存器读取和写入执行。其主要功能是描述 admin qpair 的参数（主机内存地址和队列大小），然后切换启用位（CC.EN）。切换启用位后，控制器将开始其内部初始化，并在完成时设置就绪位（CSTS.RDY）。主机等待就绪位变为设置，然后继续下一阶段。
- 控制器设置——此阶段由主机使用现在可以在上一阶段设置的 admin qpair 上提交的 admin 命令执行。
  - 提交 CDW10（command dword 10）设置为 1 的 IDENTIFY 命令。这表示控制器返回与控制器关联的 IDENTIFY 数据。
  - 提交 CREATE_IO_CQ 和 CREATE_IO_SQ 命令以构造 I/O qpairs。分配的 I/O qpairs 数量通常等于系统上的 CPU 核心数与控制器支持的最大 I/O qpairs 数的最小值。这将在本文稍后更详细地描述。
  - 为控制器 IDENTIFY 数据报告的每个命名空间提交 IDENTIFY 命令。命名空间 IDENTIFY 数据中的关键信息是命名空间的大小和格式。命名空间在 IDENTIFY 命令中使用 NSID 字段指定，它总是从 1 开始（永远没有命名空间 0）！

## I/O 队列分配

I/O qpair 分配是 **nvme(4)** 驱动程序的关键特性。有了许多 I/O qpairs，我们理想情况下可以为每个 CPU 核心分配一个单独的 qpair。这使每个 CPU 核心上的线程能够提交 I/O 命令而无需与在其他 CPU 核心上运行的线程同步。它还允许将完成中断绑定到提交 I/O 的 CPU 核心，以改善缓存局部性。

实际上，由于几个原因，每个核心 qpair 可能无法实现。首先，虽然 NVMe 规范允许每个控制器最多 65,535 个 I/O qpairs，但大多数 NVMe SSD 允许的 I/O qpairs 要少得多——有时少于 32 个。其次，系统上可能有有限数量的中断向量可用。后一种限制在拥有许多 NVMe SSD 和多队列 NIC 的系统上最常出现，但得益于一些 SMP 改进，现在在 FreeBSD 11 4 中应该不太可能发生。

对于无法为每个 CPU 核心分配 qpair 的情况，**nvme(4)** 将尽可能多地分配 qpair，然后将每个 qpair 与多个 CPU 核心关联。使用 mutex 同步对 qpair 的访问——不仅在提交 I/O 的不同线程之间，还与 qpair 的完成处理程序之间。

## I/O 提交

NVMe 命令主要由以下部分组成，由 FreeBSD 的 struct nvme_command 表示，位于 **/usr/include/dev/nvme/nvme.h**：

• 8 位操作码——admin 和 I/O 操作码重叠，但根据其提交
  队列类型容易区分
• 16 位命令 ID——此命令 ID 必须在同一队列上提交的任何其他命令中
  唯一
• 命名空间标识符——主要用于 I/O 命令，但某些 admin 命令如 IDENTIFY
  也使用此字段；注意这意味着可以从任何 I/O 队列向任何命名空间提交 I/O
• 两个物理区域页面（PRP）条目

  PRP 是 NVMe 版本的分散-收集列表（SGL）。不是指定起始地址和长度的分散-收集元素，
  PRP 条目仅指定起始地址。长度由
起始地址距
下一页的距离和 I/O 命令的整体大小推断。它也有局限性，例如
所有 PRP 条目（第一个除外）必须从 4KiB 边界开始，所有条目（第一个和最后一个除外）必须
描述正好 4KiB 长度的缓冲区。幸运的是，这些限制不影响 FreeBSD，因为 FreeBSD
I/O 栈始终使用虚拟连续缓冲区，这始终可以由 PRP 表示。
  细心的读者会注意到，NVMe 命令只有两个 PRP 条目——那么我们如何表示
  大命令的 I/O 缓冲区？在这些情况下，第二个 PRP 条目指向
  其他 PRP 条目的列表。但这意味着对于大型 I/O，我们需要为 PRP 列表分配缓冲区并进行 DMA 映射。

nvme_request 和 nvme_tracker
PRP 列表有两个注意事项：
• PRP 列表缓冲区需要为 DMA 映射，因此 PRP 列表缓冲区在分配 qpair 时分配和映射。这避免了在性能路径中映射 PRP 列表。
• PRP 列表缓冲区在某些环境中可能变得很大。PRP 列表缓冲区的大小与 MAXPHYS 成正比。FreeBSD 默认 MAXPHYS 为 128KiB，导致最大 PRP 列表为 256 字节。Netflix 部署时 MAXPHYS 为 1MiB，导致 2KiB PRP 列表缓冲区大小。因此，我们需要审慎决定每个 qpair 分配多少 PRP 列表。nvme 驱动程序自动将请求大小限制为 MAXPHYS 和 2MiB 中较小者，这是适合 SG 列表可用的两个 PRP 页面的最大缓冲区。
  **nvme(4)** 定义两种不同的结构——struct nvme_request 和 struct nvme_tracker。
  当分配 I/O qpair 时，分配 128 5 个 nvme_trackers，以及每个
nvme_tracker 的 PRP 列表缓冲区。这表示在
任何给定时间 qpair 上可能未完成的 I/O 最大数量。
   struct nvme_request 表示调用者已请求的命令。最简单的情况是
nvme_ns_cmd_read()：

```c
 32 int
 33 nvme_ns_cmd_read(struct nvme_namespace *ns, void *payload, uint64_t lba,
 34     uint32_t lba_count, nvme_cb_fn_t cb_fn, void *cb_arg)
 35 {
 36         struct nvme_request   *req;
 37
 38         req = nvme_allocate_request_vaddr(payload,
 39           lba_count*nvme_ns_get_sector_size(ns), cb_fn, cb_arg);
 40
 41         if (req == NULL)
 42                return (ENOMEM);
 43
 44         nvme_ns_read_cmd(&req->cmd, ns->id, lba, lba_count);
 45
 46         nvme_ctrlr_submit_io_request(ns->ctrlr, req);
 47
 48         return (0);
 49 }
```

此处调用者请求从命名空间读取数据到缓冲区 `payload` 中。首先，
nvme_allocate_request_vaddr() 使用调用者指定的参数分配一个 nvme_request 结构。
  接下来，nvme_command 结构由 nvme_ns_read_cmd() 填充。
注意 nvme_command 是一个 64 字节的提交队列条目，但此时我们只是准备提交队列条目——它稍后将被复制到实际提交队列中。最后，我们调用
nvme_ctrlr_submit_io_request()。

```c
   1207  void
   1208  nvme_ctrlr_submit_io_request(struct nvme_controller *ctrlr,
   1209      struct nvme_request *req)
   1210  {
   1211         struct nvme_qpair        *qpair;
   1212
   1213         qpair = &ctrlr->ioq[curcpu / ctrlr->num_cpus_per_ioq];
   1214         nvme_qpair_submit_request(qpair, req);
   1215  }
```

这里我们根据当前 CPU 选择 qpair。既然知道使用哪个 qpair，我们调用 nvme_qpair_submit_request()。此函数检查是否有可用的 nvme_trackers。如果有，它调用我们之前看到的 nvme_qpair_submit_tracker()。如果没有，它将 nvme_request 放入 STAILQ。稍后，一旦某些 I/O 完成，将检查此 STAILQ 并重新提交 nvme_requests。

## NVMe 命名空间

NVMe 规范允许将驱动器逻辑分区为单独的命名空间。命名空间只是一组块，地址为 0..N-1。这些命名空间可以完全配置或精简配置（实际上，模拟 NVMe 驱动器的硬件或虚拟机管理程序通常向主机隐藏这些细节）。命名空间还提供向这些命名空间独立分配属性的方法。

一个命名空间可用于存储操作系统。数据很少变化，通常写少，但读多。另一个命名空间可能包含事务日志，很少读但主要以追加 I/O 模式写入。还有另一个可能包含一直在重写的非常热的数据。NVMe 驱动器上的固件可以使用这些属性优化 NAND 存储。对于冷数据（如操作系统），固件可能将其放入磨损低且每单元存储 3 位的单元，以最大化数据密度。这些单元通常具有出色的长期保留能力。对于非常热的数据，驱动器可能选择使用更磨损的单元，并可能将其大量存储在每单元存储 1 位的单元中，以最大化速度。虽然磨损的单元无法保留数据很长时间，但它们适合热数据，因为数据不会存储很长时间，因此长寿方面的任何缺陷都不会阻碍驱动器的性能。

FreeBSD 尚不支持命名空间管理——例如创建和删除命名空间以及为这些命名空间指定属性。FreeBSD 社区正在此领域积极开展工作，应该能在 FreeBSD 12 之前完成。

## 管理 NVMe 驱动器

用于列出和配置 NVMe 控制器和命名空间的主要实用工具是 **nvmecontrol(8)**。最基本的 nvmecontrol 子命令是“nvmecontrol devlist”，它提供每个 NVMe 控制器及其命名空间的简短摘要。

```sh
         %sudo nvmecontrol devlist
         nvme0: ORCL-VBOX-NVME-VER12
          nvme0ns1 (1024MB)
```

其他 **nvmecontrol(8)** 子命令包括：

- “nvmecontrol identify”用于根据 NVMe IDENTIFY 命令的信息提供 NVMe 控制器和命名空间的详细信息
- “nvmecontrol logpage”用于从 NVMe 控制器读取日志页面；规范定义的日志页面（错误、Health/SMART 和固件插槽）有处理程序将日志页面转换为人类可读的格式
- “nvmecontrol firmware”用于下载和/或在 NVMe 控制器上激活不同的固件映像
- “nvmecontrol perftest”用于从 **nvme(4)** 驱动程序本身运行低级性能测试
- “nvmecontrol reset”向 NVMe 控制器发出控制器级重置
- “nvmecontrol power”更改电源状态或为 NVMe 控制器指定工作负载提示
- “nvmecontrol wdc”执行特定于 WDC NVMe SSD 的选项

使用 **nda(4)** 时，**camcontrol(8)** 目前可用于列出命名空间，但其他功能（如 NVMe identify 或固件下载）尚未管道化。

**nvmecontrol(8)** 的一个缺点是将 NVMe 控制器或命名空间映射到其关联的 nvd 或 nda 条目。目前最佳方法是通过“geom disk list”和“nvmecontrol identify”之间的序列号关联。随着 **camcontrol(8)** 获得更多 NVMe 功能，此问题将得到缓解。

## 总结

NVM Express 提供了一个现代的、高性能的存储接口，非常适合当今的 CPU 架构——FreeBSD 能够很好地利用它。将 NVMe 支持与 CAM 集成，以及未来对命名空间管理和 NVMe SSD 热插拔的支持，将进一步改善 FreeBSD 的 NVMe 能力。下次你看 Netflix 时，希望你能对这些位如何出现在你的屏幕上了解得更多一点！ •

1 <https://www.intel.com/content/www/us/en/solid-state-drives/optane-ssd-900p-brief.html>
2 <https://www.sandvine.com/hubfs/downloads/archive/2016-global-internet-phenomena-report-latin-america-and-north-america.pdf>
3 <https://www.intel.com/content/www/us/en/io/data-direct-i-o-technology.html>
4 <https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=199321>
5 128 是默认值。此数字可通过 `hw.nvme.io_trackers` 可调参数修改。

---

**Jim Harris** 是 Intel 数据中心小组的首席软件工程师，于 2011 年获得 FreeBSD 源代码提交权限。他是 FreeBSD nvme 和 nvd 驱动程序的原始作者，也是 nvmecontrol 管理实用程序。Jim 还帮助将库和工具（如 DPDK 和 Intel VTune Amplifier）带到 FreeBSD，并将 FreeBSD nvme 驱动程序移植到用户空间，作为他目前担任存储性能开发套件软件架构师的角色。

**Warner Losh** 是 Netflix 的高级软件工程师，负责 FreeBSD 存储系统的视频交付服务器优化。他作为 FreeBSD 贡献者已超过 20 年，目前在 FreeBSD 核心团队任第六届任期。Warner 改进了很多系统——最近改进了 FreeBSD 的引导加载程序。加入 Netflix 之前，他生产闪存驱动器并为准确性测量原子钟。他的代码仍在测量一些创建 UTC 的时钟！
