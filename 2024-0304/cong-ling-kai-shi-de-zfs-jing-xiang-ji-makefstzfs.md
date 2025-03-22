# 从零开始的 ZFS 镜像及 makefs -t zfs

- 原文链接：[ZFS Images From Scratch, or makefs -t zfs](https://freebsdfoundation.org/our-work/journal/browser-based-edition/development-workflow-and-ci/zfs-images-from-scratch-or-makefs-t-zfs/)
- 作者：Mark Johnston

长期以来，FreeBSD 项目在下载站点提供了虚拟机 (VM) 磁盘镜像：只需访问 [https://download.freebsd.org/snapshots/VM-IMAGES](https://download.freebsd.org/snapshots/VM-IMAGES)，即可找到一系列预构建的镜像来下载。这些镜像支持多种格式，常见的虚拟化平台如 QEMU、VirtualBox、VMware 和 bhyve 都可以识别。FreeBSD 项目还为多个云平台（如 EC2、Azure 和 GCP）提供了镜像。作为 FreeBSD 用户，你只需要选择镜像并创建实例，在几秒钟内即可获得一个完全预装的 FreeBSD 系统。

对于大多数用户来说，预构建的镜像已经足够使用了，但如果你有某些特殊需求，这些镜像可能无法满足。尤其是，直到最近，FreeBSD 项目的所有官方镜像都使用 UFS 作为根文件系统。当然，仍然可通过几种策略在虚拟机中使用 ZFS：

1. 将根文件系统保留在 UFS 上，但增加额外的磁盘，并用它们来实现 ZFS 池。
2. 在虚拟机中启动 FreeBSD 安装介质，使用它将 FreeBSD 安装到虚拟磁盘上，根文件系统使用 ZFS。然后可克隆该虚拟机镜像，将其用作其他镜像的模板。
3. 手动创建镜像，设置一块 `md(4)` 磁盘，然后在该磁盘上创建并导入 ZFS 池，再在该池中安装 FreeBSD。例如，`poudriere image` 就是这样工作的。

虽然这些策略可行，但每种方法都有一些注意事项：

* 方案 1) 使得使用引导环境变得困难；
* 方案 2) 需要手动创建和自定义模板镜像；
* 方案 3) 需要 root 权限，且不能在 jail 完成（目前不允许在 jail 中创建 ZFS 池）。

在很长一段时间，我一直希望能够原生构建基于 ZFS 的虚拟机镜像，以便可以同时在 UFS 和 ZFS 上运行 FreeBSD 回归测试套件。因此，在 2022 年，我开始研究如何扩展我们已经使用的工具（即 `makefs(8)`）来支持某种形式的 ZFS。

## makefs(8)

FreeBSD 项目使用 `makefs(8)` 来构建官方虚拟机镜像，`makefs` 是一款源自 NetBSD 的工具。它接受一个/多个路径作为输入，并生成一个包含这些路径内容的文件系统镜像文件。

`makefs` 支持多种文件系统，包括 UFS、FAT 和 ISO9660。它的基本使用方法是将 FreeBSD 安装到一个临时目录（例如，使用 `make installworld`），然后将 `makefs` 指向该目录。结果是个包含文件系统镜像的文件，该镜像的根目录包含 FreeBSD 的安装文件。

对于熟悉从源码构建 FreeBSD 的用户，以下示例也许能更清楚地说明这个过程：

```sh
# make installworld installkernel distribution DESTDIR=/tmp/foo
# makefs -t ffs fs.img /tmp/foo
# mdconfig -f fs.img
md0
# mount /dev/md0 /mnt
# ls /mnt/bin/sh
/mnt/bin/sh
```

这将一个预构建的 FreeBSD 发行版安装到 `/tmp/foo`，然后使用 `makefs` 生成了一个文件系统镜像 `fs.img`。可以用命令 `mdconfig(8)` 挂载该镜像，创建一个由该文件支持的字符设备。`/tmp/foo` 中文件的属性（如模式位和时间戳）在生成的镜像中会得到保留。

与此相比，传统的 FreeBSD“live”安装过程可能会像这样：

