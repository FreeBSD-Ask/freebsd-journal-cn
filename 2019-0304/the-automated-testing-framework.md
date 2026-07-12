# 自动化测试框架

尽管测试很重要，但测试工作可能枯燥又重复。我们大多数人投身开发，是因为我们更喜欢构建东西而非破坏东西。幸运的是，我们有机器擅长做枯燥、重复的工作，且从不要求加薪或休假。

自动化测试有许多优点。它意味着可重复的测试。它意味着我们确切知道自己在测试什么、不在测试什么，出了问题时通常能轻松复现。很少有事情能让开发者像收到带可复现测试用例的 bug 报告那样开心。

它也意味着更容易定期运行测试，从而更快发现东西坏了。事实证明，用户不喜欢软件突然爆炸。他们希望软件无聊又可预测。开发者可以把那些令人兴奋的意外留给自己。

## FreeBSD 的测试如何运作？

和所有伟大的艺术家一样，FreeBSD 开发者从别处“借鉴”了代码。这次的灵感来源是 NetBSD。ATF，即自动化测试框架，最初是一个谷歌编程之夏项目。它于 2007 年首次导入 NetBSD，2012 年导入 FreeBSD。

ATF 是用于编写测试的框架。另一个 NetBSD 项目 Kyua 负责运行测试并汇总结果。运行测试时，你主要与 Kyua 打交道。编写测试时，通常与 ATF 打交道。不用 ATF 也能编写测试让 Kyua 执行，但本文不讨论这种情况。

作者：Kristof Provost

## 为什么要写测试？

好问题，很高兴你问了！

写测试的理由有好几个，如果这篇文章你只能记住一件事，那应该是：你应该写测试，因为这有益于灵魂。好吧，这可能是句谎话，但肯定对项目有益。

测试能确保你依赖的功能不会损坏。FreeBSD 非常频繁地运行测试套件。amd64 上测试运行的结果可以在 <https://ci.freebsd.org/job/FreeBSD-head-amd64-test/> 查看。

编写测试用例也是帮助开发者修复你心头那只 bug 的好办法。能演示问题的简短测试用例往往是让 bug 得以修复的关键。它不仅确凿证明存在问题，还清楚地说明了如何复现，让修复容易得多。

处理 pf 的问题时，我常常发现自己花在理解配置和复现问题上的时间，远多于调试和修复的时间。bug 之所以得不到修复，往往是因为无法复现。

如果你有想合入的补丁，加上测试也会有帮助。它能演示你正在修复的问题或新增的功能，既说服人们这是个好主意，也有助于审查补丁。毕竟，替你合入补丁的开发者也要修补它可能引发的任何问题。证明你的代码经过测试，会让你的补丁更可能被接纳。

## 如何使用测试

测试（如果已安装）可以在系统的 **/usr/tests** 下找到。如果你同时有源代码树，测试的源代码可以在 **/usr/src/tests** 或它们所测试的应用程序、库或其他组件的子目录中找到。例如，**/bin/sh** 的测试在 **/usr/src/bin/sh/tests**；pfctl 的测试在 **/usr/src/sbin/pfctl/tests**。pf 本身的测试在 **/usr/src/tests/sys/netpfil/pf**。

要运行测试，你需要安装 Kyua（`pkg install kyua`）。然后可以在 **/usr/tests** 中执行 `kyua test`。许多测试需要以 root 身份运行：

```sh
/usr/tests % sudo kyua test
sbin/growfs/legacy_test:main  ->  passed  [4.710s]
sbin/devd/client_test:seqpacket  ->  passed  [0.015s]
sbin/devd/client_test:stream  ->  passed  [0.015s]
sbin/dhclient/option-domain-search_test:main  ->  passed  [0.005s]
sbin/pfctl/macro:space  ->  passed  [0.051s]
...
```

你也可以在测试子目录中运行命令，只跑一部分测试：

