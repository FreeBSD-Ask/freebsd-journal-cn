# FreeBSD 的新面孔

- 原文链接：[New Faces of FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/)
- 作者：**DRU LAVIGNE**

本栏目旨在为近期获得提交权限的贡献者提供聚焦，并向 FreeBSD 社区介绍他们。本期聚焦的是 Thomas Munro，他于 10 月获得了 src 提交权限。

**请介绍一下你自己、背景以及兴趣。**

我是 PostgreSQL 项目的黑客与提交者，受雇于 EnterpriseDB。我在新西兰惠灵顿的基地做开源工作大约四年。此前我在多种 Unix 上做了十年二十年的 Java、Lisp、C 与 C++ 专有工作，主要在欧洲，主要在金融/交易技术领域。我有兴趣寻找让 PostgreSQL 与 FreeBSD 在性能、正确性、安全性、可观测性方面更好地协作的方式。

**你是怎么了解到 FreeBSD 的？又是什么让你对 FreeBSD 产生兴趣？**

多年前我在各种场合遇到过 FreeBSD，但直到三年前才把它装到自己的硬件上——因为我想要 ZFS 做家用存储盒子。我发现自己越来越多地登录到那台机器上做各种各样的工作，因为在那里做事更有趣。不久，我把 FreeBSD 用到了几个需要 Web 服务器与数据库服务器的项目上。我也对 kqueue 与 dtrace 充满好奇。FreeBSD 易用、文档完善、积极开发、易于贡献，并且有一本非常适合好奇的内核新手黑客的好书（《FreeBSD 操作系统设计与实现》）。它具备临界规模和充满活力的社区，同时又有足够的低垂果实让新人做出有意义的贡献。再加上 BSD 的悠久历史和它对我们行业技术与文化的巨大贡献。最后，我不得不承认，第一次安装它时感觉像是一种小小的叛逆行为。

**你是如何成为提交者的？**

当我把 FreeBSD 切换为日常 PostgreSQL 工作的主要开发环境后，就希望修改它，提供缺失的能力。我成功合入了四套补丁：`PROC_PDEATHSIG_CTL`、`setproctitle_fast()`、为 truss 提供的 `shm_open()`/`shm_unlink()` 支持、为 `pam_exec.so` 提供的 `expose_authtok`。这些起点谦卑，且分布在相当不同的领域，但都解决了我当时工作的痛点，帮我熟悉了流程，也让我与几位提交者取得了联系。我还有一张待办补丁清单，正在与一位未来的导师在 Freenode IRC 上讨论其中一些。他推荐我作为潜在的提交者。我接受了。

**加入 FreeBSD 项目以来的体验如何？对想要成为 FreeBSD 提交者的读者有什么建议？**

还为时尚早，由于其他事务我的参与会比较零散，但我很高兴合入了一份通过 bugzilla 提交的补丁，修复了 **pom(6)** 中一个 30 年的老 bug。我也很期待首次参加 FOSDEM 的 BSD devroom。

至于成为提交者，我只能说对我有效的方式：在你的工作负载或兴趣领域找到 FreeBSD 的薄弱之处。动手。提交补丁。回应反馈，并留意项目做事的方式。让别人知晓你的计划与兴趣，并与人交流。

---

**DRU LAVIGNE** 是 FreeBSD 文档提交者，《BSD Hacks》和《The Best of FreeBSD Basics》的作者。
