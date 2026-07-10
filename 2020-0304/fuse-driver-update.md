# FUSE 驱动更新

你在 FreeBSD 或 Linux 笔记本上挂载过 NTFS 格式的硬盘吗？或者把 PTP 相机连到笔记本，通过文件浏览器翻看过照片？也许你还用过 MooseFS、GlusterFS 或 CephFS 这样的分布式文件系统？只要答过一个是，你就用过 FUSE。

## FUSE 是什么？

FUSE（Filesystem in USErspace，用户空间文件系统）是一种驱动程序和协议，允许用户空间进程实现文件系统，内核把它像对待其他文件系统一样呈现给其他进程。运行在用户空间让文件系统的开发和调试比内核态文件系统容易得多。NTFS 等文件系统因此使用 FUSE。用户空间程序还能访问内核态通常无法使用的库和工具，从而支持 sshfs（通过 SSH 连接挂载远程服务器的文件）和 encfs（以普通文件系统作为后端存储的加密文件系统）这类虚拟文件系统。最后，FUSE 为内核模块带来许可证上的灵活性。例如，GPLv3 内核模块无法与 CDDL 或 GPLv2 模块共存，否则无法合法再分发。但 GPLv3 的 FUSE 文件系统不是内核的派生作品，因此完全可以再分发。

FUSE 最初于 2005 年为 Linux 编写，首次出现在 2.6.14 内核版本中。它实在太好用，移植版本很快涌现。如今 OS X、OpenBSD、Illumos、Minix，当然还有 FreeBSD 都支持 FUSE。NetBSD 不使用 FUSE 协议，但通过用户空间兼容层仍支持许多 FUSE 文件系统。DragonflyBSD 的 FUSE 驱动正在开发中。

用户空间
内核空间
应用程序
VFS
UFS
ZFS
FUSEFS
libfuse
FUSE 守护进程

## FreeBSD 的 FUSE 移植

FreeBSD 的 FUSE 驱动最初是 Csaba Henk 在 2005 年 谷歌 编程之夏（GSoC）项目中编写的，但未进入基本系统。Ilya Putsikau 在 2011 年的另一个 GSoC 项目中完成了移植，Attilio Rao 随即将其合并。然而，2011 年版本仍落后于当时的协议若干年，且有一些未解决的 bug。此后 8 年间，许多 bug 未得到处理，缺乏维护和新特性，直到现在。

FUSE

驱动更新

FUSE API 栈

得益于基金会的赞助，我得以改变这一局面。我基本重写了驱动，更新了协议版本，修复了数十个 bug，并新增了若干特性和性能改进。

## 使用 FUSE 文件系统

使用 FUSE 文件系统极其简单——这正是 FUSE 的意义所在。一旦挂载，就能像普通文件系统一样访问。不同文件系统的挂载命令各异。例如挂载 ext2 文件系统：

```sh
sudo pkg install -y fusefs-lkl e2fsprogs
truncate -s 1g /tmp/ext2.img
mkfs.ext2 /tmp/ext2.img
mkdir /tmp/mnt
lklfuse -o type=ext2 /tmp/ext2.img /tmp/mnt
```

注意第二条命令里没有 `sudo`。这是有意为之。要启用此行为，把 `vfs.usermount` sysctl 设为 1。FUSE 守护进程可以无特权运行。此时挂载点只能由运行守护进程的用户访问，防止守护进程的用户窥探其他用户的 I/O。事实上，许多 FUSE 守护进程在这种模式下不做任何显式权限检查，允许挂载用户对文件系统几乎为所欲为，比如这样：

```sh
install -m 755 -o root -g wheel -d /tmp/mnt/bin
install -m 755 -o root -g wheel /bin/sh /tmp/mnt/bin/sh
```

无特权用户竟能创建 root 拥有的文件！乍看之下是个大安全漏洞。其实没问题，因为其他用户根本无法访问该文件系统。敏锐而多疑的读者可能会问：挂载用户能否创建 SUID 文件并借此提权？放心——他做不到。无特权挂载会自动设置 `nosuid`。实际发生的只是用户在修改自己拥有的 `/tmp/ext2.img`。这个特性很酷，比如可以用来创建完整的可启动镜像，用于嵌入式系统。

