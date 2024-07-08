# 第九章：使用选项文件

几乎每个软件都可以进行配置，甚至必须进行配置。MySQL 在这方面并没有太大的不同。虽然默认配置可能适用于大多数安装，但很可能您最终需要配置服务器或客户端。MySQL 提供两种配置方式：通过命令行参数选项和配置文件。由于这个文件只包含可以在命令行上指定的选项，它也被称为*选项*文件。

选项文件不仅适用于 MySQL Server。严格来说，讨论选项*文件*也不完全正确，因为几乎每个 MySQL 的安装都会有多个选项文件。大多数 MySQL 软件支持在选项文件中包含内容，我们也会涵盖这部分内容。

熟悉选项文件——理解其节和选项优先级——是有效地使用 MySQL Server 和相关软件的重要部分。通过本章的学习，您应该能够轻松配置 MySQL Server 和其他使用选项文件的程序。本章将专注于文件本身。服务器本身的配置和一些调优思路在第十一章中深入讨论。

# 选项文件的结构

MySQL 中的配置文件遵循普遍存在的 INI 文件方案。简而言之，它们是普通文本文件，旨在手动编辑。当然，您可以自动化编辑过程，但这些文件的结构故意设计得非常简单。几乎每个 MySQL 配置文件都可以使用任何文本编辑器创建和修改。这条规则只有两个例外，详细介绍在“特殊选项文件”中。

为了让你了解文件结构，让我们来看看 Fedora Linux 上 MySQL 8 随附的配置文件（请注意，您系统上选项文件的确切内容可能有所不同）。为简洁起见，我们对一些行进行了编辑：

```
$ cat /etc/my.cnf
...
[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
...
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

log-error=/var/log/mysqld.log
pid-file=/run/mysqld/mysqld.pid
```

###### 提示

在某些 Linux 发行版（如 Ubuntu）中，默认的 MySQL 安装中不存在 */etc/my.cnf* 配置文件。在这些系统上，请查找 */etc/mysql/my.cnf*，或者参考“选项文件搜索顺序”获取 `mysqld` 读取的完整选项文件列表的方法。

文件结构的几个主要部分：

部分（组）标题

这些是配置参数前面的方括号中的值。所有使用选项文件的程序都会在一个或多个命名的节中寻找参数。例如，`[mysqld]` 是 MySQL 服务器使用的节，而 `[mysql]` 是 `mysql` CLI 程序使用的节。严格来说，节的名称是任意的，你可以在那里放任何东西。但是，如果你将 `[mysqld]` 改为 `[section]`，你的 MySQL 服务器将忽略在该头部之后的所有选项。

MySQL 文档将节称为*组*，但两个术语可以互换使用。

标头控制文件如何解析，以及由哪些程序解析。在部分标头之后和下一个部分标头之前的每个选项都将归属于第一个标头。示例将更清楚地说明这一点：

```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
[mysql]
default-character-set=latin1
```

在这里，`datadir`和`socket`选项位于（并将被归属于）`[mysqld]`部分，而`default-character-set`选项位于`[mysql]`下。注意，某些 MySQL 程序读取多个部分；但我们稍后会讨论这个问题。

部分标头可以交织。以下示例完全有效：

```
[mysqld]
datadir=/var/lib/mysql
[mysql]
default-character-set=latin1
[mysqld]
socket=/var/lib/mysql/mysql.sock
[mysqld_safe]
core-file-size=unlimited
[mysqld]
core-file
```

这样的配置可能对人来说阅读起来很困难，但解析文件的程序不会关心顺序。不过，尽可能保持配置文件对人类可读可能是最好的。

选项值对

这是选项文件的主要部分，由配置变量本身及其值组成。这些对都在新行上定义，并遵循两种一般模式之一。除了前面示例中显示的`*option=value*`模式之外，还有只有`*option*`模式。例如，标准 MySQL 8 配置文件具有以下行：

```
# Remove the leading "# " to disable binary logging
# Binary logging captures changes between backups and is enabled by
# default. Its default setting is log_bin=binlog
# disable_log_bin
```

`disable_log_bin`是一个没有值的选项。如果我们取消注释，MySQL 服务器将应用该选项。

使用`*option*=*value*`模式时，如果您喜欢，可以在等号周围添加空格以提高可读性。自动截断选项名称和值前后的任何空白。

选项值也可以用单引号或双引号括起来。如果您不确定值是否会被正确解释，这很有用。例如，在 Windows 上，路径包含`\`符号，该符号被视为转义符号。因此，您应该在 Windows 上将路径放在双引号中（尽管也可以通过加倍每个`\`来转义）。引用选项值在值包含`#`符号时是必需的，否则该符号将被视为评论的开始。

