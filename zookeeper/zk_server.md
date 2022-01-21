

created on 2021-05-10

# 几个问题：

- [ ]  磁盘存储了哪些内容？如何存储和恢复的？
- [ ]  一次数据更新是如何执行的？
- [ ]  节点监听是如何添加的？节点更新如何触发监听事件的？
- [ ]  Leader选举如何进行？
- [ ]  网络连接是如何进行的？

# 入口：QuorumPeerMain类

启动入口：zkServer.sh

主类：QuorumPeerMain

```java
//集群模式 or standalone 模式运行
if (args.length == 1 && config.isDistributed()) {
    runFromConfig(config);
} else {
    LOG.warn("Either no config or no quorum defined in config, running in standalone mode");
    // there is only server in the quorum -- run as standalone
    ZooKeeperServerMain.main(args);
}
```

## Standalone模式

### 从Standalone测试类开始 ZooKeeperServerMainTest

创建并启动Server，然后启动Client，建立连接，调用create方法。

```java
/**
 * Verify the ability to start a standalone server instance.
 */
@Test
public void testStandalone() throws Exception {
    ClientBase.setupTestEnv();

    final int CLIENT_PORT = PortAssignment.unique();
		
		//利用MainThread启动Server
    **MainThread main = new MainThread(CLIENT_PORT, true, null);**
    main.start();

    assertTrue(ClientBase.waitForServerUp("127.0.0.1:" + CLIENT_PORT, CONNECTION_TIMEOUT),
            "waiting for server being up");

    clientConnected = new CountDownLatch(1);
		//Client连接Server，并进行create操作
    **ZooKeeper zk = new ZooKeeper("127.0.0.1:" + CLIENT_PORT, ClientBase.CONNECTION_TIMEOUT, this);**
    assertTrue(clientConnected.await(CONNECTION_TIMEOUT, TimeUnit.MILLISECONDS), "Failed to establish zkclient connection!");

    **zk.create("/foo", "foobar".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);**
    assertEquals(new String(zk.getData("/foo", null, null)), "foobar");
    zk.close();

    main.shutdown();
    main.join();
    main.deleteDirs();

    assertTrue(ClientBase.waitForServerDown("127.0.0.1:" + CLIENT_PORT, ClientBase.CONNECTION_TIMEOUT),
            "waiting for server down");
}

public MainThread(int clientPort, Integer secureClientPort, boolean preCreateDirs, File tmpDir, String configs) throws IOException {
            super("Standalone server with clientPort:" + clientPort);
            this.tmpDir = tmpDir;
            confFile = new File(tmpDir, "zoo.cfg");  //准备临时cfg文件

            FileWriter fwriter = new FileWriter(confFile);
            fwriter.write("tickTime=2000\n");
            fwriter.write("initLimit=10\n");
            fwriter.write("syncLimit=5\n");
            if (configs != null) {
                fwriter.write(configs);
            }

            dataDir = new File(this.tmpDir, "data");
            logDir = new File(dataDir.toString() + "_txnlog");
            if (preCreateDirs) {
                //...
                ClientBase.createInitializeFile(logDir);
            }

            String normalizedDataDir = PathUtils.normalizeFileSystemPath(dataDir.toString());
            String normalizedLogDir = PathUtils.normalizeFileSystemPath(logDir.toString());
            fwriter.write("dataDir=" + normalizedDataDir + "\n");
            fwriter.write("dataLogDir=" + normalizedLogDir + "\n");
            fwriter.write("clientPort=" + clientPort + "\n");

            if (secureClientPort != null) {
                fwriter.write("secureClientPort=" + secureClientPort + "\n");
            }
            fwriter.flush();
            fwriter.close();

            main = new TestZKSMain();
        }

        public void run() {
            String[] args = new String[1];
            args[0] = confFile.toString();
            try {
								//启动Server
                **main.initializeAndRun(args);**
            } catch (Exception e) {
                // test will still fail even though we just log/ignore
                LOG.error("unexpected exception in run", e);
            }
        }
				//...
    }

		//Server
    public static class TestZKSMain extends ZooKeeperServerMain {

        public void shutdown() {
            super.shutdown();
        }
    }
```

### Standalone模式启动流程

Standalone模式下，从ZookeeperServerMain开始执行，包括读取配置、注册Log4jBean到JMX、启动等，主要逻辑如下。

