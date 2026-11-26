---
title: Quorums
description: N, W, and R; why R + W > N means reads see the latest write; majorities; and the limits of quorums.
---

# Quorums

When data has several copies, waiting for **every** copy is slow and fails as
soon as one machine is down. Waiting for **one** copy is fast but risks reading
stale data. A **quorum** is the middle ground: wait for *enough* copies.

**Running example:** a cart stored on **3 replicas** (A, B, C) in a leaderless
database like Cassandra.

## N, W, and R

| Letter | Meaning | Example |
| --- | --- | --- |
| **N** | Number of copies | 3 |
| **W** | Copies that must confirm a write before it counts | 2 |
| **R** | Copies that must answer a read | 2 |

A write goes to all 3 replicas, but the client only waits for 2 of them to say
OK. A read asks all 3 and uses the first 2 answers.

## Why R + W > N works

With N = 3, W = 2, R = 2: the write reached at least 2 of the 3 replicas, and
the read hears from 2 of the 3. Two groups of 2 out of 3 **must overlap in at
least one replica**, so the read always includes at least one copy that has the
latest write.

```{mermaid}
flowchart LR
  subgraph write["Write 'milk' (W = 2)"]
    A1[(A: milk)]
    B1[(B: milk)]
    C1[("C: old, missed it")]
  end
  subgraph read["Read (R = 2)"]
    B2[(B: milk)]
    C2[(C: old)]
  end
  write --> read
```

The read gets "milk" from B and the old value from C. Each value carries a
version (often a timestamp), so the reader picks the newer one, "milk."

## Picking W and R

| N | W | R | R + W > N? | Behavior |
| --- | --- | --- | --- | --- |
| 3 | 2 | 2 | Yes | Balanced. The common choice. Survives 1 replica down |
| 3 | 3 | 1 | Yes | Very fast reads, but writes fail if any replica is down |
| 3 | 1 | 3 | Yes | Fast writes, but reads fail if any replica is down |
| 3 | 1 | 1 | No | Fastest. Reads may be stale (eventual consistency) |

In Cassandra, `LOCAL_QUORUM` for both reads and writes is the N = 3, W = 2,
R = 2 row. `LOCAL_ONE` is the last row.

## Majorities and failure tolerance

A **majority quorum** is more than half: `floor(N / 2) + 1`.

| N | Majority | Failures tolerated |
| --- | --- | --- |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

That's why clusters use **odd** sizes: 4 nodes tolerate no more failures than 3,
but cost more and need more votes. Two majorities always overlap, which is also
the core idea behind [consensus](consensus.md).

## Fixing replicas that missed a write

Replica C missed "milk." Leaderless databases catch it up in three ways:

- **Read repair:** a read that notices C is behind writes the newer value back
  to C.
- **Hinted handoff:** if C was down, another replica stores the write as a
  "hint" and passes it to C when C comes back.
- **Anti-entropy repair:** a background job compares replicas and fixes
  differences. In Cassandra, this is the `repair` you must run regularly.

## Limits of quorums

R + W > N makes a read *very likely* to see the latest write, but it's not a
full guarantee of strong consistency:

- **Concurrent writes:** if two clients write to the same cart at the same time,
  replicas may disagree on which came last. "Newest timestamp wins" silently
  drops one write, and server clocks can be off. [Version
  vectors](vector-clocks.md) detect these conflicts instead.
- **Sloppy quorums:** some systems accept writes on *any* reachable nodes
  during an outage, so the overlap guarantee no longer holds.
- **Partial failures:** a write that reached only 1 replica reports failure, but
  that 1 copy may still spread later.

For a strict single order of writes, use a leader chosen by
[consensus](consensus.md).

## Further reading

- [Amazon Dynamo paper (2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Cassandra: Consistency levels](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html)
