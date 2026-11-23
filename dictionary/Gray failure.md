---
description: A degraded state where a component looks healthy to some observers while it is failing for others.
---

A degraded state in which a component appears healthy to some observers while it is failing for others. Researchers describe this as differential observability: the monitoring system and the application see different realities. It differs from a clean [crash failure](./Crash%20failure.md), where the component stops and every observer eventually notices.

The typical symptom is a dashboard that stays green during a user-visible outage. A server answers its health check on a lightweight endpoint, so the load balancer keeps sending it traffic, while real requests fail because they depend on a disk, downstream connection or thread pool that the check does not exercise.

Common forms include:

| Component    | Degradation                                    | Why checks may miss it                        |
| ------------ | ---------------------------------------------- | --------------------------------------------- |
| Disk         | Some reads take seconds                        | The check does not touch the slow path        |
| Network card | Drops a small share of packets                 | Small probes usually get through or retry     |
| Process      | Worker threads stuck, listener alive           | The health endpoint runs on a separate thread |
| Dependency   | One shard (a slice of the data) is unavailable | The check queries a different shard           |

Gray failures can also prevent automatic recovery. A [failure detector](./Failure%20detector.md) that relies on [heartbeats](./Heartbeat.md) keeps reporting the [node](./Node.md) as alive, so no failover happens. A slow node can be worse than a dead one, because requests queue behind it and hold resources in its callers, which can spread the slowdown to other services.

Detection works better from the perspective of the callers. Track error rate and latency percentiles per instance, compare each instance to its peers, and remove outliers from rotation. Make health checks exercise the same dependencies as real requests, with care that a shared dependency failing does not mark every instance unhealthy at once.

_Usage:_

"All instances pass health checks, but about one percent of checkouts hang."

"Break latency down by host. If one stands out, it's a gray failure, probably its disk. Drain it even though the health check says it's fine."
