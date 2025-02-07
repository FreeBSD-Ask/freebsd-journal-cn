# 活动监控脚本（activitymonitor.sh）

- 原文链接：[Tom Once Again,  Does Stupid Things  with a Computer:  activitymonitor.sh](https://freebsdfoundation.org/wp-content/uploads/2023/01/jones_activitymonitor.pdf)
- 作者：**TOM JONES**

在今年的维也纳 EuroBSDCon 上，我讲述了我在 FreeBSD 和 Linux 上研究 QUIC 性能的工作。  

我的测量方法的核心部分是网络传输期间 CPU 饱和，利用所有可用的 CPU 周期进行发送。我通过两种方式使用 CPU 饱和度，第一种是检测 CPU 是否是系统中的瓶颈。如果它不是瓶颈，那么我需要确认网络是否是瓶颈，而不是我没有尝试测量的其他组件。其次，待我控制了 CPU 饱和度并确保没有其他瓶颈在起作用，我就可以利用实际的 CPU 使用率来估算系统如果能够饱和网络卡的 UDP，发送速度能有多快。这非常有用，因为在所有测量中，我的网络接口在任何操作系统上都只能处理大约 6Gbit/s 的 UDP 流量。但这并不会饱和 CPU，CPU 的利用率大约为 70%。通过良好的 CPU 测量，我可以发明一个指标，乐观地预测如果网络接口不成为障碍，处理器能做到的速度。

我的 EuroBSDCon 演讲基于尚未发布的学术工作，似乎是一个好主意，通过一个经过深思熟虑的方法来查看网络测试期间的 CPU 性能。我的一些测试灵感来自 Fastly 的工作，Fastly 比较了 QUIC 和 TCP 的计算效率。尽管 Fastly 似乎建立了一个很好的测试平台，但我从他们的写作中能理解到的唯一机制是"凭眼睛看" top。  

这对于发布结果或准确评估数千个测试结果来说并不足够好。如果仅仅看 top 不够，那我是否可以去了解 top 是如何查看的，并重新实现它呢？


## top 如何估算 CPU 利用率？  
top 可能是我们在想知道系统上发生了什么时，首先使用的工具。尽管它非常实用，top 其实是个非常简单的程序，它能够在屏幕上漂亮地显示信息，并且在机器上做了一些抽象层处理，使得代码更加可移植。CPU 利用率接口由 FreeBSD 提供，以两个 sysctl 节点的形式呈现：

![](https://github.com/user-attachments/assets/e25b847c-5b9d-42ba-9812-44ac1c9c757e)

```sh
$ sysctl hw.ncpu
4
$ sysctl kern.cp_time
kern.cp_time: 3832370 11408 3650627 44926 2061043745
$ sysctl kern.cp_times
kern.cp_times: 1080959 3035 954019 4354 515101995 1062132 2823 815176 1143
515263088 960320 3419 980090 37760 515162773 728956 2131 901334 1668 515510273
```

第一个节点 `kern.cp_times` 返回处理器的五个值，报告自启动以来在用户、优先级、系统、中断和空闲状态下花费的总时间。`kern.cp_times` 报告系统中每个处理器的这五个值。通过使用 `kern.cp_times` 和 `hw.ncpu`，我们可以细分这个列表。通过这两个 sysctl，我们可以获得自启动以来的系统总时间使用情况以及每个处理器的时间使用情况。自启动以来的使用情况对于理解机器的工作状态很有帮助，并且对于观察系统使用情况在短时间内如何变化也很有用。top 启动时显示系统的总时间分解，但待刷新（默认每秒刷新一次），它将显示这些字段在这一秒内的变化。

## 绘图
通过非常容易复现 top 的功能，我想知道是否可以定期抓取系统 CPU 使用率并将其绘制出来。我想我可以将这些绘图与吞吐量图并排显示，以展示在测试之间，主机几乎完全空闲，并且在测试运行时，单核 CPU 达到了饱和。

多年来，我在绘图时使用过不同的工具，但在这项工作中，我对定制工具感到厌倦，想要简洁些。我本可以使用最简单的方法——电子表格来绘制利用率，但我觉得能给我更多控制的东西会更好。在其他近期的工作中，我使用了基于 Web 的绘图工具。c3js 库提供了很好的交互式图表，但当数据点数量过多时（超过几万）会有些力不从心。考虑到我还要查看网络使用情况，数据量在一分钟的录制中会非常大。当我在考虑工具时，我想起了我为 Klara Systems 撰写的关于 `inetd` 的一篇文章。在写这篇文章时，我创建了一个自己的简单 `inetd` 服务，通过 shell 脚本实现了日期时间服务。难道我可以通过 `inetd` 从主机实时提供 CPU 使用信息吗？

## activitymonitor.sh

这将我们引向 `activitymonitor.sh`，一个不按常规出牌的创作，违背了所有良好的规范，却能在网页浏览器中提供简单的图表。`activitymonitor.sh` 是个单一的 shell 脚本，包含三个部分：  
• 由 `inetd` 运行的脚本。  
• 一个基本的 HTML 页面。  
• 一些 JavaScript 用于更新实时页面。

`inetd` 是一个互联网服务启动程序。`inetd` 的历史和功能略微超出了本文的范围，但它用于在主机过于小而无法保持等待服务时按需启动应用程序。`inetd` 负责监听流量。当接收到连接或数据报（对于基于 UDP 的协议）时，`inetd` 启动内建处理程序或指定的程序。该程序从网络连接的标准输入读取。所有写入标准输出的内容将通过网络发送。这是一个非常简单的接口，但足够强大，可以实现纯文本的服务器协议。

## 一个小脚本
这个小脚本非常简单。它有两个主要组件，然后是附加的网页和 JavaScript 数据。首先，shell 脚本处理作为 `inetd` 的客户并解析 HTTP 头部。脚本的输入将是客户端在发出请求时发送的 HTTP 头部。

```sh
# 读取客户端头信息，我们只关心其中是否有 data.json。
h=""
while read -t 1 h
do
 log $h

 if echo $h | grep -q "data.json";
 then
 page="data.json"
 contenttype="application/json"
 else
 fi
done
echo "HTTP/1.0 200 OK"
echo "Content-Type: $contenttype"
echo
```

该脚本使用带有超时的 `read` 内建命令，这意味着脚本会消耗套接字上的所有输入，直到输入行之间出现 1 秒的间隔才会继续执行。这个 `read` 超时是限速机制。然后检查 HTTP 头部，查看是否正在请求数据 URL，如果不是，则返回基础的 HTML 页面。

脚本的 `data url` 路径是我们收集关于主机的有趣数据的地方。`activitymonitor.sh` 实现了 `top` 的大部分默认接口。为了做到这一点，它使用 FreeBSD 的基础命令收集所需的信息，并将其编码为 JSON 格式的 blob 返回给请求者。

```sh
if["$contenttype" == "text/html" ]
then
 indexstart=$((cat -n $0 | grep -e 'INDEX START'\
 | awk '{print $1}' | tail -n 1+1))
 sed -n"$indexstart"',$p' $0
elif["$contenttype" == "application/json" ]
then
 psout=$(ps -ax -o \
 "user,pid,%cpu,cpu %mem,vsz,rss,state,command"\
 --libxo json)
 vmstatout=$(vmstat –libxo json)
 netstatout=$(netstat -bi –libxo json)
 # kern.cp_time(s) gives us 5 numbers for the system:
 # user nice system interrupt idle
 # kern.cp_times gives us hw.ncpu entries for those 5 values
 totalcputime=$(sysctl -n kern.cp_time)
 percputime=$(sysctl -n kern.cp_times)
 ncpu=$(sysctl -n hw.ncpu)
 loadavg=$(sysctl -n vm.loadavg)
 lastpid=$(sysctl -n kern.lastpid)
 hostname=$(sysctl -n kern.hostname)
 system=$(printf ‘{"hostname":"%s",
 "cp_time":"%s", "cp_times":"%s", "ncpu":"%s",
 "loadavg":"%s", "lastpid":"%s"}’ "$hostname"
 "$totalcputime" "$percputime" "$ncpu"
 "$loadavg" "$lastpid")
 log $system
 physmem=$(sysctl -n hw.physmem)
 pagesize=$(sysctl -n hw.pagesize)
pagecount=$(sysctl -n vm.stats.vm.v_page_count)
 wirecount=$(sysctl -n vm.stats.vm.v_wire_count)
 activecout=$(sysctl -n vm.stats.vm.v_active_count)
 inactivecount=$(sysctl -n vm.stats.vm.v_inactive_count)
 cachecount=$(sysctl -n vm.stats.vm.v_cache_count)
 freecount=$(sysctl -n vm.stats.vm.v_free_count)
 memory=$(printf '{"physmem":"%s", "pagesize":"%s",
 "pagecount":"%s", "wirecount":"%s",
 "activecout":"%s", "inactivecount":"%s",
 "cachecount":"%s", "freecount":"%s" }’
 "$physmem" "$pagesize" "$pagecount"
 "$wirecount" "$activecout" "$inactivecount"
 "$cachecount" "$freecount")
 log $totalcputime
 # deliver the data json
 printf '{"system":%s, "memory":%s, "ps":%s,
 "vmstat":%s, "netstat":%s}'"$system"
 "$memory" "$psout" "$vmstatout" "$netstatout"
fi
exit # 不要继续进入网页
```

脚本收集的第一组信息来自于支持 libxo 的 FreeBSD 工具。Libxo 是 FreeBSD 的一个非常强大的特性——支持的基础工具可以本地输出 JSON 格式的数据。我们获取了 `ps`、`vmstat` 和 `netstat` 的输出。这使我们能够显示进程、虚拟内存系统信息以及网络统计信息，如接口速率。脚本中的第二组信息直接来自 sysctl 接口。目前，我们获取了 `kern.cp_time(s)`、CPU 数量、主机名、负载平均值和关于内存的基础统计信息。每个信息都必须通过 `printf` 手动打包成 JSON 对象。所有这些信息最终被构建成一个 JSON 对象，并在简单的响应头之后由脚本打印出来。`inetd` 然后将其反馈到连接的套接字，并作为 HTTP 响应的主体返回给客户端。

### 基本网页

`activitymonitor.sh` 脚本在内部嵌入了一个小型网页，当没有请求数据 URL 时，它会返回这个页面。页面包含一个头部、一些画布用于让 JavaScript 绘制图表，以及一个用于显示 `top` 风格进程列表的 `pre` 块。它还嵌入了让所有功能得以实现的 JavaScript。HTML 页面（和 JavaScript）被追加到脚本的末尾，并用 'INDEX START' 标签标记。`activitymonitor.sh` 在自身中搜索该标签，并将标签后的所有内容作为要交付的内容，使用 `sed` 进行切割。

## 一些 JavaScript

JavaScript 执行所有繁重的任务，包括从数据中解析信息并提供用户界面。它提取并处理 `kern.cp_times` 的值，并计算绘制图表所需的增量。除了绘制之外，主要的功能是从 `activitymonitor.sh` 的数据端请求数据。当基本网页加载完成后，脚本会启动一个任务来获取 `/data.json` 链接。当数据成功到达时，它会从 JSON 结果中提取所需字段，并合并新数据以计算应该显示的内容。最后，它再次调用 `getdata` 开始此任务。由于 `activitymonitor.sh` 中的读取超时，这将在结果之间有 1 秒的间隔发生。

## 这是个坏主意

`activitymonitor.sh` 是一个有趣的小项目，最终几乎成为了一个可用的工具。CPU 绘图帮助我理解了 FreeBSD 调度器如何非常积极地在 CPU 之间移动进程，并帮助我制定了一个测量策略来考虑这一点。尽管是一个有趣的项目，但它并不适合在现实世界中使用。相反，它展示了 FreeBSD 基本系统中工具的组合能力。除了 `sysctl`，我们使用的所有工具本地都输出 JSON 格式，并且可以被强大的用户界面语言使用。这种原生对 JSON 的支持使得消费标准工具的输出变得容易，并且让自动化可以与人类消费的数据一起进行。它赋予我们用机器可读的方式构建系统的能力，使用的是我们在屏幕上读取的相同数据。这是超越传统 UNIX 接口的一项小增强，但却是非常强大的。

可在此获取 `activitymonitor.sh` 。需要类似以下的 `inetd` 配置：

```sh
http-alt stream tcp nowait tj /home/tj/code/activitymonitor.sh
activitymonitor.sh
```
---

**TOM JONES** 希望基于 FreeBSD 的项目能够获得应有的关注。他住在苏格兰东北部，并提供 FreeBSD 咨询服务。
