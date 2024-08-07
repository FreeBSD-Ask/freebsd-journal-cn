# 从零开始的 ZFS 镜像及 makefs -t zfs

BY MARK JOHNSTON 马克·约翰斯顿

长期以来，FreeBSD 项目在其下载站点上提供了虚拟机（VM）磁盘镜像：只需访问 https://download.freebsd.org/snapshots/VM-IMAGES ，即可找到一系列预构建的镜像可供下载。这些镜像采用多种格式，受到 QEMU、VirtualBox、VMWare 和 bhyve 等常见虚拟化平台的支持。FreeBSD 项目同样为各种云平台（如 EC2、Azure 和 GCP）分发镜像。作为 FreeBSD 用户，您只需选择镜像并创建实例，在几秒钟内即可获得一个完全安装的 FreeBSD 系统。

对于许多用户来说，预构建的镜像已经足够，但您可能有一些特殊要求，这些镜像无法满足。特别是，直到最近，FreeBSD 项目的所有官方镜像都使用 UFS 作为根文件系统。当然，您仍然可以通过几种策略之一在 VM 中使用 ZFS：

1. 保持根文件系统使用 UFS，但添加额外的磁盘，并将它们用于支持 ZFS 池。
2. 使用 VM 中的 FreeBSD 安装媒体引导，并在虚拟磁盘上安装 FreeBSD，使用 ZFS 作为根文件系统。生成的 VM 映像可以克隆并用作其他映像的模板。
3. 通过设置 md(4) 磁盘并在该磁盘上创建和导入 ZFS 池，手动创建映像，然后可以在该池中安装 FreeBSD。例如 poudriere image 目前是这样工作的。

尽管这些策略是有效的，它们都有一些注意事项：

* 选项 1）使得使用启动环境变得困难;
* 选项 2）需要手动努力来创建和自定义模板镜像;
* 选项 3）需要 root 权限，无法在jail内完成（目前在jails中禁止创建 ZFS 池）。

長時間以來，我一直想在本地構建基於 ZFS 的 VM 映像，這樣我就可以在 UFS 和 ZFS 上運行 FreeBSD 回歸測試套件，因此在 2022 年，我開始研究如何擴展我們已經使用來製作 UFS 映像（即 makefs(8) ）的工具，以支持某種形式的 ZFS。

## makefs(8)

FreeBSD 項目使用 makefs(8) 創建官方 VM 映像，這是一個源自 NetBSD 的實用程序。它接受一個或多個路徑作為輸入，並生成一個包含用這些路徑的內容填充的文件系統映像的單個文件。

makefs 支持多种文件系统，包括 UFS、FAT 和 ISO9660。在这里使用它的基本思路是将 FreeBSD 安装到临时目录（例如，使用 make installworld ），然后将 makefs 指向该目录。结果是生成一个包含 FreeBSD 安装内容的文件系统镜像的文件。

对于熟悉从源代码构建 FreeBSD 的用户，下面的示例可能会提供更清晰的图片：

`# make installworld installkernel distribution DESTDIR=/tmp/foo# makefs -t ffs fs.img /tmp/foo# mdconfig -f fs.imgmd0# mount /dev/md0 /mnt# ls /mnt/bin/sh/mnt/bin/sh`

这将在 /tmp/foo 安装预构建的 FreeBSD 发行版副本，然后使用 makefs 在 fs.img 中生成文件系统映像。可以使用 mdconfig(8) 命令通过创建由该文件支持的字符设备方式来挂载此映像。/tmp/foo 目录中的文件属性，如模式位和时间戳，在生成的映像中将被保留。

相反，FreeBSD 的传统“现场”安装可能看起来更像这样：

`# truncate -s 50g fs.img# mdconfig -f fs.imgmd0# newfs /dev/md0/dev/md0: 51200.0MB (104857600 sectors) ...# mount /dev/md0 /mnt# cd /usr/src# make installworld installkernel distribution DESTDIR=/mnt# umount /mnt`

这里，我们使用 newfs(8) 实用程序在目标设备上初始化一个空文件系统，然后将文件复制到其中。当然，这样做也有缺点，类似于我之前提到的创建 ZFS 池的问题： newfs(8) 需要 root 权限，并且生成的镜像是不可复制的。也就是说，如果这样从相同的预构建 FreeBSD 发行版创建两个镜像，它们不会是完全相同的，例如，因为两个镜像之间的文件访问和修改时间会有稍微不同。可复制性是构建系统的一个重要安全属性，因为它更容易检测到构建输出的恶意篡改。

