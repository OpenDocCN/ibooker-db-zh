# 附录 B. MongoDB 内部

若要有效地使用 MongoDB，无需理解其内部机制，但对希望开发工具、贡献代码或简单了解系统底层运行情况的开发者可能会感兴趣。本附录介绍了一些基础知识。MongoDB 源代码可在[*https://github.com/mongodb/mongo*](https://github.com/mongodb/mongo)获取。

# BSON

MongoDB 中的文档是一个抽象概念，具体的文档表示取决于所使用的驱动程序/语言。由于文档在 MongoDB 中广泛用于通信，因此需要一种所有驱动程序、工具和过程都能共享的文档表示。这种表示称为二进制 JSON 或 BSON（没有人知道 J 去了哪里）。

BSON 是一种轻量级的二进制格式，能够将任何 MongoDB 文档表示为一串字节。数据库理解 BSON，并且 BSON 是文档保存到磁盘的格式。

当驱动程序收到要插入的文档、用作查询等任务时，它会在将其发送到服务器之前将该文档编码为 BSON。同样，从服务器返回给客户端的文档也以 BSON 字符串形式发送。驱动程序将这些 BSON 数据解码为其本机文档表示形式，然后返回给客户端。

BSON 格式有三个主要目标：

效率

BSON 被设计为高效地表示数据，几乎不使用额外的空间。在最坏的情况下，BSON 稍微不如 JSON 高效，在最好的情况下（例如存储二进制数据或大型数字时），它则比 JSON 高效得多。

遍历性能

在某些情况下，BSON 会牺牲空间效率以使格式更易于遍历。例如，字符串值会以长度前缀的方式存储，而不是依赖终止符来表示字符串的结束。这种遍历性能在 MongoDB 服务器需要自省文档时非常有用。

性能

最后，BSON 被设计为快速编码和解码。它使用 C 风格的类型表示，在大多数编程语言中都能快速处理。

关于 BSON 的详细规范，请参见[*http://www.bsonspec.org*](http://www.bsonspec.org)。

# 网络协议

驱动程序使用轻量级的 TCP/IP 网络协议访问 MongoDB 服务器。该协议在[MongoDB 文档站点](https://oreil.ly/rVJAr)有文档记录，但基本上是 BSON 数据的薄包装。例如，插入消息包括 20 字节的头数据（包括告知服务器执行插入操作的代码和消息长度）、要插入的集合名称以及要插入的 BSON 文档列表。

# 数据文件

在 MongoDB 数据目录（默认为 */data/db/*）中，每个集合和索引都会存储在单独的文件中。文件名不对应集合或索引的名称，但可以使用 *mongo* shell 中的 `stats` 来识别特定集合的相关文件。`"wiredTiger.uri"` 字段将包含要在 MongoDB 数据目录中查找的文件名。

在 *sample_mflix* 数据库上使用 `stats` 对 *movies* 集合执行时，`"wiredTiger.uri"` 字段将给出 “collection-14--2146526997547809066” 作为结果：

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
 read_timestamp=none),block_allocation=best,\
 block_compressor=snappy,cache_resident=false,checksum=on,\
 colgroups=,collator=,columns=,dictionary=0,\
 encryption=(keyid=,name=),exclusive=false,extractor=,format=btree,\
 huffman_key=,huffman_value=,ignore_in_memory_cache_size=false,\
 immutable=false,internal_item_max=0,internal_key_max=0,\
 internal_key_truncate=true,internal_page_max=4KB,key_format=q,\
 key_gap=10,leaf_item_max=0,leaf_key_max=0,leaf_page_max=32KB,\
 leaf_value_max=64MB,log=(enabled=true),lsm=(auto_throttle=true,\
 bloom=true,bloom_bit_count=16,bloom_config=,bloom_hash_count=8,\
 bloom_oldest=false,chunk_count_limit=0,chunk_max=5GB,\
 chunk_size=10MB,merge_custom=(prefix=,start_generation=0,suffix=),\
 merge_max=15,merge_min=0),memory_page_image_max=0,\
 memory_page_max=10m,os_cache_dirty_max=0,os_cache_max=0,\
 prefix_compression=false,prefix_compression_min=4,source=,\
 split_deepen_min_child=0,split_deepen_per_child=0,split_pct=90,\
 type=file,value_format=u",
        "type" : "file",
        "uri" : "statistics:table:collection-14--2146526997547809066",
    ...
}
```

然后可以在 MongoDB 数据目录中验证文件的详细信息：

```
ls -alh collection-14--2146526997547809066.wt
-rw-------  1 braz  staff  43M 28 Sep 23:33 collection-14--2146526997547809066.wt
```

可以使用聚合框架来查找特定集合中每个索引的 URI，方法如下：

```
db.movies.aggregate([{
      $collStats:{storageStats:{}}}]).next().storageStats.indexDetails
{
    "_id_" : {
    "metadata" : {
        "formatVersion" : 8,
        "infoObj" : "{ \"v\" : 2, \"key\" : { \"_id\" : 1 },\
 \"name\" : \"_id_\", \"ns\" : \"sample_mflix.movies\" }"
    },
    "creationString" : "access_pattern_hint=none,allocation_size=4KB,\
 app_metadata=(formatVersion=8,infoObj={ \"v\" : 2, \"key\" : \
 { \"_id\" : 1 },\"name\" : \"_id_\", \"ns\" : \"sample_mflix.movies\" }),\
 assert=(commit_timestamp=none,read_timestamp=none),block_allocation=best,\
 block_compressor=,cache_resident=false,checksum=on,colgroups=,collator=,\
 columns=,dictionary=0,encryption=(keyid=,name=),exclusive=false,extractor=,\
 format=btree,huffman_key=,huffman_value=,ignore_in_memory_cache_size=false,\
 immutable=false,internal_item_max=0,internal_key_max=0,\
 internal_key_truncate=true,internal_page_max=16k,key_format=u,key_gap=10,\
 leaf_item_max=0,leaf_key_max=0,leaf_page_max=16k,leaf_value_max=0,\
 log=(enabled=true),lsm=(auto_throttle=true,bloom=true,bloom_bit_count=16,\
 bloom_config=,bloom_hash_count=8,bloom_oldest=false,chunk_count_limit=0,\
 chunk_max=5GB,chunk_size=10MB,merge_custom=(prefix=,start_generation=0,\
 suffix=),merge_max=15,merge_min=0),memory_page_image_max=0,\
 memory_page_max=5MB,os_cache_dirty_max=0,os_cache_max=0,\
 prefix_compression=true,prefix_compression_min=4,source=,\
 split_deepen_min_child=0,split_deepen_per_child=0,split_pct=90,type=file,\
 value_format=u",
    "type" : "file",
    "uri" : "statistics:table:index-17--2146526997547809066",
...
    "$**_text" : {
...
      "uri" : "statistics:table:index-29--2146526997547809066",
...
    "genres_1_imdb.rating_1_metacritic_1" : {
...
      "uri" : "statistics:table:index-30--2146526997547809066",
...
}
```

WiredTiger 将每个集合或索引存储在单个任意大的文件中。影响此文件潜在最大大小的唯一限制是文件系统大小限制。

每当更新文档时，WiredTiger 都会写入该文档的新副本。磁盘上的旧副本会被标记为可重用，并且通常在下一个检查点期间被重写。这样可以回收 WiredTiger 文件中使用的空间。可以运行`compact`命令来将此文件中的数据移动到开头，从而在末尾留下空白空间。定期间隔时，WiredTiger 通过截断文件来移除这些多余的空白空间。在压缩过程结束时，多余的空间将被返回给文件系统。

# 命名空间

每个数据库都组织成 *命名空间*，这些命名空间映射到 WiredTiger 文件。这种抽象将存储引擎的内部细节与 MongoDB 查询层分离。

# WiredTiger 存储引擎

MongoDB 的默认存储引擎是 WiredTiger 存储引擎。服务器启动时，它会打开数据文件并开始检查点和日志处理过程。它与操作系统协同工作，后者负责页面数据的进出以及将数据刷新到磁盘。这个存储引擎具有几个重要属性：

+   默认情况下，对集合和索引启用了压缩。默认压缩算法为 Google 的 snappy。其他选项包括 Facebook 的 Zstandard（zstd）和 zlib，或者不进行压缩。这可以在增加 CPU 要求的情况下最大限度地减少数据库中的存储使用。

+   文档级并发允许多个客户端在集合中同时更新不同文档。WiredTiger 使用多版本并发控制（MVCC）来隔离读写操作，确保客户端在操作开始时看到数据的一致时间点视图。

+   检查点创建数据的一致时间点快照，并且每 60 秒执行一次。它涉及将快照中的所有数据写入磁盘并更新相关元数据。

+   使用检查点技术进行日志记录确保在*mongod*进程失败时不会丢失任何数据。WiredTiger 使用预写日志（日志）在应用修改之前存储这些修改。
