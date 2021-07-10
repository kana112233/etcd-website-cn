---
title: Performance 性能
weight: 4550
description: "Understanding performance: latency & throughput"　“理解性能：延迟　& 吞吐量”
---

## Understanding performance　理解性能

etcd提供了稳定的持续的高性能。
etcd provides stable, sustained high performance. 
两个因素定义性能：　延迟和吞吐量。
Two factors define performance: latency and throughput. 
延迟是完成一个操作花费的时间。吞吐量是某个时间段内完成操作的总数。
Latency is the time taken to complete an operation. Throughput is the total operations completed within some time period. 

通常，当etcd接受并发客户端请求时，平均延迟会随着整体吞吐量的增加而增加。
Usually average latency increases as the overall throughput increases when etcd accepts concurrent client requests. 
在常见的云环境中，就像Google Compute Engine (GCE)上的标准`n-4`或AWS上的类似机器类型，一个三个成员的集群在轻负载下在不到１毫秒的时间内完成一个请求，重负载下每秒可以完成30,000多个请求。
In common cloud environments, like a standard `n-4` on Google Compute Engine (GCE) or a comparable machine type on AWS, a three member etcd cluster finishes a request in less than one millisecond under light load, and can complete more than 30,000 requests per second under heavy load.

etcd使用Raft共识算法在成员之间复制请求并达成一致。一致性性能，尤其是提交延时，是由两个物理条件限制：网络IO延时和磁盘IO延时。
etcd uses the Raft consensus algorithm to replicate requests among members and reach agreement. Consensus performance, especially commit latency, is limited by two physical constraints: network IO latency and disk IO latency. 
完成etcd请求的最短时间是成员之间的网络往返时间(RTT)，加上`fdatasync`提交数据到永久存储所需的时间。
The minimum time to finish an etcd request is the network Round Trip Time (RTT) between members, plus the time `fdatasync` requires to commit the data to permanent storage. 
在数据中心里的RTT也许只要几百微妙。
The RTT within a datacenter may be as long as several hundred microseconds.
在美国，一个典型的RTT大约是50毫秒，并且大洲之间的速度能慢到400毫秒。
 A typical RTT within the United States is around 50ms, and can be as slow as 400ms between continents. 
 旋转磁盘的典型数据同步延迟是10毫秒。对于SSD,延迟经常低到１毫秒。
 The typical fdatasync latency for a spinning disk is about 10ms. For SSDs, the latency is often lower than 1ms. 
 为了增加吞吐量，etcd对多个请求进行批次里，并将他们提交给Raft.
 To increase throughput, etcd batches multiple requests together and submits them to Raft. 
 这个批处理策略让etcd在重载的情况下实现高吞吐量。
 This batching policy lets etcd attain high throughput despite heavy load.

这有其他影响etcd整体性能的子系统
There are other sub-systems which impact the overall performance of etcd.
每个序列化的etcd请求必须通过etcd的ｂoltdb支持的MVCC存储引擎运行，通常需要几十微妙才能完成。
 Each serialized etcd request must run through etcd’s boltdb-backed MVCC storage engine, which usually takes tens of microseconds to finish.
 
 etcd定期的增量快照它的当前的应用请求，合并他们到先前的磁盘快照。
  Periodically etcd incrementally snapshots its recently applied requests, merging them back with the previous on-disk snapshot. 
  这个进程也许导致延迟飙升。
  This process may lead to a latency spike. 
  通常在SSD这通常不是一个问题，它也行会使在HDD上观察到延迟加倍。
  Although this is usually not a problem on SSDs, it may double the observed latency on HDD. 
  同样的，进行中压缩可能影响etcd的性能。
  Likewise, inflight compactions can impact etcd’s performance. 
  有趣的是，这种影响通常是微不足道的，因为压缩是交错的，所以它不对与常规请求争夺资源。RPC系统，gRPC为etcd提供了一个定义良好、可扩展的API,但它也引入了额外的延迟，特别是对于本地读取。
  Fortunately, the impact is often insignificant since the compaction is staggered so it does not compete for resources with regular requests. The RPC system, gRPC, gives etcd a well-defined, extensible API, but it also introduces additional latency, especially for local reads.

