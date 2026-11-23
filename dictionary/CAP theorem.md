---
description: During a network partition, a replicated store cannot be both linearizable and available on every non-failing node.
---

A result stating that a replicated data store cannot guarantee both [linearizability](./Linearizability.md) and full availability while a [network partition](./Network%20partition.md) is in progress. Eric Brewer proposed it as a conjecture in 2000, and Seth Gilbert and Nancy Lynch proved a formal version in 2002 for an asynchronous network.

The letters have narrower meanings in the proof than in everyday use:

| Letter | Meaning in the proof                                                                         | Common misreading                          |
| ------ | -------------------------------------------------------------------------------------------- | ------------------------------------------ |
| C      | Consistency: reads and writes on a register are linearizable                                 | Transaction validity rules (the C in ACID) |
| A      | Availability: every request to a non-failing [node](./Node.md) gets a response               | An uptime percentage                       |
| P      | Partition tolerance: the guarantees hold even if the network loses arbitrarily many messages | A feature you can opt out of               |

Suppose a partition separates two replicas (nodes holding copies of the data). A client writes a new value to one. Another client then reads from the other. To be available, the second replica must answer, but no message can tell it about the write, so it returns the old value and linearizability is broken. If it refuses to answer, availability is broken.

The popular summary, "pick two of three", is misleading. A system running over a real network cannot opt out of partitions, so the choice applies only while one is happening. With a healthy network, a system can provide both C and A.

A majority-[quorum](./Quorum.md) database is called CP, yet clients on the majority side keep working during a partition. It fails the formal A only because minority-side nodes refuse requests. A database with asynchronous replication and automatic failover meets neither definition, since it can lose acknowledged writes and rejects requests during failover. Ask what a system does in each failure case. [PACELC](./PACELC.md) extends the framing to normal operation.

_Usage:_

"The vendor says their database beats CAP."

"Ask what a minority-side node does during a partition. It either refuses the request or risks returning stale data."
