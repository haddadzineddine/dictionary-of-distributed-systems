---
description: The property that performing an operation several times has the same effect as performing it once.
---

The property that performing an operation several times has the same effect as performing it once. In a [distributed system](./Distributed%20system.md), it makes retries safe. A timed-out request may or may not have been applied, as in any [partial failure](./Partial%20failure.md), and the client can resend it without knowing which.

Whether an operation is idempotent depends on how it is expressed:

| Operation                                                | Idempotent? | Why                             |
| -------------------------------------------------------- | ----------- | ------------------------------- |
| `SET balance = 100`                                      | Yes         | Repeating leaves the same value |
| `balance = balance + 10`                                 | No          | Each repeat adds 10             |
| `DELETE WHERE id = 7`                                    | Yes         | The row stays gone              |
| `INSERT` with a generated ID                             | No          | Each repeat creates a row       |
| `INSERT` with a client-chosen ID and a unique constraint | Yes         | Repeats are rejected or ignored |

Other operations can use an idempotency key, a unique ID the client generates once per logical operation and reuses on every retry. The server stores the key with the result, and on a repeat it returns that result and skips the work. Without this, a double-clicked pay button can charge a customer twice.

The key and the effect must commit in one transaction, or a crash between them breaks the guarantee. Concurrent retries can both pass a "have I seen this?" check, so enforce uniqueness with a database constraint. Keys are eventually deleted, so the retention period must exceed the longest time a client might retry.

Idempotent does not mean safe to replay at any time. If `SET x = 1` is delayed and retried after another client has run `SET x = 2`, the retry undoes the newer write. Guarding against that takes a version check or a [fencing token](./Fencing%20token.md).

Idempotency is the receiver's half of [exactly-once semantics](./Exactly-once%20semantics.md), paired with [at-least-once delivery](./At-least-once%20delivery.md).

_Usage:_

"The mobile client retries on timeout, and we're seeing duplicate orders."

"The create endpoint isn't idempotent. Have the client send a key and put a unique constraint on it."
