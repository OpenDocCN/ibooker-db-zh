# 第十七章：分片管理

与副本集一样，您可以选择多种选项来管理分片集群。手动管理是一种选择。如今，越来越常见的做法是使用工具如 Ops Manager、Cloud Manager 和 Atlas Database-as-a-Service（DBaaS）提供的所有集群管理功能。在本章中，我们将演示如何手动管理分片集群，包括：

+   检查集群的状态：其成员是谁，数据存储在哪里，以及有哪些打开的连接

+   添加、移除和更改集群的成员

+   管理数据移动和手动移动数据

# 查看当前状态

有几个辅助工具可用于查找数据所在位置、分片信息以及集群的运行状态。

## 使用`sh.status()`获取摘要

`sh.status()`为您提供了有关分片、数据库和分片集合的概述。如果您有少量块，它将打印哪些块位于何处的详细信息。否则，它将仅提供集合的分片键，并报告每个分片有多少块：

```
> sh.status()
--- Sharding Status --- 
sharding version: {
  "_id" : 1,
  "minCompatibleVersion" : 5,
  "currentVersion" : 6,
  "clusterId" : ObjectId("5bdf51ecf8c192ed922f3160")
}
shards:
  {  "_id" : "shard01",  
     "host" : "shard01/localhost:27018,localhost:27019,localhost:27020",  
     "state" : 1 }
  {  "_id" : "shard02",  
     "host" : "shard02/localhost:27021,localhost:27022,localhost:27023",  
     "state" : 1 }
  {  "_id" : "shard03",  
     "host" : "shard03/localhost:27024,localhost:27025,localhost:27026",  
     "state" : 1 }
active mongoses:
  "4.0.3" : 1
autosplit:
  Currently enabled: yes
balancer:
  Currently enabled:  yes
  Currently running:  no
  Failed balancer rounds in last 5 attempts:  0
  Migration Results for the last 24 hours: 
    6 : Success
  databases:
    {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
      config.system.sessions
        shard key: { "_id" : 1 }
        unique: false
        balancing: true
        chunks:
          shard01	1
          { "_id" : { "$minKey" : 1 } } -->> 
          { "_id" : { "$maxKey" : 1 } } on : shard01 Timestamp(1, 0) 
    {  "_id" : "video",  "primary" : "shard02",  "partitioned" : true,
       "version" : 
          {  "uuid" : UUID("3d83d8b8-9260-4a6f-8d28-c3732d40d961"),  
             "lastMod" : 1 } }
       video.movies
         shard key: { "imdbId" : "hashed" }
         unique: false
         balancing: true
         chunks:
           shard01 3
           shard02 4
           shard03 3
           { "imdbId" : { "$minKey" : 1 } } -->> 
               { "imdbId" : NumberLong("-7262221363006655132") } on : 
               shard01 Timestamp(2, 0) 
           { "imdbId" : NumberLong("-7262221363006655132") } -->> 
               { "imdbId" : NumberLong("-5315530662268120007") } on : 
               shard03 Timestamp(3, 0) 
           { "imdbId" : NumberLong("-5315530662268120007") } -->> 
               { "imdbId" : NumberLong("-3362204802044524341") } on : 
               shard03 Timestamp(4, 0) 
           { "imdbId" : NumberLong("-3362204802044524341") } -->> 
               { "imdbId" : NumberLong("-1412311662519947087") } 
               on : shard01 Timestamp(5, 0) 
           { "imdbId" : NumberLong("-1412311662519947087") } -->> 
               { "imdbId" : NumberLong("524277486033652998") } on : 
               shard01 Timestamp(6, 0) 
           { "imdbId" : NumberLong("524277486033652998") } -->> 
               { "imdbId" : NumberLong("2484315172280977547") } on : 
               shard03 Timestamp(7, 0) 
           { "imdbId" : NumberLong("2484315172280977547") } -->> 
               { "imdbId" : NumberLong("4436141279217488250") } on : 
               shard02 Timestamp(7, 1) 
           { "imdbId" : NumberLong("4436141279217488250") } -->> 
               { "imdbId" : NumberLong("6386258634539951337") } on : 
               shard02 Timestamp(1, 7) 
           { "imdbId" : NumberLong("6386258634539951337") } -->> 
               { "imdbId" : NumberLong("8345072417171006784") } on : 
               shard02 Timestamp(1, 8) 
           { "imdbId" : NumberLong("8345072417171006784") } -->> 
               { "imdbId" : { "$maxKey" : 1 } } on : 
               shard02 Timestamp(1, 9)
```

一旦有几个块以上，`sh.status()`将总结块统计信息，而不是打印每个块。要查看所有块，请运行`sh.status(true)`（`true`告诉`sh.status()`要详细显示）。

所有`sh.status()`显示的信息都是从您的*config*数据库中收集的。

## 查看配置信息

您集群的所有配置信息都保存在配置服务器上*config*数据库的集合中。Shell 提供了几个辅助工具，以更可读的方式显示这些信息。但是，您始终可以直接查询*config*数据库以获取有关集群的元数据。

###### 警告

永远不要直接连接到配置服务器，因为您不希望意外更改或删除配置服务器数据。相反，连接到*mongos*进程，并使用*config*数据库查看其数据，就像您对待任何其他数据库一样：

```
> use config
```

如果您通过*mongos*（而不是直接连接到配置服务器）操作配置数据，*mongos* 将确保所有配置服务器保持同步，并防止意外删除*config*数据库等各种危险操作。

通常情况下，您不应直接更改*config*数据库中的任何数据（在下面的部分中会有例外）。如果您进行了任何更改，通常需要重新启动所有的*mongos*服务器才能看到其效果。

*config*数据库中有几个集合。本节介绍了每个集合包含的内容及其用途。

### config.shards

