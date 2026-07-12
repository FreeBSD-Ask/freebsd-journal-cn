# 实现系统控制节点（sysctl）

作者：John Baldwin

FreeBSD 内核提供一组系统控制节点，可用于查询和设置状态信息。这些节点可用于获取各种统计信息和配置参数。每个节点接受和提供的信息可使用多种类型，包括整数、字符串和结构。

## 节点

节点组织为树状结构。每个节点在其所在层级内分配一个唯一编号。节点在内部由一个节点编号数组标识，该数组从根遍历到所请求的节点。

树中大多数节点都有名称，命名节点可由一串以点分隔的名称标识。例如，有一个名为“kern”的顶层节点，编号为一（1），它包含一个名为“ostype”的子节点，编号为一（1）。此节点的地址是 1.1，但也可用其全名“kern.ostype”引用。用户和应用程序通常使用节点名而非节点编号。

## 访问系统控制节点

FreeBSD 的标准 C 库提供若干访问系统控制节点的例程。**sysctl(3)** 和 **sysctlbyname(3)** 函数访问单个节点。**sysctl(3)** 使用内部编号数组标识节点，而 **sysctlbyname(3)** 接受包含节点全名的字符串。每次 sysctl 访问可以查询节点当前状态、设置节点状态，或同时执行两种操作。

**sysctlnametomib(3)** 函数将节点全名映射到其内部地址。此操作使用一个内部 sysctl 节点，开销稍大，因此频繁查询某控制节点的程序可使用此例程缓存节点地址，然后用 **sysctl(3)** 而非 **sysctlbyname(3)** 查询节点。

某些控制节点具有命名前缀和未命名的叶子。例如“kern.proc.pid”节点。它为每个进程包含一个子节点。给定进程节点的内部地址由“kern.proc.pid”的地址和对应进程 pid 的第四个数字组成。

示例 1 演示了使用此机制获取当前进程信息。

```c
struct kinfo_proc kp;
int i, mib[4];
size_t len;

/* 获取 "kern.proc.pid" 前缀的地址。 */
len = 3;
sysctlnametomib("kern.proc.pid", mib, &len);

/* 获取当前进程的进程信息。 */
len = sizeof(kp);
mib[3] = getpid();
sysctl(mib, 4, &kp, &len, NULL, 0);
```

**示例 1**

## 简单控制节点

在内核中，`<sys/sysctl.h>` 头文件提供若干宏来声明控制节点。每个声明包含父节点名、分配给此节点的编号、节点名、控制节点行为的标志，以及节点描述。某些声明需要额外参数。父节点用其全名标识，但以单下划线为前缀，点替换为下划线。例如，“foo.bar”父节点用“_foo_bar”标识。要声明顶层节点，使用空父名。编号应使用宏 `OID_AUTO` 请求系统分配唯一编号。（某些节点因历史原因使用硬编码编号，但所有新节点应使用系统分配的编号。）flags 参数必须指明节点支持的访问类型（读、写或两者），也可包含其他可选标志。描述应是简短描述节点的字符串。当向 **sysctl(8)** 传递标志 `-d` 时，会显示此描述而非值。

## 整数节点

最简单和最常见的控制节点是控制单个整数的叶子节点。此类节点由宏 `SYSCTL_INT` 定义。它接受两个额外参数：一个指针和一个值。如果指针非 NULL，应指向一个将由控制节点读写（如 flags 参数所指定）的整数变量。如果指针为 NULL，则节点必须是只读节点，读取时返回值参数。

```c
SYSCTL_INT(_kern, OID_AUTO, one, CTLFLAG_RD,
    NULL, 1, "Always returns one");

int frob = 500;
SYSCTL_INT(_kern, OID_AUTO, frob, CTLFLAG_RW,
    &frob, 0, "The \"frob\" variable");
```

**示例 2**

示例 2 定义了两个整数 sysctl 节点：`kern.one` 是只读节点，始终返回值一；`kern.frob` 是读写节点，读写全局“frob”整数的值。

