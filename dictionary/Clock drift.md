---
description: The rate at which a clock runs fast or slow compared with a reference, causing its error to grow over time.
---

The rate at which a clock runs fast or slow compared with a reference clock. Computer clocks count oscillations of a quartz crystal, and no two crystals run at exactly the same frequency. Temperature, age and manufacturing variation all shift the rate.

Drift is usually measured in parts per million (ppm). A clock that is off by 50 ppm gains or loses 4.32 seconds per day.

Drift and [clock skew](./Clock%20skew.md) are related but distinct:

| Term  | Measures                             | Example     |
| ----- | ------------------------------------ | ----------- |
| Drift | Rate of error of one clock           | 20 ppm fast |
| Skew  | Gap between two clocks at one moment | 40 ms apart |

Left alone, drift turns into growing skew. [NTP](./NTP.md) counters this by measuring the clock against a time server and correcting it, and it also estimates the local frequency error so it can compensate between checks. The failure to watch for is a machine whose time synchronization has silently stopped. Nothing breaks at first. Days later, its timestamps are seconds away from its peers, and token validation or log ordering starts to go wrong on that host only.

Drift also affects timers, which matters for anything based on a duration. A [lease](./Lease.md) holder and its grantor each measure the lease period on their own clocks. If the holder's clock runs slow, it can believe the lease is still valid after the grantor considers it expired. Lease protocols handle this by assuming a maximum drift rate and building in a safety margin.

For measuring elapsed time, use a monotonic clock (a timer that only moves forward). It does not jump when the wall clock is corrected, but it still drifts, so it does not remove the need for that margin.

_Usage:_

"This one VM keeps rejecting valid auth tokens, and only that VM."

"Check its time sync daemon. If sync stopped weeks ago, clock drift has had plenty of time to push it outside the leeway window."
