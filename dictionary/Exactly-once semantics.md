---
description: Each message's effect is applied once, usually by combining retries with deduplication or an atomic commit.
---

A processing guarantee in which the effect of each message is applied exactly once, even if the message is transmitted several times. Over an unreliable network, no protocol can ensure that a message is physically delivered exactly once, because a sender that gets no acknowledgment cannot tell whether the message or the reply was lost. What systems can provide is an outcome equivalent to one delivery.

The standard construction is [at-least-once delivery](./At-least-once%20delivery.md) plus a mechanism that makes repeats harmless. Retries ensure nothing is lost, and the receiver ensures duplicates have no further effect:

| Technique                       | How it works                                                              | Boundary of the guarantee        |
| ------------------------------- | ------------------------------------------------------------------------- | -------------------------------- |
| [Idempotency](./Idempotency.md) | The receiver records message IDs and ignores repeats                      | Wherever the IDs are stored      |
| Atomic position and result      | Result and consumer position commit in one database transaction           | That one database                |
| Broker transactions             | Kafka's transactions commit output messages and consumer offsets together | Topics within that Kafka cluster |

Every technique has a boundary, and bugs often come from crossing it. A stream processor may advertise exactly-once, and within its own state and topics that claim holds. If a handler also sends an email or calls a payment API, that side effect sits outside the transaction. When the handler crashes and is re-run, the email goes out again.

For an external call, pass an idempotency key that the third party honors. If it offers none, you must choose between possibly repeating the action and possibly missing it, as with any [partial failure](./Partial%20failure.md).

Deduplication has limits. IDs are kept for a retention window, so a message redelivered after it is processed again. The ID check and the effect must also commit atomically, or a crash between them produces a duplicate or a loss.

_Usage:_

"Kafka is set to exactly-once, so why did the webhook fire twice?"

"That guarantee stops at Kafka. The webhook is an external side effect, so it needs its own idempotency key."
