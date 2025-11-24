# 现在用 Webhook 触发我

- 原文链接：[Kick Me Now with Webhooks](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-14-0/kick-me-now-with-webhooks/)
- 作者：Dave Cottlehuber

## 什么是 Webhook，为什么我需要它？

Webhook 是一种基于 HTTP 事件驱动的远程回调协议，几乎可通过所有编程语言和工具轻松调用脚本和任务。Webhook 的优点在于其普遍应用和简单性。仅一个简单的 HTTP 链接，你就可以请求远程服务器执行任务，如调暗灯光、部署代码和代表你运行任意命令。

最简单的 Webhook 可能只是智能手机浏览器中的书签链接；更为复杂的版本，则可能需要强认证和授权。

尽管有像 Ansible 和 Puppet 这样较大型的自动化工具集，但有时，简单的方案足以满足需求。Webhook 就是这种方案，它能让你在远程计算机上安全地执行任务，仅需发出请求即可。调用 Webhook 即是“触发”操作，因此本篇文章的标题亦如此。

## 集成

目前尚无官方标准，但通常情况下，Webhook 是通过 POST 请求发送，并使用 JSON 对象作为消息体，通常会启用 TLS 加密，并通过签名确保防止篡改、网络伪造和重放攻击。

常见的集成，有聊天服务如 Mattermost、Slack 和 IRC；软件仓库如 Github 和 Gitlab；通用托管服务如 Zapier 或 IFTT；以及许多家居自动化系统如 Home Assistant 等。几乎在所有地方，Webhook 都能收发，因此 Webhook 的应用范围几乎是无限的。

虽然你可以在一个小时内就编写一个最简单的 Webhook 客户端或服务器，但如今，几乎每种编程语言中都有不少现成的选择。聊天软件通常提供了内置的 Webhook 触发器，用户可以通过类似 `/command` 的语法来调用。IRC 服务器也未被遗忘，一般由守护进程和插件实现。

Webhook 另一个不太明显的优势是它能够明确划分安全性和权限。一个低权限用户可以调用远程系统上的 Webhook。可以以低权限运行远程 Webhook 服务，先进行验证和基本语法检查。然后，在验证通过后，再调用高权限任务。也许最终的任务有权限访问某个特权令牌，来重启服务、部署新代码，或者让孩子们再享受一个小时的电子娱乐时间。

像 GitHub、GitLab 和自托管选项等常见的软件仓库也提供这类功能，触发时可以包括分支名、提交记录以及做出更改的用户。

这使得构建可以更新网站、重启系统和根据需要触发更复杂工具链的工具变得相对简单。

## 架构

典型的 Webhook 架构由一台监听传入请求的服务器和一部提交请求的客户端组成，客户端可能还会带上一些参数，包括认证和授权信息。

## 服务器端

首先，我们来讨论服务器端。服务器端一般会是一个守护进程，来监听 HTTP 请求，并根据特定条件处理请求。如果请求不符合这些条件，服务器会拒绝该请求，并返回适当的 HTTP 状态码。如果请求成功提交，服务器可以从批准的请求中提取参数，然后根据需要执行自定义操作。

## 客户端

