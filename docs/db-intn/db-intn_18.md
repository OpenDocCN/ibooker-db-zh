# 第二部分总结

性能和可伸缩性是任何数据库系统的重要特性。存储引擎和节点本地读写路径对系统的*性能*（即本地处理请求的速度）可能有更大的影响。与此同时，负责集群中通信的子系统通常对数据库系统的*可伸缩性*（即最大集群大小和容量）有更大的影响。然而，如果存储引擎不可伸缩且随着数据集的增长性能下降，它可能只能用于有限数量的用例。同时，在最快的存储引擎上放置一个慢的原子提交协议也不会产生良好的结果。

分布式、集群范围和节点本地进程是相互关联的，必须全面考虑。在设计数据库系统时，您必须考虑不同子系统如何匹配和协同工作。

第二部分开始讨论了分布式系统与单节点应用程序的不同之处，以及在这种环境中可能遇到的困难。

我们讨论了基本的分布式系统构建块，不同的一致性模型以及几个重要的分布式算法类别，其中一些可以用来实现这些一致性模型：

故障检测

准确高效地识别远程进程故障。

领导者选举

快速且可靠地选择一个单一进程作为临时协调器。

传播

使用点对点通信可靠地分发信息。

反熵

识别和修复节点之间状态分歧。

分布式事务

对多个分区执行一系列操作，保证原子性。

共识

在容忍进程故障的情况下，达成远程参与者之间的一致意见。

这些算法被许多数据库系统、消息队列、调度程序和其他重要基础设施软件所使用。利用本书的知识，您将能够更好地理解它们的工作原理，从而能够更好地决定使用哪些软件，并识别潜在问题。

## 进一步阅读

每章末尾，您可以找到与本章所述材料相关的资源。在这里，您将找到可以进一步学习的书籍，涵盖了本书提到的概念以及其他概念。这个列表并非完整，但这些资源包含了对数据库系统爱好者有重要和有用的信息，其中一些信息在本书中没有涵盖：

数据库系统

Bernstein, Philip A., Vassco Hadzilacos 和 Nathan Goodman. 1987\. *数据库系统中的并发控制与恢复*. 波士顿: Addison-Wesley Longman。

Korth, Henry F. 和 Abraham Silberschatz. 1986\. *数据库系统概念*. 纽约: 麦格劳-希尔。

Gray, Jim 和 Andreas Reuter. 1992\. *事务处理: 概念与技术* (第 1 版). 旧金山: 摩根-考夫曼。

Stonebraker, Michael 和 Joseph M. Hellerstein（编者）。 1998\. *数据库系统阅读*（第 3 版）。 旧金山：摩根考夫曼。

Weikum, Gerhard 和 Gottfried Vossen. 2001\. *事务信息系统：并发控制和恢复的理论、算法和实践*。 旧金山：摩根考夫曼。

Ramakrishnan, Raghu 和 Johannes Gehrke. 2002\. *数据库管理系统*（第 3 版）。 纽约：麦格劳希尔。

Garcia-Molina, Hector, Jeffrey D. Ullman, 和 Jennifer Widom. 2008\. *数据库系统：完整手册*（第 2 版）。 上塞德尔河，新泽西州：普林斯顿大学出版社。

Bernstein, Philip A. 和 Eric Newcomer. 2009\. *事务处理原理*（第 2 版）。 旧金山：摩根考夫曼。

Elmasri, Ramez 和 Shamkant Navathe. 2010\. *数据库系统基础*（第 6 版）。 波士顿：Addison-Wesley。

Lake, Peter 和 Paul Crowther. 2013\. *数据库简明指南：实用介绍*. 纽约：斯普林格出版社。

Härder, Theo, Caetano Sauer, Goetz Graefe, 和 Wey Guy. 2015\. *预写式日志的即时恢复*. 数据库光谱。

分布式系统

Lynch, Nancy A. *分布式算法*. 1996\. 旧金山：摩根考夫曼。

Attiya, Hagit, 和 Jennifer Welch. 2004\. *分布式计算：基础、模拟和高级主题*。 霍博肯，新泽西州：约翰威利&儿子。

Birman, Kenneth P. 2005\. *可靠分布式系统：技术、Web 服务和应用*。 柏林：斯普林格出版社。

Cachin, Christian, Rachid Guerraoui, 和 Lus Rodrigues. 2011\. *可靠和安全分布式编程简介*（第 2 版）。 纽约：斯普林格出版社。

Fokkink, Wan. 2013\. *分布式算法：直观方法*. 麻省理工学院出版社。

Ghosh, Sukumar. *分布式系统：算法方法*（第 2 版）。 Chapman & Hall/CRC。

Tanenbaum Andrew S. 和 Maarten van Steen. 2017\. *分布式系统：原理与范式*（第 3 版）。 波士顿：Pearson。

运营数据库

Beyer, Betsy, Chris Jones, Jennifer Petoff, 和 Niall Richard Murphy. 2016 *网站可靠性工程：谷歌如何运行生产系统*（第 1 版）。 波士顿：O’Reilly Media。

Blank-Edelman, David N. 2018\. *寻找 SRE*. 波士顿：O’Reilly Media。

Campbell, Laine 和 Charity Majors. 2017\. *数据库可靠性工程：设计和运营弹性数据库系统*（第 1 版）。 波士顿：O’Reilly Media。 +Sridharan, Cindy. 2018\. *分布式系统可观察性：构建强大系统指南*。 波士顿：O’Reilly Media。
