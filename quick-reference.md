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

## Cheatsheet

Use this as a menu of common boxes for a system-design whiteboard. Draw a
component only when it solves a stated requirement, label important data flows,
and explain the trade-off it introduces. A diagram with fewer justified boxes
is stronger than one filled with technology names.

### Client

- **Examples:** Web browser, iOS or Android app, another backend service, or an
  IoT device.
- **Pros:** Can cache data, validate input, retry safe requests, and perform work
  near the user.
- **Cons:** It is untrusted, versions update slowly, networks disconnect, and
  retries can duplicate non-idempotent operations.
- **Important features to know:** Authentication tokens, request IDs,
  idempotency keys, pagination, exponential backoff with jitter, and local or
  HTTP caching.

### DNS and Global Traffic Routing

- **Examples:** Route 53, Cloudflare DNS, Google Cloud DNS, and Azure Traffic
  Manager.
- **Pros:** Gives a stable domain name and can route users toward a healthy,
  nearby region before they reach the application.
- **Cons:** DNS answers are cached, so failover is not instantaneous; routing
  decisions have limited knowledge of an individual request.
- **Important features to know:** A, AAAA, and CNAME records; TTLs; health
  checks; weighted, latency-based, and geographic routing; and DNS failover.

### CDN

- **Examples:** CloudFront, Cloudflare, Fastly, and Akamai.
- **Pros:** Serves static or cacheable content near users, lowers latency, and
  removes bandwidth and read traffic from the origin.
- **Cons:** Invalidation is difficult, content can be stale, and personalized or
  private responses require careful cache keys and access controls.
- **Important features to know:** Edge locations, TTLs, cache keys, purge or
  invalidation, origin shielding, signed URLs, and stale-while-revalidate.

### Load Balancer

- **Examples:** AWS ALB or NLB, NGINX, HAProxy, and Envoy.
- **Pros:** Spreads traffic across service instances, removes unhealthy nodes,
  and enables horizontal scaling and rolling deployments.
- **Cons:** Adds a network hop and configuration surface; sticky sessions can
  create uneven load and make failover harder.
- **Important features to know:** L4 versus L7 routing, health checks,
  round-robin or least-connections algorithms, TLS termination, connection
  draining, and optional session affinity.

### API Gateway

- **Examples:** AWS API Gateway, Kong, Apigee, and Envoy Gateway.
- **Pros:** Provides one public entry point for authentication, validation,
  rate limits, routing, and cross-cutting policies.
- **Cons:** Can become a bottleneck or single blast radius, adds latency, and
  may accumulate business logic or vendor-specific configuration.
- **Important features to know:** Request validation, middleware, routing,
  protocol translation, response transformation, caching, quotas, and
  authentication.

### Rate Limiter

- **Examples:** Gateway rate limits, Envoy or NGINX filters, and a distributed
  limiter backed by Redis.
- **Pros:** Protects services from abuse and overload, enforces customer quotas,
  and creates predictable capacity boundaries.
- **Cons:** Shared counters add latency and can become hot; a poor key or limit
  rejects legitimate bursts, while regional limiters may only approximate a
  global quota.
- **Important features to know:** Token bucket, leaky bucket, fixed or sliding
  window, limit keys such as user or API key, `429 Too Many Requests`,
  `Retry-After`, local versus distributed enforcement, and fail-open versus
  fail-closed behavior.

### Application Service

- **Examples:** Stateless HTTP or gRPC services running on virtual machines,
  containers, Kubernetes, ECS, or serverless functions.
- **Pros:** Encapsulates business logic and scales horizontally when instances
  are interchangeable and request state lives elsewhere.
- **Cons:** More instances do not fix a shared database bottleneck, and every
  extra service call adds latency and another failure mode.
- **Important features to know:** REST, gRPC, or GraphQL APIs; connection pools;
  deadlines; idempotency; graceful shutdown; health checks; and autoscaling.

### Cache

- **Examples:** Redis, Memcached, an in-process cache, and a CDN for static
  content.
- **Pros:** Provides very fast reads, protects the source of truth, and can
  absorb repeated access to hot data.
- **Cons:** Invalidation and staleness are hard; misses, stampedes, evictions,
  and hot keys can overload the database or one cache node.
- **Important features to know:** Cache-aside, write-through, write-behind,
  TTLs, eviction policies, replication, sharding, request coalescing, and an
  explicit consistency expectation.

### Database

- **Examples:** PostgreSQL or MySQL for relational data, DynamoDB for managed
  key-value access, Cassandra for distributed writes, and MongoDB for documents.
- **Pros:** Makes state durable and queryable. Relational databases provide rich
  queries and ACID transactions; distributed NoSQL systems can provide easier
  horizontal scale for specific access patterns.
