---
title: FAQ
weight: 1200
description: Frequently asked questions 经常被问到的问题
---

## etcd, general

### What is etcd? 什么是etcd？

etcd是一致的分布式key-value存储数据库。在分布式系统中主要用作单独的协调服务。并设计为可完全容纳在内存中的少量数据。
etcd is a consistent distributed key-value store. Mainly used as a separate coordination service, in distributed systems. And designed to hold small amounts of data that can fit entirely in memory.

### How do you pronounce etcd? etcd是怎么发音的?
etcd is pronounced **/ˈɛtsiːdiː/**, and means "distributed `etc` directory."

### Do clients have to send requests to the etcd leader? 客户端一定要向etcd leader发送请求吗？

[Raft][raft]是基于领导的; 领导处理所有客户端的请求。然而，客户端不必知道领导是哪个节点。
[Raft][raft] is leader-based; the leader handles all client requests which need cluster consensus. However, the client does not need to know which node is the leader. Any request that requires consensus sent to a follower is automatically forwarded to the leader. Requests that do not require consensus (e.g., serialized reads) can be processed by any cluster member.
这些请求可以被任意集群成员处理，不要求一致性。

## Configuration 配置

### listen-<client,peer>-urls, advertise-client-urls 或者 initial-advertise-peer-urls有什么不同？
在 `listen-client-urls` and `listen-peer-urls` 指定local addresses到etcd server来接收传入的连接。为了在一个端口上监听所有的接口，指定`0.0.0.0`作为监听IP地址。

`advertise-client-urls` and `initial-advertise-peer-urls` 指定etcd客户端或其他etcd成员，用于联接etcd服务器。这个地址必须可以到达远程机器。
`advertise-client-urls` and `initial-advertise-peer-urls` specify the addresses etcd clients or other etcd members should use to contact the etcd server. The advertise addresses must be reachable from the remote machines. Do not advertise addresses like `localhost` or `0.0.0.0` for a production setup since these addresses are unreachable from remote machines. 不要设置像`localhost` or `0.0.0.0` 这样的地址，因为这些地址无法从远程计算机访问。

### Why doesn't changing `--listen-peer-urls` or `--initial-advertise-peer-urls` update the advertised peer URLs in `etcdctl member list`?
为什么不通过`--listen-peer-urls` or `--initial-advertise-peer-urls`来修改peer URLs?

A member's advertised peer URLs come from `--initial-advertise-peer-urls` on initial cluster boot. Changing the listen peer URLs or the initial advertise peers after booting the member won't affect the exported advertise peer URLs since changes must go through quorum to avoid membership configuration split brain. 
使用`etcdctl member update`来更新成员的peer的URL。

## Deployment 部署

### System requirements 系统请求

因为etcd会把数据写入磁盘，所有它的性能强依赖于磁盘的性能。因为这个原因，非常推荐使用SSD。为了了解磁盘是否足够的快，一个可能性是使用磁盘基准测试工具，例如[fio][fio]。 有关怎么去做这个的例子，读[这里][fio-blog-post]。为了避免性能下降或者偶尔的key-value存储超载，etcd会强制将可配置的存储大小配额默认设置为2GB。为了避免swap或者内存不足，机器应该至少要足够的内存来满足配额。8GB是正常环境推荐的最大size。如果配置的值超过8GB，etcd在启动时会发出警告。

在CoreOS公司里面，一个etcd集群通常部署在专用的CoreoOS Container Linux机器，他的配置是双核，2GB的内存，至少80GB的SSD。

**注意性能的本质是工作负载; 请在产品部署前先测试。** 更多的内容请看 [hardware][hardware-setup]。

大部分稳定的生产环境具有amd64架构的Linux操作系统;更多的请看[supported platform][supported-platform]。

### 为什么集群成员是奇数？

一个集群需要一半以上节点同意去更新集群的状态。假设一个集群有n个成员，仲裁数为(n/2)+1。为了任何奇数个数的集群，添加一个节点将总是增加仲裁所需的节点数。虽然添加一个节点到奇数的集群看起来会更好，因为会有更多的机器数量，容错能力更差，因为完全相同数量的节点也许会失败但是不会丢失quorum，但是有更多的节点可能会失败并丢弃quorum。如果集群是不能容纳更多失败的状态，删除一个节点之前添加一个节点是危险的因为如果这个新的节点注册失败（例如这个地址是错误配置的）， 这个quorum将永久丢失。

### 最大的集群size是多少？

理论上，没有硬性限制。然而，一个etcd集群可能不应该超过7个节点。[Google Chubby lock service][chubby],和etcd相似的并且广泛部署在Google很多年，建议运行5个节点。5个成员的etcd集群可以容纳两个成员失败，在大部分的例子是足够的。虽然更大的集群有更好的容错，但是写性能会下降，因为数据必须跨机器复制。

### 什么是容错？

只要可以建立成员仲裁，etcd集群就会运行。如果由于短暂的网络故障（例如分区）而丢失quorum，一旦网络恢复并且恢复仲裁后，etcd会自动安全恢复;Raft强制集群一致性。对于断电，etcd会将Raft日志持久化到磁盘；etcd将日志重放到故障点，并恢复集群参与。对于永久性硬件故障，这个节点也许会通过[runtime reconfiguration][runtime reconfiguration]从集群中删除。
 
推荐集群的成员个数为奇数。因为奇数大小的集群可以容忍的故障数量与偶数大小的集群相同，但是节点数量会更少。
通过比较偶数和奇数大小的集群可以看出差异：

| Cluster Size | Majority | Failure Tolerance |
|:-:|:-:|:-:|
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |
| 6 | 4 | 2 |
| 7 | 4 | 3 |
| 8 | 5 | 3 |
| 9 | 5 | 4 |

添加一个成员以使集群的大小达到偶数不会增加额外的容错。同样的，在网络分区期间，奇数个数成员保证将始终存在多数分区，该分区可以继续运行，并且在分区结束是成为真正的source。（source是其他分区会加入这个多数分区）

### etcd是否可以在跨区域或跨数据中心部署工作吗？

部署跨区的etcd来提高etcd的容错能力因为members are in separate failure domains.
这个代价是跨数据中心边界会产生更高的一致性请求延迟。
Deploying etcd across regions improves etcd's fault tolerance since members are in separate failure domains. The cost is higher consensus request latency from crossing data center boundaries.
由于etcd依靠成员仲裁来达成共识，跨数据中心的延迟会有些明显，因为至少集群成员的一半以上必须响应一致性要求。此外，集群数据必须在所有peers之间复制数据，因此也会产生带宽成本。

延迟时间更长，默认的etcd配置也许造成频繁的选举和心跳超时。参考[tuning]来调整高延迟部署的超时。

## 操作

### 怎样备份etcd集群？

etcdctl 提供了一个`snapshot`命令去创建备份。参考[backup][backup]为了更多细节。

### 在我删除一个不健康的集群成员前，我应该先添加一个成员吗？

当替换一个etcd节点时，先删除成员然后添加是很重要的。

etcd使用基于仲裁模型的分布式共识。
etcd employs distributed consensus based on a quorum model;

(n/2)+1个成员，多数成员必须先同意一个提案，然后才能将该提案提交到集群中。

 (n/2)+1 members, a majority, must agree on a proposal before it can be committed to the cluster.
 
 这些提案包括更新key-value和成员资格的改变。
 这个模型避免任何脑裂不一致的可能性。不利的一面是永久的仲裁丢失是灾难性的。

  These proposals include key-value updates and membership changes. This model totally avoids any possibility of split brain inconsistency. The downside is permanent quorum loss is catastrophic.
  

这怎么应用于membership： 如果三个成员的集群有一个停机了，集群仍然可以继续进行因为quorum是2并且2个成员仍然存活。
How this applies to membership: If a 3-member cluster has 1 downed member, it can still make forward progress because the quorum is 2 and 2 members are still live.

然而，添加一个新的机器到3个集群成员将增加quorum到3,因为4个成员的多数是3个投票请求。因为quorum增加了，这个额外的成员在容错方面一无所获。
 However, adding a new member to a 3-member cluster will increase the quorum to 3 because 3 votes are required for a majority of 4 members. Since the quorum increased, this extra member buys nothing in terms of fault tolerance; 
 集群仍然是一个无法恢复的节点故障。
 the cluster is still one node failure away from being unrecoverable.

此外，那个新的成员是危险的因为它可能配置错误或者无法加入集群。
Additionally, that new member is risky because it may turn out to be misconfigured or incapable of joining the cluster.

字那个例子里，没有方式去恢复quorum，因为集群的成员减少了2个，成员增加了两个， 但是需要3个投票才能更改成员身份才能撤销被添加的成员资格。
默认情况下，etcd将拒绝尝试添加成员，这个尝试会以这种方式关闭集群。
 In that case, there's no way to recover quorum because the cluster has two members down and two members up, but needs three votes to change membership to undo the botched membership addition. etcd will by default reject member add attempts that could take down the cluster in this manner.

另一方面，如果downed的成员首先从集群里删除，成员的个数变成了2个，这个quorum保持为2个。
On the other hand, if the downed member is removed from cluster membership first, the number of members becomes 2 and the quorum remains at 2. 

在添加新成员移除之后，quorum也将保持稳定在2.
即使无法启动新的节点，仍然可以通过剩余的活跃成员来仲裁删除新成员。
Following that removal by adding a new member will also keep the quorum steady at 2. So, even if the new node can't be brought up, it's still possible to remove the new member through quorum on the remaining live members.

### 为什么etcd不接受我修改membership？

etcd设置`strict-reconfig-check`来拒绝会导致仲裁丢失的重新配置请求
etcd sets `strict-reconfig-check` in order to reject reconfiguration requests that would cause quorum loss. 

放弃quorum是真的有风险的（尤其是当集群已经不健康了）。虽然
Abandoning quorum is really risky (especially when the cluster is already unhealthy). Although it may be tempting to disable quorum checking if there's quorum loss to add a new member, this could lead to full fledged cluster inconsistency. 
对于许多应用，这会使问题变得更糟（"disk geometry corruption"）
For many applications, this will make the problem even worse ("disk geometry corruption" being a candidate for most terrifying).

### 为什么etcd在disk latency spikes时丢失它的leader？

这是故意的；磁盘延时是领导者存活的一部分。假设集群领导花费一分钟去同步raft日志到磁盘，但是etcd集群有一秒钟的选举超时。即使领导者可以在选举期间处理网络信息（例如发送心跳），它实际上无法使用，因为它不能提交任何新的提案（不能提交日志）。
This is intentional; disk latency is part of leader liveness. Suppose the cluster leader takes a minute to fsync a raft log update to disk, but the etcd cluster has a one second election timeout. Even though the leader can process network messages within the election interval (e.g., send heartbeats), it's effectively unavailable because it can't commit any new proposals;

它在慢盘上等待。 如果集群经常因为磁盘延时丢失它的领导者，尝试使用[tuning][tuning]来设置磁盘或etcd的时间参数。
 it's waiting on the slow disk. If the cluster frequently loses its leader due to disk latencies, try [tuning][tuning] the disk settings or etcd time parameters.

### etcd的警告 "request ignored (cluster ID mismatch)" 是什么意思？

每个新的etcd集群生成一个新的基于初始化集群配置的集群ID，并且用户提供唯一的`initial-cluster-token`值。 通过拥有的位置集群ID，etcd受到保护，免受可能破坏集群的跨集群交互。
Every new etcd cluster generates a new cluster ID based on the initial cluster configuration and a user-provided unique `initial-cluster-token` value. By having unique cluster ID's, etcd is protected from cross-cluster interaction which could corrupt the cluster.

通常这个警告发生在拆毁旧的集群，然后为新集群重用peer地址。
Usually this warning happens after tearing down an old cluster, then reusing some of the peer addresses for the new cluster. 
如果旧的集群中仍然有运行etcd并且尝试去联系新的集群。这个新的集群将识别出一个集群ID匹配错误，然后忽略请求并且发送这个警告。这个警告可以通过设置集群之间的peer地址不一样来处理。
If any etcd process from the old cluster is still running it will try to contact the new cluster. The new cluster will recognize a cluster ID mismatch, then ignore the request and emit this warning. This warning is often cleared by ensuring peer addresses among distinct clusters are disjoint.

### 这个"mvcc: database space exceeded"是什么意思，并且我应该怎么修复它？

在etcd中的[multi-version concurrency control][api-mvcc]数据模型保留了keyspace的精确的历史记录。
The [multi-version concurrency control][api-mvcc] data model in etcd keeps an exact history of the keyspace.
没有定期的压缩这个历史记录（例如通过设置`--auto-compaction`），etcd最终将消耗完它的存储空间。
如果etcd的存储不足，它将发出空间配额警告来保护集群免受进一步的写入数据。
只要警告发出，etcd会向写请求响应这个错误`mvcc: database space exceeded`.
 Without periodically compacting this history (e.g., by setting `--auto-compaction`), etcd will eventually exhaust its storage space. If etcd runs low on storage space, it raises a space quota alarm to protect the cluster from further writes. So long as the alarm is raised, etcd responds to write requests with the error `mvcc: database space exceeded`.

为了恢复这个低空间配额警告：
1. [Compact][maintenance-compact] 压缩etcd的历史记录。
2. Defragment][maintenance-defragment] 对每个etcd进行碎片整理。
3. [Disarm][maintenance-disarm] 解除警报。

To recover from the low space quota alarm:

1. [Compact][maintenance-compact] etcd's history.
2. [Defragment][maintenance-defragment] every etcd endpoint.
3. [Disarm][maintenance-disarm] the alarm.

### etcd的警告 "etcdserver/api/v3rpc: transport: http2Server.HandleStreams failed to read frame: read tcp 127.0.0.1:2379->127.0.0.1:43020: read: connection reset by peer" 是什么意思？

这是gRPC-side的警告：当client-side streams被过早的关闭，服务器接收到了一个TCP RST的标志。
例如，当gRPC服务器还没有处理完在TCP queue 上的所有的HTTP/2 frame的数据，客户端关闭了连接。
server side的一些数据也许会丢失，但是只要client的连接已经关闭就可以了。
This is gRPC-side warning when a server receives a TCP RST flag with client-side streams being prematurely closed. For example, a client closes its connection, while gRPC server has not yet processed all HTTP/2 frames in the TCP queue. Some data may have been lost in server side, but it is ok so long as client connection has already been closed.

只有[old versions of gRPC](https://github.com/grpc/grpc-go/issues/1362)记录这个。
etcd[>=v3.2.13 by default log this with DEBUG level](https://github.com/etcd-io/etcd/pull/9080),因此只有启动`--debug`标志它才可见。

Only [old versions of gRPC](https://github.com/grpc/grpc-go/issues/1362) log this. etcd [>=v3.2.13 by default log this with DEBUG level](https://github.com/etcd-io/etcd/pull/9080), thus only visible with `--debug` flag enabled.

## Performance 性能

### How should I benchmark etcd? 我应该怎么benchmark etcd？

Try the [benchmark] tool. Current [benchmark results][benchmark-result] are available for comparison.

### etcd的警告 "apply entries took too long" 是什么意思？

多数etcd成员已经同意提交了一个请求， 每个etcd server申请了他的数据存储并且持久化到磁盘。即使一个慢速机械硬盘或者一个虚拟网络磁盘， 例如Amazon’s EBS or Google’s PD, 应用申请的时间通常应少于50毫秒。如果平均的申请时间持续超过100毫秒，etcd将警告“entries are taking too long to apply”。

After a majority of etcd members agree to commit a request, each etcd server applies the request to its data store and persists the result to disk. Even with a slow mechanical disk or a virtualized network disk, such as Amazon’s EBS or Google’s PD, applying a request should normally take fewer than 50 milliseconds. If the average apply duration exceeds 100 milliseconds, etcd will warn that entries are taking too long to apply.

通常这个问题是由于慢速磁盘造成的。磁盘可能在etcd和其他应用程序之间发生争论，
或者是磁盘太慢了（例如，一个shared virtualized disk）。
为了排除慢速磁盘导致此警告，检查[backend_commit_duration_seconds][backend_commit_metrics] (p99的时间应该少于25ms)
来确认磁盘速度相当快。
如果磁盘太慢了，为etcd分配专用磁盘或者使用更快的磁盘通常将解决这个问题。
Usually this issue is caused by a slow disk. The disk could be experiencing contention among etcd and other applications, or the disk is too simply slow (e.g., a shared virtualized disk). To rule out a slow disk from causing this warning, monitor  [backend_commit_duration_seconds][backend_commit_metrics] (p99 duration should be less than 25ms) to confirm the disk is reasonably fast. If the disk is too slow, assigning a dedicated disk to etcd or using faster disk will typically solve the problem.

第二个最常见的原因是CPU不足。如果发现计算机的CPU使用率太高，etcd可能没有足够的计算容量。

The second most common cause is CPU starvation. If monitoring of the machine’s CPU usage shows heavy utilization, there may not be enough compute capacity for etcd. Moving etcd to dedicated machine, increasing process resource isolation cgroups, or renicing the etcd server process into a higher priority can usually solve the problem.
将etcd移动到专用机器、增加进程资源隔离cgroup或将etcd服务器进程重新设置为更高的优先级，通常可以解决这个问题。

访问太多keys的昂贵的用户请求也会造成很长的应用延时。
但是，每次请求访问少于几百个key时，应该总是有性能的。
Expensive user requests which access too many keys (e.g., fetching the entire keyspace) can also cause long apply latencies.
Accessing fewer than a several hundred keys per request, however, should always be performant.

如果上面的建议没有清理警告，请[open an issue][new_issue]提供详细的日志记录、监视、度量和可选的工作负载信息。
If none of the above suggestions clear the warnings, please [open an issue][new_issue] with detailed logging, monitoring, metrics and optionally workload information.

### etcd的警告"failed to send out heartbeat on time"是什么意思？

etcd使用leader-based consensus protocol为了一致性数据副本和日志执行。
集群成员选择一个单独的领导，所有其他的成员变成了followers。
etcd uses a leader-based consensus protocol for consistent data replication and log execution. Cluster members elect a single leader, all other members become followers. 
选举的leader必须定期发送心跳给其他的followers来维护它的关系。如果在一个选举间隔没有心跳Followers将推断ldeader失败，然后触发一个选举。
如果一个leader不能及时发送它的心跳但是仍然在运行，这个选举是虚假的，很可能是资源不足造成的。为了捕获这些软故障，如果leader跳过两个心跳间隔，etcd将警告它未能按时发送心跳。

The elected leader must periodically send heartbeats to its followers to maintain its leadership. Followers infer leader failure if no heartbeats are received within an election interval and trigger an election. If a leader doesn’t send its heartbeats in time but is still running, the election is spurious and likely caused by insufficient resources. To catch these soft failures, if the leader skips two heartbeat intervals, etcd will warn it failed to send a heartbeat on time.

通常这是一个慢速磁盘造成的。leader在发送带metadata的心跳包之前，它也许需要去持久化metadata到磁盘。
Usually this issue is caused by a slow disk. Before the leader sends heartbeats attached with metadata, it may need to persist the metadata to disk. 
下面的和上面的问题的内容一样，不翻译了。
The disk could be experiencing contention among etcd and other applications, or the disk is too simply slow (e.g., a shared virtualized disk). To rule out a slow disk from causing this warning, monitor  [wal_fsync_duration_seconds][wal_fsync_duration_seconds] (p99 duration should be less than 10ms) to confirm the disk is reasonably fast. If the disk is too slow, assigning a dedicated disk to etcd or using faster disk will typically solve the problem. To tell whether a disk is fast enough for etcd, a benchmarking tool such as [fio][fio] can be used. Read [here][fio-blog-post] for an example.

The second most common cause is CPU starvation. If monitoring of the machine’s CPU usage shows heavy utilization, there may not be enough compute capacity for etcd. Moving etcd to dedicated machine, increasing process resource isolation  with cgroups, or renicing the etcd server process into a higher priority can usually solve the problem.

A slow network can also cause this issue. If network metrics among the etcd machines shows long latencies or high drop rate, there may not be enough network capacity for etcd. Moving etcd members to a less congested network will typically solve the problem. However, if the etcd cluster is deployed across data centers, long latency between members is expected. For such deployments, tune the `heartbeat-interval` configuration to roughly match the round trip time between the machines, and the `election-timeout` configuration to be at least 5 * `heartbeat-interval`. See [tuning documentation][tuning] for detailed information.

If none of the above suggestions clear the warnings, please [open an issue][new_issue] with detailed logging, monitoring, metrics and optionally workload information.

### etcd的警告"snapshotting is taking more than x seconds to finish ..."是什么意思

etcd发送有完整key-value的snapshot去刷新慢followers并[backups][backup].

etcd sends a snapshot of its complete key-value store to refresh slow followers and for [backups][backup]. 

慢快照传输时间增加MTTR;如果集群正在以高吞吐量接收数据，
在完成接收快照之前，慢跟随者可能需要一个新的快照来进行实时锁定。
Slow snapshot transfer times increase MTTR; if the cluster is ingesting data with high throughput, slow followers may livelock by needing a new snapshot before finishing receiving a snapshot. 
为了捕获缓慢的snapshot性能，etcd会警告当它发送snapshot超过30秒以上，并且超过了1Gbps连接的预期传输时间。


[api-mvcc]: learning/api#revisions
[backend_commit_metrics]: metrics#disk
[backup]: op-guide/recovery#snapshotting-the-keyspace
[benchmark]: https://github.com/etcd-io/etcd/tree/master/tools/benchmark
[benchmark-result]: op-guide/performance
[chubby]: http://static.googleusercontent.com/media/research.google.com/en//archive/chubby-osdi06.pdf
[fio]: https://github.com/axboe/fio
[fio-blog-post]: https://www.ibm.com/cloud/blog/using-fio-to-tell-whether-your-storage-is-fast-enough-for-etcd
[hardware-setup]: op-guide/hardware
[maintenance-compact]:  op-guide/maintenance#history-compaction-v3-api-key-value-database
[maintenance-defragment]: op-guide/maintenance#defragmentation
[maintenance-disarm]: https://github.com/etcd-io/etcd/blob/master/etcdctl/README.md#alarm-disarm
[new_issue]: https://github.com/etcd-io/etcd/issues/new
[raft]: https://raft.github.io/raft.pdf
[runtime reconfiguration]: op-guide/runtime-configuration
[supported-platform]: op-guide/supported-platform
[tuning]: tuning
[wal_fsync_duration_seconds]: metrics#disk
