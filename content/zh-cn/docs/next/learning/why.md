---
title: etcd versus other key-value stores etcd和其他key-value存储
weight: 2875
description: History and use of etcd & comparison with other tools 历史和etcd的使用 & 和其他tools的比较
---

“etcd”的名字缘于两个idea，unix操作系统的文件夹"/etc"和"d"istributed systems中的“d”，组成了单词"etcd".
The name "etcd" originated from two ideas, the unix "/etc" folder and "d"istributed systems. 
"/etc"文件夹用于存储三个系统的配置信息，而etcd用于存储大规模分布式系统的配置信息。
The "/etc" folder is a place to store configuration data for a single system whereas etcd stores configuration information for large scale distributed systems.
因此，a "d"istributed "/etc" is "etcd".
 Hence, a "d"istributed "/etc" is "etcd".

etcd被设计为大型分布式系统的通用地基。
etcd is designed as a general substrate for large scale distributed systems. 

这些系统永远不会容忍脑裂操作，并且愿意牺牲可用性来达到这一目的。 
These are systems that will never tolerate split-brain operation and are willing to sacrifice availability to achieve this end.
etcd以一致性和容错的方式存储元数据。
 etcd stores metadata in a consistent and fault-tolerant way.
 etcd集群旨在提供具有最好的类稳定性、可靠性、可伸缩性和性能的key-value存储。
  An etcd cluster is meant to provide key-value storage with best of class stability, reliability, scalability and performance.

分布式系统使用etcd作为存储配置管理、服务发现和协调分布式工作的一致性key-value存储。
Distributed systems use etcd as a consistent key-value store for configuration management, service discovery, and coordinating distributed work.

一些[organizations][production-users]使用etcd来实现生产系统例如容器调度，服务发现服务和分布式存储。
 Many [organizations][production-users] use etcd to implement production systems such as container schedulers, service discovery services, and distributed data storage. 
 常见的分布式模式使用etcd包括[leader election 领导选举][etcd-etcdctl-elect], [distributed locks 分布式锁][etcd-etcdctl-lock]和监视机器的存活状态。
 Common distributed patterns using etcd include [leader election][etcd-etcdctl-elect], [distributed locks][etcd-etcdctl-lock], and monitoring machine liveness.

## Use cases 使用例子

- Container Linux by CoreOS: 应用运行在[Container Linux][container-linux]会得到自动的零停机时间的Linux内核更新。
- Container Linux by CoreOS: Applications running on [Container Linux][container-linux] get automatic, zero-downtime Linux kernel updates.

Container Linux使用[locksmith]来操作更新。Locksmith在etcd上实现了一个分布式信号量，以确保在任何给定时间只有一个集群的子集被重新引导。
 Container Linux uses [locksmith] to coordinate updates. Locksmith implements a distributed semaphore over etcd to ensure only a subset of a cluster is rebooting at any given time.

 - [Kubernetes][kubernetes]存储配置数据到etcd为了服务发现和集群配置；
- [Kubernetes][kubernetes] stores configuration data into etcd for service discovery and cluster management;
etcd的一致性是对于正确的调度和操作服务是至关重要的。
 etcd's consistency is crucial for correctly scheduling and operating services. 
 Kubernetes的API服务器将集群状态持久化到etcd中。它使用etcd的watch API来监视集群并进行关键的配置更改。
 The Kubernetes API server persists cluster state into etcd. It uses etcd's watch API to monitor the cluster and roll out critical configuration changes.

## Comparison chart 比较图

也许etcd已经看起来很合适了，但是和所有的技术选型一样，需要谨慎行事。
Perhaps etcd already seems like a good fit, but as with all technological decisions, proceed with caution. 
请注意这个文档是被etcd团队写的。虽然想法是对技术和特性进行公正的比较，但作者的专业知识和偏见显然倾向于etcd。仅按指示使用。
Please note this documentation is written by the etcd team. Although the ideal is a disinterested comparison of technology and features, the authors’ expertise and biases obviously favor etcd. Use only as directed.

下面的表格是一个方便的快速参考，可以一眼看出etcd和其他最流行的替代方案之间的差异。
The table below is a handy quick reference for spotting the differences among etcd and its most popular alternatives at a glance. 
对每一列的进一步的评论和详细信息在表格后面的章节中。
Further commentary and details for each column are in the sections following the table.

