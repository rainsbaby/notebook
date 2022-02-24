Flink网络通信及反压机制

## 简介

Flink中JobManager、TaskManager、ResourceManager之间，基于Akka建立RPC通信，进行集群管理、job控制等。

而job在TaskManager间的数据流转，如果基于RPC通信，显然效率不高。因此Flink建立了基于Netty的网络通信，并从基于TCP的反压发展到基于Credit的反压机制。

## 核心类

Flink版本 : 1.14 。


#### IntermediateResult
JobVertex的输出，一个JobVertex可能哟0/n个输出。

IntermediateResult可以包含0/n个输出分区IntermediateResultPartition。

#### IntermediateResultPartition

IntermediateResult的一个输出分区。


#### RecordWriter

负责写record到channel中。

主要实现类有ChannelSelectorRecordWriter和BroadcastRecordWriter，分别负责写入某个channel和写入所有channel。

ChannelSelectorRecordWriter类中有ChannelSelector，负责决定写入到哪个partition中。

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


#### MemorySegment

Flink管理的最小内存单元。可以表示堆内内存、堆外内存等。


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

```
// Server端channel handler
public ChannelHandler[] getServerChannelHandlers() {
    PartitionRequestQueue queueOfPartitionQueues = new PartitionRequestQueue();
    // 上游主要逻辑
    PartitionRequestServerHandler serverHandler =
            new PartitionRequestServerHandler(
                    partitionProvider, taskEventPublisher, queueOfPartitionQueues);

    return new ChannelHandler[] {
        messageEncoder,
        new NettyMessage.NettyMessageDecoder(),
        serverHandler,
        queueOfPartitionQueues
    };
}

// Client端channel handler
public ChannelHandler[] getClientChannelHandlers() {
	// 下游主要逻辑
    NetworkClientHandler networkClientHandler = new CreditBasedPartitionRequestClientHandler();

    return new ChannelHandler[] {
        messageEncoder,
        new NettyMessageClientDecoderDelegate(networkClientHandler),
        networkClientHandler
    };
}
```

从中可以看出，使用基于**credit**的流控机制。

负责发送数据的主要逻辑位于PartitionRequestServerHandler。

负责接收数据的主要逻辑位于CreditBasedPartitionRequestClientHandler。

#### NettyConnectionManager

负责管理Netty连接，包括创建NettyServer、NettyClient及NettyBufferPool。

NettyServer在启动时会配置**水位线**，如果Netty输出缓冲中的字节数超过了高水位值，会等到其降到低水位值一下才继续写入数据。

#### NetworkBufferPool

固定大小的MemorySegment池。

 * NetworkBufferPool创建LocalBufferPool，task从LocalBufferPool中获取buffer。
 * 当新的local buffer创建后，NetworkBufferPool动态地重新分配buffer给所有pool。
 * 每个task有一个LocalBufferPool，整个TaskManager中只有一个NetworkBufferPool，LocalBufferPool由NetworkBufferPool管理。

#### LocalBufferPool

负责管理从NetworkBufferPool得到的一批buffer。
每个task有一个LocalBufferPool，整个TaskManager中只有一个NetworkBufferPool。

## 主要流程

### Task间数据交换

Flink中的数据交换围绕以下设计原则：

    数据交换的控制流（即为了启动交换而进行的消息传递）是由接收者发起的，与原始的MapReduce很相似。
    数据交换的数据流，即数据的实际传输是由IntermediateResult的概念抽象出来的，并且是可插拔的。这意味着系统可以用相同的实现方式支持流式数据传输和批量数据传输。
    
数据交换涉及几个对象，包括：

* JobManager，主节点，负责调度任务、恢复和协调，并通过ExecutionGraph掌握job的全貌。
* TaskManagers，工作节点。一个TM在线程中同时执行许多Task。每个TM还包含一个通信管理器（CM--任务之间共享）和一个内存管理器（MM--任务之间也共享）。TM可以通过TCP连接相互交换数据，这些连接在需要时创建。

请注意，在Flink中，在网络上交换数据的是TaskManager，而不是Task。也就是说，在同一TM中的Task之间的数据交换复用一个网络连接。    

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_dataexchange_jm_tm.png)

ExecutionGraph，包含了关于job计算的 "基本事实"。它由代表计算任务的顶点（ExecutionVertex）和代表任务产生的数据的中间结果（IntermediateResultPartition）组成。顶点通过ExecutionEdges（EE）与它们所消耗的中间结果相连。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_dataexchange_intermediateresult.png)


ExecutionGraph是在JobManager中的数据结构。它们在TaskManagers中有其运行时的等效结构，负责实际的数据处理。IntermediateResultPartition的运行时等价物为ResultPartition。

**ResultPartition**（RP）代表了BufferWriter写入的一大块数据，也就是由一个task产生的一大块数据。一个RP是Result Subpartition（RS）的集合。这是为了区分给不同接收者的数据，例如，减少或连接的分区shuffle的情况。

**ResultSubpartition**（RS）表示由operator创建的数据的一个分区，以及将该数据转发给接收operator的逻辑。一个RS的具体实现决定了实际的数据传输逻辑，这也是允许系统支持各种数据传输的可插拔机制。例如，PipelinedSubpartition是一个流水线实现，支持流式数据交换。SpillableSubpartition是一种阻塞式实现，支持批量数据交换。

**InputGate**：在接收端与RP的逻辑等同。它负责收集数据的缓冲区，并将其交给上游。

**InputChannel**：在接收端与RS的逻辑对应。它负责收集特定分区的数据缓冲区。

#### Record生命周期

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_dataexchange_architecture.png)

上图更详细地介绍了record从生产者运到消费者的生命周期。

