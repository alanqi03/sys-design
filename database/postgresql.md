---
title: PostgreSQL
description: A one-page, system-design summary of PostgreSQL.
---

# PostgreSQL

## The one-page summary

PostgreSQL (often called Postgres) is an open-source relational database. It can
safely coordinate thousands of simultaneous operations, enforce business rules,
recover after a crash, and replicate its data. It is an excellent default for
applications whose data has relationships or whose correctness matters—orders,
accounts, inventory, user profiles, billing, and most business software.

### Developer Choices

Before building on PostgreSQL, a developer should make these choices explicitly:

- **Table schema.** Choose tables, column types, keys, nullability, constraints,
  and normalization based on the domain; plan migrations and partition only when
  scale or retention requires it.
- **Indexes.** Index columns used by important lookups, joins, ranges, and sorts,
  then verify with query plans. Every index consumes space and slows writes, so
  avoid indexing every column.
- **Data consistency.** Use the primary for fresh data and invariants; use
  eventually consistent replicas or caches only where stale reads are safe.
  Choose synchronous replication when stronger durability justifies added
  commit latency.
- **Isolation level.** PostgreSQL exposes four SQL-standard names but implements
  three distinct behaviors:

  - **Read Uncommitted:** in PostgreSQL this is an alias for Read Committed, so it
    never exposes dirty rows. Prefer the clearer Read Committed name.
  - **Read Committed:** the default; each statement sees a new committed
    snapshot. Use it for ordinary short CRUD transactions with constraints,
    atomic statements, or explicit locks handling conflicts.
  - **Repeatable Read:** one transaction sees one stable snapshot. Use it for
    multi-query calculations or exports, and retry the whole transaction after
    a concurrency failure.
  - **Serializable:** committed transactions behave as if run one at a time.
    Use it for cross-row business invariants, with short transactions and
    bounded retries for serialization failures.

### Pros

1. **Strong transactions and data integrity.** PostgreSQL provides ACID
   transactions, isolation levels, constraints, and foreign keys that protect
   invariants under concurrent writes.
2. **Excellent read and query performance.** A mature optimizer, several index
   families, joins, and parallel execution support point lookups and complex
   reporting.
3. **Rich features.** JSONB, arrays, full-text search, custom types, and
   extensions cover many workloads without another data store.
4. **Excellent tooling and ecosystem.** Mature drivers, managed offerings,
   backup and migration tools, connection poolers, and observability integrations
   make PostgreSQL easy to adopt and operate.

### Cons

- **Writes are harder to scale than reads.** A normal PostgreSQL cluster accepts
  writes on one primary, and durable commits wait on WAL storage; read replicas
  do not increase primary write throughput.
- **It is not a globally distributed database out of the box.** Multi-region,
  active-active writes need external tooling or sharding; synchronous replication
  adds latency, while asynchronous replication permits lag and possible loss.
- **MVCC needs housekeeping.** Autovacuum must reclaim old row versions or tables
  and indexes can bloat.
- **Connections and tuning need attention.** PostgreSQL traditionally uses a
  process per connection, so large fleets often need a pooler plus query,
  memory, and checkpoint tuning.

### Numbers to Know

For an early system-design estimate, think **thousands of non-trivial write
transactions per second** and **tens of thousands of simple, cache-hot reads per
second** on a well-sized single primary—not as limits, but as signals for when to
benchmark.

### Scaling Reads

- **Optimize first.** Use query plans and workload-shaped indexes, eliminate N+1
  queries, and keep planner statistics current.
- **Scale up.** Add RAM for a larger cache, CPU for concurrent queries, faster
  storage for cache misses, and a pooler to control connections.
- **Cache repeated work.** Application caches, CDNs, and materialized views trade
  some freshness for fewer database reads.
- **Add read replicas.** Route stale-tolerant traffic to hot standbys, but keep
  fresh, transactional, and read-after-write queries on the primary.
- **Isolate analytics.** Run long scans on a dedicated replica or warehouse so
  they do not compete with transactional requests.

### Scaling Writes

- **Make writes cheaper.** Keep transactions short, batch work, use `COPY` for
  bulk loads, remove unnecessary indexes, and avoid contention on hot rows.
- **Scale up the primary.** More CPU, RAM, storage IOPS, and lower WAL latency
  raise single-writer capacity; connection pooling and checkpoint tuning smooth
  bursts.
- **Move secondary work out of band.** Use an outbox plus a stream or queue to
  update search, analytics, and notifications asynchronously.
- **Partition carefully.** Native partitioning helps pruning and maintenance but
  does not spread writes beyond the primary.
- **Shard last.** Multiple primaries can divide writes by tenant or account, but
  add routing, rebalancing, and cross-shard transaction complexity.

### Notes

- **Optimistic locking** is a technique where we first read the current state of
  the business and then attempt to update it. If the state has changed since we
  read it, our update fails and we can retry.
- **Change Data Capture (CDC)** can read committed changes from PostgreSQL's WAL
  through logical decoding or logical replication. A connector can publish
  those changes as events to a queue or stream for downstream processing.
- **PostGIS** is a PostgreSQL extension that adds geographic and geometric data
  types, spatial indexes, and functions for querying location data.
- **Full-text search** is built into PostgreSQL and can index, rank, and query
  natural-language text; extensions can add more specialized search behavior.

## Overview

### What ACID Means

- **Atomicity:** a transaction is all-or-nothing. If transferring money requires
  a debit and a credit, either both changes commit or neither does.
