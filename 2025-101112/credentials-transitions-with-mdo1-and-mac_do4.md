# 使用 mdo(1) 与 mac_do(4) 进行凭证转换

- 原文：[Credentials Transitions with mdo(1) and mac_do(4)](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/)
- 作者：Olivier Certner

在本文中，我们探讨如何使用 mdo(1) 程序，轻松且快速地以不同的凭据启动新进程，以及系统管理员如何通过利用内核模块 mac_do(4)，使非特权用户能够发起凭据转换，从而在简单的基于角色的场景中，无需安装诸如 sudo(8) 或 doas(1) 等第三方程序。

传统的 UNIX 访问控制方法，本质上依赖于以下概念和组件：

- 用户与组。组旨在通过在某些方面对组内所有用户进行统一处理，从而简化管理。
- 进程，作为代表某个用户和组行事的主体，这些用户和组被称为其凭据。
- 文件所有权（一个用户、一个组）与权限，它们分别控制所有者、文件所属组的成员以及其他用户的访问。
- 特殊的 root 用户[1](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor003)，其拥有全部特权，尤其是不受访问控制约束。
- set-user-ID / set-group-ID 可执行文件，这类程序在启动时，其进程会分别将可执行文件的所有者作为用户、将可执行文件的组作为“主”组加以认可[2](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor000)。

系统管理员的一项主要职责，是为其用户提供对系统各类资源的适当访问权限。在大多数情况下，这意味着定义用户和组，并确保文件权限符合预期的安全策略。

UNIX 访问控制模型具有这样的灵活性：用户不必对应真实的、具体的人，而也可以表示角色，由多个需要访问特定资源和信息的真实用户来扮演。事实上，在除最简单的文件共享场景之外的所有情况下，这种依赖 UNIX 用户而不仅仅是组的基于角色的方法都是必要的。这使得临时采用另一组凭据——即基于目标用户建立的凭据——成为系统的一项重要功能，而这一功能传统上由程序 su(1) 来完成。

然而，su(1) 在切换到新用户的凭据之前，需要对该用户进行身份验证，通常是要求输入该用户的密码[3](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor005)。这对于已经被分配了角色且本身已经完成认证的人类用户来说并不方便，对于自动化场景而言同样如此。它还必然会启动目标用户的 shell，这使其无法用于那些没有有效登录 shell 的用户，而这正是角色用户通常所期望的设置——任何人都不应当能够直接以其身份登录。此外，它还使得以指定参数启动某个特定程序变得比应有的更加繁琐[4](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor006)。

为克服这些限制，系统管理员通常会安装其他用于代表其他用户运行命令的程序，例如 sudo(8) 或 doas(1)。然而，像 sudo(8) 这样的程序具有不可忽视的攻击面，部分原因在于其包含大量不常用的功能，尤其是其模块化设计，从安全角度看可能是危险的。更一般地说，安装可执行文件所有者为 root 且设置了 set-user-ID 模式位的程序，本身就是一项安全隐患，因为一旦这些程序被攻破，就可能通过以 root 用户身份执行代码而获得完整的管理权限。但传统的 UNIX 并未提供其他更改凭据的方式，这也是 su(1) 和 login(1) 等程序必须以这种方式安装的原因。

作为设置了 set-user-ID 模式位的可执行文件（通常称为“setuid 可执行文件”）的替代方案，我们提供了内核模块 mac_do(4)，它构建于 FreeBSD 的 MAC 框架之上[5](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor007)。其目的在于仅允许来自非特权进程的特定凭据转换，从而无需将相应的可执行镜像安装为“setuid”。

mdo(1) 是 mac_do(4) 的配套程序，负责向内核实际请求所需的凭据转换。mdo(1) 可以由拥有全部特权的 root 用户单独使用；否则，其请求将根据管理员的配置，由内核模块 mac_do(4) 进行审核。

在本文中，我们首先通过一系列示例，说明如何使用 mdo(1) 在新的凭据下启动命令。随后，我们解释如何配置 mac_do(4)，以在宿主系统以及 jail 中启用对特定凭据转换的支持，从而实现基于角色的方案，并对当前设计提供一些见解。最后，我们征求用户反馈，了解短期内应当提供的内容，以及可能的长期未来规划。

## 使用 mdo(1)

mdo(1) 的设计目标，是以任意一组凭据运行任意命令。如果你尚未配置 mac_do(4)（将在下节介绍），仍然可以以 root 身份运行下面的所有示例。对于大多数示例，需要 FreeBSD 15.0，因为 FreeBSD 14.3 中的 mdo(1) 仅支持选项 `-u` 和 `-i` [6](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor008)。

