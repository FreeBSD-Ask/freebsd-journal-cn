# 存储多路径综述

作者：**ALEXANDER MOTIN**

存储多路径是一种旨在提升存储可靠性的技术，通过消除单点故障，并在多个物理连接之间分担负载以改善性能。多路径可在多个不同层次实现：传输层（如 iSCSI）、外设驱动层（如 SCSI 块设备驱动），或块存储转换层（如 FreeBSD GEOM）。每种实现各有优劣。

传输层多路径允许多个物理连接作为单一传输协议连接对外呈现。例如，iSCSI 协议定义了每会话多连接（MC/S）的可选支持，SCSI 请求和响应在多个 TCP 连接之间分发，对上层 SCSI 层而言表现为单一的发起方到目标方连接。当发起方或目标方存储栈本身不支持应用层多路径时，这项技术很有用。例如，Microsoft 仅在 Windows 服务器版本中支持 SCSI 层多路径（MPIO），而 iSCSI MC/S 即便在桌面版中也完全支持。该技术最大的缺点是只有使用相同传输协议（如 iSCSI）的路径才能参与——无法用这种方式以 iSCSI 备份 Fibre Channel fabric。此外，提供单一 SCSI 发起方-目标方连接语义（请求排序、错误恢复等）的要求使传输协议实现显著复杂化。例如，某条连接出现延迟或丢包时，其他连接不得不延迟本已交付的请求，等待该连接恢复，或必须实现快速问题检测机制，将受影响的请求经其他连接重新路由。出于这些原因，新的 FreeBSD 内核空间 iSCSI 发起方和目标方实现不支持 MC/S。旧的用户空间 iSCSI 目标方实现 istgt 声称支持 MC/S，但实现极简，缺乏完善的错误恢复，有时能提升存储性能，却可能在可靠性方面造成隐患。

外设驱动层多路径允许通过多个传输层连接（如 iSCSI、Fibre Channel、SAS 等）执行上层请求（如块读取、写入、删除等）。该技术允许同时使用不同的传输协议，仅限于单一应用协议（如 SCSI 块命令）。除非将一种命令集（NVMe）翻译为另一种（SCSI），否则无法在该层用 iSCSI 备份 NVMe-over-Fabric 连接。该层的多路径对请求执行顺序持完全不同的观点。由于目标方不知道多个发起方中哪个代表同一外设驱动（即正确排序请求的来源），它无法保证经不同路径到来的请求顺序。这显著简化了支持多路径的目标方实现，但要求外设驱动在路径之间特殊编译请求分发逻辑，以防止或至少最小化不必要的请求重排，可能以降低路径利用率为代价。幸运的是，外设驱动更了解请求的性质和来源，能以比传输层更巧妙的方式处理。

对于熟悉网络协议的人，可以做一个类比：传输层多路径类似于 Multilink PPP，它按上层 PPP 栈要求保证所有分组的完整原始顺序，在不查看内部的情况下将分组片段分摊到可用链路；而外设驱动层多路径可类比为 LACP，它通过查看分组的 MAC 和 IP 地址、TCP/UDP 端口等识别独立数据流，将其分配到各链路，仅在每条流内保证顺序。虽然第一种方法理论上对于少量数据流能提供更好的吞吐量，但第二种提供了更简单的无状态实现、更低的平均延迟，以及对单链路问题更好的弹性。

作为最小化外设驱动层多路径实现的例子，可以看 SAS HDD。每块 SAS HDD 通常有两个 SAS 端口，每个可连接到背板上独立的 SAS expander，再连接到主机独立的 HBA，为主机提供两个独立的 SCSI I_T_L（Initiator-Target-Logical Unit）关联。支持多路径的 SCSI 外设驱动可通过获取并比较全局唯一的逻辑单元名称（可以是 NAA 或 EUI ID、文本或二进制字符串、UUID 等）识别这两个 I_T_L 关联指向同一逻辑单元。之后，外设驱动可向上层报告单一块存储设备，并选择策略在两条路径之间分担负载。由于典型 SAS HDD 接口吞吐量远高于介质吞吐量，且顺序访问远快于随机访问，简单的故障转移配置（一次只用一条路径）就很合适。而对于高性能 SSD，单条 SAS 链路的吞吐量可能不足以饱和后端存储，需要同时利用两条路径才能达到满性能。

如果不支持或无法使用外设驱动多路径，该功能可在更高的块存储转换层实现，如 FreeBSD 的 GEOM 或 Linux 的 Device Mapper。该层以抽象的块读取、写入、删除等请求为单位操作，与任何传输或应用协议的具体细节无关。它可使用来自下层的带外信息，或独立工作。更好的集成使其操作可自动化，从而更可靠，但也使存储栈更复杂。FreeBSD GEOM MULTIPATH 类默认不使用任何带外信息，仅依赖存储在设备最后一个扇区的自身元数据和管理员提供的配置。但它也有一种不使用磁盘元数据的操作模式，依赖第三方代码的外部治理来管理路径检测、激活、优先级、故障等，并依赖逻辑单元名称等信息。

