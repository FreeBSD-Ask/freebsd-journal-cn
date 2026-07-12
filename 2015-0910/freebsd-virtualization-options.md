# FreeBSD 虚拟化方案

- 原文：[FreeBSD Virtualization Options](https://freebsdfoundation.org/our-work/journal/browser-based-edition/cloudabi/)
- 作者：**Michael Dexter**

得益于 Docker 与“云”等技术引发的热潮，应用与操作系统的容器化正受到前所未有的关注。

自 Bill Joy 于 1981 年“加固” `chroot(8)` 命令以来，BSD Unix 便提供了引人注目的容器化平台。从 2000 年的 `jail(8)` 到 2014 年的 `bhyve(8)`，FreeBSD 一直在悄然引入优雅而强大的容器化与虚拟化技术，充分利用 FreeBSD 天然的容器化环境，包括其源码导向、Ports 系统、NanoBSD 等精简工具、`src.conf` 与 `KERNCONF`，还有无出其右的 OpenZFS 企业级文件系统。

## 容器化与安全

正如 `chroot(8)` 在 1979 年所证明的、Docker 在 2013 年再次提醒我们的，安全往往是容器化的事后补丁。这两项技术在构思时都未考虑安全，随后才陆续添加各种保护。CSRG 成员 Bill Joy 只是防止进程逃出 `chroot(8)` 环境，而 FreeBSD 开发者 Poul-Henning Kamp 在 2000 年推出的 `jail(8)` 工具将此工作提升到新高度，使标准 FreeBSD 操作系统能在“Jail”容器中启动。其结果是可在单一 FreeBSD 主机上运行数十乃至数百个容器化的 FreeBSD 实例的轻量级方案。在哥本哈根的 EuroBSDcon 上，我问 Poul-Henning，Jail 数量是否有理论上的上限。他说他用脚本不断创建最小 Jail 时，“到 64,000 个就放弃了”。凭借 15 年成熟的生产环境使用经验，FreeBSD `jail(8)` 既为全球运维者省钱，也助其赚钱，并启发了 Solaris 与 GNU/Linux 等其他操作系统上的容器化系统。通过将容器化制度化，`jail(8)` 为 Capsicum（FreeBSD 能力框架）奠定了基础，并正成为 FreeBSD 操作系统各组件首选的沙箱。

在所有目光聚焦 Docker 容器环境的当下，FreeBSD `jail(8)` 自然契合 Docker 新的可插拔容器架构，配合 OpenZFS 文件系统的快照与克隆能力。Docker 瞬时、可抛弃的应用镜像模型，要求在文件系统层面制度化地快照与克隆，而 OpenZFS 几乎是最佳选择。然而讽刺的是，Docker 解决的是应用配置一致可预测的问题，而 FreeBSD 凭借其源码导向与各类软件配置工具，传统上从未受此困扰。FreeBSD 关于软件的“逻辑”始于：整个 FreeBSD 以自托管源码形式位于 **/usr/src/**，第三方应用补充在 **/usr/ports/** 目录。这两处软件位置为运维者提供的“鸟瞰图”是在 FreeBSD 上有效管理软件的第一步。在此之上，运维者可逐个配置软件“Port”，并通过 `src.conf` 与 `KERNCONF` 内核及用户态配置文件定制整个操作系统。GNU/Linux 发行版是从众多来源累积的软件集合，而 FreeBSD 是完整的操作系统，可轻松裁减。要在构建操作系统时不带编译器或域名服务器，只需在 `src.conf` 中禁用相应组件。不需要的硬件驱动同样可在 `KERNCONF` 内核配置文件中轻松排除。综合而言，可精细化配置 FreeBSD 用户态、内核与第三方软件的能力，使 FreeBSD 成为天然的容器化平台。

## 从应用容器化到操作系统容器化

在 FreeBSD 上容器化应用，前提是它们由 FreeBSD 原生二进制构成。这一前提受到挑战：FreeBSD 内核级 Linux 兼容层在 2015 年经历了多项改进，包括执行 64 位 Linux 二进制的能力。此功能对 Linux `jail(8)` 环境很有用，也是运行原生 Linux Docker 镜像的关键。虽然 Linux 兼容性适用于许多场景，但无法满足支持其他操作系统及其原生软件的全部需求。虚拟化便在此处发挥作用，FreeBSD 在过去两年在此领域取得长足进展。首先，得益于 Tarsnap 的 Colin Percival 的工作，FreeBSD 作为客户机操作系统在 Amazon 基于 Xen 的 EC2 平台上得到良好支持。然而 FreeBSD 与 Xen hypervisor 今年早些时候迎来最大突破：FreeBSD 获得了作为 Xen Dom0 控制域主机的能力。这一进展使 FreeBSD 能够汲取成熟的 Xen 社区的丰富资源，并支持大多数（即便不是全部）DomU 虚拟机域。与多数 Xen 实现不同，FreeBSD 着眼未来，仅支持硬件辅助的 HVM 域。此举将执行客户机虚拟机的大部分负担转移至现代 CPU 中的虚拟化特性上。结果是裸机级的虚拟机性能，并附带 OpenZFS 提供的灵活存储基础设施。

## 硬件辅助的未来

若我们将一台硬件辅助的 FreeBSD Xen Dom0 主机配合基于 OpenZFS 的存储架构视为虚拟化未来的一瞥，那么此未来实际上已随 bhyve hypervisor 抵达 FreeBSD。bhyve（BSD Hypervisor）在 FreeBSD 10-RELEASE 中正式登场，并逐步增加功能，如对 AMD 处理器与 32 位虚拟机的支持。与 Xen 和 Linux KVM 不同，bhyve 放弃了 BIOS、视频与网络设备仿真，转而采用用户态引导加载器与虚拟化原生的 VirtIO 驱动模型。与 FreeBSD 上的 Xen 一样，bhyve 完全依赖近期 CPU 中的现代虚拟化扩展，带来裸机级的虚拟机性能。目前，bhyve 支持 FreeBSD、OpenBSD、NetBSD 与 GNU/Linux 虚拟机，最近的 EuroBSDcon 会议显示还支持 Microsoft Windows 与 Joyent SmartOS。

FreeBSD 长期以来在 Unix 应用与操作系统容器化中扮演着关键却低调的角色，如今已蓄势待发，为各种规模的组织提供引人注目的虚拟化方案。

**作者简介**

Michael Dexter 自 1991 年 1 月起使用 BSD Unix 系统，在 Gainframe 提供 BSD 与 ZFS 支持。十多年来，他通过下载镜像、项目、活动与组织支持 BSD Unix。他与妻子、女儿和儿子居住在俄勒冈州波特兰。
