# KDE 持续集成（CI）和 FreeBSD

 由 BEN COOKSLEY

持续集成（CI）是 KDE 近年来致力于改进的事项，KDE 软件的第一个 CI 实现始于 2011 年 8 月。自那时以来，该系统已经显著发展，不仅支持多个版本的 Qt（用于编写大多数 KDE 软件的工具包），还支持多个平台。

能够可靠且一致地运行所有这些构建，在涉及的多个操作系统上，这仅仅是因为容器的普及。要理解容器解决的挑战以及构建可伸缩 CI 系统中的其他挑战，我们必须回到 KDE CI 的起源。

当系统首次启动时，它是一个相对简单的 Jenkins 设置，构建是在托管 Jenkins 的同一台服务器上执行的。这使得生活相当简单，但也存在一些限制。随着更多项目加入系统并要求构建的增加，很快就显现出需要更多机器的需求。

这个问题似乎有点棘手，因为 KDE 软件往往需要其他 KDE 库才能构建，并且通常不仅仅是任何版本 - 通常是最新版本。 这意味着这不仅仅是增加构建器的数量的问题，我们还需要确保依赖项的最新版本仍然可用。

由于构建我们应用程序所需的完整依赖链所需的时间太长，很快就排除了每次都构建每样东西的概念 - 这意味着有必要共享那些构建结果的二进制文件。 在快速审查我们的选择后，rsync 迅速被选为我们的首选，一切又恢复了正常。

## 进入 FreeBSD

2017 年，系统遇到了它的下一个一波增长阵痛，因为开始为新平台添加支持变得令人期待，这也是 FreeBSD 第一次进入画面。

在我们的 CI 系统中，FreeBSD 支持的最初实现相对简单，并且利用了在我们的 Linux CI 工作机上运行的虚拟机。这些机器是根据 KDE on FreeBSD 团队的帮助单独设置的，就像当时的我们的 Linux 构建一样，包括构建 KDE 软件所需的一切。

然而，这种方法确实有它的缺点。虽然我们确保所有构建器都使用了同一个自定义 FreeBSD 仓库，其中包含构建我们软件所需的所有依赖项，但每台机器仍然是单独构建的。这使得系统的扩展并不简单，因为任何更改都必须逐个应用到每个构建器。

然而，它成功地确保了 KDE 软件在 FreeBSD 上可以可靠构建，并确保依赖项在 KDE 软件开始使用它们之前提前打包，大大改善了 KDE 在 FreeBSD 上的体验团队的体验。

与我们为 FreeBSD 添加支持的同时，我们还采用了当时在我们的 Linux 构建中仍然很新的东西 — Docker。这是我们第一次能够生成一个单一的主设置，可以分布在所有我们的构建器上，使得在 CI 系统中轻松推出更改而无需手动应用到每台机器上。基于容器的构建的黄金时代已经开始到来。唯一的缺点是它只适用于 Linux，因此问题仍然存在，如何在其他平台上复制这一点。

在我们能够解决这个问题之前，构建能力的所有这些增长导致一些新的、稍微意外的问题开始显现。不时地，构建会随机失败，日志显示文件丢失或符号链接损坏。后来的检查显示文件是存在的，并且随后的运行顺利完成。问题出在哪里？原子性。

到目前为止，我们只有少数几个构建节点，它们在性能上有一定的限制。然而，新的设置采用了更强大的硬件，因此完成构建的速度更快——这意味着当另一个构建尝试下载构建结果时，rsync 很可能正在进行中传输。这就是为什么我们会在某些构建中看到丢失文件和损坏的符号链接，因为不幸的是，该构建恰好在其依赖项之一在同步构建结果时开始。

幸运的是，答案再次非常简单——转而使用构建结果的 tarballs。这使我们可以在一个流畅的原子操作中发布构建的全部文件集，并且还通过切换到 SFTP 协议（以适应没有 rsync 的平台）意味着 CI 系统再次平稳运行——支持更强大的资源和更多的平台。