```sh
# truncate -s 50g fs.img
# mdconfig -f fs.img
md0
# newfs /dev/md0
/dev/md0: 51200.0MB (104857600 sectors) ...
# mount /dev/md0 /mnt
# cd /usr/src
# make installworld installkernel distribution DESTDIR=/mnt
# umount /mnt
```

在这里，我们使用工具 `newfs(8)` 初始化目标设备上的空文件系统，然后将文件复制到其中。尽管这种方法可行，但它有一些缺点，类似于我之前提到的创建 ZFS 池时的问题：`newfs(8)` 需要 root 权限，并且生成的镜像不可重现。也就是说，如果从相同的预构建 FreeBSD 发行版创建两个镜像，它们不会逐字节相同，例如，因为文件的访问时间和修改时间在两个镜像之间会略有不同。可重现性是构建系统的重要安全属性，它有助于更容易检测到构建输出中的恶意篡改。

我之前已经熟悉了 `makefs`，因为我写过一些脚本来为自己的使用创建虚拟机镜像。如前所述，我希望能够以类似的方式构建 ZFS 镜像，而且我不是唯一一个有这种需求的人；常见的用户抱怨是，虽然 ZFS 是 FreeBSD 上常用的根文件系统，可 FreeBSD 所有的官方云镜像都是基于 UFS 的。因此，我花了一些时间思考 `makefs` 生成的 ZFS 镜像可能是什么样的。

## makefs(8) 与 ZFS

那么，`makefs` 实际上是干什么的呢？要回答这个问题，首先需要了解一些文件系统的内部实现，但简而言之：`makefs` 会初始化一些全局文件系统元数据，如 UFS/FFS 超级块；然后遍历输入的目录树，将其内容复制到镜像中；并添加元数据，如指向文件数据的目录条目。传统方法是从一个空文件系统开始，然后通过 `cp(1)` 等工具让内核向其中添加数据，而 `makefs` 则在单一操作中生成了一个已填充的文件系统。因此，虽然这意味着 `makefs` 需要了解文件系统数据和元数据在磁盘上的布局，但它比内核实现的文件系统要简单得多。

例如，`makefs` 不需要通过文件名查找文件，不需要处理空间不足的情况，也不需要执行缓存操作和删除文件。

ZFS 复杂且庞大，但根据上述观察，假设有 `makefs -t zfs`，它能忽略很多细节。对于我来说，这点非常重要：当时我并非 ZFS 专家，对 ZFS 磁盘格式了解很少，因此简单化是关键。此时，我们可以问：`makefs -t zfs` 应该做什么？

我的目标是支持创建以 ZFS 为根文件系统的虚拟机镜像。更具体地说，`makefs` 需要：

1. 创建一个具有单个磁盘 vdev 的 ZFS 池。由于虚拟机镜像中的冗余并不特别有用，因此不需要支持 RAID 和镜像布局。
2. 在池中创建至少一个数据集。该数据集需要能够挂载为根文件系统。实际上，使用 ZFS 安装的 FreeBSD 通常会预创建十几个数据集，但为了简化，初步的概念验证可以忽略这一点。
3. 用输入目录中的内容填充此数据集。更具体地说，对于每个输入文件，`makefs` 需要分配一个 `dnode` 并将文件复制到镜像中的某个位置。它还需要复制文件的属性，如文件权限。

特别地，很多影响磁盘布局的 ZFS 特性可以被忽略。例如，`makefs` 无需考虑压缩和快照。因此，尽管任务看起来有些艰巨，但通过排除最必要特性之外的内容，此目标似乎完全可行。


## 尝试 #1：libzpool

作为一名 FreeBSD 内核开发者，我对 OpenZFS 的内部结构已经有了些许了解，但 ZFS 是个复杂的系统。代码被划分成多个不同的子系统，其中大多数并不了解数据在磁盘上的实际布局，而且我对那些了解磁盘布局的子系统也缺乏经验。然而，事实证明，我们可以将 OpenZFS 内核模块编译为用户空间库：`libzpool.so`。这个库主要用于测试 OpenZFS 的代码本身，但对我来说，它似乎是一个很好的开始：`libzpool.so` 了解 ZFS 的磁盘布局，因此我认为我可以避免深入学习其细节，而是编写使用高层操作的代码，类似于命令 `zpool create` 如何简单地要求内核在一组 `vdev` 上创建一个池。

