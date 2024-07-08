# 第十八章：查看应用程序的运行情况

一旦您的应用程序启动运行，如何知道它在做什么？ 本章介绍了如何查明 MongoDB 运行哪些查询、写入了多少数据以及有关实际运行情况的其他细节。 您将了解以下内容：

+   查找慢速操作并终止它们

+   获取和解释有关集合和数据库的统计信息

+   使用命令行工具来获取 MongoDB 正在执行的情况

# 查看当前操作

查找运行中的操作的简便方法是查看运行情况。 任何慢速操作更可能显示出来，并且可能运行时间更长。 虽然不能保证，但这是查找可能减缓应用程序的第一步。

要查看正在运行的操作，请使用 `db.currentOp()` 函数：

```
> db.currentOp()
{
  "inprog": [{
    "type" : "op",
    "host" : "eoinbrazil-laptop-osx:27017",
    "desc" : "conn3",
    "connectionId" : 3,
    "client" : "127.0.0.1:57181",
    "appName" : "MongoDB Shell",
    "clientMetadata" : {
        "application" : {
            "name" : "MongoDB Shell"
        },
        "driver" : {
            "name" : "MongoDB Internal Client",
            "version" : "4.2.0"
        },
        "os" : {
            "type" : "Darwin",
            "name" : "Mac OS X",
            "architecture" : "x86_64",
            "version" : "18.7.0"
        }
    },
    "active" : true,
    "currentOpTime" : "2019-09-03T23:25:46.380+0100",
    "opid" : 13594,
    "lsid" : {
        "id" : UUID("63b7df66-ca97-41f4-a245-eba825485147"),
        "uid" : BinData(0,"47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU=")
    },
    "secs_running" : NumberLong(0),
    "microsecs_running" : NumberLong(969),
    "op" : "insert",
    "ns" : "sample_mflix.items",
    "command" : {
        "insert" : "items",
        "ordered" : false,
        "lsid" : {
            "id" : UUID("63b7df66-ca97-41f4-a245-eba825485147")
        },
        "$readPreference" : {
            "mode" : "secondaryPreferred"
        },
        "$db" : "sample_mflix"
    },
    "numYields" : 0,
    "locks" : {
        "ParallelBatchWriterMode" : "r",
        "ReplicationStateTransition" : "w",
        "Global" : "w",
        "Database" : "w",
        "Collection" : "w"
    },
    "waitingForLock" : false,
    "lockStats" : {
        "ParallelBatchWriterMode" : {
            "acquireCount" : {
                "r" : NumberLong(4)
            }
        },
        "ReplicationStateTransition" : {
            "acquireCount" : {
                "w" : NumberLong(4)
            }
        },
        "Global" : {
            "acquireCount" : {
                "w" : NumberLong(4)
            }
        },
        "Database" : {
            "acquireCount" : {
                "w" : NumberLong(4)
            }
        },
        "Collection" : {
            "acquireCount" : {
                "w" : NumberLong(4)
            }
        },
        "Mutex" : {
            "acquireCount" : {
                "r" : NumberLong(196)
            }
        }
    },
    "waitingForFlowControl" : false,
    "flowControlStats" : {
        "acquireCount" : NumberLong(4)
    }
  }],
  "ok": 1
}
```

这显示数据库正在执行的操作列表。 以下是输出中一些较重要的字段：

`"opid"`

操作的唯一标识符。 您可以使用此编号来终止一个操作（参见 “终止操作”）。

`"active"`

此操作是否正在运行。 如果此字段为 `false`，则意味着操作已放弃或正在等待锁。

`"secs_running"`

此操作持续的秒数。 您可以使用此操作查找执行时间过长的查询。

`"microsecs_running"`

此操作持续的微秒数。 您可以使用此操作查找执行时间过长的查询。

`"op"`

操作类型。 一般为 `"query"`、`"insert"`、`"update"` 或 `"remove"`。 请注意，数据库命令将作为查询处理。

`"desc"`

客户端的标识符。 这可以与日志中的消息相关联。 我们示例中与连接相关的每条日志消息都将以 `[conn3]` 作为前缀，因此您可以使用它来在日志中搜索相关信息。

`"locks"`

此操作获取的锁类型的描述。

`"waitingForLock"`

