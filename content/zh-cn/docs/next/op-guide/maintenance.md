---
title: Maintenance 维护
weight: 4450
description: Periodic etcd cluster maintenance guide 定期etcd集群维护指南
---

## Overview 概览
etcd集群需要定期的维护以保持可靠性。
An etcd cluster needs periodic maintenance to remain reliable. 
取决于etcd应用程序的需求，这种维护通常可以自动执行，无需停机或显著的性能下降。
Depending on an etcd application's needs, this maintenance can usually be automated and performed without downtime or significantly degraded performance.

所有etcd维护管理着etcd的key间消耗的存储资源。
All etcd maintenance manages storage resources consumed by the etcd keyspace.
不能充分控制被保护的key空间大小 由存储空间配额
 Failure to adequately control the keyspace size is guarded by storage space quotas; 
 如果etcd成员空间不足，将有一个集群范围的警告警告，让系统进入操作受限制的维护模式。
 if an etcd member runs low on space, a quota will trigger cluster-wide alarms which will put the system into a limited-operation maintenance mode. 
 为了避免写入key空间时空间不足，etcd key空间历史必须压缩。
 To avoid running out of space for writes to the keyspace, the etcd keyspace history must be compacted. 
 存储空间本身可以通过对etcd成员进行碎片整理来回收。
 Storage space itself may be reclaimed by defragmenting etcd members. 
 最后，定期的快照备份让任何无意的数据丢失或操作失败造成的中断的恢复成为可能。
 Finally, periodic snapshot backups of etcd member state makes it possible to recover any unintended logical data loss or corruption caused by operational error.

## Raft log retention Raft日志保留
`etcd --snapshot-count` 配置在压缩之前保存在应用的Raft条目数。
`etcd --snapshot-count` configures the number of applied Raft entries to hold in-memory before compaction. 
当达到`--snapshot-count`数值，服务器首先持久化快照数据到磁盘。然后截断旧的条目。
When `--snapshot-count` reaches, server first persists snapshot data onto disk, and then truncates old entries. 
当缓慢的follower在压缩索引之前请求日志时，leader发送快照强制follower覆盖它的状态。
When a slow follower requests logs before a compacted index, leader sends the snapshot forcing the follower to overwrite its state.

