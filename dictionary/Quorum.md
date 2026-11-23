---
description: The minimum number of nodes that must respond before an operation counts, chosen so that operations overlap.
---

The minimum number of [nodes](./Node.md) that must take part before an operation counts as successful. Quorums let a system proceed without waiting for every node, while ensuring that any two operations that need to see each other share at least one node.

The common form is a majority, floor(N/2) + 1 of N nodes. Any two majorities overlap, so two conflicting decisions cannot both gather one. This is why [Raft](./Raft.md) and [Paxos](./Paxos.md) use majorities and why a [network partition](./Network%20partition.md) can leave at most one side able to make progress.

Replicated stores often tune separate sizes: N replicas (nodes holding copies), W acknowledgments for a write, and R responses for a read. If R + W > N, every read set overlaps every write set. With N = 3:

| W   | R   | Overlap? | Node losses writes survive | Node losses reads survive |
| --- | --- | -------- | -------------------------- | ------------------------- |
| 2   | 2   | Yes      | 1                          | 1                         |
| 3   | 1   | Yes      | 0                          | 2                         |
| 1   | 3   | Yes      | 2                          | 0                         |
| 1   | 1   | No       | 2                          | 2                         |

Overlap alone does not give [linearizability](./Linearizability.md). During an in-progress write, one read can reach a replica with the new value and a later read can miss it, so values appear to go backward. Closing that gap requires readers to write the newest value back to a quorum before returning, and conflicting writes need a version order.

Cluster size determines how many failures a majority can survive. A four-node majority is three, so it tolerates one failure, the same as a three-node cluster. This is why [consensus](./Consensus.md) clusters usually have an odd size. A three-node cluster that has lost two nodes stops accepting writes, because the remaining node cannot tell whether it is on the minority side of a partition.

_Usage:_

"Two of the three etcd nodes are down and all writes are hanging. Can we force it?"

"It has lost quorum, so it's refusing on purpose. Restore a node. Forcing it risks losing committed data."
