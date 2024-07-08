# 第六章：表管理

# 6.0 引言

本章涵盖与创建和填充表相关的主题，包括：

+   克隆表

+   从一张表复制到另一张表

+   使用临时表

+   生成唯一表名

+   确定表使用的存储引擎或将其从一种存储引擎转换为另一种

本章中的许多示例使用名为`mail`的表，其中包含跟踪一组主机上用户之间邮件消息流量的行（参见 Recipe 5.0）。要创建并加载此表，请进入`recipes`分发的*tables*目录，并运行此命令：

```
$ `mysql cookbook < mail.sql`
```

# 6.1 克隆表

## 问题

您想创建一个与现有表完全相同结构的表。

## 解决方案

使用`CREATE` `TABLE` … `LIKE`来克隆表结构。如果还想将原始表的一些或全部行复制到新表中，请使用`INSERT` `INTO` … `SELECT`。

## 讨论

要创建与现有表完全相同的新表，请使用此语句：

```
CREATE TABLE *`new_table`* LIKE *`original_table`*;
```

新表的结构与原始表相同，但有几个例外：`CREATE` `TABLE` … `LIKE`不复制外键定义，也不复制表可能使用的任何`DATA` `DIRECTORY`或`INDEX` `DIRECTORY`表选项。

新表为空。如果还希望内容与原始表相同，请使用`INSERT` `INTO` … `SELECT`语句复制行：

```
INSERT INTO *`new_table`* SELECT * FROM *`original_table`*;
```

要复制表的部分内容，需要添加适当的`WHERE`子句，以确定要复制的行。例如，以下语句将创建一个名为`mail2`的`mail`表的副本，其中只包含由`barb`发送的邮件：

```
CREATE TABLE mail2 LIKE mail;
INSERT INTO mail2 SELECT * FROM mail WHERE srcuser = 'barb';
```

###### 警告

从大表中选择所有内容可能很慢，不建议在生产服务器上执行此操作。我们讨论如何复制大表在 Recipe 6.7 和 Recipe 6.8 中。

## 参见

有关`INSERT` … `SELECT`的更多信息，请参阅 Recipe 6.2。

# 6.2 将查询结果保存到表中

## 问题

您希望将`SELECT`语句的结果保存到表中，而不是显示它。

## 解决方案

如果表存在，则使用`INSERT` `INTO` … `SELECT`将行检索到其中。如果表不存在，则使用`CREATE` `TABLE` … `SELECT`动态创建它。

## 讨论

MySQL 服务器通常将`SELECT`语句的结果返回给执行该语句的客户端。例如，当您在*mysql*程序内执行语句时，服务器将结果返回给*mysql*，后者将其显示在屏幕上。可以将`SELECT`语句的结果保存到表中，这在多种情况下都很有用：

+   您可以轻松地创建表的完整或部分副本。如果您正在为修改表的应用程序开发算法，最好使用表的副本，以免担心错误的后果。如果原始表很大，创建部分副本可以加快开发过程，因为对其运行的查询时间较短。

+   基于可能存在格式不正确的信息进行数据加载操作时，将新行加载到测试临时表中，执行一些初步检查，并根据需要更正行。当您确信新行没有问题时，将它们从临时表复制到主表中。

+   一些应用程序维护一个大的存储库表和一个较小的工作表，定期向其中插入行，周期性地将工作表行复制到存储库，并清除工作表。

+   要更高效地对大表执行汇总操作，请避免反复在其上运行昂贵的汇总操作。相反，将汇总信息仅选择一次到第二表中，并在进一步分析时使用该表。

此处演示如何将结果集检索到表中。示例中的表名`src_tbl`和`dst_tbl`分别指源表（从中选择行）和目标表（将其存储进去）。

如果目标表已经存在，请使用`INSERT` … `SELECT`将结果集复制到其中。例如，如果`dst_tbl`包含整数列`i`和字符串列`s`，则以下语句将行从`src_tbl`复制到`dst_tbl`，将列`val`赋给`i`，将列`name`赋给`s`：

```
INSERT INTO dst_tbl (i, s) SELECT val, name FROM src_tbl;
```

