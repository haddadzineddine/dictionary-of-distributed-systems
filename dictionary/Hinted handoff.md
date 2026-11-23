---
description: Another node temporarily stores a write for an unreachable replica and delivers it when that replica returns.
---

A repair technique in which a reachable [node](./Node.md) temporarily stores a write meant for an unreachable replica (a node assigned to hold a copy of the data), along with a hint naming the intended owner. When the owner comes back, the stored write is delivered to it. Leaderless stores such as Dynamo and Cassandra use this technique, and it is what moves data back after a [sloppy quorum](./Sloppy%20quorum.md).

| Step | Who         | What happens                                                                                                         |
| ---- | ----------- | -------------------------------------------------------------------------------------------------------------------- |
| 1    | Coordinator | Sends a write and finds replica C unreachable                                                                        |
| 2    | Hint holder | Stores the write with a hint saying it belongs to C                                                                  |
| 3    | Hint holder | Learns C is back, through a [failure detector](./Failure%20detector.md) or [gossip protocol](./Gossip%20protocol.md) |
| 4    | Hint holder | Replays the write to C                                                                                               |
| 5    | Hint holder | Deletes the hint once C acknowledges                                                                                 |

The hint holder varies by product. In the Dynamo design it is a fallback node, and in Cassandra it is normally the coordinator (the node handling the client's request).

Hinted handoff shortens the time a returning replica is out of date, but it is a best-effort mechanism. A hint holder can fail before delivering, which loses the hint. Stores also stop collecting hints after a configurable window so that a long outage does not fill their disks. The recognizable failure is a node that returns after extended maintenance, reports healthy, and serves stale or missing rows, because the outage outlasted the hint window. Any outage longer than that window needs a full [anti-entropy](./Anti-entropy.md) repair before the node can be trusted.

Replay also creates load. When a node returns, several peers may send it their stored hints at once, on top of normal traffic, which can slow it enough to look failed again.

_Usage:_

"Node six was offline for two days. It's back, so we're fine, right?"

"Hinted handoff only covers the first few hours of that. Run a full repair before it serves [quorum](./Quorum.md) reads."
