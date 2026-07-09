# FreeBSD/RISC-V 入门

介绍 FreeBSD 最新的 CPU 架构：RISC-V。本文将带你快速了解该架构所需的知识，并演示如何使用 QEMU 构建并运行你自己的 FreeBSD/RISC-V 系统。

## RISC-V 架构简介

RISC-V 是加州大学伯克利分校在过去十年中开发的开放、可扩展的 ISA（指令集架构）。它是一种 RISC 架构，旨在吸取现有计算机架构的陷阱与教训。

RISC-V 有两大亮点。其一，ISA 规范以 BSD 许可开源发布，这意味着 RISC-V 在商业与学术场景中均可免费使用。加上其实现相当简单，使其在学术界既用于研究实现，也用于计算机架构教学，极具吸引力。对于希望避免 ARM 等其他架构高昂授权费的芯片公司，RISC-V 提供了诱人的零成本替代方案。最后，还有 lowRISC（<https://www.lowrisc.org/our-work/>）与 OpenHW Group 的 CORE-V（<https://www.openhwgroup.org/news/2019/12/10/openhw-group-announces-core-v-chassis-soc-project-and-issues-industry-call-for-participation/>）等项目，旨在打造完全开源、能运行 Unix 的 RISC-V SoC。这将使运行完全自由、开源的系统成为可能——从运行其上的操作系统与应用程序，一直到 CPU 核心本身。

RISC-V 指令集在设计上是模块化的。其理念是：RISC-V 实现只需要少量整数指令（基础 ISA），并根据所需用例，以官方或非官方扩展形式添加更多指令。一个令人意外的例子是：基础整数指令集并不要求乘法与除法指令，它们作为官方 "M" 扩展提供。其思路是：小型、单核、嵌入式核心可能不需要此功能，因此通过使其可选，实现者可自由选择省略。更复杂的 SMP 系统可能需要原子操作或硬件浮点等额外功能，这些也作为官方扩展提供。"G"（通用）扩展包含运行大多数类 Unix 操作系统所需的若干扩展。RISC-V 设计者希望，通过让 RISC-V 免费可用且易于扩展，它能被广泛用于从最小的微处理器到大型多 CPU 服务器平台的各种用例。

## FreeBSD 与 RISC-V

RISC-V 是 FreeBSD 最新、最具实验性的支持架构。该移植工作始于 2015 年，由 Ruslan Bukin（br@）主导。2016 年正式导入 FreeBSD 源码树。目前 FreeBSD 的 RISC-V 支持被归类为 Tier-3，意味着它仍处于开发阶段。因此，对功能可用性、ABI 稳定性或安全、Ports 与发布工程团队的支持不作保证。这并不是说 RISC-V 移植不可用；相反，过去几年取得了缓慢但稳步的改进，基本系统的大部分功能完全正常。

特别是 RISC-V 硬件生态仍处于早期阶段，能运行 Unix 的 RISC-V SoC 尚不易获得。FreeBSD 支持 SiFive 的 HiFive Unleashed，这是当今市场上少有的此类开发板之一，但其高昂的价格与稀缺的供货使其对大多数消费者而言不切实际。目前，FreeBSD/RISC-V 的大部分开发与测试都使用 QEMU 或 Spike 等模拟器完成。随着 RISC-V 成熟与采用率提升，FreeBSD 对它的支持也将有改善的机会。

## 构建 FreeBSD/RISC-V 镜像

好了，先介绍这么多，下面进入有趣的部分。我们将从源码构建 64 位 RISC-V 系统。这可以在运行任何受支持 FreeBSD 版本的 amd64 主机上完成。

首先，我们必须获取 RISC-V 工具链。它包括从源码构建 FreeBSD 操作系统所需的交叉编译器、链接器与其他工具。我们将使用 GNU 工具链，因为它目前对 RISC-V 的支持最成熟。你可以使用 `pkg(8)` 安装 RISC-V GNU 工具链：

```sh
pkg install riscv64-xtoolchain-gcc
```

这将安装 devel/riscv64-binutils 与 devel/riscv64-gcc 包。这个预配置工具链应该就是交叉构建 FreeBSD 所需的全部。FreeBSD/RISC-V 的所有源码都在 HEAD 中，因此用你喜欢的版本控制工具获取一份 FreeBSD 源码并开始构建：

```sh
git checkout https://github.com/freebsd/freebsd.git freebsd-riscv
```

```sh
cd freebsd-riscv
```

```sh
# 首先，构建用户态库与工具
make -j4 CROSS_TOOLCHAIN=riscv64-gcc TARGET=riscv buildworld
# 接下来构建内核。我们构建 QEMU 内核配置。
make -j4 CROSS_TOOLCHAIN=riscv64-gcc TARGET=riscv KERNCONF=QEMU buildkernel
```

