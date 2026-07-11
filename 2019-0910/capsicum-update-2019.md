# 2019 年 Capsicum 更新

FreeBSD 是通用操作系统，其目标之一是为用户提供安全环境。为实现该目标，FreeBSD 引入了 Capsicum 框架。FreeBSD 社区每天都在改进它，本文将回顾 Capsicum 在过去一年的变化。

Capsicum 框架提供进程间的严格隔离。进程进入 capability 模式（沙箱）后，无法访问任何全局命名空间。进程可以通过 **cap_enter(1)** 系统调用进入该状态。Capsicum 与其他流行的沙箱框架不同之处在于，Capsicum 关注进程的 capability，而非过滤系统调用。在 Capsicum 中，文件描述符代表 capability。文件描述符非常适合充当 capability，因为可以复制（**dup(2)**）、通过 Unix 域套接字发送给其他进程，或撤销（**close(2)**）。

框架的另一组件是 capability 权限。这些权限让我们能进一步限制 capability（文件描述符）。通过 **cap_rights_limit(2)** 系统调用，可将描述符限制为只读（`CAP_READ`）或可写（`CAP_WRITE`）。即使文件描述符能在文件中重新定位偏移量（`CAP_SEEK`），也可限制其 capability。Capsicum 允许精确定义描述符的用途。目前有超过 50 种 capability。

该框架的更详细描述，建议你参阅 FreeBSD 期刊的其他期次，其中有对 Capsicum 更详尽的说明 [1] [2]。

## Casper

理解 Capsicum 环境所需的另一原语是 Casper 和 Casper 服务。Casper 通过便捷的 API 提供 capability 模式中无法获取的功能，使 Capsicum 更实用。Casper 借助进程的特权分离做到这一点。创建新的 Casper 实例后，初始进程会 fork。接下来，非特权进程进入 capability 模式。如果沙箱内的应用想在 Capsicum 中执行某些被禁止的操作，必须将这项工作委托给 Casper。Casper 通过名为 libnv（或 nvlist）的简单 API 与沙箱应用通信。

假设某个应用想解析 DNS 名称。在进入 capability 模式之前，它可以创建 Casper 服务——system.dns。每次需要解析 DNS 时，不调用 `getaddrinfo` 函数，而是调用 `cap_getaddrinfo`。Casper 函数的 API 与标准 libc 完全一致，只是多了一个接收 Casper 连接的参数。

Casper 服务的 API 很简单。得益于这一点，沙箱应用的编写轻松许多。没有 Casper，所有应用开发者都需重新实现特权分离例程，才能为新应用构建沙箱。

## fileargs 简介

fileargs 服务允许程序访问文件系统。该服务让沙箱化应用变得相对容易。**wc(1)** 和 **head(1)** 就是依赖它的应用示例。fileargs 服务并不提供完整的文件系统服务，主要针对那些以文件向量为参数打开的应用。不过，该服务也可用于其他场景。

该服务重要，原因有二。第一，如上所述，它允许沙箱化新应用。第二，因为这是第一个 API 不直接对应 libc 函数的服务。

目前，fileargs 服务提供两个主要功能——打开文件和获取文件状态。`fileargs_open`/`fileargs_fopen` 函数允许从给定路径打开文件，`fileargs_fstat` 函数用于获取文件状态。主要函数是 `fileargs_init`，该函数初始化 Casper 服务。

argc 和 argv 参数只是应用应当能打开的文件向量。flags 和 mode 参数与 `open` 的参数无异，描述了文件应如何打开、以何种模式创建。接下来是新打开文件应保留的 capability 列表。最后一个参数是服务中允许的操作。目前，服务定义了两个操作：open（`FA_OPEN`）和 lstat（`FA_LSTAT`）。

`fileargs_cinit` 函数与 `fileargs_init` 函数非常相似。唯一区别在于 `fileargs_cinit` 会复用已有的 Casper 实例。而 `fileargs_init` 函数会创建新的 Casper 服务实例。

清单 1 展示了为 **head(1)** 构建沙箱的补丁。补丁很简单。我们只需用正确的 capability 初始化 Casper 服务，将 argv 和 argc 传给它，并将 `open` 函数替换为 Casper 版本。最后进入 capability 模式。

值得注意的是，fileargs 服务 API 仍被视为实验性，可能发生变化。

## 改进 cap_sysctl

cap_sysctl 服务允许我们与内核状态交互。在最初的实现中，我们引入了 `cap_sysctlbyname` 函数。然而，当开始为 **rtsol(8)** 和 **rtsold(8)** 构建沙箱时，发现这还不够。sysctl 可以通过两种方式引用——文本表示和数字表示。

