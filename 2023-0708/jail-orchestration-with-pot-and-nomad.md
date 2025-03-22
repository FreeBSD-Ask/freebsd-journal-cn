# 使用 pot 和 nomad 管理 Jail

- 原链接：<https://freebsdfoundation.org/wp-content/uploads/2023/08/Pizzamiglio.pdf>
- 作者：LUCA PIZZAMIGLIO
- 译者：段龙甫
  

在许多服务器上，尤其是分发水平可扩展应用方面，容器是个卓越的工具。当应用数量和它们的基数增长时，手工管理数量众多的容器将会变得困难起来。

容器管理器是一个旨在简化众多容器管理，遮蔽复杂性，提高可靠性的应用程序，特别是在一个自动伸缩和持续部署的动态环境中。在本文中，我们将讨论基于 FreeBSD，使用 pot(一个支持 jail 镜像的 jail 框架)，和 nomad(一个由 HashiCorp 开发的与容器无关的管理器)。

## 系统架构

为了解释管理器是如何工作的，我们需要介绍一些服务并说明他们的角色。

### nomad 客户端

nomad 客户端就是一个服务器，接收来自管理器的命令去执行容器。Nomad 客户端也被称为节点，就像 kubernetes 一样。

在大型设备中，大多数服务是 nomad 客户端，他们负责执行用户的应用程序。在云本地术语中，nomad 客户端在集群中构成数据平面。

Nomad 客户端能支持多种容器驱动：一些驱动与操作系统无关，当其它驱动程序，像 docker 或 pot，仅在特定操作系统可用。

为了使用 pot 管理 FreeBSD 上的 Jail，我们需要基于 FreeBSD 的 nomad 客户端。

### nomad 服务器

nomad 服务器是实现管理器的机器。nomad 服务器负责保持集群状态和调度容器到 nomad 客户端。需要多实例（3 到 5 个）提供冗余并共享负载。在云本地术语中，nomad 服务器在集群中构成控制平面。

用户与 nomad 服务器交互，部署应用到集群。nomad 服务器负责保持集群的正常状态，并在客户端出现错误时重新调度容器。

Nomad 服务器能运行在任何提供支持的操作系统。要管理 jail，至少有一个 nomad 客户端是基于 FreeBSD 的。

### 容器注册表

管理器（nomad 服务器）是集群的大脑，分配容器给支持容器类型的客户端。这意味着：

- 任何 nomad 客户端能选择运行支持的容器。

- 客户端事先不知道容器在哪个主机。

- 单个客户端能执行同一个容器的多个实例。

每个客户端都需要一个服务来执行下载容器镜像。这个需求在容器注册中心来满足，提供容器镜像的服务。

当一个管理器选择一个客户端来执行一个容器，该客户端将从注册中心下载容器镜像，然后启动容器。

在我们的例子中，我们将使用 potluck，一个由 pot 社区维护的公共容器注册表，从开放源代码目录的镜像库创建镜像。但是，由于容器镜像包含二进制文件，出于安全考虑，我们强烈建议每个人拥有自己的本地注册表。

在 pot 容器中，镜像是通过 fetch(1) 下载的文件，因此注册表可以是一个简单的 web 服务器。


### 服务目录

服务目录是一个被附加的信息增强的服务列表，如实现那些服务所有的容器地址。

当一个管理器调度容器满足服务，它也会将容器地址注册到实现该服务的容器列表中。

也能将服务目录配置为定期检查所有容器的服务运行状态，因此容器地址列表只包括健康的地址。

我们要使用的服务目录基于 consul，一个服务网格应用，同样也是 HashiCorp 开发的。

### 入口（可选）

因为管理器的动态特性，它很难获知服务在哪里运行。每次发生新的部署时，容器都被调度到不同端口上的不同节点。通过入口，我们定义一个代理/负载均衡器，配置一个固定的入口点给我们的服务。

对于一个实例，我们可以这样配置一个代理，网络地址（例如，`https://example.com/foo`）提供关于目标服务的信息（即重定向到实现服务 foo 的容器），另一种常见方法是提供主机头部信息。

入口代理通过不断地与服务目录交互，动态地维护有效容器地址列表。

对于我们的例子，我们使用 traefik（反向代理、负载均衡工具），一个 traefix 实验室开发的入口代理。

### Nomad-pot-driver

Nomad 的构建支持多个容器的技术和不同于操作系统。事实上，nomad 软件包是可用的，HashiCorp 公司也为 FreeBSD 提供二进制包。

