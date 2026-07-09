# 面向 FreeBSD 用户的 Docker 入门

作者：Kurt Lidl

Docker（来自 Docker Inc.）是一款流行的容器化软件系统，用于构建、部署和运行 Linux 应用。Docker 容器提供了一种开销相对较低的机制，可运行多个相互隔离的 Linux 应用。Docker 提供了高层接口，用于配置、构建、存储和获取 Docker 镜像。本文简要回顾流行的虚拟化技术，给出一个用 Docker 设施构建容器的示例，并简要讨论 Docker 未来的演进方向。

## 虚拟化技术概览

当今使用的操作系统中存在许多不同的虚拟化技术。最基本的虚拟化是软件进程的概念。这是 Unix 及许多其他操作系统早期就向不同进程提供的传统虚拟化：每个用户进程通过内核的虚拟内存系统拥有独立的、受保护的内存映射。其他更全面的虚拟化技术，如软件虚拟化、基于 hypervisor 的虚拟化、带硬件加速的 hypervisor 虚拟化以及容器化应用，下面将分别回顾。

### 软件 virtualization/模拟

基于软件的完整机器模拟器（如 QEMU 和 SIMH）几乎可以在宿主机上模拟任何 CPU 和机器架构。这类模拟器通常相当慢，但在被模拟硬件和宿主机之间提供完全的独立性。模拟软件逐条指令地模拟目标机器，并提供目标机器硬件设备的软件实现。例如，目标机器上的磁盘驱动器常常用宿主机上的普通文件来模拟。即便是已无运行硬件的机器（如 Honeywell DPS8M）也能被模拟。在这种情况下，模拟的保真度足以让具有历史意义的 Multics 操作系统在模拟机器上无任何软件改动地运行。这类模拟器的另一个重要例子是 Connectix Virtual PC 软件，它能在基于 PowerPC 的 Mac 计算机上模拟一台完整的 x86 计算机。Connectix 公司被微软收购后，该软件已不再可用。

### 基于 hypervisor 的虚拟化

在虚拟化谱系的另一端是基于 hypervisor 的实现。基于 hypervisor 的虚拟化通常能以宿主机原生速度的可观比例运行。只有少量由 hypervisor 介入的系统功能在 hypervisor 中执行，其余用户代码在虚拟机中以原生速度运行。这类虚拟化被认为相当”重型”，因为每台虚拟机都有自己运行的操作系统副本。这种方案的一个性能问题领域是：虚拟化操作系统也必须维护自己使用的内存保护。对此类操作的硬件支持（有时称为”嵌套页表”）大大增强了 hypervisor 下客户操作系统的运行。可用的基于 hypervisor 的虚拟化平台有很多，包括：

- FreeBSD 上的 bhyve
- Linux 上的 KVM
- MacOS X 上的 xhyve
- Microsoft Windows 上的 Hyper-V
- VMware 的 ESXi 和 vSphere
- 多种操作系统和几种硬件架构上的 Xen

### 容器化虚拟化

容器是轻量级虚拟化方案，其中进程在同一主机上的不同管理组之间有某种分区和隔离。不同分区都共享单一内核应用二进制接口（ABI），运行在单一内核实例之上。通常（但不总是）能在宿主机上看到容器中运行的每个进程。这种虚拟化通常称为”容器计算”，在 hypervisor 提供的隔离级别和标准 Unix 环境”共享一切”之间提供折中。容器化是以下能力背后的根本思想：

- FreeBSD 上的 Jail 系统
- Linux 上的 Control Groups
- Nexenta OS 上的 Containers
- Solaris 上的 Containers

### 混合虚拟化技术

还有其他混合虚拟化技术，比如先运行 hypervisor 虚拟机，再在这些虚拟机上托管各种容器化应用。这种混合方式就是 Docker 在非 Linux 机器（如 MacOS 版 Docker）上的实现方式——它建立在 MacOS 的 xhyve 虚拟机之上。类似地，Windows 版 Docker 使用 Hyper-V hypervisor 创建运行 Linux 的虚拟机，再用它执行 Docker 容器的系统调用。

## Linux Control Groups 与 Docker

