---
title: Sharding
description: How to choose a shard key, spread data across shards, and handle hot spots and cross-shard queries.
---

# Sharding

Sharding splits one data set across several independent databases (shards).
Each shard holds part of the data, so you can grow past what one server can
store or write.

Read replicas already spread **reads**, but every write still goes to one
primary. Sharding is how you spread **writes and storage**.

## Running example

A social app stores posts in one PostgreSQL primary. It's running out of disk,
and write load keeps climbing. We shard the `posts` table by `user_id` across 4
shards:

```{mermaid}
flowchart LR
  request["Request<br/>user_id = 1234"] --> router[Router]
  router -->|"hash(1234) → bucket 37<br/>bucket 37 lives on shard 2"| s2[(Shard 2)]
  router -.-> s1[(Shard 1)]
  router -.-> s3[(Shard 3)]
  router -.-> s4[(Shard 4)]
```

All of user 1234's posts live on shard 2, so "show my posts" touches one shard.

## Design decisions

### 1. Make sure you need it

Try the cheaper fixes first: better indexes, caching, read replicas, archiving
old data, or a bigger server. Shard only when you can point to a limit those
can't fix, for example: "Writes are at 80% of disk IOPS and we'll run out of
storage in a year."

### 2. Pick a shard key

The shard key decides which shard owns a row. A good key:

- **Has many values** and spreads traffic evenly (`user_id`, not `country`).
- **Is in most requests**, so the router can go straight to one shard.
- **Keeps related data together**, so common queries hit one shard.
- **Doesn't always increase**. Sharding by `created_at` ranges sends every new
  write to the newest shard.

`user_id` fits the example well. Its weak spot: a celebrity with 50 million
followers puts heavy load on one shard.

### 3. Pick how keys map to shards

| Strategy | How it works | Good for | Watch out for |
| --- | --- | --- | --- |
| Hash | `hash(user_id)` picks the shard | Even spread (the usual default) | Range queries hit every shard |
| Range | `user_id` 1–1M on shard 1, 1M–2M on shard 2... | Range scans | New or popular ranges get hot |
| Directory | A lookup table maps each key to a shard | Moving big tenants to their own shard | The lookup table is a critical dependency |

With hashing, map keys to many small **virtual buckets** (say 1,024), then map
buckets to shards. Adding shard 5 means moving some buckets, not rehashing every
row.

## Challenges

### Hot keys

Even hashing can't split one busy key. If the celebrity's posts are on shard 2,
shard 2 is overloaded while the rest sit idle. Fixes: cache their hot reads, or
give very large accounts their own shard.

### Cross-shard queries

"Top trending posts across all users" must ask all 4 shards and merge the
results. That's slow, and fails if any one shard is down. Instead, compute
trending posts in the background from a stream of events and store the result
in a cache.

### Cross-shard transactions

A transaction within one shard works normally. One that spans shards (user A on
shard 1 follows user B on shard 3) needs a multi-step workflow with retries
instead of a single transaction. Choose a key that keeps your most important
transactions on one shard.

### Unique IDs

Each shard's auto-increment counter starts at 1, so IDs collide. Use UUIDs or a
central ID generator such as Snowflake IDs.

### Moving data

Adding a shard means moving buckets while writes continue: copy the data, catch
up on changes made during the copy, switch the router, then delete the old copy.

## Interview-ready answer

> The primary is running out of write capacity and disk, and read replicas
> don't help with either. I'd shard posts by `user_id` because it has many
> values, appears in almost every request, and keeps a user's posts together.
> I'd hash it into 1,024 virtual buckets spread across 4 shards. Global queries
> like trending posts get expensive, so I'd compute them from an event stream
> and cache the result. To grow, I'd add a shard and move some buckets to it.

## Checklist

- What limit makes sharding necessary now?
- Does the key spread traffic evenly and appear in most requests?
- Which queries and transactions cross shards, and how will you handle them?
- How will you handle hot keys and unique IDs?
- How will you move buckets when you add a shard?