更高的`--snapshot-count`数值将在内存保存更多的Raft条目直到发生快照，从而导致[经常性的更高的内存使用率]](https://github.com/kubernetes/kubernetes/issues/60589#issuecomment-371977156).

Higher `--snapshot-count` holds more Raft entries in memory until snapshot, thus causing [recurrent higher memory usage](https://github.com/kubernetes/kubernetes/issues/60589#issuecomment-371977156). 
因此leader保留更长的最新的Raft条目，慢速的follower在leader快照之前有更长的时间来追上leader的日志。
Since leader retains latest Raft entries for longer, a slow follower has more time to catch up before leader snapshot. 
缓慢的follower使用`--snapshot-count`这个，在高内存使用率和更好的可靠性之间有一个权衡。
`--snapshot-count` is a tradeoff between higher memory usage and better availabilities of slow followers.
从v3.2开始默认值[从10,000改变为100,000](https://github.com/etcd-io/etcd/pull/7160).
Since v3.2, the default value of `--snapshot-count` has [changed from from 10,000 to 100,000](https://github.com/etcd-io/etcd/pull/7160).
在性能方面，`--snapshot-count`超过100,000也许影响写的吞吐量。
In performance-wise, `--snapshot-count` greater than 100,000 may impact the write throughput. 
更高的内存对象可以减缓[Go GC mark phase `runtime.scanobject`](https://golang.org/src/runtime/mgc.go),不频繁的内存回收让分配变慢。
Higher number of in-memory objects can slow down [Go GC mark phase `runtime.scanobject`](https://golang.org/src/runtime/mgc.go), and infrequent memory reclamation makes allocation slow. 
不同的性能取决于工作负载和系统环境。
Performance varies depending on the workloads and system environments. 
然而，正常情况下，太频繁的压缩影响集群的可靠性和写入吞吐量。
However, in general, too frequent compaction affects cluster availabilities and write throughputs. 
看 https://www.slideshare.net/mitakeh/understanding-performance-aspects-of-etcd-and-raft 可以搜索到更多的内容。
Too infrequent compaction is also harmful placing too much pressure on Go garbage collector. See https://www.slideshare.net/mitakeh/understanding-performance-aspects-of-etcd-and-raft for more research results.

## History compaction: v3 API Key-Value Database 压缩历史： v3 API Key-Value Database

因为etcd保留其key空间的精确的历史记录，应该定期的压缩历史记录以避免性能下降和最终的存储空间消耗。
Since etcd keeps an exact history of its keyspace, this history should be periodically compacted to avoid performance degradation and eventual storage space exhaustion. 

压缩key空间历史记录删除所有关于在给定key空间修订之前被取代的key的信息。
Compacting the keyspace history drops all information about keys superseded prior to a given keyspace revision. 
然后，这些键使用的空间可用于对键空间进行额外写入。
The space used by these keys then becomes available for additional writes to the keyspace.
key空间可以使用`etcd`的窗口历史保留策略自动压缩，或者使用`etcdctl`手动压缩。
The keyspace can be compacted automatically with `etcd`'s time windowed history retention policy, or manually with `etcdctl`. 
`etcdctl`方法提供了对压缩过程的细粒度控制，而自动压缩适合只需要一段时间内关键历史记录的应用程序。
The `etcdctl` method provides fine-grained control over the compacting process whereas automatic compacting fits applications that only need key history for some length of time.

`etcdctl`启动压缩工作如下：
An `etcdctl` initiated compaction works as follows:

```sh
# compact up to revision 3
$ etcdctl compact 3
```
压缩修订之前的修订变得不可访问：
Revisions prior to the compaction revision become inaccessible:

```sh
$ etcdctl get --rev=2 somekey
Error:  rpc error: code = 11 desc = etcdserver: mvcc: required revision has been compacted
```

### Auto Compaction 自动压缩
`etcd`可以设置为使用`--auto-compaction-*`选项在几小时内自动压缩键空间：
`etcd` can be set to automatically compact the keyspace with the `--auto-compaction-*` option with a period of hours:

```sh
# keep one hour of history　保留一小时的历史记录
$ etcd --auto-compaction-retention=1
```
v3和v3.1带有`--auto-compaction-retention=10`　
每10小时在v3的key-value存储上运行定期压缩。
`--auto-compaction-retention=10`运行定期的压缩。
[v3.0.0](https://github.com/etcd-io/etcd/blob/master/CHANGELOG-3.0.md) and [v3.1.0](https://github.com/etcd-io/etcd/blob/master/CHANGELOG-3.1.md) with `--auto-compaction-retention=10` run periodic compaction on v3 key-value store for every 10-hour. 

Compactor仅仅支持定期压缩。Compactor每５分钟记录了最新的revisions，直到到达第一次压缩时期。
Compactor only supports periodic compaction. Compactor records latest revisions every 5-minute, until it reaches the first compaction period (e.g. 10-hour). 
为了保留key-value的最后压缩时期的历史记录，它使用从每５分钟收集一次的修订记录中获取的在压缩期间之前的最后一个修订。
In order to retain key-value history of last compaction period, it uses the last revision that was fetched before compaction period, from the revision records that were collected every 5-minute. 
当使用`--auto-compaction-retention=10`时，compactor使用revision100为了压缩revision,其中revision100是从10小时前获取的最新修订版。
When `--auto-compaction-retention=10`, compactor uses revision 100 for compact revision where revision 100 is the latest revision fetched from 10 hours ago. 
如果压缩成功或者请求的revision已经被压缩，它重置周期计时器并重新开始新的历史修订记录(例如，在接下来的10小时内重新启动修订版本收集和压缩)。如果压缩失败，它会在５分钟内重试。
If compaction succeeds or requested revision has already been compacted, it resets period timer and starts over with new historical revision records (e.g. restart revision collect and compact for the next 10-hour period). If compaction fails, it retries in 5 minutes.

[v3.2.0](https://github.com/etcd-io/etcd/blob/master/CHANGELOG-3.2.md) compactor runs [every hour](https://github.com/etcd-io/etcd/pull/7875). Compactor only supports periodic compaction. Compactor continues to record latest revisions every 5-minute. For every hour, it uses the last revision that was fetched before compaction period, from the revision records that were collected every 5-minute. That is, for every hour, compactor discards historical data created before compaction period. The retention window of compaction period moves to next hour. For instance, when hourly writes are 100 and `--auto-compaction-retention=10`, v3.1 compacts revision 1000, 2000, and 3000 for every 10-hour, while v3.2.x, v3.3.0, v3.3.1, and v3.3.2 compact revision 1000, 1100, and 1200 for every 1-hour. If compaction succeeds or requested revision has already been compacted, it resets period timer and removes used compacted revision from historical revision records (e.g. start next revision collect and compaction from previously collected revisions). If compaction fails, it retries in 5 minutes.

在v3.3.0，v3.3.1和v3.3.2中，`--auto-compaction-mode=revision --auto-compaction-retention=1000`
每5分钟自动在`"latest revision" - 1000`进行`Compact`(当最新的revision是30000,就在29000进行压缩)。
In [v3.3.0](https://github.com/etcd-io/etcd/blob/master/CHANGELOG-3.3.md), [v3.3.1](https://github.com/etcd-io/etcd/blob/master/CHANGELOG-3.3.md), and [v3.3.2](https://github.com/etcd-io/etcd/blob/master/CHANGELOG-3.3.md), `--auto-compaction-mode=revision --auto-compaction-retention=1000` automatically `Compact` on `"latest revision" - 1000` every 5-minute (when latest revision is 30000, compact on revision 29000). 
例如，`--auto-compaction-mode=periodic --auto-compaction-retention=72h`设置自动

For instance, `--auto-compaction-mode=periodic --auto-compaction-retention=72h` automatically `Compact` with 72-hour retention windown, for every 7.2-hour. 

For instance, `--auto-compaction-mode=periodic --auto-compaction-retention=30m` automatically `Compact` with 30-minute retention windown, for every 3-minute. Periodic compactor continues to record latest revisions for every 1/10 of given compaction period (e.g. 1-hour when `--auto-compaction-mode=periodic --auto-compaction-retention=10h`). For every 1/10 of given compaction period, compactor uses the last revision that was fetched before compaction period, to discard historical data. The retention window of compaction period moves for every 1/10 of given compaction period. For instance, when hourly writes are 100 and `--auto-compaction-retention=10`, v3.1 compacts revision 1000, 2000, and 3000 for every 10-hour, while v3.2.x, v3.3.0, v3.3.1, and v3.3.2 compact revision 1000, 1100, and 1200 for every 1-hour. Futhermore, when writes per minute are 1000, v3.3.0, v3.3.1, and v3.3.2 with `--auto-compaction-mode=periodic --auto-compaction-retention=30m` compact revision 30000, 33000, and 36000, for every 3-minute with more finer granularity.

当设置`--auto-compaction-retention=10h`，etcd首先等待10小时以进行第一次压缩，然后像这样每小时（10小时的1/10）进行一次压缩：
When `--auto-compaction-retention=10h`, etcd first waits 10-hour for the first compaction, and then does compaction every hour (1/10 of 10-hour) afterwards like this:

```
0Hr  (rev = 1)
1hr  (rev = 10)
...
8hr  (rev = 80)
9hr  (rev = 90)
10hr (rev = 100, Compact(1))
11hr (rev = 110, Compact(10))
...
```
不管是否压缩成功与否，这个过程每1/10给定的压缩周期重复一次。如果压缩成功，它只是从历史修订记录中删除压缩修订。
Whether compaction succeeds or not, this process repeats for every 1/10 of given compaction period. If compaction succeeds, it just removes compacted revision from historical revision records.

In [v3.3.3](https://github.com/etcd-io/etcd/blob/master/CHANGELOG-3.3.md), `--auto-compaction-mode=revision --auto-compaction-retention=1000` automatically `Compact` on `"latest revision" - 1000` every 5-minute (when latest revision is 30000, compact on revision 29000). Previously, `--auto-compaction-mode=periodic --auto-compaction-retention=72h` automatically `Compact` with 72-hour retention windown for every 7.2-hour.  **Now, `Compact` happens, for every 1-hour but still with 72-hour retention window.** Previously, `--auto-compaction-mode=periodic --auto-compaction-retention=30m` automatically `Compact` with 30-minute retention windown for every 3-minute. **Now, `Compact` happens, for every 30-minute but still with 30-minute retention window.** Periodic compactor keeps recording latest revisions for every compaction period when given period is less than 1-hour, or for every 1-hour when given compaction period is greater than 1-hour (e.g. 1-hour when `--auto-compaction-mode=periodic --auto-compaction-retention=24h`). For every compaction period or 1-hour, compactor uses the last revision that was fetched before compaction period, to discard historical data. The retention window of compaction period moves for every given compaction period or hour. For instance, when hourly writes are 100 and `--auto-compaction-mode=periodic --auto-compaction-retention=24h`, `v3.2.x`, `v3.3.0`, `v3.3.1`, and `v3.3.2` compact revision 2400, 2640, and 2880 for every 2.4-hour, while `v3.3.3` *or later* compacts revision 2400, 2500, 2600 for every 1-hour. Furthermore, when `--auto-compaction-mode=periodic --auto-compaction-retention=30m` and writes per minute are about 1000, `v3.3.0`, `v3.3.1`, and `v3.3.2` compact revision 30000, 33000, and 36000, for every 3-minute, while `v3.3.3` *or later* compacts revision 30000, 60000, and 90000, for every 30-minute.

## Defragmentation 碎片整理
压缩keyspace之后，后端数据库也许展示内部碎片化。
After compacting the keyspace, the backend database may exhibit internal fragmentation. 
任何内部碎片化都是后端可以免费使用但仍然消耗存储空间的空间。
Any internal fragmentation is space that is free to use by the backend but still consumes storage space. 

Compacting old revisions internally fragments `etcd` by leaving gaps in backend database. 
碎片空间可供`etcd`使用，但对主机文件系统不可用。
Fragmented space is available for use by `etcd` but unavailable to the host filesystem. 
换句话说，删除应用程序数据不会回收磁盘空间。
In other words, deleting application data does not reclaim the space on disk.
碎片整理过程将此存储空间释放会文件系统。对每个成员进行碎片整理，以便可以避免集群范围的延迟峰值。
The process of defragmentation releases this storage space back to the file system. Defragmentation is issued on a per-member so that cluster-wide latency spikes may be avoided.

对etcd成员进行碎片整理，使用`etcdctl defrag`命令：
To defragment an etcd member, use the `etcdctl defrag` command:

```sh
$ etcdctl defrag
Finished defragmenting etcd member[127.0.0.1:2379]
```

**请注意，对活动成员进行碎片整理会组织系统在重建其状态时读取和写入数据**.
**Note that defragmentation to a live member blocks the system from reading and writing data while rebuilding its states**.

**请注意，碎片整理请求不会在集群上复制。也就是说，请求只应用于本地节点。在`--endpoints` 标志或 `--cluster` 标志中指定所有成员以自动查找所有集群成员。**
**Note that defragmentation request does not get replicated over cluster. That is, the request is only applied to the local node. Specify all members in `--endpoints` flag or `--cluster` flag to automatically find all cluster members.**

为与默认端点关联的集群中的所有端点运行碎片整理操作：
Run defragment operations for all endpoints in the cluster associated with the default endpoint:

```bash
$ etcdctl defrag --cluster
Finished defragmenting etcd member[http://127.0.0.1:2379]
Finished defragmenting etcd member[http://127.0.0.1:22379]
Finished defragmenting etcd member[http://127.0.0.1:32379]
```
要在etcd未运行时直接对etcd数据目录进行碎片整理，请使用命令：
To defragment an etcd data directory directly, while etcd is not running, use the command:

``` sh
$ etcdctl defrag --data-dir <path-to-etcd-data-dir>
```

## Space quota 空间配额

`etcd`中空间配额确保集群以可靠的方式运行。
The space quota in `etcd` ensures the cluster operates in a reliable fashion. 
如果没有空间配额，如果keyspace增长过大，`etcd`可能会性能不佳，或者可能只是消耗存储空间，导致不可预测的集群行为。
Without a space quota, `etcd` may suffer from poor performance if the keyspace grows excessively large, or it may simply run out of storage space, leading to unpredictable cluster behavior. 
如果任何成员的keyspace的后端数据库超过了空间配额，`etcd`会发出集群警告，将集群至于仅接受key的读取和删除的维护模式。
If the keyspace's backend database for any member exceeds the space quota, `etcd` raises a cluster-wide alarm that puts the cluster into a maintenance mode which only accepts key reads and deletes. 
只有释放足够的空间和整理后端数据库的碎片内存后，随着空间配额警告的清除，集群才能恢复正常运行。
Only after freeing enough space in the keyspace and defragmenting the backend database, along with clearing the space quota alarm can the cluster resume normal operation.

默认情况下，`etcd`设置了适合大多数应用程序的保守空间配额，但是可以在命令行配置，以字节为单位：
By default, `etcd` sets a conservative space quota suitable for most applications, but it may be configured on the command line, in bytes:

```sh
# set a very small 16MB quota
$ etcd --quota-backend-bytes=$((16*1024*1024))
```
空间配额可以循环触发
The space quota can be triggered with a loop:

```sh
# fill keyspace
$ while [ 1 ]; do dd if=/dev/urandom bs=1024 count=1024  | ETCDCTL_API=3 etcdctl put key  || break; done
...
Error:  rpc error: code = 8 desc = etcdserver: mvcc: database space exceeded
# confirm quota space is exceeded
$ ETCDCTL_API=3 etcdctl --write-out=table endpoint status
+----------------+------------------+-----------+---------+-----------+-----------+------------+
|    ENDPOINT    |        ID        |  VERSION  | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+----------------+------------------+-----------+---------+-----------+-----------+------------+
| 127.0.0.1:2379 | bf9071f4639c75cc | 2.3.0+git | 18 MB   | true      |         2 |       3332 |
+----------------+------------------+-----------+---------+-----------+-----------+------------+
# confirm alarm is raised
$ ETCDCTL_API=3 etcdctl alarm list
memberID:13803658152347727308 alarm:NOSPACE
```

删除过多的keyspace数据并对后端数据库进行碎片整理将使集群回到配额先之内。
Removing excessive keyspace data and defragmenting the backend database will put the cluster back within the quota limits:

```sh
# get current revision
$ rev=$(ETCDCTL_API=3 etcdctl --endpoints=:2379 endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*')
# compact away all old revisions
$ ETCDCTL_API=3 etcdctl compact $rev
compacted revision 1516
# defragment away excessive space
$ ETCDCTL_API=3 etcdctl defrag
Finished defragmenting etcd member[127.0.0.1:2379]
# disarm alarm
$ ETCDCTL_API=3 etcdctl alarm disarm
memberID:13803658152347727308 alarm:NOSPACE
# test puts are allowed again
$ ETCDCTL_API=3 etcdctl put newkey 123
OK
```

这个公制`etcd_mvcc_db_total_size_in_use_in_bytes`表示历史压缩后的实际数据库使用情况，而`etcd_debugging_mvcc_db_total_size_in_bytes`显示数据库大小，包括等待碎片整理的可用空间。
The metric `etcd_mvcc_db_total_size_in_use_in_bytes` indicates the actual database usage after a history compaction, while `etcd_debugging_mvcc_db_total_size_in_bytes` shows the database size including free space waiting for defragmentation. 
后者只有在前者接近时才会增加，这意味着当这两个指标都接近配额时，需要进行历史压缩以避免触发空间配额。
The latter increases only when the former is close to it, meaning when both of these metrics are close to the quota, a history compaction is required to avoid triggering the space quota.

`etcd_debugging_mvcc_db_total_size_in_bytes` is renamed to `etcd_mvcc_db_total_size_in_bytes` from v3.4.

## Snapshot backup 快照备份

定期对`etcd`集群进行快照作为etcd的keyspace的持久备份。
Snapshotting the `etcd` cluster on a regular basis serves as a durable backup for an etcd keyspace. 
通过定期的给etcd成员的后端数据库打快照，`etcd`集群可以恢复到具有已知良好状态的时间点。
By taking periodic snapshots of an etcd member's backend database, an `etcd` cluster can be recovered to a point in time with a known good state.

用`etcdctl`打快照：
A snapshot is taken with `etcdctl`:

```sh
$ etcdctl snapshot save backup.db
$ etcdctl --write-out=table snapshot status backup.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| fe01cf57 |       10 |          7 | 2.1 MB     |
+----------+----------+------------+------------+
```
