# 使用 bhyve 进行 FreeBSD 开发

- 原文：[Using bhyve for FreeBSD Development](https://freebsdfoundation.org/our-work/journal/browser-based-edition/virtualization/using-bhyve-for-freebsd-development/)
- 作者：**John Baldwin**

FreeBSD 10.0 中令人兴奋的新特性之一是 bhyve 虚拟机监控器。虚拟机监控器和虚拟机广泛应用于多种场景。本文聚焦于如何将 bhyve 用作辅助开发 FreeBSD 本身的工具。其中涉及的细节并非全部特定于 FreeBSD 开发，许多内容对其他应用也有用。

> 注意：bhyve 虚拟机监控器仍在持续开发中，本文描述的部分特性是在 FreeBSD 10.0 发布之后加入的。这些特性大部分应会出现在 FreeBSD 10.1 中。

## 虚拟机监控器

bhyve 虚拟机监控器需要带硬件虚拟化支持的 64 位 x86 处理器。该要求让虚拟机监控器实现保持简单、干净，但需要相当新的处理器。当前的虚拟机监控器需要 Intel 处理器，但有一个支持 AMD 处理器的活跃开发分支。

虚拟机监控器本身包含用户态和内核两部分。内核驱动位于 `vmm.ko` 模块中，可在启动时从 boot loader 加载，也可在运行时加载。在创建任何客户机之前必须先加载该模块。创建客户机时，内核驱动会在 **/dev/vmm** 中创建一个设备文件，用户程序通过它来与客户机交互。

主要用户态组件是 `bhyve(8)` 程序。它构造客户机中的模拟设备树，并提供大多数模拟设备的实现。它还调用内核驱动来执行客户机。注意，客户机始终在驱动程序内部执行，因此客户机执行时间在宿主机中算作 bhyve 进程的系统时间。目前，bhyve 不向客户机提供系统固件接口（既非 BIOS 也非 UEFI）。取而代之的是，宿主机上运行的一个用户程序用于执行启动时操作，包括将客户机操作系统内核加载到客户机内存中，并设置客户机的初始状态，使其从内核入口点开始执行。对于 FreeBSD 客户机，可使用 `bhyveload(8)` 程序加载内核并为客户机执行做好准备。对其他一些操作系统的支持可通过 grub2-bhyve 程序获得，该程序可通过 **sysutils/grub2-bhyve** port 或预构建包获得。

FreeBSD 10.0 中的 `bhyveload(8)` 程序只支持 64 位客户机。对 32 位客户机的支持将包含在 FreeBSD 10.1 中。

## 网络设置

客户机和宿主机之间的网络连接可以用多种方式配置。下面介绍两种不同的设置，但它们并非唯一可能的配置。

bhyve 目前支持的唯一客户机网络驱动程序是 VirtIO 网络接口驱动。每个暴露给客户机的网络接口在宿主机上关联一个 `tap(4)` 接口。`tap(4)` 驱动允许用户程序向网络栈注入数据包，并从网络栈接收数据包。设计上，每个 `tap(4)` 接口只有在被用户进程打开并通过 `ifconfig(8)` 由管理员启用时才会传递流量。因此，每次启动客户机时都必须显式启用每个 `tap(4)` 接口。这对频繁重启的客户机来说可能不便。可以将 `tap(4)` 驱动设置为在用户进程打开接口时自动启用，方法是设置 sysctl 变量 `net.link.tap.up_on_open` 为 1。

### 桥接配置

一种简单的网络设置是将客户机的网络接口直接桥接到宿主机所连接的网络。在宿主机上创建一个 `if_bridge(4)` 接口。将客户机的 `tap(4)` 接口与宿主机上连接到目标网络的网络接口一起添加到桥接。**示例 1** 将使用 tap0 的客户机连接到宿主机 re0 接口所在的 LAN：

```sh
# ifconfig bridge0 create
# ifconfig bridge0 addm re0
# ifconfig bridge0 addm tap0
# ifconfig bridge0 up
```

**示例 1：手动将客户机连接到宿主机的 LAN**

随后，客户机可以通过 DHCP 或静态地址配置绑定到 tap0 的 VirtIO 网络接口，使其使用宿主机 re0 接口所在的 LAN。

**/etc/rc.d/bridge** 脚本允许通过 **/etc/rc.conf** 中的变量在启动时自动配置桥接。`autobridge_interfaces` 变量列出要配置的桥接接口。对每个桥接接口，`autobridge_<name>` 变量列出应作为桥接成员添加的其他网络接口。列表可包含 shell 通配符以匹配多个接口。注意，**/etc/rc.d/bridge** 不会创建指定的桥接接口——它们应当与所需的 `tap(4)` 接口一起在 `cloned_interfaces` 变量中列出。**示例 2** 列出了创建三个桥接到宿主机 re0 接口所在本地 LAN 的 `tap(4)` 接口所需的 **/etc/rc.conf** 设置。

```ini
autobridge_interfaces="bridge0"
autobridge_bridge0="re0 tap*"
cloned_interfaces="bridge0 tap0 tap1 tap2"
ifconfig_bridge0="up"
```

**示例 2：桥接配置**

### 带 NAT 的私有网络

更复杂的网络设置在宿主机上为客户机创建一个私有网络，并使用网络地址转换（NAT）提供从客户机到其他网络的有限访问。当宿主机移动或连接到不受信任的网络时，这种设置可能更合适。

此设置同样使用 `if_bridge(4)` 接口，但只将客户机使用的 `tap(4)` 接口添加为桥接成员。从私有子网为桥接成员分配地址。桥接接口也从私有子网获取地址，以将桥接连接到宿主机的网络栈。这让客户机和宿主机能在桥接使用的私有子网上通信。

宿主机充当路由器，让客户机可以访问远程系统。宿主机上启用 IP 转发，客户机连接通过 `natd(8)` 转换。客户机将宿主机在私有子网上的地址用作默认路由。**示例 3** 列出了创建三个 `tap(4)` 接口和一个使用 192.168.16.0/24 子网的桥接接口的 **/etc/rc.conf** 设置。它还通过 `natd(8)` 转换宿主机 wlan0 接口上的网络连接。

```ini
autobridge_interfaces="bridge0"
autobridge_bridge0="tap*"
cloned_interfaces="bridge0 tap0 tap1 tap2"
ifconfig_bridge0="inet 192.168.16.1/24"
gateway_enable="YES"
natd_enable="YES"
natd_interface="wlan0"
firewall_enable="YES"
firewall_type="open"
```

**示例 3：私有网络配置**

## 将 dnsmasq 与私有网络配合使用

上一示例中的私有网络工作良好，但操作起来有些繁琐。客户机必须静态配置其网络接口，且客户机与宿主机之间的网络连接必须使用硬编码的 IP 地址。`dnsmasq` 工具可以减轻大量繁琐工作。它可通过 **dns/dnsmasq** port 或预构建包安装。

`dnsmasq` 守护进程同时提供 DNS 转发服务器和 DHCP 服务器。它可以处理本地 DNS 请求，将其 DHCP 客户端的主机名映射到所分配的地址。对于私有网络设置，这意味着每个客户机可使用 DHCP 配置其网络接口。此外，所有客户机和宿主机都能解析每个客户机的主机名。

`dnsmasq` 守护进程通过 **/usr/local/etc/dnsmasq.conf** 配置文件中的设置进行配置。port 安装时会安装一份示例配置。该配置建议为 DNS 服务器启用 `domain-needed` 和 `bogus-priv` 设置，以避免向上游 DNS 服务器发送无用的 DNS 请求。要启用 DHCP 服务器，必须将 `interface` 设置为宿主机上应运行服务器的网络接口，并将 `dhcp-range` 设置为可分配给客户机的 IP 地址范围。

**示例 4** 指示 `dnsmasq` 守护进程在 bridge0 接口上运行 DHCP 服务器，并将 192.168.16.0/24 子网的一部分分配给客户机。

```ini
domain-needed
bogus-priv
interface=bridge0
dhcp-range=192.168.16.10,192.168.16.200,12h
```

**示例 4：启用 dnsmasq 的 DNS 和 DHCP 服务器**

除了为 DHCP 客户端提供本地 DNS 名称外，dnsmasq 还为宿主机上 **/etc/hosts** 中的任何条目提供 DNS 名称。在 **/etc/hosts** 中添加一条将分配给 bridge0 的 IP 地址映射到主机名（例如“host”）的条目，可让客户机使用该主机名联系宿主机。

剩下最后一件事：配置宿主机使用 dnsmasq 的 DNS 服务器。让宿主机使用 dnsmasq 的 DNS 服务器后，宿主机就能解析每个客户机的名称。`dnsmasq` 守护进程可以使用 `resolvconf(8)` 无缝处理由 DHCP 或 VPN 客户端提供的宿主机 DNS 配置更新。这是通过 `resolvconf(8)` 更新两个配置文件实现的，dnsmasq 在宿主机 DNS 配置变更时读取这两个文件。最后，宿主机应始终使用 dnsmasq 的 DNS 服务器，并依赖它将请求转发给其他上游 DNS 服务器。

要启用这一切，需对 dnsmasq 的配置文件和 **/etc/resolvconf.conf** 做出修改。关于配置 `resolvconf(8)` 的更多细节可在 `resolvconf.conf(5)` 中找到。**示例 5** 给出了让 dnsmasq 作为宿主机名称解析器的两个文件改动。

```ini
# /usr/local/etc/dnsmasq.conf
conf-file=/etc/dnsmasq-conf.conf
resolv-file=/etc/dnsmasq-resolv.conf
```

```ini
# /etc/resolvconf.conf
name_servers=127.0.0.1
dnsmasq_conf=/etc/dnsmasq-conf.conf
dnsmasq_resolv=/etc/dnsmasq-resolv.conf
```

**示例 5：让 dnsmasq 作为宿主机的解析器**

## 通过 vmrun.sh 运行客户机

执行客户机需要几步。首先，必须清除任何使用相同名称的先前客户机留下的状态——通过向 `bhyvectl(8)` 传递 `--destroy` 标志完成。其次，必须创建客户机并通过 `bhyveload(8)` 或 grub2-bhyve 将客户机内核加载到其地址空间。最后，使用 `bhyve(8)` 程序创建虚拟设备并为客户机执行提供运行时支持。每次启动客户机都手动完成这些步骤会有些繁琐，因此 FreeBSD 为 FreeBSD 客户机附带了一个包装脚本：**/usr/share/examples/bhyve/vmrun.sh**。

`vmrun.sh` 脚本管理简单的 FreeBSD 客户机。它在循环中执行上述三个步骤，使客户机在重启后像真实硬件一样重新启动。它向客户机提供一组固定的虚拟设备，包括由 `tap(4)` 接口支撑的网络接口、由磁盘镜像支撑的本地磁盘和可选的由安装镜像支撑的第二磁盘。为简化客户机安装，vmrun.sh 会检查提供的磁盘镜像是否有有效的引导扇区。如果没有找到，它会指示 `bhyveload(8)` 从安装镜像启动，否则从磁盘镜像启动。在 FreeBSD 10.1 及以后版本中，当客户机通过 ACPI 请求软关机时，vmrun.sh 会终止其循环。

引导新的 FreeBSD 客户机最简单的方法是从安装 ISO 镜像安装客户机。对于运行 9.2 或更高版本的 FreeBSD 客户机，可使用标准安装流程，将常规安装 ISO 作为传给 vmrun.sh 的可选安装镜像。FreeBSD 8.4 也可以作为 bhyve 客户机运行。但它的安装程序不完全支持 VirtIO 块设备，因此初始安装必须按 RootOnZFS 指南中类似的步骤手动执行。**示例 6** 创建名为“vm0”的 64 位客户机并引导 FreeBSD 10.0-RELEASE 安装 CD。客户机安装完成后，可去掉 `-I` 参数从磁盘镜像启动客户机。

```sh
# mkdir vm0
# truncate -s 8g vm0/disk.img
# sh /usr/share/examples/bhyve/vmrun.sh -t tap0 -d vm0/disk.img \
  -I FreeBSD-10.0-RELEASE-amd64-disc1.iso vm0
```

**示例 6：创建 FreeBSD/amd64 10.0 客户机**

`vmrun.sh` 脚本同步运行 `bhyve(8)`，并使用其标准文件描述符作为分配给客户机的第一个串行端口的后端。该串行端口用作 FreeBSD 客户机的系统控制台设备。在后台运行客户机的最简单方法是使用 `screen` 或 `tmux` 这样的工具。FreeBSD 10.1 及以后版本将发送给 `bhyve(8)` 的 SIGTERM 信号视为虚拟电源按钮。如果客户机支持 ACPI，则发送 SIGTERM 会中断客户机以请求干净关机。客户机随后应启动 ACPI 软关机，这将终止 vmrun.sh 循环。如果客户机不响应 SIGTERM，仍可通过 SIGKILL 从宿主机强制终止客户机。如果客户机不支持 ACPI，SIGTERM 将立即终止客户机。

`vmrun.sh` 脚本接受几个不同的参数来控制 `bhyve(8)` 和 `bhyveload(8)` 的行为，但这些参数只能启用这些程序所支持特性的一个子集。要控制所有可用特性或使用其他虚拟设备配置（例如多个虚拟驱动器或网络接口），要么手动调用 `bhyveload(8)` 和 `bhyve(8)`，要么以 vmrun.sh 为基础编写自定义脚本。

## 配置客户机

FreeBSD 客户机不需要大量配置设置即可运行，大多数设置可由系统安装程序设置。不过有几个约定和额外设置可能有用。

开箱即用时，9.3 和 10.1 之前的 FreeBSD 发行版期望使用视频控制台和键盘作为系统控制台。因此它们不会在串行控制台上启用登录提示。通过编辑 **/etc/ttys** 并将 ttyu0 终端标记为“on”，可在串行控制台上启用登录提示。注意，这可以在安装完成后从宿主机完成，方法是使用 `mdconfig(8)` 在宿主机上挂载磁盘镜像。

> 注意：在宿主机上挂载客户机文件系统之前，务必确保客户机已不再访问该磁盘镜像，以避免数据损坏。

如果客户机需要网络访问，则需要类似正常宿主机的配置。这包括配置客户机的网络接口 `vtnet0` 并分配主机名。有用的约定是将客户机的名称（**示例 6** 中的“vm0”）复用为主机名。`sendmail(8)` 守护进程在引导时尝试解析客户机主机名可能会挂起。可通过在客户机中完全禁用 `sendmail(8)` 来规避此问题。最后，大多数具有网络访问的客户机需要启用通过 `sshd(8)` 的远程登录。**示例 7** 列出了简单 FreeBSD 客户机的 **/etc/rc.conf** 文件。

```ini
hostname="vm0"
ifconfig_vtnet0="DHCP"
sshd_enable="YES"
dumpdev="AUTO"
sendmail_enable="NONE"
```

**示例 7：简单的 FreeBSD 客户机配置**

## 将 bhyve 客户机用作调试目标

bhyve 在开发 FreeBSD 时的用法是让宿主机像对待远程目标一样调试客户机。具体而言，可以在宿主机上构建测试内核，在客户机内部启动它，并从宿主机使用 `kgdb(1)` 调试。一旦客户机创建并配置完毕，且测试内核已在宿主机上构建完成，下一步就是在客户机中启动测试内核。

传统方法是将内核安装到客户机的文件系统中——可以通过 NFS 将构建目录导出给客户机，通过网络将内核复制到客户机，或在宿主机上通过 `mdconfig(8)` 直接挂载客户机文件系统。另一种方法可通过 `bhyveload(8)` 实现，类似于通过网络启动测试机。

### 通过 bhyveload(8) 的宿主机文件系统引导测试内核

`bhyveload(8)` 程序允许将宿主机文件系统上的一个目录导出到 loader 环境。这可用于从宿主机文件系统而非客户机的磁盘镜像加载内核和模块。宿主机文件系统上的目录通过 `-h` 标志传递给 `bhyveload(8)`。`bhyveload(8)` 向 loader 环境导出一个 `host0:` 设备。loader 环境中传递给 `host0:` 设备的路径会附加到所配置的目录以生成宿主机路径名。注意，传递给 `bhyveload(8)` 的目录必须是绝对路径名。FreeBSD 10.1 及以后版本中的 `vmrun.sh` 脚本允许通过 `-H` 参数设置该目录。脚本会在传递给 `bhyveload(8)` 之前将相对路径名转换为绝对路径名。

从宿主机在客户机内引导测试内核涉及以下三步：

1. 通过将 `DESTDIR` 变量设为该目录，在调用 `make install` 或 `make installkernel` 时将内核安装到宿主机上的目录。对该目录有写访问权的非 root 用户可通过设置 `KMODOWN` make 变量直接执行此步骤。
2. 通过 `-h` 标志传递给 `bhyveload(8)`，或通过 `-H` 标志传递给 `vmrun.sh`，将该目录的路径传给 `bhyveload(8)`。
3. 在 `bhyveload(8)` 提示符下显式加载新内核，路径为 `host0:/boot/kernel/kernel`。

**示例 8** 将配置为 GUEST 的内核安装到客户机“vm0”的宿主机目录。它使用 vmrun.sh 的 `-H` 参数指定传递给 `bhyveload(8)` 的宿主机目录。它还展示了在 loader 提示符下引导测试内核所使用的命令。

```sh
> cd ~/work/freebsd/head/sys/amd64/compile/GUEST
> make install DESTDIR=~/bhyve/vm0/host KMODOWN=john
...
> cd ~/bhyve
> sudo sh vmrun.sh -t tap0 -d vm0/disk.img -H vm0/host vm0
...
OK unload
OK load host0:/boot/kernel/kernel
host0:/boot/kernel/kernel text=0x523888 data=0x79df8+0x10e2e8 syms=[0x8+0x9fb58+0x8+0xbaf41]
OK boot
...
Copyright (c) 1992-2014 The FreeBSD Project.
Copyright (c) 1979, 1980, 1983, 1986, 1988, 1989, 1991, 1992, 1993, 1994
The Regents of the University of California. All rights reserved.
FreeBSD is a registered trademark of The FreeBSD Foundation.
FreeBSD 11.0-CURRENT #6 r261528M: Fri Feb 7 09:55:45 EST 2014
john@pippin.baldwin.cx:/usr/home/john/work/freebsd/head/sys/amd64/compile/GUEST amd64
```

**示例 8：从宿主机引导内核**

也可以使用 `nextboot(8)` 配置客户机在下一次启动时从 `host0:` 文件系统加载内核。运行 `nextboot -e bootfile=host0:/boot/kernel/kernel` 后重启即可引导 `host0:/boot/kernel/kernel` 内核。注意，这不会调整用于加载内核模块的模块路径，因此仅适用于单体内核。

### 使用 bhyve(8) 的调试端口

`bhyve(8)` 虚拟机监控器提供可选的调试端口，可供宿主机使用 `kgdb(1)` 调试客户机内核。要使用此功能，客户机内核必须包含 `bvmdebug` 设备驱动、KDB 内核调试器和 GDB 调试后端。调试端口还必须通过向 `bhyve(8)` 传递 `-g` 标志启用。该标志需要一个参数，指定 `bhyve(8)` 监听 `kgdb(1)` 连接的本地 TCP 端口。`vmrun.sh` 脚本也接受一个 `-g` 标志，传递给 `bhyve(8)`。

客户机启动时，其内核会自动检测调试端口作为可用的 GDB 后端。要在宿主机上将 `kgdb(1)` 连接到客户机，首先通过将 `debug.kdb.enter` sysctl 节点设为非零值进入内核调试器。在调试器提示符下，调用 `gdb` 命令。在宿主机上，使用客户机的内核作为内核镜像运行 `kgdb(1)`。可使用 `target remote` 命令连接到传递给 `bhyve(8)` 的 TCP 端口。一旦 `kgdb(1)` 附加到远程目标，就可用来调试客户机内核。**示例 9** 和 **示例 10** 演示了在宿主机上构建客户机内核的这些步骤。

```sh
> sudo sh vmrun.sh -t tap0 -d vm0/disk.img -H vm0/host -g 1234 vm0
...
OK load host0:/boot/kernel/kernel
host0:/boot/kernel/kernel text=0x523888 data=0x79df8+0x10e2e8 syms=[0x8+0x9fb58+0x8+0xbaf41]
OK boot
Booting...
GDB: debug ports: bvm
GDB: current port: bvm
...
root@vm0:~ # sysctl debug.kdb.enter=1
debug.kdb.enter: 0KDB: enter: sysctl debug.kdb.enter
[ thread pid 693 tid 100058 ]
Stopped at kdb_sysctl_enter+0x87: movq $0,kdb_why
db> gdb
(ctrl-c will return control to ddb)
Switching to gdb back-end
Waiting for connection from gdb
-> 0
root@vm0:~ #
```

**示例 9：使用 kgdb(1) 配合 bvmdebug：客户机内**

```sh
> cd ~/work/freebsd/head/sys/amd64/compile/GUEST
> kgdb kernel.debug
...
(kgdb) target remote localhost:1234
Remote debugging using localhost:1234
warning: Invalid remote reply:
kdb_sysctl_enter (oidp=<value optimized out>, arg1=<value optimized out>,
arg2=1021, req=<value optimized out>) at ../../../kern/subr_kdb.c:446
446 kdb_why = KDB_WHY_UNSET;
Current language: auto; currently minimal
(kgdb) c
Continuing.
```

**示例 10：使用 kgdb(1) 配合 bvmdebug：宿主机上**

### 通过虚拟串行端口使用 kgdb(1)

串行端口也可用于让宿主机调试客户机内核。这可通过在宿主机上加载 `nmdm(4)` 驱动并为用于调试的串行端口使用 `nmdm(4)` 设备来实现。

为避免在控制台上输出垃圾字符，将 `nmdm(4)` 设备连接到第二个串行端口。在虚拟机监控器中通过向 `bhyve(8)` 传递 `-l com2,/dev/nmdm0B` 启用此功能。客户机必须通过从 `bhyveload(8)` 设置内核环境变量 `hint.uart.1.flags=0x80` 来配置使用第二个串行端口进行调试。宿主机上的 `kgdb(1)` 调试器通过 `target remote /dev/nmdm0A` 连接到客户机。

## 结论

bhyve 虚拟机监控器是 FreeBSD 开发者工具箱的不错补充。客户机既可用来开发新特性，也可用来测试对 stable 分支的合并。该虚拟机监控器在 FreeBSD 开发之外也有广泛用途。

---

**John Baldwin** 于 1999 年作为提交者加入 FreeBSD 项目。他在系统的多个领域工作过，包括 SMP 基础设施、网络协议栈、虚拟内存和设备驱动支持。John 曾在核心和发布工程团队任职，并每年春季组织 FreeBSD 开发者峰会。
