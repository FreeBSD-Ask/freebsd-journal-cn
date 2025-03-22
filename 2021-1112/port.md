# 通过 iSCSI 导入 ZFS ZIL——不要在工作中这样做——就像我做的那样

- 原文链接：[Importing a ZFS ZIL via iSCSI  Don’t do this at work — like I did](https://freebsdfoundation.org/wp-content/uploads/2022/01/Importing-a-ZFS-ZIL-via-iSCSI.pdf)
- 作者：**BENEDICT REUSCHLING**

---

这篇专栏介绍了 FreeBSD 的 Ports 和软件包，这些 Port 和软件包在某些方面有用、独特，或者其他值得了解的特点。Ports 扩展了操作系统的基本功能，确保你能够完成任务，或者简单地带来一些乐趣。跟随我们一起探索，也许你会发现一些新东西。

---


我已经在 FreeBSD 虚拟机上运行了我们计算机科学系的 PostgreSQL 服务器多年，且取得了很好的效果。该服务器主要用于数据库课程和需要数据库后端的项目。我在 2019 年的 vBSDcon 上就该服务器做过一次演讲，你可以在 YouTube 上找到视频。

最近，托管这台虚拟化服务器的部门将其底层存储更改为 Ceph。通过在校园内三个不同建筑之间同步 I/O，这为他们增加了更多的容量和冗余。与此同时，数据库课程的教授设计了新的实验，帮助学生熟悉大量数据集。一个实验要求创建大量数据并将其插入到数据库表中，测量有无表索引的执行时间。一切看起来很好，但在该实验开始不久后，教授和学生开始抱怨性能不佳。在某些情况下，学生笔记本上的本地 PostgreSQL 安装运行速度比我们服务器上的速度还快，尽管我们的服务器有更多的 CPU 和内存。例如，执行一个大约 1000 万行的“SELECT COUNT(\*) from bigtable;”查询，平均耗时 2 分 5 秒。而一台本地笔记本仅需约 1 秒。第二次执行相同的查询时，服务器耗时仅 1 秒，证明查询是从更快的主内存缓存中提供的。

我开始调查 PostgreSQL，调整了一些 postgresql.conf 配置文件中的参数并重启了服务器。这只是取得了有限的成功，大家仍然抱怨插入和查询时间过长。由于有证据表明 PostgreSQL 的默认设置性能更好，问题必定与存储或 I/O 相关。当虚拟机创建时，其底层的 Ceph 存储被转化为 ZFS 池，并将其中的大部分作为 PostgreSQL 数据库的数据集提供。由于很多学生插入了相同的数据并进行了查询，ZFS ARC 将这些数据直接从内存中提供。并不是所有的数据都能适应 ARC，或者某些数据被其他查询驱逐出 ARC。待我们开始对磁盘进行写操作，随着用户生成的大量数据，性能下降变得显而易见。

为了确认我们对底层存储存在问题的怀疑，我从我们的大数据集群中选择了一台有 64 个 CPU、384 GB 内存和 4 个 512 GB NVMe 的服务器，并在其上安装了 FreeBSD。然后，我使用“zfs send”将托管 PostgreSQL 服务器的数据集复制到这台新服务器上。启动 postgres 服务后，我得到了在更强硬件上可以操作的完整服务器副本。在新服务器上运行相同的 COUNT(\*) 查询，证明它们和学生的笔记本电脑一样快（如果不是更快的话），即使学生的笔记本也有 SSD。显然，虚拟服务器上的性能是问题所在。然而，解决这个问题并不容易，因为我们的 IT 部门不能简单地为这个虚拟机附加一个 SSD 或 NVMe 来加速它。购买并将其安装到服务器上（这意味着停机）将比剩余的学期时间还要长。

我的想法是通过 iSCSI 将我们刚刚测试的服务器上的一个 NVMe 硬盘导出到虚拟机上，创建一个表空间。表空间允许数据库管理员定义数据库对象在文件系统中应该存储的位置。通过 iSCSI，存储（称为目标）可以通过网络发送到另一台计算机（称为发起者），并在该计算机上导入它。与网络共享不同，iSCSI 协议使存储在导入的计算机上显示为本地块存储——这是一个重要的区别。这个新存储就像任何其他存储一样被处理，可以像本地附加的设备一样进行分区和格式化，使用新的文件系统。

FreeBSD 默认内置 iSCSI，只需要在配置文件中进行一些更改即可设置。以下是导出 NVMe 的服务器上的配置：

首先，我在其中一个 NVMe 驱动器上创建了一个 200 GB 的卷，命名为 iscsi_export：

```sh
# zfs create -V 200g nvme/iscsi_export
```

接下来，我编辑了 /etc/ctl.conf 文件，使其包含以下内容：

```ini
portal-group pg0 {
    discovery-auth-group no-authentication
    listen ip.address.of.initiator
}

target iqn.dns-name-of-initiator:nvme {
    portal-group pg0
    chap postgres verysecurepasswordgoeshere
    lun 0 {
        path /dev/nvme/iscsi_export
        size 200G
    }
}
```


我将这个文件的所有权和权限更改为 root，因为它包含了明文密码。

在服务器重启后，iSCSI 发起器应该再次启动，因此我在 /etc/rc.conf 中加入了 `ctld_enable="YES"`：

```sh
# sysrc ctld_enable=yes
```

为了启动发起器，我启动了服务：

```sh
# service ctld start
```

这大致遵循了 FreeBSD 手册中 iSCSI 部分的描述。在导入存储磁盘的虚拟机上，我在 /etc/iscsi.conf 中加入了以下内容：

```ini
TargetAddress = ip.address.of.initiator
TargetName = iqn.dns-name-of-initiator:nvme
AuthMethod = CHAP
chapIName = postgres
chapSecret = verysecurepasswordgoeshere
}
```

由于 postgres 用户通过 SSH 登录到该服务器以运行 PostgreSQL 的命令行工具 psql，确保该文件中的密码不被窥探非常重要。通过将文件权限设置为 `chmod 0700`，并将文件所有者和组设置为 root 和 wheel，可以解决这个问题。为了在重启时启动存储导入，必须在 /etc/rc.conf 中加入一个条目（稍后会详细说明）：

```
# sysctl iscsid_enable=yes
```

接下来，我们可以通过启动服务导入磁盘：

```sh
# service iscsid start
```

成功导入后，一个新设备（可能是 da0 或类似的）出现在 /dev 中。在该设备上创建了一个独立的 ZFS 池：

```sh
# zpool create nvme_ts /dev/da0
```

是的，这并不是冗余的，但对于我们的基准测试目的，它已经足够了。在 postgres 端，以数据库超级用户身份登录 psql，表空间通过以下语句定义（有关详细信息，请参见 https://www.postgresql.org/docs/current/manage-ag-tablespaces.html）：

```sql
psql#>CREATE TABLESPACE nvme LOCATION /nvme;
```

再次检查访问权限，但命令完成后，postgres 数据库用户可以使用该表空间并在其上放置数据库对象（如表）。可以通过显式定义数据应存储的位置：

```sql
psql#>CREATE TABLE nvme_powered_table(i int) TABLESPACE nvme_ts;
```

或将表空间设置为默认表空间：

```sql
psql#>SET default_tablespace = nvme_ts;
```

通过这种新配置（首先清除缓存），并将一批新的 10 GB 数据重新加载到 nvme_powered_table 中，虚拟机上的数据库插入性能提高到了 7 秒（原来超过 2 分钟）。拥有 NVMe 表空间当然很棒，但我们进一步推进了这个过程。正是在这时，麻烦开始了……

## 没有考虑周全

我们决定将导出的存储用作 ZIL（ZFS Intent Log），以加速 Ceph 支持的池上的较慢写入。ZIL 会确认写入已达到稳定存储，并且在数据库继续工作时，稍后会将数据写入较慢的磁盘。ZIL 通常不需要很大，因为其中的数据会很快被驱逐。我们减少了 iSCSI 发起器中导出的磁盘空间，并重新导入了数据库虚拟机中的磁盘。然后，我们使用以下命令将 iSCSI 磁盘配置为 ZIL：

```sh
# zpool add pgpool log da0
```

设备立即显示并正常工作。池上的 I/O 现在迅速被确认“已写入”，数据库可以继续进行，无需等待。ZIL 将写入请求缓慢地传递到较慢的 Ceph 存储。这大大提升了数据库性能，我们进入了生产环境。

## 别在工作中尝试这个

当时我没有意识到的是，这种方式在启动过程中的集成问题。当 FreeBSD 使用仅包含 ZFS 文件系统启动时，它会尝试检测池中包含的所有存储设备。在此启动过程中，网络尚未完全配置，因此无法使用 iSCSI 服务来导入外部设备。对于 ZIL 设备来说，事实证明 ZFS 需要它来正常启动，并且会抱怨池中缺少磁盘。即使池的主 vdev 可用（但 ZIL 不可用），启动过程也会在此早期阶段被停止。你可以想象，这在生产服务器上不会顺利进行，只有服务器本身的管理控制台显示出了问题的根源。

请注意，这种情况可能通过两种方式发生：要么 iSCSI 目标（导出存储的服务器）出现故障或失去连接，要么发起者（导入设备的客户端）出现故障。经验丰富的系统管理员知道，在典型的工作日中，这种中断是常见的，往往是没有预警和意外发生的。只要时间问题，这种情况迟早会发生，而现在它已经发生，我们需要快速修复它。

接下来，我们用 FreeBSD ISO 镜像重新启动服务器，并在安装程序中选择 Live-CD 选项。从 Live-CD 的 shell 环境中，我们可以像这样将缺失的 ZIL 设备挂载到 /mnt：

```sh
# zpool import -R /mnt -m pgpool
```

导入完成后，我们可以检查池中剩余的设备：

```sh
# zpool status
```

输出显示了缺失的缓存设备及其长的唯一数字标识符。接下来的操作是将 ZIL 设备从池中移除：

```sh
# zpool remove pgpool <verylongnumericidentifier>
```

输入长标识符而不是较短的设备名称，提醒我们以后避免这种情况。待完成，并且 zpool status 的输出确认移除，池被再次导出。通常在重新启动时会执行此操作，但我们不想冒险。

```sh
# zpool export pgpool
```

在机器重新启动后，我们高兴地看到它这次顺利完成了启动，并恢复了我们熟悉的登录提示。灾难避免了，但潜在的性能问题仍然存在。

## 幸福的结局

显然，iSCSI 导出太过危险，可能会再次失败。尽管我们在整个学期里都是这样运行的，但根据墨菲定律，这种情况总会在最糟糕的时刻发生，通常是在系统管理员应该休息的时候。肯定可以通过脚本在每次关机时安全地移除 ZIL，但如果涉及 iSCSI 导出的两台机器发生电力故障或崩溃，这种方法无法涵盖。幸运的是，我们的 IT 部门终于为这台机器提供了一个 SSD 支持的 Ceph 存储作为替代方案。该导入与 iSCSI 相似，但更加稳定，且不容易崩溃。

Ceph 在 FreeBSD 上运行良好，但导入这个设备的过程却相当...有趣。Ceph 在 FreeBSD 上只通过 geom_gate 支持这种导入方式，这与 iSCSI 相似。安装 net/ceph14 包后，可以使用 rbd-ggate 命令（rbd 是 Ceph 的 Rados Block Device）。rbd-ggate(8) 的手册页相当简短，仅列出了几个命令和选项。我一开始有点担心，因为它的日期回溯到 2014 年，没有最近的更新，可能由于 FreeBSD 新版本的变化，支持已经失效了。不过，事实证明这个担心是多余的。我们只需要处理 Linux 和 FreeBSD 在命令行参数处理上的一些差异。在 Linux 中使用 --option，而在 FreeBSD 中，更常见的是使用单个 -option。最初的命令看起来像这样：

```sh
# rbd map -t ggate volumes/ssdvolume
```

volumes/ssdvolume 是我们从 IT 部门获得的 SSD Ceph 存储的路径，成功导入后会映射一个 geom gate 设备。由于没有提供进行导入的用户的 --id（用户名和密码保护了这个存储，防止他人未经授权导入），命令失败了。在这里，单破折号和双破折号的混用成了问题，因为基于 Linux 的 rbd 命令拒绝将 --id 与单个 -t 参数混合使用。我们通过将 ID 作为环境变量来解决这个问题，如下所示：

```sh
CEPH_ARGS='--id postgresdb' rbd map -t ggate volumes/postgresdb
```

通过这个组合，命令成功运行，并告诉我们：

```sh
ggate0 created
```

通过查看 /dev/ggate0 确认了这一点。这就是来自 Ceph 的导入设备，我们现在可以在其上创建一个新的 ZFS 池：

```sh
# zpool create ssdpool /dev/ggate0
```

记住上次学到的内容，我们尝试重启机器，看看它在启动时如何处理这个设备。我们很高兴地看到系统没有问题地重新启动，并且我们随后可以使用以下命令重新导入这个新的池：

```sh
# zpool import ssdpool
```

然后，我们可以创建一个小的启动脚本，在系统完成启动后自动重新导入这个池并激活其中的 postgres 数据库。postgres 数据库是通过快照和 zfs 发送从旧的较慢池克隆并接收到了更快的 ssdpool 上。这个方法效果很好，性能差异是显而易见的。当我写这篇文章时，第一批学生组已经在使用它了（他们并不知道），而我还没有收到任何投诉。

## 经验教训

衡量性能丧失的地方并隔离瓶颈。使用不同的测试用例来确认可能存在问题的位置。在将系统投入生产之前进行测试。在处理通过网络传输的存储时，确保解决方案能够经得起导出和导入机器重启的考验。保持 FreeBSD Live-CD ISO 镜像，以便在灾难发生时进行修复。记录每一步和每个命令，以便在生产环境中，用户急于恢复功能时，能够快速找到解决方案。做好实验准备，尝试新事物。最后，依赖 FreeBSD 作为存储领域的坚实基础，利用其灵活性和提供的多种解决方案。

---

**BENEDICT REUSCHLING** 是 FreeBSD 项目的文档贡献者，并且是文档工程团队的成员。他在 FreeBSD 基金会董事会担任副主席。过去，他曾在 FreeBSD 核心团队工作两届。他在德国达姆施塔特应用科技大学管理一个大数据集群，并教授“开发者的 Unix”课程给本科生。Benedict 也是每周 bsdnow.tv 播客的主持人之一。
