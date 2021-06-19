---
title: Hardware recommendations 硬件推荐
weight: 4400
description: Hardware guidelines for administering etcd clusters 管理 etcd 集群的硬件引导
---

etcd通常在有限资源下运行良好，用于开发或者测试目的。
etcd usually runs well with limited resources for development or testing purposes; 

使用笔记本或者便宜的云机器来开发etcd是很常见的。然而，当etcd集群在生产中运行时，硬件引导是有用的为了合适的引导。
it’s common to develop with etcd on a laptop or a cheap cloud machine. However, when running etcd clusters in production, some hardware guidelines are useful for proper administration. 
这些建议不是硬性规则；它们是强大的生产部署的良好起点。
These suggestions are not hard rules; they serve as a good starting point for a robust production deployment. 
与往常一样，在生产中运行之前，应在模拟工作负载上进行测试部署。
As always, deployments should be tested with simulated workloads before running in production.

## CPUs

很少见etcd部署需要大量的CPU容量。典型的集群需要两个到四个才能平稳的运行。
Few etcd deployments require a lot of CPU capacity. Typical clusters need two to four cores to run smoothly.
负载较重的etcd部署，每秒为数千个客户端或数万个请求提供服务，往往受到CPU限制，因为etcd可以从内存中处理请求。这种繁重的部署通常需要8到16个专用内核。
Heavily loaded etcd deployments, serving thousands of clients or tens of thousands of requests per second, tend to be CPU bound since etcd can serve requests from memory. Such heavy deployments usually need eight to sixteen dedicated cores.


## Memory
etcd的内存占用相对较小但是他的性能仍然取决于是否有足够的内存。
etcd has a relatively small memory footprint but its performance still depends on having enough memory. 
etcd会缓存key-value数据并且花费内存的大部分来跟踪watcher。
An etcd server will aggressively cache key-value data and spends most of the rest of its memory tracking watchers. 
一般8GB是足够的。对于具有数千个watchers和数百万个key，请相应地分配16GB到64GB的内存。
Typically 8GB is enough. For heavy deployments with thousands of watchers and millions of keys, allocate 16GB to 64GB memory accordingly.


## Disks

快速磁盘是etcd部署性能和稳定性的最关键因素。
Fast disks are the most critical factor for etcd deployment performance and stability.

一个慢速的磁盘将增加增加etcd的请求延时并且潜在的损害集群稳定性。
A slow disk will increase etcd request latency and potentially hurt cluster stability. 
因为etcd的一致性协议依赖将元数据持久化存储到日志中，大多数etcd集群成员必须将每个请求写入磁盘。
Since etcd’s consensus protocol depends on persistently storing metadata to a log, a majority of etcd cluster members must write every request down to disk. 
此外，etcd还将增量地将自状态检查点写到磁盘，以便它可以截断此日志。
Additionally, etcd will also incrementally checkpoint its state to disk so it can truncate this log. 

如果这些写入消耗了太长，心跳可能会超时并且触发选举，破坏集群的稳定性。通常，判断磁盘对于etcd是否足够快，benchmarking工具例如[fio][fio]可以使用。读这个例子[here][fio-blog-post]。
If these writes take too long, heartbeats may time out and trigger an election, undermining the stability of the cluster. In general, to tell whether a disk is fast enough for etcd, a benchmarking tool such as [fio][fio] can be used. Read [here][fio-blog-post] for an example.

etcd对磁盘写入延迟是非常敏感的。通常需要50个连续的IOPS。对于负载较重的集群，500个连续的IOPS（例如，典型的本地SSD或高性能虚拟化块设备）是推荐的。 注意大部分云提供了发布并发的IOPS而不是顺序的IOPS；发布并发的IOPS比顺序IOPS大10倍。
etcd is very sensitive to disk write latency. Typically 50 sequential IOPS (e.g., a 7200 RPM disk) is required. For heavily loaded clusters, 500 sequential IOPS (e.g., a typical local SSD or a high performance virtualized block device) is recommended. Note that most cloud providers publish concurrent IOPS rather than sequential IOPS; the published concurrent IOPS can be 10x greater than the sequential IOPS. 
要衡量实际的顺序IOPS，我们推荐使用磁盘benchmarking工具，例如[diskbench][diskbench] 或者 [fio][fio]。
To measure actual sequential IOPS, we suggest using a disk benchmarking tool such as [diskbench][diskbench] or [fio][fio].

etcd只需要适度的磁盘宽带但是更多的磁盘宽带，但是，当出现故障的成员必须赶上集群时，更多的磁盘宽带可以加快恢复速度。
etcd requires only modest disk bandwidth but more disk bandwidth buys faster recovery times when a failed member has to catch up with the cluster. 
通常10MB/s将在15秒恢复100MB。
Typically 10MB/s will recover 100MB data within 15 seconds. For large clusters, 100MB/s or higher is suggested for recovering 1GB data within 15 seconds.

如果可能，使用SSD支持etcd的存储。与旋转磁盘相比，SSD通常提供更低的写入延迟和更少的差异，从而提高etcd的稳定性和可靠性。
When possible, back etcd’s storage with a SSD. A SSD usually provides lower write latencies and with less variance than a spinning disk, thus improving the stability and reliability of etcd. 
如果使用旋转磁盘，请尽可能获得最快的磁盘。使用RAID 0 也是提高高旋转磁盘和SSD磁盘速度的有效方法。
If using spinning disk, get the fastest disks possible (15,000 RPM). Using RAID 0 is also an effective way to increase disk speed, for both spinning disks and SSD. 

