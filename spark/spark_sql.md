Spark SQL

## 简介

### SQL-on-Hadoop

目前大数据存储普遍基于Hadoop，面向hadoop的SQL查询技术及框架（统称SQL-on-Hadoop）为数据分析提供支撑。目前热门的SQL-on-Hadoop产品包括Hive、Impala、Presto、Spark SQL等。

SQL-on-Hadoop 三层架构：

- 最上层：应用层，为用户提供数据管理查询的接口。
- 中间层：分布式执行层，将SQL语句转换为对应的计算模型。
- 底层：数据存储层，对数据进行存储和管理。


### Spark SQL

主要执行流程为，对SQL进行解析后，生成逻辑执行计划，然后生成物理执行计划，进行执行。

生成逻辑计划分为三步：

- 生成未解析的逻辑算子树，仅仅是数据结构，不包含任何数据信息；
- 解析后的逻辑算子树，节点中绑定各种信息；
- 优化后的逻辑算子树，应用各种优化规则对一些低效的逻辑计划进行转换。

生成物理计划：

- 根据逻辑算子树，生成物理算子树列表；
- 从列表中按照一定的策略选择最优的物理算子树SparkPlan；
- 对物理算子树进行提交前的准备工作，如确保分区操作正确、算子树节点重用、执行代码生成等，得到PreparedSparkPlan。



## 主要内容

### 主要类

#### LogicalPlan

逻辑执行计划，与具体的平台无关。

#### SparkPlan

物理执行计划，与Spark平台关系紧密。

LogicalPlan和SparkPlan，都是TreeNode的实现。
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_sql_treenode.png)

### 编译器Parser

#### 词法和语法解析工具ANTLER

ANTLER、JavaCC等工具提供词法和语法解析功能，完成对SQL的解析，最终转换为底层逻辑执行。

spark中SqlBase.g4文件，为Antlr的配置文件，包含了spark sql的词法和语法规则。基于SqlBase.g4文件，通过maven编译生成了SqlBaseParser、SqlBaseLexer、SqlBaseBaseVisitor等class，SqlBaseLexer为词法分析器，SqlBaseParser为语法分析器。

SqlBaseBaseVisitor、SqlBaseBaseListener分别以访问者模式和监听者模式访问语法树，spark中AstBuilder继承SqlBaseBaseVisitor以访问者模式进行访问，每个visit方法表示对不同类型标签的行为。g4文件中“#”符号后面即为标签。

AbstractSqlParser中，利用SqlBaseLexer和SqlBaseParser得到SingleStatementContext后，调用AstBuilder.visitSingleStatement() 生成LogicalPlan。

Parser与AstBuilder：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_sql_parser.png)

#### AST

sql: select name from student where age > 18 order by id desc

生成的语法树：
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_sql_astpng)

AST对应的根节点为SingleStatementContext，通过不断遍历children，可以得到整个树的结构。上图即为将语法树展开之后的形态，树的叶子节点记录着具体的操作。


### 逻辑执行计划


生成LogicalPlan的主要流程如下：
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_logicalplan_process.png)


### 物理执行计划


### 执行计划生成及调用时机

测试代码：
```
private def runTestExample(spark: SparkSession): Unit = {
    val df = spark.read.json("examples/src/main/resources/people.json")
    
    df.createOrReplaceTempView("people")
    val sqlDF = spark.sql("SELECT * FROM people")
    sqlDF.show()
}
```

上面例子中spark为SparkSession对象，调用spark.sql方法时，会生成对应code的LogicalPlan，存入到返回的DataSet中。

调用DataSet.show方法时，生成物理执行计划SparkPlan，然后调用SparkPlan.executeTake方法，提交job执行。

在val df = spark.read.json("examples/src/main/resources/people.json") 这里，会提交一个spark job，读取所有的json数据。这里的目的是，读取数据获取字段名称并推测字段类型。逻辑见JsonInferSchema类。


## 参考