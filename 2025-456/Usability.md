# 使安装程序易于使用

- [Installer Usability](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-3/installer-usability/)
- 作者：Pierre Pronchery

亲爱的读者：

在撰写本文时，我完全不知道你是谁。这可能是你首次接触有关 FreeBSD 的文章；你也可能已经对系统非常熟悉了——毕竟你可能亲自编写了其中一半的代码。我也不清楚你正在使用何种硬件来阅读这篇文章；它甚至可能尚未问世，也不知你是否有任何残障或不便。

这正是操作系统安装程序所要应对的挑战。它必须在各种情况下都能正常工作，同时对所有人都要具有吸引力和可用性，甚至需要预见未来的演进。这是一项艰巨的任务。

安装程序往往是潜在用户与系统第一次交互的入口，因此它负责形成第一印象。与此同时，专业用户在个人硬件上不太需要频繁使用安装程序——基本上只在设置新系统时用到——但他们可能需要自动化安装多台系统来执行各种任务，或者在专业场景下为客户部署。

综上所述，这些不同方面可以从一个角度来看：可用性。但什么是可用？更重要的是，与其他系统相比，FreeBSD 安装程序的可用性又如何？

## 现状

### Microsoft Windows 的演进

安装程序历史悠久。我的初次接触要追溯到 1989 年，家里引入计算机的时候。那时通常先安装 MS‑DOS，然后在其上安装 Windows 3.1。显然，安装 MS‑DOS 是在文本模式（80 列 VGA）下进行的，但可以禁用颜色——这也起到了一种粗糙的无障碍功能：高对比度。

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.36.01%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.36.16%CE%93CPM.png)

虽然安装过程在文本模式下进行，但进度条和针对用户输入的高亮已经问世。

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.36.36%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.37.00%CE%93CPM.png)

另一方面，Windows 安装程序（此处为 3.11 版）很快使用了自身的图形界面进行安装，如下所示。

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.37.15%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.37.42%CE%93CPM.png)

值得一提的是，我在 VirtualBox 中，基于 archive.org 上的公共领域材料，自己制作了这些截图 ¹²³。

接下来是 Windows 95。和 Windows 3.11 一样，它在安装程序中使用了自身的用户界面，不过有两个中间步骤。通过以下截图可以看出，安装过程相对无痛，但需要用户交互的步骤明显增多 ⁴。

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.38.22%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.38.40%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.39.15%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.39.29%CE%93CPM.png)

将 Windows 98 纳入讨论值得一提，因为它带来了一些改进：安装步骤总览列表和剩余时间估算 ⁵。

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.39.51%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.40.11%CE%93CPM.png)

我就不再展示其他版本的 Windows 安装过程了。只要加上 Windows 7，就能总结出以下有关安装程序可用性的模式 ⁶：

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.40.25%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.40.38%CE%93CPM.png)

我们可以无休止地争论 FreeBSD 是否真正与 Microsoft Windows 竞争。从 FreeBSD 项目的官方网站来看，它确实有瞄准桌面系统的雄心，如今已有 Laptop Desktop Working Group（LDWG，笔记本桌面工作组）为证。不过，无论这场争论的结果如何，我认为我们都能从消费级市场中获益，哪怕只是作为参考。以下是在实际体验中浮现出的若干模式和标准：

