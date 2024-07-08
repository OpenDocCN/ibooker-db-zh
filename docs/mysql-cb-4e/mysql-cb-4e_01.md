# 第一章：使用 mysql 客户端程序

# 1.0 介绍

MySQL 数据库系统使用客户端-服务器架构。服务器*mysqld*是实际操作数据库的程序。要告诉服务器要做什么，请使用一个客户端程序，该程序通过使用结构化查询语言（SQL）编写的语句来传达您的意图。客户端程序用于各种不同的目的，但每个客户端通过连接到服务器、发送 SQL 语句以执行数据库操作并接收结果来与服务器交互。

客户端程序安装在要访问 MySQL 的机器上，但服务器可以安装在任何地方，只要客户端能够连接到它。由于 MySQL 是一个固有的网络数据库系统，客户端可以与在您自己的机器上本地运行的服务器或者地球另一侧的某处运行的服务器进行通信。

*mysql*程序是 MySQL 发行版中包含的客户端之一。在交互模式下使用时，*mysql*提示您输入语句，将其发送到 MySQL 服务器执行，并显示结果。*mysql*也可以在非交互模式下使用批处理模式，从文件中读取语句或由程序生成的语句。这使得可以在脚本或*cron*作业中使用*mysql*，或者与其他应用程序配合使用。

本章描述了*mysql*的功能，以便您可以更有效地使用它：

+   设置一个用于使用`cookbook`数据库的 MySQL 账户

+   指定连接参数和使用选项文件

+   以交互和批处理模式执行 SQL 语句

+   控制*mysql*输出格式

+   使用用户定义变量保存信息

要尝试本书中展示的示例，您需要一个 MySQL 用户账户和一个数据库。本章的前两个示例描述了如何使用*mysql*设置这些内容，基于以下假设：

+   MySQL 服务器在您自己的系统上本地运行

+   您的 MySQL 用户名和密码分别是`cbuser`和`cbpass`

+   您的数据库名为`cookbook`

如果您愿意，可以违反任何假设。您的服务器不必在本地运行，也不必使用本书中使用的用户名、密码或数据库名。当然，在这种情况下，您必须相应地修改示例。

即使您选择不使用`cookbook`作为您的数据库名称，我们建议您使用一个专门用于此处示例的数据库，而不是您还用于其他目的的数据库。否则，现有表的名称可能会与示例中使用的名称冲突，您将不得不进行不必要的修改。

本章中创建表的脚本位于伴随*MySQL Cookbook*的`recipes`发行版的*tables*目录中。其他脚本位于*mysql*目录中。要获取`recipes`发行版，请参阅前言。

# 1.1 设置 MySQL 用户账户

## 问题

你需要一个账户来连接到你的 MySQL 服务器。

## 解决方案

使用 `CREATE` `USER` 和 `GRANT` 语句设置账户。然后使用账户名和密码连接服务器。

## 讨论

连接到 MySQL 服务器需要用户名和密码。你可能还需要指定运行服务器的主机名。如果不显式指定连接参数，*mysql* 将假定默认值。例如，如果未指定主机名，*mysql* 将假定服务器运行在本地主机上。

如果其他人已经为你设置了账户并授予了权限，允许你创建和修改 `cookbook` 数据库，只需使用该账户。否则，下面的示例展示了如何使用 *mysql* 程序连接服务器并执行设置具有访问名为 `cookbook` 数据库权限的用户账户的语句。*mysql* 的参数包括 `-h` `localhost` 以连接到运行在本地主机上的 MySQL 服务器，`-u` `root` 以 MySQL `root` 用户身份连接，以及 `-p` 以提示 *mysql* 输入密码：

```
$ `mysql -h localhost -u root -p`
Enter password: `******`
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 54117
Server version: 8.0.27 MySQL Community Server - GPL

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> `CREATE USER 'cbuser'@'localhost' IDENTIFIED BY 'cbpass';`
mysql> `GRANT ALL ON cookbook.* TO 'cbuser'@'localhost';`
Query OK, 0 rows affected (0.09 sec)
mysql> ``GRANT PROCESS ON *.* to  `cbuser`@`localhost` ;``
Query OK, 0 rows affected (0,01 sec)
mysql> `quit`
Bye
```

###### 提示

如果你需要生成 MySQL 数据的转储文件，则需要 `PROCESS` 权限。另见 Recipe 1.4。

如果尝试调用 *mysql* 时出现无法找到或无效命令的错误消息，这意味着你的命令解释器不知道 *mysql* 安装在哪里。参见 Recipe 1.3 了解设置解释器用于查找命令的 `PATH` 环境变量的信息。

在显示的命令中，`$` 表示你的 shell 或命令解释器显示的提示符，`mysql>` 是 *mysql* 显示的提示符。加粗显示的文本是你要输入的内容。非加粗文本（包括提示符）是程序的输出；不要输入这些内容。

当 *mysql* 打印密码提示时，在看到 `******` 处输入 MySQL `root` 密码；如果 MySQL `root` 用户没有密码，只需在密码提示处按 Enter（或 Return）键。你会看到 MySQL 欢迎提示，这可能因你使用的 MySQL 版本而略有不同。接着按照示例输入 `CREATE` `USER` 和 `GRANT` 语句。

`quit` 命令终止你的 *mysql* 会话。你也可以使用 `exit` 命令或（在 Unix 下）输入 Ctrl-D 来终止会话。

要授予 `cbuser` 账户对除 `cookbook` 之外的数据库的访问权限，在 `GRANT` 语句中，替换掉 `cookbook` 的数据库名称。要为现有账户授予 `cookbook` 数据库的访问权限，省略 `CREATE` `USER` 语句，并在 `GRANT` 语句中用 `'cbuser'@'localhost'` 替换该账户。

###### 注意

MySQL 用户账户记录由两部分组成：用户名和主机。用户名是访问 MySQL 服务器的标识或用户。对于此部分，你可以指定任何内容。主机名是 IP 地址或访问 MySQL 服务器的主机名称。我们在 Recipe 24.0 中讨论 MySQL 安全模型和用户账户。

`'cbuser'@'localhost'` 中的主机名部分指示了*从哪里*你将连接到 MySQL 服务器。要设置一个连接到本地主机上运行的服务器的账户，请使用 `localhost`，如所示。如果你计划从另一台主机连接到服务器，请在 `CREATE` `USER` 和 `GRANT` 语句中替换该主机。例如，如果你将从名为 *myhost.example.com* 的主机连接到服务器，则语句如下：

```
mysql> `CREATE USER 'cbuser'@'myhost.example.com' IDENTIFIED BY 'cbpass';`
mysql> `GRANT ALL ON cookbook.* TO 'cbuser'@'myhost.example.com';`
```

可能你已经意识到这个过程中存在一个悖论：要设置一个可以连接到 MySQL 服务器的 `cbuser` 账户，你必须先连接到服务器，以便执行 `CREATE` `USER` 和 `GRANT` 语句。我假设你已经能够以 MySQL 的 `root` 用户连接，因为只有像 `root` 这样具有设置其他用户账户所需的管理特权的用户，才能使用 `CREATE` `USER` 和 `GRANT`。如果你无法作为 `root` 用户连接到服务器，请向你的 MySQL 管理员请求创建 `cbuser` 账户。

创建完 `cbuser` 账户后，请验证你能够使用它连接到 MySQL 服务器。从 `CREATE` `USER` 语句中命名的主机运行以下命令来执行此操作（`-h` 后命名的主机应为 MySQL 服务器运行的主机）：

```
$ `mysql -h localhost -u cbuser -p`
Enter password: `cbpass`
```

