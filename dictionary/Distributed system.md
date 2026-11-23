---
description: A set of computers that coordinate by passing messages over a network, with no shared memory and no shared clock.
---

A set of computers that work toward a shared result by passing messages over a network. Each computer, or [node](./Node.md), has its own memory and its own clock. No node can read another's state directly. It can only send a message and wait for a reply, and what it learns from that reply may already be out of date.

Three differences separate this from a program running on one machine:

| Concern       | On one machine                      | In a distributed system                                                       |
| ------------- | ----------------------------------- | ----------------------------------------------------------------------------- |
| Failure       | The program usually stops as a unit | Some parts fail while others run, a [partial failure](./Partial%20failure.md) |
| Time          | One clock orders events             | Clocks disagree by some amount of [clock skew](./Clock%20skew.md)             |
| Communication | A function call returns or throws   | Messages can be delayed, lost, duplicated or reordered                        |

These differences explain why code that passes every test on a laptop can misbehave in production. A request times out, but the write it carried was applied. Two servers each believe they are in charge, which is [split brain](./Split%20brain.md). A user saves a change and then sees the old value, because the read went to a copy that had not received the update.

Teams distribute a system to survive the loss of a machine, to hold more data or serve more load than one machine can, or to place data closer to users. Each of these goals adds coordination work. If one machine meets the requirements, using one machine avoids this class of problems.

The other entries in this dictionary name the parts of that coordination work: detecting failure, ordering events without a shared clock, keeping copies of data in step, reaching [consensus](./Consensus.md), and defining what a delivery guarantee promises.

_Usage:_

"The service keeps its job queue in memory. Can we run a second instance for redundancy?"

"Not without changing the design. Two instances means a distributed system, so we'd have to decide who owns each job and what happens when they can't reach each other."
