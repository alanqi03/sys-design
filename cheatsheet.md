---
title: Cheatsheet
description: An interview-ready guide to choosing and explaining common components on a system-design whiteboard.
---

# Cheatsheet

Use this as a menu of common boxes for a system-design whiteboard. Draw a
component only when it solves a stated requirement, label important data flows,
and explain the trade-off it introduces. A diagram with fewer justified boxes
is stronger than one filled with technology names.

## A Typical Interview Flow

Start with the shortest path that satisfies the functional requirements, then
add supporting components when a scale, latency, durability, or workflow
requirement justifies them.

```{mermaid}
flowchart LR
  client[Client] --> gateway[Load balancer or API gateway]
  gateway --> service[Application services]
  service <--> cache[(Cache)]
  service --> database[(Database)]
  service --> objects[(Object storage)]
  service --> queue[[Queue]]
  queue --> workers[Worker pool]
  service --> stream[(Event stream)]
  stream --> processor[Stream processor]
  processor --> search[(Search or read model)]
```

This is a prompt, not a required architecture. Begin with the client,
application service, and source of truth. Add a load balancer or gateway when
there are multiple services or instances, then add each remaining box only when
you can state the problem it solves.

## Core Components

These are the first components to consider for an ordinary request-response
system.

### Client

- **Examples:** Web browser, iOS or Android app, another backend service, or an
  IoT device.
- **When to use it:** Every design has a caller. Identify it first because its
  network, trust level, update cycle, and interaction pattern shape the API.
- **Pros:** Can cache data, validate input, retry safe requests, and perform work
  near the user.
- **Cons:** It is untrusted, versions update slowly, networks disconnect, and
  retries can duplicate non-idempotent operations.
- **Important features to know:** Authentication tokens, request IDs,
  idempotency keys, pagination, exponential backoff with jitter, and local or
  HTTP caching.

### Load Balancer

- **Examples:** AWS ALB or NLB, NGINX, HAProxy, and Envoy.
- **When to use it:** Add one when traffic must be spread across multiple
  interchangeable service instances or failed instances must be removed from
  rotation.
- **Pros:** Enables horizontal scaling, health-based routing, failover, and
  rolling deployments.
- **Cons:** Adds a network hop and configuration surface; sticky sessions can
  create uneven load and make failover harder.
- **Important features to know:** L4 versus L7 routing, health checks,
  round-robin or least-connections algorithms, TLS termination, connection
  draining, and optional session affinity.

### API Gateway

- **Examples:** AWS API Gateway, Kong, Apigee, and Envoy Gateway.
- **When to use it:** Add one when many APIs need a shared public entry point
  for authentication, validation, quotas, routing, or protocol translation.
- **Pros:** Centralizes cross-cutting edge policies and hides the internal
  service layout from clients.
- **Cons:** Can become a bottleneck or shared blast radius, adds latency, and
  may accumulate business logic or vendor-specific configuration.
- **Important features to know:** Request validation, middleware, routing,
  protocol translation, response transformation, caching, rate limiting, and
  authentication.

### Application Service

- **Examples:** Stateless HTTP or gRPC services running on virtual machines,
  containers, Kubernetes, ECS, or serverless functions.
- **When to use it:** Use a service to own business logic and APIs. Split it
  into additional services only when ownership, scaling, deployment, or failure
  boundaries justify the extra network calls.
- **Pros:** Encapsulates business logic and scales horizontally when instances
  are interchangeable and request state lives elsewhere.
- **Cons:** More instances do not fix a shared database bottleneck, and every
  extra service call adds latency and another failure mode.
- **Important features to know:** REST, gRPC, or GraphQL APIs; connection pools;
  deadlines; idempotency; graceful shutdown; health checks; and autoscaling.

### Database

- **Examples:** PostgreSQL or MySQL for relational data, DynamoDB for managed
  key-value access, Cassandra for distributed writes, and MongoDB for documents.
- **When to use it:** Use a relational database by default when transactions,
  relationships, and flexible queries matter. Choose a specialized NoSQL model
  when known access patterns or distributed scale justify its trade-offs.
- **Pros:** Makes state durable and queryable. Relational databases provide rich
  queries and ACID transactions; distributed NoSQL systems can scale specific
  access patterns horizontally.
- **Cons:** A relational primary can become a write bottleneck, while NoSQL
  systems often trade away joins, flexible queries, or strong multi-record
  transactions. Poor indexes or partition keys hurt either model.
