# CloudABI 的最新进展

CloudABI？

FreeBSD 期刊 2015 年 9/10 月刊刊登了一篇关于 CloudABI 的文章，这是我同年早些时候开始工作的一个开源项目。距离那篇文章发表已经过去相当一段时间了，让我们来看看此期间发生的一些进展以及接下来的计划。但首先，回顾一下 CloudABI 是什么以及使用它是什么样子。

## 理论上的 CloudABI

CloudABI 是一个运行时，你可以为它开发类 POSIX 软件。CloudABI 与 FreeBSD 原生运行时的不同之处在于，它强制程序使用依赖注入。依赖注入是一种通常用于面向对象编程领域的技术，使程序的组件（类）更易于测试和重用。你设计的类不再直接尝试与外部世界通信，而是将通信通道表示为类可执行操作的其他对象（依赖项）。

依赖注入的一个好例子是 Web 服务器类，它依赖于在构造函数中传入的独立套接字对象。这样的类可以很容易地进行单元测试——传入一个模拟套接字对象，生成一个虚拟 HTTP 请求并验证 Web 服务器的响应。这个类也是可重用的，因为对不同网络协议（IPv4 对 IPv6、TCP 对 SCTP）、加密（TLS）、速率限制、流量整形等的支持都可以作为独立的辅助类添加。Web 服务器类本身的实现可以保持不变。一个通过 **socket(2)** 系统调用直接创建自己网络套接字的 Web 服务器类则无法做到这一点。CloudABI 背后的目标是在更高层面引入依赖注入。我们不将其应用于类，而是要强制整个程序在启动时显式注入所有依赖项。在我们的案例中，依赖项由文件描述符表示。程序不再能使用绝对路径名打开文件。文件只能通过注入对应于文件本身或其某个父目录的文件描述符来访问。程序也不再能绑定到任意 TCP 端口。可以向它们注入预绑定的套接字。这个模型的妙处在于它减少了对 Jail、容器和虚拟机的需求。这些技术常用于克服由于类 Unix 系统默认缺乏隔离而无法并行运行同一软件的多种配置/版本的问题。使用 CloudABI，只需为程序的每个实例注入不同的文件、目录和套接字集合即可获得这种隔离。操作系统可以很容易地对遵循此方法的进程进行沙箱化。通过要求注入所有依赖项，操作系统可以简单地拒绝访问其他一切。无需像 Linux 的 SELinux 和 AppArmor 等框架那样单独维护安全策略。如果攻击者设法控制了基于 CloudABI 的进程，他/她实际上只能访问进程设计上需要交互的资源。CloudABI 受到 Capsicum（FreeBSD 基于能力的安全框架）的很大影响。CloudABI 与 Capsicum 的不同之处在于，Capsicum 仍允许你在未启用沙箱的情况下运行启动代码。进程必须使用 **cap_enter(2)** 手动切换到“能力模式”。CloudABI 可执行文件在程序第一条指令执行时已被沙箱化。Capsicum 模型的优势在于可以将沙箱集成到传统 Unix 程序中，这些程序通常并非设计为注入所有依赖项。CloudABI 模型的优势在于它

允许我们移除所有与沙箱不兼容的 API。这大幅减少了移植软件的工作量，因为需要修改以适配沙箱的部分现在会触发编译器错误，而非可能难以触发乃至调试的运行时错误。移除所有这些与沙箱不兼容的 API 的一个副作用是，CloudABI 变得非常精简，其他操作系统可以相对容易地实现它。这意味着你可以使用 CloudABI 构建一个无需重新编译即可在多个平台上运行的可执行文件。

## 实践中的 CloudABI

为演示实际使用 CloudABI 的样子，让我们看一个 CloudABI 的小型 Web 服务器，它能向浏览器返回固定的 HTML 响应。

```c
#include <sys/socket.h>
#include <argdata.h>
#include <program.h>
#include <string.h>
#include <unistd.h>

void program_main(const argdata_t *ad) {
    // 从配置中提取套接字和消息。
    int sockfd = -1;
    const char *message = "";
    {
        argdata_map_iterator_t it;
        argdata_map_iterate(ad, &it);
        const argdata_t *key, *value;
        while (argdata_map_next(&it, &key, &value)) {
            const char *keystr;
            if (argdata_get_str_c(key, &keystr) != 0)
                continue;
            if (strcmp(keystr, "http_socket") == 0)
                argdata_get_fd(value, &sockfd);
            else if (strcmp(keystr, "html_message") == 0)
                argdata_get_str_c(value, &message);
        }
    }

    // 处理传入请求。
    // TODO: 实际处理 HTTP 请求。
    // TODO: 使用并发。
    for (;;) {
        int connfd = accept(sockfd, NULL, NULL);
        dprintf(connfd, "HTTP/1.1 200 OK\r\n"
            "Content-Type: text/html\r\n"
            "Content-Length: %zu\r\n\r\n"
            "%s", strlen(message), message);
        close(connfd);
    }
}
```

