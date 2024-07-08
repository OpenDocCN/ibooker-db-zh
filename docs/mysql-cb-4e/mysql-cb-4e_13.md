# 第十三章： 导入和导出数据

# 13.0 引言

假设名为*somedata.csv*的文件包含以逗号分隔的值（CSV）格式的 12 个数据列。 您希望从该文件中提取仅包含第 2、11、5 和 9 列，并将它们用于在包含`name`、`birth`、`height`和`weight`列的 MySQL 表中创建数据库行。 您必须确保身高和体重为正整数，并将出生日期从*`MM/DD/YY`*格式转换为*`YYYY-MM-DD`*格式。 您可以如何做到这一点？

在将数据传输到 MySQL 时经常会出现具有特定要求的数据传输问题。 数据文件并非总是经过格式化以便无需准备即可加载到 MySQL 中。 因此，通常需要预处理信息以使其符合 MySQL 可接受的格式。 反之亦然； 从 MySQL 导出的数据可能需要调整才能对其他程序有用。

虽然有些数据准备操作需要大量手动检查和重新格式化，但在大多数情况下，您至少可以自动完成部分工作。 几乎所有这些问题都涉及到一些公共转换问题的元素集。 本章和下一章讨论了这些问题是什么，如何通过利用您可以使用的现有工具来处理它们，以及在必要时如何编写您自己的工具。 思路不是覆盖所有可能的情况（这是不可能的任务），而是展示代表性的技术和实用工具。 您可以按原样使用它们或进行调整。（虽然有商业数据处理工具，但我们这里的目的是让您自己动手做。） 关于本引言开头提出的问题，请参阅食谱 14.18 以了解我们找到的解决方案。

讨论如何将数据传输到 MySQL 和从 MySQL 中传输数据始于本地 MySQL 用于导入数据的工具（`LOAD` `DATA`语句和*mysqlimport*命令行程序），以及用于导出数据的工具（`SELECT` … `INTO` `OUTFILE`语句）。 对于本地工具不足的情况，我们继续介绍使用外部支持工具（如*sed*和*tr*）以及编写自己的工具的技术。 有两组广泛的问题需要考虑：

+   如何操作数据文件的*结构*。 当文件格式不适合导入时，必须将其转换为不同的格式。 这可能涉及问题，如更改列分隔符或行结束序列，或删除或重新排列文件中的列。 本章介绍了这些技术。

+   如何操作数据文件的*内容*。 如果你不确定文件中的值是否合法，可能需要预处理以进行检查或重新格式化。 数值可能需要验证是否在特定范围内，日期可能需要转换为或从 ISO 格式进行转换，等等。 第十四章 讨论了这些技术。

本章讨论的程序片段和脚本的源代码位于`recipes`发行版的*transfer*目录中。

## 一般的导入和导出问题

不兼容的数据文件格式和不同的解释各种值的规则，在程序之间传输数据时会带来很多麻烦。尽管如此，某些问题经常重复出现。了解这些问题可以更轻松地确定解决特定导入或导出问题所需的步骤。

在其最基本的形式中，输入流只是一组没有特定含义的字节。成功导入到 MySQL 需要识别哪些字节代表结构信息，哪些字节代表由该结构框定的数据值。因为这种识别对将输入分解成适当单位至关重要，所以最基本的导入问题如下：

+   记录分隔符是什么？知道这个可以将输入流分成记录。

+   字段分隔符是什么？知道这个可以将每个记录分成字段值。识别数据值还可能包括去掉值周围的引号或识别其中的转义序列。

将输入分解为记录和字段的能力对从中提取数据值至关重要。如果值仍未以可直接使用的形式存在，则可能需要考虑其他问题：

+   列的顺序和数量是否与数据库表的结构匹配？不匹配可能需要重新排列或跳过列。

+   `NULL`或空值应该如何处理？它们允许存在吗？甚至可以检测到`NULL`值吗？（某些系统将`NULL`值导出为空字符串，这使得无法区分它们。）

+   数据值是否需要验证或重新格式化？如果值的格式与 MySQL 的期望匹配，则无需进一步处理。否则，它们必须进行检查并可能需要重写。

对于从 MySQL 导出的情况，问题有些相反。您可以假设存储在数据库表中的值是有效的，但是需要添加列和记录分隔符，以形成其他程序可以识别的输出流，并且可能需要重新格式化以供其他程序使用。

## 文件格式

数据文件存在多种格式，其中两种在本章中频繁出现：

制表符分隔或制表符分隔值（TSV）格式

这是最简单的文件结构之一；行中包含由制表符分隔的值。一个简短的制表符分隔文件可能如下所示，其中列值之间的空格表示单个制表符字符：

```
a       b       c
a,b,c   d e     f
```

逗号分隔值（CSV）格式

以 CSV 格式编写的文件有所不同；似乎没有正式的标准描述这种格式。然而，一般的想法是，每行由逗号分隔的值组成，包含内部逗号的值用引号括起来，以防逗号被解释为值分隔符。同样，包含空格的值通常也会用引号括起来。在这个例子中，每行包含三个值：

```
a,b,c
"a,b,c","d e",f
```

处理 CSV 文件比制表符分隔文件更复杂，因为像引号和逗号这样的字符具有双重含义：它们可能代表文件结构，也可能包含在数据值的内容中。

另一个重要的数据文件特征是行尾序列。最常见的序列是回车（CR）、换行（LF）和回车/换行（CRLF）对。

数据文件通常以一行列标签开头。对于某些导入操作，必须丢弃标签行，以避免将其作为数据加载到您的表中。在其他情况下，标签非常有用：

+   对于导入到现有表中，如果数据文件列与表列的顺序不一致，标签有助于匹配它们。

+   当从数据文件自动或半自动创建新表时，这些标签可用作列名。例如，Recipe 13.20 讨论了一个工具，该工具检查数据文件并猜测用于从文件创建表的 `CREATE` `TABLE` 语句。如果存在标签行，则该工具使用这些标签作为列名。

## 关于调用 Shell 命令的注意事项

本章展示了许多程序，您可以通过像 *bash* 或 *tcsh*（Unix）或 *cmd.exe*（<q>命令提示符</q>）（Windows）这样的 shell 从命令行调用。这些程序的示例命令中，很多选项值周围使用引号，有时选项值本身就是引号字符。引用约定因 shell 而异，但以下规则似乎适用于大多数 shell（包括 Windows 下的 *cmd.exe*）：

+   对于包含空格的参数，用双引号括起来，以防 shell 将其解释为多个独立参数。Shell 去除引号并将参数完整传递给命令。

+   要在参数本身包含双引号字符，需在其前面加上反斜杠。

本章中的一些 shell 命令非常长，显示方式按照您输入它们时的多行格式，使用反斜杠字符作为行连续字符：

```
% *`prog_name`* \
    *`argument1`* \
    *`argument2 ...`*
```