当然，这只是用法之一。更传统的挂载方式也可行。例如，某文件系统在当前运行的内核中不可用时，可以 root 身份用 `allow_other` 和 `default_permissions` 选项挂载 FUSE 实现。这样所有用户都能使用，行为和任何其他文件系统一样：

```sh
umount /tmp/mnt
sudo lklfuse -o type=ext2,allow_other,default_permissions \
/tmp/ext2.img /tmp/mnt
```

## 开发 FUSE 文件系统

相比内核态文件系统，为 FUSE 开发文件系统容易得多。不仅写起来容易，可移植地写也很容易。多数 FUSE 文件系统几乎无需改动就能在多个操作系统上运行。

用户态编程的好处之一是不限于 C 语言。事实上，FUSE 绑定覆盖 Perl、Python、Rust、Javascript、Java、Ruby、Nim、C#、Go，可能还有其他语言。

启动 FUSE 守护进程略显复杂：守护进程先打开 `/dev/fuse`，再调用 `nmount` 并把该文件描述符作为参数之一传入。然后开始从同一文件描述符读取 FUSE 请求并写回响应。不过开发者很少需要操心这些细节，因为 `libfuse` 全包了。无论用 C 还是其他语言，开发者通常只需为每个支持的 FUSE 操作定义回调。剩下的接线和管道工作交给库。比如 Python 中 "Hello World" 示例的核心代码仅 37 行（完整示例见 <https://github.com/libfuse/python-fuse/blob/master/example/hello.py>）。

```python
class HelloFS(Fuse):
    def getattr(self, path):
        st = MyStat()
        if path == '/':
            st.st_mode = stat.S_IFDIR | 0o755
            st.st_nlink = 2
        elif path == hello_path:
            st.st_mode = stat.S_IFREG | 0o444
            st.st_nlink = 1
            st.st_size = len(hello_str)
        else:
            return -errno.ENOENT
        return st
    def readdir(self, path, offset):
        for r in  '.', '..', hello_path[1:]:
            yield fuse.Direntry(r)
    def open(self, path, flags):
        if path != hello_path:
            return -errno.ENOENT
        accmode = os.O_RDONLY | os.O_WRONLY | os.O_RDWR
        if (flags & accmode) != os.O_RDONLY:
            return -errno.EACCES
    def read(self, path, size, offset):
        if path != hello_path:
            return -errno.ENOENT
        slen = len(hello_str)
        if offset < slen:
            if offset + size > slen:
                size = slen - offset
            buf = hello_str[offset:offset+size]
        else:
            buf = b''
        return buf
```

FUSE 的安全模型可能出乎意料：默认情况下，守护进程负责授权所有操作。这确实给 FUSE 文件系统开发者带来额外负担。但只要始终使用 `default_permissions` 挂载选项就能达到通常的效果。好处是 FUSE 文件系统可以使用奇特的授权策略，比如定制的 ACL 格式。对于在服务器端而非客户端做授权的网络文件系统，这个特性也非常有用。

## 新特性

FreeBSD 13 全新的 `fusefs(5)` 驱动新增若干特性，现有 FUSE 文件系统应能立即受益。最有趣的几项如下：

- 内核侧权限检查（`-o default_permissions`）现已完全实现
- 支持 `mknod(2)`、`pipe(2)`、`socket(2)`，可在 fusefs 文件系统上创建任意类型的文件
- 新增 `fcntl(2)` 建议锁的服务器端支持。此前 fcntl 锁总是在内核中实现，对于做分布式锁的网络文件系统并不足够
- 以 `-o intr` 挂载时，若服务器支持，fusefs 挂载点现可完全中断。即信号可以打断因等待服务器响应而阻塞的操作。类似 NFS 同名挂载选项
- fusefs 挂载点现可通过 NFS 导出
- 服务器允许时，内核现会缓存文件名和属性。服务器也可异步淘汰部分内核缓存。服务器允许时，内核还能缓存读写。最后，进程看起来在顺序读取时，会执行预读。这些特性无需应用或配置改动即可提升性能

ALAN SOMERS 自 2013 年起担任 FreeBSD 提交者。2019 年，他在 FreeBSD 基金会委托下重写了 fusefs 驱动。目前 Alan 在 Axcient 从事 FreeBSD 存储服务器相关工作。
