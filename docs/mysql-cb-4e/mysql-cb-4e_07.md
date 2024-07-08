# 第七章：使用字符串

# 7.0 Introduction

与大多数数据类型一样，字符串值可以进行相等性或不等性比较，或相对顺序比较。然而，字符串有额外的属性需要考虑：

+   字符串可以是二进制或非二进制的。二进制字符串用于原始数据，如图像、音乐文件或加密值。非二进制字符串用于字符数据，如文本，并与字符集和排序规则（排序顺序）相关联。

+   字符集确定字符串中合法的字符。根据是否需要比较大小写敏感或不敏感，或使用特定语言的规则，可以选择排序规则。

+   二进制字符串的数据类型为`BINARY`、`VARBINARY`和`BLOB`。非二进制字符串的数据类型为`CHAR`、`VARCHAR`和`TEXT`，每种类型都允许使用`CHARACTER` `SET`和`COLLATE`属性。

+   您可以将二进制字符串转换为非二进制字符串，反之亦然，或将非二进制字符串从一个字符集或排序规则转换为另一个字符集或排序规则。

+   您可以使用字符串的全部内容或从中提取子字符串。字符串可以与其他字符串组合。

+   您可以对字符串应用模式匹配操作。

+   可以对大量文本进行高效查询的全文搜索功能可用。

本章讨论如何利用这些属性，以便根据应用程序的任何要求存储、检索和操作字符串。

用于创建本章使用的表的脚本位于`recipes`发行版的*tables*目录中。

# 7.1 字符串属性

一个字符串的属性是它是二进制的还是非二进制的：

+   二进制字符串是字节序列。它可以包含任何类型的信息，如图像、MP3 文件或压缩或加密数据。即使存储看起来像普通文本的值（例如`abc`），二进制字符串也不与字符集关联。二进制字符串按字节使用数值字节值进行逐字节比较。

+   非二进制字符串是字符序列。它存储具有特定字符集和排序规则的文本。字符集定义了可以存储在字符串中的字符。排序规则定义了字符的排序顺序，影响比较和排序操作。

要查看非二进制字符串可用的字符集，请使用以下语句：

```
mysql> `SHOW CHARACTER SET;`
+----------+-----------------------------+---------------------+--------+
| Charset  | Description                 | Default collation   | Maxlen |
+----------+-----------------------------+---------------------+--------+
| big5     | Big5 Traditional Chinese    | big5_chinese_ci     |      2 |
…
| koi8r    | KOI8-R Relcom Russian       | koi8r_general_ci    |      1 |
| latin1   | cp1252 West European        | latin1_swedish_ci   |      1 |
| latin2   | ISO 8859-2 Central European | latin2_general_ci   |      1 |
…
| utf8     | UTF-8 Unicode               | utf8_general_ci     |      3 |
| utf8mb4  | UTF-8 Unicode               | utf8mb4_0900_ai_ci  |      4 |
…
```

MySQL 8.0 中的默认字符集为`utf8mb4`，排序规则为`utf8mb4_0900_ai_ci`。如果必须在单个列中存储多种语言的字符，请考虑使用 Unicode 字符集之一（例如`utf8mb4`或`utf16`），因为它们可以表示多种语言的字符。

一些字符集只包含单字节字符，而其他字符集则允许多字节字符。一些多字节字符集中的字符长度各不相同，而另一些则所有字符长度固定。例如，Unicode 数据可以使用 `utf8mb4` 字符集存储，其中字符长度可以从一到四个字节不等，或者使用 `utf16` 字符集存储，其中所有字符长度均为两个字节。

###### 注意

在 MySQL 中，要使用包括基本多语言平面（BMP）之外的补充字符在内的完整 Unicode 字符集，请使用 `utf8mb4`。在 `utf8mb4` 中，字符的字节长度可以从一到四个字节不等。其他包括补充字符的 Unicode 字符集还有 `utf16`、`utf16le` 和 `utf32`。

要确定给定字符串是否包含多字节字符，请使用 `LENGTH()` 和 `CHAR_LENGTH()` 函数，它们分别返回字符串的字节长度和字符长度。如果对于给定字符串，`LENGTH()` 大于 `CHAR_LENGTH()`，则表示存在多字节字符：

+   `utf8` Unicode 字符集包含多字节字符，但给定的 `utf8` 字符串可能只包含单字节字符，例如以下示例：

    ```
    mysql> `SET @s = CONVERT('abc' USING utf8mb4);`
    mysql> `SELECT LENGTH(@s), CHAR_LENGTH(@s);`
    +------------+-----------------+
    | LENGTH(@s) | CHAR_LENGTH(@s) |
    +------------+-----------------+
    |          3 |               3 |
    +------------+-----------------+
    ```

+   对于 `utf16` Unicode 字符集，所有字符均使用两个字节编码，即使它们在其他字符集如 `latin1` 中是单字节字符。因此，每个 `utf16` 字符串都包含多字节字符：

    ```
    mysql> `SET @s = CONVERT('abc' USING utf16);`
    mysql> `SELECT LENGTH(@s), CHAR_LENGTH(@s);`
    +------------+-----------------+
    | LENGTH(@s) | CHAR_LENGTH(@s) |
    +------------+-----------------+
    |          6 |               3 |
    +------------+-----------------+
    ```

非二进制字符串的另一个属性是排序规则（collation），它确定字符集中字符的排序顺序。使用 `SHOW COLLATION` 查看所有可用的排序规则；添加 `LIKE` 子句以查看特定字符集的排序规则：

```
mysql> `SHOW COLLATION LIKE 'utf8mb4%';`
+----------------------------+---------+-----+---------+----------+---------+---------------+
| Collation                  | Charset | Id  | Default | Compiled | Sortlen | Pad_attribute |
+----------------------------+---------+-----+---------+----------+---------+---------------+
| utf8mb4_0900_ai_ci         | utf8mb4 | 255 | Yes     | Yes      |       0 | NO PAD        |
| utf8mb4_0900_as_ci         | utf8mb4 | 305 |         | Yes      |       0 | NO PAD        |
..
| utf8mb4_es_0900_ai_ci      | utf8mb4 | 263 |         | Yes      |       0 | NO PAD        |
| utf8mb4_es_0900_as_cs      | utf8mb4 | 286 |         | Yes      |       0 | NO PAD        |
| utf8mb4_es_trad_0900_ai_ci | utf8mb4 | 270 |         | Yes      |       0 | NO PAD        |
| utf8mb4_es_trad_0900_as_cs | utf8mb4 | 293 |         | Yes      |       0 | NO PAD        |
..
| utf8mb4_tr_0900_ai_ci      | utf8mb4 | 265 |         | Yes      |       0 | NO PAD        |
| utf8mb4_tr_0900_as_cs      | utf8mb4 | 288 |         | Yes      |       0 | NO PAD        |
| utf8mb4_turkish_ci         | utf8mb4 | 233 |         | Yes      |       8 | PAD SPACE     |
| utf8mb4_unicode_520_ci     | utf8mb4 | 246 |         | Yes      |       8 | PAD SPACE     |
```

在没有明确指定排序规则的情况下，给定字符集中的字符串使用 `Default` 列中带有 `Yes` 的排序规则。如图所示，`utf8mb4` 的默认排序规则是 `utf8mb4_0900_ai_ci`。（默认排序规则也可以通过 `SHOW CHARACTER SET` 显示。）

排序规则可以是区分大小写的（`a` 和 `A` 是不同的）、不区分大小写的（`a` 和 `A` 是相同的）、或二进制的（两个字符根据它们的数值是否相等来判断是相同还是不同）。以 `_ci`、`_cs` 或 `_bin` 结尾的排序规则名分别表示不区分大小写、区分大小写或二进制。

二进制字符串和二进制排序规则都使用数值。不同之处在于，二进制字符串比较总是基于单字节单位，而二进制排序规则使用*字符*数值比较非二进制字符串；根据字符集，其中一些可能是多字节值。

下面的示例说明了排序规则如何影响排序顺序。假设一个表包含一个 `utf8mb4` 字符串列，并具有以下行：

```
mysql> `CREATE TABLE t (c CHAR(3) CHARACTER SET utf8mb4);`
mysql> `INSERT INTO t (c) VALUES('AAA'),('bbb'),('aaa'),('BBB');`
mysql> `SELECT c FROM t;`
+------+
| c    |
+------+
| AAA  |
| bbb  |
| aaa  |
| BBB  |
+------+
```

通过在列上应用 `COLLATE` 运算符，可以选择用于排序的排序规则，从而影响结果的顺序：

+   大小写不敏感排序将`a`和`A`放在一起，并将它们放在`b`和`B`之前。但是，对于给定的字母，它不一定将一个字母的大小写顺序排在另一个字母的前面，如下结果所示：

    ```
    mysql> `SELECT c FROM t ORDER BY c COLLATE utf8mb4_turkish_ci;`
    +------+
    | c    |
    +------+
    | AAA  |
    | aaa  |
    | bbb  |
    | BBB  |
    +------+
    ```

+   大小写敏感排序将`A`和`a`放在`B`和`b`之前，并将小写字母排序在大写字母之前：

    ```
    mysql> `SELECT c FROM t ORDER BY c COLLATE utf8mb4_tr_0900_as_cs;`
    +------+
    | c    |
    +------+
    | aaa  |
    | AAA  |
    | bbb  |
    | BBB  |
    +------+
    ```

+   二进制排序使用字符的数字值进行排序。假设大写字母的数字值小于小写字母的数字值，则二进制排序结果如下：

    ```
    mysql> `SELECT c FROM t ORDER BY c COLLATE utf8mb4_bin;`
    +------+
    | c    |
    +------+
    | AAA  |
    | BBB  |
    | aaa  |
    | bbb  |
    +------+
    ```

    请注意，由于不同大小写字符具有不同的数字值，二进制排序会产生大小写敏感的顺序。但是，该顺序与大小写敏感排序不同。

如果您要求比较和排序操作使用特定语言的排序规则，请选择特定语言的排序。例如，如果您使用`utf8mb4`字符集存储字符串，则默认排序（`utf8mb4_0900_ai_ci`）将`ch`和`ll`视为两个字符的字符串。要使用传统的西班牙排序（将`ch`和`ll`视为跟随`c`和`l`的单个字符），请指定`utf8mb4_spanish2_ci`排序。这两种排序产生不同的结果，如下所示：

```
mysql> `CREATE TABLE t (c CHAR(2) CHARACTER SET utf8mb4);`
mysql> `INSERT INTO t (c) VALUES('cg'),('ch'),('ci'),('lk'),('ll'),('lm');`
mysql> `SELECT c FROM t ORDER BY c COLLATE utf8mb4_general_ci;`
+------+
| c    |
+------+
| cg   |
| ch   |
| ci   |
| lk   |
| ll   |
| lm   |
+------+
mysql> `SELECT c FROM t ORDER BY c COLLATE utf8mb4_spanish2_ci;`
+------+
| c    |
+------+
| cg   |
| ci   |
| ch   |
| lk   |
| lm   |
| ll   |
+------+
```

如果您未使用默认排序，则最好在列定义中设置排序，方法如下；

```
mysql> `CREATE TABLE t (c CHAR(2) CHARACTER SET utf8mb4 COLLATE utf8mb4_spanish2_ci);`
```

这将确保在排序操作期间避免使用错误的排序时可能导致的查询性能下降。

# 7.2 选择字符串数据类型

## 问题

您希望存储字符串数据，但不确定哪种数据类型最合适。

## 解决方案

根据要存储的信息的特征和您需要使用它的方式选择数据类型。考虑以下问题：

