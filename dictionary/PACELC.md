---
description: Extends CAP. If partitioned, choose availability or consistency. Else, choose latency or consistency.
---

An extension of the [CAP theorem](./CAP%20theorem.md), proposed by Daniel Abadi, that also covers how a replicated system behaves when the network is healthy. The name spells out the rule: if there is a Partition, choose Availability or Consistency; Else, choose Latency or Consistency.

CAP says nothing about normal operation, yet that is where a system spends most of its time and where its replication design has a constant cost. To confirm a write on several replicas ([nodes](./Node.md) holding copies of the data) before responding, a system has to wait for network round trips. To respond faster, it must acknowledge before all replicas have the update, or read from a replica that may be behind.

Systems are classified with a two-part label:

| Label | During a partition    | Normal operation   | Typical design                                                          |
| ----- | --------------------- | ------------------ | ----------------------------------------------------------------------- |
| PA/EL | Stays available       | Favors low latency | Leaderless store with low read and write [quorum](./Quorum.md) settings |
| PC/EC | Refuses some requests | Favors consistency | Every operation goes through [consensus](./Consensus.md)                |
| PC/EL | Refuses some requests | Favors low latency | Single leader for writes, with stale reads from followers               |
| PA/EC | Stays available       | Favors consistency | Consistent reads when healthy, weaker reads during failures             |

The framework explains a symptom CAP cannot. A team picks a strongly consistent multi-region database, nothing is failing, and every write still takes as long as a round trip between regions. That is the EC side of the trade, paid on each request.

Labels describe a configuration more than a product. A store such as Cassandra can move toward the consistent end by raising its read and write quorum levels. Use the framework to ask two separate questions of each workload: what should happen during a partition, and how much latency is acceptable for fresher data the rest of the time.

_Usage:_

"Nothing's down, so why does every write take 90 milliseconds?"

"We replicate synchronously across regions. In PACELC terms we chose consistency over latency, and that's the round trip."
