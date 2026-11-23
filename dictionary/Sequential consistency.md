---
description: All clients see operations in one agreed order that preserves each client's own order, without respecting real time.
---

A consistency model in which all operations appear in one total order that every client agrees on, and each client's own operations appear in that order in the sequence the client issued them. Leslie Lamport defined it in 1979, originally for multiprocessor memory.

It is weaker than [linearizability](./Linearizability.md) in one specific way. Linearizability requires the agreed order to match real time, while sequential consistency does not. An operation may appear to take effect before it started or after it finished.

| Requirement                             | Linearizable | Sequentially consistent |
| --------------------------------------- | ------------ | ----------------------- |
| One total order agreed by all clients   | Yes          | Yes                     |
| Each client's own order preserved       | Yes          | Yes                     |
| Order respects real time across clients | Yes          | No                      |
| Composable across objects               | Yes          | No                      |

Here is a history that is sequentially consistent but not linearizable. Suppose `x` starts at 0. Client A writes `x = 1` and the write returns. Later, client B reads `x` and gets 0. Placing B's read before A's write yields a valid single order, so sequential consistency allows it. Linearizability forbids it, since A's write finished before B's read began. What B cannot do is read 1 and then read 0 afterwards, because that would contradict any single order.

This produces a recognizable surprise. One service finishes a write and tells another over a queue or an HTTP call, and the second service reads stale data even though the store is meeting its guarantee. The notification traveled through a channel the storage system cannot see.

ZooKeeper shows the model in practice: updates are applied in a single order, but a client reading from a follower can see data that is behind the leader. Clients that need a fresher view call `sync` before reading.

_Usage:_

"Service A committed the config, then called service B, and B read the old version."

"The store is only sequentially consistent, so B's read can be ordered before the write. Have B call sync first."
