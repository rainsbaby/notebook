## 简介

## 主要内容

### Sql Client使用
Flink sql 使用

参考 [](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/table/sqlclient/)

使用结果：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_sql_client_usage.png)

### 主要构成

Apache Flink具有两个关系型API--表API和SQL--用于统一的流和批处理。

Flink的SQL支持是基于Apache Calcite的。Flink SQL 抽象出了 Planner 接口和 Executor 接口，用于支持从sql转换得到Flink Runtime可执行的逻辑执行计划。

#### Calcite

> Apache Calcite 是一个动态数据的管理框架，可以用来构建数据库系统的**语法解析**模块。
> 不包含数据存储、数据处理等功能。
> 可以通过编写 Adaptor 来扩展功能，以支持不同的数据处理平台。
> Flink SQL 使用并对其扩展以支持 SQL 语句的解析和验证。

Calcite 主要功能:

SQL解析：通过JavaCC将SQL解析成未经校验的AST语法树

SQL校验：校验分两部分，一种为无状态的校验，即验证SQL语句是否符合规范；一种为有状态的即通过与元数据结合验证SQL中的Schema、Field、Function是否存在。

SQL查询优化：对上个步骤的输出（RelNode）进行优化，得到优化后的物理执行计划

SQL生成：将物理执行计划生成为在特定平台/引擎的可执行程序，如生成符合Mysql or Oracle等不同平台规则的SQL查询语句等

数据连接与执行：通过各个执行平台执行查询，得到输出结果。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_sql_queryexecution.png)

Calcite主要构成：

* Catalog – 定义元数据和命名空间，包含 Schema（库）、Table（表）、RelDataType（类型信息）
* SQL Parser – 将用户编写的 SQL 语句转为 SqlNode 构成的抽象语法树（AST）
    通过 JavaCC 模版生成 LL(k) 语法分析器，主模版是 Parser.jj；可对其进行扩展负责处理各个 Token，逐步生成一棵 SqlNode 组成的 AST
* SQL Validator – 使用 Catalog 中的元数据检验上述 SqlNode AST 并生成 RelNode 组成的 AST
* Query Optimizer – 将 RelNode AST 转为逻辑计划，然后优化它，最终转为实际执行方案。以下是一些常见的优化规则（Rules）：
	移除未使用的字段
	合并多个投影（projection）列表
	使用 JOIN 来代替子查询
	对 JOIN 列表重排序
	下推（push down）投影项
	下推过滤条件

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_sql_calcite_architecture.png)

#### Planner 接口和 Executor 接口

Planner：负责 1.sql解析 2.relational planner，计划、优化、转换得到可执行的逻辑计划List<Transformation>，交给Executor执行。

Executor：执行由Planner生成的逻辑算子。


## 核心类


## 参考

[Calcite介绍](https://www.slideshare.net/JordanHalterman/introduction-to-apache-calcite)