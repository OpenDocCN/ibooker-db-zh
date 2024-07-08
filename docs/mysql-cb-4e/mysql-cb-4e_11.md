# 第十一章：使用存储例程、触发器和预定事件

# 11.0 简介

本书中，“存储程序”一词指的是存储例程、触发器和事件的总称，“存储例程”一词指的是存储函数和存储过程的总称。

本章讨论多种存储程序：

存储函数和存储过程

存储函数或过程对象封装了执行操作的代码，使您可以通过名称轻松调用对象，而不是每次需要时重复所有代码。存储函数执行计算并返回一个值，可以像内置函数（如`RAND()`、`NOW()`或`LEFT()`）一样在表达式中使用。存储过程执行不需要返回值的操作。使用`CALL`语句调用过程，不在表达式中使用。过程可能会更新表中的行或生成发送到客户端程序的结果集。

触发器

触发器是一个对象，当表通过`INSERT`、`UPDATE`或`DELETE`语句修改时被激活。例如，可以在将值插入表之前检查它们，或者指定从表中删除的任何行应记录到另一个作为数据变更日志的表中。触发器自动化这些操作。

预定事件

事件是在预定时间或多个时间点执行 SQL 语句的对象。将预定事件视为 MySQL 内部类似 Unix *cron*作业的内容。例如，事件可帮助您执行如定期删除旧表行或创建每夜摘要等管理任务。

存储程序是用户定义的数据库对象，但存储在服务器端以便以后执行。这与从客户端向服务器发送 SQL 语句进行即时执行不同。每个对象还具有其被调用时执行的其他 SQL 语句的属性。对象体是一个单一的 SQL 语句，但该语句可以使用复合语句语法（一个`BEGIN` … `END`块），其中包含多个语句。因此，对象体可以从非常简单到极其复杂不等。下面的存储过程是一个微不足道的例程，只显示当前 MySQL 版本，使用的是一个包含单个`SELECT`语句的体：

```
CREATE PROCEDURE show_version()
SELECT VERSION() AS 'MySQL Version';
```

更复杂的操作使用`BEGIN` … `END`复合语句：

```
CREATE PROCEDURE show_part_of_day()
BEGIN
  DECLARE cur_time, day_part TEXT;
  SET cur_time = CURTIME();
  IF cur_time < '12:00:00' THEN
    SET day_part = 'morning';
  ELSEIF cur_time = '12:00:00' THEN
    SET day_part = 'noon';
  ELSE
    SET day_part = 'afternoon or night';
  END IF;
  SELECT cur_time, day_part;
END;
```

在这里，`BEGIN` … `END`块包含多个语句，但本身被视为单个语句。复合语句使您能够声明局部变量，并使用条件逻辑和循环结构。这些功能为算法表达提供了比在非复合语句（如`SELECT`或`UPDATE`）中编写内联表达式时更大的灵活性。

复合语句中的每个语句必须以`;`字符结尾。如果您使用*mysql*客户端定义使用复合语句的对象，则此要求会引起问题，因为*mysql*本身会解释`;`以确定语句的边界。解决方案是在定义复合语句对象时重新定义*mysql*的语句分隔符。第 11.1 节介绍了如何做到这一点；确保在继续后面的部分之前阅读该章节。

本章通过示例说明存储例程、触发器和事件，但由于空间限制，未详细介绍它们的广泛语法。有关完整的语法描述，请参阅*MySQL 参考手册*。

本章示例中显示的脚本位于`recipes`分发的*例程*、*触发器*和*事件*目录中。用于创建示例表的脚本位于*表*目录中。

除了本章中显示的存储程序外，还可以在本书的其他地方找到其他存储程序。例如，请参阅第 7.6 节、第 8.3 节、第 16.8 节和第 24.2 节。

此处使用的存储程序是在假设`cookbook`是默认数据库的情况下创建和调用的。要从另一个数据库调用程序，请使用数据库名称限定其名称：

```
CALL cookbook.show_version();
```

或者，为您的存储程序创建一个专门的数据库，在该数据库中创建它们，并始终使用该名称调用它们。请记住为将使用它们的用户授予该数据库的`EXECUTE`权限。

# 11.1 创建复合语句对象

## 问题

您想定义一个存储程序，但其主体包含`；`语句终止符的实例。*mysql*客户端程序默认使用相同的终止符，因此*mysql*会误解定义并产生错误。

## 解决方案

使用`delimiter`命令重新定义*mysql*语句终止符。

## 讨论

每个存储程序都是一个具有主体的对象，该主体必须是一个单个 SQL 语句。然而，这些对象经常执行需要多个语句的复杂操作。为了处理这种情况，将语句写在形成复合语句的`BEGIN` … `END`块内。也就是说，该块本身是一个单一语句，但可以包含多个语句，每个语句以`;`字符结尾。`BEGIN` … `END`块可以包含诸如`SELECT`或`INSERT`之类的语句，但复合语句还允许条件语句（如`IF`或`CASE`）、循环结构（如`WHILE`或`REPEAT`）或其他`BEGIN` … `END`块。

复合语句语法提供了灵活性，但如果在*mysql*客户端内定义复合语句对象，您会很快遇到问题：每个复合语句对象内部的语句必须以`；`字符结尾，但*mysql*本身解释`；`以确定语句的结束位置，从而逐一将其发送到服务器执行。因此，当*mysql*看到第一个`；`字符时，它会停止读取复合语句，这太早了。为了解决这个问题，告诉*mysql*识别不同的语句分隔符，以便忽略对象主体内部的`；`字符。使用新的分隔符终止对象本身，*mysql*会识别并将整个对象定义发送到服务器。在定义复合语句对象后，可以将*mysql*分隔符恢复为其原始值。

以下示例使用存储函数说明如何更改分隔符，但原则适用于定义任何类型的存储程序。

假设您想创建一个存储函数，计算并返回`mail`表中列出的邮件消息的平均大小（以字节为单位）。可以像这样定义函数，其中主体由单个 SQL 语句组成：

```
CREATE FUNCTION avg_mail_size()
RETURNS FLOAT READS SQL DATA
RETURN (SELECT AVG(size) FROM mail);
```

