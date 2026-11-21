---
title: Databases
---

# Databases

A database stores your data durably and lets you query it. Pick one by asking
what queries you need and which rules must never break, not by brand name.

This section covers general principles, then
[CDC and the transactional outbox](./database/change-data-capture-outbox.md),
[sharding](./database/sharding.md), and four databases.
[Redis](./caching/redis.md) can also be a database, but it lives under Caching
because that is its most common role.

## Database comparison

| Database | Best at | Weak at | Example use |
| --- | --- | --- | --- |
| [PostgreSQL](./database/postgresql.md) (relational) | Transactions, joins, constraints, flexible SQL | Scaling writes past one primary | Orders, accounts, inventory |
| [DynamoDB](./database/dynamodb.md) (managed key-value) | Fast key lookups at any scale, no servers to run | Queries you didn't plan for; hot keys | Shopping carts, sessions |
| [Cassandra](./database/cassandra.md) (wide-column) | Huge write volume across nodes and regions | Joins, ad hoc queries, operations work | Chat messages, IoT sensor data |
| [Elasticsearch](./database/elasticsearch.md) (search index) | Full-text search, filters, aggregations | Being the source of truth | Product search, log search |

Start with PostgreSQL. Add another store only when a specific workload needs it.

## Model the workload

Before choosing, write down the main queries and how often they run.

**Example:** a food delivery app.

| Operation | Frequency | Needs |
| --- | --- | --- |
| Place an order (charge card, reserve items) | 100/s | A transaction across several rows |
| Show a restaurant's menu | 10,000/s | Fast reads; a few seconds stale is fine |
| Search "spicy noodles near me" | 2,000/s | Full-text and location search |

Orders need transactions, so they go in PostgreSQL. Menus are read-heavy, so add
a cache. Search needs Elasticsearch, fed from PostgreSQL.

## Indexes

An index is a sorted lookup structure that makes some reads fast, at the cost of
extra storage and slower writes.

```sql
-- Without an index, this scans every order.
SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC;

-- This index matches the filter, then the sort.
CREATE INDEX ON orders (customer_id, created_at);
```

Column order matters: this index helps queries filtering by `customer_id`, but
not queries filtering only by `created_at`. Check with `EXPLAIN` that the index
is actually used.

## Transactions

A transaction makes several changes all-or-nothing. Moving $50 from Alice to
Bob debits one account and credits the other; a transaction ensures you never
see only half.

Keep transactions short, and never make network calls while one is open. To
notify other services after a commit, use the
[transactional outbox](./database/change-data-capture-outbox.md).

## Replication

Replication keeps copies of the data on several machines. Usually one
**leader** takes writes and **followers** copy them and serve reads.

Copying is usually asynchronous, so a follower can lag. **Example:** a user
updates their profile photo, the next page load reads from a lagging follower,
and the old photo appears. Fix it by reading the user's own data from the leader
right after they write it.

## Partitioning (sharding)

Sharding splits data across machines by a key, such as `user_id`, so no single
machine holds everything or takes every write. A bad key creates a hot spot: one
machine gets most of the traffic. See [Sharding](./database/sharding.md).

## Consistency vs. availability (CAP)

If the network splits a distributed database, each side must either reject
requests (stay consistent) or answer with possibly stale data (stay available).

Choose per feature. A like count can be a little stale. A bank balance can't.

## Design checklist

- What are the main queries, and which rules must never break?
- Which operations need transactions?
- Which database is the source of truth?
- What happens when a replica lags or a machine fails?
- How are backups restored and schema changes rolled out?
