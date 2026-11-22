---
title: Stream-Processing Architectures
description: Processing graphs, stateful operators, materialized views, event-driven applications, checkpoints, and recovery.
---

# Stream-Processing Architectures

A stream processor reads events continuously and produces new events, alerts, or
up-to-date tables. Unlike a simple consumer that handles each event on its own, a
processor often **remembers** things across events, like "how many payments has
this card made in the last 5 minutes?"

## Example: card fraud detection

```{mermaid}
flowchart LR
  payments[(Payments stream)] --> validate[Validate]
  validate --> key["Group by card_id"]
  key --> count["Count payments<br/>in last 5 min"]
  count --> rules{"More than 10?"}
  rules -->|yes| alerts[(Fraud alerts)]
  rules -->|no| approved[(Approved)]
```

- **Validate** is **stateless**: each payment is checked on its own.
- **Count** is **stateful**: it remembers recent payments for each card.

Group state by the same key as the events (`card_id`). Each worker then owns
all the state for its cards and never needs a database call per event.

## Stateless vs. stateful

| Stateless (each event alone) | Stateful (remembers across events) |
| --- | --- |
| `map`: cents → dollars | Count or sum over a time window |
| `filter`: keep successful payments | Join two streams |
| `flatMap`: split one event into several | Drop duplicate event IDs |
| | Spot a pattern ("3 failed logins, then a success") |

## Common architectures

| Architecture | What it does | Example |
| --- | --- | --- |
| **Streaming ETL** | Clean, enrich, and route raw events | Normalize CDC events before loading the warehouse |
| **Materialized view** | Fold events into a current-state table | `OrderPlaced → Paid → Shipped` becomes one row: status = shipped |
| **Event-driven app** | React to events with new actions | Reserve inventory when `OrderPlaced` arrives |
| **Streaming analytics** | Metrics over recent time | Error rate over the last 5 minutes |
| **Pattern detection** | Find sequences across events | Logins from two countries within 10 minutes |

**Example: live conversion rate.** Count `ProductViewed` and `OrderPlaced` per
product in 5-minute windows and emit:

```json
{ "window_start": "18:40", "product_id": "p_123", "views": 820, "orders": 41, "conversion_rate": 0.05 }
```

A dashboard reads this small result instead of re-scanning every click.

## Checkpoints and recovery

What happens to the "last 5 minutes" counts if a worker crashes? The processor
regularly saves a **checkpoint**: a snapshot of all its state plus how far it
has read in each input.

```{mermaid}
sequenceDiagram
  participant S as Input stream
  participant P as Processor
  participant C as Checkpoint storage
  S->>P: events after offset 8200
  P->>C: save state + "read up to 8200"
  Note over P: crash
  C-->>P: restore state
  S-->>P: replay from 8200
```

After a crash, the processor restores the state and re-reads from the saved
position, so nothing is lost. Some output may be written twice, so outputs must
be idempotent or transactional.

A **savepoint** is a checkpoint you trigger by hand before an upgrade or
rescaling.

## Lambda vs. Kappa

- **Lambda:** a batch pipeline and a streaming pipeline side by side, with
  results merged. Accurate, but you write and run the logic twice.
- **Kappa:** one streaming pipeline. To recompute history, replay the log
  through it. Simpler, if the log keeps enough history.

If results only need to be fresh once a day, a nightly batch job may be simpler
than any streaming system.

## Scaling

- Parallelism is capped by the number of input partitions.
- Rescaling moves state between workers, which is slow when state is large.
- A hot key (one very busy card) stays on one worker however many you add.
- Watch input lag, checkpoint time, state size, and how far watermarks trail.

## Checklist

- Which steps need state, and what key splits it?
- Do you need event time, and how late can events arrive?
- Can the input be replayed, and can the output handle duplicates?
- How big will state and checkpoints get?
- Is real-time worth it, or would a batch job do?

## Further reading

- [Apache Flink: stateful stream processing](https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/stateful-stream-processing/)
- [Apache Flink: checkpoints](https://nightlies.apache.org/flink/flink-docs-stable/docs/ops/state/checkpoints/)
- [Apache Flink: event time and watermarks](https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/time/)