这在 Unix 下有效。在 Windows 上，连续字符是 `^`（或者对于 PowerShell 是 `` ` ``）。或者，无论在任何平台上，都可以将整个命令输入到一行中：

```
C:\> *`prog_name argument1 argument2 ...`*
```

# 13.1 使用 LOAD DATA 和 mysqlimport 导入数据

## 问题

您想使用 MySQL 的内置导入功能将数据文件加载到表中。

## 解决方案

使用`LOAD` `DATA`语句或者 *mysqlimport* 命令行程序。

## 讨论

MySQL 提供了一个`LOAD` `DATA`语句，作为批量数据加载器。以下是一个示例语句，从当前目录（调用*mysql*客户端的目录）读取文件*mytbl.txt*并将其加载到默认数据库中的表`mytbl`中：

```
mysql> `LOAD DATA LOCAL INFILE 'mytbl.txt' INTO TABLE mytbl;`
```

###### 警告

自 MySQL 8.0 起，默认情况下禁用了`LOCAL`加载功能，出于安全考虑。

要在测试服务器上启用它，请将变量`local_infile`设置为`ON`：

```
SET GLOBAL local_infile = 1;
```

并使用选项`--local-infile`启动*mysql*客户端：

```
mysql -ucbuser -p --local-infile
```

或者，从语句中省略`LOCAL`并指定文件的完整路径名，该文件必须可读取服务器。稍后将讨论本地与非本地数据加载的差异。

MySQL 实用程序*mysqlimport*充当了围绕`LOAD` `DATA`的包装器的角色，以便您可以直接从命令行加载输入文件。与前述`LOAD` `DATA`语句等效的*mysqlimport*命令如下，假设`mytbl`在`cookbook`数据库中：

```
$ `mysqlimport --local cookbook mytbl.txt`
```

对于*mysqlimport*，与其他 MySQL 程序一样，您可能需要指定连接参数选项，如`--user`或`--host`（参见 Recipe 1.4）。

`LOAD` `DATA` 提供了许多选项来解决章节介绍中提到的许多导入问题，例如用于识别如何将输入分割为记录的行结束序列，允许将记录分割为单独值的列值分隔符，可能围绕列值的引用字符，值内的引用和转义约定以及`NULL`值表示。

以下列表描述了`LOAD` `DATA`的一般特性和功能；*mysqlimport*共享大多数这些行为。我们会在进行时注意一些差异，但对于大多数情况，`LOAD` `DATA`所能做的与*mysqlimport*也可以做到。

+   默认情况下，`LOAD` `DATA` 期望数据文件与加载数据的表具有相同数量的列，并且列的顺序与表中的列相同。如果文件的列数或顺序与表不同，您可以指定存在的列及其顺序。如果数据文件包含的列少于表中的列数，MySQL 会为缺少的列分配默认值。

+   `LOAD` `DATA`假定数据值由制表符分隔，并且行以换行符（新行）结束。如果文件不符合这些约定，您可以明确指定其格式。

+   您可以指示数据值可能带有应该被剥离的引号，并且可以指定引号字符。

+   输入处理期间识别并转换多个特殊转义序列。默认的转义字符是反斜杠（`\`），但您可以更改它。序列 `\N` 被解释为 `NULL` 值。序列 `\b`、`\n`、`\r`、`\t`、`\\` 和 `\0` 被解释为退格、换行、回车、制表符、反斜杠和 ASCII NUL 字符。（NUL 是一个零值字节；它与 SQL 中的 `NULL` 值不同。）

+   `LOAD` `DATA` 提供关于哪些输入值导致问题的诊断信息。要显示此信息，请在 `LOAD` `DATA` 语句之后执行 `SHOW` `WARNINGS` 语句。

本章及其后几篇配方描述了如何使用 `LOAD` `DATA` 或 *mysqlimport* 处理这些问题。由于需要涵盖的内容很多，因此篇幅较长。

### 指定数据文件的位置

您可以从发出 `LOAD` `DATA` 语句的服务器主机或客户端主机上加载文件。告知 MySQL 如何找到您的数据文件是了解它查找文件的规则的问题（对于不在当前目录中的文件特别重要）。

默认情况下，MySQL 服务器假定数据文件位于服务器主机上。您可以使用 `LOAD` `DATA` `LOCAL` 而不是 `LOAD` `DATA` 来加载位于客户端主机上的本地文件，除非默认情况下禁用了 `LOCAL` 能力。

###### 注意

本章中的许多示例假定可以使用 `LOCAL`。如果您的系统不支持这一点，请调整示例：从语句中省略 `LOCAL`，确保文件位于 MySQL 服务器主机上并对服务器可读。

如果 `LOAD` `DATA` 语句中不包含 `LOCAL` 关键字，MySQL 服务器将根据以下规则在服务器主机上查找文件：

+   您的 MySQL 帐户必须具有 `FILE` 权限，要加载的文件必须位于默认数据库的数据目录中或者对所有用户可读。

+   绝对路径名完全指定了文件在文件系统中的位置，服务器从指定位置读取它。

+   相对路径名根据其是否具有单个组件或多个组件两种方式进行解释。对于单个组件文件名（例如 *mytbl.txt*），服务器在默认数据库的数据库目录中查找文件。（如果未选择默认数据库，则操作失败。）对于多个组件文件名（例如 *xyz/mytbl.txt*），服务器从 MySQL 数据目录开始查找文件。也就是说，它期望在名为 *xyz* 的目录中找到 *mytbl.txt*。

+   如果选项 `secure_file_priv` 设置为一个目录路径，MySQL 只能访问此目录中的导入和导出文件。如果使用 `secure_file_priv`，请指定绝对路径。

数据库目录直接位于服务器数据目录下，因此如果默认数据库是 `cookbook`，这两个语句是等效的：

```
mysql> `LOAD DATA INFILE 'mytbl.txt' INTO TABLE mytbl;`
mysql> `LOAD DATA INFILE 'cookbook/mytbl.txt' INTO TABLE mytbl;`
```

如果 `LOAD DATA` 语句包括 `LOCAL` 关键字，则客户端程序将在客户端主机上读取文件并将其内容发送到服务器。客户端解释路径名的方式如下：

+   绝对路径名完全指定了文件在文件系统中的位置。

+   相对路径名指定文件位置相对于您指定 *mysql* 客户端的目录。

如果您的文件位于客户端主机上，但您忘记指示它是本地的，将会发生错误：

```
mysql> `LOAD DATA 'mytbl.txt' INTO TABLE mytbl;`
ERROR 1045 (28000): Access denied for user: '*`user_name`*@*`host_name`*'
(Using password: YES)
```

那个 `Access denied` 的消息可能会让人困惑：如果您能够连接到服务器并发出 `LOAD DATA` 语句，那么看起来您已经获得了对 MySQL 的访问权限，对吧？错误消息意味着服务器（而不是客户端）尝试在服务器主机上打开 *mytbl.txt* 并无法访问它。

如果您的 MySQL 服务器运行在发出 `LOAD DATA` 语句的主机上，则 `<q>remote</q>` 和 `<q>local</q>` 指的是同一主机。但是仍适用于定位数据文件的刚讨论的规则。没有 `LOCAL`，服务器直接读取数据文件。有 `LOCAL`，客户端程序读取文件并将其内容发送到服务器。

*mysqlimport* 使用与 `LOAD DATA` 相同的规则来查找文件。默认情况下，它假定数据文件位于服务器主机上。要指示文件位于客户端主机上，请在命令行上指定 `--local`（或 `-L`）选项。

`LOAD DATA` 假定表位于默认数据库中。要将文件加载到特定数据库中，请使用数据库名称限定表名。以下语句表示 `mytbl` 表位于 `other_db` 数据库中：

```
mysql> `LOAD DATA LOCAL 'mytbl.txt' INTO TABLE other_db.mytbl;`
```

*mysqlimport* 总是需要一个数据库参数：

```
$ `mysqlimport --local cookbook mytbl.txt`
```

`LOAD DATA` 假定数据文件的名称与加载文件内容的表的名称之间没有关系。*mysqlimport* 假定数据文件名的最后一个组成部分确定表名。例如，*mysqlimport* 将 *mytbl*、*mytbl.dat*、*/home/paul/mytbl.csv* 和 *C:\projects\mytbl.txt* 解释为包含 `mytbl` 表数据的文件。

# 13.2 指定列和行分隔符

## 问题

您的数据文件使用非标准的列或行分隔符。

## 解决方案

对于 `LOAD DATA INFILE` 语句，请使用 `FIELDS TERMINATED BY` 和 `LINES TERMINATED BY` 子句，对于 *mysqlimport*，使用 `--fields-terminated-by` 和 `--lines-terminated-by` 选项。

## 讨论

默认情况下，`LOAD` `DATA`假定数据文件的行以换行符（newline）结尾，行内的值由制表符分隔。要提供有关数据文件格式的明确信息，请使用`FIELDS`子句描述行内字段的特性，并使用`LINES`子句指定行结束序列。以下`LOAD` `DATA`语句指示输入文件包含以冒号分隔的数据值，并以回车符结尾：

```
mysql> `LOAD DATA LOCAL INFILE 'mytbl.txt' INTO TABLE mytbl`
    -> `FIELDS TERMINATED BY ':' LINES TERMINATED BY '\r';`
```

每个子句都跟在表名之后。如果两者都存在，`FIELDS`必须在`LINES`之前。行和字段终止指示可以包含多个字符。例如，`\r\n`表示行以回车/换行对结尾。

`LINES`子句还有一个`STARTING` `BY`子句。它指定要从每个输入记录中剥离的序列。（从给定序列开始的所有内容都将被剥离。如果你指定`STARTING` `BY` `'X'`并且记录以`abcX`开头，则所有四个前导字符都将被剥离。）像`TERMINATED` `BY`一样，该序列可以有多个字符。如果`LINES`子句中同时存在`TERMINATED` `BY`和`STARTING` `BY`，它们可以以任意顺序出现。

对于*mysqlimport*，命令选项提供了格式说明符。与前面两个`LOAD` `DATA`语句相对应的命令如下所示：

```
$ `mysqlimport --local cookbook mytbl.txt`
$ `mysqlimport --local --fields-terminated-by=":" --lines-terminated-by="\r" \`
    `cookbook mytbl.txt`
```

对于*mysqlimport*，选项顺序并不重要。

`FIELDS`和`LINES`子句理解十六进制表示法来指定任意格式字符，这对于加载使用二进制格式代码的数据文件非常有用。假设数据文件中的行之间用 Ctrl-A 分隔字段，行末用 Ctrl-B 结束。Ctrl-A 和 Ctrl-B 的 ASCII 值分别为 1 和 2，因此你可以表示它们为`0x01`和`0x02`：

