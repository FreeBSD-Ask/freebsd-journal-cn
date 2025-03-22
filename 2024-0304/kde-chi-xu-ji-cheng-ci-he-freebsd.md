# FreeBSD 与 KDE 持续集成（CI）

- 原文链接：[KDE CI and FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/development-workflow-and-ci/kde-ci-and-freebsd/)
- 作者：Ben Cooksley


自 2011 年 8 月以降，KDE 就开始实施 CI（Continuous Integration，持续集成）系统，且持续改进着。自此，系统已经大幅演进，增加了对多个 Qt 版本（大多数 KDE 软件所使用的工具包）以及多个平台的支持。

得益于容器的普及，能在多个操作系统上可靠地运行所有这些构建。为了理解容器解决了哪些问题以及 CI 系统可扩展性中的挑战，我们需要回顾 KDE CI 的起点。

系统起初由简单的 Jenkins 配置——在同一服务器上进行构建。然而，随着更多项目接入，构建需求增加，需要更多的机器。

这带来了一个难题：KDE 软件的构建需要其他 KDE 库——通常是最新版本的。这意味着仅增加构建机器是不够的，还需要确保最新的依赖项始终可用。

由于构建应用程序所需的全部依赖项链时间较长，因此无法每次重新构建全部内容，这就需要共享构建产物。经过快速评估，选择了 rsync，它使系统运作良好。

### 引入 FreeBSD

到 2017 年，CI 系统需要支持新的平台，于是系统中加入了 FreeBSD。FreeBSD 的早期支持较为简单，在 Linux CI 工作节点上采用虚拟机运行。每台虚拟机单独配置，包含构建 KDE 软件所需的所有依赖项。

虽然这种方法确保了 KDE 软件在 FreeBSD 上的可靠构建，但系统的扩展性较差，因为需要逐台更新构建器。这一方法成功地保证了 KDE 软件在 FreeBSD 上的构建可靠性，也改善了 FreeBSD 上 KDE 团队的开发体验。

在添加 FreeBSD 支持的同时，我们还在 Linux 构建中引入了 Docker。能够首次创建一个可在所有构建器中分发的主配置，方便了 CI 系统的变更发布，标志着基于容器的构建时代的开始。唯一的缺点是 Docker 仅适用于 Linux，如何在其他平台上再现这一流程成为一个问题。

随着构建能力的扩展，出现了一些新问题。构建偶尔会随机失败，日志显示文件缺失和符号链接损坏，但随后检查发现文件存在，之后的构建可顺利完成。问题在于：原子性（Atomicity）。

在新硬件下，构建速度加快，增加了 rsync 在文件上传时，而另一个构建节点下载构建产物的概率。解决方案是使用 tar 包的构建产物，以原子操作发布完整的文件集。此方法与 SFTP 协议（用于不支持 rsync 的平台）结合，CI 系统恢复了稳定运行，资源和支持平台也有所增加。

但手动维护机器的问题仍然存在。迁移到 Gitlab 和 Gitlab CI 后，构建节点由于累积的代码检出和构建产物迅速耗尽磁盘空间，测试留下的进程也会占用 CPU。而 Linux 上的 Docker 构建不存在这些问题。

我们探讨了多种解决方案，如改进 Gitlab Runner 的“shell”执行器、清理构建产物的 cron 作业，以及基于 FreeBSD Jail 的解决方案，但均无法复现 Linux 上 Docker 的体验。

## 发现 Podman

某天早上，我们在研究 FreeBSD 的容器化选择时偶然发现了 Podman 及其搭档 ocijail。这正是我们在基于 Docker 的 Linux 设置中习惯的功能，但现在能在 FreeBSD 上实现了。

这意味着之前遇到的残留进程和需要手动清理的构建产物问题都能得以解决。此外，我们还可以利用标准的 OCI（Open Container Initiative，开放容器计划）注册表（例如 Gitlab 内置的容器注册表）来把 FreeBSD 镜像分发到所有构建器上，从而解决了单独维护每台机器的问题。

首要困难是构建一款能用的镜像。对于 Linux 系统来说，Docker 和 Podman 非常成熟，有详细的文档说明基础镜像及其包含内容。但在找到适合的 FreeBSD 基本镜像后，我们以为只需添加 FreeBSD 包仓库，安装所需软件包即可。

