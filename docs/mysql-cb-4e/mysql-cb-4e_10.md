# 第十章：生成摘要

# 10.0 简介

数据库系统不仅对数据存储和检索有用，还可以以更简洁的形式对数据进行总结。摘要在您想要整体图景而不是详细信息时非常有用。它们比长列表的记录更容易理解。它们使您能够回答诸如“有多少？”、“总共多少？”或“值的范围是多少？”这样的问题。如果您经营一家企业，您可能想知道每个州有多少客户，或者每个月的销售额是多少。

前述示例包括两种常见的摘要类型：计数摘要和内容摘要。第一种（每个州的客户记录数）是计数摘要。每条记录的内容仅在将其放入适当的组或类别以进行计数时才重要。这些摘要本质上是直方图，您将项目分类到一组箱中，并计算每个箱中的项目数。第二个示例（每月销售额）是内容摘要，其中销售总额基于订单记录中的销售值。

另一种摘要类型既不产生计数也不产生总和，而只是一个唯一值列表。如果您关心的是存在哪些值而不是每个值有多少个，这将非常有用。要确定您的客户所在的州，请获取记录中包含的唯一州名列表，而不是从每条记录中获取的州值列表。

可用于您的摘要类型取决于数据的性质。计数摘要可以从各种值生成，无论它们是数字、字符串还是日期。生成总和或平均值的摘要仅适用于数值。您可以计算客户州名称的实例数量，以生成客户基础的人口统计分析。有时，将一个摘要技术应用于另一个的结果是有意义的。例如，要确定客户居住在多少州中，请生成唯一客户州的列表，然后计数它们。

MySQL 中的摘要操作涉及以下 SQL 结构：

+   要从一组单独的值计算摘要值，请使用称为聚合函数之一的函数。它们之所以被称为聚合函数，是因为它们操作值的聚合（组）。聚合函数包括`COUNT()`，用于计算查询结果中的行或值数量；`MIN()`和`MAX()`，用于查找最小和最大值；以及`SUM()`和`AVG()`，用于生成值的总和和平均值。这些函数可以用于计算整个结果集的值，或者与`GROUP BY`子句一起用于将行分组到子集中，并为每个子集获取聚合值。

+   要获取唯一值的列表，请使用`SELECT DISTINCT`而不是`SELECT`。

+   要计算唯一值，请使用`COUNT(DISTINCT)`而不是`COUNT()`。

本章的食谱首先说明了基本的汇总技术，然后展示了如何执行更复杂的汇总操作。您将在后续章节中找到更多汇总方法的示例，特别是涉及连接和统计操作的章节。（参见第十六章和第十七章.）

汇总查询有时涉及复杂的表达式。对于您经常执行的汇总，请记住视图可以使查询更容易使用。Recipe 5.7 演示了创建视图的基本技术。Recipe 10.5 展示了它如何应用于汇总简化，您将很容易看到它如何在本章的后续部分中使用。

本章中示例使用的主要表是`driver_log`和`mail`表。这些表也在第九章中使用过，所以应该看起来很熟悉。本章中经常使用的第三个表是`states`，其中每个美国州都有几列信息：

```
mysql> `SELECT * FROM states ORDER BY name;`
+----------------+--------+------------+----------+
| name           | abbrev | statehood  | pop      |
+----------------+--------+------------+----------+
| Alabama        | AL     | 1819-12-14 |  5039877 |
| Alaska         | AK     | 1959-01-03 |   732673 |
| Arizona        | AZ     | 1912-02-14 |  7276316 |
| Arkansas       | AR     | 1836-06-15 |  3025891 |
| California     | CA     | 1850-09-09 | 39237836 |
| Colorado       | CO     | 1876-08-01 |  5812069 |
| Connecticut    | CT     | 1788-01-09 |  3605597 |
…
```

`name`和`abbrev`列列出完整的州名和相应的缩写。`statehood`列表示州加入联盟的日期。`pop`是 2010 年人口普查时的州人口，由美国人口普查局报告。

本章偶尔还使用其他表。您可以使用`recipes`发行版的`tables`目录中找到的脚本创建它们。Recipe 7.15 描述了`reviews`表。

# 10.1 使用 COUNT()进行汇总

## 问题

您想计算整个表中的行数或符合特定条件的行数。

## 解决方案

使用`COUNT()`函数。

## 讨论

`COUNT()`函数计算行数。例如，要显示表中的行，请使用`SELECT *`语句，但要计算行数，请使用`SELECT COUNT(*)`。如果没有`WHERE`子句，则该语句计算表中的所有行数，如下面显示的`driver_log`表包含多少行的语句：

```
mysql> `SELECT COUNT(*) FROM driver_log;`
+----------+
| COUNT(*) |
+----------+
|       10 |
+----------+
```

如果你不知道美国有多少个州（也许你认为有 57 个？），这个声明告诉你：

```
mysql> `SELECT COUNT(*) FROM states;`
+----------+
| COUNT(*) |
+----------+
|       50 |
+----------+
```

如果没有`WHERE`子句的`COUNT(*)`执行完整的表扫描，除非存储引擎优化了这个函数。对于存储确切行数的 MyISAM 表来说，这非常快速。对于需要扫描主键中所有条目来执行`COUNT(*)`的 InnoDB 表来说，您可能希望避免使用这个函数，因为对于大表来说速度可能会很慢。如果近似的行数足够好，请通过从`INFORMATION_SCHEMA`数据库提取`TABLE_ROWS`值来避免全表扫描：

```
SELECT TABLE_ROWS FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'cookbook' AND TABLE_NAME = 'states';
```

要仅计算符合特定条件的行数，请在`SELECT COUNT(*)`语句中包含适当的`WHERE`子句。可以选择条件以使`COUNT(*)`适用于回答许多种类的问题：

+   驾驶员一天内超过 200 英里的次数有多少次？

    ```
    mysql> `SELECT COUNT(*) FROM driver_log WHERE miles > 200;`
    +----------+
    | COUNT(*) |
    +----------+
    |        4 |
    +----------+
    ```

+   Suzi 开车了多少天？

    ```
    mysql> `SELECT COUNT(*) FROM driver_log WHERE name = 'Suzi';`
    +----------+
    | COUNT(*) |
    +----------+
    |        2 |
    +----------+
    ```

+   19 世纪有多少个美国州加入联盟？

    ```
    mysql> `SELECT COUNT(*) FROM states`
        -> `WHERE statehood BETWEEN '1800-01-01' AND '1899-12-31';`
    +----------+
    | COUNT(*) |
    +----------+
    |       29 |
    +----------+
    ```

函数`COUNT()`实际上有两种形式。我们一直在使用的形式`COUNT(*)`用于计算行数。另一种形式`COUNT(`*`expr`*`)`接受列名或表达式作为参数，用于计算非`NULL`值的数量。以下语句展示了如何同时为表格计算行数和某列非`NULL`值的数量：

```
SELECT COUNT(*), COUNT(mycol) FROM mytbl;
```

`COUNT(`*`expr`*`)`不计算`NULL`值的事实在从同一组行中生成多个计数时非常有用。要在`driver_log`表中使用单个语句计算星期六和星期日的行程数，请执行以下操作：

```
mysql> `SELECT`
    -> `COUNT(IF(DAYOFWEEK(trav_date)=7,1,NULL)) AS 'Saturday trips',`
    -> `COUNT(IF(DAYOFWEEK(trav_date)=1,1,NULL)) AS 'Sunday trips'`
    -> `FROM driver_log;`
+----------------+--------------+
| Saturday trips | Sunday trips |
+----------------+--------------+
|              3 |            1 |
+----------------+--------------+
```

或者要计算周末与工作日的行程数，请执行以下操作：

```
mysql> `SELECT`
    -> `COUNT(IF(DAYOFWEEK(trav_date) IN (1,7),1,NULL)) AS 'weekend trips',`
    -> `COUNT(IF(DAYOFWEEK(trav_date) IN (1,7),NULL,1)) AS 'weekday trips'`
    -> `FROM driver_log;`
+---------------+---------------+
| weekend trips | weekday trips |
+---------------+---------------+
|             4 |             6 |
+---------------+---------------+
```

`IF()`表达式确定每个列值是否应计数。如果是，则表达式评估为`1`，`COUNT()`对其计数。如果不是，则表达式评估为`NULL`，`COUNT()`将忽略它。这样做的效果是计算满足`IF()`第一个参数给定条件的值的数量。

###### 小贴士

函数`COUNT()`计算元素的数量，因此您可以用任何其他值替换`1`。结果将相同。

## 另请参阅

关于`COUNT(*)`和`COUNT(expr)`之间的区别进一步讨论，请参见 Recipe 10.9。

# 10.2 使用 MIN()和 MAX()进行汇总

## 问题

您想要找到数据集中最小或最大的值。

## 解决方案

使用函数`MIN()`和`MAX()`相应地。

## 讨论

在数据集中找到最小或最大的值有点类似于排序，不同之处在于它不是生成整个排序值集合，而是仅选择排序范围的一个端点的单个值。这种操作适用于关于最小、最大、最老、最新、最昂贵、最便宜等问题。查找这些值的一种方法是使用`MIN()`和`MAX()`函数。（另一种方法是使用`LIMIT`；请参见 Recipe 5.9。）

