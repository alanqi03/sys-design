---
title: DynamoDB
description: A one-page, system-design summary and technical overview of Amazon DynamoDB.
---

# DynamoDB

## The one-page summary

Amazon DynamoDB is a fully managed NoSQL key-value database. You don't run
servers; you get fast lookups at almost any scale, **as long as you design the
table around the queries you need ahead of time**.

### Example

An e-commerce app needs two queries: "get a user's profile" and "list a user's
recent orders." One table can answer both:

| Partition key (`PK`) | Sort key (`SK`) | Other attributes |
| --- | --- | --- |
| `USER#42` | `PROFILE` | name: "Ana", email: "ana@example.com" |
| `USER#42` | `ORDER#2026-11-02#9001` | total: 59.99, status: "shipped" |
| `USER#42` | `ORDER#2026-11-18#9017` | total: 12.50, status: "placed" |
| `USER#77` | `PROFILE` | name: "Raj" |

- **Get profile:** `GetItem(PK="USER#42", SK="PROFILE")`
- **Recent orders:** `Query(PK="USER#42", SK begins_with "ORDER#")`, newest
  first

The partition key decides which machine stores the item. The sort key keeps a
user's items in order, so "recent orders" is one fast range read. There are no
joins, so "all orders shipped today across all users" needs a separate index.

### Developer Choices

- **Access patterns first.** List every query before designing the table.
- **Partition key.** Pick one with many values that spreads traffic (`user_id`).
- **Sort key.** Optional; groups and orders related items (`ORDER#<date>`).
- **Secondary indexes.** A **GSI** lets you query by a different key, such as
  orders by `status`. GSIs update asynchronously and cost extra writes.
- **Consistency.** Reads are eventually consistent by default and cost half as
  much. Ask for strongly consistent reads when you must see the latest write.
- **Capacity mode.** On-demand for unpredictable traffic; provisioned for steady,
  cheaper traffic.

### Pros

1. **Fast at any scale:** single-digit-millisecond key lookups.
2. **No operations:** AWS handles servers, partitioning, replication, and
   patching.
3. **Resilient:** data is copied across three Availability Zones; global tables
   add other regions.
4. **AWS integrations:** TTL expiry, Streams to Lambda, transactions.

### Cons

- **Unplanned queries are hard.** No joins or ad hoc filters; scanning the whole
  table is slow and expensive.
- **Hot keys throttle.** One very busy partition key can be throttled even when
  the table has spare capacity.
- **Limits:** items are capped at 400 KB, and transactions cover at most 100
  items.
- **AWS lock-in,** and costs grow with scans, indexes, and large items.

### Numbers to Know

- **400 KB** max item size.
- **1 read unit** = 1 strongly consistent read/s (or 2 eventually consistent) of
  up to 4 KB.
- **1 write unit** = 1 write/s of up to 1 KB.
- **One partition** handles at most **3,000 reads/s or 1,000 writes/s**.
- **Transactions:** up to **100 items**, at double the cost.
- **Query/Scan** returns at most **1 MB** per page.

### Scaling Reads

- Use partition keys with many values so reads spread across partitions.
- Use eventually consistent reads when you can (twice the reads for the price).
- Add a GSI for each new query pattern.
- Cache hot items with DAX or Redis.

### Scaling Writes

- Spread writes across many partition keys.
- For one very hot key, like a global counter, split it into `counter#1` ...
  `counter#10` and add the shards up when reading.
- Keep items small and create only the GSIs you need.
- Push follow-up work to DynamoDB Streams instead of doing it inline.

## Overview

### Avoiding race conditions

**Conditional write:** only reserve a seat if it's still free:

```text
UpdateItem
  Key:       PK="FLIGHT#UA100", SK="SEAT#12A"
  Update:    SET holder = "USER#42"
  Condition: attribute_not_exists(holder)   -- fails if someone took it first
```

**Transaction:** for all-or-nothing changes across items, such as charging a
wallet and creating an order, use `TransactWriteItems`. It costs twice as much,
so prefer a conditional write when one item is enough.

### How it works

- **Partitions:** DynamoDB hashes the partition key to pick a physical
  partition, and splits partitions automatically as data and traffic grow.
- **Reads:** `GetItem` fetches one item by full key. `Query` reads one partition
  key's items in sort-key order. `Scan` reads the whole table; avoid it on hot
  paths. Filters run *after* the read, so you pay for filtered-out items too.
- **Writes:** each write is stored durably in three Availability Zones before
  success is returned. GSIs and Streams update shortly after.
- **Global tables:** copy the table to other regions for local reads and writes.
- **TTL:** set an expiry timestamp and DynamoDB deletes the item later, usually
  within a few days, so don't use it as an exact timer.

### When to choose DynamoDB

Choose it for keyed lookups at high scale with little operations work: sessions,
shopping carts, user settings, game state, device state, and idempotency keys.

Avoid it when you need joins, flexible queries, large records, or heavy
analytics. A common pattern keeps hot data in DynamoDB and streams changes to
Elasticsearch or a warehouse for search and analytics.

### Further reading

- [AWS documentation: core components](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html)
- [AWS documentation: partition-key best practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-design.html)
- [AWS documentation: read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
- [AWS documentation: transactions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/transaction-apis.html)
