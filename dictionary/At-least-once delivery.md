---
description: A guarantee that a message is retried until acknowledged. It will not be lost, but it may be processed more than once.
---

A delivery guarantee in which a message is retried until the receiver acknowledges it. As long as the sender keeps running or has stored the message durably, and the receiver eventually recovers, the message is not lost. It may be processed more than once, which is the trade-off against [at-most-once delivery](./At-most-once%20delivery.md).

Duplicates are an inherent part of this guarantee, because of [partial failure](./Partial%20failure.md). When an acknowledgment is lost, the sender cannot distinguish that case from the message itself being lost, so it sends again:

| Step | Who      | What happens                                                            |
| ---- | -------- | ----------------------------------------------------------------------- |
| 1    | Sender   | Stores the message and sends it                                         |
| 2    | Receiver | Processes the message                                                   |
| 3    | Receiver | Sends an acknowledgment, which is lost or never sent because of a crash |
| 4    | Sender   | Times out with no acknowledgment                                        |
| 5    | Sender   | Resends the message                                                     |
| 6    | Receiver | Processes the same message a second time                                |

The result is a recognizable class of bug: a customer gets two confirmation emails, an account is charged twice, or a counter runs higher than the true value. These often appear in clusters after a deployment or a broker failover, when many in-flight messages are redelivered together.

The fix belongs in the receiver. Make processing safe to repeat through [idempotency](./Idempotency.md), typically by recording each message ID in the same database transaction as the work. This combination is how systems usually achieve [exactly-once semantics](./Exactly-once%20semantics.md).

Retries introduce two other problems. A malformed message that always fails will be redelivered forever and can block a queue, so after a limited number of attempts it should move to a dead-letter queue (a holding area for messages needing inspection). Retries can also reorder messages, so consumers that depend on order must check a version or sequence number.

_Usage:_

"A few customers got the shipping email twice after last night's broker restart."

"The queue is at-least-once, so redelivery is expected. The email worker needs to dedupe on message ID."
