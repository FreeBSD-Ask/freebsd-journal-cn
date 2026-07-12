# pNFS

作者：**Rick Macklem**

为 FreeBSD 创建 pNFS 服务的第一次尝试使用了 GlusterFS 以及通过 fuse 与之通信的内核 nfsd。它可以工作，但性能极差，上下文切换数量巨大。因此，我重新开始，设计一个使用 nfsd 和一些 FreeBSD 系统的内核内服务，但没有集群文件系统。我将其称为 Plan B，本文讨论的就是它。

## 总体目标

pNFS 服务将读/写操作与所有其他 NFSv4.1 操作分离。希望是这种分离允许配置的 pNFS 服务超过单个 NFS 服务器的存储容量和/或 I/O 带宽限制。对于 pNFS 服务，NFS 服务器成为元数据服务器（MDS），处理除 I/O 操作外的所有操作。还有其他服务器配置为数据服务器（DSs），只处理 I/O 操作。

可以在 DSs 中配置镜像，这样 MDS 文件的数据文件将镜像在两个或更多 DSs 上。使用此功能时，DS 故障不会停止 pNFS 服务，并且故障的 DS 可以在修复后恢复，同时 pNFS 服务继续运行。虽然双向镜像是常态，但可以设置最多四个或 DSs 数量（取较小者）的镜像级别。镜像级别指的是在不同 DSs 上保留多少份数据文件。FreeBSD MDS 仍然是单点故障，就像普通 NFS 服务器一样。

## Plan B 概览

Plan B pNFS 服务由单个 MDS 和 K 个 DSs 组成，全部是 FreeBSD 12 系统。客户端将像挂载普通 NFS 服务器一样挂载 MDS。创建文件时，MDS 创建一个与普通 NFS 服务器创建的相同的文件树，不同之处在于所有常规（VREG）文件都为空。因此，如果直接在 MDS 服务器上（而不是通过 NFS 挂载）查看 MDS 上导出的树，所有文件大小都为 0。这些文件中的每一个还将具有系统属性名称空间中的两个扩展属性：

pnfsd.dsfile——此扩展属性存储 MDS 在 DS(s) 上查找此文件的数据文件所需的信息。

pnfsd.dsattr——此扩展属性存储文件的 Size、AccessTime、ModifyTime 和 Change 属性，使 MDS 不必为每次 Getattr 操作从 DS 获取属性。

对于每个常规（VREG）文件，MDS 在一个或多个（如果启用了镜像）DSs 的“dsN”子目录之一中创建数据文件。此文件的名称是 MDS 上文件的文件句柄的十六进制表示，因此名称是唯一的。在 MDS 上创建文件时，以轮询方式选择 DS(s)，这对于存储小文件似乎足够。对于存储小文件和大文件混合的服务，可能需要不同的算法。

我考虑过实现“最多可用空间”算法，但由于检查每个 DS 的可用空间的开销而没有这样做。DSs 使用名为“ds0”到“dsN”的子目录，以便没有一个目录变得太大。N 的值通过 MDS 上的 sysctl `vfs.nfsd.dsdirsize` 设置，默认值为 20。

对于将存储大量文件的生产服务器，此值可能应该大得多。可以在 MDS 上的 nfsd 守护进程未运行时增加它，前提是在 DSs 上创建额外的“dsN”子目录。

对于支持 pNFS 的 NFSv4.1 客户端，FreeBSD 服务器将向客户端返回两条信息，允许它直接对 DS 执行 I/O。

• DeviceInfo——这是相对静态的信息，定义 DS 是什么。FreeBSD 服务器返回的关键信息是 DS 的 IP 地址，对于 Flexible File 布局，它是“紧密耦合”的。有一个“deviceid”用于标识 DeviceInfo，布局通过它引用 DeviceInfo。支持 pNFS 的客户端通过 NFSv4.1 GetDeviceInfo 操作获取此信息。

