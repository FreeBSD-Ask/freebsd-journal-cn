# CloudABI

- 原文：[CloudABI: Pure capability-based security for UNIX](https://freebsdfoundation.org/our-work/journal/browser-based-edition/cloudabi/)
- 作者：**Ed Schouten**

去年年底，我业余时间在写一个分布式存储服务。它没有固定模式（不像数据库服务器用表格、文件系统用树状结构），而是让用户提供一段固件，由固件为存储于其中的数据赋予结构与意义。服务器本身能安全地分发、缓存、复制、回收与修复数据，但实际的事务处理依赖固件完成。

我最终放弃了这个项目——一如我多数业余项目——但它成了我尝试若干新技术的借口。我用 Raft 实现类 Paxos 的分布式共识，用 Dart 语言虚拟机解析并执行固件，用 Sodium 库做加密，用 Google Test 与 Google Mock 做单元测试，并用 FreeBSD 的 Capsicum 给服务加沙箱。

实现这一切时，我渐渐喜欢上 Capsicum。基于访问控制的安全框架（如 AppArmor）往往把让软件安全运行的重担推给用户，而用户未必了解应用的内部细节。AppArmor 策略一旦失灵，普通用户往往别无选择，只能将其禁用。Capsicum 把安全策略融入应用自身的设计，则解决了这一问题。应用无需任何额外配置即告安全。简洁而美好。

Capsicum 还有一点让我喜欢：它引导你写出可测试的软件。由于不再有任何全局命名空间，与系统的所有交互都必须通过句柄（即文件描述符）完成。再也无法通过硬编码路径名访问文件，也无法向网络上硬编码的主机发起连接。你可以说 Capsicum 强迫你使用依赖注入，在我看来这是好事。

## Capsicum 能扩展吗？

但 Capsicum 有一点让我困扰：我注意到它的扩展性逊于线性。我的意思是，随着应用规模增大，将其改造为在 Capsicum 下正确运行会越来越难。一旦给进程加沙箱，便指令内核禁用大量系统调用。例如，除 `*at()` 之外的所有文件系统相关系统调用都会立即返回 ENOTCAPABLE。一切直接或间接调用这些系统调用的代码都会因此表现不同。在大量使用库的应用中，沙箱化后哪些函数仍可安全使用最终会变得无法判定。我在让 Dart 虚拟机在沙箱中运行时就为此大费周章。

我发现这些问题甚至在使用 FreeBSD 核心库时就已显现。沙箱化后无法再创建 locale 对象，意味着无法按用户提供的字符集解析字符串。另一个例子是沙箱化后无法访问时区数据库。这意味着应用只能使用 UTC 与系统全局时区，前提是后者在沙箱化前已加载入内存。

我遇到的最糟糕的例子来自 Ports 树中安装的一个名为 libtomcrypt 的加密库。该库在 capabilities 模式之外只会为随机数生成器使用强熵。进入 capabilities 模式后，由于再也无法访问 **/dev/urandom**，它会悄然退化为以系统时钟作为唯一的熵源。

## CloudABI：纯粹基于能力的计算

为了更轻松地编写在此类沙箱环境中良好运行的软件，我着手开发了一个新的运行时环境，名为 CloudABI。CloudABI 与 FreeBSD 标准运行时不同之处在于，进程启动时即处于 capabilities 模式。这让我们能移除所有与 Capsicum 不兼容的系统调用。其影响在于，任何使用这些不兼容特性的代码都极易被察觉：根本无法构建。乍听之下颇为激进，但我的经验是这反而节省了大量时间。修复构建失败往往比追踪因在标准运行时中启用 Capsicum 而导致的回归要简单得多。

拥有纯粹基于能力的运行时的另一个有趣之处在于，可以直接在内核之上运行不受信任的第三方应用，而无需任何复杂的访问控制列表、安全策略、虚拟机或容器。访问权限完全通过在执行前调整文件描述符的权限来控制。它让你能创建“微型计算环境”，进程只能通过一组受限的 RPC 与环境通信。这使其成为安全微服务架构的有趣构件，CloudABI 之名也由此而来。

一旦移除所有与 Capsicum 冲突、仅为遗留原因而保留、执行管理任务、或因其他原因具备特权或冗余的系统调用，UNIX ABI 会变得非常紧凑。FreeBSD 默认启用 338 个系统调用，而 CloudABI 只有 58 个。这让为既有操作系统内核添加 CloudABI 支持变得容易，使同一份可执行文件无需重新编译即可在多个系统上运行。

CloudABI 已完整移植到 FreeBSD 与 NetBSD。两份移植都仅约 5,000 行代码。还有一个实验性的 Linux 移植。

## 构建第一个 CloudABI 程序

既然知道 CloudABI 是什么了，该动手看看如何构建并运行自己的 CloudABI 程序。先从一个简单的“Hello, world”应用开始。我们会看到，即便如此简单的应用，也能让我们对 CloudABI 在实践中如何运作有诸多洞察。

由于 CloudABI 的运行时与 FreeBSD 原生编程环境相互独立，第一步是安装一套交叉编译工具链，用来生成 CloudABI ELF 可执行文件。为此，只需从 FreeBSD ports 树安装 cloudabi-toolchain 软件包。该软件包安装一份未经修改的 LLVM 3.7，并创建一个从 x86_64-unknown-cloudabi-cc 指向 Clang 可执行文件的符号链接。通过此符号链接调用时，Clang 足够聪明，能自动检测到它正作为 CloudABI 的交叉编译器被调用。它还安装了一份针对 CloudABI 的 GNU Binutils 副本。

目前 cloudabi-toolchain 软件包仅安装 x86-64 的交叉编译器。待 CloudABI 移植到更多硬件平台后，该软件包会扩展，设置更多符号链接，方便为任意架构构建软件。

```sh
$ sudo pkg install cloudabi-toolchain
$ cat hello.c
#include <stdio.h>
int main(int argc, char *argv[]) {
    printf("Hello, world\n");
    return 0;
}
$ x86_64-unknown-cloudabi-cc -o hello hello.c
hello.c:1:10: fatal error: 'stdio.h' file not found
```

安装工具链软件包后，尝试编译我们的简单程序，会发现构建仍失败，因为编译器找不到 `<stdio.h>` 头文件。原因是 FreeBSD ports 树中的 CloudABI 工具链并不附带任何为 CloudABI 构建的库。它只包含你想在 FreeBSD 系统上运行的开发工具。但别担心，我们无需手动交叉编译任何 CloudABI 核心库。

CloudABI 项目有自己的 ports 集合，名为 CloudABI Ports。为 CloudABI 单独维护一套 ports 集合的初衷在于：它让我们不仅能生成 FreeBSD 的软件包，也能生成其他 BSD 与各种 Linux 发行版的软件包。这些软件包含有完全相同的交叉编译二进制与库，但被打包成可由系统原生包管理器安装与升级的形式。即便你的操作系统不运行 CloudABI 可执行文件，也应当能在其上开发 CloudABI 软件。

如何把 CloudABI 软件包仓库加入 `pkg` 配置，可在 CloudABI Ports 的 GitHub 页面（<https://github.com/NuxiNL/cloudabi-ports>）找到。这里不展开描述这一过程，免得你照抄那串用于校验软件包的 PEM 文件。添加仓库后，便应能安装 x86_64-unknown-cloudabi-c-runtime 软件包，获得一组 C 编程的基础库，例如 CloudABI 的 C 库。C++ 开发则可安装 x86_64-unknown-cloudabi-cxx-runtime 软件包。

```sh
$ sudo pkg update
$ sudo pkg install x86_64-unknown-cloudabi-c-runtime
$ x86_64-unknown-cloudabi-cc -o hello hello.c
hello.c:3:3: warning: implicitly declaring library function
'printf' with type 'int (const char *, ...)'
hello.c:(.text.main+0x1c): undefined reference to `printf'
```

安装 C 运行时后，编译器能找到 `<stdio.h>`。构建现在会继续，但仍有一个恼人的链接错误：找不到 `printf()` 函数。后文会看到，文件描述符 0、1、2 并无固定用途，因此 stdout 及其辅助函数并不存在。眼下，我们假定应用直接从 shell 启动，文件描述符 1 对应 TTY。可用 POSIX 的 `dprintf()` 函数直接向该文件描述符打印。

```sh
$ cat hello.c
#include <stdio.h>
int main(int argc, char *argv[]) {
    dprintf(1, "Hello, world\n");
    return 0;
}
$ x86_64-unknown-cloudabi-cc -o hello hello.c
```

既然代码能构建了，接下来运行程序。为此，需确保加载一对内核模块。它们已于 svn revision 286680 完整导入 FreeBSD 源码树，意味着任何 2015 年 8 月 13 日或之后的 FreeBSD 11.0-CURRENT 快照都默认附带这些内核模块。

约 85% 的 CloudABI 系统调用在设计上跨硬件架构行为一致。它们依赖的常量、数据类型与结构完全相同。这些与机器无关的系统调用由 cloudabi 内核模块提供。

对于依赖硬件特定属性（例如指针为 64 位）的系统调用，我们提供名为 cloudabi64 的独立内核模块。该内核模块还提供将 64 位 ELF 可执行文件载入内存并开始执行所需的代码。

如此分离的好处在于：将来 CloudABI 若也支持 32 位架构，所有与架构无关的系统调用实现皆可复用。cloudabi32 内核模块只需实现 ABI 中的一小部分。

```sh
$ ./hello
ELF binary type "17" not known.
$ sudo kldload cloudabi64
$ kldstat
Id Refs Address            Size     Name
1    4 0xffffffff80200000 1e7f940  kernel
2    1 0xffffffff82221000 5a61     cloudabi64.ko
3    1 0xffffffff82227000 b167     cloudabi.ko
$ ./hello
Hello, world
```

## 用 cloudabi-run 安全启动程序

纯粹基于能力的运行时环境带来一个后果：进程以正确的文件描述符集合启动变得极为重要。我们必须精确控制新进程可见的文件描述符。意外泄漏到新进程的、未使用的文件描述符会赋予进程超出严格所需的权限。在类 UNIX 系统上，通常通过文件描述符上的 close-on-exec 标志加以控制，但众所周知这些标志几乎从未被设到正确值。

程序越复杂，以正确的文件描述符集合启动它也越难。例如，一个 Web 服务器只能监听单个网络套接字、服务单目录中的文件时，仍可使用固定的文件描述符表布局。一旦该 Web 服务器要支持监听多个套接字、为多个虚拟主机服务文件，并向冗余的数据库后端池发送查询，正确启动进程就变得棘手。进程必须以某种方式被告知文件描述符表的布局。

为此，CloudABI 自带一个名为 `cloudabi-run(1)` 的工具。简言之，该工具解析一份用 YAML 写就的配置文件，并将其内容传给 CloudABI 程序。配置文件中可包含特殊标签，指示 cloudabi-run 把条目替换为指向套接字、文件或目录的文件描述符。只有这些文件描述符会传给程序。进程可通过遍历配置数据获取文件描述符编号。这意味着程序无需假设某些文件描述符编号为某用途保留。YAML 数据取代了传统的命令行参数，可在程序内通过替代入口点 `program_main()` 访问。

让我们通过创建一个微型 Web 服务器来实地看看 cloudabi-run 如何运作。该 Web 服务器的配置文件采用极简格式：一组键值对列表。目前只支持名为 `socket` 的单一配置项，应持有一个指向网络套接字的文件描述符。最终目标是支持更多配置选项，例如 `rootdir` 选项用于提供指向 Web 服务器根目录的文件描述符。让它运转起来就留作练习吧。

程序通过 `program_main()` 启动后，我们用 `argdata_iterate_map()` 遍历所有键值对。`parse_config()` 回调从 `socket` 条目中提取文件描述符编号。拿到网络套接字后，我们不断对其调用 `accept()` 以响应传入的网络连接。为保持示例简单，我们只向客户端返回固定响应。

配置文件中我们用特殊标签 `!socket` 指示 cloudabi-run 创建一个监听所有 IPv4 地址、使用 TCP 端口 12345 的网络套接字。它还演示了如何用 `!file` 标签提供对文件与目录的访问。启动 Web 服务器后运行 `procstat -f` 可确认，进程中只有配置文件中列出的文件描述符（和 `accept()` 正在创建中的那个）。

这个例子妙在展示了使用 cloudabi-run 后，程序中能省去多少启动代码。无需再写配置文件解析器，cloudabi-run 已替你做完所有苦差事。Web 服务器也无需再创建网络套接字。

```sh
$ cat webserver.c
#include <sys/socket.h>
#include <argdata.h>
#include <program.h>
#include <stdbool.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

static bool parse_config(
    const argdata_t *key, const argdata_t *value,
    void *sockfd) {
  const char *keystr;
  if (argdata_get_str_c(key, &keystr) == 0 &&
      strcmp(keystr, "socket") == 0) {
    argdata_get_fd(value, sockfd);
  }
  return true;
}

void program_main(const argdata_t *ad) {
  int sockfd = -1;
  argdata_iterate_map(ad, parse_config, &sockfd);
  int connfd;
  while ((connfd = accept(sockfd, NULL, NULL)) >= 0) {
    /* TODO: 解析请求，从磁盘提供数据 */
    const char buf[] = "HTTP/1.1 200 OK\r\n"
        "Content-Type: text/plain\r\n\r\n"
        "Content-Length: 13\r\n\r\n"
        "Hello, world\n";
    write(connfd, buf, sizeof(buf) - 1);
    close(connfd);
  }
  exit(1);
}
$ x86_64-unknown-cloudabi-cc -o webserver webserver.c
$ cat config.yaml
%TAG ! tag:nuxi.nl,2015:cloudabi/
---
hostname: mywebserver.com
rootdir: !file
  path: /var/www/mywebserver.com
socket: !socket
  bind: 0.0.0.0:12345
logfile: !fd stdout
$ pkg install cloudabi-utils
$ cloudabi-run webserver < config.yaml &
[1] 869
$ procstat -f 869
PID COMM      FD T ... PRO NAME
...
869 webserver  0 v ... -   /var/www/mywebserver.com
869 webserver  1 s ... TCP 0.0.0.0:12345 0.0.0.0:0
869 webserver  2 v ... -   /dev/pts/0
869 webserver  3 ? ... -   -
$ fetch -qo - http://127.0.0.1:12345/
Hello, world
```

示例中我们把 Web 服务器绑定到 IPv4 地址，但 cloudabi-run 也允许你使用 IPv6 或 UNIX 套接字，而无需对 Web 服务器做任何改动。测试 Web 服务器时 UNIX 套接字可能有用。若我们想以自定义参数创建套接字（例如不同的超时与重传策略、绑定到特定地址），这些功能都可加入 cloudabi-run，意味着任何通过 cloudabi-run 启动的应用都无需改代码即可获得这些功能。

## 结语

既然已经看到如何在 CloudABI 上构建并运行自己的应用，不妨去 CloudABI Ports 仓库看看已有哪些软件被移植。已有大量库可供你创建颇为复杂的应用，但我相信你能想出仍缺失的软件包。CloudABI Ports（及其他 CloudABI 组件）的任何贡献都欢迎之至。

## 链接

- CloudABI on GitHub：<https://github.com/NuxiNL/cloudlibc>
- CloudABI Ports Collection on GitHub：<https://github.com/NuxiNL/cloudabi-ports>

**作者简介**

**ED SCHOUTEN** 自 2008 年起担任 FreeBSD 项目的开发者。他的贡献包括 FreeBSD 8 的 SMP 安全 TTY 层、Clang 在 FreeBSD 9 中的初次导入、最终进入 FreeBSD 10 的 `vt(4)` 控制台驱动初始版本。

CloudABI 由作者位于荷兰的公司 Nuxi 开发。它将始终作为开源软件免费提供。Nuxi 为 CloudABI 提供商业支持、咨询与培训。若你有意在产品中使用 CloudABI，请通过 <info@nuxi.nl> 与 Nuxi 联系。
