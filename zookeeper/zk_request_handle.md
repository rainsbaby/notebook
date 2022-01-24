Created on 2021-09-02

## 简介
ZK的核心是保证分布式系统中的数据一致性，也就是每次数据写入、更新后，所有Server上都要达到一致的状态。之后Client向任一个Server发起查询操作，都会得到一样的结果。
那么，ZK中数据写入、更新（即一个事务）的操作是如何完成的呢？每个Server上如何达成一致的呢？

## 核心类

Leader -- Leader的主要控制者，Leader.lead()方法是Server被选举为leader后的逻辑入口。控制Leader进行ZabState.DISCOVERY -> ZabState.SYNCHRONIZATION -> ZabState.BROADCAST的状态流转。
Follower -- Follower的主要控制者，Follower.followLeader()是逻辑入口。控制Follower进行ZabState.DISCOVERY -> ZabState.SYNCHRONIZATION -> ZabState.BROADCAST的状态流转。
Observer -- Observer的主要控制逻辑，Observer.observeLeader()是逻辑入口。

ZooKeeperServer -- Server的主要逻辑。
LeaderZooKeeperServer/FollowerZooKeeperServer/ObserverZooKeeperServer -- 建立Leader/Follower/Observer 的RequestProcessor的pipeline、SessionTracker等。

RequestProcessor -- 在Server端处理Client的Request。被串联成pipeline，按顺序从前往后执行，每个RequestProcessor处理完成后，交给它的nextProcessor。
PrepRequestProcessor -- 预处理，为事务型操作（数据更新/Path更新/Watch更新等）的Request创建对应的事务Record。
LeaderRequestProcessor -- 处理Session从local 到global session的升级。
ProposalRequestProcessor -- 负责发送Proposal给所有Follower，之后交给SyncRequestProcessor和AckRequestProcessor处理。事务提交分为proposal和commit两步。
SyncRequestProcessor -- 主要负责存储request到磁盘，之后交给AckRequestProcessor/SendAckRequestProcessor来发送ACK。Leader中由AckRequestProcessor发送ACK给它自己；Follower中由SendAckRequestProcessor发送ACK给远程Leader；Observer中不发送ACK。
AckRequestProcessor -- Leader发送Proposal后，发一个ACK给自己。之后由LearnerHandler接收Follower发送的ACK，交给Leader处理。Leader判断得到过半的ACK后，表示proposal成功可以进行commit，通知所有Follower，并通知CommitProcessor进行本地Commit操作。
CommitProcessor -- 进行Server本地的Commit操作。
FollowerRequestProcessor -- 转发更新类Request给Leader。
ObserverRequestProcessor -- 转发更新类Request给Leader。
FinalRequestProcessor -- 始终位于RequestProcessor pipeline的最末尾，负责处理Query类请求，并apply request对应的事务，最后发送response给Client。


## 主要流程
RequestProcessor的pipeline拼装，在LeaderRequestProcessor/FollowerRequestProcessor/ObserverRequestProcessor中完成。
### RequestProcessor的pipeline
Leader/Follower/Observer都可以响应查询类请求，数据更新类请求统一交给Leader处理。
Leader处理事务的流程是，首先发送proposal给所有follower，收到过半的proposal ack后，发送commit给所有follower。
1. Leader端RequestProcessor 按从上到下的顺序执行：
> LeaderRequestProcessor
> PrepRequestProcessor
> ProposalRequestProcessor (交给SyncRequestProcessor进行存储，之后由SendAckRequestProcessor发送ACK给自己）
> CommitProcessor
> ToBeAppliedRequestProcessor
> FinalRequestProcessor

2. Follower端
Follower/Observer端收到数据更新类请求，会直接foward给leader处理。
收到Leader发送的Proposal后，从上到下的顺序执行：
> Follower.followerLeader()
> Follower.processPacket()
> FollowerZooKeeperServer.logRequest()
> SyncRequestProcessor
> SendAckRequestProcessor

commit的pipeline：
> FollowerRequestProcessor.followerLeader()
> Follower.processPacket()
> CommitProcessor.commit
> FinalRequestProcessor

* [ ] 如果Leader迟迟没有收到某个follower对proposal/commit的响应，会如何处理？

### Leader端Request执行

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/zk/zk_request_handle_flow.png)

### Leader->Learner的数据同步
Server赢得选举成为Leader后，与每个Learner（Follower/Observer）都建立一个socket连接，进行网络通信。通信内容包括，Leader选举完成后Leader与Follower间数据同步、Leader发送Proposal/Commit给Follower/Observer、Leader向Follower定时发送的ping消息等。




