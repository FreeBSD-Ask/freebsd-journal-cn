# 使用 Poudriere 进行 Port 批量管理

- 原文链接：[Bulk Ports Management with Poudriere](https://freebsdfoundation.org/wp-content/uploads/2020/07/FreeBSD-Guides-Bulk-Ports-Management-with-Poudriere.pdf)
- 作者：**DREW GURKOWSKI**

## 第一步：使用 Ports 安装 Poudriere

本节中的所有命令应以 root 用户身份运行；使用 su(1) 命令即可。要将 Ports 的压缩快照下载到 `/var/db/portsnap`：

```sh
# portsnap fetch
```

第一次运行 portsnap 时，解压缩快照到 /usr/ports：

```sh
# portsnap extract
```

接下来，从 Ports 中构建并安装 Poudriere：

```sh
# cd /usr/ports/ports-mgmt/poudriere
# make install clean
```

在安装过程中会出现提示，保持默认设置并安装软件包。

## 第二步：配置 Poudriere

Poudriere 是一款非常强大的工具，旨在用于软件包的生产，但它也可以用于批量管理 ports。为了继续使用它，需要对配置做一些小的调整。

首先，将配置文件复制并移动到正确的位置：

```sh
# cd /usr/local/etc
# cp poudriere.conf.sample poudriere.conf
```

然后使用 ee(1) 文本编辑器编辑复制后的配置文件：

```sh
# ee poudriere.conf
```

使用箭头键向下导航到以下行：

```sh
FREEBSD_HOST=_PROTO_://_CHANGE_THIS_
```

编辑上述行，使其看起来像这样：（使用 **退格键** 删除文本）

```sh
FREEBSD_HOST=ftp://ftp.freebsd.org
```

按 **ESC** 键，然后按两次 **回车键** 以退出并保存对配置文件的更改。

## 第三步：设置 Poudriere Jail

在继续之前，Poudriere 需要获取并解压其版本的 FreeBSD Ports。为此，请运行以下命令：

```sh
# rehash
# poudriere ports -c
```

Poudriere 执行批量功能时需要设置 FreeBSD jail。可以使用以下命令来完成：

```sh
# mkdir /usr/local/poudriere
# poudriere jail -c -j 91x64 -v 12.1-RELEASE -a amd64
```

在上述命令中，91x64 用于标识 jail，而 12.1-RELEASE 标识要使用的 FreeBSD 版本。如果需要不同的名称或 FreeBSD 版本，可以进行调整。只需记得在本指南的其余部分也进行替换。

## 第四步：创建批量 Port 列表

![](https://github.com/user-attachments/assets/9f685d00-607f-407b-b3ec-5471d735ba0b)

继续之前，运行以下命令：

```sh
# cd poudriere.d
# echo WITH_PKGNG=YES >> 91x64-make.conf
```

接下来的步骤是为 Poudriere 创建一个 Port 列表以进行编译和维护；FreeBSD Ports 中有各种各样的 Port，Poudriere 可以用来管理它们。首先，运行：

```sh
# cd /usr/local/etc
# ee poudriere-list
```

像之前一样，使用 ee(1) 文本编辑器编辑文件，添加 Poudriere 要管理的 Port 列表。上面是一个示例图像，其中包括 Firefox、i3 窗口管理器、irssi 和 tmux。可以通过使用 Port 原始路径（类别/名称）将 Port 添加到列表中。

按 **ESC 键**，然后按两次 **回车键** 以退出并保存对配置文件的更改。

## 第五步：配置 Poudriere 安装选项




![](https://github.com/user-attachments/assets/a9c24659-321e-4a9d-836e-06e9c0c46c49)


此步骤是可选的，除非需要手动配置。

尽管 Poudriere 可以用于自动化批量 Port 管理，但它仍然允许用户手动配置每个 Port。然而，这一步可以在安装之前进行，而不必在整个过程中都在场。例如，如果用户想要编辑 tmux(1)，可以使用以下命令：

```sh
# poudriere options -c sysutils/tmux
```

然后，使用安装提示（箭头键导航，空格键选择），可以手动编辑 Port。此过程可以对 Poudriere 管理的每个 Port 执行。

## 第六步：使用 Poudriere 管理 Port 

![](https://github.com/user-attachments/assets/2b628cac-88f3-4019-9d9b-53a411bf6767)


配置完成后，安装整个 Ports 列表只需一条命令：

```sh
# poudriere bulk -j 91x64 -f poudriere-list
```

Poudriere 会花费一些时间来完成批量处理，但与手动构建和安装 Port 不同，它可以在无需用户输入的情况下完成所有工作。Poudriere 还提供了基于文本的可视化显示，展示安装过程。它将允许用户在不牺牲自动化的情况下，更好地控制他们的 Port。

如果需要更新 Port 列表，可以使用以下命令来更新 Ports：

```sh
# cd /usr/local/etc
# poudriere bulk -j 91x64 -f poudriere-list
```

---

**DREW GURKOWSKI**, FreeBSD 基金会
