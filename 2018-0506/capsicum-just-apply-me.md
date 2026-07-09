# Capsicum：动手应用

应用沙箱一直显得很困难，这有历史原因。例如，使用 Linux `seccomp(2)`——最古老的沙箱技术之一——需要大量精力。`seccomp(2)` 最简单的模式——`SECCOMP_SET_MODE_STRICT`——将程序限制到不可用状态。另一方面，最流行的模式——`SECCOMP_SET_MODE_FILTER`——应用和维护都很难，也很容易反过来伤害我们。本文讨论 FreeBSD 内置的沙箱技术 Capsicum。如果读者对不同沙箱技术有更深入的对比感兴趣，作者推荐几篇文章 [1][2][3]。这里我们讨论 Capsicum 是什么、我们有哪些工具，以及最重要的——如何在应用中使用它。

## Capsicum 概览

Capsicum 基础设施可分为两部分：

- 严格沙箱
- 能力权限

严格沙箱意味着我们无法访问任何全局命名空间。全局命名空间指操作系统中有限的区域，这些区域有一组名称，允许无歧义地标识对象 [4]。简化来说，我们无法用文件路径这样的标识符操作对象。这个严格沙箱仍允许我们用对象的句柄操作。在类 UNIX 操作系统中，我们有一个通用句柄——描述符——可以用文件描述符操作文件。进程也一样。在这种模式下，我们无法用 PID 操作进程，但可以用进程描述符操作 [5]。在 Capsicum 中，这种模式称为能力模式（Capability mode）。在 FreeBSD 中，我们有一个简单的系统调用进入此模式——`cap_enter(2)`。

借助 Capsicum，我们更进一步，允许限制描述符。如果我们有一个描述符，知道它只用于读取，可以设置特定权限（`CAP_READ`），确保该描述符只读。如果尝试写入该描述符，会失败。我们将这些限制称为能力权限（capability rights）。总是可以进一步限制描述符，但无法扩展给定描述符的能力权限。这意味着如果我们有一个带 `CAP_READ` 和 `CAP_WRITE` 能力的描述符，某时决定不再读取此描述符，可以放弃该能力。出于显而易见的原因，不允许扩展它们。在 FreeBSD 中，我们有一个特殊函数——`cap_rights_limit(2)`——允许我们限制描述符。目前我们有 79 项权限，允许细粒度限制处理程序。

借助能力模式和能力权限，我们可以确保进程只能访问它真正需要的对象。这消除了环境权限问题——任何进程都能访问所有用户数据。如果攻击者利用 `grep(1)`、`patch(1)` 甚至 `cat(1)` 这样的工具，他会获得所有用户数据的访问权。攻击者还可以创建到任意服务器的新连接并发送这些文件。在 Capsicum 世界中，即使攻击者能利用这些工具中的任何一个，他也只能对磁盘上的几个文件有只读访问权。他无法覆盖任何重要数据或通过网络发送。

## 扩展能力

我们有两种方法获取能力：

- 初始化阶段
- 从其他进程获取

第一种情况，我们在进入能力模式之前预打开应用所需的全部连接、文件等，只能操作这些能力。此方法适合功能有限的小型应用。

第二种方法是使用另一个已有我们所需对象能力权限的进程。因为所有对象都由描述符表示，我们可以用 UNIX 域套接字轻松地从一个进程传递到另一个进程。如果一个进程有与服务器通信的描述符，它可以传递给另一个进程，两个进程都可以操作该单一描述符。

Capsicum 的一个非常常见的模式是使用两个进程。第一个具有环境权限，可以代表用户操作；第二个被沙箱化。环境权限进程仅用于简单的事，如打开文件列表或连接到服务器。沙箱进程执行应用的所有复杂逻辑，如解析。特权进程通过 UNIX 域套接字与沙箱进程连接，简单地服务下一个要解析的文件描述符。如果攻击者利用了我们的解析器（这很常见），他无法访问系统的任何其他部分——只能访问另一个只读文件。

## Capsicum 化

Capsicum 化是我们用来描述对现有应用进行沙箱化的有趣名称。目前在 FreeBSD 中我们有大约 60 个沙箱化应用。从非常简单的如 `yes(1)`，到稍复杂的如 `jot(1)`，再到更复杂的如 `tcpdump(1)` 和 `ping(8)`。沙箱应用数量不少，但我们仍一直在为更多应用做沙箱化。完整的应用列表和当前进度可以在 Capsicum wiki [6] 上查看。在示例 1 中，我们有一个 capsicum 化 `cmp(1)`（比较两个文件的程序）的补丁。在此补丁中，我们用一种方法在进入能力模式之前预打开所需的所有文件。`cmp(1)` 在两个描述符 fd1 和 fd2 上工作，两者都有能力 `CAP_FSTAT` 和 `CAP_FCNTL`，分别允许我们用 `fstat(2)` 函数获取文件状态和用 `fcntl(2)` 进行文件控制。（`cmp(1)` 使用 `fdopen(3)` 函数，需要 `fcntl(F_GETFL)`。在原始补丁中，我们还用 `cap_fcntls_limit(2)` 限制 `fcntls(2)` 的数量，但这超出了本文范围。）我们还给描述符 `CAP_MMAP_R` 能力，允许我们将文件映射到内存。最后，我们限制 stdout 描述符并预缓存本地语言支持数据（NLS）。