Linux 内核有一项相对较新的能力使 Docker 成为可能：Control Groups（又称”cgroups”）。这是允许将一组用户进程与另一组隔离、不让其相互影响或直接交互的根本技术。在传统 Unix 环境中，用户进程呈单一层次结构。init 进程（pid 1）是这棵树的根，所有进程都能追溯祖先到这个初始进程。Linux 内核中的 cgroups 设施允许实例化全新的进程层次结构，这些进程完全包含在新层次中，只能与同一层次中的其他进程交互。

cgroups 设施不仅能创建新的进程层次，还能设置资源限制（如内存和网络带宽），并将这些限制附加到所创建的进程层次上。虽然通过提供的系统工具管理底层 cgroups 机制是可能的，但相当繁琐。Docker 提供了更便捷的接口，在运行时控制 cgroups 机制，并配有一套易用的、用于构建之后将执行的静态环境的系统。Docker 使用 cgroups，配合 iptables（用于网络配置和控制）和 union 文件系统（UnionFS，用于将容器与宿主机文件系统隔离）等其他 Linux 内核设施。还有一种机制允许在宿主机和 Docker 容器之间显式共享目录。Docker 实现的 union 文件系统层叠在 Docker 存储驱动之上。可用的存储驱动取决于运行 Docker 的具体 Linux 系统，提供不同程度的性能和稳定性。

## Docker 术语与软件架构

镜像（image）是 Docker 对已创建并加载了特定应用所需软件层的容器化文件系统的称呼。当镜像需要运行时，会创建该镜像的写时复制快照，这个写时复制文件系统就称为容器（container）。启动容器的进程随后被放入新的 cgroup 层次。Docker 容器中初始进程派生的任何新进程都无法影响该运行容器之外的任何进程，因为所有其他进程都属于不同的 cgroups。这种隔离防止同一物理主机上运行的两个或更多 Docker 容器之间发生任何交互或干扰。

现代 Docker 安装通常至少有两个长驻守护进程：`dockerd` 和 `docker-containerd`。用户命令只有一个 `docker`，但接受多个命令关键字。这类似于许多复杂系统通过单一调度命令控制（如 `git`、`hg`、`rndc`）。`docker` 命令通过 Unix 域 socket 与 `dockerd` 进程通信。`dockerd` 进程再与 `docker-containerd` 进程通信，以指定系统上容器的管理。还有为每个容器启动的其他容器软件 shim。runC 容器运行时初始化并启动容器，然后把 stdin/stdout/stderr 的文件描述符交给 containerd-shim，后者充当运行中的容器与 docker-containerd 之间的代理。这个中间进程是必要的，这样 `dockerd` 重启（从而 `docker-containerd` 重启）后，新的 `docker-containerd` 守护进程能重新附接到每个运行中容器的 containerd-shim。

## Docker 镜像解析

Docker 镜像是虚拟文件系统，被打包为一系列层。文件系统中的每一层堆叠在它下面各层之上。文件系统的最终视图是构成 Docker 镜像的各文件系统的并集。镜像中的层由 Dockerfile 中的命令构建而成。

### Dockerfile 即配方

Dockerfile 是一个简单的文本文件，包含一条或多条命令，以及用户在文件中放置的注释。Docker 构建镜像时，按遇到的顺序执行文件中的每条命令。从这个意义上说，`docker build` 过程就像按部就班的菜谱做饭。Dockerfile 中的每条命令都会在最终镜像中生成一个新层。按惯例，Docker 命令以大写字母书写，以便与用户指定的命令区分。如果执行的任何命令失败（即返回非零退出码），镜像构建立即停止，构建被标记为失败。出于效率考虑，最好让 Dockerfile 中的每一层尽可能小。这意味着，清理任何会产生大量元数据的命令（如 `yum update`）应在生成元数据的同一条命令中完成。下面的示例 Dockerfile 会创建一个有六层的镜像。其中与上一层完全相同的层会在构建过程中自动丢弃。

下面是一个运行 Apache 的 Dockerfile 示例：

```dockerfile
# 用于运行 Apache 的镜像
FROM centos:7

MAINTAINER Ms. Nobody <nobody@example.com>

RUN yum -y --setopt=tsflags=nodocs update && \
    yum -y --setopt=tsflags=nodocs install httpd && \
    yum clean all

VOLUME ["/var/www/html", "/var/log/httpd"]

EXPOSE 80

CMD ["/usr/sbin/apachectl", "-DFOREGROUND"]
```