现在，你可以继续创建 `cookbook` 数据库并在其中创建表，如 Recipe 1.2 所述。为了方便每次调用 *mysql* 时无需指定连接参数，可以将它们放入选项文件中（参见 Recipe 1.4）。

## 参见

关于管理 MySQL 账户的更多信息，请参阅 Chapter 24。

# 1.2 创建数据库和样例表

## 问题

你想创建一个数据库并在其中设置表。

## 解决方案

使用 `CREATE` `DATABASE` 语句创建数据库，为每个表使用 `CREATE` `TABLE` 语句，并使用 `INSERT` 语句向表中添加行。

## 讨论

在 Recipe 1.1 中显示的 `GRANT` 语句设置了访问 `cookbook` 数据库的权限，但没有创建数据库。本节展示如何创建数据库，以及如何创建表并将用于本书其他部分示例的示例数据加载到其中。在本书的其他地方使用类似的指令来创建其他表。

如 Recipe 1.1 结尾所示连接到 MySQL 服务器，然后像这样创建数据库：

```
mysql> `CREATE DATABASE cookbook;`
```

现在你已经有了一个数据库，可以在其中创建表格。首先，选择`cookbook`作为默认数据库：

```
mysql> `USE cookbook;`
```

创建一个简单的表格：

```
mysql> `CREATE TABLE limbs (thing VARCHAR(20), legs INT, arms INT, PRIMARY KEY(thing));`
```

然后用几行填充它：

```
mysql> `INSERT INTO limbs (thing,legs,arms) VALUES('human',2,2);`
mysql> `INSERT INTO limbs (thing,legs,arms) VALUES('insect',6,0);`
mysql> `INSERT INTO limbs (thing,legs,arms) VALUES('squid',0,10);`
mysql> `INSERT INTO limbs (thing,legs,arms) VALUES('fish',0,0);`
mysql> `INSERT INTO limbs (thing,legs,arms) VALUES('centipede',99,0);`
mysql> `INSERT INTO limbs (thing,legs,arms) VALUES('table',4,0);`
mysql> `INSERT INTO limbs (thing,legs,arms) VALUES('armchair',4,2);`
mysql> `INSERT INTO limbs (thing,legs,arms) VALUES('phonograph',0,1);`
mysql> `INSERT INTO limbs (thing,legs,arms) VALUES('tripod',3,0);`
mysql> `INSERT INTO limbs (thing,legs,arms) VALUES('Peg Leg Pete',1,2);`
mysql> `INSERT INTO limbs (thing,legs,arms) VALUES('space alien',NULL,NULL);`
```

###### 提示

为了更轻松地输入`INSERT`语句：在输入第一个语句后，按上箭头以回忆起它，按退格键（或删除键）几次以擦除字符，直到回到最后一个开括号，然后输入下一个语句的数据值。或者，为了完全避免输入`INSERT`语句，可以直接跳到配方 1.6。

刚刚创建的表格名为`limbs`，包含三列，用于记录各种生物和物体的腿和手臂数量。最后一行的外星人生理结构使得`arms`和`legs`列的合适值无法确定；`NULL`表示<q>未知值</q>。

`PRIMARY KEY`子句定义了唯一标识表行的主键。这可以防止将不明确的数据插入表中，还有助于 MySQL 更快地执行查询。我们在第十八章讨论不明确的数据和第二十一章中的性能问题。

通过执行`SELECT`语句验证是否将行添加到`limbs`表中：

```
mysql> `SELECT * FROM limbs;`
+--------------+------+------+
| thing        | legs | arms |
+--------------+------+------+
| human        |    2 |    2 |
| insect       |    6 |    0 |
| squid        |    0 |   10 |
| fish         |    0 |    0 |
| centipede    |   99 |    0 |
| table        |    4 |    0 |
| armchair     |    4 |    2 |
| phonograph   |    0 |    1 |
| tripod       |    3 |    0 |
| Peg Leg Pete |    1 |    2 |
| space alien  | NULL | NULL |
+--------------+------+------+
11 rows in set (0,01 sec)
```

到此为止，你已经设置了一个数据库和一个表格。关于执行 SQL 语句的更多信息，请参阅配方 1.5 和配方 1.6。

###### 注意

在本书中，语句中的 SQL 关键字（如`SELECT`或`INSERT`）以大写字母显示以突出显示。这只是一种排版约定；关键字可以是任何大小写。

# 1.3 查找 mysql 客户端

## 问题

当你从命令行调用*mysql*客户端时，你的命令解释器找不到它。

## 解决方案

将安装*mysql*的目录添加到你的`PATH`设置中。然后，你可以轻松地从任何目录运行*mysql*。

## 讨论

如果在调用时，你的 shell 或命令解释器找不到*mysql*，你会看到某种错误消息。在 Unix 下可能看起来像这样：

```
$ `mysql`
mysql: Command not found.
```

或者在 Windows 下可以这样：

```
C:\> `mysql.exe`
'mysql.exe' is not recognized as an internal or external command,↩
operable program or batch file.
```

告诉你的命令解释器在哪里找到*mysql*的一种方法是每次运行时键入其完整路径名。在 Unix 下，该命令可能看起来像这样：

```
$ `/usr/local/mysql/bin/mysql`
```

或者在 Windows 下可以这样：

```
C:\> `"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql"`
```

快速输入长路径名会很快让人感到厌倦。你可以通过在运行之前将位置更改为安装*mysql*的目录来避免这样做。但是如果这样做，你可能会诱惑把所有数据文件和 SQL 批处理文件放在与*mysql*相同的目录中，从而在本应仅用于程序的位置上产生不必要的混乱。

更好的解决方案是修改你的 `PATH` 搜索路径环境变量，该变量指定命令解释器查找命令的目录。将 *mysql* 安装目录添加到 `PATH` 值中。然后，只需输入其名称，就可以从任何位置调用 *mysql*，从而省去输入路径名的麻烦。有关设置 `PATH` 变量的说明，请阅读伴随网站上的<q>在命令行中执行程序</q>（参见前言）。

在 Windows 上，另一种避免输入路径名或切换到 *mysql* 目录的方法是创建一个快捷方式，并将其放置在更方便的位置，如桌面。这样一来，只需打开快捷方式即可简单地启动 *mysql*。要指定命令选项或启动目录，请编辑快捷方式的属性。如果不总是使用相同的选项调用 *mysql*，则创建多个快捷方式可能会很有用。例如，创建一个用于普通用户一般工作连接的快捷方式，另一个用于作为 MySQL `root` 用户进行管理目的的连接。

# 1.4 指定 mysql 命令选项

## 问题

当你在没有命令选项的情况下调用 *mysql* 程序时，它会立即退出，并显示错误消息。

## 解决方案

必须指定连接参数。可以在命令行、选项文件或两者混合使用来执行此操作。

## 讨论

如果你在没有命令选项的情况下调用 *mysql*，可能会导致<q>拒绝访问</q>错误。为了避免这种情况，可以按照 Recipe 1.1 中所示连接到 MySQL 服务器，像这样使用 *mysql*：

```
$ `mysql -h localhost -u cbuser -p`
Enter password: `cbpass`
```

每个选项都有单破折号<q>短</q>形式：`-h` 和 `-u` 用于指定主机名和用户名，`-p` 用于提示输入密码。还有对应的双破折号<q>长</q>形式：`--host`、`--user` 和 `--password`。使用方法如下：

```
$ `mysql --host=localhost --user=cbuser --password`
Enter password: `cbpass`
```

要查看 *mysql* 支持的所有选项，请使用以下命令：

```
$ `mysql --help`
```

指定 *mysql* 的命令选项的方式也适用于其他 MySQL 程序，如 *mysqldump* 和 *mysqladmin*。例如，要生成名为 *cookbook.sql* 的转储文件，其中包含 `cookbook` 数据库中表的备份，可以像这样执行 *mysqldump*：

