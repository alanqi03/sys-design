---
title: Quick Reference
description: A decision guide for improving service durability, fault tolerance, read scale, write scale, latency, and burst handling.
---

# Quick Reference

Use this page to turn a system symptom into candidate design changes. These are
starting points, not automatic fixes: identify the bottleneck, choose the
smallest change that addresses it, and load-test the complete path.

## Strategy map

| Goal | Common strategies | Important cost or limit |
| --- | --- | --- |
| Improve durability and fault tolerance | Redundancy, durable queues, replication, retries, backups | More components, duplicate work, failover complexity |
| Scale reads | Horizontal service scaling, CDN, cache, read replicas, precomputed views | Staleness and invalidation |
| Scale writes | Cheaper writes, batching, asynchronous work, partitioning, sharding | Ordering, transactions, hot partitions |
| Reduce latency | Cache nearby, reduce hops, reuse connections, move optional work off-path | Stale data and more complex recovery |
| Absorb bursts | Durable queues, rate limits, backpressure, autoscaling, load shedding | Added delay and asynchronous behavior |

## Before changing the architecture

- Define the target: requests per second, data size, availability, and p95 or
  p99 latency.
- Locate the constraint using CPU, memory, network, storage IOPS, database query
  time, connection-pool wait, queue age, and dependency latency.
- Check whether one slow query, hot key, tenant, or endpoint dominates the load.
- Load-test with realistic data, request mixes, cache hit rates, and failures.

Scaling the wrong tier only moves the queue. Ten application replicas cannot
increase write throughput when all of them wait on one saturated database row.

## Improving durability and fault tolerance

**Durability** means acknowledged data survives failures. **Availability** means
the system can continue serving requests. **Fault tolerance** is the ability to
contain a failure and recover or continue with acceptable behavior. A backup can
improve durability without making a service immediately available.

### Add redundancy

- Run multiple stateless service instances behind a load balancer and spread
  them across failure domains such as availability zones.
- Use health checks to stop routing to unhealthy instances, but avoid checks
  that remove every replica during a shared dependency failure.
- Replicate durable data and practice promotion or failover. Replication is not
  a backup: corruption or accidental deletion can replicate too.
- Keep versioned backups and test point-in-time restoration against a recovery
  time objective (RTO) and recovery point objective (RPO).

### Put asynchronous work on a durable queue

For work that does not need to finish before the response, accept and durably
store a task, return `202 Accepted` with a job ID, and let workers process it.

```{mermaid}
flowchart LR
  client[Client] -->|POST command| api[API]
  api -->|Persist task| queue[[Durable queue]]
  api -->|202 + job ID| client
  queue --> worker[Worker pool]
  worker --> database[(Database)]
  client -->|Poll job status| api
```

The API is normally the trusted boundary between an external client and the
queue. Between internal services, the producer can publish directly if the
broker and authorization model allow it.

A queue helps when the downstream service is temporarily unavailable or when a
burst should be processed at a controlled rate. It changes the contract to
asynchronous completion and can redeliver work, so workers must be idempotent,
retries must be bounded, and poison tasks need a dead-letter queue. See
[Queue Fundamentals](./queues/queue-fundamentals.md).

### Make service-to-service delivery recoverable

```{mermaid}
flowchart LR
  a[Service A] -->|One transaction| db[(Business data + outbox)]
  db --> relay[Outbox relay]
  relay --> queue[[Durable queue]]
  queue --> b[Idempotent Service B]
```

Writing business state and publishing a message as two independent operations
creates a dual-write failure window. A
[transactional outbox](./database/change-data-capture-outbox.md) stores the
state change and an outgoing event in one database transaction; a relay
publishes the event afterward. Consumers deduplicate by a stable message ID.

### Contain dependency failures

- Apply deadlines to every remote call and use bounded retries with exponential
  backoff and jitter only for transient failures.
- Make retried operations idempotent so an ambiguous timeout does not duplicate
  a charge, order, or notification.
- Use circuit breakers, concurrency limits, and bulkheads to stop one failing
  dependency from consuming every request thread or connection.
- Degrade optional features and shed excess load before the entire service
  becomes unhealthy.

## Scaling Reads

Apply read strategies roughly from simpler to more structural:

1. **Make each read cheaper.** Fix N+1 calls, select only required data, add
   workload-shaped indexes, and inspect query plans.
2. **Scale stateless services horizontally.** Put more interchangeable instances
   behind a load balancer and keep request state in shared storage or a token.