其他宏可用于若干整数类型，包括：`SYSCTL_UINT` 用于无符号整数，`SYSCTL_LONG` 用于有符号长整数，`SYSCTL_ULONG` 用于无符号长整数，`SYSCTL_QUAD` 用于有符号 64 位整数，`SYSCTL_UQUAD` 用于 64 位无符号整数，`SYSCTL_COUNTER_U64` 用于由 **counter(9)** API 管理的 64 位无符号整数。只有宏 `SYSCTL_INT` 和 `SYSCTL_UINT` 可用于 NULL 指针。其他宏需要非 NULL 指针并忽略值参数。

## 其他节点类型

宏 `SYSCTL_STRING` 用于定义字符串值的叶子节点。此宏接受两个额外参数：一个指针和一个长度。指针应指向字符串起始位置。如果长度为零，字符串被视为常量字符串，尝试写入会失败（即使节点允许写访问）。如果长度非零，它指定字符串缓冲区的最大长度（包括终止空字符），尝试写入超出缓冲区大小的字符串会失败。

宏 `SYSCTL_STRUCT` 用于定义值为单个 C struct 的叶子节点。此宏接受一个额外的指针参数，应指向要控制的结构。结构大小从类型推断。

宏 `SYSCTL_OPAQUE` 用于定义值为数据缓冲区（类型未指定）的叶子节点。此宏接受三个额外参数：指向数据缓冲区起始的指针、数据缓冲区的长度，以及描述数据缓冲区格式的字符串。

```c
static SYSCTL_NODE(, OID_AUTO, demo, 0, NULL,
    "Demonstration tree");

static char name_buffer[64] = "initial name";
SYSCTL_STRING(_demo, OID_AUTO, name,
    CTLFLAG_RW, name_buffer,
    sizeof(name_buffer), "Demo name");

static struct demo_stats {
    int demo_reads;
    int demo_writes;
} stats;
SYSCTL_STRUCT(_demo, OID_AUTO, status,
    CTLFLAG_RW, &stats, demo_stats,
    "Demo statistics");
SYSCTL_OPAQUE(_demo, OID_AUTO, mi_switch,
    CTLFLAG_RD, &mi_switch, 64, "Code",
    "First 64 bytes of mi_switch()");
```

**示例 3**

示例 3 定义了一个顶层节点，包含三个叶子节点，分别描述字符串缓冲区、结构和不透明数据缓冲区。

## 节点标志

每个节点定义都需要 flags 参数。所有叶子节点和具有非 NULL 函数处理器的分支节点必须在 flags 字段中指定允许的访问（读和/或写）。flags 字段还可包含表 1 中列出的零个或多个标志。

| FLAG | 用途 |
| ---- | ---- |
| `CTLFLAG_ANYBODY` | 所有用户都可写此节点。通常只有超级用户可写节点。 |
| `CTLFLAG_SECURE` | 仅在 securelevel 小于等于零时可写。 |
| `CTLFLAG_PRISON` | 由 **jail(2)** 创建的 prison 中的超级用户可写。 |
| `CTLFLAG_SKIP` | 在树的迭代遍历中隐藏此节点，例如 **sysctl(8)** 列出节点时。 |
| `CTLFLAG_MPSAFE` | 处理器例程不需要 Giant。所有简单节点类型已设置此标志。仅对使用自定义处理器的节点显式要求。 |
| `CTLFLAG_VNET` | 如果 prison 包含自己的虚拟网络栈，则其中的超级用户可写。 |

**表 1**

## 节点类型

节点值的类型在节点 flags 的一个字段中指定。标准节点宏都使用特定类型，并调整 flags 参数以包含相应类型。宏 `SYSCTL_PROC` 不暗示特定类型，因此必须显式指定类型。注意所有节点都允许返回或接受值数组，类型仅指定一个数组成员的类型。标准节点宏都返回或接受单个值而非数组。可用类型列于表 2。

注意，由于 `SYSCTL_PROC` 仅定义叶子节点，不应使用 `CTLTYPE_NODE`。带自定义处理器的分支节点在下文介绍。

