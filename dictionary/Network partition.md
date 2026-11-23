---
description: A network fault that splits nodes into groups that cannot reach each other while each group keeps running.
---

A network fault that divides [nodes](./Node.md) into groups that cannot exchange messages across the divide, while communication within each group continues. Every node may be healthy and still serving requests. The cause can be a failed switch, a bad firewall rule, a routing change, or a lost link between data centers.

From inside one group, a partition looks the same as the other group having crashed: messages stop arriving and [heartbeats](./Heartbeat.md) time out. This ambiguity produces a typical incident. Each side concludes that the other is dead, both keep accepting writes, and when the network heals the data has diverged. That outcome is [split brain](./Split%20brain.md).

Partitions are not always clean:

| Shape        | What happens                                    |
| ------------ | ----------------------------------------------- |
| Complete     | Groups exchange no messages in either direction |
| Asymmetric   | A can reach B, but B cannot reach A             |
| Partial      | A and B cannot talk, but both can still reach C |
| Intermittent | Links drop and recover repeatedly               |

The asymmetric and partial shapes are harder to handle because nodes disagree about who is reachable. A node may stop receiving a leader's messages while its own messages still get through, and may then start elections that disrupt the cluster.

During a partition, a system holding replicated data has to make the choice described by the [CAP theorem](./CAP%20theorem.md). It can refuse some requests to preserve consistency, usually by letting only the side with a [quorum](./Quorum.md) proceed. Or it can keep serving on both sides and reconcile afterwards with a rule such as [last-write-wins](./Last-write-wins.md) or a data structure such as a [CRDT](./CRDT.md). The choice should be made per use case and tested by injecting partitions, because the code paths involved rarely run in normal operation.

_Usage:_

"Both regions look healthy on their own dashboards, but replication lag keeps climbing."

"That looks like a network partition between them. Check which side has quorum before anyone fails over, or we'll end up with two leaders."
