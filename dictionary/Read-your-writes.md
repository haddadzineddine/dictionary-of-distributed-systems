---
description: A session guarantee that once a client writes a value, its later reads see that write or something newer.
---

A session guarantee that once a client has written a value, that client's later reads will return that write or a newer one. It was described by Douglas Terry and colleagues in 1994 as one of four session guarantees for weakly consistent systems. It is also called read-after-write consistency.

The symptom is familiar. A user edits their profile, presses save, sees the page reload, and the old value is back. The write went to the leader, the reload read from a replica (a [node](./Node.md) holding a copy of the data) that had not received it yet, and a few seconds later the new value appears.

The guarantee applies per session. Other users may still see the old value for a while, which is acceptable under [eventual consistency](./Eventual%20consistency.md). It needs less coordination than [linearizability](./Linearizability.md), since only the writer's own view needs protecting.

Common ways to provide it:

| Approach         | How it works                                                                           | Trade-off                                  |
| ---------------- | -------------------------------------------------------------------------------------- | ------------------------------------------ |
| Read from leader | Send reads of a user's own data to the leader                                          | Adds load to the leader                    |
| Time window      | Route a user to the leader for N seconds after a write                                 | Fails if lag exceeds N                     |
| Version token    | Client keeps its write's log position, and replicas serve the read only once caught up | Requires passing a token through the stack |
| Sticky routing   | Keep the session on the replica that took its writes                                   | Breaks when that replica fails             |

For a user on several devices, store the version token server-side, because a token in one browser's cookie does not cover a write made on a phone.

Seeing another user's update and then losing it on the next read is a separate anomaly, covered by [monotonic reads](./Monotonic%20reads.md).

_Usage:_

"Users keep reporting that their display name reverts after saving."

"We're breaking read-your-writes. The profile page reads from a replica. Pin that user to the primary for a short window after any update."
