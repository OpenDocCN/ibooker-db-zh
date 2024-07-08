# 第五章：高级查询

在前两章中，你已经完成了 SQL 数据库查询和修改基本特性的介绍。现在你应该能够创建、修改和删除数据库结构，以及在读取、插入、删除和更新条目时处理数据。在本章和接下来的两章中，我们将深入研究更高级的概念，然后继续探讨更多关于管理和操作导向的内容。当你熟悉使用 MySQL 后，你可以略读这些章节，等到需要时再仔细阅读。

本章教你更多关于查询的内容，提供了解答复杂信息需求的技能。你将学习以下内容：

+   在查询中使用昵称或*别名*，以节省输入并允许一个表在查询中被多次使用。

+   将数据聚合成组，以便发现总和、平均值和计数。

+   以不同方式连接表格。

+   使用嵌套查询。

+   将查询结果保存在变量中，以便在其他查询中重复使用。

# 别名

别名是昵称。它们为你提供了一种简写的方式来表达列、表或函数名称，允许你：

+   编写更短的查询。

+   更清晰地表达你的查询。

+   在单个查询中以两种或更多方式使用同一张表。

+   更轻松地从程序中访问数据。

+   使用特殊类型的嵌套查询，详见 “Nested Queries”。

## 列别名

列别名对于改善查询表达、减少所需键入的字符数以及使得与诸如 Python 或 PHP 等编程语言的工作更加容易非常有用。考虑一个简单且不太实用的例子：

```
mysql> `SELECT` `first_name` `AS` `'First Name'``,` `last_name` `AS` `'Last Name'`
    -> `FROM` `actor` `LIMIT` `5``;`
```

```
+------------+--------------+
| First Name | Last Name    |
+------------+--------------+
| PENELOPE   | GUINESS      |
| NICK       | WAHLBERG     |
| ED         | CHASE        |
| JENNIFER   | DAVIS        |
| JOHNNY     | LOLLOBRIGIDA |
+------------+--------------+
5 rows in set (0.00 sec)
```

列 `first_name` 被别名为 `First Name`，列 `last_name` 被别名为 `Last Name`。你可以看到在输出中，通常的列标题 `first_name` 和 `last_name` 被别名 `First Name` 和 `Last Name` 替代了。这样做的好处是别名可能更有意义，至少对人类更易读。除此之外，它并不是非常有用，但它确实说明了一个想法，即对于一列，你可以添加关键词 `AS`，然后是一个代表你想让列被称为的字符串。指定 `AS` 关键词并非必需，但可以使事情更清晰。

###### 注意

我们将在本章中广泛使用 `LIMIT` 子句，否则几乎每个输出都会非常冗长。有时我们会明确提到它，有时则不会。你可以尝试自己从我们给出的查询中删除 `LIMIT` 来进行实验。关于 `LIMIT` 子句的更多信息可在 “The LIMIT Clause” 中找到。

现在让我们看看列别名如何发挥作用。以下是一个使用 MySQL 函数和 `ORDER BY` 子句的示例：

```
mysql> `SELECT` `CONCAT``(``first_name``,` `' '``,` `last_name``,` `' played in '``,` `title``)` `AS` `movie`
    -> `FROM` `actor` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `JOIN` `film` `USING` `(``film_id``)`
    -> `ORDER` `BY` `movie` `LIMIT` `20``;`
```

```
+--------------------------------------------+
| movie                                      |
+--------------------------------------------+
| ADAM GRANT played in ANNIE IDENTITY        |
| ADAM GRANT played in BALLROOM MOCKINGBIRD  |
| ...                                        |
| ADAM GRANT played in TWISTED PIRATES       |
| ADAM GRANT played in WANDA CHAMBER         |
| ADAM HOPPER played in BLINDNESS GUN        |
| ADAM HOPPER played in BLOOD ARGONAUTS      |
+--------------------------------------------+
20 rows in set (0.03 sec)
```

MySQL 函数`CONCAT()` *连接* 它的参数——在本例中是`first_name`、一个包含空格的常量字符串、`last_name`、常量字符串`played in`和`title`——以生成诸如`ZERO CAGE played in CANYON STOCK`之类的输出。我们给函数添加了一个别名`AS movie`，这样我们可以在整个查询中轻松地引用它作为`movie`。您可以看到，我们在`ORDER BY`子句中这样做时，请求 MySQL 按升序`movie`值对输出进行排序。这比未使用别名的替代方法要好得多，后者要求您再次编写`CONCAT()`函数：

```
mysql> `SELECT` `CONCAT``(``first_name``,` `' '``,` `last_name``,` `' played in '``,` `title``)` `AS` `movie`
    -> `FROM` `actor` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `JOIN` `film` `USING` `(``film_id``)`
    -> `ORDER` `BY` `CONCAT``(``first_name``,` `' '``,` `last_name``,` `' played in '``,` `title``)`
    -> `LIMIT` `20``;`
```

```
+--------------------------------------------+
| movie                                      |
+--------------------------------------------+
| ADAM GRANT played in ANNIE IDENTITY        |
| ADAM GRANT played in BALLROOM MOCKINGBIRD  |
| ...                                        |
| ADAM GRANT played in TWISTED PIRATES       |
| ADAM GRANT played in WANDA CHAMBER         |
| ADAM HOPPER played in BLINDNESS GUN        |
| ADAM HOPPER played in BLOOD ARGONAUTS      |
+--------------------------------------------+
20 rows in set (0.03 sec)
```

另一种方式不方便，更糟糕的是，您可能会在`ORDER BY`子句中打字错误，导致结果与您的预期不同。（请注意，我们在第一行使用了`AS movie`，以便显示的列具有标签`movie`。）

在可以使用列别名的地方有一些限制。您不能在`WHERE`子句中使用它们，也不能在我们稍后在本章中讨论的`USING`和`ON`子句中使用它们。这意味着您不能编写如下查询：

```
mysql> `SELECT` `first_name` `AS` `name` `FROM` `actor` `WHERE` `name` `=` `'ZERO CAGE'``;`
```

```
ERROR 1054 (42S22): Unknown column 'name' in 'where clause'
```

由于 MySQL 在执行`WHERE`子句之前并不总是知道列值，所以您不能这样做。但是，您可以在`ORDER BY`子句中使用列别名，在稍后在本章中讨论的`GROUP BY`和`HAVING`子句中也可以使用列别名。

`AS`关键字是可选的，正如我们前面提到的那样。因此，以下两个查询是等价的：

```
mysql> `SELECT` `actor_id` `AS` `id` `FROM` `actor` `WHERE` `first_name` `=` `'ZERO'``;`
```

```
+----+
| id |
+----+
| 11 |
+----+
1 row in set (0.00 sec)
```

```
mysql> `SELECT` `actor_id` `id` `FROM` `actor` `WHERE` `first_name` `=` `'ZERO'``;`
```

```
+----+
| id |
+----+
| 11 |
+----+
1 row in set (0.00 sec)
```

我们建议使用`AS`关键字，因为它有助于清晰地区分带别名的列，特别是在从逗号分隔的列列表中选择多列的情况下。

别名的名称有一些限制。它们最多可以为 255 个字符，并且可以包含任何字符。别名不总是需要加引号，并且遵循与表名和列名相同的规则，我们在第四章中进行了描述。如果别名是一个单词并且不包含特殊符号（例如破折号、加号或空格），并且不是关键字（如`USE`），那么您不需要在其周围加引号。否则，您需要引用别名，可以使用双引号、单引号或反引号。我们建议使用小写字母数字字符串作为别名，并使用一致的字符选择（例如下划线）来分隔单词。别名在所有平台上都不区分大小写。

## 表别名

表别名和列别名一样有用，但有时是表达查询的唯一方式。本节将向您展示如何使用表别名，而“嵌套查询”将向您展示一些其他必不可少的表别名示例。

下面是一个基本的表别名示例，向您展示如何节省输入：

```
mysql> `SELECT` `ac``.``actor_id``,` `ac``.``first_name``,` `ac``.``last_name``,` `fl``.``title` `FROM`
    -> `actor` `AS` `ac` `INNER` `JOIN` `film_actor` `AS` `fla` `USING` `(``actor_id``)`
    -> `INNER` `JOIN` `film` `AS` `fl` `USING` `(``film_id``)`
    -> `WHERE` `fl``.``title` `=` `'AFFAIR PREJUDICE'``;`
```

```
+----------+------------+-----------+------------------+
| actor_id | first_name | last_name | title            |
+----------+------------+-----------+------------------+
|       41 | JODIE      | DEGENERES | AFFAIR PREJUDICE |
|       81 | SCARLETT   | DAMON     | AFFAIR PREJUDICE |
|       88 | KENNETH    | PESCI     | AFFAIR PREJUDICE |
|      147 | FAY        | WINSLET   | AFFAIR PREJUDICE |
|      162 | OPRAH      | KILMER    | AFFAIR PREJUDICE |
+----------+------------+-----------+------------------+
5 rows in set (0.00 sec)
```

您可以看到，`film`表和`actor`表分别使用`AS`关键字别名为`fl`和`ac`。这允许您更紧凑地表示列名称，例如`fl.title`。请注意，您还可以在`WHERE`子句中使用表别名；与列别名不同，对查询中可以使用表别名的位置没有限制。从我们的例子中，您可以看到我们在`FROM`中定义`SELECT`之前引用了表别名。但是，表别名有一个陷阱：如果一个别名已用于一个表，则无法在不使用其新别名的情况下引用该表。例如，下面的语句将出错，就像我们在`SELECT`子句中提到`film`时那样：

```
mysql> `SELECT` `ac``.``actor_id``,` `ac``.``first_name``,` `ac``.``last_name``,` `fl``.``title` `FROM`
    -> `actor` `AS` `ac` `INNER` `JOIN` `film_actor` `AS` `fla` `USING` `(``actor_id``)`
    -> `INNER` `JOIN` `film` `AS` `fl` `USING` `(``film_id``)`
    -> `WHERE` `film``.``title` `=` `'AFFAIR PREJUDICE'``;`
```

```
ERROR 1054 (42S22): Unknown column 'film.title' in 'where clause'
```

与列别名一样，`AS`关键字是可选的。这意味着：

```
actor AS ac INNER JOIN film_actor AS fla
```

等同于

```
actor ac INNER JOIN film_actor fla
```

我们更喜欢`AS`风格，因为对于查看您的查询的任何人来说，它比替代方案更清晰。表别名名称的长度和内容限制与列别名相同，并且我们在选择它们时的建议也相同。

如本节介绍中所讨论的，表别名允许您编写否则难以表达的查询。考虑一个例子：假设您想知道我们收藏的两部或更多电影是否具有相同的标题，如果是这样，这些电影是什么。让我们考虑基本要求：您想知道是否有两部电影具有相同的名称。为了做到这一点，您可能会尝试像这样的查询：

```
mysql> `SELECT` `*` `FROM` `film` `WHERE` `title` `=` `title``;`
```

但这毫无意义 —— 每部电影都与自己有相同的标题，所以查询只是将所有电影作为输出：

```
+---------+------------------...
| film_id | title            ...
+---------+------------------...
|       1 | ACADEMY DINOSAUR ...
|       2 | ACE GOLDFINGER   ...
|       3 | ADAPTATION HOLES ...
|     ...                    ...
|    1000 | ZORRO ARK        ...
+---------+------------------...
1000 rows in set (0.01 sec)
```

你真正想知道的是`film`表中的两部不同电影是否有相同的名称。但如何在一个查询中做到这一点？答案是给表指定两个不同的别名；然后检查第一个别名表中的一行是否与第二个别名表中的一行匹配：

```
mysql> `SELECT` `m1``.``film_id``,` `m2``.``title`
    -> `FROM` `film` `AS` `m1``,` `film` `AS` `m2`
    -> `WHERE` `m1``.``title` `=` `m2``.``title``;`
```

```
+---------+-------------------+
| film_id | title             |
+---------+-------------------+
|       1 | ACADEMY DINOSAUR  |
|       2 | ACE GOLDFINGER    |
|       3 | ADAPTATION HOLES  |
|     ...                     |
|    1000 | ZORRO ARK         |
+---------+-------------------+
1000 rows in set (0.02 sec)
```

但仍然不起作用！我们得到了所有 1,000 部电影作为答案。原因是因为每部电影都与自己匹配，因为它在两个别名表中都出现了。

要使查询起作用，我们需要确保来自一个别名表的电影不与另一个别名表中的自身匹配。做到这一点的方法是指定每个表中的电影不应具有相同的 ID：

```
mysql> `SELECT` `m1``.``film_id``,` `m2``.``title`
    -> `FROM` `film` `AS` `m1``,` `film` `AS` `m2`
    -> `WHERE` `m1``.``title` `=` `m2``.``title`
    -> `AND` `m1``.``film_id` `<``>` `m2``.``film_id``;`
```

```
Empty set (0.00 sec)
```

您现在可以看到数据库中没有两部电影具有相同的名称。附加的`AND m1.film_id != m2.film_id`阻止报告电影 ID 在两个表中相同的答案。

表别名在使用`EXISTS`和`ON`子句的嵌套查询中也很有用。在本章稍后介绍嵌套查询时，我们将为您展示示例。

# 聚合数据

聚合函数允许您发现一组行的属性。您可以使用它们来查找表中有多少行，表中有多少行共享某个属性（例如具有相同的姓名或出生日期），查找平均值（例如 11 月份的平均温度），或查找满足某些条件的行的最大或最小值（例如找到 8 月份最冷的一天）。

本节解释了`GROUP BY`和`HAVING`子句，这是用于聚合的两个最常用的 SQL 语句。但首先它解释了`DISTINCT`子句，该子句用于报告查询输出的唯一结果。当未指定`DISTINCT`或`GROUP BY`子句时，仍然可以使用我们在本节中描述的聚合函数处理返回的原始数据。

## DISTINCT 子句

在开始讨论聚合函数之前，我们将专注于`DISTINCT`子句。这实际上不是一个聚合函数，而更像是一个后处理过滤器，允许您去除重复项。我们将其添加到本节中，因为与聚合函数一样，它关注的是从查询输出中选择示例，而不是处理单个行。

一个例子是理解`DISTINCT`的最佳方式。考虑这个查询：

```
mysql> `SELECT` `DISTINCT` `first_name`
    -> `FROM` `actor` `JOIN` `film_actor` `USING` `(``actor_id``)``;`
```

```
+-------------+
| first_name  |
+-------------+
| PENELOPE    |
| NICK        |
| ...         |
| GREGORY     |
| JOHN        |
| BELA        |
| THORA       |
+-------------+
128 rows in set (0.00 sec)
```

该查询找到了我们数据库中列出的所有参与电影的演员的所有名字，并报告每个名字的一个示例。如果删除`DISTINCT`子句，您将得到我们数据库中每部电影中每个角色的一个输出行，或者说有 5,462 行。这是很多输出，所以我们将其限制为五行，但您可以立即看到重复的名字：

```
mysql> `SELECT` `first_name`
    -> `FROM` `actor` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `LIMIT` `5``;`
```

```
+------------+
| first_name |
+------------+
| PENELOPE   |
| PENELOPE   |
| PENELOPE   |
| PENELOPE   |
| PENELOPE   |
+------------+
5 rows in set (0.00 sec)
```

因此，`DISTINCT`子句帮助您得到一个摘要。

`DISTINCT`子句应用于查询输出，并删除在查询中选择的输出列中具有相同值的行。如果重新构造前面的查询以输出`first_name`和`last_name`（但不更改`JOIN`子句并仍然使用`DISTINCT`），您将在输出中得到 199 行（这就是为什么我们使用姓氏）：

```
mysql> `SELECT` `DISTINCT` `first_name``,` `last_name`
    -> `FROM` `actor` `JOIN` `film_actor` `USING` `(``actor_id``)``;`
```

```
+-------------+--------------+
| first_name  | last_name    |
+-------------+--------------+
| PENELOPE    | GUINESS      |
| NICK        | WAHLBERG     |
| ...                        |
| JULIA       | FAWCETT      |
| THORA       | TEMPLE       |
+-------------+--------------+
199 rows in set (0.00 sec)
```

不幸的是，即使添加了姓氏，人们的名字仍然不适合作为一个糟糕的唯一键。在`sakila`数据库的`actor`表中有 200 行数据，我们遗漏了其中一个。您应该记住这个问题，因为不加区分地使用`DISTINCT`可能导致查询结果不正确。

要去除重复项，MySQL 需要对输出进行排序。如果可用的索引与所需排序的顺序相同，或者数据本身按照有用的顺序排列，这个过程的开销非常小。然而，对于大表而言，如果没有一种简单的方式访问正确顺序的数据，排序可能会非常慢。在处理大型数据集时，您应该谨慎使用`DISTINCT`（以及其他聚合函数）。如果您确实使用了它，可以使用第七章中讨论的`EXPLAIN`语句来检查其行为。

## GROUP BY 子句

`GROUP BY` 子句根据聚合的目的分组输出数据。特别是，这使我们能够在我们的投影（即`SELECT`子句的内容）包含除聚合函数之外的列时，对我们的数据使用聚合函数（在“聚合函数”中已讨论）。

让我们看看一些 `GROUP BY` 示例，这将演示它可以用于什么。在其最基本的形式中，当我们在`GROUP BY`中列出我们`SELECT`的每一列时，我们最终得到一个`DISTINCT`等效项。我们已经确定名字不是演员的唯一标识符：

