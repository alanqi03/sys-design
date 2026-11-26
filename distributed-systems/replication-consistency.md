---
title: Replication and Consistency
description: Leader-based, multi-leader, and leaderless replication, replication lag, consistency models, and CAP in plain language.
---

# Replication and Consistency

**Replication** means keeping copies of the same data on several machines,
called **replicas**.

**Running example:** a shopping cart service that stores every cart on 3
servers.

Why bother with 3 copies?

- **Survive failures:** if one server dies, the cart still exists on two others.
- **Handle more reads:** spread reads across all copies.
- **Be closer to users:** put a copy in each region.

The hard part is **keeping the copies the same** while they keep changing.

## Three ways to replicate

| Approach | Who accepts writes | Examples |
| --- | --- | --- |
| **Single leader** | One leader; followers copy it | PostgreSQL, MySQL, Redis, MongoDB |
| **Multi-leader** | One leader per region | Multi-region databases, offline-first apps |
| **Leaderless** | Any replica | Cassandra, Amazon's original Dynamo, Riak |

### Single leader (most common)

All writes go to the **leader**, which streams each change to the
**followers**. Reads can go to any copy.

```{mermaid}
flowchart LR
  app[Cart service] -->|writes| L[(Leader)]
  L -->|change stream| F1[(Follower 1)]
  L -->|change stream| F2[(Follower 2)]
  app -.->|reads| F1
  app -.->|reads| F2
```

The leader can wait for followers before confirming a write, or not:

- **Synchronous:** confirm only after a follower has the change. Nothing is lost
  if the leader dies, but writes are slower and stall if that follower is down.
- **Asynchronous:** confirm right away and copy in the background. Fast, but
  followers fall slightly behind (**replication lag**), and if the leader dies,
  its last few writes may be lost.

Most setups are asynchronous, or wait for just one follower.

**Replication lag in action:**

```{mermaid}
sequenceDiagram
  participant U as User
  participant L as Leader
  participant F as Follower
  U->>L: Add "milk" to cart
  L-->>U: OK
  U->>F: Show my cart
  F-->>U: Empty cart (change not copied yet)
  L->>F: Copy "milk" (100 ms later)
```

The user just added milk and doesn't see it. Common fixes:

- **Read your own writes:** read from the leader for a few seconds after a user
  writes, or for data only that user edits.
- **Sticky reads:** send each user to the same replica, so they never see time
  go backward.

**When the leader dies,** a follower must be promoted (**failover**). Picking
the new leader safely, without two servers both thinking they're the leader, is
a [consensus](consensus.md) problem.

### Multi-leader

Each region has its own leader and accepts writes locally, then syncs with the
others. Writes are fast everywhere, but two regions can change the same cart at
the same time. You must detect and resolve those **conflicts** (see
[version vectors](vector-clocks.md)). Avoid this approach unless you really need
local writes in every region.

### Leaderless

The client (or a coordinator node) sends each write to **all** replicas and
waits for some of them to confirm. Reads also ask several replicas. "How many
is enough?" is the subject of [Quorums](quorums.md). With no leader there's no
failover, but concurrent writes can conflict.

## Consistency: what can a read return?

| Model | Promise | Cart example |
| --- | --- | --- |
| **Strong** (linearizable) | Once a write is confirmed, every later read sees it | Add milk, and every device shows milk immediately |
| **Read your writes** | *You* always see your own writes | You see milk; your partner may not yet |
| **Eventual** | If writes stop, all copies end up the same | Milk shows up everywhere within a second or so |

Stronger promises need more coordination between replicas, so they cost
latency and availability. Pick per feature: a bank balance needs strong
consistency, a like counter is fine with eventual.

## CAP in one paragraph

A **network partition** is when replicas can't reach each other, for example
when a link between datacenters fails. During a partition, a system must choose:
**Consistency** (refuse some requests rather than risk returning stale data) or
**Availability** (answer every request, even if the answer might be out of
date). You can't have both while the partition lasts. When there's no
partition, the trade-off is between **latency and consistency**: waiting for more
replicas is safer but slower. This extension is called PACELC.

- **Leans consistent (CP):** etcd, ZooKeeper, a single-leader database with
  synchronous replication.
- **Leans available (AP):** Cassandra and DynamoDB at their default settings.

## Further reading

- *Designing Data-Intensive Applications*, chapters 5 and 9 (Martin Kleppmann)
- [Jepsen: Consistency models](https://jepsen.io/consistency)
