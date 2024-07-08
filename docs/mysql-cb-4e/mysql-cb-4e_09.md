# 第九章：排序查询结果

# 9.0 介绍

本章涵盖了排序操作，这是控制 MySQL 从`SELECT`语句显示结果的极其重要的操作。要对查询结果进行排序，请向查询添加`ORDER` `BY`子句。如果没有这样的子句，MySQL 可以自由地以任何顺序返回行，因此排序有助于使混乱的结果有序化，使查询结果更易于检查和理解。

你可以按多种方式对查询结果的行进行排序：

+   使用单个列，多个列的组合，甚至列或表达式结果的部分

+   使用升序或降序排序

+   使用区分大小写或不区分大小写的字符串比较

+   使用时间顺序排序

本章中的多个示例使用`driver_log`表，该表包含用于记录一组卡车司机每日里程日志的列：

```
mysql> `SELECT * FROM driver_log;`
+--------+-------+------------+-------+
| rec_id | name  | trav_date  | miles |
+--------+-------+------------+-------+
|      1 | Ben   | 2014-07-30 |   152 |
|      2 | Suzi  | 2014-07-29 |   391 |
|      3 | Henry | 2014-07-29 |   300 |
|      4 | Henry | 2014-07-27 |    96 |
|      5 | Ben   | 2014-07-29 |   131 |
|      6 | Henry | 2014-07-26 |   115 |
|      7 | Suzi  | 2014-08-02 |   502 |
|      8 | Henry | 2014-08-01 |   197 |
|      9 | Ben   | 2014-08-02 |    79 |
|     10 | Henry | 2014-07-30 |   203 |
+--------+-------+------------+-------+
```

许多其他示例使用`mail`表（在较早的章节中使用）：

```
mysql> `SELECT t, srcuser, srchost, dstuser, dsthost, size FROM mail;`
+---------------------+---------+---------+---------+---------+---------+
| t                   | srcuser | srchost | dstuser | dsthost | size    |
+---------------------+---------+---------+---------+---------+---------+
| 2014-05-11 10:15:08 | barb    | saturn  | tricia  | mars    |   58274 |
| 2014-05-12 12:48:13 | tricia  | mars    | gene    | venus   |  194925 |
| 2014-05-12 15:02:49 | phil    | mars    | phil    | saturn  |    1048 |
| 2014-05-12 18:59:18 | barb    | saturn  | tricia  | venus   |     271 |
| 2014-05-14 09:31:37 | gene    | venus   | barb    | mars    |    2291 |
| 2014-05-14 11:52:17 | phil    | mars    | tricia  | saturn  |    5781 |
| 2014-05-14 14:42:21 | barb    | venus   | barb    | venus   |   98151 |
| 2014-05-14 17:03:01 | tricia  | saturn  | phil    | venus   | 2394482 |
| 2014-05-15 07:17:48 | gene    | mars    | gene    | saturn  |    3824 |
| 2014-05-15 08:50:57 | phil    | venus   | phil    | venus   |     978 |
| 2014-05-15 10:25:52 | gene    | mars    | tricia  | saturn  |  998532 |
| 2014-05-15 17:35:31 | gene    | saturn  | gene    | mars    |    3856 |
| 2014-05-16 09:00:28 | gene    | venus   | barb    | mars    |     613 |
| 2014-05-16 23:04:19 | phil    | venus   | barb    | venus   |   10294 |
| 2014-05-19 12:49:23 | phil    | mars    | tricia  | saturn  |     873 |
| 2014-05-19 22:21:51 | gene    | saturn  | gene    | venus   |   23992 |
+---------------------+---------+---------+---------+---------+---------+
```

有时也会偶尔使用其他表。要创建它们，请使用`recipes`分发的*tables*目录中的脚本。

# 9.1 使用 ORDER BY 对查询结果进行排序

## 问题

查询结果中的行不按你想要的顺序显示。

## 解决方案

在查询中添加`ORDER` `BY`子句来对其结果进行排序。

## 讨论

在章节介绍中显示的`driver_log`和`mail`表的内容是杂乱无章且难以理解的。`id`和`t`列中的值仅仅是巧合的顺序排列。

当你选择行时，它们会以服务器当前使用的任何顺序返回。关系数据库不保证以任何顺序返回行，除非你通过向你的`SELECT`语句添加`ORDER` `BY`子句来告诉它如何排序。如果没有`ORDER` `BY`，你可能会发现检索顺序随着时间的推移而更改，因为你修改表内容。使用`ORDER` `BY`子句，MySQL 始终按照你指定的方式对行进行排序。

`ORDER` `BY`具有以下一般特性：

+   你可以使用一个或多个列或表达式的值进行排序。

+   你可以独立按列升序（默认）或降序排序。

+   你可以通过名称或使用别名来引用排序列。

本篇介绍了一些基本的排序技巧，比如如何命名排序列和指定排序方向。本章后面的示例演示了如何执行更复杂的排序。具有讽刺意味的是，你甚至可以使用`ORDER` `BY`来*打乱*结果集，这对于随机化行或（与`LIMIT`一起）从结果集中随机选择行很有用（参见 Recipe 17.7 和 Recipe 17.8)。

下面的示例演示了如何按单个列或多个列进行排序，以及如何按升序或降序排序。这些示例选择`driver_log`表中的行，但以不同的`ORDER` `BY`子句来展示不同的效果。

此查询使用司机名称进行单列排序：

```
mysql> `SELECT * FROM driver_log ORDER BY name;`
+--------+-------+------------+-------+
| rec_id | name  | trav_date  | miles |
+--------+-------+------------+-------+
|      1 | Ben   | 2014-07-30 |   152 |
|      9 | Ben   | 2014-08-02 |    79 |
|      5 | Ben   | 2014-07-29 |   131 |
|      8 | Henry | 2014-08-01 |   197 |
|      6 | Henry | 2014-07-26 |   115 |
|      4 | Henry | 2014-07-27 |    96 |
|      3 | Henry | 2014-07-29 |   300 |
|     10 | Henry | 2014-07-30 |   203 |
|      7 | Suzi  | 2014-08-02 |   502 |
|      2 | Suzi  | 2014-07-29 |   391 |
+--------+-------+------------+-------+
```

默认的排序方向是升序。要使升序排序的方向明确，添加`ASC`到排序列名后：

```
SELECT * FROM driver_log ORDER BY name ASC;
```

升序排序的反义词（或反向）是降序排序，通过在排序列名后添加`DESC`来指定：

```
mysql> `SELECT * FROM driver_log ORDER BY name DESC;`
+--------+-------+------------+-------+
| rec_id | name  | trav_date  | miles |
+--------+-------+------------+-------+
|      2 | Suzi  | 2014-07-29 |   391 |
|      7 | Suzi  | 2014-08-02 |   502 |
|     10 | Henry | 2014-07-30 |   203 |
|      8 | Henry | 2014-08-01 |   197 |
|      6 | Henry | 2014-07-26 |   115 |
|      4 | Henry | 2014-07-27 |    96 |
|      3 | Henry | 2014-07-29 |   300 |
|      5 | Ben   | 2014-07-29 |   131 |
|      9 | Ben   | 2014-08-02 |    79 |
|      1 | Ben   | 2014-07-30 |   152 |
+--------+-------+------------+-------+
```

仔细检查刚才显示的查询的输出，你会注意到虽然按名称对行进行了排序，但任何给定名称的行没有特定的顺序。（例如，Henry 或 Ben 的`trav_date`值不按日期顺序排列。）这是因为 MySQL 不会对某些东西进行排序，除非你告诉它：

+   查询返回的行的整体顺序是不确定的，除非你指定了`ORDER` `BY`子句。

+   在根据给定列中的值排序的行组内，除非在`ORDER` `BY`子句中命名它们，否则其他列中的值的顺序也是不确定的。

要更全面地控制输出顺序，请通过逗号分隔的每个用于排序的列列表指定多列排序。以下查询按名称升序排序，并在每个名称的行内按`trav_date`升序排序：

```
mysql> `SELECT * FROM driver_log ORDER BY name, trav_date;`
+--------+-------+------------+-------+
| rec_id | name  | trav_date  | miles |
+--------+-------+------------+-------+
|      5 | Ben   | 2014-07-29 |   131 |
|      1 | Ben   | 2014-07-30 |   152 |
|      9 | Ben   | 2014-08-02 |    79 |
|      6 | Henry | 2014-07-26 |   115 |
|      4 | Henry | 2014-07-27 |    96 |
|      3 | Henry | 2014-07-29 |   300 |
|     10 | Henry | 2014-07-30 |   203 |
|      8 | Henry | 2014-08-01 |   197 |
|      2 | Suzi  | 2014-07-29 |   391 |
|      7 | Suzi  | 2014-08-02 |   502 |
+--------+-------+------------+-------+
```

多列排序也可以是降序，但必须在*每个*列名后指定`DESC`以执行完全降序排序。

多列`ORDER` `BY`子句可以执行混合顺序排序，其中某些列按升序排序，而其他列按降序排序。以下查询按名称降序排序，然后按每个名称的`trav_date`升序排序：

```
mysql> `SELECT * FROM driver_log ORDER BY name DESC, trav_date;`
+--------+-------+------------+-------+
| rec_id | name  | trav_date  | miles |
+--------+-------+------------+-------+
|      2 | Suzi  | 2014-07-29 |   391 |
|      7 | Suzi  | 2014-08-02 |   502 |
|      6 | Henry | 2014-07-26 |   115 |
|      4 | Henry | 2014-07-27 |    96 |
|      3 | Henry | 2014-07-29 |   300 |
|     10 | Henry | 2014-07-30 |   203 |
|      8 | Henry | 2014-08-01 |   197 |
|      5 | Ben   | 2014-07-29 |   131 |
|      1 | Ben   | 2014-07-30 |   152 |
|      9 | Ben   | 2014-08-02 |    79 |
+--------+-------+------------+-------+
```

