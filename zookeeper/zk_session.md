## 简介
Client与Server建立连接时，由server端创建一个session，server与client都记录该sessionId。Server对session进行管理，包括创建、超时处理、upgrade等。


## 核心类
版本: 3.7.0.
SessionTracker -- Session管理的基础接口，包括创建、删除、check等。
SessionTrackerImpl -- SessionTracker的实现类，利用ExpiryQueue进行session过期关闭处理。
ExpiryQueue -- 利用Map记录Session，实现按超时时间排序的队列功能。
LocalSessionTracker -- 管理local session。local session只能进行读操作。
UpgradeableSessionTracker -- 支持local session的upgrade的SessionTracker。local session进行写操作时，需要upgrade到global session。
LeaderSessionTracker -- Leader端的SessionTracker。
LearnerSessionTracker -- Follower 和 Observer端管理Session。该session可能转发或未转发给leader。
ClientCnxn -- 管理client端的socket io。建立连接后，保存server端返回的sessionId。
ClientCnxnSocketNetty/ClientCnxnSocketNIO -- 负责底层的网络连接
ServerCnxn -- client与server的一个连接，连接建立成功后保存对应的sessionId。


## 主要流程


![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/zk/zk_session_manage.png)

#### 创建Session
1. Client与Server间完成三次握手建立连接；
2. Server端使用Netty管理与Client间网络连接时，由CnxnChannelHandler.channelRead读取client端request，交给NettyServerCnxn进行处理；
3. NettyServerCnxn判断session是否初始化，若没有则提交创建session类型的request，交给RequestProcessor的pipeline进行处理；
4. PrepRequestProcessor处理createSession类型请求，交给sessionTracker.trackSession，记录sessionId、timeout、owner等信息
5. FinalRequestProcessor.finishSessionInit，将sessionId、sessionTimeout返回给Client端

#### 关闭Session
1. SessionTrackerImpl线程从sessionExpiryQueue中获取过期的session，交给ZookeeperServer.expire处理，ZookeeperServer实现了SessionExpirer接口；
2. ZookeeperServer发起closeSession类型request；
3. FinalRequestProcessor处理closeSession请求，首先由ZkDatabase删除相应的节点、添加commitLog。然后执行真正的关闭session操作，即由ServerCnxnFactory通知session对应的ServerCnxn进行关闭操作，涉及channel关闭、数据清理等操作。
4. FinalRequestProcessor处理完成后，通知client端。

#### ExpiryQueue
ExpiryQueue 是按照session过期时间排序的session队列，支持队列的添加、删除、出队等操作，同时支持过期时间的更新。
```
//session -> sessoion的newExpiryTime
private final ConcurrentHashMap<E, Long> elemMap = new ConcurrentHashMap<E, Long>();

//sessoion的newExpiryTime -> session
private final ConcurrentHashMap<Long, Set<E>> expiryMap = new ConcurrentHashMap<Long, Set<E>>();

//下一个过期时间
private final AtomicLong nextExpirationTime = new AtomicLong();
//session过期的检查周期
private final int expirationInterval;

//利用下一个过期时间取出对应的session，用于过期close操作
public Set<E> poll() {
    long now = Time.currentElapsedTime();
    long expirationTime = nextExpirationTime.get();
    if (now < expirationTime) {
        return Collections.emptySet();
    }

    Set<E> set = null;
    long newExpirationTime = expirationTime + expirationInterval;
    if (nextExpirationTime.compareAndSet(expirationTime, newExpirationTime)) {
        set = expiryMap.remove(expirationTime);
    }
    if (set == null) {
        return Collections.emptySet();
    }
    return set;
}
```
#### Session upgrade
Leader和Learner中都可以配置是否开启localSession，localSession只支持读操作，当进行节点创建等写操作是需要进行Session的upgrade，upgrade成global session。
LocalSession只需要记录在当前Server的本地sessionTracker中。而GlobalSession需要通过执行OpCode.createSession类型的request来实现，即在所有server上存储且需要达成一致。