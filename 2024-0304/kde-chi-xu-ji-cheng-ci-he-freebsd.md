# FreeBSD 与 KDE 持续集成（CI）

- 原文链接：[KDE CI and FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/development-workflow-and-ci/kde-ci-and-freebsd/)
- 作者：Ben Cooksley

![KDE CI 文章配图](../png/2024-0304/kde-chi-xu-ji-cheng-ci-he-freebsd-1.png)

自 2011 年 8 月起，KDE 就开始实施 CI（Continuous Integration，持续集成）系统，并持续改进。自此，系统已经大幅演进，增加了对多个 Qt 版本（大多数 KDE 软件所使用的工具包）以及多个平台的支持。

得益于日益普及的容器，才能在涉及的多个操作系统上可靠且一致地运行所有这些构建。为了理解容器解决了哪些问题以及构建可扩展 CI 系统中的其他挑战，我们需要回顾 KDE CI 的起点。

系统起初是相对简单的 Jenkins 配置——在托管 Jenkins 的同一服务器上进行构建。这让事情变得很简单，但也有局限性。然而，随着更多项目接入，构建需求增加，需要更多的机器。

这带来了一个难题：KDE 软件的构建需要其他 KDE 库——通常是最新版本的。这意味着仅增加构建机器是不够的，还需要确保最新的依赖项始终可用。

由于构建应用程序所需的完整依赖链时间较长，因此无法每次重新构建全部内容，这就需要共享构建产物。经过快速评估，选择了 rsync，一切又恢复正常。

## 引入 FreeBSD

到 2017 年，CI 系统需要支持新的平台，于是 FreeBSD 首次加入系统。FreeBSD 的早期支持较为简单，在 Linux CI 工作节点上采用虚拟机运行。每台虚拟机在 KDE on FreeBSD 团队的协助下单独配置，包含构建 KDE 软件所需的所有依赖项。

这种方法也有不足之处。虽然我们确保所有构建器都使用相同的自定义 FreeBSD 仓库（其中包含构建软件所需的所有依赖项），但每台机器仍然单独构建。这使得系统的扩展并不轻松，因为任何变更都必须逐台应用到每个构建器上。

不过，这一方法确实保证了 KDE 软件能在 FreeBSD 上可靠构建，并确保依赖项在 KDE 软件开始使用之前就已打包就绪，大幅改善了 KDE on FreeBSD 团队的开发体验。

在添加 FreeBSD 支持的同时，我们还在 Linux 构建中引入了 Docker——当时这对 Linux 构建来说尚属新鲜事物。能够首次创建一个可在所有构建器中分发的主配置，方便了 CI 系统的变更发布，无需再手动应用到每台机器，标志着基于容器的构建黄金时代的开始。唯一的缺点是 Docker 仅适用于 Linux，如何在其他平台上再现这一流程成为一个问题。

然而，在解决那个问题之前，构建能力的扩展已经导致一些新的、略有些意外的问题开始出现。构建偶尔会随机失败，日志显示文件缺失和符号链接损坏，但随后检查发现文件存在，之后的构建可顺利完成。问题在于：原子性（Atomicity）。

此前，我们只有少量构建节点，性能有一定限制。新硬件性能更强，构建速度加快，这意味着 rsync 正在上传文件时另一个构建节点下载构建产物的概率增加。这正是我们在某些构建中看到文件缺失和符号链接损坏的原因——不幸的构建恰好在某个依赖项正在同步构建结果时启动。

幸运的是，答案再次相当简单——改用 tar 包形式的构建产物。这让我们能以一次流畅的原子操作发布构建的完整文件集，再加上协议改用 SFTP（以兼容不支持 rsync 的平台），CI 系统再次恢复稳定运行，资源更充足，支持的平台也更多。

但手动维护机器的问题仍然存在。迁移到 Gitlab 和 Gitlab CI 后，这个问题比以往任何时候都更明显，构建节点由于累积的代码检出和构建产物迅速耗尽磁盘空间，测试留下的进程也会消耗 CPU 时间（有时甚至占用整个核心）——而这些在基于 Docker 的 Linux 构建中根本不存在。

我们探讨了多种解决方案，如改进 Gitlab Runner 及其处理 `shell` 执行器的方式、清理构建产物和代码检出的 cron 作业，以及基于 FreeBSD Jail 的解决方案，但均无法复现 Linux 上 Docker 的体验。

## 发现 Podman

某天早上，我们在研究 FreeBSD 的容器化选择时偶然发现了 Podman 及其搭档 ocijail 支持。这正是我们在基于 Docker 的 Linux 设置中习惯享受的功能，但现在能在 FreeBSD 上实现了。

重要的是，这意味着之前遇到的残留进程和需要手动清理的构建产物问题都能得以解决。此外，我们还可以利用标准的 OCI（Open Container Initiative，开放容器计划）注册表（例如 Gitlab 内置的容器注册表）来把 FreeBSD 镜像分发到所有构建器上，从而解决了单独维护每台机器的问题。

首要困难是构建一款能用的镜像。对于 Linux 系统来说，Docker 和 Podman 非常成熟，有详细的文档说明可用的基础镜像及其包含内容。但在找到合适的 FreeBSD 基础镜像后，我们以为只需添加常规的 FreeBSD 包仓库，安装所需软件包即可。

