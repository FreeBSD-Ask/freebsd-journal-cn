# 学会走路——连接 GPIO 系统

- [Learning to Walk–Interfacing to the GPIO System](https://freebsdfoundation.org/our-work/journal/browser-based-edition/downstreams/learning-to-walk-interfacing-to-the-gpio-system)
- 作者：Christopher R. Bowman

在上篇文章中，我们创建了简单的电路，使主板上的 LED 灯闪烁，并且学习了两种不同方法来把电路加载到 FPGA。遗憾的是，当我们加载电路时，CPU 停止运行了。此外，尽管这有点有趣，但并没有与芯片上的 CPU 进行交互。在本篇中，我们将深入了解 Vivado，学习如何在加载电路时保持 CPU 运行，并探索 FreeBSD 中的 GPIO 系统。

之前，当我们使用 U-boot 或者 xbit2bin 和 FreeBSD 下的 `/dev/devcfg` 时，我们发现 FreeBSD 停止运行了。我认为发生的问题是处理器的系统停止了运行。原来，我们使用的 FPGA.bit 文件并未包含处理器系统的配置信息。在本期中，我们将修复这个问题。

在如何呈现本期的内容上，我犹豫了一下。从学习的角度来看，最自然的方式可能是使用 Vivado 的 GUI（图形化界面）。然而，GUI 并不适合自动化，原因显而易见——它需要人工运行 GUI。此外，描述 GUI 步骤也既困难又繁琐。幸运的是，Vivado 有两个特性使得我们可以相对轻松地绕过这些问题。在使用 GUI 时，Vivado 工具会生成一个 `.jou` 文件，这是 GUI 在后台执行的所有 TCL 命令的日志。Vivado 还提供了一个 TCL 命令 `write_project_tcl`，可以用来重新创建使用 GUI 时 Vivado 创建的项目文件。我通常倾向于使用 `.jou` 文件，因为我发现这些脚本更简洁、易于理解，而且如果我运行这些脚本，我可以直接启动 GUI，或者使用 `write_project_tcl` 来写一个项目脚本。脚本似乎也更适合像 git 这样的版本控制系统。

如果我们查看“图 1-1：Zynq-7000 SoC 概述”（来自《UG585：Zynq-7000 SoC 技术参考手册》），我们可以看到有多种外设模块（如 UART、I2C、SPI 等），这些模块可以通过多路复用器连接到外部引脚。该手册的“1.2.3 I/O 外设”部分详细描述了更多的功能。出于我们当下的需求，我们只需要注意到，我们可以相当灵活地将 GPIO 设备的信号路由到芯片上的引脚。如果我们再查看 [Arty Z7 参考手册](https://reference.digilentinc.com/reference/programmable-logic/arty-z7/reference-manual?_gl=1*c286n6*_ga*MTg4NjczMDI1NC4xNzExMzUwMjY2*_ga_JSPEFFCPBT*MTcxMjM2NzMxNi4yLjAuMTcxMjM2NzMzMy40My4wLjA.) 的第 12 节“基本 I/O”，我们可以看到，板上的绿色 LED 与芯片引脚相连，这些引脚通过电流设置电阻接地。如果我们将这些引脚设为高电平，LED 灯就会亮起；反之，设为低电平时，LED 灯就会熄灭。

为了切换这些引脚，我们将使用 Vivado 软件将 GPIO 设备的输出路由到 LED 引脚，从而能让 GPIO 设备控制这些引脚。在开始这段旅程时我并不知道，但 FreeBSD 确实有一款 GPIO 子系统，且有人已经编写了驱动程序，使得这一切能从用户空间使用。

要开始，首先把 [git 仓库](https://github.com/christopher-bowman/zynq_gpio_leds) 克隆到安装了 Vivado 工具的 Linux 主机上（在之前的文章中我展示了如何设置 bhyve），然后在仓库根目录下输入 `make`。如果工具 Vivado 已经正确配置，并且一切顺利，它应该会运行 Vivado，并拉取一个脚本，这个脚本会实例化处理器子系统，并将 GPIO 设备的前四个 EMIO 引脚连接到 LED 引脚。包含处理器子系统将解决我们之前在用 bit 流加载设备时处理器停止运行的问题。

在运行 `make` 后，查找构建的 zynq_gpio_leds.bit 文件。使用之前介绍的 xbit2bin 程序将其编程到芯片中。

```sh
# xbit2bin zynq_gpio_leds.bit
```

你应该会看到什么也没有发生。虽然不是很激动人心，但至少处理器应该仍在运行。

现在，我们需要使用 FreeBSD 的 GPIO 子系统。输入 `man gpioctl` 会给出关于可用功能的简洁总结。

以 root 用户，我们可运行 `gpioctl` 程序来列出可用的引脚：

```sh
# gpioctl -f /dev/gpioc0 -l
```

没能成功吧？是的，我对此也有点惊讶。查看 `/usr/src/sys/arm/xilinx/zy7_gpio.c` 中的 GPIO 源代码，我看到驱动程序中有 probe 和 attach 函数，但查看我的 ARTYZ7 系统的 `dmesg` 输出，我没有看到任何表明设备已被发现的内容。仔细看一下 probe 函数：

```c
static int
zy7_gpio_probe(device_t dev)
{

        if (!ofw_bus_status_okay(dev))
                return (ENXIO);

        if (!ofw_bus_is_compatible(dev, "xlnx,zy7_gpio"))
               return (ENXIO);

        device_set_desc(dev, "Zynq-7000 GPIO driver");
        return (0);
}
```

可以看到，找到设备所需的唯一条件就是 `ofw_bus_is_compatible(dev, "xlnx,zy7_gpio")` 函数返回 `true`。

在嵌入式系统中，像大多数 ARM 系统一样，硬件通常不像现代 PCIe 总线那样是自识别的。软件无法自动识别硬件的存在以及其控制寄存器在内存地址空间中的位置。出于这个原因，许多操作系统使用 FDT（Flattened Device Trees，扁平设备树）来描述其设备内存映射。FDT 是文本文件，描述了嵌入式系统的信息，包括其他内容：如有哪些设备存在，以及它们在内存中的位置。这使得软件能够处理各种设备，而不需要硬编码信息。通过使用不同的 FDT，相同的内核通常可以与略有差异的设备兼容。FDT 是通过名为 dtc（设备树编译器）的工具将 DTS 文件（设备树源文件）编译成 DTB 文件（设备树二进制文件）。dtc 提供了选项，可以用来编译 DTS 或反编译 DTB。后者非常有用。例如，你可以请求内核返回它正在使用的 DTB，并使用 dtc 将其转换为文本：

```sh
# sysctl -b hw.fdt.dtb | dtc -I dtb -O dts
```

如果我们查找 gpio 部分，我们会看到（其中包括）以下内容：

```c
             gpio@e000a000 {
                  compatible = "xlnx,zy7_gpio";
                };
```

DTB 中的兼容字符串不是驱动程序所期望的，因此它假设设备不存在。如果你查看 FreeBSD 的启动输出，你会发现内核正在使用 U-boot 提供的 EFI 固件模拟的 DTB。我们可以通过修复 U-boot 提供的内容来更改这一点，但那将需要补丁和重新编译 U-boot。相反，FreeBSD 提供了在内核加载器提示符或使用 `loader.conf` 变量加载 DTB 文件的功能。你可以使用以下 `/boot/loader.conf` 变量从加载器加载 DTB 文件：

```sh
# fdt_name="/boot/dtb/zynq-artyz7.dtb"
# fdt_type="dtb"
# fdt_load="YES"
```

这可以工作，但还有第三种方法。事实证明，你可以创建 FDT/DTS 叠加文件，这些文件是对 FDT/DTS/DTB 的补丁。我们只需要一个添加正确兼容字符串的 DTS 叠加文件，这样就可以了。以下是我在项目仓库的 dts 目录中包含的 DTS 叠加文件的核心部分，并附带一个 Makefile：

```sh
&{/axi/gpio@e000a000} { compatible = "xlnx,zy7_gpio"; };
```

我们可能会在后续的专栏中更详细地查看 FDT 文件，所以这里我不做解释。而是给你此文件的概况。构建完 DTB 叠加文件后，将生成的 DTB 文件放入 `/boot/dtb/overlays`，并将以下内容添加到 `/boot/loader.conf`：

```ini
fdt_overlays="artyz7_gpio_overlay.dtb"
```

重新启动并注意新的 dmesg 输出：

```sh
gpio0: <Zynq-7000 GPIO driver> mem 0xe000a000-0xe000afff irq 5 on simplebus0
```

现在我们再次尝试运行 gpioctl 命令，你应该能看到类似如下内容的一行：

```sh
pin 64: 0 EMIO_0<IN>
```

我们需要告诉 GPIO 子系统将引脚配置为输出，然后可以尝试切换它：

```sh
gpioctl -f /dev/gpioc0 -c EMIO_0 OUT
gpioctl -f /dev/gpioc0 -t EMIO_0
```

我会等着。灯光亮起时，请尽量保持坐姿，不要起身跳舞。看看你能否弄明白如何打开其他 LED，然后运行脚本 scripts/blink.sh，并传入参数 2。你应该会看到 LED 按照二进制计数闪烁。

如果你对此有什么问题、评论、反馈和批评，我很乐意听到你的声音。你可通过 [articles@ChrisBowman.com](mailto:articles@ChrisBowman.com) 联系我。

---

在 1989 年，**Christopher R. Bowman** 首次在 VAX 11/785 上使用了 BSD，当时他在 Johns Hopkins University 应用物理实验室的地下二层工作。后来，在 90 年代中期，他在马里兰大学使用 FreeBSD 设计了他的第一款 2 微米 CMOS 芯片。从那时起，他一直是 FreeBSD 用户，并对硬件设计和驱动硬件的软件感兴趣。他在半导体设计自动化行业工作了 20 年。
