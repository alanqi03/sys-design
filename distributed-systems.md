---
title: Distributed Systems Fundamentals
description: How copies of data stay in sync, how servers agree, and how to detect conflicting updates.
---

# Distributed Systems Fundamentals

Most systems keep **several copies** of their data on different machines, so a
single crash doesn't lose data or take the system down. Keeping those copies in
sync raises three questions:

1. **How many copies must confirm a write or answer a read?** →
   [Quorums](distributed-systems/quorums.md)
2. **How do servers agree on one leader and one order of writes?** →
   [Consensus: Raft and Paxos](distributed-systems/consensus.md)
3. **When two copies were changed at the same time, how do we notice?** →
   [Vector Clocks and Version Vectors](distributed-systems/vector-clocks.md)

Start with [Replication and Consistency](distributed-systems/replication-consistency.md)
for the basic vocabulary. [ZooKeeper](distributed-systems/zookeeper.md) shows how
consensus is packaged into a service other systems use for leader election,
locks, and configuration.

| Concept | One-line idea | Where you've seen it |
| --- | --- | --- |
| Replication | Keep copies of data on several machines | PostgreSQL replicas, Redis replicas |
| Quorum | Wait for a majority of copies, not all of them | Cassandra `LOCAL_QUORUM` |
| Consensus (Raft) | A majority agrees on one ordered log of changes | etcd, RabbitMQ quorum queues, Kafka (KRaft) |
| Version vectors | Per-copy counters that reveal conflicting writes | Amazon's Dynamo, Riak |
| ZooKeeper | A small, consistent store for coordination | Hadoop, HBase, older Kafka |

```{tip}
In interviews you rarely implement these. You need to know which problem each
one solves, and say things like "use etcd for leader election" or "use quorum
reads and writes so a read always sees the latest write."
```
