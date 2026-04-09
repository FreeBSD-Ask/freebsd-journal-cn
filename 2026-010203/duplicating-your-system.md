# 复制你的系统：使用 duplicity 备份你的 FreeBSD 桌面

- 原文：[Duplicating your System Using duplicity to back up your FreeBSD desktop](https://freebsdfoundation.org/our-work/journal/browser-based-edition/laptop-desktop/duplicating-your-system/)
- 作者：Jason Tubnor

你刚刚安装了新的 FreeBSD 桌面（或笔记本电脑），将其设置为你喜欢的样子，正要开始创作新小说或为 FreeBSD 移植你最喜欢的软件供应商刚刚发布的新软件。然后你意识到，只需一个有缺陷的 NVMe 控制器，你就可能面临数据完全丢失的风险，你的所有工作和时间都会随之消失。

在现代，我们有 ZFS 快照和复制功能，使得备份任务变得极其简单。然而，你可能无法访问另一个 ZFS 端点进行复制，需要通过使用非 ZFS 工具进行备份来分散风险，或者你只是使用 UFS2，过着复古的生活。

对于这些情况，我的第一个建议是使用 Tarsnap，因为它易于使用且简单，使恢复与备份一样容易。但某些情况需要不同的方法。也许你公司有严格的防火墙，不允许 Tarsnap 数据流从公司网络流出，或者你可以轻松访问内部存储端点，例如兼容 S3 的对象存储或具有 SFTP 访问权限的大文件存储位置。

当你面临后者时，可以在 FreeBSD 系统上作为易于安装的软件包使用 duplicity（ports 中的 sysutils/duplicity）工具：

```sh
# pkg install duplicity par2cmdline
```

在上述情况下，还安装了 par2cmdline 软件包，因为它不是 duplicity 的依赖项，但在本文中（以及 duplicity 使用）用于避免备份存档中的脏位。

Duplicity（[https://duplicity.gitlab.io/](https://duplicity.gitlab.io/)）将把你的各种文件和文件夹备份到加密的 tar 格式卷中，同时使用 librsync 来提高压缩率并对数据执行差分操作。虽然它没有一些使 Tarsnap 吸引人的真正好功能，但它确实提供了一个适用于许多用例的替代方案。

虽然 duplicity 有许多目标存储选项，但我们将在这里专注于使用兼容 S3 的对象存储，因为它可以从许多云提供商那里轻松获得，具有不同的成本、访问速度和耐用性。duplicity 的工作方式更像是一种交互式方法（尤其是在清理旧数据集时）。虽然你可能成功地将备份提交到 AWS Glacier 或类似的对象存储，但数据的恢复和管理将极其缓慢、复杂且昂贵；你已被警告过了。不要追求冷、便宜的存储；选择稍高/更热的存储以获得更好的体验。虽然它是交互式的，
但当从 cron 调用、在脚本中执行或在交互式命令行上使用时，它会完美工作。

开始之前，我们需要定义备份数据的参数和最佳实践。关键考虑因素是：

* 每月新的完整备份 — 一长串增量备份虽然备份速度快，但可能会超出你的恢复时间目标（RTO）。引入频繁的完整备份将确保 RTO 窗口保持较低。
* 加密你的备份 — 你正在备份到别人的计算机上。除非你控制远程计算机，否则数据应该在你的计算机上加密，然后再通过网络传输。
* 不要信任端点耐用性 — 使用 par2 功能在备份中加入奇偶校验块，以防脏位。默认值为 10%，这可能会增加大型存档的成本。评估你的风险并相应调整。在本示例中，我们将使用 5%。

准备 AWS 环境变量。这些可以放入你主目录中的 .duplicity.env 文件中。如果你在多用户系统上，确保 .duplicity.env 文件的权限为 chmod 0600，以防止其他系统用户获得对私有 API 密钥的未授权访问：

```sh
~/.duplicity.env

export AWS_ACCESS_KEY_ID="<key_id>"
export AWS_SECRET_ACCESS_KEY="<access_key>"
export AWS_ENDPOINT_URL="https://sg-s3.storage.bunnycdn.com"
```

然后加载我们的环境文件以供使用：

```sh
$ . .duplicity.env
```

在我们开始使用 duplicity 之前的最后一个要求是，当使用 GPG 时（默认使用加密），为你的数据的对称加密定义一个 PASSPHRASE 密钥。可以使用 GPG 公钥加密，但这超出了本文的范围（有关更多信息，请参阅 duplicity 手册页）。PASSPHRASE 变量也可以在我们上面使用的环境文件中定义，然后从你的备份脚本中调用。为了便于记录，我们将在此处导出：

```sh
$ export PASSPHRASE="4640f58235c4ff7ad359fdcb5f2d8ac48f71c26bcff4a40f6d85503d0288a45e"
```

现在我们准备从我们新配置的系统执行主目录的第一次备份：

```sh
$ duplicity backup –full-if-older-than 1M –exclude ./.cache \
–par2-redundancy 5 ./ par2+s3:///mybucket-s3backup/computer
```

上述命令启动 duplicity 的备份组件，如果前一个完整备份超过一个月，则创建一个完整备份，排除 .cache 目录，将 par2 冗余设置为 5%，从当前目录（用户主目录的根目录）获取所有文件和文件夹，并将备份写入 mybucket-s3backup 存储桶中的 /computer 文件夹，使用 par2 包装器进行块冗余。

duplicity 完成运行后，你将看到备份的统计信息：

```sh
Last full backup date: none
Last full backup is too old, forcing full backup
————–[ Backup Statistics ]————–
StartTime 1773460673.41 (Sat Mar 14 14:57:53 2026)
EndTime 1773460673.42 (Sat Mar 14 14:57:53 2026)
ElapsedTime 0.01 (0.01 seconds)
SourceFiles 15
SourceFileSize 6302 (6.15 KB)
NewFiles 15
NewFileSize 6302 (6.15 KB)
DeletedFiles 0
ChangedFiles 0
ChangedFileSize 0 (0 bytes)
ChangedDeltaSize 0 (0 bytes)
DeltaEntries 15
RawDeltaSize 6277 (6.13 KB)
TotalDestinationSizeChange 3884 (3.79 KB)
Errors 0
————————————————-
```

由于这是一个干净的系统，我们的主目录还没有太多活动，并且相当小，所以上面的备份没有花很长时间执行。我们备份的初始总量：

```sh
$ du -sh .
62K    .
```

如你所见，即使在小数据集上，62KB 的文件（分配的磁盘空间），6.15KB 是实际文件总量，考虑到压缩和加密后，目标大小为 3.79KB。

测试表明，duplicity 的工作速度与你的硬件允许的一样快；限制将由你的对象存储速度和网络出站吞吐量的大小决定。要观察备份的执行情况，请在命令行上执行 duplicity 时使用 –progress 开关。

此时，你的数据在远程存储上是加密的，如果你丢失了对称加密密钥（PASSPHRASE），即使是你也无法访问。花时间将密钥的副本放在安全的地方，以防你需要将数据恢复到新系统。

让我们查看我们第一次备份中的内容：

```sh
$ duplicity ls par2+s3:///mybucket-s3backup/computer/

Last full backup date: Sat Mar 14 14:57:52 2026
Sat Mar 14 14:00:35 2026 .
Sat Mar 14 13:33:22 2026 .boto
Sat Mar  7 15:55:14 2026 .cshrc
Thu Mar 12 22:15:04 2026 .gnupg
Thu Mar 12 22:15:04 2026 .gnupg/common.conf
Thu Mar 12 22:15:04 2026 .gnupg/private-keys-v1.d
Sat Mar 14 14:56:10 2026 .gnupg/random_seed
Sat Mar 14 14:00:35 2026 .lesshst
Sat Mar  7 15:55:14 2026 .login
Sat Mar  7 15:55:14 2026 .login_conf
Sat Mar  7 15:55:14 2026 .mail_aliases
Sat Mar  7 15:55:14 2026 .mailrc
Thu Mar 12 22:13:12 2026 .profile
Sat Mar 14 13:47:32 2026 .sh_history
Sat Mar  7 15:55:14 2026 .shrc
```

duplicity 中的 list (ls) 命令显示存档，类似于 tar -t。同样，这里已经使用 par2 包装器连接到 mybucket-s3backup，以验证每个块的一致性并实时修复。

现在是时候在我们的主目录中开始工作了。让我们先克隆 ports 分支：

```sh
$ git clone –depth 1 https://git.freebsd.org/ports.git
Cloning into 'ports'…

$ du -sh .
1.0G    .
```

这大大增加了我们的主目录。现在是时候跨多个端口备份我们的工作了：

```sh
$ duplicity backup –full-if-older-than 1M –exclude ./.cache \
–par2-redundancy 5 ./ par2+s3:///mybucket-s3backup/computer

Local and Remote metadata are synchronized, no sync needed.
Last full backup date: Sat Mar 14 14:57:52 2026
————–[ Backup Statistics ]————–
ElapsedTime 102.56 (1 minute 42.56 seconds)
SourceFiles 214568
SourceFileSize 682126695 (651 MB)
NewFiles 214554
NewFileSize 682120407 (651 MB)
RawDeltaSize 681815400 (650 MB)
TotalDestinationSizeChange 224163465 (214 MB)
————————————————-
```

如你所见，duplicity 非常高效。我们的备份仅增长了 214MB，即使当 ports 树被检出时，我们的系统报告为 1GB。

恢复备份与备份一样简单：

```sh
$ duplicity restore –path-to-restore ports \
par2+s3:///mybucket-s3backup/computer /tmp/temp/ports
Local and Remote metadata are synchronized, no sync needed.
Last full backup date: Sat Mar 14 14:57:52 2026
```

此命令仅从之前使用 par2 包装器备份到 S3 存储桶的主目录中恢复 ports 目录。ports 目录的目标位置将是 /tmp/temp/

```sh
jtubnor@disk:~ $ du -sh /tmp/temp && ls -lsa /tmp/temp

1.0G    /tmp/temp
total 18
1 drwxr-xr-x   3 jtubnor wheel    3 Mar 14 16:56 .
9 drwxrwxrwt  16 root    wheel   16 Mar 14 16:57 ..
9 drwxr-xr-x  70 jtubnor jtubnor 82 Mar 14 15:18 ports
```

上面的输出显示，我们的 1GB ports 目录现在已被恢复到 /tmp/temp 目录。如果你遇到了完整的机器故障，恢复你的主目录就像这样简单：

```sh
$ cd / && duplicity restore par2+s3:///mybucket-s3backup/computer /home/jtubnor/
```

你需要确保你当前不在要恢复到的目录中；这就是为什么我们在发出恢复命令之前先导航到根目录。

本文几乎没有触及 duplicity 中包含的许多功能的表面。如果需要，通过必要的调整，它甚至适用于整个系统配置备份。有关功能和用例示例的更多信息，请查看 duplicity 手册页（[https://duplicity.gitlab.io/stable/duplicity.1.html](https://duplicity.gitlab.io/stable/duplicity.1.html)）

Duplicity 是一个小型、成熟的程序，只有少数依赖项，但它在大多数情况下都能很好地工作，并且可以在任何 FreeBSD 系统上轻松安装。

---

Jason Tubnor 在广泛的学科领域拥有 30 余年的 IT 行业经验，目前是 Latrobe Community Health Service（澳大利亚维多利亚州）的技术运营经理。Jason 在 1990 年代中期发现了 Linux 和开源，然后在 2000 年被介绍到 OpenBSD，他使用各种 BSD 来解决不同行业组织的问题。Jason 还是每周 BSDNow 播客（[https://bsdnow.tv](https://bsdnow.tv/)）的联合主持人。
