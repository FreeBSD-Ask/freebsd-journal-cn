# FreeBSD 中的 NVMe-oF

- 原文链接：[NVMe Over Fabrics in FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/storage-and-filesystems/nvme-over-fabrics-in-freebsd-2/)
- 作者：John Baldwin

**NVM Express (NVMe) 是一种新兴的非易失性存储设备（如固态硬盘）访问标准。**

NVMe 最初是为了通过 PCIe 访问非易失性内存设备而定义的。它包括用于 PCIe 控制器设备的寄存器定义、存储在主内存中的命令提交和完成队列的布局与结构，以及一组命令。

NVMe 基本规范定义了管理员命令集，用于与每个控制器关联的单个管理员提交和完成队列对进行操作。管理员命令不处理 I/O 请求，而是用于创建 I/O 队列、获取辅助数据（如错误日志）、格式化存储设备等。NVMe 中的存储设备称为命名空间，特定命名空间的命令包括命名空间 ID。基本规范还定义了 NVM 命令集，用于对面向块的命名空间执行 I/O 请求。该规范设计为可扩展，支持后续的扩展，包括额外的 I/O 命令集（例如，针对键值存储的 I/O 命令集）。NVMe 控制器及其附加的命名空间一起被称为 NVM 子系统。

**NVMe-oF（NVMe over Fabrics）** 扩展了原始规范，使其能够通过网络传输（而非 PCIe）访问 NVM 子系统，这类似于使用 iSCSI 访问远程块存储设备作为 SCSI LUN。NVMe-oF 支持多种传输层，包括 FibreChannel、RDMA（通过 iWARP 和 ROCE）以及 TCP。为了处理这些不同的传输，Fabrics 包括了对基本 NVMe 规范的传输无关扩展，以及传输特定的绑定。

Fabrics 定义了一种新的 capsule 抽象，用于支持 NVMe 命令和 completion。每个 capsule 包含一个 NVMe 命令或 completion。此外，capsule 可能与数据缓冲区相关联。为了支持数据传输，NVMe 命令中现有的 PRP 条目被单个 NVMe SGL 条目替代。Fabrics 还将用于 PCIe 控制器的共享内存队列替换为逻辑完成和提交队列。与 PCIe I/O 队列不同，Fabrics 队列总是显式地与每个提交队列配对，后者与一个专用的完成队列相连。capsule 和数据缓冲区如何在队列对上传输和接收是传输特定的，但从抽象的角度来看，命令 capsule 在提交队列上传输，completion 则在完成队列上传输。

Fabrics 主机创建一个管理员队列对和一个/多个 I/O 队列对，连接到控制器。完整的队列对集称为关联。单个关联可以包含多个传输特定的连接。例如，TCP 传输为每个队列对使用专用连接，因此一个活动的 TCP 关联至少需要两个 TCP 连接。

除了提供对命名空间访问的 I/O 控制器外，Fabrics 还添加了一种发现控制器类型。发现控制器有了一个新的发现日志页面，用于 Fabrics NVM 子系统中可用的控制器集。日志页面可能包含一个或多个 I/O 控制器、指向其他子系统中发现控制器的引用。如果控制器可以通过多个传输访问，则日志页面可能包含该控制器的多个条目。每个日志页面条目包含控制器类型（I/O 或发现）以及传输类型和传输特定地址。对于 TCP 传输，地址包括 IP 地址和 TCP 端口号。

通过 NVMe 合格名称（NQN）标识 Fabrics 主机和控制器。NQN 是个 ASCII 字符串，应以“nqn.YYYY-MM.reverse-domain”开头，后跟可选后缀。名称的 reverse-domain 部分应该是一个有效的 DNS 名称的逆序，YYYY 和 MM 字段应该指定为：由使用该前缀的组织，持有 DNS 域名的有效年份和月份。规范定义了一个固定的子系统 NQN 用于发现控制器，并且定义了从 UUID 构建 NQN 的方案。在建立关联时，必须指定主机和子系统（控制器）的 NQN。

