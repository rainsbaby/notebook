## 简介

从spark源码中的例子JavaWordCount开始debug，了解spark job的主要执行流程，以及核心概念如stage、task的生成及执行流程。


## 主要类

Code版本：3.2.1.

### SparkContext

Spark功能主要入口。表示与Spark cluster的关联，可用于创建RDD、累加器、broadcast变量到cluster中。

### SparkSession

利用Dataset和DataFrame API对spark进行编程的入口。

### RDD

弹性分布式数据集。代表不可变的、分区的数据集，可以并行处理。

内部主要包括属性：partitions，对每个split执行对计算函数，对其他RDDs的依赖，key-value RDD的partitioner，期望计算每个split的位置。

### DAGScheduler

高层次的调度层，实现以stage为导向的调度。

计算job中stage的DAG图，跟踪RDD和stage输出，找到job执行的最小调度。之后以TaskSet的形式提交stage到TaskScheduler。TaskSet包含完全独立的task，task基于cluster上已经存在的数据执行，如果数据不存在了就会执行失败。
 
根据RDD graph的shuffle边生成stage。

- 窄依赖的操作，如map、filter，会被放入stage的一个task集中；
- 有shuffle依赖的操作，需要多个stage（一个写入数据到文件中，另一个在barrier后读取这些文件）。

每个stage只对其他stage有shuffle依赖，可能在内部计算多个operation。
operation真正的串行化，发生在RDD.compute()中。

DAGScheduler会处理shuffle输出文件丢失问题，让旧的stage重新提交计算。
stage内不是由shuffle文件丢失引起的失败，由TaskScheduler处理，重试每个task一定次数，若还失败则取消整个stage。

	Job（ActiveJob）：提交给scheduler的最高级别的任务。
	Stage：job中的一系列task，计算中间结果。每个task在RDD的各个partition上执行相同的函数。
	Stage之间由shuffle boundary分隔，这引入了barrier，在barrier处等待前一个stage完成之后才能获取它的输出文件。
	Stage分为ResultStage（最终阶段）和ShuffleMapStage（输出shuffle后的数据文件）。
	Task：一个独立单元的任务，每个被发送到一台机器上。

### DAGSchedulerEventProcessLoop

DAGScheduler中实现的EventLoop子类。

EventLoop是事件循环处理器，接收来自调用者的event，并在event线程中处理所有event。由一个单独的event线程处理所有event。

### ShuffleMapStage 与 ResultStage

- ShuffleMapStage：DAG中的中间stage，为shuffle生成数据。在shuffle操作前发生，可能包含多个串行的operation。执行时，保存输出文件，之后被reduce task获取使用。
- ResultStage： 对RDD对partition执行计算，得到action的计算结果。利用func表示要执行的函数，会应用到每个partition。有些stage可能只执行在部分partition上，如first/lookup这种action。


### TaskScheduler

低层次的task调度接口。每个TaskScheduler为一个单独的SparkContext调度任务。

接收DAGScheduler提交的一系列task，发送这些task到cluster、执行、失败时重试。返回event给DAGScheduler。

### SchedulerBackend

调度系统的后端接口，允许在TaskSchedulerImpl中插入不同的实例。

Mesos-like 模型，application获得resource后，启动task。

JavaWordCount中涉及的类型为LocalSchedulerBackend。

### RPC 相关

Spark 历史版本中曾利用Akka实现RPC，在最近的版本中已经去掉Akka，直接基于Netty实现RPC。

- RpcEndpointRef：远程RpcEndpoint的引用，持有它就可以向RpcEndpoint发送消息。NettyRpcEndpointRef是NettyRpcEnv版本的RpcEndpointRef。
- RpcEndpoint：RPC节点，定义每个消息触发的动作。保证onStart, receive and onStop按顺序执行。
- RPCEnv：RpcEndpoint 注册在RpcEnv中后接收消息。RpcEnv处理来自于RpcEndpointRef或远程node的消息，并转发给相应的RpcEndpoint。
- Dispatcher：消息分发，负责将RPC消息分发给合适的Endpoint。
- MessageLoop：Dispatcher使用MessageLoop来传递消息到endpoint。
- Inbox：存储发送给RpcEndpoint的消息，并以线程安全的方式提交给它。
- TransportServer：构建RPC的Server端，其中RpcHandler负责处理rpc消息。
- TransportClient：用于获取预先协商好的流的连续块的client端。目的是有效传输大量的数据，这些数据被分割成大小不等的几百KB到几MB的块。TransportClient向server发送request，TransportResponseHandler负责处理server端的response。

### Executor

利用一个线程池执行task。可被用于Mesos、Yarn、Kubernetes及standalone scheduler。内部有一个RPC接口用于与driver交互，Memos fine-grained模式下除外。

### ShuffleMapTask 与 ResultTask

