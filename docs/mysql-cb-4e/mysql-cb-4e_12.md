# 第十二章：处理元数据

# 12.0 介绍

到目前为止使用的大多数 SQL 语句都是为了与数据库中存储的数据一起工作而编写的。毕竟，这就是数据库设计用来保存的内容。但有时您需要的不仅仅是数据值，而是描述或表征这些值的信息，即语句元数据。元数据最常用于处理结果集，但也适用于与 MySQL 交互的其他方面。本章描述了如何获取和使用几种类型的元数据：

语句结果的信息

对于删除或更新行的语句，您可以确定更改了多少行。对于`SELECT`语句，可以获取结果集中的列数，以及有关结果集中每个列的信息，例如列名及其显示宽度。例如，为了格式化表格显示，可以确定每列的宽度以及是否将值左对齐或右对齐。

数据库和表的信息

可以查询 MySQL 服务器以确定它管理哪些数据库和表。这对于存在性测试或生成列表非常有用。例如，一个应用程序可能呈现一个显示，让用户选择可用数据库之一。可以检查表的元数据以确定列的定义；例如，确定`ENUM`或`SET`列的合法值，以生成对应于可用选择的 Web 表单元素。

MySQL 服务器的信息

数据库服务器提供关于自身和与其当前会话状态的信息。了解服务器版本对于确定其是否支持特定功能很有用，这有助于您构建适应性应用程序。

元数据与数据库系统的实现密切相关，因此往往依赖于特定的数据库系统。这意味着，如果一个应用程序使用本章展示的技术，那么如果将其移植到其他数据库系统，可能需要进行一些修改。例如，在 MySQL 中，可以通过执行`SHOW`语句来获取表和数据库列表。但是，`SHOW`是 SQL 的 MySQL 特定扩展，因此即使对于像 Perl DBI、PHP PDO、Python DB API 和 JDBC 这样的 API，它们提供了一个与数据库无关的执行语句的方式，SQL 本身仍然是 MySQL 特定的，必须进行修改才能与其他数据库系统一起工作。

更便携的元数据来源是`INFORMATION_SCHEMA`，这是一个包含有关数据库、表、列、字符集等信息的数据库。`INFORMATION_SCHEMA`相对于`SHOW`有一些优势：

+   其他数据库系统支持`INFORMATION_SCHEMA`，因此使用它的应用程序可能比使用`SHOW`语句更具可移植性。

+   `INFORMATION_SCHEMA`与标准的`SELECT`语法一起使用，因此与其他数据检索操作更相似，而不是`SHOW`语句。

因为这些优势，本章中的示例使用 `INFORMATION_SCHEMA` 而不是大多数情况下的 `SHOW`。

`INFORMATION_SCHEMA` 的一个缺点是，访问它的语句比相应的 `SHOW` 语句更冗长。当你编写程序时，这并不重要，但对于交互使用而言，`SHOW` 语句可能更吸引人，因为需要输入更少。

###### 注意

从 `INFORMATION_SCHEMA` 或 `SHOW` 检索到的结果取决于你的权限。你只会看到那些你拥有某些权限的数据库或表的信息。因此，如果存在对象但你无权访问它，对象的存在测试将返回 false。你可能需要使用具有管理权限的用户才能重复本章中我们提供的所有代码示例。

本章中使用的创建表的脚本位于 `recipes` 发行版的 *tables* 目录中。包含示例代码的脚本位于 *metadata* 目录中。（其中一些使用位于 *lib* 目录中的实用函数。）该发行版通常提供了除显示语言外的其他语言的实现。

# 12.1 确定语句影响的行数

## 问题

你想知道 SQL 语句改变了多少行。

## 解决方案

有些 API 将行数作为执行语句的函数的返回值返回。其他的则提供了一个单独的函数，在执行语句后调用。使用你使用的编程语言中可用的方法。

## 讨论

对于影响行的语句（`UPDATE`、`DELETE`、`INSERT`、`REPLACE`），每个 API 都提供了一种确定涉及行数的方法。对于 MySQL，<q>受影响的</q> 的默认含义是 <q>更改的</q>，而不是 <q>匹配的</q>。也就是说，语句未更改的行不会计数，即使它们匹配语句中指定的条件。例如，以下 `UPDATE` 语句的 <q>受影响的</q> 值为零，因为它没有更改列的当前值，无论 `WHERE` 子句匹配多少行：

```
UPDATE profile SET cats = 0 WHERE cats = 0;
```

MySQL 服务器允许客户端设置连接时的标志，以指示它想要匹配行数而不是更改行数的计数。在这种情况下，前面语句的行数将等于具有 `cats` 值为 0 的行数，即使语句对表没有净变化。但并非所有 MySQL API 都公开此标志。以下讨论指出了哪些 API 允许你选择所需的计数类型，哪些默认使用行匹配计数而不是行更改计数。

### Perl

在 Perl DBI 脚本中，`do()` 返回修改行的行数：

```
my $count = $dbh->do ($stmt);
# report 0 rows if an error occurred
printf "Number of rows affected: %d\n", (defined ($count) ? $count : 0);
```

如果首先准备语句，然后执行它，`execute()` 将返回行数：

```
my $sth = $dbh->prepare ($stmt);
my $count = $sth->execute ();
printf "Number of rows affected: %d\n", (defined ($count) ? $count : 0);
```

要告知 MySQL 返回更改行数还是匹配行数，请在连接到 MySQL 服务器时，在数据源名称（DSN）参数的选项部分指定 `mysql_client_found_rows`。将该选项设置为 0 表示更改行数，设置为 1 表示匹配行数。以下是一个示例：

```
my $conn_attrs = {PrintError => 0, RaiseError => 1, AutoCommit => 1};
my $dsn = "DBI:mysql:cookbook:localhost;mysql_client_found_rows=1";
my $dbh = DBI->connect ($dsn, "cbuser", "cbpass", $conn_attrs);
```

`mysql_client_found_rows` 在会话期间更改行报告行为。

尽管 MySQL 本身的默认行为是返回更改的行数，但当前版本的 Perl DBI 驱动程序自动请求匹配的行数，除非另有指定。对于依赖特定行为的应用程序，最好在 DSN 中明确设置 `mysql_client_found_rows` 选项以获取适当的值。

### Ruby

在 Ruby Mysql2 脚本中，`affected_rows` 方法返回修改行的行数：

```
client.query(stmt)
puts "Number of rows affected: #{client.affected_rows}"
```

如果使用准备语句的 `execute` 方法执行语句，则可以使用语句句柄的 `affected_rows` 方法在之后获取计数：

```
sth = client.prepare(stmt)
sth.execute()
puts "Number of rows affected: #{sth.affected_rows}"
```

Ruby MySQL 驱动程序默认返回更改的行数，但支持 `Mysql2::Client::FOUND_ROWS` 选项，该选项允许您控制服务器返回更改的行数或匹配的行数。例如，要请求匹配的行数，请执行以下操作：

```
client = Mysql2::Client.new(:flags=>Mysql2::Client::FOUND_ROWS, :database=>'cookbook')
```

### PHP

在 PDO 中，数据库句柄的 `exec()` 方法返回受影响的行数：

```
$count = $dbh->exec ($stmt);
printf ("Number of rows updated: %d\n", $count);
```

如果使用 `prepare()` 加 `execute()`，则可以从语句句柄的 `rowCount()` 方法获取受影响的行数：

```
$sth = $dbh->prepare ($stmt);
$sth->execute ();
printf ("Number of rows updated: %d\n", $sth->rowCount ());
```

PDO MySQL 驱动程序默认返回更改的行数，但支持 `PDO::MYSQL_ATTR_FOUND_ROWS` 属性，您可以在连接时指定该属性来控制服务器返回更改的行数或匹配的行数。`new PDO` 类构造函数在密码参数之后接受一个可选的键值数组。在此数组中传递 `PDO::MYSQL_ATTR_FOUND_ROWS => 1` 来请求匹配的行数：

```
$dsn = "mysql:host=localhost;dbname=cookbook";
$dbh = new PDO ($dsn, "cbuser", "cbpass",
                array (PDO::MYSQL_ATTR_FOUND_ROWS => 1));
```

### Python

Python 的 DB API 通过语句游标的 `rowcount` 属性提供更改的行数：

```
cursor = conn.cursor()
cursor.execute(stmt)
print("Number of rows affected: %d" % cursor.rowcount)
cursor.close()
```

要获取匹配行数，请导入 Connector/Python 客户端标志常量，并在 `connect()` 方法的 `client_flags` 参数中传递 `FOUND_ROWS` 标志：

```
from mysql.connector.constants import ClientFlag

conn = mysql.connector.connect(
  database="cookbook",
  host="localhost",
  user="cbuser",
  password="cbpass",
  client_flags=[ClientFlag.FOUND_ROWS]
)
```

### Go

Go SQL 驱动程序提供了 `Result` 类型的 `RowsAffected` 方法，返回更改的行数。

```
res, err := db.Exec(sql)
// Check and handle err
affectedRows, err := res.RowsAffected()
// Check and handle err
fmt.Printf("The statement affected %d rows\n", affectedRows)
```

要检索匹配行数，请在连接字符串中添加参数 `clientFoundRows=true`：

```
db, err := ↩
sql.Open("mysql", "cbuser:cbpass@tcp(127.0.0.1:3306)/cookbook?clientFoundRows=true")
```

### Java

对于修改行的语句，Connector/J 驱动程序提供了匹配行数而不是更改行数，以符合 Java JDBC 规范。

JDBC 接口提供两种不同的行数计数方式，取决于执行语句的方法。如果使用 `executeUpdate()`，则行数计数是其返回值：

```
Statement s = conn.createStatement ();
int count = s.executeUpdate (stmt);
s.close ();
System.out.println ("Number of rows affected: " + count);
```

如果使用了`execute()`，该方法返回 true 或 false 来指示语句是否产生了结果集。对于诸如`UPDATE`或`DELETE`这样不返回结果集的语句，`execute()`返回 false，并且可以通过调用`getUpdateCount()`方法获取行数：