- **Cons:** A relational primary can become a write bottleneck, while NoSQL
  systems often trade away joins, flexible queries, or strong multi-record
  transactions. Poor indexes or partition keys hurt either model.
- **Important features to know:** Schema and access patterns, indexes, ACID
  transactions, isolation levels, replication, consistency choices, backups,
  partition keys, read replicas, and sharding.

### Search Index

- **Examples:** Elasticsearch, OpenSearch, Solr, and a database's built-in
  full-text search for smaller workloads.
- **Pros:** Supports fast full-text search, relevance ranking, filtering,
  autocomplete, and aggregations that are awkward in the primary database.
- **Cons:** Usually holds a derived, eventually consistent copy; reindexing is
  expensive, and mapping or shard mistakes can cause operational trouble.
- **Important features to know:** Inverted indexes, analyzers and tokenizers,
  mappings, relevance scoring, shards and replicas, refresh intervals, aliases,
  and a CDC or outbox indexing pipeline.

### Object Storage

- **Examples:** Amazon S3, Google Cloud Storage, and Azure Blob Storage.
- **Pros:** Stores very large blobs cheaply with high durability and virtually
  unlimited scale, without sending file bytes through application servers.
- **Cons:** Has higher per-operation latency than memory or local disk and is
  not designed for relational queries or frequent in-place updates.
- **Important features to know:** Presigned URLs, multipart upload, range
  requests, checksums, versioning, lifecycle and retention policies, event
  notifications, and CDN integration.

### Queue

- **Examples:** Amazon SQS, RabbitMQ, Azure Service Bus, and Google Cloud Tasks.
- **Pros:** Buffers bursts, isolates failures, and lets slow or retryable work
  complete asynchronously.
- **Cons:** Completion becomes eventual, backlogs need monitoring, and
  at-least-once delivery means a task may run more than once.
- **Important features to know:** Producer, queue, competing consumers,
  acknowledgement or visibility timeout, bounded retries, dead-letter queue,
  backpressure, idempotent workers, and ordering scope.

### Pub/Sub

- **Examples:** Amazon SNS, Google Cloud Pub/Sub, NATS, and Redis Pub/Sub for
  ephemeral real-time delivery.
- **Pros:** Fans one event out to multiple independent subscribers and decouples
  the publisher from consumers.
- **Cons:** Delivery and durability vary by product. Ephemeral systems can lose
  messages while subscribers are disconnected, and slow subscribers need an
  explicit buffering strategy.
- **Important features to know:** Topics, subscriptions, fan-out, filtering,
  retention, replay, delivery guarantees, and whether each subscriber gets its
  own durable backlog. Pub/sub is a communication pattern, not automatically a
  durable queue.

### Streaming Platform

- **Examples:** Apache Kafka, Amazon Kinesis, Apache Pulsar, and Redpanda.
- **Pros:** Retains an ordered event log that multiple consumer groups can read
  independently, replay, and process at high throughput.
- **Cons:** Ordering is normally limited to a partition, consumers see data
  asynchronously, and partitioning, schema evolution, lag, and retention need
  active management.
- **Important features to know:** Topics, partitions, offsets, consumer groups,
  retention, replication, partition keys, schema registry, replay, and
  at-least-once versus exactly-once processing boundaries.

### Worker Pool

- **Examples:** Celery, Sidekiq, BullMQ, Kubernetes workers, and AWS Lambda
  consuming queue messages.
- **Pros:** Moves slow work off the request path, processes partitions in
  parallel, and scales independently from API servers.
- **Cons:** Retries can repeat side effects, poison tasks can block progress,
  and overloaded downstream systems can turn worker scaling into a failure
  amplifier.
- **Important features to know:** Idempotent job handlers, concurrency limits,
  leases or heartbeats, timeouts, retry backoff, dead-letter handling,
  autoscaling on queue age or depth, and graceful shutdown.

### Stream Processor

- **Examples:** Apache Flink, Kafka Streams, Spark Structured Streaming, and
  managed Dataflow services.
- **Pros:** Continuously transforms events into aggregates, joins, alerts, and
  materialized views without recalculating everything on each request.
- **Cons:** Late or out-of-order events, replay, state growth, and correctness
  during failures make processing more complex than a stateless consumer.
- **Important features to know:** Event time versus processing time, windows,
  watermarks, state and checkpoints, partitioning, delivery semantics, and
  backfills.

### Workflow Engine

- **Examples:** Temporal, AWS Step Functions, Google Workflows, and Azure Durable
  Functions.
- **Pros:** Durably tracks a multi-step process and provides retries, timeouts,
  waiting, and recovery without keeping one service process alive.
- **Cons:** Adds another platform and programming model, while workflow history,
  long retention, and high-volume tiny steps can increase cost.
- **Important features to know:** Durable execution, activity retries, timers,
  saga compensations, idempotent activities, workflow versioning, and human or
  external callbacks.

