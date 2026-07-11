# 用 Prometheus 进行监控与趋势分析

作者：Ed Schouten

Prometheus 由 SoundCloud 的软件工程师于 2012 年创建，是一个监控系统，其设计灵感来自 Borgmon——负责监控谷歌内部 Borg 集群上运行任务的系统。本文将通过实践方式讨论 Prometheus 的工作原理。在这个过程中，我们还会考察一些常与 Prometheus 配合使用的其他工具，例如 Grafana 以及一些 metrics 导出器。在文章末尾，我们还会介绍一款我们正在开发、近期将以开源软件形式发布的自动化工具，名为 Promenade。

## Prometheus 入门

虽然使用 Go 的构建工具（`go get`）已经能很方便地安装 Prometheus，FreeBSD Ports 套件中也提供了打包版本。安装该软件包的一个好处是它附带了 **rc(8)** 脚本，让我们可以方便地以守护进程方式运行。运行以下命令后，Prometheus 应该就能启动运行：

```sh
$ pkg install prometheus
…
New packages to be INSTALLED:
        prometheus: 1.7.1
…
Proceed with this action? [y/N]: y
…
$ sudo /usr/local/etc/rc.d/prometheus forcestart
Starting prometheus.
```

默认情况下，Prometheus 会在 9090 端口绑定一个 HTTP 服务器。如果我们用浏览器访问该服务器，会看到一个页面，可以浏览 Prometheus 中存储的数据。由于我们刚启动这个实例，能探索的数据当然不多。让我们暂时先把这个页面放一边，进入”targets”页面（见下方两张截图）。

targets 页面向我们展示了 Prometheus 正在监控哪些端点，并且已经透露出一些有趣的信息。这个 Prometheus 实例被配置为监控自身。Prometheus 是一种白盒监控系统，这意味着它能存储由目标自身报告的指标，而不仅仅是测量外部可见的因素（例如 TCP 和 HTTP 健康检查）。通过点击最左侧列中的 HTTP 链接，我们可以查看 Prometheus 服务器生成的原始指标：与 Go 运行时的垃圾回收、线程、HTTP 处理以及指标存储相关的统计信息（见右图）。

Prometheus 通过网络提供指标的格式相当简单。每个指标占据 HTTP 响应中的一行，具有一个 64 位浮点数值。每个指标由一个名称和花括号内一组可选的标签（键值对）唯一标识。标签允许一个程序返回同名但内容不同的多个指标。例如，Web 服务器可以用标签为每个注册的虚拟主机或每个 HTTP 错误码类别返回 HTTP 统计信息（例如，仅返回 **<www.freebsd.org>** 的 HTTP 200 类响应的延迟）。

为了区分由不同目标返回的同名指标，Prometheus 在摄入数据时会附加额外的标签，例如 `job` 和 `instance`。这些标签包含能唯一标识端点的值。这些标签的值会显示在 Prometheus 的 targets 页面上。在 Kumina，我们利用这一机制附加与我们环境相关的自定义标签，例如物理位置（数据中心名称）、系统所有者（客户名称）和支持合同（7×24 小时或仅办公时间）。这些标签随后可作为查询和告警条件的一部分。

Prometheus 自身无法获取任何操作系统指标，例如 CPU 负载、磁盘使用情况和网络 I/O。它只是一种通过 HTTP 摄入指标并建立索引的工具。系统级指标由一个叫做 node exporter 的工具提供。node exporter 本质上是一个 Web 服务器，被访问时通过 **/dev**、sysctl、libkvm 等方式提取内核级状态，并以 Prometheus 的指标格式返回。在 FreeBSD 上安装 node exporter 非常简单：

```sh
$ sudo pkg install node_exporter
…
$ sudo /usr/local/etc/rc.d/node_exporter forcestart
Starting node_exporter.
```

启动后，我们要扩展 Prometheus 的配置，让它也抓取 node exporter。默认情况下，node exporter 监听 9100 端口。

```sh
$ sudo vim /usr/local/etc/prometheus.yml
# 在“scrape_configs”下添加以下条目：
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
$ sudo /usr/local/etc/rc.d/prometheus onerestart
Stopping prometheus.
Starting prometheus.
```

重启 Prometheus 并刷新 targets 页面后，可以看到它现在抓取两个目标，这正是我们期望的结果（见右图）。

