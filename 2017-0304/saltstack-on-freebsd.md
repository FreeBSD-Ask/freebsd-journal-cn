# FreeBSD 上的 SaltStack：入门指南

作者：Peter Wright

配置管理有许多风格和理念。从简单的基于 Makefile 的自动化任务，到完全编排的部署和管理环境，可以采取的方法多种多样。

本文将介绍 SaltStack——一个基于 Python 的配置管理解决方案，功能丰富，以易用和可扩展著称。SaltStack 在对管理员屏蔽各种操作系统特定的实现细节方面也做得很好；例如，在 GNU/Linux 和 FreeBSD 上安装原生 OS 软件包使用的是相同的语法。这让管理异构环境的工作变得轻松许多，同时在可能的情况下鼓励代码复用，而不论目标平台是什么。

SaltStack 也为大规模部署而设计。这主要归功于其采用发布-订阅消息模型的架构决策——Master 不必为每个受管节点单独构造配置指令，而是可以在客户端（Salt 术语中称 Minion）订阅的频道上广播指令，由 Minion 自行判断该指令是否适用于自己。通过 pub/sub 方式的通信也是加密的，并支持用户可配置的密钥轮换。

SaltStack 还实现了一个对大型部署有益的功能，即所谓的事件驱动基础设施（Event-Driven Infrastructure，EDI）。它允许 Salt 客户端对本地系统状态的变化、或 SaltStack 自身外部的事件作出反应。这对于容量根据整体系统负载动态管理的云基础设施而言是个受欢迎的功能。本文旨在作为入门指南，将快速浏览 SaltStack 的架构，说明开始使用它的门槛其实相当低。我们还将通过一个在 FreeBSD 上配置 Redis 服务器的真实示例，最后指出一些值得读者自行深入探索的有趣功能。

## 架构概览

SaltStack 的架构相对直观，但得益于若干设计决策，它实际上相当可扩展，也易于按特定环境的需要定制。SaltStack 集群中的主节点称为 Master，客户端系统称为 Minion。Master 负责为 Minion 发布消息。Master 还通过接受 Minion 的初始密钥、并跟踪 Minion 自动轮换的新密钥，来管理谁是某集群的有效成员。

在 Master/Minion 通信方面，SaltStack 使用 ZeroMQ 作为其 pub/sub 消息队列的基础。SaltStack 还有 Grain 的概念，类似于 Puppet 中的 Facter，用于存储系统变量，如操作系统名称、可用核心数以及某 Minion 的主 IP 地址。这在开发模板时非常有用，后文将予以展示。SaltStack 还有 Pillar 的概念，用于存储用户自定义变量，如文件路径甚至密码。SaltStack 中的模板通过 Jinja 实现，可用于在配置中插入逻辑。它们通常会同时利用 Grains 和 Pillars，从而带来更大的灵活性和更简洁的配置指令。

## 准备主机以使用 SaltStack

使用 **pkg(7)** 安装 SaltStack 非常简单；事实上，安装 SaltStack Master 或 Minion 用的是同一个软件包。

```sh
% sudo pkg install py27-salt
```

我们将跳过 salt_master 和 salt_minion 守护进程的配置，因为 SaltStack 文档对此介绍得很充分。配置好两个系统并成功启动 salt_master 守护进程后，就可以启动 salt_minion，它会连接到 Master 并发送初始公钥以待接受。你可以在 Master 上使用 `salt-key` 命令查看并接受密钥：

```sh
$ sudo salt-key
Accepted Keys:
Denied Keys:
Unaccepted Keys:
    test0.com.puter
Rejected Keys:
```

上面可以看到，我们有一个来自“test0.com.puter”的待处理密钥。在接受该密钥前，可以在 Master 上运行 `salt-key -f test0.com.puter` 查看并确认该密钥的指纹与 Minion 上的指纹匹配。确认密钥后，即可接受它：

```sh
$ sudo salt-key -a test0.com.puter
The following keys are going to be accepted:
Unaccepted Keys:
    test0.com.puter
Proceed? [n/Y] y
Key for minion test0.com.puter accepted.
```

至此，我们已有一个 Minion 与 Master 安全配对，可以用 SaltStack 内置的一个允许执行任意命令的函数来验证。这里我们指示 Master 发布一条消息，要求所有匹配 `test*` 正则表达式的主机运行 `hostname` 命令。任何匹配该正则的系统都会执行 `cmd.run` 的参数，并将输出发回消息队列供 Master 消费：