我之前就很熟悉 makefs ，因为我曾经编写脚本来创建自己使用的 VM 镜像。正如我提到的，我想类似地构建 ZFS 镜像，而我并不孤单；一个常见的用户投诉是，尽管 ZFS 是 FreeBSD 上根文件系统的非常受欢迎的选择，但所有 FreeBSD 官方的云镜像都是基于 UFS 的。因此，我花了一些时间考虑 makefs 生成的 ZFS 镜像可能是什么样的。

## makefs(8) 遇见 ZFS

那么 makefs 实际上是做什么？对这个问题的技术性答案需要一些文件系统内部知识，简而言之： makefs 初始化一些全局文件系统元数据，例如 UFS/FFS 超级块，然后遍历输入的目录树，将它们的内容复制到镜像中，并添加元数据，例如指向文件数据的目录条目。传统上，一个空文件系统然后通过诸如 cp(1) 、 makefs 等工具向其添加数据，而 makefs 则在单个操作中生成一个填充的文件系统。因此，虽然这意味着 makefs 需要了解文件系统数据和元数据在磁盘上的排列方式，但它可能比内核对文件系统的实现要简单得多。

makefs 永远不需要通过名称查找文件，处理空间不足的情况，执行缓冲缓存或删除文件，例如。

ZFS 是一个庞大而复杂的系统，但根据上述观察，一个假设的 makefs -t zfs 可以忽略许多细节。对我来说这很重要：我当时并不是 ZFS 专家，目前也不是，对其磁盘上的格式了解甚少，因此简单才是关键。现在我们可以问： makefs -t zfs 究竟应该做什么？

我的目标是支持使用 ZFS 作为根文件系统创建 VM 镜像。更具体地说， makefs 需要做到：

1. 创建一个只有单个磁盘 vdev 的 ZFS 池。不需要支持 RAID 或镜像布局，因为对于 VM 镜像来说，额外的冗余性并不是很有用。
2. 在池中创建至少一个数据集。数据集需要作为根文件系统可挂载。实际上，FreeBSD-on-ZFS 安装时会预先创建大约十几个数据集，但为简单起见，初始概念验证可以忽略这一点。
3. 使用输入目录树的内容填充数据集。更具体地说，对于每个输入文件， makefs 需要分配一个 dnode 并将文件复制到镜像中的某个位置。它还需要复制输入文件的属性，如文件权限。

特别是，许多影响磁盘布局的 ZFS 特性可以简单忽略。例如，不需要 makefs 处理压缩或快照。因此，虽然任务仍然有些艰巨，但通过排除除最低必要功能之外的所有功能，似乎是完全可行的。

## 第一次尝试：libzpool

作为 FreeBSD 内核开发人员，我已经有一些关于 OpenZFS 内部的经验，但 ZFS 是一个复杂的东西。代码被分成许多不同的子系统，其中大多数并不知道数据实际上是如何在磁盘上布局的，而我对那些了解如何布局的子系统没有任何经验。然而，事实证明，人们可以将 OpenZFS 内核模块编译为用户空间库： libzpool.so 。这主要用于测试 OpenZFS 代码本身，但看起来像是我项目的一个绝佳起点： libzpool.so 完全了解 ZFS 的磁盘布局，因此我认为我可以避免过多地学习它，而是编写使用高级操作的代码，类似于像 zpool create 这样的命令只是要求内核在一组 vdev 上创建一个池。

不详细探讨，这种方法最终产生了一个可行的原型，但最终成为了一条死胡同。其中一些原因：

* libzpool.so 不适合“生产”应用：它没有稳定的接口，而且我的原型实际上在使用未记录的内核 API。如果我坚持这种方法，结果将会很脆弱且难以维护。
* libzpool.so 中的代码大部分是未经修改的内核代码，因此会创建大量线程并将文件数据缓存到 ARC 中，这对 makefs 的目的来说是不必要的。由此带来的一个后果是原型非常慢，并且可能消耗大量系统内存，有时会触发内存杀手。
* 结果是不可复现的。如果我用相同的输入运行原型两次，输出将不会完全一样。

尽管我不得不丢弃大部分原型，但写作对我来说是一次有益的学习经历，并激励我尝试不同的方法。