```
$ `mysqldump -h localhost -u cbuser -p cookbook > cookbook.sql`
Enter password: `cbpass`
```

一些操作需要具有管理员权限的 MySQL 帐户。*mysqladmin* 程序可以执行仅限于 MySQL `root` 帐户的操作。例如，要停止服务器，请按以下方式调用 *mysqladmin*：

```
$ `mysqladmin -h localhost -u root -p shutdown`
Enter password:        ← enter MySQL root account password here
```

如果选项的值与其默认值相同，则可以省略该选项。但是，没有默认密码。如果愿意，可以直接在命令行上使用 `-p`*`password`*（选项和密码之间*无空格*）或 `--password``=`*`password`* 指定密码。

###### 警告

我们不建议这样做，因为密码对旁观者可见，并且在多用户系统上，可能会被运行诸如*ps*报告进程信息或可以读取您的 shell 历史文件内容的其他用户发现。

因为默认主机是`localhost`，与我们明确指定的相同值，您可以在命令行中省略`-h`（或`--host`）选项：

```
$ `mysql -u cbuser -p`
```

假设您真的不想指定*任何*选项。如何让*mysql*“仅知道”要使用哪些值？这很容易，因为 MySQL 程序支持选项文件：

+   如果将选项放入选项文件中，则无需在每次调用特定程序时在命令行上指定它。

+   您可以混合使用命令行和选项文件选项。这使您可以将最常用的选项值存储在文件中，并根据需要在命令行上覆盖它们。

这一节的其余部分描述了这些功能。

### 使用选项文件指定连接参数

为了避免每次调用*mysql*时在命令行中输入选项，将它们放入一个选项文件中，*mysql*会自动读取。选项文件是纯文本文件：

+   在 Unix 下，您的个人选项文件位于家目录中的*.my.cnf*文件中。管理员还可以使用站点范围的选项文件指定适用于所有用户的参数。您可以使用 MySQL 安装目录下的*/etc*或*/etc/mysql*目录中的*my.cnf*文件，或者在*etc*目录下的*etc*目录中。

+   在 Windows 下，您可以使用的文件包括 MySQL 安装目录中的*my.ini*或*my.cnf*文件（例如*C:\Program Files\MySQL\MySQL Server 8.0*），您的 Windows 目录（可能是*C:\WINDOWS*），或*C:\*目录。

要查看允许的选项文件位置的确切列表，请调用*mysql* `--help`。

下面的示例说明了 MySQL 选项文件中使用的格式：

```
# general client program connection options
[client]
host     = localhost
user     = cbuser
password = cbpass

# options specific to the mysql program
[mysql]
skip-auto-rehash
pager="/usr/bin/less -i" # specify pager for interactive mode
```

使用刚刚显示的`[client]`组中列出的连接参数，您可以通过在命令行上没有任何选项的情况下调用*mysql*来连接为`cbuser`。

```
$ `mysql`
```

对于其他 MySQL 客户端程序（例如*mysqldump*）也是如此。

###### 警告

选项`password`以明文格式存储在配置文件中，任何有权访问此文件的用户都可以读取它。如果您想要安全存储连接凭据，应使用*mysql_config_editor*。

*mysql_config_editor*将连接凭据存储在位于 Unix 家目录下的*.mylogin.cnf*文件或 Windows 下的*%APPDATA%\MySQL*目录中。它仅支持连接参数`host`、`user`、`password`和`socket`。选项`--login-path`指定存储凭据的组。默认为`[client]`。

这是一个示例，演示如何使用*mysql_config_editor*创建加密的登录文件。

```
$ `mysql_config_editor set --login-path=client \`
> `--host=localhost --user=cbuser --password`
Enter password: `cbpass`

# print stored credentials
$ `mysql_config_editor print --all`
[client]
user = cbuser
password = *****
host = localhost
```

MySQL 选项文件具有以下特点：

+   Lines are written in groups (or sections). The first line of a group specifies the group name within square brackets, and the remaining lines specify options associated with the group. The example file just shown has a `[client]` group and a `[mysql]` group. To specify options for the server, *mysqld*, put them in a `[mysqld]` group.

+   用于指定客户端连接参数的通常选项组是`[client]`。这个组实际上被所有标准的 MySQL 客户端使用。通过在这个组中列出选项，您不仅可以更容易地调用*mysql*，还可以调用其他程序，如*mysqldump*和*mysqladmin*。只需确保您放入该组的任何选项被所有客户端程序理解。否则，调用任何不理解它的客户端将导致“未知选项”错误。

+   您可以在选项文件中定义多个组。按照惯例，MySQL 客户端会查找`[client]`组中的参数以及以程序本身命名的组。这为列出您希望所有客户端程序使用的一般客户端参数提供了方便的方法，但您仍然可以指定仅适用于特定程序的选项。上述示例选项文件说明了此约定，对于*mysql*程序来说，它从`[client]`组获取一般连接参数，并从`[mysql]`组中获取`skip-auto-rehash`和`pager`选项。

+   在一个组内，以*`name=value`*的格式编写选项行，其中*`name`*对应于选项名称（不带前导破折号），而*`value`*是选项的值。如果选项不需要值（例如 `skip-auto-rehash`），则仅列出名称，不需要尾随的*`=value`*部分。

+   在选项文件中，只允许使用选项的长形式，而不是短形式。例如，在命令行上，可以使用 `-h` *`host_name`* 或 `--host``=`*`host_name`* 来指定主机名。在选项文件中，只允许使用 `host=`*`host_name`*。

+   许多程序，包括*mysql*和*mysqld*，除了命令选项外，还有程序变量。对于服务器来说，这些称为*系统变量*；参见第 22.1 节。程序变量可以像选项一样在选项文件中指定。在内部，程序变量名称使用下划线，但在选项文件中，可以互换使用破折号或下划线。例如，`skip-auto-rehash`和`skip_auto_rehash`是等效的。要在`[mysqld]`选项组中设置服务器的`sql_mode`系统变量，`sql_mode=`*`value`* 和 `sql-mode=`*`value`* 是等效的。（破折号和下划线的互换性也适用于命令行上指定的选项或变量。）

+   在选项文件中，`=` 分隔选项名称和值时允许空格。这与命令行不同，命令行中 `=` 两侧不允许有空格。如果选项值包含空格或其他特殊字符，可以使用单引号或双引号进行引用。`pager` 选项说明了这一点。

+   使用选项文件来指定连接参数（比如 `host`、`user` 和 `password`）是很常见的。然而，这个文件可以列出其他用途的选项。在 `[mysql]` 组中显示的 `pager` 选项指定了*mysql*在交互模式下显示输出时应该使用的分页程序。这与程序如何连接服务器无关。我们不建议将 `password` 放入选项文件，因为它以明文形式存储，可能会被有文件系统访问权限但不一定有 MySQL 安装访问权限的用户发现。

+   如果选项文件中的参数出现多次，则以找到的最后一个值为准。通常，您应该按照 `[client]` 组后跟随任何特定于程序的组，以便如果这两组设置的选项有重叠，程序特定的值将覆盖更一般的选项。

+   以 `#` 或 `;` 字符开头的行被视为注释而被忽略。空行也被忽略。`#` 可以用来在选项行末尾写注释，就像 `pager` 选项所示。

