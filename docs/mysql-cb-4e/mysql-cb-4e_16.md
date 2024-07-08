# 第十六章：使用联接和子查询

# 16.0 介绍

大多数在前几章中的查询使用了单个表，但是对于任何稍微复杂的应用程序，您可能需要使用多个表。有些问题简单地无法使用单个表来回答，而当您将多个来源的信息结合在一起时，关系数据库的真正威力就显现出来：

+   结合表中的行以获得比仅从单个表中获得的信息更全面的信息

+   保持多阶段操作的中间结果

+   根据另一个表中的信息修改一个表中的行

本章重点介绍使用多个表的两种类型语句：表之间的联接和嵌套一个`SELECT`中的子查询。它涵盖以下主题：

比较表以查找匹配项或不匹配项

要解决这些问题，您应该了解适用哪些类型的联接。内联接显示一个表中与另一个表中的行匹配的行。外部联接显示匹配的行，但也找到一个表中未与另一个表中的行匹配的行。

删除不匹配的行

如果两个数据集相关但不完美，您可以确定哪些行不匹配并根据需要删除它们。

将表与自身进行比较

有些问题要求将表与自身进行比较。这类似于在不同表之间执行联接，但您必须使用表别名来消除表引用的歧义。

生成候选详细和多对多关系

当一个表中的每个项目可以与另一个表中的多个项目匹配，或者当每个表中的任一项目可以与另一个表中的多个项目匹配时，联接可以生成列表或摘要。

本章使用的创建表的脚本位于`recipes`分发的*tables*目录中。要查找实现本章讨论的技术的脚本，请查看*joins*目录。

# 16.1 在表之间查找匹配项

## 问题

您需要执行需要来自多个表的信息的任务。

## 解决方案

使用联接——即在其`FROM`子句中列出多个表并告诉 MySQL 如何匹配它们的查询。

## 讨论

联接背后的基本思想是它将一个表中的行与一个或多个其他表中的行进行匹配。当每个表只回答您感兴趣问题的一部分时，联接使您能够合并来自多个表的信息。

产生所有可能的行组合的完全连接称为笛卡尔积。例如，将一个有 100 行的表中的每一行与一个有 200 行的表中的每一行连接，产生的结果包含 100 × 200 = 20,000 行。对于更大的表或连接超过两个表的情况，笛卡尔积的结果集很容易变得巨大，因此连接通常包括一个 `ON` 或 `USING` 比较子句，仅生成表之间所需的匹配（这要求每个表都有一个或多个共同的信息列，逻辑上将它们联系在一起）。还可以包括一个 `WHERE` 子句，用于限制要选择的连接行。每个子句都可以缩小查询的焦点。

此处介绍连接语法并展示连接如何回答特定类型的问题，当您寻找表之间的匹配时。其他部分展示如何识别表之间的不匹配（参见 Recipe 16.2）以及如何将表与自身进行比较（参见 Recipe 16.4）。这些示例假定您拥有一个艺术收藏，并使用以下两个表记录您的收购信息。`artist` 列出了那些您希望收藏其作品的画家，而 `painting` 则列出了您实际购买的每幅绘画作品：

```
CREATE TABLE artist
(
  a_id  INT UNSIGNED NOT NULL AUTO_INCREMENT, # artist ID
  name  VARCHAR(30) NOT NULL,                 # artist name
  PRIMARY KEY (a_id),
  UNIQUE (name)
);

CREATE TABLE painting
(
  a_id  INT UNSIGNED NOT NULL,                # artist ID
  p_id  INT UNSIGNED NOT NULL AUTO_INCREMENT, # painting ID
  title VARCHAR(100) NOT NULL,                # title of painting
  state VARCHAR(2) NOT NULL,                  # state where purchased
  price INT UNSIGNED,                         # purchase price (dollars)
  INDEX (a_id),
  PRIMARY KEY (p_id)
);
```

您刚刚开始收藏，因此表中只包含少量行：

```
mysql> `SELECT * FROM artist ORDER BY a_id;`
+------+----------+
| a_id | name     |
+------+----------+
|    1 | Da Vinci |
|    2 | Monet    |
|    3 | Van Gogh |
|    4 | Renoir   |
+------+----------+
mysql> `SELECT * FROM painting ORDER BY a_id, p_id;`
+------+------+-------------------+-------+-------+
| a_id | p_id | title             | state | price |
+------+------+-------------------+-------+-------+
|    1 |    1 | The Last Supper   | IN    |    34 |
|    1 |    2 | Mona Lisa         | MI    |    87 |
|    3 |    3 | Starry Night      | KY    |    48 |
|    3 |    4 | The Potato Eaters | KY    |    67 |
|    4 |    5 | Les Deux Soeurs   | NE    |    64 |
+------+------+-------------------+-------+-------+
```

`painting` 表中 `price` 列中的低值揭示了您的收藏实际上只包含廉价的仿制品，而不是原作。好吧，没关系：谁能负担得起原作呢？

每张表包含关于您收藏的部分信息。例如，`artist` 表并未告诉您每位艺术家创作了哪些绘画作品，而 `painting` 表则列出了艺术家的 ID，但没有他们的名字。要使用这两个表中的信息，编写一个执行连接的查询。在 `FROM` 关键字后面命名两个或更多表来执行连接。在输出列列表中，使用 `*` 从所有表中选择所有列，*`tbl_name`*`.*` 从给定表中选择所有列，或者根据这些列的连接表或表达式命名特定列。

最简单的连接涉及两个表，并从每个表中选择所有列。下面连接 `artist` 和 `painting` 表展示了这一点（`ORDER` `BY` 子句使结果更易读）：

```
mysql> `SELECT * FROM artist INNER JOIN painting ORDER BY artist.a_id;`
+------+----------+------+------+-------------------+-------+-------+
| a_id | name     | a_id | p_id | title             | state | price |
+------+----------+------+------+-------------------+-------+-------+
|    1 | Da Vinci |    1 |    1 | The Last Supper   | IN    |    34 |
|    1 | Da Vinci |    3 |    3 | Starry Night      | KY    |    48 |
|    1 | Da Vinci |    4 |    5 | Les Deux Soeurs   | NE    |    64 |
|    1 | Da Vinci |    1 |    2 | Mona Lisa         | MI    |    87 |
|    1 | Da Vinci |    3 |    4 | The Potato Eaters | KY    |    67 |
|    2 | Monet    |    1 |    2 | Mona Lisa         | MI    |    87 |
|    2 | Monet    |    3 |    4 | The Potato Eaters | KY    |    67 |
|    2 | Monet    |    1 |    1 | The Last Supper   | IN    |    34 |
|    2 | Monet    |    3 |    3 | Starry Night      | KY    |    48 |
|    2 | Monet    |    4 |    5 | Les Deux Soeurs   | NE    |    64 |
|    3 | Van Gogh |    1 |    2 | Mona Lisa         | MI    |    87 |
|    3 | Van Gogh |    3 |    4 | The Potato Eaters | KY    |    67 |
|    3 | Van Gogh |    1 |    1 | The Last Supper   | IN    |    34 |
|    3 | Van Gogh |    3 |    3 | Starry Night      | KY    |    48 |
|    3 | Van Gogh |    4 |    5 | Les Deux Soeurs   | NE    |    64 |
|    4 | Renoir   |    1 |    1 | The Last Supper   | IN    |    34 |
|    4 | Renoir   |    3 |    3 | Starry Night      | KY    |    48 |
|    4 | Renoir   |    4 |    5 | Les Deux Soeurs   | NE    |    64 |
|    4 | Renoir   |    1 |    2 | Mona Lisa         | MI    |    87 |
|    4 | Renoir   |    3 |    4 | The Potato Eaters | KY    |    67 |
+------+----------+------+------+-------------------+-------+-------+
```

`INNER JOIN` 生成将一个表中的值与另一个表中的值组合的结果。前面的查询未指定行匹配的任何限制，因此连接生成所有行组合（即笛卡尔积）。这个结果说明为什么这样的连接通常是无用的：它产生大量无意义的输出。显然，您不会维护这些表以将每位艺术家与每幅绘画作品匹配。

###### 小贴士

在 MySQL 中，`JOIN`、`CROSS JOIN` 和 `INNER JOIN` 是语法上的等价物，并可以互换使用。可以在我们使用 `INNER JOIN` 的所有地方使用 `CROSS JOIN` 或简单的 `JOIN`。

为了有意义地回答问题，通过包含适当的连接条件仅生成相关的匹配。例如，要生成包含艺术家名称的画作列表，使用简单的 `WHERE` 子句将来自两个表的行关联起来，该子句基于艺术家 ID 列的值进行匹配，该列是两个表共同的并用于链接它们的列：

```
mysql> `SELECT * FROM artist INNER JOIN painting`
    -> `WHERE artist.a_id = painting.a_id`
    -> `ORDER BY artist.a_id;`
+------+----------+------+------+-------------------+-------+-------+
| a_id | name     | a_id | p_id | title             | state | price |
+------+----------+------+------+-------------------+-------+-------+
|    1 | Da Vinci |    1 |    1 | The Last Supper   | IN    |    34 |
|    1 | Da Vinci |    1 |    2 | Mona Lisa         | MI    |    87 |
|    3 | Van Gogh |    3 |    3 | Starry Night      | KY    |    48 |
|    3 | Van Gogh |    3 |    4 | The Potato Eaters | KY    |    67 |
|    4 | Renoir   |    4 |    5 | Les Deux Soeurs   | NE    |    64 |
+------+----------+------+------+-------------------+-------+-------+
```

`WHERE` 子句中的列名包括表限定符，以明确指出要比较的 `a_id` 值。结果显示了谁画了每幅画，反过来，每位艺术家的哪些画作在您的收藏中。

另一种编写相同连接的方式使用 `ON` 子句指示匹配条件：

```
SELECT * FROM artist INNER JOIN painting
ON artist.a_id = painting.a_id
ORDER BY artist.a_id;
```

特殊情况下，当两个表中具有相同名称的列进行等值比较时，可以使用 `INNER JOIN` 结合 `USING` 子句。这不需要表限定符，并且每个连接的列只命名一次：

```
SELECT * FROM artist INNER JOIN painting
USING (a_id)
ORDER BY a_id;
```

对于 `SELECT *` 查询，`USING` 形式产生的结果与 `ON` 形式不同：它仅返回每个连接列的一个实例，因此 `a_id` 只出现一次，而不是两次。

`ON`、`USING` 或 `WHERE` 中的任何一个都可以包括比较，那么如何确定每个子句中放置哪些连接条件呢？作为一个经验法则，通常使用 `ON` 或 `USING` 指定如何连接表，使用 `WHERE` 子句限制要选择的连接行。例如，要基于 `a_id` 列连接表，但仅选择在肯塔基州获取的画作的行，请使用 `ON`（或 `USING`）子句匹配两个表中的行，并使用 `WHERE` 子句测试 `state` 列：

