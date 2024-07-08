# 第三章： 创建、更新和删除文档

本章介绍了将数据移入和移出数据库的基础知识，包括以下内容：

+   向集合添加新文档

+   从集合中删除文档

+   更新现有文档

+   在所有这些操作中选择正确的安全性与速度级别

# 插入文档

插入是向 MongoDB 添加数据的基本方法。 要插入单个文档，请使用集合的`insertOne`方法：

```
> db.movies.insertOne({"title" : "Stand by Me"})
```

`insertOne`会为文档添加一个`"_id"`键（如果您没有提供），并将文档存储在 MongoDB 中。

## insertMany

如果您需要将多个文档插入集合，可以使用`insertMany`。 此方法使您可以将文档数组传递到数据库。 这样做效率更高，因为您的代码不会为每个插入的文档进行往返数据库，而是会批量插入。

在 shell 中，您可以按如下方式尝试：

```
> db.movies.drop()
true
> db.movies.insertMany([{"title" : "Ghostbusters"},
...                        {"title" : "E.T."},
...                        {"title" : "Blade Runner"}]);
{
      "acknowledged" : true,
       "insertedIds" : [
           ObjectId("572630ba11722fac4b6b4996"),
           ObjectId("572630ba11722fac4b6b4997"),
           ObjectId("572630ba11722fac4b6b4998")
       ]
}
> db.movies.find()
{ "_id" : ObjectId("572630ba11722fac4b6b4996"), "title" : "Ghostbusters" }
{ "_id" : ObjectId("572630ba11722fac4b6b4997"), "title" : "E.T." }
{ "_id" : ObjectId("572630ba11722fac4b6b4998"), "title" : "Blade Runner" }
```

一次性发送几十、几百甚至上千个文档可以显著加快插入速度。

如果要将多个文档插入单个集合，则`insertMany`非常有用。 如果仅导入原始数据（例如来自数据源或 MySQL），则可以使用像* mongoimport *这样的命令行工具，而不是批量插入。 另一方面，在将数据保存到 MongoDB 之前对数据进行处理（例如将日期转换为日期类型或添加自定义`"_id"`）通常很方便。 在这种情况下，可以使用`insertMany`来导入数据。

当前版本的 MongoDB 不接受超过 48 MB 的消息，因此一次批量插入中可以插入的数据量有限。 如果尝试插入超过 48 MB，则许多驱动程序将批量插入拆分为多个 48 MB 的批量插入。 有关详细信息，请查阅您的驱动程序文档。

使用`insertMany`执行批量插入时，如果数组中间的某个文档产生某种类型的错误，发生的情况取决于您是否选择了有序或无序操作。 作为`insertMany`的第二个参数，您可以指定一个选项文档。 在选项文档中为键`"ordered"`指定`true`，以确保按提供顺序插入文档。 指定`false`，MongoDB 可以重新排序插入以提高性能。 如果未指定排序，则默认为有序插入。 对于有序插入，传递给`insertMany`的数组定义了插入顺序。 如果文档产生插入错误，则不会插入数组中该点之后的任何文档。 对于无序插入，MongoDB 将尝试插入所有文档，而不管某些插入是否产生错误。

在本示例中，由于有序插入是默认的，只会插入前两个文档。 第三个文档会产生错误，因为您不能插入具有相同`"_id"`的两个文档：

```
> db.movies.insertMany([
    ... {"_id" : 0, "title" : "Top Gun"},
    ... {"_id" : 1, "title" : "Back to the Future"},
    ... {"_id" : 1, "title" : "Gremlins"},
    ... {"_id" : 2, "title" : "Aliens"}])
2019-04-22T12:27:57.278-0400 E QUERY    [js] BulkWriteError: write 
error at item 2 in bulk operation :
BulkWriteError({
    "writeErrors" : [
        {
            "index" : 2,
            "code" : 11000,
            "errmsg" : "E11000 duplicate key error collection: 
 test.movies index: _id_ dup key: { _id: 1.0 }",
            "op" : {
                "_id" : 1,
                "title" : "Gremlins"
            }
        }
    ],
    "writeConcernErrors" : [ ],
    "nInserted" : 2,
    "nUpserted" : 0,
    "nMatched" : 0,
    "nModified" : 0,
    "nRemoved" : 0,
    "upserted" : [ ]
})
BulkWriteError@src/mongo/shell/bulk_api.js:367:48
BulkWriteResult/this.toError@src/mongo/shell/bulk_api.js:332:24
Bulk/this.execute@src/mongo/shell/bulk_api.js:1186:23
DBCollection.prototype.insertMany@src/mongo/shell/crud_api.js:314:5
@(shell):1:1
```

如果我们指定无序插入，数组中的第一、第二和第四个文档将被插入。唯一失败的插入是第三个文档，同样是因为重复的`"_id"`错误。

```
> db.movies.insertMany([
... {"_id" : 3, "title" : "Sixteen Candles"},
... {"_id" : 4, "title" : "The Terminator"},
... {"_id" : 4, "title" : "The Princess Bride"},
... {"_id" : 5, "title" : "Scarface"}],
... {"ordered" : false})
2019-05-01T17:02:25.511-0400 E QUERY    [thread1] BulkWriteError: write
error at item 2 in bulk operation :
BulkWriteError({
  "writeErrors" : [
    {
      "index" : 2,
      "code" : 11000,
      "errmsg" : "E11000 duplicate key error index: test.movies.$_id_
 dup key: { : 4.0 }",
      "op" : {
        "_id" : 4,
        "title" : "The Princess Bride"
      }
    }
  ],
  "writeConcernErrors" : [ ],
  "nInserted" : 3,
  "nUpserted" : 0,
  "nMatched" : 0,
  "nModified" : 0,
  "nRemoved" : 0,
  "upserted" : [ ]
})
BulkWriteError@src/mongo/shell/bulk_api.js:367:48
BulkWriteResult/this.toError@src/mongo/shell/bulk_api.js:332:24
Bulk/this.execute@src/mongo/shell/bulk_api.js:1186.23
DBCollection.prototype.insertMany@src/mongo/shell/crud_api.js:314:5
@(shell):1:1
```