```sh
usr/tests/bin/cat % kyua test
cat_test:align  ->  passed  [0.023s]
cat_test:b_output  ->  passed  [0.025s]
cat_test:e_output  ->  passed  [0.023s]
cat_test:nonexistent  ->  passed  [0.022s]
cat_test:s_output  ->  passed  [0.026s]
cat_test:se_output  ->  passed  [0.023s]
cat_test:vt_output  ->  passed  [0.029s]
```

或者，用 `kyua list` 列出测试清单，然后挑选单个测试运行：

```sh
/usr/tests % kyua test lib/csu/static/fini_test:dso_handle_test
lib/csu/static/fini_test:dso_handle_test -> passed [0.002s]
```

要获取测试的更多信息，使用 `kyua list -v`。附加信息可能包括所需程序、所需用户或描述等内容。

不言而喻，这只应在闲置的系统上做。测试的本质决定了它们可能暴露 bug，而有些 bug 严重到足以让整个系统崩溃。

此外，由于测试会尝试演练系统的各种功能，它们很可能做出影响系统正常使用的配置变更。

未经系统管理员许可，不要运行它们。如果管理员恰好是你自己，“提交一式三份书面申请”这条要求大概可以免除。

那么，何时运行这些测试？通常在开发期间，但它们也可以作为新装系统的冒烟测试。通过测试并不保证硬件工作完美，但能增强信心。

## 如何编写测试

编写自己的测试时，实际上要讨论两个不同的话题：如何编写测试的机制，以及测试什么、如何测试的判断。

我们先简要介绍机制，这部分大多在 **atf(7)** man 页中有文档。测试可以用 shell 脚本或 C（或 C++）代码编写——哪种最好取决于测试的内容。通常 shell 版本（`man 3 atf-sh`）最适合测试整个应用程序，而 C/C++ 版本（`man 3 atf-c`、`man 3 atf-c++`）最适合测试库或（内核）API。

atf-sh 测试由 atf-sh 执行，因此首行需要是 `#!/usr/bin/env atf-sh`。不过这由安装代码处理，我们这里无需操心。它们总是包含 `atf_init_test_cases` 函数，列出测试用例。每个测试用例总有 head 和 body，可能还有 cleanup 函数：

```sh
atf_test_case tc1
tc1_head() {
    ... 第一个测试用例的头部 ...
}
tc1_body() {
    ... 第一个测试用例的主体 ...
}
atf_test_case tc2 cleanup
tc2_head() {
    ... 第二个测试用例的头部 ...
}
tc2_body() {
    ... 第二个测试用例的主体 ...
}
tc2_cleanup() {
    ... 第二个测试用例的清理 ...
}
... 其他测试用例 ...
atf_init_test_cases() {
    atf_add_test_case tc1
    atf_add_test_case tc2
    ... 添加其他测试用例 ...
}
```

注意所有测试用例函数都有相同的前缀。这个前缀在 `atf_init_test_cases()` 函数中传给测试系统，让框架能找到它们。cleanup 函数可以有任何名字，但通常命名为 cleanup 是个好主意。

head 函数用于配置测试用例。我们可以用 `atf_set "descr" "测试描述写在这里"` 设置测试描述。如果这个测试只能以特定用户（即 root）或特定架构运行，我们会加上 `atf_set require.user root` 或 `atf_set require.arch amd64`。可在此设置的选项完整清单见 `man 4 atf-test-case`。

如果需要更具体的测试，可以用 `atf_skip` 实现。例如，需要 VIMAGE 内核选项的测试可以这样：

```sh
if [ "$(sysctl -i -n kern.features.vimage)" != 1 ]; then
    atf_skip "本测试需要 VIMAGE"
fi
```

这里有个坑：这种构造不能放在 head 里，必须放在测试主体中。既然所有设置和准备工作就绪，我们终于可以看看测试主体了。这里我们运行要考验的应用程序或代码。

大部分重活由 **atf-check(1)** 完成。它执行一条命令，并能帮我们检查退出状态、标准输出和/或标准错误是否符合预期。例如，我们可能想测试 `false` 命令是否真的返回预期的退出状态（`-s exit:1`），并且 stdout（`-o empty`）或 stderr（`-e empty`）都不产生输出：