我们建议的经验法则是在不确定时使用引号。以下是一些有效的选项/值对，说明了前面的观点：

```
slow_query_log_file = "C:\mysqldata\query.log"
slow_query_log_file=C:\\mysqldata\\query.log
innodb_temp_tablespaces_dir="./#innodb_temp/"
```

在设置数值选项（如不同缓冲区和文件的大小）的值时，使用字节可能会很繁琐。为了简化生活，MySQL 理解多种代表不同单位的后缀。例如，以下几种定义了相同大小的缓冲池（268,435,456 字节）：

```
innodb_buffer_pool_size = 268435456
innodb_buffer_pool_size = 256M
innodb_buffer_pool_size = 256MB
innodb_buffer_pool_size = 256MiB
```

如果服务器足够大，您还可以指定`G`、`GB`和`GiB`表示吉字节，以及`T`、`TB`和`TiB`表示太字节。当然，还接受`K`和其他形式。MySQL 始终使用基于 2 的单位：1 GB 是 1,024 MB，而不是 1,000 MB。

你不能为选项指定小数值。例如，`0.25G` 对于 `innodb_buffer_pool_size` 变量是一个不正确的值。此外，不像从 `mysql` CLI 或其他客户端连接设置值时那样，你不能使用数学符号来表示选项值。你可以运行 `SET GLOBAL max_heap_table_size=16*1024*1024;`，但不能将相同的值放入选项文件中。

你甚至可以多次配置相同的选项，就像我们在 `innodb_buffer_pool_size` 中所做的那样。最后的设置将覆盖之前的设置，并且文件从上到下进行扫描。选项文件也有全局优先顺序；我们将在 “选项文件的搜索顺序” 中讨论这一点。

非常重要的一点是，设置不正确的选项名称将导致程序无法启动。当然，如果不正确的选项在特定程序不读取的部分下，那没问题。但是如果 `mysqld` 在 `[mysqld]` 下找到一个它不认识的选项，将导致启动失败。在 MySQL 8.0 中，你可以使用 `mysqld` 的 `--validate-config` 命令行参数验证部分选项文件中所做的更改。然而，这种验证仅涵盖核心服务器功能，不会验证存储引擎选项。

有时候你需要在 MySQL 启动时设置一个 MySQL 不认识的选项。例如，在配置可能在启动后加载的插件时这将很有用。你可以在这些选项前加上 `loose-` 前缀（或在命令行上使用 `--loose-`），当 MySQL 看到这些选项时只会输出警告而不是启动失败。以下是一个使用未知选项的示例：

```
# mysqld --validate-config
2021-02-11T08:02:58.741347Z 0 [ERROR] [MY-000067] [Server] ...
 ... unknown variable *audit_log_format=JSON*.
2021-02-11T08:02:58.741470Z 0 [ERROR] [MY-010119] [Server] Aborting
```

当选项更改为 `loose-audit_log_format` 后，我们看到以下内容。没有输出意味着所有选项都成功验证：

```
# mysqld --validate-config
#
```

注释

MySQL 选项文件经常被忽视但却非常重要的一个特性是能够添加注释。注释允许你包含任意文本，通常是该设置存在的描述，不会被任何 MySQL 程序解析。正如你在 `disable_log_bin` 示例中看到的，以 `#` 开头的行被视为注释。你也可以创建以分号 (`;`) 开头的注释；两者都被接受。你不一定需要一整行来写注释：它们也可以出现在行尾，尽管在这种情况下，它们必须以 `#` 而不是 `;` 开头。一旦 MySQL 在一行上找到 `#`（除非它被转义），那么在这个点之后的所有内容都将被视为注释。以下是一个有效的配置行：

```
innodb_buffer_pool_size = 268435456 # 256M
```

包含指令

配置文件（以及整个目录）可以在其他配置文件中包含。这样可以更轻松地管理复杂的配置，但也使得阅读选项更加困难，因为与程序不同，人类不能轻松地将文件合并在一起。然而，能够分离不同 MySQL 程序的配置是很有用的。例如，`xtrabackup`实用程序（见第十章）没有任何特殊的配置文件，并读取标准系统选项文件。通过包含，您可以将`xtrabackup`的配置整齐地组织在一个专用文件中，并清理主要的 MySQL 选项文件。然后可以这样包含它：

```
$ cat /etc/my.cnf
!include /etc/mysql.d/xtrabackup.cnf
...
```

您可以看到*/etc/my.cnf*包含*/etc/mysql.d/xtrabackup.cnf*文件，该文件反过来在`[xtrabackup]`部分列出了一些配置选项。

