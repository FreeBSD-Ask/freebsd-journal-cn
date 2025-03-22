# ZFS 简介

- 原文链接：[An Introduction to ZFS](https://freebsdfoundation.org/wp-content/uploads/2023/02/gurkowski_zfs_guide.pdf)
- 作者：**DREW GURKOWSKI**

ZFS 将卷管理器和独立文件系统的功能结合为一体，相较于独立的文件系统，它具有多个优势。它以速度、灵活性而著称，最重要的是，它非常注重防止数据丢失。虽然许多传统的文件系统必须在单块磁盘上存在，但 ZFS 知道磁盘的底层结构，然后创建存储池，即使是在多个磁盘上。当额外的磁盘被添加到存储池时，现有的文件系统会自动扩展，立即变得可供文件系统使用。

## 入门

FreeBSD 可以在系统初始化时挂载 ZFS 池和数据集。要启用此功能，请在 `/etc/rc.conf` 中添加以下行：

```sh
zfs_enable=”YES”
```

然后开启服务：

```sh
# service zfs start
```

## 识别硬件

在设置 ZFS 之前，识别与系统关联的磁盘设备名称。一种比较快的方法是使用以下命令：

```sh
# egrep "da[0-9]|cd[0-9]" /var/run/dmesg.boot
```

输出应该会识别设备名称，本文指南中的示例将使用默认的 SCSI 名称：`da0`、`da1` 和 `da2`。如果硬件不同，请确保使用正确的设备名称。

## 创建单磁盘池

要使用单个磁盘设备创建一个简单的、无冗余的池：

```sh
# zpool create example /dev/da0
```

要在池中创建供用户浏览的文件：

```sh
# cd /example
# ls
# touch testfile
```
可以用命令 `ls` 查看新文件：

```sh
# ls -al
```

我们现在可以开始使用更高级的 ZFS 特性和属性。要在池中创建启用压缩的数据集：

```sh
# zfs create example/compressed
# zfs set compression=on example/compressed
```

现在，数据集 `example/compressed` 是一个 ZFS 压缩文件系统。要禁用压缩，可以使用：

```sh
# zfs set compression=off example/compressed
```

要卸载文件系统，使用 `zfs umount`，然后使用 `df` 验证：

```sh
# zfs umount example/compressed
# df
```

验证 `example/compressed` 是否不在输出的挂载文件列表中。以通过 `zfs mount` 重新挂载文件系统可：

```sh
# zfs mount example/compressed
# df
```

挂载文件系统后，输出应包含类似如下行：

```sh
example/compressed 17547008 0 17547008 0% /example/compressed
```

ZFS 数据集的创建方式与其他文件系统一样，以下示例创建了一个名为 `data` 的新文件系统：

```sh
# zfs create example/data
```

使用 `df` 查看数据和空间使用情况（部分输出已删除以便清晰显示）。

```sh
# df
 . . .
example/compressed 17547008 0 17547008 0% /example/compressed
example/data 17547008 0 17547008 0% /example/data
```

要销毁不再需要的文件系统和池：

```
# zfs destroy example/compressed
# zfs destroy example/data
# zpool destroy example
```

## RAID-Z

RAID-Z 池需要三块及更多磁盘，但在磁盘发生故障时可以提供数据丢失保护。由于 ZFS 池可以使用多个磁盘，因此文件系统设计本身就支持 RAID。

要创建 RAID-Z 池，指定要添加到池中的磁盘：

```
# zpool create storage raidz da0 da1 da2
```

创建完 zpool 后，可以在该池中创建一个新的文件系统：

```
# zfs create storage/home
```

启用压缩并存储目录和文件的额外副本：

```
# zfs set copies=2 storage/home
# zfs set compression=gzip storage/home
```

RAID-Z 池是存储关键系统文件的绝佳位置，例如用户的主目录。

```sh
# cp -rp /home/* /storage/home
# rm -rf /home /usr/home
# ln -s /storage/home /home
# ln -s /storage/home /usr/home
```

可以创建文件系统快照，以便以后回滚，快照名称用黄色标记，可以是任意你想要的名称：

```sh
# zfs snapshot storage/home@11-01-22
```

ZFS 创建数据集的快照，用户可备份文件系统以便未来回滚或数据恢复。

```sh
# zfs rollback storage/home@11-01-22
```

使用 `zfs list` 可以列出所有可用的快照：

```sh
# zfs list -t snapshot storage/home
```

## 恢复 RAID-Z

每个软件 RAID 都有一种方法来监控其状态。使用以下命令查看 RAID-Z 设备的状态：

```sh
# zpool status -x
```

如果所有池都在线且一切正常，信息将显示：“all pools are healthy”（所有池都健康）。

如果出现问题，例如磁盘处于离线状态，池的状态将显示如下：

```sh
pool: storage
 state: DEGRADED
status: One or more devices has been taken offline by the administrator.
Sufficient replicas exist for the pool to continue functioning in a
degraded state.
action: Online the device using ‘zpool online’ or replace the device with
zpool replace.
 scrub: none requested
config:
NAME STATE READ WRITE CKSUM
storage DEGRADED 0 0 0
 raidz1 DEGRADED 0 0 0
 da0 ONLINE 0 0 0
 da1 OFFLINE 0 0 0
 da2 ONLINE 0 0 0
errors: No known data errors
```

“OFFLINE”表示管理员使用以下命令将 `da1` 离线：  

```sh
# zpool offline storage da1
```

现在关闭计算机并更换` da1`。启动计算机后，将 `da1` 重新加入池中：  

```sh
# zpool replace storage da1
```

接下来，再次检查状态，这次不使用参数 `-x`，显示所有池：

```sh
# zpool status storage
 pool: storage
 state: ONLINE
 scrub: resilver completed with 0 errors on Fri Nov 4 11:12:03 2022
config:
NAME STATE READ WRITE CKSUM
storage ONLINE 0 0 0
 raidz1 ONLINE 0 0 0
 da0 ONLINE 0 0 0
 da1 ONLINE 0 0 0
 da2 ONLINE 0 0 0
errors: No known data errors
```



## 数据验证  

ZFS 使用校验和来验证存储数据的完整性，这些数据校验和可以被验证（称为清理）以确保存储池的完整性：  

```sh
# zpool scrub storage
```

由于清理过程需要大量的输入/输出，因此一次只能运行一个清理。清理的持续时间取决于池中存储的数据量。清理完成后，使用 `zpool status` 查看状态：

```sh
# zpool status storage
 pool: storage
 state: ONLINE
 scrub: scrub completed with 0 errors on Fri Nov 4 11:19:52 2022
config:
NAME STATE READ WRITE CKSUM
storage ONLINE 0 0 0
 raidz1 ONLINE 0 0 0
 da0 ONLINE 0 0 0
 da1 ONLINE 0 0 0
 da2 ONLINE 0 0 0
errors: No known data errors
```


显示上次清理完成日期有助于决定何时开始下一次清理。定期清理有助于保护数据免受静默损坏，并确保存储池的完整性。  

## ZFS 管理  

ZFS 有两个主要的管理工具：  
- `zpool` 工具控制池的操作，允许添加、移除、替换和管理磁盘。  
- `zfs` 工具用于创建、销毁和管理数据集，包括文件系统和卷。  

虽然本入门指南未涉及 ZFS 管理，但你可以参考 `zfs(8)` 和 `zpool(8)` 以获取更多 ZFS 参数。  

## 更多资源  

虽然通过本指南创建的非冗余和 RAID-Z 池可以在大多数用例中工作，但更复杂或专业的系统可能需要进一步的 ZFS 管理和设置。本指南仅触及了 ZFS 功能的表面，因为它是一个功能强大且可定制的文件系统。OpenZFS Wiki 提供了有关安装、ZFS 系统管理和手册页的广泛文档。如果由于系统架构需要进行调优，可以在 OpenZFS 和 FreeBSD Wiki 页面上找到 ZFS 调优指南。

---

DREW GURKOWSKI，FreeBSD 基金会
