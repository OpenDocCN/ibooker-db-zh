## 附录 B. 其他资源和参考

在过去的 11 章和附录 A 中，我们涵盖了与 Redis 相关的各种主题。作为我们解决的问题的一部分，我们介绍了你可能不熟悉的概念，并提供了更多信息以供你学习。

本附录将所有这些参考以及其他内容汇集到一个位置，以便于访问与我们所涵盖的主题相关的其他软件、库、服务以及/或文档，按一般主题分组。

### B.1\. 帮助论坛

这些 URL 将带你去论坛，你可以在那里获得使用 Redis 或在其他章节中的示例的帮助：

+   [`groups.google.com/forum/#!forum/redis-db`](https://groups.google.com/forum/#!forum/redis-db)—Redis 论坛

+   [`www.manning-sandbox.com/forum.jspa?forumID=809`](http://www.manning-sandbox.com/forum.jspa?forumID=809)—《Redis 实战》的 Manning 论坛

### B.2\. 介绍性主题

这个介绍性主题列表涵盖了关于 Redis 及其使用的一些基本信息：

+   [`redis.io/`](http://redis.io/)—主要 Redis 网站

+   [`redis.io/commands`](http://redis.io/commands)—完整的 Redis 命令列表

+   [`redis.io/clients`](http://redis.io/clients)—Redis 客户端列表

+   [`redis.io/documentation`](http://redis.io/documentation)—关于 Lua、发布/订阅、复制、持久化等方面的主要参考页面

+   [`github.com/dmajkic/redis/`](http://github.com/dmajkic/redis/)—杜桑·马吉奇将 Redis 移植到 Windows 的版本

+   [`github.com/MSOpenTech/redis/`](http://github.com/MSOpenTech/redis/)—微软官方将 Redis 移植到 Windows 的版本

这个介绍性主题列表涵盖了关于 Python 及其使用的一些基本信息：

+   [`www.python.org/`](http://www.python.org/)—Python 编程语言的主要网页

+   [`docs.python.org/`](http://docs.python.org/)—主要 Python 文档页面

+   [`docs.python.org/tutorial/`](http://docs.python.org/tutorial/)—针对新用户的 Python 教程

+   [`docs.python.org/reference/`](http://docs.python.org/reference/)—Python 语言参考，提供关于语法和语义的详细信息

+   [`mng.bz/TTKb`](http://mng.bz/TTKb)—生成器表达式

+   [`mng.bz/I31v`](http://mng.bz/I31v)—Python 可加载模块教程

+   [`mng.bz/9wXM`](http://mng.bz/9wXM)—定义函数

+   [`mng.bz/q7eo`](http://mng.bz/q7eo)—可变参数列表

+   [`mng.bz/1jLF`](http://mng.bz/1jLF)—可变参数和关键字

+   [`mng.bz/0rmB`](http://mng.bz/0rmB)—列表推导式

+   [`mng.bz/uIdf`](http://mng.bz/uIdf)—Python 生成器

+   [`mng.bz/1XMr`](http://mng.bz/1XMr)—函数和方法装饰器

### B.3\. 队列和其他库

+   [`celeryproject.org/`](http://celeryproject.org/)—支持多个后端的 Python 队列库，包括 Redis

+   [`github.com/josiahcarlson/rpqueue/`](https://github.com/josiahcarlson/rpqueue/)—仅适用于 Redis 的 Python 队列库

+   [`github.com/resque/resque`](https://github.com/resque/resque)—标准的 Ruby + Redis 队列

+   [`www.rabbitmq.com/`](http://www.rabbitmq.com/)—支持多种语言的队列服务器

+   [`activemq.apache.org/`](http://activemq.apache.org/)—支持多种语言的队列服务器

+   [`github.com/Doist/bitmapist`](https://github.com/Doist/bitmapist)—支持强大的位图分析功能

### B.4\. 数据可视化和记录

+   [`www.jqplot.com/`](http://www.jqplot.com/)—与 jQuery 一起使用的开源绘图库

+   [`www.highcharts.com/`](http://www.highcharts.com/)—免费/商业绘图库

+   [`dygraphs.com/`](http://dygraphs.com/)—开源绘图库

+   [`d3js.org/`](http://d3js.org/)—通用开源数据可视化库

+   [`graphite.wikidot.com/`](http://graphite.wikidot.com/)—统计收集和可视化库

### B.5\. 数据源

在第五章中，我们使用了 IP 到位置信息的数据库。此列表包括对该数据的引用，以及可能或可能不更新的其他数据源：

+   [`dev.maxmind.com/geoip/geolite`](http://dev.maxmind.com/geoip/geolite)—IP 地址到地理位置信息，首次在第五章中使用

+   [`www.hostip.info/dl/`](http://www.hostip.info/dl/)—免费下载和使用的 IP 到地理位置信息数据库

+   [`software77.net/geo-ip/`](http://software77.net/geo-ip/)—另一个免费的 IP 到地理位置信息数据库

### B.6\. Redis 经验和文章

+   [`mng.bz/2ivv`](http://mng.bz/2ivv)—跨数据中心 Redis 复制压缩的示例架构

+   [`mng.bz/LCgm`](http://mng.bz/LCgm)—使用 Redis 进行实时更新

+   [`mng.bz/UgAD`](http://mng.bz/UgAD)—使用 Redis `STRING`s 存储一些实时指标

+   [`mng.bz/1OJ7`](http://mng.bz/1OJ7)—Instagram 在 Redis 中存储大量键值对的体验

+   [`mng.bz/X564`](http://mng.bz/X564)—一些 Redis 表现优异的问题的简要总结，其中一些我们在前面的章节中已经讨论过

+   [`mng.bz/oClc`](http://mng.bz/oClc)—在 Craigslist 中将数据分片到 Redis

+   [`mng.bz/07kX`](http://mng.bz/07kX)—Redis 在多个部分中使用的示例，用于在手机和桌面之间同步照片

+   [`mng.bz/4dgD`](http://mng.bz/4dgD)—Disqus 在生产中使用 Redis 的一种方式

+   [`mng.bz/21iE`](http://mng.bz/21iE)—使用 Redis 存储 RSS 源信息

+   [`mng.bz/L254`](http://mng.bz/L254)—早期使用 Redis `LIST`s 作为存储最近过滤的 Twitter 消息的示例