当目标端口不对等时，事情变得更有趣。SCSI 规范称之为非对称逻辑单元访问（ALUA）。ALUA 可由不同原因引起，通常由目标设备内存在多个存储控制器处理来自不同目标端口的请求导致，这些控制器因某种原因无法同时工作，或针对某些逻辑单元无法同时工作。非对称程度各异。最坏情况下，某些端口/控制器甚至无法报告逻辑单元的任何标识信息。ALUA 将此状态称为“Unavailable”。这个状态用处不大，因为它连自动设置多路径的充分信息都无法提供。下一个更好的状态叫“Standby”。在此状态下，端口能报告逻辑单元的完整标识，处理 SCSI 预留和部分管理请求，但无法访问介质。此状态允许自动多路径配置，但也要求发起方 OS 完全支持 ALUA 多路径，因为经错误路径发送的数据访问请求会失败。下一个状态叫“Active/Non-optimized”。此状态允许经该路径完全访问逻辑单元，但访问的性能特征可能多少有些次优，例如需要在存储控制器之间额外同步或传输命令/数据。此状态允许逻辑单元被使用，即便外设驱动不支持 ALUA，但若选错路径则效率低下。最后，端口可能拥有的最佳状态是“Active/Optimized”。此状态意味着经该端口可完全访问逻辑单元，性能最优。还有一个 ALUA 状态叫“Transitioning”，涵盖端口或逻辑单元正在改变状态、暂时不可访问的情况。支持 ALUA 的发起方的职责是持续监控每个逻辑单元的状态，选择最优路径发送请求。某些情况下，支持 ALUA 的发起方可显式或隐式请求端口状态转换，但这些请求的逻辑不在 SCSI 规范范围之内，由厂商自行定义。

ALUA 多路径操作可用 iXsystems Inc. 的 TrueNAS 存储设备为例来说明，该设备使用 FreeBSD CTL 子系统的高可用功能实现支持 ALUA 的多路径 SCSI 目标方。TrueNAS 高可用设备包含两个存储控制器，各自运行 FreeBSD 11，通过以太网或 PCIe Non-Transparent Bridge 内部链路连接，并可访问共享的 SAS 磁盘阵列。该设备使用 ZFS 池存储用户数据，但由于 ZFS 不是集群文件系统，任一时刻只有一个控制器能访问特定池。经 iSCSI 或 Fibre Channel 链路收到的 SCSI 请求，若接收控制器无权访问所需池，由 CTL 经内部链路代理到另一个控制器。在这种情况下，若不使用 ALUA 提供多路径功能，会根据所用内部链路类型造成额外延迟或严重的带宽限制。使用 ALUA 可让支持多路径的客户端了解当前状况，始终经最优路径提交请求。来看一些实际例子：

某双控制器存储系统配置为以 ALUA 模式提供一个 iSCSI 目标逻辑单元。其中一节点上 CTL 守护进程的多路径相关配置如下：

```sh
portal-group pg1A {
         tag 0x0001
         listen 10.20.20.234:3260
         foreign
}
portal-group pg1B {
         tag 0x8001
         listen 10.20.20.235:3260
}

lun "aluademo" {
         ctl-lun 0
         serial "ac1f6b0c248600"
         device-id "iSCSI Disk       ac1f6b0c248600                   "
         option vendor "TrueNAS"
         option product "iSCSI Disk"
         option revision "0123"
         option naa 0x6589cfc000000d7a21ae0e095faf3cea
}

target iqn.2005-08.com.ixsystems:aluademo {
         alias "aluademo"
         portal-group pg1A no-authentication
         portal-group pg1B no-authentication

         lun 0 "aluademo"
}
```

可以看到两个 iSCSI portal 配置了不同控制器的 IP。不同的 portal group tag 意味着这些 portal group 代表独立的 SCSI 端口，不能共享 MC/S 会话连接。可以看到一个 SCSI target 关联这些 portal group。还可以看到该 target 关联一个逻辑单元，包含多路径操作所需的一组不同唯一 ID。

现在用 FreeBSD iSCSI 发起方连接该目标方：

```sh
# iscsictl -A -d 10.20.20.234
# iscsictl -A -d 10.20.20.235
# iscsictl
Target name                             Target portal     State
iqn.2005-08.com.ixsystems:aluademo      10.20.20.234      Connected: da0
iqn.2005-08.com.ixsystems:aluademo      10.20.20.235      Connected: da1
```

