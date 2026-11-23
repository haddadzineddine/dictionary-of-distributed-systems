---
description: The difference between two clocks' readings at the same instant, which can make events appear out of order.
---

The difference between the readings of two clocks at the same instant. If one [node](./Node.md) reads 12:00:00.000 while another reads 12:00:00.250, the skew between them is 250 milliseconds. Every machine keeps its own clock, so some skew is always present. [Clock drift](./Clock%20drift.md) is the main cause: clocks run at slightly different rates, and the gap grows until something corrects it.

Skew causes bugs wherever code compares timestamps taken on different machines:

| Where               | What skew does                                                                    |
| ------------------- | --------------------------------------------------------------------------------- |
| Logs and traces     | An effect appears to happen before its cause                                      |
| Auth tokens         | A token looks issued in the future, or expires early                              |
| Caches              | Entries with an expiry time live too long or too short                            |
| Conflict resolution | With [last-write-wins](./Last-write-wins.md), an older write can beat a newer one |

The conflict resolution case loses data with no error. A user updates a record through a node whose clock is behind, and the update carries an earlier timestamp than the value it replaces. The store keeps the old value and reports success.

[NTP](./NTP.md) reduces skew but does not remove it, and its accuracy depends on network conditions. Code should therefore tolerate a margin: auth libraries commonly allow some leeway, for example. Monitor the offset of each host, because a broken time daemon produces no errors of its own.

Timestamps from different machines cannot establish which event came first when the gap between events is within the skew. To order events reliably, use the [happens-before](./Happens-before.md) relation through a [logical clock](./Logical%20clock.md), or a single authority that assigns sequence numbers.

Some network measurement documents use "skew" for a difference in clock rate. In writing about [distributed systems](./Distributed%20system.md), it normally means the difference in readings.

_Usage:_

"The trace says the database responded before the API sent the query."

"Those spans come from different hosts. It's clock skew, so don't read anything into the order at that scale."
