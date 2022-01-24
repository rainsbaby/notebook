## 简介
ZAB协议中一个重要流程是Leader选举。当系统刚启动或Leader崩溃时，Server进入LOOKING状态，会触发选举流程。

## 核心类
QuorumPeer -- Server节点启动入口。
QuorumPeer -- 表示集群模式下的一个Server节点，基于过半协议进行状态的流转（Leader Election/Follower/Leader）。
QuorumCnxManager -- 管理server之间的连接，专门用于Leader选举，基于TCP协议实现。为每一对server之间维护有且仅有一个tcp连接（只能由sid较大的一方发起连接），为对端的每个peer维护一个发送队列。由内部的SendWorker和RecvWorker线程负责发送和接收工作。Listener负责accept其他server发过来的连接请求。
FastLeaderElection -- 默认的选举方式
QuorumVerifier -- 负责过半协议判定的接口，即判断节点是否获得足够多的选票，包括QuorumMaj和QuorumHierarchical两个实现类。
QuorumMaj -- 默认判定方式，过半即成功。QuorumHierarchical中有分组和权重。

## 主要流程

选举中主要角色与功能：
![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/zk/zk_leader_election.png)

流程：
> 1. Server启动后，进入LOOING状态；
> 2. Server之间建立连接，由QuorumCnxManager管理
> 3. LOOING状态下，开始Leader选举流程,选票内容包括（proposedLeader, proposedZxid, proposedEpoch）。如果当前节点参与选举，则最开始投票给自己，即proposedLeader为当前server的myid，proposedZxid为当前server的snapshot中最大的zxid，proposedEpoch为本server当前的epoch；否则节点为Observer，则投票内容为（Long.MIN_VALUE,Long.MIN_VALUE,Long.MIN_VALUE）；
> 4. 选票初始化及更新后，都会通知其他所有Server；Server端记录选择自己为leader的选票数目；
> 5. Server收到其他server的投票后，与本地投票进行比较，若满足
 ((newEpoch > curEpoch)
 || ((newEpoch == curEpoch) && ((newZxid > curZxid)
 || ((newZxid == curZxid) && (newId > curId)))))
 则用收到的选票替代本地选票，然后通知其他Server；newId表示每个server的myid；
> 6. 当某个Server收到超过半数server的投票时，表示Leader选举完成。若选出的leader为本server，则成为Leader，否则为Follower。
> 7. 之后会启动相应的Leader/Follower 流程，进行DISCOVERY/SYNCHRONIZATION/BROADCAST流程。