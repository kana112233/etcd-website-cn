---
title: Glossary
weight: 2900
description: Terms used in etcd documentation, command line, and source code
---

This document defines the various terms used in etcd documentation, command line and source code.
本文档定义了etcd文档、命令行和源代码中使用的各种术语。

## Alarm 
每当集群需要操作员干预以保持可靠性时，etcd服务器就会发出警告。
The etcd server raises an alarm whenever the cluster needs operator intervention to remain reliable.

## Authentication
管理用户访问etcd资源的权限
Authentication manages user access permissions for etcd resources.

## Client
client连接etcd集群， 发送服务请求例如获取key-value对，写数据或者watch修改。
A client connects to the etcd cluster to issue service requests such as fetching key-value pairs, writing data, or watching for updates.

## Cluster
集群由多个成员组成。
Cluster consists of several members.

每个成员中的节点遵循raft共识协议来复制日志。
集群从成员中接收提议，提交他们和应用到本地存储。
The node in each member follows raft consensus protocol to replicate logs. Cluster receives proposals from members, commits them and apply to local store.

## Compaction 压缩

压缩丢弃所有的etcd历史事件，然后在给定的revision来替换先前的键。
Compaction discards all etcd event history and superseded keys prior to a given revision. 
它用于收回etcd后端数据库中的存储空间。
It is used to reclaim storage space in the etcd backend database.

## Election 选择
etcd集群在其成员中进行选举，使用raft一致性协议来选择一个领导。
The etcd cluster holds elections among its members to choose a leader as part of the raft consensus protocol.

## Endpoint 

一个指向etcd服务和资源的URL。
A URL pointing to an etcd service or resource.

## Key
用户定义的ID，用户在etcd中存储和检索用户定义的值。
A user-defined identifier for storing and retrieving user-defined values in etcd.

## Key range

A set of keys containing either an individual key, a lexical interval for all x such that a < x <= b, or all keys greater than a given key.

## Keyspace

在etcd集群中所有keys的集合。
The set of all keys in an etcd cluster.

## Lease
一个寿命较短的可更新合约，它在到期时删除与其相关的key。
A short-lived renewable contract that deletes keys associated with it on its expiry.

## Member
一个在etcd集群参与服务的逻辑上etcd server。
A logical etcd server that participates in serving an etcd cluster.

## Modification Revision
对给定key进行最后一次写入的第一个revision。
The first revision to hold the last write to a given key.

## Peer
Peer是相同集群上另一个成员。
Peer is another member of the same cluster.

## Proposal 提议 提案
proposal是需要通过raft协议的请求（例如写请求、配置更改请求）。
A proposal is a request (for example a write request, a configuration change request) that needs to go through raft protocol.

## Quorum 法定人数

修改集群的一致性状态所需要的活跃成员数。 etcd需要一半以上的人数到达法定人数。
The number of active members needed for consensus to modify the cluster state. etcd requires a member majority to reach quorum.

## Revision
一个64位的集群范围的计数器，从1开始，每次修改keysparce时递增。
A 64-bit cluster-wide counter that starts at 1 and is incremented each time the keyspace is modified.

## Role 角色

一组key range上的权限单位，可以授予一组用户来进行访问控制。
A unit of permissions over a set of key ranges which may be granted to a set of users for access control.

## Snapshot
etcd集群状态的备份时间点。
A point-in-time backup of the etcd cluster state.

## Store
集群keyspace的物理存储。
The physical storage backing the cluster keyspace.

## Transaction 事物
一组原子执行的操作。在一个事物里所有修改的keys共享了相同的修改revision。
An atomically executed set of operations. All modified keys in a transaction share the same modification revision.

## Key Version
从创建key以来对key的写操作的次数，从1开始。不存在的或者删除的key是0.
The number of writes to a key since it was created, starting at 1. The version of a nonexistent or deleted key is 0.

## Watcher
一个client打开一个watcher来监视key range的修改。
A client opens a watcher to observe updates on a given key range.
