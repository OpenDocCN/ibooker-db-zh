# 第五章：从表中选择数据

# 5.0 引言

本章重点介绍使用`SELECT`语句从数据库中检索信息。如果你的 SQL 背景有限或想了解 MySQL 特定的`SELECT`语法扩展，本章对你会很有帮助。

有许多方法可以编写`SELECT`语句；我们只会讨论其中几种。有关`SELECT`语法以及提取和操作数据的函数和运算符的更多信息，请参阅*MySQL 参考手册*或一般的 MySQL 文本。

本章中的许多示例使用了一个名为`mail`的表，该表包含跟踪用户之间邮件消息流量的行。以下显示了如何创建该表：

```
CREATE TABLE mail
(
  id      INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  t       DATETIME,    # when message was sent
  srcuser VARCHAR(8),  # sender (source user and host)
  srchost VARCHAR(20),
  dstuser VARCHAR(8),  # recipient (destination user and host)
  dsthost VARCHAR(20),
  size    BIGINT,      # message size in bytes
  INDEX (t)
);
```

`mail`表的内容如下：

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

要创建并加载`mail`表，请进入`recipes`分发的*tables*目录，并运行以下命令：

```
$ `mysql cookbook < mail.sql`
```

本章有时还会使用其他表。有些表在之前的章节中已经用过，而其他表则是新的。要创建任何表，请像创建`mail`表那样，在*tables*目录中使用适当的脚本。此外，本章中使用的许多其他脚本和程序位于*select*目录中。该目录中的文件使您可以更轻松地尝试示例。

本章中的许多示例可以在*mysql*程序内部执行，该程序在第一章中有所讨论。少数示例涉及从编程语言的上下文中发布语句。有关编程技术的信息，请参阅第四章。

# 5.1 指定要选择的列和行

## 问题

你想要从表中显示特定的列和行。

## 解决方案

要指示显示哪些列，请在输出列列表中命名它们。要指示显示哪些行，请使用一个`WHERE`子句，指定行必须满足的条件。

## 讨论

从表中显示列的最简单方法是使用`SELECT * FROM` *`tbl_name`*。`*`说明符是一个快捷方式，意思是<q>所有列</q>：

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

使用`*`很容易，但你不能选择特定的列或控制列的显示顺序。明确命名列使你能够按任意顺序选择感兴趣的列。这个查询省略了收件人列，并在日期和大小之前显示发件人：

```
mysql> `SELECT srcuser, srchost, t, size FROM mail;`
+---------+---------+---------------------+---------+
| srcuser | srchost | t                   | size    |
+---------+---------+---------------------+---------+
| barb    | saturn  | 2014-05-11 10:15:08 |   58274 |
| tricia  | mars    | 2014-05-12 12:48:13 |  194925 |
| phil    | mars    | 2014-05-12 15:02:49 |    1048 |
| barb    | saturn  | 2014-05-12 18:59:18 |     271 |
…
```

如果不限定或限制`SELECT`查询的方式，它将检索表中的每一行。为了更精确，请提供一个`WHERE`子句，指定行必须满足的一个或多个条件。

条件可以测试相等性、不等式或相对顺序。对于某些类型的数据，如字符串，可以使用模式匹配。以下语句从`mail`表中选择包含`srchost`值完全等于字符串`'venus'`或以字母`s`开头的行的列：

```
mysql> `SELECT t, srcuser, srchost  FROM mail WHERE srchost = 'venus';`
+---------------------+---------+---------+
| t                   | srcuser | srchost |
+---------------------+---------+---------+
| 2014-05-14 09:31:37 | gene    | venus   |
| 2014-05-14 14:42:21 | barb    | venus   |
| 2014-05-15 08:50:57 | phil    | venus   |
| 2014-05-16 09:00:28 | gene    | venus   |
| 2014-05-16 23:04:19 | phil    | venus   |
+---------------------+---------+---------+
mysql> `SELECT t, srcuser, srchost FROM mail WHERE srchost LIKE 's%';`
+---------------------+---------+---------+
| t                   | srcuser | srchost |
+---------------------+---------+---------+
| 2014-05-11 10:15:08 | barb    | saturn  |
| 2014-05-12 18:59:18 | barb    | saturn  |
| 2014-05-14 17:03:01 | tricia  | saturn  |
| 2014-05-15 17:35:31 | gene    | saturn  |
| 2014-05-19 22:21:51 | gene    | saturn  |
+---------------------+---------+---------+
```

前一个查询中的`LIKE`运算符执行模式匹配，其中`%`充当通配符，匹配任何字符串。配方 7.10 进一步讨论了模式匹配。

`WHERE`子句可以测试多个条件，不同的条件可以测试不同的列。下面的语句查找由`barb`发送给`tricia`的消息：

```
mysql> `SELECT t, srcuser, srchost, dstuser, dsthost, size FROM mail`
    -> `WHERE srcuser = 'barb' AND dstuser = 'tricia';`
+---------------------+---------+---------+---------+---------+-------+
| t                   | srcuser | srchost | dstuser | dsthost | size  |
+---------------------+---------+---------+---------+---------+-------+
| 2014-05-11 10:15:08 | barb    | saturn  | tricia  | mars    | 58274 |
| 2014-05-12 18:59:18 | barb    | saturn  | tricia  | venus   |   271 |
+---------------------+---------+---------+---------+---------+-------+
```

输出列可以通过评估表达式来计算。此查询使用`CONCAT()`将`srcuser`和`srchost`列组合成电子邮件地址格式的复合值：

```
mysql> `SELECT t, CONCAT(srcuser,'@',srchost), size FROM mail;`
+---------------------+-----------------------------+---------+
| t                   | CONCAT(srcuser,'@',srchost) | size    |
+---------------------+-----------------------------+---------+
| 2014-05-11 10:15:08 | barb@saturn                 |   58274 |
| 2014-05-12 12:48:13 | tricia@mars                 |  194925 |
| 2014-05-12 15:02:49 | phil@mars                   |    1048 |
| 2014-05-12 18:59:18 | barb@saturn                 |     271 |
…
```

您会注意到电子邮件地址列标签是计算它的表达式。为了提供更好的标签，请使用列别名（参见配方 5.2）。

自 MySQL 8.0.19 起，您可以使用`TABLE`语句从表中选择所有列。`TABLE`支持`ORDER BY`（参见配方 5.3）和`LIMIT`（参见配方 5.11）子句，但不允许对列或行进行任何其他过滤。