- **Important features to know:** Schema and access patterns, indexes, ACID
  transactions, isolation levels, replication, consistency choices, backups,
  partition keys, read replicas, and sharding.

## Supporting Components

Add these in roughly this order of consideration: first improve the main read
and write paths, then introduce specialized asynchronous or operational
components.

### Cache

- **Examples:** Redis, Memcached, an in-process cache, and a CDN for static
  content.
- **When to use it:** Add a cache when repeated reads or expensive computations
  dominate latency or database load and the product can define acceptable
  staleness and invalidation behavior.
- **Pros:** Provides very fast reads, protects the source of truth, and absorbs
  repeated access to hot data.
- **Cons:** Invalidation and staleness are hard; misses, stampedes, evictions,
  and hot keys can overload the database or one cache node.
- **Important features to know:** Cache-aside, write-through, write-behind,
  TTLs, eviction policies, replication, sharding, request coalescing, and an
  explicit consistency expectation.

### Queue

- **Examples:** Amazon SQS, RabbitMQ, Azure Service Bus, and Google Cloud Tasks.
- **When to use it:** Use a durable queue to move slow or expensive tasks—such
  as precomputing feeds, generating reports, or processing media—out of the
  request path. It also fits bursty traffic, retryable work, and cases where
  producers and workers must scale independently.
- **Pros:** Buffers bursts, isolates failures, and lets slow or retryable work
  complete outside the request path.
- **Cons:** Completion becomes eventual, backlogs need monitoring, and
  at-least-once delivery means a task may run more than once.
- **Important features to know:** Competing consumers, acknowledgement or
  visibility timeout, bounded retries, dead-letter queues, backpressure,
  idempotent workers, and ordering scope.

### Worker Pool

- **Examples:** Celery, Sidekiq, BullMQ, Kubernetes workers, and AWS Lambda
  consuming queue messages.
- **When to use it:** Add workers to execute queued tasks, CPU-heavy jobs, batch
  work, or slow external calls without tying up API request capacity.
- **Pros:** Moves slow work off the request path, processes jobs in parallel,
  and scales independently from API servers.
- **Cons:** Retries can repeat side effects, poison tasks can block progress,
  and excessive concurrency can overload downstream systems.
- **Important features to know:** Idempotent handlers, concurrency limits,
  leases or heartbeats, timeouts, retry backoff, dead-letter handling,
  autoscaling on queue age or depth, and graceful shutdown.

### Object Storage

- **Examples:** Amazon S3, Google Cloud Storage, and Azure Blob Storage.
- **When to use it:** Use object storage for images, video, documents, backups,
  exports, or other large immutable blobs that do not belong in database rows.
- **Pros:** Stores very large blobs cheaply with high durability and massive
  scale, without sending file bytes through application servers.
- **Cons:** Has higher per-operation latency than memory or local disk and is
  not designed for relational queries or frequent in-place updates.
- **Important features to know:** Presigned URLs, multipart upload, range
  requests, checksums, versioning, lifecycle and retention policies, event
  notifications, and CDN integration.

### CDN

- **Examples:** CloudFront, Cloudflare, Fastly, and Akamai.
- **When to use it:** Add a CDN when static or cacheable content is requested by
  geographically distributed users or origin bandwidth and latency are high.
- **Pros:** Serves content near users, lowers latency, and removes bandwidth and
  read traffic from the origin.
- **Cons:** Invalidation is difficult, content can be stale, and personalized or
  private responses require careful cache keys and access controls.
- **Important features to know:** Edge locations, TTLs, cache keys, purge or
  invalidation, origin shielding, signed URLs, and stale-while-revalidate.

### Search Index

- **Examples:** Elasticsearch, OpenSearch, Solr, and a database's built-in
  full-text search for smaller workloads.
- **When to use it:** Add a dedicated index when users need relevance-ranked
  text search, autocomplete, faceting, or complex filtering that the primary
  database cannot serve efficiently.
- **Pros:** Supports fast full-text search, relevance ranking, filtering,
  autocomplete, and aggregations.
- **Cons:** Usually holds a derived, eventually consistent copy; reindexing is
  expensive, and mapping or shard mistakes can cause operational trouble.
- **Important features to know:** Inverted indexes, analyzers and tokenizers,
  mappings, relevance scoring, shards and replicas, refresh intervals, aliases,
  and a CDC or outbox indexing pipeline.

### Pub/Sub

- **Examples:** Amazon SNS, Google Cloud Pub/Sub, NATS, and Redis Pub/Sub for
  ephemeral real-time delivery.