`RETURNS FLOAT`子句指示函数返回值的类型，而`READS SQL DATA`指示函数读取但不修改数据。函数主体遵循这些子句：一个执行子查询并将结果值返回给调用者的单个`RETURN`语句。（每个存储函数必须至少有一个`RETURN`语句。）

在*mysql*中，您可以如上所示输入该语句，没有问题。定义只需要在末尾的单个终止符，而不需要内部终止符，因此不会产生歧义。但是，假设您希望函数接受一个用户命名的参数，并按以下方式解释它：

+   如果参数为`NULL`，函数返回所有消息的平均大小（与以前相同）。

+   如果参数非`NULL`，函数返回该用户发送消息的平均大小。

为了实现这一点，函数具有更复杂的主体，使用`BEGIN` … `END`块：

```
CREATE FUNCTION avg_mail_size(user VARCHAR(8))
RETURNS FLOAT READS SQL DATA
BEGIN
  DECLARE avg FLOAT;
  IF user IS NULL
  THEN # average message size over all users
    SET avg = (SELECT AVG(size) FROM mail);
  ELSE # average message size for given user
    SET avg = (SELECT AVG(size) FROM mail WHERE srcuser = user);
  END IF;
  RETURN avg;
END;
```

如果您尝试仅输入所示的定义在*mysql*内部定义函数，*mysql*会错误地将函数体中的第一个分号解释为结束定义。相反，使用`delimiter`命令更改*mysql*分隔符，然后将分隔符恢复为其默认值：

```
mysql> `delimiter $$`
mysql> `CREATE FUNCTION avg_mail_size(user VARCHAR(8))`
    -> `RETURNS FLOAT READS SQL DATA`
    -> `BEGIN`
    ->   `DECLARE avg FLOAT;`
    ->   `IF user IS NULL`
    ->   `THEN # average message size over all users`
    ->     `SET avg = (SELECT AVG(size) FROM mail);`
    ->   `ELSE # average message size for given user`
    ->     `SET avg = (SELECT AVG(size) FROM mail WHERE srcuser = user);`
    ->   `END IF;`
    ->   `RETURN avg;`
    -> `END;`
    -> `$$`
Query OK, 0 rows affected (0.02 sec)
mysql> `delimiter ;`
```

定义存储函数后，调用它的方式与内置函数相同：

```
mysql> `SELECT avg_mail_size(NULL), avg_mail_size('barb');`
+---------------------+-----------------------+
| avg_mail_size(NULL) | avg_mail_size('barb') |
+---------------------+-----------------------+
|         237386.5625 |                 52232 |
+---------------------+-----------------------+
```

# 使用存储函数简化计算（11.2 章节）

## 问题

特定的计算以产生值必须由不同的应用程序频繁执行，但您不希望每次需要时都编写表达式来执行它。或者计算在表达式内联中难以执行，因为它需要条件或循环逻辑。或者，如果计算逻辑发生变化，您不希望在每个使用它的应用程序中进行更改。

## 解决方案

使用存储函数可以在单个位置定义这些详细信息，并使计算变得简单易行。

## 讨论

存储函数使您能够简化应用程序。编写一次在函数定义中生成计算结果的代码，然后在需要执行计算时简单调用该函数。存储函数还使您能够使用比在表达式内联写计算更复杂的算法结构。本节说明存储函数在这些方面如何有用。（当然，这个例子不是*那么*复杂，但这里使用的原则同样适用于编写更复杂的函数。）

美国不同州对销售税收取不同的费率。如果向来自不同州的人们销售商品，则必须使用适合客户居住州的税率来收税。要处理税收计算，使用一个列出每个州销售税率的表，并使用一个存储函数来查找给定州的税率。

要设置`sales_tax_rate`表，请使用`recipes`分发中`tables`目录下的*sales_tax_rate.sql*脚本。该表有两列：`state`（两字母缩写）和`tax_rate`（`DECIMAL`值而不是`FLOAT`，以保证精度）。

定义查找税率的函数`sales_tax_rate()`如下：

```
CREATE FUNCTION sales_tax_rate(state_code CHAR(2))
RETURNS DECIMAL(3,2) READS SQL DATA
BEGIN
  DECLARE rate DECIMAL(3,2);
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET rate = 0;
  SELECT tax_rate INTO rate FROM sales_tax_rate WHERE state = state_code;
  RETURN rate;
