---
description: A background process that compares replicas and copies over missing or outdated data, whether or not it is being read.
---

A background process that compares replicas (the [nodes](./Node.md) holding copies of the same data) and copies over whatever is missing or out of date. The name refers to reducing disorder between copies. It was described in work on replicated databases at Xerox PARC by Demers and colleagues in 1987.

Anti-entropy is the backstop among the convergence mechanisms in an [eventually consistent](./Eventual%20consistency.md) store, since it covers data the others miss:

| Mechanism                               | When it runs               | What it covers               | Gap                        |
| --------------------------------------- | -------------------------- | ---------------------------- | -------------------------- |
| [Hinted handoff](./Hinted%20handoff.md) | When a replica returns     | Writes during a short outage | Lost hints, long outages   |
| [Read repair](./Read%20repair.md)       | During reads               | Keys that are read           | Data nobody reads          |
| Anti-entropy                            | On a schedule or on demand | All data in scope            | Uses disk, CPU and network |

Comparing full datasets over the network would be too expensive, so implementations exchange summaries. A [Merkle tree](./Merkle%20tree.md) lets two replicas locate differing key ranges by exchanging a small number of hashes, then transfer only those ranges. Exchanges can push data, pull data, or do both.

Repair frequency has a correctness consequence around deletes. Replicated stores usually record a delete as a tombstone (a marker saying the key was deleted) and purge it after a retention period. If a replica misses the delete and repair does not run before the tombstone is purged, that replica still holds the old row and no marker exists to override it. The next repair copies the row back to the others, and deleted data reappears. In Cassandra, for this reason, repair must complete within the `gc_grace_seconds` window.

Repairs are heavy operations, so schedule them with limits and stagger them across nodes. Run one after any outage longer than the hint window, and alert when a scheduled repair has not completed.

_Usage:_

"Customers deleted those records last month, and now they're back."

"Sounds like tombstones expired before anti-entropy reached that replica. When did repair last finish on this cluster?"
