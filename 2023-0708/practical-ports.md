# 基于 Jail 的广告拦截教程

- 原链接：<https://freebsdfoundation.org/wp-content/uploads/2023/08/reuschling_practical_ports.pdf>
- 作者：BENEDICT REUSCHLING
- 译者：ykla & ChatGPT

FreeBSD 最早吸引我的一点是它能够轻松的运行服务。这些服务可以是随操作系统提供的系统服务（最简单的例子可能是 SSH 守护程序），也可以通过 pkg 或 ports 安装的第三方软件来实现。无论哪种情况，过程都是相同的：你在 `/etc/rc.conf` 中添加一行以启用该服务（可以通过 sysrc 或 `service …… enable` 来实现），使其在系统下次启动时运行。接下来通常会有一个配置文件，用于进行自定义设置以适应你的需求。通常，这需要输入要监听的 IP 地址或 DNS 主机名、网络端口以及一些软件的具体细节。从那时起，要么直接启动该服务（使用 `service …… start`），要么在下次重新启动时启动，以防它需要加载 `kldload` 无法加载的内核模块（这种情况在如今很少出现）。

这一过程很直接，将所有系统服务的配置放在一个方便的位置，并且在为一个或两个服务完成配置后，很容易复现。当然，在 FreeBSD 主机系统上运行一整套服务也是没有问题的，直到情况变得更加复杂。并行运行相同软件的不同版本是非常合理且并不罕见。这是出于测试目的——检查升级是否按预期工作，或者某些软件是否仍需要较旧版本作为依赖。其中一个例子是尝试在相继运行不同版本的 PostgreSQL 数据库。在这种情况下，pkg 和 ports 都会检查版本，如果发现相同位置的二进制文件，它们将取消安装操作，并显示一条信息，指出某些二进制文件将被放置在同一个位置，因此会相互覆盖。这是一个不希望出现的情况，用户必须为其中一个版本做出选择，因为它们无法共存。

除非涉及虚拟化或容器技术，否则就会出现这种情况。这使得在独立的执行环境中进行进程隔离成为可能，使用各种方法让多个这样的系统在同一硬件上运行。虚拟化在操作系统上添加了额外的一层，允许安装相同或不同的操作系统，并模拟硬件。容器或 jail 通过使用 chroot(8) 隔离进程来实现这一点。我们在这里将重点放在后者上，因为它在资源使用方面更轻量级，并且可以相对快速地启动。

这种隔离的好处不仅在于可以并行运行各种不同版本，还可以出于安全原因进行分离。当应用程序在 jail 容器中运行时，默认情况下，内部进程无法访问主机系统。该应用程序可以发现所有通常的设备（如网络）、目录结构和正确位置的文件，但实际上，它是一个独立的环境，模仿主机系统的行为和布局。当这样的 jail 在某种情况下被入侵时，很容易停止它，而不会影响其他 jail 或主机系统中的服务。对它们的访问被严格禁止，将任何入侵者阻隔在特定的 jail 单元中。

这还使得将系统迁移到另一个主机变得容易，只需停止、复制 jail 的目录结构到新位置，然后在那里重新启动（例如，进行一些本地修改，如新的 IP 地址）。备份和恢复也是以相同的方式进行的。通常，多个这样的 jail 由管理 jail 的软件来管理，该软件负责创建、修改和删除 jail。

其中这样的一个 jail 管理器被称为 Bastille，它完全是用 shell 脚本编写的。在这篇文章中，我们将通过设置主机系统、创建 jail，并基于模板在其中启动服务的过程，更详细地了解 Bastille。这些模板允许在中央存储库中共享配置，以便在不需要了解服务内部工作原理的情况下应用它们。通过这种方式，即使对于希望快速启动某些内容的人，也很容易为复杂的情况进行设置。

