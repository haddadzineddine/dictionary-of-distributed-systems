---
description: Network Time Protocol. Synchronizes a machine's clock with time servers by estimating offset and network delay.
---

Network Time Protocol, the standard way for machines to synchronize their clocks over a network. A client exchanges timestamped packets with one or more time servers, estimates how far its clock is off, and corrects it. It runs over UDP on port 123, and common implementations include ntpd and chrony.

Servers are arranged in levels called strata. Stratum 0 is a reference source such as an atomic clock or GPS receiver. A stratum 1 server is attached directly to one, a stratum 2 server synchronizes from stratum 1, and so on.

Each measurement uses four timestamps:

| Timestamp | Taken by | Moment            |
| --------- | -------- | ----------------- |
| t1        | Client   | Request sent      |
| t2        | Server   | Request received  |
| t3        | Server   | Response sent     |
| t4        | Client   | Response received |

From these, the client computes `offset = ((t2 - t1) + (t3 - t4)) / 2` and `delay = (t4 - t1) - (t3 - t2)`. The offset formula assumes the request and response took equal time. When the paths are asymmetric, the result is wrong by half the difference between the two one-way delays, and the client cannot detect this from the timestamps alone. NTP therefore narrows [clock skew](./Clock%20skew.md) but cannot eliminate it.

The client applies a correction in one of two ways. Slewing runs the clock slightly fast or slow until the error closes. Stepping sets the clock to the new value at once, and is normally reserved for large errors.

A step can move the wall clock backward, which produces confusing bugs: a measured duration comes out negative, or a timeout fires early or late. Code that measures elapsed time should use a monotonic clock (a timer that only moves forward). NTP also keeps correcting [clock drift](./Clock%20drift.md) between measurements, so a stopped daemon lets error build up with no visible failure.

_Usage:_

"Our request duration metric recorded negative values for a few seconds last night."

"NTP probably stepped the clock backward. That code should use a monotonic clock, not wall-clock time."
