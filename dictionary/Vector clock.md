---
description: A list of counters, one per node, that shows whether two events are causally ordered or concurrent.
---

A [logical clock](./Logical%20clock.md) made of a list of counters, one per [node](./Node.md). Each node's vector records how many events it knows about from every node, including itself. Colin Fidge and Friedemann Mattern independently published the technique in 1988.

The update rules are:

1. Before each local event or send, a node increments its own entry.
2. Each message carries the sender's full vector.
3. On receipt, a node takes the element-wise maximum of its vector and the message's vector, then increments its own entry.

Two vectors are compared element by element. V(A) is less than V(B) if every entry of A is less than or equal to the matching entry of B, and at least one is strictly less.

| A         | B         | Result              |
| --------- | --------- | ------------------- |
| [2, 1, 0] | [2, 2, 0] | A happened before B |
| [3, 2, 1] | [2, 2, 0] | B happened before A |
| [3, 0, 0] | [2, 1, 0] | Concurrent          |
| [2, 1, 0] | [2, 1, 0] | Same event          |

Vector clocks characterize [happens-before](./Happens-before.md) exactly: A happened before B if and only if V(A) < V(B). A [Lamport timestamp](./Lamport%20timestamp.md) offers only the forward direction. This is what lets a store detect a true conflict. If two clients both edit a shopping cart based on the same version, their vectors come out concurrent, so the store can keep both versions for the application to merge, where a timestamp rule would drop one. A close relative, the version vector, tracks versions of one data item across its copies.

The cost is size. Each timestamp grows with the number of participants. Pruning old entries saves space but can misclassify concurrent updates as ordered, or the reverse.

_Usage:_

"Why did the cart come back with two versions of the same item list?"

"The vector clocks were concurrent, so neither write had seen the other. The store kept both, and our merge code has to reconcile them."
