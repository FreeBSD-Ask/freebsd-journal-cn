# 在 FreeBSD 上进行可审计的凭证泄露检查，无需发送完整密码

- 原文：[Auditable Credential-Exposure Checks on FreeBSD Without Sending Full Passwords](https://freebsdfoundation.org/our-work/journal/browser-based-edition/production-deployments/auditable-credential-exposure-checks-on-freebsd-without-sending-full-passwords/)
- 作者：**Emre Çapan**

## 保护隐私的实用范围查询工作流：有界披露、严格响应处理、有用的审计证据

密码泄露检查，就是查某人打算使用的密码是否已经出现在泄露密码集合里。Have I Been Pwned（HIBP）的 Pwned Passwords 这类服务，维护着可供检索的哈希，对应的是从数据泄露及其他入侵数据中找回的密码值。用户创建或修改密码时，应用可以查询该集合，拒绝已被攻击者掌握的候选密码。

例如，某人可能选了对其账户而言全新的密码，但该密码已在别处使用并泄露。命中就是换用其他密码的理由。它不能证明此人的账户已被入侵，不能指认具体受害者，也不能揭示密码的来源。未命中只说明该值在受检集合中找不到，并不保证密码强度或安全。这项检查只是对周边身份验证控制的补充。

范围查询本身带来一个问题：客户端如何询问某个机密，又不把这个机密交给回答问题的服务？通过 HTTPS 发送完整密码，接收服务仍然会看到它。发送完整的未加盐哈希，同样给服务方或监听者留下校验器，可用来验证密码猜测。

范围查询可以限制这种披露。客户端在本地计算密码的 SHA-1 哈希，只发送前五个十六进制字符。服务返回该范围内的哈希后缀，客户端在 FreeBSD 主机上完成最后比较。完整密码和完整哈希都不会发给范围服务。这种做法通常称为 k-匿名查询，但它并非完全匿名：服务仍能看到前缀、源 IP 地址、请求时间。SHA-1 在这里只是该语料库要求的查询格式，不是密码存储方案。

本文围绕公开的 Pwned Passwords 范围接口，在 FreeBSD 上构建交互式检查器。文章沿着候选密码的路径展开：从终端输入，经过 HTTPS 请求、严格的响应校验，到最小的审计记录。程序区分命中、未命中、检查不可用或无效三种情况，超时绝不能当作干净的结果。文中还会说明响应填充、缓存限制、测试，还有何时适合使用 Jail、Capsicum 或由 rc.d 管理的工作进程。整个工作流不依赖任何商业产品。

### 真正的问题在查询，不在哈希函数

泄露检查器的回答范围很窄：候选密码是否出现在先前从数据泄露或其他公开入侵数据中观察到的值的语料库里？它不能证明使用该密码的账户已被入侵，不能指出密码来自何处，也不能在答案为否时让密码变得安全。有用的结果是决策信号：命中说明候选密码不应设置或复用，未命中只排除一项已知风险指标，常规的密码质量与账户安全控制仍要保留。

最朴素的实现把密码发给远程服务。HTTPS 保护传输中的连接，但服务仍然收到机密。表面上有改进的实现改为发送完整的 SHA-1 摘要。这样做避开了线路上的明文，但摘要仍是稳定的校验器，可用来验证猜测。常见密码可以廉价地测试，服务还会拿到确切的查询值。

前缀范围协议改变了披露内容。客户端在本地算出 160 位的 SHA-1 摘要，发送前五个十六进制字符，接收与该前缀关联的全部已知后缀。五个十六进制字符披露摘要的 20 位，从 1,048,576 个可能范围中选定一个。客户端在本地比较其余 35 个字符。远程服务只知道请求的范围，不知道完整摘要或密码。这种模式通常称为 k-匿名范围查询。不要把这个名称误当成完全匿名：返回分组的大小会有变化，服务仍然能观察到来源和前缀。

SHA-1 不适合存储密码、签署数据，也不适合用来保证抗碰撞能力。FreeBSD 自带的 **md5(1)** 手册页警告，SHA-1 容易遭受实用的碰撞攻击 **[3]**。在这个工作流里，SHA-1 不是密码存储结构，而是公开语料库的兼容键，因为该语料库已经按 SHA-1 建立索引。这一区别至关重要。本地账户数据库仍然需要专门设计的加盐密码哈希方案；新协议不应仅仅因为该范围接口用了 SHA-1 就跟着选它。

### 威胁模型与明确的限制

写代码之前，先确定哪些观察和失败值得关注。本文的工作流假定 FreeBSD 主机在输入那一刻是可信的。如果键盘记录器、恶意内核模块、敌意终端或拥有足够调试权限的进程攻陷了主机，范围查询无法保护候选密码。工作流还假定配置的 HTTPS 端点是预期服务，主机的信任库和时间都有效。

设计针对以下风险：

1. **通过进程参数泄露。** 密码从不作为命令行参数接收。查看进程就能看到参数，而且参数常被复制到 shell 历史、作业记录、支持会话记录里。
2. **通过环境变量泄露。** 密码不经环境提供。环境值可能通过诊断信息、某些权限模型下的进程检查、崩溃处理、自动化配置泄露出去。
3. **向范围服务泄露。** 离开主机的只有五个字符的前缀。完整摘要和后缀留在本地。
4. **响应大小的观察。** 请求要求服务对响应做填充。Pwned Passwords 文档描述的目标是 800 到 1,000 条结果，并用计数为零的合成条目来标识填充 **[9]**。稠密范围可能本身就包含多于该目标的真实记录，所以客户端不能强制 1,000 行的上限。填充可以减少从流量大小获得的信息，但不能消除。
5. **不安全的网络行为。** 客户端只允许 HTTPS，保留常规的证书与主机名验证，设置较短的连接超时和总超时，并把网络状态不明当作未知结果，而不是当作干净的密码。
6. **解析器混淆。** 响应在每一行获得信任之前，必须先匹配狭义的文法。过大、为空、格式错误或混合格式的响应体，一律按失败即拒绝（fail closed）处理。
7. **日志里带机密。** 审计记录省略密码、完整摘要、前缀、后缀、出现次数、响应体、命令输出。

还有若干风险仍在。服务能看到源 IP 地址、请求时间、用户代理、20 位前缀。只能看到加密流量的网络监听者，仍能看到时间和大致大小。反复检查或逐字符增量检查会形成序列，缩小机密范围。HIBP 文档特别警告不要在每输入一个字符后就检查，因为监听者可以合并这些请求 **[9]**。等候选密码输入完整，只发一次请求。

被攻陷的服务可能返回假数据。TLS 验证的是证书中指定的端点，而不是其语料库的真实性或完整性。未命中只说明该确切摘要在当时接受的响应中不存在。可用性故障、TLS 无效、超时、响应体格式错误，都必须产生不确定结果，绝不能产生未命中。

最后，shell 进程无法承诺从全部内存中确定地擦除变量。清空并 unset 变量只能缩短它的有效寿命，不提供正式的归零保证。要求更高保障的客户端应当使用小型编译程序，用 FreeBSD 的 **explicit_bzero(3)** 这类原语清除敏感缓冲区；该函数的设计目标就是不被编译器的死存储优化删除 **[7]**。

### 测试目标与依赖

这套实现最初于 2026 年 8 月 31 日在 FreeBSD 15.1-RELEASE 上验证，基本系统工具接口也对照 14.4 的文档做了核对。截至 2026 年 9 月 10 日，FreeBSD 安全页面列出的受支持版本为 15.1-RELEASE、15.0-RELEASE、14.5-RELEASE、14.4-RELEASE **[2]**。FreeBSD 14.5 于 9 月 8 日发布；它出现在该列表中，并不表示本示例在其上测试过。发布页面把生产版本、旧版本与安全支持区分开来 **[1]**。示例使用基本系统的 **sh(1)**、**sha1(1)**、**mktemp(1)**、**logger(1)**，并为了满足一项特定需求而引入 curl：范围服务的填充选项要用自定义 HTTP 请求头表达。FreeBSD 自带的 **fetch(1)** 提供 HTTPS 获取和证书控制，但它的命令行接口没有通用的自定义请求头选项 **[4]**。

从 FreeBSD 软件包仓库安装 curl：

`# pkg install curl`

FreeBSD 手册把 `pkg install curl` 记录为标准二进制软件包流程 **[5]**。该软件包通过包管理器引入它所需的 TLS 依赖。不要用 curl 的 `--insecure` 选项绕过信任库问题。应当修复信任库、系统时间、代理拦截或端点配置。

脚本把继承来的搜索路径换成一组简短的标准目录，这些目录由管理员控制。随后脚本解析包客户端和保留在变量里的基本系统工具；其他标准命令都走同一条固定路径。这一点在两个目标发行版系列之间很重要，因为基本系统工具的安装位置可能变化，即使其命令行接口仍然兼容。管理员应当在目标版本上用 `command -v curl sha1 logger uuidgen` 核对全部四项解析结果，而不是悄悄换成别的程序。软件包路径位于 **/usr/local** 之下；基本系统工具可以从已安装的手册页和文件系统确认。特别是，解析 `uuidgen` 可以避免依赖某个版本专属的绝对路径。

下面四个步骤中的 shell 代码块是同一个脚本的连续片段。请按顺序拼装。第一个代码块包含 shebang 和全部设置；后面的每个代码块都在同一个 shell 进程中继续。

### 步骤 1：读取候选密码，且不留在历史记录中

输入包装器不接受位置参数，并要求交互式终端。它临时关闭终端回显，读取一整行，恢复先前的终端状态，并在清空 shell 变量之前对确切字节做哈希。读取候选密码之前，它会清空用于机密材料的继承变量，并关闭 shell 跟踪。不要用调试器或跟踪包装器拿真实密码运行它。

```sh
#!/bin/sh
set +x
set -eu
umask 077

unset candidate digest prefix suffix count result http_code
result=

PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/sbin:/usr/local/bin
export PATH

CURL=$(command -v curl) || exit 69
SHA1=$(command -v sha1) || exit 69
LOGGER=$(command -v logger) || exit 69
UUIDGEN=$(command -v uuidgen) || exit 69

[ "$#" -eq 0 ] || {
    echo "usage: exposure-check" >&2
    exit 64
}

[ -t 0 ] || {
    echo "refusing non-interactive password input" >&2
    exit 64
}

old_tty=$(stty -g)
restore_tty() { stty "$old_tty" 2>/dev/null || true; }
trap restore_tty EXIT
trap 'restore_tty; exit 129' HUP
trap 'restore_tty; exit 130' INT
trap 'restore_tty; exit 143' TERM

printf "Candidate password: " >&2
stty -echo
IFS= read -r candidate || {
    printf "\ninput failed\n" >&2
    exit 65
}
restore_tty
trap - EXIT HUP INT TERM
printf "\n" >&2

digest=$(printf '%s' "$candidate" | "$SHA1" -q)
candidate=
unset candidate
```

引号不是装饰。`printf '%s' "$candidate"` 保留空格，并防止通配符展开。不要用 echo，它对选项和反斜杠的处理会随输入和实现而变化。不要用 `sha1 -s "$candidate"`，因为那样密码会成为另一个进程的参数。

这个接口有意拒绝管道输入。这一选择让交互式工具难以被 CI 系统或 cron 作业误用。集成同一引擎的应用，应当通过进程内 API 或范围很窄的文件描述符传入候选密码，而不是放宽包装器去接受经参数传递的机密。

### 步骤 2：拆分摘要，发一次有界请求

把摘要规范化为大写，确认它恰好包含 40 个十六进制字符，然后切成五个字符的前缀和 35 个字符的后缀。只有前缀会插入 URL。

```sh
digest=$(printf '%s'
"$digest" | tr '[:lower:]' '[:upper:]')

[ "${#digest}" -eq 40 ] || exit 70
case "$digest" in
    *[!0-9A-F]*) exit 70 ;;
esac

prefix=$(printf '%.5s' "$digest")
suffix=${digest#?????}
digest=
unset digest

response=$(mktemp -t exposure-check) || exit 70
trap 'rm -f "$response"' EXIT
trap 'exit 129' HUP
trap 'exit 130' INT
trap 'exit 143' TERM

sleep 1

http_code=
if ! http_code=$("$CURL" -q \
    --fail \
    --silent \
    --show-error \
    --proto '=https' \
    --tlsv1.2 \
    --connect-timeout 5 \
    --max-time 20 \
    --max-filesize 1048576 \
    --retry 2 \
    --retry-delay 1 \
    --retry-max-time 45 \
    --user-agent 'freebsd-exposure-check/1.1' \
    --header 'Add-Padding: true' \
    --write-out '%{http_code}' \
    --output "$response" \
    "https://api.pwnedpasswords.com/range/$prefix")
then
    result=unknown
elif [ "$http_code" != 200 ]; then
    result=unknown
fi

prefix=
http_code=
unset prefix http_code
```

`-q` 选项必须是 curl 的第一个参数。它禁止自动加载 curlrc 文件，这样个人配置就无法悄悄加入跟踪、跟随重定向或关闭证书检查 **[10]**。`--proto` 限制只允许 HTTPS。没有启用重定向跟随。客户端单独捕获 HTTP 状态，只接受 200，所以 3xx 响应不会被误当成范围数据。curl 默认验证对端证书和主机名。`--tlsv1.2` 设定最低 TLS 版本，同时不禁止客户端和服务器支持的更高版本。20 秒上限适用于每次传输尝试。`--retry-max-time 45` 防止重试窗口过去后再发起新重试，不过在限制之前已经开始的一次尝试可以在限制之后完成 **[10]**。两次重试足以应对暂时性故障，又不会形成无界循环。

一秒延迟是本地运维策略，并不表示示例服务要求如此。它的文档目前写明 Pwned Passwords 端点没有速率限制 **[9]**。本地设上限仍然有用，因为即使上游提供方没有公布固定配额，配置失误、UI 循环和批处理作业也可能造成滥用流量。

重试策略需要细致权衡。请求发出后超时，并不能证明服务没有做事，但这个操作是幂等的 GET，所以少量重试可以接受。如果端点契约发生变化，就要重新审视这一假设。绝不要在账户创建路径上无限重试，也绝不要把重试耗尽重新解释为未命中。

### 步骤 3：先校验，再比较

文档记载的 SHA-1 范围响应由若干行组成，每行是 35 个字符的十六进制后缀、冒号、十进制出现次数。填充行的形状相同，计数为零 **[9]**。搜索之前先校验整个响应。curl 的 `--max-filesize` 选项让 curl 在响应超过上限时失败。其手册页说明，在没有声明可用大小时，curl 可能只在传输途中才发现大小，所以这是一道护栏，而不是文件系统配额 **[10]**。下载后的检查会拒绝任何仍然留下的空响应体或超大响应体。这些都属于应用层限制，不能替代文件系统配额或更广泛的资源控制。

```sh
if [ "${result:-}" !=
unknown ]; then
    bytes=$(wc -c < "$response" | tr -d ' ')
    [ "$bytes" -gt 0 ] && [ "$bytes" -le 1048576 ]
|| result=unknown
fi

if [ "${result:-}" != unknown ]; then
    if ! awk -F: '
        {
            sub(/\r$/, "")
            if (NF != 2 || length($1) != 35 ||
                $1 !~ /^[0-9A-F]+$/ || $2 !~ /^[0-9]+$/) {
                exit 1
            }
        }
        END { if (NR == 0) exit 1 }
    ' "$response"
    then
        result=unknown
    fi
fi

if [ "${result:-}" != unknown ]; then
    if ! count=$(printf '%s\n' "$suffix" | awk -F: '
        FILENAME == "-" {
            if (FNR != 1 || length($0) != 35 ||
                $0 !~ /^[0-9A-F]+$/) exit 2
            wanted=$0
            next
        }
        {
            sub(/\r$/, "")
            if ($1 == wanted && ($2 + 0) > 0) {
                print $2
                exit
            }
        }
        END { if (wanted == "") exit 2 }
    ' - "$response")
    then
        result=unknown
    elif [ -n "$count" ]; then
        result=exposed
    else
        result=not_found
    fi
fi

suffix=
count=
unset suffix count
```

shell 的 printf 内建命令通过管道把选定的后缀传给 awk。不要用 `awk -v wanted="$suffix"`，那会把由密码派生出的 140 位值放进进程参数表。比较进程失败按未知处理。只匹配计数为正的后缀，可以丢弃填充记录。确切的出现次数有意不写入日志。UI 可以选择显示它，但团队应当先判断这个数字是否会改变处置方式。通常不会。有一次观察结果，就足以拒绝或轮换候选密码；而非常大的计数反而会让人对语料库是否新鲜、来源是否可靠产生虚假的精确感。

这个解析器对大写输出要求严格，因为示例端点规定后缀为大写。其他提供方可能规定别的大小写规则或别的哈希族。应当按提供方公布的契约调整文法，而不是加入宽泛的容错转换。宽松的解析器会把 HTML 错误页、代理警告或部分损坏的响应变成错误的安全决策。

### 步骤 4：让失败有自己的状态

安全检查需要三种结果，而不是两种：

- `exposed`：本地后缀匹配到一条校验通过的计数为正的记录；
- `not_found`：取到响应，大小受控，完整校验通过，且不含正匹配；
- `unknown`：检查无法确定以上任何一种。

把退出码分开，脚本就不会把运维失败误当成安全结果：

```sh
if !
event_id=$("$UUIDGEN" -r); then
    echo "Unable to create an audit event ID. Treat the result as
unknown." >&2
    exit 20
fi

case "$result" in
    exposed)
        if ! "$LOGGER" -p auth.notice -t exposure-check \
            "event=$event_id outcome=exposed source=pwned-passwords-range client=freebsd-exposure-check/1.1"
        then
            echo "Audit logging failed. Treat the result as unknown." >&2
            exit 20
        fi
        echo "Match found. Do not set or reuse this password."
        exit 10
        ;;
    not_found)
        if ! "$LOGGER" -p auth.notice -t exposure-check \
            "event=$event_id outcome=not_found source=pwned-passwords-range client=freebsd-exposure-check/1.1"
        then
            echo "Audit logging failed. Treat the result as unknown." >&2
            exit 20
        fi
        echo "No match found in the checked corpus. This is not a safety guarantee."
        exit 0
        ;;
    *)
        "$LOGGER" -p auth.warning -t exposure-check \
            "event=$event_id outcome=unknown source=pwned-passwords-range client=freebsd-exposure-check/1.1" || true
        echo "Check unavailable or invalid. Treat the result as unknown." >&2
        exit 20
        ;;
esac
```

FreeBSD 的 **logger(1)** 是 syslog 的 shell 接口，它的设施和优先级可以通过 **syslog.conf(5)** 路由 **[6]**。上面的记录所证明的事实，比许多审计设计声称的要小：某个具名客户端在系统日志所记录的时间点记下了一个结果，并带有不透明的事件标识符。它不能以密码学方式证明检查了哪个密码、服务语料库是否正确，或者用户是否依据该结果采取行动。

这一限制是有益的。日志里记下前缀，就会保留关于候选密码的 20 位信息。记下完整摘要，会造出高效的离线猜测目标。记下摘要的确定性 HMAC，会带来关联能力，而且对后来拿到 HMAC 密钥的人来说，它又会变成校验器。如果受监管的工作流需要更强的证据，就把事件绑定到内部账户操作上：在应用审计系统里存入无关的应用事件 ID 和策略决定，例如 `password_change_rejected`。把机密派生材料挡在证据记录之外。

日志访问和保留期仍然重要。FreeBSD 的日志框架可以把本地消息路由到文件或远程收集器，它的 BSM 审计设施可以捕获细粒度的安全事件 **[11, 12]**。日志更多并不自动带来更多保障。要确定谁可以读取事件、如何轮转、证据在多长时间内仍然有用。没有存储和复查计划，就不要开启大批量审计。

### 缓存，但不在磁盘上造出密码预言机

同一前缀的所有密码共用范围响应，所以缓存可以减少重复的网络调用。它也改变了数据模型。像 `21BD1.txt` 这样的缓存文件名，记录了主机上有人请求过该前缀。响应里包含来自公开语料库的数百个后缀。两者都不是明文密码，但在小型环境里，或者与其他观察结合起来时，请求历史仍然可能是敏感的。

对交互式管理工具来说，最简单的安全策略是不做持久缓存。临时响应在严格的 `umask` 下创建，只用一次，由能响应信号的 trap 删除。对繁忙的密码设置服务，改用带短生存时间的内存缓存。如果确实需要持久化：

1. 把响应存在专用服务账户拥有的目录里，权限 0700；
2. 只用前缀文件名，路径里绝不放用户 ID 或账户名；
3. 设置较短且有文档记载的过期时间；
4. 用与网络内容相同的文法校验缓存内容；
5. 获取成功后原子替换文件；
6. 绝不存储选定的后缀、完整摘要、候选密码或用户级匹配结果；
7. 在监控里包含缓存年龄，但除非策略规定了最大年龄，否则不要把它放进面向用户的通过或拒绝决定。

整个公开语料库的离线副本可以消除向远程服务的查询披露，但会引入一个庞大的数据集，必须下载、更新、校验、存储、搜索。总体上看，它并不自动更私密。正确的选择取决于查询量、出站策略、存储控制、更新纪律，还有本地请求元数据有多敏感。

### Jail、Capsicum、rc.d：选对边界

FreeBSD Jail 虚拟化文件系统、用户和网络访问，可以围绕长期运行的集成提供一层有用的边界 **[13]**。它不能让不安全的客户端变安全，对于单个管理员把候选密码敲进短命本地进程的场景，也没有必要。价值最高的控制仍然简单：不要让机密经由参数传递，不要记录机密派生值，约束出站流量，失败即拒绝。

当泄露检查成为组织级服务的一部分时，Jail 就变得合理。用专用的非特权账户运行队列消费者。只给 Jail 它需要的文件。需要独立的出站控制时，使用 VNET Jail 或主机防火墙规则。FreeBSD 基本系统包含 PF、IPFW、IPFILTER，手册说明了如何用防火墙规则控制入站和出站流量 **[14]**。把工作进程限制在策略要求的 DNS 和 HTTPS 目标上，同时为端点地址变化和证书验证做好安排。

Capsicum 是专用客户端的另一个选择。FreeBSD 把 Capsicum 描述为能力与沙箱框架，在打开所需的描述符之后限制对全局命名空间的访问 **[8]**。编译版检查器可以先打开终端、信任库、解析器通道、日志套接字、输出目标，然后在解析不可信响应数据之前进入能力模式。这种设计需要仔细拆分，因为名称解析和新建出站连接通常涉及全局命名空间。它是值得采取的下一步，光靠 shell 脚本里拨个开关做不到。

同样的克制适用于 rc.d。FreeBSD 的 rc.d 框架通过使用 **/etc/rc.subr** 的小型 **sh(1)** 脚本来启动和管理服务 **[15]**。交互式检查器不是守护进程，不应在启动时运行。如果应用使用常驻队列工作进程，就把该工作进程装到 **/usr/local/libexec** 下，在 **/usr/local/etc/rc.d** 下放一个常规控制脚本，默认禁用，并通过 `rc.subr` 调用 `run_rc_command`。把密码输入留在应用边界之内。rc.d 脚本只应管理该工作进程的生命周期，绝不能包含凭证或候选密码值。

### 可复现的测试计划

把纯解析和决策逻辑与机密输入分开测试。开发时不要使用真实账户密码。使用有意公开的测试数据，或者随机生成的合成值，这些值从不指派给任何账户。

#### 1. 已知命中

使用公开记录的演示字符串，而不是私人凭证。确认本地 SHA-1 是 40 个十六进制字符，请求只包含其中五个，校验通过的响应中含有计数为正的匹配后缀，程序以 10 退出，审计记录不含任何机密派生字段。

#### 2. 预期未命中

在本地生成一个很长的随机合成值，检查一次，预期得到 `not_found`。任何值理论上都可能出现在语料库里，所以这个断言应当表述为预期测试结果，并在单元测试中绑定到一份捕获且校验过的响应。实时集成测试可能随时间变化。

#### 3. 填充的零计数条目

给解析器喂一个有效的 35 字符后缀，计数为零。确认它被忽略。然后在另一份固定数据里放入同一后缀但计数为正，确认它能匹配。

#### 4. 格式错误的响应体

测试空文件、HTML 页面、长度错误的后缀、非十六进制字符、缺少冒号、负数计数、多余字段、超过配置上限的响应体。每种情况都必须产生 `unknown` 并以 20 退出。

#### 5. 网络与 TLS 失败

测试连接被拒、DNS 失败、超时、受控测试环境中的不可信证书、指向 HTTP URL 的 HTTPS 重定向。任何一种都不得产生 `not_found`。确认向用户显示的错误不含摘要或响应体。

#### 6. 终端中断

输入候选密码时按 Control-C，请求尚未返回时也按一次。确认回显恢复、临时文件删除。在测试 shell 中发送 HUP 和 TERM，再重复一次检查。

#### 7. 审计复查

检查配置的日志目标。可接受的记录格式应当只包含事件 ID、结果、来源标识符、由日志路径提供的系统时间，也许还有客户端版本。在测试日志中搜索候选密码、摘要、前缀、后缀、出现次数、响应行。所有搜索都必须无结果。再用合成候选密码检查子进程参数和导出的变量：候选密码、完整摘要、选定后缀都不应出现。curl 请求 URL 中的五个字符前缀仍可被本地进程检查看到；这属于有界披露的一部分，并不承诺对主机隐藏请求。

#### 8. 并发与限流

如果集成进服务，就启动并发检查，确认共享限流器限制的是总流量，而不只是每个进程的流量。确认缓存替换是原子的，刷新失败后要么留下仍然有效的旧条目，要么明确进入未知状态。

编辑或审稿人应当能够在不调用实时服务的情况下复现解析器测试。把小型合成响应固定数据放进源码树，把网络集成测试设为可选。这样的拆分让日常测试既快又确定，也不会滥用公开端点。

### 运维策略比查询本身更重要

范围检查做得再好，也可能支撑糟糕的安全策略。这项控制应当放在密码创建和修改环节，也就是候选密码提交之前。发现命中时，拒绝该候选密码，并说明它出现在已知语料库里。不要透露泄露记录，也不要暗示用户的具体账户已被入侵。把这项控制与账户工作流的速率限制、多因素认证、安全恢复、会话复查、账户接管信号监控配合起来。

组织获得授权后使用时，要明确界定范围。检查你自己身份验证工作流中提供的候选密码，或者组织在法律上有权管理的资产。不要收集无关用户的密码，不要测试第三方账户，也不要把范围端点变成枚举工具。

定期复查端点契约。前缀长度、填充行为、响应文法、可接受使用规则、客户端要求都可能变化。把假设写死在代码里又不监控文档，会形成安静的失效模式。在审计事件里记录客户端版本，软件包升级后做测试，遇到意外响应就停下来，而不是放宽校验。

### 结论

保护隐私的凭证泄露检查，靠的不是把密码哈希后调用 API。它来自完整的数据路径：候选密码如何进入进程，什么离开主机，网络请求如何受约束，不可信数据如何解析，缓存了什么，存在哪些结果，还有审计线索拒绝保留什么。

在 FreeBSD 上，小型实现可以用上熟悉的系统组件。**sha1(1)** 提供与语料库兼容的查询摘要，**mktemp(1)** 创建私有响应文件，**logger(1)** 记录最小事件，软件包系统提供 curl 以发起带填充的 HTTPS 请求。当检查器成长为一套服务时，Jail、防火墙、Capsicum、rc.d 就会派上用场，但它们应当强化健全的协议，而不是弥补带有机密的参数或日志。

最重要的结果是 `unknown`。能区分不确定与不存在的系统，失败时才安全。把超时和解析错误合并成“not found”的系统，最终会告诉别人某个未检查的密码是干净的。保留这第三种状态，只披露有界前缀，不留下可复用的校验器，就能把一次巧妙的范围查询变成可审计的安全控制。

### 参考文献

1. The FreeBSD Project，“发布信息”。 <https://www.freebsd.org/releases/>
2. The FreeBSD Project，“FreeBSD 安全信息：受支持的 FreeBSD 版本”。 <https://www.freebsd.org/security/>
3. The FreeBSD Project，**md5(1)**，其中包含 **sha1(1)**。 <https://man.freebsd.org/cgi/man.cgi?query=md5&sektion=1>
4. The FreeBSD Project，**fetch(1)**。 <https://man.freebsd.org/cgi/man.cgi?format=html&query=fetch&sektion=1>
5. The FreeBSD Documentation Project，“安装应用：软件包与 Ports”。 <https://docs.freebsd.org/en/books/handbook/ports/>
6. The FreeBSD Project，**logger(1)**。 <https://man.freebsd.org/cgi/man.cgi?format=html&query=logger&sektion=1>
7. The FreeBSD Project，**explicit_bzero(3)**。 <https://man.freebsd.org/cgi/man.cgi?format=html&query=explicit_bzero&sektion=3>
8. The FreeBSD Documentation Project，“安全：Capsicum”。 <https://docs.freebsd.org/en/books/handbook/security/#capsicum>
9. Have I Been Pwned，“API 文档：Pwned Passwords”。 <https://haveibeenpwned.com/API/v3>
10. The curl Project，“**curl(1)** 手册”。 <https://curl.se/docs/manpage.html>
11. The FreeBSD Documentation Project，“配置、服务、日志与电源管理：配置系统日志”。 <https://docs.freebsd.org/en/books/handbook/config/#configtuning-syslog>
12. The FreeBSD Documentation Project，“安全事件审计”。 <https://docs.freebsd.org/en/books/handbook/audit/>
13. The FreeBSD Documentation Project，“Jail 与容器”。 <https://docs.freebsd.org/en/books/handbook/jails/>
14. The FreeBSD Documentation Project，“防火墙”。 <https://docs.freebsd.org/en/books/handbook/firewalls/>
15. The FreeBSD Documentation Project，“BSD 中的实用 rc.d 脚本编写”。 <https://docs.freebsd.org/en/articles/rc-scripting/>

---

### 利益披露

作者隶属于 CyberVisir Solutions Ltd.，该公司运营 LeakData.io。本工作流与产品无关；提及公司是为了说明作者的隶属关系。文中使用公开文档和合成测试数据，没有复现任何私有泄露记录或真实凭证。AI 工具协助了研究组织、语言编辑和代码复查。最初的实现在 FreeBSD 15.1-RELEASE 上验证过；九月修订版加入了上述澄清和对实现的修正。

---

**Emre Çapan** 是 CyberVisir Solutions Ltd. 旗下 LeakData（<https://leakdata.io>）的联合创始人兼 CTO。他的工作聚焦于数字风险监控、泄露事件分级、注重隐私的安全工作流，还有把不确定的安全信号变成可落地的处置步骤。