| FLAG | 含义 |
| ---- | ---- |
| `CTLTYPE_NODE` | 此节点是分支节点，没有关联值。 |
| `CTLTYPE_INT` | 此节点描述一个或多个有符号整数。 |
| `CTLTYPE_UINT` | 此节点描述一个或多个无符号整数。 |
| `CTLTYPE_LONG` | 此节点描述一个或多个有符号长整数。 |
| `CTLTYPE_ULONG` | 此节点描述一个或多个无符号长整数。 |
| `CTLTYPE_S64` | 此节点描述一个或多个有符号 64 位整数。 |
| `CTLTYPE_U64` | 此节点描述一个或多个无符号 64 位整数。 |
| `CTLTYPE_STRING` | 此节点描述一个或多个字符串。 |
| `CTLTYPE_OPAQUE` | 此节点描述任意数据缓冲区。 |
| `CTLTYPE_STRUCT` | 此节点描述一个或多个数据结构。 |

**表 2**

## 节点格式字符串

每个节点除类型外还有一个格式字符串。**sysctl(8)** 工具使用此字符串格式化节点值。与节点类型一样，大多数标准宏隐式指定格式。宏 `SYSCTL_OPAQUE` 和 `SYSCTL_PROC` 要求显式指定格式。大多数格式字符串与特定类型绑定，大多数类型只有单一格式字符串。可用格式字符串列于表 3。

| FORMAT | 含义 |
| ------ | ---- |
| `A` | ASCII 字符串。与 `CTLTYPE_STRING` 一起使用。 |
| `I` | 有符号整数。与 `CTLTYPE_INT` 一起使用。 |
| `IU` | 无符号整数。与 `CTLTYPE_UINT` 一起使用。 |
| `IK` | 值单位为十分之一开尔文度的整数。**sysctl(8)** 工具在显示前将值转换为摄氏度。与 `CTLTYPE_UINT` 一起使用。 |
| `L` | 有符号长整数。与 `CTLTYPE_LONG` 一起使用。 |
| `LU` | 无符号长整数。与 `CTLTYPE_ULONG` 一起使用。 |
| `Q` | 有符号 64 位整数。与 `CTLTYPE_S64` 一起使用。 |
| `QU` | 无符号 64 位整数。与 `CTLTYPE_U64` 一起使用。 |
| `S,<foo>` | 类型为 `struct foo` 的 C 结构。与 `CTLTYPE_STRUCT` 一起使用。**sysctl(8)** 工具理解少数结构类型，如 `struct timeval` 和 `struct loadavg`。 |

**表 3**

## 处理器函数

系统控制节点处理器可用于提供读写现有变量以外的行为。处理器可用于提供输入验证（如对新节点值的范围检查）。处理器也可生成临时数据结构返回用户态，常用于返回系统状态快照（如开放网络连接列表或进程表）的处理器。

处理器函数接受四个参数并返回整数错误码。`<sys/sysctl.h>` 头文件提供一个宏来定义函数参数：`SYSCTL_HANDLER_ARGS`。它定义了四个参数：“oidp”、“arg1”、“arg2”和“req”。“oidp”参数指向描述当前调用处理器的节点的 `struct sysctl_oid` 结构。“arg1”和“arg2”参数保存定义此节点的 `SYSCTL_PROC` 调用中赋予“arg1”和“arg2”的值。“req”参数指向描述所发出特定请求的 `struct sysctl_req` 结构。返回值成功时为零，失败时为来自 `<sys/errno.h>` 的错误号。如果返回 `EAGAIN`，请求将在内核内重试，不返回用户态也不检查信号。

示例 4 定义了两个整数节点，使用自定义处理器拒绝设置无效值的尝试。它使用预定义的处理器函数 `sysctl_handle_int`（用于实现 `SYSCTL_INT`）来更新局部变量。如果请求尝试设置新值，它验证新值，仅在接受新值时更新关联变量。