你可能立即注意到，CloudABI 程序通过名为 `program_main()` 的函数启动，而非使用 C 的标准 `main()` 函数。`program_main()` 函数摒弃了 C 的字符串命令行参数，替换为类似 YAML/JSON 的树状结构，称为 Argdata。除了存储布尔值、整数和字符串等值外，Argdata 还可附加文件描述符。这是在启动时注入依赖项的机制。此 Web 服务器期望 Argdata 是一个映射（字典），同时包含用于接受传入请求的套接字（`http_socket`）和 HTML 响应字符串（`html_message`）。以下 shell 命令展示了如何构建和执行此 Web 服务器。Web 服务器可使用 Port `devel/cloudabi-toolchain` 提供的交叉编译器编译。构建完成后，可使用 Port `sysutils/cloudabi-utils` 提供的 `cloudabi-run` 启动。`cloudabi-run` 实用程序从标准输入读取 YAML 文件并将其转换为 Argdata 树，传递给 `program_main()`。YAML 文件可包含 `!fd`、`!file` 和 `!socket` 等标签。这些标签是 `cloudabi-run` 的指令，用于在 Argdata 树的相应位置插入文件描述符。只有 Argdata 引用的文件描述符最终会进入 CloudABI 进程。

```sh
$ x86_64-unknown-cloudabi-cc -o webserver webserver.c
$ cat webserver.yaml
%TAG ! tag:nuxi.nl,2015:cloudabi/
---
http_socket: !socket
  type: stream
  bind: 0.0.0.0:8080
html_message: <marquee>Hello, world!</marquee>
$ cloudabi-run webserver < webserver.yaml &
$ curl http://localhost:8080/
<marquee>Hello, world!</marquee>
```

此示例表明，CloudABI 能以直观的方式构建强沙箱化的应用程序。通过传递给 `cloudabi-run` 的配置，此 Web 服务器与系统的其余部分完全隔离，唯一例外是它可在其上接受传入连接的 HTTP 套接字。通过使用 Argdata，我们还能省略 Web 服务器中的大量样板代码，如配置文件解析和套接字创建。所有这些

功能由 `cloudabi-run` 一次性实现并可普遍重用。

## 硬件架构

CloudABI 2015 年发布时，我们仅支持为 x86-64 创建可执行文件。由于我认为 CloudABI 也是在嵌入式系统和设备上进行软件沙箱化的有用工具，我们在上一篇 CloudABI 文章发表前后将 CloudABI 移植到 ARM64。2016 年 8 月，我们将 CloudABI 移植到这些架构的 32 位等价物（i686 和 ARMv6）。将 CloudABI 移植到这些系统的一个有趣方面是获取可用的工具链。当 CloudABI 仅支持 x86-64 时，我们已经使用 Clang 作为 C/C++ 编译器。Clang 的好处在于，单个安装可轻松用于针对多种架构。它可在启动时通过检查 `argv[0]` 自动推断要使用的架构。这意味着我们只需扩展现有的 Port `devel/cloudabi-toolchain`，为我们支持的每种架构安装指向 Clang 的额外符号链接。同时，我们仍使用 GNU Binutils 链接可执行文件。Binutils 的缺点是一个安装只能用于针对单一硬件架构。更糟的是，Binutils 代码库对每对操作系统和硬件架构都需要大量修改才能支持。大约在我们开始支持更多架构时，LLVM 项目在其自己的链接器 LLD 上取得了很大进展。LLD 的出色之处在于它基本上不含任何操作系统特定的代码。它只需使用在各种情况下都能良好工作的合理默认值，即可为许多基于 ELF 的操作系统生成二进制文件。与 GNU Binutils 相比，它还有更有利的许可证（MIT 对 GPLv3）。我们开始尝试 LLD 时，注意到仍有一些阻碍使我们无法立即使用它。链接过程中的一个重要步骤是链接器

