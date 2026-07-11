# 在树莓派 3 上运行 Icinga2

作者：**Benedict Reuschling**

我的本职工作负责达姆施塔特应用技术大学用于教学和科研的 40 节点大数据集群。系统管理员的一项任务就是检查所负责的系统是否健康、是否可在网络上访问。

这通常通过监控实例定期执行检查来完成，监控实例从远程机器收集数据，并在达到某些阈值时利用这些指标触发警报。我们从中央文件服务器监控集群节点，文件服务器在不向客户端提供文件服务（基于 ZFS 的 NFSv4 导出，用于家目录和其他共享数据）时可以腾出一些 CPU 周期做这件事。

系统管理员的另一项工作，是考虑”出什么问题”的场景。这种场景可能永远不会发生，但一旦发生，你会庆幸自己早就实施了”未雨绸缪”的方案。这里的情况是”谁来监控监控者？“——如果中央监控实例宕机，会怎么样？很可能一段时间内都不会发现故障，那期间也不会通知我们其他正在发生的故障。一旦发生这种情况，我们要处理的问题不止一个，而是两个：不可用的监控系统，以及监控系统本应通知我们的那个问题。我们很快得出结论，需要第二套系统来监控主监控服务器，并在它从网络上消失时通知我们，但第二套系统自身不运行其他检查（这是可选的）。理想情况下，这应该在独立的网络上完成。如果两套系统都在同一子网，一旦整个网络不可达，我们就从两套监控系统都得不到任何信息。当然，单为监控而设立的第二套系统成本高昂（包括初期投入和电费等运行成本），所以我们需要找一个低成本的解决方案。

树莓派是长期 24/7 监控的理想工具。外形小巧、功耗低、CPU 性能足以定期检查远程系统，可以胜任多种任务。初始成本低（板子本身、一张 CF 卡、几根线缆），上手容易，不过像处理包更新这类任务可能比 amd64 系统慢一些。在 RPi3 上本地编译东西会慢得多，因此第三方软件我们依赖软件包。

首先，从 freebsd.org 下载区域的 SD Card Images 部分下载 FreeBSD-12.0-RELEASE-arm64-aarch64-RPi3.img.xz。解压后，把 SD 卡（我们用 64 GB 的；更小的也完全可以）插入下载镜像那台机器的读卡器。下面的命令行把镜像写入 SD 卡（更改设备名以匹配你的环境；小心不要覆盖其他分区）：

```sh
# dd if=/FreeBSD-12.0-RELEASE-arm64-aarch64-RPI3.img of=/dev/disk2\
bs=1m conv=sync
```

**dd(1)** 写完镜像后，卸载它，把它插入 RPi3 的 SD 卡插槽。连接其他外设，比如显示器、键盘、网线和电源。一旦通电，Pi 应该就开始启动。首次启动会花点时间根据 SD 卡大小扩展镜像。要有耐心——这只做一次，后续启动会更快。默认可以用两个用户登录：`root` 和 `freebsd`，密码与用户名相同。这是你首次登录 Pi 后应该首先修改的。然后根据自己的需求和环境做一些初始系统配置。

你可能听说过，持续向 SD 卡写入会缩短其寿命，因为构成存储介质的存储单元能承受的重写周期有限。为避免过早损坏，我们给它接了一块便宜的 32 GB SSD，并在上面创建了一个 ZFS 池。在嵌入式设备上跑 ZFS 通常不推荐，UFS 是这类设备完全合适的文件系统。令人惊讶的是，我们这个配置已经跑了几个月，没出任何问题。当然，这个 zpool 不是冗余的，池上的写入也没有大数据文件服务器那么重。你的情况可能不同，可能需要额外调优。下面是 `zfs list` 输出，展示创建的数据集：

```sh
ssd            22.4G   6.42G   22.5M  /ssd
ssd/swap       1.03G   6.45G   1.00G  -
ssd/tmp        41.6M   6.42G   41.6M  /tmp
ssd/usr        1.44G   6.42G   1.44G  /usr
ssd/usr/ports  1.08G   6.42G   1.08G  /usr/ports
ssd/var        10.8G   6.42G   10.5G  /var
ssd/var/cache  860K    6.42G   860K   /var/cache
ssd/var/db     252M    6.42G   252M   /var/db
ssd/var/log    34.8M   6.42G   34.8M  /var/log
```