| 评估标准          | MS‑DOS 6.22    | Windows 3.11 | Windows 95 | Windows 98 | Windows 7 |
| ---------- | -------------- | ------------ | ---------- | ---------- | --------- |
| 语言支持       | 为每种语言分别发布 |为每种语言分别发布  |    为每种语言分别发布  |     为每种语言分别发布      |             支持      |
| 无障碍功能      | 支持（高对比度）      | 不支持           | 不支持         | 不支持         | 支持（放大镜、屏幕键盘、高对比度） |
| 许可协议       | 不支持             | 不支持           | 支持        | 支持        | 支持       |
| 操作步骤数量     | 9（共 3 张软盘）     | 15（前 5 张软盘）  | 21         | 17         | 10        |
| 支持导航       | 不支持             | 不支持           | 支持        | 支持        | 不支持        |
| 高级模式       | 不支持             | 混合          | 支持        | 支持        | 支持       |
| 步骤清单       | 不支持             | 不支持           | 不支持         | 支持        | 混合       |
| 是否先提问      | 不支持             | 不支持           | 不支持         | 不支持         | 不支持        |
| 时间预估       | 不支持             | 不支持           | 不支持         | 支持        | 支持       |
| Live 系统支持  | 不支持             | 不支持           | 不支持         | 不支持         | 不支持        |
| 系统修复功能     | 混合            | 混合          | 混合        | 混合        | 支持       |
| 文本模式界面     | 支持            | 不支持           | 不支持         | 不支持         | 不支持        |
| 图形模式界面     | 不支持             | 混合          | 混合        | 混合        | 支持       |
| 图形化安装结果    | 不支持             | 支持          | 支持        | 支持        | 支持       |
| 自动化安装     | 不支持             | 不支持           | 不支持         | 不支持         | 支持       |


如果要总结这套安装器的优势，我会指出以下几点：

- 通常会提供整体进度的指示。
- 即便时间预估可能不够精准，它对终端用户来说依然具有参考价值。
- 仅保留最关键的决策项由用户选择（如目标磁盘和安装目录），而将技术性决策默认隐藏：
  - 文件系统的选择，
  - 驱动程序列表，
  - 桌面环境（显而易见），
  - 预装软件列表；或提供一个明确的入口进入高级模式。
- 安装过程中通常可以回退并更改之前的选择；有趣的是，Windows 7 移除了这个选项，但它同时也是安装流程最简单的版本，仅需 10 步。
- 安装完成后的系统可以直接用于桌面环境。

尽管在 Windows 7 发布之后又发布了三个主要版本，但在可用性层面上，我并未观察到显著的变化。

## FreeBSD 的现状

截至撰写本文时，FreeBSD 的最新稳定版本是 **14.2**。注 7、8

如果单从可用性（usability）的角度来看，FreeBSD 的安装器显得有些落伍了，尤其是在体验了 Windows 7 之后，这一点更加明显。

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.40.59%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.41.13%CE%93CPM.png)

| 评估标准        | FreeBSD 14.2 |
| --------- | ------------ |
| 语言支持      | 不支持           |
| 无障碍功能     | 不支持           |
| 许可协议      | 不支持           |
| 操作步骤数量    | 23           |
| 支持导航      | 不支持           |
| 是否先提问     | 不支持           |
| 高级模式      | 支持          |
| 步骤清单      | 不支持           |
| 时间预估      | 不支持           |
| Live 系统支持 | 混合（命令行）     |
| 系统修复功能    | 混合（命令行）     |
| 文本模式界面    | 支持          |
| 图形模式界面    | 不支持           |
| 图形化安装结果   | 不支持           |
| 自动化安装     | 支持          |

不过，这样的比较并不完全公平：**Microsoft Windows 明确面向非技术用户，追求“一套通吃”的图形化安装体验，并提供了完善的客户支持体系**，而 FreeBSD 更偏向资深用户和系统管理员。

那么，如果将它与更接近的竞品做比较，会是什么样的结果？


## Linux：Ubuntu Server（服务器版）

Ubuntu 会根据用途提供不同的安装镜像。为了保持对比的公平性，本文使用的是 Ubuntu Server 24.04.02 LTS。注 9

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.41.40%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.41.53%CE%93CPM.png)

乍一看，这套安装器和 FreeBSD 十分相似：整个安装过程都保持在纯文本界面（text mode）中，首启后也是登录到命令行。这种形式更符合 FreeBSD 用户的预期。不过，**实际操作中 Ubuntu 的安装流程显得更加简单直接**。