因为`MIN()`和`MAX()`确定集合中的极端值，它们用于表征范围：

+   邮件表中的行代表的是哪个日期范围？最早和最晚发送的消息是什么？

    ```
    mysql> `SELECT`
        -> `MIN(t) AS earliest, MAX(t) AS latest,`
        -> `MIN(size) AS smallest, MAX(size) AS largest`
        -> `FROM mail;`
    +---------------------+---------------------+----------+---------+
    | earliest            | latest              | smallest | largest |
    +---------------------+---------------------+----------+---------+
    | 2014-05-11 10:15:08 | 2014-05-19 22:21:51 |      271 | 2394482 |
    +---------------------+---------------------+----------+---------+
    ```

+   美国各州人口中最小和最大的是哪些？

    ```
    mysql> `SELECT MIN(pop) AS 'fewest people', MAX(pop) AS 'most people'`
        -> `FROM states;`
    +---------------+-------------+
    | fewest people | most people |
    +---------------+-------------+
    |        578803 |    39237836 |
    +---------------+-------------+
    ```

+   从字面上来看，第一个和最后一个州名是什么？最短和最长名称的长度是多少？

    ```
    mysql> `SELECT`
        -> `MIN(name) AS first,`
        -> `MAX(name) AS last,`
        -> `MIN(CHAR_LENGTH(name)) AS shortest,`
        -> `MAX(CHAR_LENGTH(name)) AS longest`
        -> `FROM states;`
    +---------+---------+----------+---------+
    | first   | last    | shortest | longest |
    +---------+---------+----------+---------+
    | Alabama | Wyoming |        4 |      14 |
    +---------+---------+----------+---------+
    ```

最后一个查询示例说明了`MIN()`和`MAX()`不必直接应用于列值；它们还对从列值派生的表达式或值非常有用。

# 10.3 使用 SUM()和 AVG()进行汇总

## 问题

您想要计算一组值的总和或平均值（均值）。

## 解决方案

使用函数`SUM()`和`AVG()`。

## 讨论

`SUM()`和`AVG()`计算一组值的总和和平均值（均值）：

+   邮件流量的总字节数及每条消息的平均大小是多少？

    ```
    mysql> `SELECT`
        -> `SUM(size) AS 'total traffic',`
        -> `AVG(size) AS 'average message size'`
        -> `FROM mail;`
    +---------------+----------------------+
    | total traffic | average message size |
    +---------------+----------------------+
    |       3798185 |          237386.5625 |
    +---------------+----------------------+
    ```

+   `driver_log`表中的驾驶员行驶了多少英里？每天的平均行驶英里数是多少？

    ```
    mysql> `SELECT`
        -> `SUM(miles) AS 'total miles',`
        -> `AVG(miles) AS 'average miles/day'`
        -> `FROM driver_log;`
    +-------------+-------------------+
    | total miles | average miles/day |
    +-------------+-------------------+
    |        2166 |          216.6000 |
    +-------------+-------------------+
    ```

+   美国的总人口是多少？

    ```
    mysql> `SELECT SUM(pop) FROM states;`
    +-----------+
    | SUM(pop)  |
    +-----------+
    | 331223695 |
    +-----------+
    ```

    该值表示 2021 年人口普查报告的人口。

`SUM()`和`AVG()`是数值函数，因此不能与字符串或时间值一起使用。但有时可以将非数值值转换为有用的数值形式。假设一个表存储表示经过时间的`TIME`值：

```
mysql> `SELECT t1 FROM time_val;`
+----------+
| t1       |
+----------+
| 15:00:00 |
| 05:01:30 |
| 12:30:20 |
+----------+
```

要计算总经过时间，请使用`TIME_TO_SEC()`将值转换为秒，然后再求和。得到的总和也是秒数；将其传递给`SEC_TO_TIME()`以将其转换回`TIME`格式：

```
mysql> `SELECT SUM(TIME_TO_SEC(t1)) AS 'total seconds',`
    -> `SEC_TO_TIME(SUM(TIME_TO_SEC(t1))) AS 'total time'`
    -> `FROM time_val;`
+---------------+------------+
| total seconds | total time |
+---------------+------------+
|        117110 | 32:31:50   |
+---------------+------------+
```

## 另请参阅

`SUM()`和`AVG()`函数在统计应用中特别有用。它们在第十七章中进一步探讨，还有一个相关函数`STD()`，用于计算标准偏差。

# 10.4 使用 DISTINCT 消除重复项

## 问题

您希望在执行计算时跳过重复值。

## 解决方案

使用关键词`DISTINCT`。

## 讨论

不使用聚合函数的汇总操作是确定数据集中唯一值或行。可以使用`DISTINCT`（或其同义词`DISTINCTROW`）来实现这一点。`DISTINCT`将查询结果简化，并经常与`ORDER BY`结合使用以使值按更有意义的顺序排列。此查询按词汇顺序列出了`driver_log`表中的驾驶员名单：

```
mysql> `SELECT DISTINCT name FROM driver_log ORDER BY name;`
+-------+
| name  |
+-------+
| Ben   |
| Henry |
| Suzi  |
+-------+
```

没有`DISTINCT`，该语句生成相同的名称，但即使是在小数据集中，也不容易理解：

```
mysql> `SELECT name FROM driver_log ORDER BY NAME;`
+-------+
| name  |
+-------+
| Ben   |
| Ben   |
| Ben   |
| Henry |
| Henry |
| Henry |
| Henry |
| Henry |
| Suzi  |
| Suzi  |
+-------+
```

要确定不同驾驶员的数量，请使用`COUNT(DISTINCT)`：

```
mysql> `SELECT COUNT(DISTINCT name) FROM driver_log;`
+----------------------+
| COUNT(DISTINCT name) |
+----------------------+
|                    3 |
+----------------------+
```

`COUNT(DISTINCT)`会忽略`NULL`值。如果需要将`NULL`视为集合中的一个值，请使用以下表达式之一：

```
COUNT(DISTINCT *`val`*) + IF(COUNT(IF(*`val`* IS NULL,1,NULL))=0,0,1)
COUNT(DISTINCT *`val`*) + IF(SUM(ISNULL(*`val`*))=0,0,1)
COUNT(DISTINCT *`val`*) + (SUM(ISNULL(*`val`*))<>0);
```

在这个例子中，我们首先计算非空值的不同值的数量，然后如果`NULL`值的总和大于零，则再加`1`。

`DISTINCT`查询通常与聚合函数结合使用，以更全面地描述您的数据。假设`customer`表包含一个指示客户位置的`state`列。对`customer`表应用`COUNT(*)`可以告诉您有多少客户，对`state`列应用`DISTINCT`可以告诉您有客户的州数，而对`state`列应用`COUNT(DISTINCT)`可以告诉您客户群体代表了多少州。

当与多列一起使用时，`DISTINCT`显示列中值的不同组合，`COUNT(DISTINCT)`计算组合的数量。以下语句显示了`mail`表中不同的发件人/收件人对及其数量：

```
mysql> `SELECT DISTINCT srcuser, dstuser FROM mail`
    -> `ORDER BY srcuser, dstuser;`
+---------+---------+
| srcuser | dstuser |
+---------+---------+
| barb    | barb    |
| barb    | tricia  |
| gene    | barb    |
| gene    | gene    |
| gene    | tricia  |
| phil    | barb    |
| phil    | phil    |
| phil    | tricia  |
| tricia  | gene    |
| tricia  | phil    |
+---------+---------+
mysql> `SELECT COUNT(DISTINCT srcuser, dstuser) FROM mail;`
+----------------------------------+
| COUNT(DISTINCT srcuser, dstuser) |
+----------------------------------+
|                               10 |
+----------------------------------+
```

# 10.5 创建一个视图以简化使用汇总

## 问题

您希望更容易地执行汇总操作。

## 解决方案

创建一个自动完成此操作的视图。

## 讨论

如果您经常需要特定的摘要，一种使您避免重复输入总结表达式的技术是使用视图（参见食谱 5.7）。例如，以下视图实现了周末与工作日行程总结的讨论（参见食谱 10.1）：

```
mysql> `CREATE VIEW trip_summary_view AS`
    -> `SELECT`
    -> `COUNT(IF(DAYOFWEEK(trav_date) IN (1,7),1,NULL)) AS weekend_trips,`
    -> `COUNT(IF(DAYOFWEEK(trav_date) IN (1,7),NULL,1)) AS weekday_trips`
    -> `FROM driver_log;`
```

从此视图中选择比直接从底层表中选择更容易：

```
mysql> `SELECT * FROM trip_summary_view;`
+---------------+---------------+
| weekend_trips | weekday_trips |
+---------------+---------------+
|             4 |             6 |
+---------------+---------------+
```

虽然

# 10.6 寻找与最小值和最大值相关的值

## 问题

你想知道包含最小值或最大值的行中其他列的值。

## 解决方案

使用两个语句和一个用户定义的变量。或者一个子查询。或者一个连接。或者一个 CTE。

## 讨论

`MIN()` 和 `MAX()` 找到值范围的端点，但您可能也对包含该值的行中的其他值感兴趣。例如，您可以像这样找到最大的州人口：

```
mysql> `SELECT MAX(pop) FROM states;`
+----------+
| MAX(pop) |
+----------+
| 39237836 |
+----------+
```