此时，我似乎必须动手学习一下 ZFS 的磁盘布局。我意识到 FreeBSD 引导加载程序会有类似的问题：为了从 ZFS 池引导 FreeBSD，加载程序需要能够找到内核文件并将其加载到内存中。加载程序在受限环境中运行，因此不能使用内核的 ZFS 代码，显然其他人已经解决了类似的问题。

## 尝试 #2：从头开始的 ZFS

幸运的是，在互联网上流传着一份 ZFS 磁盘布局规范，虽然相当不完整和过时，但总比没有好得多。除此之外，我还可以查看引导加载程序的代码。在某种意义上，它解决了 makefs 相反的问题：它只是打开并从 ZFS 池中读取数据，而不写入任何内容，而 makefs 创建一个新的池，但不需要能够读取现有的池。

与引导加载程序的双重性非常有用：我可以编写代码来创建一个池，然后通过尝试使用引导加载程序从池中读取文件（内核）来测试它。更具体地说，我会首先将 FreeBSD 内核安装到一个临时目录：

`$ cd /usr/src$ make buildkernel$ make installkernel DESTDIR=/tmp/test -DNO_ROOT`

`Then I can create a ZFS image and try to load it using the legacy bhyve loader:`

$ makefs -t zfs -o poolname=zroot zfs.img /tmp/test $ sudo bhyveload -c stdio -d zfs.img test

这里， bhyveload 正在使用 /boot/userboot.so ，这是一个编译成在用户空间运行的 FreeBSD 引导加载程序的副本。它有大部分真实引导加载程序的功能，但它不像使用 BIOS 调用或 EFI 引导服务来自磁盘读取数据，而是使用熟悉的 read(2) 系统调用从镜像文件 zfs.img 获取数据。

最初的目标是让 userboot.so 找到并加载位于 /boot/kernel/kernel 中的内核在 zfs.img 中。这是一个非常方便的测试工具，因为我可以轻松地附加调试器到 bhyveload 或向加载程序添加打印语句并重新编译 userboot.so 。我的第一个里程碑是让 vdev_probe() 将图像识别为有效的 ZFS 池。

## vdev 标签和 uberblock

vdev_probe() 检查磁盘，看它是否属于一个 ZFS 池；也就是说，它确定磁盘是否看起来是一个 vdev ，如果是，就开始加载更多元数据：

`/** Ok, we are happy with the pool so far. Lets find* the best uberblock and then we can actually access* the contents of the pool.*/vdev_uberblock_load(vdev, spa->spa_uberblock);`

ZFS 磁盘规范的第一章详细描述了 vdev 标签和 uberblock。简单来说，一个 vdev 包含一个元数据块，即 vdev 标签，其中包含描述所属池的元数据，以及“uberblock”的副本，指向 vdev 元数据树的根。因此，为了让 userboot.so 能够找到我的池，我编写了代码，向输出镜像文件添加 vdev 标签。

到这一点上， makefs 已经在使用特定于 ZFS 的数据结构，如 vdev_label_t 和 uberblock_t 。与其复制引导加载程序使用的定义， makefs 与其共享一个包含许多有用的磁盘数据结构定义的大型头文件。

## 对象集和 MOS

一旦加载程序能够探测和识别 makefs 生成的镜像，下一步就是让它挂载镜像内的数据集。处理此功能的加载程序代码主要包含在 zfs_get_root()中。

要理解 zfs_get_root() 的实现，值得阅读 ZFS 磁盘规范的第三章，其中描述了对象集。尽管规范很快进入细节，但回顾用于表示 ZFS 数据的高级结构仍然很有价值。

ZFS 有“块指针”，实际上只是指向数据块在 vdev 上的物理位置（从 makefs 的角度来看，这只是输出图像文件中的偏移量）。 ZFS 元数据对象有几十种类型，由一个 512 字节的“dnode”表示。 dnode 包含有关对象的各种元数据，例如其类型和大小，还可以包含指向其他数据的块指针。 例如，存储在 ZFS 数据集中的文件由一个 dnode（类型为 DMU_OT_PLAIN_FILE_CONTENTS ）表示，类似于传统 Unix 文件系统中的 inode。 最后，“对象集”是一个包含一组 dnode 的结构；dnode 由其所属的对象集和在数组中的索引唯一标识。

ZAP（ZFS 属性处理器）是一个包含一组键值对的 dnode。 ZAP 用于表示许多更高级别的 ZFS 元数据结构。 例如，Unix 目录由一个 ZAP 表示，其键是文件名，值是相应文件的 dnode ID。

