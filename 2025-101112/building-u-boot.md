# 嵌入式 FreeBSD：构建 U-boot

- 原文：[Embedded FreeBSD: Building U-boot](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/building-u-boot/)
- 作者：Christopher R. Bowman

一位读者写信告诉我，他在编译 U-boot 时遇到了困难，所以我想走一遍这个过程，因为我也打算启动另一块 Zynq 开发板，反正也得经历这一流程。我需要声明：下面所写内容是准确的，但这些系统很复杂，我可能在某些细节上有误。如果你发现我写错了，欢迎指正。

正如我们之前讨论的，[U-boot](https://u-boot.org/) 是第二阶段和第三阶段的引导加载程序，它运行后会加载 FreeBSD 的加载器，而 FreeBSD 加载器则负责加载 FreeBSD 内核本身。U-boot 是个开源社区的项目，广泛用于各种系统以提供启动服务。相关文档可通过其官网获取：[U-boot 文档](https://docs.u-boot.org/en/latest/index.html)。

在 AMD/Xilinx Zynq 芯片上，第一阶段引导加载程序存放在 Zynq 芯片本身的 BootROM 中。Zynq 的启动流程可参考 [Zynq 7000 SoC 技术参考手册](https://docs.u-boot.org/en/latest/index.html) 第 6 章 “Boot and Configuration”。简要描述是：设备上电时，会采样一些引脚，根据其状态选择多种启动方式之一。这允许开发板上的跳线（根据 [Zybo Z7 参考手册](https://digilent.com/reference/programmable-logic/zybo-z7/reference-manual?redirect=1) 图 2.1 的 JP5）选择启动方式，其中之一是从 SD 卡启动。如果选择这种方式，BootROM 代码会在 SD 卡的 FAT16 或 FAT32 分区上寻找名为 `boot.bin` 的文件。这就是 U-boot 所称的二级程序加载器（SPL）。Zynq 芯片包含少量板载 RAM，因此 BootROM 能加载的程序大小受到限制。在非裸机应用中，SPL 必须包含足够的代码来配置 Zynq 芯片（PLL、内存接口等），以便启动内存系统并加载完整的 U-boot。完整的 U-boot 是功能更丰富的版本（支持文件系统等）。

我们非常幸运，因为 Zybo Z7 已经在 U-boot 中得到支持。所以，我们只需要让编译流程能够正常运行即可。U-boot 通常是在主机系统上为目标系统构建的。前面提到的 U-boot 文档建议，我们应通过环境变量来设置用于交叉编译的编译器。我们将使用 GCC 编译器和 GNU 工具进行交叉编译，因此需要安装这些软件包并设置交叉编译器。U-boot 还使用 `gmake` 构建，而不是标准的 BSD `make`，因此我们需要安装 `gmake`，以及其他一些软件包：

```sh
pkg install gmake
pkg install arm-none-eabi-gcc
pkg install bison
pkg install gnutls
pkg install gmake
pkg install pkgconf
pkg install coreutils
pkg install dtc
pkg install gdd

setenv CROSS_COMPILE arm-none-eabi-
```

奇怪的是，你使用的是 `arm-none-eabi-` 而不是 `arm-none-eabi-gcc`，但这不是笔误。

接下来，我们需要为目标板配置 U-boot 源代码树。Zybo Z7 开发板由 `configs` 目录下的 `xilinx_zynq_virt_defconfig` 支持。该配置支持多块板，其中就包括 Zybo Z7。要配置源代码树，我们运行：

```sh
make xilinx_zynq_virt_defconfig
```

但我们必须注意使用 GNU `make`，而不是 BSD `make`。为此，我创建了一个目录，其中有一个名为 `make` 的符号链接指向 `/usr/local/bin/gmake`，并将该目录放在我的 `PATH` 的最前面。这种方法运行得很好。

接下来，我们只需调用 `make` 并等待（如果你有多余的 CPU 核心，我强烈建议使用 `-j` 参数）。它像我一样报错了吗？

我得到的输出如下：

```sh
make[1]: *** [scripts/Makefile.xpl:257: spl/U-boot-spl-align.bin] Error 1
make: *** [Makefile:2358: spl/U-boot-spl] Error 2
make: *** Deleting file 'spl/U-boot-spl'
```

`scripts/Makefile.xpl` 中的相关行如下：

```sh
$(obj)/$(SPL_BIN)-align.bin: $(obj)/$(SPL_BIN).bin
        @dd if=$< of=$@ conv=block,sync bs=4 2>/dev/null;
```

如果你去掉输出重定向到 `/dev/null`，你会看到 `dd` 的报错信息：

```sh
dd: record operations require cbs
```

看来 FreeBSD 的 `dd` 与 GNU 版本在命令行上并不完全兼容。最初，我只是通过安装 GNU 版本的 `dd` 并在本地 `bin` 目录创建符号链接来使用，但实际上你只需从 `dd` 命令中删除 `block` 即可。

此外，可以通过设置 `V` 这个 `make` 变量来控制构建输出的详细程度。如果你的构建失败，我强烈建议使用单核并设置 `V=1` 再次运行：

```sh
make V=1
```

如果一切顺利构建完成，你应该会得到一个 `U-boot.img` 文件和一个 `spl/boot.bin` 文件。它们分别是完整的 U-boot 和二级程序加载器。将它们复制到 SD 卡上，然后尝试启动吧！

等等，没成功？嗯！正如我之前所说，这个配置支持多块开发板，而默认的设备树并不是针对 Zybo Z7 的。参考前面提到的板级文档，我们可以通过设置 `DEVICE_TREE` 来指定默认的设备树：

```sh
setenv DEVICE_TREE zynq-zybo-z7
```

这会覆盖配置文件中的默认 DTS。重新构建并尝试启动。等等，怎么又出了问题？内核可以加载，但在探测硬件时崩溃？哦，对。FreeBSD 对 DTS 的要求与 Linux 不同。某些硬件识别所需的 `compat` 字符串不同，而且 FreeBSD 似乎要求一些 `clock-frequency` 属性，虽然我不确定这些值是否被实际使用。或许在 FreeBSD 驱动中添加与 Linux 期望一致的 `compat` 值是合理的，但我并不是提交者。我不得不在 DTS 文件 `arch/arm/dts/zynq-zybo-z7.dts` 中添加以下内容：

```sh
&sdhci0 {
        compatible = "arasan,sdhci-8.9a", "xlnx,zy7_sdhci";
   U-boot,dm-pre-reloc;
        status = "okay";
};

&devcfg {
compatible = "xlnx,zynq-devcfg-1.0", "xlnx,zy7_devcfg";
status = "okay";
};

&global_timer {clock-frequency = <50000000>;};
&ttc0 {clock-frequency = <50000000>;};
&ttc1 {clock-frequency = <50000000>;};
&scutimer {clock-frequency = <50000000>;};
```

既然我们已经学会了如何构建 U-boot，现在看看是否可以将其做成一个 Port。U-boot 有一整套 Ports，它们都是基于 Port `U-boot-master` 构建的。要使用这些 Ports，我们需要包含 master Port 的 Makefile。我们必须指定开发板、型号以及应使用的配置。对于上面所做的修改，我们还有一些补丁，最终可以得到如下内容。

```sh
MASTERDIR=      ${.CURDIR}/../U-boot-master

MODEL=          zybo-z7
BOARD_CONFIG=   xilinx_zynq_virt_defconfig
FAMILY=         zynq_7000

EXTRA_PATCHES=  ${.CURDIR}/files

BUILD_DEPENDS+= gdd:sysutils/coreutils

COMMENT=        ported by Christopher R. Bowman <my_initials>@ChrisBowman.com

.include "${MASTERDIR}/Makefile"
```

希望这些专栏对你有所帮助。欢迎提出你的评论或反馈，你可以通过 [articles@ChrisBowman.com](mailto:articles@ChrisBowman.com) 与我联系。

---

**Christopher R. Bowman** 最早在 1989 年在约翰斯·霍普金斯大学应用物理实验室地下两层的 VAX 11/785 上使用 BSD。90 年代中期，他在马里兰大学设计第一颗 2 微米 CMOS 芯片时使用了 FreeBSD。从那时起，他一直是 FreeBSD 用户，并对硬件设计及其驱动的软件感兴趣。在过去 20 年里，他一直从事着半导体设计自动化行业的工作。
