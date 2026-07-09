# 在你自己的仓库中定制 Poudriere 源

![image](../png/2023-0910/custom-poudriere-packages-1.png)

- 原文链接：<https://freebsdfoundation.org/wp-content/uploads/2023/11/reushling_poudrier.pdf>
- 作者：BENEDICT REUSCHLING
- 译者：Canvis-Me & ChatGPT

我深深感激那些让 FreeBSD 上的 Ports 和软件包安装变得如此简单的人。其他类 Unix 系统需要手动安装大量库和依赖，更不用说以文本文件形式存在的软件包来源，而 BSD 通常不需要这些。简单的 `pkg install foo` 就能替我完成一切，最终 foo 也就装好了。这些预编译软件包来自官方 FreeBSD 软件包分发系统，配置了适用于大多数默认场景的默认选项。

不幸的是，我的工作场所里默认设置并不合适。我们需要 LDAP 支持，用于查询中央数据库验证身份。很多 Ports 通过 `make config` 提供 LDAP 支持，但基于它们的软件包默认未启用该选项。因此，当我想安装带 LDAP 支持的 PostgreSQL 15 时，无法通过 `pkg install postgresql15-server` 完成，因为安装的二进制文件对数据库的 LDAP 身份验证一无所知。

第一种解决方案是编译我自己的自定义软件包。我用以下命令获取最新的 ports 树：

```sh
# portsnap auto
```

然后，我进入相关 port 的目录，即 **/usr/ports/databases/postgresql15-server**，并运行

```sh
# make config-recursive
```

接着，我仔细选择与 LDAP 相关的所有选项。如果因为依赖软件包过多而显得过于繁琐，我也可以添加

```sh
OPTIONS_SET=LDAP
```

到我的 **/etc/make.conf** 中，这样我编译的任何 ports 都会自动启用此选项（如果可用）。然后，我可以运行

```sh
# make package
```

并等待使用这些自定义选项编译软件包。生成的软件包可在 **work/pkg** 子目录中找到，通过 `pkg install ./packagename` 安装。

这一切都很棒，但如果涉及多个类似的软件包呢？或者，如果你想始终安装最新的软件包版本，但又不想在管理的每台独立机器上都这样做怎么办？迟早人们会想要自动编译自己的软件包，并在自己的网络中提供它们。针对这些场景，poudriere 应运而生。

Poudriere（法语意为火药桶）是一个在 Jail 中编译定制软件包、解决它们之间的依赖关系，并将它们作为 FreeBSD 存储库提供的框架。Ports 开发人员使用它来测试 Ports 的新版本（因此有了火药桶的比喻，因为这样的编译有时可能会爆炸或者干脆失败）。你不必是 Ports 开发人员，也不必了解任何构建模块就能使用 poudriere。每个 port（即使是最微小的依赖）都在单独的 Jail 中编译（会自动创建和删除），以确保每次都有干净的编译环境。它还能并行编译 Ports，这非常有用，因为需要编译的软件包数量可能并不显而易见（依赖项可能先编译其他依赖的 Ports，然后越陷越深）。

你需要一台 FreeBSD 机器来编译这些自定义软件包，最好配备大量 CPU、内存和良好的 I/O。Poudriere 会给系统带来很大压力，因为它试图并行运行很多软件包编译，因此它也可以看作系统基准测试工具。

开始创建我们自己的 poudriere 机器吧。我将编译机命名为 poudriere（因为我今天很有创意），并通过提示符 `poudriere#` 与本文中的其他命令区分。

## Poudriere 设置

首先，安装 poudriere 软件包。poudriere-devel 版本带有一些常规版本尚未包含的修复。到目前为止，我对两者的体验都很好，所以我选择了这个版本：

```sh
poudriere# pkg install poudriere-devel
```

下面仔细查看安装后位于 **/usr/local/etc/poudriere.conf** 的 poudriere 配置文件。我们在其中做几处修改，取决于系统资源和是否使用 ZFS（强烈推荐）。该文件中的注释会帮助你了解这些选项的作用：

