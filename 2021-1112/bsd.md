# 进行中的工作/征求反馈：OccamBSD

- 原文链接：[进行中的工作/征求反馈：OccamBSD](https://freebsdfoundation.org/wp-content/uploads/2022/01/WIP.CFT-OccamBSD.pdf)
- 作者：**TOM JONES & MICHAEL DEXTER**

---

进行中的工作/征求反馈是由 Tom Jones 主持的新栏目，将包含一些有趣的、长期进行的项目和工作进展，你可能会对此感兴趣并希望参与贡献。本期的内容是 Tom 与 OccamBSD 作者 Michael Dexter 的对话。

---

## 什么是 OccamBSD？

FreeBSD 可以通过多种方式编译——FreeBSD 操作系统有许多可以有条件构建的组件。可选构建的组件非常强大，它们帮助保持操作系统的模块化，并且使得移除不需要的功能变得容易，无论这些功能是否用于嵌入式系统。

OccamBSD 是一款用于构建小型嵌入式 FreeBSD 镜像的工具。与其复制单独的工具来制作自定义镜像或依赖外部专业的构建工具，OccamBSD 是一个 shell 脚本，它使用 FreeBSD 的构建基础设施来创建最小化的镜像，主要面向三个引导目标——jail，和 bhyve 以及 Xen 虚拟机监控程序。

生成的最小化系统包含大约 400 个文件，分布在三十多个目录中，它并不是无法识别的，而是提供了一个关于现代 BSD 出现之前 4.4BSD-Lite2 的一瞥。

通过 OccamBSD，我们有了一个独特的机会来观察大多数构建选项的实际应用，并探索一个没有“buildworld”的“世界”是什么样子，提供了一个最小的用户空间，允许在 bhyve 虚拟机上成功登录。

除了 VirtIO 驱动程序外，在 bhyve 下启动所需的最小文件大致代表了所有 FreeBSD 用户始终使用的代码。这种狭窄的范围恰恰是所有审计、文档编写和计算机科学教育工作应该从哪里开始的地方。否则，FreeBSD 对于新学生或新用户来说可能会显得过于庞大。

## 为什么它很有趣？

如果你使用 FreeBSD，这突出了你使用的代码，并且是一次很有价值的练习。

对于 FreeBSD 用户来说，OccamBSD 给你提供了一个极简的系统实例，并且让你有机会思考一个更小的系统是否适合你。OccamBSD 创建了一个更小的学习环境，相比完整的 FreeBSD 环境，这个环境更容易阅读和推理，对于课程或学术工作来说，这可能是一个很好的起点。

## 我如何贡献？

OccamBSD 在 GitHub 上开发。欢迎贡献，你可以通过测试工具、编写文档或提交补丁来参与。

OccamBSD 很高兴在 GitHub 上接受新的问题，接受通过拉取请求提交的 bug 修复，并欢迎在任何可以联系到开发者的地方报告成功情况。

[https://github.com/michaeldexter/occambsd](https://github.com/michaeldexter/occambsd)

[https://github.com/michaeldexter/occambsd/issues](https://github.com/michaeldexter/occambsd/issues)


---

**TOM JONES** 希望 FreeBSD 基础的项目能得到应有的关注。他住在苏格兰东北部，并提供 FreeBSD 咨询服务。

**MICHAEL DEXTER** 是一位位于俄勒冈州波特兰的 OpenZFS 支持提供者，喜欢讨论 bhyve 虚拟机监控程序和 OpenZFS。
