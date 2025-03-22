# 实用 Port：Prometheus 的安装与配置

- 原文链接：[Prometheus Installation & Setup](https://freebsdfoundation.org/wp-content/uploads/2023/01/reuschling_promethius.pdf)
- 作者：**BENEDICT REUSCHLING**

长时间以来，监控系统及其运行服务一直是系统管理员工作的一部分，主要有以下几个目的：

- 系统是否可达（可用性）
- 系统在上周日凌晨 4 点做了什么？（指标）
- 在不寻常的情况下（例如高负载进程）向 IT 人员发出警报（通常在不合时宜的时间）（告警）

系统管理员通常会在一个中央位置收集这些指标以供进一步分析和可视化，这比登录到各个系统并运行 `tail -F /var/log/messages` 或其他日志文件更为高效。通过查看 CPU 使用率的峰值或月初可用磁盘空间急剧下降的图表，问题何时开始就显而易见。当配置了警报时，系统会在特定事件发生时发送通知（例如，系统是否可达？）或者当某些阈值被触及时（例如，剩余磁盘空间只有 10%）。这些传统上是由 Munin、CheckMK、Nagios 或 Zabbix 等软件完成的。

Prometheus 是一个相对年轻的开源监控项目，尽管如此，它已经在 Kubernetes 和云环境中成为了一个成熟的解决方案，并且也可以在其他环境中使用。对我来说，设置过程出乎意料的简单，尤其是在我已经长时间使用 Telegraf、InfluxDB 和 Grafana 的情况下。Grafana 在这里同样用于仪表板的可视化。正如其名所示，InfluxDB 充当收集的指标的中央存储位置，Grafana 从中提取数据。数据发送是通过在每台机器上运行 Telegraf 实现的，Telegraf 会定期（例如每 10 秒）将其指标发送到 InfluxDB。

Prometheus 的架构类似：一个中央 Prometheus 服务器负责转换、存储和简化接收到的数据，所谓的 node_exporter 在客户端机器上收集指标。同样，Grafana 使用 Prometheus 数据，并通过一些现成的仪表板来展示数据，这不仅可以给你的同事留下深刻印象，还能同时提供实用价值。

其他组件包括告警管理器，当某些事件发生时，它会发送各种可配置的通知类型（电子邮件、短信、寻呼机、聊天消息）。要查询数据，Prometheus 提供了一个名为 PromQL 的查询语言，Grafana 可以理解并用于仪表板内容。用户还可以使用 PromQL 编写自己的临时查询，从而快速进行搜索，而无需先构建仪表板。

提取、格式化和发送指标的逻辑编码在导出器中。针对特定软件（如数据库）有多个不同的导出器可供使用。像 RabbitMQ、GitLab 和 Grafana 本身这样的应用程序允许将它们的应用状态导出为与 Prometheus 兼容的格式进行监控。Prometheus 是用 Go 编写的，具有高度的可扩展性，并且在运行时不需要太多资源。

在本文中，我们将设置一个基于 FreeBSD 的 Prometheus 服务器，并通过 node_exporter 让客户端（Linux 和 FreeBSD）将系统指标发送到该服务器。我们还将使用 Grafana 可视化数据，并导入现有仪表板以实现这一目的。

## Prometheus 设置
首先，我们在 FreeBSD 系统上设置 Prometheus 实例。系统安装并连接到网络后，我们开始创建 Prometheus 存储数据的目录：`/var/db/prometheus`。该目录由端口自动创建，因此这一步并不是绝对必要的。然而，作为 ZFS 上的独立数据集并启用压缩等属性进行运行是一个好习惯。

```sh
# zfs create -o compression=zstd sys/var/db/prometheus
# pkg install prometheus node_exporter
```

为了从主机提取系统级别的指标，我们将在 Prometheus 主机和所有其他我们想要监控的机器上安装 node_exporter。Port Prometheus  会在 `/usr/local/etc` 路径下安装一个名为 `prometheus.yml` 的默认配置文件。我们将修改该文件以满足我们的需求。配置文件的语法采用 YAML 格式，因此需要特别小心，避免使用制表符，并使用适当的空格进行缩进。

```yml
prometheus.yml:
# 我的全局配置
global:
 scrape_interval: 15s # 将抓取间隔设为每 15 秒一次。默认为每 1 分钟一次。
 evaluation_interval: 15s # 每 15 秒评估一次规则。默认值为每 1
minute.
# scrape_timeout 设为全局默认值（10 秒）。
# 一个包含一个要抓取的端点的抓取配置：
# 这里是 Prometheus 本身。
scrape_configs:
 # 作业名称作为标签 `job=<job_name>` 被添加到从该配置抓取的任何时间序列中。
 - job_name: "prometheus"
 static_configs:
 - targets:
 - mistwood:9090
 - job_name: bdc
 static_configs:
 - targets:
 - mistwood:9100
 # metrics_path defaults to '/metrics'
 # scheme defaults to 'http'
```

在深入配置之前，我们启用 Prometheus 服务和 node_exporter，以便在 FreeBSD 启动时自动启动。

```
# service prometheus enable
# service node_exporter enable
```

Prometheus 提供了一个基于 Web 的界面，用于查询和显示由系统（在 Prometheus 术语中称为目标）导出的度量数据。我为 Prometheus 服务的启动提供了一个额外的参数，以定义 Web 界面应该在哪个端口上可访问，我的主机名是 mistwood。

```
# sysrc prometheus_args="--web.listen-address=mistwood:9090"
```

配置完成后，我们可以像这样启动 Prometheus 服务和 node_exporter：

```
# service prometheus start
# service node_exporter start
```

几秒钟后，执行以下命令：

```
# sockstat -l
```

输出应包含以下行，确认两个服务已成功启动：

| 用户地址        |   命令      |       PID     |  FD  | 协议 |   本地地址 |    远程地址   |  
|------------|-------------|--------------|-------|-----|------|---------------|
| prometheus | prometheus  | 70027        | 8     | tcp4 | mistwood:9090 | `*:*`      |
| nobody     | node_exporter | 2950       | 3     | tcp46 | `*:9100`         | `*:*`      |

首先，让我们检查 node_exporter 是否正在提取一些指标。我们可以在浏览器中访问运行 node_exporter 服务的主机（在我的情况下是 mistwood），并在 URL 末尾添加端口 9100 和 /metrics，形成以下链接：  **http://mistwood:9100/metrics**


![](https://github.com/user-attachments/assets/0d77ca60-52bb-4dbf-952e-135285312cde)

你会看到一系列以下划线分隔的已导出指标。每 10 秒刷新此页面，你会看到 node_exporter 收集的最新数据。既然已经在浏览器中，我们还可以检查 Prometheus 的状态。URL 形式类似，但端口号为 9090（末尾没有 /metrics），用于访问 Prometheus 的 Web 界面。进入 **Status** 下拉菜单，选择 **Targets**，即可查看 prometheus.yml 配置的所有主机。在本示例中，我提取了一些针对我们大数据集群（bdc）的配置片段。当然，你可以使用自己的标签，而“mistwood”这台主机名称也可以替换为其他名称。

![](https://github.com/user-attachments/assets/7bf0b685-f11e-4f1b-a3ab-64a0bbb47ad5)

Prometheus 会检查主机是否可达。我们在 prometheus.yml 文件中添加的 node_exporter（端口 9100）也会列在这里，并可以通过点击 URL 直接访问。Prometheus 还会检查主机的可用性，在 **State** 列中用 **UP** 来表示该端点处于正常状态。  

可以为某个主机组分配标签，以便对其进行逻辑分组。所有收集到的主机指标都会带有该标签，并且可以在后续使用 PromQL 语言或直接在 Grafana 中进行筛选。我在这里的作业名称（job name）是 **bdc**，所有属于该组的机器都会列在 **targets** 下。（这里我做了简化，仅列出了一个 FreeBSD 主机和一个 Linux 主机。）  

在使用 Grafana 进行可视化之前，我们可以先在 Prometheus 的 Web 界面上创建简单的图表。进入 **Graph** 选项卡，在顶部可以看到一个输入框。在输入框左侧，有一个蓝色的 **Execute** 按钮，旁边还有一个较小的 **metrics explorer** 按钮。点击它，会打开一个搜索框，列出所有已收集的指标名称。选择你想查看的指标后，点击 **Graph** 选项卡（位于 **Table** 旁边），即可看到该指标在所有主机上的可视化图表。  

你可以拖动并选择图表的一部分来放大该时间范围，也可以使用上方的控制按钮进行缩小。尽管这种方式已经能提供快速概览，但相比完整的仪表盘，它的视觉效果可能不够直观。因此，我们引入 **Grafana**，它提供了更多功能，并支持多种不同形式的数据展示。  

在撰写本文时，**Grafana 8** 是当前版本。旧版本同样可以正常工作，因此不必总是追逐最新版本来获取美观的图表。

```sh
# pkg install grafana8
```

与之前相同，我们启用 **Grafana** 服务，使其在开机时自动运行，并立即启动它，使用以下两行命令：

```sh
# service grafana enable
# service grafana start
```

**Grafana** 在 **/usr/local/etc** 目录下有一个配置文件，但这里我们不需要修改它。请确保访问并阅读 **Grafana** 官网的文档，以便根据你的环境调整该文件。  

等待 **Grafana** 启动片刻（可以通过 **sockstat** 命令检查 **Grafana** 是否在默认端口 **3000** 上监听）。然后，在安装 **Grafana** 的主机上，使用端口 **3000** 访问 **Grafana** 的登录页面。


![](https://github.com/user-attachments/assets/41b2a387-f3e3-41a0-90d6-8ff8534f3d59)

在全新安装的 **Grafana** 中，默认用户为 **admin**，密码也是 **admin**，首次登录后需要立即更改为其他密码。你还可以添加更多具有不同权限的用户，以便仅查看特定的仪表板。  

但目前，我们首先需要连接到 **Prometheus** 以获取监控数据。这可以通过左侧的 **齿轮图标** 进入 **Configuration -> Data Sources** 来完成。点击左侧蓝色的 **“Add data source”** 按钮即可添加数据源。



![](https://github.com/user-attachments/assets/e4b8576a-94d1-47e4-8441-37f6bc7534d9)


我们为数据源指定一个具有描述性的名称，并提供之前用于访问 **Prometheus** Web UI 的 **URL**（端口 **9090**）。在页面底部，点击 **“Save & Test”** 按钮，检查 **Grafana** 是否能够访问数据源。  

**注意：** 这里提供的配置是最基本的，重点是功能实现，而非安全性。在生产环境中，你必须对度量数据进行身份验证和加密，以防止攻击者通过读取这些数据来推测你的基础设施。**Prometheus** 和 **Grafana** 官方网站均提供相关的安全配置文档。  

### **导入 Grafana 仪表板**
现在，我们已经连接了数据源，接下来需要可视化数据。虽然可以自行设计仪表板，但并非每个人都有足够的艺术天赋。幸运的是，许多开发者已经创建了精美的仪表板，并在 **Grafana 官网** 公开共享：  

🔗 **[Grafana Dashboards](https://grafana.com/grafana/dashboards/)**  

在该页面左侧，你可以使用 **过滤器** 来筛选仅基于 **Prometheus** 数据源的仪表板。进一步在 **Collector Types** 下拉菜单中选择 **Node exporter**，右侧的搜索结果将根据筛选条件自动更新。  

点击某个搜索结果，可以查看 **预览** 及其详细信息。在页面 **右侧**，复制 **仪表板 ID**，然后切换回 **Grafana** 界面。  

### **导入仪表板**
1. 在 **Grafana** 左侧菜单栏中，转到 **“Dashboards”** 选项卡，选择 **“Manage”**。  
2. 点击左侧的 **“Import”** 按钮，进入仪表板导入页面。  
3. 在输入框中粘贴刚刚复制的 **仪表板 ID**，并加载仪表板。  
4. 选择之前创建的数据源，完成导入。  

### **推荐仪表板**
为了方便使用，以下是几个适用于 **Prometheus** 和 **Node exporter** 的仪表板（最后一个专门为 **FreeBSD** 设计）：  
- **1860**  
- **11074**  
- **4260**  

你可以在 **Dashboards** 选项卡中找到已导入的仪表板，并通过点击名称进入查看。一些仪表板支持 **筛选功能**，可选择特定 **主机**，或指定 **网卡**、**存储设备** 等不同指标。



![](https://github.com/user-attachments/assets/24372088-08e4-432d-a9e9-8f07359440a5)

有些仪表板可能会一次性显示大量数据，让人感到不知所措。我个人认为，先从一个概览仪表板开始，查看所有机器的整体情况是一个不错的选择。待发现异常情况，我会使用另一个更详细的仪表板深入分析特定的主机。特别是 **Prometheus** 提供的长期趋势数据，可以帮助我理解某些内存使用的峰值是否在预期范围内，或者是否属于正常现象。  

对于每台需要监控的主机，安装并启动 **node_exporter**，然后在 **Prometheus** 服务器上，将该主机的 **URL** 添加到 **prometheus.yml** 文件的 **static_configs** 目标列表中，最后重启 **Prometheus** 服务。这一过程相当直观，并且可以通过 **Ansible** 等配置管理工具轻松自动化大规模主机的部署。  

此外，**FreeBSD** 还有许多可用的 **node_exporters**，你可以尝试不同的导出器，并寻找一个适合自己监控需求的仪表板（或者自行创建）。我发现 **Prometheus** 能够提供比我以前的监控方案更多的指标，并且它支持 **告警功能**。在 **FreeBSD** 上，还有相关的软件包可用于实现告警功能，但这部分就留给你作为学习练习吧。  

**Prometheus** 具有清晰的架构，易于设置，并可以随时扩展以监控更多主机。是时候“盗取众神的火焰”，深入探索你服务器和服务的运行状况了！  

---  

**Benedict Reuschling** 是 **FreeBSD** 项目的文档贡献者，并担任 **文档工程团队** 成员。他曾两次当选 **FreeBSD 核心团队** 成员。目前，他在德国达姆施塔特应用科技大学（University of Applied Sciences, Darmstadt）管理一个 **大数据集群**，并为本科生教授 **“Unix for Developers”** 课程。此外，Benedict 还是每周播出的 **bsdnow.tv** 播客的主持人之一。