要插入的列数必须与选定的列数相匹配，列之间的对应关系基于位置而不是名称。要复制所有列，可以简化语句如下：

```
INSERT INTO dst_tbl SELECT * FROM src_tbl;
```

要仅复制特定行，请添加一个`WHERE`子句来选择这些行：

```
INSERT INTO dst_tbl SELECT * FROM src_tbl
WHERE val > 100 AND name LIKE 'A%';
```

`SELECT`语句也可以从表达式中产生值。例如，以下语句计算`src_tbl`中每个名称出现的次数，并将计数和名称都存储在`dst_tbl`中：

```
INSERT INTO dst_tbl (i, s) SELECT COUNT(*), name
FROM src_tbl GROUP BY name;
```

如果目标表不存在，请先使用`CREATE` `TABLE`语句创建它，然后使用`INSERT` … `SELECT`将行复制到其中。或者，直接从`SELECT`的结果创建目标表，例如，要创建`dst_tbl`并将`src_tbl`的整个内容复制到其中，可以这样做：

```
CREATE TABLE dst_tbl SELECT * FROM src_tbl;
```

###### 警告

```
INSERT INTO ... SELECT ...
```

不会从源表复制索引。如果使用此语法且目标表应具有索引，请在语句完成后添加它们。我们在第 21.1 节中讨论索引。

MySQL 根据`src_tbl`中列的名称、数量和类型在`dst_tbl`中创建列。要仅复制特定行，请添加适当的`WHERE`子句。要创建一个空表，请使用选择不返回任何行的`WHERE`子句：

```
CREATE TABLE dst_tbl SELECT * FROM src_tbl WHERE FALSE;
```

要仅复制部分列，请在语句的`SELECT`部分中命名您想要的列。例如，如果`src_tbl`包含列`a`、`b`、`c`和`d`，则像这样仅复制`b`和`d`：

```
CREATE TABLE dst_tbl SELECT b, d FROM src_tbl;
```

要按与源表中出现顺序不同的顺序创建列，请按所需顺序命名它们。如果源表包含应以`c`、`a`、`b`顺序出现在目标表中的列，则执行以下操作：

```
CREATE TABLE dst_tbl SELECT c, a, b FROM src_tbl;
```

要在目标表中创建除源表中选定列之外的列，请在语句的`CREATE TABLE`部分提供适当的列定义。以下语句在`dst_tbl`中创建`id`作为`AUTO_INCREMENT`列，并从`src_tbl`添加列`a`、`b`和`c`：

```
CREATE TABLE dst_tbl
(
  id INT NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (id)
)
SELECT a, b, c FROM src_tbl;
```

结果表以`id`、`a`、`b`、`c`的顺序包含四列。已定义的列被分配它们的默认值。这意味着`id`作为`AUTO_INCREMENT`列，从 1 开始分配连续的序列号（见 Recipe 15.1）。

如果从表达式派生列的值，则其默认名称为表达式本身，稍后使用起来可能会很困难。在这种情况下，通过提供别名（见 Recipe 5.2）来为列命名更好是明智的选择。假设`src_tbl`包含列出每个发票中条目的发票信息。以下语句生成一个摘要，列出表中命名的每个发票及其条目的总费用，并使用表达式的别名：

```
CREATE TABLE dst_tbl
SELECT inv_no, SUM(unit_cost*quantity) AS total_cost
FROM src_tbl GROUP BY inv_no;
```

`CREATE TABLE … SELECT`非常方便，但由于结果集提供的信息不如`CREATE TABLE`语句中所能指定的那样全面，因此存在一些限制。例如，MySQL 不知道结果集列是否应该建立索引或其默认值是什么。如果在目标表中包含此信息很重要，请使用以下技术：

+   要使目标表成为源表的*精确*副本，请使用 Recipe 6.1 中描述的克隆技术。

+   要在目标表中包含索引，请明确指定它们。例如，如果`src_tbl`在`id`列上有一个`PRIMARY KEY`，以及在`state`和`city`上有一个多列索引，请在`dst_tbl`中也指定它们：

    ```
    CREATE TABLE dst_tbl (PRIMARY KEY (id), INDEX(state,city))
    SELECT * FROM src_tbl;
    ```

