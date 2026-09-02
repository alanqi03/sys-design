---
title: DynamoDB
description: A one-page, system-design summary and technical overview of Amazon DynamoDB.
---

# DynamoDB

## The one-page summary

Amazon DynamoDB is a fully managed, serverless NoSQL database for key-value and
document data. It is a strong choice when an application needs predictable
single-digit-millisecond access at large scale and its access patterns are known
in advance.

### Developer Choices

- **Access patterns and table design.** List every required lookup before
  designing the table; DynamoDB schemas are shaped around requests rather than
  normalized entities and ad hoc joins.
- **Primary key.** Choose a high-cardinality partition key that spreads traffic,
  plus an optional sort key that groups related items and supports ordered range
  queries.
- **Indexes.** Add global secondary indexes (GSIs) for alternate partition keys
  or local secondary indexes (LSIs) for alternate ordering within the original
  partition key. Each index adds storage and write cost, and GSI reads are always
  eventually consistent.
- **Data consistency.** Eventually consistent reads are the default and cost
  half as many read units; request strongly consistent reads from a table or LSI
  when the latest committed value is required. GSIs and streams cannot provide
  strongly consistent reads.
- **Transactions and isolation.** Use conditional writes for one-item invariants
  and transactions for all-or-nothing changes across items. Transactions provide
  serializable isolation within one Region, while `Query`, `Scan`, and batch
  operations have weaker operation-wide guarantees.
- **Capacity and geography.** Choose on-demand capacity for variable traffic or
  provisioned capacity for predictable traffic and tighter cost control. Decide
  whether one Region is sufficient or global tables are needed for regional
  latency and resilience.

### Pros

1. **Low latency at scale.** Key-based operations deliver predictable,
   single-digit-millisecond performance when keys distribute traffic well.
2. **Minimal operations.** AWS manages servers, partitioning, replication,
   patching, availability, and capacity expansion.
3. **Built for resilience.** Data is replicated across multiple Availability
   Zones, with backups, point-in-time recovery, and optional global tables.
4. **Useful platform integrations.** Transactions, conditional writes, TTL,
   Streams, Lambda triggers, DAX caching, IAM, and KMS cover common application
   and event-driven patterns.

### Cons

- **Access patterns must be designed up front.** DynamoDB has no joins, and a
  missing access path often requires a new index or data duplication; scans are
  expensive at scale.
- **Hot keys remain a bottleneck.** Automatic partitioning cannot make one
  physical partition exceed its read/write ceiling, so skew can cause throttling
  even when the table has spare capacity.
- **The API has important limits.** Items are capped at 400 KB, transactions are
  regional and bounded, and GSI updates are asynchronous.
- **Cost and lock-in require attention.** Large items, scans, indexes, streams,
  backups, global replication, and poorly shaped requests can become expensive,
  while the data model and APIs are AWS-specific.

### Numbers to Know

- **400 KB** maximum per item, including attribute names.
- **1 read unit** buys one strongly consistent read per second—or two eventually
  consistent reads—for an item up to 4 KB.
- **1 write unit** buys one write per second for an item up to 1 KB.
- **One physical partition** is designed for at most 3,000 read units or 1,000
  write units per second: up to 3,000 strong 4-KB reads, 6,000 eventual reads, or
  1,000 1-KB writes.
- **One transaction** can address at most 100 distinct items and 4 MB in one AWS
  account and Region; each item consumes two underlying reads or writes.
- **Query and Scan responses** return at most 1 MB per page before pagination.

### Scaling Reads

- **Distribute partition keys.** High-cardinality keys let DynamoDB serve reads
  from many physical partitions instead of throttling one hot partition.
- **Choose capacity deliberately.** On-demand mode follows variable traffic;
  provisioned mode plus auto scaling suits steadier workloads.
- **Use the cheapest safe consistency.** Eventually consistent reads double the
  number of 4-KB reads available per read unit.
- **Add purpose-built read paths.** GSIs support alternate queries, while DAX or
  an application cache removes repeated hot-key reads.
- **Serve near users.** Global tables place replicas in multiple Regions, but the
  chosen global consistency mode determines freshness, latency, and write cost.

### Scaling Writes

- **Spread writes across keys.** A high-cardinality partition key is essential;
  add a calculated shard suffix when one logical key, such as a counter, is too
  hot for one partition.
- **Increase table capacity.** On-demand mode expands automatically and
  provisioned mode can use auto scaling, but new peaks and sudden step changes
  still need capacity planning and retry backoff.
- **Reduce write amplification.** Keep items small, update only required
  attributes, batch independent puts/deletes, and create only necessary GSIs.
- **Move reactions to streams.** Commit the source item first, then let DynamoDB
  Streams and consumers update derived views asynchronously.
- **Use global writes carefully.** Global tables allow regional writes, but the
  application must choose eventual conflict resolution or the latency and
  constraints of multi-Region strong consistency.

## Overview

### What ACID Means

DynamoDB provides ACID guarantees through `TransactWriteItems` and
`TransactGetItems` for up to 100 distinct items and 4 MB within one AWS account
and Region:

- **Atomicity:** every action in a transaction succeeds or all of them fail.
- **Consistency:** transactions and condition expressions preserve the rules the
  application encodes, but DynamoDB does not supply relational foreign keys or
  arbitrary database-wide constraints.
- **Isolation:** transactions are serializable relative to transactional calls
  and individual standard item operations. Multi-item `Query`, `Scan`, and batch
  operations provide read-committed behavior as a whole and can observe changes
  made while the operation is running.
