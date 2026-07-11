# 使用 Git 贡献到 FreeBSD Ports

- 原文：[Contributing to FreeBSD Ports with Git](https://freebsdfoundation.org/wp-content/uploads/2022/08/mingrone_revised.pdf)
- 作者：**JOSEPH MINGRONE**

FreeBSD Ports 树于 1994 年创建，并使用 CVS 跟踪，直到 2012 年 7 月 15 日，Subversion 接管。2021 年 4 月 6 日发生第二次代码库转换，权威源从 Subversion 迁移到 Git。CVS 和 Subversion 都是集中式版本控制系统，第一次转换所需的工作流变更没有第二次迁移到 Git 时复杂，因为 Git 是分布式版本控制系统。

这不是全面的 Git 使用指南。本文的目的是引导那些不熟悉 Git 或 FreeBSD Ports 的人，帮助他们通过 Git 工作流为 FreeBSD Ports 贡献力量。所涉的主题包括：

- 重要 Git 概念的简要概述
- 保持与远程仓库同步
- 分支管理
- 提交
- 修改历史
- 使用 Phabricator 代码审查
- 使用 poudriere 测试更改
- 跟踪上游发布

详细的介绍可以参考《Pro Git》书籍和 FreeBSD 提交者指南中的 Git Primer 章节。本文未涉及如何使用 Ports 和 Ports 基础设施所编写的 make 规范，相关内容在《FreeBSD Porter’s Handbook》中有详细介绍。

Git 与像 Subversion 这样的集中式版本控制系统的根本区别在于它对分布式工作流的支持。Git 不需要包含权威版本文件副本的中央服务器，因为 1. 仓库的副本是完整的克隆，包含元数据和完整历史，2. Git 提交是仓库的快照，而不是基于文件更改的增量。快照通过哈希算法来描述，该算法将仓库的状态作为输入并生成十六进制字符串的确定性哈希值。如果两个仓库的状态相同，则描述这些副本的哈希值将相同；而两个仓库即便仅有一个比特的差异，其哈希值也会不同。Git 仓库的历史是这些快照的集合，它们通过父子关系连接在一起，每个提交指向其父提交。

Git 工作流可能包括：1. 创建本地分支以开发新功能，2. 将功能分支中的工作合并到主分支，3. 将更改推送到另一个 Git 仓库。对 FreeBSD Ports 贡献者来说，新的功能可能意味着创建或更新 port，甚至像修正拼写错误这样简单的任务。当功能分支中的工作准备就绪时，可以审查并合并到官方的 FreeBSD Ports 仓库中。Git 分支非常适合保持新功能开发的有序和独立，分支创建非常轻量，仅仅是创建指向快照的新指针。

大部分 Git 工作发生在本地，因此并没有单一的工作流要求所有贡献者遵循。可以按照自己的需要工作。尽管如此，官方的 FreeBSD Ports 仓库确实强制执行某些约定。例如，我们要求提交历史是简单、线性的，以使主分支的历史在 Git 下看起来类似于 Subversion 下的样子。为此，需要遵循一些约束，下文将讨论。其他项目使用不同的工作流，导致仓库主分支出现并行的路径。简而言之，Git 是灵活的，并没有适合所有人或项目的工作流。事实上，截至 2022 年初，FreeBSD 有工作组正在探索如何优化我们与 Git 的工作方式，因此可能会有后续改进。说明这些前提条件后，我们来探索适合为 FreeBSD Ports 树贡献的 Git 工作流。

## 安装 Git

在 FreeBSD 系统上安装 Git 的最简单方法是使用官方的 FreeBSD 包。

```sh
pkg install git
```

更多在 FreeBSD 上安装第三方软件的信息，请参阅 FreeBSD 手册中的“安装应用程序”章节。

## 克隆 Ports 树

如果你希望为树贡献新 Port，但还没有具体的想法，可以从 FreeBSD Wiki 上扫描请求的 Port 列表开始。假设我们希望为列表中的应用程序——Nyxt 浏览器——创建新 Port。第一步是克隆 FreeBSD Ports 仓库。如果你使用 ZFS，可能希望为开发 Ports 树创建专用数据集。

```sh
zfs create zroot/usr/home/ashish/freebsd/ports
```

当然，将 **zroot/usr/home/ashish/freebsd/ports** 替换为你的数据集布局。现在克隆仓库。你正在下载整个仓库，它包括超过 30,000 个 Ports 和 28 年的历史，因此这个过程会花费一些时间。

```sh
git clone -o freebsd --config remote.freebsd.fetch='+refs/notes/*:refs/notes/*'
https://git.freebsd.org/ports.git ~/freebsd/ports
```

`-o freebsd` 设置默认远程仓库的名称，用于协作（拉取和推送更改）。`--config remote.freebsd.fetch=+refs/notes/*:refs/notes/*` 将 Subversion 修订号添加到转换为 Git 之前提交的注释字段中。克隆完成后，你可以选择创建子 ZFS 数据集，构建 Ports 时存储软件分发文件。

```sh
zfs create zroot/usr/home/ashish/ports/distfiles
```

与 Ports 文件本身（主要是文本文件）不同，软件分发文件通常已经压缩，可以关闭 **zroot/usr/home/ashish/freebsd/ports/distfiles** 数据集的 ZFS 压缩。

```sh
zfs set compression=off zroot/usr/home/ashish/freebsd/ports/distfiles
```

你可以通过几种方式告诉 **make(1)** 你的 Ports 树的位置。第一种方法是将配置添加到 **/etc/make.conf** 文件中。

```c
.if ${.CURDIR:M/usr/home/ashish/freebsd/ports/*}
PORTSDIR=/usr/home/ashish/freebsd/ports
.endif
```

另一种方法是设置 `PORTSDIR` 环境变量。例如，如果你的 shell 是 zsh，可以将以下行添加到 **~/.zshrc** 文件中。

```sh
export PORTSDIR=/usr/home/ashish/freebsd/ports
```

如果你计划使用多个 Ports 树，像 `sysutils/direnv` 这样的工具可根据当前目录加载或卸载环境变量，非常有用。

## 保持更新

Ports 树处于活跃开发中，更改会经常推送到 **git.freebsd.org/ports.git**。要获取上游 FreeBSD 仓库中发生的更改，可以使用：

```sh
git -C ~/freebsd/ports fetch freebsd 
```

获取（Fetching）让你有机会在合并更改到本地分支之前检查这些更改。这里的 `-C ~/freebsd/ports` 指示 Git 在 **~/freebsd/ports** 目录下操作。如果当前工作目录是 **~/freebsd/ports**，从现在开始假设如此，那么这个标志可以省略。`freebsd` 参数表示从该远程仓库获取。

要列出推送到 `freebsd` 主分支但不属于本地主分支的提交，请运行：

```sh
git log --oneline main..freebsd/main 
```

在最上面的哈希值旁边，你会看到两个指针，`freebsd/main` 和 `freebsd/HEAD`。`HEAD` 通常是指向分支中最后一次提交的指针，在这种情况下，它和 `freebsd/main` 一样，指向远程仓库主分支的最后一次提交。如果我们运行：

```sh
git log --oneline freebsd/main
```

继续浏览提交列表时，我们最终会看到 `HEAD` 和 `main`，它们都指向本地主分支上的最后一次提交。要将来自 `freebsd/main` 的新提交集成到本地主分支中，请运行：

```sh
git merge freebsd/main --ff-only
```

`--ff-only`（仅快速前进）选项意味着只有在能够通过将主分支指针移动到与 `freebsd/main` 相同的提交时，才会将 `freebsd/main` 中的工作集成到主分支中。这只有在以下情况下才会发生：

```sh
git log --oneline main..freebsd/main
```

从本地主分支派生。如果对本地主分支做了修改，但这些修改不包含在 `freebsd/main` 中，`--ff-only` 将导致合并失败。这里描述的工作流中，我们永远不会直接修改本地主分支，因此这应该不会成为问题。但为了安全起见，可以配置合并命令始终使用 `--ff-only`，命令如下：

```sh
git config merge.ff only
```

作为便捷方式，有 `pull` 命令可以同时完成 `fetch` 和 `merge`。根据情况，使用 `pull` 可能不太明智，因为你无法检查将要合并到本地分支的内容。如果你的 Ports 仓库的主分支提交始终是 `freebsd/main` 中提交的子集（如这里所推荐的那样），这就不那么成问题了。为减少使用 `git pull` 时偏离 `freebsd/main` 的可能性，可以配置该命令仅执行快速前进合并（fast-forward merges），使用命令：

```sh
git config pull.ff only
```

## 创建本地分支

现在我们可以保持本地仓库副本与 **git.freebsd.org/ports.git** 上的仓库同步了，开始修改。Git 使用本地分支时真正展现了它的优势，本地分支提供了干净且高效的方式来组织进行中的工作。首先，创建新的特性分支来处理新的 Nyxt port。

```sh
git branch nyxt
```

现在使用以下命令切换到 nyxt 分支：

```sh
git checkout nyxt
```

创建并切换到分支的简写命令是：

```sh
git checkout -b nyxt
```

要检查当前检出的分支，可以运行：

```sh
git branch --show-current
```

你可能会发现将当前分支显示在命令行提示符中很有用。如果你使用 zsh，可以使用 Ports 树中的 `shells/git-prompt.zsh`。git-prompt-zsh 的很好功能是它会异步更新提示符，所以当 `git status` 或其他 Git 操作需要时间完成时，它不会阻塞其他工作。如果你喜欢这个功能，并且使用的不是 zsh，bash、fish 或 tcsh 也有类似的代码片段可以将 Git 状态信息显示在提示符中。

## 第一次提交

完成新的 Port 开发后，是时候提交更改了。首先，查看工作树的状态，使用：

```sh
git status
```

根据你所做的工作，这可能会告诉你添加 `SUBDIR += nyxt` 后，**www/Makefile** 文件被修改了，并且你还应该看到 **www/nyxt** 被标记为未跟踪文件。当你通过添加、编辑或删除文件与仓库中的文件系统交互时，你正在与 Git 的工作树交互。将更改提交到仓库之前，你必须先选择哪些更改将包含在下一个快照中。按 Git 的术语，你需要将文件从工作树添加到索引中。这个额外的步骤有用，因为它让你能够精确控制提交中包含的内容。要将所有更改添加到索引中，可以运行：

```sh
git add www/Makefile www/nyxt
```

现在，`git status` 会列出所有已修改或添加的文件，标记为已暂存并准备提交。然而，提交之前，还有一些一次性任务需要完成。Git 有钩子（hook）功能，可以在发生某些事件（如提交或合并）时执行自定义脚本。为配置 Git 查找 Ports 仓库中存储的与 Ports 相关的钩子，可以在当前工作目录位于仓库中的任何位置时运行：

```sh
git config --add core.hooksPath .hooks
```

该目录包含 `prepare-commit-msg` 钩子，它提供了有用的模板来格式化提交信息。我们还需要配置将用于创建提交信息的编辑器。Git 启动编辑器的顺序如下：`GIT_EDITOR` 环境变量的值、`core.editor` 配置变量、`VISUAL` 环境变量、`EDITOR` 环境变量。例如，我们可以告诉 Git 使用终端版的 Emacs 来编辑提交信息，方法是：

```sh
git config core.editor "emacs -nw"
```

如果你希望所有 Git 仓库都使用此编辑器，设置 `core.editor` 时添加 `--global` 选项：

```sh
git config --global core.editor "emacs -nw"
```

要提交更改，请运行：

```sh
git commit
```

你的编辑器现在应该显示提交模板，模板中提供了创建提交信息的提示。主题行应采用以下格式：`<变更的 Ports 树部分>: <变更的简要概述>`，理想情况下应少于 50 个字符。好的主题行可能是：

```sh
www/nyxt: (WIP) First attempt to port Nyxt browser.
```

空行之后，提交信息的正文提供更多详细信息。示例可能是：

```c
Makefile is still a skeleton.
TODO:
- Add _DEPENDS
- Add license information
- Fix QL_DEPS
- Add do-build target
```

保存并退出编辑器后，更改将提交。到目前为止，我们的更改从工作树进展到暂存区（索引），最终进入本地仓库。要检查你的提交，可以使用 `git log`，它还会确认 `HEAD` 和 `nyxt` 指针已经比主分支指针多了一次提交。

## 重写本地历史

Subversion 中提交意味着将你的更改发送到服务器，而 Git 中提交只是意味着在本地记录你的更改为新快照。因此，Git 中最好经常提交。当需要与他人共享你的工作时，你可以修改本地历史。有几种不同的方法可以重写历史。例如，如果你看到最近提交信息中的拼写错误，此时修复是好时机，因为你的更改仍然是本地的。要修改最近一次提交，请运行：

```sh
git commit --amend
```

并在编辑器中修改提交信息。如果你不小心没有在上次提交中暂存和提交 **www/Makefile** 的更改，只需运行 `git commit --amend` 之前暂存该文件，它会添加到上次提交中。关于重写历史的其他方法将在稍后讨论。

## 测试

请求审查之前，你的新 Port 必须经过测试。有两个 Port 检查工具可以提醒你常见的违规问题。使用以下命令安装它们：

```sh
pkg install portlint portfmt
```

要使用 portlint 检查你的 Port，在 **~/freebsd/ports/www/nyxt** 目录下运行：

```sh
portlint -AC
```

要使用 portclippy 检查你的 Port（来自 portfmt 包），也在 **~/freebsd/ports/www/nyxt** 目录下运行：

```sh
portclippy Makefile
```

请注意，虽然这些工具通常非常有用，但它们并不能捕捉所有错误，有时也会提出不太合理的建议。另一个有用的工具是 portfmt。顾名思义，它可以帮助格式化你的 Port 的 Makefile。

```sh
portfmt -D Makefile
```

## 使用 Poudriere 测试

《Porter’s Handbook》第 3.4 节描述了测试 Port 的步骤。它还将读者引导至第 10 章，其中包括设置 poudriere 的指南，poudriere 是 FreeBSD 的批量软件包构建器和 Port 测试工具。该章节介绍了使用 poudriere 测试的优点。“运行 poudriere testport 时会自动执行各种测试。强烈建议每个 Port 贡献者都安装并使用它来测试他们的 Port。”《Porter’s Handbook》这一章节描述了几种不同的方式来为 poudriere 设置 Ports。到达该部分时，可以告诉 poudriere 使用我们现有的 Ports，方法是

```sh
poudriere ports -c -m null -M ~/freebsd/ports
```

`-m` 选项告诉 poudriere 使用 null 方法，即使用 `-M` 参数指定位置的现有 Ports。使用 null 方法意味着我们将手动管理该树，包括保持其最新状态，测试时检出适当的分支。设置好 poudriere 后，你可以测试你的 Port。如果你创建了名为 `13amd64` 的 Jail，可以在该 Jail 中测试新 Port，方法是

```sh
poudriere testport -j 13amd64 www/nyxt
```

理想情况下，你应该在各种一级平台上测试你的 Port（当前为 12i386、12amd64、13amd64 和 13arm64）。构建新 Port 后，poudriere 可以构建包，并保持 Jail 运行，且包已经安装。

```sh
poudriere bulk -i -j 13amd64 <category>/<port>
```

选项 `-i` 告诉 poudriere 安装包后保持 Jail 运行。这对测试终端应用程序非常有用，但对像 nyxt 这样的图形应用程序则不适用。

如果 Port 有 OPTIONS，poudriere 会像官方包构建器一样测试和构建包，即选择默认的 OPTIONS。如果你想使用非默认选项来测试或构建包，可以运行

```sh
poudriere options -j 13amd64 www/nyxt
```

Poudriere 还会创建仓库，`pkg` 可以使用它安装包。如果你想在运行 poudriere 的同一系统上安装包，需要配置 `pkg` 使用该仓库。根据 **PKG.CONF(5)**，可以在 **/usr/local/etc/pkg/repos/** 下放置本地配置文件。文件名没有特殊要求，但必须以 `.conf` 结尾。要设置本地仓库配置并禁用默认的官方仓库，可以创建 **/usr/local/etc/pkg/repos/local.conf** 文件并输入以下内容：

```ini
FreeBSD: {
 enabled: no
}
Poudriere: {
 url: "file:///usr/local/poudriere/data/packages/13amd64-default"
}
```

上述路径假设 poudriere 的默认仓库位置，基于 13amd64 Jail 的仓库和默认的 Ports 树。

如果你想为远程主机提供包，需要配置 web 服务器。Poudriere 还提供了 web 界面，可以显示当前和过去的构建信息。如果你的 web 服务器是 nginx，可以在 **nginx.conf** 中添加如下的服务器配置来托管 poudriere 的界面和仓库：

```ini
server {
 listen 80 accept_filter=httpready;
 listen 443 ssl;
 server_name pkg.example.org;
 root /usr/local/share/poudriere/html;
 ssl_certificate /usr/local/etc/dehydrated/certs/example.org/fullchain.pem;
 ssl_certificate_key /usr/local/etc/dehydrated/certs/example.org/privkey.pem;
 # 如果你使用 dehydrated 作为 Let's Encrypt 客户端
 location /.well-known/acme-challenge {
alias /usr/local/www/dehydrated;
 }
 location /data {
 alias /usr/local/poudriere/data/logs/bulk;
 # 允许缓存动态文件，但确保它们会被重新检查。
 location ~* ^.+\.(log|txz|tbz|bz2|gz)$ {
 add_header Cache-Control "public, must-revalidate, proxy-revalidate";
 }
 # 不要记录 JSON 请求，因为它们频繁出现，
 # 并确保缓存按预期工作。
 location ~* ^.+\.(json)$ {
 add_header Cache-Control "public, must-revalidate, proxy-revalidate";
 access_log off;
 log_not_found off;
 }
 # 仅允许在日志目录中进行索引。
 location ~ /data/?.*/(logs|latest-per-pkg)/ {
 autoindex on;
 }
 break;
 }
 location /repo {
 alias /usr/local/poudriere/data/packages;
 autoindex on;
 }
}
```

如果你想在浏览器中显示 poudriere 的软件包构建日志，可以编辑 Nginx 的 **mime.types** 文件，将 text/plain 行修改为包含 .log 后缀，告知 Nginx 处理文本文件：

```sh
text/plain log txt; 
```

使用 `service nginx restart` 重启 Nginx 后，在浏览器中访问 <http://pkg.example.org> 查看 poudriere 的 web 界面。

## 为准备审查而重写历史

分享你的工作之前，提交历史应该井井有条，包括提交日志和提交次数。假设你的 nyxt 分支的历史包含七个 WIP（进行中的工作）提交。

```sh
% git log --oneline
061be9ca5d98 (HEAD -> nyxt) www/nyxt: (WIP) ready for testing
cddad2b5886b www/nyxt: (WIP) Add missing www/Makefile entry
e42f79383312 www/nyxt: (WIP) Add build and install targets
807099e08e33 www/nyxt: (WIP) Fix QL_DEPENDS
3cc5f266b434 www/nyxt: (WIP) Complete _DEPENDS
80d098cd8367 www/nyxt: (WIP) Add license information
9ec91c5fb244 www/nyxt: (WIP) First attempt to port Nyxt browser
9f77e9601564 (freebsd/main, freebsd/HEAD, main) net-im/toxic: upgrade to v0.11.2
```

freebsd/main、freebsd/HEAD 和 main 指针之上的提交是你在 nyxt 分支中的提交，需要整理这些提交。

```sh
git rebase -i main
```

将显示本地 nyxt 分支中的提交日志。-i 选项意味着此次变基操作将是交互式的。我们指定要修改的提交子集之前的提交。这里最简单的方式是使用 main 指针来指定该提交。我们也可以使用波浪符语法，例如 HEAD~7，表示 HEAD 之前的七个提交，但计算这七个提交比较麻烦。

这是你在编辑器中应该看到的内容：

```sh
pick 9ec91c5fb244 www/nyxt: (WIP) First attempt to port Nyxt browser
pick 80d098cd8367 www/nyxt: (WIP) Add license information
pick 3cc5f266b434 www/nyxt: (WIP) Complete _DEPENDS
pick 807099e08e33 www/nyxt: (WIP) Fix QL_DEPENDS
pick e42f79383312 www/nyxt: (WIP) Add build and install targets
pick cddad2b5886b www/nyxt: (WIP) Add missing www/Makefile entry
pick 061be9ca5d98 www/nyxt: (WIP) Ready for testing
# Rebase 9f77e9601564..061be9ca5d98 onto 9f77e9601564 (7 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup [-C | -c] <commit> = like "squash" but keep only the previous
# commit's log message, unless -C is used, in which case
# keep only this commit's message; -c is same as -C but
# opens the editor
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name 
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# . create a merge commit using the original merge commit's
# . message (or the oneline, if no original merge commit was
# . specified); use -c <commit> to reword the commit message
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
```

历史记录是按时间顺序写的，较旧的提交位于顶部。下面的注释列出了我们可以使用的命令。我们通过将这些命令放在提交旁边来指示 Git 如何修改历史记录。每个提交旁边的默认命令是 pick，即保持提交不变。这里，我们希望将这些 WIP 提交压缩为单一提交以供审核。为将最新的六个提交压缩到第一个提交中，将这些底部六个提交旁边的 pick 命令更改为 squash。

```sh
pick 9ec91c5fb244 www/nyxt: (WIP) First attempt to port Nyxt browser
squash 80d098cd8367 www/nyxt: (WIP) Add license information
squash 3cc5f266b434 www/nyxt: (WIP) Complete _DEPENDS
squash 807099e08e33 www/nyxt: (WIP) Fix QL_DEPENDS
squash e42f79383312 www/nyxt: (WIP) Add build and install targets
squash cddad2b5886b www/nyxt: (WIP) Add missing www/Makefile entry
squash 061be9ca5d98 www/nyxt: (WIP) Ready for testing
```

保存并退出编辑器时，Git 会完成变基操作，然后在编辑器中显示提交日志，以便你为新的单一提交写新的日志信息。下面是我们在与他人分享工作并请求审核时可能想要使用的示例提交信息。

```sh
www/nyxt: New port for the Nyxt browser

Nyxt is a keyboard-driven web browser designed for power users.
Inspired by Emacs and Vim, it has familiar key-bindings and is
infinitely extensible in Lisp.

WWW: https://nyxt.atlas.engineer/
```

撰写优质 FreeBSD 提交信息的更深入讨论，请参阅 2020 年 11 月的《期刊》文章。现在，运行 `git log --oneline` 将在我们的 nyxt 分支显示单一提交。

```sh
7392483f6147 (HEAD -> nyxt) www/nyxt: New port for the Nyxt browser
9f77e9601564 (freebsd/main, freebsd/HEAD, main) net-im/toxic: upgrade to v0.11.2
```

我们希望重写历史的另一种方式是将 nyxt 分支上的工作重新基于最新的 main 分支。首先，更新 main 分支。

```sh
git checkout main
git pull
```

然后切换回 nyxt 分支，并告诉 Git 执行 rebase。

```sh
git checkout nyxt
git rebase main
```

如果一切顺利，`git log` 将显示 nyxt 分支中的提交，这些提交会基于来自 main 分支的最新提交。如果在 freebsd/main 和你的 nyxt 分支中都有冲突的更改，Git 会通知你哪些文件存在冲突，并让你有机会手动解决这些冲突。

```sh
~/freebsd/ports [nyxt|✔] % git rebase main
Auto-merging www/Makefile
CONFLICT (content): Merge conflict in www/Makefile
error: could not apply 531d9081dfb1... Add new entry for nyxt browser
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit: run "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase
--abort".
Could not apply 531d9081dfb1... Add new entry for nyxt browser 
```

我们可以看到冲突出现在 **www/Makefile**，Git 会告诉我们有哪些选项可以手动解决冲突。下面是我们可能在 **www/Makefile** 中看到的示例。

```sh
<<<<<<< HEAD
SUBDIR += nyan
||||||| parent of 531d9081dfb1 (Add new entry for nyxt browser)
=======
SUBDIR += nyxt
>>>>>>> 531d9081dfb1 (Add new entry for nyxt browser)
```

在这种情况下，手动修复冲突很直接。我们要在 `nyan` 的新条目下方添加 `nyxt` 条目。编辑文件使其看起来像这样：

```sh
SUBDIR += nyan
SUBDIR += nyxt
```

告诉 Git 我们准备继续操作，使用命令：

```sh
git add www/Makefile
git rebase --continue 
```

将你的功能分支 rebase 到更新后的主分支是你经常需要做的操作，因此你可能会想使用便捷脚本来一步完成它。下面是简单的示例。在功能分支上运行 `rum`，即可一步完成 rebase 操作。

```sh
#!/bin/sh
# rum, r_ebase onto u_pdated m_ain
#
# Usage: rum
#
# globals expected in ${HOME}/.ports.conf with sample values
# No leading '/' on directory names means they are relative to $HOME
# portsd='/usr/home/ashish/ports' # ports 目录
. "$HOME/.ports.conf"
usage () {
 cat <<EOF 1>&2
Usage: ${0##*/}
EOF
}
############################################ main
[ $# != 0 ] && { usage; exit 1; }
[ -n "${portsd##/*}" ] && portsd="${HOME}/$portsd"
# current branch
cb="$(git -C "$portsd" branch --show-current)"
if [ -z "$cb" ]; then
 printf "Could not determine the current branch.\\"
 exit 1
elif [ "$cb" = "main" ]; then
 printf "The main branch is checked out.\\n"
 exit 1
fi
git -C "$portsd" checkout main && \
 pull && \
 git -C "$portsd" checkout "$cb" && \
 git rebase main
```

## 提交工作以供审查

现在我们准备好将工作提交审查。FreeBSD 目前有两种方法来完成这项操作。Bugzilla 用于提交 bug，Phabricator 用于审查源代码更改。两者都接受补丁，但 Phabricator 具有 Bugzilla 中缺少的有用功能，例如允许审查者针对补丁中的一行或多行添加评论。为涵盖这两种方法，首先在 Phabricator 中创建审查，然后在 Bugzilla 中创建新 bug，指向 Phabricator 审查。

### FreeBSD Phabricator 审查

要开始使用 FreeBSD 的 Phabricator 实例做代码审查（网址为 <https://reviews.freebsd.org>），你必须首先创建账户，然后安装 arcanist 命令行工具。

```sh
pkg install arcanist-php80
```

运行以下命令设置 **~/.arcrc** 文件并配置所需的证书：

```sh
arc install-certificate https://reviews.freebsd.org
```

并按照指示操作。接下来，将 Arcanist 配置为使用 <https://reviews.freebsd.org> 作为默认 URI。

```sh
arc set-config default https://reviews.freebsd.org/
```

要提交你的审查，从 nyxt 分支运行

```sh
arc diff --create main
```

这将创建新的审查，包含 nyxt 分支中的所有提交。在这个例子中，我们将提交合并成了单一提交，因此修订将以该单一提交创建。当你的编辑器打开时，你将有机会编辑修订中的各个字段。顶部一行将是你的提交日志的主题，如 `www/nyxt: New port for the Nyxt browser`，摘要将包含其余的提交日志内容。测试计划下，你可以列出你为测试 Port 所做的工作。例如，如果你为每个受支持版本在 tier 1 架构上执行了 `poudriere testport`，你可以写道：

```sh
poudriere testport 12/13 amd64/aarch64
```

你还必须至少添加审查人。如果你有一个或多个与之合作过的 Port 提交者，你可以在此添加他们的用户名。例如：

```sh
Reviewers: ashish rene
```

你还可以指定组审查人，格式为 #group_name，例如 #ports_committers。Subscribers: 字段与 Reviewers: 字段类似，接受用户列表，但这些用户不会拒绝或批准你的工作。审查人请求更改时，你可以使用以下命令更新修订版本：

```sh
arc diff --update <revision>
```

其中 `<revision>` 是修订版本 ID，格式为 DXXXXX。它可以在创建修订版本时发送到你邮箱的邮件中找到。例如，如果你的修订版本位于 <https://reviews.freebsd.org/D33314>，那么就使用 D33314 作为 `<revision>`。

### 提交 Bugzilla 错误报告

要创建新的 Bugzilla 错误报告，请访问 <https://bugs.freebsd.org> 并点击页面顶部的 New 链接。如果你没有登录到 FreeBSD Bugzilla 实例，系统会提示你登录。如果你没有 FreeBSD Bugzilla 账户，可以使用登录页面上的链接创建新账户。

在这里，选择 Ports & Packages 链接，因为我们正在创建新 Port，并选择 Individual Port(s) 作为组件。特定 Port 的错误，错误的主题行可以是提交的主题，前缀加上 [NEW PORT]，例如：[NEW PORT] www/nyxt: New port for the Nyxt browser。如果该 Port 不是新 Port，类别/Port 前缀会自动将错误分配给该 Port 的维护者。描述中可以添加其余的提交信息和任何对其他阅读错误的人有帮助的信息。如果你创建了 Phabricator 审查，可以将其添加到 See also 中。

当你的新 Port 被接受并推送到 **git.freebsd.org/ports.git** 后，你作为该 Port 维护者的新工作开始了。Port 维护者责任的大纲，请参考《The challenge for port maintainers》一文。为跟上上游进展，portscout 是有用的服务，能够在有新版本发布时提醒你，以便你可以提交 Port 更新。如果上游使用 GitHub，你还可以通过关注 Watch 和 Custom 链接，检查项目页面上的 Releases 以获取新版本的提醒。更新 Port 时，如果更改简单（例如仅 DISTVERSION/distinfo 更改），可能不需要提交 Phabricator 审查。在 Git 特性分支上，你可以使用 git format-patch main 创建补丁，并将其附加到新的 Bugzilla 错误报告中。使用 Git 后，我们为贡献者署名时有了更多灵活性。当你以这种方式提交补丁，并且提交者将其推送到 **git.freebsd.org/ports.git** 时，git log 会为你的工作署名。即使你提交了传统的 diff，提交者也可以选择将你设为作者。

### 主观结论

变革可能困难。许多在 Subversion 上投入大量时间并且变得高效的 FreeBSD 开发者和贡献者，转向全新的版本控制系统，尤其是基本原理上完全不同的系统，感到不情愿。我们失去了一些实用的特性，比如简单的、单调递增的提交修订号和目录与文件移动时的确定性历史保留。然而，经历了大约九个月的时间后，大多数迹象表明开发者和更广泛的社区对这一变化感到满意，并且能够高效工作。虽然很难单独归因于某些结果，但从转换日期（2021-04-06）到写作日期（2021-12-31），提交到 Ports 的次数为 29,238 次，比去年同期多了 1,748 次。希望这将是 Ports 贡献的持续趋势。

---

**JOE MINGRONE** 是 FreeBSD Ports 开发者，并为 FreeBSD 基金会工作。他同妻子和两只猫一起生活在加拿大新斯科舍省的达特茅斯。
