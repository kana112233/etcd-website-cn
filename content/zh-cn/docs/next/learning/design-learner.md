---
title: etcd learner design
weight: 2375
description: Mitigating common challenges with membership reconfiguration 通过成员重新配置缓解常见挑战
---

etcd Learner
============

*Gyuho Lee (github.com/gyuho, Amazon Web Services, Inc.), Joe Betz (github.com/jpbetz, Google Inc.)*


Background 背景
==========

成员重新配置是最大的操作挑战。让我们温习一下常用的挑战。
Membership reconfiguration has been one of the biggest operational challenges. Let’s review common challenges.

### 1. New Cluster member overloads Leader 1. 新的集群成员重载领导者
一个新的成员一开始加入时没有数据，因此，严格的从领导者更新日志数据直到赶上领导者的日志。
A newly joined etcd member starts with no data, thus demanding more updates from leader until it catches up with leader’s logs. 

然后领导者的网络更可能过载，阻止或丢弃领导者向follower的心跳。
Then leader’s network is more likely to be overloaded, blocking or dropping leader heartbeats to followers. 

在这种情况下，follower也许选举超时来开始一个新的领导选举。
In such case, a follower may election-timeout to start a new leader election. 
就是说，具有新成员的集群更容易受到领导选举的影响。
That is, a cluster with a new member is more vulnerable to leader election. 

领导选举和后续更新传播到新成员都容易导致集群不可用的时期。
Both leader election and the subsequent update propagation to the new member are prone to causing periods of cluster unavailability (see *Figure 1*).

![server-learner-figure-01](../img/server-learner-figure-01.png)

### 2. Network Partitions scenarios 2. 网络分区场景
如果发生网络分区怎么办？ 这取决于领导者分区。如果领导者仍然维持活跃的法定人数，这个集群将继续运作。
What if network partition happens? It depends on leader partition. If the leader still maintains the active quorum, the cluster would continue to operate (see *Figure 2*).

![server-learner-figure-02](../img/server-learner-figure-02.png)

#### 2.1 Leader isolation 领导隔离

如果领导者和集群的其他成员隔离了怎么办？Leader监控了每一个follower的进度。
What if the leader becomes isolated from the rest of the cluster? Leader monitors progress of each follower. 
当leader与quorum失去连接时，它会重新恢复为follower，这将影响集群的可用性。
When leader loses connectivity from the quorum, it reverts back to follower which will affect the cluster availability (see *Figure 3*).

![server-learner-figure-03](../img/server-learner-figure-03.png)
当向3个集群添加一个新的节点时，集群的size变成了4,quorum的size变成了3.
When a new node is added to 3 node cluster, the cluster size becomes 4 and the quorum size becomes 3. 
如果新的节点加入集群，然后发生网络分区怎么办？
它取决于分区后新成员所在的分区。
What if a new node had joined the cluster, and then network partition happens? It depends on which partition the new member gets located after partition.

#### 2.2 Cluster Split 3+1 集群分割 3+1
如果新的节点和leader在相同的分区，leader仍然维持3个活跃的quorum。 没有领导选举发生，没有集群可用性受到影响。
If the new node happens to be located in the same partition as leader’s, the leader still maintains the active quorum of 3. No leadership election happens, and no cluster availability gets affected (see *Figure 4*).

![server-learner-figure-04](../img/server-learner-figure-04.png)

#### 2.3 Cluster Split 2+2 集群分为2+2
如果集群是2和2分区，那么任何一个分区都不会维持3个法定人数。
If the cluster is 2-and-2 partitioned, then neither of partition maintains the quorum of 3. 
在这种情况下，领导选举就会发生。
In this case, leadership election happens (see *Figure 5*).

![server-learner-figure-05](../img/server-learner-figure-05.png)

#### 2.4 Quorum Lost 法定人数丢失
如果先发生网络分区，然后添加了新成员会怎么样？ 一个3个节点的分区集群已经有一个断开的follower。
添加新成员时，法定人数从2改成了3.现在，这个集群4个只有2个激活的，因此丢失法定人数并开始新的领导选举。
What if network partition happens first, and then a new member gets added? A partitioned 3-node cluster already has one disconnected follower. When a new member is added, the quorum changes from 2 to 3. Now, this cluster has only 2 active nodes out 4, thus losing quorum and starting a new leadership election (see *Figure 6*).

