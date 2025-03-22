# Capsicum 案例研究：Got

- 原文链接：[Capsicum Case Study: Got](https://freebsdfoundation.org/wp-content/uploads/2021/07/Capsicum-Case-Study-Got.pdf)
- 作者：**YANG ZHONG**

我已经在 FreeBSD 基金会的实习期间与 Capsicum 一起工作了一段时间。本文详细介绍了我将 Capsicum 沙箱框架应用于一个名为 Got 的大型程序的过程。在此过程中，我将简单而具体地介绍 Capsicum：它解决了什么问题、其解决方案背后的思路以及如何使用它。我们将发现 Got 非常适合 Capsicum，并且我将讨论 Got 的结构如何使该程序更适合 Capsicum。

## Capsicum 概念

Capsicum 的目的是解决计算机程序的一个简单问题：它们的权限太大。我喜欢这样理解它。在没有 Capsicum 的世界中，如果我登录到我的计算机并运行某个程序，程序就没有任何东西可以阻止它在没有警告的情况下删除我的整个主目录。当然，程序可能永远不会故意这么做，但它有足够的权限来做到这一点。当我们考虑到安全性时，这就成为一个真正的问题：如果有人发现了程序中的漏洞，他们可以利用它来做程序能够做的任何事情。因此，如果程序拥有过多的权限，攻击者可以利用它做出过多的破坏。

这时，Capsicum 应运而生，它提供了控制程序权限的工具。Capsicum 处理的一个重要概念是全局命名空间。实际上，全局命名空间是一个对象的集合（“空间”），每个对象都有一个标识符（“名称”），该标识符在所有对象中是唯一的（“全局”）。一个简单的全局命名空间例子是文件系统：空间是所有文件的集合，每个文件的绝对路径是其唯一的“名称”。FreeBSD 操作系统有许多全局命名空间，但文件系统是无处不在且非常重要的；从现在开始，我将多次提到它。

没有 Capsicum 的程序通常需要处理全局命名空间。当这些程序想要打开一个文件时，它们可以传入文件的绝对路径。虽然这看起来很正常，但程序实际上在这里使用了它巨大的权限：“在这台计算机上的所有文件中，我就要这个！”虽然文件权限等机制会阻止程序访问每一个文件，但这种权限的大小显然属于“删除你整个主目录”的范围。

更糟糕的是，当程序行使它的权限时，它是在隐性地行使这些权限。它并没有说：“我想行使我的权限来访问每个文件，这里有一个‘密钥’来验证我拥有这个权限”；程序只是总是拥有这个权限。这种“默认情况下”能够做事的权限称为环境权限（ambient authority）。在访问这些全局命名空间时，程序总是在行使它的环境权限。

因此，Capsicum 将全局命名空间视为一种基本上无法控制的权力来源，因此引入了能力模式（capability mode）——在这种模式下，程序根本无法使用全局命名空间，因此也没有环境权限。程序通过调用 `cap_enter()` 进入能力模式，并且永远无法离开。在能力模式下，程序不能再打开新文件，因此只能使用它在调用 `cap_enter()` 之前打开的文件描述符，或者通过与其他进程的连接提供的文件描述符。这个限制程序访问资源的环境被称为沙箱（sandbox），这就是为什么 Capsicum 被描述为一种沙箱技术。

Capsicum 不仅仅做这些。另一个重要概念是能力（capabilities），在 Capsicum 中，能力是扩展文件描述符的对象；它们让你可以精细地限制任何文件描述符能够执行的操作，并且在更隐性地角色上，使能力模式得以正常工作。还有一个名为 Casper 的库，它为 Capsicum 化的程序提供常见服务。

## 目标：Got

因此，利用这些工具，我们希望将一些现有的程序调整以使用它们。我负责将版本控制系统 Game of Trees（简称 Got）适配到 Capsicum。这个项目由 OpenBSD 开发人员开发并为其使用，但 FreeBSD 项目正在考虑将 Got 添加到基础系统中。因此，作为这项工作的组成部分，我的任务是弄清楚如何调整 Got，使其更适应 Capsicum，而不做剧烈的改变：理想情况下，我们希望做出足够干净的结构性更改，以便将其并入 Got 的上游版本，使 FreeBSD 上的 Capsicum 版本 Got 尽可能与原版本相似。

## Capsicum 化 Got

从高层次来看，Capsicum 化的程序通常遵循一个共同的模式：程序被分为两个部分。在第一部分，程序获取所需的资源；在第二部分，程序从这些资源中读取和写入数据。程序在第一部分之后进入能力模式。由于程序在此之后被限制在能力模式下，它的权限在危险的第二部分中被限制，而这部分工作涉及到与它在第一部分中获取的外部和不可信资源的交互。

许多程序并不是这样分开的。通常，它们会在方便的地方获取资源，导致这两个部分精细地交织在一起。在 Got 之前，我在程序 `sort()` 中处理了这个问题。在这些情况下，像 Casper 这样的辅助库非常宝贵，因为它们旨在解决常见的 Capsicum 化问题，否则这些问题需要大量的设置工作才能解决。你可以在 YouTube 上观看 Mariusz Zaborski 的《Case studies of sandboxing base system with Capsicum》视频（EuroBSDcon 2017），其中部分内容描述了 Capsicum 化程序 `bspatch()` 的过程。

幸运的是，Got 在获取文件的方式上有结构性。Got 使用两个主要的目录：一个是仓库（repository），另一个是工作树（worktree）。如果你熟悉 Git，这些与 Git 仓库和工作树非常相似。Got 接着有 `got_repository_open()` 和 `got_worktree_open()` 函数，负责查找仓库/工作树并返回一个结构体——分别是 `struct got_repo` 和 `struct got_worktree`，其中包含这两个目录的信息。

在这一点之后，Got 完全在这两个目录（和 /tmp）中工作，这意味着它不会尝试获取任何“新的”东西。这避免了之前讨论的交错问题，但 Got 仍然使用全局文件系统命名空间来实际打开新文件——例如，`got_repo` 结构体包含其关联仓库的绝对路径，因此每当需要时，Got 就会使用该路径打开目录。这与能力模式不兼容。

在这种情况下，我必须预先打开这两个目录中的每个文件，以便在能力模式下使用它们吗？幸运的是，不必。打开文件时，你会获得它的文件描述符。对于非目录文件，它的描述符仅让你访问该文件。然而，目录的文件描述符让你访问该目录中的所有内容。

为此，FreeBSD 通过 POSIX 提供了 `*at()` 系列系统调用。正常的调用使用绝对路径，而 `*at()` 调用使用文件描述符和相对路径。如果我希望打开文件“/dir/subdir/a”，并且我有一个指向 `dir` 的文件描述符 `fd`，我可以调用 `openat(fd, "subdir/a")`。这种访问方式在能力模式下是允许的，除了一些例外情况，因为我们不再通过全局命名空间搜索所有文件。

可以很容易地看出这如何帮助 Got，因为我们知道 Got 总是会在两个特定目录中工作！如果我们预先打开仓库和工作树目录，并将它们的文件描述符存储在 `got_repo` 和 `got_worktree` 结构体中，我们就可以稍后使用这些描述符打开这些目录中的文件，即使在能力模式下也是如此。在 Got 中，操作仓库或工作树内部文件的函数将以 `got_repository` 或 `got_worktree` 作为参数，这意味着我们添加的文件描述符将很容易在这里访问。

```c
static const struct got_error
update_blob( struct got_worktree *worktree,
	     struct got_fileindex *fileindex, struct got_fileindex_entry *ie,
	     struct got_tree_entry *te, const char *path,
	     struct got_repository *repo, got_worktree_checkout_cb progress_cb,
	     void *progress_arg )
{
	const struct got_error	*err	= NULL;
	struct got_blob_object	*blob	= NULL;
	char			*ondisk_path;
	unsigned char		status = GOT_STATUS_NO_CHANGE;

	truct stat sb;
	if ( asprintf( &ondisk_path, “ % s / % s ”, worktree->root_path, path ) == -1 )
		return(got_error_from_errno( “ asprintf ” ) );

	/* 使用示例 */
	int opened_file_fd = open( ondisk_path, 0 );
```

上述 Got 代码片段展示了一个函数，它接收一个 `got_worktree` 结构体，并利用它构造出该目录中文件的路径。我已经添加了一个示例，说明该函数如何通常使用新的路径。

下面是相同的代码，已修改为使用我们新的文件描述符策略：

```c
static const struct got_error
update_blob(struct got_worktree *worktree,
 struct got_fileindex *fileindex, struct got_fileindex_entry *ie,
 struct got_tree_entry *te, const char *path,
 struct got_repository *repo, got_worktree_checkout_cb progress_cb,
void *progress_arg)
{
 const struct got_error *err = NULL;
 struct got_blob_object *blob = NULL;
 char *ondisk_path;
 unsigned char status = GOT_STATUS_NO_CHANGE;

 struct stat sb;
 int path_fd_part = worktree->root_fd;
 char *path_relative_part = path;

 // 使用示例
 int opened_file_fd = openat(path_fd_part, path_relative_part, 0)
```

这非常简单！比第一个例子还要简单，因为不再需要 `asprintf()` 调用了。在这种情况下，适配 Got 以支持 Capsicum 很容易。

有些函数并不接收这些结构体，而是接收它们操作的绝对路径。将这些函数适配为兼容 Capsicum 需要更多的工作，因为我们必须将它们的参数从绝对路径更改为一对（相对路径，目录文件描述符），以便函数能够在能力模式下访问文件。

实际上，这通常只是一个小问题。函数接收的绝对路径并非凭空而来——它必须是通过使用 `got_repo` 或 `got_worktree` 结构体创建的，因此我们需要的文件描述符不会远离。下面是一个需要更改参数的函数，作为 Capsicum 化的一部分：

```c
const struct got_error *
got_fileindex_entry_update(struct got_fileindex_entry *ie,
- const char *ondisk_path, uint8_t *blob_sha1, uint8_t *commit_sha1,
- int update_timestamps)
+ int wt_fd, const char *ondisk_path, uint8_t *blob_sha1,
+ uint8_t *commit_sha1, int update_timestamps)
{
 struct stat sb;
- if (lstat(ondisk_path, &sb) != 0) {
+ if (fstatat(wt_fd, ondisk_path, &sb, AT_SYMLINK_NOFOLLOW) != 0) {
 if (!((ie->flags & GOT_FILEIDX_F_NO_FILE_ON_DISK) &&
 errno == ENOENT))
- return got_error_from_errno2(“lstat”, ondisk_path);
+ return got_error_from_errno2(“fstatat”, ondisk_path);
 sb.st_mode = GOT_DEFAULT_FILE_MODE;
 } else {
...
```

由于函数的参数发生了变化，我们还需要修改所有调用它的地方。在一些地方，调用函数创建了路径，并通过 `got_worktree` 结构体将其传递给 `got_fileindex_entry_update()`；对于这些地方，我们已经拥有所需的文件描述符，因此适应新的参数非常简单：

```c
...
 * 保留工作文件，并将已删除的 blob 条目
 * 改为调度添加条目。
 */
- err = got_fileindex_entry_update(ie, ondisk_path, NULL, NULL,
- 0);
+ err = got_fileindex_entry_update(ie, worktree->root_fd,
+ ie->path, NULL, NULL, 0);
 } else {
... 
```

一些调用函数也将路径作为参数传入，简单地将其传递给 `got_fileindex_entry_update()`。对于这些函数，我们必须类似地更改调用函数的参数：

```c
static const struct got_error *
-sync_timestamps(char *ondisk_path, unsigned char status,
+sync_timestamps(int wt_fd, const char *path, unsigned char status,
 struct got_fileindex_entry *ie, struct stat *sb)
 {
 if (status == GOT_STATUS_NO_CHANGE && stat_info_differs(ie, sb))
- return got_fileindex_entry_update(ie, ondisk_path,
+ return got_fileindex_entry_update(ie, wt_fd, path,
 ie->blob_sha1, ie->commit_sha1, 1);
... 
```

最终，路径必须来自能够访问 `got_worktree` 的函数，因此文件描述符可以始终贯穿这些调用链。尽管这并不是一个完美的解决方案，特别是当调用链过长时，但到目前为止我还没有遇到超过两个调用的链条。

## 总结

希望到目前为止，你已经相信让 Got 与能力模式一起工作是简单的。虽然我只为 Got 提交了需要做的工作的一小部分，但我怀疑大多数必要的修改都会类似于你所看到的。

当然，并不是所有程序都能如此顺利地适配 Capsicum。从根本上讲，能力模式适用于那些有意识地管理其资源的程序。Got 将其主要资源——工作树和仓库目录——转换为代码中的结构体。如果一个函数要对这些资源进行操作，它需要相应的结构体。

通过这种方式，代码明确地表示：“这个函数将需要这个资源”。此外，由于 Got 只处理很少的其他资源，代码也在表示“这个函数只需要这个资源”。这种显式性与环境权限（ambient authority）正好相反，正是能力模式所希望的！剩下的工作就是强制执行这些限制。

1. ...与其他具有相似目标但设计不同的框架一起，例如 Linux 的 seccomp 和 OpenBSD 的 pledge/unveil。已经有很多关于这些框架的比较文章；Jonathan Anderson 的《A comparison of Unix sandboxing techniques》详细分析了这些框架。
2. 你可以在 Robert N.M. Watson 等人所写的《Capsicum: practical capabilities for UNIX》中找到全面的列表。
3. Mark S. Miller 等人在《Capability Myths Demolished》中清晰地描述了环境权限（ambient authority）。
4. 实际上，你会使用 1Capsicum helpers` 并调用 caph_enter()，但本质上它们是一样的。
5. 事实上，Got 可以与普通的 Git 仓库一起使用，因此名称相似。
6. 这里我犯的一个大错误是，在调用 got_worktree_open 和 got_repo_open 函数之前尝试进入能力模式 — 经过大量的黑客操作后它是能工作的，但造成了很大的混乱，后来 Got 的首席开发人员善意地告诉我，这些函数本来也没有做什么危险的操作，因此可以在进入能力模式之前调用它们；从中我意识到，在尝试应用 Capsicum 之前，理解代码非常重要。这听起来显而易见，但我却是通过痛苦的方式学到的。
7. 路径不能是绝对路径，路径不能使用“..”组件来“逃逸”出目录，并且文件描述符不能是 AT_FDCWD。

---

**YANG ZHONG** 正在滑铁卢大学学习计算机科学。他曾在 2020 年秋季学期作为实习生在 FreeBSD 基金会工作，并在 2021 年春季学期继续实习。在空闲时间，他喜欢为滑铁卢大学数学系的学生出版物 mathNEWS 写作。
