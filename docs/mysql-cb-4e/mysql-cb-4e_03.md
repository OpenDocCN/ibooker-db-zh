# 第三章：MySQL 复制

# 3.0 介绍

MySQL 复制提供了一种设置活动（源）数据库的副本服务器（复制）的方式，然后自动持续更新此类副本，应用源服务器接收的所有更改。

副本在许多情况下都很有用，特别是：

热备份

在故障发生时，通常处于空闲状态的服务器可以替代活动服务器。

读取扩展

多台服务器从同一源复制时，可以比单台机器处理更多的并行读取请求。

地理分布

当应用程序为不同区域的用户提供服务时，位于本地的数据库服务器可以帮助更快地检索数据。

分析服务器

复杂的分析查询可能需要几小时才能运行，设置大量锁定并使用大量资源。在副本上运行它们可以最小化对应用程序其他部分的影响。

备份服务器

从活动数据库获取备份涉及高 IO 资源使用和锁定，这是必要的，以避免备份数据集与活动数据集之间的数据不一致性。从专用副本获取备份可以减少对生产环境的影响。

延迟副本

使用 `SOURCE_DELAY`（`MASTER_DELAY`）选项配置的延迟应用更新的副本允许回滚人为错误，例如重要表的删除。

###### 注意

在历史上，源服务器被称为主服务器，副本服务器被称为从服务器。最近发现，主从这些术语并不正确地反映了复制的工作方式，而这些术语本身可能是侮辱性的。在过去几年中，大多数软件供应商开始从旧术语转向新术语。对于 MySQL，这种变化从版本 8.0.22 开始，并仍在进行中。并非所有选项名称和命令都支持新语法。即使您的 MySQL 版本完全支持新语法，您在公共论坛和早期印刷书籍中可能会找到遗留术语。因此，在本书中讨论复制角色时，我们使用源和副本这些术语。对于支持新语法的命令和变量名称，我们首次提供新语法。如果变更仍在进行中，我们使用旧的语法。

MySQL 复制需要在两台服务器上进行特殊操作。

源服务器将所有更新存储在二进制日志文件中。这些文件包含编码的更新事件。源服务器在某一时刻写入单个二进制日志文件。一旦达到 `max_binlog_size`，则会旋转二进制日志，并创建一个新文件。

二进制日志文件支持两种格式：`STATEMENT` 和 `ROW`。在 `STATEMENT` 格式中，SQL 语句按原样编写，然后编码为二进制格式。在 `ROW` 格式中，SQL 语句不记录，而是存储实际的表行更新。首选 `ROW` 二进制日志格式。

###### 提示

当使用二进制日志格式 `ROW` 时，在排查复制错误时，知道源服务器收到的实际语句可能很有用。使用选项 `binlog_rows_query_log_events` 将信息记录事件与原始查询一起存储。这样的事件不参与复制，仅供信息目的检索。

副本服务器持续从源服务器请求二进制日志事件，然后将它们存储在特殊文件中，称为中继日志文件。它有一个独立的线程，称为 IO 或连接线程，专门负责此任务。另一个线程或线程组，称为 SQL 或应用程序线程，从中继日志中读取事件并将它们应用到表中。

二进制日志中的每个事件都有其自己的唯一标识符：位置。位置对每个文件都是唯一的，并在创建新文件时重置。副本可以使用二进制日志文件名和位置作为事件的唯一标识符。

虽然二进制日志位置唯一标识特定文件中的事件，但不能用于确定特定事件是否已在副本上应用。为解决此问题，引入了全局事务标识符 (GTID)。这些是唯一标识符，分配给每个事务。它们在 MySQL 安装的整个生命周期内都是唯一的。它们还使用机制唯一标识服务器，因此即使从多个源进行复制也是安全的。

副本在特殊存储库中存储有关源二进制日志坐标的信息，由变量 `master_info_repository` 定义。此类存储库可以存储在表中或文件中。

本章描述了如何设置和使用 MySQL 复制。它涵盖了所有典型的复制场景，包括：

+   两台服务器单向源-副本设置。

+   循环复制

+   多源复制

+   半同步复制

+   组复制

# 3.1 配置一源一副本之间的基本复制

## 问题

您希望为复制准备两台服务器。

## 解决方案

在源配置文件中添加配置选项 `log-bin`，为两台服务器指定唯一的 `server_id`，添加支持 GTID 和/或非默认二进制日志格式的选项，并在源上创建一个具有 `REPLICATION SLAVE` 权限的用户。

## 讨论

首先，您需要准备两台服务器以处理复制事件。

在源服务器上：

+   通过在配置文件中添加选项 `log-bin` 来启用二进制日志。更改此选项需要重新启动。自版本 8.0 起，默认情况下启用二进制日志。

+   设置唯一的 `server_id`。`server_id` 是动态变量，可以在不关闭服务器的情况下更改，但我们强烈建议在配置文件中设置它，这样在重新启动后不会被覆盖。

+   创建一个复制用户，并授予 `REPLICATION SLAVE` 权限：

    ```
    mysql> `CREATE USER repl@'%' IDENTIFIED BY 'replrepl';`
    Query OK, 0 rows affected (0,01 sec)

    mysql> `GRANT REPLICATION SLAVE ON *.* TO repl@'%';`
    Query OK, 0 rows affected (0,03 sec)
    ```

###### 警告

在 MySQL 8.0 中，默认的认证插件是 `caching_sha2_password`，它要求 TLS 连接或源公钥。因此，如果您想使用此插件，需要按照 Recipe 3.14 中描述的方法为副本启用 TLS 连接，或使用 *CHANGE REPLICATION SOURCE* (*CHANGE MASTER*) 命令的选项 `SOURCE_PUBLIC_KEY_PATH=1` (`GET_MASTER_PUBLIC_KEY=1`)。

或者您可以使用认证插件，允许不安全的连接。

```
mysql> `CREATE USER repl@'%' IDENTIFIED WITH mysql_native_password BY 'replrepl';`
Query OK, 0 rows affected (0,01 sec)

mysql> `GRANT REPLICATION SLAVE ON *.* TO repl@'%';`
Query OK, 0 rows affected (0,03 sec)
```

在副本上只需设置唯一的 `server_id`。

###### 提示

自版本 8.0 起，您可以使用 *SET PERSIST* 将动态更改的变量永久保存：

```
mysql> `SET PERSIST server_id=200;`
Query OK, 0 rows affected (0,01 sec)
```

