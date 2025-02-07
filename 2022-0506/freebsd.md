# WPT/CFT：FreeBSD 启动性能

- 原文链接：[WPT/CFT: FreeBSD Boot Performance](https://freebsdfoundation.org/wp-content/uploads/2022/06/WIPCFT-FreeBSD.pdf)
- 作者：**TOM JONES & MITCHELL HORNE**

对于我们中的许多人来说，FreeBSD 系统启动所需的时间大多数是由机器从开机到通过系统固件进入加载程序（loader）所花费的时间决定的。在服务器硬件上，初始化系统并进入加载程序的第一阶段可能需要几分钟。在这种环境中，很难过多操心 FreeBSD 启动过程中的几秒钟。

但在云环境中，云服务按秒计费，启动过程中所消耗的时间就是浪费的时间，这是你被收费却没有任何回报的时间。更快的启动时间意味着你的平台可以更容易地在动态扩展的环境中使用。如果你的系统启动需要几分钟，而不是几十秒或个位数的秒数，你就必须更多地猜测未来的负载，而无法根据需求轻松地快速启动或停止主机。

云环境是添加提升 FreeBSD 启动性能的第一个工具集的地方。2018 年，Colin Percival 在 AsiaBSDCon 上展示了《Profiling the FreeBSD kernel boot: From hammer_time to start_init》。Colin 的工作向内核中添加了一个带时间戳的事件日志——称为 TSLOG——它可以用来跟踪内核在启动过程中每个子系统花费的时间 <https://papers.freebsd.org/2018/bsdcan/percival-profiling_the_freebsd_kernel_boot/>。

TSLOG 框架跟踪通过 TSLOG 选项编译到内核中的事件。当没有该选项时，事件会编译为空。TSLOG 事件会进入一个缓冲区并累积，直到缓冲区满了然后被静默丢弃，这使得 TSLOG 更倾向于捕捉早期的事件，而不是系统中的后期事件。

通过 TSLOG，启动时间可以被跟踪和分析，以便分析系统的输出日志。Colin 使用称为 FlameCharts 的时间共享图表进行分析，FlameCharts 非常适合这个用途，FlameCharts 就像 FlameGraphs [https://www.brendangregg.com/flamegraphs.html]，不过它们按时间顺序排列，而不是按字母顺序。

在每个火焰图中，水平时间表示启动过程中在每个区域花费的时间，垂直方向则按子系统进行划分。

![](https://github.com/user-attachments/assets/e02da4a2-2b78-4489-9178-70661348ff0f)

![](https://github.com/user-attachments/assets/0d09a5d3-69fd-402f-a4dd-2bd7837ed8fd)

**图 1 和图 2 显示了 Colin 和其他人如何使用 TSLOG 来找到启动过程中慢的时间段。图 1 显示了大约 2017 年我们所处的位置，图 2 显示了过去 5 年他们已经从启动过程中去除的时间，大部分的艰苦工作发生在最近。**

## BootTrace
TSLOG 是发现内核在启动过程中花费时间的一个好方法。在 2022 年初，Mitchell Horne 开始将 NetApp 的一个框架 BootTrace 上游化。BootTrace 让我们能够执行与 TSLOG 相同的跟踪，但它有几个主要增强功能，提供了更广泛的覆盖。BootTrace 提供了增强的能力来跟踪用户空间进程，让我们能够覆盖 rc 子系统，还可以跟踪关闭过程。

关闭过程很难通过 TSLOG 跟踪，因为完成时主机已经关闭。在某些方面，关机（或重启）与开机同样重要，关闭系统所花费的时间有助于系统一致性，但在关机过程中主机实际上并没有在执行任何工作。BootTrace 绕过了这个限制，提供了一个很好的视图，让我们能够了解系统如何启动和停止。

## 如何贡献
现在，我们拥有大量工具来查看 FreeBSD 启动和关机过程的性能，但开发者只能查看他们能够访问的系统和应用工作负载。

Colin 专注于在 Amazon Web Services 上运行 FreeBSD。还有许多其他云服务提供商，每个提供商都有可能在某些方面可以进行调整，以便从 FreeBSD 系统中获得更快的启动和关机时间。

改善这些领域中 FreeBSD 性能的最佳方式是运行我们现有的工具并分享生成的 FlameCharts。Colin 渴望获得更多云系统和非云系统的启动 FlameCharts。你可以使用 FlameCharts 找出启动和关机过程中的“热点”（或者可能是“冷点”）。待这些热点被识别出来，FreeBSD 项目的开发者就可以确定如何在每种情况下提高性能。

TSLOG 带来了 FreeBSD 启动过程的显著改进。自 Colin 在 2017 年开始以来，FreeBSD 启动时间从 11.1-RELEASE 时的约 30 秒，减少到了 2022 年 3 月在 14-CURRENT 上的约 9 秒。这些改进中有些表现为秒级的提升，但更多的是以 100 毫秒为单位的减少，表现在各个子系统初始化时的时间缩短。虽然还有一段距离才能让 FreeBSD 启动时间接近 Linux，它可以在 EC2 上以约 1 秒的时间启动，但达到这一目标的途径是通过现在已有的启动性能分析工具进行测试，并向项目开发者指出需要改进的地方。

## 使用 TSLOG 跟踪启动时间
如果你有构建和使用自定义 FreeBSD 内核的经验，运行 TSLOG 是相对简单的。你需要构建一个带有 'options TSLOG' 的自定义内核，然后运行 Colin 在 freebsd-boot-profiling 仓库中提供的脚本 <https://docs.freebsd.org/en/books/handbook/kernelconfig/>。这些步骤的最新版本应该可以在 Boot Time FreeBSD wiki 页面找到 <https://wiki.freebsd.org/BootTime>。

有了 Colin 脚本生成的 FlameChart，你可以开始进行更加困难的过程——识别启动过程中那些耗时过长的环节。根据当前的信息，rtsold 在我的系统中占用了大量的用户空间启动时间。这个守护进程对进行 IPv6 自动配置非常重要，但如果你使用 DHCP6 进行网络配置，它也可能是提高系统启动性能的一个良好起点。

## 使用 BootTrace 跟踪启动和关机时间
从 FreeBSD 14-CURRENT 开始，NetApp 的 BootTrace 框架已经集成到 FreeBSD 内核中。BootTrace 是内核的一部分，但默认是禁用的。BootTrace 允许跟踪启动、运行时和关机事件，并将它们存储在三个独立的日志中。在 2022 年 3 月的 FreeBSD 14-CURRENT 系统中，你可以通过在 `/boot/loader.conf` 中设置 sysctl `kern.boottrace.enabled` 为 `1` 来启用跟踪。启用后，系统会通过 sysctl `kern.bootrace.log`  输出启动和运行时日志。

![](https://github.com/user-attachments/assets/3ab0d7fa-b8e5-4121-affc-c6a6a03f6325)


通过将 sysctl `kern.bootrace.shutdown_trace` 设置为 `1` 并关闭系统，可以进行关机跟踪。关机跟踪是一个相当困难的问题。在关机过程的最后，直到启动性能跟踪时，系统已经关闭，你无法从中获取信息。为了解决这个问题，BootTrace 会将关机日志记录到系统的控制台。要恢复该日志，你需要一个能够记录发送到它的消息的控制台（例如串口）。

我测试系统的日志如下所示：

![](https://github.com/user-attachments/assets/3c338068-5c3f-4f68-9989-5b683f7d40f6)


BootTrace 提供了三个仅写的 sysctl：boottrace、runtrace 和 shuttrace。当写入时，事件将被记录为 `${procname}: name`。这些 sysctl 使得将 BootTrace 日志添加到自己的应用程序变得更加容易。

## 处理你得到的结果

待你在你的环境中识别出启动过程中的某个区域，你就需要确定是否可以改进，且如果可能的话，提出改进的建议。从 TSLOG 的结果中，你应当首先查看 FlameChart 中占用大量启动时间的项，并深入分析这些部分。

更多的子系统和用户空间组件可以从 BootTrace 事件中受益。将这些事件添加到你的工作负载的启动和关机脚本中，可以提供有关问题所在的见解，并可能发现一些影响 FreeBSD 用户的潜在问题。

某些子系统内置了长时间的延迟，以便允许其他网络主机同步。这类延迟可能是优化启动过程的优选候选项。

FreeBSD 开发者欢迎通过 bug 跟踪器提供的良好的 bug 报告，如果你能够提供提示或补丁来解决启动性能问题，那就更好了。

FreeBSD 启动过程永远不会完成。随着时间的推移，随着服务和硬件的变化，它会有所不同。许多最显著的启动性能改进来自于减少与模拟遗留硬件的交互。在多核系统中，大量时间花费在向模拟的 VGA 控制台写入字符上。随着时间的推移，硬件将逐渐被淘汰，速度更快或更慢的设备将会出现。在你的帮助下，我们可以保持 FreeBSD 在云环境中的竞争力，避免浪费启动机器的时间。

---

**TOM JONES** 希望基于 FreeBSD 的项目能够获得应有的关注。他住在苏格兰东北部，并提供 FreeBSD 咨询服务。
