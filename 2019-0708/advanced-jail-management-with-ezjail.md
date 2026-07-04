# 使用 Ezjail 进行高级 Jail 管理

作者：Andrew Fengler

Jail 是现代系统管理员的强大工具。它们允许轻量级容器化，让你能轻松隔离服务，保持物理宿主机整洁。

在 ScaleEngine，我们用 ezjail 管理 Jail，因为它简单，容易接入我们的配置自动化系统，并且经受过实战检验。当我们需要让 Jail 执行更高级的任务时，它也不会碍事。

有人把 ezjail 描述为“一堆老化的 shell 脚本”。这个说法属实且准确。然而，一堆 shell 脚本虽然不那么精致，但简单、易于理解和调试、没有依赖，也很少有奇怪的边缘情况。它也比任何其他方案都成熟得多，当你重视稳定性时，这总是稳妥的选择。

Ezjail 是两个 shell 脚本：用于与 Jail 交互的 `ezjail-admin`，以及负责启动和停止 Jail 的 `ezjail-admin` RC 脚本。有一个单独的文件 **/usr/local/etc/ezjail.conf** 控制一些默认设置，每个 Jail 还有一个配置文件 **/usr/local/etc/ezjail/jailname**。

## Jail 的基本配置

首先，我们需要安装 ezjail。可以从 Ports 或软件包安装，即 `sysutils/ezjail`。

ezjail 的默认配置相当合理；要得到可用的设置，你大约只需启用 ZFS，它默认是关闭的。把以下内容放进你的 **/usr/local/etc/ezjail.conf**：

```sh
ezjail_use_zfs="YES"
ezjail_use_zfs_for_jails="YES"
ezjail_jailzfs="dozer/jails"
```

这会让 ezjail 在 **dozer/jails** 下为每个 Jail 创建一个新数据集。现在，我们可以让 ezjail 为我们安装 basejail 供使用：

```sh
# ezjail-admin install -m
```

注意：`-m` 标志会在 Jail 里安装手册页，因为没有什么比查不到 `ln` 的参数顺序更让人抓狂的了。

这会为 basejail 和 newjail 创建数据集。basejail 数据集通过 nullfs 挂载到每个 Jail 里，提供基础系统，并允许通过简单替换 basejail 的内容来轻松更新。newjail 数据集会复制到每个新建的 Jail 里，以提供完整可用的系统。有了这些，我们可以创建第一个 Jail：

```sh
# ezjail-admin create myjail.example.com 10.0.0.1
```

这条命令会创建一个名为 myjail.example.com 的 Jail，IP 地址为 **10.0.0.1**。你需要确保地址 **10.0.0.1** 已经绑定到某个接口。如果你想让 ezjail 自动绑定地址，可以用管道符（`|`）分隔，把接口和地址一起指定：

```sh
# ezjail-admin create myjail.example.com 'mlxen0|10.0.0.1'
```

我喜欢按主机名命名 Jail，但你也可以用任何喜欢的名字。注意，ezjail 会在 Jail 配置文件和其他几处使用 Jail 名字的地方，把句点 `.` 和大多数其他特殊字符替换为下划线 `_`。

然后我们想启动 Jail：

```sh
# ezjail-admin start myjail.example.com
```

现在我们可以用 `ezjail-admin console` 命令在 Jail 里获取 shell：

```sh
# ezjail-admin console myjail.example.com
```

大功告成。你可以用 `ezjail-admin list` 命令查看 Jail 列表，分别用 `start`、`stop` 和 `restart` 子命令启动、停止和重启它们。我们还可以用 `ezjail-admin config` 控制 Jail 是否设为运行：

```sh
# ezjail-admin config -r {run|norun}
```

当你把 Jail 设为 `norun` 时，这个工具用精妙的机制阻止 Jail 启动：把 Jail 的配置文件重命名，加上 `.norun` 扩展名。这也会阻止你手动启动 Jail，除非你在子命令前加上 `one` 前缀，即：

```sh
# ezjail-admin onestart myjail.example.com
```

