---
title: Sharding
description: How to choose a shard key and distribution strategy, plan for growth, and handle hot spots, cross-shard work, and consistency.
---

# Sharding

Sharding splits one logical data set across multiple independent databases.
Each shard owns only part of the data, so storage and traffic can grow beyond
what one database server can handle.

Sharding is mainly a **write and storage scaling technique**. Read replicas can
spread reads, but every write still reaches the primary and the full data set
still has to fit there. Sharding spreads writes and storage across several
primaries, at the cost of making queries and transactions across shards harder.

```{mermaid}
flowchart LR
  request[Request with user_id] --> router[Shard router]
  router --> hash[Hash user_id]
  hash --> map[Shard map]
  map --> s1[(Shard 1)]
  map --> s2[(Shard 2)]
  map --> s3[(Shard 3)]
```

## Summary: Design Decisions

### 1. Identify the bottleneck and justify sharding

Start with measurements: database CPU, write IOPS, storage, connection count,
lock contention, throughput, and p95 latency. Explain why simpler changes—better
indexes, query tuning, caching, read replicas, archiving, or a larger server—will
not provide enough headroom.

For example: “The primary is limited by write IOPS and will exceed its storage
capacity within a year. Read replicas help reads but cannot spread writes, so I
would shard the data across multiple database primaries.”

Do not shard merely because the system may become large someday. It adds
application and operational complexity that is difficult to reverse.

### 2. Propose a shard key

The shard key decides which shard owns a record. A good key:

- Has many possible values and distributes both data volume and traffic evenly.
- Appears in most requests, so the router can find one shard without searching
  all of them.
- Keeps data that is commonly read or updated together on the same shard.
- Does not steadily increase in a way that sends all new writes to one place.

For a social application, `user_id` may be a reasonable key: a user's profile
and posts can live together, and most requests already know the user. It is not
perfect—a celebrity can become a hot key, and queries across all users become
cross-shard operations.

### 3. Choose a distribution strategy

**Hash-based sharding** is the common default when even distribution matters.
Hash the shard key, map the result to a logical bucket, and map that bucket to a
physical shard.

Use a consistent-hash ring or many **virtual buckets** so adding a shard moves
only a fraction of the key space instead of remapping every key. Virtual buckets
are often easier to operate because their placement can be changed explicitly.

| Strategy | Choose it when | Main trade-off |
| --- | --- | --- |
| Hash-based | Even distribution and point lookups matter most | Range scans and global ordering become difficult |
| Range-based | Nearby values must support efficient range scans | Sequential or popular ranges can become hot |
| Directory-based | Tenants need custom placement or isolation | The shard map becomes a critical dependency |

### 4. Call out the trade-offs

Global queries become expensive. If the product needs “trending posts across
all users,” it may have to query every shard and combine the results. That
scatter-gather request is slow, expensive, and only as reliable as all of the
shards involved.

Instead, publish post and engagement events and let a background job or stream
processor continuously calculate trending content. Store the resulting view in
a cache or dedicated read store so requests do not query every shard.

Cross-shard joins, transactions, unique constraints, sorting, and pagination
also require additional design. Prefer a key that keeps the most important
transactional operations within one shard.

### 5. Plan for growth

Consistent hashing or virtual buckets make it easier to add shards later. If
the system needs more capacity, add a shard and reassign only some buckets or
hash ranges; only the data in those ranges moves.

This reduces movement but does not make migration automatic. A safe online
rebalance generally copies existing rows, captures writes that happen during
the copy, switches routing to the new owner, verifies the result, and removes
the old copy only after a rollback window.

## Challenges of Sharding

### Hot spots and hot keys

An apparently even hash distribution does not guarantee even traffic. One large
tenant, celebrity account, viral post, or unusually active time range can
overload a single shard while the others remain mostly idle.

Monitor load per shard and per key. Common responses include caching hot reads,
isolating a very large tenant, splitting a hot entity across several subkeys, or
changing bucket placement. Adding a random suffix can spread writes, but it also
forces reads to search and combine several keys, so use it only for access
patterns that can afford that cost.

### Cross-shard queries

Joins, aggregates, search, global sorting, and pagination may need a
scatter-gather query across every shard. Latency is determined by the slowest
shard, and one unavailable shard can make the result incomplete.

Avoid these requests on the user-facing path when possible. Denormalize small
pieces of data, pre-compute global views, maintain a search index, or send data
to an analytics store designed for distributed aggregation.

### Cross-shard writes and consistency

A transaction within one shard can use the database's normal ACID guarantees.
A transaction across shards needs coordination such as two-phase commit, which
adds latency and can reduce availability, or an asynchronous workflow such as a
saga, which temporarily exposes partial progress.

Design important invariants around one shard key whenever possible. For
workflows that must cross shards, use idempotent steps, an outbox or durable
queue, retries, and compensating actions. Decide explicitly whether temporary
eventual consistency is acceptable.

Global uniqueness is another cross-shard problem. Generate globally unique IDs
with UUIDs or a distributed ID service, or include the shard key in the unique
value. A constraint enforced on one shard cannot detect a duplicate on another.

### Routing and stale shard maps

Every request needs a reliable answer to “which shard owns this key?” The shard
map may live in a routing proxy, a client library backed by configuration, or a
metadata service. It must be highly available, versioned, and updated safely;
stale routers can send requests to the old owner during a migration.

During a move, use an explicit state such as `COPYING`, `DUAL_WRITE`, and
`MOVED`, or have the old shard forward requests to the new owner. Make retries
idempotent so a timeout during the transition cannot apply a write twice.

### Rebalancing and operations

More databases mean more schema migrations, backups, restores, failovers,
connection pools, and dashboards. Roll out schema changes across shards in a
backward-compatible sequence, and test that backups can restore the complete
logical data set—not just one shard.

Track capacity and lag per shard rather than relying only on fleet averages. A
healthy average can hide one shard that is nearly full or consistently slow.

## Interview-Ready Example

> The single primary is approaching its write-IOPS and storage limits, and read
> replicas cannot solve either problem. I would shard posts by `user_id` because
> it has high cardinality, appears in the common read and write paths, and keeps
> one user's data together. I would hash the key into many virtual buckets and
> map those buckets to database shards. The trade-off is that a global query
> such as trending posts becomes expensive, so I would pre-compute it from an
> event stream and cache the result. To grow, I would add shards and move only a
> subset of virtual buckets with an online copy, change capture, verified
> cutover, and rollback window.

## Design Checklist

- What measured bottleneck makes sharding necessary now?
- Which key distributes bytes and requests—not just row counts—evenly?
- Can the router identify one shard for the most important operations?
- Which queries and transactions will cross shards, and how will they change?
- How are hot keys detected and split or isolated?
- How are global views and uniqueness handled?
- How are buckets moved while writes continue?
- How are schema changes, backups, and restores coordinated across every shard?
