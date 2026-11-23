---
description: The problem of getting a group of nodes to agree on one value, and never change it, even when some of them fail.
---

The problem of getting a group of [nodes](./Node.md) to agree on a single value, even when some of them fail. Once the value is decided, it stays decided. A standard formal definition has three properties:

| Property    | Requirement                                  | Type     |
| ----------- | -------------------------------------------- | -------- |
| Agreement   | No two correct nodes decide different values | Safety   |
| Validity    | The decided value was proposed by some node  | Safety   |
| Termination | Every correct node eventually decides        | Liveness |

A safety property says nothing bad ever happens, and a liveness property says something good eventually happens. [FLP impossibility](./FLP%20impossibility.md) shows that a deterministic protocol cannot guarantee both in a fully asynchronous network with even one crash. Practical protocols such as [Paxos](./Paxos.md) and [Raft](./Raft.md) therefore always preserve safety and rely on timing assumptions only for progress.

Many coordination tasks reduce to consensus: [leader election](./Leader%20election.md), agreeing on cluster membership, deciding a transaction's outcome, and ordering commands in [state machine replication](./State%20machine%20replication.md). Repeating consensus for each slot of a log yields total order broadcast (delivering the same messages to all nodes in the same order), and the two problems are equivalent in the [crash failure](./Crash%20failure.md) model.

Tolerating f crash failures takes 2f + 1 nodes, because each decision needs a majority [quorum](./Quorum.md) and any two majorities overlap.

When a three-node etcd cluster loses two members, the survivor stops accepting writes even though it is running. It cannot tell a crash from a [network partition](./Network%20partition.md), and accepting writes could create a conflicting history. The protocol stops making progress so that it never disagrees with itself.

Each decision costs network round trips and durable disk writes, and adding voters adds to that cost. Systems therefore keep the voting group small and use it for metadata and coordination, with bulk data stored elsewhere.

_Usage:_

"Why not run a two-node etcd cluster? It's cheaper than three."

"A majority of two is two, so losing either node stops consensus. You'd pay for a second machine and tolerate zero failures."