除了 node exporter 之外，此时我们还能配置许多其他目标。常见的服务都有对应的导出器（例如 MySQL、Nginx、Java JMX），它们将指标从原生格式转换为通过 HTTP 提供的形式。Prometheus black box exporter 能执行 ICMP、DNS、TCP、HTTP 和 SSH 检查，并报告可用性和延迟。FreeBSD 12.x 提供了 **prometheus_sysctl_exporter(8)**，它可以提供任意 sysctl 的值。其中一些导出器由 Prometheus 项目官方维护，另一些则由社区维护。

在 Kumina，我们为 Dovecot、PHP-FPM、libvirt、OpenVPN 和 Postfix 等服务开发了导出器。我们还设计了一个基于 libpcap 的简单网络流量统计守护进程，按地址导出统计信息，名为 Promacct。所有这些工具都可以在我们公司的 GitHub 页面（<https://github.com/kumina>）找到。

如果你想用 Prometheus 从内部开发的软件中获取指标，不必使用独立的指标导出器进程。Prometheus 项目为多种编程语言（Go、Java、Python、Ruby 等）提供了客户端库，使你可以直接在代码中用指标对象进行标注。对于 Python 和 Java 等语言，这些库还提供了便捷的函数装饰器，可自动统计函数调用次数并创建其运行时间的直方图。

## PromQL：Prometheus 的查询语言

让 Prometheus 收集几个小时指标后，我们可以进入图形页面来探索 Prometheus 的数据集。我们先绘制一条由 node exporter 生成的指标 `node_network_receive_packets`。顾名思义，这一指标对应系统网络接口接收到的网络数据包数量。这个表达式在我的系统上生成一个包含两条线的图：一条对应环回接口（device=”lo0”），另一条对应物理接口（device=”em0”）。

如果我们只想绘制某些主机或网络接口的指标，可以在表达式末尾追加过滤器。例如，向查询追加 `{device!~"lo[0-9]+"}` 会通过负向正则匹配移除环回设备的指标。`{datacenter="frankfurt"}` 会只返回某个数据中心的系统结果（如果存在这样的标签）。

直接绘制网络数据包数量似乎看不出什么有用信息，只渲染出两条对角线。原因是 node exporter 将网络数据包数量报告为随时间累积的计数器。只有当运行 node exporter 的系统重启（或计数器发生整数溢出）时，它才会重置为零。让导出器使用累积计数器，可让多个 Prometheus 服务器安全地抓取它，也使导出器无需关心 Prometheus 服务器上配置的抓取间隔。即使 Prometheus 服务器因高负载需要减少抓取次数，也不会有数据包被遗漏。

要把数据包数量转换为有意义的值，我们首先要计算其导数。这可以通过对指标执行两个操作来实现。首先，我们不查询指标的标量值，而是请求连续样本的向量，称为范围向量。然后我们可以用 `rate()` 函数将这些范围向量转回标量，表示范围向量每秒的增长速率。所用的范围向量大小将决定结果图形的平滑度。5 分钟范围向量适合发现短暂的流量突发，而 1 小时范围向量可以可视化日常流量曲线。在 Kumina，我们甚至使用 7 天和 31 天范围向量进行长期网络和 CPU 容量规划。将图形表达式改为 `rate(node_network_receive_packets[5m])`，我们就能得到一个使用 5 分钟速率计算、显示每秒接收数据包数的图形。

Prometheus 的查询语法还支持聚合操作，可以减少绘制的指标数量。例如，查询 `sum(rate(node_network_receive_packets[5m])) by (instance)` 会按服务器计算带宽，而不是按网络接口显示带宽。查询 `topk(10, rate(node_network_receive_packets[5m])) by (datacenter)` 会将输出限制为每个数据中心前 10 名。

## 用 Grafana 创建仪表盘

虽然 Prometheus 内置的 Web 应用适合交互式地运行查询，但人们往往希望设计出有条理地展示图形的仪表盘。Prometheus 曾附带一个名为 Promdash 的功能，允许服务器提供模板化的 HTML 文件。该功能现已移除，因为现在有多种第三方仪表盘工具可以直接与 Prometheus 对接，其中最突出的是 Grafana。在 FreeBSD 上让 Grafana 工作起来基本上遵循我们之前看到的相同套路：

```sh
$ sudo pkg install grafana4
…
$ sudo /usr/local/etc/rc.d/grafana forcestart
Starting grafana.
```

