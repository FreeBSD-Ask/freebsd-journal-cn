# 朴素的网桥

朴素的网桥。连接河流、沟壑、峡谷、山谷两岸的桥梁……等等，我们说的不是这种桥。`if_bridge` 把两条或更多以太网链路（IEEE 802 网络，如果你较真的话）连起来，相当于把你的电脑变成一台交换机，还不用一堆乱糟糟的线缆。

`if_bridge` 的常见用法之一是把多台虚拟机（或 VNET Jail）接到真实网络接口上。

例如，这台机器上跑了几台 bhyve 虚拟机：

```sh
vmbridge: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
     ether 02:ec:9a:db:86:01
     inet 172.16.1.1 netmask 0xffffff00 broadcast 172.16.1.255
     id 00:00:00:00:00:00 priority 32768 hellotime 2 fwddelay 15
     maxage 20 holdcnt 6 proto rstp maxaddr 2000 timeout 1200
     root id 00:00:00:00:00:00 priority 32768 ifcost 0 port 0
     member: tap5 flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
             ifmaxaddr 0 port 11 priority 128 path cost 2000000
     member: tap4 flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
             ifmaxaddr 0 port 10 priority 128 path cost 2000000
     member: tap2 flags=143<LEARNING,DISCOVER,AUTOEDGE,AUTOPTP>
             ifmaxaddr 0 port 7 priority 128 path cost 2000000
     groups: bridge
     nd6 options=9<PERFORMNUD,IFDISABLED>
```

这些虚拟机连到 tap2、tap4、tap5 接口。它们都能在该（虚拟）局域网上相互通信，或通过 172.16.1.1 地址与主机通信。主机还贴心地跑着 DHCP 服务器和 NAT 防火墙，把虚拟机的流量转换到公共接口，让它们能访问互联网。

## 问题

这套配置没什么特别不对，但要是找不出问题，这篇文章就太短太无聊了。

好在我们有问题——至少对某些用户来说可能有问题。它不显眼，但 `if_bridge` 的扩展性不够好。事实上，目前的扩展性非常差。

为了演示问题，我们搭个简单测试环境：两台机器用两条 Chelsio T62100 40Gbps 链路相连。第一台机器从第一个接口发出流量，并期望从第二个接口收到。第二台机器用 `if_bridge` 把两个接口连起来。

Chelsio 驱动支持 `netmap(4)`，意味着我们可以用 pkt-get（pktgen-g2017.08.06_1）生成流量。`netmap(4)` 很快。我不打算证明这一点——你姑且信我。是的，真的。如果你怀疑我说的每一句话，我们走不下去的！

于是，用下面两条命令（第一条接收流量，第二条生成流量），就能跑一次简单的性能测试：

```sh
./pkt-gen -i vcc1 -f rx -w 4 -W
./pkt-gen -f tx -i vcc0 -l 60 -d 192.19.10.1:2000-192.19.10.10 -s 10.10.10.1:2000-10.10.19.143 -S 01:02:03:04:05:06 -D 06:05:04:03:02:01 -w 4
```

我们生成 60 字节的包。这有充分理由，不只是为了找别扭。好吧，实际上就是为了找别扭。一般而言，机器传输少量大包比传输大量小包更轻松。这容易理解，因为每个包都有一定开销（如上下文切换、判断发往何处、至关重要的包间小憩……）。所以包越大越高效。真实网络中你想要大包，但测试时我们关注最坏情况，因为这时改进的效果最明显。

测试运行后，会看到类似这样的输出：

