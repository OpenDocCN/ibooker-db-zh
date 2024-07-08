# 第二十一章：查询性能

# 21.0 引言

如果创建并按预期使用，索引用于快速查找行。以下是使用索引的主要原因。

+   高效利用`SELECT`语句中的`WHERE`子句来查找行。

+   通过索引中存储的值的唯一性（称为基数）和返回的最少行数来找到最佳的查询执行计划。

+   启用不同表之间的连接操作。

索引对于高效地扫描和搜索表中的值至关重要。没有索引，当执行查询时，MySQL 需要读取给定表中的所有行。由于不同的表大小，MySQL 必须将从表中读取的所有数据带入内存，并且仅能对选择的数据进行排序、过滤和返回值。此操作可能需要额外的资源将数据复制到新的临时表以执行排序操作。索引对查询性能至关重要；因此，未索引的表对数据库而言是一个重大负担，除非它们是小型参考表。

为了快速查询性能，每个表都需要一个代表一个或多个列的主键。在使用 InnoDB 存储引擎时，表的数据是物理排序的，以便使用主键列进行快速查找和排序。理想的表设计使用覆盖索引，在查询结果中使用索引列进行计算。MySQL 使用的大多数索引存储在 B 树中，这使得由于减少数据访问时间而实现快速数据访问成为可能。

如果表的数据量很大，并且没有任何键创建额外的字段，如`table_name_id`作为主键，在进行连接操作时可以带来显著的好处。InnoDB 表始终具有代表主键的聚簇索引，如果用户尚未创建。聚簇索引是一个表，其中数据和行按照主键值的顺序存储在表中的一方向。

###### 注意

如果查询中没有使用 WHERE 子句，则 MySQL 优化器将进行全表扫描。例如：

```
SELECT * FROM customer;
```

这不会改变`customer`表是否存在索引。

在开始使用索引策略之前，以下是一些您需要了解的关键术语：

表扫描

表扫描意味着在执行查询时读取给定表中的所有行。开发人员应尽量避免全表扫描，包括执行`COUNT(*)`操作。

树遍历

树遍历是索引用于访问数据的一种方法。索引的目标是通过遍历尽量少的跳跃来获取数据。叶节点越少，索引遍历速度越快。

叶节点

叶节点是 B 树索引结构的一部分。它们随着数据变化而维护索引的变化。它们建立了一个双向链接列表以连接索引叶节点。

B 树结构

B 树是一种自平衡树数据结构，它保持数据排序并允许以对数时间进行搜索、顺序访问、插入和删除。B 树是二叉搜索树的一种泛化形式，一个节点可以有多于两个的子节点。

