Spark SQL

## 简介

### SQL-on-Hadoop

目前大数据存储普遍基于Hadoop，面向hadoop的SQL查询技术及框架（统称SQL-on-Hadoop）为数据分析提供支撑。目前热门的SQL-on-Hadoop产品包括Hive、Impala、Presto、Spark SQL等。

SQL-on-Hadoop 三层架构：

- 最上层：应用层，为用户提供数据管理查询的接口。
- 中间层：分布式执行层，将SQL语句转换为对应的计算模型。
- 底层：数据存储层，对数据进行存储和管理。


