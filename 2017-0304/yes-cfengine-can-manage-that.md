# 没错，CFEngine 能管那个

作者：Phillip R. Jaenke

配置管理，你可能猜到了，目前是 IT 界的热门话题。随着人们随手就开出十几台 VMware 虚拟机、再在 AWS 上加 50 个实例，它比以往任何时候都更必要。曾经，在两打主机上跑 200 个环境（因为 jail 确实好用）就算大型环境了。那也是两名熟练的系统管理员能轻松维护的规模，因为每个环境都是他们亲手搭建的。当然，会有不少一次性特例，但你用一个文本文件、一张便利贴就能轻松跟踪它们。（不要禁用 clx9oa 上的密码登录，因为他们从 AS/400 上访问那台机器。）

如今，“大型”环境从大约 20,000 台主机起步，只会更大。可以想见，再高超的技术也弥补不了这种规模。而这些环境往往可以追溯到十多年前。可以保证，你迟早会把头探进一台麻烦主机，发现它是一台跑 Solaris 2.5.1 的 Sun E420R，尽管公司十多年前就已标准化到 FreeBSD 和 Windows。而且，它是唯一一台存储着全部登录数据库——所有员工和客户——的系统。

这些问题当然不是新鲜事或未知的。多年来，范围和规模发生了巨大变化，但此类设置存在并引发无数次”WTF 时刻”电话的概念，几乎和继电器里的飞蛾一样古老。一位在奥斯陆大学学院做博士后研究员的先生意识到这是一个真正的问题，于是他去当了修士——等等，不对，那是我们很多人希望能做的事。实际上，Mark Burgess 决定拿出一个解决方案，并于 1993 年发布了 CFEngine 的第一个版本。除了 shell 脚本和贴在显示器上的便利贴（也许还有 IBM 的某些冷门产品），CFEngine 是所有配置管理方案中最古老、最成熟的一个。而且不是领先一点点。本期中你能找到的第二古老的是 Puppet，其首次发布比 CFEngine 晚了 12 年。而我们这些经历过”一杯痛苦”和”成熟方案”的人一听到 20 世纪 90 年代的遗留 C 代码就脊背发凉。CFEngine 确实用 C 编写，但这是出于可移植性和性能考虑。其他配置管理器主要或完全围绕自身代码和配置管理的通用概念构建。CFEngine 的长青源于它围绕两个关键概念：收敛和 Promise 理论。Mark Burgess 没有说”情况一团糟，用简单的方式把它们都变成一样”，而是寻求理解最初做出的那些临时选择，并提供一种理解这些选择的方法。你当然可以在维基百科上读到这些。但这就是 CFEngine 二十多年间、经历两次完全重写仍能给用户带来一致而稳定的行为预期与概念的根源所在。因为它是概念和理论的实现，其实现细节远不那么重要。

## CFEngine 中一个实用的 Hello World

由于 CFEngine 使用双向通信（agent 和 server 模型）且专门针对异构环境，我们这里的示例将展示如何用 CFEngine 的领域特定语言（DSL）写一个”Hello, World”promise，同时告诉你一些关于系统的信息。首先，你需要理解流程如何运作。首先，你用策略文件定义期望状态。agent 随后确保该状态（每 5 分钟一次）。如果你使用 CFEngine Enterprise，server 可选地验证该期望状态。

> 顺便一提：Enterprise 在 25 台主机以内完全免费，但遗憾的是，它要求你的策略 hub 运行 Linux。

### Promise

```sh
# 这是一个 CFEngine Hello World Promise
bundle agent hello_world
{
    reports:
        any::
            "Hello World";
}
```

这个 promise 建立了一个名为 `hello_world` 的 bundle，它会在任何类的主机上报告。现在我们用 CFEngine 的内置类（称为硬类）来做一些更复杂的事：

```sh
# 精美的 CFEngine Hello World Promise
bundle agent hello_world_freebsd
{
    reports:
        any::
            "Hello World";
        freebsd::
            "Thanks for reading FreeBSD Journal!";
        windows::
            "I think I got lost.";
}
```

