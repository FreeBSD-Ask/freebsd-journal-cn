# 使用 TLS 改善 NFS 安全性

- 原文链接：[Using TLS to improve NFS Security](https://freebsdfoundation.org/wp-content/uploads/2021/07/Using-TLS-to-Improve-NFS-Security.pdf)
- 作者：**RICK MACKLEM**

传统上，NFS 提供的安全性非常有限，主要基于客户端的 IP 地址/DNS 主机名，使用 exports(5) 配置。这种方式可以通过 IP 地址欺骗来绕过，且对于没有固定、众所周知的 IP 地址或 DNS 主机名的移动客户端来说根本无法使用。此外，所有数据通常以明文在网络上传输，因此容易被嗅探。

RFC2203 于 1997 年 9 月发布，提供了一种机制，通过使用 GSSAPI 和 Kerberos 机制（通常称为 Kerberized NFS）来缓解至少部分上述问题。当通过“sec=krb5p”（KerberosV 带隐私）使用时，RPC 消息的参数和结果会在网络上传输时进行加密。Kerberos 在用户认证方面表现良好，但在机器认证方面不太方便。与 NFSv3 不同，NFSv4 需要一个“系统主体”，用于维护服务器上的 Open/Byte_range 锁定状态。Kerberos 有一个基于主机的 Kerberos 主体，格式为“host/@REALM”，可以为此创建一个密钥表条目并将其复制到客户端，作为“系统主体”使用。此“实例”应当保护密钥表条目不被另一个客户端使用（如果该条目被泄露）。然而，这使得这种 Kerberos 主体对于没有固定、众所周知的 DNS 主机名的移动客户端来说变得毫无用处。此外，对于“sec=krb5p”，只有 RPC 的数据负载部分是加密的，RPC 头部仍然暴露，导致将加解密工作卸载到专用硬件上变得不切实际。总之，考虑到实施 Kerberos 环境所涉及的管理工作，"sec=krb5p" 并未被广泛采用，并且对于没有固定、众所周知的 DNS 主机名的移动客户端效果不佳。

为了改善 NFS 的安全性，已撰写了一份名为“Towards Remote Procedure Call Encryption By Default”的互联网草案，描述了使用传输层安全性（TLS）来加密网络上的 RPC 消息流量，并使用 X.509 证书进行机器认证。由于 TLS 已被广泛采用，已经有了专用的硬件卸载解决方案，更不用说高效的软件实现了。本文描述了 FreeBSD 13 中的 NFS over TLS 实现，并展示了一个针对移动客户端（如笔记本电脑）的示例使用案例。

## 实现

虽然我将其称为 NFS over TLS，但更准确的说法是 Sun RPC over TLS，因为实现是在内核的 RPC（krpc）中完成的，并且对 NFS 层几乎是透明的。OpenSSL 的库提供了 TLS 和 X.509 证书在用户空间中的全面实现。然而，NFS 是在内核中实现的，将所有 NFS RPC 消息传递到用户地址空间以便由 OpenSSL 库处理似乎不切实际。幸运的是，FreeBSD 13 的内核添加了内核 TLS（KTLS）[TLS Offload in the Kernel，John Baldwin，FreeBSD Journal，2020 年 5 月/6 月 https://issue.freebsdfoundation.org/publication/?m=33057&i=667002&p=12&ver=html5]，它在内核中的网络栈内高效地处理 TLS 应用数据记录，包括加密/解密。

这为在 TLS 应用数据记录中封装/加密 RPC 消息并在接收端解密/解封装这些 RPC 消息提供了基本机制。然而，它并不处理非应用数据记录，例如用于 TLS 握手的记录。为了处理非应用数据记录，分别在用户空间实现了 rpc.tlsclntd(8) 和 rpc.tlsservd(8) 用于客户端和服务器。这些守护进程通过自定义的 RPC 从内核接收上行调用，使用 AF_LOCAL 套接字上的 krpc，以类似 gssd(8) 守护进程为 Kerberized NFS 所做的方式来工作。为了处理这些上行调用，守护进程执行 OpenSSL 库调用，承担处理非应用数据记录的重任，包括处理 TLS 握手。守护进程还使用自定义系统调用向内核中的 krpc 注册，以及处理需要将文件描述符与内核中已经存在的套接字关联的特殊情况。

当客户端希望执行 NFS over TLS 时，它会执行一个 STARTTLS 空 RPC。空 RPC 是一个没有参数或结果的 RPC，通常分配为 RPC 编号 0。为了执行 STARTTLS，空 RPC 请求使用一种新的 RPC 凭证类型 AUTH_TLS。对于 FreeBSD 中的 NFS 服务，如果 rpc.tlsservd 正在运行，krpc 会回复一个由八个 ASCII 字符“STARTTLS”组成的凭证验证器。NFS 客户端执行的此 STARTTLS 探测会触发 TLS 握手，从而在用于 RPC 消息传输的 TCP 连接上设置 TLS。

此时服务器的操作序列为：

- 阻止 TCP 套接字上的 krpc 接收。
- 发送空 RPC 回复，凭证验证器为“STARTTLS”。
- 向 rpc.tlsservd 发出握手上行调用。

- 获取 TCP 套接字的文件描述符。此时 krpc 已拥有客户端 NFS 连接的 TCP 套接字，但没有文件描述符引用。这是实现中的一个较为挑战的部分。我的解决方案是使用守护进程的自定义系统调用将文件描述符与套接字关联。待完成，套接字的关闭工作就交给守护进程，而不是 krpc。
  - 向一个链表中添加一个结构体，按唯一的 64 位引用号作为键关联套接字文件描述符。
  - 调用 SSL_set_fd() 将套接字与 SSL 上下文关联。
  - 调用 SSL_accept() 执行实际的握手。
  - 如果握手成功，调用 BIO_get_ktls_send() 和 BIO_get_ktls_recv() 来检查是否已在套接字上启用 KTLS。如果其中任何一个返回零，握手视为失败。
  - 根据守护进程的命令行选项，验证客户端提供的任何 X.509 证书，并使用证书中指定的任何用户映射来创建 POSIX 用户凭证。

- 向上行调用 RPC 回复一组标志，指示握手是否成功，是否接收到经过验证的证书，以及是否有根据证书映射的 POSIX 用户凭证（如果有）。回复中还包括套接字的唯一 64 位引用号，以及守护进程的启动日期/时间，以便内核可以在后续的上行调用中引用该套接字。启动日期/时间使得引用号与可能由守护进程的先前或后续实例使用的相同引用号区分开来。
  - 如果握手成功，则标记 krpc 套接字为使用 TLS，并将标志和凭证（如果有）包含在上行调用的回复中。
  - 解锁套接字上的内核 RPC 接收。

如果握手成功，套接字现在应该准备好处理 RPC 消息，KTLS 会在 sosend() 调用下进行应用数据记录的封装/加密，而在 krpc 使用的 soreceive() 调用下进行解密/解封装。

如果非应用数据记录位于套接字接收队列的头部，则为 soreceive() 调用引入了一个新的 MSG_TLSAPPDATA 标志，表示调用应该返回 ENXIO，从而使非应用数据记录保持在套接字接收队列的头部。ENXIO 返回触发了一个上行调用到 rpc.tlsservd 以处理非应用数据记录。内核代码会阻止 krpc 在套接字上的接收，然后执行处理记录的上行调用到守护进程。64 位引用号和守护进程的启动日期/时间作为参数传递上行调用，以便守护进程能够识别正确的套接字。

- 这个上行调用只做一个 SSL_read()，其长度参数为零。这个调用总是失败，但会处理套接字接收队列头部的非应用数据记录，然后才会失败。

第三次上行调用到守护进程是用来关闭和断开 TCP 套接字的，64 位引用号和守护进程启动日期/时间作为参数传递。

- 这个上行调用会关闭套接字并将套接字元素从链表中移除。如果还没有做（如通过 SSL_get_shutdown() 所示），这个上行调用还会在关闭套接字之前执行 SSL_shutdown()，以向客户端发送一个 Peer Reset TLS 记录。

虽然上述所有操作都由 krpc 处理，但 NFS 服务器确实使用与 TLS 相关的新标志，这些标志由 krpc 传递给 NFS 服务器，以确定基于以下 exports(5) 选项是否允许该 RPC。这里有三个新的 exports(5) 选项：

tls - 表示客户端必须使用 NFS over TLS，但不要求在 TLS 握手期间向服务器提供任何 X.509 证书。

tlscert - 表示客户端必须使用 TLS，并且必须在 TLS 握手期间提供一个已验证的 X.509 证书。

tlscertuser - 表示客户端必须使用 TLS，必须在 TLS 握手期间提供一个已验证的 X.509 证书，并且该证书必须已成功映射到一个 POSIX 用户凭证（）。此映射是从 subjectAltName 中的 otherName 组件找到的登录名生成的，登录名后面附加“@domain”，其中“domain”与服务器使用的域匹配。只有在 rpc.tlsservd 使用 -u/--certuser 命令行选项启动时，才会生成此映射。

如果未指定上述任何 exports(5) 选项，则允许使用 TLS，但不是必需的。

rpc.tlsservd 还有一个命令行选项，指定守护进程要求客户端 IP 地址的反向 DNS（rDNS）名称与客户端 X.509 证书中的 subjectAltName 的“DNS”组件匹配。这类似于 RFC 6125 建议客户端做的操作，用于验证域名应用服务的身份。由于该选项旨在防止客户端 IP 地址欺骗，无法使用 exports(5)，因为它是基于客户端的 IP 地址来设置的。因此，该选项指定所有使用 NFS over TLS 的客户端必须满足这一标准，未能满足的将导致握手失败。它是最强的客户端主机身份检查，但要求所有客户端都必须拥有已验证的 X.509 证书，并且证书的 subjectAltName 中的 DNS 组件正确。当指定此选项时，所有客户端还必须具有固定的、已知的 DNS 地址。

客户端守护进程的功能与此类似，但有所不同。与 rpc.tlsservd 不同，rpc.tlscltnd 仅在指定了 -m/--mutualverf 命令行选项时才需要证书。客户端还可以处理存储在不同文件中的多个证书，以防不同的 NFS over TLS 服务器需要不同的证书。

当 NFS 挂载与服务器建立新的 TCP 连接，并且指定了“tls”挂载选项时，krpc 将执行以下操作：

- 发送带有 AUTH_TLS 类型凭证的 Null RPC 请求。
- 如果收到带有凭证验证器（由 ASCII 字节“STARTTLS”组成）的 Null RPC 回复：  
  - 阻止内核 RPC 在新 TCP 套接字上的接收。
  - 向 rpc.tlsclntd 执行握手上行调用。为了处理握手，rpc.tlsclntd 中的操作为：  
    - 为 TCP 套接字获取文件描述符。此时 krpc 已经拥有客户端的 NFS 连接的 TCP 套接字，但没有该套接字的文件描述符引用。通过守护进程的自定义系统调用来完成这项操作，类似于 rpc.tlsservd。  
    - 调用 SSL_set_fd() 将套接字与 SSL 上下文关联。  
    - 如果守护进程是使用 -m/--mutualverf 命令行选项启动的，则调用 SSL_[ctx_]use_certificate_file()/SSL_[ctx_]use_PrivateKey_file() 在握手期间提供证书。上行调用的参数可以覆盖证书/密钥文件的默认名称。默认名称为“cert.pem”和“certkey.pem”，但可以通过“tlscertname”挂载选项在每个挂载上进行覆盖，以防不同的 NFS 服务器需要不同的证书。  
    - 调用 SSL_connect() 执行实际的握手。  
    - 如果握手成功，调用 BIO_get_ktls_send() 和 BIO_get_ktls_recv() 检查 KTLS 是否已在套接字上启用。如果其中任何一个返回零，则认为握手失败。如果握手成功：  
      - 向套接字文件描述符的链表中添加一个结构，使用唯一的 64 位引用号作为键。  
      - 向上行调用 RPC 回复，指示握手成功。回复中包含套接字的唯一 64 位引用号，以及守护进程的启动日期/时间，以便内核在随后的上行调用中可以引用该套接字。  
    否则：  
      - 关闭套接字。  
      - 向内核回复失败，这将导致所有后续的 NFS RPC 失败并返回 EACCES。  
  - 收到上行调用回复后，krpc 设置标志，指示握手是否成功，并解除对套接字的 krpc 接收阻塞。

对于客户端，如果在指定了“tls”选项的挂载时，STARTTLS Null RPC 或 TLS 握手失败，则所有 RPC 将失败并返回 EACCES。这样做是为了确保在指定了“tls”选项的 NFS 挂载只有在 TLS 成功工作时才会正常运行。

## 移动设备（如笔记本电脑）作为用例

移动设备，如笔记本电脑，通常可以从任何地方访问互联网，并且没有固定的、已知的 IP 地址。为了允许笔记本电脑从互联网上的任何地方挂载 NFSv4 文件服务器，需要一些合理的安全机制。虽然 NFS over TLS 可以用于 NFSv3 挂载，但从任何地方启用 NFSv3 挂载会比较麻烦，因为 Mount 协议通过 rpcbind 使用动态分配的端口号，而 NFSv4 挂载只使用端口 #2049。因此，本示例将使用 NFSv4 挂载。

可以使用带有隐私保护的 Kerberized NFS 从 FreeBSD 笔记本电脑进行挂载。笔记本电脑用户需要执行以下命令：

```sh
# sysctl vfs.usermount=1 - 以 su/root 身份执行
# service gssd onestart - 以 su/root 身份执行
% kinit - 获取用户的 TGT
% mount -t nfs -o sec=krb5p,nfsv4,minorversion=1 nfsv4-server.uoguelph.ca:/ /mnt
 - 以非 root 用户身份执行
 - 使用挂载点
% umount /mnt
```

通过使用挂载选项“sec=krb5p”，但不使用“gssname”挂载选项，FreeBSD 客户端将使用“用户主体”作为“系统主体”。如果用户的 TGT 在“umount”之前过期，挂载将会失败。

据我所知，这种方法并未广泛采用，可能是因为安装 Kerberos 和维护一个可以从互联网上任何地方访问的 KDC 所需的工作量。

要使用 NFS over TLS 实现这一点，需要为客户端生成一个 X.509 证书。虽然有多种方法可以创建/签署 X.509 证书，但这可以通过站点本地的 CA 和在 FreeBSD 13 中使用 openssl(1) 命令轻松完成。

- • 为笔记本电脑创建一个由站点本地 CA 签名的证书。使用 openssl(1)，可能需要执行以下命令：

  ```sh
  # openssl genrsa -aes256 -out certkey.pem
  # openssl req -new -key certkey.pem -addext "subjectAltName=otherName:1.3.6.1.4.1.2238.1.1.1;UTF8:rmacklem@uoguelph.ca" -out req.pem
  # openssl ca -in req.pem -out cert.pem
  ```
- 将 cert.pem 和 certkey.pem 以某种安全的方式复制到笔记本电脑的 /etc/rpc.tlsclntd 目录中。
- 启用客户端守护进程，使用该证书。
  - 编辑 /etc/rc.conf 并添加：

    ```sh
    tlsclntd_flags="-m"
    ```

  - 由于使用了“-aes256”选项，rpc.tlsclntd 在启动时会询问输入密码短语，因此最好在执行挂载之前手动启动守护进程，而不是在启动时自动启动。
  - 如果希望在启动时启动，请添加以下内容到 /etc/rc.conf：

    ```sh
    tlsclntd_enable="YES"
    ```

  - 或者在挂载之前手动启动：

    ```sh
    # service tlsclntd onestart
    ```
- 待笔记本电脑连接到互联网，就可以以 su/root 身份执行挂载：

  ```sh
  # mount -t nfs -o nfsv4,minorversion=1,tls nfsv4-server.uoguelph.ca:/ /mnt
  ```

由于客户端提供了由站点本地 CA 签名的证书，服务器可以合理地确认客户端拥有由站点本地 CA 管理员创建的证书。创建客户端私钥时使用的“-aes256”选项迫使 rpc.tlsclntd 在启动时询问输入密码短语。这可以防止笔记本电脑被盗或证书/密钥文件被复制到另一个客户端后轻易的泄露。如果证书要在未经授权的客户端上使用，必须通过某种方式捕获或破解密码短语。

如果由于任何原因笔记本电脑不再允许执行挂载，也可以撤销证书并将其添加到证书吊销列表（CRL）中。

在上述示例中，所有在服务器上执行的 RPC 将使用登录名“rmacklem”在 NFSv4 服务器上的 POSIX 凭证。这避免了笔记本电脑需要与服务器保持统一的 uid 和 gid 空间。它还将因笔记本电脑被入侵而导致的风险限制为“rmacklem”可访问的文件。此可选配置可能与互联网草案不符。该草案的一位共同作者同意根据 TLS 握手期间提供的 X.509 证书将客户端 RPC 凭证映射到特定用户是有用的，事实上他为此创造了“TLS 身份压缩”（TLS Identity Squashing）这个术语。然而，这位作者更倾向于使用数据库将证书的元组映射到“用户”。他认为将“用户”包含在证书中混淆了“机器”与“用户”凭证。作为一个务实主义者，我认为将“用户”包含在证书中只是实现这一目标的一种简单方法。如果这种做法成为首选实践，rpc.tlsclntd 可以修改为使用平面文件/数据库来实现这一点。

以上只是一个示例用例。守护进程的命令行选项支持多种配置，从仅要求 TLS 对传输中的 RPC 消息进行加密，到要求所有客户端提供可验证的 X.509 证书，其中 subjectAltName 的 DNS 组件必须与客户端 IP 主机地址的 rDNS 名称匹配。客户端还可以以 RFC 6125 推荐的方式验证 NFS 服务器的真实性，用于 TLS 域名应用服务。

要在 FreeBSD 13 上设置 NFS over TLS，请参阅：https://people.freebsd.org/~rmacklem/nfs-over-tls-setup.txt

另外，请查阅 rpc.tlsclntd(8)、rpc.tlsservd(8)、exports(5)、ktls(4) 和 mount_nfs(8) 的手册页。

---

**RICK MACKLEM**  Rick 已经有 30 余年的工作经验，从 1980 年开始，他是计算机科学系的系统管理员，管理着包括 BSD 系统在内的多种系统。当 VAX 11/780 替换为 MicroVAXII 系统时，需要在 4.3BSD 上实现 NFS，因此 Rick 实现了这个功能并将其贡献给了 CSRG。Usenix 论文《Lessons Learned Tuning the 4.3BSD NFS Implementation》描述了作为这项工作的部分内容，第一个在 TCP 上实现的 NFS。Rick 现已退休，继续致力于 FreeBSD 上 NFS 的实现工作。

