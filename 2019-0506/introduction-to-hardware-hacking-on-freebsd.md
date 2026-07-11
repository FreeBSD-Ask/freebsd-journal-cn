# FreeBSD 硬件黑客入门

作者：**Tom Jones**

FreeBSD 是创客和硬件黑客项目的理想平台，因为我们在各类板卡上都提供了扎实且统一的支持。

FreeBSD 对许多不同的 ARM 单板计算机（SBC）都有出色支持。Linux 随大多数 SBC 一起发布，每块 SBC 似乎都自带一份 Linux 发行版。通常，对板卡支持最好的 Linux 发行版来自板卡厂商，结果就是发行版支离破碎、软件包支持参差不齐、版本不一致。FreeBSD 做得更好——我们不一定立刻支持每块板卡，但我们支持的板卡往往提供同样优秀的体验。我们有一套单一、一致的系统，在所有支持的板卡上运行方式相同。只要能运行 FreeBSD，它运行的就跟桌面系统上是同一份 FreeBSD。

本文将展示如何利用 FreeBSD 出色的硬件支持来控制真实设备。我们将深入探讨如何在 ARM SBC 上使用 FreeBSD 的 GPIO。

通用输入输出（GPIO）设备为我们打开了一扇窗，让计算机能在现实世界中引起变化。这通常通过向内存的特定区域写入或使用处理器寄存器来控制现实世界的某些东西来实现。在 ARM SoC 上，这通常通过一段映射的内存实现，向这段内存写入即可控制暴露引脚上的电压。FreeBSD 提供设备驱动，让用户空间进程能在所有支持的硬件平台上以相同接口与 GPIO 交互。

GPIO 设备并非在所有系统上都可用，但在相当多的系统上都存在。通常，暴露的 GPIO 在系统主板的 PCB 上以针脚排针的形式出现。如果你配置过 pc engines 主板，可能注意到了串口旁边标着 GPIO 的针脚排针。如今许多 SBC 专为硬件项目设计，专门暴露 GPIO 供使用。还有 Intel 系列板卡，例如 LattePanda，运行大家熟悉的 64 位（x86）平台。总的来说，大多数为硬件项目设计的 SBC 运行 32 位或 64 位 ARM 架构，例如 BeagleBone 和树莓派系列电脑。

本文使用 ARMv7（32 位）BeagleBone Black 板卡。GPIO 和总线接口的概念对 FreeBSD 支持的所有平台通用，迁移到不同硬件也相当容易。BeagleBone Black 是一块好板子，它的 FreeBSD 支持非常成熟，并暴露了大量用于实验的 IO 端口。

要跟着本文做实验，你需要：

- 一块 BeagleBone Black SBC（其他板卡也可以，但所有 GPIO 名称都会不同）
- 一根 mini USB 数据线
- 一根以太网线，用于将 BeagleBone Black 连接到互联网
- 一块小面包板
- 一些跳线
- 3 个电阻（200 欧姆到 400 欧姆之间即可）
- 3 个 LED（1 红、1 绿、1 橙）

## 设置 BeagleBone Black

可从项目镜像下载 BeagleBone Black 的最新 FreeBSD 发行镜像（撰写本文时为 12）。下载后必须解压，并用 dd 写入 SD 卡：

```sh
$ fetch ftp://ftp.freebsd.org/pub/FreeBSD/releases/arm/armv7/ISO-IMAGES/\
12.0/FreeBSD-12.0-RELEASE-arm-armv7-BEAGLEBONE.img.xz
$ xz -d FreeBSD-12.0-RELEASE-arm-armv7-BEAGLEBONE.img.xz
# dd if=FreeBSD-12.0-RELEASE-arm-armv7-BEAGLEBONE.img of=/dev/da0 bs=1m
```

SD 卡挂载的位置因机器而异；使用 dd 命令时要小心，它会毫无保护地覆盖你指定的磁盘。

SD 卡准备好后，把 BeagleBone Black 的 mini USB 端口连接到电脑的 USB 端口。这根线既供电，也提供我们要用来连接板卡的接口。BeagleBone Black 作为 USB 串口 gadget 运行，通过这个接口创建一个虚拟串口。连接板卡后，在 `dmesg` 中查找新的 `umodem` 设备，或在 **/dev** 中查找 **/dev/ttyUX** 设备（在 Mac OS 上，该设备很可能是 **/dev/tty.usbmodemFreeBSD11**）。你应该能看到一个 USB 串口设备，可以用基本系统中自带的 `cu` 串口终端从 FreeBSD 系统建立连接，波特率为 115200：

```sh
# cu -l /dev/ttyU0 -s 115200
```

退出 `cu` 时按回车，输入 `~.`（波浪号后接句点）。通过串口连接后，使用默认用户名和密码（`root`/`root` 或 `freebsd`/`freebsd`）进入系统。

## FreeBSD 上的 GPIO