```sh
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

你需要提供 poudriere 使用的 ZFS 池名称。ZFS 可以快速克隆并删除编译 Jail，而不必为每个 port 从基本 Jail 复制。请创建 `POUDRIERE_DATA`、`DISTFILES_CACHE` 和 `CCACHE_DIR` 中定义的必要数据集（如果尚不存在）。调整 `MAX_FILES`、`PARALLEL_JOBS` 和 `PREPARE_PARALLEL_JOBS` 的值以适应你的系统。我建议一开始设得低一些，完成最初几次 poudriere 编译后再逐渐增加。如果这些值拉到最大，poudriere 编译可能让系统不堪重负，因此请为其他任务（例如 SSH 登录会话）保留一些资源。

## Ccache 设置

这部分完全是可选的，没有它 poudriere 也能正常运行。它只是用来加速后续编译的优化手段。使用 ccache 时，编译后的二进制文件会被缓存（因此叫编译器缓存），如果未更改，就使用缓存版本。这能给后续编译带来巨大加速，因为编译时间缩短了，某些情况下会大幅减少。我们已经在配置文件中告诉 poudriere 使用 ccache，但首先需要安装和配置 ccache。如果缺少 ccache，poudriere 仍能正常运行，只是不会利用编译缓存。

运行 `pkg install` 命令之前，我们创建一个单独的数据集，否则 pkg 会为它创建目录。

```sh
poudriere# zfs create \
-o mountpoint=/var/cache/ccache \
-o compression=lz4 \
-o recordsize=1M
zroot/var/cache/ccache
```

通过这些选项，我将缓存的磁盘占用减少了三分之一，但这在很大程度上取决于你编译的软件包和编译频率。

现在安装 ccache 软件包：

```sh
poudriere# pkg install ccache
```

接下来编辑 ccache 配置文件。它位于许多不同的目录（或者说，会在这些目录中被搜索）。我们创建一个参考文件，并简单地从其他位置链接到它，以减轻维护负担。幸运的是，你很少会动这个文件，但一旦修改，符号链接会确保每个位置都随之更改。

```sh
poudriere# cat << EOF > /var/cache/ccache/ccache.conf
max_size = 0
cache_dir = /var/cache/ccache
base_dir = /var/cache/ccache
hash_dir = false
EOF
```

`max_size` 选项可以将缓存大小限制为一定数量，但设为 0 时，它可以使用所需的全部磁盘空间。我并不太担心，因为 ZFS 压缩在这里效果不错。如果磁盘空间不足，我甚至可以在数据集 **zroot/var/cache/ccache** 上设置配额。

`cache_dir` 和 `base_dir` 定义缓存的位置。这里它们指向我们的数据集。将 `hash_dir` 选项设为 `false` 可以增加缓存命中率，但启用它会增加调试难度。这是我为了更好性能愿意做的权衡。此选项及其他选项的详情请参阅 ccache(1)。FreeBSD 论坛的一个帖子也讨论了这个问题：<https://forums.freebsd.org/threads/howto-speeding-up-poudriere-build-times.69431/>

在 ccache 期望找到它的位置为该配置文件创建符号链接。

```sh
poudriere# ln -s /var/cache/ccache/ccache.conf /root/.ccache/ccache.conf
poudriere# ln -s /var/cache/ccache/ccache.conf /usr/local/etc/ccache.conf
```

就是这样了。你可以使用以下命令检查缓存的状态：

```sh
poudriere# ccache -s
```

完成几次编译后，可以使用该命令查看缓存状态。现在回到 poudriere…

## 配置 Poudriere

我们已经提到 poudriere 会为构成软件包的每个单独 port 运行 Jail。为此，poudriere 需要知道软件包应该在哪个架构和哪个 FreeBSD 版本上编译。版本之间存在细微差异，但 poudriere 可以让不同版本互不干扰地并排编译。这些 Jail 互相独立。此外，你可以通过这种方式在功能强大的 amd64 服务器上为树莓派编译软件包。

从为 amd64 和 FreeBSD 13.2 创建 Jail 开始，因为这是本文写作时的当前版本。

```sh
poudriere# poudriere jail -c -j 132x64 -v 13.2-RELEASE
```

使用 `-c` 选项，我们让 poudriere 为编译创建新的基本 Jail，并提供应该使用的架构和版本。你甚至可以通过添加 `-m ftp-archive` 选项为不受支持的 FreeBSD 版本（例如 FreeBSD 12.1）编译软件包。

你可以使用以下命令列出可用的编译 Jail：

```sh
poudriere# poudriere jails -l
JAILNAME  VERSION       ARCH   METHOD  TIMESTAMP            PATH
132x64    13.2-RELEASE  amd64  http    2023-05-03 07:54:58  /usr/local/poudriere/ jails/132x64
```

你的输出可能不同，你可以为 Jail 指定任何你喜欢的名称。我建议你包含版本和架构信息，以便与你可能拥有的其他 poudriere Jail 区分。

需要 ports 树作为软件包的基础。命令很简单，如下所示：

```sh
poudriere# poudriere ports -c -p default
```

同样，我们可以列出这个 ports 树的一些详细信息：

```sh
poudriere# poudriere ports -l
PORTSTREE   METHOD     TIMESTAMP             PATH
default     git+https  2023-10-01 12:00:02   /usr/local/poudriere/ports/default
```

我们也可以拥有多个可用的 ports 树。也许我们想放慢更新节奏，只编译上个季度的 ports，而不是最新的。使用 poudriere 和 `-p` 选项即可做到。

现在我们需要一份供 poudriere 编译的软件包列表。“哦，我喜欢这个 port，而且我总是安装这个 shell，那个编辑器。还有 tmux…”这样很快就能列出清单，但它不包含依赖项（运行或编译所列软件包所需的其他软件包）。Poudriere 会替你解决这些依赖关系。

将其放在 **/usr/local/etc/poudriere.d/** 中，作为文本文件（同样，任何你喜欢的名称），每个 port 及其类别单独占一行。

我的 port 清单看起来像这样（我越喜欢 poudriere，它就越长）：

```sh
poudriere# cat /usr/local/etc/poudriere.d/pkglist.txt
databases/postgresql15-server
databases/postgresql15-client
databases/postgresql15-contrib
```

这些是需要 LDAP 支持的 ports，记得吗？你可以从简单的开始，比如 games/sl 或 shells/fish，以免等 poudriere 完成编译花太多时间。

更好的办法是让软件包系统生成软件包列表。登录到装好所有这些软件包的系统，运行 `pkg_info`。这会生成一份包括依赖关系的列表。已经是相当长的列表了！如果原本就想删除某个软件包，你可以从列表中删掉。某些软件包也可能太大而不便编译（例如 libreoffice）。

定义要编译哪些 ports 并不会自动配置它们。我们仍需告诉 poudriere 为每个 port 设置哪些选项。但只需做一次，后续编译中，poudriere 会保存我们选定的选项并复用，甚至用于软件包的将来版本。这相当于文章开头我们在 ports 目录中运行的 `make config`。

```sh
poudriere# poudriere options \
-j 132x64 \
-p default \
-f /usr/local/etc/poudriere.d/pkglist.txt
```

这会为整个 ports 列表定义选项。你也可以用以下方式为单个 port 设置：

```sh
poudriere# poudriere options \
-j 132x64 \
-p default \
databases/postgresql15-server
```

poudriere 的准备工作到此完成。现在可以开始第一次软件包编译。

## 编译和分发软件包

要启动软件包编译，请为 poudriere 的 bulk 子命令提供 Jail、ports 树和软件包列表：

```sh
poudriere# poudriere bulk \
-j 132x64 \
-p default \
-f /usr/local/etc/poudriere.d/pkglist.txt
```

坐下来享受自动化的软件包编译吧。你可以按下 CTRL-T 键查看编译状态的中间输出。我建议你在 tmux 会话中运行此操作，这样长时间编译时可以离开，稍后返回，而不必退出 shell 时停止整个过程。

编译完成后，poudriere 会告诉你已经创建了包含列表中软件包的存储库。要使用此存储库，我们需要通过定义新的存储库位置告知 FreeBSD 它的存在。

```sh
poudriere# mkdir -p /usr/local/etc/pkg/repos
```

名为 `local.conf` 的文件（同样，可以是任何名称，看到这里的主题了吧？）包含我自己的存储库配置。

```sh
poudriere# cat /usr/local/etc/pkg/repos/local.conf
Poudriere: {
        url: "file:///usr/local/poudriere/data/packages/132x64-default",
        priority: 23,
}
```

我的存储库叫 Poudriere（同样地，应该是你自己的名字，你懂的），我在 url: 部分定义了它的位置。位置信息来自 poudriere 的编译输出。priority 定义使用这些存储库的顺序。默认会使用上游的 FreeBSD.org 存储库。但由于我们希望自己的软件包具有优先权，只需给它分配高于零的优先级，让它先使用我们自己的软件包存储库。

你还可以通过添加以下内容完全禁用 FreeBSD 存储库：

```sh
FreeBSD: {
   enabled: no
}
```

但你也可以一开始同时启用两者。

使用以下命令检查正在使用的存储库：

```sh
poudriere# pkg -vvv
```

底部应该包含我们自己的存储库定义。

运行 `pkg update`：

```sh
Updating Poudriere repository catalogue...
Fetching meta.conf: 100% 163 B 0.2kB/s 00:01
Fetching packagesite.pkg: 100% 68 KiB 69.8kB/s 00:01
Processing entries: 100%
Poudriere repository update completed. 232 packages processed.
All repositories are up to date.
```

请注意软件包的数量。这绝对是我们自己的本地存储库，因为主 FreeBSD 存储库此时包含超过 31500 个软件包，而这个只有 232 个。这些是我们根据自己的 `pkglist.txt` 列表编译的软件包，加上让这些软件包编译和运行所需的依赖项。查看可用存储库的另一种方式是用 `pkg stats`，该命令在我的系统上输出如下：

```sh
Local package database:
        Installed packages: 120
        Disk space occupied: 2 GiB

