---
description: No deterministic protocol can guarantee consensus in an asynchronous system if even one node may crash.
---

A proof that no deterministic algorithm can guarantee [consensus](./Consensus.md) in a fully asynchronous system if even one process may crash. It is named after Michael Fischer, Nancy Lynch and Michael Paterson, who published it in 1985.

The result depends on its exact assumptions:

| Assumption       | Meaning in the proof                                                  |
| ---------------- | --------------------------------------------------------------------- |
| Asynchronous     | No bound on message delay or relative processing speed, and no clocks |
| Reliable network | Every message is eventually delivered, exactly once                   |
| One crash        | At most one process may stop, at any point                            |
| Deterministic    | Each step is fully determined by a process's state and input          |
| Nontrivial       | Both possible decision values can be reached                          |

The difficulty comes from a process being unable to tell a crashed peer from a slow one. The proof shows that any such protocol has an initial state from which the outcome is not yet fixed, and that an adversarial schedule of message deliveries can keep it undecided forever.

FLP does not say consensus is unachievable in practice. It says no protocol in this model can guarantee termination in every possible execution. The non-terminating schedules can be made unlikely, and systems escape the result by changing an assumption:

| Approach                                    | Added assumption                                                    | Example                                |
| ------------------------------------------- | ------------------------------------------------------------------- | -------------------------------------- |
| Partial synchrony                           | The network eventually behaves within time bounds                   | [Paxos](./Paxos.md), [Raft](./Raft.md) |
| Randomization                               | [Nodes](./Node.md) use coin flips, and terminate with probability 1 | Ben-Or's 1983 algorithm                |
| [Failure detector](./Failure%20detector.md) | An oracle that is eventually accurate enough                        | Chandra-Toueg algorithm                |

The practical consequence appears in how Raft and Paxos fail. Under bad timing, such as repeated [leader election](./Leader%20election.md) rounds on an unstable network, a cluster may stop making progress for as long as the instability lasts. It will not return conflicting decisions.

_Usage:_

"Can we prove our election always finishes within some time limit?"

"Not on an asynchronous network. FLP rules out guaranteed termination. We can prove it never elects two leaders per term, and use randomized timeouts so it converges in practice."