进入系统后，可以四处看看，这就是一份正常的 FreeBSD 安装。在 `dmesg` 中搜索，你会看到四个 GPIO 控制器已连接，并为每个控制器创建了 GPIO 总线（`gpiobus`）和控制（`gpioc`）接口：

```sh
gpio0: <TI AM335x General Purpose I/O (GPIO)> mem 0x44e07000-0x44e07fff irq 7 on simplebus0
gpiobus0: <OFW GPIO bus> on gpio0
gpioc0: <GPIO controller> on gpio0
```

系统中可以有多个 GPIO 控制器。FreeBSD 把每个控制器作为独立的 **/dev/gpiocX** 设备暴露出来。BeagleBone Black 上有四条总线。

与 Linux 不同（Linux 通过 **/sys/class** 把 GPIO 设备作为文件暴露），FreeBSD 提供了一个名为 `gpioctl` 的命令行工具（以及 C 库）来与 GPIO 引脚交互。我们试试用 `gpioctl` 列出 BeagleBone Black 上的一些可用引脚。

```sh
# gpioctl -f /dev/gpioc1 -lv
...snip...
pin 19: 0
gpio_19<IN,PD>, caps:<IN,OUT,PU,PD,UNKNOWN,UNKNOWN,UNKNOWN,UNKNOWN,UNKNOWN>
pin 20: 0
gpio_20<IN,PD>, caps:<IN,OUT,PU,PD,UNKNOWN,UNKNOWN,UNKNOWN,UNKNOWN,UNKNOWN>
pin 21: 0
gpio_21<OUT>, caps:<IN,OUT,PU,PD,UNKNOWN,UNKNOWN,UNKNOWN,UNKNOWN,UNKNOWN>
pin 22: 0
gpio_22<OUT>, caps:<IN,OUT,PU,PD,UNKNOWN,UNKNOWN,UNKNOWN,UNKNOWN,UNKNOWN>
...snip...
```

`gpioctl` 给出控制器知道的所有引脚列表。BeagleBone Black 上四个 GPIO 控制器每个都有 32 个引脚。`gpioctl` 告诉我们每个引脚的几件事。首先是它用来引用该引脚的编号。其次是该引脚当前的逻辑电平。输出中最后一列是引脚的硬件名（即 `gpio_22`）以及引脚的标志位。

某些控制器有取自 SoC 数据手册的更具描述性的引脚名称。

**gpioctl(8)** 列出引脚的可能标志：

- Input pin（输入引脚）
- Output pin（输出引脚）
- Open drain pin（开漏引脚）
- Push pull pin（推挽引脚）
- Tristate pin（三态引脚）
- Pull-up pin（上拉引脚）
- Pull-down pin（下拉引脚）
- Inverted input pin（反相输入引脚）
- Inverted output pin（反相输出引脚）

从 `gpioctl` 输出可以看出，引脚 19 默认配置为输入并配有下拉，而引脚 21 配置为输出，但可配置为输入或输出，并可加接上拉或下拉电阻。下拉是在引脚与地之间加接电阻，使引脚输出的逻辑电平保持低电平，除非被驱动为高电平。这后面会派上用场。

引脚 21 配置为输出。查看 BeagleBone Black System Reference Manual 第 6.5 节，可以看到 BeagleBone Black 有四个用户可控 LED，名为 USR0-USR3，分别连接到 GPIO1 21、GPIO2 22、GPIO2 23 和 GPIO2 24。LED USR0 可供我们使用，其余预留给其他任务。

| GPIO SIGNAL | USR | Ball |
| ----------- | --- | ---- |
| GPIO1_21 | USR0 | V15 |
| GPIO2_22 | USR1 | U15 |
| GPIO2_23 | USR2 | T15 |
| GPIO2_24 | USR3 | V16 |

用户 LED 控制信号（BBB SRM 表 7，CC-SA）。表格由 BeagleBoard.org 的 Gerald Coley 制作。更多信息见 <http://creativecommons.org/license/results-one?license_code=by-sa>。

从 `gpioctl` 输出可以看到，四个引脚的逻辑电平都是 0；找到板上 LED 最简单的办法就是把它们点亮。我们这么做。

`gpioctl` 可以把 LED 设为数字值 0 或 1。我们来配置并点亮 USR0：

```sh
# gpioctl -f /dev/gpioc1 21 OUT
# gpioctl -f /dev/gpioc1 21 1
```

尽情欣赏我们辉煌的 LED 的光辉吧。

进一步，可以用 `gpioctl` 的 `-t` 标志切换 LED 状态。当你只想让 LED 闪烁时这很有用：

```sh
# gpioctl -f /dev/gpioc1 -t 21
# gpioctl -f /dev/gpioc1 -t 21
# gpioctl -f /dev/gpioc1 -t 21
```

应该会让 LED 亮起又熄灭。

使用内置 LED 显示板卡状态信息、试用工具都很方便，但支撑这些 LED 的 GPIO 并未在 BeagleBone Black 上引出到针脚排针。我们用外接 LED 再做一次同样的事。