此操作当前是否正在阻塞，等待获取锁。

`"numYields"`

此操作放弃其锁以允许其他操作继续的次数。 一般来说，任何搜索文档（查询、更新和删除）的操作都可以放弃。 只有在排队等待获取锁的其他操作时，操作才会放弃。 基本上，如果没有处于`"waitingForLock"`状态的操作，则当前操作将不会放弃。

`"lockstats.timeAcquiringMicros"`

操作需要获取所需锁的等待时间。

您可以将 `currentOp` 过滤为仅查找满足特定条件的操作，例如在特定命名空间上的操作或已运行了一定时间的操作。 您可以通过传递查询参数来过滤结果：

```
> db.currentOp(
   {
     "active" : true,
     "secs_running" : { "$gt" : 3 },
     "ns" : /^db1\./
   }
)
```

您可以在 `currentOp` 中查询任何字段，使用所有常规的查询操作符。

## 查找问题操作

`db.currentOp()` 的最常见用法是查找慢操作。你可以使用前面章节描述的过滤技术来查找所有执行时间超过一定阈值的查询，这可能暗示缺少索引或不正确的字段过滤。

有时人们会发现意外的查询在运行，通常是因为某个应用服务器运行了旧版或有 bug 的软件。`"client"` 字段可以帮助你追踪意外操作的来源。

## 终止操作

如果你找到一个想要停止的操作，你可以通过传递`db.killOp()`的`"opid"`来杀死它：

```
> db.killOp(123)
```

并非所有操作都可以被终止。一般来说，只有在操作让出时才能终止它们——所以更新、查找和删除都可以被终止，但通常无法终止持有或等待锁的操作。

一旦你向一个操作发送了“kill”消息，它在`db.currentOp()`输出中将会有一个`"killed"`字段。但直到它从当前操作列表中消失之前，它实际上并没有死亡。

在 MongoDB 4.0 中，`killOP` 方法已扩展以允许在*mongos*上运行。它现在可以终止在集群中跨多个分片运行的查询（读操作）。在之前的版本中，这需要手动在每个分片的主 *mongod* 上发出终止命令。

## 误报

