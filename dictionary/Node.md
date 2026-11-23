---
description: One participant in a distributed system, with its own state, that communicates with the others by sending messages.
---

One participant in a [distributed system](./Distributed%20system.md). A node holds its own state and communicates with other participants by sending messages. Papers on distributed algorithms often call it a process.

What counts as a node depends on the layer being discussed:

| Context                               | What "node" usually means                          |
| ------------------------------------- | -------------------------------------------------- |
| [Consensus](./Consensus.md) algorithm | One process that votes and keeps a log             |
| Database cluster                      | One server process holding some or all of the data |
| Container orchestrator                | One machine that runs containers                   |
| Network diagram                       | Any addressable device on the network              |

The difference matters when counting how many failures a system can survive. Three database nodes running as containers on one physical host are three nodes to the database and one failure domain (a set of components that fail together). If the host loses power, all three stop. Replication settings that look safe on paper give no protection in that layout, so placement across hosts, racks or zones has to be checked separately from the node count.

A node's identity also matters. Peers remember what a node has promised, so a node that loses its disk and rejoins under the same identity can break a protocol's assumptions. A [Raft](./Raft.md) node that forgets its vote could vote twice in one election, which could produce two leaders. For this reason, a node that has lost its durable state should normally rejoin as a new member through the cluster's membership process.

Nodes are usually described as correct or faulty. A correct node follows the protocol. A faulty node has stopped, as in a [crash failure](./Crash%20failure.md), or misbehaves, as in a [Byzantine failure](./Byzantine%20failure.md).

_Usage:_

"We run three nodes, so losing one is fine."

"Check the placement first. If two of them share a host, losing that host takes out a [quorum](./Quorum.md) and writes stop."