```java
public void runFromConfig(ServerConfig config) throws IOException, AdminServerException {
    FileTxnSnapLog txnLog = null;
    try {
        try {
            metricsProvider = MetricsProviderBootstrap.startMetricsProvider(
                config.getMetricsProviderClassName(),
                config.getMetricsProviderConfiguration());
        } catch (MetricsProviderLifeCycleException error) {
            throw new IOException("Cannot boot MetricsProvider " + config.getMetricsProviderClassName(), error);
        }
        ServerMetrics.metricsProviderInitialized(metricsProvider);
        ProviderRegistry.initialize();
        // Note that this thread isn't going to be doing anything else,
        // so rather than spawning another thread, we will just call
        // run() in this thread.
        // create a file logger url from the command line args
				//读取快照和日志文件
        **txnLog = new FileTxnSnapLog(config.dataLogDir, config.dataDir);**
        JvmPauseMonitor jvmPauseMonitor = null;
        if (config.jvmPauseMonitorToRun) {
            jvmPauseMonitor = new JvmPauseMonitor(config);
        }
        **final ZooKeeperServer zkServer = new ZooKeeperServer(jvmPauseMonitor, txnLog, config.tickTime, config.minSessionTimeout, config.maxSessionTimeout, config.listenBacklog, null, config.initialConfig);**
        txnLog.setServerStats(zkServer.serverStats());

        // Registers shutdown handler which will be used to know the
        // server error or shutdown state changes.
        final CountDownLatch shutdownLatch = new CountDownLatch(1);
        zkServer.registerServerShutdownHandler(new ZooKeeperServerShutdownHandler(shutdownLatch));

        // Start Admin server
        adminServer = AdminServerFactory.createAdminServer();
        adminServer.setZooKeeperServer(zkServer);
        **adminServer.start();**

        boolean needStartZKServer = true;
        if (config.getClientPortAddress() != null) {
            cnxnFactory = ServerCnxnFactory.createFactory();
            cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), false);
            **cnxnFactory.startup(zkServer);**
            // zkServer has been started. So we don't need to start it again in secureCnxnFactory.
            needStartZKServer = false;
        }
        if (config.getSecureClientPortAddress() != null) {
            secureCnxnFactory = ServerCnxnFactory.createFactory();
            secureCnxnFactory.configure(config.getSecureClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), true);
            secureCnxnFactory.startup(zkServer, needStartZKServer);
        }

        containerManager = new ContainerManager(
            zkServer.getZKDatabase(),
            zkServer.firstProcessor,
            Integer.getInteger("znode.container.checkIntervalMs", (int) TimeUnit.MINUTES.toMillis(1)),
            Integer.getInteger("znode.container.maxPerMinute", 10000),
            Long.getLong("znode.container.maxNeverUsedIntervalMs", 0)
        );
        containerManager.start();
        ZKAuditProvider.addZKStartStopAuditLog();

        // Watch status of ZooKeeper server. It will do a graceful shutdown
        // if the server is not running or hits an internal error.
        shutdownLatch.await();

        shutdown();

        if (cnxnFactory != null) {
            cnxnFactory.join();
        }
        if (secureCnxnFactory != null) {
            secureCnxnFactory.join();
        }
        if (zkServer.canShutdown()) {
            zkServer.shutdown(true);
        }
    } catch (InterruptedException e) {
        // warn, but generally this is ok
        LOG.warn("Server interrupted", e);
    } finally {
        //。。
    }
}
```

主要启动流程可总结为：

