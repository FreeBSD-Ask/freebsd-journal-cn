# OpenRC：专访 Joe Maloney

作者：Benedict Reuschling

请简单介绍一下你自己，以及你是如何接触 OpenRC 的。

- 我叫 Joe Maloney，是 TrueOS 项目的首席系统架构师。我的主要职责包括协助把握项目整体方向、提供支持以及问答。从 PC-BSD 改名这件事上，我多少要”背点锅”，推动我们开发节奏的滚动发布模式也是我推动的。把 OpenRC 集成到 TrueOS 项目中，我更是难辞其咎。几年前，我在 vBSDcon 2015 上参加了我的第一次会议，受到启发，开始追求可移植性这一用例。直接替换 init 不会带来我们所希望的渐进式过渡，所以我们最终选择了 OpenRC——它允许我们分阶段演进项目。

在讨论替换 rc 系统之前，能否简要介绍一下 FreeBSD 的启动过程？

- 在 FreeBSD 启动过程中，内核通过 `loader.conf` 加载指定要提前运行的驱动。这意味着驱动作为模块在内核启动前就已加载。内核本身在大约 6 秒内启动。System V init 程序 **/sbin/init** 作为 pid 1 启动，并执行一个名为 **/etc/rc** 的 shell 脚本。

rc 到底是什么？

- **/etc/rc** 文件不过是由 **/sbin/init** 执行的 shell 脚本，用于在启动时执行进程。在 rc.d 出现之前，所有进程都必须硬编码进 **/etc/rc**。这种做法的缺点是：用户编辑 **/etc/rc** 时一旦出错，就会挂死启动过程。使用这种方法时，用户必须自己跟踪服务启动顺序，手动将进程守护化，自然也没有简便的方法来检查服务状态。

rc.d 是什么？

- 为解决这些问题和其他问题，FreeBSD 从 NetBSD 社区引入了 rc.d。rc.d 系统是一组 shell 脚本集合，这些脚本通常会 source 其他 shell 脚本。这让系统在启动时变慢不少——比如 `netif`，包含 source 的部分在内，要在启动时一次性处理超过 4000 行 shell。rc.d 系统允许在每个脚本中指定启动顺序、查询服务状态和守护化。基础默认配置位于 **/etc/defaults/rc.conf**。用户的 rc.d 配置一般在 **/etc/rc.conf** 中完成——在这里启用、禁用服务或设置服务附加标志。

此前对改进 rc.d 做过哪些尝试？

- PC-BSD 项目多年来在测试无线场景下的 DHCP 与 SYNCDHCP。两者区别在于：DHCP 会转入后台运行，让启动过程稍快；SYNCDHCP 则在前台运行。不幸的是，对社区中许多用户来说，不带 SYNCDHCP 标志在前台运行时，`wpa_supplicant` 不稳定；而一旦后台运行，又会挂死启动过程。我们也短暂考察过 rcorder 的补丁。当时有一系列补丁可以为 rc.d 启用并行启动。可惜多年来这些补丁无人维护，随着改动累积，它们再也无法干净地应用，除非花费大量精力。PC-BSD 创始人 Kris Moore 此前还写过补丁，加入 `rc_delay` 函数。它本可让网络或其他延迟启动的服务延后运行。但评审中有人建议改用并行启动，这些提交最终也死于 FreeBSD 评审流程。即便那些工作被接受，也算不上理想方案。比如考虑 ldap 登录的场景——网络必须先起来，延迟网络只会破坏这个用例。

OpenRC 是为什么开发的？起源于哪里？

- OpenRC 是 rc.d 的演进，本身不是 init 系统，而是与系统提供的 init 系统协同工作。OpenRC 是 rc.d 的直接替换吗？差不多是。是的，Ports 中成千上万的 init 脚本正被转换为更简洁、shell 代码更少的格式。OpenRC 主要用 C 编写。使用 OpenRC 的首要目标是用内置 C 函数减少 shell 用量，并简化服务。OpenRC 由 NetBSD 开发者 Roy Marples 编写。Gentoo、PacBSD、UbuntuBSD 等发行版都在使用。OpenRC 最初在 Gentoo 的 alt 分支（使用 BSD 内核）上起步。临近发布时，Gentoo 项目授权 Roy 以 BSD 许可证发布可移植版本。

把 OpenRC 集成到 TrueOS 基本系统中需要做什么？