+   指定文件或目录路径的选项应使用 `/` 作为路径分隔符，即使在使用 `\` 作为路径分隔符的 Windows 系统下也是如此。或者，使用 `\\` 来表示 `\`（这是必要的，因为 `\` 在字符串中是 MySQL 的转义字符）。

要查找 *mysql* 程序将从选项文件中读取哪些选项，请使用此命令：

```
$ `mysql --print-defaults`
```

您还可以使用 *my_print_defaults* 实用程序，该实用程序以选项文件组的名称作为参数。例如，*mysqldump* 在 `[client]` 和 `[mysqldump]` 组中查找选项。要检查这些组中的选项文件设置，使用此命令：

```
$ `my_print_defaults client mysqldump`
```

### 混合命令行和选项文件参数

可以混合命令行选项和选项文件中的选项。也许您想在选项文件中列出用户名和服务器主机，但宁愿不要在那里存储密码。这没问题；MySQL 程序首先读取您的选项文件以查看那里列出了哪些连接参数，然后检查命令行以获取额外的参数。这意味着您可以以一种方式指定一些选项，以另一种方式指定另一些选项。例如，您可以在选项文件中列出用户名和主机名，但在命令行上使用密码选项：

```
$ `mysql -p`
Enter password:        ← enter your password here
```

命令行参数优先于选项文件中找到的参数，因此要覆盖选项文件参数，只需在命令行上指定它。例如，您可以在选项文件中列出常规 MySQL 用户名和密码以供通用使用。然后，如果必须偶尔作为 MySQL `root`用户连接，则在命令行上指定用户和密码选项以覆盖选项文件中的值：

```
$ `mysql -u root -p`
Enter password:        ← enter MySQL root account password here
```

当选项文件中存在非空密码时，要明确指定<q>无密码</q>，请在命令行上使用`--skip-password`：

```
$ `mysql --skip-password`
```

###### 注意

从这一点开始，通常我会展示 MySQL 程序的命令而不带连接参数选项。我假设您会提供任何需要的参数，无论是在命令行上还是在选项文件中。

### 保护选项文件免受其他用户的影响

在像 Unix 这样的多用户操作系统上，保护位于您的主目录中的选项文件，以防其他用户阅读它并查找如何使用您的帐户连接到 MySQL。使用*chmod*通过设置其模式使文件私有，仅允许您访问。以下任何一个命令均可实现此目的：

```
$ `chmod 600 .my.cnf`
$ `chmod go-rwx .my.cnf`
```

在 Windows 上，您可以使用 Windows Explorer 设置文件权限。

# 1.5 交互式执行 SQL 语句

## 问题

您已经启动*mysql*。现在您想将 SQL 语句发送到 MySQL 服务器以执行。

## 解决方案

只需输入它们，让*mysql*知道每个语句的结束位置。或者，直接在命令行上指定<q>单行语句</q>。

## 讨论

当您调用*mysql*时，默认情况下会显示一个`mysql>`提示，告诉您它已准备好接收输入。要在`mysql>`提示处执行 SQL 语句，请输入语句，末尾加上分号（`;`）表示语句结束，然后按 Enter 键。必须使用显式语句终止符；*mysql*不会将 Enter 解释为终止符，因为您可以使用多行输入输入语句。分号是最常见的终止符，但您也可以使用`\g`（<q>go</q>）作为分号的同义词。因此，即使这些示例以不同的方式输入和终止，它们发出相同的语句，是等效的：

```
mysql> `SELECT NOW();`
+---------------------+
| NOW()               |
+---------------------+
| 2014-04-06 17:43:52 |
+---------------------+
mysql> `SELECT`
    -> `NOW()\g`
+---------------------+
| NOW()               |
+---------------------+
| 2014-04-06 17:43:57 |
+---------------------+
```

对于第二条语句，*mysql*会将提示从`mysql>`更改为`->`，以示仍在等待语句终止符。

语句终止符`;`和`\g`不是语句本身的一部分。它们是*mysql*程序使用的约定，该程序识别这些终止符并在发送语句到 MySQL 服务器之前将其剥离。

有些语句生成的输出行非常长，在终端上占据多行，这可能使查询结果难以阅读。为避免这个问题，通过使用`\G`而不是`；`或`\g`来终止语句，生成<q>竖直</q>输出。输出将在不同行显示列值：

```
mysql>  `USE cookbook`
mysql> `SHOW FULL COLUMNS FROM limbs LIKE 'thing'\G`
*************************** 1\. row ***************************
     Field: thing
      Type: varchar(20)
 Collation: utf8mb4_0900_ai_ci
      Null: YES
       Key:
   Default: NULL
     Extra:
Privileges: select,insert,update,references
   Comment:
```

要为会话中执行的所有语句生成垂直输出，请使用 `-E`（或 `--vertical`）选项调用 *mysql*。仅为超出终端宽度的结果生成垂直输出，请使用 `--auto-vertical-output`。

要直接从命令行执行语句，请使用 `-e`（或 `--execute`）选项指定它。这对于 <q>一行命令</q> 非常有用。例如，要计算 `limbs` 表中的行数，请使用以下命令：

```
$ `mysql -e "SELECT COUNT(*) FROM limbs" cookbook`
+----------+
| COUNT(*) |
+----------+
|       11 |
+----------+
```

要执行多个语句，请用分号分隔它们：

```
$ `mysql -e "SELECT COUNT(*) FROM limbs;SELECT NOW()" cookbook`
+----------+
| COUNT(*) |
+----------+
|       11 |
+----------+
+---------------------+
| NOW()               |
+---------------------+
| 2014-04-06 17:43:57 |
+---------------------+
```

*mysql* 还可以从文件或另一个程序中读取语句（参见 Recipe 1.6）。

# 1.6 从文件或程序中执行 SQL 语句

## 问题

您希望 *mysql* 读取存储在文件中的语句，而不必手动输入它们。或者您希望 *mysql* 读取来自另一个程序的输出。

## 解决方案

要读取文件，请重定向 *mysql* 的输入，或使用 `source` 命令。要从程序中读取，请使用管道。

## 讨论

默认情况下，*mysql* 程序从终端交互式读取输入，但您可以通过其他输入源（如文件或程序）提供语句。

对于这一目的，MySQL 支持方便地执行一组语句的批处理模式，无需每次手动输入。批处理模式使得可以轻松设置无需用户干预的 *cron* 作业。

要创建一个供 *mysql* 批量执行的 SQL 脚本，请将您的语句放入一个文本文件中。然后调用 *mysql* 并重定向其输入以从该文件中读取：

```
$ `mysql cookbook <` *`file_name`*
```

从输入文件读取的语句会替代您通常手动输入的内容，因此它们必须以 `;`、`\g` 或 `\G` 结束，就像您手动输入它们一样。交互式和批处理模式在默认输出格式上有所不同。对于交互模式，默认格式为表格（带框线）格式。对于批处理模式，默认格式为制表符分隔格式。要覆盖默认设置，请使用适当的命令选项（参见 Recipe 1.7）。

SQL 脚本也非常有用，可用于将一组 SQL 语句分发给其他人。实际上，这就是我们为本书分发 SQL 示例的方式。本书中显示的许多示例可以使用附带的 `recipes` 分发中的脚本文件运行（参见 前言）。将这些文件以批处理模式提供给 *mysql*，避免手动输入语句。例如，当一个配方展示了一个 `CREATE` `TABLE` 语句来定义一个表时，您通常会在 `recipes` 分发的 *tables* 目录中找到一个 SQL 批处理文件，可以用来创建（并可能加载数据到）该表。回顾 Recipe 1.2 显示了用于手动输入的创建和填充 `limbs` 表的语句。该分发的 *recipes* 目录中包含一个 *limbs.sql* 文件，其中包含执行相同操作的语句。文件如下：

```
DROP TABLE IF EXISTS limbs;
CREATE TABLE limbs
(
  thing VARCHAR(20),  # what the thing is
  legs  INT,          # number of legs it has
  arms  INT,          # number of arms it has
  PRIMARY KEY(thing)
);

