# FreeBSD 代码审查与 git-arc

- 原文链接：[FreeBSD Code Review with git-arc](https://freebsdfoundation.org/wp-content/uploads/2021/11/FreeBSD-Code-Review-with-git-arc.pdf)
- 作者：**JOHN BALDWIN**

FreeBSD 项目在 [https://reviews.freebsd.org](https://reviews.freebsd.org) 使用 Phabricator 的 Differential 应用作为代码审查工具。Phabricator 本身提供了多个应用来支持软件开发，但 FreeBSD 项目仅使用代码审查工具。用户和开发者可以通过直接在 Web 应用中粘贴 diff，或使用命令行工具 arc（可通过软件包或 port 安装 [devel/arcanist](https://www.freshports.org/devel/arcanist) ）来上传变更进行审查。arc 可以从命令行上传 git 分支中的提交，并且还能修改已审查提交的提交日志，以便将其推送到公共代码库。  

然而，arc 在 FreeBSD 开发中存在一些使用上的局限性，使其显得不太顺手：  

- Arcanist 使用其自带的提交日志模板，该模板的格式与 FreeBSD 现有模板不匹配，且不易修改。  

- Arcanist 假设开发分支中的所有提交都会作为单个 Differential 修订版本上传进行审查。在包含多个提交的功能分支上工作时，通常逐个审查每个提交更加高效。  

尽管可以通过一定的方法规避这些限制（例如，使用 `--head` 选项可以将单个提交作为独立的审查上传），但这些方法比较繁琐。  

git-arc 工具是 arc 的一个封装，主要用于解决这些限制。git-arc 是 git 的一个插件，它为 git 添加了与 Phabricator 交互的相关命令。  

## 安装 git-arc  

git-arc 存储在 FreeBSD 源代码库的 `tools/tools/git` 目录中。可以在该目录下运行 `make install` 来安装该脚本及其 man 手册页。请注意，git-arc 默认安装到 `/usr/local/bin`。  

```sh
> git clone https://git.freebsd.org/src.git
...
> cd src/tools/tools/git
> make
gzip -cn /usr/home/john/work/freebsd/main/tools/tools/git/git-arc.1 >
git-arc.1.gz
> make install
install -o root -g wheel -m 555
/usr/home/john/work/freebsd/main/tools/tools/git/git-arc.sh
/usr/local/bin/git-arc
install -o root -g wheel -m 444 git-arc.1.gz /usr/share/man/man1/
```

此外，git-arc 需要安装以下软件包：arcanist（[devel/arcanist](https://www.freshports.org/devel/arcanist)）、git（[devel/git](https://www.freshports.org/devel/git)）和 jq（textproc/jq）。  

由于 git-arc 是 arcanist 的封装，因此必须先初始化 arcanist。首先，在 [https://reviews.freebsd.org](https://reviews.freebsd.org) 创建一个账户。其次，为 arcanist 安装 API 令牌：  

```sh
> arc install-certificate https://reviews.freebsd.org
 CONNECT Connecting to “https://reviews.freebsd.org/api/”...
LOGIN TO PHABRICATOR
Open this page in your browser and login to Phabricator if necessary:
https://reviews.freebsd.org/conduit/login/
Then paste the API Token on that page below.
 Paste API Token from that page: cli-XXXXX
Writing ~/.arcrc...
 SUCCESS! API Token installed.
```

## 使用 git-arc 进行单个提交的审查  

审查提交的第一步是准备提交内容。提交应当是一个符合要求的候选提交，并带有合适的日志消息。任何修正（fixup）都应当被合并（squash）到提交中，以确保提交内容与最终推送到代码库的版本一致。  

### 创建审查  

提交准备完成后，下一步是使用 `create` 子命令为该提交创建审查。此命令接受一个提交的引用（如哈希值或符号引用，例如 `HEAD`）作为位置参数，位于所有选项之后。`create` 子命令会使用提交日志消息作为审查的标题和描述，在 Phabricator 上创建一个新的审查。  

可以使用 `-r` 选项添加审查者。多个审查者可以用逗号分隔，或者通过多个 `-r` 选项指定。要添加一个审查组，可以在组名前加 `#` 符号，例如 `#bhyve` 以标记负责 bhyve(8) hypervisor 的开发者。  

以下示例为 `gdb_11` 分支上的一个提交创建审查，该提交涉及 `devel/gdb` port，并将 port 维护者（pizzamig@FreeBSD.org）指定为审查者：


```sh
> git log --oneline main..gdb_11
6b21ea620990 devel/gdb: Update to 11.1.
> git arc create -r pizzamig gdb_11
commit 6b21ea62099007a0d376852731cdbde4d8d522d9 (HEAD -> gdb_11)
Author: John Baldwin <jhb@FreeBSD.org>
Date: Thu Sep 16 17:25:13 2021 -0700
 devel/gdb: Update to 11.1.
…
Does this look OK? [y/N] y
...
```

`create` 子命令首先会显示提交内容，包括日志消息和补丁。随后，它会提示用户确认是否要创建审查。输入 `y` 继续并创建审查。  

## 更新审查  

`update` 子命令用于在修改补丁后更新审查。首先，对相关的 git 提交进行更改，可以通过修改（amend）提交，或提交新的更改后再将其合并（squash）回原始提交。请注意，`update` 命令仅更新审查中的补丁，不会更新审查的标题或描述，也不会允许添加额外的审查者或订阅者。这些更改需要在 Phabricator 的 Web 应用中进行。  

>**注意**
>
>`update` 和 `stage` 子命令会使用提交日志的第一行来查找标题相同的审查。如果修改了提交日志消息的第一行，则必须在 Phabricator 中更新审查的标题，否则 `git-arc` 无法正确识别该提交对应的现有审查。  

以下示例展示了在根据审查者反馈修改原始提交后，如何更新 `devel/gdb` port 更新的审查：  

```sh
> git arc update gdb_11
commit 8d532ac873633ae12d42be709755f56f9d86c310 (HEAD -> gdb_11)
Author: John Baldwin <jhb@FreeBSD.org>
Date: Thu Sep 16 17:25:13 2021 -0700
 devel/gdb: Update to 11.1.

...
Does this look OK? [y/N] y
...
```

与 `create` 子命令类似，`update` 会先显示日志消息和补丁，然后提示用户确认更新。输入 `y` 继续并更新审查。接下来，系统会打开一个编辑器窗口（使用用户在 `$EDITOR` 环境变量中配置的编辑器）。在其中输入本次更新所做更改的描述，保存文件并退出编辑器。该描述将作为评论添加到审查中。  

## 完成审查并推送提交  

当提交准备好发布时，`stage` 子命令会将提交合并到本地 `main` 分支，并在提交日志中添加来自审查的元数据。具体来说，`git arc stage` 会向提交日志正确添加 `Reviewed by` 和 `Differential Revision` 标记。操作完成后，工作树会被设置为 `main` 分支的新顶点，并包含已暂存的提交。此时，可以使用 `git show` 进行最终审查，然后再推送到公共代码库。  

以下示例展示了 `devel/gdb` 更新的最终推送过程：

```sh
> git checkout main
> git fetch freebsd
> git merge freebsd/main
> git arc stage gdb_11
> git push freebsd
```

前三个命令确保本地 `main` 分支是最新的，然后再进行暂存提交。`stage` 子命令在合并提交后会弹出一个编辑器窗口，其中包含更新后的提交日志。这为修正提交日志中的格式问题提供了机会。保存提交日志并退出编辑器以继续操作。  

请注意，如果 `stage` 子命令在将提交合并到 `main` 时遇到冲突，则操作会失败。要解决此问题，需要先将原始提交变基（rebase）到更新后的 `main` 分支。如果冲突解决过程中产生的更改需要审查，则应更新审查记录。否则，可以使用变基后的提交重新运行 `stage` 子命令。  

## 在分支上使用 git-arc  

虽然 `git-arc` 适用于单个提交，但当处理包含多个提交的分支时，它的优势更为明显。`create`、`update` 和 `stage` 子命令都支持一次性处理多个提交。这些提交可以通过单独的引用（如前面单个提交的示例）或 Git 修订范围来指定。  

### 创建审查  

对于分支，`git-arc` 会为每个提交创建一个审查，并在 Phabricator 中将这些提交链接成一个堆栈。分支中的第一个提交的审查被标记为第二个提交审查的父修订，依此类推。这使得在 Phabricator UI 中查看分支中任意提交的审查时，都可以通过 **Stack** 选项卡找到该分支的所有提交。  

默认情况下，`create` 子命令会逐个显示每个提交的日志消息和补丁，并在每个提交后提示用户确认。对于包含许多提交的分支，这个步骤可能会很繁琐，因此 `create` 提供了可选的 `-l` 参数。如果使用该参数，`create` 将列出所有待提交的审查，并只进行一次确认提示。如果用户在提示时输入 `y`，则整个审查堆栈会一次性创建，而不会进一步提示。以下示例展示了如何为当前目录中检出的分支创建所有提交的审查。该分支是基于 `main` 分支创建的：

```sh
> git arc create -l -s emaste main..
2f7e09973ab6 cryptodev: Use ‘csp’ in the handlers for requests.
fbc805bb4d62 ccp, ccr: Simplify drivers to assume an AES-GCM IV length of 12.
8390644fd45f crypto: Permit variable-sized IVs for ciphers with a reinit hook.
f08d44eaa9ee cryptosoft, ccr: Use crp_iv directly for AES-CCM and AES-GCM.
43aaceb5afef cryptodev: Permit explicit IV/nonce and MAC/tag lengths.
0190b9e740d8 cryptodev: Permit CIOCCRYPT for AEAD ciphers.
03f07b455c80 cryptodev: Allow some CIOCCRYPT operations with an empty payload.
e0caaccf1ec0 cryptocheck: Support multiple IV sizes for AES-CCM.
ae4a0338bc8b crypto: Support multiple nonce lengths for AES-CCM.
cb18504c7712 aesni: Support multiple nonce lengths for AES-CCM.
60e1a45f2201 aesni: Handle requests with an empty payload.
aa653be04078 aesni: Permit AES-CCM requests with neither payload nor AAD.
fafcdb583930 aesni: Support AES-CCM requests with a truncated tag.
75c003b9ccbb ccr: Support multiple nonce lengths for AES-CCM.
de32cc23b0f9 ccr: Support AES-CCM requests with truncated tags.
6a88ac41d972 safexcel: Support multiple nonce lengths for AES-CCM.
ff4f260a5fd9 safexcel: Support truncated tags for AES-CCM.
e46422be0eaf cryptosoft: Fix support for variable tag lengths in AES-CCM.
d98f930cd833 crypto: Test all of the AES-CCM KAT vectors.
7ee5d373884e crypto: Support Chacha20-Poly1305 with a nonce size of 8 bytes.
Does this look OK? [y/N] y
...
```

如果希望为分支上的每个提交单独创建审查，而不将它们链接在一起，可以分别对每个提交单独运行 `git arc create` 命令。此外，也可以在 Phabricator 的 Web 界面中调整审查之间的关系。  

## 检查审查状态  

在处理分支时，检查分支上各个提交的审查状态可能会很有帮助。可以使用 `list` 子命令来完成此操作。`list` 子命令接受一个或多个提交名称或提交范围，并为每个提交输出一行状态信息。对于没有关联审查的提交，其状态会显示为 **No Review**（无审查）。  

以下示例展示了 `gcc9_universe` 分支上多个提交的状态。在此分支中，作者选择单独审查每个提交，而不是作为一个系列进行审查。因此，一些提交尚未准备好进行审查，而另一些提交已经获得批准，可以合并到 `main` 分支。

```sh
> git arc list main..gcc9_universe
f0f665a2f4a5 Accepted D26202: Switch to GCC 9 for the GCC tinderbox.
a98a78e2dabc Accepted D26203: Pass -msecure-plt to GCC for 32-bit powerpc.
b4412a18ab23 Needs Review D31933: hyperv storvsc: Don’t abuse struct sglist to hold virtual addresses.
a0eaf413441a Accepted D31934: kernel: Disable errors for -Walloca-larger-than for GCC.
daf618e9a8c4 No Review : Fix various places which cast a pointer to a vm_paddr_t or vice versa.
8c46bb47a57f Needs Review D31938: bhyve: Add an empty case for event types in mevent_kq_fflags().
46d2ac2e7b3b Accepted D31941: Use a char * to avoid alignment warnings.
b2d94deacfa1 Needs Review D31945: libmd: Only define SHA256_Transform_c when using the ARM64 ifunc.
c9be458cee94 Accepted D31948: mana: Cast an unused value to void to quiet a warning.
8a9b7debfc0c No Review : arm64: Add compat macros for system registers for GNU as.
```

## 更新审查

使用 `update` 子命令在分支上更新审查并不像其他命令那样直观。特别是，`update` 子命令无法确定与提交关联的审查是否已经是最新的（因此无需更新）。相反，如果为一个分支提供了审查列表，它将始终更新所有审查。`update` 子命令还会提示为每个提交输入描述。在处理分支时，通常更好的做法是，在单独修改每个提交后使用 `update` 子命令，而不是直接对整个分支运行该命令。  

## 完成审查并推送分支

待所有提交准备好推送，可以使用 `stage` 子命令在一次操作中将分支中的所有提交进行暂存。用户的编辑器将被调用以最终确定每个提交的提交日志。操作完成后，可以按照之前在单个提交示例中的说明通过 `git push` 推送分支。  

分支中的单个提交也可以通过使用 `stage` 子命令并指定特定的提交来推送。推送这些提交后，再将分支变基（rebase）以从分支中删除已推送的提交。随后，可以使用 `list` 子命令检查分支上剩余提交的状态。  

## 限制和注意事项

虽然 `git-arc` 在 `arc` 的基础上提供了一个简化的接口，但它也有一些自身的限制：  

- 匹配提交与现有审查需要提交日志的第一行与审查的标题一致。这意味着，如果更新了提交日志的第一行，则必须在 Phabricator 中手动更新审查标题，之后 `update`、`list` 或 `stage` 子命令才会识别该审查。  

- 类似地，`update` 子命令只能更新给定审查的补丁。如果提交日志消息已更改，它不会更新审查描述。  

- 如果在分支审查期间丢失了某些提交，`git-arc` 并不提供检测丢失提交或放弃关联审查的方法。放弃审查必须在 Phabricator 中进行。  

- 如果在分支审查期间添加了额外的提交，可以使用 `list` 子命令查找这些提交，使用 `create` 子命令创建审查。但是，`git-arc` 不提供自动调整 Phabricator 中父子关系的方法，也无法更新审查堆栈以匹配分支的新布局。这必须在 Phabricator 中手动完成。  

- `git-arc` 无法确定一个提交的审查是否已经过时（即该提交自审查创建或上次更新后是否已更新）。这样的功能将有助于在 `list` 子命令中标注过时的修订。它还可以使 `update` 子命令在使用修订范围时更有用，通过省略对未更改提交的更新。

---

**JOHN BALDWIN** 是一位系统软件开发人员。在过去的 20 年里，他直接向 FreeBSD 操作系统提交更改，涉及内核的各个部分（包括 x86 平台支持、SMP、各种设备驱动程序以及虚拟内存子系统）和用户空间程序。除了编写代码，John 还曾在 FreeBSD 核心团队和发布工程团队工作。他还为 GDB 调试器和 LLVM 作出了贡献。John 与妻子 Kimberly 和三个孩子 Janelle、Evan 和 Bella 一起生活在加利福尼亚州的康科德市。