```c
/*
 * 'arg1' 指向要导出的变量，'arg2' 指定最大值。
 * 假定不允许负值。
 */
static int
sysctl_handle_int_range(SYSCTL_HANDLER_ARGS)
{
    int error, value;

    value = *(int *)arg1;
    error = sysctl_handle_int(oidp, &value, 0, req);
    if (error != 0 || req->newptr == NULL)
        return (error);
    if (value < 0 || value >= arg2)
        return (EINVAL);
    *(int *)arg1 = value;
    return (0);
}

static int foo;
SYSCTL_PROC(_debug, OID_AUTO, foo, CTLFLAG_RW | CTLTYPE_INT, &foo, 100,
    sysctl_handle_int_range, "I", "Integer between 0 and 99");
static int bar;
SYSCTL_PROC(_debug, OID_AUTO, bar, CTLFLAG_RW | CTLTYPE_INT, &bar, 0x100,
    sysctl_handle_int_range, "I", "Integer between 0 and 255");
```

**示例 4**

此示例使用预定义处理器（`sysctl_handle_int`）发布旧值并接受新值。某些自定义处理器需要直接管理这些步骤。宏 `SYSCTL_IN` 和 `SYSCTL_OUT` 专为此提供。两个宏都接受三个参数：来自 `SYSCTL_HANDLER_ARGS` 的当前请求（“req”）指针、内核地址空间中的缓冲区指针，以及长度。宏 `SYSCTL_IN` 将数据从调用者的“new”缓冲区复制到内核缓冲区。宏 `SYSCTL_OUT` 将数据从内核缓冲区复制到调用者的“old”缓冲区。这些宏复制成功时返回零，失败时返回错误号。特别地，如果调用者缓冲区太小，宏会失败并返回 `ENOMEM`。

这些宏可多次调用。每次调用都推进调用者缓冲区中的内部偏移。多次调用 `SYSCTL_OUT` 会将内核缓冲区追加到调用者的“old”缓冲区，多次调用 `SYSCTL_IN` 会从调用者的“new”缓冲区读取连续数据块。

**sysctl(3)** 调用后返回用户态的值之一是“old”缓冲区中返回的数据量。即使复制因错误失败，计数也会按传递给 `SYSCTL_OUT` 的完整长度推进。这可用于允许用户态查询返回可变大小缓冲区的节点所需的“old”缓冲区长度。

如果生成复制到“out”缓冲区的数据开销大，且处理器能估计所需空间，则处理器可对这种情况特殊处理。调用者通过为“old”缓冲区使用 NULL 指针来查询长度。处理器可通过将 `req->oldptr` 与 NULL 比较来检测此情况。然后处理器可单次调用 `SYSCTL_OUT`，传递 NULL 作为内核缓冲区，总估计长度作为长度。如果数据大小频繁变化，处理器应高估缓冲区大小，使调用者在后续查询节点状态时不太可能得到 `ENOMEM` 错误。

宏 `SYSCTL_OUT` 和 `SYSCTL_IN` 可访问用户进程中的内存。如果用户页面当前未映射，这些访问可能触发页错误。因此，调用这些宏时不能持有不可睡眠的锁，如互斥锁和读写锁。某些控制节点返回与内核内对象列表对应的状态对象数组，其中列表由不可睡眠锁保护。此类处理器可使用的一种选择是分配足够大的临时内核缓冲区以容纳所有输出。处理器可在锁下遍历列表填充内核缓冲区，然后在释放锁后通过 `SYSCTL_OUT` 传递填充好的缓冲区。另一种选择是在遍历列表时每次调用 `SYSCTL_OUT` 前后释放锁。某些处理器可能不希望分配临时内核缓冲区（因为太大），也可能不希望释放锁（因为导致的竞争难以处理）。系统为这些处理器提供第三种选择：请求的“old”缓冲区可通过调用 `sysctl_wire_old_buffer` 固定。固定缓冲区保证对缓冲区的任何访问都不会产生页错误，从而允许在持有不可睡眠锁时使用 `SYSCTL_OUT`。注意此选项仅适用于“old”缓冲区。没有为“new”缓冲区提供相应函数。`sysctl_wire_old_buffer` 函数成功时返回零，失败时返回错误号。

如果 sysctl 节点希望在 64 位内核中由 32 位进程访问时正常工作，可通过检查 `req->flags` 中的标志 `SCTL_MASK32` 检测此情况。例如，返回长值的节点在此情况下应返回 32 位整数。返回与对象内部列表对应的结构数组的节点可能需要返回具有备用 32 位布局的结构数组。