END;
```

假设佛蒙特州和纽约州的税率分别为 1%和 9%。尝试函数以检查是否正确返回了税率：

```
mysql> `SELECT sales_tax_rate('VT'), sales_tax_rate('NY');`
+----------------------+----------------------+
| sales_tax_rate('VT') | sales_tax_rate('NY') |
+----------------------+----------------------+
|                 0.01 |                 0.09 |
+----------------------+----------------------+
```

如果您从未在表中列出的位置进行销售，函数无法确定其税率。在这种情况下，函数假定税率为 0%：

```
mysql> `SELECT sales_tax_rate('ZZ');`
+----------------------+
| sales_tax_rate('ZZ') |
+----------------------+
|                 0.00 |
+----------------------+
```

如果给定的`state_param`值没有行，`SELECT`语句无法找到销售税率，`CONTINUE`处理程序会设置税率为 0，并在`SELECT`后继续执行下一个语句。（这个处理程序是内联表达式中不可用的存储例程逻辑的示例。“在存储程序中处理错误”进一步讨论了处理程序。）

要计算购买的销售税，将购买价格乘以税率。例如，对于佛蒙特州和纽约州，对于一笔$150 的购买，税额为：

```
mysql> `SELECT 150*sales_tax_rate('VT'), 150*sales_tax_rate('NY');`
+--------------------------+--------------------------+
| 150*sales_tax_rate('VT') | 150*sales_tax_rate('NY') |
+--------------------------+--------------------------+
|                     1.50 |                    13.50 |
+--------------------------+--------------------------+
```

或者编写另一个函数来为您计算税款：

```
CREATE FUNCTION sales_tax(state_code CHAR(2), sales_amount DECIMAL(10,2))
RETURNS DECIMAL(10,2) READS SQL DATA
RETURN sales_amount * sales_tax_rate(state_code);
```

并像这样使用它：

```
mysql> `SELECT sales_tax('VT',150), sales_tax('NY',150);`
+---------------------+---------------------+
| sales_tax('VT',150) | sales_tax('NY',150) |
+---------------------+---------------------+
|                1.50 |               13.50 |
+---------------------+---------------------+
```

# 11.3 使用存储过程生成多个值

## 问题

您希望为操作生成多个值，但存储函数只能返回单个值。

## 解决方案

使用具有 `OUT` 或 `INOUT` 参数的存储过程，并在调用过程时为这些参数传递用户定义的变量。存储过程不像函数那样返回一个值，但它可以将值分配给这些参数，以便在过程返回时用户定义的变量具有所需的值。

## 讨论

与仅限于输入值的存储函数参数不同，存储过程参数可以是以下三种类型之一：

+   `IN` 参数仅用于输入。如果未指定类型，则默认为此。

+   `INOUT` 参数用于传递一个值进入，并且也可以传递一个值出去。

+   `OUT` 参数用于传递一个值出去。

因此，为了从操作中生成多个值，您可以使用 `INOUT` 或 `OUT` 参数。以下示例说明了这一点，使用 `IN` 参数作为输入，并通过 `OUT` 参数传回三个值。

Recipe 11.1 展示了一个 `avg_mail_size()` 函数，返回给定发件人的平均邮件大小。该函数只返回一个值。要生成额外的信息，例如消息数量和总消息大小，函数无法胜任。您可以编写三个单独的函数，但这很麻烦。相反，请使用一个单独的过程来检索关于给定发件人的多个值。以下过程 `mail_sender_stats()` 在 `mail` 表上运行查询，以获取有关输入值（用户名）的邮件发送统计信息，通过三个 `OUT` 参数返回该用户发送的消息数量，以及消息的总大小和平均大小（以字节为单位）：

```
CREATE PROCEDURE mail_sender_stats(IN user VARCHAR(8),
                                   OUT messages INT,
                                   OUT total_size INT,
                                   OUT avg_size INT)
BEGIN
  # Use IFNULL() to return 0 for SUM() and AVG() in case there are
  # no rows for the user (those functions return NULL in that case).
  SELECT COUNT(*), IFNULL(SUM(size),0), IFNULL(AVG(size),0)
  INTO messages, total_size, avg_size
  FROM mail WHERE srcuser = user;
END;
```

要使用该过程，请传递包含用户名和三个用户定义变量以接收 `OUT` 值的字符串。过程返回后，访问变量值：

```
mysql> `CALL mail_sender_stats('barb',@messages,@total_size,@avg_size);`
mysql> `SELECT @messages, @total_size, @avg_size;`
+-----------+-------------+-----------+
| @messages | @total_size | @avg_size |
+-----------+-------------+-----------+
|         3 |      156696 |     52232 |
+-----------+-------------+-----------+
```

此例程返回计算结果。通常也会使用 `OUT` 参数用于诊断目的，例如状态或错误指示器。

如果在存储程序中调用 `mail_sender_stats()`，您可以使用例程参数或程序本地变量传递变量给它，而不仅限于用户定义的变量。

# 11.4 使用触发器记录表格更改

## 问题

您有一个表格，用于维护您跟踪的项目（如正在竞标的拍卖）的当前值，但您还希望维护表格更改的日志（历史记录）。

## 解决方案

使用触发器来捕获表格更改并将其写入单独的日志表。

## 讨论

假设您进行在线拍卖，并在类似以下表格中维护每个当前活动拍卖的信息：

```
CREATE TABLE auction
(
  id   INT UNSIGNED NOT NULL AUTO_INCREMENT,
  ts   TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  item VARCHAR(30) NOT NULL,
  bid  DECIMAL(10,2) NOT NULL,
  PRIMARY KEY (id)
);
```

`auction`表包含关于当前活动拍卖的信息（正在竞价的物品以及每个拍卖的当前出价）。当拍卖开始时，向表中插入一行。对于物品的每次出价，更新其`bid`列，以便随着拍卖的进行，`ts`列更新以反映最近的出价时间。当拍卖结束时，`bid`值是最终价格，并且可以从表中删除该行。

为了维护一个展示从创建到移除的拍卖所有变化的日志，设置另一个表，用于记录拍卖变化的历史。可以通过触发器来实现这一策略。

为了记录每次拍卖的进展历史，请使用一个`auction_log`表，其包含以下列：

```
CREATE TABLE auction_log
(
  action ENUM('create','update','delete'),
  id    INT UNSIGNED NOT NULL,
  ts    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  item  VARCHAR(30) NOT NULL,
  bid   DECIMAL(10,2) NOT NULL,
  INDEX (id)
);
```

`auction_log`表与`auction`表有两个不同之处：

+   它包含一个`action`列，用于指示每行所做的变更类型。

+   `id`列具有非唯一索引（而不是需要唯一值的主键）。这允许每个`id`值有多行，因为一个拍卖可以在日志表中生成多行。

为了确保将`auction`表的更改记录到`auction_log`表中，请创建一组触发器。这些触发器将信息写入`auction_log`表，如下所示：

+   对于插入操作，记录一次行创建操作，显示新行中的值。

+   对于更新操作，记录一次行更新操作，显示更新行中的新值。

+   对于删除操作，记录一次行删除操作，显示已删除行中的值。

对于此应用程序，使用`AFTER`触发器，因为它们仅在对`auction`表的更改成功后激活。（如果某些原因导致行更改操作失败，则`BEFORE`触发器可能会激活。）触发器定义如下：

```
CREATE TRIGGER ai_auction AFTER INSERT ON auction
FOR EACH ROW
INSERT INTO auction_log (action,id,ts,item,bid)
VALUES('create',NEW.id,NOW(),NEW.item,NEW.bid);

CREATE TRIGGER au_auction AFTER UPDATE ON auction
FOR EACH ROW
INSERT INTO auction_log (action,id,ts,item,bid)
VALUES('update',NEW.id,NOW(),NEW.item,NEW.bid);