出于安全原因，目标进程的凭据必须被完整指定：要么通过显式列出所有用户和组及其所请求的值，要么通过建立一个基线，为每一项提供默认值，然后再通过附加选项进行修订。

在基于角色的设置中，最常见的使用场景，可以说是认可某个用户的凭据，就好像该用户刚刚登录一样。因此，mdo(1) 以尽可能简单的形式支持这一点，唯一需要的选项是 `-u`（表示“user”），用于基于指定用户建立一个基线，例如：

```sh
$ mdo -u www /usr/local/bin/occ
```

对于那些 sudo(8) 和 doas(1) 用户来说，这条命令行看起来应该与你们使用这些工具时的方式极为相似：基本上，mdo 取代了 sudo 或 doas，其余部分完全一致。

显然，如果传递的是某个用户的数值 ID 而不是用户名，这种方式就无法奏效，因为完整的登录凭据是由密码数据库和组数据库确定的，而这些数据库是按名称进行索引的[7](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor009)。使用 `-u` 并指定用户数值 ID 时，只会指定用户本身（实际上是实际、有效以及保存的用户 ID），此时 mdo(1) 还需要被告知所有目标组。这要么像下面将看到的那样显式指定，要么通过 `-i` 来完成，`-i` 表示以当前的组作为基线；否则，mdo(1) 将直接报错。

保留当前的组，例如：

```sh
$ mdo -u 10002 -i
```

这在某些情况下会很有用，例如，当你需要临时处理一些“外来”的文件，而其所有者并不存在于你的密码数据库中时。

`-i` 也可以与使用用户名的 `-u` 一起使用，且含义相同。例如，如果 foo 是你密码数据库中的某个用户，其 ID 为 10002，那么下面这条命令与前一个示例完全等价：

```sh
$ mdo -u foo -i
```

假设现在你想显式指定组，无论是因为你像上文那样使用了数值用户 ID，还是因为你想覆盖某个用户关联的组。你可以使用以下选项：

- `-g`：设置或覆盖主组[8](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor010)。
- `-G`：设置或覆盖完整的附加组集合。你在此提供的以逗号分隔的列表被视为完整列表，即应包含所有附加组。请注意，从 FreeBSD 15 开始，用户登录时，其初始组（在密码数据库中指定）也会包含在进程的附加组集合中。
- `-s`：修改附加组集合。此选项的参数由一系列逗号分隔的指令组成。使用 `+` 指令可以确保某组包含在附加组中，使用 `–` 指令可以确保某组不在附加组中，使用 `@` 指令可以重置列表，使 `-s` 的功能类似于 `-G`，但语法不同[9](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor011)。

一些示例：

```sh
$ mdo -u unprivileged_user -g wheel -G wheel,staff,operator
```

该命令会以 unprivileged_user 用户启动一个 shell，但会忽略密码数据库和组数据库中指定的组，而是用传入的组来替换它们。`-g` 和 `-G` 对于测试某个用户在特定组集合下将获得哪些权限非常有用。

然而，如果目的只是以某个用户身份登录，并额外附加一些组（例如 wheel 和 operator），即所谓“增强”的角色场景，那么应当使用选项 `-s`：

```sh
$ mdo -u unprivileged_user -s +wheel
```

或者，相反地，如果某个组的成员资格需要被临时撤销，例如当该组被用作通过 ugidfw(8)（以及 mac_bsdextended(4)）进行访问控制的标记时：

```sh
$ mdo -u unprivileged_user -s -tag_group
```

如果用户本身不需要改变，就没有必要用 `-u` 显式指定。相反，可以直接使用 `-k`（表示“keep”），其含义是以当前所有用户和组作为基线。`-k` 与 `-u` 互斥，并且隐含 `-i`（从当前组开始）。

需要注意的是，在所有情况下，都可以使用前面提到的显式选项（`-g`、`-G` 和 `-s`）来覆盖凭据中的组设置。

最后，如有需要，还可以分别覆盖用户和主组的 real、effective 和 saved 变体，分别使用 –ruid、–euid、–svuid，以及 –rgid、–egid 和 –svgid。当这三种变体都被指定时，就不再需要分别使用 `-u` 或 `-g`，尽管在使用用户名时指定 `-u` 仍然有助于获取其关联的组。

可以看到，除了在最常见使用场景下的简洁性之外，mdo(1) 相较于 su(1)、sudo(8) 或 doas(1) 的优势在于，它允许对目标凭据的各个方面进行精确控制。这使其成为在修改密码数据库和组数据库之前，用于测试或临时使用任意凭据的首选工具，或者用于这样一种基于角色的设置：角色的认可来源于加入额外的组，而不是切换用户。