最后，我们进入能力模式。

这非常简单。对于这个特定应用，我们甚至不需要重组代码，因为所有文件在解析前已在一个地方打开，所以初始化阶段很容易找到。然后我们限制了一些描述符并进入能力模式。

```sh
--- usr.bin/cmp/cmp.c
+++ usr.bin/cmp/cmp.c
@@ -68,2 +71,5 @@ main(int argc, char *argv[])
         const char *file1, *file2;
+        cap_rights_t rights;
+        unsigned long cmd;
+        uint32_t fcntls;

@@ -148,2 +154,19 @@ main(int argc, char *argv[])

+        cap_rights_init(&rights, CAP_FCNTL, CAP_FSTAT, CAP_MMAP_R);
+        if (cap_rights_limit(fd1, &rights) < 0 && errno != ENOSYS)
+                err(ERR_EXIT, "unable to limit rights for %s", file1);
+        if (cap_rights_limit(fd2, &rights) < 0 && errno != ENOSYS)
+                err(ERR_EXIT, "unable to limit rights for %s", file2);
+
+        cap_rights_init(&rights, CAP_FSTAT, CAP_WRITE, CAP_IOCTL);
+       if (cap_rights_limit(STDOUT_FILENO, &rights) < 0 && errno != ENOSYS)
+                err(ERR_EXIT, "unable to limit rights for stdout");
+
+        /*
+         * 在进入能力模式之前，缓存 NLS 数据（用于 strerror、err(3)）。
+         */
+        (void)catopen("libc", NL_CAT_LOCALE);
+
+        if (cap_enter() < 0 && errno != ENOSYS)
+                err(ERR_EXIT, "unable to enter capability mode");
+
         if (!special) {
```

示例 1. Capsicum 化 `cmp(1)`，由 Conrad E. Meyer 提出的简化补丁。

## Capsicum 辅助函数

在示例 1 中，我们有一个变化，第一次看到时并不明显——调用 `catopen(3)` 缓存 NLS 数据。通常当我们第一次调用 `err(3)` 函数时，它会去文件系统并为 NLS 打开一个文件。不幸的是，在兼容模式下这无法完成，因为它无法打开文件。

libc 中还有一个预缓存事物的已知问题，涉及操作时间。像 `localtime(3)` 这样的函数——当我们第一次调用它们时——会预缓存机器的当前时区，并且只在那一次执行。

由于这些不明确的行为，创建了 Capsicum 辅助函数。Capsicum 辅助函数是一组小型内联函数，允许我们预缓存事物，如时区和 NLS 数据。它们的 man 页面也是记录这种不明确行为的好地方。对于缓存时区和 NLS，我们会找到两个函数：`caph_cache_tzdata(3)` 和 `caph_cache_catpages(3)`。

Capsicum 辅助函数的另一个用途是减少应用所需的代码量。回到示例 1，我们可以看到我们在限制 stdout。问题是多少应用需要限制 stdio——可能是大多数——这就是为什么引入了 `caph_limit_stdin(3)`、`caph_limit_stdout(3)`、`caph_limit_stderr(3)` 和 `caph_limit_stdio(3)` 函数。这些函数允许我们用最常见的权限限制单个描述符，并用应用特定的权限限制其他描述符。如果默认限制对我们的程序足够，我们可以简单地调用 `caph_limit_stdio(3)` 函数，一次性限制所有描述符。

另一个非常常见的模式是调用 `cap_enter(2)` 函数，如果函数失败则检查 ERRNO。目的是检查进入能力模式失败是因为发生了什么，还是我们的操作系统不支持。第一种情况，应用应该在此停止工作。第二种情况，它应该仍能运行，因为我们的整个系统不支持。起初，这个检查可能看起来反直觉，不检查 ERRNO 很不常见。这就是为什么我们正在开发另一个函数 `caph_enter(3)`，它会隐藏这个检查 [7]。

## 调试工具

当沙箱被添加到应用中时，开发者可能无法注意到程序的某些条件。应用可能使用开发者不太了解的库。例如，如果应用使用一个库，而这个库通过打开 **/dev/random** 使用随机数生成器（如果可能），否则使用不安全的随机数生成器。如果开发者通过分析代码没有注意到这种行为，这可能在沙箱化应用时引入新错误。这就是为什么提供有用的调试工具是构建沙箱机制的最大挑战之一。对于 FreeBSD 中的 Capsicum，我们有两个工具：

