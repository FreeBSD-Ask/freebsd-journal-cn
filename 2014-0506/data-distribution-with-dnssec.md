# 用 DNSSEC 分发数据

- 原文：[Data Distribution With DNSSEC](https://freebsdfoundation.org/our-work/journal/browser-based-edition/networking/data-distribution-with-dnssec/)
- 作者：**Michael W. Lucas**

在公共网络中，信任身份是棘手问题。你如何确认一台服务器确实如它所声称的那样？在组织内部你可以有多种解决方案，但在公共互联网上，要么信任各种外部实体来验证身份（证书授权机构），要么自己手工搭建一套身份识别网络（OpenPGP 的信任网络）。DNS 安全扩展（DNS Security Extensions，DNSSEC）正在改变这一现状。

## 什么是 DNSSEC？

域名服务（Domain Name Service，DNS）将主机名映射到 IP 地址，反之亦然。这就是你能在浏览器里输入 <http://www.FreeBSD.org> 而不是 <http://8.8.178.110> 的原因。但 DNS 诞生于上世纪 80 年代初，原本面向的是规模小得多、用户群体也小得多的网络。那时人们只要有协议能让系统管理员输入主机名就拿到 IP 便心满意足，不必担心地址伪造、恶意软件以及以牟利为目的的入侵者。如今互联网上充斥着犯罪分子和恶意软件，我们需要一种机制来验证 DNS 信息的准确性。

DNS 安全扩展为 DNS 信息加入了密码学校验。DNS 查询的每一级都带有数字签名，本地客户端可以校验这些签名。DNS 是公开数据，因此 DNSSEC 不提供机密性保证。但 DNSSEC 让任何改动都显而易见——对任何愿意查看的人而言，DNS 都是防篡改的。

从某些角度看，DNSSEC 把信任网络与证书授权机构结合在了一起。根区有数字签名。客户端必须信任这个签名（是的，万一密钥泄露，会有协议来替换该签名）。根区为其下级区的记录签发数字签名，而这些区又为它们下级区的记录签发数字签名。因此当你用 DNSSEC 查询 `www.FreeBSD.org` 的 IP 地址时，客户端可以拿到 `www.freebsd.org` 主机的数字签名，然后利用 `.org` 内的签名来校验该域的签名，最后利用根区的签名来校验 `.org`。客户端借助根区上的可信密钥建立起一条信任链，从而信任目标站点的身份。

如果你必须信任某个外部实体，DNSSEC 与证书授权机构又有何不同？去看看你的浏览器吧。你会看到几十家证书授权机构。这些机构中，有些比另一些更为谨慎、更有信誉。许多机构把证书发给了并不具备资格的组织——举例来说，不止一家 CA 把谷歌、Apple 和 Microsoft 的证书发给了不该拿到它们的组织或个人。组织机构，即便是谨慎的 CA，也会失守。而使用 DNSSEC，你不必依赖第三方来认证身份。

此外，证书授权机构为身份认证收取费用。这笔费用或许不多，但当你的服务器多达几十乃至几百台时，开销就相当可观了。许多组织宁可不在传输中保护数据，也不愿为每一条小小的连接掏钱。

如今 DNS 的用途远不止主机和 IP 地址信息。DNS 向从服务器到手机在内的各种设备提供配置信息，列出域的合法邮件发送者，等等。一旦这个信道具备防篡改能力，你就可以把公开的安全信息放进 DNS。下面将考察两类常通过 DNS 分发的数据：SSH 公钥指纹和 SSL 证书指纹。

## SSH 主机密钥指纹

SSH 使用公钥密码学来确认你连接的服务器确实是你想要的那台，并保护客户端与服务器之间的数据交换。正确使用 SSH 的前提是：用户首次连接到一台服务器时，必须检查服务器提供的公钥，并将其与一份准确的公钥副本比对。这件事既繁琐又恼人，多数 SSH 用户根本懒得做。更糟的是，多数 SSH 用户很快就会决定忽略 SSH 的公钥警告，从而削弱了该协议的安全性。

比对两个密码学指纹正是计算机擅长的事，但直到现在也没有标准化的方式来在线安全传输这些指纹。DNSSEC 通过 SSH 指纹（SSHFP）记录提供了这样的信道。较新版本的 FreeBSD 自带的 OpenSSH 客户端会自动检查 SSHFP 记录。

首先为你的主机创建 SSHFP 记录，并将其插入该主机的区记录中。

运行 `ssh-keygen -r` 从 **/etc/ssh** 下的公钥文件生成 SSHFP 记录。把你希望出现在 SSHFP 记录中的主机名作为参数传入。下面我为我的 Web 服务器 `www.michaelwlucas.com` 创建 SSHFP 记录。

```sh
$ ssh-keygen -r www
www IN SSHFP 1 1 f44d08efc159…
www IN SSHFP 1 2 86c744ce05ba…
www IN SSHFP 2 1 9af675d68969…
www IN SSHFP 2 2 7914036e9053e14db552…
www IN SSHFP 3 1 0f7c928e3954c54f0b32…
www IN SSHFP 3 2 5c5192e78de10…
```

这会显示本机所有主机密钥的 SSHFP 记录。这些记录是本机专用的——我在 `www.FreeBSD.org` 上运行完全相同的命令，会得到指纹完全不同的记录。

如果一台主机有多个名字，而你可能会用其中任何一个名字去连接这台主机，那么每个主机名都需要 SSHFP 记录。举例来说，我的 Web 服务器又叫 `pestilence.michaelwlucas.com`。我需要在区里准备两份这样的记录，每个主机名一份，类似下面这样：

```sh
www IN A 192.0.2.33
www IN SSHFP 1 1 f44d08efc159…
www IN SSHFP 1 2 86c744ce05ba…
www IN SSHFP 2 1 9af675d68969…
www IN SSHFP 2 2 7914036e9053e14db552…
www IN SSHFP 3 1 0f7c928e3954c54f0b32…
www IN SSHFP 3 2 5c5192e78de10…
pestilence IN A 192.0.2.33
pestilence IN SSHFP 1 1 f44d08efc159…
pestilence IN SSHFP 1 2 86c744ce05ba…
```

每个主机名变体的哈希值相同。

要让 OpenSSH 客户端检查 SSHFP 记录，请在 **ssh_config** 或 **~/.ssh/config** 中将 `VerifyHostKeyDNS` 设为 `yes`。**ssh(1)** 会使用 SSHFP 记录来校验主机密钥，而不再提示用户。计算机擅长比较，你不擅长。把活交给它们。

## DANE 与 TLSA

DANE，即基于 DNS 的命名实体认证（DNS-based Authentication of Named Entities），是一种把公钥或公钥签名塞进 DNS 的协议。由于标准 DNS 很容易被伪造，没有 DNSSEC 就无法安全地做这件事。而有了 DNSSEC，你就多了一条验证公钥的途径。我们将用 DNSSEC 保护的 DNS 来验证网站 SSL 证书（有时称为 DNSSEC 装订的 SSL 证书）。

在《DNSSEC Mastery》一书中我曾预言，会有人发布一款浏览器插件来支持 DNSSEC 装订 SSL 证书的校验。这预言并不算太高明，因为当时已经有几个团队在做了。我很高兴地报告，<http://dnssec-validator.cz> 的朋友们已经完成了 TLSA 校验插件。我在 Firefox、Chrome 和 IE 中都在用它。终有一天浏览器会原生支持 DANE，但在那一天到来之前，我们需要插件。

DNS 通过 TLSA 记录提供 SSL 证书指纹。（TLSA 并不是首字母缩略词；它只是一条 TLS 记录，类型 A。估计将来我们还会演进到 TLSB。）一条 TLSA 记录长这样：

```sh
_port._protocol.hostname TLSA ( 3 0 1 hash...)
```

如果你用过 VOIP 之类的服务，这看上去应该相当眼熟。例如，主机 `dnssec.michaelwlucas.com` 上端口 443 的 TLSA 记录长这样：

```sh
_443._tcp.dnssec TLSA ( 3 0 1
4CB0F4E1136D86A6813EA4164F19D294005EBFC02F10CC400F1776C45A97F16C)
```

哈希从哪里来？对你的证书文件运行 **openssl(1)** 即可。下面我对证书文件 **dnssec.mwl.com.crt** 生成 SHA256 哈希：

```sh
# openssl x509 -noout -fingerprint -sha256 < dnssec.mwl.com.crt
SHA256 Fingerprint=4C:B0:F4:E1:13:6D:86:A6:81:3E:A4:16:4F:19:D2:94:00:5E:BF:C0:2F:10
:CC:40:0F:17:76:C4:5A:97:F1:6C
```

把指纹复制进 TLSA 记录，去掉冒号。就这么简单。

有趣的是，你也可以用 TLSA 记录校验 CA 签名的证书。用相同方式生成哈希，但把开头的字符串改成 `1 0 1`。我为 <https://www.michaelwlucas.com> 用的是一张 CA 签名的证书，同时通过 DNSSEC 用下面这条记录进行校验。

```sh
_443._tcp.www TLSA ( 1 0 1
DBB17D0DE507BB4DE09180C6FE12BBEE20B96F2EF764D8A3E28EED45EBCCD6BA )
```

那么：如果你花功夫配好了这些，客户端会看到什么？

首先在浏览器中安装 DNSSEC/TLSA Validator（<https://www.dnssec-validator.cz/>）插件。希望在本文章付梓之时，它已成为正式的 FreeBSD Port。在此期间，Peter Wemm 已经在 FreeBSD 上构建了该插件的 Firefox 版本，并提供了补丁（<http://people.freebsd.org/~peter/MF-dnssec-tlsa_validator-2.1.1-freebsd-x64.diff.txt>）和 64 位二进制文件（<http://people.freebsd.org/~peter/MF-dnssec-tlsa_validator-2.1.1-freebsd-x64.xpi>）。如果你想为 FreeBSD 出一份力，把它做成 Port 会很有意义。

该插件新增了两个状态图标。如果站点的 DNS 使用了 DNSSEC，第一个图标会变绿；如果未使用，则显示略带红色的灰色小图标。没有 DNSSEC 并不值得大惊小怪。第二个图标在 SSL 证书与 TLSA 记录匹配时变绿，没有 TLSA 记录时变灰，证书与 TLSA 记录不匹配时变红。

你需要为那张自签名证书担心吗？查一下 TLSA 记录状态。如果域主说“是的，我创建了这张证书”，那大概没问题。如果一张自签名证书未能通过 TLSA 校验——它毕竟是张自签名证书：用在邮件列表归档上大概没事，用在你的银行上就不行了。

TLSA 支持多种哈希，也可以设置各种条件。公司里所有证书都应该用 RapidSSL 签发？你可以在 TLSA 记录里指定。你有私有 CA？把它的指纹写进 TLSA 记录。如果想动手折腾一番，可参阅 RFC 6698 或我的《DNSSEC Mastery》一书。

我的笔记本在挂起之后用这个插件遇到了一些问题。家里和办公室都做了 DNSSEC 校验。当我去咖啡店并恢复笔记本时，插件会引起性能问题。重启浏览器就能解决。我估计这会很快改善，你读到本文时这也许已经不成问题了。

DNSSEC 给了你一条替代的信任途径，跳出了传统而昂贵的 CA 模式。把 TLSA 推广得更广，意味着你可以用 SSL 保护更多服务，且无需额外开支。

当然，没有可用的 DNSSEC 就无法部署上述任何一种服务。如今 DNSSEC 并不难——我能搞定，你也能。

---

Michael W. Lucas 是《Absolute FreeBSD》《Absolute OpenBSD》和《DNSSEC Mastery》等书的作者。他与妻子和一大群老鼠住在密歇根州底特律。访问他的网站：<https://www.michaelwlucas.com>。
