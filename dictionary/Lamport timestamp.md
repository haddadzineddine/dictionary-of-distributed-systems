---
description: A single counter per node that guarantees if A happened before B, then A's timestamp is smaller than B's.
---

A [logical clock](./Logical%20clock.md) made of a single integer counter on each [node](./Node.md), introduced by Leslie Lamport in 1978. It guarantees that if event A [happens before](./Happens-before.md) event B, then A's timestamp is smaller than B's.

The algorithm has three rules:

| Event           | What the node does                           |
| --------------- | -------------------------------------------- |
| Local event     | Adds 1 to its counter                        |
| Send message    | Adds 1 to its counter and attaches the value |
| Receive message | Sets its counter to max(local, received) + 1 |

For example, node A at counter 5 sends a message, which increments its counter to 6 and carries the value 6. Node B, at counter 2, receives it and moves to max(2, 6) + 1 = 7. B's receive event now sorts after A's send, even though B's counter was far behind.

The guarantee runs in one direction only. If A happens before B, then L(A) < L(B). The reverse does not hold: L(A) < L(B) does not show that A happened before B, because the two events may be concurrent. Misreading this is the usual mistake. Code that treats the higher timestamp as "the later write that saw the earlier one" will silently discard a concurrent update. To detect concurrency you need a [vector clock](./Vector%20clock.md).

Two events on different nodes can have the same counter value. Breaking ties by node ID, comparing (counter, node ID) pairs, gives a total order in which every pair of events is ranked and every node agrees on the ranking. That total order is consistent with causality, though its ordering of concurrent events is arbitrary. Lamport used this to implement mutual exclusion in his original paper, and [Paxos](./Paxos.md) implementations commonly build proposal numbers from the same counter-plus-ID pair.

_Usage:_

"This write has Lamport timestamp 42 and that one has 40, so the first one saw the second, right?"

"Not necessarily. A higher Lamport timestamp doesn't prove causality. They could be concurrent, and we'd need vector clocks to tell."
