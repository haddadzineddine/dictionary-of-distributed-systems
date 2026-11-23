---
description: A failure in which a node stops executing, as opposed to continuing to run and behaving incorrectly.
---

A failure in which a [node](./Node.md) stops executing and sends no further messages. Before it stops, the node follows its protocol correctly. This is the failure assumption behind many replication and agreement algorithms, because a stopped node cannot send false information.

Formal work separates failures into models, which define what a faulty node is allowed to do:

| Model                                 | What a faulty node may do                                      |
| ------------------------------------- | -------------------------------------------------------------- |
| Crash-stop                            | Halts permanently                                              |
| Crash-recovery                        | Halts, then may restart with whatever it saved to disk         |
| Omission                              | Keeps running but fails to send or receive some messages       |
| [Byzantine](./Byzantine%20failure.md) | Does anything, including sending false or conflicting messages |

Production systems mostly face crash-recovery, since machines reboot and processes restart. [Paxos](./Paxos.md) and [Raft](./Raft.md) are designed for this model. Their safety depends on a restarted node remembering what it wrote to stable storage, such as the votes it cast and the log entries it acknowledged.

That dependency explains a common data-loss bug. A node acknowledges a write while the data is still in the operating system's buffer, the machine loses power, and the write is gone after the restart even though the client was told it succeeded. Durable systems call fsync (an operation that forces buffered data to disk) before acknowledging, and accept the added latency.

Other nodes cannot observe a crash directly. They see only silence, which could also come from a slow node or a [network partition](./Network%20partition.md). A [failure detector](./Failure%20detector.md) can suspect a crash but, in an asynchronous network, cannot confirm one. A node that was declared dead may still be running, so recovery logic has to allow for it to return.

_Usage:_

"The replica acknowledged the write, then the host rebooted and the row was gone."

"So it acked before syncing to disk. In a crash-recovery model that promise means nothing unless it survives the restart."