```sh
957.106303 main_thread [2666] 27.015 Mpps (28.730 Mpkts 12.967 Gbps in 1063499 usec) 497.42 avg_batch 99999 min_space
957.306302 main_thread [2666] 3.668 Mpps (3.901 Mpkts 1.761 Gbps in 1063498 usec) 24.62 avg_batch 949 min_space
958.169302 main_thread [2666] 27.016 Mpps (28.718 Mpkts 12.967 Gbps in 1062999 usec) 497.42 avg_batch 99999 min_space
958.369303 main_thread [2666] 3.669 Mpps (3.900 Mpkts 1.761 Gbps in 1063001 usec) 24.68 avg_batch 885 min_space
959.232304 main_thread [2666] 27.015 Mpps (28.717 Mpkts 12.967 Gbps in 1063002 usec) 497.37 avg_batch 99999 min_space
959.377805 main_thread [2666] 3.669 Mpps (3.700 Mpkts 1.761 Gbps in 1008502 usec) 24.66 avg_batch 950 min_space
960.280311 main_thread [2666] 27.011 Mpps (28.308 Mpkts 12.965 Gbps in 1048007 usec) 497.44 avg_batch 99999 min_space
960.441302 main_thread [2666] 3.668 Mpps (3.901 Mpkts 1.761 Gbps in 1063496 usec) 24.66 avg_batch 921 min_space
...
```

我们其实看到了两个 `pkt-gen` 进程的输出：一个来自发送进程，一个来自接收进程。我们设法每秒发送 2700 万包（累加起来接近 13 Gbps），但只收到 370 万包。其余的包都丢了。

为这些可怜的丢失包默哀 370 万个短暂瞬间后，我们可以问为什么。如果你只从本文记住一件事，请记住：做基准测试时最重要的提问是"为什么？""为什么我们看到这个数字？""为什么不是更高？""为什么不是更低？"

我恰好知道——你姑且信我——如果把这些流量路由而非交换，能转发多得多的包。这看起来奇怪，因为路由比交换更费事。

幸好 FreeBSD 有出色的工具，帮我们搞清系统时间花在哪里。这类工作中，`pmcstat(8)` 工具无可替代。

下面会生成一张极具启发性的火焰图：

```sh
pmcstat -S cpu_clk_unhalted.ref_tsc -l 30 -z 50 -O data.out -q
pmcstat -R data.out -G data.stacks
stackcollapse-pmc.pl data.stacks > data.folded
flamegraph.pl data.folded > data.svg
```

`pmcstat` 是 FreeBSD 基本系统的一部分。`stackcollapse-pmc` 和 `flamegraph` 工具出自 Brendan Gregg（<https://github.com/brendangregg/FlameGraph>）。

不管怎样，看图时间到：

这图看起来有点怪，但它展示的最重要信息是系统时间花在哪里。纵轴是调用栈。从下往上读，它告诉我们 `fork_exit()` 调用了 `ithread_loop()`，后者又调用了……

调用栈底部其实不太有意思。

我们来弄清系统本可以转发包时在做什么。找一个似乎耗时不短的函数。最宽的函数——也就是我们花最多时间的地方——是 `bridge_input()`。我们 93% 的时间花在那里。

它看起来和网桥工作有关。为什么我们花这么多时间在里面？我们在 `bridge_input()` 里的大部分时间其实是在执行 `__mtx_lock_sleep()`，而不是像 `bridge_forward()` 那样有用的事。

我们不该打盹。为什么我们在睡觉？还有，`mtx` 是什么？

## 互斥锁

`__mtx_lock_sleep()` 是 `mutex(9)` 代码库的一部分。函数名里写作 `mtx`，是因为程序员懒得写全。互斥锁提供互斥。它们帮助确保同一时刻只有一个 CPU 核能访问特定内存。为什么需要这个？

最简单的例子：想象两个 CPU 核同时对一个数字加一。再想象一个总是出岔子的宇宙，吐司总是涂黄油面朝下落地。这想象可能要费点劲。不？你已经身处其中了？太好了。

那么，可能发生这样的操作顺序：

```sh
CPU1: Load number (a = 1) from memory to a register (r0 = 1)
CPU2: Load number (a = 1) from memory to a register (r1 = 1)
CPU1: Increment register (r0 = 2)
CPU1: Store register (r0 = 2) to memory (a = 2)
CPU2: Increment register (r1 = 2)
CPU2: Store register (r1 = 2) to memory (a = 2)
```

