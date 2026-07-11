# STDDEV，否则就是没发生过！

作者：Poul-Henning Kamp

## STDDEV：“It Didn’t Happen!”

许多许多年前，当 Dilbert 漫画还好笑的时候，我尝试做一种最好形容为”纳基准测试”的事情。基本思路是把两台硬件完全相同的 FreeBSD 计算机以无盘配置启动，连续运行数天，并测量对 FreeBSD 内核代码的提议补丁是改进还是恶化了性能。

很快显而易见，主板上的石英晶体更适合测量温度而非时间，这把我的精度限制在最多三位有效数字。

作为一个”time-nut”，我用外部信号替换了这些晶体，信号来自一块高质量的恒温石英晶体，而后者又由一台当时最先进的 Motorola Oncore 计时 GPS 接收器的 GPS 信号驱动。这让我达到了八位有效数字。

……是吗？

那时我主动翻开统计学教科书，开始欣赏统计学这一工具。统计学不是任何正常人喜欢的数学分支；它涉及大量数字运算，给出模糊的答案：65% 的美国人认为冷冻披萨永远不可能好吃，科学对此无能为力。来源：Widgery & Associates。误差范围 9%。基于 1993 年春对 204 名美国人的电话调查 [1]。

但即便对上一世纪的发展做最粗略的研究，也会发现它就是统计学，从 Erlang 的电话系统容量公式，到 Shewhart 对电话听筒制造的质量控制，再到 Google 的 PageRank，以及当前的“大数据”泡沫——我相信某位未来的历史学家会在一本名为《Covariances Gone Wild》的书中记录这一切。

统计学强大却艰难。我最喜爱的一篇被忽视的统计学文章——该文章将全球变暖的概率确定为至少十万比一（1992 年，即 Windows 3.1 问世那年）——花了我整整一周才理解 [2]。

由于统计学是艰苦的工作，大多数人干脆完全跳过它，因此几乎所有发到 FreeBSD 邮件列表的关于性能改进的声明，包括我自己的，都是没有证据、仅凭轶闻的断言。

在 FreeBSD 5 前后，我决定为此做点什么，但做什么？向人们推荐像 R 这样的重型统计包是行不通的：我自己试过并放弃了。项目需要的是一个”软件工具”：一个把一件事做好、无须大量脑力或其他劳动的程序。

我写了一个约 500 行的 C 程序，从一个或两个输入文件每行读取一个数字，用简陋的 ASCII 艺术图绘制数据，在图下方打印每个输入文件的基本统计量：最小值、最大值、平均值、中位数和标准差。如果给两个输入文件，它会计算学生 T 检验，并报告两个数据集是否存在超过 95% 的概率以有意义的方式不同：

```sh
$ ministat iguana.txt chameleon.txt
x iguana.txt
+ chameleon.txt
+------------------------------------------------------------------------------+
|x         *  x                   *    +                   +   x         +|
| |_________M___|___A_____________M__A_________________|             |
+------------------------------------------------------------------------------+
N          Min     Max            Median        Avg       Stddev
x    7          50      750              200        300    238.04761
+    5           150      930              500        540    299.08193
No difference proven at 95% confidence
```

Ministat 是一个极其简陋的工具。ASCII 图看起来糟糕；它没有刻度，仅勉强够发现离群点、双峰分布和其他主要数据问题，但它有一个大优势：可以作为纯文本通过电子邮件发送。

学生 T 是统计学中相对较新的突破，仅 100 年历史 [3]，虽然它并非对所有情况都是统计上最优的工具，但对输入数据而言非常稳健，且不需要太多技巧就能解释其是/否结果。正因如此，我把 ministat 提交到 FreeBSD，提交信息是：

> If your benchmarks are not significant at the 95% confidence level, we don’t want to hear about it.
> （如果你的基准测试在 95% 置信水平上不显著，我们不想听。）

随源码附带的示例数据文件来自《The Cartoon Guide to Statistics》[4]，这是一本以连环画形式写就的合格统计学入门教材，通篇是”老爹冷笑话”，作者是不朽的 Larry Gonick [5]。

Ministat 并非一夜成名；它带有”统计学”的全部污名，但经过几年提醒人们它的存在之后，它已成为 FreeBSD 中讨论性能改进的事实标准方式，我也常常在 BSDcan 演讲中看到它。

