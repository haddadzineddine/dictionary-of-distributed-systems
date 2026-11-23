---
description: Lamport's partial order on events. Event A happens before B if information from A could have reached B.
---

A relation on events, defined by Leslie Lamport in 1978, that captures when one event could have influenced another. It orders events by the flow of information and makes no reference to clock time, which makes it usable in a [distributed system](./Distributed%20system.md) where clocks disagree.

Event A happens before event B, written A → B, if any of these rules applies:

| Rule         | Condition                                                   |
| ------------ | ----------------------------------------------------------- |
| Local order  | A and B are on the same [node](./Node.md), and A came first |
| Message      | A is the sending of a message, and B is its receipt         |
| Transitivity | A → C and C → B for some event C                            |

If neither A → B nor B → A holds, the two events are concurrent. Concurrent does not mean simultaneous. It means no chain of messages connects them, so neither could have affected the other, even if one occurred an hour earlier by the wall clock.

This makes happens-before a partial order: some pairs of events have a defined order and others do not.

The relation explains a common display bug. In a chat app, a reply appears before the question it answers, because the two messages reached a replica in the wrong order or were sorted by timestamps from machines with [clock skew](./Clock%20skew.md). The reply causally depends on the question, so any correct ordering must place the question first. [Causal consistency](./Causal%20consistency.md) is the guarantee that preserves this order.

Note that A → B means A could have influenced B, not that it did. The relation tracks potential causality. A [logical clock](./Logical%20clock.md) makes it computable: a [Lamport timestamp](./Lamport%20timestamp.md) respects the order, and a [vector clock](./Vector%20clock.md) can also tell whether two events are concurrent.

_Usage:_

"These two writes are a millisecond apart. Which one came first?"

"Neither, as far as we can tell. No message links them, so there's no happens-before relation. They're concurrent and we need a conflict rule."
