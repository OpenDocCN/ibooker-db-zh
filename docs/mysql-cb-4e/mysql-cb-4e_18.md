# 第十八章：处理重复行

# 18.0 Introduction

表格或结果集有时会包含重复行。在某些情况下这是可以接受的。例如，如果你进行了记录日期、客户 IP 地址以及投票结果的网络民意调查，重复行可能是允许的，因为大量的投票可能看起来来自同一个 IP 地址，这是因为某些互联网服务通过单一代理主机路由客户流量。在其他情况下，重复行是不可接受的，你会想要采取措施来避免它们。处理重复行涉及以下操作：

+   首先防止创建重复行。如果表中的每一行都代表一个单一实体（例如一个人、目录中的一个项目或实验中的特定观察），重复的发生会使得无法明确地引用每一行，因此最好确保永远不会发生重复。

+   计算重复行的数量，以确定它们的存在程度。

+   识别重复值（或包含它们的行），以便看到它们出现的位置。

+   消除重复行以确保每一行都是唯一的。这可能包括从表中删除行，只留下唯一的行，或者选择一个结果集以使输出中不出现重复。例如，要显示你的客户所在的各州列表，你可能不希望看到来自所有客户记录的长长的州名称列表。只显示每个州名称一次就足够了，而且更容易理解。

有多种工具可供处理重复行。根据你想要实现的目标选择这些工具：

+   创建表时，包括主键或唯一索引，以防止重复行添加到表中。MySQL 使用索引作为约束来强制要求表中的每一行在索引列或列组中包含唯一键。

+   与唯一索引结合使用，`INSERT IGNORE`和`REPLACE`语句使您能够优雅地处理重复行的插入，而不会产生错误。对于批量加载操作，可以使用`LOAD DATA`语句的`IGNORE`或`REPLACE`修饰符来实现相同的选项。

+   要确定表是否包含重复值，可以使用`GROUP BY`将行分组，并使用`COUNT()`查看每个组中有多少行。第十章在生成摘要的上下文中描述了这些技术，但它们也适用于重复计数和识别。计数摘要将值分组为类别，以确定每个值出现的频率。

+   `SELECT` `DISTINCT`从结果集中删除重复行（参见 Recipe 5.4 获取更多信息）。对于已包含重复项的现有表，您可以选择唯一行到第二个表并用其替换原始表。或者，如果确定表中有*`n`*个相同行，可以使用`DELETE` … `LIMIT`从该特定行集中消除*`n`*–1 个实例。

本章节展示的示例相关脚本位于`recipes`发行版的*dups*目录中。要查找创建这里使用的表的脚本，请查看*tables*目录。

# 18.1 在表中防止重复项的出现

## 问题

您希望防止表中永远包含重复项。

## 解决方案

使用`PRIMARY` `KEY`或`UNIQUE`索引。

## 讨论

要确保表中的行是唯一的，某些列或列组合必须要求每行包含唯一值。当满足此要求时，可以使用其唯一标识符明确引用表中的任何行。为确保表具有此特性，请在表结构中包含`PRIMARY` `KEY`或`UNIQUE`索引。以下表格不包含此类索引，因此允许重复行：

```
CREATE TABLE person
(
  last_name   CHAR(20),
  first_name  CHAR(20),
  address     CHAR(40)
);
```

要防止在此表中创建具有相同名字和姓氏值的多行，请在其定义中添加`PRIMARY` `KEY`。在执行此操作时，索引列必须为`NOT` `NULL`，因为`PRIMARY` `KEY`不允许`NULL`值：

```
CREATE TABLE person
(
  last_name   CHAR(20) NOT NULL,
  first_name  CHAR(20) NOT NULL,
  address     CHAR(40),
  PRIMARY KEY (last_name, first_name)
);
```

表中的唯一索引存在时，如果向表中插入与定义索引的列中的现有行重复的行，则通常会导致错误。Recipe 18.3 讨论如何处理此类错误或修改 MySQL 的重复处理行为。

