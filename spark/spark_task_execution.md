Task执行过程中的数据存储与传输

## 简介

Spark根据宽依赖/窄依赖将job分为多个stage，stage分为ShuffleMapStage和ResultStage。ShuffleMapStage中为多个并行的ShuffleMapTask，生成的shuffle数据交给下游的ShuffleMapTask或ResultTask进行处理。

Task执行完成后，将结果存储在内存或磁盘中。下游的task执行时，从上游task的executor处获取数据进行后续的计算。

## 主要类
spark 3.2.1.

### BlockManager
在driver和executor中运行，本地或远程存取block到存储系统中，包括内存、磁盘、堆外内存。

包含BlockManagerMaster、MemoryManager、ShuffleManager、BlockTransferService等对象。

### BlockManagerMaster
负责代理BlockManager与driver上的BlockManagerMasterEndpoint通信。

### BlockManagerMasterEndpoint
位于driver上，追踪所有存储节点上BlockManager的状态。

### BlockTransferService
Block传输服务，用于不同阶段任务之间的block数据的传输与读写。

主要实现类为NettyBlockTransferService，使用netty获取block。其中RpcHandler为NettyBlockRpcServer。

### TaskMemoryManager
管理一个task中分配的内存。

包括MemoryManager的引用，以及一或多个MemoryConsumer。MemoryConsumer即内存的消耗者，向TaskMemoryManager申请内存，实现类有ExternalSorter、ShuffleExternalSorter等。

### MemoryManager
管理所有执行与存储共享内存。位于SparkEnv中，每个JVM中有一个MemoryManager。

其中包括：

- onHeapStorageMemoryPool = new StorageMemoryPool(this, MemoryMode.ON_HEAP) // 堆内存储内存
- offHeapStorageMemoryPool = new StorageMemoryPool(this, MemoryMode.OFF_HEAP) // 堆外存储内存
- onHeapExecutionMemoryPool = new ExecutionMemoryPool(this, MemoryMode.ON_HEAP) // 堆内执行内存
- offHeapExecutionMemoryPool = new ExecutionMemoryPool(this, MemoryMode.OFF_HEAP) // 堆外执行内存

execution memory指用于shuffle、join、sort、aggregation的内存，storage memory指用于cache和集群中传递数据的内存。

MemoryManager的主要实现类为UnifiedMemoryManager：

- 加强了执行和存储内存之间的软边界，二者可以向彼此借用内存。
- 只要执行内存是空闲的，就可以借用为存储内存，直到重申为执行空间。当重申为执行空间时，缓存的block会从内存中清除，直到有足够的内存被释放来满足执行内存请求。
- 执行可以借用存储内存。然而，存储不能清除已借用为执行空间的内存，因为实现起来比较复杂。因此如果执行任务消耗了大部分的存储内存，缓存block可能会失败。


### ShuffleWriter
输出ShuffleMapTask的结果到内存或磁盘。主要实现类有：SortShuffleWriter、UnsafeShuffleWriter、BypassMergeSortShuffleWriter。

- SortShuffleWriter利用ExternalSorter进行数据的排序和输出，SortShuffle。
	- ExternalSorter使用Partitioner将key分区到不同partition，之后在partition内对key进行排序。在一个单独的文件中包括多个partition，每个partition位于文件的一个区间内，用于获取shuffle数据。
	- spark默认开启的是Sort Shuffle。
- UnsafeShuffleWriter利用ShuffleExternalSorter进行数据的排序和输出。
	- ShuffleExternalSorter将数据被追加到data page。当所有数据都被插入或达到当前线程都shuffle memory上限，内存中数据会被排序。排序后的数据写入到一个单独的output文件中，文件格式与SortShuffleWriter输出的文件格式相同，每个partition作为一个单独的序列化的、压缩的流进行写入。
	- 与ExternalSorter不同的是，ShuffleExternalSorter不会对spill文件进行merge。merge由它的调用者UnsafeShuffleWriter执行，使用一个特殊的merge流程，避免流多余的序列化/反序列化。	
	- 直接对序列化的数据进行排序，spill的过程也不需要反序列化即可完成。
	- 可以申请堆内内存或利用JDK Unsafe API申请堆外内存。
