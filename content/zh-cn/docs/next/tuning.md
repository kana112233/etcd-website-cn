---
title: Tuning
weight: 1450
description: When to update the heartbeat interval and election timeout settings　何时更新心跳间隔和选举超时设置
---

etcd中的默认设置应该适用于平均网络延迟较低的本地网络上的安装。
The default settings in etcd should work well for installations on a local network where the average network latency is low. 
但是，当在多数据中心或有高延迟的网络，心跳间隔和选举超时设置需要调节。
However, when using etcd across multiple data centers or over networks with high latency, the heartbeat interval and election timeout settings may need tuning.
网络不是延迟的唯一来源。每一个请求和响应可能被在leader和follwer之间的慢磁盘影响。
The network isn't the only source of latency. Each request and response may be impacted by slow disks on both the leader and follower. 
每一个超时代表从请求到另一个机器成功响应的总时间。
Each of these timeouts represents the total time from request to successful response from the other machine.

## Time parameters　时间参数
底层的分布式一致性协议依赖于两个独立时间参数，以确保当一个节点暂停或离线时，节点可以进行切换。
The underlying distributed consensus protocol relies on two separate time parameters to ensure that nodes can handoff leadership if one stalls or goes offline. 
第一个是*心跳间隔*。　这是领导者通知追随者它仍然是领导者的频率。
The first parameter is called the *Heartbeat Interval*. This is the frequency with which the leader will notify followers that it is still the leader.

对于最佳实践，应该围绕成员之间的往返时间设置参数。默认etcd使用`100ms`的心跳间隔。
For best practices, the parameter should be set around round-trip time between members. By default, etcd uses a `100ms` heartbeat interval.

第二个参数是*选举超时*。这个超时是follower节点在尝试成为领导者之前，在没有听到心跳的情况下将运行的时间。默认，etcd使用`1000ms`作为选举超时。
The second parameter is the *Election Timeout*. This timeout is how long a follower node will go without hearing a heartbeat before attempting to become leader itself. By default, etcd uses a `1000ms` election timeout.

调整这些值是一种权衡。心跳间隔的值建议在成员之间平均往返时间(RTT)的最大值附近，通常在往返时间的0.5-1.5倍左右。
Adjusting these values is a trade off. The value of heartbeat interval is recommended to be around the maximum of average round-trip time (RTT) between members, normally around 0.5-1.5x the round-trip time. 
如果心跳间隔太低，etcd将发送不必要的信息，这会增加CPU和网络资源的使用。
If heartbeat interval is too low, etcd will send unnecessary messages that increase the usage of CPU and network resources. 
另一方面，太高的心跳间隔导致高的选举超时。
On the other side, a too high heartbeat interval leads to high election timeout. 
较高的选举超时需要较长的时间来检查leader故障。
Higher election timeout takes longer time to detect a leader failure.
测量往返时间（RTT）最简单的方法似乎使用[PING utility][ping]。
 The easiest way to measure round-trip time (RTT) is to use [PING utility][ping].
选举超时应该基于心跳间隔和成员间的平均的往返时间来设置。
The election timeout should be set based on the heartbeat interval and average round-trip time between members. 
选举超时必须至少是往返时间的10倍，这样才能说明网络中的差异。
Election timeouts must be at least 10 times the round-trip time so it can account for variance in the network. 
例如，如果往返时间是10ms,那么领导超时应该至少是100ms。
For example, if the round-trip time between members is 10ms then the election timeout should be at least 100ms.

选举超时的上限是50000ms(50s),只有在部署一个全球分布的etcd集群时才应该使用。
The upper limit of election timeout is 50000ms (50s), which should only be used when deploying a globally-distributed etcd cluster. 
美国大陆的合理往返时间是130毫秒，美国和日本之间的时间大约是350-400ms。
A reasonable round-trip time for the continental United States is 130ms, and the time between US and Japan is around 350-400ms. 
如果网络具有不均匀的性能或规则的数据包延迟/丢失，则可能需要多次重试才能成功地发送数据包。
If the network has uneven performance or regular packet delays/loss then it is possible that a couple of retries may be necessary to successfully send a packet. 
所以5s是全球往返时间的安全上限。由于选举超时应该比刚播时间大一个数量级，因此对于一个全球分布的集群，在~5s的情况下，50秒是一个合理的最大值。
So 5s is a safe upper limit of global round-trip time. As the election timeout should be an order of magnitude bigger than broadcast time, in the case of ~5s for a globally distributed cluster, then 50 seconds becomes a reasonable maximum.

