---
title: Apache Kafka
description: A system-design guide to Kafka topics, partitions, replication, offsets, and consumer groups.
---

# Apache Kafka

Apache Kafka is a distributed event log. Producers append events to **topics**,
Kafka keeps them for a set time, and each consumer group reads at its own pace.

## Example

The checkout service writes to the `orders` topic, which has 3 partitions. The
record's key is the order ID:

```json
{ "key": "order_123",
  "value": { "event_id": "evt_01J8", "type": "OrderPlaced", "total_cents": 12999 } }
```

```{mermaid}
flowchart LR
  producer[Checkout] --> p0[(Partition 0)]
  producer --> p1[(Partition 1)]
  producer --> p2[(Partition 2)]
  p0 --> b1[Billing consumer 1]
  p1 --> b1
  p2 --> b2[Billing consumer 2]
  p0 & p1 & p2 --> a1[Analytics consumer]
```

- Kafka hashes `order_123` to pick a partition. Every later event for that order
  lands in the same partition, in order.
- The **billing** group has 2 consumers, so they split the 3 partitions.
- The **analytics** group gets its own full copy of every event.

Each consumer tracks an **offset** per partition:

```text
Partition 0: 100  101  102  103
                        ^ billing's next read
```

## Developer choices

- **Key:** decides ordering. Events with the same key stay in order.
- **Partition count:** the maximum parallelism for a consumer group. Adding
  partitions later changes which partition a key maps to.
- **Replication:** usually 3 copies, with `acks=all` and
  `min.insync.replicas=2` so a write survives losing a broker.
- **Retention:** keep events for a set time or size (e.g. 7 days), or use
  **compaction** to keep only the latest event per key.
- **When to commit offsets:** usually after processing, with idempotent
  consumers.

## Writes and reads

**Write:** the producer batches records and sends them to the partition's
**leader** broker. The leader appends them to its log, followers copy them, and
the producer gets an acknowledgement.

**Read:** consumers fetch batches from their current offset and periodically
commit it. If consumers join or leave, Kafka **rebalances**, reassigning
partitions among the remaining consumers.

## Delivery guarantees

- Commit, then process → **at most once** (may skip events).
- Process, then commit → **at least once** (may repeat events). The usual
  choice.
- **Kafka transactions** give exactly-once for Kafka-to-Kafka pipelines only.
  Writes to a database or external API still need idempotency.

## Retention and compaction

Normal retention deletes old events after the time or size limit, whether or not
everyone has read them. A consumer that falls further behind than the retention
period loses events.

Compaction keeps only the latest value per key, which is handy for rebuilding a
table:

```text
user_42 → plan=free
user_7  → plan=pro
user_42 → plan=pro     ← only this one is kept for user_42
```

## Scaling

- Add consumers, up to the number of partitions.
- Add partitions for more parallelism. Note that keys will move.
- Add brokers to spread storage and traffic.
- Batch and compress records.
- Watch consumer lag, partition skew, and under-replicated partitions.

## Pros and cons

**Pros**

- Events are kept, so new consumers can replay history.
- Scales to millions of events per second through partitions.
- Many independent consumer groups can read the same data.
- A large ecosystem of connectors (CDC, databases, warehouses).

**Cons**

- Operationally complex.
- Order is only per partition, never global.
- Rebalances and hot partitions can cause delays.
- No per-message retries, delays, or priorities like a queue has.

## When to choose Kafka

Choose it for event history that many teams consume: CDC, service integration
events, analytics pipelines, and input to stream processors like Flink. Use a
queue for one-off tasks with retries, and a database when you mainly need to
query current state.

## Further reading

- [Apache Kafka introduction](https://kafka.apache.org/documentation/#intro_concepts_and_terms)
- [Kafka replication](https://kafka.apache.org/documentation/#replication)
- [Kafka design](https://kafka.apache.org/documentation/#design)
