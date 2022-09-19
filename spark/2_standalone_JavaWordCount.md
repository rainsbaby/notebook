Spark的Standalone模式


## 简介

我们在本地执行spark源码时，可以执行JavaWordCount例子的main方法，并将master设置为local，这样就以local方式运行。local方式中，所有的调度、执行、资源管理都在一个进程中运行。

尝试过local模式后，我们希望尝试cluster模式，最简单的cluster模式为standalone模式，master作为cluster manager。实际生产环境中，使用Yarn等作为cluster manager。


## Standalone模式使用

### 启动Master

	./sbin/start-master.sh

Master类：org.apache.spark.deploy.master.Master



### 启动Worker

	./sbin/start-worker.sh --webui-port 8081 spark://{master_host}:{master_port}

Worker类：org.apache.spark.deploy.worker.Worker


### 提交task

	./bin/spark-submit --class org.apache.spark.examples.JavaWordCount --master spark://{master_host}:{master_port} {path}/spark/examples/target/original-spark-examples_2.12-3.2.1.jar {path}/examples/src/main/resources/people.txt

入口：org.apache.spark.deploy.SparkSubmit.main

通过代码可以看出，默认client部署模式下，通过反射调用JavaWordCount.main方法，执行计算逻辑。

而如果加上参数 “--deploy-mode cluster”，则流程为：

- Client端启动ClientApp，通过ClientEndpoint向Master发送RequestSubmitDriver请求。
- Master收到请求后，选择worker并发送LaunchDriver请求给worker，在worker端启动driver。
- Worker端启动driver，执行JavaWordCount.main方法。

## Standalone模式架构

### Cluster架构

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_structure.png)

### Standalone模式

Standalone 集群中，主要包括Master、Worker、Client端，它们包含的主要组件和交互如下。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_standalone_structure.drawio.png)

Master、Worker都是RpcEndpoint节点，启动后开始接收RPC消息。

上图是在client部署下，driver位于client端。而在cluster部署模式下，driver位于worker中。

主要交互RPC消息：

1. Worker启动时，向Master发送RegisterWorker消息。Master端注册成功后，返回RegisteredWorker消息。
2. Client端提交task时，向Master发送Register消息。Master端注册Application成功后，返回RegisteredApplication消息。StandaloneAppClient是application与standalone cluster manager交互的接口，它通过内部类ClientEndpoint与Master进行交互。
3. Master端接收到Application注册或Worker注册后，会进行schedule，即向Worker分配task。此时，发送LaunchExecutor消息到Worker端。
4. Worker端task执行完成后，向driver发送StatusUpdate消息，从而可以进行结果获取等操作。


### Master

Standalone集群中，Master作为ClusterManager，负责管理Worker、Application、Driver，负责整个集群中资源的统一管理和分配，接收Worker、Driver的注册、更新状态、心跳信息等，也接收Application的注册。

Master继承了RpcEndpoint类，通过receive等方法接收并处理消息。

实现了LeaderElectable接口，启动后开始参与Leader选举。因此可以启动多个Master，只有一个Master是激活的，其他的都处于standby状态。
Master内部有一个JettyServer，用于查看所有worker和application。

#### PersistenceEngine

Master的状态持久化，用于master的故障恢复。

1. addApplication和addWorker方法要在完成app/worker注册之前调用。
2. removeApplication和removeWorker 可以在任何时候调用。

主要实现类有FileSystemPersistenceEngine和ZooKeeperPersistenceEngine等，分别利用文件系统和zookeeper进行持久化。

### Worker

Worker负责真正执行task，并上报task执行状态。Worker启动时，向Master发送注册消息注册到Master上。

Worker也继承了RpcEndpoint，通过receive等方法接收并处理消息。

Worker端launch executor时，启动CoarseGrainedExecutorBackend进程。CoarseGrainedExecutorBackend继承了RpcEndpoint，接收launch task消息，交给Executor处理。

Executor管理多个task的执行，每个task在一个线程中执行。

### Driver

Driver端启动时，由StandaloneAppClient向Master发送RegisterApplication消息，进行注册。

DriverEndpoint继承了RpcEndpoint，与Master和Worker交互，向Worker发送LaunchTask消息启动task。


### Master、Worker、Driver端的主要交互

下图为Master、Worker、Driver端详细的交互，省去了RPC消息传递环节。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/spark/spark_standalone_JavaWordCount_flow.drawio.png)




## High Availability

默认情况下，standalone集群可以处理Worker的故障，将工作转移到其他Worker。一个master会产生单点故障：如果Master崩溃了，就不能创建新的应用。为了规避这个问题，有两个高可用性方案。

- 基于zookeeper实现master的leader选举和状态恢复。
- 基于本地文件系统实现master的恢复，在master down掉后重启master进行恢复。


## 问题记录

### java.lang.ClassNotFoundException: com.google.common.cache.CacheLoader

实际是有Guava这个jar的，但是是provided的，所以需要在debug选项中添加配置 “Add dependencies with "provided" scope to classpath”。



### 添加spark-version-info.properties文件

执行：
	 build/spark-build-info core/target/extra-resources 3.2.1-SNAPSHOT
	 
 
## 参考
 [Spark Standalone Mode](https://spark.apache.org/docs/3.2.1/spark-standalone.html)