- BypassMergeSortShuffleWriter实现了基于排序的shuffle的hash式shuffle后退路径，HashShuffle。输出数据到不同的文件，每个partition一个文件，然后将这些文件拼接为一个单独的文件，每个reducer使用一个区间。数据不存在内存中，直接向文件中追加。数据可通过IndexShuffleBlockResolver消费。
	- 当没有map端的combine操作，且partition数目（参数spark.shuffle.sort.bypassMergeThreshold，默认为200）不超过一定阈值时，才选用这个writer。因为它同时打开所有partition的文件流，partition过多的话IO压力太大。



### 不同层级的内存管理

- Writer层次：MemoryConsumer（如ExternalSorter）执行存储时，在需要的情况下，向TaskMemoryManager申请内存。
- Task层次：由TaskMemoryManager管理task内所有内存。需要时向MemoryManager申请更多内存。
- JVM层次：由MemoryManager管理所有内存。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_task_memorymanager.drawio.png)

而BlockManager负责存取所有block，包括堆内内存、堆外内存、磁盘、远程系统中的block。

### MapOutputTrackerMaster

Driver端的类，负责跟踪stage中map输出的结果所在位置。

DAGScheduler使用这个类注册map的output状态，以及查找统计信息用于实现基于位置的reduce任务调度。
ShuffleMapStage使用这个类来跟踪可用/不可用的输出，以决定哪些task需要执行。

### ShuffleBlockFetcherIterator

获取多个block的iterator。从本地BlockManager获取本地block。使用BlockTransferService获取远程block。

initialize时，从MapOutputTracker获取所有本地block。

### BlockStoreShuffleReader

BlockStoreShuffleReader向其他节点的block store发起请求，获取shuffle task的结果block。利用ShuffleBlockFetcherIterator获取本地或远程的block的Iterator。


## 主要流程

### 架构
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_blockmanager_structure.drawio.png)

BlockManager位于SparkEnv中，driver和executor端进行SparkEnv初始化时创建BlockManager。不同之处是，driver端会创建BlockManagerMasterEndpoint，跟踪BlockManager及block信息，executor会向它发送注册、查询等RPC请求。executor端利用BlockManagerMaster存储BlockManagerMasterEndpoint的RpcEndpointRef，因此可以向BlockManagerMasterEndpoint发送信息。

BlockManager中，MemoryStore和DiskStore负责本地数据的存取，BlockTransferService负责远程数据的获取及传输数据给远程executor。

### task执行时获取输入数据

入口从TaskRunner.run() -> Task.run()开始，计算RDD。利用RDD.iterator()获取RDD，对于已经计算的RDD利用BlockManager.getOrElseUpdate() -> BlockManager.get()，从本地或远程获取block。首先从本地获取，获取不到时从远程获取。

### task执行结果的存储

TaskRunner.run()，得到task执行结果。如果结果超过一定大小就通过BlockManager存储到内存或磁盘，将blockId返回给driver；否则，直接将结果返回给driver。

### ShuffleMapTask执行及结果输出

入口为ShuffleMapTask.runTask。RDD计算完成后，通过ShuffleWriteProcessor获取不同的ShuffleWriter，利用ShuffleWriter.write()进行结果的输出。

首先分析SortShuffleWriter的write过程：

**insert**

利用ExternalSorter.insertAll()，将数据存储到内存中的PartitionedAppendOnlyMap或PartitionedPairBuffer结构。

如果内存中数据超过一定阈值，进行spill操作。spill是指，将内存中的数据spill到磁盘上。在spill之前，会尝试向TaskMemoryManager申请更多内存。

**sort&write**

如果数据都在内存中，则对内存中的数据进行partition内排序，然后按partition输出到文件中。

如果有数据spill到磁盘上，则对磁盘和内存中的数据进行merge-sort。merge过程中，根据需要进行aggregate或sort，得到每个partition的数据。merge后按partition输出到文件。

所以每个map任务最后只会生成一个磁盘文件，解决了spark早期版本中一个map输出多个bucket文件过多的问题和磁盘IO成为性能瓶颈的问题。

**commitAllPartitions**
利用IndexShuffleBlockResolver原子性地提交数据和metadata文件，使用已有数据或用新的数据代替旧的数据。

有两种类型的metadata文件：1.index文件，包含每个block的offset，以及最终的offset。2.checksum文件，包含每个block的checksum。

### Task结果存储&上报