如果节点允许调用者通过“new”值改变其状态，处理器应将 `req->newptr` 与 NULL 比较以确定是否提供“new”值。仅在 `req->newptr` 非 NULL 时，处理器才应调用 `SYSCTL_IN` 并尝试设置新值。

自定义节点处理器使用这些特性的一个例子是“kern.proc.proc”节点的实现。内核内实现更复杂，但示例 5 提供了简化版本。

```c
static int
sysctl_kern_proc_proc(SYSCTL_HANDLER_ARGS)
{
#ifdef COMPAT_FREEBSD32
    struct kinfo_proc32 kp32;
#endif
    struct kinfo_proc kp;
    struct proc *p;
    int error;

    if (req->oldptr == NULL) {
#ifdef COMPAT_FREEBSD32
        if (req->flags & SCTL_MASK32)
            return (SYSCTL_OUT(req, NULL, (nprocs + 5) *
                sizeof(struct kinfo_proc32)));
#endif
        return (SYSCTL_OUT(req, NULL, (nprocs + 5) *
            sizeof(struct kinfo_proc)));
    }
    error = sysctl_wire_old_buffer(req, 0);
    if (error != 0)
        return (error);
    sx_slock(&allproc_lock);
    LIST_FOREACH(p, &allproc, p_list) {
        PROC_LOCK(p);
        fill_kinfo_proc(p, &kp);
#ifdef COMPAT_FREEBSD32
        if (req->flags & SCTL_MASK32) {
            freebsd32_kinfo_proc_out(&kp, &kp32);
            error = SYSCTL_OUT(req, &kp32, sizeof(kp32));
        } else
#endif
        error = SYSCTL_OUT(req, &kp, sizeof(kp));
        PROC_UNLOCK(p);
        if (error != 0)
            break;
    }
    sx_sunlock(&allproc_lock);
    return (error);
}
SYSCTL_PROC(_kern_proc, KERN_PROC_PROC, proc, CTLFLAG_RD |
    CTLFLAG_MPSAFE | CTLTYPE_STRUCT, NULL, 0, sysctl_kern_proc_proc,
    "S,kinfo_proc", "Process table");
```

**示例 5**

## 复杂分支节点

通过 `SYSCTL_NODE` 声明的分支节点可指定自定义处理器。如果指定了处理器，则当任何地址以该分支节点地址开头的节点被访问时，都会调用此处理器。处理器函数的工作方式类似于上述自定义处理器。与 `SYSCTL_PROC` 不同，“arg1”和“arg2”参数不可配置。相反，“arg1”指向包含所访问节点地址的整数数组，“arg2”包含地址长度。注意“arg1”和“arg2”指定的地址相对于调用处理器的分支节点。例如，如果分支节点地址为 1.2，访问节点 1.2.3.4，分支节点的处理器将以“arg1”指向包含“3, 4”的数组、“arg2”为 2 调用。示例 6 给出了“kern.proc.pid”处理器的简化版本。回想这是示例 1 调用的节点。

```c
static int
sysctl_kern_proc_pid(SYSCTL_HANDLER_ARGS)
{
    struct kinfo_proc kp;
    struct proc *p;
    int *mib;

    if (arg2 == 0)
        return (EISDIR);
    if (arg2 != 1)
        return (ENOENT);
    mib = (int *)arg1;
    p = pfind(mib[0]);
    if (p == NULL)
        return (ESRCH);
    fill_kinfo_proc(p, &kp);
    PROC_UNLOCK(p);
    return (SYSCTL_OUT(req, &kp,
        sizeof(kp)));
}

static SYSCTL_NODE(_kern_proc, KERN_PROC_PID,
    pid, CTLFLAG_RD | CTLFLAG_MPSAFE,
    sysctl_kern_proc_pid,
    "Process information");
```

**示例 6**

## 动态控制节点

