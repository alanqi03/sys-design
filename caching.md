---
title: Caching
---

# Caching

Caching places reusable data or computation closer to the caller. It is one of
the most effective ways to reduce latency and protect a slower source, but every
cache introduces a second copy of data whose freshness, capacity, and failure
behavior must be designed deliberately.

## In this section

- [Caching Fundamentals](./caching/fundamentals.md) explains cache placement,
  read and write patterns, invalidation, eviction, stampedes, hot keys, and
  failure planning.
- [Redis](./caching/redis.md) applies those ideas to an in-memory data structure
  server that also supports counters, leaderboards, messaging, streams,
  coordination, and optional persistence.

Start with the fundamentals. Redis is a useful implementation, but choosing it
does not remove the need to define the source of truth, acceptable staleness,
memory limits, and behavior during a cache miss or outage.