```
Statement s = conn.createStatement ();
if (!s.execute (stmt))
{
  // there is no result set, print the row count
  System.out.println ("Number of rows affected: " + s.getUpdateCount ());
}
s.close ();
```

# 12.2 获取结果集元数据

## 问题

在检索行（参见 Recipe 4.4）之后，您需要了解关于结果集的其他细节，如列名和数据类型，或者有多少行和列。

## 解决方案

利用您的 API 提供的功能。

## 讨论

诸如`SELECT`这样生成结果集的语句会生成多种类型的元数据。本节讨论了通过每个 API 获取的信息，使用显示在执行示例语句（`SELECT` `name,` `birth` `FROM` `profile`）后可用的结果集元数据的程序。示例程序说明了此信息的最简单用法之一：当您从结果集检索行并希望在循环中处理列值时，存储在元数据中的列计数将作为循环迭代器的上限。

### Perl

从 Perl DBI 获取的结果集元数据的范围取决于您如何处理查询：

+   使用语句句柄

    在这种情况下，调用`prepare()`以获取语句句柄。该句柄具有一个`execute()`方法。调用它生成结果集，然后在循环中获取行。使用这种方法，在结果集处于活动状态（即调用`execute()`之后，直到到达结果集的末尾）时，可以访问元数据。当行提取方法发现没有更多的行时，会隐式地调用`finish()`，导致元数据变得不可用。（如果您显式调用`finish()`，也会发生这种情况。）因此，通常最好在调用`execute()`后立即访问元数据，并复制任何您需要在提取循环结束后使用的值的副本。

+   使用将结果集一次性返回的数据库句柄方法

    使用这种方法，处理语句时生成的任何元数据都将在方法返回时被丢弃。您仍然可以通过结果集的大小确定行数和列数。

当您使用语句句柄处理查询时，DBI 在调用句柄的`execute()`方法后会提供结果集元数据。这些信息主要以数组引用的形式提供。对于每种元数据类型，数组中每一列都有一个元素。可以将这些数组引用作为语句句柄的属性访问。例如，`$sth->{NAME}`指向列名数组，数组的各个元素即为列名：

```
$name = $sth->{NAME}->[$i];
```

或者像这样访问整个数组：

```
@names = @{$sth->{NAME}};
```

表 12-1 列出了通过数组访问元数据的属性名称以及每个数组中值的含义。以大写字母开头的名称是标准的 DBI 属性，应该适用于大多数数据库引擎。以 `mysql_` 开头的属性名称是 MySQL 特定的，不可移植的：

表 12-1\. Perl 中的元数据

| 属性名称 | 数组元素含义 |
| --- | --- |
| `NAME` | 列名 |
| `NAME_lc` | 列名小写 |
| `NAME_uc` | 列名大写 |
| `NULLABLE` | 0 或空字符串 = 列值不能为 `NULL` |
|  | 1 = 列值可以为 `NULL` |
|  | 2 = 未知 |
| `PRECISION` | 列宽度 |
| `SCALE` | 小数位数（对于数字列） |
| `TYPE` | 数据类型（数字 DBI 代码） |
| `mysql_is_blob` | 如果列具有 `BLOB`（或 `TEXT`）类型则为真 |
| `mysql_is_key` | 如果列是键的一部分则为真 |
| `mysql_is_num` | 如果列具有数字类型则为真 |
| `mysql_is_pri_key` | 如果列是主键则为真 |
| `mysql_max_length` | 结果集中列值的实际最大长度 |
| `mysql_table` | 列所属表的名称 |
| `mysql_type` | 数据类型（数字内部 MySQL 代码） |
| `mysql_type_name` | 数据类型名称 |

一些元数据类型，列在 表 12-2 中列出，作为哈希的引用而不是数组来访问。这些哈希每个列值有一个元素。元素键是列名，其值是结果集中的列位置。例如：

```
$col_pos = $sth->{NAME_hash}->{*`col_name`*};
```

表 12-2\. Perl 中的元数据，可作为哈希引用访问

| 属性名称 | 哈希元素含义 |
| --- | --- |
| `NAME_hash` | 列名 |
| `NAME_hash_lc` | 列名小写 |
| `NAME_hash_uc` | 列名大写 |

结果集中的列数可作为标量值获得：

```
$num_cols = $sth->{NUM_OF_FIELDS};
```

这个示例代码展示了如何执行语句并显示结果集元数据：

```
my $stmt = "SELECT name, birth FROM profile";
printf "Statement: %s\n", $stmt;
my $sth = $dbh->prepare ($stmt);
$sth->execute();
# metadata information becomes available at this point ...
printf "NUM_OF_FIELDS: %d\n", $sth->{NUM_OF_FIELDS};
print "Note: statement has no result set\n" if $sth->{NUM_OF_FIELDS} == 0;
for my $i (0 .. $sth->{NUM_OF_FIELDS}-1)
{
  printf "--- Column %d (%s) ---\n", $i, $sth->{NAME}->[$i];
  printf "NAME_lc:          %s\n", $sth->{NAME_lc}->[$i];
  printf "NAME_uc:          %s\n", $sth->{NAME_uc}->[$i];
  printf "NULLABLE:         %s\n", $sth->{NULLABLE}->[$i];
  printf "PRECISION:        %d\n", $sth->{PRECISION}->[$i];
  printf "SCALE:            %d\n", $sth->{SCALE}->[$i];
  printf "TYPE:             %d\n", $sth->{TYPE}->[$i];
  printf "mysql_is_blob:    %s\n", $sth->{mysql_is_blob}->[$i];
  printf "mysql_is_key:     %s\n", $sth->{mysql_is_key}->[$i];
  printf "mysql_is_num:     %s\n", $sth->{mysql_is_num}->[$i];
  printf "mysql_is_pri_key: %s\n", $sth->{mysql_is_pri_key}->[$i];
  printf "mysql_max_length: %d\n", $sth->{mysql_max_length}->[$i];
  printf "mysql_table:      %s\n", $sth->{mysql_table}->[$i];
  printf "mysql_type:       %d\n", $sth->{mysql_type}->[$i];
  printf "mysql_type_name:  %s\n", $sth->{mysql_type_name}->[$i];
}
$sth->finish ();  # release result set because we didn't fetch its rows
```

程序产生以下输出：

```
Statement: SELECT name, birth FROM profile
NUM_OF_FIELDS: 2
--- Column 0 (name) ---
NAME_lc:          name
NAME_uc:          NAME
NULLABLE:
PRECISION:        20
SCALE:            0
TYPE:             12
mysql_is_blob:
mysql_is_key:
mysql_is_num:     0
mysql_is_pri_key:
mysql_max_length: 7
mysql_table:      profile
mysql_type:       253
mysql_type_name:  varchar
--- Column 1 (birth) ---
NAME_lc:          birth
NAME_uc:          BIRTH
NULLABLE:         1
PRECISION:        10
SCALE:            0
TYPE:             9
mysql_is_blob:
mysql_is_key:
mysql_is_num:     0
mysql_is_pri_key:
mysql_max_length: 10
mysql_table:      profile
mysql_type:       10
mysql_type_name:  date
```

要从调用 `execute()` 生成的结果集中获取行数，请自行获取行并计数。在 DBI 文档中明确不推荐使用 `$sth->rows()` 来获取 `SELECT` 语句的计数。

您还可以通过调用使用数据库句柄而不是语句句柄的 DBI 方法之一（如 `selectall_arrayref()` 或 `selectall_hashref()`）来获取结果集。这些方法不提供对列元数据的访问。该信息在方法返回时已被处理，无法在脚本中使用。但是，您可以通过检查结果集本身来推导列和行计数。Recipe 4.4 讨论了几种方法产生的结果集结构以及如何使用它们来获取行和列计数。

### Ruby

Ruby Mysql2 gem 在执行语句后不提供自己的方法来访问结果集的元数据。你只能通过调用`Mysql2::Result`类的`fields`方法获取列名。

```
stmt = "SELECT name, birth FROM profile"
puts "Statement: #{stmt}"
sth = client.prepare(stmt)
res = sth.execute()
# metadata information becomes available at this point ...
puts "Number of columns: #{res.fields.size}"
puts "Note: statement has no result set" if res.count == 0
puts "Columns names: #{res.fields.join(", ")}"
res.free
```

要获取其他列的元数据，请参考我们在 Recipe 12.5 中建议的 Information Schema 查询。

### PHP

在 PHP 中，对于`SELECT`语句的元数据可以在成功调用`query()`后从 PDO 中获取。如果您使用`prepare()`加`execute()`执行语句（适用于`SELECT`或非`SELECT`语句），则在`execute()`后元数据将可用。

要确定元数据的可用性，请检查语句句柄的`columnCount()`方法是否返回大于零的值。如果是，则句柄的`getColumnMeta()`方法将返回一个包含单个列的元数据的关联数组。下表显示了该数组的元素。（对于其他数据库系统，`flags`值的格式可能有所不同。）

表格 12-3\. PHP 中的元数据

| Name | 值 |
| --- | --- |
| `pdo_type` | 列类型（对应于`PDO::PARAM_`*`XXX`*的值） |
| `native_type` | 列值的 PHP 本机类型 |
| `name` | 列名 |
| `len` | 列长度 |
| `precision` | 列精度 |
| `flags` | 描述列属性的标志数组 |
| `table` | 表格的名称 |

这个示例代码展示了如何执行语句并显示结果集的元数据：

```
$stmt = "SELECT name, birth FROM profile";
print ("Statement: $stmt\n");
$sth = $dbh->prepare ($stmt);
$sth->execute ();
# metadata information becomes available at this point ...
$ncols = $sth->columnCount ();
print ("Number of columns: $ncols\n");
if ($ncols == 0)
  print ("Note: statement has no result set\n");
for ($i = 0; $i < $ncols; $i++)
{
  $col_info = $sth->getColumnMeta ($i);
  $flags = implode (",", array_values ($col_info["flags"]));
  printf ("--- Column %d (%s) ---\n", $i, $col_info["name"]);
  printf ("pdo_type:     %d\n", $col_info["pdo_type"]);
  printf ("native_type:  %s\n", $col_info["native_type"]);
  printf ("len:          %d\n", $col_info["len"]);
  printf ("precision:    %d\n", $col_info["precision"]);
  printf ("flags:        %s\n", $flags);
  printf ("table:        %s\n", $col_info["table"]);
}
```