mdo(1) 仅关注于更改凭据。因此，其代码相对简单，并且在设计时特别强调尽可能清晰和最小化，同时保持“显然正确”。这使得程序易于审计，并生成一个非常小的二进制文件，在我的 stable/14 机器上仅略超过 7kB。相比之下，doas(1) 的体积略超 27kB，而 sudo(8) 为 229kB，加上其默认安全策略插件 sudoers(5) 为 628kB，两者都是以“setuid 可执行文件”形式安装，这与 mdo(1) 形成对比。

目前，由于 mdo(1) 针对基于角色的场景，它在请求新凭据时不要求输入任何密码或其他形式的认证，而是完全依赖请求者自身的凭据。作为未来可能的发展方向之一（在本文结论中提及），我们可能会增加支持要求当前已登录用户输入密码的功能。根据用户反馈，还可能考虑增加与切换到其他用户相关的附加功能，例如登录类（login class）、登录名、调度优先级等。

## 配置 mac_do(4)

对于非 root 用户要能够使用 mdo(1)，必须配置 mac_do(4)，因为 mdo(1) 设计上并未以“setuid”方式安装。

mac_do(4) 默认不会编译进内核，但可以很容易地以模块方式加载：

```sh
# kldload mac_do
```

然后，你可以通过 sysctl(8) 项 security.mac.do 访问其参数。目前（FreeBSD 14.3 和 15.0）可用的参数如下：

- enabled：模块是否启用（默认值为 true）。这是一个全局开关。也可以通过规则（下一个选项）或 jail 参数（见下文对应小节）在宿主系统或任意 jail 中选择性地停用 mac_do(4)。
- rules：规则列表，用于指示允许哪些凭据转换。我们将在下一小节中研究多个示例。rules 默认值为空，意味着 mac_do(4) 本身不会允许任何凭据更改。
- print_parse_error：当设置规则失败时，是否在控制台和系统日志中打印解析错误。

下面我们先通过示例说明规则，然后再讲如何配置 jail。

## 规则

结合上文给出的 mdo 示例，我们这里授权用户 unprivileged_user（UID 10001）认可 www 用户（UID 80），代表网站管理员角色：

```sh
# sysctl security.mac.do.rules='uid=10001>uid=80,gid=80,+gid=80'
```

在这个示例中，只有一条规则。规则的 `>` 符号用于分隔两部分，左边是“from”部分，也称为“match”，右边是“to”部分，也称为“target”。历史上使用 `:` 作为分隔符，现在仍然可用，但我们认为 `>` 更易读，尤其对于习惯 UNIX 的用户来说，容易将 `:` 误解为类似元素之间的列表分隔符。由于 `>` 是 shell 特殊字符，因此需要以某种方式进行引用。为简便起见，我们建议总是将传给 sysctl(8) 的值加引号。两个 token 之间可以使用任意数量的空格，这对人工阅读也有帮助，同时也需要 shell 引号以确保正确解析。

“from”部分（上述规则中的 uid=10001）非常直接，用于匹配用户 ID[10](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor012) 为 10001 的进程，从而匹配 unprivileged_user（以及可能其他具有相同用户 ID 的用户）。注意，这里只能使用数值 ID，不能使用用户名。内核确实不识别用户名，从凭据角度看无关紧要。

“to”部分（`uid=80,gid=80,+gid=80`）稍微复杂一些。它包含三条由逗号分隔的子条款。`uid=80` 和 `gid=80` 非常直观：允许在用户 ID 和初始（“主”）组 ID 方面切换到 `www`。最后一条子条款 `+gid=80` 与附加组有关，表示附加组 ID 80 是允许的，但不是强制的。通常，带标志的 gid（这里是 `+`）应用于附加组。其他可能的标志包括 `!` 和 `–`，将在下面示例中说明。

这样的规则允许例如上一节中看到的示例命令，由 unprivileged_user 执行：

```sh
$ mdo -u www /usr/local/bin/occ
```

需要注意的是，`uid=10001>uid=80,gid=80,+gid=80` 这条规则相当严格，例如，如果用户 www 还属于除 www 之外的其他组，`mdo -u www` 就无法成功执行，因为 `mdo -u www` 会尝试安装密码[11](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor013)和组数据库中要求的附加组，而该其他组未出现在规则中。

它同样禁止例如 `mdo -u www -i` 的操作，即切换到用户 www 但保留当前组（假设这些组是与 unprivileged_user 关联的，且中间未被更改）。如果管理员希望这种操作可以生效，就需要放宽对组的检查。假设 unprivileged_user 仅属于一个与其同名且 GID 为 10001 的组，则可以使用：

```sh
# sysctl security.mac.do.rules='uid=10001>uid=80,gid=80,gid=10001,+gid=80,+gid=10001'
```