强制实现唯一性的另一种方法是向表中添加`UNIQUE`索引而不是`PRIMARY` `KEY`。这两种索引类型类似，但可以在允许`NULL`值的列上创建`UNIQUE`索引。对于`person`表，可能需要填写名字和姓氏。如果是这样，仍然将列声明为`NOT` `NULL`，并且以下表定义与前面的表定义实际上是等效的：

```
CREATE TABLE person
(
  last_name   CHAR(20) NOT NULL,
  first_name  CHAR(20) NOT NULL,
  address     CHAR(40),
  UNIQUE (last_name, first_name)
);
```

如果`UNIQUE`索引确实允许`NULL`值，那么`NULL`是特殊的，因为它是唯一可能多次出现的值。其理由在于无法确定一个未知值是否与另一个相同，因此允许多个未知值。

当然，您可能希望 `person` 表反映现实世界，其中人们有时确实会有相同的姓名。在这种情况下，您不能基于姓名列设置唯一索引，因为必须允许重复的姓名。相反，每个人必须被分配某种唯一标识符，这成为区分一行与另一行的值。在 MySQL 中，通常通过使用 `AUTO_INCREMENT` 列来实现这一点：

```
CREATE TABLE person
(
  id          INT UNSIGNED NOT NULL AUTO_INCREMENT,
  last_name   CHAR(20),
  first_name  CHAR(20),
  address     CHAR(40),
  PRIMARY KEY (id)
);
```

在这种情况下，如果使用 `NULL` 值的 `id` 创建行，则 MySQL 会自动分配该列的唯一 ID。另一种可能性是外部分配标识符并将这些 ID 用作唯一键。例如，特定国家的公民可能具有唯一的纳税人身份证号码。如果是这样，这些号码可以作为唯一索引的基础：

```
CREATE TABLE person
(
  tax_id      INT UNSIGNED NOT NULL,
  last_name   CHAR(20),
  first_name  CHAR(20),
  address     CHAR(40),
  PRIMARY KEY (tax_id)
);
```

## 参见

如果现有表已包含您希望删除的重复行，请参见 Recipe 18.5。第十五章 进一步讨论了 `AUTO_INCREMENT` 列。

# 18.2 在表中拥有多个唯一键

## 问题

您需要在表中具有两个或更多列集合，这些列集合需要具有唯一值。

## 解决方案

定义所需的多个唯一键。

## 讨论

可能存在两个或多个需要独立具有唯一值的列组合。例如，在 Recipe 18.1 中的最后一个示例中的 `person` 表具有代表纳税人 ID 的 `tax_id` 列，因此需要存储唯一值。仍然可能希望在 `(last_name, first_name)` 上保持唯一索引。这样，您可以确保每个人都有自己的纳税人 ID，而任何纳税人 ID 都属于唯一的人。

任何表最多只能有一个主键。因此，您需要选择哪个键将成为主键，哪个键将成为次要的唯一键。正如我们在 Recipe 21.2 中描述的那样，对于 InnoDB 存储引擎，主键包含在所有次要索引中，使用尽可能小的数据类型来定义它们对性能至关重要。因此，可以直接为 `tax_id` 列定义主键，以及在 `(last_name, first_name)` 上定义一个次要的唯一索引。

结果表定义如下所示：

```
CREATE TABLE `person` (
  `tax_id` INT UNSIGNED NOT NULL,
  `last_name` CHAR(20) DEFAULT NULL,
  `first_name` CHAR(20) DEFAULT NULL,
  `address` CHAR(40) DEFAULT NULL,
  PRIMARY KEY (`tax_id`),
  UNIQUE KEY `last_name` (`last_name`,`first_name`)
);
```

# 18.3 在将行加载到表中时处理重复项

## 问题

您已创建了一个带有唯一索引的表，以防止索引列或列中的重复值。但是，如果尝试插入重复行，则会出现错误，您希望避免处理此类错误。

## 解决方案

一种方法是简单地忽略错误。另一种方法是使用 `INSERT IGNORE`、`REPLACE` 或 `INSERT ... ON DUPLICATE KEY UPDATE` 语句，每种方法都会修改 MySQL 的重复处理行为。对于批量加载操作，`LOAD DATA` 有修饰符，允许您指定如何处理重复项。

## 讨论