糟糕。突然 1 + 1 + 1 等于 2。解决这个问题的一种办法是确保 CPU2 不能同时运行这段代码，这正是互斥锁帮我们实现的。

```sh
CPU1: Take mutex
CPU2: Try to take mutex. Fail and wait
CPU1: Load number (a = 1) from memory to a register (r0 = 1)
CPU1: Increment register (r0 = 2)
CPU2: Try to take mutex. Fail and wait
CPU1: Store register (r0 = 2) to memory (a = 2)
CPU1: Release mutex
CPU2: Take mutex
CPU2: Load number (a = 2) from memory to a register (r1 = 2)
CPU2: Increment register (r1 = 3)
CPU2: Store register (r1 = 3) to memory (a = 3)
CPU2: Release mutex
```

`if_bridge(4)` 代码中还有更复杂的流程，但这引出了互斥锁的用途。

互斥锁的问题在于，它们虽然确保内部数据结构同一时刻只能被一个 CPU 核访问，但这也意味着同一时刻只能在一个核上做有用功。

事实证明，`if_bridge(4)` 用单个互斥锁保护它所有的内部数据结构（比如"此网桥连接 igb0 和 igb1"），所以无论你有多少 CPU 核，同一时刻只有一个能通过网桥转发包。

结果，我们每秒只能转发约 390 万包。

## 改进

一种可能的改进方式是把互斥锁改为读写锁（见 `rwlock(9)`）。读写锁区分读访问和写访问。如果代码只读数据不修改它，我们可以允许多个 CPU 核同时读取。只有当数据将被修改时，我们才需要禁止其他所有 CPU 核访问，包括只想读数据的那些。

这已经能大幅改善。沿此思路的原型把吞吐量提升到每秒约 800 万包。

## 做得更好

FreeBSD 13（尚未发布，目前为 CURRENT）带来了处理此类问题的新方式。它叫 `epoch(9)`，由比我聪明得多的人设计和实现。

为本文起见，假设我其实懂它，这样我能解释它做什么。

`epoch(9)` 设施源自 ConcurrencyKit（<http://concurrencykit.org>）。配合 ConcurrencyKit 列表（`ck_queue`），我们无需获取锁（互斥锁或读写锁皆可）就能安全使用受保护的数据结构。

对于 `if_bridge(4)`，主要数据结构是列表，典型的修改是向列表添加或从中移除条目。

向列表添加条目时，我们需要确保列表本身始终处于有效状态，但不太关心某个 CPU 核是否能看到该条目。这意味着只要我们小心，先把新数据结构完全填充好再插入列表，就能在其他核遍历列表时安全插入。

从列表移除条目时，需要更加小心。我们可以把数据从列表移除，但在确认无人再使用之前不能真正删除它。

这正是 `epoch(9)` 设施的用途。我们要读受保护数据时，通过 `NET_EPOCH_ENTER()` 包装器通知 `epoch(9)` 系统我们将访问这些数据。访问完，通过 `NET_EPOCH_EXIT()` 通知它。

这让系统能跟踪谁可能还在访问我们将要移除的数据。简单说，我们告诉它要删除某物时（通过用 `NET_EPOCH_CALL()` 注册执行最终清理的回调），它会等待每个调用过 `NET_EPOCH_ENTER()` 的 CPU 核也调用 `NET_EPOCH_EXIT()`。

到那一刻我们能确信无人还在使用该数据，可以安全删除。

还有一些额外复杂性，因为 ConcurrencyKit 列表在被修改时可安全读取，但不能同时有多个 CPU 核尝试修改。

在 `if_bridge(4)` 中，我们用现有互斥锁保护对这些列表的写访问。最终结果是我们仍然一次只能对网桥状态做一次修改（例如添加接口或学习某 MAC 地址使用哪个接口），但修改期间其他 CPU 核能继续处理包。