CREATE TRIGGER ad_auction AFTER DELETE ON auction
FOR EACH ROW
INSERT INTO auction_log (action,id,ts,item,bid)
VALUES('delete',OLD.id,OLD.ts,OLD.item,OLD.bid);
```

`INSERT`和`UPDATE`触发器使用`NEW.`*`col_name`*来访问存储在行中的新值。`DELETE`触发器使用`OLD.`*`col_name`*来访问已删除行中的现有值。`INSERT`和`UPDATE`触发器使用`NOW()`来获取行修改时间；`ts`列会自动初始化为当前日期和时间，但`NEW.ts`不会包含该值。

假设拍卖以五美元的初始出价创建：

```
mysql> `INSERT INTO auction (item,bid) VALUES('chintz pillows',5.00);`
mysql> `SELECT LAST_INSERT_ID();`
+------------------+
| LAST_INSERT_ID() |
+------------------+
|              792 |
+------------------+
```

`SELECT`语句获取拍卖 ID 值，用于后续对拍卖的操作。然后物品在拍卖结束前接收到另外三个出价，并被移除：

```
mysql> `UPDATE auction SET bid = 7.50 WHERE id = 792;`
*`... time passes ...`*
mysql> `UPDATE auction SET bid = 9.00 WHERE id = 792;`
*`... time passes ...`*
mysql> `UPDATE auction SET bid = 10.00 WHERE id = 792;`
*`... time passes ...`*
mysql> `DELETE FROM auction WHERE id = 792;`
```

此时，在`auction`表中不再有拍卖的痕迹，但是`auction_log`表中包含了发生的所有操作历史：

```
mysql> `SELECT * FROM auction_log WHERE id = 792 ORDER BY ts;`
+--------+-----+---------------------+----------------+-------+
| action | id  | ts                  | item           | bid   |
+--------+-----+---------------------+----------------+-------+
| create | 792 | 2014-01-09 14:57:41 | chintz pillows |  5.00 |
| update | 792 | 2014-01-09 14:57:50 | chintz pillows |  7.50 |
| update | 792 | 2014-01-09 14:57:57 | chintz pillows |  9.00 |
| update | 792 | 2014-01-09 14:58:03 | chintz pillows | 10.00 |
| delete | 792 | 2014-01-09 14:58:03 | chintz pillows | 10.00 |
+--------+-----+---------------------+----------------+-------+
```

使用刚刚概述的策略，`auction`表保持相对较小，您始终可以通过查看`auction_log`表来找到必要的拍卖历史信息。

# 11.5 使用事件调度数据库操作

## 问题

您希望设置一个定期运行的数据库操作，无需用户干预。

## 解决方案

MySQL 提供了一个事件调度器，使您能够设置在您定义的时间运行的数据库操作。创建一个根据时间表执行的事件。

## 讨论

本节描述了使用事件所必须做的事情，从一个简单的事件开始，以便在规则间隔时间内向表中写入一行。

从一个表开始来保存标记行。它包含一个`TIMESTAMP`列（MySQL 会自动初始化），还有一个用来存储消息的列：

```
CREATE TABLE mark_log
(
  id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  ts      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  message VARCHAR(100)
);
```

我们的日志事件将会向新行写入一个字符串。要设置它，使用`CREATE EVENT`语句：

```
CREATE EVENT mark_insert
ON SCHEDULE EVERY 5 MINUTE
DO INSERT INTO mark_log (message) VALUES('-- MARK --');
```

`mark_insert`事件导致消息`'-- MARK --'`每五分钟记录到`mark_log`表中。为了更频繁或更少频繁地记录，可以使用不同的间隔。

此事件很简单，其主体仅包含一个单个 SQL 语句。对于执行多个语句的事件主体，请使用`BEGIN`…`END`复合语句语法。在这种情况下，如果您使用*mysql*来创建事件，请在定义事件时更改语句分隔符，如 Recipe 11.1 所讨论的那样。

此时，您应该等待几分钟，然后选择`mark_log`表的内容，以验证是否按计划写入了新行。但是，如果这是您设置的第一个事件，无论等待多长时间，该表可能仍然为空：

```
mysql> `SELECT * FROM mark_log;`
Empty set (0.00 sec)
```

如果情况如此，很可能事件调度器未运行（这是其默认状态，直到版本 8.0）。通过检查`event_scheduler`系统变量的值来检查调度器状态：

```
mysql> `SHOW VARIABLES LIKE 'event_scheduler';`
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| event_scheduler | OFF   |
+-----------------+-------+
```

如果调度器未运行，可以通过执行以下语句来交互地启用它（这需要`SYSTEM_VARIABLES_ADMIN`或在版本 8.0 之前需要`SUPER`权限）：

```
SET GLOBAL event_scheduler = 1;
```

那个语句启用调度器，但仅在服务器关闭之前。要在每次服务器启动时启动调度器，请在您的*my.cnf*选项文件中启用系统变量：

```
[mysqld]
event_scheduler=1
```

或使用`SET PERSIST`语句存储变量的修改值：

```
SET PERSIST event_scheduler = 1;
```

当事件调度器被启用时，`mark_insert`事件最终会在表中创建许多行。有几种方法可以影响事件执行，以防止表无限增长：

+   删除事件：

    ```
    DROP EVENT mark_insert;
    ```

    这是停止事件发生的最简单方法。但是，如果你希望稍后恢复它，必须重新创建它。

+   禁用事件执行：

    ```
    ALTER EVENT mark_insert DISABLE;
    ```

    这样会保留事件但导致其不运行，直到您重新激活它：

    ```
    ALTER EVENT mark_insert ENABLE;
    ```

+   让事件继续运行，但设置另一个事件来`<q>过期</q>`旧的`mark_log`行。这第二个事件不需要运行得如此频繁（也许一天运行一次）。其主体应删除超过给定阈值的旧行。以下定义创建一个事件，删除两天以上的行：

    ```
    CREATE EVENT mark_expire
    ON SCHEDULE EVERY 1 DAY
    DO DELETE FROM mark_log WHERE ts < NOW() - INTERVAL 2 DAY;
    ```

    如果采用这种策略，您有协作事件：一个事件向 `mark_log` 表添加行，另一个事件将其移除。它们共同维护一个包含最近行但不会变得过大的日志。

# 11.6 为执行动态 SQL 编写辅助例程

## 问题

准备好的 SQL 语句使您能够即兴构造和执行 SQL 语句，但您希望在一个步骤中运行它们，而不是执行三个命令：`PREPARE`、`EXECUTE` 和 `DEALLOCATE PREPARE`。

## 解决方案

编写一个处理单调工作的辅助过程。

## 讨论

使用准备的 SQL 语句涉及三个步骤：准备、执行和释放。例如，如果 `@tbl_name` 和 `@val` 变量分别保存表名和要插入表中的值，您可以这样创建表并插入值：

```
SET @stmt = CONCAT('CREATE TABLE ',@tbl_name,' (i INT)');
PREPARE stmt FROM @stmt;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
SET @stmt = CONCAT('INSERT INTO ',@tbl_name,' (i) VALUES(',@val,')');
PREPARE stmt FROM @stmt;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