INSERT INTO limbs (thing,legs,arms) VALUES('human',2,2);
INSERT INTO limbs (thing,legs,arms) VALUES('insect',6,0);
INSERT INTO limbs (thing,legs,arms) VALUES('squid',0,10);
INSERT INTO limbs (thing,legs,arms) VALUES('fish',0,0);
INSERT INTO limbs (thing,legs,arms) VALUES('centipede',99,0);
INSERT INTO limbs (thing,legs,arms) VALUES('table',4,0);
INSERT INTO limbs (thing,legs,arms) VALUES('armchair',4,2);
INSERT INTO limbs (thing,legs,arms) VALUES('phonograph',0,1);
INSERT INTO limbs (thing,legs,arms) VALUES('tripod',3,0);
INSERT INTO limbs (thing,legs,arms) VALUES('Peg Leg Pete',1,2);
INSERT INTO limbs (thing,legs,arms) VALUES('space alien',NULL,NULL);
```

要执行此 SQL 脚本文件中的语句，请切换到`recipes`分发的*tables*目录并运行此命令：

```
$ `mysql cookbook < limbs.sql`
```

您会注意到脚本中包含一个语句，如果存在则删除表，然后重新创建该表并加载数据。这使您可以随时通过再次运行脚本来轻松将其恢复到基线状态，而不必担心对表进行更改。

刚刚显示的命令说明了如何在命令行上为*mysql*指定输入文件。或者，要从*mysql*会话内部的文件中读取一组 SQL 语句，请使用`source` *`filename`*命令（或`\.` *`filename`*，这是同义词）：

```
mysql> `source limbs.sql`
mysql> `\. limbs.sql`
```

SQL 脚本本身可以包含`source`或`\.`命令来包含其他脚本。这为您提供了额外的灵活性，但要注意避免循环。

*mysql*需要读取的文件不一定要手动编写；它可以是程序生成的。例如，*mysqldump*实用程序通过编写一组重新创建数据库的 SQL 语句来生成数据库备份。要重新加载*mysqldump*的输出，请将其输入到*mysql*。例如，您可以通过网络将数据库复制到另一个 MySQL 服务器：

```
$ `mysqldump cookbook > dump.sql`
$ `mysql -h other-host.example.com cookbook < dump.sql`
```

*mysql*还可以读取管道，因此它可以将其他程序的输出作为其输入。任何生成正确终止的 SQL 语句的输出命令都可以用作*mysql*的输入源。可以重新编写转储和重新加载示例，直接使用管道将两个程序连接起来，避免需要中介文件：

```
$ `mysqldump cookbook | mysql -h other-host.example.com cookbook`
```

程序生成的 SQL 还可以用于向表中填充测试数据，而无需手动编写`INSERT`语句。创建一个生成这些语句的程序，然后使用管道将其输出发送到*mysql*：

```
$ `generate-test-data | mysql cookbook`
```

Recipe 6.6 进一步讨论了*mysqldump*。

# 1.7 控制 mysql 输出的目标和格式

## 问题

你希望将*mysql*的输出发送到除了屏幕之外的其他地方。并且你并不一定想要默认的输出格式。

## 解决方案

将输出重定向到文件，或使用管道将输出发送到程序。您还可以控制*mysql*输出的其他方面，以生成表格、制表符分隔、HTML 或 XML 输出；抑制列标题；或使*mysql*更或少冗长。

## 讨论

除非将*mysql*输出发送到其他地方，否则它将显示在屏幕上。要将*mysql*的输出保存到文件中，请使用您的 shell 的重定向功能：

```
$ `mysql cookbook >` *`outputfile`*
```

如果你以重定向输出的方式交互运行*mysql*，你看不到你输入的内容，因此在这种情况下，通常也要从文件（或其他程序）中读取输入：

```
$ `mysql cookbook <` *`inputfile`* `>` *`outputfile`*
```

要将输出发送到另一个程序（例如，解析查询的输出），请使用管道：

```
$ `mysql cookbook <` *`inputfile`* `| sed -e "s/\t/:/g" > outputfile`
```

本节的其余部分展示了如何控制*mysql*的输出格式。

### 生成表格或制表符分隔的输出

*mysql* 根据其运行是否交互式选择默认输出格式。对于交互使用，*mysql* 使用表格（带框线）格式将输出写入终端：

```
$ `mysql cookbook`
mysql> `SELECT * FROM limbs WHERE legs=0;`
+------------+------+------+
| thing      | legs | arms |
+------------+------+------+
| squid      |    0 |   10 |
| fish       |    0 |    0 |
| phonograph |    0 |    1 |
+------------+------+------+
3 rows in set (0.00 sec)
```

对于非交互使用（当输入或输出被重定向时），*mysql* 写入制表符分隔的输出：

```
$ `echo "SELECT * FROM limbs WHERE legs=0" | mysql cookbook`
thing   legs    arms
squid   0       10
fish    0       0
phonograph      0       1
```

要覆盖默认输出格式，请使用适当的命令选项。考虑一个早期显示的 *sed* 命令，并更改其参数以混淆输出：

```
$ `mysql cookbook <` *`inputfile`* `| sed -e "s/table/XXXXX/g"` 
```

```
$ `mysql cookbook -e "SELECT * FROM limbs where legs=4" |` ↩
        `sed -e "s/table/XXXXX/g"`
 thing legs arms 
 XXXXX 4 0 
 armchair 4 2
```

因为 *mysql* 在非交互上下文中运行，它会生成制表符分隔的输出，这可能比表格输出难以阅读。使用 `-t`（或 `--table`）选项生成更易读的表格输出：

```
$ `mysql cookbook -t -e "SELECT * FROM limbs where legs=4" |` ↩
        `sed -e "s/table/XXXXX/g"`

+----------+------+------+
| thing    | legs | arms |
+----------+------+------+
| XXXXX    |    4 |    0 |
| armchair |    4 |    2 |
+----------+------+------+
```

反向操作是在交互模式中生成批量（制表符分隔的）输出。要执行此操作，请使用 `-B` 或 `--batch`。

### 生成 HTML 或 XML 输出

*mysql* 使用 `-H`（或 `--html`）选项从每个查询结果集生成 HTML 表格。这使您可以轻松地生成包含查询结果的网页输出。以下是一个示例（为了更容易阅读，已添加了换行符）：

```
$ `mysql -H -e "SELECT * FROM limbs WHERE legs=0" cookbook`
<TABLE BORDER=1>
<TR><TH>thing</TH><TH>legs</TH><TH>arms</TH></TR>
<TR><TD>squid</TD><TD>0</TD><TD>10</TD></TR>
<TR><TD>fish</TD><TD>0</TD><TD>0</TD></TR>
<TR><TD>phonograph</TD><TD>0</TD><TD>1</TD></TR>
</TABLE>
```

表格的第一行包含列标题。如果不需要标题行，请参阅下一节的说明。

您可以将输出保存到文件，然后使用 Web 浏览器查看。例如，在 Mac OS X 上执行以下操作：

```
$ `mysql -H -e "SELECT * FROM limbs WHERE legs=0" cookbook > limbs.html`
$ `open -a safari limbs.html`
```

要生成一个 XML 文档而不是 HTML，请使用 `-X`（或 `--xml`）选项：

```
$ `mysql -X -e "SELECT * FROM limbs WHERE legs=0" cookbook`
<?xml version="1.0"?>

<resultset statement="select * from limbs where legs=0
">
  <row>
    <field name="thing">squid</field>
    <field name="legs">0</field>
    <field name="arms">10</field>
  </row>

  <row>
    <field name="thing">fish</field>
    <field name="legs">0</field>
    <field name="arms">0</field>
  </row>

  <row>
    <field name="thing">phonograph</field>
    <field name="legs">0</field>
    <field name="arms">1</field>
  </row>
