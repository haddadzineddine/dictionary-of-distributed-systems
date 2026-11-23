---
description: If updates stop, all replicas eventually converge to the same value. Nothing is promised about reads before then.
---

A consistency model which promises that if no new updates are made to a data item, all replicas ([nodes](./Node.md) holding copies of it) will eventually return the same value. It makes no statement about how long that takes or what a read returns in the meantime.

Formally this is a liveness property: it says something good will happen eventually, and no finite observation can violate it. A read that returns stale data is allowed. So is a read that returns a newer value followed by one that returns an older value. The model also says nothing about which value replicas converge on when writes conflict, so the system needs a rule such as [last-write-wins](./Last-write-wins.md) or a [CRDT](./CRDT.md).

Systems choose it because a replica can answer from local data with no coordination. Reads and writes stay fast and can continue during a [network partition](./Network%20partition.md), which is the available side of the [CAP theorem](./CAP%20theorem.md).

The cost falls on application code, with symptoms such as:

| Symptom                                          | What happened                                   |
| ------------------------------------------------ | ----------------------------------------------- |
| A saved change disappears on refresh             | The read hit a replica without the write        |
| A like count goes up and down between loads      | Requests hit replicas at different states       |
| A new account "doesn't exist" right after signup | The next service read from a lagging replica    |
| A deleted item reappears                         | A replica missed the delete and spread old data |

Replicas converge through background mechanisms such as [read repair](./Read%20repair.md), [anti-entropy](./Anti-entropy.md), [hinted handoff](./Hinted%20handoff.md) and a [gossip protocol](./Gossip%20protocol.md).

To work with the model, decide which reads can tolerate staleness. Add session guarantees where a user notices, such as [read-your-writes](./Read-your-writes.md) and [monotonic reads](./Monotonic%20reads.md), and send the few reads that must be current to a stronger path.

_Usage:_

"QA says the settings page sometimes shows the old value after saving."

"The profile store is eventually consistent and that read hits a replica. Route it to the leader for a user's own settings."
