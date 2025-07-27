# 嵌入式 FreeBSD：自定义硬件

- 原文：[Embedded FreeBSD: Custom Hardware](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking-3/embedded-freebsd-custom-hardware/)
- 作者：Christopher R. Bowman

当我开始踏上 FPGA 和 FreeBSD 的这段旅程时，我选择了 [Digilent Arty Z7-20](https://digilent.com/shop/arty-z7-zynq-7000-soc-development-board/)，因为它是基于 Zynq 的开发板中价格较低且具有良好扩展性的型号。它既带有符合 Arduino Shield 物理规格的一组引脚，也配备了符合 [PMOD](https://en.wikipedia.org/wiki/Pmod_Interface) 标准的一组连接器。事后看来，我觉得我会选择放弃 Arduino Shield 接口（我其实从未使用过），换成更多的 [PMOD](https://en.wikipedia.org/wiki/Pmod_Interface) 接口。市面上有很多 [PMOD 设备](https://digilent.com/reference/pmod/start)，而且价格普遍较低。本文将使用其中的 [PMOD SSD](https://digilent.com/reference/pmod/pmodssd/start) 作为示例。

![](https://freebsdfoundation.org/wp-content/uploads/2025/07/bowman_Figure_1_PMOD_SSD.png)

图 1 PMOD SSD

这是一个双七段数码管显示器（SSD，这里指 Seven Segment Display，不要与固态硬盘 Solid-State Disk 混淆），可以插到 Arty Z7 开发板上的两个 PMOD 接口中。查看其[原理图](https://digilent.com/reference/pmod/pmodssd/reference-manual)，可以发现只需将几个引脚设置为高电平或低电平，就能控制点亮对应的 LED。但文档中有一句话提到：“由于任意时刻只能点亮一位数码管，想要同时显示两位数字的用户，需要以不低于 20 毫秒（50Hz）的频率交替点亮两个数码管。”也就是说，同一时间只能点亮其中一位。想要同时显示两个数字，必须每 20 毫秒切换 AA-AF 引脚的状态，并且切换 C 引脚。我还没尝试过，但我怀疑自己能否稳定地以 50 次每秒的速率调用 GPIO 系统。这个场景很适合用硬件来实现，电路并不复杂。我们可以用两个寄存器，配合一个计数器和多路复用器（MUX）以 50Hz 的频率切换，将寄存器连接到引脚。另一种简易方案是使用 14 个 GPIO 引脚，只把计数器和多路复用器做成硬件部分，这样比较快速简单。但最终，你会想做一个带有常规寄存器接口的硬件模块。于是，我们来构建一个简单的内存映射寄存器设备，并把它挂到 AXI 总线上。它将是 AXI 总线的简单应用，但同时能为更复杂的设计提供强大范例。因为这个设计，我们需要自己编写驱动程序，不过这同样是个很好的示例。

这个项目包含硬件、软件以及文档三部分。你可以用下面的命令下载代码（**译者注：已经 404 了**）：

```sh
# git clone http://github.com/axi_mm_ssd
```

现在的情况稍复杂，有些组件需在 Linux 机器或虚拟机上构建（FPGA 配置比特流），另一些则需要在 FreeBSD 上构建（其它部分）。我沿用了之前专栏介绍的 bhyve 配置来做 Linux 端，这样只需一台机器，无需每次修改硬件后都重启系统重新构建 FPGA。硬件部分应在 Linux 下构建（除非你成功让 Xilinx/AMD Vivado 工具通过 Linux 兼容层原生运行在 FreeBSD 上，如果你做到的话，请告诉我怎么弄的）。因为 Linux 下的标准构建工具是 GNU make，项目里有符合 GNU make 规范的 Makefile。你会发现硬件构建脚本相当复杂。

我一直在苦恼如何给 Vivado 的硬件构建写个好用的脚本。这次我在 GUI 里完成了大部分设计，用 Vivado 的命令 `write_project_tcl` 导出创建项目的脚本，然后对它进行了修改和扩展。虽然不完美，但目前看起来运行稳定。

借助 Vivado 的 GUI 和 IP 生成工具，我创建了一个简单的 AXI 从设备示例，它帮我生成了一个完整的 AXI4-Lite 内存映射从接口的 Verilog 代码。我在此基础上修改了设计，增加了计数器和多路复用器，并将引脚连通。之后，我用 GUI 将 AXI4-Lite 从设备连接到 AXI 总线，并用 `assign_bd_address` 指定了地址。你只需运行 `make`，脚本会自动完成这些操作。

我们先来看看将要构建的硬件。如果进入项目文档目录并用 `make` 构建文档，会看到一节介绍硬件接口。简要来说，我们要构建一个有三个寄存器的内存映射设备。第一个寄存器包含一个只读的常量值，用于唯一标识设备。这个设计很实用，尤其是在从零开始时，你不确定硬件或软件是否正确工作。如果设备不按预期工作，但内存里能看到那个神奇的标识值，至少说明设备驱动成功与硬件连接正常。我也在研究用 git 的哈希值和构建日期为设计打标签，这样每次电路加载到 FPGA 里时，就能清楚知道加载的是哪一版本。对于成熟商业厂商的成品板这不算大问题，但自己做设计、硬件软件都刚学的情况下，这极大方便调试和排错。不同于芯片制造，不用为每次硬件迭代花费五百万美元做掩膜版，我们可以承受这点额外开销，待全部调通后再去除。

另外两个寄存器分别控制数码管的两个数字，每个寄存器的每个位对应数字上的一个段。通过向寄存器的内存地址写入值，就能简单地点亮或关闭该数字的所有段。

既然我们已经讨论了硬件及其构建方式，而本文又刊载于《FreeBSD 期刊》，接下来让我们仔细看看软件部分。在项目的 `software/driver/freebsd/kld/ssd.c` 文件中，你会找到一个相当直观的 KLD（内核模块），它添加了几个 sysctl 条目，最终通过写入内存映射寄存器来驱动七段数码管显示。这段 KLD 代码是一个比较简洁的示例，类似于 Joseph Kong 的经典著作 [*FreeBSD Device Drivers: A Guide for the Intrepid*](https://www.amazon.com/FreeBSD-Device-Drivers-Guide-Intrepid-ebook/dp/B007W8OL0S/ref=sr_1_1?crid=3F1MA42Q4G4SN&keywords=FreeBSD+Device+Drivers&qid=1696841016&sprefix=freebsd+device+drivers%2Caps%2C224&sr=8-1) （《深入理解 FreeBSD 设备驱动程序开发》ISBN: 9787111411574）中提供的驱动示例。

不过，这个驱动和你在常见的 AMD64 PC 工作站及其标准 PCIe 总线上看到的驱动有些不同，有几个有趣的地方值得关注。下面是它的 probe 函数：

```c
static int
ssd_probe(device_t dev)
{


//   device_printf(dev, "probe of ssd\n");
     if (!ofw_bus_status_okay(dev))
          return (ENXIO);

     if (!ofw_bus_is_compatible(dev, "crb,ssd-1.0")){
          return (ENXIO);
   }

     //device_printf(dev, "matched ssd\n");
     device_set_desc(dev, "AXI MM seven segment display");
     return (BUS_PROBE_DEFAULT);
}
```

注意 `ofw_*` 系列函数，我们在上一期专栏中提到过它们，并讨论了如何用它们从 FDT/DTS 文件中读取属性。这次，我们将自己在 FDT overlay 中创建条目，来描述设备类型、地址以及寄存器集合。我们的 overlay 源代码位于 `software/driver/freebsd/kld/arrtyz7_ssd_overlay.dts`，其中比较有趣的部分如下所示：

```c
&{/axi} {
     axissd: ssd@043c00000 {
               compatible = "crb,ssd-1.0";
               reg = <0x43c00000 0x0004>;
          };
};
```

正如我们在上一期专栏中描述的，`compatible` 行用厂商名和版本号（以逗号分隔）来标识设备。在这个例子中，我是设备的设计者，所以用我的姓名缩写作为厂商名。这个项目基本完成，我也不太可能再修改它，但如果未来有了 V2 及后续版本，驱动程序就能通过区分不同的版本来适配可能不同的寄存器布局或编程方式。这样一个驱动就能支持多种类似设备，并根据 FDT 中的版本信息识别当前设备。

`reg` 这行告诉内核设备寄存器在物理地址空间的位置以及寄存器所占用的地址范围。这里设备被放置在物理地址 `0x43c00000`，共有四个寄存器。虽然接口只需要三个寄存器，且文档中也只描述了三个，但 Verilog 实际实现了四个寄存器。

接下来，我们看看它的 `attach` 函数，代码大致如下：

```c
static int
ssd_attach(device_t dev)
{
     struct ssd_softc *sc;

     device_printf(dev, "attaching ssd\n");
     sc = device_get_softc(dev);
     sc->dev = dev;

     int rid;

AXI_MM_SSD_LOCK_INIT(sc);

     /* 分配内存 */
     rid = 0;
     sc->mem_res = bus_alloc_resource_any(dev,
               SYS_RES_MEMORY, &rid, RF_ACTIVE);
     if (sc->mem_res == NULL) {
device_printf(dev, "Can't allocate memory for \
device\n");
          ssd_detach(dev);
          return (ENOMEM);
     }
#define MAGIC_SIGNATURE 0xFEEDFACE
#ifdef CHECKMAGIC
int32_t value = RD4(sc, AXI_MM_SSD_SGN);
     if (value != MAGIC_SIGNATURE) {
     device_printf(dev, "MAGIC_SIGNATURE 0xFEEDFACE \
not found! value = %x\n", value);
          ssd_detach(dev);
          return (ENXIO);
     }
#endif
   axi_mm_ssd_sysctl_init(sc);
   device_printf(dev, "ssd attached\n");

   return (0);

}
```

这段代码首先初始化了一个锁，稍后我会详细讲这个部分。接着，驱动程序分配了总线内存资源。我猜测设置 `rid=0` 表示请求总线 DMA 系统分配与设备相关联的第一个总线资源。虽然我没详细追踪过这段代码，但由于我们的 FDT overlay 里有 `reg = <0x43c00000 0x0004>;`，这个内存区域应该就是被分配的资源。如果有多条类似 `reg` 行，它们应该对应连续的资源 ID。如果你对此了解更深，欢迎告诉我。

因为这是我构建的第一个设备以及第一个为它写的驱动，我加了一段用 `#ifdef` 包裹的代码，用来读取第一个寄存器并查找我的“幻数”——`0xFEEDFACE`。文件里定义了一个宏 `RD4` 如下：

```c
#define RD4(sc, off) bus_read_4((sc)->mem_res, (off))
```

这个宏将通过 `bus_alloc_resource_any()` 获取的总线资源 `mem_res` 传给 `bus_read_4()`，并返回寄存器中对应偏移处的值。

借助总线资源系统，我们避免了在驱动里硬编码寄存器地址。这样如果我在设计中更改硬件地址，只需修改 FDT/DTS 即可，非常方便。此外，也避免了必须将物理地址转换成内核虚拟地址。正如我亲身经历过的，内核地址并不直接映射物理地址（而且第一个寄存器的魔数也帮我确认了这一点）。

通过这段代码，我可以比较确信，如果读出第一个寄存器返回了正确的魔数，就说明我确实在和我设计的设备通信。对于第一次使用整套工具链构建设备并将其挂载到 AXI 总线的新手来说，如果不了解内核物理地址和虚拟地址的区别，这种确认是非常安心的。如果我是个纯软件工程师，我甚至可能会尝试在 FDT/DTS overlay 里添加一个自定义资源，指定预期读取的魔数值，而不是硬编码 `0xFEEDFACE`。这很有用，不同的常量可以代表设备的不同版本或不同寄存器布局，这样驱动就能校验 FDT/DTS 是否和设备匹配。这里就留给读者作为练习（欢迎提交补丁）。

现在我们能对设备进行探测（probe）和附加（attach），也可以比较确定设备真实存在。接下来我们看看如何使用它。从用户空间看，我还不太清楚软件接口该怎么设计。Unix 哲学认为“一切皆文件”，但这个设备似乎不太像一个文件。我可以在 `/dev` 下创建一个设备节点，通过打开它并实现 ioctl，允许设置两个控制七段显示的寄存器值，这样还能用文件权限控制访问。但我选择了另一种类似的方案：没有使用文件系统设备节点，也没有 ioctl，而是用 sysctl。

在 `ssd_attach()` 返回前，它创建了两个 sysctl，一个用于获取和设置一个七段数码管的值，另一个用于另一个数码管的值。这两个 sysctl 会出现在 sysctl 树的 `dev.ssd.0.tens` 和 `dev.ssd.0.ones` 下。根据你板子放置方向不同，显示可能会有相反的感觉。

`axi_mm_ssd_sysctl_init()` 函数负责注册这两个 sysctl，下面是其中一个的代码示例：

```c
static int
axi_mm_ssd_proc0(SYSCTL_HANDLER_ARGS)
{
     int error;
     static int32_t value0 = 1;
     struct ssd_softc *sc;
     sc = (struct ssd_softc *)arg1;

     AXI_MM_SSD_LOCK(sc);

     value0 = RD4(sc, AXI_MM_SSD_SR2);

     AXI_MM_SSD_UNLOCK(sc);

     error = sysctl_handle_int(oidp, &value0,
                   sizeof(value0), req);
     if (error != 0 || req->newptr == NULL)
          return (error);

     WR4(sc, AXI_MM_SSD_SR2, value0);

     return (0);
}
```

这段代码使用了我们在 `attach` 函数中创建的锁，并通过前面提到的 `RD4` 宏读取第一个数码管对应的设备寄存器。虽然我这里只做单次读写操作，不确定是否真的需要加锁，但我用锁是为了确保多个竞争进程在设置寄存器时不会互相干扰。也许我应该把整个过程都包裹在锁中，但我对内核编程还很新，不确定哪些函数在持锁状态下是安全调用的（如果你知道，欢迎告诉我）。`sysctl_handle_int` 会读取旧值、传回用户空间，并返回新的值写入寄存器，写操作由 `WR4` 完成。John Baldwin 写过一篇关于 sysctl 系统的优秀[文章](https://freebsdfoundation.org/wp-content/uploads/2014/01/Implementing-System-Control-Nodes-sysctl.pdf)，如果你想了解 sysctl 的工作原理，推荐阅读。

现在我们有了能探测、附加和设置两个硬件寄存器的驱动，只需构建包含该驱动的 KLD，构建 FDT overlay，安装到 `/boot/dtb/overlays`，并像上一篇文章中那样把它接入 `loader.rc` 脚本。重启系统，启动后先加载 FPGA 的 bitstream，再加载 KLD，`dmesg` 应该会显示探测成功的信息。

这就是驱动的所有关键部分。我们来看看它实际运行的效果。在 Git 仓库里，你会看到一个位于 `software/app/test.csh` 的简短 C shell 脚本，内容如下：

```sh
#!/bin/tcsh
set digits = (126 48 109 121 51 91 31 112 127 115)
set delay = $argv[1]
foreach tens ($digits)
  sysctl dev.ssd.0.tens=$tens
  foreach ones ($digits)
    sysctl dev.ssd.0.ones=$ones
    sleep $delay
  end
end
```

这里有一个数组，里面的值对应着需要设置到寄存器中的位，用来编码数字 0 到 9。脚本接受一个延迟参数，从零计数到 99，每次数字变化后都会暂停。

我花了大量时间看着那个数码管计数。虽然这个设备本身用处不大，但它完整地展示了如何通过内存映射寄存器与硬件通信，并让芯片外的世界发生变化。

希望这些专栏对你有所帮助，欢迎留言反馈。你可以通过 [articles@ChrisBowman.com](mailto:articles@ChrisBowman.com) 联系我。

---

**Christopher R. Bowman** 最早于 1989 年在约翰霍普金斯大学应用物理实验室工作期间使用 BSD，当时是地下两层的 VAX 11/785。90 年代中期，他在马里兰大学用 FreeBSD 设计了他的第一个 2 微米 CMOS 芯片。从那以后，他一直是 FreeBSD 用户，对硬件设计及驱动它的软件很感兴趣。过去 20 年他一直在半导体设计自动化行业工作。