不需要在不同的文件中拥有不同的部分。例如，Percona XtraDB Cluster 在`[mysqld]`部分下有`wsrep`库配置选项。有许多这样的配置，并且它们在您的*my.cnf*中并不一定有用。您可以创建一个单独的文件，例如*/etc/mysql.d/wsrep.conf*，并在那里列出`[mysqld]`部分下的`wsrep`变量。任何读取主*my.cnf*文件的程序也将读取所有包含的文件，然后才解析不同部分下的变量。

当创建了大量此类额外的配置文件时，您可能希望直接包含包含它们的整个目录或目录，而不是包含每个单独的选项文件。可以通过另一个指令`includedir`来实现，它期望一个目录路径作为参数：

```
!includedir /etc/mysql.d
```

MySQL 程序将把该路径理解为一个目录，并尝试包含该目录树中的每个选项文件。在类 Unix 系统上，包括*.cnf*文件；在 Windows 上，包括*.cnf*和*.ini*文件。

通常，包含是在特定配置文件的开头定义的，但这并非强制要求。您可以将包含视为将包含文件的内容附加到父文件中的任何位置；在文件中定义包含的位置，将包含文件的内容放在其下。实际上，事情会更加复杂，但这种心理模型在例如思考选项优先级时是有效的，我们在“选项文件搜索顺序”中讨论了这一点。

每个包含的文件必须至少定义一个配置部分。例如，它可能在开头有`[mysqld]`。

空行

在选项文件中空行没有意义。您可以使用它们在视觉上分隔部分或单独的选项，以使文件更易于阅读。

# 选项的作用域

我们可以从两个角度谈论 MySQL 中的选项作用域。首先，每个单独的选项可以具有全局作用域、会话作用域或两者兼有，并且可以动态或静态设置。其次，我们可以讨论在选项文件中设置的选项如何通过部分作用域以及选项文件本身的作用域和优先级顺序。

部分标题定义了特定程序（或多个程序，因为没有什么能阻止一个程序读取多个部分）意图在特定标题下读取选项。一些配置选项在其部分之外是没有意义的，但有些可以在多个部分下定义，并不一定需要设置相同的值。

让我们考虑一个示例，在此示例中，我们有一个 MySQL 服务器为了兼容性原因配置为`latin1`字符集。然而，现在有新的表使用了`utf8mb4`字符集。我们希望我们的`mysqldump`逻辑导出仅使用 UTF-8，因此我们希望为这个程序覆盖字符集设置。方便的是，`mysqldump`读取自己的配置部分，因此我们可以编写如下的选项文件：

```
[mysqld]
character_set_server=latin1
[mysqldump]
default_character_set=utf8mb4
```

这个小例子展示了如何在不同的层级上设置选项。在这个特定的案例中，我们使用了不同的选项，但也可以在不同的作用域中使用相同的选项。例如，假设我们想要限制未来的`BLOB`和`TEXT`值（参见“字符串类型”）的大小为 32 MiB，但我们已经有大小达到 256 MiB 的行。我们可以通过如下配置为本地客户端添加一个人为的屏障：

```
[mysqld]
max_allowed_packet=256M
[client]
max_allowed_packet=32M
```

MySQL 服务器的`max_allowed_packet`值将在全局作用域上设置，并作为最大查询大小的硬限制（也作用于`BLOB`或`TEXT`字段大小）。客户端的值将在会话作用域上设置，并作为软限制。如果特定客户端需要更大的值（例如读取旧行），可以使用`SET`语句提升到服务器的限制。

选项文件本身也有不同的作用域。MySQL 选项文件可以按全局、客户端、服务器和额外分组：全局选项文件被所有或大多数 MySQL 程序读取，而客户端和服务器文件分别仅被客户端程序和`mysqld`读取。由于可以指定额外的配置文件供程序读取，我们也列出了“额外”类别。

让我们概述一下常规 MySQL 8.0 安装在 Linux 和 Windows 上安装和读取的选项文件。我们将从 Windows 开始，在表 9-1 中详细说明。

表 9-1\. Windows 上的 MySQL 选项文件

| 文件名 | 作用域和目的 |
| --- | --- |
| *%WINDIR%\my.ini*, *%WINDIR%\my.cnf* | 全程序读取的全局选项 |
| *C:\my.ini*, *C:\my.cnf* | 全程序读取的全局选项 |
| *BASEDIR**\my.ini*, *BASEDIR**\my.cnf* | 全程序读取的全局选项 |
| 额外配置文件 | 可选指定的文件，使用`--defaults-extra-file` |
| *%APPDATA%\MySQL\.mylogin.cnf* | 登录路径配置文件 |
| *DATADIR**\mysqld-auto.cnf* | 用于持久化变量的选项文件 |

