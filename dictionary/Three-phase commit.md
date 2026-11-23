---
description: A variant of two-phase commit that avoids blocking under crash failures, but can break atomicity during a partition.
---

A variant of [two-phase commit](./Two-phase%20commit.md) that adds a phase to avoid blocking when the coordinator crashes. Dale Skeen proposed it in the early 1980s. It is often shortened to 3PC.

Two-phase commit blocks because a participant that voted yes cannot tell whether anyone has already committed. Three-phase commit removes that uncertainty by inserting a step between voting and committing:

| Phase        | Coordinator                                      | Participants                                |
| ------------ | ------------------------------------------------ | ------------------------------------------- |
| 1. CanCommit | Asks whether each can commit                     | Vote yes or no                              |
| 2. PreCommit | If all voted yes, announces the intent to commit | Record the pre-commit state and acknowledge |
| 3. DoCommit  | After all acknowledge, tells them to commit      | Commit and acknowledge                      |

The added phase creates a useful invariant: no participant commits until every participant has voted yes and has been told so. If the coordinator crashes, the surviving participants can elect a new one and inspect their states. If any of them reached pre-commit, every participant voted yes, so they can finish the commit. If none did, nobody can have committed, so they can safely abort.

That recovery requires [crash failures](./Crash%20failure.md) only, bounded message delays, and a reliable [failure detector](./Failure%20detector.md), so that a slow [node](./Node.md) is never mistaken for a dead one.

Under a [network partition](./Network%20partition.md), the protocol can break atomicity, its main guarantee. Suppose participants that received pre-commit end up on one side and the rest on the other. The first group commits, the second sees no pre-commit and aborts, and the transaction ends up committed on some nodes and aborted on others. Two-phase commit would have blocked here and preserved atomicity.

Production systems that need non-blocking commit usually run two-phase commit over groups replicated with [consensus](./Consensus.md), using [Paxos](./Paxos.md) or [Raft](./Raft.md).

_Usage:_

"Couldn't we switch to three-phase commit so a dead coordinator stops blocking us?"

"Only if the network never partitions. Otherwise each side can decide differently. Replicating the coordinator with Raft is safer."