程序产生如下输出：

```
Statement: SELECT name, birth FROM profile
Number of columns: 2
--- Column 0 (name) ---
PDO type:     2
native type:  VAR_STRING
len:          20
precision:    0
flags:        not_null
table:        profile
--- Column 1 (birth) ---
PDO type:     2
native type:  DATE
len:          10
precision:    0
flags:
table:        profile
```

要从返回行的语句中获取行数，请获取行并自行计数。`rowCount()`方法不能保证适用于结果集。

### Python

对于产生结果集的语句，Python 的 DB API 提供了行和列计数，以及有关各个列的少量信息项。

要获取结果集的行数，请访问游标的`rowcount`属性。这要求游标被缓冲，以便立即获取查询结果；否则，您必须在获取时计算行数。直接获取列数不可用，但在调用`fetchone()`或`fetchall()`后，可以将结果集行元组的长度作为列数。也可以在不获取任何行的情况下，通过使用`cursor.description`确定列数。这是一个元组，每个元素代表结果集中每一列的元数据，因此其长度告诉您结果集中有多少列。（如果语句生成没有结果集，例如对于`UPDATE`，`description`的值为`None`。）`description`元组的每个元素都是另一个元组，表示结果集相应列的元数据。对于 Connector/Python，只有少数几个`description`值是有意义的。以下代码显示了如何访问它们：

```
stmt = "SELECT name, birth FROM profile"
print("Statement: %s" % stmt)
# buffer cursor so that rowcount has usable value
cursor = conn.cursor(buffered=True)
cursor.execute(stmt)
# metadata information becomes available at this point ...
print("Number of rows: %d" % cursor.rowcount)
if cursor.description is None:  # no result set
  ncols = 0
else:
  ncols = len(cursor.description)
print("Number of columns: %d" % ncols)
if ncols == 0:
  print("Note: statement has no result set")
for i, col_info in enumerate(cursor.description):
  # print name, then other information
  name, type, _, _, _, _, nullable, flags, _ = col_info
  print("--- Column %d (%s) ---" % (i, name))
  print("Type: %d (%s)" % (type, FieldType.get_info(type)))
  print("Nullable: %d" % (nullable))
  print("Flags: %d" % (flags))
cursor.close()
```

代码使用了导入如下的`FieldType`类：

```
from mysql.connector import FieldType
```

程序产生如下输出：

```
Statement:  SELECT name, birth FROM profile
Number of rows: 10
Number of columns: 2
--- Column 0 (name) ---
Type:     253 (VAR_STRING)
Nullable: 0
Flags:    4097
--- Column 1 (birth) ---
Type:     10 (DATE)
Nullable: 1
Flags:    128
```

### Go

Go 通过`Rows.ColumnTypes`方法返回作为`ColumnType`值数组的列元数据。您可以查询数组的每个成员以获取列的特定特性。

表 12-4 包含`ColumnType`支持的方法。

表 12-4\. Go 中的元数据

| 方法名称 | 描述 |
| --- | --- |
| `DatabaseTypeName` | 数据库类型，如`INT`或`VARCHAR`。 |
| `DecimalSize` | 十进制类型的比例和精度 |
| `Length` | 变长文本和二进制列的列类型长度。MySQL 驱动程序不支持。 |
| `Name` | 列的名称或别名。 |
| `Nullable` | 列是否可为空。 |
| `ScanType` | 适合扫描到`Rows.Scan`中的本地 Go 类型。 |

如果使用`Rows.Columns`方法，还可以获取列名列表。它返回包含列名或别名的字符串数组。

示例代码演示了如何在 Go 应用程序中获取列名和元数据。

```
package main

import (
  "fmt"
  "log"
  "github.com/svetasmirnova/mysqlcookbook/recipes/lib/cookbook"
)

func main() {
  db := cookbook.Connect()
  defer db.Close()

  stmt := "SELECT name, birth FROM profile"
  fmt.Printf("Statement: %s\n", stmt)

  rows, err := db.Query(stmt)
  if err != nil {
    log.Fatal(err)
  }
  defer rows.Close()

  // metadata information becomes available at this point ...
  cols, err := rows.ColumnTypes()
  if err != nil {
    log.Fatal(err)
  }

  ncols := len(cols)
  fmt.Printf("Number of columns: %d\n", ncols)
  if (ncols == 0) {
    fmt.Println("Note: statement has no result set")
  }

  for i := 0; i < ncols; i++ {
    fmt.Printf("---- Column %d (%s) ----\n", i, cols[i].Name())
    fmt.Printf("DatabaseTypeName: %s\n", cols[i].DatabaseTypeName())

    collen, ok := cols[i].Length()
    if ok {
      fmt.Printf("Length: %d\n", collen)
    }

    precision, scale, ok := cols[i].DecimalSize()
    if ok {
      fmt.Printf("DecimalSize precision: %d, scale: %d\n", precision, scale)
    }

    colnull, ok := cols[i].Nullable()
    if ok {
      fmt.Printf("Nullable: %t\n", colnull)
    }

    fmt.Printf("ScanType: %s\n", cols[i].ScanType())
  }
}
```

该程序生成以下输出：

```
Statement: SELECT name, birth FROM profile
Number of columns: 2
---- Column 0 (name) ----
DatabaseTypeName: VARCHAR
Nullable: false
ScanType: sql.RawBytes
---- Column 1 (birth) ----
DatabaseTypeName: DATE
Nullable: true
ScanType: sql.NullTime
```

### Java

JDBC 通过调用您的`ResultSet`对象的`getMetaData()`方法获得结果集元数据，通过`ResultSetMetaData`对象提供对几种信息的访问。其`getColumnCount()`方法返回结果集中的列数。与我们的其他 API 不同，对于 JDBC，列索引从 1 开始而不是从 0 开始：

```
String stmt = "SELECT name, birth FROM profile";
System.out.println("Statement: " + stmt);
Statement s = conn.createStatement();
s.executeQuery(stmt);
ResultSet rs = s.getResultSet();
ResultSetMetaData md = rs.getMetaData();
// metadata information becomes available at this point ...
int ncols = md.getColumnCount();
System.out.println("Number of columns: " + ncols);
if (ncols == 0)
  System.out.println ("Note: statement has no result set");
for (int i = 1; i <= ncols; i++) { // column index values are 1-based
  System.out.println("--- Column " + i
            + " (" + md.getColumnName (i) + ") ---");
  System.out.println("getColumnDisplaySize: " + md.getColumnDisplaySize (i));
  System.out.println("getColumnLabel:       " + md.getColumnLabel (i));
  System.out.println("getColumnType:        " + md.getColumnType (i));
  System.out.println("getColumnTypeName:    " + md.getColumnTypeName (i));
  System.out.println("getPrecision:         " + md.getPrecision (i));
  System.out.println("getScale:             " + md.getScale (i));
  System.out.println("getTableName:         " + md.getTableName (i));
  System.out.println("isAutoIncrement:      " + md.isAutoIncrement (i));
  System.out.println("isNullable:           " + md.isNullable (i));
  System.out.println("isCaseSensitive:      " + md.isCaseSensitive (i));
  System.out.println("isSigned:             " + md.isSigned (i));
}
rs.close();
s.close();
```

该程序生成以下输出：

```
Statement: SELECT name, birth FROM profile
Number of columns: 2
--- Column 1 (name) ---
getColumnDisplaySize: 20
getColumnLabel:       name
getColumnType:        12
getColumnTypeName:    VARCHAR
getPrecision:         20
getScale:             0
getTableName:         profile
isAutoIncrement:      false
isNullable:           0
isCaseSensitive:      false
isSigned:             false
--- Column 2 (birth) ---
getColumnDisplaySize: 10
getColumnLabel:       birth
getColumnType:        91
getColumnTypeName:    DATE
getPrecision:         10
getScale:             0
getTableName:         profile
isAutoIncrement:      false
isNullable:           1
isCaseSensitive:      false
isSigned:             false
```

结果集的行数不直接可用；您必须获取行并计数它们。

JDBC 有几个其他结果集元数据调用，但其中许多对 MySQL 没有实用信息。要尝试它们，请参考 JDBC 参考资料以查看这些调用，并修改程序以查看它们是否返回任何内容。

# 12.3 列出或检查数据库或表的存在性

## 问题

您希望列出 MySQL 服务器托管的数据库或数据库中的表，或者您想检查特定数据库或表是否存在。

## 解决方案

使用`INFORMATION_SCHEMA`获取这些信息。`SCHEMATA`表包含每个数据库的一行，`TABLES`表包含每个数据库中每个表或视图的一行。

## 讨论

要检索服务器托管的数据库列表，请使用以下语句：

```
SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA;
```

要对结果进行排序，请添加`ORDER` `BY` `SCHEMA_NAME`子句。

要检查特定数据库是否存在，请使用带有命名数据库的条件的`WHERE`子句。如果返回一行，则数据库存在。以下 Ruby 方法显示了如何执行数据库的存在性测试：

```
def database_exists(client, db_name)
  sth = client.prepare("SELECT SCHEMA_NAME
 FROM INFORMATION_SCHEMA.SCHEMATA
 WHERE SCHEMA_NAME = ?")
  return sth.execute(db_name).count > 0
end
```

要获取数据库中表的列表，请在选择`TABLES`表的语句的`WHERE`子句中命名数据库：

```
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'cookbook';
```

要对结果进行排序，请添加`ORDER` `BY` `TABLE_NAME`子句。

要获取默认数据库中表的列表，请改用此语句代替：

```
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = DATABASE();
```

如果没有选择数据库，`DATABASE()` 将返回 `NULL`，没有匹配的行，这是正确的结果。

要检查特定表是否存在，请使用带有命名表名的条件的 `WHERE` 子句。以下是在给定数据库中执行表存在性测试的 Ruby 方法：