表 9-2 将 Fedora Linux 上典型安装的选项文件进行了详细拆解。

表 9-2\. Fedora Linux 上的 MySQL 选项文件

| 文件名 | 范围和目的 |
| --- | --- |
| */etc/my.cnf*, */etc/mysql/my.cnf*, */usr/etc/my.cnf* | 所有程序读取的全局选项 |
| *$MYSQL_HOME/my.cnf* | 只有在变量设置时才会读取的服务器选项 |
| *~/.my.cnf* | 特定操作系统用户运行的所有程序读取的全局选项 |
| 额外配置文件 | 可以通过 `--defaults-extra-file` 指定的文件 |
| *~/.mylogin.cnf* | 特定操作系统用户下的登录路径配置文件 |
| *DATADIR**/mysqld-auto.cnf* | 用于持久化变量的选项文件 |

在 Linux 上，很难找到一个通用的完整的配置文件列表，因为不同 Linux 发行版的 MySQL 包可能会读取略有不同的文件或位置。作为一条经验法则，在 Linux 上，*/etc/my.cnf* 是一个很好的起点，而在 Windows 上则是 *%WINDIR%\my.cnf* 或 *BASEDIR**\my.cnf*。

我们列出的一些配置文件在不同系统中的路径可能略有不同。*/usr/etc/my.cnf* 也可以写成 *SYSCONFIGDIR**/my.cnf*，路径在编译时定义。*$MYSQL_HOME/my.cnf* 只有在设置了该变量时才会读取。默认打包的 `mysqld_safe` 程序（用于启动 `mysqld` 守护进程）在运行 `mysqld` 前会将 `$MYSQL_HOME` 设置为 *BASEDIR*。你不会在任何操作系统用户的环境中找到设置了 `$MYSQL_HOME`，该变量的设置仅在手动启动 `mysqld` 时有效—也就是说，不使用 `service` 或 `systemctl` 命令。

在 Windows 和 Linux 之间有一个显著的差异。在 Linux 上，MySQL 程序会读取位于给定操作系统用户家目录下的一些配置文件。在 表 9-2 中，家目录用 `~` 表示。而在 Windows 上，MySQL 则缺乏这种能力。这类配置文件的一个常见用途是基于其操作系统用户控制客户端选项。通常，它们会包含凭据。但是，在 “特殊选项文件” 中描述的登录路径工具使这种功能变得多余。

使用`--defaults-extra-file`在每次读取全局文件后读取额外的配置文件，在表中的位置。例如，当您想要运行程序以测试新变量时，这是一个有用的选项。但是，过度使用此选项可能会导致难以理解当前生效的选项集（请参阅“确定生效的选项”）。`--defaults-extra-file`选项并非唯一可以修改选项文件处理方式的选项。`--no-defaults`阻止程序完全读取*任何*配置文件。`--defaults-file`强制程序读取单个文件，如果您将自定义配置全部放在一个地方，这将非常有用。

到目前为止，您应该对大多数 MySQL 安装使用的选项文件有了牢固的理解。下一节将更详细地讨论不同程序如何按不同顺序读取不同文件，并从这些文件中读取哪些特定组或组。

# 选项文件搜索顺序

此时，您应该了解选项文件的结构以及它们的位置。大多数 MySQL 程序都会读取一个或多个选项文件，了解程序搜索和读取这些文件的特定顺序非常重要。本节涵盖了搜索顺序和选项优先级的主题，并讨论了它们的重要性。

如果 MySQL 程序读取任何选项文件，您可以找到它读取的具体文件以及读取它们的顺序。配置文件的一般顺序将与表 9-1 和表 9-2 中概述的完全相同或非常相似。您可以使用以下命令查看确切的顺序：

```
$ mysqld --verbose --help | grep "Default options" -A2
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf
The following groups are read: mysqld server mysqld-8.0
```

在 Windows 上，您需要运行`mysqld.exe`而不是`mysqld`，但输出将保持不变。该输出包括读取的配置文件列表及其顺序。您还可以看到`mysqld`读取的选项组列表：`[mysqld]`、`[server]`和`[mysqld-8.0]`。请注意，您可以通过添加`--defaults-group-suffix`选项修改任何程序读取的选项组列表：

```
$ mysqld --defaults-group-suffix=-test --verbose --help | grep "groups are read"
The following groups are read: mysqld server mysqld-8.0 ...
... mysqld-test server-test mysqld-8.0-test
```

