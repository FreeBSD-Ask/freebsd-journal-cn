# SVN 动态

- 原文：[svn update](https://freebsdfoundation.org/our-work/journal/browser-based-edition/mips-and-arm64/)
- 作者：**Glen Barber**

又到了这个时候——新年伊始，正是回顾 FreeBSD 开发者过去一年中带来的新功能和改进的好时机。坐下来，放松，欣赏这些亮点。

无论你是 FreeBSD 系统管理员、应用开发者还是业余爱好者，系统的变化，尤其是工具和配置的变化，都是需要密切关注的重要内容。以下是即将发布的 FreeBSD 版本中可以期待的一些变化。

## mailwrapper.8 工具

<http://svnweb.freebsd.org/changeset/base/270675>

`mailwrapper.8` 工具用于根据配置文件 `mailer.conf.5` 的指定调用合适的 MTA（邮件传输代理），例如 Sendmail、Postfix 或 qmail。`mailer.conf.5` 文件用于将邮件子系统使用的某些命令映射到应执行该命令的程序绝对路径。

例如，`mailer.conf.5` 可能包含‘sendmail’、‘mailq’、‘newaliases’和‘hoststat’等命令。在 FreeBSD 上，这些命令映射到 **/usr/libexec/sendmail/sendmail** 程序，该程序根据调用它的命令名决定行为，以不同方式修改邮件系统的各部分。`newaliases.8` 会重建 `aliases.5` 数据库文件 aliases.db；`hoststat.8` 会打印 SMTP 事务中使用的主机状态数据库等。历史上，`mailer.conf.5` 存放在基本系统配置目录 **/etc/mail** 中，但从 revision 276917 开始，可以将 `mailwrapper.8` 配置文件存放在系统其他位置，避免冲突或意外撤销本地修改。`mailwrapper.8` 工具现在会尊重 `LOCALBASE` 环境变量，优先在该目录下查找配置文件，如果存在则覆盖 FreeBSD 基本系统默认的 `mailer.conf.5`。在 FreeBSD 术语中，`LOCALBASE` 是非基本系统安装路径的根，即 **/usr/local**。

## rc.8 子系统

<http://svnweb.freebsd.org/changeset/base/276918>

`rc.8` 子系统是 FreeBSD 中最关键的子系统之一，用于决定某项服务是否在启动时开启，以及服务启动时使用的命令行参数。尽管 `rc.8` 子系统可以说是 FreeBSD 中最复杂的部分之一，但对管理员而言其配置一直保持相当简洁，而它的简洁性和可扩展性又迎来了一次更新。`rc.8` 子系统现在支持在 **/etc/rc.conf.d** 和 `LOCALBASE`/etc/rc.conf.d（默认展开为 **/usr/local/etc/rc.conf.d**）中放置服务启动配置文件，管理员可以为系统配置创建更小的配置文件。

rc.conf.d 目录中的每个文件可以包含某种服务类型，进而包含该服务的配置参数。一个常见示例是创建 **/etc/rc.conf.d/jail** 文件来存放所有 `jail.8` 启动参数，以及 **/usr/local/etc/rc.conf.d/postfix** 文件来存放 Postfix 邮件服务器的启动参数。

## linux.4 ABI 层

<http://svnweb.freebsd.org/changeset/base/271982>

经过大量工作和测试，加上更多现实需求的推动，`linux.4` ABI（应用二进制接口）层从 Fedora Core 10 支持升级到 CentOS 6 支持，使许多仅限 Linux 的较新应用程序能在 FreeBSD 上运行。要查看系统上的 Linux ABI 兼容版本，可查看 `compat.linux.osrelease` `sysctl.8` 的输出：

- 2.6.16：Fedora Core 10
- 2.6.18：CentOS 6

如果需要继续使用 Fedora Core 10 ABI 层，可在 `sysctl.conf.5` 中添加 `compat.linux.osrelease=2.6.16` 将默认值回滚到 Fedora Core 10。

## bsdinstall.8 安装器

<http://svnweb.freebsd.org/changeset/base/272274>

`bsdinstall.8` 安装器自 FreeBSD 9.0-RELEASE 起成为默认安装工具。自 9.0-RELEASE 以来，`bsdinstall.8` 接收了各种更新和增强，以取代其前任 `sysinstall.8`。

revision 272274 中的变更对系统管理员和开发者都很重要，尤其是使用 root-on-ZFS 安装时——默认的 **/var** 数据集现在设置了 `canmount=off` ZFS 属性，因此虽然 **/var** 默认会创建，但不会在启动时自动挂载。这让系统可以拥有多个 **/var** 数据集，配合 sysutils/beadm port 等多引导环境使用时，可防止多个独立环境的 **/var** 目录相互冲突。

**/var** 下有大量非常重要的文件。从 `pkg.8` 数据库、`periodic.8` 运行生成的备份文件、`crontab.5` 文件到系统崩溃转储，都存放在这里。因此，确保多个共享物理硬件（以及用于控制默认启动环境的底层工具）的独立环境不共享这些数据非常重要，这样一个环境就不会与另一个环境冲突，例如已安装的软件包。

## crypto.4 驱动

<http://svnweb.freebsd.org/changeset/base/275732>

`crypto.4` 驱动是 OpenCrypto 框架的一部分，支持两种新的加密模式——AES-ICM 和 AES-GCM。`crypto.4` 驱动为用户空间提供对硬件加速加密设备的访问，并与 `aesni.4`、`ipsec.4`、`padlock.4` 和 `random.4` 等其他加密设备配合使用。

OpenCrypto 框架的更新由 FreeBSD 基金会赞助。

## vxlan.4 设备

<http://svnweb.freebsd.org/changeset/base/273331>

`vxlan.4` 设备最近加入 FreeBSD，类似于 `vlan.4`，用于创建虚拟隧道端点。`vxlan.4` 网段是叠加在三层端点上的虚拟二层网络，相比 `vlan.4`，更适合多租户数据中心环境。

## gre.4 驱动

<http://svnweb.freebsd.org/changeset/base/274246>

`gre.4` 驱动提供通用路由封装接口，现已拆分为两个独立模块——`gre.4` 和 `me.4`。`gre.4` 模块本身已大幅改写，其对应模块 `me.4` 则提供三层内更精简的封装接口。

感谢读者支持 FreeBSD 社区、FreeBSD 期刊，当然还有 FreeBSD 基金会。

别忘了，FreeBSD-CURRENT 和 FreeBSD-STABLE 分支的开发版 ISO 和预装虚拟机镜像（VHD、VMDK、QCOW2 和 RAW 格式）可在 FTP 镜像上找到，每周构建：<ftp://ftp.freebsd.org/pub/FreeBSD/snapshots/>。一如既往，开发版快照不适用于生产环境；但我们鼓励定期测试，这样我们才能让下一个 FreeBSD 发布版本如你所期待的那样出色。

---

**Glen Barber** 以业余爱好者身份于 2007 年前后深度参与 FreeBSD 项目。此后他参与了多项工作，最近的职位让他专注于项目中的系统管理和发布工程。Glen 居住在美国宾夕法尼亚州。
