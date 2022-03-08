
[toc]

在集群中，每个TaskManager都是一个单独的进程（非MiniCluster模式）。TaskManager为每个Task分配独立的执行线程。

源码版本：Flink 1.14。

### StreamElement

在流中传输的数据主要有StreamElement和Event，它们的主要类别如下。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_StreamElement_uml.png)

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_RuntimeEvent_uml.png)


### 内存管理

JVM内存管理的不足：

* 有效数据密度低
* 垃圾回收不可控
* OOM问题影响稳定性
* 缓存未命中问题

因为JVM存在诸多问题，所以越来越多的大数据计算引擎选择自行管理JVM内存，如Spark、Flink、HBase，尽量达到C/C++ 一样的性能，同时避免OOM的发生。

Flink自主内存管理，包括内存管理、定制的序列化工具、缓存友好的数据结构和算法、堆外内存等。

对于可以用作 key 的数据类型，TypeInfomation 还可以生成 TypeComparator，用来直接在序列化后的二进制数据上进行 compare、hash 等操作。

这种基于 MemorySegment 和二进制数据直接管理数据对象的方式可以带来如下好处：

    保证内存安全：由于分配的 MemorySegment 的数量是固定的，因而可以准确地追踪 MemorySegment 的使用情况。在 Batch 模式下，如果 MemorySegment 资源不足，会将一批 MemorySegment 写入磁盘，需要时再重新读取。这样有效地减少了 OOM 的情况。
    减少了 GC 的压力：因为分配的 MemorySegment 是长生命周期的对象，数据都以二进制形式存放，且 MemorySegment 可以回收重用，所以 MemorySegment 会一直保留在老年代不会被 GC；而由用户代码生成的对象基本都是短生命周期的，Minor GC 可以快速回收这部分对象，尽可能减少 Major GC 的频率。此外，MemorySegment 还可以配置为使用堆外内存，进而避免 GC。
    节省内存空间：数据对象序列化后以二进制形式保存在 MemorySegment 中，减少了对象存储的开销。
    高效的二进制操作和缓存友好的计算：可以直接基于二进制数据进行比较等操作，避免了反复进行序列化于反序列；另外，二进制形式可以把相关的值，以及 hash 值，键值和指针等相邻地放进内存中，这使得数据结构可以对高速缓存更友好。

### OperatorChain 内部的数据传递

OperatorChain 的内部类 ChainingOutput 实现了 WatermarkGaugeExposingOutput 接口，它持有一个 OneInputStreamOperator, 即 OperatorChain 中当前算子的下游算子。当 ChainingOutput 接收到当前算子提交的数据时，直接将调用下游算子的 processElement 方法。

通过在 ChainingOutput 中保存下游 StreamOperator 的引用，ChainingOutput 直接将对象的引用传递给下游算子。

但是 ExecutionConfig 有一个配置项，即 objectReuse，在默认情况下会禁止对象重用。如果不允许对象重用，则不会使用 ChainingOutput，而是会使用 CopyingChainingOutput。顾名思义，它和 ChainingOutput 的区别在于，它会对记录进行拷贝后传递给下游算子。


### Slot管理

TaskExecutor中TaskSlotTable管理单个TaskExecutor中的slot。

ResourceManager对所有TaskExecutor中的slot进行管理。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_resourcemanage_architecture.png)


### 状态管理

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_statebackend_uml.png)

定义流式应用的状态如何在集群内存储。主要有两种类型的StateBackend：

 * HashMapStateBackend在TaskManager的内存中存储state，轻量级且没有额外的依赖。
 * EmbeddedRocksDBStateBackend在RocksDB中存储state，能够存储大量的数据，只受限于TaskManager的磁盘容量。
 
Flink 中的状态分为Keyed State 和 Operator State。 Keyed State 是和具体的 Key 相绑定的，只能在 KeyedStream 上的函数和算子中使用。 Opeartor State 则是和 Operator 的一个特定的并行实例相绑定的，

StateBackend接口创建的CheckpointableKeyedStateBackend和OperatorStateBackend，定义了key state和operator state的存储方式。同时定义了如何checkpoint state。




### Window
Window的主要处理逻辑，即对应Operator主要有WindowOperator，其中包含WindowAssigner、Evictor、Trigger等，主要逻辑位于processElement()。

