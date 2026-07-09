# Jail 迁移

作者：Benedict Reuschling

系统中时不时需要把东西挪来挪去。通常是为了给其他东西腾地方，或者新位置更合适。简单的文件和目录如此，ZFS 数据集如此，有时整个安装也是如此。我的情况是 jail，最初作为测试系统起步，我很喜欢它的安装，不想在原来那台宿主机改作他用时从头再搭一遍。幸好我用 iocage 作为管理框架搭建了这个 jail。我知道它支持迁移，但从未用过。所以这正好是个机会，把我的学习整理成一篇文章。

## 迁移

### 冷迁移和热迁移

迁移分两种：冷迁移和热迁移。冷迁移时，系统或服务在迁移期间通常关停，之后再启动。热迁移不需要这样，能继续提供原本提供的所有服务和功能。通常通过共享介质，甚至巧妙的网络技巧，让已建立的网络连接继续运行来实现。有些变体中，热迁移发生得极快，用户察觉不到它真的短暂关停过又在新位置启动。其他方法如热备系统在一台主机迁移时接管，营造服务持续可用的假象。这类故障转移场景需要规划，必须（理想情况下一开始就）以此方式搭建，通常不简单。

冷迁移完成、放到新位置后，jail 可重新启动。这过程听起来简单，但有陷阱。多数陷阱在于新环境会有不同网络。移动后需要考虑新的接口名和/或 IP 地址。

我的情况是单个 jail 跑了些不需要传说中 99.999% 正常运行时间的服务。少量停机可接受，机器上数据也不大，停机时间会很短。iocage 通过为构成 jail 的 ZFS 数据集创建归档以及用于校验完整性的校验和来完成冷迁移。如前所述，jail 需要关停，才能得到没有打开套接字或进程仍在 jail 主内存中运行的一致归档。

### 导出

在 iocage 中导出 jail，用 UUID 标识 jail，运行：

```sh
# iocage list
+-----+--------+-------+--------------+-----------+
| JID |  NAME  | STATE |   RELEASE    |    IP4    |
+=====+========+=======+==============+===========+
| 1   | icinga | up    | 12.0-RELEASE | 10.0.0.15 |
+-----+--------+-------+--------------+-----------+
```

这是我的 icinga 监控 jail，可用 NAME 列提供的名称控制。要导出该 jail，先要关停它。用 `iocage stop <UUID>` 命令：

```sh
# iocage stop icinga
* Stopping icinga
+ Executing prestop OK
+ Stopping services OK
+ Tearing down VNET OK
+ Removing devfs_ruleset: 5 OK
+ Removing jail process OK
+ Executing poststop OK
```

再次运行 `iocage list` 确认 jail 已停止：

```sh
+-----+--------+-------+--------------+-----------+
| JID |  NAME  | STATE |   RELEASE    |    IP4    |
+=====+========+=======+==============+===========+
| -   | icinga | down  | 12.0-RELEASE | 10.0.0.15 |
+-----+--------+-------+--------------+-----------+
```

iocage 安装时，在专用 pool 上创建了名为 `images` 的数据集，以及其他数据集。运行迁移命令时，构成 jail 的归档和校验和会放到 `images` 目录中：

```sh
# iocage export icinga
Exporting dataset: mypool/iocage/jails/icinga
Exporting dataset: mypool/iocage/jails/icinga/root

Preparing compressed file: /mypool/iocage/images/icinga_2019-10-10.zip.
```

根据 jail 中数据多少，导出过程可能需要一些时间完成。校验和也是如此。从输出可见，生成的 zip 文件以 jail 名加上导出日期命名。

将 zip 文件复制到新主机。复制完成后，确保校验和仍与导出步骤生成的校验和文件匹配。验证校验和的简单方法：

```sh
# sha256 -r icinga_2019-10-10.zip
# cat icinga_2019-10-10.zip
```

这样两个校验和上下排列，便于比较。接下来在新主机上安装 iocage（如果尚未安装）。激活 pool（用 `iocage activate`）后，用 `iocage fetch` 创建目录结构。结构存在后，zip 文件必须放到 `images` 目录。然后 iocage 在运行导入命令时会取用它：

```sh
# iocage import icinga
Importing dataset: icinga
Importing dataset: icinga/root
Imported: icinga
```

jail 现在会出现在 `iocage list` 输出中：

```sh
# iocage list
+-----+--------+-------+--------------+-----------+
| JID |  NAME  | STATE |   RELEASE    |    IP4    |
+=====+========+=======+==============+===========+
| -   | icinga | down  | 12.0-RELEASE | 10.0.0.15 |
+-----+--------+-------+--------------+-----------+
```

根据新目标地的 jail 设置，必须设置新的 IP 地址才能像之前一样让 jail 联网。默认情况下，迁移保留所有设置，包括旧 IP 地址，这在新位置可能正确也可能不正确。设置新地址可能简单到用 `iocage set ip4_addr` 设置新 IP。iocage 还支持共享 IP 的 jail 和 VNET jail。详见 iocage 手册页。我也推荐 Michael W Lucas 的《FreeBSD Mastery: Jails》一书，书中用可直接使用的示例详细介绍了 jail 的多种使用场景。

我的情况是把 IP 地址改到完全不同的网络。

再次运行 `iocage list` 确认设置已应用：

```sh
# iocage list
+-----+--------+-------+--------------+---------------+
| JID |  NAME  | STATE |   RELEASE    |      IP4      |
+=====+========+=======+==============+===============+
| -   | icinga | down  | 12.0-RELEASE | 192.168.1.128 |
+-----+--------+-------+--------------+---------------+
```

### 启动 Jail

接着可用 `iocage start` 启动 jail。确保旧位置的 jail 仍处于 down 状态，否则两者会争夺 IP 地址控制权。jail 启动后，确保其中运行的服务监听新地址。通常包括修改相应配置文件然后重启服务以应用。一旦对结果满意、jail 在新位置运行，旧 jail 可归档（备份）或直接删除。这腾出旧主机上的空间。别忘了 **iocage/images** 目录中的导出文件。

Jail 迁移没有听起来那么吓人。稍作准备、留出传输生成文件的时间，它就是可行的方案，无需从头重新搭建 jail。 •

---

**Benedict Reuschling** 2009 年加入 FreeBSD 项目。2010 年获得完整文档提交权限后，他积极指导其他人成为 FreeBSD committer。2015 年加入 FreeBSD 基金会，现任副主席。Benedict 拥有计算机科学硕士学位，在德国达姆施塔特应用技术大学教授”面向软件开发者的 UNIX”课程。他与 Allan Jude 共同主持每周一期的 BSDNow.tv（`http://BSDNow.tv`）播客。

---

## FreeBSD Mastery: Jails

FreeBSD Mastery：Jails 切开繁杂，揭示 jail 内部机制，在你的服务中释放它们的力量。

Jail 是 FreeBSD 最传奇的特性：

- 以强大著称，
- 难以驾驭，
- 数十年来笼罩在可疑传说中。

舒适地在 jail 限制内工作；对 jail 特性实现细粒度控制；构建虚拟网络；部署分层 jail；约束 jail 资源使用；以及更多更多！

各地书店有售。

囚禁你的软件！

- 理解 jail 如何实现轻量级虚拟化
- 理解基本系统的 jail 工具和 iocage 工具包
- 优化硬件配置
- 从宿主机和 jail 内部管理 jail
- 优化磁盘空间使用以支持数千个 jail
- 囚禁你的软件！

作者：MICHAEL W LUCAS
