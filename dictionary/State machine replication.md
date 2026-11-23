---
description: Keeping replicas identical by having each apply the same deterministic commands in the same order.
---

A technique for keeping several copies of a service identical by having each copy apply the same deterministic commands in the same order. If every replica (a [node](./Node.md) holding a copy) starts in the same state and processes the same sequence of inputs, every replica ends in the same state. Leslie Lamport introduced the approach in 1978, and Fred Schneider's 1990 tutorial is a standard reference.

The hard part is agreeing on the sequence, which is a [consensus](./Consensus.md) problem. Protocols such as [Raft](./Raft.md) and Multi-[Paxos](./Paxos.md) maintain a replicated log of commands. A typical leader-based write looks like this:

| Step | Who          | What happens                                        |
| ---- | ------------ | --------------------------------------------------- |
| 1    | Client       | Sends a command to the leader                       |
| 2    | Leader       | Appends it to its log and sends it to followers     |
| 3    | Followers    | Store the entry durably and acknowledge             |
| 4    | Leader       | Marks it committed once a majority has stored it    |
| 5    | Each replica | Applies committed entries to its state in log order |
| 6    | Leader       | Returns the result to the client                    |

With 2f + 1 replicas, this tolerates f [crash failures](./Crash%20failure.md).

Commands must be deterministic, and violations cause replicas to diverge silently. If a command calls `now()` or `random()`, or iterates over a hash map whose order varies, each replica computes a different result from the same log. The usual fix is for the leader to resolve such values first and put the concrete result into the log entry.

Two practical details follow. Logs grow without limit, so replicas periodically save a snapshot of their state and discard earlier entries. Clients also retry after timeouts, which can place one command in the log twice, so commands carry a client ID and sequence number that give [idempotency](./Idempotency.md) when applied.

_Usage:_

"Two replicas have different balances, but their logs are identical."

"Then something in the apply path isn't deterministic. State machine replication only works if the same log produces the same state. Check for timestamps."