由于服务器使用 HTTP，几乎所有客户端都能用来发送请求。[cURL](https://curl.se/) 是一种非常普遍的选择，但我们会使用一个更加友好的工具——[gurl](https://github.com/skunkwerks/gurl)，它内置了对 HMAC 签名的支持。

## 消息

消息通常是个 JSON 对象。对于那些关注重放/时间攻击的用户，你应该在消息体中包含时间戳，并在进一步处理前验证该时间戳。如果你的 Webhook 工具包能够对特定的头部进行签名和验证，那也是一个可选方案，但大多数工具包不支持该功能。

## 安全性

可以使用共享的密钥对 HTTP 请求的主体进行签名，生成的签名作为消息头部提供。这既提供了身份验证的手段，又证明了请求在传输过程中未被篡改。它依赖于共享密钥，使两端可以独立验证消息签名，通过附加的 HTTP 头部和消信息体来完成验证。

最常见的签名方法是 HMAC-SHA256。这是两种加密算法的组合——我们熟悉的 SHA256 哈希算法可以对较大信息进行安全摘要，在这里是指 HTTP 的主体，另外 HMAC 方法使用一个密钥与信息结合生成一个唯一的代码，也即数字签名。

这两种功能结合起来，用于检测信息是否被篡改。它就像是对内容的数字印章，确认信息必定是由知道共享密钥的一方发送的。

请注意，使用 TLS 加密和签名能够提供信息的机密性和完整性，但不能保证可用性。精心策划的攻击者可能会中断或淹没网络，从而导致信息丢失而没有任何通知。

一般做法是，在 Webhook 的主体中包含时间戳，且由于 HMAC 签名的保护，可以有效抵御时间攻击和重放攻击。

请注意，未经时间戳的主体总是会有相同的签名。这在某些情况下是有用的。例如，可以预先计算 HMAC 签名，并使用一个不变的 HTTP 请求来触发远程操作，而无需在发起 Webhook 请求的系统上公开 HMAC 密钥。

## 整合实现

我们将安装一些实用工具，包括 [Webhook 服务器](https://github.com/adnanh/webhook)、常用工具 [curl](https://github.com/skunkwerks/gurl)，以及 gurl——使 Webhook 签名变得轻松的工具。

```sh
$ sudo pkg install -r FreeBSD www/webhook ftp/curl www/gurl
```

让我们启动服务器，运行个简单的例子，将其保存为 `webhooks.yaml`。

它将使用命令 `logger(1)`，在 `/var/log/messages` 中写入一个短条目，记录成功调用 Webhook 的 HTTP User-Agent 头。

注意，这里有一个 `trigger-rule` 键，请确保 HTTP 查询参数 `secret` 的值与字符串 `squirrel` 匹配。

目前我们没有 TLS 安全性，也没有 HMAC 签名，因此系统的安全性还不高。

```yaml
---
- id: logger
  execute-command: /usr/bin/logger
  pass-arguments-to-command:
  - source: string
    name: '-t'
  - source: string
    name: 'webhook'
  - source: string
    name: 'invoked with HTTP User Agent:'
  - source: header
    name: 'user-agent'
  response-message: |
    webhook executed
  trigger-rule-mismatch-http-response-code: 400
&nbs;&nbs;trigger-rule:
    match:
      type: value
      value: squirrel
    &nbs; parameter:
       source: url
       name: secret
```

然后在终端运行 `webhook -debug -hotreload -hooks webhook.yaml`。上述参数浅显易懂。

在其他终端里，运行 `tail -qF /var/log/messages | grep webhook`，这样我们就可以实时查看结果。

最后，我们使用 `curl` 来触发 Webhook，首先不带查询参数，然后再带上查询参数：

```sh
$ curl -4v ‘http://localhost:9000/hooks/logger’
* Trying 127.0.0.1:9000…
* Connected to localhost (127.0.0.1) port 9000
› GET /hooks/logger HTTP/1.1
› Host: localhost:9000
› User-Agent: curl/8.3.0
› Accept: */*
›
‹ HTTP/1.1 400 Bad Request
‹ Date: Fri, 20 Oct 2023 12:50:35 GMT|
‹ Content-Length: 30
‹ Content-Type: text/plain; charset=utf-8
‹
* Connection #0 to host localhost left intact
Hook rules were not satisfied.
```

可以看到，失败的请求被拒绝，并且使用 `webhooks.yaml` 配置文件中指定的 HTTP 状态码返回，HTTP 响应体解释了失败的原因。

提供所需的查询和 `secret` 参数：

```sh
$ curl -4v 'http://localhost:9000/hooks/logger?secret=squirrel'
* Trying 127.0.0.1:9000...
* Connected to localhost (127.0.0.1) port 9000
› GET /hooks/logger?secret=squirrel HTTP/1.1
› Host: localhost:9000
› User-Agent: curl/8.3.0
› Accept: */*
›
‹ HTTP/1.1 200 OK
‹ Date: Fri, 20 Oct 2023 12:50:39 GMT
‹ Content-Length: 17
‹ Content-Type: text/plain; charset=utf-8
‹
webhook executed
* Connection #0 to host localhost left intact
```

Webhook 被成功执行后，我们可以在 syslog 输出中看到结果：

```sh
Oct 20 12:50:39 akai webhook[67758]: invoked with HTTP User Agent: curl/8.3.0
```

## 使用 HMAC 来保护 Webhook

前面提到的 HMAC 签名，当应用于 HTTP 正文并作为签名发送时，可以防止篡改，提供认证和完整性保护，但只针对正文，而不包括头部。让我们来实现这一点。我们的第一步是生成一个简短的密钥，并修改 `webhook.yaml` 以要求进行验证。

```sh
$ export HMAC_SECRET=$(head /dev/random | sha256)
```

为方便记忆，在本文章中我们使用 `n0decaf` 作为密钥，但你应使用一个强密码。

替换 `webhook.yml` 文件为以下内容，这将从负载中提取两个 JSON 值（负载是经签名的，因此可信），并将它们传给我们的命令以执行。

```yaml
---
- id: echo
  execute-command: /bin/echo
  include-command-output-in-response: true
  trigger-rule-mismatch-http-response-code: 400
  trigger-rule:
    and:
    # ensures payload is secure -- headers are not trusted
    - match:
        type: payload-hmac-sha256
        secret: n0decaf
        parameter:
          source: header
          name: x-hmac-sig
  pass-arguments-to-command:
  - source: ‘payload’
    name: ‘os’
  - source: ‘payload’
    name: ‘town’
```

使用 `openssl dgst` 计算正文的签名：

```sh
$ echo -n ‘{“os”:”freebsd”,”town”:”vienna”}’ \
    | openssl dgst -sha256 -hmac n0decaf
SHA2-256(stdin)= f8cb13e906bcb2592a13f5d4b80d521a894e0f422a9e697bc68bc34554394032
```

现在，带上正文和签名，让我们发出第一个签名请求：

```sh
$ curl -v http://localhost:9000/hooks/echo \
    --json {“os”:”freebsd”,”town”:”vienna”} \
    -Hx-hmac-sig:sha256=f8cb13e906bcb2592a13f5d4b80d521a894e0f422a9e697bc68bc34554394032

*  Trying [::1]:9000...
* Connected to localhost (::1) port 9000
› POST /hooks/echo HTTP/1.1
› Host: localhost:9000
› User-Agent: curl/8.3.0
› x-hmac-sig:sha256=f8cb13e906bcb2592a13f5d4b80d521a894e0f422a9e697bc68bc34554394032
› Content-Type: application/json
› Accept: application/json
› Content-Length: 32
›
‹ HTTP/1.1 200 OK
‹ Date: Sat, 21 Oct 2023 00:41:57 GMT
‹ Content-Length: 15
‹ Content-Type: text/plain; charset=utf-8
‹
freebsd vienna
* Connection #0 to host localhost left intact
```

在服务器端，运行 `-debug` 模式时，输出如下：

```sh
[webhook] 2023/10/21 00:41:57 [9d5040] incoming HTTP POST request from [::1]:11747
[webhook] 2023/10/21 00:41:57 [9d5040] echo got matched
[webhook] 2023/10/21 00:41:57 [9d5040] echo hook triggered successfully
[webhook] 2023/10/21 00:41:57 [9d5040] executing /bin/echo (/bin/echo) with arguments [“/bin/echo” “freebsd” “vienna”] and environment [] using as cwd
[webhook] 2023/10/21 00:41:57 [9d5040] command output: freebsd vienna

[webhook] 2023/10/21 00:41:57 [9d5040] finished handling echo
‹ [9d5040] 0
‹ [9d5040]
‹ [9d5040] freebsd vienna
[webhook] 2023/10/21 00:41:57 [9d5040] 200 | 15 B | 1.277959ms | localhost:9000 | POST /hooks/echo
```

每次单独计算签名是容易出错的。`gurl` 是一个早期项目的分支，它自动生成 HMAC 签名，并且简化了 JSON 处理。

签名类型和签名头部名称被加到密钥前面，并用 `:` 连接。它作为环境变量导出，这样它就不会直接显示在 shell 历史中。

```sh
$ export HMAC_SECRET=sha256:x-hmac-sig:n0decaf
$ gurl -json=true -hmac HMAC_SECRET \
  POST http://localhost:9000/hooks/echo \
  os=freebsd town=otutahi

POST /hooks/echo HTTP/1.1
Host: localhost:9000
Accept: application/json
Accept-Encoding: gzip, deflate
Content-Type: application/json
User-Agent: gurl/0.2.3
X-Hmac-Sig: sha256=f634363faff03deed8fbcef8b10952592d43c8abbb6b4a540ef16af0acaff172

{“os”:”freebsd”,”town”:”otutahi”}
```

如上所示，签名会为我们生成，并且添加 JSON 键=值对时无需引用和转义。

返回的响应也为我们进行了美化的格式化：HMAC 已被服务器验证，两个键的值已提取并作为参数传递给我们的 `echo` 命令，结果被捕获并返回在 HTTP 响应体中。

```sh
HTTP/1.1 200 OK
Date : Sat, 21 Oct 2023 00:50:25 GMT
Content-Length : 16
Content-Type : text/plain; charset=utf-8

freebsd otutahi
```

可以在 Port 的 [sample webhook.yaml](https://cgit.freebsd.org/ports/tree/www/webhook/files/webhook.yaml) 和 [详细文档](https://github.com/adnanh/webhook/tree/master/docs) 中找到更复杂的示例。

## 保护 Webhook 内容

虽然使用 HMAC 可以防止篡改信息正文，但它仍然是明文显示的，黑客依然可以看到内容。

我们可以通过添加传输层安全性（TLS）来进一步保护，使用自签名的 TLS 密钥和证书，为本地的 webhook 服务器提升安全性，并重启 webhook 服务器：

```sh
$ openssl req -newkey rsa:2048 -keyout hooks.key \
  -x509 -days 365 -nodes -subj ‘/CN=localhost’ -out hooks.crt

$ webhook -debug -hotreload \
  -secure -cert hooks.crt -key hooks.key \
  -hooks webhook.yaml
```

由于我们使用的是自签名证书，`curl` 命令需要额外加上参数 `-k` 来忽略证书验证，其他步骤与之前相同：

```sh
curl -4vk https://localhost:9000/hooks/logger?secret=squirrel
'
* Trying 127.0.0.1:9000...
* Connected to localhost (127.0.0.1) port 9000
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
* ALPN: server accepted http/1.1
* Server certificate:
* subject: CN=localhost
* start date: Oct 20 13:05:09 2023 GMT
* expire date: Oct 19 13:05:09 2024 GMT
* issuer: CN=localhost
* SSL certificate verify result: self-signed certificate (18), continuing anyway.
* using HTTP/1.1
› GET /hooks/logger?secret=squirrel HTTP/1.1
› Host: localhost:9000
› User-Agent: curl/8.3.0
› Accept: */*
›
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
› HTTP/1.1 200 OK
› Date: Fri, 20 Oct 2023 13:12:07 GMT
› Content-Length: 17
› Content-Type: text/plain; charset=utf-8
›
webhook executed
* Connection #0 to host localhost left intact
```

[gurl](https://github.com/skunkwerks/gurl) 不提供类似的参数，并且要求你正确地配置。对于生产环境，建议使用反向代理服务器，如 [nginx](https://nginx.org/) 和 [haproxy](https://haproxy.org/)，提供稳健的 TLS 终止，并通过 Let's Encrypt 等服务使用公共 TLS 证书。

## 使用 Github 和 Webhook 更新网站

要成功完成此操作，你需要有一个自己的域名和一台小型服务器或虚拟机来托管 daemon。虽然本文无法覆盖所有细节，如设置自己的网站、TLS 加密证书和 DNS 配置，但以下步骤大致适用于任何软件平台。

你需要设置一台代理服务器，例如 Caddy、nginx、haproxy 或类似的，确保启用有效的 TLS。一个好选择是通过 Let's Encrypt 使用 ACME 协议自动管理证书。

调整你的代理服务器，使其将适当的请求路由到 webhook daemon。你可以限制可以访问的 IP 地址，以及限制 HTTP 方法。GitHub 的 API 提供了 `/meta` 端点用于检索其 IP 地址，但需要保持更新。

启用 webhook 服务，再使用之前相同的参数启动你的 daemon：

```sh
# /etc/rc.conf.d/webhook
webhook_enable=YES
webhook_facility=daemon
webhook_user=www
webhook_conf=/usr/local/etc/webhook/webhooks.yml
webhook_options=” \
  -verbose \
  -hotreload \
  -nopanic \
  -ip 127.0.0.1 \
  -http-methods POST \
  -port 1999 \
  -logfile /var/log/webhooks.log \
  “
```

从外部验证该 URL 和 webhook daemon 是否可访问。

在你的代码托管平台（如 GitHub）创建一个新的 JSON 格式 webhook，并使用共享的 HMAC 密钥，在每次推送到仓库时触发它。

例如，在 GitHub 中，你需要提供：

- Payload URL，指向你的代理 webhook daemon 的外部链接
- Content-Type 设置为 `application/json`
- 共享密钥（如示例中的 `n0decaf`）

![](https://freebsdfoundation.org/wp-content/uploads/2024/01/02_add_webhook.png)

在 GitHub 上创建 webhook 后，你应该能够确认接收到了成功的事件。在你下次推送代码时，可以查看 GitHub 网站，查看 GitHub 发送的请求和 daemon 返回的响应。

![](https://freebsdfoundation.org/wp-content/uploads/2024/01/03_webhook_req.png)