按默认设置，Grafana 会启动一个监听 3000 端口的 Web 服务器。用浏览器访问它并以默认凭据（用户名 `admin`，密码 `admin`）登录后，我们便看到 Grafana 的主界面。我们想做的第一件事是点击“Add data source”配置 Grafana 应使用的 Prometheus 服务器地址——本例中是 <http://localhost:9090/>。

完成后，我们可以点击主界面上的下一个按钮，标题为“Create your first dashboard”。随后我们会看到一个空的仪表盘页面，可以在上面放置面板，例如图形、表格、热力图和列表。对于 Prometheus，大多数情况下使用图形最为合理。创建图形时，我们可以使用与之前相同的查询语法。通过页面顶部的保存图标，仪表盘会保存在 Grafana 服务器上。

根据硬件规格，你可能会发现随着图形数量增加、查询变得复杂，仪表盘渲染时间会变长。同时执行许多复杂查询可能给 Prometheus 服务器带来显著负载。为解决这一问题，Prometheus 提供了在抓取时预计算复杂查询并将结果以不同名称存储的能力，这称为录制规则（recording rules）。经验法则：当图形查询比简单选择表达式更复杂时，就应该使用录制规则。

```sh
$ sudo vim /usr/local/etc/prometheus-rules.yml
# 添加以下内容：
my_recording_rule =
    sum(rate(node_network_receive_packets[5m]))
    by (instance)
$ sudo vim /usr/local/etc/prometheus.yml
# 在“rule_files”下添加以下条目：
  - prometheus-rules.yml
$ sudo /usr/local/etc/rc.d/prometheus onerestart
```

上面的命令展示了如何配置 Prometheus 使用录制规则。本例中我们添加了一个名为 `my_recording_rule` 的录制规则，从现在起它可在图形查询中使用。录制规则可以任意命名，但为了提高可读性，应用一些最佳实践是有意义的。Prometheus 文档中有一篇文章介绍了开发者建议的命名方案。对于上面的录制规则，Prometheus 文档建议将其命名为 `instance:node_network_receive_packets:rate5m`。

## 告警

除了绘图外，Prometheus 还能根据指标值生成告警。可以通过声明告警规则来配置告警，告警规则可放在与录制规则相同的文件中。下面是一个告警规则示例：

```sh
ALERT TargetFailedToScrape
   IF up == 0
   FOR 15m
   LABELS { severity = "page" }
   ANNOTATIONS {
      summary = "Instance {{ $labels.instance }} is down!",
      playbook_url = "http://intranet.company.com/...",
      ...
   }
```

当名为 `up` 的指标至少 15 分钟保持为零时，这个告警会触发。`up` 指标由 Prometheus 隐式创建，用于表示它是否成功抓取了某个目标。告警表达式中使用的指标所附带的标签也会附加到告警本身。这些标签对于格式化用户友好的告警消息很有用，也可用于创建”静默”（silences）——即应当暂时抑制的告警模式（例如因计划维护）。Prometheus 会在其“Alerts”页面上显示所有已注册的告警规则及其状态。

为了保持设计简洁，Prometheus 服务器只支持一种机制来通知活动告警，即向其他服务发送 REST 调用。Prometheus 项目提供了一个独立的守护进程 Alertmanager，可以处理这些 REST 调用，生成电子邮件、SMS 和 Slack 消息，并管理静默。Prometheus 用于执行 REST 调用的 URL 可以通过 `--alertmanager.url` 命令行标志配置。可能还需要将 `--web.external-url` 标志设置为 Prometheus 服务器的公共 URL，这样 Alertmanager 就能在其告警消息中添加指向 Prometheus 的可点击链接。

## 联邦：Prometheus 服务器的层级结构

在某些场景下，用单个 Prometheus 服务器为整个基础设施收集指标并不合适。要抓取的目标数量和要摄入的指标数量可能对单个 Prometheus 实例而言过大。目标也可能在地理上分散，意味着没有一个位置能可靠地抓取所有目标。为解决这一问题，Prometheus 服务器可在 **/federate?match[]=...** 接收 HTTP GET 请求，以自身格式输出选定的已存储指标，使其他服务器可以摄入它们。

实践中你会发现，这种机制常被用来引入层级。一个跨多数据中心部署的服务可以在每个地点部署一台 Prometheus 服务器，只抓取附近的系统。这些 Prometheus 服务器随后用录制规则为整个数据中心聚合关键指标，再由一台全局 Prometheus 服务器抓取。在排查问题时，可以首先查看全局 Prometheus 实例提供的仪表盘来确定哪些数据中心受影响。要获取系统级的指标访问权限，可以切换到相应的本地实例。

