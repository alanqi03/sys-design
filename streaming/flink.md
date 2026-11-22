---
title: Apache Flink
description: A system-design guide to stateful stream processing with Apache Flink.
---

# Apache Flink

Apache Flink is a stream-processing engine. Kafka **stores** events; Flink
**computes** over them continuously: counts, windows, joins, alerts. It keeps
the in-progress results (state) safe across crashes.

## Example: revenue per product every 5 minutes

```sql
SELECT window_start, product_id, SUM(quantity * unit_price) AS revenue
FROM TABLE(
  TUMBLE(TABLE orders, DESCRIPTOR(event_time), INTERVAL '5' MINUTES)
)
GROUP BY window_start, window_end, product_id;
```

```{mermaid}
flowchart LR
  kafka[(Kafka: orders)] --> source[Read]
  source --> key["Group by product_id"]
  key --> window["Sum per 5-min window<br/>(state)"]
  window --> sink[(Dashboard table)]
  checkpoint[(Checkpoint storage)] -.-> window
```

- Flink reads orders from Kafka and groups them by `product_id`.
- It keeps a running sum per product per window. That running sum is its
  **state**.
- When the **watermark** says the 18:40–18:45 window is complete, Flink emits
  the totals.
- It regularly saves state to checkpoint storage, so a crash loses nothing.

## When you need Flink

A plain Kafka consumer is enough if each event is handled on its own, like
"send an email for this order." Reach for Flink when you need:

- counts or sums over time windows;
- joins between streams;
- correct handling of late events (event time);
- large state spread across many machines, recovered automatically after
  crashes.

## How it runs

```{mermaid}
flowchart TB
  jm[JobManager] --> tm1[TaskManager 1]
  jm --> tm2[TaskManager 2]
  jm --> tm3[TaskManager 3]
  tm1 & tm2 & tm3 --> storage[(Checkpoint storage)]
```

- **JobManager:** the coordinator. Schedules work, triggers checkpoints, and
  restarts after failures.
- **TaskManagers:** the workers. Each runs a slice of the job.

## Keyed state

`keyBy(product_id)` sends all events for a product, and that product's state, to
the same worker:

```text
Worker 1: product_1, product_9, product_42
Worker 2: product_3, product_7
Worker 3: product_2, product_8
```

Each worker updates its own products locally, with no database call per event.
Set a TTL on state if the number of keys grows forever.

## Event time and watermarks

Flink can group events by when they **happened**, not when they arrived. A
watermark of `18:45` means "events before 18:45 have probably all arrived," so
windows ending at or before then can close. Events arriving after that can
update the result, go to a side output, or be dropped.

## Checkpoints and savepoints

- **Checkpoint:** taken automatically. Records all state plus the Kafka offsets
  read so far. After a crash, Flink restores it and re-reads Kafka from those
  offsets.
- **Savepoint:** taken by hand before an upgrade or rescaling, then restored by
  the new version of the job.

Flink's state is exactly-once, but writes to external systems may repeat after a
recovery unless the sink is transactional or idempotent.

## Scaling

- More parallelism splits keys across more workers, moving their state.
- Parallelism can't exceed the number of Kafka partitions you read from.
- A slow output database limits the whole job.
- Watch input lag, backpressure, checkpoint duration, and state size.

## Pros and cons

**Pros**

- Built for stateful, event-time processing.
- Automatic recovery from checkpoints.
- SQL for simple jobs; Java/Python APIs for complex ones.
- Handles both live streams and backfills.

**Cons**

- A significant cluster to run and tune.
- Large state makes upgrades and rescaling slow.
- Overkill when each event can be handled on its own.

## When to choose Flink

Choose it for real-time analytics, fraud detection, streaming ETL, live
dashboards, and session analysis: anything that needs state across events. Use
a simple consumer for per-event work, and a queue or workflow engine for task
orchestration.

## Further reading

- [Apache Flink documentation](https://nightlies.apache.org/flink/flink-docs-stable/)
- [Flink stateful stream processing](https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/stateful-stream-processing/)
- [Flink checkpoints](https://nightlies.apache.org/flink/flink-docs-stable/docs/ops/state/checkpoints/)