+   字符串是二进制还是非二进制？

+   大小写是否重要？

+   最大字符串长度是多少？

+   您是否希望存储固定长度还是可变长度的值？

+   您是否需要保留尾随空格？

+   是否有一组固定的允许值？

## 讨论

MySQL 提供多种二进制和非二进制字符串数据类型。这些类型成对出现，如下表所示。最大长度以字节为单位，无论类型是二进制还是非二进制。对于非二进制类型，包含多字节字符的字符串的最大*字符*数较少，如我们在表 7-1 中所示

表 7-1\. 每种数据类型的最大字符数

| 二进制数据类型 | 非二进制数据类型 | 最大长度 |
| --- | --- | --- |
| `BINARY` | `CHAR` | 255 |
| `VARBINARY` | `VARCHAR` | 65,535 |
| `TINYBLOB` | `TINYTEXT` | 255 |
| `BLOB` | `TEXT` | 65,535 |
| `MEDIUMBLOB` | `MEDIUMTEXT` | 16,777,215 |
| `LONGBLOB` | `LONGTEXT` | 4,294,967,295 |

对于`BINARY`和`CHAR`数据类型，MySQL 使用固定宽度存储列值。例如，在`BINARY(10)`或`CHAR(10)`列中存储的值始终占据 10 个字节或 10 个字符。存储时必要时会将较短的值填充到所需长度。对于`BINARY`，填充值为`0x00`（零值字节，也称为 ASCII NUL）。`CHAR`值在存储时用空格填充，并且在检索时删除尾随空格。

对于`VARBINARY`、`VARCHAR`以及`BLOB`和`TEXT`类型，MySQL 仅使用所需的存储空间存储值，不超过最大列长度。在存储或检索值时不会添加或删除填充。

要保留存储的原始字符串中存在的尾随填充值，请使用不会执行剥离的数据类型。例如，如果存储可能以空格结尾的字符（非二进制）字符串，并且希望保留它们，请使用`VARCHAR`或其中一个`TEXT`数据类型。以下语句说明了`CHAR`和`VARCHAR`列处理尾随空格的差异：

```
mysql> `DROP TABLE IF EXISTS t;`
mysql> `CREATE TABLE t (c1 CHAR(10), c2 VARCHAR(10));`
mysql> `INSERT INTO t (c1,c2) VALUES('abc       ','abc       ');`
mysql> `SELECT c1, c2, CHAR_LENGTH(c1), CHAR_LENGTH(c2) FROM t;`
+------+------------+-----------------+-----------------+
| c1   | c2         | CHAR_LENGTH(c1) | CHAR_LENGTH(c2) |
+------+------------+-----------------+-----------------+
| abc  | abc        |               3 |              10 |
+------+------------+-----------------+-----------------+
```

这表明，如果将包含尾随空格的字符串存储到`CHAR`列中，检索该值时会删除这些空格。

表可以包含混合的二进制和非二进制字符串列，其非二进制列可以使用不同的字符集和排序规则。声明非二进制字符串列时，如果需要特定的字符集和排序规则，请使用`CHARACTER` `SET`和`COLLATE`属性。例如，如果需要存储`utf8mb4`（Unicode）和`sjis`（日语）字符串，可以像这样定义包含两列的表：

```
CREATE TABLE mytbl
(
  utf8str VARCHAR(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_danish_ci,
  sjisstr VARCHAR(100) CHARACTER SET sjis COLLATE sjis_japanese_ci
);
```

在列定义中，`CHARACTER` `SET`和`COLLATE`子句都是可选的：

+   如果指定了`CHARACTER` `SET`并省略`COLLATE`，则使用字符集的默认排序规则。

+   如果指定了`COLLATE`并省略`CHARACTER` `SET`，则使用排序规则名称隐含的字符集（名称的第一部分）。例如，`utf8mb4_danish_ci`和`sjis_japanese_ci`分别隐含`utf8mb4`和`sjis`。这意味着在前述`CREATE` `TABLE`语句中可以省略`CHARACTER` `SET`属性。

+   如果你同时省略`CHARACTER` `SET`和`COLLATE`，则该列将被赋予表的默认字符集和排序规则。表定义可以在`CREATE` `TABLE`语句的右括号后包含这些属性。如果存在这些属性，则它们适用于没有显式字符集或排序规则的列。如果省略，则表的默认值来自数据库的默认值。您可以在使用`CREATE` `DATABASE`语句创建数据库时指定数据库的默认值。如果省略，则服务器默认值适用于数据库。

MySQL 8.0 的服务器默认字符集和排序规则分别为 `utf8mb4` 和 `utf8mb4_0900_ai_ci`，因此字符串默认使用 `utf8mb4` 字符集并且不区分大小写。要更改此设置，可以在服务器启动时设置 `character_set_server` 和 `collation_server` 系统变量（参见 Recipe 22.1）。

MySQL 还支持 `ENUM` 和 `SET` 字符串类型，用于具有固定允许值集的列。这些数据类型也适用于 `CHARACTER` `SET` 和 `COLLATE` 属性。

# 7.3 设置客户端连接字符集

## 问题

当你执行不使用默认字符集的 SQL 语句或生成查询结果时。

## 解决方案

使用 `SET` `NAMES` 或其等效方法设置连接的适当字符集。

## 讨论

当你在应用程序和服务器之间发送信息时，可能需要告诉 MySQL 使用适当的字符集。例如，默认字符集是 `latin1`，但这可能不是连接到服务器的正确字符集。如果有希腊数据，使用 `latin1` 在屏幕上显示将会产生乱码。如果在 `utf8mb4` 字符集中使用 Unicode 字符串，`latin1` 可能无法表示所有你可能需要的字符。

要解决此问题，请配置连接以使用适当的字符集。有几种方法可以实现这一点：

+   在连接后执行 `SET` `NAMES` 语句：

    ```
    mysql> `SET NAMES 'utf8mb4';`
    ```

    `SET` `NAMES` 允许指定连接排序规则：

    ```
    mysql> `SET NAMES 'utf8mb4' COLLATE 'utf8mb4_0900_ai_ci';`
    ```

+   如果你的客户端程序支持 `--default-character-set` 选项，可以在程序调用时使用它指定字符集。*mysql* 就是这样的程序。将选项放入选项文件中，以便每次连接到服务器时生效：

    ```
    [mysql]
    default-character-set=utf8mb4
    ```

+   如果你在 Unix 上使用 `LANG` 或 `LC_ALL` 环境变量或在 Windows 上设置代码页，则 MySQL 客户端程序会自动检测要使用的字符集。例如，将 `LC_ALL` 设置为 `en_US.UTF-8` 会导致诸如 *mysql* 的程序使用 `utf8`。

+   一些编程接口提供了它们自己设置字符集的方法。例如，MySQL Connector/J 用于 Java 客户端会在连接时自动检测服务器端使用的字符集，但你也可以在连接 URL 中明确指定不同的集合，使用 `characterEncoding` 属性。该属性的值应为 Java 风格的字符集名称。要选择 `utf8mb4`，你可以使用如下连接 URL：

    ```
    jdbc:mysql://localhost/cookbook?characterEncoding=UTF-8
    ```

    这比使用 `SET` `NAMES` 更可取，因为 Connector/J 会代表应用程序执行字符集转换，但如果使用 `SET` `NAMES`，它不知道适用哪个字符集。对于其他 API 编写的程序也适用类似原则。对于 PHP Data Objects (PDO)，在数据源名称 (DSN) 字符串中使用 `charset` 选项（在 PHP 5.3.6 或更高版本中有效）：

    ```
    $dsn = "mysql:host=localhost;dbname=cookbook;charset=utf8mb4";
    ```

    对于 Connector/Python，请指定`charset`连接参数：

    ```
    conn_params = {
      "database": "cookbook",
      "host": "localhost",
      "user": "cbuser",
      "password": "cbpass",
      "charset": "utf8mb4",
    }
    ```

    对于 Go，请指定`charset`连接参数：

    ```
    db, err := sql.Open("mysql",
              "cbuser:cbpass@tcp(127.0.0.1:3306)/cookbook?charset=utf8mb4")
    ```

    一些 API 也可以提供参数以指定校对顺序。

###### 注意

一些字符集不能用作连接字符集：`utf16`，`utf16le`，`utf32`。

您还应确保显示设备使用的字符集与您用于 MySQL 的字符集匹配。否则，即使 MySQL 正确处理数据，它也可能显示为垃圾。假设您在终端窗口中使用*mysql*程序，并配置 MySQL 使用`utf8mb4`存储`utf8mb4`编码的土耳其数据。如果将终端窗口设置为使用`euc-tr`编码（也是土耳其语），但其土耳其字符的编码与`utf8mb4`不同，数据将不会按预期显示。（如果使用自动检测，这不应该是一个问题。）

在一个示例中，插入表中的土耳其字符将在使用不同字符集进行连接时显示为乱码。

```
mysql> `DROP TABLE IF EXISTS t;`
mysql> `CREATE TABLE t (c CHAR(3) CHARACTER SET utf8mb4);`
mysql> `INSERT INTO t (c) VALUES('iii'),('şşş'),('ööö'),('ççç');`
```

使用`Latin1`客户端字符集从另一个连接将得到以下结果。

```
mysql> `\s`
--------------
mysql  Ver 8.0.27 for Linux on x86_64 (MySQL Community Server - GPL)
...
Server characterset:    utf8mb4
Db     characterset:    utf8mb4
Client characterset:    latin1
Conn.  characterset:    latin1
...
`SELECT c from t;`
+------+
| c    |
+------+
| iii  |
| ???  |
| ���  |
| ���  |
+------+
```

要验证您是否连接到 MySQL 命令行界面并显示正确的字符集，请执行以下操作。

```
mysql> `\s`
--------------
mysql  Ver 8.0.27 for Linux on x86_64 (MySQL Community Server - GPL)
...
Server characterset:	utf8mb4
Db     characterset:	utf8mb4
Client characterset:	utf8mb4
Conn.  characterset:	utf8mb4
...
`SELECT c from t;`
+------+
| c    |
+------+
| iii  |
| şşş  |
| ööö  |
| ççç  |
+------+
```

# 7.4 编写字符串文字

## 问题

您需要在 SQL 语句中编写文字字符串。

## 解决方案

学习规定字符串值的语法规则。

## 讨论

你可以按多种方式编写字符串：

+   将字符串的文本放在单引号或双引号中：

    ```
    'my string'
    "my string"
    ```

    当启用`ANSI_QUOTES` SQL 模式时，不能使用双引号引用字符串：服务器会将双引号解释为标识符（如表或列名）的引用字符，而不是字符串（见 Recipe 4.6）。如果采用始终使用单引号编写带引号的字符串的约定，MySQL 会将它们解释为字符串，而不管`ANSI_QUOTES`设置如何。

+   使用十六进制表示法。每对十六进制数字产生字符串的一个字节。`abcd`可以用以下任一格式写入：

    ```
    0x61626364
    X'61626364'
    x'61626364'
    ```

    MySQL 将使用十六进制表示法编写的字符串视为二进制字符串。不巧的是，应用程序在构造引用二进制值的 SQL 语句时通常使用十六进制字符串：

    ```
    INSERT INTO t SET binary_col = 0xdeadbeef;
    ```

+   要指定用于解释文字字符串的字符集，请使用由下划线引导的字符集名称：

    ```
    _utf8mb4 'abcd'
    _utf16 'abcd'
    ```

    引导符告诉服务器如何解释其后的字符串。对于 `_utf8mb4` `'abcd'`，服务器生成由四个单字节字符组成的字符串。对于 `_ucs2` `'abcd'`，服务器生成由两个双字节字符组成的字符串，因为`ucs2`是双字节字符集。