![server-learner-figure-06](../img/server-learner-figure-06.png)

因为增加成员操作可以改变法定人数的size，始终建议先“member remove”再替换不健康的节点。
Since member add operation can change the size of quorum, it is always recommended to “member remove” first to replace an unhealthy node.

向只有1个节点的集群添加一个新的成员来改变法定人数为2，立刻就引发领导选举，当前任领导发现法定人数不活跃时。
这是因为“member add”操作是2步过程，用户需要先应用“member add”命令，然后再启动新节点过程。
Adding a new member to a 1-node cluster changes the quorum size to 2, immediately causing a leader election when the previous leader finds out quorum is not active. This is because “member add” operation is a 2-step process where user needs to apply “member add” command first, and then starts the new node process (see *Figure 7*).

![server-learner-figure-07](../img/server-learner-figure-07.png)

### 3. Cluster Misconfigurations 集群配置错误
更糟糕的情况是添加的成员配置错误。
An even worse case is when an added member is misconfigured. 

Membership重新配置是一个两步过程：“etcdctl member add”然后开始一个带peer URL的etcd服务器进程。
Membership reconfiguration is a two-step process: “etcdctl member add” and starting an etcd server process with the given peer URL. 
就是说， 无论URL是什么，“member add”命令都适用，无论这个URL是非法的。如果第一步是非法的URL，第二部不可以开始一个新的etcd。一旦集群丢失了quorum，没有办法恢复会员资格改变。
That is, “member add” command is applied regardless of URL, even when the URL value is invalid. If the first step is applied with invalid URLs, the second step cannot even start the new etcd. Once the cluster loses quorum, there is no way to revert the membership change (see *Figure 8*).

![server-learner-figure-08](../img/server-learner-figure-08.png)
同样适用于多节点集群。例如，集群有2个成员down（一个启动失败了，另一个配置错误了）2个成员up，但是现在它要求至少3个选票去改变集群的关系。
Same applies to a multi-node cluster. For example, the cluster has two members down (one is failed, the other is misconfigured) and two members up, but now it requires at least 3 votes to change the cluster membership (see *Figure 9*).

![server-learner-figure-09](../img/server-learner-figure-09.png)

如上所示，一个简单的错误配置可能会使整个集群陷入无法运行的状态。
As seen above, a simple misconfiguration can fail the whole cluster into an inoperative state. 
在这种状态下，一个操作需要使用`etcd --force-new-cluster`标志手动重新创建集群。
In such case, an operator need manually recreate the cluster with `etcd --force-new-cluster` flag. 

由于etcd已经成为Kubernetes的关键任务服务，即使是最轻微的中断也可能对用户产生重大的影响。
As etcd has become a mission-critical service for Kubernetes, even the slightest outage may have significant impact on users. 

我们怎样才能更好地使让etcd此类操作更容易？
What can we better to make etcd such operations easier? 
除了其他事项外，领导选举对集群可用性是最关键： 我们能否通过不改变法定人数来减少成员关系重新配置的破坏性？
Among other things, leader election is most critical to cluster availability: Can we make membership reconfiguration less disruptive by not changing the size of quorum? 
一个新的节点是否可以空闲，只向leader请求最小的更新，直到它赶上leader的日志？
Can a new node be idle, only requesting the minimum updates from leader, until it catches up?

成员关系配置错误是否总是可逆并且可以用一种更安全的方式处理（错误的成员添加命令运行应该永远不会使集群失败）？
Can membership misconfiguration be always reversible and handled in a more secure way (wrong member add command run should never fail the cluster)? 
 
 当添加一个新的成员时，用户应该担心网络拓扑吗？
Should an user worry about network topology when adding a new member? 
 
无论节点的位置和正在进行的网络分区添加成员的API都可以用吗？
Can member add API work regardless of the location of nodes and ongoing network partitions?

Raft Learner
============

