# 进行中的工作/征求反馈：mkjail

- 原文：[WPT/CFT: mkjail](https://freebsdfoundation.org/wp-content/uploads/2022/03/wipcft-mkjail.pdf)
- 作者：**TOM JONES**

Jail 是 FreeBSD 最著名的功能之一。它们使单一服务器能够提供多个服务，同时保持服务器运行的应用程序之间的明确分离。虽然 Jail 是容器化技术最早的应用之一，但 FreeBSD 并没有获得大多数心智份额，相反，像 docker 这样的工具获得了广泛的使用。

部分原因可能是，正常的 FreeBSD 安装中包含的高质量工具，已经足够用来管理单个主机上的一系列服务。随着时间的推移，工具变得更加友好，而在 FreeBSD 中，缺少单一、简单的管理和自动化框架来处理 Jail 管理任务。这可以从许多旨在为 FreeBSD 基础工具添加额外层次来管理 Jail 及其生命周期的项目中看出。

mkjail 不是这些 Jail 管理层，实际上，它是用于创建、更新和升级 Jail 的简单接口。mkjail 只做这三件事。Jail 生命周期的其他管理工作由 FreeBSD 的正常 Jail 基础设施处理。

mkjail 避免了许多其他功能更全面的 Jail 框架所带来的复杂性。其他框架允许复杂的配置层次和覆盖层，使用基础 Jail 上的精简 Jail 来稍微减少占用空间。相反，mkjail 是简单的工具，用于创建 Jail——它把 FreeBSD 系统装进“瓶子”里。它源代码的第一行很好地描述了它：“懒惰、肮脏的工具，用于创建胖 Jail。”

mkjail 通过减少其工作场景而变得简单。它仅提供创建所谓的“胖”Jail，而每个胖 Jail 都是独立的 Jail。

mkjail 使用小的配置文件来描述将要创建的 Jail 的 zfs 数据集位置、它们在文件系统中的位置，以及应该安装到 Jail 中的安装集。若使用 mkjail 创建了 Jail，它必须以与使用基本系统工具管理的其他 Jail 相同的方式集成到 **/etc/jails.conf** 中。

使用基本系统工具创建 Jail 很简单——bsdinstall 有 jail 命令，可以将 Jail 安装到任何指定的目录中。mkjail 扩展了这个功能，并处理与 zfs 工具的集成，以及执行更新和升级的简洁接口。

mkjail 在更新和升级 Jail 时表现出色，将这些操作转化为单个命令。因为 mkjail 只运行胖 Jail，所以在执行更新和升级时，不会出现与精简 Jail 相关的管理问题。

mkjail[1] 最初由 Mark Felder 开发，由 BSDCan 的 Dan Lagille 上传到 GitHub，并得到了 Andrew Fyfe 的贡献。该项目在 GitHub 上进行开发（<https://github.com/mkjail/mkjail>），并且仍然非常年轻——源代码树导入到 GitHub 可以追溯到 2021 年中。

mkjail 旨在成为小工具，用于增强和简化 Jail 的创建、更新和升级。它是通过几个小型 shell 脚本编写的，足够小，可以集成到 FreeBSD 的基本系统中。

mkjail 很高兴通过其 GitHub 接受贡献，形式可以是代码或文档的 pull 请求，也可以在 GitHub 上讨论错误和新功能的 issues。mkjail 完全用 sh 编写，这使得贡献代码相对简单。

由于 mkjail 是年轻的项目，它可以从测试中受益，既包括其已实现的子命令测试，也包括通过在实际工作流程中使用它获得的反馈。贡献 mkjail 的最佳方式是尝试使用它，并反馈你在文档、代码或任何痛点方面遇到的问题，或者在小的改进能使你的工作流程更顺畅的地方提供建议。

---

**TOM JONES** 希望基于 FreeBSD 的项目能够获得应有的关注。他住在苏格兰东北部，并提供 FreeBSD 咨询服务。
