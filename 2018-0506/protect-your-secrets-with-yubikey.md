# 用 YubiKey 保护你的秘密

你创建复杂密码吗？记住新密码有多难？你把凭据放在显示器旁边的便利贴上吗？如果你做了以上任何一项，我们有一个替代方案，可以帮助保护你的隐私并让你更容易保持安全。

## YubiKey 概览

关于 Yubico 生产的 YubiKey 已经说了很多。它已成为提供安全 2FA——双因素认证——的最流行解决方案之一。它也可以用作辅助密码因子 U2F——通用第二因子。它提供强认证且易于使用。

YubiKey 是一个类似 USB 的设备。在最简单的用例中，当我们将 YubiKey 连接到计算机时，它在操作系统中被检测为键盘，该设备允许我们存储最多两个密码。根据我们按下设备上按钮的时间长短，它会释放第一个或第二个密码。在这个简单的用例中，我们可以用它来集成到几乎任何 Web 服务。

在这种场景下，我们可以用它来记住密码，不再需要使用便利贴。对于高级用户，此场景可以帮助保护他们的数据。轮换密码很困难，因为记住新密码很麻烦。现在，我们可以轮换最常用的密钥（例如访问我们的密码库），而不必费心记住它（虽然备份总是好主意）。将主要密码保存在 YubiKey 设备上的同时，我们仍可使用其他设备（如手机）作为第二因素。

## 认证方法

除了存储静态密码外，某些型号的 YubiKey 还支持更复杂的认证方法，如：

- **一次性密码（OTP）**——认证机制，生成只能使用一次的密码。
- **OATH – HOTP**——事件令牌，用 HOTP 算法生成 6 或 8 字符 OTP 密码。
- **OATH – TOTP**——6 或 8 字符 OTP 密码，TOTP 算法基于时间函数。
- **PIV（个人身份验证）兼容智能卡**——支持使用存储在 YubiKey 上的 RSA/ECC 私钥进行签名和解密。此模式像智能卡一样工作。
- **OpenPGP**——用存储在 YubiKey 上的 RSA 和 ECC 私钥加密和签名，使用 PKCS #11 等标准套件。
- **U2F**——开放认证标准，支持安全访问任意数量的在线服务。只需单个设备，无需额外驱动程序或客户端软件。

可以使用两步验证与谷歌、Facebook、GitHub 和 Hotmail 等 Web 服务配合使用。这很有用，因为即使攻击者窃取了第一个认证因素（账户的用户名和密码），他们也无法登录。我们可以通过使用支持 U2F 的 YubiKey 设备来确保这一点。当然，我们必须在所选 Web 服务上启用两步验证。

## 型号

有几种不同类型的 YubiKey 可用于各种用途。最基本的版本是 FIDO U2F Security Key。它支持静态密码认证，可以用 U2F 与 Gmail 和 Facebook 等最流行的应用集成。它不支持 HOTP 或 OpenPGP 等方法。此设备是撰写本文时他们提供的最实惠的产品，价格为 18 美元。

更高级的型号，称为 YubiKey 4 系列（也包括 YubiKey 4C、YubiKey Nano 和 YubiKey 4C Nano）已经支持 OTP 或 OATH 模式等基本加密方法。差异在于设备尺寸和 USB 端口类型。它们的用途绝对比前面提到的型号更广泛，文章后面将详细解释一些示例。

另一个可用选项是 YubiKey NEO。此型号支持的附加功能是通过 NFC 通信。例如，轻触设备后，智能手机可以读取 YubiKey 发出的 OTP。差异摘要见表 1。

| 特性 | FIDO U2F Security Key | YubiKey 4 系列 | YubiKey NEO |
| :--- | :-------------------: | :------------: | :---------: |
| OTP | | ✓ | ✓\* |
| OATH – HOTP | | ✓ | ✓ |
| OATH – TOTP | | | ✓\* |
| OpenPGP | | ✓ | ✓ |
| U2F | ✓ | ✓ | ✓ |
| Secure Element | | ✓ | ✓ |
| Smart Card (PIV) | | ✓ | ✓ |
| 支持 NFC 通信 | | | ✓ |

