---
title: Streaming
---

# Streaming

Streaming systems retain an ongoing sequence of events and continuously process
new records as they arrive. They decouple producers from consumers, let several
consumer groups build independent views, and allow old events to be replayed.

```{mermaid}
flowchart LR
  producers[Producers] --> log[(Partitioned event log)]
  log --> group1[Operational consumers]
  log --> group2[Analytics consumers]
  log --> processor[Stream processor]
  processor --> views[(Materialized views)]
```

## In this section

- [Streaming Fundamentals](./streaming/streaming-fundamentals.md) covers events,
  partitions, offsets, ordering, delivery semantics, event time, windows,
  backpressure, and replay.
- [Stream-Processing Architectures](./streaming/stream-processing-architectures.md)
  covers stateless and stateful operators, processing graphs, materialized
  views, event-driven applications, and recovery.
- [Apache Kafka](./streaming/kafka.md) explains topics, partitions, brokers,
  replication, offsets, and consumer groups.
- [Apache Flink](./streaming/flink.md) explains distributed stateful processing,
  event-time computation, checkpoints, savepoints, and scaling.

## Stream versus queue

- A **stream** preserves reusable facts such as `OrderPlaced`; every interested
  consumer group can process and replay them.
- A **queue** assigns a task such as `GenerateInvoicePDF` to one worker and
  normally removes it after success.

See [Queues](./queues.md) for acknowledgements, visibility leases, retries,
dead-letter queues, worker idempotency, SQS, and RabbitMQ.
