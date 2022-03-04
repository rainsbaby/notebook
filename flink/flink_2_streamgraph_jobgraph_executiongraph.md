## 简介

通过debug 例子程序WindowWordCount，得到StreamGraph、JobGraph、ExecutionGraph的转换流程。


## 核心类

源码版本：Flink 1.14。

### StreamExecutionEnvironment

其中包含的List<Transformation<?>>，表示用户视角的各种转换过滤等逻辑。



### Transformation

由各种SourceFunction、Map等Function构成的Transformation链路。

开发者视角操作的是各种DataStream，DataStream与Transformation对应关系如下图。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_datastream2transform.png)


Transformation 接口继承关系：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_transformation_uml.png)



### StreamGraph

StreamGraphGenerator基于StreamExecutionEnvironment中Transformations来生成StreamGraph。

StreamGraph中Map<Integer, StreamNode> streamNodes，表示节点集合。

StreamNode中主要包括：
```
// 节点operator信息，不同operator对应不同的factory
private StreamOperatorFactory<?> operatorFactory;

// 上游和下游
private List<StreamEdge> inEdges = new ArrayList<StreamEdge>();
private List<StreamEdge> outEdges = new ArrayList<StreamEdge>();

//节点中可执行部分
private final Class<? extends TaskInvokable> jobVertexClass;
```

#### StreamOperatorFactory

StreamOperator的工厂类。

#### StreamOperator

Stream operator的基类。

实现类可以实现OneInputStreamOperator或TwoInputStreamOperator接口。

抽象子类AbstractUdfStreamOperator，基于userFunction提供类自定义处理函数的实现。


![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_streamoperator_uml.png)




### JobGraph

由StreamingJobGraphGenerator基于StreamGraph生成。StreamGraph和JobGraph都在Client端生成。

对StreamGraph进行了一些优化，如对一些operator形成Operator Chain，执行时可以放在一个slot中执行。

提交job时，JobManager 接收JobGraph，并基于它生成ExecutionGraph，进行调度。

JobGraph中利用Map<JobVertexID, JobVertex> taskVertices构建DAG结构。

JobVertex主要包括：

```
/**
* 节点包含的所有operator
* The IDs of all operators contained in this vertex.
*
* <p>The ID pairs are stored depth-first post-order; for the forking chain below the ID's would
* be stored as [D, E, B, C, A].
*
* <pre>
*  A - B - D
*   \    \
*    C    E
* </pre>
*
* <p>This is the same order that operators are stored in the {@code StreamTask}.
*/
private final List<OperatorIDPair> operatorIDs;

//产生的数据集
private final ArrayList<IntermediateDataSet> results = new ArrayList<>();

//输入
private final ArrayList<JobEdge> inputs = new ArrayList<>();

//运行时operator的管理者。
private final ArrayList<SerializedValue<OperatorCoordinator.Provider>> operatorCoordinators = new ArrayList<>();
    
//可触发的class名
private String invokableClassName;
```

### ExecutionGraph

执行时的DAG视图，在JobManager中由生成。由Map<JobVertexID, ExecutionJobVertex> tasks组成DAG。

JobMaster创建时，创建Scheduler，由Scheduler通过JobGraph转换得到ExecutionGraph，具体逻辑可见SchedulerBase类。

ExecutionGraph主要构成：
```
// 组成DAG
private final Map<JobVertexID, ExecutionJobVertex> tasks;

// 中间结果
private final Map<IntermediateDataSetID, IntermediateResult> intermediateResults;

// 当前执行
private final Map<ExecutionAttemptID, Execution> currentExecutions;

// job状态监听者
private final List<JobStatusListener> jobStatusListeners;
```

ExecutionJobVertex,对应于JobGraph中的JobVertex。

ExecutionJobVertex表示有1/n个并行度的operation，其中operation的一个并发对应一个ExecutionVertex。ExecutionJobVertex的一次执行称为Execution。


主要工具类：

* StreamGraphGenerator: List<Transformation> -> StreamGraph。
* StreamingJobGraphGenerator: StreamGraph -> JobGraph。
* DefaultExecutionGraphBuilder: JobGraph -> ExecutionGraph。


### TaskInvokable
表示Task中可执行部分

TaskInvokable实现类有OneInputStreamTask/TwoInputStreamTask等，主要基类为StreamTask。


### ResultPartition

由ResultPartitionFactory类，根据ResultPartitionType类型创建不同的ResultPartition。

### InputGate
InputGate消费Intermediate Result中的一个或多个partition。

封装了InputChannel，内部包含多个InputChannel。每个intermediate result partition有一个对应的input channel。

### InputChannel
一个InputChannel消费一个ResultSubpartitionView。

子类RemoteInputChannel是请求远端partition queue的InputChannel。利用一个队列管理接收到的buffer。由网络I/O线程负责入队，由接收到task线程负责消费。


### ShuffleEnvironment

主要实现类为NettyShuffleEnvironment。包含跟踪所有intermediate result和shuffle 数据变化的数据。

NettyShuffleEnvironment主要成员如下。

```
// 相关的NetworkBufferPool
private final NetworkBufferPool networkBufferPool;

// 管理input channel的物理连接
private final ConnectionManager connectionManager;

// 跟踪TaskManager当前生成/消费的所有partition
private final ResultPartitionManager resultPartitionManager;

private final FileChannelManager fileChannelManager;

// 管理InputGate
private final Map<InputGateID, SingleInputGate> inputGatesById;

// 负责创建ResultPartition 
private final ResultPartitionFactory resultPartitionFactory;

// 负责创建SingleInputGate及内部的InputChannel
private final SingleInputGateFactory singleInputGateFactory;
```




## 主要流程

### 从ExecutionGraph到具体执行

JobMaster接收到启动任务的命令后，才开始调度ExecutionGraph执行任务。

任务执行具体逻辑可见DefaultScheduler和TaskExecutor类。主要包括以下步骤：

1. DefaultScheduler端，将每个Execution状态变为ExecutionState.SCHEDULED；
2. 分配slot资源；
3. slots准备完成后，开始deploy所有ExecutionVertex
	* ExecutionVertex.deploy 部署每个ExecutionVertex
	* ExecutionVertex中维护当前Execution，部署当前Execution
	* Execution调用TaskManagerGateway.submitTask()，提交Execution
4. TaskExecutor端，接收submitTask请求
	* 标记slot为active状态
	* 从BlobCache中load Job和Task信息
	* 创建Task，并交给TaskSlotTable管理
	* 启动task
5. Task端，执行run方法
	* 首先获取TaskInvokable，即要触发执行的operator类，加载类并实例化
	* 通知各方task将进入running状态
	* 执行TaskInvokable.invoke，即Operator的主要逻辑。
6. StreamTask端，执行invoke方法
	* runMailboxLoop()，Operator主要逻辑执行，无限循环处理输入的流数据
7. MailboxProcessor.runMailboxLoop()



## 总结

## 参考