如果您仔细研究这些示例，您可能会注意到`insertMany`的这两次调用的输出暗示了除了简单插入之外可能支持的其他操作。虽然`insertMany`不支持除插入之外的操作，但 MongoDB 支持 Bulk Write API，使您能够在一次调用中批量处理多种类型的操作。虽然这超出了本章的范围，您可以在 MongoDB 文档中阅读有关[批量写入 API](https://docs.mongodb.org/manual/core/bulk-write-operations/)的信息。

## 插入验证

MongoDB 对插入的数据进行最小的检查：它检查文档的基本结构，并在不存在时添加一个`"_id"`字段。基本结构检查之一是大小：所有文档的大小必须小于 16 MB。这是一个相对任意的限制（可能在未来会提高）；主要用于防止糟糕的模式设计，并确保一致的性能。要查看文档*`doc`*的二进制 JSON（BSON）大小（以字节为单位），请在 shell 中运行``Object.bsonsize(*`doc`*)``。

为了给您一个 16 MB 数据的概念，整个《战争与和平》的文本只有 3.14 MB。

这些最小的检查也意味着相对容易插入无效数据（如果你试图这样做）。因此，您应该只允许信任的来源（如您的应用程序服务器）连接到数据库。所有主要语言的 MongoDB 驱动程序（以及大多数次要语言的驱动程序）在发送任何内容到数据库之前都会检查各种无效数据（例如文档过大、包含非 UTF-8 字符串或使用未识别的类型）。

## insert

在 MongoDB 3.0 之前的版本中，`insert`是将文档插入 MongoDB 的主要方法。MongoDB 驱动程序在 MongoDB 3.0 服务器发布时引入了一个新的 CRUD API。截至 MongoDB 3.2，*mongo* shell 也支持此 API，其中包括`insertOne`和`insertMany`以及其他几种方法。当前 CRUD API 的目标是使所有 CRUD 操作在驱动程序和 shell 中的语义一致和清晰。虽然诸如`insert`之类的方法仍然支持向后兼容性，但不应在今后的应用程序中使用。您应该优先考虑使用`insertOne`和`insertMany`来创建文档。

# 删除文档

现在我们的数据库中有数据了，让我们删除它。CRUD API 提供了`deleteOne`和`deleteMany`来实现此目的。这两种方法的第一个参数都是一个过滤文档。过滤器指定要匹配以删除文档的一组条件。要删除`"_id"`值为`4`的文档，我们在*mongo* shell 中使用`deleteOne`，如下所示：

```
> db.movies.find()
{ "_id" : 0, "title" : "Top Gun"}
{ "_id" : 1, "title" : "Back to the Future"}
{ "_id" : 3, "title" : "Sixteen Candles"}
{ "_id" : 4, "title" : "The Terminator"}
{ "_id" : 5, "title" : "Scarface"}
> db.movies.deleteOne({"_id" : 4})
{ "acknowledged" : true, "deletedCount" : 1 }
> db.movies.find()
{ "_id" : 0, "title" : "Top Gun"}
{ "_id" : 1, "title" : "Back to the Future"}
{ "_id" : 3, "title" : "Sixteen Candles"}
{ "_id" : 5, "title" : "Scarface"}
```

在本例中，我们使用了一个过滤器，该过滤器只能匹配集合中的一个文档，因为 `"_id"` 值在集合中是唯一的。但是，我们也可以指定一个可以匹配集合中多个文档的过滤器。在这种情况下，`deleteOne` 将删除首个与过滤器匹配的文档。首先找到哪个文档取决于几个因素，包括文档插入的顺序，对文档的更新（对于某些存储引擎），以及指定的索引。与任何数据库操作一样，确保你知道使用 `deleteOne` 将对数据产生什么影响。

要删除匹配过滤器的所有文档，请使用 `deleteMany`：

```
> db.movies.find()
{ "_id" : 0, "title" : "Top Gun", "year" : 1986 }
{ "_id" : 1, "title" : "Back to the Future", "year" : 1985 }
{ "_id" : 3, "title" : "Sixteen Candles", "year" : 1984 }
{ "_id" : 4, "title" : "The Terminator", "year" : 1984 }
{ "_id" : 5, "title" : "Scarface", "year" : 1983 }
> db.movies.deleteMany({"year" : 1984})
{ "acknowledged" : true, "deletedCount" : 2 }
> db.movies.find()
{ "_id" : 0, "title" : "Top Gun", "year" : 1986 }
{ "_id" : 1, "title" : "Back to the Future", "year" : 1985 }
{ "_id" : 5, "title" : "Scarface", "year" : 1983 }
```

作为更实际的用例，假设你想删除 *mailing.list* 集合中每个 `"opt-out"` 值为 `true` 的用户：

```
> db.mailing.list.deleteMany({"opt-out" : true})
```

在 MongoDB 3.0 之前的版本中，`remove` 是删除文档的主要方法。MongoDB 驱动程序在 MongoDB 3.0 服务器发布时同时引入了 `deleteOne` 和 `deleteMany` 方法，而 shell 在 MongoDB 3.2 开始支持这些方法。虽然为了向后兼容仍支持 `remove` 方法，但在你的应用程序中应该使用 `deleteOne` 和 `deleteMany`。当前的 CRUD API 提供了更清晰的语义集，特别是对于多文档操作，帮助应用程序开发人员避免以前 API 的一些常见陷阱。

## drop

可以使用 `deleteMany` 来删除集合中的所有文档：

```
> db.movies.find()
{ "_id" : 0, "title" : "Top Gun", "year" : 1986 }
{ "_id" : 1, "title" : "Back to the Future", "year" : 1985 }
{ "_id" : 3, "title" : "Sixteen Candles", "year" : 1984 }
{ "_id" : 4, "title" : "The Terminator", "year" : 1984 }
{ "_id" : 5, "title" : "Scarface", "year" : 1983 }
> db.movies.deleteMany({})
{ "acknowledged" : true, "deletedCount" : 5 }
> db.movies.find()
```

删除文档通常是一个相当快速的操作。但是，如果要清空整个集合，最快的方法是使用 `drop`：

```
> db.movies.drop()
true
```

然后在空集合上重新创建任何索引。

一旦数据被删除，就永远消失了。除非恢复先前备份版本的数据，否则无法撤消删除或删除文档。详细讨论 MongoDB 备份和恢复，请参阅 第二十三章。

# 更新文档

一旦文档存储在数据库中，就可以使用几种更新方法进行更改：`updateOne`、`updateMany` 和 `replaceOne`。`updateOne` 和 `updateMany` 各自将一个过滤器文档作为其第一个参数，并将描述要进行的更改的修改器文档作为第二个参数。`replaceOne` 也接受一个过滤器作为第一个参数，但是作为第二个参数，`replaceOne` 期望一个用于替换与过滤器匹配的文档的文档。

更新文档是原子性的：如果同时发生两个更新，首先到达服务器的那一个将被应用，然后将应用下一个。因此，可以安全地连续发送冲突的更新而不会损坏任何文档：最后一个更新将“获胜”。如果不想使用默认行为，可以考虑文档版本模式（参见 “模式设计模式”）。

## 文档替换

`replaceOne`完全用新文档替换匹配的文档。这对于进行重大的模式迁移非常有用（参见第九章中的模式迁移策略）。例如，假设我们正在对用户文档进行重大更改，其格式如下：

```
{
    "_id" : ObjectId("4b2b9f67a1f631733d917a7a"),
    "name" : "joe",
    "friends" : 32,
    "enemies" : 2
}
```

我们希望将`"friends"`和`"enemies"`字段移动到`"relationships"`子文档中。我们可以在 Shell 中更改文档的结构，然后用`replaceOne`替换数据库的版本：

```
> var joe = db.users.findOne({"name" : "joe"});
> joe.relationships = {"friends" : joe.friends, "enemies" : joe.enemies};
{
    "friends" : 32,
    "enemies" : 2
}
> joe.username = joe.name;
"joe"
> delete joe.friends;
true
> delete joe.enemies;
true
> delete joe.name;
true
> db.users.replaceOne({"name" : "joe"}, joe);
```

现在，执行`findOne`显示文档的结构已更新：

```
{
    "_id" : ObjectId("4b2b9f67a1f631733d917a7a"),
    "username" : "joe",
    "relationships" : {
        "friends" : 32,
        "enemies" : 2
    }
}
```

一个常见的错误是根据条件匹配多个文档，然后使用第二个参数创建一个重复的`"_id"`值。数据库会因此抛出错误，并且不会更新任何文档。

例如，假设我们创建了几个具有相同`"name"`值的文档，但我们没有意识到：

```
> db.people.find()
{"_id" : ObjectId("4b2b9f67a1f631733d917a7b"), "name" : "joe", "age" : 65}
{"_id" : ObjectId("4b2b9f67a1f631733d917a7c"), "name" : "joe", "age" : 20}
{"_id" : ObjectId("4b2b9f67a1f631733d917a7d"), "name" : "joe", "age" : 49}
```

现在，如果是 Joe #2 的生日，我们想增加他的`"age"`键的值，我们可能会这样说：

```
> joe = db.people.findOne({"name" : "joe", "age" : 20});
{
    "_id" : ObjectId("4b2b9f67a1f631733d917a7c"),
    "name" : "joe",
    "age" : 20
}
> joe.age++;
> db.people.replaceOne({"name" : "joe"}, joe);
E11001 duplicate key on update
```

发生了什么？当您进行更新时，数据库将寻找匹配`{"name" : "joe"}`的文档。它找到的第一个将是 65 岁的 Joe。它将尝试用`joe`变量中的文档替换该文档，但在这个集合中已经有一个具有相同`"_id"`的文档。因此，更新将失败，因为`"_id"`值必须是唯一的。避免这种情况的最佳方法是确保您的更新始终指定一个唯一的文档，可能是通过匹配`"_id"`之类的键。对于前面的例子，这将是正确的更新使用方法：

```
> db.people.replaceOne({"_id" : ObjectId("4b2b9f67a1f631733d917a7c")}, joe)
```

使用`"_id"`作为过滤器也是高效的，因为`"_id"`值形成了集合的主索引的基础。我们将在第五章更详细地讨论主索引和次要索引以及索引如何影响更新和其他操作。

## 使用更新操作符

通常只需更新文档的某些部分。您可以使用原子*更新操作符*来更新文档中的特定字段。更新操作符是特殊的键，可用于指定复杂的更新操作，如更改、添加或删除键，甚至操作数组和嵌入式文档。

假设我们将网站分析保存在一个集合中，并且希望每次有人访问页面时都增加计数器。我们可以使用更新操作符来原子地进行增量操作。每个 URL 及其页面浏览次数存储在如下格式的文档中：

```
{
    "_id" : ObjectId("4b253b067525f35f94b60a31"),
    "url" : "www.example.com",
    "pageviews" : 52
}
```

每当有人访问页面时，我们可以根据其 URL 找到页面，并使用`"$inc"`修饰符增加`"pageviews"`键的值：

```
> db.analytics.updateOne({"url" : "www.example.com"},
... {"$inc" : {"pageviews" : 1}})
{ "acknowledged" : true, "matchedCount" : 1, "modifiedCount" : 1 }
```

现在，如果我们执行`findOne`，会发现`"pageviews"`已增加了一次：

```
> db.analytics.findOne()
{
    "_id" : ObjectId("4b253b067525f35f94b60a31"),
    "url" : "www.example.com",
    "pageviews" : 53
}
```

在使用操作符时，`"_id"`的值不能被更改。（请注意，通过使用整个文档替换，`"_id"`*可以*被更改。）任何其他键的值，包括其他唯一索引键，都可以被修改。

### 入门“$set”修饰符

`"$set"` 用于设置字段的值。如果字段尚不存在，它将被创建。这对于更新模式或添加用户定义的键非常方便。例如，假设您有一个简单的用户配置文件，存储为以下形式的文档：

```
> db.users.findOne()
{
    "_id" : ObjectId("4b253b067525f35f94b60a31"),
    "name" : "joe",
    "age" : 30,
    "sex" : "male",
    "location" : "Wisconsin"
}
```

这是一个相当简单的用户配置文件。如果用户想要在其配置文件中存储他喜欢的书籍，可以使用 `"$set"` 添加它：

```
> db.users.updateOne({"_id" : ObjectId("4b253b067525f35f94b60a31")},
... {"$set" : {"favorite book" : "War and Peace"}})
```

现在文档将有一个 `"favorite book"` 键：

```
> db.users.findOne()
{
    "_id" : ObjectId("4b253b067525f35f94b60a31"),
    "name" : "joe",
    "age" : 30,
    "sex" : "male",
    "location" : "Wisconsin",
    "favorite book" : "War and Peace"
}
```

如果用户决定他实际上喜欢不同的书籍，可以再次使用 `"$set"` 更改值：

```
> db.users.updateOne({"name" : "joe"},
... {"$set" : {"favorite book" : "Green Eggs and Ham"}})
```

`"$set"` 甚至可以更改它修改的键的类型。例如，如果我们善变的用户决定他实际上喜欢很多书籍，可以将 `"favorite book"` 键的值更改为数组：

```
> db.users.updateOne({"name" : "joe"},
... {"$set" : {"favorite book" :
...     ["Cat's Cradle", "Foundation Trilogy", "Ender's Game"]}})
```

如果用户意识到他实际上并不喜欢阅读，可以使用 `"$unset"` 完全删除该键：

```
> db.users.updateOne({"name" : "joe"},
... {"$unset" : {"favorite book" : 1}})
```

现在文档将与本例开始时的文档相同。

您还可以使用 `"$set"` 进行嵌入式文档的更改：

```
> db.blog.posts.findOne()
{
    "_id" : ObjectId("4b253b067525f35f94b60a31"),
    "title" : "A Blog Post",
    "content" : "...",
    "author" : {
        "name" : "joe",
        "email" : "joe@example.com"
    }
}
> db.blog.posts.updateOne({"author.name" : "joe"},
... {"$set" : {"author.name" : "joe schmoe"}})

> db.blog.posts.findOne()
{
    "_id" : ObjectId("4b253b067525f35f94b60a31"),
    "title" : "A Blog Post",
    "content" : "...",
    "author" : {
        "name" : "joe schmoe",
        "email" : "joe@example.com"
    }
}
```

您必须始终使用 `$` 修饰符来添加、更改或移除键。初学者开始时常见的错误是尝试通过类似于以下更新来将键的值设置为其他值：

```
> db.blog.posts.updateOne({"author.name" : "joe"}, 
... {"author.name" : "joe schmoe"})
```

这将导致错误。更新文档必须包含更新操作符。之前的 CRUD API 版本没有捕获这种类型的错误。在早期的更新方法中，在这种情况下通常会简单地完成整个文档的替换。正是这种陷阱导致了新的 CRUD API 的创建。

### 自增和自减

`"$inc"` 操作符可用于更改现有键的值或在尚不存在时创建新键。用于更新分析、声望、投票或任何具有可变数值的内容非常有用。

假设我们正在创建一个游戏集合，我们希望保存游戏并在其变化时更新分数。当用户开始玩，比如说弹球游戏时，我们可以插入一个文档，其中包含游戏名称和正在玩游戏的用户：

```
> db.games.insertOne({"game" : "pinball", "user" : "joe"})
```

当球撞到保险杠时，游戏应增加玩家的分数。由于弹球中的分数很容易获得，让我们假设玩家可以获得的基本单位分数是 50。我们可以使用 `"$inc"` 修饰符将 50 添加到玩家的分数中：

```
> db.games.updateOne({"game" : "pinball", "user" : "joe"},
... {"$inc" : {"score" : 50}})
```

如果我们在此更新之后查看文档，我们将看到以下内容：

```
> db.games.findOne()
{
     "_id" : ObjectId("4b2d75476cc613d5ee930164"),
     "game" : "pinball",
     "user" : "joe",
     "score" : 50
}
```

`"score"` 键尚不存在，因此 `"$inc"` 创建了它，并设置为增量金额：`50`。

如果球落入“奖励”槽中，我们希望在分数上增加 10,000。我们可以通过向 `"$inc"` 传递不同的值来实现这一点：

```
> db.games.updateOne({"game" : "pinball", "user" : "joe"},
... {"$inc" : {"score" : 10000}})
```

现在如果我们查看游戏，我们将看到以下内容：

```
> db.games.findOne()
{
     "_id" : ObjectId("4b2d75476cc613d5ee930164"),
     "game" : "pinball",
     "user" : "joe",
     "score" : 10050
}
```

`"score"` 键存在且具有数值，因此服务器向其添加了 10,000。

`"$inc"`类似于`"$set"`，但设计用于增加（和减少）数字。`"$inc"`只能用于整数、长整数、双精度浮点数或十进制数值类型的值。如果用于其他类型的值，操作将失败。这包括许多语言会自动转换为数字的类型，如空值、布尔值或数值字符的字符串：

```
> db.strcounts.insert({"count" : "1"})
WriteResult({ "nInserted" : 1 })
> db.strcounts.update({}, {"$inc" : {"count" : 1}})
WriteResult({
  "nMatched" : 0,
  "nUpserted" : 0,
  "nModified" : 0,
  "writeError" : {
    "code" : 16837,
    "errmsg" : "Cannot apply $inc to a value of non-numeric type.
 {_id: ObjectId('5726c0d36855a935cb57a659')} has the field 'count' of
 non-numeric type String"
  }
})
```

此外，`"$inc"`键的值必须是一个数字。不能按字符串、数组或其他非数值值增加。这样做将导致“Modifier `"$inc"` allowed for numbers only”错误消息。要修改其他类型，请使用`"$set"`或以下数组操作符之一。

### 数组操作符

存在大量的更新操作符用于操作数组。数组是常见且功能强大的数据结构：它们不仅可以作为可以通过索引引用的列表，还可以充当集合。

#### 添加元素

`"$push"`会将元素添加到数组末尾（如果数组存在）或创建一个新数组（如果不存在）。例如，假设我们正在存储博客文章，并希望添加一个包含数组的`"comments"`键。我们可以将评论推入不存在的`"comments"`数组中，这将创建数组并添加评论：

```
> db.blog.posts.findOne()
{
    "_id" : ObjectId("4b2d75476cc613d5ee930164"),
    "title" : "A blog post",
    "content" : "..."
}
> db.blog.posts.updateOne({"title" : "A blog post"},
... {"$push" : {"comments" :
...     {"name" : "joe", "email" : "joe@example.com",
...     "content" : "nice post."}}})
{ "acknowledged" : true, "matchedCount" : 1, "modifiedCount" : 1 }
> db.blog.posts.findOne()
{
    "_id" : ObjectId("4b2d75476cc613d5ee930164"),
    "title" : "A blog post",
    "content" : "...",
    "comments" : [
        {
            "name" : "joe",
            "email" : "joe@example.com",
            "content" : "nice post."
        }
    ]
}
```

现在，如果我们想再添加一个评论，只需再次使用`"$push"`即可：

```
> db.blog.posts.updateOne({"title" : "A blog post"},
... {"$push" : {"comments" :
...     {"name" : "bob", "email" : "bob@example.com",
...     "content" : "good post."}}})
{ "acknowledged" : true, "matchedCount" : 1, "modifiedCount" : 1 }
> db.blog.posts.findOne()
{
    "_id" : ObjectId("4b2d75476cc613d5ee930164"),
    "title" : "A blog post",
    "content" : "...",
    "comments" : [
        {
            "name" : "joe",
            "email" : "joe@example.com",
            "content" : "nice post."
        },
        {
            "name" : "bob",
            "email" : "bob@example.com",
            "content" : "good post."
        }
    ]
}
```

这是`"push"`的“简单”形式，但你也可以用它进行更复杂的数组操作。MongoDB 查询语言为某些操作符提供了修饰符，包括`"$push"`。你可以使用`"$each"`修饰符一次推送多个值到数组中的`"$push"`操作中：

```
> db.stock.ticker.updateOne({"_id" : "GOOG"},
... {"$push" : {"hourly" : {"$each" : [562.776, 562.790, 559.123]}}})
```

这将向数组推入三个新元素。

如果你只希望数组增长到特定长度，可以使用`"$slice"`修饰符和`"$push"`一起防止数组超出某个大小，有效地创建一个“前 N”项列表：

```
> db.movies.updateOne({"genre" : "horror"},
... {"$push" : {"top10" : {"$each" : ["Nightmare on Elm Street", "Saw"],
...                        "$slice" : -10}}})
```

此示例将数组限制为最后推入的 10 个元素。

如果数组在推入后少于 10 个元素，则保留所有元素。如果数组大于 10 个元素，则只保留最后 10 个元素。因此，`"$slice"`可用于在文档中创建队列。

最后，你可以在裁剪之前对`"$push"`操作应用`"$sort"`修饰符：

```
> db.movies.updateOne({"genre" : "horror"},
... {"$push" : {"top10" : {"$each" : [{"name" : "Nightmare on Elm Street",
...                                    "rating" : 6.6},
...                                   {"name" : "Saw", "rating" : 4.3}],
...                        "$slice" : -10,
...                        "$sort" : {"rating" : -1}}}})
```

这会按照数组中的`"rating"`字段对所有对象进行排序，然后保留前 10 个。请注意，你必须包含`"$each"`；不能仅使用`"$slice"`或`"$sort"`来对带有`"$push"`的数组进行操作。

#### 使用数组作为集合

如果希望将数组视为集合，只在值不存在时添加，可以在查询文档中使用`"$ne"`来实现。例如，要将作者推入引文列表，但仅在其不存在时才添加，使用以下方法：

```
> db.papers.updateOne({"authors cited" : {"$ne" : "Richie"}},
... {$push : {"authors cited" : "Richie"}})
```

这也可以通过`"$addToSet"`完成，对于`"$ne"`不适用或`"$addToSet"`描述更准确的情况非常有用。

例如，假设你有一个代表用户的文档。你可能有一组用户添加的电子邮件地址：

```
> db.users.findOne({"_id" : ObjectId("4b2d75476cc613d5ee930164")})
{
    "_id" : ObjectId("4b2d75476cc613d5ee930164"),
    "username" : "joe",
    "emails" : [
        "joe@example.com",
        "joe@gmail.com",
        "joe@yahoo.com"
    ]
}
```

在添加另一个地址时，可以使用“`$addToSet"`来防止重复：

```
> db.users.updateOne({"_id" : ObjectId("4b2d75476cc613d5ee930164")},
... {"$addToSet" : {"emails" : "joe@gmail.com"}})
{ "acknowledged" : true, "matchedCount" : 1, "modifiedCount" : 0 }
> db.users.findOne({"_id" : ObjectId("4b2d75476cc613d5ee930164")})
{
    "_id" : ObjectId("4b2d75476cc613d5ee930164"),
    "username" : "joe",
    "emails" : [
        "joe@example.com",
        "joe@gmail.com",
        "joe@yahoo.com",
    ]
}
> db.users.updateOne({"_id" : ObjectId("4b2d75476cc613d5ee930164")},
... {"$addToSet" : {"emails" : "joe@hotmail.com"}})
{ "acknowledged" : true, "matchedCount" : 1, "modifiedCount" : 1 }
> db.users.findOne({"_id" : ObjectId("4b2d75476cc613d5ee930164")})
{
    "_id" : ObjectId("4b2d75476cc613d5ee930164"),
    "username" : "joe",
    "emails" : [
        "joe@example.com",
        "joe@gmail.com",
        "joe@yahoo.com",
        "joe@hotmail.com"
    ]
}
```

您还可以将`"$addToSet"`与`"$each"`结合使用，以添加多个唯一值，这是无法通过`"$ne"`/`"$push"`组合实现的。例如，如果用户想要添加多个电子邮件地址，可以使用这些运算符：

```
> db.users.updateOne({"_id" : ObjectId("4b2d75476cc613d5ee930164")}, 
... {"$addToSet" : {"emails" : {"$each" :
...     ["joe@php.net", "joe@example.com", "joe@python.org"]}}})
{ "acknowledged" : true, "matchedCount" : 1, "modifiedCount" : 1 }
> db.users.findOne({"_id" : ObjectId("4b2d75476cc613d5ee930164")})
{
    "_id" : ObjectId("4b2d75476cc613d5ee930164"),
    "username" : "joe",
    "emails" : [
        "joe@example.com",
        "joe@gmail.com",
        "joe@yahoo.com",
        "joe@hotmail.com"
        "joe@php.net"
        "joe@python.org"
    ]
}
```

#### 删除元素

有几种方法可以从数组中删除元素。如果您想要像队列或堆栈一样处理数组，可以使用`"$pop"`，它可以从任一端删除元素。`{"$pop" : {"*`key`*" : 1}}`从数组末尾移除一个元素。`{"$pop" : {"*`key`*" : -1}}`从开头移除一个元素。

有时应根据特定标准删除元素，而不是其在数组中的位置。`"$pull"`用于删除与给定标准匹配的数组元素。例如，假设我们有一个不需要按特定顺序完成的任务清单：

```
> db.lists.insertOne({"todo" : ["dishes", "laundry", "dry cleaning"]})
```

如果我们首先洗衣服，可以使用以下方法将其从列表中移除：

```
> db.lists.updateOne({}, {"$pull" : {"todo" : "laundry"}})
```

现在，如果我们进行查找，我们会看到数组中只剩下两个元素：

```
> db.lists.findOne()
{
    "_id" : ObjectId("4b2d75476cc613d5ee930164"),
    "todo" : [
        "dishes",
        "dry cleaning"
    ]
}
```

`"$pull"`删除所有匹配的文档，而不仅仅是单个匹配。如果您有一个看起来像`[1, 1, 2, 1]`的数组，并且使用`pull 1`，您将得到一个单元素数组`[2]`。

数组运算符只能用于具有数组值的键。例如，您不能将整数推入栈或从字符串中弹出。使用`"$set"`或`"$inc"`来修改标量值。

#### 数组位置修改

当数组中有多个值且希望修改其中一些时，数组操作变得有些棘手。有两种方式可以操作数组中的值：按位置或使用位置运算符（`$`字符）。

数组使用基于 0 的索引，可以选择元素，就像它们的索引是文档键一样。例如，假设我们有一个包含几个嵌入文档的文档，如带有评论的博客文章：

```
> db.blog.posts.findOne()
{
    "_id" : ObjectId("4b329a216cc613d5ee930192"),
    "content" : "...",
    "comments" : [
        {
            "comment" : "good post",
            "author" : "John",
            "votes" : 0
        },
        {
            "comment" : "i thought it was too short",
            "author" : "Claire",
            "votes" : 3
        },
        {
            "comment" : "free watches",
            "author" : "Alice",
            "votes" : -5
        },
        {
            "comment" : "vacation getaways",
            "author" : "Lynn",
            "votes" : -7
        }
    ]
}
```

如果我们要增加第一条评论的投票数，我们可以这样说：

```
> db.blog.updateOne({"post" : post_id},
... {"$inc" : {"comments.0.votes" : 1}})
```

然而，在许多情况下，我们不知道要修改的数组的索引，除非首先查询文档并检查它。为了解决这个问题，MongoDB 引入了位置运算符`$`，它可以确定查询文档匹配的数组元素，并更新该元素。例如，如果我们有一个名为 John 的用户，他将他的名字更新为 Jim，我们可以使用位置运算符在评论中替换它：

```
> db.blog.updateOne({"comments.author" : "John"},
... {"$set" : {"comments.$.author" : "Jim"}})
```

位置运算符仅更新第一个匹配项。因此，如果 John 留下了多条评论，他的名字只会在第一条评论中更改。

#### 使用数组过滤器进行更新

MongoDB 3.6 引入了另一种更新单个数组元素的选项：`arrayFilters`。此选项使我们能够修改与特定条件匹配的数组元素。例如，如果我们想要隐藏所有具有五个或更多负面评价的评论，我们可以做如下操作：

```
db.blog.updateOne(
   {"post" : post_id },
   { $set: { "comments.$[elem].hidden" : true } },
   {
     arrayFilters: [ { "elem.votes": { $lte: -5 } } ]
   }
)
```

此命令将`elem`定义为`"comments"`数组中每个匹配元素的标识符。如果由`elem`标识的评论的`votes`值小于或等于`-5`，我们将向`"comments"`文档添加一个名为`"hidden"`的字段，并将其值设置为`true`。

## Upserts

*Upsert* 是一种特殊类型的更新。如果未找到与过滤器匹配的文档，将创建一个新文档，通过组合条件和更新文档。如果找到匹配的文档，则正常更新。Upserts 很方便，因为它们可以消除“播种”集合的需要：通常可以使用相同的代码创建和更新文档。

让我们回到记录网站每个页面访问量的示例。如果没有 upsert，我们可能会尝试查找 URL 并增加访问量，或者在 URL 不存在时创建新文档。如果将其编写为 JavaScript 程序，它可能看起来像以下内容：

```
// check if we have an entry for this page
blog = db.analytics.findOne({url : "/blog"})

// if we do, add one to the number of views and save
if (blog) {
  blog.pageviews++;
  db.analytics.save(blog);
}
// otherwise, create a new document for this page
else {
  db.analytics.insertOne({url : "/blog", pageviews : 1})
}
```

这意味着每当有人访问页面时，我们都会对数据库进行一次往返，并发送更新或插入操作。如果在多个进程中运行此代码，我们还可能会遇到竞争条件，其中多个文档可能会插入给定的 URL。

我们可以通过仅向数据库发送 upsert（`updateOne`和`updateMany`的第三个参数是一个选项文档，允许我们指定此操作）来消除竞争条件并减少代码量：

```
> db.analytics.updateOne({"url" : "/blog"}, {"$inc" : {"pageviews" : 1}}, 
... {"upsert" : true})
```

此行代码与前一个代码块完全相同，只是速度更快且原子化！通过使用条件文档作为基础并应用任何修改文档来创建新文档。

例如，如果您执行一个 upsert，该 upsert 匹配一个键并增加到该键的值，则增量将应用于匹配：

```
> db.users.updateOne({"rep" : 25}, {"$inc" : {"rep" : 3}}, {"upsert" : true})
WriteResult({
    "acknowledged" : true,
    "matchedCount" : 0,
    "modifiedCount" : 0,
    "upsertedId" : ObjectId("5a93b07aaea1cb8780a4cf72")
})
> db.users.findOne({"_id" : ObjectId("5727b2a7223502483c7f3acd")} )
{ "_id" : ObjectId("5727b2a7223502483c7f3acd"), "rep" : 28 }
```

Upsert 创建一个具有`"rep"`为`25`的新文档，然后将其增加`3`，得到一个`"rep"`为`28`的文档。如果未指定 upsert 选项，则`{"rep": 25}`不会匹配任何文档，因此不会发生任何操作。

如果我们再次运行 upsert（使用条件`{"rep": 25}`），它将创建另一个新文档。这是因为条件与集合中唯一文档不匹配（其`"rep"`为`28`）。

有时，在创建文档时需要设置字段，但在后续更新时不更改。这就是`"$setOnInsert"`的用途。`"$setOnInsert"`是一个运算符，只在插入文档时设置字段的值。因此，我们可以像这样做：

```
> db.users.updateOne({}, {"$setOnInsert" : {"createdAt" : new Date()}},
... {"upsert" : true})
{
    "acknowledged" : true,
    "matchedCount" : 0,
    "modifiedCount" : 0,
    "upsertedId" : ObjectId("5727b4ac223502483c7f3ace")
}
> db.users.findOne()
{
    "_id" : ObjectId("5727b4ac223502483c7f3ace"),
    "createdAt" : ISODate("2016-05-02T20:12:28.640Z")
}
```

如果我们再次运行此更新，它将匹配现有文档，不会插入任何内容，因此`"createdAt"`字段不会更改：

```
> db.users.updateOne({}, {"$setOnInsert" : {"createdAt" : new Date()}},
... {"upsert" : true})
{ "acknowledged" : true, "matchedCount" : 1, "modifiedCount" : 0 }
> db.users.findOne()
{
    "_id" : ObjectId("5727b4ac223502483c7f3ace"),
    "createdAt" : ISODate("2016-05-02T20:12:28.640Z")
}
```

请注意，通常不需要保留`"createdAt"`字段，因为`ObjectId`包含文档创建时的时间戳。但是，对于不使用`ObjectId`的集合，`"$setOnInsert"`可用于创建填充、初始化计数器等情况。

### 保存 shell 辅助程序

`save`是一个 shell 函数，允许您在文档不存在时插入它，并在存在时更新它。它接受一个参数：一个文档。如果文档包含`"_id"`键，`save`将执行 upsert 操作。否则，它将执行插入操作。`save`实际上只是一个便利函数，使程序员可以快速在 shell 中修改文档：

```
> var x = db.testcol.findOne()
> x.num = 42
42
> db.testcol.save(x)
```

如果没有`save`，最后一行将会更加繁琐：

```
db.testcol.replaceOne({"_id" : x._id}, x)
```

## 更新多个文档

在本章中，我们迄今为止使用了`updateOne`来说明更新操作。`updateOne`只会更新符合过滤条件的第一个找到的文档。如果有更多匹配的文档，它们将保持不变。要修改所有匹配过滤条件的文档，请使用`updateMany`。`updateMany`遵循与`updateOne`相同的语义并接受相同的参数。主要区别在于可能被更改的文档数量。

`updateMany`为执行架构迁移或向特定用户推出新功能提供了强大的工具。例如，假设我们想要给每个生日在某一天的用户送礼物。我们可以使用`updateMany`向他们的账户添加一个`"gift"`。例如：

```
> db.users.insertMany([
... {birthday: "10/13/1978"},
... {birthday: "10/13/1978"},
... {birthday: "10/13/1978"}])
{
    "acknowledged" : true,
    "insertedIds" : [
        ObjectId("5727d6fc6855a935cb57a65b"),
        ObjectId("5727d6fc6855a935cb57a65c"),
        ObjectId("5727d6fc6855a935cb57a65d")
    ]
}
> db.users.updateMany({"birthday" : "10/13/1978"},
... {"$set" : {"gift" : "Happy Birthday!"}})
{ "acknowledged" : true, "matchedCount" : 3, "modifiedCount" : 3 }
```

调用`updateMany`方法，为我们刚刚插入到*users*集合中的三个文档添加了一个`"gift"`字段。

## 返回更新后的文档

对于某些用例，返回修改后的文档非常重要。在 MongoDB 的早期版本中，`findAndModify`是这种情况下的首选方法。它非常适合操作队列和执行其他需要获取并设置样式原子性的操作。但是，`findAndModify`容易出现用户错误，因为它是一个复杂的方法，结合了删除、替换和更新（包括 upsert）的功能。

MongoDB 3.2 引入了三种新的集合方法到 shell 中，以适应`findAndModify`的功能，但其语义更易学习和记忆：`findOneAndDelete`、`findOneAndReplace`和`findOneAndUpdate`。这些方法与例如`updateOne`之间的主要区别在于它们使您能够原子地获取修改后的文档的值。MongoDB 4.2 扩展了`findOneAndUpdate`以接受用于更新的聚合管道。该管道可以包括以下阶段：`$addFields`及其别名`$set`，`$project`及其别名`$unset`，以及`$replaceRoot`及其别名`$replaceWith`。

假设我们有一系列按特定顺序运行的进程。每个进程用一个具有以下形式的文档表示：

```
{
    "_id" : ObjectId(),
    "status" : "*`state`*",
    "priority" : *`N`*
}
```

`"status"`是一个字符串，可以是`"READY"`、`"RUNNING"`或`"DONE"`。我们需要找到处于`"READY"`状态且具有最高优先级的作业，运行处理函数，然后将状态更新为`"DONE"`。我们可以尝试查询准备好的进程，按优先级排序，并将最高优先级进程的状态更新为`"RUNNING"`。一旦处理完毕，我们将状态更新为`"DONE"`。这看起来像下面这样：

```
var cursor = db.processes.find({"status" : "READY"});
ps = cursor.sort({"priority" : -1}).limit(1).next();
db.processes.updateOne({"_id" : ps._id}, {"$set" : {"status" : "RUNNING"}});
do_something(ps);
db.processes.updateOne({"_id" : ps._id}, {"$set" : {"status" : "DONE"}});
```

此算法并不优秀，因为它存在竞态条件。假设有两个线程在运行。如果一个线程（称为 A）在更新其状态为`"RUNNING"`之前检索了文档，并且另一个线程（称为 B）在 A 更新状态之前也检索了同一个文档，那么这两个线程将运行相同的进程。我们可以通过在更新查询中检查结果来避免这种情况，但这会变得复杂：

```
var cursor = db.processes.find({"status" : "READY"});
cursor.sort({"priority" : -1}).limit(1);
while ((ps = cursor.next()) != null) {
    var result = db.processes.updateOne({"_id" : ps._id, "status" : "READY"},
                              {"$set" : {"status" : "RUNNING"}});

    if (result.modifiedCount === 1) {
        do_something(ps);
        db.processes.updateOne({"_id" : ps._id}, {"$set" : {"status" : "DONE"}});
        break;
    }
    cursor = db.processes.find({"status" : "READY"});
    cursor.sort({"priority" : -1}).limit(1);
}
```

另外，根据时间的不同，一个线程可能会完成所有工作，而另一个线程则无用地跟随其后。线程 A 可能总是抢占进程，然后 B 尝试获取相同的进程，失败后留给 A 来完成所有工作。

这种情况非常适合使用`findOneAndUpdate`。`findOneAndUpdate`可以在单个操作中返回项目并更新它。在这种情况下，看起来像下面这样：

```
> db.processes.findOneAndUpdate({"status" : "READY"},
... {"$set" : {"status" : "RUNNING"}},
... {"sort" : {"priority" : -1}})
{
    "_id" : ObjectId("4b3e7a18005cab32be6291f7"),
    "priority" : 1,
    "status" : "READY"
}
```

注意，在返回的文档中状态仍然为`"READY"`，因为`findOneAndUpdate`方法默认返回修改之前的文档状态。如果我们将选项文档中的`"returnNewDocument"`字段设置为`true`，它将返回更新后的文档。选项文档作为`findOneAndUpdate`的第三个参数传递：

```
> db.processes.findOneAndUpdate({"status" : "READY"},
... {"$set" : {"status" : "RUNNING"}},
... {"sort" : {"priority" : -1},
...  "returnNewDocument": true})
{
    "_id" : ObjectId("4b3e7a18005cab32be6291f7"),
    "priority" : 1,
    "status" : "RUNNING"
}
```

因此，程序变成了以下内容：

```
ps = db.processes.findOneAndUpdate({"status" : "READY"},
                                   {"$set" : {"status" : "RUNNING"}},
                                   {"sort" : {"priority" : -1},
                                    "returnNewDocument": true})
do_something(ps)
db.process.updateOne({"_id" : ps._id}, {"$set" : {"status" : "DONE"}})
```

除此之外，还有另外两种你应该知道的方法。`findOneAndReplace`接受相同的参数，并返回与过滤器匹配的文档，无论替换前还是替换后，这取决于`returnNewDocument`的值。`findOneAndDelete`类似，但它不接受更新文档作为参数，并且具有另外两种方法的子集选项。`findOneAndDelete`返回被删除的文档。