```sh
atf-check -s exit:1 -o empty -e empty /usr/bin/false
```

我们也可以用 `atf_expect_signal` 期待测试程序返回信号，或用 `atf_expect_timeout` 期待超时。

最后，如果我们为一个已知损坏的东西写了测试，也可以把这个信息加到测试中。用 `atf_expect_fail` 可以表示我们预期这个测试失败。传统上我们还会在这条消息中包含 bug（PR）编号。测试仍会执行，但失败不会被计为失败的测试。

## C 测试

C（或 C++）测试结构类似，只是语法略有不同。

下面是简短的示例：

```c
#include <atf-c.h>
ATF_TC(tc1);
ATF_TC_HEAD(tc1, tc)
{
    atf_tc_set_md_var(tc, "require.user", "root");
}
ATF_TC_BODY(tc1, tc)
{
    ATF_CHECK_EQ(3, 2 + 1);
}
ATF_TP_ADD_TCS(tp)
{
    ATF_TP_ADD_TC(tcs, tc1);
    return atf_no_error();
}
```

在了解了前面的内容后，这个结构应该很熟悉。我们有入口点（`ATF_TP_ADD_TCS()`）列出所有测试用例，而测试用例有一个 head（`ATF_TC_HEAD()`）告诉测试框架运行此测试需要 root。

我们可能在最后这一点上撒了谎，因为测试主体所做的只是确保 2 + 1 等于 3。我们通常预期这个判断为真，所以这个测试应该通过。

## 把一切串联起来

现在我们知道了如何编写测试，还需要弄清楚如何真正运行它们。幸运的是，makefile 替我们做了所有繁重的工作，我们只需写类似这样的内容：

```makefile
# $FreeBSD$
ATF_TESTS_SH += sh_example
ATF_TESTS_C += c_example
.include <bsd.test.mk>
```

这会期望找到一个 `sh_example.sh` 文件和一个 `c_example.c` 文件，构建它们，并作为 `buildworld` 和 `installworld` make 目标的一部分安装它们。makefile 还会自动创建所需的目录结构和 Kyuafile。

## 如何调试你的测试

生活中让人难过的事实是，代码从一开始就从不完美（否则我们还需要测试吗？），测试代码也不例外。你的测试很可能不会完全按你预期的那样工作，而 kyua 的输出通常不太有帮助：

```sh
# kyua test example
example:example  ->  failed: atf-check failed
; see the output of the test for details  [0.017s]
```

幸运的是，kyua 记录的远不止这些，只要我们好好请求，就能获得关于失败测试、它的环境以及它在测试期间生成的输出的更多信息：

```sh
# kyua test example
example:example  ->  failed: atf-check failed; see the output
of the test for details  [0.017s]
# kyua report --verbose example:example
===> Execution context
Current directory: /usr/tests/sys/netpfil/pf
Environment variables:
HOME=/root
LANG=C
LC_CTYPE=en_US.UTF-8
LC_PAPER=en_GB.UTF-8
LOGNAME=root
MAIL=/var/mail/root
PATH=/home/kp/bin:/usr/local/sbin:/sbin:/usr/local/bin:/
bin:/usr/bin:/usr/sbin:/home/kp/bin:/bin:/sbin:/usr/bin:/
usr/sbin:/usr/games:/usr/local/bin:/usr/local/sbin:/usr/
pkg/bin:/usr/pkg/sbin
SHELL=/bin/csh
SUDO_COMMAND=/usr/local/bin/kyua test example
SUDO_GID=1001
SUDO_UID=1001
SUDO_USER=kp
TERM=xterm-256color
USER=root
===> example:example
Result:     failed: atf-check failed;
see the output of the test for details
Start time: 2018-12-28T17:57:49.314738Z
End time:   2018-12-28T17:57:49.332141Z
Duration:   0.017s
Metadata:
allowed_architectures is empty
allowed_platforms is empty
description is empty
has_cleanup = false
is_exclusive = false
required_configs is empty
required_disk_space = 0
required_files is empty
required_memory = 0
required_programs is empty
required_user is empty
timeout = 300
Standard output:
Executing command [ false ]
Standard error:
Fail: incorrect exit status: 1, expected: 0
stdout:
stderr:
===> Failed tests
example:example -> failed: atf-check failed; see the output of the test for details [0.017s]
===> Summary
Results read from /root/.kyua/store/results.usr_tests_sys_netpfil_pf.20181228-175749-261396.db
Test cases: 1 total, 0 skipped, 0 expected failures, 0 broken, 1 failed
Start time: 2018-12-28T17:57:49.314738Z
End time:   2018-12-28T17:57:49.332141Z
Total time: 0.017s
```

