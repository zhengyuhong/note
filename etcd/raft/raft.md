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

### ReadIndex

所有的节点（Leader与Followers）都可以处理用户的读请求，但是由于以下几种原因，导致从不同的节点读数据可能会出现不一致：

- Leader和Follower之间存在状态差：这是因为更新总是从Leader复制到Follower，因此，Follower的状态总的落后于Leader，不仅于此，Follower之间的状态也可能存在差异。从集群不同的节点上读数据，读出的结果可能不相同。
- 假如总是从某个特点节点读数据，一般是Leader，但是如果旧的Leader和集群其他节点出现了网络分区，其他节点选出了新的Leader，但是旧Leader并没有感知到新的Leader，于是出现了所谓的脑裂现象，旧的Leader依然认为自己是主，但是它上面的数据已经是过时的了，如果客户端的请求分别被旧的和新的Leader分别处理，其得到的结果也会不一致。

基本原理包含了两方面内容：

- Leader首先通过某种机制确认自己依然是Leader；
- Leader需要给客户端返回最近已应用的数据：即最新被应用到状态机的数据。

### LeaseRead

在 Raft 论文里面，提到了一种通过 clock + heartbeat 的 lease read 优化方法。也就是 leader 发送 heartbeat 的时候，会首先记录一个时间点 start，当系统大部分节点都回复了 heartbeat response，那么我们就可以认为 leader 的 lease 有效期可以到 `start + election timeout - |clock drift bound|`这个时间点。在 Raft 的选举机制，因为 follower 会在至少 `election timeout` 的时间之后，才会重新发生选举，所以下一个 leader 选出来的时间一定可以保证大于 `start + election timeout - |clock drift bound|`（时钟范围）。

虽然采用 lease 的做法很高效，但仍然会面临风险问题，也就是我们有了一个预设的前提，各个服务器的 CPU clock 的时间是准的，即使有误差，也会在一个非常小的 bound 范围里面，如果各个服务器之间 clock 走的频率不一样，有些太快，有些太慢，这套 lease 机制就可能出问题。

### reference

- https://zhuanlan.zhihu.com/p/31050303

- https://zhuanlan.zhihu.com/p/31118381

- https://zhuanlan.zhihu.com/p/27869566

- https://zhuanlan.zhihu.com/p/25367435

- https://aphyr.com/posts/313-strong-consistency-models

  