- `ktrace(1)`/`kdump(1)`
- 带 TRAPCAP 的 gdb

在沙箱化过程中，开发者可以用 `ktrace(1)` 运行程序，检查是否没有返回 `ECAPMODE` 或 `ENOTCAPABLE`。这可能给开发者带来一些问题，因为有时很难知道哪个被调用的函数失败了。也很难覆盖所有可能的运行路径。在我们之前的例子中，这个打开 **/dev/random** 的库函数仅在程序中提供的某个特定选项中使用。然而，程序有很多选项，很容易遗漏。这也是为什么回归测试如此重要。不幸的是，在这种情况下，我们需要在 `ktrace(1)` 下运行整个测试套件并分析其输出。在示例 2 中，我们有 `ktrace(1)` 的示例输出。

```sh
802 random  CALL  cap_enter
802 random  RET   cap_enter 0
802 random  CALL  openat(AT_FDCWD,0x400877,0<O_RDONLY>)
802 random  CAP   restricted VFS lookup
802 random  RET   openat -1 errno 94 Not permitted in capability mode
802 random  CALL  exit(0)
```

示例 2. 对进入能力模式并尝试打开 **/dev/random** 的程序进行 `kdump(1)` 的结果。

另一种分析程序的方法是使用 Capsicum 的新调试功能，由 Konstantin Belousov 在 FreeBSD 基金会 赞助下实现。借助此功能，当返回 `ECAPMODE` 或 `ENOTCAPABLE` 时，内核会发出 `SIGTRAP`。结果，我们会在错误发生的那一刻得到核心转储。这使得更难忽略某些错误，因为我们的程序会中止，我们会在运行时注意到。核心转储还提供了更多关于错误发生时进程状态的信息。我们可以用 `sysctl kern.trap_enotcap`（全局系统中）启用此功能。如果需要按进程启用，可以用 `proctl(2)`。为了让系统能够在能力模式下生成核心转储，我们还需要设置 `sysctl capmode_coredump`，否则沙箱中的程序无法创建核心转储。

## nvlist 库

我们已经讨论了在进程中获取更多能力的一些方法。一种方法是从另一个进程接收它们。为了更容易地在特权和非特权进程之间拆分程序，Capsicum 开发者还引入了一个非常简单的 IPC 库——nvlist。此库基于包含键值对（名称，值）的列表，允许我们保留许多原语，如数字、字符串、二进制和布尔值。然而，最特别的是它还允许我们在列表上保留描述符。此外，它提供了通过套接字发送和接收 nvlist 的函数。所有这些都是为了允许进程分离和更轻松地 Capsicum 化而设计的。

nvlist 作为容器也存在于内核中，并被一些驱动程序使用（如 ixl）。其实现使用与用户态完全相同的代码——不包含内核中不存在的原语（如描述符或套接字）。在示例 3 中，我们可以看到 nvlist 的简单使用。作者建议将 nvlist 视为序列化库的候选，因为其实现和使用非常简单。如果你对 nvlist 的用例感兴趣，请参阅 man 页面（`nv(9)`）或一些外部资料 [9][10][11]。

```sh
nvlist_t *nvl;

nvl = nvlist_create(0);
nvlist_add_string(nvl, "first", "foo");
nvlist_add_number(nvl, "second", 1234);

/*
 * nvlist 中同样有趣的是，我们只需检查 nvlist 上的最后一次操作。
 * 如果之前的某次添加失败，我们会在任何时候知道。
 */
if(nvlist_send(sock,nvl)<0){
        fprintf(stderr, "Unable to send nvlist.\n");
        exit(1);
}
```

示例 3. nvlist 的简单使用。添加两个值并通过网络发送。

## Casper 概览

在将程序拆分为特权和非特权进程时，我们会注意到一些常见模式。例如，许多网络工具需要访问 DNS 服务器。在能力模式下，我们无法直接连接到服务器，需要另一个能与之通信的进程。为简化此过程并减少所有应用所需的代码量，创建了 Casper 库。此库为我们提供了一组服务，如：

- system.dns——获取网络主机条目的服务
- system.grp——组数据库操作的服务
- system.pwd——密码数据库操作的服务
- system.random——获取熵的服务
- system.sysctl——获取或设置系统信息的服务
- system.syslog——syslog 服务

创建 Casper 实例时，它会从原始进程 fork，这要求在进入能力模式之前创建 Casper 服务（`cap_init(3)`）。我们也可以从另一个拥有服务的进程接收服务。所有服务都有良好的文档，在 man 页面中我们可以找到使用示例。

