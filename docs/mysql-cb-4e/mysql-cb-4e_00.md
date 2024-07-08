# 前言

MySQL 数据库管理系统因其快速、易于设置、使用和管理而广受欢迎。它在许多 Unix 和 Windows 的变种下运行，而基于 MySQL 的程序可以用多种语言编写。

MySQL 的流行性引发了解决其用户在如何解决特定问题方面的问题的需求。这就是*MySQL Cookbook*的目的：作为一个方便的资源，您可以在使用 MySQL 时用来快速解决问题或攻击特定类型问题的技术。当然，因为它是一本食谱书，它包含了一些配方：您可以遵循的简单指令，而不是从头开始开发自己的代码。它采用问题和解决方案的格式编写，旨在非常实用和使内容易于阅读和消化。它包含许多简短的章节，每个章节描述如何编写查询、应用技术或开发脚本来解决具有有限和特定范围问题。这本书并不开发成熟的复杂应用程序。相反，它通过帮助您解决困扰您的问题来协助您自行开发这些应用程序。

例如，一个常见问题是：<q>在编写查询时，如何处理数据值中的引号和特殊字符？</q> 这并不难，但当你不确定从哪里开始时，找出如何做会很令人沮丧。本书展示了应该做什么；它向您展示了从何处开始以及如何继续的方法。这些知识将会反复为您服务，因为一旦您了解所涉及的内容，您将能够将该技术应用于任何类型的数据，如文本、图像、声音或视频剪辑、新闻文章、压缩文件或 PDF 文档等。另一个常见问题是：<q>我可以同时从多个表中访问数据吗？</q> 答案是<q>可以</q>，而且很容易做到，只需了解适当的 SQL 语法即可。但在看到示例之前，这并不总是清楚。本书为您提供了这些示例。您将从本书中学到的其他技术包括如何：

+   使用 SQL 选择、排序和汇总行

+   在表之间查找匹配或不匹配

+   执行事务

+   确定日期或时间之间的间隔，包括年龄计算

+   标识或删除重复行

+   使用`LOAD DATA`正确读取您的数据文件或查找文件中的无效值

+   使用`CHECK`约束防止不良数据输入到您的数据库中

+   生成序列号以用作唯一行标识符

+   将视图用作<q>虚拟表</q>

+   编写存储过程和函数，设置触发器在插入或更新表行时激活以执行特定的数据处理操作，并使用事件调度程序按计划运行查询

+   设置复制

+   管理用户帐户

+   控制服务器日志记录

使用 MySQL 的一部分是理解如何与服务器通信——也就是说，如何使用 SQL，这是查询被制定的语言。因此，本书的一个主要重点是使用 SQL 来制定回答特定类型问题的查询。一个有帮助的工具用于学习和使用 SQL 是包含在 MySQL 发行版中的*mysql*客户端程序。你可以交互式地使用客户端向服务器发送 SQL 语句并查看结果。这非常有用，因为它提供了与 SQL 的直接接口；实际上，第一章专门讲述了*mysql*。

但仅仅能够发出 SQL 查询还不够。从数据库提取的信息通常需要进一步处理或以特定方式展示。如果你有具有复杂相互关系的查询，例如当你需要使用一个查询的结果作为其他查询的基础时会怎样？如果你需要生成一个具有非常特定格式要求的专门报告会怎样？这些问题带我们来到本书的另一个主要重点——如何编写与 MySQL 服务器通过应用程序接口（API）交互的程序。当你知道如何在编程语言的上下文中使用 MySQL 时，你就会获得其他利用 MySQL 能力的方法：

+   你可以保存查询结果并以后重复使用它们。

+   你可以充分利用通用编程语言的表达能力。这使得你可以根据查询的成功或失败，或返回的行内容来做出决策，并相应地调整采取的行动。

+   你可以以任何你喜欢的方式格式化和显示查询结果。如果你在编写命令行脚本，可以生成纯文本。如果是基于 Web 的脚本，可以生成 HTML 表格。如果是提取信息以传输到其他系统的应用程序，可能会生成以 XML 或 JSON 表示的数据文件。

