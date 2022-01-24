### 简介
Zookeeper是一个分布式数据一致性框架，它保证所有Server端提供的数据是一致的，这就涉及到数据的读写了，那么Server端的数据具体是怎么存储的呢？
实际上，Server端的数据读取是从内存中读，即所有的节点都要加载到内存中（因此能加载的数据并不多）。
另外，为了机器重启恢复，还需要对zxid、数据、操作记录进行持久化，因此有了Snapshot、TxnLog文件。

### 核心类
Record -- zookeeper-jute是zk中的序列化组件，Record是jute中的序列化反序列化的基础接口。server端的snapshot和transaction log都是利用jute进行序列化反序列化。
ZKDatabase -- 表示server完整的内存数据，包括sessions、datatreee、commit logs。server启动时，从snapshot和transaction log文件反序列化并重播transaction，得到ZKDatabase。
DataTree -- 维护树状结构，包括树状节点结构、dataWatches（监听节点变化）、childWatches（监听子节点变化）。
NodeHashMap -- 可以管理节点hash值的Map。实现类NodeHashMapImpl基于ConcurrentHashMap实现，为整个map（即整棵树计算）维护一个hash值，当发生节点变更时重算此hash值，用于快速判断数据是否保持同步（learner和leader间数据同步？）。
IWatchManager -- 节点监听管理。
DigestCalculator -- 计算节点的digest。
AdHash -- 维护整棵树的hash值。
PathTrie -- 维护节点的字典树，用于限额管理。
FileTxnSnapLog -- 管理SnapShot和TxnLog的上层类。
TxnLog -- Transaction Log读写。
SnapShot -- Snapshot文件读写。
QuorumPeerMain -- 集群模式下的Server。
ZooKeeperServerMain -- Standalone模式下的Server。

#### 1. FileTxnLog
Transaction表示一次操作，包括节点的增删改查、节点的监听、ACL控制、session管理等操作。
FileTxnLog表示transaction log文件，文件格式为：
>  1.FileHeader : magic numer, version, dbid
>  2.TxnList: Txn格式为[checksum, Txnlen, TxnHeader, Record]， TxnHeader中包括sessionId、cxid、zxid、time、type，Record中是具体的transaction信息。
>  3.ZeroPad填充文件

#### 2. FileSnap

FileSnap表示Snapshot文件管理。每个snapshot文件以zxid和timestamp进行唯一性标志。
文件内容包括：
> 1.FileHeader：Magic, Version, dbId
> 2.Sessions: session count, session list。session内容包括sessionId和sessionTimeout。
> 3.节点树datatree：acl，节点。

**利用哪些手段保证文件的准确性？**

1. 利用CheckedOutputStream校验文件checksum，保证数据的完整性；SnapStream类中使用jdk自带的Adler32类进行数据流的checksum计算。
2. 如果想要snapshot之后立即刷新到磁盘，可以使用zk自带的AtomicFileOutputStream（来源于HDFS），它在数据全部写入并flush到磁盘后才更新到目标文件。实现方式是：先写入到一个tmp文件，然后rename到目标文件名。

#### 3. DataTree
DataTree在内存中记录节点树，每个节点对应其中NodeHashMap的一个key。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/zk/zk_storage_datatree.png)


### 主要流程

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/zk/zk_storage_leader.png)

可以看出，存储相关的操作都由ZKDatabase负责，包括在内存中维护节点树、维护sessions、读写Snapshot、读写TransactionLog等。非常的高内聚～

> Server启动时，由ZKDatabase从文件中获取zxid、epoch，用于之后的Leader选举；
> Server成为Leader后，首先进入DISCOVERY阶段，从SnapshotFile、TxnLogFile中加载DataTree、Snapshot、TransactionLog、Session到内存中。此时从Snapshot文件恢复节点树，然后按照TransactionLog重播每次操作，即可恢复现场。之后为每个Learner开启一个LearnerHandler线程，开始同步数据，并将同步进度通知Leader；
> 超过半数的Learner ack新的epoch后，进入SYNCHRONIZATION阶段；
> 超过半数的Learner ack新的Leader后，启动ZookeeperServer，包括开启SessionTracker、RequestProcessor等，之后进入BROADCAST阶段；
> BROADCAST阶段，Leader会定时检查是否有过半的Learner保持alive并与自己保持同步。若不满足过半条件，则执行shutdown操作。



### PathTrie 字典树


Trie树即字典树/前缀树，PathTrie基于节点的路径实现了一个字典树，用于限额管理。注意，整个节点树并不是基于PathTrie，而是由NodeHashMap存储。

节点“/ab/bc/cf”在PathTrie树中位置为：
![585a8b874b3e6cf571d707ac9f5d2a39.png](evernotecid://BAB0966A-E6E2-4E29-A334-B0A11405B828/appyinxiangcom/9031308/ENResource/p600)


PathTrie主要内容：
```
/** Root node of PathTrie */
 private final TrieNode rootNode;

 private final ReadWriteLock lock = new ReentrantReadWriteLock(true);

 private final Lock readLock = lock.readLock();

 private final Lock writeLock = lock.writeLock();

 static class TrieNode {

     final String value;
     final Map<String, TrieNode> children;
     //标志一个节点是否special，如是否有子节点
     boolean property;
     TrieNode parent;
     
     //省略成员方法。。
 }
 
/**
* 寻找最大匹配前缀
* Return the largest prefix for the input path. All paths are relative to the
* root node.
*
* @param path the input path
* @return the largest prefix for the input path
*/
public String findMaxPrefix(final String path) {
 Objects.requireNonNull(path, "Path cannot be null");

 final String[] pathComponents = split(path);

 readLock.lock();
 try {
     TrieNode parent = rootNode;
     TrieNode deepestPropertyNode = null;
     for (final String element : pathComponents) {
         parent = parent.getChild(element);
         if (parent == null) {
             LOG.debug("{}", element);
             break;
         }
         if (parent.hasProperty()) {
             deepestPropertyNode = parent;
         }
     }

     if (deepestPropertyNode == null) {
         return "/";
     }

     final Deque<String> treePath = new ArrayDeque<>();
     TrieNode node = deepestPropertyNode;
     while (node != this.rootNode) {
         treePath.offerFirst(node.getValue());
         node = node.parent;
     }
     return "/" + String.join("/", treePath);
 } finally {
     readLock.unlock();
 }
}
```