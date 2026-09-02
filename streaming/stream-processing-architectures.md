---
title: Stream-Processing Architectures
description: Processing graphs, stateful operators, materialized views, event-driven applications, checkpoints, and recovery.
---

# Stream-Processing Architectures

Stream processing continuously turns input events into new events, alerts, or
queryable views. Unlike a consumer that handles one record independently, a
processor often remembers state across records and coordinates recovery across
an entire graph of operators.

## A processing graph

Consider fraud detection for card payments:

```{mermaid}
flowchart LR
  payments[(Payments stream)] --> validate[Validate schema]
  validate --> key[Key by card_id]
  key --> velocity[Count last 5 minutes]
  profiles[(Customer profiles)] --> enrich[Enrich]
  velocity --> enrich
  enrich --> rules{Fraud rules}
  rules -->|suspicious| alerts[(Fraud alerts)]
  rules -->|allowed| approved[(Approved payments)]
```

`Validate` is stateless: each record can be processed alone. The five-minute
count is stateful: it remembers recent payments for each card. Enrichment is
stateful when profiles are held locally or joined over time.

## Stateless and stateful operators

Common stateless operators include:

- `map`: turn cents into dollars;
- `filter`: keep only successful payments; and
- `flatMap`: split one envelope into several events.

Common stateful operators include:

- windowed counts and sums;
- joins between streams or with changing reference data;
- deduplication by event ID;
- pattern detection across a sequence; and
- materialized views such as current account totals.

State should be partitioned by the same key as the events that update it. That
lets each parallel operator own a disjoint subset—for example, all state for
`card_42`—without a remote database call per event.

## Example: real-time conversion rate

Inputs:

```text
ProductViewed  {user_id, product_id, occurred_at}
OrderPlaced    {user_id, product_id, occurred_at}
```

The processor groups by `product_id`, counts both events in five-minute windows,
and emits:

```json
{
  "window_start": "2026-09-02T18:40:00Z",
  "product_id": "product_123",
  "views": 820,
  "orders": 41,
  "conversion_rate": 0.05
}
```

A dashboard reads this compact output instead of rescanning raw click history.
Late events either update the result, go to a correction stream, or are dropped
according to the declared lateness policy.

## Common architectures

| Architecture | Input | Output | Example |
| --- | --- | --- | --- |
| Streaming ETL | Raw operational events | Clean, enriched events | Normalize CDC before a warehouse |
| Materialized view | Domain events | Query-optimized table | Current order status by customer |
| Event-driven application | Business events | Commands or new events | Reserve inventory after `OrderPlaced` |
| Streaming analytics | Telemetry or behavior | Windows and alerts | Error rate over the last five minutes |
| Complex event processing | Several event types | Detected patterns | Login from two countries within ten minutes |

### Streaming ETL

Parse, validate, redact, enrich, and route data continuously. Keep the original
event available so bad transformations can be repaired and replayed.

### Materialized views

Fold a history of facts into current state. For example,
`OrderPlaced → PaymentCaptured → OrderShipped` produces one row describing the
latest order status. Make view updates idempotent and store the last applied
offset or version with the view when possible.

### Event-driven applications

Services react to facts without direct synchronous calls. This reduces temporal
coupling but introduces eventual consistency. Use an outbox or equivalent atomic
publication pattern so a database commit and its event cannot silently diverge.

### Streaming analytics and patterns

Windows answer questions over recent time. Pattern processors detect sequences,
absence, or thresholds—for example, three failed logins followed by a successful
login from a new device.

## State, checkpoints, and recovery

```{mermaid}
sequenceDiagram
  participant S as Replayable source
  participant P as Stateful processor
  participant C as Checkpoint storage
  participant O as Output sink
  S->>P: records after offset 8200
  P->>O: transactional/idempotent output
  P->>C: state snapshot + source offsets
  Note over P: processor crashes
  C-->>P: restore snapshot
  S-->>P: replay from checkpointed offsets
```

A consistent checkpoint ties operator state to source positions. After failure,
the processor restores both and replays later events. End-to-end exactly-once
behavior additionally requires a sink that participates transactionally or
handles repeated output idempotently.

Automatic checkpoints optimize failure recovery. Manually retained savepoints
support planned upgrades, migrations, and rescaling. Treat them like operational
state: protect the storage, test restoration, and keep operator identities
stable across compatible deployments.

## Lambda versus Kappa

A **Lambda architecture** has separate batch and streaming paths, then merges
their results. It can recompute accurate history with mature batch tools, but
duplicates business logic and creates two systems to operate.

A **Kappa architecture** uses one replayable log and one processing path for
both live data and recomputation. It is simpler when the stream processor can
efficiently replay the required history.

Prefer the smallest architecture that meets correctness and recovery needs. A
nightly batch job may be better than permanent streaming infrastructure when a
result does not need to be fresh.

## Scaling and backpressure

Parallelism is limited by input partitions and by the ability to repartition
state. Increasing workers may trigger a shuffle and move large state snapshots.
Hot keys remain hot even when the rest of the workload is evenly distributed.

Backpressure should flow upstream rather than allowing unbounded buffers. Watch
input lag, busy time, checkpoint duration and failures, state size, watermark
delay, output latency, and skew by key.

## Design checklist

- Which operators are stateful, and what key partitions their state?
- Is event time required, and how much lateness is acceptable?
- Can the source replay and can the sink tolerate repeated writes?
- How large can state and checkpoints become?
- How will schema and application upgrades preserve state compatibility?
- Is continuous processing worth its operational cost versus periodic batch?

## Sources and further reading

- [Apache Flink: stateful stream processing](https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/stateful-stream-processing/)
- [Apache Flink: checkpoints](https://nightlies.apache.org/flink/flink-docs-stable/docs/ops/state/checkpoints/)
- [Apache Flink: event time and watermarks](https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/time/)
