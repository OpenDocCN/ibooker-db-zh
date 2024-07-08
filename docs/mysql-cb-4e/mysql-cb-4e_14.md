# 第十四章：验证和重新格式化数据

# 14.0 介绍

前一章，第十三章，侧重于通过读取行并将其分解为单独列来将数据移入和移出 MySQL 的方法。在本章中，我们将重点放在内容而不是结构问题上。例如，如果您不知道文件中包含的值或通过 Web 表单接收的值是否合法，请预处理它们以进行检查或重新格式化：

+   验证数据值通常是一个好主意，以确保它们对于存储它们的数据类型是合法的。例如，您可以确保用于 `INT`、`DATE` 和 `ENUM` 列的值分别是整数、ISO 格式的日期（*`YYYY-MM-DD`*）和合法的枚举值。

+   数据值可能需要重新格式化。您可能将信用卡的值存储为一串数字，但允许 Web 应用程序的用户通过空格或破折号分隔数字块。这些值在存储之前必须重新编写。将日期从一种格式重写为另一种格式是非常常见的；例如，如果程序以*`MM-DD-YY`*格式编写日期以便导入到 MySQL 中的 ISO 格式。如果程序只理解日期和时间格式而不理解组合的日期和时间格式（例如 MySQL 用于 `DATETIME` 和 `TIMESTAMP` 数据类型的格式），则必须将日期和时间值拆分为单独的日期和时间值。

本章主要涉及在检查整个文件的上下文中的格式和验证问题，但这里讨论的许多技术也可以应用于一次性验证。考虑一个基于 Web 的应用程序，该应用程序提供一个表单供用户填写，然后处理其内容以在数据库中创建新行。 Web API 通常将表单内容作为一组已解析的离散值提供，因此应用程序可能不需要处理记录和列分隔符。另一方面，验证问题仍然至关重要。您真的不知道用户向您的脚本发送什么样的值，因此检查它们很重要。

前三个配方介绍了 MySQL 中可用的数据验证功能。从配方 14.4 开始，我们将重点放在验证和预处理应用程序端的数据上。我们介绍了一些技巧，可以有效地处理大量数据。

本章讨论的程序片段和脚本的源代码位于 `recipes` 分发的 *transfer* 目录中，但一些实用函数包含在位于 *lib* 目录中的库文件中。

# 14.1 使用 SQL 模式拒绝不良输入值

## 问题

MySQL 接受无效、超出范围或其他不适合插入的列的数据值。您希望服务器更具限制性，不接受不良数据。

## 解决方案

检查 SQL 模式并确保它不为空。有几种模式可以用来控制服务器对数据值的严格程度。有些模式适用于所有输入值，而其他模式适用于特定的数据类型，如日期。

## 讨论

当 SQL 模式未设置或设置为空值时，MySQL 允许所有输入值用于表列，即使输入数据类型与列的数据类型不匹配。考虑以下表，其中包含整数、字符串和日期列：

```
mysql> `SELECT @@sql_mode;`
+------------+
| @@sql_mode |
+------------+
|            |
+------------+
1 row in set (0,00 sec)
mysql> `CREATE TABLE t (i INT, c CHAR(6), d DATE);`
```

向表中插入具有不合适数据值的行会导致警告（可以通过`SHOW WARNINGS`查看），但服务器会将这些值加载到表中，并将它们转换为适合该列的某个值：

```
mysql> `INSERT INTO t (i,c,d) VALUES('-1x','too-long string!','1999-02-31');`
mysql> `SHOW WARNINGS;`
+---------+------+--------------------------------------------+
| Level   | Code | Message                                    |
+---------+------+--------------------------------------------+
| Warning | 1265 | Data truncated for column 'i' at row 1     |
| Warning | 1265 | Data truncated for column 'c' at row 1     |
| Warning | 1264 | Out of range value for column 'd' at row 1 |
+---------+------+--------------------------------------------+
mysql> `SELECT * FROM t;`
+------+--------+------------+
| i    | c      | d          |
+------+--------+------------+
|   -1 | too-lo | 0000-00-00 |
+------+--------+------------+
```

防止这些转换发生的一种方法是在客户端检查输入数据，以确保其合法。在某些情况下，这是一种合理的策略（参见 Recipe 14.0 中的侧边栏），但也有一种替代方案：让服务器在服务器端检查数据值，并在无效时拒绝它们并显示错误。

要做到这一点，需要将`sql_mode`系统变量设置为启用服务器对输入数据的限制接受。通过适当的限制，否则会导致转换和警告的数据值将导致错误。在启用<q>strict</q> SQL 模式后，再次尝试上一个示例中的`INSERT`语句：

```
mysql> `SET sql_mode = 'STRICT_ALL_TABLES';`
mysql> `INSERT INTO t (i,c,d) VALUES('-1x','too-long string!','1999-02-31');`
ERROR 1265 (01000): Data truncated for column 'i' at row 1
```

这里的语句甚至没有进展到第二和第三个数据值，因为第一个对于整数列来说是无效的，服务器会报错。

如果未启用输入限制，服务器将检查日期值的月份部分是否在 1 到 12 的范围内，并且日期值对于给定的月份是否合法。这意味着`'2005-02-31'`默认情况下会生成警告（转换为零日期`'0000-00-00'`）。在严格模式下，会发生错误。

MySQL 仍然允许诸如`'1999-11-00'`或`'1999-00-00'`这样具有零部分的日期，或者称为<q>零</q>日期（`'0000-00-00'`）。为了限制这些类型的日期值，启用`NO_ZERO_IN_DATE`和`NO_ZERO_DATE` SQL 模式会导致警告，或在严格模式下导致错误。例如，要禁止具有零部分或<q>零</q>日期的日期，请像这样设置 SQL 模式：

```
mysql> `SET sql_mode = 'STRICT_ALL_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE';`
```

启用这些限制的更简单方法，以及其他几种方法，是启用`TRADITIONAL` SQL 模式。`TRADITIONAL`模式实际上是一组模式，可以通过设置和显示`sql_mode`值来查看：

```
mysql> `SET sql_mode = 'TRADITIONAL';`
mysql> `SELECT @@sql_mode\G`
*************************** 1\. row ***************************
@@sql_mode: STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ZERO_IN_DATE,
            NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,TRADITIONAL,
            NO_ENGINE_SUBSTITUTION
```

您可以在[MySQL 参考手册](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html)中详细了解各种 SQL 模式。

所示的示例设置了`sql_mode`系统变量的会话值，因此它们仅更改当前会话的 SQL 模式。要为所有客户端全局设置模式，请在启动服务器时使用`--sql_mode=`*`mode_value`*选项。或者，如果具有`SYSTEM_VARIABLES_ADMIN`或`SUPER`特权，则可以在运行时设置全局模式。

```
mysql> `SET GLOBAL sql_mode = '`*`mode_value`*`';`
```

###### 注意

在 MySQL 5.7 之前，默认情况下 SQL 模式是宽容的。更新的版本更加严格，SQL 模式设置为 `ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES, NO_ZERO_IN_DATE, NO_ZERO_DATE, ERROR_FOR_DIVISION_BY_ZERO, NO_AUTO_CREATE_USER, NO_ENGINE_SUBSTITUTION`。因此，如果您希望拥有严格的服务器，您不需要额外做任何事情，除非您有意放宽了之前的 SQL 模式。

# 14.2 使用 `CHECK` 约束拒绝无效值

## 问题

您希望验证数据以便符合应用程序的业务逻辑，并在不满足要求时拒绝值。

## 解决方案

使用 `CHECK` 约束。

## 讨论

如果一个值与 MySQL 数据类型格式匹配，并不意味着它与应用程序的逻辑匹配。例如，如果您只想存储偶数，您不能简单地使用整数数据类型，因为奇数和偶数都是有效的整数。

`CHECK` 约束在 8.0 版本中引入，允许在表列上设置自定义条件，并在值不满足条件时拒绝语句。因此，要创建一个只存储偶数的表，您需要使用 `CHECK` 来检查数字是否可以被二整除。

```
mysql> `CREATE` `TABLE` `even` `(`
    -> `even_value` `INT` `CHECK``(``even_value` `%` `2` `=` `0``)`
    -> `)` `ENGINE``=``InnoDB``;`
Query OK, 0 rows affected (0.03 sec)
```

现在，我们可以成功地插入偶数到这个表中。

```
mysql> `INSERT` `INTO` `even` `VALUES``(``2``)``;`
Query OK, 1 row affected (0.01 sec)
```

奇数值将被拒绝。

```
mysql> `INSERT` `INTO` `even` `VALUES``(``1``)``;`
ERROR 3819 (HY000): Check constraint 'even_chk_1' is violated.
```

您还可以为单个列创建多个 `CHECK` 约束。例如，为了仅接受小于 100 的偶数值，创建两个约束。

```
mysql> `CREATE` `TABLE` `even_100` `(`
    -> `even_value` `INT` `CHECK``(``even_value` `%` `2` `=` `0``)` `CHECK``(``even_value` `<` `100``)`
    -> `)` `ENGINE``=``InnoDB``;`
Query OK, 0 rows affected (0.02 sec)
```

在这种情况下，MySQL 将检查第一个条件，如果满足则处理第二个条件。