到目前为止显示的查询中的`ORDER` `BY`子句是通过名称引用排序的列。你也可以使用别名命名列。也就是说，如果输出列有别名，你可以在`ORDER` `BY`子句中引用别名：

```
mysql> `SELECT name, trav_date, miles AS distance FROM driver_log`
    -> `ORDER BY distance;`
+-------+------------+----------+
| name  | trav_date  | distance |
+-------+------------+----------+
| Ben   | 2014-08-02 |       79 |
| Henry | 2014-07-27 |       96 |
| Henry | 2014-07-26 |      115 |
| Ben   | 2014-07-29 |      131 |
| Ben   | 2014-07-30 |      152 |
| Henry | 2014-08-01 |      197 |
| Henry | 2014-07-30 |      203 |
| Henry | 2014-07-29 |      300 |
| Suzi  | 2014-07-29 |      391 |
| Suzi  | 2014-08-02 |      502 |
+-------+------------+----------+
```

# 9.2 使用表达式进行排序

## 问题

你想根据从列计算而不是实际存储在列中的值对查询结果进行排序。

## 解决方案

将计算值的表达式放入`ORDER` `BY`子句。

## 讨论

`mail`表的一个列显示每个邮件消息的大小，以字节为单位：

```
mysql> `SELECT t, srcuser, srchost, dstuser, dsthost, size FROM mail;`
+---------------------+---------+---------+---------+---------+---------+
| t                   | srcuser | srchost | dstuser | dsthost | size    |
+---------------------+---------+---------+---------+---------+---------+
| 2014-05-11 10:15:08 | barb    | saturn  | tricia  | mars    |   58274 |
| 2014-05-12 12:48:13 | tricia  | mars    | gene    | venus   |  194925 |
| 2014-05-12 15:02:49 | phil    | mars    | phil    | saturn  |    1048 |
| 2014-05-12 18:59:18 | barb    | saturn  | tricia  | venus   |     271 |
…
```

假设你想检索超过 50,000 字节的邮件（定义为大邮件），但希望它们按千字节显示和排序大小。在这种情况下，排序的值由表达式计算：

```
FLOOR((size+1023)/1024)
```

在`FLOOR()`表达式中的`+1023`将`size`值分组到最接近的 1,024 字节类别的上限。如果没有它，值将按下限分组（例如，2,047 字节的消息报告为 1 千字节大小而不是 2）。Recipe 10.13 更详细地讨论了这种技术。

要按该表达式排序，请直接将其放入`ORDER` `BY`子句中：

```
mysql> `SELECT t, srcuser, FLOOR((size+1023)/1024)`
    -> `FROM mail WHERE size > 50000`
    -> `ORDER BY FLOOR((size+1023)/1024);`
+---------------------+---------+-------------------------+
| t                   | srcuser | FLOOR((size+1023)/1024) |
+---------------------+---------+-------------------------+
| 2014-05-11 10:15:08 | barb    |                      57 |
| 2014-05-14 14:42:21 | barb    |                      96 |
| 2014-05-12 12:48:13 | tricia  |                     191 |
| 2014-05-15 10:25:52 | gene    |                     976 |
| 2014-05-14 17:03:01 | tricia  |                    2339 |
+---------------------+---------+-------------------------+
```

或者，如果排序表达式出现在输出列列表中，您可以在那里对其进行别名，并在`ORDER` `BY`子句中引用该别名：

```
mysql> `SELECT t, srcuser, FLOOR((size+1023)/1024) AS kilobytes`
    -> `FROM mail WHERE size > 50000`
    -> `ORDER BY kilobytes;`
+---------------------+---------+-----------+
| t                   | srcuser | kilobytes |
+---------------------+---------+-----------+
| 2014-05-11 10:15:08 | barb    |        57 |
| 2014-05-14 14:42:21 | barb    |        96 |
| 2014-05-12 12:48:13 | tricia  |       191 |
| 2014-05-15 10:25:52 | gene    |       976 |
| 2014-05-14 17:03:01 | tricia  |      2339 |
+---------------------+---------+-----------+
```

由于几个原因，你可能更喜欢别名方法：

+   在`ORDER BY`子句中，写别名比重复（冗长的）表达式更容易。

+   如果没有别名，如果你在一个地方更改表达式，那么你必须在另一个地方也进行更改。

+   别名可能对显示目的很有用，可以提供更好的列标签。注意前面两个查询中第三列标题的意义大不相同。

# 9.3 显示一组值但按另一组值排序

## 问题

你想使用不出现在输出列列表中的值对结果集进行排序。

## 解决方案

这不是问题。`ORDER BY`子句可以引用你不显示的列。

## 讨论

`ORDER BY`不仅限于仅对输出列列表中命名的列进行排序。它可以使用那些“隐藏”的值进行排序（即在查询输出中不显示的值）。当你有不同表示方式的值，并且想要显示一种类型的值但按另一种类型排序时，这种技术通常被使用。例如，你可能希望将邮件消息大小显示为`103K`而不是字节。你可以使用以下表达式将字节计数转换为这种类型的值：

```
CONCAT(FLOOR((size+1023)/1024),'K')
```

然而，这些值是字符串，所以它们按字典顺序而不是数字顺序排序。如果你用它们来排序，像`96K`这样的值将排在`2339K`之后，尽管它代表一个较小的数字：

```
mysql> `SELECT t, srcuser,`
    -> `CONCAT(FLOOR((size+1023)/1024),'K') AS size_in_K`
    -> `FROM mail WHERE size > 50000`
    -> `ORDER BY size_in_K;`
+---------------------+---------+-----------+
| t                   | srcuser | size_in_K |
+---------------------+---------+-----------+
| 2014-05-12 12:48:13 | tricia  | 191K      |
| 2014-05-14 17:03:01 | tricia  | 2339K     |
| 2014-05-11 10:15:08 | barb    | 57K       |
| 2014-05-14 14:42:21 | barb    | 96K       |
| 2014-05-15 10:25:52 | gene    | 976K      |
+---------------------+---------+-----------+
```

为了达到期望的输出顺序，显示字符串，但使用实际的数字大小进行排序：

```
mysql> `SELECT t, srcuser,`
    -> `CONCAT(FLOOR((size+1023)/1024),'K') AS size_in_K`
    -> `FROM mail WHERE size > 50000`
    -> `ORDER BY size;`
+---------------------+---------+-----------+
| t                   | srcuser | size_in_K |
+---------------------+---------+-----------+
| 2014-05-11 10:15:08 | barb    | 57K       |
| 2014-05-14 14:42:21 | barb    | 96K       |
| 2014-05-12 12:48:13 | tricia  | 191K      |
| 2014-05-15 10:25:52 | gene    | 976K      |
| 2014-05-14 17:03:01 | tricia  | 2339K     |
+---------------------+---------+-----------+
```

将值显示为字符串但按数字排序有助于解决一些其他情况下难以解决的问题。体育队成员通常会被分配一个球衣号码，通常你可能认为应该使用一个数字列来存储它。不那么快！一些球员喜欢球衣号码为零（`0`），一些喜欢双零（`00`）。如果一支队伍恰好有两种球员，即使两个值是不同的数字，你也不能使用数字列来表示它们。为了解决这个问题，将球衣号码存储为字符串：

```
CREATE TABLE roster
(
  name        CHAR(30),   # player name
  jersey_num  CHAR(3),     # jersey number
  PRIMARY KEY(name)
);
```

然后球衣号码将以你输入的方式显示，并且`0`和`00`将被视为不同的值。不幸的是，虽然将数字表示为字符串解决了区分`0`和`00`的问题，但引入了另一个问题。假设一个球队有以下球员：

```
mysql> `SELECT name, jersey_num FROM roster;`
+-----------+------------+
| name      | jersey_num |
+-----------+------------+
| Lynne     | 29         |
| Ella      | 0          |
| Elizabeth | 100        |
| Nancy     | 00         |
| Jean      | 8          |
| Sherry    | 47         |
+-----------+------------+
```

现在尝试按球衣号码对团队成员进行排序。如果这些数字被存储为字符串，它们将按字典顺序排序，而字典顺序通常不同于数字顺序。对于所讨论的球队，这绝对是真实的：

```
mysql> `SELECT name, jersey_num FROM roster ORDER BY jersey_num;`
+-----------+------------+
| name      | jersey_num |
+-----------+------------+
| Ella      | 0          |
| Nancy     | 00         |
| Elizabeth | 100        |
| Lynne     | 29         |
| Sherry    | 47         |
| Jean      | 8          |
+-----------+------------+
```

值`100`和`8`错位了，但这很容易解决：显示字符串值，并使用数字值进行排序。为了实现这一点，将零添加到`jersey_num`的值中以强制进行字符串到数字的转换：

```
mysql> `SELECT name, jersey_num FROM roster ORDER BY jersey_num+0;`
+-----------+------------+
| name      | jersey_num |
+-----------+------------+
| Ella      | 0          |
| Nancy     | 00         |
| Jean      | 8          |
| Lynne     | 29         |
| Sherry    | 47         |
| Elizabeth | 100        |
+-----------+------------+
```

###### 警告

请注意，由于此方法执行字符串到数字的转换，它不能使用索引，并且在表变大时会运行得更慢。作为替代方案，您可以创建一个列，用于保存此计算的结果，并在 `ORDER BY` 表达式中使用它。

