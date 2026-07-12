# Ports 报道

- 原文：[Ports Report](https://freebsdfoundation.org/our-work/journal/browser-based-edition/migrating-jail-management-from-warden-to-iocage/)
- 作者：**Frederic Culot**

9 月至 10 月期间，Ports 领域的活跃度与夏季基本持平，整体并不算高。话虽如此，我们很高兴看到几位新 committer 加入项目。此外，本期内一些主要的 Port 也得到更新，升级时可能需要按后文所述谨慎处理。

## 新任 Ports Committer 与代为保管

我们很高兴欢迎三位新老面孔加入 Ports committer 的行列。

- Soren Straarup（xride@）在时隔一年多之后，得以重新参与 Ports 工作。garga@ 与 mat@ 将担任其导师，在 xride@ 需要追上近期基础设施变化时提供帮助。

另有两位新 committer 加入我们：

- Kenji Takefu，由 hrs@ 与 mat@ 担任导师；
- Carlos Puga Medina，由 junovitch@、amdmi3@、feld@ 担任导师。

两人都已为 FreeBSD 贡献不少：Kenji 自 2006 年以来提交了 300 多份问题报告，Carlos 维护着 30 多个 Port。两份实至名归的 commit 权限——恭喜二位！

此外，部分 commit 权限因超过 18 个月未活跃而被代为保管（fluffy@、lioux@、lippe@、simon@）。

## 数据统计

来看几项数据：2015 年 9 月至 10 月这两个月期间，Ports 树共合入 4,666 次 commit，较上一周期增长超过 10%。问题报告方面，共关闭 1,073 份 PR，相较 7–8 月亦有小幅增长。衷心感谢大家的贡献！

## 重要 Ports 更新

一如既往，antoine@ 进行了大量 exp-run（共 22 次），以检查主要 Port 的更新是否安全。这些重要更新中，以下亮点值得一提：

- CMake 更新至 3.3.1
- ffmpeg 更新至 2.8
- Qt4 更新至 4.8.7

另一项值得关注的更新涉及 Firefox（41.0）和 SeaMonkey（2.38）：它们现在要求 **databases/sqlite3** 依赖在构建时启用 DBSTAT 选项——前提是你未使用二进制包，而是采用一组非默认的 Port 选项。照例，更多信息见 **/usr/ports/UPDATING** 文件，升级主要 Port 前应仔细阅读。

## 我能如何参与？

如果你喜爱 FreeBSD 并希望加入团队，不错的切入点是接手一个无人维护的 Port 并提交更新。要查找未指定具体 committer 或团队的 Port，可在线浏览 portsmon 的列表（<http://portsmon.freebsd.org/portsconcordanceformaintainer.py?maintainer=ports%40FreeBSD.org>），或执行以下命令：

```sh
make -C /usr/ports quicksearch maint=ports@FreeBSD.org
```

选定后，应阅读《Porter 手册》（<https://www.freebsd.org/doc/en/books/porters-handbook/>）巩固你的移植技能。当更新看上去正确时，通过 FreeBSD 在线问题报告界面提交（<https://bugs.freebsd.org/bugzilla/>）。如在此过程中需要帮助，我们很乐意通过论坛（<https://forums.freebsd.org/>）或 IRC（如 EFnet 上的 `#bsdports` 频道）为你提供帮助。祝你好运，期待你早日加入我们的行列！

---

**Frederic Culot** 在完成物理学计算机建模博士学位后，已在 IT 行业工作 10 年。业余时间他研习商业与管理，并刚取得 MBA 学位。Frederic 于 2010 年以 Ports committer 身份加入 FreeBSD，此后累计提交约 2,000 次 commit，指导过六位新 committer，现承担 portmgr-secretary 的职责。