```
mysql> `TABLE` `mail` `ORDER` `BY` `size` `DESC` `LIMIT` `3``;`
+----+---------------------+---------+---------+---------+---------+---------+ | id | t                   | srcuser | srchost | dstuser | dsthost | size    |
+----+---------------------+---------+---------+---------+---------+---------+ |  8 | 2014-05-14 17:03:01 | tricia  | saturn  | phil    | venus   | 2394482 |
| 11 | 2014-05-15 10:25:52 | gene    | mars    | tricia  | saturn  |  998532 |
|  2 | 2014-05-12 12:48:13 | tricia  | mars    | gene    | venus   |  194925 |
+----+---------------------+---------+---------+---------+---------+---------+ 3 rows in set (0.00 sec)
```

# 5.2 命名查询结果列

## 问题

查询结果中的列名不合适、难看或难以处理，因此您希望自己命名它们。

## 解决方案

使用别名来选择您自己的列名。

## 讨论

当您检索结果集时，MySQL 为每个输出列指定一个名称。（这就是*mysql*程序获取在结果集输出的列标题初始行中看到的名称的方式。）默认情况下，MySQL 将`CREATE` `TABLE`或`ALTER` `TABLE`语句中指定的列名分配给输出列，但如果这些默认名称不合适，则可以使用列别名指定自己的名称。

本配方解释了别名，并展示了如何在语句中使用它们来分配列名。如果您正在编写一个必须确定名称的程序，请参阅配方 12.2 以获取有关访问列元数据的信息。

如果输出列直接来自表格，MySQL 会使用表列名作为输出列名。以下语句选择四个表列，其名称成为相应的输出列名：

```
mysql> `SELECT t, srcuser, srchost, size FROM mail;`
+---------------------+---------+---------+---------+
| t                   | srcuser | srchost | size    |
+---------------------+---------+---------+---------+
| 2014-05-11 10:15:08 | barb    | saturn  |   58274 |
| 2014-05-12 12:48:13 | tricia  | mars    |  194925 |
| 2014-05-12 15:02:49 | phil    | mars    |    1048 |
| 2014-05-12 18:59:18 | barb    | saturn  |     271 |
…
```

如果通过评估表达式生成列，则表达式本身就是列名。这可能会在结果集中产生长而难以控制的名称，如下面的语句所示，该语句使用一个表达式重新格式化`t`列中的日期，并使用另一个表达式将`srcuser`和`srchost`组合成电子邮件地址格式：

```
mysql> `SELECT`
    -> `DATE_FORMAT(t,'%M %e, %Y'), CONCAT(srcuser,'@',srchost), size`
    -> `FROM mail;`
+----------------------------+-----------------------------+---------+
| DATE_FORMAT(t,'%M %e, %Y') | CONCAT(srcuser,'@',srchost) | size    |
+----------------------------+-----------------------------+---------+
| May 11, 2014               | barb@saturn                 |   58274 |
| May 12, 2014               | tricia@mars                 |  194925 |
| May 12, 2014               | phil@mars                   |    1048 |
| May 12, 2014               | barb@saturn                 |     271 |
…
```

要选择自己的输出列名，使用`AS` *`name`* 子句指定列别名（关键字`AS`是可选的）。下面的语句检索与前一个语句相同的结果，但将第一列重命名为`date_sent`，第二列重命名为`sender`：

```
mysql> `SELECT`
    -> `DATE_FORMAT(t,'%M %e, %Y') AS date_sent,`
    -> `CONCAT(srcuser,'@',srchost) AS sender,`
    -> `size FROM mail;`
+--------------+---------------+---------+
| date_sent    | sender        | size    |
+--------------+---------------+---------+
| May 11, 2014 | barb@saturn   |   58274 |
| May 12, 2014 | tricia@mars   |  194925 |
| May 12, 2014 | phil@mars     |    1048 |
| May 12, 2014 | barb@saturn   |     271 |
…
```

别名使列名称更简洁，更易于阅读，更有意义。别名受到一些限制。例如，如果它们是 SQL 关键字、完全是数字或包含空格或其他特殊字符（如果您想使用描述性短语，别名可以由多个单词组成）。下面的语句检索与前一个语句相同的数据值，但使用短语命名输出列：

```
mysql> `SELECT`
    -> `DATE_FORMAT(t,'%M %e, %Y') AS 'Date of message',`
    -> `CONCAT(srcuser,'@',srchost) AS 'Message sender',`
    -> `size AS 'Number of bytes' FROM mail;`
+-----------------+----------------+-----------------+
| Date of message | Message sender | Number of bytes |
+-----------------+----------------+-----------------+
| May 11, 2014    | barb@saturn    |           58274 |
| May 12, 2014    | tricia@mars    |          194925 |
| May 12, 2014    | phil@mars      |            1048 |
| May 12, 2014    | barb@saturn    |             271 |
…
```

如果 MySQL 对单词别名抱怨，则该单词可能是保留字。引用别名应该使其合法：

```
mysql> `SELECT 1 AS INTEGER;`
You have an error in your SQL syntax near 'INTEGER'
mysql> `SELECT 1 AS 'INTEGER';`
+---------+
| INTEGER |
+---------+
|       1 |
+---------+
```

列别名对编程目的也非常有用。如果您编写一个程序，将行获取到数组中并通过数值列索引访问它们，则列别名的存在与否不会有任何影响，因为别名不会改变结果集中列的位置。然而，如果您通过名称访问输出列，则别名会产生很大的影响，因为别名会改变这些名称。利用这一点，可以为程序提供更容易处理的名称。例如，如果您的查询使用表`mail`中的表达式`DATE_FORMAT(t,'%M %e, %Y')`显示重新格式化的消息时间值，那么该表达式也是在引用输出列时必须使用的名称。例如，在 Perl 的哈希引用中，您可以将其访问为`$ref->{"DATE_FORMAT(t,'%M %e, %Y')}"`。这很不方便。使用`AS date_sent`为列添加别名，您可以更轻松地引用它，例如`$ref->{date_sent}`。下面是一个示例，显示 Perl DBI 脚本如何处理这样的值。它将行检索到哈希中，并通过名称引用列值：

