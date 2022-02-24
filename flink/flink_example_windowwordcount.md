

## 简介

## 核心类

Flink版本 : 1.14 。

JobMaster

TaskExecutor

ExecutionJobVertex

ExecutionVertex

IntermediateResult

IntermediateResultPartition


#### ResultPartition

由ResultPartitionFactory类，根据ResultPartitionType类型创建不同的ResultPartition。

#### InputGate
InputGate消费Intermediate Result中的一个或多个partition。
封装了InputChannel，内部包含多个InputChannel。每个intermediate result partition有一个对应的input channel。

#### InputChannel
一个InputChannel消费一个ResultSubpartitionView。

子类RemoteInputChannel是请求远端partition queue的InputChannel。利用一个队列管理接收到的buffer。由网络I/O线程负责入队，由接收到task线程负责消费。


#### ShuffleEnvironment

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

Mini集群启动流程可以参考之前看过的WordCount这个例子。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_example_WordCount_flow.png)


下面重点关注集群启动后，job的deploy过程，以WindowWordCount这个例子为基础来看。


## 总结


## 参考
