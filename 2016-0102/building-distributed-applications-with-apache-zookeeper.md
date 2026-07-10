# 使用 Apache Zookeeper 构建分布式应用

- 原文：[Building Distributed Applications with Apache Zookeeper](https://freebsdfoundation.org/our-work/journal/browser-based-edition/performance-tuning/)
- 作者：**Steven Kreuzer**

分布式系统是相当复杂的庞然大物。虽然每个系统都是为了解决某个独特问题而构建，但它们都有一个共同需求：所有节点之间必须能够以可靠、容错且可扩展的方式相互通信。表面上看，如何构建一套能实现可靠分布式通信与协调的系统似乎是个轻而易举的问题，你也能毫不费力地找到大量学术论文详尽描述这些算法。久而久之，你可能不禁想自己造一套轮子，但是问问任何有过此类经历的人，我敢保证你能听到许多令人胆寒的故事，足以迅速打消你这个念头。鉴于你的时间显然不该用来调试微妙的竞态条件和死锁，更有说服力的做法是部署一套已被各类大型开源项目、高校、各行各业的公司广泛采用的现成方案。Zookeeper 就是这样一个软件，它是 Apache 旗下的项目，专注于构建一套健壮的系统以实现分布式系统原语，让开发者编写分布式应用时能够直接上手使用。它最初由 Yahoo! 创建，如今是 Hadoop 生态中积极开发且至关重要的组成部分。虽然许多 Hadoop 相关项目大量使用 Zookeeper，但除了 Java 之外它没有外部依赖，这使得把它引入你的环境极其简单。

## Zookeeper 数据模型

核心而言，Zookeeper 只提供少数几个基本操作，可以用它们组合出分布式系统中需要的诸多设计模式。应用之间通过层次化的键值存储（称为 zNode）进行通信与协调。这套接口设计得与典型的 UNIX 文件系统非常相似，每个 zNode 既可作为文件（最多存储 1MB 数据），也可作为目录（包含多个子 zNode）。此外，每个 zNode 还附带一些元数据，例如 ACL、创建时间、修改时间以及与之关联的版本号。

Zookeeper 允许创建两种不同类型的 zNode：持久 zNode 和临时 zNode。新建 zNode 时默认为持久 zNode。顾名思义，持久 zNode 会一直保留，直到显式调用 delete 函数将其删除。临时 zNode 仅在创建它的客户端保持连接期间存在。如果客户端会话因崩溃或显式终止而断开，Zookeeper 会删除该 zNode。临时 zNode 在主机与服务发现方面非常有用，也可作为检测分布式系统中故障的简单手段。临时 zNode 可以作为持久 zNode 的子节点，但临时 zNode 本身不能有子 zNode。Zookeeper 中，临时 zNode 始终是叶子节点。

Zookeeper 还提供两个额外组件，与 zNode 配合使用就能以直观方式轻松重现非常复杂的行为。持久或临时节点都可以作为原子序列的一部分，父 zNode 会自动为其分配并维护一个单调递增的序号。Zookeeper 保证这个 10 位数字始终唯一，并且大于父 zNode 下创建的任何其它子 zNode。如果你的应用有此需求，这些顺序 zNode 可以作为构建块，轻松实现分布式锁机制。Zookeeper 还引入了 watch（监视）的概念，允许客户端请求在某个 zNode 发生变化时收到通知。watch 可用作创建异步事件驱动系统的简单机制，也便于实现领导者选举算法。Zookeeper 中的 watch 是一次性触发器。变化发生并发出通知后，客户端必须重新注册 watch 才能收到后续更新的通知。

## 上手运行

Zookeeper 在 2012 年加入 FreeBSD Ports 树，因此要启动并运行一个用于测试和开发的单实例非常简单。Zookeeper 设计成开箱即用，启动所需的配置极少，你准备部署到生产环境之前，应该不需要修改配置文件。

```sh
# pkg install zookeeper
# sysrc zookeeper_enable=YES
zookeeper_enable: -> YES
# cp /usr/local/etc/zookeeper/zoo_sample.cfg /usr/local/etc/zookeeper/zoo.cfg
# service zookeeper start
Starting zookeeper.
#

```

服务启动后，你可以使用 `zkCli.sh` 命令连接服务器，并尝试几个命令验证一切工作正常。

```sh
$ zkCli.sh
Connecting to localhost:2181
Welcome to ZooKeeper!
JLine support is enabled
WATCHER::
WatchedEvent state:SyncConnected type:None path:null
[zk: localhost:2181(CONNECTED) 0] ls /
[zookeeper]
```

我们先创建一个名为“test”的新 zNode，并写入数据“hello_world”。然后列出根 zNode 的内容，就能看到新建的“test”zNode 了。

```sh
[zk: localhost:2181(CONNECTED) 1] create /test hello_world
Created /test
[zk: localhost:2181(CONNECTED) 2] ls /
[test, zookeeper]
```

对 zNode 执行 get 即可读取“test”zNode 的内容。本例中，将返回字符串“hello_world”以及该 zNode 自身的一些元数据。

```sh
[zk: localhost:2181(CONNECTED) 3] get /test
hello_world
cZxid = 0x6
ctime = Tue Dec 29 15:39:07 GMT 2015
mZxid = 0x6
mtime = Tue Dec 29 15:39:07 GMT 2015
pZxid = 0x6
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 11
numChildren = 0
```

“test”zNode 的内容也可以使用 `set` 命令更新。你会注意到命令执行后，`mtime`、`dataVersion` 等元数据也会自动更新。更新完成后，可以再执行一次 `get` 来确认 zNode 内容已修改。

```sh
[zk: localhost:2181(CONNECTED) 4] set /test freebsd_journal
cZxid = 0x6
ctime = Tue Dec 29 15:39:07 GMT 2015
mZxid = 0x7
mtime = Tue Dec 29 15:49:46 GMT 2015
pZxid = 0x6
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 15
numChildren = 0
freebsd_journal
[zk: localhost:2181(CONNECTED) 5] get /test
cZxid = 0x6
ctime = Tue Dec 29 15:39:07 GMT 2015
mZxid = 0x7
mtime = Tue Dec 29 15:49:46 GMT 2015
pZxid = 0x6
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 15
numChildren = 0
```

不再需要该 zNode 时，可以执行 `delete` 将其从数据存储中彻底删除。

```sh
[zk: localhost:2181(CONNECTED) 6] delete /test
[zk: localhost:2181(CONNECTED) 7] ls /
[zookeeper]
```

## 投入生产

虽然以独立模式运行单个 Zookeeper 实例用于测试或快速原型验证完全没问题，但这引入了单点故障，关键任务应用开始依赖它之前需要解决。当你准备将 Zookeeper 引入生产环境时，应当部署为所谓的仲裁模式（quorum mode）。Zookeeper 用于协调分布式应用，而它本身也是一个分布式应用：多台独立机器可以组成集群并选出领导者。这个集群称作 ensemble（合奏），它会把数据复制到所有成员，只要 ensemble 中大多数节点在线，Zookeeper 提供的所有服务就仍然可用。

在规划 ensemble 的需求时，建议至少从 3 台机器起步，但对于大多数环境，强烈建议至少 5 台起步。原因在于，3 节点集群中丢失 1 个节点尚可容忍，因为剩余 2 台仍构成多数。但是假设你为了例行维护从 ensemble 中移除一个节点，而此时另一个节点意外故障，那么仲裁就丢失了，剩余节点会切换到对等选举模式，断开现有客户端连接并拒绝新连接，直到选出新领导者。从至少 5 个节点起步可以避免这种特定的故障场景，让集群能容忍最多 2 个节点故障。虽然 Zookeeper 也支持在丢失仲裁时以只读模式运行，但是否启用此特性主要取决于你的应用需求，因此默认行为是停止提供客户端连接。

Zookeeper 的核心目标是确保数据可靠分布，部署 ensemble 时的另一条建议是机器数量始终保持奇数。这可以避免出现“脑裂”（split-brain）的可能：某些节点与其它节点分隔开来但继续独立运行。这种情况下，这些机器可能与另一半失去同步，故障恢复后集群将不知道如何调和差异。为此，Zookeeper 采用多数计数，并选出新的领导者节点。

Zookeeper 的核心是一套原子消息系统，旨在保持 ensemble 中每个成员同步。被选为领导者的节点接收所有写操作，并负责将这些变更发布给所有其它作为 follower 的成员。Zookeeper 通过确保数据始终按发送顺序投递来保证数据最终在 ensemble 的所有成员间一致。某条消息只有在它之前的所有消息都已投递后才会被投递，虽然两个客户端在某一时刻看到的状态可能略有不同，但它们观察变更的顺序始终一致。

在仲裁模式下，所有节点都持有同一份配置文件的副本，并且知道 ensemble 中其它每台机器。这通过在 **zoo.cfg** 文件中追加形如 `server.id=host:quorum_port:election_port` 的行来实现。

```sh
# cat /usr/local/etc/zookeeper/zoo.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/var/db/zookeeper
clientPort=2181
server.1=zook1:2888:3888
server.2=zook2:2888:3888
server.3=zook3:2888:3888
```

ensemble 中，每个节点都必须有一个 1 到 255 之间的唯一 ID。节点通过读取存储在 `dataDir` 指令所定义目录下的“myid”文件来获知自己的 ID。该文件只有一行，仅包含该机器的 ID 文本。例如，名为 zook2 的节点，配置文件中 ID 定义为 2，那么该节点上可以使用命令 `echo 2 > /var/db/zookeeper/myid` 创建 myid 文件。ensemble 中的每个节点都执行此操作后，你只需像独立模式那样启动 Zookeeper 即可。ensemble 中的每个节点会相互联络以进行选举，选出领导者，其它所有节点则成为 follower。

一大优势在于，从单机的独立模式扩展到环境中跨多台机器的高度容错仲裁模式，所需的工作量并不大。更妙的是，从开发者角度来看，所有接口都保持不变，因此应用无需额外改动。Zookeeper 最吸引人的特点是，系统设计上在初始设置后几乎不需要维护，所有复杂性对终端用户都是隐藏的，因此集成起来非常容易。

## Zookeeper 的扩展

通常在分布式系统中，你可以通过增加容量并将应用分摊到更多节点上来扩展以应对负载增长。然而，由于 ensemble 中的每个成员需要对每笔交易达成一致，随着投票成员增多，ensemble 的写性能会开始下降。除了 leader 和 follower 角色之外，Zookeeper 还允许成员以 observer 身份加入 ensemble。担任此角色时，observer 不参与原子广播协议的协商步骤，只接受已由仲裁中其它 follower 达成一致的事务。observer 的主要目标是在不牺牲写性能的前提下提供读可扩展性。此外，由于 observer 不投票也不参与领导者选举，可以为它们加载更多只读客户端，从而减轻必须参与协商的 follower 节点的负载。

将一个节点配置为 observer 的过程相当简单。需要在节点的配置文件中添加一行 `peerType=observer`，告知其担任 observer 角色。此外，其它所有节点也需要通过在对应的 server 配置行末尾追加 `:observer` 来告知哪些服务器担任 observer。

```sh
server.1=zook1:2888:3888
server.2=zook2:2888:3888
server.3=zook3:2888:3888
server.4=zook4:2888:3888:observer
```

完成这些配置变更后，按正常方式重启集群即可，客户端可以像连接 follower 节点一样连接 observer 节点。

## 结语

如果做法不当，分布式系统很快会变成一团难以驾驭、难以使用甚至更难调试的乱麻。随着数据集越来越大、工作负载越来越重，我们正在快速接近这样一个时间点：要完成更多工作，唯一的选择就是将处理任务分散到网络中尽可能多的机器上。正因为这种技术趋势的转变，Zookeeper 迅速成为如此受欢迎的开源项目并获得广泛采用也就不足为奇了。如果你需要为应用引入复杂功能，Zookeeper 很快就会成为你软件栈中不可或缺的一块基础设施。

---

**Steven Kreuzer** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车情有独钟。他与妻子、女儿和一只狗住在纽约皇后区。
