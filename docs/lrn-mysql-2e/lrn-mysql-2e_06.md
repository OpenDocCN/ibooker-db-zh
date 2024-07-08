# 第十章：备份和恢复

任何数据库管理员最重要的任务是备份数据。正确和经过测试的备份和恢复程序可以拯救公司，从而拯救工作。错误会发生，灾难会发生，错误也会发生。MySQL 是一个强大的软件，但它并不完全没有错误或崩溃。因此，了解为什么以及如何执行备份是至关重要的。

除了保存数据库内容，大多数备份方法还可以用于另一个重要目的：在不同系统之间复制数据库内容。虽然复制的重要性可能不如在发生损坏时救命那么重要，但这种复制对于绝大多数数据库操作人员来说是例行操作。开发人员通常需要使用类似于生产环境的下游环境。质量保证人员可能需要一个寿命仅为一小时的易失性环境。分析可以在专用主机上运行。这些任务中的一些可以通过复制来解决，但任何副本都是从恢复的备份开始的。

本章首先简要回顾了两种主要类型的备份，并讨论了它们的基本属性。然后，它查看了 MySQL 世界中一些用于备份和恢复目的的工具。覆盖每个工具及其参数超出了本书的范围，但通过本章的学习，您应该了解如何进行 MySQL 数据的备份和恢复。我们还将探讨一些基本的数据传输场景。最后，本章概述了一个强大的备份架构，您可以将其作为工作的基础。

对于我们认为是一个良好的备份策略的概述可以在 “数据库备份策略入门” 中找到。在决定策略之前，理解工具和运作部分非常重要，因此该部分排在最后。

# 物理备份和逻辑备份

广义上讲，大多数备份工具可以归为两大类：逻辑备份和物理备份。*逻辑* 备份操作的是内部结构：数据库（模式）、表、视图、用户和其他对象。*物理* 备份关注的是数据库结构的操作系统端表示：数据文件、事务日志等。

用一个例子可能更容易解释。想象一下备份 MySQL 数据库中的单个 MyISAM 表。正如您将在本章后面看到的那样，InnoDB 存储引擎更复杂地正确备份。知道 MyISAM 不是事务性的，并且对这个表没有正在进行的写入，我们可以继续复制与其相关的文件。这样做，我们创建了表的物理备份。我们也可以运行`SELECT *`和`SHOW CREATE TABLE`语句，并将这些语句的输出保存在某个地方。这是逻辑备份的一个非常基本的形式。当然，这只是简单的例子，在现实中，获取这两种备份的过程会更复杂和微妙。然而，这些想象中备份之间的概念差异可以转移并应用于任何逻辑和物理备份。

## 逻辑备份

逻辑备份关注的是*实际数据*，而不是*物理表示*。正如您已经看到的那样，这种备份不会复制任何现有的数据库文件，而是依赖于查询或其他手段来获取所需的数据库内容。结果通常是一些文本表示，尽管这并不是保证，逻辑备份的输出也可能是二进制编码的。让我们看一些更多这种备份可能是什么样子的例子，然后讨论它们的特性。

这里有一些逻辑备份的例子：

+   使用`SELECT ... INTO OUTFILE`语句将查询的表数据保存到外部*.csv*文件中，我们在“将数据写入逗号分隔文件”中进行了介绍。

+   可以将表或任何其他对象的定义保存为 SQL 语句。

+   一个或多个`INSERT` SQL 语句，运行在一个数据库和一个空表上，将该表填充到保留状态。

+   所有曾经触及特定表或数据库并修改数据或模式对象的语句记录。这里我们指的是 DML 和 DDL 命令；您应该熟悉这两种类型，它们在第三章和第四章中有详细介绍。

###### 注意

那个最后的例子实际上展示了 MySQL 中复制和时间点恢复是如何工作的。我们稍后会讨论这些主题，您会看到术语*逻辑*不仅仅适用于备份。

逻辑备份的恢复通常通过执行一个或多个 SQL 语句完成。延续我们之前的例子，让我们回顾一下恢复的选项：

+   从*.csv*文件中的数据可以使用`LOAD DATA INFILE`命令加载到表中。

+   可以通过运行 DDL SQL 语句来创建或重新创建表。

+   `INSERT` SQL 语句可以使用`mysql` CLI 或任何其他客户端执行。

+   回放数据库中运行的所有语句将其恢复到最后一个语句之后的状态。

逻辑备份具有一些有趣的特性，使它们在某些情况下极其有用。通常情况下，逻辑备份是某种形式的文本文件，主要由 SQL 语句组成。然而，并非必须如此，并不是其定义属性（尽管这是一个有用的属性）。创建逻辑备份的过程通常还涉及执行一些查询。这些是重要的特性，因为它们允许灵活性和可移植性。

逻辑备份非常灵活，因为它们使得非常容易备份数据库的部分内容。例如，您可以备份模式对象而不包括其内容，或者轻松地备份数据库中的几个表。甚至可以备份表的部分数据，这通常是物理备份所无法做到的。一旦备份文件准备好，您可以使用工具手动或自动地查看和修改它，这是使用数据库文件副本难以做到的事情。

可移植性来源于逻辑备份可以轻松加载到运行在不同操作系统和架构上的不同版本的 MySQL 中。通过一些修改，您实际上可以将从一个 RDBMS 中获取的逻辑备份加载到完全不同的 RDBMS 中。大多数数据库迁移工具内部使用逻辑复制正是因为这一事实。这种特性还使得这种备份类型适用于远程备份云管理数据库和它们之间的迁移。

逻辑备份的另一个有趣特性是它们在对抗*损坏*（即物理数据文件的物理损坏）方面非常有效。数据中的错误仍然可能被引入，例如由软件中的错误或存储介质逐渐退化引起。关于损坏及其对应的完整性的话题非常广泛，但这个简要解释现在应该足够了。

一旦数据文件损坏，数据库可能无法从中读取数据并处理查询。由于损坏往往是悄无声息发生的，您可能不知道何时发生。然而，如果生成逻辑备份时没有错误，那意味着它是完好的，数据也是正确的。损坏可能发生在*辅助索引*（任何非主索引；详见第四章，*使用数据库结构*了解更多细节），因此，进行全表扫描的逻辑备份可能正常生成而不会遇到错误。简言之，逻辑备份既可以帮助您早期检测到损坏（因为它扫描所有表），又可以帮助您保存数据（因为最后一次成功的逻辑备份将有一个正确的副本）。

所有逻辑备份的固有问题源于它们是通过对运行中的数据库系统执行 SQL 语句来创建和恢复的。虽然这样做可以提供灵活性和可移植性，但这也意味着这些备份会给数据库带来负载，并且通常非常慢。数据库管理员总是对某人运行不加选择地读取表中所有数据的查询感到不满，而逻辑备份工具通常就是这样做的。类似地，逻辑备份的恢复操作通常会像来自常规客户端的每个语句一样进行解释和执行。这并不意味着逻辑备份是不好或不应该使用的，但这是一个必须记住的权衡。

## 物理备份

逻辑备份关注的是数据库内容中的数据，而物理备份关注的是操作系统文件和内部关系型数据库的工作原理。记住，在备份 MyISAM 表的示例中，物理备份是表的文件副本。大多数此类备份和工具关注于复制和传输整个或部分数据库文件。

一些物理备份的示例包括以下内容：

+   冷数据库目录复制，意味着在数据库关闭时进行（与热复制相反，这是在数据库运行时进行）。

+   存储快照，用于数据库使用的卷和文件系统。

+   表数据文件的副本。

+   某种形式的数据库数据文件更改流。大多数关系型数据库管理系统使用这样的流进行崩溃恢复，有时用于复制；InnoDB 的重做日志是类似的概念。

物理备份的恢复通常通过复制回文件并使其一致来完成。让我们回顾前面示例的恢复选项：

+   冷备份可以移动到所需的位置或服务器，然后由 MySQL 实例（旧的或新的）用作数据目录。

+   快照可以在原地或另一个卷上恢复，然后由 MySQL 使用。

+   可以将表文件放置在现有文件的位置。

+   对数据文件的更改流重播将其恢复到最后一次的状态。

最简单的物理备份是一个冷数据库目录备份。是的，它很简单和基础，但它是一个非常强大的工具。

物理备份与逻辑备份不同，非常严格，对于可以备份的内容以及备份可以使用的位置几乎没有控制余地。 一般来说，大多数物理备份只能用于恢复数据库或表的确切状态。 通常，这些备份还会限制目标数据库软件版本和操作系统。 经过一些工作，您可以将从 MySQL 到 PostgreSQL 的逻辑备份还原。 但是，在 Linux 上完成的 MySQL 数据目录的冷备份在 Windows 上还原可能不起作用。 此外，如果没有对数据库服务器的物理访问权限，您将无法进行物理备份。 这意味着在云中的托管数据库上执行此类备份是不可能的：供应商可能正在后台执行物理备份，但您可能无法取回备份文件。

由于物理备份本质上是原始备份页面的复制或子集，原始备份中存在的任何损坏都将包含在备份中。 记住这一点很重要，因为这一属性使物理备份不适合用于对抗损坏。

您可能想知道为什么要使用这种看似不便的备份方式。 原因在于物理备份很快。 通过操作操作系统甚至存储级别，物理备份方法有时是实际上唯一可能的备份数据库的方法。 例如，多 TB 卷的存储快照可能需要几秒钟或几分钟，而对逻辑备份的数据进行查询和流式传输可能需要几小时或几天。 恢复也是如此。

## 逻辑和物理备份概述

现在我们已经涵盖了备份的两个类别，并准备开始探索在 MySQL 世界中用于这些备份的实际工具。 在我们开始之前，让我们总结一下逻辑备份和物理备份之间的区别，并快速查看用于创建它们的工具的属性。

逻辑备份的特性：

+   包含逻辑结构的描述和内容

+   可以人为阅读和编辑

+   相对缓慢进行备份和恢复

逻辑备份工具包括：

+   非常灵活，允许您重命名对象，合并单独的源，执行部分恢复等

+   通常不限于特定的数据库版本或平台

+   能够从已损坏的表中提取数据并保护免受损坏

+   适用于备份远程数据库（例如，在云中）

物理备份的特性：

+   是数据文件部分或整个文件系统/卷的字节复制

+   快速进行备份和恢复

+   提供很少的灵活性，并且恢复时始终产生相同的结构

