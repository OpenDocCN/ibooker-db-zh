# 第二十二章：服务器管理

# 22.0 简介

本章涵盖了如何执行涉及管理 MySQL 服务器的操作：

+   通用服务器配置

+   插件接口

+   控制服务器日志记录

+   配置存储引擎

本章不涵盖管理 MySQL 用户帐户。这是一个管理任务，在第二十四章中有详细介绍。

###### 注意

这里展示的许多技术需要管理员访问权限，例如修改 `mysql` 系统数据库中的表或使用需要 `SUPER` 特权的语句。因此，为了执行这里描述的操作，您可能需要以 `root` 而不是 `cbuser` 身份连接到服务器。

# 22.1 配置服务器

## 问题

您希望更改服务器设置，并验证您的更改是否生效。

## 解决方案

要更改设置，请在服务器启动时或运行时指定它们。要验证更改，请在运行时检查相关的系统变量。

## 讨论

MySQL 服务器提供了许多配置参数供您控制。例如，需要内存资源可以调整为更高或更低以调整资源使用。一个使用频繁的服务器需要更多的内存；一个使用较少的服务器需要更少的内存。您可以在服务器启动时设置命令选项和系统变量，并且许多系统变量也可以在运行时设置。您还可以在运行时检查您的设置，以验证配置是否如您所愿。

### 在服务器启动时的配置控制

要在启动时配置服务器，请在命令行或选项文件中指定选项。通常情况下，后者更可取，因为您可以指定设置一次，并且它们将在每次启动时生效。（有关使用命令行选项和选项文件的背景，请参阅食谱 1.4。）

命令选项名称通常使用连字符，而系统变量名称使用下划线。但是，在启动时，服务器更宽松，可以识别使用连字符或下划线写的命令选项和系统变量。例如，在命令行或选项文件中，`sql_mode` 和 `sql-mode` 是等效的。这与运行时不同，运行时对系统变量的引用*必须*使用下划线。

要在选项文件中指定服务器参数，请在服务器读取的文件的`[mysqld]`组中列出它们。例如，这里是您可能设置的一些参数：

+   默认字符集为 `utf8mb4，从 MySQL 8.0 开始`。此字符集默认使用 `utf8mb4_0900_ai_ci` 作为默认排序规则。

+   默认的 SQL 模式是 `STRICT_TRANS_TABLES`（MySQL 5.7 之后）。要默认更宽松，请删除严格的 SQL 模式，这不推荐。

+   在 MySQL 8.0 之后，默认启用事件调度程序。如果计划使用定期事件（参见食谱 11.5），必须在之前的版本上启用它。

+   对于 InnoDB 引擎，缓冲池大小默认为 128MB，在开发和测试之外是不足够的。考虑增加到足够运行数据集的内存大小。

+   时区设置为系统默认，除非在启动时指定。如果不打算使用系统时区，则需要通过设置 `--timezone=timezone_name` 在启动时设置它。

要实现这些配置想法，像这样在你的选项文件中写入 `[mysqld]` 组：

```
[mysqld]
character_set_server=utf8mb4
sql_mode=STRICT_TRANS_TABLES
event_scheduler=1
innodb_buffer_pool_size=512M
```

这些只是建议；根据自己的需求调整服务器配置。特别是有关插件和日志选项的信息，请参阅 Recipe 22.2 和 Recipe 23.0。

### 在运行时进行配置控制和验证

服务器启动后，您可以通过使用 `SET` 语句更改系统变量来进行运行时调整：

```
SET GLOBAL *`var_name`* = *`value`*;
```

这个语句设置了 *`var_name`* 的全局值；也就是说，默认情况下适用于所有客户端。在运行时更改全局值需要 `SUPER` 权限。许多系统变量还有会话值，这是特定客户端会话的值。给定变量的会话值在客户端连接时从全局值初始化，但之后客户端可以更改它。例如，DBA 可能在服务器启动时设置最大连接数：

