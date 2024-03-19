# 第四章：操作系统和硬件优化

你的 MySQL 服务器的性能只能和它最弱的环节一样好，而运行 MySQL 的操作系统和硬件通常是限制因素。磁盘大小、可用内存和 CPU 资源、网络以及连接它们的所有组件都限制了系统的最终容量。因此，你需要仔细选择硬件，并适当配置硬件和操作系统。例如，如果你的工作负载受到 I/O 限制，一种方法是设计你的应用程序以最小化 MySQL 的 I/O 工作负载。然而，升级 I/O 子系统、安装更多内存或重新配置现有磁盘通常更明智。如果你在云托管环境中运行，本章的信息仍然非常有用，特别是为了了解文件系统限制和 Linux I/O 调度程序。

# 什么限制了 MySQL 的性能？

许多不同的硬件组件可以影响 MySQL 的性能，但我们经常看到的最常见的瓶颈是 CPU 耗尽。当 MySQL 尝试并行执行太多查询或较少数量的查询在 CPU 上运行时间过长时，CPU 饱和就会发生。

I/O 饱和仍然可能发生，但发生频率要比 CPU 耗尽低得多。这在很大程度上是因为过渡到使用固态硬盘（SSD）。从历史上看，不再在内存中工作而转向硬盘驱动器（HDD）的性能惩罚是极端的。SSD 通常比 SSH 快 10 到 20 倍。如今，如果查询需要访问磁盘，你仍然会看到它们的性能不错。

内存耗尽仍然可能发生，但通常只会在尝试为 MySQL 分配过多内存时发生。我们在“配置内存使用”中讨论了防止这种情况发生的最佳配置设置，在第五章中。

# 如何为 MySQL 选择 CPU

当升级当前硬件或购买新硬件时，你应该考虑你的工作负载是否受 CPU 限制。你可以通过检查 CPU 利用率来确定工作负载是否受 CPU 限制，但不要只看整体 CPU 负载有多重，而是要看你最重要的查询的 CPU 使用率和 I/O 的平衡，并注意 CPU 是否均匀负载。

广义上说，你的服务器有两个目标：

低延迟（快速响应时间）

要实现这一点，你需要快速的 CPU，因为每个查询只会使用一个 CPU。

高吞吐量

如果你可以同时运行多个查询，你可能会从多个 CPU 为查询提供服务中受益。

如果你的工作负载没有利用所有的 CPU，MySQL 仍然可以利用额外的 CPU 执行后台任务，如清理 InnoDB 缓冲区、网络操作等。然而，与执行查询相比，这些工作通常较小。

# 平衡内存和磁盘资源

拥有大量内存的主要原因并不是为了能够在内存中保存大量数据：最终目的是为了避免磁盘 I/O，因为磁盘 I/O 比在内存中访问数据慢几个数量级。关键是平衡内存和磁盘大小、速度、成本和其他特性，以便为你的工作负载获得良好的性能。

## 缓存、读取和写入

如果你有足够的内存，你可以完全隔离磁盘免受读取请求。如果所有数据都适合内存，一旦服务器的缓存被热起来，每次读取都会是缓存命中。仍然会有来自内存的逻辑读取，但没有来自磁盘的物理读取。然而，写入是另一回事。写入可以像读取一样在内存中执行，但迟早它必须写入磁盘以便永久保存。换句话说，缓存可以延迟写入，但不能像读取那样消除写入。

实际上，除了允许延迟写入外，缓存还可以以两种重要的方式将它们分组在一起：

多写一次刷新

一条数据可以在内存中多次更改，而不需要将所有新值都写入磁盘。当数据最终刷新到磁盘时，自上次物理写入以来发生的所有修改都是永久的。例如，许多语句可以更新一个内存中的计数器。如果计数器递增了一百次然后写入磁盘，一百次修改已经被合并为一次写入。

I/O 合并

许多不同的数据可以在内存中被修改，并且修改可以被收集在一起，以便可以将物理写入作为单个磁盘操作执行。

这就是为什么许多事务系统使用预写式日志策略。预写式日志允许它们在内存中对页面进行更改而不刷新更改到磁盘，这通常涉及随机 I/O 并且非常慢。相反，它们将更改的记录写入顺序日志文件，这样做要快得多。后台线程可以稍后将修改的页面刷新到磁盘；当它这样做时，它可以优化写入。

写入受益于缓冲，因为它将随机 I/O 转换为更多的顺序 I/O。异步（缓冲）写入通常由操作系统处理，并且会被批处理，以便更优化地刷新到磁盘。同步（非缓冲）写入必须在完成之前写入磁盘。这就是为什么它们受益于在冗余磁盘阵列（RAID）控制器的电池支持写回缓存中进行缓冲（我们稍后会讨论 RAID）。

## 你的工作集是什么？

每个应用程序都有一个“工作集”数据，即它真正需要完成工作的数据。许多数据库还有很多不在工作集中的数据。你可以把数据库想象成一个带有文件抽屉的办公桌。工作集包括你需要放在桌面上以完成工作的文件。在这个类比中，桌面代表主内存，而文件抽屉代表硬盘。就像你不需要把*每一张*纸都放在桌面上才能完成工作一样，你不需要整个数据库都适合内存以获得最佳性能——只需要工作集。

在处理 HDD 时，寻找有效的内存到磁盘比例是一个好的做法。这在很大程度上是由于 HDD 的较慢的延迟和低的每秒输入/输出操作数（IOPS）。使用 SSD 时，内存到磁盘比例变得不那么重要。

# 固态存储

固态（闪存）存储是大多数数据库系统的标准，特别是在线事务处理（OLTP）。只有在非常大的数据仓库或传统系统中才会通常找到 HDD。这种转变是因为 2015 年左右 SSD 的价格显著下降。

固态存储设备使用由单元组成的非易失性闪存存储芯片，而不是磁盘盘片。它们也被称为*非易失性随机存取存储器（NVRAM）*。它们没有移动部件，这使它们的行为与硬盘非常不同。

以下是闪存性能的简要总结。高质量的闪存设备具有：

与硬盘相比，随机读写性能要好得多

闪存设备通常在读取方面比写入更好。