从这个示例中你大概可以推断出，指定多个目标子条款并使用 `gid` 和 `+gid`，意味着目标凭据中可以存在任意一个指定的组[12](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor014)。

除了前面提到的两个 mdo(1) 使用场景外，这条规则还允许 unprivileged_user 在切换到 www 的同时，同时认可组 80 和 10001[29](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor031)。如果完全不希望出现这种情况，则可以使用以下设置替代：

```sh
# sysctl security.mac.do.rules='uid=10001>uid=80,gid=80,+gid=80;uid=10001>uid=80,gid=10001,+gid=10001'
```

这一次，有两条规则用 `;` 分隔。当存在多条规则时，只要其中一条验证通过，该转换就被允许[13](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor015)。这种设置仍然允许 `mdo -u www` 和 `mdo -u www -i` 正常工作，同时排除了像 `mdo -u www -i -s +www` 或 `mdo -u www -g 10001` 这样的操作。

如果出于某种原因，即使当前组与 unprivileged_user 的数据库信息不符，也希望 `mdo -u www -i` 能生效，可以改用：


```sh
# sysctl security.mac.do.rules='uid=10001>uid=80,gid=80,+gid=80;uid=10001>uid=80'
```

上面第二条规则 `uid=10001>uid=80` 允许在不改变当前组的情况下切换用户 ID，因此非常适合在使用 mdo(1) 的 -i 选项时保留任意当前组。实际上，这条规则是 `uid=10001>uid=80,gid=.,!gid=.` 的简写，其中 . 在 gid 的情况下表示当前主组，在 !gid 的情况下表示当前附加组，更广泛地说，gid 前带其他标志时亦同。需要注意的是，这个默认部分 gid=.,!gid=. 仅在没有目标子条款以 gid 为类型（无论是否带标志）时才被隐含使用。特别地，以下规则 `uid=10001>uid=80,gid=.` 会阻止任何未删除所有附加组的切换，因为规则中没有带标志的 gid 子条款。

另一个 gid 标志 – 可用于表示某个组不应包含在最终附加组中。乍一看可能会觉得奇怪，因为允许的组必须在规则中显式指定（除了上一段解释的默认情况）。实际上，这在与 `.` 配合 `+gid` 或 `!gid` 使用时非常有用，可用于排除当前的一些组。例如，如果希望允许 unprivileged_user 切换到用户 www 并保留其当前组，同时确保 wheel 不出现在最终附加组中，则可以使用：

````sh
uid=10001>uid=80,gid=.,+gid=.,-gid=0
````

[14](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor016)

最后，在规则中，你可以用 `*` 或 `any` 来表示任意可能的用户或组 ID。例如，如果你确实希望允许 wheel 组的成员切换为 root，可以使用如下规则：

```sh
gid=0>uid=0,gid=*,+gid=*
```

基本上表示任意一组目标组都可以。进一步，如果你不想强制必须先切换到 root 再成为其他用户，也可以使用：

```sh
gid=0>uid=*,gid=*,+gid=*
```

这可以简写为：

```sh
gid=0>any
```

需要提醒的是，目前 mdo(1) 面向基于角色的方案，因此在任何情况下，即使目标用户是 root，也不会要求输入密码来切换用户。

我们刚刚展示了 mac_do(4) 规则提供的丰富实用可能性，如你所见，它们非常灵活，能够精确表达允许的目标凭据[15](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor017)。在设计时，我们努力保持语法尽可能易于理解，同时受限于 sysctl(8) 值本质上为单行的特性，这要求语法简洁、表达能力足够，并且内核只处理数值 ID，不访问密码和组数据库。即便你一开始不完全理解 security.mac.do.rules 的某个具体设置，也不必担心，稍加研究很快就能掌握，因此不要被示例淹没，根据需要花时间学习即可。

规则的更完整和正式的规范，请参见 mac_do(4) 手册页。

## Jail

FreeBSD 中的 Jail 形成一个层级结构[16](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor018)，其顶层是宿主系统[17](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor019)。每个单独的 jail 都有参数，其中一些只能在创建时设置，另一些则可以在 jail 运行时从外部修改。

mac_do(4) 支持每个 jail 的配置，通过以下参数实现：

- mac.do：每个 jail 的模块模式。
- mac.do.rules：适用于该 jail 的规则。

参数 mac.do.rules 包含适用规则，其格式与上一节中看到的 sysctl(8) 项 security.mac.do.rules  完全相同。

通常，希望能够从 jail 外部控制安全参数，这实际上也是创建 jail 时唯一可行的方法。不过，也希望 jail 的行为尽可能接近宿主系统。由于 mac_do(4) 是管理员用来授权凭据转换的工具，因此 jail 中的管理员也应能使用它。