```
[mysqld]
max_connections=1000
```

这设置了全局值。具有 `SUPER` 权限的 DBA 可以在运行时更改全局值：

```
SET GLOBAL max_connections = 1000;
```

后续连接的每个客户端的会话变量初始化为相同的值，但可以根据需要更改值。DBA 可能会增加此值以解决连接问题。

```
SET SESSION max_connections = 1000;
```

包含没有 `GLOBAL` 或 `SESSION` 修饰符的 `SET` 语句将更改会话值（如果有）。

在 MySQL 8.0 之后，您可以设置和持久化全局变量。许多全局变量是动态的，可以在运行时设置。使用 *PERSIST* 选项可以在不保存到配置文件的情况下永久设置此值，即使服务器重新启动也不会丢失。

```
SET PERSISTS max_connections = 1000;
```

```
SET PERSISTS_ONLY max_connections = 1000;
```

```
mysql> `SELECT @@GLOBAL.max_connections;`
+--------------------------+
| @@GLOBAL.max_connections |
+--------------------------+
|                     1000 |
+--------------------------+
```

要重置持久化的值，请使用：

```
RESET PERSIST;
```

```
RESET PERSIST max_connections;
```

还有另一种写系统变量引用的语法：

```
SET @@GLOBAL.*`var_name`* = *`value`*;
SET @@SESSION.*`var_name`* = *`value`*;
```

`@@` 语法更加灵活。除了 `SET` 语句之外，它还可以用于其他语句，使您能够检索或检查单个系统变量：

```
mysql> `SELECT @@GLOBAL.max_connections;`
+--------------------------+
| @@GLOBAL.max_connections |
+--------------------------+
|                     1000 |
+--------------------------+
```

使用 `@@` 语法引用系统变量时，如果没有 `GLOBAL.` 或 `SESSION.` 修饰符，将访问会话值（如果有），否则将访问全局值。

访问系统变量的其他方法包括 `SHOW VARIABLES` 语句以及从 `INFORMATION_SCHEMA` 的 `GLOBAL_VARIABLES` 和 `SESSION_VARIABLES` 表中选择。

如果一个设置仅作为命令选项存在，没有对应的系统变量，你无法在运行时检查其值。幸运的是，这种选项很少见。如今，大多数新的设置都被创建为可以在运行时检查的系统变量。

# 22.2 管理插件接口

## 问题

您希望利用某些服务器插件提供的功能。

## 解决方案

学习如何控制插件接口。

## 讨论

MySQL 支持使用扩展服务器功能的插件。有些插件实现存储引擎、身份验证方法、密码策略、`PERFORMANCE_SCHEMA`表等。服务器允许您指定要使用的插件，因此您可以只加载希望使用的插件，对于不需要的插件不会产生内存或处理开销。

本节介绍控制服务器加载哪些插件的一般背景。其他地方的讨论描述了特定插件及其对您的用处，包括身份验证插件（参见 Recipe 24.1），以及`validate_password`（参见 Recipe 24.3 和 Recipe 24.4）。

此处的示例使用`.so`（<q>共享对象</q>）文件名后缀引用插件文件。如果您的系统上后缀不同，请相应地调整名称（例如，在 Windows 上使用`.dll`）。如果您不知道给定插件文件的名称，请查看由`plugin_dir`系统变量命名的目录，服务器期望在此目录中找到插件文件。例如：

```
mysql> `SELECT @@plugin_dir;`
+------------------------------+
| @@plugin_dir                 |
+------------------------------+
| /usr/local/mysql/lib/plugin/ |
+------------------------------+
```

要查看已安装的插件，请使用`SHOW` `PLUGINS`或查询`INFORMATION_SCHEMA` `PLUGINS`表。

###### 注意

一些插件是内置的，无需显式启用，并且无法禁用。`mysql_native_password`和`sha256_password`身份验证插件属于此类别。

### 服务器启动时的插件控制

