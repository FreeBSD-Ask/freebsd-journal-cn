# SVN 动态

**作者：Steven Kreuzer**

最近我遇到了一件令人兴奋的事。某天晚上，我在 **/usr/src** 中运行 `svn up`，启动构建，然后上床睡觉。第二天早上醒来时，发现笔记本停在提示符下，要求我登录到 FreeBSD 13.0 工作站。这只能意味着一件事。我们目前正处于 FreeBSD 12.0 发布周期中！

一段时间里，HEAD 没什么动静，因为发布工程团队冻结了源码树，以创建将成为 12-STABLE 的分支。冻结现已解除，我们再次看到大量新功能和令人兴奋的变化。你能想到更好的方式来结束这一年吗？

## 添加读取 RISC-V 性能计数器 CSR 的宏

<https://svnweb.freebsd.org/changeset/base/340399>

RISC-V 规范定义了几个性能计数器 CSR，如：cycle、time、instret、hpmcounter(3...31)。它们在所有 RISC-V 架构上定义为 64 位宽。在 RV64 和 RV128 上，它们可以从单个 CSR 读取。在 RV32 上，存在附加 CSR（带后缀“h”），包含这些计数器的高 32 位，也必须读取。（详情见 User ISA Spec 第 2.8 节。）

此更改添加了在任何 RISC-V ISA 长度上安全读取这些值的宏。显然，我们目前只支持 RV64，但这确保了如果我们将来支持其他 ISA 长度，不需要更改读取这些值的方式。

## 为 RISC-V 实现 get_cyclecount(9)

<https://svnweb.freebsd.org/changeset/base/340400>

通过读取 cycle CSR，为 RISC-V 添加缺失的 get_cyclecount(9) 实现。

## 默认为 RISC-V 启用非可执行栈

<https://svnweb.freebsd.org/changeset/base/340231>

## 为 RISC-V 启用全局共享页

<https://svnweb.freebsd.org/changeset/base/340228>

machine/vmparam.h 已定义 SHAREDPAGE 常量。此更改仅为 ELF 可执行文件启用它。共享页目前唯一用途是持有信号蹦床。

## 添加新的 rc 关键字：enable、disable、delete

<https://svnweb.freebsd.org/changeset/base/339971>

向 rc/service 添加新关键字，用于启用/禁用服务的 rc.conf(5) 变量，以及“delete”用于移除变量。

当 **/etc/rc.conf** 中的 `service_delete_empty` 变量设置为“YES”（默认为“NO”）时，使用 `service $foo delete` 修改后，如果 **/etc/** 或 **/usr/local/etc** 中的 rc.conf.d 文件为空，则会被删除。

## libcasper：引入 cap_fileargs 服务

<https://svnweb.freebsd.org/changeset/base/340373>

cap_fileargs 是一个 Casper 服务，帮助沙箱化需要访问文件系统命名空间的应用程序。该服务的主要目的是让 capsicum 化处理在 argv 中传入多个文件的应用程序变得容易。

## 使用 capsicum 沙箱化 head

<https://svnweb.freebsd.org/changeset/base/340376>

## 使用 capsicum 沙箱化 wc

<https://svnweb.freebsd.org/changeset/base/340374>

## 为 netmap 添加负载均衡程序

<https://svnweb.freebsd.org/changeset/base/340279>

添加 lb 程序，能够将来自 netmap 端口的输入流量负载均衡到 M 个组，每组 N 个 netmap 管道。每个接收的数据包转发到每组中选择的管道之一（使用 L3/L4 连接一致性哈希函数）。

## 允许使用相同隧道端点配置多个 ipsec 接口

<https://svnweb.freebsd.org/changeset/base/340477>

这可用于在两台主机之间配置多个具有不同安全关联的 IPsec 隧道。

## 添加对非 ACPI 电池方法电池的支持

<https://svnweb.freebsd.org/changeset/base/340832>

移除设备必须是 ACPI 方法电池才能作为电池受支持的要求。现在要求设备位于 battery devclass 中，并实现 get_status 和 get_info 函数。这允许非 ACPI 方法电池受支持。

## 允许将本地内核模块作为内核构建的一部分构建

<https://svnweb.freebsd.org/changeset/base/339901>

添加对“本地”模块的支持。默认情况下，这些模块位于 LOCALBASE/sys/modules（LOCALBASE 默认为 **/usr/local**）。通过将 `LOCAL_MODULES` 定义为模块列表，可以连同内核一起构建单个模块。每个模块被假定为包含有效 Makefile 的子目录。如果未指定 `LOCAL_MODULES`，则构建 LOCALBASE/sys/modules 中存在的所有模块，并随内核一起安装。

这意味着安装内核模块的 port 可以选择将其源码连同合适的 Makefile 安装到 **/usr/local/sys/modules/<foo>**。未来的内核构建将使用内核配置的 opt_*.h 头文件构建该内核模块，并将其安装到 **/boot/kernel** 以及其他内核特定模块。

## 为主动 AUX 端口多路复用器添加最小支持

<https://svnweb.freebsd.org/changeset/base/340913>

主动 PS/2 多路复用是一种将最多四个 PS/2 指点设备连接到计算机的方法。启用多路复用模式允许使用路由前缀将命令定向到各个设备。多路复用模式报告输入，每个字节都标记以识别其来源。此方法与 psm(4) 当前支持的方法不同，后者将所谓的客户设备（trackpoint）连接到位于主机设备（touchpad）上的特殊接口，稍后对客户协议转换执行特殊封装数据包格式。

---

**STEVEN KREUZER** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车感兴趣。他与妻子、女儿和狗居住在纽约皇后区。
