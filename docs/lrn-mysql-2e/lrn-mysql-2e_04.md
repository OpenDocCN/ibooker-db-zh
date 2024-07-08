# 第四章：处理数据库结构

本章节向您展示如何创建您自己的数据库，添加和删除诸如表和索引之类的结构，并在您的表中做出列类型的选择。它侧重于 SQL 的语法和特性，而不是关于构思、指定和完善数据库设计的语义；您将在第二章中找到数据库设计技术的简介描述。要完成本章的学习，您需要理解如何处理现有数据库及其表，如第三章所讨论的那样。

本章节列出了示例数据库`sakila`中的结构。如果您按照“实体关系建模示例”中的说明加载了数据库，那么您已经可以使用这个数据库，并且知道在修改其结构后如何恢复它。

当您完成本章时，您将掌握创建、修改和删除数据库结构所需的所有基础知识。与您在第三章中学到的技术一起，您将具备执行各种基本操作的技能。第五章和 7 章介绍了允许您使用 MySQL 执行更高级操作的技能。

# 创建和使用数据库

当您完成数据库的设计后，使用 MySQL 的第一个实际步骤是创建它。您可以使用`CREATE DATABASE`语句完成这一步骤。假设您要创建一个名为`lucy`的数据库，这是您要输入的语句：

```
mysql> `CREATE` `DATABASE` `lucy``;`
```

```
Query OK, 1 row affected (0.10 sec)
```

我们在这里假设您知道如何使用 MySQL 客户端进行连接，正如第一章中所述。我们还假设您能够作为 root 用户或另一个能够创建、删除和修改结构的用户进行连接（您将在第八章中找到有关用户权限的详细讨论）。请注意，当您创建数据库时，MySQL 会说有一行受到影响。事实上，这并不是任何特定数据库中的正常行，而是添加到您使用`SHOW DATABASES`命令看到的列表中的新条目。

一旦您创建了数据库，下一步是使用它——也就是说，选择它作为您正在使用的数据库。您可以使用 MySQL 的`USE`命令来实现这一点：

```
mysql> `USE` `lucy``;`
```

```
Database changed
```

此命令必须在一行中输入，无需用分号终止，尽管我们通常会通过习惯自动这样做。一旦您使用（选择）了数据库，您可以开始使用下一节讨论的步骤创建表、索引和其他结构。

在我们继续创建其他结构之前，让我们讨论一下创建数据库的一些特性和限制。首先，让我们看看如果您尝试创建一个已经存在的数据库会发生什么：

```
mysql> `CREATE` `DATABASE` `lucy``;`
```

```
ERROR 1007 (HY000): Can't create database 'lucy'; database exists
```

您可以通过在语句中添加`IF NOT EXISTS`关键字短语来避免此错误：

```
mysql> `CREATE` `DATABASE` `IF` `NOT` `EXISTS` `lucy``;`
```

```
Query OK, 0 rows affected (0.00 sec)
```

你可以看到 MySQL 没有投诉，但也没做任何事情：`0 rows affected`消息表明没有数据被更改。这个附加功能在将 SQL 语句添加到脚本时非常有用：它可以防止脚本在错误时中止。

让我们看看如何选择数据库名称并使用字符大小写。数据库名称定义了磁盘上的物理目录（或文件夹）名称。在某些操作系统上，目录名称区分大小写；在其他系统上，大小写无关紧要。例如，类 Unix 系统（如 Linux 和 macOS）通常区分大小写，而 Windows 则不区分。因此，数据库名称具有相同的限制：当大小写对操作系统有影响时，MySQL 也会受到影响。例如，在 Linux 机器上，`LUCY`、`lucy`和`Lucy`是不同的数据库名称；在 Windows 上，它们指的是同一个数据库。在 Linux 或 macOS 下使用不正确的大写会导致 MySQL 投诉：

```
mysql> `SELECT` `SaKilA``.``AcTor_id` `FROM` `ACTor``;`
```

```
ERROR 1146 (42S02): Table 'sakila.ACTor' doesn't exist
```

但在 Windows 下，这通常会起作用。

###### 提示

为了使你的 SQL 代码能够跨机器运行，我们建议始终使用小写命名数据库（以及表、列、别名和索引）。当然，这并非硬性要求，在本书的早期示例中已经证明，你可以使用自己习惯的命名规则。只需保持一致，并记住 MySQL 在不同操作系统上的行为。

这种行为由`lower_case_table_names`参数控制。如果设置为`0`，则表名按指定的方式存储，并且区分大小写。如果设置为`1`，则表名以小写形式存储在磁盘上，并且不区分大小写。如果此参数设置为`2`，则表名按给定的方式存储，但比较时以小写形式进行。在 Windows 上，默认值为`1`。在 macOS 上，默认值为`2`。在 Linux 上，不支持值`2`；服务器会强制将该值设置为`0`。

数据库名称还有其他限制。名称最多可以为 64 个字符长度。您还不应该使用 MySQL 保留字，例如`SELECT`、`FROM`和`USE`作为结构的名称；这些保留字会使 MySQL 解析器混淆，无法解释语句的含义。您可以通过将保留字用反引号（`` ` ``）括起来来避免此限制，但这样做比值得的麻烦多。此外，您不能在名称中使用特定字符，包括斜线、反斜线、分号和句点字符，数据库名称不能以空格结束。再次强调，这些字符的使用会使 MySQL 解析器混淆，并可能导致不可预测的行为。例如，当您在数据库名称中插入分号时会发生什么：

```
mysql> `CREATE` `DATABASE` `IF` `NOT` `EXISTS` `lu``;``cy``;`
```

```
Query OK, 1 row affected (0.00 sec)

ERROR 1064 (42000): You have an error in your SQL syntax; check the manual
that corresponds to your MySQL server version for the right syntax to use
near 'cy' at line 1
```

由于一行中可能有多个 SQL 语句，结果是创建了一个名为`lu`的数据库，然后由非常短、意外的 SQL 语句`cy;`生成了错误。如果您确实想要创建一个带有分号名称的数据库，可以使用反引号：

```
mysql> `CREATE` `DATABASE` `IF` `NOT` `EXISTS` `` `lu;cy` ```;`

```

```

查询完成，影响 1 行（0.01 秒）

```

And you can see that you now have two new databases:

```

mysql> `SHOW` `DATABASES` `LIKE` `` `lu%` ```;`
```

```
+----------------+
| Database (lu%) |
+----------------+
| lu             |
| lu;cy          |
+----------------+
2 rows in set (0.01 sec)
```

# 创建表格

本节涵盖表结构相关主题。我们将向您展示如何：

+   通过简单示例创建表格。

+   选择表格及相关结构的名称。

+   理解并选择列类型。

+   理解并选择键和索引。

+   使用专有的 MySQL `AUTO_INCREMENT`功能。

完成本节后，您将掌握所有关于创建数据库结构的基础材料；本章的其余部分将涵盖`sakila`示例数据库及如何修改和删除现有结构。

## 基础知识

对于本节中的示例，假设尚未创建`sakila`数据库。如果您希望跟随示例操作，并且已加载了数据库，可以在本节中删除它，稍后再重新加载；删除它将删除数据库、其表格和所有数据，但可以通过以下步骤轻松恢复原始状态，详见“实体关系建模示例”。以下是临时删除它的方法：

```
mysql> `DROP` `DATABASE` `sakila``;`
```

```
Query OK, 23 rows affected (0.06 sec)
```

在本章末尾进一步讨论`DROP`语句，详见“删除结构”。

要开始，使用以下语句创建数据库`sakila`：

```
mysql> `CREATE` `DATABASE` `sakila``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

然后使用以下语句选择数据库：

```
mysql> `USE` `sakila``;`
```

```
Database changed
```

现在我们准备开始创建将存储数据的表格。让我们创建一个用于保存演员详情的表格。暂时我们将使用简化的结构，稍后再添加更复杂的内容。以下是我们使用的语句：

```
mysql> `CREATE` `TABLE` `actor` `(`
    -> `actor_id` `SMALLINT` `UNSIGNED` `NOT` `NULL` `DEFAULT` `0``,`
    -> `first_name` `VARCHAR``(``45``)` `DEFAULT` `NULL``,`
    -> `last_name` `VARCHAR``(``45``)``,`
    -> `last_update` `TIMESTAMP``,`
    -> `PRIMARY` `KEY` `(``actor_id``)`
    -> `)``;`
```

```
Query OK, 0 rows affected (0.01 sec)
```

不要惊慌——即使 MySQL 报告说未影响任何行，它已经创建了表格：

```
mysql> `SHOW` `tables``;`
```

```
+------------------+
| Tables_in_sakila |
+------------------+
| actor            |
+------------------+
1 row in set (0.01 sec)
```

让我们详细考虑所有这些内容。`CREATE TABLE`命令有三个主要部分：

1.  `CREATE TABLE`语句后跟随要创建的表格名称。在本示例中，它是`actor`。

1.  列出要添加到表中的一个或多个列。在本示例中，我们添加了相当多的列：`actor_id SMALLINT UNSIGNED NOT NULL DEFAULT 0`，`first_name VARCHAR(45) DEFAULT NULL`，`last_name VARCHAR(45)`，以及`last_update TIMESTAMP`。稍后我们会详细讨论这些内容。

1.  可选的键定义。在本示例中，我们定义了一个主键：`PRIMARY KEY (actor_id)`。稍后我们将详细讨论键和索引。

注意`CREATE TABLE`组件后跟随的是开放括号，与语句末尾的闭合括号相匹配。还要注意其他组件由逗号分隔。`CREATE TABLE`语句还可以添加其他元素，我们稍后将讨论一些内容。

让我们讨论列规范。基本语法如下：`*name* *type* [NOT NULL | NULL] [DEFAULT *value*]`。`*name*`字段是列名，具有与数据库名称相同的限制，如前一节所述。其长度最多为 64 个字符，不允许使用反斜杠和斜杠，不允许使用句点，不能以空白结束，大小写敏感性取决于底层操作系统。`*type*`字段定义了列中存储的内容和方式；例如，我们已经看到它可以设置为`VARCHAR`表示字符串，`SMALLINT`表示数字，或`TIMESTAMP`表示日期和时间。

如果指定了`NOT NULL`，则无值列的行无效；如果指定了`NULL`或省略了此子句，则行可以存在而不为列提供值。如果在`DEFAULT`子句中指定了`*value*`，则在没有其他数据的情况下，它将用于填充列；当您经常重用默认值（如国家名称）时，这尤其有用。`*value*`必须是常量（如`0`、`"cat"`或`20060812045623`），除非列的类型是`TIMESTAMP`。类型将在“列类型”中详细讨论。

`NOT NULL`和`DEFAULT`特性可以一起使用。如果指定了`NOT NULL`并添加了`DEFAULT`值，则在未为列提供值时使用默认值。有时候，这很有效：

```
mysql> `INSERT` `INTO` `actor``(``first_name``)` `VALUES` `(``'John'``)``;`
```

```
Query OK, 1 row affected (0.01 sec)
```

有时候却不是这样：

```
mysql> `INSERT` `INTO` `actor``(``first_name``)` `VALUES` `(``'Elisabeth'``)``;`
```

```
ERROR 1062 (23000): Duplicate entry '0' for key 'actor.PRIMARY'
```

它是否有效取决于数据库的底层约束和条件：在此示例中，`actor_id`具有默认值`0`，但它也是主键。不允许具有相同主键值的两行存在，因此尝试插入没有值的行（并且结果主键值为`0`）的第二次尝试失败了。我们将在“键和索引”中详细讨论主键。

列名比数据库和表名的限制少。此外，列名不区分大小写，并且在所有平台上都可以移植。列名中允许使用所有字符，但如果要以空白结束或包含句点或其他特殊字符（如分号或破折号），则需要用反引号（`` ` ``）括起来。同样，我们建议您始终选择小写名称用于开发者驱动的选择（如数据库、别名和表名），并避免需要记住使用反引号的字符。

在开始新的命名列和其他数据库对象时，命名方式有一些个人偏好（您可以查看示例数据库以获取一些灵感），或者在现有代码库上工作时需要遵循标准。一般来说，避免重复是目标：在名为 `actor` 的表中，使用列名 `first_name` 而不是 `actor_first_name`，当在复杂查询中使用表名前缀时，后者看起来会显得多余（`actor.actor_first_name` 相对于 `actor.first_name`）。唯一的例外是使用普遍存在的 `id` 列名；要么避免使用这个名称，要么为了清晰起见在前面加上表名前缀（例如 `actor_id`）。使用下划线字符来分隔单词是一个良好的做法。您也可以使用其他字符，比如破折号或斜杠，但您需要记住要用反引号将名称括起来（例如 `` `actor-id` ``）。您也可以完全省略单词间的格式化，但是“驼峰命名法”可能更难阅读。与数据库和表名一样，列名的最大允许长度为 64 个字符。

## 排序和字符集

当您比较或排序字符串时，MySQL 如何评估结果取决于所使用的字符集和排序规则。字符集定义了可以存储哪些字符；例如，您可能需要存储像 ю 或 ü 这样的非英文字符。排序规则定义了字符串的顺序，不同的语言有不同的排序规则：例如，在两种德语排序中字符 ü 在字母表中的位置不同，在瑞典语和芬兰语中又有所不同。因为不是每个人都希望存储英文字符串，因此数据库服务器能够管理非英文字符和不同的字符排序方式非常重要。

我们理解，在您刚开始学习 MySQL 时，讨论排序和字符集可能会感觉太过深奥。然而，我们认为这些是值得涵盖的话题，因为不匹配的字符集和排序可能导致意外情况，包括数据丢失和错误的查询结果。如果您愿意，可以跳过本节以及本章节后面的部分内容，等到您有兴趣学习这些特定内容时再回来阅读。这不会影响您对本书其他材料的理解。