将 SQL 与通用编程语言结合使用，可以为你发出查询和处理结果提供一个极其灵活的框架。编程语言增强了你执行复杂数据库操作的能力。但这并不意味着本书很复杂。它保持简单，展示如何使用易于理解和轻松掌握的技术构建小的构建模块。

我们将让你自己在你的程序中组合这些技术，这样你可以创建任意复杂的应用程序。毕竟，基因密码只基于四种核酸，但这些基本元素已经结合起来产生了我们周围看到的生物生命的惊人多样性。同样地，音阶只有 12 个音符，但在熟练作曲家手中，它们被编织在一起，产生丰富和无尽的音乐变化。同样地，当你拿出一组简单的技术，加上你的想象力，并将它们应用于你想要解决的数据库编程问题时，你可以创建的应用程序也许不是艺术品，但肯定是有用的，将帮助你和其他人更加高效地工作。

# 本书适合谁

无论是想要在个人项目（如博客或维基）中使用数据库的个人，还是专业数据库和网页开发人员，这本书都将对他们有所帮助。本书还适用于不了解如何使用 MySQL，但希望学习的人群。

如果你是 MySQL 的新手，这里有很多你可能不熟悉的使用方法。如果你有经验，可能已经熟悉这里提到的许多问题，但可能之前没有解决过，这本书应该会为你节省大量时间。利用本书中提供的技术，并在你自己的程序中使用它们，而不是从头开始编写代码。

本书内容从入门到高级涵盖广泛，因此，如果某个技术描述对你来说显而易见，可以跳过它。相反，如果你不理解某个技术，可以暂时搁置，稍后再回头看，也许在阅读其他技术之后会更明了。

# 本书内容概述

当你使用本书时，很可能是在开发一个应用程序，但不确定如何实现其中的某些部分。在这种情况下，你已经知道想要解决的问题类型；查看目录或索引，找到一个能展示你想要实现的操作的技术。理想情况下，该技术应该正是你所需要的。或者，你可以适应类似问题的技术来解决眼前的问题。我们解释了开发每一种技术所涉及的原理，以便你可以修改它，以适应你自己应用程序的特定需求。

另一种阅读这本书的方式是，无需特定问题，直接通读。这样可以让你更全面地了解 MySQL 能做什么，因此我们建议你偶尔翻阅这本书。如果你知道它解决的问题类型，这将是一个更有效的工具。

随着你深入后续章节，你会找到一些假定你已经掌握早期章节内容的实用技巧。这也适用于章节内部，后续部分经常使用前面章节讨论的技术。如果你跳到某章节，发现一个使用你不熟悉的技术的实例，可以查看目录或索引，找到该技术在前面章节的解释。例如，如果一个示例使用了你不理解的`ORDER BY`子句对查询结果进行排序，请参阅第九章，该章讨论了各种排序方法及其工作原理。

这里是每章的摘要，帮助你了解本书的内容概览。

第一章，“使用 mysql 客户端程序”，描述了如何使用标准的 MySQL 命令行客户端。*mysql*通常是人们使用的第一个或主要接口，了解如何利用其功能非常重要。该程序使你能够交互式地发出查询并查看其结果，因此非常适合快速实验。你还可以在批处理模式下使用它来执行 SQL 脚本或将其输出发送到其他程序。此外，该章还讨论了其他使用*mysql*的方法，如如何使长行更易读或生成各种格式的输出。

第二章，“使用 MySQL Shell”，介绍了由 MySQL 团队为 5.7 版本及更高版本开发的新 MySQL 命令行客户端。*mysqlsh* 在 SQL 模式下兼容*mysql*，同时支持 JavaScript 和 Python 编程接口的 NoSQL。使用 MySQL Shell，你可以轻松运行 SQL、NoSQL 查询，并自动化许多管理任务。

