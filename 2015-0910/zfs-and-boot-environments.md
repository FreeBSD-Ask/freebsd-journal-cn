# ZFS 与启动环境

- 原文：[ZFS and Boot Environments](https://freebsdfoundation.org/our-work/journal/browser-based-edition/cloudabi/)
- 作者：**Michael W. Lucas**

升级主机的操作系统或软件包伴随着各种风险。新内核或许会在 NFS 客户端中暴露一个隐蔽 bug，新版 Web 服务器可能无法处理你的 PHP 代码。修复这些问题需要大量经验、周密计划、调试技巧，外加一些运气。升级失败会引出那个最棘手的问题：修复问题，还是回滚？两者都意味着一大堆工作，而且多半发生在你还有别的事要做的时候。

自 FreeBSD 10.1 起，新的 ZFS 安装专门为支持启动环境而设计。启动环境早已是 Solaris 系统的特性，可让你安装多个版本的核心操作系统并选择启动哪一个。若每次改动系统都创建一个启动环境，就拥有了一条整體回滚的捷径。但启动环境也改变了系统管理的其他方面，若盲目部署反而会招致更多麻烦。

## 数据集布局

启动环境构建于 ZFS 之上。所有应纳入启动环境的内容必须位于单一数据集上。看一眼全新的 FreeBSD 10.1/amd64 安装便知端倪。

```sh
# zfs list
NAME                 USED  AVAIL  REFER  MOUNTPOINT
zroot                465M  188G   128K   none
zroot/ROOT           463M  188G   128K   none
zroot/ROOT/default   462M  188G   462M  /
zroot/tmp            149K  188G   149K  /tmp
zroot/usr            570K  188G   128K  /usr
zroot/usr/home       186K  188G   186K  /usr/home
zroot/usr/ports      128K  188G   128K  /usr/ports
zroot/usr/src        128K  188G   128K  /usr/src
zroot/var            703K  188G   128K  /var
zroot/var/crash      128K  188G   128K  /var/crash
zroot/var/log        192K  188G   192K  /var/log
zroot/var/mail       128K  188G   128K  /var/mail
zroot/var/tmp        128K  188G   128K  /var/tmp
```

此 ZFS 安装为 **/usr**、**/var**、**/var/log** 与 **/usr/ports** 等子目录创建了独立数据集。但这份列表具有欺骗性。将数据集列表与 `mount(8)` 的输出对比一下：

```sh
# mount
zroot/ROOT/default on / (zfs, local, noatime, nfsv4acls)
devfs on /dev (devfs, local, multilabel)
zroot/tmp on /tmp (zfs, local, noatime, nosuid, nfsv4acls)
zroot/usr/home on /usr/home (zfs, local, noatime, nfsv4acls)
zroot/usr/ports on /usr/ports (zfs, local, noatime, nosuid, nfsv4acls)
zroot/usr/src on /usr/src (zfs, local, noatime, nfsv4acls)
zroot/var/crash on /var/crash (zfs, local, noatime, noexec, nosuid, nfsv4acls)
zroot/var/log on /var/log (zfs, local, noatime, noexec, nosuid, nfsv4acls)
zroot/var/mail on /var/mail (zfs, local, nfsv4acls)
zroot/var/tmp on /var/tmp (zfs, local, noatime, nosuid, nfsv4acls)
zroot/var/www on /var/www (zfs, local, noatime, nfsv4acls)
```

并非所有已有数据集都挂载了！快速检查显示 **/var** 与 **/usr** 未挂载，尽管 **/usr/ports** 与 **/var/log** 等子数据集已挂载。这是怎么回事？

**/var** 与 **/usr** 数据集仅作为 **/usr/home** 与 **/var/crash** 等子数据集的父数据集存在。**/usr/home/** 下的文件位于其独立数据集上，而 **/usr/bin**、**/usr/sbin**、**/var/db** 等目录下的文件实际上位于根数据集上。这对习惯传统文件系统的人而言极不直观。ZFS 不仅允许这种情况，还将其作为特性使用。

无论是升级基本系统还是软件包，FreeBSD 上有几个目录至关重要。FreeBSD 的核心程序二进制位于 **/bin**、**/sbin**、**/usr/bin** 与 **/usr/sbin**。得益于 **/usr** 数据集不挂载，这些目录现在位于根数据集上。软件包位于 **/usr/local** 之下——但那里也在根数据集上。**/var/db** 目录包含软件包数据库、freebsd-update 记录乃至 locate 数据库。因为 **/var** 未挂载，这些关键的 **/var/db** 文件现在也是根数据集的一部分。

不属于核心系统的文件——如日志、用户主目录等——拥有各自的数据集。你可将操作系统与附加软件包作为单一实体管理。

## 启动环境与 ZFS

ZFS 的一大特性是强大的快照功能。你不仅可以对数据集做快照，还能将文件系统回滚到该状态。可将快照克隆为一份全新的活动文件系统副本。

这在系统管理上有明显的应用。升级前为系统做快照。若升级失败，回滚到已知可用的版本。要调试一次失败的升级，可把失败升级的快照复制到测试系统上调试，同时让生产系统继续在稍旧的操作系统版本上稳定运行。

“启动环境”是一种将这一切打包得整整齐齐、系上丝带的方法。你并不一定需要启动环境管理器才能使用启动环境，但它让操作更为便捷。Solaris 系统使用 `beadm(8)`（boot environment administration）来管理启动环境。FreeBSD 有一个附加的 beadm 软件包，刻意设计得与 Solaris 程序相似——我们完全可以另造一个 FreeBSD 专属工具，但实在没必要。

## 安装 beadm

用 `pkg(8)` 获取 beadm。下面在一台全新的 10.1 机器上安装 beadm。

```sh
# pkg install -y beadm
```

现在查看你拥有的启动环境。

```sh
# beadm list
BE       Active Mountpoint Space   Created
default NR     /          494.0M  2015-04-08 07:18
```

唯一的启动环境名为 default。Active 列中，N 表示该环境当前处于活动状态。R 表示该环境在重启后仍为活动。因此 default 环境当前活动，且下次重启后仍活动。

这完全合理。这台机器只有一个启动环境——我才刚装好它。我需要将这台主机升级到 FreeBSD 10.1 的最新版本 p9。这次升级可能出问题——大概率不会，但有可能。我想为它创建一个新的启动环境，取名为 install，因为这是我刚安装完。

```sh
# beadm create install
Created successfully
# beadm list
BE       Active Mountpoint Space   Created
default NR     /          646.0M  2015-04-08 07:18
install  -      -          10.7K   2015-04-08 11:43
```

“install”启动环境是现有根数据集的只读快照。若有问题，我可重新激活该版本。“default”启动环境是当前活动的根数据集。在执行任何升级或重大系统改动（如更新已安装软件包）之前，我都会创建一个反映当前系统状态的启动环境。

每一个都代表一次可能出错的升级。

```sh
# beadm list
BE        Active Mountpoint  Space   Created
default   -      -           3.2G    2015-04-28 11:53
install   -      -           126.0M  2015-04-28 12:19
10.1-p9   -      -           209.0M  2015-05-14 08:01
10.1-p10  N      /           166.0M  2015-05-24 11:02
```

## 回滚

若升级出问题，用 `beadm activate` 与启动环境名称激活上一个已知良好的启动环境。

```sh
# beadm activate 10.1-p10
Activated successfully
# beadm list
BE        Active Mountpoint  Space   Created
default   N      /           3.2G    2015-04-28 11:53
install   -      -           126.0M  2015-04-28 12:19
10.1-p9   -      -           209.0M  2015-05-14 08:01
10.1-p10  R      -           166.0M  2015-05-24 11:02
```

“default”环境的 Active 列为 N，表示它当前正在运行。“10.1-p10”环境为 R，表示重启后将成为活动环境。

从失败的升级中回滚如今变得轻而易举。

## 背后的原理

```sh
# zfs list -r zroot/ROOT
NAME                  USED  AVAIL  REFER  MOUNTPOINT
zroot/ROOT            3.21G 26.9G  96K    none
zroot/ROOT/10.1-p10   12K   26.9G  2.19G  /
zroot/ROOT/10.1-p9    12K   26.9G  2.12G  /
zroot/ROOT/default    3.21G 26.9G  2.51G  /
zroot/ROOT/install    8K    26.9G  612M   /
```

启动环境在 ZFS 层面做了什么？看一下根之下的数据集。我们如今在 zroot/ROOT 下有若干数据集。若查看它们的 origin（用 `zfs get -r origin zroot/ROOT`），会发现每一个都是 default 数据集在特定日期与时间的克隆。

若想查看旧启动环境中的内容，查看源快照。若想修改一个未使用的启动环境，将其挂载到临时位置。

## 启动环境与 FreeBSD 常见实践

启动环境能省去大量痛苦，但也可能给老 FreeBSD 人带来意外。你需要做的最大思维转变是记住 **/var/db** 与 **/usr/local** 等目录位于启动环境之下。若回滚到较旧的启动环境，这些目录也会随之回退。

FreeBSD 的 MySQL 软件包将数据库文件保存在 **/var/db/mysql**。若回滚到较旧的启动环境，将遭遇数据库回退。PostgreSQL 软件包将文件存放在 **/usr/local/pgsql**。切换启动环境会带走数据。许多 Web 服务器程序将文件塞在 **/usr/local/something**。这样的例子还有很多。

数据库与应用确实需要为它们的数据建立独立的 ZFS 数据集。我创建了 zroot/var/mysql、zroot/var/www 等，并告知应用程序使用这些数据集。这些通过 rc.conf 很容易配置。下面这行 rc.conf 配置让 mysql 软件包将数据放在 **/var/mysql**。

```sh
mysql_dbdir="/var/mysql"
```

对一个能为你省下大量痛苦的强大工具而言，这只是个很小的改动。

**作者简介**

Michael W. Lucas 是《Absolute FreeBSD》、《Absolute OpenBSD》与《DNSSEC Mastery》等书的作者。他与妻子及一大群老鼠居住在密歇根州底特律。个人网站：<https://www.michaelwlucas.com/>。