为这些应用构建沙箱的过程促使开发者扩展 cap_sysctl。引入了 `cap_sysctl` 和 `cap_sysctlnametomib` 函数。前者允许通过数字表示操作值。后者可从给定字符串表示获取 sysctl 的管理信息库（MIB）数字表示。这两个函数的接口与它们的前身非常相似。唯一变化是 Casper 函数期望多一个参数，即与 Casper 服务的连接。

接口扩展也意味着限制函数需要重新设计。我们为 cap_sysctl 服务引入了新接口：

- `cap_sysctl_limit_init` - 初始化限制结构
- `cap_sysctl_limit_name` - 通过名称表示限制单个 MIB
- `cap_sysctl_limit_mib` - 通过数字表示限制单个 MIB
- `cap_sysctl_limit` - 对给定的 Casper 服务实例设置限制并释放所有底层结构

清单 2 是使用示例。首先用 `cap_init` 创建 Casper 实例，再用 `cap_service_open` 创建 Casper 服务，这是标准方法。接下来初始化 sysctl 限制。我们将服务限制为只有一个 sysctl——`kern.trap_enotcap`。只能通过文本表示引用它。`CAP_SYSCTL_READ` 也意味着应用只能获取该 sysctl 的值。在清单 2 末尾，程序获取了该值。

## 私有服务

Mark Johnston 做了一些更令人兴奋的工作。当他为 **rtsol(8)** 和 **rtsold(8)** 构建沙箱时，实现了专用于这两个应用的私有 Casper 服务。**rtsold(8)** 是守护程序，用于在指定接口上发送 ICMPv6 路由器请求消息。该服务针对特定应用，因此没有理由将其公开。这种方法可能让我们走到这样一步：某些服务将从 Ports/打包系统安装。他的工作让我们看到，Casper 服务也可用于不同环境中的进程分离。

**rtsol(8)** 和 **rtsold(8)** 使用 Casper 创建了服务，用于在原始 ICMPv6 套接字上发送路由器请求消息。这由 cap_sendmsg 服务完成。另一个私有服务 cap_script 用于生成并收集 `rtsold` 守护程序所需脚本的状态。为该程序实现的第三个也是最后一个服务是 cap_llflags。该服务负责获取指定接口上链路本地 IPv6 地址的标志。

**rtsold(8)** 是 Casper 服务中沙箱化程序的示例，它不需要实现通用的宽泛服务。

## Super Capsicumizer 9000

这个有趣名字背后隐藏着一颗小宝石。Super Capsicumizer 9000，或简称 Capsicumizer，是开源项目，它成功实现了使用 Capsicum 的沙箱启动器 [3]。Capsicumizer 受 AppArmor 启发。AppArmor 是强制访问控制系统，允许限制进程访问。配置文件通常在系统启动时加载到内核，以此约束进程。系统管理员可管理配置文件，这些文件描述了应用应当能访问哪些资源。

Capsicumizer 也基于配置文件。区别在于，我们不在内核中加载配置文件，而是在用户态运行 Capsicumizer，让 Capsicum 处理进程限制。模式使用 UCL 语法定义。

Capsicumizer 使用 libpreopen 库预先打开所有资源目录描述符，并在 capability 安全的 libc 包装器中使用它们。多亏了该库，应用将拥有所需的所有 capability。

目前，Capsicumizer 的局限在于它只允许限制文件系统的资源。遗憾的是，不支持定义或预配置网络访问。

Capsicumizer 是令人兴奋的项目，它已经允许我们在不修改应用的情况下为其构建沙箱。目前，它仅限于文件系统。将其与 Casper 结合将会很有趣。通过这种结合，我们将能够为大量应用构建沙箱，而无需修改其代码。

## 总结

FreeBSD Capsicum 框架仍在开发中，但已得到广泛应用。Casper 服务的改进，尤其是 cap_fileargs，开启了一大批可轻松沙箱化的新应用。像 Capsicumizer 这样的项目能让我们走到这样一步：想要分离单个进程的管理员无需触碰代码即可达成目标。

---

**Mariusz Zaborski** 是 Fudo Security 的 QA 与开发经理。他自 2015 年起获得 FreeBSD 提交权限，引以为豪。Mariusz 的主要兴趣领域是操作系统安全和底层编程。在 Fudo Security，Mariusz 带领团队开发最先进的解决方案，用于监控、记录和控制 IT 基础设施中的流量。2018 年，Mariusz 组织了波兰 BSD 用户组。业余时间，他喜欢写博客——<https://oshogbo.vexillium.org>。

## 参考文献

[1] Jonathan Anderson, Stanley Godfrey, Robert N. M. Watson, Toward Oblivious Sandboxing with Capsicum

[2] Mariusz Zaborski, FreeBSD 期刊 2018 年 5/6 月刊，“Capsicum—Just apply me!”

[3] <https://github.com/myfreeweb/capsicumizer>