这个策略会多做两步。如果系统是 FreeBSD 硬类的成员，它会提供一条与 windows 成员不同的问候，而所有其他系统提供默认问候。来看看这个策略在两台不同主机上的实际运行：

```sh
root@freebsd11 # /var/cfengine/bin/cf-agent --no-lock --file ./hello_world.cf --bundlesequence hello_world_freebsd
R: Hello World
R: Thanks for reading FreeBSD Journal!
```

```sh
[root@centos7] # /var/cfengine/bin/cf-agent --no-lock --file ./hello_world.cf --bundlesequence hello_world_freebsd
R: Hello World
```

```sh
root@freebsd11# /usr/local/sbin/cf-agent --no-lock -D windows --file ./hello_world.cf --bundlesequence hello_world_freebsd
R: Hello World
R: Thanks for reading FreeBSD Journal!
R: I think I got lost.
```

如你所见，CFEngine 的类机制实际上相当容易理解和运用。这让你作为防止混乱的负责人，有了一种简便可靠的方式在策略中区分你的主机。因为你也可以用 promise 以你喜欢的几乎任何方式定义软类，这给了你一种灵活性和一致性的组合，许多其他配置管理方案往往缺乏或难以实现。但和大多数 Hello World 示例一样，这仅触及了可用能力的皮毛。

## Promise、Policy 和 Bundle，为什么？！

使用 CFEngine 时，你可能是在编辑文件，但那些文件与 Promise 理论的核心概念密不可分。Promise 理论的要旨是，自治的参与者将达成自愿合作。这听起来与实际做配置管理相去甚远。“自愿”、“自治”、“合作”这些词在此语境下听起来很怪。所以，让我们先剖析 Promise 理论究竟是什么。你可能猜到了，Mark Burgess 提出它是作为解决基于义务的策略管理方案中问题的一种方法（或者更准确地说，是为了解释 CFEngine 的运作模型）。配置指令不是由单台服务器或单组权威服务器下发，而是组内每个成员都拥有控制自主权。也就是说，它们不能被确定性命令强制成特定行为。相反，agent 只能就自身行为作出 promise，不能就其他系统的行为作出 promise。CFEngine agent 不是把自己绑在单台中央服务器上，而是 agent promise 从另一台服务器获取数据并按其指令行事。

一个很好的类比是工作中常见的一幕。你的实习生 promise，只要你告诉他怎么做，他就会修好所有 50 台系统上的问题。你给实习生一份基本步骤清单——编辑这些文件，重启这些守护进程。然后实习生按你写的步骤执行。因为他是实习生，你随后验证结果是否符合预期。这就是 Promise 理论实际运作的一个（admittedly 非常简化的）示例。你不是出于任何明确的义务或要求而合作，除了想让系统处于特定状态；实习生不对自身以外的任何行为作 promise；你不是站在实习生身后盯着或替他输入命令——你只告诉实习生你想让他做什么；实习生只就他直接知道的特定信息向你汇报；你随后验证指令是否产生了期望的结果。我知道”实习生自愿并且真正理解自己做了什么”这部分可能有些不切实际，但你明白意思。现在想象一支乐于助人、配合且博学的实习生大军。这基本上就是 CFEngine 模型。成千上万的实习生问 root：“嘿，你想让我干什么？好的，干完了，结果是这样。“理解了这一点，就更容易理解 promise、policy 和 bundle 如何在 CFEngine 中组合到一起。用最简单的话说，policy 是 promise 的集合。如果我有一个名为 `abc` 和一个名为 `123` 的 promise，我就可以把它们组合成一个名为 `321cba` 的 policy，同时纳入这两个 promise。如果你应用一个没有 promise 的 policy，什么也不会发生。我们上面的 Hello World 就是一个 promise 的示例。要把该 promise 纳入 policy，我会用一个带 bundlesequence 的 body 语句，如下：

```sh
body common control
{
    # 这些是我们希望 agent 执行的 promise
    bundlesequence => { "hello_world_freebsd" };
}

bundle agent hello_world_freebsd
{
    ...
}
```

