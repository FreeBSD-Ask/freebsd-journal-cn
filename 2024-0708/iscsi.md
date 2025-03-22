# FreeBSD iSCSI 入门

- 原文链接：[FreeBSD iSCSI Primer](https://freebsdfoundation.org/our-work/journal/browser-based-edition/storage-and-filesystems/freebsd-iscsi-primer/)
- 作者：Jason Tubnor

我们经常听说网络附属存储（NAS）能够为网络上的设备提供额外的存储空间。然而，这种存储的协议可能并不适用于所有的使用场景。

欢迎来到存储区域网络（SAN）的世界。一般来说，这些存储系统较多地出现在企业环境中，而不是家庭和小型企业，但这并不意味着它们不能在这种情况下使用。实际上，如果你进行了大量虚拟化，且需要将中央存储接入到多个计算设备，或者需要为用于工程和图形设计的 Windows 工作站提供块存储，这些工作站的存储需求超过了桌面 PC 的物理存储空间，那么你可能会发现 SAN 极为有用。

一般来说，在企业领域我们听到的 SAN 供应商是戴尔 EMC、IBM、日立和 NetApp 等。然而，我们在 FreeBSD 环境中可以得到一款内置于基本系统的高性能 iSCSI SAN 解决方案，实在是让我们倍感幸运。通过与强大的 ZFS 卷管理器和文件系统相结合，我们能得到灵活、可靠且快速的存储解决方案，并将其提供给网络客户端。iSCSI 子系统实现于 FreeBSD 10.0-RELEASE，并在 10.1-RELEASE 中有了诸多性能改进。

iSCSI（Internet Small Computer Systems Interface，互联网小型计算机系统接口）是一种基于 IP 的协议，用于通过 TCP/IP 以太网网络传输 SCSI 命令。它能将块设备存储呈现给分布在网络上的计算机。它足够灵活，还可以通过互联网进行路由（如需要），但安全性的问题和需要考虑的预防措施超出了本文的讨论范围。

在理想情况下，iSCSI 应该存在于一个独立的二层物理网络中，在这个网络中，计算主机通过专用存储接口与存储目标进行通信。这个网络段上不应有其他普通的网络流量，以避免存储与其他计算节点之间的带宽争用。在较简单的环境中，使用带 VLAN 的分段交换机是一种可选的替代方案，但需要明白，普通网络流量将和存储流量争用接口带宽。

iSCSI 的高层术语相当简单。它有发起方（客户端）和目标方（主机）。发起方是连接的主动端，而目标方是被动端——它永远不会主动连接到发起方。

在接下来的操作中，我们将准备一个简单的 iSCSI 配置，在 FreeBSD 主机上使用它作为目标方，并为 FreeBSD 发起方和 MS Windows 发起方提供 ZFS `zvol` 块设备。

我们的网络主机如下：

```sh
2001:db8:1::a/64 – FreeBSD ZFS 存储主机（目标方） 
2001:db8:1::1/64 – FreeBSD 客户端（发起方）
2001:db8:1::2/64 – Windows Server 2022（发起方）
```

首先，我们将在存储主机上配置 ZFS 卷作为 iSCSI 目标方，提供给每个发起方。这也可以是简单的 ZFS 数据集和 UFS 分区上的文件，但 ZFS 卷提供了更多对存储方面的控制，尤其是在数据需求变化和与快照和复制要求相关的持续管理方面。

```sh
zfs create -o volmode=dev -V 50G tank/fblock0
zfs create -o volmode=dev -V 50G tank/wblock0
```

在创建卷时，可以根据工作负载的需求调整属性 `volblocksize`。自 14.1-RELEASE 以降，`volblocksize` 默认为 16K，这对大多数工作负载来说是一个合理的平衡。

接下来，创建一个仅允许 root 读/写的文件 `/etc/ctl.conf`。此文件包含密钥，因此必须确保其他用户和组没有读/写该文件的权限。下面是我们将添加的内容，用于提供发起方的存储位置：

```json
auth-group ag0 {
    chap-mutual “inituser1” “secretpassw0rd” “targetuser1” “topspassw0rd”
initiator-portal [2001:db8:1::1]
}

auth-group ag1 {
    chap-mutual “inituser2” “hiddenpassw0rd” “targetuser2” “freepassw0rd”
    initiator-portal [2001:db8:1::2]
}

portal-group pg0        {
    discovery-auth-group no-authentication
    listen [2001:db8:1::a]
}

target iqn.2012-06.org.example.iscsi:target1 {
    alias “Target for FreeBSD”
    auth-group ag0
    portal-group pg0
    lun 0 {
        path /dev/zvol/tank/fblock0
        #blocksize 4096
        option naa 0x4ee0ebaf06a1acee
        option pblocksize 4096
        option ublocksize 4096
    }
}

target iqn.2012-06.org.example.iscsi:target2 {
    alias “Target for Windows”
    auth-group ag1
    portal-group pg0
    lun 1 {
        path /dev/zvol/tank/wblock0
        blocksize 4096
        option naa 0x4ee0ebaf06a1acbb
    }
}
```

我们来分解一下这个文件，理解每个组件的作用及原因：

```json
auth-group ag0 {
    chap-mutual “inituser1” “secretpassw0rd” “targetuser1” “topspassw0rd”
initiator-portal [2001:db8:1::1]
}
```


这是一个授权组，可以跨多个目标使用。这个例子将 `auth-group` 绑定到一个具有地址 `2001:db8:1::1` 的唯一发起方，并要求进行相互身份验证。你可以简单地使用 CHAP 身份验证作为“单向”认证；但建议若支持，请使用相互身份验证，以确保发起方和目标方双方都能正确地进行相互认证。

这种身份验证可能足够用于发起方和目标方位于同一物理网络的情况，但不应将其作为唯一的安全控制手段。此类身份验证应被视为确保仅分配正确存储给发起方的方法。配置松散的 iSCSI 目标可能会使错误的存储可用，可能会造成数据、分区表或其他元数据被覆盖，因此，限制访问特定目标数据集至关重要。

```json
portal-group pg0        {
    discovery-auth-group no-authentication
    listen [2001:db8:1::a]
}
```

门户组设置了提供给发起方的目标环境。在这里，我们定义了一个门户组，允许发起方通过 `2001:db8:1::a` 连接到目标方，这样它们可以在无需先进行身份验证的情况下发现包括此门户组的目标数据集。通常在受控环境中，可以这样做，以确保发起方能够找到它们需要连接的目标，但在更加敌对和不受信任的环境中，这种做法则是不理想的。

```json
target iqn.2012-06.org.example.iscsi:target1 {
    alias “Target for FreeBSD”
    auth-group ag0
    portal-group pg0
    lun 0 {
        path /dev/zvol/tank/fblock0
        #blocksize 4096
        option naa 0x4ee0ebaf06a1acee
        option pblocksize 4096
        option ublocksize 4096
    }
}
```

配置的核心部分是目标。这将包括 `auth-group` 和 `portal-group`，用来构建之前介绍的各个组件，并呈现给发起方。它们可以在每个目标的基础上进行覆盖，并且可以在没有适用的 `group` 定义的情况下定义。

iSCSI 合格名称（IQN）的格式为 `iqn.yyyy-mm.namingauthority:uniquename`。每个目标方定义都需要这个。

别名定义只是一个用于目标的可读简介。

每个目标可以有多个 LUN，但这里每个目标只有一个 LUN。

LUN 上下文允许你定义 LUN 的特征。对于在发起方内使用 ZFS 存储的情况，这一点非常重要。尽管目标上的 ZFS 卷的块大小为 16KB，但当在发起方上创建 ZFS 池时，它会报错存储的块大小不是 4KB 及更小。它不会阻止你使用它，但当你在发起方执行 `zpool status` 时，它会不断提醒你此问题。调整块大小属性为 4096 并不足以解决此问题。需要专门为 ZFS 使用案例添加参数 `pblocksize` 和 `ublocksize` 。

参数 `naa` 应为 LUN 明确定义。这是一个 64 位或 128 位的唯一十六进制标识符。在将 VMware 计算备份到 iSCSI 目标时，确保没有混淆 LUN 分配，这是非常重要的。

现在，你已经具备了启动，展示目标存储的基本配置。接下来，启用 `ctld`，再启动守护进程：

```sh
service ctld enable
service ctld start
```

为了验证存储是否已出现，你可以检查守护进程是否在监听：

```json
# netstat -na | grep 3260
tcp6    0      0 2001:db8:1::a.3260     *.*     
```

然后，使用 CAM 目标层控制实用程序验证 LUN 是否已出现：

```json
# ctladm lunlist
(7:1:0/0):<FREEBSD CTLDISK 0001> Fixed Direct Access SPC-5 SCSI device
(7:1:1/1):<FREEBSD CTLDISK 0001> Fixed Direct Access SPC-5 SCSI device
# ctladm devlist
LUN Backend     Size (Blocks)   BS Serial Number     Device ID
  0 block           104857600   512 MYSERIAL0000   MYDEVID0000
  1 block            13107200  4096 MYSERIAL0001   MYDEVID0001
```

接下来，配置你的 FreeBSD 发起方。你需要创建配置文件 `/etc/iscsi.conf`。由于此文件包含机密信息，因此也需要显式设置为仅 root 可读写：

```json
fblock0         {
targetaddress   = [2001:db8:1::a];
targetname      = iqn.2012-06.org.example.iscsi:target1;
initiatorname   = iqn.2012-06.org.example.freebsd:nobody;
authmethod      = CHAP;
chapiname       = “inituser1”;
chapsecret      = “secretpassw0rd”;
tgtChapName     = “targetuser1”;
tgtChapSecret   = “topspassw0rd”;
}
```

每个属性的含义如下：

- `fblock0`：这是一个人类可读的标识符，与东西无涉，仅用于将以下配置项分组。
- `targetaddress`：存储目标的网络地址。也可以是一个完全合格的域名。
- `targetname`：这将与在 `ctl.conf` 文件中定义的相应目标名称对齐。
- `initiatorname`：定义发起方的 IQN。
- `authmethod`：在 FreeBSD 中可以简单地定义为 CHAP。如果 `ChapName` 和 `ChapSecret` 以 `tgt` 为前缀，则会假定使用相互身份验证。
- `chapiname/chapsecret`：如前所述，在 `ctl.conf` 文件中定义的身份验证。
- `tgtChap[Name,Secret]`：目标需要完成身份验证握手的身份验证。

要启用使用，只需执行以下命令：

```sh
service iscsid enable
service iscsictl enable
service iscsid start
service iscsictl start
```

这将使目标存储呈现连接到发起方：

```sh
# iscsictl -L
Target name                           Target portal    State
iqn.2012-06.org.example.iscsi:target1 [2001:db8:1::a]  Connected: da0
```

在此，我们来看一下 `iscsictl` 命令中最常用的简单参数：

- `L` 列出已挂载到发起方的目标及其连接状态。
- `Aa` 附加在 `iscsi.conf` 文件中定义的所有目标。
- `Ra` 移除所有已连接到发起方的目标。

接下来，执行以下操作：

```sh
# gpart create -s GPT da0
da0 created
# gpart add -t freebsd-zfs -a 1M da0
da0p1 added
# zpool create tank da0p1
# zpool list tank
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  49.5G   360K  49.5G        -         -     0%     0%  1.00x    ONLINE        -
# zpool status tank
  pool: tank
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        tank        ONLINE       0     0     0
          da0p1     ONLINE       0     0     0

errors: No known data errors
```

配置、守护进程和控制工具的手册页写得非常详细，可以参考这些手册来更好地了解可用的其他功能。

这仅触及了 FreeBSD 中 iSCSI 实现的全部功能的一部分，但它为你提供了一个概念和实际示例，展示了如何利用它为你的计算基础设施提供灵活的远程存储选择。

---

**Jason Tubnor** 拥有逾 28 年的 IT 行业经验，广泛涉猎，目前是 Latrobe Community Health Service（澳大利亚维多利亚州）的 ICT 高级安全主管。在 1990 年代中期，Jason 接触到 Linux 和开源技术，2000 年开始使用 OpenBSD，他利用这些工具在各个行业的组织中解决了各种问题。Jason 还是 BSDNow 播客的联合主持人。