```
FIELDS TERMINATED BY 0x01 LINES TERMINATED BY 0x02
```

*mysqlimport*还能理解格式说明符的十六进制常量。如果你不喜欢在命令行上记住如何键入转义序列，或者在必须在其周围使用引号时，你可能会发现这种功能很有帮助。制表符是`0x09`，换行符是`0x0a`，回车符是`0x0d`。该命令指示数据文件包含以制表符分隔的行，以 CRLF 对结尾：

```
$ `mysqlimport --local --fields-terminated-by=0x09 \`
    `--lines-terminated-by=0x0d0a cookbook mytbl.txt`
```

当你导入数据文件时，不要假设`LOAD` `DATA`（或*mysqlimport*）知道比它实际了解的更多。一些`LOAD` `DATA`的挫折是因为人们期望 MySQL 知道它可能不知道的东西。记住，`LOAD` `DATA`对于你的数据文件的格式一无所知。它对输入结构有一些假设，这些假设是基于行和字段终止符的默认设置，以及引号和转义字符的设置。如果你的输入与这些假设不同，你必须告诉 MySQL。

数据文件中使用的行结束序列通常由生成文件的系统决定。Unix 文件通常以换行符结尾，你可以这样指示：

```
LINES TERMINATED BY '\n'
```

因为 `\n` 恰好是默认的行终止符，因此在这种情况下，您不需要显式指定该子句，除非您希望显式指定行结束序列。如果您的系统上的文件不使用 Unix 默认（换行符）结尾，则必须显式指定行终止符。对于以回车符或回车符/换行符对结尾的文件，分别使用适当的 `LINES` `TERMINATED` `BY` 子句：

```
LINES TERMINATED BY '\r'
LINES TERMINATED BY '\r\n'
```

例如，要加载包含制表字段和以 CRLF 对结尾的 Windows 文件，请使用以下 `LOAD` `DATA` 语句：

```
mysql> `LOAD DATA LOCAL INFILE 'mytbl.txt' INTO TABLE mytbl`
    -> `LINES TERMINATED BY '\r\n';`
```

相应的 *mysqlimport* 命令是：

```
$ `mysqlimport --local --lines-terminated-by="\r\n" cookbook mytbl.txt`
```

如果文件已从一台机器传输到另一台机器，则其内容可能以您不知道的微妙方式发生了更改。例如，如果在不同操作系统上运行的机器之间进行 FTP 传输，则通常将行尾转换为适合目标机器的行尾，前提是传输以文本模式而不是二进制（图像）模式执行。

如果不确定，可以使用十六进制转储程序或显示制表符、回车符和换行符等空白字符的其他实用程序来检查数据文件的内容。在 Unix 下，诸如 *od* 或 *hexdump* 的程序可以以各种格式显示文件内容。如果没有这些工具或类似的实用程序，`recipes` 分发目录中的 *hexdump.pl*、*hexdump.rb* 和 *hexdump.py* 是用 Perl、Ruby 和 Python 编写的十六进制转储程序，以及显示文件所有字符可打印表示的程序 (*see.pl*、*see.rb* 和 *see.py*) 可能对检查文件以查看其实际内容很有用。

# 13.3 处理引号和特殊字符

## 问题

由于您的数据文件包含引号或特殊字符，因此不能使用默认选项加载。

## 解决方案

使用 `LOAD DATA INFILE` 中的 `FIELDS` 子句，结合 `TERMINATED BY`、`ECNLOSED BY` 和 `ESCAPED BY`。对于 *mysqlimport*，使用选项 `--fields-enclosed-by` 和 `--fields-escaped-by`。

## 讨论

如果您的数据文件包含引号值或转义字符，请告知 `LOAD` `DATA` 以便它不会将未解释的数据值加载到数据库中。

`FIELDS` 子句除了 `TERMINATED` `BY` 外，还可以指定其他格式选项。默认情况下，`LOAD` `DATA` 假定值未引用，并且将反斜杠（`\`）解释为特殊字符的转义字符。要显式指示值引用字符，请使用 `ENCLOSED` `BY`；MySQL 在输入处理期间会去掉数据值两端的该字符。要更改默认的转义字符，请使用 `ESCAPED` `BY`。

可以按任意顺序使用子句 `ENCLOSED` `BY`、`ESCAPED` `BY` 和 `TERMINATED` `BY`。例如，这些 `FIELDS` 子句是等效的：

```
FIELDS TERMINATED BY ',' ENCLOSED BY '"'
FIELDS ENCLOSED BY '"' TERMINATED BY ','
```

`TERMINATED BY` 值可以由多个字符组成。如果数据值在输入行内由 `*@*` 序列分隔，指定如下：

```
FIELDS TERMINATED BY '*@*'
```

要完全禁用转义处理，请指定空转义序列：

```
FIELDS ESCAPED BY ''
```

当您指定 `ENCLOSED BY` 表示应从数据值中剥离哪个引号字符时，可以通过将其加倍或在其前面加上转义字符的方式，直接在数据值中包含引号字符。例如，如果引号字符是 `"`，转义字符是 `\`，则输入值 `"a""b\"c"` 被解释为 `a"b"c`。

对于 *mysqlimport*，用于指定引号和转义值的相应命令选项是 `--fields-enclosed-by` 和 `--fields-escaped-by`。（当使用 *mysqlimport* 选项包含引号、反斜杠或其他对您的命令解释器特殊的字符时，您可能需要引用或转义引号或转义字符。）

# 13.4 处理重复键值

## 问题

您的数据文件中存在重复项，导入时出现错误。

## 解决方案

指示 `LOAD DATA INFILE` 和 *mysqlimport* 忽略或替换重复项。

## 讨论

默认情况下，如果输入记录在形成 `PRIMARY KEY` 或 `UNIQUE` 索引的列或列中复制现有行，则会出错。要控制此行为，请在文件名后指定 `IGNORE` 或 `REPLACE`，告诉 MySQL 要么忽略重复行，要么用新行替换旧行。

假设您定期从各种监测站接收当前天气条件的气象数据，并且您将这些站点的各种测量存储在看起来像这样的表中：

```
CREATE TABLE weatherdata
(
  station INT UNSIGNED NOT NULL,
  type    ENUM('precip','temp','cloudiness','humidity','barometer') NOT NULL,
  value   FLOAT,
  PRIMARY KEY (station, type)
);
```

表包括一个主键，组合是站点 ID 和测量类型，以确保每个站点每种测量类型只包含一行。该表旨在仅保存当前条件，因此当为给定站点加载新的测量值到表中时，它们应替换掉站点之前的测量值。为此，使用 `REPLACE` 关键字：

```
mysql> `LOAD DATA LOCAL INFILE 'data.txt' REPLACE INTO TABLE weatherdata;`
```

*mysqlimport* 具有与 `LOAD DATA` 中 `IGNORE` 和 `REPLACE` 关键字对应的 `--ignore` 和 `--replace` 选项。

# 13.5 关于坏输入数据的诊断获取

## 问题

您发现数据文件与导入到数据库中的数据之间存在差异，并希望了解为什么这些值导入失败。

## 解决方案

使用语句 `SHOW WARNINGS`。

## 讨论

`LOAD DATA` 显示一行信息，指示是否存在问题输入值。如果有问题，请使用 `SHOW WARNINGS` 查找其位置和问题内容。

当 `LOAD DATA` 语句完成时，它会返回一行信息，告诉您发生了多少错误或数据转换问题。例如：

```
Records: 134  Deleted: 0  Skipped: 2  Warnings: 13
```

这些值提供了关于导入操作的一般信息：

+   `Records` 表示文件中找到的记录数。

+   `Deleted` 和 `Skipped` 与处理输入记录相关，这些记录在唯一索引值上重复现有表行。`Deleted` 指示从表中删除并由输入记录替换的行数，而 `Skipped` 指示忽略以 favor 存在的行数。

+   `Warnings` 是一个总结，指示在加载数据值到列时发现的问题数量。如果一个值正确存储到列中，或者没有存储。在后一种情况下，该值以不同的方式存储在 MySQL 中，并且 MySQL 计为一个警告。（例如，将字符串 `abc` 存储到数值列中的存储值为 `0`。）

这些值告诉您什么？`Records` 值通常应该与输入文件中的行数相匹配。如果不匹配，则表示 MySQL 解释文件格式与实际格式不同。在这种情况下，您可能还会看到一个较高的 `Warnings` 值，这表示许多值必须转换，因为它们与预期的数据类型不匹配。解决此问题的常见方法是指定适当的 `FIELDS` 和 `LINES` 子句。

假设您的 `FIELDS` 和 `LINES` 格式说明符是正确的，非零 `Warnings` 计数表示存在不良输入值。您无法从 `LOAD DATA` 信息行的数字中判断哪些输入记录存在问题或哪些列不良。要获取此信息，请发出 `SHOW WARNINGS` 语句。

假设表 `t` 有以下结构：

```
CREATE TABLE t
(
  i INT,
  c CHAR(3),
  d DATE
);
```

并假设数据文件 *data.txt* 如下所示：

```
1           1           1
abc         abc         abc
2010-10-10  2010-10-10  2010-10-10
```

将文件加载到表中会导致每个三列加载一个数字、一个字符串和一个日期。这样做会产生多个数据转换和警告，您可以在 `LOAD DATA` 后立即使用 `SHOW WARNINGS` 查看这些警告：

```
mysql> `LOAD DATA LOCAL INFILE 'data.txt' INTO TABLE t;`
Query OK, 3 rows affected, 5 warnings (0.01 sec)
Records: 3  Deleted: 0  Skipped: 0  Warnings: 5
mysql> `SHOW WARNINGS;`
+---------+------+--------------------------------------------------------+
| Level   | Code | Message                                                |
+---------+------+--------------------------------------------------------+
| Warning | 1265 | Data truncated for column 'd' at row 1                 |
| Warning | 1366 | Incorrect integer value: 'abc' for column 'i' at row 2 |
| Warning | 1265 | Data truncated for column 'd' at row 2                 |
| Warning | 1265 | Data truncated for column 'i' at row 3                 |
| Warning | 1265 | Data truncated for column 'c' at row 3                 |
+---------+------+--------------------------------------------------------+
5 rows in set (0.00 sec)
```

`SHOW WARNINGS` 输出帮助您确定哪些值已转换及其原因。结果表如下所示：

```
mysql> `SELECT * FROM t;`
+------+------+------------+
| i    | c    | d          |
+------+------+------------+
|    1 | 1    | 0000-00-00 |
|    0 | abc  | 0000-00-00 |
| 2010 | 201  | 2010-10-10 |
+------+------+------------+
```

# 13.6 跳过数据文件行

## 问题

您希望从数据文件中跳过几行。

## 解决方案

对于 `LOAD DATA INFILE` 使用 `IGNORE ... LINES` 子句和对于 *mysqlimport* 使用 `--ignore-lines` 选项。

## 讨论

要跳过数据文件的前 *`n`* 行，请在 `LOAD DATA` 语句中添加一个 `IGNORE` *`n`* `LINES` 子句。例如，文件可能包含一个包含列标签的初始行。您可以像这样跳过它：

```
mysql> `LOAD DATA LOCAL INFILE 'mytbl.txt' INTO TABLE mytbl`
    -> `IGNORE 1 LINES;`