• Layout——这是针对每个文件的，可以在不再有效时由服务器收回。对于 FreeBSD 服务器，支持两种类型的布局，分别称为 File 和 Flexible File 布局。两者都允许客户端通过 NFSv4.1 I/O 操作在 DS 上执行 I/O。Flexible File 布局是较新的变体，允许指定镜像，客户端应向所有镜像执行写操作以保持它们处于一致状态。Flexible File 布局支持两种变体，分别称为“紧密耦合”和“松散耦合”。FreeBSD 服务器始终使用“紧密耦合”变体，客户端使用与在 MDS 上相同的凭据在 DS 上执行 I/O。对于“松散耦合”变体，布局指定一个合成用户/组，客户端使用它在 DS 上执行 I/O。FreeBSD 服务器不进行条带化，始终返回整个文件的布局。布局中的关键信息是 Read 与 Read/Write，标识数据文件存储在哪些 DS(s) 上的 deviceid(s)，以及数据文件的文件句柄。支持 pNFS 的客户端通过 NFSv4.1 LayoutGet 操作获取此信息。客户端还可以执行 LayoutReturn 操作返回布局，无论是在使用完毕时，还是在 NFSv4.1 服务器通过 CBLayoutRecall 回调请求返回时。对于 Flexible File 布局，客户端可以在 LayoutReturn 参数中向 MDS 报告在 DS 上执行 I/O 时发生的 I/O 错误。

MDS 向知道如何为非镜像 DS 情况执行 pNFS 的 NFSv4.1 客户端生成 File 布局，除非 sysctl `vfs.nfsd.default_flexfile` 设置为非零，在这种情况下生成 Flexible File 布局。

镜像 DS 配置始终生成 Flexible File 布局。对于不支持 NFSv4.1 pNFS 的 NFS 客户端，所有 I/O 操作都发送到 MDS。当 MDS 接收 I/O RPC 时，它将作为代理在 DS 上执行 RPC。

如果 DS 与 MDS 在同一台机器上，MDS/DS 将作为代理在 DS 上执行 RPC，依此类推，直到机器耗尽某些资源（如会话槽或 mbufs）。因此，DS 不能与 MDS 在同一系统上。

虽然我不认为这是一个实际的生产设置，但对于测试，你可以为所有 DSs 使用单个系统，该系统也可以用作客户端，允许仅使用两台系统进行测试。对于测试，可以在 DS 系统上创建多个 DS，但必须为此 DS 系统上的每个附加 DS 分配一个别名 IP 地址，并使用这些单独的别名地址挂载附加的 DSs。换句话说，每个 IP 地址只允许一个 DS 挂载。挂载还必须为每个 DS 使用单独的导出目录来存储数据文件。

pNFS 服务位于 FreeBSD-current 中，将在 FreeBSD 12 发布时包含在其中。在 FreeBSD 12 发布之前，可以使用 FreeBSD-current 快照分发进行测试。

## 设置使用 Plan B 的 FreeBSD pNFS 服务器

让我们假设有五台 FreeBSD 12 系统，其中四台配置为 DSs，使用双向镜像和 AUTH_SYS 进行示例。

• MDS，向客户端导出 **/export**。
  nfsv4-server

• DSs，将 **/DSstore** 导出到 MDS 和客户端用于存储数据文件。
  nfsv4-data0、nfsv4-data1、nfsv4-data2 和 nfsv4-data3

在 nfsv4-server 上，你需要为客户端导出文件系统树。**/etc/exports** 中的两行可能如下所示：

```ini
       V4: /export -sec=sys -network 192.168.1.0 -mask 255.255.255.0
       /export -sec=sys -network 192.168.1.0 -mask 255.255.255.0
```

然后你需要在 **/etc/rc.conf** 中添加以下行：

```ini
       rpcbind_enable="YES"
       mountd_enable="YES"
       nfs_server_enable="YES"
       nfsv4_server_enable="YES"
       nfs_server_flags="-u -t -n 32 -p nfsv4-data0,nfsv4-data1,nfsv4-data2,
       nfsv4-data3 -m 2"
```

除非你想运行 nfsuserd 在 uid/gid 号和名称之间映射，否则将以下行放在 **/etc/sysctl.conf** 中：

```ini
       vfs.nfs.enable_uidtostring=1
       vfs.nfsd.enable_stringtouid=1
```

这将 NFSv4.1 服务器配置为在 owner 和 owner_group 字符串中使用 uid/gid 号。然后 DSs 必须挂载在 MDS 上。**/etc/fstab** 行如下：

