# 第二章：入门

MongoDB 强大而易于上手。在本章中，我们将介绍 MongoDB 的一些基本概念：

+   *文档* 是 MongoDB 的基本数据单元，大致相当于关系数据库管理系统中的一行（但更具表现力）。

+   类似地，*集合* 可以被看作是具有动态模式的表。

+   一个 MongoDB 实例可以托管多个独立的 *数据库*，每个数据库都包含其自己的集合。

+   每个文档都有一个特殊的键，`"_id"`，在集合内是唯一的。

+   MongoDB 随附一个简单但功能强大的工具，称为 *mongo shell*。*mongo* shell 提供了内置支持，用于管理 MongoDB 实例并使用 MongoDB 查询语言操作数据。它还是一个完全功能的 JavaScript 解释器，允许用户为各种目的创建和加载自己的脚本。

# 文档

MongoDB 的核心是 *文档*：一组带有关联值的有序键集。文档的表示因编程语言而异，但大多数语言都有一个自然的数据结构来适应，比如映射、哈希表或字典。例如，在 JavaScript 中，文档表示为对象：

```
{"greeting" : "Hello, world!"}
```

这个简单的文档包含一个名为 `"greeting"` 的键，其值为 `"Hello, world!"`。大多数文档会比这个简单的文档复杂，并且通常会包含多个键值对：

```
{"greeting" : "Hello, world!", "views" : 3}
```

正如您所见，文档中的值不仅仅是“blob”。它们可以是几种不同的数据类型之一（甚至是整个嵌入式文档——参见 “嵌入式文档”）。在此示例中，`"greeting"` 的值是一个字符串，而 `"views"` 的值是一个整数。

文档中的键是字符串。键中允许任何 UTF-8 字符，但有几个显著的例外：

+   键不能包含字符 *\0*（空字符）。该字符用于表示键的结尾。

+   *.* 和 *$* 字符具有一些特殊属性，只应在特定情况下使用，如后续章节所述。一般来说，它们应被视为保留字符，如果不恰当地使用，驱动程序会报错。

MongoDB 是类型敏感和区分大小写的。例如，以下文档是不同的：

```
{"count" : 5}
{"count" : "5"}
```

正如下面这些：

```
{"count" : 5}
{"Count" : 5}
```

最后需要注意的重要事项是，MongoDB 中的文档不能包含重复的键。例如，以下内容不是合法的文档：

```
{"greeting" : "Hello, world!", "greeting" : "Hello, MongoDB!"}
```

# 集合

一个 *集合* 是一组文档。如果文档是关系数据库中行的 MongoDB 类比，那么集合可以被看作是表的类比。

## 动态模式

集合具有 *动态模式*。这意味着单个集合中的文档可以具有任意数量的不同“形状”。例如，以下两个文档都可以存储在同一个集合中：

```
{"greeting" : "Hello, world!", "views": 3}
{"signoff": "Good night, and good luck"}
```

请注意，前述文档具有不同的键、不同数量的键和不同类型的值。因为任何文档都可以放入任何集合中，所以常常会问：“我们到底为什么需要单独的集合？” 没有需要为不同类型的文档创建单独模式，那么我们*为什么*需要使用多个集合呢？有几个很好的理由：

+   将不同类型的文档放在同一集合中可能会给开发人员和管理员带来噩梦。开发人员需要确保每个查询仅返回符合特定模式的文档，或者应用程序代码执行查询时可以处理不同形状的文档。如果我们要查询博客文章，那么筛选掉包含作者数据的文档会很麻烦。

+   获取集合列表要比从集合中提取文档类型列表快得多。例如，如果每个文档都有一个 `"type"` 字段，指定文档是“浏览”、“完整”还是“大块猴子”，在单个集合中查找这三个值将会慢得多，与查询三个不同的集合相比。

+   将同类文档放在同一集合中可以实现数据局部性。从仅包含帖子的集合中获取多篇博客文章可能需要的磁盘查找次数比从包含帖子和作者数据的集合中获取相同帖子少。

+   当我们创建索引时，我们开始对文档施加一些结构。（特别是在唯一索引的情况下。）这些索引是针对每个集合定义的。通过将仅包含单一类型文档的文档放入同一集合，我们可以更有效地为集合建立索引。

创建模式和将相关类型的文档分组有充分的理由。虽然不是默认要求，但为您的应用程序定义模式是一个好习惯，并且可以通过 MongoDB 的文档验证功能和许多编程语言的对象-文档映射库进行强制执行。

## 命名

集合通过其名称来识别。集合名称可以是任何 UTF-8 字符串，但有一些限制：

+   空字符串 (`""`) 不是有效的集合名称。

+   集合名称不得包含字符 *\0*（`null` 字符），因为这会标志集合名称的结尾。

+   不应创建任何以 *system.* 开头的集合名称，这是为内部集合保留的前缀。例如，*system.users* 集合包含数据库的用户，*system.namespaces* 集合包含有关数据库所有集合的信息。

