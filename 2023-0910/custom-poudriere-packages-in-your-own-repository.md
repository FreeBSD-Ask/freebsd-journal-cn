# 在你自己的仓库中定制 Poudriere 源

![image](https://github.com/Canvis-Me/freebsd-journal-cn/assets/55122738/ce755f34-18b7-4f99-a338-76aafed8d9da)


- 原文链接：<https://freebsdfoundation.org/wp-content/uploads/2023/11/reushling_poudrier.pdf>
- 作者：BENEDICT REUSCHLING
- 译者：Canvis-Me & ChatGPT

我对这些人深表感激，是他们使在 FreeBSD 上进行软件移植和安装软件变得如此简单。而其他类 Unix 系统则需要手动安装大量的库和依赖，更不用说基于源码的程序，但 BSD 通常不需要这些。一个简单的 `pkg install foo` 对我来说就足够了，最终结果是安装了 foo。这些预编译的软件来自官方的 FreeBSD 软件分发系统，并已配置了适用于大多数默认情况下的默认选项。

不幸的是，在我的工作场所中，默认设置并不适合。我们需要的是 LDAP 支持，以查询我们的中央数据库进行身份验证。很多 ports 通过“make config”提供 LDAP 支持，但基于它们的软件在默认情况下没有启用该选项。因此，当我想要安装带有 LDAP 支持的 PostgreSQL 15 时，我无法通过 `pkg install postgresql15-server` 实现这一需求，因为安装的二进制文件对数据库中的 LDAP 身份验证一无所知。

第一个解决方案是编译我的自定义软件。我使用最新的 ports 进行获取。

```
# portsnap auto
```

然后，我进入到有关 port 的目录，即 `/usr/ports/databases/postgresql15-server`，并运行

```
# make config-recursive
```

接着，我仔细选择与 LDAP 相关的所有选项。如果因为有太多的依赖软件而变得太繁琐，我也可以添加

```
OPTIONS_SET=LDAP
```

到我的 `/etc/make.conf` 中，这样就会自动选择此选项（如果可用）用于我编译的任何 ports。然后，我可以运行

```
# make package
```

并等待使用这些自定义选项编译软件。可以在 `work/pkg` 子目录中找到生成的软件，并通过 `pkg install ./packagename` 进行安装。

这一切都很棒，但如果涉及多个类似的软件呢？或者，如果你想始终安装最新的软件版本，但又不想在管理的每个独立的计算机上都这样做怎么办？人们早晚想要自动编译自己的软件，并在其网络中提供它们。为了解决这些情况，poudriere 应运而生。

Poudriere（法语意为火药桶）是一个在 jail 中编译定制软件、解决它们之间的依赖关系，并将它们作为 FreeBSD 存储库提供的框架。ports 开发人员使用它来测试 ports 的新版本（因此有了火药桶的比喻，因为这样的编译有时可能会失败或者干脆爆炸）。你不必是 ports 开发人员，也不必了解编译的任何内容来使用 poudriere。即使是最微小的依赖，每个 port 都在单独的 jail 中编译（会自动创建和删除），以确保每次都有一个干净的编译环境。它还能并行地编译 ports，这是非常有利的，因为需要编译的软件数量可能并不显而易见（依赖项可能首先编译其他依赖的 ports，然后一发不可收拾）。

你需要一台 FreeBSD 机器来编译这些自定义软件，最好配备大量的 CPU 和内存以及良好的 I/O。Poudriere 对系统的压力很大，因为它尝试并行运行很多这些软件编译，这也是为什么它也可以看作是系统基准测试工具的原因。

让我们开始创建我们自己的 poudriere 机器。我将我的编译机命名为 poudriere（因为我今天很有创意），并通过提示符 `poudriere#` 区分本文中的其他命令。

## Poudriere 设置

首先，安装 poudriere 软件。有一个 poudriere-devel 版本，其中包含一些尚未包含在常规版本中的修复。到目前为止，我个人的经验是两者都很好，所以我选择了这个版本：

```
poudriere# pkg install poudriere-devel
```

我们将仔细查看安装后位于 `/usr/local/etc/poudriere.conf` 的 poudriere 配置文件。我们在其中进行一些更改，这取决于我们的系统资源以及我们是否使用 ZFS（强烈推荐使用）。该文件中的注释将帮助你了解这些选项的作用：

```
ZPOOL=zroot
FREEBSD_HOST=ftp://ftp.freebsd.org
RESOLV_CONF=/etc/resolv.conf
BASEFS=/usr/local/poudriere
POUDRIERE_DATA=/usr/local/poudriere/data
USE_PORTLINT=no
USE_TMPFS=yes
MAX_FILES=2048
DISTFILES_CACHE=/usr/ports/distfiles
CHECK_CHANGED_OPTIONS=verbose
CHECK_CHANGED_DEPS=yes
CCACHE_DIR=/var/cache/ccache
PARALLEL_JOBS=65
PREPARE_PARALLEL_JOBS=5
ALLOW_MAKE_JOBS=yes
KEEP_OLD_PACKAGES=yes
KEEP_OLD_PACKAGES_COUNT=3
```

你需要提供 poudriere 所使用的 ZFS 池的名称。ZFS 可以快速克隆并删除编译的 jails，而不必为每个 port 从基本的 jail 复制它们。如果尚不存在，请创建在 `POUDRIERE_DATA`、`DISTFILES_CACHE` 和 `CCACHE_DIR` 中定义的必要数据集。调整 `MAX_FILES`、`PARALLEL_JOBS` 和 `PREPARE_PARALLEL_JOBS` 的值以适应你的系统。我建议一开始将它们设定为较低的值，然后在完成最初的几个 poudriere 编译之后再逐渐增加它们。如果将这些值设置为最大值，poudriere 的编译可能会使你的系统陷入困境，因此请为其他任务（例如 SSH 登录会话）保留一些资源。

## Ccache 设置

这部分完全是可选的，poudriere 可以在没有它的情况下正常运行。这只是一种优化，可以加速将来的编译。使用 ccache，编译后的二进制文件被缓存（因此称为编译器缓存），如果它们没有更改，就会使用缓存版本。这将给后续的编译提供巨大加速，因为编译时间缩短了，在某些情况下可能会大幅减少。我们已经在配置文件中告诉 poudriere 使用 ccache，但首先我们需要安装和配置 ccache。如果缺少 ccache，poudriere 将能够正常运行，但不会利用编译缓存。

在运行 `pkg install` 命令之前，我们创建一个单独的数据集，因为否则 pkg 将为其创建一个目录。

```
poudriere# zfs create \
-o mountpoint=/var/cache/ccache \
-o compression=lz4 \
-o recordsize=1M
zroot/var/cache/ccache
```

通过这些选项，我能够将缓存的磁盘占用空间减少三分之一，但这在很大程度上取决于你编译的软件和编译频率。

现在让我们安装 ccache 软件：

```
poudriere# pkg install ccache
```

接下来，让我们编辑 ccache 配置文件。它位于许多不同的目录中（或者说，会被搜索）。我们将创建一个参考文件，并简单地从其他位置链接到它，以减轻维护负担。幸运的是，你很少会触及这个文件，但如果你确实这样做，符号链接可以确保每个位置都会随之更改。

```
poudriere# cat << EOF > /var/cache/ccache/ccache.conf
max_size = 0
cache_dir = /var/cache/ccache
base_dir = /var/cache/ccache
hash_dir = false
EOF
```

`max_size` 选项可以将缓存大小限制为一定数量，但对于 0，它可以使用其所需的所有磁盘空间。我对此并不太担心，因为 ZFS 压缩在这里发挥了很好的作用。如果磁盘空间不足，我甚至可以在数据集 `zroot/var/cache/ccache` 上设置配额。

`cache_dir` 和 `base_dir` 定义了缓存的位置。在我们的情况下，它们指向我们的数据集。将 `hash_dir` 选项设置为 `false` 可以增加缓存命中率，但激活它会使调试变得困难。这是我为了更好的性能愿意付出的权衡。有关此选项和其他选项的详细信息，请参阅 ccache(1)。在 FreeBSD 论坛的一个主题中也讨论了这个问题：<https://forums.freebsd.org/threads/howto-speeding-up-poudriere-build-times.69431/>

让我们在 ccache 期望找到它的位置创建对此配置文件的符号链接。

```
poudriere# ln -s /var/cache/ccache/ccache.conf /root/.ccache/ccache.conf
poudriere# ln -s /var/cache/ccache/ccache.conf /usr/local/etc/ccache.conf
```

就是这样了。你可以使用以下命令检查缓存的状态：

```
poudriere# ccache -s
```

在完成了一些编译之后。现在让我们回到 poudriere…

## 配置 Poudriere 

我们已经提到 poudriere 将为构成软件的每个单独 port 运行 jail。为此，poudriere 需要知道软件应该在哪个架构和哪个 FreeBSD 版本上编译。版本之间存在细微差异，但 poudriere 可以在它们互不干扰的情况下并排编译不同版本。这些 jail 互相独立。此外，你可以通过这种方式在功能强大的 amd64 服务器上为你的树莓派编译软件。

让我们从为 amd64 和 FreeBSD 13.2 创建 jail 开始，因为这是本文写作时的当前版本。

```
poudriere# poudriere jail -c -j 132x64 -v 13.2-RELEASE
```

使用 `-c` 参数，我们要求 poudriere 为编译创建一个新的基本 jail，并提供应该使用的架构和版本。你甚至可以通过添加 `-m ftp-archive` 选项为不受支持的 FreeBSD 版本（例如 FreeBSD 12.1）编译软件。

你可以使用以下命令列出可用的编译 jail：

```
poudriere# poudriere jails -l
JAILNAME  VERSION       ARCH   METHOD  TIMESTAMP            PATH
132x64    13.2-RELEASE  amd64  http    2023-05-03 07:54:58  /usr/local/poudriere/ jails/132x64
```

你的输出可能会有所不同，你可以为 jail 指定任何你喜欢的名称。我鼓励你包含版本和架构信息，以便将其与你可能拥有的其他 poudriere jail 区分开。

需要 ports 来编译软件。这是一个简单的命令，如下所示：

```
poudriere# poudriere ports -c -p default
```

同样，我们可以列出有关这个 ports 树的一些详细信息：

```
poudriere# poudriere ports -l
PORTSTREE   METHOD     TIMESTAMP             PATH
default     git+https  2023-10-01 12:00:02   /usr/local/poudriere/ports/default
```

我们也可以拥有多个可用的 ports。也许我们想要在更新方面放慢步伐，只编译上个季度的 ports，而不是最新的。使用 poudriere 和参数 `-p`，这一切都是可能的。

现在我们需要一个供 poudriere 编译的软件列表。“哦，我喜欢这个 port，而且我总是安装这个 shell，那个编辑器。还有 tmux…”这可能很快就能列出一个清单，但它不包含依赖项（运行或编译列出的软件所需的其他软件）。Poudriere 会为你解决这些依赖关系。

将其放在 `/usr/local/etc/poudriere.d/` 中，作为一个文本文件（同样，任何你喜欢的名称），每个 port 及其类别都放在单独的行上。

我的 port 清单看起来像这样（随着我越来越喜欢 poudriere，它还在不断增长）：

```
poudriere# cat /usr/local/etc/poudriere.d/pkglist.txt
databases/postgresql15-server
databases/postgresql15-client
databases/postgresql15-contrib
```

这些是需要 LDAP 支持的 ports，记得吗？你可以从一些简单的东西开始，比如 games/sl 或 shells/fish，以免花费太多时间等待 poudriere 完成编译。

一个更好的方法是让软件系统创建一个软件列表。登录到安装了所有这些软件的系统，并运行 `pkg_info`。这将创建一个包括依赖关系的列表。已经是相当长的列表了！在你想要删除软件的情况下，你总是可以将其减少。软件可能也太大而无法编译（例如 libreoffice）。

定义要编译但还没有配置的 ports。我们仍然需要告诉 poudriere 为每个 port 设置哪些选项。但我们只需要做这一次，对于将来的编译，poudriere 将保存我们选择的选项并重用它们，甚至用于软件的将来版本。这相当于在文章开头我们在 ports 目录中运行 "make config"。

```
poudriere# poudriere options \
-j 132x64 \
-p default \
-f /usr/local/etc/poudriere.d/pkglist.txt
```

这为整个 ports 列表定义了选项。你也可以使用以下方式为单个 port 执行此操作：

```
poudriere# poudriere options \
-j 132x64 \
-p default \
databases/postgresql15-server
```

这完成了 poudriere 的准备工作。现在我们可以开始我们的第一次软件编译。

## 编译和分发软件

要启动软件编译，请为 poudriere 的 bulk 子命令提供 jail、ports 树和软件列表：

```
poudriere# poudriere bulk \
-j 132x64 \
-p default \
-f /usr/local/etc/poudriere.d/pkglist.txt
```

坐下来享受自动化的软件编译吧。你可以按下 CTRL-T 键查看编译状态的中间输出。我建议你在 tmux 会话中运行此操作，这样你可以在长时间编译时离开，并在稍后返回而不是在退出 shell 时停止整个过程。

编译完成后，poudriere 将告诉你它已经创建了一个包含列表中软件的存储库。要利用此存储库，我们需要通过定义新的存储库位置告知我们的 FreeBSD 它的存在。

```
poudriere# mkdir -p /usr/local/etc/pkg/repos
```

一个名为 `local.conf` 的文件（同样，可以是任何名称，看到这里的主题了吧？）包含了我自己的存储库配置。

```
poudriere# cat /usr/local/etc/pkg/repos/local.conf
Poudriere: {
        url: “file:///usr/local/poudriere/data/packages/132x64-default”,
        priority: 23,
}
```

我的存储库被称为 Poudriere（同样地，应该是你自己的名字，你懂的），我在 url: 部分定义了它的位置。我从 poudriere 的编译输出中获得了位置。priority 定义了使用这些存储库的顺序。默认情况下，会使用上游的 FreeBSD.org 存储库。但由于我们希望我们自己的软件具有优先权，只需为其分配一个高于零的优先级，以使其首先使用我们自己的软件存储库。

你还可以通过添加以下内容完全禁用 FreeBSD 存储库：

```
FreeBSD: {
   enabled: no
}
```

但起初你也可以同时运行两者。

使用以下命令检查正在使用的存储库：

```
poudriere# pkg -vvv
```

底部应该包含我们自己的存储库定义。

让我们运行 `pkg update`：

```
Updating Poudriere repository catalogue...
Fetching meta.conf: 100% 163 B 0.2kB/s 00:01
Fetching packagesite.pkg: 100% 68 KiB 69.8kB/s 00:01
Processing entries: 100%
Poudriere repository update completed. 232 packages processed.
All repositories are up to date.
```

请注意软件的数量。这绝对是我们自己的本地存储库，因为主 FreeBSD 存储库在此时包含了超过 31500 个软件，而这个只有 232 个。这些是我们根据自己的 pkglist.txt 中的列表编译的软件，以及使这些软件编译和运行所需的依赖关系。查看可用存储库的另一种方式是通过 `pkg stats`，该命令在我的系统上生成以下输出：

```
Local package database:
        Installed packages: 120
        Disk space occupied: 2 GiB

Remote package database(s):
        Number of repositories: 2
        Packages available: 34190
        Unique packages: 34190
        Total size of packages: 117 GiB
```

在这个例子中，我在 `/usr/local/etc/pkg/repos/local.conf` 里保留了 `FreeBSD:` 条目

```
FreeBSD: {
   enabled: yes
}
```

现在两个存储库都被使用，但本地存储库被优先考虑，因为它的优先级较高。如果软件不在本地存储库中，那么将根据它们的优先级检查其他存储库。在这种情况下，如果一切都失败，我们就会回退到官方存储库。

```
poudriere# pkg upgrade
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
Updating Poudriere repository catalogue...
Fetching meta.conf: 100% 163 B 0.2kB/s 00:01
Fetching packagesite.pkg: 100% 73 KiB 75.0kB/s 00:01
Processing entries: 100%
Poudriere repository update completed. 249 packages processed.
All repositories are up to date.
Checking for upgrades (40 candidates): 100%
Processing candidates (40 candidates): 100%
The following 1 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
         gdbm: 1.23 [FreeBSD]

Number of packages to be installed: 1
209 KiB to be downloaded.

Proceed with this action? [y/N]:
```

你总是可以通过方括号后面列出的存储库来确定特定软件来自哪里。gdbm 软件来自官方 FreeBSD 存储库。不过要小心这种混合。在某些情况下，你自定义编译的软件和官方 FreeBSD 存储库的软件可能无法一起工作，因为你可能删除了其他软件所需的一些选项。请谨慎测试或决定完全使用 poudriere。在这种情况下，设置

```
FreeBSD: {
   enabled: no
}
```

或者完全从 `/usr/local/etc/pkg/repos/local.conf` 中删除该存储库定义。

那么你其他运行 FreeBSD 且日益增长的机器呢？特别是对于虚拟机或嵌入式系统，你不会在每台机器上运行一个单独的 poudriere 编译器。一种方法是通过 NFS 将 repo URL 共享到每台机器。更好的方法是配置一个中央的、强大的 poudriere 编译机器，通过 http 共享其软件作为存储库。你网络中的其他 FreeBSD 机器可以像添加官方 FreeBSD 存储库一样将其添加。与 NFS 共享相比，这种方法的额外好处是你可以对这些软件进行签名。这确保了密码学上的完整性和对源的信任（这些软件确实是你自定义编译的）。这样，机器会检查编译机器的公钥是否与它们必须确保软件来自真实源的记录匹配。让我们进行接下来的设置。

为了给其他机器提供软件，我们将 nginx 安装为 web 服务器。如果其他 web 服务器能够向客户端共享一个 URL 作为文档根目录，则其他 web 服务器也可以工作。在“poudriere bulk”运行期间，我们还可以共享编译日志，以查看当前编译的进度并调试失败的 ports。

```
poudriere# pkg install nginx
```

当然，这个 nginx 也可以来自我们自己的本地存储库，如果我们想对其默认软件配置进行任何更改的话。注意：我们没有使用 SSL 对流量本身进行加密，而仅仅是软件存储库本身。为传输加密添加 SSL 是留给读者的一个练习，但并不复杂。

在编辑了 `/usr/local/etc/nginx.conf` 之后，它应该包含以下部分：

```
server {
  listen 80 default;
  server_name domain.or.ip.address;
  root /usr/local/share/poudriere/html;

  location /data {
    alias /usr/local/poudriere/data/logs/bulk;
    autoindex on;
  }
  location /packages {
    root /usr/local/poudriere/data;
    autoindex on;
  }
}
```

在保存并退出 `nginx.conf` 后，我们需要进行一个更改，以便我们的编译日志也能在浏览器中正确显示。这意味着，当打开日志文件（.log 扩展名）时，它将不会被提供为下载，而是立即在浏览器中显示。为此，我们访问 `/usr/local/etc/nginx/mime.types`。找到以 `text/plain` 开头的行，并更改它以包括日志文件：

```
text/plain             txt log;
```

通过在 `/etc/rc.conf` 中为其添加一个条目，启用系统重新启动时 nginx 的启动：

```
poudriere# service nginx enable
```

现在启动服务：

```
poudriere# service nginx start
```

你可以通过将浏览器指向 `http://server_domain_or_IP` 来查找编译状态和日志。

对于存储库签名，我们创建了一些用于存储密钥和证书的目录。

```
poudriere# cd /usr/local/etc/ssl

poudriere# mkdir keys certs
```

这些密钥不应该落入其他人的手中，因此我们只允许 root 用户查看这个目录：

```
poudriere# chmod 0600 keys
```

接下来，我们开始生成用于 poudriere 存储库的 RSA 私钥。

```
poudriere# openssl genrsa -out keys/poudriere.key 4096
```

处理该密钥就像处理你前门的物理钥匙一样：绝对不要交给别人，甚至不要让任何人看到它。如果密钥受到攻击，攻击者可以将任何类型的软件注入到你的机器中，这将让你的一天变得非常不愉快。

接下来，我们根据刚刚生成的私钥创建一个证书（公钥）：

```
poudriere# openssl rsa -in keys/poudriere.key -pubout -out
certs/poudriere.cert
```

所有必要的密钥现在都可用了。接下来，我们需要让 poudriere 知道它们，以便正在编译的软件被用它们签名。这是在 `/usr/local/etc/poudriere.conf` 中完成的，我们在之前的第一个基本 poudriere 配置中访问过这个文件。找到以 `PKG_REPO_SIGNING_KEY` 开头的行，并将其设置为我们生成的私钥的位置。最终结果如下：

```
PKG_REPO_SIGNING_KEY=/usr/local/etc/ssl/keys/poudriere.key
```

我们想要更改的另一行是一些额外的可点击链接，如下所示：

```
URL_BASE=http://my.domain.or.IP
```

由 `URL_BASE` 定义的网站显示了有关过去和当前软件编译的许多统计信息。在编译过程中，它会刷新自身以显示编译状态。你还可以深入了解每个编译，查看哪些 ports 已成功编译，被跳过或被忽略。非常不错！

添加了这些行之后，poudriere 就可以签署新编译的软件。确实，在下一个

```
poudriere# poudriere bulk \
-j 132x64 \
-p default \
-f /usr/local/etc/poudriere.d/pkglist.txt
```

运行之后，我在输出中看到了这些新行：

```
[132x64-default] [2023-08-06_09h24m25s] [load_priorities:] Queued: 0 Built: 0 Failed: 0 Skipped: 0 Ignored: 0 Fetched: 0 Tobuild: 0 Time: 00:00:08
[00:00:08] Recording filesystem state for prepkg... done
[00:00:08] Creating pkg repository
[00:00:09] Signing repository with key: /usr/local/etc/ssl/keys/poudriere.key
Creating repository in /tmp/packages: 100%
Packing files for repository: 100%
[00:00:13] Signing pkg bootstrap with method: pubkey
[00:00:13] Committing packages to repository: /usr/local/poudriere/data/packages/
132x64-default/.real_1691306678 via .latest symlink
[00:00:13] Removing old packages
```

是时候重新访问 `/usr/local/etc/pkg/repos/local.conf` 并在签名类型旁边添加证书了。编辑后，文件应该如下所示：

```
Poudriere: {
        url: “file:///usr/local/poudriere/data/packages/132x64-default”,
        priority: 23,
        mirror_type: “srv”,
        signature_type: “pubkey”,
        pubkey: “/usr/local/etc/ssl/certs/poudriere.cert”,
        enabled: yes
}
```

## 客户端配置

现在是时候将另一台机器添加到我们的签名软件存储库中了。登录到你希望在其中使用自定义、新编译软件的 FreeBSD，并为 poudriere 证书和存储库配置创建目录。

```
clienthost# cd /usr/local/etc/
clienthost# mkdir ssl/certs ssl/keys
clienthost# mkdir pkg/repos
```

然后，安全地将编译机器上 `/usr/local/etc/ssl/certs/` 中的 `poudriere.cert` 复制到我们刚刚创建的目录中。你可以使用 `scp(1)` 将其从主机传输到客户端：

```
poudriere# scp /usr/local/etc/ssl/certs/poudriere.cert
clienthost:/usr/local/etc/ssl/certs/
```

接下来，我们定义一个新的存储库位置，就像上面一样。唯一的区别在于 URL。而编译机器可以使用 `file:///`  来引用本地文件系统，远程机器将不得不通过 http 连接到我们的 nginx。`mirror_type` 也不同，但除此之外，配置完全相同，如下所示：

```
Poudriere: {
        url: “http://my.domain.or.ip/packages/132x64-default”,
        mirror_type: “http”,
        signature_type: “pubkey”,
        pubkey: “/usr/local/etc/ssl/certs/poudriere.cert”,
        priority: 23,
        enabled: yes
}

clienthost# pkg update
pkg update
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
Updating Poudriere repository catalogue...
[clienthost] Fetching meta.conf:   100%   163 B   0.2kB/s   00:01
[clienthost] Fetching packagesite.pkg:   100%   73 KiB   75.0kB/s   00:01
Processing entries: 100%
Poudriere repository update completed. 247 packages processed.
All repositories are up to date.
```

告诉我有 247 个软件被处理的那一行肯定是指它能找到我的其他存储库和其中的软件。运行

```
clienthost# pkg upgrade
```

通过确认它正在引用我的自定义存储库名称，我证实它可以工作：

```
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
Updating Poudriere repository catalogue...
Poudriere repository is up to date.
All repositories are up to date.
New version of pkg detected; it needs to be installed first.
The following 1 package(s) will be affected (of 0 checked):

Installed packages to be UPGRADED:
pkg: 1.19.1_1 -> 1.20.5 [Poudriere]

Number of packages to be upgraded: 1
The process will require 2 MiB more space.
9 MiB to be downloaded.

Proceed with this action? [y/N]:
```

成功了！该软件已从编译机器下载，并使用了我为该软件制作的自定义配置。知道我不必在运行的每个性能较弱的 FreeBSD VM 上重新编译 llvm，感觉非常迅速和令人满意。那就是强大的编译服务器的用途。

这是使用混合的 FreeBSD 和自定义存储库进行的 `pkg upgrade` 的输出：

```
New packages to be INSTALLED:
        cyrus-sasl: 2.1.28 [Poudriere]
        gdbm: 1.23 [FreeBSD]
        icu: 73.2,1 [FreeBSD]
        openldap26-client: 2.6.5 [Poudriere]
```

请注意，混合这两种软件可能会导致一些问题，因为它们之间的交互方式。一个软件期望另一个软件具有特定的选项设置，但不是在另一个存储库中的特定软件的一部分，可能会导致编译或安装错误，从而导致应用程序不能按预期工作。理想情况下，应该使用单一的存储库源，无论是上游存储库还是自己内部的存储库。

另外，请确保如果你正在为“latest”分支编译软件并将其分发到客户端，那么这些客户端也在 `/etc/pkg/freebsd.conf` 中使用 latest。否则，软件的版本会比季度版更新。我曾遇到过一种情况，上面的软件被愉快地更新和安装，但如果我运行了“pkg autoremove”，它们就被列为需要删除的软件。然后我再次升级它们，恶性循环就完成了。

当你简单地将存储库添加到一台机器而没有证书时，`pkg update` 将抱怨缺少证书：

```
clienthost# pkg update
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
Updating Poudriere repository catalogue...
Fetching meta.conf: 100%   163 B   0.2kB/s   00:01
Fetching packagesite.pkg: 100%   76 KiB   77.6kB/s   00:01
pkg: openat(/usr/local/etc/ssl/certs/poudriere.cert): No such file or directory
pkg: rsa_verify(cannot read key): No such file or directory
pkg: Invalid signature, removing repository.
Unable to update repository Poudriere
Error updating repositories!
```

在你的编译服务器上可以执行的最后一项操作是创建一个定期任务来更新 poudriere 的 ports：

```
poudriere# poudriere ports -u -p default
```

这次我们使用 `-u` 而不是 `-c`，以指示我们要更新现有的 ports。我们可以每天执行一次，甚至更早，这取决于你希望软件有多新鲜。然后，让 poudriere 运行批量编译：

```
poudriere# poudriere bulk \
-j 132x64 \
-p default \
-f /usr/local/etc/poudriere.d/pkglist.txt
```

我每天晚上都会运行这个任务，这样第二天早上我就有新编译的软件等着我。我始终可以使用 nginx 提供的 poudriere 网站检查编译状态。

当 poudriere 为不同的架构编译许多不同的软件时，它真正发挥作用。它使用 qemu 为非 amd64 架构（如我的树莓派）编译软件。由于一切都是并行进行的，因此我可以更快地编译自己的软件，而其余的可以来自默认的 FreeBSD 存储库——因为我不需要在这些软件上设置特殊的选项。随着时间的推移，你自己的自定义软件清单肯定会增长。对于使用带有你自己证书的存储库的机器数量也是如此。你完全可以控制何时以及哪些软件进行更新，甚至可以使用 Poudriere 帮助编译新的 ports。

---

BENEDICT REUSCHLING 是 FreeBSD 项目的文档提交者，并且是文档工程团队的成员。在过去，他曾在 FreeBSD 核心团队中任职两届。他在德国达姆斯塔特应用科学大学管理一个大数据集群。他还为本科生开设了“Unix for Developers”课程。Benedict 是每周 bsdnow.tv 播客的主持人之一。
