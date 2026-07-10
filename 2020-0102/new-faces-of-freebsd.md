# FreeBSD 新面孔

作者：DRU LAVIGNE

本专栏聚焦近期获得 commit 权限的贡献者，并向 FreeBSD 社区介绍他们。

本期聚光灯照向 Loïc Bartoletti，他于 1 月获得 Ports commit 权限。

## 请介绍一下你自己、你的背景与兴趣。

- **Loïc**：我的背景是历史与城市规划，职业经历从分析工具的用户演变为工具的创建者，尤其是 graphics/qgis、databases/grass 与 databases/postgis 等 Ports。我现为 GIS/CAD 工程师，C/C++/Python/SQL 开发者，正在学习 Nim/Rust 与 Qt UI。我的专业领域是使用 PostgreSQL 的 GIS、CAD、土地测量、GNSS（GPS）数据库。

我个人与职业上都投身开源地理（OSGEO）运动，并确保所开发的工具能在 FreeBSD 上使用。这涉及修正代码以合入上游 BSD、为我们的系统创建 Ports，并推广它们。

我维护的 Ports 列表反映了我的专业与个人活动。

## 你最初是如何了解到 FreeBSD 的？FreeBSD 哪一点吸引了你？

- **Loïc**：我的首次体验是在 2004/2005 年测试一套 "UNIX" 系统。我先听说 Linux，但它在我们的硬件上要么不工作，要么工作得很糟。我从苹果阵营而来，自然也听说过 FreeBSD。偶然间我找到一本法语杂志，附带一张 CD 与非常详尽的安装文档。当时我非常喜欢它：易用、稳定、base 与 packages 分离。

我出于知识好奇持续关注 BSD，直到 FreeBSD 6（约 2007/2008 年）成为我在家的常规系统。自 2018 年起，通过在 Windows 工作站上的虚拟机中逐步集成 FreeBSD，我在新工作中每日使用 FreeBSD。

## 你是如何成为 committer 的？

- **Loïc**：几个月间，我在项目中的参与逐步加深，无论是贡献还是与其他贡献者（尤其是 #kde-freebsd）的关系。最终，Tobias 推荐我加入 Ports 团队。

## 加入 FreeBSD 项目后体验如何？对希望成为 FreeBSD committer 的读者有何建议？

- **Loïc**：才刚开始，目前还说不上有足够的回顾空间，但当我向其他开发者自我介绍时，收到的消息数量让我惊叹。

我继续学习，尝试打磨我的 Ports，同时也在两位导师的支持下帮助其他维护者。

我没什么特别建议，但我觉得与他人沟通、加入你感兴趣的团队至关重要。

DRU LAVIGNE 是 FreeBSD doc committer，著有《BSD Hacks》与《The Best of FreeBSD Basics》。

---

Jail 是 FreeBSD 最具传奇色彩的特性：

以强大著称，掌握起来颇为棘手，

并被数十年的 dubious 传说所笼罩。

- 在 Jail 的限制内游刃有余地工作
- 对 Jail 特性实施精细控制
- 构建虚拟网络
- 部署分层 Jail
- 约束 Jail 资源使用
- ……以及更多！

《FreeBSD Mastery: Jails》作者：MICHAEL W LUCAS 各大书店有售

《FreeBSD Mastery: Jails》拨开迷雾，

揭示 Jail 的内部机制，将其力量为你所用。

## 禁锢你的软件！

- 理解 Jail 如何实现轻量级虚拟化
- 理解 base 系统的

```sh
jail 工具与 iocage 工具包
```

- 优化硬件配置
- 从 host 与 Jail 内部管理 Jail
- 优化磁盘空间以支持数千个 Jail
