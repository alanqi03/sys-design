---
title: Databases
---

# Databases

A database makes state durable and queryable. Selecting one begins with access
patterns and correctness requirements, not with a product name.

This section starts with general database design principles, compares
[Change Data Capture and transactional outbox patterns](./database/change-data-capture-outbox.md),
then applies the principles to systems such as
[PostgreSQL](./database/postgresql.md).

[Redis](./caching/redis.md) can persist data and act as an in-memory database,
but this book places it under Caching because its most common system-design role
is accelerating or coordinating access around another source of truth. Its
chapter also covers the cases where Redis owns durable state.

## Model the workload

Write down the principal entities, expected data volume, read/write ratio, and
critical queries. For each operation, ask:

- Does it require a transaction across multiple records?
- How fresh must reads be?
- What latency and throughput are expected?
- Which fields are queried together?
- Can old data be archived or deleted?

A relational database is a strong default when relationships, constraints, and
transactions matter. Document, key-value, wide-column, graph, and time-series
databases specialize in different access patterns or scaling models.

## Indexes

An index is an auxiliary structure that makes selected reads faster. The cost is
additional storage and work on every write. Composite indexes depend on column
order and should follow actual filter, join, and sort patterns.

Measure query plans rather than assuming an index is used. Too many indexes can
turn a read optimization into a write bottleneck.

## Transactions and isolation

A transaction groups operations into one logical unit. ACID describes common
goals: atomicity, consistency, isolation, and durability. Isolation levels trade
coordination cost for protection against anomalies such as dirty reads,
non-repeatable reads, and write skew.

Keep transactions short. Holding locks while making remote calls increases
contention and makes failure recovery ambiguous. When a workflow crosses service
boundaries, patterns such as an outbox, saga, or compensating action are often
more practical than a distributed transaction. The
[CDC and Outbox](./database/change-data-capture-outbox.md) chapter explains how
to publish a committed change without an unsafe dual write.

## Replication

Replication copies data to multiple nodes for availability, read capacity, or
geographic proximity. A common leader-based design sends writes to one leader
and replicates changes to followers.

Synchronous replication can provide stronger durability but adds write latency.
Asynchronous replication is faster, but a newly committed write may be missing
from a follower and can be lost during failover. Applications must decide
whether stale reads are acceptable and whether read-after-write consistency is
required.

## Partitioning

Partitioning, or sharding, divides a data set across nodes. A good partition key
spreads both storage and traffic while supporting common queries. A poor key can
create a hot partition even when total capacity is ample.

Range partitioning supports scans but can concentrate sequential writes. Hash
partitioning distributes load more evenly but makes range queries harder.
Rebalancing, cross-partition queries, and globally unique identifiers add
operational complexity.

## Availability and consistency

During a network partition, a distributed data store cannot guarantee both that
every request receives a successful response and that every response observes a
single, immediately consistent value. This is the practical lesson of CAP.
Outside partitions, systems still make latency-versus-consistency trade-offs.

Use the weakest consistency model that preserves product correctness—but define
that correctness with concrete invariants. A social feed can often tolerate
staleness; a balance transfer cannot tolerate creating money.

## Design checklist

- What are the access patterns and invariants?
- Which operations require transactions?
- What is the source of truth?
- How are replicas promoted and repaired?
- What happens during replication lag or a split network?
- How are schema changes, backups, and restores tested?
