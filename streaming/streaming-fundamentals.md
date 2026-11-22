---
title: Streaming Fundamentals
description: Events, logs, partitions, offsets, delivery semantics, event time, windows, backpressure, and replay.
---

# Streaming Fundamentals

**Running example:** an online store. Every time someone checks out, the
checkout service writes one `OrderPlaced` event to a stream. Four teams each
read it for their own purpose.

```json
{
  "event_id": "evt_01J8Z9Q4T2",
  "type": "OrderPlaced",
  "occurred_at": "2026-09-02T18:42:17Z",
  "order_id": "order_123",
  "customer_id": "customer_42",
  "total_cents": 12999
}
```

```{mermaid}
flowchart LR
  checkout[Checkout service] -->|OrderPlaced| orders[(Orders stream)]
  orders --> inventory[Inventory]
  orders --> billing[Billing]
  orders --> email[Email]
  orders --> analytics[Analytics]
```

## Events and logs

A stream is an **append-only log** of events, kept in order. Unlike a queue,
reading an event doesn't remove it. Each consumer remembers its own position
(its **offset**), so:

- Each team reads at its own pace.
- If analytics goes down for an hour, the others keep going. When it comes back,
  analytics continues from where it stopped.
- A new team can start from the beginning and **replay** all past orders.

Name events as **facts** (`OrderPlaced`), not instructions (`SendOrderEmail`).
A fact can be used by any number of teams; an instruction has one owner.

## Partitions and ordering

A stream is split into **partitions** so several machines can share the load.
Each event's **key** decides its partition.

```{mermaid}
flowchart LR
  producer[Checkout] --> router{"hash(customer_id)"}
  router --> p0[(Partition 0)]
  router --> p1[(Partition 1)]
  router --> p2[(Partition 2)]
```

- Order is guaranteed **within a partition**, not across partitions.
- Keying by `customer_id` keeps each customer's events in order.
- A huge customer can make one partition "hot."
- Within one consumer group, each partition goes to one consumer. With 3
  partitions, a 4th consumer sits idle.

## Offsets and consumer groups

A **consumer group** is one team's set of consumers. Billing's group and
analytics' group each get every event, and within a group the partitions are
divided among its members.

The consumer **commits** its offset to record progress. When to commit matters:

| Commit... | If the consumer crashes | Called |
| --- | --- | --- |
| Before processing | The event is skipped | At most once |
| After processing | The event is processed again | At least once (the usual choice) |

**Example:** billing charges order 123, then crashes before committing. On
restart it sees the event again. Passing `event_id` as the payment provider's
idempotency key prevents a second charge.

"Exactly once" exists only within specific systems, such as Kafka-to-Kafka
transactions. It doesn't stop an email from being sent twice. Make side effects
idempotent.

## Event time and windows

- **Event time:** when it happened.
- **Processing time:** when your system saw it.

They differ because events can arrive late, for example from a phone that was
offline:

```text
Happened at:   10:01  10:04  10:09
Arrived as:    10:01  10:09  10:04   ← the 10:04 event is late
```

To count "orders per minute" correctly, group by event time and wait a little
for stragglers. A **watermark** is the processor's estimate that "everything
before 10:05 has probably arrived," so the 10:04 window can close.

Window types:

- **Tumbling:** back-to-back fixed blocks, like revenue per minute.
- **Sliding:** overlapping blocks, like errors in the last 5 minutes, updated
  every minute.
- **Session:** a burst of activity ending after a gap, like one browsing visit.

## Lag and backpressure

**Consumer lag** is how far a consumer is behind the newest event. Retention
absorbs short bursts, but if consumers are always slower than producers, lag
grows forever. Fixes: add consumers (up to the partition count), make each event
cheaper to process, or add partitions.

Send events that keep failing to a **dead-letter stream** instead of blocking
the partition.

## Replay and schema changes

Replay lets you rebuild a broken view or start a new one from history. Throttle
it so it doesn't overload live systems.

Events live a long time, so change their shape carefully: add new optional
fields, never remove or rename fields old consumers rely on, and test new code
against old events.

## Redis Pub/Sub and Streams

[Redis](../caching/redis.md) has two options. **Pub/Sub** is live only: a
disconnected subscriber misses messages. **Redis Streams** keeps events and
supports consumer groups and replay. It works for small workloads; use Kafka for
long retention, many consumer groups, or large replays.

## Checklist

- What's the event's key, and what order do you actually need?
- When do consumers commit, and is reprocessing safe?
- How will you handle late events?
- How long is retention, and how will you monitor lag?
- How will event schemas change without breaking consumers?