但这并不能告诉你哪个州拥有这一人口。想要获取这些信息的显而易见的尝试如下：

```
mysql> `SELECT MAX(pop), name FROM states WHERE pop = MAX(pop);`
ERROR 1111 (HY000): Invalid use of group function
```

或许每个人迟早都会尝试这样做，但这是行不通的。聚合函数如 `MIN()` 和 `MAX()` 不能在 `WHERE` 子句中使用，后者需要适用于单个行的表达式。该语句的目的是确定哪一行具有最大的人口值，并显示相关的州名。问题在于，虽然你我都清楚地知道我们写这样一段话的意图，但在 SQL 中完全没有任何意义。该语句失败是因为 SQL 使用 `WHERE` 子句来确定选择哪些行，但聚合函数的值只有在选择确定该函数值的行后才能知道！因此，从某种意义上说，该语句是自相矛盾的。为了解决这个问题，将最大人口值保存在用户定义的变量中，然后将行与变量值进行比较：

```
mysql> `SET @max = (SELECT MAX(pop) FROM states);`
mysql> `SELECT pop AS 'highest population', name FROM states WHERE pop = @max;`
+--------------------+------------+
| highest population | name       |
+--------------------+------------+
|           39237836 | California |
+--------------------+------------+
```

或者，对于单语句解决方案，请在 `WHERE` 子查询中使用子查询返回最大人口值：

```
SELECT pop AS 'highest population', name FROM states
WHERE pop = (SELECT MAX(pop) FROM states);
```

即使样本亚马逊评论数据中没有实际包含最小值或最大值本身，而是从中推导出来，这种技术也同样适用。要确定样本中最短评论的长度，请执行以下操作：

```
mysql> `SELECT MAX(CHAR_LENGTH(reviews_virtual)) FROM reviews;`
+-----------------------------------+
| MIN(CHAR_LENGTH(reviews_virtual)) |
+-----------------------------------+
|                                 2 |
+-----------------------------------+
```

如果你想知道<q>这是哪一次评审？</q>请使用以下方法代替：

```
mysql> `SELECT JSON_EXTRACT(appliences_review, "$. reviewTime") as ReviewTime,`
    -> `JSON_EXTRACT(appliences_review, "$.reviewerID") as ReviwerID,` 
    -> `JSON_EXTRACT(appliences_review, "$.asin") as ProductID` 
    -> `JSON_EXTRACT(appliences_review, "$.overall") as Rating FROM` 
    -> `reviews WHERE CHAR_LENGTH(reviews_virtual) =` 
    -> `(SELECT MIN(CHAR_LENGTH(reviews_virtual)) FROM reviews);` 
+---------------+------------------+--------------+--------+
| ReviewTime    | ReviwerID        | ProductID    | Rating |
+---------------+------------------+--------------+--------+
| "03 8, 2015"  | "A3B1B4E184FSUZ" | "B000VL060M" | 5.0    |
| "03 8, 2015"  | "A3B1B4E184FSUZ" | "B0015UGPWQ" | 5.0    |
| "03 8, 2015"  | "A3B1B4E184FSUZ" | "B000VL060M" | 5.0    |
| "03 8, 2015"  | "A3B1B4E184FSUZ" | "B0015UGPWQ" | 5.0    |
| "02 9, 2015"  | "A3B1B4E184FSUZ" | "B0042U16YI" | 5.0    |
| "07 25, 2016" | "AJPRN1TD1A0SD"  | "B00BIZDI0A" | 3.0    |
+---------------+------------------+--------------+--------+
```

选择包含最小值或最大值的行中的其他列的另一种方法是使用连接。将该值选入另一个表，然后将其与原始表连接以选择与该值匹配的行。要找到人口最多的州的行，请使用如下连接：

```
mysql> `CREATE TEMPORARY TABLE tmp SELECT MAX(pop) as maxpop FROM states;`
mysql> `SELECT states.* FROM states INNER JOIN tmp`
    -> `ON states.pop = tmp.maxpop;`
+------------+--------+------------+----------+
| name       | abbrev | statehood  | pop      |
+------------+--------+------------+----------+
| California | CA     | 1850-09-09 | 39237836 |
+------------+--------+------------+----------+
```

自 MySQL 8.0 起，您可以使用公共表达式（CTE）执行相同的搜索。

```
mysql> `WITH` `maxpop`
    -> `AS` `(``SELECT` `MAX``(``pop``)` `as` `maxpop` `FROM` `states``)`
    -> `SELECT` `states``.``*` `FROM` `states`
    -> `JOIN` `maxpop` `ON` `states``.``pop` `=` `maxpop``.``maxpop``;`
+------------+--------+------------+----------+ | name       | abbrev | statehood  | pop      |
+------------+--------+------------+----------+ | California | CA     | 1850-09-09 | 39237836 |
+------------+--------+------------+----------+ 1 row in set (0.00 sec)
```

上述两个代码片段使用了相同的思想：创建一个临时表来存储最大人口数，并将其与原始表连接。但后者在单个查询中执行此操作，因此您无需在获得结果后担心销毁临时表。我们将在配方 10.18 中详细讨论 CTE。

## 参见

配方 16.7 扩展了对在数据集中查找包含多个组的最小或最大值的行的讨论。

# 10.7 控制 `MIN()` 和 `MAX()` 的字符串大小写敏感性

## 问题

`MIN()` 和 `MAX()` 在您不希望它们以区分大小写的方式选择字符串时，或者反之时，以区分大小写的方式选择字符串。

## 解决方案

使用字符串的不同比较特性。

## 讨论

配方 7.1 讨论了字符串比较属性如何取决于字符串是二进制还是非二进制的：

+   二进制字符串是字节序列。它们按照数值字节值逐字节进行比较。字符集和大小写对比较没有意义。

+   非二进制字符串是字符序列。它们有字符集和排序规则，并按照排序规则定义的顺序逐字符进行比较。

这些属性也适用于作为 `MIN()` 或 `MAX()` 函数参数的字符串列，因为它们基于比较。要改变这些函数如何处理字符串列的方式，请修改列的比较属性。配方 7.7 讨论了如何控制这些属性，而配方 9.4 展示了它们如何应用于字符串排序。相同的原则适用于查找最小和最大字符串值，因此我在这里只做总结；阅读配方 9.4 获取更多详情。

+   要以区分大小写的方式比较大小写不敏感的字符串，请使用区分大小写的排序规则对值进行排序：

    ```
    SELECT
    MIN(str_col COLLATE utf8mb4_0900_as_cs) AS min,
    MAX(str_col COLLATE utf8mb4_0900_as_cs) AS max
    FROM tbl;
    ```

+   要以区分大小写的方式比较大小写敏感的字符串，请使用区分大小写的排序规则对值进行排序：

    ```
    SELECT
    MIN(str_col COLLATE utf8mb4_0900_ai_ci) AS min,
    MAX(str_col COLLATE utf8mb4_0900_ai_ci) AS max
    FROM tbl;
    ```

    另一个可能性是比较已全部转换为相同大小写的值，从而使大小写无关紧要。但这也会改变检索到的值：

    ```
    SELECT
    MIN(UPPER(str_col)) AS min,
    MAX(UPPER(str_col)) AS max
    FROM tbl;
    ```

+   二进制字符串使用数值字节值进行比较，因此没有字母大小写的概念。然而，因为不同大小写的字母具有不同的字节值，所以二进制字符串的比较实际上是大小写敏感的。（即，`a` 和 `A` 是不相等的。）要以区分大小写的顺序比较二进制字符串，请将它们转换为非二进制字符串并应用适当的排序规则：

    ```
    SELECT
    MIN(CONVERT(str_col USING utf8mb4) COLLATE utf8mb4_0900_ai_ci) AS min,
    MAX(CONVERT(str_col USING utf8mb4) COLLATE utf8mb4_0900_ai_ci) AS max
    FROM tbl;
    ```

    如果默认排序规则不区分大小写（例如`utf8mb4`），则可以省略`COLLATE`子句。

# 10.8 将摘要分成子组

## 问题

您希望为一组行的每个子组生成摘要，而不是整体摘要值。

## 解决方案

使用 `GROUP BY` 子句将行分组。

## 讨论

到目前为止显示的汇总语句计算了结果集中所有行的汇总值。例如，以下语句确定了`mail`表中的记录数，因此也是发送的邮件总数：

```
mysql> `SELECT COUNT(*) FROM mail;`
+----------+
| COUNT(*) |
+----------+
|       16 |
+----------+
```

要将一组行排列成子组并总结每个组，请与`GROUP BY`子句一起使用聚合函数。要确定每个发件人的消息数量，请按发件人名称对行进行分组，计算每个名称出现的次数，并显示名称及其计数：

```
mysql> `SELECT srcuser, COUNT(*) FROM mail GROUP BY srcuser;`
+---------+----------+
| srcuser | COUNT(*) |
+---------+----------+
| barb    |        3 |
| gene    |        6 |
| phil    |        5 |
| tricia  |        2 |
+---------+----------+
```

该查询汇总了用于分组的同一列（`srcuser`），但这并不总是必要的。假设您想要快速了解`mail`表，显示在其中列出的每个发送者发送的总流量（以字节为单位）及每条消息的平均字节数。在这种情况下，您仍然使用`srcuser`列对行进行分组，但汇总`size`值：