可选地，在编译前编辑 `sys/riscv/conf/QEMU`，将 `ROOTDEVNAME=/dev/vtbd0p1` 行指向此设置的正确根文件系统。

> **提示**：对于稍有冒险精神的读者，如果你运行 CURRENT，FreeBSD 树内版本的 clang 与 lld 应具备构建 FreeBSD/RISC-V 所需的支持。可尝试 `make -j4 TARGET=riscv buildworld`，但效果可能因情况而异！

等待一段时间（短或长）直到编译完成。如果一切顺利，即可进入下一步。我们希望将新构建的 FreeBSD/RISC-V 根文件系统安装到某个用户可访问的目录，以便后续生成镜像文件。你可以通过 `DESTDIR` make 变量指定目录。

```sh
# NO_ROOT 允许我们以普通用户身份安装文件。
make TARGET=riscv -DNO_ROOT DESTDIR=$HOME/riscv-root installworld
make TARGET=riscv -DNO_ROOT DESTDIR=$HOME/riscv-root distribution
make TARGET=riscv -DNO_ROOT DESTDIR=$HOME/riscv-root installkernel
```

我们已安装所有必要文件，接下来要创建一个可被 QEMU 读取的磁盘镜像。我们将首先用 `makefs(8)` 生成 ufs 根文件系统，然后用 `mkimg(1)` 创建包含交换分区的镜像。

切换到我们刚刚填充的根文件系统目录。你可以使用以下脚本生成镜像。

```sh
#!/bin/sh
# 创建 /etc/rc.conf 并追加到 METALOG
echo 'hostname="qemu"' > etc/rc.conf
s=$(($(cat etc/rc.conf | wc -c)))
echo "./etc/rc.conf type=file uname=root gname=wheel mode=0644 size=$s" >> METALOG
# 创建 /etc/fstab 并追加到 METALOG
echo "/dev/vtbd0p1 / ufs rw,noatime 1 1" > etc/fstab
echo "/dev/vtbd0p2 / swap sw 0 0" >> etc/fstab
s=$(($(cat etc/fstab | wc -c)))
echo "./etc/fstab type=file uname=root gname=wheel mode=0644 size=$s" >> METALOG
# 创建 FreeBSD ufs 根分区
makefs -D -B little \
     -o label=freebsd_root \
     -o version=2 \
     -s 20g -f 65% \
     riscvroot.ufs METALOG
# 创建最终的 .img
mkimg -s gpt -p freebsd-ufs:=riscvroot.ufs -p freebsd-swap::4G -o riscvroot.img
```

## 启动 FreeBSD

我们已经构建了自己的 FreeBSD 镜像，现在来测试它。如前所述，我们将使用 QEMU。请从 Ports 安装 `emulators/qemu-devel` 包。注意：常规的 `emulators/qemu` 足以运行 FreeBSD，但 RISC-V QEMU 平台已有许多改进，因此推荐使用 `emulators/qemu-devel`。

你还需要通过 `sysutils/opensbi` 安装 OpenSBI。OpenSBI 将充当引导加载程序，并通过其 Supervisor Binary Interface（SBI）为 FreeBSD 内核提供所需的低级固件功能。

安装 OpenSBI 后，即可使用以下命令启动 FreeBSD：

```sh
qemu-system-riscv64 -machine virt -smp 2 -m 2G -nographic \
     -kernel $HOME/riscv-root/boot/kernel/kernel \
     -bios /usr/local/share/opensbi/platform/qemu/virt/firmware/fw_jump.elf \
     -drive file=/path/to/riscvroot.img,format=raw,id=hd0 \
     -device virtio-blk-device,drive=hd0
```

如果一切顺利，你应看到 FreeBSD 开始启动。如果你跳过了之前的可选步骤，系统将无法挂载根文件系统。发生时，在提示符处输入 `ufs:/dev/vtbd0p1` 即可。

你可能注意到没有看到熟悉的 `loader(8)` 提示符。这是因为 `loader(8)` 尚未移植到 RISC-V，因此内核直接从 OpenSBI 启动。目前你还无法享受 loader 微调或可调参数，但未来一定会支持。

你应看到系统启动到登录提示符。以 root 登录，并用 `passwd(1)` 修改 root 密码。

## 网络

大多数系统没有网络连接就没什么用处，因此我们关机来解决这个问题。启动 QEMU 时在命令行中添加以下参数：

```sh
-netdev user,hostfwd=tcp::10000-:22,id=net0 -device virtio-net-device,netdev=net0
```

