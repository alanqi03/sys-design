---
title: Change Data Capture vs. Outbox and Queue Patterns
description: How CDC, transactional outbox, and queue-first designs reliably move database changes between services.
---

# Change Data Capture vs. Outbox and Queue Patterns

A service often needs to update its database and tell another system what
happened. The hard part is making those two effects reliable without a
distributed transaction.

- **Change Data Capture (CDC)** observes committed database changes, usually
  from a transaction log, and converts them into a change stream.
- A **transactional outbox** writes an intentional domain event beside the
  business data in the same database transaction. A relay publishes it later.
- A **queue-first command** durably stores requested work before a worker changes
  the database. It makes the operation asynchronous rather than publishing a
  completed fact.

The patterns can be combined. A common design writes an outbox row and uses CDC
as the relay that publishes that row to a broker.

## Quick comparison

| | Raw CDC | Transactional outbox | Queue-first command |
| --- | --- | --- | --- |
| Captures | Row changes | Deliberate domain events | Work that should happen |
| First durable record | Business database change | Business change and outbox row together | Queue message |
| Application change | Often little for existing tables | Must create an event in the transaction | API and workflow become asynchronous |
| Event contract | Coupled to storage schema unless transformed | Designed by the owning service | Command schema |
| Best fit | Replication, search, analytics, cache/view updates | Service integration and business events | Slow, bursty, or retryable background work |
| Main cost | Connector operations and schema coupling | Outbox table, relay, cleanup, duplicate delivery | Eventual completion and idempotent workers |

## The dual-write problem

This code contains a failure window:

```text
1. COMMIT order to database
2. Publish OrderPlaced to broker
```

If the process crashes between steps 1 and 2, the order exists but no event is
published. Reversing the steps is also unsafe: the event can be published and
the database transaction can later fail.

```{mermaid}
sequenceDiagram
  participant S as Order service
  participant D as Database
  participant B as Broker
  S->>D: COMMIT order
  D-->>S: Success
  Note over S: Process crashes
  S-xB: OrderPlaced never published
```

Retrying the publish helps only if the service still knows that publication is
pending. CDC and the outbox pattern each create a durable record from which
publication can resume.

## Change Data Capture

A transactional database records changes in a log for recovery or replication:
PostgreSQL has WAL, MySQL has the binary log, and many managed databases expose
an equivalent change stream. A CDC connector tracks a durable position, decodes
committed changes, and writes change events to a stream, queue, or sink.

```{mermaid}
flowchart LR
  service[Service] -->|Transaction| db[(Database)]
  db --> log[(Transaction log)]
  log --> connector[CDC connector]
  connector --> stream[(Change stream)]
  stream --> search[Search indexer]
  stream --> warehouse[Warehouse loader]
  stream --> cache[Cache/view updater]
```

A row-level event might resemble:

```json
{
  "operation": "update",
  "table": "orders",
  "key": {"id": "order_123"},
  "before": {"status": "paid"},
  "after": {"status": "shipped"},
  "position": "0/16B6C50",
  "committedAt": "2026-09-03T18:42:00Z"
}
```

The exact envelope and availability of `before` values, transaction metadata,
and schema events depend on the database and connector.

### Strengths

- Captures every committed change in scope, including changes made by scripts or
  older applications that do not publish events themselves.
- Avoids adding a broker publish to the application's transaction path.
- Works well for search indexing, analytics ingestion, audit pipelines, cache
  invalidation, and materialized views.
- Preserves the database log's order within the guarantees exposed by the
  connector.

### Costs and risks

- Row changes describe storage mechanics, not necessarily business meaning. An
  update to `orders.status` does not explain why it changed.
- Consumers can become coupled to table names, columns, tombstones, and database
  migrations. Use an owned transformation layer and versioned event contract.
- Initial snapshots, schema changes, deletes, large transactions, and log
  retention need explicit handling.
- A stalled replication slot or connector can retain logs and exhaust database
  storage. Monitor lag in both time and bytes.
- Delivery is commonly at least once around crashes; consumers must tolerate
  duplicates.

Choose raw CDC when downstream systems genuinely need database changes or an
existing application cannot easily emit domain events. Avoid exposing raw table
events as a permanent public contract for many independent teams.

## Transactional Outbox

The service writes business state and an event record in one local transaction:

