# 字符设备驱动程序教程（第二部分）

- 原文链接：[Character Device Driver Tutorial (Part 2)](https://freebsdfoundation.org/our-work/journal/browser-based-edition/virtualization-2/character-device-driver-tutorial-part-2/)
- 作者：**John Baldwin**

在这篇三部分系列的[上一篇文章](https://freebsdfoundation.org/our-work/journal/browser-based-edition/kernel-development/character-device-driver-tutorial/)中，我们构建了一个简单的字符设备驱动程序，该程序支持由固定缓冲区支持的 I/O 操作。在本文中，我们将扩展此驱动程序，以支持 FIFO 数据缓冲区，并支持非阻塞 I/O 和事件报告。每个版本的驱动程序的完整源代码可以在[https://github.com/bsdjhb/cdev_tutorial](https://github.com/bsdjhb/cdev_tutorial)找到。

然而，在继续之前，我们必须处理上一篇文章中未完成的部分。细心的读者 Virus-V [指出](https://github.com/bsdjhb/cdev_tutorial/issues/1)，文章中的 echo 驱动程序的最终版本没有在卸载时销毁 `/dev/echo` 设备，并且在卸载后访问该设备会触发内核 panic。这个 bug 的第一个线索出现在内核在卸载模块时发出的关于内存泄漏的警告消息，panic 发生之前就已经出现了这个警告。如前文所述，这是内核模块在可能的情况下应该使用专用 malloc 类型的原因之一。该 bug 出现在最后一组更改中添加的 `echodev_create()` 函数中。我们未能通过将值存储在 `*scp` 中将指向新分配的 softc 结构的指针返回给调用者。因此，`echo_softc` 变量始终为 NULL，softc 在模块卸载时未被销毁。修复方法是在 `echodev_create()` 中添加一行代码，在成功时将指针存储到 `*scp` 中，指向新的 softc。

## 使用 FIFO 数据缓冲区

上一篇文章中的 echo 驱动程序使用了一个平面的数据缓冲区进行 I/O 操作。读取和写入可以访问数据缓冲区的任何区域，并且数据缓冲区的整个范围始终有效。这些语义类似于访问一个在写入末尾时不会增长的文件。然而，字符设备驱动程序可以实现一系列不同的语义。对于本文，我们将修改 echo 驱动程序，使其像 [`pipe`](https://man.freebsd.org/pipe/2) 或 [`fifo`](https://man.freebsd.org/mkfifo/2) 这样的 FIFO 流设备一样对待用户 I/O 数据。I/O 写请求将数据追加到逻辑数据缓冲区的尾部，读取请求将从数据缓冲区的头部读取数据。像 [`pread(2)`](https://man.freebsd.org/pread/2) 这样的文件偏移量将被忽略。驱动程序将继续使用内核中的缓冲区来保存用户数据的临时副本。写入将数据存储在此缓冲区中，读取将从此缓冲区中消耗数据。这意味着驱动程序现在需要跟踪缓冲区中有效数据的数量以及缓冲区的长度。为了简化实现，数据缓冲区的开始部分将始终被视为缓冲区的头部。读取请求如果读取了部分可用数据，将把剩余数据复制到缓冲区的前面。

然而，这确实提出了几个额外的问题。首先，如何处理希望从缓冲区中读取超过可用数据量的读取请求？其次，如何处理希望存储比缓冲区能容纳的更多数据的写入请求？为了简单起见，我们将首先对读取请求返回可用字节的短读取，并将写请求截断，只存储缓冲区中有足够空间的数据量。第三，对于一个 `ECHODEV_SBUFSIZE` 请求，如果将缓冲区缩小到小于有效数据量的大小，应该如何处理？我们选择使此类请求失败并返回错误。也可以选择丢弃一些数据，但必须决定丢弃哪些数据。列表 1 提供了更新后的读写方法。注意，softc 中新增了一个 `valid` 成员，用于跟踪缓冲区中有效数据的数量。示例 1 展示了这个更新后的驱动程序的一些场景。最初，设备为空，但待提供输入，就可以读取数据。最后几个命令跨两个请求读取了一系列字节。

**列表 1：使用 FIFO 数据缓冲区的读取和写入**

```c
static int
echo_read(struct cdev *dev, struct uio *uio, int ioflag)
{
      struct echodev_softc *sc = dev->si_drv1;
      size_t todo;
      int error;

      sx_xlock(&sc->lock);
      todo = MIN(uio->uio_resid, sc->valid);
      error = uiomove(sc->buf, todo, uio);
      if (error == 0) {
            sc->valid -= todo;
            memmove(sc->buf, sc->buf + todo, sc->valid);
      }
      sx_xunlock(&sc->lock);
      return (error);
}

static int
echo_write(struct cdev *dev, struct uio *uio, int ioflag)
{
      struct echodev_softc *sc = dev->si_drv1;
      size_t todo;
      int error;

      sx_xlock(&sc->lock);
      todo = MIN(uio->uio_resid, sc->len - sc->valid);
      error = uiomove(sc->buf + sc->valid, todo, uio);
      if (error == 0)
            sc->valid += todo;
      sx_xunlock(&sc->lock);
      return (error);
}
```

**示例 1：简单的 FIFO I/O**

```sh
# hd < /dev/echo

# echo “foo” > /dev/echo
# cat /dev/echo
foo
# echo “12345678” > /dev/echo
# dd if=/dev/echo bs=1 count=4 status=none | hd
00000000 31 32 33 34 |1234|
00000004
# cat /dev/echo
5678
```

## 阻塞 I/O

虽然这个版本的回显驱动实现了一个简单的数据流，但它也有一些局限性。如果一个进程想要使用这个设备共享一个比数据缓冲区大的数据块，它必须等到读者消耗完之前缓冲区中的数据，才能写入额外的缓冲区数据。这要求写入进程要么与读取进程协调，要么使用定时器并定期重试写操作。两种解决方案都不太实际。相反，驱动程序可以通过在缓冲区满时在写请求中睡眠，直到请求完成，从而允许更大的写入。读者在空间可用时会唤醒等待的写入者，从而允许写入者继续进行写入。类似地，读者可以阻塞等待数据返回。为了更贴近管道和套接字的语义，我们选择使读取请求仅在请求开始时阻塞，并在数据可用时尽快返回短读取。然而，对于写入操作，我们尝试一次性清空整个缓冲区。为了处理阻塞，我们使用了 [`sx_sleep(9)`](https://man.freebsd.org/sx_sleep/9) 函数，它在将当前线程置于休眠状态的同时，原子地释放设备的锁。将 `PCATCH` 传递给该函数允许信号中断休眠，若中断发生，`sx_sleep()` 将返回一个非零的错误值。列表 2 显示了更新后的读取方法。写入方法也进行了类似的更新，但增加了一个额外的循环，直到写入完全完成。请注意，在写入方法中，如果写入操作部分完成，我们不需要“隐藏”错误。`dofilewrite()` 函数中的通用写系统调用处理将 `sx_sleep()` 的错误映射为成功，只要至少有一些数据被写入。一些 `ioctl` 处理程序也需要更新，以便在缓冲区增大或清空内容时唤醒正在休眠的写入者。

在测试这个版本的驱动时，示例 2 显示了一些可能令人惊讶的行为。尽管先前写入的数据会被返回，但 [`cat(1)`](https://man.freebsd.org/cat/1) 会继续等待更多数据，直到被信号终止。

**列表 2**：阻塞读取方法

```c
static int
echo_read(struct cdev *dev, struct uio *uio, int ioflag)
{
      struct echodev_softc *sc = dev->si_drv1;
      size_t todo;
      int error;

      if (uio->uio_resid == 0)
            return (0);

      sx_xlock(&sc->lock);

      /* Wait for bytes to read. */
      while (sc->valid == 0) {
            error = sx_sleep(sc, &sc->lock, PCATCH, “echord”, 0);
            if (error != 0) {
                  sx_xunlock(&sc->lock);
                  return (error);
            }
      }

      todo = MIN(uio->uio_resid, sc->valid);
      error = uiomove(sc->buf, todo, uio);
      if (error == 0) {
            /* 唤醒任何等待的写入者。 */
            if (sc->valid == sc->len)
                  wakeup(sc);

            sc->valid -= todo;
            memmove(sc->buf, sc->buf + todo, sc->valid);
      }
      sx_xunlock(&sc->lock);
      return (error);
}
```

**示例 2**：阻塞 I/O 永久挂起

```sh
# echo “12345678” > /dev/echo
# cat /dev/echo
12345678
^C
```

## 安全卸载

我们稍后会讨论示例 2 中的令人惊讶的行为。当前的驱动程序还有另一个问题。当一个进程在读取或写入方法中被阻塞时，卸载驱动程序会导致卸载模块的进程挂起，直到第一个进程被信号杀死。这不是管理员卸载模块时所期望的行为。相反，回显驱动程序应该在设备销毁时唤醒任何休眠的线程，并确保它们能够从驱动程序方法中返回，而不会再次进入休眠状态。为了支持这一点，下一个修改添加了一个 `dying` 标志到 `softc` 中，并且如果该标志被设置，读取和写入方法将返回 `ENXIO` 错误，而不是阻塞。在设备销毁期间，`dying` 标志被设置，并且在调用 `destroy_dev()` 之前，所有休眠的线程都会被唤醒。列表 3 显示了读取方法中的更改行和更新的 `echodev_destroy()` 函数。

**列表 3：卸载时唤醒线程**

```c
static int
echo_read(struct cdev *dev, struct uio *uio, int ioflag)
{
      ...
      /* 等待读取字节。 */
      while (sc->valid == 0) {
            if (sc->dying)
                  error = ENXIO;
            else
                  error = sx_sleep(sc, &sc->lock, PCATCH, “echord”, 0);
            if (error != 0) {
                  sx_xunlock(&sc->lock);
                  return (error);
            }
      }
      ...
}

...

static void
echodev_destroy(struct echodev_softc *sc)
{
      if (sc->dev != NULL) {
            /* 强制任何正在休眠的线程退出驱动程序。*/
            sx_xlock(&sc->lock);
            sc->dying = true;
            wakeup(sc);
            sx_xunlock(&sc->lock);

            destroy_dev(sc->dev);
      }
      free(sc->buf, M_ECHODEV);
      sx_destroy(&sc->lock);
      free(sc, M_ECHODEV);
}
```

## 读取时的条件阻塞

在示例 2 中，令人惊讶的是，在读取了设备上的可用数据后，`cat(1)` 仍然继续阻塞。然而，这种行为确实是我们驱动程序的自然结果，因为 `cat(1)` 只是循环调用 [`read(2)`](https://man.freebsd.org/read/2)，直到收到 EOF，而第二次调用 `read(2)` 会阻塞，等待更多数据。其他流设备（如管道和 FIFO）的语义是，如果没有进程打开设备进行写入，读取将返回 EOF 而不是阻塞。如果有设备被打开进行写入，则读取将阻塞，等待更多数据。

我们可以很容易地在回显驱动程序中实现这些语义。我们向 `softc` 中添加一个写入者计数器，只有当计数器非零时，读取方法才会阻塞。为了检测写入者，我们添加了一个打开方法，当每个请求写权限的打开操作时增加计数。可以通过检查传递给打开方法的文件标志中的 `FWRITE` 标志来确定这一点。一个新的关闭方法在写入者关闭时减少计数。默认情况下，关闭字符设备开关方法仅在没有剩余文件描述符时为设备的最后一次关闭调用。相反，我们设置了 `D_TRACKCLOSE` 字符设备开关标志，这样每次文件描述符关闭时都会调用关闭方法。如果最后一个写入者关闭，关闭方法会唤醒任何正在等待的读取者。列表 4 显示了新的打开和关闭方法，以及读取方法中的更改行。重试示例 2 中的步骤后，`cat(1)` 现在在读取完可用数据后会退出，不再出现令人惊讶的行为。

**列表 4：跟踪打开的写入者**

```c
static int
echo_open(struct cdev *dev, int fflag, int devtype, struct thread *td)
{
      struct echodev_softc *sc = dev->si_drv1;

      if ((fflag & FWRITE) != 0) {
            /* Increase the number of writers. */
            sx_xlock(&sc->lock);
            if (sc->writers == UINT_MAX) {
                  sx_xunlock(&sc->lock);
                  return (EBUSY);
            }
            sc->writers++;
            sx_xunlock(&sc->lock);
      }
      return (0);
}

static int
echo_close(struct cdev *dev, int fflag, int devtype, struct thread *td)
{
      struct echodev_softc *sc = dev->si_drv1;

      if ((fflag & FWRITE) != 0) {
            sx_xlock(&sc->lock);
            sc->writers--;
            if (sc->writers == 0) {
                  /* 唤醒任何等待的读取者。 */
                  wakeup(sc);
            }
            sx_xunlock(&sc->lock);
      }
      return (0);
}

static int
echo_read(struct cdev *dev, struct uio *uio, int ioflag)
{
      ...
      /* 等待读取字节。 */
      while (sc->valid == 0 && sc->writers != 0) {
      ...
}
```

## 非阻塞 I/O

现在我们的回显设备支持阻塞 I/O。然而，一些消费者可能希望使用非阻塞 I/O。进程可以通过将 `O_NONBLOCK` 标志传递给 [`open(2)`](https://man.freebsd.org/open/2)，或者通过 [`fcntl(2)`](https://man.freebsd.org/fcntl/2) 切换已打开文件描述符的 `O_NONBLOCK` 标志来请求非阻塞 I/O。字符设备驱动程序可以通过检查传递给读取和写入方法的文件标志中是否存在 `O_NONBLOCK` 标志来检查是否启用了非阻塞 I/O。如果请求了非阻塞 I/O，则应返回错误 `EWOULDBLOCK`，而不是在驱动程序会阻塞时阻塞。对于回显设备，这意味着在读取和写入方法中添加额外的检查，以确保在阻塞之前处理非阻塞 I/O。这本身足以处理在打开时请求的非阻塞 I/O。然而，为了支持通过 `fcntl(2)` 切换标志，还需要额外的步骤。

每次尝试通过 `fcntl(2)` 的 `F_SETFL` 操作设置文件标志时，都会在字符设备上调用两个 I/O 控制命令：`FIONBIO` 和 `FIOASYNC`。即使请求没有更改关联的 `O_NONBLOCK` 和 `O_ASYNC` 标志的状态，也会调用这些 I/O 控制命令。如果任何一个 I/O 控制命令失败，整个 `F_SETFL` 操作将失败，文件标志保持不变。因此，想要支持 `F_SETFL` 的字符设备驱动程序必须实现对这两个 I/O 控制命令的支持。

`FIONBIO` 和 `FIOASYNC` 将一个 int 值作为命令参数。如果新文件标志中清除了关联的文件标志，则该 int 值为零；如果新文件标志中设置了关联的标志，则该 int 值为非零。I/O 控制处理程序应该在请求的标志设置被支持时返回零，或者在请求的设置不被支持时返回错误。回显设备支持设置 `O_NONBLOCK` 标志，但不支持设置 `O_ASYNC` 标志，因此，如果 int 参数非零，回显设备的 `FIOASYNC` 处理程序会失败。

列表 5 显示了支持非阻塞 I/O 的读取和 I/O 控制方法的相关更改。

**列表 5：支持非阻塞 I/O**

```c
static int
echo_read(struct cdev *dev, struct uio *uio, int ioflag)
{
      ...
      /* 等待读取字节。 */
      while (sc->valid == 0 && sc->writers != 0) {
            if (sc->dying)
                  error = ENXIO;
            else if (ioflag & O_NONBLOCK)
                  error = EWOULDBLOCK;
            else
                  error = sx_sleep(sc, &sc->lock, PCATCH, “echord”, 0);
      ...
}

...

static int
echo_ioctl(struct cdev *dev, u_long cmd, caddr_t data, int fflag,
struct thread *td)
{
      ...
      switch (cmd) {
      ...
      case FIONBIO:
            /* 支持 O_NONBLOCK*/
            error = 0;
            break;
      case FIOASYNC:
            /* 支持 O_ASYNC */
            if (*(int *)data != 0)
                  error = EINVAL;
            else
                  error = 0;
            break;
      ...
}
```

## 投票 I/O 状态

使用非阻塞 I/O 的应用程序通常使用事件循环来处理多个文件描述符的请求。每次循环迭代，应用程序会阻塞，等待一个或多个文件描述符准备好（例如，数据可供读取，或有空间写入更多数据）。然后，应用程序会处理每个已准备好的文件描述符，之后再次等待。FreeBSD 为此类事件循环支持两个系统调用：[`select(2)`](https://man.freebsd.org/select/2) 和 [`poll(2)`](https://man.freebsd.org/poll/2)。在内核中，`select(2)` 和 `poll(2)` 是使用一个公共框架实现的。每个请求的文件描述符都被单独轮询，以确定它是否准备好。如果没有文件描述符准备好，则调用系统调用的线程可以睡眠，直到至少一个文件描述符变为可用。如果文件描述符在一个线程等待时变得可用，则该文件描述符必须唤醒正在休眠的线程。

[`selrecord(9)`](https://man.freebsd.org/selrecord/9) 函数族管理线程的休眠和唤醒。支持轮询的文件描述符必须为它支持的事件类型创建一个 `struct selinfo` 对象。该对象应通过将整个对象清零来初始化（例如，使用 `memset()`）。如果文件描述符的轮询函数发现文件描述符没有准备好，它必须对每个请求的事件调用 `selrecord()`，并将关联的 `struct selinfo` 对象传递给它。每当发生一个事件可能使文件描述符准备好时，必须调用 `selwakeup()` 在该事件的 `struct selinfo` 对象上。最后，在销毁 `struct selinfo` 对象之前，应该使用 `seldrain()` 唤醒任何剩余的线程。

对于字符设备，文件描述符轮询函数调用字符设备的轮询方法。此方法接受一个 `poll(2)` 事件的位掩码作为函数参数，并必须返回当前为真的事件掩码。此外，如果没有请求的事件为真，该函数还负责调用 `selrecord()`。请注意，字符设备不支持通过读写方法提供不同类型的优先数据，仅支持正常数据。

对于回显设备，我们支持读和写事件。我们在 softc 中添加了两个 `struct selinfo` 对象，每个事件一个。由于整个 softc 在创建时已被清零，因此在初始化 softc 时不需要进一步的更改。每个读写方法都会调用 `selwakeup()` 来唤醒可能等待的线程，针对的是 *另一个* 事件。其他一些地方也可以使回显设备变为准备状态，需要调用 `selwakeup()`。如果某个 I/O 控制命令扩展了缓冲区或清除了其内容，则可能会使设备准备好进行写操作。如果最后一个写入者关闭了设备，也可以使设备准备好进行读取。一个新的轮询方法确定设备的当前状态，并根据需要调用 `selrecord()`。最后，在销毁设备时，每个事件都会调用 `seldrain()`。列表 6 显示了在读取方法中对 `selwakeup()` 的添加调用以及新的轮询方法。请注意，对于读取方法，`selwakeup()` 用于写事件。

**列表 7：设备轮询**

```c
static int
echo_read(struct cdev *dev, struct uio *uio, int ioflag)
{
      ...
      error = uiomove(sc->buf, todo, uio);
      if (error == 0) {
            /* Wakeup any waiting writers. */
            if (sc->valid == sc->len)
                  wakeup(sc);

            sc->valid -= todo;
            memmove(sc->buf, sc->buf + todo, sc->valid);
            selwakeup(&sc->wsel);
      }
      ...
}

...

static int
echo_poll(struct cdev *dev, int events, struct thread *td)
{
      struct echodev_softc *sc = dev->si_drv1;
      int revents;

      revents = 0;
      sx_slock(&sc->lock);
      if (sc->valid != 0 || sc->writers == 0)
            revents |= events & (POLLIN | POLLRDNORM);
      if (sc->valid < sc->len)
            revents |= events & (POLLOUT | POLLWRNORM);
      if (revents == 0) {
            if ((events & (POLLIN | POLLRDNORM)) != 0)
                  selrecord(td, &sc->rsel);
            if ((events & (POLLOUT | POLLWRNORM)) != 0)
                  selrecord(td, &sc->wsel);
      }
      sx_sunlock(&sc->lock);
      return (revents);
}
```

一对通用的 I/O 控制命令对于检查文件描述符的状态也很有用。`FIONREAD` 和 `FIONWRITE` 分别返回可以在不阻塞的情况下读取或写入的字节数。字节数作为类型为 `int` 的控制命令参数返回。列表 8 显示了回显设备对这些 I/O 控制命令的支持。请注意，返回的值被限制为 `INT_MAX` 以避免溢出。

**列表 8：FIONREAD 和 FIONWRITE**

```c
static int
echo_ioctl(struct cdev *dev, u_long cmd, caddr_t data, int fflag,
struct thread *td)
{
      ...
      switch (cmd) {
      ...
      case FIONREAD:
            sx_slock(&sc->lock);
            *(int *)data = MIN(INT_MAX, sc->valid);
            sx_sunlock(&sc->lock);
            error = 0;
            break;
      case FIONWRITE:
            sx_slock(&sc->lock);
            *(int *)data = MIN(INT_MAX, sc->len - sc->valid);
            sx_sunlock(&sc->lock);
            error = 0;
            break;
      ...
}
```

为了使这个功能更容易演示，我们在 `echoctl` 工具中添加了一个新的轮询命令。这个命令使用 `poll(2)` 查询回显设备的当前状态。如果设备可读，它会使用 `FIONREAD` I/O 控制命令输出可读取的字节数。如果设备可写，它会使用 `FIONWRITE` 输出可以写入的字节数。示例 3 显示了这个命令的几次调用，以及其他对回显设备的操作。请注意，由于在这个示例中没有其他写入者，设备即使为空也仍然是可读的。

**示例 3：轮询 I/O 状态**

```sh
# echoctl poll
Returned events: POLLIN|POLLOUT
0 bytes available to read
room to write 64 bytes
# echo “foo” > /dev/echo
# echoctl poll
Returned events: POLLIN|POLLOUT
4 bytes available to read
room to write 60 bytes
# cat /dev/echo
foo
# echoctl poll -r
Returned events: POLLIN
0 bytes available to read
```

## 通过 kqueue(2) 报告 I/O 状态

FreeBSD 提供了 [`kqueue(2)`](https://man.freebsd.org/kqueue/2) 内核事件通知机制，这是一个与 `select(2)` 和 `poll(2)` 独立的 API。通过 `kqueue(2)`，应用程序为每个所需事件在内核中注册一个持久性的通知。内核生成一系列事件，应用程序可以消费并采取相应的行动。与 `select(2)` 和 `poll(2)` 不同，应用程序无需每次等待新事件时都注册它关心的所有事件。这减少了应用程序的开销，同时也允许内核更高效地跟踪所需的事件。

一个内核事件由一个过滤器（事件类型）和标识符组成。某些事件的行为可以通过不同的标志进一步定制。对于文件描述符的 I/O，两个主要的过滤器是 `EVFILT_READ` 和 `EVFILT_WRITE`，分别用于判断文件描述符是否可读或可写。这些事件过滤器的标识符字段是整数文件描述符。此外，对于读写事件，内核事件结构还返回可以读取或写入的数据量，作为一个单独的字段。这避免了单独调用 `FIONREAD` 和 `FIONWRITE` I/O 控制命令的需要。

在内核中，内核事件由 `struct knote` 对象描述。此结构包含用于生成返回给应用程序的事件的事件字段副本。活动事件的列表存储在 `struct knlist` 对象中。由于 `select(2)` 和 `poll(2)` 处理的 I/O 事件通常与内核事件关联，因此 `struct selinfo` 将 `struct knlist` 作为其 `si_note` 成员嵌入其中。每个 knote 还与指向 `struct filterops` 对象的 `kn_fop` 成员相关联。这个结构以及用于操作 knote 和 knote 列表的 API 在 [`kqueue(9)`](https://man.freebsd.org/kqueue/9) 中有描述。

对于字符设备，`kqfilter` 方法负责将 `struct filterops` 对象附加到 knote 上。这包括设置 `kn_fop` 成员并将 knote 添加到正确的 knote 列表中。因此，字符设备的 `struct filterops` 对象不使用 `f_attach` 成员。`struct knote` 的 `kn_hook` 成员是一个不透明指针，`kqfilter` 方法可以设置该指针来将状态传递给 `struct filterops` 方法，类似于 `struct cdev` 中的 `si_drv1` 字段。

对于回显驱动程序，我们定义了两个 `struct filterops` 对象：一个用于读事件，另一个用于写事件。每个事件包括 `f_detach` 和 `f_event` 方法。我们重用了现有的读写 `struct selinfo` 对象中嵌入的 knote 列表。由于回显驱动程序使用了 `sx(9)` 锁，我们定义了自定义的锁回调函数，在创建回显设备时与 `knlist_init()` 一起使用。`f_detach` 方法使用 `knlist_remove()` 从关联的 knote 列表中移除 knote。`f_event` 方法将 `kn_data` 字段设置为适当的字节数，并在字节数非零时标记事件为已就绪。对于读事件的 `f_event` 方法，如果没有写入者，它还会设置 `EV_EOF`。新的 kqfilter 字符设备方法将 knote 附加到 `EVFILT_READ` 和 `EVFILT_WRITE` 过滤器的新 `struct filterops` 对象上。它还将 knote 的 `kn_hook` 成员设置为 softc 指针。最后，驱动程序中所有调用 `selrecord()` 以唤醒来自 `poll(2)` 或 `select(2)` 的睡眠线程的地方，现在还会调用 `KNOTE_LOCKED()` 来报告与读或写事件关联的 knote 的事件。列表 9 显示了用于读取过滤器的 `struct filterops` 对象及其关联方法。列表 10 显示了新的 `kqfilter` 字符设备方法。

**列表 9：EVFILT_READ 过滤器**

```c
static struct filterops echo_read_filterops = {
      .f_isfd =      1,
      .f_detach =      echo_kqread_detach,
      .f_event =      echo_kqread_event
};

...

static void
echo_kqread_detach(struct knote *kn)
{
      struct echodev_softc *sc = kn->kn_hook;

      knlist_remove(&sc->rsel.si_note, kn, 0);
}

static int
echo_kqread_event(struct knote *kn, long hint)
{
      struct echodev_softc *sc = kn->kn_hook;

      kn->kn_data = sc->valid;
      if (sc->writers == 0) {
            kn->kn_flags |= EV_EOF;
            return (1);
      }
      kn->kn_flags &= ~EV_EOF;
      return (kn->kn_data > 0);
}
```

**列表 10：kqfilter 设备方法**

```c
static int
echo_kqfilter(struct cdev *dev, struct knote *kn)
{
      struct echodev_softc *sc = dev->si_drv1;

      switch (kn->kn_filter) {
      case EVFILT_READ:
            kn->kn_fop = &echo_read_filterops;
            kn->kn_hook = sc;
            knlist_add(&sc->rsel.si_note, kn, 0);
            return (0);
      case EVFILT_WRITE:
            kn->kn_fop = &echo_write_filterops;
            kn->kn_hook = sc;
            knlist_add(&sc->wsel.si_note, kn, 0);
            return (0);
      default:
            return (EINVAL);
      }
}
```

与 `poll(2)` 支持一样，我们通过向 echoctl 工具添加另一个命令来演示 `kevent(2)` 支持。新的 events 命令为 echo 设备注册读取和写入事件，并在接收到每个事件时输出一行。由于读取和写入事件默认是按级别触发的，echoctl 在为 echo 设备注册事件时设置了 `EV_CLEAR` 标志。这样，仅在设备状态变化时才报告事件，触发驱动程序内的 `KNOTE_LOCKED()` 调用。示例 4 显示了跨一系列操作的 events 命令输出。前两个事件报告的是 echo 设备处于空闲状态时的初始状态，此时没有打开的读取器或写入器。在另一个 shell 中，我们执行命令 `jot -c -s "" 80 48 > /dev/echo` 向 echo 设备写入 81 字节数据。由于默认缓冲区大小为 64 字节，此命令在写入 64 字节后会在 `write(2)` 系统调用中阻塞。64 字节的写入触发了下一个 `EVFILT_READ` 事件，报告 64 字节可供读取。最后，在第三个 shell 中，我们执行命令 "`cat /dev/echo`" 从 echo 设备读取所有数据。`cat(1)` 的第一次 `read(2)` 系统调用读取了 64 字节输出，并触发了一个 `EVFILT_WRITE` 事件。然而，在 echoctl 进程查询 echo 设备状态之前，`jot(1)` 进程已被唤醒，并将剩余的 17 字节数据写入缓冲区，腾出了 47 字节的空间。这解释了第三个事件块中报告的第一个 `EVFILT_WRITE` 事件。写入剩余的 17 字节也触发了一个 `EVFILT_READ` 事件。然而，在此事件被报告时，`jot(1)` 进程已经退出，`cat(1)` 已经读取了剩余的 17 字节，因此 `EVFILT_READ` 事件报告 `EV_EOF`，并且没有字节可读。`cat(1)` 读取 17 字节时也触发了一个 `EVFILT_WRITE` 事件，这被报告为倒数第二个事件。最后，`cat(1)` 第三次调用 `read(2)`，返回 0 以表示 EOF。此读取还触发了一个 `EVFILT_WRITE` 事件，这是最后一个报告的事件。此最后一系列事件是非确定性的，在不同的运行中可能以不同的顺序或略有不同的值出现（例如，如果 `jot(1)` 尚未写入剩余的 17 字节，第一个 `EVFILT_WRITE` 可能报告 64 字节可写）。

**示例 4：通过内核事件报告 I/O 状态**

```sh
# echoctl events -W
EVFILT_READ: EV_EOF 0 bytes
EVFILT_WRITE: 64 bytes
...
EVFILT_READ: 64 bytes
...
EVFILT_WRITE: 47 bytes
EVFILT_READ: EV_EOF 0 bytes
EVFILT_WRITE: 64 bytes
EVFILT_WRITE: 64 bytes
```

## 结论

在本文中，我们扩展了 echo 设备，以支持具有阻塞和非阻塞 I/O 的 FIFO 数据缓冲区。我们还添加了通过 `poll(2)` 和 `kevent(2)` 查询设备状态的功能。本系列的最后一篇文章将描述字符设备如何为通过 [`mmap(2)`](https://man.freebsd.org/mmap/2) 创建的内存映射提供后备存储。

---

John Baldwin 是一名系统软件开发人员。他在过去二十多年中，直接向 FreeBSD 操作系统提交了多项更改，涵盖了内核的各个部分（包括 x86 平台支持、SMP、各种设备驱动程序和虚拟内存子系统）以及用户空间程序。除了编写代码外，John 还曾在 FreeBSD 核心团队和发布工程团队任职。他还为 GDB 调试器做出了贡献。John 与妻子 Kimberly 和三个孩子 Janelle、Evan 和 Bella 一起居住在弗吉尼亚州的阿什兰。