为了仅在给定服务器调用期间安装插件，请在服务器启动时使用`--plugin-load-add`选项，指定包含插件的文件名。如果选项值命名多个插件，请用分号分隔它们。或者，多次使用该选项，每次指定一个插件。这样可以通过使用`#`字符有选择性地注释相应的行来轻松启用或禁用单个插件：

```
[mysqld]
plugin-load-add=caching_sha2_password.so
plugin-load-add=adt_null.so
#plugin-load-add=semisync_master.so
#plugin-load-add=semisync_slave.so
```

`--plugin-load-add`选项在 MySQL 5.6 中引入。在 MySQL 8.0 中，您可以使用单个`--plugin-load`选项，该选项列出要加载的所有插件的分号分隔列表：

```
[mysqld]
plugin-load=validate_password.so;caching_sha2_password.so
```

显然，对于处理多个插件，使用`--plugin-load-add`更便于管理。

### 运行时的插件控制

要在运行时安装插件并使其持久化，请使用`INSTALL` `PLUGIN`。服务器加载插件（立即可用），并在`mysql.plugin`系统表中注册它，以使其在后续重启时自动加载。例如：

```
INSTALL PLUGIN caching_sha2_password SONAME 'caching_sha2_password.so';
```

`SONAME`（<q>共享对象名称</q>）子句指定包含插件的文件。

要在运行时禁用插件，请使用`UNINSTALL` `PLUGIN`。服务器卸载插件并从`mysql.plugin`表中删除其注册：

```
UNINSTALL PLUGIN caching_sha2_password;
```

`INSTALL` `PLUGIN`和`UNINSTALL` `PLUGIN`需要分别对`mysql.plugin`表具有`INSERT`和`DELETE`权限。

# 22.3 控制服务器日志记录

## 问题

您希望利用服务器提供的日志信息。

## 解决方案

了解控制日志记录的服务器选项。

## 讨论

MySQL 服务器可以生成多个日志：

错误日志

错误日志包含服务器遇到的问题或异常条件的信息。这是调试的有用信息。特别是，如果服务器退出，请检查错误日志查找原因。例如，如果在启动后立即发生退出，则可能是服务器选项文件中某些设置拼写错误或设置为无效值。错误日志将包含相关消息。

一般查询日志

一般查询日志显示每个客户端连接和断开连接的时间，以及它执行的 SQL 语句。这告诉您每个客户端参与的活动量和活动内容。

慢查询日志

慢查询日志记录执行时间较长的语句（请参阅[MySQL 参考手册](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html)了解“较长时间”的含义，因为它可能受到几个选项的影响）。在此日志中重复出现的查询可能是值得调查的瓶颈，以查看是否可以使其更有效率。

二进制日志

二进制日志包含服务器进行的数据更改记录。要设置复制，必须在源服务器上启用二进制日志：它作为将更改发送到副本服务器的存储介质。在数据恢复操作期间，二进制日志与备份文件一起使用。

每个日志都有不同的用途，大多数可以根据需要打开，以满足您的管理需求。每个日志可以写入文件，有些可以写入其他目的地。错误日志可以发送到您的终端或`syslog`设施。一般和慢查询日志可以写入文件，也可以写入`mysql`数据库中的表，或者两者兼而有之。

要控制服务器日志记录，请在服务器选项文件中添加指定所需日志类型的行（有些设置也可以在运行时更改）。例如，服务器选项文件中的以下行将错误日志发送到数据目录中的*err.log*文件，启用将一般查询和慢查询日志写入`mysql`数据库中的表，并启用将二进制日志写入*/var/mysql-logs*目录，文件名以*binlog*开头：

```
[mysqld]
log_error=err.log
log_output=TABLE
general_log=1
slow_query_log=1
log-bin=/var/mysql-logs/binlog
```

