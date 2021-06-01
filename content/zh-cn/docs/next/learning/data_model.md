---
title: Data model 数据模型
weight: 2125
description: etcd data storage methodologies etcd数据存储方法论
---

etcd
etcd的宗旨是在可靠地存储不经常更新的数据并提供可靠的watch查询。
etcd is designed to reliably store infrequently updated data and provide reliable watch queries. 
etcd暴露先前的版本去支持廉价的snapshots并且watch历史事件(“time travel queries”)
etcd exposes previous versions of key-value pairs to support inexpensive snapshots and watch history events (“time travel queries”). 
持久的多版本并发控制数据模型非常适合这些用例。
A persistent, multi-version, concurrency-control data model is a good fit for these use cases.


etcd将数据存储在多版本[persistent][persistent-ds]key-value存储中。当永久key-value存储区的值被新数据取代时，它会保留key-value对的先前版本。
key-value对存储是不可变的；他的操作不会更新内部的结构，而是生成一个新的更新结构。修改后所有的过去的keys版本也是可以访问和watchable。为了防止数据存储随着时间无限增长并保持旧版本，可以对存储进行压缩以释放被取代的最旧的版本。

etcd stores data in a multiversion [persistent][persistent-ds] key-value store. The persistent key-value store preserves the previous version of a key-value pair when its value is superseded with new data. The key-value store is effectively immutable; its operations do not update the structure in-place, but instead always generate a new updated structure. All past versions of keys are still accessible and watchable after modification. To prevent the data store from growing indefinitely over time and from maintaining old versions, the store may be compacted to shed the oldest versions of superseded data.



### Logical view 逻辑视图

存储的逻辑视图是纯二进制key空间。key空间有一个按照字节字符串key的词法排序索引因此范围查询更inexpensive。
The store’s logical view is a flat binary key space. The key space has a lexically sorted index on byte string keys so range queries are inexpensive.

key space 会包含多个**revisions**。当存储被创建了，初始的revision是1. 每一个原子修改操作会在key space上（例如，一个transaction操作也许包含多个操作）创建一个新的revision。以前修改的所有数据保持不变。

The key space maintains multiple **revisions**. When the store is created, the initial revision is 1. Each atomic mutative operation (e.g., a transaction operation may contain multiple operations) creates a new revision on the key space. All data held by previous revisions remains unchanged. 
老版本的key仍然可以通过revisoins去访问。同样的，revisions也会被索引；范围
Old versions of key can still be accessed through previous revisions. Likewise, revisions are indexed as well;

ranging over revisions with watchers is efficient.

如果使用压缩来节省存储空间，压缩之前的revisions将被删除。Revisions在集群的生命周期里是单调递增的。
If the store is compacted to save space, revisions before the compact revision will be removed. Revisions are monotonically increasing over the lifetime of a cluster.

key的寿命是一代人，从创建到删除。
A key's life spans a generation, from creation to deletion.
每个key也许有一个或多个generations。 创建一个key增加这个key的**version**，如果在当前的revision中key不存在，这个version就从1开始。
 Each key may have one or multiple generations. Creating a key increments the **version** of that key, starting at 1 if the key does not exist at the current revision.
 
 删除一个key生成一个关键的tombstone，通过将key的版本重置为0来结束key的当前生成。
  Deleting a key generates a key tombstone, concluding the key’s current generation by resetting its version to 0.
  key每次修改都会增加它的version； 因此，在key 的一个generation内都是单调递增的。一旦压缩发生，在压缩修改版本将被删除之前结束的任何一代，
   Each modification of a key increments its version; so, versions are monotonically increasing within a key's generation. Once a compaction happens, any generation ended before the compaction revision will be removed, 
   除了最新的版本外，压缩修改之前设置的值将被删除。
   and values set before the compaction revision except the latest one will be removed.

### Physical view 物理视图

etcd存储了持久化的key-value对的物理数据。
etcd stores the physical data as key-value pairs in a persistent [b+tree][b+tree].

为了提高效率，store's state的每个revision仅包含其先前revision的增量。
 Each revision of the store’s state only contains the delta from its previous revision to be efficient. 
 一个单独的revision也许对应了tree中多个key。
 A single revision may correspond to multiple keys in the tree.

key-value对的key是一个3个元素的元祖(major, sub, type)。
The key of key-value pair is a 3-tuple (major, sub, type).
Major是保存了key的revision。
Major is the store revision holding the key. 
Sub是在同的一个revison中，区分不同的key。
Sub differentiates among  keys within the same revision. 

Type是一个可选的特殊值前缀（例如 `t`如果value包含一个tombstone）。
Type is an optional suffix for special value (e.g., `t` if the value contains a tombstone).

key-value对的value包含先前revision的修改，因此先前revision的一个增量
 The value of the key-value pair contains the modification from previous revision, thus one delta from previous revision.
 b+tree通过键以词法字节顺序排序。范围查询revision
The b+tree is ordered by key in lexical byte-order.

对revision增量的范围查找是快速的；
Ranged lookups over revision deltas are fast;
这使您能够快速地从一个特定的revision找到另一个特定revision。
this enables quickly finding modifications from one specific revision to another.
压缩删除过期的key-value对。 
Compaction removes out-of-date keys-value pairs.

etcd也有第二个内存内的btree索引用来加速范围查询。在btree索引的这些key是向用户暴露的key。这个key是一个修改了b+tree的指针。压缩会删除dead的指针。
etcd also keeps a secondary in-memory [btree][btree] index to speed up range queries over keys. The keys in the btree index are the keys of the store exposed to user. The value is a pointer to the modification of the persistent b+tree. Compaction removes dead pointers.

[b+tree]: https://en.wikipedia.org/wiki/B%2B_tree
[btree]: https://en.wikipedia.org/wiki/B-tree
[persistent-ds]: https://en.wikipedia.org/wiki/Persistent_data_structure