在我们先前的字符串比较示例中，我们忽略了排序和字符集问题，只是让 MySQL 使用其默认值。在 MySQL 8.0 之前的版本中，默认字符集为 `latin1`，默认排序规则为 `latin1_swedish_ci`。MySQL 8.0 更改了默认设置，现在默认字符集为 `utf8mb4`，默认排序规则为 `utf8mb4_0900_ai_ci`。MySQL 可以在连接、数据库、表和列级别配置使用不同的字符集和排序顺序。这里显示的输出来自 MySQL 8.0。

您可以使用 `SHOW CHARACTER SET` 命令列出服务器上可用的字符集。此命令显示每个字符集的简短描述、默认排序规则以及该字符集中每个字符的最大字节数：

```
mysql> `SHOW` `CHARACTER` `SET``;`
```

```
+----------+---------------------------------+---------------------+--------+
| Charset  | Description                     | Default collation   | Maxlen |
+----------+---------------------------------+---------------------+--------+
| armscii8 | ARMSCII-8 Armenian              | armscii8_general_ci |      1 |
| ascii    | US ASCII                        | ascii_general_ci    |      1 |
| big5     | Big5 Traditional Chinese        | big5_chinese_ci     |      2 |
| binary   | Binary pseudo charset           | binary              |      1 |
| cp1250   | Windows Central European        | cp1250_general_ci   |      1 |
| cp1251   | Windows Cyrillic                | cp1251_general_ci   |      1 |
| ...                                                                       |
| ujis     | EUC-JP Japanese                 | ujis_japanese_ci    |      3 |
| utf16    | UTF-16 Unicode                  | utf16_general_ci    |      4 |
| utf16le  | UTF-16LE Unicode                | utf16le_general_ci  |      4 |
| utf32    | UTF-32 Unicode                  | utf32_general_ci    |      4 |
| utf8     | UTF-8 Unicode                   | utf8_general_ci     |      3 |
| utf8mb4  | UTF-8 Unicode                   | utf8mb4_0900_ai_ci  |      4 |
+----------+---------------------------------+---------------------+--------+
41 rows in set (0.00 sec)
```

例如，`latin1` 字符集实际上是支持西欧语言的 Windows 代码页 1252 字符集。这种字符集的默认排序规则是 `latin1_swedish_ci`，遵循瑞典的惯例对带重音的字符进行排序（英语按照预期处理）。此排序规则是不区分大小写的，如字母 `ci` 所示。最后，每个字符占用 1 字节。相比之下，如果使用默认的 `utf8mb4` 字符集，每个字符将占用多达 4 个字节的存储空间。有时候，更改默认设置是有道理的。例如，没有理由在 `utf8mb4` 中存储 Base64 编码的数据（它定义为 ASCII）。

类似地，您可以列出排序规则和它们适用的字符集：

```
mysql> `SHOW` `COLLATION``;`
```

```
+---------------------+----------+-----+---------+...+---------------+
| Collation           | Charset  | Id  | Default |...| Pad_attribute |
+---------------------+----------+-----+---------+...+---------------+
| armscii8_bin        | armscii8 |  64 |         |...| PAD SPACE     |
| armscii8_general_ci | armscii8 |  32 | Yes     |...| PAD SPACE     |
| ascii_bin           | ascii    |  65 |         |...| PAD SPACE     |
| ascii_general_ci    | ascii    |  11 | Yes     |...| PAD SPACE     |
| ...                                            |...|               |
| utf8mb4_0900_ai_ci  | utf8mb4  | 255 | Yes     |...| NO PAD        |
| utf8mb4_0900_as_ci  | utf8mb4  | 305 |         |...| NO PAD        |
| utf8mb4_0900_as_cs  | utf8mb4  | 278 |         |...| NO PAD        |
| utf8mb4_0900_bin    | utf8mb4  | 309 |         |...| NO PAD        |
| ...                                            |...|               |
| utf8_unicode_ci     | utf8     | 192 |         |...| PAD SPACE     |
| utf8_vietnamese_ci  | utf8     | 215 |         |...| PAD SPACE     |
+---------------------+----------+-----+---------+...+---------------+
272 rows in set (0.02 sec)
```

###### 注意

可用的字符集和排序规则取决于 MySQL 服务器的构建和打包方式。我们展示的示例来自默认的 MySQL 8.0 安装，相同的数字在 Linux 和 Windows 上也可以看到。然而，MariaDB 10.5 有 322 种排序规则，但只有 40 种字符集。

您可以按以下方式查看服务器的当前默认设置：

```
mysql> `SHOW` `VARIABLES` `LIKE` `'c%'``;`
```

```
+--------------------------+--------------------------------+
| Variable_name            | Value                          |
+--------------------------+--------------------------------+
| ...                                                       |
| character_set_client     | utf8mb4                        |
| character_set_connection | utf8mb4                        |
| character_set_database   | utf8mb4                        |
| character_set_filesystem | binary                         |
| character_set_results    | utf8mb4                        |
| character_set_server     | utf8mb4                        |
| character_set_system     | utf8                           |
| character_sets_dir       | /usr/share/mysql-8.0/charsets/ |
| ...                                                       |
| collation_connection     | utf8mb4_0900_ai_ci             |
| collation_database       | utf8mb4_0900_ai_ci             |
| collation_server         | utf8mb4_0900_ai_ci             |
| ...                                                       |
+--------------------------+--------------------------------+
21 rows in set (0.00 sec)
```

在创建数据库时，您可以设置数据库及其表的默认字符集和排序顺序。例如，如果要使用 `utf8mb4` 字符集和 `utf8mb4_ru_0900_as_cs`（区分大小写）的排序顺序，您可以编写：

```
mysql> `CREATE` `DATABASE` `rose` `DEFAULT` `CHARACTER` `SET` `utf8mb4`
    -> `COLLATE` `utf8mb4_ru_0900_as_cs``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

如果您已正确为您的语言和地区安装了 MySQL，并且不打算国际化应用程序，则通常不需要这样做。从 MySQL 8.0 开始，默认字符集为 `utf8mb4`，甚至更少需要更改字符集。您还可以控制单独表或列的字符集和排序规则，但我们不会在这里详细讨论如何执行此操作。我们将讨论排序规则如何影响字符串类型的问题，详见“字符串类型”。

## 其他特性

本节简要描述了 `CREATE TABLE` 语句的其他功能。其中包括使用 `IF NOT EXISTS` 功能的示例，以及在本书中查找更多关于高级功能及其详细信息的列表。所示的语句是从 `sakila` 数据库中完整表示的表，不像之前的简化示例。

在创建表时，可以使用 `IF NOT EXISTS` 关键字短语，其工作方式与创建数据库类似。以下是一个示例，即使 `actor` 表已存在也不会报错：

```
mysql> `CREATE` `TABLE` `IF` `NOT` `EXISTS` `actor` `(`
    -> `actor_id` `SMALLINT` `UNSIGNED` `NOT` `NULL` `AUTO_INCREMENT``,`
    -> `first_name` `VARCHAR``(``45``)` `NOT` `NULL``,`
    -> `last_name` `VARCHAR``(``45``)` `NOT` `NULL``,`
    -> `last_update` `TIMESTAMP` `NOT` `NULL` `DEFAULT`
    -> `CURRENT_TIMESTAMP` `ON` `UPDATE` `CURRENT_TIMESTAMP``,`
    -> `PRIMARY` `KEY`  `(``actor_id``)``,`
    -> `KEY` `idx_actor_last_name` `(``last_name``)``)``;`
```

```
Query OK, 0 rows affected, 1 warning (0.01 sec)
```

您可以看到没有影响任何行，并且报告了一个警告。让我们来看看：

```
mysql> `SHOW` `WARNINGS``;`
```

```
+-------+------+------------------------------+
| Level | Code | Message                      |
+-------+------+------------------------------+
| Note  | 1050 | Table 'actor' already exists |
+-------+------+------------------------------+
1 row in set (0.01 sec)
```