比硬盘更好的顺序读写性能

然而，与随机 I/O 相比，并没有那么显著的改进，因为硬盘在随机 I/O 方面比顺序 I/O 慢得多。

比硬盘更好的并发性支持

闪存设备可以支持更多的并发操作，事实上，只有在有很多并发时它们才能真正达到最高吞吐量。

最重要的是随机 I/O 和并发性能的改进。闪存给您提供了在高并发情况下非常好的随机 I/O 性能。

## 闪存存储概述

有旋转盘片和摆动磁头的硬盘具有固有的限制和特性，这些特性是物理学所涉及的结果。固态存储也是如此，它是建立在闪存之上的。不要以为固态存储很简单。在某些方面，它实际上比硬盘更复杂。闪存的限制相当严重且难以克服，因此典型的固态设备具有复杂的架构，包含许多抽象、缓存和专有的“魔法”。

闪存的最重要特性是它可以快速多次读取小单位，但写入要困难得多。一个单元不能在没有特殊擦除操作的情况下重写，并且只能在大块中擦除，例如 512 KB。擦除周期很慢，最终会使块磨损。一个块可以容忍的擦除周期数量取决于它使用的基础技术——稍后会详细介绍。

写入的限制是固态存储复杂性的原因。这就是为什么一些设备提供稳定、一致的性能，而其他设备则不提供。这些“魔法”都在专有的固件、驱动程序和其他组件中，使固态设备运行起来。为了使写入操作性能良好并避免过早磨损闪存块，设备必须能够重新定位页面并执行垃圾回收和所谓的*磨损均衡*。术语*写入放大*用于描述由于将数据从一个地方移动到另一个地方而导致的额外写入，由于部分块写入而多次写入数据和元数据。

## 垃圾回收

垃圾回收是很重要的。为了保持一些块的新鲜度并为新的写入做好准备，设备会回收块。这需要设备上的一些空闲空间。设备要么会有一些您看不到的内部保留空间，要么您需要通过不完全填满设备来自行保留空间；这因设备而异。无论哪种方式，随着设备填满，垃圾收集器必须更加努力地保持一些块的清洁，因此写入放大因子会增加。

因此，许多设备在填满时会变慢。每个供应商和型号的减速程度各不相同，这取决于设备的架构。一些设备即使在相当满时也设计为高性能，但总的来说，100 GB 文件在 160 GB SSD 上的表现与在 320 GB SSD 上的表现不同。减速是由于在没有空闲块时必须等待擦除完成。写入到空闲块需要几百微秒，但擦除速度要慢得多——通常是几毫秒。

# RAID 性能优化

存储引擎通常将它们的数据和/或索引保存在单个大文件中，这意味着 RAID 通常是存储大量数据的最可行选项。RAID 可以帮助提高冗余性、存储容量、缓存和速度。但与我们一直在研究的其他优化一样，RAID 配置有许多变体，选择适合您需求的配置非常重要。

我们不会在这里涵盖每个 RAID 级别，也不会详细介绍不同 RAID 级别如何存储数据的具体细节。相反，我们专注于 RAID 配置如何满足数据库服务器的需求。以下是最重要的 RAID 级别：

RAID 0

RAID 0 是最便宜且性能最高的 RAID 配置，至少在您简单地衡量成本和性能时是这样（例如，如果包括数据恢复，它开始看起来更昂贵）。由于它不提供冗余性，我们认为 RAID 0 在生产数据库上永远不合适，但如果您真的想要节省成本，它可以是开发环境中的选择，其中完整服务器故障不会变成事故。

再次注意，*RAID 0 不提供任何冗余性*，尽管“冗余”是 RAID 首字母缩略词中的 R。事实上，RAID 0 阵列失败的概率实际上*高于*任何单个磁盘失败的概率，而不是低于！

RAID 1

RAID 1 对于许多场景提供了良好的读取性能，并且它会在磁盘之间复制您的数据，因此具有良好的冗余性。对于读取来说，RAID 1 比 RAID 0 稍快一点。它适用于处理日志和类似工作负载的服务器，因为顺序写入很少需要许多底层磁盘才能表现良好（与随机写入相反，后者可以从并行化中受益）。对于需要冗余但只有两个硬盘的低端服务器来说，这也是一个典型选择。

RAID 0 和 RAID 1 非常简单，通常可以很好地在软件中实现。大多数操作系统都可以让您轻松创建软件 RAID 0 和 RAID 1 卷。

RAID 5

RAID 5 曾经对数据库系统来说是相当可怕的，主要是由于性能影响。随着 SSD 变得普遍，现在它是一个可行的选择。它将数据分布在许多磁盘上，并使用分布式奇偶校验块，因此如果任何一个磁盘故障，数据可以从奇偶校验块重建。如果两个磁盘故障，整个卷将无法恢复。从每单位存储空间的成本来看，这是最经济的冗余配置，因为整个阵列只损失一个磁盘的存储空间。

RAID 5 最大的“坑”是如果一个磁盘故障时阵列的性能如何。这是因为数据必须通过读取所有其他磁盘来重建。这在 HDD 上严重影响了性能，这就是为什么通常不鼓励使用。如果您有很多磁盘，情况会更糟。如果您尝试在重建过程中保持服务器在线，不要指望重建或阵列的性能会很好。其他性能成本包括由于奇偶校验块的限制而导致的有限可扩展性——RAID 5 在超过 10 个磁盘左右时性能不佳——以及缓存问题。良好的 RAID 5 性能严重依赖于 RAID 控制器的缓存，这可能会与数据库服务器的需求发生冲突。正如我们之前提到的，SSD 在 IOPS 和吞吐量方面提供了显着改进的性能，而随机读/写性能不佳的问题也消失了。

RAID 5 的一个缓解因素是它非常受欢迎。因此，RAID 控制器通常针对 RAID 5 进行了高度优化，尽管存在理论限制，但使用缓存良好的智能控制器有时可以在某些工作负载下表现得几乎与 RAID 10 控制器一样好。这实际上可能反映出 RAID 10 控制器的优化程度较低，但无论原因是什么，这就是我们看到的情况。

RAID 6

