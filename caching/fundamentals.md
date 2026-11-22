---
title: Caching Fundamentals
---

# Caching Fundamentals

A cache stores a copy of data closer to where it's needed, so reads are faster
and the source does less work.

**Running example:** an online store's product page. It's viewed 10,000 times
per second, but a product's price changes maybe once a day. Reading it from the
database on every view is wasted work.

## Where caches live

| Layer | Example | Shared by |
| --- | --- | --- |
| Browser | The product image, cached for a day | One user |
| CDN | Product images and JS served from a city near the user | All users in a region |
| In-process | A small map in each app server's memory | One server |
| Distributed cache | Redis holding `product:123` | Every app server |
| Database | Postgres keeping hot pages in RAM | Automatic |

## Caching patterns

### Cache-aside (the default)

The app checks the cache first. On a miss, it reads the database and stores the
result in the cache.

```python
product = cache.get("product:123")
if product is None:                       # miss
    product = db.load_product(123)
    cache.set("product:123", product, ttl=300)
return product
```

When the price changes, update the database, then **delete** `product:123`. The
next read reloads it.

### Other patterns

| Pattern | How it works | Use it for |
| --- | --- | --- |
| **Read-through** | Like cache-aside, but the cache library loads misses for you | Sharing one loading policy across many callers |
| **Write-through** | Every write updates the database and the cache together | Data that must be fresh right after a write, like account settings |
| **Write-around** | Writes go to the database only; the cache fills on the next read | Data written often but rarely read right away, like bulk imports |
| **Write-behind** | Writes go to the cache now and reach the database later | Very high-volume data where losing a little is OK, like view counts |
| **Refresh-ahead** | Popular entries are reloaded just before they expire | Expensive, predictable reads, like a busy dashboard |

Avoid write-behind for anything important, such as payments or inventory. If
the cache crashes before flushing, those writes are gone.

## Keeping data fresh

- **TTL (time to live):** the entry expires after a set time. With a 5-minute
  TTL, a price change shows up within 5 minutes at most. Shorter TTLs are
  fresher but cause more misses.
- **Invalidate on write:** delete the key when the data changes, so the change
  shows up almost immediately. Keep a TTL as a backup in case a delete is
  missed.

Choose per feature how stale is acceptable:

| Data | Stale is OK? | Approach |
| --- | --- | --- |
| Product description, like counts, feeds | Yes, for seconds to minutes | Cache-aside with TTL and invalidation |
| Account balance, stock at checkout | No | Skip the cache; read the database |

```{note}
Treat cached data as disposable. If losing the cache would lose data, it's
being used as a database.
```

## Eviction

When the cache is full, it must drop something:

- **LRU** (least recently used) drops what hasn't been read in the longest time.
  The usual default.
- **LFU** (least frequently used) drops what's read least often. Better when a
  few items are always popular.

## What goes wrong

- **Stampede:** the product page's cache entry expires, and 10,000 requests miss
  at once and all hit the database. Fixes: let one request reload while the
  others wait, add random jitter to TTLs, or refresh popular keys early.
- **Hot key:** one key gets so much traffic it overloads its cache server. Fix:
  keep a short-lived copy in each app server's memory.
- **Cold start:** after a restart the cache is empty and all traffic hits the
  database. Fix: preload the most popular keys and make sure the database can
  survive the misses.
- **Cache outage:** if the cache goes down, the full load lands on the database
  and can take it down too. Fix: limit concurrent database queries and fail
  gracefully.

Judge a cache by hit rate, latency on a miss, and how much load it removes from
the database.

## Checklist

- What's the source of truth, and can every cached value be rebuilt from it?
- How stale can each kind of data be?
- How are entries invalidated when data changes?
- What happens when a popular key expires or the cache restarts empty?
- Can the database survive if the cache goes down?