表 1：不同 YubiKey 型号的比较

\*需要额外应用（缺少内置实时时钟）。

## FreeBSD 工具

YubiKey 附带一组开源工具，这些工具是在类 UNIX 环境中集成 YubiKey 所必需的。对于 FreeBSD 用户，大多数这些工具可在 Ports 中以及二进制软件包仓库中获得。最有趣的两个软件包是 `security/ykpers` 和 `security/yubico-pam`。`security/ykpers` 软件包包含几个管理 YubiKey 的命令行工具：

- **ykpersonalize(1)**——编程 YubiKey 所必需；它处理几乎每个 YubiKey 功能的配置选项。
- **ykinfo(1)**——用于检索 YubiKey 的基本信息。
- **ykchalresp(1)**——允许在挑战-响应操作模式下用 YubiKey 签名数据。

## GELI 全盘加密

GELI 是 FreeBSD 上最流行的磁盘加密方法。使用 GELI 和 FreeBSD 最安全的方法之一是使用密码短语和密钥文件进行双因素认证。密钥文件保存在 U 盘上。当我们启动机器时，需要提供两个因素。解密设备后，我们取走带有文件的 U 盘。这样，如果有人偷了我们的计算机，他们还需要看到我们的密码并偷走我们的 U 盘。如果我们想更加偏执，可以把内核和密钥文件保存在 U 盘上。这样，即使有人能访问我们的计算机，也无法与我们的软件集成。入侵者仍可能篡改我们的硬件，但这要困难得多。

FreeBSD 最近开始支持全盘加密，这意味着连内核也是加密的——只有小的引导加载程序分区未加密。不幸的是，在当前实现中，我们不支持密钥文件，所以只能用密码短语加密磁盘。借助 YubiKey（可被检测为简单键盘），我们可以用它在启动时提供密码短语。这样使用时，我们失去了一个因素。我们可以通过使用普通键盘提供的密码短语和保存在 YubiKey 上的更长第二部分密码来缓解此问题。在这种场景下，有人不仅需要看到我们输入的密码短语，还需要偷走我们的 YubiKey。

为使设备满足这些要求，我们在静态模式下编程 Yubico 的第二个槽：

```sh
$ ykpersonalize -2 -o static-flag -o append-cr
```

借助 `append-cr` 标志，我们不必在每次使用此槽后按 Enter。在 GELI 的情况下，我们先提供密码短语，然后用 YubiKey 提供第二个因素。默认情况下，在 GELI 中输入密码时没有输出。在我们的情况下，我们会看到密码短语已提交，因为 YubiKey 提供的因素会包含尾随的 ENTER。

## 与 FreeBSD 登录集成

Yubico 提供一个 PAM 模块，可部署在现有认证系统中。该模块可在 `security/pam_yubico` 软件包中找到，它有两种工作模式：在线和离线认证。前者提供基于一次性密码的第二因素，但将认证委托给 Yubico 云服务。因此，它需要稳定的互联网连接。另一方面，为系统账户设置新购买的 Yubico 作为第二认证因素非常容易。

每个设备的第一个槽上都有制造商预设的密钥。该槽在所谓的 Yubico OTP 模式下工作。从此槽生成的每个密码由静态 12 字符部分和剩余的动态部分组成。静态部分从不改变，可视为设备的公共标识符。这些出厂默认值已被 Yubico 云服务所知。我们需要做的就是用 Yubico 网站上的特殊表单检索 API 令牌和用户 ID。

第二种操作模式更实用。它要求 YubiKey 在挑战-响应模式下工作，设备可被签发来签名用户应用发送的数据。在这个特定示例中，我们的应用是 Yubico PAM 模块。

首先，我们必须编程设备在挑战-响应模式下工作。在下面的示例中，我们将使用第一个槽：

```sh
$ ykpersonalize -1 -o chal-resp -o chal-btn-trig -o chal-hmac -o hmac-lt64 -o serial-api-visible
```

我们希望设备等待用户对每个操作的确认，这由 `chal-btn-trig` 标志保证。用户必须按下 YubiKey 上的按键才能登录系统，不幸的是要按两次。第一次按下是检查文件中的挑战是否与设备响应匹配。如果是，第二次按下会生成新的挑战-响应对并存储以供以后使用。我们需要生成一个初始挑战，默认存储在 **~/.yubico** 目录中：