```
def table_exists(client, db_name, tbl_name)
  sth = client.prepare("SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES
 WHERE TABLE_SCHEMA = ? AND TABLE_NAME = ?")
  return sth.execute(db_name, tbl_name).count > 0
end
```

一些 API 提供了一种与数据库无关的方法来获取数据库或表列表。在 Perl DBI 中，数据库句柄的 `tables()` 方法返回默认数据库中表的列表：

```
@tables = $dbh->tables ();
```

对于 Java，有专门设计用于返回数据库或表列表的 JDBC 方法。对于每种方法，调用连接对象的 `getMetaData()` 方法，并使用生成的 `DatabaseMetaData` 对象检索所需的信息。以下是生成数据库列表的方法：

```
// get list of databases
DatabaseMetaData md = conn.getMetaData ();
ResultSet rs = md.getCatalogs ();
while (rs.next ())
  System.out.println (rs.getString (1));  // column 1 = database name
rs.close ();
```

要列出数据库中的表，请执行以下操作：

```
// get list of tables in database named by dbName; if
// dbName is the empty string, the default database is used
DatabaseMetaData md = conn.getMetaData ();
ResultSet rs = md.getTables (dbName, "", "%", null);
while (rs.next ())
  System.out.println (rs.getString (3));  // column 3 = table name
rs.close ();
```

# 12.4 列出或检查视图的存在

## 问题

您想要检查您的数据库是否包含视图。

## 解决方案

仅从 `INFORMATION_SCHEMA.TABLES` 表中选择那些具有 `TABLE_TYPE` 等于 `VIEW` 的表。

## 讨论

方法，在 Recipe 12.3 中显示了物理表和视图。如果您需要区分它们，请使用子句 `WHERE TABLE_TYPE='VIEW'` 仅列出视图：

```
mysql> `SELECT` `TABLE_SCHEMA``,` `TABLE_NAME``,` `TABLE_TYPE` 
    -> `FROM` `INFORMATION_SCHEMA``.``TABLES` 
    -> `WHERE` `TABLE_TYPE``=``'VIEW'` `AND` `TABLE_SCHEMA``=``'cookbook'``;`
+--------------+---------------------+------------+ | TABLE_SCHEMA | TABLE_NAME          | TABLE_TYPE |
+--------------+---------------------+------------+ | cookbook     | patients_statistics | VIEW       |
+--------------+---------------------+------------+ 1 row in set (0,00 sec)
```

如果您只想列出物理表，可以使用条件 `TABLE_TYPE='BASE TABLE'`：

```
mysql> `SELECT` `TABLE_SCHEMA``,` `TABLE_NAME``,` `TABLE_TYPE` 
    -> `FROM` `INFORMATION_SCHEMA``.``TABLES` 
    -> `WHERE` `TABLE_TYPE``=``'BASE TABLE'` `AND` `TABLE_SCHEMA``=``'cookbook'` 
    -> `AND` `TABLE_NAME` `LIKE` `'trip%'``;`
+--------------+------------+------------+ | TABLE_SCHEMA | TABLE_NAME | TABLE_TYPE |
+--------------+------------+------------+ | cookbook     | trip_leg   | BASE TABLE |
| cookbook     | trip_log   | BASE TABLE |
+--------------+------------+------------+ 2 rows in set (0,00 sec)
```

# 12.5 访问表列定义

## 问题

您想要找出表具有哪些列及其定义方式。

## 解决方案

有几种方法可以做到这一点。您可以从 `INFORMATION_SCHEMA`、`SHOW` 语句或 *mysqldump* 中获取列定义。

## 讨论

关于表结构的信息使您能够回答诸如 <q>表包含哪些列及其类型？</q> 或 <q>`ENUM` 或 `SET` 列的合法值是什么？</q> 等问题。以下是这种信息的一些应用程序：

显示列列表

表信息的一个简单用途是呈现表的列列表。这在基于 Web 或 GUI 的应用程序中很常见，用户可以通过从列表中选择表列并输入要与列值进行比较的值来交互地构造语句。

交互式记录编辑

对表结构的了解对于修改数据的交互应用非常有用。假设一个应用从数据库检索记录，显示包含记录内容的表单以便用户编辑，然后在用户修改表单并提交后更新数据库中的记录。您可以使用表结构信息来验证列值，确保不会尝试插入无效值到数据库中。如果某列是 `ENUM` 类型，您可以查找有效的枚举值，并检查用户提交的值是否合法。如果列是整数类型，检查提交的值确保完全由数字组成，可能前面带有 `+` 或 `−` 符号。如果列包含日期，查找合法的日期格式。

但如果用户将字段留空呢？如果字段对应于表中的 `CHAR` 列，您将列值设置为 `NULL` 还是空字符串？这也是可以通过检查表结构来回答的问题。确定列是否可以包含 `NULL` 值。如果可以，将列设置为 `NULL`；否则，将其设置为空字符串。

将列定义映射到网页元素

某些数据类型，如 `ENUM` 和 `SET`，自然对应于 Web 表单的元素：

+   `ENUM` 具有固定的一组值，您可以从中选择单个值。这类似于一组单选按钮、弹出菜单或单选滚动列表。

+   `SET` 列类似，但您可以选择多个值；这对应于一组复选框或多选滚动列表。

通过使用表元数据访问这些列类型的定义，您可以轻松确定列的合法值，并将它们映射到适当的表单元素上。Recipe 12.6 讨论了如何获取这些列类型的定义。

MySQL 提供多种方法来获取表结构的信息：

+   从 `INFORMATION_SCHEMA` 检索信息。`COLUMNS` 表包含列定义。

+   使用 `SHOW` `COLUMNS` 语句。

+   使用 `SHOW` `CREATE` `TABLE` 语句或 *mysqldump* 命令行程序获取显示表结构的 `CREATE` `TABLE` 语句。

下面的讨论展示了如何使用每种方法向 MySQL 请求表信息。要尝试这些示例，请创建一个 `item` 表，列出每个项目的 ID、名称和颜色：

```
CREATE TABLE item
(
  id      INT UNSIGNED NOT NULL AUTO_INCREMENT,
  name    CHAR(20),
  colors  ENUM('chartreuse','mauve','lime green','puce') DEFAULT 'puce',
  PRIMARY KEY (id)
);
```

### 使用 INFORMATION_SCHEMA 获取表结构

要获取表中单个列的信息，请查询 `INFORMATION_SCHEMA.COLUMNS` 表：

```
mysql> `SELECT * FROM INFORMATION_SCHEMA.COLUMNS`
    -> `WHERE TABLE_SCHEMA = 'cookbook' AND TABLE_NAME = 'item'`
    -> `AND COLUMN_NAME = 'colors'\G`
*************************** 1\. row ***************************
           TABLE_CATALOG: def
            TABLE_SCHEMA: cookbook
              TABLE_NAME: item
             COLUMN_NAME: colors
        ORDINAL_POSITION: 3
          COLUMN_DEFAULT: puce
             IS_NULLABLE: YES
               DATA_TYPE: enum
CHARACTER_MAXIMUM_LENGTH: 10
  CHARACTER_OCTET_LENGTH: 10
       NUMERIC_PRECISION: NULL
           NUMERIC_SCALE: NULL
      DATETIME_PRECISION: NULL
      CHARACTER_SET_NAME: utf8mb4
          COLLATION_NAME: utf8mb4_0900_ai_ci
             COLUMN_TYPE: enum('chartreuse','mauve','lime green','puce')
              COLUMN_KEY:
                   EXTRA:
              PRIVILEGES: select,insert,update,references
          COLUMN_COMMENT:
```

要获取所有列的信息，请在 `WHERE` 子句中省略 `COLUMN_NAME` 条件。

下面是可能最有用的一些 `COLUMNS` 表列：

+   `COLUMN_NAME`：列名。

+   `ORDINAL_POSITION`：列在表定义中的位置。

+   `COLUMN_DEFAULT`：列的默认值。

+   `IS_NULLABLE`: `YES` 或 `NO` 指示列是否可以包含 `NULL` 值。

+   `DATA_TYPE`, `COLUMN_TYPE`: 数据类型信息。`DATA_TYPE` 是数据类型关键字，`COLUMN_TYPE` 包含类型属性等附加信息。

+   `CHARACTER_SET_NAME`, `COLLATION_NAME`: 字符串列的字符集和排序规则。非字符串列的这些值为 `NULL`.

+   `COLUMN_KEY`: 列索引信息。

从程序内部使用 `INFORMATION_SCHEMA` 内容非常简单。以下是一个 PHP 函数示例，演示了这一过程。它接受数据库和表名参数，从 `INFORMATION_SCHEMA` 中选择以获取表的列名，并将列名作为数组返回。`ORDER` `BY` `ORDINAL_POSITION` 子句确保数组中的名称按照表定义顺序返回：

```
function get_column_names ($dbh, $db_name, $tbl_name)
{
  $stmt = "SELECT COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS
 WHERE TABLE_SCHEMA = ? AND TABLE_NAME = ?
 ORDER BY ORDINAL_POSITION";
  $sth = $dbh->prepare ($stmt);
  $sth->execute (array ($db_name, $tbl_name));
  return ($sth->fetchAll (PDO::FETCH_COLUMN, 0));
}
```

`get_column_names()` 返回一个仅包含列名的数组。如果需要额外的列信息，可以编写一个更通用的 `get_column_info()` 程序，它返回列信息结构的数组。要查看 PHP 以及其他语言的实现，请查看 `recipes` 发行版中 *lib* 目录下的库文件。

### 使用 `SHOW COLUMNS` 获取表结构

`SHOW COLUMNS` 语句为表中的每列生成一行输出，每行提供有关相应列的各种信息。以下示例演示了 `item` 表 `colors` 列的 `SHOW COLUMNS` 输出：

```
mysql> `SHOW COLUMNS FROM item LIKE 'colors'\G`
*************************** 1\. row ***************************
  Field: colors
   Type: enum('chartreuse','mauve','lime green','puce')
   Null: YES
    Key:
Default: puce
  Extra:
```