对于生成日志输出文件的选项中的文件名，除非使用完整路径名指定，否则日志文件将写入数据目录下。使用完整路径名的通常原因是将日志文件写入与包含数据目录的文件系统不同的文件系统，这对于在物理设备之间分割磁盘空间使用和 I/O 活动是一种有用的技术。

本节的其余部分提供了控制各个日志的具体细节。示例显示了在服务器选项文件中包含的行，以生成特定的日志记录行为。关于使用日志进行诊断或活动评估的一些想法，请参见 Recipe 22.6。

###### 警告

对于启用的任何日志，请参见 Recipe 22.4 和 Recipe 22.5 获取日志维护技术。随着时间的推移，日志大小会增加，因此您需要有一个管理计划。

### 错误日志

错误日志无法禁用，但您可以控制其写入位置。默认情况下，在 Unix 上，错误输出转到您的终端，或者如果使用*mysqld_safe*启动服务器，则在数据目录中为*`host_name`**.err*。在 Windows 上，默认为数据目录中的*`host_name`**.err*。要指定错误日志文件名，请设置`log_error`系统变量。

示例：

+   将错误日志写入数据目录中的*err.log*文件：

    ```
    [mysqld]
    log_error=err.log
    ```

+   自 MySQL 5.7.2 起，您可以通过设置`log_error_verbosity`系统变量来影响错误日志输出的数量。允许的值范围从 1（仅错误）到 3（错误、警告、注释；默认）。要仅查看错误，请执行以下操作：

    ```
    [mysqld]
    log_error=err.log
    log_error_verbosity=1
    ```

+   在 Unix 上，如果使用*mysqld_safe*启动服务器，则可以将错误日志重定向到`syslog`设施：

    ```
    [mysqld_safe]
    syslog
    ```

### 一般查询和慢查询日志

几个系统变量控制一般查询和慢查询日志。每个变量可以在服务器启动时设置或在运行时更改：

+   `log_output`控制日志目标。其值为`FILE`（日志写入文件，默认）、`TABLE`（日志写入表）、`NONE`（禁用日志记录），或者以逗号分隔的值的组合，顺序任意。如果值为`NONE`，则其他这些日志的设置无效。目标控制适用于一般查询和慢查询日志；您不能将一个写入文件，另一个写入表。

+   `general_log`和`slow_query_log`启用或禁用各自的日志。默认情况下，每个日志都被禁用。如果启用其中任何一个，则服务器将日志写入由`log_output`指定的目标，除非该变量为`NONE`。

+   `general_log_file`和`slow_query_log_file`指定日志文件名。默认名称为*`host_name`**.log*和*`host_name`**-slow.log*；但是，除非`log_output`指定`FILE`记录，否则这些设置无效。

示例：

+   将一般查询日志写入数据目录中的*query.log*文件：

    ```
    [mysqld]
    log_output=FILE
    general_log=1
    general_log_file=query.log
    ```

+   将一般查询和慢查询日志写入`mysql`数据库中的表（表名为`general_log`和`slow_log`，且不能更改）：

    ```
    [mysqld]
    log_output=TABLE
    general_log=1
    slow_query_log=1
    ```

+   将一般查询日志写入名为*query.log*的文件和`general_log`表：

    ```
    [mysqld]
    log_output=FILE,TABLE
    general_log=1
    general_log_file=query.log
    ```

### 二进制日志

在 MySQL 8 之前，默认情况下禁用了二进制日志记录。要启用二进制日志，请使用`--log-bin`选项，可选择性地指定日志文件的基本名称作为选项值。在 MySQL 8.0 中，要禁用二进制日志记录，可以使用`--skip-log-bin`选项或`--disable-log-bin`选项。默认基本名称是*`binlog`*。此选项的值为基本名称，因为服务器会自动为二进制日志文件添加编号顺序后缀，例如*.000001*，*.000002*等。服务器在启动时、刷新日志时以及当前文件达到最大日志文件大小（由`max_binlog_size`系统变量控制）时，会切换到序列中的下一个文件。在 MySQL 8.0 中，`expire_logs_days`已被弃用，并替换为`binlog_expire_logs_seconds`。要让服务器为您过期日志文件，请将`binlog_expire_logs_seconds`系统变量设置为文件变为可删除状态的秒数。*binlog_expire_logs_seconds*的默认值为 30 天（30*24*60*60 秒）。要禁用二进制日志的自动清除，请将*binlog_expire_logs_seconds*设置为 0。

