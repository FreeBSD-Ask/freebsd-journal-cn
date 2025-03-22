# 实用软件：使用 Zabbix 监控主机

- 原文链接：[Practical Ports: Monitor Your Hosts with Zabbix](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-10th-anniversary/practical-ports-monitor-your-hosts-with-zabbix/)
- 作者：Benedict Reuschling

我想了解发生了什么，尤其是我负责的服务器和机器发生了什么。监控这些系统已经成为我的一种好习惯。每天需要检查的机器实在太多了，而且在大多数时候，也不会发生什么特别的事情。但当出现问题时，我也希望能够知道：哪怕是在我睡觉时，或者不在终端旁边的时候。监控也能让我对我的机器群有个整体了解。有时我会发现某个主机已经完成了它的任务，但还未被回收。甚至有时候，我发现可以把新服务迁移到一台未充分利用的机器上，而不是启动一台新主机和 jail。

图表在显示机器在一段时间内的活动方面对我非常有帮助。上周的系统负载是否异常？还是大学的实验小组开始活动了？那块硬盘逐渐满了，最好去处理一下。不时会出现像这样的问题，而通过图表展示的机器指标可以帮助我回答这些问题。

任何一位好的系统管理员都会因为知道他们的系统在运行而睡得更香。监控软件能告诉你它们正在运行，你甚至可以为管理层提供每台机器的正常运行时间报告，制成 PPT。

有一款能够做这些事情的监控系统，它叫做 Zabbix。它是一种开源的 IT 基础设施监控解决方案，由一家公司开发，必要时还能提供专业支持。我使用的完整功能版本运行在长期支持周期中，我发现其安装过程有很好的文档支持。一台中央服务器会从运行代理的系统中收集数据。服务器提供了 Web UI，里面有仪表板、图表、某些事件的警报等功能。对我来说，另一个好处是 FreeBSD 并非一款陌生的操作系统，Zabbix 提供了机器模板来监控重要的指标，比如 CPU、内存，哪怕是 ZFS。

我在 jail 中运行 Zabbix，未遇到过什么大问题。这个基于 PHP 的解决方案需要一款数据库来存储指标。可以使用 Postgres、MySQL/MariaDB、SQLite，甚至是 IBM 和 Oracle 的商业 DB2 数据库服务器。监控本身通过 SNMP/IPMI、SSH 和简单的 ping 来检查可用性。对于主动监控（收集实时机器指标），需要在主机上安装代理。Zabbix 还提供了监控整个子网的功能，可检测新主机，在它们出现时自动将其添加到监控中。还可以为特定的主机和特定情况（例如磁盘已满）设置触发器，可以通过 Web 界面配置这些触发器。可以通过电子邮件、Jabber、SMS 和自定义脚本操作发送相关事件的警报。

## 安装设置 Zabbix

接下来，让我们看看如何安装这个监控解决方案。我从新的 FreeBSD 14.0 jail 开始，这个 jail 连接到我正在监控的网络，并且从 Web 或者单独的 Poudriere 服务器下载所需的包。请注意，我将 Port 默认的 MySQL 配置改为 PostgreSQL，可以通过在 `net-mgmt/zabbix64-server` 目录下运行 `make config` 来进行更改。

我们需要安装以下包：

```sh
# pkg install zabbix64-server zabbix64-frontend-php82 postgresql15-server nginx
```

在之前的尝试中，我使用的是 `zabbix6-server` 和前端，因为我以为 6.4 版本最终会变成 6.5 版本。但 Zabbix 的发布周期有所不同。长期支持版本是主要版本（此处为 6 版），而小版本发布（如 6.4）的支持周期较短。我转而使用长期支持版本，因为我不想总是处于监控软件的最前沿，而更倾向于当前功能集被使用过一段时间。稳定性是我的首要目标。你不会希望每次新版本发布时都去修复监控软件，尤其是当你依赖它来监控关键设备时。

使用 `sysrc` 启动时启用相关服务：

```sh
# sysrc zabbix_server_enable=yes
# sysrc zabbix_agentd_enable=yes
# sysrc postgresql_enable=yes
# sysrc nginx_enable=yes
# sysrc php_fpm_enable=yes
```

在我的配置中，我没有使用 SNMP（以后可能会变），但是 `pkg-message` 里有关于如何启用 SNMP 守护进程的详细信息。如果你需要的话，可以参考。

在安装包之后，我们的首要任务是设置 Zabbix 数据库。我喜欢使用 Postgres，所以在这个设置中我使用了 Postgres。正如前文所述，也支持其他的数据库，你可以根据自己的喜好选择数据存储解决方案。