您现在知道哪些选项文件和选项组被读取了。但是，了解这些选项文件的优先级顺序也很重要。毕竟，不会阻止您在多个配置文件中设置一个或多个选项。对于 MySQL 程序而言，配置文件的优先级顺序很简单：稍后读取的文件中的选项优先于先前读取的文件中的选项。直接作为命令行参数传递给命令的选项优先于任何配置文件中的配置选项。

在表 9-1 和表 9-2 中，文件按从上到下的顺序读取。列表中配置文件越低，该处选项的“权重”越高。例如，对于非`mysqld`程序，*.mylogin.cnf*中的值优先于任何其他配置文件中的值，并且仅低于通过命令行参数设置的值。对于`mysqld`，在`*DATADIR*` */mysqld-auto.cnf*中设置的持久变量也是如此。

通过包含指令在其他文件中包含配置文件的能力使事情变得稍微复杂了些，但您始终可以在表 9-1 和表 9-2 中列出的一个或多个选项文件中包含额外的文件。您可以将此视为 MySQL 在读取父配置文件之前将包含的文件附加到其父配置文件的过程中，将每个文件插入到包含指令之后。因此，全局选项的优先级是父配置文件的优先级。在生成的文件本身中（按顺序添加了所有包含的文件），后定义的选项优先于先定义的选项。

# 特殊选项文件

MySQL 使用两个特殊配置文件，这两个文件是与“选项文件结构”中概述的结构不同的例外情况。

## 登录路径配置文件

首先，有一个*.mylogin.cnf*文件，它作为*登录路径*系统的一部分。尽管您可以认为其结构类似于常规选项文件，但这个特定文件不是常规文本文件。事实上，它是一个加密的文本文件。该文件旨在通过 MySQL 提供的特殊程序`mysql_config_editor`创建和修改，通常在客户端包中提供。它是加密的，因为*.mylogin.cnf*（以及整个登录路径系统）的目的是以方便和安全的方式存储 MySQL 连接选项，包括密码。

默认情况下，`mysql_config_editor`和其他 MySQL 程序将在 Linux 和各种 Unix 版本的当前用户的*HOME*中以及 Windows 的*%APPDATA%\MySQL*中查找*.mylogin.cnf*。可以通过设置`MYSQL_TEST_LOGIN_FILE`环境变量来更改文件的位置和名称。

如果尚不存在此文件，您可以通过在其中存储`root`用户的密码来创建此文件：

```
$ mysql_config_editor set --user=root --password
Enter password:
```

输入密码并确认输入后，我们可以查看文件的内容：

```
$ ls -la ~/.mylogin.cnf
-rw-------. 1 skuzmichev skuzmichev 100 Jan 18 18:03 .mylogin.cnf
$ cat ~/.mylogin.cnf

>pZ
   prI
         R86w"># &.h.m:4+|DDKnl_K3>73x$
$ file ~/.mylogin.cnf
.mylogin.cnf: data
$ file ~/.my.cnf
.my.cnf: ASCII text
```

正如您所见，*.mylogin.cnf*至少在表面上不是常规配置文件。因此，它需要特殊处理。除了创建文件外，您还可以使用`mysql_config_editor`查看和修改*.mylogin.cnf*。让我们从如何实际查看其中内容开始。该选项是`print`：

```
$ mysql_config_editor print
[client]
user = "root"
password = *****
```

`client`是默认的登录路径。在没有显式登录路径规范的情况下使用`mysql_config_editor`执行的所有操作都会影响`client`登录路径。在之前运行`set`时我们没有指定任何登录路径，所以`root`的凭据被写入`client`路径下。但是，可以针对任何操作指定特定的登录路径。让我们将`root`的凭据放在名为`root`的登录路径下：

```
$ mysql_config_editor set --login-path=root --user=root --password
Enter password:
```

要指定登录路径，请使用`--login-path`或`-G`选项，并在使用`print`时添加`--all`选项查看所有路径：

```
$ mysql_config_editor print --login-path=root
[root]
user = root
password = *****
$ mysql_config_editor print --all
[client]
user = root
password = *****
[root]
user = root
password = *****
```

你可以看到输出结果类似于选项文件，因此可以将*.mylogin.cnf*视为一个具有特殊处理的选项文件。只是不要手动编辑它。说到编辑，让我们在`set`命令中添加一些更多选项，正如`mysql_config_editor`所称呼的那样。我们将在此过程中创建一个新的登录路径。

`mysql_config_editor`支持`--help`（或`-?`）参数，可以与其他选项结合使用，以获取有关`print`或`set`的帮助。让我们从查看稍微缩短的`set`的帮助输出开始：