第三章，“MySQL 复制”，描述了如何设置和使用复制功能。本章的部分内容较为高级。然而，我们决定将其放在书的开头，因为复制对于能够抵御诸如数据损坏或硬件故障等灾难的稳定 MySQL 安装是必要的。实际上，任何生产环境中的 MySQL 安装都应该使用其中一种复制设置。虽然设置复制是一项管理任务，但我们认为所有 MySQL 用户都需要了解复制的工作原理，并因此编写出在源服务器和副本服务器上均能高效执行的有效查询。

第四章，“编写基于 MySQL 的程序”，演示了 MySQL 编程的基本要素：如何连接到服务器、发出查询、检索结果和处理错误。还讨论了如何在查询中处理特殊字符和`NULL`值，如何编写库文件以封装常用操作的代码，以及获取连接到服务器所需参数的各种方法。

第五章，“从表中选择数据”，涵盖`SELECT`语句的多个方面，这是从 MySQL 服务器检索数据的主要方式：指定要检索的列和行，处理`NULL`值，以及选择查询结果的一部分。后面的章节将更详细地讨论这些主题，但本章提供了这些概念的概述，这些概念是它们依赖的基础，如果您需要一些关于行选择的介绍性背景或者还不太了解 SQL。

第六章，“表管理”，涵盖表克隆、将结果复制到其他表中、使用临时表以及检查或更改表的存储引擎。

第七章，“处理字符串”，描述如何处理字符串数据。涵盖字符集和校对规则、字符串比较、处理大小写敏感问题、模式匹配、拆分和组合字符串，以及执行`FULLTEXT`搜索。

第八章，“处理日期和时间”，展示如何处理时间数据。描述了 MySQL 的日期格式以及如何以其他格式显示日期值。还涵盖了如何使用 MySQL 的特殊`TIMESTAMP`数据类型，如何设置时区，如何在不同的时间单位之间转换，如何进行日期算术以计算间隔或从一个日期生成另一个日期，以及如何执行闰年计算。

第九章，“排序查询结果”，描述如何按照所需顺序排列查询结果的行。这包括指定排序方向、处理`NULL`值、考虑字符串的大小写敏感性，以及按日期或部分列值排序。还提供了示例，展示如何对特殊类型的值进行排序，如域名、IP 数字和`ENUM`值。

第十章，“生成摘要”，展示评估一组数据的一般特征的技术，例如它包含多少个值或其最小、最大和平均值。

第十一章，“使用存储过程、触发器和定期事件”，描述如何编写存储在服务器端的存储函数和过程，触发器在修改表时激活，以及按计划执行的事件。

第十二章，“处理元数据”，讨论如何获取查询返回的数据的信息，例如结果中的行数或列数，或每列的名称和数据类型。还展示了如何询问 MySQL 可用的数据库和表，或确定表的结构。

第十三章，“导入和导出数据”，描述了如何在 MySQL 和其他程序之间传输信息。包括如何使用 `LOAD DATA`、将文件从一种格式转换为另一种格式，以及确定适合数据集的表结构。

第十四章，“数据验证和重排”，描述了如何从数据文件中提取或重新排列列，检查和验证数据，并重新编写诸如日期之类常见格式的数值。

第十五章，“生成和使用序列”，讨论了 `AUTO_INCREMENT` 列，MySQL 生成序列号的机制。展示了如何生成新的序列值或确定最近的值，如何重新排列列以及如何使用序列生成计数器。还展示了如何使用 `AUTO_INCREMENT` 值来维护表之间的主从关系，包括需要避免的陷阱。

第十六章，“使用连接和子查询”，展示了如何执行从多个表中选择行的操作。它演示了如何比较表以找到匹配或不匹配项，生成主从列表和摘要，并列举多对多关系。

第十七章，“统计技术”，说明了如何生成描述性统计信息、频率分布、回归和相关性。还涵盖了如何对一组行进行随机化或从集合中随机选择行的内容。

第十八章，“处理重复项”，讨论了如何识别、计数和删除重复行，以及如何在首次防止它们的发生。

