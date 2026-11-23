---
description: A session guarantee that once a client has seen a value, later reads never return an older state.
---

A session guarantee that once a client has read a value, its later reads will return that value or a newer one. It is one of the four session guarantees from Douglas Terry and colleagues in 1994.

The symptom is data that appears and then vanishes. A user refreshes a page and sees a new comment, refreshes again and it has gone, and refreshes a third time and it is back. No data changed. A load balancer sent each request to a different replica (a [node](./Node.md) holding a copy of the data), and those replicas were at different points in replication:

| Refresh | Replica | Replica's lag | New comment visible? |
| ------- | ------- | ------------- | -------------------- |
| 1       | A       | 0 seconds     | Yes                  |
| 2       | B       | 5 seconds     | No                   |
| 3       | A       | 0 seconds     | Yes                  |

Under [eventual consistency](./Eventual%20consistency.md) this is permitted, but it looks like a bug to users, and it can break application logic. A worker that reads a job as finished and then reads it as pending may run it again.

There are two common ways to provide the guarantee. The first pins each session to one replica, for example by hashing the user ID to choose it. If that replica fails, the session can go backward. The second has the client remember the highest log position or version it has seen and send it with each read, so a replica that is further behind waits, or the request goes to another replica.

Monotonic reads does not promise fresh data. A client pinned to a replica ten minutes behind satisfies the guarantee as long as its view never moves backward. It also does not imply [read-your-writes](./Read-your-writes.md), because the client's own write may not have reached that replica. [Causal consistency](./Causal%20consistency.md) includes each of them.

_Usage:_

"The order shows as shipped, then processing, then shipped again when I refresh."

"That's a monotonic reads violation. The reads are bouncing between replicas with different lag. Let's route each session to one replica."
