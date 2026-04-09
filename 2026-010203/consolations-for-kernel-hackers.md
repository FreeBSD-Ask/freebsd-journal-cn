# 内核黑客的慰藉

- 原文：[Consolations for Kernel Hackers](https://freebsdfoundation.org/our-work/journal/browser-based-edition/laptop-desktop/consolations-for-kernel-hackers/)
- 作者：**Tom Jones**

打开机器，嗡嗡作响，发出一声鸣响。显示屏跳转到一个明亮但黑屏的界面，这是第一个迹象，表明这些声音不仅仅是遥远的运动。若干来自 Phoenix 或 American 的文本滚动而过，然后你看到了 FreeBSD 启动提示符的奇迹。10 秒后，倒计时和文本开始滚动。对一些人来说，这是一台原始的机器，但对我们许多人来说，这是最好的部分。在开始时接近硬件。

内核消息开始滚动，从 FreeBSD 版权信息开始，接着是设备被发现和连接的过程，直到内核将控制权交给用户空间。文本的外观变化表明了这一点，我们开始看到服务启动。所有内核黑客都曾坐下来看着这些文本滚动，寻找他们添加到新驱动程序或旧代码中的消息，这些代码的行为与我们所有人的预期不符。我们等待着调用我们的 printf，证明我们已经构建并运行了带有我们更改的内核，证明我们终于驯服了构建过程，可以开始真正的工作。

所有开发人员都使用 printf 调试和打印语句来跟踪执行流程，弄清楚事情如何或是否工作，以及它们在哪里连接。大多数时候，内核的这个核心部分运行得完美无缺；我们得到文本输出，或者内核在显示之前就已经死亡。这个影院由一个控制台驱动程序支持，这是一段位于 printf 底部的代码，位于硬件和软件之间的接口。如果你正忙于开发控制台驱动程序或处于非常早期的启动阶段，你想知道控制台通常如何工作，这样你就可以让它异常工作，并获得一些你可能无法以其他方式获得的调试信息。

## 终端和控制台

我们现在使用的机器与设计 UNIX 时所围绕的机器有很大不同，这意味着系统控制台、访问终端和图形输出之间的界限不是很清晰。在过去，系统会配备一个专用的系统控制台，它位于计算机所在的机房内或附近。终端可以位于其他更安静的地方，要么在终端室，要么在某个幸运者的办公桌上。

然而，控制台是机器的核心；如果系统需要向操作员报告令人兴奋或危险的故障，它会在控制台上弹出。控制台是系统的核心组件；大多数时候，它在聊天，在图形输出上打印消息，这些消息要么未连接，要么输出到没有连接电缆的串行端口。大多数时候，这些消息未被使用。它们是内核在发现、检测和配置设备时输出的启动文本。在运行时，这些消息指示硬件和系统错误，并通知我们缓冲区溢出和用尽的限制。当我们需要查看这些消息时，我们可以使用 dmesg 命令提取缓冲区。通常，这就是所有人真正需要的。

但内核黑客是一群奇怪的人，可悲的是，大多数时候，printf 调试仍然是追踪错误和从新驱动程序获得生命迹象的最佳方式。如果你分心并开始深入研究 printf 在内核中究竟如何工作，你最终会开始挖掘 cn\* 函数和控制台驱动程序的调度表。

## 从 printf 深入底层

对于大多数开发人员来说，内核代码是底层的，但我们今天的最高点是任何可以调用 printf(9) 的代码。让我们从 printf 向下走，穿过一些层，以了解在调用控制台驱动程序之前发生了什么。printf(9) 让我们看到通常是最简单的情况。printf(9) 及其友函数在 sys/kern/subr\_prf.c 中实现；这些函数主要是围绕 \_vprintf 的包装器，后者还处理输出任何缓冲的消息并通过 kvprintf 将字节放到控制台设备上。

```c
static int
_vprintf(int level, int flags, const char *fmt, va_list ap)
{
struct putchar_arg pca;
int retval;
#ifdef PRINTF_BUFR_SIZE
char bufr[PRINTF_BUFR_SIZE];
#endif
TSENTER();
pca.tty = NULL;
pca.pri = level;
pca.flags = flags;
#ifdef PRINTF_BUFR_SIZE
pca.p_bufr = bufr;
pca.p_next = pca.p_bufr;
pca.n_bufr = sizeof(bufr);
pca.remain = sizeof(bufr);
*pca.p_next = '\0';
#else
/* 不要缓冲控制台输出。 */
pca.p_bufr = NULL;
#endif
retval = kvprintf(fmt, putchar, &pca, 10, ap);
#ifdef PRINTF_BUFR_SIZE
/* 写入任何缓冲的控制台/日志输出： */
if (*pca.p_bufr != '\0')
prf_putbuf(pca.p_bufr, flags, level);
#endif
TSEXIT();
return (retval);
}
```

它解析格式字符串，这会解析为对 putchar() 的调用。putchar 调用 cnput\_char，最后我们调用驱动程序调度表进行驱动程序特定的调用。kvprintf 接受一个函数指针，用于实际输出字节。

```c
int
kvprintf(char const *fmt, void (*func)(int, void*), void *arg, int radix, va_list ap)
{
#define PCHAR(c) {int cc=(c); if (func) (*func)(cc,arg); else *d++ = cc; retval++; }
char nbuf[MAXNBUF];
char *d;
const char *p, *percent, *q;
u_char *up;
int ch, n, sign;
uintmax_t num;
```

在我们上面的例子中，函数点指向 putchar

```c
/*
 * 在控制台或用户终端上打印一个字符。如果目标是
 * 控制台，则最后一堆字符保存在 msgbuf 中，以便
 * 以后检查。
 */
static void
putchar(int c, void *arg)
{
struct putchar_arg *ap = (struct putchar_arg*) arg;
struct tty *tp = ap->tty;
int flags = ap->flags;

/* 不要在 panic 后或在 ddb 中使用 tty 代码。 */
if (kdb_active) {
      if (c != '\0')
             cnputc(c);
      return;
}
if ((flags & TOTTY) && tp != NULL && !KERNEL_PANICKED())
       tty_putchar(tp, c);
if ((flags & TOCONS) && cn_mute) {
       flags &= ~TOCONS;
       ap->flags = flags;
 }
 if ((flags & (TOCONS | TOLOG)) && c != '\0')
       putbuf(c, ap);
}
```

cnputc 位于内核的底部，有许多路径通向它。

```c
void
cnputc(int c)
{
struct cn_device *cnd;
struct consdev *cn;
const char *cp;
if (cn_mute || c == '\0')
return;
(2)
#ifdef EARLY_PRINTF
if (early_putc != NULL) {
if (c == '\n')
early_putc('\r');
early_putc(c);
return;
}
#endif
(1)    STAILQ_FOREACH(cnd, &cn_devlist, cnd_next) {
cn = cnd->cnd_cn;
if (!kdb_active || !(cn->cn_flags & CN_FLAG_NODEBUG)) {
if (c == '\n')
cn->cn_ops->cn_putc(cn, '\r');
cn->cn_ops->cn_putc(cn, c);
}
}
if (console_pausing && c == '\n' && !kdb_active) {
for (cp = console_pausestr; *cp != '\0'; cp++)
cnputc(*cp);
cngrab();
if (cngetc() == '.')
console_pausing = false;
cnungrab();
cnputc('\r');
for (cp = console_pausestr; *cp != '\0'; cp++)
cnputc(' ');
cnputc('\r');
}
}
```

cnputc 有两个重要的代码部分，循环中的代码 (1) 遍历连接的控制台列表并调用控制台的 cn\_putc 回调。由于与调试器中的锁定相关的原因，如果内核调试器 kdb 处于活动状态，它不会调用这些方法。

这是 FreeBSD 实现对多个控制台的支持的地方。如果你曾经想知道如何在图形显示和串行端口上获得玻璃终端，那就是在这里发生的。代码 (2)，用 #ifdef EARLY\_PRINTF 包裹，是非常平台特定的，启用了来自内核映像第一个字节的输出。如果在你的硬件上定义并支持，你可以从加载器交接中检索字节。

作为 early\_putc 如何平台特定的一个例子，这里是 x86/amd64 实现。

```c
#if CHECK_EARLY_PRINTF(ns8250)
#if !(defined(__amd64__) || defined(__i386__))
#error ns8250 early putc is x86 specific as it uses inb/outb
#endif
static void
uart_ns8250_early_putc(int c)
{
u_int stat = UART_NS8250_EARLY_PORT + REG_LSR;
u_int tx = UART_NS8250_EARLY_PORT + REG_DATA;
int limit = 10000; /* 10ms 足够了 */
while ((inb(stat) & LSR_THRE) == 0 && --limit > 0)
continue;
outb(tx, c);
}
early_putc_t *early_putc = uart_ns8250_early_putc;
#endif /* EARLY_PRINTF */
```

这个 early\_putc 使用 x86 outb 指令在串行端口上传输字节，但它从未获得过对现在相当常见的 MMIO（内存映射 IO）UART 的支持。

（在我写这篇文章时，arm64 上的 MMIO UART 支持已添加。）

## 控制台生命周期

要查看控制台生命周期，最好从 FreeBSD 加载器开始。启动 FreeBSD 需要两个操作系统组件：内核和我们的引导加载器（loader(8)）。所有硬件平台都以某种固件开始。在旧的 x86 系统中，这是寻找引导块的 bios。现代 x86 系统通过固件启动，该固件提供 EFI 接口。非 PC 系统以不同的方式启动。在嵌入式设备中，片上系统上的引导 ROM 测试并尝试不同的引导介质；第一个有效的介质被使用。在设备上，这可能意味着引导 ROM 按以下顺序尝试：

* NAND 闪存
* SD 卡
* USB 介质
* USB 固件模式

在许多情况下，引导固件会通过那个可靠的串行接口记录它正在尝试的内容。通常，在 ARM 和 RISC-V 系统上，会运行第二阶段引导加载器，称为二级程序加载器或 SPL。对我们来说，通常就是 u-boot。早期固件只会打印到串行端口，但 u-boot 在许多情况下有足够的驱动程序，可以在可能的情况下使用帧缓冲设备。

Loader 是模块化的，根据基础平台以多种不同形式运行；我们从 u-boot、作为 EFI 应用程序、在开放固件之上以及在其他地方运行。对于 bhyve 虚拟机映像，FreeBSD 加载器作为用户空间应用程序运行。这些平台中的大多数使用静态控制台作为默认输出方法，并且如果可能的话，可以配置为使用额外的输出。

FreeBSD 的加载器作为链式应用程序或子应用程序在固件或 u-boot 之上运行。U-boot 将运行 EFI 应用程序，这使得创建可用的 FreeBSD 引导介质变得更加容易。在没有任何其他配置的情况下，FreeBSD 的加载器确定使用什么作为系统控制台（消息去向的地方）。在 EFI 系统上，我们将使用 EFI 控制台，它具有有时使用错误的 EFI 接口来输出字符的不幸特性。如果你曾经在 EFI 系统的串行端口上使用过 FreeBSD 加载器，你可能会在串行端口上看到重复的字节 —— 这是由于在通常的 EFI 控制台输出和 EFI 串行控制台输出中使用的 EFI 输出例程。

Loader 可以通过配置文件或从命令行配置为使用多个控制台，方法是向 consoles 环境变量添加更多控制台。

```sh
> set console="efi comconsole"
```

Loader 通过内核映像中的配置信息以及一些变量将配置信息传递给内核。控制台类型和是否使用多个控制台由 boothow 变量控制，该变量设置为 RB\_* 值。历史上（整个世界都是 VAX！），这些是 reboot(9) 的系统调用参数，但它们已经被重用并随着时间的推移变得交织在一起。从加载器，我们经常得到 RB\_SERIAL（使用串行端口）、RB\_MULTIPLE（使用多个控制台）、RB\_SINGLE（引导到单用户）和 RB\_VERBOSE，后者在内核中设置 bootverbose 并填满你的消息缓冲区。

## 启动内核

最终，加载器将启动选定的内核并传递参数和环境。加载器所做的一切都是临时的，所有工作都必须在内核中再次完成，以获得内核控制的工作系统。控制台初始化与平台特定的硬件密切相关。每个 FreeBSD 平台都有自己的机器相关代码，在系统可以执行机器无关的工作之前运行并设置内存和设备。

在大多数平台上，是一个 init\* 调用（如 initarm、init386、initriscv 或令人沮丧的不匹配的 powerpc\_init），但在 amd64 上，这是 hammer\_time。这些例程中的每一个都可能尝试初始化控制台两次（目前只有 i386 和 amd64 这样做），基于来自加载器的配置。hammer\_time 的相关部分如下所示：

```c
/*
  * 控制台和 kdb 应该更早初始化，但
  * 一些控制台驱动程序直到 getmemsize() 之后才工作。
  * 默认使用后期控制台初始化来支持这些驱动程序。
  * 这主要损失了 getmemsize() 中的 printf() 和早期调试。
  */
  TUNABLE_INT_FETCH("debug.late_console", &late_console);
  if (!late_console) {
         cninit();
         amd64_kdb_init();
  }

  getmemsize(physfree);
  init_param2(physmem);

 / 现在运行在新的页表上，已配置，u/iom 可访问 /
```

```c
#ifdef DEV_PCI
/* 此调用可能会调整 phys_avail[]。 */
pci_early_quirks();
#endif

if (late_console)
        cninit();
```

你可以从示例中看到，根据 late\_console 变量的值（默认为 true），可能会尝试调用 cninit 两次。在这两次调用之间是系统内存的发现和分配。在这些对 cninit 的调用之前，无法向系统控制台输出任何内容（它不存在！），cninit 之前的任何 printf 都会蒸发到以太空中。

例外情况是如果定义了 eary\_putc 如所述，那么你可以进行一些早期调试。cninit 遍历静态定义的控制台列表并尝试找到最佳可能的控制台。

```c
/*
  * 找到第一个优先级最高的控制台。
  */
 best_cn = NULL;
 SET_FOREACH(list, cons_set) {
     cn = *list;
     cnremove(cn);
     /* 跳过 cons_consdev。 */
     if (cn->cn_ops == NULL)
           continue;
     cn->cn_ops->cn_probe(cn);
     if (cn->cn_pri == CN_DEAD)
           continue;
     if (best_cn == NULL || cn->cn_pri > best_cn->cn_pri)
           best_cn = cn;
     if (boothowto & RB_MULTIPLE) {
           /*
            * 初始化控制台，并附加到它。
            */
           cn->cn_ops->cn_init(cn);
           cnadd(cn);
     }
 }
 if (best_cn == NULL)
       return;
 if ((boothowto & RB_MULTIPLE) == 0) {
       best_cn->cn_ops->cn_init(best_cn);
       cnadd(best_cn);
 }
 if (boothowto & RB_PAUSE)
       console_pausing = true;
 /*
  * 使最佳控制台成为首选控制台。
  */
 cnselect(best_cn);
```

```c
#ifdef EARLY_PRINTF
/*
* 释放早期控制台。
*/
early_putc = NULL;
#endif
```

控制台按控制台驱动程序在其 cn\_probe 函数被调用时设置的优先级排序。最后，我们必须有一个控制台（总是有一个默认值），我们释放 early\_putc，以便它永远消失。我没有在 loader.efi 中显示匹配的代码，但它非常相似。对于任何黑客 EFI 控制台的人来说，重要的例外是缺乏 early\_putc 机制，无法给你一些最后的机会进行实际的 printf 调试。然而，这可以被黑客解决。在 EFI 环境中，输出例程是可用的，如果你在进行初始化时只是保留一个控制台，你可以更清楚地了解正在发生的事情。

## 可用的控制台

系统控制台在 kernel 构建时定义。在上面的 cninit 中，我们遍历 cons\_set —— 由构建过程为我们生成的链接器集。每个控制台不是作为普通设备驱动程序声明的；相反，它使用 CONSOLE\_DRIVER 宏定义，该宏将控制台添加到链接器集。从 16-CURRENT 开始，链接器集不会在内核模块加载时填充，这意味着你不能在运行时动态向内核添加控制台驱动程序。

CONSOLE\_DRIVER 宏采用要添加到链接器集的控制台名称，并根据名称处理所有回调的创建：

CONSOLE\_DRIVER 位于 sys/sys/cons.h

```c
#define CONSOLE_DEVICE(name, ops, arg)                                  \
static struct consdev name = {                      \
.cn_ops = &ops,                     \
.cn_arg = (arg),                   \
};                                  \
DATA_SET(cons_set, name)
#define CONSOLE_DRIVER(name, …)                             \
static const struct consdev_ops name##_consdev_ops = {          \
/* 必需的方法。 */             \
.cn_probe = name##_cnprobe,             \
.cn_init = name##_cninit,            \
.cn_term = name##_cnterm,               \
.cn_getc = name##_cngetc,               \
.cn_putc = name##_cnputc,               \
.cn_grab = name##_cngrab,               \
.cn_ungrab = name##_cnungrab,             \
/* 可选字段。 */     \
__VA_ARGS__                   \
};                             \
CONSOLE_DEVICE(name##_consdev, name##_consdev_ops, NULL)
```

该宏定义了控制台驱动程序必须实现的必需回调，并提供了添加额外代码的空间。目前，uart\_tty 利用这一点添加了一个 resume 方法。控制台回调都非常小；我们真的希望控制台在所有情况下都能工作。cn\_probe 需要将控制台优先级（cn\_pri）设置为非 CN\_DEAD 的值，以便控制台驱动程序被 cninit 考虑。

cn\_init 由 cninit 为选定的控制台设备调用（如果我们使用 RB\_MULTIPLE 引导，则为最佳的一个或所有非死亡控制台）。在 cn\_init 中，控制台驱动程序可以建立状态，例如 uart\_cninit 存储一个 cookie，以便控制台不会初始化两次。如果控制台驱动程序被移除，cn\_term 会被调用来清理。cn\_grab 和 cn\_ungrab 用于在内核要打印一系列字符（或读取一个字符）时禁用和启用中断。最后，cn\_putc 和 cn\_get 处理控制台设备的输出和输入。我们关心的魔法。

## 输出字节

有许多不同的设备可以用作系统控制台。有时很难将 pc 与 x86（或 AMD64）平台分开，但使其成为平台而不仅仅是处理器架构的很大一部分是可以预期存在的外设和硬件生态系统。在 PC 上，我们可以预期看到 1 个或更多 UART（串行端口）。这是如此基本，以至于 FreeBSD 15.0 仍然假设会有两个并静态配置它们。UART 通常连接到显式指令（inb 和 outb），这允许调用者用单个指令通过串行端口发送数据。

这个简单的接口是故意的；如果你正在从无到有地启动一台机器，能够通过最小的概念验证从系统获取调试信息是非常方便的。

这个简单的接口能让程序（或驱动程序）与外部世界交互。如果你是新机器的操作员，想知道系统在做什么，那么连接串行打印机、电传打字机，或者在 2026 年，终端程序和串行电缆，让你即使从最小的代码示例中也能获取调试信息。

串行端口上输入和输出字节的指令对处理器和硬件设计者施加了限制。你不能期望每个设备都有一个指令，因此我们主要转向内存映射外设。原始 PC 中使用的相同 8250 和 16650 设备在现代系统中可用，在现代片上系统设计中具有兼容的硬件块。它们不是使用 inb 和 outb，而是内存映射外设。

其他系统为其控制台公开略有不同的设备，但在几乎每个平台上，这都归结为使用旧的 National Semiconductor 8250 或 16650 UART 接口将字节传输到线路上。这意味着我们看到平台特定的控制台设备，但它们中的许多都解析为相同的经过实战测试的代码。

我认为在过去 30 年中使用过计算机的任何人都不会对仅串行端口感到非常满意。从一开始，pc 就支持图形输出，BIOS 通过非常简单的早期程序加载器使访问它变得实用。有了串行端口和图形输出，将 outb 明确到 printf 中以从串行端口获取字节似乎是不正确的。因此，虽然一个非常简单的控制台驱动程序是可能的，但我们最终构建了一个控制台驱动程序子系统，让我们可以重用大部分代码，同时仍然允许平台和硬件特定的变化。

## 加载器

到目前为止，我们已经多次提到加载器，因为它在内核启动期间是相关的，但它本身是一个操作系统。加载器准备内核的操作环境并配置根卷，为内核模块提供用户空间环境。FreeBSD 运行的所有平台都有某种加载器（好吧，我们可以在某些情况下直接引导内核，但这会留下很多功能）。

加载器由平台固件或二级加载器启动。在大多数部署中，加载器在 EFI 环境中运行（在 arm64、arm32、amd64 和 riscv 上）。这个环境为加载器提供了很多早期服务，使其工作更容易，可移植性更直接。由于加载器在许多环境中运行并支持 2 个脚本环境（Lua 和 Forth，是的，2026 年的 Forth），它提供了一些自己的抽象，使核心服务更易于使用。加载器也有控制台驱动程序，它们看起来与内核的非常相似。

stand/common/bootstrap.h：

```c
/*
* 模块化控制台支持。
*/
struct console
{
const char      *c_name;
const char      *c_desc;
int             c_flags;
#define C_PRESENTIN     (1<<0)      /* 控制台可以提供输入 */
#define C_PRESENTOUT    (1<<1)      /* 控制台可以提供输出 */
#define C_ACTIVEIN      (1<<2)      /* 用户希望从控制台输入 */
#define C_ACTIVEOUT     (1<<3)      /* 用户希望输出到控制台 */
#define C_WIDEOUT       (1<<4)      /* c_out 例程理解宽字符 */
/* 设置 c_flags 以匹配硬件 */
void    (* c_probe)(struct console *cp);
/* 重新初始化 XXX 可能需要更多参数 */
int             (* c_init)(int arg);
/* 发出 c */
void            (* c_out)(int c);
/* 等待并返回输入 */
int             (* c_in)(void);
/* 如果输入等待，返回非零 */
int             (* c_ready)(void);
};
```

加载器控制台驱动程序有 in 和 out 方法，就像内核的一样；一个 init 方法，一个 probe 方法，和一个 ready 方法来加速轮询输入。加载器控制台驱动程序没有优先级，而是有一个 flags 参数。每个加载器变体都有自己的 conf.c 文件，定义其平台特定的功能和实现。其中一个功能是可在该平台上使用的控制台列表。对于 loader.efi，我们有：

```c
#if defined(__amd64__) || defined(__i386__)
extern struct console comconsole;
extern struct console nullconsole;
extern struct console spinconsole;
#endif
struct console *consoles[] = {
&efi_console,
&eficom,
#if defined(__aarch64__) && __FreeBSD_version < 1500000
&comconsole,
#endif
#if defined(__amd64__) || defined(__i386__)
&comconsole,
&nullconsole,
&spinconsole,
#endif
NULL
};
```

对于 amd64 机器，我们的完整控制台列表是：

* efi\_console：使用 efi 输出例程的控制台
* eficom：使用 efi 串行输出例程的控制台。这个可能有点奇怪，会导致重复输出，因为 EFI 实现使用帧缓冲区和串行端口。
* comconsole：传统 COM 端口上的控制台，使用的串行控制台
* nullconsole：所有输出字节消失到虚空中。
* spinconsole：每个输出字节使旋转器前进一步；这提供了发生某些事情的视觉指示，但没有调试或提供输入的方法。

userboot 加载器（用于启动 bhyve 实例）、kboot（从 Linux 启动）、u-boot 和其他平台都有平台特定的控制台。在 amd64 上，comconsole 和 eficom 映射到相同的驱动程序函数。

## EFI 加载器控制台初始化

load cons\_probe 函数处理发现控制台并配置它们。它在 efi/loader/main.c 中非常早地被调用：

```c
EFI_STATUS
main(int argc, CHAR16 *argv[])
{
…
#if !defined(__arm__)
efi_smbios_detect();
#endif

/* 获取我们加载的映像协议接口结构。 */
  (void) OpenProtocolByHandle(IH, &imgid, (void **)&boot_img);

  /* 尽早报告 RSDP。 */
  acpi_detect();

  /*
   * 鸡生蛋和蛋生鸡的问题；我们希望尽早有控制台输出，但
   * 一些控制台属性可能取决于从例如引导
   * 设备读取，我们还不能这样做。一旦这样做，我们就可以使用 printf() 等。所以，我们将其设置为 efi 控制台，然后调用控制台初始化。这
   * 让我们尽早使用 printf，同时也为所有未来的控制台
   * 更改做好准备，无论它们来自哪里。
   */
  setenv("console", "efi", 1);
  uhowto = parse_uefi_con_out();
```

```c
#if defined(__riscv)
/*
* 这个解决方法可能是在掩盖一个真正的问题
*/
if ((uhowto & RB_SERIAL) != 0)
setenv("console", "comconsole", 1);
#endif
cons_probe();
/* 设置 print_delay 变量以就位钩子。 */
env_setenv("print_delay", EV_VOLATILE, "", setprint_delay, env_nounset);

/* 设置 currdev 变量以就位钩子。 */
  env_setenv("currdev", EV_VOLATILE, "", gen_setcurrdev, env_nounset);

  /* 初始化时间源 */
  efi_time_init();
```

鸡生蛋和蛋生鸡的评论在源代码中很常见。基本上，我们需要能够尽早调试正在发生的事情，而控制台是设置这一点的最简单方法。我们现在拥有的 EFI 环境可以输出字符串，但我们没有控制台基础设施。我们在所有平台上都玩了一个技巧，在寻找控制台之前静态选择一个控制台。

控制台探测发生在我们可以从存储中读取之前，所以它从假设开始，然后重新评估。在 EFI 上，我们可以在 cons\_probes 之前通过直接 EFI 与 ST->ConOut->OutputString 输出数据。我们有控制台基础设施，这让我们可以使用 printf，所以我们应该这样做。cons\_probe 遍历列表并找到一个工作的控制台。选定的控制台被添加到加载器中的 console 环境变量中，可以通过 loader.conf 或命令行上的环境变量进行更改或添加。

只要选择了控制台，它们就会以类似于内核控制台的方式插入到 printf 中。

common/console.c：

```c
void
putchar(int c)
{
int     cons;
/* 展开换行符 */
if (c == '\n')
putchar('\r');
for (cons = 0; consoles[cons] != NULL; cons++) {
if ((consoles[cons]->c_flags & (C_PRESENTOUT | C_ACTIVEOUT)) ==
(C_PRESENTOUT | C_ACTIVEOUT))
consoles[cons]->c_out(c);
}
/* 如果设置了打印延迟，在打印换行符后暂停 */
if (print_delay_usec != 0 && c == '\n')
delay(print_delay_usec);
}
```

## 调试控制台驱动程序

所以，控制台驱动程序中有所有的魔法，仔细反思后，当我们详细查看它时，它有点暗淡。printf 相当简单，它驱动一个单个字符输出例程。它在内核和加载器之间是相似的，我知道你想知道为什么我用加载器重复了这个深入研究，但这是为了表明它是相同的，并且在内核中有效的调试技巧可能在加载器环境中也有效。如果你卡在控制台前或有一个扰乱工作控制台的问题，你会伸手去拿任何你能用来调试正在发生的事情的东西。

那么我们如何调试控制台驱动程序的问题呢？

我有两个最好的建议：使用调试器或第二个控制台。如果你可以为加载器或早期引导环境使用调试器，那么你真的应该这样做；它会切穿所有其他的东西。如果你有第二个工作控制台（内核调试器利用控制台基础设施）或者如果你可以整体控制正在开发的机器，你可能能够使用调试器。要么使用像 qemu 这样的模拟器，要么使用像 JTAG 这样的硬件调试接口。设置这些超出了本文的范围，我知道如果你读到这里，你会想：“但我这样做只是因为我无法调试！”

第二个控制台是调试控制台的好选择，但它在内核早期可能不会有太大帮助。如果你有一个工作的控制台和关于非工作控制台的问题，你可以绕过大部分基础设施，要么静态配置你知道会在那里的东西，要么直接挂钩到你想要使用的控制台函数。

这就是控制台列表的链接器集和加载器中的静态列表真正有帮助的地方。只需从列表中拉出你想要的控制台并使用 put\_c 做你需要的事情。这样做会跳过所有的锁并失去 printf，但它也会跳过所有的锁，可能会让你获得调试信息。

我曾经使用这种直接捕获来调试加载器控制台驱动程序；加载器没有 early\_putc，所以需要黑客手段。通常，在加载器的 cons\_probe 期间，我们遍历所有控制台并移除 ACTIVE\_OUT 标志；这意味着如果我们使用 printf，我们不会得到任何数据输出。为了调试控制台检测，我静态配置了第一个控制台（知道它会在那里并且工作），并使用 printf 来调试我感兴趣的驱动程序的启动。

更改看起来像这样：

```diff
diff –git a/stand/common/console.c b/stand/common/console.c
index 65ab7ffad622..97201a7479b3 100644
— a/stand/common/console.c
+++ b/stand/common/console.c
@@ -107,14 +107,18 @@ cons_probe(void)
env_setenv("twiddle_divisor", EV_VOLATILE, "16", twiddle_set,
env_nounset);

+ /* 捕获第一个控制台并在解析其他控制台时保持其工作 */
+ consoles[0]->c_flags |= C_ACTIVEIN | C_ACTIVEOUT;
+ consoles[0]->c_init(0);
+ /* 执行所有控制台探测 */
  –    for (cons = 0; consoles[cons] != NULL; cons++) {
+ for (cons = 1; consoles[cons] != NULL; cons++) {
     consoles[cons]->c_flags = 0;
     consoles[cons]->c_probe(consoles[cons]);
     }
     /* 现在找到第一个工作的 */
     active = -1;
     –    for (cons = 0; consoles[cons] != NULL && active == -1; cons++) {
+ for (cons = 1; consoles[cons] != NULL && active == -1; cons++) {
     consoles[cons]->c_flags = 0;
     consoles[cons]->c_probe(consoles[cons]);
     if (consoles[cons]->c_flags == (C_PRESENTIN | C_PRESENTOUT))
```

我对控制台驱动程序的最后一个黑客手段是明确控制输出函数。early\_putc 是一个很好的功能，但它只对某些设备有效，并在 cons\_probe 时被丢弃。如果你拉出你需要使用或测试的控制台驱动程序的 cn\_putc，那么你有一个输出方法，你可以在需要时使用 sans printf。

最近，为了理解为什么 uart 不会输出任何东西，我在 cons\_probe 中添加了这个罪行：

```c
if (uart == NULL || cp == NULL) {
printf("uart is NULL\n");
} else {
// 如果我们启动多，这不会工作，它会在重新初始化时 panic
printf("trying to init\n");
uart->cn_ops->cn_init(uart);
printf("trying to print\n");
uart->cn_ops->cn_putc(uart, 'H');
uart->cn_ops->cn_putc(uart, 'E');
uart->cn_ops->cn_putc(uart, 'L');
uart->cn_ops->cn_putc(uart, 'L');
uart->cn_ops->cn_putc(uart, '\r');
uart->cn_ops->cn_putc(uart, '\n');

#define TEST_COUNT 100
printf("Tring to print %d 4's\n", TEST_COUNT);
for (int i = 0; i < TEST_COUNT; i++)
uart->cn_ops->cn_putc(uart, '4');

uart->cn_ops->cn_putc(uart, '\r');
    uart->cn_ops->cn_putc(uart, '\n');

}

for( int i = 0; i < 40; i++) {
DELAY(100000);
}
```

我们手动设置 uart 并输出字节，我试图理解为什么没有数据从端口出来，那个输出 4 的循环给了我一个同步示波器的东西。

## 慰藉

如果你必须调试控制台驱动程序或在加载器或内核的预引导或非常早期的引导中工作，那么你会接受你能得到的东西，以从系统中获取数据。我们不必切换 GPIO；谢天谢地，控制台系统在设计和使用上相当简单。这意味着它应该工作，但如果它不工作，你可以弯曲它来在你需要时输出数据。这些类型的中间黑客手段很可怕，永远不会出现在源代码树中，但当你需要调试某些东西而它不配合时，它们会非常有用。

--

Tom Jones 是 FreeBSD 提交者，对保持网络堆栈快速感兴趣。