我们仍然从 UFS 启动，但主要写入密集的文件系统（如 **/usr** 和 **/var**）位于 SSD 池上。操作很简单，就是创建数据集（同时启用 lz4 压缩），从 UFS 复制目录内容，再设置挂载点。挂载 **/usr** 时要格外小心，因为它会把 ZFS 数据集 **/usr** 挂载到当前系统之上，很可能造成一些中断。为避免这种情况并让 Pi 正常启动，我在 **/etc/rc.local** 中加了下面几行：

```sh
/sbin/zpool import -f ssd
/sbin/zfs mount -a
swapon /dev/zvol/ssd/swap
/usr/sbin/ntpdate -b de.pool.ntp.org
```

可以看到，我还从 SSD 跑交换分区，并在重启时从我附近的时间服务器同步时间。为完整起见，这是我在 **/boot/loader.conf** 中的条目：

```ini
# 配置 USB OTG；见 usb_template(4)。
hw.usb.template=3
#umodem_load="YES"
# 启用多控制台（串口+efi gop）。
boot_multicons="YES"
boot_serial="YES"
# 禁用 beastie 菜单和颜色
beastie_disable="YES"
loader_color="NO"
geom_label_load="YES"
verbose_loading="YES"
autoboot_delay="2"
zfs_load="YES"
```

**/etc/rc.conf** 相当简洁，只有几处改动：

```ini
hostname="rpi3.mydomain.local"
ifconfig_DEFAULT="DHCP"
sshd_enable="YES"
sendmail_enable="NONE"
sendmail_submit_enable="NO"
sendmail_outbound_enable="NO"
sendmail_msp_queue_enable="NO"
growfs_enable="YES"
keymap="de.kbd"
hostid_enable="YES"
fsck_y_enable="NO"
background_fsck="no"
mixer_enable="no"
```

重启后看 Pi 能否回到登录提示符而没有任何错误，以及能否通过 ssh 登录。这是让 Pi 在没有显示器的情况下无头运行的重要步骤。

我们希望 RPi3 在监控系统检测到异常时通知我们，于是配置 SSMTP。它提供简单、仅发送的邮件投递系统。本文中我们使用 Gmail 投递邮件。请注意所发邮件的敏感性质，因此 Gmail 可能并非所有环境下的最佳方案。请替换为你信任的邮件投递方式来发送可能遇到的系统问题邮件。

首先，安装 `ssmtp` 包：

```sh
# pkg install ssmtp
```

配置文件位于 **/usr/local/etc/ssmtp/**，这里需要修改两个文件（不存在就创建，或复制同名 `.sample` 文件再修改）：`ssmtp.conf` 和 `revaliases`。

`ssmtp.conf` 中只需要保留以下几行：

```ini
root=yourusername@gmail.com
mailhub=smtp.gmail.com:587
AuthUser=yourusername@gmail.com
AuthPass=thepasswordforthegmailaccount
UseSTARTTLS=YES
rewriteDomain=gmail.com
hostname=rpi3.mydomain.local
FromLineOverride=YES
UseTLS=YES
```

`revaliases` 文件只有两行：

```ini
root:yourusername@gmail.com:smtp.gmail.com:587
otheruser:yourotherusername@gmail.com:smtp.gmail.com:587
```

这会配置本地映射，决定哪个本地用户应该收到邮件。这里至少应配置 `root` 和一个非特权账户。定期系统邮件默认发送给 `root`，下面的邮箱地址定义了应该投递到哪里。下一行针对那个非特权用户的个人邮件同理。**/etc** 中的 `mailer.conf` 应包含以下几行（很可能由 port/package 完成）：

```ini
#sendmail
/usr/libexec/sendmail/sendmail
#send-mail
/usr/libexec/sendmail/sendmail
#mailq
/usr/libexec/sendmail/sendmail
#newaliases
/usr/libexec/sendmail/sendmail
#hoststat
/usr/libexec/sendmail/sendmail
#purgestat
/usr/libexec/sendmail/sendmail
sendmail
/usr/local/sbin/ssmtp
send-mail
/usr/local/sbin/ssmtp
mailq
/usr/local/sbin/ssmtp
newaliases
/usr/local/sbin/ssmtp
hoststat
/usr/bin/true
purgestat
/usr/bin/true
```