```ini
   nfsv4-data0:/  /data0   nfsrw,nfsv4,minorversion=1,soft,retrans=2  0      0
   nfsv4-data1:/  /data1   nfsrw,nfsv4,minorversion=1,soft,retrans=2  0      0
   nfsv4-data2:/  /data2   nfsrw,nfsv4,minorversion=1,soft,retrans=2  0      0
   nfsv4-data3:/  /data3   nfsrw,nfsv4,minorversion=1,soft,retrans=2  0      0
```

并且 **/data0**、**/data1**、**/data2** 和 **/data3** 目录需要在 MDS 上创建用于挂载 DS 数据文件。注意，`soft,retrans=2` 通常不用于 NFSv4 挂载，但这是一个例外，因为 DS 上不执行状态操作。这样做允许对 DS 的代理操作失败，并在发生时禁用失败的 DS。

在 DSs 上，你需要 mkdir 并将 **/DSstore** 目录导出到 MDS 和客户端。**/etc/exports** 行可能如下所示：

```ini
   V4: /DSstore -sec=sys -network 192.168.1.0 -mask 255.255.255.0
   /DSstore -sec=sys -maproot=root nfsv4-server
   /DSstore -sec=sys -network 192.168.1.0 -mask 255.255.255.0
```

你需要在 **/etc/rc.conf** 中添加以下行：

```ini
   rpcbind_enable="YES"
   mountd_enable="YES"
   nfs_server_enable="YES"
   nfsv4_server_enable="YES"
   nfs_server_flags="-u -t -n 32"
```

并在 **/etc/sysctl.conf** 中：

```ini
   vfs.nfs.enable_uidtostring=1
   vfs.nfsd.enable_stringtouid=1
```

你需要在 **/DSstore** 下创建“dsN”目录。在 DSs 上的 **/DSstore** 中执行的以下命令可以做到：（从这里开始的所有命令都需要 root/su 执行。）

```sh
     # jot -w ds 20 0 | xargs mkdir -m 700
```

设置完这些系统后，MDS 应该准备好接受客户端挂载。FreeBSD 客户端需要在它们的 **/etc/rc.conf** 文件中添加以下内容：

```ini
rpcbind_enable="YES"
nfs_client_enable="YES"
nfscbd_enable="YES"
```

然后，在 FreeBSD 客户端上，挂载命令可能如下：

```sh
# mount -t nfs -o nfsv4,minorversion=1,pnfs nfsv4-server:/ /mnt
```

然后客户端可以像使用普通 NFS 挂载一样使用 **/mnt**。如果在 nfsv4-server 上执行 `nfsstat -E -s`，你不应该看到很多读或写操作。大多数读和写操作应该出现在 DSs 上执行的 `nfsstat -E -s` 中。在 MDS 上有少量读和写操作是正常的，因为这是客户端因任何原因未能获取有效布局时回退的方式。

如果改用 NFSv3 挂载，命令可能是：

```sh
# mount -t nfs nfsv4-server:/ /mnt
```

现在，如果在 nfsv4-server 上执行 `nfsstat -E -s`，你会看到很多读和写操作，因为它们都通过 MDS 完成，MDS 充当 DSs 的代理。

假设你在 **/mnt** 上创建一个名为“abc.c”的文件，大小为 274 字节。在 **/mnt** 中执行 `ls -l` 会显示如下行：

```sh
-rw-r--r--   1 ricktst  wheel  274 Jun  5 18:02 abc.c
```

而如果你转到 nfsv4-server 上的 **/export**，`ls -l` 行会如下所示：

```sh
-rw-r--r--   1 ricktst  wheel   0 Jun  5 18:02 abc.c
```

然后，在同一目录中执行 `lsextattr system abc.c` 会显示：

```sh
abc.c pnfsd.dsfile     pnfsd.dsattr
```

执行 `pnfsdsfile abc.c` 会显示：

```sh
abc.c: nfsv4-data2
ds5/207508569ff983350c000000a9730200eec58e800000000000000000
nfsv4-data3
ds5/207508569ff983350c000000a9730200eec58e800000000000000000
```

（pnfsdsfile 命令显示“pnfsd.dsfile”中的内容，除非指定了命令行参数。）

这告诉你“abc.c”的数据存储在 nfsv4-data2 和 nfsv4-data3 的子目录“ds5”中，文件名为“2075...”。

