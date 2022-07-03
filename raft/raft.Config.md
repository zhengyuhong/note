## raft.go

### Config

```go
// Config contains the parameters to start a raft.
type Config struct {
	// ID is the identity of the local raft. ID cannot be 0.
	ID uint64

	// ElectionTick is the number of Node.Tick invocations that must pass between
	// elections. That is, if a follower does not receive any message from the
	// leader of current term before ElectionTick has elapsed, it will become
	// candidate and start an election. ElectionTick must be greater than
	// HeartbeatTick. We suggest ElectionTick = 10 * HeartbeatTick to avoid
	// unnecessary leader switching.
	ElectionTick int
	// HeartbeatTick is the number of Node.Tick invocations that must pass between
	// heartbeats. That is, a leader sends heartbeat messages to maintain its
	// leadership every HeartbeatTick ticks.
	HeartbeatTick int

	// Storage is the storage for raft. raft generates entries and states to be
	// stored in storage. raft reads the persisted entries and states out of
	// Storage when it needs. raft reads out the previous state and configuration
	// out of storage when restarting.
	Storage Storage
	// Applied is the last applied index. It should only be set when restarting
	// raft. raft will not return entries to the application smaller or equal to
	// Applied. If Applied is unset when restarting, raft might return previous
	// applied entries. This is a very application dependent configuration.
	Applied uint64

	// MaxSizePerMsg limits the max byte size of each append message. Smaller
	// value lowers the raft recovery cost(initial probing and message lost
	// during normal operation). On the other side, it might affect the
	// throughput during normal replication. Note: math.MaxUint64 for unlimited,
	// 0 for at most one entry per message.
	MaxSizePerMsg uint64
  
	// MaxCommittedSizePerReady limits the size of the committed entries which
	// can be applied.
	MaxCommittedSizePerReady uint64 // 可以应用的已提交的条目大小
	// MaxUncommittedEntriesSize limits the aggregate byte size of the
	// uncommitted entries that may be appended to a leader's log. Once this
	// limit is exceeded, proposals will begin to return ErrProposalDropped
	// errors. Note: 0 for no limit.
	MaxUncommittedEntriesSize uint64
	// MaxInflightMsgs limits the max number of in-flight append messages during
	// optimistic replication phase. The application transportation layer usually
	// has its own sending buffer over TCP/UDP. Setting MaxInflightMsgs to avoid
	// overflowing that sending buffer. TODO (xiangli): feedback to application to
	// limit the proposal rate?
	MaxInflightMsgs int // 已经发出去且未收到应答的最大消息个数

	// CheckQuorum specifies if the leader should check quorum activity. Leader
	// steps down when quorum is not active for an electionTimeout.
	CheckQuorum bool // 是否开启checkQuorum机制

	// PreVote enables the Pre-Vote algorithm described in raft thesis section
	// 9.6. This prevents disruption when a node that has been partitioned away
	// rejoins the cluster.
	PreVote bool

	// ReadOnlyOption specifies how the read only request is processed.
	//
	// ReadOnlySafe guarantees the linearizability of the read only request by
	// communicating with the quorum. It is the default and suggested option.
	//
	// ReadOnlyLeaseBased ensures linearizability of the read only request by
	// relying on the leader lease. It can be affected by clock drift.
	// If the clock drift is unbounded, leader might keep the lease longer than it
	// should (clock can move backward/pause without any bound). ReadIndex is not safe
	// in that case.
	// CheckQuorum MUST be enabled if ReadOnlyOption is ReadOnlyLeaseBased.
	ReadOnlyOption ReadOnlyOption

	// Logger is the logger used for raft log. For multinode which can host
	// multiple raft group, each raft group can have its own logger
	Logger Logger

	// DisableProposalForwarding set to true means that followers will drop
	// proposals, rather than forwarding them to the leader. One use case for
	// this feature would be in a situation where the Raft leader is used to
	// compute the data of a proposal, for example, adding a timestamp from a
	// hybrid logical clock to data in a monotonically increasing way. Forwarding
	// should be disabled to prevent a follower with an inaccurate hybrid
	// logical clock from assigning the timestamp and then forwarding the data
	// to the leader.
	DisableProposalForwarding bool
}
```



## 参考