为此，sysctl(8) 项 security.mac.do.rules 被设计为感知 jail，即它反映当前 jail 的设置，并且可以从 jail 内部进行设置。jail 内的 security.mac.do.rules 与对应 jail 的 mac.do.rules 参数实际上是同一个变量，因此它们的值始终一致。外部修改 mac.do.rules 会立即在 jail 内生效，反之，从 jail 内读取参数也会反映对 security.mac.do.rules 的任何内部修改。

参数 mac.do 指示 mac_do(4) 在 jail 中的工作方式。对于支持 jail 的模块的主控参数来说，通常接受或报告以下值：

- new：jail 的配置独立于父 jail。
- inherit：jail 的配置继承自父 jail。
- disable：在该 jail 中禁用 mac_do(4)。

出于显而易见的安全原因，默认值为 disable，除非显式设置了 mac.do.rules。

你可能会想，这个参数与 mac.do.rules 的具体交互关系如何，因为两者似乎有些冗余。如本节开头所述，将 rules 设置为空字符串会导致 mac_do(4) 忽略凭据更改请求，并且由于 rules 是每个 jail 独立的，这也可以作为每个 jail 的开关来禁用 mac_do(4)，类似于 disable 的作用。反之，从 jail 外部设置 mac.do.rules，或在 jail 内部设置 security.mac.do.rules，总是会建立每个 jail 的设置，从概念上对应于 new。

我们引入[18](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor020) mac.do jail 参数有两个原因。首先，大多数支持 jail 的内核模块都会提供一个单一的开关来启用或禁用其在 jail 内的功能，我们认为设置这样一个开关既有利于系统一致性，也提供了比将 rules 设置为空字符串更自然的方式来禁用 mac_do(4)。其次，它提供了引入新的继承模式的机会，即 inherit 值，这对于希望一组 jail 行为一致的管理员非常有用。

在探讨继承的具体含义之前，先看看 mac.do 与 mac.do.rules 如何保持一致。内部上，每个 jail 都有一个标志，用于指示它是否继承自父 jail；如果不继承，则会保存一份规则设置（mac.do.rules）及其内部表示，从而避免信息冗余[19](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor021)。实际上，我们并不存储任何直接对应 mac.do 参数的值。该参数在读取时是根据现有数据生成的。由此可知，当继承标志被设置时，读取 mac.do 返回 inherit；否则，如果没有指定规则（空字符串）返回 disable；否则返回 new。在显式设置 mac.do 时，mac_do(4) 会检查其值是否与 mac.do.rules 一致。如果 mac.do 设置为 new，则必须指定 mac.do.rules。对于其他情况，我们应用鲁棒性原则[20](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor022)，即使严格来说 jail 参数中不应存在空字符串的 mac.do.rules，也会容忍其存在。

当 mac.do 设置为 inherit 时，mac_do(4) 会直接使用适用于父 jail 的配置，而父 jail 本身可能继承自更高层的 jail。主要结果是，对父 jail 中任意规则的更改（直到第一个不继承的父 jail）会自动且立即在继承的 jail 中生效。这减轻了管理员在树状结构中保持多个 jail 配置同步时的工作量。如前所述，在 jail 上显式设置规则（无论通过 mac.do.rules 还是 security.mac.do.rules）会建立独立的 per-jail 配置，有效地打破继承。之后随时可以重新启用继承，只需再次将 mac.do 设置为 inherit。

与其他 jail 参数一样，你可以在创建 jail 时轻松使用这些参数进行配置，例如直接在 jail(8) 命令行上：

```sh
jail -c name=test_jail path=/ mac.do=inherit
```

或者通过 jail.conf(5) 配置。在 jail 运行时修改某些参数，可照常使用 jail -m，例如：

```sh
jail -m name=test_jail mac.do=disable
```

## 启动配置

由于宿主系统上的 mac_do(4) 配置是通过 sysctl(8) knob（同时也是可调参数）完成的，因此可以使用基本系统提供的两种机制之一，在启动时进行设置。

第一种方式是调整 loader.conf(5) 配置，添加如下行：

```sh
security.mac.do.rules='uid=10001>uid=80,gid=80,+gid=80;uid=10001>uid=80'
```

这种方式适用于规则必须在启动早期就可用的情况，或者例如你没有使用基本系统的 rc(8) 启动框架时。

否则，你也可以将完全相同的行添加到 sysctl.conf(5) 中，当 rc(8) 执行时，sysctl(8) knob 会相应地被设置[21](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor023)。

