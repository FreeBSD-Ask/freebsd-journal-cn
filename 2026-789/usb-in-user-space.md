# 用户空间中的 USB

- 原文：[USB in User Space](https://freebsdfoundation.org/our-work/journal/browser-based-edition/production-deployments/usb-in-user-space/)
- 作者：**Rick Parrish**

## USB 工具简介：如何编写自己的自定义 USB 工具或用户空间驱动

本文先概述 FreeBSD 自带的 USB 工具。随后面向开发者，介绍 **libusb20(3)** 访问库。这个库提供了一种对开发者友好的方式，用来与通用 USB 内核模块交互，无需再把字节硬塞进 **ioctl(2)** 调用。

### FreeBSD 上的 USB 工具

下面概述几款无需编写任何代码就能使用的工具，并给出对应手册章节的参考。想了解 FreeBSD 上 USB 的总体情况，可以看看[架构手册第 13 章](https://docs.freebsd.org/en/books/arch-handbook/usb/index.html)。

**USB 存储与 OTG**

[存储（第 21 章）](https://docs.freebsd.org/en/books/handbook/disks/)完全是另一回事，本文不讨论。OTG 或 [gadget 模式（第 29 章）](https://docs.freebsd.org/en/books/handbook/usb-device-mode/)也一样。

**USB HID**（键盘与鼠标）

工具 [**usbhidaction(1)**](https://man.freebsd.org/cgi/man.cgi?query=usbhidaction(1)) 可以监视 HID 接口，捕捉遥控器发来的按键。典型用例是把键盘配置成：按下特殊的“Fn”键时执行与 OEM 丝印相符的动作，例如静音、调高或调低音量、调节屏幕亮度、开关 WiFi 等。作为 **usbhidaction(1)** 的配套工具，另见 [**usbhidctl(1)**](https://man.freebsd.org/cgi/man.cgi?query=usbhidctl)。

**USB 网络共享**（[第 35 章](https://docs.freebsd.org/en/books/handbook/advanced-networking/#network-usb-tethering)）

每当我需要笔记本上网而 WiFi 又不配合时，都可以退回到 USB 网络共享，使用智能手机提供的移动数据。把智能手机连到笔记本，在手机上打开 USB 网络共享。查看 **dmesg(8)**，应当能看到一个新设备“ue0”。唯一的小技巧是在新接口上运行 dhclient，其实一点也不麻烦——以 root 身份运行 `dhclient ue0` 即可。下面这条 **devd(8)** 规则可以让它自动完成。

```sh
notify 20 {
    match "system" "ETHERNET";
     match "type"  "IFATTACH";
     match "subsystem" "ue[0-9]+";
     action "/sbin/dhclient $subsystem";
};
```

**ls(1)**

也许你还不清楚，USB 设备节点出现在 **/dev/ugenX.Y** 和 **/dev/usb/X.Y.Z** 下，其中 X、Y、Z 都是十进制数字。我认为 **ls(1)** 最有用之处是查看节点的组和用户访问权限。**/dev/ugenX.Y** 设备节点指整个 USB 设备。例如：

```sh
$ ls -la /dev/ugen*
lrwxr-xr-x  1 root wheel 9 Jul 27 19:59 /dev/ugen0.1 -> usb/0.1.0
lrwxr-xr-x  1 root wheel 9 Jul 27 19:59 /dev/ugen0.2 -> usb/0.2.0
lrwxr-xr-x  1 root wheel 9 Jul 27 19:59 /dev/ugen0.3 -> usb/0.3.0
lrwxr-xr-x  1 root wheel 9 Jul 27 19:59 /dev/ugen1.1 -> usb/1.1.0
lrwxr-xr-x  1 root wheel 9 Jul 27 19:59 /dev/ugen1.2 -> usb/1.2.0
lrwxr-xr-x  1 root wheel 9 Jul 27 19:59 /dev/ugen1.3 -> usb/1.3.0
lrwxr-xr-x  1 root wheel 9 Jul 27 19:59 /dev/ugen1.4 -> usb/1.4.0
lrwxr-xr-x  1 root wheel 9 Jul 27 19:59 /dev/ugen2.1 -> usb/2.1.0
lrwxr-xr-x  1 root wheel 9 Jul 27 19:59 /dev/ugen2.2 -> usb/2.2.0
```

注意，**/dev/ugenX.Y** 设备节点都是指向 **/dev/usb/X.Y.0** 设备节点的符号链接（即第三位数字始终为零的设备节点）。下面是 **/dev/usb/X.Y.Z** 设备节点的示例列表：

```sh
$ ls -la /dev/usb
dr-xr-xr-x   2 root wheel     512 Jul 27 19:59 .
dr-xr-xr-x  13 root wheel     512 Jul 27 14:59 ..
crw-------   1 root operator 0x2f Jul 27 19:59 0.1.0
crw-------   1 root operator 0x45 Jul 27 19:59 0.1.1
crw-------   1 root operator 0x80 Jul 27 19:59 0.2.0
crw-------   1 root operator 0x84 Jul 27 19:59 0.2.4
crw-------   1 root operator 0x85 Jul 27 19:59 0.2.5
crw-------   1 root operator 0x86 Jul 27 19:59 0.2.6
crw-------   1 root operator 0x87 Jul 27 19:59 0.2.7
crw-------   1 root operator 0x88 Jul 27 19:59 0.2.8
crw-------   1 root operator 0x89 Jul 27 19:59 0.2.9
crw-------   1 root operator 0x8e Jul 27 19:59 0.3.0
crw-------   1 root operator 0x90 Jul 27 19:59 0.3.1
crw-------   1 root operator 0x91 Jul 27 19:59 0.3.2
```

……为节省篇幅，此处省略……

```sh
crw-------   1 root operator 0x33 Jul 27 19:59 2.1.0
crw-------   1 root operator 0x43 Jul 27 19:59 2.1.1
crw-------   1 root operator 0x82 Jul 27 19:59 2.2.0
crw-------   1 root operator 0x8a Jul 27 19:59 2.2.1
```

许多 **/dev/usb** 设备节点并不以零结尾。以非零数字结尾的节点，是同一设备提供的附加端点/接口组合。在上面的 **/dev/usb** 示例中，2.1.0 和 2.1.1 属于同一个设备；2.2.0 与 2.2.1 同理，也是同一个设备。

**dmesg 与 usbconfig**

运行 **usbconfig(8)** 可以查看已连接的 USB 设备。以 root 身份运行时，输出内容要丰富得多。以普通用户身份运行时，它会告诉你哪些设备可以访问。如果想要的设备没有出现，你可能需要创建一条 devd 规则，在设备接入时授予自己访问权限。Linux 上与之对应的工具大概是 lsusb。

**dmesg(8)** 会记录 USB 设备的接入和拔出事件。插上设备后不久，查看 **dmesg(8)** 的尾部，就能看到分配了哪个设备节点。下面这个例子中，我先拔下、再重新插入两个不同的 USB 适配器。

```sh
ugen0.2: <Realtek 802.11ac WLAN Adapter> at usbus0 (disconnected)
ugen0.2: <Realtek 802.11ac WLAN Adapter> at usbus0
ugen0.5: <Logitech USB Receiver> at usbus0 (disconnected)
ugen0.5: <Logitech USB Receiver> at usbus0
```

大多数 USB 设备都有相应的 FreeBSD 内核模块支持，这些模块把主机层的操作转换成 USB 设备能理解的形式，例如读取鼠标按键，或者为 WiFi 网卡收发 IP 数据包。

有些设备太过冷门，没有对应的内核模块；又或者厂商改为提供用户态库，从而免去编写内核模块的必要。

如今，从用户进程直接与 USB 设备通信，最可移植——也最流行——的方式是使用 **libusb(3)** 访问库，它在众多操作系统上都能用。FreeBSD 自带自己的实现。

FreeBSD 的 **libusb(3)** API 实现，底层是一套 FreeBSD 原生访问库 API，相比 libusb API 具备一些优势。本文关注的就是 FreeBSD 独有的这套原生访问库 API。

接下来，我们会看原生的 USB API——**libusb20(3)**——并完全用 FreeBSD 原生的组件，自己实现一份 **usbconfig(8)** / lsusb。除了 FreeBSD 标准自带的内容，我们不需要任何新的内核模块。这套 USB API 提供了所需的一切：定位想要的 USB 设备，像打开文件或 tty 设备那样打开它，再发送命令控制设备。

与任何 USB 设备通信的第一步，是从当前已连接的 USB 设备列表中找出目标设备。在 shell 提示符下，用 root 权限登录后运行 `usbconfig list`，就能看到有哪些设备。输出示例：

```sh
ugen0.1: <XHCI root HUB Intel> at usbus0, cfg=0 md=HOST spd=SUPER (5.0Gbps) pwr=SAVE (0mA)
ugen1.1: <EHCI root HUB Intel> at usbus1, cfg=0 md=HOST spd=HIGH (480Mbps) pwr=SAVE (0mA)
ugen2.1: <EHCI root HUB Intel> at usbus2, cfg=0 md=HOST spd=HIGH (480Mbps) pwr=SAVE (0mA)
ugen0.2: <AC600 wireless Realtek RTL8811AU [Archer T2U Nano] TP-Link> at usbus0, cfg=0 md=HOST spd=HIGH (480Mbps) pwr=ON (500mA)
ugen2.2: <Integrated Rate Matching Hub Intel Corp.> at usbus2, cfg=0 md=HOST spd=HIGH (480Mbps) pwr=SAVE (0mA)
ugen1.2: <Integrated Rate Matching Hub Intel Corp.> at usbus1, cfg=0 md=HOST spd=HIGH (480Mbps) pwr=SAVE (0mA)
ugen0.3: <Nano Receiver Logitech, Inc.> at usbus0, cfg=0 md=HOST spd=FULL (12Mbps) pwr=ON (98mA)
```

USB 设备主要靠一对 16 位标识来区分——厂商 ID（VID）和产品 ID（PID）。要查看每个设备的 VID 和 PID，运行 `usbconfig dump_device_desc`。VID 和 PID 通常显示为四位十六进制值。如果有两个或更多设备的 VID 和 PID 相同，往往还能靠序列号区分。在 USB 领域，序列号是字母数字字符串（不是整数），并不限于十进制数字。如果两个或更多设备的 VID、PID、序列号都相同，或许还能通过 USB 总线区分。

为特定设备编写的程序可以枚举所有设备，寻找 VID 和 PID 符合要求的候选设备。程序可以把匹配结果列给用户，也可以直接选第一个。

**接口与配置**

USB 设备会提供一个或多个接口，这样单个 USB 设备通常就能承担多种功能。处理批量传输和等时传输时，你需要指明使用哪个接口。**usbconfig(8)** 工具可以列出指定设备通告的所有接口。

```sh
usbconfig -d ugen0.1 dump_curr_config_desc
```

本文只使用默认配置，不过有些设备支持多种配置。**usbconfig(8)** 可以列出全部配置。

```sh
usbconfig -d ugen0.1 dump_all_config_desc
```

注意，我没有详细介绍接口和配置。它们的含义和数量因设备而异，可能性太多，这里无法深入。你只要知道它们存在就够了。

**devd(8)**

本文的重点是编写用户态——而非内核——代码来访问设备，并且无需提升到 root 权限。

在设备节点上精准地执行一次 **chmod(1)** 或 **chown(1)**，就不再需要 root。

一条简短的 **devd(8)** 规则就能让这一步自动完成。

有个取巧的办法：先把目标设备拔下来。把它插到空闲的 USB 插槽，查看 **dmesg(8)**，看分配到了哪个设备节点。知道了 VID 和 PID，一条简单的 devd 规则就能在设备每次重新接入主机时替你调整权限。用 **ls(1)** 验证 ugen 设备节点的权限。

为了方便，我们可以写一条只用来记录日志的 **devd(8)** 规则。

```sh
notify 100 {
       match "system"          "USB";
       match "subsystem"       "DEVICE";
       match "type"            "ATTACH";
       match "vendor"          "0x1234";
       match "product"         "0x5678";
       action "logger $cdev device ";
};
```

*match “vendor”* 子句匹配四位 VID，0x 前缀表明该值是十六进制。*match “product”* 子句匹配四位 PID，同样带 0x 十六进制前缀。上面的片段会在设备接入时记录设备节点。看到这样的日志消息，就说明规则里的匹配子句生效了。

USB 设备一般支持多个接口。如果能够成功记录设备接入的日志，我们就可以再加一条规则，作用于接口以修改组和用户权限。把命令写进 **devd(8)** 规则之前，应当先手动试一试。**chown(1)** 可以同时指定组和用户，像这样：

```sh
chown ordinary:wheel /dev/ugen1.2
chown ordinary:wheel /dev/usb/1.2.*
```

这里假定目标用户是“ordinary”，而他恰好属于“wheel”组。你可以按需修改。真正的 **devd(8)** 规则 action 子句要稍复杂一些，因为我想同时处理 **/dev/ugenX.Y** 节点和关联的 **/dev/usbX.Y.Z** 节点。完整片段如下：

```sh
notify 100 {
       match "system"          "USB";
       match "subsystem"       "INTERFACE";
       match "type"            "ATTACH";
       match "vendor"          "0x1234";
       match "product"         "0x5678";
       action "cdev=$cdev; usb=$(expr ${cdev} : 'ugen\([0-9]\.[0-9]\)'); chown -h ordinary:wheel /dev/${cdev}; chown -h ordinary:wheel /dev/usb/${usb}.*";
};
```

记得用 **ls(1)** 验证所有者是否设置正确。

**usbipd(8)**

**usbipd(8)** 守护进程和配套的 **usbip(8)** 命令可以连接远程主机上的 USB 设备。这个工具与 Linux 上一套知名的 USB-over-IP 实现兼容。你的 FreeBSD 主机既可以作客户端，也可以作服务端，或者两者兼作。

我在本机上通过 localhost 测试过：挑了一个在我的主机上显示为 ugen0.3 的设备。下面是具体步骤和结果。

```sh
git clone https://github.com/hackermaskee/freebsd-usbip
cd freebsd-usbip
make kmod
make
[sudo|doas|mdo]make install

[sudo|doas|mdo]vi /etc/rc.conf
usbipd_devices="ugen0.3"
usbipd_enable="YES"

[sudo|doas|mdo]kldload vhci
[sudo|doas|mdo]service usbipd start
[sudo|doas|mdo]usbip attach -r loocalhost -b ugen0.3
```

此时，usbconfig 会把该设备列出两次：

```sh
$ usbconfig
ugen3.2:  at usbus3, cfg=0 md=HOST spd=HIGH (480Mbps) pwr=ON (500mA)
ugen0.3:  at usbus0, cfg=0 md=HOST spd=HIGH (480Mbps) pwr=ON (500mA)
```

ugen3.2 设备节点是 ugen0.3 的复制设备。访问其中任何一个设备（这里用 rtl_test）效果都一样。值得一提的是，新设备节点也遵循了我的 devd 规则。

### libusb20(3)

**libusb20(3)** 是 FreeBSD 自己的访问库，让进程直接与 USB 设备通信，而不必经由某个领域专用的内核模块间接通信。通用 USB 内核模块对外提供 **ioctl(2)** 接口，把常用的 USB 操作开放给用户空间。**libusb20(3)** 访问库提供了一种对开发者友好的方式，用来与通用 USB 模块交互，无需再把字节硬塞进 **ioctl(2)** 调用。

管理 USB 设备大致涉及以下几方面：

1. **devd(8)**
2. 设备枚举
3. 传输
   - 控制传输
   - 批量传输
   - 中断传输
   - 等时传输

**devd(8)** 前面已经讲过。

**枚举**

要找到应用程序需要的特定设备，就必须能够枚举 USB 设备。下面是一些实现枚举的代码片段。我假定你会读写 C 或 C++。你的源文件需要包含以下头文件。

```c
#include <libusb20.h>
#include <libusb20_desc.h>
#include <dev/usb/usb_ioctl.h>
```

枚举涉及以下函数：

- `libusb20_be_alloc_default`
- `libusb20_be_device_foreach`
- `libsub20_dev_get_device_desc`
- `libusb20_be_free`

**libusb20(3)** 引入了后端（backend，BE）的概念，用来定位设备。说明一下，这与 ZFS 的 BE 毫无关系，完全是两码事。理论上你可以编写自己的后端，用来创建模拟 USB 设备以便测试，或者实现类似 USB over TCP 的东西。到目前为止，我还没见过有人用这种方式实现 USB over IP。这里我们只需要默认后端——不过本文末尾会给出一个纯软件的模拟 USB 设备示例。

调用 **libusb20_device_foreach(3)** 可以逐个节点枚举设备列表。**libusb20_dev_get_device_desc(3)** 能提供设备节点的更多细节，对枚举可能有用。

这段代码假定有一个外部函数 `candidate`，当 VID 和 PID 匹配时返回 true，否则返回 false。

```c
bool (*candidate)(uint16_t vid, uint16_t pid);
```

它还假定有一个函数 `acceptable`，当 usb_device_info 数据对应用程序来说可接受时返回 true。

```c
bool (*acceptable)(struct usb_device_info *info);
```

至于什么才算可接受，由你决定。

```c
libusb20_device *handle = nullptr;

libusb20_device *walker = nullptr;
auto be = libusb20_be_alloc_default();
if (be == nullptr)
{
   printf(stderr, "libusb20_be_alloc()\n");
   return false;
}

// true 表示找到了合适的设备，可以停止查找。
bool okay = false;
walker = libusb20_be_device_foreach(be, walker);
while (walker != NULL)
{
   auto ddp = libusb20_dev_get_device_desc(walker);
   // 匹配 vid 和 pid 吗？
   if ( candidate(ddp->idVendor, ddp->idProduct) )
   {
       // 是：打开它以获取更多信息。
       if (libusb20_dev_open(walker, reserve) == LIBUSB20_SUCCESS)
       {
           struct usb_device_info info{};
           libusb20_dev_get_info(walker, &info);
           // 我们可以检查 usb_device_info 的内容，
           // 进一步把这个设备与其他设备区分开。
           okay  = acceptable(walker, info);
           if (okay)
           {
               handle = walker;
               libusb20_be_dequeue_device(be, walker);
               break;
           }
           // 不是我们想要的设备，继续查找。
           libusb20_dev_close(walker);
       }
   }
   walker = libusb20_be_device_foreach(be, walker);
}
libusb20_be_free(be);
return okay;
```

上面的代码可以改成把一组 usb_device_info 对象作为选择列表呈现给用户。就目前的写法而言，代码返回指向它认为可接受的第一个候选设备的设备指针。

注意 **libusb20_dev_open(3)** 的 `reserve` 参数。讲到传输时我们会解释它的用途。

找到合适的设备后，调用 **libusb20_be_dequeue_device(3)** 把设备从设备列表中移除。这是预留设备的一种方式，不过并没有我希望的那么有用。如果你重新枚举同一个设备列表，它不会出现在里面。但是，在同一进程或另一个进程中调用 **libusb20_be_device_foreach(3)** 会创建一份新列表，设备依然在里面。这并不会给你对设备的独占访问权。

现在我们有了设备指针，下一步是用 **libusb20_dev_open(3)** 打开设备。此后一直使用同一个设备指针，没有单独的设备句柄。这不会授予独占访问权。一种解决办法是在设备上放一个劝告锁，但只有当所有相关方都遵守这个劝告锁时才有效。我个人觉得允许两个进程共享一个 USB 设备（比如温度/湿度传感器）挺酷，不过这属于少见场景。有人在讨论扩展 **libusb20(3)** 的 ABI 以接受独占标志，但至今没有下文。

设备打开之后，我们就可以做些有意思的事情了。

其中之一是获取设备信息，另一个是执行四种传输中的一种。那么这四种传输分别是什么呢？

**控制传输**最常见。它们是一些简短的、设备专用的命令。每条命令由 8 字节头部和可选负载组成。有的向设备发送数据，有的从设备取回数据。为简单起见，控制传输通常同步完成。

**中断传输**属于低带宽、偶发的传输。键盘和鼠标是利用中断传输的两个主要例子。这类设备通常由内核模块处理，本文不讨论中断传输。

**等时传输**和**批量传输**非常相似，区别在于等时传输对交付提供一些额外保证（若无法满足这些保证就会失败）。我主要使用批量传输，但两者的机制极为相似，需要做等时传输的人可以参照批量的例子。

在介绍传输之前，先做个不太严格的类比：把“ugen”（通用 USB 设备）比作 **/dev/tty** 串口。在串口上收发字符数据之前，必须先设置波特率、每字符位数、起始位、停止位、校验位。这一步相当于给 USB 设备下达命令的控制传输。串口字符数据的收发，则相当于 USB 的批量传输或等时传输。

**控制传输**

控制传输常用于初始化设备，或让设备进入某种工作模式。

控制传输有两个方向——IN 或 OUT。它包含一个固定部分（`LIBUSB20_CONTROL_SETUP_DECODED`）和一个可选的数据部分。`libusb20_dev_request_sync` 函数带有一个参数，用来捕获实际传输的字节数。这是除 setup 头部所占字节之外的“额外”字节数。

```c
struct LIBUSB20_CONTROL_SETUP_DECODED setup;
```

这个宏把上面的结构体初始化为合理的起始值。

```c
LIBUSB20_INIT(LIBUSB20_CONTROL_SETUP, &setup);
```

成员 `bmRequestType` 通常是两个值之一——用于输入或用于输出。

```c
setup.bmRequestType = LIBUSB20_REQUEST_TYPE_VENDOR | LIBUSB20_ENDPOINT_OUT;

setup.bmRequestType = LIBUSB20_REQUEST_TYPE_VENDOR | LIBUSB20_ENDPOINT_IN;

setup.bRequest = request;
setup.wIndex = index
setup.wValue = value;
setup.wLength = length;
```

上述四个参数的具体取值由设备决定。`wLength` 是额外字节数。对于 `LIBUSB20_ENDPOINT_OUT` 请求类型，它是要写入 USB 设备的额外数据字节数。对 Web 开发者来说，这大致相当于 HTTP POST。当然，你是向 USB 设备“POST”，而不是向 HTTP Web 服务器。

```c
int status = libusb20_dev_request_sync(dev, &setup, data, &actual, TIMEOUT, 0);
```

`dev` 是设备指针。`setup` 是 `LIBUSB20_CONTROL_SETUP_DECODED` 的实例。`data` 是指向可选额外数据的指针。`actual` 捕获实际传输的额外字节数。对于 `LIBUSB20_ENDPOINT_IN`，`actual` 可能小于或等于 `setup.wLength` 中指定的值。对 Web 开发者来说，这大致相当于 HTTP GET。

`libusb20_dev_request_sync` 会阻塞，直到控制传输完成。下面的批量传输示例不是同步的，因此你可以发出重叠的请求。我避免让控制传输重叠，因为下一次控制传输往往取决于上一次控制传输是否成功（以及它返回的数据）。

**中断传输**

大多数支持中断传输的设备都有对应的内核模块，因此很少需要以编程方式从用户层访问。

**批量传输**

批量传输可以发送或接收数据块，很像通过套接字或文件句柄传输数据。

有些设备对批量/等时传输的大小有限制。我支持过的一些芯片组要求传输大小必须是 512 字节的整数倍。

对于流式数据（如音频或视频），你需要在延迟和开销之间取得平衡。如果把 1 MB 数据切成一个个 512 字节的小块来传输，就要发出 2048 个请求。你也可以用单个请求发送同样的数据。数据块越大，CPU 开销越低，但如果你想中途停止流式传输或更换内容，延迟代价也越大。把数据块大小调整到与目标帧率（比如 30 fps）相匹配，就能比用——比方说——一分钟的数据块更早暂停传输。

假设你要以 192 ksps 传输立体声音频采样。每个采样 4 字节（每声道 16 位），也就是每秒 192k 采样 × 每采样 4 字节 = 768 KB。按 30 fps 计算，一帧就是 25600 字节。注意这恰好是 512 的整数倍。如果你的帧大小不是 512 的整数倍，就需要向上或向下取整。这意味着实际帧率不会正好是 30 fps，不过在大多数场合都没问题。

要连续流式传输，就需要管理多个未完成的传输。你希望有一个传输正在进行，另外至少还有两个传输在等待。一个传输完成后，缓冲区就可以复用——并重新提交。按上面的例子，你至少需要三个各 25600 字节的传输。要双向流式传输，每个方向都需要这么多传输。

我觉得把流式传输拆成四步最有用：

1. 预留，
2. 启动，
3. 停止，
4. 释放。

**预留**负责分配一组传输缓冲区。为了便于管理，我会创建一个传输上下文，把与传输相关的零碎信息放在一起。

```c
// USB 传输上下文。
struct Transfer
{
   void(unsigned char *data, unsigned size) *callback;
   unsigned char *buffer;
   struct libusb20_transfer *transfer;
   uint32_t size;
   uint32_t timeout;
};

struct libusb20_transfer *transfer = libusb20_tr_get_pointer(dev, i);
Transfer *context = new Transfer(back, transfer, bucket, 3 * timeout);
result = libusb20_tr_open(context->transfer, context->size, 1, 0x81);
// 每个缓冲区约 30 毫秒，因此整整一秒的时间足够完成传输。
libusb20_tr_setup_bulk(context->transfer, context->buffer, context->size, context->timeout);
libusb20_tr_set_callback(context->transfer, bulk);
libusb20_tr_set_priv_sc1(context->transfer, context);
```

注意，上面的上下文只跟踪一个传输。要做三个传输，就需要三个 Transfer 对象。还记得 **libusb20_dev_open(3)** 的 `reserve` 参数吗？它告诉宿主内核预期有多少个传输。对于只做批量和控制传输的设备，`reserve` 等于批量传输数加上控制传输数。本文使用 2 个控制传输和 3 个批量传输，所以 `reserve` 参数为 5。

**libusb20_tr_open(3)** 接受帧数和端点。我把每个传输都作为一个帧发出。IN 和 OUT 端点各有 16 个。给出的示例——0x81——是一个批量 IN 端点。与配置和接口一样，端点的编号因设备而异。注意，按 USB 术语，发往特定端点的一组传输也称为管道（pipe）。

> **提示**：如果控制传输正常，而 **libusb20_tr_open(3)** 失败，请检查你的配置索引。

**libusb20_tr_set_priv_sc1(3)** 让我们能把一个裸指针与传输关联起来。我把这个指针设成管理该传输的上下文对象。它的用处很快就会显现。

**启动**为每个传输缓冲区调用一次 **libusb20_tr_start(3)**。

**停止**为每个传输缓冲区调用一次 **libusb20_tr_stop(3)**。

**释放**释放每个传输关联的内存和资源。

在预留阶段，我喜欢用一次 malloc 调用分配一大块内存，而不是分多次分配小块。这样释放阶段就简化为一次 free 调用。

**libusb20_tr_set_callback(3)** 设置完成例程的函数指针。对于出站流，这个例程用新数据填充传输缓冲区；对于入站流，它从传输缓冲区复制数据。它还会重新提交传输，让流持续下去。

完成例程看起来很像预留和启动两步的结合。

```c
uint8_t status = libusb20_tr_get_status(transfer);
uint32_t actual = libusb20_tr_get_actual_length(transfer);
uint32_t size = libusb20_tr_get_max_total_length(transfer);
Transfer *context = (Transfer *)libusb20_tr_get_priv_sc1(transfer);
// 向缓冲区复制数据或从缓冲区复制数据的步骤放在这里。
libusb20_tr_setup_bulk(transfer, context->buffer, context->size, context->timeout);
libusb20_tr_set_callback(transfer, bulk);
libusb20_tr_set_priv_sc1(transfer, context);
libusb20_tr_submit(transfer);
```

检查 `status` 就能知道有没有出错。理想情况下，`actual` 应当与 `size` 相等。向缓冲区复制数据或从缓冲区复制数据之后，再调用剩下的函数。注意 **libusb20_tr_get_priv_sc1(3)** 如何让我们从传输中取回上下文指针。

你可以先做预留这一步，然后任意多次执行启动/停止步骤。除非你想退出，或者想改变传输缓冲区的规模（更多缓冲区、更少缓冲区、更大缓冲区、更小缓冲区），否则不需要执行释放这一步。如果新的规模超出 **libusb20_dev_open(3)** 中的 `reserve` 参数，就必须关闭设备，再用新的 `reserve` 值重新打开。

务必在释放之前先停止。如果尚未完成的 I/O 修改了你已经释放的数据缓冲区，堆会非常生气——这就成了“释放后使用”漏洞。

**等时传输**

我一直没有需要处理等时传输的场合，所以只好让你自己去摸索了。据我了解，它的设置过程与批量传输非常相似。参见 **libusb20_tr_setup_isoc(3)**。

**事件泵**

最后一块是事件泵。它与 X.org 应用程序中的事件泵类似。区别在于泵送的不是 X 窗口事件，而是 USB 事件。这个执行线程最终会调用各个完成例程。没有事件泵，你的代码就不知道传输何时完成。事件泵的代码很短。大多数程序把它放在后台线程里。下面的例子每秒轮询 30 次。

```c
while (flag)
{
   int milliseconds = 1000/30;
   if (libusb20_dev_process(device) == 0)
   {
       libusb20_dev_wait_process(device, milliseconds);
   }
}
```

**测量延迟**

测量延迟有助于判断进程跟上输入数据的能力。**libusb20_tr_pending(3)** 函数会询问 USB 协议栈是否认为某个传输已经完成。统计已完成（但尚未处理）的缓冲区数量，就能大致看出进程落后了多少。如果所有缓冲区都已完成，进程很可能丢了数据，因为没有空闲传输来捕获数据。短期解决办法是增加缓冲区数量。更好的办法是优化处理代码，缩短每个传输的处理周转时间。

**清除暂停**

设备可以把某个端点标记为暂停（halt）。暂停对控制传输一般不成问题，但它会让批量传输和等时传输真正停下来。根本原因可能是线缆松动，或者其他古怪的毛病。清除暂停后，主机就能恢复与该端点的收发。**libusb20_tr_clear_stall_sync(3)** 函数需要一个传输指针。与其复用某个批量传输指针，我会在 **libusb20_dev_open(3)** 中多预留一个传输指针。这样我总有一个专用于清除暂停的备用传输指针。

举例来说，你的应用程序需要 N 个批量传输指针，于是以最大传输数 N + 1 调用 `libusb20_dev_open`，下面是清除端点 0x81 暂停的步骤。

```c
const uint8_t ep = 0x81;
auto transfer = libusb_tr_get_pointer(handle, N);
auto transfer = libusb_tr_open(handle, 0, 1, ep);
if (transfer != nullptr)
{
   libusb20_tr_clear_stall_sync(transfer);
   libusb_tr_close(handle);
}
```

注意 `libusb20_tr_open` 的参数——尤其是大小为 0、数量为 1。清除端点暂停所需的就这些。

**修改后端**

下面这个依赖注入的小技巧，利用 USB 后端机制注入一个完全存在于软件中的伪设备。所有交互都通过 **libusb20(3)**（以及 libusb）库进行。首先，创建你自己的 **libusb20_be_alloc_default(3)** 实现。

你的实现会用 **dlsym(3)** 找到并调用真正的 **libusb20_be_alloc_default(3)**。

它会返回一份 **libusb20_be_alloc_default(3)** 所返回后端结构的修改副本。这个修改后的结构里含有函数指针。你可以把自己的代码接进去，拦截对默认后端的调用。由此就能做一些事情，比如拦截对 **libusb20_dev_open(3)** 的调用，返回一个能响应控制传输和批量或等时传输的伪设备。

这样就能对 **libusb20(3)** 和 **libusb(3)** 的客户端代码做相当细粒度的单元测试，无需真实硬件。编写自己的后端需要以下两个头文件，顺序如下：

```c
#include <sys/queue.h>
#include <libusb20_int.h>
```

**DTrace**

围绕访问库编写代码虽然让调试轻松许多，但有时你需要看清原始的 ioctl。例如，对 **libusb20_tr_open(3)** 的调用失败，而库只返回一个非常笼统的 `LIBUSB20_ERROR_INVALID_PARAM`。下面这段 dtrace 会告诉你实际的返回值。知道这个值，你就可以查看 ugen 驱动的内核源码，寻找失败原因的线索。

```sh
dtrace -n 'fbt:kernel:usb_ioctl:return /execname == "my_usb_code"/ { printf("%d\n", (int)arg1); }'
```

**libusb20(3) 相对于 libusb(3) 和 WinUSB 的优势**

我喜欢 **libusb20(3)** ABI 的简洁直接。它的复杂度刚好够用，同时让代码保持可读。

最大的优势是软件许可。许可和版权很重要。Linux 和 Windows 上真正的 **libusb(3)** 库采用 LGPL，而在很多场合 LGPL 是完全可以接受的。

**libusb20(3)**（以及兼容 **libusb(3)** 的 shim）是 FreeBSD 的一部分，采用 BSD-2-Clause 许可。这是非常宽松的许可。FreeBSD 版 pthreads 也是如此。在 FreeBSD 上，更容易避开 (L)GPL 和链接例外方面的合规问题。许可和版权就说到这里。

一项主要的技术优势是：即使设备已被另一个进程占用，我仍能找到该设备，并从设备读取字符串描述符。在 Linux 和 Windows 的 **libusb(3)** 实现中，设备会从设备列表中消失。这历来让最终用户困惑：他们常用的软件提示那个闪亮的 USB 设备不存在——只因为别的进程先占用了它。

像 **usbconfig(8)** 这样的工具没法用 **libusb(3)** 实现——至少在非 FreeBSD 平台上不行，因为这些平台上 **libusb(3)** 的枚举实现方式不允许。看不见的东西，自然描述不出来。

我还把微软的 WinUSB 拿来一起比较。在 WinUSB 中发起传输的方式非常相似，所以在这个层面上差别不大。不过，Windows 的 USB 设备枚举糟糕透顶：FreeBSD 或 Linux 只需一段代码，它却要几百行。曾几何时，Windows 各版本之间（7 与 8 之间）还有破坏兼容的 WinUSB（通过 SetupAPI）变更。就我个人而言，还没见过 **libusb20(3)** 出现破坏兼容的变更。

**libusb20(3) 相对于 libusb(3) 和 WinUSB 的劣势**

前面已经暗示过，它也有劣势。

一项主要劣势是缺少独占设备访问。在 Windows 和 Linux 上，如果两个进程枚举到同一个设备并尝试打开它，先到者胜出。在 FreeBSD 上，两者都能打开，这可能造成一些混乱。对那些假定 Windows 或 Linux 行为的 Ports 来说，这是个值得注意的问题。

劝告锁可以解决独占问题，但只有当所有相关方都采用时才有效。

最大的劣势是缺乏可移植性。除了 FreeBSD 及 FreeBSD 衍生操作系统，你在别处看不到 **libusb20(3)**。就我的用例而言，这完全没问题。

如果你有一个想在 FreeBSD 上支持的 USB gadget，可以考虑用 **libusb20(3)** 写些代码来测试这个设备。从用户态调试比调试内核模块容易。等代码能正常工作了，事情就完成了。如果你坚持要做内核模块，那就以这份用户态代码为基础，编写用同样方式驱动该设备的新内核模块。

下面是一个用 libusb20 实现的最简 usb-list 程序。

```c
#include <assert.h>
#include <unistd.h>
#include <stdio.h>
#include <libusb20_desc.h>
#include <libusb20.h>
#include <sys/queue.h>
#include <libusb20_int.h>
#include <dev/usb/usb_ioctl.h>

int main(int, const char *[])
{
   auto be = libusb20_be_alloc_default();
   struct libusb20_device *device = nullptr;
   while ( (device = libusb20_be_device_foreach(be, device)) != nullptr)
   {
       int error = libusb20_dev_open(device, 0);
       if (error == LIBUSB20_SUCCESS)
       {
           struct usb_device_info info{};
           libusb20_dev_get_info(device, &info);
           fprintf(stdout, "%s %04hX (%s) %04hX (%s) %s [%d]\n",
                   libusb20_dev_get_desc(device),
                   info.udi_vendorNo,
                   info.udi_vendor,
                   info.udi_productNo,
                   info.udi_product,
                   info.udi_serial,
                   info.udi_config_index);
           libusb20_dev_close(device);
       }
   }
   libusb20_be_free(be);
   return 0;
}
```

下面是用 libusb 实现的大致等价版本，应当可以移植到其他宿主操作系统。

```c
#include <assert.h>
#include <unistd.h>
#include <stdio.h>
#include <libusb.h>

int main(int, const char *[])
{
   libusb_context *context = nullptr;
   libusb_device **list = nullptr;
   unsigned index = 0;
   libusb_device *device = nullptr;
   int result = libusb_init(&context);
   if (result == LIBUSB_SUCCESS)
   {
       if (libusb_get_device_list(context, &list) >= 0)
       {
           while ( (device = list[index++]) != nullptr )
           {
               libusb_device_handle *handle = nullptr;
               struct libusb_device_descriptor desc{};
               result = libusb_get_device_descriptor(device, &desc);
               if (result == LIBUSB_SUCCESS)
               {
                   result = libusb_open(device, &handle);
                   if (result == LIBUSB_SUCCESS)
                   {
                       int config = 0;
                       uint8_t make[128]{};
                       uint8_t product[128]{};
                       uint8_t serial[128]{};
                       result = libusb_get_string_descriptor_ascii(handle,
                           desc.iManufacturer, make, sizeof make);
                       if (result <= 0)
                           sprintf((char *)make, "vendor %04hX", desc.idVendor);
                       result = libusb_get_string_descriptor_ascii(handle,
                           desc.iProduct, product, sizeof product);
                       if (result <= 0)
                           sprintf((char *)product, "product %04hX", desc.idProduct);
                       libusb_get_string_descriptor_ascii(handle, desc.iSerialNumber,
                           serial, sizeof serial);
                       libusb_get_configuration(handle, &config);
                       fprintf(stdout, "ugen%d.%d %04hX (%s) %04hX (%s) %s [%d]\n",
                       libusb_get_bus_number(device),
                               libusb_get_device_address(device),
                               desc.idVendor,
                               make,
                               desc.idProduct,
                               product,
                               serial,
                               config);
                       libusb_close(handle);
                   }
               }
           }
           libusb_free_device_list(list, 0);
       }
       libusb_exit(context);
   }
   return 0;
}
```

**延伸阅读**

DTrace —— 第 28 章（FreeBSD 手册）
<https://docs.freebsd.org/en/books/handbook/dtrace/>

USB 设备 —— 第 13 章（架构手册）
<https://docs.freebsd.org/en/books/arch-handbook/usb/index.html>

USB 设备 —— 第 29 章（FreeBSD 手册）
<https://docs.freebsd.org/en/books/handbook/usb-device-mode/index.html>

---

**Rick Parrish** 是一位资深 C++ 开发者，他对 USB 设备的兴趣让他发现了 FreeBSD 的 libusb20 访问库。他发现该库文档存在空白，于是向 FreeBSD 文档团队提交了一些小的改进。