|  | etcd | ZooKeeper | Consul | NewSQL (Cloud Spanner, CockroachDB, TiDB) |
| --- | --- | --- | --- | --- |
| Concurrency Primitives |  [Lock RPCs][etcd-v3lock], [Election RPCs][etcd-v3election], [command line locks][etcd-etcdctl-lock], [command line elections][etcd-etcdctl-elect], [recipes][etcd-recipe]  in go | External [curator recipes][curator] in Java | [Native lock API][consul-lock] | [Rare][newsql-leader], if any |
| Linearizable Reads | [Yes][etcd-linread] | No | [Yes][consul-linread] | Sometimes |
| Multi-version Concurrency Control | [Yes][etcd-mvcc] | No | No | Sometimes |
| Transactions | [Field compares, Read, Write][etcd-txn] | [Version checks, Write][zk-txn] | [Field compare, Lock, Read, Write][consul-txn]  | SQL-style |
| Change Notification | [Historical and current key intervals][etcd-watch] | [Current keys and directories][zk-watch] | [Current keys and prefixes][consul-watch] | Triggers (sometimes) |
| User permissions | [Role based][etcd-rbac] | [ACLs][zk-acl] | [ACLs][consul-acl] | Varies (per-table [GRANT][cockroach-grant], per-database [roles][spanner-roles]) |
| HTTP/JSON API | [Yes][etcd-json] | No | [Yes][consul-json] | Rarely |
| Membership Reconfiguration | [Yes][etcd-reconfig] | [>3.5.0][zk-reconfig] | [Yes][consul-reconfig] | Yes |
| Maximum reliable database size | Several gigabytes | Hundreds of megabytes (sometimes several gigabytes) | Hundreds of MBs | Terabytes+ |
| Minimum read linearization latency | Network RTT | No read linearization | RTT + fsync | Clock barriers (atomic, NTP) |

### ZooKeeper

Zookeeper和etcd解决相同的问题: 分布式系统协调和元数据存储。
ZooKeeper solves the same problem as etcd: distributed system coordination and metadata storage. 
然而，etcd从ZooKeeper的设计和实现的工程和运营经验中获得了丰富的后见之明。
However, etcd has the luxury of hindsight taken from engineering and operational experience with ZooKeeper’s design and implementation. 

从Zookeeper中学到的经验无疑为etcd的设计提供了信息，帮助它支持Kubernetes这样的大型系统。
The lessons learned from Zookeeper certainly informed etcd’s design, helping it support large scale systems like Kubernetes.

etcd对Zookeeper的改进:
The improvements etcd made over Zookeeper include:

* 动态集群成员重新配置
* 高负载下的稳定的读写
* 多版本并发控制数据模型
* 可靠的key监控，从不默默的丢弃事件
* 租约原语将连接与会话的解耦
* 用于安全分布式共享锁的API

* Dynamic cluster membership reconfiguration
* Stable read/write under high load
* A multi-version concurrency control data model
* Reliable key monitoring which never silently drop events
* Lease primitives decoupling connections from sessions
* APIs for safe distributed shared locks


etcd支持各种现成的语言和框架。而Zookeeper有它自己的自定义Jute RPC protocol, 
这是唯一的，并且限制了zk的语言支持[supported language bindings][zk-bindings],
Furthermore, etcd supports a wide range of languages and frameworks out of the box. Whereas Zookeeper has its own custom Jute RPC protocol, which is totally unique to Zookeeper and limits its [supported language bindings][zk-bindings], 

etcd的客户端协议内建了[gRPC][grpc], 一个流行的支持go, C++, Java和其它语言的RPC框架。同样的，gRPC可以通过HTTP序列化成JSON，因此，即使是像`curl`这样的工具也可以和它对话。
etcd’s client protocol is built from [gRPC][grpc], a popular RPC framework with language bindings for go, C++, Java, and more. Likewise, gRPC can be serialized into JSON over HTTP, so even general command line utilities like `curl` can talk to it. 
因为系统可以从各种选择中进行选择，所以它们是用原生工具构建在etcd上的，而不是用单一的固定技术围绕着etcd。
Since systems can select from a variety of choices, they are built on etcd with native tooling rather than around etcd with a single fixed set of technologies.

当在新应用计划中需要使用Zookeeper来实现一致的键值存储，如果需要考虑特性、支持和稳定性时，最好选择etcd。
When considering features, support, and stability, new applications planning to use Zookeeper for a consistent key value store would do well to choose etcd instead.

### Consul

