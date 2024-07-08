# 第十五章：生成和使用序列

# 15.0 引言

序列是按需生成的整数集合（1、2、3、...）。由于许多应用程序要求表中的每行包含唯一值，因此序列在数据库中经常使用，它们为生成这些值提供了一种简便方法。本章描述了如何在 MySQL 中以以下五种方式使用序列：

使用`AUTO_INCREMENT`列

`AUTO_INCREMENT`列是 MySQL 生成一系列行的机制。每次在包含`AUTO_INCREMENT`列的表中创建行时，MySQL 自动将序列中的下一个值生成为该列的值。该值作为唯一标识符，使序列成为创建项目（如客户 ID 号码、发货包裹运单号、发票或采购订单号、错误报告 ID、票号或产品序列号等）的简便方法。

检索序列值

对于许多应用程序，仅创建序列值是不够的。还必须确定刚插入行的序列值。Web 应用程序可能需要重新显示用户刚刚提交的表单内容创建的行的内容。可能需要检索该值，以便将其存储在相关表的行中。

重新排序技术

由于行删除而导致序列中存在空缺，可以重新编号序列，将删除的值重新使用到序列顶部，或者向表中没有序列列的情况下添加序列列。

管理多个同时进行的序列

在需要跟踪多个序列值（例如在创建具有`AUTO_INCREMENT`列的多个表行时）时，需要特别小心。

使用单行序列生成器

序列可用作计数器。例如，在投票中计算投票时，每当候选人获得一票，您可能会递增计数器。给定候选人的计数形成一个序列，但因为计数本身是唯一感兴趣的值，所以无需生成新行以记录每次投票。MySQL 提供了一种解决此问题的机制，使用此机制可以在单个表行内轻松生成序列。要在表中存储多个计数器，请使用标识每个计数器的列。相同的机制还可以使序列以非一或非均匀值递增。

大多数数据库系统的引擎提供序列生成功能，尽管实现方式可能依赖于引擎。MySQL 也是如此，因此即使在 SQL 层面上，本节中的材料几乎完全是特定于 MySQL 的。换句话说，生成序列的 SQL 本身是不可移植的，即使您使用像 DBI 或 JDBC 这样的 API 提供了一个抽象层。抽象接口可能帮助您可移植地处理 SQL 语句，但它们并不能使不可移植的 SQL 变得可移植。

本章示例相关的脚本位于`recipes`发行版的*sequences*目录中。对于在此处使用的创建表的脚本，请查看*tables*目录。

# 15.1 使用自增列生成序列

## 问题

你的表包含一个只能包含唯一 ID 的列，你需要插入值到这列中，确保它们是序列的一部分。

## 解决方案

使用`AUTO_INCREMENT`列生成一个序列。

## 讨论

这个配方提供了使用`AUTO_INCREMENT`列的基本背景，从展示序列生成机制的示例开始。这个示例围绕一个收集昆虫的场景展开：你的八岁儿子朱尼尔被分配了在学校的一个班级项目中收集昆虫的任务。对于每只昆虫，朱尼尔都要记录其名称（如“蚂蚁”，“蜜蜂”等），以及其收集的日期和位置。从朱尼尔还是个孩子的时候起，你就向他解释了 MySQL 在记录方面的优势，因此当你那天下班回家时，他立即宣布完成这个项目的必要性，然后直视你的眼睛，断言这显然是 MySQL 非常适合的任务。你又有什么好争论的呢？于是你们俩开始工作。小朱尼尔放学后在等你回家的时候已经收集了一些标本，并在他的笔记本中记录了以下信息：

| 名称 | 日期 | 来源 |
| --- | --- | --- |
| 马陆 | 2014-09-10 | 车道 |
| 家蝇 | 2014-09-10 | 厨房 |
| 蚱蜢 | 2014-09-10 | 前院 |
| 臭虫 | 2014-09-10 | 前院 |
| 甘蓝蝴蝶 | 2014-09-10 | 菜园 |
| 蚂蚁 | 2014-09-10 | 后院 |
| 蚂蚁 | 2014-09-10 | 后院 |
| 白蚁 | 2014-09-10 | 厨房木制品 |

看着小朱尼尔的笔记，你高兴地发现，即使在他年幼的时候，他也已经学会了用 ISO 格式写日期。然而，你也注意到他收集了一只马陆和一只白蚁，但实际上它们都不是昆虫。你决定暂且放过他；小朱尼尔忘记带回家项目的书面说明，所以目前尚不清楚这些标本是否合适。 （你还有点惊讶地注意到小朱尼尔在家里发现了白蚁，并在心里做了个记号，准备找灭虫工处理。）

当你考虑如何创建一个表来存储这些信息时，显然你需要至少`名称`，`日期`和`来源`列，这些信息对应小朱尼尔需要记录的类型：

```
CREATE TABLE insect
(
  name    VARCHAR(30) NOT NULL,   # type of insect
  date    DATE NOT NULL,          # date collected
  origin  VARCHAR(30) NOT NULL    # where collected
);
```

然而，这些列还不足以使表易于使用。请注意，迄今收集的记录并不唯一；两只蚂蚁是同时和地点采集的。如果将信息放入具有刚显示结构的`insect`表中，则无法单独引用任何蚂蚁行，因为没有什么可以区分一个蚂蚁和另一个蚂蚁。为了使行不同且提供使每行易于引用的值，唯一的 ID 将非常有帮助。`AUTO_INCREMENT`列非常适合此目的，因此更好的`insect`表的结构如下所示：

```
CREATE TABLE insect
(
  id      INT UNSIGNED NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (id),
  name    VARCHAR(30) NOT NULL,   # type of insect
  date    DATE NOT NULL,          # date collected
  origin  VARCHAR(30) NOT NULL    # where collected
);
```

继续使用这第二个`CREATE` `TABLE`语句创建`insect`表。(Recipe 15.2 讨论了`id`列定义的详细信息。)

现在，您有了一个`AUTO_INCREMENT`列，可以用它来生成新的序列值。`AUTO_INCREMENT`列的一个有用属性是，您无需自己分配其值：MySQL 会为您做这件事。有两种方法可以在`id`列中生成新的`AUTO_INCREMENT`值。一种方法是将`id`列明确设置为`NULL`。以下语句以这种方式将 Junior 的前四个标本插入`insect`表中：

```
mysql> `INSERT INTO insect (id,name,date,origin) VALUES`
    -> `(NULL,'housefly','2014-09-10','kitchen'),`
    -> `(NULL,'millipede','2014-09-10','driveway'),`
    -> `(NULL,'grasshopper','2014-09-10','front yard'),`
    -> `(NULL,'stink bug','2014-09-10','front yard');`
```

或者，完全省略`INSERT`语句中的`id`列。MySQL 允许创建行，而无需为具有默认值的列明确指定值。MySQL 为每个缺失的列分配其默认值，而`AUTO_INCREMENT`列的默认值是其下一个序列号。因此，此语句将 Junior 的其他四个标本添加到`insect`表中，并在根本不命名`id`列的情况下生成序列值：

```
mysql> `INSERT INTO insect (name,date,origin) VALUES`
    -> `('cabbage butterfly','2014-09-10','garden'),`
    -> `('ant','2014-09-10','back yard'),`
    -> `('ant','2014-09-10','back yard'),`
    -> `('termite','2014-09-10','kitchen woodwork');`
```

无论您使用哪种方法，MySQL 都会确定每行的序列号并将其分配给`id`列，您可以进行验证：

```
mysql> `SELECT * FROM insect ORDER BY id;`
+----+-------------------+------------+------------------+
| id | name              | date       | origin           |
+----+-------------------+------------+------------------+
|  1 | housefly          | 2014-09-10 | kitchen          |
|  2 | millipede         | 2014-09-10 | driveway         |
|  3 | grasshopper       | 2014-09-10 | front yard       |
|  4 | stink bug         | 2014-09-10 | front yard       |
|  5 | cabbage butterfly | 2014-09-10 | garden           |
|  6 | ant               | 2014-09-10 | back yard        |
|  7 | ant               | 2014-09-10 | back yard        |
|  8 | termite           | 2014-09-10 | kitchen woodwork |
+----+-------------------+------------+------------------+
```

随着 Junior 收集更多标本，向表中添加更多行，它们将被分配为序列中的下一个值（9、10、…）。

`AUTO_INCREMENT`列背后的概念原则上非常简单：每次创建新行时，MySQL 生成序列中的下一个数字并将其分配给行。但是，有一些关于这些列需要了解的微妙之处，以及不同存储引擎处理`AUTO_INCREMENT`序列的差异。了解这些问题可以更有效地使用序列并避免意外。例如，如果将`id`列明确设置为非`NULL`值，则可能会发生以下两种情况之一：

+   如果该值已存在于表中，且该列不能包含重复项，则会出现错误。对于`insect`表，`id`列是`PRIMARY` `KEY`，禁止重复：

    ```
    mysql> `INSERT INTO insect (id,name,date,origin) VALUES`
        -> `(3,'cricket','2014-09-11','basement');`
    ERROR 1062 (23000): Duplicate entry '3' for key 'PRIMARY'
    ```

+   如果表中不存在该值，MySQL 将使用该值插入行。此外，如果该值大于当前序列计数器，表的计数器将重置为该值加一。在此时的`insect`表中，序列值为 1 到 8。如果插入`id`列设为 20 的新行，则该值将成为新的最大值。随后自动生成`id`值的插入将从 21 开始。值 9 到 19 变为未使用，导致序列中出现间隔。

下一个示例将更详细地讨论如何定义`AUTO_INCREMENT`列及其行为。

# 15.2 选择序列列的数据类型

## 问题

您希望选择正确的数据类型来定义序列列。

## 解决方案

考虑序列应包含多少个唯一值，并相应选择数据类型。

## 讨论

在创建`AUTO_INCREMENT`列时，您应遵循特定的原则。例如，考虑一下 Recipe 15.1 如何声明`insect`表中的`id`列：

```
id INT UNSIGNED NOT NULL AUTO_INCREMENT,
PRIMARY KEY (id)
```

`AUTO_INCREMENT`关键字告知 MySQL 应为列的值生成连续的序列号，但其他信息同样重要：

