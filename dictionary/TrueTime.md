---
description: Google Spanner's clock API, which returns a time interval guaranteed to contain the true time.
---

Google's clock API, built for its Spanner database and described in a 2012 paper. Where an ordinary clock returns one value that may be wrong by an unknown amount, TrueTime returns an interval and promises that the true current time lies inside it. This makes [clock skew](./Clock%20skew.md) explicit.

| Call           | Returns                                               |
| -------------- | ----------------------------------------------------- |
| `TT.now()`     | An interval `[earliest, latest]` containing true time |
| `TT.after(t)`  | True if time `t` has definitely passed                |
| `TT.before(t)` | True if time `t` has definitely not arrived           |

To keep the interval narrow, Google runs time masters in each data center with GPS receivers and atomic clocks. Machines poll the masters, and between polls the uncertainty widens according to an assumed worst-case [clock drift](./Clock%20drift.md) rate.

Spanner uses the interval to order transactions through a procedure called commit wait:

| Step | What happens                                                      |
| ---- | ----------------------------------------------------------------- |
| 1    | The transaction acquires its locks                                |
| 2    | The leader picks commit timestamp `s`, at least `TT.now().latest` |
| 3    | The leader replicates the commit through [Paxos](./Paxos.md)      |
| 4    | The leader waits until `TT.after(s)` is true, overlapping step 3  |
| 5    | With both done, the leader releases locks and exposes the result  |

The wait ensures that `s` is in the past before anyone can observe the transaction. If transaction T1 commits before T2 starts, T1 therefore has the smaller timestamp. Google calls this external consistency, and it is equivalent to strict serializability: transactions appear to run one at a time in real-time order, extending [linearizability](./Linearizability.md) to multiple objects.

The cost is that each read-write transaction waits out the uncertainty, so write latency depends on how narrow the interval is. The guarantee also relies on the bound being correct. Without comparable time infrastructure, systems often use a [hybrid logical clock](./Hybrid%20logical%20clock.md) and accept weaker guarantees.

_Usage:_

"Why can't we copy Spanner's commit protocol onto our own Postgres cluster?"

"Commit wait is only safe with a trustworthy uncertainty bound. Without something like TrueTime, plain [NTP](./NTP.md) can't give us that."