### 数据库设置

首先，我切换到 postgres 用户，该用户是作为安装包的一部分进行安装的。

```sh
# su postgres
```

切换到 `/var/db/postgres`（这是 postgres 的主目录）后，我运行命令 `initdb` 来初始化数据库集群。然后，我使用命令 `pg_ctl` 启动数据库，因为我们需要一个运行中的数据库来导入 Zabbix 的基础表。

```sh
$ cd /var/db/postgres
$ initdb data
$ pg_ctl -D ./data start
```

在数据库启动后，我切换到以下目录：

```sh
$ cd /usr/local/share/zabbix64/server/database/postgresql/
```

在这儿，存放着数据库模板文件，用于创建表、触发器以及其他相关数据库对象。

接下来，我运行 PostgreSQL 的交互式命令行工具 `psql`。在 psql 中，我创建了一个新的数据库 `zabbix`，还为其创建一个同名的用户、设置了密码。在授予该用户对 `zabbix` 数据库的所有权限后，我退出了 psql，切换到我们刚刚创建的 `zabbix` 用户。

```sql
psql -d template1
psql> create database zabbix;
psql> CREATE USER zabbix WITH password 'yourZabbixPassword’;
psql> GRANT ALL PRIVILEGES ON DATABASE zabbix to zabbix;
psql> exit
```

再次使用 psql 作为新创建的 zabbix 用户登录到空的 `zabbix` 数据库中。然后，我们依次加载包含 Zabbix 表定义（以及其他一些数据库对象）的三个文件：`schema.sql`、`images.sql` 和 `data.sql`，这些文件位于我们之前切换到的本地目录中。完成后，我们可以再次退出数据库。

```sql
$ psql -U zabbix zabbix
psql> \i schema.sql
psql> \i images.sql
psql> \i data.sql
psql> exit
```

至此，数据库设置完成。接下来，我们配置 Zabbix。

### 配置 Zabbix

Zabbix 需要知道用哪款数据库来存储监控指标、监控哪些机器以及其他大量的配置信息。Zabbix 的主要配置文件位于 `/usr/local/etc/zabbix64/zabbix_server.conf`，它是一个简单的 `键=值` 格式文件。文件中的注释介绍了每个配置项的作用，大多数配置项是被注释掉的。在完成必要的修改后，我的配置文件如下所示：

```ini
SourceIP=IP.address.of.monitoring-host
LogFile=/var/log/zabbix/zabbix_server.log
DBHost=
DBName=zabbix
DBUser=zabbix
DBPassword=yourZabbixPassword
Timeout=4
LogSlowQueries=3000
StatsAllowedIP=127.0.0.1
```

我首先定义了中央 Zabbix 服务器的 SourceIP 地址，用于收集监控指标。请注意，前端（Web UI）不一定要与监控服务器位于同一主机，但我在这里将它们集中管理。然后，我定义了 Zabbix 运行时发生事件的日志文件路径（LogFile）。如果尚未创建该路径和文件，我需要手动创建之，再将文件所有者设置为 Zabbix 用户（这是随安装包创建的用户）。

另外，我并未忘记为 DBHost 配置项设置值。当使用 PostgreSQL 时，此项应该为空。以后，如果你需要自定义配置文件，你可能会想起这一点，再做出不同的配置。对我而言，这样可以减少输入，当然，使用其他数据库时可能需要设置 DBHost 的值，所以如果你不使用 PostgreSQL，请根据实际情况进行调整。

我们在数据库设置中创建了 `DBName`、`DBUser` 和 `DBPassword`（明文密码）。这些配置项将 Zabbix 连接到数据库实例，以便在监控过程中存储和检索数据。我使用了 `Timeout` 和 `LogSlowQueries` 的默认值。到目前为止我没有调整它们，但我知道，如果遇到资源限制，调整这些值会是个好办法。Zabbix 本身还会收集一些内部统计信息，参数 `StatsAllowedIP` 定义了可以接受这些统计信息的 IP 地址。对我的使用场景而言，localhost 已完全足够。如果你有多个 Zabbix 实例并且希望它们互相监控，那么你可以在此处定义多个 IP 地址。

这就是 Zabbix 服务器配置文件的基本设置。我使用的是代理监控方式来监控客户端和服务器本身。如果你使用 SNMP，可能还需要其他配置项。详细信息可查阅 Zabbix 的文档。

### 配置 Zabbix Agent

代理程序名为 Zabbix Agentd（经典的 Unix 守护进程），应该在服务器 jail 启动时运行它。可以通过命令 `sysrc` 在 `/etc/rc.conf` 添加启动项，如下所示：