+   用户创建的集合名称不应包含保留字符 *$*。尽管各种可用于数据库的驱动程序支持在集合名称中使用 *$*，因为一些系统生成的集合中包含它，但除非您正在访问这些集合之一，否则不应在名称中使用 *$*。

### 子集合

一种组织集合的常用约定是使用由 *.* 字符分隔的命名空间子集合。例如，一个包含博客的应用程序可能有一个名为 *blog.posts* 的集合和一个单独的名为 *blog.authors* 的集合。这仅仅是为了组织目的——*blog* 集合与其“子集合”之间没有关系（甚至 *blog* 集合本身都不一定存在）。

虽然子集合没有任何特殊属性，但它们非常有用，并被整合到许多 MongoDB 工具中。例如：

+   GridFS，用于存储大文件的协议，使用子集合将文件元数据与内容块分开存储（有关 GridFS 的更多信息，请参见 第六章）。

+   大多数驱动程序提供一些语法糖来访问给定集合的子集合。例如，在数据库 shell 中，`db.blog` 将给您 *blog* 集合，而 `db.blog.posts` 将给您 *blog.posts* 集合。

对于许多用例，子集合是在 MongoDB 中组织数据的一种好方法。

# 数据库

除了按集合分组文档外，MongoDB 还将集合分组到 *数据库* 中。MongoDB 的单个实例可以托管多个数据库，每个数据库都可以组合零个或多个集合。一个很好的经验法则是将单个应用程序的所有数据存储在同一个数据库中。在同一 MongoDB 服务器上存储多个应用程序或用户数据时，单独的数据库非常有用。

与集合类似，数据库通过名称标识。数据库名称可以是任何 UTF-8 字符串，但有以下限制：

+   空字符串（*“”*）不是有效的数据库名称。

