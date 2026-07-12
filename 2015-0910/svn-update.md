# SVN 动态

- 原文：[svn Update](https://freebsdfoundation.org/our-work/journal/browser-based-edition/cloudabi/)
- 作者：**Glen Barber**

FreeBSD 10.2-RELEASE 虽刚发布，本期的 SVN 动态仍将聚焦即将到来的 FreeBSD 版本中一些令人期待的更新。敬请享用！

## GENERIC 内核中的 IPsec

<http://svnweb.freebsd.org/changeset/base/285142>

IPsec 是一组网络协议，用于在 Internet 上的主机之间进行安全的端到端通信，对每个 IP 数据包进行加密。其实现细节见 RFC4301。

在对底层加密代码和网络协议栈进行大量更新后，IPsec 已在 FreeBSD 所有 GENERIC 内核配置中默认启用。

启用 IPsec 默认支持前的一些更新包括：

- 新增对软件和硬件 AES 模式的支持，可在 CPU 支持 AES-NI 指令的系统上使用硬件加密加速（r285336）——由 Rubicon Communications（Netgate）赞助。
- 移除了对 SKIPJACK 的支持。
- 新增 AES-GCM 与 AES-ICM 加密的硬件加速支持。

将 IPsec 纳入 GENERIC 内核后，需要该功能的第三方软件（如 `security/ike` Port）无需重新编译内核即可开箱即用。

## 初步的 NUMA 亲和性与策略配置

<http://svnweb.freebsd.org/changeset/base/285387>

非统一内存访问（Non-Uniform Memory Access，NUMA）是一种计算机体系结构设计，其中访问特定内存或输入/输出设备的延迟取决于该内存或设备所连接的处理器。在多核系统上，若处理器需要访问连接在其他处理器上的内存地址或设备，延迟可能进一步增加。

FreeBSD 上首次出现 NUMA 实现是在 9.0 版本，但当时不可配置。截至 r285387，引入了可配置的 NUMA 策略与线程/进程亲和性分配的初步实现——由 Norse Corp. Inc. 与 Dell Inc. 赞助。

NUMA 策略可通过 `numactl(1)` 工具修改和查询，配置则由 `numa_getaffinity()` 与 `numa_setaffinity(2)` 系统调用管理。

若系统未在内核配置文件中定义 MAXMEMDOM 选项，或 MAXMEMDOM=1，则不受此变更影响；但可将内核配置中的 MAXMEMDOM 设为大于 1 的值，以利用 NUMA 策略与亲和性调优。（更多信息见 `numa(4)` 手册页。）

## jail(8) 用法更新

<http://svnweb.freebsd.org/changeset/base/286064>

`jail(8)` 工具用于在运行中的系统上创建、修改或移除现有的 Jail。Jail（或称“prison”）是一种 `chroot(8)` 环境，拥有独立的 IP 地址空间、用户账户和进程，可用于进程隔离，与运行中的系统及其他 Jail 分离开来。

`jexec(8)` 用户态工具通过指定命令与目标 Jail 交互。过去，必须显式指定要在 Jail 中运行的命令，即便目标是 Jail 中的 shell 也一样。自 r286064 起，若未指定命令参数，`jexec(8)` 会在目标 Jail 中运行 shell。此外新增了标志 `-l`，用于丢弃登录类中的所有环境变量，仅保留 HOME、SHELL、TERM 与 USER。

## 工具链组件更新

<http://svnweb.freebsd.org/changeset/base/288943>

主要的外部贡献工具链组件均已更新，与上游版本保持一致。受影响组件包括 clang、llvm、lldb、compiler_rt 与 libc++，全部更新至上游版本 3.7.0。

各组件的发布说明见：

- <http://llvm.org/releases/3.7.0/docs/ReleaseNotes.html>
- <http://llvm.org/releases/3.7.0/tools/clang/docs/ReleaseNotes.html>

## bhyve(4) 增强

<http://svnweb.freebsd.org/changeset/base/288522>

`bhyve(4)` 是自 FreeBSD 10.0-RELEASE 起原生提供的 BSD 授权 hypervisor。

bhyve 近期迎来多项改进，最值得注意的是增强了 UEFI 支持。其中最引人关注的是，该更新允许在 bhyve 中初步运行 Windows。

目前安装 Windows Server 更容易，因为它支持无头串行控制台安装，而桌面版 Windows 则不行。

详情及 How-To 文档链接，请参见原始公告邮件：<https://lists.freebsd.org/pipermail/freebsd-virtualization/2015-October/003832.html>

## EOF

亲爱的读者，感谢你支持 FreeBSD 社区、FreeBSD 期刊，当然还有 FreeBSD 基金会。

别忘了，FreeBSD-CURRENT 与 FreeBSD-STABLE 分支的开发 ISO 与预装虚拟机镜像（VHD、VMDK、QCOW2 与 RAW 格式）每周构建一次，可在 FTP 镜像站点获取：<ftp://ftp.freebsd.org/pub/FreeBSD/snapshots/>。一如既往，开发快照不适用于生产环境；不过我们鼓励大家定期测试，以便让下一个 FreeBSD 版本如你所期望的那样出色。

**作者简介**

作为业余爱好者，Glen Barber 自 2007 年前后开始深度参与 FreeBSD 项目。此后他承担过各种职责，最新的角色让他能够专注于项目中的系统管理与发布工程。Glen 居住在美国宾夕法尼亚州。
