# Druid查询之：Timeseries queries

Timeseries queries是根据指定的时间区间或按某一时间粒度进行的聚合查询

## 示例

一个简单的时间序列查询示例如下:
```json
{
  "queryType": "timeseries",
  "dataSource": "sample_datasource",
  "granularity": "day",
  "descending": "true",
  "filter": {
    "type": "and",
    "fields": [
      { "type": "selector", "dimension": "sample_dimension1", "value": "sample_value1" },
      { "type": "or",
        "fields": [
          { "type": "selector", "dimension": "sample_dimension2", "value": "sample_value2" },
          { "type": "selector", "dimension": "sample_dimension3", "value": "sample_value3" }
        ]
      }
    ]
  },
  "aggregations": [
    { "type": "longSum", "name": "sample_name1", "fieldName": "sample_fieldName1" },
    { "type": "doubleSum", "name": "sample_name2", "fieldName": "sample_fieldName2" }
  ],
  "postAggregations": [
    { "type": "arithmetic",
      "name": "sample_divide",
      "fn": "/",
      "fields": [
        { "type": "fieldAccess", "name": "postAgg__sample_name1", "fieldName": "sample_name1" },
        { "type": "fieldAccess", "name": "postAgg__sample_name2", "fieldName": "sample_name2" }
      ]
    }
  ],
  "intervals": [ "2012-01-01T00:00:00.000/2012-01-03T00:00:00.000" ]
}
```

Timeseries查询有7个主要的部分：

| 字段名        | 描述    |    是否必须 |
| --------   | :----------:   |:----------: |
| queryType        | Timeseries查询的这个字段应该始终是timeseries    | 是|
| dataSource        | 要查询数据源DataSource名字，类似于关系数据库中的表      | 是|
| descending        | 是否要降序排序。默认是false(升序)     | 否|
| intervals        | 查询时间范围,ISO-8601格式     | 是|
| granularity        | 查询的时间粒度     | 是|
| filter        | 过滤器     | 否|
| aggregations        | 聚合器     | 否|
| postAggregations        | 后聚合器     | 否|
| context        | 一些查询参数     | 否|

上面的查询例子将返回一个size为2的JSON数组，因为`granularity`为`day`,所以`intervals`时间范围内2012-01-01、2012-01-03将有这两部分结果集。每一部分结果集包括这些数据的`SUM(sample_fieldName1)`、`SUM(sample_fieldName2)`、`sample_fieldName1 / sample_fieldName2 `

输出结果如下面这种格式
```json
[
  {
    "timestamp": "2012-01-01T00:00:00.000Z",
    "result": { "sample_name1": <some_value>, "sample_name2": <some_value>, "sample_divide": <some_value> }
  },
  {
    "timestamp": "2012-01-02T00:00:00.000Z",
    "result": { "sample_name1": <some_value>, "sample_name2": <some_value>, "sample_divide": <some_value> }
  }
]
```

## 填充0
`Timeseries`查询会默认填充0在时间段数据为空时，如果设置`day`为时间粒度查询， 查询时间范围`interval`为 2012-01-01/2012-01-04，并且2012-01-02没有数据，返回结果将是：
```json
[
  {
    "timestamp": "2012-01-01T00:00:00.000Z",
    "result": { "sample_name1": <some_value> }
  },
  {
   "timestamp": "2012-01-02T00:00:00.000Z",
   "result": { "sample_name1": 0 }
  },
  {
    "timestamp": "2012-01-03T00:00:00.000Z",
    "result": { "sample_name1": <some_value> }
  }
]
```

在`interval`时间范围之外的部分不会被填充0，如果时间对应的`Segment`不存在也不会补0.

可以在`context`字段设置`skipEmptyBuckets`为true来关闭填充0