第十九章，“处理 JSON”，说明了如何在 MySQL 中使用 JSON。涵盖了验证、搜索和操作 JSON 数据等主题。本章还讨论了如何将 MySQL 用作文档存储。

第二十章，“执行事务”，展示了如何处理必须作为一个单位一起执行的多个 SQL 语句。讨论了如何控制 MySQL 的自动提交模式以及如何提交或回滚事务。

第二十二章，“服务器管理”，专为数据库管理员编写。涵盖了服务器配置、插件接口和日志管理。

第二十三章，“监视 MySQL 服务器”，说明了如何监视和排除 MySQL 问题，如启动或连接失败等。展示了如何使用 MySQL 日志文件、内置工具和标准操作系统实用程序获取关于 MySQL 查询和内部结构性能的信息。

第二十四章，“安全”，是另一个管理员章节。讨论了用户账户管理，包括创建账户、设置密码和分配权限。还描述了如何实施密码策略，查找和修复不安全账户，以及设置或取消密码过期。

# 本书中使用的 MySQL API

MySQL 编程接口支持许多语言，包括 C、C++、Eiffel、Go、Java、Perl、PHP、Python、Ruby 和 Tcl。鉴于这一事实，编写 MySQL 食谱对作者来说是一个挑战。本书应该提供许多有趣和有用的 MySQL 操作方法，但是应该使用哪种 API 或多种 API？在每种语言中显示每个配方的实现要么覆盖很少的配方，要么得到一个非常非常大的书！当不同语言中的实现非常相似时，还会导致冗余。另一方面，利用多种语言是值得的，因为对于解决特定问题，通常有一种语言比另一种更合适。

为了解决这个困境，我们选择了少量 API 来编写本书中的配方。这使得它的范围可管理，同时允许从多个 API 中选择：

+   Perl 的 DBI 模块

+   Ruby，使用 Mysql2 gem

+   PHP，使用 PDO 扩展

+   Python，使用 MySQL Connector/Python 驱动程序的 DB API

+   Go，使用 Go-MySQL-Driver 用于 `sql` 接口

+   Java，使用 MySQL Connector/J 驱动程序的 JDBC 接口

为什么选择这些语言？Perl 是一种广泛使用的语言，第一版本书出版时编写 MySQL 程序非常流行，至今仍在许多应用中使用。Ruby 拥有易于使用的数据库访问模块。PHP 在 Web 上广泛部署。Go 近来变得非常流行，并在许多 MySQL 应用中取代其他语言，尤其是 Perl。Python 和 Java 各自拥有大量的追随者。

我们认为这些语言一起很好地反映了 MySQL 程序员现有用户群体的大多数。如果您喜欢本书未显示的某些语言，请务必仔细阅读 第四章，熟悉本书的主要 API。了解如何使用这里使用的编程接口执行数据库操作将有助于您为其他语言翻译配方。

# 版本和平台说明

本书中的代码开发是在 MySQL 5.7 和 8.0 下进行的。由于 MySQL 定期添加新功能，因此某些示例在旧版本下不起作用。例如，MySQL 5.7 引入了群组复制，MySQL 8.0 引入了 `CHECK` 约束和通用表达式。

我们不假设您正在使用 Unix，尽管这是我们自己喜欢的开发平台。（在本书中，“Unix”也指 Unix-like 系统，如 Linux 和 macOS X。）这里的大部分材料适用于 Unix 和 Windows。

# 本书使用的约定

本书使用以下字体约定：

`Constant` `width`

用于程序清单，以及段落内引用程序元素，如变量或函数名称、数据库、数据类型、环境变量、语句和关键字。

**`常量`** **`宽度`** **`粗体`**

用于指示在运行命令时键入的文本。

*`常量`* *`宽度`* *`斜体`*

用于指示可变输入；您应该替换为自己选择的值。

*斜体*

用于 URL、主机名、目录和文件名、Unix 命令和选项、程序，偶尔用于强调。

###### 提示

这个元素表示一个提示或建议。

###### 注意

