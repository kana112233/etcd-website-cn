---
title: Design of runtime reconfiguration 运行配置的设计
weight: 4650
description: The design of etcd’s runtime reconfiguration commands etcd运行时重新配置命令的设计
---

运行时重新配置是分布式系统里最难和最容易出错的功能，尤其是在基于一致性系统像etcd.
Runtime reconfiguration is one of the hardest and most error prone features in a distributed system, especially in a consensus based system like etcd.

读取学习etcd运行时配置命令以及我们怎样处理这些问题。
Read on to learn about the design of etcd's runtime reconfiguration commands and how we tackled these problems.

## Two phase config changes keep the cluster safe 两个阶段的配置更改保证了集群的安全
在etcd中，为了安全起见，每次运行时重配置必须经过[两阶段][添加成员]]
In etcd, every runtime reconfiguration has to go through [two phases][add-member] for safety reasons. 
例如，添加一个成员，首先通知集群新配置，然后启动新成员。
For example, to add a member, first inform the cluster of the new configuration and then start the new member.

阶段１ - 通知集群新的配置
Phase 1 - Inform cluster of new configuration

添加成员到etcd集群，使用API调用请求向集群添加一个新成员。这是向已经存在的集群里添加一个新的成员的唯一方式。
To add a member into an etcd cluster, make an API call to request a new member to be added to the cluster. This is the only way to add a new member into an existing cluster. 
当集群同意了配置的修改，API就会返回调用。
The API call returns when the cluster agrees on the configuration change.

Phase 2 - Start new member　阶段2 开始新的成员

为了加入新的etcd成员到已经存在的集群，尤其是正确的开始，它将首先联系现有集群然后验证当前集群配置是否匹配`initial-cluster`中指定的值。
To join the new etcd member into the existing cluster, specify the correct `initial-cluster` and set `initial-cluster-state` to `existing`. When the member starts, it will contact the existing cluster first and verify the current cluster configuration matches the expected one specified in `initial-cluster`. 
当新的成员成功开始，集群已经达到预期的配置。
When the new member successfully starts, the cluster has reached the expected configuration.

通过将过程分为两个离散的阶段，用户必须明确集群成员的变化。
By splitting the process into two discrete phases users are forced to be explicit regarding cluster membership changes. 
这个确实给用户更多的灵活，让事情更容易推理。
This actually gives users more flexibility and makes things easier to reason about. 
例如，如果尝试在集群添加已存在的ID相同的成员，在第一阶段，操作将立即失败，而不会影响正在运行的集群。
For example, if there is an attempt to add a new member with the same ID as an existing member in an etcd cluster, the action will fail immediately during phase one without impacting the running cluster. 
提供了类似的保护，以防止错误的添加新成员。
Similar protection is provided to prevent adding new members by mistake. 
如果集群接收了配置修改之前，新的etcd成员尝试加入集群，它将不被集群接收。
If a new etcd member attempts to join the cluster before the cluster has accepted the configuration change, it will not be accepted by the cluster.

如果没有集群成员资格的显示工作流，etcd将容易受到意外的集群成员资格更改的影响。
Without the explicit workflow around cluster membership etcd would be vulnerable to unexpected cluster membership changes. 
例如，如果etcd在初始化系统(如systemd)下运行,etcd将通过成员API删除后重新启动，在集群启动时　尝试重新加入集群。
For example, if etcd is running under an init system such as systemd, etcd would be restarted after being removed via the membership API, and attempt to rejoin the cluster on startup. 
每次通过API删除一个成员，并且systemd设置为在失败后重新启动etcd时，此循环都将继续，这是意外的。
This cycle would continue every time a member is removed via the API and systemd is set to restart etcd after failing, which is unexpected.

我们期望运行时重新配置是不频繁的操作。我们决定让它保持它的显示和用户驱动，以确保配置安全，并使集群始终在显示控制下平稳运行。
We expect runtime reconfiguration to be an infrequent operation. We decided to keep it explicit and user-driven to ensure configuration safety and keep the cluster always running smoothly under explicit control.

## Permanent loss of quorum requires new cluster　永久性的仲裁丢失需要新的集群

If a cluster permanently loses a majority of its members, a new cluster will need to be started from an old data directory to recover the previous state.

It is entirely possible to force removing the failed members from the existing cluster to recover. However, we decided not to support this method since it bypasses the normal consensus committing phase, which is unsafe. If the member to remove is not actually dead or force removed through different members in the same cluster, etcd will end up with a diverged cluster with same clusterID. This is very dangerous and hard to debug/fix afterwards.

With a correct deployment, the possibility of permanent majority loss is very low. But it is a severe enough problem that is worth special care. We strongly suggest reading the [disaster recovery documentation][disaster-recovery] and preparing for permanent majority loss before putting etcd into production.

## Do not use public discovery service for runtime reconfiguration

The public discovery service should only be used for bootstrapping a cluster. To join member into an existing cluster, use the runtime reconfiguration API.

The discovery service is designed for bootstrapping an etcd cluster in a cloud environment, when the IP addresses of all the members are not known beforehand. After successfully bootstrapping a cluster, the IP addresses of all the members are known. Technically, the discovery service should no longer be needed.

It seems that using public discovery service is a convenient way to do runtime reconfiguration, after all discovery service already has all the cluster configuration information. However relying on public discovery service brings troubles:

1. it introduces external dependencies for the entire life-cycle of the cluster, not just bootstrap time. If there is a network issue between the cluster and public discovery service, the cluster will suffer from it.

2. public discovery service must reflect correct runtime configuration of the cluster during its life-cycle. It has to provide security mechanisms to avoid bad actions, and it is hard.

3. public discovery service has to keep tens of thousands of cluster configurations. Our public discovery service backend is not ready for that workload.

To have a discovery service that supports runtime reconfiguration, the best choice is to build a private one.

[add-member]: runtime-configuration#add-a-new-member
[disaster-recovery]: recovery
