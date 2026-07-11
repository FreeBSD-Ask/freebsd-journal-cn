# 现在用 Webhook 触发我

- 原文：[Kick Me Now with Webhooks](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-14-0/kick-me-now-with-webhooks/)
- 作者：**Dave Cottlehuber**

## 什么是 Webhook，为什么需要它？

Webhook 是一种基于 HTTP 的事件驱动远程回调协议，允许从几乎任何编程语言或工具轻松调用脚本和任务。Webhook 的优势在于其普及性和简洁性。只需一个简单的 HTTP URL，你就能请求远程服务器调暗灯光、部署代码或代为执行任意命令。

最简单的 Webhook 可以只是智能手机浏览器中一个加书签的链接，较复杂的版本可能需要强认证和授权。

虽然 Ansible 和 Puppet 等大型自动化工具集已经存在，但有时候更简单的方案就够了。Webhook 正是如此，让你能够按需安全地在远程计算机上执行任务。调用 webhook 可以看作是 “踢一脚”，这也是本文标题的由来。

## 集成

目前还没有官方标准，但通常 webhook 通过 POST 请求发送 JSON 对象体，使用 TLS 加密，并常用签名来防止篡改、网络伪造和重放攻击。

常见的集成包括 Mattermost、Slack 和 IRC 等聊天服务，GitHub 和 GitLab 等软件代码仓库，Zapier 或 IFTTT 等通用托管服务，以及 Home Assistant 等家庭自动化套件。webhook 的应用无处不在，可能性无限。

虽然你可以在一小时内写出一个最小的 webhook 客户端或服务器，但目前几乎每种编程语言都有许多现成选项。聊天软件通常提供内置的 webhook 触发器，用户可以通过 `/命令` 风格的语法来调用。IRC 服务器也没有被遗忘，主要通过守护进程或插件实现。

webhook 的一个不太明显的优势是能够划分安全和权限。低权限用户可以在远程系统上调用 webhook。远程 webhook 服务也可以低权限运行，负责验证和基本语法检查，待条件满足后再调用更高权限的任务。或许最终任务拥有访问特权令牌的权限，可以重启服务、部署新代码，或者让孩子们多玩一小时屏幕时间。

GitHub、GitLab 等常见软件代码仓库以及自托管方案也提供 webhook，可以在触发时包含分支名称、提交和做出更改的用户信息。

这使得构建工具来更新网站、重启系统或按需触发更复杂的工具链变得相对容易。

## 架构

典型的部署方式包括一个监听传入请求的服务器和一个提交请求及参数的客户端，可能还包含某些认证和授权。

## 服务器

首先讨论服务器端。通常，这是一个监听 HTTP 请求的守护进程，当请求匹配某些条件时才处理。如果条件不满足，请求将以相应的 HTTP 状态码拒绝；对于成功的提交，可以从已批准的请求中提取参数，然后按需调用自定义操作。

## 客户端