```

*mysqlimport* 支持一个 `--ignore-lines=` *`n`* 选项，对应于 `IGNORE` *`n`* `LINES`。

# 13.7 指定输入列顺序

## 问题

数据文件和表中的列顺序不同，您需要为导入做出更改。

## 解决方案

导入时指定列的顺序。

## 讨论

`LOAD DATA`假设数据文件中的列与表中的列具有相同的顺序。如果不是这样，请指定一个列表来指示数据文件列应加载到哪些表列中。假设您的表具有列`a`、`b`和`c`，但数据文件的连续列对应于列`b`、`c`和`a`。像这样加载文件：

```
mysql> `LOAD DATA LOCAL INFILE 'mytbl.txt' INTO TABLE mytbl (b,c,a);`
```

*mysqlimport*有一个对应的`--columns`选项，用于指定列名列表：

```
$ `mysqlimport --local --columns=b,c,a cookbook mytbl.txt`
```

# 13.8 在插入之前预处理输入值

## 问题

数据文件中的值不能直接插入到数据库中。您需要在插入之前对它们进行修改。

## 解决方案

使用`LOAD DATA INFILE`和 MySQL 函数的`SET`子句来修改值。

## 讨论

`LOAD DATA`在插入之前可以对输入值进行有限的预处理，有时可以在加载到表之前将输入数据映射到更合适的值，这在值不适合加载到表中的情况下非常有用（例如，它们的单位不正确，或者两个输入字段必须组合并插入到单个列中）。

前一节展示了如何为`LOAD DATA`指定列名列表，以指示输入字段如何对应到表列。列名还可以命名用户定义变量，以便为每个输入记录将输入字段分配给变量。然后，您可以在一个`SET`子句中指定这些变量的计算，命名一个或多个*`col_name`* `=` *`expr`*赋值，用逗号分隔。

假设数据文件具有以下列，并且第一行提供列标签：

```
Date        Time        Name        Weight      State
2006-09-01  12:00:00    Bill Wills  200         Nevada
2006-09-02  09:00:00    Jeff Deft   150         Oklahoma
2006-09-04  03:00:00    Bob Hobbs   225         Utah
2006-09-07  08:00:00    Hank Banks  175         Texas
```

还假设文件将加载到具有以下列的表中：

```
CREATE TABLE t
(
  dt         DATETIME,
  last_name  CHAR(10),
  first_name CHAR(10),
  weight_kg  FLOAT,
  st_abbrev  CHAR(2)
);
```

要导入文件，您必须解决其字段与表列之间的几个不匹配：

+   文件包含单独的日期和时间字段，必须将它们组合成日期时间值，以便插入到`DATETIME`列中。

+   文件包含一个名字字段，必须将其拆分为单独的名和姓值，以便插入到`first_name`和`last_name`列中。

+   文件包含一个以磅为单位的重量，必须将其转换为公斤，以便插入到`weight_kg`列中（1 磅等于 0.454 公斤）。

+   文件包含州名，但表中包含两个字母的缩写。可以通过在`states`表中进行查找来将名称映射到缩写。

要处理这些转换，请跳过包含列标签的第一行，将每个输入列分配给用户定义的变量，并编写一个`SET`子句来执行计算：

```
mysql> `LOAD DATA LOCAL INFILE 'data.txt' INTO TABLE t`
    -> `IGNORE 1 LINES`
    -> `(@date,@time,@name,@weight_lb,@state)`
    -> `SET dt = CONCAT(@date,' ',@time),`
    ->     `first_name = SUBSTRING_INDEX(@name,' ',1),`
    ->     `last_name = SUBSTRING_INDEX(@name,' ',-1),`
    ->     `weight_kg = @weight_lb * .454,`
    ->     `st_abbrev = (SELECT abbrev FROM states WHERE name = @state);`
```

导入操作后，表中包含以下行：

```
mysql> `SELECT * FROM t;`
+---------------------+-----------+------------+-----------+-----------+
| dt                  | last_name | first_name | weight_kg | st_abbrev |
+---------------------+-----------+------------+-----------+-----------+
| 2006-09-01 12:00:00 | Wills     | Bill       |      90.8 | NV        |
| 2006-09-02 09:00:00 | Deft      | Jeff       |      68.1 | OK        |
| 2006-09-04 03:00:00 | Hobbs     | Bob        |    102.15 | UT        |
| 2006-09-07 08:00:00 | Banks     | Hank       |     79.45 | TX        |
+---------------------+-----------+------------+-----------+-----------+
```

`LOAD` `DATA`可以执行数据值重新格式化，就像刚才展示的那样。其他显示此功能用法的示例在其他地方也有出现。（例如，Recipe 13.12 将其用于映射`NULL`值，而 Recipe 14.16 在数据导入期间将非 ISO 日期重写为 ISO 格式。）然而，虽然`LOAD` `DATA`可以将输入值映射到其他值，但它不能完全拒绝包含不合适值的输入记录。要做到这一点，要么预处理输入文件以删除这些记录，要么在加载文件后发出`DELETE`语句。

# 13.9 忽略数据文件列

## 问题

你的数据文件包含额外的字段，不应添加到数据库中。

## 解决方案

在导入数据时指定列顺序。在需要忽略的列位置指定用户定义变量。

## 讨论

如果一行包含比表中列数更多的列，`LOAD` `DATA`会忽略它们（尽管它可能会生成非零的警告计数）。

在行中跳过中间的列涉及到更多步骤。为了处理这个问题，使用一个包含`LOAD` `DATA`指定要忽略列的列列表，并将它们分配给一个虚拟用户定义变量。假设你要加载 Unix 密码文件*/etc/passwd*中的信息，该文件包含以下格式的行：

```
account:password:UID:GID:GECOS:directory:shell
```

还假设你不想加载密码和目录列。用来保存剩余列信息的表如下所示：

```
CREATE TABLE passwd
(
  account   CHAR(8),  # login name
  uid       INT,      # user ID
  gid       INT,      # group ID
  gecos     CHAR(60), # name, phone, office, etc.
  shell     CHAR(60),  # command interpreter
  PRIMARY KEY(account)
);
```

要加载文件，请指定列分隔符为冒号。还要告诉`LOAD` `DATA`跳过包含密码和目录的第二和第六字段。为此，在语句中添加一个列列表。列表应包括每个要加载到表中的列的名称，并且对于要忽略的列使用一个虚拟用户定义变量（你可以对所有这些列使用相同的变量）。生成的语句如下所示：

```
mysql> `LOAD DATA LOCAL INFILE '/etc/passwd' INTO TABLE passwd`
    -> `FIELDS TERMINATED BY ':'`
    -> `(account,@dummy,uid,gid,gecos,@dummy,shell);`
