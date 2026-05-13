---
title: "The beauty of Kademlia: the XOR distance"
date: 2023-07-10
draft: false
tags: ["distributed-systems", "p2p", "kademlia"]
summary: "The working principle of Kademlia, in particular relating to the XOR distance metric."
---

Distributed systems need a way to find things without a central index.
Kademlia, introduced in 2002 by Petar Maymounkov and David Mazières, solves this with a deceptively simple idea:
treat node IDs as points in a binary space, and measure distance with XOR.

## Why XOR?

XOR has a property that makes it perfect for routing: for any two points
`A` and `B`, the distance `d(A, B) = A XOR B` is:

- **Symmetric**: `d(A, B) = d(B, A)`
- **Zero iff equal**: `d(A, A) = 0`
- **Triangle inequality holds** (as a metric over GF(2)^n)
- **Unidirectional**: for a given point `A` and distance `d`, there is
  exactly one point `B` such that `d(A, B) = d`. This means lookups
  converge monotonically — you never overshoot.

## One space for everything

A foundational decision in Kademlia is that node IDs and content keys
**live in the same space**. This is a very important point that I didn't precisely catch at the beginning of my study of Kademlia, and precisely what makes Kademlia work _:)_.

A node is assigned a random 160-bit ID; a
file or value is stored under a key that is also a 160-bit identifier
(typically the SHA-1 or SHA-256 hash of the content). Both are just
points in the same binary space, and XOR distance applies equally to
both.

This means routing to a value works exactly like routing to a node:
you treat the content key as if it were a node ID and walk toward it.
No separate lookup mechanism is needed.

The simplicity this iplies is
considerable: one algorithm, one metric, one routing table, for both
node discovery and content retrieval.

## The routing table: k-buckets

Each node maintains a routing table structured as a binary tree. For each
possible distance prefix `[2^i, 2^(i+1))`, it keeps at most `k` contacts
(typically `k = 20`). These are the **k-buckets**.

When a bucket is full and a new contact arrives, Kademlia checks whether
the *least-recently seen* node is still alive. If it is, the new node is
dropped; if it has gone silent, it is evicted. This bias toward long-lived
nodes makes the routing table robust — nodes that have been up for an hour
are statistically more likely to stay up than nodes that just joined.

## Node lookup

To find the node closest to a target key `t`, you start with what we call
_"the `α` (alpha)"_, typically the 3 closest known nodes and send
them parallel `FIND_NODE(t)` requests. Each responds with its own closest known nodes.

You recurse, always advancing toward the target,
**until** the closest node you've found stops changing.

The algorithm terminates in O(log n) rounds.

_Why?_

Because each step gets you at least **1 bit closer** to the target in the **XOR ordering**, i.e. you halve the remaining distance in the XOR sense. There are at most 160 bits of prefix to exhaust, but with n nodes the effective depth is log(n) since the tree is sparse beyond that.

## Storing and retrieving values

`STORE(key, value)` publishes to the `k` nodes closest to the key.

`FIND_VALUE(key)` runs like a node lookup but stops as soon as any node
returns the stored value. Publishers re-announce every hour so content
stays alive even as nodes churn.

Thus the k closest nodes all store a copy of the value — k (typically 20) is chosen to be large enough that even with significant churn (many nodes coming and going in a short time), at least one node holding the value is likely to remain online. 
This is deliberate replication for availability.

The re-announcement every hour (publishers re-publish the value periodically) is a complementary mechanism for resilience: as the set of closest nodes shifts due to churn, the value gets replicated to newly-closest nodes before old ones disappear.

## Some trailing words about the XOR metric

The XOR metric is an example of how the right mathematical abstraction collapses complexity: one operator unifies node lookup and content retrieval, routing and storage.

Moreover, the fact that for any two points there is exactly one point at a given XOR distance, this makes convergence inevitable!

**Comparison with "Chord" alternative:**
Chord (by Stoica et al., 2001) is another Distributed Hash Table (DHT),
this one organizing nodes on a one-dimensional ring, ordered by ID.
Routing hops along the ring using finger tables that jump by powers of two.
It achieves O(log n) lookups like Kademlia, but the ring metric is asymmetric,
i.e. the distance from A to B is not the same as from B to A,
so routing tables require careful directional logic and lookups
have edge cases at ring boundaries. XOR is symmetric
by definition, which iplies a node's routing table covers both directions
simultaneously and the lookup algorithm needs no special cases.

The deeper point is that XOR is not just a convenience — it is the *tightest
possible* metric for this problem. It maps naturally onto binary tries, it
guarantees monotonic convergence, and it makes the keyspace for nodes and
content truly interchangeable. Chord achieves similar complexity but with more _"machinery"_.
Kademlia finds the elegant path: one binary operator, and the rest
follows.

## References

- [bmuller/kademlia](https://github.com/bmuller/kademlia) — clean Python reference; `routing.py` for the bucket logic
- [go-libp2p-kad-dht](https://github.com/libp2p/go-libp2p-kad-dht) — IPFS's implementation; `routing_table.go`

- Maymounkov, P., & Mazières, D. (2002). **Kademlia: A peer-to-peer information system based on the XOR metric**. *Proceedings of the 1st International Workshop on Peer-to-Peer Systems (IPTPS '02)*. Lecture Notes in Computer Science, vol 2429. Springer.
- Baumgart, I., & Mies, S. (2007). **S/Kademlia: A practicable approach towards secure key-based routing**. *13th International Conference on Parallel and Distributed Systems (ICPADS '07)*. IEEE.
- Stoica, I., Morris, R., Karger, D., Kaashoek, M. F., & Balakrishnan, H. (2001). **Chord: A scalable peer-to-peer lookup service for internet applications**. *Proceedings of ACM SIGCOMM '01*.