+   数据库名称不能包含以下任何字符：*/*、*\*、*.*、*"*、***、*<*、*>*、*:*、*|*、*?*、*$*、（单个空格）或 *\0*（`null` 字符）。基本上，保持使用字母数字 ASCII。

+   数据库名称对大小写不敏感。

+   数据库名称限制为最多 64 个字节。

在使用 WiredTiger 存储引擎之前，历史上数据库名称变成了文件名。现在不再是这种情况。这也解释了为什么先前存在这些限制。

也有一些保留的数据库名称，您可以访问但它们具有特殊的语义。这些如下：

*admin*

*admin* 数据库在认证和授权中起着重要作用。此外，对于某些管理操作，需要访问这个数据库。更多关于 *admin* 数据库的信息请参见 第十九章。

*local*

此数据库存储特定于单个服务器的数据。在副本集中，*local*存储用于复制过程中使用的数据。*local*数据库本身永远不会被复制。（有关复制和本地数据库的更多信息，请参见第十章。）

*config*

分片 MongoDB 集群（参见第十四章）使用*config*数据库来存储每个分片的信息。

通过将数据库名称与其中的集合连接起来，您可以获得一个完全限定的集合名称，这称为*命名空间*。例如，如果您正在使用*cms*数据库中的*blog.posts*集合，则该集合的命名空间将是*cms.blog.posts*。命名空间长度限制为 120 字节，在实践中应少于 100 字节长。有关命名空间和 MongoDB 中集合的内部表示的更多信息，请参见附录 B。

# 获取和启动 MongoDB

要启动服务器，请在您选择的 Unix 命令行环境中运行*mongod*可执行文件：

```
$ mongod
2016-04-27T22:15:55.871-0400 I CONTROL  [initandlisten] MongoDB starting : 
pid=8680 port=27017 dbpath=/data/db 64-bit host=morty
2016-04-27T22:15:55.872-0400 I CONTROL  [initandlisten] db version v4.2.0
2016-04-27T22:15:55.872-0400 I CONTROL  [initandlisten] git version:
34e65e5383f7ea1726332cb175b73077ec4a1b02
2016-04-27T22:15:55.872-0400 I CONTROL  [initandlisten] allocator: system
2016-04-27T22:15:55.872-0400 I CONTROL  [initandlisten] modules: none
2016-04-27T22:15:55.872-0400 I CONTROL  [initandlisten] build environment:
2016-04-27T22:15:55.872-0400 I CONTROL  [initandlisten]     distarch: x86_64
2016-04-27T22:15:55.872-0400 I CONTROL  [initandlisten]     target_arch: x86_64
2016-04-27T22:15:55.872-0400 I CONTROL  [initandlisten] options: {}
2016-04-27T22:15:55.889-0400 I JOURNAL  [initandlisten] 
journal dir=/data/db/journal
2016-04-27T22:15:55.889-0400 I JOURNAL  [initandlisten] recover : 
no journal files
present, no recovery needed
2016-04-27T22:15:55.909-0400 I JOURNAL  [durability] Durability thread started
2016-04-27T22:15:55.909-0400 I JOURNAL  [journal writer] Journal writer thread 
started
2016-04-27T22:15:55.909-0400 I CONTROL  [initandlisten]
2016-04-27T22:15:56.777-0400 I NETWORK  [HostnameCanonicalizationWorker] 
Starting hostname canonicalization worker
2016-04-27T22:15:56.778-0400 I FTDC     [initandlisten] Initializing full-time 
diagnostic data capture with directory '/data/db/diagnostic.data'
2016-04-27T22:15:56.779-0400 I NETWORK  [initandlisten] waiting for connections 
on port 27017
```

如果您使用 Windows，请运行以下命令：

```
> mongod.exe
```

###### 提示

有关在您的系统上安装 MongoDB 的详细信息，请参见附录 A 或 MongoDB 文档中相应的[安装教程](https://oreil.ly/5WP5e)。

当不带参数运行时，*mongod*将使用默认数据目录*/data/db/*（或 Windows 当前卷上的*\data\db\*）。如果数据目录不存在或不可写，则服务器将无法启动。在启动 MongoDB 之前，创建数据目录（例如，`mkdir -p /data/db/`）并确保您的用户有写入目录的权限非常重要。

在启动时，服务器将打印一些版本和系统信息，然后开始等待连接。默认情况下，MongoDB 监听端口 27017 上的套接字连接。如果该端口不可用，服务器将无法启动——最常见的原因是已经运行另一个 MongoDB 实例。

###### 提示

您应该始终保护您的*mongod*实例。有关如何保护 MongoDB 的更多信息，请参见第十九章。

您可以在启动*mongod*的命令行环境中键入 Ctrl-C 安全停止*mongod*服务器。

###### 提示

有关启动或停止 MongoDB 的更多信息，请参见第二十一章。

# MongoDB Shell 简介

MongoDB 附带一个 JavaScript shell，允许从命令行与 MongoDB 实例交互。这个 shell 对于执行管理功能、检查运行中的实例或仅仅探索 MongoDB 非常有用。*mongo* shell 是使用 MongoDB 的关键工具。我们将在本文的其余部分广泛使用它。

## 运行 Shell

要启动 shell，请运行*mongo*可执行文件：

```
$ mongo
MongoDB shell version: 4.2.0
connecting to: test
>
```

Shell 在启动时会自动尝试连接到本地机器上运行的 MongoDB 服务器，请确保在启动 Shell 之前启动 *mongod*。

Shell 是一个功能齐全的 JavaScript 解释器，能够运行任意的 JavaScript 程序。为了说明这一点，让我们执行一些基本的数学运算：

```
> x = 200;
200
> x / 5;
40
```

我们还可以利用所有标准的 JavaScript 库：

```
> Math.sin(Math.PI / 2);
1
> new Date("20109/1/1");
ISODate("2019-01-01T05:00:00Z")
> "Hello, World!".replace("World", "MongoDB");
Hello, MongoDB!
```

我们甚至可以定义和调用 JavaScript 函数：

```
> function factorial (n) {
... if (n <= 1) return 1;
... return n * factorial(n - 1);
... }
> factorial(5);
120
```

请注意，您可以创建多行命令。当您按 Enter 键时，Shell 将检测 JavaScript 语句是否完整。如果语句不完整，Shell 将允许您在下一行继续编写它。连续按三次 Enter 将取消未完成的命令，并返回到 `>` 提示符。

## 一个 MongoDB 客户端

尽管执行任意 JavaScript 的能力非常有用，但 Shell 的真正强大之处在于它也是一个独立的 MongoDB 客户端。在启动时，Shell 连接到 MongoDB 服务器上的 *test* 数据库，并将此数据库连接分配给全局变量 `db`。这个变量是通过 Shell 访问 MongoDB 服务器的主要接入点。

要查看当前分配给 `db` 的数据库，请键入 `db` 并按 Enter：

```
> db
test
```

Shell 包含一些附加组件，这些组件不是有效的 JavaScript 语法，但由于其对 SQL shell 用户的熟悉性而实现。这些附加组件不提供任何额外的功能，但它们是很好的语法糖。例如，最重要的操作之一是选择要使用的数据库：

```
> use video
switched to db video
```

现在，如果查看 `db` 变量，可以看到它指向 *video* 数据库：

```
> db
video
```

因为这是一个 JavaScript shell，键入一个变量名将导致该名称作为表达式进行评估。然后打印出值（在本例中是数据库名称）。

您可以从 `db` 变量访问集合。例如：

```
> db.movies
```

返回当前数据库中 *movies* 集合。既然我们可以在 Shell 中访问集合，我们几乎可以执行任何数据库操作。

## Shell 的基本操作

我们可以使用四个基本操作，即创建（Create）、读取（Read）、更新（Update）和删除（Delete）（CRUD），来操作和查看 Shell 中的数据。

### 创建

`insertOne` 函数将文档添加到集合中。例如，假设我们要存储一部电影。首先，我们将创建一个名为 `movie` 的本地变量，它是一个表示我们文档的 JavaScript 对象。它将具有键 `"title"`、`"director"` 和 `"year"`（发布年份）：

```
> movie = {"title" : "Star Wars: Episode IV - A New Hope",
... "director" : "George Lucas",
... "year" : 1977}
{
	"title" : "Star Wars: Episode IV - A New Hope",
	"director" : "George Lucas",
	"year" : 1977
}
```

此对象是一个有效的 MongoDB 文档，因此我们可以使用 `insertOne` 方法将其保存到 *movies* 集合中：

```
> db.movies.insertOne(movie)
{
	"acknowledged" : true,
	"insertedId" : ObjectId("5721794b349c32b32a012b11")
}
```

电影已保存到数据库中。我们可以通过在集合上调用 `find` 来查看它：

```
> db.movies.find().pretty()
{
	"_id" : ObjectId("5721794b349c32b32a012b11"),
	"title" : "Star Wars: Episode IV - A New Hope",
	"director" : "George Lucas",
	"year" : 1977
}
```

我们可以看到添加了 `"_id"` 键，并且其他键值对都按我们输入的方式保存了下来。关于 `"_id"` 字段突然出现的原因在本章末尾有解释。

### 读取

`find` 和 `findOne` 可用于查询集合。如果我们只想从集合中看到一个文档，我们可以使用 `findOne`：

```
> db.movies.findOne()
{
	"_id" : ObjectId("5721794b349c32b32a012b11"),
	"title" : "Star Wars: Episode IV - A New Hope",
	"director" : "George Lucas",
	"year" : 1977
}
```

`find` 和 `findOne` 也可以通过查询文档的形式传递条件。这将限制查询匹配的文档。Shell 将自动显示最多匹配 `find` 的 20 个文档，但可以获取更多。（有关查询的详细信息，请参见第四章。）

### 更新

如果我们想修改我们的文章，可以使用 `updateOne`。`updateOne` 至少需要两个参数：第一个是找到要更新的文档的条件，第二个是描述要进行的更新的文档。假设我们决定为之前创建的电影启用评论。我们需要在我们的文档中添加一个评论数组作为新键的值。

要执行更新操作，我们需要使用更新操作符 `set`：

```
> db.movies.updateOne({title : "Star Wars: Episode IV - A New Hope"},
... {$set : {reviews: []}})
WriteResult({"nMatched": 1, "nUpserted": 0, "nModified": 1})
```

现在文档有一个 `"reviews"` 键。如果我们再次调用 `find`，我们可以看到新键：

```
> db.movies.find().pretty()
{
	"_id" : ObjectId("5721794b349c32b32a012b11"),
	"title" : "Star Wars: Episode IV - A New Hope",
	"director" : "George Lucas",
	"year" : 1977,
	"reviews" : [ ]
}
```

有关更新文档的详细信息，请参见“更新文档”。

### 删除

`deleteOne` 和 `deleteMany` 从数据库中永久删除文档。这两种方法都接受一个过滤文档，指定要删除的条件。例如，这将删除我们刚刚创建的电影：

```
> db.movies.deleteOne({title : "Star Wars: Episode IV - A New Hope"})
```

使用 `deleteMany` 删除匹配过滤器的所有文档。

# 数据类型

本章的开头介绍了文档的基础知识。现在你已经能够在 MongoDB 中使用 Shell 并进行尝试，本节将深入探讨一些内容。MongoDB 支持各种数据类型作为文档中的值。在本节中，我们将概述所有支持的类型。

## 基本数据类型

在 MongoDB 中，文档可以被视为“类似 JSON 的”，因为它们在概念上类似于 JavaScript 中的对象。[JSON](http://www.json.org) 是数据的简单表示形式：其规范可以用大约一段话描述（该网站证明了这一点），并且只列出六种数据类型。从许多方面来说，这是一件好事：它易于理解、解析和记忆。另一方面，由于其只有 null、布尔值、数字、字符串、数组和对象这六种类型，JSON 的表达能力有限。

虽然这些类型可以表达出色的表现力，但大多数应用程序，特别是在处理数据库时，关键的几种附加类型仍然至关重要。例如，JSON 没有日期类型，这使得处理日期比通常更加烦人。虽然有数值类型，但只有一种——无法区分浮点数和整数，更别提区分 32 位和 64 位数字了。也无法表示其他常用类型，如正则表达式或函数。

MongoDB 在保持 JSON 的基本键/值对性质的同时，增加了对许多其他数据类型的支持。每种类型的值如何表示因语言而异，但这是一个常见支持的类型列表以及它们在 shell 中作为文档的一部分如何表示的列表。最常见的类型有：

空

空类型可用于表示空值和不存在的字段：

```
{"x" : null}
```

布尔

存在布尔类型，可用于值`true`和`false`：

```
{"x" : true}
```

数字

shell 默认使用 64 位浮点数。因此，这些数字在 shell 中看起来“正常”：

```
{"x" : 3.14}
```

```
{"x" : 3}
```

对于整数，使用`NumberInt`或`NumberLong`类，分别表示 4 字节或 8 字节的有符号整数。

```
{"x" : NumberInt("3")}
{"x" : NumberLong("3")}
```

字符串

任何 UTF-8 字符的字符串都可以使用字符串类型表示：

```
{"x" : "foobar"}
```

日期

MongoDB 将日期存储为表示自 Unix 纪元（1970 年 1 月 1 日）以来的毫秒数的 64 位整数。不存储时区：

```
{"x" : new Date()}
```

正则表达式

查询可以使用 JavaScript 的正则表达式语法来使用正则表达式：

```
{"x" : /foobar/i}
```

数组

值的集合或列表可以表示为数组：

```
{"x" : ["a", "b", "c"]}
```

嵌入式文档

文档可以包含作为父文档中值嵌入的整个文档：

```
{"x" : {"foo" : "bar"}}
```

对象 ID

对象 ID 是文档的 12 字节 ID：

```
{"x" : ObjectId()}
```

有关详细信息，请参见“_id 和 ObjectIds”部分。

还有一些较少见的类型，您可能需要使用，包括：

二进制数据

二进制数据是一串任意字节的字符串。无法从 shell 中操作它。二进制数据是将非 UTF-8 字符串保存到数据库的唯一方法。

代码

MongoDB 还可以在查询和文档中存储任意 JavaScript：

```
{"x" : function() { /* ... */ }}
```

最后，还有一些大多数情况下仅在内部使用的类型（或已被其他类型取代）。这些将根据需要在文本中描述。

有关 MongoDB 数据格式的更多信息，请参见附录 B。

## 日期

在 JavaScript 中，`Date`类用于 MongoDB 的日期类型。创建新的`Date`对象时，始终调用`new Date()`，而不仅仅是`Date()`。调用构造函数作为函数（即不包括`new`）会返回日期的字符串表示，而不是实际的`Date`对象。这不是 MongoDB 的选择；这是 JavaScript 的工作原理。如果不小心始终使用`Date`构造函数，可能会得到一堆字符串和日期。字符串与日期不匹配，反之亦然，因此这可能会导致删除、更新、查询等方面出现问题。

有关 JavaScript 的`Date`类的完整解释和构造函数的可接受格式，请参见[ECMAScript 规范的第 15.9 节](http://www.ecma-international.org)。

shell 中的日期使用本地时区设置显示。但是，数据库中的日期仅存储为自纪元以来的毫秒数，因此它们没有与之关联的时区信息。（当然，时区信息可以作为另一个键的值存储。）

## 数组

数组是可以在有序操作（如列表、堆栈或队列）和无序操作（如集合）中互换使用的值。

在以下文档中，键`"things"`具有一个数组值：

```
{"things" : ["pie", 3.14]}
```

从这个例子中可以看出，数组可以包含不同的数据类型作为值（在本例中是字符串和浮点数）。实际上，数组值可以是任何正常键/值对支持的值类型，甚至是嵌套数组。

文档中数组的一个很棒的地方是 MongoDB“理解”它们的结构，并知道如何深入数组内部执行操作。这使得我们能够在数组上进行查询并使用它们的内容构建索引。例如，在前面的例子中，MongoDB 可以查询所有包含`3.14`作为`"things"`数组元素的文档。如果这是一个常见的查询，甚至可以在`"things"`键上创建一个索引来提高查询速度。

MongoDB 还允许原子更新，可以修改数组的内容，例如深入数组并将值`"pie"`更改为`pi`。我们将在文本中看到更多这类操作的示例。

## 嵌入式文档

文档可以作为键的值。这称为*嵌入式文档*。嵌入式文档可以用于以比仅有键/值对的平坦结构更自然的方式组织数据。

例如，如果我们有一个表示人的文档，并希望存储该人的地址，我们可以将这些信息嵌套在嵌入式`"address"`文档中：

```
{
    "name" : "John Doe",
    "address" : {
        "street" : "123 Park Street",
        "city" : "Anytown",
        "state" : "NY"
    }
}
```

在这个例子中，`"address"`键的值是一个带有自己的键/值对（`"street"`、`"city"`和`"state"`）的嵌入式文档。

就像数组一样，MongoDB“理解”嵌入式文档的结构，并能够深入其中来构建索引，执行查询或进行更新。

我们将深入讨论架构设计，但即使从这个基本例子中，我们也可以开始看到嵌入式文档如何改变我们处理数据的方式。在关系数据库中，前面的文档可能会被建模为两个不同表格（*people*和*addresses*）中的两行。使用 MongoDB，我们可以直接将`"address"`文档嵌入`"person"`文档中。因此，当正确使用时，嵌入式文档可以提供更自然的信息表示。

这样做的反面是在 MongoDB 中可能会有更多的数据重复。假设*addresses*在关系数据库中是一个单独的表，我们需要修正一个地址中的拼写错误。当我们与*people*和*addresses*进行联接时，我们会为所有共享地址的人得到更新后的地址。在 MongoDB 中，我们需要在每个人的文档中修正这个拼写错误。

## _id 和 ObjectIds

存储在 MongoDB 中的每个文档必须具有一个 `"_id"` 键。`"_id"` 键的值可以是任何类型，但默认为 `ObjectId`。在单个集合中，每个文档的 `"_id"` 必须具有唯一值，这确保了集合中的每个文档都可以被唯一标识。也就是说，如果您有两个集合，每个集合都可以有一个 `"_id"` 值为 `123` 的文档。但是，不能在任一集合中包含多个具有 `"_id"` 为 `123` 的文档。

### 对象标识符

`ObjectId` 是 `"_id"` 的默认类型。`ObjectId` 类被设计为轻量级，同时又能够以在不同机器间全局唯一的方式生成。MongoDB 的分布式特性是它使用 `ObjectId` 而不是像自增主键这样更传统的东西的主要原因：在多台服务器上同步自增主键是困难且耗时的。因为 MongoDB 设计为分布式数据库，能够在分片环境中生成唯一标识符至关重要。

`ObjectId`s 使用 12 字节的存储空间，这使它们具有一个包含 24 个十六进制数字的字符串表示形式：每个字节有 2 个数字。这导致它们看起来比它们实际上更大，这让一些人感到紧张。重要的是要注意，即使 `ObjectId` 经常被表示为一个巨大的十六进制字符串，这个字符串实际上是存储的数据的两倍长。

如果您快速连续创建多个新的 `ObjectId`s，您会看到每次仅最后几位数字发生变化。此外，如果您将创建间隔几秒钟，`ObjectId` 中间的一些数字将发生变化。这是由于 `ObjectId`s 的生成方式。`ObjectId` 的 12 字节生成如下：

| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 时间戳 | 随机值 | 计数器（随机起始值） |

`ObjectId` 的前四个字节是自纪元以来的秒数时间戳。这提供了一些有用的特性：

+   时间戳与接下来的五个字节（稍后会描述）结合使用，以秒为粒度提供唯一性。

+   因为时间戳首先出现，`ObjectId`s 将按照*大致*插入顺序排序。这不是一个强有力的保证，但确实具有一些好的属性，例如使 `ObjectId`s 易于索引。

+   在这四个字节中存在一个隐含的时间戳，表示每个文档创建的时间。大多数驱动程序提供了从 `ObjectId` 提取此信息的方法。

因为当前时间用于 `ObjectId`s，一些用户担心他们的服务器需要具有同步时钟。尽管出于其他原因同步时钟是一个好主意（参见 “同步时钟”），实际的时间戳对 `ObjectId`s 并不重要，只要它通常是新的（每秒一次）并且递增的。

`ObjectId`的下一个五个字节是一个随机值。最后三个字节是一个计数器，从一个随机值开始，以避免在不同机器上生成冲突的`ObjectId`：

因此，`ObjectId`的前九个字节确保了在单个秒内跨机器和进程的唯一性。最后三个字节仅仅是一个递增计数器，负责确保在单个进程内每秒生成的唯一性。这允许在单个秒内生成多达 256³（16,777,216）个*进程内*的唯一`ObjectId`。

### 自动生成的 _id

如前所述，如果在插入文档时不存在`"_id"`键，则将自动向插入的文档添加一个。这可以由 MongoDB 服务器处理，但通常由客户端驱动程序执行。

# 使用 MongoDB Shell

本节涵盖了如何将 shell 作为命令行工具的一部分使用、自定义它以及使用一些更高级的功能。

尽管我们上面连接到了一个本地的*mongod*实例，你可以连接你的 shell 到你的机器可以访问的任何 MongoDB 实例。要连接到不同机器或端口的*mongod*，请在启动 shell 时指定主机名、端口和数据库：

```
$ mongo some-host:30000/myDB
MongoDB shell version: 4.2.0
connecting to: some-host:30000/myDB
>
```

现在，`db`将引用*some-host:30000*的`myDB`数据库。

有时，在启动*mongo* shell 时根本不连接到*mongod*也很方便。如果你使用`--nodb`启动 shell，它将在启动时不尝试连接任何东西：

```
$ mongo --nodb
MongoDB shell version: 4.2.0
>
```

一旦启动，你可以通过运行``new Mongo("*`hostname`*")``随意连接到*mongod*：

```
> conn = new Mongo("some-host:30000")
connection to some-host:30000
> db = conn.getDB("myDB")
myDB
```

在执行这两个命令之后，你可以正常使用`db`。你可以随时使用这些命令连接到不同的数据库或服务器。

## 使用 Shell 的技巧

因为*mongo*仅仅是一个 JavaScript shell，你可以通过简单地在线查阅 JavaScript 文档为其获取大量帮助。对于特定于 MongoDB 的功能，该 shell 包含内置帮助，可通过输入``**`help`**``来访问：

```
> help
    db.help()                    help on db methods
    db.mycoll.help()             help on collection methods
    sh.help()                    sharding helpers
    ...

    show dbs                     show database names
    show collections             show collections in current database
    show users                   show users in current database
    ...
