## Clickhouse简介



## Clickhouse架构

Clickhouse是一个**MPP架构的列式存储**数据库。

### MPP（massively parallel processing）架构

MPP 是由多台服务器通过一定的节点互联网络进行连接，协同工作，完成相同的任务，从用户的角度来看是一个服务器系统。每个节点只访问自己的资源，所以是一种完全无共享（Share Nothing）结构。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/clickhouse/mpp_structure.png)

MPP 架构特征：

    任务并行执行;
    数据分布式存储(本地化);
    分布式计算;
    高并发，单个节点并发能力大于 300 用户;
    横向扩展，支持集群节点的扩容;
    Shared Nothing（完全无共享）架构。
    
#### 批处理架构和 MPP 架构

批处理架构（如 MapReduce）与 MPP 架构的异同点，以及它们各自的优缺点是什么呢？

相同点：

首先相同点，批处理架构与 MPP 架构都是分布式并行处理，将任务并行的分散到多个服务器和节点上，在每个节点上计算完成后，将各自部分的结果汇总在一起得到最终的结果。

不同点：

批处理架构和 MPP 架构的不同点可以举例来说：我们执行一个任务，首先这个任务会被分成多个 task 执行，对于 MapReduce 来说，这些 tasks 被随机的分配在空闲的 Executor 上；而对于 MPP 架构的引擎来说，每个处理数据的 task 被绑定到持有该数据切片的指定 Executor 上。 

正是由于以上的不同，使得两种架构有各自优势也有各自缺陷：

批处理的优势：

对于批处理架构来说，**如果某个 Executor 执行过慢，那么这个 Executor 会慢慢分配到更少的 task 执行**，批处理架构有个推测执行策略，推测出某个 Executor 执行过慢或者有故障，则在接下来分配 task 时就会较少的分配给它或者直接不分配。

MPP 的缺陷：

对于 MPP 架构来说，因为 task 和 Executor 是绑定的，如果某个 Executor 执行过慢或故障，将会导致整个集群的性能就会受限于这个故障节点的执行速度(所谓木桶的短板效应)，所以 MPP 架构的最大缺陷就是——**短板效应**。另一点，集群中的节点越多，则某个节点出现问题的概率越大，而一旦有节点出现问题，对于 MPP 架构来说，将导致整个集群性能受限，所以一般实际生产中 MPP 架构的集群节点不易过多。

批处理的缺陷：

任何事情都是有代价的，**对于批处理而言，会将中间结果写入到磁盘中**，这严重限制了处理数据的性能。

MPP 的优势：

**MPP 架构不需要将中间数据写入磁盘**，因为一个单一的 Executor 只处理一个单一的 task，因此可以简单直接将数据 stream 到下一个执行阶段。这个过程称为pipelining，它提供了很大的性能提升。

举个例子：要实现两个大表的 join 操作，对于批处理而言，如 Spark 将会写磁盘 3 次(第一次写入：表 1 根据join key进行 shuffle；第二次写入：表 2 根据join key进行 shuffle；第三次写入：Hash 表写入磁盘)， 而 MPP 只需要一次写入(Hash 表写入)。这是因为 MPP 将 mapper 和 reducer 同时运行，而 MapReduce 将它们分成有依赖关系的 tasks(DAG),这些 task 是异步执行的，因此必须通过写入中间数据共享内存来解决数据的依赖。   

### 列式存储与数据压缩

MergeTree表引擎中，数据按列进行组织，属于同一列的数据会被保存在一起，列与列之间也会由不同的文件分别保存。数据默认使用LZ4算法压缩。

### 向量化执行引擎

向量化执行，可以简单地看作一项消除程序中循环的优化。

为了实现向量化执行，需要利用CPU的SIMD（Single Instruction Multiple Data）指令，即用单条指令操作多条数据。它的原理是在CPU寄存器层面实现数据的并行处理。

### 多样化的表引擎

Clickhouse把存储部分进行了抽象，把存储引擎作为一层独立的接口。

Clickhouse有MergeTree、内存、文件、接口和其他6大类20多种表引擎，用户可以根据实际业务场景选择合适的表引擎。

### 多线程与分布式

### 多主架构

### 数据分片与分布式查询


## MergeTree表引擎

MergeTree是合并树引擎家族中最基础的表引擎，提供了主键索引、数据分区、数据副本和数据采样等基本能力，而家族中其他表引擎则在MergeTree的基础上各有所长。

### MergeTree的存储结构

MergeTree在写入一批数据时，数据总会以数据片段的形式写入磁盘，且数据片段不可修改。为了避免片段过多，会有后台线程**定期合并**这些数据片段。

MergeTree表引擎中的数据是拥有无力存储，数据会按照分区目录的形式保存到磁盘之上。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/clickhouse/clickhouse_mergetree_storage.png)

- partition： 分区目录。
- checksums.txt：校验文件。
- columns.txt： 列信息文件，保存此分区下的列字段信息。
- count.txt：计数文件，记录当前数据分区目录下数据的总行数。
- primary.idx： 一级索引文件，用于存放稀疏索引。
- [Column].bin：数据文件，使用压缩格式存储，默认为LZ4压缩格式，用于存储某一列的数据。由于MergeTree采用列式存储，所以每一个列字段都拥有独立的.bin数据文件，并以列字段命名。
- [Column].mrk：列字段标记文件。是衔接一级索引和数据的桥梁。与.bin文件一一对应，用于记录数据在.bin文件中的偏移量信息。
- partition.dat 和 minmax_[Column].idx：如果使用了分区键，则会额外生成 partition.dat 与 minmax_JoinTime.idx 索引文件。
- skp_idx_ [IndexName].idx 和 skp_idx_[IndexName].mrk3：如果在建表语句中指定了二级索引，则会额外生成相应的二级索引文件与标记文件。

数据分区（partition）与数据分片（shard）：分区是针对本地数据而言，是对数据的纵向切分；分片是将数据分布到多个节点，是横向切分。

### 一级索引

一级索引采用稀疏索引实现。

#### 索引粒度index_granularity

数据以index_granularity的粒度被标记成多个小的区间。一级索引和数据标记的间隔粒度相同（同为index_granularity行），彼此对齐。而数据文件也会依照index_granularity的间隔粒度生成压缩数据块。


### 数据存储

#### 各列独立存储

#### 压缩数据块

.bin文件由多个压缩数据块构成。一个压缩数据块由头信息和压缩数据两部分构成。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/clickhouse/clickhouse_mergetree_storage_zip.png)


## 副本与分片

### 数据副本

只有使用了ReplicatedMergeTree复制表系列引擎，才能应用副本的能力。

ReplicatedMergeTree在MergeTree的基础上增加了分布式协调能力，主要利用Zookeeper实现。在Zookeeper内创建一系列的监听节点，并以此实现多个实例之间的通信。在整个通信过程中，Zookeeper不会设计表数据的传输。

## 参考

[MPP大规模并行处理架构详解](https://cloud.tencent.com/developer/news/835349)