确保 `ssmtp` 拥有 **/var/spool/clientmqueue** 目录，使用 chown：

```sh
# chown -R smmsp:smmsp /var/spool/clientmqueue
# chmod -R 0775 /var/spool/clientmqueue
```

以 `root` 用户和那个非特权用户各发一封测试邮件给自己来测试邮件配置。你可能需要在谷歌账户设置中允许不安全应用才能发送邮件。邮件能收到后，就可以开始配置 Icinga2 这个监控系统了。

先安装 Icinga2 的包，包括 PostgreSQL 和 nginx 所需的 PHP 库，我们用 nginx 作为运行 Icingaweb2 前端的 Web 服务器：

```sh
# pkg install icinga2 icingaweb2 postgresql95-server nginx \
ImageMagick6-nox11 php72-pecl-imagick
```

启用 `icinga` 和 `postgresql` 服务在重启时运行。先不要启动这些服务，因为它们尚未配置：

```sh
# sysrc icinga2_enable=yes
# sysrc postgresql_enable=yes
```

下一步是把 PostgreSQL 配置为后端数据库。由于我们跑在 SSD 上，数据库可从一些针对 ZFS 的调优中受益。注意这并非必需，postgres 在 UFS 上也跑得很好。

创建名为 `pgdata` 的数据集，设置一些属性，并将挂载点的所有者改为 `pgsql` 用户：

```sh
# zfs create -o mountpoint=/usr/local/pgsql/data ssd/pgdata
# zfs set recordsize=8k ssd/pgdata
# zfs set logbias=throughput ssd/pgdata
# zfs set redundant_metadata=most ssd/pgdata
# zfs set primarycache=metadata ssd/pgdata
# chown pgsql:pgsql /usr/local/pgsql/data
```

后续步骤以 `pgsql` 用户身份完成，可以用 **su(1)** 切换，或者（配置好后）通过 `sudo -u pgsql`。

```sh
pgsql$ cd
pgsql$ initdb --no-locale --encoding=utf-8 --lc-collate=C -E UTF8 -D ./data
psql$ pg_ctl start -D ./data
```

这会初始化数据库集群并用 `pg_ctl` 启动数据库。Icinga 要求数据库使用 UTF-8 编码，因此设置为 UTF-8。postgresql 服务运行后，创建名为 `icinga` 的用户和数据库：

```sh
pgsql$ createuser -dPrs icinga
pgsql$ createdb -O icinga -E UTF8 icinga
```

要允许该用户访问数据库，需要在 **/usr/local/pgsql/data/pg_hba.conf** 中加入以下几行：

```ini
local  icinga  icinga                      md5
host   icinga  icinga  127.0.0.1/32        md5
host   icinga  icinga  ::1/128             md5
```

Icinga 使用自己的数据库模式存储主机、服务、用户、通知等表。这些表必须先存在，Icinga 才能开始监控。安装时附带一个 SQL 脚本，用 `psql` 设置数据库：

```sh
pgsql$ psql -U icinga -d icinga < /usr/local/share/icinga2-ido-pgsql/
schema/pgsql.sql
```

Icinga 包/port 后续更新也需要更新数据库模式。这些更新位于 **/usr/local/share/icinga2-ido-pgsql/schema/upgrades** 下，必须按顺序应用。详见 Icinga 网站上的更新说明。数据库模式建好后，退出 `pgsql` 用户。

跟 nginx 一样，Icinga 也有类似插件的机制来激活某些功能。我们的配置至少需要激活 `ido-pgsql`（因为我们用它做后端数据库）和 `command`（即通过 Icingaweb2 调度服务检查）模块。`api` 功能未来会替代 `command` 模块，但目前为稳妥起见，我们把它一并激活。

```sh
# icinga2 feature enable ido-pgsql api command
```

要告诉 Icinga 关于 PostgreSQL 数据库的细节，位于 **/usr/local/etc/icinga2/features-enabled/** 的 `ido-pgsql.conf` 文件中必须包含以下几行：

```ini
object IdoPgsqlConnection "ido-pgsql" {
  user = "icinga"
  password = "thepasswordyousetwhencreatingthedatabaseabove"
  host = "localhost"
  database = "icinga"
}
```

运行命令 `icinga api setup` 并修改 **/usr/local/etc/icinga2/conf.d/api-users.conf**，为 `icingaweb2` 设置独立的 API 用户：