RAID 5 的最大问题是丢失两个磁盘将是灾难性的。阵列中的磁盘越多，磁盘故障的概率就越高。RAID 6 通过添加第二个奇偶校验磁盘来帮助遏制故障可能性。这使您可以承受两个磁盘故障并仍然重建阵列。不足之处在于计算额外的奇偶校验会使写入速度比 RAID 5 慢。

RAID 10

RAID 10 对于数据存储是一个非常好的选择。它由镜像对组成，这些镜像对是条带化的，因此它既能很好地扩展读取又能扩展写入。与 RAID 5 相比，它重建速度快且容易。它也可以在软件中实现得相当好。

当一个硬盘故障时，性能损失仍然可能很显著，因为该条带可能成为瓶颈。根据工作负载的不同，性能可能会降低高达 50%。要注意的一件事是，某些 RAID 控制器使用“串联镜像”实现 RAID 10。这是次优的，因为缺乏条带化：您最常访问的数据可能只放在一对磁盘上，而不是分布在许多磁盘上，因此性能会很差。

RAID 50

RAID 50 由条带化的 RAID 5 阵列组成，如果你有很多硬盘，它可以在 RAID 5 的经济性和 RAID 10 的性能之间取得很好的折衷。这主要适用于非常大的数据集，比如数据仓库或极大型的 OLTP 系统。

表 4-1 总结了各种 RAID 配置。

表 4-1\. RAID 级别比较

| 级别 | 摘要 | 冗余性 | 所需硬盘 | 更快读取 | 更快写入 |
| --- | --- | --- | --- | --- | --- |
| RAID 0 | 便宜，快速，危险 | 否 | N | 是 | 是 |
| RAID 1 | 快速读取，简单，安全 | 是 | 2（通常） | 是 | 否 |
| RAID 5 | 便宜，与 SSD 一起快速 | 是 | N + 1 | 是 | 取决于 |
| RAID 6 | 类似于 RAID 5 但更具弹性 | 是 | N + 2 | 是 | 取决于 |
| RAID 10 | 昂贵，快速，安全 | 是 | 2N | 是 | 是 |
| RAID 50 | 用于非常大型数据存储 | 是 | 2(N + 1) | 是 | 是 |

## RAID 故障、恢复和监控

RAID 配置（除了 RAID 0）提供冗余性。这很重要，但很容易低估同时硬盘故障的可能性。你不应该认为 RAID 是数据安全的强有力保证。

RAID 并不能消除——甚至不能减少——备份的需求。当出现问题时，恢复时间将取决于你的控制器、RAID 级别、阵列大小、硬盘速度以及在重建阵列时是否需要保持服务器在线。

硬盘同时发生故障的可能性是存在的。例如，电力波动或过热很容易导致两个或更多硬盘损坏。然而，更常见的是两个硬盘故障发生在较短的时间内。许多这样的问题可能不会被注意到。一个常见的原因是很少访问的物理介质上的损坏，这可能会在几个月内不被发现，直到你尝试读取数据或另一个硬盘故障并且 RAID 控制器尝试使用损坏的数据重建阵列。硬盘越大，这种情况发生的可能性就越大。

这就是为什么监视你的 RAID 阵列很重要。大多数控制器提供一些软件来报告阵列的状态，你需要跟踪这些信息，否则你可能完全不知道硬盘故障。你可能会错过恢复数据的机会，只有当第二个硬盘故障时才发现问题，那时已经太迟了。你应该配置一个监控系统，在硬盘或卷更改为降级或失败状态时通知你。

你可以通过定期主动检查阵列的一致性来减轻潜在损坏的风险。一些控制器的背景巡逻读取功能可以在所有硬盘在线时检查损坏的介质并修复它，也可以帮助避免这些问题。与恢复一样，非常大的阵列可能检查速度较慢，因此在创建大型阵列时一定要做好计划。

你还可以添加一个热备用硬盘，它是未使用的，并配置为控制器自动用于恢复的待机硬盘。如果你依赖每台服务器，这是一个好主意。对于只有少量硬盘的服务器来说，这是昂贵的，因为拥有一个空闲硬盘的成本相对较高，但如果你有很多硬盘，不配置热备用几乎是愚蠢的。请记住，随着硬盘数量的增加，硬盘故障的概率会迅速增加。

除了监视驱动器故障，你还应该监视 RAID 控制器的电池备份单元和写缓存策略。如果电池故障，默认情况下大多数控制器会通过将缓存策略更改为写穿透而不是写回来禁用写缓存。这可能会导致性能严重下降。许多控制器还会定期通过学习过程循环电池，在此期间缓存也被禁用。你的 RAID 控制器管理实用程序应该让你查看和配置学习周期何时安排，以免让你措手不及。新一代的 RAID 控制器通过使用使用 NVRAM 存储未提交写入的闪存支持缓存来避免这种情况，而不是使用电池支持的缓存。这避免了学习周期的整个痛苦。

你可能还想使用写穿透的缓存策略对系统进行基准测试，这样你就会知道可以期待什么。首选的方法是在低流量时段安排电池学习周期，通常在晚上或周末。如果在任何时候使用写穿透时性能严重下降，你也可以在学习周期开始之前切换到另一台服务器。作为最后的手段，你可以通过更改`innodb_flush_log_at_trx_commit`和`sync_binlog`变量来重新配置服务器，以降低耐久性设置。这将减少写穿透期间的磁盘利用率，并可能提供可接受的性能；然而，这真的应该作为最后的手段。降低耐久性会对在数据库崩溃期间可能丢失的数据量以及恢复数据的能力产生重大影响。

## RAID 配置和缓存

通常可以通过在机器的引导序列期间输入其设置实用程序或通过从命令提示符运行来配置 RAID 控制器本身。尽管大多数控制器提供了许多选项，但我们关注的两个是条带阵列的*块大小*和*控制器缓存*（也称为*RAID 缓存*；我们可以互换使用这些术语）。

### RAID 条带块大小

最佳条带块大小是与工作负载和硬件特定的。理论上，对于随机 I/O，拥有较大的块大小是有好处的，因为这意味着更多的读取可以从单个驱动器中满足。