```sh
# sysrc zabbix_agentd_enable=yes
```

与服务器配置文件相邻的还有代理的配置文件。代理配置文件名为 `zabbix_agentd.conf`，位于 `/usr/local/etc/zabbix64/zabbix_agentd.conf`。在我做了一些自定义修改后，文件内容如下所示：

```sh
LogFile=/var/log/zabbix/zabbix_agentd.log
SourceIP=IP.address.of.host-to-monitor
Server=IP.address.of.monitoring-host
ServerActive=127.0.0.1
Hostname=Zabbix server
```

我们可以在此文件中找到一些熟悉的配置项。应把 `LogFile` 设置为一个不同的文件，以便区分来自服务器和客户端的消息，尤其是服务器端的消息。被监控的机器一般不会运行服务器部分，因此它们不会与服务器日志混淆。目录应该已经存在，但若没有，可使用命令 `touch` 手动创建该文件。`SourceIP` 与监控主机的发送指标相同。对于其他客户端，可以将其设置为目标主机的 IP 地址或者 DNS 地址。`ServerIP` 在所有系统中保持一致，因为我们使用的是集中式系统来收集监控数据。`ServerActive` 前面已经讨论过，而 `Hostname` 是 Zabbix 内部使用的标识符，用于区分不同的系统。

### Zabbix 前端

Zabbix 前端需要一台已启用 PHP 的，正在运行的 Web 服务器，且需要一个自己的配置文件。前端需要知道如何从后台获取指标和其他数据，这也是另一个配置文件需要数据库凭证的原因。该文件位于 `/usr/local/www/zabbix64/conf/zabbix.conf.php`，正好与其余的 Zabbix 网站文件放在一起。文件中有几个注释，指引你填入适当的值。我的配置如下所示：

```php
<?php
// Zabbix GUI 配置文件
  $DB['TYPE']             = 'POSTGRESQL';
  $DB['SERVER']           = 'localhost';
  $DB['PORT']             = '0';
  $DB['DATABASE']         = 'zabbix';
  $DB['USER']             = 'zabbix';
  $DB['PASSWORD']         = 'yourZabbixPassword';
  $DB['SCHEMA']           = '';
```

关于 TLS 加密的额外设置留给读者自行研究。强烈建议采用 TLS 加密，否则如果有恶意监听者在网络中间，可能会获取到发送监控数据的主机的许多信息。我在这里并未讨论 TLS 配置，以便教程专注于如何让 Zabbix 正常运行。

我选择的 Web 服务器是 nginx，但其他 Web 服务器也可以正常工作。修改后的 nginx 主配置文件 `nginx.conf` 如下所示：

```ini
worker_processes 1;
events {
  worker_connections 1024;
}
http {
  include             mime.types;
  default_type        application/octet-stream;
  sendfile            von;
  keepalive_timeout   65;
  server {
    listen            80;
    server_name       localhost;
    root /usr/local/www/zabbix64;
    index index.php index.html index.htm;
    location / {
      try_files $uri $uri/ =404;
    }
    location ~ .php$ {
      fastcgi_split_path_info ^(.+\.php)(/.+)$;
      fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
      fastcgi_param PATH_INFO $fastcgi_path_info;
      fastcgi_param REMOTE_USER $remote_user;
      fastcgi_pass unix:/var/run/php-fpm.sock;
      fastcgi_index index.php;
      include fastcgi_params;
    }
    error_page    500 502 503 504   /50x.html;
    location = /50x.html {
      root   /usr/local/www/nginx-dist;
    }
}}
```

请注意，这里也缺少了现在 Web 上普遍采用的 SSL/TLS 设置。我使用了一个 SSL 代理来处理请求，但使用 Let’s Encrypt 解决方案既便宜又容易实现。再次强调，这里我专注于 Zabbix 本身。

对于 PHP 配置，有几项设置需要调整才能在生产环境中使用。编辑 `/usr/local/etc/php-fpm.d/www.conf` 文件，并将以下设置修改为：

```ini
listen = "/var/run/php-fpm.sock"
```

取消注释以下行：

```ini
listen.owner
listen.group
listen.mode
```

`/usr/local/etc/php.ini` 文件应该使用生产环境值，因此请将 `/usr/local/etc/php.ini-production` 复制再重命名为 `php.ini`。我们还没有完全完成配置，因为 Zabbix 安装后会检查一些 PHP 设置，并且会报错某些值过低（或完全错误）。为了顺利完成配置，请修改这些值并将时区更改为你所在的时区：

```ini
date.timezone = Europe/Berlin
post_max_size = 16M
max_execution_time = 300
max_input_time = 300
```

