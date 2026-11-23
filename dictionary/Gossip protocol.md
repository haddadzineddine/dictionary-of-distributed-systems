---
description: Nodes periodically exchange state with random peers, so information spreads through a cluster like an epidemic.
---

A communication method in which each [node](./Node.md) periodically exchanges information with a few randomly chosen peers. Those peers pass it on in later rounds, so an update spreads through the cluster the way an infection spreads through a population. These are also called epidemic protocols.

Each round, a node picks one or a few peers and does one of the following:

| Style     | What is exchanged                               |
| --------- | ----------------------------------------------- |
| Push      | The node sends the peer what it knows           |
| Pull      | The node asks the peer for what it is missing   |
| Push-pull | Both sides send each other what the other lacks |

Because the number of informed nodes grows roughly exponentially at first, an update reaches all N nodes in about O(log N) rounds with high probability. Each node sends a fixed amount per round, and there is no central coordinator. Random peer selection also routes around failed nodes and many [network partitions](./Network%20partition.md) that are not complete.

Gossip commonly carries cluster membership, node health and metadata such as which node owns which data. Cassandra and Consul are examples. It often feeds a [failure detector](./Failure%20detector.md), since peers can report on nodes they have heard from recently.

The trade-off is that delivery is probabilistic and delayed, giving [eventual consistency](./Eventual%20consistency.md) for the gossiped state. During a rollout, different nodes can briefly hold different views of the cluster, so one routes a request to a member that another already considers gone.

Removals need particular care. A node that was removed can reappear because a peer still holds its old entry and gossips it back. Protocols prevent this by attaching a generation or version number to each entry, and by keeping a removal marker long enough for it to spread everywhere.

_Usage:_

"We decommissioned that node an hour ago, but it keeps showing up in the member list."

"A peer's still gossiping its old state. Check that the removal marker has propagated before restarting anything."