![132d74149a5500a3a465940484df9dfd.png](evernotecid://BAB0966A-E6E2-4E29-A334-B0A11405B828/appyinxiangcom/9031308/ENResource/p580)@w=400


### ZooKeeperServerMain

核心方法是runFromConfig，启动一个Standalone模式的Server。

### ZooKeeperServer

Standalone模式的ZookeeperServer。

建立连接及request处理流程：

![c7ab07bb8365a72b409a94bf9087b48d.png](evernotecid://BAB0966A-E6E2-4E29-A334-B0A11405B828/appyinxiangcom/9031308/ENResource/p581)@w=400


![a1673119fccd64a2899310d6e4b75bbb.png](evernotecid://BAB0966A-E6E2-4E29-A334-B0A11405B828/appyinxiangcom/9031308/ENResource/p582)@w=500


### AdminServer

执行zk四字命令的入口，常见的有lead、mntr、dirs等。

实现类为**JettyAdminServer**，访问AdminServer提供的http接口即可查询zk集群当前状态、当前leader信息、监控集群状态、添加对某个路径的watch等。

调用ZookeeperServer实现command的执行。

### ServerCnxnFactory

主要实现类有NettyServerCnxnFactory、NIOServerCnxnFactory等。

- [ ]  Netty & NIO实现的区别？

### FileTxnSnapLog

zk的trxnlog及snapshot相关的处理，包括从文件中加载snapshot和生成snapshot等。主要成员有：

```java
//the directory containing the
//the transaction logs
final File dataDir;
//the directory containing the
//the snapshot directory
final File snapDir;
TxnLog txnLog;    //事务日志
SnapShot snapLog;   //快照数据
private final boolean autoCreateDB;
private final boolean trustEmptySnapshot;
```

### ZKDatabase

在内存中维护的zk Server状态，包括session、datatree、transaction log、snapshot等。

```java
protected DataTree dataTree; //维护zk的树形结构数据，包括nodes、watcher管理等
protected ConcurrentHashMap<Long, Integer> sessionsWithTimeouts;
protected FileTxnSnapLog snapLog; //管理transaction log及snapshot
protected long minCommittedLog, maxCommittedLog;
```

### ContainerManager

定时检测ContainerZNode，清理Container类型的ZNode。

Container类型ZNode是指，如果该节点没有子节点，该节点就会被删除。

### 核心类

FileTxnSnapLog -- 管理Snapshot和Transaction Log文件
ZooKeeperServer -- Standalone的zk server
AdminServer -- 接受并处理四字命令
ZKDatabase -- 管理数据，包括树形结构、session、snapshot、transaction log等
ServerCnxnFactory -- 网络连接管理
ServerCnxn -- 代表Server与Client间的一个socket

## 集群模式

### 入口QuorumServerMain

启动过程中，启动DatadirCleanupManager 线程，定时运行删除translog和snapshot文件，只保留最新的n个文件。

主要逻辑是创建、启动QuorumPeer。

### 核心QuorumPeer

主要启动流程如下。与Standalone模式server的主要区别在于要进行LeaderElection。

```java
public synchronized void start() {
        if (!getView().containsKey(myid)) {
            throw new RuntimeException("My id " + myid + " not in the peer list");
        }
        //从磁盘文件加载datatreee、snapshot、translog
        loadDataBase();
        //启动网络连接ServerCnxnFactory
        startServerCnxnFactory();
        try {
            //adminServer负责处理四字命令
            adminServer.start();
        } catch (AdminServerException e) {
            LOG.warn("Problem starting AdminServer", e);
            System.out.println(e);
        }
        //开始选举
        startLeaderElection();
        //启动JvmPauseMonitor
        startJvmPauseMonitor();
        //主流程，进行looking/observing/following/leading状态下的处理
        super.start();
    }
}
```

### QuorumVerifier
QuorumVerifier核心方法是containsQuorum(),用于判断一组server是否构成一个quorum，即它们的投票是否能决定一个leader。
```
boolean containsQuorum(Set<Long> set);
```
它的主要实现类为QuorumMaj，以过半原则判断是否构成一个quorum。另外还有实现类QuorumHierarchical，根据分组和权重判断是否构成一个quorum。


### LeaderElection
目前主要为FastLeaderElection。

### QuorumCnxManager
管理server之间的连接，专门用于Leader选举，基于TCP协议实现。为每一对server之间维护有且仅有一个tcp连接（只能由sid较大的一方发起连接），为对端的每个peer维护一个发送队列。
由内部的SendWorker和RecvWorker线程负责发送和接收工作。Listener负责accept其他server发过来的连接请求。

### 选举的核心流程
![51c7e58e4e2d7b7c7e65bf4fffb10d3c.png](evernotecid://BAB0966A-E6E2-4E29-A334-B0A11405B828/appyinxiangcom/9031308/ENResource/p585)@w=500

### 节点的角色转变


### 不同角色的操作：
#### 1.Leader

ZAB协议状态流转：
![a6c4fa7e0fbf16cf8c10475cbe2d7571.png](evernotecid://BAB0966A-E6E2-4E29-A334-B0A11405B828/appyinxiangcom/9031308/ENResource/p587)@w=500


#### 2.Follower
Follower端状态流转：
![1840af944447ca84047d2b46932aa54c.png](evernotecid://BAB0966A-E6E2-4E29-A334-B0A11405B828/appyinxiangcom/9031308/ENResource/p588)@w=500


#### 3.Observer
Observer不参与选举。它负责响应度请求，将写请求转发给Leader或ObserverMaster。

#### ObserverMaster
若开启QuorumPeer.ObserverMaster开关，由Followers代替Leader来同步数据到Observer。
开启后，Follower端创建一个ObserverMaster线程服务Observer。可以减轻Leader的压力，提高Observer的性能。
若开关未开启，Observer直接与Leader连接。




[节点数据读写、快照、日志及Watch相关 — ZKDatabase]((https://app.yinxiang.com/shard/s41/nl/9031308/9d5700be-42ce-4e07-ad9d-58a036cb223a/))

[Server端网络连接]([https://app.yinxiang.com/shard/s41/nl/9031308/9d5700be-42ce-4e07-ad9d-58a036cb223a/](https://app.yinxiang.com/shard/s41/nl/9031308/9d5700be-42ce-4e07-ad9d-58a036cb223a/))