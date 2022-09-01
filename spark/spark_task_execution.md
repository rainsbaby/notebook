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

### ShuffleWriter
输出ShuffleMapTask的结果到内存或磁盘。主要实现类有：SortShuffleWriter、UnsafeShuffleWriter、BypassMergeSortShuffleWriter。

- SortShuffleWriter利用ExternalSorter进行数据的排序和输出。
	- ExternalSorter使用Partitioner将key分区到不同partition，之后在partition内对key进行排序。在一个单独的文件中包括多个partition，每个partition位于文件的一个区间内，用于获取shuffle数据。
- UnsafeShuffleWriter利用ShuffleExternalSorter进行数据的排序和输出。
	-ShuffleExternalSorter将数据被追加到data page。当所有数据都被插入或达到当前线程都shuffle memory上限，内存中数据会被排序。排序后的数据写入到一个单独的output文件中，文件格式与SortShuffleWriter输出的文件格式相同，每个partition作为一个单独的序列化的、压缩的流进行写入。
	- 与ExternalSorter不同的是，ShuffleExternalSorter不会对spill文件进行merge。merge由它的调用者UnsafeShuffleWriter执行，使用一个特殊的merge流程，避免流多余的序列化/反序列化。	
- BypassMergeSortShuffleWriter实现了基于排序的shuffle的hash式shuffle后退路径。输出数据到不同的文件，每个partition一个文件，然后将这些文件拼接为一个单独的文件，每个reducer使用一个区间。数据不存在内存中，直接向文件中追加。数据可通过IndexShuffleBlockResolver消费。
	- 当没有map端的combine操作，且partition数目不超过一定阈值时，才选用这个writer。因为它同时打开所有partition的文件流，partition过多的话IO压力太大。

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


## 参考

[Spark中几种ShuffleWriter的区别](https://cloud.tencent.com/developer/article/1466602)