MOS（元对象集）是池的根对象集。 超级块包含指向 MOS 的指针，并且从 MOS 可以到达池中的所有其他元数据（因此也是数据）。 有了这些信息，就更容易理解 zfs_get_root()：它获取具有 ID 1 的 dnode（期望它是 ZAP 对象），使用它查找包含池属性的 ZAP 对象并查找“bootfs”属性的值，用于查找根数据集的 dnode。

创建池时， makefs 在 pool_init() 中分配并开始填充 MOS。一旦 userboot.so 能够处理 MOS，就可以导入 makefs 生成的池，从那时起，我开始使用 zdb(8) 来检查生成的池。zdb 的命令行用法相当晦涩，但像这样简单的调用

`# zdb -dddd zroot 1`

来自 MOS 的 dnode 1 的转储非常有助于弄清楚 OpenZFS 在导入池时希望看到的内容。例如，转储 ZAP 对象时， zdb 可以打印 ZAP 中的所有键值对。许多配置 ZAP 键具有值，这些值是 dnode ID，因此 zdb 可以轻松用于检查池和数据集配置的不同“层”。

## 数据集和文件

ZFS 数据集具有名称，并且组织成一棵树。根数据集的名称与池自身相同（例如，“zroot”），子数据集的名称将以父数据集的名称为前缀。虽然我为 makefs 的 ZFS 支持初始原型自动将所有文件放置在根数据集中，但这并不足以能够创建基于 ZFS 的虚拟机镜像： bsdinstall 和其他 FreeBSD 安装程序会自动创建许多子数据集。其中一些，如 zroot/var ，从不挂载，而只存在以提供仅由子数据集继承的设置，例如 zroot/var/log 。我的目标是让 makefs 能够创建与 bsdinstall 提供的布局匹配的数据集树。

发行镜像构建脚本演示了创建多个数据集的语法。每个数据集由一个包含数据集名称和以分号分隔的属性列表的 -o fs 选项描述。目前，仅支持少量属性 - 如 zfsprops(8) 手册页面中所述。

当 makefs -t zfs 完成初始化各种结构之后，它开始处理输入目录树。每个输入文件由一个 fsnode 结构表示，这些结构被组织成代表文件树的树。首先， makefs 确定每个已挂载数据集的根对应的 fsnode 。然后，它遍历 fsnode 树，为每个文件分配一个 dnode；这发生在确定分配 dnode 的对象集的数据集的上下文中。

拷贝常规文件 makefs 会从当前对象集中分配一个 dnode，并在循环中从输出文件分配空间块，并将输入文件的数据复制到这些分配中。ZFS 支持从 4KB 到 128KB 的 2 的幂块大小，因此较小的文件不会产生过多的内部碎片。镜像中的所有分配块都使用位图进行跟踪，该位图由 vdev_space_alloc()函数更新。

位图跟踪的分配空间必须记录在输出镜像中；ZFS 使用一个称为“空间映射”的中心数据结构来跟踪 vdev 当前分配的区域。 makefs 使用位图作为所有块分配的内部表示，并在镜像生成的最后步骤之一生成空间映射，一旦完成所有块分配。

## 结论

将 ZFS 支持添加到 makefs 中花了相当多的努力，但最终结果是我相信将对许多 FreeBSD 用户有用，同时避免了大量的维护负担。在 makefs 中有大约 2,600 行 ZFS 特定代码（总共 15,000 行中的一部分），这相当少。还有一个回归测试套件，提供了很好的覆盖范围。

在这 2600 行代码中，超过 100 行都是对 assert() 的调用，所以只需验证不变量即可。这些断言在开发过程中非常有用，因为大量代码是以不完整的方式编写的，只是为了让引导加载程序正常工作，并在后来更充分地展开；它们用于记录各种功能的限制，并在我添加越来越多的功能时帮助捕获许多错误。

现在 FreeBSD 14.0 已经发布，并且支持使用 ZFS 的根 VM 映像，希望许多用户能够利用这个新功能。在发布周期内发现并修复了一些错误，因此至少有一些用户已经尝试过。目前没有计划为 makefs -t zfs 添加任何增强功能，但这可能会根据反馈而改变 - 如果您发现有改进的余地，请提交错误报告。

MARK JOHNSTON 是一位软件开发者和居住在加拿大安大略省多伦多的 FreeBSD src 开发者。在没有坐在电脑前时，他喜欢和朋友们一起参加城市躲避球联赛。
