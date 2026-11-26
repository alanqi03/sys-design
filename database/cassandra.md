---
title: Cassandra
description: A one-page, system-design summary and technical overview of Apache Cassandra.
---

# Cassandra

## The one-page summary

Apache Cassandra is an open-source distributed database built for **huge write
volume** and **staying up when machines or whole datacenters fail**. There is no
primary: every node can take reads and writes.

### Example

A chat app needs "show the latest 50 messages in a channel." In Cassandra, you
design one table per query:

```sql
CREATE TABLE messages_by_channel (
  channel_id UUID,
  bucket     TEXT,       -- e.g. '2026-11', keeps each partition bounded
  sent_at    TIMEUUID,
  sender_id  UUID,
  body       TEXT,
  PRIMARY KEY ((channel_id, bucket), sent_at)
) WITH CLUSTERING ORDER BY (sent_at DESC);

SELECT * FROM messages_by_channel
WHERE channel_id = ? AND bucket = '2026-11'
LIMIT 50;
```

- **Partition key** `(channel_id, bucket)` decides which nodes store the rows.
  One channel-month lives together.
- **Clustering column** `sent_at` keeps the rows sorted newest first, so the
  query is one sequential read.
- The `bucket` stops a busy channel from growing one partition forever.

To show "all messages from user X," you'd create a second table,
`messages_by_user`, and write each message to both. Duplicating data is normal
in Cassandra because it has no joins.

### Developer Choices

- **Queries → tables.** List queries first; create one table per query.
- **Partition key.** Spreads traffic and keeps each partition bounded in size.
- **Replication factor.** Usually 3 copies per datacenter.
- **Consistency level per request.** How many of those copies must respond:
  - `LOCAL_ONE`: one copy in the local datacenter. Fastest; may read stale data.
  - `LOCAL_QUORUM`: a majority (2 of 3) in the local datacenter. The usual
    choice when correctness matters.
  - `QUORUM` / `EACH_QUORUM` / `ALL`: more copies, across datacenters. Slower,
    and fails more easily during outages.

  If writes and reads both use `LOCAL_QUORUM` with 3 copies, every read
  overlaps at least one copy that has the latest write. See
  [Quorums](../distributed-systems/quorums.md) for why.

### Pros

1. **Very fast writes:** a write is a log append plus an in-memory update.
2. **Always on:** no primary to fail; copies span racks and datacenters.
3. **Scales by adding nodes:** storage and throughput grow roughly linearly.
4. **Tunable consistency:** choose speed or freshness per request.

### Cons

- **Rigid queries:** no joins or ad hoc filters; a new query often needs a new
  table.
- **Heavy to operate:** compaction, repairs, and tuning need ongoing work.
- **Last write wins:** concurrent updates to the same value are resolved by
  timestamp, so one silently overwrites the other.
- **Oversized or hot partitions** slow everything down.

### Numbers to Know

- **3 copies** per datacenter is standard; a quorum is **2**.
- Keep partitions under about **100 MB** and **100,000 rows**.
- Run repair at least every **10 days**, or deleted data can reappear.
- Expect **thousands to tens of thousands of operations/s per node**; add nodes
  to reach millions.

### Scaling Reads

- Make every important read a single-partition query against a table built for
  it.
- Add nodes so the data and the reads spread out.
- Use `LOCAL_ONE` when slightly stale data is OK, and `LOCAL_QUORUM` when it
  isn't.
- Keep partitions small.

### Scaling Writes

- Use partition keys with many values; add a time bucket for busy keys.
- Add nodes.
- Use the lowest consistency level that's safe.
- Send many small writes in parallel rather than one giant batch.

## Overview

### How it works

```{mermaid}
flowchart LR
  client[Client] --> coord["Any node<br/>(coordinator)"]
  coord --> r1[(Replica 1)]
  coord --> r2[(Replica 2)]
  coord --> r3[(Replica 3)]
```

- **Routing:** the client talks to any node. That node hashes the partition key
  to find which 3 nodes own it, forwards the request, and waits for as many
  replies as the consistency level requires.
- **Writes:** each replica appends the write to a commit log on disk (for
  durability) and to an in-memory table. The in-memory table is later flushed to
  an immutable file on disk (an SSTable). Nothing is updated in place.
- **Reads:** a replica may have to merge one row's pieces from memory and
  several SSTables. **Compaction** merges SSTables in the background to keep
  reads fast.
- **Deletes:** a delete writes a marker (a tombstone) instead of erasing data.
  **Repair** syncs replicas so every copy sees the tombstone before it's
  removed.
- **Multiple datacenters:** each datacenter keeps its own copies. Use `LOCAL_*`
  consistency levels so requests don't wait on other regions.

### When to choose Cassandra

Choose it for write-heavy, always-on workloads with huge data and known queries:
chat messages, activity feeds, IoT and time-series data, event logs.

Avoid it when you need joins, flexible queries, or multi-row transactions, or
when your data fits comfortably in PostgreSQL. If you want this model on AWS
without running a cluster, consider DynamoDB.

### Further reading

- [Apache Cassandra documentation: architecture](https://cassandra.apache.org/doc/stable/cassandra/architecture/index.html)
- [Apache Cassandra documentation: data modeling](https://cassandra.apache.org/doc/stable/cassandra/developing/data-modeling/intro.html)
- [Apache Cassandra documentation: storage engine](https://cassandra.apache.org/doc/stable/cassandra/architecture/storage-engine.html)
- [Apache Cassandra documentation: repair](https://cassandra.apache.org/doc/stable/cassandra/managing/operating/repair.html)
