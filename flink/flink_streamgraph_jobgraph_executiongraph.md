
通过debug WindowWordCount，得到StreamGraph、JobGraph、ExecutionGraph的转换流程。

### Transformation

由各种SourceFunction、Map等Function构成的Transformation链路。

开发者视角操作的是各种DataStream，DataStream与Transformation对应关系如下图。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_datastream2transform.png)

### StreamGraph


### JobGraph

对StreamGraph进行了一些优化，如对一些operator形成Operator Chain，执行时可以放在一个slot中执行。

JobManager 接收JobGraph，并基于它生成ExecutionGraph，进行调度。

### ExecutionGraph

执行时的DAG视图，在JobManager中生成。由Map<JobVertexID, ExecutionJobVertex> tasks组成DAG。

ExecutionJobVertex的一次执行称为Execution。

主要工具类：

* StreamGraphGenerator: List<Transformation> -> StreamGraph。
* StreamingJobGraphGenerator: StreamGraph -> JobGraph。
* DefaultExecutionGraphBuilder: JobGraph -> ExecutionGraph。