```

相应的*mysqlimport*命令包括`--columns`选项：

```
$ `mysqlimport --local \`
    `--columns="account,@dummy,uid,gid,gecos,@dummy,shell" \`
    `--fields-terminated-by=":" cookbook /etc/passwd`
```

## 参见

忽略列的另一种方法是预处理输入文件以删除列。实用程序*yank_col.pl*包含在配方分发中，可以按任意顺序提取和显示数据文件列。

# 13.10 导入 CSV 文件

## 问题

你想要加载一个 CSV 格式的文件。

## 解决方案

使用适当的格式指示符与`LOAD` `DATA`或*mysqlimport*。

## 讨论

CSV 格式的数据文件包含由逗号分隔而不是制表符的值，并且可能用双引号括起来。一个包含以回车/换行对结束的行的 CSV 文件*mytbl.txt*可以使用`LOAD` `DATA`加载到`mytbl`中：

```
mysql> `LOAD DATA LOCAL INFILE 'mytbl.txt' INTO TABLE mytbl`
    -> `FIELDS TERMINATED BY ',' ENCLOSED BY '"'`
    -> `LINES TERMINATED BY '\r\n';`
```

或者像这样使用*mysqlimport*：

```
$ `mysqlimport --local --lines-terminated-by="\r\n" \`
    `--fields-terminated-by="," --fields-enclosed-by="\"" \`
    `cookbook mytbl.txt`
```

# 13.11 从 MySQL 导出查询结果

## 问题

你想要将 MySQL 查询的结果导出到一个文件或另一个程序。

## 解决方案

使用`SELECT` … `INTO` `OUTFILE`语句，或者重定向*mysql*程序的输出。

## 讨论

`SELECT` … `INTO` `OUTFILE` 语句直接将查询结果导出到服务器主机上的文件中。要在客户端主机上捕获结果，可以重定向 *mysql* 程序的输出。这两种方法各有优劣；请了解它们并根据具体情况选择合适的方法。

### 使用 `SELECT ... INTO OUTFILE` 语句导出

`SELECT ... INTO OUTFILE` 语句的语法将普通的 `SELECT` 与 `INTO` `OUTFILE` *`file_name`* 结合在一起。默认输出格式与 `LOAD` `DATA` 相同，因此以下语句将 `passwd` 表导出为一个制表符分隔、换行符结束的文件 */tmp/passwd.txt*：

```
mysql> `SELECT * FROM passwd INTO OUTFILE '/tmp/passwd.txt';`
```

要更改输出格式，请使用类似于 `LOAD` `DATA` 使用的选项，指示如何引用和分隔列和记录。例如，要以 CSV 格式导出先前在 Recipe 13.1 中创建的 `passwd` 表，并以 CRLF 结束行，请使用此语句：

```
mysql> `SELECT * FROM passwd INTO OUTFILE '/tmp/passwd.txt'`
    -> `FIELDS TERMINATED BY ',' ENCLOSED BY '"'`
    -> `LINES TERMINATED BY '\r\n';`
```

`SELECT` … `INTO` `OUTFILE` 具有以下特性：

+   输出文件由 MySQL 服务器直接创建，因此文件名应指示在服务器主机上写入文件的位置。文件位置的确定使用与未使用 `LOCAL` 的 `LOAD` `DATA` 相同的规则，如 Recipe 13.1 中所述。（该语句没有类似于 `LOAD` `DATA` 的 `LOCAL` 版本。）

+   执行 `SELECT` … `INTO` `OUTFILE` 语句必须具有 MySQL 的 `FILE` 特权。

+   输出文件不能已经存在。（这可以防止 MySQL 覆盖可能很重要的文件。）

+   您应该在服务器主机上拥有登录帐户或某种方式访问该主机上的文件。如果您无法检索输出文件，`SELECT` … `INTO` `OUTFILE` 对您毫无价值。

+   在 Unix 环境下，在 MySQL 8.0.17 之前，该文件是全局可读的，并由用于运行 MySQL 服务器的帐户拥有。这意味着虽然您可以读取文件，但除非您可以使用该帐户登录，否则可能无法删除它。从 MySQL 8.0.17 开始，文件是全局可写的。

+   如果设置了选项 `secure_file_priv`，则只能导出到指定目录。

### 使用 mysql 客户端程序导出

因为 `SELECT` … `INTO` `OUTFILE` 在服务器主机上写入数据文件，所以除非您的 MySQL 帐户具有 `FILE` 特权，否则无法使用它。要将数据导出到您自己拥有的本地文件，请使用其他策略。如果您只需要制表符分隔的输出，请通过使用 *mysql* 程序执行 `SELECT` 语句并将输出重定向到文件来进行“穷人的导出”。这样，您可以将查询结果写入到本地主机上的文件，而无需 `FILE` 特权。以下是一个示例，导出 `passwd` 表中的登录名和命令解释器列：

```
$ `mysql -e "SELECT account, shell FROM passwd" --skip-column-names \`
    `cookbook > shells.txt`
```

`-e`选项指定要执行的语句（参见 Recipe 1.5），`--skip-column-names`告诉 MySQL 不要写入通常在语句输出之前的列名行（参见 Recipe 1.7）。运算符`>`指示*mysql*将输出重定向到文件中。否则结果将被打印到屏幕上。

注意，MySQL 将`NULL`值写为字符串 <q>NULL</q>。根据输出文件的处理方式，可能需要进行一些后处理来转换它们。我们在 Recipe 13.12 中讨论如何在导出和导入过程中处理`NULL`值。

可以通过将查询结果发送到后处理过滤器来生成除了制表符分隔之外的其他格式的输出。例如，要使用井号作为分隔符，将所有制表符转换为`#`字符（*`TAB`*指示输入命令时输入制表符字符的位置）：

```
$ `mysql --skip-column-names -e "`*`your statement here`*`"` *`db_name`* `\`
    `| sed -e "s/`*`TAB`*`/#/g" >` *`output_file`*
```

您也可以为此目的使用*tr*，尽管该实用程序的语法在不同实现中有所不同。对于 Mac OS X 或 Linux，命令如下所示：

```
$ `mysql --skip-column-names -e "`*`your statement here`*`"` *`db_name`* `\`
    `| tr "\t" "#" >` *`output_file`*
```

刚才展示的*mysql*命令使用`--skip-column-names`来防止输出中出现列标签。在某些情况下，包含这些标签可能会有用（例如，在稍后导入文件时）。在这方面，使用*mysql*导出查询结果比`SELECT` … `INTO` `OUTFILE`更灵活，因为后者无法生成包含列标签的输出。

另一种将查询结果导出到客户端主机上的文件的方法是使用配方发布中提供的*mysql_to_text.pl*实用程序。该程序具有选项，使您能够明确指定输出格式。要将查询结果导出为 Excel 电子表格或 XML 文档，请使用*mysql_to_excel.pl*和*mysql_to_xml.pl*实用程序。

# 13.12 导入和导出 NULL 值

## 问题

您需要在数据文件中表示`NULL`值。

## 解决方案

使用一个在其他情况下不存在的值，这样你就可以区分`NULL`和所有其他合法的非`NULL`值。当你导入文件时，将该值的实例转换为`NULL`。

## 讨论

数据文件中没有关于表示`NULL`值的标准，这使得它们在导入和导出操作中变得棘手。困难之处在于`NULL`表示*值的缺失*，而这在数据文件中很难直接表示。使用空列值是最明显的做法，但对于字符串值列来说存在歧义，因为无法区分以这种方式表示的`NULL`和真正的空字符串。对于其他数据类型，空值也可能是个问题。例如，如果将空值加载到数值列中，则会将其存储为`0`而不是`NULL`，因此在输入中它变得无法与真正的`0`区分开。

此问题的通常解决方案是使用数据中未包含的值来表示`NULL`。这就是`LOAD DATA`和*mysqlimport*处理此问题的方式：它们按约定理解`\N`的值表示`NULL`。（只有当`\N`单独出现时，例如`x\N`或`\Nx`，`\N`才会被解释为`NULL`。）例如，如果使用`LOAD DATA`加载以下数据文件，则它将`\N`的实例视为`NULL`：

```
str1    13      1997-10-14
str2    \N      2009-05-07
\N      15      \N
\N      \N     1973-07-14
```

但是您可能希望将`\N`以外的值解释为表示`NULL`，并且可能在不同列中有不同的约定。考虑以下数据文件：

```
str1    13  1997-10-14
str2    -1  2009-05-07
Unknown 15
Unknown -1  1973-07-15
```