| 评估标准         | Ubuntu Server 24.04 |
| --------- | ------------------- |
| 语言支持      | 支持                 |
| 无障碍功能     | 不支持                  |
| 许可协议      | 不支持                  |
| 操作步骤数量    | 15                  |
| 支持导航      | 支持                 |
| 是否先提问     | 支持                 |
| 高级模式      | 支持                 |
| 步骤清单      | 不支持                  |
| 时间预估      | 不支持                  |
| Live 系统支持 | 混合（命令行）            |
| 系统修复功能    | 混合（命令行）            |
| 文本模式界面    | 支持                 |
| 图形模式界面    | 不支持                  |
| 图形化安装结果   | 不支持                  |
| 自动化安装     | 支持                 |


### 相较于 FreeBSD 的明显优势

- 安装器支持多种语言选择。
- 安装问题集中一次性提问，实际安装过程一气呵成。
- 安装期间可以选择预定义用途（如通过 Snaps 安装服务组件）。

这些细节虽然小，却对用户体验有明显的正向影响。如果 FreeBSD 安装器希望跟上当今的趋势，这些正是可以借鉴的改进方向。

## Linux Ubuntu（桌面版）

接下来，我尝试了 Ubuntu Desktop，版本同样是 24.04.02 LTS。

正如预期，这个版本提供了图形化的用户体验。但更重要的是，在我个人的感受中，其整体的质量与打磨程度处于另一个层次。

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.42.11%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.42.24%CE%93CPM.png)

| 评估标准    | Ubuntu（桌面版） |
| ------- | ----------- |
| 语言支持    | 有           |
| 辅助功能    | 有           |
| 许可协议    | 无           |
| 安装步骤数量  | 15 步        |
| 导航      | 有           |
| 先提问再操作  | 有           |
| 高级模式    | 有           |
| 步骤列表    | 无           |
| 时间估算    | 无           |
| Live 系统 | 有           |
| 修复系统    | 混合          |
| 文本模式    | 无           |
| 图形模式    | 有           |
| 图形化结果   | 有           |
| 自动化安装     | 有           |

在可用性方面，这无疑是一个参考标准。至于 Microsoft Windows，这种表现可以说是商业支持下的解决方案应有的水平。但为了更公平地比较 FreeBSD，我们应将其与另一个社区项目进行对比。因此，在深入探讨 FreeBSD 自身的安装程序之前，不妨先看看 Ubuntu 的母项目、非商业发行版：Debian。

## Debian GNU/Linux

**译者注：Debian 亦有高级安装模式（Expert Mode），原作者测评的是普通安装模式。在高级安装模式下可对每个选项进行精确规划。**

与 Ubuntu 不同，这个比较是一个“二合一”：Debian 的 netinstall 安装器体积仅有 632 MB，却同时包含了文本模式和图形化模式两种形式。

顺带一提：在我主要转向 BSD 系统之前，我长期使用的类 UNIX 发行版就是 Debian。在惊喜地发现其混合式安装方式之后，我再次遇到了令人沮丧的熟悉体验：安装过程提出一连串看似无止境的问题，而且每一组问题之间还伴随着耗时的处理过程。

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.42.44%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.42.56%CE%93CPM.png)

| 评估标准    | Debian GNU/Linux |
| ------- | ---------------- |
| 语言支持    | 有                |
| 辅助功能    | 无                |
| 许可协议    | 无                |
| 安装步骤数量  | 26 步             |
| 导航      | 混合               |
| 先提问再操作  | 否                |
| 高级模式    | 否                |
| 步骤列表    | 无                |
| 时间估算    | 混合               |
| Live 系统 | 有（其他镜像）          |
| 修复系统    | 混合（命令行）          |
| 文本模式    | 有                |
| 图形模式    | 有                |
| 图形化结果   | 有                |
| 自动化     | 有                |

尽管提供了图形模式，但其视觉外观相当粗糙。我也感到惊讶，除了启动菜单中的高对比度模式外，并未发现其他辅助功能；当然，也有可能是我漏看了。

不过，从非常积极的方面来说，要获得一个适用于服务器或桌面用途的安装环境是非常容易的。这一点与其安装器的混合设计非常契合。