默认情况下，当插入重复现有唯一键值的行时，MySQL 会生成错误。假设`person`表具有以下结构，并在`last_name`和`first_name`列上有唯一索引：

```
CREATE TABLE person
(
  last_name   CHAR(20) NOT NULL,
  first_name  CHAR(20) NOT NULL,
  address     CHAR(40),
  PRIMARY KEY (last_name, first_name)
);
```

尝试在索引列中插入具有重复值的行会导致错误：

```
mysql> `INSERT INTO person (last_name, first_name)`
    -> `VALUES('Pinter', 'Marlene');`
Query OK, 1 row affected (0.00 sec)
mysql> `INSERT INTO person (last_name, first_name)`
    -> `VALUES('Pinter', 'Marlene');`
ERROR 1062 (23000): Duplicate entry 'Pinter-Marlene' for key 'person.PRIMARY'
```

如果您在*mysql*程序中交互地发出这些语句，您可以简单地说：<q>好吧，那不起作用，</q>忽略错误，然后继续。但如果你编写一个程序来插入行，错误可能会终止程序。避免这种情况的一种方法是修改程序的错误处理行为以捕获错误，然后忽略它。参见 Recipe 4.2 获取有关错误处理技术的信息。

要在第一次就防止错误发生，可以考虑使用两个查询方法来解决重复行问题：

+   发出`SELECT`以检查行是否已存在。

+   如果行不存在，则发出`INSERT`。

但这并不起作用：另一个客户可能会在`SELECT`和`INSERT`之间插入相同的行，这样你的`INSERT`仍然会出错。为了确保这种情况不会发生，你可以使用事务或锁定表，但这样一来，你的两个语句就变成了四个。MySQL 提供了处理重复行问题的三种单查询解决方案。根据你想要的重复处理行为，从中选择一种：

+   若要在重复出现时保留原始行，请使用`INSERT` `IGNORE`而不是`INSERT`。如果行不重复现有行，则 MySQL 会像往常一样插入它。如果行是重复的，则`IGNORE`关键字告诉 MySQL 静默丢弃它，而不生成错误：

    ```
    mysql> `INSERT IGNORE INTO person (last_name, first_name)`
        -> `VALUES('Brown', 'Bartholomew');`
    Query OK, 1 row affected (0.00 sec)
    mysql> `INSERT IGNORE INTO person (last_name, first_name)`
        -> `VALUES('Brown', 'Bartholomew');`
    Query OK, 0 rows affected, 1 warning (0.00 sec)
    ```

    行计数值指示是否插入或忽略了行。从程序内部，您可以通过检查 API 提供的影响行数函数来获取此值（参见 Recipe 4.4 和 Recipe 12.1）。

+   若要在重复出现时用新行替换原始行，请使用`REPLACE`而不是`INSERT`。如果行是新的，则像`INSERT`一样插入它。如果是重复的，则新行将替换旧行：

    ```
    mysql> `REPLACE INTO person (last_name, first_name, address)`
        -> `VALUES('Baxter', 'Wallace', '57 3rd Ave.');`
    Query OK, 1 row affected (0.00 sec)
    mysql> `REPLACE INTO person (last_name, first_name, address)`
        -> `VALUES('Baxter', 'Wallace', '57 3rd Ave., Apt 102');`
    Query OK, 2 rows affected (0.00 sec)
    ```

    在第二种情况下，受影响的行数为 2，因为原始行被删除，新行插入到其位置。

+   要在重复出现时修改现有行的列，请使用`INSERT`…`ON` `DUPLICATE` `KEY` `UPDATE`。如果行是新的，则插入它。如果是重复的，则`ON` `DUPLICATE` `KEY` `UPDATE`子句指示如何在表中修改现有行。换句话说，此语句可以根据需要插入或更新行。受影响的行数指示发生了什么：插入为 1，更新为 2。

`INSERT` `IGNORE`比`REPLACE`更高效，因为它实际上不会插入重复项。因此，在您只想确保表中存在给定行的副本时，它最为适用。另一方面，`REPLACE`通常更适用于需要替换其他非关键列的表。`INSERT` … `ON` `DUPLICATE` `KEY` `UPDATE`在必须在记录不存在时插入记录，但如果新记录在索引列中是重复的，则仅更新其中一些列时非常适用。