```
mysql> `SELECT * FROM artist INNER JOIN painting`
    -> `ON artist.a_id = painting.a_id`
    -> `WHERE painting.state = 'KY';`
+------+----------+------+------+-------------------+-------+-------+
| a_id | name     | a_id | p_id | title             | state | price |
+------+----------+------+------+-------------------+-------+-------+
|    3 | Van Gogh |    3 |    3 | Starry Night      | KY    |    48 |
|    3 | Van Gogh |    3 |    4 | The Potato Eaters | KY    |    67 |
+------+----------+------+------+-------------------+-------+-------+
```

前面的查询使用 `SELECT *` 来显示所有列。为了更具选择性，只命名您感兴趣的那些列：

```
mysql> `SELECT artist.name, painting.title, painting.state, painting.price`
    -> `FROM artist INNER JOIN painting`
    -> `ON artist.a_id = painting.a_id`
    -> `WHERE painting.state = 'KY';`
+----------+-------------------+-------+-------+
| name     | title             | state | price |
+----------+-------------------+-------+-------+
| Van Gogh | Starry Night      | KY    |    48 |
| Van Gogh | The Potato Eaters | KY    |    67 |
+----------+-------------------+-------+-------+
```

Joins 可以使用超过两个表。假设您希望在前面的查询结果中看到完整的州名而不是缩写。在早期章节中使用的 `states` 表将州的缩写映射到名称；将其添加到前面的查询中，以显示名称而不是缩写：

```
mysql> `SELECT artist.name, painting.title, states.name, painting.price`
    -> `FROM artist INNER JOIN painting INNER JOIN states`
    -> `ON artist.a_id = painting.a_id AND painting.state = states.abbrev`
    -> `WHERE painting.state = 'KY';`
+----------+-------------------+----------+-------+
| name     | title             | name     | price |
+----------+-------------------+----------+-------+
| Van Gogh | Starry Night      | Kentucky |    48 |
| Van Gogh | The Potato Eaters | Kentucky |    67 |
+----------+-------------------+----------+-------+
```

另一种常见的三表连接用途是枚举多对多关系（见 Recipe 16.6）。

通过在连接中包含适当的条件，您可以回答非常具体的问题：

+   凡·高画了哪些画？使用 `a_id` 值查找匹配行，添加 `WHERE` 子句以限制输出到包含艺术家名称的行，并从这些行中选择标题：

    ```
    mysql> `SELECT painting.title`
        -> `FROM artist INNER JOIN painting ON artist.a_id = painting.a_id`
        -> `WHERE artist.name = 'Van Gogh';`
    +-------------------+
    | title             |
    +-------------------+
    | Starry Night      |
    | The Potato Eaters |
    +-------------------+
    ```

+   谁画了 *蒙娜丽莎*？再次使用 `a_id` 列连接行，但这次使用 `WHERE` 子句将输出限制为包含标题的行，并从这些行中选择艺术家名称：

    ```
    mysql> `SELECT artist.name`
        -> `FROM artist INNER JOIN painting ON artist.a_id = painting.a_id`
        -> `WHERE painting.title = 'Mona Lisa';`
    +----------+
    | name     |
    +----------+
    | Da Vinci |
    +----------+
    ```

+   您在肯塔基州或印第安纳州购买了哪些艺术家的画作？这与前一语句类似，但在 `painting` 表中测试不同的列（`state`），以将输出限制为 `KY` 或 `IN` 的行：

    ```
    mysql> `SELECT DISTINCT artist.name`
        -> `FROM artist INNER JOIN painting ON artist.a_id = painting.a_id`
        -> `WHERE painting.state IN ('KY','IN');`
    +----------+
    | name     |
    +----------+
    | Da Vinci |
    | Van Gogh |
    +----------+
    ```

    该语句还使用 `DISTINCT` 显示每位艺术家的名称仅一次。尝试去掉 `DISTINCT`；Van Gogh 会出现两次，因为您在肯塔基州获得了两幅 Van Gogh 的作品。

+   与聚合函数一起使用的连接生成摘要。此语句显示每位艺术家有多少幅绘画：

    ```
    mysql> `SELECT artist.name, COUNT(*) AS 'number of paintings'`
        -> `FROM artist INNER JOIN painting ON artist.a_id = painting.a_id`
        -> `GROUP BY artist.name;`
    +----------+---------------------+
    | name     | number of paintings |
    +----------+---------------------+
    | Da Vinci |                   2 |
    | Renoir   |                   1 |
    | Van Gogh |                   2 |
    +----------+---------------------+
    ```

    更为复杂的语句使用聚合，还显示了您为每位艺术家的绘画支付了多少，总计和平均数：

    ```
    mysql> `SELECT artist.name,`
        -> `COUNT(*) AS 'number of paintings',`
        -> `SUM(painting.price) AS 'total price',`
        -> `AVG(painting.price) AS 'average price'`
        -> `FROM artist INNER JOIN painting ON artist.a_id = painting.a_id`
        -> `GROUP BY artist.name;`
    +----------+---------------------+-------------+---------------+
    | name     | number of paintings | total price | average price |
    +----------+---------------------+-------------+---------------+
    | Da Vinci |                   2 |         121 |       60.5000 |
    | Renoir   |                   1 |          64 |       64.0000 |
    | Van Gogh |                   2 |         115 |       57.5000 |
    +----------+---------------------+-------------+---------------+
    ```

前述摘要语句仅为艺术家表中确实已获取其绘画的艺术家生成输出。（例如，Monet 在艺术家表中列出，但在摘要中未列出，因为您尚未拥有他的任何绘画。）要总结所有艺术家，包括那些您尚未获得其绘画的艺术家，您必须使用不同类型的连接，具体来说是外连接：

+   使用 `INNER JOIN` 编写的连接是内连接。它们仅为一个表中与另一个表中的值匹配的值生成结果。