在这篇文章中，我们将部署一个名为 AdGuard 的服务，由 AdGuard Software Limited 提供。通过在网络中运行 AdGuard 服务，将连接其 DNS 解析到该服务的客户端可以过滤出网页浏览活动中的广告。这有助于避免广告商的跟踪和用户配置文件的建立，同时还可以加快页面加载速度，因为它们不必在用户想要查看的内容旁边传输广告。AdGuard 通过过滤列表和 DNS sinkholing 来实现这一点。基于过滤列表，AdGuard 会在在浏览器中呈现广告之前，通过发送无效的地址响应来阻止已知的广告站点。有多种使用 AdGuard 服务的方式——作为个人设备的浏览器扩展、桌面应用程序，或者将其作为递归 DNS 解析器运行。请注意，AdGuard 并不能完全防止所有形式的广告（尤其是动态嵌入在视频站点中的广告），但它在从网页中移除广告的方面表现出色。

![image](https://github.com/FreeBSD-Ask/freebsd-journal-cn/assets/10327999/cfb9b796-2366-4bc5-a90f-40dd6fcb35da)

我们从树莓派开始，因为这个服务基本上一直在运行，而且我们希望功耗较低。我这里有一个树莓派 3，但其他能够运行 FreeBSD 的设备（包括完整的服务器）同样适用。安装操作系统，应用最新的安全补丁，并使用 SSH 密钥锁定远程访问。

## 环境设置

我连接了一个旧的 32GB 固态硬盘到树莓派上，这个固态硬盘通过一个单磁盘的 ZFS 池来执行大部分 I/O 密集的操作，而不是使用速度较慢的存储卡。在撰写本文时，我正在运行 FreeBSD 13.2，我十分有信心未来的版本也会同样表现出色，或者只需要进行一些小的调整。

```
# pkg install bastille git-lite
```

由于 Bastille 是一个 Shell 脚本，安装相当迅速，没有额外的依赖。虽然它在一些功能方面可能没有其他 jail 管理器那么全面，但它仍然能够正常工作。为了从 Bastille 的 GitLab 存储库中克隆 adguard home 模板（以及其他模板），需要安装 Git。接下来，我们在 `/etc/pf.conf` 中创建一个针对 bastille 的 PF 配置，内容如下：

```
ext_if=”ue0” ## <- 将“ue0”更改为你机器的配置。
set block-policy return
scrub in on $ext_if all fragment reassemble
set skip on lo
table <jails> persist
nat on $ext_if from <jails> to any -> ($ext_if:0)
rdr-anchor “rdr/*”
block in all
pass out quick keep state
pass in inet proto tcp from any to any port ssh flags S/SA keep state
pass in inet proto tcp from any to any port bootps flags S/SA keep state
pass in inet proto tcp from any to any port {9100,9124} flags S/SA modulate state
```

请确保在顶行即 "ext_if" 行中将其更改为你正在使用的接口。在我的树莓派上，网线连接到了 "ue0"，所以我输入了 ue0。pf.conf 将为我们的 jail 流量（使用 NAT）创建一个表。Bastille 支持多个用于网络的参数，这使得它足够灵活，既适用于办公室和家庭网络，也适用于托管服务提供的网络。这些选项在此处有详细描述：<https://docs.bastillebsd.org/en/latest/chapters/networking.html>。

我将使用基于 VNET 的 jail，因为我在本地网络上有一个可用的 IP 地址。在编辑配置文件后，我们在启动时添加了一个条目来启动 PF 和 pf 日志设备，以及其他服务。Bastille 也应该会启动，我们列出了将为 AdGuard 创建的 jail 的名称（我的命名方案既传奇又无聊）。

```
# sysrc pf_enable=YES
# sysrc pflog_enable=YES
# sysrc bastille_enable=yes
# sysrc bastlle_list=”adguard”
```

在启动防火墙之前，检查防火墙规则集是否存在错误是一个很好的做法。使用：

```
# pfctl -nvf /etc/pf.conf
```

用于进行此类检查。成功时，它将回显整个规则集，或者显示你可能遇到的任何错误。请注意，它无法检查逻辑错误，比如会阻止 SSH 端口 22，而这可能是唯一的远程连接方式。幸运的是，已经存在一条（默认）规则允许 SSH 流量通过。检查完成后，请启动 PF 服务，并开始过滤流量：

```
# service pf start
# service pflog start
```

预计你的 SSH 连接将会断开，因此请保持一个单独的终端连接，以防你无法再次访问。重新连接后，我们需要编辑另外一些配置文件。启用 VNET 的 jail 需要在 `/etc/devfs.rules`（而不是 `.conf`）中添加一个条目，在新安装的系统上可能不存在这个文件。只需创建该文件并添加以下规则：

```
[bastille_vnet=13]
add path bpf* unhide
```

这使得 Bastille 能够看到 VNET 接口上的流量，并将 jail 连接到外部世界。这可能是对正在进行的情况的非专业人士的说明。幸运的是，我们不需要过于担心它（也许我需要在我的下一次网络考试中深入研究它）。

我们还需要访问的另一个文件是 `/etc/sysctl.conf`，需要添加以下行：

```
sysctl net.inet.ip.forwarding=1
sysctl net.link.bridge.pfil_bridge=0
sysctl net.link.bridge.pfil_onlyip=0
sysctl net.link.bridge.pfil_member=0
```

当 Bastille 运行时，它会动态地为我们在树莓派的外部接口（ue0）与 jail 的网络接口（vtnet）之间创建一个桥接。这两个接口通过一根虚拟网线连接在一起，一端连接在主机的接口上，另一端连接在 jail 上，通过它进行流量交换。

将这些更改应用到正在运行的系统中，而无需重新启动，可以使用以下命令：

```
# sysctl -f /etc/sysctl.conf
```

当我完成设置并重新启动树莓派后，发现 jail 无法再访问网络时，我感到非常困惑。经过一番思考后，我从以下的交流中得知了原因：


<https://www.mail-archive.com/freebsd-net@freebsd.org/msg64577.html>


这在 FreeBSD 13，需要在 `/boot/loader.conf` 中添加额外的一行。这可能会让你发疯，所以在变得疯狂之前，请将以下内容添加到其中，以确保在未来的重启中正常工作：

```
if_bridge_load=”YES”
```

已经正确加载桥接接口，这也导致 sysctl 出现，使得 sysctl.conf 可以将它们从默认值 1 更改为 0。尽管如此，在完成之前，我们还要访问一个最后的文件。

Bastille 的配置文件位于 `/usr/local/etc/bastille/bastille.conf`。你可以直接编辑它（它有很详细的注释），或者如果你不介意输入很多内容，可以使用 sysrc 命令。由于我正在一个连接到我的树莓派的 ZFS 池上运行，我设置了 `bastille_zfs_enable` 以给它指定我的池的名称。

```
# sysrc -f /usr/local/etc/bastille/bastille.conf bastille_zfs_enable=YES
# sysrc -f /usr/local/etc/bastille/bastille.conf bastille_zfs_zpool=rpi3
```

如果你的池名称与 `bastille_zfs_zpool` 行上的名称不同，请将其更改为你的池名称。我还更改了一个选项，即参数 `bastille_network_gateway=""`。我输入了我的默认网关地址，因为在后续的过程中，我遇到了一些解析 jail 名称的问题。你可能需要或不需要设置这个选项，但如果你确实遇到问题，请重新查看这个选项，看看是否可以解决问题。

## Bastille 自举

现在，所有设置都已经就位，是时候让 Bastille 在我们分配给它的池上创建数据集结构了。它将下载一个简单的 FreeBSD 13.2 RELEASE，并更新其中后续发布的任何补丁。执行以下命令，直至它完成：

```
# bastille bootstrap 13.2-RELEASE update
Bootstrapping FreeBSD distfiles...
/usr/local/bastille/cache/13.2-RELEASE/MANIFES 782 B 1670 kBps 00s
/usr/local/bastille/cache/13.2-RELEASE/base.tx 168 MB 6526 kBps 26s
Validated checksum for 13.2-RELEASE: base.txz
MANIFEST: 7d1b032a480647a73d6d7331139268a45e628c9f5ae52d22b110db65fdcb30ff
DOWNLOAD: 7d1b032a480647a73d6d7331139268a45e628c9f5ae52d22b110db65fdcb30ff
Extracting FreeBSD 13.2-RELEASE base.txz.
Bootstrap successful.
See ‘bastille —help’ for available commands.
src component not installed, skipped
Looking up update.FreeBSD.org mirrors... 2 mirrors found.
Fetching metadata signature for 13.2-RELEASE from update2.freebsd.org... done.
Fetching metadata index... done.
Inspecting system... done.
Preparing to download files... done.
The following files will be updated as part of updating to
13.2-RELEASE-p1:
/bin/freebsd-version
/usr/lib/libpam.a
/usr/lib/pam_krb5.so.6
/usr/share/locale/zh_CN.GB18030/LC_COLLATE
/usr/share/locale/zh_CN.GB18030/LC_CTYPE
/usr/share/man/man8/pam_krb5.8.gz
Installing updates...
Restarting sshd after upgrade
Performing sanity check on sshd configuration.
Stopping sshd.
Waiting for PIDS: 1063.
Performing sanity check on sshd configuration.
Starting sshd.
Scanning /usr/local/bastille/releases/13.2-RELEASE/usr/share/certs/blacklisted for certificates...
Scanning /usr/local/bastille/releases/13.2-RELEASE/usr/share/certs/trusted for certificates...
 done.
```

在自举操作之后，我的池中增加了这些数据集：

```
# zfs list -r rpi3/bastille
NAME USED AVAIL REFER MOUNTPOINT
rpi3 621M 28.0G 24K /rpi3
rpi3/bastille 584M 28.0G 26K /usr/local/bastille
rpi3/bastille/backups 24K 28.0G 24K /usr/local/bastille/backups
rpi3/bastille/cache 169M 28.0G 24K /usr/local/bastille/cache
rpi3/bastille/cache/13.2-RELEASE 169M 28.0G 169M /usr/local/bastille/cache/13.2-RELEASE
rpi3/bastille/jails 24K 28.0G 24K /usr/local/bastille/jails
rpi3/bastille/logs 24K 28.0G 24K /var/log/bastille
rpi3/bastille/releases 414M 28.0G 24K /usr/local/bastille/releases
rpi3/bastille/releases/13.2-RELEASE 414M 28.0G 414M /usr/local/bastille/releases/13.2-RELEASE
rpi3/bastille/templates 24K 28.0G 24K /usr/local/bastille/templates
```

让我们运行另一个自举操作，这次是为了提供 AdGuard Home 模板。

```
# bastille bootstrap https://gitlab.com/bastillebsd-templates/adguardhome
warning: redirecting to https://gitlab.com/bastillebsd-templates/adguardhome.git/
Already up to date.
Detected Bastillefile hook.
[Bastillefile]:
PKG ca_root_nss adguardhome
CP usr /
SYSRC adguardhome_enable=YES
SERVICE adguardhome start
RDR tcp 80 80
RDR udp 53 53
Template ready to use.
```

这很快就会完成。Bastille 拥有自己的模板语言，你可以在大写的命令（如 PKG、CP 等）中看到它们。它们具有与其小写形式中的系统等效功能相同。借助这些命令，可以在 jail 中按正确的顺序设置服务。它们大多是自解释的。最后的两个 RDR 命令将网络端口从主机系统重定向到 jail 中。所有其他端口仍然受到防火墙的保护，因此只有端口 80 从主机连接到 jail（以及反向连接），以及 DNS 端口 53。在你的  `/etc/pf.conf` 中检查 `rdr-anchor "rdr/*"` 这一行。这就是使其如此灵活的地方。不必为所有 jail 都打开端口，每个 jail 都可以打开所需的端口并保持其他端口关闭。

现在是创建和启动我们的第一个 Bastille jail 的时候了。因为我们正在使用 VNET，所以我们需要在 bastille create 命令中传递参数 `-V` ，以及 jail 的名称、要运行的版本，随后是分配给 jail 的本地网络上的 IP 地址，以及主机的网络接口用于桥接。组合起来，命令如下：

```
# bastille create -V adguard 13.2-RELEASE 192.168.2.55 ue0
Valid: (192.168.2.55).
Valid: (ue0).
Creating a thinjail...
[adguard]:
e0a_bastille0
e0b_bastille0
adguard: created
[adguard]:
Applying template: default/vnet...
[adguard]:
Applying template: default/base...
[adguard]:
[adguard]: 0
[adguard]:
syslogd_flags: -s -> -ss
[adguard]:
sendmail_enable: NO -> NO
[adguard]:
sendmail_submit_enable: YES -> NO
[adguard]:
sendmail_outbound_enable: YES -> NO
[adguard]:
sendmail_msp_queue_enable: YES -> NO
[adguard]:
cron_flags: -> -J 60
[adguard]:
/etc/resolv.conf -> /usr/local/bastille/jails/adguard/root/etc/resolv.conf
Template applied: default/base
No value provided for arg: GATEWAY6
[adguard]:
ifconfig_e0b_bastille0_name: -> vnet0
[adguard]:
ifconfig_vnet0: -> inet 192.168.2.55
[adguard]:
defaultrouter: NO -> 192.168.2.1
[adguard]: 0
[adguard]:
[adguard]: 0
Template applied: default/vnet
[adguard]:
adguard: removed
no IP address found for -
[adguard]:
e0a_bastille0
e0b_bastille0
adguard: created
```

你可以看到我之前提到的虚拟网线的两端：e0a_bastile0 和 e0b_bastile0 形成了主机系统与 jail 之间的连接。在主机上检查 ifconfig 输出，可以看到从 jail 的流量创建的新桥接。

在创建 jail 期间应用的设置是相当标准的，主要是禁用了我们不会使用的服务。在创建了 jail 之后，我的池中还存在两个数据集，其中保存了所有 jail 的数据：

```
# zfs list|grep adguard
rpi3/bastille/jails/adguard 2.36M 28.0G 26.5K /usr/local/bastille/jails/adguard
rpi3/bastille/jails/adguard/root 2.34M 28.0G 2.34M /usr/local/bastille/jails/adguard/
root
```

这构成了 jail 的根文件系统，并遵循其他 jail 管理器的布局。要将文件复制到或从 jail 中复制出来，只需使用前缀 `/usr/local/bastille/jails/adguard/root` 来访问 jail 的根目录。

`jls` 命令将列出 bastille jail 及其设置：

```
# bastille list -a
 JID State IP Address Published Ports Hostname Release Path
 adguard Up 192.168.2.55 - adguard 13.2-RELEASE-p1 /usr/local/bastille/
jails/adguard/root
```

此时，jail 已经在运行。唯一缺少的是 adguard home 的安装。由于我们之前已经引导了该模板，我们可以使用以下命令将其应用于 jail：

```
# bastille template adguard bastillebsd-templates/adguardhome
bastille template adguard bastillebsd-templates/adguardhome
[adguard]:
Applying template: bastillebsd-templates/adguardhome...
[adguard]:
Bootstrapping pkg from pkg+http://pkg.FreeBSD.org/FreeBSD:13:aarch64/quarterly, please wait...
Verifying signature with trusted certificate pkg.freebsd.org.2013102301... done
[adguard] Installing pkg-1.19.1_1...
[adguard] Extracting pkg-1.19.1_1: 100%
Updating FreeBSD repository catalogue...
[adguard] Fetching meta.conf: 100% 163 B 0.2kB/s 00:01
[adguard] Fetching packagesite.pkg: 100% 6 MiB 6.5MB/s 00:01
Processing entries: 100%
FreeBSD repository update completed. 31664 packages processed.
All repositories are up to date.
Updating database digests format: 100%
The following 2 package(s) will be affected (of 0 checked):
New packages to be INSTALLED:
 adguardhome: 0.107.22_5
 ca_root_nss: 3.89
Number of packages to be installed: 2
The process will require 41 MiB more space.
7 MiB to be downloaded.
[adguard] [1/2] Fetching adguardhome-0.107.22_5.pkg: 100% 6 MiB 6.7MB/s 00:01
[adguard] [2/2] Fetching ca_root_nss-3.89.pkg: 100% 266 KiB 272.1kB/s 00:01
Checking integrity... done (0 conflicting)
[adguard] [1/2] Installing ca_root_nss-3.89...
[adguard] [1/2] Extracting ca_root_nss-3.89: 100%
[adguard] [2/2] Installing adguardhome-0.107.22_5...
[adguard] [2/2] Extracting adguardhome-0.107.22_5: 100%
=====
Message from ca_root_nss-3.89:
—
FreeBSD does not, and can not warrant that the certification authorities
whose certificates are included in this package have in any way been
audited for trustworthiness or RFC 3647 compliance.
Assessment and verification of trust is the complete responsibility of the
system administrator.
This package installs symlinks to support root certificates discovery by
default for software that uses OpenSSL.
This enables SSL Certificate Verification by client software without manual
intervention.
If you prefer to do this manually, replace the following symlinks with
either an empty file or your site-local certificate bundle.
* /etc/ssl/cert.pem
 * /usr/local/etc/ssl/cert.pem
 * /usr/local/openssl/cert.pem
=====
Message from adguardhome-0.107.22_5:
—
You installed AdGuardHome: Network-wide ads & trackers blocking DNS server.
In order to use it please start the service ‘adguardhome’ and
then access the URL http://0.0.0.0:3000/ in your favorite browser.
[adguard]:
/usr/local/bastille/templates/bastillebsd-templates/adguardhome/usr -> /usr/local/bastille/jails/
adguard/root/usr
/usr/local/bastille/templates/bastillebsd-templates/adguardhome/usr/local -> /usr/local/bastille/
jails/adguard/root/usr/local
/usr/local/bastille/templates/bastillebsd-templates/adguardhome/usr/local/bin -> /usr/local/bastille/jails/adguard/root/usr/local/bin
/usr/local/bastille/templates/bastillebsd-templates/adguardhome/usr/local/bin/AdGuardHome.yaml ->
/usr/local/bastille/jails/adguard/root/usr/local/bin/AdGuardHome.yaml
[adguard]:
adguardhome_enable: -> YES
[adguard]:
moving old config /usr/local/bin/AdGuardHome.yaml to the new location /usr/local/etc/AdGuardHome.
yaml
Starting adguardhome.
stdin:2: syntax error
pfctl: Syntax error in config file: pf rules not loaded
tcp 80 80
stdin:2: syntax error
pfctl: Syntax error in config file: pf rules not loaded
udp 53 53
Template applied: bastillebsd-templates/adguardhome
```

它只需要在 jail 中执行模板中的指令（PKG、CP 等）。这也是一个很好的测试，可以看到网络是否设置正确。如果没有设置正确，jail 将无法从存储库获取软件包。最后的 pfctl 警告让我有点担心，但尽管有这些警告，它还是正常工作了。

满怀期待，我按照屏幕上的一条消息的指示，打开了浏览器，并将其指向 jail 的 IP 地址。果然，AdGuard Home 的登录界面出现了。但是，凭证在哪里呢？我在 Bastille 模板网站 <https://gitlab.com/bastillebsd-templates> 上查找了一下，没有找到有效的信息。Bastille 的博客文章提到了将 AdGuard 作为用户名，但密码不适用。因此，我不得不创建自己的凭证，这实际上更安全，因为默认密码很容易被不良分子扫描到。

我使用以下命令在 jail 中打开了一个控制台：

```
# bastille console adguard
```

在 jail 中，我发现 AdGuard 将其配置文件放在了 `/usr/local/etc/AdGuardHome.yaml` 下。在顶部附近，我找到了以下部分：

```
users:
 - name: adguard
 password: some password not in clear text
```

再次退出后，我需要一种方法来创建 BCrypt 密码。htpasswd 工具可以做到这一点，因此我安装了包含该工具的 apache24 web 服务器：

```
# pkg install apache24
```

运行“refresh”命令后，我可以运行 htpasswd 工具。查看它的 man 页面，我必须构建一个类似这样的命令行：

```
htpasswd -Bnb adguard BastilleBSD!
```

我使用了参数 `-B` 创建了一个 BCrypt 密码，接着是应用于此密码的用户（我们已经在配置文件中有了这个信息，但也许你想要其他用户或多个用户），然后是明文密码。是的，这不是一种安全的方式，因为这会出现在你的 shell 历史记录中。但是在本教程的任何地方，我是否曾表示过它已经准备好投入生产？恰恰相反，我没有。我尽职尽责地运行了 htpasswd，它在命令行上输出了生成的密码，我将其复制并粘贴到了 AdGuard 的配置文件中。

然后我执行了：

```
# service adguardhome restart
```

（请注意，仍在我的 jail 中）来重新启动服务并应用新的设置。该文件中的其他设置在 AdGuard Home Wiki 中有详细记录：<https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration> 刷新我的网页浏览器后，我输入了我的新凭证，并被重定向到了主 AdGuard 仪表板。成功！

在顶部，有一个设置向导，显示了如何使用你的新 AdGuard 服务，无论是用于你的路由器（以覆盖整个网络）还是各种设备，并为移动设备和桌面操作系统都进行了描述。太棒了！

![image](https://github.com/FreeBSD-Ask/freebsd-journal-cn/assets/10327999/2119e84d-b4ed-4a0f-85fc-9dee97a3116d)

在我手机上这样做后——出于测试目的——我稍微浏览了一下网页，就看到了仪表板中出现了统计数据。这表明我们的设置正在运行，我们应该将互联网更名为 SnooperNet。几乎所有的网站在某种程度上都会追踪你或显示让你不悦的广告。树莓派能够处理这种负载，我在 `AdGuardHome.yaml` 的 ratelimit 参数中微调了连接数。

你可以在 jail 的 `/var/log/adguardhome.log` 目录中找到 AdGuard 为该服务编写的日志。

## 总结

这就是本教程的内容。我发现 AdGuard 的文档很完善，并且由于模板创建者的工作，很容易入门。我已经享受到在互联网上留下更少的痕迹并看到更少的广告。它作为一个 DNS 服务的好处是，你网络上的任何设备都可以使用它：个人电脑、笔记本电脑、智能手机、平板电脑、电视、物联网设备，甚至可能还有邻居家的智能猫门。

Bastille 可能需要一些初始配置，但之后，创建 jail 就是一个简单的过程。也许你会发现其他你想要在 Bastille 模板上运行的服务：<https://gitlab.com/bastillebsd-templates>？

---

**BENEDICT REUSCHLING** 是 FreeBSD 项目中的文档提交者，并且是文档工程团队的成员。在过去，他曾连任两届 FreeBSD 核心团队成员。他在德国达姆施塔特应用技术大学管理着一个大数据集群。他还为本科生开设了“Unix for Developers”课程。Benedict 还是每周 bsdnow.tv 播客的主持人之一。
