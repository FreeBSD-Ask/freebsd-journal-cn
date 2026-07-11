# 在 FreeBSD 上配置全盘加密

作者：Roller Angel

在 FreeBSD 上配置全盘加密有多种方法，本文聚焦其中一种，提供一条简单路径让你使用 GELI 快速上手。如果你的机器上已安装 FreeBSD，想为附加到现有安装的另一块磁盘启用 GELI 全盘加密，可参阅 Michael W Lucas 所著的 FreeBSD Storage Essentials 一书了解详情。

## 安装步骤

在本文使用的机器上，我们全新安装 FreeBSD，并使用标准安装程序在 ZFS 上启用 GELI 全盘加密。按照常规方式运行安装程序，进入分区界面时，选择 Auto (ZFS)。接下来选择 Pool Type/Disks 选项，选择你想完全加密并安装 FreeBSD 的磁盘。选择 stripe 作为虚拟设备类型。用空格键选中磁盘，然后按回车键。这会回到 ZFS 配置菜单，向下移动到 Encrypt Disks? 并按回车将 NO 改为 YES。向上移动到 Proceed with Installation，让工具为你的全新 FreeBSD 安装启用全盘加密。

下一个屏幕会要求你输入密码短语，该密码短语将在每次启动机器时用于解密磁盘。我们使用存储在 YubiKey 上的 38 字符静态密码短语，加上一个记忆在脑中并手动输入的密码短语，输入顺序在 YubiKey 存储的密码短语之前。首先输入记忆的密码短语，然后按下 YubiKey 的按钮让它输出 38 字符密码短语，再按回车。记忆的这部分密码短语可以防止小偷在使用你的 YubiKey 解密你的机器时得逞——假设他同时拿到了你的机器和 YubiKey。他还必须知道你记忆的前半部分，因此请务必保密。

## 配置 YubiKey

你可以参考 FreeBSD 期刊 2019 年 1/2 月刊中的 FreeBSD 虚拟与真实硬件入门指南，搭建桌面环境以便与 YubiKey 个性化工具交互，并将 YubiKey 编程为最多存储 38 个字符的密码短语。YubiKey 编程完成后，可以重新安装 FreeBSD 以获得全盘加密的好处。好处在于：如果有人偷走你的电脑，或在你关机时暂时获取了访问权，他无法访问其中的文件，因为磁盘在静止时是加密的，存储在上面的数据在用你的密码短语通过 GELI 解密磁盘之前无法访问。

要编程 YubiKey，首先需要安装 Yubico 提供的软件，输入以下命令：

```sh
# pkg install -y yubikey-personalization-gui
```

然后可以在 Lumina 菜单中选择新安装的软件打开它，或运行以下命令打开：

```sh
yubikey-personalization-gui
```

访问 Static Password 选项卡。接下来点击 Scan Code 按钮。第一步是选择要使用的 Configuration Slot。请参阅 Configuration Slot 选择气泡旁边的 ? 图标提供的配置槽说明和使用方法。接下来，在 Password 部分，找到带有 Choose-a-Layout 下拉菜单的 Keyboard 标签，选择你将使用的键盘布局。现在可以将你的随机密码短语插入 Password 字段旁边的框中。最后，插入 YubiKey 后，选择 Write Configuration 将该密码短语保存到 YubiKey 的指定配置槽。现在，当你按住 YubiKey 上的按钮时，会看到密码短语自动输入，就像你在键盘上手动输入一样。注意，按住按钮的时间长短取决于你编程的是哪个 Configuration Slot。

## 重启

重启刚刚安装 FreeBSD 的机器，你首先会看到一个要求输入密码短语的屏幕。正确输入密码短语后，系统将正常启动。

Roller Angel 是热心的 BSD 用户，乐于探索 BSD 技术的各种奇妙用途。他曾基于 FreeBSD 举办编程工作坊，目前正致力于搭建一个在线培训平台，教授 BSD 及相关技术。详见 BSD.pw。
