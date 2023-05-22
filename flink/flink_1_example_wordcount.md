# 简介

本篇内容是我看Flink源码的第一篇记录。Flink的内容很多，刚开始看时可能感觉茫然无头绪。

好在Flink提供了大量的例子，也有大数据组件中经典的WordCount示例。因此，我从example目录下WordCount开始，一步步debug，终于理清了集群启动、job执行的基本流程。

# 核心类

Flink版本 : 1.14 。

#### StreamExecutionEnvironment

Streaming项目执行的上下文。

提供方法用于控制job执行（设置并行度/容错/checkpoint参数），与外部进行交互（数据读写）。

#### DataStream

表示同一种类型的元素组成的流。可以由map等方法转换成其他类型等DataStream

#### StreamGraph

[ ] StreamGraph与JobGraph的区别


#### JobGraph


#### ExecutionGraph

ExecutionGraph用于控制data flow的分布式执行，粒度细到每个并行的task、每个中间流及其交互。

由以下元素构成：
1. ExecutionJobVertex，表示JobGraph里的一个vertex。它聚合里该vertex对应的所有并行执行的执行状态。由JobVertexID标识。
2. ExecutionVertex，表示vertex的某个具体的并行subtask。用ExecutionJobVertex+subtask的index标识。
3. Execution，表示ExecutionVertex的一次执行。ExecutionVertex失败后重试、重复执行时，对应不同的Execution。
用ExecutionAttemptID标识。JobManager与TaskManager间，关于task部署及更新的交互，都基于ExecutionAttemptID。

#### MiniCluster

本地执行Flink job时启动的集群。

#### TaskExecutor

TaskManager的实现类，其中可以执行多个Task。

#### ResourceManagerService

负责管理ResourceManager。

#### Dispatcher

负责job的提交、存储、JobManager生成及在JobManager的master宕机时进行恢复。

#### BlobServer

负责监听request，并分配给某个线程处理。
主要负责jar包的管理。

#### JobMaster

JobManager的实现。负责一个JobGraph的执行。

#### LeaderElectionService

使一个contender可以开始参与选举的接口。

#### HighAvailabilityServices

用于为ResourceManager、JobManager等提供高可用服务。
提供高可用的存储、注册，以及分布式累加器、leader选举等功能。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_ha_uml.png)

#### AkkaRpcActor

接收Rpc消息，交给对应等对象进行处理。
Flink基于Akka搭建RPC服务，用于JobManager、TaskManager等的交互。

# 主要流程

#### 本地执行job的主要流程

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_example_WordCount_flow.png)

* 创建StreamExecutionEnvironment；
* 创建StreamGraph；
* 创建JobGraph；
* 启动MiniCluster；
* 启动TaskExecutor；
* 启动ResourceManager；
* 启动Dispatcher，Dispatcher启动JobMaster的leader选举；
* JobMasterLeadershipRunner判断成为leader后，启动JobMaster，JobMaster启动后开始执行job；

本地执行job时，由于只有一个JobMaster、一个TaskExecutor，因此HighAvailabilityServices、TaskExecutor启动流程与集群模式有所不同，不需要进行真正的leader选举过程，也不需要TaskExecutor的增减过程。


#### RPC通信

Flink基于Akka实现了JobMaster、TaskExecutor、Dispatcher、ResourceManager等之间的信息交互。

接口继承关系如下图。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_rpc_gateway_uml.png)

从JobMaster等类等构造函数可以看出，创建JobMaster等对象时，都首先启动相应的RPC节点，从而构造RPC服务。

Flink中基于Akka实现了JobManager、TaskManager、JobClient间的分布式通信，代替了原来的RPC服务。Akka是一个基于Actor模型的**异步通信框架**。

Akka中有多个Actor，每个Actor有一个Mailbox，用于接收其他Actor发来的消息。

涉及模块: flink-rpc。

AkkaRpcActor：flink rpc中主要的Actor，负责启动TaskManager等。

AkkaRpcService：负责管理Actor，构建Rpc服务，利用map维护Actor与RpcEndpoint的关系。

```
// JobMaster类
super(
            rpcService,
            RpcServiceUtils.createRandomName(JOB_MANAGER_NAME),
            jobMasterId); // 启动相应RPC节点
```

#### ResourceManager

ResourceManager负责资源申请/释放/管理。

它基于ResourceManagerDriver来请求/释放外部资源，目前包括Yarn和Kubernetes资源管理。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_rm_driver_uml.png)

# 总结

通过对WordCount这个例子一步步的debug，了解了job执行的最简单流程，也找到了Flink的最核心内容，如JobManager、TaskManager等，为之后分模块深入地看打下了基础。