+   `INT`是列的基本数据类型。您不一定要使用`INT`，但列应为整数类型之一：`TINYINT`、`SMALLINT`、`MEDIUMINT`、`INT`或`BIGINT`。

+   `UNSIGNED`禁止负列值。这不是`AUTO_INCREMENT`列的必需属性，但序列仅包含正整数（通常从 1 开始），因此无需允许负值。此外，*不*声明列为`UNSIGNED`会将序列范围减半。例如，`TINYINT`的范围是-128 到 127。因为序列仅包含正值，`TINYINT`序列的有效范围是 1 到 127。`TINYINT UNSIGNED`的范围是 0 到 255，这将序列的上限增加到 255。具体的整数类型决定了最大序列值。以下表显示了每种类型的最大无符号值；使用这些信息选择足够大以容纳您所需的最大值的类型：

    | 数据类型 | 最大无符号值 |
    | --- | --- |
    | `TINYINT` | 255 |
    | `SMALLINT` | 65,535 |
    | `MEDIUMINT` | 16,777,215 |
    | `INT` | 4,294,967,295 |
    | `BIGINT` | 18,446,744,073,709,551,615 |

    有时人们会省略`UNSIGNED`，以便在序列列中创建包含负数的行（例如使用-1 表示<q>没有 ID</q>）。这是一个坏主意。MySQL 不保证如何处理`AUTO_INCREMENT`列中的负数，因此使用它们相当于玩火。例如，如果重新排序列，所有负值都会变成正序列号。

+   `AUTO_INCREMENT`列不能包含`NULL`值，因此`id`被声明为`NOT` `NULL`。（确实，当您插入新行时，可以指定`NULL`作为列值，但对于`AUTO_INCREMENT`列，这实际上意味着<q>生成下一个序列值。</q>）如果您忘记，MySQL 会自动将`AUTO_INCREMENT`列定义为`NOT` `NULL`。

+   `AUTO_INCREMENT`列必须被索引。通常情况下，由于存在顺序列来提供唯一标识符，您可以使用`PRIMARY` `KEY`或`UNIQUE`索引来确保唯一性。表只能有一个`PRIMARY` `KEY`，因此，如果表已经有其他`PRIMARY` `KEY`列，您可以声明`AUTO_INCREMENT`列为`UNIQUE`索引：

    ```
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    UNIQUE (id)
    ```

当您创建包含`AUTO_INCREMENT`列的表时，还重要考虑使用哪种存储引擎（如 InnoDB、MyISAM 等）。引擎会影响诸如重用从顺序顶部删除的值等行为（见 Recipe 15.3）。

# 15.3 删除行而不更改序列

## 问题

您希望从包含`AUTO_INCREMENT`列的表中删除少数行。

## 解决方案

使用常规`DELETE`语句。MySQL 不会更改现有行的生成序列号。

## 讨论

到目前为止，我们考虑了 MySQL 在仅向表中添加行的情况下如何生成`AUTO_INCREMENT`列的序列值。但是假设永远不会删除行是不现实的。那么序列会发生什么变化？

再次参考 Junior 的昆虫收集项目，目前您拥有一个看起来像这样的`insect`表：

```
mysql> `SELECT * FROM insect ORDER BY id;`
+----+-------------------+------------+------------------+
| id | name              | date       | origin           |
+----+-------------------+------------+------------------+
|  1 | housefly          | 2014-09-10 | kitchen          |
|  2 | millipede         | 2014-09-10 | driveway         |
|  3 | grasshopper       | 2014-09-10 | front yard       |
|  4 | stink bug         | 2014-09-10 | front yard       |
|  5 | cabbage butterfly | 2014-09-10 | garden           |
|  6 | ant               | 2014-09-10 | back yard        |
|  7 | ant               | 2014-09-10 | back yard        |
|  8 | termite           | 2014-09-10 | kitchen woodwork |
+----+-------------------+------------+------------------+
```

这将要发生变化，因为 Junior 记得带回项目的书面说明后，您阅读并发现两件影响表内容的事情：

+   标本应包括只有昆虫，而不是类似昆虫的生物，如千足虫和白蚁。

+   项目的目的是尽可能收集*不同*的标本，而不仅仅是尽可能多的标本。这意味着只允许一行蚂蚁。

这些指令要求从表中删除几行——具体来说，是`id`值为 2（千足虫）、8（白蚁）和 7（重复的蚂蚁）。因此，尽管 Junior 显然对他的收藏减少感到失望，你指示他通过发出`DELETE`语句来删除这些行：

```
mysql> `DELETE FROM insect WHERE id IN (2,8,7);`
```

这个声明说明为什么具有唯一 ID 值很有用：它们使您能够明确指定任何一行。蚂蚁行除了`id`值外完全相同。如果表中没有这一列，将更难删除其中的一行（虽然不是不可能；参见 Recipe 18.5）。

在移除不适当的行后，表中剩余的行如下：

```
mysql> `SELECT * FROM insect ORDER BY id;`
+----+-------------------+------------+------------+
| id | name              | date       | origin     |
+----+-------------------+------------+------------+
|  1 | housefly          | 2014-09-10 | kitchen    |
|  3 | grasshopper       | 2014-09-10 | front yard |
|  4 | stink bug         | 2014-09-10 | front yard |
|  5 | cabbage butterfly | 2014-09-10 | garden     |
|  6 | ant               | 2014-09-10 | back yard  |
+----+-------------------+------------+------------+
```

`id` 列现在有一个空隙（第 2 行缺失），顶部的值 7 和 8 也不再存在。这些删除操作会如何影响未来的插入操作？下一个新行会得到哪个序列号？

删除第 2 行会在序列中间创建一个空隙。这不会对后续的插入操作产生影响，因为 MySQL 不会尝试填补序列中的空隙。另一方面，删除第 7 和第 8 行会移除序列顶部的值。对于 InnoDB 或 MyISAM 表，这些值不会被重用。下一个序列号是未曾使用过的最小正整数。（例如，序列当前为 8，即使先删除第 7 和第 8 行，下一个行得到的值仍然是 9。）如果需要严格单调递增的序列，可以使用这些存储引擎。对于其他存储引擎，移除序列顶部的值可能会或者可能不会被重用。在使用之前，请检查引擎的属性。

如果表使用的引擎与您需要的值重用行为不同，请使用`ALTER TABLE`将表更改为更合适的引擎。例如，要更改表以使用 InnoDB（以防止在删除行后重新使用序列值），请执行以下操作：

```
ALTER TABLE *`tbl_name`* ENGINE = InnoDB;
```

如果不知道表使用的是哪种引擎，请查询`INFORMATION_SCHEMA`或使用`SHOW TABLE STATUS`或`SHOW CREATE TABLE`进行查询。例如，以下语句指示`insect`是一个 InnoDB 表：

```
mysql> `SELECT ENGINE FROM INFORMATION_SCHEMA.TABLES`
    -> `WHERE TABLE_SCHEMA = 'cookbook' AND TABLE_NAME = 'insect';`
+--------+
| ENGINE |
+--------+
| InnoDB |
+--------+
```

要清空表并重置序列计数器（即使对于通常不重新使用值的引擎也是如此），请使用`TRUNCATE TABLE`：

```
TRUNCATE TABLE *`tbl_name`*;
```

# 15.4 检索序列值

## 问题

创建包含新序列号的行后，您希望知道该数字是多少。

## 解决方案

调用`LAST_INSERT_ID()`函数。如果您正在编写程序，MySQL API 可能提供一种直接获取值的方法，而无需发出 SQL 语句。

## 讨论

应用程序通常需要知道新创建行的`AUTO_INCREMENT`值。例如，如果您为 Junior 的`insect`表编写一个基于 Web 的前端用于输入行，可能会在按下提交按钮后立即在新页面上格式化显示每个新行。为了实现这一点，您必须知道新的`id`值，以便可以检索到正确的行。还有一种情况需要`AUTO_INCREMENT`值，即在使用多个表时：在主表中插入行后，需要其 ID 来创建与主表中行相关的其他相关表中的行。（Recipe 15.11 展示了如何实现这一点。）

当生成新的`AUTO_INCREMENT`值时，从服务器获取值的一种方法是执行调用`LAST_INSERT_ID()`函数的语句。此外，许多 MySQL API 提供了客户端机制，以使值在不发出其他语句的情况下可用。本篇讨论了这两种方法并比较了它们的特性。

### 使用 LAST_INSERT_ID() 获取 AUTO_INCREMENT 值

确定新行的 `AUTO_INCREMENT` 值的显而易见（但不正确）方法利用了 MySQL 生成值时成为列中最大序列号的事实。因此，您可能会尝试使用 `MAX()` 函数来检索它：

```
SELECT MAX(id) FROM insect;
```

这是不可靠的；如果另一个客户端在您发出 `SELECT` 语句之前插入一行，则 `MAX(id)` 返回该客户端的 ID，而不是您的。可以通过将 `INSERT` 和 `SELECT` 语句分组为事务或锁定表来解决此问题，但 MySQL 提供了一种更简单的方法来获取正确的值：调用 `LAST_INSERT_ID()` 函数。它返回在您的会话中生成的最近的 `AUTO_INCREMENT` 值，而不管其他客户端在做什么。例如，要将一行插入 `insect` 表并检索其 `id` 值，执行以下操作：

```
mysql> `INSERT INTO insect (name,date,origin)`
    -> `VALUES('cricket','2014-09-11','basement');`
mysql> `SELECT LAST_INSERT_ID();`
+------------------+
| LAST_INSERT_ID() |
+------------------+
|                9 |
+------------------+
```

或者您可以使用新值检索整行，甚至不知道它是什么：

```
mysql> `INSERT INTO insect (name,date,origin)`
    -> `VALUES('moth','2014-09-14','windowsill');`
mysql> `SELECT * FROM insect WHERE id = LAST_INSERT_ID();`
+----+------+------------+------------+
| id | name | date       | origin     |
+----+------+------------+------------+
| 10 | moth | 2014-09-14 | windowsill |
+----+------+------------+------------+
```

服务器基于会话特定方式维护由 `LAST_INSERT_ID()` 返回的值。这一属性是有意设计的，因为它防止客户端相互干扰。当您生成 `AUTO_INCREMENT` 值时，`LAST_INSERT_ID()` 返回该特定值，即使其他客户端在此期间向同一表中生成新行。