然而，手工维护各个机器的问题并没有消失。迁移到 Gitlab 和 Gitlab CI 使这个问题比以往任何时候都更加显而易见，因为构建节点开始由于累积的代码检出和其他构建产物而耗尽磁盘空间。我们还要解决测试中遗留进程占用 CPU 时间的问题（有时甚至是整个核心）——所有这些问题在我们基于 Docker 的 Linux 构建中根本不存在。

“关于这个问题讨论了许多选项和解决方案，包括改进 Gitlab Runner 以及它如何处理“shell”执行程序，使用定时作业清理构建产物和代码检出，还有在 FreeBSD Jails 上构建某些东西。然而，这些都无法复制我们在 Linux 上使用 Docker 时的体验。”

## “寻找 Podman”

“所以有一天早晨，当我们在查看 FreeBSD 容器化选项时，我们偶然发现了 Podman 及其伴侣 ocijail 支持。这为我们承诺了一切我们习惯于在基于 Docker 的 Linux 环境中享受到的东西，但在 FreeBSD 上。”

显著的是，这意味着我们以前遇到的关于流浪进程和残留构建工件的问题，需要手动清理的问题将得到解决。而且，这还让我们能够利用标准的开放容器倡议注册表（例如 GitLab 内置的容器注册表）来分发 FreeBSD 的镜像给我们所有的构建机器——解决了我们需要单独维护这些机器的问题。

建立一个可工作的镜像将是我们面临的第一个挑战。对于 Linux 系统，Docker 和 Podman 已经非常成熟，并且有详细的文档介绍可用的基础镜像及这些镜像包含的内容。然而，在找到适当的 FreeBSD 基础镜像之后，我们认为只需添加我们正常使用的 FreeBSD 软件包仓库，并安装所有我们通常需要的东西就可以了。

很快我们就会在道路上遇到第一个障碍，出乎意料的是，CMake 在容器中第一次构建时提示找不到编译器。我们觉得这非常奇怪，因为通常 FreeBSD 系统都会预装编译器。经过一番调查，我们发现了 FreeBSD 容器与正常 FreeBSD 系统之间的第一个主要差异——即它们被大幅精简，因此不包括编译器。

几次迭代后，我们不得不添加编译器和 C 库开发头文件——这样我们的第一个 KDE 软件就可以在 FreeBSD 容器中构建了。认为一切都已经准备就绪，我们继续前进——但随后的一些 KDE 软件失败了，因为需要额外的开发包。经过多次迭代（包括安装了大量的开发和非开发 FreeBSD-*包），我们终于得到了一些关键 KDE 软件的完成构建。

有了这个解决方案，注意力现在转向 Gitlab Runner 所称的“辅助映像”，这是它用来执行 Git 操作并从构建上传构建产物到 Gitlab 本身等其他操作的东西。虽然我们可以利用 FreeBSD 支持来运行 Linux 二进制文件，但那将是一个不完美的解决方案。因此，我们自然而然地开始为 FreeBSD 本地构建这个。复制了 Gitlab 自己构建镜像的方式，但是在 FreeBSD 中进行了复制，很快我们就有了我们认为将是最后一块就绪的部分。

现在冒险的有趣部分开始了，我们深入研究了 Gitlab Runner 和 Podman 的内部工作机制。第一个障碍是在我们将 Gitlab Runner 连接到 Podman 后的片刻（使用它的 Docker 兼容选项）,当我们的第一个构建收到“不支持的操作系统类型：freebsd”消息。

通过快速搜索 Gitlab Runner 代码库，发现对于 Docker，它会检查远程 Docker（或在我们的情况下：Podman）主机的操作系统。稍作补丁并重新构建 Gitlab Runner 后，我们遇到了一个非常相似但并非完全相同的错误：“不支持的 OSType：freebsd”。随后对 Gitlab Runner 进行了更多的补丁，只得到了第三个更加不祥的错误，特别是考虑到 Gitlab Runner 是用 Go 编写的：