### Real-Time Connection Layer

- **Examples:** WebSocket gateways, Server-Sent Events endpoints, Socket.IO,
  managed services such as API Gateway WebSocket APIs, and WebRTC for peer media.
- **Pros:** Pushes updates immediately instead of making every client poll and
  can support bidirectional commands when WebSockets are used.
- **Cons:** Long-lived connections consume resources, disconnect frequently,
  and require routing, backpressure, authentication refresh, and reconnection
  logic.
- **Important features to know:** Connection registry, heartbeats, reconnect and
  resume, per-connection buffers, presence, fan-out through pub/sub, and the
  difference between SSE, WebSockets, and WebRTC.

### Observability

- **Examples:** OpenTelemetry, Prometheus and Grafana, Datadog, CloudWatch, and
  an ELK or OpenSearch logging stack.
- **Pros:** Reveals bottlenecks and failures, supports alerting, and provides the
  evidence needed to evaluate latency and availability requirements.
- **Cons:** High-cardinality telemetry and long retention can be expensive;
  noisy alerts and unstructured logs hide important signals.
- **Important features to know:** Metrics, logs, distributed traces, correlation
  IDs, RED signals—rate, errors, duration—queue age, saturation, dashboards,
  SLOs, and actionable alerts.

### A typical interview flow

```{mermaid}
flowchart LR
  client[Client] --> dns[DNS and global routing]
  dns --> cdn[CDN]
  dns --> gateway[Load balancer or API gateway]
  gateway --> limiter[Rate limiter]
  limiter --> service[Application services]
  service <--> cache[(Cache)]
  service --> database[(Database)]
  service --> objects[(Object storage)]
  service --> queue[[Queue]]
  queue --> workers[Worker pool]
  service --> stream[(Event stream)]
  stream --> processor[Stream processor]
  processor --> search[(Search or read model)]
```

This is a prompt, not a required architecture. Start with client, service, and
source of truth; add the cache, queue, stream, search index, or workflow engine
only when a functional or non-functional requirement calls for it.

## Latency targets for interviews

When requirements are unspecified, these are reasonable starting assumptions
for user-facing, end-to-end latency—not universal guarantees:

- **Ordinary API read:** p95 under **200–300 ms**.
- **API write:** p95 under **300–500 ms**.
- **Complex search or feed generation:** p95 under **500 ms**.
- **Anything over one second:** usually show progress or make the operation
  asynchronous.

Always state the percentile, measurement boundary, user geography, and whether
the path is a cache hit or miss. Then divide the target into budgets for the
client network, gateway, service calls, storage, and safety margin.

## Fast capacity estimation

Use decimal units for quick interview math: **1 KB = 1,000 bytes, 1 GB = 1
billion bytes, and 1 TB = 1 trillion bytes**. Real systems may report binary
units such as GiB and TiB, but consistency matters more than converting every
number perfectly on a whiteboard.

| Estimate | Quick formula |
| --- | --- |
| Raw stored data | `records × average bytes per record` |
| New data per day | `average writes/second × bytes per write × 86,400` |
| Average operations/second | `operations per day ÷ 86,400` |
| Peak operations/second | `average operations/second × stated peak factor` |
| Database reads after caching | `read requests/second × reads per request × (1 - cache hit rate)` |
| Provisioned cache | `hot data × object overhead × copies ÷ target utilization` |

### Database size example

Suppose the system has **500 million users** and stores **5 KB per user**:

```text
500M users × 5 KB = 2,500 GB = 2.5 TB of raw user data
```

That 2.5 TB is only the starting point. Add indexes and row metadata, replicas,
backups, temporary working space, expected growth, and operational headroom.
State each factor separately instead of hiding them inside one unexplained
multiplier.

For append-heavy data, estimate growth from the average rate and retention:

```text
stored history = writes/second × bytes/write × retention seconds
```

Use the peak rate to size throughput, but use the time-weighted average rate to
size long-term storage. A one-hour traffic spike should not be multiplied by 24
hours unless it truly lasts all day.

### Cache size example

A cache usually holds the **hot working set**, not every database record. If
10% of those users are active enough to cache:

```text
50M hot users × 5 KB = 250 GB of raw cached values
250 GB × 1.25 metadata/allocator overhead × 2 copies ÷ 0.80 utilization
  ≈ 780 GB provisioned cache memory
```

The overhead, replication, and target-utilization values are assumptions—say
them aloud. Also size the cache cluster for operations/second and hot-key load;
enough memory does not guarantee enough throughput.

### TPS and QPS example

First clarify the unit:

- **QPS or operations/second** counts individual queries or operations.
- **TPS** counts completed transactions; one transaction may contain several
  reads or writes.
- **API RPS** counts requests at the service boundary; one request may create
  multiple database transactions.

