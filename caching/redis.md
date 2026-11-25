---
title: Redis
description: A system-design guide to Redis for caching, data structures, messaging, coordination, persistence, and scaling.
---

# Redis

## The one-page summary

Redis is an in-memory key-value store. It's best known as a cache, but because
it's fast and its commands are atomic, it's also used for sessions, counters,
rate limits, leaderboards, and lightweight messaging.

### Example

A game uses Redis for three things at once:

```text
# 1. Cache a player's profile for 5 minutes
SET cache:player:42 '{"name":"Ada","level":31}' EX 300

# 2. Count logins without a race condition
INCR stats:logins:2026-11-20

# 3. Keep a live leaderboard
ZADD leaderboard:weekly 9800 player:42
ZADD leaderboard:weekly 10400 player:7
ZREVRANGE leaderboard:weekly 0 9 WITHSCORES   # top 10
```

Each command runs in well under a millisecond and is atomic, so two servers
running `INCR` at the same time never lose a count.

### Developer Choices

- **Cache or database?** A cache holds copies you can rebuild. If Redis holds the
  only copy of something, you need persistence, replicas, and backups.
- **Key names.** Use colon-separated names like `user:42:profile`.
- **Data types.** Pick the type that matches the job (see
  [Data types](#data-types)).
- **TTL and memory limit.** Set `maxmemory` and an eviction policy (usually LRU).
- **Persistence.** None for a pure cache; snapshots or an append-only log if the
  data matters.
- **Topology.** One node → add replicas for failover → Redis Cluster to shard.

### Pros

1. **Very fast:** sub-millisecond operations from memory.
2. **Atomic data structures:** counters, sets, and sorted sets with no race
   conditions.
3. **One tool, many jobs:** cache, sessions, rate limits, leaderboards, queues.
4. **Everywhere:** clients for every language; managed on every cloud.

### Cons

- **Memory is expensive:** the whole data set must fit in RAM.
- **Durability is optional:** depending on settings, a crash can lose recent
  writes.
- **One slow command blocks others:** Redis runs commands one at a time per
  shard, so `KEYS *` or a huge value stalls everyone.
- **Clustering limits multi-key commands:** keys used together must live in the
  same slot.

### Numbers to Know

- **Under 1 ms** per simple command on the server.
- **Tens of thousands of operations/s** per shard, and **hundreds of thousands**
  with pipelining.
- Redis Cluster splits keys into **16,384 hash slots**.

### Scaling Reads

- Keep app servers close to Redis and reuse connections.
- Add replicas for reads that can be slightly stale.
- Shard with Redis Cluster.
- Cache the hottest keys in app-server memory too.

### Scaling Writes

- Pipeline commands to cut network round trips.
- Shard with Redis Cluster.
- Split hot counters (`views:1` ... `views:10`) and sum them when reading.
- Avoid huge values and commands that scan everything.

## Overview

(data-types)=
### Data types

Keys are always strings; values have a type:

| Type | Example | Good for |
| --- | --- | --- |
| String | `SET user:42:name "Ada"` | Cached JSON, counters |
| Hash | `HSET user:42 plan pro region us-west` | Objects with fields |
| List | `LPUSH queue:emails job1` | Simple queues |
| Set | `SADD post:9:likes user:42` | Unique members, "who liked this" |
| Sorted set | `ZADD leaderboard 9800 player:42` | Rankings, priority queues |
| Stream | `XADD orders * id 123 status paid` | Durable event logs |

### Caching with Redis

The cache-aside pattern from [Caching Fundamentals](./fundamentals.md):

```python
key = f"cache:product:{product_id}"
product = redis.get(key)
if product is None:
    product = db.load_product(product_id)
    redis.set(key, serialize(product), ex=300)
return product
```

After updating the product in the database, `DEL cache:product:123`.

### Rate limiting

Allow 100 requests per user per minute:

```text
INCR rate:user:42:2026-11-20T10:15    # returns this minute's count
EXPIRE rate:user:42:2026-11-20T10:15 60
```

If the count is over 100, reject the request.

These are two separate commands. If the server crashes between them, the key
never expires and the user stays blocked. The next section fixes that.

### Lua scripts: several commands as one atomic step

Each Redis command is atomic on its own, but a sequence of commands is not:
another client can run in between. A **Lua script** runs entirely on the Redis
server, and **nothing else runs until it finishes**, so the whole script acts as
one atomic step.

The rate limiter as one script:

```lua
-- KEYS[1] = rate:user:42:2026-11-20T10:15   ARGV[1] = limit   ARGV[2] = window in seconds
local count = redis.call("INCR", KEYS[1])
if count == 1 then
  redis.call("EXPIRE", KEYS[1], ARGV[2])   -- set the expiry on the first request only
end
if count > tonumber(ARGV[1]) then
  return 0                                  -- over the limit: reject
end
return 1                                    -- allowed
```

```python
rate_limit = redis.register_script(LUA_SOURCE)
allowed = rate_limit(keys=["rate:user:42:2026-11-20T10:15"], args=[100, 60])
```

Scripts are also how you safely release a [distributed lock](#distributed-locks),
which needs "delete the key only if it still holds my token":

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
end
return 0
```

Rules of thumb:

- **Keep scripts short.** A slow script blocks every other client on that shard.
- **Pass every key in `KEYS`.** In Redis Cluster, all of a script's keys must be
  in the same hash slot (use `{braces}`).
- **Use `MULTI`/`EXEC`** when you just need to run fixed commands together.
  Use a script when a later step depends on an earlier result, like "only set
  the expiry if the count is 1."

### Pub/Sub vs. Streams

- **Pub/Sub** delivers only to clients connected right now. A disconnected
  client misses messages. Fine for live notifications.

  ```text
  SUBSCRIBE notifications:user:42
  PUBLISH notifications:user:42 '{"type":"order_shipped"}'
  ```

- **Streams** keep messages until they're processed, so workers can crash and
  resume. Use it for jobs that must not be lost.

  ```text
  XADD orders:events * order_id 123 status paid
  XREADGROUP GROUP workers worker-1 STREAMS orders:events >
  XACK orders:events workers <message-id>
  ```

(distributed-locks)=
### Distributed locks

```text
SET lock:order:123 <random-token> NX PX 10000
```

`NX` only sets the key if it doesn't exist, and `PX 10000` auto-releases it
after 10 seconds. To release, delete the key only if it still holds your token,
using the Lua script above.
A paused process can wake up after its lock expired, so don't rely on the lock
alone for anything irreversible like charging a card.

### Persistence

| Option | What it does | Risk |
| --- | --- | --- |
| None | Memory only | Lose everything on restart (fine for a cache) |
| RDB snapshots | Save to disk every few minutes | Lose changes since the last snapshot |
| AOF | Log every write; replay on restart | With `everysec`, lose about 1 second |

### Replication and Cluster

- **Replicas** copy the primary asynchronously. If the primary fails, a replica
  takes over, but it may be missing the last few writes.
- **Redis Cluster** spreads keys across shards by hash slot. To keep related
  keys on one shard (for a multi-key transaction), wrap the shared part in
  braces: `{user:42}:profile` and `{user:42}:settings`.

### When to choose Redis

Choose it for caching, sessions, rate limiting, counters, leaderboards, and
short-lived coordination: anything key-based that needs to be fast and fits in
memory.

Don't make it your only database when you need complex queries, transactions
across many keys, or guaranteed durability, or when the data is much bigger than
your RAM budget.

### Further reading

- [Redis data types](https://redis.io/docs/latest/develop/data-types/)
- [Redis key eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [Redis persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Scaling with Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)
- [Distributed locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