3. **Cache repeated results.** Use browser or HTTP caches, a CDN for public
   content, and an application cache such as Redis for shared computed data.
4. **Add database read replicas.** Route stale-tolerant queries to followers;
   keep correctness-sensitive and read-after-write traffic on the primary.
5. **Precompute expensive views.** Denormalize or build materialized views and
   search indexes when the same join or aggregation is requested repeatedly.
6. **Partition the read path.** Split data by tenant, geography, or another
   stable key when one machine cannot hold or serve the working set.

```{mermaid}
flowchart LR
  clients[Clients] --> cdn[CDN]
  clients --> lb[Load balancer]
  lb --> s1[Service 1]
  lb --> s2[Service 2]
  s1 <--> cache[(Shared cache)]
  s2 <--> cache
  s1 --> primary[(Primary)]
  s2 --> replica[(Read replica)]
  primary -. replicate .-> replica
```

Watch cache hit rate, eviction rate, stampedes, and hot keys. Define acceptable
staleness and invalidation before adding a cache. Replica lag can violate
read-after-write expectations, and every service replica still needs a bounded
database connection pool. See [Caching Fundamentals](./caching/fundamentals.md)
and [Databases](./database.md).

## Scaling Writes

Horizontal application scaling accepts more concurrent requests, but the shared
database or hot record may remain the write bottleneck. Improve the entire path:

1. **Make each write cheaper.** Keep transactions short, batch compatible work,
   remove unnecessary indexes, avoid repeated updates, and use efficient bulk
   ingestion.
2. **Scale up the writer.** More CPU, memory, network, and storage IOPS is often
   simpler than distributing correctness-sensitive data.
3. **Move secondary work off the synchronous path.** Commit the source-of-truth
   change, then update search, analytics, notifications, and derived views
   through an outbox and queue or stream.
4. **Partition independent work.** Route by a stable key so workers or storage
   owners can process partitions in parallel while preserving per-key order.
5. **Shard the data.** Give each shard its own write capacity and route tenants
   or records by shard key. Plan rebalancing, cross-shard queries, and shard
   failure before adopting it.
6. **Use a database designed for distributed writes** when sustained scale or
   multi-region requirements justify weaker transactions or greater operational
   complexity.

```{mermaid}
flowchart LR
  api[Service replicas] --> router{Shard router}
  router -->|tenant A–H| s1[(Shard 1)]
  router -->|tenant I–P| s2[(Shard 2)]
  router -->|tenant Q–Z| s3[(Shard 3)]
```

A good shard key spreads bytes and requests while keeping common transactions
inside one shard. More shards do not fix a single hot tenant or counter; split
that workload further, batch updates, or give it a serialized owner. Treat
sharding as a major architecture decision, not the first response to a slow
query. Compare [PostgreSQL](./database/postgresql.md),
[DynamoDB](./database/dynamodb.md), and [Cassandra](./database/cassandra.md).

## Reducing Latency

- Place static content at the edge and services near users or dependent data.
- Cache expensive, frequently reused results at the narrowest safe scope.
- Reuse connections, batch calls, and parallelize independent requests.
- Remove optional emails, analytics, and derived-index updates from the critical
  path using asynchronous processing.
- Avoid unnecessary service hops; every hop adds network time and another queue
  where tail latency can accumulate.

Optimize p95 and p99, not only the average. Hedged requests can reduce tail
latency for safe, idempotent reads, but they increase load and should be delayed
and tightly bounded.

## Absorbing Bursts and Applying Backpressure

- Buffer deferrable work in a durable queue and scale workers using queue age,
  not only CPU.
- Bound every in-memory queue, connection pool, and concurrency limit. An
  unbounded buffer turns overload into an out-of-memory failure.
- Rate-limit by customer or operation, reserve capacity for critical traffic,
  and return explicit overload responses such as `429` or `503`.
- Propagate backpressure rather than accepting work that cannot complete within
  its deadline.
- Use admission control or load shedding to preserve a smaller amount of useful
  work during overload.

## Final checklist

- What resource is saturated, and what measurement proves it?
- Does the change improve capacity, latency, durability, or availability?
- What consistency or ordering guarantee becomes weaker?
- What new failure mode, stale-data path, or operational component is added?
- Are retries idempotent and bounded?
- Can the system recover if the cache, queue, replica, or entire zone fails?
- Has the new design been load-tested and its failover practiced?