```
mysql> `SELECT srcuser,`
    -> `SUM(size) AS 'total bytes',`
    -> `AVG(size) AS 'bytes per message'`
    -> `FROM mail GROUP BY srcuser;`
+---------+-------------+-------------------+
| srcuser | total bytes | bytes per message |
+---------+-------------+-------------------+
| barb    |      156696 |        52232.0000 |
| gene    |     1033108 |       172184.6667 |
| phil    |       18974 |         3794.8000 |
| tricia  |     2589407 |      1294703.5000 |
+---------+-------------+-------------------+
```

使用尽可能多的分组列以达到您需要的细粒度分组。早先显示每个发送者发送的消息数量的查询是一种粗略的汇总。要更具体地了解每个发送者从每个主机发送了多少消息，请使用两个分组列。这会生成一个带有嵌套组的结果（组中的组）：

```
mysql> `SELECT srcuser, srchost, COUNT(srcuser) FROM mail`
    -> `GROUP BY srcuser, srchost;`
+---------+---------+----------------+
| srcuser | srchost | COUNT(srcuser) |
+---------+---------+----------------+
| barb    | saturn  |              2 |
| barb    | venus   |              1 |
| gene    | mars    |              2 |
| gene    | saturn  |              2 |
| gene    | venus   |              2 |
| phil    | mars    |              3 |
| phil    | venus   |              2 |
| tricia  | mars    |              1 |
| tricia  | saturn  |              1 |
+---------+---------+----------------+
```

本节中的前述示例使用了`COUNT()`、`SUM()`和`AVG()`来进行每组汇总。您也可以使用`MIN()`或`MAX()`。配合`GROUP BY`子句，它们会返回每组的最小值或最大值。以下查询将`mail`表的行按消息发送者分组，显示每个发送者发送的最大消息的大小和最近消息的日期：

```
mysql> `SELECT srcuser, MAX(size), MAX(t) FROM mail GROUP BY srcuser;`
+---------+-----------+---------------------+
| srcuser | MAX(size) | MAX(t)              |
+---------+-----------+---------------------+
| barb    |     98151 | 2014-05-14 14:42:21 |
| gene    |    998532 | 2014-05-19 22:21:51 |
| phil    |     10294 | 2014-05-19 12:49:23 |
| tricia  |   2394482 | 2014-05-14 17:03:01 |
+---------+-----------+---------------------+
```

您可以按多个列进行分组，并显示每对这些列值之间的最大值。此查询查找了在`mail`表中列出的每对发送者和接收者值之间发送的最大消息的大小：

```
mysql> `SELECT srcuser, dstuser, MAX(size) FROM mail GROUP BY srcuser, dstuser;`
+---------+---------+-----------+
| srcuser | dstuser | MAX(size) |
+---------+---------+-----------+
| barb    | barb    |     98151 |
| barb    | tricia  |     58274 |
| gene    | barb    |      2291 |
| gene    | gene    |     23992 |
| gene    | tricia  |    998532 |
| phil    | barb    |     10294 |
| phil    | phil    |      1048 |
| phil    | tricia  |      5781 |
| tricia  | gene    |    194925 |
| tricia  | phil    |   2394482 |
+---------+---------+-----------+
```

在使用聚合函数生成每组汇总值时，请注意以下陷阱，这涉及选择与分组列不相关的非汇总表列。假设您想知道`driver_log`表中每位驾驶员的最长行程：

```
mysql> `SELECT name, MAX(miles) AS 'longest trip'`
    -> `FROM driver_log GROUP BY name;`
+-------+--------------+
| name  | longest trip |
+-------+--------------+
| Ben   |          152 |
| Henry |          300 |
| Suzi  |          502 |
+-------+--------------+
```

但是如果你还想显示每位司机最长行程发生的日期怎么办？你只需将`trav_date`添加到输出列列表中吗？抱歉，这样不起作用：

```
mysql> `SELECT name, trav_date, MAX(miles) AS 'longest trip'`
    -> `FROM driver_log GROUP BY name;`
+-------+------------+--------------+
| name  | trav_date  | longest trip |
+-------+------------+--------------+
| Ben   | 2014-07-30 |          152 |
| Henry | 2014-07-29 |          300 |
| Suzi  | 2014-07-29 |          502 |
+-------+------------+--------------+
```

查询确实产生了结果，但是如果您将其与完整表格（如此处所示）进行比较，您会看到虽然本和亨利的日期是正确的，但苏茜的日期却不正确：

```
+--------+-------+------------+-------+
| rec_id | name  | trav_date  | miles |
+--------+-------+------------+-------+
|      1 | Ben   | 2014-07-30 |   152 |    ← Ben's longest trip
|      2 | Suzi  | 2014-07-29 |   391 |
|      3 | Henry | 2014-07-29 |   300 |    ← Henry's longest trip
|      4 | Henry | 2014-07-27 |    96 |
|      5 | Ben   | 2014-07-29 |   131 |
|      6 | Henry | 2014-07-26 |   115 |
|      7 | Suzi  | 2014-08-02 |   502 |    ← Suzi's longest trip
|      8 | Henry | 2014-08-01 |   197 |
|      9 | Ben   | 2014-08-02 |    79 |
|     10 | Henry | 2014-07-30 |   203 |
+--------+-------+------------+-------+
```

到底发生了什么？为什么汇总语句产生了不正确的结果？这是因为当您在查询中包含`GROUP BY`子句时，您只能有意义地选择分组列或从分组计算的汇总值。如果显示其他表列，它们与分组列不相关，显示的值是不明确的。（对于刚刚显示的语句，MySQL 可能会简单地选择每个驱动程序的第一个日期，而不管它是否与驱动程序的最大里程值匹配。）

若要避免不小心假定`trav_date`的值是正确的并且设置`ONLY_FULL_GROUP_BY` SQL 模式，使得选择不明确的值非法：

```
mysql> `SET sql_mode = 'ONLY_FULL_GROUP_BY';`
mysql> `SELECT name, trav_date, MAX(miles) AS 'longest trip'`
    -> `FROM driver_log GROUP BY name;`
ERROR 1055 (42000): 'cookbook.driver_log.trav_date' isn't in GROUP BY
```

SQL 模式`ONLY_FULL_GROUP_BY`自 MySQL 5.7 默认设置的一部分。但我们已经看到许多旧版应用程序禁用了此选项。我们建议您始终启用`ONLY_FULL_GROUP_BY`并修复返回错误的查询。

解决显示与最小或最大组值相关联的行内容的一般问题涉及联接。该技术在 Recipe 16.7 中有描述。对于手头的问题，可以按以下方式生成所需的结果：

```
mysql> `CREATE TEMPORARY TABLE t`
    -> `SELECT name, MAX(miles) AS miles FROM driver_log GROUP BY name;`
mysql> `SELECT d.name, d.trav_date, d.miles AS 'longest trip'`
    -> `FROM driver_log AS d INNER JOIN t USING (name, miles) ORDER BY name;`
+-------+------------+--------------+
| name  | trav_date  | longest trip |
+-------+------------+--------------+
| Ben   | 2014-07-30 |          152 |
| Henry | 2014-07-29 |          300 |
| Suzi  | 2014-08-02 |          502 |
+-------+------------+--------------+
```

或者，通过使用 CTE：

```
mysql>  `WITH` `t` `AS`
    ->  `(``SELECT` `name``,` `MAX``(``miles``)` `AS` `miles` `FROM` `driver_log` `GROUP` `BY` `name``)`
    ->  `SELECT` `d``.``name``,` `d``.``trav_date``,` `d``.``miles` `AS` `'longest trip'`
    ->  `FROM` `driver_log` `AS` `d` `INNER` `JOIN` `t` `USING` `(``name``,` `miles``)` `ORDER` `BY` `name``;`
+-------+------------+--------------+ | name  | trav_date  | longest trip |
+-------+------------+--------------+ | Ben   | 2014-07-30 |          152 |
| Henry | 2014-07-29 |          300 |
| Suzi  | 2014-08-02 |          502 |
+-------+------------+--------------+ 3 rows in set (0.01 sec)
```

# 10.9 使用聚合函数处理`NULL`值

## 问题

您正在汇总可能包含`NULL`值的一组值，并且需要知道如何解释结果。

## 解决方案

了解聚合函数如何处理`NULL`值。

## 讨论

大多数聚合函数会忽略`NULL`值。`COUNT()`不同：`COUNT(`*`expr`*`)`会忽略`NULL`实例的*`expr`*，但`COUNT(*)`会计算行数，不考虑内容。

假设一个`expt`表包含要为每个科目进行四次测试的实验结果，并且对于尚未进行的测试，将测试分数标记为`NULL`：

```
mysql> `SELECT subject, test, score FROM expt ORDER BY subject, test;`
+---------+------+-------+
| subject | test | score |
+---------+------+-------+
| Jane    | A    |    47 |
| Jane    | B    |    50 |
| Jane    | C    |  NULL |
| Jane    | D    |  NULL |
| Marvin  | A    |    52 |
| Marvin  | B    |    45 |
| Marvin  | C    |    53 |
| Marvin  | D    |  NULL |
+---------+------+-------+
```

