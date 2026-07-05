# FreeBSD 入门指南：在虚拟机与真实硬件上安装

- 原文链接：[A Guide to Getting Started with FreeBSD on Virtual and Real Hardware](https://freebsdfoundation.org/our-work/journal/browser-based-edition/)
- 作者：**ROLLER ANGEL**

FreeBSD 入门很容易，至于在哪里安装和运行 FreeBSD，有多种选择。本文聚焦两种常见场景：在虚拟机中安装，以及在真实硬件上安装——本例为一台 amd64 桌面电脑。

首先，在一台可用电脑上访问 FreeBSD.org，找到 beastie logo 旁边那个显眼的“Download FreeBSD”按钮。在该页面，从“Installer Images”列表中选择合适的镜像。我们选择“amd64”。接下来，从列表中挑选 FreeBSD-12.0-RELEASE-amd64-bootonly.iso 这个镜像，因为它是 FreeBSD 最新的 RELEASE 版本，且体积相当小。强烈建议你也查看 CHECKSUM.SHA512-FreeBSD-12.0-RELEASE-amd64 文件，在下载完成后自己做一次 SHA512 校验和验证，看两者是否匹配。如果一致，就说明下载没有问题。

下载完安装镜像后，就可以用它启动虚拟机了。在真实硬件上有几个额外步骤：把 .iso 刻录成实物 CD，并在启动时选择启动设备。这里假定你的真实硬件配有 CD 驱动器，已将 .iso 文件刻录成 CD 并放入驱动器中。首次启动电脑时，BIOS 界面会决定从哪个设备启动。要把该设备改为安装 CD，可能需要查阅电脑手册。多数电脑可以尝试开机后立即反复按 `F8`，直到“Boot Device Selection”菜单出现。从列表中选择正确设备以启动安装程序。我们选择了 CD 设备的 UEFI 版本，本例中为一台 Pioneer BDR-205 光盘读写器。

在 VirtualBox 中，点击“New”按钮启动创建虚拟机的向导。需要留意的几个关键选项：操作系统类型——我们选择 FreeBSD 64 bit；以及分配给虚拟机的内存大小。选择大于 512MB 但不会让 VirtualBox 向导“滑块”变红的某个数值。让滑块保持在绿色区域，给机器分配足够的内存即可。其它设置采用默认值。向导完成后，可以启动虚拟机。会出现另一个向导，让你为该新虚拟机指定安装镜像。如果想跳过这第二个向导，只需在首次启动前编辑虚拟机设置，点击“Storage”链接，再点击“Empty Disc”部分，即可在该界面挂载虚拟镜像。VirtualBox 中的这些步骤等同于组装硬件并插入安装 CD。在 VirtualBox 中按“Start”，就像按下真实机器的“Power”按钮启动一样。

首次从安装 CD 启动时，FreeBSD 欢迎界面会询问是否开始安装。在键盘上按“Enter/Return”选择“Install”。此时你会注意到 VirtualBox 提示它正在捕获鼠标和键盘以便在虚拟机内使用。直接忽略这些提示即可。要在虚拟机之外再次使用鼠标和键盘，需要按下主机键。VirtualBox 在窗口右下角显示当前主机键。本次安装显示的是“Right Ctrl”，意味着你需要先按下键盘右侧的 Ctrl 键，才能从虚拟机取回鼠标和键盘控制权。

按以下要点继续安装：本次安装继续使用默认的 US 键盘布局，将电脑命名为“getting-started-with-freebsd”，在“Distribution Select”界面直接按“ok”采用默认设置。在“Network Configuration”部分，我们在虚拟机上选择 `em0` 接口，在真实硬件上选择 `re0` 接口。是，配置 IPv4；是，使用 DHCP；是，使用 IPv6；是，尝试 SLAAC。解析器配置应至少填入一个 DNS 值。在键盘上按“Tab”键切换到“ok”继续。在“Mirror Configuration”部分，我们选择了“Main Site”。接下来需要决定如何分区磁盘。在“Partitioning”界面有 Auto (ZFS) 和 Auto (UFS) 两个选项。

**ZFS 版本**

选择 Auto (ZFS)，将 ZFS 池类型选为 stripe，用空格键选择用于该池的磁盘，给池命名，然后继续安装。最后，选择“Yes”同意安装 FreeBSD。

**UFS 版本**

选择 Auto (UFS)，使用整个磁盘，分区方案选 GPT，在显示磁盘布局时选择“Finish”。最后，选择“Commit”同意擦除电脑并安装 FreeBSD。

**用户账户**

为系统的 root 账户设置密码。输入 root 密码时屏幕上没有出现 `***` 不必担心。请放心，按键已记录下来，只是不在屏幕上显示而已。

“Time Zone”部分我们依次选择“America – North and South, United States of America”、“Mountain (most areas)”，在两个“Time & Date”界面均选择“Skip”。

“System Configuration”部分，我们取消选择 `sshd`，选择 `powerd`，然后选择“ok”。在下一界面选择 `9 secure_console`，这样任何在系统控制台尝试进入单用户模式时都会出现 root 密码提示。

对添加新用户的提示选“yes”。用户名全部使用小写。输入用户的“Full name”。将用户账户加入 wheel operator video 组——在问题“Login group is .. Invite .. into other groups? []:”后回答 wheel operator video，然后按“Enter”。wheel 组成员可以运行“特权”命令。operator 组允许用户在不调用 wheel 组特权的情况下关机和重启电脑。如果打算把 FreeBSD 当桌面用，把用户加入 video 组是个好主意。我们把 shell 选为 `tcsh`，其它字段使用默认值，最后设置账户密码。如果一切正常，就不再创建更多用户账户。

“Final Configuration”部分，选择“Handbook”。我们采用默认的“en”，选择“ok”。回到配置界面后，选择“Exit”完成，并选“yes”进入手动配置。我们唯一需要做的手动配置是关闭机器以便取出 CD。在真实机器上你需要通电才能取出 CD，所以现在就把它取出来。在虚拟机上，必须断电才能避免取出 CD 时出错，所以执行下面的 shutdown 命令关闭电脑。

输入 `init 0` 或 `shutdown -p now` 关机。

在 VirtualBox 中取出虚拟机 CD：点击虚拟机名称，点击“Settings”，点击“Storage”，点击那个小小的 CD 图标。注意“Optical Drive”旁边另一个带向下小箭头的小 CD 图标。用这第二个 CD 图标执行“Remove Disk from Virtual Drive”。现在“Start”机器并继续。

**软件包安装 / 文本配置**

启动电脑并以 root 用户登录。

在虚拟机上输入以下命令：

```sh
pkg install -y xorg sudo lumina sakura virtualbox-ose-additions firefox
sysrc vboxguest_enable=YES
sysrc vboxservice_enable=YES
```

在真实机器上输入以下命令：

```sh
pkg install -y xorg sudo lumina sakura firefox
```

在两台机器上都输入以下命令：

```sh
visudo
```

（该命令会启动 vi 文本编辑器。）

输入以下内容来编辑那一行文本并退出编辑器：

```sh
/wheel
```

按回车，然后输入：

```sh
j0xxZZ
```

（命中第一个搜索结果后，`j` 下移一行，`0` 跳到行首，`xx` 删除注释，`ZZ` 保存并退出 vi。）

```sh
sysrc dbus_enable=YES
dbus-uuidgen > /etc/machine-id
service dbus start
```

按住键盘上的“control”并按“d”，注销当前用户并切换为普通用户。再次登录，这次使用刚创建的普通用户账户。然后输入以下命令：

```sh
vi .xinitrc
```

又是 vi 编辑器，要开始输入文本需要进入“插入模式”，按“I”进入插入模式并开始输入。输入以下一行配置：

```sh
exec start-lumina-desktop
```

按键盘上的“Esc”退出 vi，然后输入 `ZZ` 或 `:wq` 再按“Enter/Return”。此时虚拟机桌面可以用以下命令启动：

```sh
startx
```

要让真实机器工作，需要选择与真实硬件兼容的显示驱动。我们采用基于 scfb 显示驱动的基本非 3D 加速图形设置。要设置它，输入以下命令：

```sh
vi /usr/local/etc/X11/xorg.conf.d/driver-scfb.conf
```

用“Esc” `ZZ` 退出，向文件中添加以下配置行：

```sh
Section "Device"
Identifier "Card0"
Driver "scfb"
EndSection
```

现在真实硬件机器的桌面可以用以下命令启动：

```sh
startx
```

**桌面配置**

右键桌面 > Preferences > All Desktop Settings

在 Appearance 下点击“Theme”，然后从侧边栏选择“Icons”，选择“Material Design (light)”，点击“Apply”。关闭“Theme Settings”窗口。

回到“Desktop Settings”窗口，选择“Window Manager”。在“Window Theme”下拉菜单中选择“Twice”。点击“Save”。关闭“Window Manager Settings”窗口。

回到“Desktop Settings”窗口，选择“Applications”。将“E-mail Client”设为 Firefox，“Virtual Terminal”设为 Sakura。关闭所有设置窗口。

右键桌面，选择“Preferences” > “Wallpaper”，点击“+”，选择“solid color”。我们选了红色。点击“ok”，再点击“Save”。

**FreeBSD Handbook**

下面是如何打开我们在 Final Configuration 中安装的手册本地副本：

在 Lumina 的应用菜单中找到 Firefox。

右键桌面 > Applications > Network > Firefox Web Browser

打开 Firefox，访问以下 URL：**file:///usr/local/share/doc/freebsd/handbook/index.html**

这是 FreeBSD Handbook 的本地副本，由于不需要联网，它始终可用。

至此你应该可以开始使用 FreeBSD 了。下一步建议选择并配置一个防火墙。FreeBSD 期刊 2014 年 5/6 月号有一篇精彩文章《IPfw An Overview》，可指导你完成该过程。此外，读一读《BSD Hacks》这本书，可以让你更熟悉 `tcsh` shell，并掌握若干新技能。 •

**ROLLER ANGEL** 是一名热忱的 BSD 用户，喜欢探索 BSD 技术所能实现的种种精彩事情。他曾基于 FreeBSD 讲授过编程研讨班，正在搭建一个在线培训平台，用于教授 BSD 及相关技术。详见 BSD.pw。