Remote package database(s):
        Number of repositories: 2
        Packages available: 34190
        Unique packages: 34190
        Total size of packages: 117 GiB
```

这个例子中，我在 **/usr/local/etc/pkg/repos/local.conf** 里保留了 `FreeBSD:` 条目

```sh
FreeBSD: {
   enabled: yes
}
```

现在两个存储库都启用，但本地存储库因优先级较高而优先使用。如果软件包不在本地存储库中，会根据优先级查找其他存储库。在这种情况下，如果其他都失败，我们就回退到官方存储库。

```sh
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

根据方括号内列出的存储库，你总能判断某个软件包来自哪里。gdbm 软件包来自官方 FreeBSD 存储库。不过要小心这种混合。某些情况下，你自定义编译的软件包和官方 FreeBSD 存储库的软件包可能无法一起工作，因为你可能删除了其他软件包所需的某些选项。请谨慎测试或决定完全使用 poudriere。这种情况下，设置

```sh
FreeBSD: {
   enabled: no
}
```

或者直接从 **/usr/local/etc/pkg/repos/local.conf** 删除该存储库定义。

那么其他越来越多运行 FreeBSD 的机器怎么办？特别是虚拟机或嵌入式系统，你不会在每台机器上运行单独的 poudriere 编译机。一种方法是通过 NFS 把 repo URL 共享给每台机器。更好的办法是配置一台中央的、性能强劲的 poudriere 编译机，通过 http 把它的软件包作为存储库共享。网络中其他 FreeBSD 机器可以像添加官方 FreeBSD 存储库一样添加它。相比 NFS 共享，这种办法的额外好处是你可以对这些软件包签名。这能保证密码学完整性和对源的信任（这些软件包确实是你自定义编译的）。这样，机器会检查编译机的公钥是否与它们持有的记录匹配，确保软件包来自真实源。下面开始相关设置。