```
mysql> `SELECT` `first_name` `FROM` `actor`
    -> `WHERE` `first_name` `IN` `(``'GENE'``,` `'MERYL'``)``;`
```

```
+------------+
| first_name |
+------------+
| GENE       |
| GENE       |
| MERYL      |
| GENE       |
| MERYL      |
+------------+
5 rows in set (0.00 sec)
```

我们可以告诉 MySQL 通过给定的列分组输出以去除重复项。在这种情况下，我们只选择了一列，因此让我们使用它：

```
mysql> `SELECT` `first_name` `FROM` `actor`
    -> `WHERE` `first_name` `IN` `(``'GENE'``,` `'MERYL'``)`
    -> `GROUP` `BY` `first_name``;`
```

```
+------------+
| first_name |
+------------+
| GENE       |
| MERYL      |
+------------+
2 rows in set (0.00 sec)
```

您可以看到，原始的五行被合并或更准确地说是分组为仅两行结果。这并不是很有帮助，因为`DISTINCT`可以做同样的事情。然而值得一提的是，这并不总是适用。`DISTINCT` 和 `GROUP BY` 在查询执行的不同阶段进行评估和执行，因此即使有时效果相似，您也不应混淆它们。

根据 SQL 标准，`SELECT` 子句中投影的每一列如果不是聚合函数的一部分，应该在 `GROUP BY` 子句中列出。唯一可以违反此规则的情况是当结果组每个仅有一行时。仔细想想，这是合理的：如果你从`actor`表中选择`first_name`和`last_name`，并且仅按`first_name`分组，数据库应该如何处理？它不能输出多行具有相同的姓，因为那违反了分组规则，但可能有多个姓氏对应一个给定的名字。

长期以来，MySQL 通过允许您基于`SELECT`中定义的少列来`GROUP BY` 扩展了标准。它如何处理额外的列呢？嗯，它以不确定的方式输出一些值。例如，当您按姓但不按名分组时，您可能会得到`GENE, WILLIS`和`GENE, HOPKINS`中的任一行。这是一种非标准和危险的行为。想象一下，一年来，您得到了`Hopkins`，就好像结果是按字母顺序排列的，而且依赖于此，但然后表被重新组织，顺序发生了变化。我们坚信 SQL 标准正确地限制了这样的行为，以避免不可预测性。

还要注意，虽然 `SELECT` 中的每一列必须在 `GROUP BY` 或聚合函数中使用，但可以 `GROUP BY` 不在 `SELECT` 中的列。稍后您将看到一些示例。

现在让我们构建一个更有用的例子。演员通常在他们的职业生涯中参与许多电影。我们可能想知道某个特定演员演过多少部电影，或者为我们所知的每位演员进行计算，并通过生产率得到一个评级。首先，我们可以使用迄今为止学到的技术，在`actor`和`film_actor`表之间执行`INNER JOIN`。我们不需要`film`表，因为我们不关心电影本身的任何细节。然后我们可以按演员姓名对输出进行排序，以便更容易计算我们想要的内容：

```
mysql> `SELECT` `first_name``,` `last_name``,` `film_id`
    -> `FROM` `actor` `INNER` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `ORDER` `BY` `first_name``,` `last_name` `LIMIT` `20``;`
```

```
+------------+-----------+---------+
| first_name | last_name | film_id |
+------------+-----------+---------+
| ADAM       | GRANT     |      26 |
| ADAM       | GRANT     |      52 |
| ADAM       | GRANT     |     233 |
| ADAM       | GRANT     |     317 |
| ADAM       | GRANT     |     359 |
| ADAM       | GRANT     |     362 |
| ADAM       | GRANT     |     385 |
| ADAM       | GRANT     |     399 |
| ADAM       | GRANT     |     450 |
| ADAM       | GRANT     |     532 |
| ADAM       | GRANT     |     560 |
| ADAM       | GRANT     |     574 |
| ADAM       | GRANT     |     638 |
| ADAM       | GRANT     |     773 |
| ADAM       | GRANT     |     833 |
| ADAM       | GRANT     |     874 |
| ADAM       | GRANT     |     918 |
| ADAM       | GRANT     |     956 |
| ADAM       | HOPPER    |      81 |
| ADAM       | HOPPER    |      82 |
+------------+-----------+---------+
20 rows in set (0.01 sec)
```

通过逐一列出列表，很容易计算出每个演员有多少部电影，至少亚当·格兰特是这样。但是，如果没有`LIMIT`，查询将返回 5,462 行不同的结果，手动计算我们的计数会花费很多时间。`GROUP BY`子句可以通过按演员分组来自动化这一过程；然后我们可以使用`COUNT()`函数来计算每个组中的电影数量。最后，我们可以使用`ORDER BY`和`LIMIT`来获取拥有最多电影出演次数的前 10 名演员。以下是执行我们所需操作的查询：

```
mysql> `SELECT` `first_name``,` `last_name``,` `COUNT``(``film_id``)` `AS` `num_films` `FROM`
    -> `actor` `INNER` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `GROUP` `BY` `first_name``,` `last_name`
    -> `ORDER` `BY` `num_films` `DESC` `LIMIT` `5``;`
```

```
+------------+-------------+-----------+
| first_name | last_name   | num_films |
+------------+-------------+-----------+
| SUSAN      | DAVIS       |        54 |
| GINA       | DEGENERES   |        42 |
| WALTER     | TORN        |        41 |
| MARY       | KEITEL      |        40 |
| MATTHEW    | CARREY      |        39 |
| SANDRA     | KILMER      |        37 |
| SCARLETT   | DAMON       |        36 |
| VAL        | BOLGER      |        35 |
| ANGELA     | WITHERSPOON |        35 |
| UMA        | WOOD        |        35 |
+------------+-------------+-----------+
10 rows in set (0.01 sec)
```

你可以看到，我们请求的输出是`first_name, last_name, COUNT(film_id) as num_films`，这告诉我们确切想要知道的内容。我们按照`first_name`和`last_name`列对数据进行分组，在此过程中运行`COUNT()`聚合函数。对于上一个查询中得到的每个“桶”行，我们现在只得到一行，尽管提供了我们想要的信息。注意我们如何结合使用`GROUP BY`和`ORDER BY`来获得所需的顺序：按照电影数量，从多到少。`GROUP BY`不能保证顺序，只能进行分组。最后，我们通过`LIMIT`将输出限制为显示我们最多产的前 10 名演员，否则我们将得到 199 行输出。

让我们进一步考虑这个查询。我们将从`GROUP BY`子句开始。这告诉我们如何将行放在一起形成组：在这个例子中，我们告诉 MySQL 按`first_name, last_name`来分组行。结果是相同姓名的演员行成一个集群或“桶”，也就是说，每个不同的姓名成为一个组。一旦行被分组，它们在查询的其余部分被视为一行。所以，例如，当我们写`SELECT first_name, last_name`时，我们对每个组只得到一行。这与`DISTINCT`完全相同，正如我们已经讨论过的。`COUNT()`函数告诉我们组的属性。更具体地说，它告诉我们每个组中形成的行数；您可以计算组中的任何列，您将得到相同的答案，因此`COUNT(film_id)`几乎总是与`COUNT(*)`或`COUNT(first_name)`相同。（有关为什么我们说“几乎”，请参见“聚合函数”了解更多详细信息。）我们也可以只做`COUNT(1)`，或者实际指定任何文字。把这看作是从表中进行`SELECT 1`，然后计算结果。对于表中的每一行，都将输出一个值为 1 的值，并且`COUNT()`进行计数。一个例外是`NULL`：虽然指定`COUNT(NULL)`是完全可接受和合法的，但结果总是零，因为`COUNT()`丢弃`NULL`值。当然，您可以为`COUNT()`列使用列别名。

让我们尝试另一个例子。假设您想知道每部电影中有多少不同的演员参演，以及电影名称及其类别，并获取最大组的五部电影。这里是查询：

```
mysql> `SELECT` `title``,` `name` `AS` `category_name``,` `COUNT``(``*``)` `AS` `cnt`
    -> `FROM` `film` `INNER` `JOIN` `film_actor` `USING` `(``film_id``)`
    -> `INNER` `JOIN` `film_category` `USING` `(``film_id``)`
    -> `INNER` `JOIN` `category` `USING` `(``category_id``)`
    -> `GROUP` `BY` `film_id``,` `category_id`
    -> `ORDER` `BY` `cnt` `DESC` `LIMIT` `5``;`
```

```
+------------------+---------------+-----+
| title            | category_name | cnt |
+------------------+---------------+-----+
| LAMBS CINCINATTI | Games         |  15 |
| CRAZY HOME       | Comedy        |  13 |
| CHITTY LOCK      | Drama         |  13 |
| RANDOM GO        | Sci-Fi        |  13 |
| DRACULA CRYSTAL  | Classics      |  13 |
+------------------+---------------+-----+
5 rows in set (0.03 sec)
```

在我们讨论新内容之前，请考虑查询的一般功能。我们使用它们的标识列：`film`、`film_actor`、`film_category` 和 `category`，通过`INNER JOIN`将四个表连接在一起。暂且不考虑聚合，这个查询的输出是每个电影和演员组合的一行。

`GROUP BY`子句将行组合成集群。在这个查询中，我们希望将电影与它们的类别分组在一起。`GROUP BY`子句使用`film_id`和`category_id`来完成这一点。您可以从任何三个表中使用`film_id`列；对于此目的，`film.film_id`、`film_actor.film_id`和`film_category.film_id`是相同的。使用哪一个并不重要；`INNER JOIN`确保它们匹配。同样适用于`category_id`。

正如前面提到的，即使需要在`GROUP BY`中列出每个非聚合列，您也可以在`SELECT`之外的列上`GROUP BY`。在前面的示例查询中，我们使用`COUNT()`函数告诉我们每个组中有多少行。例如，您可以看到`COUNT(*)`告诉我们在喜剧*CRAZY HOME*中有 13 位演员。再次强调，在查询中计数的列无关紧要：例如，`COUNT(*)`与`COUNT(film.film_id)`或`COUNT(category.name)`具有相同的效果。

然后，我们按`COUNT(*)`列的别名`cnt`降序排序输出，并选择前五行。请注意，有多行的`cnt`等于 13。事实上，数据库中甚至有六个这样的行，使得这种排序有些不公平，因为具有相同演员数量的电影将随机排序。你可以添加另一个列到`ORDER BY`子句，比如`title`，使得排序更可预测。

让我们再试一个例子。`sakila`数据库不仅仅是关于电影和演员：毕竟它是基于电影租借的。我们还有顾客信息，包括他们租借的电影数据。假设我们想知道哪些顾客倾向于租借同一类别的电影。例如，我们可能想根据一个人是否喜欢不同的电影类别或者大部分时间都喜欢某一类别来调整我们的广告。我们需要仔细考虑我们的分组方式：我们不希望按电影分组，因为那只会给我们提供顾客租借它的次数。最终的查询相当复杂，尽管它仍然围绕`INNER JOIN`和`GROUP BY`进行。

```
mysql> `SELECT` `email``,` `name` `AS` `category_name``,` `COUNT``(``category_id``)` `AS` `cnt`
    -> `FROM` `customer` `cs` `INNER` `JOIN` `rental` `USING` `(``customer_id``)`
    -> `INNER` `JOIN` `inventory` `USING` `(``inventory_id``)`
    -> `INNER` `JOIN` `film_category` `USING` `(``film_id``)`
    -> `INNER` `JOIN` `category` `cat` `USING` `(``category_id``)`
    -> `GROUP` `BY` `1``,` `2`
    -> `ORDER` `BY` `3` `DESC` `LIMIT` `5``;`
```

```
+----------------------------------+---------------+-----+
| email                            | category_name | cnt |
+----------------------------------+---------------+-----+
| WESLEY.BULL@sakilacustomer.org   | Games         |   9 |
| ALMA.AUSTIN@sakilacustomer.org   | Animation     |   8 |
| KARL.SEAL@sakilacustomer.org     | Animation     |   8 |
| LYDIA.BURKE@sakilacustomer.org   | Documentary   |   8 |
| NATHAN.RUNYON@sakilacustomer.org | Animation     |   7 |
+----------------------------------+---------------+-----+
5 rows in set (0.08 sec)
```

这些客户反复从同一类别租借电影。我们不知道的是，他们是否曾多次租借了同一部电影，还是仅仅是类别内的不同电影。`GROUP BY`子句隐藏了细节。同样，我们使用`COUNT(*)`来统计组中的行数，你可以看到查询中第 2 到第 5 行之间的`INNER JOIN`。

这个查询的有趣之处在于，我们没有明确指定`GROUP BY`或`ORDER BY`子句的列名。相反，我们使用了列在`SELECT`子句中出现的位置编号（从 1 开始）。这种技术节省了输入时间，但如果稍后决定在`SELECT`中添加另一列，可能会导致顺序混乱。

与`DISTINCT`类似，`GROUP BY`也存在一些风险需要提及。考虑以下查询：

```
mysql> `SELECT` `COUNT``(``*``)` `FROM` `actor` `GROUP` `BY` `first_name``,` `last_name``;`
```

```
+----------+
| COUNT(*) |
+----------+
|        1 |
|        1 |
|      ... |
|        1 |
|        1 |
+----------+
199 rows in set (0.00 sec)
```

看起来很简单，并且它输出了给定名和姓在`actor`表中找到的组合出现次数。你可能会认为它只会输出 199 行数字`1`。然而，如果我们对`actor`表进行`COUNT(*)`，我们会得到 200 行。问题在哪里？显然，有两个演员具有相同的名和姓。这些情况偶尔会发生，你必须注意这一点。当你基于不构成唯一标识符的列进行分组时，可能会意外地将无关的行分组在一起，导致数据误导。为了找出重复项，我们可以修改一个我们在“表别名”中构建的查询，以查找具有相同名称的电影：

```
mysql> `SELECT` `a1``.``actor_id``,` `a1``.``first_name``,` `a1``.``last_name`
    -> `FROM` `actor` `AS` `a1``,` `actor` `AS` `a2`
    -> `WHERE` `a1``.``first_name` `=` `a2``.``first_name`
    -> `AND` `a1``.``last_name` `=` `a2``.``last_name`
    -> `AND` `a1``.``actor_id` `<``>` `a2``.``actor_id``;`
```

```
+----------+------------+-----------+
| actor_id | first_name | last_name |
+----------+------------+-----------+
|      101 | SUSAN      | DAVIS     |
|      110 | SUSAN      | DAVIS     |
+----------+------------+-----------+
2 rows in set (0.00 sec)
```

在我们结束本节之前，让我们再次触及一下 MySQL 如何在 `GROUP BY` 子句周围扩展 SQL 标准。在 MySQL 5.7 之前，默认情况下可以在 `GROUP BY` 子句中指定不完整的列列表，正如我们所解释的那样，这导致非分组依赖列中的随机行输出到组内。出于支持旧版软件的原因，MySQL 5.7 和 MySQL 8.0 都继续提供这种行为，尽管必须显式启用。该行为由 `ONLY_FULL_GROUP_BY` SQL 模式控制，默认设置为启用。如果你发现自己需要迁移依赖于旧版 `GROUP BY` 行为的程序，我们建议不要改变 SQL 模式。通常有两种方法来解决这个问题。第一种是了解查询逻辑是否真的需要不完整的分组，这种情况很少见。第二种是通过使用聚合函数如 `MIN()` 或 `MAX()`，或者特殊的 `ANY_VALUE()` 聚合函数来支持非分组列的随机数据行为，后者只是从组内产生一个随机值。我们接下来会更仔细地看一下聚合函数。

### 聚合函数

我们已经看到 `COUNT()` 函数如何用于告诉组内有多少行。在这里，我们将介绍一些其他常用于探索聚合行属性的函数。我们还会稍微扩展一下 `COUNT()` 的使用方式，因为它经常被用到：

`COUNT()`

返回行的数量或列中值的数量。记得我们提到过 `COUNT(*)` 几乎总是等同于 `COUNT(<column>)`。问题在于 `NULL`。`COUNT(*)` 会对返回的行数进行计数，无论这些行中的列是否为 `NULL`。然而，当你使用 `COUNT(<column>)` 时，只会计算非 `NULL` 值。例如，在 `sakila` 数据库中，客户的电子邮件地址可能为空，我们可以观察其影响：

```
mysql> `SELECT` `COUNT``(``*``)` `FROM` `customer``;`
```

```
+----------+
| count(*) |
+----------+
|      599 |
+----------+
1 row in set (0.00 sec)
```

```
mysql> `SELECT` `COUNT``(``email``)` `FROM` `customer``;`
```

```
+--------------+
| count(email) |
+--------------+
|          598 |
+--------------+
1 row in set (0.00 sec)
```

此外，我们还应该补充说明 `COUNT()` 可以在内部使用 `DISTINCT` 子句运行，例如 `COUNT(DISTINCT <column>)`，在这种情况下返回不同值的数量，而不是所有值的数量。

`AVG()`

返回组内指定列中值的平均值（均值）。例如，你可以用它来找到一个城市房屋的平均成本，当房屋按城市分组时。

```
SELECT AVG(cost) FROM house_prices GROUP BY city;
```

`MAX()`

返回组内行中的最大值。例如，你可以用它来找到一个月中最热的一天，当行按月份分组时。

`MIN()`

返回组内行中的最小值。例如，你可以用它来找到一个班级中年龄最小的学生，当行按班级分组时。

`STD()`，`STDDEV()` 和 `STDDEV_POP()`

