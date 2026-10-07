# 用 Sylve 整合家庭实验室

- 原文：[Consolidating a Home Lab with Sylve](https://freebsdfoundation.org/our-work/journal/browser-based-edition/production-deployments/consolidating-a-home-lab-with-sylve/)
- 作者：**Sven Rüdiger**

## 面向 FreeBSD 上 bhyve、Jail 与 ZFS 的浏览器管理平面

多年来，我一直想找个“省事”的办法，在 FreeBSD 服务器上原生运行虚拟机。该有的东西其实都在：bhyve 是出色的 hypervisor，ZFS 自有 ZFS 的本事，FreeBSD 网络栈更不必向谁推销。可真要日复一日把它们凑在一起用，还是得靠我手工维护的 shell 脚本和配置文件。没有任何界面能架在我已有的系统之上，又不试图接管整个系统。

我在 [bhyveCon 2025](https://bhyvecon.org/) 上看了[那场关于 Sylve 的演讲](https://www.youtube.com/watch?v=wo4oD5UON30&pp=ygUVIlN5bHZlIiBiaHl2ZUNvbiAyMDI10gcJCaMLAYcqIYzv)，看完当天就动手试了。当晚，Windows 就作为 bhyve 客户机跑在我的一台开发服务器上了，我还写了一篇[简短的博客文章](https://hackacad.net/post/windows-10-on-freebsd-with-sylve/)记录此事，主要是因为我不太敢相信整个过程竟没出多少岔子。

本文讲的是我如何用 Sylve 收拾已经失控的家庭实验室，它为什么契合我运行 FreeBSD 的既有方式，还有我把最后一台独立机器迁入之后结果如何。文末有一节循序渐进的安装说明。

### 我想解决的问题

我的实验室按大多数家庭实验室的方式长大（只是快了一点），也就是说，没怎么规划。多年来我一直把 Home Assistant 直接跑在 FreeBSD 上。起初一切正常，后来就不行了。大约两年前，Python 依赖在每次升级时都变成一场移植苦战，为了让它保持正常运转，我搭进去的夜晚越来越多，实在不划算。这方面 Home Assistant 并不特殊；不少项目其实只期望 Linux 主机，我手里也悄悄攒了几摊处境相同的负载。

虚拟化是显而易见的出路。我以前摆弄过 bhyve，老实说，问题多半出在我自己身上，而非 hypervisor。手工管理它们很痛苦，vm-bhyve 也只帮上一点忙。我没有硬着头皮上，而是绕了个弯：把 Linux 负载放到 Raspberry Pi CM4 板子上，一块板子干一件事。这办法行得通，但如今我有了小小一架单一用途的机器，名叫 ComputeBlades。后来 CM5 支持迟迟不见踪影，这条路的前景也就不明朗了。

我真正想要的，是把那些 Linux 负载拉回自己熟悉的 FreeBSD 主机上，顺带把 Intel NUC 上那一两个甩不掉的专有 Windows 工具也搬过来，而且不必抛弃多年来学到的一切。Sylve 正好补上了这道缺口。

### Sylve 为何契合以 FreeBSD 为先的环境

Sylve 最讨我喜欢的一点，是它不试图成为平台本身。它驱动的每样东西本就属于 FreeBSD 基本系统。bhyve 担任 hypervisor，ZFS 负责存储，网络栈打理底层管道。Sylve 只管调度这些部件，通过 Go 后端和 REST API 编排虚拟机、Jail、数据集、虚拟交换机与 DHCP。底层的操作系统分毫未动。

这就是不去碰 Proxmox 之类方案的全部理由。Proxmox 是好产品，可用上它就意味着放弃 FreeBSD，连同我每天依赖的各种习惯和肌肉记忆一起放弃。Sylve 让我保住自己的主机。我在意的功能留在基本系统里，那里的东西大家都很熟悉。Sylve 只是坐在上层的管理器，正如 Bastille 之于 Jail，我也只希望它做到这一步。我不需要为了界面就装整套 hypervisor 发行版，ZFS 快照和复制仍然是原汁原味的 ZFS，而不是被埋在厂商抽象层下的东西。

### 初印象与早期开发

我很早就开始用 Sylve，但从未后悔。界面有 bug，也有软件常见的毛糙之处，对年轻、尚在实验阶段的项目来说，这完全在意料之中。让我意外的是，这些问题都没波及虚拟机。哪怕我把 Sylve 装坏了，底下的负载照样在跑。因为 Sylve 管理的是基本系统的部件，而不是替换它们，界面崩掉从不会变成虚拟机停摆。管理层次与实际负载之间的这道间隙，让我决定把它真正用起来，而不是当玩具。此后它进步很大。我刚开始用时要自己从 GitHub 源码构建；如今它已经打包，安装步骤短了许多。

### 整合实践

Sylve 一就位，清理就快了起来。开几台 Linux 客户机不费吹灰之力，我还添了一台 Windows 客户机，用来运行一款没有 FreeBSD 版本的专有安全软件。手工折腾 bhyve 时始终没能养成的工作流程，突然就行得通了，因为现在无非是填个表单：CPU、内存、数据存储、交换机，外加用来看安装程序的控制台。几周前，我把最后一台独立机器也迁进了 Sylve，那架单一用途的机器就此收拢成几台性能不错的 FreeBSD 主机。

有一点值得点出，因为它很好地说明 Sylve 在一套有历史包袱的系统上表现如何。它自带 Jail 管理器，但我用 Bastille 跑二十多个 Jail 已经很长时间。我不想迁移它们，结果发现根本不必迁。两者相安无事地共存。我把现有的 Jail 留在 Bastille 里，让 Sylve 管虚拟机。旧 Jail 待在原本就能工作的地方，就不必为迁移额外花时间。那样做毫无收益，而这样还能让整套配置尽可能简单。随之而来的还有我一贯的做法：依赖越少越好，让每个工具干它真正擅长的那部分。

维护也遵循同样的思路。底下有 pkgbase，又有启动环境可作退路，让平台保持最新很容易。我可以在全新的启动环境里试跑一次升级，一旦出问题就回滚；而且因为 Sylve 并不占有基本系统，更新主机不会演变成跟管理层的拉锯战。

### 结果

简短版本：我从散落一地的单一用途机器、Pi 板子和 NUC，缩减到几台 FreeBSD 服务器。当初逼我绕道 CM4 的那些 Linux 负载，回到了我最熟悉的硬件和操作系统上，Windows 虚拟机也一样。我的 Bastille Jail 原封未动。

还有一点也该坦白说明：Sylve 同样能以集群方式运行。写作本文时，在主机之间迁移虚拟机的功能刚刚发布，用起来限制不多，比如要求两台主机上的 ZFS 池同名。对我的家庭实验室来说，一键迁移已经是很大的助力。

如果你是 FreeBSD 用户，一直在等待一款能与你的系统配合的 bhyve 界面，那 Sylve 大概就是你一直在找的工具。

### 安装 Sylve

#### 系统需求

Sylve 只需要 **FreeBSD 15.0 或更高版本**，再加上可用的 **ZFS 池**。只跑 Jail 的主机，一颗 CPU 加 512 MB 内存就能撑起来；但若要运行 bhyve 虚拟机，至少要有 **2 个 vCPU、4 GB 内存**，ZFS 池也得在 20 GB 以上。还有一点值得知道：许多 VPS 实例不支持嵌套虚拟化，在那里 bhyve 虚拟机恐怕无从谈起。话说回来，何必非要在虚拟机里套一层 hypervisor？Jail 在哪儿都能跑。

#### 安装软件包

Sylve 如今已经打包，因此在 FreeBSD 15 或更新版本上，安装本身就是一条命令：

```sh
pkg install sylve
```

开箱即用，Sylve 使用自带的证书以 HTTPS 启动，因此安装完就能直接访问界面，无须生成任何东西。它也能提供纯 HTTP 服务，但我把它排除在配置之外，只保留 HTTPS。如果你不想用随附的证书，自己又没有 CA，可以通过 Sylve 界面创建自签名证书。

证书是自签名的，所以浏览器第一次会报警告；接受例外，或者把证书导入信任库，就能让它安静下来。对于可从外部访问的主机，直接使用内置的 Let's Encrypt 或任何公共 CA 即可，但要记住，你也许并不想把 hypervisor 的管理界面放到公网上，尤其考虑到这个项目还这么年轻。

#### 创建配置

Sylve 读取 JSON 配置文件。用编辑器改 **/usr/local/etc/sylve** 下的默认 `config.json`，填入基本设置，如果证书和密钥由你自己提供，再加上它们的路径：

```sh
{
"environment": "production",
"proxyToVite": false,
"auth": {
"enablePAM": true
},
"admin": {
"email": "admin@sylve.local",
"password": "your-strong-password"
},
"logLevel": 0,
"port": 8181,
"raft": {
"reset": false
}
}
```

有几点说明。这里故意没有 `httpPort`：加上它，Sylve 就会提供纯 HTTP 服务，但我只保留 HTTPS，所以略去不写。`enablePAM` 为 `true`，这样你可以用系统账户登录——如果只允许管理员用户（以及你稍后在界面里添加的账号）进入，就把它设为 `false`。搭建期间 `logLevel` 用 `0`，会记录一切；环境稳定后改成 `3`，只记录错误。集群至少需要三个节点。把 `raft reset` 设为 `true`，就能轻松重建集群，且不丢失虚拟机。

#### 加载器与内核可调参数

如果打算用 Jail 和 ZFS，就把下面这些加到 **/boot/loader.conf**，好让 ZFS 载入并开启资源统计：

```sh
zfs_load="YES"
kern.racct.enable=1
```

我还会给 ZFS ARC 设个上限，免得它把客户机饿死。在 4 GB 的机器上，512 MB 是合理的起点，但应通过 Sylve 界面按你的负载调整。太小，ZFS 会变慢；太大，它会吃掉内存。

#### 启用并启动 Sylve

软件包已经放好了 rc.d 脚本，剩下的只是启用并启动服务：

```sh
sysrc sylve_enable=YES
service sylve start
```

换作我，这里会直接重启，而不是只启动服务，一来验证加载器可调参数是否生效，二来确认 Sylve 能自行启动。无论哪种方式，它随后都应该在 `https://<your-server-ip>:8181` 上响应。用 `config.json` 里 `admin` 的凭据登录。

#### 创建虚拟交换机

在创建第一台虚拟机之前，得先有虚拟交换机供它接入。在界面里依次进入 Network → Switches → Standard → + New，对多数家庭环境来说，在这里启用 DHCP。你之后创建的虚拟机和 Jail 都把自己的网络挂在这个交换机上。

#### 你的第一台虚拟机：Windows

以下是在 Sylve 上运行 Windows 的简要版本，取自我在自己博客上写的文章。

先从存储入手：在 Datacenter → your host → Storage → ZFS → Datasets → Volumes → + New 下为虚拟机创建 ZFS 卷，大小随你定。然后搞一个 Windows 10 ISO（LTSC 评估版就很好用），通过 Utilities → Downloader → + New 粘贴下载链接把它拉进来。

现在创建虚拟机。在 Storage 标签页选择 ZFS 和你刚创建的卷，磁盘总线为 Windows 10 选 NVMe（Windows 11 目前用 AHCI 更顺）。在 Network 标签页选择你的交换机和 VirtIO 仿真。在 Advanced 标签页，如果你想用外部 VNC 客户端连接控制台，就复制 VNC 密码。创建之后打开 Summary，点 Start。

要进入安装程序，你要么及时抓住启动提示符，要么进入 UEFI shell 自己启动它：

```sh
fs0:
cd EFI\BOOT       # 也可能是 EFI\Microsoft\Boot，取决于 ISO
BOOTX64.EFI       # 也可能是 bootmgfw.efi
```

Windows 起来之后，安装 VirtIO 驱动。同样通过 Downloader 下载，在 Windows 里挂载该 ISO，然后运行安装程序。如果 Windows 看不到磁盘或网卡，几乎总是因为缺少 VirtIO 驱动，在安装时或安装后补上驱动，问题自然解决。

#### 结语

Sylve 帮我把家庭实验室里运行的 FreeBSD 节点从 16 个缩减到 4 个。现在一切都跑在几台 FreeBSD 主机上：Linux 负载、Windows 虚拟机，还有我原有的 Bastille Jail，彼此相邻，互不干扰。

唯一的缺点是功耗——Intel 硬件的空闲功耗比一摞 ARM 板子高。但随着节点数量增长，这一差距会缩小。主机上的带外管理是个不错的附加好处，便于维护节点固件，也能应对故障（即便有启动环境，故障也可能发生）。

接下来的事，主要是看着 Sylve 的集群功能成熟起来，尤其是实时迁移。本文完成时，0.3.0 刚刚发布，带来基本的迁移功能，文档也更完善。<https://sylve.io/>

---

**Sven** 自 2000 年起就是 FreeBSD 的老用户，对开源技术怀有根深蒂固的热情。过去十年间，他一直积极为 FreeBSD Ports 集合贡献代码，最近五年担任 port 维护者。他拥有数据科学硕士学位，喜欢把大数据和数据科学工具引入 FreeBSD 生态。