由于 FreeBSD 发起方不支持外设驱动级多路径，到同一逻辑单元的两条路径被检测为两个独立的块设备 da0 和 da1。现在从存储侧查看 CTL 内的发起方和目标方端口列表：

```sh
# ctladm portlist -i
Port Online Frontend Name     pp vp
0    NO     ha        1:camsim 0  0  naa.5000000341ab1b01
    Target: naa.5000000341ab1b00
1    YES    ha        1:ioctl  0  0
2    YES    ha        1:tpc    0  0
3    YES    ha        1:iscsi  1  1  iqn.2005-08.com.ixsystems:aluademo,t,0x0001
    Target: iqn.2005-08.com.ixsystems:aluademo
    Initiator 0: iqn.1994-09.org.freebsd:mini.ixsystems.com,i,0x8047c8217cb4
128  NO     camsim   camsim   0  0  naa.50000003530efb81
    Target: naa.50000003530efb00
129  YES    ioctl    ioctl    0  0
130  YES    tpc       tpc       0  0
131  YES    iscsi    iscsi    32769 1  iqn.2005-08.com.ixsystems:aluademo,t,0x8001
    Target: iqn.2005-08.com.ixsystems:aluademo
    Initiator 0: iqn.1994-09.org.freebsd:mini.ixsystems.com,i,0x808be26435c8
```

这里看到两组目标方端口，分别由不同存储控制器服务：端口 0-127 属于控制器“a”，端口 128-255 属于控制器“b”。此时控制器“b”拥有 ZFS 池访问权。每个控制器都有自己的 iSCSI 目标方端口，名称唯一。每个 iSCSI 目标方端口都有一个接入的发起方端口，属于同一发起方系统，但会话 ID 不同（不属于 MC/S 或重连尝试）。

多路径客户端的操作由此开始。使用 sg3_utils 包中的 `sg_inq` 工具，获取 FreeBSD 报告的两个块设备 da0 和 da1 的标识信息：

```sh
# sg_inq -p di da0
VPD INQUIRY: Device Identification page
  Designation descriptor number 1, descriptor length: 59
    designator_type: T10 vendor identification,  code_set: ASCII
    associated with the Addressed logical unit
      vendor id: TrueNAS
      vendor specific: iSCSI Disk       ac1f6b0c248600
  Designation descriptor number 2, descriptor length: 20
    designator_type: NAA,  code_set: Binary
    associated with the Addressed logical unit
      NAA 6, IEEE Company_id: 0x589cfc
      Vendor Specific Identifier: 0xd7a
      Vendor Specific Identifier Extension: 0x21ae0e095faf3cea
      [0x6589cfc000000d7a21ae0e095faf3cea]
  Designation descriptor number 3, descriptor length: 48
    transport: Internet SCSI (iSCSI)
    designator_type: SCSI name string,  code_set: UTF-8
    associated with the Target port
      SCSI name string:
      iqn.2005-08.com.ixsystems:aluademo,t,0x0001
  Designation descriptor number 4, descriptor length: 8
    transport: Internet SCSI (iSCSI)
    designator_type: Relative target port,  code_set: Binary
    associated with the Target port
      Relative target port: 0x3
  Designation descriptor number 5, descriptor length: 8
    transport: Internet SCSI (iSCSI)
    designator_type: Target port group,  code_set: Binary
    associated with the Target port
      Target port group: 0x2
  Designation descriptor number 6, descriptor length: 40
    transport: Internet SCSI (iSCSI)
    designator_type: SCSI name string,  code_set: UTF-8
    associated with the Target device that contains addressed lu
      SCSI name string:
      iqn.2005-08.com.ixsystems:aluademo

# sg_inq -p di da1
VPD INQUIRY: Device Identification page
  Designation descriptor number 1, descriptor length: 59
    designator_type: T10 vendor identification,  code_set: ASCII
    associated with the Addressed logical unit
      vendor id: TrueNAS
      vendor specific: iSCSI Disk       ac1f6b0c248600
  Designation descriptor number 2, descriptor length: 20
    designator_type: NAA,  code_set: Binary
    associated with the Addressed logical unit
      NAA 6, IEEE Company_id: 0x589cfc
      Vendor Specific Identifier: 0xd7a
      Vendor Specific Identifier Extension: 0x21ae0e095faf3cea
      [0x6589cfc000000d7a21ae0e095faf3cea]
  Designation descriptor number 3, descriptor length: 48
    transport: Internet SCSI (iSCSI)
    designator_type: SCSI name string,  code_set: UTF-8
    associated with the Target port
      SCSI name string:
      iqn.2005-08.com.ixsystems:aluademo,t,0x8001
  Designation descriptor number 4, descriptor length: 8
    transport: Internet SCSI (iSCSI)
    designator_type: Relative target port,  code_set: Binary
    associated with the Target port
      Relative target port: 0x83
  Designation descriptor number 5, descriptor length: 8
    transport: Internet SCSI (iSCSI)
    designator_type: Target port group,  code_set: Binary
    associated with the Target port
      Target port group: 0x3
  Designation descriptor number 6, descriptor length: 40
    transport: Internet SCSI (iSCSI)
    designator_type: SCSI name string,  code_set: UTF-8
    associated with the Target device that contains addressed lu
      SCSI name string:
      iqn.2005-08.com.ixsystems:aluademo
```

