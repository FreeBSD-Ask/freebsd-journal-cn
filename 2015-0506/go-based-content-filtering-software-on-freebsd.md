# FreeBSD 上基于 Go 的内容过滤软件

- 原文：[Go-based Content Filtering Software on FreeBSD](https://freebsdfoundation.org/our-work/journal/browser-based-edition/measure-twice-code-once/)
- 作者：**Ganbold Tsagaankhuu**、**Esbold Unurkhaan**、**Erdenebat Gantumur**

Go 是一门较新的编程语言——与 C、C++、Java 等许多编程语言相比而言。它具备许多实用特性，在多数情况下比其中一些语言更具生产力。而 FreeBSD 历史悠久，已证明是当今最可靠、最强大的操作系统。本文将讨论在 FreeBSD 上用 Go 开发软件的问题、优势、劣势与常见陷阱。我们选择内容过滤软件作为目标，并将项目命名为 Shuultuur。Shuultuur 是一个蒙古语词汇，意思是“过滤器”。

日常生活中，我们正见证着现代世界如何迈向物联网，互联的世界如何迅速扩张。这种转变促使我们从不同视角审视既有问题。对我们而言，内容过滤就是这些问题之一，抱着这一想法，我们开始探索可能性。

有无数由各类社区支持的开源项目。然而，有些项目在其生命周期中演进得并不理想，改进既有代码库变得困难。由于我们的内容过滤项目源于特定需求，我们决定从头开发。为节省数年的开发时间，我们需要具备足够的生产力，以匹配成熟内容过滤软件的功能与性能，并进一步超越。

## 动因

- 为什么需要内容过滤？首先，就内容过滤的主要目标而言，普遍的理解是对不良内容施加某种控制。这类解决方案在企业中被广泛用于执行计算机安全策略。图书馆、学校等公共机构使用内容过滤器保护儿童免受与其年龄不相宜的内容侵害——例如成人、暴力、毒品等内容。

- 编程语言的选择。问题之一是使用哪种编程语言？本质上，我们寻找的是一种快速、轻量、易于原型化，且以相对较少的精力就能产出并维护生产级代码的编程语言。因此，我们偏好静态类型、编译型、具备强类型系统的语言。虽然总有取舍，但更快达成目标的需求使上述特性成为必需。

Go 于五年前在 谷歌 正式发布（1）。它是一门编译型、静态类型、垃圾回收、非传统面向对象的通用系统编程语言。Go 产出原生二进制，编译速度极快，并在设计之初就考虑了并发。由于 Go 的特性与我们最初的需求相符，我们将其与其他候选语言一起仔细考量并开始实验。Go 原生二进制的性能与经典的低层语言 C 大致相当，远胜于解释型语言（5）。经过广泛的研究与试验，我们得出结论：Go 编程语言最适合我们的目标。

Go 不需要额外的库来处理并发，因为它已是编程语言特性的一部分，并且 Go 对多处理有强力支持。此外，与 C 相比，Go 是更具生产力的语言，并内置了多种有用的数据结构，如 map（23）与 slice（24）。尤其在处理并发时，借助 Go 特有的 goroutine（21）与 channel 等特性，许多先进的实用方案能轻松运用于现代硬件。goroutine 是一个与其他 goroutine 在同一地址空间中并发执行的函数。它轻量，并通过 channel（22）与其他 goroutine 通信。由于 Go 是一门非常简洁、垃圾回收、静态类型的语言，源代码可以写得错误更少，bug 也理应更少。此外，对速度与内存泄漏做性能分析相对容易，这在处理生产源代码时十分便利。语法上，它松散地派生自 C，并受到 Python（3）等其他语言的影响。另外，Go 拥有数量庞大的库（2），最后，Go 是 BSD 许可的完全开源语言（4）。

在内容过滤的语境下，准确检测一个句子的含义对自动化工具而言是项艰巨任务。如我们所知，人类在没有详细信息时也无法判断句子的真实含义。然而在现实世界中，内容过滤器试图基于字符串匹配技术，将 Web 内容分类为应屏蔽的不良内容或应放行的良好内容（6）。在多数语言中，精确字符串匹配技术（模式匹配）可能需要很高的处理能力（7）。出于这一原因，我们选择开发内容过滤软件来利用 Go 的性能。

- 为什么选 FreeBSD 作为平台？我们选择 FreeBSD 操作系统作为主要开发与测试平台，主要因为：

- 它是最强大、最成熟、最稳定的操作系统之一，拥有完整、可靠、自洽的发行版。

- FreeBSD 的网络栈坚实而快速（8）。

- 选择 FreeBSD 的优势之一是其 Port 与 package 系统，使安装和部署所需的应用程序和软件十分便捷。

- 存在 NanoBSD 等便利工具，可用于轻松制作定制的 FreeBSD 镜像。

- 最后，我们热爱 FreeBSD。

- 我们还使用了以下开源软件：

- goproxy 为 Go 提供可定制的 HTTP 代理库。它支持常规 HTTP 代理、通过 CONNECT 的 HTTPS、用“中间人”式攻击“劫持”HTTPS 连接。该代理的定位是在可承受相当流量的同时保持可定制与可编程（9）。

- gcvis 实时可视化 Go 程序的 gctrace 数据（10）。

- profile 是 Go 的简单性能分析支持包（11）。

- go-nude 用 Go 进行裸露检测（12）。

- xxhash-go 是 C xxhash 的 Go 封装——一种极快的哈希算法，速度接近内存极限（13）。

- powerwalk 是一个 Go 包，用于遍历文件并并发调用用户代码处理每个文件（14）。

- redigo 是 Redis 数据库的 Go 客户端（15）。

- Redis 是一款开源、BSD 许可的高级键值缓存与存储（16）。

## 挑战

开发过程中我们遇到了几个问题：

Shallalist 黑名单包含超过 180 万条 URL/域名条目。将它们存入内存具有挑战性，起初我们按以下方式将 URL/域名条目存入 Redis：

```go
...
// 将 URL/域名存储为键，类别存储为值
conn.Do("SET", urls_or_domain, category)
...
```

这在内存利用率和性能方面都不理想。经过一番研究，我们找到了将其缩减到约 4100 个哈希键的方法。我们用 Stephane Bunel 的 xxhash-go 计算每个 URL/域名的哈希并切片，然后将这些切片存入 Redis，类似于：

```go
// 使用 xxhash 从 URL/域名获取校验和
blob := []byte(url_or_domain)
h32g := xxh.GoChecksum32(blob)
/*
* 按以下方式将其作为哈希存储在 Redis 中：
* key = 0xXXXX（URL/域名的前半部分），
* field = XXXX（URL/域名的后半部分），
* value = category
*/
hash_str := fmt.Sprintf("0x%08x", h32g)
key := hash_str[0:6]
value := hash_str[6:]
conn.Do("HSET", key, value, category)
```

禁用词与加权词组查找问题：原先它们存储在 Redis 中，在循环中访问既慢又低效。我们用图与 map 改进了这一点。词组列表（禁用、加权等）中存在的每个词都是图的一条边，且在同一类别中应唯一。例如，我们有“sex woman”、“sex man”与“mature sex”这样的禁用词组。Shuultuur 创建四条边——“sex”、“woman”、“man”、“mature”、顶点。出于编程效率的考虑，边及其相关顶点存储在 map 中。Go 提供了实现哈希表的内置 map 类型。此外，我们用 Go 实现的 Boyer Moore 搜索算法替换了基于正则表达式的搜索算法。

将 HTTP 响应体读入字符串会因大量分配导致堆内存占用膨胀，尤其在每秒连接数较高时。理想情况下，这应使用利用 `io.Reader` 接口的流式解析器来处理。此外，对入站请求限制连接速率也是一种选择。我们通过 CPU 与内存性能分析（19）对其做了优化和改进。具体做法是在 Shuultuur 中启用内存分析，并使用 Go 内置的性能分析器 pprof。下面的报告展示了开发初期阶段的内存分配情况：

```sh
# go tool pprof --alloc_space ./shuultuur_mem /tmp/profile228392328/mem.pprof
Adjusting heap profiles for 1-in-4096 sampling rate
Welcome to pprof! For help, type 'help'.
(pprof) top15
Total: 11793.7 MB
3557.7 30.2% 30.2% 3557.7 30.2% runtime.convT2E
1212.1 10.3% 40.4% 1212.1 10.3% container/list.(*List).insertValue
832.3 7.1% 47.5% 2434.8 20.6% github.com/garyburd/redigo/redis.(*conn).readReply
807.9 6.9% 54.4% 1874.6 15.9% github.com/garyburd/redigo/redis.(*Pool).Get
673.8 5.7% 60.1% 673.8 5.7% github.com/garyburd/redigo/redis.Strings
544.5 4.6% 64.7% 549.4 4.7% main.regexBannedWordsGo
521.1 4.4% 69.1% 521.1 4.4% bufio.NewReaderSize
490.9 4.2% 73.3% 490.9 4.2% bufio.NewWriter
438.2 3.7% 77.0% 438.2 3.7% runtime.convT2I
369.8 3.1% 80.1% 7622.9 64.6% main.workerWeighted
255.0 2.2% 82.3% 255.9 2.2% main.regexWeightedWordsGo
235.5 2.0% 84.3% 235.5 2.0% bytes.makeSlice
229.9 1.9% 86.2% 397.1 3.4% io.Copy
168.3 1.4% 87.6% 168.3 1.4% github.com/garyburd/redigo/redis.String
162.6 1.4% 89.0% 4048.9 34.3% main.getHkeysLen
(pprof)
```

在这份初始报告中，你可以看到 `main.workerWeighted` 与 `main.getHkeysLen` 中有大量分配。这些函数用于通过 Redis 搜索禁用与加权词组。我们通过移除这些函数、做一些代码层面的优化并引入更好的算法改进了 Shuultuur。下面是完成上述改进后由同一命令生成的报告，我们认为仍有进一步改进的空间。

```sh
# go tool pprof --alloc_space ./shuultuur /tmp/profile287823990/mem.pprof
Adjusting heap profiles for 1-in-4096 sampling rate
Welcome to pprof! For help, type 'help'.
(pprof) top30
Total: 2156.3 MB
596.9 27.7% 27.7% 1066.4 49.5% io.Copy
406.3 18.8% 46.5% 406.3 18.8% compress/flate.NewReader
177.3 8.2% 54.7% 177.4 8.2% bytes.makeSlice
113.5 5.3% 60.0% 115.4 5.4% code.google.com/p/go.net/html. (*Tokenizer).Token
78.3 3.6% 63.6% 78.3 3.6% code.google.com/p/go.net/html.(*parser).addText
68.4 3.2% 66.8% 68.4 3.2% strings.Map
41.7 1.9% 77.2% 41.7 1.9% concatstring
37.7 1.7% 78.9% 736.6 34.2% main.ProcessResp
27.9 1.3% 80.2% 27.9 1.3% makemap_c
12.8 0.6% 91.8% 44.5 2.1% bitbucket.org/hooray-976/shuultuur/db.GraphBuild
12.5 0.6% 92.4% 12.5 0.6% strings.genSplit
10.7 0.5% 92.9% 595.5 27.6% main.getContentFromHtml
```

下面是开发初期展示 CPU 使用情况的 top 报告，使用率非常高。

```sh
lastpid: 1189; load averages: 7.30, 2.42, 0.93 up 0+00:30:51 14:57:41
61 processes: 1 running, 60 sleeping
CPU: 20.5% user, 0.0% nice, 42.0% system, 6.6% interrupt, 31.0% idle
Mem: 104M Active, 63M Inact, 225M Wired, 234M Buf, 7502M Free
Swap: 16G Total, 16G Free
PID USERNAME THR PRI NICE SIZE RES STATE C TIME WCPU COMMAND
1131 tsgan 22 52 0 182M 46196K uwait 4 9:29 685.50% shuultuur
900 redis 3 52 0 69952K 42512K uwait 6 1:11 88.48% redis-server
1130 tsgan 6 20 0 37856K 9084K piperd 1 0:01 0.00% gcvis
918 tsgan 1 20 0 72136K 5832K select 5 0:00 0.00% sshd
889 squid 1 20 0 70952K 16412K kqread 5 0:00 0.00% squid
1049 tsgan 1 20 0 38388K 5168K select 11 0:00 0.00% ssh
998 tsgan 1 20 0 72136K 5904K select 9 0:00 0.00% sshd
919 tsgan 1 20 0 17564K 3528K pause 2 0:00 0.00% csh
868 root 1 20 0 22256K 3284K select 11 0:00 0.00% ntpd
```

如你所见，下面的 top 报告显示优化禁用与加权词组搜索后 CPU 使用率大幅降低。

```sh
lastpid: 1253; load averages: 0.15, 0.31, 0.32 up 0+00:55:22 11:55:42
45 processes: 1 running, 44 sleeping
CPU: 1.4% user, 0.0% nice, 0.0% system, 0.0% interrupt, 98.6% idle
Mem: 96M Active, 72M Inact, 279M Wired, 310M Buf, 7445M Free
Swap: 16G Total, 16G Free
PID USERNAME THR PRI NICE SIZE RES STATE C TIME WCPU COMMAND
1183 root 17 20 0 142M 37348K uwait 0 7:28 14.31% shuultuur
896 redis 3 52 0 78144K 62896K uwait 3 0:52 0.00% redis-server
1182 root 6 20 0 45048K 16840K uwait 9 0:16 0.00% gcvis
993 tsgan 1 20 0 72136K 6744K select 9 0:06 0.00% sshd
1187 tsgan 1 20 0 9948K 1600K kqread 10 0:03 0.00% tail
1091 tsgan 1 20 0 16596K 2548K CPU8 8 0:02 0.00% top
1204 tsgan 1 20 0 38388K 5164K select 5 0:00 0.00% ssh
1196 tsgan 1 20 0 72136K 5904K select 1 0:00 0.00% sshd
885 squid 1 20 0 70952K 16384K kqread 0 0:00 0.00% squid
```

我们实施了若干其他改进，例如学习 URL/域名，从而不必每次都在 HTTP 响应体中检查禁用与加权词组。学习模式特性按以下方式加入：

```go
// 学习并将此 URL 临时存储到 redisdb
// 使用 xxhash 从 URL/域名获取校验和
blob1 := []byte(requrl)
h32g := xxh.GoChecksum32(blob1)
// key = 0xXXXXXXXX，持续 expire_time 秒，1 表示 BLOCK，
// 2 表示 PASS
key := fmt.Sprintf("%s0x%08x", policy, h32g)
// SET key value [EX seconds] [PX milliseconds] [NX|XX]
db.Exec("SET", key, BLOCK, "EX", EXPIRE_TIME, "NX")
```

入站请求的速率限制再次借助 Redis 实现，如下：

```go
// 通过配置文件设置此项
limit := 10
// 为请求递增计数器
// （如果键不存在则会创建新键）
current, err := redis.Int(db.Incr(url_path + remote_addr))
// 检查返回的计数器是否超过限制
if current > limit {
fmt.Println(">>> Too many requests -", url_path + remote_addr)
response := goproxy.NewResponse(request,
goproxy.ContentTypeHtml, 429, "Too many requests!")
return request, response
} else if current == 1 {
fmt.Println(">>> SET counter for:", url_path + remote_addr)
// 为给定的 url_path 与 remote address 上的新计数器设置过期时间
db.Exec("SETEX", url_path + remote_addr, 1, 1)
}
```

另一项改进是增加将监听器限制为指定数量并发连接的能力：

```go
type Server struct {
*http.Server
// 限制未完成的请求数量
ListenLimit int
}
func (srv *Server) ListenAndServe() error {
l, err := net.Listen("tcp", addr)
l = netutil.LimitListener(l, srv.ListenLimit)
return srv.Serve(l)
}
if LISTEN_LIMIT_ENABLE == 1 {
srv := &Server {
ListenLimit: LISTEN_LIMIT,
Server: &http.Server{Addr: ":8080", Handler: proxy},
}
log.Fatal(srv.ListenAndServe())
} else {
log.Fatal(http.ListenAndServe(":8080", proxy)
```

HTTP 响应中慢速的图像过滤暂时禁用，直到我们找到合适的解决方案。

最后一个主要问题可能与高负载下大量 goroutine 有关，这会导致 CPU 与内存使用率高。目前我们正在调查该问题（17）。

## 基准测试结果

### 案例 1

为比较我们实现与现有方案的性能，我们使用了 Dansguardian-2.12.0.3，并在同一环境下测试。我们知道 Dansguardian 通常与 squid 配合使用，因此测试中使用了 **3.4.8_2** 版本的 squid。我们的内容过滤软件 Shuultuur 用 Go 1.3.2 编写，运行在 FreeBSD/amd64 上。我们使用同一台服务器进行性能测试对比，互联网链路速度为 5Mbps。服务器的技术规格如下：

- CPU - Intel(R) Xeon(R) X5670 2.93GHz
- 内存 - 8192MB
- FreeBSD/SMP - 12 CPUs（package(s) x 6 core(s) x 2 SMT threads）

我们使用 FreeBSD 9.2-RELEASE，**/etc/sysctl.conf** 包含以下内容：

```sh
kern.ipc.somaxconn = 27737
kern.maxfiles = 123280
kern.maxfilesperproc = 110950
kern.ipc.maxsockets = 85600
kern.ipc.nmbclusters = 262144
net.inet.tcp.maxtcptw = 47120
```

我们还得在 Redis 配置文件中将 tcp-backlog 设置改为较高值。此外，我们用 `http_load-14aug2014`（并行与速率测试）（18）对 Dansguardian 与 Shuultuur 都做了 HTTP 负载测试。`http_load` 测试中使用了以下 URL：

- <http://fxr.watson.org/fxr/source/arm/lpc/lpc_dmac.c>
- <http://www.news.mn/news.shtml>
- <http://mongolian-it.blogspot.com/>
- <http://www.patrick-wied.at/static/nudejs/demo/>
- <http://news.gogo.mn/>
- <http://www.amazon.com/>
- <http://edition.cnn.com/?refresh=1>
- <http://www.uefa.com/>
- <http://www.tmall.com/>
- <http://www.reddit.com/r/aww.json>
- <http://nginx.com>
- <http://www.yahoo.com>
- <http://slashdot.org/?nobeta=1>
- <http://www.ikon.mn>
- <http://www.gutenberg.org>
- <http://en.wikipedia.org/wiki/BDSM>
- <http://www3.nd.edu/~dpettifo/tutorials/testBAD.html>
- <http://penthouse.com/#cover_new?{}>
- <http://www.playboy.com>
- <http://www.bbc.com/earth/story/20141020-chicks-tumble-of-terror-filmed>
- <http://173.244.215.173/go/indexb.html>
- <http://breakingtoonsluts.tumblr.com/>

上述 URL 中，有的列在 Shallalist 黑名单中，有的包含禁用与加权词组列表中的词组，有的包含大量内容与 javascript，其余 URL 的选择并无特殊原因。HTTP 负载测试使用了以下测试命令：

```sh
./http_load -proxy 172.16.2.1:8080 -parallel 10 -seconds 600 urls
./http_load -proxy 172.16.2.1:8080 -rate 10 -jitter -seconds 600 urls
```

第一条命令中的 `-parallel` 选项表示要建立并维持的并发连接数，第二条命令中的 `-rate` 选项控制每秒发出的请求数，`-jitter` 选项让速率变化约 10%，`-seconds` 选项表示测试运行的秒数。

基于上述结果，Shuultuur 有其优势与劣势。例如，由于 Shuultuur 仍在开发中，它比 Dansguardian 更频繁地返回 Internal Server Error (500)。另一方面，Shuultuur 返回的成功响应 (200) 多得多。Dansguardian 有一些限制，它返回了 341 次 Service Unavailable (503)，超时也更多。在性能方面，平均而言，两项测试中 Shuultuur 的性能在多数情况下都高于 Dansguardian。

### 案例 2

场景与案例 1 几乎相同，但我们使用了不同的硬件（APU 系统主板）（20），将 Go 升级到 1.4.1，并将互联网链路速度改为 2Mbps。

硬件的技术规格如下：

- CPU – AMD G 系列 T40E，1 GHz 双核 Bobcat，支持 64 位，每核 32K 数据 + 32K 指令 + 512K L2 缓存
- 内存 - 4096MB

在 APU 上，我们使用 FreeBSD 10.1-RELEASE，**/etc/sysctl.conf** 包含以下内容：

```sh
kern.ipc.somaxconn = 4096
kern.maxfiles = 10000
kern.maxfilesperproc = 8500
kern.ipc.maxsockets = 6500
kern.ipc.nmbclusters = 20000
net.inet.tcp.maxtcptw = 4000
```

由于硬件较小，我们得在 Redis 配置文件中将 tcp-backlog 设置改为 4096。本案例中，我们也用 `http_load-03feb2015`（并行与速率测试）（18）对 Dansguardian 与 Shuultuur 做了 HTTP 负载测试。

测试结果与在服务器上所做的测试观察相似。与之前的测试一样，两项测试中 Shuultuur 的性能在多数情况下都高于 Dansguardian。图 3 与图 4 分别展示了 Shuultuur 在速率与并行测试中 `http_load` 测试的内存使用情况。

测试期间，我们抓取了 Shuultuur 与 Dansguardian 的 top 报告，如下所示。

SHUULTUUR：

```sh
lastpid: 1317; load averages: 1.52, 1.00, 0.58
71 processes: 1 running, 64 sleeping, 6 stopped
CPU: 31.4% user, 0.0% nice, 5.9% system, 1.6% interrupt, 61.2% idle
Mem: 58M Active, 189M Inact, 158M Wired, 70M Buf, 3519M Free
Swap: 978M Total, 978M Free
PID USERNAME THR PRI NICE SIZE RES STATE C TIME WCPU COMMAND
1300 user 18 25 0 84540K 43672K uwait 1 6:16 91.85% shuultuur
1299 user 5 21 0 28544K 9484K piperd 1 0:18 4.10% gcvis
822 redis 3 52 0 28108K 6540K uwait 1 0:21 0.29% redis-server
1024 root 1 20 0 43580K 17092K select 0 3:42 0.00% dansguardian
794 squid 1 20 0 164M 68400K kqread 1 1:20 0.00% squid
1030 nobody 1 20 0 43580K 18660K select 1 0:02 0.00% dansguardian
1028 nobody 1 20 0 43580K 18664K select 1 0:02 0.00% dansguardian
1029 nobody 1 20 0 43580K 18672K select 1 0:02 0.00% dansguardian
1033 nobody 1 20 0 43580K 18664K select 0 0:02 0.00% dansguardian
1032 nobody 1 20 0 43580K 18660K select 0 0:02 0.00% dansguardian
1031 nobody 1 20 0 43580K 18672K select 1 0:02 0.00% dansguardian
1025 nobody 1 20 0 31292K 5328K select 1 0:02 0.00% dansguardian
```

DANSGUARDIAN：

```sh
lastpid: 1151; load averages: 0.42, 0.68, 0.81
up 0+01:00:20 22:20:41
156 processes: 1 running, 152 sleeping, 3 stopped
CPU: 0.2% user, 0.0% nice, 10.2% system, 1.8% interrupt, 87.8% idle
Mem: 103M Active, 245M Inact, 161M Wired, 58M Buf, 3415M Free
Swap: 978M Total, 978M Free
PID USERNAME THR PRI NICE SIZE RES STATE C TIME WCPU COMMAND
1024 root 1 35 0 43580K 17092K nanslp 0 1:13 23.49% dansguardian
794 squid 1 26 0 160M 62060K kqread 0 0:13 4.59% squid
1002 user 19 42 0 93636K 51320K STOP 0 9:58 0.00% shuultuur
1001 user 6 20 0 33856K 10692K STOP 0 0:32 0.00% gcvis
822 redis 3 52 0 28108K 6452K uwait 1 0:15 0.00% redis-server
932 user 1 20 0 21916K 3244K CPU0 0 0:06 0.00% top
1028 nobody 1 20 0 43580K 18152K select 0 0:01 0.00% dansguardian
1033 nobody 1 20 0 43580K 18172K select 0 0:01 0.00% dansguardian
926 user 1 20 0 86472K 7240K select 1 0:01 0.00% sshd
1025 nobody 1 20 0 31292K 5328K select 1 0:00 0.00% dansguardian
1030 nobody 1 20 0 43580K 18304K select 0 0:00 0.00% dansguardian
1053 nobody 1 20 0 43580K 18664K select 0 0:00 0.00% dansguardian
```

如你所见，Shuultuur 工作时系统平均负载，尤其是 CPU 使用率较高。

## 结论与未来工作

用 Go 开发应用，使用其内置数据结构如 map 与 slice，简单且大多直截了当。我们在几天内就做出了首个可工作的原型。GitHub 等在线源代码仓库中有许多用 Go 编写的开源项目，其中许多对我们的开发很有帮助。

测试结果仅针对两个案例。到目前为止，我们多次进行 HTTP 负载测试，结果一致。我们预期达到首个稳定版本时，结果会好得多。

如前所述，我们的实现缺乏快速稳定的图像检查功能。在未来的工作中，我们将改进图像检查，并必须解决大量 goroutine 的问题。最后，内存使用率与 CPU 负载对嵌入式系统应用是主要问题，我们计划就此做更多研究以稳定资源占用。

**Ganbold Tsagaankhuu** 是一名自由职业者，从事各种与 FreeBSD 相关的项目。他也在蒙古推广类 Unix 操作系统与开源。他是蒙古 Unix 用户组的创始人之一。1994 年毕业于新西伯利亚国立技术大学（俄罗斯），获硕士学位。他将 FreeBSD 手册翻译成蒙古文，并于 2007 年起为 FreeBSD 项目贡献。2009 年 4 月至 2014 年 7 月，他在当地一家移动运营商工作，负责一个开发软件、管理服务器并改进公司安全的 IT 部门。

**Esbold Unurkhaan** 是蒙古科技大学（MUST）信息与通信技术学院（SICT）网络与系统安全讲师，位于乌兰巴托。2005 年获杜伊斯堡-埃森大学（德国）博士学位。2001 至 2004 年，他在杜伊斯堡-埃森大学实验数学研究所任研究员，攻读博士学位，论文题为《Secure End-to-End Transport over SCTP–A new security extension for SCTP》。在德国的研究工作结束后，他回到蒙古，在 MUST 的计算机科学与管理学院工作。

**Erdenebat Gantumur** 是 ESCRYPT Inc. 的安全工程师。2010 年获卡内基梅隆大学信息技术与信息安全理学硕士学位。他在安全领域拥有 10 年以上经验，涵盖网络安全、信息保障、计算机安全与嵌入式数据安全。自 2011 年起，他在 ESCRYPT Inc. 主导并参与多项 V2X、车载与智能电网安全项目。此前，他曾任网络工程师、系统与网络管理员、信息安全管理员。他还共同创立了蒙古首个网络事件响应团队，并担任网络安全分析师。

## 参考文献

1. Half a decade with Go. Retrieved from The Go blog: <http://blog.golang.org/5years>
2. Retrieved from Go language resources: <http://go-lang.cat-v.org/pure-go-libs>
3. Go (programming language). Retrieved from Wikipedia: <http://en.wikipedia.org/wiki/Go_%28programming_language%29>
4. The Go programming language. Retrieved from <http://golang.org/>
5. Computer Language Benchmarks Game. Retrieved from <http://benchmarksgame.alioth.debian.org/>
6. String Matching Algorithms and Their Applicability in Various Applications. (D. G. Nimisha Singla, Ed.) International Journal of Soft Computing and Engineering (IJSCE), 1 (6).
7. Fast Cache for Your Text: Accelerating Exact Pattern Matching with Feed-Forward Bloom Filters. School of Computer Science. Pittsburgh, Pennsylvania, USA: Carnegie Mellon University.
8. FreeBSD. Retrieved from <https://www.freebsd.org/internet.html>
9. goproxy. (E. Leibovich, Producer) Retrieved from <https://github.com/elazarl/goproxy>
10. gcvis. (D. Cheney) Retrieved from <https://github.com/davecheney/gcvis>
11. profile. (D. Cheney) Retrieved from <https://github.com/davecheney/profile>
12. go-nude. (Koyachi) Retrieved from <https://github.com/koyachi/go-nude>
13. xxxhash-go. (S. Bunel) Retrieved from <https://bitbucket.org/StephaneBunel/xxhash-go>
14. powerwalk. (Stretchr) Retrieved from <https://github.com/stretchr/powerwalk>
15. redigo. (G. Burd) Retrieved from <https://github.com/garyburd/redigo>
16. Redis. Retrieved from <http://redis.io>
17. HTTP ListenAndServe Goroutines throughput. Retrieved from <http://grokbase.com/t/gg/golang-nuts/147b9nb2nq/go-nuts-http-listenandserve-goroutines-throughput>
18. HTTP Load. (ACME Lab) Retrieved from <http://acme.com/software/http_load/>
19. Profiling Go programs. Retrieved from <https://blog.golang.org/profiling-go-programs>
20. PC engine APU board. <http://www.pcengines.ch/apu1d4.htm>
21. Goroutines. Retrieved from <https://golang.org/doc/effective_go.html#goroutines>
22. Channels. Retrieved from <https://golang.org/doc/effective_go.html#channels>
23. Go maps in action. Retrieved from <https://blog.golang.org/go-maps-in-action>
24. Arrays, slices (and strings): The mechanics of 'append'. Retrieved from <https://blog.golang.org/slices>
