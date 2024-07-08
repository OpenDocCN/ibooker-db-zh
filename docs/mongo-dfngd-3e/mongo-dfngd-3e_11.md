# 第九章：应用程序设计

本章涵盖了如何有效地设计与 MongoDB 协同工作的应用程序。它讨论了：

+   架构设计考虑因素

+   决定是嵌入数据还是引用数据时的权衡

+   优化技巧

+   一致性考虑因素

+   如何迁移架构

+   如何管理架构

+   当 MongoDB 不适合作为数据存储时

# 架构设计考虑因素

数据表达的关键方面是架构的设计，即数据在文档中的表示方式。此设计的最佳方法是按照应用程序希望查看数据的方式进行表示。因此，与关系数据库不同，您需要在建模架构之前首先了解查询和数据访问模式。

设计架构时需要考虑的关键方面如下：

约束

您需要了解任何数据库或硬件限制。您还需要考虑 MongoDB 的一些特定方面，例如最大文档大小为 16 MB，完整文档从磁盘读取和写入，更新重写整个文档，以及原子更新是在文档级别进行的。

查询和写入的访问模式

您需要识别和量化应用程序及更广泛系统的工作负载。工作负载包括应用程序中的读取和写入。一旦您知道查询何时运行以及频率，您可以识别最常见的查询。这些查询是您需要设计架构来支持的查询。确定了这些查询后，您应尽量减少查询数量，并确保在设计中查询到的数据存储在同一文档中。

不在这些查询中使用的数据应放入不同的集合中。也应将很少使用的数据移至不同的集合中。是否可以将动态（读/写）数据与静态（主要是读取）数据分开值得考虑。优化架构设计以支持最常见的查询可以获得最佳性能结果。

关系类型

您应考虑与您应用需求相关的数据及文档间的关系。然后，您可以确定最佳的嵌入或引用数据或文档的方法。您需要解决如何引用文档而不需要执行额外的查询，以及在关系更改时更新多少文档。还必须考虑数据结构是否易于查询，例如支持建模特定关系的嵌套数组（数组中的数组）。

基数

一旦确定了文档和数据之间的关系，应考虑这些关系的基数。具体来说，它是一对一，一对多，多对多，一对百万还是多对十亿？确立关系的基数非常重要，以确保在 MongoDB 架构中使用最佳格式对其进行建模。您还应考虑在访问许多/百万侧的对象时是否单独访问或仅在父对象的上下文中访问，以及问题数据字段的更新与读取的比率。这些问题的答案将帮助您确定是否应嵌入文档或引用文档，以及是否应跨文档对数据进行去规范化。

## 架构设计模式

在 MongoDB 中，模式设计对应用程序性能直接影响重大。模式设计中有许多常见问题可以通过已知模式或“构建块”来解决。在模式设计中最佳实践是同时使用一个或多个这些模式。

可能适用的方案设计模式包括：

*多态模式*

这适用于集合中所有文档具有相似但不完全相同结构的情况。它涉及识别文档中支持应用程序将运行的常见查询的公共字段。跟踪文档或子文档中的特定字段有助于识别数据之间的差异，以及可以在应用程序中编码的不同代码路径或类/子类。这允许在不完全相同的文档集合中使用简单查询以提高查询性能。

*属性模式*

当文档中有一个子集的字段共享公共特征，并且您想要对其进行排序或查询时，或者需要进行排序的字段仅存在于文档的子集中，或者这两种情况同时成立时，适合使用此模式。它涉及将数据重新塑造为键/值对数组，并在此数组的元素上创建索引。修饰符可以作为这些键/值对的附加字段添加。此模式有助于针对每个文档的许多类似字段，从而减少所需的索引并简化查询。

*桶模式*

此适用于时间序列数据，其中数据作为一段时间内的流捕获。在 MongoDB 中，将此数据“分桶”为一组文档，每个文档都包含特定时间范围内的数据，比创建每个时间点/数据点的文档要有效得多。例如，您可以使用一个小时的“桶”，将该小时内的所有读数放入单个文档中的数组中。文档本身将具有指示此“桶”覆盖的时间段的开始和结束时间。