`SHOW` `COLUMNS` 显示所有列名匹配 `LIKE` 模式的信息。要获取所有列的信息，请省略 `LIKE` 子句。

`SHOW` `COLUMNS` 显示的值对应于 `INFORMATION_SCHEMA` `COLUMNS` 表的以下列：`COLUMN_NAME`, `COLUMN_TYPE`, `COLUMN_KEY`, `IS_NULLABLE`, `COLUMN_DEFAULT`, `EXTRA`.

`SHOW` `FULL` `COLUMNS` 显示每列的额外 `Collation`, `Privileges`, 和 `Comment` 字段。这些对应于 `COLUMNS` 表的 `COLLATION_NAME`, `PRIVILEGES`, 和 `COLUMN_COMMENT` 列。

`SHOW` 以与 `SELECT` 语句的 `WHERE` 子句中的 `LIKE` 操作符相同的方式解释模式。如果指定了文字列名，则字符串仅匹配该名称，并且 `SHOW` `COLUMNS` 仅显示该列的信息。如果列名包含 SQL 模式字符（`%` 或 `_`），您希望字面匹配它们，必须在模式字符串中使用反斜杠进行转义，以避免同时匹配其他名称。

需要转义 `%` 和 `_` 字符以文字匹配 `LIKE` 模式，这也适用于其他允许 `LIKE` 子句中的名称模式的 `SHOW` 语句，如 `SHOW` `TABLES` 和 `SHOW` `DATABASES`.

在程序中，可以使用 API 语言的模式匹配功能在将列名放入 `SHOW` 语句之前转义 SQL 模式字符。在 Perl、Ruby 和 PHP 中，使用以下表达式。

Perl：

```
$name =~ s/([%_])/\\$1/g;
```

Ruby：

```
name = name.gsub(/([%_])/, '\\\\\1')
```

PHP：

```
$name = preg_replace ('/([%_])/', '\\\\$1', $name);
```

对于 Python，请导入 `re` 模块，并使用其 `sub()` 方法：

```
name = re.sub(r'([%_])', r'\\\1', name)
```

对于 Go，请使用 `regexp` 包中的方法：

```
import "regexp"
// ...
  re := regexp.MustCompile(`([_%])`)
  name = re.ReplaceAllString(name, "\\\\$1")
```

对于 Java，请使用 `java.util.regex` 包中的方法：

```
import java.util.regex.*;

Pattern p = Pattern.compile("([_%])");
Matcher m = p.matcher(name);
name = m.replaceAll ("\\\\$1");
```

如果这些表达式看起来有太多反斜杠，请记住 API 语言处理器本身解释反斜杠并在执行模式匹配之前剥离一级。要将字面反斜杠放入结果中，必须在模式中加倍。如果模式处理器剥离集，则需要在此之上添加另一级。

### 使用 SHOW CREATE TABLE 获取表结构

从 MySQL 获取表结构信息的另一种方法是使用定义表的 `CREATE` `TABLE` 语句。要获取此信息，请使用 `SHOW` `CREATE` `TABLE` 语句：

```
mysql> `SHOW CREATE TABLE item\G`
*************************** 1\. row ***************************
       Table: item
Create Table: CREATE TABLE `item` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` char(20) DEFAULT NULL,
  `colors` enum('chartreuse','mauve','lime green','puce') DEFAULT 'puce',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

从命令行，如果使用 `--no-data` 选项，则 *mysqldump* 提供的相同 `CREATE` `TABLE` 信息，这告诉 *mysqldump* 仅转储表的结构而不是其数据。

`CREATE` `TABLE` 格式非常信息丰富且易于阅读，因为它以与创建表时使用的格式相似的格式显示列信息。它还清楚显示索引结构，而其他方法则不然。但是，您可能会发现这种检查表结构的方法在交互中比在程序中更有用。该信息未以常规的行列格式提供，因此解析起来更困难。此外，格式在 MySQL 增强 `CREATE` `TABLE` 语句时可能会更改，这种情况偶尔会发生，因为 MySQL 的功能被扩展。

# 12.6 获取 ENUM 和 SET 列信息

## 问题

您想知道 `ENUM` 或 `SET` 列的成员。

## 解决方案

此问题是获取表结构元数据的子集。从表元数据中获取列定义，然后从定义中提取成员列表。

## 讨论

了解 `ENUM` 或 `SET` 列的允许值列表通常很有用。假设您想要呈现包含与 `ENUM` 列的每个合法值对应的选项的 Web 表单，例如可以订购服装的尺寸或递送包裹的可用递送方式。您可以将选择硬编码到生成表单的脚本中，但是如果稍后更改列（例如添加新的枚举值），则会在列和使用它的脚本之间引入差异。如果改为使用表元数据查找合法值，脚本始终可以生成包含正确值集的弹出菜单。类似的方法适用于 `SET` 列。

要确定`ENUM`或`SET`列的允许值，请使用食谱 12.5 中描述的技术之一获取其定义。例如，如果从`INFORMATION_SCHEMA`的`COLUMNS`表中选择，`item`表的`colors`列的`COLUMN_TYPE`值如下所示：

```
enum('chartreuse','mauve','lime green','puce')
```

`SET`列类似，只是它们说`set`而不是`enum`。对于任一数据类型，通过去除初始单词和括号、在逗号处分割，并从各个值中移除包围引号，可以提取允许的值。

让我们编写一个`get_enumorset_info()`例程来从数据类型定义中提取这些值。顺便说一下，我们可以使该例程返回列的类型、默认值以及值是否可以为`NULL`。然后可以通过可能需要不仅仅是值列表的脚本来使用该例程。以下是 Ruby 版本。它的参数是数据库句柄、数据库名称、表名称和列名称。它返回一个哈希，其中条目对应于列定义的各个方面（如果列不存在则为`nil`）：

```
def get_enumorset_info(client, db_name, tbl_name, col_name)
  sth = client.prepare(
          "SELECT COLUMN_NAME, COLUMN_TYPE, IS_NULLABLE, COLUMN_DEFAULT
 FROM INFORMATION_SCHEMA.COLUMNS
 WHERE TABLE_SCHEMA = ? AND TABLE_NAME = ? AND COLUMN_NAME = ?")
  res = sth.execute(db_name, tbl_name, col_name)
  return nil if res.count == 0  # no such column
  row = res.first
  info = {}
  info["name"] = row.values[0]
  return nil unless row.values[1] =~ /^(ENUM|SET)\((.*)\)$/i # not ENUM or SET
  info["type"] = $1
  # split value list on commas, trim quotes from end of each word
  info["values"] = $2.split(",").collect { |val| val.sub(/^'(.*)'$/, "\\1") }
  # determine whether column can contain NULL values
  info["nullable"] = (row.values[2].upcase == "YES")
  # get default value (nil represents NULL)
  info["default"] = row.values[3]
  return info
end
```

在检查数据类型和可空属性时，此例程使用不区分大小写的匹配。这可以防止元数据结果中未来字母大小写的更改。

以下示例显示了如何访问并显示`get_enumorset_info()`返回的哈希的每个元素：

```
info = get_enumorset_info(client, db_name, tbl_name, col_name)
puts "Information for #{db_name}.#{tbl_name}.#{col_name}:"
if info.nil?
  puts "No information available (not an ENUM or SET column?)"
else
  puts "Name: " + info["name"]
  puts "Type: " + info["type"]
  puts "Legal values: " + info["values"].join(",")
  puts "Nullable: " + (info["nullable"] ? "yes" : "no")
  puts "Default value: " + (info["default"].nil? ? "NULL" : info["default"])
end
```

该代码生成了以下输出，显示了`profile`表的`color`列：

```
Information for cookbook.profile.color:
Name: color
Type: enum
Legal values: blue,red,green,brown,black,white
Nullable: yes
Default value: NULL
```

其他 API 的等效例程类似。您可以在`recipes`分发的*lib*目录中找到实现。这些例程对于验证输入值（参见食谱 14.11）非常有用。

# 12.7 获取服务器元数据

## 问题

您希望获取有关 MySQL 服务器本身的信息，例如其版本、配置以及其组件的当前状态。

## 解决方案

几个 SQL 函数和`SHOW`语句返回有关服务器的信息。

## 讨论

MySQL 具有几个 SQL 函数和语句，可以为您提供有关服务器本身以及当前客户端会话的信息。Table 12-5 显示了一些您可能发现有用的函数。两个`SHOW`语句允许使用`GLOBAL`或`SESSION`关键字选择全局服务器值或特定于您会话的值，并使用`LIKE` `'`*`pattern`*`'`子句来限制结果仅包含与模式匹配的变量名：

表 12-5\. 获取服务器元数据的 SQL 函数和语句

| 语句 | 由语句产生的信息 |
| --- | --- |
| `SELECT VERSION()` | 服务器版本字符串 |
| `SELECT DATABASE()` | 默认数据库名称（如果没有则为`NULL`） |
| `SELECT USER()` | 连接时客户端给出的当前用户 |
| `SELECT CURRENT_USER()` | 用于检查客户端权限的用户 |
| `SHOW [GLOBAL&#124;SESSION] STATUS` | 服务器全局或会话状态指示器 |
| `SHOW [GLOBAL&#124;SESSION] VARIABLES` | 服务器的全局或会话状态配置变量 |

要获取表中任何语句提供的信息，请执行它并处理其结果集。例如，`SELECT DATABASE()` 返回默认数据库的名称，如果没有选择数据库则返回 `NULL`。以下 Ruby 代码使用该语句来呈现包含当前会话信息的状态显示：

```
db = client.query("SELECT DATABASE()").first.values[0]
puts "Default database: " + (db.nil? ? "(no database selected)" : db)
```

给定的 API 可能提供替代方法来执行 SQL 语句以访问这些类型的信息。例如，JDBC 提供了几种数据库无关的方法来获取服务器元数据。使用您的连接对象获取数据库元数据，然后调用适当的方法以获取您感兴趣的信息。查阅 JDBC 参考手册获取完整列表，但以下是一些代表性示例：