```
$ mysql_config_editor set --help
...
MySQL Configuration Utility.

Description: Write a login path to the login file.
Usage: mysql_config_editor [program options] [set [command options]]
  -?, --help          Display this help and exit.
  -h, --host=name     Host name to be entered into the login file.
  -G, --login-path=name
                      Name of the login path to use in the login file. (Default
                      : client)
  -p, --password      Prompt for password to be entered into the login file.
  -u, --user=name     User name to be entered into the login file.
  -S, --socket=name   Socket path to be entered into login file.
  -P, --port=name     Port number to be entered into login file.
  -w, --warn          Warn and ask for confirmation if set command attempts to
                      overwrite an existing login path (enabled by default).
                      (Defaults to on; use --skip-warn to disable.)
...
```

在这里你可以看到*.mylogin.cnf*的另一个有趣属性：你不能随意向其中添加参数。现在我们知道，我们基本上只能设置与登录到 MySQL 实例或实例相关的少量选项，这当然是“登录路径”文件的预期。现在，让我们回到编辑文件：

```
$ mysql_config_editor set --login-path=scott --user=scott
$ mysql_config_editor set --login-path=scott --user=scott
WARNING : *scott* path already exists and will be overwritten.
 Continue? (Press y|Y for Yes, any other key for No) : y
$ mysql_config_editor set --login-path=scott --user=scott --skip-warn
```

在这里，我们展示了`mysql_config_editor`在修改或创建登录路径时可能表现出的所有行为。如果登录路径尚不存在，则不会产生警告。如果已经存在这样的路径，则会打印警告和确认，但仅当未指定`--skip-warn`时。请注意，我们在这里讨论的是整个登录路径！不可能修改路径的单个属性：每次都会写出整个登录路径。如果要更改单个属性，您需要同时指定所有其他需要的属性。

让我们添加一些更多细节并查看结果：

```
$ mysql_config_editor set --login-path=scott \
--user=scott --port=3306 --host=192.168.122.1 \
--password --skip-warn
Enter password:
$ mysql_config_editor print --login-path=scott
[scott]
user = scott
password = *****
host = 192.168.122.1
port = 3306
```

## 持久系统变量配置文件

第二个特殊文件是*mysqld-auto.cnf*，自 MySQL 8.0 以来一直存在于数据目录中。它是新持久系统变量功能的一部分，允许您使用常规的`SET`语句在磁盘上更新 MySQL 选项。在此之前，您无法从数据库连接中更改 MySQL 的配置。通常的流程是在磁盘上更改选项文件，然后运行`SET GLOBAL`语句以在线更改配置变量。正如您可以想象的那样，这可能会导致错误，例如仅在线进行更改。新的`SET PERSIST`语句负责处理这两个任务：在线更新的变量也会在磁盘上更新。还可以仅在磁盘上更新变量。

该文件本身出人意料地与 MySQL 中的任何其他配置文件都不同。虽然 *.mylogin.cnf* 是一个加密但仍然是常规选项文件，*mysqld-auto.cnf* 使用了一个常见但完全不同的格式：JSON。

在持久化任何内容之前，*mysqld-auto.cnf* 不存在。因此，我们将首先更改一个系统变量：

```
mysql> `SELECT` `@``@``GLOBAL``.``max_connections``;`
```

```
+--------------------------+
| @@GLOBAL.max_connections |
+--------------------------+
|                      100 |
+--------------------------+
1 row in set (0.00 sec)
```

```
mysql> `SET` `PERSIST` `max_connections` `=` `256``;`
```

```
Query OK, 0 rows affected (0.01 sec)
```

```
mysql> `SELECT` `@``@``GLOBAL``.``max_connections``;`
```

```
+--------------------------+
| @@GLOBAL.max_connections |
+--------------------------+
|                      256 |
+--------------------------+
1 row in set (0.00 sec)
```

预期地，在全局范围内在线更新了变量。现在让我们来探索生成的配置文件。因为我们知道内容以 JSON 格式存在，我们将使用 `jq` 实用程序对其进行良好格式化。这不是必需的，但可以使文件更易于阅读：

```
$ cat /var/lib/mysql/mysqld-auto.cnf | jq .
{
  "Version": 1,
  "mysql_server": {
    "max_connections": {
      "Value": "256",
      "Metadata": {
        "Timestamp": 1611728445802834,
        "User": "root",
        "Host": "localhost"
      }
    }
  }
}
```

只需查看包含单个变量值的此文件，您就可以看到为什么纯 *.ini* 用于预计由人类编辑的配置文件。这太冗长了！但是，JSON 对计算机的阅读非常好，因此非常适合 MySQL 本身编写和读取的配置文件。作为附加好处，您还可以获得更改的审核：正如您所看到的，`max_connection` 属性包含元数据，其中包含更改发生的时间和更改的作者。