```
$sth = $dbh->prepare ("SELECT srcuser,
 DATE_FORMAT(t,'%M %e, %Y') AS date_sent
 FROM mail");
$sth->execute ();
while (my $ref = $sth->fetchrow_hashref ())
{
  printf "user: %s, date sent: %s\n", $ref->{srcuser}, $ref->{date_sent};
}
```

在 Java 中，您可以这样做，其中`getString()`的参数命名要访问的列：

```
Statement s = conn.createStatement ();
s.executeQuery ("SELECT srcuser,"
                + " DATE_FORMAT(t,'%M %e, %Y') AS date_sent"
                + " FROM mail");
ResultSet rs = s.getResultSet ();
while (rs.next ())  // loop through rows of result set
{
  String name = rs.getString ("srcuser");
  String dateSent = rs.getString ("date_sent");
  System.out.println ("user: " + name + ", date sent: " + dateSent);
}
rs.close ();
s.close ();
```

Recipe 4.4 显示了我们每种编程语言如何将行获取到允许按名称访问列值的数据结构中的示例。`recipes`分发的*select*目录有示例显示如何对`mail`表执行此操作。

您不能在`WHERE`子句中引用列别名。因此，以下语句是非法的：

```
mysql> `SELECT t, srcuser, dstuser, size/1024 AS kilobytes`
    -> `FROM mail WHERE kilobytes > 500;`
ERROR 1054 (42S22): Unknown column 'kilobytes' in 'where clause'
```

错误发生是因为别名命名了一个*输出*列，而`WHERE`子句操作*输入*列来确定要为输出选择哪些行。要使语句合法，将`WHERE`子句中的别名替换为代表该别名的相同列或表达式即可。

```
mysql> `SELECT t, srcuser, dstuser, size/1024 AS kilobytes`
    -> `FROM mail WHERE size/1024 > 500;`
+---------------------+---------+---------+-----------+
| t                   | srcuser | dstuser | kilobytes |
+---------------------+---------+---------+-----------+
| 2014-05-14 17:03:01 | tricia  | phil    | 2338.3613 |
| 2014-05-15 10:25:52 | gene    | tricia  |  975.1289 |
+---------------------+---------+---------+-----------+
```

# 5.3 排序查询结果

## 问题

您希望控制查询结果的排序方式。

## 解决方案

使用`ORDER BY`子句告诉它如何排序结果行。

## 讨论

在选择行时，MySQL 服务器可以按任意顺序返回它们，除非您通过排序结果来指示它。有许多排序技术可以使用，正如第九章详细探讨的那样。简言之，要对结果集排序，请在`ORDER BY`子句中指定要用于排序的列。以下语句在`ORDER BY`子句中命名多个列，按主机和每个主机内的用户排序行：

```
mysql> `SELECT t, srcuser, srchost, dstuser, dsthost, size`
    -> `FROM mail WHERE dstuser = 'tricia'`
    -> `ORDER BY srchost, srcuser;`
+---------------------+---------+---------+---------+---------+--------+
| t                   | srcuser | srchost | dstuser | dsthost | size   |
+---------------------+---------+---------+---------+---------+--------+
| 2014-05-15 10:25:52 | gene    | mars    | tricia  | saturn  | 998532 |
| 2014-05-14 11:52:17 | phil    | mars    | tricia  | saturn  |   5781 |
| 2014-05-19 12:49:23 | phil    | mars    | tricia  | saturn  |    873 |
| 2014-05-11 10:15:08 | barb    | saturn  | tricia  | mars    |  58274 |
| 2014-05-12 18:59:18 | barb    | saturn  | tricia  | venus   |    271 |
+---------------------+---------+---------+---------+---------+--------+
```

MySQL 默认按升序对行进行排序。要按逆序（降序）排序列，请在`ORDER BY`子句中列名后添加关键字`DESC`：

```
mysql> `SELECT t, srcuser, srchost, dstuser, dsthost, size`
    -> `FROM mail WHERE size > 50000 ORDER BY size DESC;`
+---------------------+---------+---------+---------+---------+---------+
| t                   | srcuser | srchost | dstuser | dsthost | size    |
+---------------------+---------+---------+---------+---------+---------+
| 2014-05-14 17:03:01 | tricia  | saturn  | phil    | venus   | 2394482 |
| 2014-05-15 10:25:52 | gene    | mars    | tricia  | saturn  |  998532 |
| 2014-05-12 12:48:13 | tricia  | mars    | gene    | venus   |  194925 |
| 2014-05-14 14:42:21 | barb    | venus   | barb    | venus   |   98151 |
| 2014-05-11 10:15:08 | barb    | saturn  | tricia  | mars    |   58274 |
+---------------------+---------+---------+---------+---------+---------+
```

# 5.4 删除重复行

## 问题

查询的输出包含重复行。您希望消除它们。

## 解决方案

使用`DISTINCT`。

## 讨论

某些查询会生成包含重复行的结果。例如，要查看谁发送了邮件，请像这样查询`mail`表：

```
mysql> `SELECT srcuser FROM mail;`
+---------+
| srcuser |
+---------+
| barb    |
| tricia  |
| phil    |
| barb    |
| gene    |
| phil    |
| barb    |
| tricia  |
| gene    |
| phil    |
| gene    |
| gene    |
| gene    |
| phil    |
| phil    |
| gene    |
+---------+
```

那个结果非常冗余。要删除重复行并生成一组唯一值，请在查询中添加`DISTINCT`：

```
mysql> `SELECT DISTINCT srcuser FROM mail;`
+---------+
| srcuser |
+---------+
| barb    |
| tricia  |
| phil    |
| gene    |
+---------+
```

要计算列中唯一值的数量，请使用`COUNT(DISTINCT)`：

```
mysql> `SELECT COUNT(DISTINCT srcuser) FROM mail;`
+-------------------------+
| COUNT(DISTINCT srcuser) |
+-------------------------+
|                       4 |
+-------------------------+
```

`DISTINCT`也适用于多列输出。以下查询显示了`mail`表中表示的日期：

```
mysql> `SELECT DISTINCT YEAR(t), MONTH(t), DAYOFMONTH(t) FROM mail;`
+---------+----------+---------------+
| YEAR(t) | MONTH(t) | DAYOFMONTH(t) |
+---------+----------+---------------+
|    2014 |        5 |            11 |
|    2014 |        5 |            12 |
|    2014 |        5 |            14 |
|    2014 |        5 |            15 |
|    2014 |        5 |            16 |
|    2014 |        5 |            19 |
+---------+----------+---------------+
```

