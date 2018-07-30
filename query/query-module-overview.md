# Druid查询之：查询概述

Druid的查询使用的是`HTTP REST`方式请求查询节点(`Broker`、`Historical`、or `Realtime`)。查询以JSON表示,并且每一个查询节点都会暴露相同的REST查询接口。对于正常的Druid操作，查询应该向`broker`请求，由broker节点转发至Historical或Realtime节点。可以像如下的方式发送到查询节点：
```
curl -X POST '<queryable_host>:<port>/druid/v2/?pretty' -H 'Content-Type:application/json' -d @<query_json_file>
```

Druid的原生查询语言是基于HTTP的JSON，虽然社区的许多成员已经用其他语言贡献了不同的客户端库来查询Druid，这些库可以在[官方文档](http://druid.io/libraries.html)中详细了解。

Druid的原生查询是低阶的，被设计为轻量级并且非常快速的完成。这意味着，对于更复杂的分析或构建更复杂的可视化数据，可能需要多个Druid查询。

## 查询内部过程
- [内部过程](./query-internal-procedure.md)


## 查询类型

基本的查询可分为三类(``聚合查询``、``元数据查询``、``搜索查询``)：

### 聚合查询(Aggregation Queries)
- [Timeseries](/TODO)
- [TopN](/TODO)
- [GroupBy](/TODO)

### 元数据查询(Metadata Queries)
- [Time Boundary](/TODO)
- [Segment Metadata](/TODO)
- [Datasource Metadata](/TODO)

### 搜索查询(Search Queries)
- [Search](/TODO)

### 其他查询(Other Queries)
- [Select](/TODO)
- [Scan](/TODO)

## 查询组件
- [Datasources](/TODO)
- [Filters](/TODO)
- [Aggregations](/TODO)
- [Post Aggregations](/TODO)
- [Granularities](/TODO)
- [DimensionSpecs](/TODO)
- [Context](/TODO)

## SQL查询
- [SQL](/TODO)
