# 非 RPi 上的媒体服务器

作者：**Roller Angel**

媒体服务器是网络的绝佳补充。它允许各类播放设备发现自己并从媒体服务器串流媒体。

把一台标准的 FreeBSD RELEASE 系统变成功能齐全的媒体中心需要做些什么？其实相当简单。首先，你必须决定要安装哪种服务器。我能推荐几款。Plex Media Server 是我的首选，因为它功能完整的界面用起来很愉悦。安装它只需输入 `# pkg install -y plexmediaserver`。Plex 确实提供付费的 Plex Pass，解锁额外功能；如果你有 Plex Pass，需使用不同的命令 `# pkg install -y plexmediaserver-plexpass`。

下一个不错的媒体服务器选择是 Emby。命令是 `# pkg install -y emby-server`。最后，你也可以只装一个 mini DLNA 服务器。由于 Plex 在自带的服务器软件之外还包含内置的 DLNA 服务器，我主要就用 Plex 的 DLNA 服务器。不过，想要简单方案的人只要输入 `# pkg install -y minidlna` 即可。许多播放器都内置支持从 DLNA 服务器定位并串流媒体。安装包之后阅读显示的信息很重要。要再次调出该信息，输入 `pkg info -D` 和包名。若希望媒体服务器在 FreeBSD 启动时自动启动，记得在 **/etc/rc.conf** 中添加条目。

## 配置

安装完成后需要做一些配置。这主要就是告诉服务器软件应该去硬盘的哪些位置寻找不同类型的媒体。把文件夹提供给软件，它会做扫描，不仅把文件纳入系统以便对外提供服务，还会尝试从互联网上匹配文件的附加信息。例如，如果你的某个文件是位于 Movies 文件夹下名为 `tron.mkv` 的电影《Tron: Legacy》，Plex 会看到它，Plex 界面中 Tron 的列表就不会显示文件名，而是漂亮地填满电影海报、演员姓名、剧情简介等许多内容。Plex 和 Emby 还有可安装在手机和智能电视上的应用，让你也可以从这些设备访问媒体服务器，而不只是从 Web 界面。

我带你过一遍配置 Plex Media Server，因为它是我最常用的。其他媒体服务器的过程大同小异。顺便说一句，我的 Plex 运行在一台 AMD64 桌面机上，正是我在 2019 年 1/2 月号 FreeBSD 期刊上撰写的《A Guide To Getting Started with FreeBSD》一文中的那台机器。安装后，我通常希望 Plex 开机自启，于是输入 `# sysrc plexmediaserver_plexpass_enable=YES`。可以看到，我有 Plex Pass。我主要用它把媒体从服务器同步到 Android 手机，以便离线时访问。我还觉得电影预告片功能很方便，可以快速看个预告片，决定是否要看那部还没看过的电影。

接下来启动 Plex，输入 `# service plexmediaserver_plexpass start`。Plex 跑起来后，在与媒体服务器同一网络的 Web 浏览器中打开下面的 URL；注意我的 IP 可能和你的不同，但端口和路径应该相同——**172.16.28.2:32400/web**。在该页面上登录 Plex 账号，告诉 Plex 我喜欢把音乐类媒体放在 **/mediacenter/Music** 数据集中，电影类媒体放在 **/mediacenter/Movies** 数据集中，电视剧类媒体放在 **/mediacenter/Tv Shows** 数据集中。设置好几种媒体类型并完成欢迎向导后，Plex 会扫描所有媒体，并尝试将能识别的媒体与互联网上的资料匹配。如果 Plex 猜错了，我可以取消匹配并使用界面查看它能找到哪些相似匹配，也可以直接提供自己的内容，比如海报、标题和简介。

家里那几台旧电视，我插了 谷歌 Chromecast。可以用 Android 手机上的 Plex 应用把任意媒体文件投屏到电视上。至于那台最新的 Samsung Smart TV，只要在电视上进入应用商店安装 Plex，就能直接用遥控器操作，串流任意媒体文件。开车上班时，我把音乐同步到 Android 手机，可以在离线模式下播放，省下不在 WiFi 时的流量。如果手机流量不是问题，可以考虑在家里搭建 OpenVPN，这样就能从设备通过 VPN 轻松远程连接到媒体服务器。如果你不想搭建 VPN，Plex 也支持在家庭网络之外访问媒体，网上有大量相关配置信息。我更愿意使用远程访问 VPN 来访问家里的机器。

## Jail

考虑把媒体服务器跑在 jail 里。Michael W Lucas 有一本帮你精通 jail 的新书，<https://mwl.io/nonfiction/os#fmjail>，书中讲解了 jail 的所有益处和特性。书中还有一条对媒体服务器很实用的技巧，与加速 FreeBSD 上的 Python 有关。显然，在 **/etc/fstab** 中挂载 **/dev/fd** 可以加速 Python。太好了！只要在 **/etc/fstab** 中加入下面这样的条目：

```ini
fdescfs /dev/fd fdescfs rw 0 0
```

## ZFS 数据集

还要考虑如何从媒体服务器操作你的文件。媒体服务器界面中的删除选项重要吗？那么应用就需要对媒体文件有读写权限。如果你愿意，也可以让服务器只有对媒体文件的读访问权限。考虑为每种媒体类型使用 ZFS 数据集。用独立的数据集，你可以试验 ZFS，根据典型媒体属性优化存储。电影文件通常很大；音频文件一般不大，但显然会有出入。照片也是一样。了解典型文件大小对调优 ZFS 数据集很有帮助。FreeBSD Handbook 是了解 ZFS 的优秀资源，还可以到 ZFSbook.com 进一步深入研究。 •

---

**Roller Angel** 是热衷 BSD 的用户，享受 BSD 技术带来的种种精彩。他基于 FreeBSD 开过编程工作坊，正在搭建一个在线培训平台，教授 BSD 和相关技术。详见 BSD.pw。
