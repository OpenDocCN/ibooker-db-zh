# 第八章：事务

事务是数据库中的逻辑处理组，每个组或事务可以包含一个或多个操作，如跨多个文档的读取和/或写入。MongoDB 支持跨多个操作、集合、数据库、文档和分片的 ACID 合规事务。在本章中，我们介绍事务，定义数据库中 ACID 的含义，突出显示如何在应用程序中使用它们，并提供调整 MongoDB 中事务的提示。我们将涵盖：

+   事务的定义

+   如何使用事务

+   调整应用程序的事务限制

# 事务简介

正如上文所述，事务是数据库中的一个逻辑处理单元，包括一个或多个数据库操作，可以是读取或写入操作。在处理的这个逻辑单元中，您的应用程序可能需要对多个文档（一个或多个集合中的多个文档）进行读取和写入。事务的一个重要方面是它永远不会部分完成——它要么成功，要么失败。

###### 注意

要使用事务，您的 MongoDB 部署必须在 MongoDB 版本 4.2 或更高版本，并且您的 MongoDB 驱动程序必须更新到 MongoDB 4.2 或更高版本。MongoDB 提供了一个[驱动程序兼容性参考页面](https://oreil.ly/Oe9NE)，您可以使用它来确保您的 MongoDB 驱动程序版本兼容。

## ACID 的定义

ACID 是事务必须满足的一组接受的属性，以成为“真正”的事务。ACID 是原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）和持久性（Durability）的首字母缩写。ACID 事务保证数据的有效性及数据库状态的一致性，即使发生断电或其他错误。

原子性确保事务内部的所有操作要么全部应用，要么全部不应用。事务永远不会部分应用；它要么被提交，要么中止。

一致性确保如果事务成功，数据库将从一个一致状态移动到下一个一致状态。

隔离性是允许多个事务同时在数据库中运行的属性。它保证一个事务不会查看任何其他事务的部分结果，这意味着多个并行事务将具有与按顺序运行每个事务相同的结果。

持久性确保当事务提交时，所有数据都将持久存在，即使系统故障也是如此。

数据库在确保满足所有这些属性并且仅处理成功事务时，被称为符合 ACID 标准。在事务完成之前发生故障的情况下，ACID 合规性确保数据不会被更改。

MongoDB 是一个分布式数据库，支持跨副本集和/或分片的 ACID 兼容事务。网络层增加了额外的复杂性。MongoDB 工程团队提供了[几个黑板讲解视频](https://www.mongodb.com/transactions)，描述了他们如何实现支持 ACID 事务所需的必要功能。

# 如何使用事务

MongoDB 提供两个 API 来使用事务。第一个 API 与关系型数据库类似（例如，`start_transaction`和`commit_transaction`），称为核心 API；第二个称为回调 API，是使用事务的推荐方法。

核心 API 不提供大多数错误的重试逻辑，并要求开发者编写操作逻辑、事务提交函数以及任何必要的重试和错误处理逻辑。

回调 API 提供一个函数，它比核心 API 包含更多功能，包括启动与指定逻辑会话相关联的事务、执行作为回调函数提供的函数，然后提交事务（或在错误时中止）。此函数还包括处理提交错误的重试逻辑。回调 API 在 MongoDB 4.2 中添加，旨在简化使用事务的应用程序开发，同时使添加应用程序重试逻辑以处理任何事务错误变得更加容易。

在这两个 API 中，开发者负责启动将用于事务的逻辑会话。两个 API 要求事务中的操作与特定的逻辑会话相关联（即，将会话传递给每个操作）。MongoDB 中的逻辑会话跟踪操作的时间和顺序，这些操作在整个 MongoDB 部署的上下文中。逻辑会话或服务器会话是客户端会话使用的基础框架的一部分，用于支持可重试写入和因果一致性。这些功能作为支持事务所需的基础的一部分，已经在 MongoDB 版本 3.6 中添加。在 MongoDB 中，具有因果关系的读写操作序列被定义为因果一致的客户端会话。客户端会话由应用程序启动并用于与服务器会话交互。

2019 年，MongoDB 的六名高级工程师在 SIGMOD 2019 大会上发表了题为[“在 MongoDB 中实现集群范围逻辑时钟和因果一致性”的论文](https://oreil.ly/IFLvm)。^(1) 该论文深入解释了 MongoDB 中逻辑会话和因果一致性背后的机制。论文记录了一个跨多团队、多年的工程项目的努力。这项工作涉及改变存储层的方面，添加新的复制一致性协议，修改分片架构，重构分片集群元数据，并添加全局逻辑时钟。这些变更为数据库提供了在实现 ACID 符合事务之前所需的基础。

推荐回调 API 而不是核心 API 的主要原因是应用程序中所需的复杂性和额外编码。这些 API 之间的差异在表 8-1 中总结。

表 8-1\. 核心 API 与回调 API 的比较

| 核心 API | 回调 API |
| --- | --- |
| 需要显式调用以开始事务并提交事务。 | 开始一个事务，执行指定的操作，并提交（或在出错时中止）。 |
| 不含有处理 `TransientTransactionError` 和 `UnknownTransactionCommitResult` 的错误处理逻辑，而是提供了灵活性以便为这些错误添加自定义错误处理。 | 自动包含了处理 `TransientTransactionError` 和 `UnknownTransactionCommitResult` 的错误处理逻辑。 |
| 需要将显式逻辑会话传递给特定事务的 API。 | 需要将显式逻辑会话传递给特定事务的 API。 |

要理解这两个 API 之间的差异，我们可以使用一个简单的电子商务站点事务示例来比较这些 API，在这个示例中，订单被下单并在销售时从可用库存中移除相应的商品。这涉及到单个事务中不同集合的两个文档。我们事务示例的核心操作将是：

```
    orders.insert_one({"sku": "abc123", "qty": 100}, session=session)
    inventory.update_one({"sku": "abc123", "qty": {"$gte": 100}},
                         {"$inc": {"qty": -100}}, session=session)
```

首先，让我们看看如何在 Python 中为我们的事务示例使用核心 API。我们事务的两个操作在以下程序清单的第 1 步中被突出显示：

```
# Define the uriString using the DNS Seedlist Connection Format 
# for the connection
uri = 'mongodb+srv://server.example.com/'
client = MongoClient(uriString)

my_wc_majority = WriteConcern('majority', wtimeout=1000)

# Prerequisite / Step 0: Create collections, if they don't already exist. 
# CRUD operations in transactions must be on existing collections.

client.get_database( "webshop",
                     write_concern=my_wc_majority).orders.insert_one({"sku":
                     "abc123", "qty":0})
client.get_database( "webshop",
                     write_concern=my_wc_majority).inventory.insert_one(
                     {"sku": "abc123", "qty": 1000})

# Step 1: Define the operations and their sequence within the transaction
def update_orders_and_inventory(my_session):
    orders = session.client.webshop.orders
    inventory = session.client.webshop.inventory

    with session.start_transaction(
            read_concern=ReadConcern("snapshot"),
            write_concern=WriteConcern(w="majority"),
            read_preference=ReadPreference.PRIMARY):

        orders.insert_one({"sku": "abc123", "qty": 100}, session=my_session)
        inventory.update_one({"sku": "abc123", "qty": {"$gte": 100}},
                             {"$inc": {"qty": -100}}, session=my_session)
        commit_with_retry(my_session)

# Step 2: Attempt to run and commit transaction with retry logic
def commit_with_retry(session):
    while True:
        try:
            # Commit uses write concern set at transaction start.
            session.commit_transaction()
            print("Transaction committed.")
            break
        except (ConnectionFailure, OperationFailure) as exc:
            # Can retry commit
            if exc.has_error_label("UnknownTransactionCommitResult"):
                print("UnknownTransactionCommitResult, retrying "
                      "commit operation ...")
                continue
            else:
                print("Error during commit ...")
                raise

# Step 3: Attempt with retry logic to run the transaction function txn_func
def run_transaction_with_retry(txn_func, session):
    while True:
        try:
            txn_func(session)  # performs transaction
            break
        except (ConnectionFailure, OperationFailure) as exc:
            # If transient error, retry the whole transaction
            if exc.has_error_label("TransientTransactionError"):
                print("TransientTransactionError, retrying transaction ...")
                continue
            else:
                raise

# Step 4: Start a session.
with client.start_session() as my_session:

# Step 5: Call the function 'run_transaction_with_retry' passing it the function
# to call 'update_orders_and_inventory' and the session 'my_session' to associate
# with this transaction.

    try:
        run_transaction_with_retry(update_orders_and_inventory, my_session)
    except Exception as exc:
        # Do something with error. The error handling code is not
        # implemented for you with the Core API.
        raise
```

现在，让我们看看如何在 Python 中使用回调 API 来完成同样的事务示例。我们事务的两个操作在以下程序清单的第 1 步中被突出显示：

```
# Define the uriString using the DNS Seedlist Connection Format 
# for the connection
uriString = 'mongodb+srv://server.example.com/'
client = MongoClient(uriString)

my_wc_majority = WriteConcern('majority', wtimeout=1000)

# Prerequisite / Step 0: Create collections, if they don't already exist.
# CRUD operations in transactions must be on existing collections.

client.get_database( "webshop",
                     write_concern=my_wc_majority).orders.insert_one({"sku":
                     "abc123", "qty":0})
client.get_database( "webshop",
                     write_concern=my_wc_majority).inventory.insert_one(
                     {"sku": "abc123", "qty": 1000})

# Step 1: Define the callback that specifies the sequence of operations to
# perform inside the transactions.

def callback(my_session):
    orders = my_session.client.webshop.orders
    inventory = my_session.client.webshop.inventory

    # Important:: You must pass the session variable 'my_session' to 
    # the operations.

    orders.insert_one({"sku": "abc123", "qty": 100}, session=my_session)
    inventory.update_one({"sku": "abc123", "qty": {"$gte": 100}},
                         {"$inc": {"qty": -100}}, session=my_session)

#. Step 2: Start a client session.

with client.start_session() as session:

# Step 3: Use with_transaction to start a transaction, execute the callback,
# and commit (or abort on error).

    session.with_transaction(callback,
                             read_concern=ReadConcern('local'),
                             write_concern=my_write_concern_majority,
                             read_preference=ReadPreference.PRIMARY)
}
```

###### 注意

在 MongoDB 的多文档事务中，您只能对现有集合或数据库执行读/写（CRUD）操作。如我们的示例所示，如果您希望将其插入到事务中，您必须首先在事务之外创建集合。不允许创建、删除或索引操作在事务中执行。

# 为您的应用程序调整事务限制

在使用事务时，有几个重要的参数需要注意。它们可以进行调整，以确保您的应用程序能够最优化地使用事务。

## 时间和 Oplog 大小限制

MongoDB 事务中存在两个主要类别的限制。第一个与事务的时间限制有关，控制特定事务可以运行的时间，事务等待获取锁的时间以及所有事务将运行的最大长度。第二个类别特别与 MongoDB oplog 条目和单个条目的大小限制相关。

时间限制

事务的默认最大运行时间是一分钟或更短。可以通过修改由`mongod`实例级别的`transactionLifetimeLimitSeconds`控制的限制来增加此时间。在分片集群的情况下，必须在所有分片副本集成员上设置此参数。超过此时间后，事务将被视为过期，并将由定期运行的清理过程中止。清理过程每 60 秒运行一次，或者每`transactionLifetimeLimitSeconds/2`，以较低者为准。

若要在事务上明确设置时间限制，建议在`commitTransaction`上指定`maxTimeMS`。如果未设置`maxTimeMS`，则将使用`transactionLifetimeLimitSeconds`，或者如果设置了但超出`transactionLifetimeLimitSeconds`，则将使用`transactionLifetimeLimitSeconds`。

事务在运行操作所需的锁之前等待的默认最大时间是 5 毫秒。可以通过修改由`maxTransactionLockRequestTimeoutMillis`控制的限制来增加此时间。如果事务在此时间内无法获取所需的锁，则会中止。`maxTransactionLockRequestTimeoutMillis`可以设置为`0`、`-1`或大于`0`的数字。将其设置为`0`意味着如果事务无法立即获取所需的所有锁，则会中止。设置为`-1`将使用由`maxTimeMS`指定的操作特定超时。任何大于`0`的数字将配置等待时间为以秒为单位的指定时间，作为事务尝试获取所需锁的期间。

Oplog 大小限制

MongoDB 将为事务中的写操作创建所需的操作日志（oplog）条目。然而，每个 oplog 条目必须在 16MB 的 BSON 文档大小限制内。

事务在 MongoDB 中提供了一个有用的功能，用于确保一致性，但应该与丰富的文档模型一起使用。这种模型的灵活性以及使用诸如模式设计等最佳实践将有助于避免在大多数情况下使用事务。事务是一个强大的功能，在应用程序中最好节制使用。

^(1) 本文的作者是分片的高级软件工程师 Misha Tyulenev；分布式系统副总裁 Andy Schwerin；分布式系统首席产品经理 Asya Kamsky；分片的高级软件工程师 Randolph Tan；分布式系统产品经理 Alyson Cabral；以及分片的软件工程师 Jack Mulrow。
