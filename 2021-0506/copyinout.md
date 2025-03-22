# copyinout 框架

——FreeBSD 和 CheriBSD 中的 ABI 独立、类型感知、能力感知、copyin 和 copyout API

- 原文链接：[The copyinout Framework](https://freebsdfoundation.org/wp-content/uploads/2021/07/The-copyinout-Framework.pdf)
- 作者：**KONRAD WITASZCZYK**

在用户空间和内核空间之间的内存复制是系统调用中的一个关键操作。它用于复制系统调用的参数以及系统调用的结果。目前的复制函数原型是与类型无关的，复制一定数量的字节从任意缓冲区。当内核将其内存复制到用户空间时，必须确保不会泄露任何可能包含机密数据的内核数据。本文概述了 FreeBSD 和 CheriBSD 中当前复制函数的局限性，并提出了一个框架，可以提高系统调用处理程序的安全性和代码质量。

所述的 copyinout 框架是作为题为《能力感知内存复制在地址空间之间》的硕士论文的一部分实现的，该论文在哥本哈根大学的 Ken Friis Larsen 和微软研究院的 David Chisnall 的指导下完成。类型感知的 copyin 和 copyout API 的最初构想是由 David Chisnall 提出的。

## FreeBSD 中的内存复制函数

FreeBSD 包含两个主要函数，用于在地址空间之间复制内存：copyin() 和 copyout()（见**清单** 1）。这两个函数接受三个参数：源地址、目标地址和要复制的字节数。copyin() 将 len 字节从用户空间地址 udaddr 复制到内核空间地址 kaddr。copyout() 方向相反，将 len 字节从内核空间地址 kaddr 复制到用户空间地址 udaddr。这些函数在成功时返回 0，如果传递了无效地址，则返回 EFAULT。

```c
int copyin(const void * __restrict udaddr, void * _Nonnull __restrict kaddr, size_t len);
int copyout(const void * _Nonnull __restrict kaddr, void * __restrict udaddr, size_t len);
```

**清单** 1.** FreeBSD 13.0-RELEASE 中的 copyin() 和 copyout() 函数原型

从函数原型中，我们可以识别出一个潜在的安全问题。复制函数操作的是任意缓冲区。如果缓冲区包含一个结构体对象，而该结构体字段之间有填充，那么填充部分也会被复制。包含敏感信息的填充泄露通常被称为内核内存泄露或内核内存泄漏。这类漏洞可能导致特权提升。它们不仅仅是 FreeBSD 特有的，许多操作系统中都可以发现类似问题 [3] [4] [5]，并且它们已成为广泛的检测 [6] 和缓解 [7] [8] [9] 技术研究的主题。

复制函数被系统调用处理程序使用，用于将系统调用参数从用户空间复制到内核空间，并将系统调用结果从内核空间复制到用户空间。由于系统调用非常频繁地执行，内核开发人员必须提供最低开销的复制函数实现。这可以通过提供与机器相关的汇编语言实现来实现。根据 CPU 架构，复制函数还可以利用 CPU 模型提供的安全功能。例如，针对 amd64（见 amd64/amd64/support.S）的实现支持监督模式访问防护（SMAP）。

## ABI 支持

FreeBSD 支持多种 ABI。具体而言：

* 针对与内核编译目标相同的程序的本地 ABI；
* 针对内核编译的架构的 32 位版本的 32 位 ABI；
* 针对 Linux 用户空间程序的 Linux ABI

这些 ABI 作为兼容层实现。每个兼容层提供其系统调用处理程序，这些处理程序实现了在内核进入或返回内核例程时，必须执行的额外逻辑。这包括在地址空间之间复制和转换对象。例如，对于 32 位 ABI，系统调用参数或系统调用结果中的指针必须从 32 位指针转换或转换为 32 位指针。

## 系统调用处理程序

每当用户空间程序发出系统调用时，内核会调用一个特定的系统调用处理程序，并将已复制的系统调用参数传入。对于每个支持的 ABI，内核保持一个 sysentvec 结构对象（见**清单** 2），描述 ABI 特定的属性和在程序执行时内核将使用的函数。该结构包括包含系统调用处理程序函数指针的 sv_table 数组，这些指针存储在由相应的系统调用编号指定的位置，以及指向一个架构特定函数的 sv_fetch_syscall_args 函数指针，该函数用于复制系统调用参数。

作为示例，我们来考虑 jail(2) 系统调用。这个系统调用有一个参数：指向 jail 结构对象的指针（见**清单** 3）。jail 结构包含描述 jail 的参数（见**清单** 4）。一旦用户空间程序执行 jail 系统调用并进入特权模式，内核将调用 jail 系统调用处理程序——sys_jail() 函数（见**清单** 5），并将已经复制进来的 jail 系统调用参数——jail_args 结构对象传入。系统调用处理程序复制一个 jail 结构，并调用实现 jail 系统调用逻辑的 kern_jail 内核例程。

```c
struct sysentvec {
 (...)
 struct sysent *sv_table;
 (...)
 int (*sv_fetch_syscall_args)(struct thread *);
 (...)
}
```

**清单** 2. 描述 ABI 特定属性和函数的 sysentvec 结构。

```c
struct jail_args {
 char jail_l_[PADL_(struct jail *)];
 struct jail * jail;
 char jail_r_[PADR_(struct jail *)];
};
```

**清单** 3. 本地 ABI 的 jail 系统调用参数结构。

```c
struct jail {
 uint32_t version;
 char *path;
 char *hostname;
 char *jailname;
 uint32_t ip4s;
 uint32_t ip6s;
 struct in_addr *ip4;
 struct in6_addr *ip6;
};

```

**清单** 4. 本地 ABI 的 jail 结构。

```c
int
sys_jail(struct thread *td, struct jail_args *uap)
{
 (...)
 int error;
 struct jail j;
 (...)
 error = copyin(uap->jail, &j, sizeof(struct jail));
 if (error)
 return (error);
 (...)
 return (kern_jail(td, &j));
}
```

**清单** 5. 本地 ABI 的 jail 系统调用处理程序。

## 兼容层

兼容层为非本地 ABI 提供系统调用实现。特别地，32 位 ABI 被实现为 freebsd32 兼容层。让我们考虑 32 位 ABI 下相同的 jail 系统调用。32 位版本的 jail 参数结构称为 freebsd32_jail_args（见**清单** 6），它包括指向 32 位版本的 jail 结构（称为 jail32，见**清单** 7）的指针。jail 和 jail32 结构之间的唯一区别是架构无关的字段类型。每个指针都被替换为 32 位无符号整数。由于指针在 32 位架构中是 32 位无符号整数，这一变化保证了为 64 位内核编译的 jail32 结构与为 32 位内核编译的 jail 结构具有相同的布局。

32 位 ABI 的 jail 系统调用处理程序实现为 freebsd32_jail 函数（见**清单** 8）。该函数复制一个 32 位的 jail 对象，使用宏（见**清单** 9）为其本地 ABI 版本翻译每个字段，并调用与本地 ABI 情况下相同的 kern_jail 内核例程。这意味着本地 ABI 和 32 位 ABI 在 jail 系统调用处理程序中的唯一区别是将用户空间的 jail 对象转换为可以被内核使用的本地 ABI 版本。

```c
struct freebsd32_jail_args {
 char jail_l_[PADL_(struct jail32 *)];
 struct jail32 * jail;
 char jail_r_[PADR_(struct jail32 *)];
};
```

**清单** 6. 32 位 ABI 的 jail 系统调用参数结构。

```c
struct jail32 {
 uint32_t version;
 uint32_t path;
 uint32_t hostname;
 uint32_t jailname;
 uint32_t ip4s;
 uint32_t ip6s;
 uint32_t ip4;
 uint32_t ip6;
};
```

**清单** 7. 32 位 ABI 的 jail 结构。

```c
int
freebsd32_jail(struct thread *td, struct freebsd32_jail_args *uap)
{
 (...)
 int error;
 struct jail j;
 (...)
 struct jail32 j32;
 error = copyin(uap->jail, &j32, sizeof(struct jail32));
 if (error)
 return (error);
 CP(j32, j, version);
 PTRIN_CP(j32, j, path);
 PTRIN_CP(j32, j, hostname);
 PTRIN_CP(j32, j, jailname);
 CP(j32, j, ip4s);
 CP(j32, j, ip6s);
 PTRIN_CP(j32, j, ip4);
 PTRIN_CP(j32, j, ip6);
 (...)
 return (kern_jail(td, &j));
}
```

**清单** 8. 32 位 ABI 的 jail 系统调用处理程序。

```c
#define CP(src, dst, fld) do { \
 (dst).fld = (src).fld; \
} while (0)
#define PTRIN(v) (void *)(uintptr_t)(v)
#define PTRIN_CP(src, dst, fld) do { \
 (dst).fld = PTRIN((src).fld); \
} while (0)
```

**清单** 9. 32 位 ABI 的 jail 系统调用处理程序使用的辅助宏

## copyinout API

前面描述的内核内存泄漏和代码重复问题是由于 copy 函数缺乏类型意识引起的。如果 copyin() 和 copyout() 函数能理解存储在复制缓冲区中的底层对象的结构，它们就能复制对象的字段并在用户进程 ABI 与内核 ABI 不同的情况下进行转换。

为了解决这些问题，我们引入了 copyinout API。对于每个在用户空间和内核空间之间复制的类型 foo，我们引入了类型感知的 copy 函数变体（见**清单** 10），它们会复制类型 foo 的每个字段，并根据源 ABI 转换为目标 ABI，例如，将 32 位指针转换为 64 位指针，适用于 64 位架构上的 32 位进程。此外，copy 函数还可以根据 CPU 型号执行额外的操作，例如，针对 CHERI CPU 编译的内核可以创建 CHERI 能力来为字段设置边界或权限。与原始 copy 函数不同，类型感知的 copy 函数只需要两个参数：源地址和目标地址。copyin_foo() 将存储在 uaddr 用户空间地址的对象复制到存储在 kaddr 内核空间地址的对象。copyout_foo() 以相反的方式工作，将存储在 kaddr 内核空间地址的对象复制到存储在 uaddr 用户空间地址的对象。例如，前面描述的 jail 结构可以使用以下函数调用进行复制：

```c
copyin_jail(uap->jail, &j);
```

```c
int copyin_foo(const void *uaddr, struct foo *kaddr);
int copyout_foo(const struct foo *kaddr, const void *uaddr);
```

**清单** 10. 类型 foo 的 copyin() 和 copyout() 函数变体

## copyinout 框架

类型感知的 copy 函数的实现与 copyinout API 本身是独立的。为了提供这些实现，我们引入了 copyinout 框架，该框架由以下部分组成：

* 描述应该为某个类型生成哪些 copy 函数的类型注解；
* 每个 ABI 的 copy 函数指针表；
* 一个内核接口，用于动态注册 copy 函数并调用它们；
* 一个代码生成工具，用于根据类型注解生成 copy 函数

内核定义了类型注解，指示应该为某个类型生成什么样的 copy 函数：

* `__copyin` 生成 copyin();
* `__copyout` 生成 copyout();
* `__copyinout` 生成 copyin() 和 copyout()。

此外，内核定义了字段注解，描述字段中存储的值：

* `__uaddr_array(bar)` 用于存储指向数组的指针，该数组的元素数量存储在字段 bar 中；
* `__uaddr_bounded(bar)` 用于存储指向缓冲区的指针，该缓冲区的字节数存储在字段 bar 中；
* `__uaddr_code` 用于存储代码指针的字段；
* `__uaddr_object` 用于存储指向对象的指针的字段；
* `__uaddr_unbounded` 用于存储指向具有未知字节数的缓冲区的指针的字段；
* `__uaddr_str` 用于存储指向字符串的指针的字段，因此可以通过 `strlen()` 来计算其边界。

字段注解可以用于生成利用安全相关机制的 copy 函数，例如构建 CHERI 能力。通过集成 copyinout 框架，内核开发者只需为**清单** 11 中定义的类型 foo 添加适当的注解并运行代码生成工具，就可以生成新的 copy 函数。在这种情况下，生成的 copy 函数复制字段 `len` 并翻译存储在指针数组中的指针。此外，在 CheriBSD 中，它们构建了带有边界的能力数组，边界值设置为存储在字段 `len` 中的值，作为字段翻译的一部分。

```c
struct foo {
size_t len;
int *array;
};
```

```c
struct foo {
size_t len;
__uaddr_array(len) int * __capability array;
} __copyinout;
```

**清单** 11. 类型 foo 在 copyinout 变更前后的定义。

为了为不同的 ABIs 提供单独的 copy 函数，copyinout 框架实现了 copyinout 表，该表包含特定类型的 copyin() 和 copyout() 函数指针（见**清单** 12），作为 sysentvec 结构的一部分（见**清单** 13，与**清单** 2 对比）。每个 ABI 分配自己的 copyinout 表，并通过 SYSINIT 框架使用 Linker Set 技术动态填充条目 [10]。这些 ABIs 的 SYSINIT() 和 SYSUNINIT() 宏调用与生成的 copy 函数一起由代码生成工具生成。这允许生成用于内核模块中的 copy 函数，并在模块加载和卸载时注册和注销。通过 copyinout 表，copyinout API 可以定义为内核中的宏，这些宏会强制转换指针并调用与当前正在运行的线程 ABI 对应的 copyinout 表中的函数（见**清单** 14）。特定函数的 copyinout 表条目索引是一个全局变量，在 SYSINIT() 调用过程中初始化。例如，类型 foo 及其生成的函数具有一个关联变量 `copyinout_foo_idx`，该变量可以通过 `COPYIN_CALL()` 和 `COPYOUT_CALL()` 宏间接使用，以确定函数地址并进行函数调用（见**清单** 15）

```c
typedef int copyin_t(const void * __capability uaddr, void * __capability kaddr);
typedef int copyout_t(const void * __capability kaddr, void * __capability uaddr);
struct copyinout {
 copyin_t *ce_copyin;
 copyout_t *ce_copyout;
};
```

**清单** 12. 包含特定类型的 copy 函数指针的 copyinout 结构。

```c
struct sysentvec {
 (...)
 struct sysent *sv_table;
 (...)
 int (*sv_fetch_syscall_args)(struct thread *);
 (...)
 const struct copyinout *sv_copyinout;
}
```

**清单** 13. 带有 copyinout 表的 sysentvec 结构。

```c
#define THREAD_COPYINOUT(thread, type) \
 (thread)->td_proc->p_sysent->sv_copyinout[copyinout_##type##_idx]
#define COPYIN_FUN(type) \
 THREAD_COPYINOUT(curthread, type).ce_copyin
#define COPYIN_CALL(type, uaddr, kaddr) \
 ((int (*)(const void * __capability, \
 struct type * __capability))COPYIN_FUN(type)) \
 (uaddr, kaddr)
#define COPYOUT_FUN(type) \
 THREAD_COPYINOUT(curthread, type).ce_copyout
#define COPYOUT_CALL(type, kaddr, uaddr) \
 ((int (*)(const struct type * __capability, \
 void * __capability))COPYOUT_FUN(type)) \
 (kaddr, uaddr)
```

**清单** 14. 用于 copyinout API 调用的内核宏。

```c
#define copyin_foo(uaddr, kaddr) \
 COPYIN_CALL(foo, uaddr, kaddr)
#define copyout_foo(kaddr, uaddr) \
 COPYOUT_CALL(foo, kaddr, uaddr)
```

**清单** 15. 类型 foo 的 copyinout API 函数调用宏。

copyinout 框架的主要部分是一个用 C++ 编写的代码生成工具，使用 libclang 库进行代码分析。它遍历包含所有应生成 copy 函数的类型定义的头文件输入 Clang AST 树，并打印 copy 函数的原型、定义或实现（见**清单** 16）。AST 树可以通过以下命令生成：

```sh
clang -Xclang -ast-dump -fsyntax-only -c header-file
```

copyinout 框架包含一个辅助脚本，该脚本为输入的基础源代码树生成 AST 树，运行 copyinout 工具，并将生成的函数放入基础源代码树中。这个脚本可以被添加到 FreeBSD 构建系统中，以实现代码生成的自动化。

目前，函数实现可以为任何架构（通用）生成 C 语言代码，或者为 MIPS 和 CHERI-MIPS 架构生成汇编语言代码。例如，**清单** 17 展示了由 copyinout 工具为类型 foo（见**清单** 11）和原生 ABI（一个混合的 CheriBSD 内核）生成的汇编语言版本的 copyin() 函数。

```sh
$ ./copyinout
用法：copyinout 原型 kernel-space-ast
 copyinout 定义 native|freebsd32|cheri kernel-space-ast
 copyinout 实现 native|freebsd32|cheri generic|mips user-space-ast
kernel-space-ast
```

**清单** 16. copyinout 工具的用法。

```x
struct foo {
 size_t len;
 __uaddr_array(len) int * __capability array; |
} __copyinout;
```

```c
LEAF(native_copyin_foo)
 cgetbase t0 , $c3
 blt t0, zero, _C_LABEL(copyerr)
 nop
 GET_CPU_PCPU(v1)
 PTR_L t0, PC_CURPCB(v1)
 PTR_LA t1 , copyerr
PTR_S t1, U_PCB_ONFAULT(t0)
cld v0, zero, 0($c3)
 PTR_S zero, U_PCB_ONFAULT(t0)
 csd v0, zero, 0($c4)

PTR_S t1, U_PCB_ONFAULT(t0)
 cld v0, zero, 8($c3)
 cfromptr $c5 , $ddc , v0
 cld v0, zero, 0($c3)
 li a0, 96
 multu a0, v0
 mflo v0
 csetbounds $c5 , $c5 , v0
 PTR_S zero, U_PCB_ONFAULT(t0)
 csc $c5, zero, 16($c4)

 j ra
 move v0, zero
END(native_copyin_foo)
```

**清单** 17. 为类型 foo 和本地 ABI（混合 CheriBSD 内核）生成的 copyin() 函数。

## 使用 copyinout API 进行内存拷贝

有了 copyinout 框架，我们可以简化系统调用结构和处理程序。对于之前讨论的 jail 系统调用，我们可以用一个单一的结构体替代 jail（见 **清单** 4）和 jail32（见 **清单** 7）结构体（见 **清单** 18）。对于本地 ABI 的 jail 系统调用（见 **清单** 5），我们可以修改为使用类型感知的 copyin() 函数变体 copyin_jail()（见 **清单** 19）。由于 copyin_jail() 函数也是 ABI 独立的，并且可以自动将 32 位 ABI 对象转换为其本地 ABI 版本，我们还可以修改 32 位 ABI 的 jail 系统调用处理程序（见 **清单** 8），直接调用 sys_jail() 函数（见 **清单** 20）。

```c
struct jail {
 uint32_t version;
 __uaddr_str char * __capability path;
 __uaddr_str char * __capability hostname;
 __uaddr_str char * __capability jailname;
 uint32_t ip4s;
 uint32_t ip6s;
 __uaddr_array(ip4s) struct in_addr * __capability ip4;
 __uaddr_array(ip6s) struct in6_addr * __capability ip6;
} __copyinout;
```

**清单** 18. 对所有 ABI 应用 copyinout 变更后的 jail 结构。

```c
int
sys_jail(struct thread *td, struct jail_args *uap)
{
 (...)
 int error;
 struct jail j;
 (...)
 error = copyin_jail(uap->jail, &j);
 if (error)
 return (error);
 (...)
 return (kern_jail(td, &j));
}
```

**清单** 19. 对本地 ABI 应用 copyinout 变更后的 jail 系统调用处理程序。

```c
int
freebsd32_jail(struct thread *td, struct freebsd32_jail_args *uap)
{
 struct jail_args args;
 args.jailp = uap->jailp;
 return (sys_jail(td, &args));
}
```

**清单** 20. 对 32 位 ABI 应用 copyinout 变更后的 jail 系统调用处理程序。

## 结论

copyinout 框架的初步实现表明，生成拷贝函数可以提高系统调用处理程序的代码质量，并消除填充中的潜在内核内存泄漏。然而，代码生成工具必须改进，以支持更复杂的数据类型，并为所有支持的平台生成汇编拷贝函数实现。虽然框架的工作已经暂停了一段时间，但我们希望能够尽快恢复并实施这些改进。

## 后续工作

除了 copyinout 框架的基本功能外，还有几个想法可以在未来改进或应用它：

* 汇编拷贝函数实现优化；

  当前的汇编实现没有使用任何优化技术来最小化在拷贝过程中使用的寄存器数量或周期。例如，两个相邻的半字字段可以通过一条加载指令和一个寄存器加载为一个字，而不是使用两条加载指令。

* 系统调用处理程序的简化；

  兼容层实现了许多系统调用处理程序，这些处理程序只是将对象在 ABI 之间转换，并没有为它们的本地 ABI 对应项引入任何额外的逻辑。将 copyinout API 应用于这些处理程序后，系统调用处理程序似乎是多余的。例如，32 位 ABI 的 jail 系统调用处理程序（见 **清单** 20）仅将指向 jail 对象的指针从 jail 系统调用参数结构中复制，并调用本地 ABI 的 jail 系统调用处理程序。将 copyinout 框架应用于系统调用参数结构并从兼容层中删除这些系统调用处理程序将是一个有趣的尝试。

* 面向仿真器的跨平台兼容层。

  在 QEMU 中，用户模式仿真允许在与主机架构不同的架构上运行程序，而不需要完全的系统仿真。每次遇到系统调用时，QEMU 会将系统调用参数从仿真 ABI 版本转换为主机用户空间 ABI 版本，在主机上执行系统调用，并将结果转换回其仿真 ABI 版本。我们可以研究是否可以实现一个兼容层，包含针对仿真平台的系统调用处理程序，并通过 copyinout 框架直接从仿真用户空间复制和翻译系统调用对象。

## 参考文献

[1] CTSRD. CheriBSD. FreeBSD 适配 CHERI-MIPS、CHERI-RISC-V 和 Arm Morello. https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/cheribsd.html.

[2] Konrad Witaszczyk. 跨地址空间的能力感知内存拷贝。哥本哈根大学，2019 年。

[3] The FreeBSD Project. FreeBSD-EN-18:12.mem. https://www.freebsd.org/security/advisories/FreeBSD-EN-18:12.mem.asc.

[4] The MITRE Corporation. CVE-2017-16994. https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-16994.

[5] The MITRE Corporation. CVE-2010-4082. https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-4082.

[6] Mateusz Jurczyk. 使用 x86 仿真和污点追踪检测内核内存泄漏。https://googleprojectzero.blogspot.com/2018/06/detecting-kernel-memory-disclosure.html.

[7] Alexander Popov. 引入 STACKLEAK 功能并为其添加测试。https://lwn.net/Articles/735584/.

[8] Kees Cook. mm: 强化用户拷贝。https://lwn.net/Articles/693745/.

[9] Thomas Barabosch, Maxime Villard. KLEAK: 实用的内核内存泄漏检测。https://www.netbsd.org/gallery/presentations/maxv/kleak.pdf.

[10] The FreeBSD Project. FreeBSD 架构手册，第 5 章。SYSINIT 框架。https://docs.freebsd.org/en/books/arch-handbook/sysinit/.

[11] Robert N. M. Watson 等人。能力硬件增强 RISC 指令：CHERI 指令集架构（第 8 版）。技术报告 UCAM-CL-TR-951，剑桥大学计算机实验室，2020 年。

[12] Robert N. M. Watson 等人。CHERI 简介。技术报告 UCAM-CL-TR-941，剑桥大学计算机实验室，2019 年。

---

**KONRAD WITASZCZYK** 是剑桥大学的研究员，参与 CHERI 项目。他毕业于雅盖隆大学，取得理论计算机科学学士学位，取得哥本哈根大学计算机科学硕士学位，并在 Fudo Security 工作了近 7 年，致力于 FreeBSD 及其与安全相关的技术。目前，Konrad 住在波兰华沙。
