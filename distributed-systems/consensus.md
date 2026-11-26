---
title: "Consensus: Raft and Paxos"
description: How a cluster agrees on one leader and one ordered log despite crashes, explained through Raft, with a short note on Paxos.
---

# Consensus: Raft and Paxos

**Consensus** is how several servers agree on something, such as "who's the
leader?" or "what's the next change?", even when some of them crash or
messages are delayed. Once a majority agrees, the decision is final.

**Running example:** a 5-node etcd cluster storing configuration such as
`db_primary = node-2`.

## Why it's hard: split brain

Suppose the rule is "if you can't reach the leader, become the leader." Now a
network link fails and splits the cluster in two:

```{mermaid}
flowchart LR
  subgraph right["Side 2 (3 nodes)"]
    direction TB
    n3[Node 3: new leader]
    n4[Node 4]
    n5[Node 5]
  end
  subgraph left["Side 1 (2 nodes)"]
    direction TB
    n1[Node 1: leader]
    n2[Node 2]
  end
```

The two sides can no longer reach each other.

Both sides now have a leader and both accept writes. One side sets
`db_primary = node-2`, the other sets `db_primary = node-4`. When the link
heals, the data conflicts. This is **split brain**.

The fix is the **majority rule**: a leader needs votes from a majority (3 of 5),
and a change counts only once a majority stores it. Only one side of any split
can have a majority, so only one side can make progress.

## Raft

Raft (2014) was designed to be easy to understand. It splits the problem into
**leader election** and **log replication**.

### Roles and terms

Each node is a **follower**, **candidate**, or **leader**. Time is divided into
numbered **terms**, and each term has at most one leader.

```{mermaid}
flowchart LR
  F[Follower] -->|"no heartbeat, timeout"| C[Candidate]
  C -->|"wins majority of votes"| L[Leader]
  C -->|"another leader appears"| F
  L -->|"sees a higher term"| F
```

### Leader election

1. The leader sends a **heartbeat** to every follower every ~50 ms.
2. If a follower hears nothing for a random timeout (say 150–300 ms), it
   assumes the leader is dead, becomes a **candidate**, increases the term from
   4 to 5, and asks everyone for a vote.
3. Each node votes for **at most one** candidate per term.
4. A candidate with **3 of 5** votes becomes leader for term 5 and starts
   sending heartbeats.

The timeouts are random so that usually one follower times out first and wins,
instead of several splitting the vote. A node also refuses to vote for a
candidate whose log is behind its own, so a new leader always has every
committed change.

### Log replication

Every change is an entry in an ordered **log**. Only the leader adds entries.

```{mermaid}
sequenceDiagram
  participant C as Client
  participant L as Leader
  participant F1 as Follower 1
  participant F2 as Follower 2
  C->>L: set db_primary = node-2
  L->>L: append entry 7 to log
  L->>F1: append entry 7
  L->>F2: append entry 7
  F1-->>L: stored
  F2-->>L: stored
  Note over L: 3 of 5 have it (leader + 2), so entry 7 is committed
  L->>L: apply to state
  L-->>C: OK
```

Followers 3 and 4 may be slow or down. That's fine, because a majority already
stored the entry, and the leader keeps retrying until they catch up.

### What happens during the split

With the 2 | 3 split from earlier:

- If the old leader is on the **2-node side**, it can't get a majority, so none
  of its new writes commit. Clients there get errors or time out.
- The **3-node side** elects a new leader in term 5 and keeps working.
- When the link heals, the old leader sees term 5, steps down, and replaces its
  uncommitted entries with the new leader's log.

The minority side becomes **unavailable**, not wrong. Raft chooses consistency
over availability.

### Costs

- Every write waits for a round trip to a majority, so writes are slower than
  on a single machine.
- All writes go through one leader, which limits write throughput.
- Clusters are usually **3 or 5 nodes**. More nodes means more votes per write,
  not more speed.

That's why consensus stores small, important data, like who the leader is,
configuration, locks, and metadata, rather than all of an app's data. Systems
that need more throughput, like CockroachDB and TiKV, run many separate Raft
groups, one per data shard.

## Paxos

**Paxos** (Leslie Lamport, 1989) solves the same problem with the same
majority idea, and came first. It's famous for being hard to understand and
even harder to implement correctly, and the basic version only agrees on a
single value. Real systems use extensions like Multi-Paxos. Google's Chubby and
Spanner use Paxos. Raft was created as an easier-to-follow alternative with
the same guarantees.

## Who uses consensus

| System | Uses it for |
| --- | --- |
| **etcd** (Raft) | Kubernetes cluster state, leader election |
| **ZooKeeper** (Zab, similar to Raft) | Coordination for Hadoop, HBase, older Kafka. See [ZooKeeper](zookeeper.md) |
| **Kafka KRaft** (Raft) | Kafka's own cluster metadata, replacing ZooKeeper |
| **RabbitMQ quorum queues** (Raft) | Replicating each queue's messages |
| **CockroachDB, TiKV** (Raft) | Every row, via one Raft group per shard |
| **Consul** (Raft) | Service discovery and configuration |

## Consensus vs. quorums

Both rely on majorities, but they do different jobs:

| | [Quorum reads/writes](quorums.md) | Consensus |
| --- | --- | --- |
| Leader | None | One elected leader |
| Order of writes | Not agreed, so concurrent writes can conflict | One agreed order |
| Typical use | High-volume data (Cassandra) | Small, critical data (etcd) |

**In an interview,** don't design a consensus algorithm. Say "use etcd or
ZooKeeper for leader election," or "this database replicates with Raft, so a
write is safe once a majority has it."

## Further reading

- [The Raft paper: In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)
- [Raft visualization](https://raft.github.io/)
- [Paxos Made Simple (Lamport)](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