记住单个 CPU 核每秒能处理 370 万包，所以可以放心假设每秒也能处理数十万乃至数百万次状态变更。

## 最终测量

采用基于 epoch 的方案后，性能测试显示：

```sh
283.745634 main_thread [2666] 18.637 Mpps (18.673 Mpkts 8.946 Gbps in 1001923 usec) 82.70 avg_batch 800 min_space
284.108710 main_thread [2666] 26.982 Mpps (27.009 Mpkts 12.951 Gbps in 1001000 usec) 494.37 avg_batch 99999 min_space
284.746709 main_thread [2666] 18.620 Mpps (18.640 Mpkts 8.938 Gbps in 1001076 usec) 82.80 avg_batch 816 min_space
285.172211 main_thread [2666] 26.981 Mpps (28.695 Mpkts 12.951 Gbps in 1063501 usec) 494.47 avg_batch 99999 min_space
285.747709 main_thread [2666] 18.640 Mpps (18.659 Mpkts 8.947 Gbps in 1001000 usec) 82.84 avg_batch 805 min_space
286.235212 main_thread [2666] 26.976 Mpps (28.676 Mpkts 12.949 Gbps in 1063001 usec) 494.34 avg_batch 99999 min_space
286.748709 main_thread [2666] 18.638 Mpps (18.656 Mpkts 8.946 Gbps in 1001000 usec) 83.01 avg_batch 800 min_space
287.236712 main_thread [2666] 26.976 Mpps (27.017 Mpkts 12.949 Gbps in 1001500 usec) 494.41 avg_batch 99999 min_space
287.750709 main_thread [2666] 18.633 Mpps (18.671 Mpkts 8.944 Gbps in 1002000 usec) 83.05 avg_batch 803 min_space
288.300211 main_thread [2666] 26.977 Mpps (28.690 Mpkts 12.949 Gbps in 1063499 usec) 494.41 avg_batch 99999 min_space
...
```

所以，我们现在每秒转发约 1860 万包，提升 5 倍。也有一张对应的漂亮图（火焰图）：

这里我们看到，我们不再把全部时间（其实是任何时间）花在 `__mtx_lock_sleep()` 上，而是在做有用功。几乎全部时间都用在直接与"判断包要去哪里并发送出去"相关的工作上。

## 结语

本文讨论的是进行中的工作，所以如果你发现 bug：干得漂亮！告诉作者，他可能还没注意到。

感兴趣的读者可在以下地址找到待提交的补丁：

- <https://reviews.freebsd.org/D24249>
- <https://reviews.freebsd.org/D24250>

本项目由 FreeBSD 基金会赞助支持。

KRISTOF PROVOST 是一名自由嵌入式软件工程师，专注于网络和视频应用。他是 FreeBSD 提交者、FreeBSD 中 pf 防火墙的维护者，以及 EuroBSDCon 基金会董事会成员。他有个不幸的倾向——会撞上 uClibc 的 bug，并且对 FTP 痛恨至极。别和他谈 IPv6 分片。

Jail 是 FreeBSD 最传奇的特性：
威力强大、难以驾驭，
数十年来笼罩在层迷雾传说中。

- 在 Jail 的限制内游刃有余
- 精细控制 Jail 各项特性
- 搭建虚拟网络
- 部署分层 Jail
- 约束 Jail 资源使用
- ……以及更多更多！

《FreeBSD Mastery: Jails》作者 MICHAEL W LUCAS，各大书店有售

《FreeBSD Mastery: Jails》拨开迷雾，
揭示 Jail 的内部机制，让它的威力为你所用。

约束你的软件！

- 理解 Jail 如何实现轻量级虚拟化
- 理解基本系统的

```sh
jail 工具与 iocage 工具包
```

- 优化硬件配置
- 从主机和 Jail 内部管理 Jail
- 优化磁盘空间，支持数千 Jail

约束你的软件！