*shards*集合跟踪集群中的所有分片。*shards*集合中典型的文档可能如下所示：

```
> db.shards.find()
{ "_id" : "shard01", 
  "host" : "shard01/localhost:27018,localhost:27019,localhost:27020", 
  "state" : 1 }
{ "_id" : "shard02", 
  "host" : "shard02/localhost:27021,localhost:27022,localhost:27023", 
  "state" : 1 }
{ "_id" : "shard03", 
  "host" : "shard03/localhost:27024,localhost:27025,localhost:27026", 
  "state" : 1 }
```

分片的`"_id"`是从副本集名称中提取的，因此集群中的每个副本集必须具有唯一的名称。

当更新复制集配置（例如，添加或移除成员）时，`"host"` 字段将自动更新。

### config.databases

*databases* 集合跟踪集群了解的所有数据库，无论是分片的还是非分片的：

```
> db.databases.find()
{ "_id" : "video", "primary" : "shard02", "partitioned" : true, 
  "version" : { "uuid" : UUID("3d83d8b8-9260-4a6f-8d28-c3732d40d961"), 
  "lastMod" : 1 } }
```

如果在数据库上运行了 `enableSharding`，`"partitioned"` 将为 `true`。`"primary"` 是数据库的“主基地”。默认情况下，该数据库中的所有新集合都将在主分片上创建。

### config.collections

*collections* 集合跟踪所有分片集合（未分片的集合未显示）。典型文档看起来像这样：

```
> db.collections.find().pretty()
{
    "_id" : "config.system.sessions",
    "lastmodEpoch" : ObjectId("5bdf53122ad9c6907510c22d"),
    "lastmod" : ISODate("1970-02-19T17:02:47.296Z"),
    "dropped" : false,
    "key" : {
        "_id" : 1
    },
    "unique" : false,
    "uuid" : UUID("7584e4cd-fac4-4305-a9d4-bd73e93621bf")
}
{
    "_id" : "video.movies",
    "lastmodEpoch" : ObjectId("5bdf72c021b6e3be02fabe0c"),
    "lastmod" : ISODate("1970-02-19T17:02:47.305Z"),
    "dropped" : false,
    "key" : {
        "imdbId" : "hashed"
    },
    "unique" : false,
    "uuid" : UUID("e6580ffa-fcd3-418f-aa1a-0dfb71bc1c41")
}
```

重要字段是：

`"_id"`

集合的命名空间。

`"key"`

分片键。在这种情况下，它是对 `"imdbId"` 进行散列分片键。

`"unique"`

表示分片键不是唯一索引。默认情况下，分片键不是唯一的。

### config.chunks

*chunks* 集合中记录了所有集合中每个 chunk 的记录。*chunks* 集合中的典型文档看起来像这样：

```
> db.chunks.find().skip(1).limit(1).pretty()
{
    "_id" : "video.movies-imdbId_MinKey",
    "lastmod" : Timestamp(2, 0),
    "lastmodEpoch" : ObjectId("5bdf72c021b6e3be02fabe0c"),
    "ns" : "video.movies",
    "min" : {
        "imdbId" : { "$minKey" : 1 }
    },
    "max" : {
        "imdbId" : NumberLong("-7262221363006655132")
    },
    "shard" : "shard01",
    "history" : [
        {
            "validAfter" : Timestamp(1541370579, 3096),
            "shard" : "shard01"
        }
    ]
}
```

最有用的字段是：

`"_id"`

chunk 的唯一标识符。通常是命名空间、分片键和较低 chunk 边界。

`"ns"`

此 chunk 所属的集合。

`"min"`

chunk 的范围中的最小值（包括在内）。

`"max"`

所有 chunk 中的值均小于此值。

`"shard"`

chunk 所在的分片。

`"lastmod"` 字段跟踪 chunk 的版本信息。例如，如果 chunk `"video.movies-imdbId_MinKey"` 被拆分为两个 chunk，我们需要一种方法来区分新的、更小的 `"video.movies-imdbId_MinKey"` chunk 和它们之前作为单个 chunk 的版本。因此，`Timestamp` 值的第一个组件反映了 chunk 迁移到新分片的次数。该值的第二个组件反映了拆分次数。`"lastmodEpoch"` 字段指定了集合的创建时期。它用于区分在集合被删除并立即重新创建的情况下对同一集合名称的请求。

`sh.status()` 使用 *config.chunks* 集合来收集大部分信息。

### config.changelog

*changelog* 集合对于跟踪集群正在执行的操作非常有用，因为它记录了所有已发生的拆分和迁移。

拆分记录在如下所示的文档中：

```
> db.changelog.find({what: "split"}).pretty()
{
    "_id" : "router1-2018-11-05T09:58:58.915-0500-5be05ab2f8c192ed922ffbe7",
    "server" : "bob",
    "clientAddr" : "127.0.0.1:64621",
    "time" : ISODate("2018-11-05T14:58:58.915Z"),
    "what" : "split",
    "ns" : "video.movies",
    "details" : {
        "before" : {
            "min" : {
                "imdbId" : NumberLong("2484315172280977547")
            },
            "max" : {
                "imdbId" : NumberLong("4436141279217488250")
            },
            "lastmod" : Timestamp(9, 1),
            "lastmodEpoch" : ObjectId("5bdf72c021b6e3be02fabe0c")
        },
        "left" : {
            "min" : {
                "imdbId" : NumberLong("2484315172280977547")
            },
            "max" : {
                "imdbId" : NumberLong("3459137475094092005")
            },
            "lastmod" : Timestamp(9, 2),
            "lastmodEpoch" : ObjectId("5bdf72c021b6e3be02fabe0c")
        },
        "right" : {
            "min" : {
                "imdbId" : NumberLong("3459137475094092005")
            },
            "max" : {
                "imdbId" : NumberLong("4436141279217488250")
            },
            "lastmod" : Timestamp(9, 3),
            "lastmodEpoch" : ObjectId("5bdf72c021b6e3be02fabe0c")
        }
    }
}
```

