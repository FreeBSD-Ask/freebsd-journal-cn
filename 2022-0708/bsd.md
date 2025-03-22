# FreeBSD 入门研讨会

- 原文链接：[Getting Started with FreeBSD Workshop](https://freebsdfoundation.org/wp-content/uploads/2022/08/angel_workshop.pdf)
- 作者：**ROLLER ANGEL**

## 与 Front Range BSD 用户组的合作

我与 **Front Range BSD 用户组** 合作，已连续几年在 **SCaLE**（Southern California Linux Expo）上教授《FreeBSD 入门》（Getting Started with FreeBSD）研讨会。我之所以这样做，是因为我深知 FreeBSD 工具的强大，也乐于通过研讨会分享自己的经验。  

每次授课后，我都会收获新的见解，对自己的 FreeBSD 配置方案进行验证，同时也能得到学员们的反馈，持续改进研讨会内容。最近的一次授课是在 **SCaLE 19x**，而在此之前，我也曾在 **SCaLE 17x 和 18x** 进行过类似的分享。  

![](https://github.com/user-attachments/assets/629172b4-8f59-4aa8-a05e-05d3b781a141)


## SCaLE 19x 研讨会回顾

学员们陆续走进会场，映入眼帘的是投影屏幕上《FreeBSD 入门》的标题幻灯片——由 FreeBSD 基金会的市场协调员 **Drew Gurkowski** 设计。这次演示还通过 YouTube 进行了直播，链接如下：  
📺 [观看直播](https://www.youtube.com/watch?v=ByFCRwMJATM)  

往年，研讨会通常由 FreeBSD 基金会的执行董事 **Deb Goodkin** 进行开场介绍，而这一次，**Drew Gurkowski** 出色地完成了这一任务。  

在研讨会中，我们的目标是确保学员能够克服困难，顺利完成安装过程。由于现场没有专门的实验室设备，我们使用 **VirtualBox** 创建和配置虚拟机，然后在其中安装 FreeBSD。  

基本流程如下：  

1. **插入虚拟光盘**：将 FreeBSD 安装 ISO 挂载到虚拟机的光驱。  
2. **从光盘启动并安装 FreeBSD**：引导虚拟机从光盘启动，然后将 FreeBSD 安装到虚拟硬盘。  
3. **完成安装后执行 `shutdown -p now` 关机**：这样可以**移除虚拟光盘**，避免下次启动时再次从光盘引导。  
4. **后续引导时改为从硬盘启动**。  

在这一阶段，我们会进行短暂的休息，我会利用这段时间帮助尚未跟上的学员。  



## 常见问题与解决方案

在研讨会中，我遇到的最常见问题主要与虚拟机的错误配置*有关。例如：  

- **I/O APIC 选项被禁用**：有人在 VirtualBox 的系统设置中关闭了 **“Enable I/O APIC”**，导致无法正确启动。只需勾选此选项即可修复问题。  
- **虚拟机架构设置错误**：有学员将虚拟机类型设为 32 位，而 FreeBSD 需要 64 位。改为 64 位后，问题迎刃而解。  
- **输入错误的包名和配置**：在排查问题时，即使是个小小的拼写错误，也可能导致系统无法正常运行。  

**新问题**：随着 Apple M1/M2 处理器的普及，一些学员无法在 Mac 设备上使用 VirtualBox，因为 VirtualBox 不支持 Apple 的 ARM 架构。  

此外，我们还遇到了会场 WiFi 带来的小问题：  

- 由于 DHCP 提供的 DNS 设置有问题，我们不得不手动修改 `/etc/resolv.conf` 文件中的 **nameserver** 配置，以保证网络连接正常。  

![](https://github.com/user-attachments/assets/54d3652f-6654-4bfa-9d87-fabe3b078a80)


## 总结  

每年的 SCaLE 研讨会都让我受益匪浅，我不仅能帮助学员入门 FreeBSD，也能从他们的反馈中不断优化课程内容。将来，我希望能继续完善这一研讨会，让更多人了解并使用 FreeBSD！

## 服务器与桌面的界限

接下来的步骤是演示服务器与桌面之间的界限有多么模糊。只需安装几个软件包并修改一些配置文件，桌面环境就能立即启动。我们只需运行 `startx` 命令，就可以让 FreeBSD 启动图形桌面。  

当学员们意识到他们已经自己搭建了完整的桌面环境，并且不需要特定的发行版或 FreeBSD 衍生系统来运行 X Window 系统窗口管理器时，这一刻是非常令人兴奋的。例如，我们使用了 **XFCE**，但也演示了如何轻松安装和配置 **KDE Plasma 5、Lumina 和 GNOME** 等其他窗口管理器。  

在窗口管理器运行后，学员们可以与 GUI 应用程序进行交互，例如网页浏览器、编程 IDE、文件管理器等，进一步体验 FreeBSD 作为桌面操作系统的能力。  

## 构建自定义软件包仓库

在研讨会中，我还专门介绍了构建自定义软件包仓库（package repository）的过程。这很重要，因为如果学员遇到需要自定义 ports 的情况，他们应该知道如何避免混合使用 ports 和二进制软件包的常见问题。  

我们使用的工具是 **Poudriere**，它让构建自己的软件包仓库变得非常简单和直观。  


## 自动化管理与 Ansible

在学员们开始熟悉命令行输入后，我们引入了 **Ansible**，这是一款用于**配置管理**的工具，特别适用于通过 SSH 远程管理多台服务器。  

我们演示了如何克隆 FreeBSD 虚拟机，并通过 SSH 连接到它。然后，我们使用 **Ansible Playbooks** 来自动执行命令，帮助学员理解 **Ansible** 在 FreeBSD 环境中的应用。  

研讨会中，我们还提供了**一个 Ansible Playbook**，用于在远程服务器上搭建 Poudriere，并构建自定义软件包：  

1. **租用一台高性能服务器**，短时间内完成软件包的编译。  
2. **下载编译好的软件包**到本地，之后销毁服务器，减少成本。  
3. **修改软件包仓库设置**，将 *FreeBSD 软件包源从默认的 `https://download.FreeBSD.org` 改为本地路径（`file://`），以便使用我们自定义构建的软件包。  

## FreeBSD Jail 与 iocage

![](https://github.com/user-attachments/assets/1d2d7683-37fd-41fe-9d9c-f1a0f596048c)


最后，我们讨论了 **FreeBSD Jail**，让学员体验它们的强大功能，并学习如何使用 **iocage** 进行管理。  

对于希望深入学习的学员，我们推荐 **MWL.io**，该网站提供了 **Michael W Lucas** 撰写的 FreeBSD 书籍，包括：  

- **《FreeBSD Mastery》** 系列  
- **《Poudriere 使用指南》**  
- **《FreeBSD 安装手册》**  
- **《FreeBSD Jail 深度解析》**  

这些资源可以帮助学员在研讨会之后进一步拓展知识，掌握 FreeBSD 高级功能。

## 研讨会参与者

这次研讨会的参与者构成非常多元化，大家来自不同背景，经验水平各异，但都沉浸在技术讨论中，探索新知识，以便在未来的工作中更好地应用 FreeBSD。只要掌握了安装与配置的基本流程，就能很快发现 FreeBSD 在解决实际问题中的优势。而借助 **Ansible** 等配置管理工具，学员不仅可以记录所有配置文件的修改和已安装的软件包，还能轻松恢复进度，进一步深化对 FreeBSD 的理解和应用。  

这次的学员们热情高涨，有些人甚至专门带来了笔记本电脑，计划在现场安装 FreeBSD。其中，一些人对 FreeBSD 的 WiFi 配置颇感兴趣。对于这类问题，我们提供了一些简单的方法：  

- **USB 共享网络（USB Tethering）**：使用 **Android 设备** 共享网络连接，只需插入数据线、启用 USB 共享网络，然后以 root 用户运行：  

  ```sh
  dhclient ue0
  ```  

  该命令会通过 **DHCP** 获取第一个 USB 以太网设备的 IP 地址。  
- **内部 WiFi 配置**：可通过 **/etc/rc.conf** 和 **/etc/wpa_supplicant.conf** 进行配置，详细选项请参考 **FreeBSD 手册的第 5 章**。  
- **更多 WiFi 相关信息**：请参阅 **《FreeBSD 手册》第 32 章：高级网络配置**，其中包含 FreeBSD 对 WiFi 的详细支持信息。  

我们的目标不仅是回答常见问题，更是帮助学员利用 FreeBSD 做出有趣的事情，并用技术解决现实问题。  



## 研讨会后的小故事

值得一提的是，一位学员中途才到达，进度严重落后，无法跟上课程。但在研讨会结束后，我在大厅里陪他一起调试，最终帮助他完成了整个安装过程。由于他的笔记本使用的是酷睿 2 Duo 处理器，安装过程比其他人耗时更久。不过，他对我的帮助非常感激，并表示对 BSD 生态充满兴趣，希望能进一步学习。  

我向他推荐了几个学习资源：  

- **《FreeBSD Journal》**——权威的 FreeBSD 期刊  
- **MWL.io**——Michael W Lucas 的技术书籍网站  
- **BSD 用户组（BSD User Groups）**——加入社区，和更多 BSD 爱好者交流  

此外，我也告诉他，我的个人网站 [http://BSD.pw](http://BSD.pw) **随时欢迎他来交流**。  

---

**ROLLER ANGEL** 主要致力于帮助人们通过技术实现目标，他是一名经验丰富的 FreeBSD 系统管理员 和 Python 开发者，热衷于探索开源技术（尤其是 FreeBSD 和 Python）在解决实际问题中的应用。他坚信，只要有心学习，就能掌握任何知识。  

他喜欢挑战问题，善于寻找创造性解决方案，不断学习新技术，保持技能的敏锐性。此外，他还活跃于研究社区，乐于分享自己的想法，并通过技术推动创新。
