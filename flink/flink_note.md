



内存管理

JVM内存管理的不足：

* 有效数据密度低
* 垃圾回收不可控
* OOM问题影响稳定性
* 缓存未命中问题

因为JVM存在诸多问题，所以越来越多的大数据计算引擎选择自行管理JVM内存，如Spark、Flink、HBase，尽量达到C/C++ 一样的性能，同时避免OOM的发生。

Flink自主内存管理，包括内存管理、定制的序列化工具、缓存友好的数据结构和算法、堆外内存等。


每个TaskManager都是一个Java进程，TaskManager为每个Task分配独立的执行线程。


### 状态管理

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink/flink_statebackend_uml.png)

主要有两种类型的StateBackend：

* HashMapStateBackend
* EmbeddedRocksDBStateBackend