- **When to use it:** Use pub/sub to refresh or invalidate caches and to push
  live updates such as chat messages, clicks, sensor readings, or stock prices
  to many listeners. Choose a durable stream when consumers also need history
  and replay.
- **Pros:** Decouples publishers from subscribers and makes adding new consumers
  easier.
- **Cons:** Delivery and durability vary by product. Ephemeral systems can lose
  messages while subscribers are disconnected, and slow subscribers need an
  explicit buffering strategy.
- **Important features to know:** Topics, subscriptions, fan-out, filtering,
  retention, replay, delivery guarantees, and whether each subscriber gets its
  own durable backlog. Pub/sub is a pattern, not automatically a durable queue.

### Streaming Platform

- **Examples:** Apache Kafka, Amazon Kinesis, Apache Pulsar, and Redpanda.
- **When to use it:** Use a stream when events must be retained and replayed,
  several consumer groups need independent history, or high-throughput ordered
  processing is required.
- **Pros:** Retains an ordered event log that consumers can read independently,
  replay, and process at high throughput.
- **Cons:** Ordering is normally limited to a partition, consumers see data
  asynchronously, and partitioning, schema evolution, lag, and retention need
  active management.
- **Important features to know:** Topics, partitions, offsets, consumer groups,
  retention, replication, partition keys, schema registry, replay, and
  at-least-once versus exactly-once processing boundaries.

### Stream Processor

- **Examples:** Apache Flink, Kafka Streams, Spark Structured Streaming, and
  managed Dataflow services.
- **When to use it:** Add one when an event stream must continuously produce
  windows, joins, alerts, aggregates, or materialized read models.
- **Pros:** Builds derived results incrementally instead of recalculating them
  on every request.
- **Cons:** Late or out-of-order events, replay, state growth, and correctness
  during failures make processing more complex than a stateless consumer.
- **Important features to know:** Event time versus processing time, windows,
  watermarks, state and checkpoints, partitioning, delivery semantics, and
  backfills.

### Real-Time Connection Layer

- **Examples:** WebSocket gateways, Server-Sent Events endpoints, Socket.IO,
  managed WebSocket services, and WebRTC for peer media.
- **When to use it:** Use SSE for one-way server updates, WebSockets for
  bidirectional low-latency commands, and WebRTC for peer-to-peer audio, video,
  or data.
- **Pros:** Pushes updates immediately instead of making every client poll and
  can support interactive bidirectional communication.
- **Cons:** Long-lived connections consume resources, disconnect frequently,
  and require routing, backpressure, authentication refresh, and reconnection
  logic.
- **Important features to know:** Connection registry, heartbeats, reconnect and
  resume, per-connection buffers, presence, fan-out through pub/sub, and the
  differences among SSE, WebSockets, and WebRTC.

### Workflow Engine

- **Examples:** Temporal, AWS Step Functions, Google Workflows, and Azure Durable
  Functions.
- **When to use it:** Use a workflow engine for long-running, multi-step
  processes that must survive crashes, wait on timers or callbacks, and recover
  with retries or compensating actions.
- **Pros:** Durably tracks progress and provides retries, timeouts, waiting, and
  recovery without keeping one service process alive.
- **Cons:** Adds another platform and programming model, while workflow history,
  long retention, and high-volume tiny steps can increase cost.
- **Important features to know:** Durable execution, activity retries, timers,
  saga compensations, idempotent activities, workflow versioning, and human or
  external callbacks.

### Observability

- **Examples:** OpenTelemetry, Prometheus and Grafana, Datadog, CloudWatch, and
  an ELK or OpenSearch logging stack.
- **When to use it:** Instrument every production component; emphasize it on the
  whiteboard when availability, latency, incident detection, or capacity
  planning is an explicit requirement.
- **Pros:** Reveals bottlenecks and failures, supports alerting, and provides the
  evidence needed to evaluate latency and availability requirements.
- **Cons:** High-cardinality telemetry and long retention can be expensive;
  noisy alerts and unstructured logs hide important signals.
- **Important features to know:** Metrics, logs, distributed traces, correlation
  IDs, RED signals—rate, errors, duration—queue age, saturation, dashboards,
  SLOs, and actionable alerts.

## Final Whiteboard Check

- Can you explain which requirement justifies every box?
- Is the source of truth obvious?
- Are synchronous and asynchronous arrows distinguishable?
- Where can data be stale, duplicated, reordered, or lost?
- What happens when each dependency is slow or unavailable?
- Which component reaches its capacity limit first, and how does it scale?
