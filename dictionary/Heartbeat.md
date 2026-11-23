---
description: A small message a node sends at regular intervals so that other nodes can tell it is still reachable.
---

A small message that a [node](./Node.md) sends at regular intervals so that other nodes can tell it is still reachable. If the expected heartbeats stop arriving for longer than a timeout, the receiver begins to suspect the sender has failed. In [Raft](./Raft.md), for example, a leader sends periodic messages to its followers, and a follower that hears nothing for its election timeout starts a new election.

A missing heartbeat does not prove a crash. The sender may also be paused, overloaded, or cut off by a [network partition](./Network%20partition.md). A heartbeat system therefore produces suspicion, which a [failure detector](./Failure%20detector.md) turns into a decision.

Two settings control the behavior, and each one trades detection speed against false alarms:

| Setting  | Lower value                                | Higher value                          |
| -------- | ------------------------------------------ | ------------------------------------- |
| Interval | Faster detection, more network and CPU use | Less overhead, slower detection       |
| Timeout  | Faster failover, more false positives      | Fewer false positives, longer outages |

A timeout set too low produces a recognizable pattern: leaders change repeatedly during load spikes although no machine has failed. A long garbage collection pause (the runtime halting a program to reclaim memory) or a saturated disk delays a heartbeat past the timeout, the cluster fails over, and the failover adds more load. The timeout should sit well above the normal variation in heartbeat delay, and the interval is usually a fraction of the timeout so that one lost message does not trigger suspicion.

A heartbeat shows only that the code sending it is running. If it runs on a dedicated thread, it can continue while the threads doing real work are stuck, which is one source of [gray failure](./Gray%20failure.md). Tying the heartbeat to real progress, such as a worker completing its loop, narrows that gap.

_Usage:_

"We had three [leader elections](./Leader%20election.md) during the batch job, but no node went down."

"The heartbeats are probably missing the timeout during GC pauses. Check the pause times before we raise it."
