# 简介
Checkpoint/savepoint机制是Flink中的重要内容，主要是定时保存或手动触发保存所有节点的状态，存储到内存或HDFS等外部存储中。
在全部或部分节点异常重启时，可基于checkpoint进行恢复，或整个任务异常终止、升级时基于savepoint进行恢复，用于保证At Least Once和Exactly Once。


# 核心类
Flink版本 : 1.14 。

JobMaster -- 一个Job的JobManager，每个job在同一时刻有且仅有一个JobMaster。
在HighAvailability配置下，由JobMasterServiceLeadershipRunner参与选举，成为Leader后，启动相应job的JobMaster，并启动job。
其中包括BlobWriter、HeartbeatServices、SlotPoolService、LeaderRetrievalService(ResourceManager相关)、SchedulerNG、JobManagerJobStatusListener等组件。

SchedulerNG -- Job的调度接口，负责job调度、异常处理等。
实现类主要为SchedulerBase，包括ExecutionGraph、ExecutionGraphHandler、OperatorCoordinatorHandler等。

ExecutionGraph -- 控制整个job的data flow的分布式执行，粒度细到每个并行的task、每个中间流及其交互。
实现类主要为DefaultExecutionGraph，主要包括Map<JobVertexID, ExecutionJobVertex> tasks组成的DAG、JobStatusListener、CheckpointCoordinator等。

CheckpointCoordinator -- 负责Checkpoint/Savepoint等的核心类，发送消息到相应task来触发checkpoint的创建，接收task的回应。
包括Map<Long, PendingCheckpoint>、CompletedCheckpointStore(已完成的checkpoint)等。

Execution -- JobManager端一个Subtask/ExecutionVertex的一次执行。
一个Subtask多次执行时(失败恢复/重计算/更新配置等原因)，对应多个Execution。

TaskExecutor -- TaskManager的对应Class。
负责多个Task的执行。

Task -- TaskManager中，一个Subtask的一个并行度的执行。
包括operator执行、输入、输出、与JobManager的交互。
每个Task由一个固定的线程执行。
Task不负责与其他task的交互，也不知道是否是第一次执行/重复执行，这些信息由JobManager维护。

OperatorChain --  表示由一个StreamTask执行的一串Operator。
入口为mainOperator，它负责拉取输入并生产数据push给后续其他operator。

StreamTask --  每个StreamTask执行一个/多个StreamOperator（如连续的map/flatmap/filter的组成operator chain）。
Operator chain在一个线程中同步执行，因此有同样的stream paritition。
Operator chain中有一个head operator和多个chained operator。
有one-input和two-input 类型的head operator。

CheckpointedInputGate -- 基于CheckpointBarrierHandler，处理从InputGate得到的CheckpointBarrier及cancel/end等checkpoint相关Event。

CheckpointBarrierHandler -- 处理接收到的checkpoint barrier。

TwoPhaseCommitSinkFunction -- 
KafkaSink -- 





# 主要流程

**架构**

Checkpoint执行架构如下图所示。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink_checkpoint_architecture.drawio.png)

* JobMaster中CheckpointCoordinator负责调度，通知SourceOperator所在的TaskExecutor开始Checkpoint；
* Source发送Barrier给下游Operator，并执行自己的snapshot；
* 后续Operator收到上游的Barrier后，继续发送Barrier给下游，并执行自己的snapshot；
* 每个Task中Checkpoint执行完成后，向JobMaster上报成功/失败状态。
* CheckpointCoordinator收集所有Task的Checkpoint状态。当所有task执行snapshot成功时，通知各个task；当checkpoint出现异常时，通知各task取消执行checkpoint。


**Checkpoint详细流程**

Checkpoint 执行详细流程如下。
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/flink_checkpoint_flow_detail.png)

* JobMaster启动，创建Scheduler；
* Scheduler创建ExecutionGraph；
* ExecutionGraph创建CheckpointCoordinator，并注册CheckpointCoordinator为JobStatusListener；
* Job启动成功后，CheckpointCoordinator被通知，开始调度Checkpoint；
* CheckpointCoordinator通过Execution，发送Checkpoint RPC给TaskExecutor；
* TaskExecutor触发相应SourceTask的Checkpoint；
* SourceTask处理Checkpoint命令，发送Checkpoint Barrier给下游，并执行自己的snapshot；
* 下游Operator从InputGate中接收Checkpoint Barrier，继续发送Barrier给下游，并执行自己的snapshot；
* Task执行checkpoint时，将异步部分任务交给AsyncCheckpointRunnable处理；AsyncCheckpointRunnable判断tasksnapshot完成后，上报给TaskStateManager；
* TaskStateManager发送RPC给JobMaster，acknowledgeCheckpoint上报checkpoint状态。
* JobMaster接收acknowledgement，转发给scheduler处理；
* scheduler转发acknowledgement给ExecutionGraph；
* ExecutionGraph转发acknowledgement给CheckpointCoordinator；
* CheckpointCoordinator处理acknowledgement，并维护checkpoint总体状态；



**基于 Checkpoint 的恢复**
基于Checkpoint的恢复，主要内容可见CheckpointCoordinator的restoreLatestCheckpointedStateToAll等方法。
其调度入口位于DefaultScheduler.handleGlobalFailure()。该方法的职责是，当集群启动或task出现异常时，根据异常具体原因判断，基于最近的checkpoint进行全部恢复或部分task恢复。

**如何保证At Least Once 和 Exactly Once ？**

At Least Once：

Exactly Once：



# 总结

Checkpoint是Flink中的一个重要内容，是Flink保证At Least Once和Exactly Once的基础之一。