## 参见

第十章重新讨论了`DISTINCT`和`COUNT(DISTINCT)`。第十八章更详细地讨论了重复项的移除。

# 5.5 处理 NULL 值

## 问题

您试图比较列值和`NULL`，但却无法正常工作。

## 解决方案

使用适当的比较操作符：`IS NULL`、`IS NOT NULL`或`<=>`。

## 讨论

涉及`NULL`的条件很特殊，因为`NULL`意味着<q>未知值。</q>因此，像*`value`* `=` `NULL`或*`value`* `<>` `NULL`这样的比较总是产生`NULL`（而不是真或假），因为无法确定它们是真还是假。即使`NULL` `=` `NULL`也产生`NULL`，因为您无法确定一个未知值是否与另一个相同。

要查找`NULL`或非`NULL`的值，请使用`IS NULL`或`IS NOT NULL`运算符。假设名为`expt`的表包含待给每个主题四次测试的实验结果，并且表示尚未进行的测试使用`NULL`：

```
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

您可以看到`=`和`<>`无法识别`NULL`值：

```
mysql> `SELECT * FROM expt WHERE score = NULL;`
Empty set (0.00 sec)
mysql> `SELECT * FROM expt WHERE score <> NULL;`
Empty set (0.00 sec)
```

将语句写成这样：

```
mysql> `SELECT * FROM expt WHERE score IS NULL;`
+---------+------+-------+
| subject | test | score |
+---------+------+-------+
| Jane    | C    |  NULL |
| Jane    | D    |  NULL |
| Marvin  | D    |  NULL |
+---------+------+-------+
mysql> `SELECT * FROM expt WHERE score IS NOT NULL;`
+---------+------+-------+
| subject | test | score |
+---------+------+-------+
| Jane    | A    |    47 |
| Jane    | B    |    50 |
| Marvin  | A    |    52 |
| Marvin  | B    |    45 |
| Marvin  | C    |    53 |
+---------+------+-------+
mysql> `SELECT * FROM expt WHERE score <=> NULL;`
+---------+------+-------+
| subject | test | score |
+---------+------+-------+
| Jane    | C    |  NULL |
| Jane    | D    |  NULL |
| Marvin  | D    |  NULL |
+---------+------+-------+
```

MySQL 特定的`<=>`空安全比较运算符，与`=``运算符不同，即使是两个`NULL`值也为真：

```
mysql> `SELECT NULL = NULL, NULL <=> NULL;`
+-------------+---------------+
| NULL = NULL | NULL <=> NULL |
+-------------+---------------+
|        NULL |             1 |
+-------------+---------------+
```

有时候将`NULL`值映射到应用程序上下文中具有更多含义的其他值是很有用的。例如，使用`IF()`将`NULL`映射到字符串`Unknown`：

```
mysql> `SELECT subject, test, IF(score IS NULL,'Unknown', score) AS 'score'`
    -> `FROM expt;`
+---------+------+---------+
| subject | test | score   |
+---------+------+---------+
| Jane    | A    | 47      |
| Jane    | B    | 50      |
| Jane    | C    | Unknown |
| Jane    | D    | Unknown |
| Marvin  | A    | 52      |
| Marvin  | B    | 45      |
| Marvin  | C    | 53      |
| Marvin  | D    | Unknown |
+---------+------+---------+
```

这种基于`IF()`的映射技术适用于任何类型的值，但对于`NULL`值尤其有用，因为`NULL`往往被赋予多种含义：未知、缺失、尚未确定、超出范围等等。选择在特定上下文中最合适的标签。

前面的查询可以更简洁地使用 `IFNULL()` 编写，它测试其第一个参数，如果不为 `NULL` 则返回该参数，否则返回其第二个参数：

```
SELECT subject, test, IFNULL(score,'Unknown') AS 'score'
FROM expt;
```

换句话说，这两个测试是等效的：

```
IF(*`expr1`* IS NOT NULL,*`expr1`*,*`expr2`*)
IFNULL(*`expr1`*,*`expr2`*)
```

从可读性角度来看，`IF()` 往往比 `IFNULL()` 更容易理解。从计算的角度来看，`IFNULL()` 更有效率，因为 *`expr1`* 不需要像 `IF()` 那样评估两次。

映射 `NULL` 值的另一种方法是使用函数 `COALESCE`，它从参数列表中返回第一个非空元素。

```
SELECT subject, test, COALESCE(score,'Unknown') AS 'score' FROM expt;
```

## 另请参阅

当用于排序和汇总操作时，`NULL` 值的行为也不同。参见 Recipe 9.11 和 Recipe 10.9。

# 5.6 编写涉及程序中的 NULL 的比较

## 问题

你正在编写一个程序，查找包含特定值的行，但当该值为 `NULL` 时失败。

## 解决方案

根据比较值是否为 `NULL`，选择适当的比较运算符。

## 讨论

Recipe 5.5 讨论了在 SQL 语句中使用与非 `NULL` 值不同比较运算符的需要。这个问题在构建程序内的语句字符串时可能会导致微妙的危险。如果存储在变量中的值可能表示 `NULL` 值，则在比较中使用该值时必须考虑到这一点。例如，在 Python 中，`None` 表示 `NULL` 值，因此要构建一个语句来查找 `expt` 表中与某个 `score` 变量中的任意值匹配的行，不能这样做：

```
cursor.execute("SELECT * FROM expt WHERE score = %s", (score,))
```

当 `score` 是 `None` 时，语句失败，因为结果语句变成了：

```
SELECT * FROM expt WHERE score = NULL;
```

`score` `=` `NULL` 的比较永远不成立，因此该语句不返回行。为了考虑到 `score` 可能是 `None` 的情况，构建语句时应使用适当的比较运算符，例如这样：

```
operator = "IS" if score is None else "="
cursor.execute("SELECT * FROM expt WHERE score {} %s".format(operator), (score,))
```

这导致对于 `score` 值为 `None` (`NULL`) 或 43（不为 `NULL`）的语句如下所示：

```
SELECT * FROM expt WHERE score IS NULL
SELECT * FROM expt WHERE score = 43;
```

对于不等式测试，应像这样设置 `operator`：

```
operator = "IS NOT" if score is None else "<>"
```

# 5.7 使用视图简化表访问

## 问题

您希望引用从表达式计算的值，而不是每次检索时都写入表达式。

