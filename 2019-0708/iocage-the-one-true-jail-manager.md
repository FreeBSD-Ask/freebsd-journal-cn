# iocage：真正的 Jail 管理器

作者：**Michael W Lucas**

iocage
真正的 Jail 管理器

本文标题不完全是谎言，但也不是全部真相。

其他 Jail 管理器确实存在。有些在特定场景下用得还不错。但对于中大型 Jail 服务器——你期望单台宿主机上运行几十甚至几百个 Jail——iocage 提升了可管理性。iocage 可以和 docker 叫板。iocage 利用 ZFS。iocage 自动化批量升级 Jail，包括软件包。iocage 配置虚拟网络。iocage 解决你甚至不知道自己有的问题。iocage 总是用小写“i”开头，但在句首看起来很傻，所以我拒绝那样写。

如果你今天在部署 Jail 宿主机：用 iocage。（并且用 FreeBSD 12 或更高版本，但那是另一个话题。）是的，iocage 需要 ZFS 和 Python。如今的 Jail 宿主机应该用 ZFS，而 Python 几乎无处不在。如果你的宿主机磁盘够多，把宿主机操作系统放在冗余 ZFS 池上，把 Jail 放在另一个。锁定宿主机的服务；然后从软件包或 GitHub 安装 iocage。

首先，告诉 iocage 用哪个池来放 Jail。

```sh
# iocage activate jails
```

如果你不指定池，iocage 会把所有 Jail 放在根池上。

## iocage 命令行

所有操作都用 **iocage(8)** 命令完成。语法和 ZFS 极为相似，好像 iocage 的开发者看到好主意就知道是好的。我们先用 `iocage get` 查看 iocage 的默认参数。我们加上标志 `-a` 获取所有参数，并加上 Jail 名 `default` 来查看默认参数。

```sh
# iocage get -a default
CONFIG_VERSION:20
allow_chflags:0
allow_mlock:0
allow_mount:0
…
```

iocage 里的参数用下划线而不是句点分隔，因为 Python。参数 `allow_chflags` 和 jail.conf 的参数 `allow.chflags` 相同。`allow_chflags` 设为 0 时，iocage 默认不设置 `allow.chflags`。iocage 用参数配置 Jail 的一切，有些参数没有 jail.conf 的等价物。参数 `resolver` 告诉 iocage 从哪里获取 Jail 的 resolv.conf。

```ini
…
resolver:/etc/resolv.conf
…
```

Jail 从宿主机复制 **/etc/resolv.conf**。用 `iocage set` 命令更改设置。你可以指定某个 Jail，或者更改默认值让它对今后创建的所有 Jail 生效。

```sh
# iocage set resolver=/etc/resolv.conf.jail default
```

如果默认值看起来合理，创建你的第一个 Jail。

创建 Jail 之前需要先有 FreeBSD 发行版。用 `iocage fetch` 命令查看有哪些可用。

```sh
# iocage fetch
[0] 11.2-RELEASE
[1] 11.3-RELEASE
[2] 12.0-RELEASE
```

输入想要的 RELEASE 的编号。按 [Enter] 获取默认选择：（12.0-RELEASE）。它默认获取最新发行版，但你也可以选择更早的。选定的发行版会被下载、解压到本地磁盘，并更新所有相关安全补丁，这样你的 Jail 就可以为它们的文件系统克隆它。用 `iocage list -r` 查看所有已下载的发行版。

### 创建 Jail 和设置参数

你可以创建没有主机名或 IP 地址的 Jail，但 iocage 会给它分配随机的 UUID 而不是有用的名字，而且这个 Jail 不会有网络访问。用 `-n` 指定 Jail 名，同时设置参数 `ip4_addr` 或 `ip6_addr` 给它网络。用 `-r` 选择发行版。

```sh
# iocage create -n wwww1 ip4_addr="203.0.113.234" -r 11.2-RELEASE
wwww1 successfully created!
```

iocage 为 Jail ZFS 克隆所选的发行版。这让创建 Jail 非常快，但新 Jail 看起来小得有欺骗性。如果你预期这个 Jail 会用很久，可以加上标志 `-T` 做厚 Jail。用 `iocage rename` 更改 Jail 名。

```sh
# iocage create -n wwww1 ip4_addr="203.0.113.234" -r 11.2-RELEASE
wwww1 successfully created!

# iocage rename wwww1 www1
```

用 `iocage destroy` 删除 Jail。

```sh
# iocage destroy www1
```

现在可以运行 Jail 了。

### iocage 启动和关闭

和任何合理的 Jail 管理器一样，iocage 假定 Jail 不应该在系统启动时自动运行。用参数 `boot` 控制 Jail 在启动时是否运行。用 `iocage get -r` 递归获取所有 Jail 的某个参数值。

```sh
# iocage get -r boot
+------+-------------+
| NAME | PROP - boot |
+======+=============+
| www1 | 0           |
+------+-------------+
| www2 | 0           |
+------+-------------+
```