### Jail 的进阶配置

Jail 新手最常搜索的问题之一是“为什么我无法从 Jail 里 ping？”Ping 需要使用原始套接字，出于安全考虑默认是禁用的。我们通常应该保持禁用，但有时你需要它，不管是调试，还是像 Nagios 这样需要能 ping 的程序。Jail 有参数 `allow.raw_sockets`，默认设为 0。我们可以用 Jail 配置文件中名副其实的 `parameters` 选项，让 ezjail 为我们的 Jail 设置参数。

Jail 的 ezjail 配置文件只是一个 shell 脚本，启动脚本在启动 Jail 时会包含它。所以，我们所有的 Jail 设置就是 **/usr/local/etc/ezjail/myjail.example.com** 里类似下面这样的行：

```sh
export jail_myjail_example_com_parameters="allow.raw_sockets=1"
```

注意 `.` 到 `_` 的转换。这些参数在 Jail 启动时设置，所以如果它已经在运行，我们需要重启才能生效。

Jail 常见情形是：在一台有公网 IP 的服务器上，你可能有些程序需要互联网访问，但你要么 IP 地址有限，要么不想让这个程序暴露在公网的“温柔关怀”下。这并不难，除非你的服务器路由器不做 NAT——许多独立服务器和托管服务商就是这种情况。

路由器不做 NAT，任何使用私有 IP 的 Jail 都无法向外连接。我们可以通过在某个地方运行自己的 NAT 来解决，但我们大概不想给整台服务器做 NAT。

我们可以为 Jail 更改 FIB，也就是路由表。尝试之前，确保你已在 **/boot/loader.conf** 里把 `net.fibs` 设为大于 1 的数字。在你的 Jail 配置文件里，设置：

```sh
export jail_myjail_example_com_fib="2"
```

如果我们把它设为 2，那么（重）启动 Jail 后，Jail 就会使用 FIB 2。我们可以为 FIB 2 配置 Jail 所需的任何特殊路由，而不影响宿主机。

ZFS 和 Jail 的另一个强大功能是把数据集委托给 Jail。通过把数据集委托进 Jail，Jail 里的 root 用户就能创建和销毁子数据集、调整数据集属性、执行复制等等。这意味着我们可以创建存储设置，与宿主操作系统隔离，这样即便拥有提权权限的失控脚本也不会破坏宿主机。

如果我们把数据集的 `jailed` 参数设为 `on`，作为安全措施，宿主机将无法再挂载或管理该数据集及其任何子数据集。设置好那个参数后，我们就可以通过设置以下选项，让 ezjail 把它委托给 Jail：

```sh
export jail_myjail_example_com_zfs_datasets="dozer/customerfiles"
```

现在重启 Jail 后，我们假想的客户就能管理这个数据集，利用 ZFS 的全部功能，而不会搞乱我们的宿主机。

Jail 是每个系统管理员工具箱里都该有的出色工具，不管是出于安全、隔离，还是仅仅因为它能让你的系统保持整洁。 •

---

ANDREW FENGLER 是 ScaleEngine Inc.（一家视频 CDN）的系统管理员。他负责管理一支横跨全球的服务器队伍，其中大部分运行 FreeBSD。

---

Jail 是 FreeBSD 最传奇的功能：以强大著称，难以精通，并笼罩在数十年的可疑传说中。

《FreeBSD Mastery: Jails》拨开繁杂，揭示 Jail 的内部机制，在你的服务中释放它的力量。

- 从容地在 Jail 限制内工作
- 实现 Jail 功能的细粒度控制
- 构建虚拟网络
- 部署分层 Jail
- 约束 Jail 资源使用
- ……以及更多！

束缚你的软件！

- 理解 Jail 如何实现轻量级虚拟化
- 理解基础系统的 Jail 工具和 iocage 工具包
- 优化配置硬件
- 从宿主机和 Jail 内部管理 Jail
- 优化磁盘空间以支持数千个 Jail

各地书店有售

《FreeBSD Mastery: Jails》MICHAEL W LUCAS 著