我们收到一些有限反馈，有少数人觉得 mac_do(4) 仅处理数值 ID 并且 sysctl(8) knob 语法过于简洁，不够实用。这本质上是由于将转换规则放在内核中，以应对某些用户态组件可能被破坏的强威胁模型。然而，我们理解大多数人并不需要如此高的安全级别，从密码和组数据库内容生成最终规则的用户态工具对他们可能更有用。曾有一个方向的提议，设计专用可执行文件和配置文件来支持 mac_do(8)，目前处于搁置状态，因为我们仍在反思整体设计，包括如何组织可执行文件、未来可能的配置文件以及如何避免冲突。如果更多人对此功能感兴趣，这一方向可能会尽快取得进展。

另见本文最后一节，我们计划在 mdo(1) 中添加的短期功能，其中之一将弥补这一差距的重要部分。

## 关于 mac_do(4) 设计的一些说明

Baptiste Daroussin 最初启动 mac_do(4)/mdo(1) 项目的目标，是在不使用“setuid 可执行文件”的情况下实现基于角色的凭据转换。在高安全性、规范严格的环境中，即便可以安装这些可执行文件，也可能需要经历冗长复杂的安全审计，而且随着可执行文件的升级，这些审计通常需要重新进行。因此，mac_do(4) 被设计为基于内核的替代方案，借助 MAC 框架[5]，能够授权非特权进程成功更改凭据。除了减少对“setuid 可执行文件”的依赖，这种架构还能立即降低凭据更改程序被攻击或出现编程漏洞时的潜在影响。

mac_do(4) 的最初实现仅监控 setuid() 系统调用，根据匹配原始用户和目标用户的规则授权特定调用。为了让 mdo(1) 按目标用户的密码和组数据库修改组信息，mac_do(4) 必须接受任何 setgroups() 和 setgid() 系统调用。为了避免任意程序独立利用这些调用，mac_do(4) 仅授权来自 mdo(1) 程序生成的进程提出的凭据转换请求。

因为允许 setgroups() 和 setgid() 的任意请求，会严重削弱减少攻击或漏洞影响的效果，我们修改了 mac_do(4)，对完整的凭据转换进行验证，并通过规则指定哪些组可以出现在最终凭据中。

验证或拒绝完整转换从根本上要求原子性，这意味着需要对传统 UNIX 的安全 API 进行修改。一种自然的方法是为其添加事务模式，在该模式下连续修改凭据的调用不会立即生效，而是累积这些更改，最终在“提交”时一次性应用。这种方法在一定程度上可以简化对现有程序的修改，以及可能的凭据属性扩展，但被认为对内核代码的侵入性较大，并且改变了现有系统调用 MAC 钩子的范式[22](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor024)。因此，我们采用了另一种方案：新增独立系统调用 setcred(2)，能够一次性设置所有凭据属性。这些属性通过一个结构体传递，并可根据需要通过标志进行扩展或版本控制。新增的 MAC 钩子会传入当前凭据和请求的凭据，使 mac_do(4) 能一次性看到当前状态和目标状态，并据此做出决策。

即便在这些修改之后，我们仍保留了 mac_do(4) 只能授权由 mdo(1) 可执行文件生成的进程的限制，因为这可能允许在 mdo(1) 内实现额外的转换限制。2025 年 Google Summer of Code（GSoC 2025）的一名学生 Kushagra Srivastava 负责包括引入对该限制的可配置性，使管理员能够指定 mac_do(4) 可以授权哪些可执行文件。

有些人可能会觉得，将根据规则检查并决定凭据转换的代码移入内核而不是在用户态执行，这一做法显得奇怪，甚至可能构成安全隐患。诚然，一旦内核被攻破，其后果无疑比“setuid executables”更加灾难性，但我们认为，前者发生的可能性远低于后者，原因如下。

首先，mac_do(4) 所接受的规则是完全明确定义的、自包含的，解析相对简单，也希望是容易理解的。

其次，执行凭据变更的“setuid executables”通常涉及的组件远多于 mac_do(4) 实际使用的部分。后者本质上只依赖 MAC framework 以及 jail 和 OSD 子系统，这些组件被广泛使用和测试，且不常发生频繁或深层次的变更；而前者则依赖用于读取密码和组数据库的库，这些库可能涉及网络访问，还包括用户态配置解析器，以及用于建立新会话所有特征（包括凭据）的代码，这些代码有时甚至属于独立的库，更不用说常见的用户态支撑代码，例如动态链接器。