然而，在容器中首次构建时，CMake 报告无法找到编译器。我们认为这很奇怪，因为 FreeBSD 系统通常预装了编译器。经调查，我们发现 FreeBSD 容器与正常的 FreeBSD 系统的主要区别在于：容器经过大幅精简，默认未包含编译器。

经过数次迭代，我们添加了编译器和 C 库开发头文件，从而在 FreeBSD 容器中成功构建了首款 KDE 软件。虽然我们认为一切顺利，但后续构建依然失败，因为需要额外的开发包。经过多次迭代和安装更多 FreeBSD 软件包后，我们终于完成了多个关键 KDE 软件包的构建。

接下来我们将注意力转向 Gitlab Runner 的“辅助镜像”，用于执行 Git 操作和上传构建产物到 Gitlab。尽管可以在 FreeBSD 上运行 Linux 二进制文件，但我们希望能在 FreeBSD 上原生构建。仿照 Gitlab 的方法构建镜像后，顺利得到了预期的结果。

冒险之旅的乐趣部分开始了：深入 Gitlab Runner 和 Podman 的内部机制。首次将 Gitlab Runner 连接到 Podman 时，构建遇到报错“unsupported os type: freebsd”。

在 Gitlab Runner 代码库中发现，Docker 需要检查远程 Docker（或在我们的例子中是 Podman）主机的操作系统类型。我们对 Gitlab Runner 进行了补丁和重建，解决了此错误，但紧接着又出现了类似的报错：“unsupported OSType: freebsd”。进一步修补后，又遇到一个更严重的错误，尤其是使用 Go 语言编写 Gitlab Runner 时：

```go
ERROR: Job failed (system failure): prepare environment:
Error response from daemon: runtime error: invalid memory address or nil pointer
dereference(docker.go:624:0s.
Check https://docs.gitlab.com/runner/shells/index.html#shell-profile-loading for more
information.
```

显然，要使该功能正常工作需要进行更多修改，但由于容器化构建的潜力，我们继续研究该问题。最终，我们找到了导致失败的代码：

```sh
inspect, err := e.client.ContainerInspect(e.Context, resp.ID)
```

我们发现 Podman 的守护进程崩溃，从而中断了请求。这个问题可以通过尝试运行 `podman inspect` 进行复现。原因归结为专门用于“inspect”操作的代码调用了 Linux 专用的构造。经过又一次补丁后，我们的 `podman inspect` 不再崩溃，终于成功启动了第一次 FreeBSD 构建。

## 在 FreeBSD 上运行构建

首次构建仍然失败（由于 Gitlab Runner 与非 root 用户的容器交互的已知问题），但我们在 FreeBSD 上已成功运行了构建。

你可能以为此时我们可以为所有 KDE 项目推行基于 FreeBSD 的容器化构建了。然而，最终测试发现 FreeBSD 容器的网络速度远低于预期，与 FreeBSD 主机相比明显较慢。

幸运的是，这个问题不是新问题。我们预计会遇到此问题，原因是大型接收卸载（LRO）。简单地更改配置后，最终我们达到了预期的性能，可以投入生产。

今天，KDE 使用 Podman 和基于 ocijail 的容器来运行 FreeBSD CI 构建，有 5 台 FreeBSD 主机处理构建请求。构建中使用了两台不同的 CI 镜像——分别适用于 Qt 5 和 Qt 6 的版本，确保 KDE 软件可以从零开始构建，并可选地通过所有单元测试。

自从从 FreeBSD 专用虚拟机迁移到 FreeBSD 容器化构建后，我们从每周甚至每日对构建器进行维护，逐渐减少到每隔几周维护一次，几乎没有收到开发人员的投诉。

我们已成功向上游了我们提交编写的补丁，现在所有人都可以使用这些补丁构建自己的 CI 系统。

容器化的优势——尤其对持续集成系统来说——不可低估。所有维护系统的团队都应考虑容器化，尽管初始的迁移成本较高，但回报非常值得。

---

**Ben Cooksley** 是一名会计师和计算机科学家，因其在 KDE 社区的贡献而闻名，尤其是在系统管理和基础设施方面。他对系统管理的兴趣源于对系统运作和集成的好奇心。