这个元素表示警告或注意事项。

###### 注意

这个元素表示一般说明。

通常会显示带有提示符的命令，以说明它们的使用上下文。从命令行发出的命令显示为`$`提示符：

```
$ `chmod 600 my.cnf`
```

Unix 用户习惯看到的提示符，但这并不一定表示该命令仅在 Unix 下有效。除非另有说明，带有`$`提示符的命令通常也应在 Windows 下工作。

如果你应该以 Unix 的`root`用户身份运行命令，则提示符是`#`：

```
# `perl -MCPAN -e shell`
```

专用于 Windows 的命令使用`C:\>`提示符：

```
C:\> `"C:\Program Files\MySQL\MySQL Server 5.6\bin\mysql"`
```

从*mysql*客户端程序中发出的 SQL 语句显示为`mysql>`提示符并以分号终止：

```
mysql> `SELECT * FROM my_table;`
```

对于显示查询结果的示例，如同使用*mysql*时看到的那样，有时会截断输出，使用省略号（`...`）表示结果包含比显示的更多行。以下查询生成大量输出行，其中中间的部分已被省略：

```
mysql> `SELECT name, abbrev FROM states ORDER BY name;`
+----------------+--------+
| name           | abbrev |
+----------------+--------+
| Alabama        | AL     |
| Alaska         | AK     |
| Arizona        | AZ     |
…
| West Virginia  | WV     |
| Wisconsin      | WI     |
| Wyoming        | WY     |
+----------------+--------+
```

显示仅 SQL 语句语法的示例不包括`mysql>`提示符，但会根据需要包括分号，以更清楚地指示语句结束。例如，这是一个单一语句：

```
CREATE TABLE t1 (i INT)
SELECT * FROM t2;
```

但这个示例代表两个语句：

```
CREATE TABLE t1 (i INT);
SELECT * FROM t2;
```

分号在*mysql*中作为语句终止符号，是一种符号方便，但不是 SQL 本身的一部分，因此当你在程序中（例如使用 Perl 或 Java）发出 SQL 语句时，不要包括终止分号。

如果语句或命令输出过长而无法适合书页，我们使用符号`↩`来显示该行已缩进以适应。

```
mysql> `SELECT 'Mysql: The Definitive Guide to Using, Programming,`↩
    -> `and Administering Mysql 4 (Developer\'s Library)' AS book;`