虽然 B 树索引通常在 MySQL 存储引擎中使用，但哈希索引使用不同类型的数据结构。哈希索引具有不同的特性，并且具有其自己的用例。有关详细信息，请参阅 MySQL 用户参考手册中的[“B-Tree 和 Hash 索引比较”](https://dev.mysql.com/doc/refman/8.0/en/index-btree-hash.html)。

###### 警告

虽然索引有助于更快地检索行，但是过度创建或保留未使用的索引会对数据库的 I/O 操作构成负担。每个索引叶页（索引的最低级别，其中所有键按排序顺序出现）必须针对所有`UPDATE`/`INSERT`/`DELETE`操作进行维护，因此会产生额外的开销。

# 21.1 创建索引

## 问题

您的查询响应非常慢。

## 解决方案

在您的列上创建一个索引，以仅检索您正在寻找的行。

## 讨论

没有索引的表只是随机写入的日志数据，没有查找的参考。因此，对这样的表的大多数查询都很慢。唯一的例外是仅适用于具有有限行数的参考表，具体取决于模式设计。

MySQL 建议为每个表的每一行都提供一个具有`NOT NULL`特性的主键列。

我们有一张名为`top_names`的表，来自`Names_2010Census.csv`数据。

```
mysql> `CREATE` `TABLE` `` ` ```top_names``` ` `` `(`
  `` ` ```top_name``` ` `` `varchar``(``25``)` `DEFAULT` `NULL``,`
  `` ` ```name_rank``` ` `` `smallint` `DEFAULT` `NULL``,`
  `` ` ```name_count``` ` `` `int` `DEFAULT` `NULL``,`
  `` ` ```prop100k``` ` `` `decimal``(``8``,``2``)` `DEFAULT` `NULL``,`
  `` ` ```cum_prop100k``` ` `` `decimal``(``8``,``2``)` `DEFAULT` `NULL``,`
  `` ` ```pctwhite``` ` `` `decimal``(``5``,``2``)` `DEFAULT` `NULL``,`
  `` ` ```pctblack``` ` `` `decimal``(``5``,``2``)` `DEFAULT` `NULL``,`
  `` ` ```pctapi``` ` `` `decimal``(``5``,``2``)` `DEFAULT` `NULL``,`
  `` ` ```pctaian``` ` `` `decimal``(``5``,``2``)` `DEFAULT` `NULL``,`
  `` ` ```pct2prace``` ` `` `decimal``(``5``,``2``)` `DEFAULT` `NULL``,`
  `` ` ```pcthispanic``` ` `` `decimal``(``5``,``2``)` `DEFAULT` `NULL`
`)` `ENGINE``=``InnoDB` `DEFAULT` `CHARSET``=``utf8mb4` `COLLATE``=``utf8mb4_0900_ai_ci``;`
```

然后加载数据：

```
mysql> `LOAD` `DATA` `LOCAL` `INFILE` `'Names_2010Census.csv'` `into` `table` `top_names` 
    -> `FIELDS` `TERMINATED` `BY` `','` `ENCLOSED` `BY` `'"'` `LINES` `TERMINATED` `BY` `'\n'``;`
Query OK, 162255 rows affected, 65535 warnings (0.93 sec)
Records: 162255  Deleted: 0  Skipped: 0  Warnings: 444813
```

现在我们已经创建并加载了我们的表，可以继续进行以下查询。

```
mysql> `SELECT` `names_id``,``top_name``,``name_rank` 
	`-``>` `FROM` `top_names` `WHERE` `top_name` `=` `"BROWN"``;`
+----------+----------+-----------+ | names_id | top_name | name_rank |
+----------+----------+-----------+ |        5 | BROWN    |         4 |
+----------+----------+-----------+ 1 row in set (0.04 sec)
```

如下所示，MySQL 必须对该表进行完整的表扫描，以查找其`PRIMARY KEY`之外的任何行。

```
mysql> `EXPLAIN SELECT names_id,top_name,name_rank  	-> FROM top_names WHERE top_name = "BROWN"\G`
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 161533
     filtered: 10.00
        Extra: Using where
1 row in set, 1 warning (0.01 sec)
```

我们的示例查询在`top_name`字段上寻找字符串匹配；因此，在这种数据类型上创建索引将提高查询性能。首先，我们创建一个索引以满足该查询的`WHERE`子句。

```
mysql> `CREATE` `INDEX` `idx_names` `ON` `top_names``(``top_name``)``;` 
Query OK, 0 rows affected (0.28 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

然后我们检查优化器是否选择了同一查询的这个新索引。

```
mysql> `EXPLAIN` `SELECT` `names_id``,``top_name``,``name_rank` 
`-``>` `FROM` `top_names` `WHERE` `top_name` `=` `"BROWN"``\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: ref
possible_keys: idx_names
          key: idx_names
      key_len: 103
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

可能需要删除索引的原因有几个。确保不再需要索引或需要重新创建索引后，可以使用以下语法删除它们。

```
DROP INDEX index_name ON tbl_name;
```

# 21.2 创建代理主键

## 问题

没有主键的表性能不够好。

## 解决方案

在所有 InnoDB 表上添加主键。

## 讨论

主键为您提供了在表中唯一标识行的方式。在 InnoDB 中，主键等同于聚集索引：一种特殊的索引，用于存储行数据。当用户在没有明确定义主键的情况下创建 InnoDB 表时，InnoDB 会选择表中存在的第一个带有 B-Tree 结构的唯一索引，并将其作为聚集索引。聚集索引通常也称为记录在磁盘上的物理顺序。如果没有唯一索引存在，则 InnoDB 会创建一个名为`GEN_CLUST_INDEX`的代理键，这是一个自动生成的唯一 6 字节标识符。

当 InnoDB 创建辅助索引时，复制主键列到每个辅助索引行是有用的，因为它能解决查询。如果主键不必要地大，则所有辅助索引也会很大。因此，选择适合的列作为主键非常重要。

在我们的示例中，在 Recipe 21.1 中，自然主键是`top_name`，占用 26 字节。将`top_name`定义为主键将使每个辅助索引中的每一行的大小增加 26 字节。因此，我们在这里展示了一种创建 4 字节整数代理键的技术，其属性为`AUTO_INCREMENT`，因此它是单调递增的。这比 InnoDB 显式创建的代理键更好，因为它更小且我们完全控制其值。

我们的表相对较小，但对于大表，这种差异可能是关键的。除了空间外，更大的索引需要更多时间进行搜索。

这张表缺少一个带有`PRIMARY KEY`的字段。包含这种表的最佳方法是添加一个具有`AUTO INCREMENT NOT NULL`属性的`id`字段。理想情况下，在加载任何数据到表之前创建这个字段，以便在表空间中物理上对表进行排序。

```
mysql> `` ALTER TABLE `top_names` `` 
    -> ``ADD COLUMN `names_id` int unsigned NOT NULL      -> AUTO_INCREMENT PRIMARY KEY FIRST;``
Query OK, 0 rows affected (0.44 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

尽管以下是一个完整的索引扫描，它将使用我们创建的新`PRIMARY KEY`字段来计算表中行的数量。

```
mysql> `SELECT COUNT(names_id) FROM top_names;`
+-----------------+
| count(names_id) |
+-----------------+
|          162255 |
+-----------------+
1 row in set (0.04 sec)

mysql> `EXPLAIN SELECT COUNT(names_id) FROM top_names\G`
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: index
possible_keys: NULL
          key: idx_name_rank_count
      key_len: 8
          ref: NULL
         rows: 161533
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
```

## 参见

若要获取更多信息，请参阅 MySQL 文档中关于[主键优化](https://dev.mysql.com/doc/refman/8.0/en/primary-key-optimization.html)的详细信息。

# 21.3 索引维护

## 问题

您想知道现有的索引是否对您的查询有效，并且丢弃那些无效的索引。

## 解决方案

学习基本的索引操作。

## 讨论

为了更好地控制您的数据，通过研究模式数据和访问类型有效地使用索引。继续我们之前示例中的例子，我们将检查`top_names`表的现有索引。

```
mysql> `SHOW` `INDEXES` `FROM` `top_names` `\``G`
*************************** 1. row ***************************
        Table: top_names
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: names_id
    Collation: A
  Cardinality: 161807
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE
      Comment:
Index_comment:
      Visible: YES
   Expression: NULL
*************************** 2. row ***************************
        Table: top_names
   Non_unique: 1
     Key_name: idx_names
 Seq_in_index: 1
  Column_name: top_name
    Collation: A
  Cardinality: 161708
     Sub_part: NULL
       Packed: NULL
         Null: YES
   Index_type: BTREE
      Comment:
Index_comment:
      Visible: YES
   Expression: NULL
2 rows in set (0.00 sec)
```

在这里，最重要的是索引的基数。如果列具有许多不同的值，则索引的利用率会更高。总之，索引在布尔和冗余值上效率低下。

在我们的情况中，`idx_names` 索引的基数接近主键的基数。这显示该索引具有良好的选择性。实际上，这个索引也可以是 `unique` 的，我们可以通过查询该列中不同值的数量来确认。

```
mysql> `SELECT` `COUNT``(``DISTINCT` `top_name``)``,` `COUNT``(``*``)` `FROM` `top_names``;`
+--------------------------+----------+ | count(distinct top_name) | count(*) |
+--------------------------+----------+ |                   162254 |   162254 |
+--------------------------+----------+ 1 row in set (0,18 sec)
```

由于我们已经在 `top_name` 列上创建了一个索引，我们可以删除该索引，然后创建一个新的唯一索引。首先执行以下命令来删除索引。

```
mysql> `DROP` `INDEX` `idx_names` `ON` `top_names``;`
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

或者，也可以使用 `ALTER TABLE` 语法。

```
ALTER TABLE tbl_name DROP INDEX name;
```

若要在 `CREATE INDEX` 命令中创建唯一索引，请指定关键字 `UNIQUE`：

```
CREATE UNIQUE INDEX idx_names ON top_names(top_name);
```

您可以重命名在表上创建的现有索引。

```
mysql> `ALTER TABLE top_names RENAME` 
    -> `INDEX idx_names to idx_top_name, ALGORITHM=INPLACE, LOCK=NONE;`
```

###### 警告

并非所有的索引操作都是原地的；一些索引操作会导致表重建，这可能会对大数据量下的服务器性能产生负面影响。在执行 DDL 操作之前必须小心。DDL（数据定义语言）指的是更改表定义或称为数据库架构的结构。详细信息请参阅 [MySQL Documentation.](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html#online-ddl-index-operations)

# 21.4 决定查询何时可以使用索引

## 问题

您的表有一个索引，但查询仍然很慢。

## 解决方案

使用 `EXPLAIN` 检查查询计划，确保使用了正确的索引。

## 讨论

索引是查询计划的一部分，通过最短可能的路径访问数据以加快访问速度。当 MySQL 优化器做出决策时，它考虑索引、基数、行数等因素。以下是一个查询的示例，其中存在一个列的索引，但 MySQL 无法利用它。

```
mysql> `EXPLAIN` `SELECT` `name_rank``,``top_name``,``name_count` `FROM` `top_names` 
    -> `WHERE` `name_rank` `<` `10` `ORDER` `BY` `name_count``\``G`
      *************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 161604
     filtered: 33.33
        Extra: Using where; Using filesort
1 row in set, 1 warning (0.00 sec)
```

从 `Explain` 计划输出来看，我们没有符合查询关键条件的索引。表上有索引，看起来我们需要在 `name_rank` 字段上再创建一个索引。

```
mysql> `CREATE` `INDEX` `idx_name_rank` `ON` `top_names``(``name_rank``)``;`
	Query OK, 0 rows affected (0.16 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

创建新索引后再次检查查询计划：

```
mysql> `EXPLAIN` `SELECT` `name_rank``,``top_name``,``name_count` `FROM` `top_names` 
    -> `WHERE` `name_rank` `<` `10` `ORDER` `BY` `name_count``\``G`
      *************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: range
possible_keys: idx_name_rank
          key: idx_name_rank
      key_len: 3
          ref: NULL
         rows: 11
     filtered: 100.00
        Extra: Using index condition; Using filesort
1 row in set, 1 warning (0.00 sec)
```

我们的查询在 `top_names` 表中寻找小于十的 `name_rank`。如果没有新创建的 `idx_name_rank` 索引，优化器必须评估表中的所有 161604 行才能返回 11 行。有了索引，它只需访问这 11 行。

# 21.5 决定多列索引的顺序

## 问题

您希望加快多列查询的速度。

## 解决方案

使用包含多列的覆盖索引。

## 讨论

如果查询结果完全是从索引页计算而不是读取实际表数据，则可以实现最佳查询性能。覆盖索引是针对引用多个列的查询的解决方案。这种类型的索引包含所需的数据，因此不需要在表上执行额外的读取。

在以下示例中，我们有一个查询，需要在一个列（`name_rank`）上添加过滤器，并按另一个列（`name_count`）排序。

```
mysql> `SELECT` `name_rank``,``top_name``,``name_count` `FROM` `top_names` 
    -> `WHERE` `name_rank` `<` `10` `ORDER` `BY` `name_count``\``G`
```

我们将在认为优化器选择最快路径所需的列上创建一个索引。

```
mysql> `CREATE` `INDEX` `idx_name_rank_count` `ON` `top_names``(``name_count``,``name_rank``)``;`
Query OK, 0 rows affected (0.18 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

在这种情况下，MySQL 无法对以下查询使用索引，最终需要再次执行全表扫描。原因是，尽管查询的两列都在过滤器中，但索引的顺序是反向的。

```
mysql> `EXPLAIN` `SELECT` `name_rank``,``top_name``,``name_count` `FROM` `top_names` 
    -> `WHERE` `name_rank` `<` `10` `ORDER` `BY` `name_count``\``G`
      *************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 161604
     filtered: 33.33
        Extra: Using where; Using filesort
1 row in set, 1 warning (0.00 sec)
```

为了演示索引列顺序的重要性，让我们看下面的例子。

对于 ``KEY `idx_name_rank_count` (`name_rank`,`name_count`)``，首先按相反顺序删除先前的索引，然后创建一个新索引。

```
mysql> `DROP` `INDEX` `idx_name_rank_count` `ON` `top_names``;`
Query OK, 0 rows affected (0.01 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

```
mysql> `CREATE` `INDEX` `idx_name_rank_count` `ON` `top_names``(``name_rank``,``name_count``)``;`
Query OK, 0 rows affected (0.15 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

我们已为我们的 `SELECT` 语句提议的 `name_rank` 和 `name_count` 过滤器创建了覆盖索引。

```
mysql> `EXPLAIN` `SELECT` `name_rank``,``top_name``,``name_count` `FROM` `top_names` 
    -> `WHERE` `name_rank` `<` `10` `ORDER` `BY` `name_count``\``G`
	*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: range
possible_keys: idx_name_rank_count
          key: idx_name_rank_count
      key_len: 3
          ref: NULL
         rows: 11
     filtered: 100.00
        Extra: Using index condition; Using filesort
1 row in set, 1 warning (0.00 sec)
```

正如您从 `EXPLAIN` 输出中可以看到的那样，优化器为此查询选择了 `idx_name_rank_count`，并创建了一个新的覆盖索引。

# 21.6 使用升序和降序索引

## 问题

您希望在没有性能惩罚的情况下按升序或降序扫描数据。

## 解决方案

使用升序和降序索引。

## 讨论

MySQL 可以以反向顺序扫描索引，这会导致性能下降，因为索引页面是物理排序的。为了为 `ORDER BY` 子句创建匹配的索引，使用 `DESC` 表示降序，`ASC` 表示升序索引类型。

理想的查询性能来源于避免反向扫描索引。这也是排序和使用 `DESC` 索引进行过滤的结合体。当 MySQL 优化器选择查询计划时，它评估是否可以在需要降序时利用这些优势。

请记住，降序索引仅受 InnoDB 存储引擎支持，并且在使用时存在一些限制。

此外，降序索引具有以下特性：

+   它们受所有数据类型支持。

+   `DISTINCT` 子句可以使用具有匹配列的任何索引。

+   它们可以在不与 `GROUP BY` 子句一起使用时用于 `MIN()`/`MAX()` 优化。

+   它们仅限于 `BTREE` 和 `HASH` 索引。

+   它们不支持 `FULLTEXT` 或 `SPATIAL` 索引类型。

以下示例从创建所需字段的覆盖索引开始。

```
CREATE INDEX idx_desc_01 ON top_names(top_name, prop100k ASC, cum_prop100K DESC);
```

```
mysql> `SELECT` `top_name``,``prop100k``,``cum_prop100k` `FROM` `top_names` 
    -> `ORDER` `BY` `` ` ```top_name``` ` ```,``` ` ```prop100k``` ` ```,``` ` ```cum_prop100k``` ` `` `DESC` `LIMIT` `10``;`
+----------+----------+--------------+ | top_name | prop100k | cum_prop100k |
+----------+----------+--------------+ | NULL     |     2.43 |     60231.65 |
| AAB      |     0.05 |     88770.96 |
| AABERG   |     0.16 |     82003.18 |
| AABY     |     0.07 |     86239.41 |
| AADLAND  |     0.13 |     83329.35 |
| AAFEDT   |     0.05 |     88567.34 |
| AAGAARD  |     0.10 |     84574.52 |
| AAGARD   |     0.12 |     83769.42 |
| AAGESEN  |     0.06 |     87383.27 |
| AAKER    |     0.12 |     83574.66 |
+----------+----------+--------------+ 10 rows in set (0.00 sec)
```

在为所有 `ORDER BY` 子句创建覆盖索引后，优化器选择 `idx_desc_01`。这对于索引优化和排序特别有利。

```
mysql> `EXPLAIN SELECT top_name,prop100k,cum_prop100k FROM top_names` 
    -> ``ORDER BY `top_name`,`prop100k`,`cum_prop100k` DESC LIMIT 10\G``
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: index
possible_keys: NULL
          key: idx_desc_01
      key_len: 113
          ref: NULL
         rows: 10
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
```

当我们执行 `SELECT * FROM top_names` 时，而不是指定 `top_name` 字段的列顺序，它使用先前创建的索引，默认为升序排列。

```
mysql> `EXPLAIN` `SELECT` `*` `FROM` `top_names` `ORDER` `BY` `top_name` `ASC` `LIMIT` `10` `\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: index
possible_keys: NULL
          key: idx_top_name
      key_len: 103
          ref: NULL
         rows: 10
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

为了演示降序索引的使用，我们将创建一个新索引，并使用 `DESC` 应用它。

```
mysql> `CREATE` `INDEX` `idx_desc_02` `ON` `top_names``(``top_name` `DESC``,` `prop100k``,` `cum_prop100K``)``;`
Query OK, 0 rows affected (0.38 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> `EXPLAIN` `SELECT` `*` `FROM` `top_names` `ORDER` `BY` `top_name` `DESC` `LIMIT` `10``\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: index
possible_keys: NULL
          key: idx_desc_02
      key_len: 113
          ref: NULL
         rows: 10
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

再次使用 `top_name` 和另一列 `prop100k` 来说明在 `top_name` 列上使用 `DESC` 索引的用法。

```
mysql> `EXPLAIN` `SELECT` `top_name` `FROM` `top_names`
    -> `ORDER` `BY` `top_name` `DESC``,``prop100k` `ASC` `LIMIT` `10` `\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: index
possible_keys: NULL
          key: idx_desc_02
      key_len: 113
          ref: NULL
         rows: 10
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
```

###### 注意

顺序很重要，因为 MySQL 使用左侧最左匹配规则来与`ORDER BY`子句中的索引进行比较。在复合索引中更改列的顺序将改变查询结果的行为。此外，在排序多个字段时小心使用`SELECT * FROM`，因为`*`将使用表定义中的列顺序，可能导致与`ORDER BY`子句意图不符的字段。

# 21.7 使用基于函数的索引

## 问题

您需要按表达式搜索或排序，但 MySQL 为每行计算表达式的结果，因此无法使用索引。查询的性能较差。

## 解决方案

使用功能索引。

## 讨论

有些类型的信息更容易通过不是原始值而是从它们计算得到的表达式来分析。例如，表`mail`中的列`size`存储的是字节大小，在第一次查看时很难解释。如果使用千字节(`KB`)会更容易处理。但是，您可能不想失去存储在字节中提供的精度。

如果您将数据存储在字节中并使用表达式查询表，则可以兼顾精度和可用性。例如，要查找大于 100 KB 的消息，请使用以下查询。

```
mysql> `SELECT` `t``,` `srcuser``,` `srchost``,` `size``,` `ROUND``(``size``/``1024``)` `AS` `size_KB` 
    ->  `FROM` `mail` `WHERE` `ROUND``(``size``/``1024``)` `>` `100``;`
+---------------------+---------+---------+---------+---------+ | t                   | srcuser | srchost | size    | size_KB |
+---------------------+---------+---------+---------+---------+ | 2014-05-12 12:48:13 | tricia  | mars    |  194925 |     190 |
| 2014-05-14 17:03:01 | tricia  | saturn  | 2394482 |    2338 |
| 2014-05-15 10:25:52 | gene    | mars    |  998532 |     975 |
+---------------------+---------+---------+---------+---------+ 3 rows in set (0.00 sec)
```

然而，MySQL 无法使用列`size`上的索引来解决此查询，因为它为每行计算表达式。为解决此问题，请使用基于函数的索引。

函数索引的语法如下：

```
CREATE INDEX *`index_name`* ON *`table_name`* ((*`expression`*));
```

注意双括号：如果省略一对括号，MySQL 将认为您传递的是列名而不是表达式，并返回错误。

让我们在`ROUND(size/1024)`上创建一个索引，并检查 MySQL 是否会使用它来解决查询。

```
mysql> `CREATE` `INDEX` `size_KB` `ON` `mail` `(``(``ROUND``(``size``/``1024``)``)``)``;`
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> `EXPLAIN` `SELECT` `t``,` `srcuser``,` `srchost``,` `size``,` `ROUND``(``size``/``1024``)` `AS` `size_KB` 
    ->  `FROM` `mail` `WHERE` `ROUND``(``size``/``1024``)` `>` `100``\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: mail
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 16
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

该索引将不会被用于解决查询，因为函数`ROUND`返回具有浮点的数据类型`NEWDECIMAL`和`100`是`LONGLONG`。如果您使用选项`--column-type-info`启动 *mysql* 客户端，您可以检查结果：

```
$ `unbuffer mysql --column-type-info -e "SELECT ROUND(10.5)" | grep Type`
Type:       NEWDECIMAL
$ `unbuffer mysql --column-type-info -e "SELECT 100" | grep Type`
Type:       LONGLONG
```

###### 小贴士

我们需要使用命令*unbuffer*，因为*mysql*会缓冲`--column-type-info`的结果，否则不能将其传输到*grep*。

为了澄清，强制 MySQL 使用索引，您需要将表达式的结果与浮点值进行比较。

```
mysql> `EXPLAIN` `SELECT` `t``,` `srcuser``,` `srchost``,` `size``,` `ROUND``(``size``/``1024``)` `AS` `size_KB`
    -> `FROM` `mail` `WHERE` `ROUND``(``size``/``1024``)` `>` `100``.``0``\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: mail
   partitions: NULL
         type: range
possible_keys: size_KB
          key: size_KB
      key_len: 10
          ref: NULL
         rows: 3
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

或者，在创建索引时将函数`ROUND`的结果转换为整数值。这也会强制 MySQL 使用索引来解决查询。

```
mysql> `DROP` `INDEX` `size_KB` `ON` `mail``;`
Query OK, 0 rows affected (0.01 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> `CREATE` `INDEX` `size_KB` `ON` `mail` `(``(``CAST``(``ROUND``(``size``/``1024``)` `AS` `SIGNED``)``)``)``;`
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> `EXPLAIN` `SELECT` `t``,` `srcuser``,` `srchost``,` `size``,` `ROUND``(``size``/``1024``)` `AS` `size_KB` 
    -> `FROM` `mail` `WHERE` `ROUND``(``size``/``1024``)` `>` `100``\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: mail
   partitions: NULL
         type: range
possible_keys: size_KB
          key: size_KB
      key_len: 9
          ref: NULL
         rows: 3
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

# 21.8 使用生成列上的索引与 JSON 数据

## 问题

您想在 JSON 数据内执行搜索，但速度较慢。

## 解决方案

使用一个从搜索 JSON 值的表达式创建的生成列，并在此列上创建索引。

## 讨论

在这个示例中，我们将讨论一个表`book_authors`。

```
CREATE TABLE `book_authors` (
  `id` int NOT NULL AUTO_INCREMENT,
  `author` json NOT NULL,
  PRIMARY KEY (`id`)
);
```

表中包含每位作者的书籍记录在 JSON 列中。

```
mysql> `SELECT` `*` `FROM` `book_authors``\``G`
*************************** 1. row ***************************
      id: 1
  author: {"id": 1, "name": "Paul", ↩
           "books": [ ↩
             "Software Portability with imake: Practical Software Engineering", ↩
             "Mysql: The Definitive Guide to Using, Programming, ↩
             and Administering Mysql 4 (Developer's Library)", ↩
             "MYSQL Certification Study Guide", ↩
             "MySQL (OTHER NEW RIDERS)", ↩
             "MySQL Cookbook", ↩
             "MySQL 5.0 Certification Study Guide", ↩
             "Using csh & tcsh: Type Less, Accomplish More (Nutshell Handbooks)", ↩
             "MySQL (Developer's Library)"], ↩
           "lastname": "DuBois"}
lastname: "DuBois"
*************************** 2. row ***************************
      id: 2
  author: {"id": 2, "name": "Alkin", "books": ["MySQL Cookbook"],↩
           "lastname": "Tezuysal"}
lastname: "Tezuysal"
*************************** 3. row ***************************
      id: 3
  author: {"id": 3, "name": "Sveta", ↩
           "books": ["MySQL Troubleshooting", "MySQL Cookbook"], ↩
           "lastname": "Smirnova"}
lastname: "Smirnova"
3 rows in set (0,00 sec)
```

如果要搜索特定作者，可以考虑按其名和姓进行搜索。

`CREATE INDEX`命令在表的列上创建索引。JSON 数据存储在单列中，因此使用简单的`CREATE INDEX`命令创建的任何索引都将索引整个 JSON 文档，而您可能只需要搜索其中的一部分。

此外，`CREATE INDEX`命令将无法用于 JSON 列。

```
mysql> `CREATE` `INDEX` `author_name` `ON` `book_authors``(``author``)``;`
ERROR 3152 (42000): JSON column 'author' supports indexing only ↩
via generated columns on a specified JSON path.
```

解决此问题的方法是使用生成列并在其上创建索引。生成列的值是在列创建时定义的表达式生成的。

```
ALTER TABLE book_authors ADD COLUMN lastname VARCHAR(255) 
GENERATED ALWAYS AS(JSON_UNQUOTE(JSON_EXTRACT(author, '$.lastname')));
```

在此示例中，我们创建了一个从表达式`JSON_EXTRACT(author, '$.lastname')`生成的列。我们还可以使用运算符`->`和`->>`来提取 JSON 值：

```
ALTER TABLE book_authors ADD COLUMN name VARCHAR(255) 
GENERATED ALWAYS AS (author->>'$.name');
```

我们在表达式中使用了函数`JSON_UNQUOTE`和运算符`->>`来删除作者姓名末尾的引号（如果存在）。

两个新列`name`和`lastname`不占用空间，并且每次查询访问表时都会生成。

###### 提示

如果要通过增加存储空间和写入时的延迟来提高`SELECT`查询性能，请定义带有关键字`STORED`的生成列。在这种情况下，表达式仅在插入或修改时执行一次，并且物理存储在磁盘上。

现在我们可以在新生成的列上创建索引。

```
CREATE INDEX author_name ON book_authors(lastname, name);
```

使用新创建的索引访问数据时，请将新列视为其他任何列。

```
mysql> `SELECT` `author``-``>``'$.books'` `FROM` `book_authors` 
    -> `WHERE` `name` `=` `'Sveta'` `AND` `lastname``=``'Smirnova'``;`
+---------------------------------------------+ | author->'$.books'                           |
+---------------------------------------------+ | ["MySQL Troubleshooting", "MySQL Cookbook"] |
+---------------------------------------------+ 1 row in set (0,00 sec)
```

`EXPLAIN`确认新索引的使用情况。

```
mysql> `EXPLAIN` `SELECT` `author``-``>``'$.books'` `FROM` `book_authors` 
    -> `WHERE` `name` `=` `'Sveta'` `AND` `lastname``=``'Smirnova'``\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: book_authors
   partitions: NULL
         type: ref
possible_keys: author_name
          key: author_name
      key_len: 2046
          ref: const,const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0,00 sec)
```

## 另请参阅

关于在 MySQL 中使用 JSON 的更多信息，请参见第十九章。

# 21.9 使用全文索引

## 问题

您希望利用关键词搜索，但文本字段的查询速度较慢。

## 解决方案

使用`FULLTEXT`索引进行全文搜索。

## 讨论

MySQL 支持`FULLTEXT`索引，但仅限于流行的存储引擎（如 InnoDB 和 MyISAM）。尽管这两种存储引擎原本不设计用于索引大文本操作，但仍可用于特定查询的性能优化。

`FULLTEXT`索引还有另外两个条件。

1.  它只能用于`CHAR`、`VARCHAR`或`TEXT`列。

1.  只能在`SELECT`语句中的`MATCH()`或`AGAINST()`子句中使用。

在 MySQL 中，`MATCH()`函数通过接受一个逗号分隔的列名列表来执行全文搜索，而`AGAINST()`则接受一个要搜索的字符串。

###### 注意

`FULLTEXT`索引可以与同一列上的 B-tree 索引结合使用，因为它们的目的不同。`FULLTEXT`用于查找关键词，而不是匹配字段中的值。

`FULLTEXT`文本搜索也有三种不同的模式：

+   自然语言模式（默认）是用于简单短语的搜索模式。

    ```
    SELECT top_name,name_rank FROM top_names WHERE MATCH(top_name) 
          AGAINST("ANDREW" IN NATURAL LANGUAGE MODE) \G
    ```

+   `布尔模式`用于在搜索模式中使用布尔运算符。回想一下，与 Recipe 7.17 中讨论的策略类似，在这里也使用了操作符。

    ```
    SELECT top_name,name_rank FROM top_names WHERE MATCH(top_name) 
          AGAINST("+ANDREW +ANDY -ANN" IN BOOLEAN MODE) \G
    ```

+   `Query expansion mode` 是用于搜索与搜索表达式相似或相关值的搜索模式。简而言之，这种模式将返回与搜索关键词相关的匹配项。

    ```
    SELECT top_name,name_rank FROM top_names WHERE MATCH(top_name) 
          AGAINST("ANDY"  WITH QUERY EXPANSION) \G
    ```

InnoDB 存储引擎可以利用以下优化。

+   只返回搜索排名的 `ID` 字段的查询。搜索排名定义为关联性排名，用作显示匹配质量的度量。

+   对匹配行进行降序排序的查询

优化器将选择在 `top_name` 列上选择非全文索引。这是因为查询被编写为优化 InnoDB 的 B-Tree 索引，使用模式匹配而不是文字比较。有关更多信息，请参见 Recipe 7.10。在这个示例中，这种类型的查询非常高效，因为我们的数据类型具有在 `top_name` 列中索引的唯一字符串值。

```
mysql> `EXPLAIN` `SELECT` `top_name``,``name_rank` `FROM` `top_names`
    -> `WHERE` `top_name``=``"ANDREW"` `\``G`
    *************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: ref
possible_keys: idx_top_name,idx_desc_01,idx_desc_02
          key: idx_top_name
      key_len: 103
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

```
mysql> `CREATE` `FULLTEXT` `INDEX` `idx_fulltext` `ON` `top_names``(``top_name``)``;`
    Query OK, 0 rows affected, 1 warning (1.94 sec)
   Records: 0  Duplicates: 0  Warnings: 1
```

```
mysql> `EXPLAIN` `SELECT` `top_name``,``name_rank` `FROM` `top_names`
    -> `WHERE` `top_name``=``"ANDREW"` `\``G`
    *************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: ref
possible_keys: idx_top_name,idx_desc_01,idx_desc_02,idx_fulltext
          key: idx_top_name
      key_len: 103
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

现在，如果我们尝试对同一列进行模式匹配，我们将能够利用给定列的全文索引。

```
mysql> `EXPLAIN` `SELECT` `top_name``,``name_rank` `FROM` `top_names`
    -> `MATCH``(``top_name``)` `AGAINST``(``"ANDREW"``)` `\``G`
    *************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: fulltext
possible_keys: idx_fulltext
          key: idx_fulltext
      key_len: 0
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using where; Ft_hints: sorted
1 row in set, 1 warning (0.01 sec)
```

在这种情况下，我们可以看到 MySQL 选择使用`FULLTEXT`索引。尽管在 MySQL 中拥有`FULLTEXT`索引是有用的，但它带来了许多限制。请参考 MySQL 文档了解[全文检索的限制](https://dev.mysql.com/doc/refman/8.0/en/fulltext-restrictions.html)的详细信息。

###### 注意

尽管 InnoDB 存储引擎中有全文索引，但在市场上可能有更好的选择来减轻 MySQL 的工作负荷，并将其放在另一个优化的存储系统中。

# 21.10 利用空间索引和地理数据

## 问题

您希望有效地存储和查询地理坐标。

## 解决方案

使用 MySQL 的改进空间参考系统。

## 讨论

MySQL 8 包含来自 EPSG（欧洲石油调查集团）机构的所有 SRS（空间参考系统）标识。这些 SRS 标识以唯一名称和 SRID 存储在 `information_schema` 中。

这些系统代表了地理数据参考的不同变体。您可以从 `information_schema` 查询这些的详细信息。

```
mysql> `SELECT` `*` `FROM` `INFORMATION_SCHEMA``.``ST_SPATIAL_REFERENCE_SYSTEMS`
    ->  `WHERE` `SRS_ID``=``4326` `OR` `SRS_ID``=``3857` `ORDER` `BY` `SRS_ID` `DESC``\``G`
    *************************** 1. row ***************************
                SRS_NAME: WGS 84
                  SRS_ID: 4326
            ORGANIZATION: EPSG
ORGANIZATION_COORDSYS_ID: 4326
              DEFINITION: GEOGCS["WGS 84",DATUM["World Geodetic System 1984",
              SPHEROID["WGS 84",6378137,298.257223563,AUTHORITY["EPSG","7030"]],
              AUTHORITY["EPSG","6326"]],PRIMEM["Greenwich",0,AUTHORITY["EPSG","8901"]],
              UNIT["degree",0.017453292519943278,AUTHORITY["EPSG","9122"]],
              AXIS["Lat",NORTH],AXIS["Lon",EAST],AUTHORITY["EPSG","4326"]]
             DESCRIPTION: NULL
*************************** 2. row ***************************
                SRS_NAME: WGS 84 / Pseudo-Mercator
                  SRS_ID: 3857
            ORGANIZATION: EPSG
ORGANIZATION_COORDSYS_ID: 3857
              DEFINITION: PROJCS["WGS 84 / Pseudo-Mercator",GEOGCS["WGS 84",
              DATUM["World Geodetic System 1984",SPHEROID["WGS 84",6378137,
              298.257223563,AUTHORITY["EPSG","7030"]],AUTHORITY["EPSG","6326"]],
              PRIMEM["Greenwich",0,AUTHORITY["EPSG","8901"]],UNIT["degree",
              0.017453292519943278,AUTHORITY["EPSG","9122"]],AXIS["Lat",NORTH]
              ,AXIS["Lon",EAST],AUTHORITY["EPSG","4326"]],PROJECTION["Popular 
              Visualisation Pseudo Mercator",AUTHORITY["EPSG","1024"]],
              PARAMETER["Latitude of natural origin",0,AUTHORITY["EPSG","8801"]],
              PARAMETER["Longitude of natural origin",0,AUTHORITY["EPSG","8802"]],
              PARAMETER["False easting",0,AUTHORITY["EPSG","8806"]],↩
              PARAMETER["False northing",
              0,AUTHORITY["EPSG","8807"]],UNIT["metre",1,AUTHORITY["EPSG","9001"]],↩
              AXIS["X",EAST],
              AXIS["Y",NORTH],AUTHORITY["EPSG","3857"]]
             DESCRIPTION: NULL
2 rows in set (0.00 sec)
```

SRS_ID 4326 表示广泛使用的网络地图投影，如谷歌地图、OpenStreetMaps 等，而 4326 则是用于跟踪位置的 GPS 坐标。

假设我们有兴趣点数据存储在我们的数据库中。我们将创建一张表，并使用 SRID 4326 加载样本数据到其中。

```
mysql> `CREATE` `TABLE` `poi` 
    -> `(` `poi_id`  `INT` `UNSIGNED` `AUTO_INCREMENT` `NOT` `NULL`  `PRIMARY` `KEY``,` 
    -> `position` `POINT` `NOT` `NULL` `SRID` `4326``,` `name` `VARCHAR``(``200``)``)``;`
Query OK, 0 rows affected (0.02 sec)

mysql> `INSERT` `INTO` `poi` `VALUES` `(``1``,` `ST_GeomFromText``(``'POINT(41.0211 29.0041)'``,` `4326``)``,`
    -> `'Maiden\'``s` `Tower``');` Query OK, 1 row affected (0.00 sec)
msyql> `INSERT INTO poi VALUES (2, ST_GeomFromText('``POINT``(``41``.``0256` `28``.``9742``)``', 4326),` -> `'``Galata` `Tower``'``)``;`
Query OK, 1 row affected (0.00 sec)
```

现在在几何列上创建索引。

```
mysql> `CREATE` `SPATIAL` `INDEX` `position` `ON` `poi` `(``position``)``;`
Query OK, 0 rows affected (0.01 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

我们将演示如何测量这两个兴趣点之间的距离。

```
mysql> `SELECT` `ST_AsText``(``position``)` `FROM` `poi` `WHERE` `poi_id` `=` `1` `INTO` `@``tower1``;`
Query OK, 1 row affected (0.00 sec)
mysql> `SELECT` `ST_AsText``(``position``)` `FROM` `poi` `WHERE` `poi_id` `=` `2` `INTO` `@``tower2``;`
Query OK, 1 row affected (0.00 sec)
mysql> `SELECT` `ST_Distance``(``ST_GeomFromText``(``@``tower1``,` `4326``)``,`
    -> `ST_GeomFromText``(``@``tower2``,` `4326``)``)` `AS` `distance``;`
+--------------------+ | distance           |
+--------------------+ | 2563.9276036976544 |
+--------------------+ 1 row in set (0.00 sec)
```

这是两个兴趣点之间的直线表示。当然，这不是汽车路径规划的示例；这更像是从点 A 到点 B 的鸟瞰飞行距离（以米为单位）。

让我们检查 MySQL 在查询优化中使用了什么。

```
mysql> `EXPLAIN` `SELECT` `ST_Distance``(``ST_GeomFromText``(``@``tower1``,` `4326``)``,` 
-> `ST_GeomFromText``(``@``tower2``,` `4326``)``)` `AS` `dist` `\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: NULL
   partitions: NULL
         type: NULL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
     filtered: NULL
        Extra: No tables used
1 row in set, 1 warning (0.00 sec)
```

由于 `ST_Distance` 函数不使用表来计算两个位置之间的距离，它在查询中不使用表；因此不允许进行索引优化。

您可以进一步改进关于地球球形的距离计算，应使用`ST_Distance_Sphere`，这将导致略有不同的结果。

```
mysql> `SELECT` `ST_Distance_Sphere``(``ST_GeomFromText``(``@``tower1``,` `4326``)``,` 
    -> `ST_GeomFromText``(``@``tower2``,` `4326``)``)` `AS` `dist``;`
+--------------------+ | dist               |
+--------------------+ | 2557.7412439442496 |
+--------------------+ 1 row in set (0.00 sec)
```

假设我们有一个围绕伊斯坦布尔的多边形，用于覆盖我们的目标搜索区域。可以通过另一个应用程序生成所需的多边形坐标。

```
mysql> `SET` `@``poly` `:``=` `ST_GeomFromText` `(` `'POLYGON(( 41.104897239651905 28.876082545638166,` -> `41.05727989444261 29.183699733138166,` -> `40.90384226781947 29.137007838606916,` -> `40.94119778455447 28.865096217513166,` -> `41.104897239651905 28.876082545638166))'``,` `4326``)``;`
```

这次我们将使用`ST_Within`函数从该多边形区域搜索兴趣点。MySQL 的空间引用实现内置了许多函数。详情请参阅 MySQL 文档[空间分析函数](https://dev.mysql.com/doc/refman/8.0/en/spatial-analysis-functions.html)。空间函数可以分为几类，例如：

+   以各种格式创建几何。

+   在各种格式之间转换几何。

+   访问几何的定性和定量属性。

+   描述两个几何之间的关系。

+   从现有几何创建新几何。

这些函数允许开发人员更快地访问数据，并更好地利用 MySQL 中的空间分析。

在下面的查询中，我们同时使用`ST_AsText`和`ST_Within`函数。

```
mysql> `SELECT`  `poi_id``,` `name``,` `ST_AsText``(``` ` ```position``` ` ```)` 
    -> `AS` `` ` ```towers``` ` `` `FROM` `poi` `WHERE` `ST_Within``(` `` ` ```position``` ` ```,` `@``poly``)` `;`

+--------+----------------+------------------------+ | poi_id | 名称           | 塔                   |

+--------+----------------+------------------------+ |      1 | 少女塔          | POINT(41.0211 29.0041) |

|      2 | 加拉塔塔          | POINT(41.0256 28.9742) |
| --- | --- | --- |

+--------+----------------+------------------------+ 2 行集（0.00 秒）

```

Check whether the spatial index used or not?

```

mysql> `EXPLAIN`  `SELECT`  `poi_id``,` `name``,`

    -> `ST_AsText``(``` ` ```position``` ` ```)` `AS` `` ` ```towers``` ` `` `FROM` `poi` `WHERE` `ST_Within``(` `` ` ```position``` ` ```,` `@``poly``)` `\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: poi
   partitions: NULL
         type: range
possible_keys: position
          key: position
      key_len: 34
          ref: NULL
         rows: 2
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

# 21.11 创建和使用直方图

## 问题

您想要连接两个或更多表，但 MySQL 的优化器未选择正确的查询计划。

## 解决方案

使用优化器直方图来辅助决策。

## 讨论

索引有助于解决查询计划，但它们并不总是用于创建最佳的查询执行计划。当优化器需要确定两个或更多表的连接顺序时，情况就会如此。

假设您有两个表。一个存储商店中的产品类别，另一个存储销售数据。类别数量少，而已售出的物品数量巨大。您可能有数十个类别和数百万个已售出的物品。当您连接两个表时，MySQL 必须决定首先查询哪个表。如果它首先查询小表，它必须检索所选类别中的所有物品，然后过滤它们。如果所选类别中的物品数量巨大，而过滤后的物品数量较少，则查询将不会有效。另一方面，如果您需要来自较大表的范围内的单个类别的物品，并且所选范围返回所有类别的许多行，则您将不得不丢弃除单个类别之外的所有行。

解决此问题的一个方法是在较大表中使用将类别 ID 和`WHERE`条件及`JOIN`子句组合的联合索引。但是，当此联合索引不适用于`WHERE`条件和`JOIN`子句的组合时，此解决方案可能无效。

另一个索引的问题在于它们根据基数（唯一值的数量）进行操作。但当数据分布不均匀时，优化器仅使用基数可能会得出错误的结论。假设您有一百万个具有某种特征的项目和十个具有另一种特征的项目。如果优化器决定选择满足第一个条件的数据，则与首先选择满足第二个条件的数据相比，查询将需要更多的时间。不幸的是，仅通过基数的信息无法得出正确的结论。

为了解决这个问题，MySQL 8.0 引入了优化器直方图。它们是一种轻量级数据结构，用于存储每个数据桶中存在多少个唯一值的信息。

为了说明优化器直方图的工作原理，让我们考虑一个包含六行的表。

```
mysql> `CREATE` `TABLE` `histograms``(``f1` `INT``)``;`
Query OK, 0 rows affected (0,03 sec)

mysql> `INSERT` `INTO` `histograms` `VALUES``(``1``)``,``(``2``)``,``(``2``)``,``(``3``)``,``(``3``)``,``(``3``)``;`
Query OK, 6 rows affected (0,00 sec)
Records: 6  Duplicates: 0  Warnings: 0

mysql> `SELECT` `f1``,` `COUNT``(``f1``)` `FROM` `histograms` `GROUP` `BY` `f1``;`
+------+-----------+ | f1   | COUNT(f1) |
+------+-----------+ |    1 |         1 |
|    2 |         2 |
|    3 |         3 |
+------+-----------+ 3 rows in set (0,00 sec)
```

如您所见，该表包含一个值为 1 的行，两个值为 2 的行和三个值为 3 的行。

如果我们在查询中选择此表中的不同行并运行`EXPLAIN`，我们将注意到从结果中过滤的行数是相同的，无论我们寻找哪个值。

```
mysql> `\``P` `grep` `filtered`
PAGER set to 'grep filtered'
mysql> `EXPLAIN` `SELECT` `*` `FROM` `histograms` `WHERE` `f1``=``1``\``G`
     filtered: 16.67
1 row in set, 1 warning (0,00 sec)

mysql> `EXPLAIN` `SELECT` `*` `FROM` `histograms` `WHERE` `f1``=``2``\``G`
     filtered: 16.67
1 row in set, 1 warning (0,00 sec)

mysql> `EXPLAIN` `SELECT` `*` `FROM` `histograms` `WHERE` `f1``=``3``\``G`
     filtered: 16.67
1 row in set, 1 warning (0,00 sec)
```

过滤的行数显示从检索结果中将被过滤的行数。由于我们的表没有索引，MySQL 首先从表中检索所有行，然后过滤满足条件的行。在没有任何提示的情况下，优化器认为 MySQL 将从结果中留下只一行，无论使用哪个条件。

创建一个直方图，检查是否有任何变化。

```
mysql> `ANALYZE` `TABLE` `histograms` `UPDATE` `HISTOGRAM` `ON` `f1``\``G`
*************************** 1. row ***************************
   Table: cookbook.histograms
      Op: histogram
Msg_type: status
Msg_text: Histogram statistics created for column 'f1'.
1 row in set (0,01 sec)
```

直方图存储在数据字典表`column_statistics`中，并且可以通过查询信息模式中的`COLUMN_STATISTICS`表来检查。

```
mysql> `SELECT` `*` `FROM` `information_schema``.``column_statistics`
    -> `WHERE` `table_name``=``'histograms'``\``G`
*************************** 1. row ***************************
SCHEMA_NAME: cookbook
 TABLE_NAME: histograms
COLUMN_NAME: f1
  HISTOGRAM: {"buckets": [[1, 0.16666666666666666], [2, 0.5], [3, 1.0]], 
              "data-type": "int", 
              "null-values": 0.0, 
              "collation-id": 8, 
              "last-updated": "2021-05-23 17:29:46.595599", 
              "sampling-rate": 1.0, 
              "histogram-type": "singleton", 
              "number-of-buckets-specified": 100}
1 row in set (0,00 sec)
```

三个桶包含关于数据范围的信息。值 1 占表的 1/6（六分之一）（六行中的一行），值 1 和 2 各占一半（0.5）的表，并与值 3 一起填充表。每个桶中的项目数量存储为一的分数。字段`number-of-buckets-specified`包含在创建直方图时指定的桶数。默认值为 100，但您可以自由地指定介于 1 和 1024 之间的任何数字。如果列中唯一元素的数量超过桶的数量，则`histogram-type`将从`singleton`更改为`equi-height`，并且每个桶可以包含值范围，而不是仅在`singleton`情况下包含一个值。

直方图会影响`EXPLAIN`输出中`filtered`字段的值。

在下面的示例中，过滤行的值是正确的，并反映了表的内容。如果我们搜索值 1，则预计将从结果集中删除六个表行中的五个，这是正确的。对于值 2，只有两行（33.33%）将留在结果集中，而对于值 3，表的一半将被过滤。

```
mysql> `\``P` `grep` `filtered`
PAGER set to 'grep filtered'
mysql> `EXPLAIN` `SELECT` `*` `FROM` `histograms` `WHERE` `f1``=``1``\``G`
     filtered: 16.67
1 row in set, 1 warning (0,00 sec)

mysql> `EXPLAIN` `SELECT` `*` `FROM` `histograms` `WHERE` `f1``=``2``\``G`
     filtered: 33.33
1 row in set, 1 warning (0,00 sec)

mysql> `EXPLAIN` `SELECT` `*` `FROM` `histograms` `WHERE` `f1``=``3``\``G`
     filtered: 50.00
1 row in set, 1 warning (0,00 sec)
```

直方图不会帮助访问数据：它们只是统计信息，不是像索引那样的物理结构。它们影响查询执行计划，特别是表连接的顺序。例如，如果我们决定将 `histograms` 表与自身连接，根据条件顺序会有所不同。

```
mysql> `\``P` `grep` `-``B` `3` `table`
PAGER set to 'grep -B 3 table'
mysql> `EXPLAIN` `SELECT` `*` `FROM` `histograms` `h1` `JOIN` `histograms` `h2`
    -> `WHERE` `h1``.``f1``=``1` `and` `h2``.``f1``=``3``\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: h1
-- *************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: h2
2 rows in set, 1 warning (0,00 sec)

mysql> `EXPLAIN` `SELECT` `*` `FROM` `histograms` `h1` `JOIN` `histograms` `h2`
    -> `WHERE` `h1``.``f1``=``3` `and` `h2``.``f1``=``1``\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: h2
-- *************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: h1
2 rows in set, 1 warning (0,00 sec)
```

直方图的真正威力在于大表上得到了展示。相关的 GitHub 仓库有两个表的数据：`goods_shops` 和 `goods_characteristics`。它们默认情况下没有使用直方图，但有索引。

```
CREATE TABLE `goods_shops` (
  `id` int NOT NULL AUTO_INCREMENT,
  `good_id` varchar(30) DEFAULT NULL,
  `location` varchar(30) DEFAULT NULL,
  `delivery_options` varchar(30) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `good_id` (`good_id`,`location`,`delivery_options`),
  KEY `location` (`location`,`delivery_options`)
);

CREATE TABLE `goods_characteristics` (
  `id` int NOT NULL AUTO_INCREMENT,
  `good_id` varchar(30) DEFAULT NULL,
  `size` int DEFAULT NULL,
  `manufacturer` varchar(30) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `good_id` (`good_id`,`size`,`manufacturer`),
  KEY `size` (`size`,`manufacturer`)
);
```

如果我们想找到屏幕尺寸小于 13 英寸且制造商是联想、戴尔、东芝、三星或宏碁中的一家的笔记本电脑的数量，并且可以通过高级或紧急交付在莫斯科或基辅获取，我们可以使用以下查询。

```
mysql> `SELECT` `COUNT``(``*``)` `FROM` `goods_shops` 
    -> `JOIN` `goods_characteristics` `USING` `(``good_id``)` 
    -> `WHERE` `size` `<` `13` `AND` `manufacturer` 
    -> `IN` `(``'Lenovo'``,` `'Dell'``,` `'Toshiba'``,` `'Samsung'``,` `'Acer'``)` 
    -> `AND` `(``location` `IN` `(``'Moscow'``,` `'Kiev'``)` 
    -> `OR` `delivery_options` `IN` `(``'Premium'``,` `'Urgent'``)``)``;`
+----------+ | count(*) |
+----------+ |   816640 |
+----------+ 1 row in set (6 min 31,75 sec)
```

查询耗时超过 6 分钟，对于少于 50 万行的两个表来说，这是相当长的时间。原因在于 `goods_shops` 表只包含少数满足商店位置和交付选项条件的行，而 `goods_characteristics` 表中有更多满足笔记本电脑尺寸和制造商条件的行。在这种情况下，最好先从 `goods_shops` 表中选择数据，而优化器则选择了相反的方法。

```
mysql> `EXPLAIN` `SELECT` `COUNT``(``*``)` `FROM` `goods_shops` `JOIN` `goods_characteristics`
    -> `USING``(``good_id``)` `WHERE` `size` `<` `13` `AND` 
    -> `manufacturer` `IN` `(``'Lenovo'``,` `'Dell'``,` `'Toshiba'``,` `'Samsung'``,` `'Acer'``)` `AND`
    -> `(``location` `IN` `(``'Moscow'``,` `'Kiev'``)` `OR` `delivery_options` `IN` `(``'Premium'``,` `'Urgent'``)``)``\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: goods_characteristics
   partitions: NULL
         type: index
possible_keys: good_id,size
          key: good_id
      key_len: 251
          ref: NULL
         rows: 137026
     filtered: 25.00
        Extra: Using where; Using index
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: goods_shops
   partitions: NULL
         type: ref
possible_keys: good_id,location
          key: good_id
      key_len: 123
          ref: cookbook.goods_characteristics.good_id
         rows: 66422
     filtered: 36.00
        Extra: Using where; Using index
2 rows in set, 1 warning (0,00 sec)
```

索引在这里不会起作用，因为它们使用的基数对于索引列中的任何值都是相同的。这就是直方图可以展示其威力的时候。

```
mysql> `ANALYZE` `TABLE` `goods_shops` `UPDATE` `HISTOGRAM` `ON` `location``,` `delivery_options``\``G`
*************************** 1. row ***************************
   Table: cookbook.goods_shops
      Op: histogram
Msg_type: status
Msg_text: Histogram statistics created for column 'delivery_options'.
*************************** 2. row ***************************
   Table: cookbook.goods_shops
      Op: histogram
Msg_type: status
Msg_text: Histogram statistics created for column 'location'.
2 rows in set (0,24 sec)

mysql> `SELECT` `COUNT``(``*``)` `FROM` `goods_shops` `JOIN` `goods_characteristics`
    -> `USING``(``good_id``)` `WHERE` `size` `<` `13` `AND` 
    -> `manufacturer` `IN` `(``'Lenovo'``,` `'Dell'``,` `'Toshiba'``,` `'Samsung'``,` `'Acer'``)` `AND` 
    -> `(``location` `IN` `(``'Moscow'``,` `'Kiev'``)` `OR` `delivery_options` `IN` `(``'Premium'``,` `'Urgent'``)``)``;`
+----------+ | COUNT(*) |
+----------+ |   816640 |
+----------+ 1 row in set (1,42 sec)

mysql> `EXPLAIN` `SELECT` `COUNT``(``*``)` `FROM` `goods_shops` `JOIN` `goods_characteristics`
    -> `USING``(``good_id``)` `WHERE` `size` `<` `13` `AND` 
    -> `manufacturer` `IN` `(``'Lenovo'``,` `'Dell'``,` `'Toshiba'``,` `'Samsung'``,` `'Acer'``)` `AND`
    -> `(``location` `IN` `(``'Moscow'``,` `'Kiev'``)` `OR` `delivery_options` `IN` `(``'Premium'``,` `'Urgent'``)``)``\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: goods_shops
   partitions: NULL
         type: index
possible_keys: good_id,location
          key: good_id
      key_len: 369
          ref: NULL
         rows: 66422
     filtered: 0.09
        Extra: Using where; Using index
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: goods_characteristics
   partitions: NULL
         type: ref
possible_keys: good_id,size
          key: good_id
      key_len: 123
          ref: cookbook.goods_shops.good_id
         rows: 137026
     filtered: 25.00
        Extra: Using where; Using index
2 rows in set, 1 warning (0,00 sec)
```

一旦创建了直方图，优化器按照有效的顺序连接表，并且查询时间略长于之前的运行中的六秒钟。

## 参见

有关在 MySQL 中使用直方图的更多信息，请参阅[Billion Goods in Few Categories: How Histograms Save a Life?](https://www.percona.com/resources/webinars/billion-goods-few-categories-how-histograms-save-life)。

# 21.12 编写高性能查询

## 问题

您希望编写高效的查询。

## 解决方案

研究 MySQL 如何访问数据，并调整您的查询以帮助 MySQL 更快地完成其工作。

## 讨论

正如我们在本章中所看到的，MySQL 中索引实现有许多迭代。虽然我们利用了这些索引类型，但我们也需要知道 MySQL 如何访问数据。优化器是 MySQL 中非常先进的部分，但仍然不能总是做出正确的决策。当它没有选择正确的路径时，我们将面临查询性能不佳的问题，这可能导致生产中的服务降级或中断。编写高性能查询的最佳方式是了解 MySQL 如何访问数据。

另一个要点是，在规模上的不同与在单体环境中使用应用程序不同。随着并发性增加和数据规模，决策优化器将选择更复杂但更快的数据路由。

MySQL 使用基于成本的模型来估计查询执行过程中各种操作的成本，顺序如下。

1.  找到最佳方法。

1.  检查访问方法是否有用。

1.  估算使用访问方法的成本。

1.  选择可能的最低成本访问方法。

以下是 MySQL 选择的查询执行顺序：

1.  表扫描

1.  索引扫描

1.  索引查找

1.  范围扫描

1.  索引合并

1.  松散索引扫描

以下是一些已知的使用索引但性能差的慢索引查找的原因。

低基数性

当数据不足以识别快速遍历时，MySQL 最终会执行全表扫描。

大数据集

返回大数据集通常会导致问题。即使它们经过正确过滤，由于应用程序无法快速处理它们，它们可能是无用的。只针对查询中需要的数据，并过滤其余数据。

多索引遍历

如果查询命中多个索引，通过页面进行额外的 I/O 操作会导致查询性能变慢。

非主导列查找

如果不使用主导列来创建覆盖索引，则无法使用覆盖索引。

数据类型不匹配

当查询列的数据类型不匹配时，索引无法帮助。

字符集/校对规则不匹配

数据访问应统一围绕查询的字符集和校对规则进行。

后缀查找

查找后缀会显著降低性能。

索引作为参数

使用索引列作为参数将不能有效使用该索引。

过期统计信息

MySQL 根据索引基数更新统计信息。这有助于优化器做出尽可能快的路径决策。

MySQL 错误

这种情况很少见，但可能存在 MySQL 的错误，可能导致索引查找变慢。

### 查询类型

在设计应用程序时，识别常见的查询模式及其适用性和非适用性是有用的。

点选

访问数据的最快方法之一是直接针对索引列进行点选，这种情况下，如果索引存在于该列中，优化器已经知道数据所在的页面。

```
mysql> `SELECT names_id,top_name,name_rank FROM top_names` 
    -> `WHERE names_id=699 \G`
*************************** 1\. row ***************************
 names_id: 699
 top_name: KOCH
name_rank: 698
1 row in set (0.00 sec)
```

在这种情况下，`names_id`列是表的主键列，因此优化器直接访问该页面路径。

范围选择

当需要从数据集中获取一系列行时，就会使用这种`SELECT`。MySQL 仍然可以使用索引直接访问数据，使用与查询的`WHERE`子句中相同列的索引。这种访问方法使用单个索引或来自索引的值子集。范围索引也被称为使用单个或多个部分索引。在以下示例中，优化器使用`name_rank`字段的`<`和`>`运算符进行比较。对于所有索引类型，`AND`或`OR`组合都将成为范围条件。对于 MySQL 来说，最快的查找是主键。记住这也是表的物理顺序。

```
mysql> `EXPLAIN SELECT names_id,top_name,name_rank FROM top_names` 
    -> `WHERE names_id>800 AND names_id < 900 \G`
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: range
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 99
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

覆盖索引

覆盖索引是可以用于解析查询而无需访问行数据的索引。为了确保其他支持索引覆盖您的查询，我们有时应该使用次要索引。索引应该从最左边的字段开始，并且每个附加字段在复合键中。查询不应访问索引中不存在的列（参见[Recipe 21.5](https://wiki.example.org/nch-queryperf-queryperf-index-mulitple)）。

下面的示例中，索引用于解析查询条件而不访问表数据，但最终仍访问了表数据，因为我们要求的`top_name`列在索引中不存在。在`EXPLAIN`输出的`Extra`字段中，语句`Using index condition`确认了这一点。

```
mysql> `EXPLAIN SELECT name_rank,top_name,name_count`
 -> `FROM top_names WHERE name_rank < 10 ORDER BY name_count\G`
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: range
possible_keys: idx_name_rank_count
          key: idx_name_rank_count
      key_len: 3
          ref: NULL
         rows: 10
     filtered: 100.00
        Extra: Using index condition; Using filesort
1 row in set, 1 warning (0.00 sec)
```

此查询使用了覆盖索引。语句`Using index`确认了这一点。主键已经是覆盖索引的一部分，因此不需要将`names_id`包含在覆盖索引中。

```
mysql> `EXPLAIN` `SELECT` `names_id``,` `name_rank``,` `name_count` `FROM` `top_names` 
    -> `WHERE` `name_rank` `<` `10` `ORDER` `BY` `name_count``\``G`
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: range
possible_keys: idx_name_rank_count
          key: idx_name_rank_count
      key_len: 3
          ref: NULL
         rows: 10
     filtered: 100.00
        Extra: Using where; Using index; Using filesort
1 row in set, 1 warning (0,00 sec)
```

数据类型匹配

数据类型是有效使用索引的另一个关键因素。对于优化器而言，使用数字进行数值比较至关重要。以下查询是一个不好的例子，展示了当涉及到具有字符串的`names_id`字段的数据类型转换时，MySQL 并不喜欢这种情况。请看下面我们得到的警告消息。

```
mysql> `EXPLAIN SELECT names_id,top_name,name_rank FROM` 
    -> `top_names WHERE names_id= '123 names' \G`
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0,00 sec)

mysql> `SHOW WARNINGS\G`
*************************** 1\. row ***************************
  Level: Warning
   Code: 1292
Message: Truncated incorrect DOUBLE value: '123 names'
*************************** 2\. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select '123' AS `names_id`,'WALLACE' AS `top_name`, ↩
         '123' AS `name_rank` from `cookbook`.`top_names` where true
2 rows in set (0,00 sec)
```

当查询返回结果时，MySQL 必须执行任务将字符串转换为数字并丢失精度。

负条件

在这些类型的查询中，通常最有效的索引无法使用。这是因为 MySQL 必须从表或索引中选择所有行，然后过滤那些不在列表中的行。

如果可能的话，尽量避免使用否定子句，因为它们效率低下：

+   `IS NOT`

+   `IS NOT NULL`

+   `NOT IN`

+   `NOT LIKE`

```
mysql> `EXPLAIN SELECT names_id,name_rank, top_name FROM` 
    -> `top_names WHERE top_name NOT IN ("LARA","ILAYDA","ASLIHAN") \G`
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: ALL
possible_keys: idx_top_name,idx_desc_01,idx_desc_02,idx_fulltext
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 161533
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

ORDER BY 操作

排序操作可能会随着数据集的增长变得昂贵，特别是如果查询无法使用索引来解析`ORDER BY`。

```
mysql> `EXPLAIN SELECT names_id,name_rank, top_name FROM` 
    -> `top_names WHERE name_rank > 15000 ORDER BY top_name \G`
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 161533
     filtered: 33.33
        Extra: Using where; Using filesort
1 row in set, 1 warning (0.00 sec)
```

同样适用于`LIMIT`操作。这类查询通常返回少量数据，但成本很高。

```
mysql> `EXPLAIN SELECT names_id,name_rank, top_name FROM top_names WHERE` 
    -> `name_rank > 15000 ORDER BY name_rank LIMIT 10\G`
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: top_names
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 161533
     filtered: 33.33
        Extra: Using where; Using filesort
1 row in set, 1 warning (0.00 sec)
```

上面的查询选择大量行，然后丢弃其中大多数。

连接

连接操作是一种将来自两个或多个表的数据进行组合或引用的原始方法。虽然 SQL 连接服务于特定目的，但如果不正确使用，它们可能会在查询结果上创建笛卡尔积。强烈建议在 `SELECT` 语句中使用 `INNER` 连接来仅过滤表的交集，而不是使用 `LEFT JOIN`。尽管使用 `INNER JOIN` 并不总能符合所需的业务逻辑。在这些情况下，仍然针对索引字段进行目标查询将有利于查询执行时间。

```
SELECT a.col_a, a.col_b, b.col_a FROM table_a a
INNER JOIN table_b b
ON a.key = b.key;
```

###### 提示

在 MySQL 中，`JOIN` 是 `INNER JOIN` 的同义词。

为了帮助优化器通过创建正确的索引和编写高效的查询来选择访问数据的最佳路径。这种方法将全面提高你的吞吐量。只添加你需要的索引，不要对表进行过度索引。避免重复索引是实现高性能查询的另一种最佳实践。识别你的表中是否存在相同的索引可能会导致读写都变慢。
