---
description: A relaxed quorum that lets fallback nodes accept writes when designated replicas are unreachable, weakening read overlap.
---

A relaxed [quorum](./Quorum.md) in which a write can be acknowledged by any reachable [nodes](./Node.md), including nodes that are not among the key's designated replicas (the nodes assigned to hold its copies). Amazon's Dynamo paper described the technique in 2007. It keeps writes succeeding when too few of the designated replicas can be reached.

| During a failure    | Strict quorum                     | Sloppy quorum                                           |
| ------------------- | --------------------------------- | ------------------------------------------------------- |
| Who can acknowledge | Designated replicas only          | Any W reachable nodes                                   |
| Write availability  | Fails below W designated replicas | Continues while W nodes are reachable                   |
| R + W > N overlap   | Guaranteed                        | Not guaranteed                                          |
| After recovery      | Normal replication resumes        | [Hinted handoff](./Hinted%20handoff.md) moves data back |

A sloppy quorum weakens read overlap. Say a key lives on A, B and C, with N = 3, W = 2 and R = 2. During a [network partition](./Network%20partition.md), a client can reach only C, D and E. A sloppy write lands on C and D, and the client receives a success. After the partition heals, a read contacts A and B before the handoff has run. Both hold the old value, so the read returns stale data even though R + W > N.

A sloppy quorum is therefore a durability and availability measure, and it provides only [eventual consistency](./Eventual%20consistency.md) until repair completes. The data exists on W nodes, but not necessarily on nodes a reader will ask.

Products differ. Riak exposes settings that require acknowledgments from primary replicas. Cassandra stores hints for down replicas but does not count them toward its consistency levels, except the weakest level, `ANY`. Check how your store behaves before relying on quorum settings for correctness.

_Usage:_

"We write and read at quorum, so how did this read come back stale?"

"Sloppy quorum. During the outage the write landed on fallback nodes, and the read hit replicas that hadn't received the handoff yet."