```
DatabaseMetaData md = conn.getMetaData();
// can also get this with SELECT VERSION()
System.out.println("Product version: " + md.getDatabaseProductVersion());
// this is similar to SELECT USER() but doesn't include the hostname
System.out.println("Username: " + md.getUserName());
```

## 另请参阅

更多关于在服务器监控环境中使用 `SHOW`（和 `INFORMATION_SCHEMA`）的讨论，请参见第 23.2 章。

# 12.8 编写适应 MySQL 服务器版本的应用程序

## 问题

您希望使用的某个功能仅适用于特定版本的 MySQL。

## 解决方案

向服务器请求其版本号。如果服务器太旧无法支持特定功能，也许可以退而求其次，如果有可行的解决方案的话。或者建议用户升级。

## 讨论

每次 MySQL 的新发布都会添加新功能。如果您正在编写需要特定功能的应用程序，请检查服务器版本以确定其是否存在；如果不存在，则必须执行某种解决方案（假设存在）。

要获取服务器版本，请调用 `VERSION()` 函数。结果是一个字符串，看起来类似于 `5.7.33-debug-log` 或 `8.0.25`。换句话说，它返回一个由主要、次要和<q>修订</q>版本号组成的字符串，可能在<q>修订</q>版本的末尾有一些非数字字符，可能还有一些后缀。版本字符串可以直接用于演示目的，但是为了比较，用数字更简单——特别是使用 *`Mmmtt`* 格式的五位数，在该格式中，*`M`*、*`mm`*、*`tt`* 分别代表主要、次要和修订版本号。通过在句点处拆分字符串、去除第三部分中以第一个非数字字符开始的后缀，并连接这些部分来执行转换。例如，`5.7.33-debug-log` 变成 `50733`，而 `8.0.25` 变成 `80025`。

这是一个 Perl DBI 函数，它接受一个数据库句柄参数，并返回一个包含服务器版本的字符串和数字形式的两元素列表。该代码假设次要版本和修订版本部分都小于 100，因此每部分最多两位数。这应该是一个有效的假设，因为 MySQL 本身的源代码也使用相同的格式：

```
sub get_server_version
{
my $dbh = shift;
my ($ver_str, $ver_num);
my ($major, $minor, $patch);

  # fetch result into scalar string
  $ver_str = $dbh->selectrow_array ("SELECT VERSION()");
  return undef unless defined ($ver_str);
  ($major, $minor, $patch) = split (/\./, $ver_str);
  $patch =~ s/\D.*$//; # strip nonnumeric suffix if present
  $ver_num = $major*10000 + $minor*100 + $patch;
  return ($ver_str, $ver_num);
}
```

要一次获取两种形式的版本信息，请这样调用函数：

```
my ($ver_str, $ver_num) = get_server_version ($dbh);
```

要仅获取其中一个值，请按以下方式调用：

```
my $ver_str = (get_server_version ($dbh))[0]; # string form
my $ver_num = (get_server_version ($dbh))[1]; # numeric form
```

以下示例演示了如何使用数字版本值检查服务器是否支持某些功能：

```
my $ver_num = (get_server_version ($dbh))[1];
printf "Event scheduler:    %s\n", ($ver_num >= 50106 ? "yes" : "no");
printf "4-byte Unicode:     %s\n", ($ver_num >= 50503 ? "yes" : "no");
printf "Fractional seconds: %s\n", ($ver_num >= 50604 ? "yes" : "no");
printf "SHA-256 passwords:  %s\n", ($ver_num >= 50606 ? "yes" : "no");
printf "ALTER USER:         %s\n", ($ver_num >= 50607 ? "yes" : "no");
printf "INSERT DELAYED:     %s\n", ($ver_num >= 50700 ? "no" : "yes");
```

`recipes` 发行版 *metadata* 目录中包含其他 API 语言中的 `get_server_version()` 实现，并且 *routines* 目录包含用于 SQL 语句的 `server_version()` 存储函数。后者仅返回数字值，因为 `VERSION()` 已经生成了字符串值。以下示例显示了如何使用它来实现一个存储过程，如果服务器支持 `ALTER USER ... FAILED_LOGIN_ATTEMPTS` 语句（MySQL 8.0.19 或更高版本），则对 *`N`* 次失败登录尝试进行密码锁定：

```
CREATE PROCEDURE enable_failed_login_attempts(
                                  user TEXT, host TEXT, failed_atempts INT)
BEGIN
  DECLARE account TEXT;
  SET account = CONCAT(QUOTE(user),'@',QUOTE(host));
  IF server_version() >= 80019 AND user <> '' THEN
    CALL exec_stmt(CONCAT('ALTER USER ',account,' 
 FAILED_LOGIN_ATTEMPTS ', failed_atempts));
  END IF;
END;
```

`expire_password()` 需要 `exec_stmt()` 辅助例程（参见 Recipe 11.6）。这两者都在 *routines* 目录中可用。关于密码过期的更多信息，请参见 Recipe 24.5。

# 12.9 获取通过外键约束引用特定表的子表

## 问题

您想知道哪些其他表通过外键约束引用您的表作为父表。

## 解决方案

查询表 `INFORMATION_SCHEMA.TABLE_CONSTRAINTS` 和 `INFORMATION_SCHEMA.KEY_COLUMN_USAGE`。

## 讨论

外键约束提供数据完整性检查，正如我们在 “使用外键强制实现参照完整性和防止不匹配” 中讨论的那样。它们通过阻止修改由链接表引用的数据的语句来执行，以防止语句的结果破坏完整性。外键帮助保持数据正确，但同时可能引发难以排查的 SQL 错误。虽然很容易找出特定子表的父表是哪个，但要找出特定父表的子表却并不容易。但如果你计划修改它，则知道表是否被子表引用将会很有帮助。

表 `INFORMATION_SCHEMA.TABLE_CONSTRAINTS` 包含为您的 MySQL 安装创建的所有约束。要选择外键约束，请使用子句 `WHERE CONSTRAINT_TYPE='FOREIGN KEY'` 来缩小搜索范围：

```
mysql> `SELECT` `TABLE_SCHEMA``,` `TABLE_NAME``,` `CONSTRAINT_NAME` 
    -> `FROM` `INFORMATION_SCHEMA``.``TABLE_CONSTRAINTS` 
    -> `WHERE` `CONSTRAINT_TYPE``=``'FOREIGN KEY'` `AND` `TABLE_SCHEMA``=``'cookbook'``;`
+--------------+--------------------+---------------------------+ | TABLE_SCHEMA | TABLE_NAME         | CONSTRAINT_NAME           |
+--------------+--------------------+---------------------------+ | cookbook     | movies_actors_link | movies_actors_link_ibfk_1 |
| cookbook     | movies_actors_link | movies_actors_link_ibfk_2 |
+--------------+--------------------+---------------------------+ 2 rows in set (0,00 sec)
```

上述列表打印了我们在 Recipe 11.10 示例中创建的外键。但是，此输出仍然只列出了子表。要找出哪个表是父表，我们需要将 `INFORMATION_SCHEMA.TABLE_CONSTRAINTS` 与表 `INFORMATION_SCHEMA.KEY_COLUMN_USAGE` 进行连接：

```
mysql> `SELECT`  `ku``.``CONSTRAINT_NAME``,` `ku``.``TABLE_NAME``,` `ku``.``COLUMN_NAME``,` 
    -> `ku``.``REFERENCED_TABLE_NAME``,` `ku``.``REFERENCED_COLUMN_NAME` 
    -> `FROM` `INFORMATION_SCHEMA``.``TABLE_CONSTRAINTS` `tc` 
    -> `JOIN` `INFORMATION_SCHEMA``.``KEY_COLUMN_USAGE` `ku` 
    -> `USING` `(``CONSTRAINT_NAME``,` `TABLE_SCHEMA``,` `TABLE_NAME``)` 
    -> `WHERE` `CONSTRAINT_TYPE``=``'FOREIGN KEY'` `AND` `ku``.``TABLE_SCHEMA``=``'cookbook'``\``G`
*************************** 1. row ***************************
       CONSTRAINT_NAME: movies_actors_link_ibfk_1
            TABLE_NAME: movies_actors_link
           COLUMN_NAME: movie_id
 REFERENCED_TABLE_NAME: movies
REFERENCED_COLUMN_NAME: id
*************************** 2. row ***************************
       CONSTRAINT_NAME: movies_actors_link_ibfk_2
            TABLE_NAME: movies_actors_link
           COLUMN_NAME: actor_id
 REFERENCED_TABLE_NAME: actors
REFERENCED_COLUMN_NAME: id
2 rows in set (0,00 sec)
```

在上述列表中，列 `TABLE_NAME` 和 `COLUMN_NAME` 指代子表，列 `REFERENCED_TABLE_NAME` 和 `REFERENCED_COLUMN_NAME` 指代父表。

对于 InnoDB 表，您还可以查询表 `INNODB_FOREIGN` 和 `INNODB_FOREIGN_COLS`：

```
mysql> `SELECT` `ID``,` `FOR_NAME``,` `FOR_COL_NAME``,` `REF_NAME``,` `REF_COL_NAME` 
    -> `FROM` `INFORMATION_SCHEMA``.``INNODB_FOREIGN` `JOIN` 
    -> `INFORMATION_SCHEMA``.``INNODB_FOREIGN_COLS` `USING``(``ID``)` 
    -> `WHERE` `ID` `LIKE` `'cookbook%'``\``G`
*************************** 1. row ***************************
          ID: cookbook/movies_actors_link_ibfk_1
    FOR_NAME: cookbook/movies_actors_link
FOR_COL_NAME: movie_id
    REF_NAME: cookbook/movies
REF_COL_NAME: id
*************************** 2. row ***************************
          ID: cookbook/movies_actors_link_ibfk_2
    FOR_NAME: cookbook/movies_actors_link
FOR_COL_NAME: actor_id
    REF_NAME: cookbook/actors
REF_COL_NAME: id
2 rows in set (0,01 sec)
```

请注意，这些表从内部 InnoDB 数据字典中获取数据，该字典将数据库和表名存储在一个字段中。因此，您需要使用操作符 `LIKE` 来限制结果到特定的数据库或表。

# 12.10 列出触发器