FreeBSD 15 支持通过主机内核驱动程序访问远程命名空间，以及支持将本地存储设备导出为命名空间到远程主机。Fabrics 的内核实现包括一个传输抽象层（由 `nvmf_transport.ko` 提供），用于隐藏大部分传输特定的细节，以便主机和控制器模块使用。这个模块会根据需要自动加载。独立的内核模块提供对各个传输的支持。这些模块必须显式加载才能启用传输。目前，FreeBSD 能通过 `nvmf_tcp.ko` 提供 TCP 传输的支持。可以在 [nvmf_tcp(4)](https://man.freebsd.org/nvmf_tcp/4) 中查看 TCP 相关的细节。

## 主机

FreeBSD 中的 Fabrics 主机内置了新的 [nvmecontrol(8)](https://man.freebsd.org/nvmecontrol/8) 命令和内核驱动程序 [nvmf(4)](https://man.freebsd.org/nvmf/4) 。内核驱动程序将远程控制器暴露为类似于 PCIe NVMe 控制器的新总线设备 nvmeX。远程命名空间通过磁盘设备 [nda(4)](https://man.freebsd.org/nda/4) 以 CAM 暴露。与 PCIe 的驱动程序 [nvme(4)](https://man.freebsd.org/nvme/4)不同，Fabrics 主机驱动程序不支持磁盘驱动程序 [nvd(4)](https://man.freebsd.org/nvd/4)。所有新的 nvmecontrol(8) 命令都使用从主机 UUID 生成的主机 NQN，除非显式指定主机 NQN。


### 发现服务

`nvmecontrol(8)` 的命令 `discover` 查询来自发现控制器的发现日志页面，并显示其内容。示例 1 显示了来自在 Linux 系统上运行的 Fabrics 控制器的日志页面。对于 TCP 传输，由服务标识符字段标识远程控制器的 TCP 端口。

**示例 1：来自 Linux 控制器的发现日志页面**

```sh
# nvmecontrol discover ubuntu:4420
Discovery
=========
Entry 01
========
Transport type: TCP
 Address family:       AF_INET
Subsystem type:        NVMe
 SQ flow control:      optional
 Secure Channel:       Not specified
 Port ID:              1
 Controller ID:        Dynamic
 Max Admin SQ Size:    32
 Sub NQN:              nvme-test-target
 Transport address:    10.0.0.118
 Service identifier:   4420
 Security Type:        None
```

### 连接到 I/O 控制器

`nvmecontrol(8)` 的 `connect` 命令与远程控制器建立关联。建立关联后，关联会传递给驱动程序 `nvmf(4)`，后者会创建一个新的设备 `nvmeX`。`connect` 命令需要远程控制器的网络地址和子系统 NQN。**示例 2** 连接到 **示例 1** 中列出的 I/O 控制器。

**示例 2：连接到 I/O 控制器**

```sh
# kldload nvmf nvmf_tcp
# nvmecontrol connect ubuntu:4420 nvme-test-target
```

建立关联后，内核会将图 1 中的文本输出到系统控制台和系统信息缓冲区。设备 `nvmeX` 在设备描述中包括远程子系统 NQN，每个远程命名空间都列举为外设 `ndaX`。

**图 1：连接时的控制台信息**

```sh
nvme0: <Fabrics: nvme-test-target>
nda0:  at nvme0 bus 0 scbus0 target 0 lun 1
nda0: <Linux 5.15.0-8 843bf4f791f9cdb03d8b>
nda0:   Serial Number 843bf4f791f9cdb03d8b
nda0:  nvme version 1.3
nda0:  1024MB (2097152 512 byte sectors)
```

来自图 1 的 `nvme0` 设备能与其他 `nvmecontrol(8)` 命令一道使用，如 `identify`，类似于 PCIe 控制器。**示例 3** 显示了 `nvmecontrol(8)` 显示的部分控制器标识数据。能像任何其他 NVMe 磁盘设备一样使用磁盘设备 `nda0`。

**示例 3：标识远程 I/O 控制器**

```sh
# nvmecontrol identify nvme0
Controller Capabilities/Features
================================
...
Model Number:                Linux
Firmware Version:            5.15.0-8
...

Fabrics Attributes
==================
I/O Command Capsule Size:    16448 bytes
I/O Response Capsule Size:   16 bytes
In Capsule Data Offset:      0 bytes
Controller Model:            Dynamic
Max SGL Descriptors:         1
Disconnect of I/O Queues:    Not Supported
```

### 通过发现连接

`nvmecontrol(8)` 的命令 `connect-all` 从指定的发现控制器获取发现日志页面，并为每个日志页面条目创建关联。**示例 2** 中的关联可以通过执行 `nvmecontrol connect-all ubuntu:4420` 来创建，而不需要先获取发现日志页面并使用命令 `connect` 。

### 断开连接

`nvmecontrol(8)` 的命令 `disconnect` 从远程控制器断开命名空间并销毁关联。**示例 4** 断开 **示例 2** 中创建的关联。`disconnect-all` 命令销毁与所有远程控制器的关联。

**示例 4：断开与远程 I/O 控制器的连接**

```sh
# nvmecontrol disconnect nvme0
```

### 重新连接

如果连接中断（例如，一个/多个 TCP 连接失败），活动关联会被拆除（所有队列都会断开连接），但 `nvmeX` 设备会保持静止状态。所有挂起的远程命名空间的 I/O 请求也会保持挂起。在这种状态下，可以使用 `reconnect` 命令重新建立关联，以恢复与远程控制器的操作。**示例 5** 重新连接到 **示例 2** 中的控制器。请注意，`reconnect` 命令与 `connect` 命令类似，需要显式指定网络地址。

**示例 5：重新连接到远程 I/O 控制器**

```sh
# nvmecontrol reconnect nvme0 ubuntu:4420
```

## 控制器

FreeBSD 上的 Fabrics 控制器将本地块设备作为 NVMe 命名空间暴露给远程主机。FreeBSD 上的控制器支持包括发现控制器的用户空间实现和内核中的 I/O 控制器。与 FreeBSD 中现有的 iSCSI 目标类似，内核中的 I/O 控制器使用 CAM 的目标层 ([ctl(4)](https://man.freebsd.org/ctl/4))。块设备是通过使用 [ctladm(8)](https://man.freebsd.org/ctladm/8) 添加 ctl(4) LUN 创建的。发现服务和 I/O 控制器连接的初步处理由守护进程 [nvmfd(8)](https://man.freebsd.org/nvmfd/8) 管理。内核中的 I/O 控制器由 [nvmft(4)](https://man.freebsd.org/nvmft/4) 模块提供。**示例 6** 将名为 `pool/lun0` 的 ZFS 卷作为 ctl(4) LUN 添加，再启动守护进程 `nvmfd(8)` 。远程主机可以将此 ZFS 卷作为 NVMe 命名空间进行访问。

**示例 6：导出本地 ZFS 卷**

```sh
# kldload nvmft nvmf_tcp
# ctladm create -b block -o file=/dev/zvol/pool/lun0
LUN created successfully
backend:        block
device type:   0
LUN size:      4294967296 bytes
blocksize      512 bytes
LUN ID:        0
Serial Number: MYSERIAL0000
Device ID:     MYDEVID0000
# nvmfd -F -p 4420 -n nqn.2001-03.com.chelsio:frodo0 -K
```

每当远程主机连接到 I/O 控制器时，内核都会记录一条日志，列出远程主机的 NQN（如图 2 所示）。

**图 2：新关联的日志消息**

```sh
nvmft0: associated with
nqn.2014-08.org.nvmexpress:uuid:00000000-0000-0000-0000-ffffffffffff
```

在 `nvmfd(8)` 运行时，可以通过 `ctladm(8)` 添加和删除 LUN。如果在添加和删除 LUN 时有远程主机连接，则会向远程主机报告异步事件。这使得远程主机能够在连接的同时注意到命名空间的添加和删除。

`ctladm(8)` 添加了两个新命令来管理 Fabrics 关联。`nvlist` 命令列出所有来自远程主机的活动关联。**示例 7** 显示了命令 `nvlist` 的输出，当一个主机连接到 **示例 6** 中的控制器时。

**示例 7：列出活动关联**

```sh
# ctladm nvlist
  ID Transport        HostNQN                              SubNQN
   0 TCP
nqn.2014-08.org.nvmexpress:uuid:00000000-0000-0000-0000-ffffffffffff
nqn.2001-03.com.chelsio:frodo0
```

`nvterminate` 可命令关闭一个和多个关联。可以终止单个连接或 NQN 的关联，也可以终止所有活动关联。**示例 8** 终止了 **示例 7** 中的关联。关联终止后，内核记录图 3 中的日志消息。

**示例 8：终止关联**

```sh
# ctladm nvterminate -c 0
NVMeoF connections terminated
```

**图 3：终止后的日志消息**

```sh
nvmft0: disconnecting due to administrative request
nvmft0: association terminated
```

## 结论

对 NVMe-oF 的支持即包含主机和控制器的功能会引入 FreeBSD 15.0。Fabrics 功能的开发由 Chelsio Communications, Inc. 赞助。

---

**John Baldwin** 是一名系统软件开发人员，已在 FreeBSD 操作系统中直接提交了二十年余的修改，涉及内核的各个部分（包括 x86 平台支持、SMP、各种设备驱动程序和虚拟内存子系统）及用户空间程序。除了编写代码，John 还曾在 FreeBSD 核心和发布工程团队中工作，并为 GDB 调试器做出了贡献。John 住在加州的康科德，与妻子 Kimberly 和三个孩子 Janelle、Evan 和 Bella 一起生活。