目前我们正在开发一个服务——system.fileargs。此服务的目标是提供一个简单的工具，用于沙箱化以文件列表为参数的应用。此服务将通过类似 `open(2)` 的 API 提供描述符。借助此接口，将其应用到现有应用应该很简单。虽然此服务仍在开发中，项目同意将其最初作为实验性服务处理 [8]。

我们还计划实现其他服务，如：

- system.login——访问登录类能力数据库的服务
- system.tls——用 TLS/SSL 创建安全连接的服务
- system.socket——创建网络连接的服务
- system.configuration——获取统一配置的服务

这些目前都只是想法。

## Casper 与 dhclient(8)

新的服务之一是 system.syslog。让我们讨论一下为什么创建它。

在启用 `kern.trap_enotcap` sysctl 引导操作系统时，我们注意到 `dhclient(8)` 在进行核心转储。示例 4 展示了这种情况。

```sh
Starting devd.
Starting dhclient.
pid 336 (dhclient), uid (65): Path `/var/crash/dhclient.65.0.core'
failed on initial open test, error = 2
pid 336 (dhclient), uid 65: exited on signal 5
Trace/BPT trap/etc/rc.d/dhclient: WARNING: failed to start dhclient
Starting syslogd.
```

示例 4. `dhclient(8)` 在启动时进行核心转储。

```sh
void syslog(int priority, const char *message, ...);
void vsyslog(int priority, const char *message, va_list args);
void openlog(const char *ident, int logopt, int facility);
void closelog(void);
int setlogmask(int maskpri);
```

示例 5. syslog API

分析此程序后发现，dhclient 用 syslog 报告其状态。如果我们再看示例 4，会看到 `syslogd(8)` 在 `dhclient(8)` 之后启动。这是典型的鸡生蛋蛋生鸡问题。`syslogd(8)` 有时需要网络配置，而 `dhclient(8)` 需要 `syslogd(8)` 来报告状态。历史上，我们决定在运行 `syslogd(8)` 之前运行 `dhclient(8)`。

值得注意的是，`dhclient(8)` 在进入能力模式之前尝试连接到 `syslogd(8)`，因为服务器不存在而失败。令人惊讶的是，每次程序未连接时，syslog 函数都尝试连接它！当然，因为我们现在在 Capsicum 中运行，永远无法建立连接。因此，为解决此问题，我们决定引入一个新的 Casper 服务 syslog。它尝试连接到 `syslogd(8)`，如果失败，会在有内容要报告时再试。也值得注意的是，syslog API（如示例 5 所示）在出问题时不会报告任何问题。

## 总结

沙箱化基本系统是一个持续的过程。我们引入了许多工具，应该能降低希望使用 Capsicum 的新人的门槛。

Capsicum 化是学习操作系统的好方法——它们如何工作、如何交互、一些库如何行为。仍有许多小程序等待 Capsicum 化。这可以成为关于操作系统的绝佳一课——尤其对于那些梦想成为 FreeBSD 开发者的人。

---

MARIUSZ ZABORSKI 是 Wheel Systems 的首席软件开发者。他自 2015 年起自豪地拥有 FreeBSD commit bit。Mariusz 的主要兴趣领域是操作系统安全和底层编程。在 Wheel Systems，Mariusz 领导一个团队，开发监控、记录和控制 IT 基础设施流量的最先进解决方案。业余时间，他喜欢写博客（<http://oshogbo.vexillium.org>）。

## 参考文献

[1] J. Anderson, A Comparison of Unix Sandboxing Techniques, FreeBSD 期刊 Sept/Oct 2017
[2] P.Dawidek, M.Zaborski, Sandboxing with Capsicum, ;login: issue:December 2014, Vol. 39, No. 6
[3] M.Zaborski, Capsicum and Casper - a fairy tale about solving security problems, AsiaBSDCon 2016 <http://oshogbo.vexillium.org/pdf/AsiaBSDcon2016.pdf>
[4] <http://en.wikipedia.org/wiki/Namespace>
[5] Robert N.M. Watson, Jonathan Anderson, Ben Laurie, Kris Kennaway, Introducing Capsicum: Practical Capabilities for UNIX, 2010
[6] FreeBSD wiki, Capsicum page, <https://wiki.freebsd.org/Capsicum>
[7] Introduce caph_enter(), FreeBSD phabricator, <https://reviews.freebsd.org/D14557>
[8] Introduce system.fileargs, FreeBSD phabricator, <https://reviews.freebsd.org/D14407>
[9] Introduction to nvlist part 1, <http://oshogbo.vexillium.org/blog/42/>
[10] Introduction to nvlist part 2 - dnvlist, <http://oshogbo.vexillium.org/blog/43/>
[11] Introduction to nvlist part 3 - simple traversing, <http://oshogbo.vexillium.org/blog/45/>
