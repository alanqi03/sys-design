---
title: Cassandra
description: A one-page, system-design summary and technical overview of Apache Cassandra.
---

# Cassandra

## The one-page summary

Apache Cassandra is an open-source, distributed wide-column database designed
for high availability, high write throughput, and horizontal scale across many
machines or datacenters. It is a strong fit when the application can model data
around known queries and must continue operating through node, rack, or regional
failures.

### Developer Choices

- **Queries and tables.** List the queries first, then create tables that answer
  them directly; duplicating data across query-specific tables is normal because
  Cassandra has no joins.
- **Partition and clustering keys.** Choose a partition key that distributes
  traffic and keeps each partition bounded. Use clustering columns to order and
  range-query related rows inside that partition.
- **Replication topology.** Set a replication factor per datacenter with
  `NetworkTopologyStrategy`; replication factor 3 is a common starting point for
  tolerating a failed replica.
- **Consistency level.** Choose the number and location of replica responses per
  operation rather than setting one database-wide consistency mode:

  - **`LOCAL_ONE`:** wait for one replica in the local datacenter. Use it for the
    lowest local latency when stale reads are acceptable.
  - **`ONE`:** wait for one replica anywhere. It favors availability, but can
    cross datacenters and is less predictable than `LOCAL_ONE` in multi-DC setups.
  - **`LOCAL_QUORUM`:** wait for a majority of replicas in the local datacenter.
    Use it for most correctness-sensitive regional reads and writes; with RF=3,
    two local replicas must respond.
  - **`QUORUM`, `EACH_QUORUM`, or `ALL`:** wait for broader or complete replica
    agreement. Reserve these for requirements that justify higher latency and
    lower availability, especially across datacenters.
- **Conflict handling.** Cassandra reconciles ordinary concurrent updates by
  timestamp, so clocks, idempotency, and ownership of each field matter. Use a
  lightweight transaction only when compare-and-set semantics are essential.
- **Compaction, TTL, and repair.** Select a compaction strategy for the workload,
  model TTL-heavy data carefully, and schedule repairs before tombstones can be
  garbage-collected.

### Pros

1. **High write throughput.** The append-oriented commit-log and memtable path
   avoids read-before-write work and distributes writes across nodes.
2. **High availability without a primary.** Any node can coordinate a request,
   and replica placement spans failure domains.
3. **Horizontal and geographic scale.** Adding nodes expands storage and
   throughput; one cluster can replicate across several datacenters.
4. **Tunable consistency.** Each operation can trade consistency against latency
   and availability instead of accepting one policy for every workload.

### Cons

- **Data modeling is query-specific.** Denormalization and duplicated writes are
  common, while ad hoc queries, joins, and cross-partition aggregations are poor
  fits.
- **It is operationally demanding.** Compaction, repairs, tombstones, JVM
  behavior, disk headroom, topology changes, and rolling upgrades need active
  engineering.
- **Strong coordination is expensive.** Quorum requests cost latency and
  availability; Paxos-based lightweight transactions cost substantially more
  than ordinary reads and writes.
- **Bad partitions hurt badly.** Hot or oversized partitions concentrate load,
  increase read amplification, and make compaction and repair harder.

### Numbers to Know

- **Replication factor 3** is a common production baseline; a quorum is then
  **2 replicas**.
- Keep a partition below roughly **100,000 values and 100 MB on disk** as a
  general starting guideline, then test the real workload.
- The default tombstone grace period is **864,000 seconds (10 days)**, so every
  replica must be repaired more frequently than that to avoid deleted data
  resurfacing.
- A typical production node has at least **8 CPU cores and 32 GB RAM**, but disk,
  network, compaction, and data shape often determine capacity first.
- For early design, assume **thousands to tens of thousands of small operations
  per second per well-sized node**, then benchmark; balanced clusters can scale
  into hundreds of thousands or millions of operations per second by adding
  nodes.

### Scaling Reads

- **Model a table per access pattern.** A direct partition-key query is far
  cheaper and more predictable than filtering across token ranges.
- **Add nodes and balance tokens.** More replicas can serve more reads when
  partition keys and client requests distribute evenly.