```

`db.help()`提供数据库级帮助，`db.foo.help()`提供集合级帮助。

弄清楚一个函数在做什么的一个好方法是不带括号地键入它。这将打印函数的 JavaScript 源代码。例如，如果你想知道`update`函数是如何工作的，或者无法记住参数的顺序，你可以执行以下操作：

```
> db.movies.updateOne
function (filter, update, options) {
    var opts = Object.extend({}, options || {});

    // Check if first key in update statement contains a $
    var keys = Object.keys(update);
    if (keys.length == 0) {
        throw new Error("the update operation document must contain at
 least one atomic operator");
    }
    ...
```

## 使用 Shell 运行脚本

除了交互式使用 shell 外，你还可以将 shell JavaScript 文件传递给执行。只需在命令行中传递你的脚本：

```
$ mongo script1.js script2.js script3.js
MongoDB shell version: 4.2.1
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 4.2.1

loading file: script1.js
I am script1.js
loading file: script2.js
I am script2.js
loading file: script3.js
I am script3.js
...
```

*mongo* shell 将执行列出的每个脚本并退出。

如果你想使用连接到非默认主机/端口*mongod*的连接运行脚本，请首先指定地址，然后再指定脚本(s)：

```
$ mongo server-1:30000/foo --quiet script1.js script2.js script3.js
```

这将在*server-1:30000*上的*foo*数据库上设置`db`，并执行这三个脚本。

您可以在脚本中使用 `print` 函数将输出打印到 stdout（如前述脚本所示）。这使您可以将 shell 作为命令管道的一部分使用。如果您计划将 shell 脚本的输出管道传递给另一个命令，请使用 `--quiet` 选项阻止打印“MongoDB shell version v4.2.0”横幅。

您还可以使用 `load` 函数在交互式 shell 内运行脚本：

```
> load("script1.js")
I am script1.js
true
>
```

脚本可以访问 `db` 变量（以及任何其他全局变量）。但是，诸如 `use db` 或 `show collections` 这样的 shell 辅助程序在文件中无效。每个这类辅助程序都有其有效的 JavaScript 等效方法，如 Table 2-1 所示。

表 2-1\. shell 辅助程序的 JavaScript 等效功能

| 辅助功能 | 等效功能 |
| --- | --- |
| `use video` | `db.getSisterDB("video")` |
| `show dbs` | `db.getMongo().getDBs()` |
| `show collections` | `db.getCollectionNames()` |

您还可以使用脚本将变量注入到 shell 中。例如，您可以编写一个简单的脚本，初始化您常用的辅助函数。例如，下面的脚本可能对 Part III 和 Part IV 有帮助。它定义了一个名为 `connectTo` 的函数，该函数连接到给定端口上的本地运行数据库，并将 `db` 设置为该连接：

```
// defineConnectTo.js

/**
 * Connect to a database and set db.
 */
var connectTo = function(port, dbname) {
    if (!port) {
        port = 27017;
    }

    if (!dbname) {
        dbname = "test";
    }

    db = connect("localhost:"+port+"/"+dbname);
    return db;
};
```

如果在 shell 中加载此脚本，则现在已定义了 `connectTo`：

```
> typeof connectTo
undefined
> load('defineConnectTo.js')
> typeof connectTo
function
```

除了添加辅助函数外，您还可以使用脚本自动化常见任务和管理活动。

默认情况下，shell 将查找您启动 shell 的目录（使用 `pwd()` 查看该目录）。如果脚本不在当前目录中，可以向 shell 提供相对或绝对路径。例如，如果您想将 shell 脚本放在 *~/my-scripts* 中，可以使用 `load("/home/myUser/my-scripts/defineConnectTo.js")` 加载 *defineConnectTo.js*。请注意，`load` 无法解析 `~`。

您可以使用 `run` 从 shell 运行命令行程序。您可以将参数作为参数传递给该函数：

```
> run("ls", "-l", "/home/myUser/my-scripts/")
sh70352| -rw-r--r--  1 myUser myUser 2012-12-13 13:15 defineConnectTo.js
sh70532| -rw-r--r--  1 myUser myUser 2013-02-22 15:10 script1.js
sh70532| -rw-r--r--  1 myUser myUser 2013-02-22 15:12 script2.js
sh70532| -rw-r--r--  1 myUser myUser 2013-02-22 15:13 script3.js
```

由于输出格式化不正常且不支持管道，这是有限使用的。

## 创建一个 .mongorc.js

如果您经常加载脚本，可能希望将它们放在 *.mongorc.js* 文件中。每当您启动 shell 时，该文件都会运行。

例如，假设您希望 shell 在您登录时向您打招呼。在您的主目录中创建一个名为 *.mongorc.js* 的文件，然后将以下行添加到该文件中：

```
// .mongorc.js

var compliment = ["attractive", "intelligent", "like Batman"];
var index = Math.floor(Math.random()*3);

print("Hello, you're looking particularly "+compliment[index]+" today!");
```

然后，当您启动 shell 时，您将看到如下内容：

```
$ mongo
MongoDB shell version: 4.2.1
connecting to: test
Hello, you're looking particularly like Batman today!
>
```

更实际地说，您可以使用此脚本设置任何您想要使用的全局变量，将长名称别名为更短的名称，并覆盖内置函数。*.mongorc.js* 最常用的用途之一是删除一些更“危险”的 shell 辅助程序。您可以通过无操作或完全未定义来覆盖诸如 `dropDatabase` 或 `deleteIndexes` 等函数：

```
var no = function() {
    print("Not on my watch.");
};

// Prevent dropping databases
db.dropDatabase = DB.prototype.dropDatabase = no;

// Prevent dropping collections
DBCollection.prototype.drop = no;

// Prevent dropping an index
DBCollection.prototype.dropIndex = no;

// Prevent dropping indexes
DBCollection.prototype.dropIndexes = no;
```

现在，如果您尝试调用任何这些函数，它将简单地打印一个错误消息。请注意，这种技术不能保护您免受恶意用户的攻击；它只能帮助防止输错。

您可以通过在启动 shell 时使用 `--norc` 选项来禁用加载您的 *.mongorc.js*。

## 自定义您的提示符

可通过将 `prompt` 变量设置为字符串或函数来覆盖默认的 shell 提示符。例如，如果您运行一个需要几分钟才能完成的查询，您可能希望有一个提示符显示当前时间，以便您知道最后操作完成的时间：

```
prompt = function() {
    return (new Date())+"> ";
};
```

另一个方便的提示可能会显示你正在使用的当前数据库：

```
prompt = function() {
    if (typeof db == 'undefined') {
        return '(nodb)> ';
    }

    // Check the last db operation
    try {
        db.runCommand({getLastError:1});
    }
    catch (e) {
        print(e);
    }

    return db+"> ";
};
```

请注意，提示函数应返回字符串，并且在捕获异常时应非常谨慎：如果您的提示变成异常，这可能会极其令人困惑！

通常，您的提示函数应包括对 `getLastError` 的调用。这可以捕获写入时的错误，并在 shell 断开连接时自动重新连接您（例如，如果重新启动 *mongod*）。

如果您希望始终使用自定义提示符（或设置几个可以在 shell 中切换的自定义提示符），*.mongorc.js* 文件是一个不错的选择。

## 编辑复杂的变量

Shell 中的多行支持有些有限：您无法编辑之前的行，当您意识到第一行有拼写错误并且当前正在处理第 15 行时，这可能会令人恼火。因此，对于更大的代码块或对象，您可能希望在编辑器中编辑它们。要这样做，请在 shell 中设置 `EDITOR` 变量（或在您的环境中设置，但既然您已经在 shell 中了…）：

```
> EDITOR="/usr/bin/emacs"
```

现在，如果您想编辑一个变量，您可以说 *`edit varname`* —例如：

```
> var wap = db.books.findOne({title: "War and Peace"});
> edit wap
```

当您完成更改时，请保存并退出编辑器。变量将被解析并加载回到 shell 中。

向您的 *.mongorc.js* 文件添加 ``EDITOR="*`/path/to/editor`*";``，这样您就不必担心再次设置它了。

## 不方便的集合名称

使用 ``db.*`collectionName`*`` 语法获取集合几乎总是有效，除非集合名称是保留字或无效的 JavaScript 属性名称。

例如，假设我们正在尝试访问 *version* 集合。我们不能说 `db.version`，因为 `db.version` 是 `db` 上的一个方法（它返回运行中的 MongoDB 服务器的版本）：

```
> db.version
function () {
    return this.serverBuildInfo().version;
}
```

要实际访问 *version* 集合，您必须使用 `getCollection` 函数：

```
> db.getCollection("version");
test.version
```

这也适用于具有不是有效 JavaScript 属性名称的字符的集合名称，例如 *foo-bar-baz* 和 *123abc*（JavaScript 属性名称只能包含字母、数字、*$* 和 *_*，并且不能以数字开头）。

另一种避免无效属性的方法是使用数组访问语法。在 JavaScript 中，`x.y` 和 `x['y']` 是相同的。这意味着可以使用变量访问子集合，而不仅仅是字面名称。因此，如果你需要对每个 *blog* 子集合执行某些操作，可以像这样进行迭代：

```
var collections = ["posts", "comments", "authors"];

for (var i in collections) {
    print(db.blog[collections[i]]);
}
```

而不是这样：

```
print(db.blog.posts);
print(db.blog.comments);
print(db.blog.authors);
```

注意，你不能执行 `db.blog.i`，这会被解释为 `test.blog.i`，而不是 `test.blog.posts`。你必须使用 `db.blog[i]` 语法，使 `i` 被解释为一个变量。

你可以使用这种技术来访问命名奇特的集合：

```
> var name = "@#&!"
> db[name].find()
```

尝试查询 `db.@#&!` 是不合法的，但 `db[name]` 是可以的。