### 使用特定 API 方法获取 AUTO_INCREMENT 值

`LAST_INSERT_ID()` 是一个 SQL 函数，因此您可以在能够执行 SQL 语句的任何客户端中使用它。另一方面，您必须执行单独的语句以获取其值。当您编写自己的程序时，可能会有另一种选择。许多 MySQL 接口包含一个 API 特定扩展，该扩展可以返回 `AUTO_INCREMENT` 值，而无需执行额外的语句。我们大多数的 API 都具备此功能。

Perl

使用 `mysql_insertid` 属性获取语句生成的 `AUTO_INCREMENT` 值。此属性通过数据库句柄或语句句柄访问，具体取决于您如何发出语句。以下示例通过数据库句柄引用它：

```
$dbh->do ("INSERT INTO insect (name,date,origin)
 VALUES('moth','2014-09-14','windowsill')");
my $seq = $dbh->{mysql_insertid};
```

要将 `mysql_insertid` 作为语句句柄属性访问，请使用 `prepare()` 和 `execute()`：

```
my $sth = $dbh->prepare ("INSERT INTO insect (name,date,origin)
 VALUES('moth','2014-09-14','windowsill')");
$sth->execute ();
my $seq = $sth->{mysql_insertid};
```

Ruby

Ruby Mysql2 gem 使用 `last_id` 方法公开客户端端 `AUTO_INCREMENT` 值：

```
client.query("INSERT INTO insect (name,date,origin)
 VALUES('moth','2014-09-14','windowsill')")
seq = client.last_id
```

PHP

PDO 接口为 MySQL 提供了 `lastInsertId()` 数据库句柄方法，返回最近的 `AUTO_INCREMENT` 值：

```
$dbh->exec ("INSERT INTO insect (name,date,origin)
 VALUES('moth','2014-09-14','windowsill')");
$seq = $dbh->lastInsertId ();
```

Python

DB API 提供的 Connector/Python 驱动程序提供了 `lastrowid` 游标对象属性，返回最近的 `AUTO_INCREMENT` 值：

```
cursor = conn.cursor()
cursor.execute('''
 INSERT INTO insect (name,date,origin)
 VALUES('moth','2014-09-14','windowsill')
 ''')
seq = cursor.lastrowid
```

Java

Connector/J JDBC 驱动程序 `getGeneratedKeys()` 方法返回 `AUTO_INCREMENT` 值。如果在语句执行过程中提供额外的 `Statement.RETURN_GENERATED_KEYS` 参数以指示您要检索序列值，则它可以与 `Statement` 或 `PreparedStatement` 对象一起使用。

对于 `Statement`：

```
Statement s = conn.createStatement ();
s.executeUpdate ("INSERT INTO insect (name,date,origin)"
                 + " VALUES('moth','2014-09-14','windowsill')",
                 Statement.RETURN_GENERATED_KEYS);
```

对于 `PreparedStatement`：

```
PreparedStatement s = conn.prepareStatement (
                    "INSERT INTO insect (name,date,origin)"
                    + " VALUES('moth','2014-09-14','windowsill')",
                    Statement.RETURN_GENERATED_KEYS);
s.executeUpdate ();
```

然后，从`getGeneratedKeys()`生成一个新的结果集以访问序列值：

```
long seq;
ResultSet rs = s.getGeneratedKeys ();
if (rs.next ())
{
  seq = rs.getLong (1);
}
else
{
  throw new SQLException ("getGeneratedKeys() produced no value");
}
rs.close ();
s.close ();
```

Go

Go MySQL 驱动程序提供了`Result`接口的`LastInsertId`方法，返回最新的`AUTO_INCREMENT`值。

```
res, err := db.Exec(`INSERT INTO insect (name,date,origin)
 VALUES ('moth','2014-09-14','windowsill')`)
seq, err := res.LastInsertId()
```

### 与服务器端和客户端端比较序列值检索

如前所述，服务器在会话特定基础上维护`LAST_INSERT_ID()`的值。相比之下，用于直接访问`AUTO_INCREMENT`值的 API 特定方法在客户端上实现。服务器端和客户端端序列值检索方法有一些相似之处，但也有一些不同之处。

所有方法，无论是服务器端还是客户端，都要求你在生成它的同一个 MySQL 会话中访问`AUTO_INCREMENT`值。如果生成了`AUTO_INCREMENT`值，然后在尝试访问该值之前断开与服务器的连接并重新连接，你将得到零值。在给定的会话中，服务器端的`AUTO_INCREMENT`值的持久性可能会更长：

+   在执行生成`AUTO_INCREMENT`值的语句后，只要没有其他语句生成`AUTO_INCREMENT`值，该值仍然可通过`LAST_INSERT_ID()`访问。

+   使用客户端 API 方法可获得的序列值通常适用于*每个*语句，而不仅仅是生成`AUTO_INCREMENT`值的语句。如果你执行生成新值的`INSERT`语句，然后在访问客户端序列值之前执行其他语句，那么它可能已经被设置为零。具体行为在不同的 API 之间有所不同，但为了安全起见，你可以这样做：当语句生成一个你暂时不会使用的序列值时，将该值保存在一个变量中，以便稍后引用。否则，当你尝试访问时，可能会发现序列值已被清除。（更多信息，请参见食谱 15.10。）

# 15.5 重新编号现有序列

## 问题

你的序列列中有间隙，你想重新排序它。

## 解决方案

首先，考虑是否需要重新排序。在许多情况下，这是不必要的。但如果必须这样做，请定期重新排序`AUTO_INCREMENT`列。

## 讨论

如果你向具有`AUTO_INCREMENT`列的表插入行，并且从不删除任何行，则列中的值形成一个连续的序列。如果删除行，则序列开始出现间隙。例如，Junior 的`insect`表目前看起来像是这样，序列中有间隙（假设你已经插入了食谱 15.4 中显示的蟋蟀和蛾行）。

```
mysql> `SELECT * FROM insect ORDER BY id;`
+----+-------------------+------------+------------+
| id | name              | date       | origin     |
+----+-------------------+------------+------------+
|  1 | housefly          | 2014-09-10 | kitchen    |
|  3 | grasshopper       | 2014-09-10 | front yard |
|  4 | stink bug         | 2014-09-10 | front yard |
|  5 | cabbage butterfly | 2014-09-10 | garden     |
|  6 | ant               | 2014-09-10 | back yard  |
|  9 | cricket           | 2014-09-11 | basement   |
| 10 | moth              | 2014-09-14 | windowsill |
+----+-------------------+------------+------------+
```

MySQL 不会尝试通过填充未使用的值来消除这些间隙，当插入新行时。不喜欢这种行为的人倾向于定期重新排序`AUTO_INCREMENT`列以消除这些间隙。本文档中的示例展示了如何做到这一点。还可以扩展现有序列的范围（参见配方 15.6），强制顶部序列中删除的值被重复使用（参见配方 15.7），按特定顺序编号行（参见配方 15.8），或者向当前没有序列列的表添加序列列（参见配方 15.9）。

在决定重新排序`AUTO_INCREMENT`列之前，请考虑是否真的有必要。通常情况下并不需要，而且在某些情况下可能会导致实际问题。例如，不应该重新排序包含其他表引用值的列。重新编号这些值会破坏它们与其他表中值的对应关系，从而无法正确地将两个表中的行关联起来。

以下是一些推进重新排序列的高级原因：

美学

有些人更喜欢不间断的序列，而不是有间隙的序列。如果这是你想要重新排序的原因，我们可能无法说服你改变想法。尽管如此，这并不是一个特别好的理由。

性能

重新排序的动机可能源于这样的观念，即这样做可以通过移除间隙来“压缩”一个序列列，并使 MySQL 运行语句更快。这是不正确的。MySQL 并不关心是否有空白，而且通过重新编号`AUTO_INCREMENT`列并不会带来性能上的提升。事实上，重新排序会在某种意义上对性能产生负面影响，因为在 MySQL 执行此操作时表会保持锁定状态——对于大表来说可能需要相当长的时间。其他客户端在此过程中可以读取表中的数据，但试图插入新行的客户端会被阻塞，直到操作完成。

编号耗尽

序列列的数据类型和有符号性确定了其上限（参见配方 15.2）。如果一个`AUTO_INCREMENT`序列接近其数据类型的上限，重新编号会压缩序列并释放更多顶部的值。这可能是重新排序列的正当理由，但在许多情况下仍然是不必要的。您可以尝试更改列数据类型以增加其上限，而不改变列中存储的值；参见配方 15.6。

如果你仍然决定重新排序某列，这很容易做到：从表中删除该列，然后再放回。MySQL 会按照不间断的顺序重新编号列中的值。以下示例展示了如何使用这种技术来重新编号`insect`表中的`id`值：

```
mysql> `ALTER TABLE insect DROP id;`
mysql> `ALTER TABLE insect`
    -> `ADD id INT UNSIGNED NOT NULL AUTO_INCREMENT FIRST,`
    -> `ADD PRIMARY KEY (id);`
```

第一个`ALTER TABLE`语句删除了`id`列（因此也删除了`PRIMARY KEY`，因为它所引用的列不再存在）。第二个语句将列恢复到表中，并将其设为`PRIMARY KEY`。（`FIRST`关键字将列放在表中的第一位，这就是它最初的位置。通常情况下，`ADD`会将列放在表的末尾。）

当您向表中添加一个`AUTO_INCREMENT`列时，MySQL 会自动对所有行进行连续编号，因此`insect`表的内容如下所示：

```
mysql> `SELECT * FROM insect ORDER BY id;`
+----+-------------------+------------+------------+
| id | name              | date       | origin     |
+----+-------------------+------------+------------+
|  1 | housefly          | 2014-09-10 | kitchen    |
|  2 | grasshopper       | 2014-09-10 | front yard |
|  3 | stink bug         | 2014-09-10 | front yard |
|  4 | cabbage butterfly | 2014-09-10 | garden     |
|  5 | ant               | 2014-09-10 | back yard  |
|  6 | cricket           | 2014-09-11 | basement   |
|  7 | moth              | 2014-09-14 | windowsill |
+----+-------------------+------------+------------+
```