```sh
$ sudo salt 'test*' cmd.run 'hostname'
test0.com.puter:
    test0.com.puter
-------------------------------------------
Summary
-------------------------------------------
# of minions targeted: 1
# of minions returned: 1
# of minions that did not return: 0
# of minions with errors: 0
-------------------------------------------
```

这个示例实际上告诉我们关于 SaltStack 的其他几件有趣的事。首先，Salt 命令接受正则表达式作为输入，以指定要对其执行命令的主机。所以如果我有一组主机名中包含共同元素的系统，我可以通过 Salt CLI 工具简洁地一次命中它们。其次，Salt 支持对已注册 Minion 运行临时命令。例如，利用这个功能，我可以执行一个临时查询来验证我所有 Web 服务器都已安装相应的 Apache 2.4 软件包。但让我们看看 SaltStack 更传统的用法——向新节点应用一个配置状态（称为 SaltState）。

## 熟悉 SaltStack 命令

上述命令在 Master 上调用时，以所有主机名匹配 `test*` 正则的已配置 Minion 为目标。你可以用 `grains.items` 替代 `grains.get` 查看给定节点的所有可用键值对。能在状态配置文件中嵌入模板和逻辑很强大；但 SaltStack 也允许你对文件资源使用模板，接下来就讲这个。

```sh
$ sudo salt 'test*' grains.get os
test0.iad0.tribdev.com:
    FreeBSD
------------------------------------
Summary
------------------------------------
# of minions targeted: 1
# of minions returned: 1
# of minions that did not return: 0
# of minions with errors: 0
------------------------------------
```

## 使用 SaltState 文件管理 Minion

SaltStack 配置指令存储在所谓 statefile 中，以 `.sls` 扩展名标识。每个 SaltStack 安装都有一个所谓的”top file”，定义整体结构以及 SaltState 与受管主机的关联。出于我们的目的，从一个非常简单的 top file 开始，它把两个状态关联到我们的节点：

```yaml
base:
  '*':
    - motd

dev:
  'test*':
    - redis_demo
```

SaltStack statefile 遵循标准 YAML 语法和层级规则。在上面的示例中，第 1–3 行定义了一个匹配所有系统的 base 类，关联了一个名为 `motd` 的 salt state 文件。motd statefile 如下：

```jinja
{% if grains['os'] == 'FreeBSD' %}
/etc/motd:
  file.managed:
    - source: salt://global_files/motd_freebsd
    - user: root
    - group: wheel
    - mode: 644
{% endif %}
```

这个示例 statefile 中嵌入了 Jinja 模板代码，在我们的场景中，它判断我们运行所在的主机是否为 FreeBSD，若是，就把 `motd_freebsd` 文件以指定的属主和权限应用到主机。执行该 statefile 的 Minion 会搜索其 grain 库存，检查 `os` 键的值是否匹配 FreeBSD。如果匹配，就把 FreeBSD motd 文件复制到 **/etc/motd**，并确保其属主和权限。你也可以通过 SaltStack CLI 与 Grains 交互：

## 环境隔离小插曲

这是了解 SaltStack 如何在磁盘上组织文件的好时机。我的 Master 上，statefile 和文件资源的目录结构如下：

```sh
$ cd $SALT_ROOT
$ find .
./states/top.sls
./states/global_files/motd_freebsd
./states/dev/states/redis_demo.sls
./states/dev/states/dev_files/redis_demo.conf
./states/dev/states/dev_files/redis_rc
```

值得注意的是，`top.sls` 文件与 `redis_demo.sls` 文件位于不同位置；`redis_demo.sls` 是 **dev/states** 目录层级的子项。我这里所做的，是用这种结构创建了环境隔离。具体而言，根级别包含 `top.sls` 文件和 `motd_freebsd` 文件。此处的状态作用于所有主机，因为它们具有全局作用域。然后我创建了一个 `dev` 目录结构，其中包含我们的 Redis statefile 和配套的文件资源。这个 dev 环境在前面的 top 文件中通过第 5–7 行体现，如第 6 行所示，我们通过正则表达式对匹配的主机更加严格。SaltStack 中也可以有任意数量的环境，非常方便。例如，在我较大的 SaltStack 部署中，除了预生产和生产环境外，还有若干开发环境。这让我能按环境划分访问权限，同时也有助于确保在一个环境中做的改动不会影响其他环境。