如果我们转到 nfsv4-data2 上的 **/DSstore/ds5**，`ls -l *a97302*` 会显示：

```sh
-rw-r--r--  1 ricktst  wheel  274 Jun  5 18:02
207508569ff983350c000000a9730200eec58e800000000000000000
```

在 nfsv4-data3 上：

```sh
-rw-r--r--  1 ricktst  wheel  274 Jun  5 18:02
207508569ff983350c000000a9730200eec58e800000000000000000
```

注意，所有权和权限与 MDS 上的“abc.c”相同。这是因为它是“紧密耦合”的 Flexible File 布局服务，在 DS 上强制执行权限检查。对于 Flexible File 布局，客户端可以并发写入 nfsv4-data2 和 nfsv4-data3，因此不应因镜像而显著影响性能。但是，使用的存储空间将是非镜像配置的两倍。“atime”通常不会在 DSs 之间保持一致，但如果 sysctl `vfs.nfsd.pnfsstrictatime` 设置为 1，则会保持一致。设置此项确实会导致大量开销。

好的，让我们玩玩。当以下三种情况之一发生时，MDS 将禁用镜像 DS：

1. 元数据服务器（MDS）在尝试在 DS 上执行代理操作时检测到问题。这就是为什么 DS 服务器在 MDS 上以选项 `soft,retrans=2` 挂载。这可能需要几分钟才能在 DS 故障或网络分区发生后触发。

2. pNFS 客户端可以在 LayoutReturn 操作的参数中向 MDS 报告关于 DS 的 I/O 错误。

3. 系统管理员可以在 MDS 上执行 **pnfsdskill(1)** 命令禁用 DS。如果系统管理员执行 **pnfsdskill(1)** 并以 ENXIO（设备未配置）失败，通常意味着 DS 已通过 #1 或 #2 禁用。由于这样做无害，一旦系统管理员知道镜像 DS 有问题，建议执行该命令。

让我们做 #3，因为它很简单：

```sh
     # pnfsdskill /data2
```

这将在控制台上记录一条消息：

```sh
    pNFS server: mirror nfsv4-data2 failed
```

nfsv4-data2 现已被标记为禁用，并为所有使用 nfsv4-data2 的布局执行了 CBLayoutRecall 回调。然后我们可以卸载 nfsv4-data2 的 **/DSstore**：

```sh
    # umount -N /data2
```

（选项 `-N` 确保即使线程卡住尝试在失败的 nfsv4-data2 上执行 RPC，umount 也能工作。）

现在，让我们将一些数据写入 **/mnt/abc.c**，所以客户端上的 `ls -l` 行如下所示：

```sh
    -rw-r--r--  1 ricktst  wheel  586 Jun  5 19:11 abc.c
```

如果我们转到 nfsv4-data2 上的 **/DSstore/ds5**，`ls -l *a97302*` 会显示：

```sh
    -rw-r--r--  1 ricktst  wheel  274 Jun  5 18:02
    207508569ff983350c000000a9730200eec58e800000000000000000
```

而在 nfsv4-data3 上：

```sh
    -rw-r--r--  1 ricktst  wheel  586 Jun  5 19:11
    207508569ff983350c000000a9730200eec58e800000000000000000
```

注意，nfsv4-data2 没有被写入，因为它被禁用。文件“abc.c”仍然可以读/写，但不再有冗余副本。

修复 nfsv4-data2 的第一步是确保 nfsv4-data2 上的文件的过期副本在 nfsv4-data2 重新上线时不被使用。为此，我们转到 MDS 上的 **/export** 目录，并通过以下命令将 nfsv4-data2 替换为 IP 地址 0.0.0.0：

```sh
    # pnfsdsfile -r nfsv4-data2 abc.c
    abc.c:    0.0.0.0
    ds5/207508569ff983350c000000a9730200eec58e800000000000000000
    nfsv4-data3.home.rick
    ds5/207508569ff983350c000000a9730200eec58e800000000000000000
```

现在，它不会尝试使用 nfsv4-data2。这必须对 **/export** 中指定 nfsv4-data2 作为 DS 的所有文件执行，所以使用 **find(1)** 的命令是：

