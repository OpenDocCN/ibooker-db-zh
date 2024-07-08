# 前言

MySQL 文献中存在着基本 MySQL 知识和高级 MySQL 性能之间的差距。前者有几本书，后者只有一本：《*高性能 MySQL*》，第四版，作者 Silvia Botros 和 Jeremy Tinley（O'Reilly 出版社）。这是弥合这一差距的第一本书。

这个差距的存在是因为 MySQL 非常复杂，而不解决这个复杂性就很难教授性能问题——这就像房间里的大象一样显而易见。但是使用 MySQL 而不是管理 MySQL 的工程师不应该需要成为 MySQL 专家才能实现出色的 MySQL 性能。为了弥合这个差距，这本书效率非凡——不要关注大象；它很友好。

*高效*的 MySQL 性能意味着*专注*：只学习和应用那些直接影响出色 MySQL 性能的最佳实践和技术。专注显著减少了 MySQL 复杂性的范围，并允许我向你展示通过 MySQL 性能广阔而复杂的领域的更简单和更快速的路径。这个旅程从第一章的第一句开始，“性能即查询响应时间”。从那里开始，我们迅速探讨索引、数据、访问模式等等。

在一个从一到五的评级中——其中一表示适合任何人，五表示深入探讨的专家级——本书的评级范围从三到四：深入，但远非底线。我假设你是一个有经验的工程师，具有关系数据库（MySQL 或其他）的基本知识和经验，所以我不解释 SQL 或数据库基础知识。我假设你是一位经验丰富的程序员，负责一个或多个使用 MySQL 的应用程序，所以我持续提到*应用程序*，并相信你了解*你的应用程序*的细节。我还假设你对计算机有一定了解，所以我可以自由地讨论硬件、软件、网络等等。

由于本书侧重于使用 MySQL 的工程师的 MySQL 性能，而不是管理 MySQL，因此在必要时会提到几个 MySQL 配置，但不进行解释。如果需要帮助配置 MySQL，请问问你工作的 DBA。如果你没有 DBA，可以雇佣一个 MySQL 顾问——有很多价格合理的顾问可供选择。你也可以通过阅读[*MySQL 参考手册*](https://oreil.ly/Y1W2r)来学习。MySQL 手册非常好，并且专家们经常使用它，所以你不用担心。

# 本书使用的约定

本书使用以下排版约定：

*斜体*

表示新术语、URL、电子邮件地址、文件名和文件扩展名。

`固定宽度`

用于程序清单，以及段落内引用程序元素，如变量或函数名、数据库、数据类型、环境变量、语句和关键字。

**`固定宽度粗体`**

显示用户应该直接输入的命令或其他文本。

*`固定宽度斜体`*

显示应由用户提供的值或由上下文确定的值替换的文本。

###### 提示

这个元素表示提示或建议。

###### 注意

这个元素表示一般注意事项。

###### 警告

这个元素表示警告或注意事项。

# 使用代码示例

附加材料（代码示例、练习等）可从[*https://github.com/efficient-mysql-performance*](https://github.com/efficient-mysql-performance)下载。

如果您有技术问题或在使用代码示例时遇到问题，请发送电子邮件至*bookquestions@oreilly.com*。

本书旨在帮助您完成工作。通常情况下，如果本书提供示例代码，您可以在您的程序和文档中使用它。除非您复制了代码的大部分，否则无需征得我们的许可。例如，编写一个使用本书多个代码片段的程序并不需要许可。销售或分发 O’Reilly 图书示例代码需要许可。引用本书并引用示例代码回答问题不需要许可。将本书大量示例代码整合到产品文档中需要许可。

我们感谢，但通常不需要署名。署名通常包括书名、作者、出版商和 ISBN。例如：“*Efficient MySQL Performance* by Daniel Nichter (O’Reilly). Copyright 2022 Daniel Nichter, 978-1-098-10509-9.”

如果您认为您使用的代码示例超出了合理使用范围或上述授权，请随时与我们联系至*permissions@oreilly.com*。

# O’Reilly Online Learning

###### 注意

超过 40 年来，[*O’Reilly Media*](http://oreilly.com)提供技术和商业培训，知识和见解，帮助企业成功。

我们独特的专家和创新者网络通过书籍、文章以及我们的在线学习平台分享他们的知识和专业知识。O’Reilly 的在线学习平台为您提供按需访问的实时培训课程、深入学习路径、交互式编码环境以及来自 O’Reilly 和其他 200 多家出版商的广泛文本和视频。更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

请将有关本书的评论和问题发送至出版商：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   Sebastopol, CA 95472

+   800-998-9938（美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为本书设置了一个网页，列出勘误、示例和任何额外信息。您可以访问[*https://oreil.ly/efficient-mysql-performance*](https://oreil.ly/efficient-mysql-performance)查看此页面。

电子邮件*bookquestions@oreilly.com*用于评论或就本书提出技术问题。

了解更多关于我们的书籍和课程的新闻和信息，请访问[*http://oreilly.com*](http://oreilly.com)。

在 Twitter 上关注我们：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

在 YouTube 上关注我们：[*http://youtube.com/oreillymedia*](http://youtube.com/oreillymedia)。

# 致谢

感谢那些审阅本书的 MySQL 专家：Vadim Tkachenko、Frédéric Descamps 和 Fernando Ipar。感谢审阅本书部分内容的 MySQL 专家：Marcos Albe、Jean-François Gagné和 Kenny Gryp。感谢多年来帮助、教导我并为我提供机会的许多其他 MySQL 专家：Peter Zaitsev、Baron Schwartz、Ryan Lowe、Bill Karwin、Emily Slocombe、Morgan Tocker、Shlomi Noach、Jeremy Cole、Laurynas Biveinis、Mark Callaghan、Domas Mituzas、Ronald Bradford、Yves Trudeau、Sveta Smirnova、Alexey Kopytov、Jay Pipes、Stewart Smith、Aleksandr Kuzminsky、Alexander Rubin、Roman Vynar，还有再次提到的 Vadim Tkachenko。

感谢 O’Reilly 和我的编辑们：Corbin Collins、Katherine Tozer、Andy Kwan，以及所有在幕后工作的人员。

也感谢我的妻子 Moon，在我写作这本书的耗时过程中一直支持我。
