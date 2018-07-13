# Druid 简介

[Druid](https://github.com/apache/incubator-druid) 是一个开源的亚秒级的实时数据存储分析系统，主要被用于 OLAP 场景下的海量数据查询，它提供了实时的数据写入、灵活的数据查询、以及快速的数据聚合。正因为 Druid 优秀的设计和表现，Druid 已在 2018-02-28 进入 [Apache 孵化器](http://incubator.apache.org/projects/druid.html)。

目前 Druid 生产集群已达到万亿级事件和 PB 级数据规模，已得到的反馈是：

- 每月 3 万亿以上的事件
- 每秒 3 百万以上的实时事件摄入
- 100PB 以上的原始数据
- 50 万亿以上的事件
- 每秒几千次的查询
- 成千上万个 CPU 核