第三，我们在设计和编写 mac_do(4) 时格外注意清晰性与简洁性，特别关注底层子系统的约束，并确保所依赖的部分不会在不被察觉的情况下被修改，通过断言进行监控。结果是，尽管进行了大量测试，我们至今尚未在 mac_do(4) 核心功能中发现漏洞（这可能是著名的“最后一句话”）。在收到的少量错误报告中，仅有两个在实际场景下被证实为真实问题，而这些场景起初确实未充分考虑或测试[23](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor025)，这促使我们对代码进行了再次审计。我们的 GSoC 学生还被委派开发自动化测试，这些测试将在未来几周内加入官方源代码树。它们将作为额外保障，并在 mac_do(4) 及其依赖子系统演进时帮助维护代码质量。

## 展望未来

这里的核心信息是，尽管我们有一些简单的短期计划，以及更宽泛的长期设想，未来方向在很大程度上将取决于当前或潜在用户的反馈。我们渴望听取对小改进或全新功能的建议，无论您是已经在使用 mac_do(4)/mdo(1)、计划使用，还是希望使用但现有功能无法满足需求。这将帮助我们在保持整体设计合理性的前提下，选择优先开发的内容。即便仅仅告知您正在使用这些工具，也属于有价值的反馈，因为了解用户数量及使用方式同样重要。

短期内，我们计划为 mac_do(4)/mdo(1) 增加类似审计的功能。显示传递给内核的最终凭据有助于检查调用是否符合预期目标。生成 mac_do(4) 规则中授权特定 mdo(1) 调用的目标部分，可以帮助管理员构建 mac_do(4) 配置，或更好地理解为何某些规则未按预期工作。与 audit(4) 子系统集成将允许事后跟踪凭据变更。通过 syslog(3) 记录失败尝试，将与 login(1) 及其他凭据变更程序的行为保持一致。mac_do(4) 很快还将允许配置它所考虑的可执行文件，以支持 thin-jails 场景及其他用户态程序[24](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor026)。此外，它还将监控传统系统调用，如 setuid(2)，而不仅仅是 setcred(2)，每个调用都被视为一次完整的凭据转换。

长期来看，我们可能考虑提供类似 su 或 doas 的功能，例如要求输入密码，或者更广泛地利用 pam(3)、设置资源限制及其他属性（如完整登录过程），或仅允许启动特定命令。然而，目前尚不清楚这些功能如何整合到 mdo(1) 中，因为它并非“setuid executable”，也不确定是否应走不同的实现路径。

例如，我们进行了初步研究，探讨如何为某些凭据转换增加密码请求支持。由于 mdo(1) 可以被任何用户启动，因此需要一种机制来检查密码，而密码数据库并非所有人都可直接读取[25](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor027)。这种情况类似于使用 CAPSICUM 能力模式[26](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor028)的程序，有时需要访问比其直接权限更高的数据。这可以通过让一个不受限制的进程代表能力模式下的进程执行必要的访问来解决。libcasper(3) 是 FreeBSD 针对多个服务实现这一思路的方案，包括提供 cap_pwd(3) 服务以访问密码和组数据库。不幸的是，直接使用 libcasper 不可行，因为 cap_enter() 会创建并连接到一个使用相同凭据启动的进程。mdo(1) 将需要一个具有特权的外部守护进程来提供 cap_pwd(3) 服务。

