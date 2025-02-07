# 在 FreeBSD 的 DDB 内核调试器中编写自定义命令

- 原文链接：[Writing Custom Commands in FreeBSD’s DDB Kernel Debugger](https://freebsdfoundation.org/wp-content/uploads/2023/01/Baldwin_DDB.pdf)
- 作者：**JOHN BALDWIN**


DDB 是一款交互式内核调试器，可用于检查系统状态并控制正在运行的内核。DDB 最初作为 Mach 操作系统的一部分开发，后来被移植到 386BSD，并由包括 FreeBSD、NetBSD 以及 OpenBSD 在内的多个操作系统继承。本文重点介绍了 FreeBSD 中 DDB 的实现。

DDB 在系统控制台上运行，当调试器激活时，系统执行将被暂停，从而能在一致的状态下检查系统。虽然可以手动进入 DDB，但通常是在内核崩溃后使用。FreeBSD 的内核可以配置为在内核崩溃后自动进入 DDB，从而让用户和系统管理员在重启之前检查系统状态。在默认情况下，从 FreeBSD 主分支构建的调试内核即如此。  

DDB 提供了许多调试器常见的功能。它支持诸如单步执行和断点等运行控制，并在硬件支持的平臺上支持硬件监视点。DDB 包含多个命令，用于显示系统信息，包括堆栈跟踪和内存转储。

与许多其他调试器不同，DDB 无法理解类型信息，也无法美化结构体的输出或在表达式中计算结构体或联合体成员。不过，DDB 可以通过定义新命令来扩展，新命令甚至可以在内核模块中实现，这些模块可以在启动后加载。

### DDB 执行上下文

DDB 在一种与普通内核执行上下文有几处不同的特殊上下文中执行：

- 当 DDB 激活时，系统处于暂停状态，并借用了每个 CPU 上当前执行线程的执行上下文。在此上下文中，正常的内核调度器不会运行，并且不允许借用的每个 CPU 上的线程进行上下文切换。这意味着在此上下文中的代码执行不得进入休眠状态或因等待锁而阻塞。
- 如果在执行 DDB 命令期间发生故障或陷阱，当前线程将使用 longjmp() 返回到 DDB 的主循环继续执行。
- DDB 直接访问控制台设备以实现控制台的输入和输出。

由于这些独特的行为，DDB 命令的实现应遵循以下准则：

- **命令应避免副作用。**  
  如果在命令执行期间发生错误，则无法撤销任何已产生的副作用。因此，最安全的方法是在可能的情况下避免副作用。

- **命令不应使用锁。**  
  由于所有 CPU 的执行均被暂停，系统中大多数数据结构的状态不会发生变化，因此不需要通过锁与其他 CPU 进行同步。此外，获取锁本身也是一种副作用，如果在持锁状态下命令出错，这种副作用无法回退。在特殊情况下，如果命令希望以安全的方式修改系统状态，可以使用尝试锁（try locks）。目前，使用这种方式的命令之一是 `kill` 命令，该命令可以向进程发送信号。

- **命令应避免使用复杂的 API。**  
  高级 API 往往会修改系统状态或包含其他副作用（例如获取锁）。

- **大多数 DDB 命令仅检查系统状态而不进行修改，并输出系统某一部分状态的易读描述。**  
  其中许多命令是用于美化输出的打印程序，它们会打印关于特定数据结构或数据结构列表的信息。

- **命令必须使用 DDB 的 API 进行控制台输入和输出。**  
  这主要意味着在输出时应使用 `db_printf()` 而不是 `printf()`。DDB 提供了一个简单的控制台输出 API。`db_printf()` 函数类似于普通内核中的 `printf()`，并支持所有相同的格式说明符。该函数直接写入控制台设备，绕过系统日志设备。此外，`db_printf()` 还包含简单的分页支持。每当向控制台输出一个换行符时，`db_printf()` 会检查是否应暂停输出。如果需要暂停，`db_printf()` 会在控制台上输出提示，允许用户控制在下一次暂停前显示多少行内容。待用户对提示作出响应，`db_printf()` 就会返回。如果用户请求当前命令退出（停止生成输出），`db_printf()` 会将全局变量 `db_pager_quit` 设置为非零值。如果命令在循环中生成输出（例如，使用循环遍历一个链表数据结构），则该命令应在每次循环迭代中检查 `db_pager_quit`，如果其被设置，则提前退出循环。


### 命令函数

大多数 DDB 命令遵循 ddb(4) 中描述的简单语法：

```c
command[/modifier] [address[,count]]
```

当用户在 DDB 提示符下输入命令时，DDB 会解析该命令行。其中，地址（address）和计数（count）字段被视为表达式，可以包含对命名符号的引用以及许多 C 语言算术运算符；而命令（command）和修饰符（modifier）字段则被视为简单字符串。DDB 使用命令字段来查找一个 C 函数指针，并调用该 C 函数来执行命令。

实现 DDB 命令的函数使用如下签名：

```c
void fn(db_expr_t addr, bool have_addr, db_expr_t count, char *modif)
```


这里的 `addr` 参数包含命令要操作的地址。该地址可以是用户显式提供的地址，也可以是上一次命令使用的地址； `have_addr` 参数为 `true` 时表示地址是显式提供的；`count` 参数包含计数字段的值。如果计数字段未指定，则 `count` 的值设为 -1；`modif` 参数是一个指向包含修饰符字段的 C 字符串的指针。如果未指定修饰符，则 `modif` 将指向一个空字符串。

命令函数通过 DDB 维护的内部表与命令名称关联。DDB 提供了一些辅助宏，用于抽象出注册新命令的大部分细节。每个宏接受两个参数：第一个参数是命令名称，第二个参数是与该命令关联的 C 函数名称。除了在表中注册链接之外，这些宏还提供了 C 函数的声明，并应紧跟着函数体。每个宏都与特定的命令表相关联。  
- **DB_COMMAND** 宏用于定义一个新的顶级命令；  
- **DB_SHOW_COMMAND** 宏用于在 “show” 表中定义一个新命令；  
- **DB_SHOW_ALL_COMMAND** 宏用于在 “show all” 表中定义一个新命令。  

例如，`DB_SHOW_COMMAND(bar, db_show_bar_func)` 定义了一个新的 “show bar” 命令，同时也定义了一个新的 C 函数 `db_show_bar_func`，用于实现该命令。按照最佳实践（但不是强制要求），建议使用 `db__cmd` 这种命名模式来命名与命令关联的 C 函数。

清单 1 展示了一个名为 “double” 的简单命令的源码。该命令将用户提供的地址乘以 2，并输出结果。

清单 2 则展示了该命令的一些使用案例。第三个案例的输出可能会让人感到意外，因为 32 乘以 2 显然不等于 100。造成这种现象的原因是 DDB 解析整数值时默认使用 16 进制（这一默认进制由 DDB 内部的 `$radix` 变量控制）。在 16 进制中，32 的计算结果对应的十进制值为 50。

```c
DB_COMMAND(double, db_double_cmd)
{
if (have_addr)
 db_printf("%u\n", (u_int)addr * 2);
else
 db_printf("no address\n");
}
```

**清单 1：“double ”命令的源代码**

```c
db> double
no address
db> double 4
8
db> double 32
100
```

**清单 2： “double” 命令的示例输出**

**具有自定义语法的命令**  

DDB 命令不必仅使用上述简单语法。命令函数可以选择支持其他语法。命令在注册时通过传递额外的标志来请求这种功能。另一组宏接受命令标志作为第三个参数，分别为：`DB_COMMAND_FLAGS`、`DB_SHOW_COMMAND_FLAGS` 和 `DB_SHOW_ALL_COMMAND_FLAGS`。

有两个标志可用于控制命令行解析。

- **CS_MORE** 表示一个命令大体上遵循简单语法，但该命令支持多个地址。当指定此标志时，DDB 的主循环仍然会按正常方式解析命令行，但在调用命令函数之前不会丢弃词法分析器中剩余的所有标记。这允许命令函数在命令行中解析额外的选项。

- 第二个标志 **CS_OWN** 表示命令函数将自行完成所有解析工作。当指定此标志时，DDB 的主循环在读取命令名称后就停止对命令行进行解析，接下来的解析由命令函数通过 DDB 的词法分析器完成。

无论指定哪种标志，命令函数在返回之前都必须调用 **db_skip_to_eol()** 来丢弃当前命令行中剩余的标记。

DDB 提供了一些函数来解析命令行参数：

- **db_expression()** 用于解析算术表达式。它可以消耗多组输入，并支持完整的 DDB 表达式语法，包括符号解析以及各种 C 语言运算符。如果没有更多命令行参数可供解析，db_expression() 返回 0；如果表达式解析成功，则返回非零值，并将表达式的结果存储在其唯一参数所指向的变量中。如果 db_expression() 在解析表达式时遇到语法错误，它会打印一条信息，并通过 longjmp() 中止当前命令。命令函数在调用 db_expression() 时应避免任何副作用，因为如果用户提供无效输入，这些副作用将无法回退。

- 另外还有两个函数提供了对 DDB 词法分析器的低级接口：

  - **db_read_token()** 解析命令行中的下一个标记，并返回一个常量，该常量标识所解析的标记类型。这些常量命名为 tXXX，并定义在相关头文件中。大多数常量与 C 语言运算符和其他特殊标记相关，但有几个常量对自定义命令很有用：
    - **tEOL** 在遇到命令行末尾时返回。
    - **tEOF** 用于表示无效输入，例如包含无效字符的数字。
    - **tIDENT** 在解析到单词（标识符）时返回。该单词的副本保存在全局变量 **db_tok_string** 中。
    - **tNUMBER** 在解析到数字时返回。该数值以整数形式保存在全局变量 **db_tok_number** 中。需要注意的是，DDB 的词法分析器假定任何以十进制数字开头的单词都是数字，而任何以字母、下划线或反斜杠开头的单词都是标识符。

  - **db_unread_token()** 将单个标记插入到下一次调用 db_read_token() 时返回。传递给 db_unread_token() 的值应为上述 t 常量之一。通常，该函数用于在 db_read_token() 返回了无效或意外标记时，将刚读取的标记放回去。

DDB 还提供了两个额外的函数来处理解析错误：

- **db_error()** 会打印出调用者提供的消息，刷新词法分析器状态，并调用 longjmp() 来中止当前命令并返回到 DDB 的主循环。
- **db_flush_lex()** 则仅刷新词法分析器状态，丢弃当前命令行。如果需要更详细的错误消息或在 longjmp() 不合适的情况下撤销额外状态，可以使用 db_flush_lex()。

清单 3 展示了一个名为 “sum” 的命令的源代码。该命令计算命令行上给出的所有表达式的和。它使用了标志 CS_MORE，并在循环中使用 db_expression() 来解析命令行中的其他表达式。

清单 4 显示了该命令的一些示例输出。请注意，在第三个示例中，db_expression() 解析了表达式 “9 * 3”，并将值 27 返回给了 db_sum_cmd() 中的循环。

```c
DB_COMMAND_FLAGS(sum, db_sum_cmd, CS_MORE)
{
long total;
db_expr_t value;
if (!have_addr)
 db_error("no values to sum\n");
total = addr;
while (db_expression(&value))
 total += value;
db_skip_to_eol();
db_printf("Total is %lu\n", total);
}
```

**清单 3：“sum ”命令的源代码**

```c
db> sum 1
Total is 1
db> sum 1 2 3
Total is 6
db> sum 9 * 3 4
Total is 31
```

**清单 4： “sum” 命令的示例输出**

清单 5 包含了 “show softc” 命令的源代码。该命令接受设备名称作为单个命令行参数。如果找到该设备，命令将打印出指向该设备 softc 结构的指针值。该结构包含设备驱动程序维护的每个设备的信息。此命令使用 CS_OWN 标志以请求对命令行解析的完全控制。它使用 db_read_token() 从命令行中获取设备名称。如果给出了有效的设备名称，将返回一个 tIDENT 标记，并在 db_tok_string 中保存该设备名称。

清单 6 显示了此命令的一些示例输出。

```c
DB_SHOW_COMMAND_FLAGS(softc, db_show_softc_cmd, CS_OWN)
{
device_t dev;
int token;
token = db_read_token();
if (token != tIDENT)
 db_error("Missing or invalid device name");
dev = device_lookup_by_name(db_tok_string);
db_skip_to_eol();
if (dev == NULL)
 db_error("device not found\n");
db_printf("%p\n", device_get_softc(dev));
}
```

**清单 5：“show softc ”命令的源代码**

```c
db> show softc 4
Missing or invalid device name
db> show softc foo0
device not found
db> show softc pci0
0xfffff800039380f0
```

**清单 6： “show softc” 命令的示例输出**  

### 自定义命令表  

DDB 命令表包含一组命令。可以通过在现有表中定义一个特殊命令来创建额外的命令表，从而构建命令表的层级结构。  

定义新表的命令不会指定用于处理该命令的函数。相反，表必须定义并初始化一个 `struct db_command_table` 类型的变量，该变量包含一个链接列表，用于存储属于该表的命令。然后，父表中的命令条目会关联到这个新表的指针。按照惯例，该变量的命名格式应为 `db__table`。  

截至目前，还没有类似 `DB_COMMAND` 这样高度抽象的宏可用于定义新表。因此，定义新表必须使用 “内部” 宏 `_DB_SET`。属于该表的命令必须使用 “内部” 宏 `_DB_FUNC` 定义，或者通过定义一个类似 `DB_SHOW_COMMAND` 的辅助宏来封装 `_DB_FUNC`。  

**清单 7** 包含了一个名为 “demo” 的命令表的源代码，以及属于该表的两个命令。  

代码首先定义了 `db_demo_table` 变量，以存储属于该新表的 DDB 命令列表。 `_DB_SET` 的调用将 “demo” 命令添加到顶级命令表（类似于 `DB_COMMAND`）。需要注意的是，`_DB_SET` 的第三个参数（通常用于存放函数处理程序的指针）被设为 `NULL`，而最后一个参数指向新定义的表。  

代码的其余部分定义了属于该新表的两个简单命令。 `_DB_FUNC` 的第二个和第三个参数类似于 `DB_COMMAND` 的两个参数。第四个参数用于指定新命令所属的父表。第五个参数用于标识诸如 `CS_MORE` 或 `CS_OWN` 之类的标志，最后一个参数应设为 `NULL`。  

`_DB_SET` 和 `_DB_FUNC` 的第一个参数应为父表的名称，前面加上下划线，并将空格替换为下划线。如果父表是主命令表，则应使用 `"_cmd"`。  

**清单 8 显示了这些命令的一些示例输出。** 

```c
/* 保存“demo *”命令的列表。*/
static struct db_command_table db_demo_table = LIST_HEAD_INITIALIZER(db_demo_table);
6 of 8
FreeBSD Journal • November/December 2022 11
/* 定义了一个“demo”顶级命令。 */
_DB_SET(_cmd, demo, NULL, db_cmd_table, 0, &db_demo_table);
_DB_FUNC(_demo, one, db_demo_one_cmd, db_demo_table, 0, NULL)
{
db_printf("one\n");
}
_DB_FUNC(_demo, two, db_demo_two_cmd, db_demo_table, 0, NULL)
{
db_printf("two\n");
}
```

**清单 7：“deom”命令的源代码** 

```c
db> demo
Subcommand required; available subcommands:
one two
db> demo one
one
db> demo two
two
```

### **支持分页的命令**  

我们的最后一个示例命令展示了如何遵循 DDB 的输出分页机制。  

大多数分页操作（例如继续输出一页或单行）都由 `db_printf()` 内部的分页实现处理。然而，如果用户请求停止分页，`db_pager_quit` 这个全局变量会被设置为非零值（如前文所述）。因此，任何在循环中生成输出的命令都应检查该变量，并在其被设置时终止循环。  

**清单 9** 提供了一个精简的示例命令，该命令会检查 `db_pager_quit` 的状态。该命令实现了 Internet 上的 “chargen”（字符生成）服务，它会在屏幕上持续生成输出行，直到用户通过分页机制请求退出，从而终止循环。  

这个示例的关键之处在于主循环的最后两行：如果 `db_pager_quit` 被设置，循环将立即退出。

```c
DB_COMMAND(chargen, db_chargen_cmd)
{
char *rs;
int len;
for (rs = ring;;) {
 …
 db_printf("\n");
 if (db_pager_quit)

Break;
  }
}
```

**清单 8：“chargen”命令的简略源代码**

### 结论  

DDB 提供了一个相对简单的框架，用于添加新命令。新命令甚至可以在系统引导后通过加载包含新命令的内核模块来添加。  

FreeBSD 的源码树中包含许多自定义命令的示例，可作为开发新命令的参考。可以通过搜索 `DB.*_COMMAND` 或 `db_printf` 来查找这些示例。此外，一个包含本文所有命令的内核模块可以在以下地址找到：  
[https://github.com/bsdjhb/ddb_commands_demo](https://github.com/bsdjhb/ddb_commands_demo)  

---

**John Baldwin** 是一名系统软件开发人员。在过去二十多年里，他在 FreeBSD 操作系统的多个内核组件（包括 x86 平台支持、SMP、各类设备驱动程序以及虚拟内存子系统）和用户空间程序中直接提交了代码修改。除了编写代码，他还曾担任 FreeBSD 核心团队和发布工程团队的成员，并为 GDB 调试器做出了贡献。John 与他的妻子 Kimberly 及三个孩子 Janelle、Evan 和 Bella 现居住在加利福尼亚州康科德市。