应用重定位：存储在目标文件中的一系列规则，描述机器代码在链接到程序或库时需要如何调整以指向正确的变量和函数地址。我们观察到 LLD 错误地应用了若干类型的重定位，导致生成的可执行文件几乎立即访问无效内存地址。这是因为 LLD 开发者主要专注于让动态链接的可执行文件工作，而 CloudABI 使用静态链接。在提交 bug 报告和向上游发送各种补丁后，我们设法让 LLD 至少在 x86-64、i686 和 ARM64 上可靠工作。要获得完整的 ARMv6 支持，我们不得不等待 LLD 4.0 发布，因为 ARMv6 使用 LLD 尚不支持的 C++ 异常元数据自定义格式（EHABI）。LLD 对我们来说工作得如此之好，以至于有一次我们决定完全停止使用 GNU Binutils。与谷歌的 Fuchsia 操作系统一起，CloudABI 现在是完全切换到 LLD 的系统之一。Port `devel/cloudabi-toolchain` 现在安装基于 LLVM 4.0 的工具链，设置 `*-unknown-cloudabi-ld` 指向 LLD 的符号链接。

## 操作系统和模拟器

运行 CloudABI 程序的原始要求之一是你需要一个能够原生执行它们的操作系统内核。在 FreeBSD 上，这很容易实现，因为 FreeBSD 11 及更高版本默认附带相关内核模块（称为 `cloudabi32.ko` 和 `cloudabi64.ko`，都依赖于 `cloudabi.ko` 中的公共代码）。Linux 内核补丁集也已随时间成熟，但尚未上游化，意味着用户仍需安装自定义构建的内核。在 macOS 等系统上，安装修改过的操作系统内核是不可取的。为降低在这些系统上至少尝试 CloudABI 的门槛，我们开发了一个能够在未修改的类 Unix 操作系统之上运行 CloudABI 可执行文件的模拟器。模拟器通过将可执行文件映射到同一地址空间并跳转到其入口点来工作。代码原生执行，无需解释或动态重编译。系统调用最终调用模拟

器，由它转发给宿主操作系统。在此过程中，我们希望防止模拟器中任何可通过改进 CloudABI 本身来避免的复杂性。例如，64 位架构上的 CloudABI 可执行文件现在被要求为位置无关。虽然 HardenedBSD 和 OpenBSD 等系统主要对使用位置无关可执行文件（PIE）以允许地址空间布局随机化（ASLR）感兴趣，但我们将其视为保证 CloudABI 可执行文件可被模拟器映射而不与模拟器内部使用的地址范围冲突的有用工具。我们做的另一项改进是 CloudABI 可执行文件不再尝试通过 `int 0x80` 和 `syscall` 等特殊硬件指令直接调用系统调用。这很重要，因为我们不希望 CloudABI 可执行文件在被模拟时调用宿主系统的内核。它们必须调用模拟器。运行时现在需要在启动时向 CloudABI 可执行文件提供一个内存中共享库（虚拟动态共享对象，vDSO），为运行时支持的每个系统调用公开一个函数。在模拟器中运行时，vDSO 指向模拟器中的系统调用处理程序。原生运行时，内核向进程提供 vDSO，其中包含确实使用特殊硬件指令强制切换到内核模式的微型包装器：

```asm
ENTRY(cloudabi_sys_fd_sync)
    mov $15, %eax
    syscall
    ret
END(cloudabi_sys_fd_sync)
```

以这种方式使用 vDSO 的优势在于，随着时间的推移添加和移除系统调用变得容易得多。由于系统调用现在由字符串而非数字标识，第三方可以轻松地在不会与 CloudABI 系统调用集冲突的自定义前缀下放置扩展（例如 `acmecorp_sys_*`，而非 `cloudabi_sys_*`）。程序可在启动时通过扫描 vDSO 的符号表轻松检测哪些系统调用存在和缺失。

最后，我们还在 CloudABI 实现线程局部存储（TLS）的方式上做了一些改进。它现在设计为模拟器可更高效地在宿主和客户进程使用的上下文之间切换。这通过要求客户的线程控制块（TCB）始终保留指向宿主 TLS 区域的指针来实现。执行系统调用时，模拟器可通过从其客户的 TCB 中提取来临时恢复自己的 TLS 区域。以这种方法构建模拟器的目标之一是拥有一种简单的嵌入 CloudABI 程序执行的方式，对许多与模拟无关的目的也很有用。现在可以完全在用户空间设计类似 **truss(8)** 的系统调用追踪工具，而无需依赖 **ptrace(2)** 等特殊内核接口。其他有趣的用例包括多线程代码的用户空间死锁检测器，以及注入随机故障的模糊测试器。CloudABI 的用户空间模拟器已集成到 `cloudabi-run` 中，可通过传入 `-e` 命令行标志轻松启用。以下是在 macOS 上构建和运行 CloudABI 程序的记录。