通过使用`GROUP BY`子句按科目名称排列行，可以像这样计算每个科目的测试次数以及总数、平均数、最低分和最高分：

```
mysql> `SELECT subject,`
    -> `COUNT(score) AS n,`
    -> `SUM(score) AS total,`
    -> `AVG(score) AS average,`
    -> `MIN(score) AS lowest,`
    -> `MAX(score) AS highest`
    -> `FROM expt GROUP BY subject;`
+---------+---+-------+---------+--------+---------+
| subject | n | total | average | lowest | highest |
+---------+---+-------+---------+--------+---------+
| Jane    | 2 |    97 | 48.5000 |     47 |      50 |
| Marvin  | 3 |   150 | 50.0000 |     45 |      53 |
+---------+---+-------+---------+--------+---------+
```

您可以从标记为`n`（测试次数）的列中的结果中看到，查询仅计算了五个值，即使表中包含了八个。为什么？因为该列中的值对应于每个科目的非`NULL`测试分数的数量。其他摘要列也仅显示从非`NULL`分数计算出的结果。

对于聚合函数忽略`NULL`值是有很多意义的。如果它们遵循通常的 SQL 算术规则，将`NULL`添加到任何其他值会产生一个`NULL`结果。这将使得聚合函数非常难以使用：为了避免得到`NULL`结果，您每次执行汇总时都必须过滤`NULL`值。通过忽略`NULL`值，聚合函数变得更加方便。

请注意，即使聚合函数可能会忽略`NULL`值，其中一些仍然可能会生成`NULL`作为结果。如果没有要总结的内容，就会发生这种情况，即值集合为空或仅包含`NULL`值。以下查询与前一个查询相同，只有一个小差异。它仅选择`NULL`测试分数，以说明当聚合函数没有要操作的内容时会发生什么：

```
mysql> `SELECT subject,`
    -> `COUNT(score) AS n,`
    -> `SUM(score) AS total,`
    -> `AVG(score) AS average,`
    -> `MIN(score) AS lowest,`
    -> `MAX(score) AS highest`
    -> `FROM expt WHERE score IS NULL GROUP BY subject;`
+---------+---+-------+---------+--------+---------+
| subject | n | total | average | lowest | highest |
+---------+---+-------+---------+--------+---------+
| Jane    | 0 |  NULL |    NULL |   NULL |    NULL |
| Marvin  | 0 |  NULL |    NULL |   NULL |    NULL |
+---------+---+-------+---------+--------+---------+
```

对于`COUNT()`，每个科目的分数数量为零，并以此方式报告。另一方面，当没有要总结的值时，`SUM()`、`AVG()`、`MIN()`和`MAX()`返回`NULL`。如果不希望将聚合值为`NULL`显示为`NULL`，请使用`IFNULL()`适当映射它：

```
mysql> `SELECT subject,`
    -> `COUNT(score) AS n,`
    -> `IFNULL(SUM(score),0) AS total,`
    -> `IFNULL(AVG(score),0) AS average,`
    -> `IFNULL(MIN(score),'Unknown') AS lowest,`
    -> `IFNULL(MAX(score),'Unknown') AS highest`
    -> `FROM expt WHERE score IS NULL GROUP BY subject;`
+---------+---+-------+---------+---------+---------+
| subject | n | total | average | lowest  | highest |
+---------+---+-------+---------+---------+---------+
| Jane    | 0 |     0 |  0.0000 | Unknown | Unknown |
| Marvin  | 0 |     0 |  0.0000 | Unknown | Unknown |
+---------+---+-------+---------+---------+---------+
```

`COUNT()`在处理`NULL`值方面与其他聚合函数略有不同。与其他聚合函数一样，`COUNT(`*`expr`*`)`仅计算非`NULL`值，但`COUNT(*)`计算行数，无论其内容如何。您可以通过以下方式查看`COUNT()`的不同形式之间的区别：

```
mysql> `SELECT COUNT(*), COUNT(score) FROM expt;`
+----------+--------------+
| COUNT(*) | COUNT(score) |
+----------+--------------+
|        8 |            5 |
+----------+--------------+
```

这告诉我们`expt`表中有八行，但只有五行填写了`score`值。`COUNT()`的不同形式对于计算缺失值非常有用。只需进行差异计算：

```
mysql> `SELECT COUNT(*) - COUNT(score) AS missing FROM expt;`
+---------+
| missing |
+---------+
|       3 |
+---------+
```

还可以为子组确定缺失和非缺失计数。以下查询对每个科目都这样做，提供了一种评估实验完成程度的简便方法：

```
mysql> `SELECT subject,`
    -> `COUNT(*) AS total,`
    -> `COUNT(score) AS 'nonmissing',`
    -> `COUNT(*) - COUNT(score) AS missing`
    -> `FROM expt GROUP BY subject;`
+---------+-------+------------+---------+
| subject | total | nonmissing | missing |
+---------+-------+------------+---------+
| Jane    |     4 |          2 |       2 |
| Marvin  |     4 |          3 |       1 |
+---------+-------+------------+---------+
```

# 10.10 仅选择具有特定特征的群组

## 问题

您想计算群组摘要，但仅显示符合特定条件的群组结果。

## 解决方案

使用`HAVING`子句。

## 讨论

您熟悉使用`WHERE`来指定语句必须满足的行条件。因此，自然而然地使用`WHERE`编写涉及总结值的条件。唯一的麻烦在于这样做行不通。要识别在`driver_log`表中驾驶超过三天的驾驶员，您可能会像这样写语句：

```
mysql> `SELECT COUNT(*), name FROM driver_log`
    -> `WHERE COUNT(*) > 3`
    -> `GROUP BY name;`
ERROR 1111 (HY000): Invalid use of group function
```

问题在于`WHERE`指定了初始约束条件，确定了要选择哪些行，但只有在选择行后才能确定`COUNT()`的值。解决方案是将`COUNT()`表达式放在`HAVING`子句中。`HAVING`类似于`WHERE`，但它适用于组特征，而不是单个行。也就是说，`HAVING`在已选择和分组的行集上操作，根据聚合函数结果应用额外约束，这些结果在初始选择过程中是未知的。因此，前面的查询应该像这样编写：

```
mysql> `SELECT COUNT(*), name FROM driver_log`
    -> `GROUP BY name`
    -> `HAVING COUNT(*) > 3;`
+----------+-------+
| COUNT(*) | name  |
+----------+-------+
|        5 | Henry |
+----------+-------+
```

当您使用`HAVING`时，仍然可以包括`WHERE`子句，但仅选择要总结的行，而不是测试已计算的汇总值。

`HAVING`可以引用别名，因此可以像这样重写先前的查询：

```
mysql> `SELECT COUNT(*) AS count, name FROM driver_log`
    -> `GROUP BY name`
    -> `HAVING count > 3;`
+-------+-------+
| count | name  |
+-------+-------+
|     5 | Henry |
+-------+-------+
```

# 10.11 使用计数确定值是否唯一

## 问题

您想知道表中的值是否是唯一的。

## 解决方案

结合 `COUNT()` 使用 `HAVING`。

## 讨论

`DISTINCT` 可以消除重复项，但不显示原始数据中实际重复的值。您可以使用 `HAVING` 找到 `DISTINCT` 不适用的情况中的唯一值。`HAVING` 可以告诉您哪些值是唯一的或非唯一的。

下面的语句显示了仅有一名司机活跃的日期以及有多名司机活跃的日期。它们基于使用 `HAVING` 和 `COUNT()` 来确定哪些 `trav_date` 值是唯一或非唯一的：

```
mysql> `SELECT trav_date, COUNT(trav_date) FROM driver_log`
    -> `GROUP BY trav_date HAVING COUNT(trav_date) = 1;`
+------------+------------------+
| trav_date  | COUNT(trav_date) |
+------------+------------------+
| 2014-07-26 |                1 |
| 2014-07-27 |                1 |
| 2014-08-01 |                1 |
+------------+------------------+
mysql> `SELECT trav_date, COUNT(trav_date) FROM driver_log`
    -> `GROUP BY trav_date HAVING COUNT(trav_date) > 1;`
+------------+------------------+
| trav_date  | COUNT(trav_date) |
+------------+------------------+
| 2014-07-29 |                3 |
| 2014-07-30 |                2 |
| 2014-08-02 |                2 |
+------------+------------------+
```

这种技术也适用于值的组合。例如，要查找仅发送了一条消息的消息发送者/接收者对，请查找在 `mail` 表中仅出现一次的组合：

```
mysql> `SELECT srcuser, dstuser FROM mail`
    -> `GROUP BY srcuser, dstuser HAVING COUNT(*) = 1;`
+---------+---------+
| srcuser | dstuser |
+---------+---------+
| barb    | barb    |
| gene    | tricia  |
| phil    | barb    |
| tricia  | gene    |
| tricia  | phil    |
+---------+---------+
```

注意，此查询不会打印计数。前面的示例这样做是为了展示计数的正确使用，但您可以在 `HAVING` 子句中引用聚合值，而无需将其包含在输出列列表中。

# 10.12 按表达式结果分组

## 问题

您想要根据从表达式计算的值将行分组为子组。

## 解决方案

在 `GROUP` `BY` 子句中，使用将值分类的表达式。