假设您为包含电子邮件地址和密码哈希值的 Web 应用程序维护名为`passtbl`的表，并且该表由电子邮件地址索引：

```
CREATE TABLE passtbl
(
  email    VARCHAR(60) NOT NULL,
  password VARBINARY(60) NOT NULL,
  PRIMARY KEY (email)
);
```

如何为新用户创建新行，但为现有用户更改现有行的密码？以下是处理行维护的典型算法：

1.  发出`SELECT`以检查是否已存在具有给定`email`值的行。

1.  如果不存在这样的行，请使用`INSERT`添加新行。

1.  如果行存在，则使用`UPDATE`更新它。

必须在事务内执行这些步骤或锁定表，以防其他用户在您使用表时更改表。在 MySQL 中，您可以使用`REPLACE`来简化这两种情况为同一个单语句操作：

```
REPLACE INTO passtbl (email,password) VALUES(*`address`*,*`hash_value`*);
```

如果不存在具有给定电子邮件地址的行，则 MySQL 创建一个新行。否则，MySQL 替换它，实际上更新与该地址相关的行的`password`列。

`INSERT` `IGNORE`和`REPLACE`在您尝试插入行时知道确切的值应存储在表中时很有用。情况并非总是如此。例如，您可能希望仅在行不存在时插入一行，否则仅更新其中的某些部分。这在您使用表进行计数时经常发生。假设您在选举中记录候选人的投票，使用以下表格：

```
CREATE TABLE poll_vote
(
  poll_id      INT UNSIGNED NOT NULL AUTO_INCREMENT,
  candidate_id INT UNSIGNED,
  vote_count   INT UNSIGNED,
  PRIMARY KEY (poll_id, candidate_id)
);
```

主键是投票和候选人编号的组合。应该像这样使用表：

+   对于给定投票候选人的第一次投票收到，插入一个新行，投票计数为 1。

+   对于该候选人的后续投票，增加现有记录的投票计数。

在这里既不适用`INSERT` `IGNORE`也不适用`REPLACE`，因为除第一次投票外，您不知道投票计数应该是多少。在这种情况下，`INSERT` … `ON` `DUPLICATE` `KEY` `UPDATE`更为合适。以下示例显示了它的工作原理，从空表开始：

```
mysql> `SELECT * FROM poll_vote;`
Empty set (0.00 sec)
mysql> `INSERT INTO poll_vote (poll_id,candidate_id,vote_count) VALUES(14,3,1)`
    -> `ON DUPLICATE KEY UPDATE vote_count = vote_count + 1;`
Query OK, 1 row affected (0.00 sec)
mysql> `SELECT * FROM poll_vote;`
```

```
+---------+--------------+------------+
| poll_id | candidate_id | vote_count |
+---------+--------------+------------+
|      14 |            3 |          1 |
+---------+--------------+------------+
1 row in set (0.00 sec)
mysql> `INSERT INTO poll_vote (poll_id,candidate_id,vote_count) VALUES(14,3,1)`
    -> `ON DUPLICATE KEY UPDATE vote_count = vote_count + 1;`
Query OK, 2 rows affected (0.00 sec)
mysql> `SELECT * FROM poll_vote;`
+---------+--------------+------------+
| poll_id | candidate_id | vote_count |
+---------+--------------+------------+
|      14 |            3 |          2 |
+---------+--------------+------------+
1 row in set (0.00 sec)
```

对于第一个`INSERT`，不存在候选人的行，因此插入该行。对于第二个`INSERT`，行存在，因此 MySQL 只更新投票计数。使用`INSERT` … `ON` `DUPLICATE` `KEY` `UPDATE`，您无需检查行是否存在；MySQL 会为您执行。行计数指示`INSERT`语句执行的操作：对于新行为 1，对于现有行的更新为 2。

刚才描述的技术的好处是减少可能需要的事务开销。但是，这种好处是以可移植性为代价的，因为它们都涉及特定于 MySQL 的语法。如果可移植性是高优先级的话，您可能更喜欢使用我们在 第二十章 中讨论的事务性方法。