使用单独的`ALTER TABLE`语句对列进行重新排序的一个问题是，在两次操作之间，表中将没有该列。这可能会导致其他客户端在此期间尝试访问表时遇到困难。为了防止这种情况发生，请使用单个`ALTER TABLE`语句执行这两个操作：

```
mysql> `ALTER TABLE insect`
    -> `DROP id,`
    -> `ADD id INT UNSIGNED NOT NULL AUTO_INCREMENT FIRST;`
```

MySQL 允许使用`ALTER TABLE`执行多个操作（并非所有数据库系统都支持）。但请注意，这种多操作语句并不简单地将两个单操作`ALTER TABLE`语句连接起来。不同之处在于，不需要重新建立`PRIMARY KEY`：除非在执行`ALTER TABLE`语句中指定的所有操作后，索引列已经丢失，否则 MySQL 不会删除它。

# 15.6 扩展序列列的范围

## 问题

您希望避免重新对列进行排序，但是您的新序列号已经用完了。

## 解决方案

检查您是否可以将列设置为`UNSIGNED`或更改为使用更大的整数类型。

## 讨论

通过扩展列的范围而不是其内容，可以避免重新排序`AUTO_INCREMENT`列的影响，这会改变表的结构而不是其内容：

+   如果数据类型为有符号的，请将其设置为`UNSIGNED`，以加倍可用值的范围。假设当前一个`id`列被定义如下：

    ```
    id MEDIUMINT NOT NULL AUTO_INCREMENT
    ```

    有符号`MEDIUMINT`列的上限范围为 8,388,607。要将其增加到 16,777,215，请使用`ALTER TABLE`使该列为`UNSIGNED`：

    ```
    ALTER TABLE *`tbl_name`* MODIFY id MEDIUMINT UNSIGNED NOT NULL AUTO_INCREMENT;
    ```

+   如果您的列已经是`UNSIGNED`，并且还不是最大的整数类型（`BIGINT`），将其转换为更大的类型会增加其范围。对于前面例子中的`id`列，也要使用`ALTER TABLE`进行转换，从`MEDIUMINT`转换为`BIGINT`，如下所示：

    ```
    ALTER TABLE *`tbl_name`* MODIFY id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT;
    ```

15.2 章节的配方展示了每种整数数据类型的范围，这可以帮助您选择合适的类型。

# 15.7 重新使用序列顶部的值

## 问题

您已删除了序列顶端的行，并且希望避免重新对列进行排序，但仍然要重用这些值。

## 解决方案

是的。使用`ALTER TABLE`重置序列计数器。新的序列号将从表中当前最大值加一开始。

## 讨论

如果您只从顺序的顶部删除了行，则剩余的行仍然按顺序排列而无间隙。（例如，如果您有从 1 到 100 编号的行，并删除了编号为 91 到 100 的行，则剩余的行仍然从 1 到 90 连续编号。）在这种特殊情况下，重新编号列是不必要的。而是告诉 MySQL 通过执行此语句恢复从最高现有序列号大一的值开始的序列计数器向下重置新行：

```
ALTER TABLE *`tbl_name`* AUTO_INCREMENT = 1;
```

如果一个序列列中间存在间隙，则可以使用`ALTER` `TABLE`重置序列计数器，但仍然只重用从顺序的顶部删除的值。它不会消除间隙。假设表包含从 1 到 10 的序列值，您删除了值为 3、4、5、9 和 10 的行。最大剩余值为 8，因此如果您使用`ALTER` `TABLE`重置序列计数器，则下一行将被赋值为 9，而不是 3。要重新排序表以消除间隙，请参见食谱 15.5。

# 15.8 确保行按特定顺序重新编号

## 问题

您重新排序了一列，但 MySQL 没有按您想要的方式对行进行编号。

## 解决方案

使用`ORDER` `BY`子句将行选择到另一张表中，按您想要的顺序排列，并让 MySQL 根据排序顺序对它们编号，同时执行操作。

## 讨论

当您重新排序`AUTO_INCREMENT`列时，MySQL 可以自由地从表中选择行，因此它不一定会按您期望的顺序重新编号它们。如果您唯一的要求是每行有唯一的标识符，则这一点无关紧要。但是您可能有一个应用程序，其中重要的是按特定顺序为行分配序列号。例如，您可能希望序列与创建行的顺序对应，如由`TIMESTAMP`列指示。要按特定顺序分配编号，请使用以下程序：

1.  创建一个空表的克隆（参见食谱 6.1）。

1.  使用`INSERT` `INTO` … `SELECT`将行从原表复制到克隆表。除了`AUTO_INCREMENT`列外，复制所有列，使用`ORDER` `BY`子句指定复制行的顺序（从而指定 MySQL 分配`AUTO_INCREMENT`列号的顺序）。

1.  删除原始表并将克隆重命名为原始表的名称。