```sh
    # find . -type f -exec pnfsdsfile -q -r nfsv4-data2 {} \;
```

注意，许多文件不会指定 nfsv4-data2 作为 DS，但该命令知道对这些文件只 `exit(0)`，因此可以安全地用于任何文件。不幸的是，`rename` 或 `link/unlink` 的某些组合可能导致 **find(1)** 遗漏某些文件。要检查遗漏的文件，命令是：

```sh
# find . -type f -exec pnfsdsfile {} \; | sed "/nfsv4-data2/!d" | sed "s/:.*//"
```

（使用 `sh` shell 搜索仍指定 nfsv4-data2 的任何文件。）

上述命令输出任何文件名都需要执行 `pnfsdsfile -r` 命令。现在，我们可以在修复 nfsv4-data2 后安全地将其重新上线。对于本练习，我们只需进入 nfsv4-data2 上的 **/DSstore** 并清理它：

```sh
# cd /DSstore
# rm -rf *
# jot -w ds 20 0 | xargs mkdir -m 700
```

现在，在 MDS 上，我们可以通过挂载它并重启 nfsd 守护进程来使其重新上线：

```sh
# mount -t nfs -o nfsv4,minorversion=1,soft,retrans=2 nfsv4-data2:/ /data2
# /etc/rc.d/nfsd restart
```

执行上述操作后，新创建的文件可能分配给 nfsv4-data2，但之前在 nfsv4-data2 上镜像的文件仍需恢复。为此，我们转到 MDS 上的 **/export** 并执行命令：

```sh
# pnfsdscopymr -r /data2 abc.c
```

这会将 abc.c 的数据复制到 nfsv4-data2。

执行此命令后，**pnfsdsfile(1)** 显示：

```sh
# pnfsdsfile abc.c
abc.c:     nfsv4-data2.home.rick
ds5/207508569ff983350c000000a9730200eec58e800000000000000000
nfsv4-data3.home.rick
ds5/207508569ff983350c000000a9730200eec58e800000000000000000
```

如果我们转到 nfsv4-data2 上的 **/DSstore/ds5**，`ls -l *a97302*` 会显示：

```sh
-rw-r--r--  1 ricktst  wheel  586 Jun  5 19:11
207508569ff983350c000000a9730200eec58e800000000000000000
```

在 nfsv4-data3 上：

```sh
-rw-r--r--  1 ricktst  wheel  586 Jun  5 19:11
207508569ff983350c000000a9730200eec58e800000000000000000
```

在内核中实现此复制的代码有些复杂。算法的简要描述如下：

• MDS 文件的 vnode 被锁定，阻止 LayoutGet 操作。
• 通过 nfsdontlist 禁用为文件发出读/写布局，以便在 MDS 文件的 vnode 解锁后它们将被禁用。
• 设置 nfsrv_recalllist 以便可以召回读/写布局。
• 解锁 MDS 文件的 vnode，以便客户端可以在完成 LayoutRecall 回调请求的 LayoutReturn 时，为文件执行代理写入、LayoutCommit 和 LayoutReturns。
• 为所有读/写布局发出 CBLayoutRecall 回调并等待它们返回。（如果 CBLayoutRecall 回调回复 NFSERR_NOMATCHLAYOUT，它们已经消失，无需 LayoutReturn。）
• 独占锁定 MDS 文件的 vnode。这确保在 DS 文件复制期间没有代理写入正在进行或可以发生。它还阻止 Setattr 操作。
• 在修复的镜像上创建文件。
• 从运行中的 DS 复制文件。
• 将 MDS 文件中的任何 ACL 复制到新的 DS 文件。
• 将新的 DS 文件的修改时间设置为 MDS 文件的时间。
• 更新 MDS 文件的扩展属性。
• 通过删除 nfsdontlist 条目启用发出读/写布局。
• 解锁 MDS 文件的 vnode，允许操作正常继续，因为它再次被镜像。

同样，这必须对所有文件执行，因此 **find(1)** 命令是：

```sh
   # find . -type f -exec pnfsdscopymr -r /data2 {} \;
```

要检查 **find(1)** 遗漏的文件：

```sh
   # find . -type f -exec pnfsdsfile {} \; | sed "/0\.0\.0\.0/!d" | sed "/:.*//"
```

