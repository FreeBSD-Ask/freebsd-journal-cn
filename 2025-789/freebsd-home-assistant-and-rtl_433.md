# FreeBSD、Home Assistant 与 rtl_433

- 原文：[FreeBSD, Home Assistant, and rtl_433](https://freebsdfoundation.org/our-work/journal/browser-based-edition/embedded-2/freebsd-home-assistant-and-rtl_433/)
- 作者：Vanja Cvelbar

在很久以前，我就开始玩 Home Assistant 了，我对自己做出的几个自动化很自豪。浴室里有个运动传感器，能开灯。如果太阳已经落山，灯会以极暗的橙色亮起；否则，它会最大化亮度并使用白光。还有个自动化，当相对湿度超过预设值时，自动开启除湿机。另外还有个安全措施。不幸的是，我们有一扇地下室窗户，如果从南边下雨，就容易被淹（虽然这种情况比较少见）。我内外各安装了一个湿度传感器；当它们被触发时，我们会在 Telegram 上收到一条消息，并能据此采取措施。整个系统托管在运行于 HAOS 的 bhyve 虚拟机中。

## 需求

在阳台和花园里有两个无线电控温度计，我希望能有个简单的概览，更重要的是一张清晰的室外温度曲线图。这些设备很简单，定期通过 433MHz 无线电发送数据。研究之后发现，用一个标准的软件无线电（SDR）适配器就能监听这些信号并进一步处理。将它与优秀的 rtl_433 和 rtl-sdr 项目结合，就是答案。

把一根基于 RTL2832 的接收棒插到笔记本电脑上后，我开始收到温度计的数据，同时也收到了邻居的设备，甚至是路过汽车的胎压监测系统（TPMS）的数据。

好在 rtl_433 软件已经支持将数据发布到 MQTT 服务器。Home Assistant 里本来就有一项用于 ZigBee 网络的 MQTT 服务，所以只需要创建一新用户来发布数据即可。


```sh
/var/log/rtl_433.log
{"time" : "2025-04-13 18:42:55.702554", "protocol" : 19, "model" : "Nexus-TH", "id" : 209, "channel" : 1, "battery_ok" : 1, "temperature_C" : 13.200, "humidity" : 61, "mod" : "ASK", "freq" : 433.910, "rssi" : -0.431, "snr" : 30.360, "noise" : -30.791}
{"time" : "2025-04-13 18:42:56.539182", "protocol" : 8, "model" : "LaCrosse-TX", "id" : 31, "temperature_C" : 11.300, "mic" : "PARITY", "mod" : "ASK", "freq" : 433.892, "rssi" : -12.208, "snr" : 19.251, "noise" : -31.459}
```

JSON 日志记录了 rtl_433 解码外部温度计无线电传输的数据。

## 第一次实现

主机架位于地下室，因此那里的无线信号很差。我把 SDR 放在一楼，并将其连接到现有的 pfSense 路由器。这需要手动安装软件包，并费力地让它在启动时运行。务实的解决方案是启动 screen 会话，在其中运行 rtl_433。

这确实有效，数据被发送到了 MQTT，Home Assistant 绘制出了我选择的温度传感器的漂亮曲线图。但这种方案不够稳定。每次重启、升级或防火墙故障后，我都得记得重新启动它。它不够可靠，我也对此不满意。

![](https://freebsdfoundation.org/wp-content/uploads/2025/10/cvelbar_chart1.png)

Home Assistant 顺利绘制出了由花园温度传感器捕获的温度曲线。

![](https://freebsdfoundation.org/wp-content/uploads/2025/10/cvelbar_chart2.png)

Home Assistant 系统捕获的两个传感器的温度曲线图。

## 第二次实现

我手头有一块闲置的 Pine A64+。我开始验证它对 FreeBSD 的支持，但很快放弃了，因为它在重启时有些问题。该问题在软件重启后仍然存在，表明是硬件问题。我记得必须物理接触主板并断开电源供应，这太麻烦，也不实用。

## EuroBSDCon 来解围

2019 年在挪威举办的 EuroBSDCon 上，Tom Jones 做了一项题为《使用 FreeBSD 进行硬件黑客入门》的教程。每个参与者都收到了一只小盒子，里面有开关、LED、电缆和其他组件，最重要的是一块 Arm SBC。

那是一块 FriendlyELEC 的 NanoPi NEOLTS。会议结束后，我用了一段时间，但现在它主要闲置在抽屉里。所以，我开始评估它作为运行 rtl_433 并获取无线数据的平台。

安装的软件已经过时，所以我买了一张新的存储卡，并按照 Tom Jones 页面上发布的说明，成功把采集系统迁移到了上面。([https://adventurist.me/posts/00297](https://adventurist.me/posts/00297))

一开始一切运行正常，也很有趣，但几天后系统突然停止工作，无法通过 SSH 访问，所以我只好给它断电重启。最好是连接一个串口控制台来找出问题所在，但当时我没时间去正确排查。

怀疑是与操作系统版本存在某些不兼容，我后来写了个粗糙的 shell 脚本，基于 Tom 的说明来自动化镜像创建。运行了几次之后，我开始把它参数化，以便更快地更换版本，并添加了一些其它实用的小功能。

![](https://freebsdfoundation.org/wp-content/uploads/2025/10/Cvelbar_Final_Implementation.png)

最终实现 —— NanoPi 与插在防火墙上的 USB SDR 接收棒。

## 做成脚本

最终版本的脚本比第一版多了几个步骤。首先，我们定义想要下载的 FreeBSD 版本和源 URL。

```sh
!/bin/sh
VERSION="14.2"
RELEASE_VERSION="$VERSION-RELEASE"
URL_IMG="https://download.freebsd.org/releases/arm/armv7/ISO-IMAGES/$VERSION/FreeBSD-$RELEASE_VERSION-arm-armv7-GENERICSD.img.xz"
URL_CHECKSUM="https://download.freebsd.org/releases/arm/armv7/ISO-IMAGES/$VERSION/CHECKSUM.SHA512-FreeBSD-$RELEASE_VERSION-arm-armv7-GENERICSD"
```

接下来，我们定义目标镜像名称、目录树、所需软件包、u-boot 引导加载程序位置、时区、要在目标镜像上安装的软件包，以及初始目标镜像大小。

```sh
TARGET_IMG="nanopi-rtl-433-FreeBSD-$RELEASE_VERSION.img"
IMG_TARGET_DIR="img-target"
IMG_MNT_DIR="img-mnt"
IMG_RELEASE_DIR="img-release"
CUSTOM_DIR="customization"
PACKAGES="u-boot-nanopi_neo-2024.07"
BOOTLOADER="/usr/local/share/u-boot/u-boot-nanopi_neo/u-boot-sunxi-with-spl.bin"
TIME_ZONE="/usr/share/zoneinfo/CET"
TARGET_PACKAGES="rtl-433 monit"
SD_SIZE="6G"
```

出于安全原因，我们会设置个性化的用户密码。

```sh
# 用户密码
TARGET_USER_PW=`date +%s | sha256sum | base64 | head -c 13`
```

接下来的步骤包括准备目录树，并定义校验和文件以及镜像的位置。

```sh
# 准备目录树
echo "Preparing the directory tree ..."
mkdir -p "$IMG_TARGET_DIR"
mkdir -p "$IMG_MNT_DIR"
mkdir -p "$IMG_RELEASE_DIR"
mkdir -p "$CUSTOM_DIR/usr/local/etc"
digest="$IMG_RELEASE_DIR/CHECKSUM.SHA512-FreeBSD-$RELEASE_VERSION-arm-armv7-GENERICSD"
image="$IMG_RELEASE_DIR/FreeBSD-$RELEASE_VERSION-arm-armv7-GENERICSD.img.xz"
digest_ext="$IMG_RELEASE_DIR/CHECKSUM.SHA512-FreeBSD-$RELEASE_VERSION-arm-armv7-GENERICSD.img"
image_ext="$IMG_RELEASE_DIR/FreeBSD-$RELEASE_VERSION-arm-armv7-GENERICSD.img"
```

镜像会下载到 `$IMG_RELEASE_DIR`，但仅当镜像的校验和与现有的不同时才会下载。

```sh
if ! [ -f $digest_ext ]; then
    echo "Digest=Initial" > $digest_ext
fi
# get the release image checksum and check the existing image
echo "Checking compressed image checksum"
fetch -o "$IMG_RELEASE_DIR" "$URL_CHECKSUM"
if ! sha512 -q -c $(cut -f2 -d= $digest) $image; then
   echo "Download compressed image"
   fetch -o "$IMG_RELEASE_DIR" "$URL_IMG"
   # reset extracted image checksum
   echo "Digest=Refresh" > $digest_ext
   rm -f $image_ext
else
   echo "Compressed image OK"
fi
```

如果镜像的校验和与现有的不一致，就会解压该镜像。

```sh
# 提取镜像
echo "Checking extracted image ..."
if ! sha512 -q -c $(cut -f2 -d= $digest_ext) $image_ext; then
   echo "Extracting image and creating checksum"
   rm -f $image_ext
   xz -dk $image
   sha512 $image_ext > $digest_ext
else
   echo "Extracted image OK"
fi
```

现在会安装主机系统所需的软件包。

```sh
# 安装所需的软件包
echo "Installing packages"
pkg install -y $PACKAGES
```

会创建一个使用上述定义大小的目标镜像，并将其连接到一个内存磁盘。

```sh
# 创建目标镜像
echo "Create target image and connect to the memory disk"
truncate -s $SD_SIZE $IMG_TARGET_DIR/$TARGET_IMG
mdisk=`mdconfig -f $IMG_TARGET_DIR/$TARGET_IMG`
echo $mdisk
```

我们将镜像和引导加载程序写入目标。

```sh
# 将解压后的镜像写入内存盘
echo "Writing the extracted image to the memory disk"
dd if=$image_ext of=/dev/$mdisk bs=1m
# write the bootloader to the memory disk
echo "Writing the bootloader to the memory disk"

if [ -f $BOOTLOADER ]; then
   dd if=$BOOTLOADER of=/dev/$mdisk bs=1k seek=8 conv=sync
else
   echo "The bootloader $BOOTLOADER does not exist. Aborting ..."
   exit
fi
```

最后，目标镜像会被挂载，并使用 **pkg–chroot** 选项安装软件包，这使我们能够优雅地为不同的硬件平台安装软件包。需要注意的是，目标系统上必须存在正确的 **resolv.conf** 文件。

```sh
# 挂载目标镜像
echo "Mounted target image on $IMG_MNT_DIR"
mount /dev/"$mdisk"s2a $IMG_MNT_DIR

# 安装目标软件包
echo "Installing packages on target"
cp /etc/resolv.conf $IMG_MNT_DIR/etc/
pkg --chroot $IMG_MNT_DIR install -y $TARGET_PACKAGES
```

正在目标系统上启用服务。同样，我们使用 **sysrc** 工具的一个便捷选项，直接在已挂载的镜像上访问配置文件。虽然不是严格必须的，但我们还启用了远程 syslog，它应在目标主机上通过 `-a <peer>` 启用。这一功能是在调试系统时启用的，当时系统已经挂起了一天半，如上所述。

```sh
# 对目标启用服务
sysrc -f $IMG_MNT_DIR/etc/rc.conf hostname="nanopi" ntpdate_enable="YES"
ntpd_enable="YES" rtl_433_enable="YES" monit_enable="YES" syslogd_enable="YES"
syslogd_flags="-s -v -v"
```

在自定义目录中，我们复制系统树中所需的部分：

```sh
customization/
├── home
│  └─ freebsd
│      └─ .ssh
│           └─ authorized_keys
├── usr
│  └─ local
│       └─ etc
│           ├─ monitrc
│           ├─ rtl_433.conf
│           └─ syslog.d
│               └─ remote_anirul.d112.conf
└── var
    └─ log
        └─ rtl_433.log
```

```sh
# 复制自定义内容
echo "Copy $CUSTOM_DIR to $IMG_MNT_DIR"
cp -a  $CUSTOM_DIR/* $IMG_MNT_DIR
```

时区的设置很简单，只需将定义复制到目标系统的 `/etc/localtime` 即可。

有趣的是 —— 在桌面机器上复制 CET 定义会导致 Mozilla 产品出现问题。比如在 Thunderbird 中，邮件列表里的时间会显示为 UTC，但打开邮件后，时间又显示正确。在这种情况下，更好的做法是使用 **tzsetup** 并指定正确的城市，或者复制某个城市的时区定义。我至今仍需要想办法报告这个 bug。

```sh
# 设置时区
echo “Set the timezone: copy $IMG_MNT_DIR/$TIME_ZONE to $IMG_MNT_DIR/etc/localtime”
cp $IMG_MNT_DIR/$TIME_ZONE $IMG_MNT_DIR/etc/localtime
```

接下来，会为 FreeBSD 用户设置新密码，并将其复制到主目录中的一个文件，以及镜像文件旁边。最后的清理步骤包括卸载镜像、移除内存磁盘，并向用户提供一些最终的使用说明。

```sh
# 配置用户
echo "Configuring users on target"
# change freebsd user password
echo "$TARGET_USER_PW" | pw -R $IMG_MNT_DIR mod user freebsd -h 0
echo "$TARGET_USER_PW" > $IMG_MNT_DIR/"home/freebsd/.pwd"

# 卸载目标镜像
echo "Unmounted target image from $IMG_MNT_DIR"
umount $IMG_MNT_DIR

# 创建密码文件
echo "$TARGET_USER_PW" > $IMG_TARGET_DIR/$TARGET_IMG".pwd"

# 移除内存盘
echo "Removing memory disk $mdisk"
mdconfig -d -u $mdisk

# 最终说明
echo "You can copy the image with \"dd if=$IMG_TARGET_DIR/$TARGET_IMG of=/dev/<USB Disk> status=progress bs=1m\""
echo "The password for the user freebsd is \"$TARGET_USER_PW\""
echo "The password is also available in the file \""$IMG_TARGET_DIR/$TARGET_IMG".pwd\""
```

在自定义部分中，我们准备了服务的配置文件。**Monit** 用于监控 **rtl_433** 的状态，并在需要时重新启动它。下面是 `monitrc` 中定义 **rtl_433** 服务的几行配置：

```sh
# rtl_433
check process rtl_433 with pidfile /var/run/rtl_433.pid
start program = "/usr/local/etc/rc.d/rtl_433 start" with timeout 60 seconds
stop program = "/usr/local/etc/rc.d/rtl_433 stop"
```

当然，我们还必须配置 **rtl_433**。与示例配置相比，差异如下：

```sh
output mqtt://<homeassistant>,usr=<mqtt user>,pass=<mqtt user pwd> ,retain=0,devices=rtl_433[/model][/id]
output json:/var/log/rtl_433.log
output log
```

输出被发送到可供 Home Assistant 访问的 MQTT broker，在我的场景中，它运行在同一台虚拟机中。**retain** 标志被设为 `false`，因为我不希望用瞬时设备的数据弄乱 MQTT 主题，并且可以接受在 Home Assistant 中重新发现设备的速度慢一些。

JSON 输出也会写入 `/var/log` 中的 JSON 文件。

## 经验教训

rtl_433 配置文件中的最后一行才是关键。在我的原型系统里，日志被写入 `/tmp`，大约 36 小时就会填满，从而导致系统宕机。所以，如果你不打算定期清理，千万不要把日志写到 `/tmp`。

硬件必须可靠。我在 Pine A64+ 上遇到了很多问题，尤其是它的重启问题。

我学到了不少 FreeBSD 基础配置的知识，再次体会到了其工具的优雅之处。

## 当前状态

自上次暴风雨和停电以来，该系统已经保持了 57 天的正常运行时间。在那次事件中，它在无人干预的情况下自动完成了启动。该系统可靠、易于复制，并且能够轻松升级到新的 FreeBSD 版本。只需更换 SD 卡，就可以轻松测试新版本或新配置，并在需要时恢复到原有状态。

日后，我希望探索使用 **nanobsd** 进行开发的可能性，并考虑这个项目是否可以作为深入研究 **Crochet/Poudriere 镜像创建**的一个好机会。

---

**Vanja Cvelbar** 是意大利国家研究委员会材料车间研究所的技术合作者，担任 IT 组负责人，负责管理支撑科研与运行的 IT 基础设施与服务。他的专长涵盖系统管理、网络以及开源技术。在以 Linux 为主的工作二十年后，他又回归了 FreeBSD —— 他的首个系统是 3.2 版。如今，他专注于部署和维护基于 FreeBSD 的环境，为研究所提供核心 IT 服务。