## 参见

对于使用 `LOAD` `DATA` 语句从文件加载一组行到表中进行批量记录加载操作，请使用该语句的 `IGNORE` 和 `REPLACE` 修饰符来控制重复行处理。这些修饰符的行为类似于 `INSERT` `IGNORE` 和 `REPLACE` 语句。有关更多信息，请参阅 Recipe 13.1。

Recipe 15.12 进一步展示了使用 `INSERT` … `ON` `DUPLICATE` `KEY` `UPDATE` 来初始化和更新计数的方法。

# 18.4 计数和识别重复项

## 问题

您想确定表中是否存在重复项，以及它们的程度。或者您想查看包含重复值的行。

## 解决方案

使用显示重复值的计数摘要。要查看包含重复值的行，请将摘要与原始表连接，以显示匹配的行。

## 讨论

假设您的网站有一个注册页面，访客可以在该页面上将自己添加到邮寄产品目录的邮件列表中。但是，当您创建表时，忘记在表中包含唯一索引，现在您怀疑有些人多次注册。也许他们忘记了已经在列表上，或者可能有人添加了已经在列表上的朋友。无论如何，重复行的结果是您会重复邮寄目录。这对您来说是额外的开支，也会令收件人感到烦恼。本节讨论如何确定表中是否存在重复行，它们的普遍程度以及如何显示它们。（对于包含重复项的表，Recipe 18.5 描述了如何消除它们。）

要确定表中是否存在重复项，请使用计数摘要（在 第十章 中讨论）。摘要技术可应用于通过 `GROUP` `BY` 对行进行分组并使用 `COUNT()` 计数每个组中的行来识别和计数重复项。在这里的示例中，假设收件人目录在名为 `catalog_list` 的表中列出，其内容如下：

```
+-----------+-------------+--------------------------+
| last_name | first_name  | street                   |
+-----------+-------------+--------------------------+
| Isaacson  | Jim         | 515 Fordam St., Apt. 917 |
| Baxter    | Wallace     | 57 3rd Ave.              |
| McTavish  | Taylor      | 432 River Run            |
| Pinter    | Marlene     | 9 Sunset Trail           |
| BAXTER    | WALLACE     | 57 3rd Ave.              |
| Brown     | Bartholomew | 432 River Run            |
| Pinter    | Marlene     | 9 Sunset Trail           |
| Baxter    | Wallace     | 57 3rd Ave., Apt 102     |
+-----------+-------------+--------------------------+
```

假设您使用 `last_name` 和 `first_name` 列来定义 <q>重复</q>。也就是说，拥有相同姓名的收件人被视为同一人。以下语句描述了表格并评估了重复值的存在和程度：

+   表中的总行数：

    ```
    mysql> `SELECT COUNT(*) AS rows FROM catalog_list;`
    +------+
    | rows |
    +------+
    |    8 |
    +------+
    ```

+   不同名称的数量：

    ```
    mysql> `SELECT COUNT(DISTINCT last_name, first_name) AS 'distinct names'`
        -> `FROM catalog_list;`
    +----------------+
    | distinct names |
    +----------------+
    |              5 |
    +----------------+
    ```

+   含有重复名称的行数：

    ```
    mysql> `SELECT COUNT(*) - COUNT(DISTINCT last_name, first_name)`
        ->   `AS 'duplicate names'`
        -> `FROM catalog_list;`
    +-----------------+
    | duplicate names |
    +-----------------+
    |               3 |
    +-----------------+
    ```

+   包含唯一或非唯一名称的行的比例：

    ```
    mysql> `SELECT COUNT(DISTINCT last_name, first_name) / COUNT(*)`
        ->   `AS 'unique',`
        -> `1 - (COUNT(DISTINCT last_name, first_name) / COUNT(*))`
        ->   `AS 'nonunique'`
        -> `FROM catalog_list;`
    +--------+-----------+
    | unique | nonunique |
    +--------+-----------+
    | 0.6250 |    0.3750 |
    +--------+-----------+
    ```

