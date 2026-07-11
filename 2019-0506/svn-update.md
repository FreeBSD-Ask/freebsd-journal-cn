# SVN 动态

作者：**Steven Kreuzer**

每隔两个月，我就坐到电脑前翻阅数千条提交信息，寻找最激动人心、最有趣的变更，再花几个小时把它们整理成名为 SVN 动态的专栏。随着女儿长大、工作责任加重，挤出那几个小时变得越来越难。

我终于可以对自己承认，是时候停下这些随笔了。希望你们像我撰写它们时那样喜欢阅读这些专栏。我从心底感谢大家。晚安，祝好运。

## 通过 sysctl 暴露内核的 build-ID

<https://svnweb.freebsd.org/changeset/base/348611>

在我们（某些架构）迁移到 lld 之后，内核构建时带有唯一的 build-ID。通过 sysctl 和 **uname(1)** 暴露它，让用户能识别正在运行的内核。

## 修改 mountd，使其在重载时增量更新内核导出

<https://svnweb.freebsd.org/changeset/base/348590>

没有这个补丁时，`mountd` 收到 SIGHUP 会从导出文件中删除/加载所有导出。这对小型导出文件没问题，但当导出文件系统数量巨大（10,000+）时会耗时数秒。大部分时间花在执行系统调用以删除/导出每个文件系统上。当指定了 `-S` 选项（如今是默认值）时，`nfsd` 线程在重载完成前会被挂起数秒。

这个补丁修改 `mountd`，使其只对与前一次加载/重载导出文件相比导出发生变更/新增/删除的文件系统执行系统调用。基本原理是，当 SIGHUP 发送给 `mountd` 时，它保存前一次加载的 `exportlist` 结构，并从当前导出文件创建一组新结构，然后对比当前与之前，只对发生变更/新增/删除的情况执行系统调用。`nfsd` 线程只需在对比步骤进行时挂起。结果，对于有 10,000+ 个导出文件系统的服务器，挂起时间只有毫秒级。

## 为加密内核崩溃转储添加 Chacha20 模式

<https://svnweb.freebsd.org/changeset/base/348197>

Chacha20 不要求消息为块大小的整数倍，因此可以在非块大小的消息上使用该加密算法，无需 AES-CBC 所需的显式填充。因此允许与转储压缩同时使用。（继续禁止 AES-CBC EKCD 与压缩同时使用。）

**dumpon(8)** 增加 `-C cipher` 标志，在 chacha 和 aes-cbc 之间选择。未提供 `-C` 选项时默认为 chacha。man 页面记录了该行为。

## 添加 RFC8086 定义的 GRE-in-UDP 封装支持

<https://svnweb.freebsd.org/changeset/base/346630>

这种 GRE-in-UDP 封装允许将 UDP 源端口字段用作熵字段，用于在传输网络中对 GRE 流量进行负载均衡。此外，大多数多队列网卡能将入站 UDP 数据报分发到不同 NIC 队列，而能对 GRE 数据包做到这点的网卡很少。

## 为 bsnmp 添加 IPv6 传输

<https://svnweb.freebsd.org/changeset/base/345797>

这个补丁新增表 `begemotSnmpdTransInetTable`，使用 `InetAddressType` 文本约定，可用于为 IPv4、IPv6、带区域标识的 IPv6 以及基于 DNS 名称创建监听端口。它还通过在表索引中加入协议标识符来支持未来向 UDP 之外的扩展。

## 允许在 jail 中使用显式分配的 IPv6 回环地址

<https://svnweb.freebsd.org/changeset/base/316328>

如果 jail 显式分配了 IPv6 回环地址，则允许使用它，而不是将对回环地址的请求重映射到分配给 jail 的第一个 IPv6 地址。这修复了应用尝试检测其绑定端口时遇到的问题——它们请求回环地址（本应可用），但内核将其重映射为 jail 的第一个地址。

## 从默认 i386 GENERIC 内核配置中移除 i486

<https://svnweb.freebsd.org/changeset/base/314669>

Intel 于 2007 年 9 月停止 486 生产。从 GENERIC 内核中移除 486 配置选项可略微提升性能。

移除 `I486_CPU` 此刻是合理的：我们不支持任何不带 FPU 的处理器，而频繁使用 i486 CPU 的 PC-98 架构也已不复存在，因此我们不再测试此类平台。

---

**STEVEN KREUZER** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车感兴趣。他与妻子、女儿和狗住在纽约皇后区。
