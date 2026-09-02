---
title: Apache Kafka
description: A system-design guide to Kafka topics, partitions, replication, offsets, and consumer groups.
---

# Apache Kafka

Apache Kafka is a distributed event-log platform. Producers append records to
partitioned topics, brokers retain those records, and consumer groups read them
at their own pace using offsets.

```{mermaid}
flowchart LR
  producer[Order producer] --> topic[(orders topic)]
  topic --> p0[Partition 0 leader]
  topic --> p1[Partition 1 leader]
  topic --> p2[Partition 2 leader]
  p0 --> replicas0[Followers]
  p1 --> replicas1[Followers]
  p2 --> replicas2[Followers]
  p0 --> operations[Operations group]
  p1 --> operations
  p2 --> operations
  p0 --> analytics[Analytics group]
  p1 --> analytics
  p2 --> analytics
```

Every consumer group receives the topic independently. Within a group, each
partition is assigned to one consumer at a time, so consumers share the work.

## Developer choices

- **Topic boundaries:** group events with compatible ownership, retention,
  security, and scaling needs—not one topic per event type by reflex.
- **Record key:** related records with the same key go to the same partition and
  preserve their local order. For orders, `order_id` or `customer_id` may be
  appropriate depending on the invariant.
- **Partition count:** more partitions increase parallelism and broker work but
  are not free; increasing them can change key-to-partition placement.
- **Replication and acknowledgement:** choose replication factor,
  `min.insync.replicas`, and producer acknowledgements together.
- **Retention:** time or size-based retention keeps history; compaction keeps the
  latest record per key. Some topics need both.
- **Consumer commits:** decide when offsets commit and how repeated processing is
  made safe.

## Example record

A producer sends a record to topic `orders.events` with key `order_123`:

```json
{
  "key": "order_123",
  "value": {
    "event_id": "evt_01J8Z9Q4T2",
    "type": "OrderPlaced",
    "order_id": "order_123",
    "total_cents": 12999
  }
}
```

Kafka hashes the key to select a partition. All later records using
`order_123` reach that partition and are appended in order.

## What happens on a write

1. The producer serializes the key and value and selects a partition.
2. It batches records and sends them to that partition's leader broker.
3. The leader appends the batch to its log and followers replicate it.
4. The producer receives success according to its acknowledgement settings.

Batching, compression, and a small linger interval can greatly improve
throughput at the cost of a little latency. An idempotent producer protects
against duplicates caused by producer retries within Kafka's supported path.

## What happens on a read

1. Consumers with the same `group.id` join a consumer group.
2. Kafka assigns each topic partition to one group member.
3. Consumers fetch sequential batches beginning at their current offsets.
4. The group commits offsets so a restarted member knows where to resume.

```text
Partition 0: offset 100  101  102  103
Consumer A:                     ^ next read

Partition 1: offset 205  206  207
Consumer B:                ^ next read
```

A group cannot use more active consumers than assigned partitions. Membership
changes trigger rebalancing; cooperative assignment can reduce how much work
stops and moves.

## Delivery and processing semantics

- Commit before processing for at-most-once behavior.
- Process then commit for at-least-once behavior; duplicates are possible.
- Kafka transactions can atomically consume, produce, and commit offsets inside
  a Kafka-to-Kafka pipeline when configured correctly.

An external database or HTTP API is outside that Kafka transaction. Use an
idempotency key, conditional update, or transactional outbox/inbox pattern for
end-to-end correctness.

## Replication and durability

Each partition has a leader and follower replicas on other brokers. Producers
and consumers normally use the leader. If it fails, an in-sync follower can be
elected.

A common durable policy requires acknowledgements from all in-sync replicas and
sets a minimum number that must remain in sync. This reduces the chance of loss
but makes writes unavailable when too few replicas are healthy. Retention is not
a backup: operator mistakes or application-level deletion can propagate across
replicas.

## Retention and compaction

Time or size retention deletes old log segments regardless of whether consumers
have read them. A consumer whose lag exceeds retention cannot resume from the
missing offsets.

Log compaction retains the latest value for each key:

```text
user_42 -> plan=free
user_7  -> plan=pro
user_42 -> plan=pro    # latest value retained after compaction
```

Compacted topics are useful for rebuilding keyed tables or distributing
configuration. Tombstone records represent deletion.

## Scaling

- Add consumers to a group up to the number of partitions.
- Add partitions when more parallelism is required, after checking key-ordering
  and repartitioning consequences.
- Add brokers and rebalance partition replicas to spread storage and traffic.
- Batch and compress records to use network and disk sequentially.
- Isolate hot keys or redesign the key when one partition dominates.

Measure produce/fetch latency, bytes per second, partition skew, under-replicated
partitions, consumer lag, rebalance frequency, request saturation, and disk
capacity relative to retention.

## Pros

- Durable replay lets new consumers and repaired views process old events.
- Partitions provide horizontal throughput and ordering within a key scope.
- Consumer groups support many independent applications over the same history.
- Mature connectors and processing integrations form a broad ecosystem.

## Cons

- Partitioning, replication, upgrades, security, and capacity planning require
  operational expertise.
- Global ordering conflicts with horizontal parallelism.
- Rebalances, hot partitions, and lag can create large tail delays.
- Kafka is not ideal for arbitrary per-message priorities, long scheduling
  delays, or task-specific ACK/NACK workflows.

## When to choose Kafka

Choose Kafka for durable event history, CDC, integration events, analytics
ingestion, and replayable inputs to processors such as Flink. Choose a queue for
one-owner tasks with retry and DLQ semantics. Choose a database when the primary
operation is querying and updating current state rather than consuming a log.

## Sources and further reading

- [Apache Kafka introduction](https://kafka.apache.org/documentation/#intro_concepts_and_terms)
- [Kafka consumer groups and offsets](https://kafka.apache.org/41/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html)
- [Kafka replication](https://kafka.apache.org/documentation/#replication)
- [Kafka producer configuration](https://kafka.apache.org/documentation/#producerconfigs)
- [Kafka design](https://kafka.apache.org/documentation/#design)
