---
title: PostgreSQL
description: A one-page, system-design summary of PostgreSQL.
---

# PostgreSQL

## The one-page summary

PostgreSQL (Postgres) is an open-source relational database. It is the default
choice for data with relationships or strict correctness rules: orders,
accounts, inventory, billing.

### Example

An online store keeps customers, products, and orders in related tables:

```sql
CREATE TABLE products (
  id    BIGINT PRIMARY KEY,
  name  TEXT NOT NULL,
  stock INT  NOT NULL CHECK (stock >= 0)
);

CREATE TABLE orders (
  id          BIGINT PRIMARY KEY,
  customer_id BIGINT NOT NULL REFERENCES customers(id),
  product_id  BIGINT NOT NULL REFERENCES products(id),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- One query answers "what did customer 42 buy?"
SELECT p.name, o.created_at
FROM orders o JOIN products p ON p.id = o.product_id
WHERE o.customer_id = 42;
```

The `CHECK` and `REFERENCES` constraints mean the database itself rejects
negative stock or an order for a missing product.

### Developer Choices

- **Schema.** Tables, column types, keys, and constraints that match the domain.
- **Indexes.** Add them for important lookups and sorts, and check with
  `EXPLAIN`. Each index slows writes, so don't index every column.
- **Primary or replica.** Read fresh or critical data from the primary. Send
  reads that can be slightly stale to replicas.
- **Isolation level.** Controls what a transaction sees while others run.
  - **Read Committed** (default): each statement sees the latest committed data.
    Fine for normal CRUD.
  - **Repeatable Read:** the whole transaction sees one snapshot. Good for
    reports where several queries must agree.
  - **Serializable:** transactions behave as if run one at a time. Use it for
    rules across rows, like "total bookings ≤ room capacity", and retry when
    Postgres reports a conflict.

### Pros

1. **Strong correctness:** ACID transactions, constraints, and foreign keys.
2. **Flexible queries:** joins, aggregates, and a mature query planner.
3. **Many features in one place:** JSONB, full-text search, PostGIS for
   location, and `pgvector` for embeddings.
4. **Huge ecosystem:** managed hosting everywhere, plus mature tools and drivers.

### Cons

- **One primary takes all writes.** Replicas add read capacity, not write
  capacity.
- **Not globally distributed by default.** Writing in several regions at once
  needs extra tooling or sharding.
- **Needs upkeep.** Old row versions must be cleaned up by autovacuum, and each
  connection is a process, so large fleets need a connection pooler such as
  PgBouncer.

### Numbers to Know

On one well-sized primary, expect **thousands of write transactions per second**
and **tens of thousands of simple reads per second**. Treat these as a cue to
benchmark, not as hard limits.

### Scaling Reads

1. Fix slow queries and add the right indexes.
2. Use a bigger machine with more RAM, so more data stays in memory.
3. Cache repeated reads (Redis, CDN).
4. Add read replicas for reads that can be slightly stale.
5. Move analytics to a separate replica or data warehouse.

### Scaling Writes

1. Make writes cheaper: short transactions, batched inserts, fewer indexes.
2. Use a bigger primary.
3. Do side work asynchronously, such as updating search or sending emails,
   through an [outbox](./change-data-capture-outbox.md).
4. Shard across several primaries, as a last resort. See
   [Sharding](./sharding.md).

## Overview

### What ACID means

Using a $50 transfer from account 101 to account 202:

- **Atomicity:** the debit and the credit both happen, or neither does.
- **Consistency:** constraints hold afterward, e.g. no balance goes negative.
- **Isolation:** another transaction never sees the money "in flight."
- **Durability:** once committed, the transfer survives a crash.

(concurrent-writes)=
### Concurrent writes

**Problem:** one item is left in stock. Two buyers both read `stock = 1` and
both buy it. There are four standard fixes.

**1. One atomic statement (best when it fits).** Check and update in one step:

```sql
UPDATE products SET stock = stock - 1
WHERE id = 42 AND stock > 0
RETURNING stock;   -- 0 rows returned = sold out
```

**2. Pessimistic locking.** Lock the row so the second buyer waits:

```sql
BEGIN;
SELECT stock FROM products WHERE id = 42 FOR UPDATE;  -- the other buyer blocks here
UPDATE products SET stock = stock - 1 WHERE id = 42;
COMMIT;
```

Good when conflicts are common. Keep the transaction short.

**3. Optimistic locking.** Don't lock; instead, fail if someone changed the row
since you read it:

```sql
-- You read the row earlier: stock = 8, version = 7.
UPDATE products SET stock = 7, version = version + 1
WHERE id = 42 AND version = 7;   -- 0 rows = someone else won; reload and retry
```

Good when conflicts are rare, or when a user spends time editing a form.

**4. A transaction across several rows.** For the transfer:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 101 AND balance >= 50;
UPDATE accounts SET balance = balance + 50 WHERE id = 202;
INSERT INTO ledger (from_id, to_id, amount) VALUES (101, 202, 50);
COMMIT;   -- roll back instead if the first UPDATE changed 0 rows
```

| Situation | Use |
| --- | --- |
| The change fits in one `UPDATE ... WHERE` | Atomic statement |
| Conflicts are common | Pessimistic lock |
| Conflicts are rare, or a user is editing | Optimistic version check |
| A rule spans several rows | Transaction, possibly `SERIALIZABLE` |

### How it works

- **Storage:** tables are files of 8 KB pages. Postgres keeps frequently used
  pages in memory and reads others from disk as needed.
- **Writes:** a change is first recorded in the **write-ahead log (WAL)**. Once
  the WAL is on disk the commit is durable, even before the table pages are
  written. After a crash, Postgres replays the WAL.
- **MVCC:** an update writes a new row version instead of overwriting the old
  one. Each transaction reads the versions that match its snapshot, so readers
  and writers don't block each other. Autovacuum later removes old versions.
- **Replication:** the primary streams WAL to replicas, which replay it and can
  serve reads. Replicas usually lag slightly.
- **CDC:** the same WAL can be streamed as change events.
  See [CDC](./change-data-capture-outbox.md).

### When to choose PostgreSQL

Choose it when you need transactions, relationships, and flexible queries,
which covers most applications. Look elsewhere only when you need more writes
than one machine can handle, low-latency writes in several continents, or a
specialized workload like full-text search at scale. Even then, Postgres often
stays the source of truth.

### Further reading

- [PostgreSQL documentation: transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL documentation: explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [PostgreSQL documentation: MVCC](https://www.postgresql.org/docs/current/mvcc-intro.html)
- [PostgreSQL documentation: write-ahead logging](https://www.postgresql.org/docs/current/wal-intro.html)