```ini
object ApiUser "icingaweb2" {
  password = "somerandompasswordstrongerthanthisone"
  permissions = [ "status/query", "actions/*", "objects/modify/*",
  "objects/query/*" ]
}
```

权限字段允许该用户执行某些操作。可以为不同操作创建其他 API 用户，权限更少。这超出了本文范围，详情可查阅 Icinga2 文档（<https://icinga.com/docs/>）。

是时候重启数据库并启动 icinga2 了：

```sh
# service postgresql restart
# service icinga2 start
```

把 nginx 配置为 icingaweb2 的 Web 服务器。需要以下步骤：

```sh
# sysrc php_fpm_enable=yes
# sysrc nginx_enable=yes
# sed -i '' "s/listen\ =\ 127.0.0.1:9000/listen\ =\ \/var\/run\/php5-fpm.sock/" /usr/local/etc/php-fpm.d/www.conf
# sed -i '' "s/; listen.owner/listen.owner/" /usr/local/etc/php-fpm.d/www.conf
# sed -i '' "s/; listen.group/listen.group/" /usr/local/etc/php-fpm.d/www.conf
# sed -i '' "s/; listen.mode/listen.mode/" /usr/local/etc/php-fpm.d/www.conf
```

在位于 **/usr/local/etc/nginx/nginx.conf** 的 nginx 配置文件中，在 `location / { ... }` 部分之前加入以下配置段：

```nginx
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

注意，配置 SSL 超出了本文范围，但借助 letsencrypt 之类的服务可以相当简单地完成。

从随包附带的 `php.ini-production` 文件创建一份 `php.ini`：

```sh
# cp /usr/local/etc/php.ini-production /usr/local/etc/php.ini
```

编辑该文件，把 `date.timezone` 一行设为服务器的时区。

Web 服务器和 php-fpm 现在可以启动了：

```sh
# service nginx start
# service php-fpm start
```

打开浏览器，访问 RPi3 的 IP/DNS 名，末尾加上 **/icingaweb2/setup**。icingaweb 的安装向导受令牌保护，需要从命令行创建令牌：

```sh
# icingacli setup token create –config=/usr/local/etc/icingaweb2
# chown -R www:www /usr/local/etc/icingaweb2
```

按步骤操作，并在提示时提供 PostgreSQL 数据库信息（用户名、密码）。维护 icinga2 port 的 Lars Engels 写过一篇博客文章，带你完成设置（<http://lme.postach.io/post/installing-icinga-web-2-with-apache-2-4-icinga-2-and-mysql-on-freebsd>）。

## 配置要监控的主机和服务

Icinga 检查分为主动检查和被动检查。主动检查在被监控主机自身执行，结果发送回 icinga 服务器做进一步处理。被动检查由 icinga 服务器从主机外部执行，如 ping。我们只介绍被动检查；不过主动检查在树莓派上也工作得很好。被监控主机的配置文件位于 **/usr/local/etc/icinga2/conf.d/hosts.conf**，我的默认配置中已包含 icinga 服务器自身。用以下模板添加新主机：

```nginx
object Host "myhost" {
  import "generic-host"
  address = "ip.address.of.myhost"
  vars.notification["mail"] = {
    groups = [ "icingaadmins" ]
  }
}
```

重启 icinga 之前，务必让它先检查配置文件，再重启服务：

```sh
# icinga2 daemon -C && service icinga2 restart
```

该主机现在应该出现在 icingaweb2 界面中。树莓派会持续定期联系该主机，并在主机或服务状态从 UP 变为 DOWN 或反过来时发出警报。

Icinga 提供大量功能。根据要检查的主机和服务数量，RPi3 可能需要一些时间处理。对于少量主机/服务，这不会成为问题，是具备大量自定义选项的成本效益方案。 •

---

**Benedict Reuschling** 于 2009 年加入 FreeBSD Project。2010 年获得完整文档提交权限后，他积极指导其他人成为 FreeBSD 提交者。他于 2015 年加入 FreeBSD 基金会，目前担任副主席。Benedict 拥有计算机科学理学硕士学位，在德国达姆施塔特应用技术大学讲授面向软件开发者的 UNIX 课程。他与 Allan Jude 共同主持每周播出的 BSDNow.tv（<http://BSDNow.tv>）播客。