`CREATE TABLE`语句中可以添加各种附加功能，本示例中仅列出了其中几个。许多这些功能都是高级功能，在本书中未涉及，但您可以在 MySQL 参考手册的[`CREATE TABLE`语句](https://oreil.ly/ZGQgq)部分找到更多信息。

数值列的`AUTO_INCREMENT`功能

此功能允许您为表自动创建唯一标识符。我们在“AUTO_INCREMENT 功能”中详细讨论它。

列注释

可以向列添加注释；当您使用稍后在本节讨论的`SHOW CREATE TABLE`命令时，会显示这些注释。

外键约束

您可以告诉 MySQL 检查一个或多个列中的数据是否与另一个表中的数据匹配。例如，`sakila`数据库在`address`表的`city_id`列上有一个外键约束，引用`city`表的`city_id`列。这意味着在`city`表中不存在的城市中无法有地址。我们在第二章介绍了外键约束，并将在“备用存储引擎”中查看哪些引擎支持外键约束。并非 MySQL 的每个存储引擎都支持外键。

创建临时表

如果使用`CREATE TEMPORARY TABLE`关键字短语创建表，则在连接关闭时将删除（丢弃）该表。这对于复制和重新格式化数据很有用，因为您无需记住清理工作。有时临时表也用作优化，用于保存一些中间数据。

高级表选项

您可以使用表选项控制表的广泛功能范围。这些包括`AUTO_INCREMENT`的起始值、索引和行存储方式以及覆盖 MySQL 查询优化器从表中收集的信息的选项。还可以指定*生成的列*，包含像两个其他列的总和之类的数据，以及这些列的索引。

索引结构控制

MySQL 中的一些存储引擎允许您指定和控制 MySQL 在其索引上使用的内部结构类型，如 B 树或哈希表。您还可以告诉 MySQL 希望在列上创建全文或空间数据索引，允许特殊类型的搜索。

分区

MySQL 支持不同的分区策略，您可以在创建表时或以后选择。我们不会在本书中涵盖分区。

您可以使用在第三章中介绍的`SHOW CREATE TABLE`语句查看用于创建表的语句。这通常显示了包含我们刚讨论的一些高级功能的输出；输出很少与实际用于创建表的内容匹配。以下是`actor`表的一个示例：

```
mysql> `SHOW` `CREATE` `TABLE` `actor``\``G`
```

```
*************************** 1\. row ***************************
       Table: actor
Create Table: CREATE TABLE `actor` (
  `actor_id` smallint unsigned NOT NULL AUTO_INCREMENT,
  `first_name` varchar(45) NOT NULL,
  `last_name` varchar(45) NOT NULL,
  `last_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`actor_id`),
  KEY `idx_actor_last_name` (`last_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)
```

您会注意到输出包含 MySQL 添加的内容，这些内容不在我们原始的`CREATE TABLE`语句中：

+   表和列的名称用反引号括起来。这并非必需，但可以避免由于使用保留字和特殊字符而引起的任何解析问题，如前面讨论的。

+   包括额外默认的`ENGINE`子句，明确说明应使用的表类型。在 MySQL 的默认安装中设置为`InnoDB`，因此在此示例中没有影响。

+   包括额外的`DEFAULT CHARSET`子句，告诉 MySQL 表中的列使用的字符集是什么。同样，在默认安装中没有影响。

## 列类型

本节描述了您可以在 MySQL 中使用的列类型。它解释了何时应使用每种类型以及它的任何限制。这些类型按其用途分组。我们将涵盖最常用的数据类型，并简要提及更高级或不常用的类型。这并不意味着它们没有用处，但考虑学习它们作为一种练习。您很可能不会记住每种数据类型及其特定的复杂性，这没关系。值得稍后重新阅读本章节并查阅有关主题的 MySQL 文档，以保持您的知识最新。

### 整数类型

我们将从数值数据类型开始，更具体地说是整数类型，或者保存特定整数的类型。首先，两种最流行的整数类型：

`INT[(*width*)] [UNSIGNED] [ZEROFILL]`

这是最常用的数字类型；它在范围内存储整数（整数）值-2,147,483,648 至 2,147,483,647。如果添加可选的`UNSIGNED`关键字，则范围为 0 至 4,294,967,295。关键字`INT`是`INTEGER`的缩写，它们可以互换使用。`INT`列需要 4 个字节的存储空间。

`INT`，以及其他整数类型，具有两个特定于 MySQL 的属性：可选的`*width*`和`ZEROFILL`参数。它们不是 SQL 标准的一部分，并且自 MySQL 8.0 起已被弃用。尽管如此，您肯定会在许多代码库中注意到它们，因此我们将简要介绍它们。

`*width*`参数指定显示宽度，可以作为列元数据的一部分被应用程序读取。与其他数据类型类似位置的参数不同，此参数对特定整数类型的存储特性没有影响，也不限制可用值的范围。对于数据存储目的，`INT(4)`和`INT(32)`是相同的。

`ZEROFILL`是一个额外的参数，用于将值左填充为零，直到达到`*width*`参数指定的长度。如果使用`ZEROFILL`，MySQL 会自动将`UNSIGNED`添加到声明中（因为零填充只在正数的情况下有意义）。

在一些需要`ZEROFILL`和`*width*`的应用程序中，可以使用`LPAD()`函数，或者可以将数字存储为格式化的`CHAR`列。

`BIGINT[(*width*)] [UNSIGNED] [ZEROFILL]`

在数据规模不断增长的世界中，拥有数十亿行计数的表格变得越来越普遍。即使是简单的`id`类型列，可能也需要比普通的`INT`提供的范围更广。`BIGINT`解决了这个问题。它是一个大整数类型，有一个带符号范围从–9,223,372,036,854,775,808 到 9,223,372,036,854,775,807。无符号的`BIGINT`可以存储从 0 到 18,446,744,073,709,551,615 的数字。这种类型的列将需要 8 字节的存储空间。

在 MySQL 中，所有计算都是使用带符号的`BIGINT`或`DOUBLE`值完成的。这其中的重要结果是，在处理非常大的数字时，必须非常小心。有两个问题需要注意。首先，大于 9,223,372,036,854,775,807 的无符号大整数只能与位函数一起使用。其次，如果算术运算的结果大于 9,223,372,036,854,775,807，则可能会观察到意外的结果。

例如：

```
mysql> `CREATE` `TABLE` `test_bigint` `(``id` `BIGINT` `UNSIGNED``)``;`
```

```
Query OK, 0 rows affected (0.01 sec)
```

```
mysql> `INSERT` `INTO` `test_bigint` `VALUES` `(``18446744073709551615``)``;`
```

```
Query OK, 1 row affected (0.01 sec)
```

```
mysql> `INSERT` `INTO` `test_bigint` `VALUES` `(``18446744073709551615``-``1``)``;`
```

```
Query OK, 1 row affected (0.01 sec)
```

```
mysql> `INSERT` `INTO` `test_bigint` `VALUES` `(``184467440737095516``*``100``)``;`
```

```
ERROR 1690 (22003): BIGINT value
is out of range in '(184467440737095516 * 100)'
```

即使 18,446,744,073,709,551,600 小于 18,446,744,073,709,551,615，由于内部使用带符号的`BIGINT`进行乘法，观察到了超出范围的错误。

###### 提示

`SERIAL`数据类型可以用作`BIGINT UNSIGNED NOT NULL AUTO_INCREMENT UNIQUE`的别名。除非必须优化数据大小和性能，否则考虑在类似`id`的列中使用`SERIAL`。即使是`UNSIGNED INT`，其超出范围的速度也比你预期的快得多，并且通常出现在最糟糕的时候。

请记住，虽然可以将每个整数存储为`BIGINT`，但从存储空间的角度来看这是浪费的。此外，正如我们讨论过的，`*width*`参数并不限制值的范围。为了节省空间并对存储的值设置约束，应该使用不同的整数类型：

`SMALLINT[(*width*)] [UNSIGNED] [ZEROFILL]`

存储小整数，带有带符号范围从–32,768 到 32,767 和无符号范围从 0 到 65,535。它需要 2 字节的存储空间。

`TINYINT[(*width*)] [UNSIGNED] [ZEROFILL]`

最小的数值数据类型，可以存储更小的整数。这种类型的范围是带符号的–128 到 127，无符号的 0 到 255。它只需要 1 字节的存储空间。

`BOOL[(*width*)]`

缩写为`BOOLEAN`，是`TINYINT(1)`的同义词。通常，布尔类型只接受两个值：真或假。但是，由于 MySQL 中的`BOOL`是整数类型，你可以在`BOOL`中存储–128 到 127 的值。值为 0 将被视为假，所有非零值将被视为真。还可以使用特殊的`true`和`false`别名来代表 1 和 0。以下是一些例子：

```
mysql> `CREATE` `TABLE` `test_bool` `(``i` `BOOL``)``;`
```

```
Query OK, 0 rows affected (0.04 sec)
```

```
mysql> `INSERT` `INTO` `test_bool` `VALUES` `(``true``)``,``(``false``)``;`
```

```
Query OK, 2 rows affected (0.00 sec)
Records: 2  Duplicates: 0  Warnings: 0
```

```
mysql> `INSERT` `INTO` `test_bool` `VALUES` `(``1``)``,``(``0``)``,``(``-``128``)``,``(``127``)``;`
```

```
Query OK, 4 rows affected (0.02 sec)
Records: 4  Duplicates: 0  Warnings: 0
```

```
mysql> `SELECT` `i``,` `IF``(``i``,``'true'``,``'false'``)` `FROM` `test_bool``;`
```

```
+------+----------------------+
| i    | IF(i,'true','false') |
+------+----------------------+
|    1 | true                 |
|    0 | false                |
|    1 | true                 |
|    0 | false                |
| -128 | true                 |
|  127 | true                 |
+------+----------------------+
6 rows in set (0.01 sec)
```

`MEDIUMINT[(*width*)] [UNSIGNED] [ZEROFILL]`

存储有带符号范围从–8,388,608 到 8,388,607 和无符号范围从 0 到 16,777,215 的值。它需要 3 字节的存储空间。

`BIT[(*M*)]`

用于存储位值的特殊类型。`*M*` 指定每个值的位数，默认为 1（如果省略）。MySQL 使用 `b'*value*` 语法来表示二进制值。

### 固定点类型

在 MySQL 中，`DECIMAL` 和 `NUMERIC` 数据类型是相同的，因此虽然我们这里只描述 `DECIMAL`，但此描述也适用于 `NUMERIC`。固定点和浮点类型的主要区别在于精度。对于固定点类型，检索的值与存储的值相同；对于包含小数点的类型（例如后面描述的 `FLOAT` 和 `DOUBLE` 类型），情况并非总是如此。这是 `DECIMAL` 数据类型的最重要属性，它是 MySQL 中常用的数值类型之一：

`DECIMAL[(*width*[,*decimals*])] [UNSIGNED] [ZEROFILL]`

存储固定点数，如工资或距离，总共有`*width*`位数，其中一些较小的数字跟随小数点。例如，声明为 `price DECIMAL(6,2)` 的列可用于存储范围内的值 -9,999.99 到 9,999.99。`price DECIMAL(10,4)` 允许像 123,456.1234 这样的值。

在 MySQL 5.7 之前，如果尝试存储超出此范围的值，它将存储为允许范围内最接近的值。例如，100 将存储为 99.99，而 -100 将存储为 -99.99。然而，从版本 5.7.5 开始，默认的 SQL 模式包含 `STRICT_TRANS_TABLES` 模式，禁止此类和其他不安全的行为。使用旧的行为仍然可能，但可能导致数据丢失。

SQL 模式是特殊设置，控制 MySQL 在查询时的行为。例如，它们可以限制“不安全”行为或影响查询的解释方式。为了学习 MySQL，建议您保持默认设置，因为它们是安全的。根据 MySQL 的发布版本，可能需要更改 SQL 模式以与旧应用程序兼容。

`*width*` 参数是可选的，当省略时默认为 10。`*decimals*` 的数量也是可选的，当省略时默认为 0；`*decimals*` 的最大值不得超过 `*width*` 的值。`*width*` 的最大值为 65，`*decimals*` 的最大值为 30。

如果只存储正值，可以像`INT`一样使用 `UNSIGNED` 关键字。如果需要零填充，可以使用 `ZEROFILL` 关键字，与`INT`中描述的行为相同。`DECIMAL` 关键字有三个相同且可互换的替代词：`DEC`、`NUMERIC` 和 `FIXED`。

在`DECIMAL`列中存储的值采用二进制格式。该格式对于每九位数字使用 4 字节。

### 浮点类型

除了上一节中描述的固定点 `DECIMAL` 类型外，还有两种支持小数点的类型：`DOUBLE`（也称为 `REAL`）和 `FLOAT`。它们设计用于存储近似数值而不是 `DECIMAL` 存储的精确值。

为什么要使用近似值？答案是许多带有小数点的数字都是真实量的近似值。例如，假设您的年收入为$50,000，并且希望将其存储为月工资。当您将其转换为每月金额时，是$4,166 加上 66 和 2/3 美分。如果将此存储为$4,166.67，则不精确到足以转换为年薪（因为 12 乘以$4,166.67 是$50,000.04）。但是，如果以足够多的小数位数存储 2/3，则是一个更接近的近似值。您将发现，在像 MySQL 这样的高精度环境中，仅需进行少量舍入，就足以正确地乘以以获得原始值。这就是 `DOUBLE` 和 `FLOAT` 有用的地方：它们允许您存储诸如 2/3 或 pi 等值，带有大量小数位数，从而精确地近似表示确切量。稍后，您可以使用 `ROUND()` 函数将结果恢复到给定的精度。

让我们继续上一个示例，使用 `DOUBLE`。假设您创建了以下表：

```
mysql> `CREATE` `TABLE` `wage` `(``monthly` `DOUBLE``)``;`
```

```
Query OK, 0 rows affected (0.09 sec)
```

现在可以使用以下方式插入月工资：

```
mysql> `INSERT` `INTO` `wage` `VALUES` `(``50000``/``12``)``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

并查看存储内容：

```
mysql> `SELECT` `*` `FROM` `wage``;`
```

```
+----------------+
| monthly        |
+----------------+
| 4166.666666666 |
+----------------+
1 row in set (0.00 sec)
```

然而，当您将其乘以以获得年度值时，会得到一个高精度的近似值：

```
mysql> `SELECT` `monthly``*``12` `FROM` `wage``;`
```

```
+--------------------+
| monthly*12         |
+--------------------+
| 49999.999999992004 |
+--------------------+
1 row in set (0.00 sec)
```

要恢复原始值，您仍然需要以所需的精度执行舍入。例如，您的业务可能要求精确到五位小数点位数。在这种情况下，您可以使用以下方法将原始值恢复：

```
mysql> `SELECT` `ROUND``(``monthly``*``12``,``5``)` `FROM` `wage``;`
```

```
+---------------------+
| ROUND(monthly*12,5) |
+---------------------+
|         50000.00000 |
+---------------------+
1 row in set (0.00 sec)
```

但是精确到八位小数点位数不会得到原始值：

```
mysql> `SELECT` `ROUND``(``monthly``*``12``,``8``)` `FROM` `wage``;`
```

```
+---------------------+
| ROUND(monthly*12,8) |
+---------------------+
|      49999.99999999 |
+---------------------+
1 row in set (0.00 sec)
```

理解浮点数据类型的不精确和近似性质是非常重要的。

以下是 `FLOAT` 和 `DOUBLE` 类型的详细信息：

`FLOAT[(*宽度*, *小数位数*)] [UNSIGNED] [ZEROFILL]` 或 `FLOAT[(*精度*)] [UNSIGNED] [ZEROFILL]`

存储浮点数。它有两种可选的语法：第一种允许可选的`*小数位数*`和可选的显示`*宽度*`，第二种允许可选的`*精度*`，该精度控制以比特为单位的近似精度。在没有参数的情况下（典型用法），该类型存储小型的、4 字节的单精度浮点数值。当`*精度*`在 0 到 24 之间时，采用默认行为。当`*精度*`在 25 到 53 之间时，该类型的行为类似于`DOUBLE`。`*宽度*`参数不影响存储内容，只影响显示内容。`UNSIGNED` 和 `ZEROFILL` 选项的行为与 `INT` 类型相同。

`DOUBLE[(*宽度*, *小数位数*)] [UNSIGNED] [ZEROFILL]`

存储浮点数。允许指定可选的`*小数位数*`和可选的显示`*宽度*`。在没有参数的情况下（典型用法），该类型存储普通的 8 字节双精度浮点数值。`*宽度*`参数不影响存储内容，只影响显示内容。`UNSIGNED` 和 `ZEROFILL` 选项的行为与 `INT` 类型相同。`DOUBLE` 类型有两个相同的同义词：`REAL` 和 `DOUBLE PRECISION`。

### 字符串类型

字符串数据类型用于存储文本和不那么明显的二进制数据。MySQL 支持以下字符串类型：

`[国家] VARCHAR(*width*) [字符集 *charset_name*] [排序 *collation_name*]`

可能是最常用的字符串类型之一，`VARCHAR` 可以存储长度可变的字符串，最大长度为 `*width*`。`*width*` 的最大值为 65,535 个字符。大部分适用于该类型的信息也适用于其他字符串类型。

`CHAR` 和 `VARCHAR` 类型非常相似，但有几个重要的区别。`VARCHAR` 需要额外的一到两个字节的开销来存储字符串的值，具体取决于值是大于还是小于 255 字节。注意这个大小与字符长度不同，因为某些字符可能需要最多 4 个字节的空间。因此，`VARCHAR` 看起来效率较低并不总是正确的。因为 `VARCHAR` 可以存储任意长度的字符串（最多定义为 `*width*`），比起相似长度的 `CHAR`，较短的字符串将需要更少的存储空间。

`CHAR` 和 `VARCHAR` 之间的另一个差异是它们对尾随空格的处理。`VARCHAR` 保留尾随空格直到指定的列宽，并将多余部分截断，会产生警告。稍后将展示，`CHAR` 值会右填充到列宽，并且不会保留尾随空格。对于 `VARCHAR`，尾随空格是重要的，除非它们被修剪，并且将计为唯一值。让我们演示一下：

```
mysql> `CREATE` `TABLE` `test_varchar_trailing``(``d` `VARCHAR``(``2``)` `UNIQUE``)``;`
```

```
Query OK, 0 rows affected (0.02 sec)
```

```
mysql> `INSERT` `INTO` `test_varchar_trailing` `VALUES` `(``'a'``)``,` `(``'a '``)``;`
```

```
Query OK, 2 rows affected (0.01 sec)
Records: 2  Duplicates: 0  Warnings: 0
```

```
mysql> `SELECT` `d``,` `LENGTH``(``d``)` `FROM` `test_varchar_trailing``;`
```

```
+------+-----------+
| d    | LENGTH(d) |
+------+-----------+
| a    |         1 |
| a    |         2 |
+------+-----------+
2 rows in set (0.00 sec)
```

我们插入的第二行有一个尾随空格，由于列 `d` 的 `*width*` 为 2，该空格对行的唯一性计数起作用。然而，如果我们尝试插入一个有两个尾随空格的行：

```
mysql> `INSERT` `INTO` `test_varchar_trailing` `VALUES` `(``'a  '``)``;`
```

```
ERROR 1062 (23000): Duplicate entry 'a '
for key 'test_varchar_trailing.d'
```

MySQL 拒绝接受新行。`VARCHAR(2)` 会隐式截断超出设定 `*width*` 的尾随空格，因此存储的值会从 `"a "`（*a* 后有双空格）变为 `"a "`（*a* 后只有单空格）。由于已经存在这样一个值的行，报告了重复条目错误。可以通过更改列排序规则来控制 `VARCHAR` 和 `TEXT` 的此行为。一些排序规则，如 `latin1_bin`，具有 `PAD SPACE` 属性，意味着在检索时它们会用空格填充到 `*width*`。这不影响存储，但会影响唯一性检查以及 `GROUP BY` 和 `DISTINCT` 操作符的工作方式，我们将在第五章中讨论。您可以通过运行 `SHOW COLLATION` 命令来检查排序规则是否为 `PAD SPACE` 或 `NO PAD`，正如我们在“排序规则和字符集”中所示。让我们通过创建一个具有 `PAD SPACE` 排序规则的表来查看其效果：

```
mysql> `CREATE` `TABLE` `test_varchar_pad_collation``(`
    -> `data` `VARCHAR``(``5``)` `CHARACTER` `SET` `latin1`
    -> `COLLATE` `latin1_bin` `UNIQUE``)``;`
```

```
Query OK, 0 rows affected (0.02 sec)
```

```
mysql> `INSERT` `INTO` `test_varchar_pad_collation` `VALUES` `(``'a'``)``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

```
mysql> `INSERT` `INTO` `test_varchar_pad_collation` `VALUES` `(``'a '``)``;`
```

```
ERROR 1062 (23000): Duplicate entry 'a '
for key 'test_varchar_pad_collation.data'
```

`NO PAD` 排序规则是 MySQL 8.0 的新添加。在之前的 MySQL 版本中，你可能经常看到的每种排序规则都隐含具有 `PAD SPACE` 属性。因此，在 MySQL 5.7 及之前的版本中，保留尾部空格的唯一选择是使用二进制类型：`VARBINARY` 或 `BLOB`。

###### 注意

`CHAR` 和 `VARCHAR` 数据类型都不允许存储超过 `*width*` 的值，除非禁用严格 SQL 模式（即未启用 `STRICT_ALL_TABLES` 或 `STRICT_TRANS_TABLES`）。禁用保护后，超过 `*width*` 的值将被截断，并显示警告。我们不建议启用旧版行为，因为可能导致数据丢失。

`VARCHAR`、`CHAR` 和 `TEXT` 类型的排序和比较根据分配的字符集的排序规则进行。您可以看到，可以为每个单独的字符串类型列指定字符集和排序规则。还可以指定 `binary` 字符集，有效地将 `VARCHAR` 转换为 `VARBINARY`。不要将 `binary` 字符集误认为是字符集的 `BINARY` 属性；后者是 MySQL 的一种仅用于指定二进制（`_bin`）排序规则的简写。

此外，可以在 `ORDER BY` 子句中直接指定排序规则。可用的排序规则将取决于列的字符集。继续使用 `test_varchar_pad_collation` 表，可以在其中存储一个 `ä` 符号，然后查看排序规则对字符串排序的影响：

```
mysql> `INSERT` `INTO` `test_varchar_pad_collation` `VALUES` `(``'ä'``)``,` `(``'z'``)``;`
```

```
Query OK, 2 rows affected (0.01 sec)
Records: 2  Duplicates: 0  Warnings: 0
```

```
mysql> `SELECT` `*` `FROM` `test_varchar_pad_collation`
    -> `ORDER` `BY` `data` `COLLATE` `latin1_german1_ci``;`
```

```
+------+
| data |
+------+
| a    |
| ä    |
| z    |
+------+
3 rows in set (0.00 sec)
```

```
mysql> `SELECT` `*` `FROM` `test_varchar_pad_collation`
    -> `ORDER` `BY` `data` `COLLATE` `latin1_swedish_ci``;`
```

```
+------+
| data |
+------+
| a    |
| z    |
| ä    |
+------+
3 rows in set (0.00 sec)
```

`NATIONAL`（或其等效的缩写形式 `NCHAR`）属性是指定字符串类型列必须使用预定义字符集的标准 SQL 方式。MySQL 使用 `utf8` 作为此字符集。重要的是要注意 MySQL 5.7 和 8.0 在 `utf8` 的具体定义上存在差异：前者将其用作 `utf8mb3` 的别名，后者则用于 `utf8mb4`。因此，最好不要使用 `NATIONAL` 属性以及含糊不清的别名。对于任何涉及文本列和数据的最佳实践是尽可能明确和具体。

`[国家] CHAR(*width*) [字符集 *charset_name*] [排序规则 *collation_name*]`

`CHAR` 存储长度固定的字符串（例如姓名、地址或城市），长度为 `*width*`。如果未提供 `*width*`，则假定为 `CHAR(1)`。`*width*` 的最大值为 255。与 `VARCHAR` 一样，`CHAR` 列中的值始终存储在指定的长度上。存储在 `CHAR(255)` 列中的单个字母将占用 255 字节（在 `latin1` 字符集下），并且会填充空格。读取数据时会移除填充，除非启用了 `PAD_CHAR_TO_FULL_LENGTH` SQL 模式。再次强调，这意味着存储在 `CHAR` 列中的字符串将丢失所有尾部空格。

在过去，`CHAR`列的`*width*`通常与字节大小相关联。 现在情况并非总是如此，而且默认情况下也不是如此。 例如，默认的 MySQL 8.0 中的多字节字符集，如`utf8mb4`，可能导致更大的值。 如果最大大小超过 768 字节，则 InnoDB 实际上将固定长度列编码为可变长度列。 因此，在 MySQL 8.0 中，默认情况下 InnoDB 将`CHAR(255)`列存储为`VARCHAR`列。 以下是一个例子：

```
mysql> `CREATE` `TABLE` `test_char_length``(`
    ->   `utf8char` `CHAR``(``10``)` `CHARACTER` `SET` `utf8mb4`
    -> `,` `asciichar` `CHAR``(``10``)` `CHARACTER` `SET` `binary`
    -> `)``;`
```

```
Query OK, 0 rows affected (0.04 sec)
```

```
mysql> `INSERT` `INTO` `test_char_length` `VALUES` `(``'Plain text'``,` `'Plain text'``)``;`
```

```
Query OK, 1 row affected (0.01 sec)
```

```
mysql> `INSERT` `INTO` `test_char_length` `VALUES` `(`'的開源軟體'`,` `'Plain text'``)``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

```
mysql> `SELECT` `LENGTH``(``utf8char``)``,` `LENGTH``(``asciichar``)` `FROM` `test_char_length``;`
```

```
+------------------+-------------------+
| LENGTH(utf8char) | LENGTH(asciichar) |
+------------------+-------------------+
|               10 |                10 |
|               15 |                10 |
+------------------+-------------------+
2 rows in set (0.00 sec)
```

由于值左对齐并在右侧填充空格，并且根本不考虑`CHAR`中的任何尾随空格，因此无法比较仅由空格组成的字符串。 如果发现自己处于这种情况下是重要的，`VARCHAR`是要使用的数据类型。

`BINARY[(*width*)]`和`VARBINARY(*width*)`

这些类型与`CHAR`和`VARCHAR`非常相似，但存储二进制字符串。 二进制字符串具有特殊的`binary`字符集和排序规则，排序依赖于存储的值的字节的数值。 存储字节字符串而不是字符字符串。 在前面讨论`VARCHAR`时，我们描述了`binary`字符集和`BINARY`属性。 仅`binary`字符集将`VARCHAR`或`CHAR`“转换”为其相应的`BINARY`形式。 将`BINARY`属性应用于字符集不会改变存储字符字符串的事实。 与`VARCHAR`和`CHAR`不同，`*width*`在这里确切地是字节数。 对于`BINARY`，当省略`*width*`时，默认为 1。

与`CHAR`类似，`BINARY`列中的数据在右侧填充。 但是，作为二进制数据，它使用零字节进行填充，通常写为`0x00`或`\0`。 `BINARY`将空格视为显著字符，而不是填充。 如果您需要存储可能以对您重要的零字节结尾的数据，请使用`VARBINARY`或`BLOB`类型。

在使用这两种数据类型时，牢记二进制字符串的概念至关重要。 尽管它们接受字符串，但它们不是使用文本字符串的数据类型的同义词。 例如，您无法更改存储的字母的大小写，因为该概念实际上不适用于二进制数据。 当您考虑实际存储的数据时，这一点变得非常明显。 让我们看一个例子：

```
mysql> `CREATE` `TABLE` `test_binary_data` `(`
    ->   `d1` `BINARY``(``16``)`
    -> `,` `d2` `VARBINARY``(``16``)`
    -> `,` `d3` `CHAR``(``16``)`
    -> `,` `d4` `VARCHAR``(``16``)`
    -> `)``;`
```

```
Query OK, 0 rows affected (0.03 sec)
```

```
mysql> `INSERT` `INTO` `test_binary_data` `VALUES` `(`
    ->   `'something'`
    -> `,` `'something'`
    -> `,` `'something'`
    -> `,` `'something'``)``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

```
mysql> `SELECT` `d1``,` `d2``,` `d3``,` `d4` `FROM` `test_binary_data``;`
```

```
*************************** 1\. row ***************************
d1: 0x736F6D657468696E6700000000000000
d2: 0x736F6D657468696E67
d3: something
d4: something
1 row in set (0.00 sec)
```

```
mysql> `SELECT` `UPPER``(``d2``)``,` `UPPER``(``d4``)` `FROM` `test_binary_data``;`
```

```
*************************** 1\. row ***************************
UPPER(d2): 0x736F6D657468696E67
UPPER(d4): SOMETHING
1 row in set (0.01 sec)
```

注意 MySQL 命令行客户端实际上以十六进制格式显示二进制类型的值。 我们认为这比在 MySQL 8.0 之前执行的沉默转换要好得多，这可能导致误解。 要获取实际的文本数据，您必须将二进制数据显式转换为文本：

```
mysql> `SELECT` `CAST``(``d1` `AS` `CHAR``)` `d1t``,` `CAST``(``d2` `AS` `CHAR``)` `d2t`
    -> `FROM` `test_binary_data``;`
```

```
+------------------+-----------+
| d1t              | d2t       |
+------------------+-----------+
| something        | something |
+------------------+-----------+
1 row in set (0.00 sec)
```

`BINARY`填充在转换执行时会被转换为空格。

`BLOB[(*width*)]`和`TEXT[(*width*)] [CHARACTER SET *charset_name*] [COLLATE *collation_name*]`

`BLOB` 和 `TEXT` 是常用的用于存储大数据的数据类型。可以将 `BLOB` 视为可以容纳任意数据的 `VARBINARY`，`TEXT` 和 `VARCHAR` 同理。`BLOB` 和 `TEXT` 类型分别可以存储最多 65,535 个字节或字符。请注意，多字节字符集确实存在。`*width*` 属性是可选的，当指定时，MySQL 实际上会将 `BLOB` 或 `TEXT` 数据类型更改为能够容纳该数据量的最小类型。例如，`BLOB(128)` 将导致使用 `TINYBLOB`：

```
mysql> `CREATE` `TABLE` `test_blob``(``data` `BLOB``(``128``)``)``;`
```

```
Query OK, 0 rows affected (0.07 sec)
```

```
mysql> `DESC` `test_blob``;`
```

```
+-------+----------+------+-----+---------+-------+
| Field | Type     | Null | Key | Default | Extra |
+-------+----------+------+-----+---------+-------+
| data  | tinyblob | YES  |     | NULL    |       |
+-------+----------+------+-----+---------+-------+
1 row in set (0.00 sec)
```

对于 `BLOB` 类型及相关类型，数据的处理与 `VARBINARY` 相同。也就是说，不假设字符集，并且基于实际存储的字节的数值进行比较和排序。对于 `TEXT`，可以指定确切的字符集和排序规则。对于这两种类型及其变体，在 `INSERT` 时不进行填充，在 `SELECT` 时不进行修剪，非常适合精确存储数据。此外，不允许使用 `DEFAULT` 子句，并且在 `BLOB` 或 `TEXT` 列上创建索引时，必须定义一个前缀，限制索引值的长度。我们在 “键和索引” 中会详细讨论这一点。

`BLOB` 和 `TEXT` 之间的一个潜在区别是它们对尾随空格的处理方式。正如我们已经展示的那样，根据使用的排序规则，`VARCHAR` 和 `TEXT` 可能会填充字符串。`BLOB` 和 `VARBINARY` 都使用 `binary` 字符集和单一的 `binary` 排序规则，不进行填充，并且不受排序规则混淆及相关问题的影响。有时，使用这些类型可以增加额外的安全性。除此之外，在 MySQL 8.0 之前，这些类型是唯一保留尾随空格的类型。

`TINYBLOB` 和 `TINYTEXT [字符集 *charset_name*] [排序规则 *collation_name*]`

这些类型与 `BLOB` 和 `TEXT` 完全相同，只是最多可以存储 255 个字节或字符。

`MEDIUMBLOB` 和 `MEDIUMTEXT [字符集 *charset_name*] [排序规则 *collation_name*]`

这些类型与 `BLOB` 和 `TEXT` 完全相同，只是最多可以存储 16,777,215 个字节或字符。类型 `LONG` 和 `LONG VARCHAR` 映射到 `MEDIUMTEXT` 数据类型，以保持兼容性。

`LONGBLOB` 和 `LONGTEXT [字符集 *charset_name*] [排序规则 *collation_name*]`

这些类型与 `BLOB` 和 `TEXT` 完全相同，只是最多可以存储 4 GB 的数据。请注意，即使是在 `LONGTEXT` 的情况下，这也是一个硬性限制，因此多字节字符集中的字符数可能少于 4,294,967,295。客户端可以存储的数据的有效最大大小将受到可用内存量以及 `max_packet_size` 变量值（默认为 64 MiB）的限制。

`ENUM(**value1**[,**value2**[, …]]) [字符集 *charset_name*] [排序规则 *collation_name*]`

这种类型存储一组字符串值的列表，或*枚举*。`ENUM`类型的列可以设置为列表`*value1*`、`*value2*`等中的一个值，最多可达到 65,535 个不同的值。虽然这些值以字符串形式存储和检索，但在数据库中存储的是整数表示。`ENUM`列可以包含`NULL`值（存储为`NULL`）、空字符串`''`（存储为`0`）或任何有效元素（存储为`1`、`2`、`3`等）。您可以通过在创建表时将列声明为`NOT NULL`来阻止接受`NULL`值。

这种类型提供了一种紧凑的方式来存储预定义值列表中的值，例如州或国家名称。考虑以下使用水果名称的示例；名称可以是预定义值`Apple`、`Orange`或`Pear`（以及`NULL`和空字符串）中的任何一个：

```
mysql> `CREATE` `TABLE` `fruits_enum`
    -> `(``fruit_name` `ENUM``(``'Apple'``,` `'Orange'``,` `'Pear'``)``)``;`
```

```
Query OK, 0 rows affected (0.00 sec)
```

```
mysql> `INSERT` `INTO` `fruits_enum` `VALUES` `(``'Apple'``)``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

如果尝试插入不在列表中的值，MySQL 会产生错误，告诉您它没有存储您请求的数据：

```
mysql> `INSERT` `INTO` `fruits_enum` `VALUES` `(``'Banana'``)``;`
```

```
ERROR 1265 (01000): Data truncated for column 'fruit_name' at row 1
```

也不接受几个允许的值列表：

```
mysql> `INSERT` `INTO` `fruits_enum` `VALUES` `(``'Apple,Orange'``)``;`
```

```
ERROR 1265 (01000): Data truncated for column 'fruit_name' at row 1
```

显示表的内容，您会看到没有存储无效值：

```
mysql> `SELECT` `*` `FROM` `fruits_enum``;`
```

```
+------------+
| fruit_name |
+------------+
| Apple      |
+------------+
1 row in set (0.00 sec)
```

MySQL 的早期版本会产生警告而不是错误，并在无效值的位置存储空字符串。通过禁用默认的严格 SQL 模式可以启用该行为。还可以指定除空字符串以外的默认值：

```
mysql> `CREATE` `TABLE` `new_fruits_enum`
    -> `(``fruit_name` `ENUM``(``'Apple'``,` `'Orange'``,` `'Pear'``)`
    -> `DEFAULT` `'Pear'``)``;`
```

```
Query OK, 0 rows affected (0.01 sec)
```

```
mysql> `INSERT` `INTO` `new_fruits_enum` `VALUES``(``)``;`
```

```
Query OK, 1 row affected (0.02 sec)
```

```
mysql> `SELECT` `*` `FROM` `new_fruits_enum``;`
```

```
+------------+
| fruit_name |
+------------+
| Pear       |
+------------+
1 row in set (0.00 sec)
```

在这里，不指定值会导致存储默认值`Pear`。

`SET( *value1* [, *value2* [, …]]) [CHARACTER SET *charset_name*] [COLLATE *collation_name*]`

这种类型存储一组字符串值。`SET`类型的列可以设置为列表`*value1*`、`*value2*`等中的零个或多个值，最多可达到 64 个不同的值。虽然这些值是字符串，但在数据库中存储的是整数表示。`SET`与`ENUM`不同之处在于每行只能在列中存储一个`ENUM`值，但可以存储多个`SET`值。这种类型适用于从列表中存储选择的选项，例如用户偏好。考虑以下使用水果名称的示例；名称可以是预定义值的任意组合：

```
mysql> `CREATE` `TABLE` `fruits_set`
    -> `(` `fruit_name` `SET``(``'Apple'``,` `'Orange'``,` `'Pear'``)` `)``;`
```

```
Query OK, 0 rows affected (0.08 sec)
```

```
mysql> `INSERT` `INTO` `fruits_set` `VALUES` `(``'Apple'``)``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

```
mysql> `INSERT` `INTO` `fruits_set` `VALUES` `(``'Banana'``)``;`
```

```
ERROR 1265 (01000): Data truncated for column 'fruit_name' at row 1
```

```
mysql> `INSERT` `INTO` `fruits_set` `VALUES` `(``'Apple,Orange'``)``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

```
mysql> `SELECT` `*` `FROM` `fruits_set``;`
```

```
+--------------+
| fruit_name   |
+--------------+
| Apple        |
| Apple,Orange |
+--------------+
2 rows in set (0.00 sec)
```

再次注意，我们可以在单个字段中存储来自集合的多个值，并且对于无效输入会存储空字符串。

与数值类型一样，我们建议始终选择最小可能的类型来存储值。例如，如果要存储城市名称，请使用`CHAR`或`VARCHAR`而不是`TEXT`类型。较短的列有助于减小表的大小，从而在服务器需要搜索表时提高性能。

使用固定大小的 `CHAR` 类型通常比使用可变大小的 `VARCHAR` 类型更快，因为 MySQL 服务器知道每行的起始和结束位置，并且可以快速跳过行以找到需要的行。然而，对于固定长度字段来说，未使用的空间将会被浪费。例如，如果允许城市名最多为 40 个字符，则 `CHAR(40)` 将始终使用 40 个字符，无论实际的城市名有多长。如果声明城市名为 `VARCHAR(40)`，则只会使用所需的空间，再加上 1 个字节来存储名称的长度。如果平均城市名长度为 10 个字符，这意味着使用可变长度字段每个条目将平均少占用 29 个字节的空间。如果要存储数百万个地址，这可能会产生很大的差异。

一般来说，如果存储空间有限或者预期字符串长度变化较大，请使用可变长度字段；如果性能是优先考虑的因素，请使用固定长度字段。

### 日期和时间类型

这些类型用于存储特定的时间戳、日期或时间范围。在处理时区时需要特别注意。我们将尽力解释细节，但当您真正需要处理时区时，建议重新阅读本节和文档。

`DATE`

用 `*YYYY-MM-DD*` 格式存储和显示日期范围为 1000-01-01 至 9999-12-31 的日期。日期必须始终以年、月、日的三元组输入，但输入的格式可以有所不同，如下例所示：

`*YYYY-MM-DD*` 或 `*YY-MM-DD*`

提供两位数年份或四位数年份都是可选的。我们强烈建议使用四位数版本以避免世纪混淆。实际上，如果使用两位数版本，您会发现 70 至 99 被解释为 1970 至 1999 年，而 00 至 69 被解释为 2000 至 2069 年。

`*YYYY/MM/DD*`, `*YYYY:MM:DD*`, `*YY-MM-DD*` 或其他带标点的格式

MySQL 允许使用任何标点符号来分隔日期的各个组成部分。我们建议使用破折号，并再次避免使用两位数的年份。

`*YYYY-M-D*`, `*YYYY-MM-D*` 或 `*YYYY-M-DD*`

当使用标点符号时（再次强调，允许使用任何标点符号），可以指定单个数字的天和月。例如，2006 年 2 月 2 日可以指定为 `2006-2-2`。同样提供了两位数年份的等效形式，但不建议使用。

`*YYYYMMDD*` 或 `*YYMMDD*`

这两种日期格式可以省略标点符号，但数字序列必须是六位或八位长度。

您还可以通过提供`DATETIME`和`TIMESTAMP`后面描述的格式来输入日期和时间的组合，但只有日期组件存储在`DATE`列中。无论输入类型如何，存储和显示类型始终为`*YYYY-MM-DD*`。*零日期* `0000-00-00`在所有版本中都被允许，并且可用于表示未知或虚拟值。如果输入日期超出范围，则存储零日期。但是，只有 MySQL 版本包括和之前的 5.6 默认设置允许此行为：`STRICT_TRANS_TABLES`、`NO_ZERO_DATE`和`NO_ZERO_IN_DATE`。

如果您正在使用较旧版本的 MySQL，我们建议您将这些模式添加到当前会话中：

```
mysql> `SET` `sql_mode``=``CONCAT``(``@``@``sql_mode``,`
    -> `',STRICT_TRANS_TABLES'``,`
    -> `',NO_ZERO_DATE'``,` `',NO_ZERO_IN_DATE'``)``;`
```

###### 提示

您还可以在全局服务器级别和配置文件中设置`sql_mode`变量。该变量必须列出您要启用的每个模式。

这里有一些使用 MySQL 8.0 默认设置插入日期的示例：

```
mysql> `CREATE` `TABLE` `testdate` `(``mydate` `DATE``)``;`
```

```
Query OK, 0 rows affected (0.00 sec)
```

```
mysql> `INSERT` `INTO` `testdate` `VALUES` `(``'2020/02/0'``)``;`
```

```
ERROR 1292 (22007): Incorrect date value: '2020/02/0'
for column 'mydate' at row 1
```

```
mysql> `INSERT` `INTO` `testdate` `VALUES` `(``'2020/02/1'``)``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

```
mysql> `INSERT` `INTO` `testdate` `VALUES` `(``'2020/02/31'``)``;`
```

```
ERROR 1292 (22007): Incorrect date value: '2020/02/31'
for column 'mydate' at row 1
```

```
mysql> `INSERT` `INTO` `testdate` `VALUES` `(``'2020/02/100'``)``;`
```

```
ERROR 1292 (22007): Incorrect date value: '2020/02/100'
for column 'mydate' at row 1
```

一旦执行`INSERT`语句，表将包含以下数据：

```
mysql> `SELECT` `*` `FROM` `testdate``;`
```

```
+------------+
| mydate     |
+------------+
| 2020-02-01 |
+------------+
1 row in set (0.00 sec)
```

MySQL 可以防止"坏"数据存储在您的表中。有时候您可能需要保留实际的输入并稍后手动处理它。您可以通过从`sql_mode`变量的模式列表中移除上述 SQL 模式来实现。在这种情况下，在运行前面的`INSERT`语句后，您将得到以下数据：

```
mysql> `SELECT` `*` `FROM` `testdate``;`
```

```
+------------+
| mydate     |
+------------+
| 2020-02-00 |
| 2020-02-01 |
| 0000-00-00 |
| 0000-00-00 |
+------------+
4 rows in set (0.01 sec)
```

再次注意，日期显示为`*YYYY-MM-DD*`格式，无论输入方式如何。

`TIME [*fraction*]`

以`*HHH:MM:SS*`格式存储时间范围为–838:59:59 至 838:59:59。这对于存储某些活动的持续时间非常有用。可以存储的值超出了 24 小时制时钟的范围，以允许计算和存储时间值之间的大差异（最多 34 天 22 小时 59 分钟 59 秒）。在`TIME`和其他相关数据类型中，`*fraction*`指定了分数秒精度范围为 0 至 6。默认值为 0，表示不保存分数秒。

时间必须按照*天*、*小时*、*分钟*、*秒*的顺序输入，使用以下格式：

`*DD HH:MM:SS[.fraction]*`、`*HH:MM:SS[.fraction]*`、`*DD HH:MM*`、`*HH:MM*`、`*DD HH*`或`*SS[.fraction]*`

`*DD*`表示 0 到 34 范围内的一位或两位数字的天值。`*DD*`值与小时值`*HH*`之间用空格分隔，而其他组件之间用冒号分隔。请注意，`*MM:SS*`不是有效的组合，因为它不能与`*HH:MM*`区分开来。如果`TIME`定义不指定`*fraction*`或将其设置为 0，则插入分数秒将导致值被四舍五入到最近的秒。

例如，如果在 `TIME` 列中插入 `2 13:25:58.999999`，带有 `*小数*` 为 0，则存储值为 `61:25:59`，因为 2 天（48 小时）加上 13 小时等于 61 小时。从 MySQL 5.7 开始，默认 SQL 模式设置禁止插入不正确的值。然而，可以启用旧版行为。然后，如果尝试插入超出范围的值，则会生成警告，并且该值被限制为最大可用时间。类似地，如果尝试插入无效值，则会生成警告，并且该值被设置为零。您可以使用 `SHOW WARNINGS` 命令报告上一个 SQL 语句生成的警告的详细信息。我们建议保持默认严格的 SQL 模式。与 `DATE` 类型不同，允许不正确的 `TIME` 条目似乎没有任何好处，除了在应用程序端更容易管理错误和保持旧版行为。

让我们在实践中尝试所有这些：

```
mysql> `CREATE` `TABLE` `test_time``(``id` `SMALLINT``,` `mytime` `TIME``)``;`
```

```
Query OK, 0 rows affected (0.00 sec)
```

```
mysql> `INSERT` `INTO` `test_time` `VALUES``(``1``,` `"2 13:25:59"``)``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

```
mysql> `INSERT` `INTO` `test_time` `VALUES``(``2``,` `"35 13:25:59"``)``;`
```

```
ERROR 1292 (22007): Incorrect time value: '35 13:25:59'
for column 'mytime' at row 1
```

```
mysql> `INSERT` `INTO` `test_time` `VALUES``(``3``,` `"900.32"``)``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

```
mysql> `SELECT` `*` `FROM` `test_time``;`
```

```
+------+----------+
| id   | mytime   |
+------+----------+
|    1 | 61:25:59 |
|    3 | 00:09:00 |
+------+----------+
2 rows in set (0.00 sec)
```

`*H:M:S*` 和单、双、三位数字组合

在插入或更新数据时，可以使用不同的数字组合；MySQL 将它们转换为内部时间格式并一致显示。例如，`1:1:3` 等同于 `01:01:03`。可以混合不同数量的数字；例如，`1:12:3` 等同于 `01:12:03`。考虑以下示例：

```
mysql> `CREATE` `TABLE` `mytime` `(``testtime` `TIME``)``;`
```

```
Query OK, 0 rows affected (0.12 sec)
```

```
mysql> `INSERT` `INTO` `mytime` `VALUES`
    -> `(``'-1:1:1'``)``,` `(``'1:1:1'``)``,`
    -> `(``'1:23:45'``)``,` `(``'123:4:5'``)``,`
    -> `(``'123:45:6'``)``,` `(``'-123:45:6'``)``;`
```

```
Query OK, 4 rows affected (0.00 sec)
Records: 4  Duplicates: 0  Warnings: 0
```

```
mysql> `SELECT` `*` `FROM` `mytime``;`
```

```
+------------+
| testtime   |
+------------+
|  -01:01:01 |
|   01:01:01 |
|   01:23:45 |
|  123:04:05 |
|  123:45:06 |
| -123:45:06 |
+------------+
5 rows in set (0.01 sec)
```

注意，小时显示为两位数，范围在 -99 到 99 之间。

`*HHMMSS*`、`*MMSS*` 和 `*SS*`

标点可以省略，但数字序列必须是两位、四位或六位数字长度。注意，最右边的数字对始终被解释为 `*SS*`（秒）值，右数第二对（如果存在）被解释为 `*MM*`（分钟），右数第三对（如果存在）被解释为 `*HH*`（小时）。结果是，例如 `1222` 被解释为 12 分钟 22 秒，而不是 12 小时 22 分。

您还可以通过提供 `DATETIME` 和 `TIMESTAMP` 的描述格式的日期和时间来输入时间，但只有时间部分存储在 `TIME` 列中。无论输入类型如何，存储和显示类型始终为 `*HH:MM:SS*`。*零时间* `00:00:00` 可用于表示未知或虚拟值。

`时间戳[（*小数*）]`

存储和显示格式为 `*YYYY-MM-DD HH:MM:SS[.fraction][time zone offset]*` 的日期和时间对，范围从 1970-01-01 00:00:01.000000 到 `2038-01-19 03:14:07.999999`。这种类型与 `DATETIME` 类型非常相似，但有一些区别。两种类型都可以接受一个时区修饰符作为输入值 MySQL 8.0，并且两种类型将以相同的方式存储和呈现数据给同一时区内的任何客户端。但是，`TIMESTAMP` 列中的值始终在 UTC 时区内部存储，使得在处理不同时区的客户端时可以自动获取本地时区。这本身是一个非常重要的区别需要记住。可以说，`TIMESTAMP` 在处理不同时区时更方便使用。

在 MySQL 5.6 之前，仅 `TIMESTAMP` 类型支持自动初始化和更新。此外，每个给定表中只能有一个这样的列。但是，从 5.6 开始，`TIMESTAMP` 和 `DATETIME` 都支持这些行为，并且可以有任意数量的列执行此操作。

存储在 `TIMESTAMP` 列中的值始终匹配模板 `*YYYY-MM-DD HH:MM:SS[.fraction][time zone offset]*`，但可以以多种格式提供值：

`*YYYY-MM-DD HH:MM:SS*` 或 `*YY-MM-DD HH:MM:SS*`

日期和时间组件遵循与之前描述的`DATE`和`TIME`组件相同的宽松限制。这包括允许任何标点字符，包括（与`TIME`不同）时间组件中使用标点灵活性。例如，`0` 是有效的。

`*YYYYMMDDHHMMSS*` 或 `*YYMMDDHHMMSS*`

可以省略标点符号，但字符串的长度应为 12 或 14 位数字。出于与 `DATE` 类型相同的原因，我们建议仅使用明确的 14 位版本。您可以指定其他长度的值而不提供分隔符，但我们不建议这样做。

让我们更详细地看看自动更新功能。你可以通过在创建表时或稍后修改时，向列定义中添加以下属性来控制此功能，我们将在“修改结构”中详细解释：

1.  如果你希望时间戳仅在将新行插入表中时设置，可以在列声明的末尾添加`DEFAULT CURRENT_TIMESTAMP`。

1.  如果你不希望有默认时间戳，但希望每当行中的数据更新时使用当前时间，可以在列声明的末尾添加`ON UPDATE CURRENT_TIMESTAMP`。

1.  如果你希望以上两者同时实现——即，希望时间戳在每个新行中设为当前时间，并在修改现有行时也设为当前时间——在列声明的末尾添加`DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`。

如果对于`TIMESTAMP`列不指定`DEFAULT NULL`或`NULL`，它将以`0`作为默认值。

`YEAR[(4)]`

存储一个位于 1901 到 2155 年之间的四位数年份，以及 *零年*，0000。非法值会被转换为零年。您可以将年份值输入为字符串（如 `'2005'`）或整数（如 `2005`）。`YEAR` 类型需要 1 字节的存储空间。

在早期的 MySQL 版本中，可以指定`*digits*`参数，传递`2`或`4`。两位数版本存储的值从 70 到 69，表示 1970 到 2069 年。MySQL 8.0 不支持两位数的 `YEAR` 类型，并且为了显示目的而指定 `*digits*` 参数已被弃用。

`DATETIME[(*fraction*)]`

以`*YYYY-MM-DD HH:MM:SS[.fraction][time zone offset]*`格式存储和显示日期和时间对，范围从`1000-01-01 00:00:00`到`9999-12-31 23:59:59`。对于 `TIMESTAMP`，存储的值总是匹配模板`*YYYY-MM-DD HH:MM:SS*`，但值可以以 `TIMESTAMP` 描述中列出的相同格式输入。如果只向 `DATETIME` 列分配日期，则假定零时间`00:00:00`。如果只向 `DATETIME` 列分配时间，则假定零日期`0000-00-00`。该类型具有与 `TIMESTAMP` 相同的自动更新特性。除非为 `DATETIME` 列指定 `NOT NULL` 属性，否则 `NULL` 值为默认值；否则，默认值为 `0`。与 `TIMESTAMP` 不同，`DATETIME` 值不会转换为 UTC 时区进行存储。

### 其他类型

到目前为止，截至 MySQL 8.0，空间和 `JSON` 数据类型归入这一广泛类别。使用这些是一个非常高级的主题，我们不会深入讨论它们。

空间数据类型涉及存储几何对象，MySQL 有与 OpenGIS 类对应的类型。处理这些类型是一个值得单独撰写一本书的主题。

`JSON` 数据类型允许原生存储有效的 JSON 文档。在 MySQL 5.7 之前，JSON 通常存储在 `TEXT` 或类似的列中。然而，这有许多缺点：例如，文档不会被验证，也不会进行存储优化（所有 JSON 只是以文本形式存储）。使用原生的 `JSON` 类型，则以二进制格式存储。如果我们要总结成一句话：亲爱的读者，请为 JSON 使用 `JSON` 数据类型。

## 键和索引

您会发现，您使用的几乎所有表都在其 `CREATE TABLE` 语句中声明了 `PRIMARY KEY` 子句，有时还有多个 `KEY` 子句。为什么需要主键和次要键的原因在 第二章 中已经讨论过。本节讨论了如何声明主键，当您这样做时背后发生的事情，以及为什么您可能还希望在您的数据上创建其他键和索引。

*主键* 在表中唯一标识每一行。更重要的是，对于默认的 InnoDB 存储引擎，主键也用作*聚集索引*。这意味着所有实际的表数据都存储在一个索引结构中。这与 MyISAM 不同，后者将数据和索引分开存储。当表使用聚集索引时，称为聚集表。在聚集表中，每一行都存储在一个索引内，而不是通常所说的*堆*中。对表进行聚集将导致其行根据聚集索引的顺序排序，并实际物理存储在该索引的叶子页中。每个表不能有多于一个聚集索引。对于这样的表，二级索引引用聚集索引中的记录，而不是实际的表行。这通常会提高查询性能，尽管可能对写入性能有所损害。InnoDB 不允许您在聚集和非聚集表之间进行选择；这是您无法更改的设计决策。

主键通常是任何数据库设计的推荐部分，但对于 InnoDB 来说是必需的。事实上，如果在创建 InnoDB 表时不指定 `PRIMARY KEY` 子句，MySQL 将使用第一个 `UNIQUE NOT NULL` 列作为聚集索引的基础。如果没有这样的列可用，则创建一个隐藏的聚集索引，基于由 InnoDB 分配给每行的 ID 值。

鉴于 InnoDB 是 MySQL 的默认存储引擎并且是当今的事实标准，我们将在本章集中讨论其行为。备选存储引擎如 MyISAM、MEMORY 或 MyRocks 将在 “备选存储引擎” 中进行讨论。

如前所述，当定义主键时，它将成为一个聚集索引，表中的所有数据都存储在该索引的叶子块中。InnoDB 使用 B 树索引（更具体地说是 B+ 树变体），除了空间数据类型的索引使用 R 树结构。其他存储引擎可能实现不同类型的索引，但如果未指定表的存储引擎，则可以假定所有索引都是 B 树。

有一个聚集索引，或者换句话说，有索引组织的表，可以加快涉及主键列的查询和排序。然而，一个缺点是修改主键列是昂贵的。因此，一个好的设计将需要一个基于经常用于查询过滤但很少修改的列的主键。请记住，如果根本没有主键，InnoDB 将使用隐式的聚集索引；因此，如果你不确定要选择哪些列作为主键，考虑使用类似`id`的合成列。例如，`SERIAL` 数据类型在这种情况下可能非常合适。

离开 InnoDB 的内部细节，当您在 MySQL 中为表声明主键时，它会创建一个存储有关表中每行数据存储位置信息的结构。此信息称为*索引*，其目的是加快使用主键进行搜索的速度。例如，在`sakila`数据库的`actor`表中声明`PRIMARY KEY (actor_id)`时，MySQL 创建一个结构，允许它非常快速地找到与特定`actor_id`（或一系列标识符）匹配的行。

这对于匹配演员与电影或电影与类别非常有用。可以使用`SHOW INDEX`（或`SHOW INDEXES`）命令显示表上可用的索引：

```
mysql> `SHOW` `INDEX` `FROM` `category``\``G`
```

```
*************************** 1\. row ***************************
        Table: category
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: category_id
    Collation: A
  Cardinality: 16
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE
      Comment:
Index_comment:
      Visible: YES
   Expression: NULL
1 row in set (0.00 sec)
```

*基数*是索引中唯一值的数量；对于主键索引，这与表中行的数量相同。

请注意，所有作为主键一部分的列都必须声明为`NOT NULL`，因为它们必须具有值才能使行有效。如果没有索引，查找表中的行的唯一方法是从磁盘读取每一行，并检查它是否与您正在搜索的`category_id`匹配。对于行数很多的表，这种详尽的顺序搜索非常慢。但是，您不能只是索引一切；我们将在本节末尾回到这一点。

您可以在表中的数据上创建其他索引。这样做是为了使其他搜索（无论是在其他列上还是在列的组合上）都能快速进行，并避免顺序扫描。例如，再次看看`actor`表。除了在`actor_id`上有主键之外，还在`last_name`上有一个辅助键，以改善按演员姓氏搜索的效率：

```
mysql> `SHOW` `CREATE` `TABLE` `actor``\``G`
```

```
*************************** 1\. row ***************************
       Table: actor
Create Table: CREATE TABLE `actor` (
  `actor_id` smallint unsigned NOT NULL AUTO_INCREMENT,
  ...
  `last_name` varchar(45) NOT NULL,
  ...
  PRIMARY KEY (`actor_id`),
  KEY `idx_actor_last_name` (`last_name`)
) ...
1 row in set (0.00 sec)
```

您可以看到关键字`KEY`用于告诉 MySQL 需要额外的索引。或者，您可以在`KEY`的位置使用`INDEX`。在关键字之后是索引名称，然后是包含在括号中的索引列。您也可以在创建表后添加索引——实际上，您几乎可以更改表的任何内容。这在“修改结构”中讨论。

可以在多个列上建立索引。例如，考虑以下来自`sakila`的修改后的表：

```
mysql> `CREATE` `TABLE` `customer_mod` `(`
    -> `customer_id` `smallint` `unsigned` `NOT` `NULL` `AUTO_INCREMENT``,`
    -> `first_name` `varchar``(``45``)` `NOT` `NULL``,`
    -> `last_name` `varchar``(``45``)` `NOT` `NULL``,`
    -> `email` `varchar``(``50``)` `DEFAULT` `NULL``,`
    -> `PRIMARY` `KEY` `(``customer_id``)``,`
    -> `KEY` `idx_names_email` `(``first_name``,` `last_name``,` `email``)``)``;`
```

```
Query OK, 0 rows affected (0.06 sec)
```

您可以看到，我们在`customer_id`标识符列上添加了主键索引，还添加了另一个名为`idx_names_email`的索引，按照`first_name`、`last_name`和`email`列的顺序包含在内。现在让我们考虑如何使用这个额外的索引。

您可以利用`idx_names_email`索引快速搜索三个名称列的组合。例如，在以下查询中非常有用：

```
mysql> `SELECT` `*` `FROM` `customer_mod` `WHERE`
    -> `first_name` `=` `'Rose'` `AND`
    -> `last_name` `=` `'Williams'` `AND`
    -> `email` `=` `'rose.w@nonexistent.edu'``;`
```

我们知道它有助于搜索，因为索引中列出的所有列都在查询中使用。您可以使用`EXPLAIN`语句检查您认为应该发生的事情是否确实发生了：

```
mysql> `EXPLAIN` `SELECT` `*` `FROM` `customer_mod` `WHERE`
    -> `first_name` `=` `'Rose'` `AND`
    -> `last_name` `=` `'Williams'` `AND`
    -> `email` `=` `'rose.w@nonexistent.edu'``\``G`
```

```
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: customer_mod
   partitions: NULL
         type: ref
possible_keys: idx_names_email
          key: idx_names_email
      key_len: 567
          ref: const,const,const
         rows: 1
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
```

您可以看到 MySQL 报告的 `possible_keys` 是 `idx_names_email`（这意味着该索引可以用于此查询），它决定使用的 `key` 是 `idx_names_email`。所以，您期望的和实际发生的是一样的，这是个好消息！您将在第七章中了解更多关于 `EXPLAIN` 语句的内容。

我们创建的索引还可用于仅涉及 `first_name` 列的查询。例如，它可用于以下查询：

```
mysql> `SELECT` `*` `FROM` `customer_mod` `WHERE`
    -> `first_name` `=` `'Rose'``;`
```

您可以再次使用 `EXPLAIN` 来检查索引是否被使用。它可以被使用的原因是 `first_name` 列是索引中列出的第一列。在实践中，这意味着索引会将所有具有相同名字的人的行信息*聚集*或存储在一起，因此可以使用索引来查找任何具有匹配名字的人。

索引还可以用于涉及名字和姓氏组合的搜索，原因与刚刚讨论的相同。索引将具有相同名字的人聚集在一起，并将相同名字的人按姓氏聚集在一起。因此，它可用于此查询：

```
mysql> `SELECT` `*` `FROM` `customer_mod` `WHERE`
    -> `first_name` `=` `'Rose'` `AND`
    -> `last_name` `=` `'Williams'``;`
```

但是，此查询无法使用索引，因为索引中最左列 `first_name` 在查询中不存在：

```
mysql> `SELECT` `*` `FROM` `customer_mod` `WHERE`
    -> `last_name` `=` `'Williams'` `AND`
    -> `email` `=` `'rose.w@nonexistent.edu'``;`
```

索引应帮助缩小结果集，使其变为可能的答案的更小集合。要使 MySQL 能够使用索引，查询必须同时满足以下两个条件：

1.  在查询中必须包含在 `KEY`（或 `PRIMARY KEY`）子句中列出的最左列。

1.  查询中不能包含未建立索引的列的 `OR` 条件。

同样，您始终可以使用 `EXPLAIN` 语句来检查特定查询是否可以使用索引。

在完成本节之前，以下是一些关于选择和设计索引的想法。在考虑添加索引时，请考虑以下几点：

+   索引占用磁盘空间，并且在数据更改时需要更新。如果您的数据频繁更改，或者在进行更改时更改了大量数据，索引将减慢这一过程。然而，实际上，由于 `SELECT` 语句（数据读取）通常比其他语句（数据修改）更常见，索引通常是有益的。

+   只添加经常使用的索引。在确定用户和应用程序需要的查询之前，不要为列添加索引。之后可以随时添加索引。

+   如果索引中的所有列都在所有查询中使用，则列出具有最高重复项数的列，放在 `KEY` 子句的最左边。这样可以最小化索引大小。

+   索引越小，速度越快。如果对大列建立索引，索引也会变大。这就是设计表时确保列尽可能小的一个好理由。

+   对于长列，您可以仅使用列中的前缀创建索引。您可以通过在列定义后面加上括号中的值来实现此目的，例如`KEY idx_names_email (first_name(3), last_name(2), email(10))`。这意味着仅对`first_name`的前 3 个字符进行索引，然后是`last_name`的前 2 个字符，最后是`email`的 10 个字符。与从三列中索引 140 个字符相比，这节省了大量空间！这样做会使您的索引无法唯一标识行，但它会变得更小，仍然可以有效地找到匹配的行。对于像`TEXT`这样的长类型，使用前缀是强制的。

结束本节时，我们需要讨论 InnoDB 中关于次要键的一些特殊情况。请记住，所有表数据都存储在聚集索引的叶子节点中。这意味着，使用`actor`示例，如果我们需要通过`last_name`进行过滤时获取`first_name`数据，即使我们可以使用`idx_actor_last_name`进行快速过滤，我们仍需要通过主键访问数据。因此，在 InnoDB 中，每个次要键的定义隐式地附加了所有主键列。因此，在 InnoDB 中不必要地长主键会导致次要键显著膨胀。

这一点在`EXPLAIN`输出中也可以看到（注意第一个命令的第一个输出中的`Extra: Using index`）：

```
mysql> `EXPLAIN` `SELECT` `actor_id``,` `last_name` `FROM` `actor` `WHERE` `last_name` `=` `'Smith'``\``G`
```

```
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: actor
   partitions: NULL
         type: ref
possible_keys: idx_actor_last_name
          key: idx_actor_last_name
      key_len: 182
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
```

```
mysql> `EXPLAIN` `SELECT` `first_name` `FROM` `actor` `WHERE` `last_name` `=` `'Smith'``\``G`
```

```
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: actor
   partitions: NULL
         type: ref
possible_keys: idx_actor_last_name
          key: idx_actor_last_name
      key_len: 182
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

事实上，`idx_actor_last_name`是第一个查询的*覆盖索引*，这意味着 InnoDB 可以仅从该索引中提取所有所需的数据。但是，对于第二个查询，InnoDB 将需要额外查找聚集索引以获取`first_name`列的值。

## 自增特性

MySQL 的专有`AUTO_INCREMENT`特性允许您为行创建唯一标识符，而无需运行`SELECT`查询。它是如何工作的？让我们再次看看简化的`actor`表：

```
mysql> `CREATE` `TABLE` `actor` `(`
    -> `actor_id` `smallint` `unsigned` `NOT` `NULL` `AUTO_INCREMENT``,`
    -> `first_name` `varchar``(``45``)` `NOT` `NULL``,`
    -> `last_name` `varchar``(``45``)` `NOT` `NULL``,`
    -> `PRIMARY` `KEY` `(``actor_id``)`
    -> `)``;`
```

```
Query OK, 0 rows affected (0.03 sec)
```

可以在不指定`actor_id`的情况下向该表中插入行：

```
mysql> `INSERT` `INTO` `actor` `VALUES` `(``NULL``,` `'Alexander'``,` `'Kaidanovsky'``)``;`
```

```
Query OK, 1 row affected (0.01 sec)
```

```
mysql> `INSERT` `INTO` `actor` `VALUES` `(``NULL``,` `'Anatoly'``,` `'Solonitsyn'``)``;`
```

```
Query OK, 1 row affected (0.01 sec)
```

```
mysql> `INSERT` `INTO` `actor` `VALUES` `(``NULL``,` `'Nikolai'``,` `'Grinko'``)``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

当您查看此表中的数据时，可以看到每行为`actor_id`列分配了一个值：

```
mysql> `SELECT` `*` `FROM` `actor``;`
```

```
+----------+------------+-------------+
| actor_id | first_name | last_name   |
+----------+------------+-------------+
|        1 | Alexander  | Kaidanovsky |
|        2 | Anatoly    | Solonitsyn  |
|        3 | Nikolai    | Grinko      |
+----------+------------+-------------+
3 rows in set (0.00 sec)
```

每次插入新行时，都会为`actor_id`列创建一个唯一值。

考虑这个特性是如何工作的。您可以看到，`actor_id`列被声明为带有`NOT NULL AUTO_INCREMENT`子句的整数。`AUTO_INCREMENT`告诉 MySQL，当没有为此列提供值时，分配的值应该比当前表中存储的最大值大一。对于空表，`AUTO_INCREMENT`序列从 1 开始。

对于`AUTO_INCREMENT`列，必须使用`NOT NULL`子句；当插入`NULL`（或 0，尽管不建议这样做）时，MySQL 服务器会自动查找下一个可用的标识符，并将其分配给新行。如果列未定义为`UNSIGNED`，则可以手动插入负值；然而，对于下一个自动增量，MySQL 将简单地使用列中的最大（正）值，或者如果没有正值，则从 1 开始。

`AUTO_INCREMENT`特性有以下要求：

+   它所用的列必须被索引。

+   它所用的列不能有默认值。

+   每个表只能有一个`AUTO_INCREMENT`列。

MySQL 支持不同的存储引擎；我们将在“备用存储引擎”中详细讨论这些内容。在使用非默认的 MyISAM 表类型时，可以在由多列组成的键上使用`AUTO_INCREMENT`特性。实际上，您可以在单个`AUTO_INCREMENT`列内拥有多个独立的计数器。然而，在 InnoDB 中是不可能的。

虽然`AUTO_INCREMENT`特性很有用，但它在其他数据库环境中并不具备可移植性，并且隐藏了创建新标识符的逻辑步骤。它还可能导致歧义；例如，删除或截断表会重置计数器，但使用`WHERE`子句删除选定行则不会重置计数器。此外，如果在事务中插入了一行但然后回滚该事务，则标识符仍然会被使用。举个例子，让我们创建包含自动增量字段`counter`的表`count`：

```
mysql> `CREATE` `TABLE` `count` `(``counter` `INT` `AUTO_INCREMENT` `KEY``)``;`
```

```
Query OK, 0 rows affected (0.13 sec)
```

```
mysql> `INSERT` `INTO` `count` `VALUES` `(``)``,``(``)``,``(``)``,``(``)``,``(``)``,``(``)``;`
```

```
Query OK, 6 rows affected (0.01 sec)
Records: 6  Duplicates: 0  Warnings: 0
```

```
mysql> `SELECT` `*` `FROM` `count``;`
```

```
+---------+
| counter |
+---------+
| 1       |
| 2       |
| 3       |
| 4       |
| 5       |
| 6       |
+---------+
6 rows in set (0.00 sec)
```

插入多个值的工作效果如预期。现在，让我们删除几行，然后添加六行新数据：

```
mysql> `DELETE` `FROM` `count` `WHERE` `counter` `>` `4``;`
```

```
Query OK, 2 rows affected (0.00 sec)
```

```
mysql> `INSERT` `INTO` `count` `VALUES` `(``)``,``(``)``,``(``)``,``(``)``,``(``)``,``(``)``;`
```

```
Query OK, 6 rows affected (0.00 sec)
Records: 6  Duplicates: 0  Warnings: 0
```

```
mysql> `SELECT` `*` `FROM` `count``;`
```

```
+---------+
| counter |
+---------+
| 1       |
| 2       |
| 3       |
| 4       |
| 7       |
| 8       |
| 9       |
| 10      |
| 11      |
| 12      |
+---------+
10 rows in set (0.00 sec)
```

在这里，我们看到计数器未被重置，并继续从 7 开始。然而，如果我们截断表，从而删除所有数据，计数器将被重置为 1：

```
mysql> `TRUNCATE` `TABLE` `count``;`
```

```
Query OK, 0 rows affected (0.00 sec)
```

```
mysql> `INSERT` `INTO` `count` `VALUES` `(``)``,``(``)``,``(``)``,``(``)``,``(``)``,``(``)``;`
```

```
Query OK, 6 rows affected (0.01 sec)
Records: 6  Duplicates: 0  Warnings: 0
```

```
mysql> `SELECT` `*` `FROM` `count``;`
```

```
+---------+
| counter |
+---------+
| 1       |
| 2       |
| 3       |
| 4       |
| 5       |
| 6       |
+---------+
6 rows in set (0.00 sec)
```

总结一下：`AUTO_INCREMENT`保证了事务性和单调递增值序列。然而，它并不以任何方式保证每个提供的单个标识符会严格遵循前一个标识符。通常，`AUTO_INCREMENT`的这种行为已经足够清晰，不应该是一个问题。然而，如果您的特定用例要求计数器保证没有间隙，您应该考虑使用某种解决方法。不幸的是，这可能会在应用程序端实现。

# 修改结构

我们已经向您展示了创建数据库、表、索引和列所需的所有基础知识。在本节中，您将学习如何在已经存在的结构中添加、删除和更改列、数据库、表和索引。

## 添加、删除和更改列

您可以使用`ALTER TABLE`语句向表中添加新列，删除现有列，并更改列名、类型和长度。

让我们从考虑如何修改现有列开始。考虑一个示例，我们将重命名表列。`language`表有一个名为`last_update`的列，其中包含记录修改时间。要将此列名更改为`last_updated_time`，您应该编写：

```
mysql> `ALTER` `TABLE` `language` `RENAME` `COLUMN` `last_update` `TO` `last_updated_time``;`
```

```
Query OK, 0 rows affected (0.03 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

这个特定示例利用了 MySQL 的*在线 DDL*功能。实际上背后发生的是 MySQL 仅修改了元数据，而不需要以任何方式重写表。通过受影响行数的缺失可以看出这一点。并非所有 DDL 语句都可以在线执行，因此在许多您进行的更改中可能不会出现这种情况。

###### 注意

DDL 代表数据定义语言，在 SQL 的上下文中，它是用于创建、修改和删除模式对象（如数据库、表、索引和列）的语法和语句的子集。例如，`CREATE TABLE`和`ALTER TABLE`都是 DDL 操作。

执行 DDL 语句需要特殊的内部机制，包括特殊的锁定——这是一件好事，因为您可能不希望在运行查询时表发生变化！这些特殊锁在 MySQL 中称为元数据锁，我们在“元数据锁”中详细介绍了它们的工作原理。

注意所有 DDL 语句，包括通过在线 DDL 执行的语句，都需要获取元数据锁。从这个意义上说，在线 DDL 语句并不那么“在线”，但它们在运行时不会完全锁定目标表。在负载运行的系统上执行 DDL 语句是一次冒险：即使是应该几乎立即执行的语句，也可能造成严重破坏。我们建议您仔细阅读第六章中关于元数据锁的内容以及[MySQL 文档](https://oreil.ly/xNZYg)中的链接，并尝试在有和无并发负载的情况下运行不同的 DDL 语句。在学习 MySQL 时，这可能并不是太重要，但我们认为提前告诫您是值得的。说了这些，让我们回到我们对`language`表进行的`ALTER`操作。

您可以使用`SHOW COLUMNS`语句检查结果：

```
mysql> `SHOW` `COLUMNS` `FROM` `language``;`
```

```
+-------------------+------------------+------+-----+-------------------+...
| Field             | Type             | Null | Key | Default           |...
+-------------------+------------------+------+-----+-------------------+...
| language_id       | tinyint unsigned | NO   | PRI | NULL              |...
| name              | char(20)         | NO   |     | NULL              |...
| last_updated_time | timestamp        | NO   |     | CURRENT_TIMESTAMP |...
+-------------------+------------------+------+-----+-------------------+...
3 rows in set (0.01 sec)
```

在前面的示例中，我们使用了带有`RENAME COLUMN`关键字的`ALTER TABLE`语句。这是 MySQL 8.0 的功能。为了兼容性，我们也可以使用带有`CHANGE`关键字的`ALTER TABLE`：

```
mysql> `ALTER` `TABLE` `language` `CHANGE` `last_update` `last_updated_time` `TIMESTAMP`
    -> `NOT` `NULL` `DEFAULT` `CURRENT_TIMESTAMP` `ON` `UPDATE` `CURRENT_TIMESTAMP``;`
```

```
Query OK, 0 rows affected (0.04 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

在此示例中，您可以看到我们向带有`CHANGE`关键字的`ALTER TABLE`语句提供了四个参数：

1.  表名，`language`

1.  原始列名，`last_update`

1.  新列名，`last_updated_time`

1.  列类型，`TIMESTAMP`，带有许多额外的属性，这些属性是必需的，以避免改变原始定义

您必须提供所有四个参数；这意味着您需要重新指定类型和任何相关的子句。在这个例子中，由于我们使用的是默认设置的 MySQL 8.0，`TIMESTAMP`不再具有显式的默认值。如您所见，使用`RENAME COLUMN`比`CHANGE`要简单得多。

如果您想要修改列的类型和子句，但不修改其名称，可以使用`MODIFY`关键字：

```
mysql> `ALTER` `TABLE` `language` `MODIFY` `name` `CHAR``(``20``)` `DEFAULT` `'n/a'``;`
```

```
Query OK, 0 rows affected (0.14 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

您还可以使用`CHANGE`关键字，但需指定相同的列名两次：

```
mysql> `ALTER` `TABLE` `language` `CHANGE` `name` `name` `CHAR``(``20``)` `DEFAULT` `'n/a'``;`
```

```
Query OK, 0 rows affected (0.03 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

修改类型时请小心：

+   不要使用不兼容的类型，因为您依赖 MySQL 成功地将数据从一种格式转换为另一种格式（例如，将`INT`列转换为`DATETIME`列可能不会达到您的预期效果）。

+   除非您需要这样做，否则不要截断数据。如果缩小类型的大小，值将被编辑以匹配新的宽度，可能会丢失数据。

假设您想要向现有表添加额外的列。以下是使用`ALTER TABLE`语句的方法：

```
mysql> `ALTER` `TABLE` `language` `ADD` `native_name` `CHAR``(``20``)``;`
```

```
Query OK, 0 rows affected (0.04 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

您必须提供`ADD`关键字、新列名以及列类型和子句。此示例将新列`native_name`添加为表中的最后一列，如`SHOW COLUMNS`语句所示：

```
mysql> `SHOW` `COLUMNS` `FROM` `artist``;`
```

```
+-------------------+------------------+------+-----+-------------------+...
| Field             | Type             | Null | Key | Default           |...
+-------------------+------------------+------+-----+-------------------+...
| language_id       | tinyint unsigned | NO   | PRI | NULL              |...
| name              | char(20)         | YES  |     | n/a               |...
| last_updated_time | timestamp        | NO   |     | CURRENT_TIMESTAMP |...
| native_name       | char(20)         | YES  |     | NULL              |...
+-------------------+------------------+------+-----+-------------------+...
4 rows in set (0.00 sec)
```

如果您希望它作为第一列，使用`FIRST`关键字如下：

```
mysql> `ALTER` `TABLE` `language` `ADD` `native_name` `CHAR``(``20``)` `FIRST``;`
```

```
Query OK, 0 rows affected (0.08 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

```
mysql> `SHOW` `COLUMNS` `FROM` `language``;`
```

```
+-------------------+------------------+------+-----+-------------------+...
| Field             | Type             | Null | Key | Default           |...
+-------------------+------------------+------+-----+-------------------+...
| native_name       | char(20)         | YES  |     | NULL              |...
| language_id       | tinyint unsigned | NO   | PRI | NULL              |...
| name              | char(20)         | YES  |     | n/a               |...
| last_updated_time | timestamp        | NO   |     | CURRENT_TIMESTAMP |...
+-------------------+------------------+------+-----+-------------------+...
4 rows in set (0.01 sec)
```

如果您希望它添加到特定位置，请使用`AFTER`关键字：

```
mysql> `ALTER` `TABLE` `language` `ADD` `native_name` `CHAR``(``20``)` `AFTER` `name``;`
```

```
Query OK, 0 rows affected (0.08 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

```
mysql> `SHOW` `COLUMNS` `FROM` `language``;`
```

```
+-------------------+------------------+------+-----+-------------------+...
| Field             | Type             | Null | Key | Default           |...
+-------------------+------------------+------+-----+-------------------+...
| language_id       | tinyint unsigned | NO   | PRI | NULL              |...
| name              | char(20)         | YES  |     | n/a               |...
| native_name       | char(20)         | YES  |     | NULL              |...
| last_updated_time | timestamp        | NO   |     | CURRENT_TIMESTAMP |...
+-------------------+------------------+------+-----+-------------------+...
4 rows in set (0.00 sec)
```

要删除列，请使用`DROP`关键字，后跟列名。以下是如何去除新添加的`native_name`列的方法：

```
mysql> `ALTER` `TABLE` `language` `DROP` `native_name``;`
```

```
Query OK, 0 rows affected (0.07 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

这将删除列结构及其中包含的任何数据。它还将从包含该列的任何索引中删除列；如果它是索引中的唯一列，则也会删除索引。如果一张表中只有一列，则无法删除该列；此时应删除整个表，如“删除结构”中所述。删除列时要小心，因为表结构更改时，通常需要修改任何用于按特定顺序插入值的`INSERT`语句。有关更多信息，请参阅“INSERT 语句”。

MySQL 允许您在单个`ALTER TABLE`语句中指定多个修改，通过逗号分隔它们。以下是一个示例，添加新列并调整另一个列：

```
mysql> `ALTER` `TABLE` `language` `ADD` `native_name` `CHAR``(``255``)``,` `MODIFY` `name` `CHAR``(``255``)``;`
```

```
Query OK, 6 rows affected (0.06 sec)
Records: 6  Duplicates: 0  Warnings: 0
```

请注意，这次您可以看到已更改了六条记录。在先前的`ALTER TABLE`命令中，MySQL 报告未影响任何行。不同之处在于，这次我们未执行在线 DDL 操作，因为更改任何列的类型将始终导致表被重建。我们建议在计划更改时阅读有关[在线 DDL 操作](https://oreil.ly/EEOw0)的参考手册。组合在线和离线操作将导致离线操作。

在不使用在线 DDL 或任何修改是“离线”的情况下，将多个修改操作合并为单个操作非常高效。这样做可能会节省创建新表、将数据从旧表复制到新表、删除旧表并将新表重命名为旧表名的成本。

## 添加、删除和更改索引

正如我们之前讨论的，通常很难在构建应用程序之前知道哪些索引是有用的。您可能会发现应用程序的某个特定功能比预期更受欢迎，这会导致您评估如何改进相关查询的性能。因此，您会发现在应用程序部署后能够动态添加、修改和删除索引非常有用。本节将向您展示如何操作。请注意，修改索引不会影响表中存储的数据。

我们将从添加新索引开始。假设经常使用`language`表，并使用指定`name`的`WHERE`子句进行查询。为了加快这些查询的速度，您决定添加一个名为`idx_name`的新索引。以下是如何在创建表后添加它的方法：

```
mysql> `ALTER` `TABLE` `language` `ADD` `INDEX` `idx_name` `(``name``)``;`
```

```
Query OK, 0 rows affected (0.05 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

同样，您可以交替使用`KEY`和`INDEX`这两个术语。您可以使用`SHOW CREATE TABLE`语句检查结果：

```
mysql> `SHOW` `CREATE` `TABLE` `language``\``G`
```

```
*************************** 1\. row ***************************
       Table: language
Create Table: CREATE TABLE `language` (
  `language_id` tinyint unsigned NOT NULL AUTO_INCREMENT,
  `name` char(255) DEFAULT NULL,
  `last_updated_time` timestamp NOT NULL
    DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`language_id`),
  KEY `idx_name` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=8
    DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

新索引如预期地成为表结构的一部分。在创建表后，您还可以为表指定主键：

```
mysql> `CREATE` `TABLE` `no_pk` `(``id` `INT``)``;`
```

```
Query OK, 0 rows affected (0.02 sec)
```

```
mysql> `INSERT` `INTO` `no_pk` `VALUES` `(``1``)``,``(``2``)``,``(``3``)``;`
```

```
Query OK, 3 rows affected (0.01 sec)
Records: 3  Duplicates: 0  Warnings: 0
```

```
mysql> `ALTER` `TABLE` `no_pk` `ADD` `PRIMARY` `KEY` `(``id``)``;`
```

```
Query OK, 0 rows affected (0.13 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

现在让我们考虑如何删除索引。要删除非主键索引，执行以下操作：

```
mysql> `ALTER` `TABLE` `language` `DROP` `INDEX` `idx_name``;`
```

```
Query OK, 0 rows affected (0.08 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

删除主键索引的方法如下：

```
mysql> `ALTER` `TABLE` `no_pk` `DROP` `PRIMARY` `KEY`;
```

```
Query OK, 3 rows affected (0.07 sec)
Records: 3  Duplicates: 0  Warnings: 0
```

MySQL 不允许在表中有多个主键。如果要更改主键，必须先删除现有的索引，然后再添加新的。不过，我们知道可以将 DDL 操作进行分组。考虑以下示例：

```
mysql> `ALTER` `TABLE` `language` `DROP` `PRIMARY` `KEY``,`
    -> `ADD` `PRIMARY` `KEY` `(``language_id``,` `name``)``;`
```

```
Query OK, 0 rows affected (0.09 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

一旦创建了索引，就无法修改它。但有时您可能会需要；例如，您可能希望减少从列索引的字符数或向索引中添加另一个列。执行此操作的最佳方法是删除索引，然后使用新的规范重新创建索引。例如，假设您决定只将`idx_name`索引包含艺术家名`artist_name`的前 10 个字符。只需执行以下操作：

```
mysql> `ALTER` `TABLE` `language` `DROP` `INDEX` `idx_name``,`
    -> `ADD` `INDEX` `idx_name` `(``name``(``10``)``)``;`
```

```
Query OK, 0 rows affected (0.05 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

## 重命名表格和修改其他结构

我们已经看到如何修改表中的列和索引；现在让我们看看如何修改表本身。重命名表很容易。假设您想将`language`重命名为`languages`，请使用以下命令：

```
mysql> `ALTER` `TABLE` `language` `RENAME` `TO` `languages``;`
```

```
Query OK, 0 rows affected (0.04 sec)
```

`TO`关键字是可选的。

使用`ALTER`语句还可以执行其他几项操作，包括：

+   更改数据库、表格或列的默认字符集和排序规则。

+   管理和更改约束。例如，您可以添加和删除外键。

+   向表添加分区，或修改当前分区定义。

+   更改表的存储引擎。

你可以在 MySQL 参考手册的[` ALTER DATABASE`](https://oreil.ly/2FpoZ)和[` ALTER TABLE`](https://oreil.ly/68PA3)章节找到更多关于这些操作的信息。相同语句的另一种更简短的表示法是` RENAME TABLE`：

```
mysql> `RENAME` `TABLE` `languages` `TO` `language``;`
```

```
Query OK, 0 rows affected (0.04 sec)
```

有一件事是不可能改变特定数据库的名称。然而，如果你使用 InnoDB 引擎，你可以使用` RENAME`来在不同数据库之间移动表：

```
mysql> `CREATE` `DATABASE` `sakila_new``;`
```

```
Query OK, 1 row affected (0.05 sec)
```

```
mysql> `RENAME` `TABLE` `sakila``.``language` `TO` `sakila_new``.``language``;`
```

```
Query OK, 0 rows affected (0.05 sec)
```

```
mysql> `USE` `sakila``;`
```

```
Database changed
```

```
mysql> `SHOW` `TABLES` `LIKE` `'lang%'``;`
```

```
Empty set (0.00 sec)
```

```
mysql> `USE` `sakila_new``;`
```

```
Database changed
```

```
mysql> `SHOW` `TABLES` `LIKE` `'lang%'``;`
```

```
+------------------------------+
| Tables_in_sakila_new (lang%) |
+------------------------------+
| language                     |
+------------------------------+
1 row in set (0.00 sec)
```

# 删除结构

在前一节中，我们展示了如何从数据库中删除列和行；现在我们将描述如何删除数据库和表。

## 删除数据库

移除或*删除*一个数据库是直接的。这是你如何删除` sakila`数据库：

```
mysql> `DROP` `DATABASE` `sakila``;`
```

```
Query OK, 25 rows affected (0.16 sec)
```

响应中返回的行数是删除的表的数量。删除数据库时应当小心，因为它的所有表、索引和列都会被删除，MySQL 用于维护它们的所有关联磁盘文件和目录也会被删除。

如果一个数据库不存在，尝试删除它会导致 MySQL 报告一个错误。让我们再次尝试删除` sakila`数据库：

```
mysql> `DROP` `DATABASE` `sakila``;`
```

```
ERROR 1008 (HY000): Can't drop database 'sakila'; database doesn't exist
```

你可以通过使用` IF EXISTS`短语来避免错误，这在将语句包含在脚本中时非常有用：

```
mysql> `DROP` `DATABASE` `IF` `EXISTS` `sakila``;`
```

```
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

你可以看到一个警告被报告，因为` sakila`数据库已经被删除。

## 删除表

删除表和删除数据库一样简单。让我们从` sakila`数据库创建并删除一个表：

```
mysql> `CREATE` `TABLE` `temp` `(``id` `SERIAL` `PRIMARY` `KEY``)``;`
```

```
Query OK, 0 rows affected (0.05 sec)
```

```
mysql> `DROP` `TABLE` `temp``;`
```

```
Query OK, 0 rows affected (0.03 sec)
```

不要担心：` 0 rows affected`消息是误导性的。你会发现表确实已经消失了。

你可以使用` IF EXISTS`短语来防止错误。让我们再次尝试删除` temp`表：

```
mysql> `DROP` `TABLE` `IF` `EXISTS` `temp``;`
```

```
Query OK, 0 rows affected, 1 warning (0.01 sec)
```

和往常一样，你可以用` SHOW WARNINGS`语句来调查警告：

```
mysql> `SHOW` `WARNINGS``;`
```

```
+-------+------+-----------------------------+
| Level | Code | Message                     |
+-------+------+-----------------------------+
| Note  | 1051 | Unknown table 'sakila.temp' |
+-------+------+-----------------------------+
1 row in set (0.00 sec)
```

你可以在单个语句中通过用逗号分隔表名来删除多个表：

```
mysql> `DROP` `TABLE` `IF` `EXISTS` `temp``,` `temp1``,` `temp2``;`
```

```
Query OK, 0 rows affected, 3 warnings (0.00 sec)
```

在这种情况下有三个警告，因为这些表都不存在。
