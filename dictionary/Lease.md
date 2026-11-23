---
description: A time-limited grant of a right or resource, which expires automatically unless the holder renews it.
---

A time-limited grant of a right, such as being leader or holding a lock, that expires automatically unless the holder renews it. Cary Gray and David Cheriton described leases in 1989 for file cache consistency. If a [node](./Node.md) holding an ordinary lock crashes, the lock may never be released. A lease ends on its own.

| Step | Who     | What happens                                                      |
| ---- | ------- | ----------------------------------------------------------------- |
| 1    | Holder  | Records its local time, then requests a lease of duration D       |
| 2    | Grantor | Grants the lease and starts timing D                              |
| 3    | Holder  | Works, and renews well before D runs out                          |
| 4    | Holder  | Stops work if it cannot renew by its local expiry                 |
| 5    | Grantor | After D plus a safety margin, may grant the lease to another node |

The holder starts counting before sending its request, so its expiry falls earlier than the grantor's. This keeps the two views safe without synchronized wall clocks, provided [clock drift](./Clock%20drift.md) stays below an assumed bound. Durations should use a monotonic clock (a timer that only moves forward).

Lease duration is a trade-off. A short lease recovers faster after a crash but needs more renewal traffic and is easier to lose in a brief stall.

A paused holder can outlive its lease. A process checks its lease, sees it is valid, and is then paused by garbage collection (the runtime halting the program to reclaim memory) or a VM migration. The lease expires, another node acquires it, and the first process wakes and carries on with its write. No check inside the process can close this gap, since the pause can land right after the check. Protecting the resource takes a [fencing token](./Fencing%20token.md).

_Usage:_

"Worker A's lease expired during a long GC pause, and it still wrote to the bucket."

"A lease can't stop that by itself. The storage layer has to reject stale holders, so let's add a fencing token."