当您显示一个值但按另一个值排序的技术，在显示由多个列组成且无法按您希望的方式排序的值时，也很有用。例如，`mail` 表列出使用单独的 `srcuser` 和 `srchost` 值的消息发送者。要以 `srcuser@srchost` 格式显示 `mail` 表中的消息发送者作为电子邮件地址，并确保用户名排在前面，请使用以下表达式构造这些值：

```
CONCAT(srcuser,'@',srchost)
```

然而，如果要将主机名视为比用户名更重要的内容来排序，那些值就不适合排序。而是使用底层列值而不是显示的复合值来排序结果：

```
mysql> `SELECT t, CONCAT(srcuser,'@',srchost) AS sender, size`
    -> `FROM mail WHERE size > 50000`
    -> `ORDER BY srchost, srcuser;`
+---------------------+---------------+---------+
| t                   | sender        | size    |
+---------------------+---------------+---------+
| 2014-05-15 10:25:52 | gene@mars     |  998532 |
| 2014-05-12 12:48:13 | tricia@mars   |  194925 |
| 2014-05-11 10:15:08 | barb@saturn   |   58274 |
| 2014-05-14 17:03:01 | tricia@saturn | 2394482 |
| 2014-05-14 14:42:21 | barb@venus    |   98151 |
+---------------------+---------------+---------+
```

同样的想法通常也适用于排序人名。假设 `names` 表包含姓和名。要按姓氏首先显示行，当分别显示列时，查询非常简单：

```
mysql> `SELECT last_name, first_name FROM name`
    -> `ORDER BY last_name, first_name;`
+-----------+------------+
| last_name | first_name |
+-----------+------------+
| Blue      | Vida       |
| Brown     | Kevin      |
| Gray      | Pete       |
| White     | Devon      |
| White     | Rondell    |
+-----------+------------+
```

如果您想将每个姓名显示为由名字、一个空格和姓组成的单个字符串，查询可以如下开始：

```
SELECT CONCAT(first_name,' ',last_name) AS full_name FROM name ...
```

但是如何按姓氏顺序排序这些姓名呢？显示复合姓名，但在 `ORDER BY` 子句中引用各组成值：

```
mysql> `SELECT CONCAT(first_name,' ',last_name) AS full_name`
    -> `FROM name`
    -> `ORDER BY last_name, first_name;`
+---------------+
| full_name     |
+---------------+
| Vida Blue     |
| Kevin Brown   |
| Pete Gray     |
| Devon White   |
| Rondell White |
+---------------+
```

# 9.4 控制字符串排序的大小写敏感性

## 问题

当您不希望字符串排序操作区分大小写时，或者反之。

## 解决方案

更改排序值的比较特性。

## 讨论

食谱 7.1 讨论了字符串比较属性如何取决于字符串是二进制还是非二进制的：

+   二进制字符串是字节序列。它们按照数值字节值逐字节比较。字符集和大小写对比较无意义。

+   非二进制字符串是字符序列。它们具有字符集和排序规则，并且按照由排序规则定义的顺序逐字符比较。

这些属性也适用于字符串排序，因为排序基于比较。要修改字符串列的排序属性，请修改其比较属性。（有关字符串数据类型是二进制还是非二进制的摘要，请参阅 食谱 7.2。）

本节中的示例使用一个具有大小写不敏感和大小写敏感的非二进制列以及一个二进制列的表：

```
CREATE TABLE str_val
(
  ci_str   CHAR(3) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci,
  cs_str   CHAR(3) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_as_cs,
  bin_str  BINARY(3)
);
```

假设表中有以下内容：

```
+--------+--------+---------+
| ci_str | cs_str | bin_str |
+--------+--------+---------+
| AAA    | AAA    | AAA     |
| aaa    | aaa    | aaa     |
| bbb    | bbb    | bbb     |
| BBB    | BBB    | BBB     |
+--------+--------+---------+
```

###### 提示

自 MySQL 8.0.19 起，*mysql* 客户端以十六进制格式打印二进制数据。

```
mysql> `select` `*` `from` `str_val``;`
+--------+--------+------------------+ | ci_str | cs_str | bin_str          |
+--------+--------+------------------+ | AAA    | AAA    | 0x414141         |
| aaa    | aaa    | 0x616161         |
| bbb    | bbb    | 0x626262         |
| BBB    | BBB    | 0x424242         |
+--------+--------+------------------+ 4 rows in set (0.00 sec)
```

要以 ASCII 格式打印值，请使用选项 `--binary-as-hex=0` 启动 *mysql*。

每列包含相同的值，但是不同数据类型的列数据的自然排序顺序会产生三种不同的结果：

+   不区分大小写的排序将`a`和`A`放在一起，将它们排在`b`和`B`之前。然而，对于给定的字母，它并不一定会将一个大小写字母排在另一个字母之前，如下结果所示：

    ```
    mysql> `SELECT ci_str FROM str_val ORDER BY ci_str;`
    +--------+
    | ci_str |
    +--------+
    | AAA    |
    | aaa    |
    | bbb    |
    | BBB    |
    +--------+
    ```

+   大小写敏感的排序将`a`和`A`排在`b`和`B`之前，并将小写字母排在大写字母之前：

    ```
    mysql> `SELECT cs_str FROM str_val ORDER BY cs_str;`
    +--------+
    | cs_str |
    +--------+
    | aaa    |
    | AAA    |
    | bbb    |
    | BBB    |
    +--------+
    ```

+   二进制字符串按数字顺序排序。假设大写字母的数值小于小写字母的数值，二进制排序将产生以下顺序：

    ```
    mysql> `SELECT bin_str FROM str_val ORDER BY bin_str;`
    +---------+
    | bin_str |
    +---------+
    | AAA     |
    | BBB     |
    | aaa     |
    | bbb     |
    +---------+
    ```

    对于具有二进制排序的非二进制字符串列，如果列包含单字节字符（例如`CHAR(3)` `CHARACTER` `SET` `latin1` `COLLATE` `latin1_bin`），则会得到相同的结果。对于多字节字符，二进制排序仍然会产生数字排序，但字符值使用多字节数字。

要改变每列的排序属性，请使用食谱 7.7 中描述的技术来控制字符串比较：

+   要按区分大小写的方式排序不区分大小写的字符串，请使用区分大小写的排序来排序排序值：

    ```
    mysql> `SELECT ci_str FROM str_val`
        -> `ORDER BY ci_str COLLATE utf8mb4_0900_as_cs;`
    +--------+
    | ci_str |
    +--------+
    | aaa    |
    | AAA    |
    | bbb    |
    | BBB    |
    +--------+
    ```

+   要按区分大小写的方式排序区分大小写的字符串，请使用不区分大小写的排序来排序排序值：

    ```
    mysql> `SELECT cs_str FROM str_val`
        -> `ORDER BY cs_str COLLATE utf8mb4_0900_ai_ci;`
    +--------+
    | cs_str |
    +--------+
    | AAA    |
    | aaa    |
    | bbb    |
    | BBB    |
    +--------+
    ```

    或者，使用转换为相同大小写的值进行排序，使大小写无关紧要：

    ```
    mysql> `SELECT cs_str FROM str_val`
        -> `ORDER BY UPPER(cs_str);`
    +--------+
    | cs_str |
    +--------+
    | AAA    |
    | aaa    |
    | bbb    |
    | BBB    |
    +--------+
    ```

+   二进制字符串按照数字字节值排序，因此没有字母大小写的概念。然而，由于不同大小写的字母具有不同的字节值，对二进制字符串的比较实际上是大小写敏感的（即`a`和`A`不相等）。要使用不区分大小写的排序来排序二进制字符串，请将其转换为非二进制字符串并应用适当的排序。例如，要执行不区分大小写的排序，请使用以下语句：

    ```
    mysql> `SELECT bin_str FROM str_val`
        -> `ORDER BY CONVERT(bin_str USING utf8mb4) COLLATE utf8mb4_0900_ai_ci;`
    +---------+
    | bin_str |
    +---------+
    | AAA     |
    | aaa     |
    | bbb     |
    | BBB     |
    +---------+
    ```

    如果字符集默认的排序不区分大小写（如`utf8mb4`），可以省略`COLLATE`子句。

# 9.5 按时间顺序排序

## 问题

您希望按时间顺序对行进行排序。

## 解决方案

使用日期或时间列进行排序。如果值的某些部分对于您想要完成的排序无关紧要，请忽略它们。

## 讨论

许多数据库表包括日期或时间信息，通常需要按时间顺序对结果进行排序。MySQL 知道如何对时间数据类型进行排序，因此没有特殊的技巧来对其进行排序。接下来的几个示例使用包含`DATETIME`列的`mail`表，但相同的排序原则适用于`DATE`、`TIME`和`TIMESTAMP`列。

这里是由`phil`发送的消息：

```
mysql> `SELECT t, srcuser, srchost, dstuser, dsthost, size`
    -> `FROM mail WHERE srcuser = 'phil';`
+---------------------+---------+---------+---------+---------+-------+
| t                   | srcuser | srchost | dstuser | dsthost | size  |
+---------------------+---------+---------+---------+---------+-------+
| 2014-05-12 15:02:49 | phil    | mars    | phil    | saturn  |  1048 |
| 2014-05-14 11:52:17 | phil    | mars    | tricia  | saturn  |  5781 |
| 2014-05-15 08:50:57 | phil    | venus   | phil    | venus   |   978 |
| 2014-05-16 23:04:19 | phil    | venus   | barb    | venus   | 10294 |
| 2014-05-19 12:49:23 | phil    | mars    | tricia  | saturn  |   873 |
+---------------------+---------+---------+---------+---------+-------+
```

要显示最近发送的消息，先使用`ORDER` `BY`和`DESC`进行排序：

