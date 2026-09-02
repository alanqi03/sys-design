---
title: Caching Fundamentals
---

# Caching Fundamentals

A cache stores a reusable result closer to where it is needed. It can reduce
latency, database load, network transfer, and cost. It also creates another copy
of data whose freshness and failure behavior must be managed.

## Where caches live

Caches appear at several layers:

- browsers cache static assets and responses;
- CDNs cache content near users;
- reverse proxies cache shared HTTP responses;
- applications keep local or distributed values; and
- databases cache data and query pages in memory.

Each layer has a different scope. A process-local cache is extremely fast but is
not shared and disappears on restart. A distributed cache is shared across the
fleet but adds a network hop and another service to operate.

## Common patterns

**Cache-aside** is the most common read pattern. The application checks the
cache, reads the database on a miss, and then fills the cache. It is simple, but
concurrent misses can stampede the database.

**Read-through** puts loading logic behind the cache interface. **Write-through**
updates the cache and backing store together. **Write-behind** buffers writes and
persists them later, improving throughput at the cost of durability risk and
more complex recovery.

## Invalidation and freshness

A time to live (TTL) bounds how long an entry may remain without refresh. A short
TTL improves freshness but increases misses; a long TTL does the reverse.
Explicit invalidation can reduce staleness but is difficult to make perfectly
reliable.

Versioned keys are useful when old entries may safely expire on their own.
Event-driven invalidation can propagate updates quickly, but consumers must
handle duplicates, delay, and missed events.

```{note}
Treat cached data as disposable. If losing the cache would lose the only copy of
important state, it is acting as a database.
```

## Eviction and admission

Finite caches need an eviction policy. LRU favors recently used entries; LFU
favors frequently used entries. Random eviction can be inexpensive and
surprisingly effective. Large, rarely reused objects can pollute a cache, so an
admission policy may reject items that are unlikely to pay for their storage.

## Failure modes

- A **cache stampede** occurs when many callers regenerate the same expired
  value. Request coalescing, early refresh, and randomized TTLs help.
- A **hot key** overloads one shard. Replication or local caching can spread its
  reads.
- A **cold start** shifts full traffic to the backing store after restart or
  flush. Warm critical keys gradually and ensure the source can survive misses.
- A **cache outage** can become a database outage. Use bounded concurrency and
  consider serving slightly stale data where safe.

Evaluate a cache with hit rate, miss latency, eviction rate, memory use, and load
removed from the source—not hit rate alone.

## Design checklist

- What is the source of truth, and can every cached value be reconstructed?
- Which layer should own the cache, and who shares each entry?
- How stale may data become, and how will entries be invalidated?
- What happens when a popular key expires or the whole cache is cold?
- Which eviction policy matches the access pattern?
- Can the backing service survive misses, timeouts, and a cache outage?
