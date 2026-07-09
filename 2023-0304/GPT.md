# 让我们来试试 ChatGPT

- 原文：[Practical Ports: Let’s Talk, GPT](https://freebsdfoundation.org/wp-content/uploads/2023/05/GPT_Practical_ports.pdf)
- 作者：**Benedict Reuschling**
- 译者：冰
- 校对整理：ykla【】部分为 ykla 注

你现在可能已经听说了：在人工智能（AI）领域有一颗新星，充满争议，叫做 ChatGPT。人们已经写了许多关于它的文章，既有讨论它如何通过完成一些被视为杂务的任务来改善人们的生活，也有讨论它如何让机器人取胜从而毁灭我们所有人。各种观点不一，关于是否应该对人工智能进行监管，目前还没有定论。既然这是 Practical Ports 专栏，我将实际看看这种类似聊天的工具能做些什么。

我记得自己大学时上过关于人工智能和自然语言处理的课程。前者是一门枯燥乏味的讲座，没什么价值，因为当时 AI 领域没有重大突破，所以讲座大多围绕该领域过去的努力展开——这简直是一门历史课。

**全能的、无所不知的程序的幻觉当然被打破了。**

我硕士阶段的自然语言课程更有吸引力。课堂上的一个任务是为一个简单版本的 Eliza 编程。当年，当计算机化的治疗师第一次出现时，它备受赞誉，仿佛能让精神分析学家第二天就申请失业。但现在来看，这当然没有发生。由于我们学生必须自己实现这个程序，我们对程序的实际作用有了更好的了解。看到它能做什么令人激动，加上一点随机性，它并没有很快变得太可预测或无聊。看了弗兰肯斯坦软件怪物【《游荡者：弗兰肯斯坦的怪物》是一款游戏。】的内部之后，全能的、无所不知的程序的幻觉当然就破灭了。

ChatGPT 显然更复杂，被灌输的知识也比 Eliza 多得多。这个软件令人印象深刻，但当然应该谨慎对待，下面的实验就证明了这一点。

在工作中，我用老式的方法安装集群机器：启动 FreeBSD ISO 镜像，进入 shell 并设置基本网络（`ifconfig ixl0 up && dhclient ixl0`）。然后，我使用 netcat【netcat 是个计算机网络公用程序，用来对网络连线 TCP 或者 UDP 进行读写。】将 shell 脚本传输到主机上并运行。虽然肯定有更好的自动化方法（如 PXE 启动、预置镜像等），但这个方法我已经用了一段时间。这个脚本并不复杂，但由于底层硬件的细微变化，已经经历了多次演变：一些节点只有 HDD，较新的节点有 NVMe 驱动器，还有一些是 HDD 和 SSD 的混合。虽然脚本运行良好，但我想让 AI 检查一下，看它是否发现了任何明显的错误或有什么改进建议。

节点中四个磁盘的启动分区、交换分区和 freebsd 分区的磁盘设置，再加上 ZFS，总是有点棘手。我的目标是创建 RAID-10 zpool（2 个镜像条带在一起），磁盘上有磁盘序列号作为标识，像这样：

```sh
pool: mypool
 state: ONLINE
config:

NAME              STATE  READ WRITE CKSUM
mypool           ONLINE    0     0     0
 mirror-0        ONLINE    0     0     0 
   gpt/ABC543210 ONLINE    0     0     0
   gpt/DEF123450 ONLINE    0     0     0
 mirror-1        ONLINE    0     0     0
   gpt/IJK098761 ONLINE    0     0     0
   gpt/LMN678901 ONLINE    0     0     0
```

（看，gpt 无处不在，但这个 gpt 指的是 GUID 分区表）。由于每个系统的序列号都不一样，我有一小段例程，可以算出每个磁盘的序列号并创建池，确保将 mirror 关键字添加到序列号列表中，因为 ZFS 需要它来创建 RAID10。

当我第一次听到 GPT 这个词时，我最初的想法是：“为什么会有人和 GUID 分区表对话？”我一定是在日常工作中与分区和磁盘设置打交道太多，所以我想，试一试吧。由于 ChatGPT 中的 GPT 代表生成式预训练变换器（generative pre-trained transformer），区别很明显。在 <https://chat.openai.com/chat> 创建账户后，我看到相当平淡无奇的聊天窗口。于是，我输入了以下内容：

**BCR**【作者向 ChatGPT 提问】：你将如何为这段代码创建函数：

```sh
for S in $SSD; do
      gpart add -t freebsd-swap -l swap${COUNTER} -s ${SWAP} -a 4k "${S}"
      echo "Setting ${SWAP} of swap space on disk ${S}."
      SERIAL=$(camcontrol identify "${S}"|awk '/serial number/ { print $3 }')
          gpart add -t freebsd-zfs -l "{$SERIAL}" -a 4k "${S}"
      if [ $((COUNTER%2)) -eq 0 ]; then
            SERIALS="${SERIALS} mirror"
      fi
      SERIALS="${SERIALS} /dev/gpt/${SERIAL}"
      COUNTER=$((COUNTER+1))
      NUM_SSDS=${COUNTER}
done
```

现在，这些代码既不美观，也不优雅，但能用。这就是 ChatGPT 的回答：

**ChatGPT**：根据你提供的代码片段，我认为你是想创建一个在多个磁盘上设置交换和 ZFS 分区的函数。这里有一个函数样本，可以完成这个任务：

```sh
setup_partitions() {
    local SSD=$1
    local SWAP_SIZE=$2
    local COUNTER=1
    local SERIALS=""
     
    for S in $SSD; do
        # 创建交换分区
        gpart add -t freebsd-swap -l swap${COUNTER} -s ${SWAP_SIZE} -a 4k "${S}"
        echo "Setting ${SWAP_SIZE} of swap space on disk ${S}."
        
        # 创建 ZFS 分区
        SERIAL=$(camcontrol identify "${S}"|awk '/serial number/ { print $3 }')
        gpart add -t freebsd-zfs -l "${SERIAL}" -a 4k "${S}"
        
        # 添加到序列号列表并增加计数器
        if [ $((COUNTER%2)) -eq 0 ]; then
            SERIALS="${SERIALS} mirror"
        fi
        SERIALS="${SERIALS} /dev/gpt/${SERIAL}"
        COUNTER=$((COUNTER+1))
    done
    
    NUM_SSDS=${COUNTER}
}

```

**ChatGPT**：你可以通过传递两个参数来调用这个函数：SSD 数组和所需的交换空间大小。例如，如果你有两个固态硬盘（**/dev/sda** 和 **/dev/sdb**），你希望它们每个都有 4GB 的交换空间，你可以这样调用这个函数：

```sh
SSD=("/dev/sda" "/dev/sdb")
SWAP_SIZE="4g"
setup_partitions "$SSD" "$SWAP_SIZE"
```

好吧，这确实是令人印象深刻的开始。它能找出我想做的事情，并提供了解决方案和合适的解释。仔细审视，它需要做一些修改：首先，代码是用 bash 写的（正如代码片段窗口所标注的那样）。由于我们处于安装程序环境中，而 FreeBSD 默认不包含 bash，所以 bash 特有的结构无法工作。第二，它假定我使用的是 Linux，因为磁盘设备节点（**/dev/sda**）表明了这一点。不过，其余的代码都很好，而且它不必修改我自己的大部分代码，只是将其包装成函数。

现在，我的提示（我在聊天中的输入）对我正在运行的系统没有太具体的说明，所以人工智能不得不对我正在运行的 shell 和设备名称做出最佳猜测。我应该事先提供这些信息以避免人工智能走错方向。

令我印象深刻的是，答案是如此具体。感觉就像我在和同事聊天，通过 IRC 或其他文字交流方式讨论这个代码。这种具体性是不容小觑的。在我读书的时候（我现在觉得自己老了），互联网刚刚成为主流，我们仍然要到图书馆去获取某些信息。如果我遇到这样的问题，我就不得不去找一本关于 shell 脚本的书，或者问同学。但在书中，我可以得到关于编程的一般概念、解释和示例代码。我仍然需要将所读内容应用到问题代码上，想出合适的解决方案。

现在，人们会用谷歌搜索他们的问题，并在 stackoverflow 等网站或编程相关论坛上找到类似问题。如果我自己找不到解决方案，我甚至可以在那里发布代码并寻求帮助。但这需要花时间等待答案（如果真的有人愿意回复的话），也许还要经过几次反复，讨论各种方法。

有了 ChatGPT 这样的工具，就像有个人编码助理可以讨论你的代码，而不需要长时间等待答案。就像总工程师 LaForge 与企业号的计算机进行“舒适的小谈话”一样，第二双眼睛可以提供新的想法和见解，而这些是一个人不会想到的。我甚至可以重新生成 ChatGPT 给我的答案，或者要求提供更多细节。由于人工智能似乎了解问题领域（比书本可能更好），所以当它获得足够的信息时，它可以提出具体的建议。由于我的同事并非 24/7 都能与我讨论一段代码，我可以看到，当我需要快速得到好的答案时，人工智能是很有用的。

受到鼓励，我继续给人工智能更大的例子：

**BCR**：你会如何将这段代码统一为一个通用函数，供两者使用？

```sh
for HD in $HDD; do
        gpart add -t freebsd-swap -l swap${COUNTER} -s ${SWAP} -a 4k "${HD}"
        echo "Setting ${SWAP} of swap space on disk ${HD}."
        SERIAL=$(camcontrol inquiry "${HD}" -S|sed -e 's/^[[:space:]]*//')
        
        if [ $((COUNTER%2)) -eq 0 ]; then
                SERIALS="${SERIALS} mirror"
        fi
        SERIALS="$SERIALS /dev/gpt/${SERIAL}"
        
        gpart add -t freebsd-zfs -l "${SERIAL}" -a 4k "${HD}"
        COUNTER=$((COUNTER+1))
done

COUNTER=0
# 我们从其中一个 SSD 启动
for S in $SSD; do
        gpart add -t freebsd-swap -l swap${COUNTER} -s ${SWAP} -a 4k "${S}"
        echo "Setting ${SWAP} of swap space on disk ${S}."
        SERIAL=$(camcontrol identify "${S}"|awk '/serial number/ { print $3 }')
gpart add -t freebsd-zfs -l "${SERIAL}" -a 4k "${S}"

        if [ $((COUNTER%2)) -eq 0 ]; then
                SERIALS="${SERIALS} mirror"
        fi
        SERIALS="$SERIALS /dev/gpt/${SERIAL}"
        COUNTER=$((COUNTER+1))
        NUM_SSDS=${COUNTER}
done
```

**ChatGPT**：为了将这两段代码统一到通用函数中，你可以创建函数，该函数需要三个参数：磁盘数组、所需的交换空间大小、指示磁盘是 HDD 还是 SSD 的选项。下面是它的示例函数：

```sh
setup_partitions() {
    local disks=("$@")
    local swap_size=$2
    local is_hdd=$3
    local counter=1
    local serials=""
    
for disk in "${disks[@]}"; do
    # 创建交换分区
    gpart add -t freebsd-swap -l swap$counter -s $swap_size -a 4k "$disk"
    echo "Setting $swap_size of swap space on disk ${disk}."
    
    # 确定磁盘序列号
    if [[ $is_hdd == "true" ]]; then
        serial=$(camcontrol inquiry "$disk" -S | sed -e 's/^[[:space:]]*//')
    else
        serial=$(camcontrol identify "$disk" | awk '/serial number/ { print $3 }')
    fi
    
    # 创建 ZFS 分区
    gpart add -t freebsd-zfs -l "$serial" -a 4k "$disk"
    
    # 添加到序列号列表并增加计数器
    if [ $((counter%2)) -eq 0 ]; then
        serials="$serials mirror"
    fi
    serials="$serials /dev/gpt/$serial"
    counter=$((counter+1))
done

NUM_DISKS=$counter
}
```

**ChatGPT**: 你可以用磁盘数组、所需的交换空间大小和指示磁盘是 HDD 还是 SSD 的选项来调用这个函数。例如，要在两个 HDD（**/dev/ada0** 和 **/dev/ada1**）上设置具有 2GB 交换空间的分区，和两个 SSD（**/dev/ada2** 和 **/dev/ada3**）上设置具有 4GB 交换空间的分区，你可以这样调用这个函数：

```sh
DD=("/dev/ada0" "/dev/ada1")
SSD=("/dev/ada2" "/dev/ada3")
SWAP_SIZE_HDD="2g"
SWAP_SIZE_SSD="4g"
setup_partitions "${HDD[@]}" "$SWAP_SIZE_HDD" "true"
setup_partitions "${SSD[@]}" "$SWAP_SIZE_SSD" "false"
```

**ChatGPT**：请注意，我添加了一个选项，以表明这些磁盘是 HDD 还是 SSD，因为每种磁盘类型的序列号检索方法不同。

当然，这是另一个令人印象深刻的回答。尤其是除了给我一些随机的代码片段外，回复和解释的内容如此丰富。它确实找出了两个函数的相似之处，并能区分出我的例子中 SSD 和 HDD 的交换空间大小是不同的。它很聪明地找出了正确的磁盘设备节点（**/dev/ada0** 而不是 **/dev/sda**）。此外，我的一些编码风格（如大写变量名）被改成了小写。这可能合你的意，也可能不合，需要事后做一些手工编辑。代码仍然是用 bash 给出的，其中包含一些 bash 特有的语言特性，如数组，在 Bourne shell【即 sh】中是没有的。

但我用下面的提示纠正了我的重复错误：

**BCR**：你能不能只在 **/bin/sh** 中提供同样的代码（没有 bash）？

**ChatGPT**：当然可以！这里是 **/bin/sh** 语法的相同代码：

```sh
#!/bin/sh

setup_partitions() {
    disks="$1"
    swap_size="$2"
    is_hdd="$3"
    counter=1
    serials=""
    
for disk in $disks; do
    # 创建交换分区
    gpart add -t freebsd-swap -l swap$counter -s $swap_size -a 4k "$disk"
    echo "Setting $swap_size of swap space on disk ${disk}."
    
    # 确定磁盘序列号
    if [ "$is_hdd" = "true" ]; then
        serial=$(camcontrol inquiry "$disk" -S | sed -e 's/^[[:space:]]*//')
    else
        serial=$(camcontrol identify "$disk" | awk '/serial number/ { print $3 }')
fi

    # 创建 ZFS 分区
    gpart add -t freebsd-zfs -l "$serial" -a 4k "$disk"
    
    # 添加到序列号列表并增加计数器
    if [ $((counter%2)) -eq 0 ]; then
        serials="$serials mirror"
    fi
    serials="$serials /dev/gpt/$serial"
    counter=$((counter+1))
done

NUM_DISKS=$counter
}

HDD="/dev/ada0 /dev/ada1"
SSD="/dev/ada2 /dev/ada3"
SWAP_SIZE_HDD="2g"
SWAP_SIZE_SSD="4g"

setup_partitions "$HDD" "$SWAP_SIZE_HDD" "true"
setup_partitions "$SSD" "$SWAP_SIZE_SSD" "false"
```

**ChatGPT**：请注意，**/bin/sh** 和 **/bin/bash** 在语法上有一些不同，比如使用 `=` 而不是 `==` 来进行字符串比较，缺少数组变量。为了解决这些差异，我将磁盘列表作为空格分隔的字符串传递，并使用 `$1`、`$2` 和 `$3` 来访问函数参数。

因此，它能够将代码转化为可运行的 Bourne shell 语法（我每次都测试了结果）。现在我看了一下统一的函数，我又有了想法（这也可能来自与同事讨论时一起查看代码）：

**BCR**: 当我们用同样的 4g 作为 HDD 和 SSD 的交换空间大小时，代码会是什么样子？

**ChatGPT**: 如果我们想对 HDD 和 SSD 使用同样的 4g 交换空间大小，我们可以修改 `setup_partitions` 函数，使其只接受一个磁盘参数，并删除 `is_hdd` 参数和确定磁盘序列号的条件逻辑。下面是更新后的代码在 **/bin/sh** 语法中的样子：

```sh
#!/bin/sh

setup_partitions() {
    disks="$1"
    swap_size="$2"
    counter=1
    serials=""
    
    for disk in $disks; do
        # 创建交换分区
        gpart add -t freebsd-swap -l swap$counter -s $swap_size -a 4k "$disk"
        echo "Setting $swap_size of swap space on disk ${disk}."
        
        # 确定磁盘序列号
        serial=$(camcontrol identify "$disk" | awk '/serial number/ { print $3 }')
        
        # 创建 ZFS 分区
        gpart add -t freebsd-zfs -l "$serial" -a 4k "$disk"
        
        # 添加到序列号列表并增加计数器
        if [ $((counter%2)) -eq 0 ]; then
            serials="$serials mirror"
        fi
        serials="$serials /dev/gpt/$serial"
        counter=$((counter+1))
done

NUM_DISKS=$counter
}

HDD="/dev/ada0 /dev/ada1"
SSD="/dev/ada2 /dev/ada3"
SWAP_SIZE="4g"

setup_partitions "$HDD $SSD" "$SWAP_SIZE"
```

**ChatGPT**：在这里，我们将 HDD 和 SSD 磁盘列表作为一个空格分隔的字符串传递给 `setup_partitions` 函数，并对两种类型的磁盘使用相同的交换空间大小 4g。

好吧，作为重构步骤，这当然不错。这一次，它确实从我先前的输入中了解到我想要 **/bin/sh** 代码，所以它在接下来的回复中默认使用了它。

请记住，结果可能会有所不同，可能会也可能不会像这里显示的那样好用。人工智能是不断发展的，我肯定用我的提示对它进行了一点点训练，以获得更好的效果。每个回复都可以选择用熟悉的拇指向下或向上的按钮进行评分。

接下来，我想看看人工智能对任何可能的优化有什么看法：

**BCR**：你能建议对代码进行其他优化吗？

**ChatGPT**：当然可以，这里有几个潜在的优化方案：

1. 使用一种更有效的方式来增加计数器变量：在循环中不要使用 `counter=$((counter+1))`，你可以使用快捷方式 `counter=$((counter++))`，甚至只使用 `((counter++))`。这样可以避免创建子 shell 来执行算术操作，这样会比较慢。
2. 使用 echo 的 `-n` 选项可以避免打印换行符：当打印关于设置交换空间的信息时，你可以使用 echo 的 `-n` 选项来避免打印换行符。因为你在它后面紧接着另一条信息。这将使输出看起来更干净。
3. 使用参数扩展来删除空白：与其使用 `tr -d '[:space:]'` 来删除序列号中的空白，不如使用参数扩展来达到同样的效果：`${variable##*[[:space:]]}`。这就避免了产生额外进程的开销，而且可能会更快。

下面是应用这些优化后的代码：

```sh
#!/bin/sh

setup_partitions() {
    disks="$1"
    swap_size="$2"
    counter=1
    serials=""
    
for disk in $disks; do
    # 创建交换分区
    gpart add -t freebsd-swap -l swap$counter -s $swap_size -a 4k "$disk"
    echo -n "Setting $swap_size of swap space on disk ${disk}. "
    
    # 确定磁盘序列号
    serial=$(camcontrol identify "$disk" | grep "serial number" | cut -d':' -f2 |
    ${SED-sed} 's/^[[:space:]]*//')
    
    # 创建 ZFS 分区
    gpart add -t freebsd-zfs -l "$serial" -a 4k "$disk"
    
    # 添加到序列号列表并增加计数器
    if [ $((counter%2)) -eq 0 ]; then
        serials="$serials mirror"
    fi
    serials="$serials /dev/gpt/$serial"
    ((counter++))
  done
  
  NUM_DISKS=$counter
}

HDD="/dev/ada0 /dev/ada1"
SSD="/dev/ada2 /dev/ada3"
SWAP_SIZE="4g"

setup_partitions "$HDD $SSD" "$SWAP_SIZE"
```

**ChatGPT**：这些优化应该能让代码效率略微提高，但请记住，对性能的影响可能可以忽略不计，这取决于磁盘的大小和正在创建的分区的数量。

这些当然是我没有考虑过的一些优化。当通过这样的人工智能运行你的代码时，我建议你以开放的心态对待它，把你的程序员的自负留在登录提示符处。人无完人，人工智能可能会提出一些理论上不正确的建议，或者让你担心失去工作，因为到目前为止你写的那些可怕的代码可能会传到你老板的耳朵里。但是，既然这只是你和人工智能之间的事，有什么能阻止你实施一些建议，看看它们是否真的能改善情况？

这就是我发现这些工具是一把双刃钝剑的地方：如果学生用它来生成所布置任务的解决方案，没有人阻止他们。这和其他工具一样，无论是你的 shell、你的编译器，还是方便的键盘快捷键【此处指复制粘贴结果】。但是，如果他们不能解释，或者只是拿着代码而不反思它的作用，就会变得很危险。作弊者是不会成功的，尤其是当他们有一天必须在没有任何助手的情况下在白板上向考试委员会展示解决方案时。需要理解，也需要对所提交的解决方案进行批判性反思。它可能有缺陷，效率低下，或者完全是从别人那里偷来的，那人在论坛或 stackoverflow 上提供了这个确切的解决方案，并被送入人工智能的训练数据。这是专家们仍在讨论的争议问题。

试着把它看作帮助工具——第二双人工的眼睛看你的代码，提供有价值的见解。另外，要注意考官知道这一点，并可能自己使用这些工具——或者为每个学生设计独特的作业或考试。

我自己与人工智能的讨论又继续了一会儿，聊了这段代码的一些部分。我还让它在另一个聊天中给我建议，关于某些尺寸的站立式办公桌（我的工作空间有点有限），以及我可以在 hadoop 配置文件中做哪些优化，以加快对 HDFS 的写入速度。

我的建议是，人们可以尝试一下，但要对输出的结果持谨慎态度，并在有意义的时候使用它。我认为这些工具可以改善某些领域，比如使代码更健壮，或者激发新的想法，然后必须由你来实现。我估计不会很快因为这些工具的存在而失去自己的工作，因为我们人类拥有的某些技能在 ChatGPT 及其同类工具中还不具备。

---

**BENEDICT REUSCHLING** 是 FreeBSD 项目的文档提交者和文档工程小组的成员。他曾在 FreeBSD 核心团队任职两届。他在德国达姆施塔特应用科学大学管理着大数据集群。他还在为本科生教授"Unix for Developers"【UNIX 开发】课程。他还是每周 bsdnow.tv 播客的主持人之一。
