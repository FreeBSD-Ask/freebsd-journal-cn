# Poudriere

- 原文：[Poudriere](https://freebsdfoundation.org/wp-content/uploads/2014/03/Poudriere.pdf)
- 作者：**Bryan Drewery**

停止在你的服务器上使用 portmaster、portupgrade 和 ports，转而使用包。用 Poudriere 搭建你自己的包构建只需几分钟，未来会为你节省大量时间。

在 Pkg 可用之前，我从未真正考虑过使用包。旧式的 `pkg_install` 包用于初始系统安装尚可，但除了全部删除并安装新的一套之外，没有内置的升级途径。你不得不使用 portmaster 或 portupgrade 这样的工具，并检出一个 INDEX 或 Ports 树。这些工具看起来在包升级方面做得不错，但它们遗漏很多内容并制造额外工作。

## 为何使用包

如果你维护着不止一台 FreeBSD 系统却还没有使用包，你应该试试。我只维护 20 台服务器，但在每台系统上构建 Ports 占用了我大量时间，也浪费了生产机器的资源。在多台服务器上构建 Ports 时，它们的选项或版本很容易失去同步。通过在一台系统上一次性构建包，我减轻了系统负载，减少了必须做的工作，并使所有系统保持一致。与其在每台系统上处理同样的故障，我只需在构建系统上处理一次。

自 2013 年 11 月起，FreeBSD 为 Pkg（前称 pkgng）提供官方包。10 版本还带来了首批签名包。该项目使用 Poudriere 构建包。

通常 Ports 更新时，`PORTREVISION` 并不会被提高以强制重建包。这有时被遗忘，另一些时候并不现实，因为需要提高数千个 Ports 的版本号才能跟进某次依赖更新。Pkg 处理这种情况比旧系统更好。Pkg 还能检测已安装包所选选项与远程可用包相比是否发生了变化，并会自动重装它们。旧工具有时需要递归重装包，有时又不需要。旧包系统涉及太多手工操作。Pkg 的目标是拥有去除手工干预的内置升级流程。在消除某些手工干预方面仍有一些工作要做，但它已经远胜旧系统。

## 定制包选项

为什么需要偏离官方包？Ports 框架为 Ports 提供选项支持以更改构建时配置。并非所有应用都支持运行时配置。某些应用必须根据启用的功能以不同方式编译。另一些则提供选项，仅仅为了减少默认 Port 中的功能和依赖。对服务器管理员而言，这很快会导致发现某些默认包无法满足需求。常见例子是 PHP 默认以 CGI 模式提供，不支持 apache+mod_php 或更灵活的 PHP-FPM。默认包的另一个常见问题是它们附带 X11 支持，这在非桌面环境中可能并不理想。也许你有定制的 Ports 或针对某些 Ports 的定制补丁。通过自行构建包，你重新掌握了包构建时使用哪些选项以及更新发布的频率。

自行构建包的另外一些理由是：涉及 FreeBSD 项目无法分发的限制性许可证，或你的系统高度定制且与 FreeBSD 不 ABI 兼容。

获取定制包有几种方式。Pkg 支持使用多个仓库。可以配置为以官方 FreeBSD 仓库为主、定制仓库为辅。Pkg 可跟踪的仓库数量不受限制，并可按优先级重新排序。多仓库的问题是目前维护起来可能比较困难。

当 Pkg 检测到已安装包的选项或依赖与所跟踪仓库不同时，该包将从任意远程版本重装。你可以在升级期间用 `pkg lock PKGNAME` 和 `pkg unlock PKGNAME` 锁定包，或用 `pkg annotate -A PKGNAME repository REPONAME` 将其绑定到特定仓库。还有微妙的问题：让定制仓库的 Ports 树与 FreeBSD 包保持同步。由于包是基于每周一次的 Ports 树快照构建的，如果你的定制仓库与之不匹配，可能导致冲突。最简单的做法是用你想要的选项构建一套恰好你需要的完整包集。在你的系统上只跟踪你自己的仓库，不包含 FreeBSD 官方仓库。这样做还有好处：使用你自己的基础设施分发包，能显著加快升级。

## 构建包

长期以来 Tinderbox 是构建包的主流首选工具。另一些人会在一台系统上安装所有 Ports，然后从该系统创建包并复制到其他系统。这种方法不推荐，因为包是在不断变大、不断污染的不洁环境中创建的。即便今天使用 portmaster 配合 Ports 并从中创建 Pkg 包用于分发，出于同样原因也不推荐。最好使用专门为创建包集而设计的系统。

Poudriere（大致读作 poo-dree-year，法语意为“火药桶”）作为 Tinderbox 的更快更简单的替代品而编写。它由 Pkg 作者 Baptiste Daroussin 编写，现主要由我与 Baptiste 以及其他一些贡献者维护。它迅速成为 FreeBSD 事实上的 Port 测试和包构建工具。它是官方构建集群工具，也被 FreeBSD Ports 项目用于在所谓“exp-runs”中测试大范围补丁。它用 POSIX shell 编写，并正缓慢向 C 组件迁移。与 Tinderbox 不同，它没有 web 界面依赖，也不需要数据库。它在所有操作上都经过大幅优化以实现高度并行。它使用 Jail 在非常严格的条件下于沙箱环境中构建 Ports。Jail 创建通过一条简单命令一次完成。构建期间，Jail 会自动为每个使用的 CPU 克隆一份，给 Ports 提供干净的构建场所。构建可以在 UFS、ZFS 或 TMPFS 文件系统上进行。UFS 是高 I/O、低 RAM、慢速构建，TMPFS 是低 I/O、高 RAM、快速构建。它也可配置为只让构建的某些部分使用 TMPFS，其余使用 UFS/ZFS，以便在低内存机器上达成某种折中。amd64 主机也能毫不费力地构建 i386 包。可以为主机当前版本或更老版本构建包。例如，主机是 9.2 时，可构建 9.2、9.1 和 8.3 包集。

Poudriere 默认增量构建，只重建需要的部分。增量构建会检查选项变更、缺失依赖、依赖变更、新版本以及 pkgname 变更。任一项变化都会重建该 Port。这也会导致依赖该 Port 的任何内容被重建。这有时是过度处理，但能确保包创建中不遗漏任何 Port 变更。它还内置 ccache 支持，能在依赖变化时帮助缩短 Port 重建时间。包集的构建时间各不相同，但在一台拥有多 CPU 和足够 RAM 的系统上，几百个 Port 通常能在一两小时内构建完成。

Poudriere 提供只读的实时 web 界面，用于监控构建状态。该界面不需要任何服务端 CGI 或脚本支持，因为 Poudriere 只是把状态文件以 JSON 写出，再由 web 界面使用。它不如 Tinderbox 界面精美，但未来有计划进一步改进。3.1 版本经过增量改进，响应更灵敏，并允许对每个子包列表搜索和排序。

Poudriere 还有称为“set”的功能。这允许为每个命名的“set”保存多套选项、**make.conf** 文件和最终包集。这就免去了为同一目标版本/架构维护多个 Jail 的需要。例如，可以在构建系统上用同一个 Jail 创建名为“php53”的 PHP 5.3 包集和名为“php55”的 PHP 5.5 包集。构建该 set 时用 `-z setname` 指定，例如 `bulk -a php53 -j 91amd64` 会产出包于 **/usr/local/poudriere/data/packages/91amd64-default-php53**。其中 `default` 指 Ports 树，可用 `-p` 选项更改。

即将发布的 Poudriere 3.1 还带来一些有趣的新功能。其中一项主要功能名为 `ATOMIC_PACKAGE_REPOSITORY`。它防止仓库在构建完成前被修改。目前在 3.0 中，仓库在启动时删除包，构建期间修改包，因此无法直接通过 http 提供服务。该功能默认启用。其工作方式是启动时将包目录硬链接复制到 `.building` 目录，构建完成时 `.building` 目录重命名为 `.real_TIMESTAMP` 目录，仓库顶层的 `.latest` 符号链接更新为指向新构建。在 `pkg upgrade` 任务期间改变仓库仍可能存在问题，但问题窗口比不启用该功能时小得多（框 1）。

此功能还允许使用 `bulk -n` 做 dry-run，查看会执行什么操作。

原子包仓库还允许保留旧包集。这默认不启用，但可通过设置 `KEEP_OLD_PACKAGES` 和 `KEEP_OLD_PACKAGES_COUNT` 启用。默认保留 5 套。借此你可以将 `.latest` 符号链接指向旧集，然后在服务器上运行 `pkg upgrade -f` 强制从远程仓库重装所有包，从而回滚系统。这会把所有包降级到旧集。

3.1 的另一项即将推出的功能名为 poudriered。它允许通过通往 root 守护进程的 socket 以非 root 用户使用 poudriere。这允许排队作业，也允许为所有 Jail 排队作业。它的配置方式类似 sudo，可以限制子命令，甚至限制到特定用户和组的参数。更多改进（如守护进程权限分离）计划在 3.2/4.0 中实现。

Poudriere 的设置和使用简单快速。安装 poudriere、创建 Jail、检出 Ports 树、创建 Port 列表文件、可选地为签名仓库创建公私钥对，然后构建！（框 2）。

如果需要构建多个 set，应在使用 `options` 和 `bulk` 命令时使用 `-z` 标志，并在 **/usr/local/etc/poudriere.d** 中设置 **SET-make.conf**，写入该 set 专属的配置。

Poudriere 官方网站有一份创建和维护仓库的指南。手册页也可在此在线查阅。还有一份使用 Jenkins 定时构建包的指南，在三部分系列中有详细文档。

## 框 1：原子包仓库布局

```sh
/usr/packages/exp-91amd64-commit-test # ls -al
total 13
drwxr-xr-x 7 root wheel 12 Mar 2 01:59 ./
drwxr-x—x 26 root wheel 32 Mar 2 01:13 ../
lrwxr-xr-x 1 root wheel 16 Mar 2 01:59 .latest@ -> .real_1393747164
drwxr-xr-x 4 root wheel 7 Mar 1 16:59 .real_1393714735/
drwxr-xr-x 4 root wheel 7 Mar 2 00:40 .real_1393742366/
drwxr-xr-x 4 root wheel 7 Mar 2 00:58 .real_1393743542/
drwxr-xr-x 4 root wheel 7 Mar 2 01:05 .real_1393743901/
drwxr-xr-x 4 root wheel 7 Mar 2 02:00 .real_1393747164/
lrwxr-xr-x 1 root wheel 11 Nov 19 17:20 All@ -> .latest/All
lrwxr-xr-x 1 root wheel 14 Nov 19 17:20 Latest@ -> .latest/Latest
lrwxr-xr-x 1 root wheel 19 Nov 19 17:20 digests.txz@ -> .latest/digests.txz
lrwxr-xr-x 1 root wheel 23 Nov 19 17:20 packagesite.txz@ -> .latest/packagesite.txz
```

## 框 2：典型 Poudriere 设置

```sh
$ pkg install ports-mgmt/poudriere
$ cp /usr/local/etc/poudriere.conf.sample /usr/local/etc/poudriere.conf
# 修改配置。
$ vim /usr/local/etc/poudriere.conf
# 在 /usr/local/poudriere/ports/default 创建 Ports 树
$ poudriere ports -c -m svn+https
# 从快照创建 Jail
$ poudriere -j 10amd64 -v 10.0-RELEASE -a amd64
# 从源码创建 head Jail
$ poudriere -j head-amd64 -v head -a amd64 -m svn+https
# 创建 Port origin 列表（cat/port），每行一个。
$ vim /usr/local/etc/poudriere.d/ports.list
# 为仓库创建公私钥对
$ cd /etc/ssl
$ openssl genrsa -out repo.key 2048
$ chmod 0400 repo.key
$ openssl rsa -in repo.key -out repo.pub -pubout
# 配置 poudriere 使用你的公钥
$ echo "PKG_REPO_SIGNING_KEY=/etc/ssl/repo.key" >> /usr/local/etc/poudriere.conf
# 创建 make.conf
$ echo "WITH_PKGNG=yes" >> /usr/local/etc/poudriere.d/make.conf
# 配置构建选项
$ poudriere options -f /usr/local/etc/poudriere.d/ports.list
# 构建包
$ poudriere bulk -j 91amd64 -f /usr/local/etc/poudriere.d/ports.list
```

## FreeBSD 如何构建包

FreeBSD 项目过去只在发布时和偶尔在 STABLE 分支上构建包。旧包构建器使用一套名为 Portbuild 的分布式系统。它使用一大群 2GB-4GB 的小型机器来构建包。这种方式容易出错且缓慢，主要由于机器老旧。一次完整构建仍可能花费一周。

如今包使用单台大型机器配合 Poudriere 构建。FreeBSD 基金会慷慨地采购了几台 24-32 CPU、96GB 内存的机器来替换旧集群。使用新系统配合 Poudriere，整个 Ports 树可在一台机器上从头构建，耗时约 16 小时。

包为每个分支的最老版本构建。这些包应与这些分支上的所有未来版本和该版本的 STABLE 分支 ABI/KBI 兼容。这意味着为 8.3 构建的包可在 8.4 上工作，但不保证能在 9.x 上工作。对于官方 FreeBSD 包构建，每周二晚间会做一次 Ports 树快照并开始构建包。目前包从 8.3、9.1、10.0 和 head 为 i386 和 amd64 构建。季度 Ports 分支也为 10.0 在 i386 和 amd64 上构建。这合计每周需构建 10 套独立的包集。我们将其分到两台独立服务器，一台 i386，一台 amd64。并非所有 24,000 个 Port 每周都为每套集构建，因为 Poudriere 足够聪明，只构建需要重建的部分。所有集构建完成只需几天。每套包集构建完成后，其仓库由我们的签名服务器生成并签名。我们如何签名包的示例见 `pkg-repo.8` 手册页。随后包通过 rsync 上传到我们的内容分发网络供公众使用。

## 部署

官方 FreeBSD 包 URL 使用 SRV DNS 记录来通告可用的镜像，除此之外就是普通 HTTP。如果你的网络完全在内网，部署方面无需太多工作。只需要一台内部 FTP 或 HTTP 服务器来提供包仓库。我托管的服务器分布在世界各地，不在同一网络中。起初我尝试只用一台 HTTP 服务器提供包，但很快发现同时更新所有服务器会严重拖慢一切。如果你有大带宽管道，这仍可能可行。我最终使用 Amazon S3，体验好得多。我为构建写了使用 s3sync 上传包的 bulk 钩子。我在上面托管 5GB 包集，每周更新服务器。算下来每月只要几美元甚至更少就能更新我的 20 台服务器。

要让服务器使用你的仓库，在 **/usr/local/etc/pkg/repos/MYREPO.conf** 创建配置文件，并将仓库公钥放在 **/usr/local/etc/pkg/repos/MYREPO.pem**（框 3）。此配置需要在仓库中设置 ABI 符号链接。这是一次性操作。它允许你在所有服务器上使用同一仓库配置，而无需更改使用的版本和架构。Pkg 获取包时会更改 ABI 的值。它应类似这样（框 4）。

至于保持服务器最新，我是 Ports 提交者也是 Pkg 开发者，喜欢观察所有能观察的升级以发现问题。我也只是偏执，想确保升级顺利，所以我在服务器上手动运行 `pkg upgrade`，不使用自动化 crontab。我的系统上有许多在 Ports 之外构建的应用，所以我必须保留共享库直到那些应用能重新构建。这类似于 portupgrade 和 portmaster 加 `-p` 标志所做的事。示例可在我的 github 上找到。手动运行升级脚本对我来说不是问题，因为我只维护少数几台服务器。然而，Pkg 受 puppet、salt 和 ansible 支持。但要注意，包升级偶尔仍需要手工干预。这通常只发生在包的 origin 改变时，需要运行 `pkg set -o old/origin:new/origin`。最常见的情况是 Perl 之类更新时。你可以在脚本中通过运行 `pkg upgrade -Fy` 检测这种情况和其他冲突包情况，该命令会列出升级中所有冲突包。Ports 框架和 Pkg 目前没有自动处理被替换包的机制，但最终会实现。这些情况目前仍记录在 **/usr/ports/UPDATING** 文件中。你可以在 Ports svnweb 上关注此文件。

Pkg 讨论在 freebsd-pkg 邮件列表和 Freenode 上 IRC 频道 #pkgng 进行。Poudriere 讨论在 Freenode 上 IRC 频道 #poudriere 进行。有任何问题或想法欢迎随时来访。

## 框 3：典型 pkg 仓库配置

```ini
MYREPO: {
  url: "http://url.to.your.repository/${ABI}",
  enabled: true,
  signature_type: "pubkey",
  pubkey: "/usr/local/etc/pkg/repos/MYREPO.pem"
}

# 可选：禁用 FreeBSD 仓库。
FreeBSD: {
  enabled: false
}
```

## 框 4：ABI 符号链接设置

```sh
# ls -al /usr/local/poudriere/data/packages
total 19
drwxr-xr-x 7 root wheel 15 Jul 11 2013 ./
drwxr-x—x 26 root wheel 32 Mar 2 01:13 ../
drwxr-xr-x 2 root wheel 2 Jul 8 2013 10amd64/
drwxr-xr-x 7 root wheel 39 Mar 2 10:05 83amd64/
drwxr-xr-x 7 root wheel 39 Mar 2 10:30 83i386/
lrwxr-xr-x 1 root wheel 7 Jul 8 2013 freebsd:10:x86:64@ -> 10amd64
lrwxr-xr-x 1 root wheel 6 Jan 26 2013 freebsd:8:x86:32@ -> 83i386
lrwxr-xr-x 1 root wheel 7 Jan 26 2013 freebsd:8:x86:64@ -> 83amd64
```

## 元包

在管理我的服务器时，我使用元包。这类包只依赖其他包，自身不安装任何文件。它们可由不实际构建的 Port 创建。在服务器上安装元包，可以保证每台服务器都安装相同的包，只要它们各自只安装你创建的那少数几个元包。使用元包还让移除不需要的包变得简单得多。Pkg 有一项功能可跟踪哪些包是显式请求安装的、哪些是作为依赖自动引入的。如果你在系统上对所有包执行 `pkg install`，那么 `pkg autoremove` 永远不会移除任何东西。反之，如果你只安装一个元包而它引入了 100 个依赖，日后更新该元包使其不再需要某个包时，`pkg autoremove` 会正确检测并移除那个包。

我更进一步，使用基于角色的元包。例如，我有一个基础元包，包含所有我服务器都需要的包，比如配置管理工具。然后我为每种服务器类型准备元包，只依赖它所需的内容和基础元包。例如，我的 DNS 服务器使用 dns-server 元包，依赖 bind 和其他一些 DNS 工具以及基础元包。我的 Web 应用 Jail 都使用 web-server 元包，依赖 PHP、nginx 和基础元包。这简化了服务器管理，因为只有少数几个包需要显式安装和监控。我的方法是使用 Ports 树 overlay 创建元包，但你也可以直接使用 pkg manifest 格式创建包。对于 Ports 树 overlay，我只是把 git 跟踪的树 rsync 到 Ports 的 SVN 检出之上。对于 Poudriere，只需指定要构建的元包，它会一并构建所有需要的依赖。我在博客上有一份使用元包管理基于角色服务器的指南。

---

**Bryan Drewery** 自 2004 年起用 FreeBSD 管理共享托管服务。他于 2012 年作为 Ports 提交者加入项目，是 Portmgr 成员，最近成为 Src 提交者。他是 Portupgrade、Portmaster 的当前上游维护者，也是 Pkg 和 Poudriere 的开发者。在 Portmgr 中，他协助处理 Ports 框架、管理包构建系统、包构建及在系统上测试 Ports 补丁。
