# FreeBSD 内核开发工作流程

- 原文链接：[FreeBSD Kernel Development Workflow](https://freebsdfoundation.org/our-work/journal/browser-based-edition/development-workflow-and-ci/freebsd-kernel-development-workflow/)
- 作者：Navdeep Parhar

如同其他软件那样，内核亦采用传统的工作流程进行开发，即克隆、编辑、构建、测试、调试和提交。但与用户空间软件不同，内核开发必然涉及反复重启（以及死锁和内核 panic），而且没有专用的测试系统，开发过程会非常不便。本文介绍了一种适用于内核开发的实用设置，使用虚拟机进行测试和调试。

基于虚拟机的内核开发设置有诸多优点：

- **隔离性**：虚拟机重启和崩溃不会影响主机系统。
- **速度**：虚拟机的重启速度远快于裸系统。
- **可调试性**：虚拟机易于设置用于实时源代码级的内核调试。
- **灵活性**：虚拟机可以用来构建“盒中网络”，进行网络代码的开发，而无需实际的物理网络。例如，可以在飞机上的笔记本里进行开发。
- **可管理性**：虚拟机易于创建、重新配置、克隆和迁移。

## 概述

用于构建内核的系统还运行着测试虚拟机，所有虚拟机都连接到一个内部桥接网络。主机提供 DHCP、DNS 和其他服务，供配置在桥接网络上的 IP 网络使用。源代码和构建产物都保存在主机上的一个自包含工作区域内，并通过内部网络导出。工作区域挂载在测试虚拟机内，并从那里安装新的测试内核。虚拟机配置有额外的串口用于远程内核调试，其中虚拟机内核中的 gdb 调试器与主机系统上的 gdb 客户端通过虚拟空调制解调器连接。

最简单的设置方式是使用一台系统（通常是笔记本和工作站）来处理所有任务。这是最便捷的开发环境，但该设备必须有足够的资源（CPU、内存和存储）来构建 FreeBSD 源代码、运行虚拟机。

在工作环境中，更常见的是有专用的开发和测试服务器，分别与开发人员的工作站分离。服务器级系统比桌面系统规格更高，更适合构建源代码和运行虚拟机。它们还提供了更多种类的 PCIe 扩展槽，非常适合 PCIe 设备驱动开发。

![](https://freebsdfoundation.org/wp-content/uploads/2024/05/dev_workflow_redrawn.png)

## 配置

本文其余部分假设使用两台系统，并使用主机名“desktop”和“builder”来指代它们。源代码的主副本存储在 desktop 上，用户在此编辑并同步到 builder。接下来的操作都在 builder 上以 root 身份进行。builder 在其内部网络中也被称为“vmhost”。

```sh
WS=dev
DWSDIR=~/work/ws
WSDIR=/ws
```

在 desktop 上的所有检出的代码都假设位于一个公共父目录 `${DWSDIR}` 下的 `${WS}` 目录中。示例使用位于 `~user/work/ws` 目录中的工作区“dev”。builder 上的目录 `${WSDIR}` 是一个自包含的工作区，包含构建配置文件、源代码、共享的目录 obj 以及 gdb 的 sysroot。示例中使用 builder 上的 `/ws` 路径。

## 桌面设置

### 源代码

可以在 [https://git.FreeBSD.org/src.git](https://git.freebsd.org/src.git) 下载 FreeBSD 源代码。通常在主分支上进行新的开发工作，适合最新稳定分支的更改会在经过一段时间的稳定期后从主分支合并回去。

通过克隆官方仓库或镜像中的分支来创建本地工作副本。

```sh
desktop# pkg install git

desktop$ git ls-remote https://git.freebsd.org/src.git heads/main heads/stable/*

desktop$ git clone --single-branch -b main ${REPO} ${DWSDIR}/${WS}
desktop$ git clone --single-branch -b main https://git.freebsd.org/src.git ~/work/ws/dev
```

### 开发专用的自定义 KERNCONF

每个内核都是根据一个纯文本的内核配置（KERNCONF）文件构建的。在传统上，内核的标识字符串（`uname -i` 的输出）与其配置文件的名称相匹配。例如，GENERIC 内核是从名为 GENERIC 的文件构建的。为了调试和诊断，KERNCONF 中有许多参数，在早期开发过程中启用这些参数非常有用。主分支中的 GENERIC 配置已经启用了一个恰当的子集，适合开发工作。然而，现代编译器似乎会在低优化级别下也会 aggressively 优化掉变量和其他调试信息，因此有时需要禁用所有优化来构建内核。可以使用此处显示的自定义 KERNCONF（名为 DEBUG0）来实现这一目的。一般，包含现有配置再使用 `nooptions/options` 和 `nomakeoptions/makeoptions` 来进行调整，回比从头开始编写配置文件更为简单。

参数 DEBUG 会被添加到内核和模块的编译参数中。需增加堆栈大小，以适应未优化代码的更大堆栈占用。

```sh
desktop$ cat ${DWSDIR}/${WS}/sys/amd64/conf/DEBUG0
desktop$ cat ~/work/ws/dev/sys/amd64/conf/DEBUG0
include GENERIC
ident DEBUG0
nomakeoptions DEBUG
makeoptions DEBUG=”-g -O0”
options KSTACK_PAGES=16
```

### 将源代码传输到构建机

将源代码复制到构建机，在构建机上以 root 用户身份进行构建。切记，在对桌面上的源代码进行更改、在构建机上进行构建之前，要先同步内容。

```sh
desktop# pkg install rsync

desktop$ rsync -azO --del --no-o --no-g ${DWSDIR}/${DWS} root@builder:${WSDIR}/src/
desktop$ rsync -azO --del --no-o --no-g ~/work/ws/dev root@builder:/ws/src/
```

## 构建机设置

### 构建配置

在构建机的工作区中创建文件 `make.conf` 和 `src.conf`，而不要去修改 `/etc` 中的全局配置文件。目录 `obj` 也位于工作区，而非 `/usr/obj`。使用 meta 模式进行快速增量重建。meta 模式需要 `filemon`。在默认情况下，`KERNCONF=` 列表中的所有内核都会被构建，第一个内核会被安装。可以在命令行中用一个 KERNCONF 来覆盖默认设置。

```sh
builder# kldload -n filemon
builder# sysrc kld_list+=”filemon”

builder# mkdir -p $WSDIR/src $WSDIR/obj $WSDIR/sysroot
builder# mkdir -p /ws/src /ws/obj /ws/sysroot

builder# cat $WSDIR/src/src-env.conf
builder# cat /ws/src/src-env.conf
MAKEOBJDIRPREFIX?=/ws/obj
WITH_META_MODE=”YES”
```

```sh
builder# cat /ws/src/make.conf
KERNCONF=DEBUG0 GENERIC-NODEBUG
INSTKERNNAME?=dev
```

```sh
builder# cat /ws/src/src.conf
WITHOUT_REPRODUCIBLE_BUILD=”YES”
```

### 网络配置

首先，选择一个未使用的网络和子网掩码来作为内部网络。示例中使用的是 `192.168.200.0/24`。第一个主机（192.168.200.1）始终是虚拟机主机（构建机）。两位数的主机号保留给已知虚拟机。三位数的主机号（例如 100）由 DHCP 服务器分配给未知虚拟机。

1. 为作为虚拟交换机连接所有虚拟机和主机的桥接接口创建桥接接口，同时为桥接接口分配固定的 IP 地址和主机名。

   ```sh
   builder# echo '192.168.200.1 vmhost' >> /etc/hosts
   builder# sysrc cloned_interfaces="bridge0"
   builder# sysrc ifconfig_bridge0="inet vmhost/24 up"
   builder# service netif start bridge0
   ```

2. 配置主机进行 IP 转发和 NAT，供虚拟机使用。这个步骤是可选的，仅当虚拟机需要访问外部网络时才进行配置。示例中使用的公共接口是 igb1。

   ```sh
   builder# cat /etc/pf.conf
   ext_if="igb1"
   int_if="bridge0"
   set skip on lo0
   scrub in
   nat on $ext_if inet from !($ext_if) -> ($ext_if)
   pass out
   builder# sysrc pf_enable="YES"
   builder# sysrc gateway_enable="YES"
   ```

3. 在主机上启动 ntpd 服务，DHCP 服务器将把自己作为 ntp 服务器提供给虚拟机。

   ```sh
   builder# sysrc ntpd_enable="YES"
   builder# service ntpd start
   ```

4. 配置 DHCP 和 DNS。
   安装 dnsmasq，并将其配置为内部网络的 DHCP 和 DNS 服务器。

   ```sh
   builder# pkg install dnsmasq
   builder# cat /usr/local/etc/dnsmasq.conf
   no-poll
   interface=bridge0
   domain=vmlan,192.168.200.0/24
   localhost-record=vmhost,vmhost.vmlan,192.168.200.1
   synth-domain=vmlan,192.168.200.100,192.168.200.199,anon-vm*
   dhcp-range=192.168.200.100,192.168.200.199,255.255.255.0
   dhcp-option=option:domain-search,vmlan
   dhcp-option=option:ntp-server,192.168.200.1
   dhcp-hostsfile=/ws/vm-dhcp.conf
   ```

   将其添加为本地 `resolv.conf` 文件中的第一个名称服务器。`dnsmasq` 解析器仅为内部网络和构建机的回环接口提供服务。

   ```sh
   builder# sysrc dnsmasq_enable="YES"
   builder# service dnsmasq start
   builder# head /etc/resolv.conf
   search corp-net.example.com
   nameserver 127.0.0.1
   ...
   ```

   1. 将整个工作区导出到内部网络。

      ```sh
      builder# cat /etc/exports
      /ws/ws -ro -mapall=root -network 192.168.200.0/24
      builder# sysrc nfs_server_enable="YES"
      builder# sysrc nfsv4_server_only="YES"
      builder# service nfsd start
      ```

### vm-bhyve (bhyve 前端)

vm-bhyve 是一款易于使用的 bhyve 前端工具。

选择一个 ZFS 存储池来存储虚拟机数据，并在存储池上创建一个用于 vm-bhyve 的数据集。在 `rc.conf` 中指定该存储池和数据集的名称，在正确设置 vm_dir 后初始化 vm-bhyve。

```sh
builder# kldload -n vmm
builder# kldload -n nmdm
builder# sysrc kld_list+=”vmm nmdm”
builder# pkg install vm-bhyve
builder# zfs create rpool/vm
builder# sysrc vm_dir=”zfs:rpool/vm”
builder# vm init
builder# sysrc vm_enable=”YES”
builder# service vm start
```

所有虚拟机都将使用文本模式的串口控制台，可通过 tmux 访问。

```sh
builder# pkg install tmux
builder# vm set console=tmux
```

将之前创建的桥接接口添加为 vm-bhyve 的交换机。

```sh
builder# vm switch create -t manual -b bridge0 vmlan
```

为新虚拟机设置合理的默认值。根据需要编辑默认模板文件 `$vm_dir/.templates/default.conf`。至少指定 2 个串口—一个用于串口控制台，另一个用于远程调试。将所有新虚拟机连接到 vmlan 交换机。

```sh
builder# vim /rpool/vm/.templates/default.conf
loader=”uefi”
cpu=2
memory=2G
comports=”com1 com2”
network0_type=”virtio-net”
network0_switch=”vmlan”
disk0_size=”20G”
disk0_type=”virtio-blk”
disk0_name=”disk0.img”
```

### 镜像种子

将 FreeBSD 安装在新虚拟机中最简单的方法是使用一个已经预安装 FreeBSD 的磁盘镜像。虚拟机将启动默认内核/开发内核，其用户空间需要与二者兼容，因此最好使用与开发相同版本的 FreeBSD。

可以在 FreeBSD.org 获取 RELEASE 版本和 main 分支，STABLE 分支的最新快照的磁盘镜像。

```sh
# fetch https://download.freebsd.org/releases/VM-IMAGES/14.0-RELEASE/amd64/Latest/FreeBSD-14.0-RELEASE-amd64.raw.xz
# fetch https://download.freebsd.org/snapshots/VM-IMAGES/15.0-CURRENT/amd64/Latest/FreeBSD-15.0-CURRENT-amd64.raw.xz

# unxz -c FreeBSD-14.0-RELEASE-amd64.raw.xz > seed-14_0.img
# unxz -c FreeBSD-15.0-CURRENT-amd64.raw.xz > seed-main.img
# du -Ash seed-main.img; du -sh seed-main.img
6.0G seed-main.img
1.6G seed-main.img
```

也可以从源代码构建磁盘镜像。以下示例展示了如何构建一个无调试内核且进行了一些其他节省空间的参数的镜像。

```sh
# cd /usr/src
# make -j1C KERNCONF=GENERIC-NODEBUG buildworld buildkernel
# make -j1C -C release WITH_VMIMAGES=1 clean obj
# make -j1C -C release WITHOUT_KERNEL_SYMBOLS=1 WITHOUT_DEBUG_FILES=1 \
NOPORTS=1 NOSRC=1 WITH_VMIMAGES=1 VMFORMATS=raw VMSIZE=4g SWAPSIZE=2g \
KERNCONF=GENERIC-NODEBUG vm-image

# cp /usr/obj/usr/src/amd64.amd64/release/vm.ufs.raw seed-main.img
# du -Ash seed-main.img; du -sh seed-main.img
6.0G seed-main.img
626M seed-main.img
```

修改标准镜像，以便将其用作内部网络上的测试虚拟机。

从镜像中创建一个内存磁盘并挂载 UFS 分区。当虚拟机启动时，这将是预安装操作系统的根分区。

```sh
# mdconfig -af seed-main.img
md0
# gpart show -p md0
# mount /dev/md0p4 /mnt
```

从 `rc.conf` 中删除主机名配置，以强制使用 DHCP 服务器提供的主机名。

```sh
# sysrc -R /mnt -x hostname
# sysrc -R /mnt -x ifconfig_DEFAULT
# sysrc -R /mnt ifconfig_vtnet0=”SYNCDHCP”
# sysrc -R /mnt ntpd_enable=”YES”
# sysrc -R /mnt ntpd_sync_on_start=”YES”
# sysrc -R /mnt kld_list+=”filemon”
```

启用 SSH 访问虚拟机。请注意，这是个实验环境，网络在实验室内，并且不担心作为 root 用户运行及重用相同的主机密钥。将主机密钥和 root 用户的 `.ssh` 文件复制到正确的位置。在所有虚拟机上使用相同的密钥很方便。更新 SSH 配置以允许 root 登录并启用 SSH 服务。

```sh
# cp -a .../vm-ssh-hostkeys/ssh_host_*key* /mnt/etc/ssh/
# cp -a .../vm-root-dotssh /mnt/root/.ssh
# vim /mnt/etc/sshd_config
PermitRootLogin yes
# sysrc -R /mnt sshd_enable=”YES”
```

将第一个串口配置为潜在控制台，将第二个串口配置为远程内核调试端口。

```sh
# vim /mnt/boot/loader.conf
kern.msgbuf_show_timestamp=”2”
hint.uart.0.flags=”0x10”
hint.uart.1.flags=”0x80”
```

创建工作区的挂载点，并在 fstab 中添加条目，以便在启动时挂载。`/dev/fd` 和 `/proc` 一般是有用的。

```sh
# mkdir -p /mnt/ws
# vim /mnt/etc/fstab
...
fdesc   /dev/fd fdescfs rw      0       0
proc    /proc   procfs  rw      0       0
vmhost:/ /ws nfs ro,nfsv4 0 0
```

待就绪，卸载并销毁内存磁盘。

```sh
# umount /mnt
# mdconfig -du 0
```

`seed-main.img` 文件已准备好使用。

## 新建测试虚拟机

创建一个新的虚拟机并记录其自动生成的 MAC 地址。更新配置，使得 DHCP 服务为已知虚拟机提供分配的主机名和 IP 地址。这些静态分配的地址不能与动态地址池中的地址重叠。本文中的约定是使用两位数字的主机号来表示已知虚拟机，三位数字的主机号表示动态分配的 dhcp-range。

创建一个 `dhcp-host` 条目，包含分配给虚拟机的主机名、其 MAC 地址和一个固定的 IP 地址，该地址不属于动态范围。然后重新加载解析器。

```sh
builder# vm create vm0
builder# vm info vm0 | grep fixed-mac-address
builder# echo ‘vm0,58:9c:fc:03:40:dc,192.168.200.10’ >> /ws/vm-dhcp.conf
builder# service dnsmasq reload
```

将 `disk0.img` 文件替换为种子镜像的副本，再将其大小增加到所需的虚拟机磁盘大小。虚拟机的磁盘镜像可以在虚拟机停止运行时随时调整大小。第一次启动虚拟机时使用调整大小后的磁盘时，运行“service growfs onestart”。

```sh
builder# cp seed-main.img /rpool/vm/vm0/disk0.img
builder# truncate -s 30G /rpool/vm/vm0/disk0.img
```

### 第一次启动

在首次启动前，检查虚拟机的配置。

```sh
builder# vm configure vm0
```

以前台启动虚拟机并进入控制台，或在后台启动虚拟机后再连接到其控制台。控制台只是一个名为虚拟机名称的 tmux 会话。

```sh
builder# vm start -i vm0

builder# vm start vm0
builder# vm console vm0
```

第一次启动虚拟机时，检验以下内容：

- 虚拟机的主机名是由 DHCP 服务器分配的。登录提示中可以看到主机名和 tty。

  ```sh
  FreeBSD/amd64 (vm0) (ttyu0)
  login:
  ```

- 虚拟机的 uart0 是控制台，uart1 用于远程调试。

  ```sh
  vm0# dmesg | grep uart
  [1.002244] uart0: console (115200,n,8,1)
  ...
  [1.002252] uart1: debug port (115200,n,8,1)
  ```

- 工作区已挂载到预期位置。

  ```sh
  vm0# mount | grep nfs
  vmhost:/ on /ws (nfs, read-only, nfsv4acls)
  vm0# ls /ws
  ...
  ```

- 可以通过 SSH 在物理机和桌面访问（使用虚拟机物理机作为跳板）虚拟机。

  ```sh
  builder# ssh root@vm0

  desktop$ ssh -J root@builder root@vm0
  ```

## 在虚拟机中进行 PCIe 设备驱动开发

PCI 直通允许物理机将 PCIe 设备导出（通过直通）到虚拟机，从而让虚拟机直接访问 PCIe 设备。能在虚拟机内进行真实 PCIe 硬件的设备驱动开发。

设备由物理机上的驱动程序 ppt 声明，并在虚拟机中显示为连接到虚拟机的 PCIe 根复合体。系统上的 PCIe 设备由 BSF（或 BDF）三元组标识，在虚拟机中可能会有所不同。

使用 `pciconf` 和 `vm-bhyve` 能获取系统中 PCIe 设备的列表，并记录需要直通的设备的 BSF 三元组。注意，`pciconf` 选择器以冒号分隔 BSF，而 `bhyve/vmm/ppt` 使用斜杠（/）分隔来标识设备。例如，选择器为“none193@pci0:136:0:4”的 PCIe 设备，在 bhyve/ppt 标注中是“136/0/4”。

```sh
builder# pciconf -ll
builder# vm passthru
```

让驱动程序 ppt 声明将要直通的设备。这可以防止正常驱动程序附加到该设备。

```sh
builder# vim /boot/loader.conf
pptdevs="136/0/4 137/0/4"
```

重启以使 `loader.conf` 的更改生效，或者尝试在系统运行时将设备从其驱动程序中分离并附加到 ppt。

```sh
builder# devctl detach pci0:136:0:4
builder# devctl clear driver pci0:136:0:4
builder# devctl set driver pci0:136:0:4 ppt
# (对 137 设备重复以上步骤)
```

验证 ppt 驱动程序是否附加到设备，并检查 vm-bhyve 是否已准备好使用它们。

```sh
builder# pciconf -ll | grep ppt
ppt0@pci0:136:0:4:      020000   00   00   1425   640d   1425   0000
ppt1@pci0:137:0:4:      020000   00   00   1425   640d   1425   0000
builder# vm passthru | awk ‘NR == 1 || $3 != “No” {print}’
DEVICE     BHYVE ID     READY        DESCRIPTION
ppt0       136/0/4      Yes         T62100-CR Unified Wire Ethernet Controller
ppt1       137/0/4      Yes          T62100-CR Unified Wire Ethernet Controller
```

重新配置测试虚拟机，并列出应直通到该虚拟机的设备。

```sh
builder# vm configure vm0
passthru0="136/0/4"
passthru1="137/0/4"
```

启动测试虚拟机并验证 PCIe 设备是否可见。请注意，虚拟机中的 BSF 与物理机中的实际硬件 BSF 不同。

```sh
vm0# pciconf -ll
...
none0@pci0:0:6:0:       020000   00   00   1425   640d   1425   0000
none1@pci0:0:7:0:       020000   00   00   1425   640d   1425   0000
...
```

## 主工作流循环（编辑、构建、安装、测试、重复）

### 编辑

在桌面上编辑源代码，再将其发送到构建机。

```sh
desktop$ cd ~/work/ws/dev
desktop$ gvim sys/foo/bar.c
...
desktop$ rsync -azO --del --no-o --no-g ~/work/ws/dev root@builder:/ws/src/
```

### 构建

```sh
builder# alias wsmake=’__MAKE_CONF=${WSDIR}/src/make.conf SRC_ENV_CONF=${WSDIR}/src/src-env.conf SRCCONF=${WSDIR}/src/src.conf make -j1C’
builder# alias wsmake=’__MAKE_CONF=/ws/src/make.conf SRC_ENV_CONF=/ws/src/src-env.conf SRCCONF=/ws/src/src.conf make -j1C’

builder# cd ${WSDIR}/src/${WS}
builder# cd /ws/src/dev
builder# wsmake kernel-toolchain（仅需一次)
builder# wsmake buildkernel
```

### 安装

1. 将内核安装到虚拟机中。`INSTKERNNAME` 在 `make.conf` 中设置，因此 `/boot/${INSTKERNNAME}` 中的测试内核不会与 `/boot/kernel` 中的原始内核冲突，后者是发生问题时的安全回退。亦可在命令行中显式指定。

   ```sh
   vm0# alias wsmake='__MAKE_CONF=${WSDIR}/src/make.conf SRC_ENV_CONF=${WSDIR}/src/src-env.conf SRCCONF=${WSDIR}/src/src.conf make -j1C'
   vm0# alias wsmake='__MAKE_CONF=/ws/src/make.conf SRC_ENV_CONF=/ws/src/src-env.conf SRCCONF=/ws/src/src.conf make -j1C'

   vm0# cd ${WSDIR}/src/${WS}
   vm0# cd /ws/src/dev
   vm0# wsmake installkernel
   ```

2. 如果在构建机上使用 gdb 进行源代码级调试，还需要将内核安装到构建机的 sysroot 中。使用与虚拟机中相同的 `INSTKERNNAME` 和 `KERNCONF`。

   ```sh
   builder# cd /ws/src/dev
   builder# wsmake installkernel DESTDIR=/ws/sysroot
   ```

### 测试

选择下次重启时使用的测试内核，或永久性地设置。

```sh
vm0# nextboot -k ${WS}
vm0# nextboot -k dev
vm0# shutdown -r now

vm0# sysrc -f /boot/loader.conf kernel=”${WS}”
vm0# sysrc -f /boot/loader.conf kernel=”dev”
vm0# shutdown -r now
```

在初始测试时，使用调试版的 `KERNCONF`（例如，之前提到的自定义 `DEBUG0` 或主线中的 `GENERIC`）是个好习惯，然后可以切换到发布版内核（例如，主线中的 `GENERIC-NODEBUG`）。

## 调试测试内核

验证当前是否正在运行测试内核。

```sh
vm0# uname -i
DEBUG0
vm0# sysctl kern.bootfile
kern.bootfile: /boot/dev/kernel
```

后端

提供了两个调试后端，并且可以随时切换当前后端。

```sh
vm0# sysctl debug.kdb.available
vm0# sysctl debug.kdb.current

vm0# sysctl debug.kdb.current=ddb
vm0# sysctl debug.kdb.current=gdb
```

### 进入调试器

1. 自动：当发生 panic 时。如果设置了此 `sysctl`，内核会在发生 panic 时进入调试器（而非重启）。

   ```sh
   vm0# sysctl debug.debugger_on_panic
   ```

2. 手动：从虚拟机内部进入。

   ```sh
   vm0# sysctl debug.kdb.enter=1
   ```

3. 手动：从物理机进入。如果虚拟机卡住且无响应，可以向虚拟机注入一个 NMI。

   ```sh
   builder# bhyvectl --vm=vm0 --inject-nmi
   ```

### 源代码级调试与 gdb

源代码级调试需要源代码、二进制文件和调试文件，这些文件在物理机和虚拟机中都有，但位置各异。

### 实时远程调试

确保调试后端设置为 gdb。如果虚拟机已经通过 ddb 后端进入调试器，可以通过交互方式切换到 gdb 后端。

```sh
vm0# sysctl debug.kdb.current=gdb

db> gdb
```

当内核进入调试器时，内核中的远程 gdb 存根会激活。从物理机连接到 gdb 存根。连接通过虚拟串口线进行，该线连接到虚拟机的第二个串口（虚拟机内部的 uart1）。

```sh
builder# gdb -iex ‘set sysroot ${WSDIR}/sysroot’ -ex ‘target remote /dev/nmdm-${VM}.2B’ ${WSDIR}/sysroot/boot/${INSTKERNNAME}/kernel
builder# gdb -iex ‘set sysroot /ws/sysroot’ -ex ‘target remote /dev/nmdm-vm0.2B’ /ws/sysroot/boot/dev/kernel
```

### 核心转储分析

与实时调试相同，只是目标是 vmcore，而非远程调试。

```sh
builder# gdb -iex ‘set sysroot ${WSDIR}/sysroot’ -ex ‘target vmcore ${VMCORE}’ ${WSDIR}/sysroot/boot/${INSTKERNNAME}/kernel

builder# scp root@vm0:/var/crash/vmcore.0 /ws/tmp/
builder# gdb -iex ‘set sysroot /ws/sysroot’ -ex ‘target vmcore /ws/tmp/vmcore.0’ /ws/sysroot/boot/dev/kernel
```

---

**Navdeep Parhar** 使用 FreeBSD 已逾 20 年，自 2009 年起成为 FreeBSD 开发者。他目前在 Chelsio Communications 工作，负责为 Chelsio Terminator 系列网卡开发 FreeBSD 软件。他是 cxgbe(4) 驱动程序的作者和维护者，感兴趣的领域包括网络栈、设备驱动程序、通用内核调试和分析。