`"details"` 字段提供了关于原始文档的信息以及它是如何分割的。

此输出显示了集合第一个 chunk 拆分的情况。请注意，每个新 chunk 的 `"lastmod"` 的第二个组件已更新，因此其值分别为 `Timestamp(9, 2)` 和 `Timestamp(9, 3)`。

迁移稍微复杂，实际上创建了四个单独的变更日志文档：一个标记迁移开始，一个为“源”分片，一个为“目标”分片，以及一个在迁移最终化时发生的提交。中间两个文档很有意思，因为它们详细说明了过程中每个步骤所花费的时间。这可以让你了解是磁盘、网络还是其他原因导致迁移瓶颈。

例如，“源”分片创建的文档如下：

```
> db.changelog.findOne({what: "moveChunk.to"})
{
    "_id" : "router1-2018-11-04T17:29:39.702-0500-5bdf72d32ad9c69075112f08",
    "server" : "bob",
    "clientAddr" : "",
    "time" : ISODate("2018-11-04T22:29:39.702Z"),
    "what" : "moveChunk.to",
    "ns" : "video.movies",
    "details" : {
        "min" : {
            "imdbId" : { "$minKey" : 1 }
        },
        "max" : {
            "imdbId" : NumberLong("-7262221363006655132")
        },
        "step 1 of 6" : 965,
        "step 2 of 6" : 608,
        "step 3 of 6" : 15424,
        "step 4 of 6" : 0,
        "step 5 of 6" : 72,
        "step 6 of 6" : 258,
        "note" : "success"
    }
}
```

`"details"`中列出的每个步骤都有时间限制，并且`"step*`N`* of *`N`*"`消息显示了每个步骤所花费的时间，以毫秒为单位。

当“源”分片从*mongos*接收到`moveChunk`命令时，它：

1.  检查命令参数。

1.  确认配置服务器可以为迁移获取分布式锁。

1.  尝试联系“目标”分片。

1.  复制数据。这被称为“关键部分”，并记录在日志中。

1.  与“目标”分片和配置服务器协调以确认迁移。

请注意，“目标”和“源”分片必须从“第 4 步到第 6 步”开始密切通信：分片直接与彼此和配置服务器通信以执行迁移。如果“源”服务器在最后几个步骤中的网络连接不稳定，则可能会导致无法撤消迁移并且无法继续进行迁移的状态。在这种情况下，*mongod*将关闭。

“目标”分片的变更日志文档与“源”分片类似，但步骤略有不同。它看起来是这样的：

```
> db.changelog.find({what: "moveChunk.from", "details.max.imdbId": 
  NumberLong("-7262221363006655132")}).pretty()
{
    "_id" : "router1-2018-11-04T17:29:39.753-0500-5bdf72d321b6e3be02fabf0b",
    "server" : "bob",
    "clientAddr" : "127.0.0.1:64743",
    "time" : ISODate("2018-11-04T22:29:39.753Z"),
    "what" : "moveChunk.from",
    "ns" : "video.movies",
    "details" : {
        "min" : {
            "imdbId" : { "$minKey" : 1 }
        },
        "max" : {
            "imdbId" : NumberLong("-7262221363006655132")
        },
        "step 1 of 6" : 0,
        "step 2 of 6" : 4,
        "step 3 of 6" : 191,
        "step 4 of 6" : 17000,
        "step 5 of 6" : 341,
        "step 6 of 6" : 39,
        "to" : "shard01",
        "from" : "shard02",
        "note" : "success"
    }
}
```

当“目标”分片从“源”分片接收到命令时，它：

1.  迁移索引。如果该分片以前从未持有过迁移集合的分块，它需要知道哪些字段被索引。如果这不是首次将该集合的分块移动到该分片，则应该是一个无操作。

1.  删除分块范围内的任何现有数据。可能会有来自失败的迁移或恢复过程的数据残留，我们不希望干扰当前数据。

1.  将分块中的所有文档复制到“目标”分片。

1.  在复制期间重新执行任何操作（在“目标”分片上）。

1.  等待“目标”分片将新迁移的数据复制到大多数服务器。

1.  通过更改分块的元数据来提交迁移，表示它存储在“目标”分片上。

### config.settings

此集合包含代表当前平衡器设置和分块大小的文档。通过更改此集合中的文档，您可以启用或禁用平衡器，或更改分块大小。请注意，您应该始终连接到*mongos*，而不是直接连接到配置服务器，以更改此集合中的值。

# 跟踪网络连接

集群组件之间有许多连接。本节涵盖了一些关于分片的特定信息（更多信息请参见第二十四章有关网络的内容）。

## 获取连接统计信息

命令 `connPoolStats` 返回有关当前数据库实例与分片集群或副本集中其他成员之间的开放传出连接的信息。

为了避免干扰任何正在运行的操作，`connPoolStats` 不会锁定任何内容。因此，随着 `connPoolStats` 收集信息，计数可能会略有变化，导致主机和池连接计数之间存在轻微差异：