Task是一个执行单元，内部是串行的operator。包括ShuffleMapTask和ResultTask。

一个job包含一或多个stage。job的最后stage包含多个ResultTask，前面的stage包含ShuffleMapTask。

ResultTask执行后发送输出到driver上。ShuffleMapTask执行后，分割输出到多个bucket（依据task的partitioner分割）。


## 主要流程


![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_JavaWordCount_flow.drawio.png)

JavaWordCount中，参数spark.master为“local”，driver、executor都在同一个JVM中运行。

与local模式相对应的还有cluster模式。cluster模式中，Driver向ClusterManager提交任务，ClusterManager将任务分配给Worker运行，每个Worker中可能有多个Executor，Executor负责具体的task执行。

cluster模式中，可以有Standalone、Yarn、Mesos等部署模式。在standalone中，ClusterManager为Master；在Yarn中，ClusterManager为ResourceManager。结构如下图。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_cluster_structure_overview.png)

Standalone集群，目前支持两种部署模式：
	
	- client模式：driver在client端提交application的相同进程中运行。
	- cluster模式：driver在集群中某一个worker上运行，driver完成提交application的任务后就退出，不会等待application完成。


### RPC 主要内容

Spark RPC主要内容如下图。基于Netty实现的RPC，TransportClient负责从Outbox中获取并发送Message，NettyRpcHandler负责处理消息及放入到Inbox中。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_rpc_structure.drawio.png)


以下为NettyRPCEnv.send()，可以看出发送给本地RPC节点和远程节点的逻辑不同。

- 当发送给本地RPC endpoint时，直接由dispatcher发送到Inbox。
- 当发送给远程RPC endpoint时，由client直接发送或者先放入Outbox之后再发送。然后由NettyRpcHandler接收并经过MessageLoop放入到相应的Inbox中。


```
// NettyRPCEnv.scala
private[netty] def send(message: RequestMessage): Unit = {
    val remoteAddr = message.receiver.address
    if (remoteAddr == address) {
      // Message to a local RPC endpoint.
      try {
        dispatcher.postOneWayMessage(message)
      } catch {
        case e: RpcEnvStoppedException => logDebug(e.getMessage)
      }
    } else {
      // Message to a remote RPC endpoint.
      postToOutbox(message.receiver, OneWayOutboxMessage(message.serialize(this)))
    }
  }
```

### stage划分及Job执行

Job执行时，会划分stage，得到由stage组成的DAG。DAG的生成是在DAGScheduler中完成的，入口为handleJobSubmitted()方法。

我们知道，stage分为ShuffleMapStage和ResultStage，ResultStage是job中最后一个stage，即对RDD执行action运算得到的。

ShuffleMapStage和ResultStage都可能有一个或多个依赖，称为parents。

Job执行过程：

- 获取ResultStage。
	- 获取ResultStage的ShuffleDependencies，方法getShuffleDependenciesAndResourceProfiles。ShuffleDependency即为宽依赖，还有NarrowDependency为窄依赖。
	- 计算ResultStage的parents，方法getOrCreateParentStages。在这个过程中，递归计算所有祖先stage，即得到了DAG。
	- 基于parents、rdd、func、partitions等创建ResultStage。
- 提交ResultStage。
	- 查看stage的parents数据是否存在，若存在则提交本stage；若不存在，则先提交parents，parent执行完成后唤醒子stage的执行。这样，就实现了从上至下的执行。
	- 提交stage中task到TaskScheduler。
	- LocalSchedulerBackend通过local的RpcEndpointRef发送ReviveOffers消息，LocalEndpoint接收命令后，获取task并交给Executor执行。Executor利用一个线程池执行task。
	- TaskRunner线程执行单个task的计算。

#### 如何划分stage？

从最后一个RDD开始向上回溯，若RDD对父RDD的依赖为ShuffleDependency即宽依赖，则找到了stage的起点；若不是宽依赖，则继续向上回溯，直到找到第一个ShuffleDependency。

可见方法getShuffleDependenciesAndResourceProfiles。


#### Task中串行的operation如何执行？

一个stage由一或多个并行的task组成。task中可能有多个串行的operation，如filter、map等。在task计算时，是如何实现operation的串行计算的呢？

以ShuffleMapTask的执行为例，流程为：

- 计算一个rdd的iterator过程：获取当前rdd的parent.iterator，然后计算当前rdd的iterator。因此实现task的串行执行。以JavaWordCount为例，ShuffleMapStage中最后一个rdd为MapPartitionsRDD，向上循环得到第一个rdd为FileScanRDD。首先计算FileScanRDD，然后依次串行执行后面的rdd的operation。
- 执行ShuffleWriteProcessor.write过程，调用ShuffleWriter.write进行输出。在SortShuffleWriter.write中，会利用sorter进行排序，然后输出。


