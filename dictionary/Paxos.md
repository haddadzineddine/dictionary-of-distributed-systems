---
description: Lamport's consensus protocol, in which proposers and a majority of acceptors choose one value in two phases.
---

A [consensus](./Consensus.md) protocol devised by Leslie Lamport and published in 1998. It lets a group of [nodes](./Node.md) choose one value while tolerating [crash failures](./Crash%20failure.md) of a minority, message loss and message delay.

Paxos defines three roles, and one server often plays all of them:

- Proposers suggest values.
- Acceptors vote on proposals.
- Learners find out which value was chosen.

Single-decree Paxos decides one value in two phases:

| Phase       | From → to            | What happens                                                                                                                                            |
| ----------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1a Prepare  | Proposer → acceptors | Proposer picks a unique number `n` and sends `prepare(n)`                                                                                               |
| 1b Promise  | Acceptor → proposer  | If `n` is above any prepare it has answered, the acceptor promises to ignore lower numbers and returns the highest-numbered proposal it has accepted    |
| 2a Accept   | Proposer → acceptors | With promises from a majority, proposer sends `accept(n, v)`. `v` must be the value of the highest-numbered proposal returned, or its own value if none |
| 2b Accepted | Acceptor → learners  | Acceptor accepts unless it has promised a number above `n`                                                                                              |

A value is chosen once a majority of acceptors have accepted the same proposal. The rule in step 2a provides safety: any later proposer must learn of a possibly chosen value through the overlapping [quorum](./Quorum.md) and propose it again. Acceptors must persist promises and acceptances to disk before replying, or a restart could break those promises.

Two proposers can block each other indefinitely by issuing ever-higher numbers, each invalidating the other's phase 1. A distinguished leader or randomized backoff makes this livelock unlikely.

Real systems need a sequence of decisions for [state machine replication](./State%20machine%20replication.md). Multi-Paxos runs one instance per log slot, and a stable leader can skip phase 1 for later slots. [Raft](./Raft.md) was designed partly because the original papers leave these engineering details open.

_Usage:_

"Throughput dropped to nothing and both servers are busy sending prepares."

"Dueling proposers, preempting each other in phase 1. Add a stable leader or randomized backoff."