这些语句帮助您描述重复项的范围，但不显示重复的值。要查看`catalog_list`表中重复的名称，请使用显示非唯一值以及计数的汇总语句：

```
mysql> `SELECT COUNT(*), last_name, first_name`
    -> `FROM catalog_list`
    -> `GROUP BY last_name, first_name`
    -> `HAVING COUNT(*) > 1;`
+----------+-----------+------------+
| COUNT(*) | last_name | first_name |
+----------+-----------+------------+
|        3 | Baxter    | Wallace    |
|        2 | Pinter    | Marlene    |
+----------+-----------+------------+
```

语句包含一个`HAVING`子句，该子句限制输出仅包括出现多次的名称。通常，要识别重复的值集，请执行以下操作：

1.  确定包含可能重复值的列。

1.  在列选择列表中列出这些列，并包括`COUNT(*)`。

1.  在`GROUP` `BY`子句中列出列。

1.  添加一个`HAVING`子句，通过要求组计数大于一来消除唯一值。

以以下形式构建的查询：

```
SELECT COUNT(*), *`column_list`*
FROM *`tbl_name`*
GROUP BY *`column_list`*
HAVING COUNT(*) > 1;
```

在程序内很容易生成此类查找重复项的查询，只需提供数据库和表名以及非空列名集。例如，下面是一个 Perl 函数`make_dup_count_query()`，用于生成查找和计算指定列中重复值的正确查询：

```
sub make_dup_count_query
{
my ($db_name, $tbl_name, @col_name) = @_;

  return "SELECT COUNT(*)," . join (",", @col_name)
         . "\nFROM $db_name.$tbl_name"
         . "\nGROUP BY " . join (",", @col_name)
         . "\nHAVING COUNT(*) > 1";
}
```

`make_dup_count_query()`将查询作为字符串返回。如果像这样调用它：

```
$str = make_dup_count_query ("cookbook", "catalog_list",
                             "last_name", "first_name");
```

`$str`的结果是：

```
SELECT COUNT(*),last_name,first_name
FROM cookbook.catalog_list
GROUP BY last_name,first_name
HAVING COUNT(*) > 1;
```

对查询字符串的处理方式取决于您。您可以在创建它的脚本中执行它，将它传递给另一个程序，或者将其写入文件以供稍后执行。`recipes`分发的*dups*目录包含一个名为*dup_count.pl*的脚本，您可以使用它来尝试该函数（以及其他语言的翻译）。Recipe 18.5 讨论了使用`make_dup_count_query()`来实现重复删除技术。

摘要技术对于评估重复项的存在性、频率以及显示哪些值重复是有用的。但是，如果仅使用表的一部分列确定重复项，则汇总本身无法显示包含重复值的行的整个内容。（例如，迄今显示的摘要仅显示了`catalog_list`表中重复名称的计数或名称本身，但没有显示与这些名称相关联的地址。）要查看包含重复名称的原始行，请将汇总信息与生成它的表进行连接。以下示例显示如何执行此操作以显示包含重复名称的`catalog_list`行。摘要写入临时表，然后与`catalog_list`表连接以生成与这些名称匹配的行：

```
mysql> `CREATE TABLE tmp`
    -> `SELECT COUNT(*) AS count, last_name, first_name FROM catalog_list`
    -> `GROUP BY last_name, first_name HAVING count > 1;`
mysql> `SELECT catalog_list.*`
    -> `FROM tmp INNER JOIN catalog_list USING (last_name, first_name)`
    -> `ORDER BY last_name, first_name;`
+-----------+------------+----------------------+
| last_name | first_name | street               |
+-----------+------------+----------------------+
| Baxter    | Wallace    | 57 3rd Ave.          |
| BAXTER    | WALLACE    | 57 3rd Ave.          |
| Baxter    | Wallace    | 57 3rd Ave., Apt 102 |
| Pinter    | Marlene    | 9 Sunset Trail       |
| Pinter    | Marlene    | 9 Sunset Trail       |
+-----------+------------+----------------------+
```

# 18.5 从表中消除重复项

## 问题

您希望从表中删除重复行，仅保留唯一行。

## 解决方案

