---
title: Failure modes 失败的模式
weight: 4250
description: Kinds of failures and ectd's tolerance for them 各种失败和etcd对他们的容错
---
在一个大的机器部署中失败是常见的。
Failures are common in a large deployment of machines. 
当硬件或软件出现故障时，机器就会出现故障。
A machine fails when its hardware or software malfunctions. 
当有停电或者网络错误时，多个集群就会一起故障。
Multiple machines fail together when there are power failures or network issues. 
多种故障也可能同时发生； 枚举所有可能的失败安利几乎是不可能的。
Multiple kinds of failures can also happen at once; it is almost impossible to enumerate all possible failure cases.

在此章节，我们对各种故障进行分类并讨论etcd是如何设计来容忍这些故障。
In this section, we catalog kinds of failures and discuss how etcd is designed to tolerate these failures. 
大部分用户，（如果不是全部）都可以将特定故障映射为一种故障。
Most users, if not all, can map a particular failure into one kind of failure. 
要为罕见或[unrecoverable failures][unrecoverable]做好准备，请始终[back up][backup]etcd集群。
To prepare for rare or [unrecoverable failures][unrecoverable], always [back up][backup] the etcd cluster.

## Minor followers failure 最小的followers失败

当小于一半的follower失败，etcd集群仍然可以接受请求并在没有任何重大中断的情况下取得进展。
When fewer than half of the followers fail, the etcd cluster can still accept requests and make progress without any major disruption. 
例如，两个follower失败不将影响5个成员的etcd集群操作。
For example, two follower failures will not affect a five member etcd cluster’s operation. 
然而，客户端将丢失失败的成员的连接。客户端库应该通过重连其他成员来隐藏用户读取请求的这些中断。
However, clients will lose connectivity to the failed members. Client libraries should hide these interruptions from users for read requests by automatically reconnecting to other members. 
由于重新连接，操作员应该预计其他成员的系统负载会增加。
Operators should expect the system load on the other members to increase due to the reconnections.

## Leader failure 领导失败
当领导失败，etcd集群自动选举新的领导。一旦leader失败，选举不会立即发生。
When a leader fails, the etcd cluster automatically elects a new leader. The election does not happen instantly once the leader fails. 
由于故障检测模型是基于超时的，因此选举新的leader需要大约选举超时。
It takes about an election timeout to elect a new leader since the failure detection model is timeout based.

在leader选举leader不能处理任何写入。
During the leader election the cluster cannot process any writes. 
选举期间发送的写入请求将排队等待处理，知道选举出新的领导者。
Write requests sent during the election are queued for processing until a new leader is elected.

已经发送给旧领导者单尚未提交的写操作可能会丢失。
Writes already sent to the old leader but not yet committed may be lost. 
新的领导者权重写前任领导者的任何未提交的条目。
The new leader has the power to rewrite any uncommitted entries from the previous leader. 
从用户的角度，一些写请求在一个新领导选举后可能超时。
From the user perspective, some write requests might time out after a new leader election. 
然而，未提交的写入会丢失。
However, no committed writes are ever lost.

新领导为所有租约自动延长超时时间。
The new leader extends timeouts automatically for all leases. 
这种机制确保租约不会在授予的TTL之前到期，即使它是由旧领导授权的。
This mechanism ensures a lease will not expire before the granted TTL even if it was granted by the old leader.

## Majority failure 多数者失败
当集群的大多数成员失败时，etcd集群失败并且无法接受更多的写入。
When the majority members of the cluster fail, the etcd cluster fails and cannot accept more writes.

一旦大多数成员可用，etcd集群只能从多数故障中恢复。如果大多数成员不能重新上线，那么运营商必须启动[disaster recovery][unrecoverable]来恢复集群。
The etcd cluster can only recover from a majority failure once the majority of members become available. If a majority of members cannot come back online, then the operator must start [disaster recovery][unrecoverable] to recover the cluster.

一旦大多数成员工作，etcd集群自动选举一个新的领导然后返回健康的状态。
Once a majority of members works, the etcd cluster elects a new leader automatically and returns to a healthy state. 
新的领导者会自动为所有租约延长超时时间。
The new leader extends timeouts automatically for all leases. This mechanism ensures no lease expires due to server side unavailability.

## Network partition 网络分区

网络分区类似与次要的follower失败或者领导失败。
A network partition is similar to a minor followers failure or a leader failure. 
网络分区切分etcd集群为两部分；一个成员占多数，另一个成员占少数。
A network partition divides the etcd cluster into two parts; one with a member majority and the other with a member minority. 
多数方成为可用集群，少数方不可用。
The majority side becomes the available cluster and the minority side is unavailable. 
etcd中没有“脑裂”，因为集群成员会随着当前大多数成员的批准而明确添加/删除。
There is no “split-brain” in etcd because cluster members are explicitly added/removed with each such change is approved by the current majority of members.
如果领导者站在多数一方，那么从多数的角度来看，失败是少数追随者的失败。
If the leader is on the majority side, then from the majority point of view the failure is a minority follower failure. 
如果领导在少数的一方，然后领导失败了。在少数方的领导下台，多数方选举新的领导。
If the leader is on the minority side, then it is a leader failure. The leader on the minority side steps down and the majority side elects a new leader.
一旦网络分区消失，少数方自动从多数方识别leader并且恢复他的状态。
Once the network partition clears, the minority side automatically recognizes the leader from the majority side and recovers its state.

## Failure during bootstrapping 启动期间失败
只有所有的成员启动成功，那么集群才算启动成功。
A cluster bootstrap is only successful if all required members successfully start. 

如果在启动期间任意一个发生了失败，删除所有成员的目录然后带一个新的集群token或目录token重启集群。
If any failure happens during bootstrapping, remove the data directories on all members and re-bootstrap the cluster with a new cluster-token or new discovery token.

当然，可以像恢复正在运行的集群一样恢复失败的自启动集群。
Of course, it is possible to recover a failed bootstrapped cluster like recovering a running cluster. 
然而，恢复那个集群几乎总是比启动一个新的集群花费更多的时间和资源，因为没有数据要恢复。
However, it almost always takes more time and resources to recover that cluster than bootstrapping a new one, since there is no data to recover.

[backup]: maintenance#snapshot-backup
[unrecoverable]: recovery
