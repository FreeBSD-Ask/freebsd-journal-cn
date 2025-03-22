# 基于 Samba 的时间机器备份

- 原文链接：[Samba-based Time Machine Backups](https://freebsdfoundation.org/our-work/journal/browser-based-edition/storage-and-filesystems/samba-based-time-machine-backups/)
- 作者：Benedict Reuschling


“我真希望我能把带宽节省下来做备份之类的有用事情，因为我永远不会需要它们”——从未有人这么说过。在发生灾难时——而灾难总是不期而至——备份是 IT 弹性的重要组成部分。硬盘故障、笔记本电脑被盗、固件驱动程序损坏导致数据不可读等等，都是备份所要应对的情况。定期备份并确保其可用性对于持续的业务运营至关重要，而测试备份的可恢复性同样重要。但如果备份解决方案不再受支持，并且无法在较新的系统上工作该怎么办？新的系统如何备份，又如何与现有的解决方案集成？

我在《FreeBSD 期刊》2022 年 3/4 月号中写了一篇关于如何设置 FreeBSD 苹果时间机器的文章。这个设置我已经运行了很久，没有遇到问题——无论是在备份方面，还是在我需要恢复数据时。随着时间的推移，苹果改变了底层的 Apple 文件协议（AFP）。在更新的 macOS 版本中（我一直定期升级以获得安全和功能增强），该协议经历了一些变化，使得我原来的设置不再适用了。如上所述，当前的设置仍然有效，但对于新的时间机器系统，我再也无法使用它了。从 macOS 10.9（Mavericks）开始，Apple 将 SMB（Samba 更广为人知）集成到协议中。此次迁移在 macOS 11（Big Sur）完成，该版本移除了 AFP 服务器部分，改为将 SMB 作为新的标准，时间机器也开始内部使用 SMB。

当我设置新的时间机器备份系统时，我发现了这一点。许多年前，我将我在 2022 年的文章中的设置写成了一个 Ansible playbook 以便更轻松地部署。这个 playbook 仍然有效，但它导致新版本的 macOS 无法将导出的驱动器挂载为时间机器的存储位置。幸运的是，我发现其他人碰到了同样的问题，并且已经找到了相应的解决方案。哪怕这些说明并不像我希望的那样简洁，但它们已经解决了问题。以下的设置结合了不同的来源——Reddit、个人博客、FreeBSD 论坛以及 Samba 文档。我已经在两台不同的机器上进行了测试，补充了一些说明，并添加了缺失的命令以确保它按预期工作。我保留了早期的“基于 ZFS”，因为那是我用来存储重要数据的系统。你可以选择不使用 ZFS，去用别的文件系统，但如果没成功，也别怪我！

## 系统要求

为了设置此备份系统，你需要一台机器（在我的例子中是 FreeBSD）来存储备份。它应该有良好的网络连接和快速的存储。存储还需要以某种方式实现冗余，比如 RAID1 及更高等级的 RAID。存储容量取决于两个因素：备份数据的人数和他们的数据量。时间机器配置对话框允许你设置磁盘配额和加密，这两者都是不错的选择。ZFS 支持配额和保留空间，因此我在文件系统层面上设置了相同的值。时间机器会自动删除较旧的备份，当可用存储空间不足以容纳新的备份数据时。存储越多，你可以恢复的备份历史也就越长。

加密功能也很有用，特别是当我们将数据通过可能未加密或者经 VPN 网络传输时。在接收系统上，可以为静态数据添加加密的 ZFS 数据集。但是，值得注意的是，当备份目标需要重新启动时，除非有人输入用于挂载数据集的密码，否则加密的数据集将不会被挂载。你可以配置 ZFS 从某个文件获取密码，但我将其作为一个安全访问的练习，来保障存储密码的文件不被泄露。

在发送端，所有 macOS 系统都能够挂载、配置该驱动器，并通过个别用户凭证进行保护。这能让多人向同一时间机器备份。考虑到可能同时进行多个备份，因此有足够带宽的服务器变得尤为重要。当在 Mac 上配置多个时间机器备份位置时，系统会智能地避免同时备份到两个位置，从而防止系统 I/O 被长时间的备份任务所阻塞。

## 配置备份服务器

首先安装你选择的 FreeBSD 版本（理想情况下仍在支持范围内），并确保安装了最新的安全补丁。我们在这里不选择使用 jail，但我们用 jail 也没问题。首先安装 Samba 包：

```sh
# pkg install samba419
```

接下来，我们为 ZFS 配置备份存储。在我的例子中，我有一个专门的池，命名为 `backup`，并挂载在目录 `/backup`。我为时间机器创建了一个单独的数据集，并设置了配额和保留空间，因为我还在其中存储其他数据，并且希望为这些文件保留一定的空间。

```sh
# zfs create -o quota=1.5T -o reservation=1.5T backup/timemachine
```

我知道会有两个用户（Tammy 和 Tim；Alice 和 Bob 在度假）将他们的 Mac 备份到该位置。我平等地对待他们，因此我为两者设置了相同的空间保留和配额。请记住，数据集上的配额和保留空间也会应用于其下的所有数据集。1.5 TB 也将应用于他们的数据集，已对其进行了限制。但每人 500GB 是足够的，因此我为每个数据集分别设置了 `refquota` 和 `refreservation`。

```sh
# zfs create -o refquota=500g -o refreservation=500g backup/timemachine/tammy
# zfs create -o refquota=500g -o refreservation=500g backup/timemachine/tim
```

他们两人都永远无法登录到我的备份服务器（他们也不在乎），但他们仍然需要在系统上拥有一个用户来挂载时间机器的存储。我为他们两人都运行了 `adduser`，不给他们家目录（`/var/empty`），也不给他们 shell 访问权限（`/usr/sbin/nologin`）。

运行命令 `chmod` 和 `chown` 来保护他们的数据集挂载点，防止外部窥探。

```sh
chmod -R 0700 /backup/timemachine/tammy
chmod -R 0700 /backup/timemachine/tim
chown -R tim /backup/timemachine/tim
chown -R tammy /backup/timemachine/tammy
```

最后，设置 Samba 端的密码，供这两个用户使用。这就是在 macOS 中挂载时间机器备份时的密码提示。

```sh
# smbpasswd -a tim
# smbpasswd -a tammy
```

### Avahi 配置

到此为止，所需的软件已经安装完毕——简单易行。接下来，我们需要创建两个配置文件，一个是时间机器服务的配置，另一个是 Samba 配置。时间机器服务通过 Avahi 运行——不是来自《马达加斯加》的狐猴，而是这个软件。Avahi 是一款 Zeroconf 网络实现，允许程序在本地网络中发布和发现服务（比如我们的时间机器）。配置文件是基于 XML 格式的，位于 `/usr/local/etc/avahi/services/timemachine.service`（如果该文件不存在，需要创建），文件内容如下：

```xml
<?xml version=”1.0” standalone='no'?>
<!DOCTYPE service-group SYSTEM “avahi-service.dtd”>
<service-group>
<name replace-wildcards=”yes”>%h</name>
<service>
<type>_smb._tcp</type>
<port>445</port>
</service>
<service>
<type>_device-info._tcp</type>
<port>0</port>
<txt-record>model=RackMac</txt-record>
</service>
<service>
<type>_adisk._tcp</type>
<txt-record>sys=waMa=0,adVF=0x100</txt-record>
<txt-record>dk0=adVN=FreeBSD TimeMachine,adVF=0x82</txt-record>
</service>
</service-group>
```

该文件定义了监听 Samba 服务端口（445），挂载驱动器的图标样式（`RackMac`）和显示名称（`FreeBSD TimeMachine`）。你可以更改显示名称，以便为其指定一个更具陈述性的名称。我未更改文件中的其他部分。

### Samba 配置

Samba 是 SMB 协议的开源实现，已经有 32 年历史。最初，它的主要目的是实现 Unix 系统与 Windows 之间的兼容性。随着 Windows 添加了更多附加功能，Samba 也加入了 Active Directory（AD，活动目录）集成、域控制器等功能。由于此设置中不涉及任何 Windows 系统（你能听到松了一口气的声音），我们在这里使用 Samba 的文件共享功能来代替 AFP。

Samba 的配置文件位于 `/usr/local/etc/smb4.conf`，内容如下：

```ini
[global]
workgroup = WORKGROUP
security = user
passdb backend = tdbsam
fruit:aapl = yes
fruit:model = MacSamba
fruit:advertise_fullsync = true
fruit:metadata = stream
fruit:veto_appledouble = no
fruit:nfs_aces = no
fruit:wipe_intentionally_left_blank_rfork = yes
fruit:delete_empty_adfiles = yes

[TimeMachine]
path = /backup/timemachine/%U
valid users = %U
browseable = yes
writeable = yes
vfs objects = catia fruit streams_xattr zfsacl
fruit:time machine = yes
create mask = 0600
directory mask = 0700
```

两个部分（`global` 和 `TimeMachine`）定义了备份目标所需的参数。以 `fruit:` 为前缀的行用于与 macOS 的兼容性。有关这些行的详细文档，请参见 Samba 文档（见文章末尾的参考文献）。在 `[TimeMachine]` 部分中，将路径行更改为之前在 FreeBSD 上创建的路径。`%U` 部分是一个占位符，表示个别用户名（在我们的例子中是 tammy 和 tim）进行文件备份。这样，当以后添加另一个用户时，我们就不需要更改此行了。`create mask` 和 `directory mask` 确保正确的权限，避免文件被混合，且用户无法看到和更改其他用户的备份。

## 启动

剩下的步骤就是启用并启动 dbus（avahi）和 samba 服务。

```sh
# service dbus enable
# service dbus start
# service samba_server enable
# service samba_server start
```

在 macOS 端（备份客户端），打开 Finder 并按下 CMD-K（“连接到服务器”的快捷键）。输入 `smb://server.ip.or.dns`。若一切顺利，输入用户名和密码。这就是我们之前在 `smbpasswd` 对话框中为 tim 和 tammy 设置的密码。如果成功，共享目录将挂载到系统中。接下来，进入时间机器配置对话框，再添加新的 Time Machine 卷。记得点击其中的选项按钮，再勾选加密备份框。此设置只能在初次备份之前设置，之后无法更改。你还可以限制备份占用的磁盘空间，但这不是强制的，因为我们已经在 ZFS 层面上设置了。然后，首次备份将开始进行。当完成后，时间机器将自动挂载和卸载该共享，进行定期的备份。

## 总结

就这样，Samba 配置相当简单，用户应该能够根据自己的需求进行调整。我发现这个新解决方案和旧的解决方案一样可靠。我已经调整了我的 Ansible playbook，以便使用新的基于 Samba 的设置。我依然喜欢那种“一键备份”的方式，知道在需要时，我能恢复单个文件和整个系统，恢复的文件都是我最近使用的最新文件。

## 参考文献和来源

* [Samba 文档](https://www.samba.org/samba/docs/current/man-html/vfs_fruit.8.html)
* [Reddit 帖子](https://www.reddit.com/r/homelab/comments/83vkaz/howto_make_time_machine_backups_on_a_samba/)
* [FreeBSD 论坛帖子](https://forums.freebsd.org/threads/samba-functions-but-unable-to-use-it-as-a-macos-time-machine-destination.79896/#post-655905)
* [Dan Langille 的博客](https://dan.langille.org/2023/09/28/creating-a-time-capsule-instance-using-samba-freebsd-and-zfs/)

---

**BENEDICT REUSCHLING** 是 FreeBSD 项目的文档提交者，也是文档工程团队的成员。过去，他曾两次担任 FreeBSD 核心团队成员。他在德国达姆施塔特应用技术大学管理一个大数据集群，并为本科生开设了“开发者的 Unix”课程。Benedict 也是每周 [bsdnow.tv](https://bsdnow.tv/) 播客的主持人之一。
