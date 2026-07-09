# SVN 动态

作者：Steven Kreuzer

生活总是有些忙乱，所以我没能跟上 FreeBSD 快节奏的开发。有一天日月相合，不仅我的两个女儿都熟睡了，而且我也有足够的精力不仅睁开眼睛，还能形成连贯的思想。不知道用这些空闲时间做什么，我做了一个正常人都会做的事——开始翻阅 subversion 日志。令我惊讶的是，我发现 FreeBSD 现在支持 pNFS，有全新的 TCP 拥塞算法，甚至 cron 也获得了新功能！

从 projects/pnfs-planb-server 合并 pNFS 服务器代码到 head — <https://svnweb.freebsd.org/changeset/base/335012>

pNFS 服务将读/写操作与所有其他 NFSv4.1 元数据操作分离。希望这种分离允许配置的 pNFS 服务超过单个 NFS 服务器的存储容量和/或 I/O 带宽限制。可以在数据服务器（DSs）中配置镜像，这样 MDS 文件的数据存储文件将镜像在两个或更多 DSs 上。使用此功能时，DS 故障不会停止 pNFS 服务，并且故障的 DS 可以在修复后恢复，同时 pNFS 服务继续运行。虽然双向镜像是常态，但可以设置最多四个或 DSs 数量（取较小者）的镜像级别。元数据服务器始终是单点故障，就像单个 NFS 服务器一样。

允许 sysctl(8) 为单个节点设置数值数组 — <https://svnweb.freebsd.org/changeset/base/330711>

一些节点返回值数组（例如，kern.cp_time）。sysctl(8) 知道如何显示返回多个值的节点的值（它打印每个数值，用空格分隔）。但是，直到现在，sysctl(8) 只能为 sysctl 节点设置单个值。此更改允许 sysctl 接受包含多个值（由空格或逗号分隔）的数字 sysctl 节点的新值。sysctl(8) 将此列表解析为数组，并将该数组作为“新”值传递给 sysctl(2)。

引入 TCP 高精度定时器系统（tcp_hpts）— <https://svnweb.freebsd.org/changeset/base/332770>

这是引入 Rack 和 BBR 的先驱/基础工作，它们使用 hpts 来推进数据包。该功能是可选的，需要启用 TCPHPTS 选项后功能才会激活。使用它的 TCP 模块必须确保它们加载的内核中编译了基础组件。

引入新的重构 TCP 栈 Rack — <https://svnweb.freebsd.org/changeset/base/334804>

RACK（“Recent ACKnowledgment”）是一种用于 TCP 的基于时间的快速丢包检测算法。RACK 使用时间的概念而非数据包或序列计数来检测现代 TCP 实现的丢包，这些实现支持每数据包时间戳和选择性确认（SACK）选项。它旨在替代传统的 DUPACK 阈值方法及其变体，以及其他非标准方法。

AF_UNIX：使 unix 套接字锁定更精细 — <https://svnweb.freebsd.org/changeset/base/333744>

此更改转向使用引用计数，在锁 drop/reacquire 中保证活性。目前在 unix 套接字上的发送在读锁列表锁时严重争用。will-it-scale 中的 unix1_processes 在 6 个进程时达到峰值然后下降。通过此更改，我们看到在 96 个进程时每秒操作数有显著改善。

修复低延迟网络上的虚假重传恢复 — <https://svnweb.freebsd.org/changeset/base/333346>

TCP 的平滑 RTT（SRTT）可能比实际观察到的 RTT 大得多。这可能是因为 hz 将 VM 中可计算的 RTT 限制为 10ms，或使用默认 1000hz 时为 1ms，或者仅仅因为 SRTT 最近纳入了较大的值。如果 ACK 在计算的 badrxtvin（now + SRTT）之前到达：

```c
tp->t_badrxtwin = ticks + (tp->t_srtt >> (TCP_RTT_SHIFT + 1));
```

我们将错误地将 snd_una 重置为 snd_max。如果多个段被丢弃并且这种情况重复发生，传输速率将限制为每 RTO 1MSS，直到我们重新传输所有丢弃的段。

提升线程优先级同时更改 CPU 频率 — <https://svnweb.freebsd.org/changeset/base/333325>

当用户空间线程设置其亲和性以调整其频率时，提升它们的优先级。这避免了具有相同亲和性的 CPU 密集型内核线程在下时钟核心上运行并会“阻塞”powerd 升频核心直到内核线程让出的情况。这可能导致性能不佳以及潜在地卡在 Giant 上。

导入 netdump 客户端代码 — <https://svnweb.freebsd.org/changeset/base/333283>

这是一种系统组件，允许内核在 panic 后将核心转储到远程主机，而不是本地存储设备。服务器组件在 Ports 树中可用。netdump 在无盘系统上特别有用。netdump(4) 手册页包含一些描述协议的详细信息。要使用 netdump，内核必须使用 NETDUMP 选项编译。

改善 VM 页队列可扩展性 — <https://svnweb.freebsd.org/changeset/base/332974>

当前必须同时持有页锁和页队列锁才能在给定页队列中入队、出队或重新排队页。队列锁是许多工作负载中的可扩展性瓶颈。此更改通过批处理队列操作减少页队列锁争用。为了解开页和页队列锁，使用每 CPU 批队列来引用具有待处理队列操作的页。请求的操作编码在页的 aflags 字段中，在持有页锁后，页入队以进行延迟批处理。页队列扫描同样经过优化以最小化持有页队列锁时执行的工作量。

为 cron(1) 添加新功能和语法，以允许作业以给定间隔运行 — <https://svnweb.freebsd.org/changeset/base/334817>

实际目标是避免前一次作业调用重叠到新调用，或避免持续时间长且没有立即启动点的作业间隔过短。间隔作业的另一个有用效果可以在集群中的机器周期性通信时注意到。基于时间运行任务会在节点上造成过多负载。基于间隔运行可分散调用，跨集群中的机器。

使用新的 SO_REUSEPORT_LB 选项负载均衡套接字 — <https://svnweb.freebsd.org/changeset/base/332894>

此补丁添加一个新的套接字选项 SO_REUSEPORT_LB，允许多个程序或线程绑定到同一端口，传入连接将使用哈希函数进行负载均衡。

添加 TCP 黑盒记录器 — <https://svnweb.freebsd.org/changeset/base/331347>

TCP 黑盒记录器允许你在环形缓冲区中捕获 TCP 连接上的事件。它随事件存储元数据。它可选地存储与事件关联的 TCP 头（如果事件与数据包关联），也可选地存储套接字上的信息。它支持在 TCP 连接上设置日志 ID，并使用它关联共享公共日志 ID 的多个连接。你可以以不同模式记录连接。如果你正在与特定连接进行协调测试，可以告诉系统将其置于模式 4（持续转储）。或者，如果你只想监控错误，可以将其置于模式 1（环形缓冲区），并在收到该连接 ID 的错误信号时转储与该连接 ID 关联的所有环形缓冲区。你可以设置将应用于特定传入连接比例的默认模式。你也可以使用套接字选项手动设置模式。

  STEVEN KREUZER 是一名 FreeBSD 开发者和 Unix 系统管理员，对
  复古计算和风冷大众汽车感兴趣。他与妻子、
  女儿和狗住在纽约皇后区。