由于服务器使用 HTTP，几乎任何客户端都可以使用。[cURL](https://curl.se/) 是一个非常流行的选择，但我们将使用一个更顺手的工具 [gurl](https://github.com/skunkwerks/gurl)，它内置了对 HMAC 签名的支持。

## 消息

消息通常是 JSON 对象。如果你关心重放或时序攻击，应该在消息体中包含时间戳，并在进一步处理前验证它。如果你的 webhook 工具包可以签名和验证特定的 header，那也是一种选择，但大多数工具包不支持。

## 安全性

HTTP 请求的正文可以用共享密钥签名，然后将此签名作为消息 header 提供。这既提供了认证手段，也证明了请求在传输过程中未被篡改。它依赖共享密钥让两端都能使用附加的 HTTP header 和消息正文独立验证消息签名。

最常见的签名方法是 HMAC-SHA256。这是两种加密算法的组合——我们熟悉的 SHA256 哈希算法，它为更大的消息（在我们的场景中是 HTTP 正文）提供安全摘要；以及 HMAC 方法，它将密钥与消息混合产生唯一码，即数字签名。

这些函数组合在一起，可对消息是否被篡改提供高完整性校验。它就像内容上的数字封印，确认消息必定来自知道共享密钥的一方。

注意，同时使用 TLS 加密和签名可提供消息的机密性和完整性，但不提供可用性。位置得当的攻击者可以中断或涌入中间网络，消息将在无通知的情况下丢失。

通常做法是在 webhook 正文中包含时间戳，由于这被 HMAC 签名覆盖，可以缓解时序和重放攻击。

注意，不含时间戳的正文将始终具有相同的签名。这可能很有用。例如，这允许预计算 HMAC 签名，并使用不变的 HTTP 请求来触发远程操作，而无需在发出 webhook 请求的系统上提供 HMAC 密钥。

## 动手实践

我们将安装几个辅助包：一个 [webhook 服务器](https://github.com/adnanh/webhook)、知名工具 [curl](https://github.com/skunkwerks/gurl)，以及 gurl——一个让 webhook 签名变得简单的工具。

```sh
$ sudo pkg install -r FreeBSD www/webhook ftp/curl www/gurl
```

让我们用这个最小示例启动服务器，将其保存为 webhooks.yaml。

它将使用 **logger(1)** 命令向 **/var/log/messages** 写入一条简短记录，包含成功 webhook 的 HTTP User-Agent header。

注意，有一个 trigger-rule 键确保 HTTP 查询参数 `secret` 与单词 `squirrel` 匹配。

目前我们既没有 TLS 安全也没有 HMAC 签名，所以这还不是一个非常安全的系统。

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
  trigger-rule:
    match:
      type: value
      value: squirrel
    parameter:
       source: url
       name: secret
```

在一个终端中运行 `webhook -debug -hotreload -hooks webhook.yaml`。使用的标志含义不言自明。

在另一个终端中运行 `tail -qF /var/log/messages | grep webhook` 以便实时查看结果。

最后，让我们用 curl 触发 webhook，先不带查询参数，再带上：

```sh
$ curl -4v 'http://localhost:9000/hooks/logger'

*   Trying 127.0.0.1:9000...
* Connected to localhost (127.0.0.1) port 9000
> GET /hooks/logger HTTP/1.1
> Host: localhost:9000
> User-Agent: curl/8.3.0
> Accept: */*
>

< HTTP/1.1 400 Bad Request
< Date: Fri, 20 Oct 2023 12:50:35 GMT
< Content-Length: 30
< Content-Type: text/plain; charset=utf-8
<

* Connection #0 to host localhost left intact
Hook rules were not satisfied.
```

注意失败的请求如何使用 webhooks.yaml 配置文件中指定的 HTTP 状态码拒绝，返回的 HTTP 正文解释了原因。

提供必需的查询参数 `secret`：

```sh
$ curl -4v 'http://localhost:9000/hooks/logger?secret=squirrel'
*   Trying 127.0.0.1:9000...
* Connected to localhost (127.0.0.1) port 9000
> GET /hooks/logger?secret=squirrel HTTP/1.1
> Host: localhost:9000
> User-Agent: curl/8.3.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Fri, 20 Oct 2023 12:50:39 GMT
< Content-Length: 17
< Content-Type: text/plain; charset=utf-8
<

webhook executed
* Connection #0 to host localhost left intact
```

hook 已执行，我们可以在 syslog 输出中看到结果：

```sh
Oct 20 12:50:39 akai webhook[67758]: invoked with HTTP User Agent: curl/8.3.0
```

## 使用 HMAC 保护 Webhook

前面描述的 HMAC 签名应用于 HTTP 正文并作为签名发送，是防篡改的，提供了认证和完整性，但仅限于正文，不包括 header。让我们实现它。第一步是生成一个简短的密钥并修改 webhook.yaml 以要求验证。

```sh
$ export HMAC_SECRET=$(head /dev/random | sha256)
```

本文中我们将使用一个更好记的密钥 `n0decaf`，但你应该使用一个强健的密钥。

用以下内容替换 webhook.yml 文件，它将从 payload（已签名，因此可信）中提取两个 JSON 值，并传递给命令执行。

```yaml
---
- id: echo
  execute-command: /bin/echo
  include-command-output-in-response: true
  trigger-rule-mismatch-http-response-code: 400
  trigger-rule:
    and:
    # 确保 payload 安全——header 不受信任
    - match:
        type: payload-hmac-sha256
        secret: n0decaf
        parameter:
          source: header
          name: x-hmac-sig
  pass-arguments-to-command:
  - source: 'payload'
    name: 'os'
  - source: 'payload'
    name: 'town'
```

使用 `openssl dgst` 计算正文的签名：

```sh
$ echo -n '{"os":"freebsd","town":"vienna"}' \
    | openssl dgst -sha256 -hmac n0decaf
SHA2-256(stdin)= f8cb13e906bcb2592a13f5d4b80d521a894e0f422a9e697bc68bc34554394032
```

有了正文和签名，现在让我们发起第一个签名请求：

```sh
$ curl -v http://localhost:9000/hooks/echo \
    --json {"os":"freebsd","town":"vienna"} \
    -Hx-hmac-sig:sha256=f8cb13e906bcb2592a13f5d4b80d521a894e0f422a9e697bc68bc34554394032

*  Trying [::1]:9000...
* Connected to localhost (::1) port 9000
> POST /hooks/echo HTTP/1.1
> Host: localhost:9000
> User-Agent: curl/8.3.0
> x-hmac-sig:sha256=f8cb13e906bcb2592a13f5d4b80d521a894e0f422a9e697bc68bc34554394032
> Content-Type: application/json
> Accept: application/json
> Content-Length: 32
>
< HTTP/1.1 200 OK
< Date: Sat, 21 Oct 2023 00:41:57 GMT
< Content-Length: 15
< Content-Type: text/plain; charset=utf-8
<

freebsd vienna
* Connection #0 to host localhost left intact
```

在服务器端，`-debug` 模式正在运行：

```sh
[webhook] 2023/10/21 00:41:57 [9d5040] incoming HTTP POST request from [::1]:11747
[webhook] 2023/10/21 00:41:57 [9d5040] echo got matched
[webhook] 2023/10/21 00:41:57 [9d5040] echo hook triggered successfully
[webhook] 2023/10/21 00:41:57 [9d5040] executing /bin/echo (/bin/echo) with arguments ["/bin/echo" "freebsd" "vienna"] and environment [] using  as cwd
[webhook] 2023/10/21 00:41:57 [9d5040] command output: freebsd vienna

[webhook] 2023/10/21 00:41:57 [9d5040] finished handling echo
< [9d5040] 0
< [9d5040]
< [9d5040] freebsd vienna
[webhook] 2023/10/21 00:41:57 [9d5040] 200 | 15 B | 1.277959ms | localhost:9000 | POST /hooks/echo
```

每次单独计算签名容易出错。gurl 是一个早期项目的分支，增加了自动 HMAC 生成以及一些处理 JSON 的便利功能。

签名类型和签名 header 名称前置于密钥并以 `:` 连接。这作为环境变量导出，使其不会直接出现在 shell 历史记录中。

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

{"os":"freebsd","town":"otutahi"}
```

如上所示，签名自动为我们生成，添加 JSON 键值对也很简单，不需要引号和转义。

响应返回，为我们格式化打印：HMAC 已被服务器验证，两个键的值被提取并作为参数传递给 echo 命令，结果被捕获并返回在 HTTP 响应正文中。

```sh
HTTP/1.1 200 OK
Date : Sat, 21 Oct 2023 00:50:25 GMT
Content-Length : 16
Content-Type : text/plain; charset=utf-8

freebsd otutahi
```

更复杂的示例可在 port 的 [webhook.yaml 示例](https://cgit.freebsd.org/ports/tree/www/webhook/files/webhook.yaml) 或 [详细文档](https://github.com/adnanh/webhook/tree/master/docs) 中找到。

## 保护 Webhook 内容

虽然使用 HMAC 可以防止消息正文被篡改，但那些卑劣的黑客仍能看到明文内容。

让我们为 localhost 上的 webhooks 服务器添加一些传输层安全，使用自签名 TLS 密钥和证书，然后重新启动 webhook 服务器：

```sh
$ openssl req -newkey rsa:2048 -keyout hooks.key \
  -x509 -days 365 -nodes -subj '/CN=localhost' -out hooks.crt

$ webhook -debug -hotreload \
  -secure -cert hooks.crt -key hooks.key \
  -hooks webhook.yaml
```

curl 命令需要额外的 `-k` 参数来忽略自签名证书，但其余过程与之前相同：

```sh
curl -4vk https://localhost:9000/hooks/logger?secret=squirrel

*   Trying 127.0.0.1:9000...
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
*  subject: CN=localhost
*  start date: Oct 20 13:05:09 2023 GMT
*  expire date: Oct 19 13:05:09 2024 GMT
*  issuer: CN=localhost
*  SSL certificate verify result: self-signed certificate (18), continuing anyway.
* using HTTP/1.1
> GET /hooks/logger?secret=squirrel HTTP/1.1
> Host: localhost:9000
> User-Agent: curl/8.3.0
> Accept: */*
>
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
> HTTP/1.1 200 OK
> Date: Fri, 20 Oct 2023 13:12:07 GMT
> Content-Length: 17
> Content-Type: text/plain; charset=utf-8
>

webhook executed
* Connection #0 to host localhost left intact
```

[gurl](https://github.com/skunkwerks/gurl) 没有此选项，期望你以正规方式操作。对于生产环境，最好使用反向代理如 [nginx](https://nginx.org/) 或 [haproxy](https://haproxy.org/) 来提供可靠的 TLS 终止，并允许通过 Let’s Encrypt 等类似服务使用公共 TLS 证书。

## 用 GitHub 和 Webhook 更新网站

要成功实现此功能，你需要拥有自己的域名和一台小型服务器或虚拟机来托管守护进程。

虽然本文无法涵盖设置自己的网站、TLS 加密证书和 DNS 的全部细节，但以下步骤对于任何软件代码仓库都大致相似。

你需要设置代理服务器，如 Caddy、nginx、haproxy 等，并配置可用的 TLS。通过 Let’s Encrypt 使用 ACME 协议来维护是一个很好的选择。

你需要调整代理服务器，将适当的请求路由到 webhook 守护进程。考虑限制可访问的 IP 地址，以及限制 HTTP 方法。GitHub 的 API 有一个 `/meta` 端点可获取其 IP 地址，但你需要保持更新。

使用与之前相同的选项启用 webhook 服务，并通过 `sudo service webhook start` 启动守护进程：

```sh
# 文件：/etc/rc.conf.d/webhook
webhook_enable=YES
webhook_facility=daemon
webhook_user=www
webhook_conf=/usr/local/etc/webhook/webhooks.yml
webhook_options=" \
  -verbose \
  -hotreload \
  -nopanic \
  -ip 127.0.0.1 \
  -http-methods POST \
  -port 1999 \
  -logfile /var/log/webhooks.log \
  "
```

你需要从外部验证 URL 和 webhook 守护进程是否可访问。

在软件代码仓库端，创建一个新的 JSON 格式 webhook，使用共享的 HMAC 密钥，在每次推送到仓库时调用。

例如，使用 GitHub 时你必须提供：

- payload URL，指向代理后的内部 webhook 守护进程的外部 URL
- content-type application/json
- 共享密钥，如示例中的 `n0decaf`

在 GitHub 端创建 webhook 后，你应该能够确认成功接收到事件，然后在下次推送代码时，可以在 GitHub 网站上查看 GitHub 发送的请求以及你的守护进程提供的响应。
