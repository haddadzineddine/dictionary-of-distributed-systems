---
description: A component that reports which nodes it suspects have failed. Over an unreliable network, its answers can be wrong.
---

A component that reports which [nodes](./Node.md) it suspects have failed, usually based on missing [heartbeats](./Heartbeat.md) or timed-out requests. In an asynchronous network, where messages have no upper bound on delay, a slow node and a dead node look identical, so no detector can be right every time.

Chandra and Toueg formalized failure detectors in the 1990s using two kinds of property:

| Property     | Question it answers                         | Violation looks like                  |
| ------------ | ------------------------------------------- | ------------------------------------- |
| Completeness | Is every crashed node eventually suspected? | A dead node stays in rotation         |
| Accuracy     | Are working nodes left unsuspected?         | A healthy node is removed or replaced |

Real detectors have to trade one against the other. A short timeout finds crashes quickly but raises false positives, and a long timeout does the reverse. A false positive is costly because it can trigger a failover, data rebalancing, or a leader change on a cluster that was working. A pause on one node can then turn into cluster-wide churn.

Some detectors output a level of suspicion in place of a yes-or-no answer. The phi accrual detector, used by Cassandra and Akka, compares the time since the last heartbeat with the recent distribution of arrival times. Each application then picks its own threshold for acting.

Failure detectors also matter in theory. [FLP impossibility](./FLP%20impossibility.md) rules out deterministic [consensus](./Consensus.md) in a purely asynchronous system with even one crash. Chandra and Toueg showed that consensus becomes solvable with a majority of correct nodes and a detector that may make mistakes, provided it eventually suspects every crashed node and eventually stops suspecting at least one correct node.

Treat a detector's output as a hint. Actions that are unsafe if the node is still alive need protection such as a [fencing token](./Fencing%20token.md).

_Usage:_

"The detector marked node four dead, but it's serving traffic."

"It's a suspicion, not a fact. Node four probably stalled for a few seconds. Make sure the failover fences it off."