第一列包含字符串，`Unknown`表示`NULL`。第二列包含整数，`-1`表示`NULL`。第三列包含日期，空值表示`NULL`。该怎么办？

要处理这种情况，请使用`LOAD DATA`的输入预处理功能：指定一个列列表，将输入值分配给用户定义的变量，并使用`SET`子句将特殊值映射到真正的`NULL`值。如果数据文件命名为*has_nulls.txt*，则以下`LOAD DATA`语句可以正确解释其内容：

```
mysql> `LOAD DATA LOCAL INFILE 'has_nulls.txt'`
    -> `INTO TABLE t (@c1,@c2,@c3)`
    -> `SET c1 = IF(@c1='Unknown',NULL,@c1),`
    ->     `c2 = IF(@c2=-1,NULL,@c2),`
    ->     `c3 = IF(@c3='',NULL,@c3);`
```

导入后的数据如下所示：

```
+------+------+------------+
| c1   | c2   | c3         |
+------+------+------------+
| str1 |   13 | 1997-10-14 |
| str2 | NULL | 2009-05-07 |
| NULL |   15 | NULL       |
| NULL | NULL | 1973-07-15 |
+------+------+------------+
```

前述讨论涉及将`NULL`值解释为导入到 MySQL 中，但在将数据从 MySQL 传输到其他程序时，也需要考虑`NULL`值。以下是一些示例：

+   `SELECT` … `INTO OUTFILE` 将`NULL`值写入为`\N`。其他程序是否能理解这种约定？如果不能，请将`\N`转换为程序理解的内容。例如，`SELECT`语句可以使用以下表达式导出列：

    ```
    IFNULL(*`col_name`*,'Unknown')
    ```

