---
description: Fixing stale replicas during a read, by comparing their responses and writing the newest version back.
---

A technique that fixes out-of-date replicas (the [nodes](./Node.md) holding copies of a key) as a side effect of reading. When a read contacts several replicas and their versions disagree, the coordinator (the node handling the request) sends the newest version to the replicas that returned an older one.

| Step | Who         | What happens                                         |
| ---- | ----------- | ---------------------------------------------------- |
| 1    | Coordinator | Sends the read to R replicas                         |
| 2    | Replicas    | Return their value, or a digest (a short hash) of it |
| 3    | Coordinator | Compares versions and identifies the newest          |
| 4    | Coordinator | Writes the newest value to the stale replicas        |
| 5    | Coordinator | Returns the newest value to the client               |

Steps 4 and 5 can happen in either order, and the choice changes the guarantee. In blocking read repair, the coordinator waits until a write [quorum](./Quorum.md) holds the value before responding, so a later quorum read cannot return the older version. In asynchronous read repair, the client gets a faster response, but a following read can still see the old value.

Read repair produces a recognizable pattern in debugging. A read from one replica returns an old value, the next attempt returns the correct one, and by the time someone investigates, all replicas agree. The first read triggered the fix and removed the evidence.

The technique only repairs keys that are read. Rarely read data can stay inconsistent indefinitely, so it is paired with background [anti-entropy](./Anti-entropy.md).

Read repair is only as correct as its version comparison. With [last-write-wins](./Last-write-wins.md), a write carrying a timestamp inflated by [clock skew](./Clock%20skew.md) is treated as newest and copied over a genuinely newer value. Stores that detect concurrent versions, for example with a [vector clock](./Vector%20clock.md), must merge them or keep both.

_Usage:_

"The stale row fixed itself before I could reproduce it."

"Probably read repair. Your first query found the mismatch and updated the lagging replica. Check the repair metrics for that table."