- **Use local consistency.** `LOCAL_ONE` minimizes latency; `LOCAL_QUORUM` gives
  overlapping local read/write quorums when stronger visibility is required.
- **Reduce read amplification.** Choose the right compaction strategy, keep
  partitions bounded, and allow Bloom filters and the OS page cache to avoid
  unnecessary disk work.
- **Use indexes selectively.** Cassandra 5.0 Storage-Attached Indexing (SAI)
  supports additional filters, but primary-key query tables remain the most
  predictable path for critical high-volume reads.

### Scaling Writes

- **Spread partition keys.** High-cardinality keys distribute mutations across
  coordinators and replicas; bucket hot entities by time or shard when needed.
- **Add nodes.** Cassandra's shared-nothing design expands write throughput and
  storage horizontally when data and traffic remain balanced.
- **Use the lowest safe consistency level.** Fewer required acknowledgements
  improve latency and availability but weaken immediate read visibility.
- **Write concurrently, not in giant batches.** Use asynchronous independent
  mutations for throughput; logged batches are for atomicity, not bulk loading,
  and work best when changes share a partition.
- **Protect background I/O.** Fast commit-log and data disks, disk headroom, and
  a workload-appropriate compaction strategy keep flushes and compaction from
  stalling foreground writes.

## Overview

### How It Works

#### Nodes, tokens, and coordinators

Cassandra has a peer-to-peer architecture with no permanent leader. A client
connects to any node, which becomes the **coordinator** for that request. The
coordinator hashes the partition key into a token, finds the nodes responsible
for that token range, and contacts the replicas required by the request's
consistency level.

Each keyspace defines a replication strategy and replication factor.
`NetworkTopologyStrategy` places copies across racks and datacenters according
to the configured topology. Gossip and failure detection help nodes understand
cluster membership and which peers appear reachable.

#### Storage and memory

Cassandra uses a Log-Structured Merge (LSM) storage engine. Recent mutations
live in per-table in-memory **memtables** and durable append-only **commit logs**.
Memtables eventually flush into immutable, sorted files called **SSTables**.
Updates do not modify an existing SSTable in place, so one logical partition can
have versions spread across several SSTables until compaction merges them.

The Java heap holds active structures, while off-heap memory and the operating
system page cache accelerate indexes and SSTable data. Cassandra does not load a
whole table—or necessarily a whole partition—into memory to query it.

#### What happens on a write

1. The coordinator hashes the partition key and sends the mutation to every
   replica selected by the keyspace's replication strategy, regardless of the
   requested consistency level.
2. Each receiving replica appends the mutation to its commit log for durability
   and applies it to the table's memtable in memory.
3. The coordinator returns success after enough replicas acknowledge the write
   for the selected consistency level. A lower level returns sooner and can
   succeed through more failures.
4. When a memtable reaches a threshold, Cassandra writes it sequentially to a
   new immutable SSTable. The commit-log segments covered by flushed data can
   then be recycled.
5. Background compaction later reads several SSTables, reconciles timestamps and
   tombstones, and writes fewer replacement SSTables. This creates write
   amplification even though the foreground write path is fast.

If a replica is unavailable, the coordinator may retain a hint and replay it
later. Hints are best effort; repair is still required to synchronize replicas
that missed changes.

#### What happens on a read

1. The coordinator routes the request to enough replicas to satisfy the chosen
   consistency level, preferring nearby and responsive replicas.
2. Each replica checks its memtable and relevant SSTables. Bloom filters help
   skip SSTables that definitely do not contain the partition; indexes locate
   candidate data within files.
3. The replica merges versions from memory and disk, applies tombstones and TTLs,
   and sends its result or digest to the coordinator.
4. The coordinator reconciles replica responses by timestamp and returns the
   newest values it can establish at that consistency level. Conflicting
   ordinary writes use last-write-wins timestamp resolution.
5. Depending on read-repair configuration and detected disagreement, Cassandra
   may repair replicas on the read path; scheduled repair remains necessary for
   full anti-entropy.

Reads usually touch more structures than writes. Large partitions, many
overlapping SSTables, tombstones, and broad range scans therefore increase read
latency and resource use.

#### Deletes, compaction, and repair

