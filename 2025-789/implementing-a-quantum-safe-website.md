# 实现量子安全网站

- 原文：[implementing-a-quantum-safe-website](https://freebsdfoundation.org/our-work/journal/browser-based-edition/embedded-2/implementing-a-quantum-safe-website/)
- 作者：Gergely Poór

不久前，在我目前的工作单位里，有人提醒我：传统密码学将不再被视为安全。

据多方消息预计，在未来 10 年内，量子计算的进步将达到这样一个水平：我们习以为常的现代密码算法将会在几秒钟内被轻易攻破。很自然地，我开始研究这个话题，想弄清楚情况到底有多糟，以及我们今天能做些什么来为这种情形做准备。所谓的“量子威胁”通常被描述为日后某个时点，量子计算机能拥有足够的处理能力，可以在几分钟甚至几秒钟内破解传统加密。听起来很糟，对吧？更糟的是，还有一个相关现象叫“现在存储——以后解密”（Store now – decrypt later）或“现在收集——以后解密”（Harvest now – decrypt later），它的基本意思是：当前在互联网上传输的“安全”数据会被拦截并保存，等到足够强大的量子计算机广泛可用时，再用它来解密此前捕获的数据。

但什么是“当前的安全”呢？就非对称加密而言，存在密钥交换，其中使用最广的是 RSA，密钥长度为 2048、3072 或 4096 位。这些密钥交换（不仅限于 RSA）可以出现在到远程服务器的 SSH 会话、用于输入你银行账号信息的 TLS 加密网站，甚至两个远程位置之间基于 IKEv2 的 IPSec VPN 隧道中。通过这些通道发送的所有数据都可能被拦截并保存，等以后再解密。可问题在于，如何解密 RSA 密钥？答案是秀尔（Shor）算法，它由 Peter Shor 在 20 世纪 90 年代提出，旨在用量子计算机对大整数进行因数分解。但这与密钥交换算法有什么关系？以 RSA 为例。简单说，它通过将两个大素数相乘来得到一个更大的数。两枚素数被保密，并与其他若干数一起构成私钥的一部分。它们的乘积与一些额外数值一起构成公钥的一部分。使其脆弱的地方在于：若设法找到了这两个起始素数，你就可以用它们计算出私钥的一部分，进而求得私钥的其余部分。有了私钥，你就能解密一方发送的内容；再解一次，就能完全解开这段已捕获的信息交换。当然，这需要巨大的计算能力，因为公钥越大，需要的计算就越多。秀尔算法提供了一种方法，利用量子计算机一次性并行计算所有可能的解，从而加速这一过程。

我们该如何抵御这类攻击？有三种答案：

你可以继续使用标准算法，但采用更长的密钥。目前认为，超出 4096 位的 RSA 密钥在短期内仍难以被破解。如果想更保险，可以使用 8192 位或更长的密钥。

不过，还有其他替代方法来解决这个问题。第二种是 PQC（后量子密码学，Post-Quantum Cryptography）。研究人员和数学家开始研制能对抗秀尔算法的其他算法。这些算法基于所谓的数学“陷门”函数：从方程到结果很容易，但反过来（几乎）不可能。美国国家标准与技术研究院（NIST）开始收集这些算法，众多密码学专家与数学家对它们进行测试，以判断其是否能抵抗秀尔算法。这就是“后量子密码学项目”。多年来经历了数轮遴选与淘汰。2024 年，NIST 发布了 FIPS 203 标准，指定 ML-KEM（此前称为 CRYSTALS-Kyber 或 Kyber）为通用加密的主要标准；并在 2025 年表示，若 ML-KEM 出现问题，将以 HQC 作为后备。

第三种选择是 QKD（量子密钥分发，Quantum Key Distribution），它基于对光子粒子的量子物理操控来安全地产生密钥。该方法极其昂贵，需要专用设备，以及在通信双方之间预先存在的光纤网络连接，目前两端点之间的距离上限约为 100 km。

## 项目目标

我有一个之前做的小网站，仅使用 nginx，没有什么 HTML、CSS、PHP 等。它是个简单的网站，会返回客户端的 IP 地址（类似 icanhazip.com 或 ifconfig.me）。它最初是我学习 nginx 时的一个兴趣项目，但后来我在更大的网络里部署它，用于测试客户端是否在 NAT 之后。这正好是我进行 PQC 实验的良好起点。需求很简单：既要能适配广泛的软件（网页浏览器、fetch(1) 或 curl(1) 等命令行工具），又要遵循我在使用 FreeBSD 与 Linux 多年中体会到的 UNIX 哲学——尽可能简单，同时要稳定，因为它以后也可能跑在云端 VPS 上。因此，我自然选择 FreeBSD 作为操作系统、nginx 作为平台。剩下的就是 PQC 的实际实现。

我做了一些研究，找到了 Open Quantum Safe（OQS）项目的子项目——[oqs-provider](https://github.com/open-quantum-safe/oqs-provider)。它是一款适用于 OpenSSL 3 版本的开源 C 库与 provider，实现了包括 ML-KEM 在内的多种算法。它在 FreeBSD 与多种 Linux 发行版上均可用。

## 工作原理

简单讲，它把各种用于密钥交换与签名的 PQC 算法集成到了 OpenSSL 中。就密钥交换（及其他）而言，它支持 ML-KEM 与多种基于椭圆曲线的 Diffie-Hellman 密钥交换，如 X25519、p384、p521、SecP384r1。它们可以从名称上直观识别，比如 X25519MLKEM768，表示使用 Curve 25519 搭配 768 位的 ML-KEM 密钥。PQC 算法需要 TLSv1.3，但出于兼容性考虑，我们会将最低版本设置为 TLSv1.2，这样旧系统仍能访问该网站。要注意的是，可能会出现“[TL;DR fail](https://tldr.fail/)”错误：如果客户端软件没有正确配置以在 TLSv1.3 上支持 PQC 算法，就会导致 TLS 失败；不过，随着软件在客户端上更新与推广，这个问题会逐渐缓解。如果你不在乎旧客户端或有缺陷的软件，并且想要 100% 量子安全的网站，可以完全禁用 TLSv1.2（稍后你会在 nginx 配置里看到）。理论部分说完，进入有趣的部分：实际实现！

## 实现

oqsprovider 需要 OpenSSL 3.2 或更高版本。根据其 GitHub 页面说明，从 3.4 版本开始又新增了一些功能。我的 FreeBSD 安装里是 OpenSSL 3.0.16，不支持 oqsprovider。

在撰写本文时，OpenSSL 3.5.0 已发布，带有原生的 PQC 算法支持，但 pkg(8) 的说明指出它处于 beta 阶段，不适合生产环境。因此，在余下实现中，我将继续使用 OpenSSL 3.4.1。

首先，我把虚拟机更新到 FreeBSD 14.3-RELEASE。我们还需要安装更新版本的 OpenSSL 以及 nginx，并且要让 nginx 能使用较新的 OpenSSL。为了尽量减少折腾，我们将通过 pkg(8) 安装 openssl34 与 openssl-oqsprovider，而 nginx 则通过 ports 系统构建。为此需要在 /usr/ports 下准备好 ports。我的系统里没有 security/openssl34，因此我将拉取 ports 树的 2025Q2 分支。我需要它来让 nginx 链接到 openssl34。首先，我会安装 git(1)，这是 FreeBSD 手册推荐的安装 / 更新 ports 树的方法。


```sh
# pkg install -y git
```

待系统上安装了 **git(1)**，就可以用它来管理 ports 树了。不过，由于我之前通过基本系统安装过 ports 树，所以现在会先删除现有的 `/usr/ports` 目录。这样在克隆仓库时，就不会提示 `/usr/ports` 已存在。当然，还有其他办法解决这个问题，但我更喜欢从一个干净的环境开始。

```sh
# rm -rf /usr/ports
# git clone --depth 1 https://git.FreeBSD.org/ports.git -b 2025Q2 /usr/ports
```

之后，我们需要安装 OpenSSL 3.4.1 和 oqsprovider。

```sh
# pkg install -y openssl34 openssl-oqsprovider
```

然后，正如安装信息所提示的那样，我们需要将 `/usr/local/openssl/oqsprovider.cnf` 的内容合并到 `/usr/local/openssl/openssl.cnf` 中。由于我们刚刚安装了新版 OpenSSL，合并之后，`/usr/local/openssl/openssl.cnf` 的内容将如下所示：

```ini
…
[provider_sect]
default = default_sect
oqsprovider = oqsprovider_sect
….
[default_sect]
activate = 1

[oqsprovider_sect]
activate = 1
module = /usr/local/lib/ossl-modules/oqsprovider.so
…
```

现在我们要编译 nginx。

```sh
# cd /usr/ports/www/nginx
```

我将设置一些环境变量，以便让 nginx 链接到新安装的 OpenSSL 3.4.1。

```
# export OPENSSL_BASE=/usr/local
# export OPENSSL_LIBS="-L/usr/local/lib"
# export OPENSSL_CFLAGS="-I/usr/local/include"
```

接下来我们将配置 nginx，确保启用了 **HTTP_SSL**（它应该是默认开启的，但最好还是确认一下）。我不会调整其他任何设置。

```sh
# make config
```

现在我们已经准备好开始编译 nginx 了。设置一些环境变量来指定新安装的 OpenSSL 的路径，然后按下回车即可。

```sh
# make OPENSSLBASE=/usr/local OPENSSLDIR=/usr/local/openssl install clean
```

在 nginx 编译的过程中，可以去拿一杯你喜欢的饮料，处理一些工单，或者把编译输出展示给朋友们，让他们看看你有多酷。

编译完成后，我们来验证 nginx 是否已经链接到了安装在 `/usr/local` 下的 OpenSSL 3.4.1：

```sh
# nginx -V 2>&1 | grep -i openssl
built with OpenSSL 3.4.1 11 Feb 2025
# ldd /usr/local/sbin/nginx | grep ssl
    libssl.so.16 => /usr/local/lib/libssl.so.16 (0x16a83b849000)
```

注意：括号中的十六进制标识符可能会有所不同。

如果你的输出和我的一样，那么你就已经成功为 nginx 添加了 OpenSSL 3.4 的支持。接下来，我们将为网站创建一个包含 PQC 的配置文件。让我们进入 `/usr/local/etc/nginx` 目录，首先备份原始的 **nginx.conf** 文件：

```sh
# cd /usr/local/etc/nginx
# mv nginx.conf nginx.conf.orig
```

现在我们来创建新的配置：

```sh
# vi nginx.conf
```

在文件中添加以下若干行：

```sh
events{}
http{
     server{
         listen 443 ssl;
         ssl_certificate /usr/local/etc/nginx/server.crt;
         ssl_certificate_key /usr/local/etc/nginx/private.key;
         ssl_protocols TLSv1.2 TLSv1.3; # 如果不需要向后兼容，可以移除 TLSv1.2
compatibility
         ssl_prefer_server_ciphers off;
         ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305; #if using pure TLSv1.3 you can remove this line according to Mozilla’s SSL Configuration Generator’s “modern” settings
         ssl_ecdh_curve X25519MLKEM768:X25519:prime256v1:secp384r1; #这是 PQC 部分，其余内容仅用于非 PQC 的兼容性。
         location /{
            return 200 "$remote_addr\n";
         }
     }
     server{
         listen 80;
         location /{
            return 301 https://$host$request_uri;
         }
     }
}
```

此配置实现了以下功能：

- 监听 80 和 443 端口
- 将明文 HTTP 请求重定向到 HTTPS
- 使用 TLS 1.3 和 TLS 1.2 以保证兼容性
- 使用平衡的加密套件，在提供较好安全性的同时兼顾广泛的兼容性（加密套件和曲线列表来源于 Mozilla SSL Configuration Generator）

接下来，在本演示中我将创建一张自签名证书，但在生产环境中（以及为了兼容性），你应当获取由可信 CA 签发的有效证书。你可以使用 Let’s Encrypt 证书，并通过 **certbot(1)** 来自动化证书续期。为此，只需运行以下命令：

```sh
# pkg install -y py311-certbot
```

之后，按照 Certbot 的使用说明进行操作。

如果要创建自签名证书，可以使用下面这行命令（记得将 DNS 和 IP 字段改为与你的环境相匹配）：

```sh
# openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout private.key -out server.crt \
  -subj "/CN=quantum" \
  -addext "subjectAltName=DNS:quantum,IP:192.168.2.40"
```

它将使用 2048 位的 RSA 密钥，但你也可以将其提升到 8192 位（记住，目前认为超过 4K 的密钥在量子计算面前仍是安全的）。不过要注意，这可能会导致某些旧系统的兼容性问题。证书的有效期为 1 年，这对测试目的来说已经绰绰有余。（此时我想起那句话：“没有什么比临时解决方案更永久的了。”）同时，证书中包含了我 FreeBSD 虚拟机的 IP 地址和主机名。如果没有这些，一些软件（例如 PowerShell）会抱怨无法信任证书，从而无法与网站建立或完成 TLS 会话。

当证书和私钥都准备好之后，你就可以启动 nginx 了。但在此之前，我们需要在 `/etc/rc.conf` 中指定它应当在开机时自动启动。


```sh
# sysrc nginx_enable=YES
# service nginx start
```

它会在启动之前先验证 **nginx.conf** 的语法，并对配置进行一次简单测试。如果一切正常，你应该会看到如下输出：

```sh
Performing sanity check on nginx configuration:
nginx: the configuration file /usr/local/etc/nginx/nginx.conf syntax is ok
nginx: configuration file /usr/local/etc/nginx/nginx.conf test is successful
Starting nginx.
```

## 测试

现在我们已经有了一个正在运行的网站，接下来让我们检查它是否正常工作。在测试时，我将使用多种方法：网页浏览器以及一些命令行客户端，例如 **curl(1)**。我已经安装好了 curl，如果你还没有，可以通过 **pkg** 来安装：


```sh
# pkg install -y curl
```

你可以使用 **curl** 来检查网站（注意由于使用的是自签名证书，因此需要加上 `--insecure` 参数）：

```sh
# curl --insecure https://127.0.0.1
```

它会返回 **127.0.0.1** 以及一个换行符。如果想获取更多信息，可以加上 `-v` 参数，让输出更详细。


```sh
# curl --insecure -v https://127.0.0.1
```

在我的环境中，由于 curl 链接的是系统默认的 OpenSSL 版本，它并不支持 PQC 算法，因此会回退到 **X25519**：


```sh
…
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519 / RSASSA-PSS
…
```

你也可以使用以下命令来验证重定向：

```sh
# curl --insecure -vL http://127.0.0.1
```

如果你看到如下输出，就说明它成功了：

```sh
…
* Request completely sent off
< HTTP/1.1 301 Moved Permanently
….
* Clear auth, redirects to port from 80 to 443
…
```

对于基于 Microsoft Windows 的主机，如果你使用的是自签名证书，就需要将该证书（在我们的示例中是 **server.crt**）导入到 **“受信任的根证书颁发机构”** 存储区中，然后运行以下 PowerShell 命令：

```PowerShell
(Invoke-WebRequest https://192.168.2.40).Content
```

或者，如果你想要更详细的输出，可以使用：


```PowerShell
Invoke-WebRequest https://192.168.2.40
```

使用 curl 也可以，不过在 Windows 上它只是 **Invoke-WebRequest** 的前端，不具备 FreeBSD 或 Linux 版本中的那些参数。

为了测试与其他硬件的兼容性，我登录到了我的 **MikroTik 路由器**，并在命令行中调用了该 URL：


```sh
/tool fetch url="https://192.168.2.40" output=user check-certificate=no
    status: finished
  downloaded: 0KiB
     total: 0KiB
    duration: 1s
     data: 192.168.2.1n
```

注意末尾的那个 **"n"**。如果你想去掉它，可以在 `/usr/local/etc/nginx/nginx.conf` 中修改以下这一行：

```sh
return 200 "$remote_addr\n";
```

改为：

```sh
return 200 "$remote_addr";
```

这样一来，如果使用 curl 之类的工具调用，它就不会再返回换行符了。你可以根据自己的需求决定保留哪种版本。如果你打算在脚本中使用这个 IP 地址，最好去掉 `\n`；而对我来说，目前只是调试用途，所以我会保持原样。

对于网页浏览器，如果你希望启用 PQC 支持，在某些版本中可能需要手动开启相关功能。我正在使用 **Firefox 139.0.4**，从 **132 版本**起就已经默认启用了 PQC 支持。而 **Chrome** 从 **124 版本**开始也支持 PQC，但在某些情况下需要你手动启用：

```sh
chrome://flags/#enable-tls13-kyber
```

你可以通过进入 **about:config** 来检查 Firefox 是否支持该功能，并查找以下配置项：

```sh
security.tls.enable_kyber
```

接下来，按 **F12** 打开开发者工具：

- 在 **Firefox** 中切换到 **“network”** 选项卡
- 在 **Chrome** 中进入 **“隐私和安全”**

然后输入你的 FreeBSD 安装的 IP 地址并按回车。屏幕上应该只显示一个 IP 地址（即请求的来源）。在开发者工具中，点击对应你 FreeBSD IP 地址的请求条目：

- 如果使用 Firefox，还需要点击右侧的 **“Security”** 标签页。

如果你的浏览器支持 PQC，你会看到密钥交换方式是：

- **mlkem768x25519**（Firefox）
- **X25519MLKEM768**（Chrome）



![](https://freebsdfoundation.org/wp-content/uploads/2025/10/poor_chart1.png)

如果看到了上述结果，那么恭喜你！🎉 你已经成功在 FreeBSD 上部署了一家带有 PQC 支持、同时兼容传统加密的安全网站。欢迎来到未来！


## 总结

为 TLS 密钥交换添加 PQC 支持并不算太难，但整体流程包含了一些额外却必要的步骤。一个关键问题是你必须从源码编译 nginx 才能使用新版 OpenSSL。这意味着每次有安全补丁发布时，你都需要重新编译，而不像使用 **freebsd-update(8)** 或 **pkg(8)** 热修复那样方便。

不过，**OpenSSL 3.5.0** 已经发布，它原生支持多种 PQC 算法，并且是长期支持（LTS）版本，官网称将支持至 2030 年。随着量子威胁的加剧以及全行业急于尽快部署 PQC，我非常希望这一版本能被集成到 FreeBSD 基本系统中。这将省去绝大部分让 nginx（以及可能的其他软件）支持 PQC 的额外步骤。

当然，FreeBSD 对稳定性的要求很高，在 OpenSSL 3.5 被充分测试和验证之前，基础安装中很可能仍会保留较旧的版本。本文展示的网站只是一个量子安全加密的小例子，但可以想象更多软件能从 PQC 中获益。比如银行、医疗或政府网站若部署 ML-KEM，将几乎不可能被未来的量子计算机解密，银行账户信息、患者资料以及个人身份信息在传输过程中将得到保护。

虽然这种普及不会马上发生，但随着越来越多人接触到“量子威胁”这一概念，意识也会逐渐提高，我们也会更接近一个后量子密码学成为日常生活一部分的世界。

---

**Gergely Poór** 是一名 Linux/BSD 系统与网络工程师、电工，同时也是一位 FreeBSD 爱好者，自 2018 年高中毕业起一直从事 IT 工作。他的经验涵盖了从中小企业桌面支持到企业级混合云以及工业/物联网系统。他始终热衷于学习新知识。现居住在匈牙利布达佩斯，与深爱的妻子 Kriszti 一起生活，业余时间喜欢使用 sh/bash 编程，并开发自己的智能家居系统。