对于至少三个集群成员，RAID的镜像和奇偶校验变体是不必要的；
With at least three cluster members, mirroring and/or parity variants of RAID are unnecessary;
etcd的一致复制已经得到了高可用。
etcd's consistent replication already gets high availability.


## Network

多成员etcd部署收益与快速可靠的网络。为了使etcd具有一致性和分区容错性，具有分区中断的不可靠网络将导致可用性不佳。
Multi-member etcd deployments benefit from a fast and reliable network. In order for etcd to be both consistent and partition tolerant, an unreliable network with partitioning outages will lead to poor availability. 
低延迟确保etcd成员可以快速交流。高宽带可以减少恢复失败的etcd成员的时间。
Low latency ensures etcd members can communicate fast. High bandwidth can reduce the time to recover a failed etcd member. 

1GBE足以用于常见的etcd部署。对于大的etcd集群，10GbE网络将减少平均恢复时间。
1GbE is sufficient for common etcd deployments. For large etcd clusters, a 10GbE network will reduce mean time to recovery.

尽可能在三个数据中心内部署etcd成员，以避免延迟开销并减少分区事件的可能性。
Deploy etcd members within a single data center when possible to avoid latency overheads and lessen the possibility of partitioning events. 

如果需要另一个数据中的故障domain，选择一个离现有数据中心更近的数据中心。
If a failure domain in another data center is required, choose a data center closer to the existing one. 
另请阅读[tuning][tuning]文档以获取更多的有关跨数据中心部署的信息。
Please also read the [tuning][tuning] documentation for more information on cross data center deployment.


## Example hardware configurations 硬件配置例子

以下是AWS和GCE环境中的一些硬件设置示例。
Here are a few example hardware setups on AWS and GCE environments. 
正如先前所说，但无论如何必须强调，管理员应在将etcd部署投入生产之前使用模拟工作负载对其进行测试。
As mentioned before, but must be stressed regardless, administrators should test an etcd deployment with a simulated workload before putting it into production.

注意这些配置假设这些机器完全专用于etcd。在集群上的机器也许造成资源争夺并且导致资源不稳定。
Note that these configurations assume these machines are totally dedicated to etcd. Running other applications along with etcd on these machines may cause resource contentions and lead to cluster instability.

### Small cluster 小型集群

一个小型的集群服务器小于100个客户端，每秒小于200个请求，存储的数据少于100MB。
A small cluster serves fewer than 100 clients, fewer than 200 of requests per second, and stores no more than 100MB of data.

Example application workload: A 50-node Kubernetes cluster

| Provider | Type | vCPUs | Memory (GB) | Max concurrent IOPS | Disk bandwidth (MB/s) |
|----------|------|-------|--------|------|----------------|
| AWS | m4.large | 2 | 8 | 3600 | 56.25 |
| GCE | n1-standard-2 + 50GB PD SSD | 2 | 7.5 | 1500 | 25 |


### Medium cluster 中等集群

一个中型的集群服务器小于500个客户端，每秒小于1000个请求，存储的数据少于500MB。
A medium cluster serves fewer than 500 clients, fewer than 1,000 of requests per second, and stores no more than 500MB of data.

Example application workload: A 250-node Kubernetes cluster

| Provider | Type | vCPUs | Memory (GB) | Max concurrent IOPS | Disk bandwidth (MB/s) |
|----------|------|-------|--------|------|----------------|
| AWS | m4.xlarge | 4 | 16 | 6000 | 93.75 |
| GCE | n1-standard-4 + 150GB PD SSD | 4 | 15 | 4500 | 75 |


### Large cluster 大型集群

一个大型集群服务器小于1500个客户端，每秒小于10000个请求，存储的数据少于1GB。
A large cluster serves fewer than 1,500 clients, fewer than 10,000 of requests per second, and stores no more  than 1GB of data.

Example application workload: A 1,000-node Kubernetes cluster

| Provider | Type | vCPUs | Memory (GB) | Max concurrent IOPS | Disk bandwidth (MB/s) |
|----------|------|-------|--------|------|----------------|
| AWS | m4.2xlarge | 8 | 32 | 8000 | 125 |
| GCE | n1-standard-8 + 250GB PD SSD | 8 | 30 | 7500 | 125 |


### xLarge cluster 超大集群

一个超大型集群服务器大于1500个客户端，每秒大于10000个请求，存储的数据大于1GB。
An xLarge cluster serves more than 1,500 clients, more than 10,000 of requests per second, and stores more than 1GB data.

Example application workload: A 3,000 node Kubernetes cluster

| Provider | Type | vCPUs | Memory (GB) | Max concurrent IOPS | Disk bandwidth (MB/s) |
|----------|------|-------|--------|------|----------------|
| AWS | m4.4xlarge | 16 | 64 | 16,000 | 250 |
| GCE | n1-standard-16 + 500GB PD SSD | 16 | 60 | 15,000 | 250 |


[diskbench]: https://github.com/ongardie/diskbenchmark
[fio]: https://github.com/axboe/fio
[fio-blog-post]: https://www.ibm.com/cloud/blog/using-fio-to-tell-whether-your-storage-is-fast-enough-for-etcd
[tuning]: ../tuning.md
