# ZK -- 网络连接（Server端）

## 核心类

ServerCnxn -- 表示一个client和一个server间的网络连接
ServerCnxnFactory -- 负责建立端口监听。
NettyServerCnxnFactory -- 基于Netty的网络管理。其子类CnxnChannelHandler负责进行网络包处理，新的client连接建立时，创建NettyServerCnxn。

### ServerCnxn

ServerCnxn类是一个抽象类，表示**一个client和server间的网络连接**，实现类主要有NettyServerCnxn和NIOServerCnxn。ServerCnxn实现了Watcher接口，即当相应client监听的节点发生变更时，由ServerCnxn负责处理相关事件并通知client。

ServerCnxn主要成员为：

```java
final ZooKeeperServer zkServer;
//将要发送给client的response进行序列化，并放入outgoing queue中等待发送给client
public abstract int sendResponse(ReplyHeader h, Record r, String tag,
                                      String cacheKey, Stat stat, int opCode) throws IOException;
//处理WatchedEvent
public abstract void process(WatchedEvent event);
```

### NettyServerCnxn

NettyServerCnxn是ServerCnxn的实现类，

```java
private final Channel channel;
private CompositeByteBuf queuedBuffer;
private ByteBuffer bb;
private final ByteBuffer bbLen = ByteBuffer.allocate(4);
private long sessionId;
private int sessionTimeout;
private Certificate[] clientChain;
private final NettyServerCnxnFactory factory;
```

- [ ]  jdk NIO中bytebuffer部分

```java
public int sendResponse(ReplyHeader h, Record r, String tag,
                         String cacheKey, Stat stat, int opCode) throws IOException {
    // cacheKey and stat are used in caching, which is not
    // implemented here. Implementation example can be found in NIOServerCnxn.
    if (closingChannel || !channel.isOpen()) {
        return 0;
    }
    ByteBuffer[] bb = serialize(h, r, tag, cacheKey, stat, opCode);
    int responseSize = bb[0].getInt();
    bb[0].rewind();
    sendBuffer(bb);
    decrOutstandingAndCheckThrottle(h);
    return responseSize;
}
```

子类SendBufferWriter负责分块发送reponse到client端，当指定force为true或buffer大小超过2048时发送buffer内容给client。

### ServerCnxnFactory

ServerCnxnFactory是一个抽象类，根据server端配置生成对应的子类NettyServerCnxnFactory和NIOServerCnxnFactory。其主要成员为sessionMap，用于关闭session。

```java
final ConcurrentHashMap<Long, ServerCnxn> sessionMap = new ConcurrentHashMap<Long, ServerCnxn>();
protected ZooKeeperServer zkServer;
//netty引导类
private final ServerBootstrap bootstrap;
private Channel parentChannel;
private final ChannelGroup allChannels = new DefaultChannelGroup("zkServerCnxns", new DefaultEventExecutor());
private final Map<InetAddress, AtomicInteger> ipMap = new ConcurrentHashMap<>();
private InetSocketAddress localAddress;
//Netty的ChannelPipeline中last handler，处理Channel的Active、Inactive、Read、Write等事件
//Active时创建NettyServerCnxn，并将其存入相应channel的AttributeMap中
//Inactive时移除channel，关闭NettyServerCnxn
//channelRead时从channel的AttributeMap中获取相应NettyServerCnxn，并将msg交给NettyServerCnxn处理
CnxnChannelHandler channelHandler = new CnxnChannelHandler();
```

### 子类NettyServerCnxnFactory

负责netty的ServerBootstrap创建，包括bossGroup、workerGroup、ChannelPipeline配置等。

启动流程：

```java
public void startup(ZooKeeperServer zks, boolean startServer) throws IOException, InterruptedException {
    //指定本地ip和监听端口号，启动网络监听
    start();
    setZooKeeperServer(zks);
    if (startServer) {
        //从磁盘加载trxnlog及snapshot
        zks.startdata();
				//启动zk的SessionTracker、RequestProcessor、限流RequestThrottler等
        zks.startup();
    }
}
```

NettyServerCnxnFactory与NettyServerCnxn：


![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/zk/zk_server_netty.png)




[ZK架构图](https://www.processon.com/diagraming/60a38ece5653bb3d82df3401)