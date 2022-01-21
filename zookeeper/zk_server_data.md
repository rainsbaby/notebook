

## 1.简介

ZKDatabase在内存中维护的zk Server状态，包括session、datatree、transaction log、snapshot等。

```java
protected DataTree dataTree;
protected ConcurrentHashMap<Long, Integer> sessionsWithTimeouts;
protected FileTxnSnapLog snapLog;
```

## 2.DataTree

DataTree维护树形的数据结构。主要包括：

```java
private final NodeHashMap nodes;
private IWatchManager dataWatches;
private IWatchManager childWatches;
private static final String procZookeeper = Quotas.procZookeeper;   //"/zookeeper"目录
private static final String quotaZookeeper = Quotas.quotaZookeeper;  //"/zookeeper/quota"目录
private static final String configZookeeper = ZooDefs.CONFIG_NODE;  //"/zookeeper/config"目录
```

### Node信息NodeHashMap

NodeHashMap维护node的map信息，与一般map的不同之处是它提供了**digest信息**。

digest的作用是快速验证follower/observer的node数据是否与leader保持同步。

实现类NodeHashMapImpl中使用AdHash类来维护整个map的hash值，作为map的digest信息，整个map的digest值基于每个node的digest计算出来。

当map进行put、remove时，同步进行hash值的变更。

### IWatchManager

负责管理每个路径上的watcher，包含path→watcher的map和watcher→path到map，以及WatcherModeManager（作用是什么还没搞明白）。

```java
private final Map<String, Set<Watcher>> watchTable = new HashMap<>();
private final Map<Watcher, Set<String>> watch2Paths = new HashMap<>();
private final WatcherModeManager watcherModeManager = new WatcherModeManager();
```

IWatchManager包含addWatch、removeWatch、triggerWatch等方法。

一个Watcher的生命周期如下：

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/zk/zk_server_watcher.png)



WatcherModeManager：

WatcherMode：

1. STANDARD
2. PERSISTENT
3. PERSISTENT_RECURSIVE

### PathTrie

用于路径的前缀匹配。

- [ ]  具体用于什么时候？

## 3.FileSnap

Snapshot是持久层的接口，主要功能是对DataTree和sessions进行序列化、反序列化。

FileSnap是Snapshot接口的实现类，负责将DataTree和sessions序列化到文件中，同时提供反序列化。

**一个Snapshot文件的主要构成：**

```
1.Header部分，包括magic（“ZKSN”）、version、dbid；
2.Session部分，包括session数目、具体每个session的信息（id+timeout）
3.DataTree部分，包括ACL、所有node信息、root节点信息
4.checksum值
```

**利用哪些手段保证文件的准确性？**

1. 利用CheckedOutputStream校验文件checksum，保证数据的完整性；SnapStream类中使用jdk自带的Adler32类进行数据流的checksum计算。
2. 如果想要snapshot之后立即刷新到磁盘，可以使用zk自带的AtomicFileOutputStream（来源于HDFS），它在数据全部写入并flush到磁盘后才更新到目标文件。实现方式是：先写入到一个tmp文件，然后rename到目标文件名。

## 4.FileTxnLog

TxnLog是读写TransactionLog的持久层接口，有read、append、truncate等方法。

FileTxnLog是TxnLog接口的实现类，负责TxnLog文件的读写。

**一个TxnLog文件的主要构成：**

```
1.Header部分，包括TXNLOG_MAGIC（“ZKLG“）、version、dbid；
2.每个TxnLog记录，包括该条log的checksum、具体的log信息；
一条TxnLog包括header（clientId、cxid、zxid、time、type）、具体record、txnlog的digest摘要信息；
```