由于这是一个文本文件，与二进制的登录路径配置文件不同，可以手动编辑 *mysqld-auto.cnf*。但是，不太可能有许多需要这样做的情况。

# 确定正在生效的选项

几乎所有与 MySQL 一起工作的人都会面临的最后例行任务是查找变量的值，以及它们在哪些选项文件中设置（以及为什么，但有时没有技术可以帮助人类的推理！）。

到此为止，我们知道 MySQL 程序读取的文件、读取顺序及其优先级。我们还知道命令行参数会覆盖任何其他设置。但是，理解某个变量确切设置在哪里可能是一项艰巨的任务。多个文件被扫描，可能存在嵌套包含，这可能导致长时间的调查。

让我们从查看当前程序使用的选项开始。对于一些程序，如 MySQL 服务器（`mysqld`），这很容易。您可以通过运行 `SHOW GLOBAL VARIABLES` 来获取 `mysqld` 使用的当前值列表。无法更改 `mysqld` 使用的选项值而不在全局变量状态中看到影响的效果。对于其他程序，情况变得更加复杂。要了解 `mysql` 使用的选项，请运行它，然后检查 `SHOW VARIABLES` 和 `SHOW GLOBAL VARIABLES` 的输出，以查看哪些选项在会话级别上被覆盖。但即使在成功连接到服务器之前，`mysql` 必须读取或接收连接信息。

在程序启动时确定正在生效的选项列表有两种简单的方法：通过向该程序传递`--print-defaults`参数或使用特殊的`my_print_defaults`程序。让我们看看在 Linux 上执行的前一选项。您可以忽略`sed`部分，但这可以使输出对人眼更加友好：

```
$ mysql --print-defaults
mysql would have been started with the following arguments:
--user=root --password=*****
$ mysqld --print-defaults | sed 's/--/\n--/g'
/usr/sbin/mysqld would have been started with the following arguments:
```

```
--datadir=/var/lib/mysql
--socket=/var/lib/mysql/mysql.sock
--log-error=/var/log/mysqld.log
--pid-file=/run/mysqld/mysqld.pid
--max_connections=100000
--core-file
--innodb_buffer_pool_in_core_file=OFF
--innodb_buffer_pool_size=256MiB
```

这里获取的变量来自我们之前讨论过的所有选项文件。如果一个变量值被多次设置，最后出现的值将优先生效。然而，`--print-defaults`实际上会输出每个设置的选项。例如，尽管`innodb_buffer_pool_size`设置了五次，但实际生效的值将是 384 M：

```
$ mysqld --print-defaults | sed 's/--/\n--/g'
/usr/sbin/mysqld would have been started with the following arguments:

--datadir=/var/lib/mysql
--socket=/var/lib/mysql/mysql.sock
--log-error=/var/log/mysqld.log
--pid-file=/run/mysqld/mysqld.pid
--max_connections=100000
--core-file
--innodb_buffer_pool_in_core_file=OFF
--innodb_buffer_pool_size=268435456
--innodb_buffer_pool_size=256M
--innodb_buffer_pool_size=256MB
--innodb_buffer_pool_size=256MiB
--large-pages
--innodb_buffer_pool_size=384M
```

你也可以将`--print-defaults`与其他命令行参数组合使用。例如，如果你打算使用命令行参数运行程序，你可以查看它们是否会覆盖或重复已设置的配置选项值：

```
$ mysql --print-defaults --host=192.168.4.23 --user=bob | sed 's/--/\n--/g'
mysql would have been started with the following arguments:

--user=root
--password=*****
--host=192.168.4.23
--user=bob
```

打印变量的另一种方式是使用`my_print_defaults`程序。它将一个或多个部分头作为参数，并打印在扫描的文件中落入请求组的所有选项。这可能比仅需要查看一个选项组时使用`--print-defaults`更好。在 MySQL 8.0 中，`[mysqld]`程序读取以下组：`[mysqld]`、`[server]`和`[mysqld-8.0]`。选项的组合输出可能很长，但如果我们只需要查看专门为 8.0 设置的选项怎么办？例如，我们已将`[mysqld-8.0]`选项组添加到选项文件中，并在那里放置了一些配置参数值：

```
$ my_print_defaults mysqld-8.0
--character_set_server=latin1
--collation_server=latin1_swedish_ci
```