详细信息请参阅 [MySQL 用户手册中的持久化系统变量](https://dev.mysql.com/doc/refman/8.0/en/persisted-system-variables.html)。

在此阶段，您可以调整其他选项，这些选项会影响复制的安全性和性能，特别是：

`binlog_format`

二进制日志格式

GTID 支持

支持全局事务标识符

`replica_parallel_type` (`slave_parallel_type`) 和 `replica_parallel_workers` (`slave_parallel_workers`)

多线程副本支持

副本上的二进制日志

定义副本是否以及如何使用二进制日志。

我们将在接下来的几个步骤中详细介绍这些选项。

# 3.2 在新安装环境中基于位置的复制

## 问题

您想要设置一个刚安装的 MySQL 服务器的副本，使用基于位置的配置。

## 解决方案

如 Recipe 3.1 中描述的准备源和副本服务器，然后在源服务器上使用 *SHOW MASTER STATUS* 命令获取当前二进制日志位置，并使用 *CHANGE REPLICATION SOURCE ... source_log_file='BINARY LOG FILE NAME', source_log_pos=POSITION;* (*CHANGE MASTER ... master_log_file='BINARY LOG FILE NAME', master_log_pos=POSITION;*) 命令将副本指向适当的位置。

## 讨论

对于此示例，我们假设您有两台刚安装的服务器，其中没有任何用户数据。任何服务器上都没有写活动。

首先，按照 Recipe 3.1 中的描述准备它们以供复制使用。然后，在源上运行 *SHOW MASTER STATUS* 命令：

```
mysql> `SHOW MASTER STATUS;`
+-------------------+----------+--------------+------------------+-------------------+
| File              | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+-------------------+----------+--------------+------------------+-------------------+
| master-bin.000001 |      156 |              |                  |                   |
+-------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

`File` 字段包含当前二进制日志的名称，而 `Position` 字段包含当前位置。记录这些字段的值。

在副本上运行 [*CHANGE REPLICATION SOURCE* (*CHANGE MASTER*)](https://dev.mysql.com/doc/refman/8.0/en/change-replication-source-to.html) 命令：

```
mysql> `CHANGE` `REPLICATION` `SOURCE` 
    -> `TO` `SOURCE_HOST``=``'sourcehost'``,`         `-- Host of the source server`
    -> `SOURCE_PORT``=``3306``,`                    `-- Port of the source server`
    -> `SOURCE_USER``=``'repl'``,`                  `-- Replication user`
    -> `SOURCE_PASSWORD``=``'replrepl'``,`          `-- Password`
    -> `SOURCE_LOG_FILE``=``'source-bin.000001'``,` `-- Binary log file`
    -> `SOURCE_LOG_POS``=``156``,`                  `-- Start position`
    -> `GET_SOURCE_PUBLIC_KEY``=``1``;`
Query OK, 0 rows affected, 1 warning (0.06 sec)
```

要启动副本，请使用命令 *START REPLICA* (*START SLAVE*)：

```
mysql> `START REPLICA;`
Query OK, 0 rows affected (0.01 sec)
```

要检查副本是否正在运行，请使用 *SHOW REPLICA STATUS* (*SHOW SLAVE STATUS*)：

```
mysql> `\P grep Running`
PAGER set to 'grep Running'
mysql> `SHOW REPLICA STATUS\G`
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
    Replica_SQL_Running_State: Slave has read all relay log;↩
                               waiting for more updates
1 row in set (0.00 sec)
```

上面的列表确认了 IO（连接）和 SQL（应用程序）副本线程都在运行，并且复制状态良好。我们将在 Recipe 3.15 中讨论 *SHOW REPLICA STATUS* 命令的完整输出。

现在您可以在源服务器上启用写入。

# 3.3 设置一个基于位置的 MySQL 安装的副本，该安装已在使用中

## 问题

设置新安装服务器的副本与未来源已经具有数据的情况不同。在后一种情况下，特别注意不要通过指定错误的起始位置引入数据不一致性。在本文中，我们提供了如何设置正在使用的 MySQL 安装的副本的详细说明。

## 解决方案

按照 Recipe 3.1 中描述的方法准备源服务器和副本服务器，停止源服务器上的所有写操作，进行备份，然后使用 *SHOW MASTER STATUS* 命令获取当前二进制日志位置，此位置将用于使用 *CHANGE REPLICATION SOURCE ... source_log_file='BINARY LOG FILE NAME', source_log_pos=POSITION;* 命令将副本指向适当位置。

## 讨论

与安装新副本的情况类似，两台服务器都需要按照 Recipe 3.1 中描述的方式配置复制使用。在启动设置之前，您需要确保两台服务器都具有唯一的 `server_id`，并且源服务器已启用二进制日志记录。您可以在此时创建复制用户，或者在设置副本之前执行此操作。

如果您有一个已经运行一段时间的服务器，并希望设置其副本，您需要首先进行备份，然后在副本上恢复，并将副本指向源服务器。此设置的挑战在于使用正确的二进制日志位置：如果服务器在备份运行时接受写入，则位置会一直变化。因此，*SHOW MASTER STATUS* 命令将返回错误的结果，除非您在进行备份时停止所有写操作。

标准备份工具在备份未来源服务器用于副本时支持特殊选项，以绕过此问题。

*mysqldump*，在 Recipe 6.6 中描述，具有选项 `--source-data` (`--master-data`)。如果设置为 1，在备份开始时将写入 *CHANGE REPLICATION SOURCE* 语句和坐标到结果转储文件中，并在加载转储时执行。

```
$ `mysqldump --host=127.0.0.1 --user=root \`
>  `--source-data=1 --all-databases > mydump.sql`
$ `grep -b5 "CHANGE REPLICATION SOURCE" -m1 mydump.sql`
906-
907---
910--- Position to start replication or point-in-time recovery from
974---
977-
978:CHANGE REPLICATION SOURCE TO SOURCE_LOG_FILE='source-bin.000002',↩
    SOURCE_LOG_POS=156;
1052-
1053---
1056--- Current Database: `mtr`
1083---
1086-
```

###### 提示

如果您希望在生成的转储文件中包含复制位置，但不希望自动执行 *CHANGE REPLICATION SOURCE* 命令，请将选项 `--source-data` 设置为 2：在这种情况下，该语句将作为注释写入。稍后可以手动执行它。

像 [Percona XtraBackup](https://www.percona.com/doc/percona-xtrabackup/8.0/index.html) 或 [MySQL Enterprise Backup](https://dev.mysql.com/doc/mysql-enterprise-backup/8.0/en/) 这样的在线二进制备份工具，在特殊的元数据文件中存储二进制日志坐标。请参考您的备份工具文档，了解如何安全地备份源服务器。

###### 提示

MySQL 有几种备份方式。执行在线备份的工具无需停止 MySQL 服务器。逻辑备份生成一组命令的文件，允许恢复数据。二进制备份复制物理数据库文件。与逻辑备份相比，二进制备份通常要快得多。与逻辑备份相比，二进制备份的恢复速度大大加快。

最简单和最快的二进制备份实用程序是*cp*，需要停止 MySQL 服务器。在线备份工具允许在服务器运行时复制二进制数据，适合大数据集的首选解决方案。

逻辑备份解决方案，对版本之间的差异更兼容，可以用来恢复数据。在需要迁移小部分数据时，如表格或表格的一部分，它们也非常方便。

一旦有备份，请在副本上恢复它。对于*mysqldump*，请使用*mysql*客户端加载转储：

```
$ `mysql < mydump.sql`
```

恢复备份后，使用*START REPLICA*命令启动复制。

# 3.4 设置基于 GTID 的复制

## 问题

您想使用全局事务标识符（GTIDs）设置副本。

## 解决方案

在源和副本配置文件中添加选项`gtid_mode=ON`和`enforce_gtid_consistency=ON`，然后使用*CHANGE REPLICATION SOURCE ... SOURCE_AUTO_POSITION=1*命令将副本指向源服务器。

## 讨论

基于位置的复制设置很容易，但容易出错。如果您混淆并指定未来的位置会怎样？在这种情况下，一些事务将被跳过。或者，如果您指定过去的位置会发生什么？在这种情况下，同一事务将被应用两次。您最终会得到重复的、丢失的或损坏的行。

为了解决这个问题，引入了全局事务标识符（GTIDs），用于唯一标识服务器上的每个事务。GTID 由两部分组成：首次执行此事务的服务器的唯一 ID 和此服务器上的事务唯一 ID。源服务器 ID 通常是`server_uuid`全局变量的值，事务 ID 是从 1 开始的数字。

```
mysql> `SHOW MASTER STATUS\G`
*************************** 1\. row ***************************
             File: binlog.000001
         Position: 358
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 467ccf91-0341-11eb-a2ae-0242dc638c6c:1
1 row in set (0.00 sec)

mysql> `select @@gtid_executed;`
+----------------------------------------+
| @@gtid_executed                        |
+----------------------------------------+
| 467ccf91-0341-11eb-a2ae-0242dc638c6c:1 |
+----------------------------------------+
1 row in set (0.00 sec)
```

服务器执行的事务存储在 GTID 集中，它们的 GTID 在*SHOW MASTER STATUS*输出中可见，以及`gtid_executed`变量的值。该集包含原始服务器的唯一 ID 和事务编号范围。

在下面的示例中，`467ccf91-0341-11eb-a2ae-0242dc638c6c`是源服务器的唯一标识，`1-299`是在此服务器上执行的事务编号范围。

```
mysql> `select @@gtid_executed;`
+--------------------------------------------+
| @@gtid_executed                            |
+--------------------------------------------+
| 467ccf91-0341-11eb-a2ae-0242dc638c6c:1-299 |
+--------------------------------------------+
1 row in set (0.00 sec)
```

GTID 集可以包含范围、单个事务和以冒号分隔的事务组。具有不同源 ID 的 GTID 由逗号分隔：

```
mysql> `select @@gtid_executed\G`
*************************** 1\. row ***************************
@@gtid_executed: 000bbf91-0341-11eb-a2ae-0242dc638c6c:1,
467ccf91-0341-11eb-a2ae-0242dc638c6c:1-310:400
1 row in set (0.00 sec)
```

通常情况下，GTIDs 会自动分配，您不需要关注它们的值。

然而，要使用 GTIDs，您需要为您的服务器添加额外的准备步骤。

启用 GTID 需要两个配置选项：`gtid_mode=ON`和`enforce-gtid-consistency=ON`。在启动复制之前，这两个选项必须在两个服务器上启用。

如果您正在设置一个新的副本，源端启用了 GTID，则只需将这些选项添加到配置文件中并重新启动服务器即可。完成后，您可以使用*CHANGE REPLICATION SOURCE ... SOURCE_AUTO_POSITION=1*命令启用复制并启动它：

```
mysql> `CHANGE REPLICATION SOURCE TO`
    -> `SOURCE_HOST='sourcehost',    -- Host of the source server`
    -> `SOURCE_PORT=3306,            -- Port of the source server`
    -> `SOURCE_USER='repl',          -- Replication user`
    -> `SOURCE_PASSWORD='replrepl',  -- Password`
    -> `GET_SOURCE_PUBLIC_KEY=1,`
    -> `SOURCE_AUTO_POSITION=1;`
Query OK, 0 rows affected, 1 warning (0.06 sec)

mysql> `START REPLICA;`
Query OK, 0 rows affected (0.01 sec)
```

但是，如果复制已经使用基于位置的设置运行，则需要执行其他步骤：

1.  停止所有更新，使两个服务器都变为只读状态：

    ```
    mysql> `SET GLOBAL super_read_only=1;`
    Query OK, 0 rows affected (0.01 sec)
    ```

1.  等待副本从源服务器上的所有更新追赶上来：源服务器上*SHOW MASTER STATUS*输出的`File`和`Position`值应与副本上*SHOW REPLICA STATUS*中的`Relay_Source_Log_File`和`Exec_Source_Log_Pos`值匹配。

    ###### 警告

    不要依赖于`Seconds_Behind_Source`值，因为它不准确。

    例如，在源服务器上的以下输出中：

    ```
    mysql> `SHOW` `MASTER` `STATUS``\``G`
    *************************** 1. row ***************************
                 File: master-bin.000001
             Position: 9614
         Binlog_Do_DB: 
     Binlog_Ignore_DB: 
    Executed_Gtid_Set: 
    1 row in set (0.00 sec)
    ```

    二进制日志位置为 7090。

    ```
    mysql> `\``P` `grep` `-``E` `"Source_Log_Pos|Seconds_Behind_Source"`
    PAGER set to 'grep -E "Source_Log_Pos|Seconds_Behind_Source"'
    mysql> `SHOW` `REPLICA` `STATUS``\``G`
              Read_Source_Log_Pos: 9614
              Exec_Source_Log_Pos: 7308
            Seconds_Behind_Source: 0
    1 row in set (0.00 sec)
    ```

    在副本上，IO 线程读取的`Read_Source_Log_Pos`位置与源服务器上的值相同，而最新执行事件`Exec_Source_Log_Pos`的位置为 7308：在二进制日志文件中稍早的位置。 `Seconds_Behind_Source`的值为 0 是正常的，因为 MySQL 服务器可以每秒执行数千次更新。但这并不意味着副本完全追赶上了源服务器。

1.  一旦副本追赶上，停止两个服务器，启用`gtid_mode=ON`和`enforce-gtid-consistency=ON`选项，然后启动它们并启用复制：

    ```
    mysql> `CHANGE REPLICATION SOURCE TO` 
        -> `SOURCE_HOST='sourcehost',    -- Host of the source server`
        -> `SOURCE_PORT=3306,            -- Port of the source server`
        -> `SOURCE_USER='repl',          -- Replication user`
        -> `SOURCE_PASSWORD='replrepl',  -- Password`
        -> `GET_SOURCE_PUBLIC_KEY=1,`
        -> `SOURCE_AUTO_POSITION=1;`
    Query OK, 0 rows affected, 1 warning (0.06 sec)

    mysql> `START REPLICA;`
    Query OK, 0 rows affected (0.01 sec)
    ```

###### 提示

如果在开始从基于位置到基于 GTID 的复制切换之前，副本已知复制源连接选项，则可以省略它们。

###### 注意

您无需在复制中启用二进制日志记录以使用 GTID。但如果您要在复制外写入副本，则其事务将不具有自己的 GTID 分配。 GTID 仅用于复制事件。

## 参见

关于如何设置带 GTID 的 MySQL 复制的更多信息，请参见[MySQL 用户参考手册](https://dev.mysql.com/doc/refman/8.0/en/replication-gtids-howto.html)。

# 3.5 配置二进制日志格式

## 问题

您希望使用最适合您应用程序的二进制日志格式。

## 解决方案

决定哪种格式最适合您的需求，并使用配置选项`binlog_format`进行设置。

## 讨论

默认的 MySQL 二进制日志格式自版本 5.7.7 起为`ROW`。这是最安全的格式，适合大多数应用程序。它存储由二进制日志事件修改的编码表行。

然而，`ROW`格式的二进制日志可能会产生比`STATEMENT`格式更多的磁盘和网络流量。这是因为它将修改后的行存储到二进制日志文件中两次：变更前和变更后。如果一个表有很多列，即使只修改了一列，所有列的值也会被记录两次。

如果希望二进制日志仅存储修改的列和用于唯一标识修改行的列（通常是主键），可以使用配置选项`binlog_row_image=minimal`。这在源服务器和其复制件的表完全相同的情况下可以完美工作，但如果列数、数据类型或主键定义不匹配可能会导致问题。

要存储完整行，除非语句未更改`TEXT`或`BLOB`列并且不需要唯一标识修改行，请使用选项`binlog_row_image=noblob`。

如果行格式仍然产生过多流量，可以将其切换到`STATEMENT`。在这种情况下，修改行的语句将被记录，然后由复制件执行。要使用二进制日志格式`STATEMENT`，请设置选项`binlog_format=STATEMENT`。

不推荐使用`STATEMENT`格式是因为某些语句可能会在不同的服务器上产生不同的更新，即使数据最初是相同的。这些语句被称为非确定性的。为了解决这个缺点，MySQL 有一个特殊的二进制日志格式：`MIXED`，它通常以`STATEMENT`格式记录事件，并在语句是非确定性的时候自动切换到`ROW`格式。

###### 警告

如果在复制件上启用了二进制日志，它应该使用与源服务器相同的二进制日志格式或者`MIXED`，除非你使用选项`log_replica_updates=OFF`（`log_slave_updates=OFF`）禁用了复制事件的二进制日志记录。这是必须的，因为复制件不会转换二进制日志格式，而只是将接收到的事件复制到自己的二进制日志文件中。如果格式不匹配，复制过程将因错误而停止。

可以动态地在全局或会话级别更改二进制日志格式。要在全局级别更改格式，请运行：

```
mysql> `set global binlog_format='statement';`
Query OK, 0 rows affected (0,00 sec)
```

要在全局级别更改格式并永久存储，请使用：

```
mysql> `set persist binlog_format='row';`
Query OK, 0 rows affected (0,00 sec)
```

请注意，这不会更改现有连接的二进制日志记录格式。要在会话级别更改格式，请执行：

```
mysql> `set session binlog_format='mixed';`
Query OK, 0 rows affected (0,00 sec)
```

尽管`STATEMENT`格式通常产生的流量比`ROW`少，但并非总是如此。例如，具有长`WHERE`或`IN`子句的复杂语句，仅修改少数行，使用`STATEMENT`格式会生成更大的二进制日志事件。

另一个使用`STATEMENT`格式的问题是，复制件会像源服务器上运行的方式一样执行接收到的事件。因此，如果一个语句在原地无效，复制件上也会运行缓慢。例如，对于那些含有`WHERE`子句并且无法使用索引解析的大型表，通常会很慢。在这种情况下，切换到`ROW`格式可能会提高性能。

###### 警告

通常，`ROW`事件使用主键在副本上查找需要更新的行。如果表没有主键，`ROW`格式可能工作非常缓慢。旧版的 MySQL 甚至可能由于已修复的错误而更新错误的行。InnoDB 存储引擎使用的自动生成的主键在这里无济于事，因为它可能会为源服务器和副本服务器上的同一行生成不同的值。因此，在使用二进制日志格式`ROW`时，定义表的主键是强制性的。

# 3.6 使用复制过滤器

## 问题

您希望仅复制特定数据库或表的事件。

## 解决方案

在源、副本或两者上使用复制过滤器。

## 讨论

MySQL 可以过滤特定数据库或表的更新。您可以在源服务器上设置这些过滤器，以防止它们记录在二进制日志中，或者在副本服务器上，以便复制不会执行它们。

### 在源服务器上进行过滤

###### 警告

如果错误设置，复制过滤器可能导致数据丢失。非常仔细地研究这个配方，并且在将其部署到生产环境之前，始终测试它们在您的设置中的工作方式。

要仅记录对特定数据库的更新，请使用配置选项`binlog-do-db=db_name`。对于此选项，没有相应的变量，因此更改二进制日志过滤器需要重新启动。要记录对两个或更多特定数据库的更新，请多次指定`binlog-do-db`选项：

```
[mysqld]
binlog-do-db=cookbook
binlog-do-db=test
```

`ROW`和`STATEMENT`二进制日志格式的二进制日志过滤器行为不同。对于基于语句的日志记录，只考虑默认数据库。如果您使用全限定表名，例如`mydatabase.mytable`，它们将基于默认数据库值而不是更新的数据库部分进行记录。

因此，对于上述配置文件片段，以下三个更新将在二进制日志中记录：

+   ```
    $ `mysql cookbook`
    mysql> `INSERT INTO limbs (thing, legs, arms) VALUES('horse', 4, 0);`
    Query OK, 1 row affected (0,01 sec)
    ```

+   ```
    mysql> `USE cookbook`
    Database changed
    mysql> `DELETE FROM limbs WHERE thing='horse';`
    Query OK, 1 row affected (0,00 sec)
    ```

+   ```
    mysql> `USE cookbook`
    Database changed
    mysql> `INSERT INTO donotlog.onlylocal (mysecret)      -> values('I do not want to replicate it!');`
    Query OK, 1 row affected (0,01 sec)
    ```

然而，对于食谱数据库的此更新将不会被记录：

```
mysql> `use donotlog`
Database changed
mysql> `UPDATE cookbook.limbs set arms=8 WHERE thing='squid';`
Query OK, 1 row affected (0,01 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

当使用二进制日志格式`ROW`时，将忽略默认数据库以获取完全合格的表名。因此，所有这些更新都将被记录：

```
$ `mysql cookbook`
mysql> `INSERT INTO limbs (thing, legs, arms) VALUES('horse', 4, 0);`
Query OK, 1 row affected (0,01 sec)
mysql> `USE cookbook`
Database changed
mysql> `DELETE FROM limbs WHERE thing='horse';`
Query OK, 1 row affected (0,00 sec)
mysql> `USE donotlog`
Database changed
mysql> `UPDATE cookbook.limbs SET arms=10 WHERE thing='squid';`
Query OK, 1 row affected (0,01 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

然而，此语句将不会被记录：

```
mysql> `USE cookbook`
Database changed
mysql> `INSERT INTO donotlog.onlylocal (mysecret)` 
    -> `VALUES('I do not want to replicate it!');`
Query OK, 1 row affected (0,01 sec)
```

对于多表更新，只有属于过滤器指定数据库的表的更新才会被记录。在以下示例中，仅记录对表`cookbook.limbs`的更新：

```
mysql> `use donotlog`
Database changed
mysql> `UPDATE cookbook.limbs, donotlog.onlylocal SET arms=1,` 
    -> `mysecret='I do not want to log it!';`
Query OK, 12 rows affected (0,01 sec)
Rows matched: 12  Changed: 12  Warnings: 0
mysql> `USE cookbook`
Database changed
mysql> `UPDATE cookbook.limbs, donotlog.onlylocal SET arms=0,` 
    -> `mysecret='I do not want to log and replicate this!'` 
    -> `WHERE cookbook.limbs.thing='table';`
Query OK, 2 rows affected (0,00 sec)
Rows matched: 2  Changed: 2  Warnings: 0
```

###### 警告

DDL 语句，例如`ALTER TABLE`，始终以`STATEMENT`格式复制。因此，不管`binlog_format`变量的值如何，这种格式的过滤规则都适用于它们。

如果您希望记录服务器上所有数据库的更新，并仅跳过其中的一些，请使用`binlog-ignore-db`过滤器。多次指定以忽略多个数据库。

```
[mysqld]
binlog-ignore-db=donotlog
binlog-ignore-db=mysql
```

`binlog-ignore-db`过滤器与`binlog-do-db`过滤器类似。在`STATEMENT`二进制日志记录的情况下，它们遵循默认数据库，并在使用`ROW`二进制日志格式时忽略它。如果未指定默认数据库并且使用`STATEMENT`二进制日志格式，将记录所有更新。

如果使用二进制日志格式`MIXED`，则根据更新存储在`STATEMENT`或`ROW`格式中应用过滤规则。

要查找当前正在使用的二进制日志过滤器，请运行*SHOW MASTER STATUS*命令：

```
mysql> `SHOW MASTER STATUS\G`
*************************** 1\. row ***************************
             File: binlog.000008
         Position: 1202
     Binlog_Do_DB: cookbook,test
 Binlog_Ignore_DB: donotlog,mysql
Executed_Gtid_Set: 
1 row in set (0,00 sec)
```

###### 警告

二进制日志文件不仅用于复制，还用于故障时的时间点恢复（PITR）。在这种情况下，过滤后的更新无法恢复，因为它们没有存储在任何地方。如果要将二进制日志用于 PITR 并且仍要过滤某些数据库：请在源服务器上记录所有内容，并在副本上进行过滤。

### 在副本上的过滤器

副本有更多选项来过滤事件。您可以过滤特定数据库或表。您还可以使用通配符。

数据库级别的过滤与源服务器上的过滤相同。由选项`replicate-do-db`和`replicate-ignore-db`控制。如果要过滤多个数据库，请多次指定这些选项。

要过滤特定表，请使用选项`replicate-do-table`和`replicate-ignore-table`。它们将完全限定的表名作为参数。

```
[mysqld]
replicate-do-db=cookbook
replicate-ignore-db=donotlog
replicate-do-table=donotlog.dataforeveryone
replicate-ignore-table=cookbook.limbs
```

但复制过滤的最灵活和安全的语法是`replicate-wild-do-table`和`replicate-wild-ignore-table`。顾名思义，它们在参数中接受通配符。通配符语法与`LIKE`子句中使用的相同。有关`LIKE`子句语法的详细信息，请参阅 Recipe 7.10。

符号`_`替换正好一个字符。因此，`replicate-wild-ignore-table=cookbook.standings_`过滤表`cookbook.standings1`和`cookbook.standings2`，但不过滤`cookbook.standings12`和`cookbook.standings`。

符号`%`替换零个或多个字符。因此，`replicate-wild-do-table=cookbook.movies%`指示副本将更新应用于表`cookbook.movies`、`cookbook.movies_actors`和`cookbook.movies_actors_link`。

如果表名本身包含您不希望替换的通配符字符，则需要对其进行转义。因此，选项`replicate-wild-ignore-table=cookbook.trip_l_g`将过滤表`cookbook.trip_leg`、`cookbook.trip_log`，但也会过滤`cookbook.tripslag`。而`replicate-wild-ignore-table=cookbook.trip\_l_g`只会过滤更新到表`cookbook.trip_leg`和`cookbook.trip_log`。注意，如果在命令行上指定此选项，可能需要根据使用的`SHELL`版本对通配符字符进行双重转义。

###### 提示

表级过滤器与二进制日志格式无关，与默认数据库无关。因此使用它们更安全。如果要在特定数据库或多个数据库中过滤所有表，请使用通配符：

```
[mysqld]
replicate-wild-do-table=cookbook.%
replicate-wild-ignore-table=donotlog.%
```

但是，与数据库过滤器不同，`replicate-wild-do-table`和`replicate-wild-ignore-table`无法过滤存储过程或事件。如果您需要过滤它们，必须使用数据库级别的过滤器。

可以为特定的复制通道设置复制过滤器（Recipe 3.10）。要指定每个通道的过滤器前缀数据库、表名或通配符表达式，后跟冒号：

```
[mysqld]
replicate-do-db=first:cookbook
replicate-ignore-db=second:donotlog
replicate-do-table=first:donotlog.dataforeveryone
replicate-ignore-table=second:cookbook.hitlog
replicate-wild-do-table=first:cookbook.movies%
replicate-wild-ignore-table=second:cookbook.movies%
```

您可以通过*CHANGE REPLICATION FILTER*命令指定复制过滤器，不仅可以通过配置选项。

```
mysql> `CHANGE REPLICATION FILTER`
    -> `REPLICATE_DO_DB = (cookbook),`
    -> `REPLICATE_IGNORE_DB = (donotlog),`
    -> `REPLICATE_DO_TABLE = (donotlog.dataforeveryone),`
    -> `REPLICATE_IGNORE_TABLE = (cookbook.limbs),`
    -> `REPLICATE_WILD_DO_TABLE = ('cookbook.%'),`
    -> `REPLICATE_WILD_IGNORE_TABLE = ('cookbook.trip\_l_g');`
Query OK, 0 rows affected (0.00 sec)
```

###### 提示

每次更改复制参数时，您需要使用*STOP REPLICA*（*STOP SLAVE*）命令停止复制。

要查看当前应用的复制过滤器，请使用*SHOW REPLICA STATUS\G*命令或查询 Performance Schema 中的`replication_applier_filters`和`replication_applier_global_filters`表。

```
mysql> `SHOW REPLICA STATUS\G`
*************************** 1\. row ***************************
               Replica_IO_State: 
                  Source_Host: 127.0.0.1
                  Source_User: root
                  Source_Port: 13000
                Connect_Retry: 60
              Source_Log_File: binlog.000001
          Read_Source_Log_Pos: 156
               Relay_Log_File: Delly-7390-relay-bin.000002
                Relay_Log_Pos: 365
        Relay_Source_Log_File: binlog.000001
             Replica_IO_Running: No
            Replica_SQL_Running: No
              Replicate_Do_DB: cookbook
          Replicate_Ignore_DB: donotlog
           Replicate_Do_Table: donotlog.dataforeveryone
       Replicate_Ignore_Table: cookbook.limbs
      Replicate_Wild_Do_Table: cookbook.%
  Replicate_Wild_Ignore_Table: cookbook.trip\_l_g
...

mysql> `SELECT * FROM performance_schema.replication_applier_filters\G`
*************************** 1\. row ***************************
 CHANNEL_NAME: 
  FILTER_NAME: REPLICATE_DO_DB
  FILTER_RULE: cookbook
CONFIGURED_BY: CHANGE_REPLICATION_FILTER
 ACTIVE_SINCE: 2020-10-04 13:43:21.183768
      COUNTER: 0
*************************** 2\. row ***************************
 CHANNEL_NAME: 
  FILTER_NAME: REPLICATE_IGNORE_DB
  FILTER_RULE: donotlog
CONFIGURED_BY: CHANGE_REPLICATION_FILTER
 ACTIVE_SINCE: 2020-10-04 13:43:21.183768
      COUNTER: 0
*************************** 3\. row ***************************
 CHANNEL_NAME: 
  FILTER_NAME: REPLICATE_DO_TABLE
  FILTER_RULE: donotlog.dataforeveryone
CONFIGURED_BY: CHANGE_REPLICATION_FILTER
 ACTIVE_SINCE: 2020-10-04 13:43:21.183768
      COUNTER: 0
*************************** 4\. row ***************************
 CHANNEL_NAME: 
  FILTER_NAME: REPLICATE_IGNORE_TABLE
  FILTER_RULE: cookbook.limbs
CONFIGURED_BY: CHANGE_REPLICATION_FILTER
 ACTIVE_SINCE: 2020-10-04 13:43:21.183768
      COUNTER: 0
*************************** 5\. row ***************************
 CHANNEL_NAME: 
  FILTER_NAME: REPLICATE_WILD_DO_TABLE
  FILTER_RULE: cookbook.%
CONFIGURED_BY: CHANGE_REPLICATION_FILTER
 ACTIVE_SINCE: 2020-10-04 13:43:21.183768
      COUNTER: 0
*************************** 6\. row ***************************
 CHANNEL_NAME: 
  FILTER_NAME: REPLICATE_WILD_IGNORE_TABLE
  FILTER_RULE: cookbook.trip\_l_g
CONFIGURED_BY: CHANGE_REPLICATION_FILTER
 ACTIVE_SINCE: 2020-10-04 13:43:21.183768
      COUNTER: 0
6 rows in set (0.00 sec)
```

## 参见

有关复制过滤器的更多信息，请参阅[服务器如何评估复制过滤规则](https://dev.mysql.com/doc/refman/8.0/en/replication-rules.html)。

# 3.7 在副本上重写数据库

## 问题

您希望将数据复制到副本上的数据库，该数据库与源服务器上使用的数据库名称不同。

## 解决方案

在副本服务器上使用`replicate-rewrite-db`选项。

## 讨论

MySQL 允许在使用复制过滤器`replicate-rewrite-db`时动态重写数据库名称。

您可以在配置文件、命令行中设置这样的过滤器。

```
[mysqld]
replicate-rewrite-db=cookbook->recipes
```

或通过*CHANGE REPLICATION FILTER*命令：

```
mysql> `CHANGE REPLICATION FILTER` 
    -> `REPLICATE_REWRITE_DB=((cookbook,recipes));`
```

或者，对于多通道副本：

```
[mysqld]
replicate-rewrite-db=channel_id:cookbook->recipes
```

或通过*CHANGE REPLICATION FILTER*命令：

```
mysql> `CHANGE REPLICATION FILTER` 
    -> `REPLICATE_REWRITE_DB=((cookbook,recipes))`
    -> `FOR CHANNEL 'channel_id';`
```

###### 警告

为过滤器值使用双括号和频道名称使用引号。

MySQL 不支持`RENAME DATABASE`操作。因此，要重命名数据库，您需要首先创建一个具有不同名称的数据库，然后将原始数据库的数据恢复到这个新数据库中。

```
mysql> `CREATE DATABASE recipes;`
$ `mysql recipes < cookbook.sql`
```

您需要使用*mysqldump*命令对单个数据库进行导出。如果您使用了`--databases`选项，则还需指定`--no-create-db`选项，以便生成的文件不包含*CREATE DATABASE*语句。

# 3.8 使用多线程副本

## 问题

副本安装在比源更好的硬件上，服务器之间的网络连接良好，但复制延迟正在增加。

## 解决方案

使用多个复制应用线程。

## 讨论

MySQL 服务器是多线程的。它以高度并发的方式应用传入的更新。默认情况下，它在处理应用程序请求时使用所有硬件 CPU 核心。但是，默认情况下，副本服务器仅使用单个线程来应用来自源服务器的传入事件。因此，它使用更少的资源来处理复制事件，甚至在良好的硬件上可能会延迟。

要解决这个问题，请使用多个应用线程。为此，将变量`replica_parallel_workers`设置为大于 1 的值。这指定副本用于应用事件的并行线程数。将此变量的值设置为虚拟 CPU 核心数量是有意义的。变量没有直接影响：您必须重新启动复制以应用更改。

```
mysql> `SET GLOBAL replica_parallel_workers=8;`
Query OK, 0 rows affected (0.01 sec)

mysql> `STOP REPLICA SQL_THREAD;`
Query OK, 0 rows affected (0.01 sec)

mysql> `START REPLICA;`
Query OK, 0 rows affected (0.04 sec)
```

并非所有复制事件都可以并行应用。如果二进制日志包含两个更新同一行的语句，会怎样呢？

```
update limbs set arms=8 where thing='squid';
update limbs set arms=10 where thing='squid';
```

根据事件顺序，乌贼的桌面肢体可能会有八只或十只胳膊。如果这两个语句在源端和副本上以不同顺序执行，则最终数据将不同。

MySQL 使用特殊算法之一进行依赖跟踪。当前算法由副本上的变量`replica_parallel_type`和源端的`binlog_transaction_dependency_tracking`设置。

在 8.0.27 版本之前，变量`replica_parallel_type`的默认值是`DATABASE`，而自此版本起是`LOGICAL_CLOCK`。使用此值时，可以并行应用属于不同数据库的更新，而对同一数据库的更新则顺序应用。此值与源端的`binlog_transaction_dependency_tracking`没有相关性。

数据库级别的并行化对于更新数据库数量少于副本 CPU 核心数量的设置并不会有更好的表现。为了解决这个问题，引入了`replica_parallel_type=LOGICAL_CLOCK`。对于这种类型，属于同一二进制日志组提交的事务会在源端并行应用。

更改变量`replica_parallel_type`后，需要重新启动副本。

**源**服务器上变量`binlog_transaction_dependency_tracking`的值定义了哪些事务属于同一提交组。默认值是`COMMIT_ORDER`，由源端的时间戳生成。使用此值时，源服务器上几乎同时提交的事务将在副本上并行执行。如果源服务器不经常提交，则可能发生副本将在实际上可以并行执行的那些事务（尽管它们在源端被提交的时间不同）顺序执行的情况。

要解决这个问题，引入了`binlog_transaction_dependency_tracking`模式`WRITESET`和`WRITESET_SESSION`。在这些模式下，MySQL 使用由变量`transaction_write_set_extraction`指定的哈希算法来决定事务是否相互依赖，可以是`XXHASH64`（默认）或`MURMUR32`。这意味着如果事务修改了一组行，而这些行彼此独立，那么它们可以并行执行，无论它们之间的提交时间有多长。

使用`binlog_transaction_dependency_tracking`模式设置为`WRITESET`，即使最初在同一会话中执行的事务也可以并行应用。这可能会在某些时间段内导致副本看到不同顺序的更改。根据您的应用需求，这可能是可以接受的或不可接受的。为避免这种情况，您可以启用选项`replica_preserve_commit_order`（`slave_preserve_commit_order`），指示副本按照在源服务器上最初执行它们的顺序应用二进制日志事件。另一种解决方案是将`binlog_transaction_dependency_tracking`设置为`WRITESET_SESSION`。此模式确保来自同一会话的事务永不并行应用。

变量`binlog_transaction_dependency_tracking`是动态的，可以在不停止服务器的情况下进行修改。您还可以仅对特定会话设置它。

## 另请参阅

关于多线程副本的更多信息，请参见[通过基于写集的依赖跟踪改进并行应用程序](https://mysqlhighavailability.com/improving-the-parallel-applier-with-writeset-based-dependency-tracking/)。

# 设置环形复制

## 问题

您想要设置一系列相互复制的服务器。

## 解决方案

使链中的每个服务器既是其对等体的源，又是其副本。

## 讨论

有时您可能需要向多个 MySQL 服务器写入并希望每个服务器上的更新都可见。通过 MySQL 复制，这是可能的。它支持诸如两个服务器、一系列服务器（`A -> B -> C -> D -> ...`）、环形、[星形](https://en.wikipedia.org/wiki/Star_network)等流行设置，以及您可以想象的任何创意设置。对于环形复制的示例，您只需将每个服务器设置为彼此的源和副本即可。

当使用这种复制时，您需要非常小心。因为更新来自任何服务器，它们可能会互相冲突。

想象两个节点同时插入`id=42`的一行。首先，每个节点都插入一行，然后从二进制日志接收到完全相同的事件。复制将因重复键错误而停止。

如果然后您尝试在两个节点上删除`id=42`的行，则会再次收到错误！因为在接收到删除语句时，复制通道中的行已经被删除。

但如果您使用相同的`ID`更新一行，则可能会发生最糟糕的情况。想象一下，如果`node1`将值设置为`42`，而`node2`将值设置为`25`。在应用复制事件后，`node1`将具有值为`25`的行，而`node2`将具有值为`42`的行。这与它们在本地更新后最初拥有的值不同！

依然可以有非常合理的理由使用循环复制。例如，您可能希望将一个节点主要用于一个应用程序，另一个节点用于另一个应用程序。您可以选择适合两者的选项和硬件。或者您可能在不同的地理位置（例如国家）有服务器，并希望将本地数据存储在离用户更近的地方。或者您可以将服务器主要用于读取，但仍然需要更新它们。最后，您可能设置一个热备用服务器，技术上允许写入，但实际上只有在主源服务器宕机时才会接收写入。

在这个方案中，我们将讨论如何设置三个服务器的链条。您可以修改此方案以适用于两个或更多服务器。然后我们将讨论使用复制链条所需的安全考虑事项。

### 设置三个服务器的循环复制

准备用于循环复制的服务器

+   遵循 Recipe 3.1 中的说明为源服务器

+   确保选项 `log_replica_updates` 已启用。否则，如果您的复制链包括两个以上的服务器，则更新仅应用于相邻的服务器。

+   确保选项 `replicate-same-server-id` 已禁用。否则，可能会出现同一更新循环应用的情况。

将节点互相指向

在每个服务器上运行*CHANGE REPLICATION SOURCE*命令，如 Recipe 3.2 或 Recipe 3.4 中描述的那样。指定正确的连接值。例如，如果您想要一个服务器圈 `hostA -> hostB -> hostC -> hostA`，您需要将 `hostB` 指向 `hostA`，`hostA` 指向 `hostC`，`hostC` 指向 `hostB`：

```
hostA> `CHANGE REPLICATION SOURCE TO SOURCE_HOST='hostC', ...`
hostB> `CHANGE REPLICATION SOURCE TO SOURCE_HOST='hostA', ...`
hostC> `CHANGE REPLICATION SOURCE TO SOURCE_HOST='hostB', ...`
```

启动复制

使用*START REPLICA*命令启动复制。

### 使用复制链条时的安全考虑事项

当向多个相互复制的服务器写入时，需要在逻辑上分离要写入的对象。您可以在不同的级别上进行。

业务逻辑

在应用程序级别确保您不会同时在多个服务器上更新同一行。

服务器

一次只向一个服务器写入。这是创建热备用服务器的好解决方案。

数据库和表格

在您的应用程序中：将特定的表集分配给每个服务器。例如，在 `nodeA` 上仅写入表 `movies`、`movies_actors`、`movies_actors_link`；在 `nodeB` 上写入表 `trip_leg` 和 `trip_log`；在 `nodeC` 上写入表 `weatherdata` 和 `weekday`。

行

如果仍然需要在所有服务器上写入同一表，请单独为每个节点分开行。如果您使用带有 `AUTO_INCREMENT` 选项的整数主键，则可以通过将 `auto_increment_increment` 设置为服务器数量，并将 `auto_increment_offset` 设置为链中的服务器编号（从 1 开始）来完成此操作。例如，在我们的三服务器设置中，将 `auto_increment_increment` 设置为 3，在 `nodeA` 上将 `auto_increment_offset` 设置为 1，在 `nodeB` 上设置为 2，在 `nodeC` 上设置为 3。我们在 食谱 15.14 中讨论了如何调整 `auto_increment_increment` 和 `auto_increment_offset`。

如果不使用 `AUTO_INCREMENT`，则需要在应用程序级别创建规则，使标识符在每个节点上遵循其自己的唯一模式。

# 3.10 使用多源复制

## 问题

您希望副本能够应用来自两个或多个相互独立的源服务器的事件。

## 解决方案

通过运行命令 *CHANGE REPLICATION SOURCE ... FOR CHANNEL ‘my source’;* 来创建多个复制通道，针对每个源服务器。

## 讨论

您可能希望从多个服务器复制到一个服务器。例如，如果单独的源服务器由不同的应用程序更新，并且您希望使用复制进行备份或分析。要实现这一点，您需要使用多源复制。

准备服务器进行复制

按照 食谱 3.1 中描述的准备源和复制服务器。对于复制服务器添加额外步骤：配置 `master_info_repository` 和 `relay_log_info_repository` 以使用表：

```
mysql> `SET PERSIST master_info_repository = 'TABLE';`
mysql> `SET PERSIST relay_log_info_repository = 'TABLE';`
```

备份源服务器上的数据

创建完整备份或仅备份您希望复制的数据库。例如，如果您想从一个服务器复制数据库 `cookbook`，从另一个服务器复制数据库 `production`，则仅备份这些数据库。

如果要使用基于位置的复制，请使用 *mysqldump* 并使用选项 `--source-data=2`，该选项指示工具记录 `CHANGE REPLICATION SOURCE` 命令，但将其注释掉。

```
$ `mysqldump --host=source_cookbook --single-transaction --triggers --routines \`
> `--source-data=2 --databases cookbook > cookbook.sql`
```

对于基于 GTID 的复制，使用选项 `--set-gtid-purged=COMMENTED`。

```
$ `mysqldump --host=source_production --single-transaction --triggers --routines \`
> `--set-gtid-purged=COMMENTED --databases production > production.sql`
```

###### 提示

您可以为不同通道使用基于位置和基于 GTID 的复制。您还可以在源服务器上使用不同的二进制日志格式，但在这种情况下，您需要将复制服务器上的二进制日志格式设置为 `MIXED`，以便能够以任何格式存储更新。

恢复副本上的数据

恢复从源服务器收集的数据。

```
$ `mysql < cookbook.sql`
$ `mysql < production.sql`
```

###### 警告

确保源服务器上的数据没有相同名称的数据库。如果有，则需要重命名其中一个数据库，并使用 `replicate-rewrite-db` 过滤器，在应用复制事件时重新写入数据库名称。详细信息请参见 食谱 3.7。

配置复制通道

对于位基复制定位在转储文件 `CHANGE REPLICATION SOURCE` 命令：

```
$ `cat cookbook.sql | grep "CHANGE REPLICATION SOURCE"`
-- CHANGE REPLICATION SOURCE TO SOURCE_LOG_FILE='binlog.000008', ↩
   SOURCE_LOG_POS=2603;
```

并使用结果坐标设置复制。使用`CHANGE REPLICATION SOURCE`命令的`FOR CHANNEL`子句指定要使用的通道。

```
mysql> `CHANGE REPLICATION SOURCE TO`
    -> `SOURCE_HOST='source_cookbook',`
    -> `SOURCE_LOG_FILE='binlog.000008',`
    -> `SOURCE_LOG_POS=2603`
    -> `FOR CHANNEL 'cookbook_channel';`
```

要使用基于 GTID 的复制，请首先找到`SET @@GLOBAL.GTID_PURGED`语句：

```
$ `grep GTID_PURGED production.sql`
/* SET @@GLOBAL.GTID_PURGED='+9113f6b1-0751-11eb-9e7d-0242dc638c6c:1-385';*/
```

对所有使用基于 GTID 的复制的通道执行此操作：

```
$ `grep GTID_PURGED recipes.sql`
/* SET @@GLOBAL.GTID_PURGED='+910c760a-0751-11eb-9da8-0242dc638c6c:1-385';*/
```

然后将它们组合成单个集合：`'9113f6b1-0751-11eb-9e7d-0242dc638c6c:1-385,910c760a-0751-11eb-9da8-0242dc638c6c:1-385'`，运行*RESET MASTER*来重置 GTID 执行历史记录，并将`GTID_PURGED`设置为您刚刚编译的集合：

```
mysql> `RESET MASTER;`
Query OK, 0 rows affected (0,03 sec)

mysql> `SET @@GLOBAL.gtid_purged = '9113f6b1-0751-11eb-9e7d-0242dc638c6c:1-385,`
    '> `910c760a-0751-11eb-9da8-0242dc638c6c:1-385';`
Query OK, 0 rows affected (0,00 sec)
```

然后使用`CHANGE REPLICATION SOURCE`命令设置新通道：

```
mysql> `CHANGE REPLICATION SOURCE TO`
    -> `SOURCE_HOST='source_production',`
    -> `SOURCE_AUTO_POSITION=1`
    -> `FOR CHANNEL 'production_channel';`
```

启动复制

使用*START REPLICA*命令启动复制：

```
mysql> `START REPLICA FOR CHANNEL'cookbook_channel';`
Query OK, 0 rows affected (0,00 sec)

mysql> `START REPLICA FOR CHANNEL 'production_channel';`
Query OK, 0 rows affected (0,00 sec)
```

确认复制正在运行

运行*SHOW REPLICA STATUS*并检查所有通道的记录：

```
mysql> `SHOW REPLICA STATUS\G`
...
             Replica_IO_Running: Yes
            Replica_SQL_Running: Yes
            ...
                 Channel_Name: cookbook_channel
           Source_TLS_Version: 
       Source_public_key_path: 
        Get_source_public_key: 0
            Network_Namespace: 
*************************** 2\. row ***************************
...
             Replica_IO_Running: Yes
            Replica_SQL_Running: Yes
            ...
                 Channel_Name: production_channel
           Source_TLS_Version: 
       Source_public_key_path: 
        Get_source_public_key: 0
            Network_Namespace: 
2 rows in set (0.00 sec)
```

或查询性能模式：

```
mysql> `SELECT CHANNEL_NAME, io.SERVICE_STATE as io_status,` 
    -> `sqlt.SERVICE_STATE as sql_status,`
    -> `COUNT_RECEIVED_HEARTBEATS, RECEIVED_TRANSACTION_SET`
    -> `FROM performance_schema.replication_connection_status AS io` 
    -> `JOIN performance_schema.replication_applier_status AS sqlt USING(channel_name)\G`
*************************** 1\. row ***************************
             CHANNEL_NAME: cookbook_channel
                io_status: ON
               sql_status: ON
COUNT_RECEIVED_HEARTBEATS: 11
 RECEIVED_TRANSACTION_SET: 9113f6b1-0751-11eb-9e7d-0242dc638c6c:1-387
*************************** 2\. row ***************************
             CHANNEL_NAME: production_channel
                io_status: ON
               sql_status: ON
COUNT_RECEIVED_HEARTBEATS: 11
 RECEIVED_TRANSACTION_SET: 910c760a-0751-11eb-9da8-0242dc638c6c:1-385
2 rows in set (0.00 sec)
```

# 3.11 使用半同步复制插件

## 问题

在*COMMIT*操作完成之前，确保至少有一个副本已更新。

## 解决方案

使用半同步复制插件。

## 讨论

MySQL 复制是异步的。这意味着源服务器可以非常快速地接受写入。它只需要将数据存储在表中，并将更改信息写入二进制日志文件。但是，它不知道副本是否接收到任何更新，如果接收到，是否已应用这些更新。

我们无法保证异步副本是否应用更新，但我们可以设置确保更新已接收并存储在中继日志文件中，以防灾难恢复。这并不保证更新将被应用，或者如果应用了更新，结果与源服务器上的值相同，但保证至少有两台服务器记录了可以应用的更新。为此，请使用半同步复制插件。

半同步复制插件应安装在源服务器和副本服务器上。

在源服务器上运行：

```
mysql> `INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';`
Query OK, 0 rows affected (0.03 sec)
```

在副本上运行：

```
mysql> `INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';`
Query OK, 0 rows affected (0.00 sec)
```

安装后，您可以启用半同步复制。在源端将全局变量`rpl_semi_sync_source_enabled`设置为 1。在副本上使用变量`rpl_semi_sync_replica_enabled`。

###### 警告

半同步复制仅与默认复制通道兼容。您不能与多源复制一起使用。

您可以通过变量来控制半同步复制的行为，如表 3-1 所示：

表 3-1\. 控制半同步复制插件行为的变量

| 变量 | 控制什么 | 默认值 |
| --- | --- | --- |
| `rpl_semi_sync_source_timeout` | 等待来自副本响应的毫秒数。如果超过此值，复制将静默转换为异步。 | 10000 |
| `rpl_semi_sync_source_wait_for_replica_count` | 在提交事务之前，源服务器需要接收确认的副本数量。 | 1 |
| `rpl_semi_sync_source_wait_no_replica` | 如果连接的副本数低于 `rpl_semi_sync_source_wait_for_replica_count`，会发生什么情况。只要这些服务器稍后重新连接并确认事务，半同步仍然可用。如果此变量为 `OFF`，则一旦副本数量低于 `rpl_semi_sync_source_wait_for_replica_count`，则复制转换为异步。 | ON |
| `rpl_semi_sync_source_wait_point` | 期望从副本接收到事务的确认的时机。该变量支持两种可能的值。在 `AFTER_SYNC` 的情况下，源服务器将每个事务写入二进制日志，然后将其同步到磁盘。源服务器等待副本关于已接收更改的确认，然后提交事务。在 `AFTER_COMMIT` 的情况下，源服务器提交事务，然后等待副本的确认，成功后返回给客户端。 | `AFTER_SYNC` |

要查看半同步复制的状态，请使用变量 `Rpl_semi_sync_*`。源服务器上有很多这样的变量。

```
mysql> `SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync%';`
+--------------------------------------------+-------+
| Variable_name                              | Value |
+--------------------------------------------+-------+
| Rpl_semi_sync_source_clients               | 1     |
| Rpl_semi_sync_source_net_avg_wait_time     | 0     |
| Rpl_semi_sync_source_net_wait_time         | 0     |
| Rpl_semi_sync_source_net_waits             | 9     |
| Rpl_semi_sync_source_no_times              | 3     |
| Rpl_semi_sync_source_no_tx                 | 6     |
| Rpl_semi_sync_source_status                | ON    |
| Rpl_semi_sync_source_timefunc_failures     | 0     |
| Rpl_semi_sync_source_tx_avg_wait_time      | 1021  |
| Rpl_semi_sync_source_tx_wait_time          | 4087  |
| Rpl_semi_sync_source_tx_waits              | 4     |
| Rpl_semi_sync_source_wait_pos_backtraverse | 0     |
| Rpl_semi_sync_source_wait_sessions         | 0     |
| Rpl_semi_sync_source_yes_tx                | 4     |
+--------------------------------------------+-------+
14 rows in set (0.00 sec)
```

最重要的是 `Rpl_semi_sync_source_clients`，它显示半同步是否当前正在使用以及连接了多少半同步副本。如果 `Rpl_semi_sync_source_clients` 为零，则没有半同步副本连接，使用异步复制。

在副本服务器上，只有变量 `Rpl_semi_sync_replica_status` (`Rpl_semi_sync_slave_status`) 可用，并且可能的值为 `ON` 或 `OFF`。

###### 警告

如果在 `rpl_semi_sync_source_timeout` 毫秒内没有副本接受写入，则复制将切换到异步，对客户端没有任何消息或警告。唯一确定复制模式切换为异步的方法是检查变量 `Rpl_semi_sync_source_clients` 的值或者检查错误日志文件以查找如下消息：

```
2020-10-12T22:25:17.654563Z 0 [ERROR] [MY-013129] [Server] ↩
A message intended for a client cannot be sent there as ↩
no client-session is attached. Therefore, ↩
we're sending the information to the error-log instead: ↩

MY-001158 - Got an error reading communication packets

2020-10-12T22:25:20.083796Z 198 [Note] [MY-010014] [Repl] ↩
While initializing dump thread for slave with UUID ↩
<09bf4498-0cd2-11eb-9161-98af65266957>, ↩
found a zombie dump thread with the same UUID. ↩
Master is killing the zombie dump thread(180).

2020-10-12T22:25:20.084088Z 180 [Note] [MY-011171] [Server] ↩
Stop semi-sync binlog_dump to slave (server_id: 2).

2020-10-12T22:25:20.084204Z 198 [Note] [MY-010462] [Repl] ↩
Start binlog_dump to master_thread_id(198) slave_server(2), ↩ 
pos(, 4)

2020-10-12T22:25:20.084248Z 198 [Note] [MY-011170] [Server] ↩
Start asynchronous binlog_dump to slave (server_id: 2), pos(, 4).

2020-10-12T22:25:20.657800Z 180 [Note] [MY-011155] [Server] ↩
Semi-sync replication switched OFF.
```

我们在 Recipe 23.2 讨论错误日志文件。

# 3.12 使用组复制

## 问题

您希望将更新应用到所有节点或者不应用。

## 解决方案

使用组复制。

## 讨论

从版本 5.7.17 开始，MySQL 通过 Group Replication 插件完全支持同步复制。如果使用了该插件，MySQL 服务器（称为节点）将创建一个组，共同提交或者如果其中一个成员未能应用事务，则回滚。这样，更新要么复制到所有组成员，要么不复制。确保了高可用性。

组内最多可以有九台服务器。超过九台不受支持。对于这种限制有一个非常好的理由：更多的服务器意味着更高的复制延迟。在同步复制的情况下，所有更新在事务完成之前都会应用到所有节点上。每个更新都会传输到每个节点，等待应用，然后才提交。因此，复制延迟取决于最慢节点的速度和网络传输速率。

尽管在组复制设置中技术上可以少于三个服务器，但更少的数量无法提供适当的高可用性。这是因为[Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science))算法，由[Group Communication Engine](https://dev.mysql.com/doc/refman/8.0/en/group-replication-plugin-architecture.html)使用，需要`2F + 1`个节点来创建一个仲裁，其中`F`是任意自然数。换句话说，在灾难发生时，活动节点的数量应大于断开连接的节点数量。

组复制有限制。首先，最重要的是，它仅支持存储引擎 InnoDB。在启用插件之前，您需要禁用其他存储引擎。每个复制的表必须有主键。您应该将服务器放置在本地网络中。虽然跨互联网进行组复制是可能的，但由于网络超时，这可能导致应用事务和将节点从组中断开连接需要更长的时间。语句*LOCK TABLE*和*GET_LOCK*不会考虑用于确保事务是否应在所有节点上应用或回滚的认证过程，这意味着它们是本地节点的，容易出错。您可以在[Group Replication Limitations](https://dev.mysql.com/doc/refman/8.0/en/group-replication-limitations.html)用户参考手册中找到完整的限制列表。

要启用组复制，您需要按照 Recipe 3.1 中描述的方式准备所有参与的服务器，因为它们将作为源和副本同时运行，并进行额外的准备工作。不要启动复制。

1.  准备配置文件

    ```
    [mysqld]
    # Disable unsupported storage engines
    disabled_storage_engines="MyISAM,BLACKHOLE,FEDERATED,ARCHIVE,MEMORY"

    # Set unique server ID. Each server in the group should have its own ID
    server_id=1

    # Enable GTIDs
    gtid_mode=ON
    enforce_gtid_consistency=ON

    # Enable replica updates
    log_replica_updates=ON

    # Only ROW binary log format supported
    binlog_format=ROW

    # For versions before 8.0.21
    binlog_checksum=NONE

    # Ensure that replication repository is TABLE
    master_info_repository=TABLE
    relay_log_info_repository=TABLE

    # Ensure that transaction_write_set_extraction is enabled
    # This option is deprecated starting from version 8.0.26
    transaction_write_set_extraction=XXHASH64

    # Add Group Replication options
    plugin_load_add='group_replication.so'

    # Any valid UUID, should be same for all group members.
    # Use SELECT UUID() to generate a UUID
    group_replication_group_name="dc527338-13d1-11eb-abf7-98af65266957"

    # Host of the local node and port which will be used 
    # for communication between members
    # Put either hostname (in our case node1) or IP address here
    # Port number should be different from from the one, used for serving clients
    # E.g., if default MySQL port is 3306, specify any different number here
    group_replication_local_address= "node1:33061"

    # Ports and addresses of all nodes in the group. 
    # Should be same on all nodes
    group_replication_group_seeds= "node1:33061,node2:33061,node3:33061"

    # Since we did not setup Group replication at this stage,
    # it should not be started on boot
    # You may set this option ON after bootstrapping the group
    group_replication_start_on_boot=off
    group_replication_bootstrap_group=off

    # Request source server public key for 
    #the authentication plugin caching_sha2_password
    group_replication_recovery_get_public_key=1
    ```

1.  启动服务器。但是先不要启用复制。

1.  选择将成为组中第一个节点的节点。

1.  仅在第一个成员上创建复制用户，如 Recipe 3.1 中描述的那样，并额外授予`BACKUP_ADMIN`权限。

    ```
    node1> `CREATE USER repl@'%' IDENTIFIED BY 'replrepl';`
    Query OK, 0 rows affected (0,01 sec) 

    node1> `GRANT REPLICATION SLAVE, BACKUP_ADMIN ON *.* TO repl@'%';`
    Query OK, 0 rows affected (0,03 sec)
    ```

    您不需要在其他组成员上创建复制用户，因为*CREATE USER*语句将被复制。

1.  在第一个成员上设置复制以使用此用户：

    ```
    node1> `CHANGE REPLICATION SOURCE TO SOURCE_USER='repl',`
        -> `SOURCE_PASSWORD='replrepl'` 
        -> `FOR CHANNEL 'group_replication_recovery';`
    Query OK, 0 rows affected (0,01 sec)
    ```

    通道名`group_replication_recovery`是组复制通道的特殊内置名称。

    ###### 提示

    如果您不希望复制凭证以明文形式存储在复制存储库中，请跳过此步骤，并在运行*START GROUP_REPLICATION*时稍后提供凭证。还参见 Recipe 3.13

1.  引导节点。

    ```
    node1> `SET GLOBAL group_replication_bootstrap_group=ON;`
    Query OK, 0 rows affected (0,00 sec) 

    node1> `START GROUP_REPLICATION;`
    Query OK, 0 rows affected (0,00 sec) 

    node1> `SET GLOBAL group_replication_bootstrap_group=OFF;`
    Query OK, 0 rows affected (0,00 sec)
    ```

1.  通过从`performance_schema.replication_group_members`中选择来检查组复制状态。

    ```
    node1> `SELECT * FROM performance_schema.replication_group_members\G`
    *************************** 1\. row ***************************
      CHANNEL_NAME: group_replication_applier
         MEMBER_ID: d8a706aa-16ee-11eb-ba5a-98af65266957
       MEMBER_HOST: node1
       MEMBER_PORT: 33361
      MEMBER_STATE: ONLINE
       MEMBER_ROLE: PRIMARY
    MEMBER_VERSION: 8.0.21
    1 row in set (0.00 sec)
    ```

    等待第一个成员状态变为`ONLINE`。

1.  在第二个和第三个节点上启动复制。

    ```
    node2> `CHANGE REPLICATION SOURCE TO SOURCE_USER='repl',`
        -> `SOURCE_PASSWORD='replrepl'` 
        -> `FOR CHANNEL 'group_replication_recovery';`
    Query OK, 0 rows affected (0,01 sec) 

    node2> `START GROUP_REPLICATION;`
    Query OK, 0 rows affected (0,00 sec)
    ```

一旦确认所有成员都处于`ONLINE`状态，您可以使用组复制。查询表`performance_schema.replication_group_members`以获取此信息。健康的设置将输出如下内容：

```
node1> `SELECT * FROM performance_schema.replication_group_members\G`
*************************** 1\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: d8a706aa-16ee-11eb-ba5a-98af65266957
   MEMBER_HOST: node1
   MEMBER_PORT: 33061
  MEMBER_STATE: ONLINE
   MEMBER_ROLE: PRIMARY
MEMBER_VERSION: 8.0.21
*************************** 2\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: e14043d7-16ee-11eb-b77a-98af65266957
   MEMBER_HOST: node2
   MEMBER_PORT: 33061
  MEMBER_STATE: ONLINE
   MEMBER_ROLE: SECONDARY
MEMBER_VERSION: 8.0.21
*************************** 3\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: ea775284-16ee-11eb-8762-98af65266957
   MEMBER_HOST: node3
   MEMBER_PORT: 33061
  MEMBER_STATE: ONLINE
   MEMBER_ROLE: SECONDARY
MEMBER_VERSION: 8.0.21
3 rows in set (0.00 sec)
```

###### 警告

命令*SHOW REPLICA STATUS*在组复制中不起作用。

如果您希望使用现有数据启动组复制，请在引导第一个节点之前将其恢复。当其他节点加入组时，数据将被复制。

最后，在节点配置文件中启用`group_replication_start_on_boot=on`选项，以便在节点重新启动后启用复制。

###### 提示

在本示例中，我们在单主模式下启动了组复制。这种模式只允许在组中的一个成员上进行写操作。这是最安全和推荐的选项。然而，如果您希望在多个节点上写入，可以使用函数*group_replication_switch_to_multi_primary_mode*切换到多主节点：

```
mysql> `SELECT group_replication_switch_to_multi_primary_mode();`
+--------------------------------------------------+
| group_replication_switch_to_multi_primary_mode() |
+--------------------------------------------------+
| Mode switched to multi-primary successfully.     |
+--------------------------------------------------+
1 row in set (1.01 sec)

mysql> `SELECT * FROM performance_schema.replication_group` ↩
`_members\G`
*************************** 1\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: d8a706aa-16ee-11eb-ba5a-98af65266957
   MEMBER_HOST: node1
   MEMBER_PORT: 33061
  MEMBER_STATE: ONLINE
   MEMBER_ROLE: PRIMARY
MEMBER_VERSION: 8.0.21
*************************** 2\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: e14043d7-16ee-11eb-b77a-98af65266957
   MEMBER_HOST: node2
   MEMBER_PORT: 33061
  MEMBER_STATE: ONLINE
   MEMBER_ROLE: PRIMARY
MEMBER_VERSION: 8.0.21
*************************** 3\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: ea775284-16ee-11eb-8762-98af65266957
   MEMBER_HOST: node3
   MEMBER_PORT: 33061
  MEMBER_STATE: ONLINE
   MEMBER_ROLE: PRIMARY
MEMBER_VERSION: 8.0.21
3 rows in set (0.00 sec)
```

欲了解更多详情，请查看[更改组模式用户手册](https://dev.mysql.com/doc/refman/8.0/en/group-replication-changing-group-mode.html)。

## 另请参阅

欲了解有关组复制的更多信息，请参阅[用户参考手册中的组复制](https://dev.mysql.com/doc/refman/8.0/en/group-replication.html)。

# 3.13 安全存储复制凭据

## 问题

默认情况下，如果在*CHANGE REPLICATION SOURCE*命令中指定，则复制凭据会显示在复制信息存储库中。您希望通过不授权用户的偶发访问来隐藏它们。

## 解决方案

使用*START REPLICA*命令的`USER`和`PASSWORD`选项。

## 讨论

当您使用*CHANGE REPLICATION SOURCE*命令指定复制用户凭据时，无论`master_info_repository`选项如何，它们都以明文、未加密的形式存储。

因此，如果`master_info_repository='TABLE'`（自 8.0 版本起为默认设置），任何具有对`mysql`数据库的读取访问权限的用户都可以查询`slave_master_info`表并读取密码：

```
mysql> `SELECT User_name, User_password FROM slave_master_info;`
+-----------+---------------+
| User_name | User_password |
+-----------+---------------+
| repl      | replrepl      |
+-----------+---------------+
1 row in set (0.00 sec)
```

或者，如果`master_info_repository='FILE'`，则任何可以访问该文件的操作系统用户（默认情况下位于 MySQL 数据目录中）都可以获取复制凭据：

```
$ `head -n6 var/mysqld.3/data/master.info`
31
binlog.000001
688
127.0.0.1
repl
replrepl
```

如果这不是期望的行为，您可以在*START REPLICA*或*START GROUP_REPLICATION*命令中指定复制凭据：

```
mysql> `START REPLICA USER='repl' PASSWORD='replrepl';`
Query OK, 0 rows affected (0.01 sec)
```

但是，如果您先前在*CHANGE MASTER*命令中指定了复制凭据，它们将继续显示在主信息存储库中。要清除先前输入的用户和密码，请使用空参数运行*CHANGE MASTER*命令以清除`MASTER_USER`和`MASTER_PASSWORD`：

```
mysql> `SELECT User_name, User_password FROM slave_master_info;`
+-----------+---------------+
| User_name | User_password |
+-----------+---------------+
| repl      | replrepl      |
+-----------+---------------+
1 row in set (0.00 sec)

mysql> `CHANGE REPLICATION SOURCE TO SOURCE_USER='', SOURCE_PASSWORD='';`
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> `START REPLICA USER='repl' PASSWORD='replrepl';`
Query OK, 0 rows affected (0.01 sec)

mysql> `SELECT User_name, User_password FROM slave_master_info;`
+-----------+---------------+
| User_name | User_password |
+-----------+---------------+
|           |               |
+-----------+---------------+
1 row in set (0.00 sec)
```

###### 警告

一旦您从源信息存储库中清除了复制凭据，它们就不会存储在任何地方，您将需要每次重新启动复制时提供它们。

# 3.14 使用 TLS（SSL）进行复制

## 问题

您希望在源和副本之间安全传输数据。

## 解决方案

为复制通道设置传输层安全性（[Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security)）连接。

## 讨论

源服务器和复制服务器之间的连接在技术上类似于连接到 MySQL 服务器的任何其他客户端连接。因此，通过 TLS 加密它需要类似于在 Recipe 24.10 中描述的加密客户端连接的准备工作。

要创建加密复制设置，请按照以下步骤操作。

1.  根据 Recipe 24.10 的描述获取或创建 TLS 密钥和证书。

1.  确保源服务器在[mysqld]部分下具有 TLS 配置参数：

    ###### 注意

    虽然 MySQL 在最新版本中使用了现代更安全的 TLS 协议，但其配置选项仍然使用缩写 SSL。MySQL 用户参考手册也经常将 TLS 称为 SSL。

    ```
    [mysqld]
    ssl_ca=ca.pem
    ssl_cert=server-cert.pem
    ssl_key=server-key.pem
    ```

    您可以检查系统变量`have_ssl`的值是否启用了 TLS。

    ```
    mysql> `SHOW VARIABLES LIKE 'have_ssl';`
    +---------------+-------+
    | Variable_name | Value |
    +---------------+-------+
    | have_ssl      | YES   |
    +---------------+-------+
    1 row in set (0,01 sec)
    ```

1.  如果运行不安全的复制，请停止复制 IO 线程：

    ```
    mysql> `STOP REPLICA IO_THREAD; -- (STOP SLAVE IO_THREAD;)`
    Query OK, 0 rows affected (0.00 sec)
    ```

1.  在复制服务器上，在配置文件的`[client]`部分下放置 TLS 客户端密钥和证书的路径：

    ```
    [client]
    ssl-ca=ca.pem
    ssl-cert=client-cert.pem
    ssl-key=client-key.pem
    ```

    并为*CHANGE REPLICATION SOURCE*命令指定选项`SOURCE_SSL=1`：

    ```
    mysql> `CHANGE REPLICATION SOURCE TO SOURCE_SSL=1;`
    Query OK, 0 rows affected (0.03 sec
    ```

    或者，您可以将客户端密钥和证书的路径作为*CHANGE REPLICATION SOURCE*命令的一部分指定：

    ```
    mysql> `CHANGE REPLICATION SOURCE TO`
        -> `SOURCE_SSL_CA='ca.pem',`
        -> `SOURCE_SSL_CERT='client-cert.pem',`
        -> `SOURCE_SSL_KEY='client-key.pem',`
        -> `SOURCE_SSL=1;`
    Query OK, 0 rows affected (0.02 sec)
    ```

    ###### 注意

    我们出于简洁起见故意省略了*CHANGE REPLICATION SOURCE*命令的其他参数，例如`SOURCE_HOST`。但您需要按照 Recipe 3.2 或 Recipe 3.4 中描述的方式使用它们。

1.  启动复制：

    ```
    mysql> `START REPLICA;`
    Query OK, 0 rows affected (0.00 sec)
    ```

*CHANGE REPLICATION SOURCE* 命令支持其他 TLS 修饰符，与常规客户端连接加密选项兼容。例如，您可以在`SOURCE_SSL_CIPHER`子句中指定要使用的密码，或者在`SOURCE_SSL_VERIFY_SERVER_CERT`子句中强制验证源服务器证书。

## 参见

要获取有关在源和复制服务器之间设置加密连接的更多信息，请参阅[Setting Up Replication to Use Encrypted Connections](https://dev.mysql.com/doc/refman/8.0/en/replication-solutions-encrypted-connections.html)。

# 3.15 复制故障排除

## 问题

复制未正常工作，您希望修复它。

## 解决方案

使用*SHOW REPLICA STATUS*命令，查询 Performance Schema 中的复制表，并检查错误日志文件以了解复制失败的原因，然后进行修复。

## 讨论

复制由两种类型的线程管理：IO 和 SQL（或连接和应用）。 IO 或连接线程负责连接到源服务器，检索更新并将其存储在中继日志文件中。每个复制通道始终有一个 IO 线程。 SQL 或应用程序线程从中继日志文件读取数据并将更改应用于表。一个复制通道可能有多个 SQL 线程。连接和应用程序线程完全独立，其错误由不同的复制诊断工具报告。

诊断复制错误有两种主要工具：*SHOW REPLICA STATUS* 命令和性能模式中的复制表。*SHOW REPLICA STATUS* 从一开始就存在，而性能模式中的复制表是从版本 5.7 开始添加的。使用这两种工具可以获得非常相似的信息，使用哪种取决于个人偏好。在我们看来，*SHOW REPLICA STATUS* 适合在命令行中进行手动审查，而使用性能模式进行监控警报和查询比解析 *SHOW REPLICA STATUS* 输出要容易得多。

### SHOW REPLICA STATUS

*SHOW REPLICA STATUS* 包含有关 IO 和 SQL 线程配置、状态和错误的所有信息。所有数据都在单行中打印。但是，此行使用空格和换行符进行格式化。通过 *mysql* 客户端的 `\G` 修改器可以轻松地检查它。对于多源复制，*SHOW REPLICA STATUS* 分别打印每个通道的信息。

```
mysql> `SHOW REPLICA STATUS\G`
*************************** 1\. row ***************************
               Replica_IO_State: Waiting for master to send event
                  Source_Host: 127.0.0.1
                  Source_User: root
                  Source_Port: 13000
                Connect_Retry: 60
              Source_Log_File: binlog.000001
          Read_Source_Log_Pos: 156
               Relay_Log_File: Delly-7390-relay-bin-cookbook.000002
                Relay_Log_Pos: 365
        Relay_Source_Log_File: binlog.000001
             Replica_IO_Running: Yes
            Replica_SQL_Running: Yes
            ...
                 Channel_Name: cookbook
           Source_TLS_Version: 
       Source_public_key_path: 
        Get_source_public_key: 0
            Network_Namespace: 
*************************** 2\. row ***************************
               Replica_IO_State: Waiting for master to send event
                  Source_Host: 127.0.0.1
                  Source_User: root
                  Source_Port: 13004
                Connect_Retry: 60
              Source_Log_File: binlog.000001
          Read_Source_Log_Pos: 156
               Relay_Log_File: Delly-7390-relay-bin-test.000002
                Relay_Log_Pos: 365
        Relay_Source_Log_File: binlog.000001
             Replica_IO_Running: Yes
            Replica_SQL_Running: Yes
            ...
                 Channel_Name: test
           Source_TLS_Version: 
       Source_public_key_path: 
        Get_source_public_key: 0
            Network_Namespace: 
2 rows in set (0.00 sec)
```

为了简洁起见，我们有意跳过了部分输出。我们不会描述每个字段，只描述处理停止复制所需的字段（见 表 3-2）。如果您想了解其他字段的含义，请参阅 [SHOW REPLICA STATUS Statement](https://dev.mysql.com/doc/refman/8.0/en/show-replica-status.html) 用户参考手册。

表 3-2\. *SHOW REPLICA STATUS* 字段含义，用于理解和修复错误

| Field | 描述 | 子系统 |
| --- | --- | --- |
| `Replica_IO_State` (`Slave_IO_State`) | IO 线程的状态。包含连接线程运行时的信息，如果 IO 线程停止则为空，如果连接尚未建立则为 `Connecting`。 | IO 线程状态 |
| `Source_Host` (`Master_Host`) | 源服务器的主机名。 | IO 线程配置 |
| `Source_User` (`Master_User`) | 复制用户。 | IO 线程配置 |
| `Source_Port` (`Master_Port`) | 源服务器的端口号。 | IO 线程配置 |
| `Source_Log_File` (`Master_Log_File`) | IO 线程当前正在读取的源服务器的二进制日志。 | IO 线程状态 |
| `Read_Source_Log_Pos` (`Read_Master_Log_Pos`) | IO 线程正在读取的源服务器二进制日志文件中的位置。 | IO 线程状态 |
| `Relay_Log_File` | 当前的中继日志文件：SQL 线程当前正在执行的文件。 | IO 线程状态 |
| `Relay_Log_Pos` | SQL 线程已执行到的中继日志文件位置。 | IO 线程状态 |
| `Relay_Source_Log_File` (`Relay_Master_Log_File`) | SQL 线程执行事件时源服务器上的二进制日志。 | SQL 线程状态 |
| `Replica_IO_Running` (`Slave_IO_Running`) | 指示 IO 线程是否正在运行。使用此字段快速识别连接线程的健康状态。 | IO 线程状态 |
| `Replica_SQL_Running` (`Slave_SQL_Running`) | SQL 线程是否正在运行。用于快速识别应用程序线程的健康状况。 | SQL 线程状态 |
| `Replicate_*` | 复制过滤器。 | SQL 线程配置 |
| `Exec_Source_Log_Pos` (`Exec_Master_Log_Pos`) | SQL 线程执行事件时源二进制日志文件的位置。 | SQL 线程状态 |
| `Until_Condition` | 如有任何条件，直到的条件。 | SQL 线程配置 |
| `Source_SSL_*` (`Master_SSL_*`) | 连接到源服务器的 SSL 选项。 | IO 线程配置 |
| `Seconds_Behind_Source` (`Seconds_Behind_Master`) | 源服务器和副本之间的预估延迟时间。 | SQL 线程状态 |
| `Last_IO_Errno` | IO 线程的最后一个错误编号。解决后清除。 | IO 线程状态 |
| `Last_IO_Error` | IO 线程上的最新错误。解决后清除。 | IO 线程状态 |
| `Last_Errno`, `Last_SQL_Errno` | SQL 线程接收的最后一个错误编号。解决后清除。 | SQL 线程状态 |
| `Last_Error`, `Last_SQL_Error` | SQL 线程的最后错误。解决后清除。 | SQL 线程状态 |
| `Replica_SQL_Running_State` (`Slave_SQL_Running_State`) | SQL 线程的状态。如果停止则为空。 | SQL 线程状态 |
| `Last_IO_Error_Timestamp` | 上次 IO 错误发生的时间。解决后清除。 | IO 线程状态 |
| `Last_SQL_Error_Timestamp` | 上次 SQL 错误发生的时间。解决后清除。 | SQL 线程状态 |
| `Retrieved_Gtid_Set` | 连接线程检索的 GTID 集合。 | IO 线程状态 |
| `Executed_Gtid_Set` | SQL 线程执行的 GTID 集合。 | SQL 线程状态 |
| `Channel_Name` | 复制通道的名称。 | IO 和 SQL 线程配置 |

在讨论如何处理特定 IO 和 SQL 线程错误时，我们将参考此表格。

### 性能模式中的复制表格

另一种诊断解决方案：性能模式中的表格，不像 *SHOW REPLICA STATUS*，不会将所有信息存储在单个位置，而是在单独的空间中保存。

IO 线程配置信息存储在表 `replication_connection_configuration` 中，其状态信息存储在表 `replication_connection_status` 中。

关于 SQL 线程的信息存储在六个表中，如 表 3-3 所示。

表 3-3\. 包含特定 SQL 线程信息的表格

| 表名 | 描述 |
| --- | --- |
| `replication_applier_configuration` | SQL 线程配置。 |
| `replication_applier_global_filters` | 全局复制过滤器：适用于所有通道。 |
| `replication_applier_filters` | 特定于特定通道的复制过滤器。 |
| `replication_applier_status` | SQL 线程的全局状态。 |
| `replication_applier_status_by_worker` | 对于多线程副本：每个 SQL 线程的状态。 |
| `replication_applier_status_by_coordinator` | 对于多线程复制：协调者视角下 SQL 线程的状态。  |

最后，您将在 `replication_group_members` 表中找到组复制网络配置和状态，以及在 `replication_group_member_stats` 表中找到组复制成员的统计信息。

### 修复 IO 线程问题

您可以通过检查 *SHOW REPLICA STATUS* 的 `Replica_IO_Running` 字段的值来查看复制 IO 线程是否存在问题。如果值不是 `Yes`，连接线程可能遇到问题。发生这种情况的原因可以在 `Last_IO_Errno` 和 `Last_IO_Error` 字段中找到。

```
mysql> `SHOW REPLICA STATUS\G`
*************************** 1\. row ***************************
...
             Replica_IO_Running: Connecting
            Replica_SQL_Running: Yes
...
                Last_IO_Errno: 1045
                Last_IO_Error: error connecting to master 'repl@127.0.0.1:13000' - ↩
                               retry-time: 60 retries: 1 message: ↩
                               Access denied for user 'repl'@'localhost'↩
                               (using password: NO)
...
```

就像上面的示例中，副本无法连接到源服务器，因为用户 `'repl'@'localhost'` 的访问被拒绝。IO 线程仍在运行，并将在 60 秒后重新尝试连接（`retry-time: 60`）。导致此类失败的原因很明确：要么在源服务器上不存在该用户，要么其权限不足。您需要连接到源服务器并修复用户帐户。一旦修复，下一次连接尝试将成功。

或者，您可以查询性能模式下的 `replication_connection_status` 表：

```
mysql> `SELECT SERVICE_STATE, LAST_ERROR_NUMBER,`
    -> `LAST_ERROR_MESSAGE, LAST_ERROR_TIMESTAMP`
    -> `FROM performance_schema.replication_connection_status\G`
*************************** 1\. row ***************************
       SERVICE_STATE: CONNECTING
   LAST_ERROR_NUMBER: 2061
  LAST_ERROR_MESSAGE: error connecting to master 'repl@127.0.0.1:13000' -↩
                      retry-time: 60 retries: 1 ↩
                      message: Authentication plugin 'caching_sha2_password' ↩
                      reported error: Authentication requires secure connection.
LAST_ERROR_TIMESTAMP: 2020-10-17 13:23:03.663994
1 row in set (0.00 sec)
```

在此示例中，字段 `LAST_ERROR_MESSAGE` 包含 IO 线程无法连接的原因：源服务器上的用户帐户使用要求安全连接的身份验证插件 `caching_sha2_password`。要修复此错误，您需要停止复制，然后使用参数 `SOURCE_SSL=1` 或 `GET_SOURCE_PUBLIC_KEY=1` 运行 *CHANGE REPLICATION SOURCE*。在后一种情况下，副本与源服务器之间的流量将保持不安全，只有密码交换通信将得到保护。有关详细信息，请参阅 Recipe 3.14。

### 修复 SQL 线程问题

要查找为什么应用程序线程已停止，请检查 `Replica_SQL_Running`、`Last_SQL_Errno` 和 `Last_SQL_Error` 字段：

```
mysql> `SHOW REPLICA STATUS\G`
*************************** 1\. row ***************************
...
            Replica_SQL_Running: No
...
               Last_SQL_Errno: 1007
               Last_SQL_Error: Error 'Can't create database 'cookbook'; ↩
                               database exists' on query. ↩
                               Default database: 'cookbook'. ↩
                               Query: 'create database cookbook'
```

在上面的列表中，错误消息显示 *CREATE DATABASE* 命令失败，因为在副本上已存在这样的数据库。

同样的信息也可以在性能模式下的 `replication_applier_status_by_worker` 表中找到：

```
mysql> `SELECT SERVICE_STATE, LAST_ERROR_NUMBER,`
    -> `LAST_ERROR_MESSAGE, LAST_ERROR_TIMESTAMP`
    -> `FROM performance_schema.replication_applier_status_by_worker\G`
*************************** 1\. row ***************************
       SERVICE_STATE: OFF
   LAST_ERROR_NUMBER: 1007
  LAST_ERROR_MESSAGE: Error 'Can't create database 'cookbook'; ↩
                      database exists' on query. ↩
                      Default database: 'cookbook'.↩
                      Query: 'create database cookbook'
LAST_ERROR_TIMESTAMP: 2020-10-17 13:58:12.115821
1 row in set (0.01 sec)
```

解决此问题的方法有几种。首先，您可以简单地在副本上删除数据库并重新启动 SQL 线程：

```
mysql> `DROP DATABASE cookbook;`
Query OK, 0 rows affected (0.04 sec)

mysql> `START REPLICA SQL_THREAD;`
Query OK, 0 rows affected (0.01 sec)
```

如果副本上启用了二进制日志，请禁用它。

如果您希望在副本上保留数据库：例如，如果它应该包含源服务器上不存在的额外表格，您可以跳过复制的事件。

如果您使用基于位置的复制，请使用变量 `sql_replica_skip_counter`（`sql_slave_skip_counter`）：

```
mysql> `SET GLOBAL sql_replica_skip_counter=1;`
Query OK, 0 rows affected (0.00 sec)

mysql> `START REPLICA SQL_THREAD;`
Query OK, 0 rows affected (0.01 sec)
```

在此示例中，我们跳过了二进制日志中的一个事件，然后重新启动了复制。

对于基于 GTID 的复制设置，`sql_replica_skip_counter`不起作用，因为它不包括 GTID 信息。相反，您需要生成具有无法执行的事务的 GTID 的空事务。要找出失败的 GTID，请检查*SHOW REPLICA STATUS*的`Retrieved_Gtid_Set`和`Executed_Gtid_Set`字段：

```
mysql> `SHOW REPLICA STATUS\G`
*************************** 1\. row ***************************
...
           Retrieved_Gtid_Set: de7e85f9-1060-11eb-8b8f-98af65266957:1-5
            Executed_Gtid_Set: de7e85f9-1060-11eb-8b8f-98af65266957:1-4,
de8d356e-1060-11eb-a568-98af65266957:1-3
...
```

在此示例中，`Retrieved_Gtid_Set`包含事务`de7e85f9-1060-11eb-8b8f-98af65266957:1-5`，而`Executed_Gtid_Set`仅包含事务`de7e85f9-1060-11eb-8b8f-98af65266957:1-4`。很明显，事务`de7e85f9-1060-11eb-8b8f-98af65266957:5`未被执行。带有 UUID`de8d356e-1060-11eb-a568-98af65266957`的事务是本地事务，不是由复制应用程序线程执行。

您还可以在`replication_applier_status_by_worker`表的`APPLYING_TRANSACTION`字段中找到失败的事务：

```
mysql> `select LAST_APPLIED_TRANSACTION, APPLYING_TRANSACTION`
    -> `from performance_schema.replication_applier_status_by_worker\G`
*************************** 1\. row ***************************
LAST_APPLIED_TRANSACTION: de7e85f9-1060-11eb-8b8f-98af65266957:4
    APPLYING_TRANSACTION: de7e85f9-1060-11eb-8b8f-98af65266957:5
1 row in set (0.00 sec)
```

一旦找到失败的事务，插入相同的 GTID 为空的事务，并重新启动 SQL 线程。

```
mysql> `-- set explicit GTID`
mysql> `SET gtid_next='de7e85f9-1060-11eb-8b8f-98af65266957:5';`
Query OK, 0 rows affected (0.00 sec)

mysql> `-- inject empty transaction`
mysql> `BEGIN;COMMIT;`
Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

mysql> `-- revert GTID generation back to automatic`
mysql> `SET gtid_next='automatic';`
Query OK, 0 rows affected (0.00 sec)

mysql> `-- restart SQL thread`
mysql> `START REPLICA SQL_THREAD;`
Query OK, 0 rows affected (0.01 sec)
```

###### 警告

虽然跳过二进制日志事件或事务有助于重新启动复制，但可能会导致更大的问题，并导致源和副本之间的数据不一致，从而导致将来的错误。始终分析为什么首次发生错误并尝试修复原因，而不仅仅是跳过事件。

虽然*SHOW REPLICA STATUS*和表`replication_applier_status_by_worker`都存储错误消息，如果使用多线程副本，则该表可以提供更详细的信息。例如，在此示例中，错误消息无法完全理解失败的原因：

```
mysql> `SHOW REPLICA STATUS\G`
*************************** 1\. row ***************************
...
               Last_SQL_Errno: 1146
               Last_SQL_Error: Coordinator stopped because there were error(s) ↩
                               in the worker(s). The most recent failure being: ↩
                               Worker 8 failed executing transaction ↩
                               'de7e85f9-1060-11eb-8b8f-98af65266957:7' at ↩
                               master log binlog.000001, end_log_pos 1818\. ↩
                               See error log and/or performance_schema.↩
                               replication_applier_status_by_worker table ↩
                               for more details about this failure or others, if any.
...
```

报告工作者 8 失败，但未说明原因。查询`replication_applier_status_by_worker`表的信息如下：

```
mysql> `select SERVICE_STATE, LAST_ERROR_NUMBER, LAST_ERROR_MESSAGE, LAST_ERROR_TIMESTAMP`
    -> `from performance_schema.replication_applier_status_by_worker where worker_id=8\G`
*************************** 1\. row ***************************
       SERVICE_STATE: OFF
   LAST_ERROR_NUMBER: 1146
  LAST_ERROR_MESSAGE: Worker 8 failed executing transaction ↩
                     'de7e85f9-1060-11eb-8b8f-98af65266957:7' at master log↩
                     binlog.000001, end_log_pos 1818; Error executing row event: ↩
                     'Table 'cookbook.limbs' doesn't exist'
LAST_ERROR_TIMESTAMP: 2020-10-17 14:28:01.144521
1 row in set (0.00 sec)
```

现在很明显，特定的表不存在。您可以分析原因并修正错误。

### 组复制故障排除

*SHOW REPLICA STATUS*对于组复制不可用。因此，您需要使用性能模式来解决与其相关的问题。性能模式仅针对组复制有两个特殊表：`replication_group_members`，显示所有成员的详细信息和`replication_group_member_stats`，显示它们的统计信息。但是，这些表没有关于 IO 和 SQL 线程错误的信息。我们讨论过的标准异步复制有这些详细信息。

让我们更仔细地看一看组复制的故障排除选项。

快速识别组复制中是否出现问题的方法是检查`replication_group_members`表。

```
mysql> `SELECT * FROM performance_schema.replication_group_members\G`
*************************** 1\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: de5b65cb-16ae-11eb-826c-98af65266957
   MEMBER_HOST: Delly-7390
   MEMBER_PORT: 33361
  MEMBER_STATE: ONLINE
   MEMBER_ROLE: PRIMARY
MEMBER_VERSION: 8.0.21
*************************** 2\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: e9514d63-16ae-11eb-8f6e-98af65266957
   MEMBER_HOST: Delly-7390
   MEMBER_PORT: 33362
  MEMBER_STATE: RECOVERING
   MEMBER_ROLE: SECONDARY
MEMBER_VERSION: 8.0.21
*************************** 3\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: f1e717ab-16ae-11eb-bfd2-98af65266957
   MEMBER_HOST: Delly-7390
   MEMBER_PORT: 33363
  MEMBER_STATE: RECOVERING
   MEMBER_ROLE: SECONDARY
MEMBER_VERSION: 8.0.21
3 rows in set (0.00 sec)
```

在上述列表中，只有`PRIMARY`成员处于`MEMBER_STATE: ONLINE`状态，这意味着它很健康。两个`SECONDARY`成员都处于`RECOVERING`状态，并且在加入组时遇到了问题。

失败的成员将在一段时间内保持`RECOVERING`状态，而组复制试图自我恢复。如果错误无法自动恢复，则离开该组并保持`ERROR`状态。

```
mysql> `SELECT * FROM performance_schema.replication_group_members\G`
*************************** 1\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: e9514d63-16ae-11eb-8f6e-98af65266957
   MEMBER_HOST: Delly-7390
   MEMBER_PORT: 33362
  MEMBER_STATE: ERROR
   MEMBER_ROLE: 
MEMBER_VERSION: 8.0.21
1 row in set (0.00 sec)
```

两个清单都在组的同一次要成员上获取，但在它离开组后，仅报告自身为 Group Replication 成员，并且不显示有关其他成员的信息。

要查找失败的原因，您需要检查 `replication_connection_status` 和 `replication_applier_status_by_worker` 表。

在我们的示例中，成员 `e9514d63-16ae-11eb-8f6e-98af65266957` 因 SQL 错误而停止。您可以在 `replication_applier_status_by_worker` 表中找到错误详情：

```
mysql> `SELECT CHANNEL_NAME, LAST_ERROR_NUMBER,`
    -> `LAST_ERROR_MESSAGE, LAST_ERROR_TIMESTAMP,`
    -> `APPLYING_TRANSACTION` 
    -> `FROM performance_schema.replication_applier_status_by_worker\G`
*************************** 1\. row ***************************
        CHANNEL_NAME: group_replication_recovery
   LAST_ERROR_NUMBER: 3635
  LAST_ERROR_MESSAGE: The table in transaction de5b65cb-16ae-11eb-826c-98af65266957:15 ↩
                      does not comply with the requirements by an external plugin.
LAST_ERROR_TIMESTAMP: 2020-10-25 20:31:27.718638
APPLYING_TRANSACTION: de5b65cb-16ae-11eb-826c-98af65266957:15
*************************** 2\. row ***************************
        CHANNEL_NAME: group_replication_applier
   LAST_ERROR_NUMBER: 0
  LAST_ERROR_MESSAGE: 
LAST_ERROR_TIMESTAMP: 0000-00-00 00:00:00.000000
APPLYING_TRANSACTION: 
2 rows in set (0.00 sec)
```

错误消息表示事务中表的定义 `de5b65cb-16ae-11eb-826c-98af65266957:15` 与 Group Replication 插件不兼容。要查找原因，请参阅[Group Replication 要求和限制](https://dev.mysql.com/doc/refman/8.0/en/group-replication-requirements-and-limitations.html)，识别事务中使用的表并修复错误。

`replication_applier_status_by_worker` 表中的错误消息没有任何提示表明在事务中使用了哪个表。但是错误日志文件可能有。打开错误日志文件，搜索 `LAST_ERROR_TIMESTAMP` 和 `LAST_ERROR_NUMBER` 来识别错误，并检查前后行是否有更多信息。

```
2020-10-25T17:31:27.718600Z 71 [ERROR] [MY-011542] [Repl] Plugin group_replication↩
reported: 'Table al_winner does not have any PRIMARY KEY. This is not compatible↩
with Group Replication.'
2020-10-25T17:31:27.718644Z 71 [ERROR] [MY-010584] [Repl] Slave SQL for channel↩
'group_replication_recovery': The table in transaction↩
de5b65cb-16ae-11eb-826c-98af65266957:15 does not comply with the requirements↩
by an external plugin. Error_code: MY-003635
```

在这个示例中，上一行的错误消息包含表名：`al_winner`，不兼容 Group Replication 的原因是表没有主键。

要修复错误，您需要在 `PRIMARY` 和失败的 `SECONDARY` 节点上修复表定义。

首先，登录到 `PRIMARY` 节点，并添加代理主键：

```
mysql> `set sql_log_bin=0;`
Query OK, 0 rows affected (0.00 sec)

mysql> `alter table al_winner add id int not null auto_increment primary key;`
Query OK, 0 rows affected (0.09 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> `set sql_log_bin=1;`
Query OK, 0 rows affected (0.01 sec)
```

您需要禁用二进制日志记录，否则此更改将被复制到辅助成员，并且由于重复列名错误，复制将停止。

然后在辅助节点上运行相同命令以修复表定义，并重新启动 Group Replication。

```
mysql> `set global super_read_only=0;`
Query OK, 0 rows affected (0.00 sec)

mysql> `set sql_log_bin=0;`
Query OK, 0 rows affected (0.00 sec)

mysql> `alter table al_winner add id int not null auto_increment primary key;`
Query OK, 0 rows affected (0.09 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> `set sql_log_bin=1;`
Query OK, 0 rows affected (0.01 sec)

mysql> `stop group_replication;`
Query OK, 0 rows affected (1.02 sec)

mysql> `start group_replication;`
Query OK, 0 rows affected (3.22 sec)
```

如果节点在单主模式下运行，则需要首先禁用由 Group Replication 插件设置的 `super_read_only`。

一旦错误修复，节点将加入组，并将其状态报告为 `ONLINE`。

```
mysql> `SELECT * FROM performance_schema.replication_group_members\G`
*************************** 1\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: d8a706aa-16ee-11eb-ba5a-98af65266957
   MEMBER_HOST: Delly-7390
   MEMBER_PORT: 33361
  MEMBER_STATE: ONLINE
   MEMBER_ROLE: PRIMARY
MEMBER_VERSION: 8.0.21
*************************** 2\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: e14043d7-16ee-11eb-b77a-98af65266957
   MEMBER_HOST: Delly-7390
   MEMBER_PORT: 33362
  MEMBER_STATE: ONLINE
   MEMBER_ROLE: SECONDARY
MEMBER_VERSION: 8.0.21
2 rows in set (0.00 sec)
```

###### 提示

您可以通过运行带有选项 `verbose` 的 mysqlbinlog 命令来查找失败的事务正在做什么：

```
$ `mysqlbinlog data1/binlog.000001` 
> `--include-gtids=de5b65cb-16ae-11eb-826c-98af65266957:15 --verbose`
...
SET @@SESSION.GTID_NEXT= 'de5b65cb-16ae-11eb-826c-98af65266957:15'/*!*/;
# at 4015
#201025 13:44:34 server id 1  end_log_pos 4094 CRC32 0xad05e64e 	Query ↩
thread_id=10	exec_time=0	error_code=0
SET TIMESTAMP=1603622674/*!*/;
...
### INSERT INTO `cookbook`.`al_winner`
### SET
###   @1='Mulder, Mark' /* STRING(120) meta=65144 nullable=1 is_null=0 */
###   @2=21 /* INT meta=0 nullable=1 is_null=0 */
### INSERT INTO `cookbook`.`al_winner`
### SET
###   @1='Clemens, Roger' /* STRING(120) meta=65144 nullable=1 is_null=0 */
###   @2=20 /* INT meta=0 nullable=1 is_null=0 */
### INSERT INTO `cookbook`.`al_winner`
...
### INSERT INTO `cookbook`.`al_winner`
### SET
###   @1='Sele, Aaron' /* STRING(120) meta=65144 nullable=1 is_null=0 */
###   @2=15 /* INT meta=0 nullable=1 is_null=0 */
# at 4469
#201025 13:44:34 server id 1  end_log_pos 4500 CRC32 0xddd32d63 	Xid = 74
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

解码行事件需要选项 `verbose`。

我们在一个节点上修复了错误，但第三个节点没有加入组。在检查表 `performance_schema.replication_connection_status` 内容后，我们发现复制连接选项未正确设置：

```
mysql> `SELECT CHANNEL_NAME, LAST_ERROR_NUMBER, LAST_ERROR_MESSAGE, LAST_ERROR_TIMESTAMP`
    -> `FROM performance_schema.replication_connection_status\G`
*************************** 1\. row ***************************
        CHANNEL_NAME: group_replication_applier
   LAST_ERROR_NUMBER: 0
  LAST_ERROR_MESSAGE: 
LAST_ERROR_TIMESTAMP: 0000-00-00 00:00:00.000000
*************************** 2\. row ***************************
        CHANNEL_NAME: group_replication_recovery
   LAST_ERROR_NUMBER: 13117
  LAST_ERROR_MESSAGE: Fatal error: Invalid (empty) username when attempting ↩
                      to connect to the master server. Connection attempt terminated.
LAST_ERROR_TIMESTAMP: 2020-10-25 21:31:31.413876
2 rows in set (0.00 sec)
```

要解决此问题，我们需要运行正确的 *CHANGE REPLICATION SOURCE* 命令：

```
mysql> `STOP GROUP_REPLICATION;`
Query OK, 0 rows affected (1.01 sec)

mysql> `CHANGE REPLICATION SOURCE TO SOURCE_USER='repl', SOURCE_PASSWORD='replrepl'`
    -> `FOR CHANNEL 'group_replication_recovery';`
Query OK, 0 rows affected, 2 warnings (0.03 sec)

mysql> `START GROUP_REPLICATION;`
Query OK, 0 rows affected (2.40 sec)
```

修复后，节点将因与先前相同的 SQL 错误而失败，必须按照我们上面描述的方式进行修复。最终，在 SQL 错误恢复后，节点将加入集群并显示为 `ONLINE`。

```
mysql> `SELECT * FROM performance_schema.replication_group_members\G`
*************************** 1\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: d8a706aa-16ee-11eb-ba5a-98af65266957
   MEMBER_HOST: Delly-7390
   MEMBER_PORT: 33361
  MEMBER_STATE: ONLINE
   MEMBER_ROLE: PRIMARY
MEMBER_VERSION: 8.0.21
*************************** 2\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: e14043d7-16ee-11eb-b77a-98af65266957
   MEMBER_HOST: Delly-7390
   MEMBER_PORT: 33362
  MEMBER_STATE: ONLINE
   MEMBER_ROLE: SECONDARY
MEMBER_VERSION: 8.0.21
*************************** 3\. row ***************************
  CHANNEL_NAME: group_replication_applier
     MEMBER_ID: ea775284-16ee-11eb-8762-98af65266957
   MEMBER_HOST: Delly-7390
   MEMBER_PORT: 33363
  MEMBER_STATE: ONLINE
   MEMBER_ROLE: SECONDARY
MEMBER_VERSION: 8.0.21
3 rows in set (0.00 sec)
```

要检查 Group Replication 查询表 `performance_schema.replication_group_member_stats` 的性能。

```
mysql> `SELECT * FROM performance_schema.replication_group_member_stats\G`
*************************** 1\. row ***************************
                              CHANNEL_NAME: group_replication_applier
                                   VIEW_ID: 16036502905383892:9
                                 MEMBER_ID: d8a706aa-16ee-11eb-ba5a-98af65266957
               COUNT_TRANSACTIONS_IN_QUEUE: 0
                COUNT_TRANSACTIONS_CHECKED: 10154
                  COUNT_CONFLICTS_DETECTED: 0
        COUNT_TRANSACTIONS_ROWS_VALIDATING: 9247
        TRANSACTIONS_COMMITTED_ALL_MEMBERS: d8a706aa-16ee-11eb-ba5a-98af65266957:1-18,
dc527338-13d1-11eb-abf7-98af65266957:1-1588
            LAST_CONFLICT_FREE_TRANSACTION: dc527338-13d1-11eb-abf7-98af65266957:10160
COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE: 0
         COUNT_TRANSACTIONS_REMOTE_APPLIED: 5
         COUNT_TRANSACTIONS_LOCAL_PROPOSED: 10154
         COUNT_TRANSACTIONS_LOCAL_ROLLBACK: 0
*************************** 2\. row ***************************
                              CHANNEL_NAME: group_replication_applier
                                   VIEW_ID: 16036502905383892:9
                                 MEMBER_ID: e14043d7-16ee-11eb-b77a-98af65266957
               COUNT_TRANSACTIONS_IN_QUEUE: 0
                COUNT_TRANSACTIONS_CHECKED: 10037
                  COUNT_CONFLICTS_DETECTED: 0
        COUNT_TRANSACTIONS_ROWS_VALIDATING: 9218
        TRANSACTIONS_COMMITTED_ALL_MEMBERS: d8a706aa-16ee-11eb-ba5a-98af65266957:1-18,
dc527338-13d1-11eb-abf7-98af65266957:1-1588
            LAST_CONFLICT_FREE_TRANSACTION: dc527338-13d1-11eb-abf7-98af65266957:8030
COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE: 5859
         COUNT_TRANSACTIONS_REMOTE_APPLIED: 4180
         COUNT_TRANSACTIONS_LOCAL_PROPOSED: 0
         COUNT_TRANSACTIONS_LOCAL_ROLLBACK: 0
*************************** 3\. row ***************************
                              CHANNEL_NAME: group_replication_applier
                                   VIEW_ID: 16036502905383892:9
                                 MEMBER_ID: ea775284-16ee-11eb-8762-98af65266957
               COUNT_TRANSACTIONS_IN_QUEUE: 0
                COUNT_TRANSACTIONS_CHECKED: 10037
                  COUNT_CONFLICTS_DETECTED: 0
        COUNT_TRANSACTIONS_ROWS_VALIDATING: 9218
        TRANSACTIONS_COMMITTED_ALL_MEMBERS: d8a706aa-16ee-11eb-ba5a-98af65266957:1-18,
dc527338-13d1-11eb-abf7-98af65266957:1-37
            LAST_CONFLICT_FREE_TRANSACTION: dc527338-13d1-11eb-abf7-98af65266957:6581
COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE: 5828
         COUNT_TRANSACTIONS_REMOTE_APPLIED: 4209
         COUNT_TRANSACTIONS_LOCAL_PROPOSED: 0
         COUNT_TRANSACTIONS_LOCAL_ROLLBACK: 0
3 rows in set (0.00 sec)
```

重要的字段是`COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE`，显示在辅助节点队列中等待应用的事务数量，以及`TRANSACTIONS_COMMITTED_ALL_MEMBERS`，显示所有成员上已应用的事务数量。有关更多详细信息，请参阅[用户参考手册](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-replication-group-member-stats-table.html)。

# 3.16 使用进程列表了解复制性能

## 问题

复制品落后于源服务器，延迟正在增加。您想了解发生了什么。

## 解决方案

使用性能模式中的复制表以及常规的 MySQL 性能工具检查 SQL 线程的状态。

## 讨论

如果 SQL 线程在应用更新时比源服务器慢，则副本可能落后于源。这可能是因为源上的更新正在并发运行，而在副本上使用的线程少于处理相同工作量的源服务器上的活动线程。即使在具有与源相同或更高 CPU 核心数量的副本上，这种差异也可能发生，因为您设置了比源服务器上活动线程少的`replica_parallel_workers`，或者因为由于安全措施未充分利用而未完全使用它们以防止副本以错误的顺序应用更新。

要了解活动的并行工作者数量，您可以查询`replication_applier_status_by_worker`表。

```
mysql> `SELECT WORKER_ID, LAST_APPLIED_TRANSACTION, APPLYING_TRANSACTION` 
    -> `FROM performance_schema.replication_applier_status_by_worker;`
+-----------+---------------------------------+---------------------------------+
| WORKER_ID | LAST_APPLIED_TRANSACTION        | APPLYING_TRANSACTION            |
+-----------+---------------------------------+---------------------------------+
|         1 | de7e85f9-...-98af65266957:26075 | de7e85f9-...-98af65266957:26077 |
|         2 | de7e85f9-...-98af65266957:26076 | de7e85f9-...-98af65266957:26078 |
|         3 | de7e85f9-...-98af65266957:26068 | de7e85f9-...-98af65266957:26079 |
|         4 | de7e85f9-...-98af65266957:26069 |                                 |
|         5 | de7e85f9-...-98af65266957:26070 |                                 |
|         6 | de7e85f9-...-98af65266957:26071 |                                 |
|         7 | de7e85f9-...-98af65266957:25931 |                                 |
|         8 | de7e85f9-...-98af65266957:21638 |                                 |
+-----------+---------------------------------+---------------------------------+
8 rows in set (0.01 sec)
```

在上面的列表中，您可能注意到目前只有三个线程正在应用事务，而其他线程处于空闲状态。这不是稳定的信息，您需要多次运行相同的查询以确定这是否是一种趋势。

性能模式中的`threads`表包含当前在 MySQL 服务器上运行的所有线程的列表，包括后台线程。它具有一个名为`name`的字段，其值为`thread/sql/replica_worker`（在复制 SQL 线程的情况下为`thread/sql/slave_worker`）。您可以查询它并找到每个 SQL 线程工作者正在执行的详细信息。

```
mysql> `SELECT THREAD_ID AS TID, PROCESSLIST_ID AS PID,`
    -> `PROCESSLIST_DB, PROCESSLIST_STATE`
    -> `FROM performance_schema.threads WHERE NAME = 'thread/sql/replica_worker';`
+-----+-----+----------------+----------------------------------------+
| TID | PID | PROCESSLIST_DB | PROCESSLIST_STATE                      |
+-----+-----+----------------+----------------------------------------+
|  54 |  13 | NULL           | waiting for handler commit             |
|  55 |  14 | sbtest         | Applying batch of row changes (update) |
|  56 |  15 | sbtest         | Applying batch of row changes (delete) |
|  57 |  16 | NULL           | Waiting for an event from Coordinator  |
|  58 |  17 | NULL           | Waiting for an event from Coordinator  |
|  59 |  18 | NULL           | Waiting for an event from Coordinator  |
|  60 |  19 | NULL           | Waiting for an event from Coordinator  |
|  61 |  20 | NULL           | Waiting for an event from Coordinator  |
+-----+-----+----------------+----------------------------------------+
8 rows in set (0.00 sec)
```

在上面的列表中，线程 54 正在等待事务提交，线程 55 和 56 正在应用一批行更改，而其他线程则在等待来自协调器的事件。

由于源服务器在大量线程中应用更改，我们可能会注意到复制延迟正在增加。

```
mysql> `\P grep Seconds_Behind_Source`
PAGER set to 'grep Seconds_Behind_Source'
mysql> `SHOW REPLICA STATUS\G SELECT SLEEP(60); SHOW REPLICA STATUS\G`
        Seconds_Behind_Source: 232
1 row in set (0.00 sec)

1 row in set (1 min 0.00 sec)

        Seconds_Behind_Source: 238
1 row in set (0.00 sec)
```

对于此类问题的一个解决方案是在源服务器上设置选项`binlog_transaction_dependency_tracking`为`WRITESET_SESSION`或`WRITESET`。这些选项在配方 3.8 中讨论，并允许在副本上实现更高的并行化。请注意，更改不会立即生效，因为副本将必须应用使用默认的`binlog_transaction_dependency_tracking`值`COMMIT_ORDER`记录的二进制日志事件。

然而，过了一段时间，您可能会注意到所有 SQL 线程工作者都变得活跃，并且复制延迟开始减少。

```
mysql> `SELECT WORKER_ID, LAST_APPLIED_TRANSACTION, APPLYING_TRANSACTION`
    -> `FROM performance_schema.replication_applier_status_by_worker;`
+-----------+----------------------------------+-----------------------------------+
| WORKER_ID | LAST_APPLIED_TRANSACTION         | APPLYING_TRANSACTION                        |
+-----------+----------------------------------+-----------------------------------+
|         1 | de7e85f9-...-98af65266957:170966 | de7e85f9-...-98af65266957:170976 |
|         2 | de7e85f9-...-98af65266957:170970 | de7e85f9-...-98af65266957:170973 |
|         3 | de7e85f9-...-98af65266957:170968 | de7e85f9-...-98af65266957:170975 |
|         4 | de7e85f9-...-98af65266957:170960 | de7e85f9-...-98af65266957:170967 |
|         5 | de7e85f9-...-98af65266957:170964 | de7e85f9-...-98af65266957:170972 |
|         6 | de7e85f9-...-98af65266957:170962 | de7e85f9-...-98af65266957:170969 |
|         7 | de7e85f9-...-98af65266957:170971 | de7e85f9-...-98af65266957:170977 |
|         8 | de7e85f9-...-98af65266957:170965 | de7e85f9-...-98af65266957:170974 |
+-----------+----------------------------------+-----------------------------------+
8 rows in set (0.00 sec)

mysql> `SELECT THREAD_ID, PROCESSLIST_ID, PROCESSLIST_DB, PROCESSLIST_STATE`
    -> `FROM performance_schema.threads WHERE NAME = 'thread/sql/replica_worker';`
+-----------+----------------+----------------+----------------------------------------+
| thread_id | PROCESSLIST_ID | PROCESSLIST_DB | PROCESSLIST_STATE                      |
+-----------+----------------+----------------+----------------------------------------+
|        54 |             13 | sbtest         | Applying batch of row changes (update) |
|        55 |             14 | NULL           | waiting for handler commit             |
|        56 |             15 | sbtest         | Applying batch of row changes (delete) |
|        57 |             16 | sbtest         | Applying batch of row changes (delete) |
|        58 |             17 | sbtest         | Applying batch of row changes (update) |
|        59 |             18 | sbtest         | Applying batch of row changes (delete) |
|        60 |             19 | sbtest         | Applying batch of row changes (update) |
|        61 |             20 | sbtest         | Applying batch of row changes (write)  |
+-----------+----------------+----------------+----------------------------------------+
8 rows in set (0.00 sec)

mysql> `\P grep Seconds_Behind_Source`
PAGER set to 'grep Seconds_Behind_Source'
mysql> `SHOW REPLICATION SOURCE STATUS\G SELECT SLEEP(60); SHOW REPLICA STATUS\G`
        Seconds_Behind_Source: 285
1 row in set (0.00 sec)

1 row in set (1 min 0.00 sec)

        Seconds_Behind_Source: 275
1 row in set (0.00 sec)
```

复制滞后的另一个常见原因是本地命令，影响由复制更新的表。如果查询 `replication_applier_status_by_worker` 表，并将 `APPLYING_TRANSACTION_START_APPLY_TIMESTAMP` 字段的值与当前时间进行比较，则可能会注意到这种情况发生。

```
mysql> `SELECT WORKER_ID, APPLYING_TRANSACTION, TIMEDIFF(NOW(),`
    -> `APPLYING_TRANSACTION_START_APPLY_TIMESTAMP) AS exec_time`
    -> `FROM performance_schema.replication_applier_status_by_worker;`
+-----------+---------------------------------------------+-----------------+
| WORKER_ID | APPLYING_TRANSACTION                        | exec_time       |
+-----------+---------------------------------------------+-----------------+
|         1 | de7e85f9-1060-11eb-8b8f-98af65266957:226091 | 00:05:14.367275 |
|         2 | de7e85f9-1060-11eb-8b8f-98af65266957:226087 | 00:05:14.768701 |
|         3 | de7e85f9-1060-11eb-8b8f-98af65266957:226090 | 00:05:14.501099 |
|         4 | de7e85f9-1060-11eb-8b8f-98af65266957:226097 | 00:05:14.232062 |
|         5 | de7e85f9-1060-11eb-8b8f-98af65266957:226086 | 00:05:14.773958 |
|         6 | de7e85f9-1060-11eb-8b8f-98af65266957:226083 | 00:05:14.782274 |
|         7 | de7e85f9-1060-11eb-8b8f-98af65266957:226080 | 00:05:14.843808 |
|         8 | de7e85f9-1060-11eb-8b8f-98af65266957:226094 | 00:05:14.327028 |
+-----------+---------------------------------------------+-----------------+
8 rows in set (0.00 sec)
```

在上面的列表中，所有线程的事务执行时间都相似，大约为五分钟。这太长了！

要找出事务执行时间如此之长的原因，请查询 Performance Schema 中的 `threads` 表：

```
mysql> `SELECT THREAD_ID, PROCESSLIST_ID, PROCESSLIST_DB, PROCESSLIST_STATE`
    -> `FROM performance_schema.threads WHERE NAME = 'thread/sql/replica_worker';`
+-----------+----------------+----------------+------------------------------+
| thread_id | PROCESSLIST_ID | PROCESSLIST_DB | PROCESSLIST_STATE            |
+-----------+----------------+----------------+------------------------------+
|        54 |             13 | NULL           | Waiting for global read lock |
|        55 |             14 | NULL           | Waiting for global read lock |
|        56 |             15 | NULL           | Waiting for global read lock |
|        57 |             16 | NULL           | Waiting for global read lock |
|        58 |             17 | NULL           | Waiting for global read lock |
|        59 |             18 | NULL           | Waiting for global read lock |
|        60 |             19 | NULL           | Waiting for global read lock |
|        61 |             20 | NULL           | Waiting for global read lock |
+-----------+----------------+----------------+------------------------------+
8 rows in set (0.00 sec)
```

很明显，复制 SQL 线程并没有执行任何有用的工作，只是在等待全局读锁。

要找出哪个线程持有全局读锁，请再次查询 Performance Schema 中的 `threads` 表，但这次要过滤掉副本线程：

```
mysql> `SELECT THREAD_ID, PROCESSLIST_ID, PROCESSLIST_DB,` 
    -> `PROCESSLIST_STATE, PROCESSLIST_INFO`
    -> `FROM performance_schema.threads`
    -> `WHERE NAME != 'thread/sql/replica_worker' AND PROCESSLIST_ID IS NOT NULL\G`
*************************** 1\. row ***************************
        thread_id: 46
   PROCESSLIST_ID: 7
   PROCESSLIST_DB: NULL
PROCESSLIST_STATE: Waiting on empty queue 
 PROCESSLIST_INFO: NULL
*************************** 2\. row ***************************
        thread_id: 50
   PROCESSLIST_ID: 9
   PROCESSLIST_DB: NULL
PROCESSLIST_STATE: Suspending
 PROCESSLIST_INFO: NULL
*************************** 3\. row ***************************
        thread_id: 52
   PROCESSLIST_ID: 11
   PROCESSLIST_DB: NULL
PROCESSLIST_STATE: Waiting for master to send event 
 PROCESSLIST_INFO: NULL
*************************** 4\. row ***************************
        thread_id: 53
   PROCESSLIST_ID: 12
   PROCESSLIST_DB: NULL
PROCESSLIST_STATE: Waiting for slave workers to process their queues
 PROCESSLIST_INFO: NULL
*************************** 5\. row ***************************
        thread_id: 64
   PROCESSLIST_ID: 23
   PROCESSLIST_DB: performance_schema
PROCESSLIST_STATE: executing
 PROCESSLIST_INFO: SELECT THREAD_ID, PROCESSLIST_ID, PROCESSLIST_DB, PROCESSLIST_STATE, ↩
                   PROCESSLIST_INFO FROM performance_schema.threads WHERE ↩
                   NAME != 'thread/sql/slave_worker' AND PROCESSLIST_ID IS NOT NULL
*************************** 6\. row ***************************
        thread_id: 65
   PROCESSLIST_ID: 24
   PROCESSLIST_DB: NULL
PROCESSLIST_STATE: NULL
 PROCESSLIST_INFO: flush tables with read lock
6 rows in set (0.00 sec)
```

在我们的示例中，问题线程是执行 *FLUSH TABLES WITH READ LOCK* 的线程。这是由备份程序执行的常见安全锁。既然我们知道了副本停顿的原因，我们可以选择等待此任务完成或终止该线程。完成后，副本将继续执行更新。

## 参见

性能故障排除是一个较长的话题，本书不涵盖更多详细信息。有关故障排除的额外信息，请参阅 [MySQL 故障排除](https://www.oreilly.com/library/view/mysql-troubleshooting/9781449317836/)。

# 3.17 设置自动复制

## 问题

您想要设置复制，但不想手动配置它。

## 解决方案

使用 MySQL Shell 中提供的 MySQL Admin API（第二章）。

## 讨论

MySQL Shell 提供 MySQL Admin API，允许您自动化标准复制管理任务，例如使用一个或多个副本创建源服务器的 ReplicaSet，或使用 Group Replication 创建 InnoDB Cluster。

### InnoDB ReplicaSet

如果要自动化复制设置，请在 MySQL Shell 中使用 MySQL Admin API 和 InnoDB ReplicaSet。InnoDB ReplicaSet 允许您创建单主复制拓扑，以及任意数量的次要只读服务器。您稍后可以将其中一个次要服务器提升为主服务器。不支持多主设置、复制过滤器和自动故障转移。

首先需要准备服务器。确保：

1.  MySQL 版本为 8.0 或更新版本

1.  启用 GTID 选项 `gtid_mode` 和 `enforce_gtid_consistency`

1.  二进制日志格式为 `ROW`

1.  默认存储引擎为 InnoDB：设置选项 `default_storage_engine=InnoDB`

1.  并行复制相关选项：

    ```
    binlog_transaction_dependency_tracking=WRITESET
    replica_preserve_commit_order=ON
    replica_parallel_type=LOGICAL_CLOCK
    ```

###### 警告

如果您使用的是 Ubuntu 并且希望在本地机器上设置 ReplicaSet，请编辑 `/etc/hosts` 文件，并删除回环地址 `127.0.1.1` 或替换为 `127.0.0.1`。MySQL Shell 不支持除 `127.0.0.1` 之外的回环地址。

一旦服务器为复制做好准备，您可以使用 MySQL Shell 开始配置它们：

```
 MySQL  JS > `\c root@127.0.0.1:13000`
Creating a session to 'root@127.0.0.1:13000'
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 12
Server version: 8.0.28 MySQL Community Server - GPL
No default schema selected; type \use <schema> to set one.
 MySQL  127.0.0.1:13000 ssl  JS > `dba.configureReplicaSetInstance(`
                               -> `'root@127.0.0.1:13000', {clusterAdmin: "'repl'@'%'"})`
                               ->
Please provide the password for 'root@127.0.0.1:13000': 
Save password for 'root@127.0.0.1:13000'? [Y]es/[N]o/Ne[v]er (default No): 
Configuring local MySQL instance listening at port 13000 for use in an InnoDB ReplicaSet...

This instance reports its own address as Delly-7390:13000
Clients and other cluster members will communicate with it through↩
this address by default. If this is not correct,  ↩
the report_host MySQL system variable should be changed.
Password for new account: ********
Confirm password: ********

applierWorkerThreads will be set to the default value of 4.

The instance 'Delly-7390:13000' is valid to be used in an InnoDB ReplicaSet.
Cluster admin user 'repl'@'%' created.
The instance 'Delly-7390:13000' is already ready to be used in an InnoDB ReplicaSet.

Successfully enabled parallel appliers.
```

命令 *dba.configureReplicaSetInstance* 接受两个参数：用于连接到服务器的 URI 和配置选项。选项 `clusterAdmin` 指示创建一个复制用户。然后在提示时提供密码。

为 ReplicaSet 中的所有服务器重复配置步骤。指定相同的复制用户名和密码。

一旦所有实例配置完成，创建一个 ReplicaSet：

```
 MySQL  127.0.0.1:13000 ssl  JS > `var rs = dba.createReplicaSet("cookbook")`
 A new replicaset with instance 'Delly-7390:13000' will be created.

* Checking MySQL instance at Delly-7390:13000

This instance reports its own address as Delly-7390:13000
Delly-7390:13000: Instance configuration is suitable.

* Updating metadata...

ReplicaSet object successfully created for Delly-7390:13000.
Use rs.addInstance() to add more asynchronously replicated instances to this  ↩
replicaset and rs.status() to check its status.
```

命令 *dba.createReplicaSet* 创建命名 ReplicaSet 并返回 ReplicaSet 对象。将其保存到变量中以进行进一步管理。

在内部，它在 MySQL Shell 连接的实例中创建了一个描述 ReplicaSet 设置的 `mysql_innodb_cluster_metadata` 数据库和表。同时，这第一个实例设置为 PRIMARY ReplicaSet 成员。您可以通过运行命令 *rs.status()* 来检查它：

```
 MySQL  127.0.0.1:13000 ssl  JS > `rs.status()`
{
    "replicaSet": {
        "name": "cookbook", 
        "primary": "Delly-7390:13000", 
        "status": "AVAILABLE", 
        "statusText": "All instances available.", 
        "topology": {
            "Delly-7390:13000": {
                "address": "Delly-7390:13000", 
                "instanceRole": "PRIMARY", 
                "mode": "R/W", 
                "status": "ONLINE"
            }
        }, 
        "type": "ASYNC"
    }
}
```

一旦设置了 PRIMARY 实例，请添加尽可能多的次要实例：

```
 MySQL  127.0.0.1:13000 ssl  JS > `rs.addInstance('root@127.0.0.1:13002')`
Adding instance to the replicaset...

* Performing validation checks

This instance reports its own address as Delly-7390:13002
Delly-7390:13002: Instance configuration is suitable.

* Checking async replication topology...

* Checking transaction state of the instance...

NOTE: The target instance 'Delly-7390:13002' has not been pre-provisioned  ↩
(GTID set is empty). The Shell is unable to decide whether replication can  ↩
completely recover its state.
The safest and most convenient way to provision a new instance is through  ↩
automatic clone provisioning, which will completely overwrite the state of  ↩
'Delly-7390:13002' with a physical snapshot from an existing replicaset member.  ↩
To use this method by default, set the 'recoveryMethod' option to 'clone'.

WARNING: It should be safe to rely on replication to incrementally recover  ↩
the state of the new instance if you are sure all updates ever executed in  ↩
the replicaset were done with GTIDs enabled, there are no purged transactions  ↩ 
and the new instance contains the same GTID set as the replicaset or a subset  ↩
of it. To use this method by default, set the 'recoveryMethod' option to 'incremental'.

Please select a recovery method [C]lone/[I]ncremental recovery/[A]bort (default Clone): 
* Updating topology
Waiting for clone process of the new member to complete. Press ^C to abort the operation.
* Waiting for clone to finish...
NOTE: Delly-7390:13002 is being cloned from delly-7390:13000
** Stage DROP DATA: Completed
** Clone Transfer  
    FILE COPY  ########################################################  100%  Completed
    PAGE COPY  ########################################################  100%  Completed
    REDO COPY  ########################################################  100%  Completed

NOTE: Delly-7390:13002 is shutting down...

* Waiting for server restart... ready
* Delly-7390:13002 has restarted, waiting for clone to finish...
** Stage RESTART: Completed
* Clone process has finished: 60.00 MB transferred in about 1 second (~60.00 MB/s)

** Configuring Delly-7390:13002 to replicate from Delly-7390:13000
** Waiting for new instance to synchronize with PRIMARY...

The instance 'Delly-7390:13002' was added to the replicaset and is replicating 
from Delly-7390:13000.
```

每个次要实例从主要成员执行初始数据复制。它可以使用 `clone` 插件或从二进制日志进行增量恢复来复制数据。对于已经有数据的服务器，`clone` 方法是首选的。但您可能需要手动重新启动服务器以完成安装。如果选择增量恢复，请确保没有包含数据的二进制日志被清除。否则，复制设置将失败。

一旦添加了所有次要成员，ReplicaSet 就准备好可以用于写入和读取。您可以通过运行命令 *rs.status()* 来检查其状态。它支持选项 `extended`，控制输出的详细程度。但它不显示有关复制健康状况的所有信息。如果您希望获取所有细节，请使用 *SHOW REPLICA STATUS* 命令或查询性能模式。

如果您想要更改哪个服务器是 PRIMARY，请使用 *rs.setPrimaryInstance* 命令。因此，*rs.setPrimaryInstance(“127.0.0.1:13002”)* 将主服务器从运行在端口 13000 上的服务器切换到监听端口 13002 的服务器。

如果您从参与 ReplicaSet 的服务器断开连接或销毁了 `ReplicaSet` 对象，重新连接到 ReplicaSet 成员之一并运行命令 *rs=dba.getReplicaSet()* 来重新创建 ReplicaSet 对象。

###### 警告

如果您希望使用 MySQL Shell 管理 ReplicaSet，请不要直接通过运行 *CHANGE REPLICATION SOURCE* 命令修改复制设置。所有管理都应通过 MySQL Shell 中的 Admin API 进行。

### InnoDB Cluster

要自动化 Group Replication，请创建 [MySQL InnoDB Cluster](https://dev.mysql.com/doc/refman/8.0/en/mysql-innodb-cluster-introduction.html)。InnoDB Cluster 是一个完整的高可用解决方案，允许您轻松配置和管理至少三个 MySQL 服务器的组。

在设置 InnoDB Cluster 之前，请准备好服务器。组中的每个服务器都应具有：

1.  唯一的服务器 ID

1.  启用 GTID

1.  选项 `disabled_storage_engines` 设置为 `"MyISAM,BLACKHOLE,FEDERATED,ARCHIVE,MEMORY"`

1.  选项 `log_replica_updates` 已启用

1.  具有管理特权的用户帐户

1.  与并行复制相关的选项：

    ```
    binlog_transaction_dependency_tracking=WRITESET
    replica_preserve_commit_order=ON
    replica_parallel_type=LOGICAL_CLOCK
    transaction_write_set_extraction=XXHASH64
    ```

您可以设置其他选项（Recipe 3.12），用于组复制，但也可以通过 MySQL Shell 进行配置。

安装并启动 MySQL 实例后，将 MySQL Shell 连接到要设置为主实例的实例。您需要使用具有管理员权限的帐户（在我们的情况下是 `root`）启动配置过程。

```
 MySQL  127.0.0.1:33367 ssl  JS > `dba.configureInstance('root@127.0.0.1:33367',` 
                               -> `{clusterAdmin: "grepl",` 
                               -> `clusterAdminPassword: "greplgrepl"})`
                               ->
Please provide the password for 'root@127.0.0.1:33367': 
Configuring local MySQL instance listening at port 33367 for use in an InnoDB cluster...

This instance reports its own address as Delly-7390:33367
Clients and other cluster members will communicate with it through this address by default. 
If this is not correct, the report_host MySQL system variable should be changed.
Assuming full account name 'grepl'@'%' for grepl

The instance 'Delly-7390:33367' is valid to be used in an InnoDB cluster.

Cluster admin user 'grepl'@'%' created.
The instance 'Delly-7390:33367' is already ready to be used in an InnoDB cluster.
```

为集群中的其他实例重复配置。

###### 警告

如果某个实例已手动配置为组复制，MySQL Shell 将无法更新其选项，并且不能保证组复制配置在重启后持续存在。始终在设置 InnoDB Cluster 之前运行 *dba.configureInstance*。

在配置实例后，创建集群：

```
 MySQL  127.0.0.1:33367 ssl  JS > `var cluster = dba.createCluster('cookbook',` 
                               -> `{localAddress: ":34367"})`
                               ->
A new InnoDB cluster will be created on instance '127.0.0.1:33367'.

Validating instance configuration at 127.0.0.1:33367...

This instance reports its own address as Delly-7390:33367

Instance configuration is suitable.
Creating InnoDB cluster 'cookbook' on 'Delly-7390:33367'...

Adding Seed Instance...
Cluster successfully created. Use Cluster.addInstance() to add MySQL instances.
At least 3 instances are needed for the cluster to be able to withstand up to
one server failure.
```

然后将实例添加到集群中：*cluster.addInstance('root@127.0.0.1:33368', {localAddress: “:34368"})*。当 MySQL Shell 要求您选择恢复方法时，请选择“克隆”。然后，根据您的服务器是否支持 *RESTART* 命令，等待其恢复在线或手动启动节点。成功后，您将看到类似以下的消息：

```
State recovery already finished for 'Delly-7390:33368'

The instance '127.0.0.1:33368' was successfully added to the cluster.
```

将其他实例添加到集群中。

###### 小贴士

MySQL Shell 构建了一个本地地址，组节点使用此地址通过系统变量 `report_host`（主机地址）和公式 `(当前实例端口) * 10 + 1`（端口号）进行通信。如果自动生成的值超过 65535，则实例无法添加到集群中。因此，如果使用非标准端口，请为选项 `localAddress` 指定自定义值。

添加实例后，InnoDB Cluster 已准备就绪。要查看其状态，请使用 *cluster.status()* 命令，支持 `extended` 键，控制输出的详细程度。默认为 0：仅打印基本信息。通过选项 2 和 3，您可以查看每个成员接收和应用的事务。命令 *cluster.describe()* 给出集群拓扑的简要概述。

```
 MySQL  127.0.0.1:33367 ssl  JS > `cluster.describe()`
{
    "clusterName": "cookbook", 
    "defaultReplicaSet": {
        "name": "default", 
        "topology": [
            {
                "address": "Delly-7390:33367", 
                "label": "Delly-7390:33367", 
                "role": "HA"
            }, 
            {
                "address": "Delly-7390:33368", 
                "label": "Delly-7390:33368", 
                "role": "HA"
            }, 
            {
                "address": "Delly-7390:33369", 
                "label": "Delly-7390:33369", 
                "role": "HA"
            }
        ], 
        "topologyMode": "Single-Primary"
    }
}
```

如果您销毁了 Cluster 对象（例如通过关闭会话），请重新连接到集群成员之一，并通过运行命令 *cluster = dba.getCluster()* 重新创建它。

###### 注意

InnoDB ReplicaSet 和 InnoDB Cluster 都支持软件路由器 [MySQL Router](https://dev.mysql.com/doc/mysql-router/8.0/en/)，您可以用它进行负载均衡。我们跳过了此部分，因为它超出了本书的范围。有关如何与 InnoDB ReplicaSet 和 InnoDB Cluster 设置 MySQL Router 的信息，请参阅用户参考手册。

## 另请参阅

关于复制自动化的更多信息，请参阅 [MySQL Shell 用户参考手册](https://dev.mysql.com/doc/mysql-shell/8.0/en/)。
