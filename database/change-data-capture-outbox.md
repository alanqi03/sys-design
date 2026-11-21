---
title: Change Data Capture vs. Transactional Outbox
description: Two simple ways to reliably tell other systems that your database changed.
---

# Change Data Capture vs. Transactional Outbox

## The problem

A service saves an order and then publishes an `OrderPlaced` event:

```text
1. Save order to database
2. Publish OrderPlaced to the broker
```

If the service crashes between the two steps, the order exists but nobody hears
about it. Swapping the steps is no better: the event goes out, then the save
fails. You can't put the database and the broker in one transaction, so you need
another way to make sure the event eventually goes out.

## Why use them

Why not just have the order service call billing, email, and search directly?
The outbox and CDC give you three things a direct call doesn't:

1. **Decoupling.** The order service only records what happened. It doesn't
   need to know who cares about a new order, and adding a consumer (say,
   analytics) needs no change to the order service.
2. **No lost or phantom events.** An event goes out if and only if the database
   change committed. Other systems never miss an order, and never hear about
   one that was rolled back.
3. **Availability.** Saving an order only needs the database. If the broker or a
   consumer is down, the request still succeeds and the event goes out once it
   recovers.

## Transactional outbox

Write the event to an `outbox` table **in the same database transaction** as the
order. A separate relay process reads the outbox and publishes each event to the
broker.

```{mermaid}
flowchart LR
  service[Order service] -->|"one transaction:<br/>order + outbox row"| db[(Database)]
  db --> relay[Relay]
  relay --> broker[(Broker)]
  broker --> consumers[Consumers]
```

```sql
BEGIN;
INSERT INTO orders (id, status) VALUES ('order_123', 'placed');
INSERT INTO outbox (event_id, type, payload)
  VALUES ('evt_789', 'OrderPlaced', '{"orderId": "order_123"}');
COMMIT;
```

Either both rows are saved or neither is, so an order can never exist without
its event. If the broker is down, the event just waits in the outbox until the
relay can publish it.

## Change Data Capture (CDC)

Every database keeps a log of committed changes (PostgreSQL's WAL, MySQL's
binlog). A CDC tool such as Debezium reads that log and turns each row change
into an event, with no change to application code.

```{mermaid}
flowchart LR
  service[Service] --> db[(Database)]
  db --> log[(Change log)]
  log --> cdc[CDC tool]
  cdc --> stream[(Event stream)]
  stream --> search[Search index]
  stream --> warehouse[Data warehouse]
```

The catch is that CDC events describe *rows*, not *business events*. "The
`status` column changed to `shipped`" exposes your table layout, and consumers
break when you rename a column.

## Comparison

| | Transactional outbox | CDC |
| --- | --- | --- |
| What gets published | Business events you choose (`OrderPlaced`) | Every row change |
| App code changes | Write an outbox row in each transaction | None |
| Best for | Telling other services what happened | Copying data to search, analytics, or caches |

They combine well: write events to an outbox table and use CDC as the relay
that publishes them. You get clean business events without writing a relay
that polls the table.

## Consumers must handle duplicates

Both patterns deliver events **at least once**. A relay can publish an event
and crash before recording that it did, then publish it again on restart. Give
every event a unique ID and have consumers skip IDs they've already processed.

## Further reading

- [Debezium: Outbox Event Router](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html)
- [AWS Prescriptive Guidance: Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