```sh
$ ykpamcfg -1 -v
```

下一步是配置 PAM 模块。我们希望将第二因素与静态密码一起使用，以保护计算机的任何登录。为此，我们将修改 **/etc/pam.d/system** 文件，使“auth”部分如下所示：

```sh
# auth
auth   required   pam_unix.so        no_warn try_first_pass
auth   required   /usr/local/lib/security/pam_yubico.so mode=challenge-response
```

现在执行 `sudo -s` 命令，输入静态密码并按下设备上的按钮。它会等待用户操作最多 15 秒，在此期间 LED 指示灯会闪烁。当用户不采取行动或 YubiKey 未连接到 USB 端口时，认证将失败。

## 与 SSH 集成

YubiKey 的另一个有用应用是加强到 SSH 服务的远程认证。通常描述的方法利用 Yubico PAM 模块的在线操作模式。然而，Yubico 令牌符合 OATH-HOTP 标准，因此它可以与任何支持此标准的认证服务器配合工作。例如，Wheel Cerb AS 就是这样的多因素用户认证解决方案。我们将使用 `security/oath-toolkit` 软件包中包含的 `pam_oath` 模块，它不需要连接到任何外部云服务。

我们需要重新编程 YubiKey 以支持 OATH 模式。不幸的是，我们的设备上只有两个槽，所以我们需要覆盖之前的某个配置：

```sh
$ ykpersonalize -1 -o oath-hotp -o oath-hotp8 -o append-cr
```

我们要使用 8 位长、基于计数器的密码。命令输出应包含一行十六进制形式的密钥：

```sh
...
key: h:c621245c5f05eefec1d9f2960f34b865849dd074
...
```

我们需要将用户名和密钥存储在目标机器上的 pam_oath 数据库文件中（当然“alice”和密钥应替换为真实值）：

```sh
$ echo "HOTP alice - c621245c5f05eefec1d9f2960f34b865849dd074" >> /usr/local/etc/users.oath
```

下一步是修改 sshd PAM 配置以启用 `pam_oath` 模块。我们可以通过修改 **/etc/pam.d/sshd** 文件来实现，使“auth”部分如下所示：

```sh
# auth
...
auth        required      pam_unix.so         no_warn   try_first_pass
auth        required      /usr/local/lib/security/pam_oath.so usersfile=/usr/local/etc/users.oath window=16 digits=8
```

我们需要确保 **/etc/ssh/sshd_config** 文件中的 sshd 配置包含几个设置为适当值的选项，如下所示：

```sh
ChallengeResponseAuthentication yes
PasswordAuthentication no
UsePAM yes
```

最后，重新加载 **sshd(8)** 服务并尝试登录远程服务器，输入静态密码，然后使用令牌生成的动态密码。

## 结论

YubiKey 是一个非凡的设备，可用于企业或个人提高安全性。这个小设备支持许多不同的认证方法，可与许多流行的 Web 服务和 SSH 等程序配合使用。它还允许我们利用一些不支持 2FA 的工具的不完美之处。它可以用作第二因素或保存主密码。它是手机应用等其他解决方案的有趣替代品。YubiKey 还允许我们无痛更改密码，无需任何记忆。

---

**JAROSLAW ZUREK** 是 Wheel Systems 的软件开发者，他支持一个创建特权会话管理器的项目。他对加密、TLS 和底层/硬件编程感兴趣。

**MICHAL BORYSIAK** 是 Wheel Systems 的软件开发者，从事集中认证系统工作。他对底层操作系统概念、网络和网络安全着迷。

**MARIUSZ ZABORSKI** 是 Wheel Systems 的首席软件开发者。他自 2015 年起自豪地拥有 FreeBSD commit bit。Mariusz 的主要兴趣领域是操作系统安全和底层编程。在 Wheel Systems，Mariusz 领导一个团队，开发监控、记录和控制 IT 基础设施流量的最先进解决方案。业余时间，他喜欢写博客（<http://oshogbo.vexillium.org>）。
