---
description: All nodes see causally related operations in the same order, while concurrent operations may appear in different orders.
---

A consistency model in which every [node](./Node.md) sees causally related operations in the same order, while concurrent operations may be seen in different orders. Two operations are causally related if one could have influenced the other, as defined by the [happens-before](./Happens-before.md) relation. A client's earlier operations precede its later ones, and a write precedes any read that observes it.

Consider a comment thread. Ana posts a question (W1). Ben reads it and posts an answer (W2). Cleo, who has seen neither, posts an unrelated comment (W3). W1 causally precedes W2, and W3 is concurrent with both.

| Order seen by a replica | Allowed? | Reason                                  |
| ----------------------- | -------- | --------------------------------------- |
| W1, W2, W3              | Yes      | W1 is before W2                         |
| W3, W1, W2              | Yes      | W3 is concurrent, so it can go anywhere |
| W1, W3, W2              | Yes      | W1 is before W2                         |
| W2, W1, W3              | No       | The answer appears before its question  |

This model prevents the anomaly in which a reply shows up with no visible question, or a user removes a colleague from a shared album and then posts a photo, only for the colleague to see the photo because the two updates arrived in the opposite order.

Causal consistency sits between [eventual consistency](./Eventual%20consistency.md) and [sequential consistency](./Sequential%20consistency.md). It includes the session guarantees [read-your-writes](./Read-your-writes.md) and [monotonic reads](./Monotonic%20reads.md). Unlike [linearizability](./Linearizability.md), it can keep serving during a [network partition](./Network%20partition.md), provided each client stays with a replica (a node holding a copy of the data) that has the client's history.

Implementations track dependencies, often with a [vector clock](./Vector%20clock.md) or explicit lists of the writes each update depends on, and delay applying an update until its dependencies have arrived. Concurrent writes to the same key still conflict, so the system also needs a merge rule.

_Usage:_

"Some users see replies to comments that don't exist yet."

"Our replicas apply writes in arrival order. We need causal consistency, so a reply waits until its parent comment is there."
