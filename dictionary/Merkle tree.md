---
description: A tree of hashes that lets two parties find which parts of large datasets differ by exchanging only a few hashes.
---

A tree of hashes in which each leaf is the hash of a block of data and each parent is the hash of its children. The root therefore summarizes the entire dataset, and any change to a block changes every hash on the path up to the root. Ralph Merkle devised the structure around 1979.

In a [distributed system](./Distributed%20system.md), Merkle trees answer a specific question: which parts of two large datasets differ? Sending all the data, or even all the keys, between replicas ([nodes](./Node.md) holding copies) would consume a large amount of bandwidth just to confirm they mostly match. Two nodes can instead compare trees from the top down:

| Step | What happens                                                 |
| ---- | ------------------------------------------------------------ |
| 1    | Each node builds a tree over the same key ranges             |
| 2    | The nodes compare root hashes. If equal, the ranges match    |
| 3    | If different, they compare the hashes of the root's children |
| 4    | They descend only into subtrees whose hashes differ          |
| 5    | At the leaves, they exchange the data for mismatched ranges  |

If only one leaf differs, the comparison follows a single path, so the number of levels visited grows with the logarithm of the number of leaves. This is why [anti-entropy](./Anti-entropy.md) in stores such as Dynamo and Cassandra uses these trees. Git and blockchains use the same structure or close variants.

Matching hashes indicate matching data with overwhelming probability, not certainty.

Tree granularity is a practical trade-off. If each leaf covers many keys, the tree stays small, but one differing key forces the whole range to be transferred, a problem called overstreaming. It shows up as a repair moving more data than differed. More leaves reduce that waste and use more memory.

_Usage:_

"Repair streamed 200 GB, but only a few thousand rows were out of sync."

"The Merkle tree leaves are too coarse, so one bad key resends the whole range. Try increasing the tree depth on that table."
