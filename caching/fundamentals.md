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

## Caching Patterns

| Pattern | How it works | When to choose it | Main trade-off |
| --- | --- | --- | --- |
| **Cache-aside** | The application checks the cache, reads the database on a miss, then puts the result in the cache. Writes usually update the database and invalidate the cached value. | **The default choice when there is no special use case.** It is simple, works with most databases and caches, and lets the application decide what to cache. | Data can remain stale until invalidation or TTL expiry, and simultaneous misses can stampede the database. |
| **Read-through** | The application asks the cache for data; on a miss, the cache or caching library loads the value from the source automatically. | Choose it when many callers should share one standardized loading and refresh policy. | It couples loading logic to the cache layer, and the first request for an uncached value is still slow. |
| **Write-through** | A write goes through the cache layer, which synchronously updates the source of truth before reporting success. The cache is also updated with the new value. | Choose it when **read-after-write freshness or strong consistency is important** and each write can afford database latency—for example, account settings or product configuration. | Cache and database updates are not automatically one atomic transaction; partial failures and concurrent writes still need careful handling. It also spends cache space on values that may never be read. |
| **Write-around** | Writes go directly to the database and bypass or invalidate the cache. A later read fills the cache. | Choose it for write-heavy data that is rarely read immediately, such as bulk imports or archival records. | The first read after a write is a cache miss, and incorrect invalidation can expose an older cached value. |
| **Write-behind** | The cache accepts the write immediately and asynchronously flushes it to durable storage later. | Choose it for **very high write volume where some delay or loss is acceptable**, such as metrics, counters, or aggregated activity signals. | A cache failure can lose acknowledged writes, readers may see different states, and retry, ordering, and recovery logic become harder. Avoid it for payments or critical inventory. |
| **Refresh-ahead** | Popular entries are refreshed shortly before they expire instead of making a user wait for the next miss. | Choose it for predictable hot reads whose source computation is expensive, such as a frequently viewed dashboard. | Refreshes may waste work on entries that nobody reads again, and a failed refresh needs a stale-data policy. |
| **CDN caching** | Static or cacheable HTTP content is copied to edge locations near users. | Choose it for images, videos, JavaScript, CSS, downloads, and public responses that can be reused across users. | Invalidation is distributed and may take time; private or personalized data needs careful cache keys and headers. |
| **In-process caching** | Each service instance keeps a small local copy in its own memory, often in front of a shared cache. | Use it as an optimization layer for **extremely hot keys** that would otherwise hammer one Redis key or network endpoint—for example, small configuration or popular lookup values. | Every instance can hold a different or stale value, memory is duplicated, and entries disappear on restart. Keep TTLs short and retain the shared source of truth. |

“Strong consistency” in the write-through row means callers should not observe
an old cached value after a successful write. Achieving that guarantee still
requires an ordered write protocol, safe retries, and a plan for the case where
the database succeeds but the cache update fails.

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
