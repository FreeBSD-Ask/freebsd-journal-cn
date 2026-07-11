# 什么是 Icinga？

作者：Lars Engels 和 Benedict Reuschling

在网络监控的早期，有一种名为 Netsaint 的监控软件，后因法律原因改名为 Nagios。Nagios 在开源世界非常成功，但来自社区的补丁常常被拒绝或非常缓慢地被采纳。2009 年，Nagios 社区成员创建了一个分支，命名为 Icinga（祖鲁语意为”它审视”）。自 2014 年发布 Icinga 2.0 以来，它已不再是 Nagios 的分支，而是一次彻底重写，加入了分布式监控、HA 集群、REST API 等新功能。Icinga 可重用所有 Nagios 监控插件，同时提供现代 Web 界面和对其核心功能的强大扩展。

## 在 FreeBSD/ZFS 上安装

本文中，我们将演示在 FreeBSD 上搭建 Icinga 和 Icinga Web 2 来监控主机。配置数据、事件和用户认证将存储在 PostgreSQL 数据库中（默认也提供 MySQL 支持）。NGINX 将为 Icinga Web 2 提供 Web 页面服务。

假设已有一份 FreeBSD 基本安装并已连接网络、能下载软件包且能通过 ping 和 ssh 访问其他主机。这套设置已在树莓派 3 上测试过，也在普通服务器上测试过，不需要任何特殊硬件或调优参数。

Icinga 文档（<https://docs.icinga.com>）对必要的 Icinga 安装步骤有出色描述，甚至还提供了 FreeBSD 专属说明（例如路径不同时）。FreeBSD port/软件包也可用，并已包含一些在其他 Unix 发行版上用户仍需手动执行的步骤。

> **注意**
>
> 本指南专注于让 Icinga 2 跑起来，不关注性能、SSL 设置和其他安全选项（密码除外）。待你更熟悉 Icinga 2 及其功能后，可自行深入这些主题。

在 FreeBSD 上安装 Icinga 2 最简单的方式是使用软件包 **net-mgmt/icinga2**。本指南在以 `#` 开头的命令前表示应由 root 用户执行，`$` 表示非 root 用户。或者可用 sudo 临时提升权限并以其他用户身份执行命令。

## 安装 Icinga 2

我们先安装几个 setup 所需的软件包（见框 1）。Icinga 2 会安装基本监控组件。Icinga Web 2 是可选组件，用于在基于浏览器的仪表盘中显示事件和检查结果。撰写本文时，PostgreSQL 9.5 是所用数据库服务器。

虽然这里描述的是 NGINX 的设置，但 Apache 2 用户同样能运行 Icinga Web 2。

软件包 ImageMagick-nox11 和 pecl-imagick 需要单独安装，因为默认不会被作为依赖拉入。当我们开始配置 Icinga Web 2 时，如果未安装这些组件它会报警告。这些组件用于在生成报告时创建 PDF 输出中的图形。没有它们，报告 PDF 只会有文字形式的统计和指标。其他依赖（如 Icinga Web 2 使用的 PHP）会由这里列出的软件包自动拉入。

**框 1**

```sh
# pkg install icinga2 icingaweb2 postgresql95-server nginx ImageMagick-nox11 pecl-imagick
```

所有组件安装完毕后，我们用 **sysrc(8)** 添加条目（见框 2）到 **/etc/rc.conf**，让这些组件在重启后自动启动。

**框 2**

```sh
# sysrc icinga2_enable=yes
# sysrc postgresql_enable=yes
```

## 设置 PostgreSQL

在开始设置 PostgreSQL 数据库并为 Icinga 创建用户之前，我们先创建一个 ZFS 数据集来存放数据（UFS 用户可只用 mkdir 跟着做）。在以下示例中，将 `monitor` 替换为你的池名：

**框 3**

```sh
# zfs create -o mountpoint=/usr/local/pgsql/data monitor/pgdata
# zfs set compression=lz4 monitor/pgdata
# zfs set recordsize=8k monitor/pgdata
# zfs set logbias=throughput monitor/pgdata
# zfs set redundant_metadata=most monitor/pgdata
# zfs set primarycache=metadata monitor/pgdata
# chown pgsql:pgsql /usr/local/pgsql/data
```

被监控的系统越多，数据库的写入就越频繁，所以我们相应地调优 ZFS。使用这些设置，数据库事务不仅在磁盘上压缩，还会尊重数据库使用的元数据设置，因此 ZFS 不会在写盘时自作聪明地认为自己比 PostgreSQL 更懂。

现在以 `pgsql` 用户登录，并用刚创建的数据目录（数据集）运行 initdb 创建数据库集群：

**框 4**

```sh
# su pgsql
$ cd
$ initdb -D ./data -E UTF8
$ pg_ctl start -D ./data
```

我们创建一个新数据库用户 `icinga`、一个同名数据库，并将 `icinga` 用户指定为所有者。提示时为 `icinga` 用户提供口令。稍后设置 Icinga Web 2 时会用到，确保不要忘记。

**框 5**

```sh
$ createuser -dPrs icinga
$ createdb -O icinga -E UTF8 icinga
```