```sql
BEGIN;

INSERT INTO orders (id, customer_id, status, total_cents)
VALUES ('order_123', 'customer_42', 'placed', 12999);

INSERT INTO outbox_events (
  event_id, aggregate_type, aggregate_id, event_type, payload, created_at
) VALUES (
  'evt_789',
  'order',
  'order_123',
  'OrderPlaced',
  '{"orderId":"order_123","customerId":"customer_42","totalCents":12999}',
  now()
);

COMMIT;
```

If the transaction rolls back, neither row exists. If it commits, the event is
durably waiting even when the broker is unavailable.

```{mermaid}
flowchart LR
  service[Order service] -->|One transaction| db[(Orders + outbox)]
  db --> relay[Outbox relay]
  relay --> broker[(Broker)]
  broker --> consumers[Consumers]
```

The relay can use either approach:

- **Polling publisher:** claim unpublished rows in batches, publish them, and
  mark or delete them. It is simple and database-portable but adds polling load
  and requires careful concurrent claiming.
- **CDC relay:** capture inserts from the outbox table and route them to broker
  topics. It avoids polling and an extra status update but requires CDC
  infrastructure and log-retention operations.

### Design the outbox event

Include a stable `event_id`, event type, schema version, occurrence time,
aggregate ID, and payload. Use the aggregate ID—such as `order_123`—as the broker
partition key when per-order ordering matters. Keep the payload meaningful to
consumers rather than mirroring every database column.

Outbox events are normally append-only. Retain them until the relay has safely
published them, then archive or delete them without allowing the table and its
indexes to grow forever.

The outbox makes **recording the intent to publish** atomic with the business
change. It does not make broker publication exactly once. A relay can publish an
event and crash before recording progress, so it publishes the same event again
after restart.

## Queue-First Commands

Sometimes the correct first durable action is to enqueue work:

```{mermaid}
flowchart LR
  client[Client] -->|POST /exports| api[API]
  api -->|GenerateExport command| queue[[Durable queue]]
  api -->|202 + job ID| client
  queue --> worker[Worker]
  worker --> db[(Job state)]
  worker --> files[(Object storage)]
```

This is a good fit for reports, media processing, notifications, imports, and
other work that can finish later. The queue absorbs bursts and retries failed
workers. See [Queue Fundamentals](../queues/queue-fundamentals.md).

A queue-first design is not a substitute for an outbox when a synchronous
database transaction must publish a fact afterward. Publishing a queue message
and updating a separate database are still two writes unless one is derived
reliably from the other. Queue workers must also be idempotent because a message
can be delivered again after an ambiguous failure.

## The common hybrid: Outbox plus CDC

```{mermaid}
flowchart LR
  app[Application] -->|Commit business row + domain event| db[(Database)]
  db --> wal[(Transaction log)]
  wal --> cdc[CDC connector]
  cdc -->|Only outbox inserts| broker[(Broker)]
  broker --> billing[Billing]
  broker --> email[Email]
  broker --> analytics[Analytics]
```

This hybrid keeps event semantics in application code while CDC handles
reliable publication. It is often the strongest default for domain events when
the organization already operates CDC connectors.

## Delivery, ordering, and consumers

Design every option around at-least-once delivery:

1. Give each event or command a stable ID.
2. Partition by the entity whose order matters, not by a random event ID.
3. Make the consumer's state change and deduplication record atomic when
   possible.
4. Advance the consumer offset or acknowledge the message only after its local
   work commits.
5. Use bounded retries and a dead-letter path for poison messages.

An inbox table is one way to deduplicate:

```sql
BEGIN;

INSERT INTO consumed_events (consumer_name, event_id)
VALUES ('billing', 'evt_789')
ON CONFLICT DO NOTHING;

-- Apply the billing state change only if the insert added a row.

COMMIT;
```

“Exactly once” requires a precisely defined boundary. A connector or broker may
avoid duplicates within its own transaction, while an email, external payment,
or separate database can still happen twice after a crash.

## Choosing a pattern

- Choose **raw CDC** for replication-like consumers, data integration, or
  existing tables whose changes must all be observed.
- Choose a **transactional outbox** when the service should publish a stable,
  intentional business event.
- Choose a **queue-first command** when accepting work durably now and completing
  it asynchronously is part of the API contract.
- Choose **outbox plus CDC** when you want domain events without operating a
  polling publisher.

Whichever pattern you choose, define ownership, schemas, ordering scope,
duplicate handling, replay behavior, retention, and monitoring before treating
the pipeline as reliable.

## Further reading

- [PostgreSQL: Logical decoding concepts](https://www.postgresql.org/docs/current/logicaldecoding-explanation.html)
- [Debezium: Outbox Event Router](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html)
- [AWS Prescriptive Guidance: Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