这里最有趣的是标准输出和标准错误日志。它们告诉我们，我们执行了命令 `false`，期待错误状态 `0`，但实际得到了 `1`。在这个例子里，是因为我们的测试写错了。我们当然预期 `false` 以状态 1 退出，而不是 0。

对于更复杂的 atf-sh 测试，在测试主体顶部加上 `set -x` 非常有用。这会让 shell 记录它执行的一切，便于看清运行了哪些命令、测试停在哪里。

## 什么是好测试？

知道如何编写测试只是成功了一半。知道测试什么同样重要。什么是好测试没有硬性规则，但有一些技巧能帮助找到好的测试场景。

最简单的办法是从已知 bug 入手。如果你知道某个东西坏了，为它写个测试是有价值的，哪怕只是为了确保这个特定的 bug 不再发生。bug 往往聚集在特定功能或代码段附近，所以这个场景的变体可能捕获更多 bug。

另一类有价值的测试用例是检查校验代码。许多代码会检查用户输入，确保落在预期范围内。挑一组值，确保它们被适当地拒绝或接受。通常，很低或很高的值、刚好在有效范围内和刚好在范围外的值，都是好的测试点。

例如，假设某个配置旋钮应接受 0 到 10（含）之间的值。好的测试应确保 -1000、-1、11 和 1000 被拒绝，而 0、5 和 10 被接受。

思考测试时，人自然会想到“坏”场景：坏的输入、坏的配置、一切都很坏。测试一切按预期工作的场景也很有用。

最后，重要的是记住有一些测试远胜于完全没有。不要让“该测试什么”的不确定阻止你。写一个测试，任何测试，你就为系统的整体稳定性和质量做出了贡献。即使非常基础的测试也能暴露问题。

## vnet

最后，谈谈我最爱的测试：网络栈和防火墙测试。

12.0 是第一个默认启用 vnet 的 release。我不会详细展开 vnet，那是别人写的另一篇文章的话题。

vnet 将网络栈虚拟化。如果这句术语对你像天书，试试这样理解：vnet 给每个 jail 自己的网络栈。网络接口可以专属某个 jail。jail 可以自行设置 IP 地址，无需主机帮忙。甚至可以在 jail 内部使用 ipfw 或 pf。jail 看起来就很像一台虚拟机了。

长话短说，这意味着你可以在一台笔记本上，在几秒内创建通常需要多台机器的网络配置。这意味着为整个网络栈编写测试是可能的，而且实际上相当容易。站在我这个完全中立、毫无偏见的 pf 维护者视角来看，这意味着你应该为 pf 写测试。

示例可在 **/usr/src/tests/sys/netpfil/pf** 找到。

其他人也利用这个示例为 IPSec 编写了测试，见 **/usr/src/tests/sys/netipsec**。 •

---

**KRISTOF PROVOST** 是一名自由嵌入式软件工程师，专攻网络和视频应用。他是 FreeBSD 提交者、FreeBSD 中 pf 防火墙的维护者、EuroBSDCon 基金会董事会成员。Kristof 不幸总爱踩中 uClibc 的 bug，并对 FTP 怀有刻骨的厌恶。不要跟他提 IPv6 分片。
