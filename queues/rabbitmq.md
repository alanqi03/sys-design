---
title: RabbitMQ
description: A concise system-design guide to RabbitMQ exchanges, queues, routing, acknowledgements, and reliability.
---

# RabbitMQ

RabbitMQ is a message broker you run yourself (or rent as a managed service).
Its main difference from SQS is **routing**: producers publish to an
**exchange**, and the exchange decides which queues get a copy.

## Example: routing order events

The order service publishes a message with routing key `order.paid`. It doesn't
know which queues exist:

```{mermaid}
flowchart LR
  producer[Order service] -->|"order.paid"| exchange{orders exchange}
  exchange -->|"binding: order.paid"| fulfillment[[Fulfillment queue]]
  exchange -->|"binding: order.*"| analytics[[Analytics queue]]
  fulfillment --> w1[Fulfillment workers]
  analytics --> w2[Analytics workers]
```

- The fulfillment queue only wants paid orders, so it binds to `order.paid`.
- The analytics queue wants every order event, so it binds to `order.*`.
- Both get their own copy. Workers on the same queue share its messages.

Adding a new consumer just means binding a new queue. The producer doesn't
change.

## Exchange types

| Type | Routes when | Example |
| --- | --- | --- |
| Direct | The routing key matches exactly | `order.paid` → payments queue |
| Topic | The key matches a pattern (`*` = one word, `#` = any) | `order.*` → analytics queue |
| Fanout | Always; copies to every bound queue | Broadcast cache invalidations |
| Headers | Message headers match | Route by `region=eu` |

## Acknowledgements

RabbitMQ pushes messages to workers and waits for a reply:

- `ack`: done; remove the message.
- `nack` / `reject`: failed; requeue it or send it to a dead-letter queue.

If the worker's connection drops before it replies, RabbitMQ requeues the
message automatically. That means messages can arrive twice, so workers must be
idempotent.

**Prefetch** limits how many unacknowledged messages one worker holds at once.
A worker that runs 4 jobs in parallel might use a prefetch of 4–8.

## Retries with a delay

Requeueing a failed message immediately can create a tight failure loop. A
common pattern sends it to a retry queue with a TTL, which routes it back after
the delay:

```{mermaid}
flowchart LR
  work[[Work queue]] --> worker[Worker]
  worker -->|ack| done[Done]
  worker -->|temporary failure| retry[[Retry queue, 30 s TTL]]
  retry -->|TTL expires| work
  worker -->|too many attempts| dlq[[Dead-letter queue]]
```

## Durability

To survive a broker restart, you need all of:

- **Quorum queues**, which replicate messages across nodes (the modern default
  for important data);
- **persistent** messages; and
- **publisher confirms**, so the producer knows RabbitMQ actually stored the
  message.

## Pros and cons

**Pros**

- Flexible routing: direct, topic, fanout, and headers.
- Push delivery, prefetch, priorities, TTLs, and dead-lettering built in.
- Runs anywhere: any cloud or your own datacenter.

**Cons**

- More to run and configure than SQS.
- Easy to misconfigure durability or retries.
- One queue has a throughput limit; scaling out may mean splitting the work
  across queues.

## When to choose RabbitMQ

Choose it when you need routing rules, push delivery, or a broker that runs
anywhere. Choose SQS to avoid running a broker on AWS. Choose Kafka when
consumers need to replay history.

## Further reading

- [RabbitMQ exchanges](https://www.rabbitmq.com/docs/4.1/exchanges)
- [Consumer acknowledgements and publisher confirms](https://www.rabbitmq.com/docs/4.1/confirms)
- [RabbitMQ quorum queues](https://www.rabbitmq.com/docs/4.1/quorum-queues)
- [RabbitMQ dead-letter exchanges](https://www.rabbitmq.com/docs/4.1/dlx)
