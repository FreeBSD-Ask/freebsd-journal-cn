# 教授本科生 Unix 课程

- 原文链接：[Teaching an Undergraduate Unix Course](https://freebsdfoundation.org/wp-content/uploads/2022/08/reuschling_teaching_unix.pdf)
- 作者：**BENEDICT REUSCHLING**

## 背景故事——接手 Unix 课程  

2002 年，我还是一名计算机科学专业的本科生，那是我初次真正接触 Unix 系统。我清楚地记得，在我的第一堂编程课上，教授打开了他的 ThinkPad 笔记本，同时以这样一句话开始了讲座：  

> “本系的教授分为两种：一种使用 Unix——我属于这一类；另一种使用 Windows——那就是其他所有人。”  

他不仅教我们编程，还完全在命令行中进行教学。他用 `cat` 命令展示程序，在 `vi` 编辑器中编写代码，使用 `make` 进行编译，在 X11 下播放幻灯片。所有这些都很基础，但却让我产生了浓厚的兴趣——或许是因为我从小用 DOS，却从未意识到终端还能带来什么样的体验。毕竟，在当时的计算机科学圈子里，能在自己的电脑上运行 Linux 或其他 Unix 系统，就意味着你是个“酷”家伙。  

在他的课上，我学完了高级编程、操作系统，接着又修了分布式系统课程。到那时，我已经在自己的笔记本上安装了 Linux，并在发现 FreeBSD 后，很快就完全转向了它。  

这位教授还开设了一门非必修课程，名为 **Unix for Developers**（面向开发者的 Unix）。在这门课上，Unix 本身就是主要的学习内容。虽然它的风格有些“老派”，但对我来说却非常有吸引力。我们学习了各种 Unix 工具，学会了如何高效地编辑文件，并使用 shell 和 `awk` 编写脚本，以便在需要时补充缺失的功能。这门课很有挑战性，但我渴望学习，并在自己的系统上尽可能多地尝试。  

随着时间推移，教授最终换用了 Mac（许多学生也随之跟进），并且参与了更多的课程和项目工作，因而没有时间再教授 **Unix for Developers** 这门课了。那时，我已经毕业，并在系里担任实验室工程师。他可能还记得我对这门课的热情，于是问我是否愿意接手。我答应了，他向我介绍了课程的运作方式。  

当时的课程内容已经有些过时，比如仍然讲授 Linux 2.4 版本的内核，而 Linux 2.6 早已发布。因此，我不仅对课程内容进行了更新，还在其中加入了一些 BSD 相关的内容。  

几年后，我用德语教授了这门课程后，学校问我是否愿意用英语授课，以便让交换生也能学习。由于很多 Unix 相关的概念本身就是英语，翻译成德语反而不够准确，我便重新整理了一遍课程讲义并翻译成了英文。从那以后，我一直用英语授课，甚至偶尔在一些会议上将部分内容作为教程分享。  

但让我们回到最初的那些日子……

## 课程组织结构  

就这样，我接手了一门必修课，并需要安排一个学期的课程内容。这门课在冬季学期授课，大致覆盖 10 月到 1 月间的 15 周，中间有圣诞假期。书面考试安排在 2 月，这意味着从最后一节课到考试日，学生约有两周的时间进行复习。  

课程包括每周一次的 90 分钟讲座，以及单次 3 小时的实验课。由于这门选修课很受欢迎，通常会有 40 名学生选修。因此，每个实验小组大约 16 人，每 14 天轮流上一次实验课，这样每个小组总共会有 5 次实验。实验课虽然不评分，但学生必须完成（通常是两人一组）才能获得考试资格。课程没有期中考核，最终成绩完全由考试结果决定。  

这个评分体系曾被多次讨论，特别是是否应该对实验课进行评分，毕竟学生投入了不少精力。但在这所德国大学中，大多数课程都采用类似的结构（尽管有少数例外），因此它已成为一种固定模式。  

由于这是门选修课，选修的学生都是对这个主题感兴趣的。没有任何学生必须选修这门课才能毕业——他们只需要在成绩单上修满一定数量的选修课程，而具体选哪几门完全由他们自己决定。这不仅减少了总体的学生数量，也导致选修这门课的学生大致可以分为两类：一类是想学习 Unix 但几乎没有基础的，另一类是已经熟悉 Unix，想进一步深入学习（或者只是想拿个轻松的学分）。稍后我们会看到，如何平衡这两类学生的需求并不总是那么容易……  

## 讲座  

正如前面提到的，当我接手这门课程时，它已经有些过时，并且在此之前已经停开了一段时间。我认为可以加入一些 BSD 相关的内容，使课程更具 Unix 通用性，反正我也需要重新制作幻灯片。  

每学期的第一节课，总会有学生问：  

> “我们要用哪款 Linux 发行版？”  

这个问题往往会引起争论，因为 Linux 发行版实在太多了。如果我说 **Ubuntu**，Linux Mint 用户会失望；如果我说 **Mint**，则 Arch Linux 用户会大跌眼镜。  

为了避免这种情况（同时也让所有人有机会学习新知识，而不影响他们已有的习惯），我通常会这样回答：  

> “所有发行版，即没有特定的发行版。我们将使用 FreeBSD，但如果有人想用自己喜欢的发行版，我不会阻止你。毕竟，这是门入门课程，而不是填鸭课程。”

我向学生解释，如果他们是新手，那么从 FreeBSD 入门和使用所有 Linux 发行版的起点是一样的，概念也可以很好地迁移。而对于已经熟悉某款 Linux 发行版的学生，他们可以借此机会接触 BSD 系统，并会发现很多基础知识都是通用的。这通常能让所有人都满意，同时我也成功避开了发行版之争。  

不过，如果学生选择使用自己喜欢的发行版，那他们就不能指望从我这里得到任何帮助。这是他们的系统，他们的选择，他们的管理工作，而非我的责任。  

不同 Linux 发行版的流行程度随着学生一代代更替而变化，而 FreeBSD 一直保持相对稳定——无意外、易上手，而且免费。值得注意的是，我教授的很多内容实际上是与发行版无关的，因此在不同系统之间基本没有区别。有时候，在课堂演示时我们会对比不同系统的细节（稍后会提到），但本质上，我们使用的命令都能完成预期的任务。  

## 课程内容  

本课程包含以下内容：  

- **Unix 概述**（基础知识，如登录、`ls`、`cp` 等命令）  
- **编辑器**（`vi/vim` 速成）  
- **Shell**（命令历史、Tab 补全、重定向、管道、Here 文档、后台作业）  
- **Shell 脚本编程 I**（变量、控制结构、循环、调试）  
- **Shell 脚本编程 II**（`cdialog` 编程、函数、信号捕获 `trap`）  
- **`grep`、`sed`、`awk`**（从文件中提取数据，并进行各种处理，`awk` 编程）  
- **文件系统**（在这一部分介绍 ZFS）  
- **Ansible**（安装、运行临时命令、编写 Playbook，并对多个 jail 并行更改）  

这看起来可能不多，但对于 15 周的课程来说，内容已经相当紧张了。有人可能会质疑，“文件系统”是否适合放在一门偏向编程的课程里。其实，这部分是我当年上这门课时留下的遗产，我认为用 ZFS 替代传统的文件系统概念是个不错的折中方案。  

此外，Shell 脚本编程部分的内容被增补了，因为有些（但不是全部）教授会在操作系统必修课中讲解 Shell。我还增加了对话框编程（可以参考 FreeBSD 安装程序的界面来理解它的用途）、函数和信号捕获（trap）。这些内容相互配合得很好，因为 trap 机制通常需要在执行时调用特定的函数。  

有些内容会随着时间推移而调整，这取决于我的兴趣和学生的反馈。从现实情况来看，在一节 90 分钟的课上，考虑到提问和讨论，每张幻灯片大约需要 2-3 分钟，因此大概只能舒适地展示 40 张幻灯片。  

## 课堂教学模式  

当疫情爆发后，这种传统的授课方式变得难以实施，所以我和大多数同事都转向了翻转课堂（Inverted Classroom）模式。  

在这种教学模式下，学生需要提前自学课程内容，而课堂时间主要用于解答问题和讨论难点。这种方式让教师能够提供更丰富的材料，同时利用课堂时间来判断学生是否遇到了共性问题，并进行集中讲解。  

当然，这种模式也对学生的主动性提出了更高的要求。如果课堂上没有太多问题，我会假设他们已经理解了（这对内向的学生可能不太友好），然后进行一两个演示——通过投影仪/视频会议共享我的终端。  

我发现学生们很喜欢这种方式，因为他们可以立即在自己的机器上尝试，还能看到我犯错误（毕竟人无完人）。相较于传统的“逐张幻灯片讲解”方式，这样的课堂更加动态，也更具吸引力。

课程内容会根据学生的反馈不断添加和更新。如果我发现学生对某个概念感到吃力，我会额外准备几张幻灯片来帮助他们理解。这也取决于学生是否有相关的基础经验，或者完全是 Unix 新手。总体来说，即使是经验丰富的 Unix 用户，在课程中也能学到新东西，所以有些学生以前接触过 Unix 也没有太大影响。  

通常，ZFS 和 Ansible 对学生来说都是新颖且令人兴奋的内容，尤其是 ZFS。很多学生在后来（比如进入硕士阶段时）都会告诉我，他们很高兴当初学了 ZFS，还在家用 NAS 上应用了它。  

## 实验课  

实验课的目的是让学生通过实际操作来证明他们理解了某个主题，并能够将其应用到具体问题中。学生通常以两人一组合作，并向我展示他们的成果进行评估。所有 5 次实验都必须完成，才能参加期末考试。  

实验内容会紧跟课程讲授的内容，但有时也会涉及课堂上没有讲解的部分。这些内容可能是因为太细碎而不适合课堂讲解，或者是一个独立主题，不便纳入课程主线。  

实验任务通常包括：  

- **熟悉系统**（了解 Unix 的基础概念）  
- **编程练习**（或者在此之前，创建有用的 shell 管道）  
- **文本处理**  
- **修改系统配置** 等类似任务  

对我（教师）来说，最难的实验一直是第一个实验——安装 Unix 系统。  

**原因？** 我们的学生通常分为两类：  

- 一部分学生甚至连最基础的安装程序都会卡住；  
- 另一部分学生则早已把系统配置得完美无缺，进入实验室 5 分钟就完成演示，然后直接离开。  

如果学生之前安装过任意一款 Unix 发行版，那么 FreeBSD 的特定部分通常很好上手。但如果是初次接触 Unix，新手可能会觉得很难。  

第一节实验课的目标是让所有学生最终都拥有一款能用的系统，以便后续课程跟上节奏。待所有人都成功安装了系统，后续实验的难度就会大大降低，因为大家都在同一基础环境上进行练习。  

多年来，我尝试了不同的实验教学形式，以找到最适合的方法，既让新手不会被压垮，同时又不让有经验的学生感到无聊。  

起初，我使用投影仪，在 VirtualBox 虚拟机里演示安装过程，边操作边讲解相关概念和术语。这种方法部分有效，但也存在问题——有经验的学生往往会直接跳到下一个步骤，而新手则还在听我讲解。这种情况最终让安装指导变成了一场额外的讲座，而不是实验课本身。

后来，我改用另一种方式——提供一份安装后的系统要求，例如：特定的分区布局、一个独立于 root 的本地用户，以及可用的网络连接。  

然而，这种方法带来的结果五花八门，尽管所有学生都使用了相同的虚拟硬件平台。有些学生到了 14 天后的实验课忘记了自己的密码，有些人分区划得太小，导致无法安装任何软件。此外，作弊也变得更加容易，因为学生可以直接共享已经配置好的虚拟机映像，然后其他人只需导入即可。  

更重要的是，学生并未真正理解安装过程。他们只是一路按**回车键**，完全不了解背后发生了什么（如分区操作、DHCP 网络配置等）。  

## 手动安装方案  

为了解决这些问题，同时让学生更深入理解系统安装的细节，我决定让他们手动安装系统：  

- 进入 **shell**，手动设置分区  
- 解压 **FreeBSD 源码**  
- 配置基础网络  
- 安装 **bootloader**  

同时，我还提供了详细的说明，解释每个命令的作用。  

为了防止作弊，我要求他们在 **gpart** 中用 VirtualBox 生成的唯一磁盘 ID 来标记分区。这样，每个系统都有自己的 ID，我可以轻松检查是否有重复的安装配置。  

这套方案在一定程度上有效，但仍然有一些挑战：  

- 有些学生安装完后重启，却发现系统无法启动，可能是因为不小心把引导代码写到了错误的分区（例如写到了 swap 分区）。  
- 有些学生中途暂停虚拟机，做其他事情后再继续安装，结果发现 FreeBSD ISO 提供的虚拟 CD 盘已经被卸载，导致所有输入的命令都报“command not found”。  
- 还有些学生安装完后，一直只是挂起虚拟机而不重启，直到几周后第一次重启，才发现系统根本无法启动。而最糟糕的是，他们所有的解决方案都存储在这个无法启动的系统里。  

这些学生通常会来问我如何修复他们的安装问题，而我则需要在几个月后回溯他们的具体安装步骤，来找出问题所在……  

尽管我后来在手动安装指南中加入了虚拟机快照的建议，以便学生能随时回滚，但问题依然存在。很多学生不看我的讲解，而是直接翻到下一条命令输入，完全忽略 12 页的详细说明（包括示例图片）。这显然违背了手动安装的初衷——帮助他们理解 FreeBSD 安装过程及其组成部分。  

当然，新手在这个实验上的挣扎远多于 Unix 经验丰富的学生。不过，好消息是，大多数学生最终都能顺利完成实验，只有少数学生遇到困难。  

## 后续的新实验模式  

目前，我正在和一个小型学生团队合作，尝试新的实验课模式。我们计划给每个学生提供一个预配置的 jail 服务器，其中运行着某个应用程序。  

学生的任务是：保持该应用正常运行，而我的任务是在他们的系统中注入各种故障，让他们找到并修复问题。  

为了增加挑战性，我们会引入全球排行榜，根据学生团队解决问题的速度进行排名。我们设计了自动检查程序，它会扫描系统，判断注入的错误是否已被修复，并给予相应分数。  

此外，我还可以为每个团队注入不同类型的错误，甚至同时引入多个故障，例如：  

- **关闭关键服务**  
- **移除执行权限**  
- **删除关键文件**  

这些故障的可能性几乎是无限的（至少在我的设想中是这样）。  

**核心目标是：**  

1. 让学生学会如何维持系统的正常运行，而不是一开始就被安装过程绊住（这更符合企业环境的实际情况）。  
2. 让他们掌握常见错误的排查与修复，提升问题诊断能力。  

目前，我们仍在完善实验细节，但我相信这将会是一种更有趣、更具挑战性的学习方式。

## 考试  

要谈论考试，又不能透露太多具体内容，该怎么说呢？  

自从几年前我改用全英文授课后，学生们一直担心自己听不懂考试题目。但事实证明，这并不是问题。考试题目通常与编程相关，比如：“找出这段短小的 shell 脚本中的错误”，这种形式能很好地弥合语言障碍。此外，考试还包括：  

- **多项选择题**  
- **填空题**  
- **编写一个简短的脚本**  
- **解释 ZFS 里的 CoW（写时复制）是什么意思**  

到了这个阶段，学生们已经熟悉这些题型。从成绩来看，新手和有 Unix 经验的学生在本课程中获得好成绩的机会是相同的。我无法判断后者是否有认真复习，但可以肯定的是，完全不复习并不能保证好成绩。  

考试通常是实验课内容的变形，这也让我能看出实验中谁真正做了练习，谁只是“搭顺风车”。虽然这对学生和我来说都是个迟来的真相，但有时候我的直觉也未必准确，考试成绩往往能带来意想不到的结果。  



## 考试后的反思  

评分完成后，学生可以查看自己的试卷（尽管他们很少会这样做），课程就算正式结束了。但这并不意味着老师的工作也就此告一段落。  

由于这是一门每年开设的课程，我会在暑假期间反思和调整教学内容，基于课堂和实验课中的反馈，优化甚至完全重写某些部分。一般来说，我会重点修改：  

- 在实验课上引发大量提问的内容  
- 在考试中学生提出的小问题  

同时，我也会关注 Unix 领域的新发展，寻找值得加入课程的新内容。在系统管理工作中，我偶尔会遇到一些有趣的代码片段或问题，之后就可能把它们改编成考试题目。暑假里收集这些材料，不仅能让我自己保持新鲜感，也能为下一届学生带来更丰富的课程体验。  

因此，连续两年的课程很少会完全一样，否则不仅我自己会觉得无聊，学生们也可能通过往年的实验和考试答案轻松蒙混过关。  



## 课程的局限性与调整  

我是否能教授 Unix 的所有内容，或者所有我认为学生应该掌握的知识？当然不可能。我所能做的只是揭开 Unix 的一角，希望学生能对它产生兴趣，在课程结束后继续深入学习。  

一些更高级的主题由其他老师负责，例如：  

- **云应用开发管理**  
- **Rust 语言的系统编程**  
- **其他选修课涉及的进阶内容**  

偶尔也有学生抱怨我没有讲 Docker，但我会提醒他们，我们已经学习了 jail，它同样具备强大的功能。  

课程内容也需要适应时代发展。例如，几年前，我们仍然需要在另一门课程中介绍 HTML 基础，但现在大多数学生已经在学校和业余时间接触过它了。类似的情况也发生在硬件知识上——许多学生从未自己组装过电脑，只使用过成品机器。因此，当我们讨论 CPU、RAM 和存储之间的交互时，这些内容对他们来说可能是全新的，尽管它们已经包含在操作系统必修课里。  

如果学生只使用平板电脑，或者只熟悉图形界面，那么让他们适应纯文本的 shell 界面（一个闪烁的光标）可能会很困难。但这不仅仅是 FreeBSD 课程的问题，因为每种 Unix 系统最终都会依赖 shell 交互，即使它们运行在功能齐全的 GUI 上。  



## 课程的意义

尽管 Unix 在操作系统课程中也有所涉及，但那些课程通常更关注：  

- **调度器的工作原理**  
- **MMU（内存管理单元）的功能**  
- **系统调用在编程中的作用**  

相比之下，我的课程更贴近实际，专注于 Unix 作为日常操作系统的使用。  

这门课程当然并不完美，需要不断适应变化。但我喜欢它现在的形式，学生们似乎也很喜欢。  

---

**Benedict Reuschling** 是 FreeBSD 项目的文档提交者，同时是 **文档工程团队** 的成员。他曾连任两届 **FreeBSD 核心团队** 成员，并在德国达姆施塔特应用科技大学**管理一个大数据集群**。此外，他还为本科生教授《Unix for Developers》课程，并担任 **BSDNow.tv** 播客的联合主持人。
