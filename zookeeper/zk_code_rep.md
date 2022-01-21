# ZK code的几个模块

**zookeeper-client**

client端


**zookeeper-compability-tests**

兼容性测试

**zookeeper-contrib**

一些工具，包括log filter等，非核心


**zookeeper-docs**

文档


**zookeeper-it**

测试工具

**zookeeper-jute**

序列化反序列化工具。来自于hadoop Record I/O，性能不是很优秀，但是它不是zk性能的瓶颈，因此没有被替换。

hadoop目前出现了Avro等新的序列化方式。

**zookeeper-metrics-providers**

基于prometheus的metric统计监控

z**ookeeper-recipes**

基于Zk的一些上层工具

1.zookeeper-recipes-election

Leader 选举，逻辑是，每个client请求在dir目录下建立子节点，由子节点id最小者作为leader。
 同时，第二小的id节点监听leader节点，当leader节点被删除时，由第二个节点作为leader。

2.zookeeper-recipes-lock

实现分布式写锁，逻辑是，每个节点都在路径dir下建立子节点，子节点按照id排序，最大的id获得写锁。
同时，第二大的id监听最大的id对应节点变化，当最大的id节点被删除时，重新发起lock请求，即由第二大的id节点获得写锁。

3.zookeeper-recipes-queue

 分布式队列，实现逻辑是，在dir目录下建立子节点，子节点目录排序后形成的队列作为分布式队列

**zookeeper-server**

server端，核心逻辑