- **Consistency:** a successful transaction leaves the database obeying its
  declared rules, such as data types, unique constraints, checks, and foreign
  keys. The application is still responsible for declaring the right rules.
- **Isolation:** concurrent transactions are kept from observing unsafe partial
  results from one another. PostgreSQL offers several isolation levels so an
  application can trade stricter behavior for more concurrency.
- **Durability:** after PostgreSQL confirms a commit, the change survives a
  process or machine crash under the configured durability settings. WAL is the
  key mechanism behind this guarantee.

### How It Works

#### Connections and processes

PostgreSQL is a client/server system. A main server process accepts connections
and normally gives each connection its own backend process. That process parses
SQL, asks the query planner for an execution plan, and runs the plan. A
connection pooler lets many application connections share a smaller, controlled
number of database connections.

#### Storage and memory

Tables and indexes are persisted on disk as files divided into fixed-size pages,
normally 8 KiB each. PostgreSQL does **not** load an entire table into memory
before using it. Its shared-buffer cache holds a working set of recently or
frequently accessed pages; the operating system may cache those files too. If a
needed page is absent from both caches, PostgreSQL reads it from storage. More
RAM helps when the active data and indexes can remain cached, but large tables
can be queried a page at a time.

#### What happens on a read

1. PostgreSQL parses the SQL and validates names, types, and permissions.
2. The planner compares possible strategies—such as an index scan, sequential
   scan, join order, or parallel plan—using table statistics and estimated cost.
3. The executor follows the chosen plan and requests the necessary table and
   index pages. It finds each page in shared buffers or loads it from the OS
   cache or storage.
4. PostgreSQL applies filters and checks **MVCC visibility**. Each transaction
   sees a consistent snapshot, so it reads the row version valid for that
   snapshot and ignores versions it should not yet—or can no longer—see.
5. The executor performs any joins, grouping, sorting, or aggregation and returns
   the result to the client.

Because reads use snapshots, readers normally do not block writers, and writers
normally do not block readers. They can still compete for CPU, memory, and I/O,
and explicit locks or schema changes can block work.

#### What happens on a write

1. PostgreSQL finds and locks the rows that must change, preventing conflicting
   writers from silently overwriting one another.
2. An `INSERT` creates a row version. An `UPDATE` creates a new version and
   leaves the old one for transactions whose snapshots still need it. A
   `DELETE` marks a version as no longer visible rather than immediately erasing
   its bytes.
3. PostgreSQL creates **write-ahead log (WAL)** records describing the change and
   updates the affected table and index pages in memory. These pages are now
   “dirty.”
4. On a durable commit, the relevant WAL is flushed to persistent storage before
   success is returned. The dirty table pages can be written later because a
   crash can replay WAL to reconstruct the committed state.
5. Background writers and checkpoints gradually flush dirty pages. Autovacuum
   later reclaims row versions that no active transaction can see and updates
   statistics used by the planner.

Indexes speed up reads but make this write path more expensive: each affected
index may need its own new entry and WAL records.

#### Replication and recovery

The primary can stream WAL to standby servers. A hot standby replays those
changes and can serve read-only queries. Asynchronous replication keeps primary
commits fast but allows lag; synchronous replication can wait for a standby and
therefore adds latency. Archived WAL plus a base backup also enables
point-in-time recovery. If the primary crashes, PostgreSQL replays WAL so changes
from committed transactions are restored even if their table pages had not been
flushed yet.

### Features

- ACID transactions, savepoints, two-phase commit, and read committed,
  repeatable read, and serializable isolation
- Primary keys, foreign keys, unique/check/exclusion constraints, and generated
  columns
- Joins, common table expressions, recursive queries, window functions, views,
  and materialized views
- B-tree, hash, GiST, SP-GiST, GIN, and BRIN indexes, including partial,
  expression, multicolumn, and covering indexes
- Rich types including JSON/JSONB, arrays, UUIDs, ranges, network addresses,
  XML, enums, and user-defined types
- Full-text search and optional geospatial (PostGIS) and vector (`pgvector`)
  search through extensions
- Stored functions/procedures, triggers, notifications, extensions, and foreign
  data wrappers
- Declarative range, list, and hash table partitioning
- Physical streaming replicas, hot standbys, logical replication, WAL archiving,
  backups, and point-in-time recovery
- Roles, TLS, row-level security, column privileges, and auditing extensions

### When to choose PostgreSQL

Choose PostgreSQL when transactions, relational data, flexible queries, and a
proven ecosystem matter. Reconsider a single PostgreSQL primary when the design
requires sustained writes beyond one machine, low-latency writes in several
continents at once, or an access pattern better served by a specialized store.
Even then, PostgreSQL often remains the system of record while caches, search
engines, warehouses, or streams serve specialized paths.

### Sources and further reading

- [PostgreSQL documentation: transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL documentation: MVCC](https://www.postgresql.org/docs/current/mvcc-intro.html)
- [PostgreSQL documentation: write-ahead logging](https://www.postgresql.org/docs/current/wal-intro.html)
- [PostgreSQL feature matrix](https://www.postgresql.org/about/featurematrix/)
- [PostgreSQL documentation: table partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [PostgreSQL documentation: log-shipping and standby servers](https://www.postgresql.org/docs/current/warm-standby.html)
- [PostgreSQL documentation: database page layout](https://www.postgresql.org/docs/current/storage-page-layout.html)
- [PostgreSQL documentation: using `EXPLAIN`](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL documentation: routine vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