If the system expects **50K writes/second at peak** and each write stores 5 KB:

```text
peak ingest = 50K × 5 KB = 250 MB/second
if sustained for one hour = 250 MB × 3,600 = 900 GB
if sustained all day = 250 MB × 86,400 = 21.6 TB/day
```

If “write” means one transaction, the database target is 50K write TPS. If ten
independent writes can safely be batched into each transaction, the target is
5K TPS but still 50K record writes/second. Benchmark the real transaction,
including indexes, constraints, logging, and replication.

Caching changes the database read target. For example:

```text
100K API reads/second × 2 cacheable lookups = 200K cache gets/second
200K × (1 - 90% hit rate) = 20K database reads/second
```

### Turn estimates into machines

Calculate both storage-based and throughput-based capacity, then use the larger:

```text
units for storage = total provisioned bytes ÷ usable bytes per unit
units for load = peak operations/second ÷ safe tested operations/second per unit
```

Round up and leave capacity for a node failure, maintenance, rebalancing, and
growth. Replicas improve availability and may scale reads, but they usually do
not increase the primary's write throughput.

## Component ballparks for interviews

Use these as order-of-magnitude starting assumptions, not limits or vendor
promises. State the payload size, operation complexity, durability and
replication settings, cache state, and whether a number is per node, partition,
or cluster.

### Cache, such as Redis

- **Latency:** expect roughly **0.5–2 ms** at the application for a simple,
  same-region cache operation. Redis itself commonly averages below **1 ms**,
  but that measurement excludes application serialization and network time.
- **Throughput:** **100k+ simple operations/second per node** is plausible with
  enough concurrency; pipelining can make benchmark results much higher.
  Payload size, command complexity, network bandwidth, and hot keys matter.
- **Capacity:** caches are memory-bound, but **1 TB is not a general maximum**.
  Estimate values plus keys, metadata, allocator overhead, replicas, and
  headroom; shard across nodes when the working set is too large for one node.

See Redis's [latency guidance](https://redis.io/docs/latest/operate/rs/monitoring/observability/)
and [benchmark notes](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/benchmarks/).

### Relational database

- **Latency:** budget roughly **1–10 ms** for a simple, cache-hot indexed read
  and **5–20 ms** for a short durable write in the same region. Joins,
  contention, disk misses, synchronous replicas, and cross-region round trips
  can make either much slower.
- **Throughput:** a useful single-primary starting range is **10k–50k simple,
  cache-hot reads/second** and **1k–10k non-trivial write
  transactions/second**. The proposed 10k–20k writes/second is achievable for
  favorable workloads, but is too optimistic as a generic assumption.
- **Capacity:** think from gigabytes to **tens of TiB per instance**, then
  partition or shard beyond one machine. For a concrete vendor example, Amazon
  RDS supports up to **64 TiB** for PostgreSQL and several other engines; this
  is an RDS limit, not a universal database limit.

See Amazon RDS's [storage limits](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Storage.html).

### Message queues and durable logs

- **Latency:** **1–5 ms** can be possible for a local or memory-first path, but
  **5–100 ms** is a safer planning range for durable, replicated, end-to-end
  delivery. Broker acknowledgements and consumer processing are different
  measurement boundaries.
- **Throughput:** use **10k–100k durable messages/second per queue or partition**
  as a conservative starting range. Partitioned logs, streams, and batching can
  reach **100k–1M+ messages/second**, so 1M is plausible but not a generic
  per-broker guarantee.
- **Capacity:** **50 TB is plausible, not a standard maximum**. Estimate
  `messages/second × average message bytes × retention seconds × replication
  factor`; ordinary work queues should usually have bounded backlogs, while
  durable logs may intentionally retain terabytes or more.

For scale, RabbitMQ reports about **80k messages/second** for one quorum queue,
about **100k** for one classic queue, and millions for well-batched streams in
its [queue and stream comparison](https://www.rabbitmq.com/docs/compare/kafka).

### Servers

- **Concurrent connections:** **10k–100k mostly idle or lightweight connections
  per tuned asynchronous node** can be possible. Active requests doing TLS,
  application work, or large transfers consume much more CPU and bandwidth, so
  load-test instead of treating 100k as a default.
- **Machine size:** **8–64 vCPU and 64–512 GiB RAM** describes a medium-to-large
  server, not a universal standard. Many stateless services need less; large
  memory-optimized machines offer more.
- **Upper bound:** **2 TB is not a server maximum**. Current EC2 high-memory
  instance names cover **3–32 TiB**; specialized hardware aside, CPU frequency
  alone is not a useful capacity estimate—request cost, vCPU count, memory,
  network, and storage are what matter.

See the EC2 [instance naming and memory ranges](https://docs.aws.amazon.com/ec2/latest/instancetypes/instance-type-names.html).

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
