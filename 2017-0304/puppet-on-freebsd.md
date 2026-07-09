# FreeBSD 上的 Puppet

作者：Brad Davis、Andrew Fengler

在 Puppet 之前，主要的配置管理工具是 1993 年问世的 CFEngine。Puppet 于 2005 年首次发布，助推了配置管理领域的一次复兴。如今它被广泛认为是最流行的配置管理工具之一。

## 可编程的配置

Puppet 以及任何配置管理工具的主要优势，在于以可编程的方式控制配置元素。Puppet 通过一个面向对象的系统实现这一点。最基本的元素是资源定义，一个资源可以是一个文件、一个软件包、一条要执行的命令，或许多类似的东西。如果内置的资源类型不够用，还可以定义自定义资源。资源定义与条件结构组合构成类。这些类是按主机进行控制的主要元素。每个主机——Puppet 称之为 node——都有一个节点定义，在其中控制该特定主机的配置。虽然可以在此处配置所有资源，但更合理的做法是创建可复用的类：例如，只需引入一个定义 Web 服务器应如何配置的类，就能轻松让任意数量的 Web 服务器达到相同状态。

## 应用配置

运行 Puppet 有两种方式。第一种是使用 `puppet apply` 从本地一组配置安装。这是最简单的方式，但需要某种方法把配置复制到每个节点。它也无法使用 PuppetDB。另一种方式是运行专用的 Puppetmaster。在此配置下，所有节点都连接到中央 Puppetmaster 并从那里获取配置。这种方式的另一个优势是节点会把它们所有的事实（facts）发送给 Puppetmaster。事实是关于节点的信息片段，如操作系统、网络掩码、CPU 核心数等。这些事实可在条件判断中作为变量使用，从而控制资源：例如，可以根据操作系统版本选择软件包仓库。有了 Puppetmaster 上的所有事实，Puppetboard 这类工具就能提供所有服务器上正在发生什么的视图。它会展示事实的分类、Puppet 执行的结果，以及哪些内容正在变化或已失败的详情。事实由 Puppet 依赖的一个名为 'Facter' 的库提供。这个工具可以从命令行运行，查看给定机器上的结果。查看 Puppet 特有的事实，可运行：

```sh
# puppet facts
```

## 在 FreeBSD 上运行

Puppet、Puppetserver 和 PuppetDB 都可以从 ports 树安装，在 FreeBSD 上起步非常简单。例如，要安装截至本文撰写时的当前版本 Puppet，使用：

```sh
# pkg install puppet4
```

其他版本也提供向后兼容，只要它们仍受 Puppet 项目支持。安装最新版 PuppetDB 使用：

```sh
# pkg install puppetdb4
```

PuppetDB 并非必需，但确实为 Puppet 安装提供了一些高级功能。例如，能在不同节点上使用其他节点的事实。本文不会涵盖 PuppetDB，但想向高级用户提及其可用性。接下来，只需在一个类中写几个资源定义，在节点定义中引入它，然后把 Puppet agent 指向服务器或配置——取决于你选择的方式——Puppet 就跑起来了！然后就是管理一切的乐趣部分。FreeBSD 上的坑出奇地少，而且从 Puppet 4 起，pkgng 是开箱即用的支持软件包提供者。配置文件可通过模板轻松管理。Puppet 使用 Ruby 的 ERB 模板系统，允许把 Ruby 代码本身嵌入文件中，并可访问 Puppet 所知道的关于环境的所有变量。

## 示例：使用无 agent 模式

所谓的无 agent（agentless）模式是测试 recipe（配方）在应用到大量机器之前的好方法。快速启动一个 jail 或 VM 并运行 apply 来测试配置非常方便。例如，如果我们创建一个名为 `test.pp` 的文件，其中包含如下安装 nginx 的 recipe：

```sh
package { 'nginx':
    ensure => installed,
}
```

可轻松通过运行以下命令执行：

```sh
# puppet apply test.pp
```

要获取更多调试信息以查看什么可能不工作，启用完整调试模式和详细模式会很有帮助：

```sh
# puppet apply -d -v test.pp
```

## 示例：使用 master/agent

安装 Puppet 后，在 master 上启用它：

```sh
# sysrc puppet_master_enable=YES
```

然后启动 master：

```sh
# service puppet_master start
```

在客户端（可以是同一台机器）上：

```sh
# sysrc puppet_enable=YES
```

修改 puppet.conf 指向 master 的 IP 或主机名。这里我们指向 localhost：

