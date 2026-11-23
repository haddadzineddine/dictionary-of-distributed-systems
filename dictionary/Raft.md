---
description: A leader-based consensus protocol for replicating a log, designed to be easier to understand than Paxos.
---

A [consensus](./Consensus.md) protocol for maintaining a replicated log, published by Diego Ongaro and John Ousterhout in 2014 with understandability as an explicit goal. It provides fault tolerance comparable to Multi-[Paxos](./Paxos.md), and etcd and Consul use it.

Each [node](./Node.md) is in one of three roles:

| Role      | What it does                                                                     |
| --------- | -------------------------------------------------------------------------------- |
| Follower  | Responds to requests, and starts an election if it hears no leader               |
| Candidate | Increments the term, votes for itself and requests votes                         |
| Leader    | Handles client writes, replicates the log and sends [heartbeats](./Heartbeat.md) |

Time is divided into numbered terms, and each term has at most one leader. A follower whose randomized election timeout expires becomes a candidate. It wins with votes from a majority [quorum](./Quorum.md), and each node casts at most one vote per term, recorded on disk. A voter refuses any candidate whose log is less up to date than its own, so a new leader always holds every committed entry.

The leader appends each command to its log and sends it to followers with AppendEntries messages, which include the index and term of the preceding entry so followers can reject mismatches. An entry from the current term is committed once a majority stores it. Entries from earlier terms become committed only indirectly, when a later entry from the current term commits. Committed entries are applied in order, as in [state machine replication](./State%20machine%20replication.md).

A common operational problem is election churn. If disk or network latency delays heartbeats past the election timeout, followers keep starting elections, and writes stall even though every node is up. Raising the timeout or moving the log to a faster disk usually helps.

A deposed leader may not know it has been replaced, so linearizable reads must confirm leadership first.

_Usage:_

"etcd keeps logging leader changes, but all three members are up."

"Check disk sync latency. If the disk stalls, Raft heartbeats miss the election timeout and followers call new elections."