那么，FreeBSD 能否也达成类似的成果？

### 特别提及：macOS

最后但同样重要的一点，我想提及 macOS。虽然 macOS 最初也基于 BSD 系统，但 macOS 能够依赖一套众所周知的自家硬件设备数据库。系统固件内置了多种备用机制，用于以不同方式（重新）安装系统——包括无需任何安装介质：整个系统可以由固件直接从互联网下载并恢复。

这一点令人印象深刻，堪比前文提到的商业解决方案；但同样，这样的比较对我们心爱的 FreeBSD 并不公平。不过，我仍想强调一点：系统中应当至少提供足够的基本工具，以便进行分析和修复操作；这在意外情况下可能就是救命稻草。

## 深入探讨 FreeBSD

根据我自己阅读其源码的经验，目前 FreeBSD 安装器的状况，直接源自一些架构层面的设计决策。我想强调的是，这并非对其实现方式或当时决策的全盘否定；重点在于理解我们如今所处的位置：这个安装器是否已到重写的时机，或者是否还有一些战役可以通过简单的手段去赢得？

### 架构

大多数 FreeBSD 安装器由 shell 脚本组成，其中部分步骤通过专门的 C 程序实现。可在 `src.git` 仓库中找到其完整源码，但其底层架构实际上由三大组件构成：

- **bsdconfig**：位于 **usr.sbin/bsdconfig**，作为一套 shell 例程库被 **bsdinstall** 调用使用。
- **bsddialog**：位于 **usr.bin/bsddialog**，是与用户交互的工具，负责收集输入和显示进度。
- 最后是 **bsdinstall** 本身，位于 **usr.sbin/bsdinstall**。

当启动 FreeBSD 安装镜像时，有专门的 `rc.local` 启动脚本（来自 `release/rc.local`）在启动过程结束时运行 **startbsdinstall**。这个脚本提供欢迎界面，并允许将安装介质作为 Live 系统使用。在默认情况下，Live 模式只是启动一个 shell，这对有经验的 UNIX 用户来说是熟悉的环境，但对一般用户而言，并不算是真正意义上的 Live 或修复系统。

聚焦安装过程，其整体流程如下：

- 启动 **startbsdinstall**（如上所述）。

- **bsdinstall** 可直接调用特定操作，其子命令位于 `/usr/libexec/bsdinstall`，默认使用 `auto`（本次选择）。

- `auto` 会依次执行以下操作：

  - 如果存在，运行 **local.pre-everything**。
  - **keymap**：键盘布局选择。
  - **hostname**：设置目标系统的主机名。
  - 选择系统组件，根据 **MANIFEST** 文件（至少应包含 **base.txz** 和 **kernel.txz**）。

  如果安装介质中缺少某些系统组件：

  - 启动 **netconfig**。
  - （如存在）运行 **local.pre-partition**。
  - 根据检测到的平台（如 SMBIOS 或启动架构）应用已知的兼容性修正。
  - 选择分区方案（UFS 自动分区、手动或支持则可选择 ZFS）。
  - 应用分区方案并挂载目标文件系统。
  - （如存在）运行 **local.pre-fetch**。
  - 如果缺少系统组件：
    - 执行 **fetchmissingdists**（将缺失组件保存在目标系统的 **/usr/freebsd-dist** 中）。
  - 使用 **checksum** 与 **distextract** 验证并解压系统组件。
  - 执行 **bootconfig** 配置启动加载器。
  - （如存在）执行 **local.pre-configure**。
  - 执行 **rootpass** 设置 root 密码。
  - 如尚未配置网络，则再次运行 **netconfig**。
  - 一系列问题：
    - **time**：设置日期、时间与时区。
    - **services**：选择开机启动的服务。
    - **hardening**：应用若干加固选项。
    - **firmware**：选择安装所需固件包。
    - **adduser**：添加用户账户，此操作通过 **chroot(8)** 执行 **adduser(8)**，会带来与其他步骤略异的体验。
  - **finalconfig**：一个“并不终结”的菜单，能在重启前进行进一步更改。
  - 非交互式步骤 **config**：应用上述所有配置项。
  - 清除下载的系统组件。
  - （如存在）运行 **local.post-configure**。
  - 最后一个菜单：允许用户进入 **chroot(8)** 环境中的目标系统。
  - 非交互式安全步骤：保存熵（entropy）。
  - 最终运行 **umount**，以便安全重启。