为了简化为每个动态创建的语句执行这些步骤的负担，使用一个辅助例程，它接受一个语句字符串，准备、执行和释放它：

```
CREATE PROCEDURE exec_stmt(stmt_str TEXT)
BEGIN
  SET @_stmt_str = stmt_str;
  PREPARE stmt FROM @_stmt_str;
  EXECUTE stmt;
  DEALLOCATE PREPARE stmt;
END;
```

`exec_stmt()` 例程使得执行相同的语句变得更加简单：

```
CALL exec_stmt(CONCAT('CREATE TABLE ',@tbl_name,' (i INT)'));
CALL exec_stmt(CONCAT('INSERT INTO ',@tbl_name,' (i) VALUES(',@val,')'));
```

`exec_stmt()` 使用一个中间的用户定义变量 `@_stmt_str`，因为 `PREPARE` 只接受一个文字字符串或用户定义变量指定的语句。存储在例程参数中的语句不起作用。（至少在期望其值跨 `exec_stmt()` 调用持续时，避免将 `@_stmt_str` 用于您自己的目的。）

现在，怎样才能更安全地构造包含可能来自外部源（如 Web 表单输入或命令行参数）的值的语句字符串？这些信息不能信任，应视为潜在的 SQL 注入攻击向量：

+   `QUOTE()` 函数用于引用数据值。

+   对于标识符没有相应的函数，但很容易编写一个函数，将内部反引号加倍，并在开头和结尾添加一个反引号：

    ```
    CREATE FUNCTION quote_identifier(id TEXT)
    RETURNS TEXT DETERMINISTIC
    RETURN CONCAT('`',REPLACE(id,'`','``'),'`');
    ```

修改前述示例以确保数据值和标识符的安全性，我们有：

```
SET @tbl_name = quote_identifier(@tbl_name);
SET @val = QUOTE(@val);
CALL exec_stmt(CONCAT('CREATE TABLE ',@tbl_name,' (i INT)'));
CALL exec_stmt(CONCAT('INSERT INTO ',@tbl_name,' (i) VALUES(',@val,')'));
```

使用 `exec_stmt()` 的限制是并非所有的 SQL 语句都可以作为准备语句执行。有关限制，请参阅*MySQL 参考手册*。

# 11.7 使用条件处理程序检测<q>没有更多的行</q>条件

## 问题

您希望检测到<q>没有更多的行</q>条件，并优雅地处理它们，而不是中断存储程序的执行。

## 解决方案

条件处理程序的一个常见用途是检测<q>没有更多的行</q>条件。为了一次处理一个查询结果行，请与捕获数据结束条件的条件处理程序结合使用基于游标的取行循环。该技术具有这些基本元素：

+   与读取行的 `SELECT` 语句相关联的游标。打开游标开始读取，关闭游标停止。

+   当游标到达结果集末尾并引发数据末尾条件（`NOT FOUND`）时激活的条件处理程序。我们在 Recipe 11.2 中使用了类似的处理程序。

+   一个指示循环终止的变量。将该变量初始化为`FALSE`，然后在条件处理程序中，当出现数据末尾条件时将其设置为`TRUE`。

+   使用游标的循环，逐行抓取并在循环终止变量变为`TRUE`时退出。

## 讨论

以下示例实现了一个抓取循环，逐行处理 _ch `states`表以计算总的美国人口：

```
CREATE PROCEDURE us_population()
BEGIN
  DECLARE done BOOLEAN DEFAULT FALSE; ![1](img/1.png)
  DECLARE state_pop, total_pop BIGINT DEFAULT 0;
  DECLARE cur CURSOR FOR SELECT pop FROM states; ![2](img/2.png)
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE; ![3](img/3.png)

  OPEN cur;
  fetch_loop: LOOP
    FETCH cur INTO state_pop; ![4](img/4.png)
    IF done THEN ![5](img/5.png)
      LEAVE fetch_loop;
    END IF;
    SET total_pop = total_pop + state_pop; ![6](img/6.png)
  END LOOP;
  CLOSE cur;
  SELECT total_pop AS 'Total U.S. Population'; ![7](img/7.png)