## 解决方案

使用定义了其列执行所需计算的视图。

## 讨论

假设你从 `mail` 表中检索了几个值，使用表达式计算大多数值：

```
mysql> `SELECT`
    -> `DATE_FORMAT(t,'%M %e, %Y') AS date_sent,`
    -> `CONCAT(srcuser,'@',srchost) AS sender,`
    -> `CONCAT(dstuser,'@',dsthost) AS recipient,`
    -> `size FROM mail;`
+--------------+---------------+---------------+---------+
| date_sent    | sender        | recipient     | size    |
+--------------+---------------+---------------+---------+
| May 11, 2014 | barb@saturn   | tricia@mars   |   58274 |
| May 12, 2014 | tricia@mars   | gene@venus    |  194925 |
| May 12, 2014 | phil@mars     | phil@saturn   |    1048 |
| May 12, 2014 | barb@saturn   | tricia@venus  |     271 |
…
```

如果你经常要发出这样的语句，每次都写表达式很不方便。为了更容易访问语句结果，可以使用视图，这是一个不包含数据的虚拟表。相反，它被定义为检索感兴趣数据的 `SELECT` 语句。以下视图 `mail_view` 等效于刚刚显示的 `SELECT` 语句：

```
mysql> `CREATE VIEW mail_view AS`
    -> `SELECT`
    -> `DATE_FORMAT(t,'%M %e, %Y') AS date_sent,`
    -> `CONCAT(srcuser,'@',srchost) AS sender,`
    -> `CONCAT(dstuser,'@',dsthost) AS recipient,`
    -> `size FROM mail;`
```

要访问视图内容，请像访问任何其他表一样引用它。你可以选择它的一些或所有列，添加`WHERE`子句以限制要检索的行，使用`ORDER BY`来对行进行排序，等等。例如：

```
mysql> `SELECT date_sent, sender, size FROM mail_view`
    -> `WHERE size > 100000 ORDER BY size;`
+--------------+---------------+---------+
| date_sent    | sender        | size    |
+--------------+---------------+---------+
| May 12, 2014 | tricia@mars   |  194925 |
| May 15, 2014 | gene@mars     |  998532 |
| May 14, 2014 | tricia@saturn | 2394482 |
+--------------+---------------+---------+
```

存储过程提供了另一种封装计算的方式（参见食谱 11.2）。

# 5.8 从多个表中选择数据

## 问题

答案需要从多个表中获取数据，因此需要从多个表中选择数据。

## 解决方案

使用连接或子查询。

## 讨论

到目前为止，所展示的查询从单个表中选择数据，但有时必须从多个表中检索信息。实现此目的的两种类型的语句是连接和子查询。连接将一个表中的行与另一个表中的行匹配，并使你能够检索包含来自任一表或两个表的列的输出行。子查询是嵌套在另一个查询中的查询，用于通过内部查询选择的值与外部查询选择的值进行比较。

本示例显示了几个简短的例子，以说明基本思想。其他示例出现在其他地方：子查询在本书的各处示例中使用（例如，食谱 5.10 和食谱 10.6）。第十六章详细讨论连接，包括从超过两个表中选择的连接。

下面的示例使用了在第四章介绍的`profile`表。回想一下，它列出了你的好友名单：

```
mysql> `SELECT * FROM profile;`
+----+---------+------------+-------+-----------------------+------+
| id | name    | birth      | color | foods                 | cats |
+----+---------+------------+-------+-----------------------+------+
|  1 | Sybil   | 1970-04-13 | black | lutefisk,fadge,pizza  |    0 |
|  2 | Nancy   | 1969-09-30 | white | burrito,curry,eggroll |    3 |
|  3 | Ralph   | 1973-11-02 | red   | eggroll,pizza         |    4 |
|  4 | Lothair | 1963-07-04 | blue  | burrito,curry         |    5 |
|  5 | Henry   | 1965-02-14 | red   | curry,fadge           |    1 |
|  6 | Aaron   | 1968-09-17 | green | lutefisk,fadge        |    1 |
|  7 | Joanna  | 1952-08-20 | green | lutefisk,fadge        |    0 |
|  8 | Stephen | 1960-05-01 | white | burrito,pizza         |    0 |
+----+---------+------------+-------+-----------------------+------+
```

让我们扩展使用`profile`表，包括另一个名为`profile_contact`的表。此第二个表示如何通过各种社交媒体服务联系`profile`表中列出的人员，并定义如下：

```
CREATE TABLE profile_contact
(
  id           INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  profile_id   INT UNSIGNED NOT NULL, # ID from profile table
  service      VARCHAR(20) NOT NULL,  # social media service name
  contact_name VARCHAR(25) NOT NULL,  # name to use for contacting person
  INDEX (profile_id)
);
```

该表通过`profile_id`列将每行与适当的`profile`行关联起来。`service`和`contact_name`列命名了媒体服务和用于通过该服务联系给定人员的名称。对于示例，请假设该表包含以下行：

```
mysql> `SELECT profile_id, service, contact_name`
    -> `FROM profile_contact ORDER BY profile_id, service;`
+------------+----------+--------------+
| profile_id | service  | contact_name |
+------------+----------+--------------+
|          1 | Facebook | user1-fbid   |
|          1 | Twitter  | user1-twtrid |
|          2 | Facebook | user2-msnid  |
|          2 | LinkedIn | user2-lnkdid |
|          2 | Twitter  | user2-fbrid  |
|          4 | LinkedIn | user4-lnkdid |
+------------+----------+--------------+
```

一个需要来自两个表信息的问题是：<q>对于`profile`表中的每个人员，向我展示我可以使用哪些服务进行联系，以及每个服务的联系人姓名。</q>为了回答这个问题，使用连接。从两个表中选择并通过比较`profile`表的`id`列与`profile_contact`表的`profile_id`列来匹配行：

```
mysql> `SELECT profile.id, name, service, contact_name`
    -> `FROM profile INNER JOIN profile_contact ON profile.id = profile_id;`
+----+---------+----------+--------------+
| id | name    | service  | contact_name |
+----+---------+----------+--------------+
|  1 | Sybil   | Twitter  | user1-twtrid |
|  1 | Sybil   | Facebook | user1-fbid   |
|  2 | Nancy   | Twitter  | user2-fbrid  |
|  2 | Nancy   | Facebook | user2-msnid  |
|  2 | Nancy   | LinkedIn | user2-lnkdid |
|  4 | Lothair | LinkedIn | user4-lnkdid |
+----+---------+----------+--------------+
```