- 回到 **startbsdinstall**，如果 **bsdinstall(8)** 成功运行，用户将看到提示信息，并可选择重启、关机，或启动一个 shell。

正面的第一点是，即便安装介质中缺少部分系统组件，安装流程也能妥善处理。这允许创建不同大小的安装介质，而无需改动安装逻辑。

但另一方面，这一过程对用户提问过多。虽然每一个问题单独来看都显得重要而有用，但前文描述的其他系统，仅需三分之一的交互步骤就能构建出完整可用的系统。不幸的是，用户界面的问题还不仅限于此。

## 限制与使用场景

对 **bsddialog(1)**（或 C 程序中使用的 **libbsddialog**）的高度耦合，引入了与 Debian 安装器类似的限制：每一步的用户界面都相当基础，且无法根据用户输入动态调整交互。例如，网络配置步骤不能依据常见习惯自动建议合适的网络掩码、DNS 服务器或网关。这不仅会节省用户一些按键输入和不必要的精力，还会让整体体验显得更现代、更精致。

**可能的解决方案：Lua**

这个具体问题可以通过在 **libbsddialog**（或 **bsddialog(1)**）中引入 Lua 脚本支持来解决。这个想法由 **bsddialog** 的作者 Alfonso Siciliano（asiciliano@）提出，他目前正考虑将该扩展实现到项目中。

**初学者用户**

对于初学者，或者说一般桌面用户来说，显而易见的改进方向是：能够方便地安装一个在首次重启后就可直接使用的桌面系统。虽然这意味着需要增加一个安装步骤，但可以借鉴上文提到的 Debian 安装器，为服务器和桌面系统都提供适配的选项。

目前在这个方向已有实验性工作（专门由 Alfonso 牵头），但“写起来容易，实现起来吃力”。其中一些挑战包括对应固件文件和驱动程序的安装，尤其是内核模块 DRM 一直是个难点。当前已有一项尝试（致谢 manu@）试图将这些模块从 Ports 移回基本系统中，这应该能缓解当前二进制 Ports 中常见的版本不匹配问题。

**企业用途**

这次要说点积极的方面：FreeBSD 的安装器可以调整并自动化，以适应企业部署中的典型需求。

首先，它默认就支持设置 jail，而不是执行常规安装。这种行为可以通过选择 jail 模式（而不是默认的 auto）来实现。操作如下所示：

```sh
# bsdinstall jail /path/to/the/new/jail
```

如果未来要对 FreeBSD 安装器进行重构或大幅修改，保留这一功能将是值得考虑的。另一方面，目前已有许多 jail 管理方案存在于 **bsdinstall(8)** 之外。

更重要的是，安装器还支持 script 模式，此时需要提供一个安装脚本，在 **bsdinstall** 上下文中执行。这种方式由系统管理员编写脚本，从而实现完全自动化的 FreeBSD 安装过程。

在本节结尾，我必须提及一个至今仍缺失的功能，而其他对比对象都已具备这一能力：安装器默认假设无需代理服务器即可访问互联网。这个问题记录在 Bugzilla 的条目 #214390 中。根据我自己的经验，在为数十家客户提供渗透测试部署的场景中，这种限制往往直接导致 FreeBSD 安装器无法采用。

## 开发环境

公平地说，开发 **bsdinstall(8)** 并不容易。测试某些修改通常需要构建一个完整的可启动镜像，在虚拟环境中运行它，并重复整个安装流程，直到抵达需要测试的那一特定步骤。为了提高开发安装器时的效率，我尝试构建了一个专门适配此目标的开发环境。

