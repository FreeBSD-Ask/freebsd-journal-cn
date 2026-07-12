# 实用软件：实现无纸化（Paperless）

- 原文链接：[Practical Ports: Go Paperless](https://freebsdfoundation.org/our-work/journal/browser-based-edition/kernel-development/practical-ports-go-paperless/)
- 作者：Benedict Reuschling

![实用软件](../png/2024-0910/Go-1.png)

在疫情最严重的时候，我一直待在家里，日子基本上和其他日子没什么区别。时间过得很慢，环顾四周，我痛苦地意识到我的房间变得相当凌乱。尤其是我的工作桌旁，堆积的书籍、笔记、信件和其他纸张已经开始争相堆到桌面上。我决定做点什么。我打开播客，开始整理、归类、扔掉，一直清理到桌面整洁。这种巨大的成就感在全球危机时期格外强烈。受到这一感觉的激励，我决定开始扫描那些纸张，减少堆积的纸张数量。

我前阵子买了一台移动扫描仪，它通过 USB 连接，并使用一些专有但高效的扫描软件。这个扫描仪的大小不亚于擀面杖，能够扫描一张纸。当时正好有时间，我继续一张一张地扫描每张纸，为每个生成的 PDF 添加描述和日期，然后继续扫描下一张。虽然这很繁琐，但最终我完成了。之后整理这些文件又是一个漫长的任务：税务、保险、合同、收据、账单（包括已发送和已收到的）、记录、证书等，都需要放入正确的目录中。

扫描软件只有 32 位，几年后整个扫描设备停止工作了——恰好在另一堆纸危险地接近倒塌的时候。是时候寻找替代方案了。几个月前，我发现了 `textproc/py-ocrmypdf`。这款软件本质上是重新扫描 PDF，将有时候是一张大图的文字转换为可以提取和搜索单个单词的文本。它通过模式识别来识别单词和语言，结果出奇地好。我将其应用于现有的扫描文件，现在我可以对文档进行全文搜索，例如找到我参加 BSDCan 2014 时的机票费用。

## 介绍 Paperless-ngx

然后我发现了 paperless-ngx，`ocrmypdf` 是它的一部分，但相对整体而言只是较小的一部分。它是文档管理系统，你可以将文档导入其中，它会创建完全可搜索的档案，自动检测内容（AI 无处不在）并根据你定义的规则将其归档。基本上，我可以把一封信交给它，它会识别出这是来自银行的文件，并根据检测到的日期将其归档，同时使用 `ocrmypdf` 使其变得可搜索，并添加有用的元数据。文件最终存储在目录中，目录可以是主题相关的（例如我从该银行收到的所有文件），或按年 - 月 - 日 - 描述分类，或完全由你决定。你还可以将整个目录导入到 paperless-ngx，它会自动识别哪些文档扫描过，并跳过那些文档，剩下的则通过处理管道处理。随着每个文档的处理，未来类似文档被正确分类的概率也会增加。此外，它还配备了漂亮的 Web 用户界面，可以轻松地将文件拖放到界面中进行扫描，并方便地找到已有文档。另一种将文档导入的方法是通过“incoming”文件夹，你可以在办公室与同事共享这个文件夹，或者将文件作为附件（还记得电子邮件吗？）发送给它。

该软件堆栈本身令人印象深刻，甚至可能太令人生畏，尽管 [paperless-ngx 网站](https://docs.paperless-ngx.com/) 提供了优秀的文档。为了获得顺畅的扫描体验，许多软件和服务需要协同工作。幸运的是，`deskutils/py-paperless-ngx` 下有相关的 Port。更棒的是，Port 维护者创建了一份安装后手册页，详细列出了如何启动正常工作的 paperless-ngx 堆栈的所有步骤。我是不是提到过，我真心喜欢 Port 维护者？有了这些说明，我迅速设置好了自己的 paperless-ngx。首先在树莓派 3 上设置，然后在 Pi 4 上也成功运行了。尽管 Pi 3 由于处理能力有限，可能需要更多的耐心来获得最终结果，但 Pi 4 体验很好，扫描时间也合适。你可以在办公室或家里运行它，几乎不会对电费账单产生影响，同时允许其他人扫描文档而不会看到他人的文件。如果你处理很多文档并希望将其数字化，看看我们正在进行的 paperless-ngx 设置吧，等你体验之后再感谢我…

## Paperless-ngx 设置

无论你使用的是树莓派、其他嵌入式设备，还是一台完整的服务器，其实并不重要。只要它运行 FreeBSD，你就可以跟着做。我不会花时间介绍基础安装或系统硬化，因为有很多其他好文章覆盖了这些内容。只要确保在将你的 paperless-ngx 服务连接到网络，供其他人使用时，做好相应的安全配置。

首先，安装 Port paperless-ngx：

```sh
# pkg install deskutils/py-paperless-ngx
```

安装完成后，你将看到 `pkg-message`，其中建议你查看手册页以获取更多说明。如果没有这些说明，你将只能使用基本服务，此时它的功能并不多。

大多数文件最终会存储在 **/var/db/paperless**，你可能希望将它放在单独的 ZFS 数据集上，但根据我的经验，压缩节省的空间并不值得。不过，你的使用情况可能不同，ZFS 通常是存储这些重要文档的好选择。

Paperless-ngx 需要访问 Redis 实例，接下来我们将安装它：

```sh
# pkg install redis
# service redis enable
# service redis start
```

非常简单，使用这三条命令，你可以安装并设置 Redis，使其开机时自动启动，并且当前会话也能运行。如果你的 Redis 实例运行在网络中的其他地方，你需要修改 **/usr/local/etc/paperless.conf** 并添加相应凭据。如果在本地主机上运行，默认情况下无需特殊权限，因为这样它不会被其他主机访问。

配置文件有很好的文档说明，包含了注释。像 `THREADS_PER_WORKER`（在我的 RPI 4 上设置为 1）、`PAPERLESS_URL`（IP 地址或 DNS 名称）和 `PAPERLESS_TIME_ZONE`（我使用的是 UTF）等项需要根据你的系统和网络修改。许多其他设置在初始扫描时默认配置就足够了。你可以稍后回头查看此文件并修改。

Paperless-ngx 使用数据库来存储各种信息。初始化数据库非常简单，可以使用以下命令：

```sh
# service paperless-migrate onestart
```

如果你希望每次系统启动时都运行这个服务，也可以执行这个 `service` 命令：

```sh
# service paperless-migrate enable
```

完成之后，我们将按顺序启动 Paperless 所使用的后台服务：

```sh
# service paperless-beat enable
# service paperless-consumer enable
# service paperless-webui enable
# service paperless-worker enable
```

你可以在 paperless-ngx 网站上找到这些服务的详细描述。我们希望在不重启系统的情况下使用 paperless-ngx，接下来启动所有这些服务：

```sh
# service paperless-beat start
# service paperless-consumer start
# service paperless-webui start
# service paperless-worker start
```

机器学习是人工智能炒作背后的热门话题。Paperless-ngx 也使用了机器学习，但主要用于辅助字符识别，以确定当前文档的语言。为此，它使用了自然语言工具包（NLTK）。要下载必要的文件，可以使用以下单行命令（如有需要，请替换 Python 版本）：

```sh
# su -l paperless -c '/usr/local/bin/python3.11 -m nltk.downloader \
stopwords snowball_data punkt -d /var/db/paperless/nltkdata'
```

文档以不同方式分类，这是 Celery 组件的职责。扫描时自动完成分类，但你也可以通过以下命令手动触发：

```sh
# su -l paperless -c '/usr/local/bin/paperless document_create_classifier'
```

Celery 还运行可选组件 Flower。它监控 Celery 控制的工作节点集群。这个组件是可选的，我的实例没有运行它。但对于那些想要更多功能的人，这里是启动它的方式：

```sh
# service paperless-flower enable
# service paperless-flower start
```

## 设置 Web UI

为了保护基于 Django 的 Web UI，其中存储了迄今为止所有扫描的文档，你可以像这样设置超级用户密码：

```sh
# su -l paperless -c '/usr/local/bin/paperless createsuperuser'
```

我已经在运行 nginx web 服务器（SSL 代理），所以我可以重用它来指向我的 paperless-ngx 网站。如果你还没有 web 服务器，Port 也提供了现成的配置文件，位于 **/usr/local/share/examples/paperless-ngx/nginx.conf**，你只需将其复制到 **/usr/local/etc/nginx/** 目录即可。这个配置文件还包括 SSL 配置，避免有人窃听流量，获取登录信息并做出其他恶意行为。要创建有效期为一年的密钥，可以运行以下较长的 `openssl` 命令（或者通过 `lets-encrypt` 获取密钥）：

```sh
# openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
-keyout /usr/local/etc/nginx/selfsigned.key \
-out /usr/local/etc/nginx/selfsigned.crt
```

当然，你可以在必要时调整 `nginx.conf`。完成后，启用它开机自启，并在当前会话中启动：

```sh
# service nginx enable
# service nginx start
```

Voila! 现在，你可以在浏览器中访问 `paperless.conf` 中定义的 web URL 并登录应用程序。

## Web UI 中的基础配置

在扫描第一份文档之前，我建议首先在左侧的“管理”部分设置一些项目。首先，定义“通信方”——这些是向你发送纸质文档的人或组织。可以是银行、保险公司，也可以是个人。你可以给他们描述性的名称，并配置 paperless-ngx，以便它在检测到某些关键词或其他条件时，将文档归档到该通信方下。

接下来，定义文档类型。合同与情书不同，账单与证书不同，依此类推。这样，你可以让 paperless-ngx 区分某人是向你发送账单，还是合同。两者都有可能发生，尤其是政府机构（至少在我所在的地方）往往会在不同的上下文中与你通信，而你希望将这些文档分开保存。这正是 paperless-ngx 的优势所在：一旦定义了最活跃的通信方和典型文档，你就不需要再担心正确分类了。只需添加文档，让 paperless-ngx 自动完成分类。通过一些调整，你就可以扫描大量文档。那么，如何给它们排序呢？这就是存储路径的作用。

这些路径定义了文档应存放在文件系统中的位置和目录层次结构。我个人使用的路径是 `{created_year}/{correspondent}/{title}`，这意味着我有像 `2024/insuranceXZY/YearlyReport.pdf` 这样的目录。如果你希望将所有与税务相关的文档存放在单独的目录中，请在存储路径部分定义该规则，并设置条件以匹配符合此条件的文档。最棒的是，如果你改变了排序方式，修改存储路径将自动移动并重新命名你扫描过的文档，而无需你手动执行繁琐的 `mkdir`、`cp`、`mv`、`rm` 等操作。

## 准备、开始、扫描

暂时就这些了。将一份你手头的 PDF 文档拖放到 Web UI 中，看看 paperless-ngx 如何开始处理它。左侧的“日志”部分提供了 paperless-ngx 如何选择匹配通信方和其他详细信息，帮助你调整匹配规则。处理完成后，你可以在仪表板或文档文件夹中找到最终结果。继续扫描其他文档，它们将全部存储在 **/var/db/paperless/media/documents/archive** 目录中（如果你没有在 `paperless.conf` 中修改该路径），然后按照存储路径定义进行存放。希望你能像我一样发现 paperless-ngx 对文档的管理非常有用。我总是期待收到下一封信，以便用 paperless-ngx 扫描它。感谢创建 paperless-ngx 的团队，和使 FreeBSD Port 安装体验如此出色的所有人。

---

**BENEDICT REUSCHLING** 是 FreeBSD 项目的文档提交者，也是文档工程团队的成员。过去，他曾任两届 FreeBSD 核心团队成员。他在德国达姆施塔特应用科技大学管理一个大数据集群，还为本科生教授“Unix for Developers”课程。Benedict 也是每周 bsdnow.tv 播客的主持人之一。