```sh
$ cat hello.c
#include <argdata.h>
#include <program.h>
#include <stdio.h>
#include <stdlib.h>

void program_main(const argdata_t *ad) {
    int fd = -1;
    argdata_get_fd(ad, &fd);
    dprintf(fd, "Hello, world!\n");
    exit(0);
}
```

```sh
$ x86_64-unknown-cloudabi-cc -o hello hello.c
$ cat hello.yaml
%TAG ! tag:nuxi.nl,2015:cloudabi/
---
!fd stdout
$ cloudabi-run hello < hello.yaml
Failed to start executable: Exec format error
$ cloudabi-run -e hello < hello.yaml
Hello, world!
```

## 更好的 C++ 支持

CloudABI 最初开发时，我们的重点是让 C 编写的代码工作。在 CloudABI 的 C 库变得相对完整以及相当数量用 C 编写的软件包开始出现后，我们将重点转向改善将 C++ 编写的软件移植到 CloudABI 的体验。

早期我们设法让 LLVM 的 C++ 运行时库（libcxx、libcxxabi 和 libunwind）工作，但它们仍需要大量本地补丁。许多情况下，这些补丁是并非 CloudABI 特有的清理工作。它们使代码整体上更具可移植性。自上一篇文章发表以来，我们已能将几乎所有这些补丁集成。同时，我们还打包了 Boost——一个常用的 C++ 框架。我们移植到 CloudABI 的一个有趣的 C++ 软件是 LevelDB。LevelDB 是一个实现经过高度优化的有序键值存储的库，使用一种称为日志结构合并树的数据结构。它在谷歌内部被用作 BigTable 的构建块——BigTable 是支撑其许多 Web 服务的数据库系统。移植 LevelDB 真正展示了 CloudABI 的优势：通过省略任何与 Capsicum 不兼容的接口，我们能轻松找到需要修补才能与沙箱良好工作的代码部分。就 LevelDB 而言，它直接指向 `leveldb::Env`——实现所有文件系统 I/O 的类。我们通过更改此类使其持有文件系统操作应限制到的目录的文件描述符，使得沙箱化软件中使用 LevelDB 成为可能。你通常如下使用 LevelDB 的 API 访问数据库：

```c
leveldb::Options opt;
opt.create_if_missing = true;
opt.env = leveldb::Env::Default();
leveldb::DB *db;
leveldb::Status status = leveldb::DB::Open(opt, "/var/...", &db);
db->Put(leveldb::WriteOptions(), "my_key", "my_value");
```

你可以在 CloudABI 上这样使用 LevelDB：

```c
leveldb::Options opt;
opt.create_if_missing = true;
opt.env = leveldb::Env::DefaultWithDirectory(db_directory_fd);
leveldb::DB *db;
leveldb::Status status = leveldb::DB::Open(opt, ".", &db);
db->Put(leveldb::WriteOptions(), "my_key", "my_value");
```

有了这些现成可用的库，我们现在能够开始将整个程序移植到 CloudABI。Bitcoin 的主要开发者之一 Wladimir van der Laan 恰好发现，Bitcoin 所用的大多数库的

参考实现，如 Boost 和 LevelDB，已被我们移植和打包。因此，Wladimir 已成功将 `bitcoind` 移植到 CloudABI。此项目的初始目标是将 Bitcoin 与同一系统上运行的其他进程隔离。未来目标是使用 CloudABI 执行权限分离，使网络协议处理中的安全缺陷不能被攻击者利用来直接访问存储用户比特币的钱包。

## 运行沙箱化的 Python 代码

2015 年底，我在汉堡的 32C3 安全会议上做了一个关于 CloudABI 的演讲。演讲中，我简要提到我有计划将 Python 解释器移植到 CloudABI。这似乎给观众留下了深刻印象，因为会后不久我收到了 Alex Willmer 的邮件，主动提出帮忙。在接下来的几个月里，Alex 和我大量合作，为 Python 和 CloudABI 的 C 库提出补丁，让 Python 尽可能干净地构建。在让解释器构建成功后，Alex 致力于扩展 Python 的模块加载器 `importlib`，允许你使用目录文件描述符在 `sys.path` 中包含路径，而非使用