`ERROR: Job failed (system failure): prepare environment:Error response from daemon: runtime error: invalid memory address or nil pointerdereference(docker.go:624:0s.Check<span> </span><a href="https://docs.gitlab.com/runner/shells/index.html#shell-profile-loading">https://docs.gitlab.com/runner/shells/index.html#shell-profile-loading</a><span> </span>for moreinformation.`

由此明显需要做更多的工作才能让这个工作起来，然而，考虑到容器化构建可能带来的好处，我们坚持并开始查找出现问题的地方。经过对 Gitlab Runner 代码库的一些研究，我们发现一些代码似乎并没有做什么特别的事情：

`inspect, err := e.client.ContainerInspect(e.Context, resp.ID)`

于是开始了漫长的调试之旅，寻找为什么这一行代码在 FreeBSD 上失败，但在 Linux 上却完全正常（无论是 Podman 还是 Docker）。然而最终，我们发现了问题的根源：Podman 守护程序本身崩溃并放弃了请求。有了这些信息，问题很快就能通过尝试对运行中的容器运行“podman inspect”来轻松重现，从而得到我们想要看到的崩溃。成功！

通过搜索 Gitlab Runner，现在将焦点转向 Podman 本身。不久之后，问题被缩小到了专门调用“inspect”操作的代码，不久之后，确定了一行代码，试图与 Linux 特定的结构进行交互，无论平台如何。然而，另一个补丁之后，我们有了一个不会崩溃的“podman inspect”，然后不久之后，我们的第一个 FreeBSD 构建成功启动。

## 在 FreeBSD 上运行构建

那第一个构建可能失败了（由于 Git 存在已知问题，以及 Gitlab Runner 与作为用户而非 root 运行的容器交互的方式），但重要的是，我们在 FreeBSD 上成功运行构建。

此时，您可能会认为我们已经尽快并且可以开始将基于 FreeBSD 的容器化构建应用到所有 KDE 项目中。但是最终测试揭示了一个最终问题：我们在 FreeBSD 容器中的网络速度似乎比我们预期的要慢得多，远远低于 FreeBSD 主机所能达到的速度。

幸运的是，这不是一个新问题，其他人之前遇到过这个问题，我们预料到会遇到这个问题。这个特定的问题在过去由 Tara Stella 进行了详细描述，他们在深入研究 Podman 和 FreeBSD 容器的世界时遇到了这个问题，这个问题是由大型接收卸载（Large Receive Offload，LRO）引起的。稍作配置更改后，我们获得了预期的性能，最终准备好投入使用。

如今，KDE 专门使用 Podman 和基于 ocijail 的容器进行 FreeBSD CI 构建，有 5 个 FreeBSD 主机系统处理构建请求。这些构建是使用两种不同的 CI 映像执行的——每种映像分别用于两个支持的 Qt 版本（即 Qt 5 和 Qt 6），确保 KDE 软件可以从头开始进行清洁构建，并且可以选择完全通过单元测试。

自从从 FreeBSD 专用虚拟机迁移到 FreeBSD 容器化构建后，我们从开发人员因构建器故障而投诉并每周甚至每天多次进行维护，转变为数周未收到投诉，仅需定期维护。

我们编写的补丁（仅为 Podman 和 Gitlab Runner 各几行）已成功上游，并现在应可供所有人在构建自己的 CI 设置时使用和享用。

切换到容器的好处，特别是对于持续集成系统，不容小觑，任何维护系统的团队都应考虑调查，因为回报远远超过迁移的初始成本。

BEN COOKSLEY 是一名会计师，也是一名计算机科学家，以其在 KDE 社区中在系统管理和基础设施方面的贡献而闻名。他对系统管理员工作的兴趣源于对系统运作和集成方式的好奇。