要了解为什么会这样，请考虑你的工作负载的典型随机 I/O 操作的大小。如果块大小至少与该大小相同，并且数据不跨越块之间的边界，只需要一个驱动器参与读取。但是，如果块大小小于要读取的数据量，就无法避免多个驱动器参与读取。

理论就到此为止。实际上，许多 RAID 控制器不适用于大块。例如，控制器可能将块大小用作其缓存中的缓存单元，这可能是浪费的。控制器还可能匹配块大小、缓存大小和读取单元大小（单次操作中读取的数据量）。如果读取单元太大，其缓存可能不太有效，并且可能会读取比实际需要的数据量更多，即使是对于微小的请求。

也很难知道任何给定数据是否会跨越多个驱动器。即使块大小为 16 KB，与 InnoDB 的页面大小相匹配，你也不能确定所有读取是否都对齐在 16 KB 边界上。文件系统可能会使文件碎片化，并且通常会将碎片对齐在文件系统块大小上，通常为 4 KB。一些文件系统可能更智能，但你不应该指望它。

### RAID 缓存

RAID 缓存是物理安装在硬件 RAID 控制器上的（相对较小的）内存量。它可用于在数据在磁盘和主机系统之间传输时作为缓冲区。以下是 RAID 卡可能使用缓存的一些原因：

缓存读取

在控制器从磁盘中读取一些数据并将其发送到主机系统后，它可以存储数据；这将使它能够在不再需要再次访问磁盘的情况下满足对相同数据的未来请求。

这通常是 RAID 缓存的非常糟糕的用法。为什么？因为操作系统和数据库服务器有自己更大的缓存。如果其中一个缓存中有缓存命中，RAID 缓存中的数据将不会被使用。反之，如果高级别缓存中有缓存未命中，那么 RAID 缓存中有缓存命中的机会几乎为零。由于 RAID 缓存要小得多，它几乎肯定已经被刷新并填充了其他数据。无论从哪个角度看，将读取缓存到 RAID 缓存中都是一种浪费内存的行为。

缓存预读数据

如果 RAID 控制器注意到对数据的顺序请求，它可能会决定进行预读操作——即预取它预测很快会需要的数据。但在数据被请求之前，它必须有地方存放数据。它可以使用 RAID 缓存来实现这一点。这种操作的性能影响可能会有很大的变化，你应该检查以确保它确实有帮助。如果数据库服务器正在执行自己的智能预读操作（如 InnoDB 所做的），预读操作可能不会有帮助，并且可能会干扰同步写入的重要缓冲。

缓存写入

RAID 控制器可以在其缓存中缓冲写入并安排它们在稍后执行。这样做的优点是双重的：首先，它可以比实际在物理磁盘上执行写入更快地向主机系统返回“成功”，其次，它可以累积写入并更有效地执行它们。

内部操作

一些 RAID 操作非常复杂——特别是 RAID 5 写入，它们必须计算可以用于在发生故障时重建数据的奇偶校验位。控制器需要为这种类型的内部操作使用一些内存。这是 RAID 5 在某些控制器上性能不佳的原因之一：它需要将大量数据读入缓存以获得良好的性能。一些控制器无法平衡缓存写入和 RAID 5 奇偶校验操作的缓存。

一般来说，RAID 控制器的内存是一种稀缺资源，你应该明智地使用它。将其用于读取通常是浪费，但将其用于写入是提高 I/O 性能的重要方式。许多控制器允许你选择如何分配内存。例如，你可以选择将多少内存用于缓存写入，将多少用于读取。对于 RAID 0、RAID 1 和 RAID 10，你可能应该将控制器内存的 100% 用于缓存写入。对于 RAID 5，你应该保留一些控制器内存用于其内部操作。这通常是一个好建议，但并不总是适用——不同的 RAID 卡需要不同的配置。

当你使用 RAID 缓存进行写入缓存时，许多控制器允许你配置延迟写入的时间（一秒、五秒等）。延迟时间更长意味着更多的写入可以被分组并优化地刷新到磁盘上。缺点是你的写入将更“突发”。这并不是一件坏事，除非你的应用程序恰好在控制器缓存填满时发出一堆写入请求，即将刷新到磁盘上。如果没有足够的空间来处理应用程序的写入请求，它将不得不等待。保持延迟时间较短意味着你将有更多的写入操作，它们将更不高效，但它可以平滑地处理尖峰，并帮助保持更多的缓存空闲以处理应用程序的突发请求。（我们在这里进行了简化——控制器通常具有复杂的、供应商特定的平衡算法，所以我们只是试图涵盖基本原则。）

写缓存对于同步写入非常有帮助，例如在事务日志上发出`fsync()`调用和启用`sync_binlog`创建二进制日志，但除非您的控制器有电池备份单元（BBU）或其他非易失性存储，否则不应启用它。在没有 BBU 的情况下缓存写入可能会导致数据库甚至事务性文件系统在断电时损坏。然而，如果您有 BBU，启用写缓存可以提高性能，例如对于执行大量日志刷新操作的工作负载，例如在事务提交时刷新事务日志。

最后一个考虑因素是许多硬盘都有自己的写缓存，可以通过欺骗控制器向物理介质写入数据来“伪造”`fsync()`操作。直接连接的硬盘（而不是连接到 RAID 控制器）有时可以让它们的缓存由操作系统管理，但这也并不总是有效的。这些缓存通常会在`fsync()`时被刷新，并且在同步 I/O 时被绕过，但是硬盘可能会撒谎。你应该确保这些缓存在`fsync()`时被刷新，或者禁用它们，因为它们没有备用电源。操作系统或 RAID 固件未正确管理的硬盘已经导致了许多数据丢失的情况。

