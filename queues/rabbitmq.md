---
title: RabbitMQ
description: A concise system-design guide to RabbitMQ exchanges, queues, routing, acknowledgements, and reliability.
---

# RabbitMQ

RabbitMQ is a message broker. Producers publish to an **exchange**, bindings
route each message to one or more **queues**, and consumers receive from those
queues. This routing layer is the main conceptual difference from simpler queue
services.

```{mermaid}
flowchart LR
  producer[Order service] --> exchange{orders exchange}
  exchange -->|order.paid| fulfillment[[Fulfillment queue]]
  exchange -->|order.*| analytics[[Analytics queue]]
  fulfillment --> worker1[Fulfillment worker]
  analytics --> worker2[Analytics worker]
```

Each queue receives its own copy when bindings match. Workers on the same queue
compete for its messages.

## Exchanges and routing

- **Direct:** route when the binding key exactly matches the routing key.
- **Topic:** route with patterns such as `order.*` or `order.#`.
- **Fanout:** copy every message to every bound queue.
- **Headers:** route from message-header matches instead of a routing key.

For example, publish an event with routing key `order.paid`. A fulfillment queue
can bind exactly to `order.paid`, while an analytics queue binds to `order.#`.
The publisher does not need to know either queue name.

## Delivery lifecycle

With manual acknowledgements, RabbitMQ keeps a delivery unacknowledged until the
consumer responds:

```text
basic.ack     -> processed successfully; discard from the queue
basic.nack    -> failed; requeue or discard/dead-letter
basic.reject  -> reject one delivery; requeue or discard/dead-letter
```

If the channel or connection closes before ACK, RabbitMQ automatically requeues
the unacknowledged delivery. The `redelivered` flag tells the consumer that the
message may have been seen before, but workers must still be idempotent.

## Prefetch and worker load

Prefetch limits how many unacknowledged deliveries RabbitMQ sends to a consumer.
A prefetch of `1` gives fair distribution for slow jobs but can underuse workers
when network latency is significant. A larger value improves throughput but
increases per-worker buffering and the amount of work that must be requeued
after a crash.

Start near each worker's real concurrency. A process with four job slots might
prefetch four to eight tasks, then tune from processing time and memory use.

## Queue types and durability

- **Quorum queues** replicate data using a consensus protocol and are the usual
  choice when queue availability and data safety matter.
- **Classic queues** are non-replicated in modern RabbitMQ and suit cases where
  the queue can be rebuilt or node-local behavior is intentional.

Durability requires agreement across several settings: declare durable queues
and exchanges, publish persistent messages, and use publisher confirms so the
producer knows RabbitMQ accepted responsibility. Consumer ACKs solve a separate
problem: they confirm processing after delivery.

## Retries and dead lettering

Immediate `nack(requeue=true)` can create a tight failure loop. A common design
dead-letters failures to a retry queue with a TTL; expiration routes them back to
the work exchange after a delay. After the maximum attempt count, route them to
a terminal DLQ.

```{mermaid}
flowchart LR
  work[[Work queue]] --> worker[Worker]
  worker -->|ACK| done[Complete]
  worker -->|temporary failure| retry[[Retry queue + TTL]]
  retry -->|delay expires| work
  worker -->|permanent or max attempts| dlq[[Dead-letter queue]]
```

Track attempts in a trusted header or dead-letter metadata, and guard against
cycles that route a poison message forever.

## Pros

- Flexible exchange and binding model for direct, topic, and fanout routing.
- Push consumers, acknowledgements, prefetch, priorities, TTLs, and dead-letter
  exchanges support sophisticated job delivery.
- Mature AMQP clients and deployment options across clouds and data centers.
- Quorum queues provide replicated, strongly coordinated queue storage.

## Cons

- More concepts and operational work than a managed queue such as SQS.
- Routing, durability, and retry policies can be misconfigured in combinations.
- A single queue has finite throughput; scale-out may require partitioning work
  across multiple queues.
- Requeueing and concurrent consumers weaken any assumption of strict
  end-to-end processing order.

## When to choose RabbitMQ

Choose RabbitMQ when applications need rich broker-side routing, AMQP clients,
push delivery, controlled prefetch, or portability across environments. Choose
SQS when minimizing broker operations inside AWS matters more. Choose Kafka or
another stream when retained history and independent replay are core needs.

## Sources and further reading

- [RabbitMQ consumers](https://www.rabbitmq.com/docs/4.1/consumers)
- [Consumer acknowledgements and publisher confirms](https://www.rabbitmq.com/docs/4.1/confirms)
- [RabbitMQ queues](https://www.rabbitmq.com/docs/4.1/queues)
- [RabbitMQ dead-letter exchanges](https://www.rabbitmq.com/docs/4.1/dlx)
- [RabbitMQ quorum queues](https://www.rabbitmq.com/docs/4.1/quorum-queues)