第一条命令 `FROM centos:7` 创建镜像的第一层，指定从中央 Docker 仓库把 CentOS 7 的基础镜像拉到本地机的文件系统层缓存。这一层是镜像的底层。`FROM` 命令必须是 Dockerfile 中的第一条命令，并初始化新的构建。

第二条命令 `MAINTAINER ...` 在镜像的元数据中设置一个特殊标签，用于标识镜像创建者。还有一条 `LABEL` 命令可以替代使用，为镜像设置任意数量的标签。标签可由最终用户用于任何目的。

第三条命令 `RUN yum -y update ...` 更新基础镜像中过时的软件包。下一条命令 `yum -y install httpd` 安装 Apache 的 `httpd` 包。命令最后部分 `yum clean` 清除 yum 包管理系统维护的所有包/仓库元数据，以尽量缩小生成层的大小。同理，yum 命令通过 `nodocs` 标志，在升级和安装包时被指示忽略文档。`RUN` 命令的参数由 shell 进程执行，因此构建镜像中所生成层的复杂程度可以相当高。

第四条命令 `VOLUME [ ... ]` 标记一组在 UnionFS 文件系统中用作外部挂载点的目录。容器执行期间，这些挂载点会挂载外部文件系统。UnionFS 层不应尝试把这种活动捕获到写时复制文件系统。这是容器在执行它的写时复制镜像之外持久化数据的方法之一。

第五条命令 `EXPOSE 80` 提供的信息会在容器从该镜像启动时使用。运行时可以把宿主机的一个端口映射到容器指定的 TCP 端口号。或者，在运行时指定不同的网络选项，容器的端口可让同一主机上运行的其他容器访问。

最后第六条命令 `CMD ...` 指定从该镜像启动容器时执行的默认命令。本例中，它在前台启动 Apache Web 服务器。当 Apache 进程退出时，容器会自动停止。在 Docker 容器内启动的守护进程周围常常写一个小包装脚本来重启守护进程，避免它停止运行。通过自动重启守护进程，Docker 容器可以持续运行而无需重启。

既然每行的用途都清楚了，构建镜像就很简单。注意，构建过程的部分输出已被删除，并换行以提高可读性。镜像通过运行命令 `docker build` 目录创建，其中”目录”是包含 Dockerfile 的目录路径（输出如下）：

```sh
# docker build -t centos-apache-testimage .
Sending build context to Docker daemon  2.048kB
Step 1/6 : FROM centos:7
 ---> 3bee3060bfc8
Step 2/6 : MAINTAINER Ms. Nobody <nobody@example.com>
 ---> Using cache
 ---> 7f88dbad6a42
Step 3/6 : RUN yum -y --setopt=tsflags=nodocs update && \
    yum -y --setopt=tsflags=nodocs install httpd && \
    yum clean all
 ---> Using cache
 ---> f50595808f75
Step 4/6 : VOLUME ["/var/www/html", "/var/log/httpd"]
 ---> Running in bce2b6331fc8
 ---> 51b4c07c8eba
Removing intermediate container bce2b6331fc8
Step 5/6 : EXPOSE 80
 ---> Running in 073e6fac8709
 ---> 5bf7cadf8102
Removing intermediate container 073e6fac8709
Step 6/6 : CMD /usr/sbin/apachectl -DFOREGROUND
 ---> Running in e5a44065f0d7
 ---> 4d119d3a4776
Removing intermediate container e5a44065f0d7
Successfully built 4d119d3a4776
Successfully tagged centos-apache-testimage:latest
```

### Docker 镜像检视

通过 `docker inspect` 命令查看该镜像的一些元数据很有启发：

```sh
# docker inspect centos-apache-testimage:latest
[
    {
        "Id": "sha256:db9314a42feb [...]",
        "RepoTags": [
            "centos-apache-testimage:latest"
        ],
        "ContainerConfig": {
            "ExposedPorts": {
                "80/tcp": {}
            },
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin: [ ... ]"
            ],
            "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) ",
                "CMD [\"/usr/sbin/apachectl\" \"-DFOREGROUND\"]"
            ],
            "Volumes": {
                "["/var/www/html",": {},
                ""/var/log/httpd"]": {}
            }
        },
        "DockerVersion": "17.06.0-ce",
        "Author": "Ms. Nobody <nobody@example.com>",
        "Architecture": "amd64",
        "Os": "linux",
        "Size": 275797466,
        "GraphDriver": {
            "Data": null,
            "Name": "aufs"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:dc1e2dcd [...]",
                "sha256:41fc3fb9 [...]"
            ]
        }
    }
]
```

