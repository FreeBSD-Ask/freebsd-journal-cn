# 在 FreeBSD/GhostBSD 上开始 Java 开发

- 原文链接：[Getting Started with Java Development in FreeBSD/GhostBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/)
- 作者：**ASHIK SALAHUDEEN**

本文简要介绍如何在运行 FreeBSD 12.0 或基于其的桌面操作系统（如 GhostBSD 或 Trident 桌面）的电脑上搭建 Java 开发环境。

## Java 究竟是什么

Java 是一种通用的面向对象编程语言，由 Sun Microsystems 开发，于 1996 年首次发布。它的主要口号是“一次编写，到处运行”，让开发者能够编写 Java 代码、编译为字节码，并在任何具备 Java 运行时的操作系统上执行。因此，某人可以在 Windows 上编写代码，却在 FreeBSD 或 GNU/Linux 服务器上执行。多年下来，Java 已变得极其流行，是编写服务端应用以及 Android 应用的可靠选择。

## FreeBSD 平台上的 Java

Oracle 公司收购了 Sun，现持有官方 Java 运行时与开发套件的实现。他们为 Windows、Mac OSX 与 Linux 操作系统提供实现。官方参考实现以 GPLv2（含链接异常）开源，因此 FreeBSD 上可以拥有 OpenJDK 实现。

Java 当前的长期支持版本是 2018 年 9 月发布的 Java 11。目前还没有针对 FreeBSD 的构建。上一个稳定版本是 2014 年 3 月发布的 Java 8，该版本有对应的构建。本文余下部分基于 Java 8 openjdk。

## 入门：安装 Java 开发工具包

要用 Java 编写应用，需要 Java 开发工具包，它提供编译器、运行时与标准库。

幸运的是，它以预构建包形式提供，对开发用途而言二进制包已足够。使用以下命令安装（以 root 身份）：

```sh
pkg install openjdk8
```

这不会花费太长时间，可以在终端调用 Java 编译器验证安装是否成功：

```sh
$ javac -version
javac 1.8.0_181
```

## 编辑 Java 代码

你可以用纯文本编辑器编辑 Java 代码，但由于该语言的工作方式，很快就会变得繁琐。几乎所有 Java 程序员都使用 IDE 编辑代码。有多种可选编辑器，部分为商业产品。下面是若干推荐的 IDE：

**Intellij IDEA 社区版**

IntelliJ 是一款优秀的 IDE，以二进制包提供，采用 Apache 2.0 许可证发布。使用以下命令安装：

```sh
pkg install intellij
```

该 IDE 的开发者 Jetbrains 还提供名为 IntelliJ Ultimate 的商业（付费）版本，功能更多。对所有实用目的而言，社区版已足够。

**Netbeans**

Netbeans 是另一款历史悠久的优秀 IDE，同样采用 Apache 2.0 许可证发布，并以预构建包提供。使用以下命令安装：

```sh
pkg install netbeans
```

**Eclipse**

Eclipse 是另一款流行 IDE，采用 Eclipse Public License 提供，同样以预构建包形式提供。使用以下命令安装：

```sh
pkg install eclipse
```

## 一些让事情更顺手的通用配置

总体而言，设置一些环境变量/配置会让 Java 开发更顺遂。下面是一份简要清单：

**Java Home**

该变量指定 JDK/JRE 的位置。建议通过 `.profile` 文件设置环境变量。添加以下行：

```sh
JAVA_HOME=/usr/local/openjdk8
```

**Maven Home**

Maven 是 Java 项目流行的项目管理工具。它是纯 Java 应用，可通过二进制包获得。不过该项目更新频繁，若想获取最新 Maven 二进制，最好仍直接从项目站点下载。若选择该方式，使用环境变量 `M2_HOME` 设置 Maven 安装目录的位置。该路径是 Maven 安装中“bin”目录所在之处。

```sh
M2_HOME=/path/to/apache-maven-3.x.0/
```

**Intellij —— 修复字体渲染**

若 IDE 中的字体看起来模糊，可以通过向启动脚本传递以下参数来修复该问题。IntelliJ 通过“Help > Edit Custom VM Options”菜单让这一切变得简单。把以下内容粘贴到打开的文本文件中。关闭 IDE 并重新启动。

```sh
-Dsun.java2d.renderer=sun.java2d.marlin.MarlinRenderingEngine
-Dawt.useSystemAAFontSettings=on
-Dswing.aatext=true
-Dsun.java2d.xrender=true
```

**Netbeans —— 修复字体渲染**

Netbeans 的字体渲染可通过传递相同参数来修复，但它未提供便捷方式。以 root（或 sudo）身份用文本编辑器打开 **/usr/local/netbeans-8.2/etc/netbeans.conf**。若使用更新版本的 Netbeans，把上述路径中的“8.2”改为对应版本号。找到设置变量 `netbeans_default_options` 的行，把以下内容追加到选项字符串末尾。这是一整行。

```sh
-J-Dswing.aatext=true -J-Dawt.useSystemAAFontSettings=on -J-Dsun.java2d.renderer=sun.java2d.marlin.MarlinRenderingEngine -J-Dsun.java2d.xrender=true
```

## 小结

以上内容应足以让你开始开发 Java 应用，但本文未深入探讨如何编写 Java 程序、定制 IDE 或在运行 JVM 时可传递的各种选项。如有疑问，可通过 <aashiks@gmail.com> 联系我。 •

**ASHIK SALAHUDEEN** 已在计算机行业工作约 18 年，主要使用类 Unix 操作系统与开源技术。他在 Accordium 领导工程团队，业余时间参与其他 FOSS 社区——主要是印度语计算小组 Swathanthra Malayalam Computing。
