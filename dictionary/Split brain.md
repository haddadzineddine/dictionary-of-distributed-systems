---
description: Two parts of a cluster each act as the authority at the same time, accepting writes that conflict.
---

A state in which two parts of a cluster each act as the authority at the same time. Usually this means two [nodes](./Node.md) both believe they are the leader and both accept writes. The histories then diverge, and when the cluster reunites there may be no correct way to combine them.

The usual trigger is a [network partition](./Network%20partition.md): each side stops hearing from the other, assumes it has died, and promotes or keeps its own leader. A pause can do the same without any network fault. A leader stalls during a long garbage collection (the runtime halting the program to reclaim memory), the others elect a replacement, and the old leader wakes up still believing it is in charge.

The damage appears as conflicting records. Two customers hold the same reserved seat, or an account balance reflects only half of its transactions. Repair often means choosing one history and manually salvaging the other.

Systems layer several defences:

| Defence                               | How it helps                                                     |
| ------------------------------------- | ---------------------------------------------------------------- |
| Majority [quorum](./Quorum.md)        | At most one side of a partition can hold a majority              |
| Term or epoch numbers                 | A newer leader's number outranks the old one, so peers reject it |
| [Fencing token](./Fencing%20token.md) | Storage rejects writes from a superseded leader                  |
| Node fencing                          | The cluster cuts power or storage access to the old leader       |

A quorum does not stop an old leader from believing it still leads. In [Raft](./Raft.md), a deposed leader cut off from the majority can hold that belief for a while, but it cannot commit new entries. It could still serve stale reads unless it confirms its leadership first. Two-node clusters cannot form a majority after losing contact, so they need a third voter, often called a witness or tiebreaker.

_Usage:_

"After the switch came back, both database nodes were claiming to be primary."

"That's split brain. Stop writes on one before they diverge further, and add a witness so a two-node pair can't do this again."