+   不会复制诸如`AUTO_INCREMENT`之类的列属性和列的默认值到目标表。要保留这些属性，请先创建表，然后使用`ALTER TABLE`来修改列定义。例如，如果`src_tbl`有一个`id`列不仅是`PRIMARY KEY`还是`AUTO_INCREMENT`列，请复制表并修改副本：

    ```
    CREATE TABLE dst_tbl (PRIMARY KEY (id)) SELECT * FROM src_tbl;
    ALTER TABLE dst_tbl MODIFY id INT UNSIGNED NOT NULL AUTO_INCREMENT;
    ```

# 6.3 创建临时表

## 问题

您仅需要一个表用于短时间，之后希望它自动消失。

## 解决方案

使用`TEMPORARY`关键字创建表，并让 MySQL 负责删除。

## 讨论

有些操作需要仅临时存在并在不再需要时应消失的表。当然，您可以显式执行`DROP` `TABLE`语句来在完成后移除表。另一个选择是使用`CREATE` `TEMPORARY` `TABLE`。此语句类似于`CREATE` `TABLE`，但创建一个在会话结束时自动消失的临时表，如果您还没有手动移除它的话。这种行为非常有用，因为 MySQL 会自动删除这个表；您不需要记住去做这件事。`TEMPORARY`可以与通常的表创建方法一起使用：

+   根据明确的列定义创建表：

    ```
    CREATE TEMPORARY TABLE *`tbl_name`* (*`...column definitions...`*);
    ```

+   根据现有表创建表：

    ```
    CREATE TEMPORARY TABLE *`new_table`* LIKE *`original_table`*;
    ```

+   根据结果集即时创建表：

    ```
    CREATE TEMPORARY TABLE *`tbl_name`* SELECT ... ;
    ```

临时表是会话特定的，因此多个客户端可以分别创建同名的临时表而不会互相干扰。这使得编写使用临时表的应用程序更加容易，因为您不需要确保每个客户端的表都具有唯一的名称。（有关表命名问题的进一步讨论，请参见 Recipe 6.4。）

临时表可以与永久表同名。在这种情况下，临时表<q>隐藏</q>了永久表，只要存在，就可以使其复制的表，并且可以在不意外影响原表的情况下进行修改。在以下示例中的`DELETE`语句从临时表`mail`中删除行，而不影响原始永久表：

```
mysql> `CREATE TEMPORARY TABLE mail SELECT * FROM mail;`
mysql> `SELECT COUNT(*) FROM mail;`
+----------+
| COUNT(*) |
+----------+
|       16 |
+----------+
mysql> `DELETE FROM mail;`
mysql> `SELECT COUNT(*) FROM mail;`
+----------+
| COUNT(*) |
+----------+
|        0 |
+----------+
mysql> `DROP TEMPORARY TABLE mail;`
mysql> `SELECT COUNT(*) FROM mail;`
+----------+
| COUNT(*) |
+----------+
|       16 |
+----------+
```

虽然使用`CREATE` `TEMPORARY` `TABLE`创建的临时表具有刚讨论的好处，但请记住以下注意事项：

+   要在给定会话中重新使用临时表，您必须在重新创建之前显式删除它。尝试使用相同名称创建第二个临时表将导致错误。

+   如果您修改一个临时表<q>隐藏</q>了同名永久表，请务必测试由于已断开连接导致的错误，特别是使用具有重新连接功能的编程接口时。如果客户端程序在检测到断开连接后自动重新连接，则在重新连接后修改将影响永久表，而不是临时表。

+   一些 API 支持持久连接或连接池。这些功能会阻止临时表在脚本结束时按照您的期望被删除，因为连接仍然保持开放以供其他脚本重用。您的脚本无法控制连接何时关闭。这意味着最好在创建临时表之前执行以下语句，以防它仍然存在于脚本的前一次执行中：

    ```
    DROP TEMPORARY TABLE IF EXISTS *`tbl_name`*;
    ```

    `TEMPORARY` 关键字在这里很有用，如果临时表已经被删除，可以避免删除任何与其同名的永久表。

