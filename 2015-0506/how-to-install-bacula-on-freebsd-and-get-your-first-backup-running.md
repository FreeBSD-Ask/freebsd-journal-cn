# 如何在 FreeBSD 上安装 Bacula 并完成首次备份

- 原文：[How to Install Bacula on FreeBSD and Get Your First Backup Running](https://freebsdfoundation.org/our-work/journal/browser-based-edition/measure-twice-code-once/)
- 作者：**Dan Langille**

## 关于本文

本文中你将学到：

- 如何安装 Bacula
- 如何完成首次备份

## 我将使用

- Bacula 7.0.5
- PostgreSQL 9.3.5
- FreeBSD 9.3
- ZFS

你不必使用完全相同的选择也能从本文中有所收获。你将学到 Bacula 的概念，助你跨越常见的误解。我保证你会有所收获。

除非另有说明，Bacula 的所有配置文件都位于 **/usr/local/etc/bacula/**。

Bacula 指令忽略空格。因此 `Working Directory` 与 `WorkingDirectory` 是相同的。用你喜欢的写法即可。

有些值在“引号”中，有些则没有。用你喜欢的写法即可，但切记如果值包含空格，“就需要把它放在引号里。”

Bacula 组件或概念的词将以加粗、首字母大写的形式呈现。这有助于你区分 **File** 指令与磁盘上的 file，或 **Volume** 与卷。

程序或命令会以 `bconsole` 这样的形式出现。文件会以 **/etc/rc.conf** 这样的形式出现。

## 本文涵盖内容

- 客户端（Clients）
- 控制器（Director）
- 存储（Storage）
- 备份（Backups）
- 恢复（Restores）

坦白说：我有偏好。我喜欢 PostgreSQL、FreeBSD、ZFS 与 Bacula 的组合。事实上，我太喜欢它了，以至于为 Bacula 编写了 PostgreSQL 后端。从那以后，我使用 Bacula 已超过 10 年。我备份到 HDD、DDS、DLT 与 SDLT。Bacula 已证明可靠，它就是能用。客户端覆盖了广泛的操作系统，Bacula 也不在乎你用的是什么文件系统。

我也是 FreeBSD 的 Port Bacula 维护者。

我最初为 O’Reilly 撰写过关于 Bacula 的文章。后来我把那篇文章转载到了我的日记里。我为本期 FreeBSD 期刊更新了那篇文章。

关于 ZFS，本文其实没有太多有用信息，但写完本文后我发现要涵盖的内容太多，省略 ZFS 的细节是合理的。

### Bacula 简介

这里的描述简短而概括。我没有提到的东西，并不意味着 Bacula 做不到。

Bacula 是一款基于网络客户端-服务器模型的备份解决方案。它可备份到磁盘或磁带。社区活跃且蓬勃发展。如果你需要，也有出色的商业支持。备份的控制围绕一个 **Director** 集中进行。扩展时，你可以拥有任意数量的 Director。每个 Director 有 **Client**，这些 Client 可与其他 Director 共享。

Director 联系 Client，告诉它：把这个 **FileSet** 备份到这个 **Storage**。然后 Client 照办，备份文件。之后，那个备份作业可能被复制或迁移到另一个 Storage，可以是本地的也可以是远程的。例如，你可能备份到磁盘，然后希望把该作业复制到磁带。或复制到远程位置。

无论你对备份做什么，**Catalog** 都会跟踪备份了什么、何时备份、来自哪个 Client、现在位于何处。Catalog 也用于保留目的、清理与清除。

### 主要组件

Bacula 不是一个程序，而是多个。这些联网的程序通过 TCP/IP 互相通信。Bacula 的主要组件有：

1. **Director**——守护进程（`bacula-dir`），控制作业。
2. `bconsole`——命令行界面控制台。
3. **Storage**——守护进程（`bacula-sd`），从 Client 接收备份并存储到存储设备（例如磁带、磁盘）。
4. **Client**——守护进程（`bacula-fd`），读取文件并发送给 Storage。
5. **Catalog**——一个数据库，跟踪已运行的作业及其存储位置。我推荐用 PostgreSQL。也可用 MySQL，但 PostgreSQL 性能更好。我不推荐为此使用 SQLite。

本节余下内容可以跳过。它对理解如何首次运行 Bacula 并非必需。所有这些组件都需要能互相通信。请牢记这一点，尤其要考虑你的防火墙。特别是当你阅读下面内容并看到通信顺序时。

- `bacula-dir` 联系 `bacula-fd`，告诉它备份什么
- `bacula-fd` 联系 `bacula-sd`，把备份发送给它
- `bacula-sd` 联系 `bacula-dir`，汇报已完成备份的状态
- `bacula-dir` 联系数据库服务器，汇报状态
- `bconsole` 联系 `bacula-dir`
- 你可以通过 `bconsole` 查询 `bacula-dir`、`bacula-sd` 与 `bacula-fd` 的状态。执行此操作时，`bacula-dir` 联系 `bacula-sd`，再联系 `bacula-fd`，询问它们的状态并转达给你
- 如果把作业从一个 storage 复制到另一个，`bacula-sd` 需要与另一个 `bacula-sd` 通信
- 如果你使用 SD Calls Client 指令，`bacula-sd` 需要能联系 `bacula-fd`

### 我的备份

当我刚开始用 Bacula 时，我直接备份到磁带。如今 HDD 更大更便宜，我备份到磁盘。有些人认为这就够了。我不这么想。一次电源故障就可能让所有 HDD 报废。ZFS 是最好的文件系统，但它无法抵御灾难性悲剧。我相信备份要多份副本。至少一份在这里，一份在那里。这就是为什么我备份到磁带，然后把磁带移到别处。

我也有计划把备份发送到远程的 `bacula-sd`。我还没实施，但我会的。很快。

### 安装

安装 Bacula 相当简单。如果你想用 PostgreSQL（我推荐你这么做），可以从预构建的 package 安装。否则，从 Ports 树安装。

以下是我安装的内容：

```sh
# pkg install bacula-server
# pkg install postgresql93-server
```

我使用 PostgreSQL 9.3，因为它是 FreeBSD Ports/packages 系统当前的默认版本。其他版本也可以。

### 数据库设置

这是配置 PostgreSQL 配合 Bacula 使用的简要介绍。官方文档更为详尽。

在这些步骤中，我们先创建一个新的 PostgreSQL 数据库集群，然后启动服务器：

```sh
# service postgresql initdb
```

输出如下：

```sh
The files belonging to this database system will be owned by user "pgsql".
This user must also own the server process.
The database cluster will be initialized with locale "C".
The default text search configuration will be set to "english".
Data page checksums are disabled.
creating directory /usr/local/pgsql/data. . . ok
creating subdirectories. . . ok
selecting default max_connections. . . 100
selecting default shared_buffers. . . 128MB
creating configuration files. . . ok
creating template1 database in /usr/local/pgsql/data/base/1. . . ok
initializing pg_authid. . . ok
initializing dependencies. . . ok
creating system views. . . ok
loading system objects' descriptions. . . ok
creating collations. . . ok
creating conversions. . . ok
creating dictionaries. . . ok
setting privileges on built-in objects. . . ok
creating information schema. . . ok
loading PL/pgSQL server-side language. . . ok
vacuuming database template1. . . ok
copying template1 to template0. . . ok
copying template1 to postgres. . . ok
syncing data to disk. . . ok
WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option-A, or
-auth-local and-auth-host, the next time you run initdb.
Success. You can now start the database server using:
/usr/local/bin/postgres-D /usr/local/pgsql/data
or
/usr/local/bin/pg_ctl-D /usr/local/pgsql/data-l logfile start
```

```sh
# service postgresql start
LOG: could not create IPv6 socket: Protocol not supported
LOG: ending log output to stderr
HINT: Future log output will go to log destination "syslog".
```

接下来，我们将创建 Bacula 要用的数据库，即 Catalog。本例中部分输出已裁剪，因为重复且读起来乏味。

```sh
# su pgsql
# cd /usr/local/share/bacula/
$ createuser -d bacula
$ ./create_bacula_database
Creating postgresql database
CREATE DATABASE
ALTER DATABASE
Creation of bacula database succeeded.
Database encoding OK
$ psql -l
List of databases
Name | Owner |Encoding |Collate | Ctype | Access privileges
------+-----+-------+-----+----+----------
bacula | pgsql |SQL_ASCII |C | C |
postgres | pgsql |UTF8 |C | C |
template0 | pgsql |UTF8 |C | C | =c/pgsql+pgsql=CTc/pgsql
template1 | pgsql |UTF8 |C | C | =c/pgsql+pgsql=CTc/pgsql
(4 rows)
$ ./make_bacula_tables
Making postgresql tables
CREATE TABLE
ALTER TABLE
CREATE INDEX
CREATE TABLE
ALTER TABLE
[some output trimmed]
INSERT 0 1
INSERT 0 1
INSERT 0 1
[some output trimmed]
INSERT 0 1
Creation of Bacula PostgreSQL tables succeeded.
$ ./grant_bacula_privileges
Granting postgresql privileges
psql::2: ERROR: role "bacula" already exists
GRANT
GRANT
GRANT
[some output trimmed]
GRANT
GRANT
GRANT
Privileges for user bacula granted on database bacula.
$ psql -l
List of databases
Name | Owner | Encoding | Collate | Ctype |Access privileges
------+----+------+-----+----+----------
bacula | pgsql | SQL_ASCII | C | C |
postgres | pgsql | UTF8 | C | C |
template0 | pgsql | UTF8 | C | C |=c/pgsql +pgsql=CTc/pgsql
template1 | pgsql | UTF8 | C | C |=c/pgsql +pgsql=CTc/pgsql
(4 rows)
```

这就是你现在应当看到的。

### 配置

所有这些配置都将在单一 IP 地址上完成。对入门而言，我推荐这种方式。这能让你先把东西跑起来，再迁移到其他服务器。一步一个脚印。

我使用的 IP 地址是 **10.55.0.76**，如果所有东西都安装在同一系统上，这是你在配置中唯一需要改的值。

Bacula 有四个主要配置文件。默认位置是 **/usr/local/etc/bacula/**：

1. `bacula-dir.conf`
2. `bacula-fd.conf`
3. `bacula-sd.conf`
4. `bconsole.conf`

这些文件名不言自明。接下来几节中，我会给你一套能让你上手的最小配置。我保证。

下面的示例中，我们会慢慢开始，然后加快速度。你看到的第一个 `bacula-dir.conf` 文件会随着进展不断添加内容。我会在文末提供所有配置文件的链接。

#### `bacula-dir.conf`

`bacula-dir.conf` 是你为 Bacula 创建的最大的配置文件。所幸，你可以把它拆分成逻辑区域，放在任意多个（或少至任意几个）文件中。`bacula-dir.conf` 指定的内容中包括：

- 这个特定 `bacula-dir` 的身份
- 标识它要备份的 Client
- 列出将在该 Client 上运行的 Job
- 指定记录这些备份的 Catalog
- 列出该 Job 的 FileSet（文件列表）
- 指定该 Job 的 Schedule
- 标识要使用的 Storage

还有其他指令（例如 Pool），但我们把那个讨论推迟到后面。我喜欢像这样整理我的配置文件。我觉得这样更易读。这并非必需。

```sh
Director {
  Name = MyBaculaDirector
  Password = "the bacula-dir password"

  QueryFile = "/usr/local/share/bacula/query.sql"

  PidDirectory = "/var/run"
  WorkingDirectory = "/usr/local/bacula/working"

  Messages = MyMessages
}

Catalog {
  Name = MyCatalog
  dbname = bacula
  dbaddress = 10.55.0.76
  user = bacula
  password = ""
}

Messages {
  Name = MyMessages
  mailcommand = "/usr/local/sbin/bsmtp-h localhost-f
  \"\(Bacula\)%r\"-s\"Bacula: %t %e of %c %l\" %r"
  operatorcommand = "/usr/local/sbin/bsmtp-h localhost-f \"\(Bacula\)%r\"
  -s \"Bacula: Intervention needed for %j\" %r"

  operator = root@localhost = mount
  mail = root@localhost = all
  console = all
  append = "/var/log/bacula.log" = all, !skipped, !restored
  catalog = all, !skipped, !saved
}
```

你可以直接使用这份配置。我建议首次设置时保留那个密码。等一切工作正常后再改。

那个 **Name** 与 **Password** 是此 Director 通过 `bconsole` 登录时接受的凭据。这些值会出现在两处：

1. 在 `bacula-dir.conf` 中，让 `bacula-dir` 知道接受什么凭据
2. 在 `bconsole.conf` 中，让 `bconsole` 知道发送什么凭据

这种“出现在两处”是应当尽早理解的重要概念。Bacula 在多处采用这种设计。我会在后续示例中指出。

随着 `bacula-dir.conf` 文件增长，你可以把配置拆分到单独的文件中。每个文件都可以用如下命令包含进来：

```sh
@/usr/local/etc/bacula/client-myclient.conf
```

稍后我会用到这个特性。

Catalog 数据库由第 13-19 行标识。它不指定数据库服务器类型（例如 PostgreSQL）；该选项在编译时设置。

第 21-31 行指定消息如何送达你。这份配置假设你的系统上有 SMTP 服务器，但你不需要它也能跟着本示例走。如果你开始依赖备份，则推荐配置。

#### `bacula-fd.conf`

下面是一个最小的 `bacula-fd.conf`：

```sh
Director {
  Name = MyBaculaDirector
  Password = "the bacula-fd password"
}

FileDaemon {
  Name = MyFirstBaculaClient
  WorkingDirectory = /var/db/bacula
  Pid Directory = /var/run
}

Messages {
  Name = Standard
  director = MyBaculaDirector = all, !skipped, !restored
}
```

与 `bacula-dir.conf` 一样，**Name** 与 **Password** 代表凭据。但这里 **Name** 与 **Password** 标识哪个 Director 可以联系此 Director。两者合起来是与这个 `bacula-fd` 通信所需的凭据。它会存在两处：

1. 此 `bacula-fd.conf` 中，让此 `bacula-fd` 知道接受什么凭据
2. 在 `bacula-dir.conf` 中，让 `bacula-dir` 知道联系此 `bacula-fd` 时出示什么凭据。

我稍后会展示第 2 项。

高级设置可以有多个 Director，此时一个 `bacula-fd.conf` 可以有多个 Director 指令，但本简单示例不涉及。

#### `bacula-sd.conf`

这是我们的最小 `bacula-sd.conf`：

```sh
Storage {
  Name = MyBaculaStorage
  WorkingDirectory = "/usr/local/bacula/working"
  Pid Directory = "/var/run"
}

Director {
  Name = MyBaculaDirector
  Password = "the bacula-sd password"
}

Device {
  Name = MyFirstStorageDevice
  Media Type = MyMediaType
  Archive Device = /usr/local/bacula/volumes
  LabelMedia = yes
  Random Access = yes
  AutomaticMount = yes
  RemovableMedia = no
  AlwaysOpen = no
}

Messages {
  Name = Standard
  director = MyBaculaDirector = all
}
```

我想你已看出规律。那个 **Name** 与 **Password** 标识哪个 Director 可以联系我们。那组凭据会存储在：

1. `bacula-sd.conf` 中，让 `bacula-sd` 知道必须出示什么凭据
2. `bacula-dir.conf` 中，让 `bacula-dir` 知道向此 `bacula-sd` 出示什么凭据

与 `bacula-fd.conf` 一样，我们稍后回到第 2 项。

#### `bconsole.conf`

这个最小的 `bconsole.conf` 能让你与 `bacula-dir` 交互：

```sh
Director {
  Name = MyBaculaDirector
  Password = "the bacula-dir password"

  Address = 10.55.0.76
}
```

这个 **Name** 与 **Password** 与 `bacula-dir.conf` 中出现的相同。它们就是前面提到的与 `bacula-dir` 通信所需的凭据。`bconsole` 启动时会读取此文件。

### 这只是开始

前几节你看到了最基础的内容。接下来几节你会看到它们如何串联起来。我会向你展示 Director 如何知道 Client、Job 与 FileSet。到目前为止，我们只是在定义基础。现在我们开始构建备份系统。

### Client

本节中，我们将向 Bacula 配置添加一个 Client。这可以直接加到 `bacula-dir.conf` 中，但我更愿意把所有与 Client 具体相关的内容放在单独的文件里。

我把以下内容放入 `client-myclient.conf`：

```sh
Client {
  Name = MyFirstBaculaClient
  Password = "the bacula-fd password"
  Address = 10.55.0.76
  Catalog = MyCatalog
}
```

为了把这份配置拉入 `bacula-dir.conf`，我向 `bacula-dir.conf` 添加了以下内容：

```sh
@/usr/local/etc/bacula/client-myclient.conf
```

这份配置会告诉 `bacula-dir` 包含该文件，文件里是我们 `bacula-fd` 的联系信息。

还记得凭据如何存两处吗？一处给发送方，一处给接收方？下面是一个示例：

```sh
# grep "the bacula-fd password" *
bacula-fd.conf: Password = "the bacula-fd password"
client-myclient.conf: Password = "the bacula-fd password"
```

第一处告诉 `bacula-fd` 接受什么密码。第二处告诉 `bacula-dir` 联系那个 `bacula-fd` 时使用什么凭据。

请注意：每当 Director 联系 Client 时，它必须使用的凭据是 Director 的 **Name**（不是 Client 的 Name）和 `bacula-dir.conf` 中 Client 资源里指定的 **Password**。需要说明的是，在我们的示例中，Director 必须传给 Client 的正确值是：

1. `MyBaculaDirector`
2. `the bacula-fd password`

因此，Client 资源中的 Name 用来命名该 client，并不参与凭据。这非常重要。

### FileSet

我们已告诉 Bacula 关于 Client 的信息，接下来要指定我们要备份哪些文件：一个 FileSet。

我把以下内容放入 `filesets.conf`。

```sh
FileSet {
  Name = "MyFirstFileSet"
  Include {
    Options {
      signature = MD5
    }

    File = /usr/local/etc/bacula/
  }

  Exclude {
    File = *~
  }
}
```

在这个 FileSet 中，我们备份一个目录。不要被那个名字 `File` 搞混。我选择备份 **/usr/local/etc/bacula/** 是因为我知道你会有这个目录。

每个文件都会记录其 MD5。这是可选的，但强烈推荐。运行 Verify Job 时会用它来比较磁盘上的内容与备份内容。

我们将排除任何以 `~`（波浪号）结尾的文件。

注意：

- 每个 Job 一个 FileSet，
- 一个 FileSet 可被多个 Job 使用。

为告诉 `bacula-dir` 这个 FileSet，我在 `bacula-dir.conf` 中添加了这一行：

```sh
@/usr/local/etc/bacula/filesets.conf
```

下一步：创建一个备份 Job。

### Job

Job 标识要备份什么、它位于哪个 Client 上、将存储此备份的 `bacula-sd`。这是我加到 `client-myclient.conf` 中的内容：

```sh
Job {
  Name = "MyFirstJob"
  Client = MyFirstBaculaClient
  Type = "Backup"
  FileSet = "MyFirstFileSet"
  Storage = MyFirstStorage
  Schedule = MyFirstSchedule
  Messages = MyMessages

  Pool = FullFile # 所有 Job 的必需参数，尽管接下来几行看起来并非如此

  Full Backup Pool = FullFile
  Differential Backup Pool = DiffFile
  Incremental Backup Pool = IncrFile
}
```

现在我们几乎告诉了 Bacula 一切：备份什么、从哪里来、存到哪里。现在我们得告诉 Bacula 何时运行这个 Job。

### Schedule

Schedule 非常灵活。我们从简单经典开始。下面告诉我们 Job 何时运行、以何种 Level（Incremental、Differential、Full）运行。我把它放入 `schedules.conf`。

```sh
Schedule {
  Name = MyFirstSchedule
  Run = Level=Full 1st sun at 8:15
  Run = Level=Differential 2nd-5th sun at 8:15
  Run = Level=Incremental mon-sat at 8:15
}
```

我把这一行加到 `bacula-dir.conf` 来包含上述内容：

```sh
@/usr/local/etc/bacula/schedules.conf
```

我的服务器使用 UTC 时间，所以这些会在大约 EST 凌晨 3:15 运行（我住在费城郊外）。

1. 每月第一个星期日运行 Full 备份。
2. 其余所有星期日运行 Differential 备份。
3. 每天运行 Incremental 备份。

Level 的最佳描述见 Job Resource 文档。

### Storage

在那个 Job 中，我们告诉 Bacula 把备份发到哪里：`MyFirstStorage`。我们已经在 `bacula-sd.conf` 中定义了这个 Storage，但还没有在 `bacula-dir.conf` 中告知它。这听起来奇怪。为什么要指定两次？Bacula 基于组件构建。本示例中所有东西都在同一台计算机上运行。然而，它们可以轻松地在不同计算机上运行。因此，`bacula-dir.conf` 必须标识它能使用的 `bacula-sd`。`bacula-sd.conf` 可能在另一台计算机上。

下面是我如何告诉 `bacula-dir` 它应当使用哪个 `bacula-sd`。我把这加到 `bacula-dir.conf`：

```sh
Storage {
  Name = MyFirstStorage
  Address = 10.55.0.76
  Password = "the bacula-sd password"
  Device = MyFirstStorageDevice
  Media Type = MyMediaType
}
```

又是 **Name** 与 **Password** 的概念。但这次略有不同。这里我们指定 `bacula-dir` 联系该 Address 上的 `bacula-sd` 时应使用的凭据。

Device 与 Media Type 的值必须与 `bacula-sd.conf` 中指定的匹配。

我想重复一下把 Client 资源加到 `bacula-dir.conf` 时提到的一点。这个 Storage 资源的 **Name** 不构成任何凭据的一部分。当我们的 Director 联系 **10.55.0.76** 上的 `bacula-sd` 时，它会使用这些凭据：

1. `MyBaculaDirector`
2. `the bacula-sd password`

### Pool 与 Volume

到目前为止，我对 Pool 一笔带过。用磁带来描述 Pool 更容易引入概念。设想你有一盒用于 Full 备份的磁带。那盒里每盘磁带都保留 12 个月。你还有一盒用于 Incremental 备份的磁带。每盘保留 15 天。另一盒有 Differential 备份，每盘保留 3 个月。

每个盒子就是一个 Pool。每盘磁带就是一个 Volume。

不要把 Bacula Volume 与文件系统或你对卷的其他概念混淆。Bacula Volume 包含一个备份（或备份的一部分），也可能是空的。

Pool 定义用于描述 Volume 的初始和/或共同特性。因此，Pool 可视为共享共同特性（如 Retention）的 Volume 集合。

这些是我加到 `pools.conf` 的 Pool 定义：

```sh
Pool {
  Name = FullFile
  Pool Type = Backup
  Recycle = yes
  AutoPrune = yes
  Volume Retention = 3years
  Storage = MyFirstStorage

  Maximum Volume Bytes = 5G
  Maximum Volumes = 5

  LabelFormat = "FullAuto-"
}

Pool {
  Name = DiffFile
  Pool Type = Backup
  Recycle = yes
  AutoPrune = yes
  Volume Retention = 6 weeks
  Storage = MyFirstStorage

  Maximum Volume Bytes = 5G
  Maximum Volumes = 5

  LabelFormat = "DiffAuto-"
}

Pool {
  Name = IncrFile
  Pool Type = Backup
  Recycle = yes
  AutoPrune = yes
  Volume Retention = 3 weeks
  Storage = MyFirstStorage

  Maximum Volume Bytes = 5G
  Maximum Volumes = 5

  LabelFormat = "IncrAuto-"
}
```

这指定了三个 Pool，每种备份 Level 一个，各有不同的 Retention 值。

如果你想跳到运行作业，可以跳过接下来几节。

### Pool 限制

你会注意到我创建的每个 Pool 都有两个最大设置。除非你另行告知，Bacula 会一直使用一个 Volume。我选择通过指定 Maximum Volume Bytes 来限制我的 Pool。当一个 Volume 达到此大小，Bacula 会停止使用它并创建另一个。这种自动创建之所以能发生，是因为这些指令：

1. Pool 有 LabelFormat，
2. `bacula-sd.conf` 中的 Device 设置了 `LabelMedia = yes`。

我还在每个 Pool 中指定了最多 5 个 Volume。这把每个 Pool 的大小限制在 25G。显然，这些只是入门用的值。等东西跑起来后再相应调整你的值。

Pool 文档中还有许多此类选项。

### Retention

当你初次想到 Retention 时，多数人会考虑备份保留多久。就 Bacula 而言，这并非严格正确的定义。Bacula Retention 指的是 Catalog，指定元数据在数据库中保留多久。Catalog 记录哪个 Job 备份了哪个 File、它位于哪个 Volume。

Bacula 使用三个 Retention 周期：

1. File Retention
2. Job Retention
3. Volume Retention

File Retention 与 Job Retention 在 `bacula-dir.conf` 的每个 Client 资源中指定。我没有为我的 Client 指定任何值，因此默认分别为 60 天与 180 天。

Volume Retention 在 Pool 定义中指定。至关重要的是 Volume Retention 的起始点。Volume Retention 周期从一个 Volume 不再可追加时开始。Volume Retention 的倒计时只有在 Volume 状态变为 Append 以外的状态（例如 Full、Used、Purged 等）后才开始。

通过 Pruning 从 Catalog 中移除元数据。你会记得我的每个 Pool 都有 `AutoPrune = yes`。

请留意，一旦某个备份的 File Retention 已过（且 Catalog 已被 prune），你将无法从该备份中选择要恢复的文件。你需要恢复整个 Job，而一旦该备份的 Job Retention 已过（且 Catalog 已被 prune），你将无法在不恢复到 `bextract` 或 `bscan` 的情况下恢复该备份。如果不得不这样做，那绝不会是你这一周的高光时刻。

磁盘空间便宜，时间不便宜。把 File 与 Job Retention 设高，让恢复更容易。我认为我的 Catalog 数据库 dump 并压缩后大约 10GB。建议不要为节省磁盘空间而不必要地降低 Job 或 File Retention。

我把 File 与 Job Retention 设为较大的值。然后让 Volume Retention 决定元数据保留多久。这样，我可以把同一个备份复制到不同的 Pool，让该 Pool 的 Volume Retention 决定该备份保留多久。

#### 恢复被 Prune 的数据

如果已发生 Pruning，你可以使用 `bscan`。我希望你永远不必在紧急情况下使用它。我把 `bscan` 视为万不得已的最后手段。

### 回收

按设计，Bacula 避免覆盖备份。它会竭尽全力避免这样做。你必须施加表明允许 Bacula 覆盖的限制。如果不这样做，Bacula 会一直追加到现有 Volume，直到空间耗尽。注意：磁盘空间监控不在本文范围内；使用 Nagios。

一旦施加限制，回收即可发生，Bacula 即可覆盖 Volume。这些限制的细节在文档中定义。我在此不深入回收，但此处给出的示例非常严格。

### 运行 Job

我们来运行第一个作业。哦……嗯，我们还有些最后的事情要做。

### 最后的事

以下目录同时被 `bacula-dir` 与 `bacula-sd` 使用：

```sh
mkdir -p /usr/local/bacula/working
chown bacula:bacula /usr/local/bacula/working
```

以下日志由 `bacula-dir` 使用：

```sh
touch /var/log/bacula.log
chown bacula:bacula /var/log/bacula.log
```

以下内容允许入站数据库连接。你可能希望稍后收紧这一点，也许使用密码。我把这条加到 **/usr/local/pgsql/data/pg_hba.conf** 末尾：

```sh
host bacula bacula 10.55.0.76/32 trust
```

这里是 `bacula-sd` 配置为存储备份的位置。这里是你的 Volume 将存储的位置。这在 `bacula-sd.conf` 中设置。

```sh
mkdir -p /usr/local/bacula/volumes
chown bacula:bacula /usr/local/bacula/volumes
```

### 启动守护进程

用这些命令，我启动了 Bacula 守护进程：

```sh
# service bacula-fd start
# service bacula-sd start
# service bacula-dir start
```

它们可以按任何顺序启动。

### 运行 `bconsole`

`bconsole` 是你的朋友。你可以用 `bconsole` 运行备份与恢复。启动时你应当看到如下内容：

```sh
# bconsole
Connecting to Director 10.55.0.76:9101
1000 OK: 1 MyBaculaDirector Version: 7.0.5 (28 July 2014)
Enter a period to cancel a command.
*
```

如果没有看到，你可能需要验证 `bacula-dir` 是否在运行。

### 状态

本节中，我们将在 Director、Client 与 Storage 上运行 status。

```sh
*status director
MyBaculaDirector Version: 7.0.5 (28 July 2014) amd64-portbld-freebsd9.1
freebsd 9.1-RELEASE-p19
Daemon started 10-Jan-15 20:14. Jobs: run=0,running=0 mode=0,0
Heap: heap=0 smbytes=253,680 max_bytes=254,453 bufs=168 max_bufs=176

Scheduled Jobs:
Level Type Pri Scheduled Job Name Volume
==========================================================================================
Differential Backup 10 11-Jan-15 08:15 MyFirstJob *unknown*
====

Running Jobs:
Console connected at 10-Jan-15 20:20
No Jobs running.
====
No Terminated Jobs.
====
*
```

你可以缩写命令。我本可以输入 `st di` 并得到相同输出。

```sh
*st client
Automatically selected Client: MyFirstBaculaClient
Connecting to Client MyFirstBaculaClient at 10.55.0.76:9102

MyFirstBaculaClient Version: 7.0.5 (28 July 2014) amd64-portbld-freebsd9.1 freebsd 9.1-RELEASE-p19
Daemon started 10-Jan-15 20:14. Jobs: run=0 running=0.
Heap: heap=0 smbytes=185,618 max_bytes=185,765 bufs=50 max_bufs=51
Sizes: boffset_t=8 size_t=8 debug=0 trace=0 mode=0,0 bwlimit=0kB/s

Running Jobs:
Director connected at: 10-Jan-15 20:22
No Jobs running.
====

Terminated Jobs:
====
*
```

由于我们只有一个 Client，`bacula-dir` 自动选择该 Client 并向我们展示状态。执行该命令时，`bacula-dir` 联系 `bacula-fd` 询问其状态。看到此类输出可确认 `bacula-dir` 拥有该 client 的正确凭据。这对运行作业至关重要。

让我们检查 storage。

```sh
*st stor
Automatically selected Storage: MyFirstStorage
Connecting to Storage daemon MyFirstStorage at 10.55.0.76:9103

MyBaculaStorage Version: 7.0.5 (28 July 2014) amd64-portbld-freebsd9.1
freebsd 9.1-RELEASE-p19
Daemon started 10-Jan-15 20:26. Jobs: run=0,running=0.
Heap: heap=0 smbytes=187,799 max_bytes=312,674 bufs=56 max_bufs=57
Sizes: boffset_t=8 size_t=8 int32_t=4 int64_t=8 mode=0,0

Running Jobs:
No Jobs running.
====

Jobs waiting to reserve a drive:
====

Terminated Jobs:
====

Device status:

Device "MyFirstStorage" (/usr/local/bacula/volumes) is not open.
==
====

Used Volume status:
====

====

*
```

不必为 ‘Device “MyFirstStorage” (/usr/local/bacula/volumes) is not open’ 这条消息担心。当没有作业进行时，这完全正常。

### 你的第一个作业

我们来运行一个 Job。

```sh
*run
Automatically selected Catalog: MyCatalog
Using Catalog "MyCatalog"
A job name must be specified.
Automatically selected Job: MyFirstJob
Run Backup job
JobName: MyFirstJob
Level: Incremental
Client: MyFirstBaculaClient
FileSet: MyFirstFileSet
Pool: FullFile (From Job resource)
Storage: MyFirstStorage (From Pool resource)
When: 2015-01-10 20:35:08
Priority: 10
OK to run? (yes/mod/no): yes
Job queued. JobId=4
You have messages.
*m
10-Jan 20:35 MyBaculaDirector JobId 4: No prior Full backup Job record found.
10-Jan 20:35 MyBaculaDirector JobId 4: No prior or suitable Full backup
found in catalog. Doing FULL backup.
*m
10-Jan 20:35 MyBaculaDirector JobId 4: Start Backup JobId 4,Job=MyFirstJob.
2015-01-10_20.35.10_03
10-Jan 20:35 MyBaculaDirector JobId 4: Created new Volume="FullAuto-0001",Pool=
"FullFile",MediaType="MyMediaType" in catalog.
10-Jan 20:35 MyBaculaDirector JobId 4: Using Device "MyFirstStorageDevice"
to write.
10-Jan 20:35 MyBaculaStorage JobId 4: Labeled new Volume "FullAuto-0001" on
file device "MyFirstStorageDevice" (/usr/local/bacula/volumes).
10-Jan 20:35 MyBaculaStorage JobId 4: Wrote label to prelabeled Volume "FullAuto-
0001" on file device "MyFirstStorageDevice" (/usr/local/bacula/volumes)
*m
You have no messages.
*m
You have no messages.
*m
You have no messages.
*m
10-Jan 20:35 MyBaculaStorage JobId 4: Elapsed time=00:00:10, Transfer
rate=4.517 K Bytes/second
*m
10-Jan 20:35 MyBaculaDirector JobId 4: Bacula MyBaculaDirector 7.0.5 (28Jul14):
Build OS: amd64-portbld-freebsd9.1 freebsd 9.1-RELEASE-p19
JobId: 4
Job: MyFirstJob.2015-01-10_20.35.10_03
Backup Level: Full (upgraded from Incremental)
Client: "MyFirstBaculaClient" 7.0.5 (28Jul14) amd64-port
bld-freebsd9.1,freebsd,9.1-RELEASE-p19
FileSet: "MyFirstFileSet" 2015-01-10 20:27:51
Pool: "FullFile" (From Job FullPool override)
Catalog: "MyCatalog" (From Client resource)
Storage: "MyFirstStorage" (From Pool resource)
Scheduled time: 10-Jan-2015 20:35:08
Start time: 10-Jan-2015 20:35:13
End time: 10-Jan-2015 20:35:23
Elapsed time: 10 secs
Priority: 10
FD Files Written:20
SD Files Written:20
FD Bytes Written:42,576 (42.57 KB)
SD Bytes Written:45,179 (45.17 KB)
Rate: 4.3 KB/s
Software Compression: None
VSS: no
Encryption: no
Accurate: no
Volume name(s): FullAuto-0001
Volume Session Id: 2
Volume Session Time: 1420921921
Last Volume Bytes: 46,495 (46.49 KB)
Non-fatal FD errors: 0
SD Errors: 0
FD termination status: OK
SD termination status: OK
Termination: Backup OK

10-Jan 20:35 MyBaculaDirector JobId 4: Begin pruning Jobs older than 6 months .
10-Jan 20:35 MyBaculaDirector JobId 4: No Jobs found to prune.
10-Jan 20:35 MyBaculaDirector JobId 4: Begin pruning Files.
10-Jan 20:35 MyBaculaDirector JobId 4: No Files found to prune.
10-Jan 20:35 MyBaculaDirector JobId 4: End auto prune.

*
```

完成。你的第一个备份！

不过，看到那个 JobId 4 了吗？是的，我花了四次尝试才让第一个作业跑起来。我有些配置要修复和调整。如果你复制配置文件、调整 IP 地址并完成所有最后的步骤，应该不会遇到这样的问题。

这里有一些说明帮你理解输出。编号对应上面输出的行号。

- 18–`m` 是 messages 的缩写，会显示 Director 的所有待处理消息
- 20–该作业提升为 Full。当不存在可作 Incremental 或 Differential 基础的 Job 时会发生
- 23–在 FullFile Pool 中创建了一个新 Volume，使用指定的标签格式。由于我们备份到磁盘，文件名会与 Volume 名匹配
- 25–标签也存储在 Volume 内部
- 51-52–保存了 20 个文件。你可能想知道为什么是 20 个文件？这比我配置里的多得多。

我们来看看：

```sh
[root@baculatest:/usr/local/etc/bacula] # find .
.
./INSTALLED
./INSTALLED/bacula-dir.conf
./INSTALLED/bacula-dir.conf.sample
./INSTALLED/bacula-sd.conf.sample
./INSTALLED/bacula-barcodes.sample
./INSTALLED/bacula-fd.conf
./INSTALLED/bacula-sd.conf
./INSTALLED/bacula-fd.conf.sample
./INSTALLED/bacula-barcodes
./INSTALLED/bconsole.conf
./INSTALLED/bconsole.conf.sample
./bacula-dir.conf~
./bacula-dir.conf
./filesets.conf
./pools.conf
./bacula-fd.conf
./client-myclient.conf
./schedules.conf
./bacula-sd.conf
./bconsole.conf
```

我把默认安装的文件移到了一个名为 INSTALLED 的子目录。有多少个文件？

```sh
[root@baculatest:/usr/local/etc/bacula] # find . | wc -l
21
```

21 个文件？只备份了 20 个？Bacula 按 FileSet 中的指定忽略了 `bacula-dir.conf~`。

### 恢复

你只需要一个 Restore Job。恢复是手动进行的，虽然你可以脚本化。所有 Job 参数都可从 `bconsole` 中更改。这意味着你不必指定所有可能需要的组合。

```sh
*restore
Using Catalog "MyCatalog"
No Restore Job Resource found in bacula-dir.conf.
You must create at least one before running this command.
*
```

哦。让我们把那个 Restore Job 加到 `bacula-dir.conf`。

```sh
Job {
  Name = "RestoreFiles"
  Type = Restore
  Client = MyFirstBaculaClient
  FileSet = MyFirstFileSet
  Storage = MyFirstStorage
  Schedule = MyFirstSchedule
  Messages = MyMessages
  Pool = FullFile

  Where = /tmp/bacula-restores
}
```

尽管这个 Job 引用了 `MyFirstBaculaClient`，我有意不把它加到 `client-myclient.conf` 中，因为它可用于任何 Client。

注意 Restore Job 包含一个 Where 指令。这指定了所有要恢复文件目录名的前缀。这允许把文件恢复到与备份位置不同的位置。如果未指定 Where，文件会恢复到其原始位置。Where 也可从 `bconsole` 中指定/覆盖（见后面的示例）。

我倾向于始终指定 Where 指令，作为安全/理智预防措施。

现在。我们来运行那个 Restore。

```sh
*restore
Automatically selected Catalog: MyCatalog
Using Catalog "MyCatalog"

First you select one or more JobIds that contain files
to be restored. You will be presented several methods
of specifying the JobIds. Then you will be allowed to
select which files from those JobIds are to be restored.

To select the JobIds, you have the following choices:
1: List last 20 Jobs run
2: List Jobs where a given File is saved
3: Enter list of comma separated JobIds to select
4: Enter SQL list command
5: Select the most recent backup for a client
6: Select backup for a client before a specified time
7: Enter a list of files to restore
8: Enter a list of files to restore before a specified time
9: Find the JobIds of the most recent backup for a client
10: Find the JobIds for a backup for a client before a specified time
11: Enter a list of directories to restore for found JobIds
12: Select full restore to a specified Job date
13: Cancel
Select item: (1-13):
```

在所列选项中，我推荐 #5 与 #6。其余的也不错，但这俩是我最常用的。

我们只有一个备份，所以我选 #5：

```sh
Select item: (1-13): 5
Automatically selected Client: MyFirstBaculaClient
Automatically selected FileSet: MyFirstFileSet
+----+----+-----+-----+------+------+
|jobid | level | jobfiles |jobbytes | starttime | volumename |
+----+----+-----+-----+------+------+-------+
|4 | F | 20 |42,576 | 2015-01-10 20:35:13 | FullAuto-0001|
+----+----+-----+-----+------+------+-------+
You have selected the following JobId: 4

Building directory tree for JobId(s) 4 . . .
18 files inserted into the tree.

You are now entering file selection mode where you add (mark) and
remove (unmark) files to be restored. No files are initially added, unless
you used the "all" keyword on the command line.
Enter "done" to leave this mode.

cwd is: /
$
```

Bacula 自动选择了 Client 与 FileSet，因为各只有一个。

18 个文件？我以为我们备份了 20 个文件？是的我们备份了，而其中两个“文件”是目录。此时我们处于一个特殊的 Bacula“shell”中，可以在目录中移动并选择要恢复的文件。我们本可以指定 `restore all` 来自动标记所有文件以恢复，或现在发出 `restore all` 命令。

我将标记 **/usr/local/etc/bacula** 中的所有 `*.conf` 文件。

```sh
$ ls
usr/
$ cd usr/local/etc
cwd is: /usr/local/etc/
$ ls
bacula/
$ cd bacula
cwd is: /usr/local/etc/bacula/
$ ls
INSTALLED/
bacula-dir.conf
bacula-fd.conf
bacula-sd.conf
bconsole.conf
client-myclient.conf
filesets.conf
pools.conf
schedules.conf
$ mark *.conf
8 files marked.
$ done
Bootstrap records written to
/usr/local/bacula/working/MyBaculaDirector.restore.2.bsr

The Job will require the following (*=>InChanger):
Volume(s) Storage(s) SD Device(s)
===========================================================================

FullAuto-0001 MyFirstStorage 25 MyFirstStorageDevice

Volumes marked with "*" are in the Autochanger.


8 files selected to be restored.

Run Restore job
JobName: RestoreFiles
Bootstrap: /usr/local/bacula/working/MyBaculaDirector.restore.2.bsr
Where: /tmp/bacula-restores
Replace: always
FileSet: MyFirstFileSet
Backup Client: MyFirstBaculaClient
Restore Client: MyFirstBaculaClient
Storage: MyFirstStorage
When: 2015-01-10 21:17:48
Catalog: MyCatalog
Priority: 10
OK to run? (yes/mod/no):
```

你现在可以用 `mod` 命令覆盖任何 Job 指令。我直接回答 yes，让文件进入 **/tmp/bacula-restores**。

```sh
OK to run? (yes/mod/no): yes
Job queued. JobId=5
*m
10-Jan 21:20 MyBaculaDirector JobId 5: Start Restore Job RestoreFiles.
2015-15 01-10_21.20.07_04
10-Jan 21:20 MyBaculaDirector JobId 5: Using Device "MyFirstStorageDevice" to read.
*m
10-Jan 21:20 MyBaculaStorage JobId 5: Ready to read from volume
"FullAuto-0001" on file device "MyFirstStorageDevice"
(/usr/local/bacula/volumes).
10-Jan 21:20 MyBaculaStorage JobId 5: Forward spacing Volume
"FullAuto-0001" to file:block 0:216.
10-Jan 21:20 MyBaculaStorage JobId 5: Elapsed time=00:00:01,
Transfer rate=5.177 K Bytes/second
10-Jan 21:20 MyBaculaDirector JobId 5: Bacula MyBaculaDirector 7.0.5 (28Jul14):
Build OS: amd64-portbld-freebsd9.1 freebsd 9.1-RELEASE-p19
JobId: 5
Job: RestoreFiles.2015-01-10_21.20.07_04
Restore Client: MyFirstBaculaClient
Start time: 10-Jan-2015 21:20:09
End time: 10-Jan-2015 21:20:09
Files Expected: 8
Files Restored: 8
Bytes Restored: 4,168
Rate: 0.0 KB/s
FD Errors: 0
FD termination status: OK
SD termination status: OK
Termination: Restore OK

10-Jan 21:20 MyBaculaDirector JobId 5: Begin pruning Jobs older than 6 months .
10-Jan 21:20 MyBaculaDirector JobId 5: No Jobs found to prune.
10-Jan 21:20 MyBaculaDirector JobId 5: Begin pruning Files.
10-Jan 21:20 MyBaculaDirector JobId 5: No Files found to prune.
10-Jan 21:20 MyBaculaDirector JobId 5: End auto prune.

*
```

完成。轰。就这样。

我们来看看它们。

```sh
# cd /tmp/bacula-restores/usr/local/etc/bacula/
# ls
bacula-dir.conf bacula-fd.conf bacula-sd.conf bconsole.conf
client-myclient.conf filesets.conf pools.conf schedules.conf
```

我们在这里做一下 diff：

```sh
[root@baculatest:/tmp/bacula-restores/usr/local/etc/bacula] # diff . /usr/local/etc/bacula/
Only in /usr/local/etc/bacula/: INSTALLED
diff ./bacula-dir.conf /usr/local/etc/bacula/bacula-dir.conf
53a54,66
> Job {
> Name = "RestoreFiles"
> Type = Restore
> Client = MyFirstBaculaClient
> FileSet = MyFirstFileSet
> Storage = MyFirstStorage
> Schedule = MyFirstSchedule
> Messages = MyMessages
> Pool = FullFile
>
> Where = /tmp/bacula-restores
> }
>
Only in /usr/local/etc/bacula/: bacula-dir.conf~
```

我第一次读到时困扰了一秒，但没错，这完全正确。Restore 作业是在初始备份运行后添加的。

### 但等等！还有更多！很多很多！

你刚刚走过了不少内容。关于 Bacula 有很多要学。有许多配置选项与可能的选择。我喜欢这一点。做事没有唯一正确的方式。你可以挑选。

我很想很快尝试的是 `bacula-sd` 到 `bacula-sd` 的 Job 复制。我想把一些备份存到远程位置，而不必把磁带运过去。

本节余下内容是我在写本文时想到的各种事情。我认为它们应当被包含，值得各自独立成节。

### 我使用的 Retention 周期

我把 File 与 Job Retention 值设得非常高，高于我的 Volume Retention。一个 Volume 在其包含的所有 Job/File 从数据库（Catalog）中被 prune 之前无法被回收。通过把 File 与 Job Retention 设为 3 年，我确保我的 Volume 至少 3 年或 Volume Retention 周期（取较小者）内不受影响。

### Volume 上可能仍有你的备份

即使 File、Job 与 Volume Retention 都已过，且 Volume 状态变为 Purged，备份数据仍完整保留在该 Volume 上。直到 Volume 被回收并向 Volume 写入新数据，你的备份数据才会丢失。

这一设计考量是 Bacula 绝对不愿覆盖你数据的一部分。如果你绝对需要来自 Purged Volume 的备份，你可以用 Bacula 实用程序取回。我推荐尝试 `bls` 与 `bextract`。

你也应当研究在你的 Job 中使用 Bootstrap Files，因为它们与 `bextract` 配合使用。

### Catalog 与配置的备份

除了用 Bacula 备份我的 Catalog dump 外，我还把那个 dump、所有 Bacula 配置文件、所有 Bootstrap Files 复制到多个位置。我这样做是为了避免不得不使用 `bscan` 从 Volume 重建 Catalog。

### 问题？

`bacula-dir` 通常是最难配置和启动的守护进程。留意 **/var/log/bacula.log** 与 **/var/log/messages**。如果其他方法都失败，尝试直接从命令行启动 `bacula-dir`：

```sh
/usr/local/sbin/bacula-dir-u bacula-g bacula-v-c \
/usr/local/etc/bacula/bacula-dir.conf-f-d10
```

### 感谢阅读

我在本文中使用的 Bacula 配置文件可从 <http://www.langille.org/examples/bacula7/> 下载。

Bacula 为我提供了一个备份系统的出色平台。它每天都被使用。我备份到启用压缩与自动快照的 ZFS 文件系统。这太棒了。每天，这些备份都被复制到磁带。一些磁带存到远程位置，以防万一。希望我永远不必从真正的灾难中恢复，但我知道 Bacula 多次在我删除/覆盖某些东西时救了我。

希望它对你同样有用。

**Dan Langille** 自 1998 年起使用 FreeBSD，几乎立刻就开始记录他的经历。这份在线日记后来成了 The FreeBSD Diary。他非常擅长描述执行各种任务的分步过程，从更改提示符到创建和维护 Jail。

作为软件工程师出身，Dan 一路掌握了系统管理与数据库技能。这份热情引领他进入 [Cisco Talos 安全情报与研究小组](http://blogs.cisco.com/tag/talos2) 的当前 DevOps 工作，在那里他既做编码也做系统管理，乐在其中。

他是 BSDCan、PGCon、FreshPorts 与 FreshSource 的创始人。他热爱山地自行车，似乎在截止日期前如鱼得水。