+   外连接也可以生成这些匹配项，但还可以显示一个表中缺少另一个表中的哪些值。[Recipe 16.2](https://example.org/recipe_16.2) 介绍了外连接。

在连接中始终允许以表名限定列名的 *`tbl_name.col_name`* 符号化表示，但如果该名称仅出现在连接的一个表中，则可以缩短为 *`col_name`*。在这种情况下，MySQL 可以明确确定列来自哪个表，无需表名限定符。我们无法在以下连接中这样做。两个表都有一个 `a_id` 列，因此 `ON` 子句列引用存在歧义：

```
mysql> `SELECT * FROM artist INNER JOIN painting ON a_id = a_id;`
ERROR 1052 (23000): Column 'a_id' in on clause is ambiguous
```

相比之下，以下查询是明确的。每个 `a_id` 实例都带有适当的表名限定，只有 `artist` 有一个 `name` 列，而只有 `painting` 有 `title` 和 `state` 列：

```
mysql> `SELECT name, title, state FROM artist INNER JOIN painting`
    -> `ON artist.a_id = painting.a_id`
    -> `ORDER BY name;`
+----------+-------------------+-------+
| name     | title             | state |
+----------+-------------------+-------+
| Da Vinci | The Last Supper   | IN    |
| Da Vinci | Mona Lisa         | MI    |
| Renoir   | Les Deux Soeurs   | NE    |
| Van Gogh | Starry Night      | KY    |
| Van Gogh | The Potato Eaters | KY    |
+----------+-------------------+-------+
```

为了让语句的含义对人类读者更加清晰，即使对于 MySQL 而言可能不是严格必需的，经常使用限定列名也是有用的。因此，我们倾向于在连接示例中使用限定名称。

为了在限定列引用时避免编写完整的表名，给每个表分配一个简短的别名，并使用该别名引用其列。以下两个语句是等效的：

```
SELECT artist.name, painting.title, states.name, painting.price
FROM artist INNER JOIN painting INNER JOIN states
ON artist.a_id = painting.a_id AND painting.state = states.abbrev;

SELECT a.name, p.title, s.name, p.price
FROM artist AS a INNER JOIN painting AS p INNER JOIN states AS s
ON a.a_id = p.a_id AND p.state = s.abbrev;
```

在 `AS` *`alias_name`* 子句中，`AS` 是可选的。

对于选择许多列的复杂语句，使用别名可以节省大量输入。此外，对于某些类型的语句，别名不仅方便而且必要，特别是在涉及自连接的情况下（参见 [Recipe 16.4](https://example.org/recipe_16.4)）。

# 16.2 寻找表之间的不匹配

## 问题

您想要查找一个表中没有与另一个表匹配的行。或者，您想要基于表之间的连接生成一个列表，并且希望该列表包括第一个表中的每一行条目，包括第二个表中没有匹配的行。

## 解决方案

使用外连接（`LEFT JOIN`或`RIGHT JOIN`）或`NOT IN`子查询。

## 讨论

Recipe 16.1 专注于内连接，它查找两个表之间的匹配项。然而，对于某些问题的答案需要确定哪些行*没有*匹配（或者换句话说，另一个表中缺少值的行）。例如，你可能想知道`artist`表中哪些艺术家还没有作品。在其他情况下也会出现类似的问题：

+   你有一个潜在客户列表，还有一个已下订单的人员列表。为了将销售努力集中在尚未成为实际客户的人员上，生成第一个列表中存在但第二个列表中不存在的人员集合。

+   你有一个棒球运动员列表，还有一个击出全垒打的运动员列表。为了确定第一个列表中哪些运动员*没有*击出全垒打，生成第一个列表中存在但第二个列表中不存在的运动员集合。

这些类型的问题需要使用外连接。像内连接一样，外连接查找表之间的匹配项。但与内连接不同，外连接还确定一个表中哪些行在另一个表中没有匹配。外连接有两种类型，分别是`LEFT JOIN`和`RIGHT JOIN`。

要了解外连接的实用性，考虑一下以下问题：确定`artist`表中哪些艺术家在`painting`表中不存在。目前，这些表很小，所以可以通过目测轻松查看，发现你没有莫奈的绘画作品（`painting`表中没有`a_id`值为 2 的行）：

```
mysql> `SELECT * FROM artist ORDER BY a_id;`
+------+----------+
| a_id | name     |
+------+----------+
|    1 | Da Vinci |
|    2 | Monet    |
|    3 | Van Gogh |
|    4 | Renoir   |
+------+----------+
mysql> `SELECT * FROM painting ORDER BY a_id, p_id;`
+------+------+-------------------+-------+-------+
| a_id | p_id | title             | state | price |
+------+------+-------------------+-------+-------+
|    1 |    1 | The Last Supper   | IN    |    34 |
|    1 |    2 | Mona Lisa         | MI    |    87 |
|    3 |    3 | Starry Night      | KY    |    48 |
|    3 |    4 | The Potato Eaters | KY    |    67 |
|    4 |    5 | Les Deux Soeurs   | NE    |    64 |
+------+------+-------------------+-------+-------+
```

随着你获得更多绘画作品和表变得更大，通过目测来回答问题将不再那么容易。你能用 SQL 来回答吗？当然可以，尽管首次尝试的解决方案通常看起来像以下语句，它使用不等条件来查找两个表之间的不匹配：

```
mysql> `SELECT * FROM artist INNER JOIN painting`
    -> `ON artist.a_id <> painting.a_id`
    -> `ORDER BY artist.a_id;`
+------+----------+------+------+-------------------+-------+-------+
| a_id | name     | a_id | p_id | title             | state | price |
+------+----------+------+------+-------------------+-------+-------+
|    1 | Da Vinci |    4 |    5 | Les Deux Soeurs   | NE    |    64 |
|    1 | Da Vinci |    3 |    4 | The Potato Eaters | KY    |    67 |
|    1 | Da Vinci |    3 |    3 | Starry Night      | KY    |    48 |
|    2 | Monet    |    1 |    1 | The Last Supper   | IN    |    34 |
|    2 | Monet    |    4 |    5 | Les Deux Soeurs   | NE    |    64 |
|    2 | Monet    |    3 |    4 | The Potato Eaters | KY    |    67 |
|    2 | Monet    |    3 |    3 | Starry Night      | KY    |    48 |
|    2 | Monet    |    1 |    2 | Mona Lisa         | MI    |    87 |
|    3 | Van Gogh |    1 |    2 | Mona Lisa         | MI    |    87 |
|    3 | Van Gogh |    1 |    1 | The Last Supper   | IN    |    34 |
|    3 | Van Gogh |    4 |    5 | Les Deux Soeurs   | NE    |    64 |
|    4 | Renoir   |    3 |    3 | Starry Night      | KY    |    48 |
|    4 | Renoir   |    1 |    2 | Mona Lisa         | MI    |    87 |
|    4 | Renoir   |    1 |    1 | The Last Supper   | IN    |    34 |
|    4 | Renoir   |    3 |    4 | The Potato Eaters | KY    |    67 |
+------+----------+------+------+-------------------+-------+-------+
```

查询看起来可能是合理的，但其结果显然是错误的。例如，它错误地表明每幅画都是由几位不同的艺术家绘制的。问题在于该语句列出了两个表中艺术家 ID 值不相同的所有值组合。实际上，你需要的是`artist`中在`painting`中根本不存在的值列表，但内连接只能基于两个表中都存在的值生成结果。它无法告诉你有关其中一个表中缺少的值的任何信息。

当面对需要在一个表中找到在另一个表中没有匹配或缺失的值时，你应该养成思维的习惯，<q>啊哈，这是一个 `LEFT` `JOIN` 的问题。</q> `LEFT` `JOIN` 是一种外连接的类型：它类似于内连接，它将第一个（左）表中的行与第二个（右）表中的行进行匹配。此外，如果左表的行在右表中没有匹配，`LEFT` `JOIN` 仍然会生成一行——其中来自右表的所有列都设置为 `NULL`。这意味着你可以通过查找 `NULL` 来找到在右表中缺失的值。通过逐步进行，你更容易理解这是如何发生的。从显示匹配行的内连接开始：

```
mysql> `SELECT * FROM artist INNER JOIN painting`
    -> `ON artist.a_id = painting.a_id`
    -> `ORDER BY artist.a_id;`
+------+----------+------+------+-------------------+-------+-------+
| a_id | name     | a_id | p_id | title             | state | price |
+------+----------+------+------+-------------------+-------+-------+
|    1 | Da Vinci |    1 |    1 | The Last Supper   | IN    |    34 |
|    1 | Da Vinci |    1 |    2 | Mona Lisa         | MI    |    87 |
|    3 | Van Gogh |    3 |    3 | Starry Night      | KY    |    48 |
|    3 | Van Gogh |    3 |    4 | The Potato Eaters | KY    |    67 |
|    4 | Renoir   |    4 |    5 | Les Deux Soeurs   | NE    |    64 |
+------+----------+------+------+-------------------+-------+-------+
```

在这个输出中，第一个 `a_id` 列来自 `artist` 表，第二个来自 `painting` 表。

现在将 `INNER` 替换为 `LEFT`，看看外连接的结果：

```
mysql> `SELECT * FROM artist LEFT JOIN painting`
    -> `ON artist.a_id = painting.a_id`
    -> `ORDER BY artist.a_id;`
+------+----------+------+------+-------------------+-------+-------+
| a_id | name     | a_id | p_id | title             | state | price |
+------+----------+------+------+-------------------+-------+-------+
|    1 | Da Vinci |    1 |    1 | The Last Supper   | IN    |    34 |
|    1 | Da Vinci |    1 |    2 | Mona Lisa         | MI    |    87 |
|    2 | Monet    | NULL | NULL | NULL              | NULL  |  NULL |
|    3 | Van Gogh |    3 |    3 | Starry Night      | KY    |    48 |
|    3 | Van Gogh |    3 |    4 | The Potato Eaters | KY    |    67 |
|    4 | Renoir   |    4 |    5 | Les Deux Soeurs   | NE    |    64 |
+------+----------+------+------+-------------------+-------+-------+
```

与内连接相比，外连接为每个没有 `painting` 表匹配的 `artist` 行生成了额外的行，其中所有 `painting` 列都设置为 `NULL`。

接下来，为了仅限制输出到不匹配的 `artist` 行，添加一个 `WHERE` 子句，在其中查找任何 `painting` 列中的 `NULL` 值，这些列本来不可能包含 `NULL`。这样就会过滤掉内连接生成的行，只留下外连接生成的行：

```
mysql> `SELECT * FROM artist LEFT JOIN painting`
    -> `ON artist.a_id = painting.a_id`
    -> `WHERE painting.a_id IS NULL;`
+------+-------+------+------+-------+-------+-------+
| a_id | name  | a_id | p_id | title | state | price |
+------+-------+------+------+-------+-------+-------+
|    2 | Monet | NULL | NULL | NULL  | NULL  |  NULL |
+------+-------+------+------+-------+-------+-------+
```

最后，为了仅显示在 `painting` 表中缺失的 `artist` 表值，编写输出列列表以仅命名来自 `artist` 表的列。结果是 `LEFT` `JOIN` 列出了那些包含右表中不存在的 `a_id` 值的左表行：

```
mysql> `SELECT artist.* FROM artist LEFT JOIN painting`
    -> `ON artist.a_id = painting.a_id`
    -> `WHERE painting.a_id IS NULL;`
+------+-------+
| a_id | name  |
+------+-------+
|    2 | Monet |
+------+-------+
```

类似的操作报告每个左表值以及指示它是否在右表中存在的指标。为此，执行一个计算每个左表值在右表中出现次数的 `LEFT` `JOIN`。计数为零表示该值不存在。下面的语句列出了 `artist` 表中的每位艺术家，并显示该艺术家是否有任何绘画作品：

```
mysql> `SELECT artist.name,`
    -> `IF(COUNT(painting.a_id)>0,'yes','no') AS 'in collection?'`
    -> `FROM artist LEFT JOIN painting ON artist.a_id = painting.a_id`
    -> `GROUP BY artist.name;`
+----------+----------------+
| name     | in collection? |
+----------+----------------+
| Da Vinci | yes            |
| Monet    | no             |
| Renoir   | yes            |
| Van Gogh | yes            |
+----------+----------------+
```

`RIGHT` `JOIN` 是一种外连接，类似于 `LEFT` `JOIN`，但是反转了左右表的角色。语义上，`RIGHT` `JOIN` 强制匹配过程生成右表中每个表的行，即使在左表中没有相应的行也是如此。从语法上讲，*`tbl1`* `LEFT` `JOIN` *`tbl2`* 等效于 *`tbl2`* `RIGHT` `JOIN` *`tbl1`*。因此，本书中对 `LEFT` `JOIN` 的引用如果反转表的角色也适用于 `RIGHT` `JOIN`。

另一种识别一个表中存在但另一个表中缺失值的方法是使用 `NOT` `IN` 子查询。下面的示例查找在 `painting` 表中没有代表的艺术家；与之前回答同样问题的 `LEFT` `JOIN` 进行对比：

```
mysql> `SELECT * FROM artist`
    -> `WHERE a_id NOT IN (SELECT a_id FROM painting);`
+------+-------+
| a_id | name  |
+------+-------+
|    2 | Monet |
+------+-------+
```

## 另见

正如本节所示，`LEFT` `JOIN`对于查找另一张表中没有匹配的值或显示每个值是否匹配非常有用。`LEFT` `JOIN`还可用于生成包含列表中所有项目的摘要，即使在汇总期间没有要汇总的内容也是如此。这在候选表与详细表之间的关系中非常常见。例如，`LEFT` `JOIN`可以生成<q>每位客户的总销售额</q>报告，列出所有客户，即使在汇总期间没有购买任何商品的客户也包括在内。（有关候选-详细列表的信息，请参见 Recipe 16.5。）

当您收到两个应该相关的数据文件并且想要确定它们是否真的相关时，`LEFT` `JOIN`也非常有用于一致性检查。也就是说，您希望检查它们的关系是否完整。将每个文件导入到 MySQL 表中，然后运行一对`LEFT` `JOIN`语句以确定一个表中是否有未连接的行，即另一个表中没有匹配的行。Recipe 16.3 讨论了如何识别（并可选地删除）这些未连接的行。

# 16.3 标识和删除不匹配或未连接的行

## 问题

您有两个相关的数据集，但可能不完全相关。您希望确定任一数据集中是否存在记录是<q>未连接</q>的（没有任何另一个数据集中的记录匹配），并且如果有，则可能删除这些记录。

## 解决方案

要在每个表中标识不匹配的值，使用`LEFT` `JOIN`或`NOT` `IN`子查询。要删除它们，请使用带有`NOT` `IN`子查询的`DELETE`语句。

## 讨论

内连接对于识别匹配项很有用，外连接对于识别不匹配项很有用。当您有相关数据集，并且这些数据集之间的关系可能不完美时，外连接的这个属性非常有价值。例如，在必须验证来自外部来源的两个数据文件的完整性时，可能会发现不匹配。

当您有相关的未匹配行的表时，可以使用 SQL 语句来分析和修改它们。具体来说，恢复它们的关系是识别未连接的行，然后删除它们：

+   要识别未连接的行，请使用`LEFT` `JOIN`，因为这是一个<q>查找未匹配行</q>的问题；或者，可以使用`NOT` `IN`子查询（参见 Recipe 16.2）。

+   要删除不匹配的行，使用带有`NOT` `IN`子查询的`DELETE`语句。

了解未匹配的数据很有用，因为您可以提醒提供数据的人。数据收集方法可能存在需要纠正的缺陷。例如，对于销售数据，缺少的地区可能意味着某些地区经理没有报告，而这个遗漏被忽视了。

以下示例展示了如何使用描述销售区域和每个区域销售量的两个数据集来识别和删除不匹配的行。一个数据集包含每个区域的 ID 和位置：

```
mysql> `SELECT * FROM sales_region ORDER BY region_id;`
+-----------+------------------------+
| region_id | name                   |
+-----------+------------------------+
|         1 | London, United Kingdom |
|         2 | Madrid, Spain          |
|         3 | Berlin, Germany        |
|         4 | Athens, Greece         |
+-----------+------------------------+
```

另一个数据集包含销售量数据。每行包含一年四分之一的销售额，并指示适用于该行的销售区域：

```
mysql> `SELECT region_id, year, quarter, volume`
    -> `FROM sales_volume ORDER BY region_id, year, quarter;`
+-----------+------+---------+--------+
| region_id | year | quarter | volume |
+-----------+------+---------+--------+
|         1 | 2014 |       1 | 100400 |
|         1 | 2014 |       2 | 120000 |
|         3 | 2014 |       1 | 280000 |
|         3 | 2014 |       2 | 250000 |
|         5 | 2014 |       1 |  18000 |
|         5 | 2014 |       2 |  32000 |
+-----------+------+---------+--------+
```

一点视觉检查显示，两个表都没有完全匹配对方。销售区域 2 和 4 在销售量表中没有表示，而销售量表包含销售区域表中不存在的区域 5 的行。但我们不想通过视觉检查来检查表格。我们希望通过使用执行工作的 SQL 语句来查找不匹配的行。

识别不匹配是使用外连接的问题。例如，要查找没有销售量行的销售区域，请使用以下`LEFT` `JOIN`：

```
mysql> `SELECT sales_region.region_id AS 'unmatched region row IDs'`
    -> `FROM sales_region LEFT JOIN sales_volume`
    ->   `ON sales_region.region_id = sales_volume.region_id`
    -> `WHERE sales_volume.region_id IS NULL;`
+--------------------------+
| unmatched region row IDs |
+--------------------------+
|                        2 |
|                        4 |
+--------------------------+
```

相反，要查找未与任何已知区域关联的销售量行，请反转两个表的角色：

```
mysql> `SELECT sales_volume.region_id AS 'unmatched volume row IDs'`
    -> `FROM sales_volume LEFT JOIN sales_region`
    ->   `ON sales_volume.region_id = sales_region.region_id`
    -> `WHERE sales_region.region_id IS NULL;`
+--------------------------+
| unmatched volume row IDs |
+--------------------------+
|                        5 |
|                        5 |
+--------------------------+
```

在这种情况下，如果缺少区域的多个体积行，则一个 ID 在列表中出现多次。要仅查看每个不匹配的 ID 一次，请使用`SELECT` `DISTINCT`：

```
mysql> `SELECT DISTINCT sales_volume.region_id AS 'unmatched volume row IDs'`
    -> `FROM sales_volume LEFT JOIN sales_region`
    ->   `ON sales_volume.region_id = sales_region.region_id`
    -> `WHERE sales_region.region_id IS NULL`
+--------------------------+
| unmatched volume row IDs |
+--------------------------+
|                        5 |
+--------------------------+
```

您还可以使用`NOT` `IN`子查询识别不匹配：

```
mysql> `SELECT region_id AS 'unmatched region row IDs'`
    -> `FROM sales_region`
    -> `WHERE region_id NOT IN (SELECT region_id FROM sales_volume);`
+--------------------------+
| unmatched region row IDs |
+--------------------------+
|                        2 |
|                        4 |
+--------------------------+
mysql> `SELECT region_id AS 'unmatched volume row IDs'`
    -> `FROM sales_volume`
    -> `WHERE region_id NOT IN (SELECT region_id FROM sales_region);`
+--------------------------+
| unmatched volume row IDs |
+--------------------------+
|                        5 |
|                        5 |
+--------------------------+
```

要摆脱不匹配的行，请在`DELETE`语句中使用`NOT` `IN`子查询。要删除与没有`sales_volume`行匹配的`sales_region`行，请执行以下操作：

```
DELETE FROM sales_region
WHERE region_id NOT IN (SELECT region_id FROM sales_volume);
```

要删除不匹配的`sales_volume`行，这些行不与任何`sales_region`行匹配，可以使用类似的语句，只是表的角色相反：

```
DELETE FROM sales_volume
WHERE region_id NOT IN (SELECT region_id FROM sales_region);
```

# 16.4 比较表与自身

## 问题

您想要比较表中的行与同一表中的其他行。例如，您想找出您收藏的所有由绘制了*《土豆食者》*的艺术家的绘画。或者您想知道列在`states`表中的哪些州与纽约在同一年加入了联盟。或者您想知道哪些州没有与其他任何州在同一年加入联盟。

## 解决方案

需要比较表与自身的问题涉及一种称为自连接的操作。它的执行方式与其他连接类似，但您必须使用表别名，以便在语句中可以以不同方式引用同一表。

## 讨论

当一个表连接到另一个表时发生的一种特殊情况是，当两个表相同时。这被称为自连接。起初可能会感到困惑或奇怪，但这是完全合法的。您可能会经常使用自连接，因为它们非常重要。

需要自连接的一个提示是，您想知道表中哪些行对某些条件满足。假设您最喜欢的绘画是*《土豆食者》*，您想识别出您收藏的所有由同一位艺术家绘制的物品。我们从艺术家 ID 和绘画标题开始看起来像这样：

```
mysql> `SELECT a_id, title FROM painting ORDER BY a_id;`
+------+-------------------+
| a_id | title             |
+------+-------------------+
|    1 | The Last Supper   |
|    1 | Mona Lisa         |
|    3 | Starry Night      |
|    3 | The Potato Eaters |
|    4 | Les Deux Soeurs   |
+------+-------------------+
```

解决问题的方法如下：

1.  确定包含标题*The Potato Eaters*的`painting`表行，以便引用其`a_id`值。

1.  匹配表中具有相同`a_id`值的其他行。

1.  显示那些匹配行的标题。

关键在于使用正确的表示法。首次尝试将表与自身连接通常看起来像这样：

```
mysql> `SELECT title`
    -> `FROM painting INNER JOIN painting`
    -> `ON a_id = a_id`
    -> `WHERE title = 'The Potato Eaters';`
ERROR 1066 (42000): Not unique table/alias: 'painting'
```

在该语句中，列引用是模糊的，因为 MySQL 无法确定给定列名指向哪个`painting`表的实例。解决方案是至少给表的一个实例设置别名，这样你就可以通过使用不同的表限定符来区分列引用。以下语句显示了如何做到这一点，使用别名`p1`和`p2`来区分`painting`表的不同实例：

```
mysql> `SELECT p2.title`
    -> `FROM painting AS p1 INNER JOIN painting AS p2`
    -> `ON p1.a_id = p2.a_id`
    -> `WHERE p1.title = 'The Potato Eaters';`
+-------------------+
| title             |
+-------------------+
| Starry Night      |
| The Potato Eaters |
+-------------------+
```

该语句输出显示了自连接的典型特征：当你从一个表实例（*The Potato Eaters*）中的一个参考值开始，去查找第二个表实例（同一艺术家的绘画作品）中的匹配行时，输出包括参考值本身。这是有道理的：毕竟，该参考值与自身匹配。为了仅找到同一艺术家的*其他*绘画作品，请明确从输出中排除参考值：

```
mysql> `SELECT p2.title`
    -> `FROM painting AS p1 INNER JOIN painting AS p2`
    -> `ON p1.a_id = p2.a_id`
    -> `WHERE p1.title = 'The Potato Eaters' AND p2.title <> p1.title`
+--------------+
| title        |
+--------------+
| Starry Night |
+--------------+
```

前面的语句使用 ID 值比较来匹配两个表实例中的行，但可以使用任何类型的值。例如，要使用`states`表回答问题<q>哪些州与纽约同年加入联邦？</q>，请基于表中`statehood`列中日期的年份部分执行时间上的成对比较：

```
mysql> `SELECT s2.name, s2.statehood`
    -> `FROM states AS s1 INNER JOIN states AS s2`
    -> `ON YEAR(s1.statehood) = YEAR(s2.statehood) AND s1.name <> s2.name`
    -> `WHERE s1.name = 'New York'`
    -> `ORDER BY s2.name;`
+----------------+------------+
| name           | statehood  |
+----------------+------------+
| Connecticut    | 1788-01-09 |
| Georgia        | 1788-01-02 |
| Maryland       | 1788-04-28 |
| Massachusetts  | 1788-02-06 |
| New Hampshire  | 1788-06-21 |
| South Carolina | 1788-05-23 |
| Virginia       | 1788-06-25 |
+----------------+------------+
```

###### 注意

在上面的示例中，我们没有指定纽约加入联邦的年份。相反，我们比较了“New York”州名称行的`statehood`列的值和其他州相同的`statehood`列的值。

现在假设你想找到*每对*在同一年加入联邦的州。在这种情况下，输出可能包括`states`表中任意一对行。自连接非常适合解决这个问题：

```
mysql> `SELECT YEAR(s1.statehood) AS year,`
    -> `s1.name AS name1, s1.statehood AS statehood1,`
    -> `s2.name AS name2, s2.statehood AS statehood2`
    -> `FROM states AS s1 INNER JOIN states AS s2`
    -> `ON YEAR(s1.statehood) = YEAR(s2.statehood) AND s1.name <> s2.name`
    -> `ORDER BY year, name1, name2;`
+------+----------------+------------+----------------+------------+
| year | name1          | statehood1 | name2          | statehood2 |
+------+----------------+------------+----------------+------------+
| 1787 | Delaware       | 1787-12-07 | New Jersey     | 1787-12-18 |
| 1787 | Delaware       | 1787-12-07 | Pennsylvania   | 1787-12-12 |
| 1787 | New Jersey     | 1787-12-18 | Delaware       | 1787-12-07 |
| 1787 | New Jersey     | 1787-12-18 | Pennsylvania   | 1787-12-12 |
| 1787 | Pennsylvania   | 1787-12-12 | Delaware       | 1787-12-07 |
| 1787 | Pennsylvania   | 1787-12-12 | New Jersey     | 1787-12-18 |
…
| 1912 | Arizona        | 1912-02-14 | New Mexico     | 1912-01-06 |
| 1912 | New Mexico     | 1912-01-06 | Arizona        | 1912-02-14 |
| 1959 | Alaska         | 1959-01-03 | Hawaii         | 1959-08-21 |
| 1959 | Hawaii         | 1959-08-21 | Alaska         | 1959-01-03 |
+------+----------------+------------+----------------+------------+
```

在`ON`子句中的条件要求州名对不相同，从而消除了显示每个州与自己同年加入联邦的重复行。但你会注意到每对剩余的州仍然出现两次。例如，有一行列出了特拉华州和新泽西州，另一行列出了新泽西州和特拉华州。自连接经常会产生这种情况：它们生成包含相同值但值的顺序不同的行对。

因为值在行内未按相同顺序列出，它们并不相同，你无法通过在语句中添加`DISTINCT`来摆脱这些<q>近似重复项</q>。为了解决这个问题，请以一种只有一行从每对中出现在查询结果中的方式选择行。稍微修改`ON`子句，从：

```
ON YEAR(s1.statehood) = YEAR(s2.statehood) AND s1.name <> s2.name
```

to:

```
ON YEAR(s1.statehood) = YEAR(s2.statehood) AND s1.name < s2.name
```

使用`<`而不是`<>`只选择第一个州名称字母顺序小于第二个州名称的行，并消除名称按相反顺序出现的行（以及州名称相同的行）。由此产生的查询输出所需的输出而无重复项：

```
mysql> `SELECT YEAR(s1.statehood) AS year,`
    -> `s1.name AS name1, s1.statehood AS statehood1,`
    -> `s2.name AS name2, s2.statehood AS statehood2`
    -> `FROM states AS s1 INNER JOIN states AS s2`
    -> `ON YEAR(s1.statehood) = YEAR(s2.statehood) AND s1.name < s2.name`
    -> `ORDER BY year, name1, name2;`
+------+----------------+------------+----------------+------------+
| year | name1          | statehood1 | name2          | statehood2 |
+------+----------------+------------+----------------+------------+
| 1787 | Delaware       | 1787-12-07 | New Jersey     | 1787-12-18 |
| 1787 | Delaware       | 1787-12-07 | Pennsylvania   | 1787-12-12 |
| 1787 | New Jersey     | 1787-12-18 | Pennsylvania   | 1787-12-12 |
…
| 1912 | Arizona        | 1912-02-14 | New Mexico     | 1912-01-06 |
| 1959 | Alaska         | 1959-01-03 | Hawaii         | 1959-08-21 |
+------+----------------+------------+----------------+------------+
```

对于<q>表中哪些值*不*与其他行匹配？</q>类型的自连接问题，使用`LEFT` `JOIN`而不是`INNER` `JOIN`。这种情况的一个实例是问题<q>哪些州没有在同一年加入联盟？</q>在这种情况下，解决方案使用`states`表与其自身的`LEFT` `JOIN`：

```
mysql> `SELECT s1.name, s1.statehood`
    -> `FROM states AS s1 LEFT JOIN states AS s2`
    -> `ON YEAR(s1.statehood) = YEAR(s2.statehood) AND s1.name <> s2.name`
    -> `WHERE s2.name IS NULL`
    -> `ORDER BY s1.name;`
+----------------+------------+
| name           | statehood  |
+----------------+------------+
| Alabama        | 1819-12-14 |
| Arkansas       | 1836-06-15 |
| California     | 1850-09-09 |
| Colorado       | 1876-08-01 |
| Illinois       | 1818-12-03 |
| Indiana        | 1816-12-11 |
| Iowa           | 1846-12-28 |
| Kansas         | 1861-01-29 |
| Kentucky       | 1792-06-01 |
…
| Tennessee      | 1796-06-01 |
| Utah           | 1896-01-04 |
| Vermont        | 1791-03-04 |
| West Virginia  | 1863-06-20 |
| Wisconsin      | 1848-05-29 |
+----------------+------------+
```

对于`states`表中的每一行，该语句选择具有与该州在同一年具有`statehood`值的行，但不包括该州本身。对于没有这种匹配的行，`LEFT` `JOIN`强制输出仍然包含一行，其中所有`s2`列设置为`NULL`。这些行标识出在同一年没有其他州加入联盟的州。

# 16.5 生成候选-详细列表和摘要

## 问题

两个表之间有一种关系，即一个表中的行，通常称为具有候选键的父表，被另一个表中的一个或多个行引用，通常称为具有详细行的子表。在这种情况下，您希望生成一个显示每个父行及其详细行的列表，或者生成一个显示每个父行的详细行摘要的列表。

## 解决方案

这是一种一对多的关系。解决此问题的方法涉及连接，但连接的类型取决于您想要回答的问题。要生成仅包含存在某些详细行的父行的列表，请使用基于父表中的主键的内连接。要生成包含所有父行的列表，甚至包括没有详细行的父行，请使用外连接。

## 讨论

要从具有候选-详细或父-子关系的两个表中生成列表，一个表中的给定行可能与另一个表中的多个行匹配。这些关系经常发生。例如，在业务背景下，一对多的关系涉及每个客户的发票或每张发票的项目。

此处的示例提供了一些候选-详细问题，您可以使用本章前面的`artist`和`painting`表来询问（并回答）。

这是这些表的一种候选-详细问题，<q>每位艺术家绘制了哪些绘画？</q>这是一个简单的内连接（见第 16.1 节）。根据艺术家 ID 值，将每个`artist`行与其相应的`painting`行进行匹配：

```
mysql> `SELECT artist.name, painting.title`
    -> `FROM artist INNER JOIN painting ON artist.a_id = painting.a_id`
    -> `ORDER BY name, title;`
+----------+-------------------+
| name     | title             |
+----------+-------------------+
| Da Vinci | Mona Lisa         |
| Da Vinci | The Last Supper   |
| Renoir   | Les Deux Soeurs   |
| Van Gogh | Starry Night      |
| Van Gogh | The Potato Eaters |
+----------+-------------------+
```

要列出您没有绘画的艺术家，连接输出应包括一个表中没有与另一个表匹配的行。这是一种需要外连接的<q>查找非匹配行</q>问题（见第 16.2 节）。因此，要列出每个`artist`行，无论是否匹配任何`painting`行，请使用`LEFT` `JOIN`：

```
mysql> `SELECT artist.name, painting.title`
    -> `FROM artist LEFT JOIN painting ON artist.a_id = painting.a_id`
    -> `ORDER BY name, title;`
+----------+-------------------+
| name     | title             |
+----------+-------------------+
| Da Vinci | Mona Lisa         |
| Da Vinci | The Last Supper   |
| Monet    | NULL              |
| Renoir   | Les Deux Soeurs   |
| Van Gogh | Starry Night      |
| Van Gogh | The Potato Eaters |
+----------+-------------------+
```

结果中在 `title` 列中有 `NULL` 的行对应于 `artist` 表中列出的艺术家，对于这些艺术家，您没有绘画。

生成使用候选表和详细表的汇总时，同样的原则适用。 例如，要按艺术家的绘画数量汇总您的艺术收藏，您可以问，“在 `painting` 表中每位艺术家有多少幅画？” 要根据艺术家 ID 找到答案但显示艺术家的姓名（来自 `artist` 表），请使用此语句计算绘画数量：

```
mysql> `SELECT artist.name, COUNT(painting.a_id) AS paintings`
    -> `FROM artist INNER JOIN painting ON artist.a_id = painting.a_id`
    -> `GROUP BY artist.name;`
+----------+-----------+
| name     | paintings |
+----------+-----------+
| Da Vinci |         2 |
| Renoir   |         1 |
| Van Gogh |         2 |
+----------+-----------+
```

另一方面，您可能会问，“每位艺术家画了多少幅画？” 这与前一个问题相同（同一个语句回答），只要 `artist` 表中的每位艺术家至少有一行对应的 `painting` 表。 但是，如果 `artist` 表中有尚未由您收藏中的任何绘画代表的艺术家，则它们不会出现在语句输出中。 要生成包括在 `painting` 表中没有绘画的艺术家的汇总，请使用 `LEFT` `JOIN`：

```
mysql> `SELECT artist.name, COUNT(painting.a_id) AS paintings`
    -> `FROM artist LEFT JOIN painting ON artist.a_id = painting.a_id`
    -> `GROUP BY artist.name;`
+----------+-----------+
| name     | paintings |
+----------+-----------+
| Da Vinci |         2 |
| Monet    |         0 |
| Renoir   |         1 |
| Van Gogh |         2 |
+----------+-----------+
```

在撰写这种类型的语句时，要注意一个微妙的错误，这种错误很容易犯。 假设您稍微不同地编写了 `COUNT()` 函数，如下所示：

```
mysql> `SELECT artist.name, COUNT(*) AS paintings`
    -> `FROM artist LEFT JOIN painting ON artist.a_id = painting.a_id`
    -> `GROUP BY artist.name;`
```

```
+----------+-----------+
| name     | paintings |
+----------+-----------+
| Da Vinci |         2 |
| Monet    |         1 |
| Renoir   |         1 |
| Van Gogh |         2 |
+----------+-----------+
```

现在每位艺术家似乎至少有一幅画。 为什么会有这种差异？ 问题在于使用 `COUNT(*)` 而不是 `COUNT(painting.a_id)`。 `LEFT` `JOIN` 对于左表中未匹配的行的处理方式是生成一个行，其中右表的所有列均设置为 `NULL`。 在此示例中，右表是 `painting`。 使用 `COUNT(painting.a_id)` 的语句运行正确，因为 `COUNT(`*`expr`*`)` 仅计算非 `NULL` 值。 使用 `COUNT(*)` 的语句是错误的，因为它计算包括那些对应于缺失艺术家的 `NULL` 的行在内的*行*。

`LEFT` `JOIN` 也适用于其他类型的汇总。 要为 `artist` 表中每位艺术家的绘画总价和平均价添加额外列，请使用以下语句：

```
mysql> `SELECT artist.name,`
    -> `COUNT(painting.a_id) AS 'number of paintings',`
    -> `SUM(painting.price) AS 'total price',`
    -> `AVG(painting.price) AS 'average price'`
    -> `FROM artist LEFT JOIN painting ON artist.a_id = painting.a_id`
    -> `GROUP BY artist.name;`
+----------+---------------------+-------------+---------------+
| name     | number of paintings | total price | average price |
+----------+---------------------+-------------+---------------+
| Da Vinci |                   2 |         121 |       60.5000 |
| Monet    |                   0 |        NULL |          NULL |
| Renoir   |                   1 |          64 |       64.0000 |
| Van Gogh |                   2 |         115 |       57.5000 |
+----------+---------------------+-------------+---------------+
```

请注意，对于未被代表的艺术家，`COUNT()` 为零，但 `SUM()` 和 `AVG()` 为 `NULL`。 当应用于没有非 `NULL` 值的值集时，后两个函数返回 `NULL`。 在这种情况下显示零的和平均值，请用 `IFNULL(SUM(`*`expr`*`)，0)` 和 `IFNULL(AVG(`*`expr`*`)，0)` 替换 `SUM(`*`expr`*`)` 和 `AVG(`*`expr`*`)`。

# 16.6 枚举一对多关系

## 问题

当任一表中的任何行可能与另一表中的多行匹配时，您希望显示表之间的关系。

## 解决方案

这是一对多关系。 它需要一个第三个表来关联您的两个主表，并进行三向连接以生成它们之间的对应关系。

## 讨论

在之前的部分中使用的`artist`和`painting`表有一对多的关系：一个艺术家可能制作了许多绘画，但每幅绘画只由一个艺术家创作。一对多关系相对简单，可以使用两个相关表中共有的列进行连接。

表之间的多对多关系更复杂。当一个表中的一行可以在另一个表中有多个匹配项，反之亦然时，就会出现多对多关系。电影和演员之间的关系是一个例子：每部电影可能有多位演员，每位演员可能出演过多部电影。表示这种关系的一种方法是使用以下结构的表，每个电影-演员组合有一行：

```
mysql> `SELECT * FROM movies_actors ORDER BY year, movie, actor;`
+------+----------------------------+---------------+
| year | movie                      | actor         |
+------+----------------------------+---------------+
| 1997 | The Fifth Element          | Bruce Willis  |
| 1997 | The Fifth Element          | Gary Oldman   |
| 1997 | The Fifth Element          | Ian Holm      |
| 1999 | The Phantom Menace         | Ewan McGregor |
| 1999 | The Phantom Menace         | Liam Neeson   |
| 2001 | The Fellowship of the Ring | Elijah Wood   |
| 2001 | The Fellowship of the Ring | Ian Holm      |
| 2001 | The Fellowship of the Ring | Ian McKellen  |
| 2001 | The Fellowship of the Ring | Orlando Bloom |
| 2005 | Kingdom of Heaven          | Liam Neeson   |
| 2005 | Kingdom of Heaven          | Orlando Bloom |
| 2010 | Red                        | Bruce Willis  |
| 2010 | Red                        | Helen Mirren  |
| 2011 | Unknown                    | Diane Kruger  |
| 2011 | Unknown                    | Liam Neeson   |
+------+----------------------------+---------------+
```

此表捕捉了这种多对多关系的本质，但也是在非规范化形式，因为它不必要地存储了重复的信息。例如，每部电影的信息被记录了多次。为了更好地表示这种多对多关系，使用多个表：

+   在名为`movies`的表中仅存储每部电影的年份和名称一次。

+   在名为`actors`的表中仅存储每个演员的姓名一次。

+   创建第三个表 `movies_actors_link`，它存储电影-演员关联，并作为两个主要表之间的链接或桥梁。为了最小化存储在此表中的信息，分别为每部电影和每位演员分配唯一的 ID，并仅在`movies_actors_link`表中存储这些 ID。

结果的`movie`和`actor`表如下所示：

```
mysql> `SELECT * FROM movies ORDER BY id;`
+----+------+----------------------------+
| id | year | movie                      |
+----+------+----------------------------+
|  1 | 1997 | The Fifth Element          |
|  2 | 1999 | The Phantom Menace         |
|  3 | 2001 | The Fellowship of the Ring |
|  4 | 2005 | Kingdom of Heaven          |
|  5 | 2010 | Red                        |
|  6 | 2011 | Unknown                    |
+----+------+----------------------------+
mysql> `SELECT * FROM actors ORDER BY id;`
+----+---------------+
| id | actor         |
+----+---------------+
|  1 | Bruce Willis  |
|  2 | Diane Kruger  |
|  3 | Elijah Wood   |
|  4 | Ewan McGregor |
|  5 | Gary Oldman   |
|  6 | Helen Mirren  |
|  7 | Ian Holm      |
|  8 | Ian McKellen  |
|  9 | Liam Neeson   |
| 10 | Orlando Bloom |
+----+---------------+
```

`movies_actors_link` 表将电影和演员关联如下：

```
mysql> `SELECT * FROM movies_actors_link ORDER BY movie_id, actor_id;`
+----------+----------+
| movie_id | actor_id |
+----------+----------+
|        1 |        1 |
|        1 |        5 |
|        1 |        7 |
|        2 |        4 |
|        2 |        9 |
|        3 |        3 |
|        3 |        7 |
|        3 |        8 |
|        3 |       10 |
|        4 |        9 |
|        4 |       10 |
|        5 |        1 |
|        5 |        6 |
|        6 |        2 |
|        6 |        9 |
+----------+----------+
```

你一定会注意到，从人类的角度来看，`movies_actors_link` 表的内容完全毫无意义。这没关系：我们永远不需要明确显示它。它的实用性在于能够在查询中链接两个主要表，而不会出现在查询输出中。接下来的几个例子说明了这个原理。它们通过使用三表连接来回答关于电影或演员的问题，将两个主要表关联到链接表。

+   列出所有显示每部电影及其演员的配对。此语句枚举了`movie`和`actor`表之间的所有对应关系，并复制了最初位于非规范化`movies_actors`表中的信息：

    ```
    mysql> `SELECT m.year, m.movie, a.actor`
        -> `FROM movies AS m INNER JOIN movies_actors_link AS l`
        -> `INNER JOIN actors AS a`
        -> `ON m.id = l.movie_id AND a.id = l.actor_id`
        -> `ORDER BY m.year, m.movie, a.actor;`
    +------+----------------------------+---------------+
    | year | movie                      | actor         |
    +------+----------------------------+---------------+
    | 1997 | The Fifth Element          | Bruce Willis  |
    | 1997 | The Fifth Element          | Gary Oldman   |
    | 1997 | The Fifth Element          | Ian Holm      |
    | 1999 | The Phantom Menace         | Ewan McGregor |
    | 1999 | The Phantom Menace         | Liam Neeson   |
    | 2001 | The Fellowship of the Ring | Elijah Wood   |
    | 2001 | The Fellowship of the Ring | Ian Holm      |
    | 2001 | The Fellowship of the Ring | Ian McKellen  |
    | 2001 | The Fellowship of the Ring | Orlando Bloom |
    | 2005 | Kingdom of Heaven          | Liam Neeson   |
    | 2005 | Kingdom of Heaven          | Orlando Bloom |
    | 2010 | Red                        | Bruce Willis  |
    | 2010 | Red                        | Helen Mirren  |
    | 2011 | Unknown                    | Diane Kruger  |
    | 2011 | Unknown                    | Liam Neeson   |
    +------+----------------------------+---------------+
    ```

+   列出给定电影中的演员：

    ```
    mysql> `SELECT a.actor`
        -> `FROM movies AS m INNER JOIN movies_actors_link AS l`
        -> `INNER JOIN actors AS a`
        -> `ON m.id = l.movie_id AND a.id = l.actor_id`
        -> `WHERE m.movie = 'The Fellowship of the Ring'`
        -> `ORDER BY a.actor;`
    +---------------+
    | actor         |
    +---------------+
    | Elijah Wood   |
    | Ian Holm      |
    | Ian McKellen  |
    | Orlando Bloom |
    +---------------+
    ```

+   列出一个给定演员参演的所有电影：

    ```
    mysql> `SELECT m.year, m.movie`
        -> `FROM movies AS m INNER JOIN movies_actors_link AS l`
        -> `INNER JOIN actors AS a`
        -> `ON m.id = l.movie_id AND a.id = l.actor_id` 
        -> `WHERE a.actor = 'Liam Neeson'`
        -> `ORDER BY m.year, m.movie;`
    +------+--------------------+
    | year | movie              |
    +------+--------------------+
    | 1999 | The Phantom Menace |
    | 2005 | Kingdom of Heaven  |
    | 2011 | Unknown            |
    +------+--------------------+
    ```

# 16.7 查找每组的最小或最大值

## 问题

你希望找出表中每个组中包含给定列的最大或最小值的行。例如，你想要确定你收藏的每位艺术家的最贵的绘画。

## 解决方案

创建一个临时表来保存每组的最大或最小值，然后将临时表与原始表连接以提取每组的匹配行。如果你更喜欢单查询解决方案，则在`FROM`子句中使用子查询，而不是临时表。

## 讨论

许多问题涉及查找特定表列中的最大或最小值，但通常还想知道包含该值的行中的其他值。例如，使用来自第 10.6 节的`artist`和`painting`表的技术，可以回答诸如<q>收藏品中最昂贵的画作是什么，由谁绘制？</q>的问题。一种解决方案是将最高价格存储在用户定义变量中，然后使用该变量标识包含价格的行，以便可以从中检索其他列：

```
mysql> `SET @max_price = (SELECT MAX(price) FROM painting);`
mysql> `SELECT artist.name, painting.title, painting.price`
    -> `FROM artist INNER JOIN painting`
    -> `ON painting.a_id = artist.a_id`
    -> `WHERE painting.price = @max_price;`
+----------+-----------+-------+
| name     | title     | price |
+----------+-----------+-------+
| Da Vinci | Mona Lisa |    87 |
+----------+-----------+-------+
```

可以通过创建一个临时表来执行相同的操作，以存储最大价格并将其与其他表连接：

```
CREATE TABLE tmp SELECT MAX(price) AS max_price FROM painting;
SELECT artist.name, painting.title, painting.price
FROM artist INNER JOIN painting INNER JOIN tmp
ON painting.a_id = artist.a_id
AND painting.price = tmp.max_price;
```

表面上看，使用临时表和连接只是比使用用户定义变量回答问题更复杂的一种方式。这种技术是否有实际价值呢？有，因为它可以导致一种更一般的技术来回答更难的问题。前面的陈述仅显示整个`painting`表中最昂贵的单幅画信息。如果你的问题是：<q>每位艺术家最昂贵的画作是什么？</q>你不能使用用户定义变量来回答这个问题，因为答案需要找到每位艺术家的一个价格，并且变量只能保存单个值。但是使用临时表的技术效果很好，因为表可以保存多行，连接可以找到所有匹配项。

要回答问题，选择每个艺术家 ID 及其对应的最大画作价格放入临时表中。该表不仅包含最大画作价格，而且在每个组内都是最大的，其中<q>组</q>定义为<q>某位艺术家的画作。</q>然后使用临时表中存储的艺术家 ID 和价格与`painting`表中的行匹配，并将结果与`artist`表连接以获取艺术家姓名：

```
mysql> `CREATE TABLE tmp`
    -> `SELECT a_id, MAX(price) AS max_price FROM painting GROUP BY a_id;`
mysql> `SELECT artist.name, painting.title, painting.price`
    -> `FROM artist INNER JOIN painting INNER JOIN tmp`
    -> `ON painting.a_id = artist.a_id`
    -> `AND painting.a_id = tmp.a_id`
    -> `AND painting.price = tmp.max_price;`
+----------+-------------------+-------+
| name     | title             | price |
+----------+-------------------+-------+
| Da Vinci | Mona Lisa         |    87 |
| Van Gogh | The Potato Eaters |    67 |
| Renoir   | Les Deux Soeurs   |    64 |
+----------+-------------------+-------+
```

要避免显式创建临时表并以单个语句获得相同结果，请使用通用表达式（CTEs）。

```
WITH tmp AS (SELECT a_id, MAX(price) AS max_price FROM painting GROUP BY a_id) 
SELECT artist.name, painting.title, painting.price 
FROM artist INNER JOIN painting INNER JOIN tmp 
ON painting.a_id = artist.a_id AND 
painting.a_id = tmp.a_id AND painting.price = tmp.max_price;
```

我们在第 10.18 节中详细讨论了 CTEs。

通过在`FROM`子句中使用子查询检索与临时表中包含的相同行，可以以单个语句获得相同结果的另一种方法：

```
SELECT artist.name, painting.title, painting.price
FROM artist INNER JOIN painting INNER JOIN
(SELECT a_id, MAX(price) AS max_price FROM painting GROUP BY a_id) AS tmp
ON painting.a_id = artist.a_id
AND painting.a_id = tmp.a_id
AND painting.price = tmp.max_price;
```

另一种回答每组最大值问题的方法是使用`LEFT JOIN`将表与自身连接。以下语句识别每位艺术家 ID 的最高价画作（使用`IS NULL`选择`p1`中所有没有在`p2`中有更高价格行的行）：

```
mysql> `SELECT p1.a_id, p1.title, p1.price`
    -> `FROM painting AS p1 LEFT JOIN painting AS p2`
    -> `ON p1.a_id = p2.a_id AND p1.price < p2.price`
    -> `WHERE p2.a_id IS NULL;`
+------+-------------------+-------+
| a_id | title             | price |
+------+-------------------+-------+
|    1 | Mona Lisa         |    87 |
|    3 | The Potato Eaters |    67 |
|    4 | Les Deux Soeurs   |    64 |
+------+-------------------+-------+
```

要显示艺术家姓名而不是 ID 值，请将`LEFT JOIN`的结果与`artist`表连接：

```
mysql> `SELECT artist.name, p1.title, p1.price`
    -> `FROM painting AS p1 LEFT JOIN painting AS p2`
    -> `ON p1.a_id = p2.a_id AND p1.price < p2.price`
    -> `INNER JOIN artist ON p1.a_id = artist.a_id`
    -> `WHERE p2.a_id IS NULL;`
+----------+-------------------+-------+
| name     | title             | price |
+----------+-------------------+-------+
| Da Vinci | Mona Lisa         |    87 |
| Van Gogh | The Potato Eaters |    67 |
| Renoir   | Les Deux Soeurs   |    64 |
+----------+-------------------+-------+
```

使用临时表来回答最大-每组问题可能比使用临时表、CTE 或子查询更不直观的自连接`LEFT JOIN`方法。

## 另请参见

本篇介绍了如何通过将摘要信息选择到临时表中并将该表与原始表连接，或者通过在`FROM`子句中使用子查询来回答最大-每组问题的技术。这些技术在许多场景中都有应用。其中之一是计算团队排名，其中每个团队组的排名是通过将该组中的每个团队与成绩最佳的团队进行比较来确定的。第 17.12 节讨论了如何做到这一点。

# 16.8 使用联接填充或标识列表中的空白

## 问题

您希望按类别生成总结，但是一些类别在要总结的数据中不存在。因此，总结中也缺少这些类别。

## 解决方案

创建一个列出每个类别的参考表，并基于列表和包含数据的表之间的`LEFT` `JOIN`生成总结。结果中将显示参考表中的每个类别，即使这些类别在要总结的数据中不存在。

## 讨论

通常，总结查询仅对实际存在于数据中的类别生成条目。假设您想要总结`driver_log`表（见第九章介绍），以确定每天上路的司机数量。表具有以下行：

```
mysql> `SELECT * FROM driver_log ORDER BY rec_id;`
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

简单的总结显示每天活跃司机的数量如下：

```
mysql> `SELECT trav_date, COUNT(trav_date) AS drivers`
    -> `FROM driver_log GROUP BY trav_date ORDER BY trav_date;`
+------------+---------+
| trav_date  | drivers |
+------------+---------+
| 2014-07-26 |       1 |
| 2014-07-27 |       1 |
| 2014-07-29 |       3 |
| 2014-07-30 |       2 |
| 2014-08-01 |       1 |
| 2014-08-02 |       2 |
+------------+---------+
```

在这里，总结类别是日期，但从字面上来看，总结是<q>不完整的</q>，因为它仅包括`driver_log`表中表示的日期的条目。要生成一个包含所有类别（表中日期范围内的所有日期）的总结，包括那些没有活跃司机的日期，请创建一个参考表，列出每个日期：

```
mysql> `CREATE TABLE dates (d DATE);`
mysql> `INSERT INTO dates (d)`
    -> `VALUES('2014-07-26'),('2014-07-27'),('2014-07-28'),`
    -> `('2014-07-29'),('2014-07-30'),('2014-07-31'),`
    -> `('2014-08-01'),('2014-08-02');`
```

然后，使用`LEFT` `JOIN`将参考表与`driver_log`表联接：

```
mysql> `SELECT dates.d, COUNT(driver_log.trav_date) AS drivers`
    -> `FROM dates LEFT JOIN driver_log ON dates.d = driver_log.trav_date`
    -> `GROUP BY d ORDER BY d;`
+------------+---------+
| d          | drivers |
+------------+---------+
| 2014-07-26 |       1 |
| 2014-07-27 |       1 |
| 2014-07-28 |       0 |
| 2014-07-29 |       3 |
| 2014-07-30 |       2 |
| 2014-07-31 |       0 |
| 2014-08-01 |       1 |
| 2014-08-02 |       2 |
+------------+---------+
```

现在，总结包括范围内的每个日期的行，因为`LEFT` `JOIN`强制输出包含参考表中的每个日期的行，即使在`driver_log`表中缺少这些日期也是如此。

刚才展示的示例使用参考表与`LEFT` `JOIN`来填充总结中的空白。还可以使用参考表来*检测*数据集中的空白，即确定哪些类别在要总结的数据中不存在。以下语句显示了通过查找没有与类别值匹配的`driver_log`表行的参考行来找出在哪些日期上没有活跃司机：

```
mysql> `SELECT dates.d`
    -> `FROM dates LEFT JOIN driver_log ON dates.d = driver_log.trav_date`
    -> `WHERE driver_log.trav_date IS NULL;`
+------------+
| d          |
+------------+
| 2014-07-28 |
| 2014-07-31 |
+------------+
```

包含类别列表的参考表在总结上下文中非常有用，就像刚才展示的那样。但是手动创建这样的表格是枯燥乏味且容易出错的。使用递归 CTE 来实现这一目的要简单得多。

```
WITH RECURSIVE dates (d)  AS (
  SELECT '2014-07-26'
  UNION ALL
  SELECT d + INTERVAL 1 day
  FROM dates
  WHERE d < '2014-08-02') 
SELECT dates.d, COUNT(driver_log.trav_date) AS drivers
FROM dates LEFT JOIN driver_log ON dates.d = driver_log.trav_date
GROUP BY d ORDER BY d;
```

我们在第 15.16 节中更详细地讨论了递归 CTE。

如果您需要一个非常长的日期列表，预计会经常重复使用，您可能更喜欢将它们存储在表中，而不是每次都生成系列。在这种情况下，使用类别值范围的端点来生成参考表的存储过程将帮助自动化该过程。实质上，这种类型的存储过程充当一个迭代器，为范围内的每个值生成一行。以下存储过程 `make_date_list()` 展示了这种方法的一个示例。它创建一个包含特定日期范围内每个日期的参考表。它还对表进行索引，以便在大型连接中速度快：

```
CREATE PROCEDURE make_date_list(db_name TEXT, tbl_name TEXT, col_name TEXT,
                                min_date DATE, max_date DATE)
BEGIN
  DECLARE i, days INT;
  SET i = 0, days = DATEDIFF(max_date,min_date)+1;

  # Make identifiers safe for insertion into SQL statements. Use db_name
  # and tbl_name to create qualified table name.
  SET tbl_name = CONCAT(quote_identifier(db_name),'.',
                        quote_identifier(tbl_name));
  SET col_name = quote_identifier(col_name);
  CALL exec_stmt(CONCAT('DROP TABLE IF EXISTS ',tbl_name));
  CALL exec_stmt(CONCAT('CREATE TABLE ',tbl_name,'(',
                        col_name,' DATE NOT NULL, PRIMARY KEY(',
                        col_name,'))'));
  WHILE i < days DO
    CALL exec_stmt(CONCAT('INSERT INTO ',tbl_name,'(',col_name,') VALUES(',
                          QUOTE(min_date),' + INTERVAL ',i,' DAY)'));
    SET i = i + 1;
  END WHILE;
END;
```

使用 `make_date_list()` 生成参考表 `dates`，如下所示：

```
CALL make_date_list('cookbook', 'dates', 'd', '2014-07-26', '2014-08-02');
```

然后像本节前面展示的那样使用 `dates` 表，以填补摘要中的空洞或检测数据集中的空洞。

您可以在 `recipes` 发行版的 *joins* 目录中找到 `make_date_list()` 存储过程。它需要 *routines* 目录中的 `exec_stmt()` 和 `quote_identifier()` 辅助例程（参见 Recipe 11.6）。*joins* 目录还包含一个 Perl 脚本 *make_date_list.pl*，实现了一种替代方法；它可以从命令行生成日期参考表。

# 16.9 使用连接控制查询排序顺序

## 问题

您希望使用输出的特性对语句的输出进行排序，但无法使用 `ORDER BY` 指定输出的特性。例如，您希望按子组对一组行进行排序，首先放置具有最多行的组，最后放置具有最少行的组。但是，“每个组中的行数”不是单个行的属性，因此无法用于排序。

## 解决方案

派生排序信息并将其存储在辅助表中。然后将原始表与辅助表连接，使用辅助表来控制排序顺序。

## 讨论

大多数时候，您会使用 `ORDER BY` 子句对查询结果进行排序，指定用于排序的列。但有时，要排序的值并不在要排序的行中。这种情况出现在您希望使用组特征来排序行时。以下示例使用 `driver_log` 表来说明这一点。以下查询通过 ID 列对表进行排序，该列存在于行中：

```
mysql> `SELECT * FROM driver_log ORDER BY rec_id;`
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

但是，如果您想显示一个列表并根据行中不存在的汇总值对其进行排序，那就有点棘手了。假设您想按日期显示每个司机的行，但要先显示驾驶里程最多的司机。您无法使用汇总查询来实现这一点，因为那样您将无法获得单个司机的行。但是，如果没有汇总查询，也无法实现，因为排序需要汇总值。摆脱困境的方法是创建另一个表，其中包含每个司机的汇总值，并将其与原始表连接。这样，您可以生成单个行，并根据汇总值对它们进行排序。

要将司机总计汇总到另一个表中，请执行以下操作：

```
mysql> `CREATE TABLE tmp`
    -> `SELECT name, SUM(miles) AS driver_miles FROM driver_log GROUP BY name;`
```

这将产生我们需要按正确总英里数排序名称的值：

```
mysql> `SELECT * FROM tmp ORDER BY driver_miles DESC;`
+-------+--------------+
| name  | driver_miles |
+-------+--------------+
| Henry |          911 |
| Suzi  |          893 |
| Ben   |          362 |
+-------+--------------+
```

然后使用`name`值将汇总表连接到`driver_log`表，并使用`driver_miles`值对结果进行排序：

```
mysql> `SELECT tmp.driver_miles, driver_log.*`
    -> `FROM driver_log INNER JOIN tmp ON driver_log.name = tmp.name`
    -> `ORDER BY tmp.driver_miles DESC, driver_log.trav_date;`
+--------------+--------+-------+------------+-------+
| driver_miles | rec_id | name  | trav_date  | miles |
+--------------+--------+-------+------------+-------+
|          911 |      6 | Henry | 2014-07-26 |   115 |
|          911 |      4 | Henry | 2014-07-27 |    96 |
|          911 |      3 | Henry | 2014-07-29 |   300 |
|          911 |     10 | Henry | 2014-07-30 |   203 |
|          911 |      8 | Henry | 2014-08-01 |   197 |
|          893 |      2 | Suzi  | 2014-07-29 |   391 |
|          893 |      7 | Suzi  | 2014-08-02 |   502 |
|          362 |      5 | Ben   | 2014-07-29 |   131 |
|          362 |      1 | Ben   | 2014-07-30 |   152 |
|          362 |      9 | Ben   | 2014-08-02 |    79 |
+--------------+--------+-------+------------+-------+
```

上述声明显示了结果中的英里总数。这只是为了澄清数值如何排序。实际上并不需要显示它们；它们只需要用于`ORDER BY`子句。

为了避免使用临时表，可以使用 CTE：

```
WITH tmp AS
(SELECT name, SUM(miles) AS driver_miles FROM driver_log GROUP BY name)
SELECT tmp.driver_miles, driver_log.*
FROM driver_log INNER JOIN tmp ON driver_log.name = tmp.name
ORDER BY tmp.driver_miles DESC, driver_log.trav_date;
```

或者，在`FROM`子句中使用子查询选择相同的行：

```
SELECT tmp.driver_miles, driver_log.*
FROM driver_log INNER JOIN
(SELECT name, SUM(miles) AS driver_miles
FROM driver_log GROUP BY name) AS tmp
ON driver_log.name = tmp.name
ORDER BY tmp.driver_miles DESC, driver_log.trav_date;
```

# 16.10 连接多个查询的结果

## 问题

您希望连接两个或多个查询的结果。

## 解决方案

运行查询并将结果存储在临时表中，然后访问这些临时表以获取最终结果。或者，使用命名子查询，然后连接它们的结果。或者，使用我们最喜欢的方法：使用 CTE 以最简单和清晰的方式执行此任务。

## 讨论

您可能不仅需要连接表，还需要连接其他查询的结果。假设您正在使用`recipes`分发中的`city`和`states`表，并希望找到属于人口最多的 10 个州的首府名称。同时，您希望仅将最大城市与首府相同的州包含在搜索结果中。

这个任务如果你首先将其分解为三个部分，非常容易解决：

1.  查找所有首府和最大城市相同的州。可以通过以下查询完成：

    ```
    SELECT * FROM city WHERE capital=largest;
    ```

1.  找出人口最多的 10 个州：

    ```
    SELECT * FROM states ORDER BY pop DESC LIMIT 10;
    ```

1.  连接结果以选择存在于两者中的行。

有三种方法可以做到这一点：通过创建中间临时表、通过连接子查询结果以及使用 CTE。

### 使用中间临时表

将查询结果存储到临时表中，然后从中选择。

```
mysql> `CREATE` `TEMPORARY` `TABLE` `large_capitals`
    -> `SELECT` `*` `FROM` `city` `WHERE` `capital``=``largest``;`
Query OK, 17 rows affected (0,00 sec)
Records: 17  Duplicates: 0  Warnings: 0

mysql> `CREATE` `TEMPORARY` `TABLE` `top10states`
    -> `SELECT` `*` `FROM` `states` `ORDER` `BY` `pop` `DESC` `LIMIT` `10``;`
Query OK, 10 rows affected (0,00 sec)
Records: 10  Duplicates: 0  Warnings: 0

mysql> `SELECT` `state``,` `capital``,` `pop` `FROM`
    -> `large_capitals` `JOIN` `top10states`
    -> `ON``(``large_capitals``.``state` `=` `top10states``.``name``)``;`
+---------+----------+----------+ | state   | capital  | pop      |
+---------+----------+----------+ | Georgia | Atlanta  | 10799566 |
| Ohio    | Columbus | 11780017 |
+---------+----------+----------+ 2 rows in set (0,00 sec)
```

###### 提示

`CREATE TABLE`语句中的关键字`TEMPORARY`指示 MySQL 创建一个表，仅对当前会话可见，会话关闭后将被销毁。有关详细信息，请参见 Recipe 6.3。

### 使用命名子查询

如果您只需要访问一次中间结果，可以通过使用子查询和连接它们的结果来避免创建临时表。

```
mysql> `SELECT` `state``,` `capital``,` `pop` `FROM` ![1](img/1.png)
    -> `(``SELECT` `*` `FROM` `city` `WHERE` `capital``=``largest``)` `AS` `large_capitals``,` ![2](img/2.png)
    -> `(``SELECT` `*` `FROM` `states` `ORDER` `BY` `pop` `DESC` `LIMIT` `10``)` `AS` `top10states` ![3](img/3.png)
    -> `WHERE` `large_capitals``.``state` `=` `top10states``.``name``;` ![4](img/4.png)
+---------+----------+----------+ | state   | capital  | pop      |
+---------+----------+----------+ | Georgia | Atlanta  | 10799566 |
| Ohio    | Columbus | 11780017 |
+---------+----------+----------+ 2 rows in set (0,00 sec)
```

![1](img/#co_nch-multi-multi-subquery-join-outer_co)

从选择最终结果中所需的列开始查询

![2](img/#co_nch-multi-multi-subquery-join-lc_co)

将第一个子查询放入括号中，并分配一个唯一名称。

![3](img/#co_nch-multi-multi-subquery-join-ts_co)

对第二个子查询执行相同操作。

![4](img/#co_nch-multi-multi-subquery-join-where_co)

使用`WHERE`子句缩小搜索范围。

### 使用通用表达式（CTEs）

使用 CTEs 首先为您的子查询命名，然后像处理常规 MySQL 表一样连接它们的结果。

```
mysql> `WITH`
    -> `large_capitals` `AS` `(``SELECT` `*` `FROM` `city` `WHERE` `capital``=``largest``)``,`
    -> `top10states` `AS` `(``SELECT` `*` `FROM` `states` `ORDER` `BY` `pop` `DESC` `LIMIT` `10``)`
    -> `SELECT` `state``,` `capital``,` `pop` 
    -> `FROM` `large_capitals` `JOIN` `top10states`
    -> `ON` `(``large_capitals``.``state` `=` `top10states``.``name``)``;`
+---------+----------+----------+ | state   | capital  | pop      |
+---------+----------+----------+ | Georgia | Atlanta  | 10799566 |
| Ohio    | Columbus | 11780017 |
+---------+----------+----------+ 2 rows in set (0,00 sec)
```

# 16.11 在程序中引用连接输出列名

## 问题

您需要在程序内部处理连接的结果，但结果集中的列名不唯一。

## 解决方案

重写查询，使用列别名使每列具有唯一名称。或者，按位置引用列。

## 讨论

连接通常从相关表中检索列，并且从不同表中选择的列具有相同的名称并不罕见。考虑以下显示艺术品收藏中项目的联接的情况。对于每幅绘画，它显示艺术家姓名，绘画标题，您获取项目的州份及其价格：

```
mysql> `SELECT artist.name, painting.title, states.name, painting.price`
    -> `FROM artist INNER JOIN painting INNER JOIN states`
    -> `ON artist.a_id = painting.a_id AND painting.state = states.abbrev;`
+----------+-------------------+----------+-------+
| name     | title             | name     | price |
+----------+-------------------+----------+-------+
| Da Vinci | The Last Supper   | Indiana  |    34 |
| Da Vinci | Mona Lisa         | Michigan |    87 |
| Van Gogh | Starry Night      | Kentucky |    48 |
| Van Gogh | The Potato Eaters | Kentucky |    67 |
| Renoir   | Les Deux Soeurs   | Nebraska |    64 |
+----------+-------------------+----------+-------+
```

该语句对每个输出列使用表限定符，但 MySQL 不包括表名在列标题中，因此输出中并非所有列名都是唯一的。如果在程序中处理连接结果，并将行提取到引用列值的数据结构中，非唯一列名会导致值无法访问。假设您在 Perl DBI 脚本中这样提取行：

```
while (my $ref = $sth->fetchrow_hashref ())
{
  *`... process row hash here ...`*
}
```

将行提取到哈希中产生三个哈希元素（`name`，`title`，`price`）；其中一个`name`元素丢失了。为了解决此问题，提供使列名唯一的别名：

```
SELECT artist.name AS painter, painting.title,
  states.name AS state, painting.price
FROM artist INNER JOIN painting INNER JOIN states
ON artist.a_id = painting.a_id AND painting.state = states.abbrev;
```

现在将行提取到哈希中产生四个哈希元素（`painter`，`title`，`state`，`price`）。

为了解决不更改列名的问题，将行提取到除哈希之外的其他内容。例如，将行提取到数组中，并按数组中的序数位置引用列：

```
while (my @val = $sth->fetchrow_array ())
{
  print "painter: $val[0], title: $val[1], "
        . "state: $val[2], price: $val[3]\n";
}
```