# 6.4 生成唯一表名

## 问题

您需要创建一个表，其名称保证不存在。

## 解决方案

生成一个对您的客户端程序唯一的值，并将其合并到表名中。

## 讨论

MySQL 是一个多客户端数据库服务器，因此如果一个创建临时表的脚本可能同时被多个客户端调用，请注意多个脚本实例不要争夺相同的表名。如果脚本使用 `CREATE TEMPORARY TABLE` 创建表，那么不会有问题，因为不同客户端可以创建同名的临时表而不会发生冲突。

如果您不能或不想使用 `TEMPORARY` 表，请确保脚本的每次调用都创建一个唯一命名的表，并在不再需要时将其删除。为了实现这一点，将一些保证在每次调用中唯一的值合并到名称中。如果可能在时间戳分辨率内调用两个脚本实例，则时间戳将不起作用。随机数可能更好，但随机数只能减少名称冲突的可能性，而不能消除它。由函数 `UUID` 生成的值是唯一值的更好来源。函数 `UUID` 根据[RFC 4122，“通用唯一标识符 (UUID) URN 命名空间”](http://www.ietf.org/rfc/rfc4122.txt)生成一个 128 位字符串，这个字符串在空间和时间上都是唯一的。虽然此函数生成的值不一定是唯一的，但足以生成唯一的临时表名。

可以通过使用准备好的语句将 `UUID` 合并到 SQL 中的表名中来将 `UUID` 集成到表名中。以下示例说明了这一点，在 `CREATE TABLE` 语句中引用表名和预防性 `DROP TABLE` 语句中引用表名：

```
SET @tbl_name = CONCAT('tmp_tbl_', UUID());
SET @stmt = CONCAT('CREATE TABLE `', @tbl_name, '` (i INT)');
PREPARE stmt FROM @stmt;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

# 6.5 检查或更改表的存储引擎

## 问题

您想要检查表使用的存储引擎，以便确定哪些引擎功能适用。或者您需要更改表的存储引擎，因为意识到另一个引擎的功能更适合您使用表的方式。

## 解决方案

要确定表的存储引擎，可以使用几种语句之一。要更改表的引擎，请使用带有 `ENGINE` 子句的 `ALTER TABLE`。

## 讨论

MySQL 支持多个存储引擎，这些引擎具有不同的特性。例如，InnoDB 引擎支持事务，而 Memory 引擎不支持。如果需要知道表是否支持事务，请检查它使用的存储引擎。如果表的引擎不支持事务，可以将表转换为使用支持事务的引擎。

要确定表格的当前引擎，请检查`INFORMATION_SCHEMA`或使用`SHOW` `TABLE` `STATUS`或`SHOW` `CREATE` `TABLE`语句。对于`mail`表格，获取引擎信息如下：

```
mysql> `SELECT ENGINE FROM INFORMATION_SCHEMA.TABLES`
    -> `WHERE TABLE_SCHEMA = 'cookbook' AND TABLE_NAME = 'mail';`
+--------+
| ENGINE |
+--------+
| InnoDB |
+--------+

mysql> `SHOW TABLE STATUS LIKE 'mail'\G`
*************************** 1\. row ***************************
           Name: mail
         Engine: InnoDB
…

mysql> `SHOW CREATE TABLE mail\G`
*************************** 1\. row ***************************
       Table: mail
Create Table: CREATE TABLE `mail` (
*`... column definitions ...`*
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

要更改表格的存储引擎，请使用带有`ENGINE`说明符的`ALTER` `TABLE`。例如，要将`mail`表格转换为使用 Memory 存储引擎，请使用以下语句：

```
ALTER TABLE mail ENGINE = Memory;
```

请注意，将大表格转换为不同的存储引擎可能需要很长时间，并且在 CPU 和 I/O 活动方面可能很昂贵。

要确定您的 MySQL 服务器支持哪些存储引擎，请检查`SHOW` `ENGINES`语句的输出或查询`INFORMATION_SCHEMA` `ENGINES`表。

# 6.6 使用 mysqldump 复制表格

## 问题

您希望复制一个或多个表格，无论是在 MySQL 服务器管理的数据库之间，还是从一个服务器复制到另一个。

## 解决方案

使用*mysqldump*程序。

## 讨论

*mysqldump*程序创建一个备份文件，可以重新加载以重新创建原始表格或表格：

```
$ `mysqldump cookbook mail > mail.sql`
```

输出文件*mail.sql*包括一个`CREATE` `TABLE`语句来创建`mail`表格和一组`INSERT`语句来插入其行。如果原始表格丢失，您可以重新加载文件来重新创建表格：

```
$ `mysql cookbook < mail.sql`
```

此方法还可轻松处理表格的任何触发器。默认情况下，*mysqldump*将触发器写入转储文件，因此重新加载文件会将触发器与表格一起复制，无需特殊处理。

默认情况下，*mysqldump*在`CREATE TABLE`之前包括`DROP TABLE IF EXISTS`语句。如果您不希望在加载转储时删除表格，并且宁愿操作失败，请使用选项`--skip-add-drop-table`运行*mysqldump*。

除了恢复表格外，*mysqldump* 还可用于通过将输出重新加载到不同的数据库来复制它们。（如果目标数据库不存在，请先创建。）以下示例展示了一些有用的表格复制命令。

### 复制单个 MySQL 服务器内的表格

+   将单个表格复制到不同的数据库中：

    ```
    $ `mysqldump cookbook mail > mail.sql`
    $ `mysql other_db < mail.sql`
    ```

    要转储多个表格，请在数据库名称参数后命名它们。

+   复制数据库中的所有表格到另一个数据库：

    ```
    $ `mysqldump cookbook > cookbook.sql`
    $ `mysql other_db < cookbook.sql`
    ```

    当您在数据库名称后没有指定表格名称时，*mysqldump* 会将它们全部转储。要同时包括存储过程和事件，请在 *mysqldump* 命令中添加`--routines`和`--events`选项。（虽然还有`--triggers`选项，但因为*mysqldump*默认将触发器与其关联的表格一起转储，所以不需要。）

+   复制表格，并使用不同的名称进行复制：

    1.  转储表格：

        ```
        $ `mysqldump cookbook mail > mail.sql`
        ```

    1.  将表格重新加载到不包含具有该名称的表格的不同数据库中：

        ```
        $ `mysql other_db < mail.sql`
        ```

    1.  重命名表格：

        ```
        $ `mysql other_db`
        mysql> `RENAME mail TO mail2;`
        ```

        或者，同时将表格移动到另一个数据库中，请使用数据库名称限定新名称：

        ```
        $ `mysql other_db`
        mysql> `RENAME mail TO cookbook.mail2;`
        ```

若要在没有中介文件的情况下执行表格复制操作，请使用管道连接*mysqldump*和*mysql*命令：

```
$ `mysqldump cookbook mail | mysql other_db`
$ `mysqldump cookbook | mysql other_db`
```

###### 提示

您可以考虑使用较新的工具 *mysqlpump*，其工作方式类似于 *mysqldump*，但支持更智能的过滤器和并行处理。我们在 Recipe 13.13 中讨论了 *mysqlpump*。

### 在 MySQL 服务器之间复制表格

前述命令使用 *mysqldump* 在由单个 MySQL 服务器管理的数据库之间复制表格。*mysqldump* 的输出也可以用于从一个服务器复制表格到另一个服务器。假设您想要从本地主机的 `cookbook` 数据库复制 `mail` 表格到主机 *other-host.example.com* 上的 `other_db` 数据库。一种方法是将输出转储到一个文件：

```
$ `mysqldump cookbook mail > mail.sql`
```

然后将 *mail.sql* 复制到 *other-host.example.com*，在那里运行以下命令将表格加载到该 MySQL 服务器的 `other_db` 数据库中：

```
$ `mysql other_db < mail.sql`
```

要实现此目标而无需中间文件，请使用管道将 *mysqldump* 的输出直接通过网络发送到远程 MySQL 服务器。如果您可以从本地主机连接到两台服务器，请使用以下命令：

```
$ `mysqldump cookbook mail | mysql -h other-host.example.com other_db`
```

*mysqldump* 命令的前半部分连接到本地服务器并将转储输出写入管道。*mysql* 命令的后半部分连接到远程 MySQL 服务器 *other-host.example.com*。它从管道读取输入并将每个语句发送到 *other-host.example.com* 服务器。

如果无法直接使用本地主机上的 *mysql* 连接到远程服务器，请将转储输出发送到使用 *ssh* 在 *other-host.example.com* 上远程调用 *mysql* 的管道中：

```
$ `mysqldump cookbook mail | ssh other-host.example.com mysql other_db`
```

*ssh* 连接到 *other-host.example.com* 并在那里启动 *mysql*。然后，它从管道中读取 *mysqldump* 的输出并将其传递给远程的 *mysql* 进程。*ssh* 可以用来将转储通过网络发送到一个因防火墙而阻止 MySQL 端口连接但允许 SSH 端口连接的机器。

关于要复制哪些表格，适用于本地复制的类似原则同样适用。要通过网络复制多个表格，请在 *mysqldump* 命令的数据库参数后命名它们。要复制整个数据库，请在数据库名称后不指定任何表格名称；*mysqldump* 会转储其所有表格。要复制驻留在您的 MySQL 实例上的所有数据库，请指定选项 `--all-databases`。

# 6.7 使用可传输表空间复制 InnoDB 表格

## 问题

想要复制一个 InnoDB 表，但表太大了，以人类可读格式导出数据需要很长时间。重载也不够快。

## 解决方案

使用可传输表空间。

## 讨论

当处理较小的表格或者想要在将结果的 SQL 转储应用到目标服务器之前自行检查时，像 *mysqldump* 或 *mysqlpump* 这样的工具非常有效。然而，对于在磁盘上占据几十 GB 的表进行复制，将需要大量的时间。这也会给服务器增加额外的负载。更糟糕的是，保护机制会影响使用相同表的其他连接。

要解决这样的问题，存在二进制备份和恢复方法。这些方法在二进制表文件上工作，而不进行任何额外的数据操作，因此性能与在 Linux 上运行 *cp* 命令或在 Windows 上运行 *copy* 命令时相同。

从版本 8.0 开始，MySQL 将表定义存储在数据字典中，而数据存储在单独的文件中。这些文件的格式和名称取决于存储引擎。对于 InnoDB，它们是单独的、通用的和系统表空间。单独的表空间文件为每个表单独存储数据，可以用于我们在本节中描述的方法。如果您的表存储在系统或通用表空间中，则首先需要将它们转换为使用单独表空间格式。

```
ALTER TABLE tbl_name TABLESPACE = innodb_file_per_table;
```

要查看您的表是否位于系统或通用表空间中，请在信息模式中查询表 `INNODB_TABLES`：

```
mysql> `SELECT` `NAME``,` `SPACE_TYPE` `FROM` `INFORMATION_SCHEMA``.``INNODB_TABLES`
    -> `WHERE` `NAME` `LIKE` `'test/%'``;`
+----------------------------------------+------------+ | NAME                                   | SPACE_TYPE |
+----------------------------------------+------------+ | test/residing_in_system_tablespace     | System     |
| test/residing_in_individual_tablespace | Single     |
| test/residing_in_general_tablespace    | General    |
+----------------------------------------+------------+
```

一旦准备好复制表空间，请登录到 *mysql* 客户端并执行：

```
FLUSH TABLES limbs FOR EXPORT;
```

此命令将准备表空间文件以供复制，并额外创建扩展名为 `.cfg` 的配置文件，其中包含表元数据。

保持 MySQL 客户端打开，在另一个终端窗口中将表空间和配置文件复制到所需位置。

```
cp /var/lib/mysql/cookbook/limbs.{cfg,ibd} .
```

复制完成后解锁表格。

```
UNLOCK TABLES;
```

现在您可以将表空间导入到远程服务器或同一本地服务器上的不同数据库中。

第一步是创建与原始表完全相同定义的表。您可以通过运行 *SHOW CREATE TABLE* 命令找到表定义。

```
source> `SHOW` `CREATE` `TABLE` `limbs``\``G`
*************************** 1. row ***************************
       Table: limbs
Create Table: CREATE TABLE `limbs` (
  `thing` varchar(20) DEFAULT NULL,
  `legs` int DEFAULT NULL,
  `arms` int DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)
```

一旦获取它，请连接到目标数据库并创建表格。

```
destination> `USE` `cookbook_copy``;`
Database changed
destination> `CREATE` `TABLE` `` ` ```四肢``` ` `` `(`
          ->   `` ` ```东西``` ` `` `varchar``(``20``)` `DEFAULT` `NULL``,`
          ->   `` ` ```腿``` ` `` `int` `DEFAULT` `NULL``,`
          ->   `` ` ```臂``` ` `` `int` `DEFAULT` `NULL`
          -> `)` `ENGINE``=``InnoDB` `DEFAULT` `CHARSET``=``utf8mb4` `COLLATE``=``utf8mb4_0900_ai_ci``;`
Query OK, 0 rows affected (0.03 sec)
```

创建新的空表后，丢弃其表空间：

```
ALTER TABLE limbs DISCARD TABLESPACE;
```

###### 警告

`DISCARD TABLESPACE` 删除表空间文件。对于此命令要非常小心。如果输入错误并丢弃了错误表的表空间，则无法恢复。

在表空间被丢弃后，将表文件复制到新的数据库目录中。

```
$ `sudo cp limbs.{cfg,ibd} /var/lib/mysql/cookbook_copy`
$ `sudo chown mysql:mysql /var/lib/mysql/cookbook_copy/limbs.{cfg,ibd}`
```

然后导入表空间。

```
ALTER TABLE limbs IMPORT TABLESPACE;
```

## 参见

关于在 MySQL 数据库和服务器之间交换表空间文件的更多信息，请参阅 [导入 InnoDB 表](https://dev.mysql.com/doc/refman/8.0/en/innodb-table-import.html)。

# 6.8 使用 sdi 文件复制 MyISAM 表

## 问题

您想在 MySQL 8.0 上复制一个大的 MyISAM 表。

## 解决方案

使用 `IMPORT TABLE` 命令。

## 讨论

使用 MyISAM 存储引擎的表支持使用 *IMPORT TABLE* 语句导入原始表文件。为了在迁移期间无风险地导出 MyISAM 表而不会损坏数据，请首先打开 MySQL 连接，并使用读锁定刷新表文件到磁盘。

```
FLUSH TABLES limbs_myisam WITH READ LOCK;
```

然后复制表数据、索引和元数据文件到备份位置。

```
$ `sudo cp /var/lib/mysql/cookbook/limbs_myisam.{MYD,MYI} .`
$ `sudo bash -c 'cp /var/lib/mysql/cookbook/limbs_myisam_*.sdi . '`
```

解锁原始表格。

元数据文件的扩展名为 `.sdi`，其名称中有随机的数字序列，因此使用 *sudo* 复制它以允许 shell 进程扩展文件全局模式。

要将 MyISAM 表格复制到所需目的地，请将带有扩展名`sdi`的表格元数据文件放入由选项`--secure-file-priv`指定的目录，或者放入任何目录，只要目标 MySQL 服务器可读取，如果未设置此选项。然后将索引和数据文件复制到目标数据库目录。

```
$ `sudo cp limbs_myisam.{MYD,MYI} /var/lib/mysql/cookbook_copy/`
$ `sudo chown mysql:mysql /var/lib/mysql/cookbook_copy/limbs_myisam.{MYD,MYI}`
```

然后连接到数据库并导入表格。

```
IMPORT TABLE FROM '/tmp/limbs_myisam_11560.sdi';
```

如果要将表格复制到具有不同名称的数据库中，您需要手动编辑`sdi`文件，并将`schema_ref`的值替换为目标数据库名称。
