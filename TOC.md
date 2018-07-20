* [Druid 是什么](./intro/what-is-druid.md)
  * [历史](./intro/druid-history.md)
  * 特点
  * Druid 基本概念
  * 应用场景和案例

* [Druid 架构](./arch/druid-arch.md)
  * Realtime
    * [Indexing Service](./arch/indexing-service/indexing-service.md)
      * [任务提交](./arch/indexing-service/indexing-service-submit-task.md)
  * Coordinator
  * Historical
  * Broker

* 数据摄取 (Data Ingestion )
  * 批量
  * 实时
    * Kafka Indexing Service

* 查询
  * 查询的内部过程
  * Timeseries
  * TopN
  * GroupBy
  * Time Boundary
  * Search
  * Select
  * Segment Metadata
  * DataSource Metadata
  * 查询组件
    * Datasources
    * Filters
    * Aggregations
    * Post Aggregations
    * Granularities
    * DimensionSpecs
    * Context

* Druid 数据结构和算法
  * Druid 数据文件
  * Segment
  * Bitmap
  * Inverted Index
  * HyperLogLog
  * LSM-tree

* 监控
  * 查询相关
  * ingest 相关
  * Cache 相关
  * segment 相关
  * Historical 相关
  * Coordinator 相关
  * JVM 相关

* 优化

* TODO 欢迎继续补充