从WindowOperator可以看出，当消息到达时，在窗口算子中的主要处理流程如下：

* 通过 WindowAssigner 确定消息所在的窗口（可能属于多个窗口）
* 将消息加入到对应窗口的状态中
* 根据 Trigger.onElement 确定是否应该触发窗口结果的计算，如果使用 InternalWindowFunction 对窗口进行处理
* 注册一个定时器，在窗口结束时清理窗口状态
* 如果消息太晚到达，提交到 side output 中

#### WindowAssigner
为element分配0或多个窗口。

#### Evictor
在调用Trigger后，在WindowFunction计算前/后从pane中移除element。

#### Trigger

决定一个窗口何时关闭进行计算。
 
 WindowOperator端，处理element时调用onElement等方法进行判断。


### 双流操作

#### Window Join and CoGroup

Window Join 操作，顾名思义，是基于时间窗口对两个流进行关联操作。相比于 Join 操作， CoGroup 提供了一个更为通用的方式来处理两个流在相同的窗口内匹配的元素。 Join 复用了 CoGroup 的实现逻辑。

JoinFunction 主要关注的是两个流中按照 key 匹配的每一对元素，而 CoGroupFunction 的参数则是两个中 key 相同的所有元素。JoinFunction 的逻辑更类似于 INNER JOIN，而 CoGroupFunction 除了可以实现 INNER JOIN，也可以实现 OUTER JOIN。

#### Connected Streams

更为通用的双流操作。

#### Interval Join

Window Join 的一个局限是关联的两个数据流必须在同样的时间窗口中。

但有些时候，我们希望在一个数据流中的消息到达时，在另一个数据流的一段时间内去查找匹配的元素。更确切地说，如果数据流 b 中消息到达时，我们希望在数据流 a 中匹配的元素的时间范围为 a.timestamp + lowerBound <= b.timestamp <= a.timestamp + upperBound；同样，对数据流 a 中的消息也是如此。在这种情况，就可以使用 Interval Join。



### 异步I/O

#### 作用

处理实时数据时，有时需要从外部查询数据，如外表维表数据。在外部查询较慢的情况下，如果采用同步调用，会严重影响系统的吞吐量。

虽然可以通过增加并行度的方式，来提升系统吞吐量，但是也会增加系统的资源需求。

因此可以采用异步调用的方式，调用外部接口，从而提升系统的吞吐量。

#### 使用方式：

```
class AsyncDatabaseRequest extends RichAsyncFunction<String, Tuple2<String, String>> {

    /** The database specific client that can issue concurrent requests with callbacks */
    private transient DatabaseClient client;

    @Override
    public void open(Configuration parameters) throws Exception {
        client = new DatabaseClient(host, post, credentials);
    }

    @Override
    public void close() throws Exception {
        client.close();
    }

    @Override
    public void asyncInvoke(String key, final ResultFuture<Tuple2<String, String>> resultFuture) throws Exception {

        // issue the asynchronous request, receive a future for result
        final Future<String> result = client.query(key);

        // set the callback to be executed once the request by the client is complete
        // the callback simply forwards the result to the result future
        CompletableFuture.supplyAsync(new Supplier<String>() {
            @Override
            public String get() {
                try {
                    return result.get();
                } catch (InterruptedException | ExecutionException e) {
                    // Normally handled explicitly.
                    return null;
                }
            }
        }).thenAccept( (String dbResult) -> {
            resultFuture.complete(Collections.singleton(new Tuple2<>(key, dbResult)));
        });
    }
}

// create the original stream
DataStream<String> stream = ...;

// apply the async I/O transformation
DataStream<Tuple2<String, String>> resultStream =
    AsyncDataStream.unorderedWait(stream, new AsyncDatabaseRequest(), 1000, TimeUnit.MILLISECONDS, 100);

```

#### 内部原理：

对应Operator为AsyncWaitOperator，Function基类为AsyncFunction。

**AsyncWaitOperator：**

AsyncWaitOperator中利用StreamElementQueue队列，存储处理中的element。

* element到达时，首先将其放入队列中，然后发起异步调用。异步调用的回调（ResultHandler）中，将结果更新到队列中对应entry，并输出队列中所有已完成的element；
* watermark到达时，先放入队列中，然后输出队列中所有已完成的element到下游。因为watermark到达时，表示没有更早的element，所以可以输出队列中所有已完成的element。

