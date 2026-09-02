---
title: Streaming
---

# Streaming

Streaming systems move an ongoing sequence of events from producers to
consumers. They decouple services in time, support multiple independent views of
the same events, and make high-throughput asynchronous processing possible.

## Events and logs

An event records something that happened: an order was placed, a file was
uploaded, or a sensor produced a reading. An append-only log stores events in a
stable order. Consumers track their position with an offset and can replay old
events to rebuild state or add a new derived view.

Events should have a stable identifier, timestamp, type, schema version, and
enough context for consumers to interpret them. Prefer facts over commands when
the stream is intended for multiple independent consumers.

For example, an order service can publish this fact after its database
transaction commits:

```json
{
  "event_id": "evt_01J8Z9Q4T2",
  "type": "OrderPlaced",
  "schema_version": 1,
  "occurred_at": "2026-09-02T18:42:17Z",
  "order_id": "order_123",
  "customer_id": "customer_42",
  "total_cents": 12999
}
```

Each consumer group reads the same event for a different purpose:

```{mermaid}
flowchart LR
  checkout[Checkout service] -->|OrderPlaced| orders[(Orders stream)]
  orders --> inventory[Inventory consumer]
  orders --> billing[Billing consumer]
  orders --> email[Email consumer]
  orders --> analytics[Analytics consumer]
```

If analytics is offline for an hour, the other consumers continue. Analytics
later resumes from its own offset and replays what it missed.

## Partitions and ordering

A stream is often divided into partitions for parallelism. Events within one
partition have an order; events in different partitions generally do not. A
partition key places related events together—for example, all changes for one
account—so their local order is preserved.

The number of partitions limits consumer parallelism and affects rebalancing
cost. Skewed keys create hot partitions, so partitioning must reflect the actual
traffic distribution.

For the order stream above, `customer_id` is a useful partition key when events
for one customer must remain ordered. It also means a single unusually active
customer can create a hot partition. A random event ID would distribute traffic
well but would not preserve customer-level ordering.

## Delivery semantics

**At-most-once** delivery may lose an event but does not redeliver it.
**At-least-once** delivery retries until acknowledged, so consumers may see
duplicates. **Exactly-once** processing requires coordination across reading,
state changes, and output; platforms that offer it do so within specific
boundaries.

At-least-once delivery with idempotent consumers is a robust default. Consumers
can deduplicate by event ID, use conditional updates, or make the intended state
converge regardless of repetitions.

For example, a billing consumer might charge an order and crash before
acknowledging its event. The stream redelivers the event. Supplying
`event_id = evt_01J8Z9Q4T2` as the payment provider's idempotency key prevents a
second charge.

## Event time and windows

Processing time is when a consumer handles an event; event time is when the
event actually occurred. Network and service delays cause events to arrive late
or out of order. Windowed computations therefore need a policy for lateness,
often expressed with a watermark that estimates how far event time has advanced.

Tumbling windows are fixed and non-overlapping. Sliding windows overlap. Session
windows group activity separated by gaps. The business question determines the
window, not the implementation convenience.

## Backpressure and recovery

When producers outpace consumers, lag grows. Durable retention absorbs temporary
bursts, but it does not solve a permanent capacity mismatch. Monitor consumer
lag in both records and time, then scale consumers, reduce per-event work, or
partition further.

Poison events that fail repeatedly should not block a partition forever. After a
bounded retry policy, preserve the event and failure context in a dead-letter
stream for inspection and controlled replay.

## Stream versus queue

A work queue is ideal when one of several workers should perform a task. A
durable stream is ideal when multiple consumer groups need independent access to
an ordered history. Real systems often use both: a stream distributes facts,
while queues dispatch retryable units of work.

| Question | Stream | Queue |
| --- | --- | --- |
| What is stored? | Facts that happened | Tasks that should be performed |
| Who receives an item? | Every interested consumer group | One worker in the worker pool |
| Is history useful? | Usually retained for replay | Usually removed after success |
| Typical state | Offset per consumer group | Ready, claimed, retrying, or complete |
| Example | `OrderPlaced` | `GenerateInvoicePDF` |

### Stream example: order lifecycle

An order may emit `OrderPlaced`, `PaymentCaptured`, `OrderPacked`, and
`OrderShipped`. Search, support, fraud detection, and analytics can build their
own views by reading that history. Adding a new consumer does not require the
order service to call it directly.

Good stream use cases include:

- **Change Data Capture:** propagate database changes into search indexes,
  warehouses, and downstream services.
- **Audit and event history:** preserve account, order, or security events for
  investigation and rebuilding state.
- **Product analytics:** collect clicks, impressions, searches, and purchases
  for real-time and batch analysis.
- **IoT telemetry:** ingest readings from devices and compute alerts or time
  windows.
- **Integration events:** notify several services that a business event, such as
  `CustomerCreated`, has occurred.

### Queue example: image processing

Suppose an upload API must create thumbnails. Doing the CPU-heavy work inside
the request would keep the user waiting, so the API places a job on a queue:

```json
{
  "job_id": "job_987",
  "type": "GenerateThumbnails",
  "image_uri": "s3://uploads/photo-123.jpg",
  "sizes": [64, 256, 1024],
  "attempt": 1
}
```

```{mermaid}
flowchart LR
  user[User] --> api[Upload API]
  api --> original[(Object storage)]
  api --> jobs[[Image jobs queue]]
  jobs --> worker1[Worker 1]
  jobs --> worker2[Worker 2]
  jobs --> worker3[Worker 3]
  worker1 --> thumbnails[(Thumbnails)]
  worker2 --> thumbnails
  worker3 --> thumbnails
```

Only one worker should claim `job_987`. On failure, the queue makes it available
for retry; after repeated failures, it moves to a dead-letter queue.

Good queue use cases include:

- **Media processing:** resize images, transcode video, or scan uploads.
- **Notifications:** send email, SMS, or push notifications outside the request
  path.
- **Webhook delivery:** retry transient failures with backoff without blocking
  the originating service.
- **Document generation:** create invoices, reports, and exports asynchronously.
- **Scheduled or rate-limited work:** delay a task or control calls to a slow
  third-party API.

```{tip}
Use a stream when the message is a reusable fact: **an order was placed**. Use a
queue when the message is an instruction with one owner: **generate this
invoice**.
```

## Redis messaging

[Redis](./caching/redis.md) provides two distinct messaging models. Pub/Sub is a
live, at-most-once broadcast: a disconnected subscriber misses messages sent
while it is away. Redis Streams retains an append-only sequence and supports
consumer groups, acknowledgements, pending entries, and replay, making
at-least-once processing possible.

Use Pub/Sub for ephemeral notifications such as presence or cache invalidation
when loss is acceptable or repaired elsewhere. Consider Redis Streams for a
compact operational workflow, and compare it with a dedicated streaming
platform when the design needs long retention, many independent consumer
groups, large-scale replay, or a broader stream-processing ecosystem.

## Design checklist

- What is the event schema, identity, and retention period?
- What ordering is actually required?
- How is the partition key chosen?
- Can consumers process duplicates safely?
- How are late, malformed, and poison events handled?
- How will lag, replay, and schema evolution be operated?