`FROM`子句指示要从中选择数据的表，`ON`子句告诉 MySQL 使用哪些列在表之间查找匹配项。在结果中，行包括`profile`表的`id`和`name`列，以及`profile_contact`表的`service`和`contact_name`列。

这是另一个需要两个表才能回答的问题：<q>列出所有 Nancy 的 `profile_contact` 记录。</q> 要从 `profile_contact` 表中提取正确的行，您需要 Nancy 的 ID，该 ID 存储在 `profile` 表中。要编写查询而不自行查找 Nancy 的 ID，请使用一个子查询，该子查询根据她的姓名为您查找：

```
mysql> `SELECT profile_id, service, contact_name FROM profile_contact`
    -> `WHERE profile_id = (SELECT id FROM profile WHERE name = 'Nancy');`
+------------+----------+--------------+
| profile_id | service  | contact_name |
+------------+----------+--------------+
|          2 | Twitter  | user2-fbrid  |
|          2 | Facebook | user2-msnid  |
|          2 | LinkedIn | user2-lnkdid |
+------------+----------+--------------+
```

此处子查询显示为括在括号中的嵌套 `SELECT` 语句。

# 5.9 从查询结果的开始、结尾或中间选择行

## 问题

您只希望从结果集中获取特定的行，如第一行、最后五行或第 21 至 40 行。

## 解决方案

使用 `LIMIT` 子句，也许与 `ORDER` `BY` 子句结合使用。

## 讨论

MySQL 支持 `LIMIT` 子句，告诉服务器仅返回结果集的一部分。`LIMIT` 是 SQL 的 MySQL 特定扩展，在结果集包含比您想要一次看到的行数更多时非常有价值。它允许您检索结果集的任意部分。典型的 `LIMIT` 使用包括以下类型的问题：

+   回答有关第一个或最后一个、最大或最小、最新或最旧、最便宜或最昂贵等问题。

+   将结果集分成几个部分，以便可以逐部分处理它。此技术在 Web 应用程序中很常见，用于在多页上显示大型搜索结果。以部分显示结果可以显示更小、更易理解的页面。

以下示例使用 Recipe 5.8 中显示的 `profile` 表。要查看 `SELECT` 结果的前 *`n`* 行，将 `LIMIT` *`n`* 添加到语句的末尾：

```
mysql> `SELECT * FROM profile LIMIT 1;`
+----+-------+------------+-------+----------------------+------+
| id | name  | birth      | color | foods                | cats |
+----+-------+------------+-------+----------------------+------+
|  1 | Sybil | 1970-04-13 | black | lutefisk,fadge,pizza |    0 |
+----+-------+------------+-------+----------------------+------+
mysql> `SELECT * FROM profile LIMIT 3;`
+----+-------+------------+-------+-----------------------+------+
| id | name  | birth      | color | foods                 | cats |
+----+-------+------------+-------+-----------------------+------+
|  1 | Sybil | 1970-04-13 | black | lutefisk,fadge,pizza  |    0 |
|  2 | Nancy | 1969-09-30 | white | burrito,curry,eggroll |    3 |
|  3 | Ralph | 1973-11-02 | red   | eggroll,pizza         |    4 |
+----+-------+------------+-------+-----------------------+------+
```

`LIMIT` *`n`* 意味着 <q>返回 *最多* *`n`* 行。</q> 如果指定 `LIMIT` `10`，而结果集只有四行，则服务器返回四行。

前述查询结果中的行未按任何特定顺序返回，因此可能无太大意义。更常见的技术是使用 `ORDER` `BY` 对结果集排序并使用 `LIMIT` 查找最小值和最大值。例如，要找到具有最小（最早）出生日期的行，请按 `birth` 列排序，然后添加 `LIMIT` `1` 以检索第一行：

```
mysql> `SELECT * FROM profile ORDER BY birth LIMIT 1;`
+----+--------+------------+-------+----------------+------+
| id | name   | birth      | color | foods          | cats |
+----+--------+------------+-------+----------------+------+
|  7 | Joanna | 1952-08-20 | green | lutefisk,fadge |    0 |
+----+--------+------------+-------+----------------+------+
```

这有效是因为 MySQL 处理 `ORDER` `BY` 子句来对行进行排序，然后应用 `LIMIT`。

要获取结果集末尾的行，请按相反顺序排序它们。查找最近出生日期的语句类似于前一个语句，但排序顺序是降序：

```
mysql> `SELECT * FROM profile ORDER BY birth DESC LIMIT 1;`
+----+-------+------------+-------+---------------+------+
| id | name  | birth      | color | foods         | cats |
+----+-------+------------+-------+---------------+------+
|  3 | Ralph | 1973-11-02 | red   | eggroll,pizza |    4 |
+----+-------+------------+-------+---------------+------+
```

要查找日历年内最早或最晚的生日，请按 `birth` 值的月和日排序：

```
mysql> `SELECT name, DATE_FORMAT(birth,'%m-%d') AS birthday`
    -> `FROM profile ORDER BY birthday LIMIT 1;`
+-------+----------+
| name  | birthday |
+-------+----------+
| Henry | 02-14    |
+-------+----------+
```

您可以通过在没有 `LIMIT` 的情况下运行这些语句并忽略除第一行以外的所有内容来获得相同的信息。`LIMIT` 的优点是服务器仅返回第一行，而额外的行根本不跨网络。这比检索整个结果集并丢弃除一个行以外的所有行要高效得多。

要从结果集的中间提取行，请使用 `LIMIT` 的两参数形式，这使您可以选择任意部分的行。参数指示要跳过多少行以及要返回多少行。这意味着您可以使用 `LIMIT` 来执行诸如跳过两行并返回下一行的操作，从而回答诸如“第三个最小值或第三个最大值是什么？”这些问题 `MIN()` 或 `MAX()` 不适合，但 `LIMIT` 却很容易实现：

