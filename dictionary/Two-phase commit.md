---
description: A protocol that makes a transaction commit on all participants or none, but can block if the coordinator fails.
---

A protocol that makes a transaction across several [nodes](./Node.md) either commit on all of them or abort on all of them. One node acts as coordinator and the others are participants. It is often shortened to 2PC, and differs from two-phase locking, a technique for isolating concurrent transactions.

| Step | Who         | What happens                                                                       |
| ---- | ----------- | ---------------------------------------------------------------------------------- |
| 1    | Coordinator | Sends prepare to all participants                                                  |
| 2    | Participant | Durably records its changes and vote, keeps its locks, and replies yes or no       |
| 3    | Coordinator | Decides commit if all voted yes, otherwise abort, and records the decision durably |
| 4    | Coordinator | Sends the decision to all participants                                             |
| 5    | Participant | Applies the decision, releases locks and acknowledges                              |

A yes vote is binding. The participant must be able to commit even after a crash and restart, and it may not decide alone.

That promise creates the protocol's known weakness. If the coordinator fails after participants vote yes but before they hear the decision, they are in doubt. They cannot commit, because another participant may have voted no. They cannot abort, because the coordinator may have decided to commit. So they wait, holding locks, and other transactions touching those rows queue behind them. The symptom is a table that appears frozen with no active query to blame, until the coordinator recovers or an operator intervenes.

2PC is not a [consensus](./Consensus.md) protocol in the fault-tolerant sense. It requires every participant to respond, so one failure can stop progress, where consensus needs only a majority. A common remedy combines the two: each participant and the coordinator are replicated with [Paxos](./Paxos.md) or [Raft](./Raft.md), so that a single machine failure cannot leave a transaction stuck. Spanner uses this design. [Three-phase commit](./Three-phase%20commit.md) tried a different remedy.

_Usage:_

"This table has been locked for an hour, but nothing's running."

"Check pg_prepared_xacts. It's likely an in-doubt two-phase commit whose coordinator died. Get the outcome from the coordinator's log before resolving it."