我们很快遇到了第一个障碍：在容器中首次构建时，CMake 出乎意料地报告无法找到编译器。我们认为这很奇怪，因为 FreeBSD 系统通常预装了编译器。经调查，我们发现了 FreeBSD 容器与正常 FreeBSD 系统的第一个主要区别：容器经过大幅精简，因此不包含编译器。

经过数次迭代，我们添加了编译器和 C 库开发头文件，从而在 FreeBSD 容器中成功构建了首款 KDE 软件。我们以为一切搞定，便继续推进——结果后续的 KDE 软件构建失败，因为需要额外的开发包。经过多次迭代（包括安装更多开发和非开发 FreeBSD-* 包）后，我们终于完成了多个关键 KDE 软件包的构建。

搞定这些后，注意力转向 Gitlab Runner 所称的“辅助镜像”，它用于执行 Git 操作和上传构建产物到 Gitlab 等工作。虽然可以借助 FreeBSD 的支持来运行 Linux 二进制文件，但那不是完美的解决方案。于是我们自然决定为 FreeBSD 原生构建。仿照 Gitlab 自己构建镜像的方法，但在 FreeBSD 上进行，我们很快就得到了认为是最后一块拼图的镜像。

冒险之旅的乐趣部分开始了：深入 Gitlab Runner 和 Podman 的内部机制。首次将 Gitlab Runner 连接到 Podman（使用其 Docker 兼容性选项）后不久，构建就遇到报错“unsupported os type: freebsd”。

快速搜索 Gitlab Runner 代码库后发现，Docker 会检查远程 Docker（或在我们的例子中是 Podman）主机的操作系统。我们对 Gitlab Runner 进行了快速补丁和重建，得到了一个非常相似但不完全相同的报错：“unsupported OSType: freebsd”。进一步修补后，又遇到第三个更不祥的错误，尤其是考虑到 Gitlab Runner 是用 Go 语言编写的：

```go
ERROR: Job failed (system failure): prepare environment:
Error response from daemon: runtime error: invalid memory address or nil pointer
dereference(docker.go:624:0s.
Check https://docs.gitlab.com/runner/shells/index.html#shell-profile-loading for more
information.
```

显然，要使该功能正常工作需要进行更多修改，但由于容器化构建的潜力，我们坚持研究该问题。经过对 Gitlab Runner 代码库的研究，我们找到了一段看起来没什么特别的代码：

```sh
inspect, err := e.client.ContainerInspect(e.Context, resp.ID)
```

于是开始了数小时的调试，寻找为什么这一行代码在 FreeBSD 上失败，但在 Linux 上却完全正常（无论是 Podman 还是 Docker）。最终，我们偶然发现了原因：Podman 守护进程本身崩溃并中断了请求。掌握这一信息后，通过尝试对正在运行的容器运行 `podman inspect`，很容易就复现了问题，得到了我们想看到的预期崩溃。成功！

在 Gitlab Runner 中搜索完毕后，注意力转向 Podman 本身。不久，原因被缩小到专门用于 `inspect` 操作的代码，随后又确定了一行无论平台如何都试图与 Linux 专用构造交互的代码。再次修补后，我们的 `podman inspect` 不再崩溃，随后不久，第一次 FreeBSD 构建成功启动。

## 在 FreeBSD 上运行构建

首次构建可能失败了（由于已知的 Git 以及 Gitlab Runner 与以非 root 用户身份运行的容器交互的问题），但重要的是我们在 FreeBSD 上已成功运行构建。

你可能以为此时我们大功告成，可以为所有 KDE 项目推行基于 FreeBSD 的容器化构建了。然而，最终测试发现最后一个问题：FreeBSD 容器的网络速度似乎比预期慢不少，与 FreeBSD 主机的能力相比明显较慢。

幸运的是，这个问题不是新问题，其他人之前也遇到过，我们也预计会碰到。Tara Stella 过去曾详细记录过这个问题，那是在她探索 Podman 和 FreeBSD 容器世界的经历中，由大型接收卸载（LRO）引起的。简单更改配置后，我们达到了预期的性能，终于可以投入生产。

如今，KDE 专门使用 Podman 和基于 ocijail 的容器来运行 FreeBSD CI 构建，有 5 台 FreeBSD 主机处理构建请求。构建中使用两个不同的 CI 镜像——分别适用于 Qt 5 和 Qt 6 两个支持的版本，确保 KDE 软件可以从零开始干净构建，并可选地通过所有单元测试。

自 FreeBSD 专用虚拟机迁移到 FreeBSD 容器化构建后，我们曾经因构建器故障而收到开发人员投诉，每周需要维护数次（有时甚至每天），如今已减少到几周无投诉，只需定期维护。

我们编写的补丁（Podman 和 Gitlab Runner 各仅几行）已成功提交至上游，现在所有人都可以使用并享受这些补丁来构建自己的 CI 系统。

容器化的优势——尤其对持续集成系统来说——不可低估。所有维护系统的团队都应考虑容器化，尽管初始的迁移成本较高，但回报非常值得。

---

**Ben Cooksley** 是一名会计师和计算机科学家，因其在 KDE 社区的贡献而闻名，尤其是在系统管理和基础设施方面。他对系统管理的兴趣源于对系统运作和集成的好奇心。