END;
```

![1](img/#co_nch-routines-routines-end-of-data_done_co)

变量`done`用作标志，当过程决定是否继续执行或停止时进行检查。

![2](img/#co_nch-routines-routines-end-of-data_query_co)

用于抓取每个州人口的查询的游标。

![3](img/#co_nch-routines-routines-end-of-data_ch_co)

当 MySQL 遇到未找到错误时，它会停止执行。为了防止这种情况，我们声明了一个`CONTINUE`处理程序，该处理程序将变量`done`的值设置为`TRUE`。

![4](img/#co_nch-routines-routines-end-of-data_fetch_co)

将每个州的人口抓取到变量`state_pop`中。

![5](img/#co_nch-routines-routines-end-of-data_leave_co)

如果变量`done`不为真，则继续循环，否则退出循环。

![6](img/#co_nch-routines-routines-end-of-data_total_co)

我们将变量`state_pop`的值添加到代表美国人口的变量`total_pop`中。

![7](img/#co_nch-routines-routines-end-of-data_result_co)

离开循环后，打印变量`total_pop`的值。

虽然此示例主要用于说明，在任何实际应用程序中，您将使用聚合函数来计算总数。但这也为我们提供了一个独立的检查，以确定抓取循环是否计算了正确的值：

```
mysql> `CALL us_population();`
+-----------------------+
| Total U.S. Population |
+-----------------------+
|             331223695 |
+-----------------------+
mysql> `SELECT SUM(pop) AS 'Total U.S. Population' FROM states;`
+-----------------------+
| Total U.S. Population |
+-----------------------+
|             331223695 |
+-----------------------+
```

`NOT FOUND`处理程序也非常有用，用于检查`SELECT ... INTO *var_name*`语句是否返回任何结果。Recipe 11.2 显示了一个示例。

# 11.8 捕获并忽略带条件处理程序的错误

## 问题

您希望忽略良性错误或防止不存在用户的错误。

## 解决方案

使用条件处理程序捕获和处理您想要忽略的错误。

## 讨论

如果您认为错误是良性的，可以使用处理程序来忽略它。例如，MySQL 中的许多`DROP`语句都有一个`IF EXISTS`子句，以便在要删除的对象不存在时抑制错误。但有些`DROP`语句没有这样的子句，因此无法抑制错误。`DROP INDEX`就是其中之一：

```
mysql> `DROP INDEX bad_index ON limbs;`
ERROR 1091 (42000): Can't DROP 'bad_index'; check that column/key exists
```

为了防止针对不存在用户出现错误，请在捕获代码 1091 并忽略它的存储过程中调用`DROP INDEX`：

```
CREATE PROCEDURE drop_index(index_name VARCHAR(64), table_name VARCHAR(64))
BEGIN
  DECLARE CONTINUE HANDLER FOR 1091
    SELECT CONCAT('Unknown index: ', index_name) AS Message;
  CALL exec_stmt(CONCAT('DROP INDEX ', index_name, ' ON ', table_name));
END;
```

如果索引不存在，`drop_index()`在条件处理程序中写入消息，但不会发生错误：

```
mysql> `CALL drop_index('bad_index', 'limbs');`
+--------------------------+
| Message                  |
+--------------------------+
| Unknown index: bad_index |
+--------------------------+
```

要完全忽略错误，请使用空的`BEGIN` … `END`块编写处理程序：

```
DECLARE CONTINUE HANDLER FOR 1091 BEGIN END;
```

另一种方法是生成警告，如下一个示例所示。

# 11.9 抛出错误和警告

## 问题

您希望对语句引发错误，对 MySQL 有效但不适用于您正在处理的应用程序。

## 解决方案

在检测到异常情况时，在存储程序内部生成自己的错误，请使用`SIGNAL`语句。

## 讨论

本示例展示了一些示例，而第 11.11 节展示了在触发器中使用`SIGNAL`拒绝错误数据的用法。

假设一个应用程序执行除法操作，您期望除数永远不为零，否则您希望引发错误。 您可能希望自从版本 5.7.4 起，默认启用 SQL 模式`ERROR_FOR_DIVISION_BY_ZERO`将自动执行此行为。 但是，仅在数据修改操作（例如`INSERT`）的上下文中，除零会产生警告：

```
mysql> `SELECT @@sql_mode\G`
*************************** 1\. row ***************************
@@sql_mode: ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,↩
            ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
1 row in set (0,00 sec)

mysql> `SELECT 1/0;`
+------+
| 1/0  |
+------+
| NULL |
+------+
1 row in set, 1 warning (0.00 sec)
mysql> `SHOW WARNINGS;`
+---------+------+---------------+
| Level   | Code | Message       |
+---------+------+---------------+
| Warning | 1365 | Division by 0 |
+---------+------+---------------+
```

要确保在任何情况下出现除零错误，请编写一个函数执行除法，但首先检查除数，并使用`SIGNAL`在出现“无法发生”条件时引发错误：

```
CREATE FUNCTION divide(numerator FLOAT, divisor FLOAT)
RETURNS FLOAT DETERMINISTIC
BEGIN
  IF divisor = 0 THEN
    SIGNAL SQLSTATE '22012'
           SET MYSQL_ERRNO = 1365, MESSAGE_TEXT = 'unexpected 0 divisor';
  END IF;
  RETURN numerator / divisor;
END;
```

在非修改上下文中测试该函数，以验证它是否会产生错误：

```
mysql> `SELECT divide(1,0);`
ERROR 1365 (22012): unexpected 0 divisor
```

`SIGNAL`语句指定了 SQLSTATE 值以及一个可选的`SET`子句，您可以使用它来分配错误属性的值。 `MYSQL_ERRNO`对应于 MySQL 特定的错误代码，而`MESSAGE_TEXT`是您选择的字符串。

`SIGNAL`还可以引发警告条件，而不仅仅是错误。 下面的例程`drop_user_warn()`类似于之前显示的`drop_user()`例程，但是不会为不存在的用户打印消息，而是生成一个警告，可以使用`SHOW` `WARNINGS`显示。 SQLSTATE 值`01000`和错误 1642 指示用户定义的未处理异常，该例程会发出该异常，并附上适当的消息：

```
CREATE PROCEDURE drop_user_warn(user TEXT, host TEXT)
BEGIN
  DECLARE account TEXT;
  DECLARE CONTINUE HANDLER FOR 1396
  BEGIN
    DECLARE msg TEXT;
    SET msg = CONCAT('Unknown user: ', account);
    SIGNAL SQLSTATE '01000' SET MYSQL_ERRNO = 1642, MESSAGE_TEXT = msg;
  END;
  SET account = CONCAT(QUOTE(user),'@',QUOTE(host));
  CALL exec_stmt(CONCAT('DROP USER ',account));