为确保字符串是二进制字符串或非二进制字符串具有特定字符集或校对顺序，请使用 Recipe 7.5 中给出的字符串转换说明。

当执行一个 API 或者在*mysql*批处理模式中运行时，包含相同引号字符的引用字符串会产生语法错误：

```
`mysql -e "SELECT 'I'm asleep'"`
ERROR 1064 (42000) at line 1: You have an error in your SQL syntax; ↩
check the manual that corresponds to your MySQL server version ↩
for the right syntax to use near 'asleep'' at line 1
```

如果通过*mysql*客户端交互执行，则会等待闭合引号。

```
mysql> `SELECT` `'I'``m` `asleep``';` '>
    '> `'``\``c`
mysql>
```

您有几种方法可以处理这个问题：

+   将包含单引号的字符串放在双引号内（假设未启用 `ANSI_QUOTES`），或将包含双引号的字符串放在单引号内。

    ```
    mysql> `SELECT "I'm asleep", 'He said, "Boo!"';`
    +------------+-----------------+
    | I'm asleep | He said, "Boo!" |
    +------------+-----------------+
    | I'm asleep | He said, "Boo!" |
    +------------+-----------------+
    ```

+   要在同一种引号内的字符串中包含引号字符，可以将引号字符加倍或在其前面加上反斜杠。当 MySQL 读取语句时，它会剥离额外的引号或反斜杠：

    ```
    mysql> `SELECT 'I''m asleep', 'I\'m wide awake';`
    +------------+----------------+
    | I'm asleep | I'm wide awake |
    +------------+----------------+
    | I'm asleep | I'm wide awake |
    +------------+----------------+
    mysql> `SELECT "He said, ""Boo!""", "And I said, \"Yikes!\"";`
    +-----------------+----------------------+
    | He said, "Boo!" | And I said, "Yikes!" |
    +-----------------+----------------------+
    | He said, "Boo!" | And I said, "Yikes!" |
    +-----------------+----------------------+
    ```

    反斜杠会关闭后续字符的任何特殊含义，包括它本身。要在字符串中写入文字反斜杠，请将其加倍：

    ```
    mysql> `SELECT 'Install MySQL in C:\\mysql on Windows';`
    +--------------------------------------+
    | Install MySQL in C:\mysql on Windows |
    +--------------------------------------+
    | Install MySQL in C:\mysql on Windows |
    +--------------------------------------+
    ```

    反斜杠会暂时关闭正常字符串处理规则，因此称为转义序列的序列，例如 `\'`、`\"` 和 `\\`。MySQL 还识别 `\b`（退格）、`\n`（换行，也称为换行符）、`\r`（回车）、`\t`（制表符）和 `\0`（ASCII NUL）等其他转义序列。

+   将字符串写成十六进制值：

    ```
    mysql> `SELECT 0x49276D2061736C656570;`
    +------------------------+
    | 0x49276D2061736C656570 |
    +------------------------+
    | I'm asleep             |
    +------------------------+
    ```

###### 警告

从 8.0 版本开始，默认情况下*mysql*客户端将使用选项 `--binary-as-hex` 运行。如果不禁用此选项，您将以十六进制值获取二进制输出。例如，对于上述命令，您将看到：

```
mysql> `SELECT` `0``x49276D2061736C656570``;`
+------------------------------------------------+ | 0x49276D2061736C656570                         |
+------------------------------------------------+ | 0x49276D2061736C656570                         |
+------------------------------------------------+ 1 row in set (0,00 sec)
```

要获取可读的人类输出，请使用选项 `--binary-as-hex=0` 启动*mysql*客户端。

## 另请参阅