心跳间隔和选举超时在集群的所有成员应该是相同的。设置不同的值也许破坏集群的稳定性。
The heartbeat interval and election timeout value should be the same for all members in one cluster. Setting different values for etcd members may disrupt cluster stability.

默认值可以在命令行被复写：
The default values can be overridden on the command line:

```sh
# Command line arguments:
$ etcd --heartbeat-interval=100 --election-timeout=500

# Environment variables:
$ ETCD_HEARTBEAT_INTERVAL=100 ETCD_ELECTION_TIMEOUT=500 etcd
```
这些值以毫秒为单位指定。
The values are specified in milliseconds.

## Snapshots　快照
etcd将所有key更改附加到日志文件。此日志将永远增长，并且是对键所做的每个更改的完整性历史记录。　
etcd appends all key changes to a log file. This log grows forever and is a complete linear history of every change made to the keys. 

一个完整的历史对于很少使用的集群来说是很好的，但是大量使用的集群会携带一个大的日志。
A complete history works well for lightly used clusters but clusters that are heavily used would carry around a large log.

为了避免日志过大，etcd会定期创建快照。这些快照为etcd提供了一种通过保存系统的当前状态和删除旧日志来压缩日志的方法。
To avoid having a huge log etcd makes periodic snapshots. These snapshots provide a way for etcd to compact the log by saving the current state of the system and removing old logs.

### Snapshot tuning　快照调整
使用V2后端创建快照的成本很高，因此只有在对etcd进行了给定数量的更改之后才能创建快照。
Creating snapshots with the V2 backend can be expensive, so snapshots are only created after a given number of changes to etcd. 
默认，每10,000个更改之后创建快照。如果etcd的内存使用和磁盘使用是太高的，尝试通过设置下面的命令来降低快照的阈值：
By default, snapshots will be made after every 10,000 changes. If etcd's memory usage and disk usage are too high, try lowering the snapshot threshold by setting the following on the command line:

```sh
# Command line arguments:
$ etcd --snapshot-count=5000

# Environment variables:
$ ETCD_SNAPSHOT_COUNT=5000 etcd
```

## Disk　磁盘

etcd集群对磁盘延迟是非常敏感的。因为etcd必须将提议保存到日志中，来自其他进程的磁盘活动可能会导致较长的`fsync`延迟。结果是etcd可能会错过心跳，导致请求超时和临时的leader丢失。
An etcd cluster is very sensitive to disk latencies. Since etcd must persist proposals to its log, disk activity from other processes may cause long `fsync` latencies. The upshot is etcd may miss heartbeats, causing request timeouts and temporary leader loss. 
当给定高磁盘优先级时，etcd服务器有时可以稳定地与这些进程并且运行。
An etcd server can sometimes stably run alongside these processes when given a high disk priority.

在Linux上,　etcd的磁盘优先级可以使用`ionice`配置：
On Linux, etcd's disk priority can be configured with `ionice`:

```sh
# best effort, highest priority
$ sudo ionice -c2 -n0 -p `pgrep etcd`
```

## Network　网络

如果etcd的leader服务于大量的并发客户端请求，它可能由于网络拥塞而延迟处理follwer对等请求。这表现为在follower节点上发送缓冲区错误消息。
If the etcd leader serves a large number of concurrent client requests, it may delay processing follower peer requests due to network congestion. This manifests as send buffer error messages on the follower nodes:

```
dropped MsgProp to 247ae21ff9436b2d since streamMsg's sending buffer is full
dropped MsgAppResp to 247ae21ff9436b2d since streamMsg's sending buffer is full
```

These errors may be resolved by prioritizing etcd's peer traffic over its client traffic. On Linux, peer traffic can be prioritized by using the traffic control mechanism:

```
tc qdisc add dev eth0 root handle 1: prio bands 3
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 match ip sport 2380 0xffff flowid 1:1
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 match ip dport 2380 0xffff flowid 1:1
tc filter add dev eth0 parent 1: protocol ip prio 2 u32 match ip sport 2379 0xffff flowid 1:1
tc filter add dev eth0 parent 1: protocol ip prio 2 u32 match ip dport 2379 0xffff flowid 1:1
```

[ping]: https://en.wikipedia.org/wiki/Ping_(networking_utility)