如果你查找慢操作，可能会看到一些长时间运行的内部操作被列出。根据你的设置，MongoDB 可能有几个长时间运行的请求。最常见的是复制线程（它将继续从同步源获取更多操作，尽可能长时间运行）和用于分片的写回监听器。可以忽略任何在*local.oplog.rs*上长时间运行的查询，以及任何[*writebacklistener*命令](https://oreil.ly/95e3x)。

如果你终止这些操作中的任意一个，MongoDB 将会重新启动它们。但一般情况下不建议这样做。终止复制线程将会暂时停止复制，而终止写回监听器可能导致*mongos*错过合法的写入错误。

## 防止幽灵操作

有一个奇怪的、特定于 MongoDB 的问题，你可能会遇到，特别是当你将数据大批量加载到一个集合中时。假设你有一个作业正在向 MongoDB 发送数千个更新操作，并且 MongoDB 几乎停顿下来了。你迅速停止了作业并杀死了所有当前正在进行的更新。然而，即使作业不再运行，你仍然会看到新的更新操作出现在你杀死旧操作之后！

如果您使用未确认的写入加载数据，您的应用程序可能会比 MongoDB 更快地向其发送写入请求。如果 MongoDB 被堵塞，这些写入将堆积在操作系统的套接字缓冲区中。当您终止 MongoDB 正在处理的写入时，这将允许 MongoDB 开始处理缓冲区中的写入。即使停止客户端发送写入，任何进入缓冲区的写入也将由 MongoDB 处理，因为它们已经“接收到”（只是尚未处理）。

要防止这些幻写的最佳方法是进行*确认的*写入：确保每次写入都等到前一次写入完成，而不仅仅是等到前一次写入已经在数据库服务器的缓冲区中。

# 使用系统分析器

要查找慢速操作，您可以使用系统分析器，它会记录一个特殊的*system.profile*集合中的操作。分析器可以为您提供大量关于长时间操作的信息，但这也会带来代价：它会减慢*mongod*的整体性能。因此，您可能只想定期打开分析器来捕获部分流量。如果系统已经负载过重，您可能希望使用本章描述的其他技术来诊断问题。

默认情况下，分析器处于关闭状态，并且不记录任何内容。您可以通过在 shell 中运行`db.setProfilingLevel()`来打开它：

```
> db.setProfilingLevel(2)
{ "was" : 0, "slowms" : 100, "ok" : 1 }
```

级别 2 表示“profile everything”。数据库接收的每个读取和写入请求都将记录在当前数据库的*system.profile*集合中。分析是按数据库启用的，并且会产生严重的性能损失：每次写入都必须额外写入一次，每次读取都必须获取写入锁（因为它必须将条目写入*system.profile*集合）。但它将为您提供系统正在做什么的详尽列表：

```
> db.foo.insert({x:1})
> db.foo.update({},{$set:{x:2}})
> db.foo.remove()
> db.system.profile.find().pretty()
{
    "op" : "insert",
    "ns" : "sample_mflix.foo",
    "command" : {
        "insert" : "foo",
        "ordered" : true,
        "lsid" : {
            "id" : UUID("63b7df66-ca97-41f4-a245-eba825485147")
        },
        "$readPreference" : {
            "mode" : "secondaryPreferred"
        },
        "$db" : "sample_mflix"
    },
    "ninserted" : 1,
    "keysInserted" : 1,
    "numYield" : 0,
    "locks" : { ... },
    "flowControl" : {
        "acquireCount" : NumberLong(3)
    },
    "responseLength" : 45,
    "protocol" : "op_msg",
    "millis" : 33,
    "client" : "127.0.0.1",
    "appName" : "MongoDB Shell",
    "allUsers" : [ ],
    "user" : ""
}
{
    "op" : "update",
    "ns" : "sample_mflix.foo",
    "command" : {
        "q" : {

        },
        "u" : {
            "$set" : {
                "x" : 2
            }
        },
        "multi" : false,
        "upsert" : false
    },
    "keysExamined" : 0,
    "docsExamined" : 1,
    "nMatched" : 1,
    "nModified" : 1,
    "numYield" : 0,
    "locks" : { ... },
    "flowControl" : {
        "acquireCount" : NumberLong(1)
    },
    "millis" : 0,
    "planSummary" : "COLLSCAN",
    "execStats" : { ...
        "inputStage" : {
            ...
        }
    },
    "ts" : ISODate("2019-09-03T22:39:33.856Z"),
    "client" : "127.0.0.1",
    "appName" : "MongoDB Shell",
    "allUsers" : [ ],
    "user" : ""
}
{
     "op" : "remove",
    "ns" : "sample_mflix.foo",
    "command" : {
        "q" : {

        },
        "limit" : 0
    },
    "keysExamined" : 0,
    "docsExamined" : 1,
    "ndeleted" : 1,
    "keysDeleted" : 1,
    "numYield" : 0,
    "locks" : { ... },
    "flowControl" : {
        "acquireCount" : NumberLong(1)
    },
    "millis" : 0,
    "planSummary" : "COLLSCAN",
    "execStats" : { ...
        "inputStage" : { ... }
    },
    "ts" : ISODate("2019-09-03T22:39:33.858Z"),
    "client" : "127.0.0.1",
    "appName" : "MongoDB Shell",
    "allUsers" : [ ],
    "user" : ""
}
```

您可以使用`"client"`字段查看哪些用户将哪些操作发送到数据库。如果使用了身份验证，您还可以看到每个操作由哪个用户执行。

通常情况下，您可能不关心数据库正在进行的大多数操作，只关心慢速操作。为此，您可以将分析级别设置为 1。默认情况下，级别 1 会记录持续时间超过 100 毫秒的操作。您还可以指定第二个参数，定义对您而言什么是“慢速”操作。这将记录所有持续时间超过 500 毫秒的操作：

```
> db.setProfilingLevel(1, 500)
{ "was" : 2, "slowms" : 100, "ok" : 1 }
```

要关闭分析，将分析级别设置为 0：

```
> db.setProfilingLevel(0)
{ "was" : 1, "slowms" : 500, "ok" : 1 }
```

通常不建议将`slowms`设置为一个较低的值。即使关闭了分析，`slowms`也会对*mongod*产生影响：它设置在日志中打印慢速操作的阈值。因此，如果将`slowms`降低以进行分析，然后关闭分析之前可能需要将其提高。

您可以使用`db.getProfilingLevel()`来查看当前的分析级别。分析级别不是持久的：重新启动数据库会清除级别。

有用于配置分析级别的命令行选项，即``--profile *`level`*``和``--slowms *`time`*``，但增加分析级别通常是一种临时调试措施，不是长期配置的内容。

在 MongoDB 4.2 中，扩展了读/写操作的分析器条目和诊断日志消息，以帮助改进慢查询的识别，增加了`queryHash`和`planCacheKey`字段。`queryHash`字符串表示查询形状的哈希，仅依赖于查询形状。每个查询形状都与一个`queryHash`相关联，这样可以更容易地突出显示使用相同形状的查询。`planCacheKey`是与查询关联的计划缓存条目的关键的哈希。它包括查询形状的详细信息以及当前可用的索引的详细信息。这些帮助您将分析器的可用信息与查询性能诊断相关联。

如果您启用了性能分析，并且*system.profile*集合尚不存在，MongoDB 会为其创建一个小的固定大小集合（几兆字节）。如果您希望长时间运行分析器，则这可能不足以记录您需要记录的操作数量。您可以通过关闭分析、删除*system.profile*集合，并创建所需大小的新*system.profile*固定大小集合来创建一个较大的*system.profile*集合。然后在数据库上启用分析。

# 计算大小

为了正确配置磁盘和 RAM 的数量，了解文档、索引、集合和数据库占用的空间是很有用的。参见“计算工作集”以获取有关计算工作集的信息。

## 文档

获取文档大小的最简单方法是使用 Shell 的`Object.bsonsize()`函数。传入任何文档即可获取存储在 MongoDB 中时的大小。

例如，您可以看到将`_id`存储为`ObjectId`比存储为字符串更有效：

```
> Object.bsonsize({_id:ObjectId()})
22
> // ""+ObjectId() converts the ObjectId to a string
> Object.bsonsize({_id:""+ObjectId()})
39
```

更实际地说，您可以直接从您的集合中传入文档：

```
> Object.bsonsize(db.users.findOne())
```

这显示了文档在磁盘上占用了多少字节。然而，这并不包括填充或索引的计数，这些通常是集合大小的重要因素之一。

## 集合

要查看整个集合的信息，可以使用`stats`函数：

```
>db.movies.stats()
{
    "ns" : "sample_mflix.movies",
    "size" : 65782298,
    "count" : 45993,
    "avgObjSize" : 1430,
    "storageSize" : 45445120,
    "capped" : false,
    "wiredTiger" : {
        "metadata" : {
            "formatVersion" : 1
        },
        "creationString" : "access_pattern_hint=none,allocation_size=4KB,\
 app_metadata=(formatVersion=1),assert=(commit_timestamp=none,\
 read_timestamp=none),block_allocation=best,block_compressor=\
 snappy,cache_resident=false,checksum=on,colgroups=,collator=,\
 columns=,dictionary=0,encryption=(keyid=,name=),exclusive=\
 false,extractor=,format=btree,huffman_key=,huffman_value=,\
 ignore_in_memory_cache_size=false,immutable=false,internal_item_\
 max=0,internal_key_max=0,internal_key_truncate=true,internal_\
 page_max=4KB,key_format=q,key_gap=10,leaf_item_max=0,leaf_key_\
 max=0,leaf_page_max=32KB,leaf_value_max=64MB,log=(enabled=true),\
 lsm=(auto_throttle=true,bloom=true,bloom_bit_count=16,bloom_\
 config=,bloom_hash_count=8,bloom_oldest=false,chunk_count_limit\
 =0,chunk_max=5GB,chunk_size=10MB,merge_custom=(prefix=,start_\
 generation=0,suffix=),merge_max=15,merge_min=0),memory_page_image\
 _max=0,memory_page_max=10m,os_cache_dirty_max=0,os_cache_max=0,\
 prefix_compression=false,prefix_compression_min=4,source=,split_\
 deepen_min_child=0,split_deepen_per_child=0,split_pct=90,type=file,\
 value_format=u",
        "type" : "file",
        "uri" : "statistics:table:collection-14--2146526997547809066",
        "LSM" : {
            "bloom filter false positives" : 0,
            "bloom filter hits" : 0,
            "bloom filter misses" : 0,
            "bloom filter pages evicted from cache" : 0,
            "bloom filter pages read into cache" : 0,
            "bloom filters in the LSM tree" : 0,
            "chunks in the LSM tree" : 0,
            "highest merge generation in the LSM tree" : 0,
            "queries that could have benefited from a Bloom filter 
 that did not exist" : 0,
            "sleep for LSM checkpoint throttle" : 0,
            "sleep for LSM merge throttle" : 0,
            "total size of bloom filters" : 0
        },
        "block-manager" : {
            "allocations requiring file extension" : 0,
            "blocks allocated" : 1358,
            "blocks freed" : 1322,
            "checkpoint size" : 39219200,
            "file allocation unit size" : 4096,
            "file bytes available for reuse" : 6209536,
            "file magic number" : 120897,
            "file major version number" : 1,
            "file size in bytes" : 45445120,
            "minor version number" : 0
        },
        "btree" : {
            "btree checkpoint generation" : 22,
            "column-store fixed-size leaf pages" : 0,
            "column-store internal pages" : 0,
            "column-store variable-size RLE encoded values" : 0,
            "column-store variable-size deleted values" : 0,
            "column-store variable-size leaf pages" : 0,
            "fixed-record size" : 0,
            "maximum internal page key size" : 368,
            "maximum internal page size" : 4096,
            "maximum leaf page key size" : 2867,
            "maximum leaf page size" : 32768,
            "maximum leaf page value size" : 67108864,
            "maximum tree depth" : 0,
            "number of key/value pairs" : 0,
            "overflow pages" : 0,
            "pages rewritten by compaction" : 1312,
            "row-store empty values" : 0,
            "row-store internal pages" : 0,
            "row-store leaf pages" : 0
        },
        "cache" : {
            "bytes currently in the cache" : 40481692,
            "bytes dirty in the cache cumulative" : 40992192,
            "bytes read into cache" : 37064798,
            "bytes written from cache" : 37019396,
            "checkpoint blocked page eviction" : 0,
            "data source pages selected for eviction unable to be evicted" : 32,
            "eviction walk passes of a file" : 0,
            "eviction walk target pages histogram - 0-9" : 0,
            "eviction walk target pages histogram - 10-31" : 0,
            "eviction walk target pages histogram - 128 and higher" : 0,
            "eviction walk target pages histogram - 32-63" : 0,
            "eviction walk target pages histogram - 64-128" : 0,
            "eviction walks abandoned" : 0,
            "eviction walks gave up because they restarted their walk twice" : 0,
            "eviction walks gave up because they saw too many pages 
 and found no candidates" : 0,
            "eviction walks gave up because they saw too many pages 
 and found too few candidates" : 0,
            "eviction walks reached end of tree" : 0,
            "eviction walks started from root of tree" : 0,
            "eviction walks started from saved location in tree" : 0,
            "hazard pointer blocked page eviction" : 0,
            "in-memory page passed criteria to be split" : 0,
            "in-memory page splits" : 0,
            "internal pages evicted" : 8,
            "internal pages split during eviction" : 0,
            "leaf pages split during eviction" : 0,
            "modified pages evicted" : 1312,
            "overflow pages read into cache" : 0,
            "page split during eviction deepened the tree" : 0,
            "page written requiring cache overflow records" : 0,
            "pages read into cache" : 1330,
            "pages read into cache after truncate" : 0,
            "pages read into cache after truncate in prepare state" : 0,
            "pages read into cache requiring cache overflow entries" : 0,
            "pages requested from the cache" : 3383,
            "pages seen by eviction walk" : 0,
            "pages written from cache" : 1334,
            "pages written requiring in-memory restoration" : 0,
            "tracked dirty bytes in the cache" : 0,
            "unmodified pages evicted" : 8
        },
        "cache_walk" : {
            "Average difference between current eviction generation 
 when the page was last considered" : 0,
            "Average on-disk page image size seen" : 0,
            "Average time in cache for pages that have been visited 
 by the eviction server" : 0,
            "Average time in cache for pages that have not been visited
 by the eviction server" : 0,
            "Clean pages currently in cache" : 0,
            "Current eviction generation" : 0,
            "Dirty pages currently in cache" : 0,
            "Entries in the root page" : 0,
            "Internal pages currently in cache" : 0,
            "Leaf pages currently in cache" : 0,
            "Maximum difference between current eviction generation 
 when the page was last considered" : 0,
            "Maximum page size seen" : 0,
            "Minimum on-disk page image size seen" : 0,
            "Number of pages never visited by eviction server" : 0,
            "On-disk page image sizes smaller than a single allocation unit" : 0,
            "Pages created in memory and never written" : 0,
            "Pages currently queued for eviction" : 0,
            "Pages that could not be queued for eviction" : 0,
            "Refs skipped during cache traversal" : 0,
            "Size of the root page" : 0,
            "Total number of pages currently in cache" : 0
        },
        "compression" : {
            "compressed page maximum internal page size 
 prior to compression" : 4096,
            "compressed page maximum leaf page size 
 prior to compression " : 131072,
            "compressed pages read" : 1313,
            "compressed pages written" : 1311,
            "page written failed to compress" : 1,
            "page written was too small to compress" : 22
        },
        "cursor" : {
            "bulk loaded cursor insert calls" : 0,
            "cache cursors reuse count" : 0,
            "close calls that result in cache" : 0,
            "create calls" : 1,
            "insert calls" : 0,
            "insert key and value bytes" : 0,
            "modify" : 0,
            "modify key and value bytes affected" : 0,
            "modify value bytes modified" : 0,
            "next calls" : 0,
            "open cursor count" : 0,
            "operation restarted" : 0,
            "prev calls" : 1,
            "remove calls" : 0,
            "remove key bytes removed" : 0,
            "reserve calls" : 0,
            "reset calls" : 2,
            "search calls" : 0,
            "search near calls" : 0,
            "truncate calls" : 0,
            "update calls" : 0,
            "update key and value bytes" : 0,
            "update value size change" : 0
        },
        "reconciliation" : {
            "dictionary matches" : 0,
            "fast-path pages deleted" : 0,
            "internal page key bytes discarded using suffix compression" : 0,
            "internal page multi-block writes" : 0,
            "internal-page overflow keys" : 0,
            "leaf page key bytes discarded using prefix compression" : 0,
            "leaf page multi-block writes" : 0,
            "leaf-page overflow keys" : 0,
            "maximum blocks required for a page" : 1,
            "overflow values written" : 0,
            "page checksum matches" : 0,
            "page reconciliation calls" : 1334,
            "page reconciliation calls for eviction" : 1312,
            "pages deleted" : 0
        },
        "session" : {
            "object compaction" : 4
        },
        "transaction" : {
            "update conflicts" : 0
        }
    },
    "nindexes" : 5,
    "indexBuilds" : [ ],
    "totalIndexSize" : 46292992,
    "indexSizes" : {
        "_id_" : 446464,
        "$**_text" : 44474368,
        "genres_1_imdb.rating_1_metacritic_1" : 724992,
        "tomatoes_rating" : 307200,
        "getMovies" : 339968
    },
    "scaleFactor" : 1,
    "ok" : 1
}
```

`stats`从命名空间（`"sample_mflix.movies"`）开始，接着是集合中所有文档的计数。接下来的几个字段涉及集合的大小。`"size"`是调用每个元素的`Object.bsonsize()`时得到的值总和：这是未压缩时集合文档在内存中占用的实际字节数。类似地，如果将`"avgObjSize"`乘以`"count"`，你将得到未压缩的`"size"`内存中的值。

正如前文所述，仅统计文档字节总数并未计算集合压缩后的节省空间。`"storageSize"`可能比`"size"`小，反映了通过压缩节省的空间。

`"nindexes"`是集合上的索引数。索引在建立完成之前不计入`"nindexes"`，并且在此列表中出现之前不能使用。通常，索引会比它们存储的数据量大得多。通过使用右平衡索引（如“复合索引简介”中所述），可以最小化这种空闲空间。随机分布的索引通常会有约 50%的空闲空间，而升序索引会有 10%的空闲空间。

随着你的集合变得越来越大，可能会发现阅读`stats`输出变得困难，因为其尺寸可能达到数十亿字节甚至更大。因此，你可以传入一个缩放因子：`1024`表示千字节，`1024*1024`表示兆字节，以此类推。例如，以下命令将以 TB 单位获取集合的统计信息：

```
> db.big.stats(1024*1024*1024*1024)
```

## 数据库

数据库有一个类似于集合的`stats`函数：

```
> db.stats()
{
    "db" : "sample_mflix",
    "collections" : 5,
    "views" : 0,
    "objects" : 98308,
    "avgObjSize" : 819.8680982219148,
    "dataSize" : 80599593,
    "storageSize" : 53620736,
    "numExtents" : 0,
    "indexes" : 12,
    "indexSize" : 47001600,
    "scaleFactor" : 1,
    "fsUsedSize" : 355637043200,
    "fsTotalSize" : 499963174912,
    "ok" : 1
}
```

首先，我们有数据库的名称、其包含的集合数量以及数据库的视图数。`"objects"`是此数据库中所有集合中文档的总数。

文档的大部分包含关于数据大小的信息。`"fsTotalSize"`应始终最大：它是 MongoDB 实例存储数据的文件系统的总容量大小。`"fsUsedSize"`表示目前 MongoDB 在该文件系统中使用的总空间。这应该对应于数据目录中所有文件使用的总空间。

接下来最大的字段通常会是`"dataSize"`，它是此数据库中保存的未压缩数据的大小。这与`"storageSize"`不匹配，因为在 WiredTiger 中数据通常是压缩的。`"indexSize"`是此数据库所有索引占用的空间大小。

`db.stats()`可以像集合的`stats`函数一样接受一个缩放参数。如果在不存在的数据库上调用`db.stats()`，则所有值都将为零。

请记住，在高锁定百分比系统上列出数据库可能会非常缓慢，并阻塞其他操作。如果可能的话，请避免执行此操作。

# 使用 mongotop 和 mongostat

MongoDB 配备了几个命令行工具，可以通过每隔几秒钟打印统计信息来帮助您确定它正在执行的操作。

mongotop 类似于 Unix 的 *top* 实用程序：它提供了最繁忙的集合概览。您还可以运行 `mongotop --locks` 来获取每个数据库的锁定统计信息。

mongostat 提供了全局服务器信息。默认情况下，`mongostat` 每秒打印一次统计信息，但可以通过命令行传递不同的秒数进行配置。每个字段显示的是自上次打印以来活动发生的次数：

`insert/query/update/delete/getmore/command`

每种操作的简单计数。

`flushes`

*mongod* 将数据刷新到磁盘的次数。

`mapped`

*mongod* 映射的内存量。通常大致等于您的数据目录的大小。

`vsize`

*mongod* 正在使用的虚拟内存量。这通常是数据目录大小的两倍（一次用于映射文件，再次用于日志记录）。

`res`

*mongod* 正在使用的内存量。这通常应尽可能接近机器上所有内存的总量。

`locked db`

在最后的时间片中锁定时间最长的数据库。此字段报告了数据库被锁定的时间百分比，以及全局锁定持续的时间，因此该值可能超过 100%。

`idx miss %`

索引访问百分比，由于索引条目或搜索的索引部分不在内存中（因此 *mongod* 必须访问磁盘），导致页面错误。这是输出中最令人困惑的字段。

`qr|qw`

读取和写入队列的大小（即有多少读取和写入操作正被阻塞等待处理）。

`ar|aw`

当前活跃客户端数量（即当前执行读取和写入操作的客户端数）。

`netIn`

MongoDB 计算的网络字节输入量（与操作系统测量的不一定相同）。

`netOut`

MongoDB 计算的网络字节输出量。

`conn`

此服务器打开的连接数，包括传入和传出的连接。

`time`

统计数据获取时的时间。

您可以在副本集或分片集群上运行 *mongostat*。如果使用 `--discover` 选项，*mongostat* 将尝试从初始连接的成员找到集合或群集的所有成员，并为每个服务器每秒打印一行数据。对于大型群集，这可能会快速变得不可管理，但对于小型群集和可以消费数据并以更可读的形式呈现的工具，这可能是有用的。

*mongostat* 是快速获取数据库正在执行的操作快照的好方法，但是对于长期监控，推荐使用类似 MongoDB Atlas 或 Ops Manager 的工具（参见 第二十二章）。