```
mysql> `SELECT t, srcuser, srchost, dstuser, dsthost, size`
    -> `FROM mail WHERE srcuser = 'phil' ORDER BY t DESC;`
+---------------------+---------+---------+---------+---------+-------+
| t                   | srcuser | srchost | dstuser | dsthost | size  |
+---------------------+---------+---------+---------+---------+-------+
| 2014-05-19 12:49:23 | phil    | mars    | tricia  | saturn  |   873 |
| 2014-05-16 23:04:19 | phil    | venus   | barb    | venus   | 10294 |
| 2014-05-15 08:50:57 | phil    | venus   | phil    | venus   |   978 |
| 2014-05-14 11:52:17 | phil    | mars    | tricia  | saturn  |  5781 |
| 2014-05-12 15:02:49 | phil    | mars    | phil    | saturn  |  1048 |
+---------------------+---------+---------+---------+---------+-------+
```

有时，时间排序仅使用日期或时间列的一部分。在这种情况下，使用提取所需部分的表达式，并使用该表达式对结果进行排序。以下讨论中给出了一些示例。

### 按照一天中的时间排序

可以以不同的方式对时间进行排序，具体取决于列类型。如果值存储在名为`timecol`的`TIME`列中，只需直接使用`ORDER BY timecol`进行排序。要按天时序排序`DATETIME`或`TIMESTAMP`值，提取时间部分并进行排序。例如，`mail`表包含`DATETIME`值，可以按一天中的时间进行排序如下：

```
mysql> `SELECT t, srcuser, srchost, dstuser, dsthost, size FROM mail ORDER BY TIME(t);`
+---------------------+---------+---------+---------+---------+---------+
| t                   | srcuser | srchost | dstuser | dsthost | size    |
+---------------------+---------+---------+---------+---------+---------+
| 2014-05-15 07:17:48 | gene    | mars    | gene    | saturn  |    3824 |
| 2014-05-15 08:50:57 | phil    | venus   | phil    | venus   |     978 |
| 2014-05-16 09:00:28 | gene    | venus   | barb    | mars    |     613 |
| 2014-05-14 09:31:37 | gene    | venus   | barb    | mars    |    2291 |
| 2014-05-11 10:15:08 | barb    | saturn  | tricia  | mars    |   58274 |
| 2014-05-15 10:25:52 | gene    | mars    | tricia  | saturn  |  998532 |
| 2014-05-14 11:52:17 | phil    | mars    | tricia  | saturn  |    5781 |
| 2014-05-12 12:48:13 | tricia  | mars    | gene    | venus   |  194925 |
…
```

### 按日历日排序

若要按日历顺序排序日期值，请忽略日期的年份部分，仅使用月份和日期按日历年中的位置排序值。假设`occasion`表如下所示，当值按日期排序时：

```
mysql> `SELECT date, description FROM occasion ORDER BY date;`
+------------+-------------------------------------+
| date       | description                         |
+------------+-------------------------------------+
| 1215-06-15 | Signing of the Magna Carta          |
| 1732-02-22 | George Washington's birthday        |
| 1776-07-14 | Bastille Day                        |
| 1789-07-04 | US Independence Day                 |
| 1809-02-12 | Abraham Lincoln's birthday          |
| 1919-06-28 | Signing of the Treaty of Versailles |
| 1944-06-06 | D-Day at Normandy Beaches           |
| 1957-10-04 | Sputnik launch date                 |
| 1989-11-09 | Opening of the Berlin Wall          |
+------------+-------------------------------------+
```

要按日历顺序排列这些项目，请按月份和每月内的日期排序：

```
mysql> `SELECT date, description FROM occasion`
    -> `ORDER BY MONTH(date), DAYOFMONTH(date);`
+------------+-------------------------------------+
| date       | description                         |
+------------+-------------------------------------+
| 1809-02-12 | Abraham Lincoln's birthday          |
| 1732-02-22 | George Washington's birthday        |
| 1944-06-06 | D-Day at Normandy Beaches           |
| 1215-06-15 | Signing of the Magna Carta          |
| 1919-06-28 | Signing of the Treaty of Versailles |
| 1789-07-04 | US Independence Day                 |
| 1776-07-14 | Bastille Day                        |
| 1957-10-04 | Sputnik launch date                 |
| 1989-11-09 | Opening of the Berlin Wall          |
+------------+-------------------------------------+
```

MySQL 具有`DAYOFYEAR()`函数，你可能会认为它在日历日排序中很有用。然而，它可能会为不同的日历日期生成相同的值。例如，闰年的 2 月 29 日和非闰年的 3 月 1 日具有相同的一年中的日值：

```
mysql> `SELECT DAYOFYEAR('1996-02-29'), DAYOFYEAR('1997-03-01');`
+-------------------------+-------------------------+
| DAYOFYEAR('1996-02-29') | DAYOFYEAR('1997-03-01') |
+-------------------------+-------------------------+
|                      60 |                      60 |
+-------------------------+-------------------------+
```

这意味着`DAYOFYEAR()`可以将实际上出现在不同日历日期上的日期分组。

如果表使用单独的年、月和日列表示日期，那么日历排序不需要提取日期部分。直接按相关列进行排序即可。对于大型数据集，使用单独的日期部分列进行排序可能比基于提取`DATE`值片段的排序要快得多。这里的原则是，应设计表格以便轻松提取或按预期频繁使用的值排序。

### 按星期几排序

按星期几排序类似于按日历日排序，只是使用不同的函数来获取相关的排序值。

使用`DAYNAME()`可以获取星期几，但这会按字典顺序而不是按星期几的顺序排序（星期日、星期一、星期二等）。在这里，显示一个值但按另一个值排序的技术很有用（见配方 9.3）。使用`DAYNAME()`显示星期几，但使用`DAYOFWEEK()`按星期几排序，它返回从 1 到 7 的数字值，分别对应星期日到星期六：

```
mysql> `SELECT DAYNAME(date) AS day, date, description`
    -> `FROM occasion`
    -> `ORDER BY DAYOFWEEK(date);`
+----------+------------+-------------------------------------+
| day      | date       | description                         |
+----------+------------+-------------------------------------+
| Sunday   | 1776-07-14 | Bastille Day                        |
| Sunday   | 1809-02-12 | Abraham Lincoln's birthday          |
| Monday   | 1215-06-15 | Signing of the Magna Carta          |
| Tuesday  | 1944-06-06 | D-Day at Normandy Beaches           |
| Thursday | 1989-11-09 | Opening of the Berlin Wall          |
| Friday   | 1957-10-04 | Sputnik launch date                 |
| Friday   | 1732-02-22 | George Washington's birthday        |
| Saturday | 1789-07-04 | US Independence Day                 |
| Saturday | 1919-06-28 | Signing of the Treaty of Versailles |
+----------+------------+-------------------------------------+
```

若要按星期几排序，但将星期一视为一周的第一天，星期日视为最后一天，则使用模运算和`MOD()`函数将星期一映射为 0，星期二映射为 1，……，星期日映射为 6：

```
mysql> `SELECT DAYNAME(date), date, description`
    -> `FROM occasion`
    -> `ORDER BY MOD(DAYOFWEEK(date)+5, 7);`
+---------------+------------+-------------------------------------+
| DAYNAME(date) | date       | description                         |
+---------------+------------+-------------------------------------+
| Monday        | 1215-06-15 | Signing of the Magna Carta          |
| Tuesday       | 1944-06-06 | D-Day at Normandy Beaches           |
| Thursday      | 1989-11-09 | Opening of the Berlin Wall          |
| Friday        | 1957-10-04 | Sputnik launch date                 |
| Friday        | 1732-02-22 | George Washington's birthday        |
| Saturday      | 1789-07-04 | US Independence Day                 |
| Saturday      | 1919-06-28 | Signing of the Treaty of Versailles |
| Sunday        | 1776-07-14 | Bastille Day                        |
| Sunday        | 1809-02-12 | Abraham Lincoln's birthday          |
+---------------+------------+-------------------------------------+
```

表 9-1 显示了`DAYOFWEEK()`表达式，用于在排序中将任何一天作为第一天：

表 9-1\. 使用模运算正确排序一周的日子

| 列出第一天 | DAYOFWEEK()表达式 |
| --- | --- |
| 星期日 | `DAYOFWEEK(date)` |
| 星期一 | `MOD(DAYOFWEEK(date)+5, 7)` |
| 星期二 | `MOD(DAYOFWEEK(date)+4, 7)` |
| 星期三 | `MOD(DAYOFWEEK(date)+3, 7)` |
| 星期四 | `MOD(DAYOFWEEK(date)+2, 7)` |
| 星期五 | `MOD(DAYOFWEEK(date)+1, 7)` |
| 星期六 | `MOD(DAYOFWEEK(date)+0, 7)` |

您还可以使用`WEEKDAY()`进行按星期几排序，尽管它返回不同的值集（星期一到星期日分别为 0 到 6）。

# 9.6 按列值的子字符串排序

## 问题

您希望使用每个值的一个或多个子字符串进行排序。

## 解决方案

提取您想要的部分并将它们单独排序。

## 讨论

这是排序表达式值的特定应用（见 Recipe 9.2）。要仅使用列值的特定部分对行进行排序，请提取您需要的子字符串，并在`ORDER BY`子句中使用它。如果子字符串在列内具有固定的位置和长度，则这样做最为简单。对于位置或长度可变的子字符串，如果您有可靠的识别方法，仍然可以用它们进行排序。接下来的几个示例展示如何使用子字符串提取来生成专门的排序顺序。

# 9.7 按固定长度子字符串排序

## 问题

您希望使用在列内给定位置出现的部分来进行排序。

## 解决方案