它由一个基于 PXE 的设置组成，可针对传统的 PXE 环境以及更现代的、基于 UEFI 的网络引导进行调整。我将此引导序列与一台 TFTP 服务器（由 `inetd` 管理）、DHCP 服务（使用 `isc-dhcp44-server` 软件包）以及只读的 NFS 挂载结合在一起。完整描述该设置超出了本文的范围，但归根结底，FreeBSD 安装器的可用性也取决于其开发环境的便利性。你可以根据自己的需求自由调整以下配置片段。

**对于 `/etc/exports`：**

```ini
/jail/bsdinstall -ro -network 192.168.x.0 -mask 255.255.255.0 -maproot 0:0
```

**对于 `/etc/inetd.conf`：**

```ini
tftp dgram udp   wait root /usr/libexec/tftpd   tftpd -l -s /tftpboot
tftp dgram udp6  wait root /usr/libexec/tftpd   tftpd -l -s /tftpboot
```

**对于 `/etc/rc.conf`：**

```ini
mountd_enable="YES"
nfs_server_enable="YES"
nfs_server_flags="-h 192.168.x.y"
rpcbind_enable="YES"
rpcbind_flags="-h 192.168.x.y"
```

**对于 `/usr/local/etc/dhcpd.conf`：**

```ini
option subnet-mask 255.255.255.0;
default-lease-time 600;
max-lease-time 7200;

subnet 192.168.x.0 netmask 255.255.255.0 {
  range 192.168.x.128 192.168.x.254;
  option routers 192.168.x.y;
  option subnet-mask 255.255.255.0;
}

option arch code 93 = unsigned integer 16;

host example {
  hardware ethernet aa:bb:cc:dd:ee:ff;
  fixed-address 192.168.x.127;
  next-server 192.168.x.y;
  if option arch = 0:07 {
    # UEFI
    filename "FreeBSD/boot/loader.efi";
  } else {
    # BIOS
    filename "FreeBSD/boot/pxeboot";
  }
  option root-path "192.168.x.y:/jail/bsdinstall/";
}
```

请将 `loader.efi` 和 `pxeboot` 文件放置于 **`/tftpboot/FreeBSD/boot`** 目录中。

你可以将 **`/jail/bsdinstall`** 文件夹设置为一个典型的 jail，并根据安装器的常规启动环境进行调整，或者也可以直接暴露 release 构建中的 `disc1` 目录。无论哪种方式，这种方法都能避免在开发 `bsdinstall` 时频繁地生成、传输或重启镜像文件。

需要注意的是，这一设置在物理硬件和虚拟机中都同样适用，因为 VirtualBox 等虚拟化平台支持从 PXE 引导。

## 图形模式

最后，是时候回答这个问题了：我们能否改进 FreeBSD 安装器，使其功能达到例如 Debian 安装器的同等水平？

### 背景

2024 年初，FreeBSD 基金会委托我调研开源领域中图形化安装器的最新进展。其根本目标是为 FreeBSD 项目寻找一种可行方案来增加图形安装功能。Linux 上一个新的安装框架 Calamares12 是显而易见的候选者——前提是确认它可以用于 FreeBSD 作为目标系统。不幸的是，由于 Calamares 使用 GPL 许可证，与 FreeBSD 基本系统的目标用户群不兼容。

另一个当时关注度较高、且特定于 FreeBSD 的替代方案是 PC-BSD。它经过首次品牌更名为 TrueOS 后，于 2020 年停止维护。随后建议的替代品包括 GhostBSD、MidnightBSD 和 NomadBSD。

GhostBSD 提供了图形安装器 gbi。虽然是个不错的候选，但它与当前 bsdinstall 代码库差异较大，同时还需要完整的 Python 和 Gtk+ 环境。在我看来，更高效的方案是可行的。

据我所知，MidnightBSD 和 NomadBSD 均未提供图形安装器（**译者注：NomadBSD 有图形安装器，这里原作者是错误的**）。

### 最小侵入式方案