## 更复杂的 Statefile

了解了环境和 SaltStack 文件系统布局后，让我们看看 `redis_demo.sls` 文件，它安装若干软件包并引用一个模板化的文件资源：

```yaml
/usr/local/etc/redis.conf:
  file.managed:
    - require:
      - pkg: redis_pkgs
    - source: salt://dev_files/redis_demo.template
    - template: jinja
    - user: root
    - group: wheel
    - mode: 644

/var/db/redis/:
  file.directory:
    - user: redis
    - group: redis
    - mode: 755

/etc/rc.conf.d/redis:
  file.managed:
    - source: salt://dev_files/redis_rc
    - user: root
    - group: wheel
    - mode: 644

redis_pkgs:
  pkg.installed:
    - pkgs:
      - redis
```

第 1–9 行定义了部署到 **/usr/local/etc/redis.conf** 的文件的特征。第 3–4 行创建了对第 24–27 行定义的软件包的依赖，要求在该文件资源部署前先安装这些软件包。接着，第 5 行引用了一个 Jinja 模板作为我们的 Redis 主配置文件。最后，第 11–22 行确保存在一个具有正确权限的目录，供 Redis 将数据检查点写入磁盘；我们还通过 rc 确保 Redis 守护进程已启用。让我们看看 `redis_demo.template`，它进一步展示了如何利用 SaltStack Grains 和模板编写一个可部署到多个系统的配置文件：

```jinja
{% set pri_ipv4 = grains['ipv4'][0] %}
protected-mode no
port 6379
tcp-backlog 511
bind {{ pri_ipv4 }}
timeout 0
tcp-keepalive 300
```

以上片段是 Redis 配置文件的前 9 行，展示了 Grains 和模板如何以相当强大的方式组合使用。第一行定义了一个变量 `pri_ipv4`，并用 SaltStack Grains 系统报告的 `ipv4` 键中的第一个值填充它。要在 shell 中查看该 Grain 的样子，可以执行以下命令：

```sh
$ sudo salt 'test*' grains.get ipv4
test0.iad0.tribdev.com:
    - 10.3.16.51
    - 127.0.0.1
-----------------------------------
Summary
-----------------------------------
# of minions targeted: 1
# of minions returned: 1
# of minions that did not return: 0
# of minions with errors: 0
-----------------------------------
```

SaltStack 以列表形式返回这些值，因此我们必须指定列表中代表我们要使用的值的元素。所以，当 Minion 评估这个 salt stage 时，它会扫描 grain 系统中名为 `ipv4` 的键，并用存储的公共 IPv4 地址设置 `pri_ipv4` 变量。然后该数据按第 6 行所示插入 Minion 上的本地配置文件，其中 `{{ pri_ipv4 }}` 被替换为 IP 地址。这是一个非常简单的示例，但其威力应当相当明显。例如，设想你管理着跨越多个子域的一群系统。利用 Grains 和模板，你可以在模板中抽象掉许多按域和按系统的配置，甚至用逻辑决定在给定 Minion 上渲染模板的哪些部分。SaltStack 还让你能通过编写简单的 Python 类来轻松扩展默认 Minion。

## 应用 SaltState 配置

既然我们已经过完 statefile，是时候把它们应用到我们的 Minion 了。我们从 Master 执行以下命令：

```sh
$ sudo salt 'test*' state.apply
```

这条命令把所有相关状态——如 `top.sls` 文件中所定义——应用到主机名以 `test` 开头的系统。输出如下：

