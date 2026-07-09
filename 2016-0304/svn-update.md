# SVN 动态

- 作者：**Steven Kreuzer**

10.3-RELEASE 的代码冻结已经持续了一段时间。你读到本栏目时，它应该已经可用。本期 SVN 动态我本打算介绍一些新特性和改进，但 11-CURRENT 最近的更新让我兴奋不已，于是临时改了主意。你们许多人也在翘首等待 10.3 的到来，stable/10 的发行说明中列出了自 10.2-RELEASE 以来的变更（<https://www.freebsd.org/relnotes/10-STABLE/relnotes/article.html>）。如果你想参与发布周期提供帮助，强烈建议试用现已可下载的 10.3-BETA 构建，并报告任何你可能遇到的回归问题。

## 对 RISC-V 指令集架构（ISA）的支持

i386 和 amd64 虽最流行，但 FreeBSD 支持众多架构，让你可以在从嵌入式 ARM 微控制器到大型 SPARC 工作站等异构硬件上安装自己喜欢的操作系统。近期一次更新加入了对一种全新的、完全开放的 ISA 的支持——RISC-V，它以 BSD 许可证向任何人开放。RISC-V 最初在加州大学伯克利分校开发，用于支持计算机架构研究与教育，现在正成为业界实现的标准开放架构。FreeBSD 是首个为 RISC-V 提供可启动的树内支持的操作系统，由于 FreeBSD 的历史也可追溯至伯克利，支持这一激动人心的新架构可谓顺理成章。（<https://svnweb.freebsd.org/changeset/base/295041>）

## 对 POWER7 和 POWER8 上 Vector-Scalar eXtension（VSX）的内核支持

IBM POWER 架构通过 VMX 和 VSX 指令集提供向量和向量标量运算，它们是 2.06 版 POWER ISA 的一部分。FreeBSD/powerpc 移植版已加入对该指令集的支持，把 32 个 64 位标量浮点寄存器与 32 个 128 位向量寄存器统一为 64 个 128 位寄存器的单一寄存器组。（<https://svnweb.freebsd.org/changeset/base/279189>）

## Xen Netfront 驱动获得多队列支持

在 Citrix Systems R&D 的持续支持下，FreeBSD 上的 Xen 正在积极开发中，近期一次更新将帮助缓解虚拟化环境中性能下降的一个主要来源。除了大规模重构外，netfront 驱动还加入了支持多对 TX/RX 队列的能力。网络负载沉重的客户机将看到显著改善，因为虚拟化 I/O 时最大的成本之一是如何高效地让多个虚拟机安全共享单一设备的访问。（<https://svnweb.freebsd.org/changeset/base/294442>）

## OpenSSH 升级至 7.1p2

最新版本的 OpenSSH 已提交到树中。新版本解决了最近的 Use Roaming 安全问题（详见 CVE-2016-0777 和 CVE-2016-0778），并修复了包处理代码中的越界读访问。此外，还在多处缓冲处理代码路径中增加了对 `explicit_bzero` 的使用，以防范编译器激进做死存储消除。（<https://svnweb.freebsd.org/changeset/base/294496>）

## HPN 已从 OpenSSH 中移除

HPN 是来自匹兹堡超级计算中心的一组补丁，移除了 OpenSSH 中的若干瓶颈以改进网络性能，尤其是在长距离和高带宽网络链路上。遗憾的是，这些补丁的实用性有限，而在树中维护它们又需要相当大的工作量。因此，它们连同 None 密码一起被移除。虽然大多数用户不会受此变更影响，但受影响的用户应改用 openssh-portable port，后者仍提供这两类补丁并默认启用 HPN。（<https://svnweb.freebsd.org/changeset/base/294325>）

## OpenSSL 已更新至 1.0.2f 版

新版本的 OpenSSL 已导入树中，解决了 CVE-2016-0701——`DH_check_pub_key` 函数未确保素数适合 Diffie-Hellman（DH）密钥交换，使得远程攻击者通过与选择不当素数的对端多次握手，更容易发现私有 DH 指数。此外，新版本还解决了 CVE-2015-3197——OpenSSL 未阻止使用被禁用的密码，使得中间人攻击者更容易通过对 SSLv2 流量做计算来攻破密码保护机制，相关函数为 `get_client_master_key` 和 `get_client_hello`。（<https://svnweb.freebsd.org/changeset/base/295009>）

## rc_conf_files 可在 rc.conf(5) 中重新定义

这一变更我想会受到系统管理员的热烈欢迎——尤其是那些需要维护成百上千台 FreeBSD 机器的基础设施的人。能够为机器角色创建专门的 rc.conf 文件，并将其引入你的环境而无需编辑 **/etc/rc.conf**，就更容易让初级管理员把这部分知识纳入配置管理系统，而无需学习这些系统所需的晦涩语法或模板。虽然这一变更看似微小，但 FreeBSD 开发者降低入门门槛、让初学者日子更好过的每一步，都是确保 FreeBSD 在各行各业持续获得广泛采用的正确方向上的一步。（<https://svnweb.freebsd.org/changeset/base/295342>）

---

**Steven Kreuzer** 是一名 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车有兴趣。他与妻子、女儿和狗住在纽约皇后区。
