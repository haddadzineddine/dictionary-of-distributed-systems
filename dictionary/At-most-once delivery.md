---
description: A guarantee that a message is processed zero or one times. It is never duplicated, but it can be lost.
---

A delivery guarantee in which each message is processed zero or one times. A message is never handled twice, but it can be lost. The sender makes one attempt, or the receiver records progress before doing the work, so that a failure leaves the message unprocessed and nothing tries again.

The guarantee is a consequence of the same ambiguity behind every [partial failure](./Partial%20failure.md). If a sender gets no acknowledgment, it cannot know whether the message arrived. It must choose between resending, which risks a duplicate, and not resending, which risks a loss. At-most-once chooses not to resend. [At-least-once delivery](./At-least-once%20delivery.md) makes the opposite choice.

In a message queue, the order of two consumer steps determines which guarantee you get:

| Step | At-most-once consumer | If it crashes right after this step        |
| ---- | --------------------- | ------------------------------------------ |
| 1    | Receives the message  | Message is redelivered, nothing lost       |
| 2    | Commits its position  | Message is lost, because it counts as done |
| 3    | Processes the message | Work is complete                           |

The symptom is a gap: a queue dashboard shows everything consumed and no errors, yet some records never reached the destination, often clustered around a deployment or crash. Auto-commit settings in consumer libraries can produce this behavior without anyone having chosen it, since the position may be committed on a timer while work is still in progress.

At-most-once suits data where a small loss costs less than a duplicate or a retry: high-volume metrics, debug logs, telemetry samples, or live position updates that the next message replaces anyway.

It does not suit payments, orders, or anything that must eventually happen. If a loss is unacceptable, use at-least-once delivery with [idempotency](./Idempotency.md), which together produce [exactly-once semantics](./Exactly-once%20semantics.md).

_Usage:_

"We lost about forty events during the last deploy, but the consumer shows no errors."

"It auto-commits offsets before processing finishes, so that's at-most-once delivery. Commit after the work is done."