## 讨论

`GROUP` `BY` 和 `ORDER` `BY` 一样，可以引用表达式。这意味着您可以使用计算作为分组的依据。与 `ORDER` `BY` 类似，您可以直接在 `GROUP` `BY` 子句中编写分组表达式，或者在输出列列表中使用别名来引用该表达式，并在 `GROUP` `BY` 中引用该别名。

要找出一年中有多个州加入联盟的日期，请按州加入日期和日进行分组，然后使用 `HAVING` 和 `COUNT()` 找到非唯一组合：

```
mysql> `SELECT`
    -> `MONTHNAME(statehood) AS month,`
    -> `DAYOFMONTH(statehood) AS day,`
    -> `COUNT(*) AS count`
    -> `FROM states GROUP BY month, day HAVING count > 1;`
+----------+------+-------+
| month    | day  | count |
+----------+------+-------+
| February |   14 |     2 |
| June     |    1 |     2 |
| March    |    1 |     2 |
| May      |   29 |     2 |
| November |    2 |     2 |
+----------+------+-------+
```

# 10.13 总结非分类数据

## 问题

您想要总结一组不自然分类的值。

## 解决方案

使用表达式将值分组为类别。

## 讨论

Recipe 10.12 展示了如何按表达式结果分组行。这一重要应用是对非分类值进行分类。这是有用的，因为 `GROUP` `BY` 最适合具有重复值的列。例如，您可以尝试使用 `states` 表中的值对行进行分组以执行人口分析。由于列中的独特值较多，这并不奏效。事实上，它们*全部*都是独特的：

```
mysql> `SELECT COUNT(pop), COUNT(DISTINCT pop) FROM states;`
+------------+---------------------+
| COUNT(pop) | COUNT(DISTINCT pop) |
+------------+---------------------+
|         50 |                  50 |
+------------+---------------------+
```

在像这样的情况下，数值不能很好地分组到少数集合中，可以使用一种强制将其强制分为类别的转换。首先确定人口值的范围：

```
mysql> `SELECT MIN(pop), MAX(pop) FROM states;`
+----------+----------+
| MIN(pop) | MAX(pop) |
+----------+----------+
|   578803 | 39237836 |
+----------+----------+
```

您可以从该结果看出，如果将`pop`值除以 500 万，它们将分成八个类别——一个合理的数目。（类别范围将是 1 到 5,000,000，5,000,001 到 10,000,000 等等）。要将每个人口值放入正确的类别，请将其除以 500 万，并使用整数结果：

```
mysql> ``SELECT FLOOR(pop/5000000) AS `max population (millions)`,``
    -> `` COUNT(*) AS `number of states` ``
    -> `` FROM states GROUP BY `max population (millions)` ``
    -> ``ORDER BY `max population (millions)`;``
+---------------------------+------------------+
| max population (millions) | number of states |
+---------------------------+------------------+
|                         0 |               26 |
|                         1 |               14 |
|                         2 |                6 |
|                         3 |                1 |
|                         4 |                1 |
|                         5 |                1 |
|                         7 |                1 |
+---------------------------+------------------+
```

嗯。这还不太对。该表达式将人口值分组到少量类别中，但未正确报告类别值。让我们尝试将`FLOOR()`结果乘以五：

```
mysql> ``SELECT FLOOR(pop/5000000)*5 AS `max population (millions)`,``
    -> `` COUNT(*) AS `number of states` ``
    -> `` FROM states GROUP BY `max population (millions)` ``
    -> ``ORDER BY `max population (millions)`;``
+---------------------------+------------------+
| max population (millions) | number of states |
+---------------------------+------------------+
|                         0 |               26 |
|                         5 |               14 |
|                        10 |                6 |
|                        15 |                1 |
|                        20 |                1 |
|                        25 |                1 |
|                        35 |                1 |
+---------------------------+------------------+
```

这仍然不正确。最大州人口为 35,893,799，应该放入 4000 万类别中，而不是 3500 万类别中。问题在于生成类别表达式将值分组到每个类别的下限。为了将值分组到每个类别的上限，可以使用以下技术。对于大小为*n*的类别，使用以下表达式将值*x*放入正确的类别：

```
FLOOR((*`x`*+(*`n`*-1))/*`n`*)
```

因此，我们的查询的最终形式如下所示：

```
mysql> ``SELECT FLOOR((pop+4999999)/5000000)*5 AS `max population (millions)`,``
    -> `` COUNT(*) AS `number of states` ``
    -> `` FROM states GROUP BY `max population (millions)` ``
    -> ``ORDER BY `max population (millions)`;``
+---------------------------+------------------+
| max population (millions) | number of states |
+---------------------------+------------------+
|                         5 |               26 |
|                        10 |               14 |
|                        15 |                6 |
|                        20 |                1 |
|                        25 |                1 |
|                        30 |                1 |
|                        40 |                1 |
+---------------------------+------------------+
```

结果清楚地显示，大多数美国州的人口为五百万或更少。

在某些情况下，将群组按对数刻度分类可能更为合适。例如，如下处理州人口数值：

```
mysql> ``SELECT FLOOR(LOG10(pop)) AS `log10(population)`,``
    -> `` COUNT(*) AS `number of states` ``
    -> ``FROM states GROUP BY `log10(population)`;``
+-------------------+------------------+
| log10(population) | number of states |
+-------------------+------------------+
|                 5 |                5 |
|                 6 |               35 |
|                 7 |               10 |
+-------------------+------------------+
```

查询显示了人口分别以数十万、百万和数千万进行测量的州的数量。

您可能已经注意到，前面的查询中使用反引号（标识符引用）编写别名，而不是单引号（字符串引用）。在`GROUP BY`子句中引用的别名必须使用标识符引用，否则别名将被视为常量字符串表达式，导致分组产生错误的结果。标识符引用使 MySQL 明确知道别名是指输出列。输出列列表中的别名可以使用字符串引用进行编写；我们在这里使用反引号是为了避免在给定查询中混合别名引用风格。

# 10.14 查找最小或最大的摘要值

## 问题

您想计算每个群组的摘要值，但仅显示其中最小或最大的值。

## 解决方案

在语句中添加`LIMIT`子句。或者使用用户定义变量或子查询来选择适当的摘要。

## 讨论

`MIN()`和`MAX()`可以找到值集的端点值，但是要找到摘要值集的端点值，这些函数将无法工作。它们的参数不能是另一个聚合函数。例如，您可以轻松找到每位驾驶员的里程总数：

```
mysql> `SELECT name, SUM(miles)`
    -> `FROM driver_log`
    -> `GROUP BY name;`
+-------+------------+
| name  | SUM(miles) |
+-------+------------+
| Ben   |        362 |
| Henry |        911 |
| Suzi  |        893 |
+-------+------------+
```

要仅选择具有最多英里数的驾驶员行，以下方法不起作用：

```
mysql> `SELECT name, SUM(miles)`
    -> `FROM driver_log`
    -> `GROUP BY name`
    -> `HAVING SUM(miles) = MAX(SUM(miles));`
ERROR 1111 (HY000): Invalid use of group function
```

相反，首先按最大的`SUM()`值对行进行排序，并使用`LIMIT`选择第一行：

```
mysql> `SELECT name, SUM(miles)`
    -> `FROM driver_log`
    -> `GROUP BY name`
    -> `ORDER BY SUM(miles) DESC LIMIT 1;`
+-------+------------+
| name  | SUM(miles) |
+-------+------------+
| Henry |        911 |
+-------+------------+
```

但是，如果多个行具有给定的摘要值，则`LIMIT 1`查询将不会告诉您。例如，您可能尝试查明州名字的最常见的初始字母：

```
mysql> `SELECT LEFT(name,1) AS letter, COUNT(*) FROM states`
    -> `GROUP BY letter ORDER BY COUNT(*) DESC LIMIT 1;`
+--------+----------+
| letter | COUNT(*) |
+--------+----------+
| M      |        8 |
+--------+----------+
```

但是，还有八个州名称以`N`开头。要找出可能有多个最频繁值时，请使用用户定义变量或子查询确定最大计数，然后选择计数等于最大值的这些值：

```
mysql> `SET @max = (SELECT COUNT(*) FROM states`
    -> `GROUP BY LEFT(name,1) ORDER BY COUNT(*) DESC LIMIT 1);`
mysql> `SELECT LEFT(name,1) AS letter, COUNT(*) FROM states`
    -> `GROUP BY letter HAVING COUNT(*) = @max;`
+--------+----------+
| letter | COUNT(*) |
+--------+----------+
| M      |        8 |
| N      |        8 |
+--------+----------+
mysql> `SELECT LEFT(name,1) AS letter, COUNT(*) FROM states`
    -> `GROUP BY letter HAVING COUNT(*) =`
    ->   `(SELECT COUNT(*) FROM states`
    ->   `GROUP BY LEFT(name,1) ORDER BY COUNT(*) DESC LIMIT 1);`
+--------+----------+
| letter | COUNT(*) |
+--------+----------+
| M      |        8 |
| N      |        8 |
+--------+----------+
```

# 10.15 生成基于日期的摘要

## 问题

您希望根据日期或时间值生成摘要。

## 解决方案