bundlesequence 可以看作一个 policy。CFEngine 会按 bundlesequence 语句中包含的 promise 顺序执行。然后我可以用这些 promise 控制其他 promise 的行为。例如，我可能用一个 promise 判断系统的主机名是否以字母 qa 开头，若是则安装不同的 sudoers 文件。我也可以把主机名检查完全放在 sudoers promise 内部。但什么是 bundle，它又如何融入其中？尤其我们刚说 promise 进入 policy！bundle 是把相似或相关的 promise 收集到单个 promise 中的一种方式。例如，我的 policy 可以 promise，任何 `any::` 系统都接收 `standard_users` bundle。在该 bundle 内，我随后定义适当的 promise 覆盖每个用户和操作系统。换言之，bundle 提供了一种基于单个策略决策作出许多 promise 的方式。bundle 的限制在于，bundle 内的每个 promise 不能作为单独的 promise 对待，也就是说你不能在 bundle 之外复用这些 promise，除非把它做成单独的独立 promise。如果所有 promise 都需要在另一个 bundle 之前检查，你也不能使用它。这让 bundle 非常适合粗粒度的配置项。但 bundle promise 也能做独立 promise 能做的一切。所以那些粗线条其实可以极其针对具体系统——比如设置正确的 IP 地址、在 Active Directory 中注册、甚至加入集群。一切都取决于你如何编写 promise 本身。

## 那么我该怎么搭起来？

正如我们刚提到的（你可能也注意到了），CFEngine 是基于 agent 的架构。agent 会连到一台服务器——这台服务器称为你的 Policy Server。Policy Server 是所有策略文件以及你想分发的任何其他文件的中央仓库。此外，使用 Enterprise，你可以用它把 CFEngine 二进制文件或软件包分发到所有 agent 系统——得益于硬类，会按操作系统、版本和字节序自动选择！除了 Enterprise（称为 Nova Hub）有额外的界面和报告工具外，任何能运行 CFEngine 的系统都可以是 Policy Server。只有两个基本要求。第一，它必须运行与你客户端兼容的 CFEngine 版本。第二，它必须能存放你想用 CFEngine 本身分发的任何文件。（显然，如果你更愿意用 HTTP/HTTPS 分发或用 NFS 处理大文件，可以完全无视这一点。）