shuffle task执行完成后，得到结果大小若小于一定阈值则直接将结果上报给DriverEndpoint；若结果较大，则通过BlockManager存在executor端，然后将blockId上报给DriverEndpoint，之后下游通过blockId来向上游executor请求数据。

### 下游获取ShuffleMapTask的结果

Task的计算，会转换为一层层RDD的迭代计算，不同的数据转换对应不同的RDD.compute()逻辑。常见的RDD有FileScanRDD、MapPartitionsRDD、ShuffleRDD等，可以看到它们的compute逻辑各不相同。

下游获取shuffle数据，是通过ShuffleRDD.compute()得到的。进而通过BlockStoreShuffleReader获取本地和远程RDD数据的Iterator，进行之后的计算。

**如何知道每个远程block的位置？**


## 静态内存管理与统一内存管理

Spark 为存储内存和执行内存的管理提供了统一的接口——MemoryManager，同一个 Executor 内的任务都调用这个接口的方法来申请或释放内存。

MemoryManager中内存管理的主要方法如下：

```
//申请存储内存
def acquireStorageMemory(blockId: BlockId, numBytes: Long, memoryMode: MemoryMode): Boolean

//申请展开内存
def acquireUnrollMemory(blockId: BlockId, numBytes: Long, memoryMode: MemoryMode): Boolean

//申请执行内存
def acquireExecutionMemory(numBytes: Long, taskAttemptId: Long, memoryMode: MemoryMode): Long

//释放存储内存
def releaseStorageMemory(numBytes: Long, memoryMode: MemoryMode): Unit

//释放执行内存
def releaseExecutionMemory(numBytes: Long, taskAttemptId: Long, memoryMode: MemoryMode): Unit

//释放展开内存
def releaseUnrollMemory(numBytes: Long, memoryMode: MemoryMode): Unit
```

MemoryManager的具体实现上，在spark 1.6之前采用静态内存管理（3.0.0版本中已经移除），1.6版本之后采用统一内存管理（UnifiedMemoryManager）。

### 静态内存管理

在 Spark 最初采用的静态内存管理机制下，存储内存、执行内存和其他内存的大小在 Spark 应用程序运行期间均为固定的，但用户可以应用程序启动前进行配置，堆内内存的分配如图所示：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_static_memorymanager_onheap.png)

堆外的空间分配较为简单，只有存储内存和执行内存:

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_static_memorymanager_offheap.png)

静态内存管理机制实现起来较为简单，但如果用户不熟悉 Spark 的存储机制，或没有根据具体的数据规模和计算任务或做相应的配置，很容易造成"一半海水，一半火焰"的局面，即存储内存和执行内存中的一方剩余大量的空间，而另一方却早早被占满，不得不淘汰或移出旧的内容以存储新的内容。

### 统一内存管理

Spark 1.6 之后引入的统一内存管理机制，与静态内存管理的区别在于存储内存和执行内存共享同一块空间，可以**动态占用**对方的空闲区域。

堆内内存：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_unified_memorymanager_onheap.png)

堆外内存：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_unified_memorymanager_offheap.png)

堆外内存中，存储和执行内存也可以相互借用。

其中最重要的优化在于动态占用机制，其规则如下：

    设定基本的存储内存和执行内存区域（spark.storage.storageFraction 参数），该设定确定了双方各自拥有的空间的范围
    双方的空间都不足时，则存储到硬盘；若己方空间不足而对方空余时，可借用对方的空间;（存储空间不足是指不足以放下一个完整的 Block）
    执行内存的空间被对方占用后，可让对方将占用的部分转存到硬盘，然后"归还"借用的空间
    存储内存的空间被对方占用后，无法让对方"归还"，因为需要考虑 Shuffle 过程中的很多因素，实现起来较为复杂
    
凭借统一内存管理机制，Spark 在一定程度上提高了堆内和堆外内存资源的利用率，降低了开发者维护 Spark 内存的难度，但并不意味着开发者可以高枕无忧。譬如，所以如果存储内存的空间太大或者说缓存的数据过多，反而会导致频繁的全量垃圾回收，降低任务执行时的性能，因为缓存的 RDD 数据通常都是长期驻留内存的。    

## 参考

[Spark中几种ShuffleWriter的区别](https://cloud.tencent.com/developer/article/1466602)