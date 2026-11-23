---
description: A counter-based mechanism for ordering events by causality instead of by physical time.
---

A mechanism that assigns numbers to events so they can be ordered by causality instead of physical time. It counts events and messages and does not measure seconds. A [distributed system](./Distributed%20system.md) uses one because physical clocks on different [nodes](./Node.md) disagree through [clock skew](./Clock%20skew.md), so their readings cannot reliably show which of two nearby events came first.

The order a logical clock tracks is the [happens-before](./Happens-before.md) relation. If event A could have influenced event B, the clock gives A a value that sorts before B's. Nodes achieve this by attaching their clock value to each message and updating their own clock when a message arrives, so knowledge of order travels with the data.

The main variants differ in what they can tell you and what they cost:

| Variant                                               | Stores                       | Detects concurrency? | Close to wall-clock time? |
| ----------------------------------------------------- | ---------------------------- | -------------------- | ------------------------- |
| [Lamport timestamp](./Lamport%20timestamp.md)         | One counter                  | No                   | No                        |
| [Vector clock](./Vector%20clock.md)                   | One counter per node         | Yes                  | No                        |
| [Hybrid logical clock](./Hybrid%20logical%20clock.md) | Physical time plus a counter | No                   | Yes                       |

Choose by the question you need answered. If every node must rank events the same way, for example to break ties between updates, a Lamport timestamp with a node ID tie-breaker is enough. If you need to know that two updates conflicted, because neither saw the other, you need a vector clock. If you want timestamps that respect causality and can also be queried by approximate real time, as with snapshot reads in a database, a hybrid logical clock fits.

A logical clock cannot see causal links that exist outside the system. If one user makes a change and phones a colleague, who then makes a second change, the system sees no message linking the two and may treat them as concurrent.

_Usage:_

"Can we sort these events by created_at across services?"

"Not safely, the hosts' clocks differ. Have each service carry a logical clock in its messages and sort on that."