Nomad 有一个插件结构允许它扩展支持新的容器技术。Esteban Barrios 编写并开源了 [Nomad-pot-driver 插件](https://github.com/bsdpot/nomad-pot-driver/)。这个插件作为 nomad 客户端和 pot 容器之间的接口，提供管理 Jail 所需的特性。

管理器将工作负载调度到使用插件与 pot 交互的客户端。

图片：**体系结构概述和不同服务的文件。**

![R%(G0Z M53 6`KY9Y2 M(70](https://github.com/FreeBSD-Ask/freebsd-journal-cn/assets/10327999/21b66fef-1c8f-4172-908b-7d5c3efd6593)


### 精简的 pot 容器

Minipot 是一个在一台 FreeBSD 机器上安装和配置所有上述服务的包，也是我们使用它来展示例子的参考安装。

Minipot 对于测试是个很有用的配置，但不适用于专业安装，因为它将把所有服务集中在一台机器上，将集群减少到一个节点来完成所有工作。

特别是，它将安装和配置 consul、traefik 和 nomad。Nomad 将作为客户端和服务器运行，扮演管理者和执行者的双重角色。

在 [Klara 网站](https://klarasystems.com/articles/cluster-provisioning-with-nomad-and-pot-on-freebsd/)上有一篇关于如何安装 minipot 的详细指南。

## 调度任务

minipot 初始化后，并且所有服务正在运行，我们就可以使用下面的任务说明文件在 nomad 中启动一个任务。


```
 job “nginx-minipot” {
   datacenters = [“minipot”]
   type = “service”
   group “group1” {
     count = 1
     network {
       port “http” {}
     }

     task “www1” {
      driver = “pot”

      service {
        tags = [“nginx”, “www”]
        name = “hello-web”
        port = “http”

        check {
          type     = “tcp”
          name     = “tcp”
          interval = “5s”
          timeout  = “2s”
        }
     }

     config {
       image = “https://potluck.honeyguide.net/nginx-nomad”
       pot = “nginx-nomad-amd64-13_1”
       tag = “1.1.13”
       command = “nginx”
       args = [“-g”,”’daemon off;’”]

       port_map = {
         http = “80”
       }
     }

       resources {
         cpu = 200
         memory = 64
       }
     }
   }
 }
```

上述任务段落说明了 nomad 调度任务所需的所有细节。

任务“nginx-minipot” (第 1 行) 有一个组名“group1” (4),它有一个任务叫做“www1” (9)。

任务“www1”是一个 pot 容器 (10),注册镜像是 potluck(23),pot 镜像是 nginx(24)，版本是 1.1.13 (25)。规定任务是基于 pot 驱动，管理器将在 pot 支持的客户端调度这个任务。在我们的例子中，服务器和客户端都是相同的机器。

与通常使用的使用 rc 脚本引导的 Jail 相比，我们将直接执行 nginx(26),不需要任何额外的服务。变量参数 (27) 是重要的，允许 nomad 服务恰当地遵循容器生命周期并捕捉日志。Pot 将负责初始化网络和任何需要的事情。

port_map 段 (28) 和网络段 (6) 告诉 nginx 在 jail 中监听 80 端口，但 nomad 客户端将使用另一端口 (“http”),被 nomad 服务器动态指派。

服务段 (11) 提供 nomad 服务将注册服务到 consul 的信息。在我们的例子中，任务“www1”实现了服务“hello-web” (11)，它将使用 nomad 服务器分配的端口“http”(14) 注册到 consul。对于 IP 地址，nomad 服务器将使用在调度期间定义的 nomad 客户端 IP 地址。

在我们的例子中，任务还配置了一个 tcp 健康检查，consul 将每 5 秒钟运行一次该检查，以确定实例的健康状况。

这个任务说明需要被保存成一个文件（即 nginx.job），任何用户能通过服务运行。

```
$ nomad job nginx.job
```

注意：第一次部署需要一些时间，因为客户端需要下载镜像。在连接缓慢的情况下，第一次部署还可能因为超时而失败。下载完成后，可以安全地重新运行将在几秒钟内执行的部署。

### 检查 nomad

任务被调度后，就可以通过命令检查部署状态：

```
$ nomad jobs allocs nginx-minipot
ID       Node ID  Task Group Version Desired Status  Created    Modified
636d3241 c375b833 group1     3       run     running 28m43s ago 28m27s ago
$ nomad alloc status 636d3241
[...]
Allocation Addresses:
Label Dynamic Address
*http yes     2003:f1:c709:de00:faac:65ff:fe86:9458:22854
[...]
$ curl “[2003:f1:c709:de00:faac:65ff:fe86:9458]:22854”
```

“Allocation”是 nomad 用来标识容器（任务的实例）的名字。

IPv6 地址是 nomad 客户端地址。

22854 端口是 nomad 选择的端口，用于将 nomad 客户端引导到容器的 80 端口。

查看端口重定向设置，我们可以使用下面命令：

```
$ sudo pot show
```

作为命令行的另一种选择，nomad 服务器也被配置为提供一个强大的 web UI，可以通过 **localhost:4646** 访问，我们可以看到“nginx-minipot”任务，导航到有关的集群，配置，客户端等等所有信息。

通过 nomad，我们能直接看到所有容器的状态。

进入配置页面，点击“执行”按钮启动 shell 脚本（/bin/sh）进入运行的容器中。

### 检查 consul

通过命令行，我们可以看到 consul 目录中的服务清单（consul 目录服务），但没有详细。但是，我们能通过访问 web UI 来检查“hello-web”服务的状态：

```
localhost:8500
```

从这里，我们可以导航到检查“hello-web”服务和检查 tcp”的状态。

### 检查 traefik

代理 traefix 被配置到交换机的 8080 端口，提供 web-ui 在 9200 端口（**localhost:9200**）监控状态。Traefik 被配置为从 consul 同步服务目录。

通过选择了 http 服务，我们查看“hello-web”服务（标记为 **hello-web@consulcatalog**）。

点击服务，我们查看服务明细和路由选项。

配置基于主机头，在我们的例子中是“hello-web.minipot”。

现在通过入口到达服务 hello-web：

```
$ curl -H Host:hello-web.minipot http://127.0.0.1:8080
```

或者，我们可以添加条目

```
127.0.0.1 hello-web.minipot
```

到达 **/etc/hosts** 然后直接使用主机名：

```
$ curl http://hello-web.minipot:8080
```

我们将得到与直接从 jail 下载到的相同的输出。

### 水平扩展

要查看管理器的活动状态，我们现在简单地将任务文件（第 5 行）中的计数从 1 改成 2，并重新提交任务。

```
$ nomad run nginx.job
```

调度完成后，运行中的 nomad 配置是 2，consul 中的服务“hello-web”有两个实例，就像 traefik 中的服务器。

- 我们可以验证入口的循环分布

- 跟踪一个容器的日志 (**$ nomad alloc logs -f allocation1**)

- 跟踪其它容器的日志 (**$ nomad alloc logs -f allocation2**)

- 在入口执行 curl(**$ curl -H Host:hello-web.minipot http://127.0.0.1:8080**)

在每次执行 curl 时，代理都会在容器之间分发请求，这可以从容器的日志中看出来。

### 拆掉一切

为了停掉我们的例子，我们建议这样进行拆除操作：

- 停止 nomad 任务（**$ nomad stop nginx-minipot**）
- 停止 traefik（**$ sudo service traefik stop**）
- 停止 nomad（**$ sudo service nomad stop**）
- 停止 consul（**$ sudo service consul stop**）

### 从平台到生产

Minipot 是一个有用的单节点安装的作为学习或本地测试的平台。

生产环境应以不同方式部署：

- 3 或 5 个不同的 consul 服务器
- 3 或 5 个不同的 nomad 服务器
- 2 入口代理服务器 (HA 配置)

几个 nomad 客户端服务器（取决于预期的工作负载和可靠性需求，如过度供应因素）

值得一提的是，前面提到的设置可以混合不同的操作系统：唯一必须运行 FreeBSD 的服务器是针对 jail/pot 工作负载的 nomad 客户端。

Nomad 或 consul 服务器可以在 Linux 或 Solaris 上运行，允许你重用可能已经可用基础设施。


作为入口代理，我们使用 traefik，与本地 consul 同步。但是，可以使用其它服务，如 nginx 或 ha-proxy，以及 consul-template 来实现相同的结果。在这种配置中，consul-template 负责监测 consul 的变更，呈现代理配置模板，并将新配置通知代理。

此外，所有服务的配置都需要改进，比如向 nomad 增加身份验证。

## 鸣谢

我想强调实现这一点所需要的社区的努力，从第一个 nomad-pot-driver 开发人员 Esteban Barrios 到 Michael Gmelin（grembo@），他们的确为提高该解决方案的可靠性和稳定性提供了提供很多帮助。我还想提及 Stephan Lichtenauer 和 Bretton Vine，他们参与了 Potluck，公共镜像注册，以及许多其它项目，如 Ansible 剧本和[博客文章](https://honeyguide.eu/tags/pot)，致力于 pot 和 nomad 使用实例。

---

**LUCA PIZZAMIGLIO** 是 FreeBSD 项目的 port 提交者，还是 port 管理组成员。在 2017 年，他开始了 pot 项目，在 FreeBSD 上可以被理解为容器。