```
> db.adminCommand({"connPoolStats": 1})
{
    "numClientConnections" : 10,
    "numAScopedConnections" : 0,
    "totalInUse" : 0,
    "totalAvailable" : 13,
    "totalCreated" : 86,
    "totalRefreshing" : 0,
    "pools" : {
        "NetworkInterfaceTL-TaskExecutorPool-0" : {
            "poolInUse" : 0,
            "poolAvailable" : 2,
            "poolCreated" : 2,
            "poolRefreshing" : 0,
            "localhost:27027" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 1,
                "refreshing" : 0
            },
            "localhost:27019" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 1,
                "refreshing" : 0
            }
        },
        "NetworkInterfaceTL-ShardRegistry" : {
            "poolInUse" : 0,
            "poolAvailable" : 1,
            "poolCreated" : 13,
            "poolRefreshing" : 0,
            "localhost:27027" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 13,
                "refreshing" : 0
            }
        },
        "global" : {
            "poolInUse" : 0,
            "poolAvailable" : 10,
            "poolCreated" : 71,
            "poolRefreshing" : 0,
            "localhost:27026" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 8,
                "refreshing" : 0
            },
            "localhost:27027" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 1,
                "refreshing" : 0
            },
            "localhost:27023" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 7,
                "refreshing" : 0
            },
            "localhost:27024" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 6,
                "refreshing" : 0
            },
            "localhost:27022" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 9,
                "refreshing" : 0
            },
            "localhost:27019" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 8,
                "refreshing" : 0
            },
            "localhost:27021" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 8,
                "refreshing" : 0
            },
            "localhost:27025" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 9,
                "refreshing" : 0
            },
            "localhost:27020" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 8,
                "refreshing" : 0
            },
            "localhost:27018" : {
                "inUse" : 0,
                "available" : 1,
                "created" : 7,
                "refreshing" : 0
            }
        }
    },
    "hosts" : {
        "localhost:27026" : {
            "inUse" : 0,
            "available" : 1,
            "created" : 8,
            "refreshing" : 0
        },
        "localhost:27027" : {
            "inUse" : 0,
            "available" : 3,
            "created" : 15,
            "refreshing" : 0
        },
        "localhost:27023" : {
            "inUse" : 0,
            "available" : 1,
            "created" : 7,
            "refreshing" : 0
        },
        "localhost:27024" : {
            "inUse" : 0,
            "available" : 1,
            "created" : 6,
            "refreshing" : 0
        },
        "localhost:27022" : {
            "inUse" : 0,
            "available" : 1,
            "created" : 9,
            "refreshing" : 0
        },
        "localhost:27019" : {
            "inUse" : 0,
            "available" : 2,
            "created" : 9,
            "refreshing" : 0
        },
        "localhost:27021" : {
            "inUse" : 0,
            "available" : 1,
            "created" : 8,
            "refreshing" : 0
        },
        "localhost:27025" : {
            "inUse" : 0,
            "available" : 1,
            "created" : 9,
            "refreshing" : 0
        },
        "localhost:27020" : {
            "inUse" : 0,
            "available" : 1,
            "created" : 8,
            "refreshing" : 0
        },
        "localhost:27018" : {
            "inUse" : 0,
            "available" : 1,
            "created" : 7,
            "refreshing" : 0
        }
    },
    "replicaSets" : {
        "shard02" : {
            "hosts" : [
                {
                    "addr" : "localhost:27021",
                    "ok" : true,
                    "ismaster" : true,
                    "hidden" : false,
                    "secondary" : false,
                    "pingTimeMillis" : 0
                },
                {
                    "addr" : "localhost:27022",
                    "ok" : true,
                    "ismaster" : false,
                    "hidden" : false,
                    "secondary" : true,
                    "pingTimeMillis" : 0
                },
                {
                    "addr" : "localhost:27023",
                    "ok" : true,
                    "ismaster" : false,
                    "hidden" : false,
                    "secondary" : true,
                    "pingTimeMillis" : 0
                }
            ]
        },
        "shard03" : {
            "hosts" : [
                {
                    "addr" : "localhost:27024",
                    "ok" : true,
                    "ismaster" : false,
                    "hidden" : false,
                    "secondary" : true,
                    "pingTimeMillis" : 0
                },
                {
                    "addr" : "localhost:27025",
                    "ok" : true,
                    "ismaster" : true,
                    "hidden" : false,
                    "secondary" : false,
                    "pingTimeMillis" : 0
                },
                {
                    "addr" : "localhost:27026",
                    "ok" : true,
                    "ismaster" : false,
                    "hidden" : false,
                    "secondary" : true,
                    "pingTimeMillis" : 0
                }
            ]
        },
        "configRepl" : {
            "hosts" : [
                {
                    "addr" : "localhost:27027",
                    "ok" : true,
                    "ismaster" : true,
                    "hidden" : false,
                    "secondary" : false,
                    "pingTimeMillis" : 0
                }
            ]
        },
        "shard01" : {
            "hosts" : [
                {
                    "addr" : "localhost:27018",
                    "ok" : true,
                    "ismaster" : false,
                    "hidden" : false,
                    "secondary" : true,
                    "pingTimeMillis" : 0
                },
                {
                    "addr" : "localhost:27019",
                    "ok" : true,
                    "ismaster" : true,
                    "hidden" : false,
                    "secondary" : false,
                    "pingTimeMillis" : 0
                },
                {
                    "addr" : "localhost:27020",
                    "ok" : true,
                    "ismaster" : false,
                    "hidden" : false,
                    "secondary" : true,
                    "pingTimeMillis" : 0
                }
            ]
        }
    },
    "ok" : 1,
    "operationTime" : Timestamp(1541440424, 1),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1541440424, 1),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
```

在此输出中：

+   `"totalAvailable"` 显示了当前 *mongod*/*mongos* 实例与分片集群或副本集中其他成员之间可用的所有传出连接的总数。

+   `"totalCreated"` 报告了当前 *mongod*/*mongos* 实例与分片集群或副本集中其他成员之间迄今为止创建的所有传出连接的总数。

+   `"totalInUse"` 提供了当前 *mongod*/*mongos* 实例与分片集群或副本集中其他成员之间正在使用的所有传出连接的总数。

+   `"totalRefreshing"` 显示了当前 *mongod*/*mongos* 实例与分片集群或副本集中其他成员之间当前正在刷新的所有传出连接的总数。