使用`LEFT()`、`SUBSTRING()`（`MID()`）或`RIGHT()`拉出您需要的部分，并对它们进行排序。

## 讨论

假设`housewares`表格记录家居用品家具，每个都由 10 个字符的 ID 值标识，包括三个子部分：三个字符的类别缩写（例如用于<dining room>餐厅的`DIN`或<kitchen>厨房的`KIT`），五位数的序列号，以及指示零件制造地的两个字符的国家代码：

```
mysql> `SELECT * FROM housewares;`
+------------+------------------+
| id         | description      |
+------------+------------------+
| DIN40672US | dining table     |
| KIT00372UK | garbage disposal |
| KIT01729JP | microwave oven   |
| BED00038SG | bedside lamp     |
| BTH00485US | shower stall     |
| BTH00415JP | lavatory         |
+------------+------------------+
```

这并不一定是存储复杂 ID 值的好方法，稍后我们将考虑如何使用单独的列来表示它们。现在假设这些值必须按照所示存储。

要根据`id`值对来自该表的行进行排序，请使用整个列值：

```
mysql> `SELECT * FROM housewares ORDER BY id;`
+------------+------------------+
| id         | description      |
+------------+------------------+
| BED00038SG | bedside lamp     |
| BTH00415JP | lavatory         |
| BTH00485US | shower stall     |
| DIN40672US | dining table     |
| KIT00372UK | garbage disposal |
| KIT01729JP | microwave oven   |
+------------+------------------+
```

但您可能还需要按照这三个子部分的任意一个进行排序（例如，按制造国家排序）。对于这种操作，诸如`LEFT()`、`MID()`和`RIGHT()`的函数非常有用，用于提取`id`值的组件：

```
mysql> `SELECT id,`
    -> `LEFT(id,3) AS category,`
    -> `MID(id,4,5) AS serial,`
    -> `RIGHT(id,2) AS country`
    -> `FROM housewares;`
+------------+----------+--------+---------+
| id         | category | serial | country |
+------------+----------+--------+---------+
| DIN40672US | DIN      | 40672  | US      |
| KIT00372UK | KIT      | 00372  | UK      |
| KIT01729JP | KIT      | 01729  | JP      |
| BED00038SG | BED      | 00038  | SG      |
| BTH00485US | BTH      | 00485  | US      |
| BTH00415JP | BTH      | 00415  | JP      |
+------------+----------+--------+---------+
```

###### 提示

函数`MID()`是函数`SUBSTRING()`的同义词。

这些`id`值的固定长度子字符串可以单独或组合用于排序。例如，要按产品类别排序，请在`ORDER BY`子句中提取和使用类别：

```
mysql> `SELECT * FROM housewares ORDER BY LEFT(id,3);`
+------------+------------------+
| id         | description      |
+------------+------------------+
| BED00038SG | bedside lamp     |
| BTH00485US | shower stall     |
| BTH00415JP | lavatory         |
| DIN40672US | dining table     |
| KIT00372UK | garbage disposal |
| KIT01729JP | microwave oven   |
+------------+------------------+
```

要按产品序列号排序，请使用`MID()`从`id`值中提取中间的五个字符，从第四个字符开始：

```
mysql> `SELECT * FROM housewares ORDER BY MID(id,4,5);`
+------------+------------------+
| id         | description      |
+------------+------------------+
| BED00038SG | bedside lamp     |
| KIT00372UK | garbage disposal |
| BTH00415JP | lavatory         |
| BTH00485US | shower stall     |
| KIT01729JP | microwave oven   |
| DIN40672US | dining table     |
+------------+------------------+
```

这似乎是数字排序，但实际上是字符串排序，因为`MID()`返回字符串。在这种情况下，词汇和数字的排序顺序相同，因为<numbers>具有前导零，使它们长度相同。

要按国家代码排序，请使用`id`值的最后两个字符（`ORDER BY RIGHT(id,2)`）。

您还可以按组合的子字符串进行排序；例如，按国家代码和国内序列号排序：

```
mysql> `SELECT * FROM housewares ORDER BY RIGHT(id,2), MID(id,4,5);`
+------------+------------------+
| id         | description      |
+------------+------------------+
| BTH00415JP | lavatory         |
| KIT01729JP | microwave oven   |
| BED00038SG | bedside lamp     |
| KIT00372UK | garbage disposal |
| BTH00485US | shower stall     |
| DIN40672US | dining table     |
+------------+------------------+
```

刚刚显示的`ORDER BY`子句足以按`id`值的子串排序，但如果此类操作在表上常见，则可能值得以不同方式表示家居设备 ID；例如，使用单独的列表示。此表`housewares2`与`housewares`类似，但使用从`id`列生成的`category`、`serial`和`country`列：

```
CREATE TABLE `housewares2` (
  `id` varchar(20) NOT NULL,
  `category` varchar(3) GENERATED ALWAYS AS (left(`id`,3)) STORED,
  `serial` char(5) GENERATED ALWAYS AS (substr(`id`,4,5)) STORED,
  `country` varchar(2) GENERATED ALWAYS AS (right(`id`,2)) STORED,
  `description` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
);
```

在此示例中，我们使用了基于表达式生成的生成列，在列创建时定义。

将 ID 值拆分为单独的部分后，排序操作更容易指定；直接引用各个列而不是提取原始`id`列的子串。您还可以通过在这些列上添加索引，使对`serial`和`country`列进行排序的操作更加高效。

```
mysql> `SELECT category, serial, country, id`
    -> `FROM housewares2 ORDER BY category, country, serial;`
+----------+--------+---------+------------+
| category | serial | country | id         |
+----------+--------+---------+------------+
| BED      |  00038 | SG      | BED00038SG |
| BTH      |  00415 | JP      | BTH00415JP |
| BTH      |  00485 | US      | BTH00485US |
| DIN      |  40672 | US      | DIN40672US |
| KIT      |  01729 | JP      | KIT01729JP |
| KIT      |  00372 | UK      | KIT00372UK |
+----------+--------+---------+------------+
```

此示例说明了一个重要原则：您可能会以某种方式考虑值（`id`值作为单个字符串），但不一定需要在数据库中以此方式表示。如果另一种表示方式（单独列）更有效或更易于处理，那么即使必须重新格式化底层列使其看起来符合人们的期望，也可能值得使用。

# 9.8 变长子串排序

## 问题

您想使用不在列中给定位置出现的部分进行排序。

## 解决方案

确定如何识别您需要的部分，以便可以提取它们。

## 讨论

如果用于排序的子串长度不同，您需要一种可靠的方法来仅提取您想要的部分。为了了解其工作原理，让我们创建一个`housewares3`表，其类似于配方 9.7 中使用的`housewares`表，但`id`值的序号部分没有前导零：

```
mysql> `SELECT * FROM housewares3;`
+------------+------------------+
| id         | description      |
+------------+------------------+
| DIN40672US | dining table     |
| KIT372UK   | garbage disposal |
| KIT1729JP  | microwave oven   |
| BED38SG    | bedside lamp     |
| BTH485US   | shower stall     |
| BTH415JP   | lavatory         |
+------------+------------------+
```

可以使用`LEFT()`和`RIGHT()`提取和排序`id`值的类别和国家部分，就像`housewares`表一样。但是现在值的数值部分长度不同，不能使用简单的`MID()`调用来提取和排序。而是使用其完整版本`SUBSTRING()`跳过前三个字符。从从第四个字符开始的余下部分（第一个数字），取除最右侧两列之外的所有内容。一个方法如下：

```
mysql> `SELECT id, LEFT(SUBSTRING(id,4),CHAR_LENGTH(SUBSTRING(id,4)-2))`
    -> `FROM housewares3;`
+------------+------------------------------------------------------+
| id         | LEFT(SUBSTRING(id,4),CHAR_LENGTH(SUBSTRING(id,4)-2)) |
+------------+------------------------------------------------------+
| DIN40672US | 40672                                                |
| KIT372UK   | 372                                                  |
| KIT1729JP  | 1729                                                 |
| BED38SG    | 38                                                   |
| BTH485US   | 485                                                  |
| BTH415JP   | 415                                                  |
+------------+------------------------------------------------------+
```

但这比必要的更复杂。`SUBSTRING()`函数接受一个可选的第三个参数，指定所需的结果长度，我们知道中间部分的长度等于字符串长度减去五（开头三个字符和结尾两个字符）。以下查询演示了如何通过开始于 ID 并剥离最右侧后缀来获取数值中间部分：

```
mysql> `SELECT id, SUBSTRING(id,4), SUBSTRING(id,4,CHAR_LENGTH(id)-5)`
    -> `FROM housewares3;`
+------------+-----------------+-----------------------------------+
| id         | SUBSTRING(id,4) | SUBSTRING(id,4,CHAR_LENGTH(id)-5) |
+------------+-----------------+-----------------------------------+
| DIN40672US | 40672US         | 40672                             |
| KIT372UK   | 372UK           | 372                               |
| KIT1729JP  | 1729JP          | 1729                              |
| BED38SG    | 38SG            | 38                                |
| BTH485US   | 485US           | 485                               |
| BTH415JP   | 415JP           | 415                               |
+------------+-----------------+-----------------------------------+
```

不幸的是，尽管最终表达式正确地从 ID 中提取了数值部分，但结果值是字符串。因此，它们按字典顺序而不是数值顺序排序：

```
mysql> `SELECT * FROM housewares3`
    -> `ORDER BY SUBSTRING(id,4,CHAR_LENGTH(id)-5);`
+------------+------------------+
| id         | description      |
+------------+------------------+
| KIT1729JP  | microwave oven   |
| KIT372UK   | garbage disposal |
| BED38SG    | bedside lamp     |
| DIN40672US | dining table     |
| BTH415JP   | lavatory         |
| BTH485US   | shower stall     |
+------------+------------------+
```

