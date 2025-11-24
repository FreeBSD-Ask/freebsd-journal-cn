# 用 Git 更新 FreeBSD

- 原文链接：[Updating FreeBSD from Git](https://freebsdfoundation.org/wp-content/uploads/2021/08/Updating-FreeBSD-From-Git.pdf)
- 作者：**DREW GURKOWSKI**

随着 FreeBSD 正在从 Subversion 迁移到 Git，更新 FreeBSD 源代码的系统也进行了相应的调整。本指南将介绍如何从 Git 获取源代码、如何更新它们以及如何进行源代码的二分查找。它旨在为一般用户提供新的操作机制的入门。

## 1. 保持 FreeBSD 源代码树的最新状态

首先，必须下载源代码树。这是一个相当简单的过程。第一步：克隆源代码树。这将下载整个源代码树。有两种方式下载。在默认情况下，Git 会进行深度克隆，这符合大多数人的需求。然而，在某些情况下，您可能希望进行浅克隆。

### 1.1 分支名称

新 Git 仓库中的分支名称与 Subversion 中的名称类似。对于稳定分支，它们是 stable/X，其中 X 是主版本号（如 12 或 13）。新仓库中的开发分支是 main。

注意：如果您省略下面的分支选项 `-b` ，则默认使用 main 分支。

### 1.2 仓库

• 官方的地理分布式镜像供公众使用，访问 URL 为：https://git.freebsd.org/src.git  
• 该仓库也可以通过 SSH 访问：ssh://anongit@git.freebsd.org/src.git  
• 还有几个官方维护的外部镜像，列表可在 https://docs.freebsd.org/en/books/handbook/mirrors/#external-mirrors 查找  
• 如果使用 Web 浏览器查看内容，可以通过 cgit Web 界面访问：https://cgit.freebsd.org/src  
• 还有一个旧的实验性 GitHub 仓库，位于 https://github.com/freebsd/freebsd-legacy/（以前是 https://github.com/freebsd/freebsd），与新的 Git 仓库类似。然而，GitHub 仓库中存在大量错误，在我们将 Git 仓库作为项目的真源时，需要重新生成该仓库的导出。

这些仓库的哈希值是不同的。要从旧仓库迁移到新仓库，请参阅 https://github.com/freebsd/freebsd-legacy/commit/de1aa3dab-23c06fec962a14da3e7b4755c5880cf

在以下命令中，使用您选择的仓库替换 `$URL`。

#### 1.2.1 从 Ports/Pkg 安装 git

在克隆源代码树之前，需要先安装 git。最简单的安装方法是通过包管理器：

```sh
# pkg install git
```

还有 git-lite 和 git-tiny 两个仅包含基本依赖的包。这些包足以执行本文中的命令。

#### 1.2.2 深度克隆

深度克隆会拉取整个源代码树以及所有的历史记录和分支。这是最简单的方法。它还允许你使用 git 的 worktree 功能，将所有活动的分支检出到不同的目录中，但只保留一个仓库副本。

```sh
% git clone $URL -b branch [dir]
```

这是进行深度克隆的命令。branch 应该是 1.1 节中列出的分支之一，如果省略，它将使用仓库的默认分支：main。Dir 是一个可选的目录，用来放置克隆的内容（默认情况下将使用仓库的名称，如 src）。例如：

```sh
% git clone https://git.freebsd.org/src.git
```

如果你对历史感兴趣，计划进行本地更改，或计划在多个分支上工作，那么你会希望使用深度克隆。它也是最容易保持最新的。如果你对历史感兴趣，但只在一个分支上工作且空间有限，也可以使用 --single-branch 来只下载一个分支（尽管一些合并提交可能不会引用合并自的分支，对于一些用户来说，这可能对详细的历史版本很重要）。

#### 1.2.3 浅克隆

浅克隆仅复制最新的代码，而没有或仅有少量历史记录。当你需要构建 FreeBSD 的特定修订版本，或者刚刚开始并计划更全面地跟踪源代码树时，浅克隆非常有用。你也可以用它来限制历史记录，仅包含若干个修订版本。

```sh
% git clone -b branch --depth 1 $URL [dir]
```

使用默认分支的示例如下：

```sh
% git clone --depth 1 https://git.freebsd.org/src.git
```

这会克隆仓库，但只包含仓库中的最新修订版。其余的历史记录不会被下载。如果你以后改变主意，可以使用 `git fetch --unshallow` 来获取完整的历史记录。

## 2. 从源代码构建和更新

克隆 FreeBSD 仓库后，下一步是从源代码构建。构建过程相对没有变化，仍然使用 `make` 和 `install`。Git 提供了 `git pull` 和 `git checkout` 来更新和选择特定的分支或修订版本。

### 2.1 从源代码构建

可以按照手册中的描述进行构建，例如：

```sh
% cd src
% make buildworld
% make buildkernel
% make installkernel
% make installworld
```

注意，你可以指定 `-j` 来加速并行构建。

### 2.2 从源代码更新

以下命令将更新两种类型的源代码树。这将拉取自上次更新以来的所有修订版本。

```sh
% git pull --ff-only
```

这将更新源代码树。在 git 中，快速前进合并是指只需要设置一个新的分支指针，而无需重新创建提交。通过始终执行快速前进合并/拉取，你可以确保拥有与 FreeBSD 树相同的副本。如果你想保持本地补丁，这一点非常重要。

见下文了解如何管理本地更改。最简单的方式是使用：

```sh
% git pull --autostash
```

但也有更复杂的选项可供选择。

### 2.3 选择特定的修订版

在 git 中，`git checkout` 命令不仅可以用于切换分支，也可以用于检出特定的修订版本。git 的修订版是长哈希值，而不是一个顺序数字。

当你检出特定的修订版时，只需在命令行中指定你想要的哈希值（`git log` 命令可以帮助你决定想要的哈希值）：

```sh
% git checkout 08b8197a74
```

然而，与许多 git 操作一样，这并不像看起来那么简单。你将看到类似以下的消息：

```c
Note: checking out ‘08b8197a742a96964d2924391bf9fdfeb788865d’.
You are in ‘detached HEAD’ state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.
If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:
git checkout -b <new-branch-name>
HEAD is now at 08b8197a742a hook gpiokeys.4 to the build
```

其中最后一行是从你正在检出的哈希生成的，而第一行则是该修订版的提交消息。哈希值也可以缩写。这就是为什么你会在不同命令或其输出中看到它们具有不同长度的原因。这些超长哈希通常在某些字符之后是唯一的，具体取决于仓库的大小，因此 git 允许你缩写哈希值，而且在呈现给用户时有所不一致。使用 `git rev-parse --short <full-hash>` 可以显示足够长度的短哈希，以便在仓库中区分开来。当前 FreeBSD 源仓库的哈希长度为 12。

## 3. 二分查找/其他注意事项

### 3.1 使用 `git bisect` 进行二分查找

有时候，事情会出错。最后一个修订版有效，但你刚更新的修订版无效。开发人员可能会要求你使用二分查找来追踪哪个提交导致了回归问题。

如果你读过上一节，可能会想，"我该怎么在这种疯狂的修订号下做二分查找？" 那么本节正是为你准备的。如果你没有这么想，但也想进行二分查找，那也是为你准备的。

幸运的是，git 提供了命令 `git bisect` 。这里是如何使用它的简要概述。更多信息，我建议查阅 https://git-scm.com/docs/git-bisect。man 页面对于描述可能出错的情况、在修订版无法构建时该怎么办、何时想使用其他术语（如 good 和 bad）等问题非常有用，这些内容在这里不会详细讲解。

`git bisect start` 将启动二分查找过程。接下来，你需要指定一个范围。`git bisect good XXXXXX` 会告诉它哪个修订版是正常的，`git bisect bad XXXXX` 会告诉它哪个修订版是有问题的。通常情况下，坏的修订版就是 HEAD（它是你当前检出的特殊标签）。好的修订版通常是你最后一次检出的修订版。

顺便提一下：如果你想知道你最后检出的修订版，可以使用 `git reflog`：

```sh
5ef0bd68b515 (HEAD -> master, origin/master, origin/HEAD) HEAD@{0}: pull --ff-only: Fast-forward
a8163e165c5b (upstream/master) HEAD@{1}: checkout: moving from b6fb97efb682994f59b21fe4efb3fcfc0e5b9eeb to master
...
```

显示了我将工作树切换到 master 分支（a816...），然后从上游更新（到 5ef0...）。在这种情况下，坏的修订版是 HEAD（或 5rf0bd68），好的修订版是 a8163e165。从输出中可以看到，`HEAD@{1}` 通常也有效，但如果在更新之后但在发现需要进行二分查找之前对 git 树做了其他修改，它就不一定可靠。

回到 `git bisect`。首先设置好的修订版，然后设置坏的修订版（不过顺序无关紧要）。当你设置坏的修订版时，它会给你一些关于该过程的统计信息：

```sh
% git bisect start
% git bisect good a8163e165c5b
% git bisect bad HEAD
```

```sh
Bisecting: 1722 revisions left to test after this (roughly 11 steps)
[c427b3158fd8225f6afc09e7e6f62326f9e4de7e] Fixup r361997 by balancing parens.
```

然后，你需要构建/安装该修订版。如果它是好的，你可以输入 `git bisect good`，否则输入 `git bisect bad`。每一步你都会看到类似于上述的消息。当你完成时，将坏的修订版报告给开发者（或者自己修复 bug 并发送补丁）。`git bisect reset` 会结束该过程，并将你返回到最初的位置（通常是 `main` 的最新提交）。同样，`git-bisect` 手册（如上链接）是当事情出错或遇到不寻常的情况时的一个很好的资源。

### 3.2 文档和 Ports 考虑事项

文档树是第一个转换为 git 的仓库。该仓库只有一个开发分支 `main`，包含了 [https://www.freebsd.org](https://www.freebsd.org) 和 [https://docs.freebsd.org](https://docs.freebsd.org) 的源代码。

- 仓库 URL 是 [https://git.freebsd.org/doc.git](https://git.freebsd.org/doc.git)
- 该仓库也可以通过 SSH 访问：`ssh://anongit@git.freebsd.org/doc.git`
- 以及 cgit Web 仓库浏览器：[https://cgit.freebsd.org/doc](https://cgit.freebsd.org/doc)

Ports 树以类似的方式操作。分支名称不同，仓库的位置也不同。

- 仓库 URL 是 [https://git.freebsd.org/ports.git](https://git.freebsd.org/ports.git)
- 该仓库也可以通过 SSH 访问：`ssh://anongit@git.freebsd.org/ports.git`
- 以及 cgit Web 仓库浏览器：[https://cgit.freebsd.org/ports](https://cgit.freebsd.org/ports)

与 ports 一样，最新的开发分支是 `main`。季度分支与 FreeBSD 的 svn 仓库中的命名相同。它们用于最新和季度分支的 `pkg`。


### 3.3 处理本地更改

这里有一些适用于跟踪 FreeBSD 的更高级用户的话题。如果你没有本地更改，你可以停止阅读（这是最后一节，可以跳过）。

一个重要的事项：所有更改在推送之前都是本地的。与 svn 不同，git 使用分布式模型。对于用户来说，大多数情况下差别不大。然而，如果你有本地更改，你可以使用相同的工具来管理它们，就像你用来从 FreeBSD 拉取更改一样。所有未推送的更改都是本地的，并且可以轻松修改（`git rebase`，将在下面讨论）。

### 3.4 保留本地更改

保留本地更改（尤其是微小的更改）最简单的方法是使用 `git stash`。在最简单的形式下，你使用 `git stash` 来记录更改（这会将它们推送到堆栈中）。大多数人使用此方法在更新树之前保存更改，正如上面所述。然后，他们使用 `git stash apply` 将更改重新应用到树中。`stash` 是一组可以通过 `git stash list` 查看更改的堆栈。`git-stash` 手册页（[https://git-scm.com/docs/gitstash](https://git-scm.com/docs/gitstash)）包含了所有详细信息。

这种方法适用于你对树进行微小调整的情况。当你有非微小的更改时，可能最好保持一个本地分支并进行 rebase。它还与 `git pull` 命令集成：只需在命令行中添加 `--autostash`。

## 4. FreeBSD 分支

### 4.1 保持本地分支

与 subversion 相比，使用 git 保持本地分支要容易得多。在 subversion 中，你需要合并提交并解决冲突。这是可以管理的，但可能导致一个复杂的历史，如果以后需要将其推送到上游，或需要复制它时可能会很困难。Git 也允许合并，但同样存在这些问题。这是管理分支的一种方式，但它的灵活性最差。

Git 有一个 "rebase" 的概念，你可以使用它来避免这些问题。`git rebase` 命令基本上会相对于父分支的更新位置重新播放所有提交。此部分将简要介绍如何操作，但不会涵盖所有情况。


### 4.2 创建一个分支

假设你想修改 FreeBSD 的 `ls(1)` 命令，使其永远不使用颜色显示。做这个修改有很多原因，但这个例子将以此为基准。FreeBSD 的 `ls(1)` 命令会不时发生变化，你需要处理这些变化。幸运的是，通过 `git rebase`，这通常是自动完成的。

```sh
% cd src
% git checkout main
% git checkout -b no-color-ls
% cd bin/ls
% vi ls.c # 将更改合并进去
% git diff # 检查合并
```

```c
diff --git a/bin/ls/ls.c b/bin/ls/ls.c
index 7378268867ef..cfc3f4342531 100644
--- a/bin/ls/ls.c
+++ b/bin/ls/ls.c
@@ -66,6 +66,7 @@ __FBSDID("$FreeBSD$");
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
+#undef COLORLS
#ifdef COLORLS
#include <termcap.h>
#include <signal.h>
```

```sh
% # these look good, make the commit...
% git commit ls.c
```

提交时，系统会弹出一个编辑器，供你描述所做的更改。一旦输入完毕，你就拥有了自己在 git 仓库中的本地分支。像平常一样，按照手册中的指示进行构建和安装。git 与其他版本控制系统的不同之处在于，你必须明确告诉它使用哪些文件。这里是在提交命令行中完成的，但你也可以使用 git add 来完成，许多更深入的教程会涉及这一点。

## 5. 更新到新的 FreeBSD 版本

### 5.1 更新到新的 FreeBSD 修订版

当需要引入新修订版时，操作几乎与没有分支时一样。你将像上面所述那样更新，但在更新之前和之后会有一个额外的命令。以下假设你是从一个未修改的树开始的。开始进行 rebase 操作时，从干净的树开始是非常重要的（git 通常要求这样）。

```sh
% git checkout main
% git pull
% git rebase -i main no-color-ls
```

这将弹出一个编辑器，列出其中的所有提交。在这个例子中，不要做任何更改。这通常是在更新基线时进行的操作（虽然你也可以使用 git rebase 命令来整理你在分支中的提交）。

完成上述操作后，你就已经将 ls.c 中的提交从 FreeBSD 的旧修订版更新到了较新的修订版。

有时会出现合并冲突。没关系，不要慌张。你可以像处理任何其他合并冲突一样处理它们。为了简单起见，我将仅描述一个你可能会看到的常见问题。关于更详细的处理方法，可以在本节末尾找到相关链接。

假设这包括了上游对 terminfo 的 radical 更改以及选项名称的更改。当你更新时，你可能会看到类似这样的内容：

```c
Auto-merging bin/ls/ls.c
CONFLICT (content): Merge conflict in bin/ls/ls.c
error: could not apply 646e0f9cda11... no color ls
Resolve all conflicts manually, mark them as resolved with
"git add/rm <conflicted_files>", then run "git rebase --continue".
You can instead skip this commit: run "git rebase --skip".
To abort and get back to the state before "git rebase", run "git rebase --abort".
Could not apply 646e0f9cda11... no color ls
```

这看起来很吓人。如果你打开编辑器，你会看到它是一个典型的三方合并冲突解决方式，可能你在其他源代码管理系统中也见过（其余的 ls.c 内容已省略）：

```c
<<<<<<< HEAD
#ifdef COLORLS_NEW
#include <terminfo.h>
=======
#undef COLORLS
#ifdef COLORLS
#include <termcap.h>
>>>>>>> 646e0f9cda11... no color ls
```

新代码在前，你的代码在后。正确的修复方法是先在 `#ifdef` 前添加一个 `#undef COLORLS_NEW`，然后删除旧的更改：

```c
#undef COLORLS_NEW
#ifdef COLORLS_NEW
#include <terminfo.h>
```

保存文件。由于 rebase 被中断，因此你需要完成它：

```c
% git add ls.c
% git rebase --cont
```

这告诉 git ls.c 已经改变，并继续执行 rebase 操作。由于存在冲突，你将被带入编辑器，可能需要更新提交信息。

如果在 rebase 过程中遇到困难，不要慌张。使用 `git rebase --abort` 可以让你回到干净的状态。然而，重要的是要从未修改的树开始。

关于这个主题的更多信息，可以参考 [freeCodeCamp 的最终指南](https://www.freecodecamp.org/news/the-ultimate-guide-to-gitmerge-and-git-rebase/)，它对这个话题进行了非常详细的讲解。它涵盖了我为了简化而没有涉及的很多案例，这些案例是非常有用的，因为它们时常会出现。

### 5.2 更新到新的 FreeBSD 分支

假设你想将 FreeBSD 从 stable/12 升级到 current。只要你有一个深度克隆，这也很容易做到：

```sh
% git checkout main
% # 在这里构建并安装...
```

这样就完成了。如果你有一个本地分支，则有一两个注意事项。首先，rebase 会重写历史记录，因此你可能需要做点什么来保存它。其次，跳转分支往往会遇到更多的冲突。如果我们假设上面的例子是相对于 stable/12 的，那么要跳转到 main，我建议执行以下操作：

```sh
% git checkout no-color-ls
% git checkout -b no-color-ls-stable-12 # 为此分支创建另一个名称
% git rebase -i stable/12 no-color-ls --onto main
```

上述命令会做的是，首先切换到 no-color-ls 分支。然后为它创建一个新名称（no-color-ls-stable-12），以防你需要返回到它。接着，你会将它 rebase 到 main 分支上。这将找到当前 no-color-ls 分支的所有提交（直到它与 stable/12 分支合并的地方），然后将这些提交重放到 main 分支上，创建一个新的 no-color-ls 分支（这就是为什么我让你创建一个占位名称）。

## 参考文献

[1] 使用 Git，FreeBSD 手册，https://docs.freebsd.org/en/books/handbook/mirrors/#git  
[2] Git，FreeBSD 维基，https://wiki.freebsd.org/Git  
[3] 从源代码更新 FreeBSD，FreeBSD 手册，https://docs.freebsd.org/en/books/handbook/cutting-edge/#makeworld

---

**DREW GURKOWSKI**，来自 FreeBSD 基金会