我认为 OpenBSD 是它在 FreeBSD 之外出现的第一个地方，但今天 Linux 也有软件包，至少还有用 Python、Ruby 和 Erlang 编写的 ministat 移植版。

而且时不时有陌生人请我喝啤酒，因为 ministat 当然必须采用 beerware 许可证：学生 T 由 William Gosset 发现，他是 Guinness 啤酒厂的一名员工 [6]。

## PHK 的好基准测试小贴士

- 在单用户模式下运行。**cron(8)** 和所有其他守护进程只会增加噪声。
- 确保 CPU 始终以同一时钟速度运行。（BIOS 设置等；AC 电源等。）
- 如果生成了 syslog 事件，用空的 `syslogd.conf` 运行 syslogd；否则不要运行。
- 尽量减少磁盘 I/O；能做到完全避免最好。
- 不要挂载你不需要的文件系统。
- 尽可能以只读方式挂载文件系统。
- 对你的可读写测试文件系统 newfs，并从 tar 或 dump 文件填充它；然后在每次运行前卸载并重新挂载。
- 如果可以，使用 malloc 后端或预加载的 **MD(4)** 分区。写入时，SSD 比旋转盘片更不可预测。
- 从内核中移除所有不必要的设备驱动。例如，如果测试不需要 USB，就不要把 USB 放进内核。attach 的驱动常常有定时器在计时。
- 卸载不使用的硬件。如果测试不使用，用 atacontrol 和 camcontrol 分离磁盘。
- 除非在测试网络，否则不要连接或配置网络。
- 除非测试需要数天才能完成，否则不要运行 NTPD。
- 尽量减少到串口或 VGA 控制台的输出。把输出写入内存盘上的文件会减少抖动。
- 确保测试足够长但不要太长。如果测试太短，时间戳会成为问题；如果太长，温度变化会毁掉它。经验法则：多于 1 分钟，少于 1 小时。
- 尽量保持机器周围温度稳定。温度变化会影响石英晶体、CPU 热管理和磁盘驱动器的寻道算法。
- 至少对”修改前”和”修改后”代码各运行 3 次，最好多得多，并用 ministat 分析结果。

（续下页）

## PHK 的常用测试计划

```sh
a[1] b[1]
   做一次合理性检查。
a[2] b[2] a[3] b[3]
   对 a[1:3] vs b[1:3] 运行 ministat
   如果没有差异，不要绝望。
a[4] a[5] b[4] b[5] a[6] a[7]
b[6] b[7] a[8] a[9] b[8] b[9]
   对 a[4,6,8] vs a[5,7,9] 以及 b[4,6,8] vs b[5,7,9] 运行 ministat
   如果存在差异，停下来思考。
   对 a[1:9] vs b[1:9] 运行 ministat
```

```sh
a[10] a[11] a[12] b[10] b[11] b[12]
a[13] a[14] a[15] b[13] b[14] b[15]
a[16] a[17] a[18] b[16] b[17] b[18]
   对 a[1:3] vs a[16:18] 以及 b[1:3] vs b[16:18] 运行 ministat
   如果存在差异，停下来思考。
   对 a[1:18] vs b[1:18] 运行 ministat
   这是你的最终结果；不太可能再改善多少。
```

[1] Michael Moore: Adventures in a TV Nation, p. 9. ISBN 0-330-41914-5
[2] Statistics of Extreme Events with Application to Climate Change. <https://fas.org/irp/agency/dod/jason/statistics.pdf>
[3] Walter A. Shewhart，统计质量控制之父，在 1926 年听说学生 T 检验时对其赞不绝口。<https://archive.org/details/bstj5-2-308>
[4] <http://www.larrygonick.com/titles/science/the-cartoon-guide-to-statistics/>
[5] <http://www.larrygonick.com/> 强烈推荐他的所有书，尤其是历史系列。
[6] <https://en.wikipedia.org/wiki/William_Sealy_Gosset>

---

> **POUL-HENNING KAMP** 是 <phk@FreeBSD.org>，曾是项目中最活跃的内核程序员之一。这些日子他的主要项目是 Varnish HTTP 缓存，但他的笔记本仍运行 FreeBSD-current。
