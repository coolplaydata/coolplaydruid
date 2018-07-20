# Indexing Service

Indexing Service 是高可用分布式的服务，一般运行索引相关的任务，比如创建或销毁 Druid segments。
它采用类似 master/slave 的架构：

![](./indexing-service-01.png)

如上， Indexing Service 有三个组件：
- peon： 运行单项任务的组件。
- Middle Manager： 管理多个 peon。
- Overlord： 接收客户端的任务提交， 管理着对 Middle Manager 的任务分配。

Middle Manager 和 Overlord 可以在同一节点或不同节点上，但 Middle Manager 和 Peon 总是在同一节点（后面看了源码就有更清晰的了解）。

架构图中只描述了基本运行原理，下面将从源码上介绍 Indexing Service 的主要实现，内容分为三部分：

- [任务提交](./indexing-service-submit-task.md)
- [任务的执行](/TODO)
- [任务的结束](/TODO)