Consul是端到端的服务发现框架。它提供了内建了健康检查，失败检查和DNS服务。
Consul is an end-to-end service discovery framework. It provides built-in health checking, failure detection, and DNS services.
额外的，Consul暴露了一个使用RESTful的key value存储。
 In addition, Consul exposes a key value store with RESTful HTTP APIs.
 
 这个存储系统不能像其它的系统（例如etcd和zookeeper这样的key-value操作）可以扩展，
 数以百万计的key的系统将面临高延迟和内存压力。

[As it stands in Consul 1.0][dbtester-comparison-results], the storage system does not scale as well as other systems like etcd or Zookeeper in key-value operations; 
systems requiring millions of keys will suffer from high latencies and memory pressure.
这个key value API是缺失的，最明显的是多版本keys，有条件的事物，可靠的流watches。
The key value API is missing, most notably, multi-version keys, conditional transactions, and reliable streaming watches.

etcd和COnsul解决不同的问题。如果搜索分布式一致的key value存储，etcd是比Consul更好的选择。
etcd and Consul solve different problems. If looking for a distributed consistent key value store, etcd is a better choice over Consul.
如果搜索端到端集群服务发现，etcd将没有足够的功能；选择Kubernetes, Consul, or SmartStack.
 If looking for end-to-end cluster service discovery, etcd will not have enough features; choose Kubernetes, Consul, or SmartStack.

### NewSQL (Cloud Spanner, CockroachDB, TiDB)

etcd和NewSQL 数据库提供了强数据一致性保障和高可用性。
Both etcd and NewSQL databases (e.g., [Cockroach][cockroach], [TiDB][tidb], [Google Spanner][spanner]) provide strong data consistency guarantees with high availability. 

然而，显著不同的系统设计参数导致了显著不同的客户端API和性能特征。
However, the significantly different system design parameters lead to significantly different client APIs and performance characteristics.

NewSQL 数据库意味着跨数据中心的水平伸缩。
NewSQL databases are meant to horizontally scale across data centers. 
这些系统通常跨多个一致性复制组对数据进行分区，可能相距很远，以TB或以上的顺序存储数据集。
These systems typically partition data across multiple consistent replication groups (shards), potentially distant, storing data sets on the order of terabytes and above.


This sort of scaling makes them poor candidates for distributed coordination as they have long latencies from waiting on clocks and expect updates with mostly localized dependency graphs. 
数据被组织成表格，包括比etcd语义更丰富的SQL风格查询工具，但是以处理、规划和优化查询的额外复杂性为代价。
The data is organized into tables, including SQL-style query facilities with richer semantics than etcd, but at the cost of additional complexity for processing, planning, and optimizing queries.

简而言之，选择etcd是为了元数据存储和协调分布式应用。如果存储超过几GB的数据或者如果需要完整的SQL查询，请选择NewSQL数据库。
In short, choose etcd for storing metadata or coordinating distributed applications. If storing more than a few GB of data or if full SQL queries are needed, choose a NewSQL database.

## Using etcd for metadata 为了元数据使用etcd

etcd复制单个一致复制组中的所有数据。对于以一致的顺序存储多达几GB的数据，这是最有效的方法。
etcd replicates all data within a single consistent replication group. For storing up to a few GB of data with consistent ordering, this is the most efficient approach. 

集群状态的每次修改，可能会修改多个key，都会从一个单调递增的计数器中分配一个全局唯一ID，在etcd中称为revision，用于推理过度排序。
Each modification of cluster state, which may change multiple keys, is assigned a global unique ID, called a revision in etcd, from a monotonically increasing counter for reasoning over ordering. 

由于只有一个replication group, 修改请求只需要通过raft协议提交即可。
Since there’s only a single replication group, the modification request only needs to go through the raft protocol to commit. 
通过将共识限制在一个复制组，etcd通过简单的协议获得分布式一致性，同时实现低延迟和高吞吐量。
By limiting consensus to one replication group, etcd gets distributed consistency with a simple protocol while achieving low latency and high throughput.

etcd背后的复制无法横向扩展，因为它缺乏数据分片。
The replication behind etcd cannot horizontally scale because it lacks data sharding.
相比之下，NewSQL数据库通常将数据分片到多个一致的复制组，以TB级及以上的顺序存储数据集。
 In contrast, NewSQL databases usually shard data across multiple consistent replication groups, storing data sets on the order of terabytes and above. 
 但是，为了给每个修改分配一个全局唯一却不懂增加的ID，每个请求必须在复制组之间通过一个额外的协调协议。
 However, to assign each modification a global unique and increasing ID, each request must go through an additional coordination protocol among replication groups. 
 
