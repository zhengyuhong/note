# Raft library

## Features

This raft implementation is a full feature implementation of Raft protocol. Features includes:

- Leader election
- Log replication
- Log compaction
- Membership changes
- Leadership transfer extension
- Efficient linearizable read-only queries（线性一致读） served by both the leader and followers
  - leader checks with quorum and bypasses Raft log before processing read-only queries
  - followers asks leader to get a safe read index before processing read-only queries
- More efficient lease-based linearizable read-only queries served by both the leader and followers
  - leader bypasses Raft log and processing read-only queries locally
  - followers asks leader to get a safe read index before processing read-only queries
  - this approach relies on the clock of the all the machines in raft group

## 线性一致读

所有的节点（Leader与Followers）都可以处理用户的读请求，但是由于以下几种原因，导致从不同的节点读数据可能会出现不一致：

1. Leader和Follower之间存在状态差：这是因为更新总是从Leader复制到Follower，因此，Follower的状态总的落后于Leader，不仅于此，Follower之间的状态也可能存在差异。从集群不同的节点上读数据，读出的结果可能不相同。
2. Raft 协议规定所有读写操作都只对 Leader 进行，但是如果旧的Leader和集群其他节点出现了网络分区，其他节点选出了新的Leader，但是旧Leader并没有感知到新的Leader，于是出现了所谓的脑裂现象，旧的Leader依然认为自己是主，但是它上面的数据已经是过时的了，如果客户端的请求分别被旧的和新的Leader分别处理，其得到的结果也会不一致。

Raft 针对这种场景的办法是 Read Index 和 Lease Read 两种机制

### ReadIndex

基本原理包含了两方面内容：

- Leader首先通过某种机制确认自己依然是Leader；
- Leader需要给客户端返回最近已应用的数据：即最新被应用到状态机的数据(AppliedIndex >= CommittedIndex)。

使用 ReadIndex，可以非常方便地提供 follower read 的功能，Follower 收到 read 请求之后，直接给 Leader 发送一个获取 ReadIndex 的命令，Leader仍然需要完成确认自己依然是Leader的步骤，然后将 ReadIndex 返回给 Follower，Follower 等到当前的状态机的 AppliedIndex 超过 ReadIndex 之后，就可以 read 然后将结果返回给 client 了。

### LeaseRead

每一次read Leader仍然需要完成确认自己依然是Leader的步骤，每次都有Heartbeat 的开销，所以我们可以考虑做更进一步的优化。在 Raft 论文里面，提到了一种通过 clock + heartbeat 的 lease read 优化方法。也就是 leader 发送 heartbeat 的时候，会首先记录一个时间点 start，当系统大部分节点都回复了 heartbeat response，那么我们就可以认为 leader 的 lease 有效期可以到 `start + election timeout - |clock drift bound|`这个时间点。在 Raft 的选举机制，因为 follower 会在至少 `election timeout` 的时间之后，才会重新发生选举，所以下一个 leader 选出来的时间一定可以保证大于 `start + election timeout - |clock drift bound|`（时钟范围）。

虽然采用 lease 的做法很高效，但仍然会面临风险问题，也就是我们有了一个预设的前提，各个服务器的 CPU clock 的时间是准的，即使有误差，也会在一个非常小的 bound 范围里面，如果各个服务器之间 clock 走的频率不一样，有些太快，有些太慢，这套 lease 机制就可能出问题。

## concept

- Revision 版本号，作为 etcd 数据的逻辑时钟

- Auth revision 鉴权操作所用的版本号，为了避免 TOCTOU 问题引入

- Propose 发起一次 Raft 请求提案

- Committed 一半以上的节点同意这次请求后的状态，此时数据可以被应用层 apply

- Apply 应用层实际将 Committed 的数据应用到 DB

- Term 任期，每进行一次 leader 选举 Term 会增加 1

- Index 单调递增，每次经过 Raft 模块发起变更操作时由 leader 增加

- CommittedIndex 经过 Raft 协议同意提交的数据 Index

- AppliedIndex 已经被应用层应用的 Index

- ConsistentIndex 为保证不重复 Apply 同一条数据引入，保证 Apply 操作的幂等性

- ReadIndex 通过 Raft 模块获取 leader 当前的 committedIndex

### AppliedIndex

etcd 的所有 apply 操作，幂等性都依赖 consistentIndex 来保证，当进行 apply 操作时，会判断当前要 apply 的 Entry 的 Index 是否大于 consistentIndex，如果 Index 大于 consistentIndex，则会将 consistentIndex 设为 Index，并允许该条记录被 apply。否则，则认为该请求被重复执行了，不会进行实际的 apply 操作。

## 引用

- https://zhuanlan.zhihu.com/p/31050303
- https://zhuanlan.zhihu.com/p/31118381
- https://zhuanlan.zhihu.com/p/27869566
- https://zhuanlan.zhihu.com/p/25367435
- https://aphyr.com/posts/313-strong-consistency-models
- https://jicki.cn/etcd-working-principle/
- https://zhuanlan.zhihu.com/p/25367435

## 扩展阅读

https://segmentfault.com/a/1190000022248118

https://pingcap.com/zh/blog/linearizability-and-raft