使用`GROUP` `BY`将时间值放入适当时段的类别中。通常涉及使用提取日期或时间重要部分的表达式。

## 讨论

要根据时间对行进行排序，请使用带有时间列的`ORDER` `BY`。要根据时间间隔将行进行汇总，则需要确定如何将行分类到适当的间隔，并使用`GROUP` `BY`相应地对它们进行分组。

例如，要确定每天路上有多少驾驶员及每天驾驶了多少英里，请按日期将`driver_log`表中的行分组：^(1)

```
mysql> `SELECT trav_date,`
    -> `COUNT(*) AS 'number of drivers', SUM(miles) As 'miles logged'`
    -> `FROM driver_log GROUP BY trav_date;`
+------------+-------------------+--------------+
| trav_date  | number of drivers | miles logged |
+------------+-------------------+--------------+
| 2014-07-26 |                 1 |          115 |
| 2014-07-27 |                 1 |           96 |
| 2014-07-29 |                 3 |          822 |
| 2014-07-30 |                 2 |          355 |
| 2014-08-01 |                 1 |          197 |
| 2014-08-02 |                 2 |          581 |
+------------+-------------------+--------------+
```

然而，随着表中行数的增加，这种按天汇总会变得越来越长。随着时间的推移，不同日期的数量将变得非常多，使得摘要失去实用性，您可能决定增加类别的大小。例如，此查询按月进行分类：

```
mysql> `SELECT YEAR(trav_date) AS year, MONTH(trav_date) AS month,`
    -> `COUNT(*) AS 'number of drivers', SUM(miles) As 'miles logged'`
    -> `FROM driver_log GROUP BY year, month;`
+------+-------+-------------------+--------------+
| year | month | number of drivers | miles logged |
+------+-------+-------------------+--------------+
| 2014 |     7 |                 7 |         1388 |
| 2014 |     8 |                 3 |          778 |
+------+-------+-------------------+--------------+
```

现在，随着时间的推移，摘要行的数量增长速度大大放缓。最终，您可以仅基于年份进行汇总，以进一步折叠行。

时间分类的用途很多：

+   要从可能包含许多唯一值的`DATETIME`或`TIMESTAMP`列生成每日摘要，请去除一天中的时间部分，将所有出现在给定日期内的值折叠为相同值。以下任何一个`GROUP` `BY`子句都可以做到这一点，尽管最后一个可能最慢：

    ```
    GROUP BY DATE(*`col_name`*)
    GROUP BY FROM_DAYS(TO_DAYS(*`col_name`*))
    GROUP BY YEAR(*`col_name`*), MONTH(*`col_name`*), DAYOFMONTH(*`col_name`*)
    GROUP BY DATE_FORMAT(*`col_name`*,'%Y-%m-%e')
    ```

+   要生成每月或每季度的销售报告，请按照`MONTH(`*`col_name`*`)`或`QUARTER(`*`col_name`*`)`对日期进行分组，以确保数据能正确地对应到年度的相应部分。

# 10.16 同时处理每组和总体摘要值

## 问题

您希望生成一个需要不同级别摘要细节的报告。或者您想将每组的摘要值与总体摘要值进行比较。

## 解决方案

使用两个语句检索不同级别的摘要信息。或者使用子查询检索一个摘要值，并在外部查询中引用其他摘要值。对于仅显示多个摘要级别（而不对其执行其他计算）的应用程序，`WITH` `ROLLUP`可能已足够。

## 讨论

一些报告涉及多层次的摘要信息。以下报告显示了从`driver_log`表中每位驾驶员的总英里数，以及每位驾驶员英里数占整个表总英里数的百分比：

```
+-------+--------------+------------------------+
| name  | miles/driver | percent of total miles |
+-------+--------------+------------------------+
| Ben   |          362 |                16.7128 |
| Henry |          911 |                42.0591 |
| Suzi  |          893 |                41.2281 |
+-------+--------------+------------------------+
```

百分比表示每位驾驶员英里数占所有驾驶员总英里数的比例。为执行百分比计算，您需要每组摘要以获取每位驾驶员的英里数，并且还需要总体摘要以获取总英里数。首先运行一个查询来获取总里程数：

```
mysql> `SELECT @total := SUM(miles) AS 'total miles' FROM driver_log;`
+-------------+
| total miles |
+-------------+
|        2166 |
+-------------+
```

然后计算每组的值，并使用总体总计计算百分比：

```
mysql> `SELECT name,`
    -> `SUM(miles) AS 'miles/driver',`
    -> `(SUM(miles)*100)/@total AS 'percent of total miles'`
    -> `FROM driver_log GROUP BY name;`
+-------+--------------+------------------------+
| name  | miles/driver | percent of total miles |
+-------+--------------+------------------------+
| Ben   |          362 |                16.7128 |
| Henry |          911 |                42.0591 |
| Suzi  |          893 |                41.2281 |
+-------+--------------+------------------------+
```

要将两个语句合并为一个，使用子查询计算总英里数：

```
SELECT name,
SUM(miles) AS 'miles/driver',
(SUM(miles)*100)/(SELECT SUM(miles) FROM driver_log)
  AS 'percent of total miles'
FROM driver_log GROUP BY name;
```

一个类似的问题使用多个摘要级别比较每组摘要值与相应的总体摘要值。假设您希望显示驾驶员的平均每日英里数低于组平均值。在子查询中计算总体平均值，然后使用 `HAVING` 子句将每个驾驶员的平均值与总体平均值进行比较：

```
mysql> `SELECT name, AVG(miles) AS driver_avg FROM driver_log`
    -> `GROUP BY name`
    -> `HAVING driver_avg < (SELECT AVG(miles) FROM driver_log);`
+-------+------------+
| name  | driver_avg |
+-------+------------+
| Ben   |   120.6667 |
| Henry |   182.2000 |
+-------+------------+
```

要显示不同级别的摘要值（而不是执行一个摘要级别相对于另一个的计算），请将 `WITH` `ROLLUP` 添加到 `GROUP` `BY` 子句中：

```
mysql> `SELECT name, SUM(miles) AS 'miles/driver'`
    -> `FROM driver_log GROUP BY name WITH ROLLUP;`
+-------+--------------+
| name  | miles/driver |
+-------+--------------+
| Ben   |          362 |
| Henry |          911 |
| Suzi  |          893 |
| NULL  |         2166 |
+-------+--------------+
mysql> `SELECT name, AVG(miles) AS driver_avg FROM driver_log`
    -> `GROUP BY name WITH ROLLUP;`
+-------+------------+
| name  | driver_avg |
+-------+------------+
| Ben   |   120.6667 |
| Henry |   182.2000 |
| Suzi  |   446.5000 |
| NULL  |   216.6000 |
+-------+------------+
```

在每种情况下，`name` 列中带有 `NULL` 的输出行代表所有驾驶员的总和或平均值。

如果按多列分组，则 `WITH` `ROLLUP` 会生成多个摘要级别。以下语句显示了每对用户之间发送的邮件消息数量：

```
mysql> `SELECT srcuser, dstuser, COUNT(*)`
    -> `FROM mail GROUP BY srcuser, dstuser;`
+---------+---------+----------+
| srcuser | dstuser | COUNT(*) |
+---------+---------+----------+
| barb    | barb    |        1 |
| barb    | tricia  |        2 |
| gene    | barb    |        2 |
| gene    | gene    |        3 |
| gene    | tricia  |        1 |
| phil    | barb    |        1 |
| phil    | phil    |        2 |
| phil    | tricia  |        2 |
| tricia  | gene    |        1 |
| tricia  | phil    |        1 |
+---------+---------+----------+
```

添加 `WITH` `ROLLUP` 会导致输出包含每个 `srcuser` 值的中间计数（这些是 `dstuser` 列中带有 `NULL` 的行），以及最后的总计数：

```
mysql> `SELECT srcuser, dstuser, COUNT(*)`
    -> `FROM mail GROUP BY srcuser, dstuser WITH ROLLUP;`
+---------+---------+----------+
| srcuser | dstuser | COUNT(*) |
+---------+---------+----------+
| barb    | barb    |        1 |
| barb    | tricia  |        2 |
| barb    | NULL    |        3 |
| gene    | barb    |        2 |
| gene    | gene    |        3 |
| gene    | tricia  |        1 |
| gene    | NULL    |        6 |
| phil    | barb    |        1 |
| phil    | phil    |        2 |
| phil    | tricia  |        2 |
| phil    | NULL    |        5 |
| tricia  | gene    |        1 |
| tricia  | phil    |        1 |
| tricia  | NULL    |        2 |
| NULL    | NULL    |       16 |
+---------+---------+----------+
```

# 10.17 生成包含摘要和列表的报告

## 问题

您希望创建一个报告，显示摘要以及与每个摘要值相关联的行的列表。

## 解决方案

使用两个语句检索不同级别的摘要信息。或者使用编程语言执行部分工作，以便您可以使用单个语句。

## 讨论

假设您想生成类似以下内容的报告：

```
Name: Ben; days on road: 3; miles driven: 362
  date: 2014-07-29, trip length: 131
  date: 2014-07-30, trip length: 152
  date: 2014-08-02, trip length: 79
Name: Henry; days on road: 5; miles driven: 911
  date: 2014-07-26, trip length: 115
  date: 2014-07-27, trip length: 96
  date: 2014-07-29, trip length: 300
  date: 2014-07-30, trip length: 203
  date: 2014-08-01, trip length: 197
Name: Suzi; days on road: 2; miles driven: 893
  date: 2014-07-29, trip length: 391
  date: 2014-08-02, trip length: 502
```