- 对 TrueOS 而言，我们移除了 Linux 可移植性相关的代码。我们发现这是使用 gmake 的主要前提。要集成到我们的 FreeBSD 分支并用 world 构建，必须做出改动。我们的 OpenRC 实现现在使用 BSD make，并随基本系统一起打包。我们为 **/etc/rc**、**/etc/rc.shutdown**、**/etc/rc.devd** 添加了钩子。我们并没有完全替换这些脚本，只是像上游用 gmake 安装时那样，加上启动 OpenRC 的钩子。我们把许多 rc.d 脚本重写为与 init.d 兼容。**/etc/init.d/** 目录用于存放基本系统的脚本，取代 **/etc/rc.d**。**/etc/defaults/rc.conf** 和 **/etc/rc.conf** 不再启动或停止服务，但仍可用于设置标志。**/etc/conf.d/** 目录取代了 **/etc/conf.d/** 目录——后者并不为广大用户常用。最终在我们分支中的样子如下：

挂接 OpenRC 的修改：

- <https://github.com/trueos/freebsd/blob/drm-next/etc/rc>
- <https://github.com/trueos/freebsd/blob/drm-next/etc/rc.devd>
- <https://github.com/trueos/freebsd/blob/drm-next/etc/rc.shutdown>

仅修改顶层 Makefiles 即可构建这些新目录的新增内容：

- <https://github.com/trueos/freebsd/tree/drm-next/etc/init.d>
- <https://github.com/trueos/freebsd/tree/drm-next/lib/libeinfo>
- <https://github.com/trueos/freebsd/tree/drm-next/libexec/rc>
- <https://github.com/trueos/freebsd/tree/drm-next/bin/rc-status>
- <https://github.com/trueos/freebsd/tree/drm-next/sbin/openrc>
- <https://github.com/trueos/freebsd/tree/drm-next/sbin/rc-service>
- <https://github.com/trueos/freebsd/tree/drm-next/sbin/rc-update>
- <https://github.com/trueos/freebsd/tree/drm-next/sbin/start-stop-daemon>
- <https://github.com/trueos/freebsd/tree/drm-next/sbin/supervise-daemon>

迁移过程中遇到过什么障碍？

- 在迁移过程中，我们清楚认识到：无论是并行启动还是 `rc_delay`，都没真正解决多少问题——尤其是当进程在前台而非后台运行时。我认为 rc.d 本身可以通过现代化许多脚本、尽可能精简代码来改进。证据就是：用一个更小、更简单的 shell 网络脚本，亲眼看到差异。我们的第一次尝试可以在这里看到：<https://github.com/trueos/freebsd/blob/fa10ac7ceb7048baad84fd3455a77edeb118c0a2/etc/init.d/network>

那个脚本支持并行启动，而 `netif` 不支持。FreeBSD 自带的 netif 脚本本身相当精简：<https://github.com/freebsd/freebsd/blob/master/etc/rc.d/netif>

但它 source 的功能函数文件才是性能问题的源头：<https://github.com/freebsd/freebsd/blob/master/etc/network.subr> <https://github.com/freebsd/freebsd/blob/master/etc/rc.subr>

当时还缺少 IP 别名、lagg 等一些边缘场景功能。因此出于整体兼容性考虑，我们决定暂时切回 netif。这在一定程度上拖累了性能，也使并行启动在当前条件下无法正常工作。但我们无法以此为由，剥夺用户使用这些功能的权利。当然，我们可以把 netif 从启动 runlevel 中移除，通过让 devd 在稍后启动网络来获得更好的性能。在 TrueOS 中，我们需要一种机制，在需要时才应用特定配置，而不是只在启动时评估一次。再说一遍，我们又回到了那个老问题——这对那些需要登录管理器启动时就有网络的用户来说，仍没真正解决什么。我们中有些人认为，前进方向可能是写一个网络管理器，能在事件驱动场景下与 devd 更高效地协作。另一个考虑是，我们目前默认使用 `dhcpcd` 作为 DHCP 客户端：<https://github.com/trueos/dhcpcd>

我们尚未完成 NetBSD 那样直接集成到源码树的任务：<http://cvsweb.netbsd.org/bsdweb.cgi/src/external/bsd/dhcpcd/>

不过，这已列入 TrueOS 完全集成 dhcpcd 的路线图，同时也为愿意使用的人提供 base 中的 dhclient 集成。我们发现，在并行化工作中，`dhcpcd` 在这个用例下表现更好，所以我们选它作为默认 DHCP 客户端。未来还得重新审视 dhclient。我们的待办清单总结如下：

- 将 `dhcpcd` 导入 base
- 增加 `dhclient` 支持
- 尝试修复 `netif` 并行化
- 未来可能编写网络管理器，并将其设为默认选项

迁移到 OpenRC 带来了哪些改进？

- 即使没有并行启动，TrueOS 在大多数情况下平均启动时间为 10 到 15 秒，提升了 127%。有人可能会说，他们不用 OpenRC 也能 9 秒启动，但显然他们没有开 TrueOS 开箱即用的众多服务——这些服务是为了给普通用户更通用的使用场景而设的。当然，没有登录管理器，直接进 fluxbox 而不开 cups、avahi 等众多服务，是会快一些。但启动 100 个服务时，使用 rc.d 你会看到明显变慢——这正是并行启动发挥作用的地方。

我们还在 OpenRC 中直接集成了服务监督能力。目前 sysadm 用它来监控服务，并在崩溃时重启。其他监督后端（如 s6）也可以在不替换 init 的情况下使用，从而提供 socket activation 功能。不过内置的监督器在我们的用例中已证明足够好用。

借助 openrc-run 的内置函数，我们的服务文件更简单了。某些服务只需指定名称和命令即可启动。所有服务现在都能正确报告服务状态——之前用 FreeBSD 的 rc.d 时，许多服务状态报告并不正确；迁移到 OpenRC 后，我们发现这些服务的状态报告都修好了。

OpenRC 还为服务启动提供了良好的可视化表示，让失败一目了然，并用颜色代码给出警告。出错的服务显示红色，有警告的服务显示黄色。并行启动时主色调为蓝色，常规启动时主色调为绿色。当然，想要传统的 rc 风格，可以通过 `rc.conf` 中的配置参数关闭颜色。

最后，我们获得了提供 runlevel 的能力。这些 runlevel 与 Linux SysV 的 runlevel 不同，但允许我们通过把服务分组到 runlevel 中来排序。改进总结如下：

- 并行启动能力
- 服务监督能力
- 简化的启动脚本
- 改进的服务状态报告
- 改进的服务启动显示
- runlevel 与堆叠 runlevel

最后，我们最近的版本（包含 OpenRC 26.2）加入了显示受监督服务运行时长的能力，我们现在正在监控服务。

能否提供一个项目托管的 init.d 脚本示例？

- 这里有一个使用新的 supervise-daemon 的示例：<https://github.com/trueos/sysadm/blob/master/src/init.d/sysadm>

sysadm 服务使用 supervise 守护进程，即使服务崩溃也会保持运行。只有用户干净地停止服务，进程才会停止。

那 Ports 树中的集成脚本示例呢？

- Ports 使用以下 overlay 结构：<https://github.com/trueos/freebsd-ports/blob/trueos-master/devel/dbus/Makefile.trueos> <https://github.com/trueos/freebsd-ports/blob/trueos-master/devel/dbus/files/openrc-dbus.in>

上游的原始 rc.d 脚本被保留，仍可在系统中共存：<https://github.com/trueos/freebsd-ports/blob/trueos-master/devel/dbus/files/dbus.in>

把 rc.d 脚本转换为 OpenRC 难吗？

- 大多数情况下，rc.d 脚本可以简单地转换。通常只需删除几行简单代码。常见的”罪魁祸首”：

- `./etc/rc.subr`
- `-rcvar=`
- `-exit_code=0-`
- `-load_rc_config ${name}`

更详细的指南见 TrueOS 博客：<https://www.trueos.org/blog/openrcupdate-simplifying-openrc-scripts/>。我们也强烈推荐使用 `man openrc-run` 查看编写 init.d 兼容脚本的所有可用选项。TrueOS 项目 100% 完成向 Ports 添加 init.d 兼容脚本的工作。

你认为 FreeBSD 也应该考虑迁移到 OpenRC 吗？

- 我把 FreeBSD 视为开发家电设备的优秀载体。我目前认为，TrueOS 也应被视为基于 FreeBSD 的另一种家电设备。要做的事情还很多，但我们希望继续推进 OpenRC，让它在未来成为更有吸引力的选项。当然，TrueOS 项目认为 OpenRC 的许可证模式恰到好处；除了 netif 并行化这一瓶颈和一般性的脚本转换外，已没有太多工作要做。

---

**BENEDICT REUSCHLING** 于 2009 年加入 FreeBSD 项目。2010 年获得完整文档提交权限后，他积极指导其他人成为 FreeBSD 提交者。他是 BSD 认证小组的考官，2015 年加入 FreeBSD 基金会，目前担任副主席。Benedict 拥有计算机科学硕士学位，在德国达姆施塔特应用技术大学教授”面向软件开发者的 Unix”课程。
