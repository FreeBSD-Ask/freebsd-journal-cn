# SVN 动态

**作者：Steven Kreuzer**

FreeBSD 11 发布以来，src 中相当活跃，我猜这是因为开发者有想做的修改但不希望发布周期内提交。如今回归常态，各子系统都出现一波活动。其中较有趣的变化是 cron 与 syslog 都增加了功能，使你更容易用喜欢的配置管理系统推行变更。随着基础设施规模增长、我们走向“牛而非宠物”的服务器管理哲学，不必写复杂逻辑就能快速推送变更，我相信任何系统管理员都会张开双臂欢迎。希望这是基础系统中其他程序也开始跟进的趋势。

## daemon(8)：允许将守护进程的 stdout/stderr 记录到文件或 syslog

daemon 工具会从控制终端分离并执行其参数所指定的程序。过去，进程产生的任何输出都会写到本地磁盘文件。此变更允许将输出集中到标准位置，甚至发往远程 syslog 服务器。

链接：<https://svnweb.freebsd.org/changeset/base/307769>

## 对 树莓派 3 的支持

已提交对 树莓派 3 的初步支持。虽然 SMP、VCHIQ 与 RNG 驱动目前不支持，但多位 FreeBSD 开发者仍在为这一非常流行的平台做开发，请关注 freebsd-arm 邮件列表以跟踪进展。

链接：<https://svnweb.freebsd.org/changeset/base/307257>

## cron(8)：添加对 **/etc/cron.d** 和 **/usr/local/etc/cron.d** 的支持

cron 任务现在可拆分为单独文件，与修改 **/etc/crontab** 相比，用各种自动化工具部署和管理这些任务容易得多。

链接：<https://svnweb.freebsd.org/changeset/base/308139>

## 允许 SMP 内核在 Cortex-A8 上启动

此变更允许 FreeBSD 在所有支持 32 位的 ARMv7+ Cortex-A 核心上启动。

链接：<https://svnweb.freebsd.org/changeset/base/308213>

## syslogd(8)：新增 'include' 关键字

include 关键字后所跟目录中所有不以 '#' 开头的 .conf 文件都会被包含。默认 syslogd.conf 已更新为 include **/etc/syslog.d** 与 **/usr/local/etc/syslog.d**。

链接：<https://svnweb.freebsd.org/changeset/base/308160>

## 对 Allwinner H3 音频编解码器的支持

H3 中的音频控制器与 A10/A20 大体相同——只是部分寄存器被调整。但混音器接口在不同 SoC 之间完全不同。已分别提供 `a10_mixer_class` 与 `h3_mixer_class` 实现，使为其他 SoC 添加支持更为容易。

链接：<https://svnweb.freebsd.org/changeset/base/308269>

## 为 bhyve 添加 virtio-console 支持

bhyve 新增一个设备驱动，允许在主机与客户机之间创建多达 16 条双向字符流。

链接：<https://svnweb.freebsd.org/changeset/base/305898>

## zfsbootcfg(8) 提供为 zfsboot 设置一次性下次启动选项的能力

zfsboot 会从一块特殊的 ZFS 池区域读取一次性启动指令。该区域此前被称为“Boot Block Header”，现称 Pad2，在池创建时被标记为保留并清零。新代码使用与 boot.config 相同的格式解读该区域的数据（如果有的话）。

链接：<https://svnweb.freebsd.org/changeset/base/308089>

## 从用户态操作 EFI 变量

`efivar(1)` 是操作可扩展固件接口（Extensible Firmware Interface）变量的新工具。其命令行界面与 Linux 等价工具相似，并新增许多有用功能以便在 shell 脚本中使用。

链接：<https://svnweb.freebsd.org/changeset/base/307072>

## 为 Freescale PowerPC e500v2 核心引入新的 MACHINE_ARCH

Freescale e500v2 PowerPC 核心不使用标准 FPU，而是使用信号处理引擎（Signal Processing Engine，SPE）——一种 DSP 风格的向量处理器单元，兼作 FPU。PowerPC SPE ABI 与标准 powerpc ABI 不兼容，因此创建新的 MACHINE_ARCH 来处理。

链接：<https://svnweb.freebsd.org/changeset/base/307761>

STEVEN KREUZER 是 FreeBSD 开发者兼 Unix 系统管理员，爱好复古计算与风冷大众汽车。他与妻女和狗住在纽约皇后区。