*异常模式*

处理偶尔的文档查询超出应用程序正常模式的罕见情况。这是为受欢迎度影响的高级架构模式设计的情况。这可以在社交网络中看到，例如重要影响者、图书销售、电影评论等。它使用一个标志来指示文档是异常值，并将额外的内容存储到一个或多个文档中，这些文档通过`"_id"`引用回第一个文档。标志将由您的应用程序代码使用，以获取溢出文档。

*计算模式*

当数据需要频繁计算时使用，也可以在数据访问模式为读取密集型时使用。这种模式建议在后台进行计算，并定期更新主文档。这样可以有效地近似计算字段或文档，而无需为每个查询持续生成这些内容。这样可以显著减少 CPU 的负担，避免重复计算相同的内容，特别是在读操作触发计算且读写比高的情况下。

*子集模式*

当您的工作集超过机器可用 RAM 时使用。这可能是由包含许多信息但未被应用程序使用的大型文档引起的。这种模式建议将频繁使用的数据和不经常使用的数据分割到两个单独的集合中。典型的例子可能是电子商务应用程序在“主”（经常访问）集合中保留产品的最近 10 条评论，并将所有较旧的评论移动到第二个集合中，仅在应用程序需要超过最近 10 条评论时才查询该集合。

*扩展引用模式*

用于存在许多不同逻辑实体或“物件”，每个实体都有自己的集合，但您可能希望为特定功能将这些实体聚集在一起的情景。典型的电子商务架构可能为订单、客户和库存分别设置不同的集合。当我们想要从这些单独的集合中汇总单个订单的所有信息时，这可能会对性能产生负面影响。解决方案是识别经常访问的字段，并在订单文档内部进行重复。对于电子商务订单而言，这可能是我们要将商品发运给的客户的姓名和地址。这种模式通过数据重复换取减少汇总信息所需的查询数量。

*近似模式*

对于需要资源密集型（时间、内存、CPU 周期）计算但精确性并非绝对必需的情况，这是非常有用的。例如，像/赞数或页面访问计数器这样的图片或帖子，在这些情况下，应用这种模式可以大大减少写入数量，例如每 100 次或更多次浏览后才更新计数器，而不是每次浏览都更新。

*树模式*

当您有大量查询且数据主要是分层结构时，可以应用此模式。它遵循将通常一起查询的数据存储在一起的早期概念。在 MongoDB 中，您可以在同一文档中的数组中轻松存储层次结构。在电子商务网站的产品目录示例中，通常有属于多个类别或属于其他类别的类别的产品。例如，“硬盘”本身是一个类别，但属于“存储”类别，后者又属于“计算机配件”类别，后者又是“电子产品”类别的一部分。在这种情况下，我们将有一个字段来跟踪整个层次结构，并且另一个字段将保留直接类别（“硬盘”）。在这些值上使用多键索引，可以确保轻松找到与层次结构中类别相关的所有项目。直接类别字段允许找到与此类别直接相关的所有项目。

*预分配模式*

虽然最初主要用于 MMAP 存储引擎，但现在仍有此模式的用途。该模式建议创建一个初始空结构，稍后将其填充。例如，用于管理按天管理资源的预订系统，跟踪资源是否空闲或已预订/不可用。资源（*x*）和天数（*y*）的二维结构使得检查可用性和执行计算非常简单。

*文档版本化模式*

这提供了一种机制，可以保留文档的旧版本。需要向每个文档添加一个额外的字段来跟踪“主”集合中的文档版本，以及一个包含所有文档修订版本的附加集合。这种模式有一些假设，特别是每个文档有限数量的修订版本，需要版本化的文档数量不多，并且主要查询都是在每个文档的当前版本上进行。在这些假设不成立的情况下，您可能需要修改模式或考虑不同的模式设计方案。