前面描述的控制节点是静态控制节点。静态节点在源文件中以固定名称定义，在内核初始化 sysctl 子系统或加载内核模块时创建。内核模块中的静态节点在卸载模块时移除。传递给静态节点处理器的参数也在链接时解析。这意味着静态节点通常基于全局变量操作。

内核还支持动态控制节点。与静态节点不同，动态节点可在任何时候创建或销毁。它们可使用动态生成的名称和引用动态分配的变量。动态节点既可作为静态节点的子节点，也可作为动态节点的子节点创建。

## sysctl 上下文

为安全移除动态控制节点，每个节点必须显式跟踪并以安全顺序移除（叶子先于分支）。手动完成此工作繁琐且易出错，因此内核提供 sysctl 上下文抽象。sysctl 上下文是跟踪零个或多个动态控制节点的容器。它允许其包含的所有控制节点在原子事务中安全移除。

典型做法是通过调用 `sysctl_ctx_init` 为每组相关节点创建一个上下文。所有节点在初始化期间（如驱动附加到设备时）添加到上下文。只需维护对上下文的引用。在拆卸时（如驱动从设备分离时），单次调用 `sysctl_ctx_free` 即足以移除整组控制节点。

## 添加动态控制节点

动态控制节点通过使用 `<sys/sysctl.h>` 中的某个宏 `SYSCTL_ADD_*` 创建。每个宏对应一个用于创建静态节点的宏，但有以下区别：

- 动态宏在代码块内调用，而非在顶层。动态宏返回指向所创建节点的指针。
- 动态宏添加一个额外参数，即新节点应关联的 sysctl 上下文指针。这是宏的第一个参数。
- 父参数指定为指向父节点子节点列表的指针。提供两个辅助宏来定位这些指针。父节点为静态控制节点时应使用宏 `SYSCTL_STATIC_CHILDREN`。它以父节点名为唯一参数。名称格式与声明静态节点时指定父节点的方式相同。对于父节点为动态节点的情况，应使用宏 `SYSCTL_CHILDREN`。它接受指向父节点的指针（由先前 `SYSCTL_ADD_NODE` 调用返回）作为唯一参数。
- 名称参数指定为指向 C 字符串的指针，而非未加引号的标识符。内核会复制此字符串用作节点名。这允许在需要时在临时缓冲区中构造名称。
- 内核也会复制描述参数，因此可在需要时在临时缓冲区中构造。

示例 7 定义了两个函数来管理动态 sysctl 节点。第一个函数初始化 sysctl 上下文并创建新节点。第二个函数销毁节点并销毁上下文。

```c
static struct sysctl_ctx_list ctx;
static int
load(void)
{
    static int value;
    int error;

    error = sysctl_ctx_init(&ctx);
    if (error)
        return (error);
    if (SYSCTL_ADD_INT(&ctx, SYSCTL_STATIC_CHILDREN(_debug), OID_AUTO,
        "dynamic", CTLFLAG_RW, &value, 0, "An integer") == NULL)
        return (ENXIO);
    return (0);
}

static int
unload(void)
{
    return (sysctl_ctx_free(&ctx));
}
```

**示例 7**

## 可调参数

另一个常与控制节点配合使用的内核 API 是可调参数 API。可调参数是存储在内核环境中的值。内核环境由引导加载器填充，也可在运行时由 **kenv(1)** 命令修改。内核环境由带字符串值的命名变量组成，类似于用户进程的环境。API 在 `<sys/kernel.h>` 中提供两组宏。

第一组（`TUNABLE_*`）在顶层声明，类似于静态控制节点，在引导时或加载模块时获取可调参数的值。第二组宏（`TUNABLE_*_FETCH`）可在代码块中用于在运行时获取可调参数。

每个宏接受一个 C 字符串名称作为第一个参数，指定要读取的可调参数名。将可调参数与控制节点一起使用时，惯例是使用控制节点名作为可调参数名。