对比输出可以看到，所有与被寻址逻辑单元相关的描述符完全相同。这意味着这两个设备确实代表同一逻辑单元。与目标方端口相关的描述符不同，提供了在目标方设备内标识它们的完整信息。可以看到其中的编号与 `ctladm portlist` 命令报告的信息匹配。

现在 ALUA 登场。获取端口状态的主要 SCSI 命令是 REPORT TARGET PORT GROUPS：

```sh
# sg_rtpg da0
Report target port groups:
  target port group id : 0x2 , Pref=0, Rtpg_fmt=0
    target port group asymmetric access state : 0x01
    T_SUP : 1, O_SUP : 0, LBD_SUP : 0, U_SUP : 1, S_SUP : 1, AN_SUP : 1, AO_SUP : 1
    status code : 0x02
    vendor unique status : 0x00
    target port count : 03
    Relative target port ids:
      0x01
      0x02
      0x03
  target port group id : 0x3 , Pref=0, Rtpg_fmt=0
    target port group asymmetric access state : 0x00
    T_SUP : 1, O_SUP : 0, LBD_SUP : 0, U_SUP : 1, S_SUP : 1, AN_SUP : 1, AO_SUP : 1
    status code : 0x02
    vendor unique status : 0x00
    target port count : 03
    Relative target port ids:
      0x81
      0x82
      0x83
```

这里看到目标方报告了两个端口组（每个存储控制器一个）和 6 个端口（每个存储控制器 3 个）。组和端口的标识与之前 `sg_inq` 和 `ctladm portlist` 命令的输出匹配。除已知信息外，该命令还告知每个组支持多种 ALUA 状态，且端口组 0x2（控制器“a”）当前处于“Active/Non-optimized”状态（0x01），而端口组 0x3（控制器“b”）处于“Active/Optimized”状态（0x00）。这告诉我们此刻最好经块设备 da1 发送所有请求，但如果经该路径失去连接，da0 设备也能处理请求，只是更慢。

遗憾的是，FreeBSD SCSI 磁盘外设驱动不支持使用该信息自动进行多路径操作。GEOM 层的多路径设置必须由管理员手动完成，或由某些外部脚本完成，例如 FreeNAS 软件对多路径 SAS 磁盘所做的那样。这里手动完成：

```sh
# kldload geom_multipath
# gmultipath label mp0 da0 da1
# gmultipath prefer mp0 da1
# gmultipath status
          Name   Status  Components
multipath/mp0  OPTIMAL  da0 (PASSIVE)
                           da1 (ACTIVE)
```

可以看到 da0 和 da1 设备现在都作为路径归属于 multipath/mp0 设备，da1 设备将处理 I/O，直到发生错误或管理员另行指示。其他发起方可能完全自动完成。例如，VMware vSphere 自动检测多路径设备并使用 ALUA 配置其操作。

发起方正确识别两条路径属于同一目标方，并经正确的控制器“b”发送 I/O。如果该控制器有多于一条路径，可使用最近使用（Most Recently Used，默认）或轮询（Round Robin）路径选择策略。轮询策略下，vSphere 在一定请求数后切换活动路径（默认 1,000 次）。较高的默认值无法用单台服务器完全利用所有路径的吞吐量，但如前所述，它维持了原始请求顺序。

开箱即用时，Windows Server 仅支持 MC/S 传输层多路径。上层多路径需作为单独的可选功能安装。但组件安装并数次系统重启后，如果系统还活着 ;），ALUA 多路径目标方应能被正确检测并工作：

总结：作为目标方使用时，FreeBSD CAM Target Layer（CTL）能提供不错的 SCSI 多路径功能，支持 ALUA 和高可用集群，并兼容许多第三方发起方。iSCSI MC/S 的传输层多路径因高复杂性和有限范围而不被支持，但在有许多 Windows 桌面的环境中可能有用。作为发起方使用时，FreeBSD 目前只能提供基本的 GEOM 层多路径，不支持 ALUA。某种程度上可由外部脚本补偿，但明智之举是开箱即用便实现带 ALUA 支持的完整 SCSI 层多路径。

---

**ALEXANDER MOTIN** 是 iXsystems Inc. OS/服务团队负责人，自 2007 年起成为 FreeBSD 源代码提交者。