## 问题

您希望列出为您的表定义的触发器。

## 解决方案

查询表 `INFORMATION_SCHEMA.TRIGGERS`。

## 讨论

在调优性能时，了解特定表定义的触发器非常有用，特别是在以下情况下：

+   简单的更新，影响了几行，运行时间比您预期的长。

+   未参与应用负载并且在进程列表中不可见的表，等待或持有锁。

+   磁盘 IO 很高。

例如，要列出为表 `auction` 创建的触发器，请使用以下查询：

```
mysql> `SELECT` `EVENT_MANIPULATION``,` `ACTION_TIMING``,` `TRIGGER_NAME``,` `ACTION_STATEMENT` 
    -> `FROM` `INFORMATION_SCHEMA``.``TRIGGERS`
    -> `WHERE` `TRIGGER_SCHEMA``=``'cookbook'` `AND` `EVENT_OBJECT_TABLE` `=` `'auction'``\``G`
*************************** 1. row ***************************
EVENT_MANIPULATION: INSERT
     ACTION_TIMING: AFTER
      TRIGGER_NAME: ai_auction
  ACTION_STATEMENT: INSERT INTO auction_log (action,id,ts,item,bid)
VALUES('create',NEW.id,NOW(),NEW.item,NEW.bid)
*************************** 2. row ***************************
EVENT_MANIPULATION: UPDATE
     ACTION_TIMING: AFTER
      TRIGGER_NAME: au_auction
  ACTION_STATEMENT: INSERT INTO auction_log (action,id,ts,item,bid)
VALUES('update',NEW.id,NOW(),NEW.item,NEW.bid)
*************************** 3. row ***************************
EVENT_MANIPULATION: DELETE
     ACTION_TIMING: AFTER
      TRIGGER_NAME: ad_auction
  ACTION_STATEMENT: INSERT INTO auction_log (action,id,ts,item,bid)
VALUES('delete',OLD.id,OLD.ts,OLD.item,OLD.bid)
3 rows in set (0,01 sec)
```

这样，您可以获取触发器何时被触发以及其体定义的信息。如果有多个触发器，您将看到它们所有的信息。

# 12.11 列出存储过程和计划事件

## 问题

您想知道在您的数据库中创建了哪些存储过程、函数和计划事件。

## 解决方案

查询表 `INFORMATION_SCHEMA.ROUTINES` 和 `INFORMATION_SCHEMA.EVENTS`。

## 讨论

要列出存储函数和存储过程，请查询表 `INFORMATION_SCHEMA.ROUTINES`。如果要区分是哪种类型的例程，可以通过 `WHERE` 条件指定 `ROUTINE_TYPE` 为 `FUNCTION` 或 `PROCEDURE`。

例如，在我们讨论 Recipe 15.17 中的序列生成中，列出所有参与的例程，请使用以下代码：

```
mysql> `SELECT` `ROUTINE_NAME``,` `ROUTINE_TYPE` `FROM` `INFORMATION_SCHEMA``.``ROUTINES`
    -> `WHERE` `ROUTINE_SCHEMA``=``'cookbook'` `AND` `ROUTINE_NAME` `LIKE` `'%sequence%'``;`
+---------------------+--------------+ | ROUTINE_NAME        | ROUTINE_TYPE |
+---------------------+--------------+ | sequence_next_value | FUNCTION     |
| create_sequence     | PROCEDURE    |
| delete_sequence     | PROCEDURE    |
+---------------------+--------------+ 3 rows in set (0,01 sec)
```

您还可以选择列 `ROUTINE_DEFINITION` 以获取例程体。

要获取计划事件列表，请查询表 `INFORMATION_SCHEMA.EVENTS`：

```
mysql> `SELECT` `EVENT_NAME``,` `EVENT_TYPE``,` `INTERVAL_VALUE``,` `INTERVAL_FIELD``,` `LAST_EXECUTED``,`
    -> `STATUS``,` `ON_COMPLETION``,` `EVENT_DEFINITION` `FROM` `INFORMATION_SCHEMA``.``EVENTS``\``G`
*************************** 1. row ***************************
      EVENT_NAME: mark_insert
      EVENT_TYPE: RECURRING
  INTERVAL_VALUE: 5
  INTERVAL_FIELD: MINUTE
   LAST_EXECUTED: 2021-07-07 05:10:45
          STATUS: ENABLED
   ON_COMPLETION: NOT PRESERVE
EVENT_DEFINITION: INSERT INTO mark_log (message) VALUES('-- MARK --')
*************************** 2. row ***************************
      EVENT_NAME: mark_expire
      EVENT_TYPE: RECURRING
  INTERVAL_VALUE: 1
  INTERVAL_FIELD: DAY
   LAST_EXECUTED: 2021-07-07 02:56:14
          STATUS: ENABLED
   ON_COMPLETION: NOT PRESERVE
EVENT_DEFINITION: DELETE FROM mark_log WHERE ts < NOW() - INTERVAL 2 DAY
2 rows in set (0,00 sec)
```

此表不仅保存事件定义，还保存了诸如上次执行时间、计划间隔及其是否启用或禁用等元数据。

# 12.12 列出安装的插件

## 问题

您想知道为您的 MySQL 服务器安装了哪些插件。

## 解决方案

查询表 `INFORMATION_SCHEMA.PLUGINS`。

## 讨论

MySQL 是高度模块化的系统。它的许多部分都是可插拔的。例如，所有存储引擎也是插件。因此，了解服务器上可用的插件非常重要。要获取有关已安装插件的信息，请查询表 `INFORMATION_SCHEMA.PLUGINS` 或运行命令 `SHOW PLUGINS`。虽然后者适用于交互式使用，但前者提供了更多信息。

```
mysql> `SELECT` `*` `FROM` `INFORMATION_SCHEMA``.``PLUGINS` 
    -> `WHERE` `PLUGIN_NAME` `IN` `(``'caching_sha2_password'``,` `'InnoDB'``,` `'Rewriter'``)``\``G`
*************************** 1. row ***************************
           PLUGIN_NAME: caching_sha2_password
        PLUGIN_VERSION: 1.0
         PLUGIN_STATUS: ACTIVE
           PLUGIN_TYPE: AUTHENTICATION
   PLUGIN_TYPE_VERSION: 2.0
        PLUGIN_LIBRARY: NULL
PLUGIN_LIBRARY_VERSION: NULL
         PLUGIN_AUTHOR: Oracle Corporation
    PLUGIN_DESCRIPTION: Caching sha2 authentication
        PLUGIN_LICENSE: GPL
           LOAD_OPTION: FORCE
*************************** 2. row ***************************
           PLUGIN_NAME: InnoDB
        PLUGIN_VERSION: 8.0
         PLUGIN_STATUS: ACTIVE
           PLUGIN_TYPE: STORAGE ENGINE
   PLUGIN_TYPE_VERSION: 80025.0
        PLUGIN_LIBRARY: NULL
PLUGIN_LIBRARY_VERSION: NULL
         PLUGIN_AUTHOR: Oracle Corporation
    PLUGIN_DESCRIPTION: Supports transactions, row-level locking, and foreign keys
        PLUGIN_LICENSE: GPL
           LOAD_OPTION: FORCE
*************************** 3. row ***************************
           PLUGIN_NAME: Rewriter
        PLUGIN_VERSION: 0.2
         PLUGIN_STATUS: ACTIVE
           PLUGIN_TYPE: AUDIT
   PLUGIN_TYPE_VERSION: 4.1
        PLUGIN_LIBRARY: rewriter.so
PLUGIN_LIBRARY_VERSION: 1.10
         PLUGIN_AUTHOR: Oracle Corporation
    PLUGIN_DESCRIPTION: A query rewrite plugin that rewrites queries using the parse tree.
        PLUGIN_LICENSE: GPL
           LOAD_OPTION: ON
3 rows in set (0,01 sec)
```

对于存储引擎，如果查询表 `INFORMATION_SCHEMA.ENGINES` 或运行命令 `SHOW ENGINES`，可以获取更多详细信息。以下是 InnoDB 存储引擎的表内容：

```
mysql> `SELECT` `*` `FROM` `INFORMATION_SCHEMA``.``ENGINES` `WHERE` `ENGINE` `=` `'InnoDB'``\``G`
*************************** 1. row ***************************
      ENGINE: InnoDB
     SUPPORT: DEFAULT
     COMMENT: Supports transactions, row-level locking, and foreign keys
TRANSACTIONS: YES
          XA: YES
  SAVEPOINTS: YES
1 row in set (0,00 sec)
```

# 12.13 列出字符集和排序规则

## 问题

排序顺序定义了哪些字母是相等的，但对您不起作用，您想了解您还有哪些其他选项。

## 解决方案

通过查询表`INFORMATION_SCHEMA.CHARACTER_SETS`和`INFORMATION_SCHEMA.COLLATIONS`，获取字符集的列表，它们的默认排序和可用排序。

## 讨论

在 Recipe 7.5 中，我们讨论了如何更改或设置字符串的字符集和排序。但是如何选择最适合您应用程序要求的排序呢？

幸运的是，MySQL 本身可以帮助您找到答案。在 MySQL 客户端内部，从`INFORMATION_SCHEMA.CHARACTER_SETS`表中选择以获取所有可用字符集、它们的默认排序和它们可以存储的最大字符长度的列表。例如，要列出所有 Unicode 字符集，请运行以下查询：

```
mysql> `SELECT` `*` `FROM` `INFORMATION_SCHEMA``.``CHARACTER_SETS` 
    -> `WHERE` `DESCRIPTION` `LIKE` `'%Unicode%'` `ORDER` `BY` `MAXLEN` `DESC``;`
+--------------------+----------------------+------------------+--------+ | CHARACTER_SET_NAME | DEFAULT_COLLATE_NAME | DESCRIPTION      | MAXLEN |
+--------------------+----------------------+------------------+--------+ | utf16              | utf16_general_ci     | UTF-16 Unicode   |      4 |
| utf16le            | utf16le_general_ci   | UTF-16LE Unicode |      4 |
| utf32              | utf32_general_ci     | UTF-32 Unicode   |      4 |
| utf8mb4            | utf8mb4_0900_ai_ci   | UTF-8 Unicode    |      4 |
| utf8               | utf8_general_ci      | UTF-8 Unicode    |      3 |
| ucs2               | ucs2_general_ci      | UCS-2 Unicode    |      2 |
+--------------------+----------------------+------------------+--------+ 6 rows in set (0,00 sec)
```

