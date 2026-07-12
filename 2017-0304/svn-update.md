# SVN 动态

作者：Steven Kreuzer

拥有可复现构建系统的目标相当简单：编译某段源代码时，所产出的二进制文件应当与过去或未来编译出的任何二进制文件完全一致。可复现构建的最大好处之一，是你可以确信在生产环境运行的任何代码都如你所期望的那样。但它还带来诸多其他好处，比如大幅提升缓存共享，从而缩短构建时间，甚至能解决开发者必须维护加密密钥以签名代码、或信任构建二进制文件所用机器这类复杂问题。开发者早在 2013 年就开始为支持可复现构建添加工作，过去几个月里，我们看到若干提交让这项工作更接近完成。本期专栏，我决定着重介绍几项大小不一的改动，借以展示如此复杂的开发项目所投入的功夫。这类项目对最终用户而言往往不可见，但正是这类任务帮助构建出如此健壮且可验证的系统。

- 添加 `WITH_REPRODUCIBLE_BUILD` **src.conf(5)** 旋钮以禁用内核元数据。<https://svnweb.freebsd.org/changeset/base/310128>
- 在 `WITH_REPRODUCIBLE_BUILD` 时以可复现方式构建 loader。<https://svnweb.freebsd.org/changeset/base/310268>
- 移除选项 `-vd` 以使 **iasl(8)** 可复现。<https://svnweb.freebsd.org/changeset/base/311529>
- 在 vchi 驱动中以硬编码字符串替换不可复现的 `__DATE__`/`__TIME__`。<https://svnweb.freebsd.org/changeset/base/310560>
- 避免在 mlx 驱动中使用 `__DATE__` 以使构建可复现。<https://svnweb.freebsd.org/changeset/base/310425>
- 移除 `srand()` 以确保 **bhnd(4)** 输出确定性。<https://svnweb.freebsd.org/changeset/base/310371>
- 在 `newvers.sh` 中添加选项 `-R`，仅对未修改的 src 树包含元数据。<https://svnweb.freebsd.org/changeset/base/310273>
- 在 `newvers.sh` 中添加选项以消除内核构建元数据。<https://svnweb.freebsd.org/changeset/base/310112>
- 使 `makewhatis` 的输出可复现。<https://svnweb.freebsd.org/changeset/base/307003>
- 在 man 手册页中使用 changelog 日期而非文件修改日期。<https://svnweb.freebsd.org/changeset/base/306740>
- 将 UEFI boot loader 的 PE/COFF 时间戳设为已知值以支持可复现构建。<https://svnweb.freebsd.org/changeset/base/305160>

然而，可复现构建并非本月 src 树中唯一令人兴奋的变更。LLVM 3.9.1 也已导入，整个 FreeBSD 用户空间加内核已成功使用 LLVM 的 LLD 链接（尚有一个 FreeBSD 补丁待提交，修复的是一个存在了 18 年的 bug）。

- 将 clang、llvm、lld、lldb、compiler-rt 和 libc++ 升级至 3.9.1 release。<https://svnweb.freebsd.org/changeset/base/310194>
- btxldr：处理所有 PT_LOAD 段，而非仅前两个。<https://svnweb.freebsd.org/changeset/base/310702>

最后，Amazon 宣布已在 15 个区域的 EC2 上推出 IPv6 支持，未来的 FreeBSD 发行版将默认在 EC2 上支持 IPv6。现有 EC2 实例也可支持 IPv6，但需要你运行几条简单的命令。

---

**STEVEN KREUZER** 是 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车感兴趣。他与妻子、女儿和狗住在纽约皇后区。
