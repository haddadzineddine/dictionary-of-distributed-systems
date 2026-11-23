# Distributed Systems Dictionary

A plain-English dictionary of what goes wrong when computers cooperate over a network: failure, time, consistency, replication, consensus and delivery guarantees.

Why did two nodes both think they were the leader? Why did the user not see their own write? Why is exactly-once so hard?

Each of these has a short explanation once you know the right words. Entries define the term, describe the failure it explains where there is one, and end with a short example of engineers using it.

---

## Table of contents

- **[Section 1 — Failure](#section-1--failure)**
  - [Distributed system](#distributed-system)
  - [Node](#node)
  - [Partial failure](#partial-failure)
  - [Network partition](#network-partition)
  - [Fallacies of distributed computing](#fallacies-of-distributed-computing)
  - [Crash failure](#crash-failure)
  - [Byzantine failure](#byzantine-failure)
  - [Gray failure](#gray-failure)
  - [Heartbeat](#heartbeat)
  - [Failure detector](#failure-detector)
  - [Split brain](#split-brain)
- **[Section 2 — Time and Order](#section-2--time-and-order)**
  - [Clock skew](#clock-skew)
  - [Clock drift](#clock-drift)
  - [NTP](#ntp)
  - [Happens-before](#happens-before)
  - [Logical clock](#logical-clock)
  - [Lamport timestamp](#lamport-timestamp)
  - [Vector clock](#vector-clock)
  - [Hybrid logical clock](#hybrid-logical-clock)
  - [TrueTime](#truetime)
- **[Section 3 — Consistency Models](#section-3--consistency-models)**
  - [CAP theorem](#cap-theorem)
  - [PACELC](#pacelc)
  - [Linearizability](#linearizability)
  - [Sequential consistency](#sequential-consistency)
  - [Causal consistency](#causal-consistency)
  - [Eventual consistency](#eventual-consistency)
  - [Read-your-writes](#read-your-writes)
  - [Monotonic reads](#monotonic-reads)
- **[Section 4 — Replication and Convergence](#section-4--replication-and-convergence)**
  - [State machine replication](#state-machine-replication)
  - [Quorum](#quorum)
  - [Sloppy quorum](#sloppy-quorum)
  - [Hinted handoff](#hinted-handoff)
  - [Read repair](#read-repair)
  - [Anti-entropy](#anti-entropy)
  - [Merkle tree](#merkle-tree)
  - [Gossip protocol](#gossip-protocol)
  - [Last-write-wins](#last-write-wins)
  - [CRDT](#crdt)
- **[Section 5 — Consensus and Coordination](#section-5--consensus-and-coordination)**
  - [Consensus](#consensus)
  - [FLP impossibility](#flp-impossibility)
  - [Paxos](#paxos)
  - [Raft](#raft)
  - [Leader election](#leader-election)
  - [Lease](#lease)
  - [Distributed lock](#distributed-lock)
  - [Fencing token](#fencing-token)
- **[Section 6 — Transactions and Delivery](#section-6--transactions-and-delivery)**
  - [Two-phase commit](#two-phase-commit)
  - [Three-phase commit](#three-phase-commit)
  - [At-most-once delivery](#at-most-once-delivery)
  - [At-least-once delivery](#at-least-once-delivery)
  - [Exactly-once semantics](#exactly-once-semantics)
  - [Idempotency](#idempotency)

## Section 1 — Failure

### Distributed system

A set of computers that work toward a shared result by passing messages over a network. Each computer, or [node](#node), has its own memory and its own clock. No node can read another's state directly. It can only send a message and wait for a reply, and what it learns from that reply may already be out of date.

Three differences separate this from a program running on one machine:

| Concern       | On one machine                      | In a distributed system                                                       |
| ------------- | ----------------------------------- | ----------------------------------------------------------------------------- |
| Failure       | The program usually stops as a unit | Some parts fail while others run, a [partial failure](#partial-failure) |
| Time          | One clock orders events             | Clocks disagree by some amount of [clock skew](#clock-skew)             |
| Communication | A function call returns or throws   | Messages can be delayed, lost, duplicated or reordered                        |

These differences explain why code that passes every test on a laptop can misbehave in production. A request times out, but the write it carried was applied. Two servers each believe they are in charge, which is [split brain](#split-brain). A user saves a change and then sees the old value, because the read went to a copy that had not received the update.

Teams distribute a system to survive the loss of a machine, to hold more data or serve more load than one machine can, or to place data closer to users. Each of these goals adds coordination work. If one machine meets the requirements, using one machine avoids this class of problems.

The other entries in this dictionary name the parts of that coordination work: detecting failure, ordering events without a shared clock, keeping copies of data in step, reaching [consensus](#consensus), and defining what a delivery guarantee promises.

_Usage:_

"The service keeps its job queue in memory. Can we run a second instance for redundancy?"

"Not without changing the design. Two instances means a distributed system, so we'd have to decide who owns each job and what happens when they can't reach each other."

### Node

One participant in a [distributed system](#distributed-system). A node holds its own state and communicates with other participants by sending messages. Papers on distributed algorithms often call it a process.

What counts as a node depends on the layer being discussed:

| Context                               | What "node" usually means                          |
| ------------------------------------- | -------------------------------------------------- |
| [Consensus](#consensus) algorithm | One process that votes and keeps a log             |
| Database cluster                      | One server process holding some or all of the data |
| Container orchestrator                | One machine that runs containers                   |
| Network diagram                       | Any addressable device on the network              |

The difference matters when counting how many failures a system can survive. Three database nodes running as containers on one physical host are three nodes to the database and one failure domain (a set of components that fail together). If the host loses power, all three stop. Replication settings that look safe on paper give no protection in that layout, so placement across hosts, racks or zones has to be checked separately from the node count.

A node's identity also matters. Peers remember what a node has promised, so a node that loses its disk and rejoins under the same identity can break a protocol's assumptions. A [Raft](#raft) node that forgets its vote could vote twice in one election, which could produce two leaders. For this reason, a node that has lost its durable state should normally rejoin as a new member through the cluster's membership process.

Nodes are usually described as correct or faulty. A correct node follows the protocol. A faulty node has stopped, as in a [crash failure](#crash-failure), or misbehaves, as in a [Byzantine failure](#byzantine-failure).

_Usage:_

"We run three nodes, so losing one is fine."

"Check the placement first. If two of them share a host, losing that host takes out a [quorum](#quorum) and writes stop."

### Partial failure

A condition in which some parts of a [distributed system](#distributed-system) fail while others keep working. A program on one machine usually fails as a unit. Across a network, one [node](#node) can stop, slow down or lose contact while the rest continue, and the nodes still running may not know.

The practical difficulty is that a caller often cannot tell what happened. When a remote call times out, any of these may be true:

| What happened                      | Was the work done?     | What the caller sees |
| ---------------------------------- | ---------------------- | -------------------- |
| Request lost on the way            | No                     | Timeout              |
| Receiver crashed before processing | No                     | Timeout              |
| Receiver crashed after processing  | Yes                    | Timeout              |
| Response lost on the way back      | Yes                    | Timeout              |
| Receiver is slow and still working | Not yet, but it may be | Timeout              |

A timeout therefore means "outcome unknown", not "failed". Code that treats it as a failure produces a familiar bug: a payment call times out, the client retries, and the customer is charged twice because the first attempt had succeeded. Code that does not retry has the opposite problem, and the payment may never happen.

Handling partial failure means designing each remote operation for the unknown case. Set a timeout on every call, so a lost response cannot hang the caller indefinitely. Make retried operations safe through [idempotency](#idempotency), usually with a client-generated request ID that the receiver records. For multi-step work, decide how an interrupted sequence is finished or undone, and run a periodic reconciliation (a job that compares records and corrects mismatches) where money or other important state is involved.

_Usage:_

"The charge endpoint returned a 504, so the order was marked failed. But the customer's bank shows the payment."

"That's partial failure. The gateway timed out after the charge went through. We should record the attempt with an idempotency key and query its status before retrying."

### Network partition

A network fault that divides [nodes](#node) into groups that cannot exchange messages across the divide, while communication within each group continues. Every node may be healthy and still serving requests. The cause can be a failed switch, a bad firewall rule, a routing change, or a lost link between data centers.

From inside one group, a partition looks the same as the other group having crashed: messages stop arriving and [heartbeats](#heartbeat) time out. This ambiguity produces a typical incident. Each side concludes that the other is dead, both keep accepting writes, and when the network heals the data has diverged. That outcome is [split brain](#split-brain).

Partitions are not always clean:

| Shape        | What happens                                    |
| ------------ | ----------------------------------------------- |
| Complete     | Groups exchange no messages in either direction |
| Asymmetric   | A can reach B, but B cannot reach A             |
| Partial      | A and B cannot talk, but both can still reach C |
| Intermittent | Links drop and recover repeatedly               |

The asymmetric and partial shapes are harder to handle because nodes disagree about who is reachable. A node may stop receiving a leader's messages while its own messages still get through, and may then start elections that disrupt the cluster.

During a partition, a system holding replicated data has to make the choice described by the [CAP theorem](#cap-theorem). It can refuse some requests to preserve consistency, usually by letting only the side with a [quorum](#quorum) proceed. Or it can keep serving on both sides and reconcile afterwards with a rule such as [last-write-wins](#last-write-wins) or a data structure such as a [CRDT](#crdt). The choice should be made per use case and tested by injecting partitions, because the code paths involved rarely run in normal operation.

_Usage:_

"Both regions look healthy on their own dashboards, but replication lag keeps climbing."

"That looks like a network partition between them. Check which side has quorum before anyone fails over, or we'll end up with two leaders."

### Fallacies of distributed computing

Eight false assumptions that developers commonly make when writing software for a [distributed system](#distributed-system). The list is usually attributed to L Peter Deutsch and colleagues at Sun Microsystems in the 1990s.

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

The fallacies persist because a development machine makes them approximately true. On localhost, calls do not get lost, latency is negligible and nobody else changes the configuration. Code written there can pass its tests and then fail under production conditions, often as a [partial failure](#partial-failure) that is hard to reproduce locally.

Libraries for remote procedure calls (calling a function on another machine) add to the problem by making a network call look like a local one.

To counter the assumptions, treat each network call as an operation that can fail. Give it a timeout, decide whether it can be retried safely, batch work to reduce round trips, and authenticate traffic between services. Test with injected latency and packet loss, so the failure paths run before production exercises them.

_Usage:_

"The export worked in staging, but in production it takes twenty minutes."

"It makes a separate call to the pricing service per row. That's the 'latency is zero' fallacy. Staging runs both services on one box."

### Crash failure

A failure in which a [node](#node) stops executing and sends no further messages. Before it stops, the node follows its protocol correctly. This is the failure assumption behind many replication and agreement algorithms, because a stopped node cannot send false information.

Formal work separates failures into models, which define what a faulty node is allowed to do:

| Model                                 | What a faulty node may do                                      |
| ------------------------------------- | -------------------------------------------------------------- |
| Crash-stop                            | Halts permanently                                              |
| Crash-recovery                        | Halts, then may restart with whatever it saved to disk         |
| Omission                              | Keeps running but fails to send or receive some messages       |
| [Byzantine](#byzantine-failure) | Does anything, including sending false or conflicting messages |

Production systems mostly face crash-recovery, since machines reboot and processes restart. [Paxos](#paxos) and [Raft](#raft) are designed for this model. Their safety depends on a restarted node remembering what it wrote to stable storage, such as the votes it cast and the log entries it acknowledged.

That dependency explains a common data-loss bug. A node acknowledges a write while the data is still in the operating system's buffer, the machine loses power, and the write is gone after the restart even though the client was told it succeeded. Durable systems call fsync (an operation that forces buffered data to disk) before acknowledging, and accept the added latency.

Other nodes cannot observe a crash directly. They see only silence, which could also come from a slow node or a [network partition](#network-partition). A [failure detector](#failure-detector) can suspect a crash but, in an asynchronous network, cannot confirm one. A node that was declared dead may still be running, so recovery logic has to allow for it to return.

_Usage:_

"The replica acknowledged the write, then the host rebooted and the row was gone."

"So it acked before syncing to disk. In a crash-recovery model that promise means nothing unless it survives the restart."

### Byzantine failure

A failure in which a [node](#node) deviates from its protocol in any way, including sending false data or telling different peers different things. The name comes from the Byzantine Generals Problem, published by Lamport, Shostak and Pease in 1982, in which loyal generals must agree on a plan even though some of their number are traitors.

A Byzantine node need not be malicious. Faulty memory, a firmware bug, corrupted data or two software versions that interpret a message differently can all make a node produce wrong output while appearing to work. An attacker who controls a node is the other main case.

This model is harder to tolerate than a [crash failure](#crash-failure), because other nodes cannot take a message at face value:

| Failure model | Nodes needed to tolerate f faults with [consensus](#consensus) in a partially synchronous network |
| ------------- | ----------------------------------------------------------------------------------------------------- |
| Crash         | 2f + 1                                                                                                |
| Byzantine     | 3f + 1                                                                                                |

Partially synchronous means the network may misbehave for a while but eventually delivers messages within some bound. Under those conditions, tolerating one Byzantine node takes four nodes, compared with three for one crash.

[Paxos](#paxos) and [Raft](#raft) do not tolerate Byzantine failures. A single Raft leader that lies can corrupt the replicated log. Byzantine fault-tolerant protocols, such as PBFT (Practical Byzantine Fault Tolerance), add extra message rounds and cryptographic verification. They are mainly used where participants do not trust each other, as in blockchains.

Systems run by one organization usually assume crash failures and handle the likeliest forms of corruption separately. Checksums on disk blocks and network messages turn silent corruption into a detectable error, which the system can then treat as a crash and repair from another copy.

_Usage:_

"One replica keeps returning a different hash for the same block. Could Raft have caught that?"

"No, Raft assumes nodes don't lie. That's a Byzantine failure, probably bad memory or disk. Add checksums so corruption shows up as an error."

### Gray failure

A degraded state in which a component appears healthy to some observers while it is failing for others. Researchers describe this as differential observability: the monitoring system and the application see different realities. It differs from a clean [crash failure](#crash-failure), where the component stops and every observer eventually notices.

The typical symptom is a dashboard that stays green during a user-visible outage. A server answers its health check on a lightweight endpoint, so the load balancer keeps sending it traffic, while real requests fail because they depend on a disk, downstream connection or thread pool that the check does not exercise.

Common forms include:

| Component    | Degradation                                    | Why checks may miss it                        |
| ------------ | ---------------------------------------------- | --------------------------------------------- |
| Disk         | Some reads take seconds                        | The check does not touch the slow path        |
| Network card | Drops a small share of packets                 | Small probes usually get through or retry     |
| Process      | Worker threads stuck, listener alive           | The health endpoint runs on a separate thread |
| Dependency   | One shard (a slice of the data) is unavailable | The check queries a different shard           |

Gray failures can also prevent automatic recovery. A [failure detector](#failure-detector) that relies on [heartbeats](#heartbeat) keeps reporting the [node](#node) as alive, so no failover happens. A slow node can be worse than a dead one, because requests queue behind it and hold resources in its callers, which can spread the slowdown to other services.

Detection works better from the perspective of the callers. Track error rate and latency percentiles per instance, compare each instance to its peers, and remove outliers from rotation. Make health checks exercise the same dependencies as real requests, with care that a shared dependency failing does not mark every instance unhealthy at once.

_Usage:_

"All instances pass health checks, but about one percent of checkouts hang."

"Break latency down by host. If one stands out, it's a gray failure, probably its disk. Drain it even though the health check says it's fine."

### Heartbeat

A small message that a [node](#node) sends at regular intervals so that other nodes can tell it is still reachable. If the expected heartbeats stop arriving for longer than a timeout, the receiver begins to suspect the sender has failed. In [Raft](#raft), for example, a leader sends periodic messages to its followers, and a follower that hears nothing for its election timeout starts a new election.

A missing heartbeat does not prove a crash. The sender may also be paused, overloaded, or cut off by a [network partition](#network-partition). A heartbeat system therefore produces suspicion, which a [failure detector](#failure-detector) turns into a decision.

Two settings control the behavior, and each one trades detection speed against false alarms:

| Setting  | Lower value                                | Higher value                          |
| -------- | ------------------------------------------ | ------------------------------------- |
| Interval | Faster detection, more network and CPU use | Less overhead, slower detection       |
| Timeout  | Faster failover, more false positives      | Fewer false positives, longer outages |

A timeout set too low produces a recognizable pattern: leaders change repeatedly during load spikes although no machine has failed. A long garbage collection pause (the runtime halting a program to reclaim memory) or a saturated disk delays a heartbeat past the timeout, the cluster fails over, and the failover adds more load. The timeout should sit well above the normal variation in heartbeat delay, and the interval is usually a fraction of the timeout so that one lost message does not trigger suspicion.

A heartbeat shows only that the code sending it is running. If it runs on a dedicated thread, it can continue while the threads doing real work are stuck, which is one source of [gray failure](#gray-failure). Tying the heartbeat to real progress, such as a worker completing its loop, narrows that gap.

_Usage:_

"We had three [leader elections](#leader-election) during the batch job, but no node went down."

"The heartbeats are probably missing the timeout during GC pauses. Check the pause times before we raise it."

### Failure detector

A component that reports which [nodes](#node) it suspects have failed, usually based on missing [heartbeats](#heartbeat) or timed-out requests. In an asynchronous network, where messages have no upper bound on delay, a slow node and a dead node look identical, so no detector can be right every time.

Chandra and Toueg formalized failure detectors in the 1990s using two kinds of property:

| Property     | Question it answers                         | Violation looks like                  |
| ------------ | ------------------------------------------- | ------------------------------------- |
| Completeness | Is every crashed node eventually suspected? | A dead node stays in rotation         |
| Accuracy     | Are working nodes left unsuspected?         | A healthy node is removed or replaced |

Real detectors have to trade one against the other. A short timeout finds crashes quickly but raises false positives, and a long timeout does the reverse. A false positive is costly because it can trigger a failover, data rebalancing, or a leader change on a cluster that was working. A pause on one node can then turn into cluster-wide churn.

Some detectors output a level of suspicion in place of a yes-or-no answer. The phi accrual detector, used by Cassandra and Akka, compares the time since the last heartbeat with the recent distribution of arrival times. Each application then picks its own threshold for acting.

Failure detectors also matter in theory. [FLP impossibility](#flp-impossibility) rules out deterministic [consensus](#consensus) in a purely asynchronous system with even one crash. Chandra and Toueg showed that consensus becomes solvable with a majority of correct nodes and a detector that may make mistakes, provided it eventually suspects every crashed node and eventually stops suspecting at least one correct node.

Treat a detector's output as a hint. Actions that are unsafe if the node is still alive need protection such as a [fencing token](#fencing-token).

_Usage:_

"The detector marked node four dead, but it's serving traffic."

"It's a suspicion, not a fact. Node four probably stalled for a few seconds. Make sure the failover fences it off."

### Split brain

A state in which two parts of a cluster each act as the authority at the same time. Usually this means two [nodes](#node) both believe they are the leader and both accept writes. The histories then diverge, and when the cluster reunites there may be no correct way to combine them.

The usual trigger is a [network partition](#network-partition): each side stops hearing from the other, assumes it has died, and promotes or keeps its own leader. A pause can do the same without any network fault. A leader stalls during a long garbage collection (the runtime halting the program to reclaim memory), the others elect a replacement, and the old leader wakes up still believing it is in charge.

The damage appears as conflicting records. Two customers hold the same reserved seat, or an account balance reflects only half of its transactions. Repair often means choosing one history and manually salvaging the other.

Systems layer several defences:

| Defence                               | How it helps                                                     |
| ------------------------------------- | ---------------------------------------------------------------- |
| Majority [quorum](#quorum)        | At most one side of a partition can hold a majority              |
| Term or epoch numbers                 | A newer leader's number outranks the old one, so peers reject it |
| [Fencing token](#fencing-token) | Storage rejects writes from a superseded leader                  |
| Node fencing                          | The cluster cuts power or storage access to the old leader       |

A quorum does not stop an old leader from believing it still leads. In [Raft](#raft), a deposed leader cut off from the majority can hold that belief for a while, but it cannot commit new entries. It could still serve stale reads unless it confirms its leadership first. Two-node clusters cannot form a majority after losing contact, so they need a third voter, often called a witness or tiebreaker.

_Usage:_

"After the switch came back, both database nodes were claiming to be primary."

"That's split brain. Stop writes on one before they diverge further, and add a witness so a two-node pair can't do this again."

## Section 2 — Time and Order

### Clock skew

The difference between the readings of two clocks at the same instant. If one [node](#node) reads 12:00:00.000 while another reads 12:00:00.250, the skew between them is 250 milliseconds. Every machine keeps its own clock, so some skew is always present. [Clock drift](#clock-drift) is the main cause: clocks run at slightly different rates, and the gap grows until something corrects it.

Skew causes bugs wherever code compares timestamps taken on different machines:

| Where               | What skew does                                                                    |
| ------------------- | --------------------------------------------------------------------------------- |
| Logs and traces     | An effect appears to happen before its cause                                      |
| Auth tokens         | A token looks issued in the future, or expires early                              |
| Caches              | Entries with an expiry time live too long or too short                            |
| Conflict resolution | With [last-write-wins](#last-write-wins), an older write can beat a newer one |

The conflict resolution case loses data with no error. A user updates a record through a node whose clock is behind, and the update carries an earlier timestamp than the value it replaces. The store keeps the old value and reports success.

[NTP](#ntp) reduces skew but does not remove it, and its accuracy depends on network conditions. Code should therefore tolerate a margin: auth libraries commonly allow some leeway, for example. Monitor the offset of each host, because a broken time daemon produces no errors of its own.

Timestamps from different machines cannot establish which event came first when the gap between events is within the skew. To order events reliably, use the [happens-before](#happens-before) relation through a [logical clock](#logical-clock), or a single authority that assigns sequence numbers.

Some network measurement documents use "skew" for a difference in clock rate. In writing about [distributed systems](#distributed-system), it normally means the difference in readings.

_Usage:_

"The trace says the database responded before the API sent the query."

"Those spans come from different hosts. It's clock skew, so don't read anything into the order at that scale."

### Clock drift

The rate at which a clock runs fast or slow compared with a reference clock. Computer clocks count oscillations of a quartz crystal, and no two crystals run at exactly the same frequency. Temperature, age and manufacturing variation all shift the rate.

Drift is usually measured in parts per million (ppm). A clock that is off by 50 ppm gains or loses 4.32 seconds per day.

Drift and [clock skew](#clock-skew) are related but distinct:

| Term  | Measures                             | Example     |
| ----- | ------------------------------------ | ----------- |
| Drift | Rate of error of one clock           | 20 ppm fast |
| Skew  | Gap between two clocks at one moment | 40 ms apart |

Left alone, drift turns into growing skew. [NTP](#ntp) counters this by measuring the clock against a time server and correcting it, and it also estimates the local frequency error so it can compensate between checks. The failure to watch for is a machine whose time synchronization has silently stopped. Nothing breaks at first. Days later, its timestamps are seconds away from its peers, and token validation or log ordering starts to go wrong on that host only.

Drift also affects timers, which matters for anything based on a duration. A [lease](#lease) holder and its grantor each measure the lease period on their own clocks. If the holder's clock runs slow, it can believe the lease is still valid after the grantor considers it expired. Lease protocols handle this by assuming a maximum drift rate and building in a safety margin.

For measuring elapsed time, use a monotonic clock (a timer that only moves forward). It does not jump when the wall clock is corrected, but it still drifts, so it does not remove the need for that margin.

_Usage:_

"This one VM keeps rejecting valid auth tokens, and only that VM."

"Check its time sync daemon. If sync stopped weeks ago, clock drift has had plenty of time to push it outside the leeway window."

### NTP

Network Time Protocol, the standard way for machines to synchronize their clocks over a network. A client exchanges timestamped packets with one or more time servers, estimates how far its clock is off, and corrects it. It runs over UDP on port 123, and common implementations include ntpd and chrony.

Servers are arranged in levels called strata. Stratum 0 is a reference source such as an atomic clock or GPS receiver. A stratum 1 server is attached directly to one, a stratum 2 server synchronizes from stratum 1, and so on.

Each measurement uses four timestamps:

| Timestamp | Taken by | Moment            |
| --------- | -------- | ----------------- |
| t1        | Client   | Request sent      |
| t2        | Server   | Request received  |
| t3        | Server   | Response sent     |
| t4        | Client   | Response received |

From these, the client computes `offset = ((t2 - t1) + (t3 - t4)) / 2` and `delay = (t4 - t1) - (t3 - t2)`. The offset formula assumes the request and response took equal time. When the paths are asymmetric, the result is wrong by half the difference between the two one-way delays, and the client cannot detect this from the timestamps alone. NTP therefore narrows [clock skew](#clock-skew) but cannot eliminate it.

The client applies a correction in one of two ways. Slewing runs the clock slightly fast or slow until the error closes. Stepping sets the clock to the new value at once, and is normally reserved for large errors.

A step can move the wall clock backward, which produces confusing bugs: a measured duration comes out negative, or a timeout fires early or late. Code that measures elapsed time should use a monotonic clock (a timer that only moves forward). NTP also keeps correcting [clock drift](#clock-drift) between measurements, so a stopped daemon lets error build up with no visible failure.

_Usage:_

"Our request duration metric recorded negative values for a few seconds last night."

"NTP probably stepped the clock backward. That code should use a monotonic clock, not wall-clock time."

### Happens-before

A relation on events, defined by Leslie Lamport in 1978, that captures when one event could have influenced another. It orders events by the flow of information and makes no reference to clock time, which makes it usable in a [distributed system](#distributed-system) where clocks disagree.

Event A happens before event B, written A → B, if any of these rules applies:

| Rule         | Condition                                                   |
| ------------ | ----------------------------------------------------------- |
| Local order  | A and B are on the same [node](#node), and A came first |
| Message      | A is the sending of a message, and B is its receipt         |
| Transitivity | A → C and C → B for some event C                            |

If neither A → B nor B → A holds, the two events are concurrent. Concurrent does not mean simultaneous. It means no chain of messages connects them, so neither could have affected the other, even if one occurred an hour earlier by the wall clock.

This makes happens-before a partial order: some pairs of events have a defined order and others do not.

The relation explains a common display bug. In a chat app, a reply appears before the question it answers, because the two messages reached a replica in the wrong order or were sorted by timestamps from machines with [clock skew](#clock-skew). The reply causally depends on the question, so any correct ordering must place the question first. [Causal consistency](#causal-consistency) is the guarantee that preserves this order.

Note that A → B means A could have influenced B, not that it did. The relation tracks potential causality. A [logical clock](#logical-clock) makes it computable: a [Lamport timestamp](#lamport-timestamp) respects the order, and a [vector clock](#vector-clock) can also tell whether two events are concurrent.

_Usage:_

"These two writes are a millisecond apart. Which one came first?"

"Neither, as far as we can tell. No message links them, so there's no happens-before relation. They're concurrent and we need a conflict rule."

### Logical clock

A mechanism that assigns numbers to events so they can be ordered by causality instead of physical time. It counts events and messages and does not measure seconds. A [distributed system](#distributed-system) uses one because physical clocks on different [nodes](#node) disagree through [clock skew](#clock-skew), so their readings cannot reliably show which of two nearby events came first.

The order a logical clock tracks is the [happens-before](#happens-before) relation. If event A could have influenced event B, the clock gives A a value that sorts before B's. Nodes achieve this by attaching their clock value to each message and updating their own clock when a message arrives, so knowledge of order travels with the data.

The main variants differ in what they can tell you and what they cost:

| Variant                                               | Stores                       | Detects concurrency? | Close to wall-clock time? |
| ----------------------------------------------------- | ---------------------------- | -------------------- | ------------------------- |
| [Lamport timestamp](#lamport-timestamp)         | One counter                  | No                   | No                        |
| [Vector clock](#vector-clock)                   | One counter per node         | Yes                  | No                        |
| [Hybrid logical clock](#hybrid-logical-clock) | Physical time plus a counter | No                   | Yes                       |

Choose by the question you need answered. If every node must rank events the same way, for example to break ties between updates, a Lamport timestamp with a node ID tie-breaker is enough. If you need to know that two updates conflicted, because neither saw the other, you need a vector clock. If you want timestamps that respect causality and can also be queried by approximate real time, as with snapshot reads in a database, a hybrid logical clock fits.

A logical clock cannot see causal links that exist outside the system. If one user makes a change and phones a colleague, who then makes a second change, the system sees no message linking the two and may treat them as concurrent.

_Usage:_

"Can we sort these events by created_at across services?"

"Not safely, the hosts' clocks differ. Have each service carry a logical clock in its messages and sort on that."

### Lamport timestamp

A [logical clock](#logical-clock) made of a single integer counter on each [node](#node), introduced by Leslie Lamport in 1978. It guarantees that if event A [happens before](#happens-before) event B, then A's timestamp is smaller than B's.

The algorithm has three rules:

| Event           | What the node does                           |
| --------------- | -------------------------------------------- |
| Local event     | Adds 1 to its counter                        |
| Send message    | Adds 1 to its counter and attaches the value |
| Receive message | Sets its counter to max(local, received) + 1 |

For example, node A at counter 5 sends a message, which increments its counter to 6 and carries the value 6. Node B, at counter 2, receives it and moves to max(2, 6) + 1 = 7. B's receive event now sorts after A's send, even though B's counter was far behind.

The guarantee runs in one direction only. If A happens before B, then L(A) < L(B). The reverse does not hold: L(A) < L(B) does not show that A happened before B, because the two events may be concurrent. Misreading this is the usual mistake. Code that treats the higher timestamp as "the later write that saw the earlier one" will silently discard a concurrent update. To detect concurrency you need a [vector clock](#vector-clock).

Two events on different nodes can have the same counter value. Breaking ties by node ID, comparing (counter, node ID) pairs, gives a total order in which every pair of events is ranked and every node agrees on the ranking. That total order is consistent with causality, though its ordering of concurrent events is arbitrary. Lamport used this to implement mutual exclusion in his original paper, and [Paxos](#paxos) implementations commonly build proposal numbers from the same counter-plus-ID pair.

_Usage:_

"This write has Lamport timestamp 42 and that one has 40, so the first one saw the second, right?"

"Not necessarily. A higher Lamport timestamp doesn't prove causality. They could be concurrent, and we'd need vector clocks to tell."

### Vector clock

A [logical clock](#logical-clock) made of a list of counters, one per [node](#node). Each node's vector records how many events it knows about from every node, including itself. Colin Fidge and Friedemann Mattern independently published the technique in 1988.

The update rules are:

1. Before each local event or send, a node increments its own entry.
2. Each message carries the sender's full vector.
3. On receipt, a node takes the element-wise maximum of its vector and the message's vector, then increments its own entry.

Two vectors are compared element by element. V(A) is less than V(B) if every entry of A is less than or equal to the matching entry of B, and at least one is strictly less.

| A         | B         | Result              |
| --------- | --------- | ------------------- |
| [2, 1, 0] | [2, 2, 0] | A happened before B |
| [3, 2, 1] | [2, 2, 0] | B happened before A |
| [3, 0, 0] | [2, 1, 0] | Concurrent          |
| [2, 1, 0] | [2, 1, 0] | Same event          |

Vector clocks characterize [happens-before](#happens-before) exactly: A happened before B if and only if V(A) < V(B). A [Lamport timestamp](#lamport-timestamp) offers only the forward direction. This is what lets a store detect a true conflict. If two clients both edit a shopping cart based on the same version, their vectors come out concurrent, so the store can keep both versions for the application to merge, where a timestamp rule would drop one. A close relative, the version vector, tracks versions of one data item across its copies.

The cost is size. Each timestamp grows with the number of participants. Pruning old entries saves space but can misclassify concurrent updates as ordered, or the reverse.

_Usage:_

"Why did the cart come back with two versions of the same item list?"

"The vector clocks were concurrent, so neither write had seen the other. The store kept both, and our merge code has to reconcile them."

### Hybrid logical clock

A timestamp that combines a physical clock reading with a [logical clock](#logical-clock) counter. It preserves the [happens-before](#happens-before) order and stays close to real time. Kulkarni and colleagues described the design in 2014, and databases such as CockroachDB use a variant of it.

Each timestamp has two parts: `l`, the largest physical time this [node](#node) has seen, and `c`, a counter that orders events sharing the same `l`. Timestamps compare by `l` first, then `c`. On a local event or send, `l` advances to the physical clock if that is larger, and `c` resets to 0. Otherwise `c` increments. On a receive, `l` becomes the largest of the local `l`, the message's `l` and the physical clock, and `c` is set so the new timestamp sorts after both inputs, or to 0 if the physical clock alone is largest.

Plain physical timestamps can violate causality under [clock skew](#clock-skew), and a [Lamport timestamp](#lamport-timestamp) has no relation to real time, so it cannot answer "what was the state at 14:00?". A hybrid logical clock gives a single sortable value that does both.

Its guarantee has the same one-way shape as Lamport's. If A happened before B, then A's timestamp is smaller. A smaller timestamp does not prove causality, so it cannot detect concurrency the way a [vector clock](#vector-clock) can.

It still depends on reasonable clock synchronization. One node with a clock far ahead pushes `l` forward on every node it talks to, and those nodes carry that future time until physical time catches up. Systems therefore enforce a maximum tolerated offset. CockroachDB, for example, shuts down a node whose clock strays too far from its peers.

_Usage:_

"Why did this node crash with a clock offset error?"

"Its clock drifted past the configured maximum. Our hybrid logical clock relies on that bound, so the node stops itself before it can break ordering."

### TrueTime

Google's clock API, built for its Spanner database and described in a 2012 paper. Where an ordinary clock returns one value that may be wrong by an unknown amount, TrueTime returns an interval and promises that the true current time lies inside it. This makes [clock skew](#clock-skew) explicit.

| Call           | Returns                                               |
| -------------- | ----------------------------------------------------- |
| `TT.now()`     | An interval `[earliest, latest]` containing true time |
| `TT.after(t)`  | True if time `t` has definitely passed                |
| `TT.before(t)` | True if time `t` has definitely not arrived           |

To keep the interval narrow, Google runs time masters in each data center with GPS receivers and atomic clocks. Machines poll the masters, and between polls the uncertainty widens according to an assumed worst-case [clock drift](#clock-drift) rate.

Spanner uses the interval to order transactions through a procedure called commit wait:

| Step | What happens                                                      |
| ---- | ----------------------------------------------------------------- |
| 1    | The transaction acquires its locks                                |
| 2    | The leader picks commit timestamp `s`, at least `TT.now().latest` |
| 3    | The leader replicates the commit through [Paxos](#paxos)      |
| 4    | The leader waits until `TT.after(s)` is true, overlapping step 3  |
| 5    | With both done, the leader releases locks and exposes the result  |

The wait ensures that `s` is in the past before anyone can observe the transaction. If transaction T1 commits before T2 starts, T1 therefore has the smaller timestamp. Google calls this external consistency, and it is equivalent to strict serializability: transactions appear to run one at a time in real-time order, extending [linearizability](#linearizability) to multiple objects.

The cost is that each read-write transaction waits out the uncertainty, so write latency depends on how narrow the interval is. The guarantee also relies on the bound being correct. Without comparable time infrastructure, systems often use a [hybrid logical clock](#hybrid-logical-clock) and accept weaker guarantees.

_Usage:_

"Why can't we copy Spanner's commit protocol onto our own Postgres cluster?"

"Commit wait is only safe with a trustworthy uncertainty bound. Without something like TrueTime, plain [NTP](#ntp) can't give us that."

## Section 3 — Consistency Models

### CAP theorem

A result stating that a replicated data store cannot guarantee both [linearizability](#linearizability) and full availability while a [network partition](#network-partition) is in progress. Eric Brewer proposed it as a conjecture in 2000, and Seth Gilbert and Nancy Lynch proved a formal version in 2002 for an asynchronous network.

The letters have narrower meanings in the proof than in everyday use:

| Letter | Meaning in the proof                                                                         | Common misreading                          |
| ------ | -------------------------------------------------------------------------------------------- | ------------------------------------------ |
| C      | Consistency: reads and writes on a register are linearizable                                 | Transaction validity rules (the C in ACID) |
| A      | Availability: every request to a non-failing [node](#node) gets a response               | An uptime percentage                       |
| P      | Partition tolerance: the guarantees hold even if the network loses arbitrarily many messages | A feature you can opt out of               |

Suppose a partition separates two replicas (nodes holding copies of the data). A client writes a new value to one. Another client then reads from the other. To be available, the second replica must answer, but no message can tell it about the write, so it returns the old value and linearizability is broken. If it refuses to answer, availability is broken.

The popular summary, "pick two of three", is misleading. A system running over a real network cannot opt out of partitions, so the choice applies only while one is happening. With a healthy network, a system can provide both C and A.

A majority-[quorum](#quorum) database is called CP, yet clients on the majority side keep working during a partition. It fails the formal A only because minority-side nodes refuse requests. A database with asynchronous replication and automatic failover meets neither definition, since it can lose acknowledged writes and rejects requests during failover. Ask what a system does in each failure case. [PACELC](#pacelc) extends the framing to normal operation.

_Usage:_

"The vendor says their database beats CAP."

"Ask what a minority-side node does during a partition. It either refuses the request or risks returning stale data."

### PACELC

An extension of the [CAP theorem](#cap-theorem), proposed by Daniel Abadi, that also covers how a replicated system behaves when the network is healthy. The name spells out the rule: if there is a Partition, choose Availability or Consistency; Else, choose Latency or Consistency.

CAP says nothing about normal operation, yet that is where a system spends most of its time and where its replication design has a constant cost. To confirm a write on several replicas ([nodes](#node) holding copies of the data) before responding, a system has to wait for network round trips. To respond faster, it must acknowledge before all replicas have the update, or read from a replica that may be behind.

Systems are classified with a two-part label:

| Label | During a partition    | Normal operation   | Typical design                                                          |
| ----- | --------------------- | ------------------ | ----------------------------------------------------------------------- |
| PA/EL | Stays available       | Favors low latency | Leaderless store with low read and write [quorum](#quorum) settings |
| PC/EC | Refuses some requests | Favors consistency | Every operation goes through [consensus](#consensus)                |
| PC/EL | Refuses some requests | Favors low latency | Single leader for writes, with stale reads from followers               |
| PA/EC | Stays available       | Favors consistency | Consistent reads when healthy, weaker reads during failures             |

The framework explains a symptom CAP cannot. A team picks a strongly consistent multi-region database, nothing is failing, and every write still takes as long as a round trip between regions. That is the EC side of the trade, paid on each request.

Labels describe a configuration more than a product. A store such as Cassandra can move toward the consistent end by raising its read and write quorum levels. Use the framework to ask two separate questions of each workload: what should happen during a partition, and how much latency is acceptable for fresher data the rest of the time.

_Usage:_

"Nothing's down, so why does every write take 90 milliseconds?"

"We replicate synchronously across regions. In PACELC terms we chose consistency over latency, and that's the round trip."

### Linearizability

A consistency model in which every operation appears to take effect atomically at a single instant between its invocation and its response. Maurice Herlihy and Jeannette Wing defined it in 1990. To clients, the object behaves as if one copy of the data existed, even with several replicas ([nodes](#node) holding copies).

Formally, a history of operations is linearizable if they can be arranged in one total order that meets two conditions. The order must be legal for the object, so each read returns the latest value written before it. It must also respect real time: if operation A completes before operation B starts, A comes before B. Here `x` starts at 0:

| Time | Client A              | Client B                 | Client C                |
| ---- | --------------------- | ------------------------ | ----------------------- |
| 1    | `write(x, 1)` starts  |                          |                         |
| 2    |                       | `read(x)` returns 0 or 1 |                         |
| 3    | `write(x, 1)` returns |                          |                         |
| 4    |                       |                          | `read(x)` must return 1 |

A read that overlaps a write may return either value. Once any read returns the new value, every read starting after it must return that value or a later one.

It is defined per operation on a single object, unlike serializability (transactions appear to run one at a time, with no real-time requirement). The combination of the two is called strict serializability. Linearizability is also composable: if each object is linearizable, the system of objects is too. [Sequential consistency](#sequential-consistency) lacks this property.

Without linearizability, a client can update a value, tell another client, and that client can still read the old value from a lagging replica. Locks, [leader election](#leader-election) and uniqueness checks all rely on this guarantee. Providing it requires coordination, usually [consensus](#consensus), which adds latency. By the [CAP theorem](#cap-theorem), it also cannot remain available on both sides of a [network partition](#network-partition).

_Usage:_

"I acquired the lock, but another worker read it as free a moment later."

"Then those reads aren't linearizable. They're probably hitting a follower. Lock checks have to go through the consensus path."

### Sequential consistency

A consistency model in which all operations appear in one total order that every client agrees on, and each client's own operations appear in that order in the sequence the client issued them. Leslie Lamport defined it in 1979, originally for multiprocessor memory.

It is weaker than [linearizability](#linearizability) in one specific way. Linearizability requires the agreed order to match real time, while sequential consistency does not. An operation may appear to take effect before it started or after it finished.

| Requirement                             | Linearizable | Sequentially consistent |
| --------------------------------------- | ------------ | ----------------------- |
| One total order agreed by all clients   | Yes          | Yes                     |
| Each client's own order preserved       | Yes          | Yes                     |
| Order respects real time across clients | Yes          | No                      |
| Composable across objects               | Yes          | No                      |

Here is a history that is sequentially consistent but not linearizable. Suppose `x` starts at 0. Client A writes `x = 1` and the write returns. Later, client B reads `x` and gets 0. Placing B's read before A's write yields a valid single order, so sequential consistency allows it. Linearizability forbids it, since A's write finished before B's read began. What B cannot do is read 1 and then read 0 afterwards, because that would contradict any single order.

This produces a recognizable surprise. One service finishes a write and tells another over a queue or an HTTP call, and the second service reads stale data even though the store is meeting its guarantee. The notification traveled through a channel the storage system cannot see.

ZooKeeper shows the model in practice: updates are applied in a single order, but a client reading from a follower can see data that is behind the leader. Clients that need a fresher view call `sync` before reading.

_Usage:_

"Service A committed the config, then called service B, and B read the old version."

"The store is only sequentially consistent, so B's read can be ordered before the write. Have B call sync first."

### Causal consistency

A consistency model in which every [node](#node) sees causally related operations in the same order, while concurrent operations may be seen in different orders. Two operations are causally related if one could have influenced the other, as defined by the [happens-before](#happens-before) relation. A client's earlier operations precede its later ones, and a write precedes any read that observes it.

Consider a comment thread. Ana posts a question (W1). Ben reads it and posts an answer (W2). Cleo, who has seen neither, posts an unrelated comment (W3). W1 causally precedes W2, and W3 is concurrent with both.

| Order seen by a replica | Allowed? | Reason                                  |
| ----------------------- | -------- | --------------------------------------- |
| W1, W2, W3              | Yes      | W1 is before W2                         |
| W3, W1, W2              | Yes      | W3 is concurrent, so it can go anywhere |
| W1, W3, W2              | Yes      | W1 is before W2                         |
| W2, W1, W3              | No       | The answer appears before its question  |

This model prevents the anomaly in which a reply shows up with no visible question, or a user removes a colleague from a shared album and then posts a photo, only for the colleague to see the photo because the two updates arrived in the opposite order.

Causal consistency sits between [eventual consistency](#eventual-consistency) and [sequential consistency](#sequential-consistency). It includes the session guarantees [read-your-writes](#read-your-writes) and [monotonic reads](#monotonic-reads). Unlike [linearizability](#linearizability), it can keep serving during a [network partition](#network-partition), provided each client stays with a replica (a node holding a copy of the data) that has the client's history.

Implementations track dependencies, often with a [vector clock](#vector-clock) or explicit lists of the writes each update depends on, and delay applying an update until its dependencies have arrived. Concurrent writes to the same key still conflict, so the system also needs a merge rule.

_Usage:_

"Some users see replies to comments that don't exist yet."

"Our replicas apply writes in arrival order. We need causal consistency, so a reply waits until its parent comment is there."

### Eventual consistency

A consistency model which promises that if no new updates are made to a data item, all replicas ([nodes](#node) holding copies of it) will eventually return the same value. It makes no statement about how long that takes or what a read returns in the meantime.

Formally this is a liveness property: it says something good will happen eventually, and no finite observation can violate it. A read that returns stale data is allowed. So is a read that returns a newer value followed by one that returns an older value. The model also says nothing about which value replicas converge on when writes conflict, so the system needs a rule such as [last-write-wins](#last-write-wins) or a [CRDT](#crdt).

Systems choose it because a replica can answer from local data with no coordination. Reads and writes stay fast and can continue during a [network partition](#network-partition), which is the available side of the [CAP theorem](#cap-theorem).

The cost falls on application code, with symptoms such as:

| Symptom                                          | What happened                                   |
| ------------------------------------------------ | ----------------------------------------------- |
| A saved change disappears on refresh             | The read hit a replica without the write        |
| A like count goes up and down between loads      | Requests hit replicas at different states       |
| A new account "doesn't exist" right after signup | The next service read from a lagging replica    |
| A deleted item reappears                         | A replica missed the delete and spread old data |

Replicas converge through background mechanisms such as [read repair](#read-repair), [anti-entropy](#anti-entropy), [hinted handoff](#hinted-handoff) and a [gossip protocol](#gossip-protocol).

To work with the model, decide which reads can tolerate staleness. Add session guarantees where a user notices, such as [read-your-writes](#read-your-writes) and [monotonic reads](#monotonic-reads), and send the few reads that must be current to a stronger path.

_Usage:_

"QA says the settings page sometimes shows the old value after saving."

"The profile store is eventually consistent and that read hits a replica. Route it to the leader for a user's own settings."

### Read-your-writes

A session guarantee that once a client has written a value, that client's later reads will return that write or a newer one. It was described by Douglas Terry and colleagues in 1994 as one of four session guarantees for weakly consistent systems. It is also called read-after-write consistency.

The symptom is familiar. A user edits their profile, presses save, sees the page reload, and the old value is back. The write went to the leader, the reload read from a replica (a [node](#node) holding a copy of the data) that had not received it yet, and a few seconds later the new value appears.

The guarantee applies per session. Other users may still see the old value for a while, which is acceptable under [eventual consistency](#eventual-consistency). It needs less coordination than [linearizability](#linearizability), since only the writer's own view needs protecting.

Common ways to provide it:

| Approach         | How it works                                                                           | Trade-off                                  |
| ---------------- | -------------------------------------------------------------------------------------- | ------------------------------------------ |
| Read from leader | Send reads of a user's own data to the leader                                          | Adds load to the leader                    |
| Time window      | Route a user to the leader for N seconds after a write                                 | Fails if lag exceeds N                     |
| Version token    | Client keeps its write's log position, and replicas serve the read only once caught up | Requires passing a token through the stack |
| Sticky routing   | Keep the session on the replica that took its writes                                   | Breaks when that replica fails             |

For a user on several devices, store the version token server-side, because a token in one browser's cookie does not cover a write made on a phone.

Seeing another user's update and then losing it on the next read is a separate anomaly, covered by [monotonic reads](#monotonic-reads).

_Usage:_

"Users keep reporting that their display name reverts after saving."

"We're breaking read-your-writes. The profile page reads from a replica. Pin that user to the primary for a short window after any update."

### Monotonic reads

A session guarantee that once a client has read a value, its later reads will return that value or a newer one. It is one of the four session guarantees from Douglas Terry and colleagues in 1994.

The symptom is data that appears and then vanishes. A user refreshes a page and sees a new comment, refreshes again and it has gone, and refreshes a third time and it is back. No data changed. A load balancer sent each request to a different replica (a [node](#node) holding a copy of the data), and those replicas were at different points in replication:

| Refresh | Replica | Replica's lag | New comment visible? |
| ------- | ------- | ------------- | -------------------- |
| 1       | A       | 0 seconds     | Yes                  |
| 2       | B       | 5 seconds     | No                   |
| 3       | A       | 0 seconds     | Yes                  |

Under [eventual consistency](#eventual-consistency) this is permitted, but it looks like a bug to users, and it can break application logic. A worker that reads a job as finished and then reads it as pending may run it again.

There are two common ways to provide the guarantee. The first pins each session to one replica, for example by hashing the user ID to choose it. If that replica fails, the session can go backward. The second has the client remember the highest log position or version it has seen and send it with each read, so a replica that is further behind waits, or the request goes to another replica.

Monotonic reads does not promise fresh data. A client pinned to a replica ten minutes behind satisfies the guarantee as long as its view never moves backward. It also does not imply [read-your-writes](#read-your-writes), because the client's own write may not have reached that replica. [Causal consistency](#causal-consistency) includes each of them.

_Usage:_

"The order shows as shipped, then processing, then shipped again when I refresh."

"That's a monotonic reads violation. The reads are bouncing between replicas with different lag. Let's route each session to one replica."

## Section 4 — Replication and Convergence

### State machine replication

A technique for keeping several copies of a service identical by having each copy apply the same deterministic commands in the same order. If every replica (a [node](#node) holding a copy) starts in the same state and processes the same sequence of inputs, every replica ends in the same state. Leslie Lamport introduced the approach in 1978, and Fred Schneider's 1990 tutorial is a standard reference.

The hard part is agreeing on the sequence, which is a [consensus](#consensus) problem. Protocols such as [Raft](#raft) and Multi-[Paxos](#paxos) maintain a replicated log of commands. A typical leader-based write looks like this:

| Step | Who          | What happens                                        |
| ---- | ------------ | --------------------------------------------------- |
| 1    | Client       | Sends a command to the leader                       |
| 2    | Leader       | Appends it to its log and sends it to followers     |
| 3    | Followers    | Store the entry durably and acknowledge             |
| 4    | Leader       | Marks it committed once a majority has stored it    |
| 5    | Each replica | Applies committed entries to its state in log order |
| 6    | Leader       | Returns the result to the client                    |

With 2f + 1 replicas, this tolerates f [crash failures](#crash-failure).

Commands must be deterministic, and violations cause replicas to diverge silently. If a command calls `now()` or `random()`, or iterates over a hash map whose order varies, each replica computes a different result from the same log. The usual fix is for the leader to resolve such values first and put the concrete result into the log entry.

Two practical details follow. Logs grow without limit, so replicas periodically save a snapshot of their state and discard earlier entries. Clients also retry after timeouts, which can place one command in the log twice, so commands carry a client ID and sequence number that give [idempotency](#idempotency) when applied.

_Usage:_

"Two replicas have different balances, but their logs are identical."

"Then something in the apply path isn't deterministic. State machine replication only works if the same log produces the same state. Check for timestamps."

### Quorum

The minimum number of [nodes](#node) that must take part before an operation counts as successful. Quorums let a system proceed without waiting for every node, while ensuring that any two operations that need to see each other share at least one node.

The common form is a majority, floor(N/2) + 1 of N nodes. Any two majorities overlap, so two conflicting decisions cannot both gather one. This is why [Raft](#raft) and [Paxos](#paxos) use majorities and why a [network partition](#network-partition) can leave at most one side able to make progress.

Replicated stores often tune separate sizes: N replicas (nodes holding copies), W acknowledgments for a write, and R responses for a read. If R + W > N, every read set overlaps every write set. With N = 3:

| W   | R   | Overlap? | Node losses writes survive | Node losses reads survive |
| --- | --- | -------- | -------------------------- | ------------------------- |
| 2   | 2   | Yes      | 1                          | 1                         |
| 3   | 1   | Yes      | 0                          | 2                         |
| 1   | 3   | Yes      | 2                          | 0                         |
| 1   | 1   | No       | 2                          | 2                         |

Overlap alone does not give [linearizability](#linearizability). During an in-progress write, one read can reach a replica with the new value and a later read can miss it, so values appear to go backward. Closing that gap requires readers to write the newest value back to a quorum before returning, and conflicting writes need a version order.

Cluster size determines how many failures a majority can survive. A four-node majority is three, so it tolerates one failure, the same as a three-node cluster. This is why [consensus](#consensus) clusters usually have an odd size. A three-node cluster that has lost two nodes stops accepting writes, because the remaining node cannot tell whether it is on the minority side of a partition.

_Usage:_

"Two of the three etcd nodes are down and all writes are hanging. Can we force it?"

"It has lost quorum, so it's refusing on purpose. Restore a node. Forcing it risks losing committed data."

### Sloppy quorum

A relaxed [quorum](#quorum) in which a write can be acknowledged by any reachable [nodes](#node), including nodes that are not among the key's designated replicas (the nodes assigned to hold its copies). Amazon's Dynamo paper described the technique in 2007. It keeps writes succeeding when too few of the designated replicas can be reached.

| During a failure    | Strict quorum                     | Sloppy quorum                                           |
| ------------------- | --------------------------------- | ------------------------------------------------------- |
| Who can acknowledge | Designated replicas only          | Any W reachable nodes                                   |
| Write availability  | Fails below W designated replicas | Continues while W nodes are reachable                   |
| R + W > N overlap   | Guaranteed                        | Not guaranteed                                          |
| After recovery      | Normal replication resumes        | [Hinted handoff](#hinted-handoff) moves data back |

A sloppy quorum weakens read overlap. Say a key lives on A, B and C, with N = 3, W = 2 and R = 2. During a [network partition](#network-partition), a client can reach only C, D and E. A sloppy write lands on C and D, and the client receives a success. After the partition heals, a read contacts A and B before the handoff has run. Both hold the old value, so the read returns stale data even though R + W > N.

A sloppy quorum is therefore a durability and availability measure, and it provides only [eventual consistency](#eventual-consistency) until repair completes. The data exists on W nodes, but not necessarily on nodes a reader will ask.

Products differ. Riak exposes settings that require acknowledgments from primary replicas. Cassandra stores hints for down replicas but does not count them toward its consistency levels, except the weakest level, `ANY`. Check how your store behaves before relying on quorum settings for correctness.

_Usage:_

"We write and read at quorum, so how did this read come back stale?"

"Sloppy quorum. During the outage the write landed on fallback nodes, and the read hit replicas that hadn't received the handoff yet."

### Hinted handoff

A repair technique in which a reachable [node](#node) temporarily stores a write meant for an unreachable replica (a node assigned to hold a copy of the data), along with a hint naming the intended owner. When the owner comes back, the stored write is delivered to it. Leaderless stores such as Dynamo and Cassandra use this technique, and it is what moves data back after a [sloppy quorum](#sloppy-quorum).

| Step | Who         | What happens                                                                                                         |
| ---- | ----------- | -------------------------------------------------------------------------------------------------------------------- |
| 1    | Coordinator | Sends a write and finds replica C unreachable                                                                        |
| 2    | Hint holder | Stores the write with a hint saying it belongs to C                                                                  |
| 3    | Hint holder | Learns C is back, through a [failure detector](#failure-detector) or [gossip protocol](#gossip-protocol) |
| 4    | Hint holder | Replays the write to C                                                                                               |
| 5    | Hint holder | Deletes the hint once C acknowledges                                                                                 |

The hint holder varies by product. In the Dynamo design it is a fallback node, and in Cassandra it is normally the coordinator (the node handling the client's request).

Hinted handoff shortens the time a returning replica is out of date, but it is a best-effort mechanism. A hint holder can fail before delivering, which loses the hint. Stores also stop collecting hints after a configurable window so that a long outage does not fill their disks. The recognizable failure is a node that returns after extended maintenance, reports healthy, and serves stale or missing rows, because the outage outlasted the hint window. Any outage longer than that window needs a full [anti-entropy](#anti-entropy) repair before the node can be trusted.

Replay also creates load. When a node returns, several peers may send it their stored hints at once, on top of normal traffic, which can slow it enough to look failed again.

_Usage:_

"Node six was offline for two days. It's back, so we're fine, right?"

"Hinted handoff only covers the first few hours of that. Run a full repair before it serves [quorum](#quorum) reads."

### Read repair

A technique that fixes out-of-date replicas (the [nodes](#node) holding copies of a key) as a side effect of reading. When a read contacts several replicas and their versions disagree, the coordinator (the node handling the request) sends the newest version to the replicas that returned an older one.

| Step | Who         | What happens                                         |
| ---- | ----------- | ---------------------------------------------------- |
| 1    | Coordinator | Sends the read to R replicas                         |
| 2    | Replicas    | Return their value, or a digest (a short hash) of it |
| 3    | Coordinator | Compares versions and identifies the newest          |
| 4    | Coordinator | Writes the newest value to the stale replicas        |
| 5    | Coordinator | Returns the newest value to the client               |

Steps 4 and 5 can happen in either order, and the choice changes the guarantee. In blocking read repair, the coordinator waits until a write [quorum](#quorum) holds the value before responding, so a later quorum read cannot return the older version. In asynchronous read repair, the client gets a faster response, but a following read can still see the old value.

Read repair produces a recognizable pattern in debugging. A read from one replica returns an old value, the next attempt returns the correct one, and by the time someone investigates, all replicas agree. The first read triggered the fix and removed the evidence.

The technique only repairs keys that are read. Rarely read data can stay inconsistent indefinitely, so it is paired with background [anti-entropy](#anti-entropy).

Read repair is only as correct as its version comparison. With [last-write-wins](#last-write-wins), a write carrying a timestamp inflated by [clock skew](#clock-skew) is treated as newest and copied over a genuinely newer value. Stores that detect concurrent versions, for example with a [vector clock](#vector-clock), must merge them or keep both.

_Usage:_

"The stale row fixed itself before I could reproduce it."

"Probably read repair. Your first query found the mismatch and updated the lagging replica. Check the repair metrics for that table."

### Anti-entropy

A background process that compares replicas (the [nodes](#node) holding copies of the same data) and copies over whatever is missing or out of date. The name refers to reducing disorder between copies. It was described in work on replicated databases at Xerox PARC by Demers and colleagues in 1987.

Anti-entropy is the backstop among the convergence mechanisms in an [eventually consistent](#eventual-consistency) store, since it covers data the others miss:

| Mechanism                               | When it runs               | What it covers               | Gap                        |
| --------------------------------------- | -------------------------- | ---------------------------- | -------------------------- |
| [Hinted handoff](#hinted-handoff) | When a replica returns     | Writes during a short outage | Lost hints, long outages   |
| [Read repair](#read-repair)       | During reads               | Keys that are read           | Data nobody reads          |
| Anti-entropy                            | On a schedule or on demand | All data in scope            | Uses disk, CPU and network |

Comparing full datasets over the network would be too expensive, so implementations exchange summaries. A [Merkle tree](#merkle-tree) lets two replicas locate differing key ranges by exchanging a small number of hashes, then transfer only those ranges. Exchanges can push data, pull data, or do both.

Repair frequency has a correctness consequence around deletes. Replicated stores usually record a delete as a tombstone (a marker saying the key was deleted) and purge it after a retention period. If a replica misses the delete and repair does not run before the tombstone is purged, that replica still holds the old row and no marker exists to override it. The next repair copies the row back to the others, and deleted data reappears. In Cassandra, for this reason, repair must complete within the `gc_grace_seconds` window.

Repairs are heavy operations, so schedule them with limits and stagger them across nodes. Run one after any outage longer than the hint window, and alert when a scheduled repair has not completed.

_Usage:_

"Customers deleted those records last month, and now they're back."

"Sounds like tombstones expired before anti-entropy reached that replica. When did repair last finish on this cluster?"

### Merkle tree

A tree of hashes in which each leaf is the hash of a block of data and each parent is the hash of its children. The root therefore summarizes the entire dataset, and any change to a block changes every hash on the path up to the root. Ralph Merkle devised the structure around 1979.

In a [distributed system](#distributed-system), Merkle trees answer a specific question: which parts of two large datasets differ? Sending all the data, or even all the keys, between replicas ([nodes](#node) holding copies) would consume a large amount of bandwidth just to confirm they mostly match. Two nodes can instead compare trees from the top down:

| Step | What happens                                                 |
| ---- | ------------------------------------------------------------ |
| 1    | Each node builds a tree over the same key ranges             |
| 2    | The nodes compare root hashes. If equal, the ranges match    |
| 3    | If different, they compare the hashes of the root's children |
| 4    | They descend only into subtrees whose hashes differ          |
| 5    | At the leaves, they exchange the data for mismatched ranges  |

If only one leaf differs, the comparison follows a single path, so the number of levels visited grows with the logarithm of the number of leaves. This is why [anti-entropy](#anti-entropy) in stores such as Dynamo and Cassandra uses these trees. Git and blockchains use the same structure or close variants.

Matching hashes indicate matching data with overwhelming probability, not certainty.

Tree granularity is a practical trade-off. If each leaf covers many keys, the tree stays small, but one differing key forces the whole range to be transferred, a problem called overstreaming. It shows up as a repair moving more data than differed. More leaves reduce that waste and use more memory.

_Usage:_

"Repair streamed 200 GB, but only a few thousand rows were out of sync."

"The Merkle tree leaves are too coarse, so one bad key resends the whole range. Try increasing the tree depth on that table."

### Gossip protocol

A communication method in which each [node](#node) periodically exchanges information with a few randomly chosen peers. Those peers pass it on in later rounds, so an update spreads through the cluster the way an infection spreads through a population. These are also called epidemic protocols.

Each round, a node picks one or a few peers and does one of the following:

| Style     | What is exchanged                               |
| --------- | ----------------------------------------------- |
| Push      | The node sends the peer what it knows           |
| Pull      | The node asks the peer for what it is missing   |
| Push-pull | Both sides send each other what the other lacks |

Because the number of informed nodes grows roughly exponentially at first, an update reaches all N nodes in about O(log N) rounds with high probability. Each node sends a fixed amount per round, and there is no central coordinator. Random peer selection also routes around failed nodes and many [network partitions](#network-partition) that are not complete.

Gossip commonly carries cluster membership, node health and metadata such as which node owns which data. Cassandra and Consul are examples. It often feeds a [failure detector](#failure-detector), since peers can report on nodes they have heard from recently.

The trade-off is that delivery is probabilistic and delayed, giving [eventual consistency](#eventual-consistency) for the gossiped state. During a rollout, different nodes can briefly hold different views of the cluster, so one routes a request to a member that another already considers gone.

Removals need particular care. A node that was removed can reappear because a peer still holds its old entry and gossips it back. Protocols prevent this by attaching a generation or version number to each entry, and by keeping a removal marker long enough for it to spread everywhere.

_Usage:_

"We decommissioned that node an hour ago, but it keeps showing up in the member list."

"A peer's still gossiping its old state. Check that the removal marker has propagated before restarting anything."

### Last-write-wins

A conflict resolution rule that keeps the write with the highest timestamp and discards the others. When replicas (the [nodes](#node) holding copies of a key) hold different versions, each compares timestamps and retains the winner, with a tie-breaker such as node ID. Every replica applies the same rule, so all converge on one value. This gives [eventual consistency](#eventual-consistency) with little machinery, and it is the default in stores such as Cassandra.

The rule's weakness is that "last" means highest timestamp, which may not be the most recent write. With [clock skew](#clock-skew), an older write can beat a newer one:

| Real order           | Node | Node's clock | Write            |
| -------------------- | ---- | ------------ | ---------------- |
| First                | A    | 10:00:00.300 | `status = "old"` |
| Second, 100 ms later | B    | 10:00:00.100 | `status = "new"` |

Node A's clock runs 300 ms ahead, so its earlier write has the higher timestamp. The store keeps `"old"`, and the write of `"new"` disappears even though its client received a success.

The rule also loses data when clocks are perfect. If two users edit a record concurrently, in the [happens-before](#happens-before) sense, then neither saw the other's change, and one edit is dropped with no error. Whole-record writes make it worse: two users changing different fields overwrite each other's work.

Several measures reduce the risk:

- Store each field separately, so edits to different fields do not conflict.
- Use immutable records with unique keys, so no two writes target the same key.
- Use compare-and-set (write only if the value is unchanged) where a lost update is unacceptable.
- Use a [hybrid logical clock](#hybrid-logical-clock), so a causally later write gets a higher timestamp.
- Use a [CRDT](#crdt) or a [vector clock](#vector-clock) when concurrent updates must be preserved.

_Usage:_

"Support says two agents updated the ticket, and one note is gone."

"The store uses last-write-wins on the whole row, so the second save replaced the first. We should move notes into their own records."

### CRDT

Conflict-free replicated data type: a data structure whose copies can be updated independently, with no coordination, and that converge when they exchange updates. Marc Shapiro and colleagues formalized the idea in 2011. The guarantee is called strong [eventual consistency](#eventual-consistency): any two replicas (the [nodes](#node) holding copies) that have received the same set of updates are in the same state.

State-based CRDTs send their state and merge it with a commutative, associative and idempotent function, so messages can repeat or arrive in any order, which suits [at-least-once delivery](#at-least-once-delivery). Operation-based CRDTs send individual operations that commute when concurrent, and rely on a delivery layer that delivers each one once and, for many types, in causal order.

| Type         | Supports             | How it merges                                                                                     |
| ------------ | -------------------- | ------------------------------------------------------------------------------------------------- |
| G-Counter    | Increment            | One count per node, merged by taking each maximum                                                 |
| PN-Counter   | Increment, decrement | Two G-Counters, value is increments minus decrements                                              |
| OR-Set       | Add, remove          | Each add gets a unique tag, and remove deletes only the tags it saw, so a concurrent add survives |
| LWW-Register | Set a value          | Highest timestamp wins, as in [last-write-wins](#last-write-wins)                             |

CRDTs fit collaborative editing and offline-first apps, where a user keeps working during a [network partition](#network-partition) and merges later. Automerge and Yjs are examples of libraries.

Convergence does not mean the merged result is what users wanted. Two people editing one sentence can produce an interleaved result that is valid but unreadable. A plain CRDT also cannot enforce global invariants, such as "balance never goes below zero" or "usernames are unique", because each replica accepts updates before hearing from the others. Those checks need [consensus](#consensus) or quotas allocated to each replica in advance.

_Usage:_

"Can we make the inventory count a CRDT so warehouses can work offline?"

"The count would converge, but two warehouses could both sell the last unit. A plain counter CRDT can't enforce stock never going negative."

## Section 5 — Consensus and Coordination

### Consensus

The problem of getting a group of [nodes](#node) to agree on a single value, even when some of them fail. Once the value is decided, it stays decided. A standard formal definition has three properties:

| Property    | Requirement                                  | Type     |
| ----------- | -------------------------------------------- | -------- |
| Agreement   | No two correct nodes decide different values | Safety   |
| Validity    | The decided value was proposed by some node  | Safety   |
| Termination | Every correct node eventually decides        | Liveness |

A safety property says nothing bad ever happens, and a liveness property says something good eventually happens. [FLP impossibility](#flp-impossibility) shows that a deterministic protocol cannot guarantee both in a fully asynchronous network with even one crash. Practical protocols such as [Paxos](#paxos) and [Raft](#raft) therefore always preserve safety and rely on timing assumptions only for progress.

Many coordination tasks reduce to consensus: [leader election](#leader-election), agreeing on cluster membership, deciding a transaction's outcome, and ordering commands in [state machine replication](#state-machine-replication). Repeating consensus for each slot of a log yields total order broadcast (delivering the same messages to all nodes in the same order), and the two problems are equivalent in the [crash failure](#crash-failure) model.

Tolerating f crash failures takes 2f + 1 nodes, because each decision needs a majority [quorum](#quorum) and any two majorities overlap.

When a three-node etcd cluster loses two members, the survivor stops accepting writes even though it is running. It cannot tell a crash from a [network partition](#network-partition), and accepting writes could create a conflicting history. The protocol stops making progress so that it never disagrees with itself.

Each decision costs network round trips and durable disk writes, and adding voters adds to that cost. Systems therefore keep the voting group small and use it for metadata and coordination, with bulk data stored elsewhere.

_Usage:_

"Why not run a two-node etcd cluster? It's cheaper than three."

"A majority of two is two, so losing either node stops consensus. You'd pay for a second machine and tolerate zero failures."

### FLP impossibility

A proof that no deterministic algorithm can guarantee [consensus](#consensus) in a fully asynchronous system if even one process may crash. It is named after Michael Fischer, Nancy Lynch and Michael Paterson, who published it in 1985.

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
| Partial synchrony                           | The network eventually behaves within time bounds                   | [Paxos](#paxos), [Raft](#raft) |
| Randomization                               | [Nodes](#node) use coin flips, and terminate with probability 1 | Ben-Or's 1983 algorithm                |
| [Failure detector](#failure-detector) | An oracle that is eventually accurate enough                        | Chandra-Toueg algorithm                |

The practical consequence appears in how Raft and Paxos fail. Under bad timing, such as repeated [leader election](#leader-election) rounds on an unstable network, a cluster may stop making progress for as long as the instability lasts. It will not return conflicting decisions.

_Usage:_

"Can we prove our election always finishes within some time limit?"

"Not on an asynchronous network. FLP rules out guaranteed termination. We can prove it never elects two leaders per term, and use randomized timeouts so it converges in practice."

### Paxos

A [consensus](#consensus) protocol devised by Leslie Lamport and published in 1998. It lets a group of [nodes](#node) choose one value while tolerating [crash failures](#crash-failure) of a minority, message loss and message delay.

Paxos defines three roles, and one server often plays all of them:

- Proposers suggest values.
- Acceptors vote on proposals.
- Learners find out which value was chosen.

Single-decree Paxos decides one value in two phases:

| Phase       | From → to            | What happens                                                                                                                                            |
| ----------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1a Prepare  | Proposer → acceptors | Proposer picks a unique number `n` and sends `prepare(n)`                                                                                               |
| 1b Promise  | Acceptor → proposer  | If `n` is above any prepare it has answered, the acceptor promises to ignore lower numbers and returns the highest-numbered proposal it has accepted    |
| 2a Accept   | Proposer → acceptors | With promises from a majority, proposer sends `accept(n, v)`. `v` must be the value of the highest-numbered proposal returned, or its own value if none |
| 2b Accepted | Acceptor → learners  | Acceptor accepts unless it has promised a number above `n`                                                                                              |

A value is chosen once a majority of acceptors have accepted the same proposal. The rule in step 2a provides safety: any later proposer must learn of a possibly chosen value through the overlapping [quorum](#quorum) and propose it again. Acceptors must persist promises and acceptances to disk before replying, or a restart could break those promises.

Two proposers can block each other indefinitely by issuing ever-higher numbers, each invalidating the other's phase 1. A distinguished leader or randomized backoff makes this livelock unlikely.

Real systems need a sequence of decisions for [state machine replication](#state-machine-replication). Multi-Paxos runs one instance per log slot, and a stable leader can skip phase 1 for later slots. [Raft](#raft) was designed partly because the original papers leave these engineering details open.

_Usage:_

"Throughput dropped to nothing and both servers are busy sending prepares."

"Dueling proposers, preempting each other in phase 1. Add a stable leader or randomized backoff."

### Raft

A [consensus](#consensus) protocol for maintaining a replicated log, published by Diego Ongaro and John Ousterhout in 2014 with understandability as an explicit goal. It provides fault tolerance comparable to Multi-[Paxos](#paxos), and etcd and Consul use it.

Each [node](#node) is in one of three roles:

| Role      | What it does                                                                     |
| --------- | -------------------------------------------------------------------------------- |
| Follower  | Responds to requests, and starts an election if it hears no leader               |
| Candidate | Increments the term, votes for itself and requests votes                         |
| Leader    | Handles client writes, replicates the log and sends [heartbeats](#heartbeat) |

Time is divided into numbered terms, and each term has at most one leader. A follower whose randomized election timeout expires becomes a candidate. It wins with votes from a majority [quorum](#quorum), and each node casts at most one vote per term, recorded on disk. A voter refuses any candidate whose log is less up to date than its own, so a new leader always holds every committed entry.

The leader appends each command to its log and sends it to followers with AppendEntries messages, which include the index and term of the preceding entry so followers can reject mismatches. An entry from the current term is committed once a majority stores it. Entries from earlier terms become committed only indirectly, when a later entry from the current term commits. Committed entries are applied in order, as in [state machine replication](#state-machine-replication).

A common operational problem is election churn. If disk or network latency delays heartbeats past the election timeout, followers keep starting elections, and writes stall even though every node is up. Raising the timeout or moving the log to a faster disk usually helps.

A deposed leader may not know it has been replaced, so linearizable reads must confirm leadership first.

_Usage:_

"etcd keeps logging leader changes, but all three members are up."

"Check disk sync latency. If the disk stalls, Raft heartbeats miss the election timeout and followers call new elections."

### Leader election

The process by which a group of [nodes](#node) chooses one member to coordinate work, and chooses a replacement when that member fails. A single leader can order writes, assign tasks or run a scheduled job without negotiating each decision.

An election protocol has two goals. Safety means at most one leader is recognized for a given term (a numbered period of leadership). Liveness means that a leader is eventually chosen as long as enough nodes are working. [FLP impossibility](#flp-impossibility) means liveness cannot be guaranteed on a fully asynchronous network, so protocols depend on timeouts.

| Approach                         | How a leader is chosen                                                        | Safe under partition?            |
| -------------------------------- | ----------------------------------------------------------------------------- | -------------------------------- |
| [Consensus](#consensus) vote | A majority votes in a numbered term, as in [Raft](#raft)                  | Yes, one leader per term         |
| Coordination service             | Nodes compete for a [lease](#lease) in a system such as etcd or ZooKeeper | Yes, if stale leaders are fenced |
| Bully algorithm                  | The highest-ID reachable node wins                                            | No, each side can elect its own  |
| Manual or static                 | An operator or config file names the leader                                   | Depends on the operator          |

A correct election does not ensure that only one node believes it is leader at every moment. A leader that pauses or loses contact can continue acting on an old belief while a new leader is elected. The guarantee is per term, and that gap is how [split brain](#split-brain) reaches external systems. When a leader writes to shared storage, protect the write with a [fencing token](#fencing-token), such as the term number.

Elections also cost availability. Between the old leader failing and a new one taking over, writes fail or hang, typically for the detection timeout plus the election itself. Timeouts set too aggressively cause repeated elections under load, when a [failure detector](#failure-detector) mistakes slowness for failure.

_Usage:_

"The cron job ran twice last night, once on each scheduler."

"The old scheduler was paused during the handover and woke up still thinking it led. The job needs a fencing check."

### Lease

A time-limited grant of a right, such as being leader or holding a lock, that expires automatically unless the holder renews it. Cary Gray and David Cheriton described leases in 1989 for file cache consistency. If a [node](#node) holding an ordinary lock crashes, the lock may never be released. A lease ends on its own.

| Step | Who     | What happens                                                      |
| ---- | ------- | ----------------------------------------------------------------- |
| 1    | Holder  | Records its local time, then requests a lease of duration D       |
| 2    | Grantor | Grants the lease and starts timing D                              |
| 3    | Holder  | Works, and renews well before D runs out                          |
| 4    | Holder  | Stops work if it cannot renew by its local expiry                 |
| 5    | Grantor | After D plus a safety margin, may grant the lease to another node |

The holder starts counting before sending its request, so its expiry falls earlier than the grantor's. This keeps the two views safe without synchronized wall clocks, provided [clock drift](#clock-drift) stays below an assumed bound. Durations should use a monotonic clock (a timer that only moves forward).

Lease duration is a trade-off. A short lease recovers faster after a crash but needs more renewal traffic and is easier to lose in a brief stall.

A paused holder can outlive its lease. A process checks its lease, sees it is valid, and is then paused by garbage collection (the runtime halting the program to reclaim memory) or a VM migration. The lease expires, another node acquires it, and the first process wakes and carries on with its write. No check inside the process can close this gap, since the pause can land right after the check. Protecting the resource takes a [fencing token](#fencing-token).

_Usage:_

"Worker A's lease expired during a long GC pause, and it still wrote to the bucket."

"A lease can't stop that by itself. The storage layer has to reject stale holders, so let's add a fencing token."

### Distributed lock

A lock that coordinates processes on different machines, so that only one of them works on a resource at a time. Because a holder can crash without releasing it, a distributed lock is almost always implemented as a [lease](#lease) that expires. That expiry is the source of its main weakness: a holder can lose the lock without knowing.

The typical incident looks like this. Worker A acquires the lock and stalls, perhaps in a garbage collection pause (the runtime halting to reclaim memory). Its lease expires. Worker B acquires the lock and starts writing. Worker A resumes, still believing it holds the lock, and also writes. The same job has run twice or a file is corrupted, even though the locking code contains no bug.

The right lock depends on its purpose:

| Purpose     | If two holders overlap             | Suitable approach                                                                                            |
| ----------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Efficiency  | Some work is duplicated            | A simple lock, such as Redis `SET key value NX PX ttl`                                                       |
| Correctness | Data is corrupted or money is lost | A [consensus](#consensus)-backed lock in etcd or ZooKeeper, plus a [fencing token](#fencing-token) |

A lock on a single [node](#node) is unavailable or lost if that node fails, and failover with asynchronous replication can grant the same lock twice. A consensus-backed service avoids that but cannot stop a paused client. Only the protected resource can, by rejecting stale requests.

Give each acquisition a unique value and release only if the stored value still matches, or a slow worker can delete a lock now held by someone else. Where possible, avoid the lock altogether by using [idempotency](#idempotency) or a compare-and-set write (a write that succeeds only if the value is unchanged), which puts the check at the resource.

_Usage:_

"We put a Redis lock around the invoice job and still got duplicates."

"A lock only narrows that window. If the worker pauses past the expiry, two can run. Make the invoice insert idempotent."

### Fencing token

A number that increases with every grant of a [lease](#lease) or lock, which the holder attaches to each request to the protected resource. The resource remembers the highest token it has seen and rejects anything lower. This stops a former holder that does not yet know it has been replaced.

Fencing addresses the gap that a [distributed lock](#distributed-lock) cannot close on its own, because a client can pause at any moment and resume after its lease has expired:

| Step | Actor    | Action                                  | Storage's highest token |
| ---- | -------- | --------------------------------------- | ----------------------- |
| 1    | Client 1 | Acquires lock, receives token 33        | 32                      |
| 2    | Client 1 | Pauses, and its lease expires           | 32                      |
| 3    | Client 2 | Acquires lock, receives token 34        | 32                      |
| 4    | Client 2 | Writes with token 34, accepted          | 34                      |
| 5    | Client 1 | Resumes, writes with token 33, rejected | 34                      |

Without step 5's check, Client 1 would overwrite Client 2's work, and neither client would see an error.

Two requirements make this work. The tokens must come from a source that never gives one number to two holders or goes backward, which in practice means a system built on [consensus](#consensus). A [Raft](#raft) term, an etcd revision and a ZooKeeper transaction ID (zxid) are common sources. The resource must also perform the comparison and the write atomically, because a separate check followed by a write reopens the race.

The second requirement is the practical limit. Fencing needs the resource to participate, and many do not. A database can do it with a conditional update such as `UPDATE ... WHERE fence_token <= 34`. A third-party API usually cannot, so for those calls use an [idempotency](#idempotency) key (a unique ID that lets the receiver discard repeats) and accept that the protection is weaker.

_Usage:_

"The old leader came back after the pause and overwrote the new leader's output."

"Nothing was checking who was current. Pass the term as a fencing token and have storage reject anything lower than it's seen."

## Section 6 — Transactions and Delivery

### Two-phase commit

A protocol that makes a transaction across several [nodes](#node) either commit on all of them or abort on all of them. One node acts as coordinator and the others are participants. It is often shortened to 2PC, and differs from two-phase locking, a technique for isolating concurrent transactions.

| Step | Who         | What happens                                                                       |
| ---- | ----------- | ---------------------------------------------------------------------------------- |
| 1    | Coordinator | Sends prepare to all participants                                                  |
| 2    | Participant | Durably records its changes and vote, keeps its locks, and replies yes or no       |
| 3    | Coordinator | Decides commit if all voted yes, otherwise abort, and records the decision durably |
| 4    | Coordinator | Sends the decision to all participants                                             |
| 5    | Participant | Applies the decision, releases locks and acknowledges                              |

A yes vote is binding. The participant must be able to commit even after a crash and restart, and it may not decide alone.

That promise creates the protocol's known weakness. If the coordinator fails after participants vote yes but before they hear the decision, they are in doubt. They cannot commit, because another participant may have voted no. They cannot abort, because the coordinator may have decided to commit. So they wait, holding locks, and other transactions touching those rows queue behind them. The symptom is a table that appears frozen with no active query to blame, until the coordinator recovers or an operator intervenes.

2PC is not a [consensus](#consensus) protocol in the fault-tolerant sense. It requires every participant to respond, so one failure can stop progress, where consensus needs only a majority. A common remedy combines the two: each participant and the coordinator are replicated with [Paxos](#paxos) or [Raft](#raft), so that a single machine failure cannot leave a transaction stuck. Spanner uses this design. [Three-phase commit](#three-phase-commit) tried a different remedy.

_Usage:_

"This table has been locked for an hour, but nothing's running."

"Check pg_prepared_xacts. It's likely an in-doubt two-phase commit whose coordinator died. Get the outcome from the coordinator's log before resolving it."

### Three-phase commit

A variant of [two-phase commit](#two-phase-commit) that adds a phase to avoid blocking when the coordinator crashes. Dale Skeen proposed it in the early 1980s. It is often shortened to 3PC.

Two-phase commit blocks because a participant that voted yes cannot tell whether anyone has already committed. Three-phase commit removes that uncertainty by inserting a step between voting and committing:

| Phase        | Coordinator                                      | Participants                                |
| ------------ | ------------------------------------------------ | ------------------------------------------- |
| 1. CanCommit | Asks whether each can commit                     | Vote yes or no                              |
| 2. PreCommit | If all voted yes, announces the intent to commit | Record the pre-commit state and acknowledge |
| 3. DoCommit  | After all acknowledge, tells them to commit      | Commit and acknowledge                      |

The added phase creates a useful invariant: no participant commits until every participant has voted yes and has been told so. If the coordinator crashes, the surviving participants can elect a new one and inspect their states. If any of them reached pre-commit, every participant voted yes, so they can finish the commit. If none did, nobody can have committed, so they can safely abort.

That recovery requires [crash failures](#crash-failure) only, bounded message delays, and a reliable [failure detector](#failure-detector), so that a slow [node](#node) is never mistaken for a dead one.

Under a [network partition](#network-partition), the protocol can break atomicity, its main guarantee. Suppose participants that received pre-commit end up on one side and the rest on the other. The first group commits, the second sees no pre-commit and aborts, and the transaction ends up committed on some nodes and aborted on others. Two-phase commit would have blocked here and preserved atomicity.

Production systems that need non-blocking commit usually run two-phase commit over groups replicated with [consensus](#consensus), using [Paxos](#paxos) or [Raft](#raft).

_Usage:_

"Couldn't we switch to three-phase commit so a dead coordinator stops blocking us?"

"Only if the network never partitions. Otherwise each side can decide differently. Replicating the coordinator with Raft is safer."

### At-most-once delivery

A delivery guarantee in which each message is processed zero or one times. A message is never handled twice, but it can be lost. The sender makes one attempt, or the receiver records progress before doing the work, so that a failure leaves the message unprocessed and nothing tries again.

The guarantee is a consequence of the same ambiguity behind every [partial failure](#partial-failure). If a sender gets no acknowledgment, it cannot know whether the message arrived. It must choose between resending, which risks a duplicate, and not resending, which risks a loss. At-most-once chooses not to resend. [At-least-once delivery](#at-least-once-delivery) makes the opposite choice.

In a message queue, the order of two consumer steps determines which guarantee you get:

| Step | At-most-once consumer | If it crashes right after this step        |
| ---- | --------------------- | ------------------------------------------ |
| 1    | Receives the message  | Message is redelivered, nothing lost       |
| 2    | Commits its position  | Message is lost, because it counts as done |
| 3    | Processes the message | Work is complete                           |

The symptom is a gap: a queue dashboard shows everything consumed and no errors, yet some records never reached the destination, often clustered around a deployment or crash. Auto-commit settings in consumer libraries can produce this behavior without anyone having chosen it, since the position may be committed on a timer while work is still in progress.

At-most-once suits data where a small loss costs less than a duplicate or a retry: high-volume metrics, debug logs, telemetry samples, or live position updates that the next message replaces anyway.

It does not suit payments, orders, or anything that must eventually happen. If a loss is unacceptable, use at-least-once delivery with [idempotency](#idempotency), which together produce [exactly-once semantics](#exactly-once-semantics).

_Usage:_

"We lost about forty events during the last deploy, but the consumer shows no errors."

"It auto-commits offsets before processing finishes, so that's at-most-once delivery. Commit after the work is done."

### At-least-once delivery

A delivery guarantee in which a message is retried until the receiver acknowledges it. As long as the sender keeps running or has stored the message durably, and the receiver eventually recovers, the message is not lost. It may be processed more than once, which is the trade-off against [at-most-once delivery](#at-most-once-delivery).

Duplicates are an inherent part of this guarantee, because of [partial failure](#partial-failure). When an acknowledgment is lost, the sender cannot distinguish that case from the message itself being lost, so it sends again:

| Step | Who      | What happens                                                            |
| ---- | -------- | ----------------------------------------------------------------------- |
| 1    | Sender   | Stores the message and sends it                                         |
| 2    | Receiver | Processes the message                                                   |
| 3    | Receiver | Sends an acknowledgment, which is lost or never sent because of a crash |
| 4    | Sender   | Times out with no acknowledgment                                        |
| 5    | Sender   | Resends the message                                                     |
| 6    | Receiver | Processes the same message a second time                                |

The result is a recognizable class of bug: a customer gets two confirmation emails, an account is charged twice, or a counter runs higher than the true value. These often appear in clusters after a deployment or a broker failover, when many in-flight messages are redelivered together.

The fix belongs in the receiver. Make processing safe to repeat through [idempotency](#idempotency), typically by recording each message ID in the same database transaction as the work. This combination is how systems usually achieve [exactly-once semantics](#exactly-once-semantics).

Retries introduce two other problems. A malformed message that always fails will be redelivered forever and can block a queue, so after a limited number of attempts it should move to a dead-letter queue (a holding area for messages needing inspection). Retries can also reorder messages, so consumers that depend on order must check a version or sequence number.

_Usage:_

"A few customers got the shipping email twice after last night's broker restart."

"The queue is at-least-once, so redelivery is expected. The email worker needs to dedupe on message ID."

### Exactly-once semantics

A processing guarantee in which the effect of each message is applied exactly once, even if the message is transmitted several times. Over an unreliable network, no protocol can ensure that a message is physically delivered exactly once, because a sender that gets no acknowledgment cannot tell whether the message or the reply was lost. What systems can provide is an outcome equivalent to one delivery.

The standard construction is [at-least-once delivery](#at-least-once-delivery) plus a mechanism that makes repeats harmless. Retries ensure nothing is lost, and the receiver ensures duplicates have no further effect:

| Technique                       | How it works                                                              | Boundary of the guarantee        |
| ------------------------------- | ------------------------------------------------------------------------- | -------------------------------- |
| [Idempotency](#idempotency) | The receiver records message IDs and ignores repeats                      | Wherever the IDs are stored      |
| Atomic position and result      | Result and consumer position commit in one database transaction           | That one database                |
| Broker transactions             | Kafka's transactions commit output messages and consumer offsets together | Topics within that Kafka cluster |

Every technique has a boundary, and bugs often come from crossing it. A stream processor may advertise exactly-once, and within its own state and topics that claim holds. If a handler also sends an email or calls a payment API, that side effect sits outside the transaction. When the handler crashes and is re-run, the email goes out again.

For an external call, pass an idempotency key that the third party honors. If it offers none, you must choose between possibly repeating the action and possibly missing it, as with any [partial failure](#partial-failure).

Deduplication has limits. IDs are kept for a retention window, so a message redelivered after it is processed again. The ID check and the effect must also commit atomically, or a crash between them produces a duplicate or a loss.

_Usage:_

"Kafka is set to exactly-once, so why did the webhook fire twice?"

"That guarantee stops at Kafka. The webhook is an external side effect, so it needs its own idempotency key."

### Idempotency

The property that performing an operation several times has the same effect as performing it once. In a [distributed system](#distributed-system), it makes retries safe. A timed-out request may or may not have been applied, as in any [partial failure](#partial-failure), and the client can resend it without knowing which.

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

Idempotent does not mean safe to replay at any time. If `SET x = 1` is delayed and retried after another client has run `SET x = 2`, the retry undoes the newer write. Guarding against that takes a version check or a [fencing token](#fencing-token).

Idempotency is the receiver's half of [exactly-once semantics](#exactly-once-semantics), paired with [at-least-once delivery](#at-least-once-delivery).

_Usage:_

"The mobile client retries on timeout, and we're seeing duplicate orders."

"The create endpoint isn't idempotent. Have the client send a key and put a unique constraint on it."