为了给其他机器提供软件包，我们安装 nginx 作为 web 服务器。其他 web 服务器只要能向客户端共享 URL 作为文档根目录，也可以使用。`poudriere bulk` 运行期间，我们还可以共享编译日志，查看当前编译的进度并调试失败的 ports。

```sh
poudriere# pkg install nginx
```

当然，如果我们想修改 nginx 默认的软件包配置，这个 nginx 也可以来自我们自己的本地存储库。注意：我们没有用 SSL 加密流量本身，只是加密软件包存储库本身。为传输加密添加 SSL 留给读者练习，并不复杂。

编辑 **/usr/local/etc/nginx.conf** 后，它应该包含以下部分：

```sh
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

保存并退出 `nginx.conf` 后，我们还要改一处，让编译日志也能在浏览器中正确显示。也就是说，打开日志文件（.log 扩展名）时，它不会作为下载提供，而是直接在浏览器中显示。为此，我们打开 **/usr/local/etc/nginx/mime.types**。找到以 `text/plain` 开头的行，改为也包含日志文件：

```sh
text/plain             txt log;
```

在 **/etc/rc.conf** 中添加条目，让 nginx 开机自启：

```sh
poudriere# service nginx enable
```

现在启动服务：

```sh
poudriere# service nginx start
```

你可以把浏览器指向 `http://server_domain_or_IP` 来查看编译状态和日志。

