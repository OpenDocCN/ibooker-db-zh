# 附录 B. Kubernetes 上的 MySQL

如果您在过去五年中一直在科技领域工作，很可能已经听说过 Kubernetes，与运行 Kubernetes 的团队合作，或者看过很多关于 Kubernetes 的会议演讲或阅读了很多博客文章。如果您的组织运行自己的 Kubernetes 集群，那么在某个时候，您会被问及是否在其中运行 MySQL 也是一个好主意。从表面上看，这似乎是一个合理的选择。管理许多 Kubernetes 集群是一个复杂的任务，通常需要专门的人力资源，因此您的组织希望利用这种专业知识来处理不仅仅是无状态工作负载。但是，有很多理由可以探索在 Kubernetes 上运行 MySQL，也有一些不太好的理由。让我们在这里揭开一些关于在 Kubernetes 上运行 MySQL 的恐惧、不确定性和怀疑。

# 使用 Kubernetes 配置资源

在 Kubernetes 达到技术高峰之前，许多公司要么构建了完全定制的技术堆栈来配置和管理虚拟机和裸金属服务器，要么将开源项目粘合在一起，完成资源生命周期的较小部分。然后 Kubernetes 出现了，作为一个更完整的生态系统，用于管理计算和存储资源，将其作为统一的配置堆栈的前景变得越来越吸引人。然而，像 MySQL 这样的有状态负载仍然落后，因为普遍看法是“你不能在容器上运行数据库。”

## 仔细界定您的目标

要记住的重要事情是“我们想在这里获得什么具体价值？”Kubernetes 对于无状态负载非常强大，因为它带来了计算资源的弹性和效率。然而，在查看统一的配置堆栈时，将胜利范围缩小到“我们只想使用 Kubernetes 来配置和配置数据库资源系统。”这意味着您需要事先明确，将使用 Kubernetes 配置的数据库工作负载将与无状态工作��载分开管理，需要不同的操作员技能集，并且将以不同方式处理容器故障。

## 选择您的控制平面

现在有各种 MySQL 操作员，但最佳选择将主要取决于您决定的 Kubernetes 管理 MySQL 的范围。您是否需要一个可以完成所有工作的操作员：配置、故障转移和管理连接到数据库？还是您只需将 Kubernetes 用作配置堆栈，并在服务后使用其他方法来管理数据库？早早决定您对控制平面的期望，因为这将驱动许多更精细的可操作性细节。

## 更详细的细节

一旦您决定开始使用 Kubernetes 配置 MySQL 资源，您需要在组织中就适合此解决方案的数据规模达成一致。请记住，这现在是一个新的操作模型，用于运行关系型数据库，而在这条少有人走的道路上，随着规模的扩大，一切都变得更加复杂。以下是一些重要事项，您在与 Kubernetes 工程团队合作时（希望您有专门的团队负责此事）需要考虑如何支持有状态工作负载：

+   单个数据库实例支持的最大数据集大小是多少？

+   您是否将卷挂载到容器中，并将容器恢复与数据挂载分开管理？还是数据将成为容器的一部分？

+   将支持的最大查询吞吐量是多少？您将如何管理资源？

+   您将如何确保运行数据库工作负载的 Kubernetes 节点专用于此，而不与无状态、更具弹性的工作负载共享？

+   您将使用什么控制平面来运行数据库实例？它是否是 Kubernetes 本地的？

+   备份将如何工作？恢复过程是什么？

+   您将如何控制并安全地推出配置更改和 MySQL 升级？

+   您将如何升级 Kubernetes 集群本身而不会造成中断？

与您的合作伙伴 Kubernetes 工程团队就这个解决方案的工作方式达成一致意见，将有助于为希望使用该解决方案的功能团队建立完善的 SLO，并在正确传达解决方案解决了什么以及团队仍需自行解决什么方面。

我们在运行 MySQL on Kubernetes 时的建议是投资于学习一个已经在 Kubernetes 生态系统中经过验证和证明的控制平面，比如 Vitess。但在尝试运行之前，也要先爬行。在您的组织中，MySQL 不应该是在 Kubernetes 上运行工作负载的第一个实验对象。始终要证明可行性，并与无状态工作负载团队一起学习尖锐的边缘，然后再尝试运行像 MySQL 这样更复杂的用例。在确定最佳的初始采用用例时，从小数据集（仅在磁盘上几个千兆字节的数据库）和较少关键数据集开始，以使您的团队、Kubernetes 团队和功能团队熟悉在 Kubernetes 上运行有状态工作负载的新操作模型，同时对业务风险较小。

在 Kubernetes 上运行有状态工作负载已经成熟了几年，并且随着那些投入大量工程时间使其更具可行性的公司的重要贡献，它仍处于初期阶段，与直接在 VM 上运行相比，您会发现缓慢和谨慎的采用方法才是长远回报的关键。特别要考虑 MySQL 在 Kubernetes 上的故障模式是什么样的，并问自己：如果一切都出错了，我该如何重新组合？我会丢失数据吗？确保您有答案。

# 总结

Kubernetes 是当前技术领域中增长最快的基础设施平台之一，而且理由充分。它所带来的工程速度和由云原生基金会支持的丰富生态系统使其成为公司吸引投资的有吸引力的选择。但您应该通过风险和回报的视角来考虑像在 Kubernetes 上运行 MySQL 这样的决策，以及对您的团队和公司的影响。确保您对组织的 Kubernetes 之旅中像数据存储这样的有状态服务的位置有共同的理解。想要利用 Kubernetes 对所有工作负载的现有投资是可以理解的，但这需要与数据存储层的稳定性需求进行良好的平衡。

¹关于在 Kubernetes 上运行数据库工作负载的出色“实战”会议演讲，我们推荐由 Alice Goldfuss 主持的[“容器操作员手册”](https://oreil.ly/TVD6c)主题演讲。