A delete writes a timestamped **tombstone** instead of immediately removing
bytes from every replica and SSTable. Reads use the tombstone to hide older
values. After `gc_grace_seconds` and a qualifying compaction, the marker can be
purged—but only safely if repair has already delivered the deletion to all
replicas.

Repair compares replica data for shared token ranges using Merkle trees and
streams differences. Cassandra does not automatically run cluster-wide repair;
operators must schedule it. Compaction and repair consume significant disk and
network I/O, so production capacity needs headroom beyond foreground traffic.

#### Multi-datacenter behavior

One Cassandra cluster can span datacenters with an independent replication
factor in each. Applications normally send requests to local coordinators and
use `LOCAL_ONE` or `LOCAL_QUORUM` so a slow inter-region link is not on every
request's critical path. Writes are still forwarded to remote replicas, and
eventual repair resolves missed mutations.

Requiring `EACH_QUORUM` or `ALL` adds cross-datacenter coordination and reduces
availability during a regional or network failure. Multi-DC design must define
which trade-off each operation actually needs.

### Features

- CQL tables with partition keys, compound partition keys, clustering columns,
  static columns, collections, tuples, and user-defined types
- Tunable consistency levels including `ONE`, `LOCAL_ONE`, `QUORUM`,
  `LOCAL_QUORUM`, `EACH_QUORUM`, and `ALL`
- Multi-datacenter, rack-aware replication with `NetworkTopologyStrategy`
- Conditional writes and lightweight transactions using Paxos; logged and
  unlogged batches for specific atomic grouping patterns
- Per-value and table-default TTLs, timestamp-based conflict resolution, and
  atomic counters
- Storage-Attached Indexing (SAI), including numeric/text filtering and vector
  search in Cassandra 5.0
- Change Data Capture (CDC), triggers, user-defined functions and aggregates,
  and pluggable storage/index extensions
- Multiple compaction strategies, compression, Bloom filters, caching, snapshots,
  incremental backups, hinted handoff, and repair
- Authentication, role-based authorization, client encryption, internode
  encryption, and audit logging
- Metrics through JMX and virtual tables, plus `nodetool` and `cqlsh` operational
  tooling

### When to choose Cassandra

Choose Cassandra for write-heavy, always-on workloads with very large data sets,
known key-based access patterns, and a need to scale across machines or
datacenters without a single write primary. Common fits include time-series and
IoT data, activity feeds, messaging metadata, event capture, observability data,
and durable high-volume state keyed by user, device, or tenant.

Reconsider it when the product needs joins, flexible ad hoc querying, strong
multi-row transactions, database-enforced relationships, or a small data set
that does not justify operating a distributed cluster. PostgreSQL is usually a
better default for relational correctness; DynamoDB may be preferable when a
fully managed AWS service is more important than Cassandra's portability and
operational control.

### Sources and further reading

- [Apache Cassandra documentation: architecture](https://cassandra.apache.org/doc/stable/cassandra/architecture/index.html)
- [Apache Cassandra documentation: Dynamo model, replication, and consistency](https://cassandra.apache.org/doc/stable/cassandra/architecture/dynamo.html)
- [Apache Cassandra documentation: storage engine](https://cassandra.apache.org/doc/stable/cassandra/architecture/storage-engine.html)
- [Apache Cassandra documentation: guarantees](https://cassandra.apache.org/doc/stable/cassandra/architecture/guarantees.html)
- [Apache Cassandra documentation: data-model analysis](https://cassandra.apache.org/doc/stable/cassandra/developing/data-modeling/intro.html)
- [Apache Cassandra documentation: repair](https://cassandra.apache.org/doc/stable/cassandra/managing/operating/repair.html)
- [Apache Cassandra documentation: compaction and tombstones](https://cassandra.apache.org/doc/stable/cassandra/managing/operating/compaction/overview.html)
- [Apache Cassandra documentation: hardware choices](https://cassandra.apache.org/doc/stable/cassandra/managing/operating/hardware.html)
- [Apache Cassandra documentation: Storage-Attached Indexing](https://cassandra.apache.org/doc/stable/cassandra/developing/cql/indexing/sai/sai-overview.html)
- [Apache Cassandra 5.0 release announcement](https://cassandra.apache.org/_/blog/Apache-Cassandra-5.0-Announcement.html)

