# 字符设备驱动程序教程

- 原文地址：[Character Device Driver Tutorial](https://freebsdfoundation.org/our-work/journal/browser-based-edition/kernel-development/character-device-driver-tutorial/)
- 作者：John Baldwin

**字符设备提供了由设备文件系统（[devfs(5)](https://man.freebsd.org/devfs/5)）暴露到用户空间应用程序的伪文件**。与标准文件系统不同，在标准文件系统中，像读取和写入等操作的语义，在文件系统内的所有文件间是一样的；而所有字符设备为每个文件操作都定义了自己的语义。字符设备驱动程序会声明一个字符设备 switch（character device switch）（`struct cdevsw`），其中包含了每个文件操作的函数指针。

字符设备 switch 通常作为硬件设备驱动程序的一部分实现。例如，FreeBSD 的内核提供了几种包装器 API，它们在一组更简单的操作之上实现了字符设备。例如，[disk(9)](https://man.freebsd.org/disk/9) API 在 `struct disk` 的方法之上实现了一个内部字符设备 switch。某些设备驱动程序提供了字符设备以暴露未与现有内核子系统映射的设备行为到用户空间。

其他字符设备 switch 完全以软件构造实现。例如，字符设备 `/dev/null` 和 `/dev/zero` 并未与任何硬件设备关联。

在三篇系列文章中，本文是第一篇，我们将逐步构建一款简单的字符设备驱动程序，逐步添加新功能，以探索字符设备 switch 及其驱动程序能实现的多项操作。可以在 [https://github.com/bsdjhb/cdev_tutorial](https://github.com/bsdjhb/cdev_tutorial) 上找到每个版本的设备驱动程序的完整源代码。我们将从一款创建单个字符设备的基本驱动程序开始。

## 生命周期管理

字符设备驱动程序负责显式创建和销毁字符设备。活动的字符设备通过 `struct cdev` 的实例表示。字符设备通过函数 [make_dev_s(9)](https://man.freebsd.org/make_dev_s/9) 创建。此函数接受一个指向参数结构体的指针、一个指向字符设备对象指针的指针，以及一个 printf 风格的格式字符串及其后跟参数。格式字符串和后跟参数用于构建字符设备的名称。

参数结构体包含几个必填字段和若干可选字段。在设置字段之前，必须通过调用 `make_dev_args_init()` 对结构体进行初始化。`mda_devsw` 成员必须指向字符设备切换。`mda_uid`、`mda_gid` 和 `mda_mode` 字段应设置为设备节点的初始用户 ID、组 ID 和权限。大多数字符设备由 `root:wheel` 拥有，可以使用常量 `UID_ROOT` 和 `GID_WHEEL`。`mda_flags` 字段还应设置为 `MAKEDEV_NOWAIT` 或 `MAKEDEV_WAITOK`。如果需要，还可以通过 C 或操作符包含其他标志。对于我们的示例驱动程序，我们设置了 `MAKEDEV_CHECKNAME`，以便在设备已经存在时优雅地失败并返回错误，而不是使系统崩溃。

字符设备通过将字符设备的指针传递给 `destroy_dev()` 来销毁。此函数将在所有引用该字符设备的地方被移除后阻塞，包括等待当前在字符设备切换方法中执行的任何线程返回。待 `destroy_dev()` 返回，就可以安全地释放字符设备使用的任何资源。或者，字符设备可以通过 `destroy_dev_sched()` 或 `destroy_dev_sched_cb()` 异步销毁。这些函数将字符设备销毁任务调度到内部内核线程。对于 `destroy_dev_sched_cb()`，在字符设备销毁后，提供的回调将与提供的参数一起调用。此回调可用于释放字符设备使用的资源。请记住，字符设备使用的资源之一是字符设备切换方法。这意味着，例如，模块卸载必须等待所有使用该模块中定义的函数的字符设备被销毁。

对于我们的初始驱动程序（清单 1），我们使用一个模块事件处理程序，在模块加载时创建一个 `/dev/echo` 设备，在模块卸载时销毁它。构建并加载该模块后，设备存在，但如示例 1 所示，它无法执行太多操作。该驱动程序的字符设备切换（`echo_cdevsw`）仅初始化了两个必需字段：`d_version` 必须始终设置为常量 `D_VERSION`，`d_name` 应设置为驱动程序名称。

**清单 1：基础驱动程序**

```c
#include <sys/param.h>
#include <sys/conf.h>
#include <sys/kernel.h>
#include <sys/module.h>

static struct cdev *echodev;

static struct cdevsw echo_cdevsw = {
      .d_version =      D_VERSION,
      .d_name =         “echo”
};

static int
echodev_load(void)
{
      struct make_dev_args args;
      int error;

      make_dev_args_init(&args);
      args.mda_flags = MAKEDEV_WAITOK | MAKEDEV_CHECKNAME;
      args.mda_devsw = &echo_cdevsw;
      args.mda_uid = UID_ROOT;
      args.mda_gid = GID_WHEEL;
      args.mda_mode = 0600;
      error = make_dev_s(&args, &echodev, “echo”);
      return (error);
}

static int
echodev_unload(void)
{
      if (echodev != NULL)
            destroy_dev(echodev);
      return (0);
}

static int
echodev_modevent(module_t mod, int type, void *data)
{
      switch (type) {
      case MOD_LOAD:
            return (echodev_load());
      case MOD_UNLOAD:
            return (echodev_unload());
      default:
            return (EOPNOTSUPP);
      }
}

DEV_MODULE(echodev, echodev_modevent, NULL);
```

**示例 1：使用基础驱动程序**

```sh
# ls -l /dev/echo
crw-------  1 root wheel 0x39 Oct 25 13:06 /dev/echo
# cat /dev/echo
cat: /dev/echo: Operation not supported by device
```

## 读取和写入

现在我们有了一个字符设备，接下来让我们添加一些行为。正如“echo”（回显）这个名字所暗示的，这个设备应该通过写入设备来接受输入，并通过从设备读取来回显这些输入。为实现这一点，我们将在字符设备切换中添加读取和写入方法。

字符设备的读取和写入请求通过 `struct uio` 对象描述。这个结构中的两个字段对于字符设备驱动程序非常有用：`uio_offset` 是请求开始的逻辑文件偏移量（例如来自 [lseek(2)](https://man.freebsd.org/lseek/2)），`uio_resid` 是要传输的字节数。数据通过 [uiomove(9)](https://man.freebsd.org/uiomove/9) 函数在应用程序缓冲区和内核缓冲区之间传输。该函数会更新 `uio` 对象的成员，包括 `uio_offset` 和 `uio_resid`，并且可以多次调用。请求可以通过将一部分字节从应用程序缓冲区传输到内核缓冲区或相反完成，从而作为一个短操作。

第二版的回显驱动程序添加了一个全局静态缓冲区，用作读取和写入请求的后备存储。逻辑文件偏移量被视为全局缓冲区的偏移量。请求会被截断为缓冲区的大小，因此读取超出缓冲区末尾的部分会触发零字节读取，表示文件结束（EOF）。超出缓冲区末尾的写入会因错误 `EFBIG` 而失败。为了防止并发访问，使用全局 [sx(9)](https://man.freebsd.org/sx/9) 锁来保护缓冲区。由于 `uiomove()` 在访问应用程序缓冲区的页面时可能会休眠，因此使用 `sx(9)` 锁而不是常规的互斥锁。清单 2 显示了使用全局缓冲区的读取和写入字符设备方法。

**清单 2：使用全局缓冲区进行读取和写入**

```c
static int
echo_read(struct cdev *dev, struct uio *uio, int ioflag)
{
      size_t todo;
      int error;

      if (uio->uio_offset >= sizeof(echobuf))
            return (0);

      sx_slock(&echolock);
      todo = MIN(uio->uio_resid, sizeof(echobuf) - uio->uio_offset);
      error = uiomove(echobuf + uio->uio_offset, todo, uio);
      sx_sunlock(&echolock);
      return (error);
}

static int
echo_write(struct cdev *dev, struct uio *uio, int ioflag)
{
      size_t todo;
      int error;

      if (uio->uio_offset >= sizeof(echobuf))
            return (EFBIG);

      sx_xlock(&echolock);
      todo = MIN(uio->uio_resid, sizeof(echobuf) - uio->uio_offset);
      error = uiomove(echobuf + uio->uio_offset, todo, uio);
      sx_xunlock(&echolock);
      return (error);
}
```

这些方法的主体基本相同。原因之一是，`uiomove()` 的参数对于读取和写入操作是相同的。这是因为 `uio` 对象将数据传输的方向作为其状态的一部分进行编码。

如果我们加载此版本的驱动程序，现在可以通过读取和写入设备与其进行交互。示例 2 展示了几次交互，演示了回显行为。请注意，`jot` 的输出超出了驱动程序 64 字节缓冲区的大小，因此随后的设备读取被截断。

**示例 2：使用全局缓冲区回显数据**

```sh
# cat /dev/echo
# echo foo > /dev/echo
# cat /dev/echo
foo
# jot -c -s “” 70 48 > /dev/echo
# cat /dev/echo
0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmno#
```

## 通过 ioctl() 配置设备

全局缓冲区的固定大小是该设备的一个特殊之处。我们可以通过为该设备添加一个自定义的 [ioctl(2)](https://man.freebsd.org/ioctl/2) 命令来允许更改缓冲区的大小。I/O 控制命令通过命令常量命名，并接受一个可选的参数。

命令常量是通过 \<sys/ioccom.h\> 头文件中的 `_IO`、`_IOR`、`_IOW` 或 `_IOWR` 宏定义的。这些宏都接受一个组和一个数字作为前两个参数。两个值都是 8 位的。通常，组使用 ASCII 字母字符，而给定驱动程序的所有命令使用相同的组。FreeBSD 的内核定义了几个现有的 I/O 控制命令集。一个可以与任何文件描述符一起使用的通用命令集在 \<sys/filio.h\> 中定义，使用组 'f'。其他命令集则用于特定类型的文件描述符，例如在 \<sys/sockio.h\> 中为套接字定义的命令。对于字符设备驱动程序的自定义命令，不要使用 'f' 组，以避免与 \<sys/filio.h\> 中的通用命令发生冲突。每个命令应使用不同的数字参数值。如果命令接受可选参数，则必须将参数类型作为第三个参数传递给 `_IOR`、`_IOW` 或 `_IOWR` 宏。`_IOR` 宏定义一个从驱动程序返回值到用户空间应用程序的命令（该命令“读取”来自驱动程序的参数）。`_IOW` 宏定义一个向驱动程序传递值的命令（该命令“写入”参数到驱动程序）。`_IOWR` 宏定义一个既由驱动程序读取也写入的命令。参数的大小被编码在命令常量中。这意味着具有相同组和编号但参数大小不同的命令将具有不同的命令常量。在实现对替代用户空间 ABI（例如，支持 64 位内核上的 32 位用户空间应用程序）的支持时，这一点非常有用，因为替代 ABI 将使用不同的命令常量。

如 FreeBSD 等 BSD 内核在通用系统调用层管理 I/O 控制命令参数的复制。这与 Linux 不同，后者将原始用户空间指针传递给设备驱动程序，要求设备驱动程序将数据复制到用户空间并从用户空间复制回来。相反，BSD 内核使用命令常量中编码的大小参数来分配一个内核缓冲区，用于请求的大小。如果命令使用 `_IOW` 或 `_IOWR` 定义，则通过从用户空间应用程序复制参数值来初始化缓冲区。如果命令使用 `_IOR` 定义，则将缓冲区清零。设备驱动程序的 ioctl 例程完成后，如果命令使用 `_IOR` 或 `_IOWR` 定义，则缓冲区的内容将被复制到用户空间应用程序中。

对于回显驱动程序，让我们定义三个新的控制命令。第一个命令返回全局缓冲区的当前大小。第二个命令允许设置全局缓冲区的新大小。第三个命令通过将所有字节重置为零来清除缓冲区的内容。

这些命令在清单 3 中的一个新的 `echodev.h` 头文件中定义。使用头文件是为了使常量可以在用户空间应用程序和驱动程序之间共享。请注意，第一个命令将缓冲区大小读取到用户空间的 `size_t` 参数中，第二个命令将新的缓冲区大小写入用户空间的 `size_t` 参数中，第三个命令不接受参数。所有三个命令都使用 'E' 组，并分配了唯一的命令号。

**清单 3：I/O 控制命令常量**

```c
#define     ECHODEV_GBUFSIZE   _IOR('E', 100, size_t)   /* 获取 buffer 大小 */
#define     ECHODEV_SBUFSIZE   _IOW('E', 101, size_t)   /* 设置 buffer 大小 */
#define     ECHODEV_CLEAR      _IO('E', 102)            /* 清除 buffer */
```

支持动态大小缓冲区需要对驱动程序进行一些更改。全局缓冲区被替换为指向动态分配缓冲区的全局指针，并且一个新的全局变量包含缓冲区的当前大小。指针和长度在模块加载时初始化，并在模块卸载时释放当前的缓冲区。由于缓冲区的大小不再是常量，因此现在必须在持有锁的情况下进行越界读取和写入的检查。

FreeBSD 内核中的 [malloc(9)](https://man.freebsd.org/malloc/9) 分配器要求在分配和释放例程中都提供额外的 malloc 类型参数。Malloc 类型跟踪分配请求，并提供细粒度的统计信息。这些统计信息可以通过 [vmstat(8)](https://man.freebsd.org/vmstat/8) 命令的 `-m` 标志查看，命令会为每种类型显示一行。内核确实包括一个通用的设备缓冲区 malloc 类型（`M_DEVBUF`），驱动程序可以使用它。然而，最佳实践是让驱动程序定义一个专用的 malloc 类型。特别是对于内核模块中的驱动程序来说尤其如此。当模块卸载时，内核模块中定义的 malloc 类型会被销毁。如果仍然有分配引用这些 malloc 类型，内核将发出关于泄漏分配的警告。更细粒度的统计信息对于调试和性能分析也很有用。新的 malloc 类型通过 `MALLOC_DEFINE` 宏定义。第一个参数提供新类型的变量名。按惯例，类型名使用全大写，并以“`M_`”作为前缀。对于此驱动程序，我们将使用名称 `M_ECHODEV`。第二个参数是一个短字符串名称，工具（如 `vmstat(8)`）将显示该名称。最佳实践是避免在短名称中使用空格字符。第三个参数是对该类型的字符串描述。

驱动程序对自定义控制命令的支持在清单 4 中的新函数中实现。`cmd` 参数包含请求的命令常量，`data` 参数指向包含可选命令参数的内核缓冲区。函数的整体结构是一个基于 `cmd` 参数的 switch 语句。对于未知命令，默认的错误值是 `ENOTTY`，即使对于非 TTY 设备也是如此。接受大小参数的两个命令在解引用之前将 `data` 强制转换为正确的指针类型。`ECHODEV_GBUFSIZE` 命令将当前大小写入 `*data`，而 `ECHODEV_SBUFSIZE` 命令从 `*data` 中读取所需的新大小。

对于改变设备状态的命令，驱动程序要求文件描述符具有写权限（即使用 `O_RDWR` 或 `O_WRONLY` 打开的文件描述符）。为了强制执行这一点，`ECHODEV_SBUFSIZE` 和 `ECHODEV_CLEAR` 命令要求 `fflag` 中设置 `FWRITE` 标志。`fflag` 参数包含在 \<sys/fcntl.h\> 中定义的文件描述符状态标志。这些标志将 `O_RDONLY`、`O_WRONLY` 和 `O_RDWR` 映射为 `FREAD` 和 `FWRITE` 标志的组合。`open(2)` 中的所有其他标志直接包含在文件描述符状态标志中。请注意，可以通过 [fcntl(2)](https://man.freebsd.org/fcntl/2) 在已打开的文件描述符上更改这些标志的子集。

**清单 4：I/O 控制处理程序**

```c
static int
echo_ioctl(struct cdev *dev, u_long cmd, caddr_t data, int fflag,
    struct thread *td)
{
      int error;

      switch (cmd) {
      case ECHODEV_GBUFSIZE:
            sx_slock(&echolock);
           *(size_t *)data = echolen;
            sx_sunlock(&echolock);
            error = 0;
            break;
      case ECHODEV_SBUFSIZE:
      {
            size_t new_len;

            if ((fflag & FWRITE) == 0) {
                  error = EPERM;
                  break;
            }

            new_len = *(size_t *)data;
            sx_xlock(&echolock);
            if (new_len == echolen) {
                  /* Nothing to do. */
            } else if (new_len < echolen) {
                  echolen = new_len;
            } else {
                  echobuf = reallocf(echobuf, new_len, M_ECHODEV,
                      M_WAITOK | M_ZERO);
                  echolen = new_len;
            }
            sx_xunlock(&echolock);
            error = 0;
            break;
      }
      case ECHODEV_CLEAR:
            if ((fflag & FWRITE) == 0) {
                  error = EPERM;
                  break;
            }

            sx_xlock(&echolock);
            memset(echobuf, 0, echolen);
            sx_xunlock(&echolock);
            error = 0;
            break;
      default:
            error = ENOTTY;
            break;
      }
      return (error);
```

为了从用户空间调用这些命令，我们需要一个新的用户应用程序。该代码库包含一个 `echoctl` 程序，在示例 3 中使用。`size` 命令输出当前缓冲区的大小，`resize` 命令设置新的缓冲区大小，`clear` 命令清除缓冲区内容。请注意，在这个示例中，`jot` 的输出不再被截断。该示例中的最后一条命令显示了使用 `M_ECHODEV` 分配的驱动程序的动态分配统计信息。

**示例 3：调整全局缓冲区大小**

```sh
# echoctl size
64
# echo foo > /dev/echo
# echoctl clear
# cat /dev/echo
# echoctl resize 80
# jot -c -s “” 70 48 > /dev/echo
# cat /dev/echo
0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdefghijklmnopqrstu
# vmstat -m | egrep 'Type|echo'
           Type Use Memory Req Size(s)
        echodev    1   128    2 64,128
```

## 每实例数据

到目前为止，我们的设备驱动程序使用全局变量来保存其状态。对于一个简单的演示驱动程序，且仅有一个设备实例，这样是可以的。然而，大多数字符设备都是硬件设备驱动程序的一部分，并且需要在单个系统中支持多个设备实例。为了支持这一点，驱动程序定义一个包含单个设备实例软件上下文的结构体。在 BSD 内核中，这个软件上下文被称为“softc”。驱动程序通常定义一个结构类型，其名称以“\_softc”作为后缀，而指向 softc 结构体的变量通常命名为“sc”。

字符设备提供了对每实例数据的直接支持。`struct cdev` 包含三个成员，可以用来存储驱动程序特定的数据。`si_drv0` 存储一个整数值，而 `si_drv1` 和 `si_drv2` 存储任意指针。设备驱动程序可以在创建字符设备时，通过 `struct make_dev_args` 结构体中的 `mda_unit`、`mda_si_drv1` 和 `mda_si_drv2` 字段来设置这些变量。然后，这些值可以作为 `struct cdev` 参数中的成员，供字符设备开关方法访问。历史上，设备驱动程序使用单元号来跟踪每实例数据。现代 FreeBSD 设备驱动程序将一个 softc 指针存储在 `si_drv1` 字段中，并且很少使用其他两个字段。

对于我们的 echo 设备驱动程序，我们定义了一个 `struct echodev_softc` 类型，包含 echo 设备实例所需的所有状态。设备驱动程序仍然存储一个全局变量，用于在模块加载和卸载时保存单个实例的 softc，但驱动程序的其余部分通过 softc 指针访问状态。这些更改不会改变驱动程序的任何功能，但确实需要重构驱动程序的各个部分。清单 5 显示了新的 softc 结构体类型。清单 6 演示了每个字符设备开关方法所需的重构类型，通过展示更新后的读取方法。最后，清单 7 显示了模块加载和卸载时使用的更新例程。

**清单 5：softc 结构体**

```c
struct echodev_softc {
      struct cdev *dev;
      char *buf;
      size_t len;
      struct sx lock;
};
```

**清单 6：使用 softc 结构体的驱动程序方法**

```c
static int
echo_read(struct cdev *dev, struct uio *uio, int ioflag)
{
      struct echodev_softc *sc = dev->si_drv1;
      size_t todo;
      int error;

      sx_slock(&sc->lock);
      if (uio->uio_offset >= sc->len) {
            error = 0;
      } else {
            todo = MIN(uio->uio_resid, sc->len - uio->uio_offset);
error = uiomove(sc->buf + uio->uio_offset, todo, uio);
      }
      sx_sunlock(&sc->lock);
      return (error);
}
```
**清单 7：使用 softc 结构体的模块加载和卸载**

```c
static int
echodev_create(struct echodev_softc **scp, size_t len)
{
      struct make_dev_args args;
      struct echodev_softc *sc;
      int error;

      sc = malloc(sizeof(*sc), M_ECHODEV, M_WAITOK | M_ZERO);
      sx_init(&sc->lock, “echo”);
      sc->buf = malloc(len, M_ECHODEV, M_WAITOK | M_ZERO);
      sc->len = len;
      make_dev_args_init(&args);
      args.mda_flags = MAKEDEV_WAITOK | MAKEDEV_CHECKNAME;
      args.mda_devsw = &echo_cdevsw;
      args.mda_uid = UID_ROOT;
      args.mda_gid = GID_WHEEL;
      args.mda_mode = 0600;
      args.mda_si_drv1 = sc;
      error = make_dev_s(&args, &sc->dev, “echo”);
      if (error != 0) {
            free(sc->buf, M_ECHODEV);
            sx_destroy(&sc->lock);
            free(sc, M_ECHODEV);
      }
      return (error);
}

static void
echodev_destroy(struct echodev_softc *sc)
{
      if (sc->dev != NULL)
            destroy_dev(sc->dev);
      free(sc->buf, M_ECHODEV);
      sx_destroy(&sc->lock);
      free(sc, M_ECHODEV);
}

static int
echodev_modevent(module_t mod, int type, void *data)
{
      static struct echodev_softc *echo_softc;

      switch (type) {
      case MOD_LOAD:
            return (echodev_create(&echo_softc, 64));
      case MOD_UNLOAD:
            if (echo_softc != NULL)
                   echodev_destroy(echo_softc);
            return (0);
      default:
            return (EOPNOTSUPP);
      }
}
```

## 结论

感谢你阅读到这里。本系列的下一篇文章将扩展这个驱动程序，实现一个 FIFO 缓冲区，包括对非阻塞 I/O 和通过 [poll(2)](https://man.freebsd.org/poll/2) 和 [kevent(2)](https://man.freebsd.org/kevent/2) 的 I/O 事件报告的支持。

---

**John Baldwin** 是一位系统软件开发人员。他在过去的二十多年里，直接向 FreeBSD 操作系统提交了包括 x86 平台支持、SMP、各种设备驱动程序和虚拟内存子系统在内的多个内核部分以及用户空间程序的更改。除了编写代码，John 还曾在 FreeBSD 核心团队和发布工程团队中担任职务。他还为 GDB 调试器做出了贡献。John 与妻子 Kimberly 和三个孩子：Janelle、Evan 和 Bella 一起住在弗吉尼亚州的 Ashland。