这种层级设置的优点是可以显著减少磁盘空间占用。本地 Prometheus 实例可配置为使用易失性存储并具有较短的保留期（天数），而全局实例可使用持久化存储并具有很长的保留期（年），因为它只需要保留数据的一小部分。将数据保存多年对长期容量规划很有用。

## 用 Python 通过 Promenade 创建仪表盘

在 Kumina，我们注意到大规模管理录制规则颇具挑战。重构 Prometheus 配置文件总有破坏现有 Grafana 仪表盘的风险，尤其是因为仪表盘存储在基于 Web 的工具中，而不是与 Prometheus 配置文件一同存放在版本控制系统中。因此我们正在实现一个工具，可以通过编程方式配置 Prometheus 和 Grafana 安装，名为 Promenade。

用 Promenade，你可以编写 Python 类来声明 Grafana 仪表盘，如下所示：

```python
class UnboundMetricsBuilder:
   def construct(self):
       yield Dashboard(
          title='Unbound',
          rows=[
              DashboardRow(title='Queries', graphs=self._row_queries()),
              ...
          ])
   def _row_queries(self):
       yield Graph(
           title='Query rate per data center',
           queries=[
              GraphQuery(
                 expression=Sum(
                    expression=Rate(
                       expression=Metric('unbound_queries_total'),
                       duration=datetime.timedelta(minutes=5)),
                    by={'datacenter'}),
                 format='Data center %(datacenter)s')
           ],
           unit=Unit.OPERATIONS_PER_SECOND,
           stacking=Stacking.STACKED,
           width=WIDTH_FULL // 2)
        yield Graph(
           ...
```

Promenade 一个有趣的方面是它可以考虑 Prometheus 服务器的层级，这同样可以用 Python 代码声明。在下面的代码中，我们先声明一个 DAG（有向无环图）描述指标上的标签之间如何相互关联（例如，所有具有相同 `datacenter` 标签值的指标也会有相同的 `country` 标签值）。然后我们声明一组希望为其生成配置文件的 Prometheus 服务器对象的层级。

```python
label_implications = LabelImplications({
   ('instance', 'customer'),
   ('instance', 'rack'),
   ('rack', 'datacenter'),
   ('datacenter', 'country'),
   ...
})

local = ScrapingRecordingServer(label_implications)
global = CompositeRecordingServer(
   label_implications, 'datacenter', {local}, ...)
```

将仪表盘的 Python 代码与该层级结合后，Promenade 会在本地 Prometheus 实例上自动创建如下录制规则：

```sh
datacenter:unbound_queries:rate5m =
     sum(rate(unbound_queries_total[5m]))
     by (datacenter, country)
```

它还会为全局 Prometheus 实例生成一个配置文件，使其从所有本地实例抓取 `datacenter:unbound_queries:rate5m`，这样 Grafana 就可以访问它们。

在 Kumina，我们目前正将所有现有的 Prometheus 和 Grafana 配置迁移到基于 Promenade 之上，这也是 Promenade 的设计仍在调整以满足我们需求的原因。我们计划在代码稳定后立即在我们公司的 GitHub 页面（<https://github.com/kumina>）发布，敬请关注！

## 结语

希望本文能让你感受到，开始用 Prometheus 监控系统是多么容易。在 Kumina，我们使用 Prometheus 已约一年，深感满意。它是一个稳健、灵活且可扩展的监控系统，拥有健康的开发者和用户生态。未来几个月内，我们将看到 Prometheus 2.0 发布，其中会包含大量新功能。在 Kumina，我们最期待的是存储层的重新设计，它将让我们能在商用硬件上每分钟收集数百万个样本。

自 2016 年起，Prometheus 团队每年举办一次名为 PromCon（<https://promcon.io/>）的会议。今年的会议于 8 月 17—18 日在德国慕尼黑举行。如果你想了解 Prometheus 项目的最新进展，请务必查看会议网站，所有演讲的录像都公开提供。•

---

> **ED SCHOUTENS** 是 Kumina（位于荷兰埃因霍温的托管服务提供商和咨询公司）的首席软件开发者。Kumina 为企业提供完全托管平台，并提供 Prometheus 和 Kubernetes 的支持、培训和咨询。欢迎访问我们的网站 <https://kumina.nl/> 或发送邮件至 <info@kumina.nl>，告诉我们你的项目或索取关于我们服务的更多信息。
