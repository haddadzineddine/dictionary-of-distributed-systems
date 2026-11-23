---
description: An increasing number issued with each lock grant, letting a resource reject writes from a holder that has been replaced.
---

A number that increases with every grant of a [lease](./Lease.md) or lock, which the holder attaches to each request to the protected resource. The resource remembers the highest token it has seen and rejects anything lower. This stops a former holder that does not yet know it has been replaced.

Fencing addresses the gap that a [distributed lock](./Distributed%20lock.md) cannot close on its own, because a client can pause at any moment and resume after its lease has expired:

| Step | Actor    | Action                                  | Storage's highest token |
| ---- | -------- | --------------------------------------- | ----------------------- |
| 1    | Client 1 | Acquires lock, receives token 33        | 32                      |
| 2    | Client 1 | Pauses, and its lease expires           | 32                      |
| 3    | Client 2 | Acquires lock, receives token 34        | 32                      |
| 4    | Client 2 | Writes with token 34, accepted          | 34                      |
| 5    | Client 1 | Resumes, writes with token 33, rejected | 34                      |

Without step 5's check, Client 1 would overwrite Client 2's work, and neither client would see an error.

Two requirements make this work. The tokens must come from a source that never gives one number to two holders or goes backward, which in practice means a system built on [consensus](./Consensus.md). A [Raft](./Raft.md) term, an etcd revision and a ZooKeeper transaction ID (zxid) are common sources. The resource must also perform the comparison and the write atomically, because a separate check followed by a write reopens the race.

The second requirement is the practical limit. Fencing needs the resource to participate, and many do not. A database can do it with a conditional update such as `UPDATE ... WHERE fence_token <= 34`. A third-party API usually cannot, so for those calls use an [idempotency](./Idempotency.md) key (a unique ID that lets the receiver discard repeats) and accept that the protection is weaker.

_Usage:_

"The old leader came back after the pause and overwrote the new leader's output."

"Nothing was checking who was current. Pass the term as a fencing token and have storage reject anything lower than it's seen."