搭建 Policy Server 在大局上同样简单。首先，你需要从 ports 构建并安装 CFEngine。然后，构建并安装对应的 CFEngine masterfiles。例如 3.7 版，你会使用 `sysutils/cfengine37` 和 `sysutils/cfengine-masterfiles37`。注意，由于一些怪癖，CFEngine 不能从 **/usr/local** 运行，因此你需要先把二进制文件从 **/usr/local/sbin/** 复制或软链接到 **/var/cfengine/bin/**。最后，在 rc.conf 中启用 CFEngine 并运行命令：

```sh
/var/cfengine/bin/cf-agent --bootstrap --policy-server 127.0.0.1
```

你已成功搭建了 Policy Server！真的，就这些。你现在可以开始编写策略并连接客户端——只需把刚才运行的 policyserver 参数的 IP 地址改掉即可。CFEngine 的通信完全加密，并自动处理创建和维护安全密钥的所有繁琐工作。

CFEngine 让人常常意外的一点是，客户端具有生存能力。假设你需要把 Policy Server 下线换一根坏了的 DIMM。（而不是因为初级管理员 Jimmy 用 root 跑了从互联网上下载的随机 shell 脚本。顺便说一句，现在有个初级管理员空缺。）但你的环境继续运转，继续执行策略。因为——作为 Promise 理论的一部分——CFEngine agent 也是自治节点。一旦成功 bootstrap，它们会继续执行已有的策略。如果服务器恰好下线，它们会耸耸肩说：“好吧，我没有新策略”，然后继续执行现有的。它们当然会伺机尝试重连，但不会停止工作。

举个例子，假设你的策略要求 root 密码必须是 `r34lly$ecur3`，而整个网络断了。Bob the Developer（我们都有个 Bob）发现机会，从 CD 启动并把一台客户端的 root 密码改成 `b0b0wnz!`，因为他不喜欢当不了 root。那台系统重启后，会立刻记录有人动了密码，重置密码，并在能连上时告诉 Policy Server：“嘿，有人试图改这个密码。“哪怕有人拔掉网线、把交换机点了也一样。

## 说到那些策略……

我一直在策略上语焉不详是有原因的，因为它们在 CFEngine 中确实非常灵活。因为系统的每个可能方面都可用领域特定语言（DSL）在策略中表达（几乎所有状态也可如此表达），你的策略能做什么基本上没有限制。确定你自己的网络中想通过策略实现什么，只有你能决定（希望别让会计盯着你看）。那种灵活性当然也让真正解释或描述你能做什么几乎不可能。每个示例都有十几种以上的其他做法。也许你想用一条策略根据主机名的正则匹配为 FreeBSD 系统分配类——你能做到！也许你想改用静态方式——同样可以！CFEngine 唯一真正的限制是你的想象力（以及你愿意花多少时间写策略）。简洁通常是最好的答案，不是因为你做不到，而是你可能轻松地把余生都花在敲策略上，什么也不干。由于这种力量，大多数人选择同时实现一个优秀的第三方 promise 库以减轻工作量。最受欢迎的有 Neil H. Watson 的 Evolve Thinking 库、Normation 的 ncf 库，当然还有 CFEngine 社区开放 Promise-Body 库（又名 cfengine_stdlib.cf）——后者你已作为 masterfiles 的一部分安装了。所有这些库都用 CFEngine 的 DSL 实现，因此与操作系统无关。你对每个系统使用相同的 promise 库，不论发行版和操作系统。自然，CFEngine 版本兼容性有常见注意事项，库中某些函数是操作系统特定的。但因为它们用 DSL 编写，几乎能在所有操作系统上不做修改地运行，不会在不适用的操作系统上运行。所以，与其怀疑你能否在策略中做某件事（能！），决策过程变成了策略中应该实现什么。彻底的疯狂，对吧？根据什么适合你的环境做决策，而不是根据本周哪家供应商正在为没真正交付的东西道歉？接下来呢，是不是要 accommodate 所有那些特别的雪花而不破坏一切？哦。对。我已经提到过你也能做到这一点。那么，让我们谈谈一个实际示例：我的环境。我运行 FreeBSD、AIX、Linux、VMware、HyperV 和 Windows。这显然是个复杂环境，又因使用 NFSv4 而更加复杂——因此 Kerberos 是强制的。我还用 automount 管理主目录、Jenkins、一个真正的 poudriere 集群（没开玩笑！）、<此处插入强制云热词>，以及 Juniper——你知道为什么用 Juniper。当我搭建一台新的 FreeBSD 系统时，我使用 VMware 中一个已安装好 cfengine37 的模板。我要做的只是在策略中为 vnx0 添加 MAC 地址，把它关联到一个主机名。我用一条命令 bootstrap agent，CFEngine 就为我完成以下所有事：

- 终止 dhclient 并安装正确的 **/etc/resolv.conf**
- 用 DNS 为 vnx0 配置 IPv4 地址和 defaultrouter
- 用 DNS 为 vnx0 配置 IPv6 地址和默认路由
- 安装正确的 pkg 仓库配置并更新 base 中的任何软件包
- 安装并配置我需要的最新版支持软件包——krb5、nss-pam-ldapd、sasl、kstart 等——这些不属于模板
- 安装正确的 **/etc/krb5.conf** 并用 DNS 获取机器的 keytab
- 把 root 密码设为本月（或本周，如果我又忘了）该用的那个
- 安装正确的 autofs 脚本并在 **/etc/rc.conf** 中启用 autofs
- 给我发一份报告，告知系统成功 bootstrap 并详述配置