如何处理？一种方法是添加零，告诉 MySQL 执行字符串到数字的转换，这样可以对序列号值进行数字排序：

```
mysql> `SELECT * FROM housewares3`
    -> `ORDER BY SUBSTRING(id,4,CHAR_LENGTH(id)-5)+0;`
+------------+------------------+
| id         | description      |
+------------+------------------+
| BED38SG    | bedside lamp     |
| KIT372UK   | garbage disposal |
| BTH415JP   | lavatory         |
| BTH485US   | shower stall     |
| KIT1729JP  | microwave oven   |
| DIN40672US | dining table     |
+------------+------------------+
```

在前面的示例中，提取可变长度子字符串的能力基于`id`值中中间字符与两端字符的不同类型（即数字与非数字）。在其他情况下，您可能可以使用分隔符字符来拆分列值。对于下一个示例，请假设一个具有以下`id`值的`housewares4`表：

```
mysql> `SELECT * FROM housewares4;`
+---------------+------------------+
| id            | description      |
+---------------+------------------+
| 13-478-92-2   | dining table     |
| 873-48-649-63 | garbage disposal |
| 8-4-2-1       | microwave oven   |
| 97-681-37-66  | bedside lamp     |
| 27-48-534-2   | shower stall     |
| 5764-56-89-72 | lavatory         |
+---------------+------------------+
```

要从这些值中提取段，使用`SUBSTRING_INDEX(`*`str`*`,`*`c`*`,`*`n`*`)`。它搜索字符串 *`str`* 中给定字符 *`c`* 的第 *`n`* 次出现，并返回该字符左边的所有内容。例如，以下调用返回`13-478`：

```
SUBSTRING_INDEX('13-478-92-2','-',2)
```

如果 *`n`* 为负数，则从右边开始搜索 *`c`* 并返回最右边的字符串。这个调用返回`478-92-2`：

```
SUBSTRING_INDEX('13-478-92-2','-',-3)
```

通过将`SUBSTRING_INDEX()`调用与正负索引结合使用，可以从每个`id`值中提取连续的片段：提取值的第一个 *`n`* 段并去掉最右边的一个。通过将 *`n`* 从 1 变化到 4，我们从左到右获取连续的片段：

```
SUBSTRING_INDEX(SUBSTRING_INDEX(id,'-',1),'-',-1)
SUBSTRING_INDEX(SUBSTRING_INDEX(id,'-',2),'-',-1)
SUBSTRING_INDEX(SUBSTRING_INDEX(id,'-',3),'-',-1)
SUBSTRING_INDEX(SUBSTRING_INDEX(id,'-',4),'-',-1)
```

那些表达式中的第一个可以进行优化，因为内部的`SUBSTRING_INDEX()`调用返回单段字符串，并且单独就足以返回最左边的`id`段：

```
SUBSTRING_INDEX(id,'-',1)
```

获取子字符串的另一种方法是提取值的最右边 *`n`* 段并去掉第一个。在这里，我们将 *`n`* 从 -4 变化到 -1：

```
SUBSTRING_INDEX(SUBSTRING_INDEX(id,'-',-4),'-',1)
SUBSTRING_INDEX(SUBSTRING_INDEX(id,'-',-3),'-',1)
SUBSTRING_INDEX(SUBSTRING_INDEX(id,'-',-2),'-',1)
SUBSTRING_INDEX(SUBSTRING_INDEX(id,'-',-1),'-',1)
```

再次，可以进行优化。对于第四个表达式，内部`SUBSTRING_INDEX()`调用足以返回最终子字符串：

```
SUBSTRING_INDEX(id,'-',-1)
```

这   这些表达式可能很难阅读和理解，尝试几个实验看看它们的工作方式可能会有所帮助。以下是一个示例，展示如何从`id`值中获取第二和第四段：

```
mysql> `SELECT`
    -> `id,`
    -> `SUBSTRING_INDEX(SUBSTRING_INDEX(id,'-',2),'-',-1) AS segment2,`
    -> `SUBSTRING_INDEX(SUBSTRING_INDEX(id,'-',4),'-',-1) AS segment4`
    -> `FROM housewares4;`
+---------------+----------+----------+
| id            | segment2 | segment4 |
+---------------+----------+----------+
| 13-478-92-2   | 478      | 2        |
| 873-48-649-63 | 48       | 63       |
| 8-4-2-1       | 4        | 1        |
| 97-681-37-66  | 681      | 66       |
| 27-48-534-2   | 48       | 2        |
| 5764-56-89-72 | 56       | 72       |
+---------------+----------+----------+
```

要使用子字符串进行排序，请在`ORDER` `BY`子句中使用适当的表达式。（如果要进行数字排序而不是词法排序，请记得通过添加零来强制进行字符串到数字的转换。）以下两个查询基于第二个`id`段对结果进行排序。第一个是按字母顺序排序，第二个是按数字顺序排序：

```
mysql> `SELECT * FROM housewares4`
    -> `ORDER BY SUBSTRING_INDEX(SUBSTRING_INDEX(id,'-',2),'-',-1);`
+---------------+------------------+
| id            | description      |
+---------------+------------------+
| 8-4-2-1       | microwave oven   |
| 13-478-92-2   | dining table     |
| 873-48-649-63 | garbage disposal |
| 27-48-534-2   | shower stall     |
| 5764-56-89-72 | lavatory         |
| 97-681-37-66  | bedside lamp     |
+---------------+------------------+
mysql> `SELECT * FROM housewares4`
    -> `ORDER BY SUBSTRING_INDEX(SUBSTRING_INDEX(id,'-',2),'-',-1)+0;`
+---------------+------------------+
| id            | description      |
+---------------+------------------+
| 8-4-2-1       | microwave oven   |
| 873-48-649-63 | garbage disposal |
| 27-48-534-2   | shower stall     |
| 5764-56-89-72 | lavatory         |
| 13-478-92-2   | dining table     |
| 97-681-37-66  | bedside lamp     |
+---------------+------------------+
```

这里的子字符串提取表达式很混乱，但至少我们应用表达式的列值具有一致的段数。对具有不同段数的值进行排序，这项工作可能更加困难。第 9.9 节展示了一个示例，说明了其中的原因。

# 9.9 按域名顺序对主机名排序

## 问题

你想按域名顺序对主机名进行排序，右边的名称部分比左边的部分更重要。

## 解决方案

拆分名称，并从右到左对这些部分进行排序。

## 讨论

主机名是字符串，因此它们的自然排序顺序是词法顺序。但通常希望按域顺序对主机名进行排序，其中主机名值的右侧段比左侧段更重要。假设一个`hostname`表包含以下名称：

```
mysql> `SELECT name FROM hostname ORDER BY name;`
+--------------------+
| name               |
+--------------------+
| dbi.perl.org       |
| jakarta.apache.org |
| lists.mysql.com    |
| mysql.com          |
| svn.php.net        |
| www.kitebird.com   |
+--------------------+
```

