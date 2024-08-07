# 第十二章：从您的应用程序连接到副本集

本章涵盖应用程序与副本集的交互，包括：

+   连接和故障转移的工作方式

+   等待写入复制完成

+   将读取路由到正确的成员

# 客户端到副本集连接行为

MongoDB 客户端库（MongoDB 术语中称为“驱动程序”）旨在管理与 MongoDB 服务器的通信，无论服务器是独立的 MongoDB 实例还是副本集。对于副本集，默认情况下，驱动程序将连接到主要成员并将所有流量路由到主要成员。您的应用程序可以像与独立服务器交互一样执行读取和写入操作，而您的副本集则在后台保持准备就绪的热备份。

连接到副本集的方式类似于连接到单个服务器。在您的驱动程序中使用`MongoClient`类（或等效类），并为驱动程序提供一个种子列表以连接到。种子列表只是一系列服务器地址。*种子*是您的应用程序将从中读取和写入数据的副本集成员。您不需要在种子列表中列出所有成员（尽管可以这样做）。当驱动程序连接到种子时，它将从它们那里发现其他成员。连接字符串通常如下所示：

```
"mongodb://server-1:27017,server-2:27017,server-3:27017"
```

有关详细信息，请参阅您的驱动程序文档。

要增强可靠性，您还应使用[DNS 种子连接格式](https://oreil.ly/Uq4za)来指定应用程序如何连接到您的副本集。使用 DNS 的优势在于，托管 MongoDB 副本集成员的服务器可以轮换更改，而无需重新配置客户端（具体来说是它们的连接字符串）。

所有 MongoDB 驱动程序遵循[服务器发现和监控（SDAM）规范](https://oreil.ly/ZsS8p)。它们持续监视您的副本集的拓扑结构，以便检测应用程序能否访问集合的所有成员是否有任何变化。此外，驱动程序还监视集合以保持关于哪个成员是主要成员的信息。

副本集的目的是在面对网络分区或服务器宕机时使您的数据高度可用。在正常情况下，副本集通过选举新的主要成员来优雅地响应此类问题，以便应用程序可以继续读取和写入数据。如果主要成员宕机，驱动程序将自动找到新的主要成员（一旦选举出一个）并尽快将请求路由到它。然而，在没有可达的主要成员时，您的应用程序将无法执行写操作。

在选举期间可能短时间内没有主要成员，或者长时间内没有可达的成员能够成为主要成员。默认情况下，驱动程序在此期间不会处理任何请求（读取或写入）。如果对您的应用程序有必要，可以配置驱动程序使用次要成员进行读取请求。

一个常见的愿望是使驱动程序隐藏选举过程的全部细节（主节点失效和选出新的主节点）对用户不可见。然而，没有驱动程序以这种方式处理故障转移，原因有几个。首先，驱动程序只能在有限的时间内隐藏没有主节点的情况。其次，驱动程序通常是因为操作失败而发现主节点已经宕机，这意味着驱动程序无法确定主节点在宕机之前是否已处理了操作。这是一个基本的分布式系统问题，不可能避免，因此我们在问题出现时需要一种处理策略。如果新的主节点迅速选出，我们应该重试操作吗？假设它已经在旧的主节点上处理了？检查并查看新的主节点是否有该操作？

正确的策略，结果证明，最多重试一次。嗯？解释一下，让我们考虑一下我们的选择。这些可以归结为以下几点：不重试，在重试固定次数后放弃，或者最多重试一次。我们还需要考虑可能导致问题的错误类型。在尝试写入副本集时，可能出现三种类型的错误：瞬时网络错误，持久性中断（无论是网络还是服务器），或者由服务器拒绝的命令错误（例如，未经授权）。针对每种错误类型，让我们考虑我们的重试选项。

为了讨论的完整性，让我们以简单地增加计数器的写入为例。如果我们的应用程序试图增加计数器但未收到服务器的响应，我们不知道服务器是否收到了消息并执行了更新。因此，如果我们遵循不重试此写入的策略，对于瞬时网络错误，我们可能会计数不准确。对于持久性中断或命令错误，不重试是正确的策略，因为再次尝试写操作不会产生预期的效果。

如果我们遵循重试固定次数的策略，对于瞬时网络错误，我们可能会计数过多（在第一次尝试成功的情况下）。对于持久性中断或命令错误，多次重试只会浪费资源。

现在让我们看看只重试一次的策略。对于瞬时网络错误，我们可能会计数过多。对于持久性中断或命令错误，这是正确的策略。然而，如果我们能确保我们的操作是幂等的呢？幂等操作是指无论我们执行一次还是多次，其结果都相同。对于幂等操作，仅在网络错误时重试一次具有最佳处理所有三种错误类型的可能性。

从 MongoDB 3.6 起，服务器和所有 MongoDB 驱动程序都支持可重试写选项。请查阅您驱动程序的文档以获取有关如何使用此选项的详细信息。使用可重试写入时，驱动程序将自动采用最多一次重试的策略。命令错误将返回给应用程序以供客户端处理。网络错误将在适当的延迟后重试一次，以便在普通情况下适应主节点选举。启用可重试写入后，服务器将为每个写操作维护唯一标识符，因此可以确定驱动程序何时尝试重试已成功的命令。而不是再次应用写入，它将简单地返回一个表明写入成功的消息，从而解决由于暂时网络问题导致的问题。

# 等待写入的复制

根据您的应用程序需求，可能希望在服务器确认之前将所有写入复制到大多数副本集。在极少数情况下，如果副本集的主节点宕机，新选举的主节点（之前是副本）未能复制前主节点的最后写入，那么当前主节点恢复时将会回滚这些写入。这些写入可以恢复，但需要手动干预。对许多应用程序而言，少量写入回滚并非问题。例如，在博客应用程序中，回滚一两条读者评论几乎没有真正的危险性。

但是，对于其他应用程序，应尽量避免任何写入的回滚。假设您的应用程序将写入发送到主节点，并收到写入已成功的确认，但在任何副本有机会复制该写入之前，主节点崩溃了。您的应用程序认为它能够访问该写入，但副本集的当前成员却没有它的副本。

在某些时候，一个副本可能会被选为主节点并开始接受新的写入。当先前的主节点恢复时，它将发现自己有一些当前主节点没有的写入。为了纠正这一点，它将撤销任何与当前主节点操作顺序不匹配的写入。这些操作并未丢失，而是写入特殊的回滚文件中，需要手动应用到当前主节点上。由于可能与其他在宕机后发生的写入冲突，MongoDB 无法自动应用这些写入。因此，这些写入实际上会消失，直到管理员有机会将回滚文件应用到当前主节点上（详见第十一章有关回滚的更多详情）。

写入到大多数的要求防止了这种情况：如果应用程序收到确认写入成功的确认，那么新的主节点必须复制写入才能被选举（成员必须保持最新以被选举为主节点）。如果应用程序未收到服务器的确认或收到错误，则将知道需要再次尝试，因为写入在主节点崩溃之前未传播到大多数集合。

因此，为了确保写入被持久化，不管集群中发生什么情况，我们必须确保每次写入都传播到集合中大多数成员。我们可以使用`writeConcern`来实现这一点。

自 MongoDB 2.6 起，`writeConcern`与写操作集成在一起。例如，在 JavaScript 中，我们可以如下使用`writeConcern`：

```
try {
   db.products.insertOne(
       { "_id": 10, "item": "envelopes", "qty": 100, type: "Self-Sealing" },
       { writeConcern: { "w" : "majority", "wtimeout" : 100 } }
   );
} catch (e) {
   print (e);
}
```

在你的驱动程序中的具体语法将根据编程语言而异，但语义保持不变。在这个例子中，我们指定了`"majority"`的写关注点。成功后，服务器将响应以下消息：

```
{ "acknowledged" : true, "insertedId" : 10 }
```

但是在此写入操作复制到复制集大多数成员之前，服务器将不会响应。只有在此写入操作成功复制到指定的超时时间内，我们的应用程序才会收到确认消息：

```
WriteConcernError({
   "code" : 64,
   "errInfo" : {
      "wtimeout" : true
   },
   "errmsg" : "waiting for replication timed out"
})
```

写入关注点大多数和复制集选举协议确保在主节点选举事件中，只有更新了已确认写入的从节点才能被选为主节点。通过超时选项，我们还有一个可调整的设置，可以在应用程序层检测和标记任何长时间运行的写入。

## 其他关于“w”的选项

`"majority"`不是唯一的`writeConcern`选项。MongoDB 还允许您通过向`"w"`传递数字来指定要复制到的任意数量的服务器，如下所示：

```
db.products.insertOne(
    { "_id": 10, "item": "envelopes", "qty": 100, type: "Self-Sealing" },
    { writeConcern: { "w" : 2, "wtimeout" : 100 } }
);
```

这将等待两个成员（主节点和一个从节点）写入完成。

注意，`"w"`值包括主节点。如果要将写入传播到``*`n`*``个从节点，您应将`"w"`设置为``*`n`*``+1（包括主节点）。设置``"`w`" : 1``与根本不传递`"w"`选项相同，因为它只是检查写入是否在主节点上成功。

使用字面数字的缺点是，如果复制集配置更改，您必须更改应用程序。

# 自定义复制保证

向集合中的大多数写入被认为是“安全的”。然而，某些集合可能具有更复杂的要求：您可能希望确保写入至少传播到每个数据中心的一个服务器或大多数非隐藏节点。复制集允许您创建自定义规则，可以将其传递给`"getLastError"`，以保证复制到所需的服务器组合。

## 确保每个数据中心有一个服务器

数据中心之间的网络问题比数据中心内部的问题要普遍得多，整个数据中心停电的可能性也比多个数据中心中的散落服务器停电的可能性要大得多。因此，您可能希望为写操作引入一些特定于数据中心的逻辑。在确认成功之前保证向每个数据中心写入数据意味着，如果写入后数据中心断电，其他每个数据中心都将至少有一份本地副本。

要设置此项，我们首先按数据中心对成员进行分类。我们通过向副本集配置中添加一个`"tags"`字段来完成这项任务：

```
> var config = rs.config()
> config.members[0].tags = {"dc" : "us-east"}
> config.members[1].tags = {"dc" : "us-east"}
> config.members[2].tags = {"dc" : "us-east"}
> config.members[3].tags = {"dc" : "us-east"}
> config.members[4].tags = {"dc" : "us-west"}
> config.members[5].tags = {"dc" : "us-west"}
> config.members[6].tags = {"dc" : "us-west"}
```

`"tags"`字段是一个对象，每个成员可以有多个标签。例如，它可能是`"us-east"`数据中心中的“高质量”服务器，此时我们希望`"tags"`字段如`{"dc": "us-east", "quality" : "high"}`。

第二步是通过在我们的副本集配置中创建一个`"getLastErrorModes"`字段来添加一条规则。名称`"getLastErrorModes"`在某种程度上是遗留的，因为在 MongoDB 2.6 之前，应用程序使用称为`"getLastError"`的方法来指定写入关注点。在副本配置中，对于`"getLastErrorModes"`，每个规则的形式为``"*`name`*" : {"*`key`*" : *`number`*}}``。`"*`name`*"`是规则的名称，应该以客户端可以理解的方式描述规则的作用，因为在调用`getLastError`时，他们将使用此名称。在本例中，我们可以称此规则为`"eachDC"`或者更抽象的名称，如`"user-level safe"`。

``"*`key`*"``字段是标签中的键字段，因此在本例中，它将是`"dc"`。*`number`*是需要满足此规则的组数。在本例中，*`number`*为 2（因为我们希望至少从`"us-east"`和`"us-west"`各选取一个服务器）。*`number`*始终表示“至少从每个*`number`*组中选取一个服务器”。

我们按以下方式向副本集配置中添加`"getLastErrorModes"`并重新配置以创建此规则：

```
> config.settings = {}
> config.settings.getLastErrorModes = [{"eachDC" : {"dc" : 2}}]
> rs.reconfig(config)
```

`"getLastErrorModes"`位于副本集配置的`"settings"`子对象中，其中包含几个集级可选设置。

现在我们可以为写操作使用这条规则：

```
db.products.insertOne(
    { "_id": 10, "item": "envelopes", "qty": 100, type: "Self-Sealing" },
    { writeConcern: { "w" : "eachDC", wtimeout : 1000 } }
);
```

请注意，规则与应用开发者有些抽象：他们不必知道“每个 DC”中有哪些服务器才能使用该规则，并且该规则可以更改而无需更改其应用程序。我们可以添加一个数据中心或更改成员集合，而应用程序无需了解这些更改。

## 保证大多数非隐藏成员

通常，隐藏成员有点像二等公民：你永远不会将故障切换到它们，它们肯定不会进行任何读取操作。因此，您可能只关心非隐藏成员是否收到写入，并让隐藏成员自行处理。

假设我们有五个成员，*host0* 到 *host4*，其中 *host4* 是隐藏成员。我们想确保大多数非隐藏成员有写入权限，即至少 *host0*、*host1*、*host2* 和 *host3* 中的三个。为此，首先我们为每个非隐藏成员分配自己的标签：

```
> var config = rs.config()
> config.members[0].tags = [{"normal" : "A"}]
> config.members[1].tags = [{"normal" : "B"}]
> config.members[2].tags = [{"normal" : "C"}]
> config.members[3].tags = [{"normal" : "D"}]
```

隐藏成员 *host4* 没有被分配标签。

现在我们为大多数这些服务器添加一个规则：

```
> config.settings.getLastErrorModes = [{"visibleMajority" : {"normal" : 3}}]
> rs.reconfig(config)
```

最后，我们可以在我们的应用程序中使用这个规则：

```
db.products.insertOne(
    { "_id": 10, "item": "envelopes", "qty": 100, type: "Self-Sealing" },
    { writeConcern: { "w" : "visibleMajority", wtimeout : 1000 } }
);
```

这将等待至少三个非隐藏成员进行写入。

## 创建其他的保证。

你可以创建的规则是没有限制的。记住创建自定义复制规则有两个步骤：

1.  通过分配键值对为成员打上标签。键描述分类，例如 `"data_center"`、`"region"` 或 `"serverQuality"`。值确定服务器在分类内属于哪个组。例如，对于键 `"data_center"`，你可能会有一些服务器标记为 `"us-east"`，一些标记为 `"us-west"`，还有其他的标记为 `"aust"`。

1.  基于你创建的分类创建一个规则。规则总是采用这种形式 ``{"*`name`*" : {"*`key`*" : *`number`*}}``，其中至少来自 *`number`* 组的一个服务器必须在成功之前写入。例如，你可以创建一个规则 `{"twoDCs" : {"data_center" : 2}}`，这意味着至少有两个被标记的数据中心中的一个服务器必须确认写入成功。

然后你可以在 `getLastErrorModes` 中使用这个规则。

规则是配置复制的强大方式，尽管理解和设置它们很复杂。除非你有相当复杂的复制需求，否则坚持使用 `"w" : "majority"` 是非常安全的选择。

# 向次要节点发送读请求

默认情况下，驱动程序将所有请求路由到主节点。这通常是你想要的，但你可以通过在驱动程序中设置读取偏好来配置其他选项。读取偏好允许你指定查询应该发送到哪些类型的服务器。

通常向次要节点发送读请求是一个不好的做法。在某些特定情况下，这样做是有意义的，但你应该在允许之前非常仔细地权衡利弊。本节涵盖了为什么这是一个不好的想法以及在何种特定条件下这样做是有意义的。

## 一致性考虑

需要强一致性读取的应用程序不应从次要节点读取。

辅助节点通常应与主节点相差几毫秒。然而，并没有这方面的保证。有时候，由于负载、配置错误、网络错误或其他问题，辅助节点可能会落后数分钟、数小时甚至数天。客户端库无法判断辅助节点的实时性，因此客户端可能会愉快地将查询发送到远远落后的辅助节点。可以隐藏辅助节点以防止客户端读取，但这是一个手动过程。因此，如果您的应用程序需要可预测的实时数据，就不应该从辅助节点读取。

如果您的应用程序需要读取自己的写入（例如，插入文档然后查询并找到它），则不应将读取操作发送到辅助节点（除非写入使用`"w"`等待所有辅助节点的复制，如前所示）。否则，应用程序可能会执行成功的写入操作，尝试读取该值，但却找不到它（因为它将读取操作发送到尚未复制的辅助节点）。客户端可以比复制操作更快地发出请求。

要始终将读取请求发送到主节点，请将读取首选项设置为`primary`（或者保持不变，因为`primary`是默认设置）。如果没有主节点，查询将报错。这意味着如果主节点宕机，则您的应用程序无法执行查询。但是，如果您的应用程序可以在故障转移或网络分区期间处理停机时间，或者不能接受获取过时数据，这绝对是一个可接受的选项。

## 负载考虑

许多用户将读取操作发送到辅助节点以分担负载。例如，如果您的服务器每秒只能处理 10,000 个查询，而您需要处理 30,000 个查询，您可能会设置几个辅助节点并让它们承担部分负载。然而，这种扩展的方式存在危险性，因为很容易意外地过载系统，并且一旦过载，很难从中恢复。

例如，假设您有刚才描述的情况：每秒 30,000 次读取。您决定创建一个包含四个成员的复制集（其中一个将配置为非投票成员，以防止选举中的平局）来处理此问题：每个辅助节点都远低于其最大负载，并且系统运行良好。

直到其中一个辅助节点崩溃。

现在剩下的每个成员都在处理他们可能的全部负载。如果需要重建崩溃的成员，可能需要从其他服务器中复制数据，从而使剩余的服务器负担过重。服务器过载通常会使其性能下降，进一步降低集合的容量，并迫使其他成员承担更多负载，导致它们陷入一个越来越慢的螺旋式崩溃。

过载还会导致复制速度减慢，使其余辅助节点落后。突然间，一个成员崩溃，一个成员滞后，而整个系统过载到没有任何余地。

如果你对服务器可以承受多大负载有一个良好的了解，你可能会觉得你可以更好地计划这一点：使用五台服务器而不是四台，即使有一台故障，整个系统也不会过载。然而，即使你计划得很完美（并且只丢失了你预期的服务器数量），你仍然需要处理其他服务器在比预期更大的压力下的情况。

更好的选择是使用分片来分发负载。我们将在第十四章中介绍如何设置分片。

## 从次要节点读取的原因

有几种情况下将应用程序读取发送到次要节点是合理的。例如，如果主节点宕机（而你不在乎这些读取可能有些陈旧），你可能希望你的应用程序仍然能执行读取操作。这是将读取分发到次要节点的最常见情况：当你的集群失去主节点时，你希望有一个临时的只读模式。这种读取偏好被称为`primaryPreferred`。

从次要节点读取的一个常见论点是获得低延迟读取。你可以将`nearest`作为你的读取偏好，根据从驱动程序到副本集成员的平均 ping 时间路由请求到最低延迟的成员。如果你的应用程序需要在多个数据中心中低延迟访问相同的文档，这是唯一的方法。然而，如果你的文档更加基于位置（此数据中心的应用服务器需要低延迟访问一些数据，或者另一个数据中心的应用服务器需要低延迟访问其他数据），这应该通过分片来完成。请注意，如果你的应用程序要求低延迟读取和写入，你必须使用分片：副本集只允许写入到一个位置（主节点所在位置）。

如果你从可能尚未复制所有写入的成员读取数据，你必须愿意牺牲一致性。或者，如果你希望等待写入已经复制到所有成员再进行写入，你可以牺牲写入速度。

如果你的应用程序可以接受任意陈旧的数据，你可以使用`secondary`或`secondaryPreferred`读取偏好。`secondary`总是将读取请求发送到次要节点。如果没有次要节点可用，它会报错而不是将读取请求发送到主节点。这可以用于那些不关心陈旧数据并且只想使用主节点进行写入的应用程序。如果你对数据的陈旧性有任何顾虑，不建议使用这种方法。

`secondaryPreferred`会将读取请求发送到次要节点（如果有）。如果没有次要节点可用，请求将被发送到主节点。

有时，读取负载与写入负载显著不同——即，您读取的数据完全不同于您写入的数据。您可能希望为离线处理设置数十个索引，这些索引不希望在主要数据库上存在。在这种情况下，您可能希望设置一个具有不同索引的次要数据库。如果您想要为此目的使用次要数据库，您可能会直接从驱动程序创建连接，而不是使用复制集连接。

考虑哪种选项适合您的应用程序。您还可以结合选项使用：如果某些读取请求必须来自主要数据库，请使用`primary`。如果您可以接受其他读取请求不具有最新数据，请为这些请求使用`primaryPreferred`。如果某些请求需要低延迟而不需要一致性，请为这些请求使用`nearest`。