```
mysql> `SELECT * FROM profile ORDER BY birth LIMIT 2,1;`
+----+---------+------------+-------+---------------+------+
| id | name    | birth      | color | foods         | cats |
+----+---------+------------+-------+---------------+------+
|  4 | Lothair | 1963-07-04 | blue  | burrito,curry |    5 |
+----+---------+------------+-------+---------------+------+
mysql> `SELECT * FROM profile ORDER BY birth DESC LIMIT 2,1;`
+----+-------+------------+-------+-----------------------+------+
| id | name  | birth      | color | foods                 | cats |
+----+-------+------------+-------+-----------------------+------+
|  2 | Nancy | 1969-09-30 | white | burrito,curry,eggroll |    3 |
+----+-------+------------+-------+-----------------------+------+
```

`LIMIT` 的两参数形式还可以将结果集分成较小的部分。例如，要从结果中每次检索 20 行，重复发出 `SELECT` 语句，但变化其 `LIMIT` 子句如下：

```
SELECT ... FROM ... ORDER BY ... LIMIT 0, 20;
SELECT ... FROM ... ORDER BY ... LIMIT 20, 20;
SELECT ... FROM ... ORDER BY ... LIMIT 40, 20;
…
```

###### 警告

对于大数据集，使用 `LIMIT` 子句的这种方式可能会导致性能下降，因为它需要读取至少 `OFFSET` 加 `LIMIT` 行。这意味着要获得 `LIMIT 0, 20` 语句的结果，MySQL 必须从表中读取 20 行，要获得 `LIMIT 20, 20` 的结果，它需要读取 40 行，依此类推。

要确定结果集中的行数，以便确定部分的数量，请首先发出 `COUNT()` 语句。例如，要按名称顺序显示 `profile` 表行，每次三个，您可以使用以下语句找出行数：

```
mysql> `SELECT COUNT(*) FROM profile;`
+----------+
| COUNT(*) |
+----------+
|        8 |
+----------+
```

这告诉你有三组行（最后一组少于三行），你可以按以下方式检索：

```
SELECT * FROM profile ORDER BY name LIMIT 0, 3;
SELECT * FROM profile ORDER BY name LIMIT 3, 3;
SELECT * FROM profile ORDER BY name LIMIT 6, 3;
```

## 参见

结合 `RAND()` 使用 `LIMIT` 可以从一组项目中进行随机选择。请参阅 Recipe 17.8。

您可以使用 `LIMIT` 限制 `DELETE` 或 `UPDATE` 语句的影响范围，以防止删除或更新除特定行之外的所有行。有关使用 `LIMIT` 去重的更多信息，请参阅 Recipe 18.5。

# 5.10 当 `LIMIT` 和最终结果要求不同的排序顺序时该怎么办

## 问题

`LIMIT` 通常最好与排序行的 `ORDER BY` 子句结合使用。但有时候这种排序顺序与你希望的最终结果不同。

## 解决方案

在子查询中使用 `LIMIT` 检索所需的行，然后在外部查询中对它们进行排序。

## 讨论

如果要获取结果集的最后四行，可以通过以相反顺序排序并使用 `LIMIT 4` 轻松获取它们。以下语句返回 `profile` 表中最近出生的四个人的姓名和出生日期：

```
mysql> `SELECT name, birth FROM profile ORDER BY birth DESC LIMIT 4;`
+-------+------------+
| name  | birth      |
+-------+------------+
| Ralph | 1973-11-02 |
| Sybil | 1970-04-13 |
| Nancy | 1969-09-30 |
| Aaron | 1968-09-17 |
+-------+------------+
```

但这要求以降序排序 `birth` 值，以将它们放在结果集的前面。如果希望输出行按升序顺序显示怎么办？将 `SELECT` 用作外部语句的子查询，以按所需的最终顺序重新对行进行排序：

```
mysql> `SELECT * FROM`
    -> `(SELECT name, birth FROM profile ORDER BY birth DESC LIMIT 4) AS t`
    -> `ORDER BY birth;`
+-------+------------+
| name  | birth      |
+-------+------------+
| Aaron | 1968-09-17 |
| Nancy | 1969-09-30 |
| Sybil | 1970-04-13 |
| Ralph | 1973-11-02 |
+-------+------------+
```

因为 `FROM` 子句中引用的任何表都必须有一个名称，所以在此处使用 `AS` `t`，即使是从子查询生成的 <q>派生</q> 表也是如此。

# 5.11 从表达式计算 LIMIT 值

## 问题

您想使用表达式来指定 `LIMIT` 的参数。

## 解决方案

`LIMIT` 的参数必须是文字整数 —— 除非您在允许动态构建语句字符串的上下文中发出该语句。在这种情况下，您可以自行评估表达式，并将结果值插入语句字符串中。

## 讨论

`LIMIT` 的参数必须是文字整数，而不是表达式。类似以下语句是不合法的：

```
SELECT * FROM profile LIMIT 5+5;
SELECT * FROM profile LIMIT @skip_count, @show_count;
```

如果在构造语句字符串的程序中使用表达式计算 `LIMIT` 值，则同样适用 <q>不允许表达式</q> 原则。您必须先评估表达式，然后将结果值放入语句中。例如，如果您在 Perl 或 PHP 中生成语句字符串如下所示，在执行语句时会出现错误：

```
$str = "SELECT * FROM profile LIMIT $x + $y";
```

为了避免问题，请先评估表达式：

```
$z = $x + $y;
$str = "SELECT * FROM profile LIMIT $z";
```

或者这样做（不要省略括号，否则表达式将无法正确评估）：

```
$str = "SELECT * FROM profile LIMIT " . ($x + $y);
```

要构造一个两个参数的 `LIMIT` 子句，请先评估这两个表达式，然后将它们放入语句字符串中。

# 5.12 结合两个或更多 SELECT 结果

## 问题

您希望将两个或更多 `SELECT` 语句检索的行合并到一个结果集中。

## 解决方案

使用 `UNION` 子句。

## 讨论

`mail` 表存储电子邮件发送者和接收者的用户名称和主机。但是，如果我们想知道所有可能的用户和主机组合呢？

一个简单的方法是选择发送者或接收者对。但是，如果我们进行基本测试，比较唯一用户-主机组合的数量，我们会发现每个方向的数量是不同的。

```
mysql> `SELECT` `COUNT``(``distinct` `srcuser``,` `srchost``)` `FROM` `mail``;`
+----------------------------------+ | count(distinct srcuser, srchost) |
+----------------------------------+ |                                9 |
+----------------------------------+ 1 row in set (0.01 sec)

mysql> `select` `count``(``distinct` `dstuser``,` `dsthost``)` `from` `mail``;`
+----------------------------------+ | count(distinct dstuser, dsthost) |
+----------------------------------+ |                               10 |
+----------------------------------+ 1 row in set (0.00 sec)
```