之后呢？那是我的基线策略，所以它会一直这么做。如果我在策略服务器上更新 root 密码的 promise，它会更改 root 密码。如果我在 DNS 中更改 IP 地址，它会为我更新 rc.conf。真正神奇的是，每个操作系统获得完全相同的对待，只是根据 agent 返回的类调整 promise 以适配。（好吧，真正的神奇还包括 JunOS 上的 CFEngine，但那只是你我之间的小秘密。）因为你可以把 promise、bundle 和 policy 组合成几乎任何东西，用 CFEngine 能做的事真的没有限制。可能不总有”简单”的方法，但如果你能在操作系统上用命令做到，你就绝对能用 CFEngine 做到。

## 当然也有缺点

没有软件是完美的，CFEngine 当然也不例外。因为它有二十多年的历史，有一些——怎么说呢——遗留部分和行为必须保持原样以维持各种原因——通常是某个给 CFEngine AS 付了很多钱的人。这是个正当理由——那些钱让它保持开源，你懂的！但它确实可能在现代系统上引起头痛，当你撞上其中一个时。它也不太像走火的枪，更像是走火的加特林自动炮。确实有许多方法可以降低风险。然而，你迟早会犯一个没捕捉到的错误。大多数时候，唯一的影响是 agent 拒绝运行你破损的新策略，继续执行之前的策略。但有时是 `rm -rf /$EmptyVariable/*`——这会在整个环境中并行执行，通常不到 5 分钟。因为 CFEngine 如此强大灵活，也很容易让自己被可能性淹没。说真的，可能性几乎是无限的。但也有同样多的方式让你最终得到一个由数百个文件中数千个 promise 组成的庞然大物。我个人见过能与 **/usr/src** 媲美的配置——但组织性差得多。跟上你需要做的事，同时让主文件保持受控，感觉像是互相竞争的需求。而最大的缺点之一是 FreeBSD 被 CFEngine 视为二等。这不意味着他们不支持——远非如此，多亏了 port 维护者和我们少数用户的努力。但在功能和特性方面，FreeBSD 没有得到 Linux 那样的全部待遇。你不太可能遇到做不到的事，但 cf-agent 不会”自动地”报告一些监控数据，你将不得不花更多时间编写自己的 promise 和 bundle，因为标准 promise 库不会优先考虑 FreeBSD。

## 一美元买这个！我在哪里能学到更多？

作为最成熟和稳定的配置管理系统之一，CFEngine 有大量资源可用。最佳起点是 CFEngine 官网（你还可以抓一些概览资料塞到会计门缝下）：www.cfengine.com。那里也是你可以免费获取 CFEngine Enterprise 的地方（25 台主机以内完全免费）。但和任何如此复杂和强大的产品一样，这只是开始。如果你准备一头扎进去，Vertical Sysadmin 提供了一系列培训视频和文章，价格低到完全免费。如果有人能批预算，他们还提供一些你能获得的最深入的一对一培训。事实上，我会建议每个对 CFEngine 感兴趣的人都从他们的《IT Automation with CFEngine: Business Values and Basic Concepts》视频开始。不过，说实在的。能拿到预算？对，拿不到预算。为此，CFEngine 有 help-cfengine 邮件列表。如你所料，你不仅会看到 CFEngine 冠军们定期提供帮助，还有 CFEngine 的开发者和员工。光是浏览存档，你常常就能发现问题的答案已经在那里了。还有 Freenode 上的 #CFEngine IRC 频道。写好第一个策略并开始上手后，我强烈建议在走得太远之前先看看官方的《Best Practices》指南。虽然主要面向 Enterprise 用户，但涵盖了从如何为策略文件使用版本控制，到为扩展到数千台主机调整策略的一切。祝你 promising 愉快！

---

**PHILLIP R. JAENKE** 是一名系统工程师和管理员，同时也在存储、网络和写作方面不只是浅尝辄止。他干这行时间长得曾经给 Berkeley Software Design 开过支票。在不忙于在各种 Unix 上维持大型企业的灯火不灭时，他尽可能地为各种开源项目出力，包括 CFEngine 和 FreeBSD。他还设计并开发了广为使用的 BabyDragon VMware 参考白盒机，并开发了 TaleCaster 综合媒体系统。