不深入细节，使用这种方法最终得到了一个工作的原型，但最终证明它是死胡同。以下是一些原因：

* `libzpool.so` 并不适合用于“生产”应用：它没有稳定的接口，而我的原型实际上是在使用一些没有文档的内核 API。如果继续沿着这条路走下去，结果将会非常脆弱且难以维护。
* `libzpool.so` 中的代码大部分是未修改的内核代码，因此会创建大量线程并在 ARC 中缓存文件数据，这对于 `makefs` 来说是多余的。结果是原型非常慢，并且会消耗大量系统内存，有时会触发内存溢出这个杀手。
* 结果无法重现。如果我使用相同的输入运行两次原型，输出的结果不会逐字节相同。

尽管我必须丢弃大部分原型，但编写它的过程是一次有价值的学习经验，并促使我尝试了不同的方法。

此时，我意识到我必须深入了解 ZFS 的磁盘布局。我意识到，FreeBSD 的引导加载程序会面临类似的问题：为了从 ZFS 池启动 FreeBSD，加载程序需要能够找到内核文件并将其加载到内存中。引导加载程序在一个受限的环境中运行，因此不能使用内核的 ZFS 代码，因此显然其他人已经解决了类似的问题。

## 尝试 #2：从零开始实现 ZFS

幸运的是，互联网上有本 [ZFS 磁盘布局规范](https://people.freebsd.org/~markj/Zfs_ondiskformat.pdf)；它不太完整且过时，但总比什么都没有要好。除此之外，我还可以查看引导加载程序的代码。从某种意义上说，引导加载程序解决了与 `makefs` 相反的问题：它只读取 ZFS 池中的数据，而不写入任何内容，而 `makefs` 创建了一个新的池，但不用读取现有的池。

引导加载程序的双重性非常有用：我可以编写代码创建一个池，然后通过尝试使用引导加载程序从池中读取文件（内核）来测试它。更具体地说，我首先将 FreeBSD 内核安装到一个临时目录：

```sh
$ cd /usr/src
$ make buildkernel
$ make installkernel DESTDIR=/tmp/test -DNO_ROOT
```

然后我可以创建一个 ZFS 镜像，并尝试使用传统的 bhyve 加载器加载它：

```sh
$ makefs -t zfs -o poolname=zroot zfs.img /tmp/test
$ sudo bhyveload -c stdio -d zfs.img test
```

这里，`bhyveload` 使用 `/boot/userboot.so`，它是一个经过编译以在用户空间运行的 FreeBSD 引导加载程序副本。它具有大部分真实引导加载程序的功能，但与直接使用 BIOS 调用或 EFI 启动服务从磁盘读取数据不同，它使用熟悉的 `read(2)` 系统调用从镜像文件 `zfs.img` 中获取数据。

最初的目标是让 `userboot.so` 能够找到并加载位于 `zfs.img` 中的内核文件 `/boot/kernel/kernel`。这是一个非常方便的测试平台，因为我可以轻松地将调试器附加到 `bhyveload`，或向加载程序添加打印语句并重新编译 `userboot.so`。我的第一个里程碑是让 [`vdev_probe()`](https://cgit.freebsd.org/src/tree/stand/libsa/zfs/zfsimpl.c?h=release/14.0.0#n2008) 识别该镜像为一个有效的 ZFS 池。

## vdev 标签和 uberblock

`vdev_probe()` 会查看磁盘是否属于 ZFS 池；即，它判断磁盘是否看起来是个 `vdev`，如果是，则开始加载更多元数据：

```c
/*
* 好的，我们目前对池的状态满意。接下来让我们找到
* 最佳的 uberblock，然后就可以实际访问
* 池中的内容了。
*/
vdev_uberblock_load(vdev, spa->spa_uberblock);
```

ZFS 磁盘规范的第 1 章详细说明了 `vdev` 标签和 uberblock。简而言之，`vdev` 包含一个元数据块，即 `vdev` 标签，标签中包含说明 `vdev` 所属池的元数据，以及多个“uberblock”副本，uberblock 指向 `vdev` 元数据树的根。因此，为了让 `userboot.so` 找到我的池，我编写了[代码](https://cgit.freebsd.org/src/tree/usr.sbin/makefs/zfs/vdev.c?h=release/14.0.0#n163)，将 `vdev` 标签添加到输出的镜像文件中。

此时，`makefs` 已经开始使用 ZFS 特定的数据结构，如 `vdev_label_t` 和 `uberblock_t`。为了避免重复引导加载程序中使用的定义，`makefs` 与其共享了一个[大型头文件](https://cgit.freebsd.org/src/tree/sys/cddl/boot/zfs/zfsimpl.h?h=release/14.0.0#n947)，其中包含许多有用的磁盘数据结构定义。

## 对象集与 MOS

当引导程序能够探测并识别由 `makefs` 生成的镜像后，下一步是让它能够从镜像中挂载数据集。处理这个任务的引导程序代码主要位于 [`zfs_get_root()`](https://cgit.freebsd.org/src/tree/stand/libsa/zfs/zfsimpl.c?h=release/14.0.0#n3334)。

要理解 `zfs_get_root()` 的实现，值得阅读 ZFS 磁盘布局规范的第三章，它概述了对象集。尽管规范很快就进入了详细的实现细节，但了解 ZFS 用来表示数据的高层结构还是很有价值的。

ZFS 有“块指针”，它们实际上只是指向 `vdev` 上某个数据块的物理位置（从 `makefs` 的角度看，这仅仅是输出镜像文件中的一个偏移量）。ZFS 元数据对象有几十种类型，它们由一个 512 字节的“dnode”表示。dnode 包含关于对象的各种元数据，例如对象的类型和大小，也可能包含指向额外数据的块指针。例如，存储在 ZFS 数据集中的文件由一个 dnode 表示（类型为 `DMU_OT_PLAIN_FILE_CONTENTS`），类似于传统 Unix 文件系统中的 inode。最后，“对象集”是一个包含多个 dnode 数组的结构；每个 dnode 都由它所属的对象集和在数组中的索引唯一标识。

ZAP（ZFS 属性处理器）是个包含一组键值对的 dnode。ZAP 用于表示许多高级 ZFS 元数据结构。例如，一个 Unix 目录由一个 ZAP 表示，其键是文件名，值是对应文件的 dnode ID。

MOS（元对象集）是池的根对象集。uberblock 包含指向 MOS 的指针，通过 MOS 可以访问池中的所有其他元数据（因此也能访问数据）。有了这些信息，就更容易理解 `zfs_get_root()`：它获取 ID 为 1 的 dnode（期望它是一个 ZAP 对象），利用它查找包含池属性的 ZAP 对象，并查找“bootfs”属性的值，这个值用于查找根数据集的 dnode。

在创建池时，`makefs` 会在 [`pool_init()`](https://cgit.freebsd.org/src/tree/usr.sbin/makefs/zfs.c?h=release/14.0.0#n544) 中分配并开始填充 MOS。只要 `userboot.so` 能够处理 MOS，就可以导入由 `makefs` 生成的池，此时我开始使用 `zdb(8)` 来检查生成的池。`zdb` 的命令行用法比较晦涩，但像下面这样的简单调用：

```sh
# zdb -dddd zroot 1
```

它将从 MOS 中转储 dnode 1，对于了解 OpenZFS 在导入池时预期看到的内容非常有用。例如，当转储一个 ZAP 对象时，`zdb` 会打印出 ZAP 中所有的键值对。许多配置 ZAP 的键值是 dnode ID，因此 `zdb` 可以轻松地用来检查池和数据集配置的不同“层”。

## 数据集与文件

ZFS 数据集有名称，并且组织成树形结构。根数据集的名称通常与池的名称相同（例如，“zroot”），子数据集的名称则以父数据集的名称为前缀。虽然我为 `makefs` 编写的 ZFS 支持的初始原型将所有文件自动放在根数据集中，但这不足以创建根文件系统位于 ZFS 上的虚拟机镜像：`bsdinstall` 和其他 FreeBSD 安装程序会自动创建多个子数据集。比如，`zroot/var` 这样的数据集不会被挂载，只是存在用来提供设置，供子数据集继承，如 `zroot/var/log`。我的目标是让 `makefs` 能够创建一个数据集树，符合 `bsdinstall` 提供的布局。

发布镜像构建脚本[演示](https://cgit.freebsd.org/src/tree/release/tools/vmimage.subr?h=release/14.0.0#n190)了如何创建多个数据集的语法。每个数据集由一个参数 `-o fs` 描述，该参数包含数据集名称和一个以分号分隔的属性列表。如 `zfsprops(8)` 手册页中所述，目前只支持少量属性。

当 `makefs -t zfs` 初始化各种结构后，它会开始[处理](https://cgit.freebsd.org/src/tree/usr.sbin/makefs/zfs/fs.c?h=release/14.0.0#n1048)输入的目录树。每个输入文件由一个 `fsnode` 结构表示，这些结构被组织成一个表示文件树的树形结构。首先，`makefs` 确定哪个 `fsnode` 对应于每个挂载的数据集的根。然后，它遍历 `fsnode` 树，为每个文件分配一个 dnode；这发生在数据集的上下文中，数据集决定了从哪个对象集中分配 dnode。

要[复制常规文件](https://cgit.freebsd.org/src/tree/usr.sbin/makefs/zfs/fs.c?h=release/14.0.0#n548)，`makefs` 从当前对象集中分配一个 dnode，并通过一个循环从输出文件中分配空间块，并将输入文件的数据复制到这些分配的块中。ZFS 支持从 4KB 到 128KB 的 2 的幂大小的块，因此较小的文件不会造成过多的内部碎片。所有在镜像中分配的块都通过位图进行追踪，位图由 [vdev_space_alloc()](https://cgit.freebsd.org/src/tree/usr.sbin/makefs/zfs/vdev.c?h=release/14.0.0#n235) 函数更新。

位图追踪的已分配空间必须记录在输出镜像中；ZFS 使用一个中央数据结构——“空间映射”来追踪 vdev 中当前已分配的区域。`makefs` 使用位图作为所有块分配的内部表示，并用它来生成空间映射，这是生成镜像的最后步骤之一，待所有块分配完成。

## 结论

为 `makefs` 添加 ZFS 支持花费了相当多的精力，但最终实现了一个我认为对许多 FreeBSD 用户有用的功能，同时避免了大量的维护负担。`makefs` 中约有 2600 行与 ZFS 相关的代码（总共有 15,000 行），这相对较少。还有一个[回归测试套件](https://cgit.freebsd.org/src/tree/usr.sbin/makefs/tests/makefs_zfs_tests.sh?h=release/14.0.0)，提供了相当充分的测试覆盖。

在这 2600 行中，有 100 多行是 `assert()` 调用，用于验证不变性。在开发过程中，这些断言非常有用，因为很多代码是以不完整的方式编写的，最初仅为了让引导加载程序工作，后来才逐步完善；这些断言帮助文档化了各种函数的限制，并帮助我在添加更多功能时捕捉了许多错误。

现在，FreeBSD 14.0 已经发布，并且根文件系统位于 ZFS 上的虚拟机镜像已经可以使用，我希望许多用户能够充分利用这一新功能。在发布周期中发现并修复了不少 bug，因此至少有些用户已经开始尝试。目前没有为 `makefs -t zfs` 添加新功能的打算，但如果有反馈，可能会有所改变——如果你发现有改进的空间，请提交 bug 报告。

---

**Mark Johnston** 是一名软件开发者和 FreeBSD 源代码开发者，居住在加拿大安大略省多伦多市。当不坐在电脑前时，他喜欢和朋友们一起参加城市躲避球联赛。