为了减少上一节中的这种可用性差距，[Raft §4.2.1](https://github.com/ongardie/dissertation/blob/master/stanford.pdf)介绍了一个新的节点角色“Learner”，这个角色作为一个**non-voting member**加入集群，
它会赶上领导者的日志。
In order to mitigate such availability gaps in the previous section, [Raft §4.2.1](https://github.com/ongardie/dissertation/blob/master/stanford.pdf) introduces a new node state “Learner”, which joins the cluster as a **non-voting member** until it catches up to leader’s logs.

Features in v3.4
----------------

操作员应该做尽可能少的工作来添加新的learner节点。
An operator should do the minimum amount of work possible to add a new learner node. 
`member add --learner`命令添加了一个新的learner，这个learner作为一个不投票的成员加入集群，但是它仍然会从leader接收所有的数据。
`member add --learner` command to add a new learner, which joins cluster as a non-voting member but still receives all data from leader (see *Figure 10*).

![server-learner-figure-10](../img/server-learner-figure-10.png)

当learner赶上了leader的进度，learner可以通过使用`member promote` API来晋升为可投票的成员，并且会计入法定人数。
When a learner has caught up with leader’s progress, the learner can be promoted to a voting member using `member promote` API, which then counts towards the quorum (see *Figure 11*).

![server-learner-figure-11](../img/server-learner-figure-11.png)
etcd服务器验证 提升请求 以确保其操作安全。
etcd server validates promote request to ensure its operational safety. 
只有当它完成了leader的日志后，learner才可以提升为一个可选举的成员。
Only after its log has caught up to leader’s can learner be promoted to a voting member (see *Figure 12*).

![server-learner-figure-12](../img/server-learner-figure-12.png)

Learner仅作为备用节点直到被提升： 领导不可以转为learner。
Learner only serves as a standby node until promoted: Leadership cannot be transferred to learner. 
Learner拒绝客户端读和写（客户端balancer不应该路由请求到learner）。
Learner rejects client reads and writes (client balancer should not route requests to learner). 
这意味着learner不需要向领导者发出读取索引请求。这样的限制简化了初始learner的实现在 v3.4 release (see *Figure 13*).
Which means learner does not need issue Read Index requests to leader. Such limitation simplifies the initial learner implementation in v3.4 release (see *Figure 13*).

![server-learner-figure-13](../img/server-learner-figure-13.png)

此外，etcd限制了一个集群可以拥有的learners的总数，避免通过日志复制使领导者过载。
In addition, etcd limits the total number of learners that a cluster can have, and avoids overloading the leader with log replication. 
Learner永远不会提示自己。 
Learner never promotes itself. 
虽然etcd提供learner状态信息和安全检查，但集群操作必须做出是否提升学习者的最终决定。
While etcd provides learner status information and safety checks, cluster operator must make the final decision whether to promote learner or not.

Features in v3.5
----------------

*让learner状态唯一和默认*: 
*Make learner state only and default*: 
将新的成员默认为learner将大大的提高成员重新配置的安全性，因为learner不会改变法定人数的数量。
Defaulting a new member state to learner will greatly improve membership reconfiguration safety, because learner does not change the size of quorum. 
错误配置将始终是可逆的，而且不会丢失法定人数。
Misconfiguration will always be reversible without losing the quorum.

*让投票成员晋升全自动*: 一旦learner赶上leader的日志，集群可以自动的提升learner。etcd需要被用户定义的某个阈值。一旦这个要求被满足，learner提升自己去投票。
*Make voting-member promotion fully automatic*: Once a learner catches up to leader’s logs, a cluster can automatically promote the learner. etcd requires certain thresholds to be defined by the user, and once the requirements are satisfied, learner promotes itself to a voting member. 

从用户的看法，“member add”命令的工作方式与今天相同，但学习者功能提供了更高的安全性。
From a user’s perspective, “member add” command would work the same way as today but with greater safety provided by learner feature.

*让learner作为备用故障转移节点*: learner作为备用节点加入，当集群可用性受到影响时，就会自动提升角色。
*Make learner standby failover node*: A learner joins as a standby node, and gets automatically promoted when the cluster availability is affected.

*让learner只读*: learner可以作为一个永远不会提升角色的只读节点。
*Make learner read-only*: A learner can serve as a read-only node that never gets promoted. 
在一个弱一致性模式，learner仅仅接受leader的数据并且永远也不会写。
In a weak consistency mode, learner only receives data from leader and never process writes. 

在没有共识开销的情况下在本地提供读取服务将大大减少领导者的工作量，但可能会提供过时的数据。
Serving reads locally without consensus overhead would greatly decrease the workloads to leader but may serve stale data. 
在强一致模式下，learner从leader请求读索引以提供最新的数据，但仍然拒绝写入。
In a strong consistency mode, learner requests read index from leader to serve latest data, but still rejects writes.


Learner vs. Mirror Maker [Mirror Maker](https://github.com/etcd-io/etcd/blob/main/etcdctl/doc/mirror_maker.md)
========================

etcd使用watch API实现了“mirror maker”，将key的创建和更新持续的传输到单独的集群。
etcd implements “mirror maker” using watch API to continuously relay key creates and updates to a separate cluster. 

一旦完成初始化同步，mirroring通常具有低延迟开销。
Mirroring usually has low latency overhead once it completes initial synchronization. 
Learner和Mirroring有重叠的部分，因为两者都可用于复制现有数据以进行只读。
Learner and mirroring overlap in that both can be used to replicate existing data for read-only. 
然而，mirroring并不能保证线性化。
However, mirroring does not guarantee linearizability. 
在网络断开期间，先前的key-value也许被丢弃，客户端期待按照正确的顺序验证watch的响应。因此，mirror没有顺序保证。
During network disconnects, previous key-values might have been discarded, and clients are expected to verify watch responses for correct ordering. Thus, there is no ordering guarantee in mirror. 
以一致性为代价，为了最小的延时而使用Mirror（例如跨数据中心）
Use mirror for minimum latency (e.g. cross data center) at the costs of consistency. 
使用learner保留所有的历史数据和它的顺序。
Use learner to retain all historical data and its ordering.

Appendix: Learner Implementation in v3.4 附录： Learner实现 in v3.4
========================================

将“Learner”节点类型暴露给"MemberAdd" API。
*Expose "Learner" node type to "MemberAdd" API.*

etcd客户端为learner节点的“MemberAdd”API添加了一个flag。
etcd client adds a flag to “MemberAdd” API for learner node. And etcd server handler applies membership change entry with `pb.ConfChangeAddLearnerNode` type. Once the command has been applied, a server joins the cluster with `etcd --initial-cluster-state=existing` flag. This learner node can neither vote nor count as quorum.

etcd服务器不许从领导转换为learner，因为它可能仍然落后并且不算做法定人数。
etcd server must not transfer leadership to learner, since it may still lag behind and does not count as quorum. 
etcd server限制了learner的数量，我们的learner越多，leader必须传播的数据越多。客户端也许和learner节点交流，但是learner拒绝除了可序列化读和成员状态API之外的所有请求。
etcd server limits the number of learners that cluster can have to one: the more learners we have, the more data the leader has to propagate. Clients may talk to learner node, but learner rejects all requests other than serializable read and member status API. 

这是为了简化初始实现。未来，learner可以扩展为只读的持续镜像集群数据服务器。
This is for simplicity of initial implementation. In the future, learner can be extended as a read-only server that continuously mirrors cluster data. 
客户端balancer必须提供辅助函数以排除learner节点端点。
Client balancer must provide helper function to exclude learner node endpoint. 
否则，请求发送learner也许会失败。客户端同步成员调用应考虑学习者节点类型。
Otherwise, request sent to learner may fail. Client sync member call should factor into learner node type. 
客户端端点更新调用也是如此。
So should client endpoints update call.

`MemberList` and `MemberStatus` responses should indicate which node is learner.

*Add "MemberPromote" API.*

Internally in Raft, second `MemberAdd` call to learner node promotes it to a voting member. Leader maintains the progress of each follower and learner. If learner has not completed its snapshot message, reject promote request. Only accept promote request if and only if: The learner node is in a healthy state. The learner is in sync with leader or the delta is within the threshold (e.g. the number of entries to replicate to learner is less than 1/10 of snapshot count, which means it is less likely that even after promotion leader would not need send snapshot to the learner). All these logic are hard-coded in `etcdserver` package and not configurable.

Reference
=========

- Original github issue: [etcd#9161](https://github.com/etcd-io/etcd/issues/9161)
- Use case: [etcd#3715](https://github.com/etcd-io/etcd/issues/3715)
- Use case: [etcd#8888](https://github.com/etcd-io/etcd/issues/8888)
- Use case: [etcd#10114](https://github.com/etcd-io/etcd/issues/10114)