</resultset>
```

您可以通过运行 XSLT 转换来重新格式化 XML 以适应各种用途。这使您可以使用相同的输入生成多种输出格式。

`-H`, `--html` `-X`, 和 `--xml` 选项仅为生成结果集的语句生成输出，不包括 `INSERT` 或 `UPDATE` 等语句。

要编写自己的程序以从查询结果生成 XML，请参阅 Recipe 13.15。

### 抑制查询输出中的列标题

制表符分隔的格式便于生成用于导入其他程序的数据文件。然而，默认情况下，每个查询的输出的第一行默认列出列标题，这可能并不总是您想要的。假设一个名为 *summarize* 的程序为一列数字生成描述性统计信息。如果您从 *mysql* 生成输出以与此程序一起使用，则列标题行会干扰结果，因为 *summarize* 会将其视为数据。要创建仅包含数据值的输出，请使用 `--skip-column-names` 选项抑制标题行：

```
$ `mysql --skip-column-names -e "SELECT arms FROM limbs" cookbook | summarize`
```

指定 <q>silent</q> 选项（`-s` 或 `--silent`）两次可以实现相同效果：

```
$ `mysql -ss -e "SELECT arms FROM limbs" cookbook | summarize`
```

### 指定输出列分隔符

在非交互模式下，*mysql* 通过制表符分隔输出列，并且没有选项来指定输出分隔符。为了生成使用不同分隔符的输出，需要对 *mysql* 输出进行后处理。假设你想创建一个输出文件，以供期望值用冒号字符（`:`）而不是制表符分隔的程序使用。在 Unix 下，可以使用 *tr* 或 *sed* 这样的实用工具将制表符转换为任意分隔符。以下任何一个命令都可以将制表符更改为冒号（*`TAB`* 表示输入制表符的位置）：

```
$ `mysql cookbook <` *`inputfile`*  `| sed -e "s/`*`TAB`*`/:/g" >` *`outputfile`*
$ `mysql cookbook <` *`inputfile`*  `| tr "`*`TAB`*`" ":" >` *`outputfile`*
$ `mysql cookbook <` *`inputfile`*  `| tr "\011" ":" >` *`outputfile`*
```

*tr* 的语法在不同版本之间有所不同；请查阅本地文档。此外，一些 shell 使用制表符字符用于特殊目的，如文件名补全。对于这些 shell，可以通过使用 Ctrl-V 转义字符输入制表符。

*sed* 比 *tr* 更强大，因为它理解正则表达式并允许多个替换。这对于生成像逗号分隔值（CSV）格式的输出非常有用，需要三个替换操作：

1.  如果数据中出现任何引号字符，请通过双引号转义它们，以便在使用生成的 CSV 文件时，它们不会被解释为列分隔符。

1.  将制表符更改为逗号。

1.  用引号括起列值。

*sed* 允许在单个命令行中执行所有三个替换操作：

```
$ `mysql cookbook <` *`inputfile`* `\`
    `| sed -e 's/"/""/g' -e 's/`*`TAB`*`/","/g' -e 's/^/"/' -e 's/$/"/' >` *`outputfile`*
```

至少可以说那太神秘了。你可以用其他更易读的语言达到相同的效果。这里有一个简短的 Perl 脚本，与 *sed* 命令执行相同的操作（将制表符分隔的输入转换为 CSV 输出），并包含注释以记录其工作原理：

```
#!/usr/bin/perl
# csv.pl: convert tab-delimited input to comma-separated values output
while (<>)        # read next input line
{
  s/"/""/g;       # double quotes within column values
  s/\t/","/g;     # put "," between column values
  s/^/"/;         # add " before the first value
  s/$/"/;         # add " after the last value
  print;          # print the result
}
```

如果将脚本命名为 *csv.pl*，可以像这样使用它：

```
$ `mysql cookbook <` *`inputfile`*  `| perl csv.pl >` *`outputfile`*
```

*tr* 和 *sed* 在 Windows 下通常不可用。Perl 可能更适合作为跨平台解决方案，因为它可以在 Unix 和 Windows 下运行（在 Unix 系统上，Perl 通常预安装。在 Windows 上，可以自由安装）。

另一种生成 CSV 输出的方法是使用专为此目的设计的 Perl Text::CSV_XS 模块。实用程序 *cvt_file.pl*，在配方发布中可用，使用此模块构建通用文件重新格式化器。

### 控制 mysql 的详细级别

当你以非交互方式运行 *mysql* 时，不仅默认的输出格式会改变，而且变得更加简洁。例如，*mysql* 不会打印行数或指示语句执行所需的时间。要告诉 *mysql* 更加详细，请使用 `-v` 或 `--verbose`，并多次指定该选项以增加详细程度。尝试以下命令，看看输出如何不同：

```
$ `echo "SELECT NOW()" | mysql`
$ `echo "SELECT NOW()" | mysql -v`
$ `echo "SELECT NOW()" | mysql -vv`
$ `echo "SELECT NOW()" | mysql -vvv`
```

`-v` 和 `--verbose` 的对应项是 `-s` 和 `--silent`，这些选项也可以多次使用以增加效果。

# 1.8 在 SQL 语句中使用用户定义变量

## 问题

你想在一个语句中使用由之前的语句产生的值。

## 解决方案

将值保存在用户定义的变量中以便以后使用。

## 讨论

要保存由 `SELECT` 语句返回的值，请将其赋给用户定义的变量。这使您能够在同一会话中的其他语句中引用它（但不能*跨*会话）。用户变量是标准 SQL 的 MySQL 特定扩展。它们在其他数据库引擎中无法工作。

要在 `SELECT` 语句中给用户变量赋值，使用 `@`*`var_name`* `:=` *`value`* 语法。该变量可以在后续语句中的任何允许表达式的地方使用，例如在 `WHERE` 子句或 `INSERT` 语句中。

下面是一个示例，将值赋给一个用户变量，然后稍后引用该变量。这是确定表中某一行特征值的简单方法，然后选择该特定行的一种方式：

```
mysql> `SELECT MAX(arms+legs) INTO @max_limbs FROM limbs;`
Query OK, 1 row affected (0,01 sec)
mysql> `SELECT * FROM limbs WHERE arms+legs = @max_limbs;`
+-----------+------+------+
| thing     | legs | arms |
+-----------+------+------+
| centipede |   99 |    0 |
+-----------+------+------+
```

另一个变量的用途是在创建具有 `AUTO_INCREMENT` 列的表中的新行之后，保存来自 `LAST_INSERT_ID()` 的结果：

```
mysql> `SELECT @last_id := LAST_INSERT_ID();`
```

`LAST_INSERT_ID()` 返回最近的 `AUTO_INCREMENT` 值。通过将其保存在变量中，您可以在后续语句中多次引用该值，即使您发出其他创建自己的 `AUTO_INCREMENT` 值并因此更改 `LAST_INSERT_ID()` 返回值的语句。Recipe 15.10 进一步讨论了这一技术。

用户变量只保存单个值。如果语句返回多行，则语句将失败并显示错误，但将分配第一行的值：

```
mysql> `SELECT thing FROM limbs WHERE legs = 0;`
+------------+
| thing      |
+------------+
| squid      |
| fish       |
| phonograph |
+------------+
3 rows in set (0,00 sec)

mysql> `SELECT thing INTO @name FROM limbs WHERE legs = 0;`
ERROR 1172 (42000): Result consisted of more than one row
mysql> `SELECT @name;`
+-------+
| @name |
+-------+
| squid |
+-------+
```

如果语句未返回行，则不会进行任何赋值，变量将保留其先前的值。如果之前未使用过该变量，则其值为 `NULL`：

