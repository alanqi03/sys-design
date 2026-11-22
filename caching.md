---
title: Caching
---

# Caching

A cache keeps a copy of data somewhere faster than its source. Reading a product
from Redis takes about 1 ms; reading it from the database might take 20 ms.
The cost is a second copy of the data that can go stale.

## In this section

- [Caching Fundamentals](./caching/fundamentals.md): where caches live, how to
  read and write through them, keeping them fresh, and what goes wrong.
- [Redis](./caching/redis.md): the most common cache, which also handles
  counters, leaderboards, rate limits, and messaging.

Whatever cache you use, decide up front: what's the source of truth, how stale
can the data be, and what happens when the cache is down?