例子：

+   启用二进制日志，在数据目录中写入以*binlog*开头的编号文件。此外，一周后过期日志文件：

    ```
    [mysqld]
    max_binlog_size=4G
    binlog_expire_logs_seconds=604800
    ```

二进制日志是 MySQL 服务器的重要组件，管理员需要小心处理。二进制日志包含所有数据更改事件，因此用于以下领域。

+   复制设置

+   时间点恢复

+   调试特定事件

# 22.4 旋转或过期日志文件

## 问题

除非管理，否则用于日志记录的文件会无限增长。

## 解决方案

管理日志文件的可用策略包括通过一组名称旋转日志文件和按年龄删除文件。但不同的策略适用于不同类型的日志，因此在选择策略之前请考虑日志类型。

## 讨论

日志文件旋转是一种通过一系列一个或多个名称重命名日志文件的技术。这保留文件一定数量的旋转，到达序列末尾时，文件内容通过被覆盖来丢弃。旋转可应用于错误日志、常规查询日志或慢查询日志。

日志文件到期会在文件达到一定年龄时移除文件。此技术适用于二进制日志。

两种日志管理方法都依赖于日志刷新，以确保当前日志文件已正确关闭。刷新日志时，服务器关闭并重新打开正在写入的文件。如果首先重命名错误、常规查询或慢查询日志文件，服务器将关闭当前文件并使用原始名称重新打开一个新文件；这就是在服务器运行时启用当前文件旋转的方式。服务器还会关闭当前二进制日志文件，并打开序列中下一个编号的新文件。

要刷新服务器日志，请执行`FLUSH` `LOGS`语句或使用*mysqladmin* `flush-logs`命令。（日志刷新需要`RELOAD`权限。）以下讨论显示了在命令行执行的维护操作，因此使用了*mysqladmin*。示例中使用*mv*作为文件重命名命令，在 Unix 上适用。在 Windows 上，请改用*rename*。

### 旋转错误日志、通用查询日志或慢查询日志

要在日志轮换中保持单个文件，请重命名当前的日志文件并刷新日志。假设错误日志文件在数据目录中命名为*err.log*。要进行轮换，请切换到数据目录，然后执行以下命令：

```
$ `mv err.log err.log.old`
$ `mysqladmin flush-logs`
```

当你刷新日志时，服务器会打开一个新的*err.log*文件。您可以随意删除*err.log.old*。要保留存档副本，请在删除它之前将其包含在文件系统备份中。

要维护一组多个旋转文件，可以使用一系列编号后缀很方便。例如，要维护一组三个旧的通用查询日志文件，请执行以下操作：

```
$ `mv query.log.2 query.log.3`
$ `mv query.log.1 query.log.2`
$ `mv query.log query.log.1`
$ `mysqladmin flush-logs`
```

初次执行命令序列时，只有在相应的*query.log.**`N`*文件存在时才不需要初始命令。

连续执行该命令序列将通过名称*query.log.1*、*query.log.2*和*query.log.3*轮换*query.log*；然后*query.log.3*将被覆盖并且其内容丢失。要保留存档副本，请在删除它们之前将轮换的文件包括在文件系统备份中。

### 旋转二进制日志

服务器按编号顺序创建二进制日志文件。要使其过期，只需安排在文件够旧时删除它们。几个因素影响服务器创建和维护的文件数量：

+   服务器重启和日志刷新操作的频率：每次发生其中一种情况时都会生成一个新文件。

+   文件可以增长到的大小：较大的大小导致较少的文件。要控制此大小，请设置`max_binlog_size`系统变量。

