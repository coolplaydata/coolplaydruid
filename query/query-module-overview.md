# Druid查询之：查询概述

Druid的查询使用的是`HTTP REST`方式请求查询节点(`Broker`、`Historical`、or `Realtime`)。查询以JSON表示,并且每一个查询节点都会暴露相同的REST查询接口。对于正常的Druid操作，查询应该向`broker`请求，由broker节点转发至历史节点(`Historical`)或实时节点(`Realtime`)。可以像如下的方式发送到查询节点：
```
curl -X POST '<queryable_host>:<port>/druid/v2/?pretty' -H 'Content-Type:application/json' -d @<query_json_file>
```


Druid的原生查询语言是基于HTTP的JSON，虽然社区的许多成员已经用其他语言贡献了不同的客户端库来查询Druid，这些库可以在[官方文档](http://druid.io/libraries.html)中详细了解。

Druid的原生查询是低阶的，被设计为轻量级并且非常快速的完成。这意味着，对于更复杂的分析或构建更复杂的可视化数据，可能需要多个Druid查询。

## 查询内部过程
- [内部过程](./query-internal-procedure.md)



## 查询类型

基本的查询可分为三类(`聚合查询`、`元数据查询`、`搜索查询` )：

### 聚合查询(Aggregation Queries)
- [Timeseries](./query-timeseries.md)
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

## 我们应该使用哪种查询?
在可能的情况下，我们推荐使用`Timeseries`和`TopN`查询而不是`GroupBy`. `GroupBy`是Druid查询中最灵活的，但是效率较低。对于不需要对维度进行分组的场景，优先选择`Timeseries`,`Timeseries`的速度远远快于`groupBy`查询,
如果对单个维度进行分组和排序，`topN`查询比`groupby`更优化。

## 取消查询
查询可以使用唯一标识符显示的取消，如果查询标识符是在查询时设置的，或者是已知的，如`abc123`可以按如下的方式请求broker节点取消查询
```
curl -X DELETE "http://host:port/druid/v2/abc123"
```
## 查询错误
如果一个查询失败了，将会收到http状态码为500的响应和一个像下面这种结构的json对象
```json
{
  "error" : "Query timeout",
  "errorMessage" : "Timeout waiting for task.",
  "errorClass" : "java.util.concurrent.TimeoutException",
  "host" : "druid1.example.com:8083"
}
```
响应中的字段是：

  | 字段        | 描述    |
  | --------   | :----------:   |
  | error        | 错误码的定义(见下文)    |
  | errorMessage        | 一个自由格式的消息，有更多关于错误的信息。可能是null。      |
  | errorClass        | 导致这个错误的异常的类。可能是null     |
  | host        | 这个错误发生的主机。可能是null。     |

error字段可能的错误码有：

| 字段        | 描述    |
| --------   | :----------:   |
| Query timeout	        | 查询超时    |
| Query interrupted	        | 这个查询被中断了，可能是由于JVM关闭。      |
| Query cancelled	        | 查询被取消了     |
| Resource limit exceeded	        | 查询超过了配置的资源限制（例如，groupBy maxResults）     |
| Unknown exception	        | 其他一些异常发生。检查errorMessage和errorClass      |
