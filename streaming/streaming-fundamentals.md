---
title: Streaming Fundamentals
description: Events, logs, partitions, offsets, delivery semantics, event time, windows, backpressure, and replay.
---

# Streaming Fundamentals

## Events and logs

An event records something that happened: an order was placed, a file was
uploaded, or a sensor produced a reading. An append-only log stores events in a
stable order. Consumers track their position with an offset and can replay old
events to rebuild state.

Events should have a stable identifier, type, schema version, occurrence time,
and enough context for consumers to interpret them:

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

Prefer facts such as `OrderPlaced` over instructions such as `SendOrderEmail`
when several independent consumers may use the event.

```{mermaid}
flowchart LR
  checkout[Checkout service] -->|OrderPlaced| orders[(Orders stream)]
  orders --> inventory[Inventory group]
  orders --> billing[Billing group]
  orders --> email[Email group]
  orders --> analytics[Analytics group]
```

If analytics is offline for an hour, the other groups continue. Analytics later
resumes from its own offset and replays what it missed.

## Partitions and ordering

A stream is divided into partitions for parallelism. Each partition is ordered;
there is normally no total order across partitions. The producer's partition key
places related events together.

```{mermaid}
flowchart LR
  producer[Order producer] --> router{Hash customer_id}
  router --> p0[(Partition 0)]
  router --> p1[(Partition 1)]
  router --> p2[(Partition 2)]
  p0 --> c1[Consumer A]
  p1 --> c1
  p2 --> c2[Consumer B]
```

Using `customer_id` preserves order for one customer's events. A random event ID
distributes traffic well but does not preserve customer-level order. A celebrity
or large tenant can make a seemingly reasonable key create a hot partition.

The partition count limits useful parallelism within one consumer group. Extra
consumers beyond the number of partitions remain idle.

## Offsets and consumer groups

An offset identifies a position inside one partition. A consumer group records
its offsets so it can resume after restart. Two groups can read the same event,
while consumers inside one group divide partitions among themselves.

Committing an offset before processing risks losing work. Committing after
processing permits redelivery after a crash. The second approach, combined with
idempotent processing, is a common default.

Adding or removing consumers causes a **rebalance** that reassigns partitions.
Consumers should stop cleanly, finish or checkpoint in-flight work, and avoid
long pauses that make the group mistake a healthy member for a failed one.

## Delivery semantics

- **At most once:** commit before processing; a crash can lose an event.
- **At least once:** process before committing; a crash can duplicate an event.
- **Exactly once:** coordinate input position, processor state, and output
  commits within explicitly supported boundaries.

For example, a billing consumer may charge an order and crash before committing
its offset. The event is delivered again. Supplying
`evt_01J8Z9Q4T2` as the payment provider's idempotency key prevents a second
charge.

“Exactly once” does not automatically make an arbitrary email, HTTP call, or
database update happen once. Verify the guarantee across the complete path from
source through state to sink.

## Event time and windows

**Event time** is when an event happened; **processing time** is when the system
handled it. Delays and retries make those timestamps differ.

Suppose purchases occur at `10:01`, `10:04`, and `10:09`, but the `10:04` event
arrives last:

```text
Event time:       10:01  10:04  10:09
Arrival order:    10:01  10:09  10:04
```

A watermark estimates how far event time has advanced. It lets a processor
close a window while still allowing a defined amount of lateness.

- **Tumbling window:** non-overlapping intervals, such as revenue per minute.
- **Sliding window:** overlapping intervals, such as errors in the last five
  minutes recomputed every minute.
- **Session window:** activity separated by inactivity gaps, such as one user
  browsing session.

## Backpressure and recovery

When producers outpace consumers, lag grows. Retention absorbs temporary bursts
but cannot solve a permanent capacity mismatch.

Monitor:

- consumer lag in records and wall-clock time;
- records and bytes produced and consumed per second;
- partition skew and rebalance frequency;
- processing latency and error rate; and
- storage use relative to retention.

Scale consumers up to the partition count, reduce per-event work, batch I/O, or
repartition. A poison event should follow bounded retries and then move to a
dead-letter stream with its failure context.

## Replay and schema evolution

Replay enables a new consumer to build a view from history or an existing
consumer to repair corrupted output. It also re-executes old assumptions and
bugs, so replay should be rate-limited, observable, and isolated from live
traffic where necessary.

Evolve event schemas compatibly. Add optional fields before requiring them,
version semantic changes, and test old events against new consumers. The event
log is a long-lived interface, not an internal object dump.

## Redis messaging

[Redis](../caching/redis.md) provides two distinct models. Pub/Sub is live,
at-most-once broadcast: a disconnected subscriber loses messages. Redis Streams
retains an append-only sequence and supports consumer groups, pending entries,
acknowledgements, and replay.

Redis Streams can fit compact operational workflows. Compare Kafka when the
design needs longer retention, many independent consumer groups, large replay,
or a broader stream-processing ecosystem.

## Stream versus queue

A stream distributes retained facts to independent groups. A queue assigns one
task to one worker. See [Queue Fundamentals](../queues/queue-fundamentals.md) for
ACK/NACK behavior, visibility timeouts, retry backoff, DLQs, and worker pools.

## Design checklist

- What is the event's identity, schema, partition key, and retention period?
- Which ordering scope is truly required?
- When are offsets committed, and can processing safely repeat?
- How are late and out-of-order events handled?
- How will lag, rebalancing, replay, and schema evolution be operated?