## Benchmarks

Benchmarking etcd performance can be done with the [benchmark](https://github.com/etcd-io/etcd/tree/master/tools/benchmark) CLI tool included with etcd.

For some baseline performance numbers, we consider a three member etcd cluster with the following hardware configuration:

- Google Cloud Compute Engine
- 3 machines of 8 vCPUs + 16GB Memory + 50GB SSD
- 1 machine(client) of 16 vCPUs + 30GB Memory + 50GB SSD
- Ubuntu 17.04
- etcd 3.2.0, go 1.8.3

With this configuration, etcd can approximately write:

| Number of keys | Key size in bytes | Value size in bytes | Number of connections | Number of clients | Target etcd server | Average write QPS | Average latency per request | Average server RSS |
|---------------:|------------------:|--------------------:|----------------------:|------------------:|--------------------|------------------:|----------------------------:|-------------------:|
| 10,000 | 8 | 256 | 1 | 1 | leader only | 583 | 1.6ms | 48 MB |
| 100,000 | 8 | 256 | 100 | 1000 | leader only | 44,341 | 22ms |  124MB |
| 100,000 | 8 | 256 | 100 | 1000 | all members |  50,104 | 20ms |  126MB |

Sample commands are:

```sh
# write to leader
benchmark --endpoints=${HOST_1} --target-leader --conns=1 --clients=1 \
    put --key-size=8 --sequential-keys --total=10000 --val-size=256
benchmark --endpoints=${HOST_1} --target-leader  --conns=100 --clients=1000 \
    put --key-size=8 --sequential-keys --total=100000 --val-size=256

# write to all members
benchmark --endpoints=${HOST_1},${HOST_2},${HOST_3} --conns=100 --clients=1000 \
    put --key-size=8 --sequential-keys --total=100000 --val-size=256
```

Linearizable read requests go through a quorum of cluster members for consensus to fetch the most recent data. Serializable read requests are cheaper than linearizable reads since they are served by any single etcd member, instead of a quorum of members, in exchange for possibly serving stale data. etcd can read:

| Number of requests | Key size in bytes | Value size in bytes | Number of connections | Number of clients | Consistency | Average read QPS | Average latency per request |
|-------------------:|------------------:|--------------------:|----------------------:|------------------:|-------------|-----------------:|----------------------------:|
| 10,000 | 8 | 256 | 1 | 1 | Linearizable | 1,353 | 0.7ms |
| 10,000 | 8 | 256 | 1 | 1 | Serializable | 2,909 | 0.3ms |
| 100,000 | 8 | 256 | 100 | 1000 | Linearizable | 141,578 | 5.5ms |
| 100,000 | 8 | 256 | 100 | 1000 | Serializable | 185,758 | 2.2ms |

Sample commands are:

```sh
# Single connection read requests
benchmark --endpoints=${HOST_1},${HOST_2},${HOST_3} --conns=1 --clients=1 \
    range YOUR_KEY --consistency=l --total=10000
benchmark --endpoints=${HOST_1},${HOST_2},${HOST_3} --conns=1 --clients=1 \
    range YOUR_KEY --consistency=s --total=10000

# Many concurrent read requests
benchmark --endpoints=${HOST_1},${HOST_2},${HOST_3} --conns=100 --clients=1000 \
    range YOUR_KEY --consistency=l --total=100000
benchmark --endpoints=${HOST_1},${HOST_2},${HOST_3} --conns=100 --clients=1000 \
    range YOUR_KEY --consistency=s --total=100000
```

We encourage running the benchmark test when setting up an etcd cluster for the first time in a new environment to ensure the cluster achieves adequate performance; cluster latency and throughput can be sensitive to minor environment differences.
