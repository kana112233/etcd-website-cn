---
title: Runtime reconfiguration 运行时配置
weight: 4700
description: etcd incremental runtime reconfiguration support etcd增量运行时重新配置支持
---

etcd支持增量运行时重新配置，允许用户在运行时更新集群的成员关系。
etcd comes with support for incremental runtime reconfiguration, which allows users to update the membership of the cluster at run time.

只有在大部分集群成员正常工作时，才能处理重新配置请求。
Reconfiguration requests can only be processed when a majority of cluster members are functioning. 
这是强力推荐,在生产中，集群size总是要大于两个。
It is **highly recommended** to always have a cluster size greater than two in production. 
两个成员的集群删除成员是不安全的。两个成员集群的大多数是2. 如果在删除进程期间有失败，集群也许可能无法取得进展，需要从大多数失败中重新启动。
It is unsafe to remove a member from a two member cluster. The majority of a two member cluster is also two. If there is a failure during the removal process, the cluster might not be able to make progress and need to [restart from majority failure][majority failure].

为了更好的理解运行时重配置的背后设计，请读这里[the runtime reconfiguration document][runtime-reconf]。
To better understand the design behind runtime reconfiguration, please read [the runtime reconfiguration document][runtime-reconf].

## Reconfiguration use cases　重配置的使用例子

这章将介绍重新配置集群的一些常见原因。这些原因大多只设计添加或删除成员的组合，下面的[Cluster Reconfiguration Operations][cluster-reconf]中对此进行了解释。
This section will walk through some common reasons for reconfiguring a cluster. Most of these reasons just involve combinations of adding or removing a member, which are explained below under [Cluster Reconfiguration Operations][cluster-reconf].

### Cycle or upgrade multiple machines　

如果由于计划的维护（硬件升级，网络停机时间等等）需要移动多个集群成员，推荐在某个时间修改成员。
If multiple cluster members need to move due to planned maintenance (hardware upgrades, network downtime, etc.), it is recommended to modify members one at a time.