```
mysql> `INSERT` `INTO` `even_100` `VALUES``(``101``)``;`
ERROR 3819 (HY000): Check constraint 'even_100_chk_1' is violated.
mysql> `INSERT` `INTO` `even_100` `VALUES``(``102``)``;`
ERROR 3819 (HY000): Check constraint 'even_100_chk_2' is violated.
```

如果在定义列时指定 `CHECK` 约束，它将仅验证此列。如果您希望在单个约束中检查两个或更多列，则需要单独指定它。

常见的验证任务是检查出发日期是否晚于到达日期。我们可以将此检查添加到 `patients` 表中。

```
ALTER TABLE patients ADD CONSTRAINT date_check 
CHECK((date_departed IS NULL) OR (date_departed >= date_arrived));
```

现在，它将不允许插入出发日期早于到达日期的记录。

```
mysql> `INSERT` `INTO` `patients` `(``national_id``,` `name``,` `surname``,` `gender``,` `age``,` `diagnosis``,` 
    -> `date_arrived``,` `date_departed``)`
    -> `VALUES``(``'34GD429520'``,` `'John'``,` `'Doe'``,` `'M'``,` `45``,` `'Data Phobia'``,`
    -> `'2020-07-20'``,` `'2020-05-31'``)``;`
ERROR 3819 (HY000): Check constraint 'date_check' is violated.
```

# 14.3 使用触发器拒绝输入值

## 问题

您希望验证要插入表中的数据是否遵循业务逻辑，但您的逻辑比 `CHECK` 约束能处理的更复杂。您可能还需要重写数据而不是拒绝它。或者您正在使用 MySQL 的较早版本，不支持 `CHECK` 约束。

## 解决方案

使用 `BEFORE` 触发器。

## 讨论

`CHECK` 约束有一些限制。它们不允许使用存储的或用户定义的函数、子查询或用户定义的变量。它们还不允许修改插入的数据。如果您想要格式化插入的值以满足业务标准，您可能需要探索其他解决方案，例如在应用程序端进行验证或在 MySQL 端使用 `BEFORE` 触发器。

要在 MySQL 端执行更复杂的验证，请创建触发器并在其中引发 SQL 异常。

让我们看一个例子。假设一个杂货表存储超市中产品的详细信息。在一些国家，超市在特定时间内禁止出售酒精制品。例如，在土耳其，您无法在晚上 10 点至早上 6 点之间在超市购买酒精。如果您遇到此类限制，您可能希望限制用户可以下订单的时间。

假设一个 `groceries` 表存储超市杂货的详细信息。

```
CREATE TABLE `groceries` (
  `id` int NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  `forbidden_after` time DEFAULT NULL,
  `forbidden_before` time DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```

列 `forbidden_after` 和 `forbidden_before` 定义了禁止销售特定物品的时间范围。

另一个名为 `groceries_order_items` 的表包含有关购买的信息。

```
CREATE TABLE groceries_order_items
(
  order_id INT NOT NULL,
  groceries_id INT NOT NULL,
  quantity INT DEFAULT 0,
  PRIMARY KEY (order_id,groceries_id)
) ENGINE=InnoDB;
```

要在某些时间禁止购买物品，您可以创建一个触发器，检查当前时间和对所选产品的任何限制。如果存在限制，则拒绝购买。

```
CREATE TRIGGER check_time
BEFORE INSERT ON groceries_order_items
FOR EACH ROW BEGIN
DECLARE forbidden_after_val TIME; ![1](img/1.png)
DECLARE forbidden_before_val TIME;
DECLARE name_val VARCHAR(255);
DECLARE message VARCHAR(400);

SELECT forbidden_after, forbidden_before, name ![2](img/2.png)
INTO forbidden_after_val, forbidden_before_val, name_val
FROM groceries WHERE id = NEW.groceries_id;

IF (forbidden_after_val IS NOT NULL AND TIME(NOW()) >= forbidden_after_val) ![3](img/3.png)
  OR (forbidden_before_val IS NOT NULL AND TIME(NOW()) <= forbidden_before_val)
THEN
  SET message=CONCAT('It is forbidden to buy ', name_val,
      ' between ', forbidden_after_val, ' and ', forbidden_before_val); ![4](img/4.png)
  SIGNAL SQLSTATE '45000' ![5](img/5.png)
    SET MESSAGE_TEXT = message;
END IF;
END;
```