这个额外的协调步骤可能会在全局的ID上产生潜在的冲突，迫使已排序的请求重试。
 This extra coordination step may potentially conflict on the global ID, forcing ordered requests to retry. 
 
 这个结果是一种更复杂的方法，对于严格的排序，其性能通常比etcd更差。
 The result is a more complicated approach with typically worse performance than etcd for strict ordering.

如果一个应用主要考虑元数据或元数据排序，比如协调进程，那么选择etcd。
If an application reasons primarily about metadata or metadata ordering, such as to coordinate processes, choose etcd. 
如果这个应用需要一个跨越多个数据中心的大型数据存储并且不会很强的依赖全局排序属性，选择一个NewSQL数据库。
If the application needs a large data store spanning multiple data centers and does not heavily depend on strong global ordering properties, choose a NewSQL database.

## Using etcd for distributed coordination 使用etcd为了分布式协调

etcd has distributed coordination primitives such as event watches, leases, elections, and distributed shared locks out of the box (Note that in the case of the distributed shared lock, users need to be aware about its non obvious properties. The details are described below). These primitives are both maintained and supported by the etcd developers; leaving these primitives to external libraries shirks the responsibility of developing foundational distributed software, essentially leaving the system incomplete. NewSQL databases usually expect these distributed coordination primitives to be authored by third parties. Likewise, ZooKeeper famously has a separate and independent [library][curator] of coordination recipes. Consul, which provides a native locking API, goes so far as to apologize that it’s “[not a bulletproof method][consul-bulletproof]”.

In theory, it’s possible to build these primitives atop any storage systems providing strong consistency. However, the algorithms tend to be subtle; it is easy to develop a locking algorithm that appears to work, only to suddenly break due to  thundering herd and timing skew. Furthermore, other primitives supported by etcd, such as transactional memory depend on etcd’s MVCC data model; simple strong consistency is not enough.

For distributed coordination, choosing etcd can help prevent operational headaches and save engineering effort.

### Notes on the usage of lock and lease
etcd provides [lock APIs][etcd-v3lock] which are based on [the lease mechanism][lease] and [its implementation in etcd][etcdlease]. The basic idea of the lease mechanism is: a server grants a token, which is called a lease, to a requesting client. When the server grants a lease, it associates a TTL with the lease. When the server detects the passage of time longer than the TTL, it revokes the lease. While the client holds a non revoked lease it can claim that it owns access to a resource associated with the lease. In the case of etcd, the resource is a key in the etcd keyspace. etcd provides lock APIs with this scheme. However, the lock APIs cannot be used as mutual exclusion mechanism by themselves. The APIs are called lock because [for historical reasons][chubby]. The lock APIs can, however, be used as an optimization mechanism of mutual exclusion as described below.

The most important aspect of the lease mechanism is that TTL is defined as a physical time interval. Both of the server and client measures passing of time with their own clocks. It allows a situation that the server revokes the lease but the client still claims it owns the lease.

Then how does the lease mechanism guarantees mutual exclusion of the locking mechanism? Actually, the lease mechanism itself doesn't guarantee mutual exclusion. Owning a lease cannot guarantee the owner holds a lock of the resource.

In the case of controlling mutual accesses to keys of etcd itself with etcd lock, mutual exclusion is implemented based on the mechanism of version number validation (it is sometimes called compare and swap in other systems like Consul). In etcd's RPCs like `Put` or `Txn`, we can specify required conditions about revision number and lease ID for the operations. If the conditions are not satisfied, the operation can fail. With this mechanism, etcd provides distributed locking for clients. It means that a client knows that it is acquiring a lock of a key when its requests are completed by etcd cluster successfully.

In distributed locking literature similar designs are described:
* In [the paper of Chubby][chubby], the concept of *sequencer* is introduced. We interpret that sequencer is an almost same to the combination of revision number and lease ID of etcd.
* In [How to do distributed locking][fencing], Martin Kleppmann introduced the idea of *fencing token*. The authors interpret that fencing token is revision number in the case of etcd. In [Note on fencing and distributed locks][fencing-zk] Flavio Junqueira discussed how the idea of fencing token should be implemented in the case of zookeeper.
* In [Practical Uses of Synchronized Clocks in Distributed Systems][physicalclock], we can find a description that Thor implements a distributed locking mechanism based on version number validation and lease.