END;
```

进行测试：

```
mysql> `CALL drop_user_warn('bad-user','localhost');`
Query OK, 0 rows affected, 1 warning (0.00 sec)
mysql> `SHOW WARNINGS;`
+---------+------+--------------------------------------+
| Level   | Code | Message                              |
+---------+------+--------------------------------------+
| Warning | 1642 | Unknown user: 'bad-user'@'localhost' |
+---------+------+--------------------------------------+
```

# 11.10 通过访问诊断区域记录错误

## 问题

您希望记录存储过程遇到的所有错误。

## 解决方案

使用*GET DIAGNOSTICS*语句访问诊断区域。 然后，将错误信息保存到变量中，并使用它们来记录错误。

## 讨论

您不仅可以在存储过程内部优雅地处理错误，还可以记录这些错误，以便查看并修复应用程序，以防止将来出现类似的错误。

表`movies_actors_link`在配方 Recipe 16.6 中用于演示多对多关系。它包含存储在表`movies`和`movies_actors`中的电影和电影演员的`id`。这两列都使用属性`NOT NULL`定义。每个`movie_id`和`actor_id`的组合应该是唯一的。虽然 Recipe 16.6 没有定义外键（“使用外键强制执行引用完整性并防止不匹配”），但我们可以定义它们，这样 MySQL 会拒绝没有对应条目的值。

```
ALTER TABLE movies_actors_link ADD FOREIGN KEY(movie_id) REFERENCES movies(id);
ALTER TABLE movies_actors_link ADD FOREIGN KEY(actor_id) REFERENCES actors(id);
```

当我们在*mysql*客户端执行语句时。

```
mysql> `INSERT` `INTO` `movies_actors_link` `VALUES``(``7``,` `1``)``;`
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint 
fails (`cookbook`.`movies_actors_link`, CONSTRAINT `movies_actors_link_ibfk_1` 
FOREIGN KEY (`movie_id`) REFERENCES `movies` (`id`))
```

另外，MySQL 提供对诊断区域的访问，因此您可以将其值存储在用户定义的变量中。使用命令*GET DIAGNOSTICS*来访问诊断区域。

```
mysql>  `GET` `DIAGNOSTICS` `CONDITION` `1`
    ->  `@``err_number` `=` `MYSQL_ERRNO``,`
    ->  `@``err_sqlstate` `=` `RETURNED_SQLSTATE``,`
    ->  `@``err_message` `=` `MESSAGE_TEXT``;`
Query OK, 0 rows affected (0.01 sec)
```

条件`CONDITION`指定条件号码。我们的查询仅返回一个条件，因此我们使用编号 1。如果查询返回多个条件，诊断区域将包含每个条件的数据。例如，如果查询生成多个警告，则可能会发生这种情况。

要访问由*GET DIAGNOSTICS*检索的数据，只需选择用户定义变量的值。

```
mysql> `SELECT` `@``err_number``,` `@``err_sqlstate``,` `@``err_message``\``G`
*************************** 1. row ***************************
  @err_number: 1452
@err_sqlstate: 23000
 @err_message: Cannot add or update a child row: a foreign key constraint 
               fails (`cookbook`.`movies_actors_link`, 
               CONSTRAINT `movies_actors_link_ibfk_1` 
               FOREIGN KEY (`movie_id`) REFERENCES `movies` (`id`))
1 row in set (0.00 sec)
```

记录所有这些错误，当用户插入数据到表`movies_actors_link`时，创建一个过程，它接受两个参数：`movie_id`和`actor_id`，并将错误信息存储在日志表中。

首先创建将存储有关错误信息的表。

```
CREATE TABLE `movies_actors_log` (
  `err_ts` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `err_number` int DEFAULT NULL,
  `err_sqlstate` char(5) DEFAULT NULL,
  `err_message` TEXT DEFAULT NULL,
  `movie_id` int unsigned DEFAULT NULL,
  `actor_id` int unsigned DEFAULT NULL
);
```

然后定义一个过程，将向表`movies_actors_link`插入一行，并在出现错误时将详细信息记录到表`movies_actors_log`中。

```
CREATE PROCEDURE insert_movies_actors_link(movie INT, actor INT)
BEGIN
  DECLARE e_number INT; ![1](img/1.png)
  DECLARE e_sqlstate CHAR(5);
  DECLARE e_message TEXT;

  DECLARE CONTINUE HANDLER FOR SQLEXCEPTION ![2](img/2.png)
    BEGIN
      GET DIAGNOSTICS CONDITION 1
        e_number = MYSQL_ERRNO, ![3](img/3.png)
        e_sqlstate = RETURNED_SQLSTATE,
        e_message = MESSAGE_TEXT;
      INSERT INTO movies_actors_log(err_number, err_sqlstate, err_message,  ![4](img/4.png)
                                    movie_id, actor_id)
        VALUES(e_number, e_sqlstate, e_message, movie, actor);
      RESIGNAL; ![5](img/5.png)
    END;

  INSERT INTO movies_actors_link VALUES(movie, actor); ![6](img/6.png)
END
```

