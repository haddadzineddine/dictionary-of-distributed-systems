---
description: Each operation appears to take effect at one instant between its start and end, in an order that respects real time.
---

A consistency model in which every operation appears to take effect atomically at a single instant between its invocation and its response. Maurice Herlihy and Jeannette Wing defined it in 1990. To clients, the object behaves as if one copy of the data existed, even with several replicas ([nodes](./Node.md) holding copies).

Formally, a history of operations is linearizable if they can be arranged in one total order that meets two conditions. The order must be legal for the object, so each read returns the latest value written before it. It must also respect real time: if operation A completes before operation B starts, A comes before B. Here `x` starts at 0:

| Time | Client A              | Client B                 | Client C                |
| ---- | --------------------- | ------------------------ | ----------------------- |
| 1    | `write(x, 1)` starts  |                          |                         |
| 2    |                       | `read(x)` returns 0 or 1 |                         |
| 3    | `write(x, 1)` returns |                          |                         |
| 4    |                       |                          | `read(x)` must return 1 |

A read that overlaps a write may return either value. Once any read returns the new value, every read starting after it must return that value or a later one.

It is defined per operation on a single object, unlike serializability (transactions appear to run one at a time, with no real-time requirement). The combination of the two is called strict serializability. Linearizability is also composable: if each object is linearizable, the system of objects is too. [Sequential consistency](./Sequential%20consistency.md) lacks this property.

Without linearizability, a client can update a value, tell another client, and that client can still read the old value from a lagging replica. Locks, [leader election](./Leader%20election.md) and uniqueness checks all rely on this guarantee. Providing it requires coordination, usually [consensus](./Consensus.md), which adds latency. By the [CAP theorem](./CAP%20theorem.md), it also cannot remain available on both sides of a [network partition](./Network%20partition.md).

_Usage:_

"I acquired the lock, but another worker read it as free a moment later."

"Then those reads aren't linearizable. They're probably hitting a follower. Lock checks have to go through the consensus path."