BeagleBone Black 有两组针脚排针，名为 P8 和 P9。以太网口在右侧时，P8 是下方排针，P9 是上方排针。P8 大部分引脚分配给 LED 引脚（可重新配置），仍然有几个可用的 GPIO 供我们使用。

我们将 LED 连接到 P8 的第 14、16、18 脚。从 System Reference Manual 可以查到它们连接到的实际芯片 GPIO：

| Ball | GPIO |
| ---- | ---- |
| T11 | GPIO0_26 |
| V13 | GPIO1_14 |
| V12 | GPIO2_1 |

需要把引脚连接到 LED 的长腿，还要使用电阻来限制流过 LED 的电流。约 200 欧姆到 470 欧姆的电阻应该合适。

LED 接好后，把它们的 GPIO 配置为输出，就可以玩了：

```sh
# gpioctl -f /dev/gpioc0 -c 26 OUT
# 红色
# gpioctl -f /dev/gpioc1 -c 14 OUT
# 橙色
# gpioctl -f /dev/gpioc2 -c 1 OUT
# 绿色
# gpioctl -f /dev/gpioc0 26 1
# gpioctl -f /dev/gpioc1 14 1
# gpioctl -f /dev/gpioc2 1 1
```

现在三个 LED 都应点亮。如果你像我一样用了三种颜色，应该有红、橙、绿一组漂亮的灯，可以用来做点有趣的事。

## 用 FreeBSD 做一个硬件项目

FreeBSD 有持续集成（CI）服务器，为不同架构构建 FreeBSD，以测试对代码树的每次提交。FreeBSD 使用 Jenkins 做 CI，Jenkins 提供了便利的状态 API，告诉我们构建状况以及是成功还是失败。

我们来做一个构建状态指示器，看看 FreeBSD ARMv7（BeagleBone Black 的架构）的构建状况。Jenkins 通过 JSON API 暴露构建状态，而 `textproc/jq` 提供了查询 JSON 对象的简单方式。Jenkins 返回两个与进行中构建相关的字段——`building` 和 `result`。如果 `building` 为 true，则不会有有效的 `result`。`result` 可以是 `SUCCESS`、`FAILURE`、`UNSTABLE` 或 `none`，具体取决于 `building` 状态以及最近一次构建的 `result`。

脚本需要 `jq` 工具来解析 CI 服务器返回的 JSON，还需要一套 SSL 证书与服务器认证。BeagleBone Black 没有实时时钟——你要与 CI 服务器进行 SSL 通信，必须有准确的时间。

```sh
# pkg install jq ca_root_nss
# ntpdate -s pool.ntp.org
```

```sh
#!/bin/sh
project=FreeBSD-head-armv7-build
# 把 led 配置为输出
gpioctl -f /dev/gpioc0 -c 26 OUT    # 红色
gpioctl -f /dev/gpioc1 -c 14 OUT    # 橙色
gpioctl -f /dev/gpioc2 -c 1 OUT     # 绿色
# 启动时闪烁所有 led
gpioctl -f /dev/gpioc0 26 1
gpioctl -f /dev/gpioc1 14 1
gpioctl -f /dev/gpioc2 1 1
sleep 2
# 每分钟检查一次构建状态
while true; do
echo fetching $project
output=`fetch -o - https://ci.freebsd.org/job/$project/
lastBuild/api/json 2>/dev/null`
building=`echo $output | jq ".building"`
result=`echo $output | jq ".result"`
# 清除所有 led
gpioctl -f /dev/gpioc0 26 0
gpioctl -f /dev/gpioc1 14 0
gpioctl -f /dev/gpioc2 1 0
if [ "$building" = "true" ]; then
echo $project is building
gpioctl -f /dev/gpioc1 14 1 # 橙色 led
for foo in `jot 60 1`;
do
gpioctl -f /dev/gpioc1 -t 14 # 橙色 led
sleep 0.5
done
else
echo $project done building
if [ "$result" = '"SUCCESS"' ]; then
echo build suceeded for $project
gpioctl -f /dev/gpioc2 1 1 # 绿色 led
else
echo build failed for $project
gpioctl -f /dev/gpioc0 26 1 # 红色 led
fi
fi
sleep 60  # 1 分钟
done
```

我们可以把三个 LED 接到 BeagleBone Black 的 GPIO 上——一个绿、一个红、一个橙。脚本在构建运行时让橙色 LED 闪烁，构建通过时绿色 LED 闪烁，构建失败时红色 LED 闪烁。

使用 LED 只是个例子。我们同样可以把继电器接到 GPIO 上来控制警笛，让红色 LED 在构建开始失败时把我们叫醒。 •

---

**Tom Jones** 是苏格兰阿伯丁一家黑客空间（57northhacklab.org.uk）的创始人和理事。他最初为了把一个项目从 Linux 移植过来而开始尝试在 FreeBSD 上做硬件黑客，结果一头扎进了内核黑客的世界。