路径名字符串。最后，我致力于将 Python 解释器与 Argdata 集成，使其可通过 `cloudabi-run` 启动。以下是使用我们的 Python 副本运行简单“Hello, world”脚本的演示。在其生命周期中，解释器只能访问 Python 的模块目录、要执行的脚本以及脚本应写入消息的终端。

虽然以这种方式运行 Python 起初看起来有些复杂，但显式注入 Python 所有依赖项的要求确实有其优势——现在维护多个 Python 环境变得容易得多。每个环境可安装不同版本的第三方模块。这种方法实际上意味着不再需要使用 Python 自己的 `virtualenv` 来实现环境间的隔离。通过注入正确的文件描述符集合即可实现。当然，上面的示例只是为了触及使用 CloudABI 的 Python 副本的皮毛。在 CloudABI 开发博客上，你可以找到一篇文章解释如何使用 `socketserver` 和 `http.server` 构建你自己的沙箱化 Web 服务。同时，我们还在移植 Django Web 应用框架。一个能够处理请求的初步版本已经打包。

```sh
$ cat hello.py
import io
import sys

stream = io.TextIOWrapper(sys.argdata['terminal'], encoding='UTF-8')
print(sys.argdata['message'], file=stream)
$ cat hello.yaml
%TAG ! tag:nuxi.nl,2015:cloudabi/
---
path:
- !file
  path: /usr/local/x86_64-unknown-cloudabi/lib/python3.6
script: !file
  path: hello.py
args:
terminal: !fd stdout
message: Hello, world!
$ cloudabi-run \
  /usr/local/x86_64-unknown-cloudabi/bin/python3 < hello.yaml
Hello, world!
```

## 二进制接口的形式化

CloudABI 的所有低层数据类型和常量最初通过一组 C 头文件定义。这些头文件的问题是随着时间的推移变得相当复杂。由于我们允许在 32 位和 64 位系统上运行 32 位 CloudABI 可执行文件，我们不得不维护假设目标原生指针大小（用户态使用）或显式假设 32 位或 64 位指针（内核态使用）的定义副本。CloudABI 的系统调用表被翻译为 FreeBSD 的内核内格式并手动同步。为清理这些，Maurice Bos 开展了一个项目来形式化 CloudABI。所有 CloudABI 数据类型、常量和系统调用现在以编程语言无关的表示法在名为 `cloudabi.txt` 的文件中描述。以下是与 `cloudabi_sys_fd_read()` 系统调用相关的定义摘录：

## Argdata：现在是独立库

虽然 Argdata 最初设计仅用于向 CloudABI 程序传递启动配置，但它在底层实际上是一个相当灵活且高效的二进制序列化库。与类似的编码格式 MessagePack 相比，它允许高效随机访问数据而无需完全反序列化。与 FreeBSD 基本系统中序列化库 **libnv(3)** 相比，它具有更常规的数据模型和更简单的 API。今年早些时候，Maurice Bos 表示有兴趣在非 CloudABI 的 C++ 应用程序中使用 Argdata 作为序列化 RPC 消息的格式。由于 Argdata 库与 CloudABI 的 C 库紧密集成，Maurice 花了一些时间使其更具可移植性并将其转变为独立库。同时，他还为 Argdata 编写了出色的 C++ 绑定，利用了现代语言版本（C++11、C++14、C++17）提供的各种特性。映射和序列可使用 C++ 的基于范围的 for 循环迭代。通过使用 `std::optional<T>`（C++ 的“可能类型”实现），处理潜在的类型不匹配变得更加容易。字符串以 `std::string_view` 对象返回，意味着它们可被 C++ 代码使用而无需从序列化数据中复制或在堆上分配副本。

```sh
opaque uint32 fd | A file descriptor number.

struct iovec | A region of memory for scatter/gather reads.
range void buf | The address and length of the buffer to be filled.

syscall fd_read | Reads from a file descriptor.
in fd           fd | The file descriptor from which data should be
                  | read.
crange iovec iovs | List of scatter/gather vectors where data
                  | should be stored.
out size         nread | The number of bytes read.
```

从这个文件中，我们现在自动生成 C 头文件、系统调用表、vDSO 和 HTML/Markdown 文档。我们最终希望使用此框架为其他语言（如 Rust、Go）生成低层绑定，使它们能移植到 CloudABI 而不依赖任何手动复制的定义。

以下是 C++ 绑定在实践中使用的示例。与引言中使用 C API 的 Web 服务器相比，C++ 代码要紧凑得多且更易理解。