相反，鉴于我已熟悉 **bsdinstall** 和 **bsddialog**，我设想可以重用现有代码库与架构，同时用图形界面执行安装流程。通过用功能等效且同样采用 BSD 许可证的工具替换 **bsddialog(1)**，可以在项目分配的几周时间内交付一个功能完善的安装镜像。

我本人对 Gtk+ 也较为熟悉，考虑到仅需做一个可用的演示，这是一种合理的折衷。Gtk+ 采用 LGPL 许可证，仍然与目标用户群的许可兼容性不完全吻合。但我还是在几天内成功实现了 **gbsddialog**，它是 **bsddialog** 的桌面应用等效版本。

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.43.22%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/freebsd_graphical_live_replacement.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.44.08%CE%93CPM.png)

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.44.23%CE%93CPM.png)

我对这个工具的最初设想略有不同，希望能做一些额外的改进或变通：

- 防止对话窗口在每次调用之间消失再重新出现，比如通过使用 GtkPlug 和 GtkSocket 小部件来实现。
- 在解决了上述问题后，使用 GtkAssistant 小部件来提升界面的美观和用户体验。
- **bsddialog** 中有一个特定的小部件 **mixedgauge**，其实现方式不适合用于图形化版本的工具：在 **bsdinstall** 中，通过滥用 **bsddialog** 控制台输出的持续存在感来营造进度感，实际上每次更新时都会执行新的 **bsddialog** 实例。在图形安装器中，这种监控只能用普通的 **gauge** 小部件替代，牺牲了部分可用性。

另一方面，我对桌面窗口的实现感到相当满意，它相当准确地复刻了原始 **bsddialog** 工具的外观与交互体验。尽管这部分代码目前还不够优雅，但它确实完成了任务。

### 修复与升级功能

去年，我有幸与一位非常积极的学生 Leaf Yen（你好！）共同指导谷歌编程之夏项目。该项目旨在为 FreeBSD 安装器引入新功能：通过可移动介质升级或修复现有系统。

项目成功完成，并在 GitHub 上提交了三个合并请求：

- \#1395，GSoC 2024 —— 改进安装器以支持修复和升级。
- \#1424，bsdinstall：在 live 环境中添加 pkg 安装支持。
- \#1427，bsdinstall：向安装菜单添加修复脚本。

我尚未将这些工作集成到图形版本安装器中，但我认为它们是现代安装器同样重要的组成部分。也请把这篇文章看作是对该项目的关注呼吁——一旦完善并合并，应该能同样适配当前安装器和图形版本。

### 作为网页应用的图形安装器

如果不提这个潜在的额外加分项，文章就不完整了。自 Gtk+ 3 起，Gtk+ 应用可以直接在网页浏览器内渲染，而不是仅限于电脑屏幕，这得益于 GDK Broadway 后端。

虽然我还没尝试过这个功能，但这似乎意味着可以把图形安装器版本变成一个网页服务器，从而实现无任何额外改动的网络安装 FreeBSD 系统！这显然是另一个潜在的轻松突破点。

### 可访问性

我想特别提到采用 Gtk+ 作为图形安装器演示器的另一个标准：可访问性。

虽然我自己也有一些残障状况，但尚未严重到让我深入尝试 Gtk+ 的可访问性特性。不过我知道它在这方面的能力，也明白对于部分需要辅助功能的人群，操作系统安装器的可用性十分关键。事实上，我听说过无论是私下还是会议上，都有人提出安装器需要支持辅助功能。

目前还有一个资助项目，旨在为 FreeBSD 安装器引入辅助功能，由 Alfonso Siciliano 负责调研。（再次问候！）

回到 Gtk+，它支持以下可访问性需求：

- 屏幕阅读器，如 Orca，用于转换语音或盲文。
- 完整的键盘导航。
- 标准小部件的无障碍行为。
- 高对比度主题或其它视觉增强主题。
- 文本和界面缩放。
- 插件系统以支持更多增强功能。