从表中选择唯一行到第二个表中，然后使用该表替换原始表。或者使用`DELETE` … `LIMIT` *`n`*来删除特定一组重复行的所有实例之外的所有行。

## 讨论

配方 18.1 讨论了如何通过创建具有唯一索引的表来防止将重复项添加到表中。然而，如果在创建表时忘记包括索引，则可能会后来发现它包含重复项，并且需要应用某种去重技术。之前使用的`catalog_list`表就是一个例子，因为它包含多个相同人物出现多次的情况：

```
mysql> `SELECT * FROM catalog_list ORDER BY last_name, first_name;`
+-----------+-------------+--------------------------+
| last_name | first_name  | street                   |
+-----------+-------------+--------------------------+
| Baxter    | Wallace     | 57 3rd Ave.              |
| BAXTER    | WALLACE     | 57 3rd Ave.              |
| Baxter    | Wallace     | 57 3rd Ave., Apt 102     |
| Brown     | Bartholomew | 432 River Run            |
| Isaacson  | Jim         | 515 Fordam St., Apt. 917 |
| McTavish  | Taylor      | 432 River Run            |
| Pinter    | Marlene     | 9 Sunset Trail           |
| Pinter    | Marlene     | 9 Sunset Trail           |
+-----------+-------------+--------------------------+
```

要消除重复项，您可以使用以下两种选项之一：

+   选择表的唯一行并插入另一个表中，然后使用该表替换原始表。这在“重复”表示“整行与另一行完全相同”的情况下有效。

+   对于特定集合的重复行，使用`DELETE` … `LIMIT` *`n`*来删除除一行外的所有行。

本配方讨论了每种去重方法。在为您的情况选择方法时，请考虑以下问题：

+   方法是否要求表具有唯一索引？

+   如果表中可能包含`NULL`的列中有重复值，该方法会移除重复的`NULL`值吗？

+   该方法是否能防止将来发生重复项？

### 使用表替换删除重复项

如果仅当整行完全相同时才认为一行是另一行的副本，那么从一个表中选择其唯一行到一个具有相同结构的新表中，然后用新表替换原始表是消除重复项的一种方法：

1.  创建具有与原始表相同结构的新表。使用`CREATE` `TABLE` … `LIKE`对此非常有用（见配方 6.1）：

    ```
    mysql> `CREATE TABLE tmp LIKE catalog_list;`
    ```

1.  使用`INSERT` `INTO` … `SELECT` `DISTINCT`将原始表中的唯一行选择到新表中：

    ```
    mysql> `INSERT INTO tmp SELECT DISTINCT * FROM catalog_list;`
    ```

    从`tmp`表中选择行以验证新表不包含重复项：

    ```
    mysql> `SELECT * FROM tmp ORDER BY last_name, first_name;`
    +-----------+-------------+--------------------------+
    | last_name | first_name  | street                   |
    +-----------+-------------+--------------------------+
    | Baxter    | Wallace     | 57 3rd Ave.              |
    | Baxter    | Wallace     | 57 3rd Ave., Apt 102     |
    | Brown     | Bartholomew | 432 River Run            |
    | Isaacson  | Jim         | 515 Fordam St., Apt. 917 |
    | McTavish  | Taylor      | 432 River Run            |
    | Pinter    | Marlene     | 9 Sunset Trail           |
    +-----------+-------------+--------------------------+
    ```

1.  创建包含唯一行的新`tmp`表后，使用它来替换原始`catalog_list`表：

    ```
    mysql> `DROP TABLE catalog_list;`
    mysql> `RENAME TABLE tmp TO catalog_list;`
    ```

此过程的有效结果是`catalog_list`不再包含重复项。

这种表替换方法适用于没有索引的情况（尽管对于大表可能会比较慢）。对于包含重复`NULL`值的表，它会删除这些重复项。它不能防止未来重复项的出现。

该方法要求行完全相同才被视为重复。因此，它将对 Wallace Baxter 的那些`street`值略有不同的行视为不同。

如果仅当表中列的子集存在重复值时，可以创建一个具有这些列的唯一索引的新表，使用`INSERT` `IGNORE`将行插入其中，并用新表替换原始表：

