# 在 FreeBSD 上轻松搭建我的世界（Minecraft）服务器

- 原文链接：[Easy Minecraft Server on FreeBSD](https://freebsdfoundation.org/wp-content/uploads/2020/09/Minecraft-Server.pdf)
- 作者：**DREW GURKOWSKI**

## 第一步：选择设备

![](https://github.com/user-attachments/assets/4de7a864-211c-4834-9b35-e2d19f79274c)


在设置服务器之前，选择设置专用设备还是运行 FreeBSD 的虚拟机。

* 专用服务器将更加可靠，但需要一个完全专用于运行服务器的设备。如果需要一个全天候可靠的服务器，这个方式非常合适。
* 在虚拟机上运行服务器能让用户在没有专用设备的情况下运行服务器，并且可以更好地控制服务器的运行时间。

如果设备或虚拟机已经在运行 FreeBSD，请继续下一步。否则，可以参考基金会的指南来设置 FreeBSD：

* 使用 VirtualBox 安装 FreeBSD
* 将 FreeBSD 作为主操作系统
* 为树莓派安装 FreeBSD

## 第二步：使用 Ports 

本节中的所有命令应以 root 身份运行；使用命令“su”也可以。要将压缩快照下载到 `/var/db/portsnap`：

```sh
# portsnap fetch
```

首次运行 Portsnap 时，将快照提取到 /usr/ports：

```sh
# portsnap extract
```

完成首次使用 Portsnap 后，正如上面所示，可以通过运行以下命令随时更新 /usr/ports：

```sh
# portsnap fetch update
```

## 第三步：Port minecraft-server 

![](https://github.com/user-attachments/assets/487d6fbc-4cb4-4d88-9f06-4560ba5582f4)


FreeBSD ports 中有各种各样的游戏 Port，其中包括一个很好的选项，用于运行基本的 Minecraft 服务器：Port minecraft-server。

第一步是安装该 Port；现在已经添加 port，可以通过以下命令简单安装：

```sh
# cd /usr/ports/games/minecraft-server/ && make install clean
```

为了写入必要的文件，用户需要通过以下命令运行程序：

```sh
# minecraft-server
```

接下来，用户需要同意 Minecraft 终端用户许可协议（EULA）。可以通过使用 vi 命令来完成。以下命令将启动 vi 文本编辑器：

```sh
# vi /usr/local/etc/minecraft-server/eula.txt
```

使用编辑器（如果不熟悉编辑器，可以查看手册页），将第 4 行修改为：

```sh
eula=true
```

然后可以通过以下命令运行服务器：

```sh
# minecraft-server
```

要停止服务器：

```sh
# stop
```

## 第四步：确定外部 IP 地址

要通过互联网连接到 Minecraft 服务器，用户需要知道设备的外部 IP 地址，并配置路由器的端口转发。这个过程对于每种路由器都不同，因此本指南提供了一些链接以帮助简化操作。

首先，确定设备的外部 IP 地址。一种非常简单的方法是使用 wget(1) 命令：

```sh
# pkg install wget
```

按照提示完成安装，输入 'y' 确认。接下来，使用 wget(1) 从网站拉取设备的外部 IP 地址：

```sh
# wget -qO - http://wtfismyip.com/text
```

这将返回一个类似于“65.214.224.57”的 IP 地址；然而，每组数字将会不同。这就是外部 IP 地址，其他设备需要这个字符串才能连接到 Minecraft 服务器。

## 第五步：端口转发

![](https://github.com/user-attachments/assets/7b192ce3-36f2-415f-827f-dd90b0e07557)


下一步是确保与设备连接的路由器正确地将所有 Minecraft 流量转发到设备上。即使网络上没有其他计算机，这也是一个必需的步骤。Minecraft 使用端口 25565，因此需要设置路由器将所有通过 25565 端口的流量发送到运行 Minecraft 服务器的计算机。路由器设置必须在另外的已设置桌面环境的计算机上完成。

如果在虚拟机上进行设置，请执行此步骤：许多虚拟机的 IP 地址与主机计算机相同，但不会接收到任何传入流量。为了解决这个问题，需要使用桥接网络适配器。可以通过打开虚拟机的网络设置，将网络适配器从 NAT 改为桥接。这样，虚拟机将获得自己的独立 IP 地址，可用于转发流量。重要的是，在接下来的步骤中使用虚拟机的 IP 地址，而不是主机计算机的 IP 地址，因为流量将无法正确转发到 Minecraft 服务器。

![](https://github.com/user-attachments/assets/7de15e4b-c8f0-4b42-8eb8-dc568f500c0c)


在设置过程中，路由器需要识别正确的设备来转发流量。许多路由器会列出设备及其内部 IP（与第 4 步中找到的 IP 不同），但情况可能并非如此。此 IP 应类似于 `192.168.1.176`，它是路由器分配给设备的号码。要识别设备的 IP 地址，可以使用以下命令：

```sh
$ ifconfig
```

每个路由器的端口转发设置过程都是独特的。访问 portforward.com 并选择路由器型号（通常能在路由器背面找到）。如果出现广告，请点击 `CLOSE`，不需要使用广告中的应用程序。

接下来的页面将提供该型号路由器的端口转发设置指南。尽管每个路由器的设置不同，但需要转发到设备的关键端口如下：

**TCP: 25565**

**UDP: 19132-19133, 25565**

外部和本地 IP 地址在设备关机或调制解调器重置时可能会发生变化。每次重启服务器时，请检查这些 IP 地址是否发生变化，并根据需要更新设置。

## 第六步：连接到服务器

现在服务器已经设置完成，任何 Minecraft 客户端都应该能够通过第 4 步中找到的外部 IP 地址连接。首先，确保服务器在主机设备或虚拟机上运行，使用以下命令：

```sh
# minecraft-server
```

接下来，在将要连接到服务器的计算机上启动 Minecraft 客户端，点击 "多人游戏" 然后选择 "直接连接"；客户端将要求输入服务器地址。此地址即为第 4 步中找到的外部 IP 地址。点击“加入服务器”即可开始游戏会话。

开始在服务器上游戏后，你可以尝试使用服务器命令；这些命令将在运行服务器的设备的终端中输入，并可用来操控游戏世界、游戏规则以及允许进入服务器的玩家。

---

**DREW GURKOWSKI**，FreeBSD 基金会
