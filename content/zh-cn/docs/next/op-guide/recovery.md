---
title: Disaster recovery 灾难恢复
weight: 4275
description: etcd v3 snapshot & restore facilities etcd v3 快照和恢复设施
---

etcd旨在容忍机器故障。etcd集群自动从临时故障中恢复（例如机器重启）并且对于N个成员的集群最多容忍*(N-1)/2*个永久故障。
etcd is designed to withstand machine failures. An etcd cluster automatically recovers from temporary failures (e.g., machine reboots) and tolerates up to *(N-1)/2* permanent failures for a cluster of N members. 
当成员永久失败时，无论是由于硬件故障还是磁盘损坏，它都会失去对集群的访问。
When a member permanently fails, whether due to hardware failure or disk corruption, it loses access to the cluster. 
如果集群永久丢失了超过*(N-1)/2*个成员然后它就灾难性地失败了，无法挽回的丢失了法定人数。
If the cluster permanently loses more than *(N-1)/2* members then it disastrously fails, irrevocably losing quorum. 
一旦法定人数丢失了，集群不能达成一致性并且因此不能继续接收更新。
Once quorum is lost, the cluster cannot reach consensus and therefore cannot continue accepting updates.

为了从灾难失败中恢复，etcd v3 提供了快照和恢复设施来重建没有ｖ3 key数据丢失集群。为了恢复v2的keys,参考[v2 admin guide][v2_recover]。
To recover from disastrous failure, etcd v3 provides snapshot and restore facilities to recreate the cluster without v3 key data loss. To recover v2 keys, refer to the [v2 admin guide][v2_recover].

[v2_recover]: /docs/v2.3/admin_guide#disaster-recovery

## Snapshotting the keyspace　keyspace快照

恢复集群首先需要一个快照，这个快照来自于etcd成员的keyspace.
Recovering a cluster first needs a snapshot of the keyspace from an etcd member. 
也许使用`etcdctl snapshot save`命令在存活的成员打快照或者通过从etcd数据目录复制`member/snap/db`文件。
A snapshot may either be taken from a live member with the `etcdctl snapshot save` command or by copying the `member/snap/db` file from an etcd data directory. 
例如，以下命令将`$ENDPOINT`服务的keyspace快照到文件`snapshot.db`:
For example, the following command snapshots the keyspace served by `$ENDPOINT` to the file `snapshot.db`:

```sh
$ ETCDCTL_API=3 etcdctl --endpoints $ENDPOINT snapshot save snapshot.db
```

## Restoring a cluster 恢复集群

为了恢复集群，所需要的只是一个快照文件"db"。
To restore a cluster, all that is needed is a single snapshot "db" file. 

A cluster restore with `etcdctl snapshot restore` creates new etcd data directories; all members should restore using the same snapshot. Restoring overwrites some snapshot metadata (specifically, the member ID and cluster ID); the member loses its former identity. This metadata overwrite prevents the new member from inadvertently joining an existing cluster. Therefore in order to start a cluster from a snapshot, the restore must start a new logical cluster.

Snapshot integrity may be optionally verified at restore time. If the snapshot is taken with `etcdctl snapshot save`, it will have an integrity hash that is checked by `etcdctl snapshot restore`. If the snapshot is copied from the data directory, there is no integrity hash and it will only restore by using `--skip-hash-check`.

A restore initializes a new member of a new cluster, with a fresh cluster configuration using `etcd`'s cluster configuration flags, but preserves the contents of the etcd keyspace. Continuing from the previous example, the following creates new etcd data directories (`m1.etcd`, `m2.etcd`, `m3.etcd`) for a three member cluster:

```sh
$ ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
  --name m1 \
  --initial-cluster m1=http://host1:2380,m2=http://host2:2380,m3=http://host3:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls http://host1:2380
$ ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
  --name m2 \
  --initial-cluster m1=http://host1:2380,m2=http://host2:2380,m3=http://host3:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls http://host2:2380
$ ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
  --name m3 \
  --initial-cluster m1=http://host1:2380,m2=http://host2:2380,m3=http://host3:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls http://host3:2380
```

Next, start `etcd` with the new data directories:

```sh
$ etcd \
  --name m1 \
  --listen-client-urls http://host1:2379 \
  --advertise-client-urls http://host1:2379 \
  --listen-peer-urls http://host1:2380 &
$ etcd \
  --name m2 \
  --listen-client-urls http://host2:2379 \
  --advertise-client-urls http://host2:2379 \
  --listen-peer-urls http://host2:2380 &
$ etcd \
  --name m3 \
  --listen-client-urls http://host3:2379 \
  --advertise-client-urls http://host3:2379 \
  --listen-peer-urls http://host3:2380 &
```

Now the restored etcd cluster should be available and serving the keyspace given by the snapshot.

## Restoring a cluster from membership mis-reconfiguration with wrong URLs

Previously, etcd panics on [membership mis-reconfiguration with wrong URLs](https://github.com/etcd-io/etcd/issues/9173) (v3.2.15 or later returns [error early in client-side](https://github.com/etcd-io/etcd/pull/9174) before etcd server panic).

Recommended way is restore from [snapshot](#snapshotting-the-keyspace). `--force-new-cluster` can be used to overwrite cluster membership while keeping existing application data, but is strongly discouraged because it will panic if other members from previous cluster are still alive. Make sure to save snapshot periodically.