+   允许旧文件变得多老：较长的过期时间导致更多的文件。要控制此年龄，请设置`binlog_expire_logs_seconds`系统变量。服务器在启动时和打开新的二进制日志文件时进行过期检查。

以下设置启用二进制日志，设置最大文件大小为 4GB，并在四天后过期文件：

```
[mysqld]
log-bin=binlog
max_binlog_size=4G
binlog_expire_logs_seconds=4
```

你还可以通过`PURGE` `BINARY` `LOGS`语句手动删除二进制日志文件。例如，要删除直到包括名为`binlog.001028`的文件为止的所有文件，请执行以下操作：

```
PURGE BINARY LOGS TO 'binlog.001028';
```

如果你的服务器是复制源，请不要过于急于删除二进制日志文件。在确定其内容已完全传输到所有副本之前，不应删除任何文件。

### 自动化日志文件轮换

为了更容易执行旋转操作，将实施操作的命令放入文件以创建一个 shell 脚本。要自动执行旋转操作，请安排在作业调度程序（如 *cron*）中执行该脚本。该脚本将需要访问连接参数，使其能够连接到服务器以刷新日志，使用具有 `RELOAD` 特权的帐户。一个策略是将参数放入选项文件，并通过 `--defaults-file=`*`file_name`* 选项将文件传递给 *mysqladmin*。例如：

```
#!/bin/sh
mv err.log err.log.old
mysqladmin --defaults-file=/usr/local/mysql/data/flush-opts.cnf flush-logs
```

# 22.5 旋转日志表或过期日志表行

## 问题

用于日志记录的表会无限增长，除非进行管理。

## 解决方案

旋转表或在其中过期行。

## 讨论

Recipe 22.4 讨论了日志文件的轮换和过期。类似的技术也适用于日志表：

+   要旋转日志表，请重命名它并打开一个新表，以使用原始名称。

+   要过期日志表中的内容，请删除超过一定年龄的行。

这里的示例演示了如何使用通用查询日志表 `mysql.general_log` 实施这些方法。相同的方法适用于慢查询日志表 `mysql.slow_log` 或任何包含具有时间戳的行的其他表。

要使用日志表轮换，创建原始表的空副本作为新表（参见 Recipe 6.1），然后重命名原始表并重命名新表以取代原始表：

```
DROP TABLE IF EXISTS mysql.general_log_old, mysql.general_log_new;
CREATE TABLE mysql.general_log_new LIKE mysql.general_log;
RENAME TABLE mysql.general_log TO mysql.general_log_old,
  mysql.general_log_new TO mysql.general_log;
```

要使用日志行过期，可以完全清空表或选择性地清空：

+   要完全清空日志表，请将其截断：

    ```
    TRUNCATE TABLE mysql.general_log;
    ```

+   要选择性地过期表，仅删除早于给定年龄的行，必须知道指示行创建时间的列名：

    ```
    DELETE FROM mysql.general_log WHERE event_time < NOW() - INTERVAL 1 WEEK;
    ```

对于自动过期，可以在计划事件中执行上述任何技术的语句（参见 Recipe 11.5）。例如：

```
CREATE EVENT expire_general_log
  ON SCHEDULE EVERY 1 WEEK
  DO DELETE FROM mysql.general_log
     WHERE event_time < NOW() - INTERVAL 1 WEEK;
```

# 22.6 配置存储引擎

## 问题

您希望确保所选择的引擎已正确配置。

## 解决方案

根据用例理解和配置每个存储引擎。

## 讨论

MySQL 默认提供几种存储引擎，例如 MyISAM 和 InnoDB。从 MySQL 8.0 开始，InnoDB 作为默认数据库引擎。除了这种流行的存储引擎，您可能还想探索其他一些。每个存储引擎都将使用来自操作系统的共享资源以及专用资源。在混合使用时，必须注意不要提供太多资源。