```
mysql> `SELECT thing INTO @name2 FROM limbs WHERE legs < 0;`
Query OK, 0 rows affected, 1 warning (0,00 sec)

mysql> `SHOW WARNINGS;`
+---------+------+-----------------------------------------------------+
| Level   | Code | Message                                             |
+---------+------+-----------------------------------------------------+
| Warning | 1329 | No data - zero rows fetched, selected, or processed |
+---------+------+-----------------------------------------------------+
1 row in set (0,00 sec)

mysql> `select @name2;`
+--------+
| @name2 |
+--------+
| NULL   |
+--------+
1 row in set (0,00 sec)
```

###### 提示

SQL 命令 `SHOW WARNINGS` 返回关于可恢复错误的信息消息，例如将空结果分配给变量或使用不推荐的功能。

要将变量显式设置为特定值，请使用 `SET` 语句。`SET` 语法可以使用 `:=` 或 `=` 作为赋值运算符：

```
mysql> `SET @sum = 4 + 7;`
mysql> `SELECT @sum;`
+------+
| @sum |
+------+
|   11 |
+------+
```

可以将 `SELECT` 结果赋给一个变量，只要将其写成标量子查询（在括号内返回单个值的查询）：

```
mysql> `SET @max_limbs = (SELECT MAX(arms+legs) FROM limbs);`
```

用户变量名称对大小写不敏感：

```
mysql> `SET @x = 1, @X = 2; SELECT @x, @X;`
+------+------+
| @x   | @X   |
+------+------+
| 2    | 2    |
+------+------+
```

用户变量只能出现在允许表达式的地方，不能出现在必须提供常量或文字标识符的地方。试图将变量用于诸如表名之类的东西是很诱人的，但这是行不通的。例如，如果尝试使用变量生成临时表名，如下所示，将会失败：

```
mysql> `SET @tbl_name = CONCAT('tmp_tbl_', CONNECTION_ID());`
mysql> `CREATE TABLE @tbl_name (int_col INT);`
ERROR 1064 (42000): You have an error in your SQL syntax; ↩
check the manual that corresponds to your MySQL server version for ↩
the right syntax to use near '@tbl_name (int_col INT)' at line 1
```

然而，你*可以*生成一个包含`@tbl_name`的准备好的 SQL 语句，然后执行结果。Recipe 6.4 演示了如何实现。

`SET` 也用于将值分配给存储过程参数和局部变量，以及系统变量。有关示例，请参见 Chapter 11 和 Recipe 22.1。

# 1.9 自定义 mysql 提示符

## 问题

你在不同的终端窗口打开了几个连接，并希望能够在视觉上区分它们。

## 解决方案

设置 *mysql* 提示符为自定义值

## 讨论

您可以通过在启动时提供选项 `--prompt` 来自定义 *mysql* 提示符：

```
$ `mysql --prompt="MySQL Cookbook> "`
MySQL Cookbook>
```

如果客户端已经启动，您可以使用命令 *prompt* 以交互方式进行更改：

```
mysql> `prompt MySQL Cookbook>` 
PROMPT set to 'MySQL Cookbook> '
MySQL Cookbook>
```

命令 *prompt*，像其他 *mysql* 命令一样，支持缩写版本：*\R*。

```
mysql> `\R MySQL Cookbook>` 
PROMPT set to 'MySQL Cookbook> '
MySQL Cookbook>
```

要在配置文件中指定提示值，请将选项 *prompt* 放在 `[mysql]` 部分下：

```
[mysql]
prompt="MySQL Cookbook> "
```

引号是可选的，只在需要特殊字符时才需要，例如提示字符串末尾的空格。

最后，您可以使用环境变量 `MYSQL_PS1` 指定提示符：

```
$ `export MYSQL_PS1="MySQL Cookbook> "`
$ `mysql`
MySQL Cookbook>
```

要将提示符重置为默认值，请运行不带参数的命令 *prompt*：

```
MySQL Cookbook> `prompt`
Returning to default PROMPT of mysql> 
mysql>
```

###### 提示

如果使用了 `MYSQL_PS1` 环境变量，则提示符的默认值将是 `MYSQL_PS1` 变量的值，而不是 `mysql`。

