# 新的 Port 提交者：oel Bodenmann (jbo@freebsd.org)

采访者：TOM JONES

![](https://freebsdfoundation.org/wp-content/uploads/2024/01/Bodenmann.jpg)

TJ: 嗨 Joel，欢迎加入项目。你能给我介绍一下你自己以及你喜欢从事的技术项目吗？

JBO: 我是一名电子工程师，主要专注于嵌入式系统。通常我喜欢处理被设计用来执行特定任务而非通用计算的系统。

TJ: FreeBSD 并不以在嵌入式系统开发中出名，你尝试过在 FreeBSD 上进行工作或者项目吗？

JBO：我认为我必须不同意你的说法。我之所以选择 FreeBSD，正是为了将其用作嵌入式系统开发的平台。虽然“嵌入式”世界在过去几年发生了巨大变化，但我通常处理的嵌入式系统是资源相对较低、实时性强的系统（即典型的基于微控制器的系统，CPU 速度‹ 120MHz，RAM‹ 128kB）。因此，这些系统并非设计用于直接运行 FreeBSD，但也不会以任何实际意义运行 Linux（也不会符合要求）。

你仍然需要一个（桌面）宿主系统进行实际开发，以及周围的支持基础设施。我通常参与的这些嵌入式项目从第一次会议到部署产品可能需要几年的时间。我对在 Linux 作为开发平台的一个不满是，Linux 生态系统处于不断变化的状态。你在基于 Linux 的系统上设置开发工作流程和环境，仅仅几个月后就被迫升级，这需要重新验证整个工作流程。

FreeBSD 以其稳定性和连贯性而闻名。与其因为以前的系统或组件不再“够好”而需从头实现一个新系统或组件不如，FreeBSD 试图设计那些系统和工具以便可维护和可扩展，大大减少开发系统管理开销。

对于一些项目，我确实需要一个非微控制器系统，微控制器系统与之通信（即长期数据记录，编排等）。在这种情况下，FreeBSD 具有相同的优势：您设置和验证系统一次，然后在接下来的几年里只需执行较小的维护任务，而对于基于 Linux 的系统，我从来不知道第二天会遇到什么情况。

粗略总结基本上是，我喜欢把时间花在实际开发/工程上，而不是因为底层的 init 系统、音频子系统、hypervisor 或最流行的容器系统或类似系统在两年内更改三次而不断更新、调试和重新验证我的开发平台。FreeBSD 在这里符合所有要求。自从我将所有服务器、网络基础设施和工作站迁移到 FreeBSD 后，我从来没有猜测过我的开发基础设施是否在第二天进入办公室时仍然可以正常启动和工作。它非常可靠。

TJ：您是否已经在ports树中承担了特定的关注领域？

JBO：到目前为止，我主要致力于处理“易致力于的果实”，这样做的想法是让我熟悉ports系统的基础知识以及工作流程和基础设施。与此同时，这也释放了更有经验的ports提交者的资源，让他们可以将精力专注于更复杂的事务。

随着我变得更加熟悉，我希望在接下来的一年里承担更多涉及的任务，比如更新拥有许多使用者的ports库。

因此，我对你的问题的回答是：不，我尽我所能地帮助。我毫不怀疑，随着经验的增加，我很快就会找到一些更大的工作要做：🙂:

TJ: 这个ports集合非常庞大，许多小工具可以完成繁重的工作。到目前为止，你做过的一些低 hanging ports 是什么？你有什么建议可以帮助其他人找到低 hanging fruit 和更容易的第一个ports 吗？

JBO: 就我个人而言，迄今为止，我认为有两种低 hanging fruit 情境：

1. PRs，这些补丁已经获得维护者的批准，并将补丁更新到新的上游次要版本的port。在这里，我建议刚开始时避开那些涉及大量消费ports 的“基础”ports，以减少触发大规模故障的风险。我的理由是简单的上游次要版本升级几乎不会出错，并且维护者倾向于小心地避免破坏他们的ports，并且他们已经具有经验，知道要特别注意他们的上游相关事项。
2. 引入新的port的 PR。这些 PR 往往是“低优先级”，因此压力相对较小，不必急于将它们落地。此外，我认为这是了解ports框架提供的各种不同系统和机制的绝佳途径，这些我可能之前尚未接触过。

截至目前，我所做的工作：我有意尝试触及各个领域的各种ports。我不专注于特定类别或类型的port。相反，我尝试处理一些我知道依赖于我尚未使用过的东西的ports的 PR。这是熟悉各种 Mk/Uses/*脚本的绝佳方式。

TJ：当您展望 FreeBSD 的未来时，从维护者的角度来看，您认为项目的主要优先事项应该是什么？

JBO：我认为使 FreeBSD 对各种用户如此具有吸引力的重要原因之一是我们通常遵循更老式的原则。FreeBSD 项目倾向于采取“稳扎稳打”的方式，而不是不断跳上最新的炒作列车不断重新发明东西。当然，这种方法也有缺点。例如，ports框架缺少一些可能被现代标准视为“基本”的功能，例如子包，建议可选安装等。但我认为，正是因为这些事情不被匆忙对待，我们最终得到了一个更稳定和更易于维护的系统。

因此，与其回答您的问题并给出需要完成的具体步骤或目标清单，我主要建议是忠于这种方法。最终会证明必要的任何事情都会发生，但通常会以一种非强制的方式发生，从而允许适当的设计、实施和测试，从而实现对我们有限人力的有效利用。

通过简单地不急于或强迫进展，可以躲避很多麻烦。在我看来，“慢就是顺畅；顺畅就是快”这句话在这里适用。

TOM JONES 是 FreeBSD 提交者，对保持网络堆栈快速感兴趣。

—FreeBSD 杂志，2023 年 11 月/12 月
