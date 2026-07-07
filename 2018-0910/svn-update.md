# SVN 动态

作者：Steven Kreuzer

我们已经进入了 12.0 发布周期！代码冻结已经开始，12.0-RELEASE 公告定于 11 月 13 日。很可能在你阅读本专栏时，releng/12.0 分支已经创建。

令我欣喜的是，越来越多的用户空间工具获得了 libxo 支持。我一直喜欢 Unix 命令行的地方在于能够将一个命令的输出管道到另一个命令进行一些操作，然后再发送到另一个命令进行最终处理。我最大的烦恼是所有这些输出并不遵循任何真正的”标准”，大多数时候你会发现自己必须解析文本块，查找某个字符串，从那里向下数几行，并执行其他纯粹疯狂的行为。libxo 通过使用一组通用的函数调用使生成文本、XML、JSON 和 HTML 输出变得容易，解决了这个问题。这是对代码树受欢迎的补充。

## pf：在通过规则失败时支持“return”语句

<https://svnweb.freebsd.org/changeset/base/335569>

通常 pf 规则预期做两件事之一：通过流量或阻止它。阻止可以是静默的——”drop”，或大声的——”return”、“return-rst”、“return-icmp”。然而还有第三类通过 pf 的流量：匹配”pass”规则但在应用规则时失败的数据包。这发生在重定向表为空或 src node 或状态创建失败时。这样的规则总是静默失败，不通知发送者。此更改允许用户也配置这种行为，以便 pf 在这些情况下返回错误数据包。

## 引入 arm64 linuxulator 存根

<https://svnweb.freebsd.org/changeset/base/335333>

这提供了 arm64 Linux vdso 和 machdep、ptrace 和 futex 的存根实现，足以执行 arm64 Linux 'hello world' 二进制文件。

## 使 UMA 和 malloc(9) 在大多数情况下返回不可执行内存

<https://svnweb.freebsd.org/changeset/base/335068>

启动后分配的大多数内核内存不需要可执行。有一些例外。例如，内核模块确实需要可执行内存，但它们不使用 UMA 或 `malloc`(9)。BPF JIT 编译器也需要可执行内存，直到 r317072 之前一直使用 `malloc`(9)。此更改使 `malloc`(9) 返回不可执行内存，除非指定了新的 `M_EXEC` 标志。此更改后，UMA zone(9) 分配器将始终返回不可执行内存，KASSERT 将捕获使用 `M_EXEC` 标志通过 `uma_zalloc()` 或其变体分配可执行内存的尝试。

## 将默认解释器翻转为 Lua

<https://svnweb.freebsd.org/changeset/base/338050>

经过多年的开发，lualoader 准备好首次亮相。两种风格的 loader 仍然默认构建，可以通过手动创建硬链接或使用 `build`(7) 中记录的 LOADER_DEFAULT_INTERP 安装为 **/boot/loader** 或 **/boot/loader.efi**。

## 为 lastlogin(8) 添加 libxo(3) 支持

<https://svnweb.freebsd.org/changeset/base/338353>

## 为 last(1) 添加 libxo(3) 支持

<https://svnweb.freebsd.org/changeset/base/338352>

## 通过记录最近绘制的字符以及前景和背景颜色来加速 vt(4)

<https://svnweb.freebsd.org/changeset/base/338316>

在 bitblt_text 函数中，将值与此缓存比较，如果字符未改变则不重绘。使显示失效时，清除此缓存以强制重绘字符；在挂起/恢复周期之间也强制完全重绘，否则可能出现异常伪影。滚动显示时（这是 vt 驱动程序中花费大部分时间的地方），如果大多数行比终端宽度短，性能会显著提升，因为这避免了在空白上重绘空白。在 c5.4xlarge EC2 实例（带有模拟文本模式 VGA）上，这将在内核启动期间在 vt(4) 中花费的时间从 1200 毫秒减少到 700 毫秒；在我的笔记本电脑上（带有 3200x1800 显示器），相应时间从 970 毫秒减少到 155 毫秒。

## 使 init 能够执行任何可执行文件，而不仅仅是 sh(1) 脚本

<https://svnweb.freebsd.org/changeset/base/337321>

这意味着用户应该能够例如用 Python 重写他们的 **/etc/rc**。

## 添加在 jail(8) 内运行 bhyve(8) 的能力

<https://svnweb.freebsd.org/changeset/base/337023>

此补丁添加了一个新的 `sysctl`(8) 旋钮 `security.jail.vmm_allowed`；默认情况下此选项被禁用。

## 将 loader(8) geli 支持扩展到所有架构和所有类磁盘设备

<https://svnweb.freebsd.org/changeset/base/336252>

这将大部分 geli 支持从 lib386/biosdisk.c 迁移到一个新的 geli/gelidev.c，后者实现了一个 devsw 类型设备，其 `dv_strategy()` 函数处理 geli 解密。对所有架构的支持来自于将尝试验证和附加代码迁移到 libsa 的 `devopen()` 函数。打开任何 DEVT_DISK 设备后，`devopen()` 调用新函数 `geli_probe_and_attach()`，它将通过创建 geli_devdesc 实例替换 open_file 中的 disk_devdesc 实例来”附加”geli 代码到 open_file 结构。这通过 geli 代码路由设备的所有 IO。添加了一个新的公共 `geli_add_key()` 函数，允许架构/供应商特定的代码添加从自定义硬件或其他来源获得的密钥。有了这些更改，geli 支持将编译到所有架构上 loader(8) 的所有变体中，因为默认是 WITH_LOADER_GELI。

## 添加 bhyve NVMe 设备仿真

<https://svnweb.freebsd.org/changeset/base/335974>

bhyve NVMe 设备仿真的初始工作由 GSoC 学生 Shunsuke Mie 完成，并由 Leon Dang 在性能、功能和客户机支持方面进行了大量修改。

## 在 GENERIC 和 MINIMAL 中启用 options NUMA

<https://svnweb.freebsd.org/changeset/base/338602>

## 将大多数命令的默认分页器切换为 less

<https://svnweb.freebsd.org/changeset/base/337497>

有时 less 就是 more，现在它在 base 中是一致的。

STEVEN KREUZER 是一名 FreeBSD 开发者和 Unix 系统管理员，对复古计算和风冷大众汽车感兴趣。他与妻子、女儿和狗住在纽约皇后区。

## 帮助创造未来。

加入 FreeBSD 项目！

FreeBSD 项目正在寻找

FreeBSD 在国际上被公认为提供高性能、安全、稳定操作系统的创新领导者。

不仅 FreeBSD 易于安装，而且它运行大量

通过查看我们的网站了解更多
下载软件

帮助开发这个健壮的操作系统。
加入我们！

已经参与了？
别忘了查看
freebsdfoundation.org 上的最新资助机会

The FreeBSD Community is proudly supported by
