---
title: Apache Flink
description: A system-design guide to stateful stream processing with Apache Flink.
---

# Apache Flink

Apache Flink is a distributed engine for stateful computation over bounded and
unbounded streams. Kafka commonly stores and transports events; Flink consumes
those events, performs continuous computation, manages state, and writes
results.

```{mermaid}
flowchart LR
  kafka[(Kafka topics)] --> source[Flink source]
  source --> key[Key by customer_id]
  key --> window[Window or keyed state]
  window --> enrich[Join and enrich]
  enrich --> sink[Flink sink]
  sink --> views[(Database or output topic)]
  checkpoint[(Checkpoint storage)] -.-> window
  checkpoint -.-> enrich
```

## What Flink adds

A plain stream consumer can parse an event and call an API. Flink is useful when
the computation needs:

- state partitioned across many workers;
- event-time windows and late-event handling;
- joins between changing streams or tables;
- coordinated checkpoints and recovery;
- rescaling without rebuilding state manually; or
- SQL or processing graphs that run continuously.

## Example: five-minute product revenue

Given an `orders` table backed by a stream, a windowed query can continuously
emit revenue per product:

```sql
SELECT
  window_start,
  window_end,
  product_id,
  SUM(quantity * unit_price) AS revenue
FROM TABLE(
  TUMBLE(TABLE orders, DESCRIPTOR(event_time), INTERVAL '5' MINUTES)
)
GROUP BY window_start, window_end, product_id;
```

Flink keeps partial sums as keyed state. A watermark tells it when a five-minute
window is unlikely to receive more on-time events. The sink receives new or
updated window results according to the table's change semantics.

## Runtime architecture

```{mermaid}
flowchart TB
  client[Job submission] --> manager[JobManager]
  manager --> tm1[TaskManager 1]
  manager --> tm2[TaskManager 2]
  manager --> tm3[TaskManager 3]
  tm1 <--> tm2
  tm2 <--> tm3
  tm1 --> storage[(Checkpoint storage)]
  tm2 --> storage
  tm3 --> storage
```

- The **JobManager** coordinates scheduling, checkpoints, recovery, and resource
  use for a job.
- **TaskManagers** execute parallel operator subtasks and exchange partitioned
  records.
- **Task slots** divide TaskManager resources among subtasks.
- A **state backend** manages working state; checkpoint storage keeps durable
  snapshots outside the workers.

An operator chain may combine compatible adjacent operators in one task to
avoid serialization and network transfer. Key changes, joins, and other
repartitions require a network shuffle.

## Keyed state

Calling `keyBy(customer_id)` partitions both events and state. One parallel
subtask owns a customer's current state at a time:

```text
Task 1: customer_1, customer_9, customer_42
Task 2: customer_3, customer_7
Task 3: customer_2, customer_8
```

Examples of keyed state include a fraud counter, the last sensor reading, a
deduplication set, or a partially complete window. State TTLs are important when
the keyspace can grow forever.

## Event time and watermarks

Flink can compute from the time carried by each event instead of wall-clock
arrival time. A watermark such as `18:45 minus 30 seconds` means the source
believes most events earlier than `18:44:30` have arrived.

Watermarks flow through the processing graph. An idle input that stops advancing
its watermark can hold back downstream windows, so sources need an idleness
policy. Late events may update a result, go to a side output, or be dropped.

## Checkpoints and recovery

A checkpoint records:

- positions in replayable sources;
- state for every stateful operator; and
- coordination metadata needed for consistent output.

If a TaskManager fails, Flink restores the latest successful checkpoint, resets
sources to its offsets, and replays later records. The checkpoint interval
trades runtime overhead against recovery work.

Exactly-once state does not guarantee every external side effect occurs exactly
once. The sink must support a coordinated transaction or idempotent writes.

## Checkpoints versus savepoints

- **Checkpoint:** automatic and optimized for recovery from unexpected failure.
- **Savepoint:** manually triggered and retained for a planned upgrade,
  migration, fork, or rescaling operation.

Give stateful operators stable identifiers before production deployment. Flink
uses those identities to map saved state back to compatible operators after an
application change.

## Scaling

Increasing operator parallelism splits key groups among more subtasks. Restoring
from a checkpoint or savepoint redistributes the corresponding state. Scaling a
stateless map is cheap; moving terabytes of keyed state is not.

Parallelism is also constrained by source partitions and sink capacity. Adding
Flink workers cannot extract more parallel Kafka reads than the relevant topic
partitions, and it cannot fix a database sink that accepts too few writes.

Watch:

- input lag and records per second;
- operator busy, idle, and backpressured time;
- watermark delay;
- state and checkpoint size;
- checkpoint duration, alignment, and failure rate; and
- restart time and sink latency.

## Pros

- First-class distributed state and event-time processing.
- Checkpoint-based recovery for long-running stateful jobs.
- SQL/Table and lower-level stream APIs for different workloads.
- Supports continuous streams and bounded backfills with related abstractions.
- Rescaling and savepoints make large state operationally manageable.

## Cons

- A stateful compute cluster adds substantial deployment and operational work.
- Checkpoints, watermarks, state backends, and connector guarantees require
  careful tuning.
- Large state makes upgrades and rescaling slower.
- End-to-end guarantees are only as strong as the source and sink connectors.
- Simple per-event jobs may be easier in an ordinary consumer service.

## When to choose Flink

Choose Flink for low-latency stateful analytics, streaming ETL, temporal joins,
fraud detection, sessionization, complex event patterns, and continuously
maintained views. Use a simpler consumer when each event is independent. Use a
workflow or queue system when the core problem is task orchestration rather than
continuous computation.

## Sources and further reading

- [Apache Flink documentation](https://nightlies.apache.org/flink/flink-docs-stable/)
- [Flink stateful stream processing](https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/stateful-stream-processing/)
- [Flink checkpoints](https://nightlies.apache.org/flink/flink-docs-stable/docs/ops/state/checkpoints/)
- [Flink savepoints](https://nightlies.apache.org/flink/flink-docs-stable/docs/ops/state/savepoints/)
- [Flink fault tolerance](https://nightlies.apache.org/flink/flink-docs-stable/docs/learn-flink/fault_tolerance/)