撤换leader是安全的，但是在选举过程中有一段短暂的停工期。
It is safe to remove the leader, however there is a brief period of downtime while the election process takes place. 
如果集群拥有了超过50MB的v2数据，推荐[迁移成员的数据目录]][member migration].
If the cluster holds more than 50MB of v2 data, it is recommended to [migrate the member's data directory][member migration].

### Change the cluster size　改变集群size

增加集群size可以提高[容错][fault tolerance table]并且提供更好的读性能。
Increasing the cluster size can enhance [failure tolerance][fault tolerance table] and provide better read performance. 
因为客户端可以从任何成员读取，增加成员数会增加整体序列化读取吞吐量。
Since clients can read from any member, increasing the number of members increases the overall serialized read throughput.

减少集群大小可以提高集群的写性能,但要权衡弹性的降低。
Decreasing the cluster size can improve the write performance of a cluster, with a trade-off of decreased resilience. 
写入集群中的数据在被任务已提交之前会复制到集群中的大多数成员。
Writes into the cluster are replicated to a majority of members of the cluster before considered committed. 
减少集群大小会降低majority(复制数据的最小成员个数)，并且每次吸入都会更快地提交。
Decreasing the cluster size lowers the majority, and each write is committed more quickly.

### Replace a failed machine　替换错误的机器

如果机器因硬件故障而发生故障，数据目录损坏，或者其他一些致命的情况，应该尽快更换。
If a machine fails due to hardware failure, data directory corruption, or some other fatal situation, it should be replaced as soon as possible. 

发生故障但尚未删除的计算机会对仲裁产生不利影响，并降低对其他故障的容忍度。
Machines that have failed but haven't been removed adversely affect the quorum and reduce the tolerance for an additional failure.

要更换计算机，请按照从集群中[删除成员][remove member]的说明操作，然后在其位置[添加新成员][add member]。如果集群容量超过50M,建议在仍然可以访问的情况下[迁移失败成员的数据目录][member migration]。
To replace the machine, follow the instructions for [removing the member][remove member] from the cluster, and then [add a new member][add member] in its place. If the cluster holds more than 50MB, it is recommended to [migrate the failed member's data directory][member migration] if it is still accessible.

### Restart cluster from majority failure　从大多数故障重启集群

如果大多数的集群丢失了或者所有的节点已经改变了IP地址，然后手动操作是安全恢复所必需的。
If the majority of the cluster is lost or all of the nodes have changed IP addresses, then manual action is necessary to recover safely. 

恢复过程中的基本步骤包括[使用旧的数据创建新的集群][disaster recovery],强制一个单独的成员作为leader,最后使用运行时配置像新集群[添加新成员][add member]
The basic steps in the recovery process include [creating a new cluster using the old data][disaster recovery], forcing a single member to act as the leader, and finally using runtime configuration to [add new members][add member] to this new cluster one at a time.

### Recover cluster from minority failure　从大多数恢复集群

如果某个特定的成员丢失，则相当于更换一台发生故障的机器。
If a specific member is lost, then it is equivalent to replacing a failed machine. 
步骤见[更换故障机器](runtime-configuration#replace-a-failed-machine).
The steps are mentioned in [Replace a failed machine](runtime-configuration#replace-a-failed-machine).

## Cluster reconfiguration operations　集群重新配置操作

考虑到这些用例，可以为每个用例描述所设计的操作。
With these use cases in mind, the involved operations can be described for each.

在进行任何更改之前，必须有etcd成员的简单多数(法定人数)。对于任何类型的etcd写入，这基本上都是相同的要求。
Before making any change, a simple majority (quorum) of etcd members must be available. This is essentially the same requirement for any kind of write to etcd.

必须按顺序对集群进行所所有更改：
All changes to the cluster must be done sequentially:

* To update a single member peerURLs, issue an update operation
* To replace a healthy single member, remove the old member then add a new member
* To increase from 3 to 5 members, issue two add operations
* To decrease from 5 to 3, issue two remove operations

All of these examples use the `etcdctl` command line tool that ships with etcd. To change membership without `etcdctl`, use the [v2 HTTP members API][member-api] or the [v3 gRPC members API][member-api-grpc].

### Update a member

#### Update advertise client URLs

To update the advertise client URLs of a member, simply restart that member with updated client urls flag (`--advertise-client-urls`) or environment variable (`ETCD_ADVERTISE_CLIENT_URLS`). The restarted member will self publish the updated URLs. A wrongly updated client URL will not affect the health of the etcd cluster.

#### Update advertise peer URLs

To update the advertise peer URLs of a member, first update it explicitly via member command and then restart the member. The additional action is required since updating peer URLs changes the cluster wide configuration and can affect the health of the etcd cluster.

To update the advertise peer URLs, first find the target member's ID. To list all members with `etcdctl`:

```sh
$ etcdctl member list
6e3bd23ae5f1eae0: name=node2 peerURLs=http://localhost:23802 clientURLs=http://127.0.0.1:23792
924e2e83e93f2560: name=node3 peerURLs=http://localhost:23803 clientURLs=http://127.0.0.1:23793
a8266ecf031671f3: name=node1 peerURLs=http://localhost:23801 clientURLs=http://127.0.0.1:23791
```

This example will `update` a8266ecf031671f3 member ID and change its peerURLs value to `http://10.0.1.10:2380`:

```sh
$ etcdctl member update a8266ecf031671f3 --peer-urls=http://10.0.1.10:2380
Updated member with ID a8266ecf031671f3 in cluster
```

### Remove a member

Suppose the member ID to remove is a8266ecf031671f3. Use the `remove` command to perform the removal:

```sh
$ etcdctl member remove a8266ecf031671f3
Removed member a8266ecf031671f3 from cluster
```

The target member will stop itself at this point and print out the removal in the log:

```
etcd: this member has been permanently removed from the cluster. Exiting.
```

It is safe to remove the leader, however the cluster will be inactive while a new leader is elected. This duration is normally the period of election timeout plus the voting process.

### Add a new member

Adding a member is a two step process:

 * Add the new member to the cluster via the [HTTP members API][member-api], the [gRPC members API][member-api-grpc], or the `etcdctl member add` command.
 * Start the new member with the new cluster configuration, including a list of the updated members (existing members + the new member).

`etcdctl` adds a new member to the cluster by specifying the member's [name][conf-name] and [advertised peer URLs][conf-adv-peer]:

```sh
$ etcdctl member add infra3 --peer-urls=http://10.0.1.13:2380
added member 9bf1b35fc7761a23 to cluster

ETCD_NAME="infra3"
ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra3=http://10.0.1.13:2380"
ETCD_INITIAL_CLUSTER_STATE=existing
```

`etcdctl` has informed the cluster about the new member and printed out the environment variables needed to successfully start it. Now start the new etcd process with the relevant flags for the new member:

```sh
$ export ETCD_NAME="infra3"
$ export ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra3=http://10.0.1.13:2380"
$ export ETCD_INITIAL_CLUSTER_STATE=existing
$ etcd --listen-client-urls http://10.0.1.13:2379 --advertise-client-urls http://10.0.1.13:2379 --listen-peer-urls http://10.0.1.13:2380 --initial-advertise-peer-urls http://10.0.1.13:2380 --data-dir %data_dir%
```

The new member will run as a part of the cluster and immediately begin catching up with the rest of the cluster.

If adding multiple members the best practice is to configure a single member at a time and verify it starts correctly before adding more new members. If adding a new member to a 1-node cluster, the cluster cannot make progress before the new member starts because it needs two members as majority to agree on the consensus. This behavior only happens between the time `etcdctl member add` informs the cluster about the new member and the new member successfully establishing a connection to the existing one.

#### Add a new member as learner

Starting from v3.4, etcd supports adding a new member as learner / non-voting member.
The motivation and design can be found in [design doc][design-learner].
In order to make the process of adding a new member safer,
and to reduce cluster downtime when the new member is added, it is recommended that the new member is added to cluster
as a learner until it catches up. This can be described as a three step process:

 * Add the new member as learner via [gRPC members API][member-api-grpc] or the `etcdctl member add --learner` command.

 * Start the new member with the new cluster configuration, including a list of the updated members (existing members + the new member).
 This step is exactly the same as before.

 * Promote the newly added learner to voting member via [gRPC members API][member-api-grpc] or the `etcdctl member promote` command.
 etcd server validates promote request to ensure its operational safety.
 Only after its raft log has caught up to leader’s can learner be promoted to a voting member.
 If a learner member has not caught up to leader's raft log, member promote request will fail
 (see [error cases when promoting a member] section for more details).
 In this case, user should wait and retry later.

In v3.4, etcd server limits the number of learners that cluster can have to one. The main consideration is to limit the
extra workload on leader due to propagating data from leader to learner.

Use `etcdctl member add` with flag `--learner` to add new member to cluster as learner.

```sh
$ etcdctl member add infra3 --peer-urls=http://10.0.1.13:2380 --learner
Member 9bf1b35fc7761a23 added to cluster a7ef944b95711739

ETCD_NAME="infra3"
ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra3=http://10.0.1.13:2380"
ETCD_INITIAL_CLUSTER_STATE=existing
```

After new etcd process is started for the newly added learner member, use `etcdctl member promote` to promote learner to voting member.
```
$ etcdctl member promote 9bf1b35fc7761a23
Member 9e29bbaa45d74461 promoted in cluster a7ef944b95711739
```

#### Error cases when adding members

In the following case a new host is not included in the list of enumerated nodes. If this is a new cluster, the node must be added to the list of initial cluster members.

```sh
$ etcd --name infra3 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state existing
etcdserver: assign ids error: the member count is unequal
exit 1
```

In this case, give a different address (10.0.1.14:2380) from the one used to join the cluster (10.0.1.13:2380):

```sh
$ etcd --name infra4 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra4=http://10.0.1.14:2380 \
  --initial-cluster-state existing
etcdserver: assign ids error: unmatched member while checking PeerURLs
exit 1
```

If etcd starts using the data directory of a removed member, etcd automatically exits if it connects to any active member in the cluster:

```sh
$ etcd
etcd: this member has been permanently removed from the cluster. Exiting.
exit 1
```

#### Error cases when adding a learner member

Cannot add learner to cluster if the cluster already has 1 learner (v3.4).
```
$ etcdctl member add infra4 --peer-urls=http://10.0.1.14:2380 --learner
Error: etcdserver: too many learner members in cluster
```

#### Error cases when promoting a learner member

Learner can only be promoted to voting member if it is in sync with leader.
```
$ etcdctl member promote 9bf1b35fc7761a23
Error: etcdserver: can only promote a learner member which is in sync with leader
```

Promoting a member that is not a learner will fail.
```
$ etcdctl member promote 9bf1b35fc7761a23
Error: etcdserver: can only promote a learner member
```

Promoting a member that does not exist in cluster will fail.
```
$ etcdctl member promote 12345abcde
Error: etcdserver: member not found
```


### Strict reconfiguration check mode (`-strict-reconfig-check`)

As described in the above, the best practice of adding new members is to configure a single member at a time and verify it starts correctly before adding more new members. This step by step approach is very important because if newly added members is not configured correctly (for example the peer URLs are incorrect), the cluster can lose quorum. The quorum loss happens since the newly added member are counted in the quorum even if that member is not reachable from other existing members. Also quorum loss might happen if there is a connectivity issue or there are operational issues.

For avoiding this problem, etcd provides an option `-strict-reconfig-check`. If this option is passed to etcd, etcd rejects reconfiguration requests if the number of started members will be less than a quorum of the reconfigured cluster.

It is enabled by default.

[add member]: #add-a-new-member
[cluster-reconf]: #cluster-reconfiguration-operations
[conf-adv-peer]: configuration#clustering-flags
[conf-name]: configuration#member-flags
[design-learner]: ../learning/design-learner
[disaster recovery]: recovery
[error cases when promoting a member]: #error-cases-when-promoting-a-learner-member
[fault tolerance table]: /docs/v2.3/admin_guide#fault-tolerance-table
[majority failure]: #restart-cluster-from-majority-failure
[member migration]: /docs/v2.3/admin_guide#member-migration
[member-api]: /docs/v2.3/members_api
[member-api-grpc]: ../dev-guide/api_reference_v3
[remove member]: #remove-a-member
[runtime-reconf]: runtime-reconf-design
