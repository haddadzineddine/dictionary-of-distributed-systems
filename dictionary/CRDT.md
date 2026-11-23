---
description: Conflict-free replicated data type. A structure whose replicas can update independently and always merge to the same state.
---

Conflict-free replicated data type: a data structure whose copies can be updated independently, with no coordination, and that converge when they exchange updates. Marc Shapiro and colleagues formalized the idea in 2011. The guarantee is called strong [eventual consistency](./Eventual%20consistency.md): any two replicas (the [nodes](./Node.md) holding copies) that have received the same set of updates are in the same state.

State-based CRDTs send their state and merge it with a commutative, associative and idempotent function, so messages can repeat or arrive in any order, which suits [at-least-once delivery](./At-least-once%20delivery.md). Operation-based CRDTs send individual operations that commute when concurrent, and rely on a delivery layer that delivers each one once and, for many types, in causal order.

| Type         | Supports             | How it merges                                                                                     |
| ------------ | -------------------- | ------------------------------------------------------------------------------------------------- |
| G-Counter    | Increment            | One count per node, merged by taking each maximum                                                 |
| PN-Counter   | Increment, decrement | Two G-Counters, value is increments minus decrements                                              |
| OR-Set       | Add, remove          | Each add gets a unique tag, and remove deletes only the tags it saw, so a concurrent add survives |
| LWW-Register | Set a value          | Highest timestamp wins, as in [last-write-wins](./Last-write-wins.md)                             |

CRDTs fit collaborative editing and offline-first apps, where a user keeps working during a [network partition](./Network%20partition.md) and merges later. Automerge and Yjs are examples of libraries.

Convergence does not mean the merged result is what users wanted. Two people editing one sentence can produce an interleaved result that is valid but unreadable. A plain CRDT also cannot enforce global invariants, such as "balance never goes below zero" or "usernames are unique", because each replica accepts updates before hearing from the others. Those checks need [consensus](./Consensus.md) or quotas allocated to each replica in advance.

_Usage:_

"Can we make the inventory count a CRDT so warehouses can work offline?"

"The count would converge, but two warehouses could both sell the last unit. A plain counter CRDT can't enforce stock never going negative."