+   可以包括损坏页面

物理备份工具包括：

+   操作繁琐

+   通常不允许轻松跨平台或甚至跨版本移植

+   没有操作系统访问权限无法备份远程数据库

###### 提示

这些并非相互冲突的方法。事实上，一般的好主意是定期执行这两种备份类型。它们服务于不同的目的并满足不同的需求。

# 复制作为备份工具

复制是一个非常广泛的主题，即将在后续章节详细讨论。在本节中，我们简要讨论了复制如何与数据库备份和恢复的概念相关联。

简而言之，复制不能替代备份。复制的具体情况是产生目标数据库的全面或部分副本。这使得您可以在许多但不是所有可能的涉及 MySQL 的故障场景中使用复制。让我们回顾两个例子。它们在本章后面也会有所帮助。

###### 注意

在 MySQL 世界中，复制是一种逻辑备份类型。这是因为它基于传输逻辑 SQL 语句。

## 基础设施故障

基础设施容易发生故障：驱动器损坏，停电，火灾发生。几乎没有系统可以提供 100%的正常运行时间，即使是大规模分布式系统也很难接近。这意味着最终*任何*数据库都会因其主机服务器的故障而崩溃。在良好情况下，重新启动可能足够恢复。在糟糕情况下，部分或全部数据可能会丢失。

恢复和恢复备份绝非瞬间操作。在复制环境中，可以执行称为*切换*的特殊操作，将副本置于失败的数据库位置。在许多情况下，切换能节省大量时间，并且让在失败系统上的工作不至于太过仓促。

想象一下有两台运行 MySQL 的相同服务器的设置。一台是专用主服务器，接收所有连接并处理所有查询。另一台是副本。有一种机制可以将连接重定向到副本，进行切换将导致 5 分钟的停机时间。

有一天，主服务器的硬盘坏了。由于这是一台简单的服务器，这一问题导致了系统崩溃和停机时间。监控系统捕捉到了问题，数据库管理员立即意识到，为了在该服务器上恢复数据库，他们需要安装新硬盘，然后恢复最近的备份。整个操作将耗费几个小时。

在这种情况下切换到副本是个好主意，因为它节省了大量宝贵的运行时间。

## 部署错误

软件缺陷是生活中必须接受的事实。系统越复杂，逻辑错误发生的可能性就越高。虽然我们都在努力限制和减少缺陷，但必须理解它们会发生并相应地进行规划。

想象一下，一个包含数据库迁移脚本的新应用版本发布了。尽管新版本和脚本在下游环境中都经过了测试，但出现了一个 bug。迁移无法恢复地损坏了所有含有“特殊”非 ASCII 符号的客户姓氏。由于脚本顺利完成，损坏是悄无声息的，而问题直到一周后由一个愤怒的客户注意到，他的名字现在是错误的。

即使生产数据库有一个副本，它的数据和逻辑损坏也是一样的。在这种情况下切换到副本 *不能* 有所帮助，必须恢复迁移前的备份以获取正确姓氏列表。

###### 注意

延迟副本可以在这种情况下为您提供保护，但是延迟越长，操作这样的副本就越不切实际。您可以创建一个一周的延迟副本，但您可能需要一小时前的数据。通常，副本延迟以分钟和小时计算。

刚刚讨论的两种故障场景涵盖了两个不同的领域：物理和逻辑。复制对于在物理问题发生时提供保护很合适，但对于逻辑问题几乎没有（或很少）保护。复制是一个有用的工具，但它不能替代备份。

# mysqldump 程序

在线备份数据库的可能最简单方法是将其内容转储为 SQL 语句。这是至关重要的逻辑备份类型。*转储*在计算中通常意味着输出某个系统或其部分的内容，结果是一个*转储*。在数据库世界中，转储通常是逻辑备份的一种，转储是获得这种备份的操作。将备份恢复到数据库涉及应用这些语句。您可以通过使用例如 `SHOW CREATE TABLE` 和一些 `CONCAT` 操作从表中的数据行中获取 `INSERT` 语句来手动生成转储，如下所示：

```
mysql> `SHOW` `CREATE` `TABLE` `sakila``.``actor``\``G`
```

```
*************************** 1\. row ***************************
       Table: actor
Create Table: CREATE TABLE `actor` (
  `actor_id` smallint unsigned NOT NULL AUTO_INCREMENT,
  `first_name` varchar(45) NOT NULL,
  `last_name` varchar(45) NOT NULL,
  `last_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`actor_id`),
  KEY `idx_actor_last_name` (`last_name`)
) ENGINE=InnoDB AUTO_INCREMENT=201 DEFAULT CHARSET=utf8mb4
        COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)
```

```
mysql> `SELECT` `CONCAT``(``"INSERT INTO actor VALUES"``,`
    -> `"("``,``actor_id``,``",'"``,``first_name``,``"','"``,`
    -> `last_name``,``"','"``,``last_update``,``"');"``)`
    -> `AS` `insert_statement` `FROM` `actor` `LIMIT` `1``\``G`
```

```
*************************** 1\. row ***************************
insert_statement: INSERT INTO actor VALUES
(1,'PENELOPE','GUINESS','2006-02-15 04:34:33');
1 row in set (0.00 sec)
```

然而，这很快变得极其不切实际。此外，还有更多需要考虑的事情：*语句的顺序*，以便在恢复时，`INSERT` 不会在表创建之前运行，以及*所有权*和*一致性*。尽管手动生成逻辑备份有助于理解，但却是繁琐且容易出错的。幸运的是，MySQL 自带一个强大的逻辑备份工具，名为 `mysqldump`，它隐藏了大部分复杂性。

MySQL 所附带的 `mysqldump` 程序允许您从正在运行的数据库实例中生成转储。`mysqldump` 的输出是一系列 SQL 语句，稍后可以应用到相同或另一个 MySQL 实例中。`mysqldump` 是一个跨平台工具，在 MySQL 服务器可用的所有操作系统上都可以使用。由于生成的备份文件只是一堆文本，因此它也是跨平台的。

`mysqldump` 的命令行参数很多，因此在使用这个工具之前最好先查阅[MySQL 参考手册](https://oreil.ly/7T8dD)。不过，最基本的情况只需要一个参数：目标数据库名称。

###### 提示

建议您按照“登录路径配置文件”中的说明为`root`用户和密码设置一个`client`登录路径。然后，您就不需要为我们本章展示的任何命令指定账户并提供其凭据了。

在下面的例子中，`mysqldump` 被调用而没有输出重定向，这个工具将把所有语句打印到标准输出：

```
$ mysqldump sakila
```

```
...
--
-- Table structure for table `actor`
--

DROP TABLE IF EXISTS `actor`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `actor` (
  `actor_id` smallint unsigned NOT NULL AUTO_INCREMENT,
  `first_name` varchar(45) NOT NULL,
  `last_name` varchar(45) NOT NULL,
  `last_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`actor_id`),
  KEY `idx_actor_last_name` (`last_name`)
) ENGINE=InnoDB AUTO_INCREMENT=201 DEFAULT CHARSET=utf8mb4
        COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `actor`
--

LOCK TABLES `actor` WRITE;
/*!40000 ALTER TABLE `actor` DISABLE KEYS */;
INSERT INTO `actor` VALUES
(1,'PENELOPE','GUINESS','2006-02-15 01:34:33'),
(2,'NICK','WAHLBERG','2006-02-15 01:34:33'),
...
(200,'THORA','TEMPLE','2006-02-15 01:34:33');
/*!40000 ALTER TABLE `actor` ENABLE KEYS */;
UNLOCK TABLES;
...
```

###### 注意

`mysqldump` 的输出非常冗长，不适合在书中打印。在这里和其他地方，输出被截断，只包括我们感兴趣的行。

您可能会注意到，此输出比您预期的更加微妙。例如，有一个 `DROP TABLE IF EXISTS` 语句，它在目标上已存在表时可以防止以下 `CREATE TABLE` 命令出错。 `LOCK` 和 `UNLOCK TABLES` 语句将提高数据插入性能，等等。

谈到模式结构，可以生成不含数据的备份。例如，为了开发环境，创建数据库的逻辑克隆就很有用。这种灵活性是逻辑备份和 `mysqldump` 的关键特性之一：

```
$ mysqldump --no-data sakila
```

```
...
--
-- Table structure for table `actor`
--