![1](img/#co_nch-format-format-triggers_declare_co)

声明变量以存储禁止购买的时间范围、产品名称和错误消息。

![2](img/#co_nch-format-format-triggers_select_co)

选择限制的时间范围和产品名称存入变量。

![3](img/#co_nch-format-format-triggers_check_co)

检查当前时间是否在所选产品的禁止范围内。

![4](img/#co_nch-format-format-triggers_message_co)

如果时间落入禁止范围内，编制一条消息，解释该产品的限制。

![5](img/#co_nch-format-format-triggers_signal_co)

引发错误并拒绝插入。

因此，您可以在凌晨 3 点购买奶酪或水，但不能在那时购买啤酒或葡萄酒。

```
mysql> `SELECT` `CURRENT_TIME``(``)``;`
+----------------+ | CURRENT_TIME() |
+----------------+ | 03:01:40       |
+----------------+ 1 row in set (0.00 sec)
mysql> `INSERT` `INTO` `groceries_order_items` `VALUES``(``1``,``3``,``1``)``;` `-- cheese`
Query OK, 1 row affected (0.03 sec)

mysql> `INSERT` `INTO` `groceries_order_items` `VALUES``(``1``,``8``,``3``)``;` `-- water`
Query OK, 1 row affected (0.01 sec)

mysql> `INSERT` `INTO` `groceries_order_items` `VALUES``(``1``,``7``,``6``)``;` `-- beer`
ERROR 1644 (45000): It is forbidden to buy beer between 22:00:00 and 06:00:00
mysql> `INSERT` `INTO` `groceries_order_items` `VALUES``(``1``,``6``,``1``)``;` `-- wine`
ERROR 1644 (45000): It is forbidden to buy wine between 22:00:00 and 06:00:00
```

白天购买限制会放宽。

```
mysql> `SELECT` `CURRENT_TIME``(``)``;`
+----------------+ | CURRENT_TIME() |
+----------------+ | 14:00:35       |
+----------------+ 1 row in set (0.00 sec)

mysql> `INSERT` `INTO` `groceries_order_items` `VALUES``(``1``,``7``,``6``)``;` `-- beer`
Query OK, 1 row affected (0.01 sec)

mysql> `INSERT` `INTO` `groceries_order_items` `VALUES``(``1``,``6``,``1``)``;` `-- wine`
Query OK, 1 row affected (0.01 sec)
```

## 参见

关于使用触发器拒绝或修改无效值的更多信息，请参见配方 11.11。

# 14.4 编写输入处理循环

## 问题

您希望确保文件中的数据值合法。

## 解决方案

编写一个输入处理循环，用于检查它们，可能会将它们重写为更合适的格式。

## 讨论

本章中展示的许多验证示例都是在读取文件并检查各列值的程序上执行的典型操作。这类文件处理实用程序的一般框架如下：

```
#!/usr/bin/python3
# loop.py: Typical input-processing loop.

# Assumes tab-delimited, linefeed-terminated input lines.

import sys

for line in sys.stdin:
    line = line.rstrip()
    # split line at tabs, preserving all fields
    values = line.split("\t")
    for val in values: # iterate through fields in line
        # ... test val here ...
        pass
```

`for()` 循环读取每个输入行。在循环内部，每行被分割为字段。内部的 `for()` 循环迭代处理每个字段，按顺序处理。如果不对所有字段统一应用特定测试，则用单独的基于列的测试替换 `for()` 循环。

此循环假设制表符分隔的、以换行符终止的输入，这是本章讨论的大多数实用程序共享的假设。要使用这些实用程序处理其他格式的数据文件，您可以尝试使用*recipes*分发中提供的*cvt_file.pl*脚本将这些文件转换为制表符分隔格式。

# 14.5 将常见测试放入库中

## 问题

您希望执行重复的验证操作。

## 解决方案

将验证操作封装为库例程。

## 讨论

对于某些验证操作频繁发生并且重复的情况并不罕见，此时您可能会发现构建函数库非常有用。通过将验证操作封装为库例程，编写基于它们的实用程序更加容易，这些实用程序使得在整个文件上执行命令行操作变得更加简单，因此您可以避免手动编辑它们。这也为操作赋予了一个比比较代码本身更清晰的名称。以下是在 Python 语言中进行的测试，执行模式匹配以检查`val`是否完全由数字（可选前导加号）组成，然后确保该值大于零：

```
p = re.compile('^\+?\d+$')
  s = p.search(val)
  valid = s and (s.group(0) != '0')
```

换句话说，该测试寻找表示正整数的字符串。为了使测试更易于使用并使其意图更清晰，请将其封装为以下形式的函数使用：

```
valid = is_positive_integer (val);
```

将函数定义如下：

```
def is_positive_integer(val):
  p = re.compile('^\+?\d+$')
  s = p.search(val)
  return s and (s.group(0) != '0')
```

现在将函数定义放入库文件中，以便多个脚本可以轻松使用。*cookbook_utils.py*模块文件位于`recipes`发行版的*lib*目录中，是一个包含多个验证函数的库文件示例。浏览一下它，看看哪些函数可能对您自己的程序有用（或作为编写自己库文件的模板）。要从脚本内部访问此模块，请包含像这样的`use`语句：

```
import cookbook_utils as cu
```

当然，您必须将模块文件安装在 Python 能找到的目录中（参见 Recipe 4.3）。

将一系列实用程序例程放入库文件的显著好处之一是，您可以将其用于各种程序。数据处理问题很少是完全独特的。如果您可以从库中挑选至少几个验证例程，即使对于高度专业化的程序，也能减少必须编写的代码量。

###### 提示

要避免编写自己的库例程，可以查看是否有其他人已经编写了可用的例程。例如，如果您查看 Perl CPAN（*cpan.perl.org*），您会找到一个 Data::Validate 模块层次结构。那里的模块提供了标准化多种常见验证任务的库例程。Data::Validate::MySQL 专门处理 MySQL 数据类型。

# 14.6 使用模式匹配验证数据

## 问题

您希望将值与难以在没有编写非常丑陋的表达式的情况下指定的一组值进行比较。

## 解决方案

使用模式匹配。

## 讨论

模式匹配是一个强大的验证工具，它使您能够使用单个表达式测试整个类别的值。您还可以使用模式测试将匹配值分解为子部分以进行进一步的个别测试，或者在替换操作中重新编写匹配的值。例如，您可以将匹配的日期分解为部分以验证月份是否在 1 到 12 的范围内，日期是否在该月的天数内。您可以使用替换来重新排列 *`MM-DD-YYYY`* 或 *`DD-MM-YYYY`* 值为 *`YYYY-MM-DD`* 格式。

接下来的几节描述如何使用模式来测试几种类型的值，但首先让我们回顾一些一般的模式匹配原则。以下讨论侧重于 Python 的正则表达式能力。Ruby、PHP、Go 和 Perl 中的模式匹配类似，但应查阅相关文档以了解任何差异。对于 Java，请使用 `java.util.regex` 包。

在 Python 中，正则表达式是 `re` 模块的一部分。模式构造函数是 `re.compile(`*`pat`*`)`：

```
pattern = re.compile(*`pat`*)
```

要查找值是否与模式匹配，请使用 `match` 方法：

```
it_matched = pattern.match(val)    # pattern match
```

您可以在 `match` 方法中构造正则表达式：

```
it_matched = re.match(*`pat`*, val)    # pattern match
```

将标志 `re.I` 作为正则表达式构造函数的第二个参数，使模式匹配不区分大小写：

```
it_matched = re.match(*`pat`*, val, re.I)   # case-insensitive match
```

要查找非匹配项，请用 `=` 操作符替换为 `=` 和 `not` 操作符的组合：

```
no_match = not re.match(*`pat`*, val)     # negated pattern match
```

要根据模式匹配在 `val` 中执行替换，请使用 `re.sub(/`*`pat`*`,` *`replacement`*`, val)`*`replacement`*`/`。如果 `val` 中出现 *`pat`*，则将其替换为 *`replacement`*。要进行大小写不敏感匹配，加上 `re.I` 标志。要执行替换，只替换 *`pat`* 的几个实例而不是所有实例，请添加选项 `count`：

```
val = re.sub(*`pat`*, *`replacement`*, val)    # substitution
val = re.sub(*`pat`*, *`replacement`*, val, flags = re.I)   # case-insensitive substitution
val = re.sub(*`pat`*, *`replacement`*, val, count = 1)   # substitution of the first match
val = re.sub(*`pat`*, *`replacement`*, val, count = 1, flags = re.I)  
             # case-insensitive and the first match
```

表 14-1 显示了 Python 正则表达式中可用的一些特殊模式元素：

表 14-1\. Python 正则表达式中的模式元素

| 模式 | 模式匹配的内容 |
| --- | --- |
| `^` | 字符串的开始 |
| `$` | 字符串的结尾 |
| `.` | 除换行符外的任意字符 |
| `\s`, `\S` | 空白或非空白字符 |
| `\d`, `\D` | 数字或非数字字符 |
| `\w`, `\W` | 单词（字母数字或下划线）或非单词字符 |
| `[...]` | 方括号内列出的任何字符 |
| `[^...]` | 方括号内未列出的任何字符 |
| *`p1`*`&#124;`*`p2`*`&#124;`*`p3`* | 选择；匹配任何模式 *`p1`*、*`p2`* 或 *`p3`* |
| `*` | 前导元素的零个或多个实例 |
| `+` | 前导元素的一个或多个实例 |
| `{`*`n`*`}` | 前导元素的 *`n`* 个实例 |
| `{`*`m`*`,`*`n`*`}` | *`m`* 到 *`n`* 个前导元素的实例 |

这些模式元素中的许多与 MySQL 的 `REGEXP` 正则表达式运算符可用元素相同（参见 Recipe 7.11）。

要匹配模式中特殊字符（如`*`、`^`或`$`）的文字实例，请在其前面加上反斜杠。类似地，要在字符类构造中包含在字符类中特殊的字符（`[`、`]`或`-`），请在其前面加上反斜杠。要在字符类中包含文字`^`，请将其列在括号之间不作为第一个字符。

下面展示的许多验证模式的形式是`^`*`pat`*`$`。用`^`和`$`开头和结尾的模式的效果是要求*`pat`*匹配您测试的整个字符串。在数据验证环境中，这是常见的，因为通常希望知道模式是否完全匹配整个输入值，而不仅仅是部分匹配。但这不是一成不变的规则，有时根据需要可以省略适当的`^`和`$`字符执行更轻松的测试。例如，如果要从值中去除前导和尾随空格，请使用一个仅锚定到字符串开头的模式，以及另一个仅锚定到字符串结尾的模式：

```
val = re.sub('^\s+', '', val)   # trim leading whitespace
val = re.sub('\s+$', '', val)   # trim trailing whitespace
```

事实上，这是一个常见操作，可以作为实用函数编写。*cookbook_utils.py*文件包含一个执行两个替换并返回结果的`trim_whitespace()`函数：

```
val = trim_whitespace (val)
```

要记住模式匹配字符串的子部分，请在相关模式部分周围使用括号。成功匹配后，可以在正则表达式内部使用变量`\1`、`\2`等引用匹配的子字符串，或者作为`group`方法的参数使用匹配编号：

```
match = re.match('^(\d+)(.*)$', '2021-04-25')
if match:
  first_part = match.group(1) # this is the year, 2021
  the_rest = match.group(2)   # this is the rest of the date, -04-25
```

如果要指示模式中的元素是可选的，请在其后面加上`?`字符。要匹配由一系列数字组成的值，可选择以负号开头，并可选择以句点结尾，请使用此模式：

```
^-?\d+\.?$
```

使用括号在模式内部分组交替项。以下模式匹配*`hh:mm`*格式的时间值，可选择后跟`AM`或`PM`：

```
^\d{1,2}:\d{2}\s*(AM|PM)?$
```

在该模式中使用括号还会记住`\1`中的可选部分。要抑制该副作用，请改用`(?:`*`pat`*`)`：

```
^\d{1,2}:\d{2}\s*(?:AM|PM)?$
```

您现在具备了足够的 Python 模式匹配背景，可以构建用于多种数据值的有用验证测试。以下食谱提供可用于测试广泛内容类型、数字、时间值以及电子邮件地址或 URL 的模式。

`recipes`发行版的*transfer*目录包含一个*test_pat.py*脚本，该脚本读取输入值，将其与多个模式进行匹配，并报告每个值匹配的模式。该脚本易于扩展，因此您可以将其用作测试框架，以尝试自己的模式。

# 14.7 使用模式匹配广泛内容类型

## 问题

您希望将值分类到不同类别中。

## 解决方案

使用一个使用类似广泛类别的模式。

## 讨论

要检查值是否为空或非空，或仅由某些类型的字符组成，可使用 表 14-2 中列出的模式。

表 14-2\. 常用字符类别

| 模式 | 该模式匹配的值类型 |
| --- | --- |
| `^$` | 空值 |
| `.` | 非空值 |
| `^\s*$` | 空格，可能为空 |
| `^\s+$` | 非空白格 |
| `\S` | 非空，且非空白 |
| `^\d+$` | 仅数字，非空 |
| `^[a-zA-Z]+$` | 仅字母字符（不区分大小写），非空 |
| `^\w+$` | 仅字母数字或下划线字符，非空 |

# 14.8 使用模式匹配数值

## 问题

您希望确保字符串看起来像一个数字。

## 解决方案

使用匹配您要查找的数字类型的模式。

## 讨论

模式可用于将值分类为多种数字类型，如 表 14-3 所示

表 14-3\. 匹配数字的模式

| 模式 | 该模式匹配的值类型 |
| --- | --- |
| `^\d+$` | 无符号整数 |
| `^-?\d+$` | 负数或无符号整数 |
| `^[-+]?\d+$` | 有符号或无符号整数 |
| `^[-+]?(\d+(\.\d*)?&#124;\.\d+)$` | 浮点数 |

模式 `^\d+$` 通过要求一个非空值，其值从开始到结束只包含数字，匹配无符号整数。如果你只关心值以整数开头，你可以匹配字符串的初始数字部分并提取它。为此，只匹配字符串的初始部分（省略要求模式匹配到字符串结尾的 `$`），并在 `\d+` 部分周围加上括号。然后，在成功匹配后，将匹配的数字称为 `group(1)`：

```
match = re.match('^(\d+)', val)
if match:
  val = match.group(1)
```

一些类型的数值具有特殊的格式或其他不寻常的约束。以下是一些示例及其处理方法：

ZIP（邮政编码）

ZIP 和 ZIP+4 是美国邮政编码，用于邮寄。它们的值如 `12345` 或 `12345-6789`（即五位数字，可能后跟一个短横线和四位数字）。要匹配其中一种形式或两种形式，请使用 表 14-4 中显示的模式：

表 14-4\. 匹配邮政编码的模式

| 模式 | 该模式匹配的值类型 |
| --- | --- |
| `^\d{5}$` | ZIP 编码，仅五位数字 |
| `^\d{5}-\d{4}$` | ZIP+4 编码 |
| `^\d{5}(-\d{4})?$` | ZIP 或 ZIP+4 编码 |

信用卡号

信用卡号通常由数字组成，但常见的写法是在数字组之间使用空格、短横线或其他字符。例如，以下号码是等效的：

```
0123456789012345
0123 4567 8901 2345
0123-4567-8901-2345
```

要匹配这样的值，请使用此模式：

```
^[- \d]+
```

（Python 允许在字符类中使用 `\d` 数字指定符号。）但是，该模式不能识别长度不正确的值，并且在存储值到 MySQL 前去除多余字符可能很有用。要求信用卡号必须包含 16 位数字，可以使用替换方法去除所有非数字字符，然后检查结果的长度：

```
val = re.sub('\D', '', val)
valid = len(val) == 16
```

# 14.9 使用模式匹配日期或时间

## 问题

您希望确保字符串看起来像是日期或时间。

## 解决方案

使用匹配您期望的时间值类型的模式。务必考虑诸如子部分之间的分隔符严格程度以及子部分的长度等问题。

## 讨论

日期是验证的头疼问题，因为它们有许多不同的格式。模式测试非常有用以清除非法值，但通常不足以进行完整验证：例如，日期可能在您期望的月份处有一个数字，但如果数字为 13，则日期无效。本节介绍了一些匹配几种常见日期格式的模式。食谱 14.14 更详细地重新讨论了这个主题，并讨论了如何将模式测试与内容验证结合起来。

要求值为 ISO（*`YYYY-MM-DD`*）格式的日期，请使用此模式：

```
^\d{4}-\d{2}-\d{2}$
```

该模式要求 `-` 字符作为日期部分之间的分隔符。要允许 `-` 或 `/` 作为分隔符，请在数字部分之间使用字符类：

```
^\d{4}[-/]\d{2}[-/]\d{2}$
```

此模式将匹配格式为 *`YYYY-MM-DD`*、*`YYYY/MM/DD`*、*`YYYY/MM-DD`* 和 *`YYYY-MM/DD`* 的日期。

要允许任何非数字分隔符（这与 MySQL 在将字符串解释为日期时的操作方式相对应），请使用此模式：

```
^\d{4}\D\d{2}\D\d{2}$
```

要允许类似 `03` 这样的值的前导零可能缺失，只需查找三个非空数字序列即可：

```
^\d+\D\d+\D\d+$
```

当然，该模式非常通用，也会匹配其他值，例如美国社会安全号码（其格式为 012-34-5678）。为了通过要求年部分中的二至四位数字和月日部分中的一到两位数字来约束子部分的长度，使用此模式：

```
^\d{2,4}?\D\d{1,2}\D\d{1,2}$
```

对于其他格式（如 *`MM-DD-YY`* 或 *`DD-MM-YY`*）的日期，类似的模式也适用，但子部分的排列顺序不同。该模式匹配这两种格式：

```
^\d{2}-\d{2}-\d{2}$
```

要检查单个日期部分的值，请在模式中使用括号，并在成功匹配后提取子字符串。例如，如果期望日期为 ISO 格式，请执行以下操作：

```
match = re.match('^(\d{2,4})\D(\d{1,2})\D(\d{1,2})$', val)
if match:
  (year, month, day) = (match.group(1), match.group(2), match.group(3))
```

在 `recipes` 分发中的库文件 *lib/cookbook_utils.py* 包含几个这些模式测试，封装为函数调用。如果日期不匹配模式，它们将返回 `None`。否则，它们将返回一个包含年、月和日分解值的数组引用。这对于对日期组件执行进一步检查非常有用。例如，`is_iso_date()` 寻找匹配 ISO 格式的日期。定义如下：

```
def is_iso_date(val):
  m = re.match('^(\d{2,4})\D(\d{1,2})\D(\d{1,2})$', val)
  return [int(m.group(1)),  int(m.group(2)), int(m.group(3))] if m else None
```

可以如下使用该函数：

```
ref = cu.is_iso_date(val)
if ref is not None:
  # val matched ISO format pattern;
  # check its subparts using ref[0] through ref[2]
  pass
else:
  # val didn't match ISO format pattern
  pass
```

通常，需要对日期进行额外处理，因为日期匹配模式有助于淘汰语法上有问题的值，但不能评估各个组件是否包含合法值。为了做到这一点，需要进行一些范围检查。配方 14.14 涵盖了这个主题。

如果愿意跳过子部分测试，只想重写部分，请使用替换。例如，将假定为 *`MM-DD-YY`* 格式的值重写为 *`YY-MM-DD`* 格式，可以这样做：

```
val = re.sub('^(\d+)\D(\d+)\D(\d+)$', r'\3-\1-\2', val)
```

时间值通常比日期更有条理，通常先写小时，最后写秒，每部分两位数：

```
^\d{2}:\d{2}:\d{2}$
```

为了更宽松，允许小时部分为单个数字，或者允许秒部分缺失：

```
^\d{1,2}:\d{2}(:\d{2})?$
```

如果想要对单独的时间部分进行范围检查或重新格式化值以包含缺失的秒部分 `00`，可以在时间模式中使用括号标记这些部分。但是，如果第三个时间部分是可选的，则在模式中括号和 `?` 字符需要特别小心。您希望整个 `:\d{2}` 在模式末尾是可选的，但不保存 `\3` 中的 `:` 字符，如果第三个时间部分存在。为了实现这一点，请使用 `(?:`*`pat`*`)`，这是一种不保存匹配子串的分组符号。在这种符号内，使用括号将数字保存起来。然后，如果秒部分不存在，`\3` 就是 `None`，否则它包含秒数：

```
m = re.match('^(\d{1,2}):(\d{2})(?::(\d{2}))?$', val)
(hour, min, sec) = (m.group(1), m.group(2), m.group(3))
sec = '00' if sec is None else sec # seconds missing; use 00
val = hour + ':' + min + ':' + sec
```

要将带有 AM 和 PM 后缀的 12 小时制时间重写为 24 小时制，可以这样做：

```
m = re.match('^(\d{1,2})\D(\d{2})\D(\d{2})(?:\s*(AM|PM))?$', val, flags = re.I)
(hour, min, sec) = (m.group(1), m.group(2), m.group(3))
# supply missing seconds
sec = '00' if sec is None else sec 
if int(hour) == 12 and (m.group(4) is None or m.group(4).upper() == "AM"):
  hour = '00' # 12:xx:xx AM times are 00:xx:xx
elif int(hour) < 12 and (m.group(4) is not None) and m.group(4).upper() == "PM":
  hour = int(hour) + 12 # PM times other than 12:xx:xx
return [hour, min, sec] # return hour, minute, second
```

时间部分被放置在 `1`、`2` 和 `3` 组中，如果秒部分缺失，则 `3` 组设为 `None`。后缀（如果存在）放入 `4` 组中。如果后缀是 `AM` 或缺失 (`None`)，则将其解释为上午时间。如果后缀是 `PM`，则将其解释为下午时间。

## 参见

这个配方展示了在数据传输目的下处理日期时的初步内容。日期和时间的测试和转换可能会高度个性化，要考虑的问题数量之多令人难以置信：

+   基本日期格式是什么？日期有几种常见的风格，例如 ISO 格式（*`YYYY-MM-DD`*）、美国格式（*`MM-DD-YY`*）和英国格式（*`DD-MM-YY`*）。这些只是更标准的格式之一。还有很多可能的格式。例如，数据文件中的日期可能写成 `June` `17,` `1959` 或 `17` `Jun` `'59`。

+   日期末尾是否允许有时间，或者可能是必须的？期望时间时，是否需要全时间还是只需时和分？

+   是否允许特殊值如 `now` 或 `today`？

+   日期部分是否需要由特定字符（如 `-` 或 `/`）分隔，或者允许使用其他分隔符？

+   日期部分是否需要特定数量的数字？月份和年份的前导零是否允许缺失？

+   月份是用数字写的，还是用像`January`或`Jan`这样的月份名表示？

+   两位数年份应如何转换为四位数？在 `00` 到 `99` 范围内，值在哪个转换点上从一个世纪变为另一个世纪？

+   应该检查日期部分以确保它们的有效性吗？模式可以识别看起来像日期或时间的字符串，但是虽然它们非常有用于检测格式错误的值，但可能还不足够。例如，`1947-15-99` 这样的值可能会匹配一个模式，但它并不是一个合法的日期。因此，模式测试在与对日期的各个部分的范围检查结合使用时最为有用。

在数据传输问题中，这些问题的普遍存在意味着你可能会偶尔编写一些自定义验证器来处理非常特定的日期格式。本章的其他部分可以提供额外的帮助。例如，配方 14.13 讨论了如何将两位数年份转换为四位数形式，而 配方 14.14 则讨论了如何对日期或时间值的组件执行有效性检查。

你可能能够通过使用现有的日期检查模块来减少一些工作量，适用于你的 API 语言。一些可能的选择包括：Perl 的 `Date` 模块；Ruby 的 `date` 模块；Python 的 `datetime` 模块；PHP 的 `DateTime` 类；Java 的 `GregorianCalendar` 和 `SimpleDateTime` 类。

# 14.10 使用模式匹配电子邮件地址或 URL

## 问题

你想确定一个值是否看起来像是一个电子邮件地址或一个 URL。

## 解决方案

在你的应用程序中使用一个模式，调整到你接受哪些地址和不接受哪些地址的期望严格程度。

## 讨论

立即前面的配方使用模式来识别诸如数字和日期之类的值的类别，这些都是正则表达式的典型应用场景。但是，模式匹配在数据验证中具有更广泛的适用性。为了给出一些使用模式匹配的其他类型值的想法，本配方展示了一些用于电子邮件地址和 URL 的测试。

要检查预期为电子邮件地址的值，模式应至少要求一个带有非空字符串的`@`字符：

```
.@.
```

###### 注意

完整的电子邮件地址规范由 [RFC5322](https://www.ietf.org/rfc/rfc5322.txt) 定义，并包含许多部分。拒绝所有无效地址并接受所有有效地址的正则表达式相当复杂。参考 [*http://emailregex.com/*](http://emailregex.com/) 获取流行编程语言的示例，以便了解更多。

在这个配方中，我们将展示一个非常简单的测试，但它仍然足以帮助纠正大多数无辜用户在输入地址时的拼写错误。

很难找到一个通用的模式，涵盖所有合法值并排除所有非法值，但写一个至少更加严格的模式却很容易。例如，除了非空外，用户名和域名应完全由`@`字符或空格以外的字符组成：

```
^[^@ ]+@[^@ ]+$
```

您可能还希望要求域名部分包含至少由点分隔的两部分：

```
^[^@ ]+@[^@ .]+\.[^@ .]+
```

要查找以`http://`、`https://`、`ftp://`或`mailto:`协议开头的 URL 值，请使用匹配任何一个在字符串开头的选择项。

```
re.compile('^(https?://|ftp://|mailto:)', flags=re.I)
```

括号内的模式选择项被分组，因为否则`^`锚定只适用于第一个。`re.I`标志跟随模式，因为 URL 中的协议标识符不区分大小写。除了允许协议标识符后跟任何内容外，该模式相当不限制。根据需要添加进一步限制。

# 14.11 使用表元数据验证数据

## 问题

您希望根据`ENUM`或`SET`列的合法成员检查输入值。

## 解决方案

获取列定义，从中提取成员列表，并检查数据值是否与列表匹配。

## 讨论

某些验证形式涉及检查输入值是否与存储在数据库中的信息相匹配。这包括要存储在`ENUM`或`SET`列中的值，可以与列定义中存储的有效成员进行比较。基于数据库的验证还适用于必须与查找表中列出的内容匹配的值以被视为合法的情况。例如，包含客户 ID 的输入记录可以要求匹配`customers`表中的行，地址中的州缩写可以验证是否与列出每个州的表匹配。本文档介绍了基于`ENUM`和`SET`的验证，而 Recipe 14.12 讨论了如何使用查找表。

检查与`ENUM`或`SET`列的合法值相对应的输入值的一种方法是使用`INFORMATION_SCHEMA`中的信息将合法列值列表转换为数组，然后执行数组成员测试。例如，`profile`表中的`color`列是作为以下定义的`ENUM`的最喜欢的颜色列：

```
mysql> `SELECT COLUMN_TYPE FROM INFORMATION_SCHEMA.COLUMNS`
    -> `WHERE TABLE_SCHEMA = 'cookbook' AND TABLE_NAME = 'profile'`
    -> `AND COLUMN_NAME = 'color';`
```

```
+----------------------------------------------------+
| COLUMN_TYPE                                        |
+----------------------------------------------------+
| enum('blue','red','green','brown','black','white') |
+----------------------------------------------------+
```

如果您从`COLUMN_TYPE`值中提取枚举成员列表并将其存储在列表`members`中，则可以像这样执行成员测试：

```
valid = True ↩
if list(map(lambda v: v.upper(), members)).count(val.upper()) > 0 ↩
else False
```

我们可以将列表`members`和`val`转换为大写以执行大小写不敏感比较，因为默认排序规则是`utf8mb4_0900_ai_ci`，即不区分大小写。(如果列具有不同的排序规则，请相应调整。我们在 Recipe 7.5 中讨论了如何更改列排序规则)

在食谱 12.6 中，我们编写了一个函数`get_enumorset_info()`，它返回`ENUM`或`SET`列的元数据。这包括成员列表，因此很容易使用该函数编写另一个实用程序例程`check_enum_value()`，获取法律枚举值并执行成员测试。该例程接受四个参数：数据库句柄、`ENUM`列的表名和列名，以及要检查的值。它返回 true 或 false，指示该值是否合法：

```
def check_enum_value(conn, db_name, tbl_name, col_name, val):
  valid = 0 
  info = get_enumorset_info(conn, db_name, tbl_name, col_name)
  if info is not None and info['type'].upper() == 'ENUM':
    # use case-insensitive comparison because default collation
    # (utf8mb4_0900_ai_ci) is case-insensitive (adjust if you use
    # a different collation)
    valid = 1 ↩
    if list(map(lambda v: v.upper(), info['values'])).count(val.upper()) > 0 ↩
    else 0
  return valid
```

对于单值测试，例如验证在网页表单中提交的值，每个值的列表查找效果很好。然而，对于测试大量值（如数据文件中的整列），最好是将枚举值一次性读入内存，然后重复使用它们来检查每个数据值。此外，在 Python 中，执行字典查找比列表查找效率要高得多。为此，获取法律枚举值并将它们存储为字典的键。然后通过检查每个输入值是否存在于字典键中来测试每个输入值。构建字典需要一些额外的工作量，这也是为什么`check_enum_value()`没有这样做的原因。但对于批量验证来说，改进的查找速度远远弥补了字典构建的开销。（要自行检查列表成员测试与字典查找的相对效率，请尝试`recipes`发行版的*transfer*目录中的*lookup_time.py*脚本。）

首先获取列的元数据，然后将法律枚举成员列表转换为字典：

```
info = get_enumorset_info(conn, db_name, tbl_name, col_name)
members={}
# convert dictionary key to consistent lettercase
for v in info['values']:
  members[v.lower()] = 1
```

`for`循环使每个枚举成员存在于字典元素的键中。这里重要的是字典键；与之关联的值是无关的。（所示示例将值设置为`1`，但你可以使用`None`、`0`或任何其他值。）请注意，在存储之前，代码将字典键转换为小写。这是因为 Python 中的字典键查找是区分大小写的。如果你检查的值也是区分大小写的，那么这样做是可以接受的，但默认情况下，`ENUM`列不区分大小写。通过在存储枚举值之前将其转换为给定的大小写，然后类似地转换你想要检查的值，实际上进行了一个不区分大小写的键存在测试：

```
valid = 1 if val.lower() in members else 0
```

该示例将枚举值和输入值转换为小写。只要所有值保持一致，你也可以使用大写。

请注意，如果输入值是空字符串，则存在测试可能失败。您必须决定如何根据列处理这种情况。例如，如果列允许`NULL`值，您可能会将空字符串解释为等同于`NULL`，因此视为合法值。

对于`SET`值的验证过程与`ENUM`值类似，只是输入值可能包含任意数量的`SET`成员，用逗号分隔。 为了使值合法，其中的每个元素都必须合法。 另外，因为<q>任意数量的成员</q>包括<q>无</q>，所以空字符串是任何`SET`列的合法值。

对于单个输入值的一次性测试，请使用类似于`check_enum_value()`的实用程序例程`check_set_value()`：

```
def check_set_value(conn, db_name, tbl_name, col_name, val):
  valid = 0 
  info = get_enumorset_info(conn, db_name, tbl_name, col_name)
  if info is not None and info['type'].upper() == 'SET':
    if val == "": 
      return 1 # empty string is legal element
    # use case-insensitive comparison because default collation
    # (utf8mb4_0900_ai_ci) is case-insensitive (adjust if you use
    # a different collation)
    valid = 1  # assume valid until we find out otherwise
    for v in val.split(','):
      if list(map(lambda x: x.upper(), info['values'])).count(v.upper()) <= 0:
        valid = 0 
        break
  return valid
```

对于批量测试，请构建从合法`SET`成员到字典的转换。 该过程与以前显示的从`ENUM`元素生成字典的过程相同。

要根据`SET`成员字典验证给定的输入值，请将其转换为与哈希键相同的大小写，将其在逗号处分割为值的各个元素的列表，然后检查每个元素。 如果任何元素无效，则整个值无效：

```
valid = 1 # assume valid until we find out otherwise
for v in val.split(","):
  if v.lower() not in members:
    valid = 0 
    break
```

循环终止后，如果值对于`SET`列合法，则`valid`为 true，否则为 false。 空字符串始终是合法的`SET`值，但此代码不对空字符串执行特殊情况测试。 在这种情况下，不需要此类测试，因为`split()`操作返回空列表，循环永不执行，并且`valid`保持为 true。

# 14.12 使用查找表验证数据

## Problem

您希望检查值以确保它们在查找表中列出。

## 解决方案

检查表中是否存在值的问题陈述。 最佳方法取决于输入值的数量和表的大小。 在这个示例中，我们将从发出单个语句开始讨论，然后从整个查找表创建哈希，最后通过记住已经看到的值来改进我们的算法，以避免在大数据集中多次查询数据库。

## 讨论

要针对查找表的内容验证输入值，技术与 Recipe 14.11 中用于检查`ENUM`和`SET`列的技术有些相似。 然而，虽然`ENUM`和`SET`列通常只有少量成员值，但查找表可以有数量基本上无限的值。 您可能不希望将它们全部读入内存中。

可以通过几种方法对输入值与查找表的内容进行验证，如下面的讨论所示。 在示例中显示的测试执行与存储在查找表中的值完全相同。 要执行不区分大小写的比较，请将所有值转换为一致的大小写。 （请参阅 Recipe 14.11 中的大小写转换讨论。）

### 发出单个语句

对于一次性操作，通过检查它是否在查找表中列出来测试一个值。 以下查询对于存在的值返回 true（非零），否则返回 false：

```
cursor.execute("select count(*) from *`tbl_name`* where val = %(val)s", {'val': *`value`*})
valid = cursor.fetchone()[0]
```

这种测试可能适用于诸如检查提交的 Web 表单中的值之类的目的，但对于验证大型数据集来说效率低下。它没有为先前看到的值的结果保存内存；因此，您需要为每个输入值执行查询。

### 从整个查找表构造哈希

要验证大量值，最好将查找值放入内存中，将它们保存在数据结构中，并检查每个输入值是否与该结构的内容匹配。使用内存中的查找可以避免执行每个值的查询的开销。

首先，运行一个查询以检索所有查找表值，并从中构建一个字典：

```
members = {}  # dictionary for lookup values
cursor.execute("SELECT val FROM *`tbl_name`*");
rows = cursor.fetchall()
for row in rows:
  members[row[0]] = 1
```

然后，执行字典键存在性测试以检查给定值：

```
valid = True if val in members else False
```

对于大型查找表，这种技术将数据库流量减少到单个查询。然而，这可能仍然是大量流量，并且您可能不希望将整个表保存在内存中。

### 记住已经看过的值，以避免数据库查找

另一种查找技术将单个语句与存储查找值存在信息的字典混合。如果你有一个非常庞大的查找表，这种方法可以很有用。从一个空字典开始：

```
members = {}  # dictionary for lookup values
```

然后，对于要测试的每个值，请检查字典中是否存在。如果不存在，请执行查询以检查该值是否存在于查找表中，并记录查询结果在字典中。输入值的有效性由与键相关联的值决定，而不是键的存在：

```
if val not in members: # haven't seen this value yet
  cursor.execute(f"SELECT COUNT(*) FROM {tbl_name} WHERE val = %(val)s",↩
                 {'val': val})
  count = cursor.fetchone()[0]
  # store true/false to indicate whether value was found
  members[val] = True if count > 0 else False
valid = members[val]
```

对于这种方法，字典充当缓存，因此您只需为任何给定值执行一次查找查询，无论它在输入中出现多少次。对于具有重复值的数据集，这种方法避免了为每个单独的测试发出单独的查询，同时仅要求字典中的每个唯一值都有一个条目。因此，在数据库流量和程序内存需求之间的权衡方面，它处于其他两种方法之间。

请注意，对于此方法，字典的使用方式与之前的方法不同。以前，字典中输入值作为键的存在确定了值的有效性，字典键相关联的值是无关紧要的。对于字典作为缓存的方法，字典中键的存在意义从“它是有效的”变为“它已经被测试过了”。对于每个键，与之相关联的值指示输入值是否存在于查找表中。（如果你只存储那些在查找表中发现为无效值的键，那么你在输入数据集中的每个无效值实例上都会发出一个查询，这是低效的。）

# 14.13 将两位数年份值转换为四位数形式

## 问题

您想将日期值中的年份从两位数转换为四位数。

## 解决方案

让 MySQL 为您完成这些操作，或者如果 MySQL 的转换规则不合适，可以自行执行操作。

## 讨论

两位数年份值是一个问题，因为数据值中没有明确的世纪。如果你知道输入的年份范围，可以在不含歧义地添加世纪。否则，只能猜测。例如，日期 10/2/69 在大多数美国人眼中是 1969 年 10 月 2 日。但如果它代表圣雄甘地的生日，则年份实际上是 1869 年。

将年份转换为四位数的一种方法是让 MySQL 处理。如果你试图插入一个包含两位数年份的日期到`YEAR`列中，MySQL 会自动将其转换为四位数形式。MySQL 使用 1970 年作为过渡点；它将值从 00 到 69 解释为 2000 到 2069 年的年份，将值从 70 到 99 解释为 1970 到 1999 年的年份。这些规则适用于 1970 到 2069 年范围内的年份值。如果你的值超出此范围，存入 MySQL 之前请自行添加正确的世纪。

```
mysql> `SELECT` `CAST``(``69` `AS` `YEAR``)` `AS` `` ` ```69``` ` ```,`

    -> `CAST``(``70` `AS` `YEAR``)` `AS` `` ` ```70``` ` ```,` 
    -> `CAST``(``22` `AS` `YEAR``)` `AS` `` ` ```22``` ` ```;`

+------+------+------+ | 69   | 70   | 22   |

+------+------+------+ | 2069 | 1970 | 2022 |

+------+------+------+

```

To use a different transition point, convert years to four-digit form yourself. Here’s a general-purpose routine that converts two-digit years to four digits and supports an arbitrary transition point:

```

def yy_to_yyyy(year, transition_point = 70):

if year < 100:

    year += 1900 if year >= transition_point else 2000

return year

```

The function uses MySQL’s transition point (70) by default. An optional second argument may be given to provide a different transition point. `yy_to_yyyy()` also verifies that the year actually is less than 100 and needs converting before modifying it. That way you can pass year values regardless of whether they include the century. Some sample invocations using the default transition point have the following results:

```

val = yy_to_yyyy (60)         # 返回 2060

val = yy_to_yyyy (1960)       # 返回 1960（未进行转换）

```

Suppose that you want to convert year values as follows, using a transition point of 50:

```

00 .. 49 -> 2000 .. 2049

50 .. 99 -> 1950 .. 1999

```

To do this, pass an explicit transition point argument to `yy_to_yyyy()`:

```

val = yy_to_yyyy (60, 50)     # 返回 1960

val = yy_to_yyyy (1960, 50)   # 返回 1960（未进行转换）

```

The `yy_to_yyyy()` function is included in the *cookbook_utils.py* library file of the `recipes` distribution.

# 14.14 Performing Validity Checking on Date or Time Subparts

## Problem

A string passes a pattern test as a date or time, but you want to perform further validity checking.

## Solution

Break the value into parts and perform the appropriate range checking on each part.

## Discussion

Pattern matching may not be sufficient for date or time checking. For example, a value like `1947-15-19` might match a date pattern, but it’s not a legal date. To perform more rigorous value testing, combine pattern matching with range checking. Break out the year, month, and day values, then check whether each is within the proper range. Years should be less than 9999 (MySQL represents dates to an upper limit of `9999-12-31`), month values must be in the range from 1 to 12, and days must be in the range from 1 to the number of days in the month. That last part is the trickiest: it’s month-dependent, and also year-dependent for February because it changes for leap years.

Suppose that you’re checking input dates in ISO format. In Recipe 14.9, we used the `is_iso_date()` function from the *cookbook_utils.py* library file to perform a pattern match on a date string and break it into component values. `is_iso_date()` returns `None` if the value doesn’t satisfy a pattern that matches ISO date format. Otherwise, it returns a reference to an array containing the year, month, and day values. The *cookbook_utils.py* file also contains `is_mmddyy_date()` and `is_ddmmyy_date()` routines that match dates in US or British format and return `None` or a reference to a list of date parts. (The parts returned are always in year, month, day order, not the order in which the parts appear in the input date string.)

To perform additional checking on the result returned by any of those routines (assuming that the result is not `None`), pass the date parts to `is_valid_date()`, another library function:

```

valid = is_valid_date(ref[0], ref[1], ref[2])

```

`is_valid_date()` returns nonzero if the date is valid, 0 otherwise. It checks the parts of a date like this:

```

def is_valid_date(year, month, day):

print(year, month, day)

if year < 0: # 或(month < 0)或(day < 1):

    返回 0

if year > 9999 or month > 12 or day > days_in_month(year, month):

    return 0

return 1

```

`is_valid_date()` requires separate year, month, and day values, not a date string. This requires that you break candidate values into components before invoking it, but makes it applicable in more contexts. For example, you can use it to check dates like `12` `February` `2003` by mapping the month to its numeric value before calling `is_valid_date()`. If `is_valid_date()` took a string argument assumed to be in a specific date format, it would be much less general.

`is_valid_date()` uses a subsidiary function `days_in_month()` to determine the number of days in the month represented by the date. `days_in_month()` requires both the year and the month as arguments because if the month is 2 (February), the number of days depends on whether the year is a leap year. This means you *must* pass a four-digit year value, two-digit years are ambiguous with respect to the century, which makes proper leap-year testing impossible. The `days_in_month()` and `is_leap_year()` functions are based on techniques taken from that recipe:

```

def is_leap_year(year):

return ((year % 4 == 0) and ((year % 100 != 0) or (year % 400 == 0) ) )

def days_in_month(year, month):

day_tbl = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31]

days = day_tbl[month - 1]

if month == 2 and is_leap_year(year):

    days += 1

返回天数

```

To perform validity checking on time values, a similar procedure applies: verify that the value matches a time pattern and break it into components, then perform range-testing on the components. For times, the ranges are 0 to 23 for the hour, and 0 to 59 for the minute and second. Here is a function `is_24hr_time()` that checks for values in 24-hour format and returns the components:

```

def is_24hr_time(val):

m = re.match('^(\d{1,2})\D(\d{2})\D(\d{2})$', val)

if m is None:

    return None

return[int(m.group(1)), int(m.group(2)), int(m.group(3))]

```

The following `is_ampm_time()` function is similar but looks for times in 12-hour format with an optional AM or PM suffix, converting PM times to 24-hour values:

```

def is_ampm_time(val):

m = re.match('^(\d{1,2})\D(\d{2})\D(\d{2})(?:\s*(AM|PM))?$', val, flags = re.I)

if m is None:

    return None

(hour, min, sec) = (int(m.group(1)), (m.group(2)), (m.group(3)))

# supply missing seconds

sec = '00' if sec is None else sec

if hour == 12 and (m.group(4) is None or m.group(4).upper() == "AM"):

    hour = '00' # 12:xx:xx AM times are 00:xx:xx

elif int(hour) < 12 and (m.group(4) is not None) and m.group(4).upper() == "PM":

    hour = hour + 12 # PM 时间除了 12:xx:xx

return [hour, min, sec] # 返回小时、分钟、秒

```

Both functions return `None` for values that don’t match the pattern. Otherwise, they return a reference to a three-element array containing the hour, minute, and second values.

After you obtain the time components, pass them to `is_valid_time()`, another utility routine, to perform range checks.

# 14.15 Writing Date-Processing Utilities

## Problem

There is a date-processing operation that you want to perform frequently.

## Solution

Write a utility that performs the date-processing operation for you.

## Discussion

Due to the idiosyncratic nature of dates, you might occasionally find it necessary to write date converters. This section shows some sample converters that serve various purposes:

*   *isoize_date.py* reads a file looking for dates in US format (*`MM-DD-YY`*) and converts them to ISO format.

*   *cvt_date.py* converts dates to and from any of ISO, US, or British formats. It is more general than *isoize_date.py*, but requires that you tell it what kind of input to expect and what kind of output to produce.

*   *monddyyyy_to_iso.py* looks for dates like `Feb.` `6`, `1788` and converts them to ISO format. It illustrates how to map dates with nonnumeric parts to a format that MySQL understands.

All three scripts are located in the *transfer* directory of the `recipes` distribution. They assume datafiles are in tab-delimited, linefeed-terminated format. To work with files that have a different format, use *cvt_file.pl*, available in the recipes distribution.

Our first date-processing utility, *isoize_date.py*, looks for dates in US format and rewrites them into ISO format. You’ll recognize that it’s modeled after the general input-processing loop, with some extra stuff thrown in to perform a specific type of conversion:

```

#!/usr/bin/python3

# isoize_date.py：读取输入数据，查找与之匹配的值

# a date pattern, convert them to ISO format. Also converts

# 2-digit years to 4-digit years, using a transition point of 70.

# By default, this looks for dates in MM-DD-[CC]YY format.

# Does not check whether dates actually are valid (for example,

# won't complain about 13-49-1928).

# Assumes tab-delimited, linefeed-terminated input lines.

import sys

import re

import fileinput

# transition point at which 2-digit XX year values are assumed to be

# 19XX (below that, they are treated as 20XX)

transition = 70

for line in fileinput.input(sys.argv[1:]):

val = line.split("\t", 10000);  # split, preserving all fields

for i in range(0, len(val)):

    # look for strings in MM-DD-[CC]YY format

    m = re.match('^(\d{1,2})\D(\d{1,2})\D(\d{2,4})$', val[i])

    if not m:

    continue

    (month, day, year) = (int(m.group(1)), int(m.group(2)), int(m.group(3)))

    # to interpret dates as DD-MM-[CC]YY instead, replace preceding

    # line with the following one:

    # (day, month, year) = (int(m.group(1)), int(m.group(2)), int(m.group(3)))

    # convert 2-digit years to 4 digits, then update value in array

    if year < 100:

    year += 1900 if year >= transition else 2000

    val[i] = "%04d-%02d-%02d" % (year, month, day)

print("\t".join (val))

```

If you feed *isoize_date.py* an input file that looks like this:

```

Sybil   04-13-70

Nancy   09-30-69

Ralph   11-02-73

Lothair 07-04-63

Henry   02-14-65

Aaron   09-17-68

Joanna  08-20-52

Stephen 05-01-60

```

It produces the following output:

```

Sybil   1970-04-13

Nancy   2069-09-30

Ralph   1973-11-02

Lothair 2063-07-04

Henry   2065-02-14

Aaron   2068-09-17

Joanna  2052-08-20

Stephen 2060-05-01

```

*isoize_date.py* serves a specific purpose: it converts only from US to ISO format. It does not perform validity checking on date subparts or permit the transition point for adding the century to be specified. A more general tool would be more useful. The next script, *cvt_date.py*, extends the capabilities of *isoize_date.py*; it recognizes input dates in ISO, US, or British formats and converts any of them to any other. It also can convert two-digit years to four digits, enable you to specify the conversion transition point, and warn about bad dates. As such, it can be used to preprocess input for loading into MySQL or postprocess data exported from MySQL for use by other programs.

*cvt_date.py* understands the following options:

`--iformat``=`*`format`*, `--oformat``=`*`format`*, `--format``=`*`format`*

Set the date format for input, output, or both. The default *`format`* value is `iso`; *cvt_date.py* also recognizes any string beginning with `us` or `br` as indicating US or British date format.

`--add-century`

Convert two-digit years to four digits.

`--columns``=`*`column_list`*

Convert dates only in the named columns. By default, *cvt_date.py* looks for dates in all columns. If this option is given, *`column_list`* should be a list of one or more column positions or ranges separated by commas. (Ranges can be given as *`m-n`* to specify columns *`m`* through *`n`*.) Positions begin at 1.

`--transition``=`*`n`*

Specify the transition point for two-digit to four-digit year conversions. The default transition point is 70\. This option turns on `--add-century`.

`--warn`

Warn about bad dates. (This option can produce spurious warnings if the dates have two-digit years and you don’t specify `--add-century`, because leap-year testing won’t always be accurate in that case.)

We won’t show the code for *cvt_date.py* here (most of it is taken up with processing command-line options), but you can examine the source for yourself if you like. As an example of how *cvt_date.py* works, suppose that you have a file *newdata.txt* with the following contents:

```

name1   01/01/99    38

name2   12/31/00    40

name3   02/28/13    42

name4   01/02/18    44

```

Running the file through *cvt_date.py* with options indicating that the dates are in US format and that the century should be added produces this result:

```

$ `cvt_date.pl --iformat=us --oformat=br newdata.txt`

name1   1999-01-01  38

name2   2000-12-31  40

name3   2013-02-28  42

name4   2018-01-02  44

```

To produce dates in British format instead with no year conversion, do this:

```

$ `cvt_date.pl --iformat=us --oformat=br newdata.txt`

name1   01-01-99    38

name2   31-12-00    40

name3   28-02-13    42

name4   02-01-18    44

```

*cvt_date.py* has no knowledge of the meaning of each data column, of course. If you have a nondate column with values that match the pattern, it rewrites that column, too. To deal with that, specify a `--columns` option to limit the columns that *cvt_date.py* converts.

*isoize_date.py* and *cvt_date.py* both operate on dates written in all-numeric formats. But dates in datafiles often are written differently, and it may be necessary to write a special-purpose script to process them. Suppose an input file contains dates in the following format (these represent the dates on which US states were admitted to the Union):

```

Delaware        Dec. 7, 1787

Pennsylvania    Dec 12, 1787

New Jersey      Dec. 18, 1787

Georgia         Jan. 2, 1788

Connecticut     Jan. 9, 1788

Massachusetts   Feb. 6, 1788

…

```

The dates consist of a three-character month abbreviation (possibly followed by a period), a numeric day of the month, a comma, and a numeric year. To import this file into MySQL, you must convert the dates to ISO format, resulting in a file that looks like this:

```

Delaware        1787-12-07

Pennsylvania    1787-12-12

New Jersey      1787-12-18

Georgia         1788-01-02

Connecticut     1788-01-09

Massachusetts   1788-02-06

…

```

That’s a somewhat specialized kind of transformation, although this general type of problem (converting a particular date format to ISO format) is hardly uncommon. To perform the conversion, identify the dates as those values matching an appropriate pattern, map month names to the corresponding numeric values, and reformat the result. The following script, *monddyyyy_to_iso.py*, illustrates how:

```

#!/usr/bin/python3

# monddyyyy_to_iso.py: Convert dates from mon[.] dd, yyyy to ISO format.

# Assumes tab-delimited, linefeed-terminated input

import re

import sys

import fileinput

import warnings

map = {"jan": 1, "feb": 2, "mar": 3, "apr": 4, "may": 5, "jun": 6,

    "jul": 7, "aug": 8, "sep": 9, "oct": 10, "nov": 11, "dec": 12

    } # map 3-char month abbreviations to numeric month

for line in fileinput.input(sys.argv[1:]):

values = line.rstrip().split("\t", 10000)    # split, preserving all fields

for i in range(0, len(values)):

    # reformat the value if it matches the pattern, otherwise assume

    # that it's not a date in the required format and leave it alone

    m = re.match('^([^.]+)\.? (\d+), (\d+)$', values[i])

    if m:

    # use lowercase month name

    (month, day, year) = (m.group(1).lower(), int(m.group(2)), int(m.group(3)))

#@ _CHECK_VALIDITY_

    if month in map:

#@ _CHECK_VALIDITY_

        values[i] = "%04d-%02d-%02d" % (year, map[month], day)

    else:

        # warn, but don't reformat

        warnings.warn("%s bad date?" % (values[i]))

print("\t".join(values))

```

The script only does reformatting, it doesn’t validate the dates. To do that, modify the script to use the *cookbook_utils.py* module by adding this statement in the beginning of the script:

```

from cookbook_utils import *

```

That gives the script access to the module’s `is_valid_date()` routine. To use it, change this line:

```

if month in map:

```

To this:

```

if month in map and is_valid_date(year, map[month], day)):

```

# 14.16 Importing Non-ISO Date Values

## Problem

You want to import date values, but they are not in the ISO (*`YYYY-MM-DD`*) format that MySQL expects.

## Solution

Use an external utility to convert the dates to ISO format before importing the data into MySQL (*cvt_date.py* is useful here). Or use `LOAD` `DATA`’s capability for preprocessing input data prior to loading it into the database.

## Discussion

Suppose that a table contains three columns, `name`, `date`, and `value`, where `date` is a `DATE` column requiring values in ISO format (*`YYYY-MM-DD`*). Suppose also that you’re given a datafile *newdata.txt* to be imported into the table, but its contents look like this:

```

name1   01/01/99    38

name2   12/31/00    40

name3   02/28/13    42

name4   01/02/18    44

```

The dates are in *`MM/DD/YY`* format and must be converted to ISO format to be stored as `DATE` values in MySQL. One way to do this is to run the file through the *cvt_date.py* script from Recipe 14.15:

```

$ `cvt_date.py --iformat=us --add-century newdata.txt > tmp.txt`

```

Then load the *tmp.txt* file into the table. This task also can be accomplished entirely in MySQL with no external utilities by using SQL to perform the reformatting operation. As discussed in Recipe 13.1, `LOAD` `DATA` can preprocess input values before inserting them. Applying that capability to the present problem, the date-rewriting `LOAD` `DATA` statement looks like this, using the `STR_TO_DATE()` function (see Recipe 8.3) to interpret the input dates:

```

mysql> `LOAD DATA LOCAL INFILE 'newdata.txt'`

    -> `INTO TABLE t (name,@date,value)`

    -> `SET date = STR_TO_DATE(@date,'%m/%d/%y');`

```

With the `%y` format specifier in `STR_TO_DATE()`, MySQL converts the two-digit years to four-digit years automatically, so the original *`MM/DD/YY`* values end up as ISO values in *`YYYY-MM-DD`* format. The resulting data after import looks like this:

```

+-------+------------+-------+

| name  | date       | value |
| --- | --- | --- |

+-------+------------+-------+

| name1 | 1999-01-01 |    38 |
| --- | --- | --- |
| name2 | 2000-12-31 |    40 |
| name3 | 2013-02-28 |    42 |
| name4 | 2018-01-02 |    44 |

+-------+------------+-------+

```

This procedure assumes that MySQL’s automatic conversion of two-digit years to four digits produces the correct century values. This means that the year part of the values must correspond to years in the range from 1970 to 2069\. If that’s not true, you must convert the year values some other way. (For some ideas on how to do this, see Recipe 14.14.)

If the dates are not in a format that `STR_TO_DATE()` can interpret, perhaps you can write a stored function to handle them and return ISO date values. In that case, the `LOAD` `DATA` statement looks like this, where `my_date_interp()` is your stored function name:

```

mysql> `LOAD DATA LOCAL INFILE 'newdata.txt'`

    -> `INTO TABLE t (name,@date,value)`

    -> `SET date = my_date_interp(@date);`

```

# 14.17 Exporting Dates Using Non-ISO Formats

## Problem

You want to export date values using a format other than MySQL’s default ISO (*`YYYY-MM-DD`*) format. This might be a requirement when exporting dates from MySQL to applications that don’t use ISO format.

## Solution

Use an external utility to rewrite the dates to non-ISO format after exporting the data from MySQL (*cvt_date.py* is useful here). Or use the `DATE_FORMAT()` function to rewrite the values during the export operation.

## Discussion

Suppose that you want to export data from MySQL into an application that doesn’t understand ISO-format dates. One way to do this is to export the data into a file, leaving the dates in ISO format. Then run the file through a utility such as *cvt_date.py* that rewrites the dates into the required format (see Recipe 14.15).

Another approach is to export the dates directly in the required format by rewriting them with `DATE_FORMAT()`. Suppose that you have the following table:

```

CREATE TABLE datetbl

(

i   INT,

c   CHAR(10),

d   DATE,

dt  DATETIME,

ts  TIMESTAMP,

PRIMARY KEY(i)

);

```

Suppose also that you need to export data from this table, but with the dates in any `DATE`, `DATETIME`, or `TIMESTAMP` columns rewritten in US format (*`MM-DD-YYYY`*). A `SELECT` statement that uses the `DATE_FORMAT()` function to rewrite the dates as required looks like this:

```

SELECT

i,

c,

DATE_FORMAT(d, '%m-%d-%Y') AS d,

DATE_FORMAT(dt, '%m-%d-%Y %T') AS dt,

DATE_FORMAT(ts, '%m-%d-%Y %T') AS ts

FROM datetbl;

```

If `datetbl` contains the following rows:

```

3       abc     2005-12-31      2005-12-31 12:05:03     2005-12-31 12:05:03

4       xyz     2006-01-31      2006-01-31 12:05:03     2006-01-31 12:05:03

```

The statement generates output that looks like this:

```

3       abc     12-31-2005      12-31-2005 12:05:03     12-31-2005 12:05:03

4       xyz     01-31-2006      01-31-2006 12:05:03     01-31-2006 12:05:03

```

# 14.18 Pre-processing and Importing a File

## Problem

Recall the scenario presented at the beginning of Chapter 13:

Suppose that a file named *somedata.csv* contains 12 data columns in comma-separated values (CSV) format. From this file you want to extract only columns 2, 11, 5, and 9, and use them to create database rows in a MySQL table that contains <q>name</q>, <q>birth</q>, <q>height</q>, and <q>weight</q> columns. You must make sure that the height and weight are positive integers, and convert the birth dates from *`MM/DD/YY`* format to *`YYYY-MM-DD`* format.

## Solution

Combine techniques that we discussed in Chapter 13 and this chapter.

## Discussion

Much of the work can be done using the utility programs developed in this chapter. Convert the file to tab-delimited format with *cvt_file.pl*, extract the columns in the desired order with *yank_col.pl*, and rewrite the date column to ISO format with *cvt_date.py* (see Recipe 14.15):

```

$ `cvt_file.pl --iformat=csv somedata.csv \`

    `| yank_col.pl --columns=2,11,5,9 \`

    `| cvt_date.py --columns=2 --iformat=us --add-century > tmp`

```

The resulting file, *tmp*, has four columns representing the `name`, `birth`, `height`, and `weight` values, in that order. It needs only to have its height and weight columns checked to make sure they contain positive integers. Using the `is_positive_integer()` library function from the *cookbook_utils.py* module file, that task can be achieved using a short special-purpose script that is little more than an input loop:

```

#!/usr/bin/python3

# validate_htwt.py: Height/weight validation example.

# Assumes tab-delimited, linefeed-terminated input lines.

# Input columns and the actions to perform on them are as follows:

# 1: name; echo as given

# 2: birth; echo as given

# 3: height; validate as positive integer

# 4: weight; validate as positive integer

import sys

import fileinput

import warnings

from cookbook_utils import *

line_num = 0

for line in fileinput.input(sys.argv[1:]):

line_num += 1

(name, birth, height, weight) = line.rstrip().split ("\t", 4)

if not is_positive_integer(height):

    warnings.warn(f"line {line_num}:height {height} is not a positive integer")

if not is_positive_integer(weight):

    warnings.warn(f"line {line_num}:weight {weight} is not a positive integer")

```

The *validate_htwt.py* script produces no output (except for warning messages) because it need not reformat any of the input values. If *tmp* passes validation with no errors, it can be loaded into MySQL with a simple `LOAD` `DATA` statement:

```

mysql> `LOAD DATA LOCAL INFILE 'tmp' INTO TABLE` *`tbl_name`*`;`

```