（使用 `sh`，搜索仍具有 DS 地址 0.0.0.0 的任何文件。）

如果这打印出文件，则需要对它们执行 `pnfsdscopymr -r` 命令。如果没有打印出任何内容，nfsv4-data2 已恢复，所有文件现在应该正确镜像。系统管理员也可以使用 **pnfsdscopymr(1)** 命令将数据文件从一个 DS 迁移到另一个 DS。要将 abc.c 的数据文件从 nfsv4-data3 移动到 nfsv4-data0，命令是：

```sh
   # pnfsdscopymr -m /data3 /data0 abc.c
```

执行此命令后，pnfsdsfile 显示：

```sh
   # pnfsdsfile abc.c
   abc.c:     nfsv4-data2.home.rick
   ds5/207508569ff983350c000000a9730200eec58e800000000000000000
   nfsv4-data0.home.rick
   ds5/207508569ff983350c000000a9730200eec58e800000000000000000
```

如果我们转到 nfsv4-data2 上的 **/DSstore/ds5**，`ls -l *a97302*` 会显示：

```sh
   -rw-r--r--  1 ricktst  wheel  586 Jun  5 19:11
   207508569ff983350c000000a9730200eec58e800000000000000000
```

在 nfsv4-data3 上：

```sh
   ls: No match.
```

在 nfsv4-data0 上：

```sh
   -rw-r--r--  1 ricktst  wheel  586 Jun  5 19:11
   207508569ff983350c000000a9730200eec58e800000000000000000
```

这可能对将大文件的数据移动到有更多可用空间的 DS 有用。`pnfsdscopymr -m` 命令只执行一些完整性检查，然后一次系统调用完成工作。此系统调用将来可用于存储/负载均衡器实现。

Linux 客户端目前有一个支持“松散耦合”变体但无法正确处理“紧密耦合”变体的 Flexible File 布局驱动程序。它始终使用合成用户/组在 DS 上执行 I/O 操作。

有两种方法可以解决此问题：

1. 修补客户端驱动程序以修复此问题。我有一个补丁：<http://people.freebsd.org/~rmacklem/flexfile.patch>，似乎可以正常工作。

2. 在 DSs 上以 `-maproot=root` 导出 **/DSstore**，然后在 MDS 上设置 `vfs.nfsd.flexlinuxhack=1`。设置此 sysctl 使 pNFS 服务器在布局中发送 (0, 0) 的合成用户/组，使 Linux 客户端始终以 `root` 身份在 DSs 上执行 I/O。

你还需要较新的 Linux 内核。我一直在使用 Linux-4.17-rc2 进行测试。Linux-4.12 可以工作，但测试期间会间歇性崩溃，更早的内核不支持 NFSv4.1 DSs 的 Flexible File 布局。这对于非镜像 pNFS 服务器不是问题，因为它会向 Linux 客户端发送 File 布局。

解决此问题后，Linux 上的挂载命令是：

```sh
  # mount -t nfs -o nfsvers=4,minorversion=1 nfs4-server:/ /mnt
```

有关 pNFS 服务设置和管理的更多信息，文档 <http://people.freebsd.org/~rmacklem/pnfs-planb-setup.txt> 可能有帮助。还有以下手册页：**pnfsdskill(1)**、**pnfsdsfile(1)**、**pnfsdscopymr(1)** 和 **pnfs(4)**。

如果你想查看协议详细信息，RFC 如下：

RFC-5661: Network File System (NFS) Version 4 Minor Version 1 Protocol, ISSN: 2070-1721.

flexible file 布局目前是互联网草案，但应很快作为 RFC 发布。希望本文发表之前。如果没有，这里是草案：

Parallel NFS (pNFS) Flexible File Layout draft-ietf-nfsv4-flex-files-19.txt.

---

**Rick Macklem** 与 BSD 打交道已太久。他 1985 年首次对 BSD 的贡献是将 4.2BSD 移植到 MicroVAXII。之后不久，他贡献了成为 4.3BSD Reno 一部分的第一个 NFS 实现。他在一所加拿大大学担任系统管理员三十年，现已退休，仍在为 FreeBSD 开发 NFS。是的，这篇文章是用 `ed` 撰写的，`ed` 仍是他选择的编辑器。
