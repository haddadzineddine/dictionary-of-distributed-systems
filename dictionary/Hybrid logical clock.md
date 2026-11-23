---
description: A timestamp combining physical time with a counter, so it respects causality while staying close to wall-clock time.
---

A timestamp that combines a physical clock reading with a [logical clock](./Logical%20clock.md) counter. It preserves the [happens-before](./Happens-before.md) order and stays close to real time. Kulkarni and colleagues described the design in 2014, and databases such as CockroachDB use a variant of it.

Each timestamp has two parts: `l`, the largest physical time this [node](./Node.md) has seen, and `c`, a counter that orders events sharing the same `l`. Timestamps compare by `l` first, then `c`. On a local event or send, `l` advances to the physical clock if that is larger, and `c` resets to 0. Otherwise `c` increments. On a receive, `l` becomes the largest of the local `l`, the message's `l` and the physical clock, and `c` is set so the new timestamp sorts after both inputs, or to 0 if the physical clock alone is largest.

Plain physical timestamps can violate causality under [clock skew](./Clock%20skew.md), and a [Lamport timestamp](./Lamport%20timestamp.md) has no relation to real time, so it cannot answer "what was the state at 14:00?". A hybrid logical clock gives a single sortable value that does both.

Its guarantee has the same one-way shape as Lamport's. If A happened before B, then A's timestamp is smaller. A smaller timestamp does not prove causality, so it cannot detect concurrency the way a [vector clock](./Vector%20clock.md) can.

It still depends on reasonable clock synchronization. One node with a clock far ahead pushes `l` forward on every node it talks to, and those nodes carry that future time until physical time catches up. Systems therefore enforce a maximum tolerated offset. CockroachDB, for example, shuts down a node whose clock strays too far from its peers.

_Usage:_

"Why did this node crash with a clock offset error?"

"Its clock drifted past the configured maximum. Our hybrid logical clock relies on that bound, so the node stops itself before it can break ordering."
