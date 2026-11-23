---
description: A lock shared across machines so only one holder acts at a time. It cannot guarantee that alone when holders can pause.
---

A lock that coordinates processes on different machines, so that only one of them works on a resource at a time. Because a holder can crash without releasing it, a distributed lock is almost always implemented as a [lease](./Lease.md) that expires. That expiry is the source of its main weakness: a holder can lose the lock without knowing.

The typical incident looks like this. Worker A acquires the lock and stalls, perhaps in a garbage collection pause (the runtime halting to reclaim memory). Its lease expires. Worker B acquires the lock and starts writing. Worker A resumes, still believing it holds the lock, and also writes. The same job has run twice or a file is corrupted, even though the locking code contains no bug.

The right lock depends on its purpose:

| Purpose     | If two holders overlap             | Suitable approach                                                                                            |
| ----------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Efficiency  | Some work is duplicated            | A simple lock, such as Redis `SET key value NX PX ttl`                                                       |
| Correctness | Data is corrupted or money is lost | A [consensus](./Consensus.md)-backed lock in etcd or ZooKeeper, plus a [fencing token](./Fencing%20token.md) |

A lock on a single [node](./Node.md) is unavailable or lost if that node fails, and failover with asynchronous replication can grant the same lock twice. A consensus-backed service avoids that but cannot stop a paused client. Only the protected resource can, by rejecting stale requests.

Give each acquisition a unique value and release only if the stored value still matches, or a slow worker can delete a lock now held by someone else. Where possible, avoid the lock altogether by using [idempotency](./Idempotency.md) or a compare-and-set write (a write that succeeds only if the value is unchanged), which puts the check at the resource.

_Usage:_

"We put a Redis lock around the invoice job and still got duplicates."

"A lock only narrows that window. If the worker pauses past the expiry, two can run. Make the invoice insert idempotent."
