# 基于证书的 Icinga 监控

- 原文链接：[Certificate-based Monitoring with Icinga](https://freebsdfoundation.org/wp-content/uploads/2023/01/Reuschling_Icinga.pdf)
- 作者：**BENEDICT REUSCHLING**

Icinga 是流行的 Nagios 主机和服务监控软件的继任者。其目标是作为 Nagios 的兼容替代品，同时提供一些新功能。本文介绍了如何设置一台中央 Icinga 主机，通过证书以自上而下的配置同步方式监控终端系统。

自上而下概述了检查（如磁盘满、负载过高等）在远程机器上的执行方式。在自上而下的监控中，中央 Icinga 主机（称为父节点）负责将配置文件同步到被监控的节点（称为子节点）。在这些子节点上，配置更改后无需手动重启，因为同步、验证和重启会自动进行。检查会在子节点的调度程序上直接执行，并定期进行。主机被组织在一个全局区域中，每个主机（父节点和子节点）都被定义为该区域中的一个终端。在这种设置中，每个子节点必须通过在票据中创建证书签名请求来验证自己是监控区域的一部分，父节点将验证并签署该请求。这建立了信任，确保机器以加密方式进行通信，并且父节点接收到的监控数据在传输过程中没有被篡改。

本文介绍的设置使用 FreeBSD 作为父节点来监控另一个 FreeBSD 主机作为子节点。当然，其他操作系统如 Linux/Windows 也可以以相同的方式使用，但在本文中未涉及，原因是简介已经很长。虽然设置概述是直接在主机上运行以减少复杂性，但作者实际上将其成功地运行在 jail 中——到目前为止没有问题。

## 准备工作

我们假设所有主机已经安装完毕，能够在同一网络上找到彼此，并且它们之间已经建立了基本的 SSH 连接。我们的中央监控服务器将被命名为 monitor.example.org，稍后我们将设置的客户端主机命名为 client.example.org（你可以看到我非常有创意的命名方式）。命令前的提示符将指示命令应该在何种主机上执行。

让我们从准备中央监控主机（父节点）开始：

- 确保主机上的时钟已同步。这对于后续证书的正确生成非常重要。通常可以通过运行 `monitor# ntpdate -b pool.ntp.org` 来实现。
- 在这台中央 FreeBSD 主机上，我们希望使用 Icinga 和其他软件包的最新版本，而不是季度版本。编辑 `/etc/pkg/FreeBSD.conf` 文件，将以 `url:` 开头的行中的 `quarterly` 改为 `latest`。然后保存并退出。
- 要更新软件包仓库并获取较新的软件包，请运行

```sh
monitor# pkg update
```

## 设置 PostgreSQL 

在安装所需的软件包（包括 PostgreSQL 作为后台数据库和 nginx 作为 web 服务器来托管 Icingaweb2 监控界面）之前，我们首先创建一个 ZFS 数据集用于存储 postgres 数据库，这样当软件包解压时，它会被填充。如果你不使用 ZFS，使用常规目录也完全可以。

以下命令将在我们的示例池 mypool 上创建一个新的数据集，路径为 `/var/db/postgres/data`。如果该路径下的数据集不存在，`-p` 参数也会创建它们。接下来，我们禁用访问时间（atime），因为这里不需要它，并且通过不在每次写入时更新文件的时间戳来节省一些 I/O。使用更新的 ZFS 2.0，我们还在数据集上使用 zstd 压缩。由于 Postgres 以 8k 为单位写入数据，我们将 ZFS 的 `recordsize` 设置为与其匹配以获得最佳性能。通过将 `logbias` 设置为 `throughput`，我们指示 ZFS 优化数据库的同步写入，以高效利用资源。挂载点设置为重叠现有的 `/var/db/postgres` 路径。当软件包安装时，它会放在该数据集上，而不是常规的 `/var/db` 目录中。在这里我们不关注 PostgreSQL 数据库的进一步调整。你可以访问 https://pgtune.leopard.in.ua/，输入 PostgreSQL 主机的参数，获取配置建议，然后将其添加到 `postgresql.conf` 文件中。

```sh
monitor# zfs create -p mypool/var/db/postgres/data
monitor# zfs set atime=off mypool/var/db
monitor# zfs set compression=zstd mypool/var/db
monitor# zfs set recordsize=8k mypool/var/db/postgres
monitor# zfs set logbias=throughput mypool/var/db/postgres
monitor# zfs set mountpoint=/var/db/postgres mypool/var/db/postgres
```

现在是安装所需软件包的时候了。使用 `pkg install` 很容易完成，包括自动解决依赖问题：

```sh
monitor# pkg install icinga2 icingaweb2-php74 postgresql13-server nginx \
ImageMagick7-nox11 php74-pecl-imagick-im7
```

在撰写本文时，这些软件包版本是最新可用的。请访问 [www.freshports.org](https://www.freshports.org) 检查是否有更新的软件包版本。特别是 PHP 可能已经更新了版本。  

其中一些服务需要在 `/etc/rc.conf` 中添加相应的条目，以确保系统启动时自动运行。包括以下服务：

```sh
monitor# sysrc sshd_enable=yes
monitor# sysrc icinga2_enable=yes
monitor# sysrc postgresql_enable=yes
```

请注意，这些服务尚未启动，因为在此之前还需要进行一些配置。  

与 `postgres` 软件包一起安装的还有同名的系统用户和用户组，因此现在可以对 `/var/db/postgres` 设置权限。

```sh
monitor# chown -R postgres:postgres /var/db/postgres
```

接下来，通过 `postgres` 用户运行 `initdb` 来初始化 PostgreSQL 数据库集群，并使用 UTF-8 作为编码。请注意，这些命令需要由 `postgres` 用户执行，尽管现在也可以通过 `service` 命令来完成（有时候我还是习惯老方法）。

```sh
monitor# su postgres
postgres@monitor$ initdb -D /var/db/postgres/data -E UTF8"
```
成功初始化数据库集群后，使用 `pg_ctl` 启动服务器：

```sh
postgres@monitor$ pg_ctl start -D /var/db/postgres/data
```

接下来需要创建 Icinga 角色和数据库，稍后将加载一些初始表和序列，以构建监控后端。

```sh
postgres@monitor$ createuser -drs icinga
postgres@monitor$ createdb -O icinga -E UTF8 icinga
```

在 `pg_hba.conf`（位于数据目录）中添加如下条目，可允许刚创建的 Icinga 用户通过本地主机访问数据库（无需将其暴露到网络，Icinga 依然可以正常工作）：

```c
local icinga icinga md5
host icinga icinga 127.0.0.1/32 md5
```

现在将 Icingaweb2 的数据库模式定义以及 IDO（Icinga Data Objects）的模式加载到数据库中：

```sql
postgres@monitor$ psql -U icinga \
 -d icinga < /usr/local/share/icinga2-ido-pgsql/schema/pgsql.sql
```

退出 postgres 用户并继续剩下的设置。在 Icinga 端，功能（features）控制监控系统的各项功能，其中包括使用哪个数据库后端。要启用 PostgreSQL 作为 IDO 的后端，请运行以下命令：

```sh
monitor# icinga2 feature enable ido-pgsql
```

在某些情况下，并非所有文件和目录都归 Icinga 系统用户所有。通过对主 icinga2 目录运行 `chown`，可以确保权限被正确设置。

```sh
monitor# chown -R icinga:icinga /usr/local/etc/icinga2
```

这部分设置完成后，我们将继续进行 Web 服务器的配置。尽管这里使用的是 nginx，但也可以使用其他 Web 服务器，如 Apache2。Icinga 文档中也有相关的配置步骤。  

## 设置 Nginx   

Icinga 的 Web 界面（被恰当地命名为 Icingaweb2，因为它是版本 2）是一个 PHP 应用程序，用于通过浏览器管理主机和服务。任何失败的检查事件也会在其中显示，你可以在一个中心位置确认问题或定义停机时间。要配置 PHP fastCGI 进程管理器（php-fpm）来处理来自 Web 服务器的请求，请启用位于 `/usr/local/etc/php-fpm.d/www.conf` 文件中的以下选项：

```sh
monitor# cd /usr/local/etc/php-fpm.d
monitor# sed -i "" 's/^;listen = 127.0.0.1:9000/listen = /var/run/php5-fpm.sock/’
www.conf
monitor# sed -i "" 's/^;listen.owner/listen.owner/' www.conf
monitor# sed -i "" 's/^;listen.group/listen.group/' www.conf
monitor# sed -i "" 's/^;listen.mode/listen.mode/' www.conf
```

这基本上是取消注释文件中已经存在的行，以激活它们，并将 `listen` 指令替换为使用本地 php5 套接字，而不是为其在主机上打开端口。  

主要的 Web 服务器配置是在位于 `/usr/local/etc/nginx` 中的 `nginx.conf` 文件中完成的，我们在其中引用了 fastCGI 套接字等内容。以下配置块会插入在被注释掉的 `#access_log` 行之后：

```ini
location ~ ^/icingaweb2/index\.php(.*)$ {
fastcgi_pass unix:/var/run/php5-fpm.sock;
fastcgi_index index.php;
include fastcgi_params;
fastcgi_param SCRIPT_FILENAME /usr/local/www/icingaweb2/public/index.php;
fastcgi_param ICINGAWEB_CONFIGDIR /usr/local/etc/icingaweb2;
fastcgi_param REMOTE_USER $remote_user;
}
location ~ ^/icingaweb2(.+)? {
alias /usr/local/www/icingaweb2/public;
index index.php;
try_files $1 $uri $uri/ /icingaweb2/index.php$is_args$args;
}
```

请注意，为了保持本教程的简洁，这里没有包含 SSL 部分。强烈建议（而且如今几乎是标准做法）为 Web 服务器生成证书，并为安全通道配置端口 443。网络上有很多关于此的教程，像 Let’s Encrypt 这样的服务使得这个过程变得简单和方便。

当安装软件包时，`/usr/local/etc/php.ini-production` 是用于生产环境的 PHP 设置的文件。这些设置对我们的目的来说是合适的，我们通过将其复制过来来激活它们，成为新的 `php.ini` 文件。

```sh
monitor# cp /usr/local/etc/php.ini-production /usr/local/etc/php.ini
```

以这个文件作为基础 PHP 配置，我们只需要替换其中的时区信息。我使用这个 `sed` 单行命令将我的安装设置为中欧时区。根据你的地理位置，使用适合的时区设置。

```sh
monitor# cd /usr/local/etc
monitor# sed -i "" s,;date.timezone =,date.timezone = Europe/Berlin, php.ini
```

我们在这里将常规的 `sed` 分隔符 `/` 替换为逗号，以避免与地区和城市之间的分隔符混淆。看，我在我的教程中也偷偷加入了一些 `sed` 技巧，稍后感谢我吧...哦，顺便说一下，基本上，这就是提供 icingaweb2 给终端用户所需的一切。接下来，我们将把注意力集中在 Icinga 配置上。

## Icinga 监控设置 


Icinga 使用多种方式来监控系统，并且对不同的监控环境和需求非常灵活。例如，一个主机可能不是一直可用（例如流动用户）或没有与中央监控主机的直接连接。在后者的情况下，卫星系统可以通过不同的网络或子网将监控数据和检查结果转发到中央实例。在我们这里使用的设置中，中央实例（称为主节点）控制对受监控系统的检查执行。通过主节点和客户端之间交换证书来建立信任。一个区域（zone）定义了监控发生的地理位置（如欧洲、非洲等），或者在我们的监控上下文中有意义的某种逻辑分组。例如，一个整个工厂、办公室、服务器机房、机架等可以各自形成自己的区域。当然，DNS 区域也是可以的，任何对整体监控或基于某些共同标准进行监控的内容都是可行的。

首先，我们生成主节点证书，后者将用于监控中央实例本身，并作为信任的基础，用于在本教程后面添加其他受监控客户端。该设置通常是交互式的，但在这里我们将所有必要的参数通过命令行传递：

```sh
monitor# icinga2 node setup --master --zone "my-zone" \
 --cn monitor.example.com \
 --listen monitor.example.com,5665 --disable-confd"
```

在这里，我们为我们的中央主机 `monitor.example.com` 生成证书，并指示 Icinga 不在 `/usr/local/etc/icinga2` 中填充 `conf.d` 子目录。我们应自己创建这些文件。

区域（zone）和 `cn` 的命名可以根据你的本地需求来选择。防火墙的 5665 端口应该是开放的，以便可以联系客户端并发送检查结果。接下来，我们切换到目录 `/usr/local/etc/icinga2` 并创建一些文件和目录：

```sh
monitor# cd /usr/local/etc/icinga2
monitor# mkdir conf.d
```

我们定义了一个 API 用户，该用户具有生成票证的权限，通过该票证，受监控的客户端可以请求成为监控区域的一部分。主服务器随后将允许或拒绝票证请求，并使用自己的证书签署客户端证书，从而在两者之间建立安全、可信的连接。

`api-users.file` 文件包含以下内容：

```ini
object ApiUser "client-pki-ticket" {
password = "randomstringthatmustbechanged"
permissions = [ "actions/generate-ticket" ]
}
```

务必将密码行更改为由随机数字和字符组成的随机字符串，长度越长越好。接下来，在 `/usr/local/etc/icinga2` 目录中创建一个 `zones` 子目录，用于存储分发到该区域所有成员的信息。典型的例子包括客户端将从中央监控实例接收到的检查信息和监控间隔。这样，客户端无需额外的本地配置，系统管理员只需更改中央区域配置，这些更改将安全地传播到所有主机。由于我们的区域名为 `my-zone`（显然，创意是我的强项），我们创建一个子目录，只包含该区域客户端相关的信息。其他区域可以完全不同，但监控配置位于一个中心位置，而不是每个组成该区域的主机上。

```sh
monitor# mkdir -p /usr/local/etc/icinga2/zones.d/my-zone
```

我们的区域包含各种信息：要监控的主机、要监控的内容（即每个主机上执行哪些检查）、监控间隔（频率）等。我们首先在一个名为 `/usr/local/etc/icinga2/zones.d/my-zone/hosts.conf` 的文件中定义中央监控主机。对于我们的主机，它的内容如下所示：

```ini
object Host "monitor.example.com" {
import "generic-host" //为所有主机输入通用设置
address = "monitor.example.org"
vars.os = "FreeBSD"
//遵循主机名 == 端点名的惯例
vars.agent_endpoint = name
}
```

每个主机都被定义为一个主机对象，并指定它在网络上可达的地址。我们还定义了一个变量，通过这个变量，我们可以在 Icingaweb2 中筛选特定的主机，以便进行分组，或者仅为匹配这些条件的特定主机定义检查。这将在后面展示。

"import 'generic-host'" 这一行引用了一个模板。模板帮助我们将通用设置应用到所有主机，而无需为每个主机重新定义这些设置。例如，每个主机应该有相同的检查间隔（即检查的频率）和其他类似设置。通过避免重复定义，这使得文件更简洁，不同的主机可以使用不同的模板设置，或者用仅对该特定系统有效的自定义设置覆盖模板设置。

`templates.conf` 文件位于 `zones.d/my-zone` 目录下，其内容如下所示：

```ini
template Host "generic-host" {
max_check_attempts = 5
check_interval = 2m
retry_interval = 30s
enable_flapping = true
check_command = "hostalive" //在主系统上执行检查
}
template Service "generic-service" {
max_check_attempts = 5 //在 "HARD"状态前重新检查 5 次
check_interval = 2m
retry_interval = 1m
enable_flapping = true
}
```


这里定义了两个模板，一个用于通用主机，另一个用于服务。总体来说，主机和服务在生成警报之前会被检查五次。这是为了避免偶尔的丢包或响应较慢的设备或进程，但这些设备或进程通常是正常工作的。`check_interval` 定义了检查的执行次数，而 `retry_interval` 定义了在一次检查未返回 OK 状态时，下一次检查的时间间隔。一定要根据你的监控需求调整这些间隔。请记住，监控频率越高，产生的流量就越大，而且检查返回的数据需要存储在数据库中，随着监控时间的延长，数据库会逐渐变大。

"Flapping" 状态指的是主机或服务似乎可用，但下一次又不可用，然后又恢复可用，反复出现这种状态（在状态之间迅速切换，且没有稳定的迹象）。Icinga 能通过在一段时间内比较最后已知状态和当前状态来检测这些 flapping 状态。这些状态默认是未启用的，但对于调试只在特定负载时间或活动期间发生的问题，flapping 状态是非常有价值的信息。一个表现异常的主机会在 Icinga 中显示出来，应该进一步调查其根本原因。问题也可能源于网络本身，因此需要排除任何可能导致问题的其他因素。`check_command` 定义了哪个 Icinga 提供的检查需要作为默认检查运行。`hostalive` 命令基本上是一个伪装的 ping，用于检查主机是否可达。之所以只在 `generic-host` 模板中定义，是因为服务通常会定义一个适合该服务的不同 `check_command`，而这个命令无法轻易通过模板进行通用化。

现在我们已经为常见功能定义了模板，接下来是定义我们要运行哪些检查以及在哪些主机上运行。这些定义存储在 `zones.d/my-zone/services.conf` 文件中。以下是我定义的检查磁盘空间的配置：


```ini
apply Service "disk" {
import "generic-service"
check_command = "disk"
// 指定远程代理作为命令执行端点，获取主机自定义变量
command_endpoint = host.vars.agent_endpoint
// 仅在主机被标记为代理端点的情况下分配
assign where host.vars.agent_endpoint
}
```

应用服务意味着将其分配给特定目标，可以是主机或表达式的结果。在这个例子中，我们定义了所有定义为端点的主机都应该运行磁盘检查。检查的执行发生在主机本身上，称为主动检查。被动检查则是在中央监控实例上运行，尝试访问远程系统、运行检查并获取结果。主动检查和被动检查都可以为主机或目标定义。它们各有优缺点，但在像这样的简单监控设置中，使用 Icinga 提供的服务是一个很好的起点。

Icinga 提供了以下常见服务作为检查项：磁盘、负载、用户、交换空间、进程、ping 和 ssh。这些为监控提供了良好的初始基础，以便检查交换空间是否不足、磁盘是否满、负载是否过高，或者突然有 200 个用户登录（这可能是正常的，也可能不是）。一个特殊的检查是测试我们的远程端点系统是否仍然可用，并且在定义的区域内。为此，我们可以扩展刚才创建的 `services.conf` 文件，加入以下代理健康检查：

```ini
apply Service "agent-health" {
check_command = "cluster-zone"
display_name = "cluster-health-" + host.name
//遵循惯例：代理区域名称的 FQDN 与主机对象名称相同。
vars.cluster_zone = host.name
assign where host.vars.agent_endpoint
}
```
除了运行集群区域检查命令（我们不需要了解太多关于它的细节就可以使用它）之外，我们还可以通过定义 `display_name` 来查看 Icingaweb2 界面中如何显示不同的检查描述。通过这种方式，我们可以一目了然地看到被监控系统的名称，并且前缀是字符串 "cluster-health"。

内部的 Icinga 数据库（IDO）也可能会出现故障，因此监控它也是很有必要的（记住要监控观察者）。尽管较新的 Icinga 版本已经完全摒弃了 IDO，取而代之的是一个独立的数据库，但小型安装仍然完全可以使用 IDO。基于 PostgreSQL 的 IDO 检查定义如下（同样在 `services.conf` 中）：

```ini
object Service "ido" {
 check_command = "ido"
 vars.ido_type = "IdoPgsqlConnection"
 vars.ido_name = "ido-pgsql"
 host_name = NodeName
}
```

我们甚至不需要将此应用于任何主机，因为此检查仅在安装了 Icinga IDO 数据库的地方运行（即中央监控实例）。`host_name = NodeName` 的赋值就解决了这个问题，因为 `NodeName` 默认定义为执行检查并收集结果的主机名称。该插件定期检查 IDO 数据库，并在成功执行后输出有关 IDO 的信息：

```
Connected to the database server (Schema version: ‘1.14.3’). Queries per second:
4.633 Pending queries: 21.000. Last failover: 2022-03-23 16:05:05 +0100
```

接下来，进入另一个文件 `zones.d/my-zone/dependencies.conf`，这是我们定义服务依赖关系的地方（你猜对了）。这使我们能够声明某些服务依赖于其他服务的功能（以及它们的检查结果），并形成一个逻辑单元。

一个典型的例子是一个由数据库和 Web 服务器组成的 Web 应用程序。如果数据库故障，运行在 Web 服务器上的应用程序将无法正常工作，因此定义两者之间的依赖关系是有意义的。因此，如果数据库检查失败，Icinga 也会将 Web 服务器（或者如果应用程序也在某种方式下被监控的话，也会标记应用程序）标记为失败。这有助于确定故障的影响。如果某个服务恢复在线，其他依赖服务也需要被检查（或重新启动），以确保持续的功能性。否则，检查可能会再次报告所有状态为绿色，但应用程序可能因为数据库的丢失而受到影响，并可能需要手动干预才能修复。

在这里，我们展示的是服务仅依赖于代理健康检查的情况。

```ini
apply Dependency "agent-health-check" to Service {
parent_service_name = "agent-health"
states = [ OK ] // 如果父服务状态切换到 NOT-OK，则失败
disable_notifications = true
//自动将所有代理端点检查指定为匹配主机上的子服务
assign where host.vars.agent_endpoint
// 避免子对母的自我参照
ignore where service.name == "agent-health"
}
```

我们可以看到 Icinga 在使用其领域特定语言时的灵活性，利用诸如"apply"这样的常见元素与占位符（如 Host、Service 或 Dependency）一起定义监控的内容和方式。当检测到状态不是 OK（例如"FAILED"或"UNREACHABLE"）时，代理健康检查将被触发。为了不为每个主机单独定义这一点，并且不会忘记稍后添加的新主机，我们再次使用 `assign` 关键字将其应用于所有定义为端点的主机。

主机或服务的组帮助我们保持对具有共同任务或标准的系统的概览，比如 Web 服务器、数据库服务器、前端主机、防火墙等。这就是 `groups.conf` 所定义的内容，但在监控的基础设施较小或过于多样化以至于没有任何共同点时，这是可选的。

```ini
object HostGroup "FreeBSD-servers" {
display_name = "FreeBSD Servers"
assign where host.vars.os == "FreeBSD"
}
```

记得上面在 `hosts.conf` 中定义的主机对象"monitor.example.com"吗？我们定义了一个本地变量 `vars.os`。现在，我们可以使用"assign where"语句根据该变量的值进行过滤。自动为基础设施中新主机添加条目的工具也可能包含有关使用的操作系统（以及其他信息），因此 Icinga 会在 Icingaweb2 显示中将这些系统分组。服务组（ServiceGroups）类似地定义。通过这种方式，报告可能会包含定期检查特定服务的系统数量。Web 服务器可能会运行与数据库服务器不同的检查，但作为服务组，它很容易将这些检查整体应用于新主机，或者定义两者的混合来形成一个全新的监控目标。

我想展示的最后一个文件是 `users.conf` 文件，其中包含所有 Icinga 能理解的用户信息，当某些检查失败时会通知他们。一个基本的定义可能如下所示：

```ini
object UserGroup "icingaadmins" {
display_name = "Icinga Admin Group"
}
object User "icingaadmin" {
display_name = "Icinga 2 Admin"
groups = [ "icingaadmins" ]
email = "icinga@localhost"
}
object User "Helpdesk" {
email = "ticket@example.org"
display_name = "The Friendly Helpdesk Folks"
groups = [ "icingaadmins" ]
}
```

用户可以是其他组的一部分，例如，在这个例子中，Helpdesk 用户是 Icinga 管理员组的一部分。个别用户可能只分配给某个主机或一组服务（他们是某一领域的专家），而不是整个被监控的基础设施。

通知规则定义了在何时通过何种方式（默认是电子邮件，但也可以是寻呼机、短信，甚至是各种即时消息）联系谁。如果问题没有得到处理（或至少没有被确认），在一定时间后，可以将问题升级到其他组，以定义某些服务级别协议或对付那些（不耐烦的）客户。

其他文件组成了 Icinga 监控系统，所有这些都在文档中有详细定义。现在，我们启动所有服务，开始我们的基础监控基础设施。尤其是在添加了所有额外文件之后，Icinga 需要知道它们，因此我们重新启动该服务。


```sh
monitor# service postgresql restart
monitor# service php-fpm start
monitor# service nginx start
monitor# service icinga2 restart
```

Icingaweb2 服务通过 web 浏览器进行配置，为此需要一个令牌，因为我们不希望一个路过的陌生人偶然访问我们的新安装的监控系统并将其误配置。令牌是通过以下命令生成并发出的：

```sh
monitor# icingacli setup token create --config=/usr/local/etc/icingaweb2
monitor# chown -R www:www /usr/local/etc/icingaweb2
```

令牌现在可以从浏览器中读取，当粘贴到网页表单中后，可以进行 Icingaweb2 的其余设置步骤。填写我们创建的数据库用户等详细信息，以及管理员用户和密码等其他信息。最后，Icingaweb2 登录界面将呈现，你可以从这个中央位置访问所有被监控的主机和服务。

## 添加新主机端点

在对 Icinga 功能初步了解后，你可能会想知道如何添加更多对象进行监控。我们将通过一个新主机来演示这一过程，并展示将其纳入监控所需的所有步骤。

在一个新安装的主机（这里使用 FreeBSD）上，名为 `client.example.org`，安装 icinga2 包。


```sh
client# pkg install icinga2
```

由于这是一个基于证书的身份验证，涉及到该主机与中央 Icinga 监控实例之间的通信，我们需要确保存放证书的目录存在并且具有正确的所有权。

```sh
client# mkdir /var/lib/icinga2/certs
client# chown icinga:icinga !$
client# chown -R icinga:icinga /usr/local/etc/icinga2
```

接下来，我们启用 icinga2 服务，使其在系统启动时自动启动。

```sh
client# sysrc icinga2_enable=yes
```

接下来，使用"icinga2 pki"子命令生成客户端证书。虽然这个命令是交互式的，但我们也可以直接在命令行提供所有必要的参数，以便在以后添加数百个主机时简化自动化。请注意，这必须在中央监控实例上运行。

```sh
monitor# icinga2 pki new-cert --cn client.example.org \
 --key /var/lib/icinga2/certs/client.example.org.key \
 --csr /var/lib/icinga2/certs/client.example.org.csr"
```
以 .csr 结尾的文件是证书签名请求，它现在与之前生成的主密钥结合使用，以创建一个新的签名客户端证书（example.org.crt）。

```sh
monitor# icinga2 pki sign-csr \
 --csr /var/lib/icinga2/certs/client.example.org.csr \
 --cert /var/lib/icinga2/certs/client.example.org.crt"
```

当我们在本文开头运行 "icinga2 node setup --master" 生成主证书以签署其他证书时，系统在 /var/lib/icinga2/certs/ 目录下创建了一个名为 monitor.example.org.crt 的文件。将此文件以安全的方式传输到客户端是必要的，以便验证服务器证书。根据你对客户端及其连接的用户的信任程度，以及客户端与服务器之间的网络（或媒介），有多种方法可以实现这一操作。

```sh
monitor# scp /var/lib/icinga2/certs/monitor.example.org.crt \
 client.example.org:/var/lib/icinga2/certs/
```

接下来，将证书导入到客户端并告诉 Icinga 信任它。

```sh
client# icinga2 pki save-cert --trustedcert \
 /var/lib/icinga2/certs/monitor.example.org.crt \
 --host client.example.org"
```

在监控服务器上为客户端创建一个新票证，以建立信任关系。本质上，客户端请求成为监控基础设施的一部分。这些请求可以自动生成，并在稍后（经过人工或第三方审查后）签名。

```sh
monitor# icinga2 pki ticket --cn client.example.org
```

请注意命令行中生成的 ticket 输出（在我们的例子中是 4f76d2ecda535753e9180838ebffbcbca242fe61），我们将在下一步中在客户端使用它。客户端将从中央监控实例获取生成的票证，并生成配置文件，就像我们手动设置监视器时所做的那样。通过此操作，建立了区域关系，使得监视器成为客户端的父节点，并在它们之间建立了信任关系。此外，我们还告诉客户端接受监视器发送给它的命令和配置更改。这是可选的，客户端也可以选择独立于主机做出自己的配置选择。将配置保存在中央服务器上，并在那里控制每个客户端的配置，简化了保持所有被监控主机同步的工作。当更改诸如新的监控间隔之类的设置时，只需要设置一次，客户端会本地应用来自监视器的更改。

```sh
client# icinga2 node setup --ticket 4f76d2ecda535753e9180838ebffbcbca242fe61 \
 --cn client.example.org --endpoint monitor --zone client.example.org \
 --parent_zone my-zone --parent_host monitor.example.org \
 --trustedcert /var/lib/icinga2/certs/monitor.example.org.crt \
 --accept-commands --accept-config --disable-confd"
```

在启动 Icinga 实例之前，我们需要验证所有文件是否正确写入并符合 Icinga 的逻辑。为此，我们告诉 Icinga 守护进程执行配置检查，使用以下命令：

```sh
client# icinga2 daemon -C
```

如果有任何错误，Icinga 会尝试帮助定位相关的文件和行。典型的错误可能是为父项或区域提供了错误的名称。验证完成后，需要在监控服务器上为这个新客户端添加一个新的条目，以便将其包含在未来的检查执行中。在 monitor.example.org 上，编辑


```sh
/usr/local/etc/icinga2/zones.d/my-zone/hosts.conf
```

并添加以下配置行，确保将其放在中央 hosts 对象定义之前：

```ini
object Host "client.example.org" {
import "generic-host" // 为所有主机输入通用设置
display_name = "My Client Host"
address = "client.example.org"
vars.os = "FreeBSD"
//按照主机名 == 端点名的惯例
vars.agent_endpoint = name
}
```

Host 对象定义起来相当简单，并且没有包含我们之前添加中央服务器时尚未看到的新字段。和之前一样，我们还需要定义这个主机是一个端点（没有其他监控客户端位于它之下，并且它不是另一个主机的父级），以及它所属的区域。通常，这些条目会放在包含 'object Zone "director-global" {' 的行之前，像这样：

```ini
object Endpoint "client.example.org" {
// 客户端自己连接自己
host = "client.example.org"
log_duration = 0
}
object Zone "client.example.org" {
endpoints = [ "client.example.org" ]
parent = "my-zone" // 建立区域层次结构
}
```

在 zone 对象中，我们只需要定义端点的名称，引用我们之前在文件中定义的主机。父级区域是我们在创建监控证书时生成的区域。Icinga 配置中应该已经有它的条目。作为端点属性的日志持续时间条目指示端点在与父级的连接丢失时，如何存储所有检查结果的回放日志。待连接恢复，客户端将回放日志，并将所有数据发送给父级。由于父级负责调度所有要在监控系统上运行的检查，将其设置为零是可以的。

我们已经完成了两个主机上的配置文件工作。剩下的唯一任务就是在客户端和服务器上启动 icinga2 服务，以便读取我们所做的配置更改。

```sh
client# service icinga2 start
monitor# service icinga2 restart
```

新的客户端现在应该出现在 Icingaweb2 概览中，状态为待处理。当下一个预定的检查间隔到来时，客户端将以安全的方式进行联系（因为它们已经交换了证书，记得吗？），执行检查，并将结果发送到中央主机。

恭喜你，现在你可以开始监控你的基础设施的常见服务，并向其中添加新的主机。确保查看 Icinga 配置，以获取有关监控相关信息以及进一步配置 Icinga 安装以满足你需求的方法。

---

**BENEDICT REUSCHLING** 是 FreeBSD 项目的文档提交者，并且是文档工程团队的成员。过去，他曾任两届 FreeBSD 核心团队。他在德国达姆施塔特应用科技大学管理一个大数据集群，并教授一门“Unix 开发者”课程。他还是每周播客 bsdnow.tv 的主持人之一。