如你所见，我们将主机的 TCP 端口 10000 转发到客户机的 22 端口。22 端口是 `ssh(1)` 使用的默认端口。在连接之前，我们必须通过将以下内容追加到 `/etc/rc.conf` 在客户机上启用 `sshd(8)`：

```sh
sshd_enable="YES"
```

接下来，用以下命令启动 `sshd(8)` 服务：

```sh
service sshd start
```

你现在应能登录到 QEMU 客户机，只需指定正确的端口：

```sh
ssh -p 10000 mhorne@localhost
```

如果你有桥接到真实网卡的 `tap(4)` 设备，也可以使用它。改为向 QEMU 追加以下参数：

```sh
-netdev tap,ifname=tap0,script=no,id=net1 -device virtio-net-device,netdev=net1
```

`tap(4)` 接口将让你的客户机在网络中表现为任何其他主机。使用客户机上 `ifconfig(8)` 输出中显示的 IP 地址连接：

```sh
ssh mhorne@10.0.1.25
```

## 更新系统

假设过了一段时间，你想更新 FreeBSD/RISC-V 系统以利用刚合入 HEAD 的某个修复或特性。对典型 FreeBSD 机器，你可能会做自托管构建，即用正在更新的同一台机器构建并安装。遗憾的是，由于使用 QEMU 运行 FreeBSD/RISC-V，我们受限于模拟器，这意味着任何自托管构建都会很慢。

你当然可以在更新源码树后按前述步骤操作，最终会得到一个全新的 FreeBSD/RISC-V 系统；但你所做的任何配置都会丢失。下面看看如何在现有的根镜像文件内更新系统。

首先，我们将执行与之前相同的步骤，交叉编译面向 riscv64 的更新版 FreeBSD。

```sh
# 更新源码树，假设使用 git
```

```sh
cd freebsd-riscv
```

```sh
git pull
# 现在重新构建并安装
make -j4 CROSS_TOOLCHAIN=riscv64-gcc TARGET=riscv buildworld
make -j4 CROSS_TOOLCHAIN=riscv64-gcc TARGET=riscv KERNCONF=QEMU buildkernel
```

更新后的系统已就绪待安装。但这次我们不安装到主机上的某个目标目录，而是要更新我们之前生成的镜像文件的内容。我们可以利用 `mdconfig(8)`，它允许我们将镜像文件作为内存盘创建并挂载。

> **提示**：本步骤及后续步骤需要超级用户权限。为防止对文件系统的并发访问，请在继续之前确保已关闭所有使用该镜像文件的 QEMU 实例。

```sh
# 创建内存设备
```

```sh
mdconfig -a -f /path/to/riscvroot.img
```

新内存设备的名称将输出到控制台。请记住，我们生成的镜像包含两个分区：ufs 根分区与交换分区。我们要挂载 ufs 分区，以便在其中安装更新后的系统。

```sh
# 对于内存设备 /dev/md0
mount /dev/md0p1 /mnt
# 将系统安装到挂载的分区
make TARGET=riscv DESTDIR=/mnt installworld
make TARGET=riscv DESTDIR=/mnt installkernel
# 可选地删除过期文件
make TARGET=riscv DESTDIR=/mnt delete-old
make TARGET=riscv DESTDIR=/mnt delete-old-libs
# 现在卸载
umount /dev/md0p1
```

大功告成！你可以使用之前创建的同一镜像文件启动全新更新的系统。需要注意的是，QEMU 命令行上指定的内核才会被启动，而非安装到镜像文件系统中的内核。因此，你可能需要在 `/usr/obj` 目录中搜索它，或在主机上执行 `installkernel` 到本地文件夹。

## 结语

本文旨在介绍 RISC-V CPU 架构，并让读者能轻松实验 FreeBSD/RISC-V。希望你现在对构建、安装并运行自己的 FreeBSD/RISC-V 系统的能力充满信心。鼓励感兴趣的用户在系统搭建完毕后，继续跟踪变更并更新系统。进一步阅读请查看 FreeBSD wiki 上的 RISC-V 页面（<https://wiki.freebsd.org/riscv>），并订阅 freebsd-riscv 邮件列表（<https://lists.freebsd.org/mailman/listinfo/freebsd-riscv>）。

诚然，对 RISC-V 整体而言仍处于早期。硬件侧的供货不足与软件侧的不完整支持意味着，RISC-V 的用处目前主要限于研究与专用计算领域。过去几年对软件支持的许多改进与 RISC-V 采用率的提升表明，这种情况不会一直如此，它可能很快会作为更易得的通用计算平台出现。随着 RISC-V 持续成长，

MITCHELL HORNE 是加拿大滑铁卢大学即将完成本科学业的大学生。他是 FreeBSD src committer，并在过去一年中成为 FreeBSD/RISC-V 移植的主要贡献者之一。