保存这些文件后，执行命令 `sysrc nginx_enable=yes`，再启动所有 Zabbix 相关服务：

```sh
service postgresql restart
service php-fpm start
service nginx start
zabbix_server start
zabbix_agentd start
```

### 网站问题

这一切顺利完成，我在 Zabbix 的日志文件中看到了几行新的记录（服务器和代理端都有）。我兴奋地打开浏览器，输入 Zabbix 的链接，结果什么也没有发生。页面上只有一片空白。于是我重启了服务，检查了日志文件，重新核对了配置文件，依然是同样的空白页面。我换了一个浏览器，排除浏览器故障，但结果还是一样的白屏。

每当你在 BSD 大会上遇到我时（这应该是很有可能的发生事情），都会注意到我的头发已经很少了。我可以归咎于基因和环境因素，但从事 IT 工作无疑是导致我在这种情况下抓狂的一个大原因。按照文档，这应该是能正常工作的，那为什么网页上什么都没有显示呢？

如果不是运气让我找到了正确的方向，这篇文章就会在这里以一个不令人不悦的结尾告终。我打开了 Zabbix 的常见问题解答，正巧看到这一段：

```ini
If “opcache” is enabled in the PHP 7.3 configuration, Zabbix frontend may show a blank screen when loaded for the first time. This is a registered PHP bug. To work around this, please set the “opcache.optimization_level” parameter to 0x7FFFBFDF in the PHP configuration (/usr/local/etc/php.ini file). https://www.zabbix.com/documentation/current/en/manual/installation/known_issues
```

```ini
如果在 PHP 7.3 配置中启用了 “opcache”，Zabbix 前端可能在首次加载时显示空白页面。这是一个已知的 PHP 错误。为了解决这个问题，请将参数 “opcache.optimization_level” 设置为 0x7FFFBFDF，并将其添加到 PHP 配置中（`/usr/local/etc/php.ini` 文件）。 https://www.zabbix.com/documentation/current/en/manual/installation/known_issues
```

### 解决方案

好的，我打开文件 `/usr/local/etc/php.ini`，并将 `opcache.optimization_level` 设置为：

```ini
opcache.optimization_level=0x7FFFBFDF
```

虽然这个值看起来有些晦涩，但当你，亲爱的读者，进行这个设置时，这个设置可能是必要的，亦可能不是。我保留了这个设置，以避免再次遇到麻烦。在重启服务后，我终于见到了 Zabbix 的 Web 配置界面。设置过程主要是确认 PHP 配置和后台数据库的值，设置 Web 前端的密码，然后第一次登录。

你可能会发现，在新安装的 Zabbix 中没有任何主机。要至少看到你自己的监控服务器，请点击左侧的“监控”（大眼睛图标），然后点击“主机”。在新页面中，点击右上角的“创建主机”按钮。会出现一个表单，你可以在其中输入主机的名称、IP 地址和使用的监控方式（在我们的例子中是 Zabbix Agent）。模板字段提供了某些类型主机的默认值，当你开始输入操作系统名称时，我们会找到一款适用于 FreeBSD 的模板。组会将具有相似特征的主机逻辑地组合在一起（你可以创建任意数量的组）。这使得过滤变得更容易，若发生故障，可能会告诉你哪些其他系统需要关注。在接口部分，点击“添加”再选择 Agent。新的输入字段将出现，你可以在其中输入 IP 地址/DNS 名称。默认的代理端口是 10050，因此请检查是否有防火墙规则阻止访问。简介是可选的，但随着服务器数量的增加，提醒自己每个系统的目的还是有用的。点击蓝色的“添加”按钮创建主机。若一切顺利，它会出现在你的主机列表中，并且很快就会有代理收集的数据和一些图表显示出来。仪表板将显示监控期间发现一切问题。

![](https://freebsdfoundation.org/wp-content/uploads/2024/02/reuschling_fig2.png)

查看 Zabbix 文档，了解如何监控整个子网，如何使用 SNMP 监控，以及如何自动添加主机，而无需在 Zabbix 中逐一点击每个主机。探索 Zabbix UI 的其他功能，发现你以前没有注意到的那些新功能，定期查看，看看是否有异常情况。祝你监控愉快！

---

**BENEDICT REUSCHLING** 是 FreeBSD 项目的文档提交者，也是文档工程团队的成员。过去，他曾担任了两届 FreeBSD 核心团队成员。他在德国达姆施塔特应用技术大学管理着一个大数据集群，并为本科生教授“Unix 开发者”课程。Benedict 还是每周播客 bsdnow.tv 的主持人之一。

