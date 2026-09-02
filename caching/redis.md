---
title: Redis
description: A system-design guide to Redis for caching, data structures, messaging, coordination, persistence, and scaling.
---

# Redis

## The one-page summary

Redis is an in-memory data structure server. Applications address values by key
and use specialized operations for strings, hashes, lists, sets, sorted sets,
streams, and other structures. It is best known as a distributed cache, but its
low latency and atomic commands also make it useful for sessions, counters,
rate limits, leaderboards, queues, messaging, and coordination.

### Developer Choices

- **Role and source of truth.** Decide whether Redis holds disposable cached
  copies, rebuildable derived state, or authoritative data. The last choice
  requires deliberate persistence, replication, backup, and recovery plans.
- **Keys and data structures.** Every Redis key is a string (technically a
  binary-safe byte string). Use colon-separated names such as
  `user:42:profile`, then choose a value type whose commands match the access
  pattern.
- **TTL and eviction.** Decide which keys expire, how TTLs are randomized, the
  `maxmemory` headroom, and whether LRU, LFU, TTL-based, random, or no eviction
  matches the workload.
- **Durability.** Choose no persistence for a disposable cache, RDB snapshots
  for periodic recovery points, AOF for a replayable write history, or both.
  More durability consumes I/O and can increase latency.
- **Topology and consistency.** A single node is simplest; replicas add read
  capacity and failover, while Redis Cluster shards data. Replication is
  normally asynchronous, so stale reads and acknowledged-write loss during
  some failures must be acceptable or mitigated.
- **Messaging model.** Choose Pub/Sub only for live, fire-and-forget broadcast.
  Choose Streams when messages need persistence, replay, consumer groups, or
  acknowledgements.

### Pros

1. **Very low latency.** Redis serves efficient operations from memory, making
   sub-millisecond server latency realistic on a healthy, nearby deployment.
2. **Useful atomic data structures.** Counters, hashes, sets, sorted sets,
   expiration, transactions, and scripts can replace application-side
   read-modify-write races.
3. **Broad set of patterns.** The same operational platform can support caches,
   sessions, rate limits, leaderboards, transient messaging, and small durable
   streams.
4. **Mature ecosystem.** Redis has widely used clients, managed offerings,
   observability tools, replication, Sentinel, and clustering support.

### Cons

- **Memory is expensive and finite.** Values, keys, metadata, replication, and
  persistence buffers all consume RAM; an incorrect limit or eviction policy
  can reject writes or discard useful data.
- **Durability is a choice, not an automatic database guarantee.** Snapshots can
  lose changes since the last save, AOF policy trades safety for latency, and
  asynchronous failover can lose acknowledged writes.
- **Slow commands affect other clients.** Large values, unbounded collection
  operations, scripts, or expensive scans can stall a shard and create tail
  latency.
- **Horizontal scaling changes the programming model.** Redis Cluster constrains
  multi-key operations to keys in the same hash slot and adds redirection,
  rebalancing, failover, and hot-shard concerns.
- **Some features have deliberately weak semantics.** Pub/Sub drops messages
  for disconnected consumers, and a distributed lock is a time-bounded lease
  whose failure assumptions must match the protected operation.

### Numbers to Know

For an early system-design estimate, think **sub-millisecond server processing
for small, efficient operations** and **tens of thousands of ordinary networked
operations per second on one shard**. Pipelining and small values can raise
throughput into the hundreds of thousands or more, but value size, command
complexity, network RTT, persistence, and hot keys matter more than a headline
benchmark.

### Scaling Reads

- **Improve the hit path first.** Keep clients close to Redis, reuse connections,
  pipeline independent commands, avoid oversized values, and select commands
  with bounded complexity.
- **Use local caching for the hottest values.** A small process-local layer can
  remove network hops and spread hot-key reads, at the cost of another freshness
  boundary.
- **Add replicas for stale-tolerant reads.** Replicas can serve read traffic, but
  asynchronous replication means they may lag and cannot guarantee immediate
  read-after-write consistency.
- **Shard the keyspace.** Redis Cluster distributes hash slots among primaries,
  increasing aggregate memory and throughput when keys and traffic distribute
  evenly.
- **Treat hot keys separately.** Replicate, locally cache, split, or precompute a
  key whose traffic cannot be spread by ordinary sharding.

### Scaling Writes

- **Batch network work.** Pipeline independent commands and combine related
  changes with an atomic command, transaction, or carefully bounded script.
- **Keep operations small.** Avoid huge values, unbounded deletes, full
  collection scans, and scripts whose runtime blocks the shard.