`pg_hba.conf` 文件在 initdb 执行时生成，控制哪些用户可以访问数据库。为让 `icinga` 用户仅能本地连接数据库，编辑 data 目录下的 `pg_hba.conf`，添加以下行：

**框 6**

```ini
local      icinga     icinga                              md5
host       icinga     icinga          127.0.0.1/32        md5
host       icinga     icinga          ::1/128             md5
```

icinga 数据库需要几张表、几个索引和引用，才能记录来自主机的监控数据。该 schema 由 port/软件包提供，位于 **/usr/local/share/icinga2-ido-pgsql/schema/pgsql.sql**。用 psql 命令解释器直接执行该 SQL 脚本：

**框 7**

```sh
$ psql -U icinga -d icinga < /usr/local/share/icinga2-ido-pgsql/schema/pgsql.sql
```

脚本成功执行后，数据库相关的配置步骤就完成了。继续之前请退出 `pgsql` 用户。Icinga 通过 IDO（Icinga Data Out to Database）连接数据库，Icinga 需要知道这一点。Icinga 2 有一个类似插件的接口，可以启用和禁用功能。要让 IDO 与 PostgreSQL 数据库配合工作，我们需要启用 `ido-pgsql` 功能。另一个需要的功能叫 `command`。之后，Icinga 2 首次启动。

**框 8**

```sh
# icinga2 feature enable ido-pgsql
# icinga2 feature enable command
# service icinga2 start
```

Icinga 现在基本运行起来，会对本地机器执行检查。拥有一个仪表盘来获取所有报告和事件的概览要好得多，所以接下来我们配置 Icinga Web 2 和 NGINX Web 服务器。

## 为 Icinga Web 2 配置 NGINX

Icinga Web 2 运行在 PHP 上，我们用 PHP FPM 让它与 NGINX 协同工作。首先，启用 `php-fpm` 和 `nginx` 在系统重启时启动：

**框 9**

```sh
# sysrc php_fpm_enable=yes
# sysrc nginx_enable=yes
```

然后用几条 sed 命令配置 php-fpm 配置文件。第一行让它监听本地域套接字，下面三行取消注释 owner、group 和 mode 选项，使用 FreeBSD 提供的值（即 `www` 用户和套接字的默认权限）。

**框 10**

```sh
# sed -i '' "s/listen\ =\ 127.0.0.1:9000/listen\ =\ \/var\/run\/php5-fpm.sock/" /usr/local/etc/php-fpm.conf
# sed -i '' "s/; listen.owner/listen.owner/" /usr/local/etc/php-fpm.conf
# sed -i '' "s/; listen.group/listen.group/" /usr/local/etc/php-fpm.conf
# sed -i '' "s/; listen.mode/listen.mode/" /usr/local/etc/php-fpm.conf
```

Icinga Web 2 在 **/usr/local/share/examples/icingaweb2/nginx/icingaweb2.conf** 中提供了一个 nginx.conf 示例文件。我们取相关部分，放到 **/usr/local/etc/nginx/nginx.conf** 中 `location / {` 行之前：

**框 11**

```nginx
location ~ ^/icingaweb2/index\.php(.*)$ {
    # fastcgi_pass 127.0.0.1:9000;
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

PHP 也要配置。所幸 FreeBSD 提供了一份适合生产环境使用的 PHP 配置文件，名为 `php.ini-production`。我们将此文件复制到 `php.ini` 文件所在位置：

**框 12**

```sh
# cp /usr/local/etc/php.ini-production /usr/local/etc/php.ini
```

还需向其中添加一行，让 PHP 知道所处时区。请用你的监控服务器所在时区替换下面的示例。

**框 13**

```sh
# echo "date.timezone = Europe/Berlin" >> /usr/local/etc/php.ini
```

启动 PHP-FPM 和 NGINX Web 服务器：

**框 14**

```sh
# service nginx start
# service php-fpm start
```

如果一切正常，打开浏览器访问以下 URL 开始 Icinga Web 2 配置：<http://local.domain.or.ip/icingaweb2/setup>。

如果出了问题，回顾上述步骤，并查看 **/var/log/icinga2** 下的日志文件寻找线索。newsyslog 的日志轮转示例可在 **/usr/local/share/examples/icinga2/newsyslog** 下找到。

接下来，我们配置 Icinga Web 2 仪表盘。

## 配置 Icinga Web 2

第一个欢迎界面会要求输入 setup token。这是为了防止有人意外闯入新装的 Icinga 服务器并误配置它（记住，这可能是恶意的互联网）。要创建必要的 setup token，运行以下命令：

**框 15**

```sh
# /usr/local/www/icingaweb2/bin/icingacli setup token create \
  --config=/usr/local/etc/icingaweb2