```c
#include <argdata.hpp>
#include <cstdlib>
#include <optional>
#include <program.h>

void program_main(const argdata_t *ad) {
    // 扫描所有配置选项并
    // 提取值。
    std::optional<int> database_directory, logfile, http_socket;
    for (auto [key, value] : ad->as_map()) {
        if (auto keystr = key->get_str(); keystr) {
            if (*keystr == "database_directory")
                database_directory = value->get_fd();
            else if (*keystr == "logfile")
                logfile = value->get_fd();
            else if (*keystr == "http_socket")
                http_socket = value->get_fd();
        }
    }

    // 如果未以所有
    // 必需描述符启动则终止。
    if (!database_directory || !logfile || !http_socket)
        std::exit(1);

    …
}
```

## 下一个项目：面向 Kubernetes 的 CloudABI？

过去几年，集群管理系统领域发生了大量发展。这些系统允许你将一大组服务器视为单一计算资源池，在其上调度作业。一个日益流行的是由谷歌设计、通过 Linux 基金会新成立的云原生计算基金会（CNCF）资助的 Kubernetes。Kubernetes 可以运行任何已打包为容器（使用 Docker、rkt 等）的软件。我观察到的 Kubernetes 使用中的一个问题是，它有一些怪癖，源于它必须能够运行非依赖注入的软件。例如，由于容器中运行的程序可自由绑定任意网络端口，集群中每个作业（“Pod”）必须拥有唯一的 IPv4 地址。这使得 Kubernetes 消耗大量网络地址，并使集群中节点的路由表非常复杂。相反，由于大多数传统软件只能连接到单个网络地址以访问后端服务，Kubernetes 为内部流量执行大量 TCP 级负载均衡，这使得追踪和调试非常复杂。集群中的作业能够创建任意网络连接这一事实也意味着，只有在集群中运行的所有容器都配置为使用加密、认证和授权（例如使用具有每服务自定义信任链的 SSL）时，Pod 间的安全才能实现。不幸的是，人们几乎从不费心正确设置，这意味着在多租户环境中运营单个 Kubernetes 集群通常是不安全的。对我们来说一个有趣的发展是，从 1.5 版本起，Kubernetes 不再直接与系统的容器引擎通信。当 Kubernetes 想在集群中的某节点上启动容器时，它会向一个附加进程发送 RPC，该进程称为容器运行时接口（Container Runtime Interface，CRI）。此机制旨在让人们通过开发自己的 CRI 更容易地试验自定义容器格式。在我们的案例中，我们可以实现自己的 CRI，允许我们在 Kubernetes 之上直接运行 CloudABI 进程。通过使用 CloudABI，我们可以强制在集群上运行的作业的所有网络连接都由辅助进程注入。此辅助进程可确保集群中的所有流量都经过加密、授权和负载均衡。在集群上运行的软件将不再需要关心底层网络模型。

## 总结

由于关于 CloudABI 的大部分讨论都在 IRC 上进行，如果你有任何问题或只是想了解最新动态，欢迎加入 EFnet 上的 #cloudabi。

我希望本文展示了过去一年 CloudABI 发生了许多有趣的事情，以及更多令人兴奋的计划正在进行中。由于 CloudABI 是 FreeBSD 11 的一部分，且大多数新功能已合并到 11-STABLE，CloudABI 已成为创建安全和可测试软件的易用工具。如果你维护的软件能从中受益，务必尝试为 CloudABI 构建它。

## 链接

FreeBSD 上的 CloudABI：<https://nuxi.nl/cloudabi/freebsd/>
CloudABI 开发博客：<https://nuxi.nl/blog/>
GitHub 上的 CloudABI：<https://github.com/NuxiNL>
CloudABI 版 Bitcoin：<https://laanwj.github.io/>

---

**ED SCHOUTEN** 自 2008 年起就是 FreeBSD 项目的开发者。他的贡献包括 FreeBSD 8 的 SMP 安全 TTY 层、Clang 在 FreeBSD 9 中的初始导入，以及最终进入 FreeBSD 10 的 **vt(4)** 控制台驱动程序的初始版本。

CloudABI 由作者的公司 Nuxi（位于荷兰）开发。它将始终作为开源软件免费提供。Nuxi 为 CloudABI 提供商业支持、咨询和培训。如果你有兴趣在任何产品中使用 CloudABI，请通过 <info@nuxi.nl> 联系 Nuxi。