+   InnoDB：支持事务和行级锁定，具有完全的 ACID 兼容性引擎。

+   MyISAM：表级锁定和简单引擎。

+   MyRocks：基于 LSM 的 B 树键/值存储引擎。^(1)

+   CSV：逗号分隔值引擎。

+   Blackhole：所有写入都发送到 /dev/null 无数据存储引擎。

+   内存：为内存工作负载存储引擎优化。

+   Archive：用于存档数据的只写引擎，以压缩格式存储引擎。

###### 警告

同时使用多个存储引擎可能会引发问题，并可能导致数据丢失，如果在同一事务中使用时请注意应用程序及其周围工具的兼容性。

由于每个引擎都以不同方式存储数据，我们必须相应地配置它们。InnoDB 使用重做日志和撤销日志空间来存储修改后的数据。这允许在硬件或服务器故障时最小化数据丢失的情况下进行恢复和时间点还原。MyRocks 是另一种先进的存储引擎，首先写入恢复日志 Write Ahead Log (WAL)，并支持每个事务的回滚。MyISAM 和 CSV 类型存储引擎直接写入数据文件。虽然二进制备份和传输更容易，但这些引擎不支持回滚操作。

要在 MySQL 8.0 中检查默认存储引擎：

```
mysql> `SELECT @@default_storage_engine;`
+--------------------------+
| @@default_storage_engine |
+--------------------------+
| InnoDB                   |
+--------------------------+
```

通过检查模式定义，我们可以看到表存储引擎类型。

```
mysql> `SHOW CREATE TABLE limbs\G`
       Table: limbs
Create Table: CREATE TABLE `limbs` (
  `thing` varchar(20) DEFAULT NULL,
  `legs` int DEFAULT NULL,
  `arms` int DEFAULT NULL,
  PRIMARY KEY(thing)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 
COLLATE=utf8mb4_0900_ai_ci
```

如果我们想在创建表后更改存储引擎类型，可以发出*ALTER*语句。

```
mysql> `ALTER TABLE cookbook.limbs  ENGINE=MYISAM;`
Query OK, 11 rows affected (0.16 sec)
Records: 11  Duplicates: 0  Warnings: 0

mysql> SHOW CREATE TABLE limbs\G
*************************** 1\. row ***************************
       Table: limbs
Create Table: CREATE TABLE `limbs` (
  `thing` varchar(20) DEFAULT NULL,
  `legs` int DEFAULT NULL,
  `arms` int DEFAULT NULL,
  PRIMARY KEY(thing)
) ENGINE=MyISAM DEFAULT CHARSET=utf8mb4 
COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)
```

###### 注意

虽然可以在创建表并加载数据后在存储引擎之间进行切换，*ALTER TABLE* 操作会为所有存储引擎锁定。对于大型数据集，请考虑使用在线模式更改工具，例如[pt-online-schema-change](https://www.percona.com/doc/percona-toolkit/3.0/pt-online-schema-change.html#:~:text=pt%2Donline%2Dschema%2Dchange%20works%20by%20creating%20an%20empty,it%20with%20the%20new%20one.) 或 [gh-ost](https://github.com/github/gh-ost)。这些工具允许模式迁移完成而不创建元数据锁，并以受控方式应用更改。

可以按以下方式检查其他存储引擎设置：

```
mysql> `SHOW GLOBAL VARIABLES LIKE "%engine%" ;`
+---------------------------------+---------------+
| Variable_name                   | Value         |
+---------------------------------+---------------+
| default_storage_engine          | InnoDB        |
| default_tmp_storage_engine      | InnoDB        |
| disabled_storage_engines        |               |
| internal_tmp_mem_storage_engine | TempTable     |
| secondary_engine_cost_threshold | 100000.000000 |
+---------------------------------+---------------+
```

^(1) 通过 Percona 和 MariaDB 设计的第三方存储引擎，旨在处理具有节省空间优势的写入密集型工作负载。
