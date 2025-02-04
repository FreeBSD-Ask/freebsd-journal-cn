# PAM 小窍门

- 原文链接：[PAMTricks and Tips](https://freebsdfoundation.org/wp-content/uploads/2022/11/lucas_pam.pdf)
- 作者：**MICHAEL W LUCAS**

可插拔身份验证模块（PAM）也许是 FreeBSD 中最不被理解、但部署最广泛的部分。每个 FreeBSD 系统都通过 OpenPAM 套件使用 PAM。大多数系统管理员都不会去修改它；如果他们不得不更改，只能按照那些看起来模糊合理的网上贴子来操作。PAM 看起来比实际要简单，但这种简单性可能会让你无意中制造出复杂性。使用 PAM 的关键就在于避免这种情况发生。

在本文中，我们将探讨 PAM 的组成部分和配置，帮助你避免一些常见错误，并找出调试错误的方法。

## 什么是 PAM？

大家都认同，用户名和密码并不是一个理想的认证系统，但对于替代方案却众说纷纭。不同的环境有不同的需求，也许你需要 Kerberos、LDAP 或 SSH 证书；也许你用硬件 DNA 扫描仪，或者用为个人牙齿模型定制的咬合护具来认证系统管理员。每种认证方法都需要自己独立的代码。你可以尝试编译每个程序以支持所有可能的认证方法，但这会导致代码量无限膨胀；或者你可以为每种认证方法构建一个兼容的共享库，仅在需要使用该方法时加载对应的库。这就是“可插拔”的意义所在。每个库就是一个“认证模块”，简称“模块”。

FreeBSD 的 OpenPAM，就像原始 Solaris 的 PAM 一样，只处理认证和与认证相关的任务。如果你曾经使用 Linux，就会发现 Linux-PAM 的开发者认为标准 PAM 过于简单，于是将其他功能挤进了他们的认证栈。而 OpenPAM 中的功能和工具适用于大多数 PAM 栈。

## PAM 配置

在 `/etc/pam.d` 中为 FreeBSD 基础系统配置 PAM，而在 `/usr/local/etc/pam.d` 中为各个软件包配置 PAM。每个守护进程都有自己的配置文件，其文件名与程序名称相同，用于定义 PAM 策略。比如，你可以去看看 `sshd(8)` 的配置文件 `/etc/pam.d/sshd`，你会看到很多类似这样的行。

```sh
auth sufficient pam_opie.so no_warn no_fake_prompts
auth requisite pam_opieaccess.so no_warn allow_local
auth required pam_unix.so no_warn try_first_pass
…
```

每一行对应一条 PAM 规则。每条规则包含四个组成部分：类型、控制、模块以及模块参数。

第一个示例中，规则的类型为 **auth**，控制为 **sufficient**，模块为 **pam_opie.so**，模块参数则为 **no_warn** 和 **no_fake_prompts**。

## 规则类型

认证不仅仅是用户凭据的问题。系统还必须检查是否允许访问、为用户提供资源，并允许对认证系统本身进行管理。这就是类型声明发挥作用的地方。

- **auth** 类型：验证用户的认证信息并设置资源限制。如果你错误地输入了密码或提供了一个不存在的用户名，auth 规则会拒绝你。auth 规则还用于设定限制，例如最大进程数或内存使用量。上面已经提到了 auth 规则。

- **account** 类型：基于除用户认证之外的限制来控制访问。如果用户尝试在非工作时间登录，account 规则会阻止该访问。

- **session** 类型：处理服务器端的设置。一个命令行用户需要一个虚拟终端、一个主目录，并且可能需要一条记录表明他们已经登录。而匿名 FTP 用户则不应获得虚拟终端，并且只能访问特定的目录。session 规则处理所有这些。

- **password** 类型：处理认证更新。强制密码更改就是由 password 规则完成的。

## 控制

你可能在防火墙和服务器配置中见过访问控制列表。PAM 规则类似于访问控制列表。每个模块可以拒绝、失败或者对认证请求“持中立态度”。控制语句告诉 PAM 如何处理每个模块的决策。PAM 有四种常用的控制语句：**required**、**requisite**、**optional** 和 **sufficient**。

- **required** 控制：表示该模块必须返回成功才能允许访问。如果该模块成功，用户将获得访问权限，除非后续的规则阻止访问。即使一个 required 模块失败，PAM 也会继续处理后续规则。

- **requisite** 控制：与 required 控制类似，但如果认证模块失败，则规则处理立即停止。这可能会泄露认证失败的原因。

- **optional** 控制：用于支持附加功能，比如 SSH 代理或 Kerberos。只有在没有其他模块拒绝或接受该会话的情况下，它们才能决定允许或拒绝访问。你可以用 optional 控制来实现例如在认证失败后增加下次尝试之间的等待时间等功能。

- **sufficient** 控制：表示如果该模块成功且之前没有任何 required 控制失败，则用户立即获得访问权限，规则处理随即停止。如果该模块失败，PAM 不会因此拒绝访问。


## 模块和参数

PAM 模块是实现特定认证功能的共享库。密码处理是一种模块；LDAP 验证是一种模块；与 SSH 代理交互也是一种模块。要理解 PAM 策略，你需要知道每个模块的具体作用。每个 OpenPAM 模块，以及大多数附加模块，都有对应的手册页。当你第一次解析一个 PAM 策略时，会阅读大量的手册页。

每个模块还可以带有参数。虽然每个模块都可以有自己特定的参数以满足其需求，但大多数模块也接受一些通用参数。  
- **debug** 标志会记录额外的信息，以期帮助你弄清楚为何认证没有按预期工作。  
- **no_warn** 标志会抑制任何关于认证请求被拒绝原因的用户反馈。  
- 对于密码处理模块，有两个特殊参数：**try_first_pass** 和 **use_first_pass**。  
  - **try_first_pass** 选项会重用用户之前输入的密码，但如果不行，该模块可以再次提示用户输入密码。  
  - **use_first_pass** 选项则要求模块使用用户之前输入的密码，如果不成功，则直接拒绝该请求。

## 阅读策略

让我们再来看一下 sshd(8) 的示例规则。这些都是 auth 规则，因此涉及接受或拒绝登录凭据。先来看第一条规则……

```sh
auth sufficient pam_opie.so no_warn no_fake_prompts
```
这条规则是 sufficient 类型的规则。如果该模块成功，认证请求就被允许，并且规则处理会立即停止。  

该模块是 **pam_opie.so**。根据手册页 **pam_opie(8)**，我们知道它支持 OPIE。 你可能需要做一些进一步的研究，以了解 OPIE 是指“一切中的一次性密码”。这条规则声明，如果有人使用 OPIE 进行认证，他们会立即获得访问权限。

```sh
auth requisite pam_opieaccess.so no_warn allow_local
```

这条规则是 **requisite** 类型的规则。如果它失败，规则处理会立即停止。这看起来……相当严格？

这条规则针对的是另一个 OPIE 模块，**pam_opieaccess**。既然在前一条规则中使用 OPIE 认证的用户会立即获得访问权限，那么为什么还要有另一条 OPIE 规则呢？查阅 **pam_opieaccess(8)** 手册可以发现，该模块用于检查用户是否配置为必须使用 OPIE。如果一个用户被配置为必须使用 OPIE，而他们正确地输入了 OPIE 信息，就绝不应该触发这条规则；相反，他们应该被拒之门外。

现在请看第三条规则。

```sh
auth required pam_unix.so no_warn try_first_pass
```

这是一条 **required** 规则。进入到这一层的用户必须通过该模块，否则将被拒绝访问。  
**pam_unix(8)** 模块检查密码文件，实现标准的用户名和密码认证。因此，该策略可以总结如下：  
- 如果用户通过 OPIE 认证，则立即允许访问；否则继续往下检测。  
- 如果用户被配置为必须使用 OPIE，则拒绝他们。正确输入密码的 OPIE 用户永远不会走到这里。  
- 如果用户输入了正确的用户名和密码，则允许访问。  

大多数账户不要求使用 OPIE，因此认证过程会直接转到密码文件的验证。这部分显得非常简略。那么，对于使用无效 shell 或设置为 nologin(8) 的用户呢？或者那些当前不允许登录的用户呢？这些情况属于 **account** 规则，而不是 **auth** 规则。如果你查看 `/etc/pam.d/sshd`，会看到专门检查用户访问权限的 account 规则部分。



## 添加认证方法

人们通常是在组织开始要求两因素认证时才接触到 PAM。可能是 Google Authenticator、Yubikey 或 Cisco 的 Duo。也许你还有专用的认证硬件，可以读取声纹、指纹或嗅觉特征。接下来，我们将使用后面三种构建一些不太常见的认证规则。下面是一个相当简单的例子。由于我还没看过这些不存在模块的文档，所以我将忽略那些选项。

```sh
auth required pam_voice.so
auth required pam_finger.so
auth required pam_aroma.so
```

这三个策略都是必需的。用户必须提交正确的声纹和指纹，而且还得气味合适，才能完成认证。

```sh
auth required pam_voice.so
auth requisite pam_finger.so
auth required pam_aroma.so
```

中间的策略（针对 pam_finger.so）是 requisite 类型的规则。如果该规则失败，认证检查会立即停止，并且应用程序会收到失败通知。进行气味分析开销很大，我们不希望浪费资源。  

也许你希望允许用户选择使用哪种认证方法。

```sh
auth sufficient pam_voice.so
auth sufficient pam_finger.so
auth required pam_aroma.so
```

在这里，我们的前两种认证方法都是 sufficient 类型的。用户可以选择使用声纹或者指纹进行认证。如果这两种方法都失败了，用户就必须提供他们的“臭味”。我也可以将最后一条规则设为 sufficient，但如果策略在此结束，我会希望添加另一条规则，调用 **pam_deny.so** 来明确拒绝认证。

## PAM 调试

你的精心调校的策略不起作用？那就糟糕了。PAM 提供的显式调试非常有限，你只有三个选择：debug 参数、pam_echo 和 pam_exec。

许多模块支持 debug 参数。这可能会将调试信息传递给用户，也可能会将调试信息输出到日志文件，或者什么也不做。模块会忽略不支持的参数，因此在策略中到处添加 debug 参数不会让 PAM 变得更糟。

**pam_echo(8)** 模块接受一个文本字符串作为参数，这个字符串会传回给用户。这些规则总是设为 optional 类型。让我们在其中一个实验性策略中添加一些 echo 调试信息。

```sh
auth optional pam_echo.so “auth policy starting, trying voice”
auth sufficient pam_voice.so
auth optional pam_echo.so “voice failed, trying finger”
auth sufficient pam_finger.so
auth optional pam_echo.so “finger failed, taking a whiff”
auth required pam_aroma.so
auth optional pam_echo.so “how did we get here?”
```

如果你觉得这看起来就像在代码中到处散布 printf() 调用，那你说对了。对于 1990 年代的 Sun 来说这已经足够用了，对你来说也同样适用。

如果程序将输出反馈给用户，他们将在终端上看到调试语句。如果 debug 参数没有提供有用的信息，而且程序也不会将调试输出回显给用户，那么你就可以使用 pam_exec(8) 来进行更复杂的调试。

pam_exec 模块可以为你执行任意命令。是的，这意味着你可以编写 Perl 脚本，通过网络从 Microsoft Excel 表格中验证用户凭据，但我希望你不会因此而自讨苦吃。大多数情况下，pam_exec 只是让入侵者有机会干扰你的认证过程，不过它非常适合用来调用一个小型的 shell 脚本。比如这样：

```sh
auth optional pam_exec.so /usr/sbin/pamdebug.sh pam_voice
auth sufficient pam_voice.so
auth optional pam_exec.so /usr/sbin/pamdebug.sh pam_finger
auth sufficient pam_finger.so
auth optional pam_exec.so /usr/sbin/pamdebug.sh pam_aroma
auth required pam_aroma.so
auth optional pam_exec.so /usr/sbin/pamdebug.sh impossible_end_of_pam_rule
```

脚本本身非常简单。

```sh
#!/bin/sh
logger “process $PPID calling $1”
```

这会记录进程 ID 以及你当前处于认证的哪个阶段。你肯定不会希望在一个忙碌的生产环境中运行它——那里的用户不断登录和注销——但在测试系统上调试时，这会使工作变得简单。

PAM 拥有的功能和选项远比我在这篇短文中能介绍的要多，但希望你已经大致了解了如何修改规则并微调认证，以达到你期望的方式来“惹恼”用户。

祝你好运。

---

**MICHAEL W LUCAS** 是《Absolute FreeBSD》、《FreeBSD Mastery: Jails》以及其他四十八本书的作者，其中一本便是（请鼓掌）《PAM Mastery》。了解更多信息，请访问 [https://mwl.io](https://mwl.io)。
