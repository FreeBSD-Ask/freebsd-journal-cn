# 为什么要为 FreeBSD 系统设置 PF？

- 原文标题：Why Set Up PF on Your FreeBSD Systems? — A Handful of Reasons... Read On...
- 作者：**Peter N. M. Hansteen**

## 还记得今夏窗口大小的麻烦吗？

2014 年北半球的夏天漫长而炎热，但如果我们暂时忽略涉及 OpenSSL 代码的那一连串安全公告和恐吓性报道，这个夏天对 BSD 而言也相当平静，几乎没有严重的安全问题。直到我注意到那封友善的催稿提醒，要从积压的待办事项中找出这篇 FreeBSD 相关文章的那天为止。

编号 FreeBSD-SA-14:19.tcp 的安全公告，本质上又是一个缺失的边界检查——检查传入的某段数据是否落在预期范围内。本例中的数据是属于已建立的 TCP 连接的数据包的序列号，理应匹配一个预期的区间或窗口，即允许的值范围。在某些情况下，缺失边界检查最多只是“无伤大雅”，最坏也不过是有些尴尬。但这段代码深植于网络栈中，可能的后果可以预见地严重：若能正确猜中端点地址和端口号，攻击者就能用一个精心构造的、来自伪造地址的 TCP SYN 拆除一条连接。

实际成功的几率不大，但也不算太糟：TCP 端口是 16 位数字，两边都猜中只会让可能值列表翻倍到 17 位，即十进制的 131072。然而在现实中，可能值的集合要小得多——TCP 连接的目标端口绝大多数都在知名端口列表中，其中大部分低于 1024，列在 **/etc/services** 文件中；而源端口则很可能高于 1024 这个特权端口的传统上限，且不在知名服务的监听端口集合内。将这些数字与你本地网络中可能有限的 IP 地址范围相结合，再加上如今网络的速度（如公告所述），即便是用随机选择源端口和目标端口的伪造数据包流，瞄准一个可能的 IP 地址，也有相当大的机会在可行的时间内干扰某人的通信，甚至可能逃避检测。

现在，请使用你最喜欢的工具将系统升级到未受漏洞影响的版本。及时跟进错误修复，并在补丁发布后尽快应用，这总是明智之举。但事实仍然是（我现在正努力不让自己听起来像傲慢的 OpenBSD 狂热分子）：如果你的 FreeBSD 系统启用了 PF，哪怕 **/etc/pf.conf** 只有四个字节——内容仅为 `pass`，这又非常接近 OpenBSD 现在默认安装中自带的默认 pf.conf——你就能免受任何试图利用该缺失边界检查漏洞的攻击者的侵害。这是因为任何有状态防火墙的基本特性之一——序列号超出预期范围的数据包会被直接丢弃，超出范围的 SYN 永远不会到达任何重要的地方。

漏洞当然应该修复，但我把这个漏洞视为一个论据：在你的网络中、任何潜在攻击者与你的系统之间的信号路径上启用有状态包过滤器，是一件有用的事。

## 有状态的麻烦移除——四字节最低限度

FreeBSD 在内核中内置了 PF，基本系统中也包含相关工具，但 PF 默认未启用。而且上次我检查时，系统也不提供默认的 pf.conf，因此仅用所提供的 rc 脚本启用 PF 会失败，直到你提供一个有效的 pf.conf。

既然你知道只包含一行

```pf
pass
```

的 pf.conf 就够了，何不从这个开始呢？

如果你尝试重新加载这个极简文件中的规则，并对 `pfctl` 使用 `-v`（verbose）选项，你会看到规则实际加载后的样子：

```sh
# pfctl -vf /etc/pf.conf
pass all flags S/SA keep state
```

自 OpenBSD 4.1 和 FreeBSD 7 起，这些就是默认的标志和状态选项。如果愿意，你可以显式设置其他组合，但这套合理的默认值使匹配 pass 规则的新连接以合理的选项创建状态，正如我们已经提到的，即使是这个基本且相当开放的默认设置，也能保护你的系统免受某些恶意的侵扰。事实上，这个四字节的 pf.conf 即使在不充当网关或不提供面向互联网服务的系统上也很有用。

## 走向绝对安全与超越

