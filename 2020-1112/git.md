# FreeBSD Git 快速入门

- 原文链接：[FreeBSD mini-Git Primer](https://freebsdfoundation.org/wp-content/uploads/2021/01/Mini-Git.pdf)
- 作者：**WARNER LOSH**

FreeBSD 项目已开始从 Subversion 迁移到 Git。这一举措经过一年多的规划，代表了 FreeBSD 持续改进工作流程的下一步。项目希望 Git 更庞大的生态系统能帮助其改进持续集成（CI）流程，使补丁提交更加便捷，并整体上提高项目质量。  

本文面向下载 FreeBSD 源码、进行本地修改，并偶尔向项目贡献修改的用户。它将介绍 FreeBSD 在 Git 方面的使用，适用于已经大致熟悉 Git 基础知识的读者。在可能的情况下，本文会提供更深入的 Git 资料。网上有许多 Git 入门指南，其中《Git Book》是较为优秀的参考资料之一。  

## 跟进 FreeBSD src 代码库的最新动态  

### 第一步：克隆代码库  

克隆操作会下载整个代码库。有两种下载方式。大多数用户会选择完整克隆（deep clone）代码库，但在某些情况下，您可能希望执行浅克隆（shallow clone）。  

### 分支名称  

新 Git 代码库的分支名称与旧名称类似。对于稳定分支，其格式为 `stable/X`，其中 `X` 代表主要版本号（如 11 或 12）。新代码库的主分支是 `main`，而旧 GitHub 镜像的主分支是 `master`。两者都反映了它们创建时 Git 的默认设置。如果在以下命令中省略 `-b branch` 或 `--branch branch` 选项，则默认使用 `main/master` 分支。  

### 代码库  

目前有两个代码库，它们的提交哈希不同。旧的 GitHub 代码库与新的 cgit 代码库类似。然而，由于 GitHub 代码库中存在大量错误，在迁移到 Git 作为项目的最终事实来源时，我们不得不重新导出数据。  

GitHub 代码库地址：  
[https://github.com/freebsd/freebsd.git](https://github.com/freebsd/freebsd.git)  

新的正式代码库地址：  

- HTTPS 方式：[https://git.freebsd.org/src.git](https://git.freebsd.org/src.git)  
- SSH 方式：`ssh://anonssh@git.freebsd.org/src.git`  

在以下命令中，这些地址将被表示为 `$URL`。  

>**注意**
>
>FreeBSD 项目并未使用 Git 子模块（submodules），因为它们不适用于我们的工作流程和开发模式。我们如何跟踪第三方应用程序的变更在其他地方有详细讨论，并且对于一般用户而言无需关心。

### 完整克隆（Deep Clone）  

完整克隆会下载整个代码库，包括所有历史记录和分支。这是最简单的方式，同时还能利用 Git 的 worktree 功能，将所有活跃分支检出到不同的目录，而只保留一份代码库副本。  

```sh
% git clone -o freebsd $URL -b branch [目录]
```

以上命令用于执行完整克隆。`branch` 应该是前一节中列出的分支之一。如果是 `main/master` 分支，则可以省略。`目录` 是可选参数，指定克隆后的存放位置（默认为克隆的代码库名称，如 `freebsd` 或 `src`）。  

如果您对历史记录感兴趣、计划进行本地修改，或者需要在多个分支上工作，完整克隆是最佳选择。此外，这种方式也最容易保持代码库的最新状态。如果您只对某个分支的历史感兴趣，并且磁盘空间有限，可以使用参数 `--single-branch` ，仅下载指定分支的历史记录（但某些合并提交可能不会引用被合并的分支，这对某些需要详细历史记录的用户来说可能很重要）。  

### 浅克隆（Shallow Clone）  

浅克隆仅复制最新的代码，而不会下载完整的历史记录或只下载一部分历史记录。这种方式适用于需要构建特定版本的 FreeBSD，或者刚开始使用并计划逐步跟踪代码库的情况。此外，它还可以用于限制历史记录的下载深度。  

```sh
% git clone -o freebsd -b branch --depth 1 $URL [目录]
```

此命令会克隆代码库，但只包含最近的版本，不会下载完整的历史记录。如果之后决定获取完整历史记录，可以使用以下命令补充历史记录：  

```sh
% git fetch --unshallow
```

### 编译（Building）  

下载完成后，编译方式与 FreeBSD 手册中的描述一致，例如：  

```sh
% cd src
% make buildworld
% make buildkernel
% make installkernel
% make installworld
```

本文不再深入介绍编译过程。

### 更新（Updating）  

更新完整克隆和浅克隆的代码库使用相同的命令。以下命令会获取自上次更新以来的所有修订版本：  

```sh
% git pull --ff-only
```

此命令会更新代码库。在 Git 中，快进合并（fast forward merge）仅需移动分支指针，而不会重新创建提交记录。始终执行快进合并/拉取（pull）可以确保您的代码库与 FreeBSD 官方代码库保持完全一致。如果您需要维护本地补丁，这一点尤为重要。  

有关如何管理本地修改的内容，请参见下文。最简单的方法是在 `git pull` 命令中使用 `--autostash` 选项，但 Git 也提供了更高级的解决方案。  

## 选择特定版本（Selecting a Specific Version）  

在 Git 中，`git checkout` 命令可用于检出分支或特定版本。Git 使用长哈希值来标识版本，而不是顺序编号。  

要检出某个特定版本，只需在命令行中指定相应的哈希值（可以使用 `git log` 命令查找所需的哈希值）：  

```sh
% git checkout 08b8197a74
```

执行后，您会看到类似以下的提示信息：  

```sh
Note: checking out ‘08b8197a742a96964d2924391bf9fdfeb788865d’.
You are in ‘detached HEAD’ state. You can look around, make experimental changes and commit them, and you can discard any commits you make in this state without  
impacting any branches by performing another checkout.  
If you want to create a new branch to retain commits you create, you may do so  
(now or later) by using -b with the checkout command again. Example:  
git checkout -b  
HEAD is now at 08b8197a742a hook gpiokeys.4 to the build
```

最后一行来自于所检出的哈希值及该提交的第一行提交信息。Git 允许使用最短唯一前缀来缩写哈希值，但 Git 本身在显示哈希值时可能不太一致，有时会显示不同的位数。

## 二分查找（Bisecting）  

有时候，代码会出现问题——上一个版本还能正常运行，而刚更新的版本却不行了。开发者可能会要求进行二分查找（bisect），以确定哪个提交导致了回归问题。  

如果你读过上一节，可能会想："这些奇怪的版本号，我该怎么用它们进行二分查找？" 那么这一节正适合你。当然，如果你没想过这个问题，但仍然想学习如何二分查找，这一节同样适合你。  

幸运的是，Git 提供了 `git bisect` 命令来完成这个任务。以下是 `git bisect` 的基本使用步骤。如需更详细的介绍，可以参考以下链接：  

- [A Beginner’s Guide to Git Bisect: The Process of Elimination](https://www.metaltoad.com/blog/beginners-guide-git-bisect-process-elimination)  
- [Git 官方文档：git-bisect](https://git-scm.com/docs/git-bisect)  

Git 的 `man` 手册对 `git bisect` 可能遇到的问题有很好的描述，例如：如何处理无法编译的版本、如何使用不同于 `good` 和 `bad` 的标记等，这些内容本文不会涵盖。  

使用 `git bisect start` 命令可以开始二分查找过程。接下来，你需要指定一个范围：

- `git bisect good XXXXXX` 告诉 Git 某个版本是正常的（即没有问题）。  
- `git bisect bad XXXXX` 告诉 Git 某个版本是有问题的（通常 `bad` 版本是 `HEAD`，即当前检出的最新版本）。  

通常，`bad` 版本几乎总是 `HEAD`，而 `good` 版本是你上次正常运行的版本。  

### 如何查找上次检出的版本  

如果你不记得上次检出的版本，可以使用 `git reflog` 命令来查看：  

```sh
5ef0bd68b515 (HEAD -> master, freebsd/master, freebsd/HEAD) HEAD@{0}:  
pull --ff-only: Fast-forward  
a8163e165c5b (upstream/master) HEAD@{1}: checkout: moving from b6fb97efb682994f59b21fe4efb3fcfc0e5b9eeb to master  
```  

显示我将工作树移动到 master 分支（`a816...`），然后从上游更新到 `5ef0...`。在本例中，`bad` 版本是 `HEAD`（或 `5ef0bd68`），`good` 版本是 `a8163e165`。从输出中可以看出，`HEAD@{1}` 也通常可以使用，但如果在更新后又对 Git 树进行了其他操作，然后才发现需要进行二分查找，这种方法可能并不可靠。  

回到 `git bisect`。先设置 `good` 版本，然后设置 `bad` 版本（顺序无关紧要）。设置 `bad` 版本后，Git 会提供一些关于二分查找过程的统计信息：  

```sh
% git bisect start
% git bisect good a8163e165c5b
% git bisect bad HEAD
Bisecting: 1722 revisions left to test after this (roughly 11 steps)
[c427b3158fd8225f6afc09e7e6f62326f9e4de7e] Fixup r361997 by balancing parens. Duh.
```

然后，你需要构建并安装该版本。如果该版本是好的，输入 `git bisect good`，否则输入 `git bisect bad`。每执行一步，你都会看到类似上面的消息。当找到问题的提交后，将 `bad` 版本报告给开发者（或者自己修复 bug 并提交补丁）。使用 `git bisect reset` 结束二分查找过程，并返回到最初的位置（通常是 `main` 分支的最新提交）。如前所述，[Git bisect 官方手册](https://git-scm.com/docs/git-bisect) 是一个很好的参考，尤其是当遇到问题或特殊情况时。  

## 关于 ports 树  

ports 树的操作方式相同，只是分支名称不同，仓库位置也不同。  

GitHub 镜像地址为：  
[https://github.com/freebsd/freebsd-ports.git](https://github.com/freebsd/freebsd-ports.git)  

cgit 镜像地址目前为：  
[https://cgit-beta.freebsd.org/ports.git](https://cgit-beta.freebsd.org/ports.git)  

正式的 Git 仓库地址将是：  
[https://git.freebsd.org/ports.git](https://git.freebsd.org/ports.git) 或  
`ssh://anonssh@git.freebsd.org/ports.git`（视你选择的传输方式而定）。  

计划在 2021 年第一季度末将 ports 仓库从 Subversion 迁移到 Git。  

和 ports 一样，当前的分支分别是 `master` 和 `main`。每个季度的分支命名方式与 FreeBSD 的 svn 仓库相同。由于转换工具的 bug，在 ports svn 仓库迁移到 Git 时，可能会像 src 和 doc 仓库一样重新计算提交哈希值。  

## 处理本地更改  

本节介绍如何跟踪本地更改。如果你没有本地更改，可以跳过本节（这是最后一节，跳过没关系）。  

一个重要的原则：所有更改在推送之前都是本地的。与 svn 不同，Git 采用分布式模型。对用户而言，大多数操作几乎没有区别。但如果你有本地更改，可以使用相同的工具来管理它们，同时从 FreeBSD 拉取更新。所有未推送的更改都是本地的，并且可以轻松修改（`git rebase` 允许这样做，后文会介绍）。  

### 保持本地更改  

保持本地更改（尤其是一些简单修改）最简单的方法是使用 `git stash`。最基本的用法是：  

- 使用 `git stash` 记录更改（将其推送到 stash 栈）。  
- 许多人会在更新树之前使用 `git stash` 保存更改，然后使用 `git stash apply` 重新应用它们。  
- `git stash list` 可以查看 stash 栈中的更改。  

更详细的信息请参考 [git-stash 官方手册](https://git-scm.com/docs/git-stash)。  

这种方法适用于对树的微小调整。如果你的更改比较复杂，建议使用本地分支并进行 rebase，而不是 stash。此外，Git 还在 `git pull` 命令中集成了 stash 功能，只需在命令行中添加 `--autostash` 选项即可。

### 保持本地分支  

在 Git 中，保持本地分支比在 Subversion 中要容易得多。在 Subversion 中，你需要合并提交并解决冲突。这虽然可行，但可能导致混乱的历史记录，使得上游合并变得困难，或者在需要复现时难以操作。Git 也支持合并（merge），但仍然存在这些问题。合并是一种管理分支的方法，但它是最不灵活的方式。  

除了合并，Git 还支持 **rebase**（变基）概念，可以避免这些问题。`git rebase` 命令会在父分支的更新位置 **重新应用** 该分支的所有提交。本节将介绍使用 rebase 处理最常见的情况。  

### 创建一个分支  

假设你想修改 FreeBSD 的 `ls` 命令，让它永远不要启用颜色显示。有很多理由可以这样做，这里就以这个需求为例。由于 FreeBSD 的 `ls` 命令会不断更新，你需要处理这些变更。幸运的是，使用 `git rebase` 通常可以自动完成。  

```sh
% cd src
% git checkout main
% git checkout -b no-color-ls
% cd bin/ls
% vi ls.c # 进行修改
% git diff # 查看修改内容
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
% # 修改看起来不错，提交更改...
% git commit ls.c
```

提交时，Git 会打开编辑器，让你输入提交说明。填写提交说明后，你就拥有了 Git 仓库中的一个本地分支。按照 FreeBSD 手册中的指引，你可以像平时一样构建和安装它。与其他版本控制系统不同，Git 需要你 **显式** 指定要提交的文件。在上面的示例中，我直接在 `git commit` 命令行中指定了文件，但你也可以使用 `git add`，许多更深入的 Git 教程都会介绍这种方法。

## 该是更新的时候了  

当你需要引入新版本时，几乎与没有分支时相同。你会像之前一样更新，但在更新之前有一个额外的命令，更新之后还有一个命令。以下假设你从未修改过的树开始操作。重要的是在进行变基操作时使用一个干净的树（Git 通常要求这样）。  

```sh
% git checkout main
% git pull --no-ff
% git rebase -i main no-color-ls
```

这将弹出一个编辑器，列出其中的所有提交。在这个例子中，不要做任何更改。这通常是在更新基准时你正在做的事情（尽管你也可以使用 `git rebase` 命令来整理你在分支中的提交）。  

完成上述操作后，你已经将 `ls.c` 的提交从旧版本的 FreeBSD 移动到较新的版本了。  

有时会出现合并冲突。没关系，别慌张。你可以像处理其他合并冲突一样处理它们。为了简便起见，我将只描述一个你可能会遇到的常见问题，完整的处理方法可以在本节末尾找到。  

假设合并包括在一个激进的 `terminfo` 变动中的上游更改，以及选项名称的更改。当你更新时，你可能会看到类似如下内容：  

```sh
Auto-merging bin/ls/ls.c
CONFLICT (content): Merge conflict in bin/ls/ls.c
error: could not apply 646e0f9cda11... no color ls
Resolve all conflicts manually, mark them as resolved with
"git add/rm <conflicted_files>", then run "git rebase --continue".
You can instead skip this commit: run "git rebase --skip".
To abort and get back to the state before "git rebase", run "git rebase --abort".
Could not apply 646e0f9cda11... no color ls
```

看起来很吓人。如果你打开编辑器，你会看到这实际上是一个典型的三方合并冲突解决方法，你可能已经从其他源代码管理系统中熟悉这种方式（`ls.c` 其余部分已省略）：

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

新的代码在前，你的代码在后。正确的修复方法是：在 `#ifdef` 之前添加 `#undef COLORLS_NEW`，然后删除旧的更改：

```c
#undef COLORLS_NEW
#ifdef COLORLS_NEW
#include <terminfo.h>
```

保存文件。由于变基操作被中断，你需要完成它：

```sh
% git add ls.c
% git rebase --continue
```

这告诉 Git `ls.c` 已经被更改，并继续变基操作。由于出现了冲突，你将被踢入编辑器，更新提交信息（如果有必要）。如果提交信息仍然准确，直接退出编辑器。

如果在变基过程中卡住了，别慌张。`git rebase --abort` 会将你带回到干净的状态。不过，重要的是从未修改过的树开始。

更多相关信息，可以参考 [FreeCodeCamp 的 Git Merge 和 Git Rebase 终极指南](https://www.freecodecamp.org/news/the-ultimate-guide-to-gitmerge-and-git-rebase/)，它提供了相当详尽的处理方法，是处理偶尔出现但对本指南来说太晦涩的问题的好资源。

### 切换到另一个 FreeBSD 分支

如果你希望从 `stable/12` 切换到当前分支，并且你有一个深度克隆，那么下面的操作就足够了：

```sh
% git checkout main
% # 在这里构建和安装...
```

但是，如果你有一个本地分支，则有一两点需要注意。首先，变基会重写历史，因此你可能需要采取某些措施来保存它。其次，切换分支时通常会遇到更多的冲突。如果我们假设上面的例子是相对于 `stable/12` 的，那么要切换到 `main`，我建议你执行以下操作：

```sh
% git checkout no-color-ls
% git checkout -b no-color-ls-stable-12 # 为该分支创建一个新的名字
% git rebase -i stable/12 no-color-ls --onto main
```

上面的操作会切换到 `no-color-ls` 分支。然后，为它创建一个新的名字 (`no-color-ls-stable-12`)，以防你需要回到该分支。接着，你将变基到 `main` 分支。这将找到当前 `no-color-ls` 分支上的所有提交（直到与 `stable/12` 分支合并的位置），然后将它们重播到 `main` 分支，创建一个新的 `no-color-ls` 分支（这就是为什么我让你创建一个占位符名字）。

### 从现有的 Git 克隆迁移

如果你基于之前的 Git 转换或本地运行的 git-svn 转换进行工作，迁移到新的仓库可能会遇到问题，因为 Git 对两者之间的连接没有任何认知。

如果你没有很多本地更改，最简单的方法是将你的更改 cherry-pick 到新的基础分支：

```sh
% git checkout main
% git cherry-pick 老分支..你的分支
```

或者，你也可以通过变基来做相同的事情：

```sh
% git rebase --onto main master 你的分支
```

如果你有很多更改，你可能更愿意执行一次合并操作。这个想法是创建一个合并点，将 `老分支` 的历史和新的真理源（`main`）合并在一起。

我们计划发布一组 SHA1 对应的列表，但如果你在本地进行转换，可以通过查找两个父分支上找到的相同提交来找出：

```sh
% git show 老分支
```

你将看到一个提交信息，现在在新分支中搜索该信息：

```sh
% git log --grep="commit message on 老分支" freebsd/main
```

你会在新的 `main` 分支上获得一个 SHA1，接着从这个 SHA1 创建一个辅助分支（在这个例子中我们称其为 `stage`）：

```sh
% git checkout -b stage SHA1_found_from_git_log
```

然后执行合并操作：

```sh
% git merge -s ours -m "Mark old branch as merged" 老分支
```

这样，你就可以在任何顺序下合并你的工作分支或 `main` 分支，且不会出现问题。最终，当你准备将你的工作提交回 `main` 时，你可以执行一次变基到 `main`，或者通过将所有内容合并成一个提交来进行压缩提交。

---

**WARNER LOSH** 是奈飞的高级软件工程师，已经成为 FreeBSD 的贡献者 20 余年。Warner 改进了 FreeBSD 中的多个系统——例如引导加载程序。在加入奈飞之前，他曾制作闪存驱动器并测量原子钟的精度。他的代码仍然在测量一些用于生成 UTC 的时钟！
