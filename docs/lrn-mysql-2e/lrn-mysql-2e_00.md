# 前言

数据库管理系统是许多公司核心的一部分。即使一家企业不是以技术为中心，它也需要以快速、安全和可靠的方式存储、访问和操作数据。由于 COVID-19 大流行，许多传统上抵制数字转型的领域，如许多国家的司法系统，现在由于旅行和会议限制而通过技术进行整合，在线购物和远程工作比以往任何时候都更受欢迎。

但并非只有灾难推动了如此深远的变化。随着 5G 的到来，我们很快将有更多的机器连接到互联网，而不是人类。已经有大量数据被收集、存储和用于训练机器学习模型、人工智能等等。我们正处在下一个革命的开端。

出现了几种数据库类型，帮助存储更多数据，特别是非结构化数据，包括 MongoDB、Cassandra 和 Redis 等 NoSQL 数据库。然而，传统的 SQL 数据库仍然很受欢迎，并且没有迹象表明它们会在近期消失。在 SQL 的世界中，无疑最受欢迎的开源解决方案是 MySQL。

本书的两位作者都与来自世界各地的许多客户合作过。在此过程中，我们学到了许多经验教训，并且涉及了大量用例，从关键任务的单体应用到更简单的微服务应用。本书充满了我们认为大多数读者在日常活动中会发现有帮助的提示和建议。

# 本书的读者对象

本书主要是为首次使用 MySQL 或将其作为第二数据库学习的人士编写的。如果您是首次进入数据库领域，第一章将向您介绍数据库设计概念，并向您展示如何在不同操作系统和云中部署 MySQL。

对于从其他生态系统（如 Postgres、Oracle 或 SQL Server）过来的人，本书涵盖了备份、高可用性和灾难恢复策略。

我们希望所有读者也会发现本书是学习或复习基础知识的好伴侣，从架构到生产环境的建议。

# 本书的组织结构

我们介绍了许多主题，从基本的安装过程、数据库设计、备份和恢复到 CPU 性能分析和错误调查。我们将本书分为四个主要部分：

1.  开始使用 MySQL

1.  使用 MySQL

1.  MySQL 在生产环境中

1.  杂项主题

让我们看看我们如何组织章节。

## 开始使用 MySQL

第一章，*安装 MySQL*解释了如何在不同操作系统上安装和配置 MySQL 软件。本章提供了比大多数书籍更详细的信息。我们知道，对于刚开始使用 MySQL 的人来说，他们通常对各种 Linux 发行版和安装选项不熟悉，并且在 MySQL 上运行“hello world”比编译任何编程语言的 hello world 需要更多步骤。您将了解如何在 Linux、Windows、macOS 和 Docker 上设置 MySQL，并快速部署实例以进行测试。

## 使用 MySQL

在我们开始创建和使用数据库之前，我们将在第二章，*数据库建模和设计*中看到如何进行正确的数据库设计。您将学习如何访问数据库的特性，并查看数据库中信息项之间的关系。您将看到，糟糕的数据库设计很难更改，并可能导致性能问题。我们将介绍强实体和弱实体及它们的关系（*外键*），并解释规范化的过程。本章还展示了如何下载和配置数据库示例，如`sakila`、`world`和`employees`。

在第三章，*基本 SQL*中，我们探讨了 CRUD（创建、读取、更新和删除）操作中的著名 SQL 命令。您将学习如何从现有的 MySQL 数据库中读取数据，存储数据，并操作现有数据。

在第四章，*数据库结构操作*，我们解释如何创建新的 MySQL 数据库，以及如何创建和修改表、索引和其他数据库结构。

第五章，*高级查询*涵盖更高级的操作，如使用嵌套查询和不同的 MySQL 数据库引擎。本章将使您能够执行更复杂的查询。

## MySQL 在生产环境中的应用

现在您已经知道如何安装 MySQL 并操纵数据，下一步是了解 MySQL 如何处理对同一数据的并发访问。我们将在第六章，*事务和锁*中探讨隔离、事务和死锁的概念。

在第七章，*MySQL 更多操作*，您将看到更复杂的查询，可以在 MySQL 中执行，并了解如何观察查询计划以检查查询是否高效。我们还将解释 MySQL 中可用的不同引擎（InnoDB 和 MyISAM 是最著名的两种）。

在第八章，*用户管理与权限*中，您将学习如何在数据库中创建和删除用户。这一步骤在安全方面非常重要，因为权限过大的用户可能会对数据库和公司的声誉造成严重损害。您将了解如何建立安全策略，授予和撤销权限，以及限制对特定网络 IP 的访问。