- **Shard writes with Redis Cluster.** Choose keys that spread across the 16,384
  hash slots; use hash tags only when related multi-key operations must remain
  together.
- **Remove hot write keys.** Partition counters or workloads when exact global
  results can be merged, or serialize deliberately when a single ordered value
  is the requirement.
- **Measure durability cost.** AOF fsync policy, replicas, and backups protect
  data but consume CPU, memory, storage, and network capacity.

### Notes

- A pipeline reduces round trips but is **not** a transaction. Other clients may
  run commands between pipelined operations.
- `MULTI`/`EXEC` serializes a queued group of commands. `WATCH` adds optimistic
  locking by aborting the transaction if a watched key changed first.
- Keep disposable caches and durable application state in separate Redis
  deployments when possible. They usually need different eviction, persistence,
  backup, and failure policies.
- Use Pub/Sub for current listeners and Streams for retained messages. They are
  different delivery models, not interchangeable APIs.

## Overview

### Keys and values

Every Redis key is a string. Keys are usually readable text, although Redis
treats them as binary-safe byte strings. Redis has no built-in namespaces, so
applications conventionally separate key parts with colons:

```text
user:42:profile
user:42:settings
session:7f3a9c
cache:product:123
rate_limit:user:42:2026-09-01T10:15
leaderboard:2026:weekly
```

Read a key from general to specific: `cache:product:123` means the cached
representation of product `123`. Keep the scheme stable and documented; key
names consume memory too. In Redis Cluster, braces can deliberately place
related keys in one hash slot: `{user:42}:profile` and `{user:42}:settings`.

The key is always a string, but its **value** has a Redis type. Match the type to
the operation instead of serializing everything into one blob:

```text
SET  user:42:name "Ada"                         # string
HSET user:42:profile plan pro region us-west   # hash
SADD user:42:roles editor reviewer             # set
ZADD leaderboard:weekly 9800 user:42           # sorted set
XADD orders:events * order_id 123 status paid  # stream
```

### Reads and writes

A read locates the key in memory and runs a type-specific command. A write
changes the in-memory value, optionally records it for persistence, and sends it
to replicas. A single command is atomic on its shard.

```text
SET cache:product:123 '{"name":"Keyboard","price":99}' EX 300
GET cache:product:123
INCR product:123:view_count
HGET user:42:profile region
```

For small commands, network round-trip time can exceed Redis processing time.
Pipelining batches independent commands into fewer round trips; `MULTI`/`EXEC`
is different—it prevents other clients' commands from running in the middle of
the transaction. Keep commands, pipelines, transactions, and scripts bounded:
one expensive operation can delay every client using that shard.

### Expiration and eviction

A TTL makes a key unavailable after a deadline:

```text
SET session:7f3a9c '{"user_id":42}' EX 1800
TTL session:7f3a9c
EXPIRE session:7f3a9c 3600
```

Redis expires keys on access and with background work, so use TTLs for freshness
or retention—not as an exact scheduler. When `maxmemory` is reached, the chosen
policy either rejects memory-adding writes or evicts keys using LRU, LFU,
random, or TTL-based selection. Leave RAM for replication, persistence, client
buffers, and fragmentation instead of assigning all machine memory to data.

### Persistence and recovery

Redis provides four practical choices:

- **None:** fastest and appropriate when every value can be reconstructed.
- **RDB snapshots:** periodic point-in-time files; recent changes can be lost.
- **AOF:** a log of writes that Redis replays at startup. Fsync on every write is
  safer and slower; periodic fsync accepts a bounded window of loss.
- **RDB plus AOF:** combines both at additional resource cost.

For example, this enables AOF and asks the operating system to flush it about
once per second:

```text
appendonly yes
appendfsync everysec
```

Persistence protects against process restart; it is not the same as high
availability or backup. Use replicas for service continuity and off-host,
restore-tested backups for operator error or correlated failure.

### Replication, Sentinel, and Cluster

A primary asynchronously sends writes to replicas. Replicas can serve stale
reads or be promoted after failure; Sentinel coordinates this failover for a
non-clustered deployment. A promoted replica can be missing the latest
acknowledged writes.

Redis Cluster partitions keys across 16,384 hash slots. Ordinarily these keys
may land on different shards:

```text
user:42:profile
user:42:settings
```

Braces create a **hash tag**, forcing the enclosed portion to determine the
slot. This lets a transaction address both keys:

```text
{user:42}:profile
{user:42}:settings
```

Use hash tags only for operations that truly need co-location; overusing one tag
concentrates traffic on a single shard.

