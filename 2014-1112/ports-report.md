# Ports 报告

作者：Frederic Culot

9 月和 10 月我们的 Ports 树相当活跃。但在介绍之前，我很高兴告诉大家，过去两个月里 Ports 树上有 4694 次提交，关闭了 1039 份问题报告。这再次表明我们的志愿者致力于让 Ports 树保持最新，并重视用户反馈。感谢所有人。下面进入正题！

## 重要的 Ports 更新

我们进行了若干次 exp-run（实际近 30 次！），以检查主要 port 更新是否安全。在这些重要更新中，我们提到以下亮点：

- 默认 ruby 版本现为 2.0
- cmake 更新至 3.0.2
- gmake 更新至 4.1
- qt 更新至 5.3.2
- kde 更新至 4.14.2
- Linux 基础从 Fedora 10 切换为 CentOS 6

照例，如果某些 port 更新需要手动步骤，这些步骤会在 **/usr/ports/UPDATING** 文件中明确说明。在 Ports 树上执行任何更新之前，强烈建议先查看该文件。

## 新的 Ports 提交者与代管

三位新成员加入 Ports 提交者行列。

Dominic Fandrey 的导师是 cs@ 和 koobs@。其次，sbruno@ 已有 src 提交权限，导师是 bdrewery@ 和 bapt@。最后，Gordon Tetlow 也获得 Ports 提交权限，导师是 erwin@ 和 mat@。

若干提交权限因长期未使用（通常超过一年）而被代管。这些开发者的 Ports svn 仓库访问权限被撤销。但他们在 FreeBSD 基础设施上的账户保留，想回来时可轻松恢复。注意开发者恢复时可能需要配备导师。视闲置期长短，Ports 树基础设施可能已发生显著变化。以下是提交权限被代管的开发者：sylvio@、pclin@、flz@、jsa@、anders@。希望他们很快回来，感谢他们至今的工作。

## Ports 树值得注意的变化

得益于 `@dirrm` 和 `@dirrmtry` 关键字被弃用，Ports 树对 pkg-plist 文件做了大规模清理。这次清理之所以可能，是因为 **pkg(8)** 现在能在需要时自动删除 **${PREFIX}** 下的目录。结果，pkg-plist 文件中包含 `@dirrm`/`@dirrmtry` 指令的多数行被删除。但请注意，可能仍需指定要清理的目录（主要是 **${PREFIX}** 之外创建的目录，如在 **/var/games/** 中创建的游戏文件），此时必须使用新的 `@dir` 关键字。

请注意 `@cwd` 关键字也已弃用。Ports 基础设施的所有这些变化都有文档记录，开发者和用户可参考两个重要来源来跟踪变化。其一是 porter’s handbook（<https://www.freebsd.org/doc/en/books/porters-handbook/>），详细解释了新关键字一出现时的用法。其二是 **/usr/ports/CHANGES** 文件，常被忽视，但其中包含主要与 Ports 提交者相关的重要技术细节，对最终用户也有启发。欢迎大家不时查阅此文件。

---

**Frederic Culot** 在 IT 行业工作了 10 年。业余时间学习商业与管理，刚完成 MBA。Frederic 2010 年以 Ports 提交者身份加入 FreeBSD，至今已提交约 2000 次，指导了六位新提交者，现担任 portmgr-secretary。