```ini
master: 127.0.0.1
```

启动客户端：

```sh
# service puppet start
```

master 和 agent 启动后，agent 会尝试连接 master。它们随后会创建一个内部证书颁发机构以相互认证。agent 会生成客户端证书并上传给 master。你可以在日志中看到 agent 正等待 master 接受其证书。在 master 上运行以下命令查看该证书：

```sh
# puppet ca list
```

然后你应该能看到来自客户端的证书，看起来像：“hostname” [证书请求的 sha256 校验和]

现在签名该证书：

```sh
# puppet ca sign hostname
```

agent 应该收到证书并继续运行。不过它不会做太多事，并会吐出一条大大的黄色警告：`ERROR: Could not fetch my node definition, but the Puppet agent run will continue`。看来我们需要创建一个节点定义。

## 创建简单配置

先进入 Puppet 配置目录，默认是 **/usr/local/etc/puppet/**。配置环境放在 `environments` 目录中。你可以有多个环境，Puppet agent 可配置选择使用哪个环境。这意味着你可以为开发等设立独立分支，但现在默认的 `production` 环境用于测试就很合适。每个环境都有 `manifests` 和 `modules` 目录。站点范围的清单（如通用配置和节点定义）放在 manifests 下，而 modules 存放可复用的配置块。Puppet 文件以 .pp 结尾，让我们创建节点定义。节点定义按主机名匹配，可以精确匹配，也可以正则匹配。创建 localhost.pp，内容如下：

```sh
node 'localhost.example.com' {
    $host_desc = "My computer"
    $colorscheme = "desert"

    file { "/etc/test":
        ensure  => present,
        user    => "root",
        group   => "wheel",
        mode    => "0644",
        content => "Hello World! This is ${host_desc}\n",
    }
}
```

这就是一个简单的节点定义。我们设置了两个变量供后续使用，并创建了你的第一个文件。可以看到文件设置了许多选项，即其属主、属组、权限（mode）以及文件内容。保存后再运行一次 Puppet。如果你启动了 Puppet 服务，它会定期运行；如果不想等，可以用 `--test` 或 `-t` 标志触发一次运行：

```sh
# puppet agent -t
```

这会发起一次运行。完成后你就得到一个崭新的文件。现在创建第一个模块。假设我们想把最喜欢的编辑器及其所有设置放到我们的机器上。在 modules 目录下创建一个名为 `vim` 的目录。其中若干子目录用于存放模块的不同部分。创建一个 `manifests` 目录，这里存放该类的所有 Puppet 配置。对于模块，必须始终有一个与模块同名的类，放在名为 `init.pp` 的文件中。这是模块的起点，我们可以按需添加更多包含额外类的文件。创建 init.pp：

```sh
class vim {
    package { "vim-lite":
        ensure   => latest,
        name     => "editors/vim-lite",
        provider => "pkgng",
    }
    file { "/home/beastie/.vimrc":
        ensure => present,
        user   => "beastie",
        group  => "beastie",
        mode   => "0644",
        source => "puppet:///modules/vim/vimrc",
    }
}
```

我们又管理了两个资源：一个用 pkgng 安装 vim-lite 的软件包，以及一个安装我们 vimrc 的文件。注意我们用了 `source` 而非 `content`。这不是把一些文本放进文件，而是从 Puppetmaster 获取一个文件。这些文件也放在 `vim` 模块中，但在 `files` 子目录而非 manifests 目录下。文件 URL 的格式是 `puppet:///modules/<模块名>/<文件名>`。所以我们的 `puppet:///modules/vim/vimrc` URL 会在当前环境下查找 `modules/vim/files/vimrc`。每种资源类型提供不同的选项集。对于文件，有各种各样的选项，其中之一是 `ensure`。它决定我们希望文件处于什么状态。当我们不再需要某文件时，可以把 ensure 设为 `absent` 来清理。现在我们有了一个类，只需在节点定义中加一行 `include vim`，它就会引入整个 vim 类。我们可以用一行把它加到任意多的节点上。但假设我们需要在某些服务器上改一个设置。用模板很容易做到。

## 模板

模板是任何自动化系统中非常强大的功能。模板系统可以访问 Facter 的所有变量，因此它了解机器的一切。它允许用简单的方式表达跨许多机器的复杂配置。例如，要模板化一个配置文件，其中需要做一些诸如设置要绑定的 IP 地址之类的事，首先为文件创建一个资源：

```sh
file { '/usr/local/etc/nginx/nginx.conf':
    ensure  => 'file',
    owner   => 'root',
    group   => 'wheel',
    mode    => '644',
    content => template('/usr/local/etc/puppet/templates/nginx.conf'),
    require => Package['nginx'],
}
```

模板引擎处理 `<%=` 和 `%>` 之间的模板代码。以下是模板可能包含的一个片段：

```sh
http {
    server {
        listen <%= @ipaddress %>:80;
        server_name <%= @fqdn %>;
        location / {
            root /usr/local/www/nginx;
            index index.html;
        }
    }
}
```

注意变量以 `@` 符号开头。这会给出如下结果：

```sh
http {
    server {
        listen 192.168.11.10:80;
        server_name test.example.com;
        location / {
            root /usr/local/www/nginx;
            index index.html;
        }
    }
}
```

模板还有许多更强大的用法，包括使用 Ruby 函数创建更高级的功能。作为一个稍微复杂一点的模板示例，曾经用 Ruby 的 gsub 函数修改过某个变量，然后把结果嵌入文件中。

## 依赖

你可能遇到的一个问题是，某些资源需要其他资源才能运作。在配置文件就位之前启动服务，很可能达不到你想要的效果。Puppet 4 中的依赖处理改善了很多，因为资源现在按你在文件中放置的顺序应用。以前并非如此，它们在执行前会被排序。但你可能仍需要指定依赖顺序。主要方式是使用 `require` 关键字。有两种用法。你可以 require 一个完整的类：

```sh
require nginx
```

这与 `include nginx` 完全一样，但会确保整个类在继续之前应用完毕。另一种方式是 require 另一个资源：

```sh
package { 'zsh':
    ensure => 'installed',
}
user { "beastie":
    ensure  => "present",
    comment => "Beastie",
    groups => ["beastie", "wheel"],
    shell   => "/usr/local/bin/zsh",
    require => Package['zsh'],
}
file { "/home/beastie":
    ensure => "directory",
    mode   => "0644",
    owner  => "beastie",
    group  => "beastie",
}
```

这个用户将依赖于 Z Shell 软件包的安装，又因为文件依赖于属主和属组，那些将自动依赖于该用户。

## 节点间交互

你可能需要服务器彼此感知，无论是为了将 IP 加入白名单，还是为了知道去哪里监听心跳。困难在于，通常这需要硬编码长长的地址清单。Puppet 有一个叫”导出资源”（exported resources）的功能。这些资源与普通资源很像，但它们不是安装在主机上，而是存储在 Puppetmaster 上供任意节点收集。资源也可以打标签，让一台服务器能用一行就捡起一整个目录的配置。我们可以这样导出一个文件：

```sh
@@file { "/usr/local/etc/nagios/puppet.d/jail_${hostname}.dns.cfg":
    ensure  => present,
    content => template('nagios/server.cfg.erb'),
    tag     => "nagios_config",
}
```

然后在另一台主机上收集所有标记为 `nagios_config` 的文件：

```sh
File <<| tag == "nagios_config" |>> { }
```

这会在收集它的服务器上创建该文件。注意我们在文件名中使用了主机名。这样，如果我们从多台主机导出配置，它们不会冲突。

## 结语

查看文档有几种不同方式，其中最好的来源之一是 Puppet 文档网站：<https://docs.puppet.com/puppet/latest/>。此外，从命令行获取特定 Puppet 命令的帮助：

```sh
puppet help <cmd>
```

这为开始使用 Puppet、让任何系统管理员或其他对自动化构建环境感兴趣的人的工作变得更轻松，提供了良好的基础。

---

**ANDREW FENGLER** 是加拿大汉密尔顿 ScaleEngine Inc. 的 Unix 管理员，用 Puppet 管理大量 Unix 系统。他有两年大规模管理 FreeBSD 系统的经验。

**BRAD DAVIS** 职业生涯中曾担任基础设施系统架构师、开发者，现职为顾问。他是超过 10 年资历的 FreeBSD 提交者，参与了项目的多个不同领域。最初从文档提交者起步，并以此切入，帮助集群管理和 Postmaster 团队。从这些团队退役后，他涉足 pkg 和 poudriere。如今他致力于各种 ports、打包 FreeBSD 基本系统，以及一个名为 RaspBSD 的项目，把 FreeBSD 和一些额外工具优雅地打包给 BeagleBone Black 和 树莓派 等 ARM 开发板。
