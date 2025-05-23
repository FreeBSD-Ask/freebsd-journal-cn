# Tarsnap 的 FreeBSD 集群

- 原文链接：[Tarsnap’s FreeBSD Cluster](https://freebsdfoundation.org/wp-content/uploads/2021/03/Tarsnaps-FreeBSD.pdf)
- 作者：**COLIN PERCIVAL**

![](https://github.com/user-attachments/assets/1d93c338-e1f5-4e35-a198-ec4ac7d49bdc)


Tarsnap 是一项在线备份服务，注重安全性——事实上，当我在 2006 年创办这家公司时，我很快就定下了口号“为真正偏执的人提供在线备份”，以体现作为一名密码学家和 FreeBSD 安全官，我的目标是提供足够安全的服务，值得我用它来保存自己的秘密。Tarsnap 也是 FreeBSD 能够在 Amazon EC2 上运行的主要原因之一：Tarsnap 需要在 EC2 上运行（其中一个原因是能够以廉价且快速的方式访问 Amazon S3 存储服务），但我需要一个我信任并且能轻松管理的操作系统——换句话说，Tarsnap 需要在 EC2 上运行 FreeBSD，而我正好满足了这个需求。

## 定制的 AMIs

虽然 Tarsnap 使用的所有 EC2 实例都运行 FreeBSD，但从未使用 FreeBSD 项目发布的“标准”FreeBSD Amazon Machine Images（AMIs）。相反，我使用我五年前创建的工具来构建轻度定制的 FreeBSD 镜像：“AMI Builder”AMI，适用于 FreeBSD 12.2-RELEASE，在 us-east-1 EC2 区域的 ami-085ee41974babf1f1。这些 AMI Builder 并不是 FreeBSD 项目提供的，而是我在每次发布后自行构建并发布的；希望在某个时候，我能将这些集成到发布工程团队执行的构建过程中。

为了创建一个“Tarsnap FreeBSD 12.2-RELEASE”镜像，我首先启动上述的 AMI Builder，然后大约等待 20 分钟，直到虚拟机启动并将（标准的）FreeBSD 12.2-RELEASE 安装到其虚拟磁盘上。然后将该磁盘挂载到 /mnt/，同时运行在内存磁盘上的 FreeBSD 系统从 / 启动并启动一个 sshd 进程。

待我可以通过 SSH 连接到 AMI Builder（像其他 EC2 中的 FreeBSD 镜像一样，使用我提供给 EC2 的 SSH 密钥和用户名 ec2-user），我就开始进行一些我希望在所有 Tarsnap 系统中使用的标准配置：

- 我从 FreeBSD ports 树中构建并安装一些软件包：pkg、djbdns、qmail、spiped 和 tarsnap。
- 我启用一些我需要的守护进程（svscan 和 spiped），禁用其他我不需要的代码（sendmail 和 firstboot pkg 安装），并在 `/etc/rc.conf` 中锁定一些设置（禁用 syslogd 中的网络监听并限制 sshd 只使用 IPv4）。
- 我指示 pkg 使用我自己构建的包，而不是 FreeBSD 项目发布的包，方法是在 `/usr/local/etc/pkg/repos/` 中创建配置文件。
- 我添加定时任务每天早上运行 `freebsd-update cron` 和 `pkg upgrade -qn`——我不希望自动安装更新，但我绝对希望在更新可用时收到电子邮件通知。
- 我设置 djbdns 提供本地 DNS 缓存，并将 `resolv.conf` 指向它。
- 我设置 qmail 通过我的邮件服务器发送外发邮件。
- 我设置 spiped 创建与我的邮件和包服务器的安全连接，并封装进入的 SSH 连接。

最后，在对 `/mnt/` 执行完我想要的所有配置后，我卸载磁盘并要求 EC2“从运行中的 EC2 实例创建一个 AMI”。尽管这个 EC2 API 调用的描述是这样的，但创建的 AMI 并不反映正在运行的状态，而是磁盘上的当前状态——换句话说，它忽略了从内存磁盘上运行的 FreeBSD 系统，并创建了一个与我在挂载在 /mnt/ 上的文件系统上执行的配置相对应的 AMI。

现在，我有了一个已配置的“Tarsnap FreeBSD”镜像，从中可以启动实例，所有我喜欢的默认设置都已配置好，最后一步是：我要求 EC2 将这个 AMI 从 us-east-1 区域复制到 us-west-2 区域。虽然 Tarsnap 几乎所有的服务器都运行在 us-east-1（当我启动 Tarsnap 时，us-east-1 是唯一的 EC2 区域），但我在 us-west-2 也有一个系统：一个监控系统，能在出现故障时提醒我。

## pkg 构建器

FreeBSD 项目提供了用于 ports 树中软件的二进制包，大多数用户希望使用这些包，而不是从源代码构建。对于 Tarsnap 的服务器，我自己构建包，主要有两个原因：

- 在某些情况下，我希望使用非默认的 Port 选项。
- 作为一个 FreeBSD 开发者，有时我会提交更新并希望立即使用它们，而不是等到下一个计划的包集发布。

可能在未来某个时候，这两种情况都不再适用——也许我设置了非默认选项的 Port 会获得提供我需要功能的“flavors”，也许 FreeBSD 项目将来会在每次 Port 更新时进行极其快速的包构建（但考虑到新的编译器发布速度不断加快且性能不断下降，这似乎不太可能）。在此期间，运行我自己的包构建既简单又方便。

我为此使用 poudriere，设置构建过程非常简单：安装 poudriere 后，只需运行 `poudriere ports -c` 和 `poudriere jail -c -j JAILNAME -v 12.2-RELEASE`。之后，我在 `/usr/local/etc/poudriere.d/make.conf` 中设置一些选项，设置一个 cron 任务来运行 `poudriere bulk -f /root/pkgs-wanted`（其中列出了我需要的包），并使用 lighttpd 来提供生成的包。

在 Tarsnap 服务器上构建我使用的完整包集大约需要 2 小时，在我使用的每月 15 美元的“t3.small”EC2 实例上，但大多数包构建运行比这快得多，因为 poudriere 以增量方式运行，不会重新编译未更改的包。事实上，如果不是有一个细节，我可能会使用一个更小的 EC2 实例：一些 C++ 编译在仅有 1 GB 内存的实例上会失败。

## Web 服务器

Tarsnap 的“集群”包括两台非常轻量级的 Web 服务器，运行在 EC2“t3.nano”实例上。在一个 Web 框架和丰富的 Web 应用程序使得 JavaScript 函数调用成为常态的时代，Tarsnap 网站可能特别之处在于其古老的设计：网站的公开部分完全是静态 HTML——除了一个用来包装内容、添加头部和导航部分的 shell 脚本外，其他完全是手写的——账户创建和管理代码由一些 CGI 脚本组成……这些脚本是用 C 语言编写的。虽然 CGI 脚本存在固有的性能问题——派生一个进程比在 Web 服务器上下文中运行代码或将请求转发到另一个长时间运行的守护进程要昂贵得多——但就 Tarsnap 而言，我倒希望能有因过多用户使用网站而导致的性能问题。

然而，Web 服务器也有一些稍微现代化的方面：TLS 流量通过 hitch 解密，以保持加密代码与 Web 服务器代码的隔离；TLS 证书通过 Let’s Encrypt 和 certbot 获取；TLS 私钥存储在 Amazon Elastic File System（即 NFS 文件系统）中。虽然 hitch、Let’s Encrypt 和 certbot 的使用非常常见，但 Amazon EFS 的使用可能值得一些解释。

使用 Amazon EFS 解决了一个引导问题：我通过“webroot”机制获取 TLS 证书，其中证书颁发请求通过将文件放置到网站的 `/.wellknown/acme-challenge` 目录中来进行授权。这只有在网站的流量被定向到相应的服务器时才能生效——但在服务器准备好响应 HTTPS 请求之前，我不想将流量定向到新的主机。将私钥存储在一个能够在 Web 服务器实例更换后依然存活的地方，解决了这个问题。

现在，NFS 并不以安全性著称，而且以明文方式运行，这似乎并不适合存储加密材料——但事实上，TLS 所提供的保证现在已经足够脆弱，以至于这里不会丧失任何安全性。如果攻击者能够截获 Amazon EC2 网络上的流量，他们可以欺骗 Let’s Encrypt 向他们颁发新证书，从而冒充 Tarsnap 的 Web 服务器——因此，拥有另一个依赖于截获 Amazon EC2 网络流量的攻击并不会扩展他们的能力。

## 邮件服务器

Tarsnap 的所有电子邮件都通过一台邮件服务器进行路由。包括：

- 我与外界之间发送的“人工”邮件。
- 由 cron 任务生成的“日志”邮件（这些邮件会进入我的收件箱）。
- 当用户注册 Tarsnap、确认其电子邮件地址、进行付款或需要提醒其（预付）账户余额不足时生成的交易邮件。
- 公共邮件列表，既包括 Tarsnap 的公告和讨论，也包括源自 Tarsnap 的开源软件。

出于历史原因，Tarsnap 的邮件服务器使用 qmail；具体来说，“这是我在大约 20 年前配置我的第一个 FreeBSD 服务器时开始使用的。”如果我从零开始，我可能会使用 postfix，但按照系统管理员的长期传统，只要它没有坏，我就不太可能去修复它。

电子邮件通过两种方式到达邮件服务器：通过连接到外部网络接口的 25 端口，或通过连接到 spiped 的 8025  端口——然后通过环回接口的 25  端口到达 qmail。以这种方式使用 spiped 不仅加密了“内部”网络流量，还巧妙地解决了邮件中继的问题：任何通过环回接口到达 qmail 的邮件都可以安全地转发到其他域。

由于我使用 qmail，自然使用 Dan Bernstein 的 ezmlm（及其扩展 ezmlm-idx）来管理 Tarsnap 的邮件列表。“tarsnap 公告”列表是有管理的，但其他列表对任何订阅者开放；幸运的是，我没有遇到过恶意评论问题，到目前为止，垃圾邮件机器人似乎还不够聪明，不能在尝试通过邮件列表发送邮件之前先订阅邮件列表。

Tarsnap 的邮件列表有通过 HTTP/HTTPS 访问的档案。我使用 mhonarc 将 ezmlm-idx 中的邮件列表帖子转换为 HTML 格式；使用 lighttpd 向全世界提供这些内容；并使用 hitch 为那些需要的人添加 TLS 层。在这里，很难想象 TLS 带来了什么安全性：邮件列表帖子是公开的，传输的字节数足以唯一标识正在下载的页面；尽管如此，一些人仍然强烈倾向于使用 TLS，即使它没有任何实际用途。

最后，外发邮件通过一个 shell 脚本分发，该脚本运行两个“qmail-remote”程序之一：原始的 qmail-remote，它通过 SMTP 直接发送邮件，或者我编写的“qmail-remote-ses”，它通过亚马逊的简单邮件服务（SES）发送邮件。不幸的是，将邮件送入收件箱变得越来越困难，因此对于交易邮件，我将任务交给亚马逊；以每千封邮件 0.10 美元的费用，亚马逊在将邮件送达方面不需要非常优秀，就能为客户提供服务，尤其是那些愿意付钱注册的人。另一方面，那些有过于严格邮件服务器的人不太可能订阅 Tarsnap 的邮件列表——因此，我在依赖 qmail 和直接 SMTP 发送邮件流量时并不感到问题。

## 监控

如前所述，尽管 Tarsnap 的大多数系统都位于亚马逊的 us-east-1 区域，我在 us-west-2 区域有一个监控系统。将该实例放在不同的区域有两个原因：首先，它可以检测与 AWS 区域外部网络连接性相关的故障；其次，它可以最大程度地减少故障同时禁用 Tarsnap 系统和监控的可能性。此外，即使故障确实导致两个 AWS 区域同时离线，（a）我不太可能做出响应，（b）很可能世界面临的更重要问题比 Tarsnap 离线更值得关注。

与使用任何广泛应用的监控框架不同，我使用一小部分简单的 shell 脚本来执行一系列监控任务（ping 服务器、获取网页、执行 Tarsnap 备份），这些任务由 cron 作业执行；每个脚本都会发出状态（“GOOD”、“FAILED”或“TIMED OUT”），另一个脚本使用 Twilio 发送短信并拨打电话。少量通知与 Twilio 基于使用的定价相结合，使得这种方法非常实惠——事实上，大部分费用是每月 1 美元的租号费，这是为了能够发送短信所必需的。

## 将一切联系在一起：spiped

细心的读者可能已经注意到，我几次提到 spiped，但没有给出很多细节。由于这是一个不常用的工具——而且是我专门为确保 Tarsnap 系统之间的连接而编写的——我认为它值得特别关注。

spiped 实用程序本质上是一个以加密安全的方式将一个套接字地址连接到另一个套接字地址的工具。从这个意义上说，它可以被看作是 stunnel 或“ssh -L”Port 转发的替代品；但是，与 stunnel 依赖过于复杂的 TLS 协议和证书，ssh 使用持久的 TCP 连接在其上复用隧道连接不同，spiped 使用预共享密钥，并为每个正在转发的连接打开一个新的连接——使得它更加简单，并且在网络故障方面具有与 TCP 相同的行为。目前，spiped 支持 IPv4 和 IPv6 上的 TCP 套接字，以及本地（“UNIX”）套接字。

该名称的起源——“secure pipe daemon”（安全管道守护进程）——可能会为其意图提供另一个线索：在 1973 年 UNIX 引入管道后，功能的爆炸式增长让工具以各种方式组合，我希望开发一种使用多种守护进程的软件——并能够连接它们，而不管它们最终是否会在同一主机上运行，或是在不安全的网络中运行。

如上所述，我使用 spiped 来保护与 Tarsnap 邮件服务器的 SMTP 连接；我还使用它来保护 POP3 连接，这样我可以以最简单且最不安全的方式使用 POP3——不使用 TLS 和明文密码——同时保持必要的安全性。（嘿，这管用，好吗？我已经打算转向 IMAP 几十年了。）

类似地，连接到 pkg 构建器的连接也由 spiped 保护；虽然包的内容并不敏感（它们都直接来自 FreeBSD 的 ports 树），但这让我能够确保包的完整性，而无需使用更复杂的选项，如使用 poudriere 对包进行签名，且在（不太可能的情况下）发现 lighttpd 中的漏洞时，也可以在某种程度上保护 pkg 构建器。

最后，我使用 spiped 保护我所有的 SSH 连接——虽然我使用的所有系统都启用了 sshd，以便我可以轻松地进行管理，但 TCP/22  Port 被阻塞，唯一可以访问 sshd 的方式是通过一个 spiped 进程，直到传入的连接经过加密认证后，才允许任何流量通过。

## 遗留系统

当然，除了前述的基础设施外，还有 Tarsnap 备份服务本身。这运行在可以称为“遗留”系统上的：由于它直接负责确保客户数据的安全——更不用说带来收入——我允许备份服务落后于其他系统（当然，安全方面除外）。目前，它使用的是较旧版本的 FreeBSD 和较旧的 EC2 实例类型——这一事实使它免受（最近新增的）ENA 网络接口驱动程序中的稳定性问题的影响，这些问题已在 FreeBSD-EN-20:11.ena Errata Notice 中得到修复——且唯一安装的第三方包是我在“Tarsnap FreeBSD AMI”中拥有的包——即那些需要让我安全连接进行管理的包，以及用于发送外发邮件的包。除此之外，唯一运行的代码是 Tarsnap 的专有代码——一些守护进程，以及用于内部性能监控和夜间计费的 cron 作业。

随着时间的推移，我预计会在备份服务中使用更多开源代码——或者我应该说，我已经发布了开源代码，预计未来会在备份服务中使用。我围绕 Tarsnap 对元数据存储的需求设计了 kivaloo 数据存储，最初发布代码并在以后使用它才是有意义的。Tarsnap 将客户数据存储在 Amazon S3 中，但为了能够按需检索数据，有必要跟踪每个数据块存储的位置。这导致了一种数据存储使用模式——键和值大约为 40 字节的表——对于大多数数据存储系统而言，优化不佳。

## 未来方向

在结尾之前，我不得不提到一件 Tarsnap 目前还未使用，但我希望能够使用的技术：“Graviton 2”arm64 EC2 实例。在我至今的所有测试中，这些实例表现非常好——而且它们的费用大约比 Tarsnap 当前使用的 x86 EC2 实例便宜 40%。

自然，虽然 arm64 EC2 实例有可用的 FreeBSD 镜像，但 Tarsnap 在完全支持 arm64 架构之前（包括通过 FreeBSD Update 提供二进制更新）不会使用它们（除了开发和测试）。我了解到这可能会在 FreeBSD 13.0 发布时实现；所以，到明年此时，我可能会用新的 arm64 实例替换许多系统。（核心备份服务，当然，仍将继续在 x86 上运行一段时间。）只有时间能证明，但可以肯定地说，这是一个让我对未来非常兴奋的领域。

---

**COLIN PERCIVAL** 自 2004 年起成为 FreeBSD 开发者，并于 2005 至 2012 年期间担任 FreeBSD 项目的安全官。2006 年，他创立了 Tarsnap 在线备份服务，并继续运营该服务。2019 年，因其将 FreeBSD 引入 EC2 的贡献，他被评为亚马逊 Web 服务英雄。