如果您从程序内执行 SQL 语句，可以通过符号引用字符串或二进制值，并让编程接口处理引用：使用语言数据库访问 API 提供的占位符机制（参见 Recipe 4.5）。另外，可以使用 `LOAD_FILE()` 函数从文件中加载诸如图像之类的二进制值（参见 [MySQL documentation](https://dev.mysql.com/doc/refman/8.0/en/string-functions.html#function_load-file)）。

# 7.5 检查或更改字符串的字符集或排序规则

## 问题

您想了解字符串的字符集或排序规则，或者将一个字符串转换为其他字符集或排序规则。

## 解决方案

要检查字符串的字符集或排序规则，请使用 `CHARSET()` 或 `COLLATION()` 函数。要更改其字符集，请使用 `CONVERT()` 函数。要更改其排序规则，请使用 `COLLATE` 操作符。

## 讨论

对于如下创建的表，您知道存储在列 `c` 中的值具有字符集 `utf8` 和排序规则 `utf8_danish_ci`：

```
CREATE TABLE t (c CHAR(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_danish_ci);
```

但有时不太清楚哪个字符集或排序规则适用于字符串。服务器配置可能会影响字面字符串和某些字符串函数，而其他字符串函数则会以特定字符集返回值。当出现排序规则不匹配错误以及比较操作失败时，或者字母大小写转换无法正常工作时，表明您可能使用了错误的字符集或排序规则。

要确定字符串的字符集或排序规则，请使用`CHARSET()`或`COLLATION()`函数。例如，您知道`USER()`函数返回一个 Unicode 字符串吗？

```
mysql> `SELECT USER(), CHARSET(USER()), COLLATION(USER());`
+------------------+-----------------+-------------------+
| USER()           | CHARSET(USER()) | COLLATION(USER()) |
+------------------+-----------------+-------------------+
| cbuser@localhost | utf8mb3         | utf8_general_ci   |
+------------------+-----------------+-------------------+
```

如果字符串值根据当前客户端配置获取其字符集和排序规则，则如果配置更改，则这些属性可能会改变。对于文字字符串，这是正确的：

```
mysql> `SET NAMES 'utf8mb4';`
mysql> `SELECT CHARSET('abc'), COLLATION('abc');`
+----------------+--------------------+
| CHARSET('abc') | COLLATION('abc')   |
+----------------+--------------------+
| utf8mb4         | utf8mb4_0900_ai_ci|
+----------------+--------------------+
mysql> `SET NAMES 'utf8mb4' COLLATE 'utf8mb4_bin';`
mysql> `SELECT CHARSET('abc'), COLLATION('abc');`
+----------------+------------------+
| CHARSET('abc') | COLLATION('abc') |
+----------------+------------------+
| utf8mb4        | utf8mb4_bin      |
+----------------+------------------+
```

对于二进制字符串，`CHARSET()`或`COLLATION()`函数返回一个`binary`值，这意味着字符串是基于数字字节值而不是字符排序值进行比较和排序的。

要将一个字符集转换为另一个字符集，请使用`CONVERT()`函数：

```
mysql> `SET @s1 = _latin1 'my string', @s2 = CONVERT(@s1 USING utf8mb4);`
mysql> `SELECT CHARSET(@s1), CHARSET(@s2);`
+--------------+--------------+
| CHARSET(@s1) | CHARSET(@s2) |
+--------------+--------------+
| latin1       | utf8mb4      |
+--------------+--------------+
```

要更改字符串的排序规则，请使用`COLLATE`运算符：

```
mysql> `SET @s1 = _latin1 'my string', @s2 = @s1 COLLATE latin1_spanish_ci;`
mysql> `SELECT COLLATION(@s1), COLLATION(@s2);`
+-------------------+-------------------+
| COLLATION(@s1)    | COLLATION(@s2)    |
+-------------------+-------------------+
| latin1_swedish_ci | latin1_spanish_ci |
+-------------------+-------------------+
```

新的排序规则必须适合字符串的字符集。例如，您可以将`utf8_general_ci`排序规则与`utf8mb3`字符串一起使用，但不能与`latin1`字符串一起使用：

```
mysql> `SELECT _latin1 'abc' COLLATE utf8_bin;`
ERROR 1253 (42000): COLLATION 'utf8_bin' is not valid for
CHARACTER SET 'latin1'
```

要同时转换字符串的字符集和排序规则，请使用`CONVERT()`来改变字符集，并对结果应用`COLLATE`运算符：

```
mysql> `SET @s1 = _latin1 'my string';`
mysql> `SET @s2 = CONVERT(@s1 USING utf8mb4) COLLATE utf8mb4_spanish_ci;`
mysql> `SELECT CHARSET(@s1), COLLATION(@s1), CHARSET(@s2), COLLATION(@s2);`
+--------------+-------------------+--------------+-----------------+
| CHARSET(@s1) | COLLATION(@s1)    | CHARSET(@s2) | COLLATION(@s2)  |
+--------------+-------------------+--------------+-----------------+
| latin1       | latin1_swedish_ci | utf8         | utf8_spanish_ci |
+--------------+-------------------+--------------+-----------------+
```

`CONVERT()`函数还可以将二进制字符串转换为非二进制字符串，反之亦然。要生成一个二进制字符串，请使用`binary`；任何其他字符集名称都会生成一个非二进制字符串：

```
mysql> `SET @s1 = _latin1 'my string';`
mysql> `SET @s2 = CONVERT(@s1 USING binary);`
mysql> `SET @s3 = CONVERT(@s2 USING utf8mb4);`
mysql> `SELECT CHARSET(@s1), CHARSET(@s2), CHARSET(@s3);`
+--------------+--------------+--------------+
| CHARSET(@s1) | CHARSET(@s2) | CHARSET(@s3) |
+--------------+--------------+--------------+
| latin1       | binary       | utf8mb4      |
+--------------+--------------+--------------+
```

或者，使用`CAST`函数生成二进制字符串，其等效于`CONVERT(`*`str`* `USING` `binary)`：

```
mysql> `SELECT CHARSET(CAST(_utf8mb4 'my string' AS binary));`
+-----------------------------------------------+
| CHARSET(CAST(_utf8mb4 'my string' AS binary)) |
+-----------------------------------------------+
| binary                                        |
+-----------------------------------------------+
```

另请参阅 Recipe 7.3 获取有关字符集的更多信息。

# 7.6 转换字符串的大小写

## 问题

您想要将一个字符串转换为大写或小写。

## 解决方案

使用`UPPER()`或`LOWER()`函数。如果它们不起作用，您可能正在尝试转换一个二进制字符串。将其转换为具有字符集和排序规则并且受大小写映射影响的非二进制字符串。

## 讨论

`UPPER()`和`LOWER()`函数可以转换字符串的大小写：

```
mysql> `SELECT thing, UPPER(thing), LOWER(thing) FROM limbs;`
+--------------+--------------+--------------+
| thing        | UPPER(thing) | LOWER(thing) |
+--------------+--------------+--------------+
| human        | HUMAN        | human        |
| insect       | INSECT       | insect       |
| squid        | SQUID        | squid        |
| fish         | FISH         | fish         |
| centipede    | CENTIPEDE    | centipede    |
| table        | TABLE        | table        |
| armchair     | ARMCHAIR     | armchair     |
| phonograph   | PHONOGRAPH   | phonograph   |
| tripod       | TRIPOD       | tripod       |
| Peg Leg Pete | PEG LEG PETE | peg leg pete |
| space alien  | SPACE ALIEN  | space alien  |
+--------------+--------------+--------------+
```

但是有些字符串是<q>顽固的</q>，抵制大小写转换。要获取可读的输出，请使用选项`binary-as-hex=0`启动 mysql 客户端。

```
mysql> `CREATE TABLE t (b VARBINARY(10)) SELECT 'aBcD' AS b;`
mysql> `SELECT b, UPPER(b), LOWER(b) FROM t;`
+------+----------+----------+
| b    | UPPER(b) | LOWER(b) |
+------+----------+----------+
| aBcD | aBcD     | aBcD     |
+------+----------+----------+
```

这个问题发生在具有`BINARY`或`BLOB`数据类型的字符串上。这些是没有字符集或排序规则的二进制字符串。大小写不适用，`UPPER()`和`LOWER()`不起作用。

要将二进制字符串映射到给定的大小写，将其转换为非二进制字符串，选择一个具有大写和小写字符的字符集。然后，大小写转换函数将按预期工作，因为排序规则提供了大小写映射：

```
mysql> `SELECT b,`
    -> `UPPER(CONVERT(b USING utf8mb4)) AS upper,`
    -> `LOWER(CONVERT(b USING utf8mb4)) AS lower`
    -> `FROM t;`
+------+-------+-------+
| b    | upper | lower |
+------+-------+-------+
| aBcD | ABCD  | abcd  |
+------+-------+-------+
```

例子使用了一个表列，但同样的原则适用于二进制字符串文字和字符串表达式。

如果您不确定字符串表达式是二进制还是非二进制，请使用`CHARSET()`函数进行确认；参见 Recipe 7.5。

要仅转换字符串的部分大小写，请将其分解为片段，转换相关片段，然后将片段组合在一起。假设您只想将字符串的初始字符转换为大写。以下表达式实现了这一点：

```
CONCAT(UPPER(LEFT(*`str`*,1)),MID(*`str`*,2))
```

但每次需要时写出这样的表达式是很麻烦的。为了方便起见，定义一个存储函数：

```
mysql> `CREATE FUNCTION initial_cap (s VARCHAR(255))`
    -> `RETURNS VARCHAR(255) DETERMINISTIC`
    -> `RETURN CONCAT(UPPER(LEFT(s,1)),MID(s,2));`
```

然后您可以更容易地将初始字符大写：

```
mysql> `SELECT thing, initial_cap(thing) FROM limbs;`
+--------------+--------------------+
| thing        | initial_cap(thing) |
+--------------+--------------------+
| human        | Human              |
| insect       | Insect             |
| squid        | Squid              |
| fish         | Fish               |
| centipede    | Centipede          |
| table        | Table              |
| armchair     | Armchair           |
| phonograph   | Phonograph         |
| tripod       | Tripod             |
| Peg Leg Pete | Peg Leg Pete       |
| space alien  | Space alien        |
+--------------+--------------------+
```

有关编写存储函数的更多信息，请参见第十一章。

# 7.7 比较字符串值

## 问题

您想知道字符串是否相等或不等，或者哪个在词法顺序中出现在前面。

## 解决方案

使用比较运算符。但请记住，字符串具有大小写敏感等属性，您必须考虑这些属性。字符串比较可能是大小写敏感的，这不是您想要的，反之亦然。

与其他数据类型类似，您可以比较字符串的相等性、不等性或相对顺序：

```
mysql> `SELECT 'cat' = 'cat', 'cat' = 'dog', 'cat' <> 'cat', 'cat' <> 'dog';`
+---------------+---------------+----------------+----------------+
| 'cat' = 'cat' | 'cat' = 'dog' | 'cat' <> 'cat' | 'cat' <> 'dog' |
+---------------+---------------+----------------+----------------+
|             1 |             0 |              0 |              1 |
+---------------+---------------+----------------+----------------+
mysql> `SELECT 'cat' < 'auk', 'cat' < 'dog', 'cat' BETWEEN 'auk' AND 'eel';`
+---------------+---------------+-------------------------------+
| 'cat' < 'auk' | 'cat' < 'dog' | 'cat' BETWEEN 'auk' AND 'eel' |
+---------------+---------------+-------------------------------+
|             0 |             1 |                             1 |
+---------------+---------------+-------------------------------+
```

## 讨论

然而，字符串的比较和排序属性存在一些其他类型数据不适用的复杂情况。例如，有时您必须确保一个本来不区分大小写的字符串比较是区分大小写的，反之亦然。本节描述了如何做到这一点。

字符串比较的属性取决于操作数是二进制还是非二进制字符串：

+   二进制字符串是字节序列，并使用数字字节值进行比较。大小写没有意义。然而，因为不同大小写的字母具有不同的字节值，因此二进制字符串的比较实际上是大小写敏感的。（即，`a`和`A`是不相等的。）为了比较二进制字符串，使字母大小写无关紧要，请将它们转换为具有不区分大小写排序规则的非二进制字符串。

+   非二进制字符串是字符序列，以字符单位进行比较。（根据字符集，某些字符可能有多个字节。）字符串具有定义法律字符和定义排序顺序的字符集，并由排序规则决定是否在比较中将不同大小写的字符视为相同。如果排序规则是大小写敏感的，并且您希望使用不区分大小写的排序规则（反之亦然），请将字符串转换为使用具有所需大小写比较属性的排序规则。

默认情况下，字符串的字符集为`utf8mb4`，排序规则为`utf8mb4_0900_ai_ci`，除非重新配置服务器（参见第 22.1 节）。这导致字符串比较时不区分大小写。

以下示例显示如何处理两个二进制字符串，使它们在比较时相等，即使它们作为不区分大小写的非二进制字符串进行比较：

```
mysql> `SET @s1 = CAST('cat' AS BINARY), @s2 = CAST('CAT' AS BINARY);`
mysql> `SELECT @s1 = @s2;`
+-----------+
| @s1 = @s2 |
+-----------+
|         0 |
+-----------+
mysql> `SET @s1 = CONVERT(@s1 USING utf8mb4) COLLATE utf8mb4_0900_ai_ci;`
mysql> `SET @s2 = CONVERT(@s2 USING utf8mb4) COLLATE utf8mb4_0900_ai_ci;`
mysql> `SELECT @s1 = @s2;`
+-----------+
| @s1 = @s2 |
+-----------+
|         1 |
+-----------+
```

在这种情况下，因为`utf8mb4`的默认排序规则是`utf8mb4_0900_ai_ci`，你可以省略`COLLATE`运算符：

```
mysql> `SET @s1 = CONVERT(@s1 USING utf8mb4);`
mysql> `SET @s2 = CONVERT(@s2 USING utf8mb4);`
mysql> `SELECT @s1 = @s2;`
+-----------+
| @s1 = @s2 |
+-----------+
|         1 |
+-----------+
```

下一个示例显示了如何以区分大小写的方式比较两个不区分大小写的字符串：

```
mysql> `SET @s1 = _latin1 'cat', @s2 = _latin1 'CAT';`
mysql> `SELECT @s1 = @s2;`
+-----------+
| @s1 = @s2 |
+-----------+
|         1 |
+-----------+
mysql> `SELECT @s1 COLLATE latin1_general_cs = @s2 COLLATE latin1_general_cs`
    ->   `AS '@s1 = @s2';`
+-----------+
| @s1 = @s2 |
+-----------+
|         0 |
+-----------+
```

如果比较二进制字符串和非二进制字符串，则比较将两个操作数视为二进制字符串：

```
mysql> `SELECT _latin1 'cat' = CAST('CAT' AS BINARY);`
+---------------------------------------+
| _latin1 'cat' = CAST('CAT' AS BINARY) |
+---------------------------------------+
|                                     0 |
+---------------------------------------+
```

因此，为了将两个非二进制字符串作为二进制字符串进行比较，请在比较它们时将它们转换为`BINARY`数据类型中的其中一个：

```
mysql> `SET @s1 = _latin1 'cat', @s2 = _latin1 'CAT';`
mysql> `SELECT @s1 = @s2, CAST(@s1 AS BINARY) = @s2, @s1 = CAST(@s2 AS BINARY);`
+-----------+---------------------------+---------------------------+
| @s1 = @s2 | CAST(@s1 AS BINARY) = @s2 | @s1 = CAST(@s2 AS BINARY) |
+-----------+---------------------------+---------------------------+
|         1 |                         0 |                         0 |
+-----------+---------------------------+---------------------------+
```

如果发现使用不适合通常使用的比较类型声明了列，请使用`ALTER TABLE`更改类型。假设此表存储新闻文章：

```
CREATE TABLE news
(
  id      INT UNSIGNED NOT NULL AUTO_INCREMENT,
  article BLOB,
  PRIMARY KEY (id)
);
```

此处`article`列声明为`BLOB`。这是一种二进制字符串类型，因此列中存储的文本比较不考虑字符集。（实际上，它们是区分大小写的。）如果这不是您想要的，请使用`ALTER TABLE`将列转换为具有不区分大小写排序规则的非二进制类型：

```
ALTER TABLE news
  MODIFY article TEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

# 7.8 十进制、八进制和十六进制格式之间的转换

## 问题

您想要在不同的数值基之间进行转换。

## 解决方案

使用此部分描述的`CONV()`函数和 SQL 模式。

## 讨论

在某些格式（如`HEX`）中，操作文本字符串作为文本字符串很困难。一种替代方法是将它们转换为二进制值。这将产生一个值为 BINARY(16)且长度为 128 位的数据类型。使用`BIN()`、`OCT()`、`HEX()`函数在十进制数和二进制、八进制和十六进制之间进行转换已经是可能的。如果我们要做反向转换呢？这就是`CONV()`函数发挥作用的地方。使用`CONV()`函数，我们可以从一个数值基系统转换为另一个数值基系统。

使用`CONV()`函数的语法：

```
 CONV(number, from_base, to_base)
```

+   `number`是我们想要从一种数值基转换到另一种数值基的值。

+   `from_base`是数字基数的原始基值，限制在 2 到 36 之间的值。

+   `to_base`是数值基的目标值。此值可以介于 2 到 36 或-2 到-36 之间。

```
mysql> `SELECT CONV(8, 10, 2) AS DectoBin;`
+----------+
| DectoBin |
+----------+
| 1000     |
+----------+
```

类似于`BIN()`函数，我们获得相同的结果。尽管`BIN()`函数返回一个字符串。

```
mysql> `SELECT BIN(8);`
+--------+
| BIN(8) |
+--------+
| 1000   |
+--------+
```

同样，我们可以反向将值之间进行转换：

```
mysql> `SELECT CONV('F', 16, 10) AS Hex2Dec;`
+---------+
| Hex2Dec |
+---------+
| 15      |
+---------+
```

# 7.9 ASCII、BIT 和十六进制格式之间的转换

## 问题

您想要在一种字符串格式和另一种字符串格式之间进行转换。

## 解决方案

使用 MySQL 的`CHAR()`、`ASCII()`、`BIT_LENGTH()`函数和此部分描述的 SQL 模式。

## 讨论

MySQL 提供了许多强大的字符串函数来支持字符串操作。在不同的使用情况下，我们可能需要将它们转换为不同的结果。使用一些字符串函数如`ASCII()`，我们可以在`BIT`和`HEX`之间进行转换。

使用`ASCII()`函数的语法：

```
 ASCII(character);
```

###### 注意

`ASCII()`函数只返回字符串的最左字符的数字值。这类似于 MySQL 的`ORD()`函数。

```
mysql> `SELECT ASCII("LARA");`
+---------------+
| ASCII("LARA") |
+---------------+
|            76 |
+---------------+
```

```
mysql> `SELECT ASCII("L");`
+---------------+
| ASCII("L") |
+---------------+
|            76 |
+---------------+
```

如您所见，两个字符串的结果是相同的。该函数只取字符串的左侧字符。

在以下示例中，我们将一个字符串值转换为`HEX`格式：

```
mysql> `SELECT DISTINCT CONV(ASCII("LARA"),10,16) as ASCII2HEX;`
+-----------+
| ASCII2HEX |
+-----------+
| 4C        |
+-----------+
```

假设我们有一个名为`name`的表，并且想要以`HEX`格式获取此表中所有唯一的`last_name`：

```
mysql> `SELECT DISTINCT CONV(ASCII(last_name),10,16) from name;`
+------------------------------+
| CONV(ASCII(last_name),10,16) |
+------------------------------+
| 42                           |
| 47                           |
| 57                           |
+------------------------------+
```

MySQL 8.0 之前的位操作处理无符号 64 位整数值。MySQL 8.0 之后，位操作扩展到处理二进制字符串参数。这允许非整数或二进制字符串被转换。根据[RFC 4122](https://datatracker.ietf.org/doc/html/rfc4122)，UUID（通用唯一标识符）是一个全局的 128 位唯一值，当需要完全的唯一性时非常有用。UUID 还可以用于安全目的，因为它不会泄露任何关于数据的信息。它以人类可读的`utf8mb4`格式表示，由五个十六进制数字符串组成。一个很好的例子是使用`UUID_TO_BIN`函数将`UUID`值转换为二进制（我们使用 *mysql* 并选择了`binary-as-hex`选项）：

```
mysql> `SELECT UUID();`
+--------------------------------------+
| UUID()                               |
+--------------------------------------+
| e52e0524-385b-11ec-99b1-054a662275e4 |
+--------------------------------------+
```

```
mysql> `SELECT UUID_TO_BIN(UUID());`
+------------------------------------------+
| UUID_TO_BIN(UUID())                      |
+------------------------------------------+
| 0xB8D11A66134E11ECB46CC15B8175C680       |
+------------------------------------------+
```

稍后我们可以将此值转换为使用`BIT_COUNT()`函数比较的值。此函数主要用于识别给定输入中的活动位。

```
mysql> `select UUID_TO_BIN(UUID()) into @bin_uuid;`
mysql> `select BIT_COUNT(@bin_uuid);`
+----------------------+
| BIT_COUNT(@bin_uuid) |
+----------------------+
|                   57 |
+----------------------+
```

`BIT_COUNT()`函数的目的是识别给定十进制值中的活动位。例如，如果我们想要找出数字 18 中的活动位。18 的二进制转换为`10010`，因此活动位仅有两个。

```
mysql> `select BIT_COUNT(18);`
+---------------+
| BIT_COUNT(18) |
+---------------+
|             2 |
+---------------+
```

`BIT_COUNT()`函数与`BIT_OR()`函数结合使用可以用来计算以下问题。`BIT_OR()`函数返回表达式中所有位的按位 OR 运算结果。假设我们想要找出十一月份的星期日数量。我们将创建一个名为`sundays`的表：

```
mysql> `CREATE TABLE sundays (     year YEAR,      month INT UNSIGNED ,     day INT UNSIGNED  );` 
```

```
mysql> `INSERT INTO sundays VALUES(2021,11,7),                             (2021,11,14),                             (2021,11,21),                             (2021,11,28);` 
```

```
mysql> `SELECT year, month, BIT_COUNT(BIT_OR(1 << day)) AS 'Number of Sundays'`
    -> `FROM sundays GROUP BY year,month;` 
+------+-------+-------------------+
| year | month | Number of Sundays |
+------+-------+-------------------+
| 2021 |    11 |                 4 |
+------+-------+-------------------+
```

这个例子可以扩展到查找给定日历年或日期范围内的假期数量。

另一个用例是 IPv6 和 IPv4 网络地址是字符串值。为了返回表示它的二进制值以便在数值上使用，可以使用`INET_ATON()`函数。该函数将点分十进制 IPv4 地址字符串表示转换为数值。虽然此函数的用途可能各不相同，但它通常用于存储 IP 地址的来源和目标，主要是用于数据日志记录。一旦 IPv4 地址存储为数值，就可以更快地进行索引和处理。

```
mysql> `SELECT INET_ATON('10.0.2.1');`
+-----------------------+
| INET_ATON('10.0.2.1') |
+-----------------------+
|             167772673 |
+-----------------------+
```

```
mysql> `SELECT HEX(INET6_ATON('10.0.2.1'));`
+-----------------------------+
| HEX(INET6_ATON('10.0.2.1')) |
+-----------------------------+
| 0A000201                    |
+-----------------------------+
```

# 7.10 使用 SQL 模式进行模式匹配

## 问题

您希望执行模式匹配，而不是字面比较。

## 解决方案

使用`LIKE`运算符和 SQL 模式在本节中描述。或者使用正则表达式模式匹配，在 Recipe 7.11 中描述。

## 讨论

模式是包含特殊字符的字符串，称为元字符，因为它们代表的是其他东西而不是它们自己。MySQL 提供两种模式匹配方法。一种基于 SQL 模式，另一种基于正则表达式。SQL 模式在不同数据库系统中更为标准，但正则表达式更为强大。这节描述了 SQL 模式。Recipe 7.11 描述了正则表达式。

此处的示例使用名为`metal`的表，其中包含以下行：

```
+----------+
| name     |
+----------+
| gold     |
| iron     |
| lead     |
| mercury  |
| platinum |
| tin      |
+----------+
```

SQL 模式匹配使用`LIKE`和`NOT LIKE`操作符，而不是`=和<>`来对模式字符串执行匹配。模式可以包含两个特殊元字符：`_`匹配任何单个字符，`%`匹配任何字符序列，包括空字符串。您可以使用这些字符来创建匹配各种值的模式：

+   以特定子字符串开头的字符串：

    ```
    mysql> `SELECT name FROM metal WHERE name LIKE 'me%';`
    +---------+
    | name    |
    +---------+
    | mercury |
    +---------+
    ```

+   以特定子字符串结尾的字符串：

    ```
    mysql> `SELECT name FROM metal WHERE name LIKE '%d';`
    +------+
    | name |
    +------+
    | gold |
    | lead |
    +------+
    ```

+   包含特定子字符串的字符串：

    ```
    mysql> `SELECT name FROM metal WHERE name LIKE '%in%';`
    +----------+
    | name     |
    +----------+
    | platinum |
    | tin      |
    +----------+
    ```

+   包含特定位置子字符串的字符串（只有在`name`列的第三个位置出现`at`时，模式才匹配成功）：

    ```
    mysql> `SELECT name FROM metal where name LIKE '__at%';`
    +----------+
    | name     |
    +----------+
    | platinum |
    +----------+
    ```

只有当 SQL 模式完全匹配比较值时才能成功匹配。在以下两个模式匹配中，只有第二个成功：

```
'abc' LIKE 'b'
'abc' LIKE '%b%'
```

要反转模式匹配的意义，请使用`NOT LIKE`。以下语句找到不包含`i`字符的字符串：

```
mysql> `SELECT name FROM metal WHERE name NOT LIKE '%i%';`
+---------+
| name    |
+---------+
| gold    |
| lead    |
| mercury |
+---------+
```

SQL 模式不匹配`NULL`值。这对于`LIKE`和`NOT LIKE`操作都是真的：

```
mysql> `SELECT NULL LIKE '%', NULL NOT LIKE '%';`
+---------------+-------------------+
| NULL LIKE '%' | NULL NOT LIKE '%' |
+---------------+-------------------+
|          NULL |              NULL |
+---------------+-------------------+
```

在某些情况下，模式匹配相当于子字符串比较。例如，使用模式在字符串的一端或另一端找到字符串，就像使用`LEFT()`或`RIGHT()`函数一样，如 Table 7-2 所示：

表 7-2\. 模式匹配与子字符串比较

| 模式匹配 | 子字符串比较 |
| --- | --- |
| *`str`* `LIKE 'abc%'` | `LEFT(`*`str`*`,3) = 'abc'` |
| *`str`* `LIKE '%abc'` | `RIGHT(`*`str`*`,3) = 'abc'` |

如果您要对一个索引列进行匹配，并且可以选择使用模式或等效的`LEFT()`表达式，您可能会发现模式匹配速度更快。MySQL 可以使用索引来缩小搜索范围，以寻找以字面字符串开头的模式。而使用`LEFT()`则不能。此外，由于优化器需要检查字符串的整个内容，因此以`%`开头的`LIKE`比较可能会很慢。

模式匹配的大小写敏感性与字符串比较类似。也就是说，它取决于操作数是二进制还是非二进制字符串，对于非二进制字符串，还取决于它们的排序规则。详细讨论请参见 Recipe 7.7。

# 7.11 使用正则表达式进行模式匹配

## 问题

您希望执行模式匹配，而不是字面上的比较。

## 解决方案

使用`REGEXP`运算符和本节描述的正则表达式模式。或者使用在 Recipe 7.10 中描述的 SQL 模式。

## 讨论

SQL 模式（参见 Recipe 7.10）可能由其他数据库系统实现，因此除了 MySQL 外，它们也具有一定的可移植性。然而，它们的功能有所限制。例如，您可以轻松地编写一个 SQL 模式 `%abc%` 来查找包含 `abc` 的字符串，但是您无法编写一个单独的 SQL 模式来识别包含字符 `a`、`b` 或 `c` 中任何一个的字符串。也不能基于字符类型（如字母或数字）匹配字符串内容。对于这些操作，MySQL 支持基于正则表达式和 `REGEXP` 运算符（或 `NOT` `REGEXP` 反转匹配的意义）的另一种模式匹配操作。`REGEXP` 匹配使用表 7-4 中显示的模式元素：

表 7-4\. 常用正则表达式语法

| 模式 | 模式匹配的内容 |
| --- | --- |
| `^` | 字符串开始位置 |
| `$` | 字符串结束位置 |
| `.` | 任意单个字符 |
| `[...]` | 方括号内列出的任何字符 |
| `[^...]` | 方括号内未列出的任何字符 |
| *`p1`*`&#124;`*`p2`*`&#124;`*`p3`* | 替换；匹配模式 *`p1`*, *`p2`*, 或 *`p3`* 中的任何一个 |
| `*` | 前一元素的零个或多个实例 |
| `+` | 前一元素的一个或多个实例 |
| `{`*`n`*`}` | 前一元素的 *`n`* 次实例 |
| `{`*`m`*`,`*`n`*`}` | 前一元素的 *`m`* 到 *`n`* 次实例 |

您可能已经熟悉这些正则表达式模式字符；其中许多与 *vi*、*grep*、*sed* 和其他支持正则表达式的 Unix 实用工具使用的字符相同。大多数这些字符也用于编程语言理解的正则表达式中。（有关数据验证和转换程序中的模式匹配讨论，请参阅 Chapter 14。）

Recipe 7.10 显示如何使用 SQL 模式来匹配字符串的开头或结尾处的子字符串，或者在字符串的任意或特定位置。您可以使用正则表达式执行相同的操作：

+   以特定子字符串开头的字符串：

    ```
    mysql> `SELECT name FROM metal WHERE name REGEXP '^me';`
    +---------+
    | name    |
    +---------+
    | mercury |
    +---------+
    ```

+   以特定子字符串结尾的字符串：

    ```
    mysql> `SELECT name FROM metal WHERE name REGEXP 'd$';`
    +------+
    | name |
    +------+
    | gold |
    | lead |
    +------+
    ```

+   在任何位置包含特定子字符串的字符串：

    ```
    mysql> `SELECT name FROM metal WHERE name REGEXP 'in';`
    +----------+
    | name     |
    +----------+
    | platinum |
    | tin      |
    +----------+
    ```

+   在特定位置包含特定子字符串的字符串：

    ```
    mysql> `SELECT name FROM metal WHERE name REGEXP '^..at';`
    +----------+
    | name     |
    +----------+
    | platinum |
    +----------+
    ```

此外，正则表达式还具有其他功能，并且可以执行 SQL 模式无法完成的匹配。例如，正则表达式可以包含字符类，该类可以匹配类中的任何字符：

+   要编写字符类，请使用方括号并在括号内列出要匹配的字符。因此，模式 `[abc]` 可以匹配 `a`、`b` 或 `c`。

+   类可以指示字符范围；使用破折号表示范围的起始和结束。`[a-z]` 匹配任何字母，`[0-9]` 匹配数字，`[a-z0-9]` 匹配字母或数字。

+   要否定一个字符类（<q>匹配除了这些字符之外的任何字符</q>），请以 `^` 字符开始列表。例如，`[⁰-9]` 匹配除了数字以外的任何内容。

MySQL 的正则表达式功能也支持 POSIX 字符类。这些类匹配特定的字符集，如 表 7-5 中所述：

表 7-5\. POSIX 正则表达式语法

| POSIX 类 | 类匹配的内容 |
| --- | --- |
| `[:alnum:]` | 字母和数字字符 |
| `[:alpha:]` | 字母字符 |
| `[:blank:]` | 空白字符（空格或制表符） |
| `[:cntrl:]` | 控制字符 |
| `[:digit:]` | 数字 |
| `[:graph:]` | 图形（非空白）字符 |
| `[:lower:]` | 小写字母字符 |
| `[:print:]` | 图形或空格字符 |
| `[:punct:]` | 标点字符 |
| `[:space:]` | 空格、制表符、换行符、回车符 |
| `[:upper:]` | 大写字母字符 |
| `[:xdigit:]` | 十六进制数字 (`0-9`, `a-f`, `A-F`) |

POSIX 类用于字符类中，因此请在方括号内使用它们。以下表达式匹配包含任何十六进制数字字符的值：

```
mysql> `SELECT name, name REGEXP '[[:xdigit:]]' FROM metal;`
+----------+----------------------------+
| name     | name REGEXP '[[:xdigit:]]' |
+----------+----------------------------+
| gold     |                          1 |
| iron     |                          0 |
| lead     |                          1 |
| mercury  |                          1 |
| platinum |                          1 |
| tin      |                          0 |
+----------+----------------------------+
```

正则表达式可以使用以下语法指定选择项：

```
*`alternative1`*|*`alternative2`*|...
```

选择项类似于字符类，因为如果任何选择项匹配，则选择项将匹配。但与字符类不同，选择项不限于单个字符。它们可以是多字符字符串甚至是模式。以下选择项匹配以元音字母开头或以 `d` 结尾的字符串：

```
mysql> `SELECT name FROM metal WHERE name REGEXP '^[aeiou]|d$';`
+------+
| name |
+------+
| gold |
| iron |
| lead |
+------+
```

括号可用于分组选择项。例如，要匹配完全由数字或完全由字母组成的字符串，您可以尝试以下模式，使用一个选择项：

```
mysql> `SELECT '0m' REGEXP '^[[:digit:]]+|[[:alpha:]]+$';`
+-------------------------------------------+
| '0m' REGEXP '^[[:digit:]]+|[[:alpha:]]+$' |
+-------------------------------------------+
|                                         1 |
+-------------------------------------------+
```

然而，查询结果显示，该模式不起作用。这是因为 `^` 与第一个选择项分组，而 `$` 与第二个选择项分组。因此，该模式实际上匹配以一个或多个数字开头的字符串，或以一个或多个字母结尾的字符串。如果在括号中分组选择项，则 `^` 和 `$` 将适用于它们两个，并且模式将按预期行事：

```
mysql> `SELECT '0m' REGEXP '^([[:digit:]]+|[[:alpha:]]+)$';`
+---------------------------------------------+
| '0m' REGEXP '^([[:digit:]]+|[[:alpha:]]+)$' |
+---------------------------------------------+
|                                           0 |
+---------------------------------------------+
```

与 SQL 模式匹配不同，正则表达式在值的任意位置成功匹配模式即可，而不必匹配整个比较值。以下两个模式匹配在某种意义上等效，即每个模式仅成功匹配包含字符 `b` 的字符串，但第一个模式更有效率，因为模式更简单：

```
'abc' REGEXP 'b'
'abc' REGEXP '^.*b.*$'
```

正则表达式不匹配 `NULL` 值。对于 `REGEXP` 和 `NOT` `REGEXP` 都是如此：

```
mysql> `SELECT NULL REGEXP '.*', NULL NOT REGEXP '.*';`
+------------------+----------------------+
| NULL REGEXP '.*' | NULL NOT REGEXP '.*' |
+------------------+----------------------+
|             NULL |                 NULL |
+------------------+----------------------+
```

因为正则表达式如果在字符串中找到模式则匹配该字符串，所以必须小心不要无意中指定一个匹配空字符串的模式。如果这样做，它会匹配任何非`NULL`值。例如，模式`a*`匹配任意数量的`a`字符，甚至是零个。如果您的目标是仅匹配包含非空`a`字符序列的字符串，请改用`a+`。`+`要求前面的模式元素至少出现一次才能匹配。

类似于使用`LIKE`执行的 SQL 模式匹配，使用`REGEXP`执行的正则表达式匹配有时等同于子字符串比较。如在表 7-6 中所示，`^` 和 `$` 元字符在寻找字面字符串时具有类似于 `LEFT()` 或 `RIGHT()` 的作用：

表 7-6\. 正则表达式与子字符串比较函数

| 模式匹配 | 子字符串比较 |
| --- | --- |
| *`str`* `REGEXP '^abc'` | `LEFT(`*`str`*`,3) = 'abc'` |
| *`str`* `REGEXP 'abc$'` | `RIGHT(`*`str`*`,3) = 'abc'` |

对于非字面模式，通常不可能构造一个等效的子字符串比较。例如，要匹配以任意非空数字序列开头的字符串，请使用以下模式匹配：

```
*`str`* REGEXP '^[0-9]+'

```

这是`LEFT()`不能做到的（事实上，`LIKE`也不能做到）。

正则表达式匹配的大小写敏感性与字符串比较类似。也就是说，它取决于操作数是二进制还是非二进制字符串，对于非二进制字符串，还取决于它们的排序规则。参见 7.7 节讨论这些因素如何适用于比较。

###### 注意

在 8.0.4 版本之前，正则表达式仅适用于单字节字符集。在 MySQL 8.0.4 中，这个限制被移除，现在您可以在诸如`utf8mb4`或`sjis`之类的多字节字符集中使用正则表达式。

# 7.12 反转字符串内容

## 问题

您想修改一个字符串，并找出它的反向形式。

## 解决方案

使用`REVERSE()`函数。

## 讨论

您可以通过使用`REVERSE()`函数来反转字符串或子字符串。该函数通过字符将任何字符串值转换为其反向形式。它经常在像本章其他许多函数一样的`SELECT`语句中使用。

使用`REVERSE()`函数的语法：

```
 REVERSE(expression)
```

以下示例展示了`REVERSE()`函数的基本功能：

```
mysql> `SELECT REVERSE("sports flow");`
+------------------------+
| REVERSE("sports flow") |
+------------------------+
| wolf strops            |
+------------------------+
```

```
mysql> `SELECT REVERSE(0123456789);`
+---------------------+
| REVERSE(0123456789) |
+---------------------+
| 987654321           |
+---------------------+
```

```
mysql> `SELECT REVERSE("0123456789");`
+-----------------------+
| REVERSE("0123456789") |
+-----------------------+
| 9876543210            |
+-----------------------+
```

下面的示例显示了当表达式为`numeric`值时，函数会省略零值。

```
mysql> `SELECT REVERSE(001122334455);`
+-----------------------+
| REVERSE(001122334455) |
+-----------------------+
| 5544332211            |
+-----------------------+
```

尽管我们可以反转任何表达式，但我们也有一些单词，它们反过来写时与原始单词相同，被称为回文。对于这样的字符串，函数`REVERSE`将返回与原始字符串相等的字符串。例如：

```
mysql> `SELECT REVERSE("STEP ON NO PETS");`
+----------------------------+
| REVERSE("STEP ON NO PETS") |
+----------------------------+
| STEP ON NO PETS            |
+----------------------------+
```

更广泛的示例使用*recipes*发行版中的`top_names`表，该表存储了最常用的姓名。在这些姓名中，我们将找出回文姓名的数量。

```
mysql> `SELECT COUNT(*) FROM top_names WHERE REVERSE(top_name) = top_name;`
+----------+
| COUNT(*) |
+----------+
|      234 |
+----------+
```

仅仅为了从这个计数中获取一个样本，我们可以查看以“U”开头的名称。

```
mysql> `SELECT top_name FROM top_names`
    -> `WHERE REVERSE(top_name) = top_name`
    -> `AND top_name LIKE "U%";`
+----------+
| top_name |
+----------+
| ULU      |
| UTU      |
+----------+
```

# 7.13 搜索子字符串

## 问题

你想知道给定的字符串是否出现在另一个字符串中。

## 解决方案

使用 `LOCATE()` 或模式匹配。

## 讨论

`LOCATE()` 函数接受两个参数，分别表示要查找的子字符串和要在其中查找的字符串。返回值是子字符串出现的位置，如果不存在则返回 `0`。可选的第三个参数可以指定开始查找的位置。

```
mysql> `SELECT name, LOCATE('in',name), LOCATE('in',name,3) FROM metal;`
+----------+-------------------+---------------------+
| name     | LOCATE('in',name) | LOCATE('in',name,3) |
+----------+-------------------+---------------------+
| gold     |                 0 |                   0 |
| iron     |                 0 |                   0 |
| lead     |                 0 |                   0 |
| mercury  |                 0 |                   0 |
| platinum |                 5 |                   5 |
| tin      |                 2 |                   0 |
+----------+-------------------+---------------------+
```

如果只需确定子字符串是否存在且不关心其位置，则可以使用 `LIKE` 或 `REGEXP` 的替代方法：

```
mysql> `SELECT name, name LIKE '%in%', name REGEXP 'in' FROM metal;`
+----------+------------------+------------------+
| name     | name LIKE '%in%' | name REGEXP 'in' |
+----------+------------------+------------------+
| gold     |                0 |                0 |
| iron     |                0 |                0 |
| lead     |                0 |                0 |
| mercury  |                0 |                0 |
| platinum |                1 |                1 |
| tin      |                1 |                1 |
+----------+------------------+------------------+
```

`LOCATE()`、`LIKE` 和 `REGEXP` 使用它们参数的排序规则来决定搜索是否区分大小写。7.5 节 和 7.7 节 讨论了如果需要改变搜索行为，则可以更改参数比较属性。

# 7.14 分解或组合字符串

## 问题

你想要从字符串中提取一部分或者组合字符串以形成更大的字符串。

## 解决方案

要获取字符串的一部分，请使用子字符串提取函数。要组合字符串，请使用 `CONCAT()`。

## 讨论

你可以使用适当的子字符串提取函数来分解字符串。例如，`LEFT()`、`MID()` 和 `RIGHT()` 函数可以从字符串的左侧、中间或右侧提取子字符串：

```
mysql> `SET @date = '2015-07-21';`
mysql> `SELECT @date, LEFT(@date,4) AS year,`
    -> `MID(@date,6,2) AS month, RIGHT(@date,2) AS day;`
+------------+------+-------+------+
| @date      | year | month | day  |
+------------+------+-------+------+
| 2015-07-21 | 2015 | 07    | 21   |
+------------+------+-------+------+
```

对于 `LEFT()` 和 `RIGHT()`，第二个参数指示从字符串左侧或右侧返回多少个字符。对于 `MID()`，第二个参数是你想要的子字符串的起始位置（从 1 开始），第三个参数表示要返回的字符数。

`SUBSTRING()` 函数接受一个字符串和一个起始位置，并返回该位置右侧的所有内容。如果省略 `MID()` 的第三个参数，它的行为与 `SUBSTRING()` 相同，因为 `MID()` 实际上是 `SUBSTRING()` 的同义词：

```
mysql> `SET @date = '2015-07-21';`
mysql> `SELECT @date, SUBSTRING(@date,6), MID(@date,6);`
+------------+--------------------+--------------+
| @date      | SUBSTRING(@date,6) | MID(@date,6) |
+------------+--------------------+--------------+
| 2015-07-21 | 07-21              | 07-21        |
+------------+--------------------+--------------+
```

使用 `SUBSTRING_INDEX(`*`str`*`,`*`c`*`,`*`n`*`)` 来返回给定字符的右侧或左侧的所有内容。它会在字符串 *`str`* 中搜索第 *`n`* 次出现的字符 *`c`*，并返回其左侧的所有内容。如果 *`n`* 是负数，则从右侧开始搜索字符 *`c`* 并返回其右侧的所有内容：

```
mysql> `SET @email = 'postmaster@example.com';`
mysql> `SELECT @email,`
    -> `SUBSTRING_INDEX(@email,'@',1) AS user,`
    -> `SUBSTRING_INDEX(@email,'@',-1) AS host;`
+------------------------+------------+-------------+
| @email                 | user       | host        |
+------------------------+------------+-------------+
| postmaster@example.com | postmaster | example.com |
+------------------------+------------+-------------+
```

如果没有字符的第 *`n`* 次出现，`SUBSTRING_INDEX()` 返回整个字符串。`SUBSTRING_INDEX()` 区分大小写。

你可以使用子字符串来执行除了显示之外的其他目的，比如执行比较。以下语句查找第一个字母位于字母表后半部分的金属名称：

```
mysql> `SELECT name from metal WHERE LEFT(name,1) >= 'n';`
+----------+
| name     |
+----------+
| platinum |
| tin      |
+----------+
```

要组合而不是分解字符串，请使用 `CONCAT()` 函数。它连接其参数并返回结果：

```
mysql> `SELECT CONCAT(name,' ends in "d": ',IF(name LIKE '%d','YES','NO'))`
    -> `AS 'ends in "d"?'`
    -> `FROM metal;`
+--------------------------+
| ends in "d"?             |
+--------------------------+
| gold ends in "d": YES    |
| iron ends in "d": NO     |
| lead ends in "d": YES    |
| mercury ends in "d": NO  |
| platinum ends in "d": NO |
| tin ends in "d": NO      |
+--------------------------+
```

连接对于在原地修改列值 <q>非常有用。</q> 例如，以下`UPDATE`语句将字符串添加到`metal`表中每个`name`值的末尾：

```
mysql> `UPDATE metal SET name = CONCAT(name,'ide');`
mysql> `SELECT name FROM metal;`
+-------------+
| name        |
+-------------+
| goldide     |
| ironide     |
| leadide     |
| mercuryide  |
| platinumide |
| tinide      |
+-------------+
```

要撤销操作，请去掉最后三个字符（`CHAR_LENGTH()`函数返回字符串的字符长度）：

```
mysql> `UPDATE metal SET name = LEFT(name,CHAR_LENGTH(name)-3);`
mysql> `SELECT name FROM metal;`
+----------+
| name     |
+----------+
| gold     |
| iron     |
| lead     |
| mercury  |
| platinum |
| tin      |
+----------+
```

对于`ENUM`或`SET`值，可以直接在原地修改列，尽管它们在内部存储为数字，但通常可以视为字符串值。例如，要将`SET`元素连接到现有的`SET`列中，使用`CONCAT()`将新值添加到现有值之前，并注意考虑现有值可能为`NULL`的情况。在这种情况下，将列值设置为新元素，而不带前导逗号：

```
UPDATE *`tbl_name`*
SET *`set_col`* = IF(*`set_col`* IS NULL,*`val`*,CONCAT(*`set_col`*,',',*`val`*));
```

# 7.15 使用全文搜索

## 问题

你想要搜索长文本列。

## 解决方案

使用`FULLTEXT`索引。

## 讨论

模式匹配使您能够查看任意数量的行，但随着文本量的增加，匹配操作可能会变得非常慢。在几个字符串列中搜索相同文本是常见任务，但使用模式匹配时，这会导致查询变得笨重：

```
SELECT * from *`tbl_name`*
WHERE *`col1`* LIKE '*`pat%`*' OR *`col2`* LIKE '*`pat%`*' OR *`col3`* LIKE '*`pat%`*' ...
```

一个有用的替代方案是全文搜索，专为搜索大量文本而设计，并可以同时搜索多个列。要使用此功能，需要在表上添加`FULLTEXT`索引，然后使用`MATCH`操作符在索引列或列中查找字符串。`FULLTEXT`索引可以用于 MyISAM 表或 InnoDB 表的非二进制字符串数据类型（`CHAR`、`VARCHAR`或`TEXT`）。

全文搜索最好用一个相当大的文本体来说明。如果没有样本数据集，可以在互联网上找到几个可自由获取的电子文本存储库。对于这里的示例，我们选择的是 Amazon 评论数据（2018 年）的样本转储，公众可以从中下载和抓取[Amazon Review Data (2018).](https://nijianmo.github.io/amazon/index.html) 由于其规模，这个数据集没有包含在`recipes`发行版中，但可以在 GitHub 仓库的说明中单独获取。`Amazon`分发包括一个名为*Appliances_5.json*的文件，其中包含每个类别的产品评论。这是更大数据集的子集。由于大多数基于文本的数据都可以在互联网上找到，因此这些数据仅以`JSON`数据格式提供。一些样本记录看起来像这样：

```
{
      "overall":        2.0,
      "verified":       false,
      "reviewTime":     "07 6, 2017",
      "reviewerID":     "A3LGZ8M29PBNGG",
      "asin":           "B000N6302Q",
      "style":          {"Color:": " Stainless Steel"},
      "reviewerName":   "nerenttt",
      "reviewText":     "Luved it for the few months it worked!↩
                         great little diamond ice cubes...",
      "unixReviewTime": 1499299200
}
```

我们感兴趣的是`reviewText`字段，它包含了我们想要检查的大量文本。

每个记录包含以下字段：

+   `overall` - 产品的评分

+   `verified` - 购买验证标志

+   `reviewTime` - 评论日期

+   `reviewerID` - 评论者的 ID（`O`或`N`），例如 A2SUAM1J3GNN3B

+   `asin` - 产品的 ID，例如 0000013714

+   `style` - 产品元数据的自由选项

+   `reviewerName` 评论者姓名。

+   评论的 `reviewText` 文本。

+   `unixReviewTime` 评论时间（Unix 时间）。

要将记录导入到 MySQL 中，请创建名为 `reviews` 的表，如下所示：

```
CREATE TABLE `reviews` (  
   `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
   `appliences_review` JSON NOT NULL,
     PRIMARY KEY (`id`)
);
```

为了将 `json` 数据加载到此表中，我们可以使用 MySQL 内置的 json 函数，稍后将在 Recipe 13.17 中介绍。对于包含大量文本数据的某些情况，可能包含换行符 `/n/n` 等转义字符，这会导致导入函数无法正常工作。为了解决这个问题，我们将使用提供在 GitHub 仓库中的名为 `load_amazon_reviews.py` 的简单脚本来加载数据。

加载数据后，我们将将 `reviewText` 列转换为生成列，并添加 `FULLTEXT` 索引以启用其在全文搜索中的使用。

```
ALTER TABLE `reviews` ADD COLUMN `reviews_virtual` TEXT 
GENERATED ALWAYS AS (`appliences_review` ->> '$.reviewText') STORED NOT NULL;
ALTER TABLE `reviews` ADD FULLTEXT idx_ft_json(reviews_virtual);
```

该表现在具有 `FULLTEXT` 索引以启用其在全文搜索中的使用。

创建 `reviews` 表后，请使用以下语句将 *Appliances_5.json* 文件加载到其中：

```
 `python3 load_amazon_reviews.py Appliances_5.json`
```

您会注意到 `reviews` 表包含完整的 `appliences_review` 数据列以及用于演示 `FULLTEXT` 索引的 `reviews_virtual` 列。

```
CREATE TABLE `reviews` (
  `id` bigint unsigned NOT NULL AUTO_INCREMENT,
  `appliences_review` json NOT NULL,
  `reviews_virtual` text GENERATED ALWAYS AS 
    (json_unquote(json_extract(`appliences_review`,_utf8mb4'$.reviewText'))) 
  STORED NOT NULL,
  PRIMARY KEY (`id`),
  FULLTEXT KEY `idx_ft_json` (`reviews_virtual`)
) ENGINE=InnoDB AUTO_INCREMENT=2278 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

要使用 `FULLTEXT` 索引进行搜索，使用 `MATCH()` 命名索引列和 `AGAINST()` 指定要查找的文本。例如，您可能想知道：“‘awesome’ 这个词出现了多少次？” 要回答这个问题，请使用以下语句搜索 `reviews_virtual` 列：

```
mysql> `SELECT COUNT(*) from reviews WHERE MATCH(reviews_virtual) AGAINST('awesome');`
+----------+
| COUNT(*) |
+----------+
|        3 |
+----------+
```

验证 `FULLTEXT` 索引是否被使用。

```
mysql> `EXPLAIN select reviews_virtual from reviews WHERE MATCH(reviews_virtual)      -> AGAINST('awesome') \G`
*************************** 1\. row ***************************
           id: 1
  select_type: SIMPLE
        table: reviews
   partitions: NULL
         type: fulltext
possible_keys: idx_ft_json
          key: idx_ft_json
      key_len: 0
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using where; Ft_hints: sorted
```

要查找在 `reviews` 中含有关键字 “excellent” 的产品，选择您想要查看的列（这里的示例截断了 `reviews_virtual` 列并使用 `\G` 使结果更适合页面）：

```
mysql> `SELECT JSON_EXTRACT(appliences_review, "$.reviewerID") as ReviwerID,` 
    -> `JSON_EXTRACT(appliences_review, "$.asin") as ProductID,`
    -> `JSON_EXTRACT(appliences_review, "$.overall") as Rating`
    -> `from reviews WHERE MATCH(reviews_virtual) AGAINST('excellent') \G`
*************************** 1\. row ***************************
ReviwerID: "A2CIEGHZ7L1WWR"
ProductID: "B00009W3PA"
   Rating: 5.0
*************************** 2\. row ***************************
ReviwerID: "A1T1YSCDW0PD25"
ProductID: "B0013DN4NI"
   Rating: 5.0
*************************** 3\. row ***************************
ReviwerID: "A1T1YSCDW0PD25"
ProductID: "B0013DN4NI"
   Rating: 5.0
*************************** 4\. row ***************************
ReviwerID: "A26M3TN8QICJ3K"
ProductID: "B004XLDE5A"
   Rating: 5.0
*************************** 5\. row ***************************
ReviwerID: "A2CIEGHZ7L1WWR"
ProductID: "B004XLDHSE"
   Rating: 5.0
```

默认情况下，全文搜索计算相关性排序并将其用于排序。为了确保搜索结果按您希望的方式排序，请添加显式的 `ORDER BY` 子句：

```
SELECT reviews_virtual
FROM reviews WHERE MATCH(reviews_virtual) AGAINST('*`search string`*')
ORDER BY {column}, {column};
```

要查看相关性排序，请在输出列列表中重复使用 `MATCH()` … `AGAINST()` 表达式。

要进一步缩小搜索范围，请包含额外的条件。为了在搜索中提供额外字段，我们将从 `json` 提取中添加以下虚拟列。

```
ALTER TABLE `reviews` 
    -> ADD COLUMN `reviews_virtual_vote` VARCHAR(10) 
    -> GENERATED ALWAYS AS (`appliences_review` ->> '$.vote') STORED;
```

```
ALTER TABLE `reviews` 
    -> ADD COLUMN `reviews_virtual_overall` VARCHAR(10) 
    -> GENERATED ALWAYS AS (`appliences_review` ->> '$.overall') STORED;
```

```
ALTER TABLE `reviews` 
    -> ADD COLUMN `reviews_virtual_verified` VARCHAR(10) 
    -> GENERATED ALWAYS AS (`appliences_review` ->> '$.verified') STORED;
```

以下查询逐步执行更具体的搜索，以确定每个关键字的出现频率。

```
mysql> `SELECT count(*) from reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('good');`
+----------+
| COUNT(*) |
+----------+
|      855 |
+----------+
mysql> `SELECT count(*) from reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('good')`
    -> `AND reviews_virtual_vote > 5;`
+----------+
| COUNT(*) |
+----------+
|      620 |
+----------+
mysql> `SELECT COUNT(*) from reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('good')`
    -> `AND reviews_virtual_overall = 5;`
+----------+
| COUNT(*) |
+----------+
|      646 |
+----------+
mysql> `SELECT COUNT(*) from reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('good')`
    -> `AND reviews_virtual_overall = 5 AND reviews_virtual_verified = "True";`
+----------+
| COUNT(*) |
+----------+
|      645 |
+----------+
```

如果您经常使用包含其他非 `FULLTEXT` 列的搜索条件，请对这些列添加常规索引，以便查询性能更好。例如，要为投票、整体评分和已验证列创建索引，请执行以下操作：

```
mysql> `ALTER TABLE reviews ADD INDEX idx_vote (reviews_virtual_vote),     ->  ADD INDEX idx_overall(reviews_virtual_overall),      ->  ADD INDEX idx_verified(reviews_virtual_verified);`
```

全文查询中的搜索字符串可以包含多个词，您可能认为添加词会使搜索更具体。但实际上，这会扩大搜索范围，因为全文搜索返回包含任何词的行。事实上，查询对任何词执行 `OR` 搜索。以下查询说明了这一点；随着添加更多搜索词，它们标识出越来越多的评论：

```
mysql> `SELECT COUNT(*) from reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('excellent');`
+----------+
| COUNT(*) |
+----------+
|      11  |
+----------+
mysql> `SELECT COUNT(*) from reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('excellent product');`
+----------+
| COUNT(*) |
+----------+
|     1480 |
+----------+
mysql> `SELECT COUNT(*) from reviews`
    -> `WHERE MATCH(reviews_text) AGAINST('excellent product for home');`
+----------+
| COUNT(*) |
+----------+
|     1486 |
+----------+
```

要执行搜索，搜索字符串中的每个单词必须存在，请参见 Recipe 7.17。

要同时搜索多个列，请在构建`FULLTEXT`索引时命名所有列：

```
ALTER TABLE *`tbl_name`* ADD FULLTEXT (*`col1`*, *`col2`*, *`col3`*);
```

要发出使用索引的搜索查询，请在`MATCH()`列表中命名相同的列：

```
SELECT ... FROM *`tbl_name`*
WHERE MATCH(*`col1`*, *`col2`*, *`col3`*) AGAINST('*`search string`*');
```

您需要为要搜索的每个不同列组合创建一个`FULLTEXT`索引。

## 参见

获取有关`FULLTEXT`索引的进一步信息，请参阅 Recipe 21.9。

# 7.16 使用全文搜索查找短词

## 问题

您对短词的全文搜索未返回任何行。

## 解决方案

更改索引引擎的最小词长参数。

## 讨论

在像`reviews`这样的文本中，某些单词具有特殊意义，如<q>ok</q>和<q>up</q>。您可能需要首先检查全文索引服务器变量，以确保引擎满足最小长度要求。

```
mysql> `SHOW GLOBAL VARIABLES LIKE 'innodb_ft_%';`
+---------------------------------+------------+
| Variable_name                   | Value      |
+---------------------------------+------------+
| innodb_ft_aux_table             |            |
| innodb_ft_cache_size            | 8000000    |
| innodb_ft_enable_diag_print     | OFF        |
| innodb_ft_enable_stopword       | ON         |
| innodb_ft_max_token_size        | 84         |
| innodb_ft_min_token_size        | 3          |
| innodb_ft_num_word_optimize     | 2000       |
| innodb_ft_result_cache_limit    | 2000000000 |
| innodb_ft_server_stopword_table |            |
| innodb_ft_sort_pll_degree       | 2          |
| innodb_ft_total_cache_size      | 640000000  |
| innodb_ft_user_stopword_table   |            |
+---------------------------------+------------+
mysql> `SELECT count(*) FROM reviews WHERE MATCH(reviews_virtual) AGAINST('ok');`
+----------+
| count(*) |
+----------+
|       0  |
+----------+
`SELECT count(*) FROM reviews WHERE MATCH(reviews_virtual) AGAINST('up'); +----------+ | count(*) | +----------+ |       0  | +----------+`
```

索引引擎的一个特性是忽略<q>太常见</q>的单词（即在超过半数行中出现的单词）。这会消除诸如<q>the</q>或<q>and</q>之类的词，但这里并非如此。您可以通过计算总行数并使用 SQL 模式匹配来计算包含每个单词的行数来验证（参见 Recipe 10.1 关于使用`COUNT()`从相同值集合生成多个计数）：

```
mysql> `SELECT COUNT(*) AS Total_Reviews,`
    -> `COUNT(IF(reviews_virtual LIKE '%good%',1,NULL)) AS Good_Reviews,`
    -> `COUNT(IF(reviews_virtual LIKE '%great%',1,NULL)) AS Great_Reviews,`
    -> `COUNT(IF(reviews_virtual LIKE '%excellent%',1,NULL)) AS Excellent_Reviews`
    -> `FROM reviews;`
+---------------+--------------+---------------+-------------------+
| Total_Reviews | Good_Reviews | Great_Reviews | Excellent_Reviews |
+---------------+--------------+---------------+-------------------+
|          2277 |          855 |          1095 |                11 |
+---------------+--------------+---------------+-------------------+
```

InnoDB 全文索引引擎不包括少于三个字符长的单词。最小词长是一个可配置参数；要更改它，请设置 MyISAM 的`ft_min_word_len`或 InnoDB 存储引擎的`innodb_ft_min_token_size`系统变量。例如，要告诉索引引擎包括三个字符长的单词，可以在*/etc/my.cnf*文件的`[mysqld]`组中添加一行（或者您用于服务器设置的任何选项文件）：

```
[mysqld]
ft_min_word_len=2 ##MyISAM
innodb_ft_min_token_size=2 ##InnoDB
```

改变这个设置后，重新启动服务器。接下来，重建`FULLTEXT`索引以利用显示新设置，首先设置`innodb_optimize_fulltext_only`参数并运行`OPTIMIZE`操作。

```
mysql> `SET GLOBAL innodb_optimize_fulltext_only=ON;`
```

```
mysql> `OPTIMIZE TABLE reviews;`
```

对于 MyISAM，此外运行*REPAIR TABLE*命令：

```
mysql> `REPAIR TABLE reviews QUICK;`
```

您还应使用`REPAIR TABLE`来重建所有其他具有`FULLTEXT`索引的 MyISAM 表的索引。

最后，尝试新索引以验证它是否包括较短的单词：

```
mysql> `SELECT count(*) from reviews WHERE MATCH(reviews_virtual) AGAINST('ok');`
+----------+
| count(*) |
+----------+
|       10 |
+----------+
mysql> `SELECT count(*) from reviews WHERE MATCH(reviews_virtual) AGAINST('up');`
+----------+
| COUNT(*) |
+----------+
|     1449 |
+----------+
```

# 7.17 要求或禁止全文搜索词

## 问题

您想要在全文搜索中要求或禁止特定单词。

## 解决方案

使用布尔模式搜索。

## 讨论

通常，全文搜索返回包含搜索字符串中任何单词的行，即使其中一些单词缺失也是如此。例如，以下语句查找包含单词<q>good</q>或<q>great</q>的行：

```
mysql> `SELECT COUNT(*) FROM reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('good great');`
+----------+
| COUNT(*) |
+----------+
|     1330 |
+----------+
```

如果要求只包含同时包含这两个单词的行，则此行为是不合适的。做法之一是重写语句，查找每个单词并使用`AND`连接条件：

```
mysql> `SELECT COUNT(*) FROM reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('good')`
    -> `AND MATCH(reviews_virtual) AGAINST('great');`
+----------+
| COUNT(*) |
+----------+
|      620 |
+----------+
```

通过布尔模式搜索是要求多个单词的更简单方法。为此，在搜索字符串中的每个单词前面加上`+`字符，并在字符串后添加`IN BOOLEAN MODE`：

```
mysql> `SELECT COUNT(*) FROM reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('+good +great' IN BOOLEAN MODE);`
+----------+
| COUNT(*) |
+----------+
|     620  |
+----------+
```

布尔模式搜索还允许您通过在每个词前面加上`-`字符来排除词。以下查询选择包含名称为<q>good</q>但不包含<q>great</q>的`reviews`行，反之亦然：

```
mysql> `SELECT COUNT(*) FROM reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('+good -great' IN BOOLEAN MODE);`
+----------+
| COUNT(*) |
+----------+
|      235 |
+----------+
mysql> `SELECT COUNT(*) FROM reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('-good +great' IN BOOLEAN MODE);`
+----------+
| COUNT(*) |
+----------+
|      475 |
+----------+
```

在布尔搜索中，另一个有用的特殊字符是`*`；当附加到搜索词后面时，它作为通配符操作符。以下语句查找包含不仅`use`，还包括诸如`user`、`useful`和`useless`等单词的行：

```
mysql> `SELECT COUNT(*) FROM reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('use*' IN BOOLEAN MODE);`
+----------+
| COUNT(*) |
+----------+
|     1475 |
+----------+
```

关于布尔全文操作符的完整列表，请参阅[MySQL 参考手册](https://dev.mysql.com/doc/refman/8.0/en/fulltext-boolean.html)。

# 7.18 执行全文短语搜索

## 问题

您希望对短语执行全文搜索；即，搜索相邻且按指定顺序出现的单词。

## 解决方案

使用全文短语搜索功能。

## 讨论

要查找包含特定短语的行，简单的全文搜索不起作用：

```
mysql> `SELECT COUNT(*) FROM reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('great product');`
+----------+
| COUNT(*) |
+----------+
|     1725 |
+----------+
```

查询返回了结果，但不是您要查找的结果。全文搜索根据每个单词的存在情况计算相关性排名，不管它在`reviews_virtual`列中出现的位置，并且只要任何单词存在，排名就不为零。因此，这种语句往往会找到太多行。

相反，请使用全文布尔模式，它支持短语搜索。在搜索字符串内用双引号括起短语：

```
mysql> `SELECT COUNT(*) FROM reviews`
    -> `WHERE MATCH(reviews_virtual) AGAINST('"great product"' IN BOOLEAN MODE);`
+----------+
| COUNT(*) |
+----------+
|      216 |
+----------+
```

如果列中包含与短语中相同的单词且顺序相同，则短语匹配成功。
