# SVN 动态

- 原文：[svn update](https://freebsdfoundation.org/our-work/journal/browser-based-edition/virtualization/svn-update/)
- 作者：**Glen Barber**

## FreeBSD 9.3-RELEASE 发布周期

FreeBSD 9.3-RELEASE 发布周期已接近尾声。FreeBSD 9.3 基于 FreeBSD 9.2 的可靠性和稳定性构建，并为 9.X 系列带来一系列修复和改进。

自 FreeBSD 9.2-RELEASE 以来的变更列表可在此处查看：
<http://www.FreeBSD.org/releases/9.3R/relnotes.html>

### ZFS 支持加入 bsdinstall(8)

- stable/9@r264437（<http://svnweb.freebsd.org/changeset/base/264437>）

在 10.0-RELEASE 发布周期接近尾声时，对 ZFS 数据集的自动安装被加入 FreeBSD 安装器 bsdinstall(8)。stable/9 分支的第 264437 次修订为 bsdinstall(8) 引入了该功能和其他若干变更，例如安装到 GELI 加密的 geom(4) 设备，并将扇区自动对齐到 4096 位。

### ttys(5) 新增“onifconsole”选项

- stable/9@r267243（<http://svnweb.freebsd.org/changeset/base/267243>）

ttys(5) 文件新增标志——“onifconsole”。当 tty 是活动的内核控制台时，新标志等同于“on”（启用串行控制台）；否则默认为“off”。这对嵌入式系统特别有用，因为这类系统可能有一条或多条可用串行通道，或者在使用 IPMI SoL（串口 over LAN）连接时，“默认”tty 可能有所不同。此变更最早出现在 head/ 的第 267243 次修订中。

### 按 Jail 名过滤进程

- stable/10@r266280（<http://svnweb.freebsd.org/changeset/base/266280>）

top(1) 工具更新后加入新标志 `-J`，可按 jail(8) 的名称或编号过滤进程表。此变更之前，运行系统上的所有进程都会显示，无论该进程是否在 jail 中运行。现在可以将进程列表限制为仅显示在指定 jail 内运行的进程。如要将进程列表限定为仅显示在 jail 环境外运行的进程，可指定 jail 编号 `0`。

### vt(4) 现已纳入 GENERIC 内核

- head/@r268045（<http://svnweb.freebsd.org/changeset/base/268045>）
- stable/9@r266269（<http://svnweb.freebsd.org/changeset/base/266269>）

vt(4) 驱动现已纳入 GENERIC 内核配置。vt(4) 驱动是与 KMS（内核模式设置）显卡集成的系统控制台驱动程序。在 vt(4) 出现之前，Xorg 环境启动后无法切回系统虚拟终端。此变更新增 loader(8) 可调参数 `kern.vty`，当其设为 `vt` 时启用 vt(4) 虚拟终端驱动。

### 可加载 Xen 内核模块

在 FreeBSD 10.0-RELEASE 之前，将 FreeBSD 作为虚拟机客户机运行在 Xen 虚拟机监控器上需要对内核配置进行特殊修改。FreeBSD 10.0 原生支持 Xen 环境，无需特殊的内核配置选项。FreeBSD 9.3 也将原生支持作为 Xen 虚拟机运行，方法是随系统附带内核模块 `xenhvm.ko`。

---

**Glen Barber** 大约在 2007 年以爱好者身份深度参与 FreeBSD 项目。此后他参与过多种工作，最近的岗位让他能够专注于 FreeBSD 项目的系统管理和发布工程。Glen 居住在美国宾夕法尼亚州。
