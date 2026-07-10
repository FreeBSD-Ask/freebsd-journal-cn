# 将 Jail 管理从 Warden 迁移到 iocage

- 原文：[Migrating Jail Management from Warden to iocage](https://freebsdfoundation.org/our-work/journal/browser-based-edition/migrating-jail-management-from-warden-to-iocage/)
- 作者：**Kris Moore**、**Brandon Schneider**

随着 PC-BSD/TrueOS 11.0 将于 2016 年发布，我们已经开始迁移到一些随系统捆绑的新工具和实用程序。其中几个主要变化包括：回归 FreeBSD 自带的引导加载器（弃用 GRUB）、转向以 Lumina Desktop 作为主环境、从 Warden Jail 管理工具切换到 iocage。本文将深入剖析 iocage，对比它与 Warden 的差异，并解释迁移的原因。

## 为何选择 iocage？

过去六年里，PC-BSD 一直内置自研的 Jail 管理工具“Warden”。该工具用 shell 实现，除 FreeBSD 基本系统外几乎没有其他依赖。它提供了易用的界面（可选基于 Qt 的 GUI），用于创建和执行 Jail 的基础管理。然而自其最初诞生以来，Jail 管理在 FreeBSD 本身、其他管理工具中都在持续演进。“base-jails”这类概念作为同时更新多个 Jail 底层 FreeBSD 基本系统的机制而流行起来，而 ZFS 凭借其独有的 Jail 管理能力，迅速成为首选文件系统。

Warden 工具虽然最终加入了 ZFS 支持，但其在设计上仍围绕 UFS 作为底层文件系统。这一点在内部代码中体现得淋漓尽致，几乎涵盖每一项主要功能，开始阻碍进一步开发。这尤为棘手，因为 PC-BSD 在 2013 年已转为“ZFS-only”系统，并正在把大量工具链改造成理解和利用 ZFS 特性的方式。到 2014 年冬，要么重写 Warden，要么用更现代的工具替代它，已势在必行。

我们开始考察市场时，有多种 Jail 管理系统可供评估：ezjail、qjail、cbsd 等。然而当我们审视 iocage 时，很快就发现它几乎具备我们在下一代 Jail 管理器中所期望的全部特性和设计细节。首先，与 Warden 一样，它完全用 shell 编写，没有带来额外复杂性的外部依赖，并采用了与之十分相似的“Warden 式”命令行语法。除了原生 shell 实现之外，它从一开始就是为仅在 ZFS 上运行而设计的。从利用 ZFS 属性来配置 Jail，到借助快照、克隆、其他原生 ZFS 特性，iocage 在 ZFS 功能上早已远超 Warden。此外，iocage 还支持由 ezjail 推广开来的“base-jail”概念，让我们在单一 Jail 管理工具中集各项特性之大成。

鉴于决定在 2016 年即将发布的 PC-BSD 11.0 中切换到 iocage，我们已为终端用户启动了相关流程。在 10.2 版本（2015 年夏）中，iocage 工具与旧版 Warden 一并发布。此举旨在让用户和开发者在正式切换之前有一段时间来试用新工具。今秋将提供迁移工具，可自动将现有 Warden Jail 迁移到 iocage，并将纳入 11.0-RELEASE。

由于 iocage 是命令行工具，我们也在开发新的 GUI 来取代原有的 Warden Qt 界面。新开发的 GUI 是基于 Web 的，作为 AppCafe 项目的一部分，目标是既适用于 PC-BSD 这类桌面环境，也适用于 FreeNAS 或 TrueOS 这类无显示器的服务器环境。新 AppCafe 界面目前正处于密集开发中，计划随 2016 年的 FreeNAS 10 和 PC-BSD 11.0 发布。除了对系统包管理（通过 pkgng）的支持，新 AppCafe 还会以几种独到的方式专门使用 iocage 来处理 Jail。

在更传统的 Jail 管理角色中，AppCafe 的 Web 界面将直接作为 iocage 的前端，提供对创建、删除和执行 Jail 基础管理的简便控制。Brandon Schneider（iocage 共同开发者，与原作者 Peter Toth 并肩）于 2015 年加入 iXsystems 后，iocage 新增了一些特性，让 Jail 的创建与分发比以往轻松许多。新的 Jail 分发机制使用 VCS 工具 git 来初次检出一个预构建好、可直接运行的 Jail 环境。这让内容创作者可以用 pkgng 或其他方式构建 Jail，然后把改动提交并推送到公共 git 服务器（例如 GitHub）。随后客户端只需运行一条 iocage 命令，就能把这个 Jail 仓库拉到本地并开始运行。这种新模式将成为 AppCafe“App Cages”特性的基础，让我们能以开箱即用的方式打包诸如 Plex Media Server 之类的应用。此外，它还提供了独到的方式来校验更新，并通过 git 直观查看变更的差异、日志。这些特性已处于可用状态，并包含在 PC-BSD 11.0-CURRENT 分支版本中，让开发者和用户对 11.0-RELEASE 即将带来的内容有了早期预览。

## iocage 入门 / 内部机制

简要介绍 iocage 的工作原理。首先，我们用 ZFS 属性来配置一切。我们没有配置文件，唯一的例外是在 **rc.conf** 中启用服务。iocage 的大部分命名与 ZFS 保持一致。当某项功能 ZFS 和 iocage 都支持时，我们尽量沿用相同名称。对其余命令，我们选择自认为最直观易懂的命名。Jail 命名采用随机生成的 UUID，以避免任何命名冲突。

让我们开始上手这款工具！如果你有多个 zpool，iocage 会选择它找到的第一个。因此我们通常先用 `iocage activate POOL` 来指定。我们使用所谓的“bases”，这些 base 是你用 iocage 获取的 RELEASE，构成了 basejail 的基础。本例中我们使用 10.2-RELEASE。要获取它，执行 `iocage fetch release=10.2-RELEASE`，然后让 iocage 自行完成工作。完成后，我们就得到了可供使用的新 base。虽然获取与创建时指定 FreeBSD 版本所用的术语有所不同，但这是因为一旦某个 RELEASE 被获取，它在 iocage 中的含义就已改变。

至此，我们已选定供 iocage 使用的池，并获取了 RELEASE。剩下要做的就是创建 Jail。与 ZFS 一样，我们允许用户指定希望在创建时设置的属性。本例中我们将使用静态 IP 并为 Jail 指定一个名称，我们称之为“tag”。开始！输入 `iocage create tag="example" ip4_addr="DEFAULT|192.168.1.100/24" base="10.2-RELEASE"`。这会创建名为“example”的 Jail，并给它分配 IPv4 地址 192.168.1.100。`DEFAULT` 是特殊关键字，告诉 iocage 自行判断要使用的默认接口。本例中我们指定了 base，但 iocage 默认会采用主机当前运行的版本。该 Jail 将使用所谓的“shared networking mode”（共享网络模式）。另一种可用的类型是 VIMAGE，它是一种非常灵活的虚拟网络协议栈。两种模式均支持 IPv6。

Jail 已创建，接下来启动它。执行 `iocage start example`，Jail 就会立即启动。我们可以用 `iocage list` 或 `iocage get state example` 验证它是否在运行，输出如下：

```sh
~% iocage list
JID  UUID                                                           BOOT  STATE   TAG         TYPE
1      89b2f41a-76b2-11e5-8df9-d05099728dbf  off       up      example  basejail
```

Jail 已启动——是时候装个 pkg 了！只需运行 `iocage console example`，我们就能直接登录到 Jail 中，像操作一台物理机那样与它交互。本例中，这意味着安装 pkg。于是我们可以执行 `pkg install tmux`，不久后 tmux 就装进了 Jail 里。就是这么简单。iocage 支持大量高级使用场景、其他各类操作。对此，我鼓励你阅读我们的 manpage 并访问文档：<https://iocage.readthedocs.org/en/latest/index.html>。

以上就是 iocage 使用入门。作为一款工具，我们力求非常易用，即便你从未用过 Jail 也能上手。当然，具备一些 Jail 的前置知识会有所助益。如有任何疑问，可在我们的 Google Group 上获得解答：<https://groups.google.com/forum/#! forum/iocage>。

Jail 启动后，那些目录就会被挂载，对用户而言使用方式完全一致。

我提到了 tag，却还没说明它对用户意味着什么。tag 允许你为 Jail 命名，这样在使用 iocage 与之交互时就无需记住 UUID。当然，如果你更愿意使用 UUID，也完全没问题！这意味着你可以仅凭 tag 执行每一项操作。tag 非常方便，我建议使用。如果创建时未提供 tag，那么 tag 将是 Jail 创建时的日期。这一功能在你希望在 Jail 启动前把一些文件复制进去、或在它停止后移除一些文件时同样适用。例如，我可以把目录切换到 **/iocage/tags/example/_**，而非 **/iocage/jails/89b2f41a-76b2-11e5-8df9-d05099728dbf/_**。这省下大量输入！

## 深入剖析

接下来深入探讨 iocage 近期的工作——对 basejail 的彻底重写。我们称之为“basejail”，是因为它们共享公共 base，让用户更新起来更快更轻松，同时还带来可观的存储空间节省。在最新的开发版本中，basejail 已成为我们的默认 Jail 类型。

我们的 basejail 结构相当直观。在 iocage 中，Jail 位于 **/iocage/jail/UUID/root**。UUID 会被你创建 Jail 时所生成的 UUID 替换。本例中我的是 `89b2f41a-76b2-11e5-8df9-d05099728dbf`，每个 UUID 都是唯一的。由于这些是只读挂载，用户无法直接修改其中的数据。我们借助出色的 unionfs 文件系统以覆盖层方式挂载它们，允许用户添加文件、修改既有文件，甚至删除文件，而完全不触碰只读层——unionfs 简直有如魔法。实现方式是把读写挂载放在名为 `_` 的目录中。下面是 basejail 所用目录列表及其挂载位置：

```sh
_/etc        ¦ /iocage/jails/89b2f41a-76b2-11e5-8df9-d05099728dbf/root/etc
_/root       ¦ /iocage/jails/89b2f41a-76b2-11e5-8df9-d05099728dbf/root/root
_/usr/home   ¦ /iocage/jails/89b2f41a-76b2-11e5-8df9-d05099728dbf/root/usr/home
_/usr/local  ¦ /iocage/jails/89b2f41a-76b2-11e5-8df9-d05099728dbf/root/usr/local
_/usr/ports  ¦ /iocage/jails/89b2f41a-76b2-11e5-8df9-d05099728dbf/root/usr/ports
_/var        ¦ /iocage/jails/89b2f41a-76b2-11e5-8df9-d05099728dbf/root/var
```

## 我们如何使用 ZFS

关于 iocage 最后要谈的，是我们实际如何使用 ZFS。我们的每一个 Jail 都有它自己的 ZFS dataset。这让你能将 Jail 取出、移到另一台系统，而当你再次使用时，iocage 会知道该怎么做。因此，当你用 `iocage set prop=value` 为 Jail 设置属性时，我们实际上是在修改该 Jail dataset 上的 ZFS 属性。这意味着每个 Jail 的所有设置都会随它一同迁移，即便你在一台全新的 iocage 安装上挂载了另一台机器曾用于 iocage 的磁盘，也是如此。当你在主机之间迁移 zpool 却不想重新配置时，这非常便利。

我们还支持对 Jail 进行快照。调用 `iocage snapshot` 时可以指定快照名称。也就是说，你可以执行 `iocage snapshot -r example@before`，递归地为该 Jail 创建快照。这让你能将 Jail 回滚到创建该快照时的任意时间点。其好处在于，当你想要尝试新东西、又不想丢失此前的状态时，可以从容试验。你还可以挂载这些快照并探索它们，这往往是无价之宝。

iocage 尽可能利用 ZFS 的全部能力，因为它本身就是一个非常强大的文件系统。这意味着你甚至可以克隆 Jail 或基于它们制作模板。所有这些操作都只占用差异空间，因此你可以放心试验而无需担忧。模板是让你能按自己喜好配置 Jail，再基于该模板生成更多 Jail 的特性。不久这将变得更简单，因为我们将在工具中引入批量创建 Jail 的功能。这将让你拥有统一的命名方案、定制的 Jail base，并能为它们编号。

希望本文已让你对 iocage 能提供什么有了清晰的认识，并激发你进一步了解的兴趣。iocage 还能做许多其他有趣的事，我们诚邀你亲自尝试！

<https://github.com/iocage/iocage>

Version 2.0 为 iocage 带来许多令人兴奋的新内容。即将到来的部分亮点包括：

- 插件
- 全新的 basejail 实现
- 对快照、导入/导出、克隆、模板等许多内容进行重写
- 轻松挂载 ports、src、linprocfs 的能力
- 批量创建 Jail
- 还有更多！

---

**作者简介**

**KRIS MOORE** 是 PC-BSD 项目的创始人和首席开发者，也是广受欢迎的 BSD Now（bsdnow.tv）视频播客的联合主持人。不在家编程时，他周游世界，在各类 Linux、BSD 大会上发表演讲并举办 BSD 相关主题的教程。他目前与妻子和五个孩子居住在美国田纳西州，业余时间（极其有限）喜欢打电子游戏。

**BRANDON SCHNEIDER** 是 iocage 项目的两位开发者之一，目前居住在美国明尼苏达州。业余时间，他喜欢玩电子游戏、谈论游戏、任何与技术相关的话题。可在 Twitter 上通过 @bschneider0922 联系他。