MongoDB 在线提供了关于模式和架构设计的多个有用资源。MongoDB University 提供了一门免费课程，[M320 数据建模](https://oreil.ly/BYtSr)，以及一个[“使用模式构建”博客系列](https://oreil.ly/MjSld)。

# 规范化与去规范化

有许多表示数据的方式，其中一个最重要的问题是考虑数据的规范化程度。*规范化*指的是将数据分成多个集合，并在集合之间建立引用关系。每个数据片段只存在于一个集合中，尽管可能有多个文档引用它。因此，要更改数据，只需更新一个文档。MongoDB 聚合框架提供了带有 `$lookup` 阶段的连接功能，通过将“加入”集合中具有源集合匹配文档的文档添加到“加入”集合中，向每个匹配文档添加一个新的数组字段，其中包含来自源集合的文档的详细信息。这些重塑后的文档随后可以在下一个阶段中进一步处理。

*去规范化*是规范化的相反：将所有数据嵌入到单个文档中。而不是文档包含对数据的一份定义性拷贝的引用，许多文档可能包含数据的拷贝。这意味着如果信息更改，需要更新多个文档，但可以使用单个查询获取所有相关数据。

决定何时规范化和何时去规范化可能很困难：通常规范化使写操作更快，而去规范化使读操作更快。因此，您需要决定哪些权衡对您的应用程序是合理的。

## 数据表示示例

假设我们正在存储关于学生和他们所选课程的信息。一种表示方法是有一个*students*集合（每个学生是一个文档），一个*classes*集合（每门课程是一个文档）。然后我们可以有一个第三个集合（*studentClasses*），其中包含对学生和他们所选课程的引用：

```
> db.studentClasses.findOne({"studentId" : id})
{
    "_id" : ObjectId("512512c1d86041c7dca81915"),
    "studentId" : ObjectId("512512a5d86041c7dca81914"),
    "classes" : [
        ObjectId("512512ced86041c7dca81916"),
        ObjectId("512512dcd86041c7dca81917"),
        ObjectId("512512e6d86041c7dca81918"),
        ObjectId("512512f0d86041c7dca81919")
    ]
}
```

如果你熟悉关系数据库，可能以前见过这种连接表（虽然通常每个文档只有一个学生和一个课程，而不是一个课程`"_id"`列表）。在 MongoDB 中，将课程放入数组可能更合适一些，但通常不建议以这种方式存储数据，因为这样需要大量查询才能获取实际信息。

假设我们想要找出一个学生正在上的课程。我们将在*students*集合中查询学生，在*studentClasses*中查询课程`"_id"`，然后在*classes*集合中查询课程信息。因此，查找这些信息将需要向服务器发送三次请求。通常情况下，这不是在 MongoDB 中结构化数据的方式，除非课程和学生经常变化且读取数据不需要快速完成。

我们可以通过在学生文档中嵌入类引用来消除一个解引用查询：

```
{
    "_id" : ObjectId("512512a5d86041c7dca81914"),
    "name" : "John Doe",
    "classes" : [
        ObjectId("512512ced86041c7dca81916"),
        ObjectId("512512dcd86041c7dca81917"),
        ObjectId("512512e6d86041c7dca81918"),
        ObjectId("512512f0d86041c7dca81919")
    ]
}
```

`"classes"`字段保留 John Doe 正在上的班级的`"_id"`数组。当我们想要了解这些班级的信息时，可以使用这些`"_id"`在*classes*集合中进行查询。这只需要两个查询。这是一种相当流行的数据结构方式，不需要立即访问且变化不频繁但也不是一成不变的数据。

如果我们需要进一步优化读取，我们可以通过完全去归一化数据并将每个班级作为嵌入文档存储在`"classes"`字段中来一次性获取所有信息的查询：

```
{
    "_id" : ObjectId("512512a5d86041c7dca81914"),
    "name" : "John Doe",
    "classes" : [
        {
            "class" : "Trigonometry",
            "credits" : 3,
            "room" : "204"
        },
        {
            "class" : "Physics",
            "credits" : 3,
            "room" : "159"
        },
        {
            "class" : "Women in Literature",
            "credits" : 3,
            "room" : "14b"
        },
        {
            "class" : "AP European History",
            "credits" : 4,
            "room" : "321"
        }
    ]
}
```

这样做的好处是只需要一个查询即可获取信息。缺点是占用更多空间且更难保持同步。例如，如果事实证明物理应该值四学分（而不是三学分），则物理班的每个学生都需要更新他们的文档（而不仅仅是更新一个中心的“物理”文档）。

最后，你可以使用之前提到的扩展引用模式，这是嵌入和引用的混合体——你创建一个包含常用信息的子文档数组，但同时有一个指向实际文档的引用以获取更多信息：

```
{
    "_id" : ObjectId("512512a5d86041c7dca81914"),
    "name" : "John Doe",
    "classes" : [
        {
            "_id" : ObjectId("512512ced86041c7dca81916"),
            "class" : "Trigonometry"
        },
        {
            "_id" : ObjectId("512512dcd86041c7dca81917"),
            "class" : "Physics"
        },
        {
            "_id" : ObjectId("512512e6d86041c7dca81918"),
            "class" : "Women in Literature"
        },
        {
            "_id" : ObjectId("512512f0d86041c7dca81919"),
            "class" : "AP European History"
        }
    ]
}
```

这种方法也是一个不错的选择，因为随着需求的变化，嵌入的信息量随时间可以改变：如果你想在页面中包含更多或更少的信息，可以在文档中嵌入更多或更少的信息。

另一个重要的考虑因素是这些信息变化的频率，以及它们被阅读的频率。如果它会经常更新，那么归一化是个好主意。然而，如果变化不频繁，那么优化更新过程可能对应用程序的每次读取都是有代价的，收益较少。

例如，教科书归一化的一个用例是将用户及其地址存储在单独的集合中。然而，人们的地址很少更改，所以通常不应因为某人可能搬迁而对每次读取进行惩罚。你的应用程序应该将地址嵌入到用户文档中。

如果决定使用嵌入式文档并需要更新它们，应设置定期任务以确保成功地将任何更新传播到每个文档。例如，假设你尝试进行多次更新，但服务器在所有文档更新完成之前崩溃。你需要一种方法来检测并重试更新。

在更新操作符方面，`"$set"`是幂等的，但`"$inc"`则不是。幂等操作无论尝试一次还是多次，结果都相同；在网络错误的情况下，重试操作足以使更新发生。对于不是幂等的操作符，操作应分为两个分别是幂等且可以安全重试的操作。这可以通过在第一个操作中包含唯一的待处理令牌，并在第二个操作中同时使用唯一键和唯一的待处理令牌来实现。这种方法允许`"$inc"`是幂等的，因为每个单独的`updateOne`操作都是幂等的。

在一定程度上，生成的信息越多，嵌入的内容就越少。如果嵌入字段的内容或嵌入字段的数量可能会无限增长，则通常应该使用引用而不是嵌入。像评论树或活动列表这样的内容应存储为它们自己的文档，而不是嵌入其中。还值得考虑使用子集模式（在“模式设计模式”中描述）来存储最新的项目（或其他子集）在文档中。

最后，包含的字段应与文档中的数据完整相关。如果在查询文档时，一个字段几乎总是被排除在结果之外，这表明它可能属于另一个集合。这些准则在表 9-1 中总结。

表 9-1\. 嵌入与引用的比较

| 嵌入更适合... | 引用更适合... |
| --- | --- |
| 小型子文档 | 大型子文档 |
| 不经常更改的数据 | 易变数据 |
| 当接受最终一致性时 | 当需要立即一致性时 |
| 少量增长的文档 | 大量增长的文档 |
| 您经常需要执行第二个查询来获取的数据 | 您经常在结果中排除的数据 |
| 快速读取 | 快速写入 |

假设我们有一个*users*集合。以下是可能包含在用户文档中的一些示例字段及其是否应该嵌入的指示：

账户偏好

这些只与此用户文档相关，并且可能会与文档中的其他用户信息一起公开。账户偏好通常应该被嵌入。

最近活动

这取决于最近活动的增长和变化情况。如果它是一个固定大小的字段（例如，最近的 10 个事物），可能有必要将这些信息嵌入或实现子集模式。

好友

通常不应完全嵌入此类信息，或者至少不应完全嵌入。参见“好友、关注者和其他不便之处”。

此用户产生的所有内容

这不应该被嵌入。

## 基数

*基数*表示一个集合对另一个集合的引用数量。常见的关系是一对一、一对多或多对多。例如，假设我们有一个博客应用程序。每篇*文章*都有一个*标题*，因此这是一对一关系。每个*作者*有多篇*文章*，所以这是一对多关系。*文章*有多个*标签*，*标签*也指向多篇*文章*，因此这是多对多关系。

在使用 MongoDB 时，将“许多”概念上分为“许多”和“少”。例如，您可能在作者和文章之间有一个一对少的关系：每个作者只写几篇文章。您可能在博客文章和标签之间有一个多对少的关系：您可能拥有比标签更多的博客文章。但是，您可能在博客文章和评论之间有一个一对多的关系：每篇文章有许多评论。

确定少关系与多关系可以帮助您决定何时嵌入何时引用。通常情况下，“少”关系与嵌入效果更好，“多”关系与引用效果更好。

## 朋友、粉丝和其他不便之处

> 让朋友近在咫尺，而敌人则深藏不露。

本节涵盖社交图数据的考虑因素。许多社交应用程序需要连接人员、内容、关注者、朋友等。找到如何平衡嵌入和引用这些高度连接信息可能有些棘手，但通常遵循、成为朋友或收藏可以简化为发布/订阅系统：一个用户订阅另一个用户的通知。因此，有两种基本操作需要高效处理：存储订阅者和通知所有感兴趣的人某个事件。

人们通常有三种方式实现订阅。第一种选择是将生产者放入订阅者的文档中，看起来像这样：

```
{
    "_id" : ObjectId("51250a5cd86041c7dca8190f"),
    "username" : "batman",
    "email" : "batman@waynetech.com"
    "following" : [
        ObjectId("51250a72d86041c7dca81910"), 
        ObjectId("51250a7ed86041c7dca81936")
    ]
}
```

现在，给定用户的文档，您可以发出以下查询来查找他们可能感兴趣的所有已发布活动：

```
db.activities.find({"user" : {"$in" :
      user["following"]}})
```

然而，如果您需要找到所有对新发布的活动感兴趣的人，您将不得不跨所有用户查询`"following"`字段。

或者，您可以将关注者附加到生产者的文档中，就像这样：

```
{
    "_id" : ObjectId("51250a7ed86041c7dca81936"),
    "username" : "joker",
    "email" : "joker@mailinator.com"
    "followers" : [
        ObjectId("512510e8d86041c7dca81912"),
        ObjectId("51250a5cd86041c7dca8190f"),
        ObjectId("512510ffd86041c7dca81910")
    ]
}
```

每当此用户执行某些操作时，所有需要通知的用户都在这里。缺点是现在您需要查询整个*用户*集合来找到每个用户关注的所有人（与前一种情况相反的限制）。

这两个选项都带来了一个额外的缺点：它们使您的用户文档变得更大并且更不稳定。`"following"`（或`"followers"`）字段通常甚至不需要返回：您想多频繁地列出每个关注者？因此，最终的选项通过进一步归一化甚至存储在另一个集合中的订阅来抵消这些缺点。彻底归一化通常是过度的，但对于经常与其余文档一起返回的极不稳定的字段可能是有用的。`"followers"`可能是以这种方式彻底归一化的一个明智的字段。

在这种情况下，您需要维护一个将发布者与订阅者匹配的集合，并且文档看起来像这样：

```
{
    "_id" : ObjectId("51250a7ed86041c7dca81936"), // followee's "_id"
    "followers" : [
        ObjectId("512510e8d86041c7dca81912"),
        ObjectId("51250a5cd86041c7dca8190f"),
        ObjectId("512510ffd86041c7dca81910")
    ]
}
```

这样可以保持您的用户文档简洁，但意味着需要额外的查询来获取关注者。

### 处理威尔·惠特恩效应

无论您使用哪种策略，只嵌入有限数量的子文档或引用是可行的。如果您有名人用户，他们可能会溢出您存储关注者的任何文档。补偿此问题的典型方法是使用“模式设计模式”中讨论的异常模式，并在必要时使用“续集”文档。例如，您可能会有：

```
> db.users.find({"username" : "wil"})
{
    "_id" : ObjectId("51252871d86041c7dca8191a"),
    "username" : "wil",
    "email" : "wil@example.com",
    "tbc" : [
        ObjectId("512528ced86041c7dca8191e"),
        ObjectId("5126510dd86041c7dca81924")
    ]
    "followers" : [
        ObjectId("512528a0d86041c7dca8191b"),
        ObjectId("512528a2d86041c7dca8191c"),
        ObjectId("512528a3d86041c7dca8191d"),
        ...
    ]
}
{
    "_id" : ObjectId("512528ced86041c7dca8191e"),
    "followers" : [
        ObjectId("512528f1d86041c7dca8191f"),
        ObjectId("512528f6d86041c7dca81920"),
        ObjectId("512528f8d86041c7dca81921"),
        ...
    ]
}
{
    "_id" : ObjectId("5126510dd86041c7dca81924"),
    "followers" : [
        ObjectId("512673e1d86041c7dca81925"),
        ObjectId("512650efd86041c7dca81922"),
        ObjectId("512650fdd86041c7dca81923"),
        ...
    ]
}
```

然后添加应用逻辑来支持获取“待续”(`"tbc"`)数组中的文档。

# 数据操作的优化

要优化您的应用程序，您必须首先通过评估其读取和写入性能来确定其瓶颈所在。优化读取通常涉及拥有正确的索引并尽可能返回单个文档中的尽可能多的信息。优化写入通常涉及最小化索引数量并尽可能高效地进行更新。

通常在为快速写入而优化的模式和为快速读取而优化的模式之间存在权衡，因此您可能必须决定对于您的应用程序来说哪个更重要。不仅要考虑读取与写入的重要性，还要考虑它们的比例：如果写入更重要，但您每次写入都进行了一千次读取，您可能仍然希望优先优化读取。

## 删除旧数据

有些数据只在短时间内重要：几周或几个月后，它只会浪费存储空间。有三种流行的方法可以删除旧数据：使用固定大小集合（capped collections）、使用 TTL 集合和按时间段删除集合。

最简单的选项是使用固定大小集合：将其设置为一个大尺寸，让旧数据“掉落”到末尾。然而，固定大小集合对您可以执行的操作有一定的限制，并且容易受到流量峰值的影响，暂时降低它们可以保持的时间长度。更多信息请参见“固定大小集合”。

第二个选项是使用 TTL 集合。这可以让您更精细地控制文档何时被移除，但对于写入量非常高的集合可能不够快速：它通过遍历 TTL 索引来移除文档，方式与用户请求的移除相同。但是，如果 TTL 集合能够跟得上，这可能是最容易实现的解决方案。有关 TTL 索引的更多信息，请参阅“生存时间索引”。

最终的选项是使用多个集合：例如，每个月一个集合。每次月份变更时，您的应用程序开始使用本月的（空）集合，并在当前和前几个月的集合中搜索数据。一旦一个集合超过了，比如说，六个月，您可以丢弃它。这种策略可以应对几乎任何量的流量，但构建应用程序会更加复杂，因为您需要使用动态集合（或数据库）名称，并可能查询多个数据库。

# 规划数据库和集合

一旦您勾画出您的文档的外观，您必须决定将它们放在哪些集合或数据库中。这通常是一个相当直观的过程，但有一些指导原则需要牢记。

通常情况下，具有相似模式的文档应放在同一个集合中。MongoDB 通常不允许将来自多个集合的数据合并，因此如果有需要一起查询或聚合的文档，这些文档适合放在一个大集合中。例如，您可能有形状相当不同的文档，但如果要对它们进行聚合，它们应该都存在于同一个集合中（或者如果它们在不同的集合或数据库中，则可以使用`$merge`阶段）。

对于集合，需要考虑的主要问题是锁定（每个文档获取读/写锁）和存储。一般来说，如果您有高写入工作负载，可能需要考虑使用多个物理卷来减少 I/O 瓶颈。当您使用`--directoryperdb`选项时，每个数据库可以驻留在自己的目录中，允许您将不同的数据库挂载到不同的卷上。因此，您可能希望数据库中的所有项具有类似的“质量”，具有类似的访问模式或类似的流量水平。

例如，假设您有一个应用程序，包括几个组件：一个创建大量非常有价值数据的日志组件，一个用户集合，以及几个用于用户生成数据的集合。这些集合都是高价值的：用户数据的安全非常重要。还有一个用于社交活动的高流量集合，重要性较低，但不像日志那么不重要。这个集合主要用于用户通知，因此几乎是一个追加记录的集合。

将这些按重要性分开，你可能会得到三个数据库：*日志*、*活动*和*用户*。这种策略的好处在于，你可能会发现你的最有价值的数据也是你最少拥有的（例如，用户生成的数据可能不如日志那么多）。你可能无法为整个数据集买 SSD，但你可能可以为用户买一个，或者你可以为用户使用 RAID10，而为日志和活动使用 RAID0。

请注意，在 MongoDB 4.2 之前及聚合框架中引入`$merge`操作符之前，使用多个数据库存在一些限制，该操作允许将聚合的结果从一个数据库存储到不同数据库和该数据库内的不同集合。另一个需要注意的点是，在将现有集合从一个数据库复制到另一个数据库时，使用`renameCollection`命令速度较慢，因为它必须将所有文档复制到新数据库。

# 管理一致性

您必须弄清楚您的应用程序读取数据需要多一致。MongoDB 支持多种一致性级别，从始终能够读取您自己的写入数据到读取未知数据的旧度。如果您正在报告过去一年的活动情况，您可能只需要正确到过去几天的数据。相反，如果您正在进行实时交易，您可能需要立即读取最新的写入数据。

要了解如何实现这些不同级别的一致性，了解 MongoDB 在幕后的操作是很重要的。服务器保持每个连接的请求队列。当客户端发送请求时，它将被放置在其连接队列的末尾。连接上的任何后续请求将在先前入队的操作处理后发生。因此，单个连接具有数据库的一致视图，并且始终可以读取其自己的写入数据。

请注意，这是一个连接队列：如果我们打开两个 shell，我们将有两个连接到数据库。如果我们在一个 shell 中进行插入操作，在另一个 shell 中进行后续查询可能不会返回插入的文档。然而，在单个 shell 中，如果我们在插入后查询文档，文档将被返回。这种行为在手动复制时可能难以复制，但在繁忙的服务器上，插入和查询可能交错发生。开发人员经常遇到这种情况，当他们在一个线程中插入数据然后在另一个线程中检查是否成功插入时。在某个瞬间，看起来数据没有插入，然后突然出现了。

当使用 Ruby、Python 和 Java 驱动程序时，特别需要注意这种行为，因为这三种语言都使用连接池。为了效率，这些驱动程序会打开多个连接（一个*池*）到服务器，并在它们之间分发请求。然而，它们都有机制来确保一系列请求由单个连接处理。有关各种语言的连接池的详细文档在[MongoDB 驱动程序连接监控和池化规范](https://oreil.ly/nAt9i)中有所介绍。

当你将读操作发送到复制集的次要节点时（参见第十二章），这就成为一个更大的问题。次要节点可能落后于主节点，导致读取数据是几秒、几分钟甚至几小时前的数据。有几种方法可以处理这个问题，最简单的方法是如果你关心数据的陈旧性，就把所有读取操作都发送到主节点。

MongoDB 提供了`readConcern`选项来控制所读取数据的一致性和隔离特性。它可以与`writeConcern`结合使用，以控制向你的应用程序所做的一致性和可用性保证。有五个级别："local"、"available"、"majority"、"linearizable"和"snapshot"。根据应用程序的不同，如果你希望避免读取数据的陈旧性，可以考虑使用"majority"，它仅返回已被大多数复制集成员确认的持久化数据，并且不会被回滚。"linearizable"也是一个选择：它返回反映了所有成功的多数确认写操作完成之前的数据。在返回带有"linearizable"的`readConcern`结果之前，MongoDB 可能会等待并发执行的写操作完成。

MongoDB 的三位资深工程师在 2019 年的 PVLDB 会议上发表了一篇名为[“MongoDB 中的可调一致性”](https://oreil.ly/PfcBx)的论文。^1 本文概述了用于复制的不同 MongoDB 一致性模型及应用程序开发人员如何利用这些模型。

# 迁移模式

随着应用程序的增长和需求的变化，你的模式可能也需要增长和变化。有几种方法可以实现这一点，但无论你选择哪种方法，你都应该仔细记录应用程序使用过的每个模式。理想情况下，你应该考虑是否适用于文档版本控制模式（参见“模式设计模式”）。

最简单的方法是随着应用程序的需要使模式逐步演变，确保应用程序支持所有旧版本的模式（例如，接受字段的存在或不存在，或优雅地处理多种可能的字段类型）。但是，如果您有冲突的模式版本，这种技术可能会变得混乱。例如，一个版本可能需要一个`"mobile"`字段，另一个版本可能需要*没有*`"mobile"`字段，而是需要一个不同的字段，而另一个版本可能将`"mobile"`字段视为可选。跟踪这些变化的需求可能会逐渐使您的代码变得混乱不堪。

为了以稍微结构化的方式处理变化的需求，您可以在每个文档中包含一个`"version"`字段（或仅使用`"v"`），并使用它来确定应用程序将接受的文档结构。这样可以更严格地执行您的模式：文档必须对某个模式版本有效，即使不是当前版本。然而，它仍然需要支持旧版本。

最后的选择是在模式更改时迁移所有数据。通常这不是一个好主意：MongoDB 允许您具有动态模式以避免迁移，因为它们会给系统施加很大压力。但是，如果您决定更改每个文档，您需要确保所有文档都已成功更新。MongoDB 支持*事务*，支持此类迁移。如果 MongoDB 在事务中途中崩溃，则会保留旧模式。

# 管理模式

MongoDB 在 3.2 版本中引入了模式验证，允许在更新和插入期间进行验证。在 3.6 版本中，它通过`$jsonSchema`操作符添加了 JSON 模式验证，现在建议在 MongoDB 中使用此方法进行所有模式验证。截至撰写本文时，MongoDB 支持 JSON 模式的草案 4，但请查阅文档获取此功能的最新信息。

验证在修改现有文档之前不会检查它们，并且配置是针对每个集合的。要向现有集合添加验证，可以使用`collMod`命令，并使用`validator`选项。在使用`db.createCollection()`时，通过指定`validator`选项可以向新集合添加验证。MongoDB 还提供了两个额外的选项，`validationLevel`和`validationAction`。`validationLevel`确定在更新现有文档期间应用验证规则的严格程度，而`validationAction`决定在出现错误加拒绝或警告加允许非法文档的情况下应该采取的操作。

# 不适合使用 MongoDB 的情况

虽然 MongoDB 是一种通用的数据库，在大多数应用中表现良好，但它并非万能。有几个原因可能需要避免使用它：

+   加入多种不同类型的数据以及跨多个不同维度进行连接是关系数据库擅长的事情。MongoDB 并不擅长这样做，并且很可能永远也不会。

+   使用关系数据库而不是 MongoDB 的一个重要（如果幸运的话是暂时的）原因是，如果你正在使用不支持 MongoDB 的工具。从 SQLAlchemy 到 WordPress，有成千上万的工具只是不支持 MongoDB。支持 MongoDB 的工具数量正在增加，但其生态系统远不及关系数据库的庞大。

^(1) 作者是复制高级软件工程师威廉·舒尔茨；复制团队团队负责人特斯·阿维塔比勒；以及分布式系统产品经理艾莉森·卡布拉尔。