可调参数 API 支持若干整数类型。每个宏接受一个指向相应类型整数变量的指针作为第二个参数。每次宏调用在内核环境中搜索请求的可调参数。如果找到可调参数且整个字符串值成功解析，整数变量将更改为解析后的值。注意溢出会被静默忽略。如果未找到可调参数或包含无效字符，整数变量保持不变。为整数提供的宏有：`TUNABLE_INT` 用于有符号整数，`TUNABLE_LONG` 用于有符号长整数，`TUNABLE_ULONG` 用于无符号长整数，`TUNABLE_QUAD` 用于有符号 64 位整数。

整数可调参数的字符串值按 **strtol(3)** 的方式解析，基数为零。具体而言，以“0x”开头的字符串解释为十六进制值，以“0”开头的字符串解释为八进制值，所有其他字符串解释为十进制值。此外，字符串可包含可选的单字符后缀，指定单位。值按单位大小缩放。单位不区分大小写。支持的单位见表 4。

| 后缀 | 值 |
| ---- | -- |
| k | 2^10 |
| m | 2^20 |
| g | 2^30 |
| t | 2^40 |

**表 4**

字符串可调参数也由宏 `TUNABLE_STR` 支持。此宏接受三个参数：可调参数名、指向字符缓冲区的指针和字符缓冲区长度。如果内核环境中不存在该可调参数，字符缓冲区保持不变。如果存在，其值复制到缓冲区。缓冲区中的字符串始终以空字符终止。如果值太长无法放入缓冲区，会被截断。

宏 `TUNABLE_*_FETCH` 接受与对应宏 `TUNABLE_*` 相同的参数。它们语义相同，但有一个额外行为：如果找到并成功解析可调参数，这些宏返回整数值零，否则返回非零值。

具有对应可调参数的系统控制节点应使用标志 `CTLFLAG_RDTUN` 或 `CTLFLAG_RWTUN` 指定节点允许的访问。注意这不会导致系统根据节点名隐式获取可调参数。可调参数必须显式获取。但它确实向 **sysctl(8)** 工具提供提示，用于诊断消息。

示例 8 演示了在设备驱动中使用可调参数获取默认参数。该参数作为只读控制节点可供用户查询（这有助于用户确定默认值）。它还包括附加例程的一部分，其中全局可调参数用于设置每设备控制变量的初始值。为每个设备创建动态 sysctl，允许独立更改每个设备的变量。sysctl 存储在 new-bus 子系统创建的每设备 sysctl 树中。它还使用每设备 sysctl 上下文，使 sysctl 在设备分离时自动销毁。

```c
static SYSCTL_NODE(_hw, OID_AUTO, foo, CTLFLAG_RD, NULL,
    "foo(4) parameters");
static int foo_widgets = 5;
TUNABLE_INT("hw.foo.widgets", &foo_widgets);
SYSCTL_INT(_hw_foo, OID_AUTO, widgets, CTLFLAG_RDTUN, &foo_widgets, 0,
    "Initial number of widgets for each foo(4) device");
static int
foo_attach(device_t dev)
{
    struct foo_softc *sc;
    char descr[64];

    sc = device_get_softc(dev);
    sc->widgets = foo_widgets;
    snprintf(descr, sizeof(descr), "Number of widgets for %s",
        device_get_nameunit(dev));
    SYSCTL_ADD_INT(device_get_sysctl_ctx(dev),
        SYSCTL_CHILDREN(device_get_sysctl_tree(dev)), OID_AUTO,
        "widgets", CTLFLAG_RW, &sc->widgets, 0, descr);
    ...
}
```

**示例 8**

可调参数的接口定义在 `<sys/kernel.h>`，实现可在 **sys/kern/kern_environment.c** 中找到。

系统控制节点的接口定义在 `<sys/sysctl.h>`，实现可在 **sys/kern/kern_sysctl.c** 中找到。检查预定义处理器的实现可能特别有用。首先，它们演示了 `SYSCTL_IN` 和 `SYSCTL_OUT` 的典型用法。其次，它们可用于在自定义处理器中编排数据。

---

**John Baldwin** 于 1999 年加入 FreeBSD 项目成为提交者。他在系统的多个领域工作过，包括 SMP 基础设施、网络栈、虚拟内存和设备驱动支持。John 曾任职于核心团队和发布工程团队，并每年春天组织 FreeBSD 开发者峰会。
