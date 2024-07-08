# 第四章：写 MySQL-Based 程序

# 4.0 引言

本章讨论了如何在通用编程语言的环境中使用 MySQL。它涵盖了基本的应用程序编程接口（API）操作，这些操作是后续章节中开发编程示例的基础。这些操作包括连接到 MySQL 服务器、执行语句和检索结果。

基于 MySQL 的客户端程序可以使用多种语言编写。本书涵盖了在表 4-1 中显示的语言和接口（有关获取接口软件的信息，请参见前言）：

表 4-1 本书涵盖的语言和接口

| 语言 | 接口 |
| --- | --- |
| Perl | Perl DBI |
| Ruby | Mysql2 gem |
| PHP | PDO |
| Python | DB API |
| Go | Go sql |
| Java | JDBC |

MySQL 客户端 API 提供以下功能，本章的各节详细介绍了每个功能：

连接到 MySQL 服务器、选择数据库和断开与服务器的连接

使用 MySQL 的每个程序必须首先与服务器建立连接。大多数程序还会选择一个默认数据库，并且表现良好的 MySQL 程序在完成后会关闭与服务器的连接。

检查错误

任何数据库操作都有可能失败。如果你知道何时发生以及原因，就可以采取适当的措施，比如终止程序或通知用户出现问题。

执行 SQL 语句和检索结果

连接到数据库服务器的目的是执行 SQL 语句。每个 API 都至少提供一种执行此操作的方式，以及处理语句结果的方法。

处理语句中的特殊字符和`NULL`值

数据值可以直接嵌入到语句字符串中。然而，某些字符如引号和反斜杠具有特殊含义，使用它们需要采取特定的预防措施。对于`NULL`值也是如此。如果处理不当，你的程序将生成错误的 SQL 语句或产生意外结果。如果将来自外部来源的数据合并到查询中，你的程序可能会成为 SQL 注入攻击的目标。大多数 API 通过使用占位符来避免这些问题，占位符在执行语句时以符号方式引用数据值，并单独提供这些值。API 将数据插入语句字符串中，正确编码任何特殊字符或`NULL`值。占位符也称为参数标记。

在结果集中标识`NULL`值

`NULL`值不仅在构造语句时是特殊的，在返回的结果中也是如此。每个 API 都提供了一种识别和处理它们的约定。

无论您使用哪种编程语言，都需要知道如何执行刚才描述的每个基本数据库 API 操作，因此本章展示了这五种语言中的每个操作。了解每个 API 如何处理给定操作应该有助于您更轻松地看到 API 之间的对应关系，并更好地理解接下来章节中显示的配方，即使它们是用您不太熟悉的语言编写的。（后续章节通常仅使用一种或两种语言实现配方。）

如果您只对特定 API 的一个语言感兴趣，看到每个配方都用几种语言编写可能会让人感到不知所措。如果是这样，请建议您仅阅读提供一般背景信息的介绍性配方部分，然后直接转到您感兴趣的语言部分。跳过其他语言；如果以后对它们产生兴趣，请回来再了解它们。

本章还讨论了以下几个主题，虽然它们不是 MySQL API 的直接组成部分，但有助于更轻松地使用它们：

写作库文件

随着您编写程序，您会发现重复执行某些操作。库文件可以封装这些操作的代码，使其可以在多个脚本中轻松执行，而无需在每个脚本中重复编写代码。这减少了代码重复，并使您的程序更具可移植性。本章展示了如何为每个 API 编写库文件，其中包括用于连接到服务器的例行操作，这是每个使用 MySQL 的程序必须执行的操作之一。后续章节将为其他操作开发额外的库程序。

获取连接参数的其他技术

早期关于连接到 MySQL 服务器的部分依赖于硬编码到代码中的连接参数。然而，还有其他（更好的）获取参数的方式，从将其存储在单独的文件中到允许用户在运行时指定。

为了避免在示例程序中手动输入，获取`recipes`源代码分发的副本（参见前言）。然后，当一个示例说“创建一个名为*xyz*的文件，其中包含以下信息…”时，你可以使用`recipes`分发中对应的文件。本章的大多数脚本位于*api*目录下；库文件位于*lib*目录下。

本章中用于示例的主要表格名为`profile`。它首次出现在 Recipe 4.4 中，如果您跳转章节并想知道它来自哪里，请了解这一点。还请参阅本章末尾关于将`profile`表重置为已知状态以在其他章节中使用的部分。

###### 注意

讨论的程序可以从命令行运行。有关如何调用本章涵盖的每种语言的程序的说明，请阅读`recipes`分发中的`cmdline.pdf`。

## 假设

要最有效地使用本章的材料，请确保满足以下要求：

+   为您计划使用的任何语言安装 MySQL 编程支持（请参阅前言）。

+   你应该已经为访问服务器设置了一个 MySQL 用户帐户和一个用于执行 SQL 语句的数据库。如食谱 1.1 所述，本书中的示例使用的是一个 MySQL 帐户，用户名和密码分别为`cbuser`和`cbpass`，我们将连接到运行在本地主机上的 MySQL 服务器，以访问名为`cookbook`的数据库。要创建帐户或数据库，请参阅该食谱中的说明。

+   此处的讨论展示了如何使用每种 API 语言执行数据库操作，但假定您对语言本身有基本的理解。如果某个食谱使用您不熟悉的编程构造，请查阅相关语言的通用参考书。

+   一些程序的正确执行可能需要您设置某些环境变量。关于如何设置环境变量的一般语法，请参阅配方分发中的`cmdline.pdf`（请参阅前言）。有关专门适用于库文件位置的环境变量的详细信息，请参阅食谱 4.3。

## MySQL 客户端 API 架构

本书涵盖的每个 MySQL 编程接口都使用了两级架构：

+   上层提供了独立于数据库的方法，以便以与使用 MySQL、PostgreSQL、Oracle 或其他任何数据库相同的便携方式访问数据库。

+   下层由一组驱动程序组成，每个驱动程序实现了单个数据库系统的详细信息。

这种两级架构使应用程序能够使用与任何特定数据库服务器的详细信息无关的抽象接口。这增强了程序的可移植性：要使用不同的数据库系统，只需选择不同的下层驱动程序。但是，完全的可移植性是难以实现的：

+   无论您使用哪个驱动程序，架构的上层提供的接口方法都是一致的，但仍然有可能编写只支持特定服务器的 SQL 语句。例如，MySQL 具有`SHOW`语句，用于提供关于数据库和表结构的信息，但在非 MySQL 服务器上使用`SHOW`可能会导致错误。

+   下层驱动程序经常扩展抽象接口，使访问数据库特定功能更方便。例如，Perl DBI 的 MySQL 驱动程序可以将最新的`AUTO_INCREMENT`值作为数据库句柄属性`$dbh->{mysql_insertid}`提供。这些功能使程序更易于编写，但更不可移植。要将程序用于另一个数据库系统，可能需要进行一些重写。

尽管这些因素在某种程度上影响可移植性，但两级架构的一般可移植特性为 MySQL 开发人员提供了显著的好处。

本书中使用的 API 共同的另一个特点是它们是面向对象的。无论您使用 Perl、Ruby、PHP、Python、Java 还是 Go 编写代码，连接到 MySQL 服务器的操作都会返回一个对象，使您能够以面向对象的方式处理语句。例如，当您连接到数据库服务器时，您会得到一个数据库连接对象，可以进一步与服务器交互。接口还提供语句、结果集、元数据等对象。

现在让我们看看如何使用这些编程接口执行最基本的 MySQL 操作：连接到服务器和断开连接。

# 4.1 连接，选择数据库和断开连接

## 问题

需要在连接到数据库服务器时建立连接，在完成后关闭连接。

## 解决方案

每个 API 都提供了连接和断开连接的例程。连接例程要求您提供指定 MySQL 服务器运行的主机和要使用的 MySQL 帐户的参数。您还可以选择一个默认数据库。

## 讨论

本节展示了执行大多数 MySQL 程序共同的一些基本操作的方法：

建立连接到 MySQL 服务器

使用 MySQL 的每个程序都要做到这一点，无论使用哪种 API。在指定连接参数方面的细节因 API 而异，有些 API 提供的灵活性更大。但是，有许多共同的参数，如运行服务器的主机，以及用于访问服务器的 MySQL 帐户的用户名和密码。

选择数据库

大多数 MySQL 程序选择一个默认数据库。

从服务器断开连接

每个 API 都提供了关闭打开连接的方法。最好在使用服务器后立即关闭连接。如果您的程序保持连接时间超过必要时间，服务器将无法释放为服务连接分配的资源。显式关闭连接也是首选。如果程序简单地终止，MySQL 服务器最终会注意到，但用户端的显式关闭使服务器能够在其端执行立即有序的关闭。

本节包括示例程序，展示如何使用每个 API 连接到服务器，选择`cookbook`数据库并断开连接。每个 API 的讨论还说明了如何在不选择任何默认数据库的情况下连接。如果您计划执行不需要默认数据库的语句，例如`SHOW VARIABLES`或`SELECT VERSION()`，或者编写一个允许用户在连接后指定数据库的程序，这可能是适用的情况。

###### 提示

这里显示的脚本使用`localhost`作为主机名。如果它们产生连接错误，指示找不到套接字文件，请尝试将`localhost`更改为本地主机的 TCP/IP 地址`127.0.0.1`。本提示适用于整本书。

### Perl

要在 Perl 中编写 MySQL 脚本，必须安装 DBI 模块以及 MySQL 特定的驱动程序模块 DBD::mysql。如果这些模块尚未安装，请参阅前言获取这些模块。

下面的 Perl 脚本*connect.pl*连接到 MySQL 服务器，选择`cookbook`作为默认数据库，并断开连接：

```
#!/usr/bin/perl
# connect.pl: connect to the MySQL server

use strict;
use warnings;
use DBI;

my $dsn = "DBI:mysql:host=localhost;database=cookbook";
my $dbh = DBI->connect ($dsn, "cbuser", "cbpass")
            or die "Cannot connect to server\n";
print "Connected\n";
$dbh->disconnect ();
print "Disconnected\n";
```

要尝试*connect.pl*，请在`recipes`分发的*api*目录下找到它，并从命令行运行。程序应打印两行，指示成功连接和断开连接：

```
$ `perl connect.pl`
Connected
Disconnected
```

在本节的其余部分中，我们将浏览代码并解释其工作原理。

###### 提示

如果连接到 MySQL 8.0 时出现`Access Denied`错误，请确保 DBD::MySQL 的版本与 MySQL 8.0 客户端库链接，或者使用身份验证插件`mysql_native_password`而不是默认的`caching_sha2_password`插件。我们在第 24.2 章中讨论了身份验证插件。

关于运行 Perl 程序的背景，请阅读配方分发中的*cmdline.pdf*（参见前言）。

`use` `strict`行打开严格变量检查，并导致 Perl 对任何在未声明的情况下使用的变量抱怨。这种预防措施有助于发现否则可能未被察觉的错误。

`use` `warnings`行打开警告模式，以便 Perl 对任何可疑的结构生成警告。我们的示例脚本没有问题，但养成在脚本开发过程中启用警告以捕获问题的习惯是个好主意。`use` `warnings`类似于指定 Perl `-w`命令行选项，但提供了更多控制要显示哪些警告的功能。（有关更多信息，请执行*perldoc* *warnings*命令。）

`use` `DBI`语句告诉 Perl 加载 DBI 模块。当脚本连接到数据库服务器时，显式加载 MySQL 驱动程序模块（DBD::mysql）是不必要的，DBI 在连接时会自行处理。

下面的两行通过设置数据源名称（DSN）和调用 DBI `connect()`方法来建立与 MySQL 的连接。`connect()`的参数是 DSN、MySQL 用户名和密码，以及您想指定的任何连接属性。DSN 是必需的。其他参数是可选的，尽管通常需要提供用户名和密码。

DSN 指定要使用的数据库驱动程序和指示连接位置的其他选项。对于 MySQL 程序，DSN 的格式为``DBI:mysql:*`options`*``。即使您未指定后续选项，DSN 中的第二个冒号也是必需的。

使用 DSN 组件如下：

+   第一个组件始终是`DBI`。大小写不敏感。

+   第二个组件告诉 DBI 要使用哪个数据库驱动程序，并且*区分大小写*。对于 MySQL，名称必须是`mysql`。

+   如果存在第三个组件，则是以分号分隔的*`name`*`=`*`value`* 对的列表，用于指定任意顺序的额外连接选项。对于我们的目的，最相关的两个选项是`host`和`database`，用于指定 MySQL 服务器运行的主机名和默认数据库。

根据这些信息，连接到本地主机*localhost*上的`cookbook`数据库的 DSN 如下所示：

```
DBI:mysql:host=localhost;database=cookbook
```

如果省略`host`选项，则其默认值为`localhost`。这两个 DSN 是等效的：

```
DBI:mysql:host=localhost;database=cookbook
DBI:mysql:database=cookbook
```

要选择没有默认数据库，请省略`database`选项。

`connect()` 调用的第二和第三个参数是您的 MySQL 用户名和密码。在密码之后，您还可以提供第四个参数来指定在发生错误时控制 DBI 行为的属性。如果没有属性，DBI 默认在发生错误时打印错误消息但不终止您的脚本。这就是为什么*connect.pl* 检查`connect()` 是否返回`undef`（表示失败）的原因：

```
my $dbh = DBI->connect ($dsn, "cbuser", "cbpass")
            or die "Cannot connect to server\n";
```

可能还有其他错误处理策略。例如，要告诉 DBI 在任何 DBI 调用中发生错误时终止脚本，请禁用`PrintError`属性，改为启用`RaiseError`：

```
my $dbh = DBI->connect ($dsn, "cbuser", "cbpass",
                        {PrintError => 0, RaiseError => 1});
```

然后您无需自行检查错误。权衡是，您也失去了决定程序如何从错误中恢复的能力。配方 4.2 进一步讨论了错误处理。

另一个常见的属性是`AutoCommit`，它设置事务的连接自动提交模式。MySQL 默认为新连接启用此功能，但从现在开始我们将显式设置初始连接状态：

```
my $dbh = DBI->connect ($dsn, "cbuser", "cbpass",
                        {PrintError => 0, RaiseError => 1, AutoCommit => 1});
```

如所示，`connect()` 的第四个参数是对属性名称/值对的哈希引用。编写此代码的另一种方法如下：

```
my $conn_attrs = {PrintError => 0, RaiseError => 1, AutoCommit => 1};
my $dbh = DBI->connect ($dsn, "cbuser", "cbpass", $conn_attrs);
```

使用您喜欢的任何风格。本书中的脚本使用`$conn_attr` 的哈希引用使`connect()` 调用更易读。

假设`connect()` 成功，它将返回一个包含连接状态信息的数据库句柄。 （在 DBI 术语中，对象的引用称为句柄。）稍后我们将看到其他句柄，如与特定语句相关联的语句句柄。本书中的 Perl DBI 脚本惯例上使用`$dbh`和`$sth`分别表示数据库和语句句柄。

#### 额外的连接参数

要在 Unix 上的*localhost*连接中指定套接字文件的路径，请在 DSN 中提供`mysql_socket`选项：

```
my $dsn = "DBI:mysql:host=localhost;database=cookbook"
          . ";mysql_socket=/var/tmp/mysql.sock";
```

要为非*localhost*（TCP/IP）连接指定端口号，请提供`port`选项：

```
my $dsn = "DBI:mysql:host=127.0.0.1;database=cookbook;port=3307";
```

### Ruby

要在 Ruby 中编写 MySQL 脚本，必须安装 Mysql2 gem。如果尚未安装此 gem，请参阅前言。

以下 Ruby 脚本*connect.rb*连接到 MySQL 服务器，选择`cookbook`作为默认数据库，并断开连接：

```
#!/usr/bin/ruby -w
# connect.rb: connect to the MySQL server

require "mysql2"

begin
  client = Mysql2::Client.new(:host => "localhost", 
                              :username => "cbuser",
                              :password => "cbpass",
                              :database => "cookbook")
  puts "Connected"
rescue => e
  puts "Cannot connect to server"
  puts e.backtrace
  exit(1)
ensure
  client.close()
  puts "Disconnected"
end
```

要尝试*connect.rb*，请在`recipes`分发的*api*目录下找到它，并从命令行运行。程序应打印两行，指示成功连接和断开连接：

```
$ `ruby connect.rb`
Connected
Disconnected
```

有关运行 Ruby 程序的背景，请阅读配方分发中的`cmdline.pdf`（请参阅前言）。

`-w`选项打开警告模式，以便 Ruby 对任何可疑构造生成警告。我们的示例脚本没有这样的构造，但养成在脚本开发过程中使用`-w`来捕捉问题的习惯是个好主意。

`require`语句告诉 Ruby 加载 Mysql2 模块。

要建立连接，请创建`Mysql2::Client`对象。将连接参数作为`new`方法的命名参数传递。

要选择无默认数据库，请省略`database`选项。

假设成功创建`Mysql2::Client`对象，则其将充当包含连接状态信息的数据库句柄。本书中的 Ruby 脚本通常使用`client`表示数据库句柄对象。

如果`new()`方法失败，则会引发异常。要处理异常，请将可能失败的语句放入`begin`块中，并使用包含错误处理代码的`rescue`子句。在脚本的顶层（即任何`begin`块之外）发生的异常将被默认异常处理程序捕获，后者会打印堆栈跟踪并退出。配方 4.2 进一步讨论了错误处理。

#### 其他连接参数

要为 Unix 上的*localhost*连接指定套接字文件路径，请为`new`方法提供`socket`选项：

```
client = Mysql2::Client.new(:host => "localhost",
                            :socket => "/var/tmp/mysql.sock",
                            :username => "cbuser",
                            :password => "cbpass",
                            :database => "cookbook")
```

要为非*localhost*（TCP/IP）连接指定端口号，请提供`port`选项：

```
client = Mysql2::Client.new(:host => "127.0.0.1",
                            :port => 3307,
                            :username => "cbuser",
                            :password => "cbpass",
                            :database => "cookbook")
```

### PHP

要编写使用 MySQL 的 PHP 脚本，您的 PHP 解释器必须已编译有 MySQL 支持。如果您的脚本无法连接到 MySQL 服务器，请查看随 PHP 分发的说明，了解如何启用 MySQL 支持。

PHP 实际上有多个扩展程序可以使用 MySQL，例如`mysql`，原始且现在已弃用的 MySQL 扩展；`mysqli`，改进的 MySQL 扩展；以及最近的 MySQL PDO（PHP 数据对象）接口驱动程序。本书中的 PHP 脚本使用 PDO。如果尚未安装 PHP 和 PDO，请参阅前言。

PHP 脚本通常是为与 Web 服务器一起使用而编写的。我假设如果您以这种方式使用 PHP，您可以将 PHP 脚本复制到服务器的文档树中，从浏览器请求它们，它们将执行。例如，如果您在主机 *localhost* 上运行 Apache 作为 Web 服务器，并且您在 Apache 文档树的顶层安装了名为 *myscript.php* 的 PHP 脚本，您应该能够通过请求以下 URL 来访问该脚本：

```
http://localhost/myscript.php
```

本书使用 *.php* 扩展名（后缀）作为 PHP 脚本文件名，因此您的 Web 服务器必须配置为识别 *.php* 扩展名。否则，当您从浏览器请求 PHP 脚本时，服务器只会发送脚本的文字内容，这将显示在您的浏览器窗口中。您不希望发生这种情况，特别是如果脚本包含连接到 MySQL 的用户名和密码。

PHP 脚本通常是 HTML 和 PHP 代码的混合体，其中 PHP 代码嵌入在特殊的 `<?php` 和 `?>` 标签之间。以下是一个示例：

```
<html>
<head><title>A simple page</title></head>
<body>
<p>
<?php
  print ("I am PHP code, hear me roar!");
?>
</p>
</body>
</html>
```

为了在完全由 PHP 代码组成的示例中简洁起见，通常会省略包围的 `<?php` 和 `?>` 标签。如果您在 PHP 示例中看不到标签，请假设 `<?php` 和 `?>` 包围着显示的整个代码块。在 HTML 和 PHP 代码之间切换的示例中会包含标签，以明确显示哪些是 PHP 代码，哪些不是。

PHP 可以配置为识别 <q>short</q> 标签，写作 `<?` 和 `?>`。本书假设您未启用短标签并且不使用它们。