DROP TABLE IF EXISTS `actor`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!50503 SET character_set_client = utf8mb4 */;
CREATE TABLE `actor` (
  `actor_id` smallint unsigned NOT NULL AUTO_INCREMENT,
  `first_name` varchar(45) NOT NULL,
  `last_name` varchar(45) NOT NULL,
  `last_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`actor_id`),
  KEY `idx_actor_last_name` (`last_name`)
) ENGINE=InnoDB AUTO_INCREMENT=201 DEFAULT CHARSET=utf8mb4
        COLLATE=utf8mb4_0900_ai_ci;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Temporary view structure for view `actor_info`
--
...
```

也可以创建数据库中单个表的备份。在下一个例子中，`sakila` 是数据库，`category` 是目标表：

```
$ mysqldump sakila category
```

将灵活性提升到一个新的层次，可以通过指定 `--where` 或 `-w` 参数从表中仅导出少量行。正如其名，语法与 SQL 语句中的 `WHERE` 子句相同：

```
$ mysqldump sakila actor --where="actor_id > 195"
```

```
...
--
-- Table structure for table `actor`
--

DROP TABLE IF EXISTS `actor`;
CREATE TABLE `actor` (
...

--
-- Dumping data for table `actor`
--
-- WHERE:  actor_id > 195

LOCK TABLES `actor` WRITE;
/*!40000 ALTER TABLE `actor` DISABLE KEYS */;
INSERT INTO `actor` VALUES
(196,'BELA','WALKEN','2006-02-15 09:34:33'),
(197,'REESE','WEST','2006-02-15 09:34:33'),
(198,'MARY','KEITEL','2006-02-15 09:34:33'),
(199,'JULIA','FAWCETT','2006-02-15 09:34:33'),
(200,'THORA','TEMPLE','2006-02-15 09:34:33');
/*!40000 ALTER TABLE `actor` ENABLE KEYS */;
UNLOCK TABLES;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;
```

到目前为止，示例只涵盖了对单个数据库 `sakila` 的全部或部分数据的导出。有时需要输出每个数据库、每个对象，甚至每个用户。 `mysqldump` 能够胜任。以下命令将有效创建数据库实例的完整逻辑备份：

```
$ mysqldump --all-databases --triggers --routines --events > dump.sql
```

触发器默认被导出，因此此选项不会出现在未来的命令输出中。如果您不希望导出触发器，可以使用 `--no-triggers`。

这个命令存在一些问题。首先，尽管我们已将命令的输出重定向到一个文件，但生成的文件可能非常大。幸运的是，其内容很可能非常适合压缩，尽管这取决于实际数据。无论如何，压缩输出是个好主意：

```
$ mysqldump --all-databases \
--routines --events | gzip > dump.sql.gz
```

在 Windows 上，通过管道压缩输出比较困难，因此只需压缩通过运行上述命令生成的*dump.sql*文件。在像我们这里使用的小型虚拟机这样的 CPU 繁忙系统上，压缩可能会显著增加备份过程的时间。这是一个需要根据您的特定系统权衡的折衷：

```
$ time mysqldump --all-databases \
--routines --events > dump.sql
```

```
real    0m24.608s
user    0m15.201s
sys     0m2.691s
```

```
$ time mysqldump --all-databases \
--routines --events | gzip > dump.sql.gz
```

```
real    2m2.769s
user    2m4.400s
sys     0m3.115s
```

```
$ ls -lh dump.sql
```

```
-rw... 2.0G ... dump.sql
-rw... 794M ... dump.sql.gz
```

第二个问题是为了确保一致性，在转储数据库时会对表加锁，阻止写入（可以继续写入其他数据库）。这既影响性能又影响备份一致性。结果的转储只在数据库内部一致，而不是整个实例。这种默认行为是必要的，因为 MySQL 使用的一些存储引擎是非事务性的（主要是较旧的 MyISAM）。另一方面，默认的 InnoDB 存储引擎具有多版本并发控制（MVCC）模型，允许维护*读取快照*。我们在“备用存储引擎”中更深入地介绍了不同的存储引擎，以及在第六章中的锁定。

使用 InnoDB 的事务功能可以通过向`mysqldump`传递`--single-transaction`命令行参数来实现。然而，这会移除表锁定，因此使得非事务性表在转储期间容易出现不一致性。例如，如果您的系统同时使用 InnoDB 和 MyISAM 表，可能需要分别转储它们，如果不需要中断写入和保持一致性的话。

###### 注意

虽然`--single-transaction`确保`mysqldump`运行时可以继续写入，但仍然有一些注意事项：并发运行的 DDL 语句可能会导致不一致性，并且长时间运行的事务（例如由`mysqldump`启动的事务）可能会对整个实例的性能产生[负面影响](https://oreil.ly/pH2pJ)。

对主要使用 InnoDB 表的系统进行转储的基本命令，可以保证对并发写入的影响很小，如下所示：

```
$ mysqldump --single-transaction --all-databases \
--routines --events | gzip > dump.sql.gz
```

在实际应用中，您可能会有更多参数用来指定连接选项。您还可以围绕`mysqldump`语句编写脚本，以捕获任何问题并在出现问题时通知您。

使用`--all-databases`进行转储会包括内部 MySQL 数据库，例如`mysql`、`sys`和`information_schema`。这些信息不一定在恢复数据时总是需要，并且可能在恢复到已经有一些数据库的实例时出现问题。但是，您应该记住，MySQL 用户详细信息只会作为`mysql`数据库的一部分进行转储。

通常情况下，使用`mysqldump`和它生成的逻辑备份可以实现以下功能：

+   在不同环境之间轻松传输数据。

+   通过人类和程序在原地编辑数据。例如，您可以从转储中删除个人或不必要的数据。

+   发现某些数据文件的损坏。

+   在不同主要数据库版本、不同平台甚至不同数据库之间的数据传输。

## 使用 `mysqldump` 引导复制

`mysqldump` 程序可以用来创建一个空的或带有数据的副本实例。为了方便起见，提供了多个命令行参数。例如，当指定 `--master-data` 时，生成的输出将包含一个 SQL 语句 (`CHANGE MASTER TO`)，它将在目标实例上正确设置复制坐标。稍后使用这些坐标在目标实例上启动复制时，数据将不会有任何间隙。在基于 GTID 的复制拓扑中，可以使用 `--set-gtid-purged` 来实现相同的结果。然而，即使没有任何额外的命令行参数，`mysqldump` 也会检测到 `gtid_mode=ON` 并包含必要的输出。

在 “使用 mysqldump 创建副本” 中提供了使用 `mysqldump` 设置复制的示例。

# 从 SQL 转储文件加载数据

在执行备份时，始终记住，您是为了能够以后还原数据。对于逻辑备份，恢复过程就像将备份文件的内容通过管道传输到 `mysql` CLI 一样简单。正如前面讨论的那样，MySQL 必须处于逻辑备份还原状态，这既有好处也有坏处：

+   在系统的其他部分正常运行时，您可以还原单个对象，这是一个优点。

+   还原过程效率低下，并且会像任何常规客户端一样加载系统，如果决定插入大量数据。这是一个缺点。

让我们来看一个简单的示例，使用单个数据库备份和还原。正如我们之前所见，`mysqldump` 将在转储中包含必要的 `DROP` 语句，因此即使对象存在，它们也将被成功还原：

```
$ mysqldump sakila > /tmp/sakila.sql
$ mysql -e "CREATE DATABASE sakila_mod"
$ mysql sakila_mod < /tmp/sakila.sql
$ mysql sakila_mod -e "SHOW TABLES"
```

```
+----------------------------+
| Tables_in_sakila_mod       |
+----------------------------+
| actor                      |
| actor_info                 |
| ...                        |
| store                      |
+----------------------------+
```

像 `mysqldump` 或 `mysqlpump`（在下一节讨论）生成的 SQL 转储一样，是一个资源密集型的操作。默认情况下，它也是一个串行过程，可能需要大量时间。您可以使用一些技巧来加快这个过程，但请记住，错误可能导致丢失或不正确还原数据。选项包括：

+   模式/数据库的并行还原

+   在模式内并行还原对象

如果使用 `mysqldump` 在每个数据库的基础上进行转储，则很容易完成第一个。如果不需要跨数据库的一致性（不会得到保证），备份过程也可以并行化。以下示例使用 `&` 修饰符，指示 shell 在后台执行前面的命令：

```
$ mysqldump sakila > /tmp/sakila.sql &
$ mysqldump nasa > /tmp/nasa.sql &
```

结果转储是独立的。除非备份 `mysql` 数据库，否则 `mysqldump` 不会处理用户和授权，因此您需要注意。还原同样简单：

```
$ mysql sakila < /tmp/sakila.sql &
$ mysql nasa < /tmp/nasa.sql &
```

在 Windows 上，还可以使用 PowerShell 命令`Start-Process`或在后续版本中相同的`&`将命令执行发送到后台。

第二个选项更为复杂。您可以根据表格基础转储（例如，`mysqldump sakila artists > sakila.artists.sql`），这会导致简单的还原，或者您需要继续编辑转储文件以将其拆分为多个文件。在极端情况下，甚至可以在表级别并行化数据插入，尽管这可能不实用。

尽管这是可行的，但最好使用专门用于此任务的工具。

# mysqlpump

`mysqlpump`是 MySQL 版本 5.7 及更高版本捆绑的实用程序程序，主要在性能和可用性等多个领域改进了`mysqldump`。其主要区别如下：

+   并行转储能力

+   内置转储压缩

+   通过延迟创建二级索引来改进恢复性能

+   更轻松地控制转储对象

+   转储用户帐户的修改行为

使用该程序与使用`mysqldump`非常相似。最主要的即时区别在于，当没有传递参数时，`mysqlpump`将默认转储所有数据库（不包括`INFORMATION_SCHEMA`、`performance_schema`、`ndbinfo`和`sys`模式）。其他显著的区别是有进度指示器，并且`mysqlpump`默认使用两个线程进行并行转储：

```
$ mysqlpump > pump.out
```

```
Dump progress: 1/2 tables, 0/530419 rows
Dump progress: 80/184 tables, 2574413/646260694 rows
...
Dump progress: 183/184 tables, 16297773/646260694 rows
Dump completed in 10680
```

`mysqlpump`中的并行概念有些复杂。您可以在不同数据库之间以及给定数据库内的不同对象之间使用并发性。默认情况下，如果没有指定其他并行选项，`mysqlpump`将使用单个队列和两个并行线程来处理所有数据库和用户定义（如果被请求）。您可以使用`--default-parallelism`参数来控制默认队列的并行级别。为了进一步微调并发性，您可以设置多个并行队列来处理不同的数据库。在选择所需并发级别时要小心，因为备份运行可能会占用大部分数据库资源。

使用`mysqlpump`时与`mysqldump`的一个重要区别在于它如何处理用户帐户。`mysqldump`通过转储`mysql.user`和其他相关表来管理用户。如果在转储中不包括`mysql`数据库，则不会保留任何用户信息。`mysqlpump`通过引入命令行参数`--users`和`--include-users`对此进行了改进。第一个参数告诉实用程序为所有用户添加与转储相关的命令，第二个参数接受用户名列表。这在旧方式的基础上有了很大改进。

让我们结合所有新功能来生成非系统数据库和用户定义的压缩转储，并在过程中使用并发性：

```
$ mysqlpump --compress-output=zlib --include-users=bob,kate \
--include-databases=sakila,nasa,employees \
--parallel-schemas=2:employees \
--parallel-schemas=sakila,nasa > pump.out
```

```
Dump progress: 1/2 tables, 0/331579 rows
Dump progress: 19/23 tables, 357923/3959313 rows
...
Dump progress: 22/23 tables, 3755358/3959313 rows
Dump completed in 10098
```

###### 注意

`mysqlpump`输出可以使用 ZLIB 或 LZ4 算法进行压缩。当操作系统级命令`lz`和`openssl zlib`不可用时，您可以使用包含在 MySQL 分发中的`lz4_decompress`和`zlib_decompress`实用程序。

由于其中的数据是交错的，从`mysqlpump`运行的转储不适合并行恢复。例如，以下是`mysqlpump`执行的结果，显示了在不同数据库的表中插入数据时创建表：

```
...,(294975,"1955-07-31","Lucian","Rosis","M","1986-12-08");
CREATE TABLE `sakila`.`store` (
`store_id` tinyint unsigned NOT NULL AUTO_INCREMENT,
`manager_staff_id` tinyint unsigned NOT NULL,
`address_id` smallint unsigned NOT NULL,
`last_update` timestamp NOT NULL DEFAULT
CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
PRIMARY KEY (`store_id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT
CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
;
INSERT INTO `employees`.`employees` VALUES
(294976,"1961-03-19","Rayond","Khalid","F","1989-11-03"),...
```

`mysqlpump`是`mysqldump`的改进版本，增加了重要的并发性、压缩和对象控制功能。然而，该工具不允许对转储进行并行恢复，实际上使其不可能。对于恢复性能的唯一改进是在主要加载完成后添加次要索引。

# mydumper 和 myloader

`mydumper`和`myloader`都是开源项目[`mydumper`](https://oreil.ly/oOo8F)的一部分。这套工具试图使逻辑备份更高效、更易管理和更友好。在这里我们不会深入讨论，因为在涵盖每种可能的 MySQL 备份变体时书中的空间很容易不够。

这些程序可以通过从项目的 GitHub 页面获取最新版本或编译源代码来安装。在撰写本文时，最新版本略落后于主分支。“设置 mydumper 和 myloader 实用程序”提供了逐步安装说明。

我们之前展示了`mysqlpump`如何改善转储性能，但提到其交织的输出对恢复没有帮助。`mydumper`结合了并行转储方法，并为使用`myloader`进行并行恢复做好了准备。这通过将每个表转储到单独的文件中实现。

`mydumper`的默认调用非常简单。该工具尝试连接到数据库，启动一致性转储，并在当前目录下创建一个目录用于导出文件。请注意，默认情况下，每个表都有自己的文件。转储操作的默认并行度设置为`4`，意味着将同时读取四个单独的表。在此目录上调用`myloader`将能够并行恢复表格。

要创建并查看转储，请执行以下命令：

```
$ mydumper -u root -a
```

```
Enter the MySQL password:
```

```
$ ls -ld export
```

```
drwx... export-20210613-204512
```

```
$ ls -la export-20210613-204512
```

```
...
-rw... sakila.actor.sql
-rw... sakila.address-schema.sql
-rw... sakila.address.sql
-rw... sakila.category-schema.sql
-rw... sakila.category.sql
-rw... sakila.city-schema.sql
-rw... sakila.city.sql
...
```

除了并行转储和恢复功能外，`mydumper`还具有一些更高级的功能：

+   轻量级备份锁支持。Percona Server for MySQL 实现了一些额外的轻量级锁定，这些锁定被 Percona XtraBackup 使用。`mydumper`在可能的情况下默认使用这些锁定。这些锁定不会阻塞对 InnoDB 表的并发读写，但会阻塞任何 DDL 语句，否则可能使备份无效。

+   使用保存点。`mydumper`使用事务保存点技巧来最小化元数据锁定。

+   元数据锁定持续时间限制。为了解决我们在 “元数据锁” 中描述的长时间元数据锁定问题，`mydumper` 提供两个选项：快速失败或终止长时间运行的查询，从而使 `mydumper` 能够成功。

`mydumper` 和 `myloader` 是先进的工具，将逻辑备份能力发挥到极致。然而，作为一个社区项目的一部分，它们缺乏其他工具提供的文档和优化。另一个主要缺点是缺乏任何支持或保证。尽管如此，它们仍然可以成为数据库运营者工具箱中有用的补充。

# 冷备份和文件系统快照

物理备份的基石，*冷备份*实际上只是数据目录及其它必要文件的一份拷贝，在数据库实例停机时完成。这种技术并不经常使用，但在需要快速创建一致备份时，它可以派上用场。随着数据库常规接近多 TB 的大小范围，仅仅复制文件可能需要很长时间。然而，冷备份仍然有其优点：

+   非常快速（可以说是快照以外最快的备份方法）

+   简单直接

+   易于使用，难以做错

现代存储系统和一些文件系统具备即时快照功能。它们允许您通过利用内部机制创建任意大小的卷的几乎瞬时副本。不同支持快照的系统的特性差异很大，使我们无法覆盖所有系统。然而，我们仍然可以从数据库的角度谈谈它们的一些特点。

大多数快照将是*写时复制*（COW）并在某个时间点上内部一致。然而，我们已经知道数据库文件在磁盘上并不一致，特别是对于像 InnoDB 这样的事务性存储引擎。这使得正确获取快照备份有些困难。有两种选择：

冷备份快照

当数据库关闭时，其数据文件可能仍然不完全一致。但是如果您对所有数据库文件（包括 InnoDB 重做日志等）进行快照，它们将允许数据库启动。这是理所当然的，否则数据库每次重新启动都会丢失数据。不要忘记您可能在许多卷中拥有数据库文件分散的情况。您将需要所有这些文件。这种方法适用于所有存储引擎。

热备份快照

在运行中的数据库中正确进行快照是一个比数据库停机时更大的挑战。如果您的数据库文件位于多个卷上，则无法保证即使同时启动的快照也能一致到达相同的时间点，这可能会导致灾难性的结果。此外，像 MyISAM 这样的非事务性存储引擎在数据库运行时也不能保证磁盘上文件的一致性。实际上，对于 InnoDB 也是如此，但 InnoDB 的重做日志始终是一致的（除非禁用了保护措施），而 MyISAM 则缺乏这种功能。

建议的热备份快照方法因此是利用一些程度的锁定。由于快照过程通常很快，因此 resulting downtime shouldn’t be significant. Here’s the process:

1.  创建一个新的会话，并使用`FLUSH TABLES WITH READ LOCK`命令锁定所有表格。此会话不能关闭，否则锁定将被释放。

1.  可选地，通过运行`SHOW MASTER STATUS`命令记录当前的 binlog 位置。

1.  根据存储系统的手册为 MySQL 数据库文件所在的所有卷创建快照。

1.  在最初打开的会话中使用`UNLOCK TABLES`命令解锁表格。

如果不是所有当前存储系统和能够执行快照的文件系统，此一般方法都应该是合适的。请注意，它们在实际过程和要求上都有微妙的差异。一些云供应商要求您还需在文件系统上执行 `fsfreeze`。

在将其实施到生产中并信任它们处理数据之前，请始终彻底测试您的备份。您只能信任您已经测试并且使用起来感到舒适的解决方案。盲目复制任意备份策略建议并不是一个好主意。

# Percona XtraBackup

实施所谓的*热备份*是物理备份的逻辑下一步，也就是在数据库运行时复制数据库文件。我们已经提到 MyISAM 表可以复制，但对于 InnoDB 和其他像 MyRocks 这样的事务性存储引擎则不适用。问题在于你不能仅仅复制文件，因为数据库正在不断变化。例如，即使现在没有写操作命中数据库，InnoDB 可能正在后台刷新一些脏页。你可以试试在运行中的系统下复制数据库目录，然后尝试恢复该目录并启动一个 MySQL 服务器。成功的可能性很小。即使有时候会成功，我们强烈建议不要冒数据库备份的风险。

能够执行热备份的能力内置于三种主要的 MySQL 备份工具中：[Percona XtraBackup](https://oreil.ly/yMK8t)、[MySQL Enterprise Backup](https://oreil.ly/rkSrr)和[`mariabackup`](https://oreil.ly/DJvoa)。我们将简要讨论它们，但主要集中在 XtraBackup 实用程序上。重要的是要理解，所有工具都共享特性，因此掌握其中一个工具的使用方法将有助于您使用其他工具。

Percona XtraBackup 是由 Percona 和更广泛的 MySQL 社区维护的免费开源工具。它能够对带有 InnoDB、MyISAM 和 MyRocks 表的 MySQL 实例进行在线备份。该程序仅在 Linux 上可用。请注意，最新版本的 MariaDB 无法使用 XtraBackup：只支持 MySQL 和 Percona Server。对于 MariaDB，请使用我们在“mariabackup”中介绍的实用程序。

这是 XtraBackup 操作的概述：

1.  记录当前日志序列号（LSN），这是操作的内部版本号

1.  开始累积 InnoDB 的*重做数据*（InnoDB 为崩溃恢复存储的数据类型）

1.  以尽可能少干扰的方式锁定表

1.  复制 InnoDB 表

1.  完全锁定非事务性表

1.  复制 MyISAM 表

1.  解锁所有表

1.  处理 MyRocks 表（如果存在）

1.  将累积的重做数据放置在复制的数据库文件旁边

XtraBackup 及热备份的主要理念是将逻辑备份的无停机特性与冷备份的性能和相对缺乏性能影响结合起来。XtraBackup 不能保证无服务中断，但与常规冷备份相比，它是一大进步。缺乏性能影响意味着 XtraBackup 只会使用必要的 CPU 和 I/O 来复制数据库文件。另一方面，逻辑备份必须通过数据库的所有内部处理每一行数据，因此本质上速度较慢。

###### 注意

XtraBackup 需要对数据库文件进行物理访问，无法远程运行。这使其不适合于管理数据库（DBaaS）的异地备份。然而，一些云供应商允许您使用此工具生成的备份导入数据库。

XtraBackup 实用程序广泛地出现在各种 Linux 发行版的仓库中，因此可以通过包管理器轻松安装。或者，您也可以直接从[Percona 官网的 XtraBackup 下载页面](https://oreil.ly/XjN4C)下载软件包和二进制发行版。

###### 警告

要备份 MySQL 8.0，必须使用 XtraBackup 8.0。理想情况下，XtraBackup 和 MySQL 的次要版本也应匹配：XtraBackup 8.0.25 保证与 MySQL 8.0.25 兼容。对于 MySQL 5.7 及更早的版本，请使用 XtraBackup 2.4。

## 备份和恢复

与我们之前提到的其他工具不同，由于 XtraBackup 是一种物理备份工具，它不仅需要访问 MySQL 服务器，还需要读取数据库文件的权限。在大多数 MySQL 安装中，这通常意味着 `xtrabackup` 程序应由 `root` 用户运行，或者必须使用 `sudo`。在本节中，我们将使用 `root` 用户，并使用 “Login Path Configuration File” 中的步骤设置登录路径。

首先，我们需要运行基本的 `xtrabackup` 命令：

```
# xtrabackup --host=127.0.0.1 --target-dir=/tmp/backup --backup
```

```
...
Using server version 8.0.25
210613 22:23:06 Executing LOCK INSTANCE FOR BACKUP...
...
210613 22:23:07 [01] Copying ./sakila/film.ibd
    to /tmp/backup/sakila/film.ibd
210613 22:23:07 [01]        ...done
...
210613 22:23:10 [00] Writing /tmp/backup/xtrabackup_info
210613 22:23:10 [00]        ...done
xtrabackup: Transaction log of lsn (6438976119)
    to (6438976129) was copied.
210613 22:23:11 completed OK!
```

如果登录路径不起作用，你可以通过命令行参数 `--user` 和 `--password` 向 `xtrabackup` 传递 `root` 用户的凭据。通常，XtraBackup 可以通过读取默认选项文件来识别目标服务器的数据目录，但如果这不起作用或者你有多个 MySQL 安装，可能还需要指定 `--datadir` 选项。尽管 `xtrabackup` 只在本地工作，但仍需要连接到本地运行的 MySQL 实例，因此需要 `--host`、`--port` 和 `--socket` 参数。根据你的特定设置，可能需要指定其中一些参数。

###### 提示

尽管我们在示例中使用 */tmp/backup* 作为备份的目标路径，你应避免在 */tmp* 下存储重要文件。这对于备份尤其重要。

那个 `xtrabackup --backup` 调用的结果是一堆数据库文件，实际上不一致到任何时间点，并且一部分重做数据是 InnoDB 无法应用的：

```
# ls -l /tmp/backup/
```

```
...
drwxr-x---.  2 root root       160 Jun 13 22:23 mysql
-rw-r-----.  1 root root  46137344 Jun 13 22:23 mysql.ibd
drwxr-x---.  2 root root        60 Jun 13 22:23 nasa
drwxr-x---.  2 root root       580 Jun 13 22:23 sakila
drwxr-x---.  2 root root       580 Jun 13 22:23 sakila_mod
drwxr-x---.  2 root root        80 Jun 13 22:23 sakila_new
drwxr-x---.  2 root root        60 Jun 13 22:23 sys
...
```

为了使备份准备好进行将来的恢复，还必须执行另一个阶段——准备阶段。这时不需要连接到 MySQL 服务器：

```
# xtrabackup --target-dir=/tmp/backup --prepare
```

```
...
xtrabackup: cd to /tmp/backup/
xtrabackup: This target seems to be not prepared yet.
...
Shutdown completed; log sequence number 6438976524
210613 22:32:23 completed OK!
```

最终的数据目录实际上已经完全可以使用了。你可以启动一个直接指向这个目录的 MySQL 实例，它将正常工作。这里一个非常常见的错误是尝试在 `mysql` 用户下启动 MySQL 服务器，而恢复和准备好的备份却由 `root` 或其他操作系统用户所有。确保在你的备份恢复过程中根据需要加入 `chown` 和 `chmod`。但是，`xtrabackup` 提供了一个有用的用户体验功能 `--copy-back`。`xtrabackup` 会保留原始数据库文件的布局位置，并在使用 `--copy-back` 时将所有文件恢复到其原始位置：

```
# xtrabackup --target-dir=/tmp/backup --copy-back
```

```
...
Original data directory /var/lib/mysql is not empty!
```

这不起作用，因为我们的原始 MySQL 服务器仍在运行，其数据目录不为空。XtraBackup 将拒绝恢复备份，除非目标数据目录为空。这应该能防止意外恢复备份。让我们关闭正在运行的 MySQL 服务器，删除或移动其数据目录，并恢复备份：

```
# systemctl stop mysqld
# mv /var/lib/mysql /var/lib/mysql_old
# xtrabackup --target-dir=/tmp/backup --copy-back
```

```
...
210613 22:39:01 [01] Copying ./sakila/actor.ibd
    to /var/lib/mysql/sakila/actor.ibd
210613 22:39:01 [01]        ...done
...
210613 22:39:01 completed OK!
```

之后，文件位于正确的位置，但所有者是 `root`：

```
# ls -l /var/lib/mysql/
```

```
drwxr-x---. 2 root root      4096 Jun 13 22:39 sakila
drwxr-x---. 2 root root      4096 Jun 13 22:38 sakila_mod
drwxr-x---. 2 root root      4096 Jun 13 22:39 sakila_new
```

你需要将文件的所有者更改回 `mysql`（或系统中使用的用户），并修复目录权限。完成后，你可以启动 MySQL 并验证数据：

```
# chown -R mysql:mysql /var/lib/mysql/
# chmod o+rx /var/lib/mysql/
# systemctl start mysqld
# mysql sakila -e "SHOW TABLES;"
```

```
+----------------------------+
| Tables_in_sakila           |
+----------------------------+
| actor                      |
...
| store                      |
+----------------------------+
```

###### 提示

最佳实践是在备份阶段同时执行备份和准备工作，从而最大程度地减少以后可能出现的意外情况。想象一下，在尝试恢复某些数据时，准备阶段失败！然而，请注意，我们稍后介绍的增量备份有特殊的处理程序，与此建议相矛盾。

## 高级功能

本节中，我们将讨论一些 XtraBackup 更高级的功能。使用这些功能并非使用该工具的必需，我们仅提供简要概述：

数据库文件验证

在执行备份时，XtraBackup 将验证正在处理的数据文件的所有页面的校验和。这是为了缓解物理备份的固有问题，即它们将包含源数据库中的任何损坏。我们建议在此检查中使用“测试和验证您的备份”中列出的其他步骤。

压缩

尽管复制物理文件比查询数据库要快得多，但备份过程可能会受到磁盘性能的限制。您无法减少读取的数据量，但可以利用压缩使备份本身变小，减少需要写入的数据量。这在备份目标是网络位置时尤为重要。此外，您将只使用更少的空间来存储备份。请注意，正如我们在“mysqldump 程序”中展示的那样，在 CPU 繁忙的系统上，压缩实际上可能会增加创建备份所需的时间。XtraBackup 使用`qpress`工具进行压缩。该工具包含在`percona-release`软件包中：

```
# xtrabackup --host=127.0.0.1 \
--target-dir=/tmp/backup_compressed/ \
--backup --compress
```

并行性

可以通过使用`--parallel`命令行参数并行运行备份和复制回归过程。

加密

除了能够处理加密数据库外，XtraBackup 还可以创建加密备份。

流式传输

XtraBackup 可以将生成的备份以`xbstream`格式流式传输，而不是创建一个充满备份文件的目录。这将产生更具可移植性的备份，并允许与`xbcloud`集成。例如，您可以通过 SSH 流式传输备份。

云上传

使用 XtraBackup 进行的备份可以通过`xbcloud`上传到任何兼容 S3 的存储设施。S3 是亚马逊的对象存储设施，是被许多公司广泛采用的 API。这个工具仅适用于通过`xbstream`流式传输的备份。

## 增量备份与 XtraBackup

如前所述，热备份是数据库中每个信息字节的副本。这是 XtraBackup 默认的工作方式。但在很多情况下，数据库的变化率是*不规则*的——新数据经常添加，而旧数据则几乎没有变化。例如，新的财务记录可能每天都在添加，账户会被修改，但在给定的一周内，只有少部分账户会发生变化。因此，改进热备份的下一个逻辑步骤是增加执行所谓的*增量备份*的能力，即仅备份已更改的数据。这将允许您通过减少空间需求来更频繁地执行备份。

要使增量备份正常工作，您首先需要对数据库进行完整备份，称为*基础备份*——否则无法进行增量备份。一旦基础备份准备就绪，您可以执行任意数量的增量备份，每个备份包含自上一个备份以来的更改（或在第一个增量备份的情况下，自基础备份以来的更改）。将其推到极致，您可以每分钟创建一个增量备份，实现所谓的*时间点恢复*（PITR），但这并不是非常实际的做法，很快您将会了解到有更好的方法来做到这一点。

这里是你可以使用的 XtraBackup 命令的示例，用于创建基础备份，然后创建增量备份。注意增量备份通过 `--incremental-basedir` 参数指向基础备份：

```
# xtrabackup --host=127.0.0.1 \
--target-dir=/tmp/base_backup --backup
# xtrabackup --host=127.0.0.1 --backup \
--incremental-basedir=/tmp/base_backup \
--target-dir=/tmp/inc_backup1
```

如果您检查备份大小，您会发现与基础备份相比，增量备份非常小：

```
# du -sh /tmp/base_backup
```

```
2.2G    /tmp/base_backup
6.0M    /tmp/inc_backup1
```

让我们创建另一个增量备份。在这种情况下，我们将前一个增量备份的目录作为基本目录传递：

```
# xtrabackup --host=127.0.0.1 --backup \
--incremental-basedir=/tmp/inc_backup1 \
--target-dir=/tmp/inc_backup2
```

```
210613 23:32:20 completed OK!
```

也许你在想是否可以将原始基础备份的目录指定为每个新增量备份的 `--incremental-basedir`。事实上，这样会产生一个完全有效的备份，这是增量备份的一种变体（或者反过来）。这种包含不仅自上一个增量备份以来的更改，而且自基础备份以来的增量备份通常被称为*累积*备份。针对任何先前备份的增量备份称为*差异*备份。累积增量备份通常会占用更多空间，但可以显著缩短在恢复备份时所需的准备时间。

重要的是，[增量备份的准备过程](https://oreil.ly/2c4LM)与常规备份的准备过程不同。让我们准备刚刚进行的备份，从基础备份开始：

```
# xtrabackup --prepare --apply-log-only \
--target-dir=/tmp/base_backup
```

`--apply-log-only` 参数告诉 `xtrabackup` 不要完成准备过程，因为我们仍然需要应用增量备份的更改。让我们来做这个：

```
# xtrabackup --prepare --apply-log-only \
--target-dir=/tmp/base_backup \
--incremental-dir=/tmp/inc_backup1
# xtrabackup --prepare --apply-log-only \
--target-dir=/tmp/base_backup \
--incremental-dir=/tmp/inc_backup2
```

所有命令执行完毕应报告`completed OK!`。一旦运行`--prepare --apply-log-only`操作，基础备份将会推进到增量备份的点，这将使得将 PITR 恢复到较早时间变得不可能。因此，在执行增量备份时立即准备并不是一个好主意。要完成准备过程，必须正常准备基础备份，其中包括从增量备份应用的更改：

```
# xtrabackup --prepare --target-dir=/tmp/base_backup
```

一旦基础备份“完全”准备好，尝试应用增量备份将失败，并显示以下消息：

```
xtrabackup: This target seems to be already prepared.
xtrabackup: error: applying incremental backup needs
    target prepared with --apply-log-only.
```

当数据库中的变更量相对较高时，增量备份效率低下。在最坏的情况下，即数据库中的每一行在完整备份和增量备份之间都发生了变化时，后者实际上只是一个完整备份，存储了 100%的数据。增量备份在大部分数据追加且旧数据变更量相对较低时效率最高。对此没有规则，但如果在基础备份和增量备份之间的数据变更量为 50%，请考虑不使用增量备份。

# 其他物理备份工具

XtraBackup 并不是唯一能够执行热 MySQL 物理备份的工具。我们选择使用这个特定工具来解释概念是基于我们的经验。然而，这并不意味着其他工具在任何方面都更差。它们可能更适合您的需求。然而，我们的空间有限，备份主题非常广泛。我们可以撰写一本*备份 MySQL*的相当大的书！

话虽如此，为了让您了解一些其他选项，让我们来看看另外两种现成的物理备份工具。

## MySQL 企业备份

MEB（MySQL Enterprise Backup）被简称为 MEB，这是 Oracle 的 MySQL 企业版的一部分。它是一个闭源专有工具，功能类似于 XtraBackup。您可以在[MYSQL 网站](https://oreil.ly/nj7xI)上找到详细的文档。目前这两个工具的功能基本相同，所以几乎所有适用于 XtraBackup 的内容同样适用于 MEB。

MEB 的突出特点是它真正是一个跨平台的解决方案。XtraBackup 仅适用于 Linux，而 MEB 还适用于 Solaris、Windows、macOS 和 FreeBSD。MEB 不支持除了 Oracle 标准版之外的 MySQL 版本。

MEB 具有的一些额外功能在 XtraBackup 中不可用，包括以下内容：

+   备份进度报告

+   离线备份

+   通过 Oracle Secure Backups 进行磁带备份

+   二进制和中继日志备份

+   恢复时的表重命名

## mariabackup

`mariabackup`是 MariaDB 用于备份 MySQL 数据库的工具。最初从 XtraBackup 分支而来，这是一个在 Linux 和 Windows 上都可用的免费开源工具。`mariabackup`的显著特性是其与 MariaDB 分支 MySQL 的无缝协作，后者在使用方式和属性上继续与主流 MySQL 和 Percona Server 显著分歧。由于这是 XtraBackup 的直接分支，你会发现这些工具在使用方式和性能上有很多相似之处。一些 XtraBackup 的新功能，如备份加密和次要索引省略，在`mariabackup`中并不存在。然而，目前使用 XtraBackup 来备份 MariaDB 是不可能的。

# 时点恢复

现在你已经熟悉了热备份的概念，你几乎拥有完成备份工具包所需的一切。到目前为止，我们讨论的所有备份类型都有一个共同点——缺陷。它们只允许在拍摄时恢复。如果你有两个备份，一个是周一 23:00 拍摄的，另一个是周二 23:00 拍摄的，你就无法恢复到周二下午 5:00。

记得在本章开始时提到的基础设施故障示例吗？现在，让我们把情况恶化，假设数据丢失了，所有驱动器都失效了，而且没有复制。事件发生在周三晚上 21:00。没有 PITR，并且每天在 23:00 进行备份，这意味着你已经永远丢失了整整一天的数据。可以说，使用 XtraBackup 进行增量备份可以在一定程度上减少这个问题，但它们仍然存在一定的数据丢失空间，而且很少有机会经常运行它们。

MySQL 维护一个称为*二进制日志*的事务日志。通过将我们迄今讨论的任何备份方法与二进制日志结合起来，我们可以恢复到任意时间点的事务。非常重要的是要理解，为了使此功能正常工作，你需要同时具备备份和二进制日志。此外，你不能回溯时间，因此无法恢复数据到最老的基础备份或转储创建之前的时间点。

二进制日志包含事务时间戳和它们的标识符。你可以依靠其中任何一个进行恢复，并且可以告诉 MySQL 恢复到某个时间戳。当你想恢复到最新时间点时，这不是问题，但在尝试执行修复逻辑不一致性时（比如在“部署错误”中描述的情况），这可能非常重要和有帮助。然而，在大多数情况下，你需要识别一个特定的问题事务，我们将向你展示如何做到这一点。

MySQL 的一个有趣特点是，它允许逻辑备份进行 PITR。“从 SQL 转储文件加载数据” 讨论了使用 `mysqldump` 存储副本提供的 binlog 位置。相同的 binlog 位置可以用作 PITR 的起点。MySQL 中的每种备份类型都适用于 PITR，与其他数据库不同。为了促进这一特性，请确保在进行备份时注意 binlog 位置。一些备份工具会自动处理这个问题。如果使用的工具没有这样做，您可以运行 `SHOW MASTER STATUS` 来获取这些数据。

## 二进制日志的技术背景

MySQL 与许多其他主流关系型数据库不同，它支持多个存储引擎，如 “替代存储引擎” 中所讨论的。不仅如此，它还支持单个数据库内的表使用多个存储引擎。因此，MySQL 中的某些概念与其他系统中的不同。

MySQL 中的二进制日志本质上是事务日志。启用二进制日志后，每个事务（不包括只读事务）都将反映在二进制日志中。有三种方法可以将事务写入二进制日志：

*语句*

在这种模式下，语句按其编写方式记录到二进制日志中，这可能在复制场景中导致非确定性执行。

*行*

在这种模式下，语句被拆分为最小的 DML 操作，每个操作修改一个特定的行。虽然它保证了确定性执行，但这种模式最为冗长，导致的文件最大，因此产生了最大的 I/O 开销。

*混合*

在这种模式下，“安全”语句按原样记录，而其他语句则被拆分。

通常，在数据库管理系统中，事务日志用于崩溃恢复、复制和 PITR。但是，由于 MySQL 支持多个存储引擎，其二进制日志不能用于崩溃恢复。相反，每个引擎都维护其自己的崩溃恢复机制。例如，MyISAM 不是崩溃安全的，而 InnoDB 则有其自己的重做日志。MySQL 中的每个事务都是分布式事务，具有两阶段提交，以适应这种多引擎的特性。如果引擎是事务性的，每个提交的事务都保证会反映在存储引擎的重做日志中，以及 MySQL 自己的事务日志（即二进制日志）中。

###### 注意

要实现 PITR，必须在 MySQL 实例中启用二进制日志。您还应该默认将 `sync_binlog=1`，以确保每次写操作的持久性。请参考[MySQL 文档](https://oreil.ly/ygjVz) 以了解禁用 binlog 同步的权衡考虑。

我们将在 第十三章 中更详细地讨论二进制日志的工作原理。

## 保留二进制日志

要允许 PITR，必须保留从最早备份的 binlog 位置开始的二进制日志。有几种方法可以做到这一点：

+   使用像`rsync`这样的现成工具“手动”复制或同步二进制日志文件。请记住，MySQL 将继续写入当前的二进制日志文件。如果您复制文件而不是持续同步它们，则不要复制当前的二进制日志文件。通过持续同步文件来解决这个问题，一旦它变为非当前文件，就会覆盖部分文件。

+   使用`mysqlbinlog`复制单个文件或连续流式传输 binlog。有关说明，请参阅[文档](https://oreil.ly/GAsjw)。

+   使用 MySQL Enterprise Backup，它具有内置的 binlog 复制功能。请注意，这不是连续复制，而是依赖增量备份来进行 binlog 复制。这允许在两个备份之间进行 PITR。

+   允许 MySQL 服务器将所有必需的二进制日志存储在其数据目录中，通过为`binlog_expire_logs_seconds`或`expire_logs_days`变量设置一个较高的值。理想情况下，不应仅使用此选项，但可以与其他任何选项一起使用。如果发生数据目录的任何事故，例如文件系统损坏，存储在那里的二进制日志也可能会丢失。

## 确定 PITR 目标

您可以使用 PITR 技术实现两个目标：

1.  恢复到最新时间点。

1.  恢复到任意时间点。

如前所述，第一个用于将完全丢失的数据库恢复到最新可用状态。第二个用于获取数据之前的状态。一个案例示例是在“部署错误”中提到的情况。要恢复丢失或错误修改的数据，您可以恢复备份，然后将其恢复到执行部署之前的时间点。

确定问题发生的实际特定时间可能是一项挑战。通常情况下，您找到所需时间点的唯一方法是检查问题发生周围写入的二进制日志。例如，如果您怀疑表被删除，您可以查找表名称，然后查找在该表上执行的任何 DDL 语句，或者专门查找`DROP TABLE`语句。

让我们举个例子来说明。首先，我们需要实际删除一个表，因此我们将删除我们在“从逗号分隔文件加载数据”中创建的`facilities`表。但在此之前，我们将插入一条在原始备份中肯定缺失的记录：

```
mysql> `INSERT` `INTO` `facilities``(``center``)`
    -> `VALUES` `(``'this row was not here before'``)``;`
```

```
Query OK, 1 row affected (0.01 sec)
```

```
mysql> `DROP` `TABLE` `nasa``.``facilities``;`
```

```
Query OK, 0 rows affected (0.02 sec)
```

现在我们可以返回并恢复我们在本章中拍摄的备份之一，但那样我们会丢失在该点和`DROP`之间对数据库所做的任何更改。相反，我们将使用`mysqlbinlog`检查二进制日志的内容，并找到在运行`DROP`语句之前的恢复目标。要查找数据目录中可用的二进制日志列表，可以运行以下命令：

```
mysql> `SHOW` `BINARY` `LOGS``;`
```

```
+---------------+-----------+-----------+
| Log_name      | File_size | Encrypted |
+---------------+-----------+-----------+
| binlog.000291 |       156 | No        |
| binlog.000292 |       711 | No        |
+---------------+-----------+-----------+
2 rows in set (0.00 sec)
```

###### 警告

MySQL 不会永远将二进制日志保留在其数据目录中。它们在超过`binlog_expire_logs_seconds`或`expire_log_days`指定的持续时间后会自动删除，也可以通过运行`PURGE BINARY LOGS`手动删除。如果要确保二进制日志可用，应按照前一节所述将其保留在数据目录之外。

现在二进制日志列表可用，您可以尝试从最新的日志到最旧的日志进行搜索，或者只需将它们的全部内容一起转储。在我们的示例中，文件很小，因此我们可以使用后一种方法。无论哪种方法，都要使用`mysqlbinlog`命令：

```
# cd /var/lib/mysql
# mysqlbinlog binlog.000291 binlog.000292 \
-vvv --base64-output='decode-rows' > /tmp/mybinlog.sql

```

检查输出文件，我们可以找到有问题的语句：

```
...
#210613 23:32:19 server id 1  end_log_pos 200 ... Rotate to binlog.000291
...
# at 499
#210614  0:46:08 server id 1  end_log_pos 576 ...
# original_commit_timestamp=1623620769019544 (2021-06-14 00:46:09.019544 MSK)
# immediate_commit_timestamp=1623620769019544 (2021-06-14 00:46:09.019544 MSK)
/*!80001 SET @@session.original_commit_timestamp=1623620769019544*//*!*/;
/*!80014 SET @@session.original_server_version=80025*//*!*/;
/*!80014 SET @@session.immediate_server_version=80025*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 576
#210614  0:46:08 server id 1  end_log_pos 711 ... Xid = 25
use `nasa`/*!*/;
SET TIMESTAMP=1623620768/*!*/;
DROP TABLE `facilities` /* generated by server */
/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
...
```

我们应该在 2021-06-14 00:46:08 之前停止我们的恢复，或者在二进制日志位置 499。我们还需要从最新备份开始，包括*binlog.00291*之前的所有二进制日志。利用这些信息，我们可以继续进行备份恢复和恢复操作。

## 时点恢复示例：XtraBackup

单独使用 XtraBackup 不提供 PITR 功能。您需要添加额外的步骤来运行`mysqlbinlog`以重放恢复后数据库上的二进制日志内容：

1.  恢复备份。详细步骤请参见“备份和恢复”。

1.  启动 MySQL 服务器。如果您直接在源实例上恢复，则建议使用`--skip-networking`选项防止非本地客户端访问数据库。否则，某些客户端可能会在您完成恢复之前更改数据库。

1.  查找备份的二进制日志位置。它在备份目录中的*xtrabackup_binlog_info*文件中可用：

    ```
    # cat /tmp/base_backup/xtrabackup_binlog_info
    ```

    ```
    binlog.000291   156
    ```

1.  找到要恢复到的时间戳或二进制日志位置（例如，在执行`DROP TABLE`之前，如前所述）。

1.  回放二进制日志至所需点。例如，我们单独保存了二进制日志*binlog.000291*，但您应使用中心化的二进制日志存储作为二进制日志的源。您可以使用`mysqlbinlog`命令来执行此操作：

    ```
    # mysqlbinlog /opt/mysql/binlog.000291 \
    /opt/mysql/binlog.000292 --start-position=156 \
    --stop-datetime="2021-06-14 00:46:00" | mysql
    ```

1.  确保恢复成功且没有数据丢失。在我们的情况下，我们将查找在删除`facilities`表之前添加的记录：

    ```
    mysql> `SELECT` `center` `FROM` `facilities`
        -> `WHERE` `center` `LIKE` `'%before%'``;`
    ```

    ```
    +------------------------------+
    | center                       |
    +------------------------------+
    | this row was not here before |
    +------------------------------+
    1 row in set (0.00 sec)
    ```

## 时点恢复示例：mysqldump

使用`mysqldump`进行 PITR 所需的步骤类似于之前使用 XtraBackup 的步骤。我们只是为了完整性和让您看到，在 MySQL 中每种备份类型中 PITR 是类似的。以下是该过程：

1.  恢复 SQL 转储。同样，如果您的恢复目标服务器是备份源，则可能希望将其对客户端不可访问。

1.  在`mysqldump`备份文件中找到二进制日志位置：

    ```
    CHANGE MASTER TO MASTER_LOG_FILE='binlog.000010',
    MASTER_LOG_POS=191098797;
    ```

1.  找到要恢复到的时间戳或二进制日志位置（例如，在执行`DROP TABLE`之前，如前所述）。

1.  回放二进制日志至所需点：

    ```
    # mysqlbinlog /path/to/datadir/mysql-bin.000010 \
    /path/to/datadir/mysql-bin.000011 \
    --start-position=191098797 \
    --stop-datetime="20-05-25 13:00:00" | mysql
    ```

# 导出和导入 InnoDB 表空间

物理备份的一个主要缺点是通常需要同时复制数据库文件的大部分内容。虽然像 MyISAM 这样的存储引擎允许复制空闲表的数据文件，但不能保证 InnoDB 文件的一致性。不过，有些情况下，您只需要转移几个表，或者只需要一个表。到目前为止，我们看到的唯一选项是利用可能速度不可接受的逻辑备份。InnoDB 的表空间导出和导入功能，官方称为 *可传输表空间*，是同时获取两者优势的一种方式。我们也称此功能为 *导出/导入* 来简洁表述。

可传输表空间功能允许您结合在线物理备份的性能和逻辑备份的粒度。本质上，它提供了将 InnoDB 表的数据文件在线复制用于导入到相同或不同表的能力。这样的复制可以用作备份，也可以用作在不同 MySQL 安装之间传输数据的方式。

当逻辑转储可以达到相同目的时，为什么要使用导出/导入？导出/导入速度更快，并且除了锁定表之外，不会显著影响服务器。这在导入时尤其如此。对于大小在多千兆字节的表格，这是数据传输的少数可行选项之一。

## 技术背景

为了帮助您理解此功能如何工作，我们将简要回顾两个概念：物理备份和表空间。

正如我们所见，为了使物理备份一致，通常可以采取两种方法。第一种是关闭实例或以有保证的方式使数据只读。第二种是使数据文件一致到某一时刻，然后累积从那时刻到备份结束的所有更改。可传输表空间功能采用第一种方法，需要将表设置为短时间只读状态。

表空间是存储表数据及其索引的文件。默认情况下，InnoDB 使用 `innodb_file_per_table` 选项，强制为每个表创建专用的表空间文件。可以创建包含多个表数据的表空间，并且可以使用“旧”行为，即所有表驻留在单个 *ibdata* 表空间中。但只有在默认配置下，即为每个表创建专用表空间时才支持导出。分区表中的每个分区都有单独的表空间存在，这允许在不同表之间转移分区或从分区创建表的有趣能力。

## 导出表空间

现在这些概念已经涵盖，您知道导出需要做什么。但是，仍然缺少一个事项，那就是表定义。尽管大多数 InnoDB 表空间文件实际上包含其表的数据字典记录的冗余副本，但当前的可传输表空间实现要求在导入之前目标上存在表。

导出表空间的步骤如下：

1.  获取表定义。

1.  停止对表（或表）的所有写入，并使其一致。

1.  准备稍后导入表空间所需的额外文件：

    +   `.cfg`文件存储用于模式验证的元数据。

    +   只有在使用加密时才会生成`.cfp`文件，并包含目标服务器解密表空间所需的过渡密钥。

要获取表定义，您可以使用我们在本书中多次展示的`SHOW CREATE TABLE`命令。MySQL 通过单个命令`FLUSH TABLE ... FOR EXPORT`自动执行所有其他步骤。该命令锁定表并在目标表的常规`.ibd`文件附近生成所需的附加文件（如果使用加密，则可能是多个文件）。让我们从`sakila`数据库导出`actor`表：

```
mysql> `USE` `sakila`
mysql> `FLUSH` `TABLE` `actor` `FOR` `EXPORT``;`
```

```
Query OK, 0 rows affected (0.00 sec)
```

执行`FLUSH TABLE`的会话应保持打开状态，因为一旦会话终止，`actor`表将被释放。在 MySQL 数据目录中，常规的`actor.ibd`文件附近应出现一个新文件，即*actor.cfg*。让我们验证一下：

```
# ls -1 /var/lib/mysql/sakila/actor.
```

```
/var/lib/mysql/sakila/actor.cfg
/var/lib/mysql/sakila/actor.ibd
```

现在可以将这对`.ibd`和`.cfg`文件复制到其他地方并稍后使用。复制文件后，通常建议通过运行`UNLOCK TABLES`语句释放表上的锁定，或关闭调用了`FLUSH TABLE`的会话。完成所有操作后，您就有了一个准备好导入的表空间。

###### 注意

分区表具有多个`.ibd`文件，每个文件都有专用的`.cfg`文件。例如：

+   `learning_mysql_partitioned#p#p0.cfg`

+   `learning_mysql_partitioned#p#p0.ibd`

+   `learning_mysql_partitioned#p#p1.cfg`

+   `learning_mysql_partitioned#p#p1.ibd`

## 导入表空间

导入表空间非常简单，包括以下步骤：

1.  使用保留的定义创建表。无法以任何方式更改表的定义。

1.  丢弃表的表空间。

1.  复制`.ibd`和`.cfg`文件。

1.  修改表以导入表空间。

如果目标服务器上存在具有相同定义的表，则无需执行步骤 1。

让我们在同一服务器的另一个数据库中恢复`actor`表。表必须存在，因此我们将创建它：

```
mysql> `USE` `nasa`
mysql> `CREATE` `TABLE` `` `actor` `` `(`
    ->  `` `actor_id` `` `smallint` `unsigned` `NOT` `NULL` `AUTO_INCREMENT``,`
    ->  `` `first_name` `` `varchar``(``45``)` `NOT` `NULL``,`
    ->  `` `last_name` `` `varchar``(``45``)` `NOT` `NULL``,`
    ->  `` `last_update` `` `timestamp` `NOT` `NULL` `DEFAULT` `CURRENT_TIMESTAMP`
    ->    `ON` `UPDATE` `CURRENT_TIMESTAMP``,`
    ->  `PRIMARY` `KEY` `(``` `actor_id` ```)``,`
    ->  `KEY` `` `idx_actor_last_name` `` `(``` `last_name` ```)`
    -> `)` `ENGINE``=``InnoDB` `AUTO_INCREMENT``=``201` `DEFAULT`
    ->    `CHARSET``=``utf8mb4` `COLLATE``=``utf8mb4_0900_ai_ci``;`
```

```
Query OK, 0 rows affected (0.04 sec)
```

一旦创建`actor`表，MySQL 就会为其创建一个`.ibd`文件：

```
# ls /var/lib/mysql/nasa/
```

```
actor.ibd  facilities.ibd
```

这带我们进入下一步：丢弃此新表的表空间。通过运行特殊的`ALTER TABLE`完成：

```
mysql> `ALTER` `TABLE` `actor` `DISCARD` `TABLESPACE``;`
```

```
Query OK, 0 rows affected (0.02 sec)
```

现在`.ibd`文件将消失：

```
# ls /var/lib/mysql/nasa/
```

```
facilities.ibd
```

###### 警告

丢弃表空间会导致关联表空间文件的完全删除，并且是不可恢复的操作。如果你意外运行了`ALTER TABLE ... DISCARD TABLESPACE`，你将需要从备份中恢复。

现在我们可以复制原始`actor`表的导出表空间以及*.cfg*文件：

```
# cp -vip /opt/mysql/actor.* /var/lib/mysql/nasa/
```

```
'/opt/mysql/actor.cfg' -> '/var/lib/mysql/nasa/actor.cfg'
'/opt/mysql/actor.ibd' -> '/var/lib/mysql/nasa/actor.ibd'
```

所有步骤完成后，现在可以导入表空间并验证数据：

```
mysql> `ALTER` `TABLE` `actor` `IMPORT` `TABLESPACE``;`
```

```
Query OK, 0 rows affected (0.02 sec)
```

```
mysql> `SELECT` `*` `FROM` `nasa``.``actor` `LIMIT` `5``;`
```

```
+----------+------------+--------------+---------------------+
| actor_id | first_name | last_name    | last_update         |
+----------+------------+--------------+---------------------+
|        1 | PENELOPE   | GUINESS      | 2006-02-15 04:34:33 |
|        2 | NICK       | WAHLBERG     | 2006-02-15 04:34:33 |
|        3 | ED         | CHASE        | 2006-02-15 04:34:33 |
|        4 | JENNIFER   | DAVIS        | 2006-02-15 04:34:33 |
|        5 | JOHNNY     | LOLLOBRIGIDA | 2006-02-15 04:34:33 |
+----------+------------+--------------+---------------------+
5 rows in set (0.00 sec)
```

你可以看到我们已经从`sakila.actor`导入到`nasa.actor`中的数据。

可能最好的一点是可传输表空间的效率。你可以使用这个功能轻松地在数据库之间移动非常大的表。

## XtraBackup 单表恢复

或许令人惊讶的是，我们将再次在可传输表空间的背景下提到 XtraBackup。这是因为 XtraBackup 允许从任何现有备份中导出表。事实上，这是恢复单个表的最方便方法，也是单表或部分数据库 PITR 的第一个构建块。

这是最先进的备份和恢复技术之一，完全基于可传输表空间功能。它还继承了所有的限制：例如，它不能在非“文件-每-表”表空间上工作。我们在这里不会给出确切的步骤，只是介绍这个技术让你知道它是可能的。

要执行单表恢复，你应该首先使用`xtrabackup`运行`--export`命令行参数准备表进行导出。你可能注意到在这个命令中没有指定表的名称，实际上每个表都会被导出。让我们在之前拍摄的一个备份上运行这个命令：

```
# xtrabackup --prepare --export --target-dir=/tmp/base_backup
# ls -1 /tmp/base_backup/sakila/
```

```
actor.cfg
actor.ibd
address.cfg
address.ibd
category.cfg
category.ibd
...
```

你可以看到我们为每个表都有一个*.cfg*文件：现在每个表空间都准备好在另一个数据库中导出和导入了。从这里开始，你可以重复上一节中的步骤来恢复其中一个表的数据。

单表或部分数据库 PITR 是棘手的，在大多数数据库管理系统中都是如此。正如你在“时间点恢复”中看到的那样，MySQL 中的 PITR 基于二进制日志。对于部分恢复，这意味着所有数据库中所有表的事务都被记录，但在应用时可以通过复制筛选二进制日志。因此，部分恢复的过程是这样的：你导出所需的表，建立一个完全独立的实例，并通过复制通道用二进制日志供其使用。

你可以在社区博客和文章如[“MySQL 单表 PITR”](https://oreil.ly/jjdfT)，[“使用 MySQL 过滤二进制日志”](https://oreil.ly/YWWpY)，以及[“如何加速 MySQL PITR”](https://oreil.ly/zoWpw)中找到更多信息。

当在正确的情况下使用并且符合特定条件时，导出/导入功能是一种强大的技术。

# 测试和验证你的备份

只有在您确信可以信任它们时，备份才是好的。有很多人在最需要时备份系统却失败的例子。频繁备份并不一定能保证数据安全。

备份可能无用或失败的多种方式：

不一致的备份

这种情况的最简单示例是，在数据库运行时，错误地从多个卷中进行快照备份。结果的备份可能损坏或缺少数据。不幸的是，您进行的一些备份可能是一致的，而另一些可能不会出现足够的错误或不一致，直到为时已晚。

源数据库的损坏

如我们详细讨论的那样，物理备份将具有所有数据库页面的副本，无论是否损坏。有些工具尝试在备份过程中验证数据，但这并不完全没有错误。成功的备份可能包含无法后续读取的坏数据。

备份数据损坏

备份本身只是数据，因此容易受到与原始数据相同的问题的影响。如果备份数据在存储过程中损坏，那么即使您的备份成功，最终也可能变得毫无用处。

Bugs

事情总会发生。您使用了十几年的备份工具可能存在错误，而您可能会发现这个错误。在最好的情况下，您的备份将失败；在最坏的情况下，它可能无法恢复。

运行错误

我们都是人类，会犯错。如果一切都自动化了，这里的风险将从人为错误变为错误。

这不是您可能面临的所有问题的全面列表，但它为您提供了一些洞察力，了解即使您的备份策略是正确的，您可能会遇到的问题。让我们回顾一些您可以采取的步骤，以帮助您更好地入眠：

+   在实施备份系统时，彻底测试它，并以各种模式进行测试。确保您可以备份系统，并使用备份进行恢复。测试负载和无负载的情况。当没有连接修改数据时，您的备份可以保持一致，而在有连接修改数据时则可能失败。

+   使用物理和逻辑备份。它们具有不同的特性和故障模式，尤其是在源数据损坏时。

+   备份您的备份，或者确保它们至少与数据库一样耐久。

+   定期进行备份恢复测试。

最后一点尤其有趣。在将备份恢复并进行测试之前，不能认为任何备份是安全的。这意味着在理想情况下，您的自动化实际上会尝试使用备份构建数据库服务器，并在成功时报告成功。此外，新数据库可以附加到源作为副本，并且可以使用像[`Percona Toolkit`的`pt-table-checksum`](https://oreil.ly/YgfuM)这样的数据验证工具来检查数据一致性。

以下是物理备份数据验证的一些可能步骤：

1.  准备备份。

1.  恢复备份。

1.  在所有*.ibd*文件上运行`innochecksum`。

    以下命令将在 Linux 上并行运行四个`innochecksum`进程：

    ```
    $ find . -type f -name "*.ibd" -print0 |\
    xargs -t -0r -n1 --max-procs=4 innochecksum
    ```

1.  使用恢复的备份启动新的 MySQL 实例。可以使用备用服务器或专用*.cnf*文件，不要忘记使用非默认端口和路径。

1.  使用`mysqldump`或任何其他工具来导出所有数据，确保数据可读并提供备份的另一份副本。

1.  将新的 MySQL 实例作为原始源数据库的复制添加，并使用`pt-table-checksum`或任何其他工具验证数据是否匹配。此过程在[`xtrabackup`文档](https://oreil.ly/fHruN)及其他来源中有详细说明。

这些步骤非常复杂，可能需要很长时间，因此您应该决定是否适合您的业务和环境来使用它们。

# 数据库备份策略简介

现在我们已经覆盖了与备份和恢复相关的许多方面，我们可以组合出一个强大的备份策略。以下是我们需要考虑的要素：

时间点恢复

我们需要决定是否需要点对点恢复（PITR）功能，因为这将影响我们关于备份策略的决策。对于您的特定情况，您需要自己做决定，但我们建议默认使用 PITR。这可能会拯救生命。如果我们决定需要这种能力，我们需要设置二进制日志记录和 binlog 复制。

逻辑备份

我们可能需要逻辑备份，要么是为了其可移植性，要么是为了防止数据损坏。由于逻辑备份会显著加载源数据库，请安排在负载最轻的时候进行。在某些情况下，可能无法在生产数据库上执行逻辑备份，无论是由于时间限制、负载限制或两者兼而有之。鉴于我们仍然希望能够运行逻辑备份，我们可以使用以下技术：

+   在复制数据库上运行逻辑备份。在这种情况下，跟踪 binlog 位置可能会有问题，建议在这种情况下使用基于 GTID 的复制。

+   将逻辑备份的创建整合到物理备份的验证过程中。准备好的备份是一个数据目录，可以立即被 MySQL 服务器使用。如果你运行一个针对备份的服务器，你会破坏该备份，因此你需要先将准备好的备份复制到其他地方。

物理备份

基于操作系统、MySQL 版本、系统属性以及仔细查阅文档，我们需要选择用于物理备份的工具。为简化起见，我们选择使用 XtraBackup。

首先需要决定平均恢复时间（MTTR）目标对我们的重要性。例如，如果我们仅进行每周基础备份，可能需要应用几乎一周的事务才能恢复备份。为了减少 MTTR，可以实施每日甚至每小时的增量备份。

如果你的系统非常庞大，即使使用物理备份工具进行热备份也不可行。在这种情况下，你需要考虑对卷的快照，如果可能的话。

备份存储

我们需要确保我们的备份安全地，并且最好是冗余存储。如果我们使用硬件存储设置，可以利用性能较低但冗余的 RAID 5 或 6 阵列，或者使用不太可靠的存储设置，同时将备份持续流向云存储，比如亚马逊的 S3。或者，如果我们的备份工具允许的话，我们也可以直接使用 S3 作为默认选项。

备份测试与验证

最后，一旦我们完成了备份，我们需要实施备份测试过程。根据可用于实施和维护此练习的预算，我们应该决定每次运行多少步骤，以及哪些步骤仅定期运行。

所有这些都完成后，我们可以说我们已经覆盖了基础并且数据库已经安全备份。考虑到备份很少被使用，这可能看起来是很大的努力，但你必须记住，你最终将面临灾难——这只是时间问题。