进行快照时，需要将队列记录到state中，包括异步调用没有完成，以及还没有发送给下游的消息。在恢复到时候，可以取出队列消息，再重新处理一遍。



### 序列化与反序列化

可以看出StreamGraph、JobGraph、ExecutionGraph中，operator、userFunction等会从Client传递给JobManager，再传递给TaskManager端进行执行，因此需要序列化反序列化。

从代码里也可以看出，它们就实现了Serializable接口。

Flink实现了自己的序列化框架。因为在 Flink 中处理的数据流通常是同一类型，由于数据集对象的类型固定，对于数据集可以只保存一份对象Schema信息，节省大量的存储空间。同时，对于固定大小的类型，也可通过固定的偏移位置存取。当我们需要访问某个对象成员变量的时候，通过定制的序列化工具，并不需要反序列化整个Java对象，而是可以直接通过偏移量，只是反序列化特定的对象成员变量。如果对象的成员变量较多时，能够大大减少Java对象的创建开销，以及内存数据的拷贝大小。

使用各类数据类型时，Flink会自动探测传入的数据类型，生成对应的TypeInformation，调用对应的序列化器，因此用户其实无需关心类型推测。

Flink支持任意的Java或是Scala类型。Flink 在数据类型上有很大的进步，不需要实现一个特定的接口（像Hadoop中的org.apache.hadoop.io.Writable），Flink 能够**自动识别数据类型**。Flink 通过 Java Reflection 框架分析基于 Java 的 Flink 程序 UDF (User Define Function)的返回类型的类型信息，通过 Scala Compiler 分析基于 Scala 的 Flink 程序 UDF 的返回类型的类型信息。类型信息由 TypeInformation 类表示，TypeInformation 支持以下几种类型：

    BasicTypeInfo: 任意Java 基本类型（装箱的）或 String 类型。
    BasicArrayTypeInfo: 任意Java基本类型数组（装箱的）或 String 数组。
    WritableTypeInfo: 任意 Hadoop Writable 接口的实现类。
    TupleTypeInfo: 任意的 Flink Tuple 类型(支持Tuple1 to Tuple25)。Flink tuples 是固定长度固定类型的Java Tuple实现。
    CaseClassTypeInfo: 任意的 Scala CaseClass(包括 Scala tuples)。
    PojoTypeInfo: 任意的 POJO (Java or Scala)，例如，Java对象的所有成员变量，要么是 public 修饰符定义，要么有 getter/setter 方法。
    GenericTypeInfo: 任意无法匹配之前几种类型的类。

前六种数据类型基本上可以满足绝大部分的Flink程序，针对前六种类型数据集，Flink皆可以自动生成对应的TypeSerializer，能非常高效地对数据集进行序列化和反序列化。

对于最后一种数据类型，Flink会使用Kryo进行序列化和反序列化。

每个TypeInformation中，都包含了serializer，类型会自动通过serializer进行序列化，然后用Java Unsafe接口写入MemorySegments。

对于可以用作key的数据类型，Flink还同时自动生成TypeComparator，用来辅助直接对序列化后的二进制数据进行compare、hash等操作。

对于 Tuple、CaseClass、POJO 等组合类型，其TypeSerializer和TypeComparator也是组合的，序列化和比较时会委托给对应的serializers和comparators。


#### TypeInformation

在Flink中，统一使用TypeInformation类表示。TypeInformation的一个重要的功能就是创建TypeSerializer序列化器，为该类型的数据做序列化。每种类型都有一个对应的序列化器来进行序列化。
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_TypeInformation_uml.png)


#### TypeSerializer

TypeSerializer即序列化器，定义了某个数据类型对应的序列化反序列化和copy方法。

因此序列化流程可以简单总结为，根据对象类型可得到对应的TypeInformation，由TypeInformation创建对应的TypeSerializer，之后画即可进行序列化工作。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_TypeSerializer_uml.png)


### 参考

[Generating Watermarks](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/datastream/event-time/generating_watermarks/)

[异步I/O](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/datastream/operators/asyncio/)

[异步I/O 设计和实现](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65870673)