以下 PHP 脚本，*connect.php*，连接到 MySQL 服务器，选择 `cookbook` 作为默认数据库，并断开连接：

```
<?php
# connect.php: connect to the MySQL server

try
{
  $dsn = "mysql:host=localhost;dbname=cookbook";
  $dbh = new PDO ($dsn, "cbuser", "cbpass");
  print ("Connected\n");
}
catch (PDOException $e)
{
  die ("Cannot connect to server\n");
}
$dbh = NULL;
print ("Disconnected\n");
?>
```

要尝试 *connect.php*，请在 `recipes` 分发的 *api* 目录下找到它，将其复制到您的 Web 服务器文档树中，并使用浏览器请求它。或者，如果您有独立版本的 PHP 解释器用于命令行使用，则可以直接执行脚本：

```
$ `php connect.php`
Connected
Disconnected
```

有关运行 PHP 程序的背景，请阅读 `cmdline.pdf` 在 recipes 分发中（请参阅 前言）。

`$dsn` 是数据源名称 (DSN)，指示如何连接到数据库服务器。它具有以下一般语法：

```
*`driver`*:*`name=value`*;*`name=value`* ...
```

*`driver`* 值是 PDO 驱动程序类型。对于 MySQL，这是 `mysql`。

在驱动程序名称之后，以分号分隔的 *`name`*`=`*`value`* 对指定连接参数，顺序任意。对我们来说，两个最相关的选项是 `host` 和 `dbname`，用于指定 MySQL 服务器运行的主机名和默认数据库。要选择没有默认数据库，请省略 `dbname` 选项。

要建立连接，调用`new` `PDO()`类构造函数，并传递适当的参数。DSN 是必需的。其他参数是可选的，尽管通常需要提供用户名和密码。如果连接尝试成功，`new` `PDO()`将返回一个数据库句柄对象，用于访问其他与 MySQL 相关的方法。本书中的 PHP 脚本惯例上使用`$dbh`表示数据库句柄。

如果连接尝试失败，PDO 会引发一个异常。为了处理这种情况，将连接尝试放在一个`try`块中，并使用一个`catch`块包含错误处理代码，或者让异常终止你的脚本。4.2 小节进一步讨论了错误处理。

要断开连接，请将数据库句柄设置为`NULL`。没有显式的断开调用。

#### 额外的连接参数

要在 Unix 上指定本地主机连接的套接字文件路径，请在 DSN 中提供`unix_socket`选项：

```
$dsn = "mysql:host=localhost;dbname=cookbook"
         . ";unix_socket=/var/tmp/mysql.sock";
```

要为非本地主机（TCP/IP）连接指定端口号，请提供`port`选项：

```
$dsn = "mysql:host=127.0.0.1;database=cookbook;port=3307";
```

### Python

要在 Python 中编写 MySQL 程序，必须安装一个模块，该模块为 Python DB API（即 Python 数据库 API 规范 v2.0，PEP 249）提供 MySQL 连接。本书使用 MySQL Connector/Python。如果尚未安装，请参阅前言获取。

要使用 DB API，在要使用的数据库驱动模块中导入它（对于使用 Connector/Python 的 MySQL 程序来说，这是`mysql.connector`）。然后通过调用驱动程序的`connect()`方法创建一个数据库连接对象。该对象提供了访问其他 DB API 方法的途径，例如服务于数据库服务器的`close()`方法。

以下 Python 脚本*connect.py*连接到 MySQL 服务器，选择`cookbook`作为默认数据库，并断开连接：

```
#!/usr/bin/python3
# connect.py: connect to the MySQL server

import mysql.connector

try:
  conn = mysql.connector.connect(database="cookbook",
                                 host="localhost",
                                 user="cbuser",
                                 password="cbpass")
  print("Connected")
except:
  print("Cannot connect to server")
else:
  conn.close()
  print("Disconnected")
```

要尝试*connect.py*，请在`recipes`发行版的*api*目录下找到它，并从命令行运行。程序应该打印两行指示成功连接和断开连接：

```
$ `python3 connect.py`
Connected
Disconnected
```

关于运行 Python 程序的背景，请阅读`recipes`发行版中的`cmdline.pdf`（参见前言）。

`import`行告诉 Python 加载`mysql.connector`模块。然后脚本尝试通过调用`connect()`建立与 MySQL 服务器的连接以获取连接对象。本书中的 Python 脚本惯例上使用`conn`表示连接对象。

如果 `connect()` 方法失败，Connector/Python 将引发异常。要处理异常，请将可能失败的语句放在 `try` 语句中，并使用包含错误处理代码的 `except` 子句。在脚本的顶层（即在任何 `try` 语句之外）发生的异常会被默认的异常处理程序捕获，该程序会打印堆栈跟踪并退出。Recipe 4.2 进一步讨论了错误处理。

`else` 子句包含在 `try` 子句未产生异常时执行的语句。这里用于关闭成功打开的连接。

因为 `connect()` 调用使用了命名参数，它们的顺序并不重要。如果从 `connect()` 调用中省略 `host` 参数，则其默认值为 `127.0.0.1`。要选择没有默认数据库，请省略 `database` 参数或传递 `""`（空字符串）或 `None` 作为 `database` 值。

另一种连接方法是使用 Python 字典指定参数，并将字典传递给 `connect()`：

```
conn_params = {
  "database": "cookbook",
  "host": "localhost",
  "user": "cbuser",
  "password": "cbpass",
}
conn = mysql.connector.connect(**conn_params)
print("Connected")
```

从现在开始，本书通常会使用这种风格。

#### 附加连接参数

要在 Unix 上指定本地主机连接的套接字文件路径，请省略 `host` 参数并提供 `unix_socket` 参数：

```
conn_params = {
  "database": "cookbook",
  "unix_socket": "/var/tmp/mysql.sock",
  "user": "cbuser",
  "password": "cbpass",
}
conn = mysql.connector.connect(**conn_params)
print("Connected")
```

要为 TCP/IP 连接指定端口号，请包含 `host` 参数并提供整数值 `port` 参数：

```
conn_params = {
  "database": "cookbook",
  "host": "127.0.0.1",
  "port": 3307,
  "user": "cbuser",
  "password": "cbpass",
}
conn = mysql.connector.connect(**conn_params)
```

### Go

