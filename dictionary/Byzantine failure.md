---
description: A failure in which a node behaves arbitrarily, including sending false or conflicting messages to different peers.
---

A failure in which a [node](./Node.md) deviates from its protocol in any way, including sending false data or telling different peers different things. The name comes from the Byzantine Generals Problem, published by Lamport, Shostak and Pease in 1982, in which loyal generals must agree on a plan even though some of their number are traitors.

A Byzantine node need not be malicious. Faulty memory, a firmware bug, corrupted data or two software versions that interpret a message differently can all make a node produce wrong output while appearing to work. An attacker who controls a node is the other main case.

This model is harder to tolerate than a [crash failure](./Crash%20failure.md), because other nodes cannot take a message at face value:

| Failure model | Nodes needed to tolerate f faults with [consensus](./Consensus.md) in a partially synchronous network |
| ------------- | ----------------------------------------------------------------------------------------------------- |
| Crash         | 2f + 1                                                                                                |
| Byzantine     | 3f + 1                                                                                                |

Partially synchronous means the network may misbehave for a while but eventually delivers messages within some bound. Under those conditions, tolerating one Byzantine node takes four nodes, compared with three for one crash.

[Paxos](./Paxos.md) and [Raft](./Raft.md) do not tolerate Byzantine failures. A single Raft leader that lies can corrupt the replicated log. Byzantine fault-tolerant protocols, such as PBFT (Practical Byzantine Fault Tolerance), add extra message rounds and cryptographic verification. They are mainly used where participants do not trust each other, as in blockchains.

Systems run by one organization usually assume crash failures and handle the likeliest forms of corruption separately. Checksums on disk blocks and network messages turn silent corruption into a detectable error, which the system can then treat as a crash and repair from another copy.

_Usage:_

"One replica keeps returning a different hash for the same block. Could Raft have caught that?"

"No, Raft assumes nodes don't lie. That's a Byzantine failure, probably bad memory or disk. Add checksums so corruption shows up as an error."