+-----------------------------------------------------+
| book                                                |
+-----------------------------------------------------+
| Mysql: The Definitive Guide to Using, Programming,↩
  and Administering Mysql 4 (Developer's Library)     |
+-----------------------------------------------------+
1 row in set (0,00 sec)
```

# MySQL Cookbook 配套的 GitHub 存储库

*MySQL Cookbook*有一个[配套 GitHub 存储库](https://github.com/svetasmirnova/mysqlcookbook)，您可以在其中获取本书开发过程中开发的示例的源代码和样本数据，以及辅助文档。

## 配方源代码和数据

本书中的示例基于名为`recipes`的发行版的源代码和样本数据，可在配套的 GitHub 存储库中找到。

`recipes`发行版是示例的主要来源，并且在整本书中都有引用。解压缩后，无论是压缩的 TAR 文件（*recipes.tar.gz*）还是 ZIP 文件（*recipes.zip*），都会创建一个名为*mysqlcookbook-*`VERSION`*/recipes*的目录。

使用`recipes`发行版可以节省大量输入时间。例如，当你在书中看到描述数据库表结构的`CREATE` `TABLE`语句时，通常可以在*tables*目录下找到相应的 SQL 批处理文件，而无需手动输入定义。切换到*tables*目录并执行以下命令，其中*`filename`*是包含`CREATE` `TABLE`语句的文件名：

```
$ `mysql cookbook <` *`filename`*
```

如果需要指定 MySQL 用户名或密码选项，请将它们添加到命令行中。

要从`recipes`发行版导入所有表格，请使用以下命令：

```
$ `mysql cookbook <` cookbook.sql
```

`recipes`发行版包含书中显示的程序，但在许多情况下还包括其他语言的实现。例如，书中使用 Python 展示的脚本可能在`recipes`发行版中也以 Perl、Ruby、PHP、Go 或 Java 的形式提供。如果您希望将书中程序转换为其他语言，这可能会节省您的翻译工作。

## Amazon Review Data (2018)

Amazon 相关评论数据在第七章，“处理字符串”中可找到，网址为[*http://deepyeti.ucsd.edu/jianmo/amazon/index.html*](http://deepyeti.ucsd.edu/jianmo/amazon/index.html)，可以通过这个[表单](https://forms.gle/A8hBfPxKkKGFCP238)下载。Justifying recommendations using distantly-labeled reviews and fined-grained aspects Jianmo Ni, Jiacheng Li, Julian McAuley Empirical Methods in Natural Language Processing (EMNLP), 2019.

## MySQL Cookbook 配套文件

之前*MySQL Cookbook*版本中包含的一些附录现在可以在配套的 GitHub 仓库中单独找到。它们为书中涵盖的主题提供了背景信息。

<q>从命令行执行程序</q> 提供了在命令提示符下执行命令和设置环境变量（如`PATH`）的说明。

# 获取 MySQL 及相关软件

要运行本书中的示例，您需要访问 MySQL，以及您想要使用的编程语言的适当的 MySQL 特定接口。以下说明描述了需要哪些软件以及如何获取它们。

如果访问由他人运行的 MySQL 服务器，则只需在您自己的机器上安装 MySQL 客户端软件。要运行自己的服务器，您需要完整的 MySQL 发行版。

要编写自己的基于 MySQL 的程序，您需要通过语言特定的 API 与服务器通信。Perl 和 Ruby 接口依赖于 MySQL C API 客户端库来处理低级客户端-服务器协议。PHP 接口也是如此，除非 PHP 配置为使用`mysqlnd`，即本机协议驱动程序。对于 Perl 和 Ruby，您必须先安装 C 客户端库和头文件。PHP 包含所需的 MySQL 客户端支持文件，但必须编译时启用 MySQL 支持，否则将无法使用它。Python、Go 和 Java 的 MySQL 驱动程序直接实现客户端-服务器协议，因此它们不需要 MySQL C 客户端库。

如果您的系统上已经安装了客户端软件，则可能不需要自己安装它。如果您与提供 MySQL 访问权限的 Internet 服务提供商（ISP）有帐户，则这种情况很常见。

## MySQL

MySQL 发行版和文档，包括*MySQL 参考手册*和*MySQL Shell*，可以从[*http://dev.mysql.com/downloads*](http://dev.mysql.com/downloads)和[*http://dev.mysql.com/doc*](http://dev.mysql.com/doc)获取。

如果需要安装 MySQL C 客户端库和头文件，它们将包含在从源分发安装 MySQL 时，或使用除 RPM 或 DEB 二进制分发以外的二进制（预编译）分发时。在 Linux 下，您可以选择使用 RPM 或 DEB 文件安装 MySQL，但除非安装开发 RPM 或 DEB，否则不会安装客户端库和头文件。（服务器、标准客户端程序以及开发库和头文件有单独的 RPM 或 DEB 文件。）

## Perl 支持

可以在[Perl 编程语言网站](http://www.perl.org)上获取一般的 Perl 信息。

您可以从[Comprehensive Perl Archive Network (CPAN)](http://cpan.perl.org)获取 Perl 软件。

要编写基于 MySQL 的 Perl 程序，您需要 DBI 模块和特定于 MySQL 的 DBD 模块，DBD::mysql。

要在 Unix 下安装这些模块，请让 Perl 自己帮助你。例如，要安装 DBI 和 DBD::mysql，请运行以下命令（可能需要以`root`身份执行）：

```
# `perl -MCPAN -e shell`
cpan> `install DBI`
cpan> `install DBD::mysql`
```

如果最后一个命令报告测试失败，请改用`force` `install` `DBD::mysql`。在 Windows 的 ActiveState Perl 中，请使用*ppm*实用程序：

```
C:\> `ppm`
ppm> `install DBI`
ppm> `install DBD-mysql`
```

您还可以使用 CPAN shell 或*ppm*安装本书中提到的其他 Perl 模块。

安装了 DBI 和 DBD::mysql 模块后，可以从命令行获取文档：

```
$ `perldoc DBI`
$ `perldoc DBI::FAQ`
$ `perldoc DBD::mysql`
```

也可以从[Perl 网站](http://dbi.perl.org)获取文档。

## Ruby 支持

主要的[Ruby 网站](http://www.ruby-lang.org)提供了访问 Ruby 发行版和文档的途径。

Ruby MySQL2 gem 可以从[RubyGems](http://www.rubygems.org)获取。

## PHP 支持

主要的 [PHP 网站](http://www.php.net) 提供 PHP 分发和文档，包括 PDO 文档。

PHP 源分发包括 PDO 支持，因此您无需单独获取它。但是，在配置分发时，您必须启用 PDO 对 MySQL 的支持。如果使用二进制分发，请确保其中包含 PDO MySQL 支持。

## Python 支持

主要的 [Python 网站](http://www.python.org) 提供 Python 分发和文档。DB API 数据库访问接口的一般文档位于 [Python Wiki](http://bit.ly/py-wiki) 上。

对于 MySQL Connector/Python，这是为 DB API 提供 MySQL 连接的驱动模块，分发和文档可从 [*http://bit.ly/py-connect*](http://bit.ly/py-connect) 和 [*http://bit.ly/py-dev-guide*](http://bit.ly/py-dev-guide) 获取。

## Go 支持

主要的 [Go 网站](https://go.dev/) 提供 Go 分发和文档，包括 sql 包和文档。

Go-MySQL-Driver 及其文档可从 [GitHub go-sql-driver/mysql 仓库](https://github.com/go-sql-driver/mysql) 获取。

## Java 支持

您需要一个 Java 编译器来构建和运行 Java 程序。*javac* 编译器是 Java 开发工具包（JDK）的一部分。如果您的系统未安装 JDK，则可以在 macOS、Linux 和 Windows 上获取版本，请访问 [Oracle 的 Java 网站](http://www.oracle.com/technetwork/java)。同一网站提供 JDBC、Servlet、JavaServer Pages（JSP）以及 JSP 标准标签库（JSTL）的文档（包括规范）。

对于 MySQL Connector/J，这是为 JDBC 接口提供 MySQL 连接的驱动程序，分发和文档可从 [*http://bit.ly/jconn-dl*](http://bit.ly/jconn-dl) 和 [*http://bit.ly/j-dev-guide*](http://bit.ly/j-dev-guide) 获取。

# 使用代码示例

如果您在使用代码示例时遇到技术问题或困难，请发送电子邮件至 *bookquestions@oreilly.com*。

本书旨在帮助您完成工作。一般来说，如果本书提供了示例代码，您可以在自己的程序和文档中使用它。除非您复制了大量代码，否则无需联系我们以获得许可。例如，编写使用本书多个代码片段的程序不需要许可。销售或分发 O’Reilly 图书中的示例则需要许可。引用本书并引用示例代码回答问题则不需要许可。将本书大量示例代码整合到产品文档中则需要许可。

我们感谢您的支持，但不要求署名。典型的署名包括书名、作者、出版商和 ISBN。例如：“*MySQL Cookbook, Fourth Edition* 作者 Sveta Smirnova, Alkin Tezuysal（O’Reilly）。版权 2022 Sveta Smirnova, Alkin Tezuysal, 978-1-492-09309-1。”

如果您觉得您对代码示例的使用超出了合理使用范围或上述许可，请随时通过*permissions@oreilly.com*与我们联系。

# O'Reilly 在线学习

###### 注意

40 多年来，[O'Reilly Media](http://oreilly.com) 一直为公司提供技术和业务培训、知识和见解，帮助它们取得成功。

我们独特的专家和创新者网络通过图书、文章和我们的在线学习平台分享他们的知识和专业知识。O’Reilly 的在线学习平台为您提供按需访问的现场培训课程、深入学习路径、交互式编码环境以及来自 O’Reilly 和其他 200 多家出版商的大量文本和视频。欲了解更多信息，请访问 [*http://oreilly.com*](http://oreilly.com).

# 如何联系我们

请将关于本书的评论和问题发送至出版商：

+   O’Reilly Media, Inc.

+   Gravenstein Highway North 1005

+   CA 95472，Sebastopol

+   800-998-9938（美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为这本书制作了一个网页，列出勘误、示例和任何额外信息。您可以访问此页面：*https://www.oreilly.com/library/view/mysql-cookbook-4th/9781492093152/*.

发送电子邮件至 *bookquestions@oreilly.com* 对本书发表评论或提出技术问题。

欲了解更多关于我们的图书、课程、会议和新闻的信息，请访问我们的网站：[*https://www.oreilly.com*](https://www.oreilly.com).

在 LinkedIn 上找到我们：[*https://linkedin.com/company/oreilly-media*](https://linkedin.com/company/oreilly-media)

在 Twitter 上关注我们：[*https://twitter.com/oreillymedia*](https://twitter.com/oreillymedia)

在 YouTube 上观看我们：[*https://www.youtube.com/oreillymedia*](https://www.youtube.com/oreillymedia)

# 致谢

感谢每一位读者阅读我们的书籍。我们希望它能为您服务，并且您觉得它有用。

## 来自 Paul DuBois，第三版

感谢我的技术审阅者 Johannes Schlüter、Geert Vanderkelen 和 Ulf Wendel。他们提出了多处改正和建议，极大地改善了文本，我非常感激他们的帮助。

Andy Oram 催促我开始第三版，并担任其编辑，Nicole Shelby 指导本书的制作，Kim Cofer 和 Lucie Haskins 进行了校对和索引工作。

感谢我的妻子 Karen，在写作过程中一直给予我鼓励和支持，这意义非凡。

## 来自 Sveta Smirnova 和 Alkin Tezuysal

我们衷心感谢本书的技术审阅者对本书的宝贵贡献。

Gillian Gunson 不仅提供了全面的技术反馈，还展示了如何让我们的文本更易被具有不同背景的人理解。她的语言建议帮助我们使菜谱更易读。她对细节的关注帮助我们识别了在数据库负载增长时可能出现的不准确和潜在风险区域。Gillian 还审阅了所有的代码示例，并建议如何使 Ruby 和 Java 代码更符合当前的标准。

Ege Gunes 审阅了所有 Go 语言示例，以便符合 Go 标准的风格。

Karthik Appigatla, Timur Solodovnikov, Daniel Guzman Burgos, Vladmir Fedorkov reviewed selected chapters of the book. Their suggested corrections helped us to improve the book a lot.

Andy Kwan 邀请我们撰写本书的第四版。Amelia Blevins 和 Jeff Bleiel 是我们的编辑，帮助使本书更易读。Rita Fernando 审阅了几章，并提供了反馈，使我们能够更符合 O’Reilly 的标准。

## Sveta Smirnova 提供。

我要感谢在 Percona 支持部门的同事们，他们理解我需要在书上进行第二轮工作，并允许我在需要时休息。

特别感谢我的丈夫 Serguei Lassounov，在我所有的职业努力中一直支持我。

## Alkin Tezuysal 提供。

我要感谢我的妻子 Aslihan 以及两个女儿 Ilayda 和 Lara，在我需要集中精力使用他们的家庭时间来写这本书时，他们的耐心和支持。

特别感谢我的 PlanetScale 同事和团队，特别是 Deepthi Sigireddi，她的额外关心和支持。特别感谢 MySQL 社区、朋友和家人们。

我也想感谢 Sveta Smirnova，在我第一次撰写书籍的旅程中给予我的无尽支持和指导。