对于 `driver_log` 表中的每个驾驶员，报告显示以下信息：

+   汇总行显示驾驶员姓名、上路天数和驾驶里程数。

+   详细说明了从中计算摘要值的各个行程的日期和里程数的列表。

这种情况是配方 10.16 中讨论的“不同级别的摘要信息”问题的变体。一开始可能看起来不像是这样，因为其中一种信息类型是列表而不是摘要。但这实际上只是“级别零”摘要。这种问题以许多其他形式出现：

+   您有一个列出政党内候选人捐款的数据库。党主席请求打印出每位候选人的捐款数和捐款总额，以及捐赠者姓名和地址列表。

+   您希望为公司演示创建手册，其中总结每个销售区域的总销售额，并在每个区域下的列表中显示该区域每个州的销售额。

这类问题有多种解决方案：

+   运行单独的语句以获取您需要的每个详细级别的信息。（单个查询不会生成每组摘要值和每组的各行列表。）

+   检索构成列表并执行摘要计算的行，以消除摘要语句。

让我们使用每种方法来生成本节开头显示的驾驶员报告。以下实现（使用 Python）使用一个查询来汇总每个驾驶员的天数和英里数，另一个查询获取每个驾驶员的单独行程行：

```
# select total miles per driver and construct a dictionary that
# maps each driver name to days on the road and miles driven
name_map = {}
cursor = conn.cursor()
cursor.execute('''
 SELECT name, COUNT(name), SUM(miles)
 FROM driver_log GROUP BY name
 ''')
for (name, days, miles) in cursor:
  name_map[name] = (days, miles)

# select trips for each driver and print the report, displaying the
# summary entry for each driver prior to the list of trips
cursor.execute('''
 SELECT name, trav_date, miles
 FROM driver_log ORDER BY name, trav_date
 ''')
cur_name = ""
for (name, trav_date, miles) in cursor:
  if cur_name != name:  # new driver; print driver's summary info
    print("Name: %s; days on road: %d; miles driven: %d" %
          (name, name_map[name][0], name_map[name][1]))
    cur_name = name
  print("  date: %s, trip length: %d" % (trav_date, miles))
cursor.close()
```

另一种实现在程序内执行摘要计算，从而减少所需的查询数量。如果您遍历行程列表并计算每个驾驶员的每日计数和里程总数，则只需一个查询即可：

```
# get list of trips for the drivers
cursor = conn.cursor()
cursor.execute('''
 SELECT name, trav_date, miles FROM driver_log
 ORDER BY name, trav_date
 ''')
# fetch rows into data structure because we
# must iterate through them multiple times
rows = cursor.fetchall()
cursor.close()

# iterate through rows once to construct a dictionary that
# maps each driver name to days on the road and miles driven
# (the dictionary entries are lists rather than tuples because
# we need mutable values that can be modified in the loop)
name_map = {}
for (name, trav_date, miles) in rows:
  if name not in name_map: # initialize entry if nonexistent
    name_map[name] = [0, 0]
  name_map[name][0] += 1     # count days
  name_map[name][1] += miles # sum miles

# iterate through rows again to print the report, displaying the
# summary entry for each driver prior to the list of trips
cur_name = ""
for (name, trav_date, miles) in rows:
  if cur_name != name:  # new driver; print driver's summary info
    print("Name: %s; days on road: %d; miles driven: %d" %
          (name, name_map[name][0], name_map[name][1]))
    cur_name = name
  print("  date: %s, trip length: %d" % (trav_date, miles))
```

如果您需要更多级别的摘要信息，则此类问题会变得更加复杂。例如，您可能希望在显示驾驶员摘要和行程日志之前，先显示显示所有驾驶员总英里数的报告：

```
Total miles driven by all drivers combined: 2166

Name: Ben; days on road: 3; miles driven: 362
  date: 2014-07-29, trip length: 131
  date: 2014-07-30, trip length: 152
  date: 2014-08-02, trip length: 79
Name: Henry; days on road: 5; miles driven: 911
  date: 2014-07-26, trip length: 115
  date: 2014-07-27, trip length: 96
  date: 2014-07-29, trip length: 300
  date: 2014-07-30, trip length: 203
  date: 2014-08-01, trip length: 197
Name: Suzi; days on road: 2; miles driven: 893
  date: 2014-07-29, trip length: 391
  date: 2014-08-02, trip length: 502
```

在这种情况下，您需要另一个查询以生成总里程，或者在您的程序中进行另一种计算来计算总体总数。

# 10.18 从临时结果集生成摘要

## 问题

想要生成摘要，但如果不使用临时结果集则无法实现。

## 解决方案

使用通用表达式（CTE）通过`WITH`子句。

## 讨论

我们已经讨论过临时表的情况，它保存来自查询的结果，有助于创建摘要。在这些情况下，我们从查询中引用了临时表，生成了结果摘要。参见 Recipe 10.6 和 Recipe 10.8 进行示例。

对于这样的任务，临时表并非总是最佳解决方案。它们有许多缺点，特别是：

+   您需要维护表格：在重新使用它时删除所有内容，并在完成使用后删除。

+   `CREATE [TEMPORARY] TABLE ... SELECT`语句隐式提交事务，因此当原始表的内容在数据插入临时表后可能发生变化时，不能使用它。您必须先创建表，然后插入数据并在多语句事务中生成摘要。例如，在 Recipe 10.8 中讨论的查找每位司机最长行程可能会得到以下代码：

    ```
    CREATE TEMPORARY TABLE t LIKE driver_log;
    START TRANSACTION;
    INSERT INTO t SELECT name, MAX(miles) AS miles FROM driver_log GROUP BY name;
    SELECT d.name, d.trav_date, d.miles AS 'longest trip'
    FROM driver_log AS d INNER JOIN t USING (name, miles) ORDER BY name;
    COMMIT;
    DROP TABLE t;
    ```

+   优化器的选项较少，无法提高查询的性能。

通用表达式（CTE）允许在查询内部创建命名的临时结果集。CTE 的语法是

```
WITH *`result_name`* AS (SELECT ...)
SELECT ...
```

然后，您可以在以下查询中引用命名的结果，就像它是一个常规表一样。您可以定义多个公共表达式（CTE），在需要时多次引用相同的命名结果。

因此，Recipe 10.17 中的示例展示了每位司机的行程次数和总里程以及行程详细信息，可以通过 CTE 解决。

```
mysql>  `WITH` ![1](img/1.png)
    ->  `trips` `AS` `(``SELECT` `name``,` `trav_date``,` `miles` `FROM` `driver_log``)``,` ![2](img/2.png)
    ->  `summaries` `AS` `(`
    ->      `SELECT` `name``,` `COUNT``(``name``)` `AS` `days_on_road``,` `SUM``(``miles``)` `AS` `miles_driven` ![3](img/3.png)
    ->      `FROM` `driver_log` `GROUP` `BY` `name``)`
    ->  `SELECT` `trips``.``name``,` `days_on_road``,` `miles_driven``,` `trav_date``,` `miles` ![4](img/4.png)
    ->  `FROM` `summaries` `LEFT` `JOIN` `trips` `USING``(``name``)``;`
+-------+--------------+--------------+------------+-------+ | name  | days_on_road | miles_driven | trav_date  | miles |
+-------+--------------+--------------+------------+-------+ | Ben   |            3 |          362 | 2014-08-02 |    79 |  ![5](img/5.png)
| Ben   |            3 |          362 | 2014-07-29 |   131 |
| Ben   |            3 |          362 | 2014-07-30 |   152 |
| Suzi  |            2 |          893 | 2014-08-02 |   502 |
| Suzi  |            2 |          893 | 2014-07-29 |   391 |
| Henry |            5 |          911 | 2014-07-30 |   203 |
| Henry |            5 |          911 | 2014-08-01 |   197 |
| Henry |            5 |          911 | 2014-07-26 |   115 |
| Henry |            5 |          911 | 2014-07-27 |    96 |
| Henry |            5 |          911 | 2014-07-29 |   300 |
+-------+--------------+--------------+------------+-------+ 10 rows in set (0.00 sec)
```

![1](img/#co_nch-sum-sum-with-with_co)

关键词`WITH`开始 CTE。

![2](img/#co_nch-sum-sum-with-trips_co)

将名称`trips`分配给`SELECT`，检索旅行数据。

![3](img/#co_nch-sum-sum-with-summaries_co)

第二个命名`SELECT`生成了每位司机的行程次数和总里程的摘要。

![4](img/#co_nch-sum-sum-with-outer_co)

主查询引用了两个命名结果集，并使用`LEFT JOIN`将它们联接，就像它们是常规表一样。

![5](img/#co_nch-sum-sum-with-result_co)

每行结果包含行程次数、驾驶的总里程数和各个行程的详细信息。

^(1) 结果仅包括实际在表中表示的日期的条目。要生成包括表中日期范围内所有日期条目的摘要，请使用联接填充“缺失”值。参见 Recipe 16.8。