说到恶意，你可能已经注意到，针对许多服务的机器人密码猜测几乎是日志噪音的持续来源，甚至更糟，而且这些年来丝毫没有好转。如果你在互联网或其他嘈杂网络上运行服务，你可能有兴趣寻找减少必须处理的噪音的方法。几条简单的 PF 规则也能在这方面帮你走很长的路。

但首先，让我们看看五字节的最大安全规则集——

```pf
block
```

它加载后是这个样子

```sh
# pfctl -vf pf.conf
block drop all
```

不需要我帮忙，你大概也能推断出这条规则不会让任何东西通过。没有流量能通过。这是有史以来最彻底的网络门挡。

## 现在扩展以允许一些有用的流量

我从未见过有系统把这个作为完整规则集运行超过几分钟，但它确实是一个非常有用的起点。如果以 block 为默认，你只允许自己知道需要的服务，最终你会得到一个设置，其中你对进出系统的内容有比更宽松的默认值好得多的控制。我见过不少系统正是以这条（block）作为第一条规则，后面跟着其他让特定类型流量通过的规则。这间接揭示了 PF 规则集的结构：最后匹配的规则胜出。在典型的简单配置中，规则集如下：

```pf
clients = "192.168.103/24"
backupserver = "192.0.2.227"
bacula_ports = "9101:9103"
tcp_ports = "{ ftp, ssh, domain, ntp, whois, www, https, auth, nntp, imaps, \
rtsp, submission 8080:8082 }"
udp_ports = "{domain, ntp}"
block
pass inet proto tcp from $clients to port $tcp_ports
pass inet proto udp from $clients to port $udp_ports
pass inet proto tcp from $backupserver to $clients port $bacula_ports
```

逻辑相当清晰。初始的 block 规则默认阻止一切，而接下来的三条规则让来自特定主机的特定流量通过，前提是它瞄准的是指定的端口。好吧，我超前了一点，还引入了另外两个功能：宏（macros）和列表（lists）。宏会就地展开，而像 tcp_ports 和 udp_ports 这样的列表会让解析器为每个列表项生成一条规则，因此实际加载的规则看起来更像是这样：

```sh
# pfctl -vf /etc/pf.conf
clients = "192.168.103/24"
backupserver = "192.0.2.227"
bacula_ports = "9101:9103"
tcp_ports = "{ ftp, ssh, domain, ntp, whois, www, https, auth, nntp, imaps, rtsp, submission 8080:8082 }"
udp_ports = "{domain, ntp}"
block drop all
pass inet proto tcp from 192.168.103.0/24 to any port = ftp flags S/SA keep state
pass inet proto tcp from 192.168.103.0/24 to any port = ssh flags S/SA keep state
pass inet proto tcp from 192.168.103.0/24 to any port = domain flags S/SA keep state
pass inet proto tcp from 192.168.103.0/24 to any port = ntp flags S/SA keep state
pass inet proto tcp from 192.168.103.0/24 to any port = nicname flags S/SA keep state
pass inet proto tcp from 192.168.103.0/24 to any port = http flags S/SA keep state
pass inet proto tcp from 192.168.103.0/24 to any port = https flags S/SA keep state
pass inet proto tcp from 192.168.103.0/24 to any port = auth flags S/SA keep state
pass inet proto tcp from 192.168.103.0/24 to any port = nntp flags S/SA keep state
pass inet proto tcp from 192.168.103.0/24 to any port = imaps flags S/SA keep state
pass inet proto tcp from 192.168.103.0/24 to any port = rtsp flags S/SA keep state
pass inet proto tcp from 192.168.103.0/24 to any port = submission flags S/SA keep state
pass inet proto tcp from 192.168.103.0/24 to any port 8080:8082 flags S/SA keep state
pass inet proto udp from 192.168.103.0/24 to any port = domain keep state
pass inet proto udp from 192.168.103.0/24 to any port = ntp keep state
pass inet proto tcp from 192.0.2.227 to 192.168.103.0/24 port 9101:9103 flags S/SA keep state
```

还不错吧？即使展开了宏和列表，配置仍然相当易读，你可以毫无困难地理解其逻辑。如果需求是这些主机需要与这些服务通信，那么这个配置能给你所需的功能，而 block 默认值能帮你挡住不想要的流量。

## 网络噪音移除，再访

让我们回到把噪音挡在网络之外、至少挡在日志之外的话题。考虑这个（不完整的）配置：