我们还可以设想多种替代方案，开发成本各不相同，包括：将密码配置完全推入 mac_do(4)（类似规则的处理）、将 mdo(1) 转为“setuid”可执行文件，但在大部分操作中及调用 setcred(2) 时放弃 root 权限，或保持 mdo(1) 原样，同时为这些需求提供另一个“setuid”可执行文件[27](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor029)。然而，除第一种方案外，其他方案提供的安全保证均低于初始方案，而第一种方案则灵活性较低，因为它不支持其他形式的认证，也不支持由用户态最佳施加的额外转换限制[28](https://freebsdfoundation.org/our-work/journal/browser-based-edition/freebsd-15-0/credentials-transitions-with-mdo1-and-mac_do4/centner.html#_idTextAnchor030)。

我们希望您会觉得 mac_do(4)/mdo(1) 有用！请分享您的反馈，以及更广泛的安全需求，即便不一定与本文介绍的框架直接相关。

### 注释

1. 实际上，是特殊的用户 ID 0。“root” 这个名称解析为 ID 0，其他名称例如 “toor” 也可能如此。
2. 更准确地说，分别是有效用户 ID 和保存的用户 ID，以及有效组 ID 和保存的组 ID。保存的用户 ID 和组 ID 在 POSIX 规范中正式称为 “Saved Set-User-ID” 和 “Saved Set-Group-ID”。
3. 其他认证机制可以通过 PAM 进行配置，入门可参见 pam(3)，针对特定应用的配置参见 pam.conf(5)，规范模块参见 pam_unix(8)。
4. 由于传递给 su(1) 的附加参数会被交给目标用户的 shell，因此程序及其参数必须通过 shell 的 `-c` 参数（或等价方式）传递。对于 sh(1) 及其后代，它们必须被组合为一个单一参数，由启动的 shell 进行解释，有时还需要额外一层引用。
5. Mandatory Access Control。参见 mac(4)。
6. 此处描述的更新版 mdo(1) 通常会随 FreeBSD 14.4 一同发布。
7. 可能存在多个用户映射到同一个数值 ID。doas(1) 的一个缺陷是它会静默地采用第一个匹配的用户名。mdo(1) 通常采取更为保守的做法，不会静默执行非显而易见的操作，这里即使只有一个匹配的用户名，也不会尝试使用它。
8. 即真实、有效和保存的组 ID，相对于补充组而言。
9. 为方便脚本使用，`-s` 实际上与 `-G` 兼容，可用于修改它，因此实际上在处理顺序上会在 `-G` 之后生效，即便它在命令行中出现在 `-G` 之前。不过，目前同时使用 `@` 和 `-G` 会被视为错误（重复指定），这一限制未来可能会被解除。
10. 匹配真实用户 ID，因为它代表用户身份，而非有效用户 ID，从而默认防止另一套规则应用于“setuid 可执行文件”。也就是说，由于在 FreeBSD 上允许非特权用户将真实用户 ID 设置为有效用户 ID，这一区别目前不是绝对限制。
11. 自 FreeBSD 15 起，密码数据库中用户的初始组也会被安装为补充组，这在 Linux/glibc、NetBSD、OpenBSD 和 illumos 上也是如此。为兼容 FreeBSD 14.3，这里演示了目标子句 `+gid=80`，在 15.0 上同样有效，而非 `!gid=80`，后者只允许在 15.0 上过渡。
12. 更正式的说法，`gid` 和 `+gid` 目标子句构成逻辑“或”关系。
13. 更正式的说法，规则构成逻辑“或”关系。
14. 如果将 `+gid=.` 替换为 `!gid=.`，规则仅在当前补充组不包含 0 时允许过渡，而不是允许过渡到除 0 外的所有当前组。后续可能放宽此限制。
15. 有一些例外。我们在前一个脚注中已经看到一个。另一个是：一方面，真实、有效和保存的用户 ID；另一方面，真实、有效和保存的组 ID，会被不加区分地对待。将它们分别处理被认为会引入额外复杂性，而收益甚微，因为 FreeBSD 当前的 setresuid() 允许非特权进程将其任一用户 ID 设置为其他任一用户 ID 的值。我们未来可能会禁止这种行为。
16. 自 FreeBSD 8.0 起。
17. 其全局 jail ID 始终为 0。jail ID 是全局的，但每个进程都会将其直接所属的 jail 的 ID 视为 0。
18. 在较早的实现中，该参数名为 mdo，本意是按此处描述的方式工作，但由于 bug 并未实现。
19. 因而会产生一致性问题。
20. 亦称 Postel 定律。“对你所接受的要宽容，对你所发送的要保守。”
21. 由脚本 `/etc/rc.d/sysctl` 完成。
22. 即要么这些钩子的现有实现需要开始支持事务模式，要么我们将完全绕过这些钩子，而后者被认为对使用者来说过于出乎意料。
23. 即在启用资源记账功能时使用 mac_do(4)，以及在 64 位架构上运行 32 位的 mdo(1)。
24. 该功能的大部分代码是在 2025 谷歌编程之夏期间编写的，应当很快会被集成。
25. 为了避免泄露可用于离线攻击的密码哈希。
26. 一种进程模式，其中对全局命名空间的大多数访问都会受到限制，只能使用已有的文件描述符。
27. 这可以采取多种形式，例如先将 doas(1) 引入基本系统，然后再针对我们独有的安全特性进行定制，尽管这在目标凭据粒度上是一种退步。或者，我们也可以创建一个可执行文件，与 mdo(1) 共享部分代码和命令行接口。结合两种方法以取长补短，也可能是可行的。
28. 但它的好处在于不会削弱当前已有的安全保证，因为密码同样会由内核进行校验。
29. 指不同的真实、有效和保存的组 ID。

---

自 2004 年底起，**Olivier Centner** 在他自己的机器以及部分他曾服务公司的机器上长期使用 FreeBSD。在此期间，他积累了一系列私有定制，包括对 rc 脚本和部分内核模块的修改。在 CAD 和金融行业工作 15 余年后，他最近回到纯粹的 IT 领域，尤其专注于操作系统开发。他的主要兴趣集中在内核开发，尤其关注电源管理、安全、调度、文件系统和 jail。他现在是 FreeBSD 基金会的合同开发者。
