# 嵌入式 FreeBSD：Fabric——起步阶段

- 原文地址：[Fabric – Baby Steps](https://freebsdfoundation.org/our-work/journal/browser-based-edition/virtualization-2/embedded-freebsd-fabric-baby-steps/)
- 作者：**Christopher R. Bowman**

在之前的专栏中，我们简要地介绍了 Zynq 芯片，提及了它的结构。自那以后，我们对他没有太多讨论。但在上篇专栏中，我们成功地在 bhyve 上运行了 CentOS 镜像，现在是时候来看我们的第一个结构电路了。本篇专栏将专注于电路，不涉及 FreeBSD，但在接下来的几期中，我们将开始研究结合这两者的系统。

除了 [Zynq-7000 SoC 技术参考手册](https://docs.amd.com/r/en-US/ug585-zynq-7000-SoC-TRM/Zynq-7000-SoC-Technical-Reference-Manual)，该手册记录了构成 Arty Z7-20 功能核心的 Zynq 芯片外，Digilent 的 [Arty Z7 参考手册](https://digilent.com/reference/programmable-logic/arty-z7/reference-manual) 提供了大量关于 Zynq 芯片如何在板上连接的有价值的信息。查看第 12 节，我们可以看到该板上有 4 个 LED 直接连接到 ZYNQ 芯片（R14、P14、N16、M14）。如果我们将这些引脚设置为逻辑 1 或高电平电压，则这些引脚将源源不断地提供电流，电流通过 LED，再通过限流电阻流向地线，导致 LED 点亮。

既然我们刚刚开始，让我们尝试构建最小且最简单的电路来点亮这些引脚。最简单的电路就是将这些引脚静态地接高电平到结构中。好吧，我们怎么做呢？我们需要引入 Verilog 来实现这一点。

就像编程早期，程序是用机器码、汇编语言编写的，最终发展到使用像 C 这样的高级语言一样，集成电路最初也是手工绘制或用 Mylar 膜带布置的。随着电路规模的增大和 CAD 程序的发展，电路设计开始通过程序来完成——电子设计自动化（EDA）。原理图捕获程序可以通过图形化方式将设备（最初是晶体管，后来是门电路）连接起来，使用图形用户界面（GUI）进行设计。最终，出现了可以用文本描述电路的语言。我在 1.2 微米时代使用过一种早期的语言 SFL 来设计我前 3 个芯片中的两个。但在过去十年中，VHDL 和 Verilog 两种语言真正主导了芯片设计。这些语言在许多概念上类似于 Ada 和 C。Ada 和 VHDL 语言冗长，并具有强类型检查。它们都被指定为美国国防部工作的首选语言，尽管两者仍然存在，但它们在各自的领域中今天的受欢迎程度差不多。另一方面，Verilog 就像 C 一样，已经成为电路设计中最流行的语言。它不像 VHDL 那样强类型，就像 C 不像 Ada 那样强类型一样。总之，现如今，如果你想设计一个数字（非模拟）电路，超过几门门电路，你将使用 VHDL 或 Verilog，而大多数新的设计都使用 Verilog。

Verilog 中的电路设计会通过综合工具转化为电路，就像我们用编译器将 C 转换为可运行程序一样。这比编译还复杂一些。例如，除了将 Verilog 转换为实现设计的连接门电路外，在设计计算机芯片时，你还需要使用工具来放置门电路并布线。幸运的是，所有这些功能都包含在 Xilinx/AMD 的 Vivado 工具中。Xilinx/AMD 不仅提供 Vivado，而且它是免费下载安装的，你还可以免费获取一个许可证来解锁 Zynq 芯片的大部分功能。如果你想在你的 Arty Z7 开发板上做任何与编程以外的工作，你将需要下载并在 Linux/Windows 机器上安装此软件。在上一期专栏中，我们已经介绍了如何设置一个运行 Linux 的 bhyve 实例，以便在 FreeBSD 机器上的 bhyve 虚拟机中运行 Vivado。

现在，我们已经做好准备，让我们开始设计第一个简单电路来点亮板上的 LED。首先，我们需要 Verilog 描述来实现这个电路。我没法在一篇小文章中教会你 Verilog，因此我将展示设计并尽量介绍它的工作原理。

```c
module top(
    output [3:0]led
);

wire [3:0]led;

assign led = {1'b1, 1'b0, 1'b1, 1'b0};

endmodule
```

首先，设计是通过模块进行捕获的，本设计包含一个名为 `led` 的 4 位总线。这条总线是通过一组 4 个连接常量来驱动的。这些常量是 1 位信号，其中一半是逻辑“高”或 1，另一半是逻辑“低”或 0。我选择了交替的值，以便可以分辨 LED 的连接方式：从最高位到最低位（msb to lsb）或从最低位到最高位（lsb to msb）。这样，我就不必通过板上的标记和文档来推测连接方式。

接下来，我们需要提供一个约束文件。约束文件有两个主要功能：它们传达时序信息，并且在 FPGA 领域，它们还传达一些布置信息。在我们的第一个实验中，我们需要告诉工具哪些芯片引脚连接到 `led` 总线的线，并且还需要告诉工具如何设置我们希望使用的 IO 引脚。Digilent 非常贴心地提供了一个包含此板和其他许多板信息的主 XDC 文件，存放在一个 GitHub 仓库中。不幸的是，他们没有在文件或 readme 中包含版权声明，因此我无法在我的项目中包含它。这里提供了几个相关的行，如果你使用我在 [static_leds 仓库](https://github.com/christopher-bowman/static_leds) 中为本文提供的 make 文件，它会从 GitHub 下载该文件并取消注释相关行。你需要在系统中安装 GNU make 和 wget。

```c
set_property -dict { PACKAGE_PIN R14 IOSTANDARD LVCMOS33 } \
     [get_ports { led[0] }]; #IO_L6N_T0_VREF_34 Sch=LED0
set_property -dict { PACKAGE_PIN P14 IOSTANDARD LVCMOS33 } \
     [get_ports { led[1] }]; #IO_L6P_T0_34 Sch=LED1
set_property -dict { PACKAGE_PIN N16 IOSTANDARD LVCMOS33 } \
     [get_ports { led[2] }]; #IO_L21N_T3_DQS_AD14N_35 Sch=LED2
set_property -dict { PACKAGE_PIN M14 IOSTANDARD LVCMOS33 } \
     [get_ports { led[3] }]; #IO_L23P_T3_35 Sch=LED3
```

最后，我们需要运行 Vivado 将其转换为 BIT 文件，这是我们设计的电路的表示形式。遗憾的是，这并不像将这两个文件传递给 Vivado 那么简单。电路设计是一个复杂的过程，涉及许多选项。Vivado，像许多 EDA（电子设计自动化）工具一样，是一个基于 TCL 的工具，需要脚本来运行。在我的仓库中，我创建了最简单的 `GNUMakefile`，用于自动下载 XDC 文件，修补它并使用 TCL 脚本运行 Vivado。我建议你查看它，但如果你只是想继续进行，更新路径变量后，在 Linux 上运行简单的 `make` 命令，应该就能完成所有操作，并生成一个名为 `implementation/static.bit` 的文件，这是我们需要加载到 Zynq 芯片中的电路文件。

那么，我已经有了我的电路文件，接下来怎么做？U-boot 包含一个 FPGA 比特流加载程序，我们将从这里开始。把 `FPGA.bit` 文件复制到 SD 卡的 MSDOS 分区，将卡插入板中并按下重置按钮。在 U-boot 提示符下，打断启动过程并运行以下命令：

```c
Zynq> fatload mmc 0 0x4000000 static.bit
4045663 bytes read in 249 ms (15.5 MiB/s)
Zynq> fpga loadb 0 0x4000000 4045663
```

第一个命令将 `static.bit` 文件从 SD 卡的 FAT 分区加载到内存中。第二个命令告诉 U-boot 用当前在内存地址 `0x4000000` 中的文件内容来编程 FPGA。此时，你应该看到板上的四个 LED 中有两个亮起了红色。恭喜！你已经构建并加载了第一个 FPGA 设计！

我们可以在这里结束，但在我们结束之前，让我们再做两件事。首先，让我们让 LED 闪烁，而不仅仅是亮起；其次，让我们看看如何在 FreeBSD 下加载 FPGA。这将为更酷的事情打下基础。

为了让 LED 闪烁，我们需要通过更改 Verilog 来修改电路。这个新的电路有一个新的 [仓库](https://github.com/christopher-bowman/blinky_leds)，但我将在这里总结这些更改。首先是我们的新 Verilog：

```c
module top(
    input clk,
    output [3:0] led
);

localparam cycles_per_second = 125000000;

reg [3:0]leds;
reg [31:0]counter;

always @ (posedge clk)
begin
  if (counter == 0) begin
    leds <= leds + 1;
    counter <= cycles_per_second;
  end else counter <= counter - 1;
end

assign led = leds;

endmodule
```

我们现在添加了一个时钟输入，它将驱动一个 31 位计数器，并且我们还添加了一个 4 位计数器。31 位计数器加载值 123,000,000，并递减到零。当它达到零时，4 位的 `leds` 计数器递增。我们选择 125,000,000，因为根据 Arty 参考手册第 11 节，我们看到以太网 PHY 在 H16 引脚上提供了一个 125MHz 的时钟。

接下来，我们需要告诉 Vivado H16 引脚上的时钟：

```sh
set_property -dict { PACKAGE_PIN H16 \
     IOSTANDARD LVCMOS33 } \
     [get_ports { clk }]; #IO_L13P_T2_MRCC_35 Sch=SYSCLK
create_clock -add -name sys_clk_pin -period 8.00 \
     -waveform {0 4} [get_ports { clk }];#set
```

同样，一个简单的 `make` 命令应该会构建出一个 `implementation/blinky.bit` 文件，该文件可以像上面一样转移到 SD 卡并加载到 FPGA 中。

现在，你应该能看到 LEDs 按二进制计数闪烁。

好了，在我们结束本期专栏之前，让我们谈谈最后一件事。让我们看看如何从 FreeBSD 中编程 FPGA。事实证明，这很简单。有一个 `/dev/devcfg` 设备，最初是为了让你直接将 bit 文件 `cat` 到该设备上，但我认为对它的工作并没有完全完成。有一个简单的 C 程序 `xbin2bit`，它有自己的 [`git 仓库`](https://github.com/christopher-bowman/xbin2bit)。如果你有 root 权限（默认情况下 `/dev/devcfg` 是 root 所有），你可以直接运行它并传递你的 bit 文件：

```sh
# xbin2bit blinky.bit
```

你运行了这个程序并看到 FreeBSD 系统停止了，但 LEDs 仍然在闪烁吗？没错，事实证明我们的 Verilog 设计需要稍微复杂一点，以确保处理器不会停止。我们将在下期专栏中进一步探讨这个问题。

如果你对这些内容有任何问题、评论、反馈或批评，我非常乐意听取。你可以通过 [articles@ChrisBowman.com](mailto:articles@ChrisBowman.com) 联系我。

---

**Christopher R. Bowman** 在 1989 年首次使用 BSD 系统，当时他在约翰霍普金斯大学应用物理实验室的地下两层楼工作。他随后在 90 年代中期使用 FreeBSD 在马里兰大学设计了自己的第一块 2 微米 CMOS 芯片。从那时起，他一直是 FreeBSD 用户，并对硬件设计以及驱动硬件的软件感兴趣。在过去的 20 年里，他一直在半导体设计自动化行业工作。