出于这个原因和其他原因，当您安装新硬件时，进行真正的崩溃测试（从墙上拔下电源插头）总是一个好主意。这通常是发现微妙的配置错误或狡猾的硬盘行为的唯一方法。可以在[在线](https://oreil.ly/2Lume)找到一个方便的脚本。

要测试您是否真的可以依赖 RAID 控制器的 BBU，请确保将电源线拔掉一段现实时间。一些设备在没有电源的情况下的持续时间可能不如预期的长。在这里，一个糟糕的环节可能使您整个存储组件链变得无用。

# 网络配置

就像延迟和吞吐量对硬盘是限制因素一样，延迟和带宽对网络连接也是限制因素。对大多数应用程序来说，最大的问题是延迟；典型应用程序进行大量小型网络传输，每次传输的轻微延迟会累积起来。

网络运行不正确也是一个主要的性能瓶颈。数据包丢失是一个常见问题。即使 1%的丢包足以导致显著的性能下降，因为协议栈中的各个层将尝试使用策略来解决问题，例如等待一段时间然后重新发送数据包，这会增加额外的时间。另一个常见问题是破损或缓慢的 DNS 解析。

DNS 足以成为一个致命弱点，因此在生产服务器上启用`skip_name_resolve`是一个好主意。破损或缓慢的 DNS 解析对许多应用程序都是一个问题，但对于 MySQL 来说尤为严重。当 MySQL 收到连接请求时，它会进行正向和反向 DNS 查找。有很多原因可能导致这种情况出错。一旦出错，将导致连接被拒绝，连接到服务器的过程变慢，并且通常会造成混乱，甚至包括拒绝服务攻击。如果启用`skip_name_resolve`选项，MySQL 将不执行任何 DNS 查找。但是，这也意味着您的用户帐户在`host`列中必须只有 IP 地址、“localhost”或 IP 地址通配符。任何在`host`列中具有主机名的用户帐户将无法登录。

调整设置以有效处理大量连接和小查询通常更为重要。其中一个更常见的调整是更改本地端口范围。Linux 系统有一系列可用的本地端口。当连接返回给调用者时，它使用本地端口。如果有许多同时连接，您可能会用完本地端口。

这是一个配置为默认值的系统：

```sql
$ cat /proc/sys/net/ipv4/ip_local_port_range
32768 61000
```

有时你可能需要将这些值更改为更大的范围。例如：

```sql
$ echo 1024 65535 > /proc/sys/net/ipv4/ip_local_port_range
```

TCP 协议允许系统排队接收连接，就像一个桶。如果桶装满了，客户端就无法连接。你可以通过以下方式允许更多连接排队：

```sql
$ echo 4096 > /proc/sys/net/ipv4/tcp_max_syn_backlog
```

对于仅在本地使用的数据库服务器，你可以缩短在关闭套接字后的超时时间，以防对等方断开连接但不关闭连接的情况。在大多数系统上，默认值是一分钟，这相当长：

```sql
$ echo <value> > /proc/sys/net/ipv4/tcp_fin_timeout
```

大多数情况下，这些设置可以保持默认值不变。通常只有在发生异常情况时才需要更改它们，比如网络性能极差或连接数量非常大。在互联网上搜索“TCP 变量”会找到很多关于这些变量和更多变量的好文章。

# 选择文件系统

你的文件系统选择在很大程度上取决于你的操作系统。在许多系统中，比如 Windows，你实际上只有一两个选择，而且只有一个（NTFS）是真正可行的。另一方面，GNU/Linux 支持许多文件系统。

许多人想知道哪种文件系统在 GNU/Linux 上为 MySQL 提供最佳性能，甚至更具体地说，哪种选择对 InnoDB 最好。实际的基准测试显示，它们在大多数方面都非常接近，但是寻求文件系统性能实际上是一个干扰。文件系统的性能非常依赖于工作负载，并且没有一个文件系统是万能的。大多数情况下，一个给定的文件系统不会比其他文件系统表现明显更好或更差。唯一的例外是如果你遇到某些文件系统限制，比如它如何处理并发性、处理许多文件、碎片化等等。

总的来说，最好使用一个日志文件系统，比如 ext4、XFS 或 ZFS。如果不这样做，在崩溃后进行文件系统检查可能需要很长时间。

如果你使用 ext3 或其后继者 ext4，你有三个选项来记录数据的方式，你可以将它们放在*/etc/fstab*挂载选项中：

`data=writeback`

这个选项意味着只有元数据写入被记录。元数据写入不与数据写入同步。这是最快的配置，通常与 InnoDB 一起使用是安全的，因为它有自己的事务日志。唯一的例外是，在 MySQL 的 8.0 版本之前，如果在恰当的时机发生崩溃，可能会导致*.frm*文件损坏。

这里有一个示例，说��这种配置可能会导致问题。假设一个程序决定扩展一个文件使其更大。元数据（文件的大小）将在实际写入数据到（现在更大的）文件之前被记录和写入。结果是文件的尾部——新扩展区域——包含垃圾。

`data=ordered`

这个选项也只记录元数据，但通过在写入数据之前写入数据来提供一些一致性，以保持一致性。它只比`writeback`选项稍慢一点，但在发生崩溃时更安全。在这种配置下，如果我们再次假设一个程序想要扩展一个文件，文件的元数据在数据写入新扩展区域之前不会反映文件的新大小。

`data=journal`

此选项提供原子日志行为，将数据写入日志后再写入最终位置。通常情况下这是不必要的，并且比其他两个选项的开销要大得多。然而，在某些情况下，它可以提高性能，因为日志记录使文件系统能够延迟将数据写入最终位置。

无论文件系统如何，最好禁用一些特定选项，因为它们不提供任何好处，而且可能会增加相当多的开销。最著名的是记录访问时间，即使您只是读取文件或目录，也需要写入。要禁用此选项，请将`noatime,nodiratime`挂载选项添加到您的*/etc/fstab*；这有时可以提高性能 5%–10%，具体取决于工作负载和文件系统（尽管在其他情况下可能没有太大差异）。以下是我们提到的 ext3 选项的示例*/etc/fstab*行：

```sql
/dev/sda2 /usr/lib/mysql ext3 noatime,nodiratime,data=writeback 0 1
```

您还可以调整文件系统的预读行为，因为这可能是多余的。例如，InnoDB 会进行自己的预读预测。在 Solaris 的 UFS 上禁用或限制预读对性能特别有益。使用`innodb_​flush_​method=​O_DIRECT`会自动禁用预读。

一些文件系统不支持您可能需要的功能。例如，如果您正在使用 InnoDB 的`O_DIRECT`刷新方法，对于直接 I/O 的支持可能很重要。此外，一些文件系统比其他文件系统更好地处理大量底层驱动器；例如，XFS 在这方面通常比 ext3 好得多。最后，如果您计划使用逻辑卷管理器（LVM）快照来初始化副本或进行备份，您应该验证您选择的文件系统和 LVM 版本是否能很好地配合。

表 4-2 总结了一些常见文件系统的特性。

表 4-2\. 常见文件系统特性

| 文件系统 | 操作系统 | 日志记录 | 大目录 |
| --- | --- | --- | --- |
| ext3 | GNU/Linux | 可选 | 可选/部分 |
| ext4 | GNU/Linux | 是 | 是 |
| 日志文件系统 (JFS) | GNU/Linux | 是 | 否 |
| NTFS | Windows | 是 | 是 |
| ReiserFS | GNU/Linux | 是 | 是 |
| UFS (Solaris) | Solaris | 是 | 可调 |
| UFS (FreeBSD) | FreeBSD | 否 | 可选/部分 |
| UFS2 | FreeBSD | 否 | 可选/部分 |
| XFS | GNU/Linux | 是 | 是 |
| ZFS | GNU/Linux, Solaris, FreeBSD | 是 | 是 |

我们通常建议使用 XFS 文件系统。ext3 文件系统有太多严重的限制，比如每个 inode 只有一个互斥锁，以及不好的行为，比如在`fsync()`上刷新整个文件系统中的所有脏块，而不仅仅是一个文件的脏块。ext4 文件系统是一个可以接受的选择，尽管在特定内核版本中可能存在性能瓶颈，您在承诺之前应该调查一下。

在考虑为数据库选择任何文件系统时，考虑它已经可用多久，它有多成熟，以及在生产环境中它已经被证明。文件系统位是数据库中最低级别的数据完整性。

## 选择磁盘队列调度程序

在 GNU/Linux 上，队列调度程序确定请求发送到底层设备的顺序。默认值是完全公平队列，或`cfq`。对于笔记本电脑和台式机的日常使用，它可以防止 I/O 饥饿，但对于服务器来说很糟糕。因为它会不必要地使一些请求在队列中停滞。

您可以使用以下命令查看可用的调度程序以及哪个是活动的：

```sql
$ cat /sys/block/sda/queue/scheduler
noop deadline [cfq]
```

你应该用感兴趣的磁盘设备名称替换*`sda`*。在我们的示例中，方括号表示此设备正在使用哪种调度程序。另外两个选择适用于服务器级硬件，在大多数情况下它们的效果差不多。`noop`调度程序适用于在后台进行自己调度的设备，例如硬件 RAID 控制器和存储区域网络（SAN），而`deadline`适用于直接连接的 RAID 控制器和磁盘。我们的基准测试显示这两者之间几乎没有区别。最重要的是使用除了`cfq`之外的任何调度程序，因为它可能导致严重的性能问题。

## 内存和交换

MySQL 在分配给它大量内存时表现最佳。正如我们在第一章中学到的，InnoDB 使用内存作为缓存以避免磁盘访问。这意味着内存系统的性能直接影响查询服务的速度。即使在今天，确保更快的内存访问的最佳方法之一是用外部内存分配器（`glibc`）替换内置内存分配器，如`tcmalloc`或`jemalloc`。许多基准测试²表明，与`glibc`相比，这两者都提供了改进的性能和减少的内存碎片化。

当操作系统将一些虚拟内存写入磁盘因为没有足够的物理内存来保存时，就会发生交换。对于运行在操作系统上的进程，交换是透明的。只有操作系统知道特定虚拟内存地址是在物理内存中还是在磁盘上。

在使用 SSD 时，性能损失不像以前使用 HDD 那样严重。你仍然应该积极避免交换，即使只是为了避免不必要的写入可能缩短磁盘的整体寿命。你也可以考虑采用不使用交换的方法，这样可以避免潜在的问题，但会使你处于内存耗尽可能导致进程终止的情况。

在 GNU/Linux 上，你可以使用*vmstat*来监视交换（我们在下一节中展示了一些示例）。你需要查看交换 I/O 活动，报告在`si`和`so`列中，而不是交换使用情况，报告在`swpd`列中。`swpd`列可能显示已加载但未使用的进程，这并不是真正的问题。我们希望`si`和`so`列的值为`0`，它们肯定应该小于每秒 10 个块。

在极端情况下，过多的内存分配可能导致操作系统的交换空间耗尽。如果发生这种情况，由于虚拟内存的缺乏，MySQL 可能会崩溃。但即使不会耗尽交换空间，非常活跃的交换也可能导致整个操作系统无响应，甚至无法登录和终止 MySQL 进程。有时候当操作系统耗尽交换空间时，Linux 内核甚至会完全挂起。我们建议你完全不使用交换空间来运行数据库。磁盘仍然比 RAM 慢一个数量级，这样可以避免这里提到的所有问题。

在极端虚拟内存压力下经常发生的另一件事是内存不足（OOM）杀手进程会启动并终止某些进程。这经常是 MySQL，但也可能是另一个进程，比如 SSH，这可能导致你的系统无法从网络访问。你可以通过设置 SSH 进程的`oom_adj`或`oom_score_adj`值来防止这种情况发生。在使用专用数据库服务器时，我们强烈建议你识别任何关键进程，如 MySQL 和 SSH，并主动调整 OOM 杀手分数，以防止它们被首先选择终止。

您可以通过正确配置 MySQL 缓冲区来解决大多数交换问题，但有时操作系统的虚拟内存系统决定无论如何交换 MySQL，有时与 Linux 中的非统一内存访问（NUMA）的工作方式有关³。这通常发生在操作系统看到 MySQL 的大量 I/O 时，因此它试图增加文件缓存以容纳更多数��。如果内存不足，必须交换出某些内容，而这些内容可能是 MySQL 本身。一些较旧的 Linux 内核版本还具有不当的优先级，会在不应该交换时交换内容，但在较新的内核中已经有所缓解。

操作系统通常允许对虚拟内存和 I/O 进行一些控制。我们在 GNU/Linux 上提到了一些控制它们的方法。最基本的是将*/proc/sys/vm/swappiness*的值更改为低值，例如`0`或`1`。这告诉内核除非对虚拟内存的需求极端，否则不要交换。例如，这是如何检查当前值的方法：

```sql
$ cat /proc/sys/vm/swappiness
60
```

显示的值为 60，是默认的 swappiness 设置（范围从 0 到 100）。这对服务器来说是非常糟糕的默认值。这只适用于笔记本电脑。服务器应设置为`0`：

```sql
$ echo 0 > /proc/sys/vm/swappiness
```

另一个选项是更改存储引擎读取和写入数据的方式。例如，使用`innodb_flush_method=O_DIRECT`可以减轻 I/O 压力。直接 I/O 不会被缓存，因此操作系统不会将其视为增加文件缓存大小的原因。此参数仅适用于 InnoDB。

另一个选项是使用 MySQL 的`memlock`配置选项，将 MySQL 锁定在内存中。这将避免交换，但可能会很危险：如果没有足够的可锁定内存剩余，当 MySQL 尝试分配更多内存时，MySQL 可能会崩溃。如果锁定了太多内存，而操作系统没有足够的内存剩余，也可能会引起问题。

许多技巧特定于内核版本，因此要小心，特别是在升级时。在某些工作负载中，很难使操作系统表现得明智，您的唯一选择可能是将缓冲区大小降低到次优值。

## 操作系统状态

您的操作系统提供了工具，帮助您了解操作系统和硬件正在做什么。在本节中，我们将向您展示如何使用两个广泛可用的工具*iostat*和*vmstat*的示例。如果您的系统没有提供这两个工具中的任何一个，那么很可能会提供类似的工具。因此，我们的目标不是让您成为*iostat*或*vmstat*的专家，而只是向您展示在尝试使用这些工具诊断问题时要寻找什么。

除了这些工具，您的操作系统可能提供其他工具，如*mpstat*或*sar*。如果您对系统的其他部分感兴趣，例如网络，您可能想使用*ifconfig*（显示发生了多少网络错误等）或*netstat*等工具。

默认情况下，*vmstat*和*iostat*只生成一个报告，显示自服务器启动以来各种计数器的平均值，这并不是很有用。但是，您可以为这两个工具提供一个间隔参数。这使它们生成增量报告，显示服务器当前正在执行的操作，这更加相关。（第一行显示自系统启动以来的统计信息；您可以忽略此行。）

### 如何阅读 vmstat 输出

让我们先看一个*vmstat*的示例。要使其每五秒打印一个新报告，以兆字节为单位报告大小，请使用以下命令：

```sql
$ vmstat -SM 5
procs -------memory------- -swap- -----io---- ---system---- ------cpu-----
 r  b swpd free buff cache  si so    bi    bo     in     cs us sy id wa st
11  0    0 2410    4 57223   0  0  9902 35594 122585 150834 10  3 85  1  0
10  2    0 2361    4 57273   0  0 23998 35391 124187 149530 11  3 84  2  0
```

您可以使用 Ctrl-C 停止*vmstat*。您看到的输出取决于您的操作系统，因此您可能需要阅读手册页以弄清楚。

正如前面所述，尽管我们要求增量输出，但第一行的值显示了自服务器启动以来的平均值。第二行显示了当前的情况，随后的行将显示每隔五秒发生的情况。这些列按以下其中一个标题分组：

procs

`r` 列显示有多少进程正在等待 CPU 时间。`b` 列显示有多少进程处于不可中断的睡眠状态，这通常意味着它们正在等待 I/O（磁盘、网络、用户输入等）。

memory

`swpd` 列显示了被交换到磁盘（分页）的块数。剩下的三列显示了多少块是 `free`（未使用的）、多少块用于缓冲区（buff），以及多少块用于操作系统的 `cache`。

swap

这些列显示了交换活动：操作系统每秒交换进（从磁盘）和交换出（到磁盘）的块数。它们比 `swpd` 列更重要。我们希望大部分时间看到 `si` 和 `so` 为 `0`，绝对不希望看到超过 10 块每秒。突发也是不好的。

io

这些列显示每秒从块设备读入的块数（`bi`）和写出的块数（`bo`）。这通常反映了磁盘 I/O。

system

这些列显示每秒中断数（`in`）和每秒上下文切换数（`cs`）。

cpu

这些列显示了总 CPU 时间的百分比，用于运行用户（非内核）代码、运行系统（内核）代码、空闲以及等待 I/O。可能的第五列（`st`）显示了如果您使用虚拟化，则从虚拟机“窃取”的百分比。这指的是在虚拟机上有东西可以运行的时间，但是 hypervisor 选择运行其他东西的时间。如果虚拟机不想运行任何东西，而 hypervisor 运行其他东西，那就不算是被窃取的时间。

*vmstat* 输出是系统相关的，因此如果您的输出与我们展示的示例不同，您应该阅读您系统的 `vmstat(8)` 手册页。

### 如何阅读 iostat 输出

现在让我们转向 *iostat*。默认情况下，它显示了与 *vmstat* 相同的一些 CPU 使用信息。不过，我们通常只对 I/O 统计感兴趣，因此我们使用以下命令仅显示扩展设备统计信息：

```sql
$ iostat -dxk 5
Device: rrqm/s wrqm/s r/s w/s rkB/s wkB/s 
sda 0.00 0.00 1060.40 3915.00 8483.20 42395.20 

avgrq-sz avgqu-sz await r_await w_await svctm %util
 20.45 3.68 0.74 0.57 0.78 0.20 98.22
```

与 *vmstat* 一样，第一个报告显示了自服务器启动以来的平均值（我们通常省略以节省空间），随后的报告显示了增量平均值。每个设备一行。

有各种选项可以显示或隐藏列。官方文档有点混乱，我们不得不深入源代码中找出到底显示了什么。以下是每列显示的内容：

`rrqm/s` 和 `wrqm/s`

每秒排队的合并读写请求数。*合并* 意味着操作系统从队列中获取多个逻辑请求，并将它们组合成一个实际设备的单个请求。

`r/s` 和 `w/s`

每秒发送到设备的读取和写入请求数。

`rkB/s` 和 `wkB/s`

每秒读取和写入的千字节数。

`avgrq-sz`

请求大小（以扇区为单位）。

`avgqu-sz`

在设备队列中等待的请求数。

`await`

在磁盘队列中花费的毫秒数。

`r_await` 和 `w_await`

发送到设备的读取请求的平均时间（以毫秒为单位），分别为读取和写入。这包括请求在队列中花费的时间以及为其提供服务的时间。

`svctm`

服务请求所花费的毫秒数，不包括队列时间。

`%util`⁴

至少有一个请求处于活动状态的时间百分比。这个名字非常令人困惑。如果您熟悉排队理论中*利用率*的标准定义，那么这就是设备的利用率。具有多个硬盘驱动器（如 RAID 控制器）的设备应该能够支持比 1 更高的并发性，但是`%util`永远不会超过 100%，除非在计算中存在四舍五入误差。因此，与文档所说的相反，它并不是设备饱和的良好指标，除非您正在查看单个物理硬盘的特殊情况。

您可以使用输出推断有关机器 I/O 子系统的一些事实。一个重要的指标是同时服务的请求数。由于读取和写入是每秒进行的，而服务时间的单位是千分之一秒，您可以使用 Little's law 推导出设备正在服务的并发请求数的以下公式：

```sql
concurrency = (r/s + w/s) * (svctm/1000)
```

将前面的样本数字插入并发公式中得到约 0.995 的并发性。这意味着在采样间隔期间，设备平均服务的请求少于一个。

## 其他有用的工具

我们展示了*vmstat*和*iostat*，因为它们是广泛可用的工具，而*vmstat*通常默认安装在许多类 Unix 操作系统上。然而，这些工具各有其局限性，比如单位混乱、采样间隔与操作系统更新统计数据的时间不对应，以及无法一次看到所有指标。如果这些工具不符合您的需求，您可能会对[*dstat*](http://dag.wieers.com/home-made/dstat)或[*collectl*](https://oreil.ly/DSvmM)感兴趣。

我们也喜欢使用*mpstat*来监视 CPU 统计信息；它提供了关于 CPU 如何单独运行的更好的想法，而不是将它们全部分组在一起。在诊断问题时，这有时非常重要。当您检查磁盘 I/O 使用情况时，您可能会发现*blktrace*也很有帮助。

Percona 编写了自己的*iostat*替代工具称为*pt-diskstats*。它是 Percona Toolkit 的一部分。它解决了一些关于*iostat*的抱怨，比如它如何将读取和写入汇总以及对并发性的可见性不足。它还是交互式的，通过按键驱动，因此您可以放大和缩小，更改聚合，过滤设备，显示和隐藏列。这是一个很好的方式来切分和分析磁盘统计数据的样本，即使您没有安装该工具，也可以通过简单的 shell 脚本收集磁盘活动的样本并通过电子邮件或保存以供以后分析。

最后，Linux 分析器*perf*是检查操作系统级别发生的情况的宝贵工具。您可以使用*perf*检查有关操作系统的一般信息，比如为什么内核使用 CPU 这么多。您还可以检查特定的进程 ID，从而查看 MySQL 如何与操作系统交互。检查系统性能是一个非常深入的过程，因此我们推荐 Brendan Gregg 的《Systems Performance, Second Edition》（Pearson）作为优秀的后续阅读。

# 总结

选择和配置 MySQL 的硬件，并为硬件配置 MySQL，并不是一门神秘的艺术。一般来说，您需要与大多数其他目的相同的技能和知识。然而，有一些 MySQL 特定的事项您应该知道。

我们通常建议大多数人在性能和成本之间找到一个良好的平衡。 首先，我们喜欢使用商品服务器，有很多原因。 例如，如果您的服务器出现问题，您需要将其停机以诊断问题，或者如果您只是想尝试用另一台服务器替换它作为诊断的一种形式，那么使用价值 5000 美元的服务器比使用价值 50000 美元或更高的服务器要容易得多。 MySQL 通常也更适合于商品硬件，无论是从软件本身还是从典型的工作负载来看。

MySQL 需要的四个基本资源是 CPU、内存、磁盘和网络资源。 网络很少会成为严重瓶颈，但 CPU、内存和磁盘确实会。 速度和数量的平衡取决于工作负载，您应该根据预算的允许程度努力实现快速和多样的平衡。 你期望的并发越多，你就应该更多地依赖更多的 CPU 来适应你的工作负载。

CPU、内存和磁盘之间的关系错综复杂，一个领域的问题通常会在其他地方显现出来。 在向问题投入资源之前，问问自己是否应该将资源投入到另一个问题上。 如果你受到 I/O 限制，你需要更多的 I/O 容量，还是只需要更多的内存？ 答案取决于工作集大小，即在给定时间内最常需要的数据集。

固态设备非常适合提高服务器整体性能，现在通常应该成为数据库的标准，特别是 OLTP 工作负载。 继续使用 HDD 的唯一理由是在极度预算受限的系统中或者需要大量磁盘空间的情况下，例如在数据仓库情况下需要 PB 级别的磁盘空间。

在操作系统方面，有一些关键的事项需要正确处理，主要涉及存储、网络和虚拟内存管理。 如果您使用 GNU/Linux，正如大多数 MySQL 用户所做的那样，我们建议使用 XFS 文件系统，并将 swappiness 和磁盘队列调度器设置为适合服务器的值。 有一些可能需要更改的网络参数，您可能希望调整其他一些参数（例如禁用 SELinux），但这些更改是个人偏好的问题。

¹ 流行的俳句：这不是 DNS。 不可能是 DNS。 就是 DNS。

² 参见博客文章[“内存分配器对 MySQL 性能的影响”](https://oreil.ly/AAJHX)和[“MySQL（或 Percona）内存使用测试”](https://oreil.ly/slp7v)进行比较。

³ 更多信息请参见[此博客文章](https://oreil.ly/VGW65)。

⁴ 软件 RAID，如 MD/RAID，可能不会显示 RAID 阵列本身的利用率。