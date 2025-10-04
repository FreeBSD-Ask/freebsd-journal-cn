# 嵌入式 FreeBSD：回顾与展望

- 原文：[Embedded FreeBSD: Looking Back and Forward](https://freebsdfoundation.org/our-work/journal/browser-based-edition/embedded-2/embedded-freebsd-looking-back-and-forward/)
- 作者：Christopher R. Bowman

在过去一年中，我们已经探索了相当多的内容。虽然我猜大多数人运行 FreeBSD 还是在传统的 AMD64 架构 PC 上，但我们研究了一块可运行 FreeBSD 的嵌入式开发板：[Digilent Arty Z7-20](https://digilent.com/shop/arty-z7-zynq-7000-soc-development-board/)。虽然价格不算低，但 Arty Z7 提供了与 CPU 相连的 FPGA 结构，这使它区别于价格更低的树莓派或 Beagle Boards。

我们首先讨论了如何获取该开发板的预构建镜像，以及如何通过串口与其通信。在接下来的文章中，我们研究了如何自己构建镜像，并利用 FreeBSD 的交叉编译基础设施为 Arty 板上的 ARMv7 系统编译。这大大加快了开发速度。我们还讨论了如何定制 FreeBSD 的构建，并将其写入 SD 卡，从而生成我们自己的定制镜像。

在学会构建和定制镜像后，我们学习了如何设置 bhyve 实例来运行 AMD/Xilinx 的 FPGA 软件，以便实验 FPGA 结构电路。

当我们有了 Linux 实例后，就研究了构建电路并将其加载到 FPGA 结构中的基本流程。这里涉及了很多细节：我们必须用一种全新的硬件设计语言 **Verilog** 来创建电路；还要学习如何使用 AMD/Xilinx 工具将电路连接到芯片上的引脚，再通过这些引脚驱动开发板上的 LED。我们使用了一个代码仓库，把这些工作结合起来，最终能在 Linux bhyve 实例中构建电路。最后，我们学会了两种方式将电路加载到芯片中：一种是在系统启动前加载，另一种是在 FreeBSD 内加载。最终，我们看到了闪烁的 LED，就像圣诞树上的灯光。

在让第一个电路成功运行后，我们开始探索了更复杂的硬件，让 CPU 和 FPGA 结构电路之间进行通信。为此，我们使用了 FreeBSD 的 GPIO 系统。但一开始在镜像构建中发现 GPIO 不工作。我们调查了 GPIO 驱动的探测，发现系统缺少该驱动的原因是硬件在 **设备树二进制文件 (DTB)** 中没有描述。这引出了对 **扁平设备树 (FDT)** 文件的简短讨论，以及它如何描述许多嵌入式板子的硬件。我们学习了如何修改 FDT 文件，并用 **DTC (设备树编译器)** 编译生成 DTB，再让 FreeBSD loader 在内核启动前加载定制 DTB。完成这些步骤后，我们终于能在用户态调用 GPIO 系统来切换外部引脚，再次点亮 LED。

在最近的一篇文章中，事情变得更有趣。我们使用了一个 PMOD 模块：双七段数码管显示器。我们在 FPGA 结构中构建了电路，让它能在两个显示器上显示数值，并通过 AXI 总线向 CPU 提供寄存器接口。我们在 FDT 中添加了条目，描述硬件的寄存器接口，并编写了驱动来控制七段数码管的显示。最后，我们使用 Unix 的 **sysctl 框架** 作为用户态 API 来设置数码管的数值。

至此，我们已经能用 Verilog 设计电路并放到 Zynq 芯片的 FPGA 结构中；我们能构建通过 AXI 总线通信的寄存器接口，让 CPU 能方便地和定制硬件交互；我们能在内核中描述这些硬件，并编写驱动让 FreeBSD 系统与之交互；我们还能从用户空间与这些硬件交互。接下来会怎样呢？

在有了基本的电路构建与 FPGA 部署能力后，我们开始探索如何在 Zynq 芯片的硬件与 CPU 子系统之间进行通信。这开启了广阔的探索与实现空间，但也存在限制。其中一个限制是 **带宽和并发性**。虽然寄存器接口功能强大且灵活，但带宽有限。CPU 写寄存器的速度有限，尤其当它还要执行其他任务时。目前，我们的硬件带宽受限。对于七段显示器这样的应用很好用，但如果需要带宽密集型任务，它就不够了。

举例来说，视频显示。我们的 Arty 板有 HDMI 输出接口。寄存器接口或许能支持字符显示，但对位图图形就不够用了。一台 24 位色深、1280x720@60Hz 的显示器需要大约 **166 MB/s** 的数据吞吐。我们显然不可能通过寄存器接口来实现。传统的做法是分配一块内存，CPU 写入数据，显示硬件读取数据。我们需要探索如何构建硬件来直接访问主存，而不依赖 CPU 搬运数据。寄存器接口知识仍然有用，因为 CPU 需要配置参数（比如基地址），但理想情况是硬件能每秒 60 次自动抓取显示缓冲，而无需 CPU 干预。

让 CPU 能描述硬件应读写的内存对象，开启了 Zynq 系统设计的新可能性。这也让我好奇，这种带宽竞争会对双核 Arm Cortex A9 系统产生什么影响。Digilent 还生产另一款基于 Zynq 的板子，和 Arty Z7-20 类似：[Digilent Zybo Z7](https://digilent.com/shop/zybo-z7-zynq-7000-arm-fpga-soc-development-board/)。它比 Arty Z7 更贵（双核版 399 美元，而 Arty 是 249 美元），但 Zybo 的内存总线宽度是 Arty 的两倍，频率几乎相同。此外，Zybo 提供 **6 个 PMOD 接口**，而 Arty 只有两个。不过，Zybo 没有 Arduino 外壳引脚。我更感兴趣 PMOD 接口。除此之外，两者基于相同的芯片，不需要新驱动，FDT 基本不变。研究它需要的改动会很有意思。

另外，还可以研究新的 PMOD 模块。在 [Digilent 网站](https://digilent.com/shop/products/fpga-boards/expansion-modules/pmods/?CommunicationProtocol=GPIO) 上可以找到很多。我们之前用过 PMOD SSD：七段数码管。Digilent 已经下架了 [PMOD GPS](https://digilent.com/reference/pmod/pmodgps/start?srsltid=AfmBOorJwOUISt4P6L25EAjCmwiWvRD4LpNuHDajzb_vP1fvAtEdZxD0)，但我在下架前买了一块。它使用 UART 接口，而 UART 恰好是 Zynq 芯片上的片上外设，可以通过 FPGA 连接到外部引脚。这应该很容易连接。我猜有开源软件能通过 UART 与它通信，实现定位和授时等 GPS 功能。更有趣的是它提供 **每秒脉冲 (PPS)** 输出。我知道 Poul-Henning Kamp 过去做过一些基于 FPGA 的计时研究，我想看看这是否能应用在这里。

目前我们还没做过中断实验。但让 FPGA 电路生成中断传递给处理器并不难，同时保持一个寄存器来记录 PPS 与当前时间之间的时钟周期数。当中断服务软件运行时，可以读取该寄存器，补偿中断与驱动运行之间的延迟。这可能对 NTP 软件有用。我不敢确定，但这是我感兴趣的探索。甚至有可能实现一个本地 GPS 同步的一类时间服务器。

我手头还有各种 PMOD 模块，比如加速度计、OLED 显示器和 LCD。将它们接入也很有趣。例如，如果你在板子上运行一个 NTP 服务器（也许利用上面描述的硬件提高精度），你可以用 LCD 持续显示原子时间和位置。

差点忘了，Zynq 板子还内置了 **模数转换器 (ADC)**。这无疑是另一个有趣的探索方向，但可能需要一些外部模拟电路（如信号调理或缓冲）才能传给 FPGA，这可能超出 FreeBSD Journal 文章的范围。不过，研究如何访问这些片上外设也是很有意义的。

当然，你也可以自己构建硬件，并很容易通过 FPGA 引脚连接。我很好奇，如果你能自己设计，你会做什么？

还有一个我原本打算研究但没提到的方向：在 FreeBSD 上运行 Vivado。如果你看过我的一些代码仓库，可能会注意到 FreeBSD 下运行 Vivado 有一些实验性支持。不过看来有人已经先我一步了：Michał Kruszewski 写了一篇详细的[博文](https://m-kru.github.io/posts/freebsd-vivado-chroot/freebsd-vivado-chroot.html)。对我来说，这已经够用了，我能在 FreeBSD 上构建和模拟电路。但还不行的是直接从 FreeBSD 主机加载比特流，以及使用 Vivado Logic Analyzer。这两点在我的 bhyve Linux 实例里也不行，不过也许等 FreeBSD 15.0 发布时，我会尝试直通实验。

希望你觉得这些专栏有用。我很欢迎你的意见或反馈。你可以通过 [articles@ChrisBowman.com](mailto:articles@ChrisBowman.com) 联系我。

---

**Christopher R. Bowman** 最早在 1989 年使用 BSD，当时他在约翰·霍普金斯大学应用物理实验室地下一层的 VAX 11/785 上工作。后来在 90 年代中期，他在马里兰大学用 FreeBSD 设计了他的第一款 2 微米 CMOS 芯片。从那时起，他就一直是 FreeBSD 用户，对硬件设计和驱动它的软件都有着浓厚兴趣。他在半导体设计自动化行业已有 20 年经验。