+   `"numClientConnections"` 标识了当前 *mongod*/*mongos* 实例与分片集群或副本集中其他成员之间活动和存储的传出同步连接的数量。这些连接是 `"totalAvailable"`、`"totalCreated"` 和 `"totalInUse"` 报告的连接的子集。

+   `"numAScopedConnection"` 报告了当前 *mongod*/*mongos* 实例与分片集群或副本集中其他成员之间活动和存储的传出作用域同步连接的数量。这些连接是 `"totalAvailable"`、`"totalCreated"` 和 `"totalInUse"` 报告的连接的子集。

+   `"pools"` 显示了按连接池分组的连接统计信息（使用中/可用/已创建/正在刷新）。*mongod* 或 *mongos* 有两组不同的传出连接池：

    +   基于 DBClient 的池（“写入路径”，在 `"pools"` 文档中由字段名 `"global"` 标识）

    +   基于 NetworkInterfaceTL 的池（“读取路径”）

+   `"hosts"` 显示了按主机分组的连接统计信息（使用中/可用/已创建/正在刷新）。它报告了当前 *mongod*/*mongos* 实例与分片集群或副本集中每个成员之间的连接情况。

在 `connPoolStats` 的输出中，你可能会看到与其他分片的连接。这些指示分片正在连接到其他分片以迁移数据。一个分片的主节点将直接连接到另一个分片的主节点，并“吸取”其数据。

当进行迁移时，分片会设置一个`ReplicaSetMonitor`（监视复制集健康状况的进程）来跟踪迁移另一侧的分片的健康状况。*mongod*永远不会销毁此监视器，因此您可能会在一个复制集的日志中看到关于另一个复制集成员的消息。这是完全正常的，不应对您的应用程序产生任何影响。

## 限制连接数

当客户端连接到*mongos*时，*mongos*会创建一个连接到至少一个分片，以传递客户端的请求。因此，每个连接到*mongos*的客户端连接至少会导致*mongos*到分片的一个传出连接。

如果您有许多*mongos*进程，它们可能会创建比您的分片可以处理的连接更多：默认情况下，*mongos*将接受最多 65536 个连接（与*mongod*相同），因此如果您有 5 个每个有 10000 个客户端连接的*mongos*进程，则可能会尝试创建 50000 个连接到分片！

为了防止这种情况发生，您可以在命令行配置中使用`--maxConns`选项限制*mongos*可以创建的连接数。可以使用以下公式来计算单个*mongos*可以处理的最大连接数：

+   *maxConns* = *maxConnsPrimary* − （每个复制集成员数 × 3） −

+   （其他 × 3）/ *numMongosProcesses*

分解此公式的各部分：

maxConnsPrimary

主节点上的最大连接数，通常设置为 20000，以避免*mongos*对分片的连接过多。

（每个复制集成员数 × 3）

主要的创建了到每个从节点的连接，并且每个从节点创建了两个到主节点的连接，总共是三个连接。

（其他 x 3）

其他是可能连接到您的*mongod*的其他杂项进程，例如监控或备份代理、直接的 shell 连接（用于管理）或者连接到其他分片进行迁移。

numMongosProcesses

在分片集群中的*mongos*总数。

请注意，`--maxConns`仅防止*mongos*创建超过此数量的连接。当达到此限制时，它不会执行任何特别有用的操作：它将简单地阻塞请求，等待连接“释放”。因此，您必须防止您的应用程序使用这么多连接，特别是随着*mongos*进程数量的增加。

当 MongoDB 实例正常退出时，它会在停止之前关闭所有连接。连接到它的成员将立即在这些连接上收到套接字错误并能够刷新它们。但是，如果 MongoDB 实例突然因为断电、崩溃或网络问题而离线，它可能不会干净地关闭所有套接字。在这种情况下，集群中的其他服务器可能会认为它们的连接是健康的，直到它们尝试在其上执行操作。此时，它们将会收到一个错误并刷新连接（如果成员此时已经恢复在线）。

当只有少量连接时，这是一个快速的过程。然而，当需要逐个刷新成千上万的连接时，可能会出现很多错误，因为必须尝试每个与停机成员的连接，确定它们是否无效，并重新建立连接。除了重新启动陷入重连风暴的进程外，没有特别好的方法来防止这种情况。

# 服务器管理

随着集群的增长，您将需要增加容量或更改配置。本节介绍如何在集群中添加和删除服务器。

## 添加服务器

您可以随时添加新的 *mongos* 进程。确保它们的 `--configdb` 选项指定了正确的配置服务器集，并且它们应立即可供客户端连接。

要添加新的分片，请使用 `addShard` 命令，如 第十五章 所示。

## 更改分片中的服务器

在使用分片集群时，您可能希望更改各个分片中的服务器。要更改分片的成员资格，请直接连接到分片的主服务器（而不是通过 *mongos*）并发出副本集重新配置命令。集群配置将捕捉更改并自动更新 *config.shards*。不要手动修改 *config.shards*。

唯一的例外是，如果您以独立服务器作为分片启动集群。

### 将分片从独立服务器更改为副本集

这样做的最简单方法是添加一个新的空副本集分片，然后删除独立服务器分片（如下一节所述）。迁移将负责将您的数据移动到新的分片。

## 移除分片

通常情况下，不应从集群中删除分片。如果您经常添加和删除分片，会给系统带来比必要的更大压力。如果添加了太多的分片，最好让系统逐步增长，而不是将其删除然后稍后再添加。但是如果有必要，您可以删除分片。

首先确保均衡器已启动。均衡器将负责在过程中移动您想要移除的分片上的所有数据到其他分片，这个过程称为 *draining*。要开始 *draining*，运行 `removeShard` 命令。`removeShard` 命令需要分片的名称，并将该分片上的所有数据块移动到其他分片上：

```
> db.adminCommand({"removeShard" : "shard03"})
{
    "msg" : "draining started successfully",
    "state" : "started",
    "shard" : "shard03",
    "note" : "you need to drop or movePrimary these databases",
    "dbsToMove" : [ ],
    "ok" : 1,
    "operationTime" : Timestamp(1541450091, 2),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1541450091, 2),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
```

如果有大量或大块要移动，*draining* 可能需要很长时间。如果有巨块 (见 “巨块”)，可能需要临时增加块大小以允许 *draining* 移动它们。

如果您想了解移动了多少数据，请再次运行 `removeShard` 以获取当前状态：

```
> db.adminCommand({"removeShard" : "shard02"})
{
    "msg" : "draining ongoing",
    "state" : "ongoing",
    "remaining" : {
        "chunks" : NumberLong(3),
        "dbs" : NumberLong(0)
    },
    "note" : "you need to drop or movePrimary these databases",
    "dbsToMove" : [ 
          "video"
       ],
    "ok" : 1,
    "operationTime" : Timestamp(1541450139, 1),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1541450139, 1),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
```

你可以随时运行 `removeShard` 多次。

可能需要分割块以便移动，因此在 *drain* 过程中您可能会看到系统中的块数量增加。例如，假设我们有一个包含以下块分布的五分片集群：

```
test-rs0    10
test-rs1    10
test-rs2    10
test-rs3    11
test-rs4    11
```

此集群总共有 52 个数据块。如果我们移除 *test-rs3*，可能会得到以下结果：

```
test-rs0    15
test-rs1    15
test-rs2    15
test-rs4    15
```

现在集群有 60 个数据块，其中 18 个来自分片 *test-rs3*（开始时有 11 个，另外 7 个来自排空分裂创建的数据块）。

一旦所有数据块都已移动，如果仍然有数据库将移除的分片作为它们的主分片，那么在移除分片之前，你需要先移除它们。分片集群中的每个数据库都有一个主分片。如果你要移除的分片也是集群中某个数据库的主分片，`removeShard` 将在 `"dbsToMove"` 字段中列出该数据库。要完成移除分片的过程，你必须在将所有数据迁移到其他分片后，移动数据库或者删除数据库，删除相关的数据文件。`removeShard` 的输出将类似于：

```
> db.adminCommand({"removeShard" : "shard02"})
{
    "msg" : "draining ongoing",
    "state" : "ongoing",
    "remaining" : {
        "chunks" : NumberLong(3),
        "dbs" : NumberLong(0)
    },
    "note" : "you need to drop or movePrimary these databases",
    "dbsToMove" : [ 
          "video"
       ],
    "ok" : 1,
    "operationTime" : Timestamp(1541450139, 1),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1541450139, 1),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
```

要完成移除操作，请使用 `movePrimary` 命令移动列出的数据库：

```
> db.adminCommand({"movePrimary" : "video", "to" : "shard01"})
{
    "ok" : 1,
    "operationTime" : Timestamp(1541450554, 12),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1541450554, 12),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
```

一旦你完成了这个步骤，再运行 `removeShard` 一次：

```
> db.adminCommand({"removeShard" : "shard02"})
{
    "msg" : "removeshard completed successfully",
    "state" : "completed",
    "shard" : "shard03",
    "ok" : 1,
    "operationTime" : Timestamp(1541450619, 2),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1541450619, 2),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
```

这不是绝对必要的，但它确认你已完成了这个过程。如果没有数据库将这个分片作为它们的主分片，那么当所有数据块都迁移出分片时，你将会得到这个响应。

###### 警告

一旦开始排空一个分片，就没有内置的方法可以停止它。

# 数据平衡

通常情况下，MongoDB 会自动处理数据的平衡。本节涵盖如何启用和禁用此自动平衡以及如何干预平衡过程。

## 负载均衡器

关闭负载均衡器是几乎所有管理活动的先决条件。有一个 shell 辅助工具可以使这一过程更容易：

```
> sh.setBalancerState(false)
{
    "ok" : 1,
    "operationTime" : Timestamp(1541450923, 2),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1541450923, 2),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
```

当负载均衡器关闭时，新的负载均衡轮不会开始，但关闭它不会立即停止正在进行的负载均衡轮——迁移通常不能立即停止。因此，你应该检查 *config.locks* 集合，以查看负载均衡轮是否仍在进行中：

```
> db.locks.find({"_id" : "balancer"})["state"]
0
```

`0` 意味着负载均衡器已关闭。

平衡会给系统带来负载：目标分片必须查询源分片中所有文档的数据块并插入它们，然后源分片必须删除它们。特别是在两种情况下，迁移可能会导致性能问题：

1.  使用热点分片键将强制进行持续的迁移（因为所有新的数据块将创建在热点上）。你的系统必须有能力处理来自热点分片的数据流。

1.  添加一个新的分片将触发大量的迁移，因为负载均衡器试图将其填充。

如果发现迁移影响了应用程序的性能，可以在 *config.settings* 集合中安排一个平衡窗口的时间。运行以下更新以仅允许在下午 1 点到下午 4 点之间进行平衡。首先确保负载均衡器已开启，然后安排窗口：

```
> sh.setBalancerState( true )
{
    "ok" : 1,
    "operationTime" : Timestamp(1541451846, 4),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1541451846, 4),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
> db.settings.update(
   { _id: "balancer" },
   { $set: { activeWindow : { start : "13:00", stop : "16:00" } } },
   { upsert: true }
)
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

如果设置了平衡窗口，请密切监视以确保 *mongos* 实际上可以在你分配给它的时间内保持集群的平衡。

如果计划结合手动均衡和自动均衡器，必须小心，因为自动均衡器总是基于集合的当前状态确定移动内容，并不考虑集合的历史。例如，假设您有 *shardA* 和 *shardB*，每个分片持有 500 个块。*shardA* 正在进行大量写入，因此您关闭了均衡器并将最活跃的 30 个块移动到 *shardB*。如果此时重新启动均衡器，它将立即从 *shardB* 移回 30 个块（可能是不同的 30 个块），以平衡块计数。

为了防止这种情况发生，在启动均衡器之前，将 30 个静止的块从 *shardB* 移动到 *shardA*。这样分片之间就不会存在不平衡，均衡器会乐意保持当前状态。或者，您可以对 *shardA* 的块执行 30 次分割，以平衡块计数。

请注意，均衡器仅使用块数作为指标，不考虑数据大小。移动一个块称为迁移，是 MongoDB 在集群中平衡数据的方法。因此，一个具有少量大块的分片可能成为从具有许多小块（但数据量较小）的分片迁移的目标。

## 修改块大小

一个块中可以包含从零到数百万个文档。一般来说，块越大，迁移到另一个分片的时间就越长。在第十四章中，我们使用了 1 MB 的块大小，这样我们可以轻松快速地观察块的移动。但是，这在实时系统中通常是不切实际的；MongoDB 将不必要地工作来保持分片大小接近几兆字节。默认的块大小是 64 MB，一般情况下提供了迁移和移动的良好平衡。

有时您可能发现 64 MB 的块大小迁移时间过长。为了加快迁移速度，您可以减小块大小。为此，通过 shell 连接到 *mongos* 并更新 *config.settings* 集合：

```
> db.settings.findOne()
{
    "_id" : "chunksize", 
    "value" : 64 
}
> db.settings.save({"_id" : "chunksize", "value" : 32})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

上述更新将把您的块大小更改为 32 MB。然而，现有的块不会立即更改；自动分割仅在插入或更新时发生。因此，如果您减小块大小，可能需要一些时间让所有块分割到新的大小。

分割无法撤销。如果增加块大小，现有的块只能通过插入或更新来增长，直到达到新的大小。块大小的允许范围是 1 到 1,024 MB，包括两端。

这是一个集群范围的设置：它影响所有集合和数据库。因此，如果您需要一个集合的小块大小和另一个集合的大块大小，则可能需要在两个理想值之间进行妥协（或将集合放入不同的集群）。

###### 提示

如果 MongoDB 迁移过多或者您的文档过大，您可能需要增加块大小。

## 移动块

正如前面提到的，块中的所有数据存储在特定的分片上。如果该分片的块比其他分片的多，MongoDB 将会将一些块从该分片移动出去。

使用`moveChunk` shell 辅助程序可以手动移动块：

```
> sh.moveChunk("video.movies", {imdbId: 500000}, "shard02") 
{ "millis" : 4079, "ok" : 1 }
```

这将移动包含 `"imdbId"` 为 `500000` 的文档的块到名为 *shard02* 的分片。您必须使用分片键（在这种情况下为 `"imdbId"`）来查找要移动的块。通常，指定一个块最简单的方法是使用其下界，虽然块中的任何值都可以工作（上界不行，因为它实际上不在块中）。该命令在返回之前将移动块，因此可能需要一段时间才能运行。如果运行时间较长，日志是查看其操作的最佳位置。

如果一个块大于最大块大小，*mongos* 将拒绝移动它：

```
> sh.moveChunk("video.movies", {imdbId: NumberLong("8345072417171006784")}, 
  "shard02")
{
    "cause" : {
        "chunkTooBig" : true,
        "estimatedChunkSize" : 2214960,
        "ok" : 0,
        "errmsg" : "chunk too big to move"
    },
    "ok" : 0,
    "errmsg" : "move failed"
}
```

在这种情况下，您必须在移动之前手动分割块，使用`splitAt`命令：

```
> db.chunks.find({ns: "video.movies", "min.imdbId": 
  NumberLong("6386258634539951337")}).pretty()
{
    "_id" : "video.movies-imdbId_6386258634539951337",
    "ns" : "video.movies",
    "min" : {
        "imdbId" : NumberLong("6386258634539951337")
    },
    "max" : {
        "imdbId" : NumberLong("8345072417171006784")
    },
    "shard" : "shard02",
    "lastmod" : Timestamp(1, 9),
    "lastmodEpoch" : ObjectId("5bdf72c021b6e3be02fabe0c"),
    "history" : [
        {
            "validAfter" : Timestamp(1541370559, 4),
            "shard" : "shard02"
        }
    ]
}
> sh.splitAt("video.movies", {"imdbId": 
  NumberLong("7000000000000000000")})
{
    "ok" : 1,
    "operationTime" : Timestamp(1541453304, 1),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1541453306, 5),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
> db.chunks.find({ns: "video.movies", "min.imdbId": 
  NumberLong("6386258634539951337")}).pretty()
{
    "_id" : "video.movies-imdbId_6386258634539951337",
    "lastmod" : Timestamp(15, 2),
    "lastmodEpoch" : ObjectId("5bdf72c021b6e3be02fabe0c"),
    "ns" : "video.movies",
    "min" : {
        "imdbId" : NumberLong("6386258634539951337")
    },
    "max" : {
        "imdbId" : NumberLong("7000000000000000000")
    },
    "shard" : "shard02",
    "history" : [
        {
            "validAfter" : Timestamp(1541370559, 4),
            "shard" : "shard02"
        }
    ]
}
```

一旦块被分割成更小的片段，它就应该可以移动了。或者，您可以提高最大块大小然后移动它，但应该尽可能地拆分大块。然而，有时候块无法分割——我们将在接下来看看这种情况。^(1)

## 巨型块

假设您选择 `"date"` 字段作为分片键。此集合中的 `"date"` 字段是一个看起来像 ``"*`year`*/*`month`*/*`day`*"` 的字符串，这意味着 *mongos* 每天最多可以创建一个块。这样做一段时间很好，直到您的应用程序突然爆红，并且某一天的流量比平常高出一千倍。

这一天的块将比其他任何一天的块都要大得多，但也完全无法分割，因为每个文档对于分片键都有相同的值。

一旦一个块大于在 *config.settings* 中设置的最大块大小，*balancer* 将不允许移动该块。这些不可分割、不可移动的块称为巨型块，处理起来很不方便。

让我们举个例子。假设你有三个分片，*shard1*、*shard2* 和 *shard3*。如果你使用“升序分片键”中描述的热点分片键模式，那么所有的写操作都将会进入一个分片——比如说 *shard1*。分片的主 *mongod* 将请求 *balancer* 将每个新的顶部块均匀地移动到其他分片，但 *balancer* 只能移动非巨型块，因此它将所有小块从热点分片迁移出去。

现在，所有的分片将会有大致相同数量的块，但 *shard2* 和 *shard3* 的所有块大小都将小于 64 MB。如果创建巨型块，*shard1* 的块会越来越多超过 64 MB。因此，*shard1* 将比其他两个分片更快填满，即使三者之间的块数量完全平衡。

因此，您有巨大块问题的一个指标是一个分片的大小增长速度比其他分片快得多。您还可以查看`sh.status()`的输出，以查看是否有巨大块，它们将用`jumbo`属性标记：

```
> sh.status()
...
    { "x" : -7 } -->> { "x" : 5 } on : shard0001 
    { "x" : 5 } -->> { "x" : 6 } on : shard0001 jumbo
    { "x" : 6 } -->> { "x" : 7 } on : shard0001 jumbo
    { "x" : 7 } -->> { "x" : 339 } on : shard0001
...
```

您可以使用`dataSize`命令检查块大小。首先，使用*config.chunks*集合找到块范围：

```
> use config
> var chunks = db.chunks.find({"ns" : "acme.analytics"}).toArray()
```

然后使用这些块范围找到可能的巨大块：

```
> use <dbName>
> db.runCommand({"dataSize" : "<dbName.collName>",
... "keyPattern" : {"date" : 1}, // shard key
... "min" : chunks[0].min, 
... "max" : chunks[0].max})
{
    "size" : 33567917,
    "numObjects" : 108942,
    "millis" : 634,
    "ok" : 1,
    "operationTime" : Timestamp(1541455552, 10),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1541455552, 10),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
```

不过要小心，`dataSize`命令确实需要扫描块的数据来确定其大小。如果可能的话，通过使用对数据的了解缩小搜索范围：是否在某个特定日期创建了巨大块？例如，如果 7 月 1 日是一个非常繁忙的一天，请查找具有该日期的块。

###### 提示

如果你正在使用 GridFS 并且按`"files_id"`分片，你可以查看*fs.files*集合来找到文件的大小。

### 分布巨大块

要解决由巨大块导致失衡的集群问题，必须将它们均匀分布在分片之间。

这是一个复杂的手动过程，但不应造成任何停机（可能会导致速度变慢，因为你将迁移大量数据）。在以下描述中，具有巨大块的分片称为“from”分片。将巨大块迁移到的分片称为“to”分片。请注意，您可能有多个希望迁移块的“from”分片。对于每个分片，重复以下步骤：

1.  关闭平衡器。在此过程中，您不希望平衡器试图“帮助”：

    ```
    > sh.setBalancerState(false)
    ```

1.  MongoDB 不允许您移动大于最大块大小的块，因此暂时增加块大小。记下您的原始块大小，然后将其更改为大型值，如`10000`。块大小以兆字节指定。

    ```
    > use config
    > db.settings.findOne({"_id" : "chunksize"})
    {
        "_id" : "chunksize", 
        "value" : 64
    }
    > db.settings.save({"_id" : "chunksize", "value" : 10000})
    ```

1.  使用`moveChunk`命令将巨大块从“from”分片移出。

1.  在“from”分片上运行`splitChunk`，直到它拥有大致相同数量的块作为“to”分片。

1.  将块大小设置回其原始值：

    ```
    > db.settings.save({"_id" : "chunksize", "value" : 64})
    ```

1.  启动平衡器：

    ```
    > sh.setBalancerState(true)
    ```

当再次启用平衡器时，它将再次无法移动巨大块；它们基本上是由它们的大小所固定在那里。

### 防止巨大块

随着存储数据量的增长，前面描述的手动过程变得难以维持。因此，如果您遇到巨大块问题，应优先采取措施防止它们形成。

要防止巨大块，请修改您的分片键以增加细粒度。您希望几乎每个文档的分片键具有唯一值，或者至少从不具有超过块大小值的数据。

例如，如果你之前使用的是年/月/日的键，可以快速通过添加小时、分钟和秒来提升精细度。同样地，如果你在粗粒度的日志级别上进行分片，可以在分片键中添加一个具有很高粒度的第二字段，例如 MD5 哈希或 UUID。这样，即使第一个字段对许多文档相同，你也可以随时分割一个块。

## 刷新配置

最后的提示是，有时*mongos*无法正确从配置服务器更新其配置。如果出现意外配置或*mongos*看起来过时或找不到已知存在的数据，可以使用`flushRouterConfig`命令手动清除所有缓存：

```
> db.adminCommand({"flushRouterConfig" : 1})
```

如果`flushRouterConfig`无法工作，重新启动所有的*mongos*或*mongod*进程可以清除任何缓存数据。

^(1) MongoDB 4.4 计划在`moveChunk`函数中添加一个新参数（`forceJumbo`），以及一个新的负载均衡器配置设置`attemptToBalanceJumboChunks`来解决超大块的问题。具体详情请参阅[这个描述工作的 JIRA 票](https://jira.mongodb.org/browse/SERVER-42273)。