虽然我还没机会与有需求者一起实际体验这些功能，但我在开发 **gbsddialog** 期间始终牢记这一点，始终遵循核心 Gtk+ API。因此，该工具兼容 Gtk+ 2 和 Gtk+ 3。

在文章结尾前，我想提一下 DeforaOS 桌面环境。它是另一个轻量级桌面环境，大部分由一位个人开发（你好！），坦白说它的完成度和质量远未达到我追求的目标。它部分被打包到 FreeBSD 的 ports 中（感谢 Olivier！），我在此图形安装器演示中也使用了它——哪怕只是为了展示其能力。

不过，我很高兴提及它带有虚拟键盘程序，并且我成功将其与图形安装器版本配合使用。

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/Screenshot_2025-06-04_at_12.44.40%CE%93CPM.png)

基于本文所述的进展，我更新了 FreeBSD 功能对照表如下：

| 评估标准    | FreeBSD 14.2     |
| ------- | ---------------- |
| 语言支持    | 否                |
| 辅助功能    | 是（放大镜、屏幕键盘、高对比度） |
| 许可协议    | 否                |
| 步骤数     | 22               |
| 导航      | 否                |
| 优先提问    | 否                |
| 高级模式    | 是                |
| 步骤列表    | 否                |
| 时间估计    | 否                |
| Live 系统 | 是                |
| 修复系统    | 是                |
| 纯文本模式   | 是                |
| 图形模式    | 是                |
| 图形结果    | 混合（进行中）          |
| 自动化     | 是                |

## 结论

这确实是一个容易引发分歧且颇具主观性的议题。我希望本文对 FreeBSD 安装流程的改进有所助益，推动未来版本的优化。不幸的是，基于现有代码库工作并非易事，每次测试都需生成和启动安装介质，极为耗时且考验耐心。简单的改动可能无意中影响硬件支持或整体用户体验质量。

显而易见，FreeBSD 安装器还有大量改进空间。在工作期间，我也听到过彻底重写的呼声。但这既不是一个容易做出的决定，也不是一个轻松实现的项目——尤其要兼顾各种使用场景、满足各方需求，同时控制在合理预算内。

坦白说，我并不觉得自己有资格判断或强加某种方案。我更希望能提供并维护一项简单的、现有安装器的渐进式演进方案，在不引入不必要变动的前提下，扩展其能力。正如本文所述，通过复用现有代码，对基于控制台的安装器所做的 bug 修复和改进同样会体现在图形版本中，反之亦然。

## 参考文献

1. [MS DOS 6.22 MICROSOFT](https://archive.org/details/MS_DOS_6.22_MICROSOFT)
2. [Windows for Workgroups](https://archive.org/details/windows-for-workgroups)
3. [VirtualBox 相关讨论](https://forums.virtualbox.org/viewtopic.php?t=102121)
4. [Windows 95 OSR 2](https://archive.org/details/win-95-osr-2)
5. [Windows 98 SE ISO 文件](https://archive.org/details/windows-98-se-isofile)
6. [Windows 7 Professional with SP1 x64 DVD](https://archive.org/details/en_windows_7_professional_with_sp1_x64_dvd_u_676939_202302)
7. [FreeBSD 14.2 amd64 ISO 镜像](https://download.freebsd.org/releases/amd64/amd64/ISO-IMAGES/14.2/)
8. [FreeBSD 14.2 aarch64 ISO 镜像](https://download.freebsd.org/releases/arm64/aarch64/ISO-IMAGES/14.2/)
9. [Ubuntu Server 下载](https://ubuntu.com/download/server)
10. [Ubuntu Desktop 下载](https://ubuntu.com/download/desktop)
11. [Debian amd64 ISO 镜像](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/)
12. [Calamares 安装器框架](https://calamares.io/)

---

**Pierre Pronchery** 是 FreeBSD 基金会的安全工程师，自 2023 年 5 月起担任用户空间软件开发者。他于 2001 年安装了第一套 FreeBSD 系统，对操作系统设计与实现充满热情。
