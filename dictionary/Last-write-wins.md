---
description: A conflict rule that keeps the write with the highest timestamp and silently discards the others.
---

A conflict resolution rule that keeps the write with the highest timestamp and discards the others. When replicas (the [nodes](./Node.md) holding copies of a key) hold different versions, each compares timestamps and retains the winner, with a tie-breaker such as node ID. Every replica applies the same rule, so all converge on one value. This gives [eventual consistency](./Eventual%20consistency.md) with little machinery, and it is the default in stores such as Cassandra.

The rule's weakness is that "last" means highest timestamp, which may not be the most recent write. With [clock skew](./Clock%20skew.md), an older write can beat a newer one:

| Real order           | Node | Node's clock | Write            |
| -------------------- | ---- | ------------ | ---------------- |
| First                | A    | 10:00:00.300 | `status = "old"` |
| Second, 100 ms later | B    | 10:00:00.100 | `status = "new"` |

Node A's clock runs 300 ms ahead, so its earlier write has the higher timestamp. The store keeps `"old"`, and the write of `"new"` disappears even though its client received a success.

The rule also loses data when clocks are perfect. If two users edit a record concurrently, in the [happens-before](./Happens-before.md) sense, then neither saw the other's change, and one edit is dropped with no error. Whole-record writes make it worse: two users changing different fields overwrite each other's work.

Several measures reduce the risk:

- Store each field separately, so edits to different fields do not conflict.
- Use immutable records with unique keys, so no two writes target the same key.
- Use compare-and-set (write only if the value is unchanged) where a lost update is unacceptable.
- Use a [hybrid logical clock](./Hybrid%20logical%20clock.md), so a causally later write gets a higher timestamp.
- Use a [CRDT](./CRDT.md) or a [vector clock](./Vector%20clock.md) when concurrent updates must be preserved.

_Usage:_

"Support says two agents updated the ticket, and one note is gone."

"The store uses last-write-wins on the whole row, so the second save replaced the first. We should move notes into their own records."