### Caching patterns

Cache-aside keeps the database authoritative:

```python
key = f"cache:product:{product_id}"
product = redis.get(key)

if product is None:
    product = database.load_product(product_id)
    redis.set(key, serialize(product), ex=300)

return product
```

Add random TTL jitter and coalesce concurrent misses so one expired hot key does
not stampede the database. After changing the product, commit the database
transaction first and then delete `cache:product:123` or publish an invalidation
event. A short-lived negative entry can also prevent repeated lookups for a
missing product.

### Counters, rate limits, and leaderboards

Atomic increments provide counters without a read-modify-write race:

```text
INCR article:99:view_count
INCRBY inventory:sku-123:reserved 4
```

An expiring counter can implement a basic fixed-window rate limit; a script can
atomically combine increment and expiration. Decide whether a Redis outage
should fail open or reject requests.

Sorted sets make a leaderboard direct:

```text
ZADD leaderboard:2026:weekly 9800 user:42
ZADD leaderboard:2026:weekly 10400 user:7
ZREVRANGE leaderboard:2026:weekly 0 9 WITHSCORES
ZREVRANK leaderboard:2026:weekly user:42
```

Plan for ties, resets, and a globally hot leaderboard that may need cached or
partitioned views.

### Pub/Sub versus Streams

Pub/Sub sends only to clients connected at that moment:

```text
SUBSCRIBE notifications:user:42
PUBLISH notifications:user:42 '{"type":"order_shipped"}'
```

It is at-most-once: disconnecting loses messages. Streams retain entries and
support consumer groups, acknowledgements, and replay:

```text
XGROUP CREATE orders:events workers 0 MKSTREAM
XADD orders:events * order_id 123 status paid
XREADGROUP GROUP workers worker-1 COUNT 10 STREAMS orders:events >
XACK orders:events workers 1712345678901-0
```

Streams enable at-least-once processing, so consumers must handle duplicates and
operators must manage retention, retries, and pending entries. Compare a
dedicated log platform when retention, replay, or fan-out becomes large.

### Distributed locks

A lock is a unique token stored only if the key does not already exist:

```text
SET lock:order:123 550e8400-e29b-41d4-a716-446655440000 NX PX 10000
```

`NX` acquires only when absent; `PX 10000` makes the lease expire after ten
seconds. Release must conditionally delete the key only if its value still
matches this owner's token—never use an unconditional `DEL`.

A paused client can resume after its lease expired and another client acquired
the lock. For irreversible work, pair the lock with an authoritative state
check or fencing mechanism; the lock alone is not proof of correctness.

### Features

- Strings, hashes, lists, sets, sorted sets, streams, bitmaps, bitfields, and
  geospatial indexes
- JSON, probabilistic structures, time series, search, and vector capabilities
  in current Redis distributions
- Per-key expiration and configurable memory eviction policies
- Atomic commands, `MULTI`/`EXEC`, optimistic locking with `WATCH`, and
  server-side scripting/functions
- Pipelining for batching commands without a network round trip per operation
- Pub/Sub channels and durable Streams with consumer groups
- RDB snapshots, AOF persistence, replication, backups, and restore tooling
- Sentinel-based monitoring and failover for non-clustered deployments
- Redis Cluster for sharding and per-shard replication
- ACLs, TLS, client limits, command statistics, latency monitoring, and slow logs

### When to choose Redis

Choose Redis when the workload benefits from very low latency, known key-based
access patterns, atomic data-structure operations, and a working set that fits
the memory budget. It is especially effective as a cache or as rebuildable
operational state beside a durable database.

Reconsider Redis as the sole source of truth when the system requires arbitrary
queries, relational constraints, multi-record transactions across shards,
durability through complex failures, or a dataset much larger than the
affordable memory tier. Reconsider Pub/Sub when messages must survive a
disconnect, and reconsider a Redis lock when violating mutual exclusion could
cause irreversible harm.

### Sources and further reading

- [Redis keys and values](https://redis.io/docs/latest/develop/using-commands/keyspace/)
- [Redis data types](https://redis.io/docs/latest/develop/data-types/)
- [Redis key eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)
- [Scaling with Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)
- [Redis pipelining](https://redis.io/docs/latest/develop/using-commands/pipelining/)
- [Redis transactions](https://redis.io/docs/latest/develop/using-commands/transactions/)
- [Redis Pub/Sub](https://redis.io/docs/latest/develop/pubsub/)
- [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/)
- [Distributed locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [Redis benchmark guidance](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/)
