# Data-model

etcd is designed to reliably store infrequently updated data and provide reliable watch queries（观察查询）. etcd exposes previous versions of key-value pairs to support inexpensive snapshots and watch history events（观察历史事件） (“time travel queries”). A persistent, multi-version, concurrency-control data model is a good fit for these use cases.

etcd stores data in a multiversion [persistent](https://en.wikipedia.org/wiki/Persistent_data_structure) key-value store（使用多版本持久化键值存储来存储数据）. The persistent key-value store preserves the previous version of a key-value pair when its value is superseded with new data. The key-value store is effectively immutable; its operations do not update the structure in-place, but instead always generate a new updated structure. All past versions of keys are still accessible and watchable after modification. To prevent the data store from growing indefinitely over time and from maintaining old versions, the store may be compacted to shed the oldest versions of superseded data（存储应该压缩来脱离被替代的数据的最旧的版本）.

## Logic view

The store’s logical view is a flat binary key space（二进制键空间）. The key space has a lexically sorted index（语义排序索引） on byte string keys so range queries are inexpensive.

The key space maintains multiple **revisions**. When the store is created, the initial revision is 1. Each atomic mutative operation（每个原子变化操作） (e.g., a transaction operation may contain multiple operations) creates a new revision on the key space. All data held by previous revisions remains unchanged. Old versions of keys can still be accessed through previous revisions. Likewise, revisions are indexed as well; ranging over revisions with watchers is efficient. If the store is compacted to save space, revisions before the compact revision will be removed. Revisions are monotonically increasing over the lifetime of a cluster.

A key’s life spans a generation, from creation to deletion. Each key may have one or multiple generations. Creating a key increments the **version** of that key, starting at 1 if the key does not exist at the current revision. Deleting a key generates a key tombstone（墓碑）, concluding the key’s current generation by resetting its version to 0. Each modification of a key increments its version; so, versions are monotonically increasing within a key’s generation. Once a compaction happens, any generation ended before the compaction revision will be removed, and values set before the compaction revision except the latest one will be removed.

## Physical view

etcd stores the physical data as key-value pairs in a persistent [b+tree](https://en.wikipedia.org/wiki/B%2B_tree). Each revision of the store’s state only contains the delta from its previous revision to be efficient. A single revision may correspond to multiple keys in the tree.

The key of key-value pair is a 3-tuple (major, sub, type). Major is the store revision holding the key（major是持有key的存储修订版本）. Sub differentiates among keys within the same revision. Type is an optional suffix for special value (e.g., `t` if the value contains a tombstone). The value of the key-value pair contains the modification from previous revision, thus one delta from previous revision. The b+tree is ordered by key in lexical byte-order. Ranged lookups over revision deltas are fast; this enables quickly finding modifications from one specific revision to another. Compaction removes out-of-date keys-value pairs.

etcd also keeps a secondary in-memory [btree](https://en.wikipedia.org/wiki/B-tree) index to speed up range queries over keys（第二B树索引来加速key的范围查询，即3-tuple中的sub）. The keys in the btree index are the keys of the store exposed to user. The value is a pointer to the modification of the persistent b+tree. Compaction removes dead pointers.

## 参考

https://etcd.io/docs/v3.5/learning/data_model/

https://www.wenjiangs.com/doc/tmoyvrkip