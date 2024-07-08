# 序言

# 本书的组织结构

本书分为六个部分，涵盖了开发、管理和部署信息。

## MongoDB 入门

在第一章中，我们提供了关于 MongoDB 的背景信息：它的创建原因、它试图实现的目标以及为什么您可能选择在项目中使用它。我们在第二章中进行了更详细的介绍，这一章介绍了 MongoDB 的核心概念和词汇。第二章还为您提供了开始使用数据库和 shell 的第一步。接下来的两章介绍了开发人员需要了解的 MongoDB 基础知识。在第三章中，我们描述了如何执行基本的写操作，包括如何在不同的安全性和速度级别下执行这些操作。第四章解释了如何查找文档并创建复杂的查询。本章还涵盖了如何迭代结果以及限制、跳过和排序结果的选项。

## 使用 MongoDB 进行开发

第五章介绍了索引的概念以及如何为您的 MongoDB 集合创建索引。第六章解释了如何使用几种特殊类型的索引和集合。第七章涵盖了使用 MongoDB 聚合数据的多种技术，包括计数、查找不同的值、文档分组、聚合框架以及将这些结果写入集合的方法。第八章介绍了事务：它们是什么，如何最好地为您的应用程序使用它们，以及如何进行调优。最后，本节以一章讲述应用程序设计结束：第九章介绍了撰写与 MongoDB 配合良好的应用程序的技巧。

## 复制

复制部分从第十章开始，提供了在本地设置副本集的快速方法，并涵盖了许多可用的配置选项。第十一章然后涵盖了与复制相关的各种概念。第十二章展示了复制如何与您的应用程序交互，第十三章涵盖了运行副本集的管理方面。

## 分片

分片部分从第十四章开始进行快速本地设置。第十五章然后概述了集群组件及其设置方式。第十六章提供了选择各种应用程序的分片键的建议。最后，第十七章涵盖了管理分片集群的内容。

## 应用程序管理

接下来的两章从您的应用程序的角度讨论了 MongoDB 管理的许多方面。第 18 章讨论了如何审视 MongoDB 正在做什么。第 19 章涵盖了 MongoDb 的安全性以及如何为您的部署配置身份验证和授权。第 20 章解释了 MongoDB 如何持久存储数据。

## 服务器管理

最后一部分集中在服务器管理上。第 21 章讨论了启动和停止 MongoDB 时的常见选项。第 22 章讨论了在监控时要查找和如何读取统计信息。第 23 章描述了如何为每种部署类型进行备份和恢复。最后，第 24 章讨论了部署 MongoDB 时需要牢记的一些系统设置。

## 附录

附录 A 解释了 MongoDB 的版本方案及如何在 Windows、OS X 和 Linux 上安装。附录 B 详细介绍了 MongoDB 的内部工作原理：其存储引擎、数据格式和协议。

# 本书使用的约定

本书使用以下排版约定：

斜体

表示新术语、网址、电子邮件地址、集合名称、数据库名称、文件名和文件扩展名。

`Constant width`

用于程序清单，以及段落中引用程序元素，如变量或函数名称、命令行实用程序、环境变量、语句和关键字。

**`Constant width bold`**

显示用户应逐字输入的命令或其他文本。

*`Constant width italic`*

显示应该用用户提供的值或由上下文确定的值替换的文本。

###### 提示

这个元素表示一个提示或建议。

###### 注意

这个元素表示一般的注释。

###### 警告

这个元素指示警告或注意事项。

# 使用代码示例

补充材料（代码示例、练习等）可在[*https://github.com/mongodb-the-definitive-guide-3e/mongodb-the-definitive-guide-3e*](https://github.com/mongodb-the-definitive-guide-3e/mongodb-the-definitive-guide-3e)下载。

如果您有技术问题或在使用代码示例时遇到问题，请发送电子邮件至*bookquestions@oreilly.com*。

本书旨在帮助您完成工作。通常情况下，如果本书提供了示例代码，您可以在自己的程序和文档中使用它。除非您复制了大部分代码，否则无需征得我们的许可。例如，编写一个使用本书多个代码片段的程序并不需要许可。出售或分发 O’Reilly 书籍的示例代码确实需要许可。引用本书回答问题并引用示例代码不需要许可。将本书大量示例代码整合到产品文档中需要许可。

我们感谢您的使用，但通常不要求您提及。署名通常包括标题、作者、出版商和 ISBN。例如：“*MongoDB: The Definitive Guide*, Third Edition by Shannon Bradshaw, Eoin Brazil, and Kristina Chodorow (O’Reilly)。版权所有 2020 年 Shannon Bradshaw 和 Eoin Brazil，978-1-491-95446-1。”

如果您认为您使用的示例代码超出了合理使用范围或上述许可，请随时通过 *permissions@oreilly.com* 联系我们。

# [O’Reilly Online Learning](https://oreil.ly/mongoDB_TDG_3e)

###### 注意

40 多年来，[O’Reilly Media](http://oreilly.com)一直为公司提供技术和业务培训、知识和见解，帮助其取得成功。

我们独特的专家和创新者网络通过书籍、文章、会议和我们的在线学习平台分享他们的知识和专长。O’Reilly 的在线学习平台让您随时访问现场培训课程、深入学习路径、交互式编码环境以及来自 O’Reilly 和 200 多家其他出版商的大量文本和视频。有关更多信息，请访问 [*http://oreilly.com*](http://www.oreilly.com)。

# 如何联系我们

有关本书的评论和问题，请联系出版商：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   Sebastopol, CA 95472

+   800-998-9938（美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们有本书的网页，上面列出了勘误、示例和任何其他信息。您可以访问此页面：[*https://oreil.ly/mongoDB_TDG_3e*](https://oreil.ly/mongoDB_TDG_3e)。

发送邮件至 *bookquestions@oreilly.com* 提出对本书的评论或技术问题。

有关我们的书籍、课程、会议和新闻的更多信息，请访问我们的网站：[*http://www.oreilly.com*](http://www.oreilly.com)。

在 Facebook 上找到我们：[*http://facebook.com/oreilly*](http://facebook.com/oreilly)

在 Twitter 上关注我们：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

在 YouTube 上观看我们：[*http://www.youtube.com/oreillymedia*](http://www.youtube.com/oreillymedia)