# chown -R www:www /usr/local/etc/icingaweb2
```

复制屏幕上回显的 setup token 并输入到 setup token 字段。点击 Next 继续。下一屏会显示安装检测到的模块。确保至少勾选“Monitoring”，然后点击 Next。为确保 Icinga 安装包含所有必需组件（主要是 PHP 模块），Icinga 会提供一个概览页，显示本地安装中检测到的内容。按本指南操作时，这些字段大多应为绿色，表示该模块存在且可用。“Linux Platform”字段可安全忽略，因为该软件在 FreeBSD 上同样运行良好。再次点击 Next。

这一屏询问用户将如何对 Icinga Web 进行认证。我们使用先前设置的 PostgreSQL 数据库，但 LDAP 在企业环境中同样可用。选择“database”后，继续到下一页输入连接信息。确保数据库类型选“PostgreSQL”；localhost 和端口应已正确。数据库名输入 `icinga`。用户名和口令即 postgresql 设置中提供的。编码输入 `UTF-8`。是否持久连接到数据库可自行决定，频繁使用 Web 界面时通常是好主意。可验证连接以确认一切正常，再继续到下一页。

此页询问使用哪个认证后端，应已提供默认条目。更多认证后端可稍后在 Icinga Web 2 中添加，此处直接继续。在下一屏输入 Icinga Web 2 管理员账户的凭据（用户名和口令）。继续前确保记住口令。

下一屏处理调试（是否创建用户可见的堆栈跟踪）、设置存储位置以及日志记录方式（日志类型和级别）。如果不想让 syslog 被 Icinga 2 消息塞满，记录到单独文件是好主意。切换到“file”时，运行 mkdir **/var/log/icingaweb2** 后跟 chown www:www **/var/log/icingaweb2**，让 Web 界面能写入日志。

下一页汇总所有设置，满意后继续到下一页配置监控组件。后端是 IDO，我们之前已用于 PostgreSQL 数据库，所以从这里继续。我们希望将所有监控信息存储在 PostgreSQL 中，因此需要输入先前使用的相同连接信息。点击 Next 前通过验证确认设置正确。

当使用多个 Icinga 实例时，命令传输很重要，但我们的场景接受本地文件传输设置并继续。无需更改默认提供的受保护变量。又一个汇总页之后，我们终于可以完成 Icinga Web 2 的设置并首次登录。使用设置早期创建的管理员账户进入监控仪表盘。

## 监控主机和服务

监控是一个复杂的话题，Icinga 有多种方式监控远程系统。Icinga 2 文档有大量示例可用。

最简单的方式是通过简单 ping 检查，我们以此为例配置一台主机。打开 **/usr/local/etc/icinga2/conf.d/hosts.conf** 并添加如下条目（框 16）：

**框 16**

```sh
object Host "Example" {
    import "generic-host"
    address = "IP-ADDRESS"
}
```

将“Example”和“IP-ADDRESS”分别替换为主机名和 IP 地址。保存并退出文件后，让 Icinga 2 检查其配置（`# service icinga2 configcheck`）并重启 `icinga2` 服务。Icinga Web 2 界面现在应显示一个新的待处理主机，稍后该主机会被 ping 定期检查。如果主机宕机，Icinga 2 会在仪表盘中显示告警，并在主机重新可达时移除告警。

## Icinga 还能做更多……

Icinga 可通过更多功能扩展，超越对机器及其服务的监控。例如，在层级式监控大量对象的复杂环境中，业务流程模块能帮助可视化。这样，构成某服务基础的数千台云机器可从一个高层级仪表盘查看。一旦出现告警，该模块允许你下钻到触发告警的具体机器。[https://github.com/Icinga/icingaweb2-module-businessprocess]

Icinga Director 让处理 Icinga 2 配置变得轻松。允许用户通过”指点和点击”灵活创建自己的对象，同时完全自动化其数据中心，系统管理员可以确信其监控解决方案会随着基础设施需求增长而扩展。[https://github.com/Icinga/icingaweb2-module-director]

另一个模块为 Icinga 2 添加了通用工单系统（TTS）功能。它允许在 Icinga Web 2 中用工单系统（TTS）的链接替换工单模式。它定义了一个工单钩子，可被核心监控模块等用于确认、停机和评论。[https://github.com/Icinga/icingaweb2-module-generictts]

最后，处理分布在众多地理位置的系统的系统管理员，可通过 Icinga Web 2 的地图模块获得更好的概览。它使用 OpenStreetMap 数据在地图上显示主机对象及其状态。同一位置的多个主机会聚在一起。有自定义主机操作可在地图上定位特定主机。点击主机标记会弹出显示该主机服务当前状态。[https://github.com/nbuchwitz/icingaweb2-module-map]

此外，还有一个模块将 Grafana 图形集成到 Icinga Web 2 中，让你能以美观的方式显示检查的性能数据。[https://github.com/Mikesch-mp/icingaweb2-module-grafana]

所有这些附加模块都可作为 FreeBSD ports/软件包提供。务必研究 Icinga 2 文档，以有效方式监控你的主机和服务。•

---

> **LARS ENGELS** 已是 FreeBSD ports 提交者 10 年，并松散地参与 Icinga 项目。
>
> **BENEDICT REUSCHLING** 于 2009 年加入 FreeBSD 项目。2010 年获得完整文档提交权限后，他开始指导他人成为 FreeBSD 提交者。他是 BSD 认证小组的监考人，现任 FreeBSD 基金会副总裁。
