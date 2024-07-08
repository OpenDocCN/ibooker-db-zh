# 第二章：使用 MySQL Shell

# 2.0 简介

我们在第一章讨论了 mysql 客户端程序。MySQL Shell 是现代的替代客户端。除了 SQL 外，它还通过 JavaScript 或 Python 编程接口支持非关系型数据库查询语法，也称为 NoSQL，并提供丰富的功能集来自动化常规任务。

在本章中，我们将讨论如何：

+   连接到 MySQL Shell 并选择正确的协议。

+   选择 SQL、JavaScript 或 Python 接口。

+   使用 SQL 和 NoSQL 语法。

+   控制输出格式。

+   使用 MySQL Shell 内置工具。

+   编写脚本以自动化您的自定义需求。

+   使用管理 API。

+   重复使用您的脚本。

尽管 MySQL Shell 是某些任务的标准工具，但它不包含在 MySQL 软件包中，需要单独安装。您可以从[MySQL Shell 下载页面](https://dev.mysql.com/downloads/shell/)下载它，或者使用操作系统的标准软件包管理器进行安装。本书不涵盖 MySQL Shell 的安装过程，因为它非常简单。

MySQL Shell 的命令名为*mysqlsh*。您可以在终端中输入*mysqlsh*来调用它。

MySQL Shell 支持两种协议：经典的 MySQL 协议（类似于*mysql*客户端使用的协议）和新的 X 协议。X 协议是一种现代协议，通过单独的端口（默认为 33060）与 MySQL 服务器通信。它支持 SQL 和 NoSQL API，并提供异步 API，允许客户端发送多个查询到服务器而无需等待先前查询的结果。使用 X 协议是使用 MySQL Shell 的首选方式，特别是如果您想使用 NoSQL 功能。

# 2.1 使用 MySQL Shell 连接到 MySQL 服务器

## 问题

当您调用*mysqlsh*时，它会打开一个新的会话，但不会连接到任何 MySQL 服务器。

## 解决方案

在 MySQL Shell 内部使用*\connect*命令或在启动时提供您的 MySQL 服务器 URI。

## 讨论

在启动工具后，MySQL Shell 允许您通过命令行参数提供连接选项来连接到 MySQL 服务器。您也可以将默认连接参数放在启动脚本中。

关于连接选项，MySQL Shell 非常灵活。您可以将它们作为 URI 或名称-值对提供，类似于*mysql*客户端接受的方式。

URI 使用以下格式：

```
[scheme://][user[:password]@]<host[:port]|socket>[/schema]↩
[?option=value&option=value...]
```

参数的含义在表格 2-1 中有解释。

表格 2-1\. URI 连接选项

| 参数 | 解释 | 默认值 |
| --- | --- | --- |
| `scheme` | 要使用的协议。可以是`mysql`（如果要使用经典协议）或`mysqlx`（如果要使用 X 协议）。 | `mysqlx` |
| `user` | 要连接的用户名。 | 您的操作系统帐户。 |
| `password` | 密码 | 请求密码。 |
| `host` | 要连接的主机。 | 没有默认值。这是唯一的**必需**参数，除非指定了`socket`选项。 |
| `port` | 要连接的端口。 | 经典协议为 3306，X 协议为 33060。 |
| `socket` | 用于本地主机连接的套接字。 | 您必须提供此参数或`host`参数。 |
| `schema` | 要连接的数据库模式。 | 无值。不要选择任何模式。 |
| `option` | 您想要使用的任何其他选项。 | 无值。选择任何或不选择任何选项。 |

因此，要使用交互界面连接到本地机器上的 MySQL 服务器，请键入 *\connect 127.0.0.1*：

```
 MySQL  localhost  JS > `\connect 127.0.0.1`
Creating a session to 'sveta@127.0.0.1'
Please provide the password for 'sveta@127.0.0.1': 
Save password for 'sveta@127.0.0.1'? [Y]es/[N]o/Ne[v]er (default No): 
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 1066144 (X protocol)
Server version: 8.0.27 MySQL Community Server - GPL
No default schema selected; type \use <schema> to set one.
```

这将创建一个使用 X 协议的连接。

###### 注意

在不指定用户名连接时，MySQL Shell 使用操作系统登录。这就是为什么连接是为用户`sveta`创建的，而不是我们在本书中到处使用的用户`cbuser`。我们将在稍后讨论如何在连接时指定 MySQL 用户帐户。

要退出 MySQL Shell 会话，请使用命令 `\exit` 或 `\quit` 及其简写形式 `\q`。

```
 MySQL  JS > `\exit`
Bye!
```

要使用套接字进行交互连接，请键入 *\c (/var/run/mysqld/mysqld.sock)*：

```
 MySQL  127.0.0.1:33060+ ssl  JS > `\c (/var/run/mysqld/mysqld.sock)`
Creating a session to 'sveta@/var%2Frun%2Fmysqld%2Fmysqld.sock'
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 1067565
Server version: 8.0.27 MySQL Community Server - GPL
No default schema selected; type \use <schema> to set one.
```

这将创建使用经典协议的连接。如果要使用 X 协议通过套接字连接，请使用 `mysqlx_socket`。如果运行查询，您将找到`mysqlx_socket`的值。

```
mysql> `SELECT` `@``@``mysqlx_socket``;`
+-----------------------------+ | @@mysqlx_socket             |
+-----------------------------+ | /var/run/mysqld/mysqlx.sock |
+-----------------------------+ 1 row in set (0,00 sec)
```

命令 *\connect* 有一个更短的版本 *\c*，我们在通过套接字连接的示例中使用了它。请注意命令参数中的括号。如果没有括号，命令将因语法错误而失败。或者，您可以用其 URI 编码值 `%2F` 替换所有后续的斜杠符号：

```
 MySQL  localhost  JS > `\connect /var%2Frun%2Fmysqld%2Fmysqld.sock`
Creating a session to 'sveta@/var%2Frun%2Fmysqld%2Fmysqld.sock'
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 1073606
Server version: 8.0.27 MySQL Community Server - GPL
No default schema selected; type \use <schema> to set one.
```

在打开 MySQL Shell 会话时使用 URI 进行连接，使用以下命令：

```
$ `mysqlsh mysqlx://cbuser:cbpass@127.0.0.1/cookbook`
Please provide the password for 'cbuser@127.0.0.1:33060': ******
Save password for 'cbuser@127.0.0.1:33060'? [Y]es/[N]o/Ne[v]er (default No): 
MySQL Shell 8.0.27

Copyright (c) 2016, 2021, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.
Creating a session to 'cbuser@127.0.0.1:33060/cookbook'
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 1076096 (X protocol)
Server version: 8.0.27 MySQL Community Server - GPL
Default schema `cookbook` accessible through db.
 MySQL  127.0.0.1:33060+ ssl  cookbook  JS >
```

在这种情况下，我们通过命令行指定了用户名和密码，并选择了`cookbook`作为默认数据库。

在调用 *mysqlsh* 命令时连接时，您还可以单独指定连接凭据，类似于使用 *mysql* 客户端连接时的方式。

```
$ `mysqlsh --host=127.0.0.1 --port=33060 --user=cbuser --schema=cookbook`
Please provide the password for 'cbuser@127.0.0.1:33060': ******
MySQL Shell 8.0.22

Copyright (c) 2016, 2020, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.
Creating a session to 'cbuser@127.0.0.1:33060/cookbook'
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 8738 (X protocol)
Server version: 8.0.22-13 Percona Server (GPL), Release '13', Revision '6f7822f'
Default schema `cookbook` accessible through db.
 MySQL  127.0.0.1:33060+ ssl  cookbook  JS >
```

如果要指定默认模式，请将其作为参数传递给配置选项`schema`。否则，*mysqlsh* 将将其视为主机名并因错误而失败。

在 MySQL Shell 中，您还可以通过命名参数指定选项。首先，您需要创建一个包含连接参数的字典，然后将其作为选项传递给内置的自动创建的`shell`对象的*connect()*方法。

```
 MySQL  127.0.0.1:33060+ ssl  JS > `connectionData={`
                                -> `"host": "127.0.0.1",`
                                -> `"user": "cbuser",`
                                -> `"schema": "cookbook"`
                                -> `}`
                                -> 
{
    "host": "127.0.0.1", 
    "schema": "cookbook", 
    "user": "cbuser"
}
 MySQL  127.0.0.1:33060+ ssl  JS > `shell.connect(connectionData)`
Creating a session to 'cbuser@127.0.0.1/cookbook'
Please provide the password for 'cbuser@127.0.0.1': ******
Save password for 'cbuser@127.0.0.1'? [Y]es/[N]o/Ne[v]er (default No): 
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 1077318 (X protocol)
Server version: 8.0.27 MySQL Community Server - GPL
Default schema `cookbook` accessible through db.
<Session:cbuser@127.0.0.1:33060>
 MySQL  127.0.0.1:33060+ ssl  cookbook  JS >
```

## 参见

有关如何通过 MySQL Shell 连接到 MySQL 服务器的更多信息，请参阅[MySQL Shell Connections](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-connections.html)。

# 2.2 选择协议

## 问题

您不想使用 MySQL Shell 的默认设置，并且希望自己选择 X 协议或经典协议。

## 解决方案

要选择 X 协议：使用选项之一 `mysqlx`、`mx`、`sqlx`。要选择经典协议：使用选项之一 `mysql`、`mc` 和 `sqlc`。

## 讨论

MySQL Shell 使用连接选项和服务器响应自动选择协议。如果未使用端口或套接字选项，则尝试使用 X 协议的默认端口或套接字。如果不可用，则默认使用经典协议。如果这不是预期的行为或者您想要显式控制使用哪种协议，可以在启动 *mysqlsh* 客户端时通过传递选项 `mysqlx`、`mx` 和 `sqlx` 来选择 X 协议，并通过选项 `mysql`、`mc` 和 `sqlc` 来选择经典协议。

```
$ `mysqlsh --host=127.0.0.1  --user=cbuser --schema=cookbook --mysqlx`
Please provide the password for 'cbuser@127.0.0.1': ******
MySQL Shell 8.0.22
...
Your MySQL connection id is 9143 (X protocol)

$ `mysqlsh --host=127.0.0.1  --user=cbuser --schema=cookbook --mysql`
Please provide the password for 'cbuser@127.0.0.1': ******
MySQL Shell 8.0.22
...
Creating a Classic session to 'cbuser@127.0.0.1/cookbook'
```

在 MySQL Shell 内部，当打开新连接时，通过将选项传递给 `connectionData` 字典的 `scheme` 键来指定值：

```
 MySQL  127.0.0.1:3306 ssl  cookbook  JS > `connectionData={` 
                                        -> `"scheme": "mysql", "host": "127.0.0.1",` 
                                        -> `"user": "cbuser", "schema": "cookbook"` 
                                        -> `}`
{
    "host": "127.0.0.1", 
    "schema": "cookbook", 
    "scheme": "mysql", 
    "user": "cbuser"
}
 MySQL  127.0.0.1:3306 ssl  cookbook  JS > `shell.connect(connectionData, "cbpass")`
Creating a Classic session to 'cbuser@127.0.0.1/cookbook'
```

在指定 URI 时，可以通过 `scheme` 前缀连接选项：

```
mysqlsh mysqlx://cbuser:cbpass@127.0.0.1/cookbook
```

```
\c mysql://cbuser:cbpass@127.0.0.1/cookbook
```

如果指定的协议无法使用，MySQL Shell 将失败并显示错误：

```
 MySQL  JS > `\c mysql://cbuser:cbpass@127.0.0.1:33060/cookbook`
Creating a Classic session to 'cbuser@127.0.0.1:33060/cookbook'
MySQL Error 2007 (HY000): Protocol mismatch; server version = 11, client version = 10
```

```
$ `mysqlsh --host=127.0.0.1 --port=3306 --user=cbuser --schema=cookbook --mx`
Please provide the password for 'cbuser@127.0.0.1:3306': ******
MySQL Shell 8.0.22

Copyright (c) 2016, 2020, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.
Creating an X protocol session to 'cbuser@127.0.0.1:3306/cookbook' ↩
MySQL Error 2027: Requested session assumes MySQL X Protocol but '127.0.0.1:3306' ↩
seems to speak the classic MySQL protocol
(Unexpected response received from server, msg-id:10)
```

您可以通过运行命令 *shell.status()* 查找当前 MySQL Shell 连接的详细信息：

```
 MySQL  127.0.0.1:33060+ ssl  cookbook  JS > `shell.status()`
MySQL Shell version 8.0.22

Connection Id:                61
Default schema:               cookbook
Current schema:               cookbook
Current user:                 cbuser@localhost
SSL:                          Cipher in use: TLS_AES_256_GCM_SHA384 TLSv1.3
Using delimiter:              ;
Server version:               8.0.22-13 Percona Server (GPL), Release '13', ↩
                              Revision '6f7822f'
Protocol version:             X protocol
Client library:               8.0.22
Connection:                   127.0.0.1 via TCP/IP
TCP port:                     33060
Server characterset:          utf8mb4
Schema characterset:          utf8mb4
Client characterset:          utf8mb4
Conn. characterset:           utf8mb4
Result characterset:          utf8mb4
Compression:                  Enabled (DEFLATE_STREAM)
Uptime:                       4 min 57.0000 sec
```

###### Tip

MySQL Shell 允许像 MySQL CLI 一样自定义其提示符。要实现这一点，需要编辑位于 MySQL Shell 配置主目录中的 `prompt.json` 文件。这是一个 JSON 格式的文件。MySQL Shell 提供了大量自定义提示符模板和解释如何修改提示的 `README.prompt` 文件。

我们不会详细介绍如何自定义 MySQL Shell 用户提示符，但会从默认提示中移除主机、端口和协议信息，这样我们的示例在书中占用的空间将更少。

MySQL Shell 的配置主目录在 Unix 上是 `~/.mysqlsh/`，在 Windows 上是 `%AppData%\MySQL\mysqlsh\`。如果设置了变量 `MYSQLSH_USER_CONFIG_HOME`，可以覆盖此位置。`README.prompt` 和示例位于 MySQL Shell 安装根目录下的 `share/mysqlsh/prompt/` 目录中。

## See Also

获取关于 *mysqlsh* 命令选项的更多信息，请参阅 [A.1 mysqlsh — The MySQL Shell](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysqlsh.html)。

# 2.3 选择 SQL、JavaScript 或 Python 模式

## Problem

MySQL Shell 启动时选择了错误的模式，您希望选择与默认模式不同的模式。

## 解决方案

在启动 *mysqlsh* 后，可以使用 `sql`、`js` 或 `py` 选项或切换模式。

## 讨论

默认情况下，MySQL Shell 以 JavaScript 模式启动。您可以通过查看提示字符串来确认：

```
 MySQL  cookbook  JS >
```

如果启动工具时使用 `--sql` 选项选择 SQL 模式或 `--py` 选项选择 Python 模式，则可以更改默认模式。要显式选择 JavaScript 模式，请使用 `--js` 选项。您将看到 MySQL Shell 客户端的提示消息会更改为所选模式。在此处，我们选择 Python 模式。

```
$ `mysqlsh cbuser:cbpass@127.0.0.1/cookbook --py`
...
Default schema `cookbook` accessible through db.
 MySQL  cookbook  Py >
```

###### Tip

对于 SQL 模式，您可以明确指示工具使用所需的模式，并使用选项 `sqlx` 选择 X 协议以及选项 `sqlc` 选择经典协议。当您通过默认 TCP/IP 端口连接时，这可能非常方便。

在*mysqlsh*中，您可以使用命令*\js*、*\py*和*\sql*更改处理模式，分别切换到 JavaScript、Python 和 SQL 模式。

```
 MySQL  cookbook  SQL > `\js`
Switching to JavaScript mode...
 MySQL  cookbook  JS > `\py`
Switching to Python mode...
 MySQL  cookbook  Py > `\sql`
Switching to SQL mode... Commands end with ;
 MySQL  cookbook  SQL >
```

# 2.4 运行 SQL 会话

## 问题

您希望具有类似*mysql*客户端的功能，但又不想离开 MySQL Shell。

## 解决方案

使用 SQL 模式。

## 讨论

使用 SQL 模式，MySQL Shell 的行为与我们在第一章中描述的*mysql*客户端完全相同。您可以运行查询，使用*\pager*命令控制输出，在系统编辑器中使用*\edit*命令编辑 SQL，在文件中使用*\source*命令执行 SQL，并使用*\system*执行系统 shell 命令。您可以查看和编辑命令行历史记录。

没有*\d*作为*delimiter*命令的快捷方式，但命令本身是有效的。

```
 MySQL  cookbook  SQL > `delimiter |`
 MySQL  cookbook  SQL > `CREATE PROCEDURE get_client_info()`
                     -> `BEGIN`
                     -> `SELECT GROUP_CONCAT(ATTR_NAME, '=', ATTR_VALUE)` 
                     -> `FROM performance_schema.session_account_connect_attrs` 
                     -> `WHERE ATTR_NAME IN ('_client_name', '_client_version');`
                     -> `END`
                     -> `|`
Query OK, 0 rows affected (0.0163 sec)
 MySQL  cookbook  SQL > `delimiter ;`
 MySQL  cookbook  SQL > `CALL get_client_info();`
+----------------------------------------------+
| GROUP_CONCAT(ATTR_NAME, '=', ATTR_VALUE)     |
+----------------------------------------------+
| _client_name=libmysql,_client_version=8.0.22 |
+----------------------------------------------+
1 row in set (0.0017 sec)

Query OK, 0 rows affected (0.0017 sec)
```

没有*tee*命令。如果要将查询结果记录到文件中，请将分页器设置为`tee -a <DESIRED LOG FILE LOCATION>`。但它不会记录 SQL 语句，它们仅在历史文件中可用。

###### 提示

默认情况下，MySQL Shell 不会在客户端会话之间保存历史记录。这意味着一旦退出 Shell，您无法访问先前的命令。如果启用选项`history.autoSave`，可以覆盖此行为：

```
 MySQL  JS > \option --persist history.autoSave=1
```

# 2.5 在 JavaScript 模式中运行 SQL

## 问题

您处于 JavaScript 模式，但想执行传统 SQL

## 解决方案

使用命令*\sql*，或使用属于`Session`类的`sql()`和`runSQL()`方法，在 JavaScript 模式中执行传统 SQL。

## 讨论

JavaScript 模式支持面向对象的数据库查询风格。或者，您可以运行纯 SQL。

如果您想要在不离开 JavaScript 模式的情况下运行单个 SQL 语句并获取结果，可以使用命令*\sql*。在下面的示例中，我们运行了一条普通 SQL 语句，从表`limbs`中选择具有两个或更多手臂的数据：

```
 MySQL  cookbook  JS > `\sql SELECT * FROM limbs WHERE arms >=2 ORDER BY arms;`
Fetching table and column names from `cookbook` for auto-completion...↩
Press ^C to stop.
+--------------+------+------+
| thing        | legs | arms |
+--------------+------+------+
| human        |    2 |    2 |
| armchair     |    4 |    2 |
| Peg Leg Pete |    1 |    2 |
| squid        |    0 |   10 |
+--------------+------+------+
4 rows in set (0.0002 sec)
```

以面向对象的方式运行 SQL 更加灵活，并提供更多选项。要运行单个语句，请使用`Session`类的*runSQL*方法。

```
 MySQL  cookbook  JS > `session.runSql(                      > "SELECT * FROM limbs WHERE arms >=2 ORDER BY arms")`
+--------------+------+------+
| thing        | legs | arms |
+--------------+------+------+
| human        |    2 |    2 |
| armchair     |    4 |    2 |
| Peg Leg Pete |    1 |    2 |
| squid        |    0 |   10 |
+--------------+------+------+
4 rows in set (0.0014 sec)
```

###### 提示

当连接到 MySQL Shell 时，它会创建`Session`类的默认实例。可以通过全局对象`session`访问它。

方法*runSQL*支持占位符：只需用`?`符号替换变量值，并将参数作为数组传递。

```
 MySQL  cookbook  JS > `session.runSql("SELECT * FROM limbs ↩         WHERE arms >= ? AND legs != ? ↩         ORDER BY arms", [2, 0])`
+--------------+------+------+
| thing        | legs | arms |
+--------------+------+------+
| human        |    2 |    2 |
| armchair     |    4 |    2 |
| Peg Leg Pete |    1 |    2 |
+--------------+------+------+
3 rows in set (0.0005 sec)
```

您可以将此方法与标准 JavaScript 语法结合，创建一个能够执行更多操作而不仅限于运行 SQL 查询的脚本：

```
 MySQL  cookbook  JS > `for (i = 1;` 
                    ->  `i <= session.sql("SELECT MAX(arms) AS maxarms FROM limbs").` ![1](img/1.png)
                    ->  `execute().fetchOne().` ![2](img/2.png)
                    ->  `getField('maxarms');` ![3](img/3.png)
                    ->  `i++)`  ![4](img/4.png)
                    -> `{`
                    ->   `species=session.sql("SELECT COUNT(*) AS countarms \`
                    ->    `FROM limbs WHERE arms =?").`
                    ->    `bind(i).execute();` ![5](img/5.png)
                    ->   `if (species.hasData() && (armscount = species.fetchOne().`
                    ->    `getField('countarms')) > 0 )` ![6](img/6.png)
                    ->   `{`
                    ->     `print("We have " + armscount + " species with " + i +` ![7](img/7.png)
                    ->     `(i == 1 ? " arm\n" : " arms\n"));`
                    ->   `}`
                    -> `}`
                    -> 
We have 1 species with 1 arm
We have 3 species with 2 arms
We have 1 species with 10 arms
```

![1](img/#co_select_max_co)

选择存储在`limbs`表中的最大手臂数。

![2](img/#co_fetch_one_co)

方法*session.sql.execute()*返回一个`SqlResult`对象，其中有一个名为*fetchOne*的方法，返回结果集的第一行。

![3](img/#co_get_field_co)

由于我们的查询应返回一行，我们没有遍历结果集，而只是调用了一个名为*getField*的方法，该方法以列名或其别名作为参数，以获取存储在`limbs`表中的最大手臂数。

![4](img/#co_for_loop_co)

我们将此数字用作*for*循环的停止条件。

![5](img/#co_bind_co)

在循环中，我们执行了查询以获取指定手臂数量的物种数量。我们使用了*sql*方法及其*bind*方法，将循环迭代器`i`的值绑定到查询中。

![6](img/#co_result_co)

检查是否收到了结果，以及手臂数量是否大于 0。

![7](img/#co_print_co)

如果两个条件都为真：打印结果。

###### 注意

当您单独执行*sql*或*runSQL*方法时，MySQL Shell 会自动为它们调用*execute*方法。但是，如果在更复杂的代码中使用这些方法，如在循环或多语句块中，您需要显式调用*execute*方法。否则，只有最后一个语句会被执行，而所有之前的调用将被忽略。

## 参见

有关 MySQL Shell API 的更多信息，请参阅[高级 MySQL 用户参考手册中的 ShellAPI](https://dev.mysql.com/doc/dev/mysqlsh-api-javascript/8.0/group___shell_a_p_i.html)。

# 2.6 在 Python 模式下运行 SQL

## 问题

您正在 Python 模式下，但希望执行传统 SQL。

## 解决方案

使用命令*\sql*或类`Session`的方法`sql`、`run_sql`。

## 讨论

就像我们在 JavaScript 模式中看到的那样，Python 模式也支持*\sql*命令。如果您要执行 SQL 语句而不想处理其结果，可以使用它。以下代码从表`movies`中选择所有行。

```
 MySQL  cookbook  Py > `\sql SELECT * FROM movies;`
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
6 rows in set (0.0008 sec)
```

Python 模式中的方法名称与 JavaScript 模式稍有不同。因此，要运行 SQL 语句，请使用`Session`对象并将参数绑定为数组，使用方法*run_sql*。

```
 MySQL  cookbook  Py > `session.run_sql("SELECT * FROM movies WHERE year < ?",[2000])`
+----+------+--------------------+
| id | year | movie              |
+----+------+--------------------+
|  1 | 1997 | The Fifth Element  |
|  2 | 1999 | The Phantom Menace |
+----+------+--------------------+
2 rows in set (0.0009 sec)
```

此示例选择了所有在 2000 年之前创建的电影。

您可以使用 Python 或 JavaScript 进行编程。例如，如果要知道每位演员出演的电影数量以及出演年份，请将表`movies`与表`movies_actors`连接，然后使用 Python 代码打印结果。

```
 MySQL  cookbook  Py > `myres=session.sql("SELECT actor, COUNT(movie) as movies,`↩ ![1](img/1.png)
                           `GROUP_CONCAT(year SEPARATOR ', ') AS years_string,`↩
                          `COUNT(year) AS years FROM movies_actors` ↩
                           `GROUP BY actor ORDER BY movies DESC").`↩
                           `execute().fetch_all()`
 MySQL  cookbook  Py > `for myrow in myres:` ![2](img/2.png)
                    -> `print(myrow[0] + " was featured in " + str(myrow[1]) +`↩ ![3](img/3.png)
                      `(" movies" if (myrow[1] > 1) else " movie") +` ↩
                      `" in " + ("years " if (myrow[3] > 1) else "the year ") +`↩
                      `myrow[2] + ".")`
                    -> 
Liam Neeson was featured in 3 movies in years 2005, 1999, 2011.
Bruce Willis was featured in 2 movies in years 1997, 2010.
Ian Holm was featured in 2 movies in years 1997, 2001.
Orlando Bloom was featured in 2 movies in years 2005, 2001.
Diane Kruger was featured in 1 movie in the year 2011.
Elijah Wood was featured in 1 movie in the year 2001.
Ewan McGregor was featured in 1 movie in the year 1999.
Gary Oldman was featured in 1 movie in the year 1997.
Helen Mirren was featured in 1 movie in the year 2010.
Ian McKellen was featured in 1 movie in the year 2001.
```

![1](img/#co_query_co)

运行查询并将其返回的所有行提取到变量`myres`中。

![2](img/#co_loop_co)

在*for ... in*循环中遍历此变量。

![3](img/#co_print_py_co)

打印结果。

###### 提示

如果您对查询语法尚不熟悉，不用担心：我们将在第五章讨论数据查询的方法，以及在配方 16.0 中如何连接两个或多个表。

## 参见

若要获取有关 Python MySQL Shell API 的额外信息，请在 Python shell 会话内使用命令*\? mysqlx*。

# 2.7 在 JavaScript 模式下使用表格

## 问题

您希望在 JavaScript 模式中使用面向对象的方式查询您的表。

## 解决方案

使用方法*getTable*选择表，然后使用方法*select*、*count*、*insert*、*update*、*delete*从表中选择、检索行数、插入、更新或删除。

## 讨论

MySQL Shell 支持面向对象的语法来查询和修改数据库对象。因此，要从表`limbs`中选择所有行，我们可以使用*select*方法。

```
 MySQL  cookbook  JS > `session.getDefaultSchema().getTable('limbs').select()`
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
11 rows in set (0.0003 sec)
```

在上面的清单中，我们首先使用*getDefaultSchema*方法选择模式，然后用*getTable*方法选择表，最后用*select*检索所有行。

方法*select*返回`TableSelect`对象，支持允许指定`WHERE`条件、`ORDER BY`、`GROUP BY`子句和 SQL `SELECT`具有的其他功能的方法。它还支持准备语句和参数绑定。因此，要仅选择表`limbs`中具有四条或更多腿的物种并按腿数排序，请尝试以下代码：

```
 MySQL  cookbook  JS > `session.getDefaultSchema().getTable('limbs').select().`
                    -> `where('legs >= :legs').orderBy('legs').bind('legs', 4)`
+-----------+------+------+
| thing     | legs | arms |
+-----------+------+------+
| table     |    4 |    0 |
| armchair  |    4 |    2 |
| insect    |    6 |    0 |
| centipede |   99 |    0 |
+-----------+------+------+
4 rows in set (0.0004 sec)
```

###### 警告

请注意，这里我们使用了命名参数作为占位符，而不是像在 SQL 查询数据库时使用问号。

MySQL Shell API 还支持以面向对象的方式插入、更新和删除数据，以及启动和完成事务。例如，如果我们想在不实际修改数据的情况下对`cookbook`数据库进行实验，我们可以在事务内进行。

```
 MySQL  cookbook  JS > `limbs = session.getDefaultSchema().getTable('limbs')` ![1](img/1.png)
<Table:limbs>
 MySQL  cookbook  JS > `session.startTransaction()` ![2](img/2.png)
Query OK, 0 rows affected (0.0006 sec)
 MySQL  cookbook  JS > `limbs.insert('thing', 'legs', 'arms').` ![3](img/3.png)
                    -> `values('cat', 4, 0).`
                    -> `values('dog', 2, 2)`
                    -> 
Query OK, 2 items affected (0.0012 sec)

Records: 2  Duplicates: 0  Warnings: 0
 MySQL  cookbook  JS > `limbs.count()` ![4](img/4.png)
13
 MySQL  cookbook  JS > `limbs.update().set('legs', 4).set('arms', 0).`
                    -> `where("thing='dog'")` ![5](img/5.png)
Query OK, 1 item affected (0.0012 sec)

Rows matched: 1  Changed: 1  Warnings: 0
 MySQL  cookbook  JS > `limbs.select().where("thing='dog'")` ![6](img/6.png)
+-------+------+------+
| thing | legs | arms |
+-------+------+------+
| dog   |    4 |    0 |
+-------+------+------+
1 row in set (0.0004 sec)
 MySQL  cookbook  JS > `limbs.delete().where("thing='cat'")` ![7](img/7.png)
Query OK, 1 item affected (0.0010 sec)
 MySQL  cookbook  JS > `limbs.count()` ![8](img/8.png)
12
 MySQL  cookbook  JS > `session.rollback()` ![9](img/9.png)
Query OK, 0 rows affected (0.0054 sec)
 MySQL  cookbook  JS > `limbs.count()` ![10](img/10.png)
11
 MySQL  cookbook  JS > `limbs.select().where("thing='dog' or thing='cat'")`
Empty set (0.0010 sec)
```

![1](img/#co_jst_limbs_co)

将表`limbs`的表对象保存到变量`limbs`中。

![2](img/#co_jst_transaction_co)

开始一个事务，这样我们就可以回滚我们的实验。

![3](img/#co_jst_insert_co)

使用*insert*方法将两行插入表`limbs`，该方法将列列表作为参数，并将要插入的值列表作为参数。

![4](img/#co_jst_count_co)

检查现在表中存在的行数。

![5](img/#co_jst_update_co)

如果回顾我们插入的行，您可能会注意到一个错误。实际上，狗有四条腿，而不是两条腿和两只手臂。要纠正这个错误，请调用*update*方法。

![6](img/#co_jst_select_co)

跟随*select*调用确认我们的更改已应用于表`limbs`。

![7](img/#co_jst_delete_co)

我们发现猫和狗并不总是彼此的朋友，于是用*delete*方法从桌子上移除了一只猫。

![8](img/#co_jst_count2_co)

确认猫已成功移除。

![9](img/#co_jst_rollback_co)

回滚事务以将表恢复到初始状态。

![10](img/#co_jst_confirm_co)

方法*count*和*select*确认表处于初始状态。

###### 注意

由于我们在交互式会话中逐条执行了所有语句，我们省略了*execute*方法。如果您在循环或程序脚本中执行 SQL 命令，则需要该方法。

## 参见

要了解如何在 JavaScript 模式下处理表的更多信息，请参见[对象`Table`的用户参考手册](https://dev.mysql.com/doc/dev/mysqlsh-api-javascript/8.0/classmysqlsh_1_1mysqlx_1_1_table.html)。

# 2.8 在 Python 模式下处理表

## 问题

您的数据库中有表格，并希望在 Python 模式下使用它们。

## 解决方案

使用方法 *get_table* 获取表对象，然后使用方法 *select*、*count*、*insert*、*update*、*delete* 来从表中选择、检索行数、插入、更新或删除。

## 讨论

像 JavaScript 一样，Python 支持以面向对象的方式处理表格。因此，要从表 `movies` 中选择所有行，请尝试类 `Table` 的方法 *select*：

```
 MySQL  cookbook  Py > `session.get_schema('cookbook').get_table('movies').select()` 
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
6 rows in set (0.0003 sec)
```

在这个示例中，我们使用了方法 *get_schema*，允许我们选择数据库中存储的任何模式。

Python 模式支持方法，允许修改表中的数据以及事务语句。

为了我们的示例，我们将表 `movies` 和 `movies_actors` 保存到变量中：

```
 MySQL  cookbook  Py > `movies=session.get_schema('cookbook').get_table('movies')`
 MySQL  cookbook  Py > `movies_actors=session.get_schema('cookbook').`↩
                          `get_table('movies_actors')`
```

然后，我们将开启一个事务，以便我们的更改将应用到两个表或完全不应用，然后我们将插入一部由 Gary Oldman 主演的电影 *“最黑暗的时刻”*。最后，我们会提交事务：

```
 MySQL  cookbook  Py > `session.start_transaction()`
Query OK, 0 rows affected (0.0003 sec)
 MySQL  cookbook  Py > `movies.insert('year', 'movie').`↩
                          `values(2017, 'Darkest Hour')`
Query OK, 1 item affected (0.0013 sec)
 MySQL  cookbook  Py > `movies_actors.insert().`↩
                          `values(1997, 'Darkest Hour', 'Gary Oldman')`
Query OK, 1 item affected (0.0011 sec)
 MySQL  cookbook  Py > `session.commit()`
Query OK, 0 rows affected (0.0075 sec)
```

要找出所有由 Gary Oldman 主演的电影，我们将使用 SQL 查询，因为 X API 不支持连接：

```
 MySQL  cookbook  Py > `session.sql("SELECT * FROM movies` ↩
                          `JOIN movies_actors USING(movie) WHERE actor = 'Gary Oldman'")`
+-------------------+----+------+------+-------------+
| movie             | id | year | year | actor       |
+-------------------+----+------+------+-------------+
| The Fifth Element |  1 | 1997 | 1997 | Gary Oldman |
| Darkest Hour      |  7 | 2017 | 1997 | Gary Oldman |
+-------------------+----+------+------+-------------+
2 rows in set (0.0012 sec)
```

哎呀！电影 *“最黑暗的时刻”* 的年份在一个表中不正确。让我们更新它：

```
 MySQL  cookbook  Py > `session.start_transaction()` ![1](img/1.png)
Query OK, 0 rows affected (0.0007 sec)
 MySQL  cookbook  Py > `movies.update().set('year', 2017).where("movie='Darkest Hour'")` ![2](img/2.png)
Query OK, 0 items affected (0.0013 sec)

Rows matched: 1  Changed: 0  Warnings: 0
 MySQL  cookbook  Py > `movies_actors.update().set('year', 2017).`↩ ![3](img/3.png)
 `where("movie='Darkest Hour'")`
Query OK, 1 item affected (0.0012 sec)

Rows matched: 1  Changed: 1  Warnings: 0
 MySQL  cookbook  Py > `session.commit()` ![4](img/4.png)
Query OK, 0 rows affected (0.0073 sec)
 MySQL  cookbook  Py > `session.run_sql("SELECT * FROM movies JOIN movies_actors`↩ ![5](img/5.png)
                           `USING(movie) WHERE actor = 'Gary Oldman'")`
+-------------------+----+------+------+-------------+
| movie             | id | year | year | actor       |
+-------------------+----+------+------+-------------+
| The Fifth Element |  1 | 1997 | 1997 | Gary Oldman |
| Darkest Hour      |  7 | 2017 | 2017 | Gary Oldman |
+-------------------+----+------+------+-------------+
2 rows in set (0.0005 sec)
```

![1](img/#co_pyt_begin_co)

开始一个事务，以便我们要么更新所有表，要么不更新任何表。

![2](img/#co_pyt_update_movies_co)

更新表 `movies`。

![3](img/#co_pyt_update_movies_actors_co)

更新表 `movies_actors`。

![4](img/#co_pyt_commit_co)

提交更改。

![5](img/#co_pyt_confirm_co)

确认更改已应用到表格。

如果我们想要删除我们新插入的电影，我们可以使用方法 *delete*：

```
 MySQL  cookbook  Py > `session.start_transaction()`
Query OK, 0 rows affected (0.0006 sec)
 MySQL  cookbook  Py > `movies.delete().where("movie='Darkest Hour'")`
Query OK, 1 item affected (0.0012 sec)
 MySQL  cookbook  Py > `movies_actors.delete().where("movie='Darkest Hour'")`
Query OK, 1 item affected (0.0004 sec)
 MySQL  cookbook  Py > `session.commit()`
Query OK, 0 rows affected (0.0061 sec)
```

在这个示例中，我们首先启动了事务，然后在两个表上调用了方法 *delete*，最后提交了事务。

## 参见

有关在 Python 模式下以面向对象的方式访问表格的更多信息，请参阅 MySQL Shell 的交互式帮助，可以通过命令 *\?* 调用。

# 2.9 在 JavaScript 模式下使用集合

## 问题

您有半结构化数据，并希望将 MySQL 用作文档存储。您还希望使用 NoSQL 查询数据，同时不离开您喜欢的编程语言的编程风格。

## 解决方案

使用 `Collection` 对象及其方法。

## 讨论

MySQL 不仅支持 SQL 语法，还支持 NoSQL。当您使用 SQL 时，您查询表格；当您使用 NoSQL 时，您查询集合。这些集合在物理上存储在具有三列的表格中：生成的唯一标识符，也是主键，一个存储文档的 `JSON` 列，以及一个存储 JSON 模式的内部列。您可以通过使用类 `Schema` 的方法 *createCollection* 来创建集合：

```
 MySQL  cookbook  JS > `collectionLimbs=session.getCurrentSchema().`
                    ->  `createCollection('CollectionLimbs')`
<Collection:CollectionLimbs>
```

上述代码创建了 NoSQL 集合 `CollectionLimbs`。

集合支持模式验证。虽然没有方法可以为现有集合添加模式验证，但我们可以在创建集合时添加模式：

```
 MySQL  cookbook  JS > `session.getCurrentSchema().`
                    -> `dropCollection('collectionLimbs')`
                    -> 
 MySQL  cookbook  JS > `schema={`
                    ->   `"$schema": "http://json-schema.org/draft-07/schema",`
                    ->   `"id": "http://example.com/cookbook.json",`
                    ->   `"type": "object",`
                    ->   `"description": "Table limbs as a collection",`
                    ->   `"properties": {`
                    ->       `"thing": {"type": "string"},`
                    ->       `"legs": {`
                    ->           `"anyOf": [{"type": "number"},{"type": "null"}],`
                    ->           `"default": 0`
                    ->       `},`
                    ->       `"arms": {`
                    ->           `"anyOf": [{"type": "number"},{"type": "null"}],`
                    ->           `"default": 0`
                    ->       `}`
                    ->   `},`
                    ->   `"required": ["thing","legs","arms"]`
                    -> `}`
                    -> 
{
    "$schema": "http://json-schema.org/draft-07/schema", 
    "description": "Table limbs as a collection", 
    "id": "http://example.com/cookbook.json", 
    "properties": {
        "arms": {
            "anyOf": [
                {
                    "type": "number"
                }, 
                {
                    "type": "null"
                }
            ], 
            "default": 0
        }, 
        "legs": {
            "anyOf": [
                {
                    "type": "number"
                }, 
                {
                    "type": "null"
                }
            ], 
            "default": 0
        }, 
        "thing": {
            "type": "string"
        }
    }, 
    "required": [
        "thing", 
        "legs", 
        "arms"
    ], 
    "type": "object"
}
 MySQL  cookbook  JS > `collectionLimbs=session.getCurrentSchema().`
                    -> `createCollection('collectionLimbs',`
                    -> `{"validation": {"level": "strict", "schema": schema}})`
                    -> 
<Collection:CollectionLimbs>
```

一旦创建了 NoSQL 集合，您可以插入、更新、删除和搜索文档。

例如，要将表`limbs`中的文档插入到集合`CollectionLimbs`中，可以使用以下代码：

```
 MySQL  cookbook  JS > `{`
                    ->   `limbs=session.getCurrentSchema().`
                    ->     `getTable('limbs').select().execute();` ![1](img/1.png)
                    ->   `while (limb = limbs.fetchOneObject()) {` ![2](img/2.png)
                    ->     `collectionLimbs.add(` ![3](img/3.png)
                    ->       `mysqlx.expr(JSON.stringify(limb))` ![4](img/4.png)
                    ->       `).execute();` ![5](img/5.png)
                    ->   `}`
                    -> `}`
                    -> 
Query OK, 1 item affected (0.0049 sec)
```

![1](img/#co_jsc_select_co)

从表`limbs`中选择所有行。

![2](img/#co_jsc_fetch_co)

方法*fetchOneObject*返回一个字典对象。

![3](img/#co_jsc_convert_co)

一个字典对象如果没有转换为适当的 JSON 对象就无法保存在集合中。因此，我们先将其转换为 JSON 字符串，然后创建一个表达式，可以将其插入到集合中。

![4](img/#co_jsc_add_co)

方法*add*将文档插入集合中。

![5](img/#co_jsc_execute_co)

在脚本块内更新数据库时，始终需要*execute*方法。

我们用花括号括起代码，因为如果代码跨多行放置，MySQL Shell 将输出*session.getCurrentSchema().getTable('limbs').select().execute()*的结果，并且变量`limbs`只包含关于受影响行数的诊断消息。

最后，我们可以检查刚刚插入到集合`CollectionLimbs`中的数据：

```
 MySQL  cookbook  JS > `collectionLimbs.count()`
11
 MySQL  cookbook  JS > `collectionLimbs.find().limit(3)`
{
    "_id": "00006002f0650000000000000060",
    "arms": 2,
    "legs": 2,
    "thing": "human"
}
{
    "_id": "00006002f0650000000000000061",
    "arms": 0,
    "legs": 6,
    "thing": "insect"
}
{
    "_id": "00006002f0650000000000000062",
    "arms": 10,
    "legs": 0,
    "thing": "squid"
}
3 documents in set (0.0010 sec)
```

您还可以修改和删除集合中的文档。我们将展示如何在 Recipe 2.10 中执行此操作的示例。

## 参见

有关如何使用 JSON 文档和 NoSQL 与 MySQL 的更多信息，请参阅第十九章。

# 2.10 在 Python 模式下使用集合

## 问题

您想要在 Python 模式下使用 DocumentStore 和 NoSQL。

## 解决方案

使用`Collection`对象及其方法。

## 讨论

正如您可以在 JavaScript 模式中做的那样，您也可以在 Python 模式中处理 NoSQL。语法也类似于 JavaScript 模式。但是，方法名称遵循推荐用于 Python 编写的程序的命名风格。

因此，要将集合分配给变量，请使用类`Schema`的*get_collection*方法：

```
 MySQL  cookbook  Py > `collectionLimbs=session.get_current_schema().`↩
                          `get_collection('collectionLimbs')`
```

要选择文档，请使用*find*方法：

```
 MySQL  cookbook  Py > `collectionLimbs.find('legs > 3 and arms > 1')`
{
    "_id": "00006002f0650000000000000066",
    "arms": 2,
    "legs": 4,
    "thing": "armchair"
}
1 document in set (0.0010 sec)
```

方法*find*支持参数，允许按照类似 SQL 中`WHERE`子句的语法搜索特定文档。它还允许聚合结果、排序和选择特定字段。它不支持集合的连接。

要插入新文档，请使用*add*方法：

```
 MySQL  cookbook  Py > `collectionLimbs.add(mysqlx.expr(`↩
                          `'{"thing": "cat", "legs": 2, "arms": 2}'))`
Query OK, 1 item affected (0.0093 sec)
 MySQL  cookbook  Py > `collectionLimbs.find('thing="cat"')`
{
    "_id": "00006002f065000000000000006b",
    "arms": 2,
    "legs": 2,
    "thing": "cat"
}
1 document in set (0.0012 sec)
 MySQL  cookbook  Py > `collectionLimbs.add(mysqlx.expr(`↩
                          `'{"thing": "dog", "legs": 2, "arms": 2}'))`
Query OK, 1 item affected (0.0086 sec)
```

要修改现有行，请使用*add_or_replace_one*或*modify*方法之一：

```
 MySQL  cookbook  Py > `collectionLimbs.add_or_replace_one(`↩
                         `'00006002f065000000000000006b',`↩
                         `{"thing": "cat", "legs": 4, "arms": 0})`
Query OK, 2 items affected (0.0056 sec)
```

方法*add_or_replace_one*将文档`_id`作为第一个参数，JSON 文档作为第二个参数。如果找不到指定`_id`的文档，则插入新文档。如果找到，则替换现有文档。

方法*modify*以搜索条件作为参数，并返回一个支持方法的`CollectionModify`类对象，允许修改参数如*set*。您可以链式调用方法`set`，按需调用多次：

```
 MySQL  cookbook  Py > `collectionLimbs.modify('thing = "dog"').set("legs", 4).set("arms", 0)` 
Query OK, 1 item affected (0.0077 sec)

Rows matched: 1  Changed: 1  Warnings: 0
```

要检查我们是否成功更改了新插入文档 `cat` 和 `dog` 的手臂和腿的数量，可以使用 *find* 方法：

```
 MySQL  cookbook  Py > `collectionLimbs.find('thing in ("dog", "cat")')`
{
    "_id": "00006002f065000000000000006b",
    "arms": 0,
    "legs": 4,
    "thing": "cat"
}
{
    "_id": "00006002f065000000000000006c",
    "arms": 0,
    "legs": 4,
    "thing": "dog"
}
2 documents in set (0.0013 sec)
```

方法 *remove* 从集合中删除文档：

```
 MySQL  cookbook  Py > `collectionLimbs.remove('thing in ("dog", "cat")')`
Query OK, 2 items affected (0.0119 sec)
 MySQL  cookbook  Py > `collectionLimbs.find('thing in ("dog", "cat")')`
Empty set (0.0011 sec)
 MySQL  cookbook  Py > `collectionLimbs.count()`
11
```

方法 *remove* 支持与 *modify* 和 *find* 方法类似的搜索条件。

## 另请参阅

有关在 MySQL 中使用 JSON 文档和 NoSQL 的更多信息，请参见 第十九章。

# 2.11 控制输出格式

## 问题

您希望以与默认格式不同的格式打印结果。

## 解决方案

使用配置选项 `resultFormat` 或命令行参数 `--result-format`、`--table`、`--tabbed`、`--vertical` 或 `--json`。

## 讨论

默认情况下，MySQL Shell 以类似于 *mysql* 客户端的默认表格式打印结果。但是，此格式可以进行自定义。

在 MySQL Shell 内，您可以借助命令 *\option* 或 `Shell` 类的 `shell.options` 成员的 *set* 方法来完成。

因此，要以制表符格式打印表 `artist` 的内容，请运行：

```
 MySQL  cookbook  JS > `\option resultFormat=tabbed`
 MySQL  cookbook  JS > `artist=session.getCurrentSchema().getTable('artist')`
<Table:artist>
 MySQL  cookbook  JS > `artist.select()`
a_id	name
1	Da Vinci
2	Monet
4	Renoir
3	Van Gogh
4 rows in set (0.0009 sec)
```

要切换到垂直格式，请运行：

```
 MySQL  cookbook  JS > `shell.options.set('resultFormat', 'vertical')`
 MySQL  cookbook  JS > `artist.select()`
*************************** 1\. row ***************************
a_id: 1
name: Da Vinci
*************************** 2\. row ***************************
a_id: 2
name: Monet
*************************** 3\. row ***************************
a_id: 4
name: Renoir
*************************** 4\. row ***************************
a_id: 3
name: Van Gogh
4 rows in set (0.0009 sec)
```

JSON 格式支持少量选项。默认情况下，如果选项 `resultFormat` 的值设置为 `json` 或 MySQL Shell 使用选项 `--json` 启动，它等同于 `json/pretty` 或 `--json=pretty`，这意味着结果以 JSON 格式输出，格式化以提高可读性：

```
 MySQL  cookbook  JS > `shell.options.set('resultFormat', 'json')`
 MySQL  cookbook  JS > `artist.select()`
{
    "a_id": 1,
    "name": "Da Vinci"
}
{
    "a_id": 2,
    "name": "Monet"
}
{
    "a_id": 4,
    "name": "Renoir"
}
{
    "a_id": 3,
    "name": "Van Gogh"
}
4 rows in set (0.0008 sec)
```

选项 `ndjson`，`json/raw` 或 `--json=raw` 生成更紧凑的原始 JSON 输出。

```
 MySQL  cookbook  JS > `shell.options.set('resultFormat', 'json/raw')`
 MySQL  cookbook  JS > `artist.select()`
{"a_id":1,"name":"Da Vinci"}
{"a_id":2,"name":"Monet"}
{"a_id":4,"name":"Renoir"}
{"a_id":3,"name":"Van Gogh"}
4 rows in set (0.0003 sec)
```

选项 `json/array` 将结果表示为 JSON 文档数组。

```
 MySQL  cookbook  JS > `shell.options.set('resultFormat', 'json/array')`
 MySQL  cookbook  JS > `artist.select()`
[
{"a_id":1,"name":"Da Vinci"},
{"a_id":2,"name":"Monet"},
{"a_id":4,"name":"Renoir"},
{"a_id":3,"name":"Van Gogh"}
]
4 rows in set (0.0010 sec)
```

如果从命令行选择数据并稍后将其传递给另一个程序，则这将特别有用。

```
$ `mysqlsh cbuser:cbpass@127.0.0.1:33060/cookbook \`
> `-i --execute="session.getSchema('cookbook').\`
> `getTable('artist').select().execute()" \`
> `--result-format=json/array --quiet-start=2 \`
> `| head -n -1 \`
> `| jq '.[] | .name'`
"Da Vinci"
"Monet"
"Renoir"
"Van Gogh"
```

在上述代码中，我们使用选项 `-i` 启动了 *mysqlsh*，启用了交互模式，因此 MySQL Shell 的行为类似于交互式运行，并使用选项 `--quiet-start=2` 禁用了所有欢迎消息。然后我们将选项 `--result-format` 设置为 `json/array` 以启用 JSON 数组输出，使用选项 `--execute` 从表 `artist` 中选择，并将输出传递给 *jq* 命令，该命令删除了所有元数据信息并仅打印了艺术家的名称。

###### 提示

命令 *head -n -1* 从结果中删除显示 *select* 方法返回的行数的最后一行。请注意，指定负数作为 *head -n* 参数的命令可能无法在所有系统上正常工作。如果您在此类系统上，可以忽略 *jq* 命令将打印的错误消息或将其重定向到其他地方：

```
$ `mysqlsh cbuser:cbpass@127.0.0.1:33060/cookbook \`
> `-i --execute="session.getSchema('cookbook').\`
> `getTable('artist').select().execute()" \`
> `--result-format=json/array --quiet-start=2 \`
> `| jq '.[] | .name' 2>/dev/null`
"Da Vinci"
"Monet"
"Renoir"
"Van Gogh"
```

当在 MySQL Shell 启动时启用 JSON 包装并使用选项 `--json[=pretty|raw]` 时，它还将在生成的 JSON 输出中打印诊断信息。

```
 MySQL  cookbook  JS > `session.getCurrentSchema().getTable('artist').select()`
{
    "hasData": true,
    "rows": [
        {
            "a_id": 1,
            "name": "Da Vinci"
        },
        {
            "a_id": 2,
            "name": "Monet"
        },
        {
            "a_id": 4,
            "name": "Renoir"
        },
        {
            "a_id": 3,
            "name": "Van Gogh"
        }
    ],
    "executionTime": "0.0007 sec",
    "affectedRowCount": 0,
    "affectedItemsCount": 0,
    "warningCount": 0,
    "warningsCount": 0,
    "warnings": [],
    "info": "",
    "autoIncrementValue": 0
}
```

如果使用命令行选项 `--result-format=json[/pretty|/raw|/array]` 启用了 JSON 输出，则不会打印此额外信息。

所有输出格式与数据选择方式无关，并且在所有模式下均可用。

## 另请参阅

有关 MySQL Shell 输出格式的更多信息，请参见 [MySQL 用户参考手册](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-output-formats.html)。

# 2.12 使用 MySQL Shell 运行报告

## 问题

您希望定期生成报告。

## 解决方案

使用命令 *\show* 和 *\watch*。

## 讨论

MySQL Shell 命令 *\show* 和 *\watch* 执行报告，包括内置和用户定义的。*\show* 一次执行报告，而 *\watch* 持续运行报告，直到被中断。

报告是一系列预定义的命令。报告可能支持参数。例如，内置报告 `query` 接受 SQL 查询作为参数。内置报告 `thread` 报告特定线程的详细信息。默认情况下，它报告当前线程的详细信息。

```
 MySQL  cookbook  SQL > `\show thread`
GENERAL
Thread ID:                1434
Connection ID:            1382
Thread type:              FOREGROUND
Program name:             mysqlsh
User:                     sveta
Host:                     localhost
Database:                 cookbook
Command:                  Query
Time:                     00:00:00
State:                    executing
Transaction state:        NULL
Prepared statements:      0
Bytes received:           20280
Bytes sent:               40227
Info:                     SELECT json_object('tid',t.THR ... ↩
                          JOIN information_schema.innodb
Previous statement:       NULL
```

###### 警告

内置报告 `thread` 查询 `performance_schema` 和 `sys` 中的表，因此您应作为具有 `performance_schema` 和 `sys` 模式上的 `SELECT` 权限以及 `sys` 模式上的 `EXECUTE` 权限的用户连接。否则，报告将因为 <q>拒绝访问</q> 错误而失败。

但是报告 `thread` 支持参数，因此您可以指定例如线程的 `Connection ID` 并输出关于特定线程的信息。

```
 MySQL  cookbook  SQL > `\show thread -c 1386`
GENERAL
Thread ID:                1438
Connection ID:            1386
Thread type:              FOREGROUND
Program name:             mysql
User:                     sveta
Host:                     localhost
Database:                 cookbook
Command:                  Sleep
Time:                     00:05:44
State:                    NULL
Transaction state:        RUNNING
Prepared statements:      0
Bytes received:           1720
Bytes sent:               29733
Info:                     NULL
Previous statement:       select * from adcount for update
```

`thread` 报告的输出类似于标准的 `PROCESSLIST` 输出，但包含额外的信息，如 `Transaction state` 和 `Previous statement`。当您试图弄清楚是什么阻止了您的事务完成时，后者尤其有用。例如，如果其中一个事务运行多个语句并锁定记录，它可能会导致其他事务等待直到锁被释放。但由于该语句已经执行，因此在常规的 `PROCESSLIST` 输出中是看不到的。

甚至在 `threads` 报告中也可以找到更多有用的信息，默认情况下输出当前用户所属的所有线程的信息。它运行 MySQL Shell 会话，但可以打印服务器上所有线程的信息，并且可以过滤它们并定义输出格式。

例如，要查找所有被阻塞和阻塞事务，可以定义选项 `--where "nblocked > 0 or nblocking > 0"`。

```
 MySQL  cookbook  SQL > `\show threads --foreground` ↩
                           `--where "nblocked > 0 or nblocking > 0"` ↩
                           `-o tid,cid,txid,txstate,nblocked,nblocking,info,pinfo`↩
                           `--vertical`
*************************** 1\. row ***************************
      tid: 1438
      cid: 1386
     txid: 292253
  txstate: RUNNING
 nblocked: 1
nblocking: 0
     info: NULL
    pinfo: select * from adcount for update
*************************** 2\. row ***************************
      tid: 3320
      cid: 3268
     txid: 292254
  txstate: LOCK WAIT
 nblocked: 0
nblocking: 1
     info: update adcount set impressions = impressions + 1 where id=3
    pinfo: NULL
```

因此，在上面的示例中，具有 `Connection ID 3268` 的线程正在尝试执行更新操作。

```
UPDATE adcount SET impressions = impressions + 1 WHERE id=3;
```

，但受到另一个事务的阻塞。否则，具有 `Connection ID 1386` 的线程没有执行任何操作，但阻止了一个线程。它的上一个语句是

```
SELECT * FROM adcount FOR UPDATE;
```

阻止写入表 `adcount` 中的所有行。这样，我们很容易找到为什么连接 3268 中的 *UPDATE* 目前无法完成的原因。

报告 *threads* 具有更多选项。如果使用报告名称运行 *\show* 命令，然后跟随选项 `--help`，您可以找到它们的所有内容。

```
 MySQL  cookbook  SQL > `\show threads --help`
NAME
      threads - Lists threads that belong to the user who owns the current
      session.

SYNTAX
      \show threads [OPTIONS]
      \watch threads [OPTIONS]

DESCRIPTION
      This report may contain the following columns:
...
```

###### 提示

所有 MySQL Shell 命令都支持帮助选项。对于内置命令，运行 *\? COMMAND*，*\help COMMAND* 或 *\h COMMAND*。对于带参数的命令，另外尝试选项 *--help*。

命令 *\watch* 不仅执行报告，还会以一定间隔重复执行。当您想要监视某个参数的变化时，它非常有用。例如，要监视解析查询创建的内部临时表的数量，请运行以下命令：

```
MySQL  cookbook  SQL > `\watch query --nocls` ↩
                          	`SHOW GLOBAL STATUS LIKE 'Created\_tmp\_%tables'`
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 4758  |
| Created_tmp_tables      | 25306 |
+-------------------------+-------+
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 4758  |
| Created_tmp_tables      | 25309 |
+-------------------------+-------+
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 4758  |
| Created_tmp_tables      | 25310 |
+-------------------------+-------+
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 4760  |
| Created_tmp_tables      | 25318 |
+-------------------------+-------+
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 4760  |
| Created_tmp_tables      | 25319 |
+-------------------------+-------+
...
```

查询使用 `LIKE` 运算符和模式来匹配两个系统变量的名称。我们讨论了 `LIKE` 运算符如何工作，并在 Recipe 7.10 中讨论了如何匹配模式。

查询默认间隔为 2 秒。参数 `--nocls` 指示命令在打印最新结果之前不清除屏幕。要停止监视，请发出终止命令 *Ctrl+C*。

###### 提示

# 2.13 使用 MySQL Shell 实用工具

## 问题

您想要使用 MySQL Shell 实用工具。

## 解决方案

在 JavaScript 或 Python 模式中：可以通过交互方式使用全局 `util` 对象的方法，或通过命令行传递方法。

## 讨论

MySQL Shell 提供了多个内置实用工具，可用于执行常见的管理任务，例如检查您的 MySQL 服务器是否可以安全更新到新版本或备份数据。在 JavaScript 和 Python 模式中，可以将这些实用工具作为全局对象 `util` 的方法调用，或作为命令行选项指定。

要查看 MySQL Shell 支持哪些实用工具，请运行命令 *\? util*。在 JavaScript 和 Python 模式下，方法的名称不同，并遵循各自语言的最佳实践命名。在 SQL 模式中，全局对象 `util` 不可用，也无法使用实用程序。

要了解某个实用工具的工作原理，请使用带有实用工具名称作为参数的帮助命令。例如，*\? checkForServerUpgrade* 将在 JavaScript 模式下打印升级检查实用程序的详尽帮助信息。*\? dump_instance* 将在 Python 模式下打印转储实用程序的详细用法说明。

调用实用程序方法与调用任何其他方法没有区别。例如，以下代码以完全引用的 CSV 格式将表 `limbs` 导出到文件 `limbs.csv` 中。

```
 MySQL  cookbook  JS > `util.exportTable(`
                    ->   `'limbs', 'BACKUP/cookbook/limbs.csv',` 
                    ->   `{dialect: "csv-unix"})`
Preparing data dump for table `cookbook`.`limbs`
Data dump for table `cookbook`.`limbs` will not use an index
Running data dump using 1 thread.
NOTE: Progress information uses estimated values and may not be accurate.
Data dump for table `cookbook`.`limbs` will be written to 1 file
91% (11 rows / ~12 rows), 0.00 rows/s, 0.00 B/s
Duration: 00:00:00s                            
Data size: 203 bytes                           
Rows written: 11                               
Bytes written: 203 bytes                       
Average throughput: 203.00 B/s                 

The dump can be loaded using:                  
util.importTable("BACKUP/cookbook/limbs.csv", {
    "characterSet": "utf8mb4",
    "dialect": "csv-unix",
    "schema": "cookbook",
    "table": "limbs"
})
```

在运行此命令之前，您需要创建目录 `BACKUP/cookbook` 或使用其他位置。

以下是将 Python 代码恢复到数据库测试中的表 `limbs` 的示例：

```
 MySQL  cookbook  Py > `\sql CREATE TABLE test.limbs LIKE limbs;`
Fetching table and column names from `cookbook` for auto-completion... ↩
Press ^C to stop.
Query OK, 0 rows affected (0.0264 sec)
 MySQL  cookbook  Py > `util.import_table("BACKUP/cookbook/limbs.csv",` ↩
                          `{"dialect": "csv-unix", "schema": "test"})`
Importing from file '/home/sveta/BACKUP/cookbook/limbs.csv' to table `test`.`limbs` ↩ 
in MySQL Server at 127.0.0.1:3306 using 1 thread
[Worker000] limbs.csv: Records: 11  Deleted: 0  Skipped: 0  Warnings: 0
100% (203 bytes / 203 bytes), 0.00 B/s
File '/home/sveta/BACKUP/cookbook/limbs.csv' (203 bytes) ↩
was imported in 0.0109 sec at 203.00 B/s
Total rows affected in test.limbs: Records: 11  Deleted: 0  Skipped: 0 Warnings: 0
```

我们省略了导入示例中除了必要选项之外的所有内容，以缩短文本。

要使用 *import_table*，您需要处于经典协议会话中。否则，命令将因错误而失败：

```
 MySQL  cookbook  Py > `util.import_table("BACKUP/cookbook/limbs.csv",` ↩
                          `{"dialect": "csv-unix", "schema": "test"})`
Traceback (most recent call last):
  File "<string>", line 1, in <module>
SystemError: RuntimeError: Util.import_table: ↩
A classic protocol session is required to perform this operation.
```

###### 提示

阅读错误消息总是很有帮助，因为它们清楚地显示了问题所在，并经常包含如何修复故障的说明。

另一个可能遇到的错误是：

```
ERROR: The 'local_infile' global system variable must be set to ON ↩
in the target server, after the server is verified to be trusted:
```

要绕过此错误，请使用以下命令启用 `local_infile` 选项：

```
SET GLOBAL local_infile=1;
```

或者等到你到达第十三章，该章节涵盖了 MySQL 数据库对象的导出和导入。

###### 提示

如果您不理解这些示例中实用程序的作用：不要担心。我们将在第十三章中涵盖 MySQL 数据库对象的导出和导入。

如果您想要在不进入交互模式的情况下运行实用程序，可以在两个破折号之后指定它们，遵循标准的 *mysqlsh* 选项：

```
$  `mysqlsh -- util check-for-server-upgrade root@127.0.0.1:13000 --output-format=JSON`
Please provide the password for 'root@127.0.0.1:13000': 
Save password for 'root@127.0.0.1:13000'? [Y]es/[N]o/Ne[v]er (default No): 
{
    "serverAddress": "127.0.0.1:13000",
    "serverVersion": "8.0.23-debug - Source distribution",
    "targetVersion": "8.0.27",
    "errorCount": 0,
    "warningCount": 0,
    "noticeCount": 0,
    "summary": "No known compatibility errors or issues were found.",
...
```

在此示例中，我们首先指定了命令名称，然后添加了两个破折号，接着是全局对象名称、我们想要使用的方法、连接字符串，最后是方法参数。

命令行使用 JavaScript 模式的方法名，即驼峰命名法：*checkForServerUpgrade*，烤肉串命名法：*check-for-server-upgrade*，或蛇形命名法：*check_for_server_upgrade*。有关如何在不进入交互模式的情况下使用全局对象的更多信息，请交互式地使用命令 *\? cmdline*。

###### 提示

您可以使用 `--` 语法在命令行上调用其他全局对象的方法：

```
$ `mysqlsh cbuser:cbpass@127.0.0.1:33060/cookbook` 
          > `-- shell status`
WARNING: Using a password on the command line interface ↩ 
can be insecure.
MySQL Shell version 8.0.22

Connection Id:                23563
Default schema:               cookbook
Current schema:               cookbook
Current user:                 cbuser@localhost
SSL:                          Cipher in use: ↩
                              TLS_AES_256_GCM_SHA384 ↩ 
                              TLSv1.3
...
```

但是，并非所有全局对象都受支持。在交互模式下使用命令 *\? cmdline* 查看支持的对象列表。

## 参见

有关 MySQL Shell 实用程序的其他信息，请参阅[MySQL Shell 实用程序](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-utilities.html)。

# 2.14 使用 Admin API 自动化复制管理

## 问题

您想要自动化例行的 DBA 任务，例如部署 MySQL 服务器。

## 解决方案

使用 AdminAPI。

## 讨论

MySQL Shell 不仅支持 X Dev API 用于查询数据库，还支持 AdminAPI，允许您管理 InnoDB ReplicaSet 和 InnoDB Cluster。AdminAPI 由三个类组成：`Dba`、`Cluster` 和 `ReplicaSet`。

AdminAPI 可从类为 `DBA` 的全局对象 `dba` 访问。它允许您配置 MySQL 实例并启动独立沙盒、复制集或集群。

要配置独立的沙盒，请在 JavaScript 模式中使用 *deploySandboxInstance* 方法，或在 Python 模式中使用 *deploy_sandbox_instance*。此方法将一个端口号和一个参数字典作为参数：

```
 MySQL  cookbook  JS > `dba.deploySandboxInstance(13000,` 
                    -> `{"portx": 13010, "mysqldOptions": ["log-bin=cookbook"]})`
A new MySQL sandbox instance will be created on this host in 
/home/sveta/mysql-sandboxes/13000

Warning: Sandbox instances are only suitable for deploying and 
running on your local machine for testing purposes and are not 
accessible from external networks.

Please enter a MySQL root password for the new instance: 

Deploying new MySQL instance...

Instance localhost:13000 successfully deployed and started.
Use shell.connect('root@localhost:13000') to connect to the instance.
```

这将创建一个带有名称、从 *cookbook* 开始的二进制日志已启用的沙盒实例，端口为 X 端口 13010：

```
 MySQL  localhost:13000 ssl  JS > `shell.connect('root@localhost:13000')`
Creating a session to 'root@localhost:13000'
Please provide the password for 'root@localhost:13000': 
Fetching schema names for autocompletion... Press ^C to stop.
Closing old connection...
Your MySQL connection id is 13
Server version: 8.0.22-13 Percona Server (GPL), Release '13', Revision '6f7822f'
No default schema selected; type \use <schema> to set one.
 MySQL  localhost:13000 ssl  JS > `\sql show variables like 'log_bin_basename';`
+------------------+--------------------------------------------------------+
| Variable_name    | Value                                                  |
+------------------+--------------------------------------------------------+
| log_bin_basename | /home/sveta/mysql-sandboxes/13000/sandboxdata/cookbook |
+------------------+--------------------------------------------------------+
1 row in set (0.0027 sec)
```

要停止实例，请在 JavaScript 模式下使用 *stopSandboxInstance* 方法，或在 Python 模式下使用 *stop_sandbox_instance*：

```
 MySQL  localhost:13000 ssl  JS > `dba.stopSandboxInstance(13000)`
The MySQL sandbox instance on this host in 
/home/sveta/mysql-sandboxes/13000 will be stopped

Please enter the MySQL root password for the instance 'localhost:13000': 

Stopping MySQL instance...

Instance localhost:13000 successfully stopped.
```

要销毁实例，请在 JavaScript 模式下使用 *deleteSandboxInstance* 方法，或在 Python 模式下使用 *delete_sandbox_instance*：

```
 MySQL  cookbook  Py > `dba.delete_sandbox_instance(13000)`

Deleting MySQL instance...

Instance localhost:13000 successfully deleted.
```

全局对象 `dba` 可从命令行访问：

```
$ `mysqlsh cbuser:cbpass@127.0.0.1:33060/cookbook -- dba kill-sandbox-instance 13000`
WARNING: Using a password on the command line interface can be insecure.

Killing MySQL instance...

Instance localhost:1300 successfully killed.
```

在 SQL 模式下，全局对象 `dba` 不可用。

## 参见

有关使用 AdminApi 创建和管理 ReplicaSet 的其他信息，请参阅食谱 3.17。有关使用 AdminApi 创建和管理 InnoDB Cluster 的其他信息，请参阅“InnoDB Cluster”

# 2.15 使用 JavaScript 对象

## 问题

您希望将文档作为对象进行操作，并且希望使用自己的方法和属性修改它们，并将它们存储在数据库中。

## 解决方案

创建一个对象，该对象具有与数据库通信的所有必要方法，并将其用作数据对象的原型。

## 讨论

JavaScript 是一种面向对象的编程语言，您可以轻松创建对象、修改对象并将其存储在数据库中。有时，直接编写 *myObject.save()* 可能比调用完整的 X DevAPI Collection 类方法链更简单。例如，您可能希望替换以下内容：

```
session.getCurrentSchema().getCollection('CollectionLimbs').↩
        addOrReplaceOne(myObject).execute()
```

使用单个

```
CollectionLimbs.save()
```

调用。

JavaScript 支持继承，因此您可以创建一个对象，该对象具有所有必要的方法，这些方法使用 `Collection` 类的方法，并将其用作包含业务逻辑的对象的原型。

让我们创建一个名为 `CookbookCollection` 的对象作为示例，它将具有 *find*、*save* 和 *remove* 方法。它们将在集合中搜索对象，在修改后保存它，并在必要时从数据库中删除。`CookbookCollection` 对象还将具有一个名为 `collection` 的属性，用于存储表示存储我们对象的集合的对象。

为了使我们的方法更简单以增加清晰度，我们不会添加错误处理。您可以自行添加此功能。例如，如果用户忘记设置集合属性，您可以抛出自定义异常或使用默认集合替代。我们依赖于 JavaScript 内置异常。

让我们开始创建我们的对象：

```
mysql-js [cookbook]> `var CookbookCollection = {` 
                  ->  `// Collection where the object is stored`
                  ->  `collection: null,`
```

首先，我们定义属性集合，存储对象。我们不在此处设置集合的名称，因为我们希望我们的原型可以与任何集合一起使用：

```
                  ->  `// Searches collection and returns first`
                  ->  `// object that satisfy search condition.`
                  -> `find: function(searchCondition) {` 
                  ->  `return this.collection.find(searchCondition).`
                  -> `execute().fetchOne();` 
                  ->  `},`
```

函数 *find* 使用任何搜索条件搜索集合。可以是 `'_id = "00006002f0650000000000000061"'` 或 `'thing="human"'`。换句话说，任何 *Collection.find* 方法接受的条件。然后，我们获取一个文档并将其作为结果返回。我们故意没有添加任何唯一性检查代码或任何其他方式来确保只有一个满足我们条件的文档，因为我们希望尽可能简单地进行示例，并使其可以与任何集合一起使用：

```
                  ->  `// Saves the object in the database`
                  ->  `save: function() {`
                  -> `// If we know _id of the object we are` 
                  ->  `// updating the existing one`
                  ->  `// We use not so effective method addOrReplaceOne`
                  ->  `// instead of modify for simplicity.`
                  ->  `if ('_id' in this) {`
                  ->  `this.collection.addOrReplaceOne(this._id,`
                  ->  `// We use double conversion, because we cannot`
                  ->  `// store an object with methods in the database`
                  ->  `JSON.parse(`
                  ->  `JSON.stringify(`
                  ->  `this, Object.getOwnPropertyNames(`
                  ->  `Object.getPrototypeOf(this))`
                  ->  `)`
                  ->  `)`
                  ->  `)`
                  ->  `} else {`
                  ->  `// In case the object does not exist in the`
                  -> `//  database yet, we add it and assign` 
                  ->  `// generated _id to its own property.`
                  ->  `// This _id could be used later if we want to update`
                  ->  `// or remove the database entry.`
                  ->  `this._id = this.collection.add(`
                  ->  `JSON.parse(`
                  ->  `JSON.stringify(`
                  ->  `this, Object.getOwnPropertyNames(`
                  ->  `Object.getPrototypeOf(this))`
                  ->  `)`
                  ->  `)`
                  ->  `).execute().getGeneratedIds()[0]`
                  ->  `}`
                  ->  `},`
```

方法 *save* 将对象存储在数据库中。如果对象中没有 `_id` 字段，通常意味着数据库中还没有此对象。因此，我们使用方法 *add* 将其插入到数据库中，并将对象的 `_id` 属性设置为由 MySQL 生成的值。如果已经存在这样的属性，这意味着该对象已经在数据库中或者我们希望显式设置 `_id`。在这种情况下，我们使用方法 *addOrReplaceOne*，它将添加具有指定唯一标识符的新对象或替换现有对象：

```
                  ->  `// Removes the entry from the database.`
                  ->  `// Once removed we unset property _id of the object.`
                  ->  `remove: function() {`
                  ->  `this.collection.remove("_id = '" + this._id + "'").`
                  ->  `execute()`
                  ->  `delete Object.getPrototypeOf(this)._id`
                  ->  `delete this._id`
                  ->  `}`
                  -> `}`
```

*remove*方法从数据库中删除记录，并额外删除我们对象的属性`_id`，因此，如果我们想要再次将其存储在数据库中，它将被视为新的对象，并生成新的唯一标识符。我们从原型和对象中删除属性`_id`。

让我们以我们在 Recipe 2.9 中创建的`CollectionLimbs`集合为例。首先，我们从当前的`session`中检索它，并将其设置为`CookbookCollection`对象的`collection`属性。

```
mysql-js [cookbook]> `CookbookCollection.collection=session.getCurrentSchema().`
                  -> `getCollection('CollectionLimbs')`
                  -> 
<Collection:CollectionLimbs>
```

###### 提示

在 Recipe 2.9 中，我们回滚了所有对`CollectionLimbs`的修改。如果您在运行本章示例之前继续进行自己的实验，请执行：

```
CookbookCollection.collection ↩
.remove("thing='cat' or thing='dog'")
```

然后让我们创建一个有两只手臂和两条腿的`cat`对象：

```
mysql-js [cookbook]> `var cat = {`
                  ->  `thing: "cat",`
                  ->  `arms: 2,`
                  ->  `legs: 2`
                  -> `}`
                  ->
```

要能够将我们的猫存储在数据库中，我们需要将对象`CookbookCollection`指定为对象`cat`的原型：

```
mysql-js [cookbook]> `cat = Object.setPrototypeOf(CookbookCollection, cat)`
{
    "arms": 2, 
    "collection": <Collection:CollectionLimbs>, 
    "find": <Function:find>, 
    "legs": 2, 
    "remove": <Function:remove>, 
    "save": <Function:save>, 
    "thing": "cat"
}
```

现在我们可以将我们的对象保存在数据库中：

```
mysql-js [cookbook]> `cat.save()`
```

我们可以检查是否可以使用*find*方法检索这样的对象：

```
mysql-js [cookbook]> `CookbookCollection.find('thing = "cat"')`
{
    "_id": "000060140a2d0000000000000007", 
    "arms": 2, 
    "legs": 2, 
    "thing": "cat"
}
```

我们还可以确认我们的对象现在具有`_id`属性：

```
mysql-js [cookbook]> `cat._id`
000060140a2d0000000000000007
```

你看到这里有什么问题吗？是的！这只猫有两只手臂和两条腿，而通常猫没有手臂而是四条腿。让我们修正一下：

```
mysql-js [cookbook]> `cat.arms=0`
0
mysql-js [cookbook]> `cat.legs=4`
4
mysql-js [cookbook]> `cat.save()`
mysql-js [cookbook]> `CookbookCollection.find('thing = "cat"')`
{
    "_id": "000060140a2d0000000000000007", 
    "arms": 0, 
    "legs": 4, 
    "thing": "cat"
}
```

现在我们的猫状态良好。

如果我们想要清理集合并将其保留在我们的实验之前的状态，我们可以从数据库中删除文档`cat`：

```
mysql-js [cookbook]>  `cat.remove()`
```

我们还可以注意到`cat._id`属性在我们的对象中不再存在：

```
mysql-js [cookbook]> `cat._id`
mysql-js [cookbook]>
```

如果我们决定再次将对象存储在数据库中，将生成新的唯一标识符。

您可以在`recipes`分发中的文件*mysql_shell/CookbookCollection.js*中找到`CookbookCollection`的代码。

# 2.16 使用 Python 的数据科学模块填充测试数据

## 问题

您想用部分随机数据填充测试表。例如，您需要 ID 按顺序排列。您还希望它们具有真实的名字和姓氏。表中的其余值可以是随机的，但索引应具有特定的基数。

## 解决方案

使用 Python 及其特定的数据科学模块进行脚本数据填充。

## 讨论

我们经常处于需要用模拟真实世界数据的假数据填充表格的情况。例如，当您开发一个应用程序并想要检查如果存储在其中的数据量增加会发生什么时。或者您遇到一个特定查询在生产中运行缓慢的情况，希望在测试服务器上进行实验，但不希望由于安全或性能原因复制生产数据。当您想要向第三方顾问寻求帮助时，可能还需要此任务。

其中一个示例是表`patients`，我们在 Recipe 24.12 中使用它。这是一张表，存储了在医院里待了一天以上的病人记录。它存储了国民身份证号、名字、姓氏、性别、人的诊断和结果，例如病人在医院里的停留日期以及他们是否康复、以相同症状退房，甚至死亡。如果你运行 *SHOW CREATE TABLE* 命令，你可以找到详细信息：

```
 MySQL  cookbook  Py > `session.sql('SHOW CREATE TABLE patients')`
*************************** 1\. row ***************************
       Table: patients
Create Table: CREATE TABLE `patients` (
  `id` int NOT NULL AUTO_INCREMENT,
  `national_id` char(32) DEFAULT NULL,
  `name` varchar(255) DEFAULT NULL,
  `surname` varchar(255) DEFAULT NULL,
  `gender` enum('F','M') DEFAULT NULL,
  `age` tinyint unsigned DEFAULT NULL,
  `additional_data` json DEFAULT NULL,
  `diagnosis` varchar(255) DEFAULT NULL,
  `result` enum('R','N','D') DEFAULT NULL↩
   COMMENT 'R=Recovered, N=Not Recovered, D=Dead',
  `date_arrived` date NOT NULL,
  `date_departed` date DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1101 ↩
DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.0009 sec)
```

当然，对于这张表的示例，我们不能想象使用真实数据。然而，我们仍然希望假装数据是真实的。例如，名字和姓氏应该是流行的名字，像一个叫 John Doe 或 Ann Smith 的人，性别应该对应正确的名字。例如，John 很可能是男性，Ann 很可能是女性。年龄应该在合理的范围内，离开日期应该大于病人到达医院的日期，而一个病人在那里待上 10 年是不太可能的。因此，我们需要一种聪明的方法来填充表格的虚假值。

Python 是一种常用于数据分析和统计的编程语言。它有像 `pandas` 这样的库，帮助操作大型数据集。它有方便的方法来读取和生成数据。这就是为什么 Python 是执行我们任务的理想方式之一。

要在 MySQL Shell 中使用 `pandas` 模块，你需要在你的机器上安装它，并将库所在的路径添加到 MySQL Shell 的 `sys.path` 中。以下是帮助你执行此任务的步骤。

+   首先检查 MySQL Shell 运行的 Python 版本。在我们的情况下，这是版本 3.7.7：

    ```
     MySQL  cookbook  Py > `import sys`
     MySQL  cookbook  Py > `sys.version`
    3.7.7 (default, Aug 12 2020, 09:13:48) 
    [GCC 4.4.7 20120313 (Red Hat 4.4.7-23.0.1)]
    ```

+   MySQL Shell 不带有 *python* 可执行文件和 *pip*，你不能在 MySQL Shell 外部运行它们。因此，你需要安装与 MySQL Shell 的 Python 版本相同的版本。我们选择保持系统范围内已安装的版本 3.8.5 不变，并从源代码安装与我们的 MySQL Shell 实例使用的相同版本 3.7.7 到本地目录中。你可能决定在系统范围内使用相同的版本。

+   一旦安装了必要版本的 Python，请检查它存储模块的位置，并将此目录添加到 MySQL Shell 的 `sys.path` 中：

    ```
     MySQL  cookbook  Py > `sys.path.append(`↩
     `"/home/sveta/bin/python-3.7.7/lib/python3.7/site-packages")`
    ```

    ###### 提示

    要避免每次想要使用不是 MySQL Shell 发行版的模块时都输入此命令，请将此命令添加到 Python 模式配置文件中。默认情况下，此文件位于 `~/.mysqlsh/mysqlshrc.py`。

+   安装必要的软件包。例如，我们使用了`numpy`、`pandas`、`random`、`string`和`datetime`。

一旦满足这些先决条件，我们就准备好用示例数据填充我们的表格了。

### 逐步填充数据

首先，我们需要导入所有必要的软件包。在 MySQL Shell 的 Python 协议会话中输入以下内容：

```
import numpy
from pandas import pandas
import random
import string
from datetime import datetime
from datetime import timedelta
```

现在我们准备生成数据了。

对于姓名和姓氏，我们决定使用在数据集中找到的真实姓名，这些数据集可以在互联网上找到。您可以在`recipes`分发的*datasets*目录中找到我们使用的数据集、它们的许可和分发权利。对于诊断，我们还使用了公开可用的前 8 个诊断数据及其频率，以及虚假的诊断“数据恐惧症”，其频率更高。性别数据与姓名一起存储。所有其他值都是生成的。我们并不在乎这些数据看起来是否真实。例如，我们可能会得到一个 16 岁的患者死于酒精性肝病，这在现实生活中不太可能发生，但是为了演示目的，这应该足够了。然而，Python 允许解决这类冲突。您可以更改我们的示例，以便创建更真实的数据。

定义最终表中行数的变量会很方便：

```
 MySQL  cookbook  Py > `num_rows=1000`
```

现在，一旦我们准备好了，让我们讨论一下我们将如何处理表`patients`的每一列。

姓名和性别

姓名和性别存储在文件`top-350-male-and-female-names-since-1848-2019-02-26.csv`中，格式如下：

```
$ `head datasets/top-350-male-and-female-names-since-1848-2019-02-26.csv`
Rank,Female Name,Count,Male Name,Count_
1,Mary,54276,John,108533
2,Margaret,49170,William,87239
3,Elizabeth,36556,James,69987
4,Sarah,28230,David,62774
5,Patricia,20689,Robert,56511
6,Catherine,19713,Michael,51768
7,Susan,19165,Peter,44758
8,Helen,18881,Thomas,42467
9,Emma,18192,George,39195
```

这意味着每一行包含一个排名从 1 到 350 的名字，一个传统上是女性的名字和一个传统上是男性的名字以及这些名字的数量。我们对排名和数量不感兴趣。我们只需要带有性别信息的女性和男性名字。因此，在读取此数据集后，我们需要对该数据集进行轻微的操作。

首先，我们使用`pandas`的*read_csv*方法读取文件。我们将读取文件两次：第一次是传统女性名字，第二次是传统男性名字。我们将仅在第一次尝试中使用`Female Name`列，并仅在第二次尝试中使用`Male Name`列。我们还将重命名此列，使其与我们数据库中的列名对应：

```
 MySQL  cookbook  Py > `female_names=pandas.\`
                    -> `read_csv(`
                    ->  `"top-350-male-and-female-names-since-1848-2019-02-26.csv",`
                    ->  `usecols=["Female Name"]`
                    -> `).rename(columns={'Female Name': 'name'})`
                    -> 
 MySQL  cookbook  Py > `male_names=pandas.\`
                    -> `read_csv(`
                    ->  `"top-350-male-and-female-names-since-1848-2019-02-26.csv",`
                    ->  `usecols=["Male Name"]`
                    -> `).rename(columns={'Male Name': 'name'})`
                    ->
```

完成后，向我们的数据集添加一个`gender`列：

```
 MySQL  cookbook  Py > `female_names['gender']=(['F']*`↩
                          `female_names.count()['name'])`
 MySQL  cookbook  Py > `male_names['gender']=(['M']*`↩
                          `male_names.count()['name'])`
```

最后，将两个数据集连接成一个：

```
 MySQL  cookbook  Py > `names=pandas.\`
                    -> `concat([female_names, male_names],`
                    ->  `ignore_index=True)`
                    ->
```

为了读取文件`top-350-male-and-female-names-since-1848-2019-02-26.csv`，它应该在当前工作目录中，或者您需要提供此文件的绝对路径。要找到当前工作目录，请运行：

```
os.getcwd()
```

要更改工作目录，请运行：

```
os.chdir('/mysqlcookbook/recipes/datasets')
```

这将允许读取位于`/mysqlcookbook/recipes/datasets`中的文件。调整目录路径以反映您的环境。

###### 提示

Python 模块`pandas`的方法`concat`类似于 SQL 的`UNION`子句。

我们可以通过键入其名称来检查使用`pandas`数据结构`DataFrame`的数据集内容：

```
 MySQL  cookbook  Py > `names`
          name gender
0         Mary      F
1     Margaret      F
2    Elizabeth      F
3        Sarah      F
4     Patricia      F
..         ...    ...
695    Quentin      M
696     Henare      M
697        Joe      M
698      Darcy      M
699       Wade      M

[700 rows x 2 columns]
```

数据帧中的行数少于我们在表中想要的行数，因此我们需要生成更多的行。我们还希望对数据进行洗牌，以获得姓名的随机分布。为此，我们将使用*sample*方法。由于我们正在创建一个比初始数据集更大的集合，因此我们需要指定选项`replace=True`。我们还将使用*pandas.Series*方法重新创建新数据帧的索引，使其有序。

```
 MySQL  cookbook  Py > `names=names.sample(num_rows, replace=True).\`
                    -> `set_index(pandas.Series(range(num_rows)))`
                    ->
```

姓氏

对于姓氏，我们将使用存储在文件 `Names_2010Census.csv` 中的数据集。它有多列，如排名、姓氏数量等等。但我们只关心第一列：`name`。我们也不需要文件的最后一行，其中包含对 `ALL OTHER NAMES` 的记录。该文件中的姓氏以大写字母存储。我们可以以不同的格式进行格式化，但我们决定保留原样。我们还将列 `name` 重命名为 `surname`，以便与我们的表定义一致。

```
 MySQL  cookbook  Py > `surnames=pandas.read_csv("Names_2010Census.csv",` 
                    ->  `usecols=['name'], skipfooter=1, engine='python').\`
                    -> `rename(columns={'name': 'surname'})`
                    ->
```

*pandas* 打印一个警告，它将使用更慢但更强大的 `python` 引擎来处理文件，但可以忽略这个警告。

我们将使用与姓名相同的方法对姓氏进行洗牌。

```
 MySQL  cookbook  Py > `surnames=surnames.sample(num_rows, replace=True).\`
                    -> `set_index(pandas.Series(range(num_rows)))`
                    ->
```

诊断

我们手动准备了文件 `diagnosis.csv`，仅包含 9 个诊断，因此我们只需要读取它，不需要指定任何选项。

```
 MySQL  cookbook  Py > `diagnosises=pandas.read_csv('diagnosis.csv')`
 MySQL  cookbook  Py > `diagnosises`
                               diagnosis  frequency
0                Acute coronary syndrome        2.1
1                Alcoholic liver disease        0.3
2                              Pneumonia        3.6
3  Chronic obstructive pulmonary disease        2.1
4                Gastro-intestinal bleed        0.8
5                          Heart failure        0.8
6                                 Sepsis        0.8
7                Urinary tract infection        2.4
8                            Data Phobia        6.2
```

诊断与姓名和姓氏不同，因为它们具有不同的频率，并且我们希望它们按照这些频率在我们的最终数据集中分布。因此，我们将向方法 `sample` 传递参数 `weights`。

```
 MySQL  cookbook  Py > `diagnosises=diagnosises.sample(`
                    ->  `num_rows, replace=True,`
                    ->  `weights=diagnosises['frequency']`
                    -> `).set_index(pandas.Series(range(num_rows)))`
                    ->
```

结果

结果的数据类型是 `ENUM`，只能包含三个可能的值：`R` 表示康复，`N` 表示未康复，`D` 表示死亡。我们不会使用任何来源来获取这样的结果，而是交互式地生成一个 DataFrame。

```
 MySQL  cookbook  Py > `results = pandas.DataFrame({`
                    ->  `"result": ["R", "N", "D"],`
                    ->  `"frequency": [6,3,1]`
                    -> `})`
                    ->
```

我们向我们的结果添加了频率。这些频率与现实无关：我们只需要它们以不同的方式分配我们的结果。

由于我们有结果的频率，我们将生成类似于我们对诊断进行的方式的数据集。

```
 MySQL  cookbook  Py > `results=results.sample(`
                    ->  `num_rows, replace=True,`
                    ->  `weights=results['frequency']`
                    -> `).set_index(pandas.Series(range(num_rows)))`
                    ->
```

表

我们已准备好主要数据集。现在我们可以逐行将行插入表中。

首先，让我们检索一个 `Table` 对象，以便可以舒适地查询它。

```
 MySQL  cookbook  Py > `patients=session.get_schema('cookbook').`↩
                          `get_table('patients')`
```

然后开始循环

```
 MySQL  cookbook  Py > `for i in range(num_rows):`
```

所有后续的生成将在循环中进行。

国家 ID

国家 ID 的格式可以在国家之间有所不同，我们只需要一些能够遵循某种模式的唯一内容。我们决定使用两位数字，后跟两个大写字母，然后是六位数字的格式。为了生成随机数字，我们将使用模块 `random` 的 *randrange* 方法，并且为了生成字母，我们将使用模块 `random` 的 *sample* 方法。我们将使用预定义的集合 `string.ascii_uppercase` 作为数据集来抽样。然后，我们将生成的数组连接到空字符串上，这样就会创建一个字符串：

```
 MySQL  cookbook  Py > `national_id=str(random.randrange(10,99)) +\`
                    -> `''.join(random.sample(string.ascii_uppercase, 2)) + \`
                    -> `str(random.randrange(100000, 999999))`
                    ->
```

年龄

对于年龄，我们将简单地选择一个 15 到 99 之间的数字。我们不关心年龄频率或特定年龄患者有某种疾病的数量：

```
 MySQL  cookbook  Py > `age=random.randrange(15, 99)`
```

患者在医院中度过的日期

对于 `date_arrived` 列，我们决定只使用 2020 年的任何日期。我们可以通过指定开始日期为 2020 年 1 月 1 日并使用 *timedelta* 方法来生成这样的日期：

```
 MySQL  cookbook  Py > `date_arrived=datetime.\`
                    -> `strptime('2020-01-01', '%Y-%m-%d') +\`
                    -> `timedelta(days=random.randrange(365))`
                    ->
```

对于 `date_departed` 列，我们将使用相同的思路，但我们将使用 `date_arrived` 作为起始日期，并间隔两个月：

```
 MySQL  cookbook  Py > `date_departed=date_arrived +\`
                    -> `timedelta(days=random.randrange(60))`
                    ->
```

此代码将 `date_arrived` 和 `date_departed` 创建为无法插入到 MySQL 表中的 `datetime` Python 对象，因此我们需要将它们转换为字符串格式：

```
 MySQL  cookbook  Py > `date_arrived=date_arrived.strftime('%Y-%m-%d')`
 MySQL  cookbook  Py > `date_departed=date_departed.strftime('%Y-%m-%d')`
```

准备行

我们有要插入到我们表的第 `i-` 行中的值，插入到 `national_id`、`age`、`date_arrived` 和 `date_departed` 列中。但其余值存储在精确所需行数的 `DataFrame` 中。我们只需要从 `DataFrame` 中检索特定行：

```
 MySQL  cookbook  Py > `name=names['name'][i]`
 MySQL  cookbook  Py > `gender=names['gender'][i]`
 MySQL  cookbook  Py > `surname=surnames['surname'][i]`
 MySQL  cookbook  Py > `result=results['result'][i]`
 MySQL  cookbook  Py > `diagnosis=diagnosises['diagnosis'][i]`
```

在表中插入行

现在我们准备将一行插入到我们的表中。我们将使用我们在 Recipe 2.8 中详细讨论的 `Table` 类的 *insert* 方法：

```
 MySQL  cookbook  Py > `patients.insert(`
                    ->  `'national_id', 'name', 'surname',`
                    ->  `'gender', 'age', 'diagnosis',`
                    ->  `'result', 'date_arrived', 'date_departed'`
                    -> `).values(`
                    ->  `national_id, name, surname,`
                    ->  `gender, age, diagnosis,`
                    ->  `result, date_arrived, date_departed`
                    -> `).execute()`
```

将所有内容组合在一起

或许将刚刚编写的代码定义为函数会更方便，这样我们就可以重复使用它。让我们创建一个名为 *generate_patients_data* 的函数。

```
def generate_patients_data(num_rows):
    # read datasets
    # names and genders
    female_names = pandas.read_csv(
        "top-350-male-and-female-names-since-1848-2019-02-26.csv", 
        usecols = ["Female Name"]
    ).rename(columns = {'Female Name': 'name'})
    female_names['gender'] = (['F']*female_names.count()['name'])
    male_names = pandas.read_csv(
        "top-350-male-and-female-names-since-1848-2019-02-26.csv", 
        usecols = ["Male Name"]
    ).rename(columns = {'Male Name': 'name'})
    male_names['gender'] = (['M']*male_names.count()['name'])
    names = pandas.concat([female_names, male_names], ignore_index=True)
    surnames = pandas.read_csv(
        "Names_2010Census.csv", 
        usecols=['name'], skipfooter=1
    ).rename(columns={'name': 'surname'})
    # diagnosises
    diagnosises = pandas.read_csv('diagnosis.csv')
    # Possible results
    results = pandas.DataFrame({
        "result": ["R", "N", "D"], 
        "frequency": [6,3,1]
    })
    # Start building data
    diagnosises = diagnosises.sample(
        num_rows, replace=True, 
        weights=diagnosises['frequency']
    ).set_index(pandas.Series(range(num_rows)))
    results = results.sample(
        num_rows, replace=True, 
        weights=results['frequency']
    ).set_index(pandas.Series(range(num_rows)))
    names=names.sample(
        num_rows, replace=True
    ).set_index(pandas.Series(range(num_rows)))
    surnames=surnames.sample(
        num_rows, replace=True
    ).set_index(pandas.Series(range(num_rows)))
    # Get table object
    patients=session.get_schema('cookbook').get_table('patients')
    # Loop, inserting rows
    for i in range(num_rows):
        national_id = str(random.randrange(10,99)) + \
            ''.join(random.sample(string.ascii_uppercase, 2)) + \
            str(random.randrange(100000, 999999))
        age = random.randrange(15, 99)
        date_arrived = datetime.strptime('2020-01-01', '%Y-%m-%d') + \
            timedelta(days=random.randrange(365))
        date_departed = date_arrived + timedelta(days=random.randrange(60))
        date_arrived = date_arrived.strftime('%Y-%m-%d')
        date_departed = date_departed.strftime('%Y-%m-%d')
        name = names['name'][i]
        gender = names['gender'][i]
        surname = surnames['surname'][i]
        result = results['result'][i]
        diagnosis = diagnosises['diagnosis'][i]
        patients.insert(
            'national_id', 'name', 'surname', 
            'gender', 'age', 'diagnosis', 
            'result', 'date_arrived', 'date_departed'
        ).values(
            national_id, name, surname, 
            gender, age, diagnosis, 
            result, date_arrived, date_departed
        ).execute()
```

我们可以通过截断 *patients* 表然后调用该函数来检查其工作方式：

```
 MySQL  cookbook  Py >  `\sql truncate table patients`
Query OK, 0 rows affected (0.0477 sec)
 MySQL  cookbook  Py >  `session.get_schema('cookbook').get_table('patients').count()`
0
 MySQL  cookbook  Py >  `generate_patients_data(1000)`
__main__:17: ParserWarning: Falling back to the 'python' engine ↩
because the 'c' engine does not support skipfooter; ↩
you can avoid this warning by specifying engine='python'.
 MySQL  cookbook  Py >  `session.get_schema('cookbook').` ↩
 `get_table('patients').count()`
1000
 MySQL  cookbook  Py >  `session.get_schema('cookbook').` ↩
`get_table('patients').select().limit(10)`
+----+-------------+----------+------------+--------+-----+-----------------+....
| id | national_id | name     | surname    | gender | age | additional_data | ...
+----+-------------+----------+------------+--------+-----+-----------------+....
|  1 | 74LM282144  | May      | NESSELRODE | F      |  83 | NULL            | ...
|  2 | 44PR883357  | Kathryn  | DAKROUB    | F      |  44 | NULL            | ...
|  3 | 60JP130066  | Owen     | CIELINSKI  | M      |  47 | NULL            | ...
|  4 | 28ST588095  | Diana    | KILAR      | F      |  35 | NULL            | ...
|  5 | 77RP202627  | Beryl    | ANGIONE    | F      |  43 | NULL            | ...
|  6 | 27MU569536  | Brian    | HOUDEK     | M      |  84 | NULL            | ...
|  7 | 94AG787006  | Fredrick | WOHLMAN    | M      |  20 | NULL            | ...
|  8 | 42BX974594  | Jarrod   | DECAPUA    | M      |  64 | NULL            | ...
|  9 | 63XJ322387  | Ruth     | PAHUJA     | F      |  16 | NULL            | ...
| 10 | 91AT797455  | Frances  | VANBRUGGEN | F      |  63 | NULL            | ...
+----+-------------+----------+------------+--------+-----+-----------------+....

+----+.....+-------------------------+--------+--------------+---------------+
| id | ... | diagnosis               | result | date_arrived | date_departed |
+----+.....+-------------------------+--------+--------------+---------------+
|  1 | ... | Data Phobia             | D      | 2020-03-20   | 2020-04-26    |
|  2 | ... | Data Phobia             | R      | 2020-03-20   | 2020-05-09    |
|  3 | ... | Pneumonia               | R      | 2020-04-05   | 2020-04-23    |
|  4 | ... | Acute coronary syndrome | R      | 2020-04-18   | 2020-05-01    |
|  5 | ... | Pneumonia               | R      | 2020-01-31   | 2020-02-07    |
|  6 | ... | Acute coronary syndrome | D      | 2020-01-25   | 2020-03-06    |
|  7 | ... | Data Phobia             | R      | 2020-08-10   | 2020-09-04    |
|  8 | ... | Pneumonia               | R      | 2020-02-12   | 2020-03-31    |
|  9 | ... | Pneumonia               | N      | 2020-11-17   | 2020-12-19    |
| 10 | ... | Sepsis                  | R      | 2020-12-11   | 2020-12-29    |
+----+.....+-------------------------+--------+--------------+---------------+
10 rows in set (0.0004 sec)
```

我们还可以将此函数存储在文件中，并稍后重新使用它。我们将在 Recipe 2.17 中讨论如何重复使用用户代码。

您可以在 `recipes` 发行版的 *mysql_shell/generate_patients_data.py* 文件中找到 *generate_patients_data* 函数的代码。

## 参见

关于 Python 模块 `pandas` 的更多信息，请参阅[pandas 文档](https://pandas.pydata.org/pandas-docs/stable/index.html)。

# 2.17 重复使用您的 MySQL Shell 脚本

## 问题

您编写了 MySQL Shell 的代码，并希望以后能够重复使用。

## 解决方案

存储您的工作，并稍后使用 *\source* 命令加载文件。或者，设置这些文件作为启动脚本。

## 讨论

MySQL Shell 允许您重新使用您的代码。您可以通过使用 *\source* 命令或将您的脚本设置为在启动时执行来实现。让我们详细检查每一种可能性。

*\source* 命令适用于每种模式，并且与 *mysql* 客户端的 *\source* 命令类似工作。唯一的区别是，您的源文件应该与所选模式使用相同的语言编写。

例如，要加载我们在 Recipe 2.15 中讨论的 `CookbookCollection` 对象，可以输入以下命令：

```
 MySQL  cookbook  JS > `\source /cookbook/recipes/mysql_shell/CookbookCollection.js`
 MySQL  cookbook  JS > `CookbookCollection`
{
    "collection": null, 
    "find": <Function:find>, 
    "remove": <Function:remove>, 
    "save": <Function:save>
}
```

正如您所见，它立即可以用于使用。

同样，您可以导入我们在 Recipe 2.16 中讨论的 *generate_patients_data* 函数的定义：

```
 MySQL  cookbook  Py > `\source /cookbook/recipes/mysql_shell/generate_patients_data.py`
```

或者，在 SQL 模式下，我们可以加载任何 SQL 文件：

```
 MySQL  cookbook  SQL > `\source /cookbook/recipes/tables/patients.sql`
Query OK, 0 rows affected (0.0003 sec)
Query OK, 0 rows affected (0.0202 sec)
Query OK, 0 rows affected (0.0001 sec)
Query OK, 0 rows affected (0.0334 sec)
Query OK, 0 rows affected (0.0001 sec)
Query OK, 20 rows affected (0.0083 sec)

Records: 20  Duplicates: 0  Warnings: 0
```

如果您想在启动时执行脚本，您需要编辑 JavaScript 模式的 `mysqlshrc.js` 文件和 Python 模式的 `mysqlshrc.py` 文件，这些文件位于 MySQL Shell 用于搜索启动脚本的位置之一。这些文件可以位于以下任一位置：

+   全局配置文件，位于 Unix 上的 `/etc/mysql/mysqlsh/mysqlshrc.[js|py]`，或者 Windows 上的 `%PROGRAMDATA%\MySQL\mysqlsh\mysqlshrc.[js|py]`。

+   你的个人配置文件可以在 Unix 下的`$HOME/.mysqlsh/mysqlshrc.[js|py]`或 Windows 下的`%APPDATA%\MySQL\mysqlsh\mysqlshrc.[js|py]`找到。或者你可以指定变量`MYSQLSH_USER_CONFIG_HOME`并将文件`mysqlshrc.[js|py]`存储在其下。

+   目录`share/mysqlsh`可以在 MySQL Shell 安装根目录下找到，或者通过变量`MYSQLSH_HOME`指定。

`mysqlshrc.[js|py]`的格式与相应模式相同。因此，为了预加载`CookbookCollection`对象，你需要将`CookbookCollection.js`转换为模块，并导出我们的对象`CookbookCollection`：

```
exports.CookbookCollection = { 
  // Collection where the object is stored
  collection: null, 
  ...
```

然后你需要在文件`mysqlshrc.js`中加入两行：

```
sys.path = [...sys.path, '/cookbook/recipes/mysql_shell'];
const cookbook=require('CookbookCollectionModule.js')
```

在第一行，我们将包含我们模块的目录添加到模块搜索路径中。在第二行，我们导入了模块本身。`CookbookCollection`对象作为全局对象`cookbook`的属性可用。

```
 MySQL  cookbook  JS > `cookbook`
{
    "CookbookCollection": {
        "collection": null, 
        "find": <Function:find>, 
        "remove": <Function:remove>, 
        "save": <Function:save>
    }
}
```

###### 提示

MySQL Shell 使用 Node.js 模块。参考[Node.js 文档](https://nodejs.org/api/modules.html)了解如何在 MySQL Shell 中编写和使用 JavaScript 模块的详细信息。

*CookbookCollectionModule.js*位于`recipes`分发的*mysql_shell*目录中。

要在启动脚本中导入 Python 函数`generate_patients_data`，我们需要在我们的 Python 文件中添加指令*import mysqlsh*，因为在加载模块时，MySQL Shell 的全局对象尚不可用。我们还将更改该行：

```
patients=session.get_schema('cookbook').get_table('patients')
```

到

```
patients=mysqlsh.globals.session.get_schema('cookbook').get_table('patients')
```

否则 Python 会因为尚未定义名称`session`而失败。

我们将模块命名为`cookbook.py`以简洁明了起见。

在我们的函数中，我们使用当前目录中的本地路径到文件，因此我们将更改默认搜索路径为包含所有数据集的目录。为此，我们将导入模块`os`并使用其方法`chdir`。然后我们简单地导入模块`cookbook`。生成的`mysqlshrc.py`将有此代码。

```
sys.path.append("/home/sveta/bin/python-3.7.7/lib/python3.7/site-packages")
sys.path.append("/cookbook/recipes/mysql_shell")

import os
os.chdir('/cookbook/recipes/datasets')
import cookbook
```

模块*cookbook.py*位于`recipes`分发的*mysql_shell*目录中。

## 参见

关于使用外部脚本定制 MySQL Shell 的更多信息，请参阅 MySQL 用户参考手册中的[定制 MySQL Shell](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-customizing.html)。
