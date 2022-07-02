## Should I add a member before removing an unhealthy member?

When replacing an etcd node, it’s important to remove the member first and then add its replacement.

etcd employs distributed consensus based on a quorum model（仲裁模型、仲裁机制）; (n/2)+1 members, a majority, must agree on a proposal before it can be committed to the cluster. These proposals include key-value updates and membership changes. This model totally avoids any possibility of split brain inconsistency（脑裂不一致性）. The downside is permanent quorum loss（仲裁丢失） is catastrophic

How this applies to membership: If a 3-member cluster has 1 downed member, it can still make forward progress because the quorum is 2 and 2 members are still live. However, adding a new member to a 3-member cluster will increase the quorum to 3 because 3 votes are required for a majority of 4 members. Since the quorum increased, this extra member buys nothing in terms of fault tolerance（从容错性角度，额外的成员并没有带来任何价值）; the cluster is still one node failure away from being unrecoverable.

Additionally, that new member is risky because it may turn out to be misconfigured or incapable of joining the cluster. In that case, there’s no way to recover quorum because the cluster has two members down and two members up, but needs three votes to change membership to undo the botched membership addition. etcd will by default reject member add attempts that could take down the cluster in this manner.

On the other hand, if the downed member is removed from cluster membership first, the number of members becomes 2 and the quorum remains at 2. Following that removal by adding a new member will also keep the quorum steady at 2. So, even if the new node can’t be brought up, it’s still possible to remove the new member through quorum on the remaining live members.



## Why won’t etcd accept my membership changes?

etcd sets `strict-reconfig-check` in order to reject reconfiguration requests that would cause quorum loss. Abandoning quorum（放弃仲裁） is really risky (especially when the cluster is already unhealthy). Although it may be tempting to disable quorum checking if there’s quorum loss to add a new member（如果添加新成员时存在仲裁丢失，集群可能会禁用仲裁检查）, this could lead to full fledged cluster inconsistency（导致完整的集群变成不一致）. For many applications, this will make the problem even worse (“disk geometry corruption” （磁盘阵列损坏）being a candidate for most terrifying).