- **Durability:** after a successful response, the write is durably persisted
  across the regional service. Transactional ACID boundaries do not extend
  across Regions in a global table.

Transactions perform a prepare and a commit operation for every item, consuming
twice the normal read or write capacity. Prefer a conditional `PutItem`,
`UpdateItem`, or `DeleteItem` when an invariant fits in one item.

### How It Works

#### Requests and partitions

Applications call a regional DynamoDB endpoint over HTTPS using an AWS SDK or
API. DynamoDB validates IAM permissions and uses an internal hash of the
partition-key value to route the request to a physical partition. Items sharing
a partition key form an item collection and, when a sort key exists, are stored
in sort-key order for efficient range access.

A physical partition is an internal allocation of SSD-backed storage and
throughput. DynamoDB creates and splits partitions as data or requested capacity
grows, so developers do not place or rebalance partitions manually. Key design
still matters: traffic concentrated on a few partition-key values can overload
their physical partitions before the table-wide limit is reached.

#### Storage and replication

DynamoDB stores an item as a primary key plus schemaless attributes: scalar,
set, list, or nested document values. The service persists and replicates table
data across multiple Availability Zones in one Region. Applications do not
control buffer pools, pages, compaction, replicas, or storage volumes as they
would with a self-managed database.

#### What happens on a read

1. `GetItem` hashes the complete partition key and retrieves one item. `Query`
   requires an exact partition-key value and may restrict or order results by
   sort key; `Scan` walks every item in a table or index.
2. DynamoDB routes the request to the relevant partition or partitions and reads
   a regional replica according to the requested consistency.
3. An eventually consistent read may return an older committed version. A
   strongly consistent table or LSI read reflects all earlier successful writes;
   GSI and stream reads are always eventual.
4. DynamoDB evaluates key conditions first, then applies filter expressions
   after reading. Filtered-out items still consume read capacity, which is why a
   well-keyed `Query` is preferable to `Scan` plus a filter.
5. The service returns up to 1 MB and a continuation key when more data remains.
   Capacity is charged in 4-KB chunks based on item data read, not only data
   returned.

#### What happens on a write

1. A `PutItem`, `UpdateItem`, or `DeleteItem` is routed by partition key.
   DynamoDB validates the item and evaluates any condition expression.
2. The service durably replicates the accepted change within the Region before
   returning success. A single-item write is atomic.
3. DynamoDB updates LSIs associated with the item and propagates changes to GSIs
   asynchronously. An underprovisioned or hot GSI can throttle writes to its base
   table.
4. If Streams is enabled, the change appears as an ordered record within the
   item's stream sequence for downstream processing. Stream consumers must still
   be idempotent.
5. Write capacity is charged in 1-KB chunks for the base item and again for each
   affected secondary index; transactional writes use two underlying operations.

#### Global replication and recovery

Global tables maintain regional replicas. Multi-Region eventual consistency
(MREC) replicates asynchronously and normally converges quickly, while
multi-Region strong consistency (MRSC) synchronously coordinates writes so
strong reads in replica Regions return the latest item. These modes differ in
latency, supported configurations, failure behavior, and cost.

On-demand backups capture table snapshots. Point-in-time recovery continuously
protects a table for a configurable 1–35 day window and restores into a new
table. TTL can expire items automatically, but deletion is asynchronous and
should not be treated as an exact scheduler.

### Features

- Schemaless key-value and document items with simple or composite primary keys
- `GetItem`, `PutItem`, `UpdateItem`, `DeleteItem`, `Query`, `Scan`, batch APIs,
  and PartiQL support
- Conditional expressions, atomic counters, optimistic concurrency patterns,
  and idempotent transactions
- Global and local secondary indexes, sparse-index patterns, and projections
- On-demand and provisioned capacity modes, auto scaling, adaptive capacity, and
  configurable maximum throughput
- Eventually and strongly consistent reads, ACID transactions, and MREC/MRSC
  global-table modes
- DynamoDB Streams and Kinesis Data Streams change-data capture with Lambda
  integration
- Time to Live, on-demand backups, point-in-time recovery, import/export to S3,
  and table classes
- DynamoDB Accelerator (DAX) for managed in-memory caching
- IAM authorization, resource policies, encryption in transit, and KMS-backed
  encryption at rest
- CloudWatch metrics, Contributor Insights, CloudTrail auditing, and deletion
  protection

### When to choose DynamoDB

Choose DynamoDB when traffic may grow sharply, operations are mostly keyed
lookups or bounded range queries, low and predictable latency matters, and a
managed AWS-native service is desirable. Common fits include sessions, carts,
user preferences, game state, device state, metadata, idempotency records, and
high-scale event-driven applications.

Reconsider it when the application needs joins, flexible ad hoc filters,
database-enforced relationships, large records, frequent full-table analytics,
or portable deployment outside AWS. A common architecture keeps operational
keyed data in DynamoDB while sending changes through Streams to search and
analytical systems.

### Sources and further reading

- [AWS documentation: DynamoDB introduction and capabilities](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
- [AWS documentation: core components](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html)
- [AWS documentation: partitions and data distribution](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.Partitions.html)
- [AWS documentation: read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
- [AWS documentation: transactions and isolation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html)
- [AWS documentation: partition-key best practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html)
- [AWS documentation: constraints](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Constraints.html)
- [AWS documentation: on-demand capacity](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/on-demand-capacity-mode.html)
- [AWS documentation: global secondary indexes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html)
- [AWS documentation: backup and point-in-time recovery](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Point-in-time-recovery.html)