*mysql* 提示符是高度可定制的。您可以设置它显示当前日期、时间、用户账号、默认数据库、服务器主机以及关于数据库连接的其他信息。您可以在 [MySQL 用户参考手册](https://dev.mysql.com/doc/refman/8.0/en/mysql-commands.html) 中找到支持选项的完整列表。

要在提示符中显示用户账号，请使用特殊序列 `\u` 显示用户名或 `\U` 显示完整用户账号：

```
mysql> `prompt \U>` 
PROMPT set to '\U> '
cbuser@localhost>
```

如果连接到不同机器上的 MySQL 服务器，您可能希望在提示中看到 MySQL 服务器主机名。有一个特殊序列 `\h` 就是为此而设计的：

```
mysql> `\R \h>` 
PROMPT set to '\h> '
Delly-7390>
```

要在提示符中显示当前默认数据库，请使用特殊序列 `\d`：

```
mysql> `\R \d>` 
PROMPT set to '\d> '
(none)> `use cookbook`
Database changed
cookbook>
```

*mysql* 支持多种选项将时间包含在提示中。您可以包含完整的日期和时间信息，或者只包含部分信息。

```
mysql> `prompt \R:\m:\s>` 
PROMPT set to '\R:\m:\s> '
15:30:10>
15:30:10> `prompt \D>` 
PROMPT set to '\D> '
Sat Sep 19 15:31:19 2020>
```

###### 警告

除非使用完整的当前日期，否则无法指定每月的当前日期。这在 [MySQL Bug #72071](https://bugs.mysql.com/bug.php?id=72071) 中有报告，但仍未修复。

特殊序列可以组合在一起，也可以与任何其他文本一起使用。*mysql* 使用 UTF-8 字符集，如果您的终端也支持 UTF-8，您可以使用笑脸字符使提示更加引人注目。例如，要获取有关连接的用户账号、MySQL 主机、默认数据库和当前时间的信息，您可以将提示设置为 `\u@\h [ߓᜤ] (ߕᜒ:\m:\s)>`：

```
mysql> `prompt \u@\h [ߓᜤ] (ߕᜒ:\m:\s)>` 
PROMPT set to '\u@\h [ߓᜤ] (ߕᜒ:\m:\s)> '
cbuser@Delly-7390 [ߓᣯokbook] (ߕᱶ:15:41)>
```

# 1.10 使用外部程序

## 问题

您希望在不离开 *mysql* 客户端命令提示符的情况下使用外部程序。

## 解决方案

使用 *system* 命令调用程序。

## 讨论

虽然 MySQL 允许为其内部用户账号生成随机密码，但仍然没有内部函数可用于生成所有其他情况下安全用户密码。运行 *system* 命令使用操作系统工具之一：

```
mysql> `system openssl rand -base64 16`
p1+iSG9rveeKc6v0+lFUHA==
```

*\!* 是 *system* 命令的简短版本：

```
mysql> `\! pwgen -synBC 16 1`
Nu=3dWvrH7o_tWiE
```

*pwgen* 可能未安装在您的操作系统上。在运行此示例之前，您需要安装 `pwgen` 软件包。

*system* 是*mysql*客户端的命令，本地执行，使用客户端所属的权限。默认情况下，MySQL 服务器以`mysql`用户身份运行，尽管您可以使用任何用户账户连接。在这种情况下，您只能访问操作系统账户允许的程序和文件。因此，普通用户无法访问属于特殊用户*mysqld*进程运行的数据目录。

```
mysql> `select @@datadir;`
+-----------------+
| @@datadir       |
+-----------------+
| /var/lib/mysql/ |
+-----------------+
1 row in set (0,00 sec)

mysql> `system ls /var/lib/mysql/`
ls: cannot open directory '/var/lib/mysql/': Permission denied
mysql> `\! id`
uid=1000(sveta) gid=1000(sveta) groups=1000(sveta)
```

出于同样的原因，*system*不会在远程服务器上执行任何命令。

您可以使用任何程序，指定选项，重定向输出并将其管道传输到其他命令。您可以从操作系统获得的一个有用的洞察是，*mysqld*进程占用的物理资源量与 MySQL 服务器内部收集的数据进行比较。

MySQL 在[`性能模式`](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-memory-summary-tables.html)中存储关于内存使用情况的信息。它的伴侣[`sys`模式](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-memory-summary-tables.html)包含视图，使您可以轻松访问此信息。特别是，您可以通过查询视图`sys.memory_global_total`找到以人类可读格式显示的分配内存总量。

```
mysql> `SELECT * FROM sys.memory_global_total;`
+-----------------+
| total_allocated |
+-----------------+
| 253.90 MiB      |
+-----------------+
1 row in set (0.00 sec)

mysql> ``\! ps -o rss hp `pidof mysqld` | awk '{print $1/1024}'``
298.66
```

操作系统命令链请求关于操作系统中物理内存使用情况的统计信息，并将其转换为人类可读格式。此示例显示，并非所有分配的内存都在 MySQL 服务器内部仪表化。

请注意，您需要在与 MySQL 服务器相同的计算机上运行*mysql*客户端才能使此功能正常工作。

# 1.11 过滤和处理输出

###### 警告

此方案仅适用于 UNIX 平台！

## 问题

您希望改变 MySQL 客户端的输出格式，超出其内建能力。

## 解决方案

将*pager*设置为一系列命令，以您希望的方式过滤输出。

## 讨论

有时，*mysql*客户端的格式化能力不允许您轻松处理结果集。例如，返回的行数可能过多，无法适应屏幕。或者列数使结果过宽，不便于在屏幕上舒适地阅读。标准操作系统分页程序，如*less*或*more*，使您能够更舒适地处理长宽文本。

在启动*mysql*客户端时，您可以通过提供`--pager`选项或交互式使用*pager*命令及其缩写版本*\P*来指定要使用的分页程序。您可以为分页程序指定任何参数。

要告诉*mysql*使用*less*作为分页程序，请指定`--pager=less`选项或交互式分配此值。像在您喜欢的 shell 中工作时一样提供命令的配置参数。在下面的示例中，我指定了选项`-F`和`-X`，因此当结果集小到可以适应屏幕时，*less*退出并在需要时正常工作。

```
mysql> `pager less -F -X`
PAGER set to 'less -F -X'
mysql> `SELECT * FROM city;`
+----------------+----------------+----------------+
| state          | capital        | largest        |
+----------------+----------------+----------------+
| Alabama        | Montgomery     | Birmingham     |
| Alaska         | Juneau         | Anchorage      |
| Arizona        | Phoenix        | Phoenix        |
| Arkansas       | Little Rock    | Little Rock    |
| California     | Sacramento     | Los Angeles    |
| Colorado       | Denver         | Denver         |
| Connecticut    | Hartford       | Bridgeport     |
| Delaware       | Dover          | Wilmington     |
| Florida        | Tallahassee    | Jacksonville   |
| Georgia        | Atlanta        | Atlanta        |
| Hawaii         | Honolulu       | Honolulu       |
| Idaho          | Boise          | Boise          |
| Illinois       | Springfield    | Chicago        |
| Indiana        | Indianapolis   | Indianapolis   |
| Iowa           | Des Moines     | Des Moines     |
| Kansas         | Topeka         | Wichita        |
| Kentucky       | Frankfort      | Louisville     |
:
mysql> `SELECT * FROM movies;`
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
6 rows in set (0,00 sec)
```

您不仅可以使用 `pager` 美化输出，还可以运行任何能够处理文本的命令。一个常见的用途是在由诊断语句打印的数据中搜索模式，使用 *grep*。例如，要在长长的 `SHOW ENGINE INNODB STATUS` 输出中只查看 `History list length`，使用 *\P grep "History list length"*。完成搜索后，使用空的 *pager* 命令重置 pager，或者指示 *mysql* 禁用 `pager` 并将输出打印到 `STDOUT`，使用 *nopager* 或 *\n*。

```
mysql> `\P grep "History list length"`
PAGER set to 'grep "History list length"'
mysql> `SHOW ENGINE INNODB STATUS\G`
History list length 30
1 row in set (0,00 sec)

mysql> `SELECT SLEEP(60);`
1 row in set (1 min 0,00 sec)

mysql> `SHOW ENGINE INNODB STATUS\G`
History list length 37
1 row in set (0,00 sec)

mysql> `nopager`
PAGER set to stdout
```

在诊断过程中，另一个有用的选项是将输出发送到无处。例如，为了衡量查询的效果，您可能想要检查会话状态变量 `Handler_*`。在这种情况下，您对查询结果不感兴趣，只关注后续诊断命令的输出。甚至更进一步，您可能希望将诊断数据发送给专业的数据库顾问，但又不希望他们看到实际的查询输出，出于安全考虑。在这种情况下，指示 `pager` 使用哈希函数或将输出发送到无处。

```
mysql> `pager md5sum`
PAGER set to 'md5sum'
mysql> `SELECT 'Output of this statement is a hash';`
8d83fa642dbf6a2b7922bcf83bc1d861  -
1 row in set (0,00 sec)

mysql> `pager cat > /dev/null`
PAGER set to 'cat > /dev/null'
mysql> `SELECT 'Output of this statement goes to nowhere';`
1 row in set (0,00 sec)

mysql> `pager`
Default pager wasn't set, using stdout.
mysql> `SELECT 'Output of this statement is visible';`
+-------------------------------------+
| Output of this statement is visible |
+-------------------------------------+
| Output of this statement is visible |
+-------------------------------------+
1 row in set (0,00 sec)
```

###### 提示

要将查询的输出、信息消息和所有键入的命令重定向到文件，请使用 *pager cat > FILENAME*。要将输出重定向到文件并仍然看到输出，请使用重定向到 *tee* 或 *mysql* 自带的 *tee* 命令及其简写 *\T*。内置命令 *tee* 在 UNIX 和 Windows 平台上都可以使用。

可以使用管道（pipes）将 *pager* 命令串联在一起。例如，将 `limbs` 表的内容以不同的字体样式打印出来，设置 *pager* 为一系列调用：

1.  *tr -d ' '* 用于去除额外的空格

1.  *awk -F'|' '{print "+"$2"+\033[3m"$3"\033[0m+\033[1m"$4"\033[0m"$5"+"}'* 用于给文本添加样式

1.  *column -s '+' -t* 用于得到格式良好的输出

```
mysql> `\P tr -d ' ' |` ↩
`awk -F'|' '{print "+"$2"+\033[3m"$3"\033[0m+\033[1m"$4"\033[0m"$5"+"}' |` ↩
`column -s '+' -t`
PAGER set to 'tr -d ' ' | ↩
awk -F'|' '{print "+"$2"+\033[3m"$3"\033[0m+\033[1m"$4"\033[0m"$5"+"}' | ↩
column -s '+' -t'
mysql> `select * from limbs;`

thing       *legs*  arms

human       *2*     2
insect      *6*     0
squid       *0*     10
fish        *0*     0
centipede   *99*    0
table       *4*     0
armchair    *4*     2
phonograph  *0*     1
tripod      *3*     0
PegLegPete  *1*     2
spacealien  *NULL*  NULL

11 rows in set (0,00 sec)
```