前面的查询展示了`name`值的自然词法排序顺序。这与域顺序不同，如[Table 9-2](https://example.org/table_9_2)所示：

表 9-2\. 词法顺序与域顺序

| 词法顺序 | 域顺序 |
| --- | --- |
| `dbi.perl.org` | `www.kitebird.com` |
| `jakarta.apache.org` | `mysql.com` |
| `lists.mysql.com` | `lists.mysql.com` |
| `mysql.com` | `svn.php.net` |
| `svn.php.net` | `jakarta.apache.org` |
| `www.kitebird.com` | `dbi.perl.org` |

生产域排序的输出是一个子字符串排序问题，需要提取每个名称段，以便按照从右到左的顺序进行排序。如果您的值包含不同数量的段落，比如我们的示例主机名，这还会带来额外的复杂性（大多数有三段，但`mysql.com`只有两段）。

要提取主机名的各部分，请使用`SUBSTRING_INDEX()`，类似于[Recipe 9.8](https://example.org/recipe_9_8)中所描述的方式。主机名的值最多有三个段，可以从左到右提取这些部分，就像这样：

```
SUBSTRING_INDEX(SUBSTRING_INDEX(name,'.',-3),'.',1)
SUBSTRING_INDEX(SUBSTRING_INDEX(name,'.',-2),'.',1)
SUBSTRING_INDEX(name,'.',-1)
```

只要所有主机名都有三个组件，这些表达式就能正常工作。但是，如果名称少于三个组件，则不会得到正确的结果，如下面的查询所示：

```
mysql> `SELECT name,`
    -> `SUBSTRING_INDEX(SUBSTRING_INDEX(name,'.',-3),'.',1) AS leftmost,`
    -> `SUBSTRING_INDEX(SUBSTRING_INDEX(name,'.',-2),'.',1) AS middle,`
    -> `SUBSTRING_INDEX(name,'.',-1) AS rightmost`
    -> `FROM hostname;`
+--------------------+----------+----------+-----------+
| name               | leftmost | middle   | rightmost |
+--------------------+----------+----------+-----------+
| svn.php.net        | svn      | php      | net       |
| dbi.perl.org       | dbi      | perl     | org       |
| lists.mysql.com    | lists    | mysql    | com       |
| mysql.com          | mysql    | mysql    | com       |
| jakarta.apache.org | jakarta  | apache   | org       |
| www.kitebird.com   | www      | kitebird | com       |
+--------------------+----------+----------+-----------+
```

注意`mysql.com`行的输出；它在`leftmost`列的值是`mysql`，但应该是空字符串。段落提取表达式通过删除右侧的*n*个段落，然后返回结果的左侧段落来工作。对于`mysql.com`的问题在于，如果段落数量不足，表达式只返回其中有的最左侧段落。要解决此问题，请在主机名值的开头添加足够数量的句点，以保证它们具有必需的段落数量：

```
mysql> `SELECT name,`
    -> `SUBSTRING_INDEX(SUBSTRING_INDEX(CONCAT('..',name),'.',-3),'.',1)`
    -> `AS leftmost,`
    -> `SUBSTRING_INDEX(SUBSTRING_INDEX(CONCAT('.',name),'.',-2),'.',1)`
    -> `AS middle,`
    -> `SUBSTRING_INDEX(name,'.',-1) AS rightmost`
    -> `FROM hostname;`
+--------------------+----------+----------+-----------+
| name               | leftmost | middle   | rightmost |
+--------------------+----------+----------+-----------+
| svn.php.net        | svn      | php      | net       |
| dbi.perl.org       | dbi      | perl     | org       |
| lists.mysql.com    | lists    | mysql    | com       |
| mysql.com          |          | mysql    | com       |
| jakarta.apache.org | jakarta  | apache   | org       |
| www.kitebird.com   | www      | kitebird | com       |
+--------------------+----------+----------+-----------+
```

这看起来相当丑陋。但是这些表达式确实可以提取所需的子字符串，以正确地按从右到左的方式排序主机名值：

```
mysql> `SELECT name FROM hostname`
    -> `ORDER BY`
    -> `SUBSTRING_INDEX(name,'.',-1),`
    -> `SUBSTRING_INDEX(SUBSTRING_INDEX(CONCAT('.',name),'.',-2),'.',1),`
    -> `SUBSTRING_INDEX(SUBSTRING_INDEX(CONCAT('..',name),'.',-3),'.',1);`
+--------------------+
| name               |
+--------------------+
| www.kitebird.com   |
| mysql.com          |
| lists.mysql.com    |
| svn.php.net        |
| jakarta.apache.org |
| dbi.perl.org       |
+--------------------+
```

如果您的主机名最多有四个段而不是三个，那么在`ORDER BY`子句中添加另一个`SUBSTRING_INDEX()`表达式，以在主机名值的开头添加三个句点。

# 9.10 按数字顺序排序点分 IP 值

## 问题

您希望按照表示 IP 号码的字符串进行数字顺序排序。

## 解决方案

拆分字符串并按数字顺序排序各部分。或者只需使用`INET_ATON()`。或考虑将值存储为数字。

## 讨论

如果表中包含以点分十进制表示的 IP 数字字符串（`192.168.1.10`），则它们按词法顺序而不是数值顺序排序。为了产生数值排序，可以将它们作为四部分值进行排序，每部分按数值顺序排序。或者更高效地，将 IP 数字表示为 32 位无符号整数，这需要更少的空间并且可以通过简单的数值排序来排序。本节展示了两种方法。

要对字符串形式的点分十进制 IP 数字进行排序，可以使用类似于主机名排序的技术（见食谱 9.9），但有以下几点不同：

+   点分四段始终存在。在提取子字符串之前不需要向值添加点。

+   点分四段按从左到右排序。在 `ORDER BY` 子句中使用的子字符串顺序与主机名排序相反。

+   点分十进制值的各段都是数字。对每个子字符串添加零以强制执行数值而不是词法排序。

假设 `hostip` 表具有一个字符串值 `ip` 列，其中包含 IP 数字：

```
mysql> `SELECT ip FROM hostip ORDER BY ip;`
+-----------------+
| ip              |
+-----------------+
| 127.0.0.1       |
| 192.168.0.10    |
| 192.168.0.2     |
| 192.168.1.10    |
| 192.168.1.2     |
| 21.0.0.1        |
| 255.255.255.255 |
+-----------------+
```

前述查询生成的输出按词法顺序排序。要按 `ip` 值进行数值排序，提取每个段并加零以将其转换为数字，如下所示：

```
mysql> `SELECT ip FROM hostip`
    -> `ORDER BY`
    -> `SUBSTRING_INDEX(ip,'.',1)+0,`
    -> `SUBSTRING_INDEX(SUBSTRING_INDEX(ip,'.',-3),'.',1)+0,`
    -> `SUBSTRING_INDEX(SUBSTRING_INDEX(ip,'.',-2),'.',1)+0,`
    -> `SUBSTRING_INDEX(ip,'.',-1)+0;`
+-----------------+
| ip              |
+-----------------+
| 21.0.0.1        |
| 127.0.0.1       |
| 192.168.0.2     |
| 192.168.0.10    |
| 192.168.1.2     |
| 192.168.1.10    |
| 255.255.255.255 |
+-----------------+
```

然而，尽管 `ORDER BY` 子句生成了正确的结果，但它很复杂。更简单的解决方案是使用 `INET_ATON()` 函数将网络地址从字符串形式转换为其基础数值，然后对这些数值进行排序：

```
mysql> `SELECT ip FROM hostip ORDER BY INET_ATON(ip);`
+-----------------+
| ip              |
+-----------------+
| 21.0.0.1        |
| 127.0.0.1       |
| 192.168.0.2     |
| 192.168.0.10    |
| 192.168.1.2     |
| 192.168.1.10    |
| 255.255.255.255 |
+-----------------+
```

如果您试图通过简单地将零添加到 `ip` 值并在结果上使用 `ORDER BY` 进行排序，考虑一下该类型的字符串到数字转换实际产生的值：

```
mysql> `SELECT ip, ip+0 FROM hostip;`
+-----------------+---------+
| ip              | ip+0    |
+-----------------+---------+
| 127.0.0.1       |     127 |
| 192.168.0.2     | 192.168 |
| 192.168.0.10    | 192.168 |
| 192.168.1.2     | 192.168 |
| 192.168.1.10    | 192.168 |
| 255.255.255.255 | 255.255 |
| 21.0.0.1        |      21 |
+-----------------+---------+
7 rows in set, 7 warnings (0.00 sec)
```

转换仅保留每个值的尽可能多的部分以解释为有效数字（因此会有警告）。其余部分因排序目的不可用，即使对于正确排序也是如此。

在 `ORDER BY` 子句中使用 `INET_ATON()` 比六个 `SUBSTRING_INDEX()` 调用更高效。此外，如果将 IP 地址存储为数字而不是字符串，则在排序时可以完全避免进行任何转换。这样做还有其他好处：数值 IP 地址有 32 位，因此可以使用 4 字节的 `INT UNSIGNED` 列来存储它们，这比字符串形式需要更少的存储空间。而且，如果对该列创建索引，查询优化器可能能够在某些查询中使用该索引。对于需要以点分十进制表示形式显示数值 IP 值的情况，可以使用 `INET_NTOA()` 函数进行转换。

# 9.11 将浮点值放在排序顺序的开头或结尾

## 问题

您希望按列的通常方式排序，但某些值应出现在排序顺序的开头或结尾。例如，您想按词法顺序排序列表，但希望某些高优先级值无论出现在正常排序顺序的何处都应首先显示。

## 解决方案

向 `ORDER BY` 子句添加一个初始排序列，以放置您希望它们出现的那些值。其余的排序列对其他值有正常效果。

## 讨论

要正常排序结果集，*除非*您希望特定值出现在首位，请创建一个额外的排序列，对于这些值为 `0`，对于其他一切为 `1`。这样可以使这些值浮动到升序排序顺序的最前面。若要将这些值放在末尾，请使用降序排序或者为希望在列表末尾的行存储 `1`，对于其他行存储 `0`。

假设某列包含 `NULL` 值：

```
mysql> `SELECT val FROM t;`
+------+
| val  |
+------+
|    3 |
|  100 |
| NULL |
| NULL |
|    9 |
+------+
```

通常，对于升序排序，`NULL` 值会排在最前面：

```
mysql> `SELECT val FROM t ORDER BY val;`
+------+
| val  |
+------+
| NULL |
| NULL |
|    3 |
|    9 |
|  100 |
+------+
```

要将它们放在末尾而不改变其他值的顺序，请引入额外的 `ORDER BY` 列，将 `NULL` 值映射到高于非 `NULL` 值的值：

```
mysql> `SELECT val FROM t ORDER BY IF(val IS NULL,1,0), val;`
+------+
| val  |
+------+
|    3 |
|    9 |
|  100 |
| NULL |
| NULL |
+------+
```

`IF()` 表达式会为排序创建一个新列，该列作为主要排序值使用。

对于降序排序，`NULL` 值会排在最后面。若要将它们放在最前面，请使用相同的技术，但反转 `IF()` 函数的第二和第三参数的顺序，以将 `NULL` 值映射到低于非 `NULL` 值的值：

```
IF(val IS NULL,0,1)
```

同样的技术也适用于浮动除 `NULL` 之外的值到排序顺序的两端。假设您希望按发件人/收件人顺序排序 `mail` 表中的消息，但希望将某个发件人的消息放在首位。在实际世界中，最感兴趣的发件人可能是 `postmaster` 或 `root`。这些名称在表中不存在，因此我们将使用 `phil` 作为感兴趣的名称：

```
mysql> `SELECT t, srcuser, dstuser, size`
    -> `FROM mail`
    -> `ORDER BY IF(srcuser='phil',0,1), srcuser, dstuser;`
+---------------------+---------+---------+---------+
| t                   | srcuser | dstuser | size    |
+---------------------+---------+---------+---------+
| 2014-05-16 23:04:19 | phil    | barb    |   10294 |
| 2014-05-12 15:02:49 | phil    | phil    |    1048 |
| 2014-05-15 08:50:57 | phil    | phil    |     978 |
| 2014-05-14 11:52:17 | phil    | tricia  |    5781 |
| 2014-05-19 12:49:23 | phil    | tricia  |     873 |
| 2014-05-14 14:42:21 | barb    | barb    |   98151 |
| 2014-05-11 10:15:08 | barb    | tricia  |   58274 |
| 2014-05-12 18:59:18 | barb    | tricia  |     271 |
| 2014-05-14 09:31:37 | gene    | barb    |    2291 |
| 2014-05-16 09:00:28 | gene    | barb    |     613 |
| 2014-05-15 17:35:31 | gene    | gene    |    3856 |
| 2014-05-15 07:17:48 | gene    | gene    |    3824 |
| 2014-05-19 22:21:51 | gene    | gene    |   23992 |
| 2014-05-15 10:25:52 | gene    | tricia  |  998532 |
| 2014-05-12 12:48:13 | tricia  | gene    |  194925 |
| 2014-05-14 17:03:01 | tricia  | phil    | 2394482 |
+---------------------+---------+---------+---------+
```

额外排序列的值对于 `srcuser` 值为 `phil` 的行为 `0`，对于所有其他行为 `1`。通过将其作为最重要的排序列，由 `phil` 发送的消息的行会浮动到输出的顶部。（若要将它们放在底部，可以使用 `DESC` 反向排序该列，或反转 `IF()` 函数的第二和第三参数的顺序。）

您还可以根据特定条件使用此技术，而不仅仅是特定值。要先放置那些发送消息给自己的行，请执行以下操作：

```
mysql> `SELECT t, srcuser, dstuser, size`
    -> `FROM mail`
    -> `ORDER BY IF(srcuser=dstuser,0,1), srcuser, dstuser;`
+---------------------+---------+---------+---------+
| t                   | srcuser | dstuser | size    |
+---------------------+---------+---------+---------+
| 2014-05-14 14:42:21 | barb    | barb    |   98151 |
| 2014-05-19 22:21:51 | gene    | gene    |   23992 |
| 2014-05-15 17:35:31 | gene    | gene    |    3856 |
| 2014-05-15 07:17:48 | gene    | gene    |    3824 |
| 2014-05-12 15:02:49 | phil    | phil    |    1048 |
...
```

如果您对表的内容有很好的了解，有时可以省略额外的排序列。例如，在 `mail` 表中，`srcuser` 永远不会是 `NULL`，因此可以将前一个查询重写如下，以在 `ORDER BY` 子句中使用少一列（这依赖于 `NULL` 值在所有非 `NULL` 值之前排序的属性）：

```
SELECT t, srcuser, dstuser, size
FROM mail
ORDER BY IF(srcuser=dstuser,NULL,srcuser), dstuser;
```

# 9.12 定义自定义排序顺序

## 问题

您想按非标准顺序对值进行排序。

## 解决方案

使用 `FIELD()` 将列值映射到一个序列，以便按照所需顺序排列这些值。

## 讨论

Recipe 9.11 展示了如何使一组特定行浮动到排序顺序的开头。要对列中的所有值施加特定顺序，请使用 `FIELD()` 函数将它们映射到一组数值，并使用这些数字进行排序。（当列包含少量不同的值时效果最佳。）

`FIELD()` 调用以下比较 *`value`* 与 *`str1`*, *`str2`*, *`str3`*, 和 *`str4`*，并根据 *`value`* 等于哪个值返回 1, 2, 3, 或 4。

```
FIELD(*`value`*,*`str1`*,*`str2`*,*`str3`*,*`str4`*)
```

如果 *`value`* 是 `NULL` 或没有匹配的值，`FIELD()` 返回 0。

可以使用 `FIELD()` 将任意一组值按任意顺序排序。例如，要按 Henry、Suzi 和 Ben 的顺序显示 `driver_log` 行，请执行以下操作：

```
mysql> `SELECT * FROM driver_log`
    -> `ORDER BY FIELD(name,'Henry','Suzi','Ben');`
+--------+-------+------------+-------+
| rec_id | name  | trav_date  | miles |
+--------+-------+------------+-------+
|     10 | Henry | 2014-07-30 |   203 |
|      8 | Henry | 2014-08-01 |   197 |
|      6 | Henry | 2014-07-26 |   115 |
|      4 | Henry | 2014-07-27 |    96 |
|      3 | Henry | 2014-07-29 |   300 |
|      7 | Suzi  | 2014-08-02 |   502 |
|      2 | Suzi  | 2014-07-29 |   391 |
|      5 | Ben   | 2014-07-29 |   131 |
|      9 | Ben   | 2014-08-02 |    79 |
|      1 | Ben   | 2014-07-30 |   152 |
+--------+-------+------------+-------+
```

# 9.13 排序 ENUM 值

## 问题

`ENUM` 值与其他字符串列不同，你希望它们按照预期顺序检索结果。

## 解决方案

研究 `ENUM` 如何存储数据，并利用这些属性为你所用。例如，可以定义自己的字符串排序顺序，存储在 `ENUM` 列中。

## 讨论

`ENUM` 是一种字符串数据类型，但 `ENUM` 值实际上是按照它们在表定义中列出的顺序存储的数值。这些数值影响枚举排序的方式，这可能非常有用。假设名为 `weekday` 的表包含一个名为 `day` 的枚举列，其成员为工作日名称：

```
CREATE TABLE weekday
(
  day ENUM('Sunday','Monday','Tuesday','Wednesday',
           'Thursday','Friday','Saturday')
);
```

在内部，MySQL 在该定义中定义枚举值 `Sunday` 到 `Saturday` 具有从 1 到 7 的数字值。要自行验证，请使用刚才展示的定义创建表，并为每周的每一天插入一行。为了使插入顺序与排序顺序不同（以便查看排序效果），以随机顺序添加这些天：

```
mysql> `INSERT INTO weekday (day) VALUES('Monday'),('Friday'),`
    -> `('Tuesday'), ('Sunday'), ('Thursday'), ('Saturday'), ('Wednesday');`
```

然后选择这些值，既作为字符串，也作为内部数值（使用 `+0` 强制将字符串转换为数字）：

```
mysql> `SELECT day, day+0 FROM weekday;`
+-----------+-------+
| day       | day+0 |
+-----------+-------+
| Monday    |     2 |
| Friday    |     6 |
| Tuesday   |     3 |
| Sunday    |     1 |
| Thursday  |     5 |
| Saturday  |     7 |
| Wednesday |     4 |
+-----------+-------+
```

注意，由于查询不包括 `ORDER BY` 子句，行以未排序顺序返回。如果添加一个 `ORDER BY day` 子句，就会明显看出 MySQL 使用内部数字值进行排序：

```
mysql> `SELECT day, day+0 FROM weekday ORDER BY day;`
+-----------+-------+
| day       | day+0 |
+-----------+-------+
| Sunday    |     1 |
| Monday    |     2 |
| Tuesday   |     3 |
| Wednesday |     4 |
| Thursday  |     5 |
| Friday    |     6 |
| Saturday  |     7 |
+-----------+-------+
```

当你希望按字典顺序排序 `ENUM` 值时怎么办？使用 `CAST()` 函数强制它们在排序时被视为字符串：

```
mysql> `SELECT day, day+0 FROM weekday ORDER BY CAST(day AS CHAR);`
+-----------+-------+
| day       | day+0 |
+-----------+-------+
| Friday    |     6 |
| Monday    |     2 |
| Saturday  |     7 |
| Sunday    |     1 |
| Thursday  |     5 |
| Tuesday   |     3 |
| Wednesday |     4 |
+-----------+-------+
```

如果你总是（或几乎总是）按特定非字典顺序排序非枚举列，请考虑将数据类型更改为 `ENUM`，其值按所需的排序顺序列出。要了解其工作原理，请创建一个包含字符串列的 `color` 表，并填充一些示例行：

```
mysql> `CREATE TABLE color (name CHAR(10), PRIMARY KEY(name));`
mysql> `INSERT INTO color (name) VALUES ('blue'),('green'),`
    -> `('indigo'),('orange'),('red'),('violet'),('yellow');`
```

此时按 `name` 列排序会产生字典顺序，因为该列包含 `CHAR` 值：

```
mysql> `SELECT name FROM color ORDER BY name;`
+--------+
| name   |
+--------+
| blue   |
| green  |
| indigo |
| orange |
| red    |
| violet |
| yellow |
+--------+
```

假设您想按照彩虹中颜色出现的顺序对列进行排序。（这是 <q>Roy G. Biv</q> 顺序；该名称的连续字母表示相应颜色名称的首字母。）产生彩虹排序的一种方法是使用 `FIELD()`：

```
mysql> `SELECT name FROM color`
    -> `ORDER BY`
    -> `FIELD(name,'red','orange','yellow','green','blue','indigo','violet');`
+--------+
| name   |
+--------+
| red    |
| orange |
| yellow |
| green  |
| blue   |
| indigo |
| violet |
+--------+
```

要实现不使用 `FIELD()` 达到相同的目的，可以使用 `ALTER` `TABLE` 将 `name` 列转换为按照彩虹中颜色出现顺序列出的 `ENUM`：

```
mysql> `ALTER TABLE color`
    -> `MODIFY name`
    -> `ENUM('red','orange','yellow','green','blue','indigo','violet');`
```

在转换表后，对 `name` 列进行排序自然地产生彩虹排序，无需特殊处理：

```
mysql> `SELECT name FROM color ORDER BY name;`
+--------+
| name   |
+--------+
| red    |
| orange |
| yellow |
| green  |
| blue   |
| indigo |
| violet |
+--------+
```

请注意，一旦您切换到 `ENUM` 数据类型，将无法插入不属于列表的任何值。如果需要更改 `ENUM` 定义，例如添加新颜色，您将需要执行另一条 `ALTER` 命令。