为存储库签名，我们创建几个目录存放密钥和证书。

```sh
poudriere# cd /usr/local/etc/ssl

poudriere# mkdir keys certs
```

这些密钥不应落入他人之手，所以我们只允许 root 用户访问该目录：

```sh
poudriere# chmod 0600 keys
```

接下来，生成用于 poudriere 存储库的 RSA 私钥。

```sh
poudriere# openssl genrsa -out keys/poudriere.key 4096
```

对待该密钥就像对待你家前门的物理钥匙：绝对不要交给别人，甚至不要让任何人看到。如果密钥泄露，攻击者可以把任何类型的软件包注入你的机器，这会让你非常难受。

接下来，我们根据刚刚生成的私钥创建一个证书（公钥）：

```sh
poudriere# openssl rsa -in keys/poudriere.key -pubout -out
certs/poudriere.cert
```

所有必要的密钥现在都备齐了。接下来要让 poudriere 知道它们，以便编译软件包时用它们签名。这要在 **/usr/local/etc/poudriere.conf** 中设置，就是我们之前做首次基本 poudriere 配置时改过的文件。找到以 `PKG_REPO_SIGNING_KEY` 开头的行，把它指向我们生成的私钥位置。最终结果如下：

```sh
PKG_REPO_SIGNING_KEY=/usr/local/etc/ssl/keys/poudriere.key
```

为了额外的可点击链接，我们还要改一行，如下所示：

```sh
URL_BASE=http://my.domain.or.IP
```

`URL_BASE` 定义的网站显示过去和当前软件包编译的许多统计信息。编译过程中它会自动刷新，显示编译状态。你还可以深入了解每次编译，查看哪些 ports 已成功编译、被跳过或被忽略。非常不错！

添加这些行后，poudriere 就能签名新编译的软件包。在下一次

```sh
poudriere# poudriere bulk \
-j 132x64 \
-p default \
-f /usr/local/etc/poudriere.d/pkglist.txt
```

运行之后，我在输出中看到了这些新行：

```sh
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

是时候重新访问 **/usr/local/etc/pkg/repos/local.conf**，在签名类型旁边添加证书。编辑后，文件应该如下所示：

```sh
Poudriere: {
        url: "file:///usr/local/poudriere/data/packages/132x64-default",
        priority: 23,
        mirror_type: "srv",
        signature_type: "pubkey",
        pubkey: "/usr/local/etc/ssl/certs/poudriere.cert",
        enabled: yes
}
```

## 客户端配置

现在该把另一台机器加入我们的签名软件包存储库了。登录到你希望使用自定义、新编译软件包的 FreeBSD，为 poudriere 证书和存储库配置创建目录。

```sh
clienthost# cd /usr/local/etc/
clienthost# mkdir ssl/certs ssl/keys
clienthost# mkdir pkg/repos
```

然后，安全地把编译机 **/usr/local/etc/ssl/certs/** 下的 `poudriere.cert` 复制到我们刚创建的目录。你可以用 `scp(1)` 从主机传到客户端：

```sh
poudriere# scp /usr/local/etc/ssl/certs/poudriere.cert
clienthost:/usr/local/etc/ssl/certs/
```

接下来定义新的存储库位置，与上面类似。唯一区别是 URL。编译机可以用 `file:///` 引用本地文件系统，远程机器则要通过 http 连接我们的 nginx。`mirror_type` 也不同，除此之外配置完全相同，如下所示：