这也可以帮助其他软件，如 PXC，或者 MySQL 的 MariaDB 版本，它们都包括多个配置组。特别是，你可能希望查看`[wsrep]`部分而不包含其他选项。当然，`my_print_defaults`也可以用来输出完整的选项集；只需传递程序读取的所有部分头。例如，`[mysql]`程序读取`[mysql]`和`[client]`选项组，因此我们可以使用：

```
$ my_print_defaults mysql client
--user=root
--password=*****
--default-character-set=latin1
```

用户和密码定义来自我们之前设置的登录路径配置中的客户端组，而字符集来自常规*.my.cnf*文件中的`[mysql]`选项组。请注意，我们手动添加了该组和字符集配置；默认情况下，该选项未设置。

你可以看到，虽然两种读取选项的方式都谈论*默认值*，但它们实际上输出了我们已经显式设置的选项，使它们变成了非默认值。这是一个有趣的细节，但在整体计划中并不会改变任何事情。

不幸的是，这两种查看选项的方式都无法完美确定生效的完整选项集。问题在于它们只会读取表格 9-1 和 9-2 中列出的配置文件，但 MySQL 程序可能会读取其他配置文件或者通过命令行参数启动。此外，通过`SET PERSIST`持久化在`*DATADIR*` */mysqld-auto.cnf*中的变量不会被默认打印例程提供。

我们提到 MySQL 程序不会从除了在 9-1 和 9-2 中列出的文件之外的任何其他文件中读取选项。然而，这些列表包括“额外配置文件”，它可以位于任意位置。除非在调用 `my_print_defaults` 或带有 `--print-defaults` 的另一个程序时指定了相同的额外文件，否则不会读取来自该额外文件的选项。额外文件通过命令行参数 `--defaults-extra-file` 指定，大多数（如果不是所有）MySQL 程序都可以指定。两个默认打印例程只读预定义的配置文件，并且会忽略该文件。但是，您可以为 `my_print_defaults` 和使用 `--print-defaults` 调用的程序都指定 `--defaults-extra-file`，那么两者都将读取额外的文件。我们前面提到的 `--defaults-file` 选项也是如此，它基本上强制 MySQL 程序只读取作为此选项值传递的单个文件。

`--defaults-extra-file` 和 `--defaults-file` 共享一个共同点：它们都是命令行参数。传递给 MySQL 程序的命令行参数会覆盖从配置文件中读取的任何选项，但同时你可能会在执行 `--print-defaults` 或 `my_print_defaults` 时忽略它们，因为它们来自于配置文件之外。更简洁地说：特定的 MySQL 程序，如 `mysqld`，可能会被某人使用未知和任意的命令行参数启动。因此，在讨论选项时，实际上我们必须考虑这些参数的存在。

在 Linux 和类 Unix 系统上，您可以使用 `ps` 实用程序（或等效工具）查看当前运行进程的信息，包括它们的完整命令行。让我们看一个在 Linux 上的例子，其中 `mysqld` 使用 `--no-defaults` 启动，并且所有配置选项都作为参数传递：

```
$ ps auxf | grep mysqld | grep -v grep
root      397830  ... \_ sudo -u mysql bash -c mysqld ...
mysql     397832  ...   \_ mysqld --datadir=/var/lib/mysql ...
```

或者，如果我们只打印 `mysqld` 进程的命令行并使用 `sed` 来使其更清晰：

```
$ ps -p 397832 -ocommand ww | sed 's/--/\n--/g'
COMMAND
mysqld
--datadir=/var/lib/mysql
--socket=/var/lib/mysql/mysql.sock
--log-error=/var/log/mysqld.log
--pid-file=/run/mysqld/mysqld.pid
...
--character_set_server=latin1
--collation_server=latin1_swedish_ci
```

请注意，对于这个示例，我们启动了 `mysqld` 而没有使用任何提供的脚本。你不经常会看到以这种方式启动 MySQL 服务器，但这是可能的。

您可以将任何配置选项作为参数传递，因此输出可能会非常长。然而，当您不确定 `mysqld` 或另一个程序的执行方式时，这是一个重要的检查点。在 Windows 上，您可以通过打开任务管理器并在进程选项卡（通过查看菜单）中添加一个“命令行”列，或者使用 `sysinternals` 包中的 Process Explorer 工具来查看正在运行程序的命令行参数。

如果您的 MySQL 程序是从脚本中启动的，您应该检查该脚本以查找所有使用的参数。虽然这对于 `mysqld` 可能会是一个罕见的情况，但是从自定义脚本运行 `mysql`、`mysqldump` 和 `xtrabackup` 是一个常见的做法。

理解当前使用的选项可能是一项艰巨的任务，但有时非常重要。希望这些指南和提示能够帮助你。