每个字符集不仅可能有默认排序，还可能有其他排序，允许您调整排序顺序。例如，土耳其大写字母<I>İ</I>和<I>İ</I>，以及<S>Ş</S>被认为是与默认排序的<q>utf8mb4</q>字符集相等。这导致 MySQL 认为土耳其单词<I>ISSIZ</I>（荒凉的）和<I>İŞSİZ</I>（失业的）是相同的：

```
mysql> `CREATE` `TABLE` `two_words``(``deserted` `VARCHAR``(``100``)``,` `unemployed` `VARCHAR``(``100``)``)``;`
Query OK, 0 rows affected (0,03 sec)

mysql> `INSERT` `INTO` `two_words` `VALUES``(``'ISSIZ'``,` `'İŞSİZ'``)``;`
Query OK, 1 row affected (0,00 sec)

mysql> `SELECT` `deserted``=``unemployed` `FROM` `two_words``;`
+---------------------+ | deserted=unemployed |
+---------------------+ |                   1 |
+---------------------+ 1 row in set (0,00 sec)
```

要解决此问题，请检查表`INFORMATION_SCHEMA.COLLATIONS`中`utf8mb4`字符集适用于土耳其语的排序。

```
mysql> `SELECT` `COLLATION_NAME``,` `CHARACTER_SET_NAME`
    -> `FROM` `INFORMATION_SCHEMA``.``COLLATIONS` 
    -> `WHERE` `CHARACTER_SET_NAME``=``'utf8mb4'` `AND` `COLLATION_NAME` `LIKE` `'%\_tr\_%'``;`
+-----------------------+--------------------+ | COLLATION_NAME        | CHARACTER_SET_NAME |
+-----------------------+--------------------+ | utf8mb4_tr_0900_ai_ci | utf8mb4            |
| utf8mb4_tr_0900_as_cs | utf8mb4            |
+-----------------------+--------------------+ 2 rows in set (0,00 sec)
```

如果我们尝试它们，我们将收到正确的结果：单词<I>deserted</I>和<I>unemployed</I>不再被认为是相同的：

```
mysql> `SELECT` `deserted``=``unemployed` `COLLATE` `utf8mb4_tr_0900_ai_ci` `FROM` `two_words``;`
+---------------------------------------------------+ | deserted=unemployed COLLATE utf8mb4_tr_0900_ai_ci |
+---------------------------------------------------+ |                                                 0 |
+---------------------------------------------------+ 1 row in set (0,00 sec)

mysql> `SELECT` `deserted``=``unemployed` `COLLATE` `utf8mb4_tr_0900_as_cs` `FROM` `two_words``;`
+---------------------------------------------------+ | deserted=unemployed COLLATE utf8mb4_tr_0900_as_cs |
+---------------------------------------------------+ |                                                 0 |
+---------------------------------------------------+ 1 row in set (0,00 sec)
```

字符集`utf8mb4`是默认的，并且适用于大多数设置。但是，如果您存储俄语单词<I>совершенный</I>（完美的）和<I>совершённый</I>（完成的）在具有默认排序的`utf8mb4`列中，MySQL 将认为这两个单词相等：

```
mysql > `CREATE` `TABLE` `` ` ```two_words``` ` `` `(`
    ->   `` ` ```perfect``` ` `` `varchar``(``100``)` `DEFAULT` `NULL``,`
    ->   `` ` ```accomplished``` ` `` `varchar``(``100``)` `DEFAULT` `NULL`
    -> `)` `ENGINE``=``InnoDB` `DEFAULT` `CHARSET``=``utf8mb4` `COLLATE``=``utf8mb4_0900_ai_ci``;`
Query OK, 0 rows affected (0,04 sec)

mysql> `INSERT` `INTO` `two_words` `VALUES``(``'совершенный'``,` `'совершённый'``)``;`
Query OK, 1 row affected (0,01 sec)

mysql> `SELECT` `perfect` `=` `accomplished` `FROM` `two_words``;`
+------------------------+ | perfect = accomplished |
+------------------------+ |                      1 |
+------------------------+ 1 row in set (0,00 sec)
```

解决此问题的直观方法是使用俄语语言的可用排序：`utf8mb4_ru_0900_ai_ci`。不幸的是，这并不起作用：

```
mysql> `SELECT` `perfect` `=` `accomplished` `COLLATE` `utf8mb4_ru_0900_ai_ci` `FROM` `two_words``;`
+------------------------------------------------------+ | perfect = accomplished COLLATE utf8mb4_ru_0900_ai_ci |
+------------------------------------------------------+ |                                                    1 |
+------------------------------------------------------+ 1 row in set (0,00 sec)
```

`utf8mb4_ru_0900_ai_ci`排序是不区分重音的原因。其区分大小写和重音的变体`utf8mb4_ru_0900_as_cs`解决了这个问题：

```
mysql> `SELECT` `perfect` `=` `accomplished` `COLLATE` `utf8mb4_ru_0900_as_cs` `FROM` `two_words``;`
+------------------------------------------------------+ | perfect = accomplished COLLATE utf8mb4_ru_0900_as_cs |
+------------------------------------------------------+ |                                                    0 |
+------------------------------------------------------+ 1 row in set (0,00 sec)
```

版本 8.0 中添加了排序`utf8mb4_ru_0900_ai_ci`和`utf8mb4_ru_0900_as_cs`。如果您仍在使用版本 5.7，并且正在处理此类差异至关重要的应用程序，则还可以检查表`INFORMATION_SCHEMA.CHARACTER_SETS`，查找支持西里尔字母的字符集，并尝试使用它：

```
mysql> `SELECT` `*` `FROM` `INFORMATION_SCHEMA``.``CHARACTER_SETS` 
    -> `WHERE` `DESCRIPTION` `LIKE` `'%Russian%'` `OR` `DESCRIPTION` `LIKE` `'%Cyrillic%'``;`
+--------------------+----------------------+-----------------------+--------+ | CHARACTER_SET_NAME | DEFAULT_COLLATE_NAME | DESCRIPTION           | MAXLEN |
+--------------------+----------------------+-----------------------+--------+ | koi8r              | koi8r_general_ci     | KOI8-R Relcom Russian |      1 |
| cp866              | cp866_general_ci     | DOS Russian           |      1 |
| cp1251             | cp1251_general_ci    | Windows Cyrillic      |      1 |
+--------------------+----------------------+-----------------------+--------+ 3 rows in set (0,00 sec)

mysql> `drop` `table` `two_words``;`
Query OK, 0 rows affected (0,02 sec)

mysql> `CREATE` `TABLE` `two_words``(``perfect` `VARCHAR``(``100``)``,` `accomplished` `VARCHAR``(``100``)``)` 
    -> `CHARACTER` `SET` `cp1251``;`
Query OK, 0 rows affected (0,04 sec)

mysql> `INSERT` `INTO` `two_words` `VALUES``(``'совершенный'``,` `'совершённый'``)``;`
Query OK, 1 row affected (0,00 sec)

mysql> `SELECT` `perfect` `=` `accomplished` `FROM` `two_words``;`
+------------------------+ | perfect = accomplished |
+------------------------+ |                      0 |
+------------------------+ 1 row in set (0,00 sec)
```

我们选择了字符集`cp1251`作为示例，但所有这些都解决了这个比较问题。

# 12.14 列出 CHECK 约束

## 问题

您想要查看数据库中定义了哪些`CHECK`约束。

## 解决方案

查询表`INFORMATION_SCHEMA.CHECK_CONSTRAINTS`和`INFORMATION_SCHEMA.TABLE_CONSTRAINTS`。

## 讨论

表 `INFORMATION_SCHEMA.CHECK_CONSTRAINTS` 包含所有约束条件的列表，以及定义它们的模式，以及实际约束定义的 `CHECK_CLAUSE`。然而，该表不存储约束是为哪个表创建的信息。要列出既包含约束条件又包含定义它们的表，请将表 `INFORMATION_SCHEMA.CHECK_CONSTRAINTS` 与表 `INFORMATION_SCHEMA.TABLE_CONSTRAINTS` 进行连接：

```
mysql> `SELECT` `TABLE_SCHEMA``,` `TABLE_NAME``,` `CONSTRAINT_NAME``,` `ENFORCED``,` `CHECK_CLAUSE` 
    -> `FROM` `INFORMATION_SCHEMA``.``CHECK_CONSTRAINTS` 
    -> `JOIN` `INFORMATION_SCHEMA``.``TABLE_CONSTRAINTS` 
    -> `USING``(``CONSTRAINT_NAME``)` 
    -> `WHERE` `CONSTRAINT_TYPE``=``'CHECK'` `ORDER` `BY` `CONSTRAINT_NAME` `DESC` `LIMIT` `2``\``G`
*************************** 1. row ***************************
   TABLE_SCHEMA: cookbook
     TABLE_NAME: even
CONSTRAINT_NAME: even_chk_1
       ENFORCED: YES
   CHECK_CLAUSE: ((`even_value` % 2) = 0)
*************************** 2. row ***************************
   TABLE_SCHEMA: cookbook
     TABLE_NAME: book_authors
CONSTRAINT_NAME: book_authors_chk_1
       ENFORCED: YES
   CHECK_CLAUSE: json_schema_valid(_utf8mb4\'{"id": ↩ 
                "http://www.oreilly.com/mysqlcookbook", "$schema": ↩
                "http://json-schema.org/draft-04/schema#", "description": ↩
                "Schema for the table book_authors", "type": "object", "properties": ↩
                {"name": {"type": "string"}, "lastname": {"type": "string"}, ↩
                "books": {"type": "array"}}, "required":["name", "lastname"]} \',`author`)
2 rows in set (0,01 sec)
```