1.  如果表是一个大的 MyISAM 表并且具有多个索引，则最好最初创建新表时没有索引，除了`AUTO_INCREMENT`列上的索引。然后将原表复制到新表中，并使用`ALTER` `TABLE`在之后添加其余索引。

    这也适用于 InnoDB。但是，InnoDB 的[Change Buffer](https://dev.mysql.com/doc/refman/8.0/en/innodb-change-buffer.html)在内存中缓存对二级索引的更改，并在后台将其刷新到磁盘，从而保持插入性能良好。

另一种方法：

1.  创建一个新表，其中包含原始表的所有列，除了`AUTO_INCREMENT`列。

1.  使用`INSERT INTO ... SELECT`将原表的非`AUTO_INCREMENT`列复制到新表中。

1.  使用`TRUNCATE TABLE`在原表上清空它；这也将重置序列计数器为 1。

1.  使用`ORDER` `BY`子句将行从新表复制回原始表，以便按照所需顺序为行分配序列号。MySQL 为`AUTO_INCREMENT`列分配序列值。

# 15.9 对未排序表进行排序

## 问题

当您创建表时忘记包含序列列时，已经太晚了吗？可以对表行进行排序吗？

## 解决方案

不。使用`ALTER TABLE`添加`AUTO_INCREMENT`列；MySQL 将创建该列并对其行进行编号。

## 讨论

假设表包含`name`和`age`列，但没有序列列：

```
mysql> `SELECT * FROM t;`
+----------+------+
| name     | age  |
+----------+------+
| boris    |   47 |
| clarence |   62 |
| abner    |   53 |
+----------+------+
```

按以下方式向表添加名为`id`的序列列：

```
mysql> `ALTER TABLE t`
    -> `ADD id INT NOT NULL AUTO_INCREMENT,`
    -> `ADD PRIMARY KEY (id);`
mysql> `SELECT * FROM t ORDER BY id;`
+----------+------+----+
| name     | age  | id |
+----------+------+----+
| boris    |   47 |  1 |
| clarence |   62 |  2 |
| abner    |   53 |  3 |
+----------+------+----+
```

MySQL 为您编号行；您无需自行分配值。非常方便。

默认情况下，`ALTER TABLE`将新列添加到表的末尾。要将列放置在特定位置，请在`ADD`子句的末尾使用`FIRST`或`AFTER`。以下`ALTER TABLE`语句类似于刚刚显示的语句，但是将`id`列放在表中的第一位置或在`name`列之后。

```
ALTER TABLE t
  ADD id INT NOT NULL AUTO_INCREMENT FIRST,
  ADD PRIMARY KEY (id);
```

```
ALTER TABLE t
  ADD id INT NOT NULL AUTO_INCREMENT AFTER name,
  ADD PRIMARY KEY (id);
```

# 15.10 同时管理多个自增值

## 问题

您正在执行多个生成`AUTO_INCREMENT`值的语句，并且有必要独立跟踪它们。例如，您正在将行插入到具有各自`AUTO_INCREMENT`列的多个表中。

## 解决方案

将序列值保存在变量中以备后用。或者，如果您从程序内执行生成序列的语句，可能可以使用单独的连接或语句对象来发出语句，以避免混淆。

## 讨论

正如 Recipe 15.4 所述，`LAST_INSERT_ID()`服务器端序列值函数在每次语句生成`AUTO_INCREMENT`值时设置，而客户端序列指示器可能会在每个语句后重置。假设您发出生成`AUTO_INCREMENT`值的语句，但在发出第二个生成`AUTO_INCREMENT`值的语句之前不想引用该值呢？在这种情况下，原始值将不再可访问，无论是通过`LAST_INSERT_ID()`还是作为客户端值。为了保留对其的访问权，请在发出第二个语句之前先保存该值。有几种方法可以做到这一点：

+   在 SQL 级别，在生成`AUTO_INCREMENT`值的语句后，将该值保存在用户定义的变量中：

    ```
    INSERT INTO *`tbl_name`* (id,...) VALUES(NULL,...);
    SET @saved_id = LAST_INSERT_ID();
    ```

    然后，您可以发布其他语句，而不考虑它们对`LAST_INSERT_ID()`的影响。要在后续语句中使用原始的`AUTO_INCREMENT`值，请参考`@saved_id`变量。

+   在 API 级别，将`AUTO_INCREMENT`值保存在 API 语言变量中。可以通过保存从`LAST_INSERT_ID()`或任何可用的 API 特定扩展返回的值来实现。

+   一些 API 允许您维护单独的客户端`AUTO_INCREMENT`值。例如，Perl DBI 语句处理有一个`mysql_insertid`属性，一个处理的属性值不受另一个活动的影响。在 Java 中，使用单独的`Statement`或`PreparedStatement`对象。

参见 Recipe 15.11 以将这些技术应用于必须向包含`AUTO_INCREMENT`列的多个表中插入行的情况。

# 15.11 使用自增值关联表

## 问题

您可以从一张表中使用序列值作为第二张表中的键，以便将两张表中的行关联起来。但是这些关联没有正确设置。

## 解决方案

您可能没有按正确顺序插入行，或者丢失了序列值的跟踪。更改插入顺序，或保存序列值以便在需要时引用它们。

## 讨论

如果您还将`AUTO_INCREMENT`值用作主表中的 ID 值，并且还将该值存储在详细表行中以便将详细行链接到适当的主表行，则需要小心。假设`invoice`表列出了客户订单的发票信息，而`inv_item`表列出了与每个发票关联的各个项目。在这里，`invoice`是主表，`inv_item`是详细表。为了唯一标识每个订单，在`invoice`表中包括一个`AUTO_INCREMENT`列`inv_id`。还应在每个`inv_item`表行中存储适当的发票号码，以便确定其属于哪个发票。这些表可能如下所示：

```
CREATE TABLE invoice
(
  inv_id  INT UNSIGNED NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (inv_id),
  date    DATE NOT NULL
  # ... other columns could go here
  # ... (customer ID, shipping address, etc.)
);
CREATE TABLE inv_item
(
  inv_id    INT UNSIGNED NOT NULL,  # invoice ID (from invoice table)
  INDEX (inv_id),
  qty       INT,                    # quantity
  description VARCHAR(40)           # description
);
```

对于这种表关系，通常首先向主表插入一行（以生成标识该行的`AUTO_INCREMENT`值），然后使用`LAST_INSERT_ID()`插入详细行以获取主行 ID。如果客户购买了一把锤子、三盒钉子以及（为了防止用锤子时砸伤手指）一打绷带，与订单相关的行可以像这样插入两张表：

```
INSERT INTO invoice (inv_id,date)
  VALUES(NULL,CURDATE());
INSERT INTO inv_item (inv_id,qty,description)
  VALUES(LAST_INSERT_ID(),1,'hammer');
INSERT INTO inv_item (inv_id,qty,description)
  VALUES(LAST_INSERT_ID(),3,'nails, box');
INSERT INTO inv_item (inv_id,qty,description)
  VALUES(LAST_INSERT_ID(),12,'bandage');
```

第一个`INSERT`将一行添加到`invoice`主表中，并为其`inv_id`列生成新的`AUTO_INCREMENT`值。随后的`INSERT`语句每个都向`inv_item`详细表中添加一行，并使用`LAST_INSERT_ID()`获取发票号码。这样可以将详细行与适当的主行关联起来。

如果您有多个要处理的发票怎么办？有正确的方法和错误的方法来输入信息。正确的方法是首先插入第一张发票的所有信息，然后继续下一张。错误的方法是将所有主行添加到`invoice`表中，然后将所有详细行添加到`inv_item`表中。如果这样做，`inv_item`表中所有新的详细行都将具有最近输入的`invoice`行的`AUTO_INCREMENT`值。因此，所有项目看起来都属于该发票，并且两个表中的行没有正确的关联。

如果详细表包含自己的`AUTO_INCREMENT`列，则在向表中添加行时必须更加小心。假设您希望`inv_item`表中的每一行都具有唯一标识符。为此，请按照以下方式创建`inv_item`表，其中包含名为`item_id`的`AUTO_INCREMENT`列：

```
CREATE TABLE inv_item
(
  inv_id  INT UNSIGNED NOT NULL,  # invoice ID (from invoice table)
  item_id INT UNSIGNED NOT NULL AUTO_INCREMENT, # item ID
  PRIMARY KEY (item_id),
  qty     INT,                                  # quantity
  description VARCHAR(40)                       # description
);
```

`inv_id`列使每个`inv_item`行能够与正确的`invoice`表行关联，就像原始表结构一样。此外，`item_id`唯一标识每个项目行。但是，现在两个表都包含`AUTO_INCREMENT`列，您不能像以前那样输入发票信息了。如果执行之前显示的`INSERT`语句，由于`inv_item`表结构的更改，它们现在会产生不同的结果。对`invoice`表的`INSERT`正常工作。同样对`inv_item`表的第一个`INSERT`也是如此；`LAST_INSERT_ID()`返回`invoice`表中主行的`inv_id`值。然而，此`INSERT`也生成自己的`AUTO_INCREMENT`值（用于`item_id`列），这会改变`LAST_INSERT_ID()`的值，并导致主行的`inv_id`值“丢失”。因此，`inv_item`表中的每个后续插入将上一行的`item_id`值存储到`inv_id`列中。这导致第二行及后续行的`inv_id`值不正确。

为避免这种困难，请保存插入主表生成的序列值，并在插入详细表时使用保存的值。要保存该值，请使用用户定义的 SQL 变量或程序维护的变量。Recipe 15.10 描述了这些技术，在此适用如下：

+   使用用户定义的变量：将主行`AUTO_INCREMENT`值保存在用户定义的变量中，以便在插入详细行时使用：

    ```
    INSERT INTO invoice (inv_id,date)
      VALUES(NULL,CURDATE());
    SET @inv_id = LAST_INSERT_ID();
    INSERT INTO inv_item (inv_id,qty,description)
      VALUES(@inv_id,1,'hammer');
    INSERT INTO inv_item (inv_id,qty,description)
      VALUES(@inv_id,3,'nails, box');
    INSERT INTO inv_item (inv_id,qty,description)
      VALUES(@inv_id,12,'bandage');
    ```

+   使用程序维护的变量：此方法类似于前面的方法，但仅适用于 API 内部。插入主行，然后将`AUTO_INCREMENT`值保存到 API 变量中，在插入详细行时使用。例如，在 Ruby 中，使用`last_id`方法访问`AUTO_INCREMENT`值：

    ```
    client.query("INSERT INTO invoice (inv_id,date) VALUES(NULL,CURDATE())")
    inv_id = client.last_id
    sth = client.prepare("INSERT INTO inv_item (inv_id,qty,description)
     VALUES(?,?,?)")
    sth.execute(inv_id, 1, "hammer")
    sth.execute(inv_id, 3, "nails, box")
    sth.execute(inv_id, 12, "bandage")
    ```

# 15.12 使用序列生成器作为计数器

## 问题

您只对计数事件感兴趣，因此希望避免为每个序列值创建新的表行。

## 解决方案

每个计数器递增单行。

## 讨论

`AUTO_INCREMENT` 列对于在一组单独的行中生成序列非常有用。但有些应用程序仅需要计数事件发生的次数，并且创建单独行并不会有任何好处。例如，网页或横幅广告点击计数器、销售商品的计数或投票数量。这些应用程序只需一个行来保存随时间变化的计数。MySQL 提供了一种机制，使得可以像处理 `AUTO_INCREMENT` 值一样处理计数，因此不仅可以增加计数，还可以轻松检索更新后的值。

若要计数单一类型的事件，使用一个带有单行和单列的简单表。例如，记录出售书籍的副本数，创建如下表：

```
CREATE TABLE booksales (copies INT UNSIGNED);
```

然而，如果您要计算多本书的销售次数，这种方法效果不佳。您肯定不希望为每本书创建单独的单行计数表。相反，通过在单个表中包含一个唯一标识每本书的列来在同一表中计数它们所有。以下表格使用 `title` 列用于书名，另外一个 `copies` 列记录销售副本数：

```
CREATE TABLE booksales
(
  title   VARCHAR(60) NOT NULL,   # book title
  copies  INT UNSIGNED NOT NULL,  # number of copies sold
  PRIMARY KEY (title)
);
```

记录给定书籍的销售，有多种方法：

+   为书籍初始化一行，`copies` 值为 0：

    ```
    INSERT INTO booksales (title,copies) VALUES('The Greater Trumps',0);
    ```

    然后为每个销售递增 `copies` 值：

    ```
    UPDATE booksales SET copies = copies+1 WHERE title = 'The Greater Trumps';
    ```

    此方法要求您记得为每本书初始化一行，否则 `UPDATE` 将失败。

+   使用 `INSERT` 和 `ON` `DUPLICATE` `KEY` `UPDATE`，它会为首次销售初始化计数为 1 的行，并递增后续销售的计数：

    ```
    INSERT INTO booksales (title,copies)
    VALUES('The Greater Trumps',1)
    ON DUPLICATE KEY UPDATE copies = copies+1;
    ```

    这更简单，因为相同的语句用于初始化和更新销售计数。

检索销售计数（例如，向客户显示消息，如<q>你刚购买了此书的第 *`n`* 本</q>），发出 `SELECT` 查询获取相同书名：

```
SELECT copies FROM booksales WHERE title = 'The Greater Trumps';
```

不幸的是，这并不完全正确。假设在您更新和检索计数之间的时间段内，其他人购买了这本书的一本副本（从而增加了 `copies` 值）。那么 `SELECT` 语句实际上不会产生您增加的销售计数值，而是最近的值。换句话说，其他客户端可能在您检索之前影响值。这类似于在 Recipe 15.4 中讨论的问题，如果您尝试通过调用 `MAX(`*`col_name`*`)` 而不是 `LAST_INSERT_ID()` 来检索列中最近的 `AUTO_INCREMENT` 值时可能会出现的问题。

有一些方法可以解决这个问题（例如将两个语句作为事务分组或锁定表），但 MySQL 基于 `LAST_INSERT_ID()` 提供了一个更简单的解决方案。如果您使用表 `booksales`，稍微修改增加计数的语句如下：

```
INSERT INTO booksales (title,copies)
VALUES('The Greater Trumps',LAST_INSERT_ID(1))
ON DUPLICATE KEY UPDATE copies = LAST_INSERT_ID(copies+1);
```

该语句同时用于初始化和递增计数的 `LAST_INSERT_ID(`*`expr`*`)` 构造。MySQL 将表达式参数视为 `AUTO_INCREMENT` 值，因此稍后可以调用不带参数的 `LAST_INSERT_ID()` 来检索该值：

```
SELECT LAST_INSERT_ID();
```

通过这种方式设置和检索 `copies` 列，即使其他客户端在此期间对其进行了更新，您也始终可以获得您设置的值。如果您在提供直接获取最新 `AUTO_INCREMENT` 值机制的 API 内发出 `INSERT` 语句，甚至无需发出 `SELECT` 查询。例如，使用 Connector/Python 更新计数并使用 `lastrowid` 属性获取新值：

```
cursor = conn.cursor()
cursor.execute('''
 INSERT INTO booksales (title,copies)
 VALUES('The Greater Trumps',LAST_INSERT_ID(1))
 ON DUPLICATE KEY UPDATE copies = LAST_INSERT_ID(copies+1)
 ''')
count = cursor.lastrowid
cursor.close()
conn.commit()
```

在 Java 中，操作如下所示：

```
Statement s = conn.createStatement ();
s.executeUpdate (
    "INSERT INTO booksales (title,copies)"
    + "VALUES('The Greater Trumps',LAST_INSERT_ID(1))"
    + "ON DUPLICATE KEY UPDATE copies = LAST_INSERT_ID(copies+1)",
    Statement.RETURN_GENERATED_KEYS);
long count;
ResultSet rs = s.getGeneratedKeys ();
if (rs.next ())
{
  count = rs.getLong (1);
}
else
{
  throw new SQLException ("getGeneratedKeys() produced no value");
}
rs.close ();
s.close ();
```

使用 `LAST_INSERT_ID(`*`expr`*`)` 生成序列时具有某些与真正的 `AUTO_INCREMENT` 序列不同的属性：

+   `AUTO_INCREMENT` 值每次递增 1，而由 `LAST_INSERT_ID(`*`expr`*`)` 生成的值可以是任何您想要的非负值。例如，要生成序列 10, 20, 30, …，每次递增 10。甚至您无需每次以相同的值递增计数器。如果您出售一本书的一打副本而不是单本，更新其销售计数如下：

    ```
    INSERT INTO booksales (title,copies)
    VALUES('The Greater Trumps',LAST_INSERT_ID(12))
    ON DUPLICATE KEY UPDATE copies = LAST_INSERT_ID(copies+12);
    ```

+   为了重置计数器，只需将其设置为所需的值。假设您想向图书买家报告本月的销售情况，而不是总销售额（例如，显示类似于 <q>你是本月的第 *`n`* 位买家</q> 的消息）。为了在每个月初将计数器清零，请使用以下语句：

    ```
    UPDATE booksales SET copies = 0;
    ```

+   其中一个不太理想的属性是，由 `LAST_INSERT_ID(`*`expr`*`)` 生成的值在所有情况下都不能通过客户端检索方法均匀地获取。您可以在 `UPDATE` 或 `INSERT` 语句之后获取它，但不能在 `SET` 语句中获取。如果您像以下示例（在 Ruby 中）生成值，则 `insert_id` 返回的客户端值为 0，而不是 48：

    ```
    client.query("SET @x = LAST_INSERT_ID(48)")
    seq = client.last_id
    ```

    在这种情况下获取值，向服务器请求即可：

    ```
    seq = client.query("SELECT LAST_INSERT_ID()").first.values[0]
    ```

# 15.13 生成重复序列

## 问题

您需要一个包含周期的序列。

## 解决方案

使用除法和模运算在序列中创建周期。

## 讨论

有些序列生成问题需要经历循环值。假设您制造药品或汽车零件等物品，并且如果稍后发现需要召回特定批次内出售的物品，则必须能够通过批号跟踪它们。假设您将物品装箱并分发，每箱 12 个单位，每箱 6 个箱子，这种情况下，物品标识符是三部分值：单位号（值从 1 到 12）、箱子号（值从 1 到 6）和批号（值从 1 到当前最高箱数）。

这个项目跟踪问题似乎要求您维护三个计数器，因此您可以使用类似以下算法来生成下一个标识符值：

```
retrieve most recently used case, box, and unit numbers
unit = unit + 1     # increment unit number
if (unit > 12)      # need to start a new box?
{
  unit = 1          # go to first unit of next box
  box = box + 1
}
if (box > 6)        # need to start a new case?
{
  box = 1           # go to first box of next case
  case = case + 1
}
store new case, box, and unit numbers
```

或者，可以简单地为每个物品分配一个序列号标识符，并从中派生相应的案例、箱子和单位号。标识符可以来自于`AUTO_INCREMENT`列或单行序列生成器。用于根据其序列号确定任何物品的案例、箱子和单位号的公式如下所示：

```
unit_num = ((seq - 1) % 12) + 1
box_num = (int ((seq - 1) / 12) % 6) + 1
case_num = int ((seq - 1)/(6 * 12)) + 1
```

以下表格展示了一些样本序列号与相应的案例、箱子和单元号之间的关系：

| seq | case | box | unit |
| --- | --- | --- | --- |
| 1 | 1 | 1 | 1 |
| 12 | 1 | 1 | 12 |
| 13 | 1 | 2 | 1 |
| 72 | 1 | 6 | 12 |
| 73 | 2 | 1 | 1 |
| 144 | 2 | 6 | 12 |

# 15.14 使用自定义增量值

## 问题

您希望递增序列不是一，而是另一个数字。

## 解决方案

使用系统变量`auto_increment_increment`和`auto_increment_offset`。

## 讨论

默认情况下，MySQL 通过一个具有`AUTO_INCREMENT`选项的列递增值。这并不总是理想的。假设您有三台服务器的复制链（Recipe 3.9）：`Venus`、`Mars`、`Saturn`，并且希望区分插入值的来源服务器。

解决此问题的最简单方法是为在`Venus`上插入的行分配`1, 4, 7, 10, ...`序列值；为在`Mars`上插入的行分配`2, 5, 8, 11, ...`序列值；为在`Saturn`上插入的行分配`3, 6, 9, 12, ...`序列值。

要执行此操作，请将系统变量`auto_increment_increment`的值设置为服务器数量：在我们的情况下为三（3），因此 MySQL 将按三递增序列值。然后在`Venus`上将`auto_increment_offset`设置为 1，在`Mars`上设置为 2，在`Saturn`上设置为 3。这将指示 MySQL 从指定值开始新的序列。

```
Venus>  `SET` `auto_increment_offset``=``1``;`
Query OK, 0 rows affected (0.00 sec)

Venus>  `SET` `auto_increment_increment``=``3``;`
Query OK, 0 rows affected (0.00 sec)

Mars>  `SET` `auto_increment_offset``=``2``;`
Query OK, 0 rows affected (0.00 sec)

Mars>  `SET` `auto_increment_increment``=``3``;`
Query OK, 0 rows affected (0.00 sec)

Saturn> `SET` `auto_increment_offset``=``3``;`
Query OK, 0 rows affected (0.00 sec)

Saturn> `SET` `auto_increment_increment``=``3``;`
Query OK, 0 rows affected (0.00 sec)
```

###### 警告

我们为示例设置了会话变量，但如果您不仅想影响自己的会话，还想影响服务器上所有连接，您需要使用*SET GLOBAL*。要在重新启动后保留配置更改，请在配置文件中设置这些值，或者从版本 8.0 开始，使用*SET PERSIST*命令。

如果您已经有带有自增列的表，请使用语句指定偏移量：

```
ALTER TABLE *`mytable`* AUTO_INCREMENT = *`N`*;
```

###### 警告

并非所有引擎都支持在*CREATE TABLE*和*ALTER TABLE*中使用`AUTO_INCREMENT`选项。在这种情况下，您可以通过插入具有所需值的行，然后删除它来设置自增列的起始值。

在准备工作完成后，MySQL 将使用`auto_increment_increment`值生成下一个序列号。

```
Venus>  `CREATE` `TABLE` `offset``(`
    ->  `id` `INT` `NOT` `NULL` `AUTO_INCREMENT` `PRIMARY` `KEY``,`
    ->  `host` `CHAR``(``32``)`
    ->  `)``;`
Query OK, 0 rows affected (0.03 sec)

Venus>  `INSERT` `INTO` `offset``(``host``)` `VALUES``(``@``@``hostname``)``;` ![1](img/1.png)
Query OK, 1 row affected (0.01 sec)

Venus>  `INSERT` `INTO` `offset``(``host``)` `VALUES``(``@``@``hostname``)``;`
Query OK, 1 row affected (0.01 sec)

Venus>  `INSERT` `INTO` `offset``(``host``)` `VALUES``(``@``@``hostname``)``;`
Query OK, 1 row affected (0.01 sec)

Venus>  `SELECT` `*` `FROM` `offset``;` ![2](img/2.png)
+----+-------+ | id | host  |
+----+-------+ |  1 | Venus |
|  4 | Venus |
|  7 | Venus |
+----+-------+ 3 rows in set (0.00 sec)

Mars>  `ALTER` `TABLE` `offset` `AUTO_INCREMENT``=``2``;`  ![3](img/3.png)
Query OK, 0 rows affected (0.36 sec)
Records: 0  Duplicates: 0  Warnings: 0

Mars>  `INSERT` `INTO` `offset``(``host``)` `VALUES``(``'Mars'``)``;`
Query OK, 1 row affected (0.00 sec)

Mars>  `INSERT` `INTO` `offset``(``host``)` `VALUES``(``'Mars'``)``;`
Query OK, 1 row affected (0.01 sec)

Mars>  `SELECT` `*` `FROM` `offset``;`
+----+-------+ | id | host  |
+----+-------+ |  1 | Venus |
|  4 | Venus |
|  7 | Venus |
|  8 | Mars  |  ![4](img/4.png)
| 11 | Mars  |
+----+-------+ 5 rows in set (0.00 sec)
```

![1](img/#co_nch-sequences-seq-increment_increment-hostname_co)

系统变量`hostname`包含 MySQL 主机的值。我们用它来区分不同的机器。

![2](img/#co_nch-sequences-seq-increment_increment-venus_co)

在`Venus`上，序列从一开始，我们有预期的值：`1, 4, 7`。

![3](img/#co_nch-sequences-seq-increment_increment-alter_co)

在`Mars`上的表已经存在。*ALTER TABLE*命令设置`AUTO_INCREMENT`序列的偏移量为所需的值。

![4](img/#co_nch-sequences-seq-increment_increment-mars_co)

由于`offset`表在 Mars 上已经有行，新的`AUTO_INCREMENT`值从 8 开始，属于序列`2, 5, 8, 11, ...`。

# 15.15 使用窗口函数为结果集编号行

## 问题

您希望枚举`SELECT`查询的结果。

## 解决方案

使用窗口函数`ROW_NUMBER()`。

## 讨论

序列不仅在将数据存储在表中时有用，而且在处理查询结果时也很有用。

假设您正在进行歌唱比赛。每个才能应该按顺序呈现。为了让每个人都有平等的机会，队列中的位置应该随机定义。

天赋存储在`name`表中。要以随机顺序检索它们，请使用函数*RAND()*：

```
mysql> `SELECT` `first_name``,` `last_name` `FROM` `name` `ORDER` `BY` `RAND``(``)``;`
+------------+-----------+ | first_name | last_name |
+------------+-----------+ | Pete       | Gray      |
| Vida       | Blue      |
| Rondell    | White     |
| Kevin      | Brown     |
| Devon      | White     |
+------------+-----------+ 5 rows in set (0.00 sec)
```

每次调用此查询都会返回不同顺序的名称列表。

窗口函数可以针对结果集中的每一行执行计算，我们可以使用它们创建一个新列，其中包含歌手表演的顺序。

窗口函数在我们的情况下适用于特定窗口，即`SELECT`查询。它们在执行时可以访问多行，但为窗口中的每行生成结果。

```
mysql>  SELECT
    ->  ROW_NUMBER() OVER *`win`* AS turn, ![1](img/1.png)
    ->  first_name, last_name FROM name  ![2](img/2.png)
    ->  WINDOW *`win`* ![3](img/3.png)
    ->  AS (ORDER BY RAND()); ![4](img/4.png)
+------+------------+-----------+
| turn | first_name | last_name |
+------+------------+-----------+
|    1 | Devon      | White     |
|    2 | Kevin      | Brown     |
|    3 | Rondell    | White     |
|    4 | Vida       | Blue      |
|    5 | Pete       | Gray      |
+------+------------+-----------+
5 rows in set (0.00 sec)
```

![1](img/#co_nch-sequences-seq-window-functions-over_co)

函数*ROW_NUMBER()*定义了歌唱日程中的位置。

![2](img/#co_nch-sequences-seq-window-functions-table_co)

我们希望在查询结果中看到`name`表中的其他列。

![3](img/#co_nch-sequences-seq-window-functions-window_co)

关键字`WINDOW`定义了一个命名窗口，我们将在其中使用函数*ROW_NUMBER*。

![4](img/#co_nch-sequences-seq-window-functions-order_co)

将窗口随机排序以获得公平的队列分布。

函数*ROW_NUMBER()*的另一个常见用途是生成一系列标识符，稍后可以用于将`SELECT`结果与另一个表连接。我们在食谱 15.16 的一个示例中讨论了这种方法。

## 参见

有关窗口函数的更多信息，请参阅[窗口函数概念和语法](https://dev.mysql.com/doc/refman/8.0/en/window-functions-usage.html)。

# 15.16 使用递归 CTE 生成系列

## 问题

您想创建一个自定义序列，例如几何级数或斐波那契数。

## 解决方案

使用递归公共表表达式（CTEs）根据自定义公式创建序列。

## 讨论

序列不应总是算术级数。它们可以是任何类型的级数，甚至是随机数或字符串。

创建自定义序列的一种方法是使用递归 CTE。它们是允许自引用的命名临时结果集。基本的递归 CTE 语法是：

```
WITH RECURSIVE *`name`*(*`column`*[, *`column`*])
(SELECT *`expressin`*[, *`expression`*]
UNION ALL 
SELECT *`expressin`*[, *`expression`*]
FROM name WHERE ...)
SELECT * FROM name;
```

因此，要生成从二开始，公比为二的几何级数，请使用以下 CTE。

```
mysql>  `WITH` `RECURSIVE` `geometric_progression``(``id``)` `AS`
    ->  `(``SELECT` `2` ![1](img/1.png)
    ->  `UNION` `ALL`
    ->  `SELECT` `id` `*` `2` ![2](img/2.png)
    ->  `FROM` `geometric_progression`
    ->  `LIMIT` `5``)` ![3](img/3.png)
    ->  `SELECT` `*` `FROM` `geometric_progression``;`
+------+ | id   |
+------+ |    2 |
|    4 |
|    8 |
|   16 |
|   32 |
+------+ 5 rows in set (0.00 sec)
```

![1](img/#co_nch-sum-sum-recursive-cte-init_co)

序列的起始值。

![2](img/#co_nch-sum-sum-recursive-cte-next_co)

几何级数中的所有后续值均为前一个数字乘以公比。

![3](img/#co_nch-sum-sum-recursive-cte-limit_co)

为了限制生成的数字数量并避免无限循环，可以使用`LIMIT`子句或任何有效的`WHERE`条件。

递归 CTE 允许同时创建多个序列。例如，我们可以使用它们来创建：

+   一个 ID，将使用常规算术级数，从一开始，公差为一

+   从三开始，公比为四的几何级数

+   一个介于一和五之间的随机数

要在单个查询中创建所有这些内容，请使用以下递归 CTE。

```
mysql>  `WITH` `RECURSIVE` `sequences``(``id``,` `geo``,` `random``)` `AS`
    ->  `(``SELECT` `1``,` `3``,` `FLOOR``(``1``+``RAND``(``)``*``5``)`
    ->  `UNION` `ALL`
    ->  `SELECT` `id` `+` `1``,` `geo` `*` `4``,` `FLOOR``(``1``+``RAND``(``)``*``5``)`
    ->  `FROM` `sequences`
    ->  `WHERE` `id` `<` `5``)`
    ->  `SELECT` `*` `FROM` `sequences``;`
+------+------+--------+ | id   | geo  | random |
+------+------+--------+ |    1 |    3 |      4 |
|    2 |   12 |      4 |
|    3 |   48 |      2 |
|    4 |  192 |      2 |
|    5 |  768 |      3 |
+------+------+--------+ 5 rows in set (0.00 sec)
```

为了说明自定义序列的使用，假设我们正在开发一种新的数据恐惧疫苗，并希望开始其第三阶段试验。第三阶段包括对真实疫苗和安慰剂进行测试。剂量将随机分配给志愿者。为了执行此试验，我们将使用`patients`表，并选取尚未诊断为数据恐惧的志愿者。我们生成一系列两个随机值，并根据结果分配真实疫苗或安慰剂。

```
mysql>  `WITH` `RECURSIVE` `trial``(``id``,` `dose``)` `AS`
    ->  `(``SELECT` `1``,` `IF``(``1``=``FLOOR``(``1``+``RAND``(``)``*``2``)``,` `'Vaccine'``,` `'Placebo'``)` ![1](img/1.png)
    ->   `UNION` `ALL`
    ->   `SELECT` `id``+``1``,` `IF``(``1``=``FLOOR``(``1``+``RAND``(``)``*``2``)``,` `'Vaccine'``,` `'Placebo'``)`
    ->     `FROM` `trial`
    -> `WHERE` `id` `<` `(``SELECT` `COUNT``(``*``)` `FROM` `patients` 
    ->                    `WHERE` `diagnosis` `!``=` `'Data Phobia'` `and` `result` `!``=` `'D'``)``)``,` ![2](img/2.png)
    ->   `volunteers` `AS` ![3](img/3.png)
    ->  `(``SELECT` `ROW_NUMBER``(``)` `OVER` `win` `AS` `id``,` ![4](img/4.png)
    ->          `national_id``,` `name``,` `surname`
    ->   `FROM` `patients` `WHERE` `diagnosis` `!``=` `'Data Phobia'` `and` `result` `!``=` `'D'`
    ->  `WINDOW` `win` `AS` `(``ORDER` `BY` `surname``)``)`
    ->  `SELECT` `national_id``,` `name``,` `surname``,` `dose` ![5](img/5.png)
    ->  `FROM` `trial` `JOIN` `volunteers` `USING``(``id``)``;`
+-------------+-----------+-----------+---------+ | national_id | name      | surname   | dose    |
+-------------+-----------+-----------+---------+ | 84DC051879  | William   | Brown     | Vaccine |
| 78FS043029  | David     | Davis     | Vaccine |
| 38BP394037  | Catherine | Hernandez | Placebo |
| 28VU492728  | Alice     | Jackson   | Vaccine |
| 71GE601633  | John      | Johnson   | Vaccine |
| 09SK434607  | Richard   | Martin    | Placebo |
| 30NC108735  | Robert    | Martinez  | Placebo |
| 02WS884704  | Sarah     | Miller    | Placebo |
| 45MY529190  | Patricia  | Rodriguez | Vaccine |
| 89AR642465  | Mary      | Smith     | Placebo |
| 99XC682639  | Emma      | Taylor    | Vaccine |
| 04WT954962  | Peter     | Wilson    | Vaccine |
+-------------+-----------+-----------+---------+ 12 rows in set (0.00 sec)
```

![1](img/#co_nch-sum-sum-recursive-cte-options_co)

函数*FLOOR(1+RAND()*2)*生成两个随机数：一或二。函数*IF*充当三元运算符：如果第一个参数为真，则返回第二个参数，否则返回第三个参数。

![2](img/#co_nch-sum-sum-recursive-cte-count_co)

我们不希望已经被诊断为数据恐惧的患者参与我们的测试，同样我们也不能在未康复的患者身上测试我们的疫苗。

![3](img/#co_nch-sum-sum-recursive-cte-volunteers_co)

尽管`patients`表具有`AUTO_INCREMENT`列`id`，但我们不能使用它，因为这样无法排除不参与测试的患者。因此，我们使用 CTE 创建命名结果集`volunteers`并为其生成自己的序列。

![4](img/#co_nch-sum-sum-recursive-cte-rownumber_co)

函数 *ROW_NUMBER()* 为参与测试的患者生成新的序列。

![5](img/#co_nch-sum-sum-recursive-cte-join_co)

将随机值生成的剂量和命名结果集 `volunteers` 的生成 id 连接，但不包括在最终结果集中。

## See Also

有关公共表达式的更多信息，请参见 Recipe 10.18。

# 15.17 创建和存储自定义序列

## 问题

您想要在表中使用自定义序列作为存储的 id 列。

## 解决方案

创建一个用于保存序列值的表，并创建一个函数来更新和选择这些值。

## 讨论

尽管 MySQL 不支持 SQL 的 `SEQUENCE` 对象，但模拟一个相当容易。

首先，您需要创建一个用于保存序列的表。

```
CREATE TABLE `sequences` (
  `sequence_name` varchar(64) NOT NULL,
  `maximum_value` bigint NOT NULL DEFAULT '9223372036854775807',
  `minimum_value` bigint NOT NULL DEFAULT '-9223372036854775808',
  `increment` bigint NOT NULL DEFAULT '1',
  `start_value` bigint NOT NULL DEFAULT '-9223372036854775808',
  `current_base_value` bigint NOT NULL DEFAULT '-9223372036854775808',
  `cycle_option` enum('yes','no') NOT NULL DEFAULT 'no',
  PRIMARY KEY (`sequence_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

对于这个示例，我们使用了与 MySQL 工程团队计划实现的 [WL#827: SEQUENCE object as in Oracle, PostgreSQL, and/or SQL:2003](https://dev.mysql.com/worklog/task/?id=827) 相同的表定义。这个定义对于实际序列实现并不是必须的，可能更简单或包含更多选项。

表 `sequences` 中的列都有特殊含义。 表 15-1 显示了它们的含义。

表 15-1\. 表 `sequences` 中的列

| 列 | 描述 | 注释 |
| --- | --- | --- |
| `sequence_name` | 序列的名称。 | 必需字段，应该是唯一的。 |
| `maximum_value` | 序列可以生成的最大值。 | 在我们的自定义序列中，我们允许负值，因此最大可能值是 9223372036854775807，这是`BIGINT SIGNED`数据类型的最大值。如果将此列设置为`BIGINT UNSIGNED`，则序列可以有两倍的值。这个选项对序列生成并不是关键，可以跳过。 |
| `minimum_value` | 序列的最小值。 | 在我们的案例中，默认值是 -9223372036854775808，这是 `BIGINT SIGNED` 类型的最小值。根据您想要创建的自定义序列方式，此列可能会被省略或具有不同的类型或不同的默认值。 |
| `increment` | 序列的增量。 | SQL 标准定义了使用算术级数的序列。该列包含了进度的公共差异。这是必需的字段。如果您创建自定义序列，比如等比级数，您可能在此字段中有一个公共比率或任何其他值，以便生成下一个值。 |
| `start_value` | 序列将从哪个值开始。 | 对于实现句子并不是必要的字段。在我们的案例中，默认值是 `minimum_value`。 |
| `current_base_value` | 当序列请求下一个值时应返回的值。返回后应替换为新生成的值。 | 这是必需的字段。默认值与 `start_value` 相同。 |
| `cycle_option` | 序列是否支持循环？ | 如果启用，当序列达到其`minimum_value`或`maximum_value`时，序列将重置为`start_value`。 |

然后我们需要创建一个存储过程，该过程将更新表`sequences`。

```
CREATE PROCEDURE create_sequence( 
    sequence_name VARCHAR(64), start_value BIGINT, increment BIGINT, 
    cycle_option ENUM('yes','no'), maximum_value BIGINT, minimum_value BIGINT) 
BEGIN 
    INSERT INTO sequences 
        (sequence_name, maximum_value, minimum_value, increment, start_value, 
         current_base_value, cycle_option) 
        VALUES(
            sequence_name, 
            COALESCE(maximum_value, 9223372036854775807), 
            COALESCE(minimum_value, -9223372036854775808), 
            COALESCE(increment, 1), 
            COALESCE(start_value, -9223372036854775808), 
            COALESCE(start_value, -9223372036854775808), 
            COALESCE(cycle_option, 'no'));
END;
```

###### 注意

使用存储过程而不是直接更新表`sequences`具有许多优点：

+   每次使用序列时无需关心如何更新`current_base_value`。

+   如果启用了`cycle_option`，则无需放置代码，当达到序列边界时每次都会重置`current_base_value`。

+   你可以限制除管理员外的任何人对表`sequences`的直接访问，同时允许应用程序用户使用序列。详细信息请参见 Recipe 24.13。

MySQL 不允许我们使用可变数量的参数调用存储函数。函数*COALESCE*允许在希望使用默认值的参数位置传递`NULL`值时放置默认值。

```
mysql>  `CALL` `create_sequence``(``'bar'``,` `1``,` `1``,` `'no'``,` `9223372036854775807``,` `-``9223372036854775808``)``;`
Query OK, 1 row affected (0.01 sec)

mysql>  `CALL` `create_sequence``(``'baz'``,` `1``,` `1``,` `'yes'``,` `10``,` `1``)``;`
Query OK, 1 row affected (0.01 sec)

mysql>  `call` `create_sequence``(``'foo'``,``null``,``null``,``null``,` `null``,` `null``)``;`
Query OK, 1 row affected (0.00 sec)

mysql>  `SELECT` `*` `FROM` `sequences``\``G`
*************************** 1. row ***************************
     sequence_name: bar
     maximum_value: 9223372036854775807
     minimum_value: 1
         increment: 1
       start_value: 1
current_base_value: 1
      cycle_option: no
*************************** 2. row ***************************
     sequence_name: baz
     maximum_value: 10
     minimum_value: 1
         increment: 1
       start_value: 1
current_base_value: 1
      cycle_option: yes
*************************** 3. row ***************************
     sequence_name: foo
     maximum_value: 9223372036854775807
     minimum_value: -9223372036854775808
         increment: 1
       start_value: -9223372036854775808
current_base_value: -9223372036854775808
      cycle_option: no
3 rows in set (0.00 sec)
```

在上面的示例中，我们首先创建了序列`bar`，从 1 开始，增量为 1，不具有循环选项，并具有默认的`maximum_value` 9223372036854775807。然后，我们创建了序列`baz`，也从 1 开始，增量为 1，但启用了`cycle_option`，`maximum_value`为 10，因此循环速度很快。最后，我们创建了序列`foo`，只具有自定义名称和所有其他默认值。

要获取下一个序列值并同时更新序列表，我们将使用一个存储函数。

```
CREATE FUNCTION sequence_next_value(name varchar(64)) RETURNS BIGINT 
BEGIN 
    DECLARE retval BIGINT; 
    SELECT current_base_value INTO retval FROM sequences 
      WHERE sequence_name=name FOR UPDATE; 
    UPDATE sequences SET current_base_value=
        IF((current_base_value+increment <= maximum_value 
            AND current_base_value+increment >= minimum_value), 
            current_base_value+increment, 
            IF('yes' = cycle_option, start_value, NULL)
        ) WHERE sequence_name=name;
    RETURN retval; 
END;
```

该函数首先使用语句`SELECT ... FOR UPDATE`检索序列的`current_base_value`，因此在我们返回值之前，其他连接不会修改序列。

我们的函数支持循环。在启用`cycle_option`且下一个序列值超出边界的情况下，它将`current_base_value`设置为由`start_value`定义的值。如果未启用`cycle_option`且下一个序列值超出边界，则会向列`current_base_value`插入`NULL`值，MySQL 将拒绝此操作并报错。您可以考虑抛出自定义异常。

为了演示`cycle_option`选项的工作原理，让我们看看当序列`baz`达到其边界时的行为。

```
mysql>  `SELECT` `sequence_next_value``(``'baz'``)``;`
+----------------------------+ | sequence_next_value('baz') |
+----------------------------+ |                         10 |
+----------------------------+ 1 row in set (0.00 sec)

mysql>  `SELECT` `sequence_next_value``(``'baz'``)``;`
+----------------------------+ | sequence_next_value('baz') |
+----------------------------+ |                          1 |
+----------------------------+ 1 row in set (0.01 sec)

mysql>  `SELECT` `sequence_next_value``(``'baz'``)``;`
+----------------------------+ | sequence_next_value('baz') |
+----------------------------+ |                          2 |
+----------------------------+ 1 row in set (0.01 sec)
```

为了演示`cycle_option`未启用时边界达到时函数的行为，我们创建了一个具有较小最大值的序列。

```
mysql>  `CALL` `create_sequence``(``'boo'``,` `1``,` `1``,` `'no'``,` `3``,` `1``)``;`
Query OK, 1 row affected (0.01 sec)

mysql>  `SELECT` `sequence_next_value``(``'boo'``)``;`
+----------------------------+ | sequence_next_value('boo') |
+----------------------------+ |                          1 |
+----------------------------+ 1 row in set (0.01 sec)

mysql>  `SELECT` `sequence_next_value``(``'boo'``)``;`
+----------------------------+ | sequence_next_value('boo') |
+----------------------------+ |                          2 |
+----------------------------+ 1 row in set (0.01 sec)

mysql>  `SELECT` `sequence_next_value``(``'boo'``)``;`
ERROR 1048 (23000): Column 'current_base_value' cannot be null
```

要在表上使用自定义序列，只需在需要下一个序列值时调用`sequence_next_value`即可。

```
mysql> `CREATE` `TABLE` `sequence_test``(` 
    ->  `id` `BIGINT` `NOT` `NULL` `PRIMARY` `KEY``,`
    ->  `-- other fields`
    ->  `)``;`
Query OK, 0 rows affected (0.04 sec)

mysql>  `CALL` `create_sequence``(``'sequence_test'``,` `10``,` `5``,` `'no'``,` `null``,` `null``)``;`
Query OK, 1 row affected (0.00 sec)

mysql>  `INSERT` `INTO` `sequence_test` `VALUES``(``sequence_next_value``(``'sequence_test'``)``)``;`
Query OK, 1 row affected (0.01 sec)

mysql>  `INSERT` `INTO` `sequence_test` `VALUES``(``sequence_next_value``(``'sequence_test'``)``)``;`
Query OK, 1 row affected (0.01 sec)

mysql>  `select` `*` `from` `sequence_test``;`
+----+ | id |
+----+ | 10 |
| 15 |
+----+ 2 rows in set (0.00 sec)
```

如果使用触发器，您可以自动化表的序列值生成。

```
CREATE TRIGGER sequence_test_bi BEFORE INSERT ON sequence_test 
FOR EACH ROW SET NEW.id=IFNULL(NEW.id, sequence_next_value('sequence_test'));
```

在此示例中，当用户尝试将`NULL`插入到表`sequence_test`的`id`列时，我们生成新的序列值。如果用户决定显式指定值，则触发器不会更改它。

```
mysql>  `INSERT` `INTO` `sequence_test` `VALUES``(``)``;`
Query OK, 1 row affected (0.01 sec)

mysql>  `INSERT` `INTO` `sequence_test` `VALUES``(``13``)``;`
Query OK, 1 row affected (0.00 sec)

mysql>  `select` `*` `from` `sequence_test``;`
+----+ | id |
+----+ | 10 |
| 13 |
| 15 |
| 20 |
+----+ 4 rows in set (0.00 sec)
```

最后，我们需要定义一个存储过程来在不需要时删除序列。

```
CREATE PROCEDURE delete_sequence(name VARCHAR(64)) 
DELETE FROM sequences WHERE sequence_name=name;
```

你会在`recipes`分发的文件`sequences/custom_sequences.sql`中找到维护自定义序列的代码。