![1](img/#co_nch-routines-routines-diagnostic-area-gd_declare_co)

声明变量，用于存储错误编号、SQLSTATE 和错误消息。

![2](img/#co_nch-routines-routines-diagnostic-area-gd_handler_co)

为`SQLEXCEPTION`创建一个`CONTINUE HANDLER`，因此过程首先记录错误，然后继续执行。

![3](img/#co_nch-routines-routines-diagnostic-area-gd_store_co)

将诊断信息存储在变量中。

![4](img/#co_nch-routines-routines-diagnostic-area-gd_log_co)

记录有关错误的详细信息到表`movies_actors_log`中。

![5](img/#co_nch-routines-routines-diagnostic-area-gd_resignal_co)

使用命令*RESIGNAL*来为调用过程的客户端引发错误。

![6](img/#co_nch-routines-routines-diagnostic-area-gd_insert_co)

*INSERT*到表`movies_actors_link`，可能成功，也可能引发错误。

为了测试过程，用不同的参数调用它几次。

```
mysql>  `CALL` `insert_movies_actors_link``(``7``,` `11``)``;`
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint 
fails (`cookbook`.`movies_actors_link`, CONSTRAINT `movies_actors_link_ibfk_1` 
FOREIGN KEY (`movie_id`) REFERENCES `movies` (`id`))
mysql>  `CALL` `insert_movies_actors_link``(``6``,` `11``)``;`
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint 
fails (`cookbook`.`movies_actors_link`, CONSTRAINT `movies_actors_link_ibfk_2` 
FOREIGN KEY (`actor_id`) REFERENCES `actors` (`id`))
mysql>  `CALL` `insert_movies_actors_link``(``null``,` `10``)``;`
ERROR 1048 (23000): Column 'movie_id' cannot be null
mysql>  `CALL` `insert_movies_actors_link``(``6``,` `null``)``;`
ERROR 1048 (23000): Column 'actor_id' cannot be null
mysql>  `CALL` `insert_movies_actors_link``(``6``,` `9``)``;`
ERROR 1062 (23000): Duplicate entry '6-9' for key 'movies_actors_link.movie_id'
```

预期地，因为我们使用了 `RESIGNAL`，该过程失败并显示错误。所有错误均已记录在 `movies_actors_log` 表中，连同我们尝试但未能插入的值及其尝试发生的时间戳。

```
mysql> `SELECT` `*` `FROM` `movies_actors_log``\``G`
*************************** 1. row ***************************
      err_ts: 2021-03-12 21:11:30
  err_number: 1452
err_sqlstate: 23000
 err_message: Cannot add or update a child row: a foreign key constraint fails 
              (`cookbook`.`movies_actors_link`, 
              CONSTRAINT `movies_actors_link_ibfk_1` 
              FOREIGN KEY (`movie_id`) REFERENCES `movies` (`id`))
    movie_id: 7
    actor_id: 11
*************************** 2. row ***************************
      err_ts: 2021-03-12 21:11:38
  err_number: 1452
err_sqlstate: 23000
 err_message: Cannot add or update a child row: a foreign key constraint fails 
              (`cookbook`.`movies_actors_link`, 
              CONSTRAINT `movies_actors_link_ibfk_2` 
              FOREIGN KEY (`actor_id`) REFERENCES `actors` (`id`))
    movie_id: 6
    actor_id: 11
*************************** 3. row ***************************
      err_ts: 2021-03-12 21:11:49
  err_number: 1048
err_sqlstate: 23000
 err_message: Column 'movie_id' cannot be null
    movie_id: NULL
    actor_id: 10
*************************** 4. row ***************************
      err_ts: 2021-03-12 21:11:56
  err_number: 1048
err_sqlstate: 23000
 err_message: Column 'actor_id' cannot be null
    movie_id: 6
    actor_id: NULL
*************************** 5. row ***************************
      err_ts: 2021-03-12 21:12:00
  err_number: 1062
err_sqlstate: 23000
 err_message: Duplicate entry '6-9' for key 'movies_actors_link.movie_id'
    movie_id: 6
    actor_id: 9
5 rows in set (0.00 sec)
```

## 参见

有关诊断区域的更多信息，请参阅 [GET DIAGNOSTICS 语句](https://dev.mysql.com/doc/refman/8.0/en/get-diagnostics.html)。

# 11.11 使用触发器预处理或拒绝数据

## 问题

有些条件您希望检查插入表中的数据，但又不想为每个 `INSERT` 编写验证逻辑。

## 解决方案

将输入测试逻辑集中到 `BEFORE INSERT` 触发器中。

## 讨论

您可以使用触发器执行多种类型的输入检查：

+   通过引发信号来拒绝不良数据。这使您可以访问存储的程序逻辑，以便在检查值方面比使用静态约束（如 `NOT NULL`）更加宽松。

+   预处理值并修改它们，如果您不想直接拒绝它们的话。例如，将超出范围的值映射到范围内，或者清理值以将其放置为规范形式，如果允许输入接近的变体。

假设您有一个包含姓名、居住州、电子邮件地址和网站 URL 的联系信息表：

```
CREATE TABLE contact_info
(
  id     INT NOT NULL AUTO_INCREMENT,
  name   VARCHAR(30),   # name of person
  state  CHAR(2),       # state of residence
  email  VARCHAR(50),   # email address
  url    VARCHAR(255),  # web address
  PRIMARY KEY (id)
);
```

对于新行的输入，您希望强制执行约束或执行预处理如下：

+   居住州的值是两位字母的美国州代码，仅当其存在于 `states` 表中时才有效。（在这种情况下，您可以将列声明为具有 50 个成员的 `ENUM`，因此更有可能使用此查找表技术，用于列的有效值集合任意大或随时间变化的情况。）

+   电子邮件地址的值必须包含 `@` 字符才有效。

+   对于网站 URL，去掉任何前导的 `http://` 或 `https://` 以节省空间。

为了处理这些要求，创建一个 `BEFORE INSERT` 触发器：

```
CREATE TRIGGER bi_contact_info BEFORE INSERT ON contact_info
FOR EACH ROW
BEGIN
  IF (SELECT COUNT(*) FROM states WHERE abbrev = NEW.state) = 0 THEN
    SIGNAL SQLSTATE 'HY000'
           SET MYSQL_ERRNO = 1525, MESSAGE_TEXT = 'invalid state code';
  END IF;
  IF INSTR(NEW.email,'@') = 0 THEN
    SIGNAL SQLSTATE 'HY000'
           SET MYSQL_ERRNO = 1525, MESSAGE_TEXT = 'invalid email address';
  END IF;
  SET NEW.url = TRIM(LEADING 'http://' FROM NEW.url);
  SET NEW.url = TRIM(LEADING 'https://' FROM NEW.url);
END;
```

要同时处理更新，可以定义一个与 `bi_contact_info` 相同体的 `BEFORE UPDATE` 触发器。

通过执行一些 `INSERT` 语句来测试触发器，以验证它接受有效值，拒绝不良值并修剪 URL：

```
mysql> `INSERT INTO contact_info (name,state,email,url)`
    -> `VALUES('Jen','NY','jen@example.com','http://www.example.com');`
mysql> `INSERT INTO contact_info (name,state,email,url)`
    -> `VALUES('Jen','XX','jen@example.com','http://www.example.com');`
ERROR 1525 (HY000): invalid state code
mysql> `INSERT INTO contact_info (name,state,email,url)`
    -> `VALUES('Jen','NY','jen','http://www.example.com');`
ERROR 1525 (HY000): invalid email address
mysql> `SELECT * FROM contact_info;`
+----+------+-------+-----------------+-----------------+
| id | name | state | email           | url             |
+----+------+-------+-----------------+-----------------+
|  1 | Jen  | NY    | jen@example.com | www.example.com |
+----+------+-------+-----------------+-----------------+
```