这两个 Jail 都不会在系统启动时自动运行。把 Jail 的参数 `boot` 改为 `on`、`yes` 或 `1`，让它在启动时运行。

```sh
# iocage set boot=on www1
boot: 0 -> 1
```

用 `start`、`stop` 和 `restart` 命令启动、停止和重启 Jail。给出 Jail 名，或者用 `ALL` 影响所有 Jail。

```sh
# iocage start ALL
```

### 查看 Jail

虽然你可以用 `jls(8)` 等标准命令查看 Jail，但你可以用 `iocage list` 获取运行中 Jail 的 iocage 特定信息。

```sh
# iocage list
+-----+------+-------+--------------+------------------+
| JID | NAME | STATE |   RELEASE    |       IP4        |
+=====+======+=======+==============+==================+
| -   | www1 | down  | 12.0-RELEASE | 203.0.113.234/24 |
+-----+------+-------+--------------+------------------+
| -   | www2 | down  | 12.0-RELEASE | 203.0.113.235/24 |
+-----+------+-------+--------------+------------------+
…
```

要查看包含 IPv6 和模板信息的更详细列表，加上标志 `-l`。

### iocage 软件包

用 `iocage pkg` 子命令管理软件包。所有软件包功能，包括升级，都能在 iocage 下工作。给出 Jail 名、**pkg(8)** 命令和软件包名。

```sh
# iocage pkg www1 install sudo
```

你也可以在宿主机上用 `pkg -j` 管理 Jail 软件包。不过我建议你选一种方法坚持用下去。

### iocage 模板

iocage 让你创建干净的模型 Jail，包含所有软件包和配置文件，然后用这个 Jail 作为其他 Jail 的模板。这个模板 Jail 是 FreeBSD 发行版的 ZFS 克隆，模板又被克隆给其他 Jail。你的 Jail 需要 LDAP 和 Kerberos？设置一次就够了。像创建任何其他 Jail 一样创建你的 Jail。模型 Jail 完善后，设置 Jail 的 `template` 属性。

```sh
# iocage set template=yes wwwtemplate
```

模板 Jail 无法启动。用 `iocage list -t` 查看所有模板。要基于模板创建 Jail，在 `iocage create` 时用标志 `-t`。

```sh
# iocage create -t wwwtemplate -n www2
```

你可以在创建时分配 `boot`、`ip4_addr` 和 `ip6_addr` 属性，也可以之后用单独的 `iocage set` 命令。

### iocage 插件

上面这些功能都不错，但它们不是我推荐 iocage 的原因。我推荐 iocage 是因为插件。

插件是预先配置好的、服务单一任务的 Jail。你想要运行杀毒的 Jail？拿 ClamAV 插件。BitTorrent 客户端？qbittorrent 插件。你会发现好几个媒体服务器、个人云、视频摄像头管理器等等。虽然任何人都可以创建插件，但官方插件在被接受进仓库之前，会由 iocage 团队做 sanity 检查。你不会找到那种一切都以 root 运行、或者跑着额外守护进程把你的登录凭据传回插件作者的官方插件，也不会看到 Docker 沼泽里冒出来的各种蠢事。

要查看所有可用插件，运行 `iocage list -PR`。

```sh
# iocage list -PR
```

你会得到所有当前插件的列表。翻看列表，我看到有 UniFi 控制器的插件。我有 UniFi 无线接入点，蹭朋友的控制器，但有自己的控制器就更好了。根据 `iocage list -PR`，这个插件叫 unificontroller。来设置它。用 `iocage fetch` 命令，但加上 `-P` 表示我们要获取插件。用 `-n` 给出插件名。

```sh
# iocage fetch -P -n "unificontroller" ip6_addr=2001.db8::9/64
Plugin: unificontroller
Official Plugin: True
Using RELEASE: 11.2-RELEASE
Using Branch: 12.0-RELEASE
Post-install Artifact: https://github.com/lbalker/iocage-plugin-unificontroller.git
These pkgs will be installed:
- unifi5
…
```

如果你还没有插件的底层发行版，iocage 会获取它，安装软件包和任何配置文件，并用给定的 IP 地址配置。用插件省去了我折腾让软件跑起来的基础工作的麻烦。我可能得为我的古怪本地环境配置它，但那是我自己的问题。

虽然这应该足以让你开始用 iocage，但它还有各种我们没涉及的功能。基础 Jail、ZFS 委托等等都可以在 iocage 里用。用上几天后，你就会明白为什么 iocage 是真正的 Jail 管理器。 •

---

**MICHAEL W LUCAS** 是《Absolute FreeBSD》《FreeBSD Mastery: Jails》以及一堆其他书的作者。新版《Sudo Mastery》应该在你读到本文后不久面世，他的犯罪惊悚小说《Terrapin Sky Tango》也刚脱稿。全部作品见 <https://mwl.io>。