返回组中行的标准差。例如，您可以使用这些来了解按大学课程分组时测试分数的分布。这三个都是同义词。`STD()`是 MySQL 扩展，`STDDEV()`是为了与 Oracle 兼容而添加的，`STDDEV_POP()`是 SQL 标准函数。

`SUM()`

返回组中行的值的总和。例如，您可以使用它来计算按月分组时的销售金额。

与`GROUP BY`一起使用的其他函数也可用，但它们的使用频率不如我们在此介绍的函数高。您可以在 MySQL 参考手册的[聚合函数描述](https://oreil.ly/QSZst)部分找到更多详细信息。

## `HAVING`子句

现在您已经熟悉了`GROUP BY`子句，它允许您对数据进行排序和分组。您应该能够使用它来查找计数、平均值、最小值和最大值。本节展示了如何使用`HAVING`子句对`GROUP BY`操作中的行进行额外控制。

假设您想知道我们数据库中有多少受欢迎的演员。您决定将参与至少 40 部电影的演员定义为受欢迎的演员。在前一节中，我们尝试了一个几乎相同的查询，但没有受欢迎度限制。我们还发现，当我们按名字分组演员时，我们丢失了一条记录，因此我们将添加对`actor_id`列的分组，我们知道它是唯一的。以下是新查询，带有额外的`HAVING`子句，添加了约束条件：

```
mysql> `SELECT` `first_name``,` `last_name``,` `COUNT``(``film_id``)`
    -> `FROM` `actor` `INNER` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `GROUP` `BY` `actor_id``,` `first_name``,` `last_name`
    -> `HAVING` `COUNT``(``film_id``)` `>` `40`
    -> `ORDER` `BY` `COUNT``(``film_id``)` `DESC``;`
```

```
+------------+-----------+----------------+
| first_name | last_name | COUNT(film_id) |
+------------+-----------+----------------+
| GINA       | DEGENERES |             42 |
| WALTER     | TORN      |             41 |
+------------+-----------+----------------+
2 rows in set (0.01 sec)
```

您可以看到，只有两位演员符合新标准。

`HAVING`子句必须包含在`SELECT`子句中列出的表达式或列。在本例中，我们使用了`HAVING COUNT(film_id) >= 40`，您可以看到`COUNT(film_id)`是`SELECT`子句的一部分。通常，`HAVING`子句中的表达式使用聚合函数，如`COUNT()`、`SUM()`、`MIN()`或`MAX()`。如果您发现自己想要编写一个`HAVING`子句，使用的列或表达式不在`SELECT`子句中，那么很可能应该使用`WHERE`子句。`HAVING`子句仅用于决定如何形成每个组或聚类，而不是用于选择输出中的行。稍后我们将展示一个示例，说明何时不应使用`HAVING`。

让我们再试一个例子。假设您想要一个列表，列出被租超过 30 次的前 5 部电影及其租赁次数，按人气逆序排序。以下是您将使用的查询：

```
mysql> `SELECT` `title``,` `COUNT``(``rental_id``)` `AS` `num_rented` `FROM`
    -> `film` `INNER` `JOIN` `inventory` `USING` `(``film_id``)`
    -> `INNER` `JOIN` `rental` `USING` `(``inventory_id``)`
    -> `GROUP` `BY` `title`
    -> `HAVING` `num_rented` `>` `30`
    -> `ORDER` `BY` `num_rented` `DESC` `LIMIT` `5``;`
```

```
+--------------------+------------+
| title              | num_rented |
+--------------------+------------+
| BUCKET BROTHERHOOD |         34 |
| ROCKETEER MOTHER   |         33 |
| FORWARD TEMPLE     |         32 |
| GRIT CLOCKWORK     |         32 |
| JUGGLER HARDLY     |         32 |
+--------------------+------------+
5 rows in set (0.04 sec)
```

您可以再次看到，在`SELECT`和`HAVING`子句中都使用了表达式`COUNT()`。不过，这次我们将`COUNT(rental_id)`函数别名为`num_rented`，并在`HAVING`和`ORDER BY`子句中使用了该别名。

现在让我们考虑一个不应该使用`HAVING`的例子。你想知道一个特定演员出演了多少部电影。以下是你不应该使用的查询：

```
mysql> `SELECT` `first_name``,` `last_name``,` `COUNT``(``film_id``)` `AS` `film_cnt` `FROM`
    -> `actor` `INNER` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `GROUP` `BY` `actor_id``,` `first_name``,` `last_name`
    -> `HAVING` `first_name` `=` `'EMILY'` `AND` `last_name` `=` `'DEE'``;`
```

```
+------------+-----------+----------+
| first_name | last_name | film_cnt |
+------------+-----------+----------+
| EMILY      | DEE       |       14 |
+------------+-----------+----------+
1 row in set (0.02 sec)
```

它得到了正确的答案，但是用了错误的方式——对于大量数据来说，这种方式要慢得多。这不是编写查询的正确方式，因为`HAVING`子句并没有被用来决定哪些行应该形成每个组，而是被错误地用来过滤要显示的答案。对于这个查询，我们应该真正使用`WHERE`子句，如下所示：

```
mysql> `SELECT` `first_name``,` `last_name``,` `COUNT``(``film_id``)` `AS` `film_cnt` `FROM`
    -> `actor` `INNER` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `WHERE` `first_name` `=` `'EMILY'` `AND` `last_name` `=` `'DEE'`
    -> `GROUP` `BY` `actor_id``,` `first_name``,` `last_name``;`
```

```
+------------+-----------+----------+
| first_name | last_name | film_cnt |
+------------+-----------+----------+
| EMILY      | DEE       |       14 |
+------------+-----------+----------+
1 row in set (0.00 sec)
```

这个正确的查询形成了组，然后根据`WHERE`子句选择要显示的组。

# 高级连接

到目前为止，在本书中，我们已经使用`INNER JOIN`子句从两个或多个表中汇集行。我们将在本节中更详细地解释内连接，并将其与我们讨论的其他连接类型进行对比：联合、左连接、右连接和自然连接。在本节结束时，你将能够回答困难的信息需求，并熟悉选择适合当前任务的正确连接方式。

## 内连接

`INNER JOIN`子句根据你在`USING`子句中提供的条件匹配两个表之间的行。例如，你现在非常熟悉`actor`和`film_actor`表的内连接：

```
mysql> `SELECT` `first_name``,` `last_name``,` `film_id` `FROM`
    -> `actor` `INNER` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `LIMIT` `20``;`
```

```
+------------+-----------+---------+
| first_name | last_name | film_id |
+------------+-----------+---------+
| PENELOPE   | GUINESS   |       1 |
| PENELOPE   | GUINESS   |      23 |
| ...                              |
| PENELOPE   | GUINESS   |     980 |
| NICK       | WAHLBERG  |       3 |
+------------+-----------+---------+
20 rows in set (0.00 sec)
```

让我们回顾一下`INNER JOIN`的关键特点：

+   `INNER JOIN`关键字短语的两侧列出了两个表（或前一个连接的结果）。

+   `USING`子句定义了一个或多个在两个表或结果中都存在并用于连接或匹配行的列。

+   不匹配的行不会被返回。例如，如果`actor`表中有一行没有在`film_actor`表中有任何匹配的电影，它将不会包含在输出中。

你实际上可以使用`WHERE`子句编写带有`INNER JOIN`关键字的内连接查询。以下是产生相同结果的前一个查询的重写版本：

```
mysql> `SELECT` `first_name``,` `last_name``,` `film_id`
    -> `FROM` `actor``,` `film_actor`
    -> `WHERE` `actor``.``actor_id` `=` `film_actor``.``actor_id`
    -> `LIMIT` `20``;`
```

```
+------------+-----------+---------+
| first_name | last_name | film_id |
+------------+-----------+---------+
| PENELOPE   | GUINESS   |       1 |
| PENELOPE   | GUINESS   |      23 |
| ...                              |
| PENELOPE   | GUINESS   |     980 |
| NICK       | WAHLBERG  |       3 |
+------------+-----------+---------+
20 rows in set (0.00 sec)
```

你可以看到我们没有明确说明内连接：我们正在从`actor`和`film_actor`表中选择标识符在表之间匹配的行。

你可以修改`INNER JOIN`语法，以一种类似于使用`WHERE`子句的方式表达连接条件。如果在表之间的标识符名称不匹配，这将非常有用，尽管在这个例子中并非如此。以下是以这种风格重写的前一个查询：

```
mysql> `SELECT` `first_name``,` `last_name``,` `film_id` `FROM`
    -> `actor` `INNER` `JOIN` `film_actor`
    -> `ON` `actor``.``actor_id` `=` `film_actor``.``actor_id`
    -> `LIMIT` `20``;`
```

```
+------------+-----------+---------+
| first_name | last_name | film_id |
+------------+-----------+---------+
| PENELOPE   | GUINESS   |       1 |
| PENELOPE   | GUINESS   |      23 |
| ...                              |
| PENELOPE   | GUINESS   |     980 |
| NICK       | WAHLBERG  |       3 |
+------------+-----------+---------+
20 rows in set (0.00 sec)
```

你可以看到`ON`子句取代了`USING`子句，并且后面跟着的列完全指定了包括表和列名。如果两个表之间的列名称不同且唯一，你可以省略表名。使用`ON`或`WHERE`子句没有真正的优势或劣势；这只是一种口味问题。通常，如今，你会发现大多数 SQL 专业人员更喜欢使用带有`ON`子句的`INNER JOIN`，而不是`WHERE`，但这并非普遍适用。

在继续之前，让我们考虑`WHERE`、`ON`和`USING`子句的作用。如果从我们刚刚展示的查询中省略`WHERE`子句，将得到一个非常不同的结果。以下是查询和输出的前几行：

```
mysql> `SELECT` `first_name``,` `last_name``,` `film_id`
    -> `FROM` `actor``,` `film_actor` `LIMIT` `20``;`
```

```
+------------+-------------+---------+
| first_name | last_name   | film_id |
+------------+-------------+---------+
| THORA      | TEMPLE      |       1 |
| JULIA      | FAWCETT     |       1 |
| ...                                |
| DEBBIE     | AKROYD      |       1 |
| MATTHEW    | CARREY      |       1 |
+------------+-------------+---------+
20 rows in set (0.00 sec)
```

输出是荒谬的：发生的情况是将`actor`表中的每一行与`film_actor`表中的每一行一起输出，得到所有可能的组合。由于`actor`表有 200 名演员和`film_actor`表有 5,462 条记录，所以输出行数为 200 × 5,462 = 1,092,400 行，我们知道其中只有 5,462 个组合是有意义的（即有 5,462 条记录是演员参演电影的记录）。我们可以看到在没有`LIMIT`的情况下我们会得到多少行，以下是查询的内容：

```
mysql> `SELECT` `COUNT``(``*``)` `FROM` `actor``,` `film_actor``;`
```

```
+----------+
| COUNT(*) |
+----------+
|  1092400 |
+----------+
1 row in set (0.00 sec)
```

这种查询类型，如果没有匹配行的子句，被称为*笛卡尔积*。顺便说一句，如果在没有使用`USING`或`ON`子句指定列的情况下执行内连接，也会得到笛卡尔积，就像这个查询一样：

```
SELECT first_name, last_name, film_id
FROM actor INNER JOIN film_actor;
```

在“自然连接”中，我们将介绍自然连接，它是基于相同列名的内连接。虽然自然连接不使用明确指定的列，但它仍然产生内连接，而不是笛卡尔积。

关键词`INNER JOIN`可以用`JOIN`或`STRAIGHT JOIN`来替代；它们的功能是相同的。然而，`STRAIGHT JOIN`会强制 MySQL 在读取右表之前始终先读取左表。我们将在第七章详细讨论 MySQL 在后台如何处理查询。关键词`JOIN`是最常见的，它是许多其他数据库系统（除了 MySQL）中用于`INNER JOIN`的标准缩写，在我们的内连接示例中也会使用它。

## 联合

`UNION`语句实际上不是一个连接运算符。相反，它允许你组合多个`SELECT`语句的输出，以提供一个汇总的结果集。在你想要从多个源中产生单一列表的情况下，或者你想要从一个单一源中创建难以在单一查询中表达的列表时，它是非常有用的。

让我们来看一个例子。如果你想在`sakila`数据库中输出所有演员和电影以及顾客的名字，你可以使用`UNION`语句。这是一个人为的例子，但你可能只是想列出所有的文本片段，而不是有意义地展示数据之间的关系。在`actor.first_name`、`film.title`和`customer.first_name`列中都有文本。以下是如何显示它的方式：

```
mysql> `SELECT` `first_name` `FROM` `actor`
    -> `UNION`
    -> `SELECT` `first_name` `FROM` `customer`
    -> `UNION`
    -> `SELECT` `title` `FROM` `film``;`
```

```
+-----------------------------+
| first_name                  |
+-----------------------------+
| PENELOPE                    |
| NICK                        |
| ED                          |
| ...                         |
| ZHIVAGO CORE                |
| ZOOLANDER FICTION           |
| ZORRO ARK                   |
+-----------------------------+
1647 rows in set (0.00 sec)
```

我们只展示了 1,647 行中的一小部分。`UNION`语句会将所有查询的结果一起输出，在第一个查询下显示适当的标题。

一个稍微不那么人为的例子是创建一个数据库中租借最多和最少的五部电影的列表。你可以很容易地使用`UNION`运算符来实现这一点：

```
mysql> `(``SELECT` `title``,` `COUNT``(``rental_id``)` `AS` `num_rented`
    -> `FROM` `film` `JOIN` `inventory` `USING` `(``film_id``)`
    -> `JOIN` `rental` `USING` `(``inventory_id``)`
    -> `GROUP` `BY` `title` `ORDER` `BY` `num_rented` `DESC` `LIMIT` `5``)`
    -> `UNION`
    -> `(``SELECT` `title``,` `COUNT``(``rental_id``)` `AS` `num_rented`
    -> `FROM` `film` `JOIN` `inventory` `USING` `(``film_id``)`
    -> `JOIN` `rental` `USING` `(``inventory_id``)`
    -> `GROUP` `BY` `title` `ORDER` `BY` `num_rented` `ASC` `LIMIT` `5``)``;`
```

```
+--------------------+------------+
| title              | num_rented |
+--------------------+------------+
| BUCKET BROTHERHOOD |         34 |
| ROCKETEER MOTHER   |         33 |
| FORWARD TEMPLE     |         32 |
| GRIT CLOCKWORK     |         32 |
| JUGGLER HARDLY     |         32 |
| TRAIN BUNCH        |          4 |
| HARDLY ROBBERS     |          4 |
| MIXED DOORS        |          4 |
| BUNCH MINDS        |          5 |
| BRAVEHEART HUMAN   |          5 |
+--------------------+------------+
10 rows in set (0.04 sec)
```

第一个查询使用带有`DESC`（降序）修饰符和`LIMIT 5`子句来查找租借次数最多的前五部电影。第二个查询使用带有`ASC`（升序）修饰符和`LIMIT 5`子句来查找租借次数最少的五部电影。`UNION`组合了结果集。请注意，具有相同`num_rented`值的多个标题，其顺序不能保证确定。在您的系统上，可能会看到不同标题列出`num_rented`值为 32 和 5 的情况。

`UNION`运算符有几个限制：

+   输出标记为来自第一个查询的列名或表达式的名称。使用列别名可以更改此行为。

+   查询必须输出相同数量的列。如果尝试使用不同数量的列，MySQL 将报错。

+   所有匹配列必须具有相同的类型。因此，例如，如果第一个查询输出的第一列是日期，则任何其他查询输出的第一列也必须是日期。

+   返回的结果是唯一的，就像你应用了`DISTINCT`到整个结果集一样。为了看到它的效果，让我们尝试一个简单的例子。记住我们之前在演员名字方面遇到的问题——名字不是一个好的唯一标识符。如果我们选择两个具有相同名字的演员，并且将这两个查询使用`UNION`，我们最终只会得到一行结果。隐式的`DISTINCT`操作隐藏了重复（对于`UNION`）的行：

    ```
    mysql> `SELECT` `first_name` `FROM` `actor` `WHERE` `actor_id` `=` `88`
        -> `UNION`
        -> `SELECT` `first_name` `FROM` `actor` `WHERE` `actor_id` `=` `169``;`
    ```

    ```
    +------------+
    | first_name |
    +------------+
    | KENNETH    |
    +------------+
    1 row in set (0.01 sec)
    ```

    如果你想显示任何重复项，请用`UNION`替换为`UNION ALL`：

    ```
    mysql> `SELECT` `first_name` `FROM` `actor` `WHERE` `actor_id` `=` `88`
        -> `UNION` `ALL`
        -> `SELECT` `first_name` `FROM` `actor` `WHERE` `actor_id` `=` `169``;`
    ```

    ```
    +------------+
    | first_name |
    +------------+
    | KENNETH    |
    | KENNETH    |
    +------------+
    2 rows in set (0.00 sec)
    ```

    这里，名为`KENNETH`的名字出现了两次。

    `UNION`执行的隐式`DISTINCT`会对性能产生非零的影响。每当使用`UNION`时，请查看是否逻辑上适合使用`UNION ALL`，以及它是否可以提高查询性能。

+   如果你想在`UNION`语句的一个单独查询中应用`LIMIT`或`ORDER BY`，请将该查询用括号括起来（如前面的例子所示）。无论如何，使用括号使查询易于理解是很有用的。

    `UNION`操作仅简单地连接组成查询的结果，不考虑顺序，因此在子查询中使用`ORDER BY`没有太大意义。在`UNION`操作中对子查询进行排序只有在想要选择结果子集时才有意义。在我们的示例中，我们通过电影租借次数对电影进行了排序，并仅选择了前五部（在第一个子查询中）和最后五部（在第二个子查询中）。

    出于效率考虑，如果在子查询中使用`ORDER BY`而没有`LIMIT`，MySQL 实际上会忽略`ORDER BY`子句。让我们看一些示例来确切了解其工作原理。

    首先，让我们运行一个简单的查询，列出特定电影的租赁信息，以及租赁发生的时间。为了与我们的其他示例保持一致，我们将查询括在括号中——这里括号实际上没有任何效果——并且没有使用`ORDER BY`或`LIMIT`子句：

    ```
    mysql> `(``SELECT` `title``,` `rental_date``,` `return_date`
        -> `FROM` `film` `JOIN` `inventory` `USING` `(``film_id``)`
        -> `JOIN` `rental` `USING` `(``inventory_id``)`
        -> `WHERE` `film_id` `=` `998``)``;`
    ```

    ```
    +--------------+---------------------+---------------------+
    | title        | rental_date         | return_date         |
    +--------------+---------------------+---------------------+
    | ZHIVAGO CORE | 2005-06-17 03:19:20 | 2005-06-21 00:19:20 |
    | ZHIVAGO CORE | 2005-07-07 12:18:57 | 2005-07-12 09:47:57 |
    | ZHIVAGO CORE | 2005-07-27 14:53:55 | 2005-07-31 19:48:55 |
    | ZHIVAGO CORE | 2005-08-20 17:18:48 | 2005-08-26 15:31:48 |
    | ZHIVAGO CORE | 2005-05-30 05:15:20 | 2005-06-07 00:49:20 |
    | ZHIVAGO CORE | 2005-06-18 06:46:54 | 2005-06-26 09:48:54 |
    | ZHIVAGO CORE | 2005-07-12 05:24:02 | 2005-07-16 03:43:02 |
    | ZHIVAGO CORE | 2005-08-02 02:05:04 | 2005-08-10 21:58:04 |
    | ZHIVAGO CORE | 2006-02-14 15:16:03 | NULL                |
    +--------------+---------------------+---------------------+
    9 rows in set (0.00 sec)
    ```

    查询返回所有租用电影的时间，顺序没有特定要求（参见第四和第五条目）。

    现在，让我们在这个查询中添加一个`ORDER BY`子句：

    ```
    mysql> `(``SELECT` `title``,` `rental_date``,` `return_date`
        -> `FROM` `film` `JOIN` `inventory` `USING` `(``film_id``)`
        -> `JOIN` `rental` `USING` `(``inventory_id``)`
        -> `WHERE` `film_id` `=` `998`
        -> `ORDER` `BY` `rental_date` `ASC``)``;`
    ```

    ```
    +--------------+---------------------+---------------------+
    | title        | rental_date         | return_date         |
    +--------------+---------------------+---------------------+
    | ZHIVAGO CORE | 2005-05-30 05:15:20 | 2005-06-07 00:49:20 |
    | ZHIVAGO CORE | 2005-06-17 03:19:20 | 2005-06-21 00:19:20 |
    | ZHIVAGO CORE | 2005-06-18 06:46:54 | 2005-06-26 09:48:54 |
    | ZHIVAGO CORE | 2005-07-07 12:18:57 | 2005-07-12 09:47:57 |
    | ZHIVAGO CORE | 2005-07-12 05:24:02 | 2005-07-16 03:43:02 |
    | ZHIVAGO CORE | 2005-07-27 14:53:55 | 2005-07-31 19:48:55 |
    | ZHIVAGO CORE | 2005-08-02 02:05:04 | 2005-08-10 21:58:04 |
    | ZHIVAGO CORE | 2005-08-20 17:18:48 | 2005-08-26 15:31:48 |
    | ZHIVAGO CORE | 2006-02-14 15:16:03 | NULL                |
    +--------------+---------------------+---------------------+
    9 rows in set (0.00 sec)
    ```

    如预期的那样，我们按照租赁日期的顺序获取了所有租用电影的时间。

    在前一个查询中添加一个`LIMIT`子句会选择前五个租赁，按时间顺序排列——这里没有意外：

    ```
    mysql> `(``SELECT` `title``,` `rental_date``,` `return_date`
        -> `FROM` `film` `JOIN` `inventory` `USING` `(``film_id``)`
        -> `JOIN` `rental` `USING` `(``inventory_id``)`
        -> `WHERE` `film_id` `=` `998`
        -> `ORDER` `BY` `rental_date` `ASC` `LIMIT` `5``)``;`
    ```

    ```
    +--------------+---------------------+---------------------+
    | title        | rental_date         | return_date         |
    +--------------+---------------------+---------------------+
    | ZHIVAGO CORE | 2005-05-30 05:15:20 | 2005-06-07 00:49:20 |
    | ZHIVAGO CORE | 2005-06-17 03:19:20 | 2005-06-21 00:19:20 |
    | ZHIVAGO CORE | 2005-06-18 06:46:54 | 2005-06-26 09:48:54 |
    | ZHIVAGO CORE | 2005-07-07 12:18:57 | 2005-07-12 09:47:57 |
    | ZHIVAGO CORE | 2005-07-12 05:24:02 | 2005-07-16 03:43:02 |
    +--------------+---------------------+---------------------+
    5 rows in set (0.01 sec)
    ```

    现在，让我们看看在执行`UNION`操作时会发生什么。在这个例子中，我们使用了两个带有`ORDER BY`子句的子查询。我们对第二个子查询使用了`LIMIT`子句，但没有对第一个子查询使用：

    ```
    mysql> `(``SELECT` `title``,` `rental_date``,` `return_date`
        -> `FROM` `film` `JOIN` `inventory` `USING` `(``film_id``)`
        -> `JOIN` `rental` `USING` `(``inventory_id``)`
        -> `WHERE` `film_id` `=` `998`
        -> `ORDER` `BY` `rental_date` `ASC``)`
        -> `UNION` `ALL`
        -> `(``SELECT` `title``,` `rental_date``,` `return_date`
        -> `FROM` `film` `JOIN` `inventory` `USING` `(``film_id``)`
        -> `JOIN` `rental` `USING` `(``inventory_id``)`
        -> `WHERE` `film_id` `=` `998`
        -> `ORDER` `BY` `rental_date` `ASC` `LIMIT` `5``)``;`
    ```

    ```
    +--------------+---------------------+---------------------+
    | title        | rental_date         | return_date         |
    +--------------+---------------------+---------------------+
    | ZHIVAGO CORE | 2005-06-17 03:19:20 | 2005-06-21 00:19:20 |
    | ZHIVAGO CORE | 2005-07-07 12:18:57 | 2005-07-12 09:47:57 |
    | ZHIVAGO CORE | 2005-07-27 14:53:55 | 2005-07-31 19:48:55 |
    | ZHIVAGO CORE | 2005-08-20 17:18:48 | 2005-08-26 15:31:48 |
    | ZHIVAGO CORE | 2005-05-30 05:15:20 | 2005-06-07 00:49:20 |
    | ZHIVAGO CORE | 2005-06-18 06:46:54 | 2005-06-26 09:48:54 |
    | ZHIVAGO CORE | 2005-07-12 05:24:02 | 2005-07-16 03:43:02 |
    | ZHIVAGO CORE | 2005-08-02 02:05:04 | 2005-08-10 21:58:04 |
    | ZHIVAGO CORE | 2006-02-14 15:16:03 | NULL                |
    | ZHIVAGO CORE | 2005-05-30 05:15:20 | 2005-06-07 00:49:20 |
    | ZHIVAGO CORE | 2005-06-17 03:19:20 | 2005-06-21 00:19:20 |
    | ZHIVAGO CORE | 2005-06-18 06:46:54 | 2005-06-26 09:48:54 |
    | ZHIVAGO CORE | 2005-07-07 12:18:57 | 2005-07-12 09:47:57 |
    | ZHIVAGO CORE | 2005-07-12 05:24:02 | 2005-07-16 03:43:02 |
    +--------------+---------------------+---------------------+
    14 rows in set (0.01 sec)
    ```

    正如预期的那样，第一个子查询返回所有租用电影的时间（输出的前九行），第二个子查询返回前五个租赁（输出的最后五行）。请注意，尽管第一个子查询有`ORDER BY`子句，前九行并未按顺序排列（参见第四和第五行）。由于我们执行了`UNION`操作，MySQL 服务器决定不对子查询的结果进行排序。第二个子查询包含了一个`LIMIT`操作，因此该子查询的结果是排序的。

    即使子查询已经排序，`UNION`操作的输出并不能保证有序，因此如果你希望最终输出有序，应该在整个查询的末尾添加一个`ORDER BY`子句。请注意，它可以是另一种顺序，不同于子查询。参见以下内容：

    ```
    mysql> `(``SELECT` `title``,` `rental_date``,` `return_date`
        -> `FROM` `film` `JOIN` `inventory` `USING` `(``film_id``)`
        -> `JOIN` `rental` `USING` `(``inventory_id``)`
        -> `WHERE` `film_id` `=` `998`
        -> `ORDER` `BY` `rental_date` `ASC``)`
        -> `UNION` `ALL`
        -> `(``SELECT` `title``,` `rental_date``,` `return_date`
        -> `FROM` `film` `JOIN` `inventory` `USING` `(``film_id``)`
        -> `JOIN` `rental` `USING` `(``inventory_id``)`
        -> `WHERE` `film_id` `=` `998`
        -> `ORDER` `BY` `rental_date` `ASC` `LIMIT` `5``)`
        -> `ORDER` `BY` `rental_date` `DESC``;`
    ```

    ```
    +--------------+---------------------+---------------------+
    | title        | rental_date         | return_date         |
    +--------------+---------------------+---------------------+
    | ZHIVAGO CORE | 2006-02-14 15:16:03 | NULL                |
    | ZHIVAGO CORE | 2005-08-20 17:18:48 | 2005-08-26 15:31:48 |
    | ZHIVAGO CORE | 2005-08-02 02:05:04 | 2005-08-10 21:58:04 |
    | ZHIVAGO CORE | 2005-07-27 14:53:55 | 2005-07-31 19:48:55 |
    | ZHIVAGO CORE | 2005-07-12 05:24:02 | 2005-07-16 03:43:02 |
    | ZHIVAGO CORE | 2005-07-12 05:24:02 | 2005-07-16 03:43:02 |
    | ZHIVAGO CORE | 2005-07-07 12:18:57 | 2005-07-12 09:47:57 |
    | ZHIVAGO CORE | 2005-07-07 12:18:57 | 2005-07-12 09:47:57 |
    | ZHIVAGO CORE | 2005-06-18 06:46:54 | 2005-06-26 09:48:54 |
    | ZHIVAGO CORE | 2005-06-18 06:46:54 | 2005-06-26 09:48:54 |
    | ZHIVAGO CORE | 2005-06-17 03:19:20 | 2005-06-21 00:19:20 |
    | ZHIVAGO CORE | 2005-06-17 03:19:20 | 2005-06-21 00:19:20 |
    | ZHIVAGO CORE | 2005-05-30 05:15:20 | 2005-06-07 00:49:20 |
    | ZHIVAGO CORE | 2005-05-30 05:15:20 | 2005-06-07 00:49:20 |
    +--------------+---------------------+---------------------+
    14 rows in set (0.00 sec)
    ```

    这里是另一个示例，包括对最终结果进行排序并限制返回结果的数量：

    ```
    mysql> `(``SELECT` `first_name``,` `last_name` `FROM` `actor` `WHERE` `actor_id` `<` `5``)`
        -> `UNION`
        -> `(``SELECT` `first_name``,` `last_name` `FROM` `actor` `WHERE` `actor_id` `>` `190``)`
        -> `ORDER` `BY` `first_name` `LIMIT` `4``;`
    ```

    ```
    +------------+-----------+
    | first_name | last_name |
    +------------+-----------+
    | BELA       | WALKEN    |
    | BURT       | TEMPLE    |
    | ED         | CHASE     |
    | GREGORY    | GOODING   |
    +------------+-----------+
    4 rows in set (0.00 sec)
    ```

    `UNION`操作有些笨重，通常有获得相同结果的替代方法。例如，前面的查询可以更简单地写成这样：

    ```
    mysql> `SELECT` `first_name``,` `last_name` `FROM` `actor`
        -> `WHERE` `actor_id` `<` `5` `OR` `actor_id` `>` `190`
        -> `ORDER` `BY` `first_name` `LIMIT` `4``;`
    ```

    ```
    +------------+-----------+
    | first_name | last_name |
    +------------+-----------+
    | BELA       | WALKEN    |
    | BURT       | TEMPLE    |
    | ED         | CHASE     |
    | GREGORY    | GOODING   |
    +------------+-----------+
    4 rows in set (0.00 sec)
    ```

## 左连接和右连接

我们到目前为止讨论的连接仅输出两个表之间匹配的行。例如，当你通过`inventory`表连接`film`和`rental`表时，只会看到已经租出的电影。未租出的电影行将被忽略。在许多情况下这是有道理的，但这并不是连接数据的唯一方式。本节将解释你可以选择的其他选项。

假设您确实想要一个全面的电影列表以及它们被租赁的次数。与本章前面的示例不同，您希望在列表中看到未被租赁的电影旁边的数字为零。您可以使用 *left join* 来实现这一点，这是一种由参与连接的两个表中的一个驱动的不同类型的连接。在左连接中，左表中的每一行（驱动表）都会被处理并输出，如果第二个表中存在匹配的数据，则显示第二个表中的匹配数据，如果第二个表中没有匹配的数据，则显示 `NULL` 值。我们将在本节后面向您展示如何编写这种类型的查询，但我们将从一个更简单的示例开始。

这里是一个简单的 `LEFT JOIN` 示例。您想列出所有电影，并在每部电影旁边显示它们被租赁的时间。如果一部电影从未被租赁过，您也想看到这一点。如果一部电影被租赁多次，您也想看到这一点。查询如下：

```
mysql> `SELECT` `title``,` `rental_date`
    -> `FROM` `film` `LEFT` `JOIN` `inventory` `USING` `(``film_id``)`
    -> `LEFT` `JOIN` `rental` `USING` `(``inventory_id``)``;`
```

```
+-----------------------------+---------------------+
| title                       | rental_date         |
+-----------------------------+---------------------+
| ACADEMY DINOSAUR            | 2005-07-08 19:03:15 |
| ACADEMY DINOSAUR            | 2005-08-02 20:13:10 |
| ACADEMY DINOSAUR            | 2005-08-21 21:27:43 |
| ...                                               |
| WAKE JAWS                   | NULL                |
| WALLS ARTIST                | NULL                |
| ...                                               |
| ZORRO ARK                   | 2005-07-31 07:32:21 |
| ZORRO ARK                   | 2005-08-19 03:49:28 |
+-----------------------------+---------------------+
16087 rows in set (0.06 sec)
```

你可以看到发生了什么：已租赁的电影有日期和时间，而未租赁的电影则没有（`rental_date` 值为 `NULL`）。还要注意，此示例中我们两次使用了 `LEFT JOIN`。首先，我们连接了 `film` 和 `inventory`，并且我们希望即使一部电影不在我们的库存中（因此根据定义不能租赁），我们仍然输出它。然后我们将 `rental` 表与从前面连接结果中的数据集连接。我们再次使用 `LEFT JOIN`，因为我们可能有一些电影不在我们的库存中，这些电影在 `rental` 表中将没有任何行。然而，我们也可能有列在我们库存中但从未被租赁的电影。这就是我们在这里需要在两个表上使用 `LEFT JOIN` 的原因。

`LEFT JOIN` 中表的顺序很重要。如果您反转上一个查询中的顺序，将得到非常不同的输出：

```
mysql> `SELECT` `title``,` `rental_date`
    -> `FROM` `rental` `LEFT` `JOIN` `inventory` `USING` `(``inventory_id``)`
    -> `LEFT` `JOIN` `film` `USING` `(``film_id``)`
    -> `ORDER` `BY` `rental_date` `DESC``;`
```

```
+-----------------------------+---------------------+
| title                       | rental_date         |
+-----------------------------+---------------------+
| ...                                               |
| LOVE SUICIDES               | 2005-05-24 23:04:41 |
| GRADUATE LORD               | 2005-05-24 23:03:39 |
| FREAKY POCUS                | 2005-05-24 22:54:33 |
| BLANKET BEVERLY             | 2005-05-24 22:53:30 |
+-----------------------------+---------------------+
16044 rows in set (0.06 sec)
```

在这个版本中，查询由 `rental` 表驱动，因此所有从它中得到的行都与 `inventory` 表和 `film` 表匹配。由于根据定义 `rental` 表中的所有行都基于 `inventory` 表，后者与 `film` 表关联，因此我们在输出中没有 `NULL` 值。不存在没有对应电影的租赁。我们通过使用 `ORDER BY rental_date DESC` 调整了查询以显示我们确实没有得到任何 `NULL` 值（这些将是最后出现的）。

现在您可以看到，在我们确信我们的 *left* 表中有一些重要数据但不确定 *right* 表中是否有数据时，左连接非常有用。我们希望从左表获取带有或不带有右表相应行的行。让我们试着将这个应用到我们在 “GROUP BY 子句” 中编写的查询中，该查询显示了从同一类别大量租赁的客户。以下是查询，作为提醒：

```
mysql> `SELECT` `email``,` `name` `AS` `category_name``,` `COUNT``(``cat``.``category_id``)` `AS` `cnt`
    -> `FROM` `customer` `cs` `INNER` `JOIN` `rental` `USING` `(``customer_id``)`
    -> `INNER` `JOIN` `inventory` `USING` `(``inventory_id``)`
    -> `INNER` `JOIN` `film_category` `USING` `(``film_id``)`
    -> `INNER` `JOIN` `category` `cat` `USING` `(``category_id``)`
    -> `GROUP` `BY` `email``,` `category_name`
    -> `ORDER` `BY` `cnt` `DESC` `LIMIT` `5``;`
```

```
+----------------------------------+---------------+-----+
| email                            | category_name | cnt |
+----------------------------------+---------------+-----+
| WESLEY.BULL@sakilacustomer.org   | Games         |   9 |
| ALMA.AUSTIN@sakilacustomer.org   | Animation     |   8 |
| KARL.SEAL@sakilacustomer.org     | Animation     |   8 |
| LYDIA.BURKE@sakilacustomer.org   | Documentary   |   8 |
| NATHAN.RUNYON@sakilacustomer.org | Animation     |   7 |
+----------------------------------+---------------+-----+
5 rows in set (0.06 sec)
```

现在，假设我们想要查看通过这种方式找到的客户是否租赁除了他们最喜欢的类别之外的电影？事实证明这实际上相当困难！

让我们考虑这个任务。我们需要从`category`表开始，因为那里有我们电影的所有分类。然后，我们需要开始构建一整个链条的左连接。首先，我们将`category`左连接到`film_category`，因为我们可能有没有电影的分类。然后，我们将结果左连接到`inventory`表，因为我们知道的一些电影可能不在我们的目录中。然后，我们将该结果左连接到`rental`表，因为顾客可能没有租用某些类别的电影。最后，我们需要将该结果左连接到我们的`customer`表。即使没有租赁记录与关联的客户记录，省略此处的左连接将导致 MySQL 丢弃没有顾客记录的类别行。

现在，在这整个长篇解释之后，我们可以继续按电子邮件地址过滤并获取我们的数据吗？不！不幸的是，通过在左连接关系中不是左侧的表上添加`WHERE`条件，我们破坏了此连接的理念。看看会发生什么：

```
mysql> `SELECT` `COUNT``(``*``)` `FROM` `category``;`
```

```
+----------+
| COUNT(*) |
+----------+
|       16 |
+----------+
1 row in set (0.00 sec)
```

```
mysql> `SELECT` `email``,` `name` `AS` `category_name``,` `COUNT``(``category_id``)` `AS` `cnt`
    -> `FROM` `category` `cat` `LEFT` `JOIN` `film_category` `USING` `(``category_id``)`
    -> `LEFT` `JOIN` `inventory` `USING` `(``film_id``)`
    -> `LEFT` `JOIN` `rental` `USING` `(``inventory_id``)`
    -> `JOIN` `customer` `cs` `ON` `rental``.``customer_id` `=` `cs``.``customer_id`
    -> `WHERE` `cs``.``email` `=` `'WESLEY.BULL@sakilacustomer.org'`
    -> `GROUP` `BY` `email``,` `category_name`
    -> `ORDER` `BY` `cnt` `DESC``;`
```

```
+--------------------------------+---------------+-----+
| email                          | category_name | cnt |
+--------------------------------+---------------+-----+
| WESLEY.BULL@sakilacustomer.org | Games         |   9 |
| WESLEY.BULL@sakilacustomer.org | Foreign       |   6 |
| ...                                                  |
| WESLEY.BULL@sakilacustomer.org | Comedy        |   1 |
| WESLEY.BULL@sakilacustomer.org | Sports        |   1 |
+--------------------------------+---------------+-----+
14 rows in set (0.00 sec)
```

我们为我们的客户获得了 14 个类别，而总共有 16 个。实际上，MySQL 将在这个查询中优化所有左连接，因为它理解到它们放在这里是没有意义的。在仅使用连接来回答我们的问题没有简单的方法——我们将在“嵌套连接中的查询”中返回到这个例子。

尽管默认情况下`sakila`没有未租用电影的电影类别，但如果我们稍微扩展我们的数据库，我们可以看到左连接的有效性：

```
mysql> `INSERT` `INTO` `category``(``name``)` `VALUES` `(``'Thriller'``)``;`

```

```
Query OK, 1 row affected (0.01 sec)
```

```
mysql> `SELECT` `cat``.``name``,` `COUNT``(``rental_id``)` `cnt`
    -> `FROM` `category` `cat` `LEFT` `JOIN` `film_category` `USING` `(``category_id``)`
    -> `LEFT` `JOIN` `inventory` `USING` `(``film_id``)`
    -> `LEFT` `JOIN` `rental` `USING` `(``inventory_id``)`
    -> `LEFT` `JOIN` `customer` `cs` `ON` `rental``.``customer_id` `=` `cs``.``customer_id`
    -> `GROUP` `BY` `1`
    -> `ORDER` `BY` `2` `DESC``;`
```

```
+---------------+------+
| category_name | cnt  |
+---------------+------+
| Sports        | 1179 |
| Animation     | 1166 |
| ...                  |
| Music         |  830 |
| Thriller      |    0 |
+---------------+------+
17 rows in set (0.07 sec)
```

如果我们在这里使用常规的`INNER JOIN`（或其同义词`JOIN`），我们将无法获取 Thriller 类别的信息，而其他类别的计数可能会有所不同。由于`category`是我们最左边的表，它驱动查询的过程，并且该表中的每一行都存在于输出中。

我们已经向您展示了`LEFT JOIN`关键词之前和之后的内容很重要。左侧的内容驱动整个过程，因此称为“左连接”。如果您真的不想重新组织查询以匹配该模板，可以使用`RIGHT JOIN`。它完全相同，只是右侧的内容驱动整个过程。

早些时候，我们展示了左连接中表的顺序的重要性，使用两个用于电影租赁信息的查询。让我们使用右连接重新编写第二个查询（显示不正确数据）：

```
mysql> `SELECT` `title``,` `rental_date`
    -> `FROM` `rental` `RIGHT` `JOIN` `inventory` `USING` `(``inventory_id``)`
    -> `RIGHT` `JOIN` `film` `USING` `(``film_id``)`
    -> `ORDER` `BY` `rental_date` `DESC``;`
```

```
...
| SUICIDES SILENCE            | NULL                |
| TADPOLE PARK                | NULL                |
| TREASURE COMMAND            | NULL                |
| VILLAIN DESPERATE           | NULL                |
| VOLUME HOUSE                | NULL                |
| WAKE JAWS                   | NULL                |
| WALLS ARTIST                | NULL                |
+-----------------------------+---------------------+
16087 rows in set (0.06 sec)
```

我们得到了相同数量的行，并且我们可以看到`NULL`值与“正确”查询给我们的值相同。右连接有时很有用，因为它允许您更自然地编写查询，以更直观的方式表达它。然而，您不经常看到它被使用，我们建议在可能的情况下避免使用它。

左连接和右连接都可以使用“内连接”中讨论的`USING`和`ON`子句。你应该选择其中之一：没有它们，你将得到笛卡尔积，正如该节中讨论的那样。

在左连接和右连接中，还可以选择使用额外的`OUTER`关键字，使它们读起来像`LEFT OUTER JOIN`和`RIGHT OUTER JOIN`。这只是一种不同的语法，实际上并没有任何不同，你不经常看到它被使用。在本书中，我们坚持使用基本版本。

## 自然连接

我们并不是这里描述的自然连接的铁杆支持者。它在这里仅仅是为了完整性，也因为你会在遇到的 SQL 语句中偶尔看到它。我们的建议是尽可能避免在可以的情况下使用它。

自然连接应该是一种自然而然的魔法。这意味着你告诉 MySQL 你想连接哪些表，它会找出如何连接它们，并给你一个`INNER JOIN`的结果集。这里有一个关于`actor_info`和`film_actor`表的例子：

```
mysql> `SELECT` `first_name``,` `last_name``,` `film_id`
    -> `FROM` `actor_info` `NATURAL` `JOIN` `film_actor`
    -> `LIMIT` `20``;`
```

```
+------------+-----------+---------+
| first_name | last_name | film_id |
+------------+-----------+---------+
| PENELOPE   | GUINESS   |       1 |
| PENELOPE   | GUINESS   |      23 |
| ...                              |
| PENELOPE   | GUINESS   |     980 |
| NICK       | WAHLBERG  |       3 |
+------------+-----------+---------+
20 rows in set (0.28 sec)
```

实际上，这并不是什么神奇的事情：MySQL 只是查找具有相同名称的列，并在幕后将这些列静默地添加到具有连接条件的`WHERE`子句中。因此，前面的查询实际上被转换成类似于这样的内容：

```
mysql> `SELECT` `first_name``,` `last_name``,` `film_id` `FROM`
    -> `actor_info` `JOIN` `film_actor`
    -> `WHERE` `(``actor_info``.``actor_id` `=` `film_actor``.``actor_id``)`
    -> `LIMIT` `20``;`
```

如果标识符列没有相同的名称，自然连接将无法工作。更危险的是，如果具有相同名称的列不是标识符，它们将被静默地添加到后台的`USING`子句中。你可以很容易地在`sakila`数据库中看到这一点。事实上，这就是为什么我们选择显示前面的例子与`actor_info`，它甚至不是一个表：它是一个视图。让我们看看如果我们使用常规的`actor`和`film_actor`表会发生什么：

```
mysql> `SELECT` `first_name``,` `last_name``,` `film_id` `FROM` `actor` `NATURAL` `JOIN` `film_actor``;`
```

```
Empty set (0.01 sec)
```

但是如何呢？问题是：`NATURAL JOIN`确实考虑了*所有*列。在`sakila`数据库中，这是一个巨大的障碍，因为每个表都有一个`last_update`列。如果你运行前面查询的`EXPLAIN`语句，然后执行`SHOW WARNINGS`，你会发现生成的查询是毫无意义的：

```
mysql> `SHOW` `WARNINGS``\``G`
```

```
*************************** 1\. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `sakila`.`customer`.`email` AS `email`,
`sakila`.`rental`.`rental_date` AS `rental_date`
from `sakila`.`customer` join `sakila`.`rental`
where ((`sakila`.`rental`.`last_update` = `sakila`.`customer`.`last_update`)
and (`sakila`.`rental`.`customer_id` = `sakila`.`customer`.`customer_id`))
1 row in set (0.00 sec)
```

你有时会看到自然连接与左连接和右连接混合使用。以下是有效的连接语法：`NATURAL LEFT JOIN`、`NATURAL LEFT OUTER JOIN`、`NATURAL RIGHT JOIN`和`NATURAL RIGHT OUTER JOIN`。前两者是没有`ON`或`USING`子句的左连接，后两者是右连接。同样，尽量避免使用它们，但是如果你看到它们被使用，你应该理解它们的含义。

## 连接中的常数表达式

在我们迄今为止提供的所有连接示例中，我们都使用列标识符来定义连接条件。当你使用`USING`子句时，这是唯一可能的方法。当你在`WHERE`子句中定义连接条件时，这也是唯一有效的方法。然而，当你使用`ON`子句时，你实际上可以添加常数表达式。

让我们考虑一个例子，列出特定演员的所有电影：

```
mysql> `SELECT` `first_name``,` `last_name``,` `title`
    -> `FROM` `actor` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `JOIN` `film` `USING` `(``film_id``)`
    -> `WHERE` `actor_id` `=` `11``;`
```

```
+------------+-----------+--------------------+
| first_name | last_name | title              |
+------------+-----------+--------------------+
| ZERO       | CAGE      | CANYON STOCK       |
| ZERO       | CAGE      | DANCES NONE        |
| ...                                         |
| ZERO       | CAGE      | WEST LION          |
| ZERO       | CAGE      | WORKER TARZAN      |
+------------+-----------+--------------------+
25 rows in set (0.00 sec)
```

我们可以像这样将 `actor_id` 子句移入联接：

```
mysql> `SELECT` `first_name``,` `last_name``,` `title`
    -> `FROM` `actor` `JOIN` `film_actor`
    ->   `ON` `actor``.``actor_id` `=` `film_actor``.``actor_id`
    ->   `AND` `actor``.``actor_id` `=` `11`
    -> `JOIN` `film` `USING` `(``film_id``)``;`
```

```
+------------+-----------+--------------------+
| first_name | last_name | title              |
+------------+-----------+--------------------+
| ZERO       | CAGE      | CANYON STOCK       |
| ZERO       | CAGE      | DANCES NONE        |
| ...                                         |
| ZERO       | CAGE      | WEST LION          |
| ZERO       | CAGE      | WORKER TARZAN      |
+------------+-----------+--------------------+
25 rows in set (0.00 sec)
```

好吧，这当然很棒，但为什么呢？这比有适当的 `WHERE` 子句更具表现力吗？对这两个问题的答案是，在联接中的常量条件与 `WHERE` 子句中的条件的评估和解析方式不同。最好通过示例来展示这一点，但前面的查询不好。常量条件在联接中的影响最好通过左连接来显示。

请记住本节中关于左连接的查询：

```
mysql> `SELECT` `email``,` `name` `AS` `category_name``,` `COUNT``(``rental_id``)` `AS` `cnt`
    -> `FROM` `category` `cat` `LEFT` `JOIN` `film_category` `USING` `(``category_id``)`
    -> `LEFT` `JOIN` `inventory` `USING` `(``film_id``)`
    -> `LEFT` `JOIN` `rental` `USING` `(``inventory_id``)`
    -> `LEFT` `JOIN` `customer` `cs` `USING` `(``customer_id``)`
    -> `WHERE` `cs``.``email` `=` `'WESLEY.BULL@sakilacustomer.org'`
    -> `GROUP` `BY` `email``,` `category_name`
    -> `ORDER` `BY` `cnt` `DESC``;`
```

```
+--------------------------------+---------------+-----+
| email                          | category_name | cnt |
+--------------------------------+---------------+-----+
| WESLEY.BULL@sakilacustomer.org | Games         |   9 |
| WESLEY.BULL@sakilacustomer.org | Foreign       |   6 |
| ...                                                  |
| WESLEY.BULL@sakilacustomer.org | Comedy        |   1 |
| WESLEY.BULL@sakilacustomer.org | Sports        |   1 |
+--------------------------------+---------------+-----+
14 rows in set (0.01 sec)
```

如果我们将 `cs.email` 子句移到 `LEFT JOIN customer cs` 部分，我们将看到完全不同的结果：

```
mysql> `SELECT` `email``,` `name` `AS` `category_name``,` `COUNT``(``rental_id``)` `AS` `cnt`
    -> `FROM` `category` `cat` `LEFT` `JOIN` `film_category` `USING` `(``category_id``)`
    -> `LEFT` `JOIN` `inventory` `USING` `(``film_id``)`
    -> `LEFT` `JOIN` `rental` `USING` `(``inventory_id``)`
    -> `LEFT` `JOIN` `customer` `cs` `ON` `rental``.``customer_id` `=` `cs``.``customer_id`
    -> `AND` `cs``.``email` `=` `'WESLEY.BULL@sakilacustomer.org'`
    -> `GROUP` `BY` `email``,` `category_name`
    -> `ORDER` `BY` `cnt` `DESC``;`
```

```
+--------------------------------+-------------+------+
| email                          | name        | cnt  |
+--------------------------------+-------------+------+
| NULL                           | Sports      | 1178 |
| NULL                           | Animation   | 1164 |
| ...                                                 |
| NULL                           | Travel      |  834 |
| NULL                           | Music       |  829 |
| WESLEY.BULL@sakilacustomer.org | Games       |    9 |
| WESLEY.BULL@sakilacustomer.org | Foreign     |    6 |
| ...                                                 |
| WESLEY.BULL@sakilacustomer.org | Comedy      |    1 |
| NULL                           | Thriller    |    0 |
+--------------------------------+-------------+------+
31 rows in set (0.07 sec)
```

这很有趣！与其仅获取 Wesley 每个类别的租赁次数，我们还得到了每个其他人按类别分解的租赁次数。甚至包括我们新的到目前为空的惊悚类别。让我们试着理解这里发生了什么。

`WHERE` 子句的内容在解析和执行联接之后逻辑应用。我们告诉 MySQL，我们只需要来自我们加入的任何内容的行，其中 `cs.email` 列等于 `'WESLEY.BULL@sakilacustomer.org'`。事实上，MySQL 足够聪明，可以优化这种情况，并且实际上会开始执行计划，就好像使用了常规的内连接。当我们在 `LEFT JOIN customer` 子句中有 `cs.email` 条件时，我们告诉 MySQL，我们希望将 `customer` 表的列添加到我们到目前为止的结果集中（包括 `category`、`inventory` 和 `rental` 表），但仅在 `email` 列中出现特定值时。由于这是 `LEFT JOIN`，在未匹配的行中，我们在 `customer` 的每一列中得到 `NULL`。

重要的是要注意这种行为。

# 嵌套查询

自 MySQL 4.1 版本以来支持的嵌套查询是最难学习的。然而，它们提供了一种强大、有用且简洁的方式来表达短 SQL 语句中难以理解的信息需求。本节将从简单的示例开始解释它们，并引导您了解 `EXISTS` 和 `IN` 语句的更复杂特性。在本节结束时，您将完成本书关于查询数据的所有内容，并且应该能够理解您遇到的几乎任何 SQL 查询。

## 嵌套查询基础

你知道如何使用 `INNER JOIN` 找到参与特定电影的所有演员的名字：

```
mysql> `SELECT` `first_name``,` `last_name` `FROM`
    -> `actor` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `JOIN` `film` `USING` `(``film_id``)`
    -> `WHERE` `title` `=` `'ZHIVAGO CORE'``;`
```

```
+------------+-----------+
| first_name | last_name |
+------------+-----------+
| UMA        | WOOD      |
| NICK       | STALLONE  |
| GARY       | PENN      |
| SALMA      | NOLTE     |
| KENNETH    | HOFFMAN   |
| WILLIAM    | HACKMAN   |
+------------+-----------+
6 rows in set (0.00 sec)
```

但还有另一种方法，使用 *嵌套查询*：

```
mysql> `SELECT` `first_name``,` `last_name` `FROM`
    -> `actor` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `WHERE` `film_id` `=` `(``SELECT` `film_id` `FROM` `film`
    -> `WHERE` `title` `=` `'ZHIVAGO CORE'``)``;`
```

```
+------------+-----------+
| first_name | last_name |
+------------+-----------+
| UMA        | WOOD      |
| NICK       | STALLONE  |
| GARY       | PENN      |
| SALMA      | NOLTE     |
| KENNETH    | HOFFMAN   |
| WILLIAM    | HACKMAN   |
+------------+-----------+
6 rows in set (0.00 sec)
```

它被称为嵌套查询，因为一个查询在另一个查询内部。*内部查询* 或 *子查询* ——被嵌套的查询——写在括号中，你可以看到它确定了具有标题 `ZHIVAGO CORE` 的电影的 `film_id`。内部查询需要用括号括起来。*外部查询* 是首先列出的查询，在这里没有括号：你可以看到它通过与 `film_actor` 的 `JOIN` 来找到与子查询结果匹配的 `film_id`，从而找到演员的 `first_name` 和 `last_name`。因此，总体而言，内部查询找到 `film_id`，而外部查询则使用它来找到演员的姓名。每当使用嵌套查询时，都可以将其重写为几个单独的查询。让我们对先前的示例进行这样的操作，因为这可能有助于您理解正在发生的情况：

```
mysql> `SELECT` `film_id` `FROM` `film` `WHERE` `title` `=` `'ZHIVAGO CORE'``;`
```

```
+---------+
| film_id |
+---------+
|     998 |
+---------+
1 row in set (0.03 sec)
```

```
mysql> `SELECT` `first_name``,` `last_name`
    -> `FROM` `actor` `JOIN` `film_actor` `USING` `(``actor_id``)`
    -> `WHERE` `film_id` `=` `998``;`
```

```
+------------+-----------+
| first_name | last_name |
+------------+-----------+
| UMA        | WOOD      |
| NICK       | STALLONE  |
| GARY       | PENN      |
| SALMA      | NOLTE     |
| KENNETH    | HOFFMAN   |
| WILLIAM    | HACKMAN   |
+------------+-----------+
6 rows in set (0.00 sec)
```

那么，哪种方法更可取：嵌套还是非嵌套？答案并不容易。从性能角度来看，答案通常是 *不是*：嵌套查询很难优化，因此几乎总是比非嵌套的替代方法运行速度慢。

这是否意味着你应该避免嵌套？答案是否定的：有时这是你唯一的选择，如果你想要编写单个查询，有时嵌套查询可以回答其他方法难以解决的信息需求。此外，嵌套查询具有表达能力。一旦你熟悉这个概念，它们是一种非常可读的方式来展示查询如何被评估。事实上，许多 SQL 设计者倡导在向你展示的前几节中展示基于连接的替代方法之前教授嵌套查询。我们将向您展示在本节中嵌套查询如何既可读又强大的示例。

在我们开始讨论可以在嵌套查询中使用的关键字之前，让我们看一个例子，这个例子不容易在一个单独的查询中完成——至少不是没有 MySQL 的非标准，尽管普遍存在的 `LIMIT` 子句！假设你想知道客户最近租了哪部电影。根据我们之前学到的方法，你可以找到该客户在 `rental` 表中最近存储行的日期和时间：

```
mysql> `SELECT` `MAX``(``rental_date``)` `FROM` `rental`
    -> `JOIN` `customer` `USING` `(``customer_id``)`
    -> `WHERE` `email` `=` `'WESLEY.BULL@sakilacustomer.org'``;`
```

```
+---------------------+
| MAX(rental_date)    |
+---------------------+
| 2005-08-23 15:46:33 |
+---------------------+
1 row in set (0.01 sec)
```

你可以将输出用作另一个查询的输入来查找电影标题：

```
mysql> `SELECT` `title` `FROM` `film`
    -> `JOIN` `inventory` `USING` `(``film_id``)`
    -> `JOIN` `rental` `USING` `(``inventory_id``)`
    -> `JOIN` `customer` `USING` `(``customer_id``)`
    -> `WHERE` `email` `=` `'WESLEY.BULL@sakilacustomer.org'`
    -> `AND` `rental_date` `=` `'2005-08-23 15:46:33'``;`
```

```
+-------------+
| title       |
+-------------+
| KARATE MOON |
+-------------+
1 row in set (0.00 sec)
```

###### 注意

在 “用户变量” 中，我们将向您展示如何使用变量来避免在第二个查询中键入值。

使用嵌套查询，你可以一次完成两个步骤：

```
mysql> `SELECT` `title` `FROM` `film` `JOIN` `inventory` `USING` `(``film_id``)`
    -> `JOIN` `rental` `USING` `(``inventory_id``)`
    -> `WHERE` `rental_date` `=` `(``SELECT` `MAX``(``rental_date``)` `FROM` `rental`
    -> `JOIN` `customer` `USING` `(``customer_id``)`
    -> `WHERE` `email` `=` `'WESLEY.BULL@sakilacustomer.org'``)``;`
```

```
+-------------+
| title       |
+-------------+
| KARATE MOON |
+-------------+
1 row in set (0.01 sec)
```

您可以看到嵌套查询结合了两个先前的查询。与使用从先前查询中发现的常量日期和时间值不同，它直接执行子查询作为子查询。这是最简单的嵌套查询类型，返回一个 *标量操作数*——即单个值。

###### 提示

上一个示例使用了等号运算符（`=`）。你可以使用所有类型的比较运算符：`<`（小于）、`<=`（小于或等于）、`>`（大于）、`>=`（大于或等于）、`!=`（不等于）或 `<>`（不等于）。

## 任意、某些、所有、IN 和 NOT IN 子句

在我们开始展示更多嵌套查询的高级特性之前，我们需要在我们的示例中切换到一个新的数据库。不幸的是，`sakila`数据库过于规范化，无法有效地展示嵌套查询的全部功能。所以，让我们添加一个新的数据库来玩耍。

我们将安装的数据库是`employees`样本数据库。您可以在[MySQL 文档](https://oreil.ly/vODJG)或数据库的 GitHub 存储库中找到安装说明。使用`git`克隆存储库或下载最新版本（在撰写本文时为[1.0.7](https://oreil.ly/zW0E1)）。一旦准备好所需的文件，您需要运行两个命令。

第一个命令创建必要的结构并加载数据：

```
$ mysql -uroot -p < employees.sql
INFO
CREATING DATABASE STRUCTURE
INFO
storage engine: InnoDB
INFO
LOADING departments
INFO
LOADING employees
INFO
LOADING dept_emp
INFO
LOADING dept_manager
INFO
LOADING titles
INFO
LOADING salaries
data_load_time_diff
00:00:28
```

第二个命令验证安装是否正确：

```
$ mysql -uroot -p < test_employees_md5.sql
INFO
TESTING INSTALLATION
table_name      expected_records        expected_crc
departments     9       d1af5e170d2d1591d776d5638d71fc5f
dept_emp        331603  ccf6fe516f990bdaa49713fc478701b7
dept_manager    24      8720e2f0853ac9096b689c14664f847e
employees       300024  4ec56ab5ba37218d187cf6ab09ce1aa1
salaries        2844047 fd220654e95aea1b169624ffe3fca934
titles  443308  bfa016c472df68e70a03facafa1bc0a8
table_name      found_records           found_crc
departments     9       d1af5e170d2d1591d776d5638d71fc5f
dept_emp        331603  ccf6fe516f990bdaa49713fc478701b7
dept_manager    24      8720e2f0853ac9096b689c14664f847e
employees       300024  4ec56ab5ba37218d187cf6ab09ce1aa1
salaries        2844047 fd220654e95aea1b169624ffe3fca934
titles  443308  bfa016c472df68e70a03facafa1bc0a8
table_name      records_match   crc_match
departments     OK      ok
dept_emp        OK      ok
dept_manager    OK      ok
employees       OK      ok
salaries        OK      ok
titles  OK      ok
computation_time
00:00:25
summary result
CRC     OK
count   OK
```

一旦完成，您可以继续执行我们将提供的示例。

要连接到新数据库，可以像这样从命令行运行`mysql`（或者指定`employees`作为您选择的 MySQL 客户端的目标）：

```
$ mysql employees
```

或者在`mysql`提示符下执行以下操作来更改默认数据库：

```
mysql> `USE` `employees`
```

现在您可以继续前进了。

### 使用 ANY 和 IN

现在，您已经创建了示例表，可以使用`ANY`尝试一个示例。假设您想找到比最不经验的经理工作时间更长的助理工程师。您可以将此信息需求表达如下：

```
mysql> `SELECT` `emp_no``,` `first_name``,` `last_name``,` `hire_date`
    -> `FROM` `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Assistant Engineer'`
    -> `AND` `hire_date` `<` `ANY` `(``SELECT` `hire_date` `FROM`
    -> `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Manager'``)``;`
```

```
+--------+----------------+------------------+------------+
| emp_no | first_name     | last_name        | hire_date  |
+--------+----------------+------------------+------------+
|  10009 | Sumant         | Peac             | 1985-02-18 |
|  10066 | Kwee           | Schusler         | 1986-02-26 |
| ...                                                     |
| ...                                                     |
| 499958 | Srinidhi       | Theuretzbacher   | 1989-12-17 |
| 499974 | Shuichi        | Piazza           | 1989-09-16 |
+--------+----------------+------------------+------------+
10747 rows in set (0.20 sec)
```

结果表明符合这些条件的人有很多！子查询查找经理被聘用的日期：

```
mysql> `SELECT` `hire_date` `FROM`
    -> `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Manager'``;`
```

```
+------------+
| hire_date  |
+------------+
| 1985-01-01 |
| 1986-04-12 |
| ...        |
| 1991-08-17 |
| 1989-07-10 |
+------------+
24 rows in set (0.10 sec)
```

外部查询遍历每个标题为`Associate Engineer`的员工，如果他们的入职日期比子查询返回的任何值都早（更旧），则返回该工程师。因此，例如，`Sumant Peac`会被输出，因为`1985-02-18`比子查询返回的至少一个值更早（正如您可以看到，经理的第二个入职日期是`1986-04-12`）。`ANY`关键字的含义正是如此：如果在子查询返回的集合中的*任何*值上，前面的列或表达式为真，那么它就是真的。以这种方式使用，`ANY`有一个别名`SOME`，这是为了更清晰地读取某些查询作为英语表达式而包含的；它并不做任何不同的事情，您很少会看到它被使用。

`ANY`关键字在表达嵌套查询时提供了更大的表达力。确实，前面的查询是本节中第一个带有*列子查询*的嵌套查询——也就是说，子查询返回的结果是来自列的一个或多个值，而不是像前一节中那样的单一标量值。有了这个，您现在可以将外部查询的列值与子查询返回的一组值进行比较了。

考虑另一个使用`ANY`的例子。假设您想知道还有其他头衔的经理。您可以使用以下嵌套查询来执行此操作：

```
mysql> `SELECT` `emp_no``,` `first_name``,` `last_name`
    -> `FROM` `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Manager'`
    -> `AND` `emp_no` `=` `ANY` `(``SELECT` `emp_no` `FROM` `employees`
    -> `JOIN` `titles` `USING` `(``emp_no``)` `WHERE`
    -> `title` `<``>` `'Manager'``)``;`
```

```
+--------+-------------+--------------+
| emp_no | first_name  | last_name    |
+--------+-------------+--------------+
| 110022 | Margareta   | Markovitch   |
| 110039 | Vishwani    | Minakawa     |
| ...                                 |
| 111877 | Xiaobin     | Spinelli     |
| 111939 | Yuchang     | Weedman      |
+--------+-------------+--------------+
24 rows in set (0.11 sec)
```

`= ANY`会导致外部查询在`emp_no`等于子查询返回的工程师员工号之一时返回经理。`= ANY`短语具有别名`IN`，您将在嵌套查询中经常看到其使用。使用`IN`，前面的示例可以重写为：

```
mysql> `SELECT` `emp_no``,` `first_name``,` `last_name`
    -> `FROM` `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Manager'`
    -> `AND` `emp_no` `IN` `(``SELECT` `emp_no` `FROM` `employees`
    -> `JOIN` `titles` `USING` `(``emp_no``)` `WHERE`
    -> `title` `<``>` `'Manager'``)``;`
```

```
+--------+-------------+--------------+
| emp_no | first_name  | last_name    |
+--------+-------------+--------------+
| 110022 | Margareta   | Markovitch   |
| 110039 | Vishwani    | Minakawa     |
| ...                                 |
| 111877 | Xiaobin     | Spinelli     |
| 111939 | Yuchang     | Weedman      |
+--------+-------------+--------------+
24 rows in set (0.11 sec)
```

当然，对于这个特定的例子，您也可以使用连接查询。请注意，在这里我们必须使用`DISTINCT`，因为否则会返回 30 行。有些人拥有多个非工程师职称：

```
mysql> `SELECT` `DISTINCT` `emp_no``,` `first_name``,` `last_name`
    -> `FROM` `employees` `JOIN` `titles` `mgr` `USING` `(``emp_no``)`
    -> `JOIN` `titles` `nonmgr` `USING` `(``emp_no``)`
    -> `WHERE` `mgr``.``title` `=` `'Manager'`
    -> `AND` `nonmgr``.``title` `<``>` `'Manager'``;`
```

```
+--------+-------------+--------------+
| emp_no | first_name  | last_name    |
+--------+-------------+--------------+
| 110022 | Margareta   | Markovitch   |
| 110039 | Vishwani    | Minakawa     |
| ...                                 |
| 111877 | Xiaobin     | Spinelli     |
| 111939 | Yuchang     | Weedman      |
+--------+-------------+--------------+
24 rows in set (0.11 sec)
```

再次说明，嵌套查询在 MySQL 中表达力强但通常速度较慢，因此尽可能使用连接。

### 使用 ALL

假设您想要找出比所有经理更有经验的助理工程师——即比最有经验的经理更有经验的助理工程师。您可以使用`ALL`关键字代替`ANY`来完成这一操作：

```
mysql> `SELECT` `emp_no``,` `first_name``,` `last_name``,` `hire_date`
    -> `FROM` `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Assistant Engineer'`
    -> `AND` `hire_date` `<` `ALL` `(``SELECT` `hire_date` `FROM`
    -> `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Manager'``)``;`
```

```
Empty set (0.18 sec)
```

您可以看到没有答案。我们可以进一步检查数据，以确定经理和助理工程师的最早雇佣日期分别是什么：

```
mysql> `(``SELECT` `'Assistant Engineer'` `AS` `title``,`
    -> `MIN``(``hire_date``)` `AS` `mhd` `FROM` `employees`
    -> `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Assistant Engineer'``)`
    -> `UNION`
    -> `(``SELECT` `'Manager'` `title``,` `MIN``(``hire_date``)` `mhd` `FROM` `employees`
    -> `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Manager'``)``;`
```

```
+--------------------+------------+
| title              | mhd        |
+--------------------+------------+
| Assistant Engineer | 1985-02-01 |
| Manager            | 1985-01-01 |
+--------------------+------------+
2 rows in set (0.26 sec)
```

查看数据，我们看到第一位经理的雇佣日期是 1985 年 1 月 1 日，而第一位助理工程师是同年 2 月 1 日雇佣的。而`ANY`关键字返回满足至少一个条件的值（布尔 OR），`ALL`关键字仅返回所有条件都满足的值（布尔 AND）。

我们可以使用`NOT IN`别名代替`<> ANY`或`!= ANY`。让我们找出所有不是高级员工的经理：

```
mysql> `SELECT` `emp_no``,` `first_name``,` `last_name`
    -> `FROM` `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Manager'` `AND` `emp_no` `NOT` `IN`
    -> `(``SELECT` `emp_no` `FROM` `titles`
    -> `WHERE` `title` `=` `'Senior Staff'``)``;`
```

```
+--------+-------------+--------------+
| emp_no | first_name  | last_name    |
+--------+-------------+--------------+
| 110183 | Shirish     | Ossenbruggen |
| 110303 | Krassimir   | Wegerle      |
| ...                                 |
| 111400 | Arie        | Staelin      |
| 111692 | Tonny       | Butterworth  |
+--------+-------------+--------------+
15 rows in set (0.09 sec)
```

作为练习，尝试使用`ANY`语法和连接查询来编写此查询。

`ALL`关键字有一些技巧和陷阱：

+   如果对任何值为 false，则为 false。假设表`a`包含值为 14 的行，表`b`包含值为 16、1 和`NULL`。如果您检查`a`中的值是否大于`b`中所有值，您将得到`false`，因为 14 不大于 16。其他值为 1 和`NULL`并不重要。

+   如果对任何值都不为 false，则除非对所有值都为 true，否则它不为 true。假设表`a`再次包含 14，`b`包含 1 和`NULL`。如果你检查`a`中的值是否大于`b`中所有值，你会得到`UNKNOWN`（既非 true 也非 false），因为不能确定`NULL`是大于还是小于 14。

+   如果子查询中的表为空，则结果始终为 true。因此，如果`a`包含 14 且`b`为空，则在检查`a`中的值是否大于`b`中所有值时将得到`true`。

使用`ALL`关键字时，要特别注意表中可能存在列值为`NULL`的情况；在这种情况下，考虑不允许`NULL`值。同时，要小心处理空表。

### 编写行子查询

在前面的示例中，子查询返回单个标量值（例如`actor_id`）或来自一个列的一组值（例如所有`emp_no`的值）。本节描述了另一种类型的子查询，即*行子查询*，它可以处理来自多行的多列。

假设您想知道经理在同一日历年内是否有其他职位。要回答这个问题，您必须匹配员工编号和职务分配日期，或者更准确地说，年份。您可以将其写为连接查询：

```
mysql> `SELECT` `mgr``.``emp_no``,` `YEAR``(``mgr``.``from_date``)` `AS` `fd`
    -> `FROM` `titles` `AS` `mgr``,` `titles` `AS` `other`
    -> `WHERE` `mgr``.``emp_no` `=` `other``.``emp_no`
    -> `AND` `mgr``.``title` `=` `'Manager'`
    -> `AND` `mgr``.``title` `<``>` `other``.``title`
    -> `AND` `YEAR``(``mgr``.``from_date``)` `=` `YEAR``(``other``.``from_date``)``;`
```

```
+--------+------+
| emp_no | fd   |
+--------+------+
| 110765 | 1989 |
| 111784 | 1988 |
+--------+------+
2 rows in set (0.11 sec)
```

但是你也可以将其编写为嵌套查询：

```
mysql> `SELECT` `emp_no``,` `YEAR``(``from_date``)` `AS` `fd`
    -> `FROM` `titles` `WHERE` `title` `=` `'Manager'` `AND`
    -> `(``emp_no``,` `YEAR``(``from_date``)``)` `IN`
    -> `(``SELECT` `emp_no``,` `YEAR``(``from_date``)`
    -> `FROM` `titles` `WHERE` `title` `<``>` `'Manager'``)``;`
```

```
+--------+------+
| emp_no | fd   |
+--------+------+
| 110765 | 1989 |
| 111784 | 1988 |
+--------+------+
2 rows in set (0.12 sec)
```

您可以看到在这个嵌套查询中使用了不同的语法：括号内跟随`WHERE`语句的两个列名列表，并且内部查询返回两列。我们将在接下来解释这个语法。

行子查询语法允许您每行比较多个值。表达式`(emp_no, YEAR(from_date))`表示每行比较子查询的输出的两个值。您可以看到在`IN`关键字后面，子查询返回两个值，`emp_no`和`YEAR(from_date)`。因此，片段如下：

```
(emp_no, YEAR(from_date)) IN (SELECT emp_no, YEAR(from_date)
FROM titles WHERE title <> 'Manager')
```

匹配经理编号和起始年份到非经理编号和起始年份，并且当找到匹配时返回一个真值。结果是，如果找到匹配对，整个查询输出结果。这是一个典型的行子查询：它找到存在于两个表中的行。

为了进一步解释语法，让我们考虑另一个例子。假设您想查看特定员工是否是高级员工。您可以使用以下查询完成：

```
mysql> `SELECT` `first_name``,` `last_name`
    -> `FROM` `employees``,` `titles`
    -> `WHERE` `(``employees``.``emp_no``,` `first_name``,` `last_name``,` `title``)` `=`
    -> `(``titles``.``emp_no``,` `'Marjo'``,` `'Giarratana'``,` `'Senior Staff'``)``;`
```

```
+------------+------------+
| first_name | last_name  |
+------------+------------+
| Marjo      | Giarratana |
+------------+------------+
1 row in set (0.09 sec)
```

它不是一个嵌套查询，但它展示了新的行子查询语法如何工作。您可以看到查询在等号之前匹配列的列表`(employees.emp_no, first_name, last_name, title)`与等号之后列和值的列表`(titles.emp_no, 'Marjo', 'Giarratana', 'Senior Staff')`。因此，当`emp_no`值匹配时，员工的全名是`Marjo` `Giarratana`，职务是`Senior Staff`时，我们从查询中获取输出。我们不建议编写像这样的查询——而是使用带有多个`AND`条件的常规`WHERE`子句——但它确实说明了正在发生的情况。作为一项练习，尝试使用连接编写此查询。

行子查询要求列中的值的数量、顺序和类型匹配。例如，我们的前一个示例将一个`INT`匹配到一个`INT`，两个字符字符串匹配到两个字符字符串。

## 存在和不存在子句

您现在已经看到了三种类型的子查询：标量子查询、列子查询和行子查询。在本节中，您将学习第四种类型，即*相关子查询*，其中外部查询中使用的表在子查询中被引用。相关子查询通常与我们已经讨论过的`IN`语句一起使用，并且几乎总是与本节重点讨论的`EXISTS`和`NOT EXISTS`子句一起使用。

### EXISTS 和 NOT EXISTS 基础知识

在我们讨论相关子查询之前，让我们探讨一下`EXISTS`子句的作用。我们需要一个简单但奇怪的例子来介绍这个子句，因为我们暂时不讨论相关子查询。所以，这里是例子：假设您想在数据库中找到所有电影的计数，但仅当数据库处于活动状态时，您定义为只有在任何分支的至少一部电影已出租时才算活动。这是执行此操作的查询（在运行此查询之前不要忘记重新连接到`sakila`数据库——提示：使用`use <db>`命令）：

```
mysql> `SELECT` `COUNT``(``*``)` `FROM` `film`
    -> `WHERE` `EXISTS` `(``SELECT` `*` `FROM` `rental``)``;`
```

```
+----------+
| COUNT(*) |
+----------+
|     1000 |
+----------+
1 row in set (0.01 sec)
```

子查询返回`rental`表中的所有行。然而，重要的是它至少返回一行；行的内容不重要，行数多少也无关紧要，或者行只包含`NULL`值也无关紧要。因此，您可以将子查询视为真或假，在这种情况下它是真的，因为它生成了一些输出。当子查询为真时，使用`EXISTS`子句的外部查询返回一行。总体结果是计算`film`表中的所有行，因为对于每一行，子查询都是真的。

让我们尝试一个子查询不为真的查询。再次编造一个查询：这次，我们将输出数据库中所有电影的名称，但只有在存在特定电影时才这样做。以下是查询：

```
mysql> `SELECT` `title` `FROM` `film`
    -> `WHERE` `EXISTS` `(``SELECT` `*` `FROM` `film`
    -> `WHERE` `title` `=` `'IS THIS A MOVIE?'``)``;`
```

```
Empty set (0.00 sec)
```

由于子查询不为真——因为`IS THIS A MOVIE?`不在我们的数据库中——外部查询不会返回结果。

`NOT EXISTS`子句则相反。想象一下，如果您不在数据库中有一个特定电影，您希望获得所有演员的列表。以下是查询：

```
mysql> `SELECT` `*` `FROM` `actor` `WHERE` `NOT` `EXISTS`
    -> `(``SELECT` `*` `FROM` `film` `WHERE` `title` `=` `'ZHIVAGO CORE'``)``;`
```

```
Empty set (0.00 sec)
```

这次，内部查询为真，但`NOT EXISTS`子句否定了它以生成假。因为它是假的，外部查询不会产生结果。

您会注意到子查询以`SELECT * FROM film`开头。当您使用`EXISTS`子句时，实际上不管您在内部查询中选择什么，因为外部查询不使用它。您可以选择一个列、所有内容，甚至常量（如`SELECT 'cat' from film`），效果都是一样的。但传统上，大多数 SQL 作者按照约定会写`SELECT *`。

### 相关子查询

到目前为止，您可能很难想象您如何使用`EXISTS`和`NOT EXISTS`子句。本节展示了它们的实际用途，演示了您通常会看到的最高级别的嵌套查询类型。

让我们考虑您可能从`sakila`数据库中获取的实际信息类型。假设您想要所有曾租用过公司产品的员工列表，或者只是顾客。您可以轻松地使用联接查询来实现这一点，我们建议您在继续之前考虑一下。您还可以使用以下使用相关子查询的嵌套查询来实现：

```
mysql> `SELECT` `first_name``,` `last_name` `FROM` `staff`
    -> `WHERE` `EXISTS` `(``SELECT` `*` `FROM` `customer`
    -> `WHERE` `customer``.``first_name` `=` `staff``.``first_name`
    -> `AND` `customer``.``last_name` `=` `staff``.``last_name``)``;`
```

```
Empty set (0.01 sec)
```

没有输出，因为没有员工也是客户（或者禁止这样做，但我们会打破规则）。让我们添加一个与某个员工具有相同详细信息的客户：

```
mysql> `INSERT` `INTO` `customer``(``store_id``,` `first_name``,` `last_name``,`
    -> `email``,` `address_id``,` `create_date``)`
    -> `VALUES` `(``1``,` `'Mike'``,` `'Hillyer'``,`
    -> `'Mike.Hillyer@sakilastaff.com'``,` `3``,` `NOW``(``)``)``;`
```

```
Query OK, 1 row affected (0.02 sec)
```

再次尝试查询：

```
mysql> `SELECT` `first_name``,` `last_name` `FROM` `staff`
    -> `WHERE` `EXISTS` `(``SELECT` `*` `FROM` `customer`
    -> `WHERE` `customer``.``first_name` `=` `staff``.``first_name`
    -> `AND` `customer``.``last_name` `=` `staff``.``last_name``)``;`
```

```
+------------+-----------+
| first_name | last_name |
+------------+-----------+
| Mike       | Hillyer   |
+------------+-----------+
1 row in set (0.00 sec)
```

所以，这个查询有效；现在，我们只需要理解如何操作！

让我们检查一下我们之前示例中的子查询。您可以看到它只列出了 `FROM` 子句中的 `customer` 表，但它在 `WHERE` 子句中使用了 `staff` 表的一个列。如果您单独运行它，您会发现这是不允许的：

```
mysql> `SELECT` `*` `FROM` `customer` `WHERE` `customer``.``first_name` `=` `staff``.``first_name``;`
```

```
ERROR 1054 (42S22): Unknown column 'staff.first_name' in 'where clause'
```

但是，当作为子查询执行时是合法的，因为外部查询中列出的表允许在子查询中访问。因此，在此示例中，`staff.first_name` 和 `staff.last_name` 的当前值在外部查询中作为常量标量值提供给子查询，并与客户的名字进行比较。如果客户的名称与员工成员的名称匹配，则子查询为真，因此外部查询输出一行。考虑两种更清晰地说明这一点的情况：

+   外部查询正在处理的 `first_name` 和 `last_name` 是 `Jon` 和 `Stephens` 时，子查询为假，因为 `SELECT * FROM customer WHERE first_name = 'Jon' and last_name = 'Stephens';` 没有返回任何行，因此 Jon Stephens 的 staff 行不作为答案输出。

+   外部查询正在处理的 `first_name` 和 `last_name` 是 `Mike` 和 `Hillyer` 时，子查询为真，因为 `SELECT * FROM customer WHERE first_name = 'Mike' and last_name = 'Hillyer';` 返回至少一行。因此，Mike Hillyer 的 staff 行被返回。

您能看到相关子查询的威力吗？您可以在内部查询中使用外部查询的值来评估复杂的信息需求。

现在我们将探讨另一个使用 `EXISTS` 的例子。让我们试着找出我们至少拥有两份副本的所有电影的数量。为了使用 `EXISTS` 来做到这一点，我们需要思考内部和外部查询应该做什么。内部查询应该在我们检查的条件为真时产生结果；在这种情况下，当库存中有至少两行属于同一电影时，它应该产生输出。外部查询应在内部查询为真时增加计数器。以下是查询：

```
mysql> `SELECT` `COUNT``(``*``)` `FROM` `film` `WHERE` `EXISTS`
    -> `(``SELECT` `film_id` `FROM` `inventory`
    -> `WHERE` `inventory``.``film_id` `=` `film``.``film_id`
    -> `GROUP` `BY` `film_id` `HAVING` `COUNT``(``*``)` `>``=` `2``)``;`
```

```
+----------+
| COUNT(*) |
+----------+
|      958 |
+----------+
1 row in set (0.00 sec)
```

这是另一个不需要嵌套而使用联接就足够的查询示例，但为了解释的目的，我们坚持使用这个版本。看一下内部查询：您可以看到 `WHERE` 子句确保电影通过唯一的 `film_id` 进行匹配，并且子查询仅考虑当前电影的匹配行。`GROUP BY` 子句将这些行按电影进行分组，但仅当库存中至少有两个条目时。因此，内部查询仅在我们的库存中的当前电影至少有两行时才会产生输出。外部查询很简单：可以将其视为在子查询产生输出时递增计数器。

在我们继续讨论其他问题之前，这里再举一个例子。此例将在`employees`数据库中进行，所以请切换你的客户端。我们已经展示了一个使用`IN`并查找同时拥有其他职位的经理的查询：

```
mysql> `SELECT` `emp_no``,` `first_name``,` `last_name`
    -> `FROM` `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Manager'`
    -> `AND` `emp_no` `IN` `(``SELECT` `emp_no` `FROM` `employees`
    -> `JOIN` `titles` `USING` `(``emp_no``)` `WHERE`
    -> `title` `<``>` `'Manager'``)``;`
```

```
+--------+-------------+--------------+
| emp_no | first_name  | last_name    |
+--------+-------------+--------------+
| 110022 | Margareta   | Markovitch   |
| 110039 | Vishwani    | Minakawa     |
| ...                                 |
| 111877 | Xiaobin     | Spinelli     |
| 111939 | Yuchang     | Weedman      |
+--------+-------------+--------------+
24 rows in set (0.11 sec)
```

让我们重写查询以使用`EXISTS`。首先考虑子查询：当有一个与经理同名的`title`记录的员工时，它应该产生输出。

其次，考虑外部查询：它应该在内部查询产生输出时返回员工的姓名。以下是重写后的查询：

```
mysql> `SELECT` `emp_no``,` `first_name``,` `last_name`
    -> `FROM` `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Manager'`
    -> `AND` `EXISTS` `(``SELECT` `emp_no` `FROM` `titles`
    -> `WHERE` `titles``.``emp_no` `=` `employees``.``emp_no`
    -> `AND` `title` `<``>` `'Manager'``)``;`
```

```
+--------+-------------+--------------+
| emp_no | first_name  | last_name    |
+--------+-------------+--------------+
| 110022 | Margareta   | Markovitch   |
| 110039 | Vishwani    | Minakawa     |
| ...                                 |
| 111877 | Xiaobin     | Spinelli     |
| 111939 | Yuchang     | Weedman      |
+--------+-------------+--------------+
24 rows in set (0.09 sec)
```

再次可以看到子查询引用了来自外部查询的`emp_no`列。

关联子查询可以与任何嵌套查询类型一起使用。这里是先前的`IN`查询使用外部引用重写后的版本：

```
mysql> `SELECT` `emp_no``,` `first_name``,` `last_name`
    -> `FROM` `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Manager'`
    -> `AND` `emp_no` `IN` `(``SELECT` `emp_no` `FROM` `titles`
    -> `WHERE` `titles``.``emp_no` `=` `employees``.``emp_no`
    -> `AND` `title` `<``>` `'Manager'``)``;`
```

```
+--------+-------------+--------------+
| emp_no | first_name  | last_name    |
+--------+-------------+--------------+
| 110022 | Margareta   | Markovitch   |
| 110039 | Vishwani    | Minakawa     |
| ...                                 |
| 111877 | Xiaobin     | Spinelli     |
| 111939 | Yuchang     | Weedman      |
+--------+-------------+--------------+
24 rows in set (0.09 sec)
```

查询比需要的更复杂，但它说明了这个想法。你可以看到子查询中的`emp_no`引用了外部查询的`employees`表。

如果查询只返回一行，也可以重写为使用等号而不是`IN`：

```
mysql> `SELECT` `emp_no``,` `first_name``,` `last_name`
    -> `FROM` `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Manager'`
    -> `AND` `emp_no` `=` `(``SELECT` `emp_no` `FROM` `titles`
    -> `WHERE` `titles``.``emp_no` `=` `employees``.``emp_no`
    -> `AND` `title` `<``>` `'Manager'``)``;`
```

```
ERROR 1242 (21000): Subquery returns more than 1 row
```

在这种情况下，它不起作用，因为子查询返回了多个标量值。让我们缩小范围：

```
mysql> `SELECT` `emp_no``,` `first_name``,` `last_name`
    -> `FROM` `employees` `JOIN` `titles` `USING` `(``emp_no``)`
    -> `WHERE` `title` `=` `'Manager'`
    -> `AND` `emp_no` `=` `(``SELECT` `emp_no` `FROM` `titles`
    -> `WHERE` `titles``.``emp_no` `=` `employees``.``emp_no`
    -> `AND` `title` `=` `'Senior Engineer'``)``;`
```

```
+--------+------------+-----------+
| emp_no | first_name | last_name |
+--------+------------+-----------+
| 110344 | Rosine     | Cools     |
| 110420 | Oscar      | Ghazalie  |
| 110800 | Sanjoy     | Quadeer   |
+--------+------------+-----------+
3 rows in set (0.10 sec)
```

现在它起作用了——每个名称只有一个经理和高级工程师头衔——所以列子查询操作符`IN`不再必要。当然，如果标题重复（例如，如果一个人在职位之间来回切换），你需要使用`IN`、`ANY`或`ALL`。

## FROM 子句中的嵌套查询

我们展示的技术都在`WHERE`子句中使用了嵌套查询。本节向您展示了当您想要操作查询中使用的数据源时，如何在`FROM`子句中替代地使用它们。

在`employees`数据库中，`salaries`表存储了员工 ID 和年薪。例如，如果您想找到月薪，可以在查询中进行一些数学计算。在这种情况下的一个选项是使用子查询：

```
mysql> `SELECT` `emp_no``,` `monthly_salary` `FROM`
    -> `(``SELECT` `emp_no``,` `salary``/``12` `AS` `monthly_salary` `FROM` `salaries``)` `AS` `ms`
    -> `LIMIT` `5``;`
```

```
+--------+----------------+
| emp_no | monthly_salary |
+--------+----------------+
|  10001 |      5009.7500 |
|  10001 |      5175.1667 |
|  10001 |      5506.1667 |
|  10001 |      5549.6667 |
|  10001 |      5580.0833 |
+--------+----------------+
5 rows in set (0.00 sec)
```

注意关注`FROM`子句后面的内容。子查询使用`salaries`表并返回两列：第一列是`emp_no`；第二列别名为`monthly_salary`，是`salary`值除以 12。外部查询很简单：它只返回通过子查询创建的`emp_no`和`monthly_salary`值。请注意，我们为子查询添加了表别名`ms`。当我们将子查询作为表使用时，这个“派生表”必须有一个别名，即使我们在查询中没有使用别名。如果省略别名，MySQL 会报错：

```
mysql> `SELECT` `emp_no``,` `monthly_salary` `FROM`
    -> `(``SELECT` `emp_no``,` `salary``/``12` `AS` `monthly_salary` `FROM` `salaries``)`
    -> `LIMIT` `5``;`
```

```
ERROR 1248 (42000): Every derived table must have its own alias
```

这里有另一个例子，现在在`sakila`数据库中。假设我们想通过租借来计算电影带来的平均收益，或者称之为平均总收益。让我们先考虑子查询。它应该返回每部电影的支付总和。然后，外部查询应该对这些值求平均以得出答案。以下是查询：

```
mysql> `SELECT` `AVG``(``gross``)` `FROM`
    -> `(``SELECT` `SUM``(``amount``)` `AS` `gross`
    -> `FROM` `payment` `JOIN` `rental` `USING` `(``rental_id``)`
    -> `JOIN` `inventory` `USING` `(``inventory_id``)`
    -> `JOIN` `film` `USING` `(``film_id``)`
    -> `GROUP` `BY` `film_id``)` `AS` `gross_amount``;`
```

```
+------------+
| AVG(gross) |
+------------+
|  70.361754 |
+------------+
1 row in set (0.05 sec)
```

你可以看到内部查询将 `payment`、`rental`、`inventory` 和 `film` 进行连接，并按电影分组销售，以便你可以获取每部电影的总和。如果单独运行它，会发生以下情况：

```
mysql> `SELECT` `SUM``(``amount``)` `AS` `gross`
    -> `FROM` `payment` `JOIN` `rental` `USING` `(``rental_id``)`
    -> `JOIN` `inventory` `USING` `(``inventory_id``)`
    -> `JOIN` `film` `USING` `(``film_id``)`
    -> `GROUP` `BY` `film_id``;`
```

```
+--------+
| gross  |
+--------+
|  36.77 |
|  52.93 |
|  37.88 |
|    ... |
|  14.91 |
|  73.83 |
| 214.69 |
+--------+
958 rows in set (0.08 sec)
```

现在，外部查询获取这些总和，它们被别名为 `gross`，并对它们进行平均处理，得出最终结果。这个查询是应用两个聚合函数到一组数据的典型方式。你不能像 `AVG(SUM(amount))` 这样级联应用聚合函数：

```
mysql> `SELECT` `AVG``(``SUM``(``amount``)``)` `AS` `avg_gross`
    -> `FROM` `payment` `JOIN` `rental` `USING` `(``rental_id``)`
    -> `JOIN` `inventory` `USING` `(``inventory_id``)`
    -> `JOIN` `film` `USING` `(``film_id``)` `GROUP` `BY` `film_id``;`
```

```
ERROR 1111 (HY000): Invalid use of group function
```

在 `FROM` 子句中使用子查询，可以返回标量值、一组列值、多行甚至整个表格。但是，不能使用相关子查询，也就是说，不能引用未在子查询中显式列出的表或列。还要注意，必须使用 `AS` 关键字为整个子查询起别名，并给它一个名称，即使在查询中没有使用该名称。

## JOIN 中的嵌套查询

我们将展示的最后一种嵌套查询用途，但不是最不实用的，是在连接中使用它们。在这种用法中，子查询的结果基本上形成一个新表，并可以在我们讨论过的任何连接类型中使用。

作为这一点的例子，让我们回到列出某个客户租用的每个类别的电影数量的查询。记住，我们在仅使用连接时编写该查询时遇到了问题：对于我们的客户没有租用的类别，我们没有得到零计数。这是该查询：

```
mysql> `SELECT` `cat``.``name` `AS` `category_name``,` `COUNT``(``cat``.``category_id``)` `AS` `cnt`
    -> `FROM` `category` `AS` `cat` `LEFT` `JOIN` `film_category` `USING` `(``category_id``)`
    -> `LEFT` `JOIN` `inventory` `USING` `(``film_id``)`
    -> `LEFT` `JOIN` `rental` `USING` `(``inventory_id``)`
    -> `JOIN` `customer` `AS` `cs` `ON` `rental``.``customer_id` `=` `cs``.``customer_id`
    -> `WHERE` `cs``.``email` `=` `'WESLEY.BULL@sakilacustomer.org'`
    -> `GROUP` `BY` `category_name` `ORDER` `BY` `cnt` `DESC``;`
```

```
+-------------+-----+
| name        | cnt |
+-------------+-----+
| Games       |   9 |
| Foreign     |   6 |
| ...               |
| ...               |
| Comedy      |   1 |
| Sports      |   1 |
+-------------+-----+
14 rows in set (0.00 sec)
```

现在我们知道了子查询和连接，以及子查询可以在连接中使用，我们可以轻松完成任务。这是我们的新查询：

```
mysql> `SELECT` `cat``.``name` `AS` `category_name``,` `cnt`
    -> `FROM` `category` `AS` `cat`
    -> `LEFT` `JOIN` `(``SELECT` `cat``.``name``,` `COUNT``(``cat``.``category_id``)` `AS` `cnt`
    ->    `FROM` `category` `AS` `cat`
    ->    `LEFT` `JOIN` `film_category` `USING` `(``category_id``)`
    ->    `LEFT` `JOIN` `inventory` `USING` `(``film_id``)`
    ->    `LEFT` `JOIN` `rental` `USING` `(``inventory_id``)`
    ->    `JOIN` `customer` `cs` `ON` `rental``.``customer_id` `=` `cs``.``customer_id`
    ->    `WHERE` `cs``.``email` `=` `'WESLEY.BULL@sakilacustomer.org'`
    ->    `GROUP` `BY` `cat``.``name``)` `customer_cat` `USING` `(``name``)`
    -> `ORDER` `BY` `cnt` `DESC``;`
```

```
+-------------+------+
| name        | cnt  |
+-------------+------+
| Games       |    9 |
| Foreign     |    6 |
| ...                |
| Children    |    1 |
| Sports      |    1 |
| Sci-Fi      | NULL |
| Action      | NULL |
| Thriller    | NULL |
+-------------+------+
17 rows in set (0.01 sec)
```

最后，我们得到所有类别的显示，并且对于那些没有租赁的类别，我们得到 `NULL` 值。让我们回顾一下我们的新查询中发生了什么。子查询被别名为 `customer_cat`，是我们之前查询的不带 `ORDER BY` 子句的版本。因此，我们知道它将返回：Wesley 租用物品的类别中的 14 行，以及每个类别的租赁数量。接下来，使用 `LEFT JOIN` 将该信息连接到 `category` 表的完整类别列表中。`category` 表驱动连接，因此它将选择每行。我们使用与子查询输出和 `category` 表列之间匹配的 `name` 列将子查询连接起来。

我们在这里展示的技术是非常强大的；然而，像所有的子查询一样，它也有代价。当子查询出现在连接子句中时，MySQL 无法像优化整个查询那样高效地操作。

# 用户变量

通常你会希望保存从查询返回的值。你可能希望这样做是为了能够轻松地在后续查询中使用一个值。你可能也只是想为稍后显示保存一个结果。在这两种情况下，用户变量可以解决问题：它们允许你存储一个结果并在以后使用它。

让我们用一个简单的例子来说明用户变量。下面的查询找到一部电影的标题，并将结果保存在一个用户变量中：

```
mysql> `SELECT` `@``film``:``=``title` `FROM` `film` `WHERE` `film_id` `=` `1``;`
```

```
+------------------+
| @film:=title     |
+------------------+
| ACADEMY DINOSAUR |
+------------------+
1 row in set, 1 warning (0.00 sec)
```

用户变量名为`film`，通过前置的`@`字符表示为用户变量。值是使用`:=`运算符赋值的。您可以使用以下非常简短的查询打印用户变量的内容：

```
mysql> `SELECT` `@``film``;`
```

```
+------------------+
| @film            |
+------------------+
| ACADEMY DINOSAUR |
+------------------+
1 row in set (0.00 sec)
```

您可能已经注意到了警告——那是关于什么的？

```
mysql> `SELECT` `@``film``:``=``title` `FROM` `film` `WHERE` `film_id` `=` `1``;`
mysql> `SHOW` `WARNINGS``\``G`
```

```
*************************** 1\. row ***************************
  Level: Warning
   Code: 1287
Message: Setting user variables within expressions is deprecated
and will be removed in a future release. Consider alternatives:
'SET variable=expression, ...', or
'SELECT expression(s) INTO variables(s)'.
1 row in set (0.00 sec)
```

让我们讨论提出的两种备选方案。首先，我们仍然可以在`SET`语句内执行嵌套查询：

```
mysql> `SET` `@``film` `:``=` `(``SELECT` `title` `FROM` `film` `WHERE` `film_id` `=` `1``)``;`
```

```
Query OK, 0 rows affected (0.00 sec)
```

```
mysql> `SELECT` `@``film``;`
```

```
+------------------+
| @film            |
+------------------+
| ACADEMY DINOSAUR |
+------------------+
1 row in set (0.00 sec)
```

其次，我们可以使用`SELECT INTO`语句：

```
mysql> `SELECT` `title` `INTO` `@``film` `FROM` `film` `WHERE` `film_id` `=` `1``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

```
mysql> `SELECT` `@``film``;`
```

```
+------------------+
| @film            |
+------------------+
| ACADEMY DINOSAUR |
+------------------+
1 row in set (0.00 sec)
```

您可以使用`SET`语句显式设置变量而不使用`SELECT`。假设您想将计数器初始化为零：

```
mysql> `SET` `@``counter` `:``=` `0``;`
```

```
Query OK, 0 rows affected (0.00 sec)
```

`:=`是可选的，您可以改用`=`并混合使用它们。您应该用逗号分隔多个赋值或将每个赋值放在独立的语句中：

```
mysql> `SET` `@``counter` `=` `0``,` `@``age` `:``=` `23``;`
```

```
Query OK, 0 rows affected (0.00 sec)
```

`SET`的替代语法是`SELECT INTO`。您可以初始化单个变量：

```
mysql> `SELECT` `0` `INTO` `@``counter``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

或同时初始化多个变量：

```
mysql> `SELECT` `0``,` `23` `INTO` `@``counter``,` `@``age``;`
```

```
Query OK, 1 row affected (0.00 sec)
```

用户变量的最常见用途是保存结果并稍后使用。您可能还记得本章早些时候的以下示例，我们用它来推动嵌套查询（对于此问题肯定是更好的解决方案）。在这里，我们想找出特定客户最近租赁的电影的名称：

```
mysql> `SELECT` `MAX``(``rental_date``)` `FROM` `rental`
    -> `JOIN` `customer` `USING` `(``customer_id``)`
    -> `WHERE` `email` `=` `'WESLEY.BULL@sakilacustomer.org'``;`
```

```
+---------------------+
| MAX(rental_date)    |
+---------------------+
| 2005-08-23 15:46:33 |
+---------------------+
1 row in set (0.01 sec)
```

```
mysql> `SELECT` `title` `FROM` `film`
    -> `JOIN` `inventory` `USING` `(``film_id``)`
    -> `JOIN` `rental` `USING` `(``inventory_id``)`
    -> `JOIN` `customer` `USING` `(``customer_id``)`
    -> `WHERE` `email` `=` `'WESLEY.BULL@sakilacustomer.org'`
    -> `AND` `rental_date` `=` `'2005-08-23 15:46:33'``;`
```

```
+-------------+
| title       |
+-------------+
| KARATE MOON |
+-------------+
1 row in set (0.00 sec)
```

您可以使用用户变量保存结果以便输入到后续查询中。以下是使用此方法重写的相同查询对：

```
mysql> `SELECT` `MAX``(``rental_date``)` `INTO` `@``recent` `FROM` `rental`
    -> `JOIN` `customer` `USING` `(``customer_id``)`
    -> `WHERE` `email` `=` `'WESLEY.BULL@sakilacustomer.org'``;`
```

```
1 row in set (0.01 sec)
```

```
mysql> `SELECT` `title` `FROM` `film`
    -> `JOIN` `inventory` `USING` `(``film_id``)`
    -> `JOIN` `rental` `USING` `(``inventory_id``)`
    -> `JOIN` `customer` `USING` `(``customer_id``)`
    -> `WHERE` `email` `=` `'WESLEY.BULL@sakilacustomer.org'`
    -> `AND` `rental_date` `=` `@``recent``;`
```

```
+-------------+
| title       |
+-------------+
| KARATE MOON |
+-------------+
1 row in set (0.00 sec)
```

这可以避免复制粘贴，并且确实有助于避免输入错误。

这里是关于使用用户变量的一些指导方针：

+   用户变量是唯一于连接的：您创建的变量对其他人不可见，两个不同的连接可以具有相同名称的两个不同变量。

+   变量名称可以是字母数字字符串，还可以包括句点（`.`）、下划线（`_`）和美元符号（`$`）字符。

+   在 MySQL 版本 5 之前，变量名称区分大小写，在版本 5 及以后则不区分大小写。

+   任何未初始化的变量都具有值`NULL`；您还可以手动将变量设置为`NULL`。

+   变量在连接关闭时被销毁。

+   您应该避免尝试将值分配给变量并将变量用作`SELECT`查询的一部分。此做法的两个原因是，新值可能不能立即在同一语句中使用，以及变量的类型在首次分配时设置；在同一 SQL 语句中稍后尝试将其用作不同类型可能会导致意外结果。

    让我们更详细地查看使用新变量`@fid`的第一个问题。由于我们之前没有使用过此变量，它是空的。现在，让我们显示具有`inventory`表中条目的电影的`film_id`。我们将`film_id`分配给`@fid`变量，而不是直接显示它。我们的查询将显示变量三次——一次是在赋值操作之前，一次作为赋值操作的一部分，一次在赋值操作之后：

    ```
    mysql> `SELECT` `@``fid``,` `@``fid``:``=``film``.``film_id``,` `@``fid` `FROM` `film``,` `inventory`
        -> `WHERE` `inventory``.``film_id` `=` `@``fid``;`
    ```

    ```
    Empty set, 1 warning (0.16 sec)
    ```

    这只返回一个废弃警告；因为变量中一开始没有任何内容，`WHERE` 子句尝试查找空的 `inventory.film_id` 值。如果我们修改查询以将 `film.film_id` 作为 `WHERE` 子句的一部分，事情将按预期工作：

    ```
    mysql> `SELECT` `@``fid``,` `@``fid``:``=``film``.``film_id``,` `@``fid` `FROM` `film``,` `inventory`
        -> `WHERE` `inventory``.``film_id` `=` `film``.``film_id` `LIMIT` `20``;`
    ```

    ```
    +------+--------------------+------+
    | @fid | @fid:=film.film_id | @fid |
    +------+--------------------+------+
    | NULL |                  1 | 1    |
    | 1    |                  1 | 1    |
    | 1    |                  1 | 1    |
    | ...                              |
    | 4    |                  4 | 4    |
    | 4    |                  4 | 4    |
    +------+--------------------+------+
    20 rows in set, 1 warning (0.00 sec)
    ```

    现在如果 `@fid` 不为空，初始查询将产生一些结果：

    ```
    mysql> `SELECT` `@``fid``,` `@``fid``:``=``film``.``film_id``,` `@``fid` `FROM` `film``,` `inventory`
        -> `WHERE` `inventory``.``film_id` `=` `film``.``film_id` `LIMIT` `20``;`
    ```

    ```
    +------+--------------------+------+
    | @fid | @fid:=film.film_id | @fid |
    +------+--------------------+------+
    |    4 |                  1 |    1 |
    |    1 |                  1 |    1 |
    |  ...                             |
    |    4 |                  4 |    4 |
    |    4 |                  4 |    4 |
    +------+--------------------+------+
    20 rows in set, 1 warning (0.00 sec)
    ```

    最好避免这种行为不被保证且因此不可预测的情况。