+   您可以使用*mysql*的批处理模式来生成制表符分隔的输出（参见[Recipe 13.11](https://example.org/nch-xfer-xfer-export-query)），但是`NULL`值会作为单词<q>NULL</q>的实例出现在输出中。如果在输出中该单词没有其他出现，您可以尝试后处理它以将其实例转换为更合适的内容。例如，您可以使用一行*sed*命令：

    ```
    $ `sed -e "s/NULL/\\N/g" data.txt > tmp`
    ```

    如果单词<q>NULL</q>出现在表示非`NULL`值的情况下，它是模棱两可的，您应该考虑以不同方式导出数据。例如，使用`IFNULL()`将`NULL`值映射为其他内容。

# 13.13 SQL 格式导出数据

## 问题

您希望以 SQL 格式导出数据。

## 解决方案

使用*mysqldump*或*mysqlpump*。

## 讨论

SQL 格式广泛用于导出和导入数据。它具有诸如可以在 MySQL 客户端中执行的优势，正如我们在 Recipe 1.6 和 Recipe 13.14 中讨论的那样。SQL 文件还可以包含特殊信息，例如复制源位置（Recipe 3.3）、默认字符集和其他信息。SQL 文件可以包含服务器上所有表格、触发器、事件和存储过程的数据，因此您可以使用它们来复制您的 MySQL 安装。

自 MySQL 分发的最早版本就包含了工具*mysqldump*，它允许将数据导出（转储）到 SQL 文件中。*mysqldump*使用非常简单。例如，要转储所有数据库，使用`--all-databases`选项运行它：

```
$ `mysqldump --all-databases > all-databases.sql`
```

要复制数据库`cookbook`中的所有表格，请将其名称作为*mysqldump*的参数：

```
$ `mysqldump cookbook > cookbook.sql`
```

若要仅导出数据库`cookbook`中的几张表，请在数据库名后指定它们的名称。因此，要复制`limbs`和`patients`表，运行以下命令：

```
$ `mysqldump cookbook limbs patients > limbs_patients.sql`
```

Shell 命令`>`将*mysqldump*的输出重定向到文件中。您还可以指定`--result-file`选项，指示*mysqldump*将结果存储在指定的文件中。

结果文件将包含 SQL 指令，允许重新创建数据库及其表格，并填充数据。

通常 MySQL 在高并发环境中工作。因此，*mysqldump*支持以下选项以确保备份文件的一致性：

`--lock-all-tables`

使用读锁在所有数据库中锁定所有表格，防止在完成转储之前对任何表格进行写入。

`--lock-tables`

分别为每个转储的数据库锁定所有表格。此保护仅阻止写入正在导出的数据库，但不能保证多数据库备份的结果一致性。

`--single-transaction`

在转储之前启动一个事务。该选项不会阻止任何写操作，但仍保证备份的一致性。这是备份使用事务存储引擎表格的推荐选项。

###### 小贴士

由于保证一致性可能影响高并发写入的性能，建议在只读复制品上运行*mysqldump*。

*mysqldump*是一个成熟的工具，但它在单线程中导出数据。这可能不符合我们现在的性能期望。因此，自版本 5.7 起，MySQL 分发包括了另一个备份工具：*mysqlpump*。

*mysqlpump*类似于*mysqldump*。您可以使用与*mysqldump*相同的选项来导出所有数据库、单个数据库或只是几张表。但*mysqlpump*还支持并行处理以加快转储过程、进度指示器、更智能地导出用户账户、过滤器等功能，这些是*mysqldump*所不具备的。

因此，要在四个线程中创建整个 MySQL 实例的转储，使用`--single-transaction`选项保护转储，并查看进度条，请运行以下命令：

```
$ `mysqlpump --default-parallelism=4 --single-transaction \`
 > `--watch-progress > all-databases.sql`
Dump progress: 1/2 tables, 0/7 rows
Dump progress: 142/143 tables, 2574113/4076473 rows
Dump completed in 1837
```

###### 注意

*mysqlpump* 支持选项 `--single-transaction`，但不支持 `--lock-all-tables` 和 `--lock-tables`。它有选项 `--add-locks`，该选项会在每个转储表周围添加 `LOCK TABLES` 和 `UNLOCK TABLES` 语句。

## 另请参阅

关于 *mysqldump* 的更多信息，请参见 [mysqldump — A Database Backup Program](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html)，关于 *mysqlpump* 的更多信息，请参见 [mysqlpump — A Database Backup Program](https://dev.mysql.com/doc/refman/8.0/en/mysqlpump.html) 在 MySQL 用户参考手册中。

# 13.14 导入 SQL 数据

## 问题

你有一个 SQL 转储文件并希望导入它。

## 解决方案

使用 *mysql* 客户端或 MySQL Shell 处理文件。

## 讨论

SQL 转储只是一个包含 SQL 命令的文件。因此，你可以像我们在 Recipe 1.6 中讨论的那样使用 *mysql* 客户端读取它。

```
$ `mysql -ucbuser -p cookbook < cookbook.sql`
```

MySQL Shell 在 SQL 模式下支持类似的功能。

要从命令行加载转储，请为 *mysqlsh* 客户端指定选项 `--sql` 并将输入重定向到其中。

```
$ `mysqlsh cbuser:cbpass@127.0.0.1:33060/cookbook --sql < all-databases.sql`
```

要在交互会话中加载转储，请切换到 SQL 模式并使用命令 *\source* 或其快捷方式 *\.*。

```
MySQL  cookbook  SQL > `\source cookbook.sql`
```

# 13.15 将查询结果导出为 XML

## 问题

你想将查询结果导出为 XML 文档。

## 解决方案

使用 *mysql* 客户端或 *mysqldump* 并选择选项 `--xml`。

## 讨论

*mysql* 客户端可以从查询结果生成 XML 格式输出（参见 Recipe 1.7）。

假设名为 `expt` 的表包含实验中的测试分数：

```
mysql> `SELECT * FROM expt;`
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

运行 *mysql* 客户端并选择选项 `--xml`：

```
$ `mysql --xml cookbook -e "SELECT * FROM expt;" < expt.xml`
```

生成的 XML 文档 *expt.xml* 如下所示：

```
<?xml version="1.0"?>

<resultset statement="SELECT * FROM expt"↩
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <row>
	<field name="subject">Jane</field>
	<field name="test">A</field>
	<field name="score">47</field>
  </row>
…
  <row>
	<field name="subject">Marvin</field>
	<field name="test">D</field>
	<field name="score" xsi:nil="true" />
  </row>
</resultset>
```

要使用 *mysqldump* 以选项 `--xml` 导出类似的输出。生成的文件将包含表定义，除非你指定选项 `--no-create-info`：

```
$ `mysqldump --xml cookbook expt`
<?xml version="1.0"?>
<mysqldump >
SET @MYSQLDUMP_TEMP_LOG_BIN = @@SESSION.SQL_LOG_BIN;
SET @@SESSION.SQL_LOG_BIN= 0;
SET @@GLOBAL.GTID_PURGED=/*!80000 '+'*/ '910c760a-0751-11eb-9da8-0242dc638c6c:1-385,
9113f6b1-0751-11eb-9e7d-0242dc638c6c:1-385,
abf2d315-fb9a-11ea-9815-02421e8c78f1:1-52911';
<database name="cookbook">
	<table_structure name="expt">
		<field Field="subject" Type="varchar(10)"↩
		    Null="YES" Key="" Extra="" Comment="" />
		<field Field="test" Type="varchar(5)"↩
		    Null="YES" Key="" Extra="" Comment="" />
		<field Field="score" Type="int"↩
		    Null="YES" Key="" Extra="" Comment="" />
		<options Name="expt" Engine="InnoDB" Version="10" Row_format="Dynamic"↩
		    Rows="8" Avg_row_length="2048" Data_length="16384" Max_data_length="0"↩
		    Index_length="0" Data_free="0" Create_time="2022-02-06 13:06:35"↩
		    Update_time="2022-02-06 13:06:35" Collation="utf8mb4_0900_ai_ci"↩
		    Create_options="" Comment="" />
	</table_structure>
	<table_data name="expt">
	<row>
		<field name="subject">Jane</field>
		<field name="test">A</field>
		<field name="score">47</field>
	</row>
...
	<row>
		<field name="subject">Marvin</field>
		<field name="test">D</field>
		<field name="score" xsi:nil="true" />
	</row>
	</table_data>
</database>
SET @@SESSION.SQL_LOG_BIN = @MYSQLDUMP_TEMP_LOG_BIN;
```

# 13.16 将 XML 导入 MySQL

## 问题

你想将 XML 文档导入到 MySQL 表中。

## 解决方案

使用 `LOAD XML` 语句。

## 讨论

导入 XML 文档取决于能够解析文档并从中提取记录内容。如何执行此操作取决于文档的编写方式。要读取由 *mysql* 客户端创建的 XML 文件，请使用 `LOAD XML` 语句。

要加载我们在 Recipe 13.15 中创建的文件 `expt.xml`，请运行以下命令：

```
LOAD XML LOCAL INFILE 'expt.xml'  INTO TABLE expt;
```

`LOAD XML` 语句会自动识别下面显示的三种不同 XML 格式。

列名作为属性，列值作为属性值

```
<row subject="Jane" test="A" score=47 />
```

列名作为标签，列值作为标签值。

```
<row>
        <subject>Jane</subject>
        <test>B</test>
        <score>50</score>
  </row>
```

列名作为标签 `field` 的属性 `name` 的值，列值作为它们的值。

```
<row>
        <field name="subject">Jane</field>
        <field name="test">C</field>
        <field name="score" xsi:nil="true" />
  </row>
```

这与 *mysql*、*mysqldump* 和其他 MySQL 工具使用的相同格式。

如果你的 XML 文件使用不同的标签名，请使用 `ROWS IDENTIFIED BY` 子句指定它。例如，如果表 `expt` 的行定义如下：

```
<test>
    <field name="subject">Jane</field>
    <field name="test">D</field>
    <field name="score" xsi:nil="true" />
  </test>
```

使用以下语句加载它们：

```
LOAD XML LOCAL INFILE 'expt.xml'  INTO TABLE expt ROWS IDENTIFIED BY '<test>';
```

# 13.17 使用 JSON 格式导入数据

## 问题

你有一个 JSON 文件并希望将其导入到 MySQL 数据库。

## 解决方案

使用 MySQL Shell 实用程序 *importJson*。

## 讨论

JSON 是存储数据的流行格式。您可以让应用程序准备好它，或者希望从 MongoDB 导入数据。

实用程序 *importJson* 接受 JSON 文件的路径和选项字典作为参数。您可以将 JSON 导入到集合中，也可以导入到表中。在后一种情况下，除非默认值 `doc` 对您有效，否则需要指定要存储文档的 `tableColumn`。

文档应包含以新行分隔的 JSON 对象列表。此列表不应是 JSON 数组或其他对象的成员。

```
{"arms": 2, "legs": 2, "thing": "human" } 
{"arms": 0, "legs": 6, "thing": "insect" } 
{"arms": 10, "legs": 0, "thing": "squid" } 
{"arms": 0, "legs": 0, "thing": "fish" } 
{"arms": 0, "legs": 99, "thing": "centipede" } 
{"arms": 0, "legs": 4, "thing": "table" } 
{"arms": 2, "legs": 4, "thing": "armchair" } 
{"arms": 1, "legs": 0, "thing": "phonograph" } 
{"arms": 0, "legs": 3, "thing": "tripod" } 
{"arms": 2, "legs": 1, "thing": "Peg Leg Pete" } 
{"arms": null, "legs": null, "thing": "space alien" }
```

您将在 `recipes` 发行版的 *collections/limbs.json* 文件中找到集合 `CollectionLimbs` 的 JSON 转储。

要将 JSON 文件中的数据插入到 `CollectionLimbs` 集合中，请运行以下代码。

```
 MySQL  cookbook  JS > `options = {`![1](img/1.png)
                    ->   `schema: "cookbook",`
                    ->   `collection: "CollectionLimbs"`
                    -> `}`
                    -> 
{
    "collection": "CollectionLimbs", 
    "schema": "cookbook"
}
 MySQL  cookbook  JS > `util.importJson("limbs.json", options)`![2](img/2.png)
Importing from file "limbscol.json" to collection `cookbook`.`CollectionLimbs` ↩
in MySQL Server at 127.0.0.1:33060

.. 11.. 11
Processed 1.42 KB in 11 documents in 0.0070 sec (11.00 documents/s)
Total successfully imported documents 11 (11.00 documents/s)
```

![1](img/#co_nch-xfer-xfer-importjson-options_co)

首先，创建一个带有选项的字典对象。最少需要指定集合名称和模式。

![2](img/#co_nch-xfer-xfer-importjson-importJson_co)

然后使用 JSON 文件的路径和选项字典作为参数调用 *util.importJson*。

您还可以从命令行调用实用程序 *importJson*，而无需进入交互式 MySQL Shell 会话。要执行此操作，请使用命令 *mysqlsh* 的选项 `--import`，并将 JSON 文件的路径以及目标集合作为参数指定。

```
$ `mysqlsh cbuser:cbpass@127.0.0.1:33060/cookbook \`
> `--import limbs.json CollectionLimbs`
WARNING: Using a password on the command line interface can be insecure.
Importing from file "limbs.json" to collection `cookbook`.`CollectionLimbs` ↩
in MySQL Server at 127.0.0.1:33060

.. 11.. 11
Processed 506 bytes in 11 documents in 0.0067 sec (11.00 documents/s)
Total successfully imported documents 11 (11.00 documents/s)
```

###### 提示

如果数据库中不存在具有特定名称的集合或表，则实用程序 *importJson* 将为您创建它。

# 13.18 从 MongoDB 导入数据

## 问题

您希望从 MongoDB 集合导入数据。

## 解决方案

使用实用程序 *mongoexport* 将集合从 MongoDB 导出到文件，并使用 *importJson* 并选择选项 `"convertBsonTypes": true` 将集合导入到 MySQL 中。

## 解决方案

*importJson* 可以导入使用实用程序 *mongoexport* 从 MongoDB 导出的文档，并将 BSON 数据类型转换为 MySQL 格式。要探索此功能，请将 `"convertBsonTypes": true` 放入选项字典中，并执行导入操作。

```
 MySQL  cookbook  JS > `options = {`
                    ->   `"schema": "cookbook",`
                    ->   `"collection": "blogs",`
                    ->   `"convertBsonTypes": true`
                    -> `}`
                    -> 
{
    "collection": "blogs", 
    "convertBsonTypes": true, 
    "schema": "cookbook"
}
 MySQL  cookbook  JS > `util.importJson("blogs.json", options)`
Importing from file "blogs.json" to collection `cookbook`.`blogs` ↩
in MySQL Server at 127.0.0.1:33060

.. 2.. 2
Processed 240 bytes in 2 documents in 0.0070 sec (2.00 documents/s)
Total successfully imported documents 2 (2.00 documents/s)
```

结果集合 `blogs` 使用 MySQL 格式的数据。我们可以通过使用 MySQL Shell 选择该集合中的所有文档来验证它。

```
 MySQL  cookbook  JS > `shell.getSession().`
                    -> `getSchema('cookbook').`
                    -> `getCollection('blogs').`
                    -> `find()`
                    ->
{
    "_id": "6029abb942e2e9c45760eabc", ![1](img/1.png)
    "author": "Ann Smith",
    "comment": "That's Awesome!",
    "date_created": "2021-02-13T23:01:13.154Z" ![2](img/2.png)
}
{
    "_id": "6029abd842e2e9c45760eabd",
    "author": "John Doe",
    "comment": "Love it!",
    "date_created": "2021-02-14T11:20:03Z"
}
2 documents in set (0.0006 sec)
```

![1](img/#co_ch-xfer-xfer-importjson-bson_oid_co)

BSON OID 值 `"_id":{"$oid":"6029abb942e2e9c45760eabc"}` 转换为 MySQL ID 格式。

![2](img/#co_ch-xfer-xfer-importjson-bson_date_co)

BSON 日期值 `"date_created":{"$date":"2021-02-13T23:01:13.154Z"}` 转换为 MySQL 日期格式。

您将在 `recipes` 发行版的 *collections/blogs.json* 文件中找到集合 `blogs` 的 JSON 转储。

# 13.19 以 JSON 格式导出数据

## 问题

您希望将 MySQL 集合导出到 JSON 文件中。

## 解决方案

使用 MySQL Shell 检索以 JSON 格式返回的结果。如有需要，将输出重定向到文件。

## 讨论

MySQL Shell 允许您以 JSON 格式检索数据。以下代码片段将集合 `CollectionLimbs` 转储并将结果重定向到文件中：

```
$ `mysqlsh cbuser:cbpass@127.0.0.1:33060/cookbook \`
> `-e "limbs=shell.getSession().getSchema('cookbook').`
> `getCollection('CollectionLimbs').` ![1](img/1.png)
> `find().execute().fetchAll();` ![2](img/2.png)
> `println(limbs);" > limbs.json` ![3](img/3.png)
```

![1](img/#co_nch-xfer-xfer-exportjson-select_co)

选择集合。

![2](img/#co_nch-xfer-xfer-exportjson-fetch_co)

从集合中获取所有行。

![3](img/#co_nch-xfer-xfer-exportjson-print_co)

打印结果并将命令输出重定向到文件中。

结果文件将包含一组 JSON 文档。这不是 MySQL Shell 实用程序 *importJson* 可以使用的相同格式。如果要将数据导入 MySQL，请修改结果文件。您可以借助实用程序 *jq* 来完成此操作。

```
$ `jq '.[]' limbs.json > limbs_fixed.json`
```

*jq* 读取文件 `limbs.json` 并将其每个数组元素打印到标准输出。然后，我们将结果重定向到文件 `limbs_fixed.json` 中。

## 另请参阅

有关实用程序 *jq* 的更多信息，请参阅 [jq 手册](https://stedolan.github.io/jq/manual/)。

# 13.20 从数据文件中猜测表结构

## 问题

有人给了您一个数据文件，并说：<q>嗨，请把这个放到 MySQL 中。</q> 但还没有表来存储数据。您需要创建一个可以容纳文件数据的表。

## 解决方案

使用一个通过检查数据文件内容来猜测表结构的实用程序。

## 讨论

有时您必须导入 MySQL 中尚未设置表的数据。您可以根据您对文件内容的了解自己创建表格。或者您可以使用 *guess_table.pl*，这是分发中 *transfer* 目录中的一个实用程序。*guess_table.pl* 读取数据文件以查看其中包含的信息类型，然后尝试生成与文件内容匹配的适当 `CREATE` `TABLE` 语句。这个脚本是不完美的，因为列内容有时是含糊不清的。（例如，包含少量不同字符串的列可能是 `VARCHAR` 列或 `ENUM` 列。）尽管如此，调整 *guess_table.pl* 生成的 `CREATE` `TABLE` 语句可能比从头开始编写更容易。尽管这个实用程序也具有诊断价值，但这并不是它的主要目的。例如，如果您认为某列只包含数字，但 *guess_table.pl* 表示它应该是 `VARCHAR` 列，这表明该列至少包含一个非数字值。

*guess_table.pl* 假设其输入为制表符分隔的换行终止格式。它还假设输入有效，因为基于可能存在缺陷的数据来猜测数据类型是注定失败的。这意味着，例如，如果要识别日期列，它应该采用 ISO 格式。否则，*guess_table.pl* 可能会将其描述为 `VARCHAR` 列。如果数据文件不满足这些假设，您可以首先使用配方分发中提供的实用程序 *cvt_file.pl* 和 *cvt_date.pl* 重新格式化它。

*guess_table.pl* 理解以下选项：

`--labels`

将第一行输入解释为列标签行，并将其用作表列名。如果没有此选项，*guess_table.pl* 将使用`c1`、`c2`等默认列名。

如果文件包含一个标签行而您省略此选项，*guess_table.pl*将标签视为数据值。由于列中存在非数字或非时间值，脚本可能将*所有*列描述为`VARCHAR`列。

`--lower`, `--upper`

强制在`CREATE TABLE`语句中使用小写或大写的列名。

`--quote-names`, `--skip-quote-names`

在`CREATE TABLE`语句中使用 `` ` `` 字符引用或不引用表和列标识符（例如，`` `mytbl` ``）。如果标识符是保留字，这可能很有用。默认情况是引用标识符。

`--report`

生成报告而不是`CREATE TABLE`语句。该脚本显示它收集到的有关每列的信息。

`--table``=`*`tbl_name`*

指定在`CREATE TABLE`语句中使用的表名称。默认名称为`t`。

这是*guess_table.pl*的工作示例。假设名为*commodities.csv*的文件以 CSV 格式存在，并且内容如下：

```
commodity,trade_date,shares,price,change
sugar,12-14-2014,1000000,10.50,-.125
oil,12-14-2014,96000,60.25,.25
wheat,12-14-2014,2500000,8.75,0
gold,12-14-2014,13000,103.25,2.25
sugar,12-15-2014,970000,10.60,.1
oil,12-15-2014,105000,60.5,.25
wheat,12-15-2014,2370000,8.65,-.1
gold,12-15-2014,11000,101,-2.25
```

第一行指示列标签，随后的行包含数据记录，每行一条。`trade_date`列中的值是日期，但它们使用的是*`MM-DD-YYYY`*格式，而不是 MySQL 期望的 ISO 格式。*cvt_date.pl* 可以将这些日期转换为 ISO 格式。但是，*cvt_date.pl*和*guess_table.pl*都要求以制表符分隔、以换行符结尾的格式输入，因此首先使用*cvt_file.pl*将输入转换为制表符分隔、以换行符结尾的格式，然后使用*cvt_date.pl*转换日期：

```
$ `cvt_file.pl --iformat=csv commodities.csv > tmp1.txt`
$ `cvt_date.pl --iformat=us tmp1.txt > tmp2.txt`
```

将生成的文件*tmp2.txt*输入到*guess_table.pl*中：

```
$ `guess_table.pl --labels --table=commodities tmp2.txt > commodities.sql`
```

*guess_table.pl* 写入 *commodities.sql* 的`CREATE TABLE`语句如下所示：

```
CREATE TABLE `commodities`
(
  `commodity` VARCHAR(5) NOT NULL,
  `trade_date` DATE NOT NULL,
  `shares` BIGINT UNSIGNED NOT NULL,
  `price` DOUBLE UNSIGNED NOT NULL,
  `change` DOUBLE NOT NULL
);
```

*guess_table.pl* 根据以下启发性原则生成该语句：

+   如果只包含数字值的列不包含小数点，则假定为`BIGINT`类型，否则为`DOUBLE`类型。

+   不包含负值的数值列可能是`UNSIGNED`类型。

+   如果列不包含空值，则*guess_table.pl*假定它可能是`NOT NULL`。

+   无法归类为数字或日期的列被视为`VARCHAR`列，长度等于该列中出现的最长值。

您可能需要编辑*guess_table.pl*生成的`CREATE TABLE`语句，以进行修改，例如使用较小的整数类型，增加字符字段的大小，将`VARCHAR`更改为`CHAR`，添加索引或更改 MySQL 中的保留字列名。

要创建表，请使用*guess_table.pl*生成的语句：

```
$ `mysql cookbook < commodities.sql`
```

然后将数据文件加载到表中（跳过初始标签行）：

```
mysql> `LOAD DATA LOCAL INFILE 'tmp2.txt' INTO TABLE commodities`
    -> `IGNORE 1 LINES;`
```

导入后生成的表格内容如下：

```
mysql> `SELECT * FROM commodities;`
+-----------+------------+---------+--------+--------+
| commodity | trade_date | shares  | price  | change |
+-----------+------------+---------+--------+--------+
| sugar     | 2014-12-14 | 1000000 |   10.5 | -0.125 |
| oil       | 2014-12-14 |   96000 |  60.25 |   0.25 |
| wheat     | 2014-12-14 | 2500000 |   8.75 |      0 |
| gold      | 2014-12-14 |   13000 | 103.25 |   2.25 |
| sugar     | 2014-12-15 |  970000 |   10.6 |    0.1 |
| oil       | 2014-12-15 |  105000 |   60.5 |   0.25 |
| wheat     | 2014-12-15 | 2370000 |   8.65 |   -0.1 |
| gold      | 2014-12-15 |   11000 |    101 |  -2.25 |
+-----------+------------+---------+--------+--------+
```