第九章，*使用选项文件*涵盖了 MySQL 配置文件，或称*选项文件*，其中包含启动 MySQL 和优化其性能所需的所有必要参数。熟悉 MySQL 的人会认识到 */etc/my.cnf* 配置文件。您还将看到可以使用特殊选项文件配置用户访问权限。

没有备份策略的数据库迟早会面临灾难。在第十章，*备份与恢复*中，我们讨论了不同类型的备份（*逻辑*与*物理*），执行此任务的可选项以及适合大型生产数据库的方法。

第十一章，*服务器配置与调优*详细讨论了在设置新服务器时需要注意的关键参数。我们提供了相应的公式，并帮助您确定是否为数据库工作负载选择了正确的参数值。

## 杂项话题

既然基础知识已经确立，现在是时候进一步深入了。第十二章，*监控 MySQL 服务器*教您如何监控数据库并从中收集数据。由于数据库工作负载行为可能会根据用户数量、事务和正在处理的数据量而变化，因此识别哪个资源已经饱和及其原因至关重要。

第十三章，*高可用性*解释了如何复制服务器以提供高可用性。我们还介绍了集群概念，重点介绍了两种解决方案：InnoDB Cluster 和 Galera/PXC Cluster。

第十四章，*MySQL 在云中*将 MySQL 的应用扩展到了云端。您将了解到数据库即服务（DBaaS）选项，以及如何使用三大主要云服务提供商（Amazon Web Services（AWS）、Google Cloud Platform（GCP）和 Microsoft Azure）提供的托管数据库服务。

在第十五章，*MySQL 负载均衡*中，我们将向您展示最常用的工具，以在不同的 MySQL 服务器之间分发查询，从而进一步提升 MySQL 的性能。

最后，在第十六章，*杂项话题*中，我们介绍了更高级的分析方法和工具，以及一些编程内容。在本章中，我们将讨论 MySQL Shell、火焰图以及如何分析 bug。

# 本书中使用的约定

本书使用以下印刷约定：

*斜体*

表示新术语、网址、电子邮件地址、文件名和文件扩展名。

`等宽字体`

用于程序列表，以及在段落中引用程序元素，如变量或函数名称、数据库、数据类型、环境变量、语句和关键字。

`**等宽粗体**`

显示用户应按字面输入的命令或其他文本。

`*等宽斜体*`

显示应用用户提供的值或由上下文确定的值的文本。

###### 提示

此元素表示提示或建议。

###### 注意

此元素表示一般注释。

###### 警告

此元素表示警告或注意事项。

# 使用代码示例