```pf
table <bruteforce> persist
block quick from <bruteforce>
pass inet proto tcp to $int_if:network port $tcp_services \
keep state (max-src-conn 100, max-src-conn-rate 15/5, \
overload <bruteforce> flush global)
```

第一行引入了关键字 `table`。PF 的表（table）是专门用于存储 IP 地址的数据结构。在许多情况下，你可以用宏来指定 IP 地址列表，但这会将你的列表转换成加载规则集中每个 IP 地址一条规则。相比之下，表在规则评估中被视为一个对象，使用 `pfctl` 子命令甚至可以在不重新加载 PF 配置的情况下更改表的内容。表在 pass 和 block 规则中都很有用。如果你编写的 pass 规则引用与网络中特定主机组相对应的表，那么为新主机添加访问权限或为已停用的主机移除访问权限，可能就像在表中添加或删除 IP 地址一样简单（更多信息见文末的参考文献部分）。

这个部分规则集的另一个值得注意的特性是 `keep state` 之后括号 `()` 中的一系列选项。这些是状态跟踪选项。第一个，`max-src-conn`，指定来自单个主机的最大同时连接数。第二个，`max-src-conn-rate`，指定新连接的最大速率，这里是 5 秒内 15 次。第三个，`overload <bruteforce>`，指定任何违反上述任一限制的主机将被添加到该表中，最后，`flush global` 表示被踢入过载表的主机的所有现有连接状态将从状态表中刷新。违反限制的主机的流量将不再匹配 pass 规则，但我们在规则集前面提供了 `block quick`，违规者重试时将被该规则阻止。你可能一直在想，`$int_if:network` 表示宏 `$int_if` 展开后的网络接口名称，以及直接连接到该接口的任何网络。你可能想用这种表示法来指定局域网。

## 根据服务和用户的需求进行区分

有几种方法可以调整状态跟踪选项以适应你的特定需求；一个明显的可能性是为特定类型的流量引入具有不同速率限制的规则。针对 ssh 和其他一些服务的密码猜测近年来有所减缓，现代规则集可能会对 ssh 使用类似这样的规则：

```pf
# 对 ssh 更严格
pass quick proto tcp to port ssh \
keep state (max-src-conn 15, max-src-conn-rate 5/3, \
overload <bruteforce> flush global)
```

大多数人无法在 3 秒内对同一主机进行 5 次登录尝试。这样的组合一定能为你节省大量认证日志中的噪音。同样重要的是要记住，表的内容会随时间增长，上周参与某种不当访问尝试的受感染主机可能已经被清理干净。因此，定期使表内容过期（使用 `pfctl -T expire`，参见手册页）可能很有用。过载表很有用，但尽管如此，这类规则并不完全适合应对 2005 年到 2008 年间开始出现的分布式慢速密码猜测者。

## 展望：你的环境，PF 混音版

FreeBSD 中的 PF（以及原始 OpenBSD 中更进一步的版本——出于各种可以理解的原因，FreeBSD 10 中附带的 PF 落后 OpenBSD 源码超过六年）提供了对许多有用功能的友好接口，以及相当多精细配置系统的可能性。PF 及相关工具能为你带来负载均衡、主机和服务冗余（本质上是集群功能）、流量整形，甚至一些轻量级而有效的反垃圾邮件功能。

本文中的配置示例大致基于我的教程以及最终演变为《The Book of PF》（现已出至第三版）的手稿版本。如需进一步阅读，可以从 FreeBSD Handbook 防火墙章节中的 PF 部分（由我贡献，但由各位文档提交者进一步开发）、《The Book of PF》，当然还有 **pf.conf(5)** 和 **pfctl(8)** 手册页以及这些文档中的任何引用开始。如果你想研究代码本身，它就在你预期的地方。

## 参考文献

- Hansteen, Peter N. M. The Book of PF. No Starch Press.（第 3 版，2014）<http://www.nostarch.com/pf3>
- Hansteen, Peter N. M. The Hail Mary Cloud and the Lessons Learned（博客文章）。（2013）<http://bsdly.blogspot.com/2013/10/the-hail-mary-cloud-and-lessons-learned.html>
- FreeBSD Handbook, PF chapter. (1995–2014) <https://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/firewalls-pf.html>
- The PF man pages **pfctl(8)** **pf.conf(5)**