我们也不知道我们的表是否存储只发送而不接收邮件的用户，以及只接收但从不发送邮件的用户。

要获取完整列表，我们需要为发送者和接收者选择成对，然后删除重复项。SQL 子句 `UNION DISTINCT` 及其简短形式 `UNION` 正好做到这一点。它将两个或更多选择相同列类型的 `SELECT` 查询结果组合在一起。

默认情况下，`UNION` 使用第一个 `SELECT` 的列名作为完整结果集的标题，但是我们也可以像 Recipe 5.2 中讨论的那样使用别名。

```
mysql> `SELECT` `DISTINCT` `srcuser` `AS` `user``,` `srchost` `AS` `host` `FROM` `mail`
    -> `UNION`
    -> `SELECT` `DISTINCT` `dstuser` `AS` `user``,` `dsthost` `AS` `host` `FROM` `mail``;`
+--------+--------+ | user   | host   |
+--------+--------+ | barb   | saturn |
| tricia | mars   |
| phil   | mars   |
| gene   | venus  |
| barb   | venus  |
| tricia | saturn |
| gene   | mars   |
| phil   | venus  |
| gene   | saturn |
| phil   | saturn |
| tricia | venus  |
| barb   | mars   |
+--------+--------+ 12 rows in set (0.00 sec)
```

您可以将每个独立查询排序，参与 `UNION`，以及整个结果。如果不希望从结果中删除重复项，请使用 `UNION ALL` 子句。

为了演示这一点，让我们创建一个查询，找出发送最多电子邮件的四个用户和接收最多电子邮件的四个用户，然后按用户名对联合结果进行排序。

```
mysql> `(``SELECT` `CONCAT``(``srcuser``,` `'@'``,` `srchost``)` `AS` `user``,` `COUNT``(``*``)` `AS` `emails` ![1](img/1.png)
    -> `FROM` `mail` `GROUP` `BY` `srcuser``,` `srchost` `ORDER` `BY` `emails` `DESC` `LIMIT` `4``)` ![2](img/2.png)
    -> `UNION` `ALL`
    -> `(``SELECT` `CONCAT``(``dstuser``,` `'@'``,` `dsthost``)` `AS` `user``,` `COUNT``(``*``)` `AS` `emails` 
    -> `FROM` `mail` `GROUP` `BY` `dstuser``,` `dsthost` `ORDER` `BY` `emails` `DESC` `LIMIT` `4``)` ![3](img/3.png)
    -> `ORDER` `BY` `user``;`![4](img/4.png)
+---------------+--------+ | user          | emails |
+---------------+--------+ | barb@mars     |      2 |
| barb@saturn   |      2 |
| barb@venus    |      2 |
| gene@saturn   |      2 |
| gene@venus    |      2 | ![5](img/5.png)
| gene@venus    |      2 |
| phil@mars     |      3 |
| tricia@saturn |      3 |
+---------------+--------+ 8 rows in set (0.00 sec)
```

![1](img/#co_nch-select-select-union-concat_co)

将用户和主机连接成用户的电子邮件地址。

![2](img/#co_nch-select-select-union-order_co)

首先按电子邮件数量降序排序第一个 `SELECT` 的结果，并限制检索到的行数。

![3](img/#co_nch-select-select-union-order2_co)

对第二个`SELECT`结果进行排序。

![4](img/#co_nch-select-select-union-order-union_co)

按用户电子邮件地址对`UNION`的结果排序。

![5](img/#co_nch-select-select-union-duplicate_co)

我们使用了`UNION ALL`子句而不是`UNION [DISTINCT]`，因此结果中`gene@venus`有两个条目。这个用户是发送和接收电子邮件最多的名单中的一员。

# 5.13 选择子查询的结果

## 问题

你希望检索的不仅是表列，还包括使用这些列的查询的结果。

## 解决方案

在列列表中使用子查询。

## 讨论

假设您不仅想知道特定用户发送了多少封电子邮件，还想知道他们收到了多少封。您不能只访问`mail`表两次来实现这一点：一次用于计算发送的电子邮件数，第二次用于计算接收的电子邮件数。

解决此问题的一个解决方案是在列列表中使用子查询。

```
mysql> `SELECT` `CONCAT``(``srcuser``,` `'@'``,` `srchost``)` `AS` `user``,` `COUNT``(``*``)` `AS` `mails_sent``,` ![1](img/1.png)
    -> `(``SELECT` `COUNT``(``*``)` `FROM` `mail` `d` `WHERE` `d``.``dstuser``=``m``.``srcuser` `AND` `d``.``dsthost``=``m``.``srchost``)`  ![2](img/2.png)
    -> `AS` `mails_received` ![3](img/3.png)
    -> `FROM` `mail` `m` 
    -> `GROUP` `BY`  `srcuser``,` `srchost`  ![4](img/4.png)
    -> `ORDER` `BY` `mails_sent` `DESC``;`
+---------------+------------+----------------+ | user          | mails_sent | mails_received |
+---------------+------------+----------------+ | phil@mars     |          3 |              0 |
| barb@saturn   |          2 |              0 |
| gene@venus    |          2 |              2 |
| gene@mars     |          2 |              1 |
| phil@venus    |          2 |              2 |
| gene@saturn   |          2 |              1 |
| tricia@mars   |          1 |              1 |
| barb@venus    |          1 |              2 |
| tricia@saturn |          1 |              3 |
+---------------+------------+----------------+ 9 rows in set (0.00 sec)
```

![1](img/#co_nch-select-select-subqueries-sent_co)

首先，我们检索了发送者的用户名和主机，并计算了他们发送的电子邮件数。

![2](img/#co_nch-select-select-subqueries-received_co)

为了找出此用户收到的电子邮件数量，我们在相同的`mail`表中使用子查询。在`WHERE`子句中，我们只选择接收者与主查询中的发送者具有相同凭据的行。

![3](img/#co_nch-select-select-subqueries-received_alias_co)

列列表中的子查询必须有自己的别名。

![4](img/#co_nch-select-select-subqueries-group_co)

为了按用户显示统计信息，我们使用了`GROUP BY`子句，因此结果按每个用户名和主机分组。我们在第十章中详细讨论了`GROUP BY`子句。