可以从[*https://github.com/learning-mysql-2nd/learning-mysql-2nd*](https://github.com/learning-mysql-2nd/learning-mysql-2nd)下载代码示例。

如果您有技术问题或在使用代码示例时遇到问题，请发送电子邮件至*bookquestions@oreilly.com*。

本书旨在帮助您完成工作。一般情况下，如果本书提供了示例代码，您可以在您的程序和文档中使用它。除非您复制了代码的大部分内容，否则您无需联系我们以获取许可。例如，编写一个使用本书中几个代码块的程序不需要许可。出售或分发奥莱利书籍中的示例代码需要许可。引用本书并引用示例代码来回答问题不需要许可。将本书中大量示例代码合并到产品文档中需要许可。

我们感谢，但通常不要求归因。归因通常包括标题、作者、出版商和 ISBN。例如：“*学习 MySQL*，第二版，作者 Vinicius M. Grippa 和 Sergey Kuzmichev（奥莱利）。版权所有 2021 年 Vinicius M. Grippa 和 Sergey Kuzmichev，978-1-492-08592-8。”

如果您觉得您对代码示例的使用超出了合理使用范围或以上所给的许可，请随时通过*permissions@oreilly.com*与我们联系。

# 奥莱利在线学习

###### 注意

40 多年来，[*奥莱利传媒*](http://oreilly.com)已为企业提供技术和商业培训、知识和见解，帮助企业取得成功。

我们独特的专家和创新者网络通过书籍、文章和我们的在线学习平台分享他们的知识和专长。奥莱利的在线学习平台让您随需学习，提供深入的学习路径、交互式编码环境以及来自奥莱利和 200 多家其他出版商的大量文本和视频内容。更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

请将有关本书的评论和问题发送给出版商：

+   奥莱利传媒公司

+   北加利福尼亚格拉文斯坦公路 1005 号

+   加利福尼亚州塞巴斯托波尔 95472

+   800-998-9938（美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为这本书制作了一个网页，列出勘误、示例和任何额外信息。您可以访问此页面[*https://oreil.ly/learn-mysql-2e*](https://oreil.ly/learn-mysql-2e)。

发送邮件至*bookquestions@oreilly.com*评论或提问关于本书的技术问题。

欲了解我们的书籍和课程的最新消息，请访问[*http://oreilly.com*](http://oreilly.com)。

在 Facebook 上找到我们：[*http://facebook.com/oreilly*](http://facebook.com/oreilly)

在 Twitter 上关注我们：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

在 YouTube 上观看我们：[*http://youtube.com/oreillymedia*](http://youtube.com/oreillymedia)

# 致谢

## 来自 Vinicius Grippa

感谢以下人员帮助改进本书：Corbin Collins、Charly Batista、Sami Ahlroos 和 Brett Holleman。没有他们，这本书就不会达到我们所追求的卓越水平。

感谢 MySQL 社区（特别是 Shlomi Noach、Giuseppe Maxia、Jeremy Cole 和 Brendan Gregg）以及[Planet MySQL](https://oreil.ly/MSFP1)、[Several Nines](https://oreil.ly/X1UZN)、[Percona Blog](https://oreil.ly/rsrAA)和[MySQL Entomologist](https://oreil.ly/yXkuy)上的所有博客作者，他们为我们提供了大量材料和优秀工具。

感谢所有在 Percona 提供写作本书支持的人，特别是 Bennie Grant、Carina Punzo 和 Marcelo Altmann，他们帮助我在职业和人生成长中得到提升。

感谢 O’Reilly 的工作人员出色地出版书籍和组织会议。

我要感谢我的父母 Divaldo 和 Regina，我的姐姐朱莉安娜，以及我的女朋友卡琳，在这个项目中给予我耐心和支持。特别感谢 Paulo Piffer，他给了我第一个能够从事我所热爱工作的机会。

最后，感谢谢尔盖·库兹米切夫，本书的共同作者。没有他的专业知识、奉献精神和辛勤工作，这本书将不可能完成。我很荣幸能与他共事，共同完成这个项目。

## 来自谢尔盖·库兹米切夫

我要感谢我的妻子 Kate，在这个艰难而有回报的项目的每一步都支持和帮助我。从思考是否承担写作这本书的责任，到写作期间的许多艰难时日，她一直在我身边。我们的第一个孩子在写作期间出生，但 Kate 仍然找到时间和力量继续激励和帮助我。

感谢我的父母、亲戚和朋友，多年来他们帮助我成长为一个人和专家。感谢你们在这个项目中对我的支持。

感谢 Percona 的出色团队在我撰写本书期间帮助我解决所有技术和非技术问题：Iwo Panowicz、Przemyslaw Malkowski、Sveta Smirnova 和 Marcos Albe。感谢 Stuart Bell 和 Percona 支持团队的每一个成员，在我们每一步的旅程中都提供了令人惊叹的帮助。

感谢 O’Reilly 的每一位领导者和帮助我们创建这个版本的人。感谢 Corbin Collins 在书的结构塑造和我们坚定的前行道路上的帮助。感谢 Rachel Head 在复制编辑阶段发现的众多问题，以及在我们的写作中发现 MySQL 技术细节问题。没有你们和 O’Reilly 的每一位成员，这本书就不会是一本书，而只是一堆松散相关的文字。

特别感谢我们的技术编辑 Sami Ahlroos、Brett Holleman 和 Charly Batista。他们在确保本书技术和非技术内容的最高质量方面发挥了关键作用。

感谢 MySQL 社区的每一位成员以开放、帮助和分享知识的方式。MySQL 的世界不是封闭的花园，而是向所有人开放的。我要提到 Valerii Kravchuk、Mark Callaghan、Dimitri Kravchuk 和 Jeremy Cole，他们通过他们的博客帮助我更好地理解 MySQL 的内部工作机制。

我要感谢本书第一版的作者们：Hugh E. Williams 和 Seyed M.M. Tahaghoghi。多亏了他们的工作，我们在一个坚实的基础上建立了这个项目。

最后但同样重要，我要感谢 Vinicius Grippa，他是一位出色的共同作者和同事。没有他，这本书就不会是现在这个样子。

我将本版献给我的儿子 Grigorii。
