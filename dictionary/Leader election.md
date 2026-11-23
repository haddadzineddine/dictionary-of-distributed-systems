---
description: The process by which a group of nodes chooses one to coordinate work, and replaces it when it fails.
---

The process by which a group of [nodes](./Node.md) chooses one member to coordinate work, and chooses a replacement when that member fails. A single leader can order writes, assign tasks or run a scheduled job without negotiating each decision.

An election protocol has two goals. Safety means at most one leader is recognized for a given term (a numbered period of leadership). Liveness means that a leader is eventually chosen as long as enough nodes are working. [FLP impossibility](./FLP%20impossibility.md) means liveness cannot be guaranteed on a fully asynchronous network, so protocols depend on timeouts.

| Approach                         | How a leader is chosen                                                        | Safe under partition?            |
| -------------------------------- | ----------------------------------------------------------------------------- | -------------------------------- |
| [Consensus](./Consensus.md) vote | A majority votes in a numbered term, as in [Raft](./Raft.md)                  | Yes, one leader per term         |
| Coordination service             | Nodes compete for a [lease](./Lease.md) in a system such as etcd or ZooKeeper | Yes, if stale leaders are fenced |
| Bully algorithm                  | The highest-ID reachable node wins                                            | No, each side can elect its own  |
| Manual or static                 | An operator or config file names the leader                                   | Depends on the operator          |

A correct election does not ensure that only one node believes it is leader at every moment. A leader that pauses or loses contact can continue acting on an old belief while a new leader is elected. The guarantee is per term, and that gap is how [split brain](./Split%20brain.md) reaches external systems. When a leader writes to shared storage, protect the write with a [fencing token](./Fencing%20token.md), such as the term number.

Elections also cost availability. Between the old leader failing and a new one taking over, writes fail or hang, typically for the detection timeout plus the election itself. Timeouts set too aggressively cause repeated elections under load, when a [failure detector](./Failure%20detector.md) mistakes slowness for failure.

_Usage:_

"The cron job ran twice last night, once on each scheduler."

"The old scheduler was paused during the handover and woke up still thinking it led. The job needs a fencing check."
