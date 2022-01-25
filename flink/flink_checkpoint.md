# 简介
Checkpoint/savepoint机制是Flink中的重要内容，主要是定时保存或手动触发保存所有节点的状态，存储到内存或HDFS等外部存储中。

在全部或部分节点异常重启时，可基于checkpoint进行恢复，或整个任务异常终止、升级时基于savepoint进行恢复，用于保证At Least Once和Exactly Once。


# 核心类
Flink版本 : 1.14 。

**JobMaster** -- 一个Job的JobManager，每个job在同一时刻有且仅有一个JobMaster。

在HighAvailability配置下，由JobMasterServiceLeadershipRunner参与选举，成为Leader后，启动相应job的JobMaster，并启动job。

其中包括BlobWriter、HeartbeatServices、SlotPoolService、LeaderRetrievalService(ResourceManager相关)、SchedulerNG、JobManagerJobStatusListener等组件。


**SchedulerNG **-- Job的调度接口，负责job调度、异常处理等。
实现类主要为SchedulerBase，包括ExecutionGraph、ExecutionGraphHandler、OperatorCoordinatorHandler等。

**ExecutionGraph** -- 控制整个job的data flow的分布式执行，粒度细到每个并行的task、每个中间流及其交互。

实现类主要为DefaultExecutionGraph，主要包括Map<JobVertexID, ExecutionJobVertex> tasks组成的DAG、JobStatusListener、CheckpointCoordinator等。

**CheckpointCoordinator** -- 负责Checkpoint/Savepoint等的核心类，发送消息到相应task来触发checkpoint的创建，接收task的回应。
包括Map<Long, PendingCheckpoint>、CompletedCheckpointStore(已完成的checkpoint)等。

**Execution** -- JobManager端一个Subtask/ExecutionVertex的一次执行。
一个Subtask多次执行时(失败恢复/重计算/更新配置等原因)，对应多个Execution。

**TaskExecutor** -- TaskManager的对应Class。
负责多个Task的执行。

**Task** -- TaskManager中，一个Subtask的一个并行度的执行。

包括operator执行、输入、输出、与JobManager的交互。

每个Task由一个固定的线程执行。

Task不负责与其他task的交互，也不知道是否是第一次执行/重复执行，这些信息由JobManager维护。

**OperatorChain** --  表示由一个StreamTask执行的一串Operator。

入口为mainOperator，它负责拉取输入并生产数据push给后续其他operator。

**StreamTask** --  每个StreamTask执行一个/多个StreamOperator（如连续的map/flatmap/filter的组成operator chain）。

Operator chain在一个线程中同步执行，因此有同样的stream paritition。

Operator chain中有一个head operator和多个chained operator。

有one-input和two-input 类型的head operator。

**CheckpointedInputGate** -- 基于CheckpointBarrierHandler，处理从InputGate得到的CheckpointBarrier及cancel/end等checkpoint相关Event。

**CheckpointBarrierHandler** -- 处理接收到的checkpoint barrier。

**TwoPhaseCommitSinkFunction** -- 

**KafkaSink** -- 





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

其调度入口位于DefaultScheduler.handleGlobalFailure()。

该方法的职责是，当集群启动或task出现异常时，根据异常具体原因判断，将job直接置为失败/基于最近的checkpoint进行全部恢复/部分task恢复。

```
private void maybeRestartTasks(final FailureHandlingResult failureHandlingResult) {
    // 根据失败详情判断是否可以重启
    if (failureHandlingResult.canRestart()) {
        // 从最近的checkpoint开始重启恢复，不一定重启所有operator
        restartTasksWithDelay(failureHandlingResult);
    } else {
        failJob(failureHandlingResult.getError(), failureHandlingResult.getTimestamp());
    }
}
```

**如何保证At Least Once 和 Exactly Once ？**

At Least Once：

保证At Least Once，主要是基于：

1. 定时执行Checkpoint，异常时基于Checkpoint恢复，进行消息重播重新处理；
2. 一个Operator有多个输入流时，输入流中的Barrier不进行对齐。

At Least Once的Barrier实现过程见类 CheckpointBarrierTracker，其主要处理过程如下。

由此可以看出，在有多个输入流的情况下，当某个输入流的Barrier到达时，并不会阻塞该流后续的数据处理，只是记录该流的Barrier已到达。

当某个Checkpoint的第一个Barrier到达时，创建一个CheckpointBarrierCount放入队列中，使用第一个Barrier的checkpointId及timestamp作为该checkpoint的标识。

当所有流的Barrier都已到达时，开始进行snapshot。

假设

```
// 所有上游总数
private int numOpenChannels;
//已接收到Barrier（但不是所有）的Checkpoint
private final ArrayDeque<CheckpointBarrierCount> pendingCheckpoints;

 // 处理Barrier
@Override
public void processBarrier(
        CheckpointBarrier receivedBarrier, InputChannelInfo channelInfo, boolean isRpcTriggered)
        throws IOException {
    final long barrierId = receivedBarrier.getId();

    //。。。
    
    // find the checkpoint barrier in the queue of pending barriers
    CheckpointBarrierCount barrierCount = null;
    int pos = 0;

    for (CheckpointBarrierCount next : pendingCheckpoints) {
        if (next.checkpointId() == barrierId) {
            barrierCount = next;
            break;
        }
        pos++;
    }

	// 不是某Checkpoint的第一个Barrier
    if (barrierCount != null) {
        // add one to the count to that barrier and check for completion
        int numChannelsNew = barrierCount.markChannelAligned(channelInfo);
        if (numChannelsNew == barrierCount.getTargetChannelCount()) {
            // checkpoint can be triggered (or is aborted and all barriers have been seen)
            // first, remove this checkpoint and all all prior pending
            // checkpoints (which are now subsumed)
            for (int i = 0; i <= pos; i++) {
                pendingCheckpoints.pollFirst();
            }

            // 通知下游，即通知StreamTask开始进行snapshot
            if (!barrierCount.isAborted()) {
                triggerCheckpointOnAligned(barrierCount);
            }
        }
    } else {
        // 最新的一个checkpoint的第一个barrier
        if (barrierId > latestPendingCheckpointID) {
            markAlignmentStart(barrierId, receivedBarrier.getTimestamp());
            latestPendingCheckpointID = barrierId;
            pendingCheckpoints.addLast(
                    new CheckpointBarrierCount(receivedBarrier, channelInfo, numOpenChannels));

            // make sure we do not track too many checkpoints
            if (pendingCheckpoints.size() > MAX_CHECKPOINTS_TO_TRACK) {
                pendingCheckpoints.pollFirst();
            }
        }
    }
}
```


如图所示：

todo: barrier不对齐


Exactly Once：

要保证Exactly Once，主要是基于：

1. 一个Operator有多个输入流时，输入流的Barrier要进行对齐。即要等所有输入流中的Barrier都到齐后，才发送Barrier到下游并进行snapshot。在这之前到达的输入数据，都保存在缓存中，不会发送给下游。
2. Sink支持两阶段提交，输出的目标（Kafka/Hdfs等）要支持事务。Sink进行snapshot，并将结果医事务形式预提交到Kafka。待所有节点的snapshot完成后，CheckpointCoordinator通知Sink端，Sink端通知Kafka完成事务。过程中发生异常，就会通知Kafka端对事务进行回滚。

如图所示：

todo: barrier对齐

todo: sink事务


# 总结

Checkpoint是Flink中的一个重要内容，是Flink保证At Least Once和Exactly Once的基础之一。