---
description: Eight false assumptions about networks, such as "the network is reliable", that lead to fragile distributed code.
---

Eight false assumptions that developers commonly make when writing software for a [distributed system](./Distributed%20system.md). The list is usually attributed to L Peter Deutsch and colleagues at Sun Microsystems in the 1990s.

| Fallacy                    | What goes wrong when code assumes it                                            |
| -------------------------- | ------------------------------------------------------------------------------- |
| The network is reliable    | Calls with no timeout or retry hang or silently lose work                       |
| Latency is zero            | A loop making one remote call per item takes seconds instead of milliseconds    |
| Bandwidth is infinite      | Large payloads saturate a link and delay unrelated traffic                      |
| The network is secure      | Unauthenticated internal traffic is open to anyone who gets inside              |
| Topology doesn't change    | Hard-coded addresses break when machines move or are replaced                   |
| There is one administrator | Another team's firewall or DNS change breaks your service                       |
| Transport cost is zero     | Serialization CPU and data transfer charges grow with traffic                   |
| The network is homogeneous | Mixed hardware, protocol versions or packet size limits fail on some paths only |

The fallacies persist because a development machine makes them approximately true. On localhost, calls do not get lost, latency is negligible and nobody else changes the configuration. Code written there can pass its tests and then fail under production conditions, often as a [partial failure](./Partial%20failure.md) that is hard to reproduce locally.

Libraries for remote procedure calls (calling a function on another machine) add to the problem by making a network call look like a local one.

To counter the assumptions, treat each network call as an operation that can fail. Give it a timeout, decide whether it can be retried safely, batch work to reduce round trips, and authenticate traffic between services. Test with injected latency and packet loss, so the failure paths run before production exercises them.

_Usage:_

"The export worked in staging, but in production it takes twenty minutes."

"It makes a separate call to the pricing service per row. That's the 'latency is zero' fallacy. Staging runs both services on one box."