```sh
$ sudo salt 'test*' state.apply
test0.iad0.tribdev.com:
----------
    ID: /etc/motd
    Function: file.managed
      Result: True
     Comment: File /etc/motd is in the correct state
     Started: 23:01:06.397619
    Duration: 581.967 ms
     Changes:
----------
    ID: redis_pkgs
    Function: pkg.installed
      Result: True
     Comment: The following packages were installed/updated: redis
     Started: 23:01:07.450629
    Duration: 1107.676 ms
     Changes:
              ----------
              redis:
                  ----------
                  new:
                      3.2.6
                  old:
----------
    ID: /usr/local/etc/redis.conf
    Function: file.managed
      Result: True
     Comment: File /usr/local/etc/redis.conf updated
     Started: 23:01:08.560307
    Duration: 535.906 ms
     Changes:
              ----------
              diff:
                  ---
                  +++
                  @@ -1,1052 +1,82 @@
                  -# Redis configuration file example.

<省略 redis.conf 的多页 diff>

----------
    ID: /var/db/redis/
    Function: file.directory
      Result: True
     Comment: Directory /var/db/redis is in the correct state
     Started: 23:01:09.096296
    Duration: 0.803 ms
     Changes:
----------
    ID: /etc/rc.conf.d/redis
    Function: file.managed
      Result: True
     Comment: File /etc/rc.conf.d/redis updated
     Started: 23:01:09.097172
    Duration: 521.206 ms
     Changes:
              ----------
              diff:
                  New file
              mode:

Summary for test0.iad0.tribdev.com
------------
Succeeded: 5 (changed=3)
Failed:
------------
Total states run:
Total run time:   2.748 s
-------------------------------------------
Summary
-------------------------------------------
# of minions targeted: 1
# of minions returned: 1
# of minions that did not return: 0
# of minions with errors: 0
-------------------------------------------
```

成功了！我们安装了 Redis 二进制文件，部署了自定义的 Redis 配置文件，还确保 **/etc/motd** 已部署且为最新。我移除了 SaltStack 在覆盖默认 `redis.conf` 时报告的一段非常长的 diff。让我们检查一下配置，确保模板和 Grain 数据正确应用：

```sh
$ head -n 10 /usr/local/etc/redis.conf
protected-mode no
port 6379
tcp-backlog 511
bind 10.3.16.51
timeout 0

tcp-keepalive 300
$ ifconfig xn0 | grep inet
    inet 10.3.16.51 netmask 0xffffff00 broadcast 10.3.16.255
$
```

大功告成！`ipv4` grain 的第 0 个值已替换进我们的配置文件，并且与该主机上 `xn0` 关联的 IPv4 地址一致。

## 结语

本文对 SaltStack 作为配置管理引擎的能力只是浅尝辄止。尽管如此，希望它对 SaltStack 的能力给出了不错的概览，并展示了这款工具的灵活性。实践中还有几个我在演示中完全略过的关键概念和组件，值得读者自行翻阅 SaltStack 文档。第一个我完全忽略的概念是 `nodegroups`，这是一种把功能相似的节点分组的方法。例如，如果我有 10 个充当 Apache 服务器的 Minion，可以在 Master 配置中创建一个名为 `webservers` 的 nodegroup。然后就可以在 state 文件中引用这个 nodegroup，从而不必像前述示例那样局限于主机名正则。以下是在 `top.sls` 中引用 nodegroup 的示例：

```yaml
dev:
  web-servers:
    - match: nodegroup
    - dev_base
```

上面的示例会对我放入 `webservers` nodegroup 的所有 Minion 应用 `dev_base` salt state。我在演示中绕过的第二个功能是 Salt Pillars。它们与 Salt Grains 非常相似，也是一个可在 statefile 或模板中查询的键值注册表。Pillars 不同于 Grains 之处在于，它们用于用户自定义变量。一个常见的用例是把数据库凭证存为 Pillar 的键值对，然后由 Minion 上的配置模板引用。此外，SaltStack 有一个 GPG 渲染器，可接收 GPG 加密的输入，并在 Pillar 中以解密形式渲染供 Minion 使用。这种方法让管理员可以把凭证存储在源代码控制系统（SCCS）中，同时确保加密载荷本身受到保护。

---

**PETER WRIGHT** 是一名系统架构师，目前在 Tronc Inc. 工作，为出版业构建基于 FreeBSD、SaltStack 和 AWS 的可扩展、安全系统。他是 NYCBUG 的长期成员（尽管住在圣莫尼卡），总是乐于把人引进 FreeBSD 的世界。不折腾 Unix 的时候，你多半会在他最爱的海滩上找到他正和儿子一起冲浪。

## 参考链接

- SaltStack 入门（官方教程）：<https://docs.saltstack.com/en/getstarted/>
- SaltStack 文档：<https://docs.saltstack.com/en/latest/contents.html>
- SaltStack Nodegroups：<https://docs.saltstack.com/en/latest/topics/targeting/nodegroups.html>
- SaltStack GPG 渲染器：<https://docs.saltstack.com/en/latest/ref/renderers/all/salt.renderers.gpg.html>
