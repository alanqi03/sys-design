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
- **Keys and data structures.** Design key names, value sizes, and access
  patterns together; select native structures whose commands perform the work
  atomically instead of repeatedly moving data to the application.
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

### Mental model

Redis is a network server holding a keyspace of typed values primarily in
memory. A client sends commands over a persistent connection using Redis's
request/response protocol. Instead of exposing a general query language, Redis
offers operations shaped around each data type: increment a counter, add a set
member, update a sorted-set score, append to a stream, or set a key only if it
does not exist.

This model is fast because the server usually performs a small, direct operation
without reading a page from disk or planning a query. It also means the
application must design the keys and access paths in advance. Redis cannot
efficiently answer an arbitrary question that was not represented in its key
or index model.

### What happens on a read

1. A client reuses a TCP or TLS connection and sends a command such as `GET`,
   `HGET`, or `ZRANGE`.
2. Redis parses the command, checks authentication and permissions, and locates
   the value in its in-memory keyspace.
3. The command runs against the value's native data structure. Its complexity
   depends on the command and collection size, so an operation can be fast even
   when the data type as a whole is large—or block the shard if it requests
   unbounded work.
4. Redis encodes and returns the result. For small commands, the network round
   trip and serialization can cost more than the lookup itself.

Pipelining sends multiple independent commands before waiting for replies. This
amortizes network and socket overhead, but clients should use bounded batches so
Redis does not accumulate an excessive reply buffer.

### What happens on a write

1. Redis locates or creates the key and applies the command to the in-memory
   data structure.
2. A single command is atomic relative to other commands on that shard. A
   transaction or script can group related work, but long-running work delays
   every client sharing the shard.
3. If AOF is enabled, Redis appends a representation of the write and flushes it
   according to the configured fsync policy. If RDB is enabled, background
   snapshots periodically capture the dataset.
4. The primary propagates the change to replicas asynchronously in the common
   configuration, then returns according to the command and durability settings.

Redis normally processes command work sequentially on a shard, which makes
individual operations easy to reason about but turns slow commands into a
shared latency problem. Modern Redis can use additional threads for parts of
I/O and background work; that does not make an unbounded command harmless.

### Expiration and eviction

A TTL makes a key unavailable after a deadline. Redis removes expired keys both
when they are accessed and through background expiration work. Applications
should treat a TTL as a freshness or retention rule, not as a promise that a
callback will run at an exact millisecond.

`maxmemory` bounds the memory used for the dataset. When Redis crosses that
limit, its policy may reject memory-adding writes or evict keys using an
all-keys or expiring-key variant of LRU, LFU, random, or nearest-expiration
selection. Leave headroom for clients, replication, persistence, fragmentation,
and operating-system memory rather than sizing `maxmemory` to all available RAM.

### Persistence and recovery

Redis provides four broad persistence choices:

- **None:** fastest and appropriate when every value can be reconstructed.
- **RDB snapshots:** compact point-in-time files with fast restart, but changes
  after the latest completed snapshot can be lost.
- **AOF:** a log of writes that Redis replays at startup. Fsync on every write is
  safer and slower; periodic fsync accepts a bounded window of loss.
- **RDB plus AOF:** combines snapshots and a write log at additional resource
  cost.

Persistence protects against process restart; it is not the same as high
availability or a tested backup. Replication protects service continuity,
while off-host backups and restore drills protect against corruption, operator
error, and correlated failure.

### Replication, Sentinel, and Cluster

A primary streams changes to replicas. Replicas can serve stale-tolerant reads
and can be promoted after a failure. Redis Sentinel monitors a non-clustered
deployment, coordinates failover, and helps clients discover the current
primary. Because replication is normally asynchronous, a promoted replica may
not contain the latest acknowledged writes.

Redis Cluster scales the keyspace horizontally. It maps each key to one of
16,384 hash slots and assigns groups of slots to primary nodes. Clients learn
the mapping and follow redirections to the node that owns a key. Replicas add
availability for each primary.

Multi-key commands, transactions, and scripts in Cluster generally require all
involved keys to share one slot. A hash tag—for example, the `{account:42}` part
of several keys—can co-locate related values, but concentrating too much traffic
under one tag creates a hot shard.

### Caching patterns

Redis commonly implements cache-aside: read Redis, fetch the source on a miss,
then populate Redis with a TTL. The application still owns miss handling and
invalidation. Use randomized TTLs, request coalescing, and bounded regeneration
to stop many simultaneous misses from overwhelming the source.

Negative caching can briefly remember that an item does not exist. Versioned
keys make invalidation safer by moving readers to a new namespace while old
entries expire. For correctness-sensitive reads, invalidate after the source
transaction commits or use a change event; do not delete the cache before a
database write that might fail.

### Counters, rate limits, and leaderboards

Atomic increments make Redis useful for approximate metrics, quotas, and rate
limits. Expiring keys support fixed windows; sorted sets or scripts can implement
sliding windows. A rate limiter must define its behavior when Redis is
unavailable: fail open protects availability, while fail closed protects the
limited resource.

Sorted sets store unique members ordered by score. They efficiently support
leaderboards, priority queues, and range-by-score queries. Plan for ties, score
updates, seasonal resets, and whether one globally hot leaderboard needs
partitioning or precomputed views.

### Pub/Sub versus Streams

Redis Pub/Sub broadcasts a message to clients subscribed at that moment. It is
at-most-once and does not retain missed messages, making it appropriate for live
notifications, presence, and cache invalidation where occasional loss is
acceptable or repaired elsewhere.

Redis Streams stores an append-only sequence. Consumer groups distribute work,
track pending messages, and support acknowledgements and replay. This enables
at-least-once processing, so consumers must be idempotent and operators must
manage retention, pending entries, retries, and poison messages. For long
retention, very high fan-out, or a large streaming ecosystem, compare Redis
Streams with a dedicated log platform rather than assuming they are equivalent.

### Distributed locks

A basic Redis lock uses an atomic conditional set with a unique owner token and
an expiration. Release must delete the key only when the stored token still
matches the owner. The expiration prevents a crashed client from holding the
lock forever, but it also means a paused client can continue working after its
lease has expired and another client has acquired it.

Use Redis locks when the protected operation tolerates their timing and failure
model. For correctness-critical or irreversible work, add an authoritative
state check or fencing mechanism and analyze partitions, failover, clock and
pause behavior explicitly. A lock should reduce concurrency; it should not be
the only proof that a business invariant is safe.

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
