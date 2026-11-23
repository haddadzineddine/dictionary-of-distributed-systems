---
description: Some parts of a distributed system fail while others keep working, leaving callers unsure what actually happened.
---

A condition in which some parts of a [distributed system](./Distributed%20system.md) fail while others keep working. A program on one machine usually fails as a unit. Across a network, one [node](./Node.md) can stop, slow down or lose contact while the rest continue, and the nodes still running may not know.

The practical difficulty is that a caller often cannot tell what happened. When a remote call times out, any of these may be true:

| What happened                      | Was the work done?     | What the caller sees |
| ---------------------------------- | ---------------------- | -------------------- |
| Request lost on the way            | No                     | Timeout              |
| Receiver crashed before processing | No                     | Timeout              |
| Receiver crashed after processing  | Yes                    | Timeout              |
| Response lost on the way back      | Yes                    | Timeout              |
| Receiver is slow and still working | Not yet, but it may be | Timeout              |

A timeout therefore means "outcome unknown", not "failed". Code that treats it as a failure produces a familiar bug: a payment call times out, the client retries, and the customer is charged twice because the first attempt had succeeded. Code that does not retry has the opposite problem, and the payment may never happen.

Handling partial failure means designing each remote operation for the unknown case. Set a timeout on every call, so a lost response cannot hang the caller indefinitely. Make retried operations safe through [idempotency](./Idempotency.md), usually with a client-generated request ID that the receiver records. For multi-step work, decide how an interrupted sequence is finished or undone, and run a periodic reconciliation (a job that compares records and corrects mismatches) where money or other important state is involved.

_Usage:_

"The charge endpoint returned a 504, so the order was marked failed. But the customer's bank shows the payment."

"That's partial failure. The gateway timed out after the charge went through. We should record the attempt with an idempotency key and query its status before retrying."