上面输出中未展示全部元数据，只挑了一些比较有趣的部分。`ContainerConfig` 部分包含从该镜像启动的任何容器中进程的完整环境。`Architecture` 和 `Os` 设置表明容器支持 amd64（又称 x86_64）机型下的 Linux syscall 接口。本文用的 Docker 镜像在一台运行 macOS Sierra 10.12.5 的 Macintosh 计算机上创建，但任何容器都将在 Linux/amd64 运行时环境中执行。这个镜像可被移到任何能运行 Linux/amd64 Docker 镜像的主机上。镜像的可移植性是 Docker 的主要优势之一——可轻松跨多台主机构建和部署，无需担心共享库冲突或破坏已安装组件的配置。Docker 支持镜像仓库（registry），可在其中存储和检索镜像。可创建私有 Docker 仓库让用户集中存储自定义镜像。镜像存入仓库后，一条命令即可拉到主机，第二条命令即可从镜像启动容器。

## 运行容器

从示例镜像创建运行中的容器很简单：

```sh
# docker run --rm -d -p 8080:80 \
    -v $(pwd)/htdocs:/var/www/html \
    -v $(pwd)/logs:/var/log/httpd \
    centos-apache-testimage:latest
```

这条命令启动容器，告诉 Docker 在容器退出时丢弃写时复制文件系统（`--rm`）。在后台运行容器（`-d`），并把 localhost:8080 端口映射到容器的 TCP 80 端口（`-p 8080:80`）。把 `$(pwd)/htdocs` 挂载到 Web 服务器的 DocumentRoot（`-v $(pwd)/htdocs:/var/www/html`），并把 `$(pwd)/logs` 挂载为容器内 Web 服务器日志文件的存放位置（`-v $(pwd)/logs:/var/log/httpd`）。最后给出用作容器初始文件系统的镜像名。

容器一旦启动，会一直运行到启动容器的初始进程退出。启动容器的用户或系统管理员可以通过 `docker stop` 命令停止容器。它向容器中的初始进程发送 SIGTERM，若容器未停止，则在宽限期后发送 SIGKILL。其意图与在 Unix 主机上运行 shutdown 命令非常相似。`docker kill` 命令只是向容器中的根进程发送 SIGKILL。`docker ps` 命令列出运行中的容器。`docker rm` 命令可删除已停止容器的写时复制文件系统。

## 其他 Docker 命令

Docker 还有其他命令，可用于自动启动容器，以及维护一组必须共同运行以完成给定任务的容器。全面研究 Docker 内所有可用命令超出本文范围。通过近期 Docker 版本中的”Docker Swarm”支持，可以跨多台主机编排运行多个容器，进行更高级的多容器协调。

## 与 FreeBSD Jail 的比较

Docker 容器在所提供的虚拟化和机器资源共享方式上，与 FreeBSD Jail 相似。两者都在宿主机上提供针对单一内核镜像运行的隔离进程。Docker 提供易用的命令行界面，用于创建、部署、运行和更新镜像。除基本的 Docker Engine 安装外，宿主机几乎不需要额外设置和配置。FreeBSD Jail 在运行和停止 Jail 方面的接口要简单得多。基本 FreeBSD 系统对把 Jail 构建和安装到目录中几乎没有提供高层支持。有几个附加 FreeBSD Port（如 ezjail、qjail、qjail4）试图让 Jail 使用不再那么繁琐，成功程度各异。Docker 相比 Jail 的一个显著优势是：为每个启动的容器派生一份写时复制的文件系统实例。这是 Docker 镜像部署和复用的根本，而每个 FreeBSD Jail 通常运行在持久化的文件系统树中。一些附加 Jail 管理系统使用 ZFS 的 snapshot 和 promote 功能为新创建的 Jail 克隆原型文件系统，但这种克隆在 Jail 多次重启间仍持久化文件层次。

Docker 容器启动时，Docker 基础设施会负责挂载各种目录，而在 FreeBSD Jail 中，任何目录的挂载（哪怕是关键的 devfs **/dev** 挂载）都必须为每个 Jail 显式处理。比如运行 Apache 的 docker 容器示例，有几个活跃挂载点：

