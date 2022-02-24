在集群中，每个TaskManager都是一个单独的进程（非MiniCluster模式）。TaskManager为每个Task分配独立的执行线程。


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

主要有两种类型的StateBackend：

* HashMapStateBackend
* EmbeddedRocksDBStateBackend

### 参考



