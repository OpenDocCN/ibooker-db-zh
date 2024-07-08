# 第二十一章：在生产环境中设置 MongoDB

在第二章中，我们讨论了启动 MongoDB 的基础知识。本章将更详细地介绍在生产环境中设置 MongoDB 时重要的选项，包括：

+   常用选项

+   启动和关闭 MongoDB

+   与安全相关的选项

+   日志考虑

# 从命令行启动

MongoDB 服务器使用`mongod`可执行文件启动。`mongod`有许多可配置的启动选项；要查看所有选项，请从命令行运行`mongod --help`。其中一些选项被广泛使用且非常重要：

`--dbpath`

指定替代数据目录使用；默认为*/data/db/*（或者在 Windows 上，在 MongoDB 二进制卷上的*\data\db\*）。每个`mongod`进程在机器上需要自己的数据目录，所以如果您在一台机器上运行三个`mongod`实例，则需要三个单独的数据目录。当`mongod`启动时，它会在其数据目录中创建一个*mongod.lock*文件，这会阻止其他任何`mongod`进程使用该目录。如果您尝试使用相同的数据目录启动另一个 MongoDB 服务器，它将报错：

```
exception in initAndListen: DBPathInUse: Unable to lock the
      lock file: \ data/db/mongod.lock (Resource temporarily unavailable).
      Another mongod instance is already running on the 
      data/db directory,
      \ terminating
```

`--port`

指定服务器侦听的端口号。默认情况下，`mongod`使用端口 27017，这个端口除了其他`mongod`进程外，不太可能被其他进程使用。如果您希望在单台机器上运行多个`mongod`进程，则需要为每个进程指定不同的端口号。如果您尝试在已被使用的端口上启动`mongod`，它将报错：

```
Failed to set up listener: SocketException: Address already in use.
```

`--fork`

在基于 Unix 的系统上，分叉服务器进程，将 MongoDB 作为守护程序运行。

如果您首次启动*mongod*（使用空数据目录），文件系统可能需要几分钟来分配数据库文件。父进程在分叉直到预分配完成并且*mongod*准备好开始接受连接之前不会返回。因此，*fork*可能会看起来像是挂起。您可以 tail 日志查看它在做什么。如果指定了`--fork`，必须使用`--logpath`。

`--logpath`

将所有输出发送到指定的文件而不是在命令行上输出。如果目录有写权限，这将创建文件（如果不存在）。如果日志文件已存在，它将覆盖日志文件，擦除任何较旧的日志条目。如果您想保留旧日志，请使用`--logappend`选项，还要使用`--logpath`（强烈推荐）。

`--directoryperdb`

把每个数据库放在它自己的目录中。这样，如果需要或希望的话，可以把不同的数据库挂载到不同的磁盘上。常见的用法包括把本地数据库放在自己的磁盘上（复制）或者在原始磁盘填满时将数据库移到不同的磁盘上。您还可以将负载更高的数据库放在更快的磁盘上，将负载较低的数据库放在较慢的磁盘上。基本上，这使您在以后移动这些内容时具有更大的灵活性。

`--config`

使用配置文件添加未在命令行上指定的其他选项。通常用于确保在重新启动之间选项保持不变。有关详细信息，请参阅“基于文件的配置”。

例如，要将服务器作为守护进程启动，并监听端口 5586，并将所有输出发送到*mongodb.log*，我们可以运行以下命令：

```
$ ./mongod --dbpath data/db --port 5586 --fork --logpath
    mongodb.log --logappend 2019-09-06T22:52:25.376-0500 I CONTROL [main]
    Automatically disabling TLS 1.0, \ to force-enable TLS 1.0 specify
    --sslDisabledProtocols 'none' about to fork child process, waiting until
    server is ready for connections. forked process: 27610 child process
    started successfully, parent exiting
```

当您首次安装并启动 MongoDB 时，建议查看日志。这可能是一个容易忽略的事情，特别是如果 MongoDB 是从 init 脚本启动的，但日志通常包含防止后续错误发生的重要警告。如果在 MongoDB 启动时的日志中没有看到任何警告，则一切正常。（启动警告也会出现在 shell 启动时。）

如果启动横幅中有任何警告，请注意。MongoDB 会警告您有各种问题：您正在运行在 32 位机器上（MongoDB 不适用于此），您已启用 NUMA（这可能使您的应用程序速度变慢），或者您的系统不允许足够的打开文件描述符（MongoDB 使用大量文件描述符）。

当您重新启动数据库时，日志前言不会改变，所以一旦您了解它们的含义，可以从 init 脚本运行 MongoDB 并忽略日志。但是，每次安装、升级或从崩溃中恢复时，最好再次检查，以确保 MongoDB 和您的系统处于同一页面。

当您启动数据库时，MongoDB 将向*local.startup_log*集合写入一个描述 MongoDB 版本、底层系统和使用的标志的文档。我们可以使用*mongo* shell 查看此文档：

```
> use local
switched to db local
> db.startup_log.find().sort({startTime: -1}).limit(1).pretty()
{
    "_id" : "server1-1544192927184",
    "hostname" : "server1.example.net",
    "startTime" : ISODate("2019-09-06T22:50:47Z"),
    "startTimeLocal" : "Fri Sep  6 22:57:47.184",
    "cmdLine" : {
        "net" : {
            "port" : 5586
        },
        "processManagement" : {
            "fork" : true
        },
        "storage" : {
            "dbPath" : "data/db"
        },
        "systemLog" : {
            "destination" : "file",
            "logAppend" : true,
            "path" : "mongodb.log"
        }
    },
    "pid" : NumberLong(27278),
    "buildinfo" : {
        "version" : "4.2.0",
        "gitVersion" : "a4b751dcf51dd249c5865812b390cfd1c0129c30",
        "modules" : [
            "enterprise"
        ],
        "allocator" : "system",
        "javascriptEngine" : "mozjs",
        "sysInfo" : "deprecated",
        "versionArray" : [
            4,
            2,
            0,
            0
        ],
        "openssl" : {
            "running" : "Apple Secure Transport"
        },
        "buildEnvironment" : {
            "distmod" : "",
            "distarch" : "x86_64",
            "cc" : "gcc: Apple LLVM version 8.1.0 (clang-802.0.42)",
            "ccflags" : "-mmacosx-version-min=10.10 -fno-omit\
 -frame-pointer -fno-strict-aliasing \
 -ggdb -pthread -Wall 
 -Wsign-compare -Wno-unknown-pragmas \
 -Winvalid-pch -Werror -O2 -Wno-unused\
 -local-typedefs -Wno-unused-function 
 -Wno-unused-private-field \
 -Wno-deprecated-declarations \
 -Wno-tautological-constant-out-of\
 -range-compare 
 -Wno-unused-const-variable -Wno\
 -missing-braces -Wno-inconsistent\
 -missing-override 
 -Wno-potentially-evaluated-expression \
 -Wno-exceptions -fstack-protector\
 -strong -fno-builtin-memcmp",
            "cxx" : "g++: Apple LLVM version 8.1.0 (clang-802.0.42)",
            "cxxflags" : "-Woverloaded-virtual -Werror=unused-result \
 -Wpessimizing-move -Wredundant-move \
 -Wno-undefined-var-template -stdlib=libc++ \
 -std=c++14",
            "linkflags" : "-mmacosx-version-min=10.10 -Wl, \
 -bind_at_load -Wl,-fatal_warnings \
 -fstack-protector-strong \
 -stdlib=libc++",
            "target_arch" : "x86_64",
            "target_os" : "macOS"
        },
        "bits" : 64,
        "debug" : false,
        "maxBsonObjectSize" : 16777216,
        "storageEngines" : [
            "biggie",
            "devnull",
            "ephemeralForTest",
            "inMemory",
            "queryable_wt",
            "wiredTiger"
        ]
    }
}
```

这一集合对于跟踪升级和行为变化非常有用。

## 基于文件的配置

MongoDB 支持从文件中读取配置信息。如果您有一组要使用的选项或正在自动化启动 MongoDB 的任务，则此选项非常有用。要告诉服务器从配置文件获取选项，请使用`-f`或`--config`标志。例如，运行`mongod --config ~/.mongodb.conf`以使用*~/.mongodb.conf*作为配置文件。

配置文件中支持的选项与命令行接受的选项相同。但是，格式不同。从 MongoDB 2.6 开始，MongoDB 配置文件使用 YAML 格式。以下是一个示例配置文件：

```
systemLog:
   destination: file
   path: "mongod.log"
   logAppend: true
storage:
   dbPath: data/db
processManagement:
   fork: true
net:
   port: 5586
...
```

此配置文件指定了我们在使用常规命令行参数启动时使用的相同选项。请注意，这些选项在我们在前一节中查看的 *startup_log* 集合文档中也有所反映。唯一的真正区别是这些选项是使用 JSON 而不是 YAML 指定的。

在 MongoDB 4.2 中，添加了扩展指令，允许加载特定配置文件选项或加载整个配置文件。扩展指令的优点是不必直接在配置文件中存储机密信息，例如密码和安全证书。`--configExpand` 命令行选项启用此功能，并必须包括您希望启用的扩展指令。`__rest` 和 `__exec` 是 MongoDB 中扩展指令的当前实现。`__rest` 扩展指令从 REST 端点加载特定配置文件值或加载整个配置文件。`__exec` 扩展指令从 shell 或终端命令加载特定配置文件值或加载整个配置文件。

# 停止 MongoDB

安全地停止运行中的 MongoDB 服务器至少与启动一个同样重要。有几种有效的方法可以实现这一点。

关闭运行中服务器的最干净方式是使用 `shutdown` 命令，`{"shutdown" : 1}`。这是一个管理员命令，必须在 *admin* 数据库上运行。Shell 提供了一个帮助函数来简化此操作：

```
> use admin
switched to db admin
> db.shutdownServer()
server should be down...
```

在主服务器上运行时，`shutdown` 命令会降级主服务器并等待辅助服务器追赶上来，然后再关闭服务器。这最大程度地减少了回滚的机会，但不能保证关闭一定成功。如果没有可以在几秒钟内追赶上来的辅助服务器，`shutdown` 命令将失败，原（主）服务器将不会关闭：

```
> db.shutdownServer()
{
    "closest" : NumberLong(1349465327),
    "difference" : NumberLong(20),
    "errmsg" : "no secondaries within 10 seconds of my optime",
    "ok" : 0
}
```

您可以使用 `force` 选项强制执行 `shutdown` 命令以关闭主服务器：

```
db.adminCommand({"shutdown" : 1, "force" : true})
```

这相当于发送 SIGINT 或 SIGTERM 信号（这三个选项都会导致干净的关闭，但可能存在未复制的数据）。如果服务器作为终端中的前台进程运行，可以通过按 Ctrl-C 发送 SIGINT。否则，可以使用像 `kill` 这样的命令来发送信号。如果 *mongod* 的 PID 是 10014，则命令为 `kill -2 10014`（SIGINT）或 `kill 10014`（SIGTERM）。

当 *mongod* 收到 SIGINT 或 SIGTERM 时，它将进行干净的关闭。这意味着它将等待任何运行中的操作或文件预分配完成（这可能需要一些时间），关闭所有打开的连接，将所有数据刷新到磁盘并停止。

# 安全性

不要设置公开可访问的 MongoDB 服务器。你应该尽可能严格地限制外部世界与 MongoDB 之间的访问。最好的方法是设置防火墙，并只允许 MongoDB 在内部网络地址上可达。第二十四章涵盖了 MongoDB 服务器和客户端之间必须允许的连接。

除了防火墙外，你可以向配置文件添加几个选项来增强其安全性：

`--bind_ip`

指定你希望 MongoDB 监听的接口。通常你希望这是一个内部 IP：应用服务器和集群的其他成员可以访问的 IP，但对外界不可访问。如果在同一台机器上运行应用服务器，对*mongos*进程来说*localhost*是可以的。对于配置服务器和分片，它们需要从其他机器可寻址，因此应使用非*localhost*地址。

从 MongoDB 3.6 开始，默认情况下*mongod*和*mongos*进程绑定到*localhost*。当仅绑定到*localhost*时，*mongod*和*mongos*只接受来自同一台机器上运行的客户端的连接。这有助于限制未安全配置的 MongoDB 实例的暴露。要绑定到其他地址，请使用`net.bindIp`配置文件设置或`--bind_ip`命令行选项来指定主机名或 IP 地址列表。

`--nounixsocket`

禁用在 UNIX 域套接字上的监听。如果你不计划通过文件系统套接字连接，最好禁止它。你只会在同时运行应用服务器的机器上通过文件系统套接字进行连接：必须是本地才能使用文件系统套接字。

`--noscripting`

禁用服务器端 JavaScript 执行。有些与 MongoDB 报告的安全问题与 JavaScript 有关，因此如果你的应用允许的话，通常最安全的做法是禁用它。

###### 注意

一些 shell 辅助程序假设服务器上有 JavaScript 可用，特别是`sh.status()`。如果尝试在禁用 JavaScript 的情况下运行任何这些辅助程序，你会看到错误。

## 数据加密

MongoDB Enterprise 支持数据加密。这些选项在 MongoDB 社区版本中不受支持。

数据加密过程包括以下步骤：

+   生成主密钥。

+   为每个数据库生成密钥。

+   使用数据库密钥加密数据。

+   使用主密钥加密数据库密钥。

当使用数据加密时，所有数据文件在文件系统中都是加密的。数据只在内存和传输过程中解密。要加密 MongoDB 的所有网络流量，你可以使用 TLS/SSL。MongoDB Enterprise 用户可以向其配置文件中添加的数据加密选项有：

`--enableEncryption`

启用 WiredTiger 存储引擎中的加密。使用此选项，存储在内存和磁盘上的数据将被加密。有时也称为“静态加密”。您必须将其设置为`true`以传递加密密钥并配置加密。此选项默认为`false`。

`--encryptionCipherMode`

在 WiredTiger 中设置数据加密的密码模式。有两种可用模式：AES256-CBC 和 AES256-GCM。AES256-CBC 是 256 位高级加密标准的密码分组链接模式的缩写。AES256-GCM 使用 Galois/Counter 模式。两者都是标准的加密算法。从 MongoDB 4.0 开始，在 Windows 上的 MongoDB 企业版不再支持 AES256-GCM。

`--encryptionKeyFile`

如果您正在使用与密钥管理互操作性协议（KMIP）不同的过程管理密钥，则需指定本地密钥文件的路径。

MongoDB 企业版还支持使用 KMIP 进行密钥管理。本书不涵盖 KMIP 的讨论。请参阅[使用 MongoDB 的 KMIP 的详细文档](https://oreil.ly/TeA4t)。

## SSL 连接

正如我们在第十八章中所见，MongoDB 支持使用 TLS/SSL 进行传输加密。此功能在 MongoDB 的所有版本中均可用。默认情况下，连接到 MongoDB 的数据是未加密的。但 TLS/SSL 确保传输加密。MongoDB 使用操作系统上可用的本机 TLS/SSL 库。使用`--tlsMode`选项及相关选项配置 TLS/SSL。有关更多详细信息，请参阅第十八章，并查阅您的驱动程序文档，了解如何使用您的语言创建 TLS/SSL 连接。

# 日志记录

默认情况下，*mongod*将其日志发送到 stdout。大多数初始化脚本使用`--logpath`选项将日志发送到文件中。如果在单台机器上有多个 MongoDB 实例（例如，一个*mongod*和一个*mongos*），请确保它们的日志存储在单独的文件中。确保您知道日志的存储位置并具有文件的读取权限。

MongoDB 会生成大量的日志消息，但请不要使用`--quiet`选项（该选项会抑制部分日志消息）。通常情况下保持默认的日志级别是最佳选择：提供了足够的基本调试信息（比如为什么运行慢，为什么启动失败等），但日志占用的空间不会过多。

如果您正在调试应用程序的特定问题，有几种选项可以从日志中获取更多信息。您可以通过运行`setParameter`命令或通过将其作为字符串传递使用`--setParameter`选项在启动时设置日志级别。

```
> db.adminCommand({"setParameter" : 1, "logLevel" : 3})
```

您还可以为特定组件更改日志级别。如果您正在调试应用程序的特定方面并需要更多信息，但只来自该组件，则这很有帮助。在此示例中，我们将默认日志详细级别设置为 1，查询组件的详细级别设置为 2：

```
> db.adminCommand({"setParameter" : 1, logComponentVerbosity:
        { verbosity: 1, query: { verbosity: 2 }}})
```

记得在调试完成后将日志级别调回到 0，否则你的日志可能会变得不必要地嘈杂。你可以将级别调到 5，此时 `mongod` 将打印出它执行的几乎每个操作，包括处理的每个请求的内容。这可能会导致大量的 I/O，因为 *mongod* 将所有内容写入日志文件，这会减慢繁忙系统的速度。如果需要查看每个操作的实时发生情况，打开分析是一个更好的选择。

默认情况下，MongoDB 记录超过 100 毫秒的查询信息。如果 100 毫秒对于你的应用程序来说太短或太长，可以使用 `setProfilingLevel` 更改阈值：

```
> // Only log queries that take longer than 500 ms
> db.setProfilingLevel(1, 500)
{ "was" : 0, "slowms" : 100, "ok" : 1 }
> db.setProfilingLevel(0)
{ "was" : 1, "slowms" : 500, "ok" : 1 }
```

第二行将关闭分析，但第一行中给出的毫秒值将继续作为日志的阈值（跨所有数据库）。你也可以通过使用 `--slowms` 选项重新启动 MongoDB 来设置此参数。

最后，设置一个 cron 作业，每天或每周轮换一次日志。如果 MongoDB 是通过 `--logpath` 启动的，向该进程发送 SIGUSR1 信号将使其轮换日志。还有一个 `logRotate` 命令可以做同样的事情：

```
> db.adminCommand({"logRotate" : 1})
```

如果 MongoDB 没有通过 `--logpath` 启动，则无法轮换日志。