```sh
Poudriere: {
        url: "http://my.domain.or.ip/packages/132x64-default",
        mirror_type: "http",
        signature_type: "pubkey",
        pubkey: "/usr/local/etc/ssl/certs/poudriere.cert",
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

显示处理了 247 个软件包的那一行，肯定意味着它能找到我的另一个存储库和其中的软件包。运行

```sh
clienthost# pkg upgrade
```

输出引用了我的自定义存储库名称，证实它可以工作：

```sh
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

成功了！该软件包已从编译机下载，并使用了我为它制作的自定义配置。知道自己不必在每台性能较弱的 FreeBSD VM 上重新编译 llvm，感觉非常爽快和满意。这正是性能强劲的编译服务器的用途。

这是混合使用 FreeBSD 和自定义存储库时 `pkg upgrade` 的输出：

```sh
New packages to be INSTALLED:
        cyrus-sasl: 2.1.28 [Poudriere]
        gdbm: 1.23 [FreeBSD]
        icu: 73.2,1 [FreeBSD]
        openldap26-client: 2.6.5 [Poudriere]
```

请注意，混合这两类软件包可能因彼此交互方式带来一些麻烦。某个软件包期望另一个软件包具有特定选项，但该选项不在另一个存储库的特定软件包中，可能引发编译或安装错误，导致应用程序无法按预期工作。理想情况下，应使用单一存储库源，要么是上游存储库，要么是自己内部的存储库。

另外，如果你为 `latest` 分支编译软件包并分发到客户端，要确保这些客户端在 **/etc/pkg/freebsd.conf** 中也使用 `latest`。否则，软件包版本会比季度版更新。我遇到过这种情况：上面的软件包顺利更新和安装，但运行 `pkg autoremove` 时它们又被列为需要删除的软件包。然后我再升级它们，恶性循环就此完成。

如果只是简单地把存储库加到某台机器而没有证书，`pkg update` 会抱怨缺少证书：

```sh
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

编译服务器上还可以做的最后一步是创建 cron 任务来更新 poudriere 的 ports 树：

```sh
poudriere# poudriere ports -u -p default
```

这次我们用 `-u` 而不是 `-c`，表示要更新现有 ports 树。我们可以每天执行一次，甚至更早，取决于你希望软件包多新。然后让 poudriere 运行批量编译：

```sh
poudriere# poudriere bulk \
-j 132x64 \
-p default \
-f /usr/local/etc/poudriere.d/pkglist.txt
```

我每天晚上运行此任务，第二天早上就有新编译的软件包等着我。我可以用 nginx 提供的 poudriere 网站查看编译状态。

Poudriere 为不同架构编译许多不同软件包时，才真正发挥出威力。它用 qemu 为非 amd64 架构（如我的树莓派）编译软件包。一切并行进行，所以我可以更快地编译自己的软件包，其余的可以从默认 FreeBSD 存储库获取——因为我不需要在这些软件包上设置特殊选项。时间一长，你自己的自定义软件包清单肯定会增长。使用你自己的证书存储库的机器数量也会如此。你完全可以控制何时更新哪些软件包，甚至可以用 Poudriere 帮助编译新的 ports。

---

**BENEDICT REUSCHLING** 是 FreeBSD 项目的文档提交者，文档工程团队成员。他曾在 FreeBSD 核心团队任职两届。他在德国达姆斯塔特应用科学大学管理大数据集群，还为本科生开设“Unix for Developers”课程。Benedict 是每周 bsdnow.tv 播客的主持人。