```
mysql> `CREATE TABLE tmp LIKE catalog_list;`
mysql> `ALTER TABLE tmp ADD PRIMARY KEY (last_name, first_name);`
mysql> `INSERT IGNORE INTO tmp SELECT * FROM catalog_list;`
mysql> `SELECT * FROM tmp ORDER BY last_name, first_name;`
+-----------+-------------+--------------------------+
| last_name | first_name  | street                   |
+-----------+-------------+--------------------------+
| Baxter    | Wallace     | 57 3rd Ave.              |
| Brown     | Bartholomew | 432 River Run            |
| Isaacson  | Jim         | 515 Fordam St., Apt. 917 |
| McTavish  | Taylor      | 432 River Run            |
| Pinter    | Marlene     | 9 Sunset Trail           |
+-----------+-------------+--------------------------+
mysql> `DROP TABLE catalog_list;`
mysql> `RENAME TABLE tmp TO catalog_list;`
```

唯一索引防止具有重复键值的行插入到 `tmp` 中，而 `IGNORE` 告诉 MySQL 如果发现重复项，则不要停止并显示错误。这种方法的一个缺点是，如果索引列可以包含 `NULL` 值，则必须使用 `UNIQUE` 索引而不是 `PRIMARY` `KEY`，在这种情况下，索引将不会删除重复的 `NULL` 键（`UNIQUE` 索引允许多个 `NULL` 值）。这种方法确实可以防止将来出现重复。

### 删除特定行的重复项

您可以使用 `LIMIT` 限制 `DELETE` 语句的影响范围，以便仅适用于删除重复行的子集。假设原始的未索引 `catalog_list` 表包含重复项：

```
mysql> `SELECT COUNT(*), last_name, first_name`
    -> `FROM catalog_list`
    -> `GROUP BY last_name, first_name`
    -> `HAVING COUNT(*) > 1;`
+----------+-----------+------------+
| COUNT(*) | last_name | first_name |
+----------+-----------+------------+
|        3 | Baxter    | Wallace    |
|        2 | Pinter    | Marlene    |
+----------+-----------+------------+
```

要删除每个名称的额外实例，请执行以下操作：

```
mysql> `DELETE FROM catalog_list WHERE last_name = 'Baxter'`
    -> `AND first_name = 'Wallace' LIMIT 2;`
mysql> `DELETE FROM catalog_list WHERE last_name = 'Pinter'`
    -> `AND first_name = 'Marlene' LIMIT 1;`
mysql> `SELECT * FROM catalog_list;`
+-----------+-------------+--------------------------+
| last_name | first_name  | street                   |
+-----------+-------------+--------------------------+
| Isaacson  | Jim         | 515 Fordam St., Apt. 917 |
| McTavish  | Taylor      | 432 River Run            |
| Brown     | Bartholomew | 432 River Run            |
| Pinter    | Marlene     | 9 Sunset Trail           |
| Baxter    | Wallace     | 57 3rd Ave., Apt 102     |
+-----------+-------------+--------------------------+
```

在没有唯一索引的情况下，这种技术可以消除重复的 `NULL` 值。这对于仅删除表内特定行集的重复项非常方便。但是，如果需要移除多个不同集合的重复项，则不建议手动进行此过程。可以通过使用食谱 18.4 中讨论的技术自动化此过程，以确定哪些值是重复的。在那里，我们编写了一个 `make_dup_count_query()` 函数来生成需要计算表中给定列集中重复值数量的语句。该语句的结果可以用来生成一组 `DELETE` … `LIMIT` *`n`* 语句，以删除重复行并仅保留唯一行。`recipes` 发行版的 *dups* 目录包含显示如何生成这些语句的代码。

通常情况下，使用 `DELETE` … `LIMIT` *`n`* 可能比通过使用第二个表或添加唯一索引来删除重复项要慢。这些方法将数据保留在服务器端，并让服务器完成所有工作。 `DELETE` … `LIMIT` *`n`* 包含大量的客户端-服务器交互，因为它使用 `SELECT` 语句来检索关于重复项的信息，然后使用多个 `DELETE` 语句来删除重复行的实例。此外，这种技术不能防止将来出现重复。