```sh
# df
Filesystem  1K-blocks  Used  Available  Use%  Mounted on
none                          2%  /
tmpfs                        0%  /dev
tmpfs                        0%  /sys/fs/cgroup
/dev/sda2                     2%  /etc/hosts
shm                           0%  /dev/shm
osxfs                        64%  /var/www/html
tmpfs                        0%  /sys/firmware
```

Docker 系统自动完成了所需挂载，只有 `osxfs` 挂载的 `/var/www/html` 例外——这是容器启动时在命令行上指定的。

Docker 与 Jail 在安全方面的收益大致相当。Docker 的安全特性通常通过命令行标志修改，而 Jail 的安全特性要么通过 sysctl 设置全局指定，要么通过 **/etc/jail.conf** 文件为每个 Jail 单独配置。当 Jail 使用 VIMAGE 网络选项运行时，每个 Jail 有独立的网络栈。这意味着每个 Jail 可以有不同的包过滤配置。基于 Linux 的宿主机上所有运行中的容器共享单一基于 iptables 的包过滤配置。

Docker 与 Jail 在控制方面差异很大。Docker 有许多命令和选项，几乎可以让所有配置都在命令行完成。FreeBSD Jail 主要依赖 **/etc/jail.conf** 文件内容指定运行哪些 Jail 以及如何配置。Docker 把大量配置内化为给定 Docker 镜像的元数据。把元数据附加到镜像上，部署到新主机就显著简化。FreeBSD Jail 没有这种直接附加到每个 Jail 的元数据。

在相关领域，一些用于帮助管理 bhyve 虚拟化主机实例的 FreeBSD Port（如 iohyve）提供类似配置辅助。这些系统用 ZFS 属性把虚拟机元数据附加到代表虚拟机文件系统的 ZFS 文件系统或 ZFS zvol 上。这些管理工具的最早版本仅使用众所周知的名称来控制参数：主机名、CPU 数、内存大小等。没有一个提供通用的、可扩展的 tag:value 配置文件来附加到 ZFS 文件系统——尽管自最早尝试以这种方式支持虚拟机元数据以来，情况可能已经变化。

## Docker 的未来方向

Docker 系统正在相当快速地演进。一个由公司组成的联盟成立了 Open Container Initiative（OCI）。OCI 的重要成员包括 Amazon、AT&T、Cisco、Docker Inc.、Facebook、谷歌、IBM、Microsoft、Oracle、RedHat 和 VMware。OCI 试图标准化容器运行时系统（“runtime-spec”）和镜像规范（“image-spec”）。作为标准化过程的起点，Docker Inc. 把自己的容器规范和运行时系统（“runC”）捐给了 OCI。OCI 中有些成员在支持不止 Linux syscall ABI 容器上有既得利益，规范明确指出需要支持多种 ABI 以及多种托管容器运行时的操作系统。标准化过程中的一个近期进展是 Oracle 发布了用 Rust 编写的 oci-runtime 开源实现 Railcar。

OCI 中的多 ABI 支持让使用 Docker 部署不再是 Linux/amd64 的单一文化。目前 Docker 实际上只在 amd64 硬件上运行 Linux ABI。Docker 社区通过 OCI 已初步同意一个多架构系统，其中 Linux 和 Windows 都将作为一等公民 ABI 环境得到支持，并跨多种硬件平台。这种跨系统支持在 IBM 的 Z-System 硬件上对 Linux ABI 已有限度可用，对 arm64 架构的支持也初具雏形。把这种多 ABI 未来扩展到 FreeBSD 应当是可能的。

## Docker 在 FreeBSD 上？

审视当前 Docker 系统的现状，让 Docker 系统原生在 FreeBSD 上运行似乎不存在不可逾越的技术障碍。Docker 的未来和对容器跨不同 ABI 的支持意味着，为容器支持原生 FreeBSD 内核 ABI 是可能的。显然，这一未来会让使用 Docker 部署不再是 Linux/amd64 的单一文化。

KURT LIDL 是 Oracle 的首席技术成员，在 Oracle 公有云构建团队工作。他从 4.2BSD 开始使用 BSD Unix，现为 FreeBSD 项目的 Committer。他与妻子和两个孩子住在美国马里兰州波托马克。
