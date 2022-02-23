Flink网络通信及反压机制

## 简介

## 核心类

Flink版本 : 1.14 。

JobMaster

TaskExecutor

ExecutionJobVertex

ExecutionVertex

IntermediateResult

IntermediateResultPartition

#### RecordWriter

负责写record到channel中。

主要实现类有ChannelSelectorRecordWriter和BroadcastRecordWriter，分别负责写入某个channel和写入所有channel。

ChannelSelectorRecordWriter类中有ChannelSelector，负责决定写入到哪个channel中。

#### ResultPartitionWriter

在RecordWriter类中负责写入record到ResultPartition。

主要实现类可见ConsumableNotifyingResultPartitionWriter，可在第一个record生成时发送一条可消费的通知，发送给JobManager。


#### ResultPartition

由ResultPartitionFactory类，根据ResultPartitionType类型创建不同的ResultPartition。

```
//ResultPartitionType类型的各个参数

/** Can the partition be consumed while being produced? */
private final boolean isPipelined;

/** Does the partition produce back pressure when not consumed? */
private final boolean hasBackPressure;

/** Does this partition use a limited number of (network) buffers? */
private final boolean isBounded;

/** This partition will not be released after consuming if 'isPersistent' is true. */
private final boolean isPersistent;

/** Can the partition be reconnected. */
private final boolean isReconnectable;
```

#### InputGate
InputGate消费Intermediate Result中的一个或多个partition。
封装了InputChannel，内部包含多个InputChannel。每个intermediate result partition有一个对应的input channel。

#### InputChannel
一个InputChannel消费一个ResultSubpartitionView。

BufferBuilder


BufferConsumer


MemorySegment


NettyServer
NettyClient

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

CreditBasedPartitionRequestClientHandler

NetworkBufferPool
LocalBufferPool


#### NettyProtocol

负责定义Netty Server和Client中的channel handler。主要方法如下。



## 主要流程

Mini集群启动流程可以参考之前看过的WordCount这个例子。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_example_WordCount_flow.png)


下面重点关注集群启动后，job的deploy过程，以WindowWordCount这个例子为基础来看。


![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_dataexchange_jm_tm.png)

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_dataexchange_intermediateresult.png)

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_dataexchange_architecture.png)

## 总结


## 参考
