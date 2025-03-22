# 如何为 FreeBSD 实现简单的 USB 驱动程序

- 原文链接：[How to Implement Simple USB Driver for FreeBSD](https://freebsdfoundation.org/wp-content/uploads/2021/11/Simple_USB_Driver_for_FreeBSD.pdf)
- 作者：**MARIUSZ ZABORSKI**

你是否曾经为你的 FreeBSD 系统购买了新设备，但结果发现某些功能无法正常工作？与其退货，你或许可以不费吹灰之力编写自己的驱动程序。本文将解释如何为 FreeBSD 编写一款简单的 USB 驱动程序。

## 案例研究

在本文中，我们将研究一个针对 Razer Ornata V2 键盘的驱动程序。该设备是一款机械膜键盘，在 FreeBSD 上运行得非常完美，唯一的小问题是：你无法更改背光颜色。在某些情况下，你可能会遇到一款具有内置颜色变化功能的键盘。这意味着颜色会根据你机器上运行的软件在某些按键组合下自动变化。对于这款键盘而言，操作系统中的驱动程序控制着背光。因此，你可以在键盘上显示炫酷的图案，比如火焰效果。缺点是，你需要为其安装驱动程序。该设备如图 1 所示。

**图 1. Razer Ornata V2**

![](https://github.com/user-attachments/assets/c6a18da9-a020-47de-b791-6291330b18fe)

## 收集信息

首先，我们需要了解驱动程序使用的协议。对于 Razer 驱动程序，有两种方式可以做到这一点：

* 查看 openrazer（一个非官方的 Razer 设备 Linux 驱动集）

* 从 Windows 驱动中嗅探 USB 协议

在本文中，我们将结合这两种方法。当我们最初研究这个问题时，openrazer 并不支持 Razer Ornata V2，因此我们必须从 USB 协议转储中推测出一些部分。最近，openrazer 已经添加了对该键盘的支持，但在编写驱动程序时，可能只能在官方的 Windows 驱动程序中找到部分内容，而无法在其他地方获得。为了教育目的，我们将假设 openrazer 不支持此键盘。

## OpenRazer

为了获得有关驱动程序所需的上下文，我们将尝试查找用于与键盘通信的包结构，因为这可以帮助我们理解来自 USB 嗅探的转储数据。可以在 https://github.com/openrazer/openrazer 上找到 openrazer 的源代码。在文件 `driver/razercommon.h` 中，我们将找到一个 `razer_report` 结构，这是驱动程序的主要结构。它在该产品的所有设备中都有使用。该结构如清单 1 所示。

**清单 1. openrazer 定义的 `razer_report` 结构体**

```c
struct razer_report {
 unsigned char status;
 union transaction_id_union transaction_id; /* */
 unsigned short remaining_packets; /* 大端 */
 unsigned char protocol_type; /*0x0*/
 unsigned char data_size;
 unsigned char command_class;
 union command_id_union command_id;
 unsigned char arguments[80];
 unsigned char crc;/* 报告的字节异或结果 */
 unsigned char reserved; /*0x0*/
};
```
## 嗅探 Windows 驱动程序

要嗅探 Windows USB 驱动程序，我们可以使用工具 usbpcap（https://desowin.org/usbpcap/）。它是一款命令行工具，非常易于使用（在清单 2 中，我们提供了一个示例）。当我们运行该命令工具时，它会显示可用的设备；接下来，它会询问我们选择要嗅探的设备以及保存 pcap 文件的位置。生成的 pcap 文件可以使用 Wireshark 轻松查看。

我们将针对 Razer Windows 驱动程序。在 Windows 上，Razer Synapse 工具允许你自定义键盘的背光颜色。让我们尝试在 usbpcap 运行时设置键盘的不同颜色。通过这个工具，我们将记录发送到键盘的所有请求（Razer Synapse 如图 2 所示）。此时，我们将整个键盘设置为红色方案。

以下可用的过滤控制设备：

```
1 \\.\USBPcap1
 \??\USB#ROOT_HUB20#4&19d0fd2a&0#{f18a0e88-c30c-11d0-8815-00a0c906bed8}
   [Port 1] Generic USB Hub
 [Port 4] ThinkPad Bluetooth 4.0
 [Port 6] Integrated Camera

2 \\.\USBPcap2
\??\USB#ROOT_HUB20#4&182122df&0#{f18a0e88-c30c-11d0-8815-00a0c906bed8}
 [Port 1] Generic USB Hub

3 \\.\USBPcap3
\??\USB#ROOT_HUB30#4&23ace5cb&0&0#{f18a0e88-c30c-11d0-8815-00a0c906bed8}
 [Port 1] Generic USB Hub
 Razer Ornata V2
   Razer Ornata V2
     Razer Ornata V2
    Razer Ornata V2
       Razer Control Device
Select filter to monitor (q to quit): 3
Output file name (.pcap): t1.pcap
```

## 结合方法

现在我们已经从转储中获取了一个 pcap 文件，可以开始分析记录的协议了。不要过于详细地研究 USB 驱动程序的工作原理；而要获取关于协议的基本思路。我们直接将大部分的值复制，因为可能需要进行更改。我们只需要生成与原始驱动程序相似的请求。

在图 3 中，我们可以看到使用 usbpcap 在 Wireshark 中创建的转储；在这种情况下，驱动程序使用了一个设置数据包。设置数据包用于 USB 设备的检测和配置。在表 1 中，我们可以看到一个由 USB 规范定义的数据包，以及驱动程序发送的值。

**图 2. Razer Synapse 工具。该工具用于配置背光颜色。**

![](https://github.com/user-attachments/assets/dd601b4c-a292-403b-b851-30d1555bc00a)


**表 1. 来自 USB 文档的设置数据格式，以及转储中的值**

![](https://github.com/user-attachments/assets/ebf0e691-eb81-4a9d-8529-47ed23bd16bf)

在设置数据之后，我们有专门针对 Razer 驱动程序的数据。在图 4 中，我们将 pcap 数据与 openrazer 项目中的 razer_report 结构结合起来。接下来，我们可以轻松地看到一些关于参数的更多信息。首先，我们有 2 个字节设置为 0，我们可以假设它们是保留字段。接下来，我们有一个字节设置为 1。当我们查看 pcap 时，可以看到许多类似的数据包，在此位置，值的范围从 0 到 5。我们稍后可以验证这一点，但实际上，它似乎是键盘上的行号。然后，我们有一个值 0x15（21），它实际上是每行的键数。最后，有一个重复了 21 次的值 0xff0000，似乎指的是我们在 RGB 中设置的红色（R: 255, G: 0, B: 0）。

**图 3. 使用 usbpcap 在 Wireshark 下生成的 pcap。设置数据包被高亮显示**

![](https://github.com/user-attachments/assets/059d3241-e726-49ba-9773-8e87d89a880f)

**图 4. 使用 openrazer 获取的结构与设置数据结合**

![](https://github.com/user-attachments/assets/5e2ce1cf-0fbc-427c-924d-d801b7c5f603)

**图 4. 使用 openrazer 获取的结构与设置数据结合**

```sh
# usbconfig dump_device_desc
ugen0.4: <Razer Razer Ornata V2> at usbus0, cfg=0 md=HOST spd=FULL (12Mbps)
pwr=ON (500mA)
 bLength = 0x0012
 bDescriptorType = 0x0001
 bcdUSB = 0x0200
 bDeviceClass = 0x0000 <Probed by interface class>
 bDeviceSubClass = 0x0000
 bDeviceProtocol = 0x0000
 bMaxPacketSize0 = 0x0040
 idVendor = 0x1532
 idProduct = 0x025d
 bcdDevice = 0x0200
 iManufacturer = 0x0001 <Razer>
 iProduct = 0x0002 <Razer Ornata V2>
 iSerialNumber = 0x0000 <no string>
 bNumConfigurations = 0x0001
```

## libusb&PyUSB

实现一个简单驱动的第一种方法是使用 libusb 和 PyUSB。这种方法允许我们在用户空间编写 USB 驱动，而无需额外的内核模块。在用户空间编写驱动是最安全的，因为如果出现 bug，只会暴露内核部分，攻击面较小。

libusb 是一款用于 USB 设备的跨平台库，因此我们可以在 FreeBSD、Linux、OpenBSD 甚至 Windows 上看到它的移植。为了进一步简化任务，我们可以不使用 C 语言编写驱动，而是通过 Python 实现，这要感谢 PyUSB 模块。PyUSB 提供了对主机 USB 系统的简便访问。我们可以使用 pkg(8) 工具简单地安装 pyusb（例如：`pkg install py38-pyusb`）。

首先，我们需要找到一个有效的设备。为此，我们使用函数 `usb.core.find` 。为了识别正确的设备，我们可以提供通过 `usbconfig(8)` 获取的产品和供应商 ID，如 Listing 4 所示。

**Listing 4. 使用 PyUSB 查找设备**

```python
# python
Python 3.8.10 (default, Jul 6 2021, 01:34:57)
>>> import usb.core
>>> dev = usb.core.find(idVendor=0x1532, idProduct=0x025d)
>>> dev.product
'Razer Ornata V2'
```

要发送一个 Setup 数据包，我们使用 `ctrl_transfer` 函数。该函数的接口与 Setup 数据包中描述的参数相对应。这里最简单的做法是复制所有嗅探到的参数。最后一步是重新构建数据包。在我们的驱动中，我们假设颜色是硬编码的。除了颜色、行和 CRC 字段之外，我们将从嗅探到的部分复制所有数据（整个过程如 Listing 5 所示）。最后，我们还需要重新计算 CRC 字段。

**Listing 5. 使用 PyUSB 发送请求更改颜色。**

```python
import usb.core
# 颜色
r = 0xff
g = 0x00
b = 0x00
def change_color(dev, line, r, g, b):
    # 重新创建数据包
    package = bytes([
        0x00, # 状态
        0x1f, # 事务 ID
        0x00, 0x00, # 剩余数据包
        0x00, # 协议类型
        0x47, # 数据大小
        0x0f, # 命令类别
        0x03, # 命令 ID
        # 参数：
        0x00, # - 未知
        0x00, # - 未知
        line, # - 行
        0x00, # - 未知
        0x15, # - 键位数量
        0x00, 0x00, # - 未知
        0x00 # - 未知
    ])
    for _ in range(0x15):
        package += bytes([r, g, b])
    # 填充至 0x47 字节大小
    for _ in range(0x3):
        package += bytes([0, 0, 0])
    # 重新计算 CRC
    crc = 0x00
    for x in package:
        crc ^= x
    package += bytes([crc, 0x00]) # CRC 和保留字段
    dev.ctrl_transfer(
        bmRequestType = 0x21,
        bRequest = 0x09,
        wValue = 0x300,
        wIndex = 0x02,
        data_or_wLength = package
    )

dev = usb.core.find(idVendor=0x1532, idProduct=0x025d)
for line in range(6):
    change_color(dev, line, r, g, b)
```

## 内核模块

在使用原生驱动程序的情况下，我们需要编写一个 FreeBSD 内核模块。我们还需要在内核与用户空间之间实现某种通信，以告知模块我们想要的颜色。为了实现这一点，我们可以暴露一些额外的 sysctl，实施一个 ioctl(9) 或从 USB 设备节点读取输入。在本文中，我们将讨论 ioctl(9) 方法。

### 编译内核模块

首先，我们需要知道如何编译内核模块。最简单的方法是使用 Makefile，并包含 bsd.kmod.mk 文件。通过这种方式，它将自动生成所有其他所需的文件和头文件。我们还需要记住包括一些文件，比如 `opt_usb.h`、`buf_if.h` 和 `device_if.h`，这些文件对于所有内核模块来说都是通用的。在 KMOD 调试工具中，我们提供编译后驱动程序的名称。Makefile 的示例如 Listing 6 所示。

**Listing 6. 在 FreeBSD 中构建内核模块的 Makefile。**

```c
SRCS=ornata.c
SRCS+=opt_usb.h bus_if.h device_if.h
KMOD=ornata
.include <bsd.kmod.mk>
```

几乎所有驱动程序都必须实现的三种标准方法是 probe、attach 和 detach。还有一些额外的方法，如 suspend 和 resume，但我们不在这里讨论它们。

probe 方法首先执行，用于检查设备并决定是否支持该驱动程序。在这里，我们可以使用 VendorID 和 ProductID 来决定是否是我们要找的设备。我们可以通过使用函数 `usbd_lookup_id_by_uaa` 来实现，它将遍历给定的厂商和产品数组，找到匹配的对。我们还需要检查设备是否处于主机模式（USB_MODE_HOST），因为这对于启动数据传输是必要的。接下来，我们要确保该设备确实是一个键盘。probe 方法的代码如 Listing 7 所示。

**Listing 7. USB probe 函数**

```c
static const STRUCT_USB_HOST_ID ornata_devs[] = {
{USB_VPI(0x1532, 0x025d, 0)},
};
static int
ornata_probe(device_t self)
{
struct usb_attach_arg *uaa = device_get_ivars(self);
if (uaa->usb_mode != USB_MODE_HOST)
 return (ENXIO);
if (uaa->info.bInterfaceProtocol == UIPROTO_BOOT_KEYBOARD)
 return (ENXIO);
if (uaa->info.bInterfaceClass != UICLASS_HID)
 return (ENXIO);
return (usbd_lookup_id_by_uaa(ornata_devs, sizeof(ornata_devs), uaa));
}
```

另两个有用的方法是 attach 和 detach。attach 函数在 probe 阶段完成并且 probe 函数返回成功时被调用。它是一个入口点，允许驱动程序初始化所有所需的资源。与此相对，我们有一个 detach 函数，允许我们在设备消失后进行清理。

在这种情况下，驱动程序在 attach 函数中将初始化用于同步的互斥锁，并分配 USB 驱动程序的入口点，通常在 `/dev` 下。最后的部分由 `usb_fifo_attach` 函数完成。在创建新节点时，我们还需要定义它支持的操作（变量 `ornata_fifo_methods` ），但我们将在后续阶段中讨论。创建节点时，我们可以定义哪个用户和组应该是所有者（在我们的例子中是 root(0) 和 wheel(0) 组），以及该节点应以什么模式初始化（在我们的例子中是每个人都可以读写（666））。此时，我们还引入了一个辅助结构，存储所有设备特定的变量。与此相对，在 detach 例程中，我们调用函数 `usb_fifo_detach`，它销毁其关联的 USB 设备节点。这些函数的代码如下所示，见 Listing 8。

**Listing 8. 驱动程序的 attach 和 detach 函数**

```c
struct ornata_softc {
	struct usb_fifo_sc	sc_fifo;
	struct mtx		sc_mtx;
	struct usb_device	*sc_udev;
};
static int
ornata_attach( device_t self )
{
	struct usb_attach_arg	*uaa	= device_get_ivars( self );
	struct ornata_softc	*sc	= device_get_softc( self );
	int			unit	= device_get_unit( self );
	int			error;
	device_set_usb_desc( self );
	mtx_init( &sc->sc_mtx, " ornata lock ", NULL, MTX_DEF );
	error = usb_fifo_attach( uaa->device, sc, &sc->sc_mtx,
				 &ornata_fifo_methods, &sc->sc_fifo,
				 unit, -1, uaa->info.bIfaceIndex,
				 0, 0, 0666 );
	if ( error )
		goto detach;
	sc->sc_udev = uaa->device;
	return(0);
detach:
	mtx_destroy( &sc->sc_mtx );
	return(error);
}


static int
ornata_detach( device_t self )
{
	struct ornata_softc *sc = device_get_softc( self );
	usb_fifo_detach( &sc->sc_fifo );
	mtx_destroy( &sc->sc_mtx );
	return(0);
}
```

最终，我们可以定义驱动程序模块，如列表 9 所示。我们使用 `DRIVER_MODULE` 宏来创建一个内核驱动程序。在这一部分，我们将 `probe`、`attach` 和 `detach` 函数设置到驱动程序结构中。`MODULE_DEPEND` 宏用于设置对另一个内核模块的依赖关系。它仅用于帮助操作系统在加载此模块之前加载所有必需的模块；然而，这并不规定加载顺序。

**列表 9. 内核模块的定义**

```c
static device_method_t	ornata_methods[] = {
	DEVMETHOD( device_probe,  ornata_probe ),
	DEVMETHOD( device_attach, ornata_attach ),
	DEVMETHOD( device_detach, ornata_detach ),
	DEVMETHOD_END
};
static driver_t		ornata_driver = {
	.name		= " ornata "
	,
	.methods	= ornata_methods,
	.size		= sizeof(struct ornata_softc)
};
static devclass_t	ornata_devclass;
DRIVER_MODULE( ornata, uhub, ornata_driver, ornata_devclass, NULL, 0 );
MODULE_DEPEND( ornata, ukbd, 1, 1, 1 );
MODULE_VERSION( ornata, 1 );
USB_PNP_HOST_INFO( ornata_devs );
```

此时，我们可以实现一个函数，向设备发送设置数据。这可以通过使用 `usbd_do_request_flags` 函数和表示请求的 `usb_device_request` 结构来完成。对于数据部分，我们可以使用来自 openrazer 的结构，因为它已经在 C 语言中实现。例如，在 Python 驱动程序的情况下，函数将期望设置颜色和行，且大多数变量只是从我们嗅探到的请求中复制过来。我们还需要记得重新计算 CRC 字段。`USET` 宏允许我们设置与 CPU 字节序无关的数据。设置背光颜色的函数如列表 10 所示。

**列表 10. 驱动程序的 `attach` 和 `detach` 函数**

```c
static void
ornata_set_color( struct ornata_softc *sc, uint8_t r, uint8_t g, uint8_t b, uint8_t
		  line )
{
	struct razer_report		rr;
	struct usb_device_request	req;
	char				crc, *ptr;
	int				i;
	memset( &rr, 0, sizeof(rr) );
	req.bmRequestType	= 0x21;
	req.bRequest		= 0x09;
	USETW( req.wValue, 0x300 );
	USETW( req.wIndex, 2 );
	USETW( req.wLength, sizeof(rr) );
	rr.status		= 0x00;
	rr.transaction_id	= 0x1f;
	rr.remaining_packets	= 0x00;
	rr.protocol_type	= 0x00;
	rr.data_size		= 0x47;
	rr.command_class	= 0x0f;
	rr.command_id		= 0x03;
	rr.arguments[2]		= line;
	rr.arguments[4]		= 0x15;
	for ( i = 8; i < 8 + 0x15 * 3; i += 3 )
	{
		rr.arguments[i]		= r;
		rr.arguments[i + 1]	= g;
		rr.arguments[i + 2]	= b;
	}
	crc = 0;
	for ( ptr = (char *) &rr; ptr != (char *) &rr + sizeof(rr); ptr++ )
	{
		crc ^= *ptr;
	}
	rr.crc = crc;
	usbd_do_request_flags( sc->sc_udev, &sc->sc_mtx, &req,
			       &rr, 0, NULL, 2000 );
}
```

### 实现 ioctl

驱动程序中唯一缺失的部分是与 USB 设备节点通信的方法。我们将实现 `ioctl` 方法，因为它是最简单的方法（但需要额外的程序来发送 `ioctl`）。

首先，我们需要定义 `ioctl`。为此，我们可以使用 `_IOW` 宏，它定义了一个写操作宏——这意味着内存将从用户态复制到内核。对于其他用途，我们可以使用 `_IOR` 定义一个读取 `ioctl`，或使用 `_IOWR` 进行读/写操作，或使用 `_IO`，它不传输任何数据。我们还将使用一个额外的结构 `ornata_color`，仅用于以有序的方式传输数据。

`ioctl` 的定义在用户态和内核之间共享，因此一个好的做法是定义一个包含这些定义的 C 文件头。该头文件如列表 11 所示。

**列表 11. 驱动程序的 `attach` 和 `detach` 函数**

```c
#ifndef _ORNATA_H_
#define _ORNATA_H_
#include <sys/ioccom.h>
struct ornata_color {
	uint8_t r;
	uint8_t g;
	uint8_t b;
};
#defineORNATA_SET_COLOR _IOW('U', 205, struct ornata_color)
#endif
```

现在，回到 `usb_fifo_attach`，我们使用了一个尚未定义的结构 `ornata_fifo_methods`。这个结构定义了设备上支持的操作；例如，打开或关闭。在我们的情况下，我们希望支持 `ioctl` 操作。`basename` 字段描述了应在 `/dev` 下创建的节点名称。使用 `ioctl` 时，内存已经安全地从用户态复制到内核，因此我们可以直接使用 `color` 结构。`ioctl` 方法的实现如列表 12 所示。

**列表 12. `ioctl` 方法的实现**

```c
static int
ornata_ioctl( struct usb_fifo *fifo, u_long cmd, void *addr, int fflags )
{
	struct ornata_softc	*sc;
	struct ornata_color	color;
	int			error;
	uint8_t			line;
	sc	= usb_fifo_softc( fifo );
	error	= 0;
	mtx_lock( &sc->sc_mtx );
	switch ( cmd )
	{
	case ORNATA_SET_COLOR:
		color = *(struct ornata_color *) addr;
		for ( line = 0; line < 6; line++ )
		{
			ornata_set_color( sc,
					  color.r,
					  color.g,
					  color.b,
					  line );
		}
		break;
	default:
		error = ENOTTY;
		break;
	}
	mtx_unlock( &sc->sc_mtx );
	return(error);
}


static struct usb_fifo_methods ornata_fifo_methods = {
	.f_ioctl	= &ornata_ioctl,
	.basename[0]	= " ornata "
};
```

这种方法的缺点是我们必须实现一个额外的用户态程序，因为没有办法通过命令行生成 `ioctl(2)`。这个程序如列表 13 所示。

**列表 13. 用户态中使用 `ioctl` 的示例**

```c
int
main( void )
{
	int fd = open( " / dev / ornata0 "
		       , 0 );
	struct ornata_color color;
	color.r = 0xFF;
	color.g = 0x00;
	color.b = 0x00;
	ioctl( fd, ORNATA_SET_COLOR, &color );
	return(0);
}
```

## 总结

实现一个用户态驱动程序并不复杂，得益于 libusb 和 pyusb。最复杂的部分实际上是理解设备使用的协议。如果协议很简单，我们可以直接从不同平台上现有的驱动程序中嗅探大量数据。如果协议更复杂，也许有一个开源项目，我们可以将其中的一部分移植到 FreeBSD。编写本地驱动程序时，我们必须保持耐心，因为这些例程更具挑战性。实现内核驱动时，我们必须非常小心，因为我们可能会引入 bug。而且，如果我们搞砸了，内核可能会崩溃，我们需要重启机器。

## 参考文献

* USB 2.0 规范 — https://www.usb.org/document-library/usb-20-specification

* 《FreeBSD 设备驱动程序：勇敢者指南》by Joseph Kong
* Openrazer 源代码 — https://github.com/openrazer/openrazer
* Roland 的主页 — 从用户空间设置 Razer ornata chroma 颜色 (https://rsmith.home.xs4all.nl/hardware/setting-the-razer-ornata-chroma-color-fromuserspace.html)

---

**MARIUSZ ZABORSKI** 目前在 4Prime 担任安全专家。自 2015 年以来，他一直是 FreeBSD 的提交者。他主要的兴趣领域是操作系统安全和低级编程。他曾在 Fudo Security 工作，领导团队开发 IT 基础设施中最先进的 PAM 解决方案。2018 年，Mariusz 组织了波兰 BSD 用户组。在空闲时间，他喜欢在 https://oshogbo.vexillium.org 撰写博客。