最初，MapDriver产生record（由Collector收集），并传递给RecordWriter对象。RecordWriter包含一些序列化器（RecordSerializer对象），每个可能会消费这些记录的消费者任务有一个。例如，在shuffle或广播中，有多少个serializer，就有多少个消费task。

ChannelSelector会选择一个或多个serializer来放置记录。例如，如果记录是广播式的，它们将被放置在每个serializer中。如果记录是hash分区的，ChannelSelector将根据record的哈希值来选择适当的序列化器。

serializer将record序列化为其二进制表示，并将其置于固定大小的buffer（记录可以跨越多个buffer）。这些serializer被交给BufferWriter，并被写入ResultPartition（RP）。RP由几个子分区（ResultSubpartitions - RSs）组成，为特定的消费者收集serializer。在图片中，缓冲区是为第二个reducer（在TaskManager 2中）准备的，它被放在RS2中。由于这是第一个buffer，RS2可用于消费（注意，这种行为实现了streaming shuffle），并通知JobManager。

JobManager查找RS2的消费者，并通知TaskManager 2有一大块数据可用。到TM2的消息被传播到应该接收这个buffer的InputChannel，它反过来通知RS2可以启动网络传输。然后，RS2把缓冲区交给TM1，TM1再把它交给netty。网络连接是长期运行的，存在于TaskManagers之间，而不是单个任务。

一旦buffer被TM2接收，它就会通过一个类似的对象层次，从InputChannel（相当于IRPQ的接收方）开始，到InputGate（包含几个IC），最后在RecordDeserializer中结束，它从serializer产生类型化的record并将它们交给接收任务，在这里是ReduceDriver。



### Flink反压机制

#### 基于TCP的反压
在基于credit的流量控制出现之前，Flink 基于TCP流控+bounded buffer实现反压，如下图所示。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_network_backpressure_1.4.png)

数据流转如下图所示。
生产端：

* Operator（如StreamMap），经过Function处理输入后，将结果输出到RecordWriter。
* RecordWriter将结果传递给ResultPartition，写入某个指定的channel，或所有channel。
* ResultPartition将record写入对应buffer。
* buffer满了或flush时，由netty消费buffer，写入tcp缓冲区。

消费端：

* TCP缓冲区有数据可消费时，Netty handler接收数据，并传递给对应InputChannel。
* InputChannel将数据传递给RecordReader。
* RecordReader将数据交给Operator处理。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_backpressure_tcpbase.png)

基于TCP反压实现简单，但有如下弊端：

	在一个 TaskManager 中可能要执行多个 Task，如果多个 Task 的数据最终都要传输到下游的同一个 TaskManager 就会复用同一个 Socket 进行传输，这个时候如果单个 Task 产生反压，就会导致复用的 Socket 阻塞，其余的 Task 也无法使用传输，checkpoint barrier 也无法发出导致下游执行 checkpoint 的延迟增大。
	依赖最底层的 TCP 去做流控，会导致反压传播路径太长，导致生效的延迟比较大。

#### 基于Credit的反压	

因此Flink 1.5中引入了**credit-base反压**，在应用层模拟TCP的反压，如下图所示。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_network_creditbase.png)

Credit-based 流量控制可确保发送端已经发送的任何数据，接收端都具有足够的能力(Buffer)来接收。

新的流量控制机制基于网络缓冲区的可用性，作为 Flink 之前机制的自然延伸。

每个远程输入通道(RemoteInputChannel)现在都有自己的一组独占缓冲区(Exclusive buffer)，而不是只有一个共享的本地缓冲池(LocalBufferPool)。

与之前不同，本地缓冲池中的缓冲区称为流动缓冲区(Floating buffer)，因为它们会在输出通道间流动并且可用于每个输入通道。

基于credit的流量控制，指数据接收方将自身缓冲区的可用性，作为对发送者的credit（1 buffer=1 credit）。每个Result Subpartition将追踪其channel credit。只有在credit可用的情况下，缓冲区才会被转发到下层网络堆栈，每发送一个缓冲区就会减少一个credit。

除了缓冲区外，我们还发送关于当前积压大小的信息，这指定了有多少缓冲区在这个子分区的队列中等待。接收方将利用这一点来要求获得适当数量的浮动缓冲区，以加快积压处理。它将尝试获得与积压大小一样多的浮动缓冲区，但这并不总是能获取到，我们可能得到一些或根本没有缓冲区。接收方将利用检索到的缓冲区，并将监听进一步的缓冲区变得可用以继续。


下图为Flink网络传输的数据传输详细过程。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_backpressure_creditbase.png)

与基于TCP的反压不同之处是，不是被动地等待所有缓冲区满，而是提前进行沟通。

ResultSubPartition向InputChannel发送消息时，会发送backlog size告知下游它准备发送多少数据。下游会计算有多少的buffer去接收消息，如果有充足的buffer就给上游一个credit，告知上游能够发送数据。

### Storm的反压机制

Storm中每个bolt都有一个监测反压的线程，一旦检测到bolt的接收队列里出现了严重阻塞，就将状态写入到zookeeper中。

Spout端监听zookeeper上状态变化，检测到有反压时，就停止发送。

检测有延迟，且可能出现部分bolt饥饿的情况。


## 总结

本文主要介绍了Task之间数据交换机制，以及反压机制的发展。

## 参考

[Data exchange between tasks](https://cwiki.apache.org/confluence/display/FLINK/Data+exchange+between+tasks)

[A Deep-Dive into Flink's Network Stack](https://flink.apache.org/2019/06/05/flink-network-stack.html)

[Flink 源码阅读笔记（8）- Task 之间的数据传输](https://blog.jrwang.me/2019/flink-source-code-data-exchange/)