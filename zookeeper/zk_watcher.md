# ZK -- Watch机制

## 简介
ZK的Watch机制是一种注册监听机制。client端向server端发起监听请求，注册针对某个节点的变化。当节点或其子节点发送变化时，server端通知client，此时client可以获取节点上的最新数据，不需要定时轮询。

## 核心类
版本: 3.7.0.
IWatchManager -- Server端管理Watch的接口，管理path与Watcher间的对应关系，并在path变化时通知相应的Watcher。
WatchManager -- IWatchManager的实现类
WatchManagerOptimized -- IWatchManager的实现类，WatchManager类的优化版本。优化手段：1.使用HashSet和BitSet存储Watcher，以实现内存使用量和时间复杂度之间的平衡；2.使用ReadWriteLock代替Synchronized降低锁开销；3.使用WatcherCleaner定时清理dead Watcher。
Watcher -- Server端处理WatchEvent的Handler接口。通过向WatchManager注册Watcher作为callback，即可在节点变化时进行相应的处理。
ZKDatabase -- 在内存中维护server数据，包括session、树形结构数据和commit log。
DataTree -- 树形数据，包括path到节点的映射，以及WatchManager负责管理Watcher。
ServerCnxn -- 负责网络连接，实现类有NettyServerCnxn和NIOServerCnxn。
RequestThrottler -- 流量控制。

### IWatchManager

负责管理每个路径上的watcher，包含path→watcher的map和watcher→path到map，以及WatcherModeManager（作用是什么还没搞明白）。

```java
private final Map<String, Set<Watcher>> watchTable = new HashMap<>();
private final Map<Watcher, Set<String>> watch2Paths = new HashMap<>();
private final WatcherModeManager watcherModeManager = new WatcherModeManager();
```

IWatchManager包含addWatch、removeWatch、triggerWatch等方法。

## 主要流程
Server端和Client端主要交互流程如下。client端逻辑比较简单，主要是网络连接。server端需要存储、管理path与watcher的对应关系，并在节点发生变更时触发watcher。
左边是对节点添加watch的过程：
* Client端向Server发起添加watch请求；
* ServerCnxn接收到请求，交给zookeeperServer；
* zookeeperServer中有RequestThrottler进行限流，之后将request交给requestProcessor的pipeline进行处理；
* FinalRequestProcessor将addWatch请求交给zkdatabase、datatree进行处理；
* 最终由watchManager为path添加对应的watch，这里设置Watcher（回调对象）为ServerCnxn。

右边为watcher的调用链路：
* Client端向Server发起节点更新、删除、添加子节点等请求；
* ServerCnxn接收到请求，交给zookeeperServer处理；
* zookeeperServer中有RequestThrottler进行限流，之后将request交给requestProcessor的pipeline进行处理；
* FinalRequestProcessor将addWatch请求交给zkdatabase、datatree进行处理；
* DataTree进行节点更新/删除等操作后，判断该节点上由没有相应的watch；
* 触发相应的watcher即ServerCnxn，由ServerCnxn处理WatchEvent。ServerCnxn的主要操作是，通知client端。
* 之后client可以向Server发起查询请求，获取最新的节点数据。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/zk/zk_watcher.png)



## WatchManagerOptimize中BitMap & BitSet
WatchManagerOptimized中使用Bitmap存储Watcher，使用ConcurrentHashMap<String, BitHashSet> pathWatches存储path与Bitmap中bit的关系，从而将path与Watcher关联起来。Bitmap使用HashSet和BitSet来存储Watcher->bit 和bit->Watcher映射关系。主要成员如下：

```
//Watcher->bit
private final Map<T, Integer> value2Bit = new HashMap<T, Integer>();
//bit->Watcher
private final Map<Integer, T> bit2Value = new HashMap<Integer, T>();

//用于存储被remove的bit，便于之后回收使用
private final BitSet freedBitSet = new BitSet();
//用于生成自增的bit
private Integer nextBit = Integer.valueOf(0);
```

在remove时，回收到freedBitSet中：
```
public int remove(T value) {
    /*
     * remove only called once when the session is closed, so use write
     * lock directly without checking read lock.
     */
    rwLock.writeLock().lock();
    try {
        Integer bit = value2Bit.get(value);
        if (bit == null) {
            return -1;
        }
        value2Bit.remove(value);
        bit2Value.remove(bit);
        freedBitSet.set(bit); //回收
        return bit;
    } finally {
        rwLock.writeLock().unlock();
    }
}

```

添加时，先到回收存储freedBitSet中查找有没有已回收的bit，如果有则使用；没有则利用nextBit自增得到新的bit。
```
public Integer add(T value) {
    Integer bit = getBit(value);
    if (bit != null) {
        return bit;
    }

    rwLock.writeLock().lock();
    try {
        bit = value2Bit.get(value);
        if (bit != null) {
            return bit;
        }
        bit = freedBitSet.nextSetBit(0);
        if (bit > -1) {
            freedBitSet.clear(bit);
        } else {
            bit = nextBit++;
        }

        value2Bit.put(value, bit);
        bit2Value.put(bit, value);
        return bit;
    } finally {
        rwLock.writeLock().unlock();
    }
}
```

测试代码：
```
@Test
public void testBitset(){
    BitMap<Integer> bitMap = new BitMap<>();
    System.out.println("bitset : " + bitMap.add(5));
    System.out.println("bitset : " + bitMap.add(6));
    bitMap.remove(new Integer(5));
    System.out.println("bitset : " + bitMap.add(7));
    System.out.println("bitset : " + bitMap.add(6));
    System.out.println("bitset : " + bitMap.add(8));
}
```

输出：
```
bitset : 0
bitset : 1
bitset : 0  //回收的bit
bitset : 1  //重复添加时，不会改变任何数据
bitset : 2
```