要在 Go 中编写 MySQL 程序，必须安装 Go SQL 驱动程序。本书使用 [Go-MySQL-Driver](https://github.com/go-sql-driver/mysql)。如果尚未安装，请安装 *Git*，然后执行以下命令。

```
$ go get -u github.com/go-sql-driver/mysql
```

要使用 Go SQL 接口，导入 `database/sql` 包和您的驱动程序包。然后通过调用 `sql.Open()` 函数创建数据库连接对象。此对象提供访问其他 `database/sql` 包函数的功能，如 `db.Close()` 关闭与数据库服务器的连接。我们还使用 `defer` 语句调用 `db.Close()`，以确保函数调用在程序执行的后续阶段执行。在本章中，您将看到这种用法。

###### 提示

Go 包 `database/sql` 和 Go-MySQL-Driver 支持上下文取消。这意味着您可以取消数据库操作（如运行查询），如果取消上下文。要使用此功能，您需要调用 `sql` 接口的支持上下文的函数。出于简洁起见，在本章的示例中，我们不会使用 `Context`。当讨论事务处理时，我们将在 Recipe 20.9 中展示使用 `Context` 的示例。

下面的 Go 脚本，*connect.go*，连接到 MySQL 服务器，选择 `cookbook` 作为默认数据库，并断开连接：

```
// connect.go: connect to MySQL server
package main

import (
    "database/sql"
    "fmt"
    "log"

    _ "github.com/go-sql-driver/mysql"
)

func main() {

    db, err := sql.Open("mysql", "cbuser:cbpass@tcp(127.0.0.1:3306)/cookbook")

    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    err = db.Ping()

    if err != nil {
        log.Fatal(err)
    }

    fmt.Println("Connected!")
}
```

要尝试 *connect.go*，请在 `recipes` 发行版的 *api/01_connect* 目录下找到它，并从命令行运行。程序应打印一行表示已连接：

```
$ `go run connect.go`
Connected!
```

`import`语句告诉 Go 加载`go-sql-driver/mysql`包。然后脚本通过调用`sql.Open()`验证连接参数并获取连接对象。*还没有建立 MySQL 连接！*

如果`sql.Open()`方法失败，`go-sql-driver/mysql`会返回一个错误。要处理错误，将其存储在一个变量中（在我们的例子中为`err`），并使用包含错误处理代码的`if`块。第 4.2 节进一步讨论了错误处理。

`db.Ping()`调用建立数据库连接。只有在这一刻我们才能说成功连接到 MySQL 服务器。

#### 附加连接参数

要在 Unix 上指定本地主机连接的套接字文件路径，请在 DSN 中省略`tcp`参数并提供`unix`参数：

```
// connect_socket.go : Connect MySQL server using socket
package main

import (
    "database/sql"
    "fmt"
    "log"

    _ "github.com/go-sql-driver/mysql"
)

func main() {
    db, err := sql.Open("mysql","cbuser:cbpass@unix(/tmp/mysql.sock)/cookbook")
    defer db.Close()

    if err != nil {
        log.Fatal(err)
    }

    var user string
    err = db.QueryRow("SELECT USER()").Scan(&user)

    if err != nil {
        log.Fatal(err)
    }

    fmt.Println("Connected User:", user, "via MySQL socket")
}
```

运行这个程序：

```
$ `go run connect_socket.go`
Connected User: cbuser@localhost via MySQL socket
```

要指定 TCP/IP 连接的端口号，请在 DSN 中包括`tcp`参数并提供一个整数值`port`端口号：

```
// connect_tcpport.go : Connect MySQL server using tcp port number
package main

import (
	"database/sql"
	"fmt"
	"log"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	db, err := sql.Open("mysql",
	"cbuser:cbpass@tcp(127.0.0.1:3306)/cookbook?charset=utf8mb4")

	if err != nil {
		log.Fatal(err)
	}

	var user string
	err2 := db.QueryRow("SELECT USER()").Scan(&user)

	if err2 != nil {
		log.Fatal(err2)
	}

	fmt.Println("Connected User:", user, "via MySQL TCP/IP localhost on port 3306")
}
```

运行这个程序：

```
$  `go run connect_tcpport.go`
Connected User: cbuser@localhost via MySQL TCP/IP localhost on port 3306
```

Go 接受这种形式的 DSN（数据源名称）：

```
[username[:password]@][protocol[(address)]]/dbname[?param1=value1&..&paramN=valueN]
```

其中`protocol`可以是`tcp`或`unix`。

完整形式的 DSN 如下所示：

```
username:password@protocol(address)/dbname?param=value
```

### Java

Java 中的数据库程序使用 JDBC 接口，以及要访问的特定数据库引擎的驱动程序。也就是说，JDBC 架构提供了一个通用接口，与数据库特定的驱动程序结合使用。

Java 编程需要 Java 开发工具包（JDK），您必须将`JAVA_HOME`环境变量设置为安装 JDK 的位置。要编写基于 MySQL 的 Java 程序，还需要一个特定于 MySQL 的 JDBC 驱动程序。本书中使用 MySQL Connector/J。如果未安装，请参阅前言获取它。有关获取 JDK 和设置`JAVA_HOME`的信息，请阅读 recipes 发行版中的`cmdline.pdf`（请参阅前言）。

下面的 Java 程序*Connect.java*连接到 MySQL 服务器，选择`cookbook`作为默认数据库，并断开连接：

```
// Connect.java: connect to the MySQL server

import java.sql.*;

public class Connect {

  public static void main (String[] args) {
    Connection conn = null;
    String url = "jdbc:mysql://localhost/cookbook";
    String userName = "cbuser";
    String password = "cbpass";

    try {
      conn = DriverManager.getConnection (url, userName, password);
      System.out.println("Connected");
    } catch (Exception e) {
      System.err.println("Cannot connect to server");
      System.exit (1);
    }

    if (conn != null) {
      try {
        conn.close();
        System.out.println("Disconnected");
      } catch (Exception e) { /* ignore close errors */ }
    }
  }
}
```

要尝试*Connect.java*，请找到它在`recipes`发行版的*api*目录下，编译并执行它。`class`语句指示程序的名称，在这种情况下是`Connect`。包含程序的文件名必须与此名称匹配，并包括*.java*扩展名，因此程序的文件名是*Connect.java*。使用*javac*编译程序：

```
$ `javac Connect.java`
```

如果你喜欢不同的 Java 编译器，请将其名称替换为*javac*。

Java 编译器生成编译的字节码，生成名为*Connect.class*的类文件。使用*java*程序运行该类文件（不带*.class*扩展名）。程序应打印两行表示成功连接和断开连接：

```
$ `java Connect`
Connected
Disconnected
```

在示例程序能够编译和运行之前，您可能需要设置您的 `CLASSPATH` 环境变量。`CLASSPATH` 的值应至少包括当前目录（`.`）和 Connector/J JDBC 驱动程序的路径。有关运行 Java 程序或设置 `CLASSPATH` 的背景，请阅读配方分发中的 `cmdline.pdf`（参见 前言）。

###### 提示

从 Java 11 开始，可以跳过 *javac* 调用，直接运行单文件程序，如下所示：

```
$ `java Connect.java`
Connected
Disconnected
```

`import` `java.sql.*` 语句引用了类和接口，这些类和接口提供了访问用于管理与数据库服务器交互的不同数据类型的功能。所有 JDBC 程序都需要这些。

要连接到服务器，请调用 `DriverManager.getConnection()` 来初始化连接并获取一个 `Connection` 对象，该对象维护有关连接状态的信息。本书中的 Java 程序通常使用 `conn` 表示连接对象。

`DriverManager.getConnection()` 接受三个参数：描述连接位置和要使用的数据库的 URL 字符串，MySQL 用户名和密码。URL 字符串的格式如下：

```
jdbc:*`driver`*://*`host_name`*/*`db_name`*
```

此格式遵循 Java 的约定，用于连接网络资源的 URL 必须以协议标识符开头。对于 JDBC 程序，协议是 `jdbc`，还需要一个子协议标识符来指定驱动程序名称（MySQL 程序使用 `mysql`）。连接 URL 的许多部分都是可选的，但是协议和子协议标识符是必需的。如果省略 *`host_name`*，默认主机值为 `localhost`。如果不选择默认数据库，则应省略数据库名称。但无论如何都不应省略任何斜杠。例如，要连接到本地主机而不选择默认数据库，URL 如下：

```
jdbc:mysql:///
```

在 JDBC 中，不要测试返回值指示错误的方法调用。相反，提供处理程序以在抛出异常时调用。第 4.2 节 进一步讨论了错误处理。

一些 JDBC 驱动程序（包括 Connector/J）允许您在 URL 的末尾指定用户名和密码作为参数。在这种情况下，可以省略 `getConnection()` 调用的第二个和第三个参数。使用这种 URL 样式，可以像下面这样编写建立连接的代码：

```
// connect using username and password included in URL
Connection conn = null;
String url = "jdbc:mysql://localhost/cookbook?user=cbuser&password=cbpass";

try
{
  conn = DriverManager.getConnection (url);
  System.out.println ("Connected");
}
```

分隔 `user` 和 `password` 参数的字符应为 `&`，而不是 `;`。

#### 额外的连接参数

Connector/J 不原生支持 Unix 域套接字文件连接，因此即使主机名为 *localhost* 的连接也是通过 TCP/IP 进行的。要指定显式端口号，请在连接 URL 中的主机名后添加 `:`*`port_num`*：

```
String url = "jdbc:mysql://127.0.0.1:3307/cookbook";
```

但是，您可以使用第三方库来支持通过套接字进行连接。有关详细信息，请参阅 [使用 Unix 域套接字连接](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-unix-socket.html) 用户参考手册页面。

# 4.2 检查错误

## 问题

程序出了问题，但你不知道是什么。

## 解决方案

每个人在使程序正常工作方面都会遇到问题。但如果你不通过检查错误来预期问题，那么工作将变得更加困难。添加一些错误检查代码，让你的程序能帮助你找出问题所在。

## 讨论

在完成 Recipe 4.1 后，你知道如何连接到 MySQL 服务器。了解如何检查错误并从 API 中检索特定的错误信息也是一个好主意，接下来我们会涵盖这些内容。你可能急于做更有趣的事情（如执行语句并获取结果），但错误检查基本上是非常重要的。程序有时会失败，特别是在开发过程中，如果你不能确定失败的原因，那么你就是在盲目操作。通过检查错误来规划失败，以便你可以采取适当的措施。

当发生错误时，MySQL 提供了三个值：

+   一个特定于 MySQL 的错误编号

+   MySQL 特定的描述性文本错误消息

+   根据 ANSI 和 ODBC 标准定义的五字符 SQLSTATE 错误代码

本示例展示了如何访问这些信息。示例程序有意设计成失败的，以便执行错误处理代码。这就是为什么它们尝试使用用户名`baduser`和密码`badpass`连接。

###### 提示

一种不特定于任何 API 的常规调试辅助工具是使用可用的日志。检查 MySQL 服务器的一般查询日志以查看服务器接收到的语句。 （这需要启用日志记录；参见 Recipe 22.3。）一般查询日志可能显示你的程序未构建你期望的 SQL 语句字符串。类似地，如果在 Web 服务器下运行脚本时失败，请检查 Web 服务器的错误日志。

### Perl

DBI 模块提供了两个属性来控制 DBI 方法调用失败时的处理方式：

+   如果启用了`PrintError`，DBI 会使用`warn()`打印错误消息。

+   `RaiseError`如果启用，会导致 DBI 使用`die()`打印错误消息。这将终止你的脚本。

默认情况下，启用了`PrintError`，而`RaiseError`未启用，因此在打印消息后脚本继续执行。可以在`connect()`调用中指定一个或两个属性。将属性设置为 1 或 0 可分别启用或禁用它们。要指定一个或两个属性，请将它们作为第四个参数传递给`connect()`调用的哈希引用。

以下代码仅设置`AutoCommit`属性，并使用默认设置处理错误属性。如果`connect()`调用失败，将产生警告消息，但脚本会继续执行：

```
my $conn_attrs = {AutoCommit => 1};
my $dbh = DBI->connect ($dsn, "baduser", "badpass", $conn_attrs);
```

如果连接尝试失败，你真的不能做什么，因此在 DBI 打印消息后退出通常是明智的选择：

```
my $conn_attrs = {AutoCommit => 1};
my $dbh = DBI->connect ($dsn, "baduser", "badpass", $conn_attrs)
            or exit;
```

要打印自己的错误消息，请禁用`RaiseError`并禁用`PrintError`。然后自行测试 DBI 方法调用的结果。当方法失败时，`$DBI::err`、`$DBI::errstr`和`$DBI::state`变量分别包含 MySQL 错误编号、描述性错误字符串和 SQLSTATE 值：

```
my $conn_attrs = {PrintError => 0, AutoCommit => 1};
my $dbh = DBI->connect ($dsn, "baduser", "badpass", $conn_attrs)
            or die "Connection error: "
                   . "$DBI::errstr ($DBI::err/$DBI::state)\n";
```

如果没有错误发生，`$DBI::err`为 0 或`undef`，`$DBI::errstr`为空字符串或`undef`，`$DBI::state`为空或`00000`。

当检查错误时，立即在调用设置它们的 DBI 方法之后访问这些变量。如果在使用它们之前调用另一个方法，DBI 会重置它们的值。

如果打印自己的消息，使用默认设置（启用`PrintError`，禁用`RaiseError`）并不那么有用。DBI 会自动打印一条消息，然后你的脚本会打印自己的消息。这是多余的，也会让使用脚本的人感到困惑。

如果启用了`RaiseError`，可以调用 DBI 方法而无需检查指示错误的返回值。如果方法失败，DBI 会打印一个错误并终止脚本。如果方法返回，可以假定它成功了。这对于脚本编写者来说是最简单的方法：让 DBI 做所有的错误检查！但是，如果同时启用了`PrintError`和`RaiseError`，DBI 可能会连续调用`warn()`和`die()`，导致错误消息被打印两次。为了避免这个问题，在启用`RaiseError`时禁用`PrintError`：

```
my $conn_attrs = {PrintError => 0, RaiseError => 1, AutoCommit => 1};
my $dbh = DBI->connect ($dsn, "baduser", "badpass", $conn_attrs);
```

本书一般采用这种方法。如果你不想启用`RaiseError`来进行自动错误检查，也不想完全自己进行检查，可以采用混合方法。单独的句柄具有可以选择启用或禁用的`PrintError`和`RaiseError`属性。例如，你可以在调用`connect()`时全局启用`RaiseError`，然后在每个句柄上选择性地禁用它。

假设脚本从命令行参数中读取用户名和密码，然后在用户输入要执行的语句时循环。在这种情况下，如果连接失败，你可能希望 DBI 自动终止并打印错误消息（在这种情况下，不能继续到语句执行循环）。然而，在连接之后，你不希望脚本仅因为用户输入了语法无效的语句而退出。相反，请打印错误消息并循环以获取下一个语句。以下代码展示了如何做到这一点。示例中使用的`do()`方法执行一个语句并返回`undef`以指示错误：

```
my $user_name = shift (@ARGV);
my $password = shift (@ARGV);
my $conn_attrs = {PrintError => 0, RaiseError => 1, AutoCommit => 1};
my $dbh = DBI->connect ($dsn, $user_name, $password, $conn_attrs);
$dbh->{RaiseError} = 0; # disable automatic termination on error
print "Enter statements to execute, one per line; terminate with Control-D\n";
while (<>)              # read and execute queries
{
  $dbh->do ($_) or warn "Statement failed: $DBI::errstr ($DBI::err)\n";
}
```

如果启用了`RaiseError`，可以在`eval`块中执行代码来捕获错误，而不终止程序。如果发生错误，`eval`会在`$@`变量中返回一条消息：

```
eval
{
  # statements that might fail go here...
};
if ($@)
{
  print "An error occurred: $@\n";
}
```

这种`eval`技术通常用于执行事务（见 20.4 章节）。

结合`eval`和`RaiseError`使用时与仅使用`RaiseError`时有所不同：

+   错误只终止`eval`块，而不是整个脚本。

+   任何错误都会终止`eval`块，而`RaiseError`仅适用于与 DBI 相关的错误。

当您使用带有启用`RaiseError`的`eval`时，请禁用`PrintError`。否则，在某些 DBI 版本中，错误可能仅会导致调用`warn()`，而不会像您期望的那样终止`eval`块。

除了使用错误处理属性`PrintError`和`RaiseError`外，还可以使用 DBI 的跟踪机制获取有关脚本执行的大量信息。调用`trace()`方法并传递指定的跟踪级别参数。级别 1 到 9 使跟踪输出更加详细，级别 0 禁用跟踪：

```
DBI->trace (1); # enable tracing, minimal output
DBI->trace (3); # elevate trace level
DBI->trace (0); # disable tracing
```

如果需要，单独的数据库和语句句柄也有`trace()`方法，因此您可以将跟踪限制在单个句柄上。

跟踪输出通常发送到您的终端（或者在 Web 脚本的情况下，发送到 Web 服务器的错误日志）。要将跟踪输出写入特定文件，请提供第二个参数指定文件名：

```
DBI->trace (1, "/tmp/trace.out");
```

如果跟踪文件已经存在，则其内容不会首先被清除；跟踪输出将追加到末尾。在开发脚本时打开跟踪文件时要小心，但在将脚本投入生产时忘记禁用跟踪。最终你会发现遗憾的是跟踪文件变得非常大。或者更糟糕的是，文件系统将填满，而你却不知道原因！

### Ruby

Ruby 通过引发异常来表示错误，Ruby 程序通过在`begin`块的`rescue`子句中捕获异常来处理错误。当 Ruby Mysql2 方法失败时会引发异常，并通过`Mysql2::Error`对象提供错误信息。要获取 MySQL 错误编号、错误消息和 SQLSTATE 值，请访问该对象的`errno`、`message`和`sql_state`方法。以下示例展示了如何在 Ruby 脚本中捕获异常并访问错误信息：

```
begin
  client = Mysql2::Client.new(:host => "localhost",
                              :username => "baduser",
                              :password => "badpass",
                              :database => "cookbook")
  puts "Connected"
rescue Mysql2::Error => e
  puts "Cannot connect to server"
  puts "Error code: #{e.errno}"
  puts "Error message: #{e.message}"
  puts "Error SQLSTATE: #{e.sql_state}"
  exit(1)
ensure
  client.close()s
end
```

### PHP

如果连接失败，`new PDO()`构造函数会引发异常，但其他 PDO 方法默认通过它们的返回值指示成功或失败。为了导致所有 PDO 方法在错误时引发异常，请使用成功连接尝试的数据库句柄设置错误处理模式。这样可以统一处理所有 PDO 错误，而无需检查每个调用的结果。以下示例展示了如何设置错误模式（如果连接尝试成功）以及如何处理异常（如果连接失败）：

```
try
{
  $dsn = "mysql:host=localhost;dbname=cookbook";
  $dbh = new PDO ($dsn, "baduser", "badpass");
  $dbh->setAttribute (PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
  print ("Connected\n");
}
catch (PDOException $e)
{
  print ("Cannot connect to server\n");
  print ("Error code: " . $e->getCode () . "\n");
  print ("Error message: " . $e->getMessage () . "\n");
}
```

当 PDO 引发异常时，结果的`PDOException`对象提供错误信息。`getCode()`方法返回 SQLSTATE 值。`getMessage()`方法返回包含 SQLSTATE 值、MySQL 错误编号和错误消息的字符串。

当发生错误时，数据库和语句句柄还提供信息。对于任一类型的句柄，`errorCode()`返回 SQLSTATE 值，而`errorInfo()`返回一个包含 SQLSTATE 值和特定于驱动程序的错误代码和消息的三元素数组。对于 MySQL，后两个值分别是错误号和消息字符串。以下示例演示了如何从异常对象和数据库句柄获取信息：

```
try
{
  $dbh->query ("SELECT"); # malformed query
}
catch (PDOException $e)
{
  print ("Cannot execute query\n");
  print ("Error information using exception object:\n");
  print ("SQLSTATE value: " . $e->getCode () . "\n");
  print ("Error message: " . $e->getMessage () . "\n");

  print ("Error information using database handle:\n");
  print ("Error code: " . $dbh->errorCode () . "\n");
  $errorInfo = $dbh->errorInfo ();
  print ("SQLSTATE value: " . $errorInfo[0] . "\n");
  print ("Error number: " . $errorInfo[1] . "\n");
  print ("Error message: " . $errorInfo[2] . "\n");
}
```

### Python

Python 通过引发异常信号错误，并通过在`try`语句的`except`子句中捕获异常来处理错误。要获取特定于 MySQL 的错误信息，请命名一个异常类，并提供一个变量来接收信息。以下是一个示例：

```
conn_params = {
  "database": "cookbook",
  "host": "localhost",
  "user": "baduser",
  "password": "badpass"
}

try:
  conn = mysql.connector.connect(**conn_params)
  print("Connected")
except mysql.connector.Error as e:
  print("Cannot connect to server")
  print("Error code: %s" % e.errno)
  print("Error message: %s" % e.msg)
  print("Error SQLSTATE: %s" % e.sqlstate)
```

如果发生异常，异常对象的`errno`、`msg`和`sqlstate`成员包含错误号、错误消息和 SQLSTATE 值。请注意，访问`Error`类是通过驱动程序模块名。

### Go

Go 不支持异常。相反，其多返回值使得在需要时传递错误变得容易。要处理 Go 中的错误，请将类型为`Error`的返回值存储在变量中（这里我们使用变量名`err`），并相应地处理它。为了处理错误，Go 提供了`defer`语句、`Panic()`和`Recover()`内置函数。

表 4-2\. Go 中的错误处理

| 函数或语句 | 含义 |
| --- | --- |
| `defer` | 将语句的执行延迟到调用函数返回之前。 |
| `Panic()` | 调用函数的正常执行停止，所有延迟函数都被执行，然后函数返回一个对栈上的 panic 调用。进程继续执行。最后，程序崩溃。 |
| `Recover()` | 允许在恐慌的 goroutine 中重新获得控制，以便程序不会崩溃并继续执行。仅在延迟函数中有效。如果在非延迟函数中调用，什么也不做并返回`nil`。 |

```
// mysql_error.go : MySQL error handling
package main

import (
    "database/sql"
    "log"
    "fmt"

    _ "github.com/go-sql-driver/mysql"
)

var actor string

func main() {

    db, err := sql.Open("mysql", "cbuser:cbpass!@tcp(127.0.0.1:3306)/cookbook")
    defer db.Close()

    if err != nil {
        log.Fatal(err)
    }

    err = db.QueryRow("SELECT actor FROM actors where actor='Dwayne Johnson'").↩
          Scan(&actor)
    if err != nil {
	    if err == sql.ErrNoRows {
		    fmt.Print("There were no rows, but otherwise no error occurred")
	    } else {
		    fmt.Println(err.Error())
	    }
    }
    fmt.Println(actor)
}
```

如果发生错误，函数将返回一个`error`类型的对象。它的函数`Error()`返回 MySQL 错误代码和`Go-MySQL-Driver`引发的错误消息。

函数`QueryRow()`与后续的`Scan()`调用有一个特殊情况。默认情况下，如果没有错误，`Scan()`返回`nil`，如果有错误，则返回`error`。但是，如果查询成功运行但没有返回任何行，此函数将返回`sql.ErrNoRows`。

### Java

Java 程序通过捕获异常来处理错误。为了做最少的工作，打印堆栈跟踪以通知用户问题所在的位置：

```
try
{
  /* ... some database operation ... */
}
catch (Exception e)
{
  e.printStackTrace ();
}
```

堆栈跟踪显示问题的位置，但不一定显示问题是什么。此外，除了程序开发人员外，它可能没有意义。为了更具体地说明，打印与异常关联的错误消息和代码：

+   所有 `Exception` 对象都支持 `getMessage()` 方法。JDBC 方法可能会抛出 `SQLException` 对象；这些对象与 `Exception` 对象类似，但还支持 `getErrorCode()` 和 `getSQLState()` 方法。`getErrorCode()` 和 `getMessage()` 返回 MySQL 特定的错误编号和消息字符串，`getSQLState()` 返回一个包含 SQLSTATE 值的字符串。

+   一些方法会生成 `SQLWarning` 对象，以提供关于非致命警告的信息。`SQLWarning` 是 `SQLException` 的子类，但警告会积累在一个列表中，而不是立即抛出。它们不会中断你的程序，你可以在方便时打印它们。

下面的示例程序，*Error.java*，展示了如何通过打印所有可用的错误信息来访问错误消息。它尝试连接到 MySQL 服务器，并在连接失败时打印异常信息。然后执行一个语句，并在语句执行失败时打印异常和警告信息：

```
// Error.java: demonstrate MySQL error handling

import java.sql.*;

public class Error {
  public static void main(String[] args) {
    Connection conn = null;
    String url = "jdbc:mysql://localhost/cookbook";
    String userName = "baduser";
    String password = "badpass";

    try {
      conn = DriverManager.getConnection(url, userName, password);
      System.out.println("Connected");
      tryQuery(conn);    // issue a query
    } catch (Exception e) {
      System.err.println("Cannot connect to server");
      System.err.println(e);
      if (e instanceof SQLException)  // JDBC-specific exception?
      {
        // e must be cast from Exception to SQLException to
        // access the SQLException-specific methods
        printException((SQLException) e);
      }
    } finally {
      if (conn != null) {
        try {
          conn.close ();
          System.out.println("Disconnected");
        } catch (SQLException e) {
          printException (e);
        }
      }
    }
  }

  public static void tryQuery(Connection conn) {
    try {
      // issue a simple query
      Statement s = conn.createStatement();
      s.execute("USE cookbook");
      s.close();

      // print any accumulated warnings
      SQLWarning w = conn.getWarnings();
      while (w != null) {
        System.err.println("SQLWarning: " + w.getMessage());
        System.err.println("SQLState: " + w.getSQLState());
        System.err.println("Vendor code: " + w.getErrorCode());
        w = w.getNextWarning();
      }
    } catch (SQLException e) {
      printException(e);
    }
  }

  public static void printException(SQLException e) {
    // print general message, plus any database-specific message
    System.err.println("SQLException: " + e.getMessage ());
    System.err.println("SQLState: " + e.getSQLState ());
    System.err.println("Vendor code: " + e.getErrorCode ());
  }
}
```

# 4.3 编写库文件

## 问题

你会注意到，在多个程序中重复编写代码来执行常见操作。

## 解决方案

编写例程来执行这些操作，将它们放入库文件中，并安排让你的程序访问该库。这使你只需编写一次代码。你可能需要设置一个环境变量，以便你的脚本可以找到该库。

## 讨论

本节描述了如何将常见操作的代码放入库文件中。封装（或模块化）实际上不是一个“食谱”，而是一种编程技术。它的主要好处是你不需要在每个编写的程序中重复编写代码。相反，只需调用库中的一个例程即可。例如，通过将连接到 `cookbook` 数据库的代码放入库例程中，你无需在每个程序中写出与建立连接相关的所有参数。只需从程序中调用该例程，就可以连接。

当然，建立连接并不是你唯一可以封装的操作。本书的后续章节将开发其他实用函数，以放置在库文件中。所有这些文件，包括本节中显示的文件，都位于 `recipes` 发行版的 *lib* 目录下。在编写自己的程序时，要注意那些经常执行的操作，并考虑将其包含到库文件中。使用本节中的技术编写你自己的库文件。

除了使编写程序更容易之外，库文件还有其他好处，例如促进可移植性。如果你直接将连接参数写入每个连接到 MySQL 服务器的程序中，如果将它们移动到使用不同参数的另一台机器上，则必须更改所有这些程序。相反，如果你编写程序以通过调用库例程连接到数据库，只需要修改受影响的库例程，而不必修改所有使用它的程序。

代码封装也可以提高安全性。如果你将私有库文件设置为仅自己可读，那么只有你运行的脚本可以执行文件中的程序。或者假设你有一些位于 Web 服务器文档树中的脚本。一个正确配置的服务器会执行这些脚本并将它们的输出发送给远程客户端。但如果服务器某种方式配置错误，结果可能是将你的脚本作为纯文本发送给客户端，从而显示你的 MySQL 用户名和密码。如果你将与 MySQL 服务器建立连接的代码放在位于文档树之外的库文件中，这些参数就不会暴露给客户端。

###### 警告

请注意，如果你将一个库文件安装为可被你的 Web 服务器读取，那么如果其他开发人员也使用同一台服务器，你的安全性就不高了。任何这些开发人员都可以编写一个 Web 脚本来读取和显示你的库文件，因为默认情况下，脚本以 Web 服务器的权限运行，因此可以访问该库。

接下来的示例演示了如何为每个 API 编写一个库文件，其中包含一个用于连接到 MySQL 服务器上的 `cookbook` 数据库的例程。调用程序可以使用第 4.2 节 中讨论的错误检查技术来确定连接尝试是否失败。每种语言的连接例程在成功时返回一个数据库句柄或连接对象，如果无法建立连接则抛出异常。

库本身没有任何用处，因此下面的讨论通过一个短的 [<q>测试驱动程序</q>](https://en.wikipedia.org/wiki/Test_harness) 程序来说明每个库的用途。要将任何这些驱动程序作为创建新程序的基础，请复制该文件并在连接和断开调用之间添加你自己的代码。

编写库文件不仅涉及文件中应该放入什么内容的问题，还包括诸如在何处安装文件以使其对你的程序可访问，以及（在类 Unix 的多用户系统上）如何设置其访问权限，以防止其内容暴露给不应查看它的人。

### 选择库文件安装位置

如果你将一个库文件安装在语言处理器默认搜索的目录中，那么使用该语言编写的程序无需特别操作即可访问该库。但是，如果你将一个库文件安装在语言处理器不默认搜索的目录中，你必须告诉你的脚本如何找到它。有两种常见的方法来做到这一点：

+   大多数语言提供了一种语句，可以在脚本内部使用，以将目录添加到语言处理器的搜索路径中。这需要你修改需要使用该库的每个脚本。

+   您可以设置一个环境或配置变量，以更改语言处理器的搜索路径。使用这种方法，每个执行需要库的脚本的用户必须设置适当的变量。或者，如果语言处理器有一个配置文件，您可能可以在文件中设置一个影响所有用户全局脚本的参数。

我们将使用第二种方法。对于我们的 API 语言，表 4-3 展示了相关变量。在每种情况下，变量值是一个目录或目录列表：

表 4-3\. 默认库路径

| 语言 | 变量名 | 变量类型 |
| --- | --- | --- |
| Perl | `PERL5LIB` | 环境变量 |
| Ruby | `RUBYLIB` | 环境变量 |
| PHP | `include_path` | 配置变量 |
| Python | `PYTHONPATH` | 环境变量 |
| Go | `GOPATH` | 环境变量 |
| Java | `CLASSPATH` | 环境变量 |

关于设置环境变量的一般信息，请阅读配方分发中的`cmdline.pdf`（见前言）。您可以使用这些说明将环境变量设置为下面讨论中的值。

假设您想要将库文件安装在语言处理器默认不搜索的目录中。作为示例，让我们在 Unix 上使用*/usr/local/lib/mcb*，在 Windows 上使用*C:\lib\mcb*。（要将文件放置在其他位置，请相应调整变量设置中的路径名。例如，您可能希望使用不同的目录，或者您可能希望将每种语言的库放在单独的目录中。）

在 Unix 下，如果您将 Perl 库文件放在*/usr/local/lib/mcb*目录中，请适当设置`PERL5LIB`环境变量。对于 Bourne shell 家族（*sh*、*bash*、*ksh*）的 shell，在适当的启动文件中设置变量如下：

```
export PERL5LIB=/usr/local/lib/mcb
```

###### 注意

对于原始的 Bourne shell，*sh*，您可能需要将此分成两个命令：

```
PERL5LIB=/usr/local/lib/mcb
export PERL5LIB
```

对于 C shell 家族（*csh*、*tcsh*）的 shell，在您的*.login*文件中设置`PERL5LIB`如下：

```
setenv PERL5LIB /usr/local/lib/mcb
```

在 Windows 下，如果您将 Perl 库文件放在*C:\lib\mcb*中，请设置`PERL5LIB`如下：

```
PERL5LIB=C:\lib\mcb
```

在每种情况下，变量值告诉 Perl 在指定目录中查找库文件，除了默认搜索的任何其他目录。如果将`PERL5LIB`设置为多个目录名称，则在 Unix 上的分隔符为冒号（`:`），在 Windows 上为分号（`;`）。

使用相同的语法指定其他环境变量（`RUBYLIB`、`PYTHONPATH`和`CLASSPATH`）。

###### 注意

如刚刚讨论的，设置这些环境变量应足以运行从命令行运行的脚本。对于打算由 Web 服务器执行的脚本，您可能还必须配置服务器，以便它可以找到库文件。

对于 PHP，搜索路径由 *php.ini* PHP 初始化文件中`include_path`变量的值定义。在 Unix 上，文件的路径名可能是 */usr/lib/php.ini* 或 */usr/local/lib/php.ini*。在 Windows 下，该文件可能位于 Windows 目录或主 PHP 安装目录下。要确定位置，请运行以下命令：

```
$ `php --ini`
```

使用如下行在 *php.ini* 中定义`include_path`的值：

```
include_path = "*`value`*"
```

使用与环境变量命名目录相同的语法指定 *`value`*。也就是说，它是一个目录名称列表，在 Unix 上用冒号分隔，在 Windows 上用分号分隔。在 Unix 上，如果您希望 PHP 在当前目录和 */usr/local/lib/mcb* 中查找包含文件，请像这样设置`include_path`：

```
include_path = ".:/usr/local/lib/mcb"
```

在 Windows 上，要在当前目录和 *C:\lib\mcb* 中搜索，请像这样设置`include_path`：

```
include_path = ".;C:\lib\mcb"
```

如果 PHP 作为 Apache 模块运行，请重新启动 Apache 以使 *php.ini* 更改生效。

### 设置库文件的访问权限

如果您使用类 Unix 的多用户系统，必须对库文件的所有权和访问模式做出决策：

+   如果库文件是私有的，并且仅包含您自己使用的代码，请将该文件放在您自己的帐户下，并且仅对您可访问。假设名为 *mylib* 的库文件已由您拥有，您可以像这样使其私有：

    ```
    $ `chmod 600 mylib`
    ```

+   如果库文件仅由您的 Web 服务器使用，请将其安装在服务器库目录中，并使其由服务器用户 ID 拥有和仅限该用户访问。您可能需要以`root`身份执行此操作。例如，如果 Web 服务器以`wwwusr`身份运行，则以下命令将文件设为仅该用户私有：

    ```
    # `chown wwwusr mylib`
    # `chmod 600 mylib`
    ```

+   如果库文件是公共的，您可以将其放在编程语言在查找库时自动搜索的位置。 （大多数语言处理器在某些默认目录中搜索库，尽管可以通过设置环境变量来影响此集合，如前述所述。）您可能需要以`root`身份在其中一个目录中安装文件。然后您可以将文件设为全局可读：

    ```
    # `chmod 444 mylib`
    ```

现在让我们为每个 API 构建一个库。本节的每个部分演示了如何编写库文件本身，并讨论了如何从程序内部使用该库。

### Perl

在 Perl 中，库文件称为模块，通常具有 *.pm* 扩展名（Perl 模块）。约定的是模块文件的基本名称与文件中`package`行上的标识符相同。以下文件 *Cookbook.pm* 实现了一个名为`Cookbook`的模块：

```
package Cookbook;
# Cookbook.pm: library file with utility method for connecting to MySQL
# using the Perl DBI module

use strict;
use warnings;
use DBI;

my $db_name = "cookbook";
my $host_name = "localhost";
my $user_name = "cbuser";
my $password = "cbpass";
my $port_num = undef;
my $socket_file = undef;

# Establish a connection to the cookbook database, returning a database
# handle.  Raise an exception if the connection cannot be established.

sub connect
{
my $dsn = "DBI:mysql:host=$host_name";
my $conn_attrs = {PrintError => 0, RaiseError => 1, AutoCommit => 1};

  $dsn .= ";database=$db_name" if defined ($db_name);
  $dsn .= ";mysql_socket=$socket_file" if defined ($socket_file);
  $dsn .= ";port=$port_num" if defined ($port_num);

  return DBI->connect ($dsn, $user_name, $password, $conn_attrs);
}

1;  # return true
```

该模块将与建立与 MySQL 服务器的连接的代码封装到一个`connect()`方法中，而`package`标识符则为该模块建立了一个`Cookbook`命名空间。要调用`connect()`方法，请使用模块名称：

```
$dbh = Cookbook::connect ();
```

模块文件的最后一行是一个无关紧要的评估为真的语句。如果模块没有返回真值，Perl 会认为有问题并退出。

Perl 通过搜索其`@INC`数组中命名的目录列表来定位库文件。要检查系统上此变量的默认值，请在命令行上调用 Perl 如下：

```
$ `perl -V`
```

命令输出的最后部分显示了`@INC`中列出的目录。如果在这些目录中的一个中安装了库文件，则您的脚本会自动找到它。如果在其他地方安装了模块，请通过设置`PERL5LIB`环境变量来告诉您的脚本在哪里找到它，如本文档介绍的内容。

安装完*Cookbook.pm*模块后，请从测试工具脚本*harness.pl*尝试：

```
#!/usr/bin/perl
# harness.pl: test harness for Cookbook.pm library

use strict;
use warnings;
use Cookbook;

my $dbh;
eval
{
  $dbh = Cookbook::connect ();
  print "Connected\n";
};
die "$@" if $@;
$dbh->disconnect ();
print "Disconnected\n";
```

*harness.pl*没有`use DBI`语句。这是不必要的，因为`Cookbook`模块本身导入了 DBI；任何使用`Cookbook`的脚本也都可以访问 DBI。

如果没有使用`eval`显式捕获连接错误，则可以更简单地编写脚本主体：

```
my $dbh = Cookbook::connect ();
print "Connected\n";
$dbh->disconnect ();
print "Disconnected\n";
```

在这种情况下，Perl 捕获任何连接异常，并在打印由`connect()`方法生成的错误消息后终止脚本。

### Ruby

下面的 Ruby 库文件*Cookbook.rb*定义了一个实现`connect`类方法的`Cookbook`类：

```
# Cookbook.rb: library file with utility method for connecting to MySQL
# using the Ruby Mysql2 module

require "mysql2"

# Establish a connection to the cookbook database, returning a database
# handle.  Raise an exception if the connection cannot be established.

class Cookbook
  @@host_name = "localhost"
  @@db_name = "cookbook"
  @@user_name = "cbuser"
  @@password = "cbpass"

  # Class method for connecting to server to access the
  # cookbook database; returns a database handle object.

  def Cookbook.connect
    return Mysql2::Client.new(:host => @@host_name,
                              :database => @@db_name,
                              :username => @@user_name, 
                              :password => @@password)
  end
end
```

`connect`方法在库中被定义为`Cookbook.connect`，因为 Ruby 类方法被定义为*`class_name.method_name`*。

Ruby 通过搜索其名为`$LOAD_PATH`（也称为`$:`）的变量（数组）中命名的目录列表来定位库文件。要检查系统上此变量的默认值，请使用交互式 Ruby 执行此语句：

```
$ `irb`
>> `puts $LOAD_PATH`
```

如果在这些目录中的一个中安装了库文件，则您的脚本会自动找到它。如果您在其他地方安装了文件，请通过设置`RUBYLIB`环境变量来告诉您的脚本在哪里找到它，如本文档介绍的内容。

安装完*Cookbook.rb*库文件后，请从测试工具脚本*harness.rb*尝试：

```
#!/usr/bin/ruby -w
# harness.rb: test harness for Cookbook.rb library

require "Cookbook"

begin
  client = Cookbook.connect
  print "Connected\n"
rescue Mysql2::Error => e
  puts "Cannot connect to server"
  puts "Error code: #{e.errno}"
  puts "Error message: #{e.message}"
  exit(1)
ensure
  client.close()
  print "Disconnected\n"
end
```

*harness.rb*没有 Mysql2 模块的`require`语句。这是不必要的，因为`Cookbook`模块本身导入了 Mysql2；任何导入`Cookbook`的脚本也都可以访问 Mysql2。

如果希望脚本在出现错误时退出而不检查异常本身，请将脚本主体编写如下所示：

```
client = Cookbook.connect
print "Connected\n"
client.close
print "Disconnected\n"
```

### PHP

PHP 库文件就像常规的 PHP 脚本一样编写。实现`Cookbook`类及其`connect()`方法的*Cookbook.php*文件如下所示：

```
<?php
# Cookbook.php: library file with utility method for connecting to MySQL
# using the PDO module

class Cookbook
{
  public static $host_name = "localhost";
  public static $db_name = "cookbook";
  public static $user_name = "cbuser";
  public static $password = "cbpass";

  # Establish a connection to the cookbook database, returning a database
  # handle.  Raise an exception if the connection cannot be established.
  # In addition, cause exceptions to be raised for errors.

  public static function connect ()
  {
    $dsn = "mysql:host=" . self::$host_name . ";dbname=" . self::$db_name;
    $dbh = new PDO ($dsn, self::$user_name, self::$password);
    $dbh->setAttribute (PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    return ($dbh);
  }

} # end Cookbook
?>
```

在类内部声明的`connect()`例程使用`static`关键字来将其声明为类方法，而不是实例方法。这使其可以直接调用，而无需通过实例化对象来调用它。

`new` `PDO()`构造函数如果连接尝试失败会引发异常。成功尝试后，`connect()`设置错误处理模式，使得其他 PDO 调用在失败时也会引发异常。这样，不需要为每个调用测试错误返回值。

虽然本书中大多数 PHP 示例没有显示`<?php`和`?>`标签，但我在这里显示它们作为*Cookbook.php*的一部分，以强调库文件必须将所有 PHP 代码置于这些标签内。PHP 解释器在开始解析库文件时不会对其内容做出任何假设，因为您可能包含一个只包含 HTML 的文件。因此，您必须使用`<?php`和`?>`显式指定库文件的哪些部分应被视为 PHP 代码而不是 HTML，就像在主脚本中所做的那样。

PHP 通过在 PHP 初始化文件中描述的`include_path`变量命名的目录中搜索库文件。

###### 注意

PHP 脚本通常放置在您的 Web 服务器文档树中，客户端可以直接请求它们。对于 PHP 库文件，我们建议将它们放在文档树之外的某个地方，特别是像*Cookbook.php*这样的文件，如果包含用户名和密码的话。

将*Cookbook.php*安装在`include_path`目录之一后，可以从测试用例脚本*harness.php*中尝试它：

```
<?php
# harness.php: test harness for Cookbook.php library

require_once "Cookbook.php";

try
{
  $dbh = Cookbook::connect ();
  print ("Connected\n");
}
catch (PDOException $e)
{
  print ("Cannot connect to server\n");
  print ("Error code: " . $e->getCode () . "\n");
  print ("Error message: " . $e->getMessage () . "\n");
  exit (1);
}
$dbh = NULL;
print ("Disconnected\n");
?>
```

`require_once`语句访问所需使用`Cookbook`类的*Cookbook.php*文件。`require_once`是几个 PHP 文件包含语句之一：

+   `require`和`include`指示 PHP 读取指定的文件。它们类似，但是如果找不到文件，`require`会终止脚本；`include`只会产生警告。

+   `require_once`和`include_once`与`require`和`include`类似，但如果文件已经被读取，则不会再次处理其内容。这对于避免在库文件包含其他库文件时可能出现的多次声明问题非常有用。

### Python

Python 库被编写为模块，并通过`import`语句从脚本中引用。要创建连接到 MySQL 的方法，请编写一个模块文件*cookbook.py*（Python 模块名称应为小写）：

```
# cookbook.py: library file with utility method for connecting to MySQL
# using the Connector/Python module

import mysql.connector

conn_params = {
  "database": "cookbook",
  "host": "localhost",
  "user": "cbuser",
  "password": "cbpass",
}

# Establish a connection to the cookbook database, returning a connection
# object.  Raise an exception if the connection cannot be established.

def connect():
  return mysql.connector.connect(**conn_params)
```

文件名的基本名称确定模块名称，因此该模块被称为`cookbook`。可以通过模块名称访问模块方法；因此，导入`cookbook`模块并调用其`connect()`方法如下：

```
import cookbook

conn = cookbook.connect();
```

Python 解释器通过`sys.path`变量中命名的目录搜索模块。要检查系统上`sys.path`的默认值，请在 Python 交互模式下运行，并输入几个命令：

```
$ `python`
>>> `import sys`
>>> `sys.path`
```

如果您将*cookbook.py*安装在由`sys.path`命名的目录之一中，则您的脚本将无需特殊处理即可找到它。如果您将*cookbook.py*安装在其他地方，则必须设置`PYTHONPATH`环境变量，如本教程的介绍部分所述。

安装了 *cookbook.py* 库文件后，可以通过测试用的 *harness.py* 脚本进行尝试：

```
#!/usr/bin/python
# harness.py: test harness for cookbook.py library

import mysql.connector
import cookbook

try:
  conn = cookbook.connect()
  print("Connected")
except mysql.connector.Error as e:
  print("Cannot connect to server")
  print("Error code: %s" % e.errno)
  print("Error message: %s" % e.msg)
else:
  conn.close()
  print("Disconnected")
```

*cookbook.py* 文件导入了 `mysql.connector` 模块，但导入 `cookbook` 的脚本并不能因此获得对 `mysql.connector` 的访问权限。如果脚本需要 Connector/Python 特定的信息（如 `mysql.connector.Error`），则脚本本身必须导入 `mysql.connector`。

如果希望脚本在发生错误时退出而无需自行检查异常，请像这样编写脚本体：

```
conn = cookbook.connect()
print("Connected")
conn.close()
print("Disconnected")
```

### Go

Go 程序被组织成包，这些包是位于同一目录中的源文件集合。包又被组织成模块，这些模块是一起发布的 Go 包的集合。模块属于 Go 代码库。一个典型的 Go 代码库只包含一个模块，但在同一个代码库中可以有多个模块。

Go 解释器在名为 `$GOPATH/src/{domain}/{project}` 的目录中搜索包。但是，使用模块时，Go 不再使用 `GOPATH`。无论您的模块安装在何处，都不需要更改此变量。我们将在示例中使用模块。

要创建连接到 MySQL 的方法，编写一个名为 *cookbook.go* 的包文件：

```
package cookbook

import (
  "database/sql"
  _"github.com/go-sql-driver/mysql"
)

func Connect() (*sql.DB, error) {
  db, err := sql.Open("mysql","cbuser:cbpass@tcp(127.0.0.1:3306)/cookbook")

  if err != nil {
    panic(err.Error())
  }

  err = db.Ping()

  return db, err
}
```

文件名的基本名称不确定包的名称：Go 通过导入路径中的所有文件搜索，直到找到具有所需包声明的文件。包方法通过包名访问。

要测试包，可以指定包文件所在目录的相对路径：

```
import "../../lib"
```

这是一种非常简单的方法来快速测试您的库，但是像 *go install* 这样的命令不能用于以这种方式导入的包。因此，每次访问时都会从头开始重建您的程序。

更好的处理包的方式是将它们作为模块的一部分发布。要执行此操作，请在存储 `cookbook.go` 的目录中运行以下命令：

```
go mod init cookbook
```

这将创建一个包含您的模块名称和 Go 版本的 `go.mod` 文件。您可以根据需要命名模块。

您可以将模块发布到互联网并像处理任何其他模块一样从本地程序中访问它。但在开发过程中，仅在本地使用该模块将非常有用。在这种情况下，您需要在将使用它的程序目录中进行一些调整。

首先，创建一个将调用包的程序，`harness.go`：

```
package main

import (
  "fmt"
  "github.com/svetasmirnova/mysqlcookbook/recipes/lib"
)

func main() {
  db, err := cookbook.Connect()

  if err != nil {
    fmt.Println("Cannot connect to server")
    fmt.Printf("Error message: %s\n", err.Error())
  } else {
    fmt.Println("Connected")
  }
  defer db.Close()
}
```

然后，在安装包的目录中，初始化模块：

```
go mod init harness
```

模块初始化完成并创建了 `go.mod` 后，通过以下方式编辑它：

```
go mod edit -replace ↩
github.com/svetasmirnova/mysqlcookbook/recipes/lib=↩
/home/sveta/src/mysqlcookbook/recipes/lib
```

用您环境中有效的 URL 和本地路径替换它们。

此命令将告诉 Go 使用本地目录替换远程模块路径。

完成后，您可以测试您的连接：

```
$ `go run harness.go`
Connected
```

### Java

Java 库文件在大多数方面与 Java 程序相似：

+   源文件中的 `class` 行指示了一个类名。

+   文件应与类名相同（使用 *.java* 扩展名）。

+   编译*.java*文件以生成*.class*文件。

Java 库文件在某些方面也与 Java 程序不同：

+   与常规程序文件不同，Java 库文件没有`main()`函数。

+   一个库文件应该以指定类在 Java 命名空间中位置的`package`标识符开始。

Java 包标识符的常见约定是使用代码作者的域作为前缀；这有助于使标识符保持唯一，并避免与其他作者编写的类发生冲突。域名在域命名空间中从右向左逐渐具体化，而 Java 类命名空间则从左向右从一般到特定。因此，要在 Java 类命名空间中使用域作为包名的前缀，需要对其进行反转。例如，Paul 的域名是*kitebird.com*，因此如果他编写一个库文件并将其放在其域命名空间中的`mcb`下，则库文件以如下`package`语句开头：

```
package com.kitebird.mcb;
```

为了确保 Java 命名空间中的唯一性，本书开发的 Java 包位于`com.kitebird.mcb`命名空间内。

下面的库文件*Cookbook.java*定义了一个`Cookbook`类，实现了一个`connect()`方法用于连接到`cookbook`数据库。如果`connect()`成功，则返回一个`Connection`对象，否则抛出异常。为了帮助调用者处理失败，`Cookbook`类还定义了`getErrorMessage()`和`printErrorMessage()`实用方法，分别返回错误消息字符串并将其打印到`System.err`：

```
// Cookbook.java: library file with utility methods for connecting to MySQL
// using MySQL Connector/J and for handling exceptions

package com.kitebird.mcb;

import java.sql.*;

public class Cookbook {
  // Establish a connection to the cookbook database, returning
  // a connection object.  Throw an exception if the connection
  // cannot be established.

  public static Connection connect() throws Exception {
    String url = "jdbc:mysql://localhost/cookbook";
    String user = "cbuser";
    String password = "cbpass";

    return (DriverManager.getConnection(url, user, password));
  }

  // Return an error message as a string

  public static String getErrorMessage(Exception e) {
    StringBuffer s = new StringBuffer ();
    if (e instanceof SQLException) { // JDBC-specific exception?
      // print general message, plus any database-specific message
      s.append("Error message: " + e.getMessage () + "\n");
      s.append("Error code: " + ((SQLException) e).getErrorCode() + "\n");
    } else {
      s.append (e + "\n");
    }
    return (s.toString());
  }

  // Get the error message and print it to System.err

  public static void printErrorMessage(Exception e) {
    System.err.println(Cookbook.getErrorMessage(e));
  }
}
```

类中的例程使用`static`关键字声明，这使它们成为类方法而不是实例方法。之所以这样做是因为该类是直接使用而不是从中创建对象并通过对象调用方法。

要使用*Cookbook.java*文件，首先编译它以生成*Cookbook.class*，然后将类文件安装在与包标识符对应的目录中。

这意味着*Cookbook.class*应安装在名为*com/kitebird/mcb*（Unix）或*com\kitebird\mcb*（Windows）的目录中，该目录位于您的`CLASSPATH`设置中指定的某个目录下。例如，如果在 Unix 下`CLASSPATH`包含*/usr/local/lib/mcb*，则可以将*Cookbook.class*安装在*/usr/local/lib/mcb/com/kitebird/mcb*目录下。（有关`CLASSPATH`变量的更多信息，请参阅 Recipe 4.1 中的 Java 讨论。）

要从 Java 程序中使用`Cookbook`类，导入它并调用`Cookbook.connect()`方法。以下测试框架程序*Harness.java*展示了如何做到这一点：

```
// Harness.java: test harness for Cookbook library class

import java.sql.*;
import com.kitebird.mcb.Cookbook;

public class Harness {
  public static void main(String[] args) {
    Connection conn = null;
    try {
      conn = Cookbook.connect ();
      System.out.println("Connected");
    } catch (Exception e) {
      Cookbook.printErrorMessage (e);
      System.exit (1);
    } finally  {
      if (conn != null) {
        try {
          conn.close();
          System.out.println("Disconnected");
        } catch (Exception e) {
          String err = Cookbook.getErrorMessage(e);
          System.out.println(err);
        }
      }
    }
  }
}
```

*Harness.java*还展示了当发生 MySQL 相关异常时如何使用`Cookbook`类的错误消息实用方法：

+   `printErrorMessage()`接受异常对象并用它打印错误消息到`System.err`。

+   `getErrorMessage()` 返回错误消息作为字符串。你可以自行显示该消息，将其写入日志文件，或者其他操作。

# 4.4 执行语句和检索结果

## 问题

你需要一个程序向 MySQL 服务器发送 SQL 语句并检索其结果。

## 解决方案

一些语句仅返回状态代码；其他则返回结果集（一组行）。某些 API 提供了执行每种类型语句的不同方法。如果是这样，请使用适当的方法执行要执行的语句。

## 讨论

你可以执行两类通用的 SQL 语句。一些从数据库检索信息；其他则更改信息或数据库本身。这两类语句有不同的处理方式。此外，一些 API 提供多个例程来执行语句，进一步增加了复杂性。在我们演示如何从每个 API 内部执行语句的示例之前，我们将描述示例使用的数据库表，并进一步讨论这两类语句，并概述处理每类语句的一般策略。

在第一章中，我们创建了一个名为`limbs`的表来尝试一些示例语句。在本章中，我们将使用名为`profile`的不同表。它基于一个<q>好友列表</q>的概念，即我们在线时想要保持联系的人。该表的定义如下：

```
CREATE TABLE profile
(
  id    INT UNSIGNED NOT NULL AUTO_INCREMENT,
  name  VARCHAR(20) NOT NULL,
  birth DATE,
  color ENUM('blue','red','green','brown','black','white'),
  foods SET('lutefisk','burrito','curry','eggroll','fadge','pizza'),
  cats  INT,
  PRIMARY KEY (id)
);
```

`profile`表指示我们关于每个好友重要的事情：姓名、年龄、喜欢的颜色、喜欢的食物和猫的数量。此外，该表使用多种不同的数据类型作为其列，并且这些类型对于说明如何解决特定数据类型相关的问题非常有用。

该表还包括一个`id`列，其中包含唯一的值，以便我们可以区分一行与另一行，即使两个好友有相同的名字。`id`和`name`被声明为`NOT NULL`，因为它们各自需要有一个值。其他列隐式允许为`NULL`（这也是它们的默认值），因为我们可能不知道为任何给定个体分配的值。也就是说，`NULL`表示<q>未知</q>。

请注意，尽管我们想追踪年龄，但表中没有`age`列。相反，有一个`DATE`类型的`birth`列。年龄会变化，因此如果存储年龄值，我们将不得不不断更新它们。存储出生日期更好：它们不会改变，并且可以随时用来计算年龄（参见 Recipe 8.14）。`color`是一个`ENUM`列；颜色值可以是所列出的任何一个值。`foods`是一个`SET`，允许值是个别集合成员的任意组合。这样我们可以记录每个好友的多个喜欢的食物。

要创建表格，请使用`recipes`分发中`tables`目录下的*profile.sql*脚本。切换到该目录，然后运行以下命令：

```
$ `mysql cookbook < profile.sql`
```

脚本还将示例数据加载到表中。您可以对表进行实验，然后如果修改了其内容，可以再次运行脚本来恢复它。（请参阅本章末尾关于修改后重要性的`profile`表的恢复。）

由*profile.sql*脚本加载的`profile`表内容如下所示：

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

尽管`profile`表中大多数列允许`NULL`值，但样本数据集中的行实际上都不包含`NULL`值。我们将`NULL`值处理的复杂性推迟到 Recipe 4.5 和 Recipe 4.7。

### SQL 语句的分类

SQL 语句可以根据是否返回结果集（一组行）分为两大类：

+   不返回结果集的语句，例如`INSERT`、`DELETE`或`UPDATE`。一般而言，这类语句通常会对数据库进行某种形式的更改。也有例外情况，比如`USE` *`db_name`*，它会改变你会话的默认数据库而不对数据库本身进行任何更改。本节中用到的数据修改语句是`UPDATE`：

    ```
    UPDATE profile SET cats = cats+1 WHERE name = 'Sybil';
    ```

    我们将介绍如何执行此语句并确定它影响的行数。

+   返回结果集的语句，例如`SELECT`、`SHOW`、`EXPLAIN`或`DESCRIBE`。我通常把这些语句泛称为`SELECT`语句，但你应理解这一类别包括返回行的任何语句。本节中用到的行检索语句是`SELECT`：

    ```
    SELECT id, name, cats FROM profile;
    ```

    我们将介绍如何执行此语句、获取结果集中的行，并确定结果集中的行数和列数。（要获取诸如列名或数据类型的信息，请访问结果集元数据。这在 Recipe 12.2 中。）

处理 SQL 语句的第一步是将其发送到 MySQL 服务器执行。某些 API（如 Perl 和 Java）识别这两类语句并为其提供单独的调用。其他 API（如 Python 或 Ruby）使用单一调用来执行所有语句。不过，所有 API 都共同点是没有特殊字符表示语句的结束。不需要终止符，因为语句字符串的结尾就是其终止符。这与在*mysql*程序中执行语句不同，在那里你使用分号(`;`)或`\g`来终止语句。（这也不同于本书通常在示例中包含分号以明确语句结束的方式。）

当你向服务器发送语句时，要准备好处理执行失败的错误。如果语句执行失败但你仍然基于其成功继续执行，你的程序将无法正常工作。大多数情况下，本节不显示错误检查代码，这是为了简洁。实际生产代码中应始终包括错误处理。示例脚本中的`recipes`分发从中提取示例包含基于 Recipe 4.2 展示的技术的错误处理。

如果语句成功执行且没有错误，接下来的步骤取决于语句类型。如果是不返回结果集的语句，则无需进一步操作，除非你想检查影响了多少行。如果语句返回结果集，则获取其行并关闭结果集。在不确定语句是否返回结果集的上下文中，Recipe 12.2 讨论了如何判断。

### Perl

Perl DBI 模块提供了两种基本的 SQL 语句执行方法，取决于是否期望得到结果集。对于像`INSERT`或`UPDATE`这样不返回结果集的语句，使用数据库句柄的`do()`方法。它执行语句并返回受其影响的行数，或者如果发生错误则返回`undef`。如果 Sybil 有了一只新猫，下面的语句会将她的`cats`计数增加一：

```
my $count = $dbh->do ("UPDATE profile SET cats = cats+1
 WHERE name = 'Sybil'");
if ($count)   # print row count if no error occurred
{
  $count += 0;
  print "Number of rows updated: $count\n";
}
```

如果语句成功执行但未影响任何行，`do()`将返回特殊值`"0E0"`（科学计数法下的零，表示为字符串）。在测试语句执行状态时，可以使用`"0E0"`因为它在布尔上下文中为真（与`undef`不同）。对于成功的语句，它还可用于计算受影响行数，因为在数值上下文中它被视为零。当然，如果直接打印该值，会显示为`"0E0"`，这可能对使用你的程序的人来说看起来很奇怪。前面的示例通过将零加到该值以强制其转换为数值形式来确保不会发生这种情况，以便显示为`0`。或者，使用`printf`和`%d`格式说明符以导致隐式数值转换：

```
if ($count)   # print row count if no error occurred
{
  printf "Number of rows updated: %d\n", $count;
}
```

如果启用了`RaiseError`，你的脚本会在 DBI 相关错误时自动终止，因此无需检查`$count`来查找`do()`是否失败，从而简化代码：

```
my $count = $dbh->do ("UPDATE profile SET cats = cats+1
 WHERE name = 'Sybil'");
printf "Number of rows updated: %d\n", $count;
```

要处理像`SELECT`这样返回结果集的语句，需采用不同的方法：

1.  通过使用数据库句柄调用`prepare()`指定要执行的语句。`prepare()`返回一个语句句柄，用于后续所有操作。如果发生错误且启用了`RaiseError`，脚本将终止；否则，`prepare()`返回`undef`。

1.  调用`execute()`来执行语句并生成结果集。

1.  循环获取语句返回的行。DBI 提供了几种方法，我们很快会介绍它们。

1.  如果不获取整个结果集，请通过调用 `finish()` 来释放与其关联的资源。

下面的示例说明了这些步骤，使用 `fetchrow_array()` 作为获取行的方法，并假定启用了 `RaiseError` 以便错误终止脚本：

```
my $sth = $dbh->prepare ("SELECT id, name, cats FROM profile");
$sth->execute ();
my $count = 0;
while (my @val = $sth->fetchrow_array ())
{
  print "id: $val[0], name: $val[1], cats: $val[2]\n";
  ++$count;
}
$sth->finish ();
print "Number of rows returned: $count\n";
```

行数组大小表示结果集中的列数。

刚才显示的获取行循环后调用了 `finish()`，它关闭了结果集并告诉服务器释放与其关联的任何资源。如果获取了集合中的每一行，DBI 在到达末尾时会注意到并为你释放资源。因此，示例可以省略 `finish()` 调用而不会产生任何不良影响。

如示例所示，当获取行时同时计数可以确定结果集包含的行数。不要使用 DBI 的 `rows()` 方法来达到此目的。DBI 文档不建议这种做法，因为 `rows()` 对于 `SELECT` 语句并不一定可靠——由于不同数据库引擎和驱动程序的行为差异。

DBI 有几种一次获取一行的方法。在前面的例子中使用的 `fetchrow_array()` 方法返回一个包含下一行的数组，或者当没有更多行时返回一个空列表。数组元素按照 `SELECT` 语句中指定的顺序存在。可以使用 `$val[0]`，`$val[1]` 等来访问它们。

`fetchrow_array()` 方法对于显式命名要选择的列的语句最为有用。（使用 `SELECT *`，不能保证数组中列的位置。）

`fetchrow_arrayref()` 类似于 `fetchrow_array()`，但它返回数组的引用，或者当没有更多行时返回`undef`。与 `fetchrow_array()` 一样，数组元素按照语句中指定的顺序存在。可以使用 `$ref->[0]`，`$ref->[1]` 等来访问它们：

```
while (my $ref = $sth->fetchrow_arrayref ())
{
  print "id: $ref->[0], name: $ref->[1], cats: $ref->[2]\n";
}
```

`fetchrow_hashref()` 返回一个指向哈希结构的引用，当没有更多行时返回`undef`：

```
while (my $ref = $sth->fetchrow_hashref ())
{
  print "id: $ref->{id}, name: $ref->{name}, cats: $ref->{cats}\n";
}
```

要访问哈希的元素，使用语句选择的列名（`$ref->{id}`，`$ref->{name}` 等）。`fetchrow_hashref()` 对于 `SELECT *` 语句特别有用，因为可以访问行的元素，而无需知道列返回的顺序。你只需要知道它们的名称即可。另一方面，建立哈希比建立数组更昂贵，因此 `fetchrow_hashref()` 比 `fetchrow_array()` 或 `fetchrow_arrayref()` 更慢。如果列名重复，可能会<q>丢失</q>行元素，因为列名必须是唯一的。在表连接时，同名列并不少见。有关此问题的解决方案，请参阅 Recipe 16.11。

除了刚刚描述的语句执行方法外，DBI 还提供了几种高级检索方法，这些方法执行语句并以单个操作返回结果集。所有这些方法都是数据库句柄方法，在返回结果集之前在内部创建和处理语句句柄。这些方法在返回结果集的形式上有所不同。有些返回整个结果集，其他返回集合的单行或单列，如 Table 4-4 所总结：

Table 4-4\. 检索结果的 Perl 方法

| 方法 | 返回值 |
| --- | --- |
| `selectrow_array()` | 结果集的第一行作为数组 |
| `selectrow_arrayref()` | 结果集的第一行作为数组引用 |
| `selectrow_hashref()` | 结果集的第一行作为哈希引用 |
| `selectcol_arrayref()` | 结果集的第一列作为数组引用 |
| `selectall_arrayref()` | 整个结果集作为数组引用的数组 |
| `selectall_hashref()` | 整个结果集作为哈希引用的哈希 |

大多数这些方法返回一个引用。`selectrow_array()` 是个例外，它选择结果集的第一行并返回一个数组或标量，具体返回取决于调用方式。在数组上下文中，`selectrow_array()` 返回整行作为数组（如果未选择任何行则返回空列表）。这对于预期仅获取单行的语句很有用。返回值可用于确定结果集大小。列数为数组的元素个数，行数为 1 或 0：

```
my @val = $dbh->selectrow_array ("SELECT name, birth, foods FROM profile
 WHERE id = 3");
my $ncols = @val;
my $nrows = $ncols ? 1 : 0;
```

`selectrow_arrayref()` 和 `selectrow_hashref()` 选择结果集的第一行并返回其引用，如果未选择任何行则返回`undef`。要访问列值，处理引用的方式与处理 `fetchrow_arrayref()` 或 `fetchrow_hashref()` 返回值相同。引用还提供了行数和列数：

```
my $ref = $dbh->selectrow_arrayref ($stmt);
my $ncols = defined ($ref) ? @{$ref} : 0;
my $nrows = $ncols ? 1 : 0;

my $ref = $dbh->selectrow_hashref ($stmt);
my $ncols = defined ($ref) ? keys (%{$ref}) : 0;
my $nrows = $ncols ? 1 : 0;
```

`selectcol_arrayref()` 返回指向结果集第一列的单列数组的引用。假设返回值不为`undef`，则可以使用 `$ref->[$i]` 访问数组的第 *`i`* 行值。数组的元素个数即为行数，列数为 1 或 0：

```
my $ref = $dbh->selectcol_arrayref ($stmt);
my $nrows = defined ($ref) ? @{$ref} : 0;
my $ncols = $nrows ? 1 : 0;
```

`selectall_arrayref()` 返回一个包含结果集每行元素的数组引用。每个元素都是一个数组的引用。要访问结果集的第 *`i`* 行，使用 `$ref->[$i]` 获得行的引用，然后像处理 `fetchrow_arrayref()` 返回值一样访问行中的各个列值。结果集的行数和列数可按如下方式获取：

```
my $ref = $dbh->selectall_arrayref ($stmt);
my $nrows = defined ($ref) ? @{$ref} : 0;
my $ncols = $nrows ? @{$ref->[0]} : 0;
```

`selectall_hashref()`返回对哈希的引用，其中每个元素都是结果行的哈希引用。要调用它，请指定一个参数，该参数指示要用于哈希键的列。例如，如果从`profile`表检索行，则主键是`id`列。

```
my $ref = $dbh->selectall_hashref ("SELECT * FROM profile", "id");
```

使用哈希的键访问行。对于键列值为`12`的行，行的哈希引用是`$ref->{12}`。该行值以列名为键，您可以使用它们来访问单个列元素（例如，`$ref->{12}->{name}`）。结果集的行数和列数如下所示：

```
my @keys = defined ($ref) ? keys (%{$ref}) : ();
my $nrows = scalar (@keys);
my $ncols = $nrows ? keys (%{$ref->{$keys[0]}}) : 0;
```

当需要多次处理结果集时，`selectall_`*`XXX`*`()`方法非常有用，因为 Perl DBI 没有提供回放结果集的方法。通过将整个结果集分配给变量，可以多次迭代其元素。

如果禁用了`RaiseError`，在使用高级方法时要小心。在这种情况下，方法的返回值可能无法让您区分错误和空结果集。例如，如果以标量上下文调用`selectrow_array()`以检索单个值，则`undef`返回值是模棱两可的，因为它可能表示三种情况之一：错误、空结果集或由单个`NULL`值组成的结果集。要检测错误，请检查`$DBI::errstr`、`$DBI::err`或`$DBI::state`的值。

### Ruby

Ruby 的 Mysql2 API 使用相同的调用来处理不返回结果集和返回结果集的 SQL 语句。在 Ruby 中处理语句时，使用`query`方法。如果语句因错误而失败，`query`会抛出异常。否则，`affected_rows`方法返回修改数据的最后一条语句的行数。

```
client.query("UPDATE profile SET cats = cats+1 WHERE name = 'Sybil'")
puts "Number of rows updated: #{client.affected_rows}"
```

对于返回结果集的`SELECT`等语句，`query`方法将结果集作为`Mysql2::Result`类的实例返回。对于这种语句，`affected_rows`方法将返回结果集中的行数。您还可以通过`Mysql2::Result`对象的`count`方法获取结果集中的行数。

```
result = client.query("SELECT id, name, cats FROM profile")
puts "Number of rows returned: #{client.affected_rows}"
puts "Number of rows returned: #{result.count}"
result.each do |row|
  printf "id: %s, name: %s, cats: %s\n", row["id"], row["name"], row["cats"]
end
```

`result.fields`包含结果集中列的名称。

### PHP

PDO 有两个连接对象方法用于执行 SQL 语句：`exec()`用于不返回结果集的语句，`query()`用于返回结果集的语句。如果启用了 PDO 异常，这两个方法在语句执行失败时都会抛出异常。（另一种方法是结合`prepare()`和`execute()`方法；请参见 Recipe 4.5。）

要执行诸如`INSERT`或`UPDATE`等不返回行的语句，请使用`exec()`。它返回一个计数，指示更改了多少行：

```
$count = $dbh->exec ("UPDATE profile SET cats = cats+1 WHERE name = 'Sybil'");
printf ("Number of rows updated: %d\n", $count);
```

对于返回结果集的`SELECT`等语句，`query()`方法返回一个语句句柄。通常，您使用此对象在循环中调用行提取方法，并计算行数（如果需要知道有多少行）。

```
$sth = $dbh->query ("SELECT id, name, cats FROM profile");
$count = 0;
while ($row = $sth->fetch (PDO::FETCH_NUM))
{
  printf ("id: %s, name: %s, cats: %s\n", $row[0], $row[1], $row[2]);
  $count++;
}
printf ("Number of rows returned: %d\n", $count);
```

要确定结果集中列的数量，请调用语句句柄的`columnCount()`方法。

示例演示了语句句柄的`fetch()`方法，该方法返回结果集的下一行或在没有更多行时返回`FALSE`。`fetch()`接受一个可选参数，指示它应返回什么类型的值。如示所示，使用`PDO::FETCH_NUM`作为参数，`fetch()`返回一个数组，可以使用数值下标访问其元素，从 0 开始。数组大小表示结果集列数。

使用`PDO::FETCH_ASSOC`作为参数，`fetch()`返回一个包含通过列名访问的值的关联数组（`$row["id"]`，`$row["name"]`，`$row["cats"]`）。

使用`PDO::FETCH_OBJ`作为参数，`fetch()`返回一个对象，可以使用列名访问其成员（`$row->id`，`$row->name`，`$row->cats`）。

如果调用`fetch()`而不带参数，则使用默认的提取模式。除非已更改模式，否则默认为`PDO::FETCH_BOTH`，类似于`PDO::FETCH_NUM`和`PDO::FETCH_ASSOC`的组合。要为连接中执行的所有语句设置默认提取模式，请使用`setAttribute`数据库句柄方法：

```
$dbh->setAttribute (PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);
```

要在给定语句中设置模式，请在执行语句之后并获取结果之前调用其`setFetchMode()`方法：

```
$sth->setFetchMode (PDO::FETCH_OBJ);
```

还可以将语句句柄用作迭代器。句柄使用当前默认的提取模式：

```
$sth->setFetchMode (PDO::FETCH_NUM);
foreach ($sth as $row)
  printf ("id: %s, name: %s, cats: %s\n", $row[0], $row[1], $row[2]);
```

`fetchAll()`方法将整个结果集作为行数组提取并返回。它允许一个可选的提取模式参数：

```
$rows = $sth->fetchAll (PDO::FETCH_NUM);
foreach ($rows as $row)
  printf ("id: %s, name: %s, cats: %s\n", $row[0], $row[1], $row[2]);
```

在这种情况下，行数是`$rows`中元素的数量。

### Python

Python DB API 对不返回结果集的 SQL 语句和返回结果集的 SQL 语句使用相同的调用方式。要在 Python 中处理语句，使用数据库连接对象获取游标对象。然后使用游标的`execute()`方法将语句发送到服务器。如果语句出错，`execute()`会引发异常。否则，如果没有结果集，语句执行完成，并且游标的`rowcount`属性指示更改了多少行：

```
cursor = conn.cursor()
cursor.execute("UPDATE profile SET cats = cats+1 WHERE name = 'Sybil'")
print("Number of rows updated: %d" % cursor.rowcount)
conn.commit()
cursor.close()
```

###### 注意

Python DB API 规范指示，数据库连接应以禁用自动提交模式开始，因此当 Connector/Python 连接到 MySQL 服务器时，它会禁用自动提交。如果使用事务表，在关闭连接之前未提交更改，这些修改将被回滚，因此前面的示例调用了`commit()`方法。有关自动提交模式的更多信息，请参阅第二十章，特别是 Recipe 20.7)。

如果语句返回结果集，则提取其行，然后关闭游标。`fetchone()`方法将下一行作为序列返回，或者在没有更多行时返回`None`：

```
cursor = conn.cursor()
cursor.execute("SELECT id, name, cats FROM profile")
while True:
  row = cursor.fetchone()
  if row is None:
    break
  print("id: %s, name: %s, cats: %s" % (row[0], row[1], row[2]))
print("Number of rows returned: %d" % cursor.rowcount)
cursor.close()
```

如前面的示例所示，`rowcount` 属性对 `SELECT` 语句也很有用；它指示结果集中的行数。

`len(row)` 告诉您结果集中的列数。

或者，将游标本身用作迭代器，依次返回每一行：

```
cursor = conn.cursor()
cursor.execute("SELECT id, name, cats FROM profile")
for (id, name, cats) in cursor:
  print("id: %s, name: %s, cats: %s" % (id, name, cats))
print("Number of rows returned: %d" % cursor.rowcount)
cursor.close()
```

`fetchall()` 方法将整个结果集作为元组列表返回。通过列表迭代以访问行：

```
cursor = conn.cursor()
cursor.execute("SELECT id, name, cats FROM profile")
rows = cursor.fetchall()
for row in rows:
  print("id: %s, name: %s, cats: %s" % (row[0], row[1], row[2]))
print("Number of rows returned: %d" % cursor.rowcount)
cursor.close()
```

DB API 提供了一种重置结果集的方法，因此当必须多次迭代结果集的行或直接访问单个值时，`fetchall()` 很方便。例如，如果 `rows` 包含结果集，则可以通过 `rows[1][2]` 访问第二行的第三列的值（索引从 0 开始，而不是 1）。

### Go

Go 的 `sql` 接口有两个连接对象函数来执行 SQL 语句：`Exec()` 用于不返回结果集的语句，`Query()` 用于返回结果集的语句。如果语句失败，两者都会返回 `error`。

要执行不返回任何行的语句（如 `INSERT`、`UPDATE` 或 `DELETE`），请使用函数 `Exec()`。其返回值可以是 `Result` 或 `error` 类型。接口 `Result` 具有函数 `RowsAffected()`，指示更改了多少行。

```
sql := "UPDATE profile SET cats = cats+1 WHERE name = 'Sybil'"
res, err := db.Exec(sql)

if err != nil {
	panic(err.Error())
}

affectedRows, err := res.RowsAffected()

if err != nil {
	log.Fatal(err)
}

fmt.Printf("The statement affected %d rows\n", affectedRows)
```

对于返回结果集的语句，通常使用函数 `Query()`。该函数返回一个 `Rows` 类型的游标对象，其中保存了查询的结果。使用函数 `Next()` 来遍历结果，并使用函数 `Scan()` 将返回的值存储在变量中。如果 `Next()` 返回 `false`，表示没有结果。

```
res, err := db.Query("SELECT id, name, cats FROM profile")

defer res.Close()

if err != nil {
	log.Fatal(err)
}

for res.Next() {

	var profile Profile
	err := res.Scan(&profile.id, &profile.name, &profile.cats)

	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("%+v\n", profile)
}
```

如果调用 `Next()` 并返回 `false`，则 `Rows` 会自动关闭。否则，需要使用函数 `Close()` 来关闭它们。

对于预期最多返回一行的查询，存在特殊函数 `QueryRow()`，返回一个 `Row` 对象，可以立即进行扫描。在调用 `Scan()` 之前，`QueryRow()` 不会返回错误。如果查询没有返回行，则 `Scan()` 返回 `ErrNoRows`。

```
row := db.QueryRow("SELECT id, name, cats FROM profile where id=3")

var profile Profile
err = row.Scan(&profile.id, &profile.name, &profile.cats)

if err == sql.ErrNoRows {
	fmt.Println("No row matched!")
} else if err != nil {
	log.Fatal(err)
} else {
	fmt.Printf("%v\n", profile)
}
```

### Java

JDBC 接口为 SQL 语句处理的各个阶段提供了特定的对象类型。在 JDBC 中，语句通过一种类型的 Java 对象执行。结果（如果有）以另一种类型的对象返回。

要执行语句，请先通过调用连接对象的 `createStatement()` 方法获取 `Statement` 对象：

```
Statement s = conn.createStatement ();
```

然后，使用 `Statement` 对象将语句发送到服务器。JDBC 提供了几种执行此操作的方法。选择适合语句类型的方法：`executeUpdate()` 用于不返回结果集的语句，`executeQuery()` 用于返回结果集的语句，`execute()` 用于其他情况。如果语句失败，每种方法都会引发异常。

`executeUpdate()` 方法向服务器发送不生成结果集的语句，并返回受影响行数。完成语句对象后，请关闭它：

```
Statement s = conn.createStatement ();
int count = s.executeUpdate(
               "UPDATE profile SET cats = cats+1 WHERE name = 'Sybil'");
s.close();   // close statement
System.out.println("Number of rows updated: " + count);
```

对于返回结果集的语句，请使用`executeQuery()`。然后获取一个结果集对象，并使用它检索行值。完成后，关闭结果集和语句对象：

```
Statement s = conn.createStatement ();
s.executeQuery("SELECT id, name, cats FROM profile");
ResultSet rs = s.getResultSet();
int count = 0;
while (rs.next ()) { // loop through rows of result set\
  int id = rs.getInt(1);   // extract columns 1, 2, and 3
  String name = rs.getString(2);
  int cats = rs.getInt(3);
  System.out.println("id: " + id
                     + ", name: " + name
                     + ", cats: " + cats);
  ++count;
}
rs.close ();  // close result set
s.close ();   // close statement
System.out.println ("Number of rows returned: " + count);
```

`Statement`对象的`getResultSet()`方法返回的`ResultSet`对象具有自己的方法，例如`next()`以获取行和各种`get`*`XXX`*`()`方法以访问当前行的列。最初，结果集位于集合的第一行之前。调用`next()`以连续获取每行，直到返回 false。要确定结果集中的行数，请自行计算，如前面的示例所示。

###### 提示

对于返回单个结果集的查询，不需要调用*getResultSet*。上面的代码可以写成：

```
ResultSet rs = s.executeQuery("SELECT id, name, cats FROM profile");
```

. 当您的查询可能返回多个结果集时，例如，如果调用存储过程，需要单独调用。

要访问列值，使用诸如`getInt()`、`getString()`、`getFloat()`和`getDate()`等方法。要将列值作为通用对象获取，使用`getObject()`。`get`*`XXX`*`()`调用的参数可以指示列位置（从 1 开始，而不是 0）或列名。前面的示例展示了如何按位置检索`id`、`name`和`cats`列。要改为按名称访问列，编写行提取循环如下：

```
while (rs.next ()) { // loop through rows of result set
  int id = rs.getInt("id");
  String name = rs.getString("name");
  int cats = rs.getInt("cats");
  System.out.println("id: " + id
                     + ", name: " + name
                     + ", cats: " + cats);
  ++count;
}
```

要检索给定列值，请使用任何对于数据类型有意义的`get`*`XXX`*`()`调用。例如，`getString()`将任何列值作为字符串检索：

```
String id = rs.getString("id");
String name = rs.getString("name");
String cats = rs.getString("cats");
System.out.println("id: " + id
                   + ", name: " + name
                   + ", cats: " + cats);
```

或使用`getObject()`将值检索为通用对象，并根据需要转换值。以下示例使用`toString()`将对象值转换为可打印形式：

```
Object id = rs.getObject("id");
Object name = rs.getObject("name");
Object cats = rs.getObject("cats");
System.out.println("id: " + id.toString()
                   + ", name: " + name.toString()
                   + ", cats: " + cats.toString());
```

要确定结果集中的列数，请访问其元数据：

```
ResultSet rs = s.getResultSet();
ResultSetMetaData md = rs.getMetaData(); // get result set metadata
int ncols = md.getColumnCount();         // get column count from metadata
```

第三个 JDBC 语句执行方法`execute()`适用于任何类型的语句。当您从外部来源接收语句字符串并不知道它是否生成结果集或返回多个结果集时，它特别有用。`execute()`的返回值指示语句类型，以便您可以适当处理它：如果`execute()`返回 true，则存在结果集，否则没有。通常，您会像这样使用它，其中`stmtStr`表示任意 SQL 语句：

```
Statement s = conn.createStatement();
if (s.execute(stmtStr)) {
  // there is a result set
  ResultSet rs = s.getResultSe();

  // ... process result set here ...

  rs.close();  // close result set
} else {
  // there is no result set, just print the row count
  System.out.println("Number of rows affected: " + s.getUpdateCount ());
}
s.close();   // close statement
```

# 4.5 处理语句中的特殊字符和 NULL 值

## 问题

您需要构造引用包含特殊字符（如引号或反斜杠）或特殊值（如`NULL`）的数据值的 SQL 语句。或者您正在使用从外部来源获取的数据构造语句，并希望防止 SQL 注入攻击。

## 解决方案

使用 API 的占位符机制或引用函数使数据安全可插入。

## 讨论

到本章为止，我们的语句使用了不需要特殊处理的<q>安全</q>数据值。例如，我们可以轻松地从程序内部编写数据值直接在语句字符串中构造以下 SQL 语句：

```
SELECT * FROM profile WHERE age > 40 AND color = 'green';

INSERT INTO profile (name,color) VALUES('Gary','blue');
```

然而，有些数据值不那么容易处理，如果不小心就会引起问题。语句可能使用包含特殊字符（如引号、反斜杠、二进制数据或`NULL`值）的数值。以下讨论描述了这些数值引起的困难以及正确的处理技术。

假设你想执行这个`INSERT`语句：

```
INSERT INTO profile (name,birth,color,foods,cats)
VALUES('Alison','1973-01-12','blue','eggroll',4);
```

如果你将`name`列的值更改为像`De'Mont`这样包含单引号的内容，这个语句就会变得语法无效：

```
INSERT INTO profile (name,birth,color,foods,cats)
VALUES('De'Mont','1973-01-12','blue','eggroll',4);
```

问题在于单引号位于单引号字符串内部。为了通过转义引号使语句合法，可以在其前面加上单引号或反斜杠：

```
INSERT INTO profile (name,birth,color,foods,cats)
VALUES('De''Mont','1973-01-12','blue','eggroll',4);
```

```
INSERT INTO profile (name,birth,color,foods,cats)
VALUES('De\'Mont','1973-01-12','blue','eggroll',4);
```

或者，用双引号引用`name`值本身，而不是单引号（假设未启用`ANSI_QUOTES` SQL 模式）：

```
INSERT INTO profile (name,birth,color,foods,cats)
VALUES("De'Mont",'1973-01-12','blue','eggroll',4);
```

如果你在程序中直接编写语句，可以手动转义或引用`name`值，因为你知道该值是什么。但是如果名称存储在变量中，你并不一定知道变量的值。更糟糕的是，你必须准备处理的不仅仅是单引号；双引号和反斜杠也会引起问题。如果数据库存储二进制数据（如图像或音频剪辑），某个值可能包含任何内容——不仅仅是引号或反斜杠，还可能包含空值（零值字节）等其他字符。正确处理特殊字符的需求在 Web 环境中尤为迫切，因为语句是使用表单输入构建的（例如，如果你搜索与远程用户输入的搜索词匹配的行）。你必须能够以通用方式处理任何类型的输入，因为你无法预测用户将提供什么样的信息。恶意用户常常输入包含问题字符的垃圾值，试图攻击服务器安全性，甚至执行致命命令，比如`DROP TABLE`。这是一种利用不安全脚本的标准技术，称为[SQL 注入](https://en.wikipedia.org/wiki/SQL_injection)。

SQL 中的`NULL`值不是特殊字符，但它也需要特殊处理。在 SQL 中，`NULL`表示<q>无值</q>。这可能根据上下文有几种含义，例如<q>未知</q>，<q>丢失</q>，<q>超出范围</q>等等。到目前为止，我们的语句还没有使用`NULL`值，以避免处理它们引入的复杂性，但现在是时候解决这些问题了。例如，如果你不知道 De’Mont 最喜欢的颜色，你可以将`color`列设为`NULL`，但不能这样写语句：

```
INSERT INTO profile (name,birth,color,foods,cats)
VALUES('De''Mont','1973-01-12','NULL','eggroll',4);
```

相反，`NULL`值必须没有引号包围：

```
INSERT INTO profile (name,birth,color,foods,cats)
VALUES('De''Mont','1973-01-12',NULL,'eggroll',4);
```

如果你在程序中直接写语句，你只需写单词<q>NULL</q>而不加引号。但如果`color`值来自变量，正确的操作就不那么明显了。您必须知道变量的值是否表示`NULL`，以确定在构造语句时是否将其置于引号中。

您有两种方法来处理诸如引号和反斜杠之类的特殊字符以及`NULL`之类的特殊值：

+   在执行语句时，使用占位符在语句字符串中象征性地引用数据值，然后将数据值绑定到占位符。这是首选方法，因为 API 本身会根据需要为您提供引号，引用或转义数据值中的特殊字符，并可能解释特殊值以映射到`NULL`而不用加引号。

+   使用引号函数（如果您的 API 提供）将数据值转换为适合在语句字符串中使用的安全形式。

本节展示了如何使用这些技术来处理每个 API 中的特殊字符和`NULL`值。这里展示的一个示例演示了如何插入包含`De'Mont`作为`name`值和`NULL`作为`color`值的`profile`表行。然而，这里展示的原则具有一般实用性，并处理任何特殊字符，包括二进制数据中的特殊字符。此外，这些原则不仅限于`INSERT`语句，它们同样适用于其他类型的语句，比如`SELECT`。这里展示的另一个示例演示了如何使用占位符执行`SELECT`语句。

处理特殊字符和`NULL`值是其他上下文中涉及到的内容：

+   此处描述的占位符和引用技术*仅*适用于数据值，而不适用于诸如数据库或表名之类的标识符。有关标识符引用的讨论，请参阅 Recipe 4.6。

+   比较`NULL`值需要与非`NULL`值不同的运算符。Recipe 5.6 讨论如何从程序内部构造执行`NULL`比较的 SQL 语句。

+   本节涵盖了将特殊字符 *插入* 到你的数据库中的问题。相关问题是反向操作，即将值中的特殊字符 *从* 你的数据库中转换，以在各种上下文中进行显示。例如，如果你生成包含来自数据库的值的 HTML 页面，则必须执行输出编码，将这些值中的 `<` 和 `>` 字符转换为 HTML 实体 `&lt;` 和 `&gt;` 以确保它们正确显示。

### 使用占位符

占位符允许你避免在 SQL 语句中直接写入数据值。使用这种方法，你可以使用占位符——特殊的标记，指示数值应该放在哪里。两种常见的参数标记是`?`和`%s`。根据标记，重写`INSERT`语句以使用如下的占位符：

```
INSERT INTO profile (name,birth,color,foods,cats)
VALUES(?,?,?,?,?);

INSERT INTO profile (name,birth,color,foods,cats)
VALUES(%s,%s,%s,%s,%s);
```

然后将语句字符串传递给数据库服务器，并单独提供数据值。API 将这些值绑定到占位符上以替换它们，生成包含数据值的语句。

占位符的一个好处是参数绑定操作会自动处理转义字符，如引号和反斜杠。这对于将二进制数据（例如图像）插入到数据库中或使用具有未知内容的数据值（例如通过网页表单由远程用户提交的输入）特别有用。此外，通常有一些特殊值，你可以绑定到占位符，以指示你希望在结果语句中得到 SQL 的 `NULL` 值。

占位符的第二个好处是，你可以预先<q>准备</q>一个语句，然后每次执行时绑定不同的值以重复使用它。预准备的语句鼓励语句重用。语句变得更通用，因为它们包含占位符而不是特定的数据值。如果你反复执行某个操作，可能可以重用一个预准备的语句，每次执行时只需绑定不同的数据值。某些数据库系统（MySQL 不在其中）具有在执行预准备语句之前执行某些预解析或执行计划的能力。对于稍后多次执行的语句，这减少了开销，因为只需在执行前完成一次工作，而不是每次执行都要完成。例如，如果一个程序在运行时多次执行特定类型的`SELECT`语句，这样的数据库系统可以构建该语句的计划，然后每次重用，而不是反复重建计划。MySQL 不预先构建查询计划，因此使用预准备语句不会带来性能提升。但是，如果你将程序移植到一个支持重用查询计划的数据库，并且你已经编写程序以使用预准备语句，那么你可以自动享受到预准备语句的这个优势。你无需从非预准备语句转换，就可以享受到这个好处。

第三个（诚然是主观的）好处是使用基于占位符的语句的代码可能更易于阅读。在您阅读本节时，请比较这里使用的语句与未使用占位符的 Recipe 4.4 中的语句，看看您更喜欢哪一种。

### 使用引号函数

一些 API 提供了一个引号函数，它以数据值作为其参数并返回一个适合安全插入到 SQL 语句中的正确引号和转义值。这种方法不如使用占位符常见，但在构造不打算立即执行的语句时可能很有用。但是，在使用此类引号函数时，必须保持对数据库服务器的连接打开，因为 API 无法在不知道数据库驱动程序的情况下选择正确的引号规则（这些规则在不同的数据库系统中有所不同）。

###### 注意

正如我们稍后将指出的那样，某些 API 在将非 `NULL` 值绑定到参数标记时会将所有非 `NULL` 值（包括数字）都视为字符串引号。在需要 *必须* 数字的情况下，这可能会成为问题，如 Recipe 5.11 中进一步描述的那样。

### Perl

要使用带有 Perl DBI 的占位符，请在 SQL 语句字符串中的每个数据值位置放置一个`?`。然后通过将这些值传递给 `do()` 或 `execute()`，或者调用专门用于占位符替换的 DBI 方法来绑定这些值到语句中。使用 `undef` 来绑定一个 `NULL` 值到占位符。

使用 `do()`，通过在同一调用中传递语句字符串和数据值，为 De’Mont 添加 `profile` 行：

```
my $count = $dbh->do ("INSERT INTO profile (name,birth,color,foods,cats)
 VALUES(?,?,?,?,?)",
                      undef,
                      "De'Mont", "1973-01-12", undef, "eggroll", 4);
```

跟在语句字符串后面的参数是 `undef`，然后是每个占位符对应的一个数据值。`undef` 参数是一个历史遗留物，但必须存在。

或者，将语句字符串传递给 `prepare()` 以获得语句句柄，然后使用该句柄将数据值传递给 `execute()`：

```
my $sth = $dbh->prepare ("INSERT INTO profile (name,birth,color,foods,cats)
 VALUES(?,?,?,?,?)");
my $count = $sth->execute ("De'Mont", "1973-01-12", undef, "eggroll", 4);
```

在任何情况下，DBI 生成此语句：

```
INSERT INTO profile (name,birth,color,foods,cats)
VALUES('De\'Mont','1973-01-12',NULL,'eggroll','4');
```

Perl DBI 占位符机制在将数据值绑定到语句字符串时会为其添加引号，因此不要在字符串中的 `?` 字符周围加引号。

请注意，占位符机制会在数值周围添加引号。DBI 依赖于 MySQL 服务器根据需要将字符串转换为数字进行类型转换。如果将 `undef` 绑定到占位符，则 DBI 将 `NULL` 放入语句中，并正确地避免添加包围引号。

要重复执行同一语句，请先使用 `prepare()`，然后每次运行时使用适当的数据值调用 `execute()`。

您可以将这些方法用于其他类型的语句。例如，以下 `SELECT` 语句使用占位符来查找 `cats` 值大于 2 的行：

```
my $sth = $dbh->prepare ("SELECT * FROM profile WHERE cats > ?");
$sth->execute (2);
while (my $ref = $sth->fetchrow_hashref ())
{
  print "id: $ref->{id}, name: $ref->{name}, cats: $ref->{cats}\n";
}
```

高级检索方法（如`selectrow_array()`和`selectall_arrayref()`）也可以与占位符一起使用。与`do()`方法一样，参数是语句字符串、`undef`和要绑定到占位符的数据值。以下是一个例子：

```
my $ref = $dbh->selectall_arrayref (
  "SELECT name, birth, foods FROM profile WHERE id > ? AND color = ?",
  undef, 3, "green"
);
```

Perl DBI 的`quote()`数据库句柄方法是使用占位符的替代方法。以下是如何使用`quote()`创建插入`profile`表中新行的语句字符串。由于`quote()`会根据需要自动提供引号，因此请不要用引号括住`%s`格式说明符。非`undef`值会带有引号插入，而`undef`值会作为`NULL`值插入而不带引号：

```
my $stmt = sprintf ("INSERT INTO profile (name,birth,color,foods,cats)
 VALUES(%s,%s,%s,%s,%s)",
                    $dbh->quote ("De'Mont"),
                    $dbh->quote ("1973-01-12"),
                    $dbh->quote (undef),
                    $dbh->quote ("eggroll"),
                    $dbh->quote (4));
my $count = $dbh->do ($stmt);
```

由此代码生成的语句字符串与使用占位符时相同。

### Ruby

Ruby DBI 在 SQL 语句中使用`?`作为占位符字符，并使用`nil`作为将 SQL `NULL`值绑定到占位符的值。

要使用`?`，请将语句字符串传递给`prepare`以获取语句句柄，然后使用该句柄调用带有数据值的`execute`：

```
sth = client.prepare("INSERT INTO profile (name,birth,color,foods,cats)
 VALUES(?,?,?,?,?)")
sth.execute("De'Mont", "1973-01-12", nil, "eggroll", 4)
```

Mysql2 在结果语句中包含正确转义的引号和正确的未引用`NULL`值：

```
INSERT INTO profile (name,birth,color,foods,cats)
VALUES('De\'Mont','1973-01-12',NULL,'eggroll',4);
```

Ruby 的 Mysql2 占位符机制在绑定到语句字符串时根据需要在数据值周围添加引号，因此不要在字符串中的`?`字符周围放置引号。

### PHP

要使用 PDO 扩展的占位符，请将语句字符串传递给`prepare()`以获取语句对象。字符串可以包含`?`字符作为占位符标记。使用此对象调用`execute()`，向其传递要绑定到占位符的数据值数组。使用 PHP 的`NULL`值将 SQL `NULL`值绑定到占位符。用于为 De’Mont 添加`profile`表行的代码如下：

```
$sth = $dbh->prepare ("INSERT INTO profile (name,birth,color,foods,cats)
 VALUES(?,?,?,?,?)");
$sth->execute (array ("De'Mont","1973-01-12",NULL,"eggroll",4));
```

结果语句包含正确转义的引号和正确的未引用`NULL`值：

```
INSERT INTO profile (name,birth,color,foods,cats)
VALUES('De\'Mont','1973-01-12',NULL,'eggroll','4');
```

PDO 占位符机制在绑定到语句字符串时在数据值周围添加引号，因此不要在字符串中的`?`字符周围放置引号。（请注意，甚至数字值`4`也会被引用；PDO 依赖 MySQL 在语句执行时进行必要的类型转换。）

### Python

Connector/Python 模块通过在 SQL 语句字符串中使用`%s`格式说明符来实现占位符。（要在语句中插入字面上的`%`字符，请在语句字符串中使用`%%`。）要使用占位符，请使用两个参数调用`execute()`方法：包含格式说明符的语句字符串和包含要绑定到语句字符串的值的序列。使用`None`将`NULL`值绑定到占位符。用于为 De’Mont 添加`profile`表行的代码如下：

```
cursor = conn.cursor()
cursor.execute('''
 INSERT INTO profile (name,birth,color,foods,cats)
 VALUES(%s,%s,%s,%s,%s)
 ''', ("De'Mont", "1973-01-12", None, "eggroll", 4))
cursor.close()
conn.commit()
```

由上述`execute()`调用发送到服务器的语句如下所示：

```
INSERT INTO profile (name,birth,color,foods,cats)
VALUES('De\'Mont','1973-01-12',NULL,'eggroll',4);
```

Connector/Python 的占位符机制在绑定到语句字符串时根据需要在数据值周围添加引号，因此不要在字符串中的`%s`格式说明符周围放置引号。

如果只有一个单值 *`val`* 需要绑定到占位符，可以使用以下语法将其写为序列： `(`*`val`*,`)`：

```
cursor = conn.cursor()
cursor.execute("SELECT id, name, cats FROM profile WHERE cats = %s", (2,))
for (id, name, cats) in cursor:
  print("id: %s, name: %s, cats: %s" % (id, name, cats))
cursor.close()
```

或者，将值写为列表，使用以下语法 `[`*`val`*`]`。

### Go

Go 的 `sql` 包使用问号 (`?`) 作为占位符标记。你可以在单个 `Exec()` 或 `Query()` 调用中使用占位符，也可以预先准备语句并稍后执行。当需要多次执行语句时，后一种方法很有用。为了为 De’Mont 添加 `profile` 表行的代码看起来像这样：

```
stmt := `INSERT INTO profile (name,birth,color,foods,cats)
 VALUES(?,?,?,?,?)`
_, err = db.Exec(stmt, "De'Mont", "1973-01-12", nil, "eggroll", 4)
```

使用 `Prepare()` 调用的相同代码看起来像这样：

```
pstmt, err := db.Prepare(`INSERT INTO profile (name,birth,color,foods,cats)
 VALUES(?,?,?,?,?)`)
if err != nil {
  log.Fatal(err)
}
defer pstmt.Close()

_, err = pstmt.Exec("De'Mont", "1973-01-12", nil, "eggroll", 4)
```

### Java

如果使用预编译语句，JDBC 提供了对占位符的支持。回顾在 JDBC 中执行非预编译语句的过程是创建 `Statement` 对象，然后将语句字符串传递给 `executeUpdate()`、`executeQuery()` 或 `execute()` 函数。相反，如果使用预编译语句，通过将包含 `?` 占位符字符的语句字符串传递给连接对象的 `prepareStatement()` 方法来创建 `PreparedStatement` 对象。然后使用 `set`*`XXX`*`()` 方法将数据值绑定到语句。最后，通过调用 `executeUpdate()`、`executeQuery()` 或 `execute()` 执行语句，并使用空参数列表。

下面是一个示例，使用 `executeUpdate()` 执行 `INSERT` 语句，为 De’Mont 添加 `profile` 表行：

```
PreparedStatement s;
s = conn.prepareStatement(
            "INSERT INTO profile (name,birth,color,foods,cats)"
            + " VALUES(?,?,?,?,?)");
s.setString(1, "De'Mont");         // bind values to placeholders
s.setString(2, "1973-01-12");
s.setNull(3, java.sql.Types.CHAR);
s.setString(4, "eggroll");
s.setInt(5, 4);
s.close();   // close statement
```

`set`*`XXX`*`()` 方法将数据值绑定到语句上，需要两个参数：占位符位置（从 1 开始，不是 0）和要绑定到占位符的值。选择每个值绑定调用以匹配绑定到的列的数据类型：`setString()` 绑定字符串到 `name` 列，`setInt()` 绑定整数到 `cats` 列，依此类推。 （实际上，我使用 `setString()` 将日期值绑定为字符串以处理 `birth`。）

JDBC 与其他 API 的一个区别是，你不需要指定特殊值（例如 Perl 中的 `undef` 或 Ruby 中的 `nil`）来将 `NULL` 绑定到占位符上。相反，调用 `setNull()` 并传递第二个参数指示列的类型：`java.sql.Types.CHAR` 表示字符串，`java.sql.Types.INTEGER` 表示整数，依此类推。

`set`*`XXX`*`()` 调用会根据需要为数据值添加引号，因此不要在语句字符串中的 `?` 占位符字符周围加上引号。

要处理返回结果集的语句，流程类似，但要使用 `executeQuery()` 而不是 `executeUpdate()` 执行准备好的语句：

```
PreparedStatement s;
s = conn.prepareStatement("SELECT * FROM profile WHERE cats > ?");
s.setInt(1, 2);  // bind 2 to first placeholder
s.executeQuery();
// ... process result set here ...
s.close();     // close statement
```

# 4.6 处理标识符中的特殊字符

## 问题

您需要构造引用包含特殊字符的标识符的 SQL 语句。

## 解决方案

对每个标识符进行引用，以便安全地插入到语句字符串中。

## 讨论

Recipe 4.5 讨论如何通过使用占位符或引用方法处理数据值中的特殊字符。标识符中也可能存在特殊字符，例如数据库、表和列名。例如，表名 `some table` 包含默认情况下不允许的空格：

```
mysql> `CREATE TABLE some table (i INT);`
ERROR 1064 (42000): You have an error in your SQL syntax near 'table (i INT)'
```

标识符中的特殊字符与数据值中的处理方式不同。为了使标识符能够安全地插入到 SQL 语句中，请使用反引号将其括起来进行引用：

```
mysql> ``CREATE TABLE `some table` (i INT);``
Query OK, 0 rows affected (0.04 sec)
```

在 MySQL 中，反引号始终允许用于标识符引用。如果启用了 `ANSI_QUOTES` SQL 模式，则也允许双引号字符。因此，在启用 `ANSI_QUOTES` 的情况下，以下两个语句等效：

```
CREATE TABLE `some table` (i INT);
CREATE TABLE "some table" (i INT);
```

如果需要知道哪些标识符引用字符是允许的，请执行 `SELECT @@sql_mode` 语句来检索您会话的 SQL 模式，并检查其值是否包含 `ANSI_QUOTES`。

如果引号字符出现在标识符本身中，请在引用标识符时加倍。例如，将 ``abc`def`` 引用为 ``` `abc``def` ```。

请注意，尽管 MySQL 中的字符串数据值通常可以使用单引号或双引号字符 (`'abc'`, `"abc"`) 进行引用，但在启用 `ANSI_QUOTES` 的情况下并非如此。在这种情况下，MySQL 将 `'abc'` 解释为字符串，`"abc"` 解释为标识符，因此您必须仅使用单引号来引用字符串。

在程序中，如果 API 提供了标识符引用例程，您可以使用该例程，如果没有，则可以自己编写一个。Perl DBI 有一个 `quote_identifier()` 方法，返回一个正确引用的标识符。对于没有此类方法的 API，您可以通过将其用反引号括起来并加倍任何出现在标识符中的反引号来引用标识符。以下是一个执行此操作的 PHP 例程：

```
function quote_identifier ($ident)
{
  return ('`' . str_replace('`', '``', $ident) . '`');
}
```

可移植性注意事项：如果编写自己的标识符引用例程，请记住其他 DBMS 可能需要不同的引用约定。

在将标识符用作数据值的上下文中，请相应处理它们。如果从 `INFORMATION_SCHEMA` 元数据数据库中选择信息，通常会通过在 `WHERE` 子句中指定数据库对象名称来指示返回哪些行。例如，此语句检索 `cookbook` 数据库中 `profile` 表的列名：

```
SELECT COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'cookbook' AND TABLE_NAME = 'profile';
```

此处使用的数据库和表名作为数据值使用，而不是标识符。如果在程序中构造此语句，使用占位符参数化它们，而不是标识符引用。例如，在 Ruby 中，执行如下操作：

```
sth = client.prepare("SELECT COLUMN_NAME
 FROM INFORMATION_SCHEMA.COLUMNS
 WHERE TABLE_SCHEMA = ? AND TABLE_NAME = ?")
names = sth.execute(db_name, tbl_name)
```

# 4.7 在结果集中识别 NULL 值

## 问题

查询结果包含 `NULL` 值，但你不确定如何识别它们。

## 解决方案

你的 API 可能通过惯例使用特殊值表示 `NULL`。你只需知道它是什么，以及如何测试它。

## 讨论

Recipe 4.5 描述了在向数据库服务器发送语句时如何引用 `NULL` 值。在本节中，我们将处理如何识别和处理从数据库服务器返回的 `NULL` 值的问题。通常，这涉及知道 API 将 `NULL` 值映射到什么特殊值，或者调用什么方法。Table 4-5 显示了这些值：

表 4-5\. 检测到的 NULL 值

| 语言 | 检测 `NULL` 值的值或方法 |
| --- | --- |
| Perl DBI | `undef` 值 |
| Ruby Mysql2 gem | `nil` 值 |
| PHP PDO | `NULL` 值 |
| Python DB API | `None` 值 |
| Go `sql` 接口 | 为可空数据类型实现的 Go Null 类型 |
| Java JDBC | `wasNull()` 方法 |

以下各节展示了检测 `NULL` 值的非常简单的应用程序示例。示例检索结果集并打印其中的所有值，将 `NULL` 值映射到可打印的字符串 `"NULL"`。

为了确保 `profile` 表中有包含一些 `NULL` 值的行，请使用 *mysql* 执行以下 `INSERT` 语句，然后执行 `SELECT` 语句以验证生成的行具有预期的值：

```
mysql> `INSERT INTO profile (name) VALUES('Amabel');`
mysql> `SELECT * FROM profile WHERE name = 'Amabel';`
+----+--------+-------+-------+-------+------+
| id | name   | birth | color | foods | cats |
+----+--------+-------+-------+-------+------+
|  9 | Amabel | NULL  | NULL  | NULL  | NULL |
+----+--------+-------+-------+-------+------+
```

`id` 列可能包含不同的数字，但其他列应该按照所示的方式出现，其中值为 `NULL`。

### Perl

Perl DBI 使用 `undef` 表示 `NULL` 值。要检测这些值，请使用 `defined()` 函数；如果启用了 Perl 的 `-w` 选项或通过在脚本中包含 `use warnings` 行，则这样做尤为重要。否则，访问 `undef` 值会导致 Perl 发出 `Use of uninitialized value` 警告。

为了避免这些警告，请在使用之前使用 `defined()` 测试可能是 `undef` 的列值。以下代码从 `profile` 表中选择几列，并为每行中任何未定义的值打印 `"NULL"`。这样可以在输出中明确地表示 `NULL` 值，而不激活任何警告消息：

```
my $sth = $dbh->prepare ("SELECT name, birth, foods FROM profile");
$sth->execute ();
while (my $ref = $sth->fetchrow_hashref ())
{
  printf "name: %s, birth: %s, foods: %s\n",
         defined ($ref->{name}) ? $ref->{name} : "NULL",
         defined ($ref->{birth}) ? $ref->{birth} : "NULL",
         defined ($ref->{foods}) ? $ref->{foods} : "NULL";
}
```

不幸的是，测试多个列值很冗长，随着列数的增加而变得更糟。为了避免这种情况，在打印之前使用循环或 `map` 测试和设置未定义的值。以下示例使用 `map`：

```
my $sth = $dbh->prepare ("SELECT name, birth, foods FROM profile");
$sth->execute ();
while (my $ref = $sth->fetchrow_hashref ())
{
  map { $ref->{$_} = "NULL" unless defined ($ref->{$_}); } keys (%{$ref});
  printf "name: %s, birth: %s, foods: %s\n",
         $ref->{name}, $ref->{birth}, $ref->{foods};
}
```

使用这种技术，执行测试的代码量是恒定的，而不是与要测试的列数成比例增加的。此外，没有引用特定的列名，因此它可以更轻松地用于其他程序或作为实用程序例程的基础。

如果你将行数据取出到一个数组而不是哈希表中，可以使用以下方式使用 `map` 来转换 `undef` 值：

```
my $sth = $dbh->prepare ("SELECT name, birth, foods FROM profile");
$sth->execute ();
while (my @val = $sth->fetchrow_array ())
{
  @val = map { defined ($_) ? $_ : "NULL" } @val;
  printf "name: %s, birth: %s, foods: %s\n",
         $val[0], $val[1], $val[2];
}
```

### Ruby

Ruby Mysql2 模块使用 `nil` 表示 `NULL` 值，可以通过将 `nil?` 方法应用于值来识别。以下示例使用 `nil?` 方法和三元运算符确定是否将结果集值原样打印，或者对于 `NULL` 值打印字符串 `"NULL"`：

```
result = client.query("SELECT name, birth, foods FROM profile")
result.each do |row|
  printf "name %s, birth: %s, foods: %s\n", 
         row["name"].nil? ? "NULL" : row["name"], 
         row["birth"].nil? ? "NULL" : row["birth"], 
         row["foods"].nil? ? "NULL" : row["foods"]
end
```

### PHP

PHP 将 SQL 结果集中的 `NULL` 值表示为 PHP 的 `NULL` 值。要确定来自结果集的值是否表示 `NULL` 值，请使用 `===` <q>三重等</q> 运算符进行比较：

```
if ($val === NULL)
{
  # $val is a NULL value
}
```

在 PHP 中，三重等号运算符意味着 <q>完全相等</q>。通常的 `==` <q>相等</q> 比较运算符在这里不适用：使用 `==` 时，PHP 认为 `NULL` 值、空字符串和 `0` 都相等。

下面的代码使用 `===` 运算符来识别结果集中的 `NULL` 值并将它们打印为字符串 `"NULL"`：

```
$sth = $dbh->query ("SELECT name, birth, foods FROM profile");
while ($row = $sth->fetch (PDO::FETCH_NUM))
{
  foreach (array_keys ($row) as $key)
  {
    if ($row[$key] === NULL)
      $row[$key] = "NULL";
  }
  print ("name: $row[0], birth: $row[1], foods: $row[2]\n");
}
```

用于 `NULL` 值测试的 `===` 的替代方法是 `is_null()`。

### Python

Python DB API 程序使用 `None` 表示结果集中的 `NULL`。以下示例显示如何检测 `NULL` 值：

```
cursor = conn.cursor()
cursor.execute("SELECT name, birth, foods FROM profile")

for row in cursor:
  row = list(row)  # convert nonmutable tuple to mutable list

  for i, value in enumerate(row):
    if value is None:  # is the column value NULL?
      row[i] = "NULL"

  print("name: %s, birth: %s, foods: %s" % (row[0], row[1], row[2]))

cursor.close()
```

内部循环通过查找 `None` 来检查 `NULL` 列值，并将它们转换为字符串 `"NULL"`。示例在循环之前将 `row` 转换为可变对象（列表），因为 `fetchall()` 返回行作为序列值，这些值是不可变的（只读）。

### 前往

Go 的 `sql` 接口提供了特殊的数据类型来处理结果集中可能包含 NULL 值的值。它们针对标准 Go 类型进行了定义。表 4-6 包含标准数据类型及其可为空的等效类型的列表。

表 4-6\. 在 Go 中处理 NULL 值

| Go 标准类型 | 可包含 NULL 值的类型 |
| --- | --- |
| `bool` | `NullBool` |
| `float64` | `NullFloat64` |
| `int32` | `NullInt32` |
| `int64` | `NullInt64` |
| `string` | `NullString` |
| `time.Time` | `NullTime` |

要定义一个变量，它既可以接受 `NULL` 值也可以接受非 `NULL` 值作为传递给 `Scan()` 函数的参数，请使用相应的可为空类型。

所有可为空类型都包含两个函数：`Valid()` 如果值不为 `NULL` 则返回 `true`，如果值为 `NULL` 则返回 `false`。第二个函数是以大写字母开头的类型名称。例如，对于 `string` 值是 `String()`，对于 `time.Time` 值是 `Time()`。当值不为 `NULL` 时，此方法返回实际值。

这里是在 Go 中处理 `NULL` 值的示例。

```
// null-in-result.go : Selecting NULL values in Go
package main

import (
	"database/sql"
	"fmt"
	"log"

	_ "github.com/go-sql-driver/mysql"
)

type Profile struct {
    name     string
    birth    sql.NullString
    foods    sql.NullString
}

func main() {

	db, err := sql.Open("mysql", "cbuser:cbpass@tcp(127.0.0.1:3306)/cookbook")

	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	sql := "SELECT name, birth, foods FROM profile"
	res, err := db.Query(sql)

	if err != nil {
		log.Fatal(err)
	}
	defer res.Close()

	for res.Next() {
		var profile Profile
		err = res.Scan(&profile.name, &profile.birth, &profile.foods)
		if err != nil {
			log.Fatal(err)
		}

        if (profile.birth.Valid && profile.foods.Valid) {
          fmt.Printf("name: %s, birth: %s, foods: %s\n", 
                      profile.name, profile.birth.String, profile.foods.String)
        } else if profile.birth.Valid {
          fmt.Printf("name: %s, birth: %s, foods: NULL\n",
                      profile.name, profile.birth.String)
        } else if profile.foods.Valid {
          fmt.Printf("name: %s, birth: NULL, foods: %s\n",
                      profile.name, profile.foods.String)
        } else {
          fmt.Printf("name: %s, birth: NULL, foods: NULL\n",
                      profile.name)
        }
	}
}
```

###### 警告

我们在 `birth` 列中简单使用了类型 `NullString`。如果您想使用类型 `NullTime`，则需要在连接字符串中添加参数 `parseTime=true`。

###### 提示

或者您可以在查询执行期间使用 MySQL 的 `COALESCE()` 函数将 `NULL` 值转换为字符串。

```
sql := `SELECT name, 
 COALESCE(birth, '') as birthday
 FROM profile WHERE id = 9`
res, err := db.Query(sql)
defer res.Close()
```

### Java

对于 JDBC 程序，如果结果集中的某列可能包含 `NULL` 值，最好显式检查它们。具体方法是获取该值，然后调用 `wasNull()` 方法，如果该列为 `NULL` 则返回 true，否则返回 false。例如：

```
Object obj = rs.getObject (index);
if (rs.wasNull ())
{ /* the value's a NULL */ }
```

上述示例使用了 `getObject()`，但同样适用于其他 `get`*`XXX`*`()` 调用。

下面是打印结果集中每行作为逗号分隔值列表的示例，对于每个 `NULL` 值都打印 `"NULL"`：

```
Statement s = conn.createStatement();
s.executeQuery("SELECT name, birth, foods FROM profile");
ResultSet rs = s.getResultSet();
ResultSetMetaData md = rs.getMetaData();
int ncols = md.getColumnCount();
while (rs.next ()) { // loop through rows of result set
  for (int i = 0; i < ncols; i++) { // loop through columns
    String val = rs.getString(i+1);
    if (i > 0)
      System.out.print(", ");
    if (rs.wasNull())
      System.out.print("NULL");
    else
      System.out.print(val);
  }
  System.out.println();
}
rs.close();  // close result set
s.close();   // close statement
```

# 4.8 获取连接参数

## 问题

您需要获取一个脚本的连接参数，以便它可以连接到一个 MySQL 服务器。

## 解决方案

有几种方法可以做到这一点。从这里描述的备选方案中选择一种。

## 讨论

任何连接到 MySQL 的程序都会指定连接参数，例如用户名、密码和主机名。到目前为止显示的方法将连接参数直接放入尝试建立连接的代码中，但这并不是您的程序获取参数的唯一方式。本讨论简要概述了一些可用的技术：

将参数硬编码到程序中

参数可以在主源文件或程序使用的库文件中给出。这种技术非常方便，因为用户无需自己输入值，但也非常不灵活。要更改参数，必须修改程序。这也是不安全的，因为访问库的每个人都可以读取您的数据库凭据。

通过交互方式询问参数

在命令行环境中，您可以询问用户一系列问题。在 Web 或 GUI 环境中，您可以通过呈现表单或对话框来执行此操作。无论哪种方式，对于经常使用应用程序的人来说，由于每次需要输入参数，这变得很繁琐。

从命令行获取参数

您可以将此方法用于交互式运行的命令或从脚本中运行的命令。与交互式获取参数的方法一样，您必须为每个命令调用提供参数。（减轻这种负担的因素是，许多 Shell 允许您轻松地从历史列表中重新调用命令以重新执行。）如果以这种方式提供凭据，则此方法可能是不安全的。

从执行环境获取参数

这样做的最常见方法是在您的 Shell 启动文件之一中设置适当的环境变量（如*.profile*对于*sh*、*bash*、*ksh*；或者*.login*对于*csh*或*tcsh*）。然后在登录会话期间运行的程序可以通过检查它们的环境来获取参数值。

从一个单独的文件中获取参数

使用这种方法，将诸如用户名和密码之类的信息存储在程序连接到 MySQL 服务器之前可以读取的文件中。从与您的程序分离的文件中读取参数使您无需每次使用程序时都输入它们，而不需要将值硬编码到程序中。此外，将值存储在文件中使您能够集中存储用于多个程序的参数，并且出于安全目的，您可以设置文件访问模式以防止其他用户读取文件。

MySQL 客户端库本身支持一个选项文件机制，尽管并非所有的 API 都提供对其的访问。对于那些不支持的情况，可能存在解决方法。（例如，Java 支持使用属性文件，并提供用于读取它们的实用程序例程。）

使用多种方法的组合。

结合方法通常很有用，以便用户可以以不同的方式提供参数。例如，MySQL 客户端（如*mysql*和*mysqladmin*）在几个位置查找选项文件并读取所有存在的选项文件。然后，它们检查命令行参数以获取更多参数。这使用户可以在选项文件或命令行中指定连接参数。

获取连接参数的这些方法确实涉及安全问题：

+   任何将连接参数存储在文件中的方法都可能危及您系统的安全，除非该文件受到未经授权用户访问的保护。无论参数存储在源文件、选项文件还是调用命令并在命令行上指定参数的脚本中，这一点都是真实的。（如果其他用户具有对服务器的管理访问权限，则只能由 Web 服务器读取的 Web 脚本不算安全。）

+   在命令行或环境变量中指定的参数并不特别安全。当程序执行时，它的命令行参数和环境可能对运行`ps` `-e`等进程状态命令的其他用户可见。特别是，将密码存储在环境变量中最好限制在你是机器上唯一的用户或者你信任所有其他用户的情况下。

本节的其余部分讨论如何处理命令行参数以获取连接参数以及如何从选项文件中读取参数。

### 从命令行获取参数

标准客户端（如*mysql*和*mysqladmin*）用于命令行参数的约定是允许使用短选项或长选项指定参数。例如，用户名`cbuser`可以指定为`-u` `cbuser`（或`-ucbuser`）或`--user=cbuser`。此外，对于密码选项（`-p`或`--password`），在选项名称后可能省略密码值，以导致程序交互式提示输入密码。

这些命令选项的标准标记是`-h`或`--host`，`-u`或`--user`，以及`-p`或`--password`。您可以编写自己的代码来遍历参数列表，但使用为此目的编写的现有选项处理模块要容易得多。在`recipes`分发的*api*目录下，您会找到示例程序，展示如何处理命令参数以获取 Perl、Ruby、Python 和 Java 的主机名、用户名和密码。附带的 PDF 文件解释了每个程序的工作原理。

###### 注意

在尽可能大的程度上，程序模仿了标准 MySQL 客户端的选项处理行为。一个例外是选项处理库可能不允许使密码值变为可选，并且如果指定了密码选项但未提供密码值，则它们不提供以交互方式提示用户输入密码的方法。因此，程序是这样编写的：如果您使用 `-p` 或 `--password`，则必须提供该选项后面的密码值。

### 从选项文件获取参数

如果您的 API 支持，可以在 MySQL 选项文件中指定连接参数，并让 API 为您从文件中读取参数。对于不直接支持选项文件的 API，您可以安排读取存储参数的其他类型的文件，或者编写自己的函数来读取选项文件。

食谱 1.4 描述了 MySQL 选项文件的格式。我们假设您已经阅读了那里的讨论，并集中在如何在程序中使用选项文件。您可以在`recipes`分发的*api*目录下找到包含此处讨论代码的文件。

在 Unix 下，特定于用户的选项按约定规定在 *~/.my.cnf* 中（即在您的主目录中的 *.my.cnf* 文件中）。但是，MySQL 选项文件机制可以在多个不同的文件中查找，尽管没有任何选项文件是 *必须* 存在的。（有关 MySQL 程序查找它们的标准位置列表，请参见 食谱 1.4。）如果存在多个选项文件，并且在其中几个文件中指定了给定参数，则最后找到的值优先。

您编写的程序不会使用 MySQL 选项文件，除非您告诉它们：

+   Perl DBI 和 Ruby Mysql2 gem 提供了直接的 API 支持以读取选项文件；只需在连接到服务器时指示您想使用它们即可。可以指定仅读取特定文件，或者使用标准搜索顺序来查找多个选项文件。

+   PHP PDO、Connector/Python、Java 和 Go 不支持选项文件。（PDO MySQL 驱动程序除外，但如果使用 `mysqlnd` 作为底层库则不支持。）作为 PHP 的一种解决方法，我们将编写一个简单的选项文件解析函数。对于 Java，我们将采用使用属性文件的不同方法。对于 Go，我们将利用 `INI` 解析库。

尽管 Unix 下用户特定选项文件的常规名称是`.my.cnf`，位于当前用户的主目录下，但并没有规定你自己的程序必须使用这个特定的文件。你可以任意命名一个选项文件，并将其放置在任何你想要的位置。例如，你可以设置一个名为`mcb.cnf`的文件，并将其安装在`/usr/local/lib/mcb`目录下，以供访问`cookbook`数据库的脚本使用。在某些情况下，你可能甚至希望创建多个选项文件。然后，在任何给定的脚本中，选择适合脚本需要的访问权限的文件。例如，你可以有一个选项文件`mcb.cnf`，列出完全访问 MySQL 帐户的参数，以及另一个文件`mcb-readonly.cnf`，列出仅需要 MySQL 只读访问权限帐户的连接参数。另一种可能性是在同一个选项文件中列出多个组，并让你的脚本从适当的组中选择选项。

#### Perl

Perl DBI 脚本可以使用选项文件。为了利用这一点，请将适当的选项规范放置在数据源名称（DSN）字符串的第三个组件中：

+   要指定一个选项组，请使用``mysql_read_default_group=*`groupname`*``。这告诉 MySQL 在标准选项文件中搜索指定组和`[client]`组中的选项。将*`groupname`*值写入，不包含周围的方括号。（如果选项文件中的组以`[my_prog]`行开始，则将*`groupname`*值指定为`my_prog`。）要搜索标准文件，但仅在`[client]`组中查找，*`groupname`*应为`client`。

+   要命名特定的选项文件，请在 DSN 中使用``mysql_read_default_file=*`filename`*``。这样做时，MySQL 仅在该文件中查找并仅使用`[client]`组中的选项。

+   如果同时指定了选项文件和选项组，MySQL 只读取指定的文件，但在指定组和`[client]`组中查找选项。

以下示例告诉 MySQL 使用标准选项文件搜索顺序来查找`[cookbook]`和`[client]`组中的选项：

```
my $conn_attrs = {PrintError => 0, RaiseError => 1, AutoCommit => 1};
# basic DSN
my $dsn = "DBI:mysql:database=cookbook";
# look in standard option files; use [cookbook] and [client] groups
$dsn .= ";mysql_read_default_group=cookbook";
my $dbh = DBI->connect ($dsn, undef, undef, $conn_attrs);
```

下一个示例明确指定了位于`$ENV{HOME}`（运行脚本的用户的主目录）中的选项文件。因此，MySQL 仅在该文件中查找并使用`[client]`组中的选项：

```
my $conn_attrs = {PrintError => 0, RaiseError => 1, AutoCommit => 1};
# basic DSN
my $dsn = "DBI:mysql:database=cookbook";
# look in user-specific option file owned by the current user
$dsn .= ";mysql_read_default_file=$ENV{HOME}/.my.cnf";
my $dbh = DBI->connect ($dsn, undef, undef, $conn_attrs);
```

如果在`connect()`调用的用户名或密码参数中传递空值（`undef`或空字符串），`connect()`将使用在选项文件中找到的任何值。在`connect()`调用中，非空的用户名或密码会覆盖选项文件中的任何值。类似地，DSN 中指定的主机会覆盖选项文件中的任何值。使用此行为使得 DBI 脚本可以从选项文件和命令行获取连接参数：

1.  创建`$host_name`、`$user_name`和`$password`变量，每个变量的值为`undef`。然后解析命令行参数，如果命令行中存在相应的选项，则将变量设置为非`undef`值。（*api*目录下的*cmdline.pl* Perl 脚本演示了如何执行此操作。）

1.  在解析命令参数后，构建 DSN 字符串，并调用`connect()`。在 DSN 中使用`mysql_read_default_group`和`mysql_read_default_file`来指定选项文件的使用方式，如果`$host_name`不是`undef`，则将`host=$host_name`添加到 DSN 中。此外，将`$user_name`和`$password`作为用户名和密码参数传递给`connect()`。这些参数默认为`undef`；如果它们从命令行参数设置了值，则这些值将覆盖任何选项文件中的值。

如果脚本遵循此过程，则用户在命令行上提供的参数将传递给`connect()`，并优先于选项文件中的内容。

#### Ruby

Ruby Mysql2 脚本可以读取选项文件，通过`default_file`连接参数指定。如果要指定默认组，请使用选项`default_group`。

此示例使用标准的选项文件搜索顺序来查找`[cookbook]`和`[client]`组中的选项：

```
client = Mysql2::Client.new(:default_group => "cookbook", :database => "cookbook")
```

以下示例使用当前用户主目录中的*.my.cnf*文件获取`[client]`组中的参数：

```
client = Mysql2::Client.new(:default_file => "#{ENV['HOME']}/.my.cnf",↩
:database => "cookbook")
```

#### PHP

正如前面提到的，PDO MySQL 驱动程序不一定支持使用 MySQL 选项文件（如果使用`mysqlnd`作为底层库则不支持）。为了解决这个限制，可以使用一个函数来读取选项文件，例如下面列表中显示的`read_mysql_option_file()`函数。它的参数包括选项文件的名称和选项组名称或包含组名称的数组（组名称应该不带方括号）。然后，它读取文件中指定组或组的任何选项。如果没有给出选项组参数，则函数默认在`[client]`组中查找。返回值是一个选项名称/值对的数组，如果发生错误则返回`FALSE`。文件不存在并不是错误。（请注意，MySQL 选项文件中的带引号的选项值和选项值后的尾随`#`风格注释是合法的，但该函数不处理这些结构。）

```
function read_mysql_option_file ($filename, $group_list = "client")
{
  if (is_string ($group_list))           # convert string to array
    $group_list = array ($group_list);
  if (!is_array ($group_list))           # hmm ... garbage argument?
    return (FALSE);
  $opt = array ();                       # option name/value array
  if (!@($fp = fopen ($filename, "r")))  # if file does not exist,
    return ($opt);                       # return an empty list
  $in_named_group = 0;  # set nonzero while processing a named group
  while ($s = fgets ($fp, 1024))
  {
    $s = trim ($s);
    if (preg_match ("/^[#;]/", $s))              # skip comments
      continue;
    if (preg_match ("/^\[([^]]+)]/", $s, $arg))  # option group line
    {
      # check whether we are in one of the desired groups
      $in_named_group = 0;
      foreach ($group_list as $group_name)
      {
        if ($arg[1] == $group_name)
        {
          $in_named_group = 1;    # we are in a desired group
          break;
        }
      }
      continue;
    }
    if (!$in_named_group)         # we are not in a desired
      continue;                   # group, skip the line
    if (preg_match ("/^([^ \t=]+)[ \t]*=[ \t]*(.*)/", $s, $arg))
      $opt[$arg[1]] = $arg[2];    # name=value
    else if (preg_match ("/^([^ \t]+)/", $s, $arg))
      $opt[$arg[1]] = "";         # name only
    # else line is malformed
  }
  return ($opt);
}
```

下面是两个示例展示如何使用`read_mysql_option_file()`。第一个示例读取用户的选项文件以获取`[client]`组参数，并将它们用于连接到服务器。第二个示例读取系统范围的选项文件，*/etc/my.cnf*，并打印在那里找到的服务器启动参数（即`[mysqld]`和`[server]`组中的参数）：

```
$opt = read_mysql_option_file ("/home/paul/.my.cnf");
$dsn = "mysql:dbname=cookbook";
if (isset ($opt["host"]))
  $dsn .= ";host=" . $opt["host"];
$user = $opt["user"];
$password = $opt["password"];
try
{
  $dbh = new PDO ($dsn, $user, $password);
  print ("Connected\n");
  $dbh = NULL;
  print ("Disconnected\n");
}
catch (PDOException $e)
{
  print ("Cannot connect to server\n");
}

$opt = read_mysql_option_file ("/etc/my.cnf", array ("mysqld", "server"));
foreach ($opt as $name => $value)
  print ("$name => $value\n");
```

PHP 确实有一个 `parse_ini_file()` 函数，用于解析 *.ini* 文件。这些文件的语法与 MySQL 选项文件类似，因此您可能会发现此函数有用。但是，需要注意一些区别。假设您有一个文件写成这样：

```
[client]
user=paul

[client]
host=127.0.0.1

[mysql]
no-auto-rehash
```

标准 MySQL 选项解析将 `user` 和 `host` 值都视为 `[client]` 组的一部分，而 `parse_ini_file()` 仅返回最终 `[client]` 节的内容；`user` 选项会丢失。此外，`parse_ini_file()` 忽略未附带值的选项，因此 `no-auto-rehash` 选项也会丢失。

#### Go

Go-MySQL-Driver 不支持选项文件。然而，INI 解析库支持读取包含以 *`name=value`* 格式为行的属性文件。以下是一个示例属性文件：

```
# this file lists parameters for connecting to the MySQL server
[client]
user=cbuser
password=cbpass
host=localhost
```

函数 `MyCnf()` 展示了读取名为 *~/.my.cnf* 的属性文件以获取连接参数的一种方法：

```
import (
        "fmt"
        "os"
        "gopkg.in/ini.v1"
)

// Configuration Parser
func MyCnf(client string) (string, error) {
    cfg, err := ini.LoadSources(ini.LoadOptions{AllowBooleanKeys: true}, ↩
                                os.Getenv("HOME")+"/.my.cnf")
    if err != nil {
        return "", err
    }
    for _, s := range cfg.Sections() {
        if client != "" && s.Name() != client {
            continue
        }
        host := s.Key("host").String()
        port := s.Key("port").String()
        dbname := s.Key("dbname").String()
        user := s.Key("user").String()
        password := s.Key("password").String()
        return fmt.Sprintf("%s:%s@tcp(%s:%s)/%s", user, password, host, port, dbname),↩
               nil
    }
    return "", fmt.Errorf("No matching entry found in ~/.my.cnf")
}
```

在本章其他地方开发的 *cookbook.go* 中定义的 `MyCnf()` 函数（参见 Recipe 4.3）。它在 *api/06_conn_params* 目录中的 *mycnf.go* 文件中使用，在 `recipes` 发行版中找到：

```
// mycnf.go : Reads ~/.my.cnf file for DSN construct
package main

import (
	"fmt"
	"github.com/svetasmirnova/mysqlcookbook/recipes/lib"
)

func main() {
    fmt.Println("Calling db.MyCnf()")
    var dsn string

    dsn, err := cookbook.MyCnf("client")
    if err != nil {
	  fmt.Printf("error: %v\n", err)
    } else {
	  fmt.Printf("DSN is: %s\n", dsn)
    }
}
```

函数 `MyCnf()` 接受节名作为参数。如果要用任何其他名称替换 `[client]` 节，请调用 `MyCnf()`，如 `MyCnf("other")`，其中 `other` 是节的名称。

#### Java

JDBC MySQL Connector/J 驱动程序不支持选项文件。然而，Java 类库支持读取以 *`name=value`* 格式为行的属性文件。这与 MySQL 选项文件格式类似但并非完全相同（例如，属性文件不允许 ``[*`groupname`*]`` 行）。以下是一个简单的属性文件：

```
# this file lists parameters for connecting to the MySQL server
user=cbuser
password=cbpass
host=localhost
```

下面的程序 *ReadPropsFile.java* 展示了读取名为 *Cookbook.properties* 的属性文件以获取连接参数的一种方法。文件必须位于您的 `CLASSPATH` 变量命名的某个目录中，或者您必须使用完整路径名指定它（这里显示的示例假定文件位于 `CLASSPATH` 目录中）：

```
import java.sql.*;
import java.util.*;   // need this for properties file support

public class ReadPropsFile {
  public static void main(String[] args) {
    Connection conn = null;
    String url = null;
    String propsFile = "Cookbook.properties";
    Properties props = new Properties();

    try {
      props.load(ReadPropsFile.class.getResourceAsStream(propsFile));
    } catch (Exception e) {
      System.err.println("Cannot read properties file");
      System.exit (1);
    }
    try {
      // construct connection URL, encoding username
      // and password as parameters at the end
      url = "jdbc:mysql://"
            + props.getProperty("host")
            + "/cookbook"
            + "?user=" + props.getProperty("user")
            + "&password=" + props.getProperty("password");
      conn = DriverManager.getConnection(url);
      System.out.println("Connected");
    } catch (Exception e) {
      System.err.println("Cannot connect to server");
    } finally {
      try {
        if (conn != null) {
          conn.close();
          System.out.println("Disconnected");
        }
      } catch (SQLException e) { /* ignore close errors */ }
    }
  }
}
```

要在未找到命名属性时使 `getProperty()` 返回特定的默认值，请将该值作为第二个参数传递。例如，要将 `127.0.0.1` 作为默认的 `host` 值，可以像这样调用 `getProperty()`：

```
String hostName = props.getProperty("host", "127.0.0.1");
```

本章其他地方开发的 *Cookbook.java* 库文件（参见 Recipe 4.3）在 *lib* 目录中的版本中包含一个额外的库调用：一个基于此处讨论的概念的 `propsConnect()` 程序。要使用它，请设置 *Cookbook.properties* 属性文件的内容，并将文件复制到安装 *Cookbook.class* 的相同位置。然后，通过导入 `Cookbook` 类并调用 `Cookbook.propsConnect()` 来在程序中建立连接，而不是调用 `Cookbook.connect()`。

# 4.9 重置 profile 表

## 问题

在本章的示例中，您改变了`profile`表的原始内容，现在希望将其恢复，以便在处理其他配方时使用。

## 解决方案

使用*mysql*客户端重新加载表格。

## 讨论

最好将本章中使用的`profile`表重置为已知状态。切换到`recipes`分发的*tables*目录，并运行以下命令：

```
$ `mysql cookbook < profile.sql`
$ `mysql cookbook < profile2.sql`
```

几个后续章节中的语句使用了`profile`表；通过重新初始化它，您可以在运行那些章节中显示的语句时获得相同的结果。

本章讨论了我们的每个 API 提供的基本操作，用于处理与 MySQL 服务器的交互的各个方面。这些操作使您能够编写执行任何类型语句并检索结果的程序。到目前为止，我们使用了简单的语句，因为重点是在 API 上而不是 SQL 上。下一章则专注于 SQL，展示如何向数据库服务器提出更复杂的问题。