Why do etcd and other systems provide lease if they provide mutual exclusion based on version number validation? Well, leases provide an optimization mechanism for reducing a number of aborted requests.

Note that in the case of etcd keys, it can be locked efficiently because of the mechanisms of lease and version number validation. If users need to protect resources which aren't related to etcd, the resources must provide the version number validation mechanism and consistency of replicas like keys of etcd. The lock feature of etcd itself cannot be used for protecting external resources.

[chubby]: https://research.google/pubs/pub27897/
[cockroach]: https://github.com/cockroachdb/cockroach
[cockroach-grant]: https://www.cockroachlabs.com/docs/stable/grant.html
[consul-acl]: https://www.consul.io/docs/security/acl
[consul-bulletproof]: https://www.consul.io/docs/dynamic-app-config/sessions
[consul-json]: https://www.consul.io/api-docs#formatted-json-output
[consul-linread]: https://www.consul.io/api-docs#consistency
[consul-lock]: https://www.consul.io/commands/lock
[consul-reconfig]: https://learn.hashicorp.com/tutorials/consul/add-remove-servers?in=consul/day-2-operations
[consul-txn]: https://www.consul.io/api/kv#txn
[consul-watch]: https://www.consul.io/docs/dynamic-app-config/watches
[container-linux]: https://coreos.com/why
[curator]: http://curator.apache.org/
[dbtester-comparison-results]: https://github.com/coreos/dbtester/tree/master/test-results/2018Q1-02-etcd-zookeeper-consul
[etcd-commonname]: ../op-guide/authentication#using-tls-common-name
[etcd-etcdctl-elect]: https://github.com/etcd-io/etcd/blob/master/etcdctl/README.md#elect-options-election-name-proposal
[etcd-etcdctl-lock]: https://github.com/etcd-io/etcd/blob/master/etcdctl/README.md#lock-options-lockname-command-arg1-arg2-
[etcd-json]: ../dev-guide/api_grpc_gateway
[etcd-linread]: api_guarantees#isolation-level-and-consistency-of-replicas
[etcd-mvcc]: data_model
[etcd-rbac]: ../op-guide/authentication#working-with-roles
[etcd-recipe]: https://godoc.org/github.com/etcd-io/etcd/client/v3/experimental/recipes
[etcd-reconfig]: ../op-guide/runtime-configuration
[etcd-txn]: api#transaction
[etcd-v3election]: https://godoc.org/github.com/etcd-io/etcd/server/etcdserver/api/v3election/v3electionpb
[etcd-v3lock]: https://godoc.org/github.com/etcd-io/etcd/server/etcdserver/api/v3lock/v3lockpb
[etcd-watch]: api#watch-streams
[etcdlease]: https://godoc.org/github.com/etcd-io/etcd/client/v3/leasing
[fencing]: https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
[fencing-zk]: https://fpj.me/2016/02/10/note-on-fencing-and-distributed-locks/
[grpc]: https://www.grpc.io
[kubernetes]: https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/
[lease]: https://web.stanford.edu/class/cs240/readings/89-leases.pdf
[locksmith]: https://github.com/coreos/locksmith
[newsql-leader]: http://dl.acm.org/citation.cfm?id=2960999
[physicalclock]: https://web.archive.org/web/20190725151657/http://www.dainf.cefetpr.br/~tacla/SDII/PracticalUseOfClocks.pdf
[production-users]: https://github.com/etcd-io/etcd/blob/master/ADOPTERS.md
[spanner]: https://cloud.google.com/spanner/
[spanner-roles]: https://cloud.google.com/spanner/docs/iam#roles
[tidb]: https://github.com/pingcap/tidb
[zk-acl]: https://zookeeper.apache.org/doc/r3.1.2/zookeeperProgrammers.html#sc_ZooKeeperAccessControl
[zk-bindings]: https://zookeeper.apache.org/doc/r3.1.2/zookeeperProgrammers.html#ch_bindings
[zk-reconfig]: https://zookeeper.apache.org/doc/current/zookeeperReconfig.html
[zk-txn]: https://zookeeper.apache.org/doc/r3.4.3/api/org/apache/zookeeper/ZooKeeper.html#multi(java.lang.Iterable)
[zk-watch]: https://zookeeper.apache.org/doc/current/zookeeperProgrammers.html#ch_zkWatches
