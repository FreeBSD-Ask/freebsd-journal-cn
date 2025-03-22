# 实用 Port：如何设置 Apple 时间机器

- 原文链接：[How to Set Up an Apple 时间机器](https://freebsdfoundation.org/wp-content/uploads/2022/04/Practical-Ports.pdf)
- 作者：**BENEDICT REUSCHLING**

苹果的时间机器已经存在许多年了，用于通过网络备份 Mac。它是个简单的解决方案，可以轻松设置并在后台运行，不会太打扰用户。用户可以从文件系统的以前版本中检索文件，甚至可以将整个计算机恢复到新设备，继续之前的工作。

最初，苹果为此任务提供了单独的硬件，称为 Time Capsule，但近年来，他们更多地专注于备份到付费的 iCloud 解决方案。

一些用户可能不愿将文件系统的内容托付给某台异地、超出自己控制范围的计算机。幸运的是，仍然可以设置 FreeBSD 系统在本地网络上作为 Time Capsule。本文将带你了解这种解决方案。

Time Capsule 本质上是项服务，它监听来自时间机器协议的传入连接，然后将提交的数据（最新的备份增量）存储在本地文件系统中。OpenZFS 的爱好者相信内置的数据完整性特性可以保持数据的完整性，这就是为什么我们将时间机器和 OpenZFS 结合使用。在我们将 Mac 上最宝贵的文件托付给 FreeBSD 系统时，我们希望确保有一天可以在迫切需要时恢复它们。

另一个考虑因素是噪音和功耗。由于这个本地 Time Capsule 系统需要 7*24 运行以备份和恢复文件，我们不希望选择噪音大、功耗高的解决方案。当然，你可以将服务限制为仅在我们实际使用 Mac 并且清醒的时段运行。有些人为了冗余原因，在工作时保持一个独立的时间机器，这样可以将服务限制为典型的 9 到 5 工作时间。不过，我们不希望大幅增加能源费用。与此类似，在家里或办公室桌子下运行带有噪音风扇的时间机器会打扰到同事和家人。解决这两个问题的办法是使用 ARM 嵌入式板进行任务处理。它们不仅便宜（成本低），而且采用小型化设计，不会占用像大服务器那样的空间。几乎所有的 ARM 板都没有风扇，运行时几乎没有噪音。由于 ARM 在芯片开发中专注于能源效率，你会惊讶地发现这些板只需要非常少的功率。最后，运行时间机器服务器不需要强大的计算能力，因为它主要处理 I/O。我还应该提到成本因素：购买一个小型 ARM 板加上外部存储设备应该仍然比购买苹果的 Time Capsule 便宜。也许你想将通过这种方式节省的钱捐赠给你选择的 BSD 基金会，以支持操作系统的持续开发？

我在一台连接了外部存储的树莓派 3 上运行了时间机器备份一段时间，期间没有出现任何问题。你可以在 FreeBSD 支持的一切最新 ARM 板上构建此解决方案（即启动并能够安装软件包），它不一定非得是树莓派。所需的配置并不复杂，待设置完成，你就可以忘记它，因为它不需要持续关注。只要你有一些外部存储，就可以从 Mac 开始进行备份。之所以使用外部存储，是因为板上的闪存通常容量有限，并且在频繁写入时可能会磨损。你可以先使用一个外部磁盘，等到下次工资到账时，再创建 ZFS 镜像。当 FreeBSD Time Capsule 的磁盘空间不足时，时间机器协议会自动删除旧的备份以腾出空间。无需干预，一切都能自行运行。

在本文中，我们假设你已经在板上运行了 FreeBSD，并且已连接到网络和外部存储。它应该足够强大来运行 ZFS，这是我偏好的解决方案。如果你的板的硬件不足以支持 ZFS，可以使用 UFS，它同样能很好地运行。按照 FreeBSD 手册的指导创建一个 ZFS 池来开始。之后我们将在其上创建必要的数据集，作为设置的一部分。

首先，我们需要安装两个主要的包及其依赖项，以提供时间机器服务。


```sh
# pkg install netatalk3 avahi-app
```

Avahi 能在本地网络上以简单的方式发现时间机器服务。时间机器基于 Apple 文件服务器协议，即 Apple Talk 版本三。二者配合使用，将使配置好的 Time Capsule 在网络上可用，从而使 Mac 可以找到并自动将数据备份到它。安装过程应该不会太长，具体取决于你的 ARM 服务器（或运行该服务的任何服务器）的性能。你也可以选择在 jail 中运行该服务，为备份专门创建一个 jailed 数据集。为了让本文足够简洁，我将这个任务留给你作为练习。

来自 Mac 的备份文件应该为每个用户存储在他们自己的目录中。这样，我们可以将不同用户的备份分开，并且允许其他人也进行备份。假设我们的 ZFS 池名为 backup（名字不重要）。以下命令将创建一个新数据集，用于存储所有用户的备份文件，并将它们保存在各自的目录中。

```sh
# zfs create backup/timemachine
```

我们还设置了一些 zfs 选项，以防它们尚未在池级别上设置。

```sh
# zfs set atime=off backup/timemachine
# zfs set refquota=1T backup/timemachine
# zfs set refreservation=1T backup/timemachine
# zfs set compression=zstd backup/timemachine
```

第一个选项禁用了文件系统访问时间，而这是我们运行时间机器时不需要的。`refquota` 和 `reservation` 选项确保为备份分配的池存储空间为 1 TB，但无论池中非时间机器文件占用多少空间，这部分空间都会得到保障。根据你的池大小和需求调整这一设置。不过，不要设置得太低，否则你将无法备份太多内容，而且旧的备份会被更频繁地删除。最后一个选项启用数据集压缩。请注意最后选项中设置的压缩算法。你的 ARM 板可能无法提供足够的 CPU 功率来进行压缩，因此可以更改为其他算法或完全禁用压缩。在我的时间机器上，压缩比率较低（目前为 1.01x），但这可能取决于你从 Mac 备份的文件类型以及它们的压缩效果。

接下来，让我们创建一个名为 `timemachinists` 的组，允许他们使用时间机器服务。你不希望吵闹的邻居使用你的时间机器进行备份，但也许可以允许办公室里的新同事备份她的 Mac。

```sh
# pw grouadd timemachinists
# pw groumod timemachinists -m bcr
```

使用 `pw groupshow timemachinists` 检查此操作的结果。目前我是该组中的唯一用户。你也可以选择不同的组名，只要在下面我们将展示的配置文件中引用它即可。每个用户应该有自己的数据集，因此让我们为自己创建一个并设置适当的权限：

```sh
# zfs create backup/timemachine/bcr
# chown bcr:timemachinists /backup/timemachine/bcr
# chmod 0700 /backup/timemachine/bcr
# chmod 0777 /backup/timemachine
```

我只允许自己访问 `bcr` 数据集，其他 `timemachinists` 组的成员应该被允许查看我的宝贵备份。尽管文件以数据库格式存储，而不是像在我的 Mac 上那样存储，但我不敢掉以轻心。在 `/backup/timemachine` 数据集上，权限更宽松，以便服务能够正确访问它。现在，让我们看看如何在时间机器配置文件中引用这个组和挂载点。配置文件位于 `/usr/local/etc/afp.conf`，包含主要的时间机器设置。要开始，应该进行以下配置更改：

```ini
[Global]
afp listen = 10.20.30.40
uam list = uams_dhx.so,uams_dhx2.so
mimic model = TimeCapsule6,116
disconnect time = 1
unix charset = UTF8
cnid scheme = dbd
file perm = 0640
directory perm = 0750
hostname = "TimeMachine"
hosts allow = "10.20.30.0/24"
zeroconf = yes
log file = /var/log/afp.log
log level = default:info
vol preset = TimeMachine
vol dbpath = /var/netatalk/CNID/$u/$v/

[TimeMachine]
path = /backup/timemachine/$u
time machine= yes
valid users = @timemachinists
```

以下是各个选项逐行的解释，从“Global”部分开始：

“afp listen”行定义了运行服务并监听传入连接的主机名或 IP 地址。这是用户稍后在 Mac 上的时间机器设置中配置的地址。这里我们选择了 10.20.30.40 作为示例，你需要根据自己的本地网络进行调整。

“uam list”指定了允许的用户访问方法。我们在这里使用的是默认选项，允许使用 Diffie-Hellman 交换协议（版本 1 和 2）编码的密码。其他可能的选项允许访客访问（大多数人不希望启用）和 Kerberos V，这在企业环境中可能会有用。

如果你在乎连接的 Mac 上显示什么图标，“mimic model”选项适合你。如果你怀旧，可以在这里显示 PowerBook（它比时间机器早几年发布）。不过，使用 Time Capsule 选项在这里是最合适的。

一个更有用的选项是“disconnect time”。有时连接可能会中断，而时间机器服务仍然保持会话开启，导致进一步连接尝试时出现“卷正在使用”错误消息。此选项会在 1 小时后清理断开的会话。如果你经常遇到此错误，可以调整此值，但使用这里的设置时，你不应该经常遇到此问题。

“cnid scheme”是用于卷后端的数据库。这是一个数据库，用来追踪已备份的文件、文件何时发生了更改以及其他管理信息。你无需了解太多有关此数据库的内容，保持默认的 dbd 对大多数人来说是可以的。

“file perm”和“directory perm”两个选项分别定义了连接客户端的文件和目录应以何种权限存储。这些权限可能比原始权限更严格或更宽松，但它们会带来比止痛药能缓解的更多麻烦，因此建议使用我们在此处推荐的设置。

“hostname”参数是你的时间机器的描述，Mac 用户在正确配置后点击状态栏中的时间机器图标时会看到该描述。如果你在这里选择一个有趣的描述，将会在备份之间显示类似“Last backup on black hole”的状态消息。你很容易就被逗乐了，是吧？

记得那个不该使用你宝贵时间机器存储的吵闹邻居吗？“hosts allow”选项限制了服务的访问主机或网络，排除其他所有人。如果你正确设置，只有与你共用四面墙的人才能进行备份，尽管周围的同事可能会抱怨。

记录时间胶囊的活动是调试用户可能遇到的问题的好方法。“log file”指定了日志文件的位置，“log level”指定了日志消息的详细程度和数量。我们在这里使用的设置适合日常操作，同时也不会因为过多的写入操作而使你的板卡存储过度负担。在调试问题时，你可以临时将其设置为警告或错误以获取更多详细信息，并重启服务。

通过“vol preset”我们在同一配置文件中定义了与卷相关的设置。通过这种方式可以在同一配置文件中配置多个卷，但对我们而言，一个卷就足够了。

“General”部分的最后一个选项是“vol dbpath”。记得我们可能有不同的用户将来会使用此服务吗？使用此设置，每个用户将在子目录中获得自己的独立设置。我的卷可能叫做“bcrvol”，它的设置会存储在 /var/netatalk/CNID/bcr/bcrvol/ 下（分别替换我的用户名和卷名）。通过按用户和卷分别进行分离，而不是为所有用户使用同一个目录，能够有效避免文件损坏的问题。如果发生问题，它只会影响到一个用户。这种情况不会经常发生，但还是“预防胜于治疗”（尤其是当涉及到备份时）。

我们还没有完全完成这个文件的配置，但最后几个选项都足够直接。我们在这里引用的 [Global] 部分涉及谁可以访问以及他们应被允许访问的地方。

“path”选项是某个用户的备份存储位置。由于我们无法预知将来会有哪些用户，而且每当有新用户加入时需要调整此文件，所以我们在此处使用了 \$u 变量。这样，我自己的文件将存储在 /backup/timemachine/bcr 中，并与我们之前创建的数据集相对应。不要搞混：路径是备份的 Mac 文件最终所在的文件系统路径。使用类似变量的“vol dbpath”选项是存储关于备份的管理数据库和元信息的地方。更好的是，待服务运行起来，你完全不需要访问这两个目录，可以忘记它们。

“valid users”行指定了允许访问的用户，或者在我们这种情况下，是 timemachinists 组（通过前缀 @ 区分用户）。看看它的灵活性吧？将来，当我们想要允许 Susan Sunshine 也备份她的 Mac 时，我们只需要为她创建一个用户，将其添加到 timemachinists 组中，并在 /backup/timemachine 下创建一个数据集并为她设置适当的权限。

```sh
# pw groumod timemachinists -m susan
# zfs create backup/timemachine/susan
# chown susan:timemachinists /backup/timemachine/susan
# chmod 0700 /backup/timemachine/susan
```

不需要再重新访问这个配置文件，因为变量和组定义会自动识别新用户。如果你仍然需要查看，可以在 afp.conf(5) 中找到每行配置的详细描述。现在，我们保存并关闭这个文件。

就像 Darth Vader 在旧主人的面前感受到原力的震动一样，我们希望通过 Avahi 自动通知 Mac 客户端有关你的时间机器服务的存在。为此，我们在以下位置创建一个新文件：

```sh
/usr/local/etc/avahi/services/afp.service 
```
并添加如下内容：

```xml
<?xml version="1.0" standalone='no' ?><!--*-nxml-*-->
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">
<service-group>
<name replace-wildcards="yes">%h</name>
<service>
 <type>_afpovertcp._tcp</type>
 <port>548</port>
</service>
</service-group>
```

这基本上定义了 AFP 协议监听的位置（来自 afp.conf 的“afp listen”行加上这里定义的端口 548）。有了这个文件，我们只需启用并启动服务，就可以完成服务器端的设置。

```sh
# service dbus enable
# service avahi_daemon enable
# service netatalk enable
```
如果你在想，dbus 是在安装软件包时作为依赖项引入的。它需要与其他服务一起运行：avahi 用于网络发现，netatalk 因为时间机器仅支持 Apple 文件协议（AFP）。

让我们立即启动这些服务，接着进行客户端配置。

```sh
# service dbus start
# service avahi_daemon start
# service netatalk start
```

检查输出：

```sh
# sockstat -l
```

侦听各自端口上的守护进程，并查看 `/var/log/afp.log` 是否有任何错误。

在需要备份到新创建的时间机器的 Mac 上，我们需要确保时间机器识别这个（非苹果官方的）网络卷。为此，请在 Terminal.app 中运行以下命令：

```sh
$ sudo defaults write com.apple.systempreferences TMShowUnsupportedNetworkVolumes 1
```
接下来，前往 Finder，按 CMD-K（或从菜单中选择“前往”，然后选择“连接到服务器...”），并输入以下内容：

```sh
afp://10.20.30.40
```

用在 afp.conf 中配置的地址或主机名替换我的示例，然后点击连接按钮。输入你在时间机器服务器上的用户名和密码（在我的例子中是 bcr）。如果一切顺利，共享文件夹将通过网络挂载，并在 Finder 的左侧边栏中显示为空驱动器。如果映射失败，请再次检查服务器的日志文件，确保你输入了正确的 IP 地址和用户凭据。

打开系统偏好设置中的时间机器，点击“添加或移除备份卷”。在这里，你应该能看到来自服务器的挂载共享。选择它，并为了额外的保护，勾选“加密备份”选项。这是唯一一次可以这样做，之后就无法更改了！是的，你也可以使用 ZFS 加密数据集，但对于我的备份需求，我对这个设置已经很满意。

时间机器现在将开始创建初始备份，这将根据你 Mac 上文件的数量和大小花费很长时间。你可以阅读本期杂志中的其他文章来打发时间。待初始备份完成，服务将自动断开连接，并在定期间隔重新连接，复制已经更改的文件。通过创建一个小的文本文件进行备份测试，等待它备份后再删除。假装你不小心删除了一个重要文件，然后通过时间机器对话框恢复它，松一口气。这就完成了，恭喜你成功创建了一个便宜、安静、节能的新备份服务。

---

**BENEDICT REUSCHLING** 是 FreeBSD 项目的文档提交者，也是文档工程团队的成员。他是 FreeBSD 基金会董事会的副主席。过去，他曾担任 FreeBSD 核心团队成员两届。他在德国达姆施塔特应用科技大学管理一个大数据集群，并且为本科生讲授“Unix for Developers”课程。Benedict 还是每周 [bsdnow.tv](https://www.bsdnow.tv/) 播客的主持人之一。


